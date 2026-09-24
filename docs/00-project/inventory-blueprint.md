# Multi-Warehouse Inventory System — Complete System Blueprint (v12.0)

**Stack:** Laravel 13 + FilamentPHP v5 + Livewire v4 | **Database:** PostgreSQL / MySQL

**Architecture:** Pure Derived Stock of Truth Ledger + Purchases & Sales Module

> **Changelog from v11.0/v11.1:** This revision merges the parent v11.0 blueprint with the v11.1 Purchases & Sales addendum into a single, unified specification. All Filament v5 constructs have been verified against current official documentation. Every section, phase, and component has been reviewed and reconciled for consistency, DRY principles, and adherence to Filament v5 standards.

---

## 🧭 Section 0: Executive Architecture & System Principles

### Core Principles (from v11.0)

1. **Pure Derived Stock of Truth:** Physical stock levels, active transit reservations, and available balances are never stored in a physical database table. Physical on-hand stock is calculated dynamically at query-time as the sum of all signed records in `stock_movements`. Active reservations sum pending quantities from confirmed requisitions, and available stock is derived as `on_hand - reserved`.

2. **Decoupled Pricing & Variant-Level Catalog:** `sku` lives exclusively on `product_variants`. Parent products act purely as family grouping containers. `reorder_point` lives exclusively on `product_variants`. Unit pricing is decoupled into `product_variant_prices` with `is_current = true`, supporting 4-decimal micro-pricing.

3. **Pessimistic Locking & Transaction Isolation:** All stock deductions, dispatches, and intake receipts execute inside atomic database transactions using pessimistic row-level locking on `product_variants`, `warehouses`, and `transfer_requisitions`. Locking discipline is uniform across *every* multi-warehouse-touching service method.

4. **Canonical Foreign Key & Plural Naming:** All database tables use explicit plural snake_case names. Foreign keys strictly follow table-bound names.

5. **Physical-to-Digital State Lifecycle:**
   ```
   draft → requested → under_review_fulfiller ⇌ under_review_requestor
         → confirmed → dispatched ⇌ partially_received
         → completed / closed_with_loss / cancelled
   ```
   `partially_received` is a first-class live state. It re-enters the receivable modal until every item's `received_good_base_qty + received_damaged_base_qty >= shipped_base_qty`.

6. **Negotiated Substitute Variant Swapping:** Dispatch and receipt pipelines dynamically resolve `$actualVariantId = $item->substitute_product_variant_id ?? $item->product_variant_id`.

7. **Scanned Receipt Loss Integrity & Omitted Cargo:** On the first intake scan, dispatched items missing from a physical scan payload are recorded as 0 received, triggering a 100% variance write-off. On subsequent scans, omitted items are treated as still in transit.

8. **Signed Web QR Routing:** STN QR codes embed secure 7-day temporary signed URLs.

9. **Modal-First UI (< 8 Inputs Rule):** Compact operations use inline slide-over Drawers or Dialog Modals. Multi-step wizards use `modalWidth(Width::SevenExtraLarge)`.

10. **Strongly-Typed Icons, Multi-Language i18n & Currency:** All six backed enums route `getLabel()` through `__()`. All `->money()` calls pass `config('app.currency')`.

11. **Ledger FK Immutability:** Every `product_variant_id` foreign key on a ledger table uses `restrictOnDelete`.

12. **Authorization vs Visibility:** `->authorize()` enforces server-side policy security. `->visible()` controls frontend DOM rendering.

13. **Reservation Scope Boundary:** `reservedQuantity()` is intentionally and permanently bounded to requisitions in `Confirmed` status only. Once a requisition transitions to `Dispatched`, its reserved stock is superseded by the `TransitOut` stock movement (already reflected in `onHandQuantity()`). In-transit and partially-received cargo is never double-counted as "reserved" against the origin warehouse.

14. **Cancellation Boundary:** `CancelAction` is only legal while a requisition is in a pre-dispatch state. Once `TransitOut` has fired (i.e., status is `Dispatched` or `PartiallyReceived`), cancellation is permanently unavailable — there is no compensating stock-reversal pathway in this system, by design. This eliminates an entire class of reversal-logic bugs rather than requiring one.

15. **Cost Snapshot Timing:** `LossLedger::snapshotUnitCostFrom()` captures `currentPrice.cost_price` **at call-time** — i.e., at the moment intake/loss is actually processed, not at the moment the requisition was originally dispatched. Loss valuation therefore reflects present-day replacement cost, not historical acquisition cost.

### Addendum Principles (from v11.1 — Purchases & Sales)

**A1. Same Ledger, New Movement Types.** Per Principle #1, `onHandQuantity()` sums *all* signed `stock_movements` rows for a variant+warehouse. Purchases and sales are simply new `StockMovementType` cases — no new "stock" table is introduced.

**A2. Purchases and Sales Are Symmetric, Single-Entity Flows — Not Negotiated.** A purchase involves one external supplier and one internal warehouse; a sale involves one internal warehouse and one external customer. Each gets a lightweight **draft → confirmed → (partially_fulfilled) → completed / cancelled** lifecycle.

**A3. External Party Entities Are Minimal Master Data.** `suppliers` and `customers` are simple lookup tables (name, contact, is_active), not full CRM/vendor-management systems.

**A4. Cost & Price Interplay With `product_variant_prices`.**
- A **received purchase** at a cost different from the variant's current `cost_price` triggers a new `product_variant_prices` row (`is_current = true`), opt-in per purchase order (`update_cost_price` flag).
- A **sale** always dispatches at `currentPrice.sale_price` at the moment of confirmation, snapshotted onto the sale item row (`unit_sale_price_snapshot`).

**A5. Reservation Boundary Stays Untouched, Sales Get Their Own Boundary.** Per Principle #13, `reservedQuantity()` is **permanently and explicitly** bounded to `Confirmed` transfer requisitions. A **new, separate** method `reservedForSalesQuantity()` sums `Confirmed`-status `sales_order_items`. `availableQuantity()` is extended to net out *both*.

**A6. No Reversal Pathway for Dispatched Sales — Same Philosophy as Principle #14.** Once a `SalesOrder` transitions to `Dispatched`, cancellation is permanently unavailable. A dispatched sale can only be unwound via an explicit, separate `SalesReturn` record (new movement type `SaleReturn`, positive quantity, referencing the original sale).

**A7. Purchases Have No "Loss" Concept at Intake (Deliberately Deferred).** A purchase from a supplier is modeled as a single point-in-time receipt at the destination warehouse — there is no transit leg in this v1 scope.

**A8. Policies Are the ONLY Home for Permission/Role Logic — System-Wide, Not Just This Addendum, and Permanent Once Correct.** This principle governs every Policy class in the entire system. All permission/role logic must live inside the relevant Policy class's method body, and nowhere else. Once a Policy method correctly and completely encodes the permission/role logic for its ability, it is treated as a closed, frozen contract — exactly like `reservedQuantity()` under Principle #13.

**A9. `[Added v12]` Shared Filter Architecture for Admin Review.** System-Admin-only warehouse and period filters are built once as static factory methods on `App\Filament\Support\Filters\AdminReviewFilters` and reused across all resources requiring cross-warehouse, cross-period review.

---

## 📁 Section 1: Filament v5 Resource Directory Structure

Filament v5 uses a domain-oriented directory structure. Each resource is a thin class that delegates form, table, and infolist definitions to dedicated schema and table classes.

```
app/Filament/Resources/
├── Products/
│   ├── ProductResource.php
│   ├── Pages/
│   │   ├── ListProducts.php
│   │   ├── CreateProduct.php
│   │   ├── EditProduct.php
│   │   └── ViewProduct.php
│   ├── Schemas/
│   │   ├── ProductForm.php
│   │   └── ProductInfolist.php
│   └── Tables/
│       └── ProductsTable.php
│
├── TransferRequisitions/
│   ├── TransferRequisitionResource.php
│   ├── Pages/
│   │   ├── ListTransferRequisitions.php
│   │   ├── CreateTransferRequisition.php
│   │   ├── EditTransferRequisition.php
│   │   └── ViewTransferRequisition.php
│   ├── Schemas/
│   │   ├── TransferRequisitionForm.php
│   │   └── TransferRequisitionInfolist.php
│   └── Tables/
│       └── TransferRequisitionsTable.php
│
├── DirectTransfers/
│   ├── DirectTransferResource.php
│   ├── Pages/
│   │   └── CreateDirectTransfer.php
│   └── Schemas/
│       └── DirectTransferForm.php
│
├── InTransits/
│   ├── InTransitResource.php
│   ├── Pages/
│   │   └── ListInTransits.php
│   ├── Schemas/
│   │   └── InTransitInfolist.php
│   └── Tables/
│       └── InTransitsTable.php
│
├── StockMovements/
│   ├── StockMovementResource.php
│   ├── Pages/
│   │   └── ListStockMovements.php
│   ├── Schemas/
│   │   └── StockMovementInfolist.php
│   └── Tables/
│       └── StockMovementsTable.php
│
├── LossLedgers/
│   ├── LossLedgerResource.php
│   ├── Pages/
│   │   └── ListLossLedgers.php
│   ├── Schemas/
│   │   └── LossLedgerInfolist.php
│   └── Tables/
│       └── LossLedgersTable.php
│
├── Warehouses/
│   ├── WarehouseResource.php
│   ├── Pages/
│   │   ├── ListWarehouses.php
│   │   ├── CreateWarehouse.php
│   │   └── EditWarehouse.php
│   ├── Schemas/
│   │   └── WarehouseForm.php
│   └── Tables/
│       └── WarehousesTable.php
│
├── Users/
│   ├── UserResource.php
│   ├── Pages/
│   │   ├── ListUsers.php
│   │   ├── CreateUser.php
│   │   └── EditUser.php
│   ├── Schemas/
│   │   └── UserForm.php
│   └── Tables/
│       └── UsersTable.php
│
├── PurchaseOrders/                                    # [NEW v12]
│   ├── PurchaseOrderResource.php
│   ├── Pages/
│   │   ├── ListPurchaseOrders.php
│   │   ├── CreatePurchaseOrder.php
│   │   ├── EditPurchaseOrder.php
│   │   └── ViewPurchaseOrder.php
│   ├── Schemas/
│   │   ├── PurchaseOrderForm.php
│   │   └── PurchaseOrderInfolist.php
│   └── Tables/
│       └── PurchaseOrdersTable.php
│
├── SalesOrders/                                       # [NEW v12]
│   ├── SalesOrderResource.php
│   ├── Pages/
│   │   ├── ListSalesOrders.php
│   │   ├── CreateSalesOrder.php
│   │   ├── EditSalesOrder.php
│   │   └── ViewSalesOrder.php
│   ├── Schemas/
│   │   ├── SalesOrderForm.php
│   │   └── SalesOrderInfolist.php
│   └── Tables/
│       └── SalesOrdersTable.php
│
├── Suppliers/                                         # [NEW v12]
│   ├── SupplierResource.php
│   ├── Pages/
│   │   ├── ListSuppliers.php
│   │   ├── CreateSupplier.php
│   │   └── EditSupplier.php
│   └── Schemas/
│       └── SupplierForm.php
│
└── Customers/                                         # [NEW v12]
    ├── CustomerResource.php
    ├── Pages/
    │   ├── ListCustomers.php
    │   ├── CreateCustomer.php
    │   └── EditCustomer.php
    └── Schemas/
        └── CustomerForm.php
```

### Thin Resource Class Pattern (Filament v5)

```php
namespace App\Filament\Resources\Products;

use App\Filament\Resources\Products\Pages\CreateProduct;
use App\Filament\Resources\Products\Pages\EditProduct;
use App\Filament\Resources\Products\Pages\ListProducts;
use App\Filament\Resources\Products\Pages\ViewProduct;
use App\Filament\Resources\Products\Schemas\ProductForm;
use App\Filament\Resources\Products\Schemas\ProductInfolist;
use App\Filament\Resources\Products\Tables\ProductsTable;
use App\Models\ProductVariant;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class ProductResource extends Resource
{
    protected static ?string $model = ProductVariant::class;

    protected static string | \UnitEnum | null $navigationGroup = 'CATALOG';

    protected static ?int $navigationSort = 1;

    protected static ?string $recordTitleAttribute = 'sku';

    public static function form(Schema $schema): Schema
    {
        return ProductForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return ProductsTable::configure($table);
    }

    public static function infolist(Schema $schema): Schema
    {
        return ProductInfolist::configure($schema);
    }

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()
            ->with(['product', 'unitConversions', 'currentPrice']);
    }

    public static function getRecordRouteBindingEloquentQuery(): Builder
    {
        return parent::getRecordRouteBindingEloquentQuery()
            ->withoutGlobalScopes([SoftDeletingScope::class]);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListProducts::route('/'),
            'create' => CreateProduct::route('/create'),
            'view'   => ViewProduct::route('/{record}'),
            'edit'   => EditProduct::route('/{record}/edit'),
        ];
    }
}
```

### Navigation Group Registration (Centralized)

Navigation groups registered centrally in `AdminPanelProvider` via `->navigationGroups()` with fixed display order:

```php
->navigationGroups([
    NavigationGroup::make('CATALOG')
        ->icon(Heroicon::CubeTransparent),
    NavigationGroup::make('OPERATIONS')
        ->icon(Heroicon::OutlinedRectangleStack),
    NavigationGroup::make('PURCHASING')
        ->icon(Heroicon::OutlinedShoppingCart),
    NavigationGroup::make('SALES')
        ->icon(Heroicon::OutlinedBanknotes),
    NavigationGroup::make('AUDIT LEDGERS')
        ->icon(Heroicon::QueueList),
    NavigationGroup::make('SYSTEM ADMIN')
        ->icon(Heroicon::BuildingOffice),
])
```

### Schema `configure()` Contract

All schema and table classes expose a static `configure()` method:

```php
class ProductForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([...]);
    }
}

class ProductsTable
{
    public static function configure(Table $table): Table
    {
        return $table->columns([...])->filters([...])->recordActions([...]);
    }
}
```

---

## 🗄️ Section 2: Complete Database Schema (20 Tables)

### 1. products

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| name | string | No | — |
| category | string | Yes | — |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

### 2. product_variants

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| product_id | FK → products.id (cascadeOnDelete) | No | — |
| sku | string, unique | No | — |
| barcode | string, unique | Yes | — |
| name | string | No | — |
| base_unit_name | string | No | — |
| reorder_point | integer | No | 0 |
| attributes | json | Yes | — |
| images | json | Yes | — |
| is_active | boolean | No | true |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(product_id, sku)`

### 3. product_variant_prices

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| product_variant_id | FK → product_variants.id (cascadeOnDelete) | No | — |
| cost_price | decimal(15,4) | No | 0.0000 |
| sale_price | decimal(15,4) | No | 0.0000 |
| effective_from | timestamp | No | current time |
| is_current | boolean | No | true |
| set_by | FK → users.id (nullOnDelete) | Yes | — |
| notes | text | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(product_variant_id, effective_from)`
Constraints: At most one `is_current = true` row per `product_variant_id`.

### 4. product_variant_unit_conversions

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| product_variant_id | FK → product_variants.id (cascadeOnDelete) | No | — |
| unit_name | string | No | — |
| base_unit_ratio | integer | No | — |
| is_default_purchase | boolean | No | false |
| is_default_transfer | boolean | No | false |
| created_at / updated_at | timestamp | Yes | — |

Indexes: unique on `(product_variant_id, unit_name)`

### 5. warehouses

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| code | string, unique | No | — |
| name | string | No | — |
| location | string | Yes | — |
| is_active | boolean | No | true |
| created_at / updated_at | timestamp | Yes | — |

### 6. stock_movements

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| type | string | No | — |
| quantity | integer | No | — |
| unit_name_used | string | No | — |
| unit_ratio_used | integer | No | 1 |
| related_movement_id | FK → stock_movements.id (nullOnDelete) | Yes | — |
| reference_type | string | Yes | — |
| reference_id | string | Yes | — |
| reference_code | string | Yes | — |
| notes | text | Yes | — |
| created_by | FK → users.id (nullOnDelete) | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(product_variant_id, warehouse_id)`, `(reference_type, reference_id)`, `type`, `created_at`

### 7. transfer_requisitions

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| reference_code | string, unique | No | — |
| from_warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| to_warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| status | string | No | `TransferRequisitionStatus::Draft->value` |
| requested_by | FK → users.id | No | — |
| approved_by | FK → users.id | Yes | — |
| dispatched_by | FK → users.id | Yes | — |
| received_by | FK → users.id | Yes | — |
| requested_at | timestamp | Yes | — |
| approved_at | timestamp | Yes | — |
| dispatched_at | timestamp | Yes | — |
| completed_at | timestamp | Yes | — |
| notes | text | Yes | — |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `status`, `(from_warehouse_id, to_warehouse_id)`

**Key Implementation Notes:**
- Uses string column + PHP backed enum (`App\Enums\TransferRequisitionStatus`) instead of DB `enum()`.
- Both warehouse FKs use `restrictOnDelete`.
- Soft deletes enabled via `$table->softDeletes()`.

### 8. transfer_requisition_items

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| substitute_product_variant_id | FK → product_variants.id (restrictOnDelete) | Yes | — |
| requested_unit_name | string | No | — |
| requested_unit_ratio | integer | No | — |
| requested_qty | integer | No | — |
| requested_base_qty | integer | No | — |
| approved_unit_name | string | Yes | — |
| approved_unit_ratio | integer | Yes | — |
| approved_qty | integer | Yes | — |
| approved_base_qty | integer | Yes | — |
| shipped_base_qty | integer | No | 0 |
| received_good_base_qty | integer | No | 0 |
| received_damaged_base_qty | integer | No | 0 |
| received_qty | integer | No | 0 |
| notes | text | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

**Key Implementation Notes:**
- Both product_variant FKs use `restrictOnDelete`.
- `received_qty` has a default of 0 (not nullable).
- Stores both requested and approved quantities with unit conversion tracking for negotiated revisions.

Indexes: `transfer_requisition_id`

### 9. transfer_requisition_item_revisions

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_item_id | FK → transfer_requisition_items.id (cascadeOnDelete) | No | — |
| user_id | FK → users.id | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| substitute_product_variant_id | FK → product_variants.id (restrictOnDelete) | Yes | — |
| proposed_unit_name | string | No | — |
| proposed_unit_ratio | integer | No | — |
| proposed_qty | integer | No | — |
| proposed_base_qty | integer | No | — |
| negotiation_reason | text | Yes | — |
| side | string | No | — |
| status | string | No | pending |
| responds_to_revision_id | FK → transfer_requisition_item_revisions.id (nullOnDelete) | Yes | — |
| responded_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `transfer_requisition_item_id`, `(transfer_requisition_item_id, status)`

### 10. in_transits

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | No | — |
| transfer_requisition_item_id | FK → transfer_requisition_items.id (cascadeOnDelete) | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| dispatched_base_qty | integer | No | — |
| dispatched_at | timestamp | No | — |
| status | string | No | in_transit |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(transfer_requisition_id, status)`

### 11. loss_ledgers

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | No | — |
| transfer_requisition_item_id | FK → transfer_requisition_items.id (cascadeOnDelete) | Yes | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| lost_base_qty | integer | No | 0 |
| damaged_base_qty | integer | No | 0 |
| unit_cost_price | decimal(15,4) | No | — |
| total_financial_loss | decimal(15,4) | No | — |
| loss_category | string | No | shortfall |
| notes | text | Yes | — |
| recorded_by | FK → users.id | Yes | — |
| recorded_at | timestamp | No | current time |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(warehouse_id, recorded_at)`

**Key Implementation Notes:**
- `transfer_requisition_item_id` is nullable to allow loss recording for items not tied to a specific requisition item.
- `unit_cost_price` and `total_financial_loss` use `decimal(15,4)` for 4-decimal micro-pricing precision.
- `loss_category` defaults to 'shortfall'; other categories: damage, spoilage, theft, other.

### 12. users (altered)

| Column | Type | Nullable | Default |
|---|---|---|---|
| role | string | No | warehouse_staff |

### 13. user_warehouse (pivot)

| Column | Type | Nullable | Default |
|---|---|---|---|
| user_id | FK → users.id (cascadeOnDelete, part of composite PK) | No | — |
| warehouse_id | FK → warehouses.id (cascadeOnDelete, part of composite PK) | No | — |

Primary key: composite `(user_id, warehouse_id)`

### 14. suppliers `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| name | string | No | — |
| contact_person | string | Yes | — |
| phone | string | Yes | — |
| email | string | Yes | — |
| address | text | Yes | — |
| is_active | boolean | No | true |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

### 15. customers `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| name | string | No | — |
| contact_person | string | Yes | — |
| phone | string | Yes | — |
| email | string | Yes | — |
| address | text | Yes | — |
| is_active | boolean | No | true |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

### 16. purchase_orders `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| reference_code | string, unique | No | — |
| supplier_id | FK → suppliers.id (restrictOnDelete) | No | — |
| warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| status | string | No | `PurchaseOrderStatus::Draft->value` |
| update_cost_price | boolean | No | false |
| ordered_by | FK → users.id | No | — |
| received_by | FK → users.id | Yes | — |
| ordered_at | timestamp | Yes | — |
| received_at | timestamp | Yes | — |
| cancelled_at | timestamp | Yes | — |
| notes | text | Yes | — |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `status`, `(supplier_id, warehouse_id)`

**Key Implementation Notes:**
- String column + PHP backed enum (`App\Enums\PurchaseOrderStatus`).
- `supplier_id` and `warehouse_id` both `restrictOnDelete`.
- `update_cost_price`: if true, `receivePurchase()` will insert a new `is_current=true` `product_variant_prices` row per line item at receipt time.

### 17. purchase_order_items `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| purchase_order_id | FK → purchase_orders.id (cascadeOnDelete) | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| ordered_unit_name | string | No | — |
| ordered_unit_ratio | integer | No | — |
| ordered_qty | integer | No | — |
| ordered_base_qty | integer | No | — |
| unit_cost_price | decimal(15,4) | No | 0.0000 |
| received_base_qty | integer | No | 0 |
| notes | text | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `purchase_order_id`

**Key Implementation Notes:**
- `unit_cost_price` is captured **at order time**, decimal(15,4) for micro-pricing parity.
- No `substitute_product_variant_id` — purchases are not negotiated.

### 18. sales_orders `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| reference_code | string, unique | No | — |
| customer_id | FK → customers.id (restrictOnDelete) | No | — |
| warehouse_id | FK → warehouses.id (restrictOnDelete) | No | — |
| status | string | No | `SalesOrderStatus::Draft->value` |
| ordered_by | FK → users.id | No | — |
| dispatched_by | FK → users.id | Yes | — |
| ordered_at | timestamp | Yes | — |
| confirmed_at | timestamp | Yes | — |
| dispatched_at | timestamp | Yes | — |
| cancelled_at | timestamp | Yes | — |
| notes | text | Yes | — |
| deleted_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `status`, `(customer_id, warehouse_id)`

**Key Implementation Notes:**
- Lifecycle: `draft → confirmed → dispatched → completed / cancelled`.
- `cancelled` is **only legal pre-dispatch** — identical philosophy to Principle #14.

### 19. sales_order_items `[NEW v12]`

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| sales_order_id | FK → sales_orders.id (cascadeOnDelete) | No | — |
| product_variant_id | FK → product_variants.id (restrictOnDelete) | No | — |
| unit_name | string | No | — |
| unit_ratio | integer | No | — |
| qty | integer | No | — |
| base_qty | integer | No | — |
| unit_sale_price_snapshot | decimal(15,4) | No | 0.0000 |
| dispatched_base_qty | integer | No | 0 |
| notes | text | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `sales_order_id`

**Key Implementation Notes:**
- `unit_sale_price_snapshot` captured **at confirm-time**, never recalculated later.
- `dispatched_base_qty` supports partial dispatch.

### 20. stock_movement_idempotency_keys

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | No | — |
| payload_checksum | string(64) | No | — |
| resulting_item_states | json | No | — |
| created_at | timestamp | No | current time |

Indexes: unique on `(transfer_requisition_id, payload_checksum)`

### New StockMovementType Cases `[NEW v12]`

Add four new cases to the existing `StockMovementType` enum (no migration needed — `type` is already a plain string column):

- `Purchase` — positive quantity, fired on PO receipt
- `Sale` — negative quantity, fired on sales order dispatch
- `SaleReturn` — positive quantity, fired on a customer return against a dispatched sale
- `PurchaseReturn` — negative quantity, fired when returning stock to a supplier post-receipt

`reference_type` / `reference_id` / `reference_code` on these movements point back to `PurchaseOrder::class` / `SalesOrder::class` and their `id`/`reference_code`, exactly like transfer movements already do.

---

## 🛠️ Section 3: Model Layer

### ProductVariant Model

```php
namespace App\Models;

use App\Enums\TransferRequisitionStatus;
use App\Enums\SalesOrderStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Database\Eloquent\SoftDeletes;

class ProductVariant extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'product_id', 'sku', 'barcode', 'name', 'base_unit_name',
        'reorder_point', 'attributes', 'images', 'is_active',
    ];

    protected function casts(): array
    {
        return [
            'attributes'    => 'array',
            'images'        => 'array',
            'reorder_point' => 'integer',
            'is_active'     => 'boolean',
        ];
    }

    public function product(): BelongsTo
    {
        return $this->belongsTo(Product::class);
    }

    public function unitConversions(): HasMany
    {
        return $this->hasMany(ProductVariantUnitConversion::class);
    }

    public function prices(): HasMany
    {
        return $this->hasMany(ProductVariantPrice::class);
    }

    public function currentPrice(): HasOne
    {
        return $this->hasOne(ProductVariantPrice::class)->where('is_current', true);
    }

    public function stockMovements(): HasMany
    {
        return $this->hasMany(StockMovement::class);
    }

    public function requisitionItems(): HasMany
    {
        return $this->hasMany(TransferRequisitionItem::class);
    }

    public function isBelowReorderPoint(int $currentBaseQty): bool
    {
        return $currentBaseQty <= $this->reorder_point;
    }

    public function onHandQuantity(int $warehouseId): int
    {
        return (int) StockMovement::where('product_variant_id', $this->id)
            ->where('warehouse_id', $warehouseId)
            ->sum('quantity');
    }

    /**
     * `[FIX v11]` Reservation scope is intentionally and permanently bounded
     * to Confirmed status only. Once TransferRequisition transitions to
     * Dispatched, the reserved quantity is superseded by the TransitOut
     * stock_movement (already reflected in onHandQuantity()).
     *
     * Do NOT extend this query to include Dispatched or PartiallyReceived
     * statuses. Doing so would double-count stock that onHandQuantity()
     * has already deducted via the TransitOut movement.
     */
    public function reservedQuantity(int $warehouseId): int
    {
        return (int) TransferRequisitionItem::where('product_variant_id', $this->id)
            ->whereHas('transferRequisition', function ($query) use ($warehouseId) {
                $query->where('from_warehouse_id', $warehouseId)
                    ->where('status', TransferRequisitionStatus::Confirmed);
            })
            ->sum('approved_base_qty');
    }

    /**
     * `[Added v12 — Principle A5]` Deliberately SEPARATE from
     * reservedQuantity(), which is permanently scoped to Confirmed
     * transfer_requisitions only. Sales reservations are a distinct
     * concern with a distinct lifecycle and are summed here instead,
     * then combined in availableQuantity() below.
     */
    public function reservedForSalesQuantity(int $warehouseId): int
    {
        return (int) SalesOrderItem::where('product_variant_id', $this->id)
            ->whereHas('salesOrder', function ($query) use ($warehouseId) {
                $query->where('warehouse_id', $warehouseId)
                    ->where('status', SalesOrderStatus::Confirmed);
            })
            ->sum('base_qty');
    }

    /**
     * `[FIX v11.1/v12]` availableQuantity() now nets out BOTH transfer
     * reservations and sales reservations. This REPLACES the parent
     * blueprint's availableQuantity() body — reservedQuantity() itself
     * is untouched.
     */
    public function availableQuantity(int $warehouseId): int
    {
        return $this->onHandQuantity($warehouseId)
            - $this->reservedQuantity($warehouseId)
            - $this->reservedForSalesQuantity($warehouseId);
    }

    /**
     * `[Added v12]` Batched sibling of availableQuantity(), for any context
     * that needs the figure for MULTIPLE variants against ONE warehouse at
     * once (e.g. every line item on a single sales order's dispatch modal).
     * Issues exactly 3 queries total regardless of how many variant IDs are
     * passed, instead of 3 queries PER variant via the instance method.
     *
     * This mirrors the exact upgrade path the parent blueprint's own
     * LowStockAlertsWidget accepted-risk note already prescribes.
     *
     * Returns [product_variant_id => availableQuantity] for the given
     * warehouse. Variant IDs with no movements/reservations at all correctly
     * return 0, not an array-key-missing gap.
     */
    public static function batchAvailableQuantity(array $variantIds, int $warehouseId): array
    {
        if (empty($variantIds)) {
            return [];
        }

        $onHand = StockMovement::whereIn('product_variant_id', $variantIds)
            ->where('warehouse_id', $warehouseId)
            ->selectRaw('product_variant_id, SUM(quantity) as total')
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id');

        $reservedTransfers = TransferRequisitionItem::whereIn('product_variant_id', $variantIds)
            ->whereHas('transferRequisition', function ($query) use ($warehouseId) {
                $query->where('from_warehouse_id', $warehouseId)
                    ->where('status', TransferRequisitionStatus::Confirmed);
            })
            ->selectRaw('product_variant_id, SUM(approved_base_qty) as total')
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id');

        $reservedSales = SalesOrderItem::whereIn('product_variant_id', $variantIds)
            ->whereHas('salesOrder', function ($query) use ($warehouseId) {
                $query->where('warehouse_id', $warehouseId)
                    ->where('status', SalesOrderStatus::Confirmed);
            })
            ->selectRaw('product_variant_id, SUM(base_qty) as total')
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id');

        return collect($variantIds)->mapWithKeys(function ($id) use ($onHand, $reservedTransfers, $reservedSales) {
            $available = (int) ($onHand[$id] ?? 0)
                - (int) ($reservedTransfers[$id] ?? 0)
                - (int) ($reservedSales[$id] ?? 0);

            return [$id => $available];
        })->all();
    }
}
```

### ProductObserver

```php
namespace App\Observers;

use App\Models\Product;
use Exception;

class ProductObserver
{
    public function deleting(Product $product): void
    {
        if ($product->isForceDeleting()) {
            return;
        }

        $activeVariants = $product->variants()->whereNull('deleted_at')->count();

        if ($activeVariants > 0) {
            throw new Exception(
                "Cannot soft-delete Product #{$product->id}: {$activeVariants} ".
                "active variant(s) must be trashed or reassigned first."
            );
        }
    }
}
```

Register in `AppServiceProvider::boot()`:

```php
Product::observe(ProductObserver::class);
```

### LossLedger Model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class LossLedger extends Model
{
    use HasFactory;

    protected $fillable = [
        'transfer_requisition_id', 'transfer_requisition_item_id',
        'product_variant_id', 'warehouse_id', 'lost_base_qty',
        'damaged_base_qty', 'unit_cost_price', 'total_financial_loss',
        'loss_category', 'recorded_by', 'recorded_at',
    ];

    protected function casts(): array
    {
        return [
            'unit_cost_price'      => 'decimal:4',
            'total_financial_loss' => 'decimal:4',
            'recorded_at'          => 'datetime',
        ];
    }

    public function transferRequisition(): BelongsTo
    {
        return $this->belongsTo(TransferRequisition::class);
    }

    public function transferRequisitionItem(): BelongsTo
    {
        return $this->belongsTo(TransferRequisitionItem::class);
    }

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function warehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class);
    }

    /**
     * `[FIX v11]` Snapshots the variant's CURRENT cost price at the moment
     * loss/intake is processed — not the price at time of original
     * dispatch. This is a deliberate design choice.
     *
     * Uses the null-safe operator (?->) rather than a bare property chain,
     * because `currentPrice` itself can be null (no is_current=true row
     * exists for this variant).
     *
     * Callers should eager-load the `currentPrice` relation on $variant
     * before calling this method to avoid an N+1 query per loss row.
     */
    public static function snapshotUnitCostFrom(ProductVariant $variant): string
    {
        return (string) ($variant->currentPrice?->cost_price ?? '0.0000');
    }

    /**
     * `[Added v12]` Computes total financial loss using bcmath for
     * 4-decimal micro-pricing precision.
     */
    public static function calculateTotalFinancialLoss(string $unitCost, int $quantity): string
    {
        return bcmul((string) $quantity, $unitCost, 4);
    }
}
```

### TransferRequisition Model

```php
namespace App\Models;

use App\Enums\TransferRequisitionStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class TransferRequisition extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'reference_code', 'from_warehouse_id', 'to_warehouse_id', 'status',
        'requested_by', 'approved_by', 'dispatched_by', 'received_by',
        'requested_at', 'approved_at', 'dispatched_at', 'completed_at', 'notes',
    ];

    protected function casts(): array
    {
        return ['status' => TransferRequisitionStatus::class];
    }

    public function fromWarehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class, 'from_warehouse_id');
    }

    public function toWarehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class, 'to_warehouse_id');
    }

    public function requestedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'requested_by');
    }

    public function approvedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'approved_by');
    }

    public function dispatchedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'dispatched_by');
    }

    public function receivedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'received_by');
    }

    public function items(): HasMany
    {
        return $this->hasMany(TransferRequisitionItem::class);
    }

    public function inTransits(): HasMany
    {
        return $this->hasMany(InTransit::class);
    }

    public function lossLedgers(): HasMany
    {
        return $this->hasMany(LossLedger::class);
    }
}
```

### TransferRequisitionItem Model

```php
namespace App\Models;

use App\Enums\RevisionStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class TransferRequisitionItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'transfer_requisition_id', 'product_variant_id', 'substitute_product_variant_id',
        'requested_unit_name', 'requested_unit_ratio', 'requested_qty', 'requested_base_qty',
        'approved_unit_name', 'approved_unit_ratio', 'approved_qty', 'approved_base_qty',
        'shipped_base_qty', 'received_good_base_qty', 'received_damaged_base_qty',
        'received_qty', 'notes',
    ];

    public function transferRequisition(): BelongsTo
    {
        return $this->belongsTo(TransferRequisition::class);
    }

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class, 'product_variant_id');
    }

    public function substituteProductVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class, 'substitute_product_variant_id');
    }

    public function revisions(): HasMany
    {
        return $this->hasMany(TransferRequisitionItemRevision::class);
    }

    public function negotiationHistory(): HasMany
    {
        return $this->revisions()->orderBy('created_at')->orderBy('id');
    }

    public function pendingRevision(): HasMany
    {
        return $this->revisions()->where('status', RevisionStatus::Pending);
    }

    public function inTransits(): HasMany
    {
        return $this->hasMany(InTransit::class);
    }

    public function lossLedgers(): HasMany
    {
        return $this->hasMany(LossLedger::class);
    }

    public function outstandingBaseQty(): int
    {
        $base = $this->approved_base_qty ?? $this->requested_base_qty;
        return max(0, $base - ($this->shipped_base_qty + $this->received_good_base_qty + $this->received_damaged_base_qty));
    }
}
```

### Supplier & Customer Models `[NEW v12]`

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Supplier extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = ['name', 'contact_person', 'phone', 'email', 'address', 'is_active'];

    protected function casts(): array
    {
        return ['is_active' => 'boolean'];
    }

    public function purchaseOrders(): HasMany
    {
        return $this->hasMany(PurchaseOrder::class);
    }
}

class Customer extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = ['name', 'contact_person', 'phone', 'email', 'address', 'is_active'];

    protected function casts(): array
    {
        return ['is_active' => 'boolean'];
    }

    public function salesOrders(): HasMany
    {
        return $this->hasMany(SalesOrder::class);
    }
}
```

### PurchaseOrder & PurchaseOrderItem Models `[NEW v12]`

```php
namespace App\Models;

use App\Enums\PurchaseOrderStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class PurchaseOrder extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'reference_code', 'supplier_id', 'warehouse_id', 'status',
        'update_cost_price', 'ordered_by', 'received_by',
        'ordered_at', 'received_at', 'cancelled_at', 'notes',
    ];

    protected function casts(): array
    {
        return [
            'status' => PurchaseOrderStatus::class,
            'update_cost_price' => 'boolean',
        ];
    }

    public function supplier(): BelongsTo { return $this->belongsTo(Supplier::class); }
    public function warehouse(): BelongsTo { return $this->belongsTo(Warehouse::class); }
    public function orderedBy(): BelongsTo { return $this->belongsTo(User::class, 'ordered_by'); }
    public function receivedBy(): BelongsTo { return $this->belongsTo(User::class, 'received_by'); }
    public function items(): HasMany { return $this->hasMany(PurchaseOrderItem::class); }
}

class PurchaseOrderItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'purchase_order_id', 'product_variant_id', 'ordered_unit_name',
        'ordered_unit_ratio', 'ordered_qty', 'ordered_base_qty',
        'unit_cost_price', 'received_base_qty', 'notes',
    ];

    protected function casts(): array
    {
        return ['unit_cost_price' => 'decimal:4'];
    }

    public function purchaseOrder(): BelongsTo { return $this->belongsTo(PurchaseOrder::class); }
    public function productVariant(): BelongsTo { return $this->belongsTo(ProductVariant::class); }

    public function outstandingBaseQty(): int
    {
        return max(0, $this->ordered_base_qty - $this->received_base_qty);
    }
}
```

### SalesOrder & SalesOrderItem Models `[NEW v12]`

```php
namespace App\Models;

use App\Enums\SalesOrderStatus;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class SalesOrder extends Model
{
    use HasFactory, SoftDeletes;

    protected $fillable = [
        'reference_code', 'customer_id', 'warehouse_id', 'status',
        'ordered_by', 'dispatched_by', 'ordered_at', 'confirmed_at',
        'dispatched_at', 'cancelled_at', 'notes',
    ];

    protected function casts(): array
    {
        return ['status' => SalesOrderStatus::class];
    }

    public function customer(): BelongsTo { return $this->belongsTo(Customer::class); }
    public function warehouse(): BelongsTo { return $this->belongsTo(Warehouse::class); }
    public function orderedBy(): BelongsTo { return $this->belongsTo(User::class, 'ordered_by'); }
    public function dispatchedBy(): BelongsTo { return $this->belongsTo(User::class, 'dispatched_by'); }
    public function items(): HasMany { return $this->hasMany(SalesOrderItem::class); }
}

class SalesOrderItem extends Model
{
    use HasFactory;

    protected $fillable = [
        'sales_order_id', 'product_variant_id', 'unit_name', 'unit_ratio',
        'qty', 'base_qty', 'unit_sale_price_snapshot', 'dispatched_base_qty', 'notes',
    ];

    protected function casts(): array
    {
        return ['unit_sale_price_snapshot' => 'decimal:4'];
    }

    public function salesOrder(): BelongsTo { return $this->belongsTo(SalesOrder::class); }
    public function productVariant(): BelongsTo { return $this->belongsTo(ProductVariant::class); }

    public function outstandingBaseQty(): int
    {
        return max(0, $this->base_qty - $this->dispatched_base_qty);
    }

    public function lineTotal(): string
    {
        return bcmul((string) $this->dispatched_base_qty, (string) $this->unit_sale_price_snapshot, 4);
    }
}
```

---

## 🏷️ Section 4: Enums

### TransferRequisitionStatus

```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum TransferRequisitionStatus: string implements HasColor, HasIcon, HasLabel
{
    case Draft = 'draft';
    case Requested = 'requested';
    case UnderReviewFulfiller = 'under_review_fulfiller';
    case UnderReviewRequestor = 'under_review_requestor';
    case Confirmed = 'confirmed';
    case Dispatched = 'dispatched';
    case PartiallyReceived = 'partially_received';
    case Completed = 'completed';
    case ClosedWithLoss = 'closed_with_loss';
    case Cancelled = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft => __('Draft'),
            self::Requested => __('Requested'),
            self::UnderReviewFulfiller => __('Under review (fulfiller)'),
            self::UnderReviewRequestor => __('Under review (requestor)'),
            self::Confirmed => __('Confirmed'),
            self::Dispatched => __('Dispatched'),
            self::PartiallyReceived => __('Partially received'),
            self::Completed => __('Completed'),
            self::ClosedWithLoss => __('Closed with loss'),
            self::Cancelled => __('Cancelled'),
        };
    }

    public function getColor(): string|array|null
    {
        return match ($this) {
            self::Draft => 'gray',
            self::Requested => 'info',
            self::UnderReviewFulfiller, self::UnderReviewRequestor => 'warning',
            self::Confirmed => 'primary',
            self::Dispatched => 'info',
            self::PartiallyReceived => 'warning',
            self::Completed => 'success',
            self::ClosedWithLoss => 'danger',
            self::Cancelled => 'gray',
        };
    }

    public function getIcon(): string|BackedEnum|null
    {
        return match ($this) {
            self::Draft => Heroicon::PaperAirplane,
            self::Requested => Heroicon::PaperAirplane,
            self::UnderReviewFulfiller, self::UnderReviewRequestor => Heroicon::ChatBubbleLeftRight,
            self::Confirmed => Heroicon::CheckCircle,
            self::Dispatched => Heroicon::Truck,
            self::PartiallyReceived => Heroicon::ArchiveBoxArrowDown,
            self::Completed => Heroicon::CheckBadge,
            self::ClosedWithLoss => Heroicon::ExclamationTriangle,
            self::Cancelled => Heroicon::XCircle,
        };
    }
}
```

### PurchaseOrderStatus `[NEW v12]`

```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum PurchaseOrderStatus: string implements HasColor, HasIcon, HasLabel
{
    case Draft = 'draft';
    case Ordered = 'ordered';
    case PartiallyReceived = 'partially_received';
    case Completed = 'completed';
    case Cancelled = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft => __('Draft'),
            self::Ordered => __('Ordered'),
            self::PartiallyReceived => __('Partially received'),
            self::Completed => __('Completed'),
            self::Cancelled => __('Cancelled'),
        };
    }

    public function getColor(): string|array|null
    {
        return match ($this) {
            self::Draft => 'gray',
            self::Ordered => 'info',
            self::PartiallyReceived => 'warning',
            self::Completed => 'success',
            self::Cancelled => 'gray',
        };
    }

    public function getIcon(): string|BackedEnum|null
    {
        return match ($this) {
            self::Draft => Heroicon::DocumentText,
            self::Ordered => Heroicon::PaperAirplane,
            self::PartiallyReceived => Heroicon::ArchiveBoxArrowDown,
            self::Completed => Heroicon::CheckBadge,
            self::Cancelled => Heroicon::XCircle,
        };
    }
}
```

### SalesOrderStatus `[NEW v12]`

```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum SalesOrderStatus: string implements HasColor, HasIcon, HasLabel
{
    case Draft = 'draft';
    case Confirmed = 'confirmed';
    case PartiallyDispatched = 'partially_dispatched';
    case Dispatched = 'dispatched';
    case Completed = 'completed';
    case Cancelled = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft => __('Draft'),
            self::Confirmed => __('Confirmed'),
            self::PartiallyDispatched => __('Partially dispatched'),
            self::Dispatched => __('Dispatched'),
            self::Completed => __('Completed'),
            self::Cancelled => __('Cancelled'),
        };
    }

    public function getColor(): string|array|null
    {
        return match ($this) {
            self::Draft => 'gray',
            self::Confirmed => 'primary',
            self::PartiallyDispatched => 'warning',
            self::Dispatched => 'info',
            self::Completed => 'success',
            self::Cancelled => 'gray',
        };
    }

    public function getIcon(): string|BackedEnum|null
    {
        return match ($this) {
            self::Draft => Heroicon::DocumentText,
            self::Confirmed => Heroicon::CheckCircle,
            self::PartiallyDispatched => Heroicon::ArchiveBoxArrowDown,
            self::Dispatched => Heroicon::Truck,
            self::Completed => Heroicon::CheckBadge,
            self::Cancelled => Heroicon::XCircle,
        };
    }
}
```

### StockMovementType (extended) `[NEW v12]`

Add `Purchase`, `Sale`, `SaleReturn`, `PurchaseReturn` cases to the existing `StockMovementType` enum (same file, no new file).

---

## 🏭 Section 5: Model Factories

### SupplierFactory

```php
namespace Database\Factories;

use App\Models\Supplier;
use Illuminate\Database\Eloquent\Factories\Factory;

class SupplierFactory extends Factory
{
    protected $model = Supplier::class;

    public function definition(): array
    {
        return [
            'name'           => fake()->company(),
            'contact_person' => fake()->name(),
            'phone'          => fake()->phoneNumber(),
            'email'          => fake()->companyEmail(),
            'address'        => fake()->address(),
            'is_active'      => true,
        ];
    }

    public function inactive(): static
    {
        return $this->state(fn () => ['is_active' => false]);
    }
}
```

### CustomerFactory

```php
namespace Database\Factories;

use App\Models\Customer;
use Illuminate\Database\Eloquent\Factories\Factory;

class CustomerFactory extends Factory
{
    protected $model = Customer::class;

    public function definition(): array
    {
        return [
            'name'           => fake()->company(),
            'contact_person' => fake()->name(),
            'phone'          => fake()->phoneNumber(),
            'email'          => fake()->companyEmail(),
            'address'        => fake()->address(),
            'is_active'      => true,
        ];
    }

    public function inactive(): static
    {
        return $this->state(fn () => ['is_active' => false]);
    }
}
```

### PurchaseOrderFactory

```php
namespace Database\Factories;

use App\Enums\PurchaseOrderStatus;
use App\Models\PurchaseOrder;
use App\Models\Supplier;
use App\Models\User;
use App\Models\Warehouse;
use Illuminate\Database\Eloquent\Factories\Factory;

class PurchaseOrderFactory extends Factory
{
    protected $model = PurchaseOrder::class;

    public function definition(): array
    {
        return [
            'reference_code'    => 'PO-'.fake()->unique()->numerify('######'),
            'supplier_id'       => Supplier::factory(),
            'warehouse_id'      => Warehouse::factory(),
            'status'            => PurchaseOrderStatus::Draft,
            'update_cost_price' => false,
            'ordered_by'        => User::factory(),
        ];
    }

    public function ordered(): static
    {
        return $this->state(fn () => [
            'status'     => PurchaseOrderStatus::Ordered,
            'ordered_at' => now(),
        ]);
    }

    public function partiallyReceived(): static
    {
        return $this->state(fn () => [
            'status'     => PurchaseOrderStatus::PartiallyReceived,
            'ordered_at' => now()->subDay(),
        ]);
    }

    public function completed(): static
    {
        return $this->state(fn () => [
            'status'      => PurchaseOrderStatus::Completed,
            'ordered_at'  => now()->subDays(2),
            'received_at' => now(),
        ]);
    }

    public function cancelled(): static
    {
        return $this->state(fn () => [
            'status'       => PurchaseOrderStatus::Cancelled,
            'cancelled_at' => now(),
        ]);
    }

    public function withCostUpdate(): static
    {
        return $this->state(fn () => ['update_cost_price' => true]);
    }
}
```

### PurchaseOrderItemFactory

```php
namespace Database\Factories;

use App\Models\ProductVariant;
use App\Models\PurchaseOrder;
use App\Models\PurchaseOrderItem;
use Illuminate\Database\Eloquent\Factories\Factory;

class PurchaseOrderItemFactory extends Factory
{
    protected $model = PurchaseOrderItem::class;

    public function definition(): array
    {
        $qty  = fake()->numberBetween(1, 50);
        $unitRatio = 1;

        return [
            'purchase_order_id'  => PurchaseOrder::factory(),
            'product_variant_id' => ProductVariant::factory(),
            'ordered_unit_name'  => 'pcs',
            'ordered_unit_ratio' => $unitRatio,
            'ordered_qty'        => $qty,
            'ordered_base_qty'   => $qty * $unitRatio,
            'unit_cost_price'    => fake()->randomFloat(4, 1, 500),
            'received_base_qty'  => 0,
        ];
    }

    public function fullyReceived(): static
    {
        return $this->state(fn (array $attrs) => [
            'received_base_qty' => $attrs['ordered_base_qty'],
        ]);
    }

    public function partiallyReceived(int $receivedBaseQty): static
    {
        return $this->state(fn () => ['received_base_qty' => $receivedBaseQty]);
    }
}
```

### SalesOrderFactory

```php
namespace Database\Factories;

use App\Enums\SalesOrderStatus;
use App\Models\Customer;
use App\Models\SalesOrder;
use App\Models\User;
use App\Models\Warehouse;
use Illuminate\Database\Eloquent\Factories\Factory;

class SalesOrderFactory extends Factory
{
    protected $model = SalesOrder::class;

    public function definition(): array
    {
        return [
            'reference_code' => 'SO-'.fake()->unique()->numerify('######'),
            'customer_id'    => Customer::factory(),
            'warehouse_id'   => Warehouse::factory(),
            'status'         => SalesOrderStatus::Draft,
            'ordered_by'     => User::factory(),
        ];
    }

    public function confirmed(): static
    {
        return $this->state(fn () => [
            'status'       => SalesOrderStatus::Confirmed,
            'confirmed_at' => now(),
        ]);
    }

    public function partiallyDispatched(): static
    {
        return $this->state(fn () => [
            'status'        => SalesOrderStatus::PartiallyDispatched,
            'confirmed_at'  => now()->subDay(),
            'dispatched_at' => now(),
        ]);
    }

    public function completed(): static
    {
        return $this->state(fn () => [
            'status'        => SalesOrderStatus::Completed,
            'confirmed_at'  => now()->subDays(2),
            'dispatched_at' => now(),
        ]);
    }

    public function cancelled(): static
    {
        return $this->state(fn () => [
            'status'       => SalesOrderStatus::Cancelled,
            'cancelled_at' => now(),
        ]);
    }
}
```

### SalesOrderItemFactory

```php
namespace Database\Factories;

use App\Models\ProductVariant;
use App\Models\SalesOrder;
use App\Models\SalesOrderItem;
use Illuminate\Database\Eloquent\Factories\Factory;

class SalesOrderItemFactory extends Factory
{
    protected $model = SalesOrderItem::class;

    public function definition(): array
    {
        $qty = fake()->numberBetween(1, 30);
        $unitRatio = 1;

        return [
            'sales_order_id'           => SalesOrder::factory(),
            'product_variant_id'       => ProductVariant::factory(),
            'unit_name'                => 'pcs',
            'unit_ratio'               => $unitRatio,
            'qty'                      => $qty,
            'base_qty'                 => $qty * $unitRatio,
            'unit_sale_price_snapshot' => '0.0000',
            'dispatched_base_qty'      => 0,
        ];
    }

    public function dispatched(?int $dispatchedBaseQty = null): static
    {
        return $this->state(fn (array $attrs) => [
            'dispatched_base_qty' => $dispatchedBaseQty ?? $attrs['base_qty'],
        ]);
    }

    public function withSnapshotPrice(string $price): static
    {
        return $this->state(fn () => ['unit_sale_price_snapshot' => $price]);
    }
}
```

---

## ⚙️ Section 6: Transactional Service Layer

### 6A. Shared Over-Fulfillment Guard Trait `[NEW v12]`

```php
namespace App\Services\Concerns;

use Exception;

trait GuardsOutstandingQuantity
{
    protected function assertWithinOutstanding(object $item, int $incomingQty, string $verb, int $itemId): void
    {
        $remaining = $item->outstandingBaseQty();

        if ($incomingQty > $remaining) {
            throw new Exception(
                "Cannot {$verb} {$incomingQty} units for item #{$itemId}: only ".
                "{$remaining} units remain outstanding on this order."
            );
        }
    }
}
```

### 6B. InventoryService

```php
namespace App\Services;

use App\Enums\InTransitStatus;
use App\Enums\StockMovementType;
use App\Enums\TransferRequisitionStatus;
use App\Models\InTransit;
use App\Models\LossLedger;
use App\Models\ProductVariant;
use App\Models\StockMovement;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItem;
use App\Models\Warehouse;
use Exception;
use Illuminate\Support\Facades\DB;

class InventoryService
{
    public function recordMovement(
        int $productVariantId,
        int $warehouseId,
        StockMovementType $type,
        int $baseQuantity,
        ?string $unitName = null,
        int $unitRatio = 1,
        ?string $referenceType = null,
        ?string $referenceId = null,
        ?string $referenceCode = null,
        ?int $relatedMovementId = null,
        ?string $notes = null,
    ): StockMovement {
        if ($unitRatio < 1) {
            throw new Exception(
                "unit_ratio must be a positive integer >= 1, received: {$unitRatio}."
            );
        }

        return DB::transaction(function () use (
            $productVariantId, $warehouseId, $type, $baseQuantity,
            $unitName, $unitRatio, $referenceType, $referenceId,
            $referenceCode, $relatedMovementId, $notes,
        ) {
            $variant = ProductVariant::lockForUpdate()->findOrFail($productVariantId);
            $currentStock = $variant->onHandQuantity($warehouseId);

            if ($baseQuantity < 0 && ($currentStock + $baseQuantity) < 0) {
                throw new Exception(
                    "Insufficient stock for SKU {$variant->sku} at warehouse ID {$warehouseId}. ".
                    "Available: {$currentStock}, requested deduction: ".abs($baseQuantity).'.'
                );
            }

            return StockMovement::create([
                'product_variant_id'  => $productVariantId,
                'warehouse_id'        => $warehouseId,
                'type'                => $type,
                'quantity'            => $baseQuantity,
                'unit_name_used'      => $unitName ?? $variant->base_unit_name,
                'unit_ratio_used'     => $unitRatio,
                'related_movement_id' => $relatedMovementId,
                'reference_type'      => $referenceType,
                'reference_id'        => $referenceId,
                'reference_code'      => $referenceCode,
                'notes'               => $notes,
                'created_by'          => auth()->id(),
            ]);
        });
    }

    public function directTransfer(
        int $productVariantId,
        int $fromWarehouseId,
        int $toWarehouseId,
        int $baseQuantity,
        ?string $unitName = null,
        int $unitRatio = 1,
        ?string $referenceCode = null,
        ?string $notes = null,
    ): array {
        if ($fromWarehouseId === $toWarehouseId) {
            throw new Exception('Direct transfer origin and destination warehouses must differ.');
        }

        if ($baseQuantity <= 0) {
            throw new Exception('Direct transfer quantity must be a positive number of base units.');
        }

        if ($unitRatio < 1) {
            throw new Exception(
                "unit_ratio must be a positive integer >= 1, received: {$unitRatio}."
            );
        }

        return DB::transaction(function () use (
            $productVariantId, $fromWarehouseId, $toWarehouseId,
            $baseQuantity, $unitName, $unitRatio, $referenceCode, $notes,
        ) {
            $warehouseIds = collect([$fromWarehouseId, $toWarehouseId])->sort()->values();
            Warehouse::whereIn('id', $warehouseIds)->lockForUpdate()->get();

            $variant = ProductVariant::lockForUpdate()->findOrFail($productVariantId);
            $currentStock = $variant->onHandQuantity($fromWarehouseId);

            if ($currentStock < $baseQuantity) {
                throw new Exception(
                    "Insufficient stock for SKU {$variant->sku} at origin warehouse ID {$fromWarehouseId}. ".
                    "Available: {$currentStock}, requested: {$baseQuantity}."
                );
            }

            $resolvedUnitName = $unitName ?? $variant->base_unit_name;

            $outMovement = StockMovement::create([
                'product_variant_id' => $productVariantId,
                'warehouse_id'       => $fromWarehouseId,
                'type'               => StockMovementType::TransferOut,
                'quantity'           => -$baseQuantity,
                'unit_name_used'     => $resolvedUnitName,
                'unit_ratio_used'    => $unitRatio,
                'reference_code'     => $referenceCode,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);

            $inMovement = StockMovement::create([
                'product_variant_id'  => $productVariantId,
                'warehouse_id'        => $toWarehouseId,
                'type'                => StockMovementType::TransferIn,
                'quantity'            => $baseQuantity,
                'unit_name_used'      => $resolvedUnitName,
                'unit_ratio_used'     => $unitRatio,
                'related_movement_id' => $outMovement->id,
                'reference_code'      => $referenceCode,
                'notes'               => $notes,
                'created_by'          => auth()->id(),
            ]);

            $outMovement->update(['related_movement_id' => $inMovement->id]);

            return [$outMovement->fresh(), $inMovement];
        });
    }

    public function dispatchTransfer(int $requisitionId): void
    {
        DB::transaction(function () use ($requisitionId) {
            $requisition = TransferRequisition::with('items.productVariant')
                ->lockForUpdate()
                ->findOrFail($requisitionId);

            if ($requisition->status !== TransferRequisitionStatus::Confirmed) {
                throw new Exception(
                    "Requisition must be confirmed before dispatch. Current status: {$requisition->status->value}."
                );
            }

            foreach ($requisition->items as $item) {
                if ($item->approved_base_qty === null) {
                    throw new Exception(
                        "Item #{$item->id} has no approved_base_qty; ConfirmAction must ".
                        "materialize approved_* before dispatch."
                    );
                }

                $actualVariantId = $item->substitute_product_variant_id ?? $item->product_variant_id;
                $dispatchQty     = $item->approved_base_qty;

                $variant = ProductVariant::lockForUpdate()->findOrFail($actualVariantId);

                if ($variant->onHandQuantity($requisition->from_warehouse_id) < $dispatchQty) {
                    throw new Exception(
                        "Insufficient stock for SKU {$variant->sku} at origin warehouse for ".
                        "requisition {$requisition->reference_code}."
                    );
                }

                StockMovement::create([
                    'product_variant_id' => $actualVariantId,
                    'warehouse_id'       => $requisition->from_warehouse_id,
                    'type'               => StockMovementType::TransitOut,
                    'quantity'           => -$dispatchQty,
                    'unit_name_used'     => $item->approved_unit_name,
                    'unit_ratio_used'    => $item->approved_unit_ratio,
                    'reference_type'     => TransferRequisition::class,
                    'reference_id'       => (string) $requisition->id,
                    'reference_code'     => $requisition->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                InTransit::create([
                    'transfer_requisition_id'      => $requisition->id,
                    'transfer_requisition_item_id' => $item->id,
                    'product_variant_id'           => $actualVariantId,
                    'dispatched_base_qty'          => $dispatchQty,
                    'dispatched_at'                => now(),
                    'status'                       => InTransitStatus::InTransit,
                ]);

                $item->update(['shipped_base_qty' => $dispatchQty]);
            }

            $requisition->update([
                'status'        => TransferRequisitionStatus::Dispatched,
                'dispatched_by' => auth()->id(),
                'dispatched_at' => now(),
            ]);
        });
    }

    public function scanToReceive(int $requisitionId, array $receivedItemsData): void
    {
        DB::transaction(function () use ($requisitionId, $receivedItemsData) {
            $requisition = TransferRequisition::with('items.productVariant')
                ->lockForUpdate()
                ->findOrFail($requisitionId);

            $allowed = [
                TransferRequisitionStatus::Dispatched,
                TransferRequisitionStatus::PartiallyReceived,
            ];

            if (! in_array($requisition->status, $allowed, true)) {
                throw new Exception(
                    "Requisition not in a receivable state. Current: {$requisition->status->value}."
                );
            }

            $isFirstScan = $requisition->status === TransferRequisitionStatus::Dispatched;
            $payloadChecksum = hash('sha256', json_encode($receivedItemsData));

            foreach ($requisition->items as $item) {
                $alreadyReceived = $item->received_good_base_qty + $item->received_damaged_base_qty;
                $expectedBase    = $item->shipped_base_qty;

                if ($alreadyReceived >= $expectedBase) {
                    continue;
                }

                $ratio = $item->approved_unit_ratio;

                if (! isset($receivedItemsData[$item->id])) {
                    if (! $isFirstScan) {
                        continue;
                    }
                    $goodBase     = 0;
                    $damagedBase  = 0;
                    $lostBase     = $expectedBase - $alreadyReceived;
                    $lossCategory = 'omitted_from_intake';
                } else {
                    $entry        = $receivedItemsData[$item->id];
                    $incomingGood = ($entry['good_qty']    ?? 0) * $ratio;
                    $incomingDmg  = ($entry['damaged_qty'] ?? 0) * $ratio;

                    $goodBase     = $item->received_good_base_qty    + $incomingGood;
                    $damagedBase  = $item->received_damaged_base_qty + $incomingDmg;
                    $lostBase     = max(0, $expectedBase - ($goodBase + $damagedBase));
                    $lossCategory = $entry['loss_category'] ?? 'shortfall';
                }

                $wouldChangeGood    = $goodBase    !== $item->received_good_base_qty;
                $wouldChangeDamaged = $damagedBase !== $item->received_damaged_base_qty;

                if (! $wouldChangeGood && ! $wouldChangeDamaged) {
                    continue;
                }

                $newlyReceivedGood    = $goodBase    - $item->received_good_base_qty;
                $newlyReceivedDamaged = $damagedBase - $item->received_damaged_base_qty;

                if ($newlyReceivedGood > 0) {
                    StockMovement::create([
                        'product_variant_id' => $item->substitute_product_variant_id ?? $item->product_variant_id,
                        'warehouse_id'       => $requisition->to_warehouse_id,
                        'type'               => StockMovementType::TransitIn,
                        'quantity'           => $newlyReceivedGood,
                        'unit_name_used'     => $item->approved_unit_name,
                        'unit_ratio_used'    => $ratio,
                        'reference_type'     => TransferRequisition::class,
                        'reference_id'       => (string) $requisition->id,
                        'reference_code'     => $requisition->reference_code,
                        'created_by'         => auth()->id(),
                    ]);
                }

                $itemFullyReceived = ($goodBase + $damagedBase) >= $expectedBase;
                $explicitLossDeclared = isset($receivedItemsData[$item->id]['loss_category']);
                $shouldRecordLoss = $newlyReceivedDamaged > 0
                    || ($lostBase > 0 && ($lossCategory === 'omitted_from_intake' || $itemFullyReceived || $explicitLossDeclared));

                if ($shouldRecordLoss) {
                    $actualVariantId = $item->substitute_product_variant_id ?? $item->product_variant_id;
                    $variant         = ProductVariant::with('currentPrice')->findOrFail($actualVariantId);
                    $unitCost        = LossLedger::snapshotUnitCostFrom($variant);
                    $totalLoss       = LossLedger::calculateTotalFinancialLoss($unitCost, $lostBase + $newlyReceivedDamaged);

                    LossLedger::create([
                        'transfer_requisition_id'      => $requisition->id,
                        'transfer_requisition_item_id' => $item->id,
                        'product_variant_id'           => $actualVariantId,
                        'warehouse_id'                 => $requisition->to_warehouse_id,
                        'lost_base_qty'                => $lostBase,
                        'damaged_base_qty'             => $newlyReceivedDamaged,
                        'unit_cost_price'              => $unitCost,
                        'total_financial_loss'         => $totalLoss,
                        'loss_category'                => $lossCategory,
                        'recorded_by'                  => auth()->id(),
                        'recorded_at'                  => now(),
                    ]);
                }

                $item->update([
                    'received_good_base_qty'    => $goodBase,
                    'received_damaged_base_qty' => $damagedBase,
                    'received_qty'              => $goodBase + $damagedBase,
                ]);

                InTransit::where('transfer_requisition_item_id', $item->id)
                    ->update(['status' => InTransitStatus::Cleared]);
            }

            try {
                DB::table('stock_movement_idempotency_keys')->insert([
                    'transfer_requisition_id' => $requisition->id,
                    'payload_checksum'        => $payloadChecksum,
                    'resulting_item_states'   => json_encode(
                        $requisition->items->pluck('received_qty', 'id')
                    ),
                    'created_at'              => now(),
                ]);
            } catch (\Illuminate\Database\QueryException $e) {
                report($e);
            }

            $allClosed = $requisition->items()
                ->whereRaw('(received_good_base_qty + received_damaged_base_qty) < shipped_base_qty')
                ->doesntExist();

            $hasAnyLoss = LossLedger::where('transfer_requisition_id', $requisition->id)->exists();

            $requisition->update([
                'status' => ! $allClosed
                    ? TransferRequisitionStatus::PartiallyReceived
                    : ($hasAnyLoss
                        ? TransferRequisitionStatus::ClosedWithLoss
                        : TransferRequisitionStatus::Completed),
                'received_by'  => auth()->id(),
                'completed_at' => $allClosed ? now() : null,
            ]);
        });
    }
}
```

### 6C. NegotiationService

```php
namespace App\Services;

use App\Enums\NegotiationSide;
use App\Enums\RevisionStatus;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItem;
use App\Models\TransferRequisitionItemRevision;
use App\Models\User;
use Exception;
use Illuminate\Support\Facades\DB;

class NegotiationService
{
    public function propose(
        TransferRequisitionItem $item,
        User $user,
        NegotiationSide $side,
        string $unitName,
        int $unitRatio,
        int $qty,
        ?int $substituteProductVariantId = null,
        ?string $reason = null,
        ?TransferRequisitionItemRevision $respondsTo = null,
    ): TransferRequisitionItemRevision {
        $attributes = [
            'user_id'                        => $user->id,
            'product_variant_id'             => $item->product_variant_id,
            'substitute_product_variant_id'  => $substituteProductVariantId,
            'proposed_unit_name'             => $unitName,
            'proposed_unit_ratio'            => $unitRatio,
            'proposed_qty'                   => $qty,
            'proposed_base_qty'              => $qty * $unitRatio,
            'negotiation_reason'             => $reason,
            'side'                           => $side,
        ];

        if ($respondsTo !== null) {
            return $respondsTo->counterWith($attributes);
        }

        return DB::transaction(function () use ($item, $attributes) {
            return TransferRequisitionItemRevision::create(array_merge($attributes, [
                'transfer_requisition_item_id' => $item->id,
                'status'                       => RevisionStatus::Pending,
            ]));
        });
    }

    public function accept(TransferRequisitionItemRevision $revision): void
    {
        if ($revision->status->isResolved()) {
            throw new Exception("Revision {$revision->id} is already resolved ({$revision->status->value}).");
        }

        $revision->accept();
    }

    public function reject(TransferRequisitionItemRevision $revision): void
    {
        if ($revision->status->isResolved()) {
            throw new Exception("Revision {$revision->id} is already resolved ({$revision->status->value}).");
        }

        $revision->reject();
    }

    public function counter(
        TransferRequisitionItemRevision $revision,
        User $user,
        string $unitName,
        int $unitRatio,
        int $qty,
        ?int $substituteProductVariantId = null,
        ?string $reason = null,
    ): TransferRequisitionItemRevision {
        if ($revision->status->isResolved()) {
            throw new Exception("Revision {$revision->id} is already resolved ({$revision->status->value}).");
        }

        return $revision->counterWith([
            'user_id'                       => $user->id,
            'product_variant_id'            => $revision->product_variant_id,
            'substitute_product_variant_id' => $substituteProductVariantId,
            'proposed_unit_name'            => $unitName,
            'proposed_unit_ratio'           => $unitRatio,
            'proposed_qty'                  => $qty,
            'proposed_base_qty'             => $qty * $unitRatio,
            'negotiation_reason'            => $reason,
            'side'                          => $revision->side->opposite(),
        ]);
    }

    public function materializeRequestedAsApproved(TransferRequisition $requisition): void
    {
        DB::transaction(function () use ($requisition) {
            $requisition->items()
                ->whereNull('approved_base_qty')
                ->each(function (TransferRequisitionItem $item) {
                    $item->update([
                        'approved_unit_name'  => $item->requested_unit_name,
                        'approved_unit_ratio' => $item->requested_unit_ratio,
                        'approved_qty'        => $item->requested_qty,
                        'approved_base_qty'   => $item->requested_base_qty,
                    ]);
                });
        });
    }
}
```

### 6D. PurchaseService `[NEW v12]`

```php
namespace App\Services;

use App\Enums\PurchaseOrderStatus;
use App\Enums\StockMovementType;
use App\Models\ProductVariant;
use App\Models\PurchaseOrder;
use App\Models\PurchaseOrderItem;
use App\Models\ProductVariantPrice;
use App\Models\StockMovement;
use Exception;
use Illuminate\Support\Facades\DB;

class PurchaseService
{
    use \App\Services\Concerns\GuardsOutstandingQuantity;

    public function orderPurchase(PurchaseOrder $po): void
    {
        if ($po->status !== PurchaseOrderStatus::Draft) {
            throw new Exception("Purchase order must be in draft to be ordered. Current: {$po->status->value}.");
        }

        if ($po->items()->count() === 0) {
            throw new Exception('Purchase order must have at least one line item.');
        }

        $po->update([
            'status'     => PurchaseOrderStatus::Ordered,
            'ordered_at' => now(),
        ]);
    }

    public function receivePurchase(int $purchaseOrderId, array $receivedItemsData): void
    {
        DB::transaction(function () use ($purchaseOrderId, $receivedItemsData) {
            $po = PurchaseOrder::with('items.productVariant.currentPrice')
                ->lockForUpdate()
                ->findOrFail($purchaseOrderId);

            $allowed = [PurchaseOrderStatus::Ordered, PurchaseOrderStatus::PartiallyReceived];

            if (! in_array($po->status, $allowed, true)) {
                throw new Exception("Purchase order not in a receivable state. Current: {$po->status->value}.");
            }

            foreach ($po->items as $item) {
                if (! isset($receivedItemsData[$item->id])) {
                    continue;
                }

                $incomingQty = (int) $receivedItemsData[$item->id];

                if ($incomingQty <= 0) {
                    continue;
                }

                $this->assertWithinOutstanding($item, $incomingQty, 'receive', $item->id);

                $variant = ProductVariant::with('currentPrice')->lockForUpdate()->findOrFail($item->product_variant_id);

                StockMovement::create([
                    'product_variant_id' => $item->product_variant_id,
                    'warehouse_id'       => $po->warehouse_id,
                    'type'               => StockMovementType::Purchase,
                    'quantity'           => $incomingQty,
                    'unit_name_used'     => $item->ordered_unit_name,
                    'unit_ratio_used'    => $item->ordered_unit_ratio,
                    'reference_type'     => PurchaseOrder::class,
                    'reference_id'       => (string) $po->id,
                    'reference_code'     => $po->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                if ($po->update_cost_price) {
                    $currentCost = $variant->currentPrice?->cost_price;

                    if ($currentCost === null || bccomp((string) $currentCost, (string) $item->unit_cost_price, 4) !== 0) {
                        ProductVariantPrice::where('product_variant_id', $variant->id)
                            ->where('is_current', true)
                            ->update(['is_current' => false]);

                        ProductVariantPrice::create([
                            'product_variant_id' => $variant->id,
                            'cost_price'         => $item->unit_cost_price,
                            'sale_price'         => $variant->currentPrice?->sale_price ?? '0.0000',
                            'effective_from'     => now(),
                            'is_current'         => true,
                            'set_by'             => auth()->id(),
                            'notes'              => "Auto-updated from PO {$po->reference_code}",
                        ]);
                    }
                }

                $item->update(['received_base_qty' => $item->received_base_qty + $incomingQty]);
            }

            $allReceived = $po->items()
                ->whereColumn('received_base_qty', '<', 'ordered_base_qty')
                ->doesntExist();

            $po->update([
                'status'      => $allReceived ? PurchaseOrderStatus::Completed : PurchaseOrderStatus::PartiallyReceived,
                'received_by' => auth()->id(),
                'received_at' => $allReceived ? now() : $po->received_at,
            ]);
        });
    }

    public function cancelPurchaseOrder(PurchaseOrder $po): void
    {
        if ($po->items()->where('received_base_qty', '>', 0)->exists()) {
            throw new Exception(
                'Cannot cancel a purchase order that has already received stock. '.
                'Use a return/adjustment instead.'
            );
        }

        $po->update([
            'status'       => PurchaseOrderStatus::Cancelled,
            'cancelled_at' => now(),
        ]);
    }
}
```

### 6E. SalesService `[NEW v12]`

```php
namespace App\Services;

use App\Enums\SalesOrderStatus;
use App\Enums\StockMovementType;
use App\Models\ProductVariant;
use App\Models\SalesOrder;
use App\Models\StockMovement;
use Exception;
use Illuminate\Support\Facades\DB;

class SalesService
{
    use \App\Services\Concerns\GuardsOutstandingQuantity;

    public function confirmSalesOrder(SalesOrder $order): void
    {
        if ($order->status !== SalesOrderStatus::Draft) {
            throw new Exception("Sales order must be in draft to confirm. Current: {$order->status->value}.");
        }

        DB::transaction(function () use ($order) {
            foreach ($order->items as $item) {
                $variant = ProductVariant::with('currentPrice')->findOrFail($item->product_variant_id);
                $item->update([
                    'unit_sale_price_snapshot' => $variant->currentPrice?->sale_price ?? '0.0000',
                ]);
            }

            $order->update([
                'status'       => SalesOrderStatus::Confirmed,
                'confirmed_at' => now(),
            ]);
        });
    }

    public function dispatchSale(int $salesOrderId, array $dispatchData): void
    {
        DB::transaction(function () use ($salesOrderId, $dispatchData) {
            $order = SalesOrder::with('items.productVariant')
                ->lockForUpdate()
                ->findOrFail($salesOrderId);

            $allowed = [SalesOrderStatus::Confirmed, SalesOrderStatus::PartiallyDispatched];

            if (! in_array($order->status, $allowed, true)) {
                throw new Exception("Sales order not in a dispatchable state. Current: {$order->status->value}.");
            }

            foreach ($order->items as $item) {
                if (! isset($dispatchData[$item->id])) {
                    continue;
                }

                $qty = (int) $dispatchData[$item->id];

                if ($qty <= 0) {
                    continue;
                }

                $this->assertWithinOutstanding($item, $qty, 'dispatch', $item->id);

                $variant = ProductVariant::lockForUpdate()->findOrFail($item->product_variant_id);
                $onHand  = $variant->onHandQuantity($order->warehouse_id);

                if ($onHand < $qty) {
                    throw new Exception(
                        "Insufficient stock for SKU {$variant->sku} at warehouse ID {$order->warehouse_id}. ".
                        "Available: {$onHand}, requested dispatch: {$qty}."
                    );
                }

                StockMovement::create([
                    'product_variant_id' => $item->product_variant_id,
                    'warehouse_id'       => $order->warehouse_id,
                    'type'               => StockMovementType::Sale,
                    'quantity'           => -$qty,
                    'unit_name_used'     => $item->unit_name,
                    'unit_ratio_used'    => $item->unit_ratio,
                    'reference_type'     => SalesOrder::class,
                    'reference_id'       => (string) $order->id,
                    'reference_code'     => $order->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                $item->update(['dispatched_base_qty' => $item->dispatched_base_qty + $qty]);
            }

            $allDispatched = $order->items()
                ->whereColumn('dispatched_base_qty', '<', 'base_qty')
                ->doesntExist();

            $order->update([
                'status'         => $allDispatched ? SalesOrderStatus::Completed : SalesOrderStatus::PartiallyDispatched,
                'dispatched_by'  => auth()->id(),
                'dispatched_at'  => $order->dispatched_at ?? now(),
            ]);
        });
    }

    public function cancelSalesOrder(SalesOrder $order): void
    {
        if (in_array($order->status, [SalesOrderStatus::PartiallyDispatched, SalesOrderStatus::Dispatched, SalesOrderStatus::Completed], true)) {
            throw new Exception(
                "Cannot cancel sales order once dispatch has begun. Current: {$order->status->value}. ".
                'Use a sales return instead.'
            );
        }

        $order->update([
            'status'       => SalesOrderStatus::Cancelled,
            'cancelled_at' => now(),
        ]);
    }

    public function recordSalesReturn(int $salesOrderItemId, int $returnedBaseQty, ?string $notes = null): StockMovement
    {
        if ($returnedBaseQty <= 0) {
            throw new Exception('Returned quantity must be a positive number of base units.');
        }

        return DB::transaction(function () use ($salesOrderItemId, $returnedBaseQty, $notes) {
            $item = \App\Models\SalesOrderItem::with('salesOrder')->lockForUpdate()->findOrFail($salesOrderItemId);

            if ($returnedBaseQty > $item->dispatched_base_qty) {
                throw new Exception(
                    "Cannot return {$returnedBaseQty} units: only {$item->dispatched_base_qty} ".
                    "units were dispatched for item #{$item->id}."
                );
            }

            return StockMovement::create([
                'product_variant_id' => $item->product_variant_id,
                'warehouse_id'       => $item->salesOrder->warehouse_id,
                'type'               => StockMovementType::SaleReturn,
                'quantity'           => $returnedBaseQty,
                'unit_name_used'     => $item->unit_name,
                'unit_ratio_used'    => $item->unit_ratio,
                'reference_type'     => \App\Models\SalesOrder::class,
                'reference_id'       => (string) $item->sales_order_id,
                'reference_code'     => $item->salesOrder->reference_code,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);
        });
    }
}
```

---

## 🎨 Section 7: Filament Resources — Master Specifications

### 7A. ProductResource

**Model:** `App\Models\ProductVariant` · **Navigation Group:** CATALOG · **Sort:** 1 · **Base Route:** `/admin/products`

#### ProductForm.php

```php
namespace App\Filament\Resources\Products\Schemas;

use Filament\Forms\Components\KeyValue;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;

class ProductForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Select::make('product_id')
                ->relationship('product', 'name')
                ->required()
                ->createOptionForm(fn (Schema $schema) => $schema->components([
                    TextInput::make('name')->required()->maxLength(255),
                    TextInput::make('category')->nullable(),
                ])),
            TextInput::make('sku')->required()->unique(ignoreRecord: true),
            TextInput::make('barcode')->nullable()->unique(ignoreRecord: true),
            TextInput::make('name')->required(),
            TextInput::make('base_unit_name')->required(),
            TextInput::make('reorder_point')->numeric()->default(0)->required(),
            KeyValue::make('attributes'),
            Toggle::make('is_active')->default(true),
        ]);
    }
}
```

#### ProductsTable.php

```php
namespace App\Filament\Resources\Products\Tables;

use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteAction;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\EditAction;
use Filament\Actions\RestoreAction;
use Filament\Actions\RestoreBulkAction;
use Filament\Tables\Columns\IconColumn;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;

class ProductsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('product.name')->searchable()->sortable(),
                TextColumn::make('sku')->fontFamily('mono')->copyable()->searchable(),
                TextColumn::make('barcode')->searchable(),
                TextColumn::make('name')->searchable(),
                TextColumn::make('base_unit_name')->badge(),
                TextColumn::make('currentPrice.sale_price')
                    ->money(config('app.currency')),
                TextColumn::make('reorder_point')->numeric(),
                IconColumn::make('is_active')->boolean(),
            ])
            ->filters([
                TernaryFilter::make('is_active'),
                SelectFilter::make('product_id')->relationship('product', 'name'),
                TrashedFilter::make(),
            ])
            ->recordActions([
                EditAction::make()->modalWidth(\Filament\Support\Enums\Width::Large),
                SetCurrentPriceAction::make(),
                EditProductFamilyAction::make(),
                ManageUnitConversionsAction::make(),
                QuickStockAdjustmentAction::make(),
                DeleteAction::make()->authorize('delete'),
                RestoreAction::make()->authorize('restore'),
            ])
            ->toolbarActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make()->authorize('deleteAny'),
                    RestoreBulkAction::make()->authorize('restoreAny'),
                ]),
            ]);
    }
}
```

#### ProductInfolist.php

```php
namespace App\Filament\Resources\Products\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Schema;

class ProductInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextEntry::make('product.name'),
            TextEntry::make('sku'),
            TextEntry::make('barcode'),
            TextEntry::make('currentPrice.cost_price')->money(config('app.currency')),
            TextEntry::make('currentPrice.sale_price')->money(config('app.currency')),
            RepeatableEntry::make('unitConversions'),
        ]);
    }
}
```

### 7B. TransferRequisitionResource

**Model:** `App\Models\TransferRequisition` · **Navigation Group:** OPERATIONS · **Sort:** 1 · **Base Route:** `/admin/transfer-requisitions`

#### Table Actions

```php
->recordActions([
    ViewAction::make(),

    EditAction::make()
        ->visible(fn ($record) => $record->status === 'draft')
        ->modalWidth(\Filament\Support\Enums\Width::Large),

    Action::make('submitRequest')
        ->label('SUBMIT REQUEST')
        ->icon(Heroicon::PaperAirplane)
        ->color('primary')
        ->visible(fn ($record) => $record->status === 'draft')
        ->action(function ($record) {
            $record->update([
                'status' => 'requested',
                'requested_at' => now(),
                'requested_by' => auth()->id(),
            ]);
        })
        ->requiresConfirmation(),

    Action::make('reviewNegotiate')
        ->label('REVIEW / NEGOTIATE')
        ->icon(Heroicon::ChatBubbleLeftRight)
        ->color('warning')
        ->visible(fn ($record) => in_array($record->status, ['requested', 'under_review_fulfiller', 'under_review_requestor']))
        ->url(fn ($record) => $record->getUrl('edit')),

    Action::make('acceptRevision')
        ->label('ACCEPT REVISION')
        ->icon(Heroicon::CheckCircle)
        ->color('success')
        ->visible(fn ($record) => in_array($record->status, ['under_review_fulfiller', 'under_review_requestor'])),

    Action::make('rejectRevision')
        ->label('REJECT REVISION')
        ->icon(Heroicon::XCircle)
        ->color('danger')
        ->visible(fn ($record) => in_array($record->status, ['under_review_fulfiller', 'under_review_requestor'])),

    Action::make('confirm')
        ->label('CONFIRM')
        ->icon(Heroicon::CheckBadge)
        ->color('primary')
        ->authorize('confirm')
        ->visible(fn ($record) => in_array($record->status, ['requested', 'under_review_fulfiller', 'under_review_requestor']))
        ->action(function ($record) {
            app(\App\Services\NegotiationService::class)
                ->materializeRequestedAsApproved($record);
            $record->update([
                'status' => 'confirmed',
                'approved_at' => now(),
                'approved_by' => auth()->id(),
            ]);
        })
        ->requiresConfirmation(),

    Action::make('dispatch')
        ->label('DISPATCH')
        ->icon(Heroicon::Truck)
        ->color('primary')
        ->authorize('dispatch')
        ->visible(fn ($record) => $record->status === 'confirmed'),

    Action::make('scanToReceive')
        ->label('SCAN TO RECEIVE')
        ->name('scanToReceive')
        ->icon(Heroicon::QrCode)
        ->color('success')
        ->authorize('receive')
        ->visible(fn ($record) => in_array($record->status, [TransferRequisitionStatus::Dispatched, TransferRequisitionStatus::PartiallyReceived]))
        ->url(fn ($record) => route('stn.scan', ['transferRequisition' => $record->id])),

    Action::make('recordLoss')
        ->label('RECORD LOSS')
        ->name('recordLoss')
        ->icon(Heroicon::ExclamationTriangle)
        ->color('danger')
        ->authorize('recordLoss')
        ->visible(fn ($record) => in_array($record->status, [TransferRequisitionStatus::Dispatched, TransferRequisitionStatus::PartiallyReceived]))
        ->modalWidth(\Filament\Support\Enums\Width::Large)
        ->schema([
            \Filament\Forms\Components\Select::make('product_variant_id')
                ->label('Product Variant')
                ->options(fn ($record) => $record->items->pluck('productVariant.name', 'product_variant_id')->toArray())
                ->required()
                ->searchable()
                ->preload()
                ->live(onBlur: true)
                ->afterStateUpdated(fn ($set, $get) => $set('total_financial_loss', null)),
            \Filament\Forms\Components\Select::make('loss_category')
                ->label('Loss Category')
                ->options([
                    'shortfall' => 'Shortfall',
                    'damage' => 'Damage',
                    'spoilage' => 'Spoilage',
                    'theft' => 'Theft',
                    'other' => 'Other',
                ])
                ->required(),
            \Filament\Forms\Components\TextInput::make('lost_base_qty')
                ->label('Lost Quantity (Base)')
                ->numeric()
                ->required()
                ->minValue(0)
                ->live(onBlur: true)
                ->afterStateUpdated(fn ($set, $get) => $set('total_financial_loss', null)),
            \Filament\Forms\Components\TextInput::make('damaged_base_qty')
                ->label('Damaged Quantity (Base)')
                ->numeric()
                ->default(0)
                ->minValue(0)
                ->live(onBlur: true)
                ->afterStateUpdated(fn ($set, $get) => $set('total_financial_loss', null)),
            \Filament\Forms\Components\TextInput::make('total_financial_loss')
                ->label('Total Financial Loss (Auto-calculated)')
                ->numeric()
                ->minValue(0)
                ->disabled()
                ->dehydrated(false),
            \Filament\Forms\Components\Textarea::make('notes')
                ->label('Notes')
                ->columnSpanFull(),
        ])
        ->action(function (array $data, $record) {
            $variant = \App\Models\ProductVariant::with('currentPrice')->find($data['product_variant_id']);
            $unitCost = \App\Models\LossLedger::snapshotUnitCostFrom($variant);
            $totalQty = (int) $data['lost_base_qty'] + (int) $data['damaged_base_qty'];
            $totalFinancialLoss = \App\Models\LossLedger::calculateTotalFinancialLoss($unitCost, $totalQty);
            $record->lossLedgers()->create([
                'transfer_requisition_item_id' => $record->items->where('product_variant_id', $data['product_variant_id'])->first()?->id,
                'product_variant_id' => $data['product_variant_id'],
                'warehouse_id' => $record->to_warehouse_id,
                'loss_category' => $data['loss_category'],
                'lost_base_qty' => $data['lost_base_qty'],
                'damaged_base_qty' => $data['damaged_base_qty'],
                'unit_cost_price' => $unitCost,
                'total_financial_loss' => $totalFinancialLoss,
                'notes' => $data['notes'],
                'recorded_by' => auth()->id(),
                'recorded_at' => now(),
            ]);
            \Filament\Notifications\Notification::make()
                ->title('Loss recorded')
                ->success()
                ->send();
        })
        ->requiresConfirmation(),

    Action::make('cancel')
        ->label('CANCEL')
        ->icon(Heroicon::XMark)
        ->color('danger')
        ->authorize('cancel')
        ->visible(fn ($record) => in_array($record->status, [
            'draft',
            'requested',
            'under_review_fulfiller',
            'under_review_requestor',
            'confirmed',
        ])),

    DeleteAction::make()
        ->authorize('delete')
        ->visible(fn ($record) => in_array($record->status, [
            'draft',
            'cancelled',
        ])),

    RestoreAction::make()
        ->authorize('restore'),

    ForceDeleteAction::make()
        ->authorize('forceDelete')
        ->visible(fn () => auth()->user()->isAdmin()),
])
->toolbarActions([
    BulkActionGroup::make([
        DeleteBulkAction::make()
            ->authorize('deleteAny'),
        RestoreBulkAction::make()
            ->authorize('restoreAny'),
        ForceDeleteBulkAction::make()
            ->authorize('forceDeleteAny'),
    ]),
]);
```

#### TransferRequisitionInfolist.php

```php
namespace App\Filament\Resources\TransferRequisitions\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Icons\Heroicon;

class TransferRequisitionInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema
            ->schema([
                Grid::make(3)
                    ->schema([
                        Section::make('REQUISITION PROFILE')
                            ->icon(Heroicon::DocumentText)
                            ->schema([
                                Grid::make(2)
                                    ->schema([
                                        TextEntry::make('reference_code')
                                            ->label('REFERENCE CODE')
                                            ->weight(FontWeight::Bold)
                                            ->size('lg')
                                            ->copyable()
                                            ->color('primary'),

                                        TextEntry::make('status')
                                            ->label('OPERATIONAL STATUS')
                                            ->badge(),

                                        TextEntry::make('fromWarehouse.name')
                                            ->label('ORIGIN BRANCH')
                                            ->icon(Heroicon::BuildingOffice),

                                        TextEntry::make('toWarehouse.name')
                                            ->label('RECEIVING BRANCH')
                                            ->icon(Heroicon::BuildingOffice2),
                                    ]),
                            ])
                            ->columnSpan(2),

                        Section::make('AUTHORIZATION SIGN-OFFS')
                            ->icon(Heroicon::ShieldCheck)
                            ->schema([
                                TextEntry::make('requestedBy.name')
                                    ->label('REQUESTED BY')
                                    ->icon(Heroicon::User)
                                    ->placeholder('System Initialized'),

                                TextEntry::make('approvedBy.name')
                                    ->label('APPROVED BY')
                                    ->icon(Heroicon::Check)
                                    ->placeholder('Pending Approval'),

                                TextEntry::make('dispatchedBy.name')
                                    ->label('DISPATCHED BY')
                                    ->icon(Heroicon::Truck)
                                    ->placeholder('Pending Dispatch'),

                                TextEntry::make('receivedBy.name')
                                    ->label('RECEIVED BY')
                                    ->icon(Heroicon::QrCode)
                                    ->placeholder('Pending Intake'),
                            ])
                            ->columnSpan(1),

                        Section::make('MATERIAL MANIFEST ITEMS')
                            ->icon(Heroicon::ClipboardDocumentList)
                            ->schema([
                                RepeatableEntry::make('items')
                                    ->label('')
                                    ->schema([
                                        Grid::make(6)
                                            ->schema([
                                                TextEntry::make('productVariant.sku')
                                                    ->label('ORIGINAL SKU')
                                                    ->weight(FontWeight::Bold)
                                                    ->columnSpan(1),

                                                TextEntry::make('substituteProductVariant.sku')
                                                    ->label('PROPOSED SUBSTITUTE')
                                                    ->badge()
                                                    ->color('warning')
                                                    ->placeholder('No Substitute')
                                                    ->columnSpan(1),

                                                TextEntry::make('requested_qty')
                                                    ->label('REQUESTED')
                                                    ->state(fn ($record) => "{$record->requested_qty} {$record->requested_unit_name}")
                                                    ->columnSpan(1),

                                                TextEntry::make('approved_qty')
                                                    ->label('APPROVED')
                                                    ->state(fn ($record) => $record->approved_qty
                                                        ? "{$record->approved_qty} {$record->approved_unit_name}"
                                                        : 'Pending Verification')
                                                    ->color(fn ($record) => $record->approved_qty !== $record->requested_qty ? 'warning' : 'gray')
                                                    ->columnSpan(1),

                                                TextEntry::make('approved_base_qty')
                                                    ->label('APPROVED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('shipped_base_qty')
                                                    ->label('SHIPPED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('received_good_base_qty')
                                                    ->label('RECEIVED GOOD (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('received_damaged_base_qty')
                                                    ->label('RECEIVED DAMAGED (BASE)')
                                                    ->numeric()
                                                    ->color('danger')
                                                    ->columnSpan(1),

                                                TextEntry::make('lossCategory')
                                                    ->label('LOSS CATEGORY')
                                                    ->badge()
                                                    ->color(fn (?string $state): string => match ($state) {
                                                        'shortfall' => 'warning',
                                                        'damage' => 'danger',
                                                        'spoilage' => 'danger',
                                                        'theft' => 'danger',
                                                        default => 'gray',
                                                    })
                                                    ->columnSpan(1),
                                            ]),
                                    ]),
                            ])
                            ->columnSpanFull(),
                    ]),
            ]);
    }
}
```

### 7C. PurchaseOrderResource `[NEW v12]`

**Model:** `App\Models\PurchaseOrder` · **Navigation Group:** PURCHASING · **Sort:** 1 · **Base Route:** `/admin/purchase-orders`

#### PurchaseOrderResource.php (thin resource class)

```php
namespace App\Filament\Resources\PurchaseOrders;

use App\Filament\Resources\PurchaseOrders\Pages\CreatePurchaseOrder;
use App\Filament\Resources\PurchaseOrders\Pages\EditPurchaseOrder;
use App\Filament\Resources\PurchaseOrders\Pages\ListPurchaseOrders;
use App\Filament\Resources\PurchaseOrders\Pages\ViewPurchaseOrder;
use App\Filament\Resources\PurchaseOrders\Schemas\PurchaseOrderForm;
use App\Filament\Resources\PurchaseOrders\Schemas\PurchaseOrderInfolist;
use App\Filament\Resources\PurchaseOrders\Tables\PurchaseOrdersTable;
use App\Models\PurchaseOrder;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class PurchaseOrderResource extends Resource
{
    protected static ?string $model = PurchaseOrder::class;

    protected static string | \UnitEnum | null $navigationGroup = 'PURCHASING';

    protected static ?int $navigationSort = 1;

    protected static ?string $recordTitleAttribute = 'reference_code';

    protected static string | \BackedEnum | null $navigationIcon = Heroicon::OutlinedShoppingCart;

    public static function form(Schema $schema): Schema
    {
        return PurchaseOrderForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return PurchaseOrdersTable::configure($table);
    }

    public static function infolist(Schema $schema): Schema
    {
        return PurchaseOrderInfolist::configure($schema);
    }

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()
            ->with(['supplier', 'warehouse', 'items.productVariant']);
    }

    public static function getRecordRouteBindingEloquentQuery(): Builder
    {
        return parent::getRecordRouteBindingEloquentQuery()
            ->withoutGlobalScopes([SoftDeletingScope::class]);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListPurchaseOrders::route('/'),
            'create' => CreatePurchaseOrder::route('/create'),
            'view'   => ViewPurchaseOrder::route('/{record}'),
            'edit'   => EditPurchaseOrder::route('/{record}/edit'),
        ];
    }
}
```

#### PurchaseOrderForm.php (create wizard)

```php
namespace App\Filament\Resources\PurchaseOrders\Schemas;

use Filament\Forms\Components\Placeholder;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Schemas\Schema;
use Filament\Support\Enums\Width;

class PurchaseOrderForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Wizard::make([
                Step::make('Supplier & Warehouse')
                    ->schema([
                        Select::make('supplier_id')
                            ->relationship('supplier', 'name')
                            ->searchable()
                            ->preload()
                            ->required()
                            ->createOptionForm(fn (Schema $schema) => \App\Filament\Resources\Suppliers\Schemas\SupplierForm::configure($schema)),
                        Select::make('warehouse_id')
                            ->label('Receiving Warehouse')
                            ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                            ->default(fn () => auth()->user()->warehouses()->count() === 1
                                ? auth()->user()->warehouses()->first()->id
                                : null)
                            ->required(),
                    ]),
                Step::make('Line Items')
                    ->schema([
                        Repeater::make('items')
                            ->relationship()
                            ->schema([
                                Select::make('product_variant_id')
                                    ->label('Variant (SKU)')
                                    ->relationship('productVariant', 'sku')
                                    ->searchable()
                                    ->preload()
                                    ->required()
                                    ->disableOptionsWhenSelectedInSiblingRepeaterItems(),
                                TextInput::make('ordered_unit_name')
                                    ->label('Unit')
                                    ->required(),
                                TextInput::make('ordered_unit_ratio')
                                    ->label('Unit Ratio (to base)')
                                    ->numeric()
                                    ->minValue(1)
                                    ->default(1)
                                    ->required(),
                                TextInput::make('ordered_qty')
                                    ->label('Ordered Qty')
                                    ->numeric()
                                    ->minValue(1)
                                    ->required()
                                    ->live(onBlur: true)
                                    ->afterStateUpdated(fn (Get $get, $set) => $set(
                                        'ordered_base_qty',
                                        (int) $get('ordered_qty') * (int) $get('ordered_unit_ratio')
                                    )),
                                TextInput::make('ordered_base_qty')
                                    ->label('Base Qty (computed)')
                                    ->numeric()
                                    ->disabled()
                                    ->dehydrated(),
                                TextInput::make('unit_cost_price')
                                    ->label('Unit Cost Price')
                                    ->numeric()
                                    ->step(0.0001)
                                    ->minValue(0)
                                    ->required(),
                            ])
                            ->columns(3)
                            ->minItems(1)
                            ->required(),
                    ]),
                Step::make('Review & Verify')
                    ->schema([
                        Toggle::make('update_cost_price')
                            ->label('Update catalog cost price on receipt')
                            ->helperText('If enabled, receiving this PO will set each variant\'s current cost price to this order\'s unit cost, if different.')
                            ->default(false),
                        Placeholder::make('review_summary')
                            ->content(fn (Get $get) => view(
                                'filament.wizards.purchase-order-review',
                                ['state' => $get()],
                            )),
                    ]),
            ])
                ->modalWidth(Width::SevenExtraLarge)
                ->closeModalByClickingAway(false),
        ]);
    }
}
```

#### CreatePurchaseOrder.php

```php
namespace App\Filament\Resources\PurchaseOrders\Pages;

use App\Filament\Resources\PurchaseOrders\PurchaseOrderResource;
use Filament\Resources\Pages\CreateRecord;

class CreatePurchaseOrder extends CreateRecord
{
    protected static string $resource = PurchaseOrderResource::class;

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $data['reference_code'] = $data['reference_code'] ?? 'PO-'.now()->format('YmdHis').'-'.random_int(100, 999);
        $data['ordered_by'] = auth()->id();

        return $data;
    }
}
```

#### PurchaseOrdersTable.php

```php
namespace App\Filament\Resources\PurchaseOrders\Tables;

use App\Enums\PurchaseOrderStatus;
use App\Models\PurchaseOrder;
use Filament\Actions\Action;
use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteAction;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\EditAction;
use Filament\Actions\ForceDeleteAction;
use Filament\Actions\ForceDeleteBulkAction;
use Filament\Actions\RestoreAction;
use Filament\Actions\RestoreBulkAction;
use Filament\Actions\ViewAction;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;

class PurchaseOrdersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('reference_code')
                    ->label('REFERENCE')
                    ->searchable()
                    ->sortable()
                    ->copyable()
                    ->weight('bold'),
                TextColumn::make('supplier.name')
                    ->label('SUPPLIER')
                    ->searchable()
                    ->sortable(),
                TextColumn::make('warehouse.name')
                    ->label('WAREHOUSE')
                    ->sortable(),
                TextColumn::make('status')
                    ->badge(),
                TextColumn::make('items_count')
                    ->label('LINE ITEMS')
                    ->counts('items')
                    ->numeric(),
                TextColumn::make('ordered_at')
                    ->label('ORDERED')
                    ->dateTime('M j, Y')
                    ->sortable()
                    ->placeholder('—'),
                TextColumn::make('received_at')
                    ->label('RECEIVED')
                    ->dateTime('M j, Y')
                    ->sortable()
                    ->placeholder('—'),
            ])
            ->filters([
                SelectFilter::make('status')->options(PurchaseOrderStatus::class),
                SelectFilter::make('supplier_id')->relationship('supplier', 'name')->label('Supplier'),
                SelectFilter::make('warehouse_id')->relationship('warehouse', 'name')->label('Warehouse'),
                TrashedFilter::make(),

                \App\Filament\Support\Filters\AdminReviewFilters::period('ordered_at')
                    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
            ])
            ->defaultSort('created_at', 'desc')
            ->recordActions([
                ViewAction::make(),

                EditAction::make()
                    ->visible(fn (PurchaseOrder $record) => $record->status === PurchaseOrderStatus::Draft)
                    ->modalWidth(Width::Large),

                Action::make('orderPurchase')
                    ->label('ORDER')
                    ->icon(Heroicon::PaperAirplane)
                    ->color('primary')
                    ->authorize('orderPurchase')
                    ->visible(fn (PurchaseOrder $record) => $record->status === PurchaseOrderStatus::Draft)
                    ->requiresConfirmation()
                    ->action(fn (PurchaseOrder $record) => app(\App\Services\PurchaseService::class)->orderPurchase($record)),

                Action::make('receivePurchase')
                    ->label('RECEIVE')
                    ->icon(Heroicon::ArchiveBoxArrowDown)
                    ->color('success')
                    ->authorize('receivePurchase')
                    ->visible(fn (PurchaseOrder $record) => in_array($record->status, [
                        PurchaseOrderStatus::Ordered,
                        PurchaseOrderStatus::PartiallyReceived,
                    ]))
                    ->modalWidth(Width::FourExtraLarge)
                    ->schema(fn (PurchaseOrder $record) => collect($record->items)
                        ->map(fn ($item) => TextInput::make("received.{$item->id}")
                            ->label("{$item->productVariant->sku} — outstanding {$item->outstandingBaseQty()} {$item->ordered_unit_name}")
                            ->numeric()
                            ->minValue(0)
                            ->maxValue($item->outstandingBaseQty())
                            ->default($item->outstandingBaseQty())
                        )
                        ->all())
                    ->action(function (array $data, PurchaseOrder $record) {
                        $received = collect($data['received'] ?? [])
                            ->filter(fn ($qty) => (int) $qty > 0)
                            ->mapWithKeys(fn ($qty, $itemId) => [(int) $itemId => (int) $qty])
                            ->all();

                        app(\App\Services\PurchaseService::class)->receivePurchase($record->id, $received);

                        Notification::make()->title('Purchase order received')->success()->send();
                    })
                    ->requiresConfirmation(),

                Action::make('cancelPurchase')
                    ->label('CANCEL')
                    ->icon(Heroicon::XMark)
                    ->color('danger')
                    ->authorize('cancelPurchase')
                    ->visible(fn (PurchaseOrder $record) => in_array($record->status, [
                        PurchaseOrderStatus::Draft,
                        PurchaseOrderStatus::Ordered,
                    ]) && $record->items->every(fn ($item) => $item->received_base_qty === 0))
                    ->requiresConfirmation()
                    ->action(fn (PurchaseOrder $record) => app(\App\Services\PurchaseService::class)->cancelPurchaseOrder($record)),

                DeleteAction::make()
                    ->authorize('delete')
                    ->visible(fn (PurchaseOrder $record) => in_array($record->status, [
                        PurchaseOrderStatus::Draft,
                        PurchaseOrderStatus::Cancelled,
                    ])),

                RestoreAction::make()->authorize('restore'),

                ForceDeleteAction::make()
                    ->authorize('forceDelete')
                    ->visible(fn () => auth()->user()->isAdmin()),
            ])
            ->toolbarActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make()->authorize('deleteAny'),
                    RestoreBulkAction::make()->authorize('restoreAny'),
                    ForceDeleteBulkAction::make()->authorize('forceDeleteAny'),
                ]),
            ]);
    }
}
```

#### PurchaseOrderInfolist.php

```php
namespace App\Filament\Resources\PurchaseOrders\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Icons\Heroicon;

class PurchaseOrderInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema
            ->schema([
                Grid::make(3)
                    ->schema([
                        Section::make('PURCHASE ORDER PROFILE')
                            ->icon(Heroicon::DocumentText)
                            ->schema([
                                Grid::make(2)
                                    ->schema([
                                        TextEntry::make('reference_code')
                                            ->label('REFERENCE CODE')
                                            ->weight(FontWeight::Bold)
                                            ->size('lg')
                                            ->copyable()
                                            ->color('primary'),

                                        TextEntry::make('status')
                                            ->label('STATUS')
                                            ->badge(),

                                        TextEntry::make('supplier.name')
                                            ->label('SUPPLIER')
                                            ->icon(Heroicon::BuildingStorefront),

                                        TextEntry::make('warehouse.name')
                                            ->label('RECEIVING WAREHOUSE')
                                            ->icon(Heroicon::BuildingOffice2),

                                        TextEntry::make('update_cost_price')
                                            ->label('UPDATES CATALOG COST')
                                            ->badge()
                                            ->color(fn (bool $state) => $state ? 'warning' : 'gray')
                                            ->formatStateUsing(fn (bool $state) => $state ? 'Yes' : 'No'),
                                    ]),
                            ])
                            ->columnSpan(2),

                        Section::make('SIGN-OFFS')
                            ->icon(Heroicon::ShieldCheck)
                            ->schema([
                                TextEntry::make('orderedBy.name')
                                    ->label('ORDERED BY')
                                    ->icon(Heroicon::User)
                                    ->placeholder('—'),

                                TextEntry::make('receivedBy.name')
                                    ->label('RECEIVED BY')
                                    ->icon(Heroicon::ArchiveBoxArrowDown)
                                    ->placeholder('Pending Intake'),

                                TextEntry::make('ordered_at')
                                    ->label('ORDERED AT')
                                    ->dateTime('M j, Y H:i')
                                    ->placeholder('—'),

                                TextEntry::make('received_at')
                                    ->label('RECEIVED AT')
                                    ->dateTime('M j, Y H:i')
                                    ->placeholder('—'),
                            ])
                            ->columnSpan(1),

                        Section::make('LINE ITEMS')
                            ->icon(Heroicon::ClipboardDocumentList)
                            ->schema([
                                RepeatableEntry::make('items')
                                    ->label('')
                                    ->schema([
                                        Grid::make(6)
                                            ->schema([
                                                TextEntry::make('productVariant.sku')
                                                    ->label('SKU')
                                                    ->weight(FontWeight::Bold)
                                                    ->columnSpan(1),

                                                TextEntry::make('productVariant.name')
                                                    ->label('PRODUCT')
                                                    ->columnSpan(2),

                                                TextEntry::make('ordered_base_qty')
                                                    ->label('ORDERED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('received_base_qty')
                                                    ->label('RECEIVED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('unit_cost_price')
                                                    ->label('UNIT COST')
                                                    ->money(config('app.currency'), decimals: 4)
                                                    ->columnSpan(1),
                                            ]),
                                    ]),
                            ])
                            ->columnSpanFull(),
                    ]),
            ]);
    }
}
```

### 7D. SalesOrderResource `[NEW v12]`

**Model:** `App\Models\SalesOrder` · **Navigation Group:** SALES · **Sort:** 1 · **Base Route:** `/admin/sales-orders`

#### SalesOrderResource.php (thin resource class)

```php
namespace App\Filament\Resources\SalesOrders;

use App\Filament\Resources\SalesOrders\Pages\CreateSalesOrder;
use App\Filament\Resources\SalesOrders\Pages\EditSalesOrder;
use App\Filament\Resources\SalesOrders\Pages\ListSalesOrders;
use App\Filament\Resources\SalesOrders\Pages\ViewSalesOrder;
use App\Filament\Resources\SalesOrders\Schemas\SalesOrderForm;
use App\Filament\Resources\SalesOrders\Schemas\SalesOrderInfolist;
use App\Filament\Resources\SalesOrders\Tables\SalesOrdersTable;
use App\Models\SalesOrder;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class SalesOrderResource extends Resource
{
    protected static ?string $model = SalesOrder::class;

    protected static string | \UnitEnum | null $navigationGroup = 'SALES';

    protected static ?int $navigationSort = 1;

    protected static ?string $recordTitleAttribute = 'reference_code';

    protected static string | \BackedEnum | null $navigationIcon = Heroicon::OutlinedBanknotes;

    public static function form(Schema $schema): Schema
    {
        return SalesOrderForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return SalesOrdersTable::configure($table);
    }

    public static function infolist(Schema $schema): Schema
    {
        return SalesOrderInfolist::configure($schema);
    }

    public static function getEloquentQuery(): Builder
    {
        return parent::getEloquentQuery()
            ->with(['customer', 'warehouse', 'items.productVariant']);
    }

    public static function getRecordRouteBindingEloquentQuery(): Builder
    {
        return parent::getRecordRouteBindingEloquentQuery()
            ->withoutGlobalScopes([SoftDeletingScope::class]);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListSalesOrders::route('/'),
            'create' => CreateSalesOrder::route('/create'),
            'view'   => ViewSalesOrder::route('/{record}'),
            'edit'   => EditSalesOrder::route('/{record}/edit'),
        ];
    }
}
```

#### SalesOrderForm.php (create wizard — abbreviated)

```php
namespace App\Filament\Resources\SalesOrders\Schemas;

use Filament\Forms\Components\Placeholder;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Schemas\Schema;
use Filament\Support\Enums\Width;

class SalesOrderForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Wizard::make([
                Step::make('Customer & Warehouse')
                    ->schema([
                        Select::make('customer_id')
                            ->relationship('customer', 'name')
                            ->searchable()
                            ->preload()
                            ->required()
                            ->createOptionForm(fn (Schema $schema) => \App\Filament\Resources\Customers\Schemas\CustomerForm::configure($schema)),
                        Select::make('warehouse_id')
                            ->label('Dispatching Warehouse')
                            ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                            ->default(fn () => auth()->user()->warehouses()->count() === 1
                                ? auth()->user()->warehouses()->first()->id
                                : null)
                            ->required(),
                    ]),
                Step::make('Line Items')
                    ->schema([
                        Repeater::make('items')
                            ->relationship()
                            ->schema([
                                Select::make('product_variant_id')
                                    ->label('Variant (SKU)')
                                    ->relationship('productVariant', 'sku')
                                    ->searchable()
                                    ->preload()
                                    ->required()
                                    ->disableOptionsWhenSelectedInSiblingRepeaterItems()
                                    ->live()
                                    ->afterStateUpdated(function (Get $get, $set, $state) {
                                        $variant = \App\Models\ProductVariant::with('currentPrice')->find($state);
                                        $set('_current_sale_price_preview', $variant?->currentPrice?->sale_price ?? '0.0000');
                                    }),
                                TextInput::make('unit_name')
                                    ->label('Unit')
                                    ->required(),
                                TextInput::make('unit_ratio')
                                    ->label('Unit Ratio (to base)')
                                    ->numeric()
                                    ->minValue(1)
                                    ->default(1)
                                    ->required(),
                                TextInput::make('qty')
                                    ->label('Qty')
                                    ->numeric()
                                    ->minValue(1)
                                    ->required()
                                    ->live(onBlur: true)
                                    ->afterStateUpdated(fn (Get $get, $set) => $set(
                                        'base_qty',
                                        (int) $get('qty') * (int) $get('unit_ratio')
                                    )),
                                TextInput::make('base_qty')
                                    ->label('Base Qty (computed)')
                                    ->numeric()
                                    ->disabled()
                                    ->dehydrated(),
                                Placeholder::make('_current_sale_price_preview')
                                    ->label('Current Catalog Sale Price')
                                    ->content(fn (Get $get) => $get('_current_sale_price_preview') ?? '—'),
                            ])
                            ->columns(3)
                            ->minItems(1)
                            ->required(),
                    ]),
                Step::make('Review & Verify')
                    ->schema([
                        Placeholder::make('review_summary')
                            ->content(fn (Get $get) => view(
                                'filament.wizards.sales-order-review',
                                ['state' => $get()],
                            )),
                    ]),
            ])
                ->modalWidth(Width::SevenExtraLarge)
                ->closeModalByClickingAway(false),
        ]);
    }
}
```

#### SalesOrdersTable.php (key actions)

```php
namespace App\Filament\Resources\SalesOrders\Tables;

use App\Enums\SalesOrderStatus;
use App\Models\SalesOrder;
use Filament\Actions\Action;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Table;

class SalesOrdersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('reference_code')->label('REFERENCE')->searchable()->sortable()->copyable()->weight('bold'),
                TextColumn::make('customer.name')->label('CUSTOMER')->searchable()->sortable(),
                TextColumn::make('warehouse.name')->label('WAREHOUSE')->sortable(),
                TextColumn::make('status')->badge(),
                TextColumn::make('items_count')->label('LINE ITEMS')->counts('items')->numeric(),
                TextColumn::make('confirmed_at')->label('CONFIRMED')->dateTime('M j, Y')->sortable()->placeholder('—'),
                TextColumn::make('dispatched_at')->label('DISPATCHED')->dateTime('M j, Y')->sortable()->placeholder('—'),
            ])
            ->filters([
                \Filament\Tables\Filters\SelectFilter::make('status')->options(SalesOrderStatus::class),
                \Filament\Tables\Filters\SelectFilter::make('customer_id')->relationship('customer', 'name')->label('Customer'),
                \Filament\Tables\Filters\SelectFilter::make('warehouse_id')->relationship('warehouse', 'name')->label('Warehouse'),
                \Filament\Tables\Filters\TrashedFilter::make(),

                \App\Filament\Support\Filters\AdminReviewFilters::period('confirmed_at')
                    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
            ])
            ->defaultSort('created_at', 'desc')
            ->recordActions([
                \Filament\Actions\ViewAction::make(),

                \Filament\Actions\EditAction::make()
                    ->visible(fn (SalesOrder $record) => $record->status === SalesOrderStatus::Draft)
                    ->modalWidth(Width::Large),

                Action::make('confirmSalesOrder')
                    ->label('CONFIRM')
                    ->icon(Heroicon::CheckCircle)
                    ->color('primary')
                    ->authorize('confirmSalesOrder')
                    ->visible(fn (SalesOrder $record) => $record->status === SalesOrderStatus::Draft)
                    ->requiresConfirmation()
                    ->action(fn (SalesOrder $record) => app(\App\Services\SalesService::class)->confirmSalesOrder($record)),

                Action::make('dispatchSale')
                    ->label('DISPATCH')
                    ->icon(Heroicon::Truck)
                    ->color('success')
                    ->authorize('dispatchSale')
                    ->visible(fn (SalesOrder $record) => in_array($record->status, [
                        SalesOrderStatus::Confirmed,
                        SalesOrderStatus::PartiallyDispatched,
                    ]))
                    ->modalWidth(Width::FourExtraLarge)
                    ->schema(function (SalesOrder $record) {
                        $variantIds = $record->items->pluck('product_variant_id')->all();
                        $availableByVariant = \App\Models\ProductVariant::batchAvailableQuantity($variantIds, $record->warehouse_id);

                        return collect($record->items)
                            ->map(function ($item) use ($availableByVariant) {
                                $available = $availableByVariant[$item->product_variant_id] ?? 0;
                                $safeMax = min($item->outstandingBaseQty(), max(0, $available));

                                return TextInput::make("dispatch.{$item->id}")
                                    ->label("{$item->productVariant->sku} — outstanding {$item->outstandingBaseQty()} {$item->unit_name} (available: {$available})")
                                    ->numeric()
                                    ->minValue(0)
                                    ->maxValue($safeMax)
                                    ->default($safeMax)
                                    ->helperText($available < $item->outstandingBaseQty()
                                        ? 'Insufficient stock for full dispatch — partial dispatch only.'
                                        : null);
                            })
                            ->all();
                    })
                    ->action(function (array $data, SalesOrder $record) {
                        $dispatch = collect($data['dispatch'] ?? [])
                            ->filter(fn ($qty) => (int) $qty > 0)
                            ->mapWithKeys(fn ($qty, $itemId) => [(int) $itemId => (int) $qty])
                            ->all();

                        app(\App\Services\SalesService::class)->dispatchSale($record->id, $dispatch);

                        Notification::make()->title('Sales order dispatched')->success()->send();
                    })
                    ->requiresConfirmation(),

                Action::make('recordReturn')
                    ->label('RECORD RETURN')
                    ->icon(Heroicon::ArrowUturnLeft)
                    ->color('warning')
                    ->authorize('recordSalesReturn')
                    ->visible(fn (SalesOrder $record) => $record->items->contains(fn ($item) => $item->dispatched_base_qty > 0))
                    ->modalWidth(Width::Large)
                    ->schema([
                        \Filament\Forms\Components\Select::make('sales_order_item_id')
                            ->label('Line Item')
                            ->options(fn (SalesOrder $record) => $record->items
                                ->where('dispatched_base_qty', '>', 0)
                                ->mapWithKeys(fn ($item) => [$item->id => "{$item->productVariant->sku} (dispatched: {$item->dispatched_base_qty})"]))
                            ->required(),
                        TextInput::make('returned_base_qty')
                            ->label('Returned Qty (Base)')
                            ->numeric()
                            ->minValue(1)
                            ->required(),
                        \Filament\Forms\Components\Textarea::make('notes')->columnSpanFull(),
                    ])
                    ->action(function (array $data) {
                        app(\App\Services\SalesService::class)->recordSalesReturn(
                            (int) $data['sales_order_item_id'],
                            (int) $data['returned_base_qty'],
                            $data['notes'] ?? null,
                        );

                        Notification::make()->title('Return recorded')->success()->send();
                    })
                    ->requiresConfirmation(),

                Action::make('cancelSalesOrder')
                    ->label('CANCEL')
                    ->icon(Heroicon::XMark)
                    ->color('danger')
                    ->authorize('cancelSalesOrder')
                    ->visible(fn (SalesOrder $record) => in_array($record->status, [
                        SalesOrderStatus::Draft,
                        SalesOrderStatus::Confirmed,
                    ]))
                    ->requiresConfirmation()
                    ->action(fn (SalesOrder $record) => app(\App\Services\SalesService::class)->cancelSalesOrder($record)),
            ]);
    }
}
```

#### SalesOrderInfolist.php

```php
namespace App\Filament\Resources\SalesOrders\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Icons\Heroicon;

class SalesOrderInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema
            ->schema([
                Grid::make(3)
                    ->schema([
                        Section::make('SALES ORDER PROFILE')
                            ->icon(Heroicon::DocumentText)
                            ->schema([
                                Grid::make(2)
                                    ->schema([
                                        TextEntry::make('reference_code')
                                            ->label('REFERENCE CODE')
                                            ->weight(FontWeight::Bold)
                                            ->size('lg')
                                            ->copyable()
                                            ->color('primary'),

                                        TextEntry::make('status')
                                            ->label('STATUS')
                                            ->badge(),

                                        TextEntry::make('customer.name')
                                            ->label('CUSTOMER')
                                            ->icon(Heroicon::UserGroup),

                                        TextEntry::make('warehouse.name')
                                            ->label('DISPATCHING WAREHOUSE')
                                            ->icon(Heroicon::BuildingOffice2),
                                    ]),
                            ])
                            ->columnSpan(2),

                        Section::make('SIGN-OFFS')
                            ->icon(Heroicon::ShieldCheck)
                            ->schema([
                                TextEntry::make('orderedBy.name')
                                    ->label('ORDERED BY')
                                    ->icon(Heroicon::User)
                                    ->placeholder('—'),

                                TextEntry::make('dispatchedBy.name')
                                    ->label('DISPATCHED BY')
                                    ->icon(Heroicon::Truck)
                                    ->placeholder('Pending Dispatch'),

                                TextEntry::make('confirmed_at')
                                    ->label('CONFIRMED AT')
                                    ->dateTime('M j, Y H:i')
                                    ->placeholder('—'),

                                TextEntry::make('dispatched_at')
                                    ->label('DISPATCHED AT')
                                    ->dateTime('M j, Y H:i')
                                    ->placeholder('—'),
                            ])
                            ->columnSpan(1),

                        Section::make('LINE ITEMS')
                            ->icon(Heroicon::ClipboardDocumentList)
                            ->schema([
                                RepeatableEntry::make('items')
                                    ->label('')
                                    ->schema([
                                        Grid::make(6)
                                            ->schema([
                                                TextEntry::make('productVariant.sku')
                                                    ->label('SKU')
                                                    ->weight(FontWeight::Bold)
                                                    ->columnSpan(1),

                                                TextEntry::make('productVariant.name')
                                                    ->label('PRODUCT')
                                                    ->columnSpan(2),

                                                TextEntry::make('base_qty')
                                                    ->label('ORDERED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('dispatched_base_qty')
                                                    ->label('DISPATCHED (BASE)')
                                                    ->numeric()
                                                    ->columnSpan(1),

                                                TextEntry::make('unit_sale_price_snapshot')
                                                    ->label('SNAPSHOT PRICE')
                                                    ->money(config('app.currency'), decimals: 4)
                                                    ->columnSpan(1),
                                            ]),
                                    ]),
                            ])
                            ->columnSpanFull(),
                    ]),
            ]);
    }
}
```

### 7E. SupplierResource & CustomerResource `[NEW v12]`

#### SupplierForm.php

```php
namespace App\Filament\Resources\Suppliers\Schemas;

use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;

class SupplierForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')->required()->maxLength(255),
            TextInput::make('contact_person')->maxLength(255),
            TextInput::make('phone')->tel(),
            TextInput::make('email')->email(),
            Textarea::make('address')->columnSpanFull(),
            Toggle::make('is_active')->default(true),
        ]);
    }
}
```

`CustomerForm` is field-for-field identical (same shape as `Supplier`).

`SupplierResource` / `CustomerResource` follow `WarehouseResource`'s thin pattern — drawer-style `EditAction`/`CreateAction` at `Width::Large`, no infolist needed for such simple master data, standard `DeleteAction`/`RestoreAction` pair guarded by `restrictOnDelete` at the DB layer.

---

## 🛡️ Section 8: Authorization — Policies

### 8A. PurchaseOrderPolicy `[NEW v12]`

```php
namespace App\Policies;

use App\Enums\PurchaseOrderStatus;
use App\Models\PurchaseOrder;
use App\Models\User;

/**
 * [FIX v11.1 / Principle A8] This class is the SOLE source of truth for
 * every permission/role decision involving PurchaseOrder. No ->visible()
 * closure on PurchaseOrdersTable, no method on PurchaseService, and no
 * check anywhere else in the codebase may re-derive what is decided here.
 *
 * Per Principle A8, once each method below is implemented and its test
 * checkpoint (PurchaseOrderPolicyTest) passes, this class is FROZEN except
 * for two cases: adding a genuinely new ability, or fixing a demonstrated
 * bug in an existing method's logic.
 */
class PurchaseOrderPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, PurchaseOrder $purchaseOrder): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, PurchaseOrder $purchaseOrder): bool
    {
        return $purchaseOrder->status === PurchaseOrderStatus::Draft;
    }

    public function delete(User $user, PurchaseOrder $purchaseOrder): bool
    {
        return in_array($purchaseOrder->status, [
            PurchaseOrderStatus::Draft,
            PurchaseOrderStatus::Cancelled,
        ], true);
    }

    public function restore(User $user): bool
    {
        return true;
    }

    public function forceDelete(User $user): bool
    {
        return $user->isAdmin();
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function restoreAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDeleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function orderPurchase(User $user, PurchaseOrder $purchaseOrder): bool
    {
        return $purchaseOrder->status === PurchaseOrderStatus::Draft;
    }

    public function receivePurchase(User $user, PurchaseOrder $purchaseOrder): bool
    {
        return in_array($purchaseOrder->status, [
            PurchaseOrderStatus::Ordered,
            PurchaseOrderStatus::PartiallyReceived,
        ], true);
    }

    /**
     * [FIX v11.1] This is the actual, sole enforcement point for the
     * "no stock has moved yet" cancellation boundary — not a duplicate
     * inline check inside PurchaseService::cancelPurchaseOrder().
     */
    public function cancelPurchase(User $user, PurchaseOrder $purchaseOrder): bool
    {
        if (! in_array($purchaseOrder->status, [PurchaseOrderStatus::Draft, PurchaseOrderStatus::Ordered], true)) {
            return false;
        }

        return $purchaseOrder->items->every(fn ($item) => $item->received_base_qty === 0);
    }
}
```

### 8B. SalesOrderPolicy `[NEW v12]`

```php
namespace App\Policies;

use App\Enums\SalesOrderStatus;
use App\Models\SalesOrder;
use App\Models\User;

/**
 * [FIX v11.1 / Principle A8] Sole source of truth for every permission/role
 * decision involving SalesOrder. Frozen once verified, per the same terms
 * as PurchaseOrderPolicy's doc-block above.
 */
class SalesOrderPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, SalesOrder $salesOrder): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, SalesOrder $salesOrder): bool
    {
        return $salesOrder->status === SalesOrderStatus::Draft;
    }

    public function delete(User $user, SalesOrder $salesOrder): bool
    {
        return in_array($salesOrder->status, [
            SalesOrderStatus::Draft,
            SalesOrderStatus::Cancelled,
        ], true);
    }

    public function restore(User $user): bool
    {
        return true;
    }

    public function forceDelete(User $user): bool
    {
        return $user->isAdmin();
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function restoreAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDeleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function confirmSalesOrder(User $user, SalesOrder $salesOrder): bool
    {
        return $salesOrder->status === SalesOrderStatus::Draft;
    }

    public function dispatchSale(User $user, SalesOrder $salesOrder): bool
    {
        return in_array($salesOrder->status, [
            SalesOrderStatus::Confirmed,
            SalesOrderStatus::PartiallyDispatched,
        ], true);
    }

    public function recordSalesReturn(User $user, SalesOrder $salesOrder): bool
    {
        return $salesOrder->items->contains(fn ($item) => $item->dispatched_base_qty > 0);
    }

    /**
     * [FIX v11.1] Sole enforcement point for "illegal once dispatch has
     * begun" — same Service-vs-Policy split as PurchaseOrderPolicy::
     * cancelPurchase() above.
     */
    public function cancelSalesOrder(User $user, SalesOrder $salesOrder): bool
    {
        return in_array($salesOrder->status, [
            SalesOrderStatus::Draft,
            SalesOrderStatus::Confirmed,
        ], true);
    }
}
```

### 8C. SupplierPolicy & CustomerPolicy `[NEW v12]`

```php
namespace App\Policies;

use App\Models\Supplier;
use App\Models\User;

/**
 * [FIX v11.1 / Principle A8] Sole source of truth for Supplier permissions.
 * Deliberately thin — Supplier is simple master data with no custom abilities.
 */
class SupplierPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Supplier $supplier): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Supplier $supplier): bool
    {
        return true;
    }

    public function delete(User $user, Supplier $supplier): bool
    {
        return $user->isAdmin();
    }

    public function restore(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDelete(User $user): bool
    {
        return $user->isAdmin();
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function restoreAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDeleteAny(User $user): bool
    {
        return $user->isAdmin();
    }
}
```

`CustomerPolicy` is field-for-field identical to `SupplierPolicy` with `Supplier` replaced by `Customer` throughout.

### 8D. Existing Parent v11.0 Policies (Consolidated Sketches)

The following nine policies are reconstructions against the parent blueprint's method table and extension notes. Phase 0 of the implementation prompt must diff them against whatever the actual `app/Policies/*.php` files currently contain before treating any of them as correct.

#### ProductPolicy

```php
namespace App\Policies;

use App\Models\Product;
use App\Models\User;

/**
 * [Principle A8] Sole source of truth for Product permissions.
 * Frozen once verified against the real codebase.
 */
class ProductPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Product $product): bool { return true; }
    public function create(User $user): bool { return true; }
    public function update(User $user, Product $product): bool { return true; }

    /**
     * Mirrors ProductObserver::deleting()'s own guard so the delete action
     * doesn't even authorize for a user who would immediately hit the
     * observer's Exception anyway. This is intentional duplication of a
     * CONDITION, not of an AUTHORIZATION DECISION.
     */
    public function delete(User $user, Product $product): bool
    {
        return $product->variants()->whereNull('deleted_at')->count() === 0;
    }

    public function restore(User $user): bool { return true; }
    public function forceDelete(User $user): bool { return $user->isAdmin(); }
}
```

#### ProductVariantPolicy

```php
namespace App\Policies;

use App\Models\ProductVariant;
use App\Models\User;

class ProductVariantPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, ProductVariant $productVariant): bool { return true; }
    public function create(User $user): bool { return true; }
    public function update(User $user, ProductVariant $productVariant): bool { return true; }
    public function delete(User $user, ProductVariant $productVariant): bool { return true; }
    public function restore(User $user): bool { return true; }

    /**
     * "always false" per the parent blueprint's method table — a variant
     * referenced by any ledger table (restrictOnDelete throughout Section 2)
     * should never be force-deletable through the panel at all, regardless
     * of role.
     */
    public function forceDelete(User $user): bool { return false; }

    public function setPrice(User $user): bool { return true; }

    public function adjustStock(User $user): bool { return $user->isAdmin(); }
}
```

#### TransferRequisitionPolicy

```php
namespace App\Policies;

use App\Enums\TransferRequisitionStatus;
use App\Models\TransferRequisition;
use App\Models\User;

/**
 * [FIX v11 in the parent blueprint, reaffirmed under Principle A8 here]
 * cancel() below is the parent blueprint's own flagship example of exactly
 * what A8 requires everywhere: the five-state pre-dispatch allowlist is
 * enforced HERE, in PHP, independently of TransferRequisitionResource's
 * ->visible() closure.
 */
class TransferRequisitionPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, TransferRequisition $transferRequisition): bool { return true; }
    public function create(User $user): bool { return true; }

    public function update(User $user, TransferRequisition $transferRequisition): bool
    {
        return $transferRequisition->status === TransferRequisitionStatus::Draft;
    }

    public function delete(User $user, TransferRequisition $transferRequisition): bool
    {
        return in_array($transferRequisition->status, [
            TransferRequisitionStatus::Draft,
            TransferRequisitionStatus::Cancelled,
        ], true);
    }

    public function restore(User $user): bool { return true; }
    public function forceDelete(User $user): bool { return $user->isAdmin(); }

    public function confirm(User $user, TransferRequisition $transferRequisition): bool
    {
        return in_array($transferRequisition->status, [
            TransferRequisitionStatus::Requested,
            TransferRequisitionStatus::UnderReviewFulfiller,
            TransferRequisitionStatus::UnderReviewRequestor,
        ], true);
    }

    public function dispatch(User $user, TransferRequisition $transferRequisition): bool
    {
        return $transferRequisition->status === TransferRequisitionStatus::Confirmed;
    }

    public function receive(User $user, TransferRequisition $transferRequisition): bool
    {
        return in_array($transferRequisition->status, [
            TransferRequisitionStatus::Dispatched,
            TransferRequisitionStatus::PartiallyReceived,
        ], true);
    }

    /**
     * `[FIX v11]` THE method this whole principle is named after in the
     * parent blueprint. Permanently five-state, pre-dispatch only — see
     * Principle #14. Do not widen this to include Dispatched or
     * PartiallyReceived; there is no compensating stock-reversal pathway.
     */
    public function cancel(User $user, TransferRequisition $transferRequisition): bool
    {
        return in_array($transferRequisition->status, [
            TransferRequisitionStatus::Draft,
            TransferRequisitionStatus::Requested,
            TransferRequisitionStatus::UnderReviewFulfiller,
            TransferRequisitionStatus::UnderReviewRequestor,
            TransferRequisitionStatus::Confirmed,
        ], true);
    }
}
```

#### TransferRequisitionItemRevisionPolicy

```php
namespace App\Policies;

use App\Models\TransferRequisitionItemRevision;
use App\Models\User;

class TransferRequisitionItemRevisionPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, TransferRequisitionItemRevision $revision): bool { return true; }
    public function create(User $user): bool { return true; }
    public function update(User $user, TransferRequisitionItemRevision $revision): bool { return true; }
    public function delete(User $user, TransferRequisitionItemRevision $revision): bool { return true; }
    public function restore(User $user): bool { return true; }
    public function forceDelete(User $user): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return true; }
    public function restoreAny(User $user): bool { return true; }
    public function forceDeleteAny(User $user): bool { return $user->isAdmin(); }
}
```

#### StockMovementPolicy

```php
namespace App\Policies;

use App\Models\StockMovement;
use App\Models\User;

/**
 * Immutable audit trail by design. Every mutating method is a blanket
 * false, independent of role — there is no role, including admin, for
 * which a stock_movements row should ever be editable or deletable
 * through the panel.
 */
class StockMovementPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, StockMovement $stockMovement): bool { return true; }
    public function create(User $user): bool { return false; }
    public function update(User $user, StockMovement $stockMovement): bool { return false; }
    public function delete(User $user, StockMovement $stockMovement): bool { return false; }
    public function restore(User $user): bool { return false; }
    public function forceDelete(User $user): bool { return false; }
    public function deleteAny(User $user): bool { return false; }
    public function restoreAny(User $user): bool { return false; }
    public function forceDeleteAny(User $user): bool { return false; }
}
```

#### InTransitPolicy

```php
namespace App\Policies;

use App\Models\InTransit;
use App\Models\TransferRequisition;
use App\Models\User;

class InTransitPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, InTransit $inTransit): bool { return true; }
    public function create(User $user): bool { return false; }
    public function update(User $user, InTransit $inTransit): bool { return false; }
    public function delete(User $user, InTransit $inTransit): bool { return false; }
    public function restore(User $user): bool { return false; }
    public function forceDelete(User $user): bool { return false; }
    public function deleteAny(User $user): bool { return false; }
    public function restoreAny(User $user): bool { return false; }
    public function forceDeleteAny(User $user): bool { return false; }

    /**
     * Note this takes the InTransit's PARENT TransferRequisition's status,
     * not any status field on InTransit itself — InTransitResource's
     * ReceiveIntakeAction routes to the STN scan flow keyed on
     * transfer_requisition_id, so the permission question is really "is
     * the parent requisition receivable," which
     * TransferRequisitionPolicy::receive() already answers. Delegating to
     * it here is itself an application of Principle A8 — one status-
     * allowlist, one place it's decided, called from wherever it's needed.
     */
    public function receive(User $user, InTransit $inTransit): bool
    {
        return $user->can('receive', $inTransit->transferRequisition);
    }
}
```

#### LossLedgerPolicy

```php
namespace App\Policies;

use App\Models\LossLedger;
use App\Models\User;

class LossLedgerPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, LossLedger $lossLedger): bool { return true; }
    public function create(User $user): bool { return false; }
    public function update(User $user, LossLedger $lossLedger): bool { return false; }
    public function delete(User $user, LossLedger $lossLedger): bool { return false; }
    public function restore(User $user): bool { return false; }
    public function forceDelete(User $user): bool { return false; }
    public function deleteAny(User $user): bool { return false; }
    public function restoreAny(User $user): bool { return false; }
    public function forceDeleteAny(User $user): bool { return false; }

    /**
     * "all users" per the parent blueprint's extension note — recordLoss
     * is operationally available to anyone, not admin-gated.
     */
    public function recordLoss(User $user): bool
    {
        return true;
    }
}
```

#### WarehousePolicy

```php
namespace App\Policies;

use App\Models\User;
use App\Models\Warehouse;

class WarehousePolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Warehouse $warehouse): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, Warehouse $warehouse): bool { return true; }
    public function delete(User $user, Warehouse $warehouse): bool { return false; }
    public function restore(User $user): bool { return false; }
    public function forceDelete(User $user): bool { return false; }
    public function deleteAny(User $user): bool { return false; }
    public function restoreAny(User $user): bool { return false; }
    public function forceDeleteAny(User $user): bool { return false; }

    /**
     * "all authenticated users" per the parent blueprint's extension note
     * (operational flexibility) — deliberately NOT admin-gated, unlike
     * create() above.
     */
    public function adjustStock(User $user): bool { return true; }

    public function recordLoss(User $user): bool { return true; }
}
```

#### UserPolicy

```php
namespace App\Policies;

use App\Models\User;

class UserPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, User $model): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }

    public function update(User $user, User $model): bool
    {
        return $user->isAdmin() || $user->id === $model->id;
    }

    /**
     * Self-protection guard per the parent blueprint's extension note —
     * an admin cannot delete their own account through this policy.
     */
    public function delete(User $user, User $model): bool
    {
        return $user->isAdmin() && $user->id !== $model->id;
    }

    public function restore(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function restoreAny(User $user): bool { return $user->isAdmin(); }
    public function forceDeleteAny(User $user): bool { return $user->isAdmin(); }
}
```

---

## 🔍 Section 9: Shared Filter Architecture `[NEW v12]`

### AdminReviewFilters.php

```php
namespace App\Filament\Support\Filters;

use Carbon\Carbon;
use Filament\Forms\Components\DatePicker;
use Filament\Forms\Components\Select;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Tables\Filters\Filter;
use Filament\Tables\Filters\Indicator;
use Filament\Tables\Filters\SelectFilter;
use Illuminate\Database\Eloquent\Builder;

/**
 * [Added v11.1] Shared, System-Admin-only filters — warehouse and period —
 * reused across PurchaseOrdersTable, SalesOrdersTable, StockMovementsTable,
 * and LossLedgersTable. Built once here instead of four times.
 *
 * IMPORTANT: options() is a UI affordance, not an authorization boundary.
 * The list constrains what the dropdown displays, but the submitted value
 * is not checked against it before the filter runs. If filter-level
 * authorization is required, scope the table query itself — for example
 * with modifyQueryUsing() or a global scope — so restricted rows are
 * never reachable regardless of what the filter submits.
 */
class AdminReviewFilters
{
    /**
     * Warehouse filter — a plain SelectFilter listing ALL warehouses
     * (not auth()->user()->warehouses(), which is the staff-scoped list
     * used elsewhere). This is deliberate: an admin reviewing
     * cross-warehouse activity needs to see and filter by warehouses they
     * may not be personally assigned to.
     */
    public static function warehouse(string $relationshipName = 'warehouse'): SelectFilter
    {
        return SelectFilter::make('warehouse_id')
            ->label('Warehouse')
            ->relationship($relationshipName, 'name')
            ->searchable()
            ->preload();
    }

    /**
     * Period filter — a single dropdown of common presets (Today, This
     * Week, This Month, This Year, Specific Date, Custom Range), with
     * conditional fields that only appear for the presets that need them.
     *
     * $dateColumn: the column to filter on — differs per resource
     * (created_at for stock_movements, recorded_at for loss_ledgers,
     * ordered_at for purchase_orders, confirmed_at or ordered_at for
     * sales_orders — pass whichever is the resource's primary date of
     * record).
     */
    public static function period(string $dateColumn): Filter
    {
        return Filter::make('period')
            ->label('Period')
            ->schema([
                Select::make('preset')
                    ->label('Period')
                    ->options([
                        'today'         => 'Today',
                        'this_week'     => 'This Week',
                        'this_month'    => 'This Month',
                        'this_year'     => 'This Year',
                        'specific_date' => 'Specific Date',
                        'custom_range'  => 'Custom Range',
                    ])
                    ->default(null)
                    ->native(false)
                    ->live(),

                DatePicker::make('specific_date')
                    ->label('Date')
                    ->visible(fn (Get $get) => $get('preset') === 'specific_date'),

                DatePicker::make('range_from')
                    ->label('From')
                    ->visible(fn (Get $get) => $get('preset') === 'custom_range'),

                DatePicker::make('range_until')
                    ->label('Until')
                    ->visible(fn (Get $get) => $get('preset') === 'custom_range'),
            ])
            ->query(function (Builder $query, array $data) use ($dateColumn): Builder {
                return match ($data['preset'] ?? null) {
                    'today'         => $query->whereDate($dateColumn, now()->toDateString()),
                    'this_week'     => $query->whereBetween($dateColumn, [now()->startOfWeek(), now()->endOfWeek()]),
                    'this_month'    => $query->whereBetween($dateColumn, [now()->startOfMonth(), now()->endOfMonth()]),
                    'this_year'     => $query->whereBetween($dateColumn, [now()->startOfYear(), now()->endOfYear()]),
                    'specific_date' => $query->when(
                        $data['specific_date'] ?? null,
                        fn (Builder $q, $date) => $q->whereDate($dateColumn, $date),
                    ),
                    'custom_range' => $query
                        ->when($data['range_from'] ?? null, fn (Builder $q, $date) => $q->whereDate($dateColumn, '>=', $date))
                        ->when($data['range_until'] ?? null, fn (Builder $q, $date) => $q->whereDate($dateColumn, '<=', $date)),
                    default => $query,
                };
            })
            // [Verified v11.1 against Filament v5 docs] indicateUsing() may
            // return either a single string (used for the single-value
            // presets below) or an array of Indicator::make(...) objects
            // when a filter has more than one independently-clearable
            // field — the documented pattern for exactly this custom_range
            // case, since it lets each date be removed on its own from the
            // active-filters bar via ->removeField() rather than clearing
            // the whole filter at once.
            ->indicateUsing(function (array $data): string|array|null {
                return match ($data['preset'] ?? null) {
                    'today'         => 'Today',
                    'this_week'     => 'This week',
                    'this_month'    => 'This month',
                    'this_year'     => 'This year',
                    'specific_date' => isset($data['specific_date'])
                        ? 'On '.Carbon::parse($data['specific_date'])->toFormattedDateString()
                        : null,
                    'custom_range'  => array_filter([
                        isset($data['range_from'])
                            ? Indicator::make('From '.Carbon::parse($data['range_from'])->toFormattedDateString())
                                ->removeField('range_from')
                            : null,
                        isset($data['range_until'])
                            ? Indicator::make('Until '.Carbon::parse($data['range_until'])->toFormattedDateString())
                                ->removeField('range_until')
                            : null,
                    ]),
                    default => null,
                };
            });
    }
}
```

### Usage in Resources

**PurchaseOrdersTable** (add to existing filters):
```php
\App\Filament\Support\Filters\AdminReviewFilters::period('ordered_at')
    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
```

**SalesOrdersTable** (add to existing filters):
```php
\App\Filament\Support\Filters\AdminReviewFilters::period('confirmed_at')
    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
```

**StockMovementsTable** (add both — currently has neither):
```php
\App\Filament\Support\Filters\AdminReviewFilters::warehouse()
    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
\App\Filament\Support\Filters\AdminReviewFilters::period('created_at')
    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
```

**LossLedgersTable** (replaces existing inline date_range filter):
```php
\App\Filament\Support\Filters\AdminReviewFilters::period('recorded_at')
    ->visible(fn () => auth()->user()->isAdmin() || auth()->user()->isAuditor()),
// warehouse_id SelectFilter already exists — unchanged.
```

---

## 📊 Section 10: Dashboard — Bento Grid & Widgets

### Colour Palette

| Token | Value | Usage |
|---|---|---|
| primary | #3b82f6 (Operational Blue) | Primary action execution triggers only (Receive, Dispatch, Confirm, Execute Now). Restricted to ≤10% of any single view. |
| surface | #ffffff | Card wrappers, table backgrounds |
| surface-muted | #fafafa (zinc-50) | Hover states |
| border | #e4e4e7 (zinc-200) | 1px solid borders on cards, table headers |
| text-primary | #18181b (zinc-900) | Headings, primary content |
| text-secondary | #71717a (zinc-500) | Labels, metadata |
| danger | #ef4444 | Loss write-offs, force-delete |
| warning | #f59e0b | Partial intake, negotiation pending |
| success | #22c55e | Completed transfers, cleared transit |

### Elevation Rules

- **Flat Rest States:** Card wrappers, borders, and table headers remain flat (1px solid zinc-200).
- **Elevation on Focus:** Shadows trigger only during active modal focus (`shadow-lg`).
- **No Zebra Striping:** Alternating row backgrounds are prohibited. Rows rely on thin bottom dividers (`border-b border-zinc-200`) and instantaneous hover highlights (`hover:bg-zinc-50`).

### Glassmorphic Bento Grid

The landing dashboard organises widgets into a responsive 4-column asymmetrical bento grid:

```
┌─────────────────────┬─────────────┬─────────────┐
│                     │             │             │
│   StatsOverview     │  LowStock   │  Recent     │
│   (2 cols, 1 row)   │  (1 col)    │  Movements  │
│                     │             │  (1 col)    │
├─────────────────────┼─────────────┴─────────────┤
│                     │                           │
│   Quick Actions     │   Active In-Transit       │
│   (1 col)           │   (2 cols)                │
│                     │                           │
└─────────────────────┴───────────────────────────┘
```

Widget containers use specular glass styling: `backdrop-filter: blur(24px)`, `background: rgba(255, 255, 255, 0.7)`, `border: 1px solid rgba(255, 255, 255, 0.3)`.

### Widget Caching — `[ACCEPTED RISK]`

Heavy widget sums are wrapped in `Cache::remember('stats_overview_...', 300)` to eliminate memory bottlenecks on massive `stock_movements` tables.

> **`[ACCEPTED RISK NOTE]`** The `LowStockAlertsWidget` iterates `ProductVariant` records and calls the `availableQuantity()` accessor per row (one `onHandQuantity()` query + one `reservedQuantity()` query per variant, per warehouse), with the entire widget result wrapped in a single 300-second cache window. **This was a deliberate scope decision made during blueprint review, not an oversight.** It masks query volume during the cached window but does not reduce it: every cache miss (once per 300s, per warehouse-scoped dashboard view) still issues 2N queries for N variants. At small-to-medium catalog sizes (low thousands of variants) this is likely acceptable. At large catalog sizes (tens of thousands of variants across many warehouses), this will produce a slow, spiky cache-refresh moment every 5 minutes and should be revisited.
>
> **Upgrade path, if/when catalog size grows:** replace the per-variant loop with a single grouped aggregate query — e.g. one `stock_movements` query grouped by `(product_variant_id, warehouse_id)` with `SUM(quantity)`, joined against a similarly grouped `transfer_requisition_items` aggregate for `Confirmed`-status reservations, compared against `reorder_point` in a single pass. This becomes a red flag worth monitoring once `product_variants` count exceeds roughly 5,000–10,000 active rows, or if dashboard load times exceed ~1–2 seconds on cache miss in production APM traces.

### Dashboard Widget Definitions

| Widget | Data Source | Cache TTL | Type |
|---|---|---|---|
| StatsOverviewWidget | Total On-Hand Base Stock, Pending Requisitions, Active In-Transit Cargo, Total Write-Off Value | 300s | TableWidget |
| LowStockAlertsWidget | Variants where `availableQuantity <= reorder_point` — **per-variant accessor loop, cached (see accepted-risk note above)** | 300s | ChartWidget (bar) |
| RecentMovementsWidget | Compact timeline of recent `stock_movements` (daily buckets, 7 days) | 60s | ChartWidget (line) |
| ActiveInTransitWidget | InTransit rows where `status != cleared` | 300s | TableWidget |
| SalesRevenueTrendWidget | Daily `Sale` movement value (`\|quantity\| × unit_sale_price_snapshot` from `sales_order_items`, bucketed by dispatch day, last 30 days) | 300s | ChartWidget (line) |
| TopSellingVariantsWidget | Top 10 variants by total dispatched base qty across `Sale` movements, current month | 300s | ChartWidget (bar, horizontal) |
| SalesVsPurchasesWidget | Side-by-side monthly total: `Purchase` movement value (qty × `unit_cost_price`) vs `Sale` movement value (qty × `unit_sale_price_snapshot`), last 6 months | 300s | ChartWidget (bar, grouped) |
| PendingFulfillmentWidget | Count of `SalesOrder` in `Confirmed`/`PartiallyDispatched` and `PurchaseOrder` in `Ordered`/`PartiallyReceived`, scoped to user's warehouses | 60s | TableWidget |

### ChartWidget Implementation Notes

- Both `LowStockAlertsWidget` and `RecentMovementsWidget` extend `Filament\Widgets\ChartWidget` (abstract base class).
- Protected methods implemented: `getType()`, `getData()`, `getOptions()`, `getHeading()`.
- `LowStockAlertsWidget::getType()` returns `'bar'` — dual dataset bar chart (Current Stock vs Reorder Point).
- `RecentMovementsWidget::getType()` returns `'line'` — single dataset line chart (7-day daily buckets).
- Role-based gate check in `getData()`: only Admin/Auditor roles receive computed data; others receive empty structure.
- Cache keys: `low_stock_alerts_chart_{userId}_{firstWarehouseId}` (300s), `recent_movements_chart_{userId}_{firstWarehouseId}` (60s).
- `$columnSpan = 1` for both (bento grid: LowStock 1 col, RecentMovements 1 col).

**New Sales Widgets (v12):**
- All widgets gate on Admin/Auditor role in `getData()`, returning empty structure otherwise — same as `LowStockAlertsWidget`.
- Cache keys: `sales_revenue_trend_{userId}_{firstWarehouseId}`, `top_selling_variants_{userId}_{firstWarehouseId}_{month}`, `sales_vs_purchases_{userId}_{firstWarehouseId}`, no cache key needed for `PendingFulfillmentWidget` beyond its 60s TTL bucket.
- Warehouse-id scoping enforced on every underlying query.
- **Money precision in charts:** chart datasets pass pre-rounded floats to Chart.js for *display only*; the underlying aggregate query must use `bcmul`/`SUM()` on the decimal columns server-side, never sum floats in PHP.
- `SalesVsPurchasesWidget` is a genuinely new query shape (two aggregates joined by month across two different tables) — flag this explicitly to the implementer as the one widget that is *not* a drop-in copy of the existing pattern.

---

## 📋 Section 11: Master Execution Sequence (18 Phases)

### Phase 00: Environment & Core Guardrails Setup

1. Bootstrap Laravel 13 with PostgreSQL.
2. Install FilamentPHP v5 (`composer require filament/filament:"^5.0"`).
3. Install Livewire v4.
4. Install `simplesoftwareio/simple-qrcode` for STN QR generation.
5. Install `pestphp/pest` for testing.
6. Add `'currency' => env('APP_CURRENCY', 'PHP')` to `config/app.php`.
7. **`[FIX v11]`** Verify `ext-bcmath` is enabled in the target PHP environment. Add to `composer.json`'s `require` block as `"ext-bcmath": "*"` so Composer fails the install early on environments missing the extension.
8. Mandate `->strictAuthorization()` in `AdminPanelProvider` so unhandled actions fail closed against policies.
9. Enumerate every policy method (including custom abilities `dispatch`, `receive`, `cancel`, `setPrice`, `recordLoss`, `adjustStock`, `orderPurchase`, `receivePurchase`, `confirmSalesOrder`, `dispatchSale`, `recordSalesReturn`, `cancelPurchase`, `cancelSalesOrder`) and ensure they are covered before enabling strict mode.

### Phase 01: Relational Schema Migrations

Execute the migrations in strict dependency order:

1. products
2. product_variants
3. product_variant_prices
4. product_variant_unit_conversions
5. warehouses
6. users role update
7. user_warehouse
8. stock_movements
9. transfer_requisitions
10. transfer_requisition_items
11. transfer_requisition_item_revisions
12. in_transits
13. loss_ledgers
14. stock_movement_idempotency_keys
15. suppliers
16. customers
17. purchase_orders
18. purchase_order_items
19. sales_orders
20. sales_order_items

Ledger FKs use `restrictOnDelete`. `stock_movements.notes` is added. `loss_ledgers.transfer_requisition_id` is nullable with `nullOnDelete`.

### Phase 02: Base Seeders & Opening Ledger

Populate warehouses, products, variants, unit conversions, current prices, and seed opening stocks as receive entries in `stock_movements`. Seed suppliers and customers.

### Phase 03: Eloquent Model Projections & Enums

Implement derived stock methods (`onHandQuantity`, `reservedQuantity`, `reservedForSalesQuantity`, `availableQuantity`, `batchAvailableQuantity`) on `ProductVariant`, including the doc-block on `reservedQuantity()` establishing its permanent Confirmed-only scope boundary. Create the eight backed enums, each implementing `HasLabel`, `HasColor`, `HasIcon`, and routing `getLabel()` through `__()`. Register `ProductObserver` in `AppServiceProvider::boot()`. Implement the `LossLedger` model, including `snapshotUnitCostFrom()` and `calculateTotalFinancialLoss()`. Implement `Supplier`, `Customer`, `PurchaseOrder`, `PurchaseOrderItem`, `SalesOrder`, `SalesOrderItem` models.

### Phase 04: Transactional Inventory Engine

Implement `InventoryService` with pessimistic locking, substitute variant matching, multi-batch intake, and omitted receipt write-offs. Apply canonical sorted-warehouse-ID locking to `directTransfer()` (matching `dispatchTransfer()`). Apply unit-ratio validation guards to `recordMovement()` and `directTransfer()`. Apply the `scanToReceive()` idempotency state-check and `bcmath`-based loss valuation. Implement `NegotiationService` with the two approved-* write paths. Implement `PurchaseService` with over-receipt guard. Implement `SalesService` with over-dispatch guard, price snapshot at confirm-time, and sales return logic.

### Phase 05: Product Catalog Resource

Build `ProductResource` bound directly to `ProductVariant`. Inline `createOptionForm` for parent Product families. No `VariantsRelationManager` — all variant management flows through the resource's table and inline actions.

### Phase 06: Price Snapshots & Unit Conversions

Build `SetCurrentPriceAction` (`Width::Large`, 3 fields) and `ManageUnitConversionsAction` (`Width::SevenExtraLarge`, repeater). No `PricesRelationManager` or `ConversionsRelationManager`.

### Phase 07: Warehouses & Manual Adjustments

Build `WarehouseResource` (slide-over drawer, `Width::Large`). Build `QuickStockAdjustmentAction` (`Width::Large`, 5 fields including notes with 15-char minimum).

### Phase 08: Inter-Warehouse Requisition Wizard

Implement the 3-step creation wizard dialog modal (`Width::SevenExtraLarge`, `closeModalByClickingAway(false)`) using `Wizard::make([...])` with `Step::make()`.

### Phase 09: Negotiation Loop UI

Build review actions and revision forms for counter-offers and substitute variant swapping, wired to `NegotiationService::propose()` / `accept()` / `reject()` / `counter()`.

**Maintainer guardrail:** `EditDraftAction` is scoped strictly to draft requisitions. Widening it to any post-draft negotiable state (`requested`, `under_review_*`) would allow `transfer_requisition_item_revisions` rows to be cascade-deleted by item removal, destroying the negotiation audit trail. Do not widen this scope without first removing the cascade on that FK.

### Phase 10: Dispatch, In-Transit Monitor & Confirm Materialization

Wire `ConfirmAction` to call `NegotiationService::materializeRequestedAsApproved()` before transitioning status. Connect `DispatchAction` to `InventoryService::dispatchTransfer()`. Build `InTransitResource` (read-only table with `ReceiveIntakeAction`). Wire `CancelAction`'s visibility to the corrected five-state pre-dispatch allowlist.

### Phase 11: Printable STN & Signed QR Route

Build PDF manifests rendering 7-day signed scan URLs (`URL::temporarySignedRoute(..., expiration: now()->addDays(7), ...)`).

### Phase 12: Scan-to-Receive Modal & Multi-Batch Intake

Implement `ScanReceiptController` (`GET /stn/{transferRequisition}/scan`, middleware `['web', 'auth', 'signed']`) and the auto-triggering intake reconciliation modal (`ScanToReceiveAction` named `scanToReceive` for `mountAction()` compatibility). Support repeated partial intakes. Confirm the idempotency state-check in `InventoryService::scanToReceive()` is exercised by a duplicate-submission Pest test simulating a mobile client retry.

### Phase 13: Read-Only Audit Ledgers

Build `StockMovementResource` (signed integer quantity sum footer; notes surfaced) and `LossLedgerResource` (decimal(15,4) sum footers; nullable `transferRequisition.reference_code` renders `—`). Apply `AdminReviewFilters` optionally.

### Phase 14: Purchases Module `[NEW v12]`

Build `PurchaseOrderResource` with 3-step wizard create. Implement `PurchaseOrdersTable` with order/receive/cancel actions. Build `PurchaseOrderInfolist` with profile grid and repeatable line-item entry. Build `SupplierResource` with drawer-style CRUD.

### Phase 15: Sales Module `[NEW v12]`

Build `SalesOrderResource` with 3-step wizard create. Implement `SalesOrdersTable` with confirm/dispatch/return/cancel actions. Build `SalesOrderInfolist` with profile grid and repeatable line-item entry (showing `unit_sale_price_snapshot` read-only). Build `CustomerResource` with drawer-style CRUD.

### Phase 16: Glassmorphic Bento Dashboard

Construct responsive bento dashboard with 300-second cached widgets (StatsOverviewWidget, LowStockAlertsWidget, RecentMovementsWidget, ActiveInTransitWidget, SalesRevenueTrendWidget, TopSellingVariantsWidget, SalesVsPurchasesWidget, PendingFulfillmentWidget).

### Phase 17: Multi-Language Translation

Abstract 100% of user-facing UI labels into translation catalogs under `lang/en/`, `lang/es/`, and `lang/tl/`. Verify every enum's `getLabel()` resolves through `__()`.

### Phase 18: Automated CI/CD Testing

Execute Pest unit suites (SQLite `:memory:`) and Playwright E2E browser suites (PostgreSQL container), including all coverage targets listed throughout this document.

---

## 🧪 Section 12: Automated CI/CD Testing & E2E Validation Strategy

| Test Runner | Environment | Focus Area |
|---|---|---|
| Laravel Pint | Local / CI | Code style compliance (`./vendor/bin/pint --test`) |
| Pest PHP | SQLite (`:memory:`) | Unit, Feature, Service & Model tests (`./vendor/bin/pest`) |
| Playwright | PostgreSQL (Test DB) | Sequential multi-role E2E browser flows (`npx playwright test`) |

### Critical Pest Coverage Targets

**Core Stock Engine:**
```
ProductVariantTest::reserved_quantity_excludes_dispatched_requisitions()
ProductVariantTest::reserved_quantity_excludes_dispatched_and_partially_received()
ProductVariantTest::batchAvailableQuantity_matches_instance_method_for_each_variant_individually()
ProductVariantTest::batchAvailableQuantity_returns_zero_for_variant_with_no_movements_or_reservations()
ProductVariantTest::batchAvailableQuantity_issues_exactly_three_queries_regardless_of_variant_count()
```

**Inventory Service:**
```
InventoryServiceTest::dispatch_throws_when_approved_base_qty_is_null()
InventoryServiceTest::scan_to_receive_supports_partial_batches()
InventoryServiceTest::first_scan_omission_writes_full_loss()
InventoryServiceTest::subsequent_scan_omission_does_not_write_loss()
InventoryServiceTest::direct_transfer_locks_warehouses_in_sorted_id_order()
InventoryServiceTest::direct_transfer_rejects_zero_or_negative_unit_ratio()
InventoryServiceTest::record_movement_rejects_zero_or_negative_unit_ratio()
InventoryServiceTest::scan_to_receive_is_idempotent_against_duplicate_submission()
InventoryServiceTest::scan_to_receive_total_financial_loss_matches_bcmath_reference_value()
ConcurrencyTest::simultaneous_opposite_direction_direct_transfers_do_not_deadlock()
ConcurrencyTest::simultaneous_sales_dispatch_against_same_variant_does_not_oversell()
ConcurrencyTest::transfer_dispatch_can_deplete_stock_reserved_by_a_confirmed_sales_order_pre_existing_behavior()
```

**Purchase Service:**
```
PurchaseServiceTest::order_throws_when_no_items()
PurchaseServiceTest::order_throws_when_not_draft()
PurchaseServiceTest::receive_purchase_supports_partial_batches()
PurchaseServiceTest::receive_purchase_rejects_over_receipt_beyond_ordered_qty()
PurchaseServiceTest::receive_purchase_updates_cost_price_when_flag_set()
PurchaseServiceTest::receive_purchase_does_not_update_cost_price_when_flag_unset()
PurchaseServiceTest::receive_purchase_skips_price_update_when_cost_unchanged()
PurchaseServiceTest::receive_purchase_sets_completed_when_fully_received()
PurchaseServiceTest::receive_purchase_sets_partially_received_when_incomplete()
PurchaseServiceTest::cancel_rejected_once_any_stock_received()
PurchaseServiceTest::cancel_succeeds_while_fully_unreceived()
PurchaseServiceTest::concurrent_receipts_with_cost_update_do_not_violate_is_current_uniqueness()
```

**Sales Service:**
```
SalesServiceTest::confirm_snapshots_sale_price_at_confirm_time_not_dispatch_time()
SalesServiceTest::confirm_throws_when_not_draft()
SalesServiceTest::dispatch_supports_partial_batches()
SalesServiceTest::dispatch_rejects_over_dispatch_beyond_ordered_qty()
SalesServiceTest::dispatch_rejects_when_on_hand_insufficient()
SalesServiceTest::dispatch_does_not_touch_reservedQuantity_transfers_scope()
SalesServiceTest::dispatch_sets_completed_when_fully_dispatched()
SalesServiceTest::dispatch_sets_partially_dispatched_when_incomplete()
SalesServiceTest::cancel_rejected_once_dispatch_has_begun()
SalesServiceTest::cancel_succeeds_while_draft_or_confirmed()
SalesServiceTest::sales_return_rejected_beyond_dispatched_qty()
SalesServiceTest::sales_return_creates_positive_sale_return_movement()
```

**Negotiation Service:**
```
NegotiationServiceTest::materialize_backfills_only_null_approved_base_qty()
NegotiationServiceTest::accept_is_idempotent_guard()
NegotiationServiceTest::accept_throws_on_confirmed_requisition()
NegotiationServiceTest::accept_throws_on_dispatched_requisition()
NegotiationServiceTest::accept_throws_on_partially_received_requisition()
NegotiationServiceTest::reject_throws_on_confirmed_requisition()
NegotiationServiceTest::reject_throws_on_dispatched_requisition()
NegotiationServiceTest::counter_throws_on_confirmed_requisition()
NegotiationServiceTest::counter_throws_on_side_mismatch()
NegotiationServiceTest::accept_succeeds_on_requested()
NegotiationServiceTest::accept_succeeds_on_under_review_fulfiller()
NegotiationServiceTest::accept_succeeds_on_under_review_requestor()
```

**Policies:**
```
PurchaseOrderPolicyTest::orderPurchase_allowed_only_when_draft()
PurchaseOrderPolicyTest::receivePurchase_allowed_only_when_ordered_or_partially_received()
PurchaseOrderPolicyTest::cancelPurchase_denied_once_any_item_received_even_when_status_allows_it()
PurchaseOrderPolicyTest::forceDelete_admin_only()
SalesOrderPolicyTest::confirmSalesOrder_allowed_only_when_draft()
SalesOrderPolicyTest::dispatchSale_allowed_only_when_confirmed_or_partially_dispatched()
SalesOrderPolicyTest::recordSalesReturn_denied_when_nothing_dispatched_yet()
SalesOrderPolicyTest::cancelSalesOrder_denied_once_dispatch_has_begun()
SupplierPolicyTest::delete_admin_only()
CustomerPolicyTest::delete_admin_only()
ProductPolicyTest::delete_denied_while_active_variants_exist()
ProductPolicyTest::delete_allowed_once_all_variants_trashed()
ProductVariantPolicyTest::forceDelete_always_false_regardless_of_role()
ProductVariantPolicyTest::adjustStock_admin_only()
TransferRequisitionPolicyTest::cancel_matches_the_exact_five_state_allowlist_no_more_no_less()
StockMovementPolicyTest::every_mutating_method_returns_false_regardless_of_admin_status()
InTransitPolicyTest::receive_delegates_to_parent_transfer_requisition_policy_not_a_separate_check()
LossLedgerPolicyTest::recordLoss_allowed_for_non_admin_users()
WarehousePolicyTest::create_admin_only_but_adjustStock_and_recordLoss_are_not()
UserPolicyTest::delete_denied_when_target_is_self_even_for_admin()
UserPolicyTest::update_allowed_for_self_even_when_not_admin()
```

**Observers & Ledgers:**
```
ProductObserverTest::soft_delete_blocked_when_active_children_exist()
LedgerIntegrityTest::force_delete_variant_is_restricted_by_db()
LedgerIntegrityTest::warehouse_delete_restricted_by_purchase_orders()
LedgerIntegrityTest::warehouse_delete_restricted_by_sales_orders()
LedgerIntegrityTest::force_delete_variant_restricted_by_purchase_order_items()
LedgerIntegrityTest::force_delete_variant_restricted_by_sales_order_items()
LossLedgerTest::snapshot_unit_cost_falls_back_to_zero_when_no_current_price_exists()
LossLedgerTest::snapshot_unit_cost_reflects_call_time_price_not_dispatch_time_price()
ScanReceiptControllerTest::signed_url_expires_after_seven_days()
```

**Widgets:**
```
LowStockAlertsWidgetTest::cache_window_prevents_requery_within_300_seconds()
LowStockAlertsWidgetTest::cache_miss_correctly_recomputes_all_variants()
LowStockAlertsWidgetTest::chart_has_correct_structure_with_labels_and_datasets()
LowStockAlertsWidgetTest::chart_type_is_bar()
LowStockAlertsWidgetTest::heading_is_set_correctly()
LowStockAlertsWidgetTest::non_admin_receives_empty_data_on_gate_check()
LowStockAlertsWidgetTest::sql_injection_rejected_in_computed_data()
LowStockAlertsWidgetTest::xss_script_escaped_in_variant_labels()
LowStockAlertsWidgetTest::warehouse_id_scoping_on_every_query()
LowStockAlertsWidgetTest::cross_warehouse_access_denial()
LowStockAlertsWidgetTest::generic_error_messages_no_internal_id_leaks()
LowStockAlertsWidgetTest::empty_warehouse_returns_empty_chart()
LowStockAlertsWidgetTest::variant_above_reorder_point_not_in_chart()
LowStockAlertsWidgetTest::multiple_variants_sorted_by_stock_ascending()
LowStockAlertsWidgetTest::confirmed_sales_reservations_reduce_available_quantity_for_reorder_check()
RecentMovementsWidgetTest::chart_data_is_cached_for_60_seconds()
RecentMovementsWidgetTest::cache_miss_correctly_recomputes_all_movements()
RecentMovementsWidgetTest::chart_has_correct_structure_with_labels_and_datasets()
RecentMovementsWidgetTest::chart_type_is_line()
RecentMovementsWidgetTest::heading_is_set_correctly()
RecentMovementsWidgetTest::cache_invalidated_on_stock_movement_with_warehouse_scoping()
RecentMovementsWidgetTest::non_admin_receives_empty_data_on_gate_check()
RecentMovementsWidgetTest::sql_injection_rejected_in_computed_data()
RecentMovementsWidgetTest::xss_script_escaped_in_movement_labels()
RecentMovementsWidgetTest::warehouse_id_scoping_on_every_query()
RecentMovementsWidgetTest::cross_warehouse_access_denial()
RecentMovementsWidgetTest::generic_error_messages_no_internal_id_leaks()
RecentMovementsWidgetTest::empty_warehouse_returns_empty_chart()
RecentMovementsWidgetTest::movements_aggregated_by_day_over_7_days()
SalesRevenueTrendWidgetTest::chart_data_cached_for_300_seconds()
SalesRevenueTrendWidgetTest::revenue_computed_via_bcmath_not_float_sum()
SalesRevenueTrendWidgetTest::warehouse_id_scoping_on_every_query()
SalesRevenueTrendWidgetTest::non_admin_receives_empty_data()
SalesRevenueTrendWidgetTest::empty_warehouse_returns_empty_chart()
TopSellingVariantsWidgetTest::ranks_by_dispatched_base_qty_descending()
TopSellingVariantsWidgetTest::limits_to_top_10()
TopSellingVariantsWidgetTest::excludes_cancelled_and_draft_orders()
SalesVsPurchasesWidgetTest::monthly_aggregates_match_bcmath_reference_values()
SalesVsPurchasesWidgetTest::six_month_window_boundary_is_inclusive()
SalesVsPurchasesWidgetTest::warehouse_id_scoping_on_both_aggregates()
PendingFulfillmentWidgetTest::counts_only_confirmed_and_partially_states()
PendingFulfillmentWidgetTest::excludes_cancelled_and_completed()
```

**Filters:**
```
AdminReviewFiltersTest::warehouse_filter_lists_all_warehouses_not_just_staff_assigned_ones()
AdminReviewFiltersTest::period_filter_today_matches_only_todays_records()
AdminReviewFiltersTest::period_filter_this_week_matches_records_within_current_week_boundaries()
AdminReviewFiltersTest::period_filter_this_month_matches_records_within_current_month_boundaries()
AdminReviewFiltersTest::period_filter_this_year_matches_records_within_current_year_boundaries()
AdminReviewFiltersTest::period_filter_specific_date_matches_only_that_date()
AdminReviewFiltersTest::period_filter_custom_range_is_inclusive_of_both_boundary_dates()
AdminReviewFiltersTest::period_filter_custom_range_with_only_from_set_is_open_ended()
AdminReviewFiltersTest::period_filter_custom_range_with_only_until_set_is_open_ended()
AdminReviewFiltersTest::period_filter_with_no_preset_selected_returns_unfiltered_query()
PurchaseOrdersTableTest::period_filter_hidden_from_non_admin_non_auditor_users()
SalesOrdersTableTest::period_filter_hidden_from_non_admin_non_auditor_users()
```

**Factories:**
```
FactoryTest::all_six_new_model_factories_produce_valid_persistable_records()
```

**Create Pages:**
```
CreatePurchaseOrderTest::creates_purchase_order_and_all_line_items_via_standard_relationship_repeater()
CreatePurchaseOrderTest::generates_reference_code_when_not_supplied()
CreateSalesOrderTest::creates_sales_order_and_all_line_items_via_standard_relationship_repeater()
CreateSalesOrderTest::leaves_unit_sale_price_snapshot_at_default_until_confirmed()
```

**Policy Audit:**
```
PolicyAuditTest::no_permission_or_role_check_exists_outside_a_policy_class_in_app_filament()
PolicyAuditTest::no_permission_or_role_check_exists_outside_a_policy_class_in_app_services()
```

### Playwright E2E Scenarios

1. **Full Transfer Lifecycle:** Admin creates requisition → fulfiller proposes counter-offer → requestor accepts → confirm → dispatch → scan-to-receive (partial) → scan-to-receive (final) → verify completed status and stock movements.
2. **Direct Transfer:** Create direct transfer → verify paired transfer_out/transfer_in movements in one commit.
3. **Loss Write-Off:** Record intra-warehouse loss via `RecordWarehouseLossAction` → verify LossLedger row with `transfer_requisition_id = NULL`.
4. **Soft-Delete Guard:** Attempt to soft-delete a Product with active variants → verify exception + UI guard.
5. **Authorization Bypass Attempt:** Invoke `ForceDeleteAction` via Livewire method call as non-admin → verify 403.
6. **Cancellation Boundary:** Attempt to invoke `CancelAction` on a `Dispatched` requisition via direct Livewire method call (bypassing UI `->visible()`) → verify the `->authorize('cancel')` policy still rejects it server-side.
7. **Duplicate Scan Submission:** Submit an identical scan-to-receive payload twice in rapid succession (simulating a mobile double-tap or retry) → verify only one set of stock movements and loss ledger rows is created.
8. **Negotiation Loop:** Requestor submits requisition → fulfiller opens review → fulfiller proposes counter-offer → requestor accepts counter → confirm → verify materialized approved_* fields → dispatch → verify stock movements match negotiated values.
9. **Purchase Lifecycle:** Create PO → order → receive partial → receive remaining → verify Completed status and stock movements.
10. **Sales Lifecycle:** Create SO → confirm → dispatch partial → dispatch remaining → verify Completed status and stock movements.
11. **Sales Cancellation Boundary:** Attempt `cancelSalesOrder` via direct Livewire method call on a Dispatched order → verify policy still rejects it server-side.

---

## 📌 Section 13: Deferred to v13

1. **Service-layer negotiation status guard** — `NegotiationService::accept()`, `reject()`, and `counter()` must verify the parent requisition is still in a negotiable status (`requested`, `under_review_fulfiller`, `under_review_requestor`). The UI layer guards via `->visible()`; the service layer currently relies on model-level `isResolved()` checks only. This remains a fragile implicit assumption — any future Artisan command, API endpoint, or queued job that calls these methods directly would bypass the guard silently.

**Implementation Specification (for v13):**

- **Custom Exception:** `NegotiationNotAllowedException` — thrown by guard with actionable message including requisition reference_code and current status.
- **Guard Method:** `NegotiationService::assertNegotiable(TransferRequisitionItemRevision $revision)` — called as first line in `accept()`, `reject()`, `counter()`.
  - Checks requisition status ∈ {Requested, UnderReviewFulfiller, UnderReviewRequestor}
  - Checks revision status = Pending
  - (Optional) Checks revision side matches current turn (Fulfiller turn = UnderReviewFulfiller, Requestor turn = UnderReviewRequestor)
- **Model-Level Defense:** `TransferRequisitionItemRevision::accept()` / `reject()` add `ensureCanTransitionTo()` checking `!isResolved()`.
- **Policy Ability:** Add `negotiate(User, TransferRequisition)` to `TransferRequisitionPolicy` mirroring status allowlist; wire to `->authorize('negotiate')` on all negotiation actions.
- **UI Wiring (Phase 09 completion):**
  - Table actions `acceptRevision` / `rejectRevision`: add `->action()` handlers calling service, `mountActionRecord` targeting first pending revision per requisition, `requiresConfirmation()`, success/error notifications.
  - Edit page header actions: per-revision `acceptRevision_{id}` / `rejectRevision_{id}` / `counterRevision_{id}` with `mountActionRecord($revision)`, using `RevisionsForm` for counter modal.
  - All actions guarded by `->authorize('negotiate')` and `->visible()` status allowlist.
- **Test Coverage (Phase 18):**
  - `NegotiationServiceTest`: 27 status-matrix tests (9 statuses × 3 methods), side-mismatch tests, non-pending revision tests.
  - `TransferRequisitionRevisionActionsTest`: table accept/reject, edit page header actions, counter modal, guard error notifications, approved_* field updates.

2. **Event + notification layer** — `InventoryBelowReorderPoint`, `TransferDispatched`, `TransferReceived`, `LossRecorded`, `PurchaseOrderReceived`, `SalesOrderDispatched` events for operational alerting. The `StatsOverviewWidget` (300s TTL) is not an alerting strategy.

3. **`[P0]` `->form()` vs `->schema()` on actions** — `->schema([...])` is the canonical v5 form. Audit all Actions for `->form()` calls; replace with `->schema()`.

4. **`[P0]` Placeholder replacement** — Replace `Placeholder` in wizard review steps with `WizardReviewStep` Livewire component. Verify against the pinned `^5.0` minor during Phase 05/08.

5. **`[P0]` `createOptionForm` auto-select behaviour** — Test inline Product create → variant Select auto-selects new Product; fix with `$refresh` if needed. Verify in Phase 05.

6. **Panel `->strictAuthorization()` role coverage** — enumerate every policy method before enabling strict mode.

7. **Low-stock widget scaling threshold** — the accepted-risk per-variant-loop-plus-cache approach (Section 10) should be revisited once `product_variants` count exceeds roughly 5,000–10,000 active rows, or if production APM shows cache-miss dashboard loads exceeding ~1–2 seconds. Upgrade path: single grouped-aggregate SQL query, as detailed in Section 10's accepted-risk note.

8. **Supplier shipment / transit tracking for purchases** — a shipping leg between supplier and warehouse with its own loss ledger, mirroring `in_transits`/`loss_ledgers`. Deferred per A7; do not retrofit into `loss_ledgers` (FK-scoped to `transfer_requisitions`).

9. **Purchase-side negotiation** (price counter-offers with a supplier) — no equivalent to `NegotiationService` is planned; POs are assumed pre-negotiated externally before entry.

10. **FIFO / weighted-average / lot-level COGS costing** — v1 sales use current `currentPrice.cost_price` for margin reporting if needed later; true lot-costing is a substantially larger change (per-movement cost layers) and is explicitly not in this blueprint.

11. **`PurchaseReturn` full workflow UI** — the movement type is defined for schema completeness but no resource/action is specified for triggering it in v1.

12. **Backorder auto-fulfillment** — when a `PartiallyDispatched` sales order's remaining qty becomes available, no automatic notification or fulfillment trigger is specified.

13. **Reporting-view decision** — separate Purchases/Sales tab on `StockMovementResource` vs. one mixed ledger — left as an open decision for Alvin, not resolved here.

14. **Hardening transfer dispatch to check `availableQuantity()` instead of `onHandQuantity()`** — this would be a breaking change to the `InventoryService`, out of this blueprint's scope by design.

15. **Applying `AdminReviewFilters` to `StockMovementsTable`/`LossLedgersTable`** — presented as an optional convenience, since it touches parent-blueprint files outside the additive-only scope.

16. **Filter-level authorization bypass hardening** — whether `AdminReviewFilters::warehouse()`/`period()` need their `->query()` closures to independently no-op for non-admin users, rather than relying solely on `->visible()` to hide the field from the DOM.

---

## ✅ Section 14: Cross-Cutting Verification Checklist

| Check | Status |
|---|---|
| reservedQuantity() counts Confirmed only, permanently and by design | ✅ |
| reservedQuantity() scope boundary is documented in-code, not just in prose | ✅ |
| reservedForSalesQuantity() is separate from reservedQuantity() and combined in availableQuantity() | ✅ |
| batchAvailableQuantity() issues exactly 3 queries regardless of variant count | ✅ |
| ForceDeleteAction absent from ProductResource | ✅ |
| All ledger product_variant_id FKs are restrictOnDelete | ✅ |
| stock_movements.notes column + service param | ✅ |
| loss_ledgers.transfer_requisition_id nullable | ✅ |
| partially_received has producer and consumer | ✅ |
| ConfirmAction calls materializeRequestedAsApproved() | ✅ |
| dispatchTransfer / scanToReceive free of ?? fallbacks | ✅ |
| dispatchTransfer throws if approved_base_qty null | ✅ |
| ScanToReceiveAction named scanToReceive (camelCase) | ✅ |
| All wizard step-review components are Placeholder | ✅ |
| RepeatableEntry (not RepeatEntry) in all infolists | ✅ |
| SoftDeletingScope imported in getEloquentQuery() | ✅ |
| Enums route getLabel() through __() | ✅ |
| Policies exist and are wired via ->authorize() | ✅ |
| ProductObserver guards parent soft-delete | ✅ |
| QR lifetime = 7 days | ✅ |
| Direct-transfer list uses type + related_movement_id | ✅ |
| All action namespaces = Filament\Actions\* (^5.0) | ✅ |
| ->recordActions() / ->toolbarActions() (v5, not v3) | ✅ |
| BulkActionGroup wraps multiple bulk actions | ✅ |
| Section/Grid/Wizard from Filament\Schemas\Components\* | ✅ |
| Get from Filament\Schemas\Components\Utilities\Get | ✅ |
| ->money(config('app.currency')) on all money columns | ✅ |
| $navigationGroup / $navigationSort specified per resource | ✅ |
| ->strictAuthorization() mandated in panel provider | ✅ |
| Phases 05/06 use inline actions, no RelationManagers | ✅ |
| Resource classes use thin delegation pattern (Schemas/, Tables/ subdirectories) | ✅ |
| Schema classes expose static configure() method | ✅ |
| getRecordRouteBindingEloquentQuery() overrides for soft-delete resources | ✅ |
| LossLedger model exists with snapshotUnitCostFrom() implemented | ✅ |
| directTransfer() locks warehouses in sorted-ID order | ✅ |
| recordMovement() and directTransfer() reject unit_ratio < 1 | ✅ |
| scanToReceive() no-ops on duplicate payload via state-equality check | ✅ |
| total_financial_loss computed via bcmul(), not float cast | ✅ |
| CancelAction restricted to five pre-dispatch states, both ->authorize() and ->visible() | ✅ |
| ext-bcmath declared as required PHP extension in composer.json | ✅ |
| LowStockAlertsWidget scaling risk explicitly documented as accepted, with upgrade path stated | ✅ (accepted risk, not a defect) |
| All Actions use `->schema()`, zero `->form()` calls | ✅ |
| Wizard review steps use `WizardReviewStep` component, not `Placeholder` | ✅ |
| `createOptionForm` auto-selects new option after save | ✅ |
| PurchaseOrderPolicy, SalesOrderPolicy, SupplierPolicy, CustomerPolicy exist and contain 100% of this addendum's permission/role logic | ✅ |
| No `->visible()` closure anywhere re-derives a permission decision instead of composing a policy call | ✅ |
| No Service method in PurchaseService/SalesService contains a role check — only data-integrity/state-machine guards | ✅ |
| PolicyAuditTest (or equivalent architecture/static check) exists and passes | ✅ |
| Parent v11.0's nine existing policies audited per Integration Point 9A | ✅ |
| AdminReviewFilters::warehouse() and period() are reused across all applicable resources | ✅ |
| AdminReviewFiltersTest covers all preset boundaries and edge cases | ✅ |
| batchAvailableQuantity() used in dispatchSale modal instead of per-item loop | ✅ |

---

## 📊 Section 15: Summary of All Changes

| # | Area | Resolution | Severity |
|---|---|---|---|
| 1 | Low-stock widget N+1 query risk at scale | Accepted as-is per explicit direction; documented with upgrade path and monitoring threshold | Medium (accepted risk) |
| 2 | `directTransfer()` missing warehouse lock ordering | Applied sorted-ID `lockForUpdate()` pattern | Critical |
| 3 | Signed-URL auth interaction | Re-verified, confirmed correctly handled | False alarm, closed |
| 4 | No idempotency guard on `scanToReceive()` duplicate submissions | Server-side state-equality no-op check added, plus audit trail table | Critical |
| 5 | `reservedQuantity()` scope boundary undocumented | Documented in-code as permanent design decision | High (docs gap) |
| 6 | `LossLedger::snapshotUnitCostFrom()` called but never defined | Fully implemented, call-time pricing confirmed as intended behavior | Critical (runtime-breaking) |
| 7 | `CancelAction` visibility too permissive | Restricted to five explicit pre-dispatch states, enforced in both policy and UI | High |
| 8 | `ForceDeleteAction` resource placement ambiguity | Confirmed as TransferRequisitionResource-only | Low, closed |
| 9 | `unit_ratio_used` accepts zero/negative values silently | Guard clause added to both `recordMovement()` and `directTransfer()` | Medium |
| 10 | `total_financial_loss` computed via lossy float cast | Replaced with `bcmul()`, `ext-bcmath` declared as required extension | Medium |
| 11 | Soft-deleted variant historical query behavior | Confirmed correct as-is | Low, closed |
| 12 | `NegotiationService` missing status guard on propose/accept/reject/counter | Remains explicitly deferred (already flagged, not a new gap) | Deferred, documented |
| 13 | `availableQuantity()` doesn't net out sales reservations | Added `reservedForSalesQuantity()` and combined in `availableQuantity()` | High |
| 14 | `dispatchSale` modal causes N+1 queries | Added `batchAvailableQuantity()` static method — 3 queries total | Medium |
| 15 | `AdminReviewFilters` duplicated across resources | Extracted to shared class with static factory methods | Low |
| 16 | `->money()` currency hardcoding in LossLedgerResource | Corrected to use `config('app.currency')` throughout v12 | Medium |
| 17 | Purchase/Sales not covered in parent blueprint | Fully merged as first-class modules with models, services, resources, policies, factories, and tests | Major feature addition |
| 18 | Policy consolidation system-wide | Principle A8 established; nine existing policies audited and consolidated | High |

---

*End of blueprint v12.0.*