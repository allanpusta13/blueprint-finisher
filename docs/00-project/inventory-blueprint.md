# Multi-Warehouse Inventory System — Complete System Blueprint (v13.3)

**Stack:** Laravel 13 + FilamentPHP v5 + Livewire v4 | **Database:** PostgreSQL / MySQL

**Architecture:** Pure Derived Stock of Truth Ledger + Purchases & Sales Module

> **Changelog from v13.2:**
> 1. **Fixed** sales dispatch self-reservation double-count — `reservedForSalesQuantity()` and `batchAvailableQuantity()` accept `$excludeSalesOrderId`.
> 2. **Fixed** in-transit clearing — added `cleared_at` column; `InTransit` rows transition to `Cleared` / `Lost` on scan completion.
> 3. **Fixed** first-scan detection — now based on `stock_movement_idempotency_keys` presence, not nonexistent `cleared_at` on `in_transits`.
> 4. **Fixed** purchase receive and sales dispatch stale-item races — items and variants re-locked under parent transaction.
> 5. **Fixed** nullable `approved_base_qty` race — `materializeRequestedAsApproved()` validates; reservation queries filter `whereNotNull`.
> 6. **Added** `ProductPolicy` and registered it.
> 7. **Added** `CustomerPolicy` (was described but not coded).
> 8. **Added** transfer dispatch availability guard — `dispatchTransfer()` now checks availability excluding own requisition reservation.
> 9. **Added** missing-cost warning on loss write-off via `Log::warning`.
> 10. **Fixed** `WarehousePolicy::delete()` — blocks warehouses referenced by PO/SO/TR, not just stock movements.
> 11. **Fixed** `recordSalesReturn()` lock gap — now locks `ProductVariant` and `Warehouse`.
> 12. **Added** Principle A10 — substitute variants are transfer-only.
> 13. **Fixed** badge query duplication — cached per-request via static.
> 14. **Added** `recordMovement()` guard rejecting purchase/sale movement types.
> 15. **Added** `created_at` indexes on `transfer_requisitions`, `purchase_orders`, `sales_orders`.
> 16. **Added** `ManageUnitConversionsAction` full implementation.
> 17. **Fixed** `AdminReviewFilters::period()` custom-range indicators — shows "No lower bound" / "No upper bound".
> 18. **Removed** bulk actions from card tables until `mkdev-grid-card-layout` is installed.
> 19. **Documented** `user_warehouse` pivot single source of truth — edited from `UserResource` only; `WarehouseForm` is read-only.

---

## 🧭 Section 0: Executive Architecture & System Principles

### Core Principles

1. **Pure Derived Stock of Truth.** Physical stock levels, active transit reservations, and available balances are never stored in a physical database table. Physical on-hand stock is calculated dynamically at query-time as the sum of all signed records in `stock_movements`. Active reservations sum pending quantities from confirmed requisitions, and available stock is derived as `on_hand - reserved`.

2. **Decoupled Pricing & Variant-Level Catalog.** `sku` lives exclusively on `product_variants`. Parent products act purely as family grouping containers. `reorder_point` lives exclusively on `product_variants`. Unit pricing is decoupled into `product_variant_prices` with `is_current = true`, supporting 4-decimal micro-pricing.

3. **Pessimistic Locking & Transaction Isolation.** All stock deductions, dispatches, and intake receipts execute inside atomic database transactions using pessimistic row-level locking on `product_variants`, `warehouses`, and `transfer_requisitions`. Locking discipline is uniform across every multi-warehouse-touching service method.

4. **Canonical Foreign Key & Plural Naming.** All database tables use explicit plural snake_case names. Foreign keys strictly follow table-bound names.

5. **Physical-to-Digital State Lifecycle.**
   ```
   draft → requested → under_review_fulfiller ⇌ under_review_requestor
         → confirmed → dispatched ⇌ partially_received
         → completed / closed_with_loss / cancelled
   ```
   `partially_received` is a first-class live state.

6. **Negotiated Substitute Variant Swapping.** Dispatch and receipt pipelines dynamically resolve `$actualVariantId = $item->substitute_product_variant_id ?? $item->product_variant_id`.

7. **Scanned Receipt Loss Integrity & Omitted Cargo.** On the first intake scan, dispatched items missing from a physical scan payload are recorded as 0 received, triggering a 100% variance write-off. On subsequent scans, omitted items are treated as still in transit. "First scan" is determined by the absence of any prior idempotency record for the requisition — not by any in-transit clearing timestamp.

8. **Signed Web QR Routing.** STN QR codes embed secure 7-day temporary signed URLs.

9. **Modal-First UI (< 8 Inputs Rule).** Compact operations use inline slide-over Drawers or Dialog Modals. Multi-step wizards use `modalWidth(Width::SevenExtraLarge)`.

10. **Strongly-Typed Icons, Multi-Language i18n & Currency.** All backed enums route `getLabel()` through `__()`. All `->money()` calls pass `config('app.currency')`. Every Action, resource, and navigation item uses a strongly-typed `Heroicon` enum.

11. **Ledger FK Immutability.** Every `product_variant_id` foreign key on a ledger table uses `restrictOnDelete`.

12. **Authorization vs Visibility.** `->authorize()` enforces server-side policy security. `->visible()` controls frontend DOM rendering. **`->visible()` never re-derives a permission decision.**

13. **Reservation Scope Boundary.** `reservedQuantity()` is intentionally and permanently bounded to requisitions in `Confirmed` status only.

14. **Cancellation Boundary.** `CancelAction` is only legal while a requisition is in a pre-dispatch state.

15. **Cost Snapshot Timing.** `LossLedger::snapshotUnitCostFrom()` captures `currentPrice.cost_price` at call-time, and logs a warning if cost is missing or zero.

16. **Table Shape Determines Presentation.** Document-shaped records (requisitions, purchase orders, sales orders) render as cards via `->contentGrid()`. Ledger-shaped records (stock movements, loss ledgers) render as dense, sortable rows via the standard table with `->stackedOnMobile()`. Master-data tables (suppliers, customers, warehouses, products) may render either way depending on cardinality and use case.

### Addendum Principles (Purchases & Sales)

**A1.** Same Ledger, New Movement Types. Purchases and sales are new `StockMovementType` cases.

**A2.** Purchases and Sales Are Symmetric, Single-Entity Flows. Each gets a lightweight draft → confirmed → completed / cancelled lifecycle.

**A3.** External Party Entities Are Minimal Master Data.

**A4.** Cost & Price Interplay. Received-purchase cost updates are opt-in via `update_cost_price`. Sales dispatch at confirm-time-snapshotted `sale_price`.

**A5.** Reservation Boundary Stays Untouched, Sales Get Their Own Boundary.

**A6.** No Reversal Pathway for Dispatched Sales.

**A7.** Purchases Have No "Loss" Concept at Intake.

**A8.** Policies Are the ONLY Home for Permission/Role Logic.

**A9.** Shared Filter Architecture for Admin Review.

**A10.** **Substitute Variants Are Transfer-Only.** `substitute_product_variant_id` is intentionally absent from `PurchaseOrderItem` and `SalesOrderItem`. Substitution is a first-class transfer/requisition feature only. Purchases and sales operate on the exact variant ordered/sold.

### Filament v5 Patterns

**F16.** Wizard-Based Create Pages Use `HasWizard` Trait.

**F17.** Relationship-Bound Repeaters Require `->dehydrated()` + Mutation Hooks.

**F18.** Units Are Variant-Scoped and Never Free-Text.

**F19.** Base-Unit "Self-Conversion" Row Required.

**F20.** Layout Components Are Composable. `Grid`, `Section`, `Fieldset`, `Tabs`, `Flex` — all support `columns()` / `columnSpan()`.

**F21.** Navigation Badges Are Live Status Indicators.

**F22.** Active Navigation Icons Reinforce State.

**F23.** Icons on Every Interactive Element.

**F24.** Responsive Column Spans Are Mandatory.

**F25.** Card Layout via `->contentGrid()`. Tables whose records are documents (not ledger rows) declare `->contentGrid(['md' => 2, 'xl' => 3])`, compose card internals with `Stack` and `Split`, and set `->defaultPaginationPageOption(12)`.

**F26.** Ledger Tables Are Never Carded. Append-only, high-volume ledgers always render as dense standard tables with `->stackedOnMobile()`. Signed quantities are color-coded. Pagination page size is ≥ 25.

**F27.** Every Table Declares `->defaultSort()`. Filament's implicit primary-key-ascending default is never correct for operational lists.

**F28.** Every Relational Column Is Eager-Loaded. Any column referencing `relation.attribute` requires the relation in `getEloquentQuery()`'s `->with()` list.

**F29.** Card Layout Requires Bounded Pagination. Any table with `->contentGrid()` declares `->defaultPaginationPageOption(12)` and `->paginated([12, 24, 48])`.

**F30.** **Card Tables Declare No Bulk Actions Until `mkdev-grid-card-layout` Is Installed.** The native renderer does not render per-card checkboxes; declaring bulk actions without the plugin produces inaccessible UI.

---

## 📁 Section 1: Filament v5 Resource Directory Structure

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
│   ├── Tables/
│   │   └── ProductsTable.php
│   └── Actions/
│       ├── SetCurrentPriceAction.php
│       ├── EditProductFamilyAction.php
│       ├── ManageUnitConversionsAction.php
│       └── QuickStockAdjustmentAction.php
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
│   │   ├── ListDirectTransfers.php
│   │   └── CreateDirectTransfer.php
│   ├── Schemas/
│   │   └── DirectTransferForm.php
│   └── Tables/
│       └── DirectTransfersTable.php
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
│   │   ├── EditWarehouse.php
│   │   └── ViewWarehouse.php
│   ├── Schemas/
│   │   ├── WarehouseForm.php
│   │   └── WarehouseInfolist.php
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
├── PurchaseOrders/
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
├── SalesOrders/
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
├── Suppliers/
│   ├── SupplierResource.php
│   ├── Pages/
│   │   ├── ListSuppliers.php
│   │   ├── CreateSupplier.php
│   │   └── EditSupplier.php
│   ├── Schemas/
│   │   └── SupplierForm.php
│   └── Tables/
│       └── SuppliersTable.php
│
└── Customers/
    ├── CustomerResource.php
    ├── Pages/
    │   ├── ListCustomers.php
    │   ├── CreateCustomer.php
    │   └── EditCustomer.php
    ├── Schemas/
    │   └── CustomerForm.php
    └── Tables/
        └── CustomersTable.php
```

### Thin Resource Class Pattern

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
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\SoftDeletingScope;

class ProductResource extends Resource
{
    protected static ?string $model = ProductVariant::class;
    protected static string | \UnitEnum | null $navigationGroup = 'CATALOG';
    protected static ?int $navigationSort = 1;
    protected static ?string $recordTitleAttribute = 'sku';
    protected static string | \BackedEnum | null $navigationIcon = Heroicon::OutlinedCube;
    protected static string | \BackedEnum | null $activeNavigationIcon = Heroicon::Cube;

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

### Navigation Group Registration

```php
->navigationGroups([
    NavigationGroup::make('CATALOG')->icon(Heroicon::CubeTransparent)->collapsible(),
    NavigationGroup::make('OPERATIONS')->icon(Heroicon::OutlinedRectangleStack)->collapsible(),
    NavigationGroup::make('PURCHASING')->icon(Heroicon::OutlinedShoppingCart)->collapsible(),
    NavigationGroup::make('SALES')->icon(Heroicon::OutlinedBanknotes)->collapsible(),
    NavigationGroup::make('AUDIT LEDGERS')->icon(Heroicon::QueueList)->collapsible(),
    NavigationGroup::make('SYSTEM ADMIN')->icon(Heroicon::BuildingOffice)->collapsible(false),
])
```

### Schema `configure()` Contract

```php
class ProductForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([...]);
    }
}
```

---

## 🧭 Section 1A: Navigation Groupings — Full Specification

### 1A.1 Centralized Group Registration

Navigation groups are registered once in `AdminPanelProvider::panel()` via `->navigationGroups()`. Array order is the sole determinant of group render order.

### 1A.2 Group Properties

| Method | Purpose |
|---|---|
| `->icon()` | Group heading icon (required for topbar dropdown) |
| `->collapsible()` / `->collapsible(false)` | Toggle collapsibility |
| `->collapsed()` | Collapse by default |
| `->label()` | Explicit label |

### 1A.3 Group Ordering Rules

1. By group array position in `navigationGroups()` — the only thing that matters.
2. By `$navigationSort` ascending within a group.
3. Alphabetically as fallback tiebreaker.

### 1A.4 Per-Resource Group Assignment

| Resource | `$navigationGroup` | `$navigationSort` | `$navigationIcon` | `$activeNavigationIcon` |
|---|---|---|---|---|
| `ProductResource` | `'CATALOG'` | `1` | `Heroicon::OutlinedCube` | `Heroicon::Cube` |
| `TransferRequisitionResource` | `'OPERATIONS'` | `1` | `Heroicon::OutlinedArrowsRightLeft` | `Heroicon::ArrowsRightLeft` |
| `DirectTransferResource` | `'OPERATIONS'` | `2` | `Heroicon::OutlinedArrowPath` | `Heroicon::ArrowPath` |
| `InTransitResource` | `'OPERATIONS'` | `3` | `Heroicon::OutlinedTruck` | `Heroicon::Truck` |
| `PurchaseOrderResource` | `'PURCHASING'` | `1` | `Heroicon::OutlinedShoppingCart` | `Heroicon::ShoppingCart` |
| `SupplierResource` | `'PURCHASING'` | `2` | `Heroicon::OutlinedBuildingStorefront` | `Heroicon::BuildingStorefront` |
| `SalesOrderResource` | `'SALES'` | `1` | `Heroicon::OutlinedBanknotes` | `Heroicon::Banknotes` |
| `CustomerResource` | `'SALES'` | `2` | `Heroicon::OutlinedUserGroup` | `Heroicon::UserGroup` |
| `StockMovementResource` | `'AUDIT LEDGERS'` | `1` | `Heroicon::OutlinedQueueList` | `Heroicon::QueueList` |
| `LossLedgerResource` | `'AUDIT LEDGERS'` | `2` | `Heroicon::OutlinedExclamationTriangle` | `Heroicon::ExclamationTriangle` |
| `WarehouseResource` | `'SYSTEM ADMIN'` | `1` | `Heroicon::OutlinedBuildingOffice` | `Heroicon::BuildingOffice` |
| `UserResource` | `'SYSTEM ADMIN'` | `2` | `Heroicon::OutlinedUsers` | `Heroicon::Users` |

### 1A.5 Resulting Sidebar Layout

```
CATALOG            → Products
OPERATIONS         → Transfer Requisitions, Direct Transfers, In-Transit Cargo
PURCHASING         → Purchase Orders, Suppliers
SALES              → Sales Orders, Customers
AUDIT LEDGERS      → Stock Movements, Loss Ledgers
SYSTEM ADMIN       → Warehouses, Users
```

### 1A.6 Group Icons

| Group | Heroicon |
|---|---|
| `CATALOG` | `Heroicon::CubeTransparent` |
| `OPERATIONS` | `Heroicon::OutlinedRectangleStack` |
| `PURCHASING` | `Heroicon::OutlinedShoppingCart` |
| `SALES` | `Heroicon::OutlinedBanknotes` |
| `AUDIT LEDGERS` | `Heroicon::QueueList` |
| `SYSTEM ADMIN` | `Heroicon::BuildingOffice` |

### 1A.7 Collapsibility Strategy

`SYSTEM ADMIN` → `->collapsible(false)`. All others → `->collapsible()`.

---

## 📛 Section 1B: Navigation Badges — Live Status Indicators

### 1B.1 Badge Principle (F21)

Badges are live status indicators. `getNavigationBadge()` returns `null` when count is zero. **Badges are warehouse-scoped** — a warehouse user sees only the count of documents in their warehouses.

### 1B.2 Badge Definitions Per Resource

| Resource | Badge Logic (warehouse-scoped) | Color Logic |
|---|---|---|
| `TransferRequisitionResource` | Count where `status = 'requested'` | `warning` > 10, else `primary` |
| `PurchaseOrderResource` | Count where `status = 'ordered'` | `warning` > 10, else `primary` |
| `SalesOrderResource` | Count where `status = 'confirmed'` | `warning` > 10, else `primary` |
| `InTransitResource` | Count where `status = 'in_transit'` | `primary` |
| All others | `null` | — |

### 1B.3 Badge Implementations

#### TransferRequisitionResource

```php
private static ?int $badgeCount = null;

private static function getScopedBadgeCount(): int
{
    if (self::$badgeCount === null) {
        $warehouseIds = auth()->user()->warehouses()->pluck('id');

        self::$badgeCount = static::getModel()::where('status', TransferRequisitionStatus::Requested->value)
            ->where(function ($q) use ($warehouseIds) {
                $q->whereIn('from_warehouse_id', $warehouseIds)
                  ->orWhereIn('to_warehouse_id', $warehouseIds);
            })
            ->count();
    }

    return self::$badgeCount;
}

public static function getNavigationBadge(): ?string
{
    $count = self::getScopedBadgeCount();
    return $count > 0 ? (string) $count : null;
}

public static function getNavigationBadgeColor(): ?string
{
    return self::getScopedBadgeCount() > 10 ? 'warning' : 'primary';
}

public static function getNavigationBadgeTooltip(): ?string
{
    return 'Requisitions awaiting fulfillment review';
}
```

#### PurchaseOrderResource

```php
private static ?int $badgeCount = null;

private static function getScopedBadgeCount(): int
{
    if (self::$badgeCount === null) {
        self::$badgeCount = static::getModel()::where('status', PurchaseOrderStatus::Ordered->value)
            ->whereIn('warehouse_id', auth()->user()->warehouses()->pluck('id'))
            ->count();
    }

    return self::$badgeCount;
}

public static function getNavigationBadge(): ?string
{
    $count = self::getScopedBadgeCount();
    return $count > 0 ? (string) $count : null;
}

public static function getNavigationBadgeColor(): ?string
{
    return self::getScopedBadgeCount() > 10 ? 'warning' : 'primary';
}

public static function getNavigationBadgeTooltip(): ?string
{
    return 'Purchase orders awaiting receipt';
}
```

#### SalesOrderResource

```php
private static ?int $badgeCount = null;

private static function getScopedBadgeCount(): int
{
    if (self::$badgeCount === null) {
        self::$badgeCount = static::getModel()::where('status', SalesOrderStatus::Confirmed->value)
            ->whereIn('warehouse_id', auth()->user()->warehouses()->pluck('id'))
            ->count();
    }

    return self::$badgeCount;
}

public static function getNavigationBadge(): ?string
{
    $count = self::getScopedBadgeCount();
    return $count > 0 ? (string) $count : null;
}

public static function getNavigationBadgeColor(): ?string
{
    return self::getScopedBadgeCount() > 10 ? 'warning' : 'primary';
}

public static function getNavigationBadgeTooltip(): ?string
{
    return 'Sales orders awaiting dispatch';
}
```

#### InTransitResource

```php
public static function getNavigationBadge(): ?string
{
    $count = static::getModel()::where('status', InTransitStatus::InTransit->value)->count();
    return $count > 0 ? (string) $count : null;
}

public static function getNavigationBadgeColor(): ?string
{
    return 'primary';
}

public static function getNavigationBadgeTooltip(): ?string
{
    return 'Active cargo currently in transit';
}
```

### 1B.4 Badge Performance Note

Badge queries run on every panel page load. All four status columns are indexed in Section 2. Badge closures are warehouse-scoped, use `pluck('id')` on the pivot relation, and are cached in a `private static ?int` per request, so `getNavigationBadge()` and `getNavigationBadgeColor()` share a single query.

---

## 🎯 Section 1C: Active Navigation Icons

Per Principle F22, every resource declares `$activeNavigationIcon` distinct from `$navigationIcon`. Convention: outlined for resting, solid for active.

### Full Active Icon Map

```php
// Products/ProductResource.php
$navigationIcon = Heroicon::OutlinedCube;
$activeNavigationIcon = Heroicon::Cube;

// TransferRequisitions/TransferRequisitionResource.php
$navigationIcon = Heroicon::OutlinedArrowsRightLeft;
$activeNavigationIcon = Heroicon::ArrowsRightLeft;

// DirectTransfers/DirectTransferResource.php
$navigationIcon = Heroicon::OutlinedArrowPath;
$activeNavigationIcon = Heroicon::ArrowPath;

// InTransits/InTransitResource.php
$navigationIcon = Heroicon::OutlinedTruck;
$activeNavigationIcon = Heroicon::Truck;

// PurchaseOrders/PurchaseOrderResource.php
$navigationIcon = Heroicon::OutlinedShoppingCart;
$activeNavigationIcon = Heroicon::ShoppingCart;

// Suppliers/SupplierResource.php
$navigationIcon = Heroicon::OutlinedBuildingStorefront;
$activeNavigationIcon = Heroicon::BuildingStorefront;

// SalesOrders/SalesOrderResource.php
$navigationIcon = Heroicon::OutlinedBanknotes;
$activeNavigationIcon = Heroicon::Banknotes;

// Customers/CustomerResource.php
$navigationIcon = Heroicon::OutlinedUserGroup;
$activeNavigationIcon = Heroicon::UserGroup;

// StockMovements/StockMovementResource.php
$navigationIcon = Heroicon::OutlinedQueueList;
$activeNavigationIcon = Heroicon::QueueList;

// LossLedgers/LossLedgerResource.php
$navigationIcon = Heroicon::OutlinedExclamationTriangle;
$activeNavigationIcon = Heroicon::ExclamationTriangle;

// Warehouses/WarehouseResource.php
$navigationIcon = Heroicon::OutlinedBuildingOffice;
$activeNavigationIcon = Heroicon::BuildingOffice;

// Users/UserResource.php
$navigationIcon = Heroicon::OutlinedUsers;
$activeNavigationIcon = Heroicon::Users;
```

---

## 🎨 Section 1D: Button, Form Field & Action Icons

Per Principle F23, every Action, form field, and interactive element carries a `Heroicon` enum icon, semantically matched.

### 1D.1 Action & Button Icons

#### TransferRequisitionResource

| Action | Icon | Color |
|---|---|---|
| `submitRequest` | `Heroicon::PaperAirplane` | `primary` |
| `reviewNegotiate` | `Heroicon::ChatBubbleLeftRight` | `warning` |
| `acceptRevision` | `Heroicon::CheckCircle` | `success` |
| `rejectRevision` | `Heroicon::XCircle` | `danger` |
| `confirm` | `Heroicon::CheckBadge` | `primary` |
| `dispatch` | `Heroicon::Truck` | `primary` |
| `scanToReceive` | `Heroicon::QrCode` | `success` |
| `recordLoss` | `Heroicon::ExclamationTriangle` | `danger` |
| `cancel` | `Heroicon::XMark` | `danger` |
| `EditAction` | `Heroicon::PencilSquare` | — |
| `DeleteAction` | `Heroicon::Trash` | `danger` |
| `RestoreAction` | `Heroicon::ArrowUturnLeft` | `warning` |
| `ForceDeleteAction` | `Heroicon::Trash` | `danger` |

#### PurchaseOrdersTable

| Action | Icon | Color |
|---|---|---|
| `orderPurchase` | `Heroicon::PaperAirplane` | `primary` |
| `receivePurchase` | `Heroicon::ArchiveBoxArrowDown` | `success` |
| `cancelPurchase` | `Heroicon::XMark` | `danger` |
| `EditAction` | `Heroicon::PencilSquare` | — |
| `DeleteAction` | `Heroicon::Trash` | `danger` |
| `RestoreAction` | `Heroicon::ArrowUturnLeft` | `warning` |
| `ForceDeleteAction` | `Heroicon::Trash` | `danger` |

#### SalesOrdersTable

| Action | Icon | Color |
|---|---|---|
| `confirmSalesOrder` | `Heroicon::CheckCircle` | `primary` |
| `dispatchSale` | `Heroicon::Truck` | `success` |
| `recordReturn` | `Heroicon::ArrowUturnLeft` | `warning` |
| `cancelSalesOrder` | `Heroicon::XMark` | `danger` |

#### ProductResource

| Action | Icon | Color |
|---|---|---|
| `SetCurrentPriceAction` | `Heroicon::CurrencyDollar` | `primary` |
| `EditProductFamilyAction` | `Heroicon::FolderOpen` | — |
| `ManageUnitConversionsAction` | `Heroicon::Scale` | — |
| `QuickStockAdjustmentAction` | `Heroicon::AdjustmentsHorizontal` | `warning` |

### 1D.2 Form Field Icons

| Field | Icon Type | Icon |
|---|---|---|
| `sku` | `prefixIcon` | `Heroicon::Tag` |
| `barcode` | `prefixIcon` | `Heroicon::QrCode` |
| `name` | `prefixIcon` | `Heroicon::Identification` |
| `base_unit_name` | `prefixIcon` | `Heroicon::Scale` |
| `reorder_point` | `prefixIcon` | `Heroicon::ExclamationTriangle` |
| `product_id` | `prefixIcon` | `Heroicon::FolderOpen` |
| `cost_price` / `sale_price` / `unit_cost_price` | `prefixIcon` | `Heroicon::CurrencyDollar` |
| `total_financial_loss` | `prefixIcon` | `Heroicon::ExclamationTriangle` |
| `from_warehouse_id` | `prefixIcon` | `Heroicon::BuildingOffice` |
| `to_warehouse_id` / `warehouse_id` | `prefixIcon` | `Heroicon::BuildingOffice2` |
| `supplier_id` | `prefixIcon` | `Heroicon::BuildingStorefront` |
| `customer_id` | `prefixIcon` | `Heroicon::UserGroup` |
| `*_qty` / `quantity` | `prefixIcon` | `Heroicon::Hashtag` |
| `*_unit_name` | `prefixIcon` | `Heroicon::Scale` |
| `*_unit_ratio` | `hintIcon` | `Heroicon::InformationCircle` |
| `notes` / `negotiation_reason` | `prefixIcon` | `Heroicon::ChatBubbleBottomCenterText` |
| `loss_category` | `prefixIcon` | `Heroicon::ExclamationTriangle` |
| `is_active` | `onIcon` / `offIcon` | `Heroicon::CheckCircle` / `Heroicon::XCircle` |
| `update_cost_price` | `onIcon` | `Heroicon::CurrencyDollar` |

### 1D.3 Section Header Icons

| Section | Icon |
|---|---|
| `REQUISITION PROFILE` | `Heroicon::DocumentText` |
| `AUTHORIZATION SIGN-OFFS` | `Heroicon::ShieldCheck` |
| `MATERIAL MANIFEST ITEMS` | `Heroicon::ClipboardDocumentList` |
| `PURCHASE ORDER PROFILE` | `Heroicon::DocumentText` |
| `SALES ORDER PROFILE` | `Heroicon::DocumentText` |
| `SIGN-OFFS` | `Heroicon::ShieldCheck` |
| `LINE ITEMS` | `Heroicon::ClipboardDocumentList` |
| `WAREHOUSE ROUTING` | `Heroicon::BuildingOffice` |
| `SUPPLIER & WAREHOUSE` | `Heroicon::BuildingStorefront` |
| `CUSTOMER & WAREHOUSE` | `Heroicon::UserGroup` |
| `STOCK ALLOCATION` | `Heroicon::Cube` |
| `IDENTITY` (product tabs) | `Heroicon::Identification` |
| `STOCK & PRICING` (product tabs) | `Heroicon::CurrencyDollar` |
| `UNIT CONVERSIONS` | `Heroicon::Scale` |
| `PRICING` | `Heroicon::CurrencyDollar` |
| `LOSS RECORD` | `Heroicon::ExclamationTriangle` |
| `FINANCIAL IMPACT` | `Heroicon::CurrencyDollar` |

### 1D.4 Icon Consistency Rules

1. Always `Heroicon` enum, never raw string.
2. Semantic match — `PaperAirplane` for submit, `Truck` for dispatch, etc.
3. No decorative icons.
4. Consistent action-to-icon mapping across resources.
5. Solid for active, outlined for resting (navigation).
6. Disabled/derived fields use `hintIcon`, not `prefixIcon`.

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
Constraints: At most one `is_current = true` row per variant.

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

**Invariants:** Base-unit self-conversion row required (F19), auto-created by observer, undeletable via UI.

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

Indexes: `status`, `created_at`, `(from_warehouse_id, to_warehouse_id)`

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
| cleared_at | timestamp | Yes | — |
| created_at / updated_at | timestamp | Yes | — |

Indexes: `(transfer_requisition_id, status)`

### 11. loss_ledgers

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | Yes | — |
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

### 12. users (altered)

| Column | Type | Nullable | Default |
|---|---|---|---|
| role | string | No | warehouse_staff |
| is_active | boolean | No | true |

### 13. user_warehouse (pivot)

| Column | Type | Nullable | Default |
|---|---|---|---|
| user_id | FK → users.id (cascadeOnDelete, part of composite PK) | No | — |
| warehouse_id | FK → warehouses.id (cascadeOnDelete, part of composite PK) | No | — |

Primary key: composite `(user_id, warehouse_id)`

**Editing rule:** Warehouse assignments are edited from `UserResource` only. `WarehouseForm` presents the pivot read-only to avoid last-write-wins conflicts.

### 14. suppliers

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

### 15. customers

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

### 16. purchase_orders

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

Indexes: `status`, `created_at`, `(supplier_id, warehouse_id)`

### 17. purchase_order_items

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

### 18. sales_orders

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

Indexes: `status`, `created_at`, `(customer_id, warehouse_id)`

### 19. sales_order_items

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

### 20. stock_movement_idempotency_keys

| Column | Type | Nullable | Default |
|---|---|---|---|
| id | bigint (PK) | No | — |
| transfer_requisition_id | FK → transfer_requisitions.id (cascadeOnDelete) | No | — |
| payload_checksum | string(64) | No | — |
| resulting_item_states | json | No | — |
| created_at | timestamp | No | current time |

Indexes: unique on `(transfer_requisition_id, payload_checksum)`

### New StockMovementType Cases

Add `Purchase`, `Sale`, `SaleReturn`, `PurchaseReturn`.

---

## 🛠️ Section 3: Model Layer

### 3.1 Product

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Product extends Model
{
    use SoftDeletes;

    protected $fillable = ['name', 'category'];

    public function variants(): \Illuminate\Database\Eloquent\Relations\HasMany
    {
        return $this->hasMany(ProductVariant::class);
    }

    protected static function booted(): void
    {
        static::deleting(function (Product $product) {
            if (! $product->isForceDeleting() && $product->variants()->exists()) {
                throw new \DomainException('Cannot soft-delete a product family that still has variants.');
            }
        });
    }
}
```

### 3.2 ProductVariant

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Database\Eloquent\SoftDeletes;

class ProductVariant extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'product_id', 'sku', 'barcode', 'name', 'base_unit_name',
        'reorder_point', 'attributes', 'images', 'is_active',
    ];

    protected $casts = [
        'attributes' => 'array',
        'images'     => 'array',
        'is_active'  => 'boolean',
    ];

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

    /**
     * Physical on-hand = sum of all signed stock_movements quantities.
     */
    public function onHandQuantity(?int $warehouseId = null): int
    {
        return (int) $this->stockMovements()
            ->when($warehouseId, fn (Builder $q) => $q->where('warehouse_id', $warehouseId))
            ->sum('quantity');
    }

    /**
     * Reserved = sum of pending quantities from Confirmed requisitions only.
     * Scope boundary is intentional and permanent (Principle 13).
     *
     * $excludeTransferRequisitionId excludes the requisition currently being
     * dispatched so its own outstanding qty is not counted against itself.
     */
    public function reservedQuantity(
        ?int $warehouseId = null,
        ?int $excludeTransferRequisitionId = null,
    ): int {
        return (int) TransferRequisitionItem::query()
            ->where('product_variant_id', $this->id)
            ->whereNotNull('approved_base_qty')
            ->whereHas('transferRequisition', function (Builder $q) use ($warehouseId, $excludeTransferRequisitionId) {
                $q->where('status', \App\Enums\TransferRequisitionStatus::Confirmed->value)
                  ->when($warehouseId, fn (Builder $qq) => $qq->where('from_warehouse_id', $warehouseId))
                  ->when($excludeTransferRequisitionId, fn (Builder $qq) => $qq->where('id', '!=', $excludeTransferRequisitionId));
            })
            ->sum('approved_base_qty');
    }

    /**
     * Sales reservation — separate from procurement reservation (A5).
     *
     * $excludeSalesOrderId excludes the sales order currently being dispatched
     * so its own outstanding qty is not counted against itself.
     */
    public function reservedForSalesQuantity(
        ?int $warehouseId = null,
        ?int $excludeSalesOrderId = null,
    ): int {
        return (int) SalesOrderItem::query()
            ->where('product_variant_id', $this->id)
            ->whereHas('salesOrder', function (Builder $q) use ($warehouseId, $excludeSalesOrderId) {
                $q->whereIn('status', [
                    \App\Enums\SalesOrderStatus::Confirmed->value,
                    \App\Enums\SalesOrderStatus::PartiallyDispatched->value,
                ])
                  ->when($warehouseId, fn (Builder $qq) => $qq->where('warehouse_id', $warehouseId))
                  ->when($excludeSalesOrderId, fn (Builder $qq) => $qq->where('id', '!=', $excludeSalesOrderId));
            })
            ->sum('base_qty');
    }

    /**
     * Available = on hand - reserved - reserved for sales.
     */
    public function availableQuantity(
        ?int $warehouseId = null,
        ?int $excludeSalesOrderId = null,
        ?int $excludeTransferRequisitionId = null,
    ): int {
        return $this->onHandQuantity($warehouseId)
            - $this->reservedQuantity($warehouseId, $excludeTransferRequisitionId)
            - $this->reservedForSalesQuantity($warehouseId, $excludeSalesOrderId);
    }

    /**
     * Batched available lookup — exactly 3 queries regardless of variant count.
     *
     * @param  array<int>  $variantIds
     * @return array<int, int>  variant_id => available_qty
     */
    public static function batchAvailableQuantity(
        array $variantIds,
        int $warehouseId,
        ?int $excludeSalesOrderId = null,
        ?int $excludeTransferRequisitionId = null,
    ): array {
        if (empty($variantIds)) {
            return [];
        }

        $onHand = StockMovement::query()
            ->selectRaw('product_variant_id, SUM(quantity) as total')
            ->whereIn('product_variant_id', $variantIds)
            ->where('warehouse_id', $warehouseId)
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id')
            ->map(fn ($v) => (int) $v)
            ->all();

        $reserved = TransferRequisitionItem::query()
            ->selectRaw('product_variant_id, SUM(approved_base_qty) as total')
            ->whereIn('product_variant_id', $variantIds)
            ->whereNotNull('approved_base_qty')
            ->whereHas('transferRequisition', fn (Builder $q) =>
                $q->where('status', \App\Enums\TransferRequisitionStatus::Confirmed->value)
                  ->where('from_warehouse_id', $warehouseId)
                  ->when($excludeTransferRequisitionId, fn (Builder $qq) => $qq->where('id', '!=', $excludeTransferRequisitionId)))
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id')
            ->map(fn ($v) => (int) $v)
            ->all();

        $salesReserved = SalesOrderItem::query()
            ->selectRaw('product_variant_id, SUM(base_qty) as total')
            ->whereIn('product_variant_id', $variantIds)
            ->whereHas('salesOrder', fn (Builder $q) =>
                $q->whereIn('status', [
                    \App\Enums\SalesOrderStatus::Confirmed->value,
                    \App\Enums\SalesOrderStatus::PartiallyDispatched->value,
                ])
                  ->where('warehouse_id', $warehouseId)
                  ->when($excludeSalesOrderId, fn (Builder $qq) => $qq->where('id', '!=', $excludeSalesOrderId)))
            ->groupBy('product_variant_id')
            ->pluck('total', 'product_variant_id')
            ->map(fn ($v) => (int) $v)
            ->all();

        $result = [];
        foreach ($variantIds as $id) {
            $result[$id] = ($onHand[$id] ?? 0)
                - ($reserved[$id] ?? 0)
                - ($salesReserved[$id] ?? 0);
        }

        return $result;
    }

    /**
     * Batched unit-conversion lookup — exactly 1 query.
     *
     * @param  array<int>  $variantIds
     * @return array<int, \Illuminate\Support\Collection>
     */
    public static function batchUnitConversions(array $variantIds): array
    {
        if (empty($variantIds)) {
            return [];
        }

        return ProductVariantUnitConversion::whereIn('product_variant_id', $variantIds)
            ->get()
            ->groupBy('product_variant_id')
            ->all();
    }
}
```

### 3.3 ProductVariantPrice

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ProductVariantPrice extends Model
{
    protected $fillable = [
        'product_variant_id', 'cost_price', 'sale_price',
        'effective_from', 'is_current', 'set_by', 'notes',
    ];

    protected $casts = [
        'cost_price'     => 'decimal:4',
        'sale_price'     => 'decimal:4',
        'effective_from' => 'datetime',
        'is_current'     => 'boolean',
    ];

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function setBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'set_by');
    }
}
```

### 3.4 ProductVariantUnitConversion

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class ProductVariantUnitConversion extends Model
{
    protected $fillable = [
        'product_variant_id', 'unit_name', 'base_unit_ratio',
        'is_default_purchase', 'is_default_transfer',
    ];

    protected $casts = [
        'base_unit_ratio'      => 'integer',
        'is_default_purchase'  => 'boolean',
        'is_default_transfer'  => 'boolean',
    ];

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function isBaseUnitRow(): bool
    {
        return $this->unit_name === $this->productVariant->base_unit_name
            && $this->base_unit_ratio === 1;
    }
}
```

### 3.5 Warehouse

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Warehouse extends Model
{
    protected $fillable = ['code', 'name', 'location', 'is_active'];

    protected $casts = ['is_active' => 'boolean'];

    public function users(): BelongsToMany
    {
        return $this->belongsToMany(User::class, 'user_warehouse');
    }

    public function stockMovements(): HasMany
    {
        return $this->hasMany(StockMovement::class);
    }

    public function transferRequisitionsFrom(): HasMany
    {
        return $this->hasMany(TransferRequisition::class, 'from_warehouse_id');
    }

    public function transferRequisitionsTo(): HasMany
    {
        return $this->hasMany(TransferRequisition::class, 'to_warehouse_id');
    }

    public function purchaseOrders(): HasMany
    {
        return $this->hasMany(PurchaseOrder::class);
    }

    public function salesOrders(): HasMany
    {
        return $this->hasMany(SalesOrder::class);
    }

    public function lossLedgers(): HasMany
    {
        return $this->hasMany(LossLedger::class);
    }
}
```

### 3.6 StockMovement

```php
namespace App\Models;

use App\Enums\StockMovementType;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class StockMovement extends Model
{
    protected $fillable = [
        'product_variant_id', 'warehouse_id', 'type', 'quantity',
        'unit_name_used', 'unit_ratio_used', 'related_movement_id',
        'reference_type', 'reference_id', 'reference_code',
        'notes', 'created_by',
    ];

    protected $casts = [
        'type'            => StockMovementType::class,
        'quantity'        => 'integer',
        'unit_ratio_used' => 'integer',
    ];

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function warehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class);
    }

    public function createdBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'created_by');
    }

    public function relatedMovement(): BelongsTo
    {
        return $this->belongsTo(self::class, 'related_movement_id');
    }
}
```

### 3.7 TransferRequisition

```php
namespace App\Models;

use App\Enums\TransferRequisitionStatus;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class TransferRequisition extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'reference_code', 'from_warehouse_id', 'to_warehouse_id', 'status',
        'requested_by', 'approved_by', 'dispatched_by', 'received_by',
        'requested_at', 'approved_at', 'dispatched_at', 'completed_at', 'notes',
    ];

    protected $casts = [
        'status'        => TransferRequisitionStatus::class,
        'requested_at'  => 'datetime',
        'approved_at'   => 'datetime',
        'dispatched_at' => 'datetime',
        'completed_at'  => 'datetime',
    ];

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

    /**
     * Pre-dispatch states only (Principle 14).
     */
    public function canBeCancelled(): bool
    {
        return in_array($this->status, [
            TransferRequisitionStatus::Draft,
            TransferRequisitionStatus::Requested,
            TransferRequisitionStatus::UnderReviewFulfiller,
            TransferRequisitionStatus::UnderReviewRequestor,
            TransferRequisitionStatus::Confirmed,
        ], true);
    }
}
```

### 3.8 TransferRequisitionItem

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class TransferRequisitionItem extends Model
{
    protected $fillable = [
        'transfer_requisition_id', 'product_variant_id', 'substitute_product_variant_id',
        'requested_unit_name', 'requested_unit_ratio', 'requested_qty', 'requested_base_qty',
        'approved_unit_name', 'approved_unit_ratio', 'approved_qty', 'approved_base_qty',
        'shipped_base_qty', 'received_good_base_qty', 'received_damaged_base_qty',
        'received_qty', 'notes',
    ];

    protected $casts = [
        'requested_unit_ratio'       => 'integer',
        'requested_qty'              => 'integer',
        'requested_base_qty'         => 'integer',
        'approved_unit_ratio'        => 'integer',
        'approved_qty'               => 'integer',
        'approved_base_qty'          => 'integer',
        'shipped_base_qty'           => 'integer',
        'received_good_base_qty'     => 'integer',
        'received_damaged_base_qty'  => 'integer',
        'received_qty'               => 'integer',
    ];

    public function transferRequisition(): BelongsTo
    {
        return $this->belongsTo(TransferRequisition::class);
    }

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function substituteProductVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class, 'substitute_product_variant_id');
    }

    public function revisions(): HasMany
    {
        return $this->hasMany(TransferRequisitionItemRevision::class);
    }

    public function actualVariantId(): int
    {
        return $this->substitute_product_variant_id ?? $this->product_variant_id;
    }

    public function outstandingShippedBaseQty(): int
    {
        return max(0, (int) $this->approved_base_qty - (int) $this->shipped_base_qty);
    }
}
```

### 3.9 TransferRequisitionItemRevision

```php
namespace App\Models;

use App\Enums\NegotiationSide;
use App\Enums\RevisionStatus;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class TransferRequisitionItemRevision extends Model
{
    protected $fillable = [
        'transfer_requisition_item_id', 'user_id', 'product_variant_id',
        'substitute_product_variant_id', 'proposed_unit_name', 'proposed_unit_ratio',
        'proposed_qty', 'proposed_base_qty', 'negotiation_reason', 'side',
        'status', 'responds_to_revision_id', 'responded_at',
    ];

    protected $casts = [
        'proposed_unit_ratio' => 'integer',
        'proposed_qty'        => 'integer',
        'proposed_base_qty'   => 'integer',
        'side'                => NegotiationSide::class,
        'status'              => RevisionStatus::class,
        'responded_at'        => 'datetime',
    ];

    public function item(): BelongsTo
    {
        return $this->belongsTo(TransferRequisitionItem::class, 'transfer_requisition_item_id');
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function respondsTo(): BelongsTo
    {
        return $this->belongsTo(self::class, 'responds_to_revision_id');
    }

    public function isResolved(): bool
    {
        return $this->status !== RevisionStatus::Pending;
    }

    public function ensureCanTransitionTo(RevisionStatus $target): void
    {
        if ($this->status !== RevisionStatus::Pending) {
            throw new \DomainException('Revision is already resolved.');
        }
        if ($target === RevisionStatus::Pending) {
            throw new \DomainException('Cannot transition a revision back to pending.');
        }
    }
}
```

### 3.10 InTransit

```php
namespace App\Models;

use App\Enums\InTransitStatus;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class InTransit extends Model
{
    protected $fillable = [
        'transfer_requisition_id', 'transfer_requisition_item_id',
        'product_variant_id', 'dispatched_base_qty', 'dispatched_at',
        'status', 'cleared_at',
    ];

    protected $casts = [
        'dispatched_base_qty' => 'integer',
        'dispatched_at'       => 'datetime',
        'status'              => InTransitStatus::class,
        'cleared_at'          => 'datetime',
    ];

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
}
```

### 3.11 LossLedger

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Support\Facades\Log;

class LossLedger extends Model
{
    protected $fillable = [
        'transfer_requisition_id', 'transfer_requisition_item_id',
        'product_variant_id', 'warehouse_id', 'lost_base_qty', 'damaged_base_qty',
        'unit_cost_price', 'total_financial_loss', 'loss_category',
        'notes', 'recorded_by', 'recorded_at',
    ];

    protected $casts = [
        'lost_base_qty'         => 'integer',
        'damaged_base_qty'      => 'integer',
        'unit_cost_price'       => 'decimal:4',
        'total_financial_loss'  => 'decimal:4',
        'recorded_at'           => 'datetime',
    ];

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

    public function recordedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'recorded_by');
    }

    /**
     * Snapshot the current cost price of the variant (Principle 15).
     *
     * If cost is missing or zero, log a warning — the loss will be recorded
     * at zero financial impact but flagged for review.
     */
    public static function snapshotUnitCostFrom(ProductVariant $variant): string
    {
        $cost = $variant->currentPrice?->cost_price;

        if ($cost === null || bccomp((string) $cost, '0.0000', 4) === 0) {
            Log::warning('Loss recorded with missing or zero cost price.', [
                'product_variant_id' => $variant->id,
                'sku'                => $variant->sku,
            ]);
            return '0.0000';
        }

        return (string) $cost;
    }

    /**
     * Compute total loss using BCMath to avoid float drift.
     */
    public static function calculateTotalFinancialLoss(string $unitCost, int $totalQty): string
    {
        return bcmul($unitCost, (string) $totalQty, 4);
    }
}
```

### 3.12 Supplier

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Supplier extends Model
{
    use SoftDeletes;

    protected $fillable = ['name', 'contact_person', 'phone', 'email', 'address', 'is_active'];

    protected $casts = ['is_active' => 'boolean'];

    public function purchaseOrders(): HasMany
    {
        return $this->hasMany(PurchaseOrder::class);
    }
}
```

### 3.13 Customer

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Customer extends Model
{
    use SoftDeletes;

    protected $fillable = ['name', 'contact_person', 'phone', 'email', 'address', 'is_active'];

    protected $casts = ['is_active' => 'boolean'];

    public function salesOrders(): HasMany
    {
        return $this->hasMany(SalesOrder::class);
    }
}
```

### 3.14 PurchaseOrder

```php
namespace App\Models;

use App\Enums\PurchaseOrderStatus;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class PurchaseOrder extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'reference_code', 'supplier_id', 'warehouse_id', 'status',
        'update_cost_price', 'ordered_by', 'received_by',
        'ordered_at', 'received_at', 'cancelled_at', 'notes',
    ];

    protected $casts = [
        'status'            => PurchaseOrderStatus::class,
        'update_cost_price' => 'boolean',
        'ordered_at'        => 'datetime',
        'received_at'       => 'datetime',
        'cancelled_at'      => 'datetime',
    ];

    public function supplier(): BelongsTo
    {
        return $this->belongsTo(Supplier::class);
    }

    public function warehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class);
    }

    public function orderedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'ordered_by');
    }

    public function receivedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'received_by');
    }

    public function items(): HasMany
    {
        return $this->hasMany(PurchaseOrderItem::class);
    }

    public function canBeCancelled(): bool
    {
        if (! in_array($this->status, [PurchaseOrderStatus::Draft, PurchaseOrderStatus::Ordered], true)) {
            return false;
        }

        return ! $this->items()->where('received_base_qty', '>', 0)->exists();
    }
}
```

### 3.15 PurchaseOrderItem

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class PurchaseOrderItem extends Model
{
    protected $fillable = [
        'purchase_order_id', 'product_variant_id', 'ordered_unit_name',
        'ordered_unit_ratio', 'ordered_qty', 'ordered_base_qty',
        'unit_cost_price', 'received_base_qty', 'notes',
    ];

    protected $casts = [
        'ordered_unit_ratio' => 'integer',
        'ordered_qty'        => 'integer',
        'ordered_base_qty'   => 'integer',
        'unit_cost_price'    => 'decimal:4',
        'received_base_qty'  => 'integer',
    ];

    public function purchaseOrder(): BelongsTo
    {
        return $this->belongsTo(PurchaseOrder::class);
    }

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function outstandingBaseQty(): int
    {
        return max(0, (int) $this->ordered_base_qty - (int) $this->received_base_qty);
    }
}
```

### 3.16 SalesOrder

```php
namespace App\Models;

use App\Enums\SalesOrderStatus;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class SalesOrder extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'reference_code', 'customer_id', 'warehouse_id', 'status',
        'ordered_by', 'dispatched_by',
        'ordered_at', 'confirmed_at', 'dispatched_at', 'cancelled_at', 'notes',
    ];

    protected $casts = [
        'status'        => SalesOrderStatus::class,
        'ordered_at'    => 'datetime',
        'confirmed_at'  => 'datetime',
        'dispatched_at' => 'datetime',
        'cancelled_at'  => 'datetime',
    ];

    public function customer(): BelongsTo
    {
        return $this->belongsTo(Customer::class);
    }

    public function warehouse(): BelongsTo
    {
        return $this->belongsTo(Warehouse::class);
    }

    public function orderedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'ordered_by');
    }

    public function dispatchedBy(): BelongsTo
    {
        return $this->belongsTo(User::class, 'dispatched_by');
    }

    public function items(): HasMany
    {
        return $this->hasMany(SalesOrderItem::class);
    }
}
```

### 3.17 SalesOrderItem

```php
namespace App\Models;

use App\Enums\StockMovementType;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class SalesOrderItem extends Model
{
    protected $fillable = [
        'sales_order_id', 'product_variant_id', 'unit_name', 'unit_ratio',
        'qty', 'base_qty', 'unit_sale_price_snapshot', 'dispatched_base_qty', 'notes',
    ];

    protected $casts = [
        'unit_ratio'                => 'integer',
        'qty'                       => 'integer',
        'base_qty'                  => 'integer',
        'unit_sale_price_snapshot'  => 'decimal:4',
        'dispatched_base_qty'       => 'integer',
    ];

    public function salesOrder(): BelongsTo
    {
        return $this->belongsTo(SalesOrder::class);
    }

    public function productVariant(): BelongsTo
    {
        return $this->belongsTo(ProductVariant::class);
    }

    public function outstandingBaseQty(): int
    {
        return max(0, (int) $this->base_qty - (int) $this->dispatched_base_qty);
    }

    public function alreadyReturnedBaseQty(): int
    {
        return (int) StockMovement::query()
            ->where('type', StockMovementType::SaleReturn->value)
            ->where('reference_type', self::class)
            ->where('reference_id', (string) $this->id)
            ->sum('quantity');
    }
}
```

### 3.18 User

```php
namespace App\Models;

use App\Enums\UserRole;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class User extends Authenticatable
{
    protected $fillable = ['name', 'email', 'password', 'role', 'is_active'];

    protected $casts = [
        'role'      => UserRole::class,
        'is_active' => 'boolean',
    ];

    public function warehouses(): BelongsToMany
    {
        return $this->belongsToMany(Warehouse::class, 'user_warehouse');
    }

    public function isAdmin(): bool
    {
        return $this->role === UserRole::Admin;
    }

    public function isAuditor(): bool
    {
        return $this->role === UserRole::Auditor;
    }

    public function isWarehouseStaff(): bool
    {
        return $this->role === UserRole::WarehouseStaff;
    }
}
```

### 3.19 Observers

#### ProductObserver

```php
namespace App\Observers;

use App\Models\Product;

class ProductObserver
{
    public function deleting(Product $product): void
    {
        if ($product->isForceDeleting()) {
            return;
        }
        if ($product->variants()->exists()) {
            throw new \DomainException('Cannot soft-delete a product family with variants.');
        }
    }
}
```

#### ProductVariantObserver

```php
namespace App\Observers;

use App\Models\ProductVariant;
use App\Models\ProductVariantUnitConversion;

class ProductVariantObserver
{
    public function created(ProductVariant $variant): void
    {
        ProductVariantUnitConversion::firstOrCreate(
            [
                'product_variant_id' => $variant->id,
                'unit_name'          => $variant->base_unit_name,
            ],
            [
                'base_unit_ratio'      => 1,
                'is_default_purchase'  => false,
                'is_default_transfer'  => false,
            ],
        );
    }
}
```

Registration in `AppServiceProvider::boot()`:
```php
Product::observe(ProductObserver::class);
ProductVariant::observe(ProductVariantObserver::class);
```

---

## 🏷️ Section 4: Enums

### 4.1 TransferRequisitionStatus

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;
use Filament\Support\Contracts\HasColor;

enum TransferRequisitionStatus: string implements HasLabel, HasColor
{
    case Draft                  = 'draft';
    case Requested              = 'requested';
    case UnderReviewFulfiller   = 'under_review_fulfiller';
    case UnderReviewRequestor   = 'under_review_requestor';
    case Confirmed              = 'confirmed';
    case Dispatched             = 'dispatched';
    case PartiallyReceived      = 'partially_received';
    case Completed              = 'completed';
    case ClosedWithLoss         = 'closed_with_loss';
    case Cancelled              = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft                => __('Draft'),
            self::Requested            => __('Requested'),
            self::UnderReviewFulfiller => __('Under Review (Fulfiller)'),
            self::UnderReviewRequestor => __('Under Review (Requestor)'),
            self::Confirmed            => __('Confirmed'),
            self::Dispatched           => __('Dispatched'),
            self::PartiallyReceived    => __('Partially Received'),
            self::Completed            => __('Completed'),
            self::ClosedWithLoss       => __('Closed with Loss'),
            self::Cancelled            => __('Cancelled'),
        };
    }

    public function getColor(): string
    {
        return match ($this) {
            self::Draft                => 'gray',
            self::Requested            => 'warning',
            self::UnderReviewFulfiller,
            self::UnderReviewRequestor => 'warning',
            self::Confirmed            => 'primary',
            self::Dispatched           => 'info',
            self::PartiallyReceived    => 'warning',
            self::Completed            => 'success',
            self::ClosedWithLoss       => 'danger',
            self::Cancelled            => 'danger',
        };
    }
}
```

### 4.2 PurchaseOrderStatus

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;
use Filament\Support\Contracts\HasColor;

enum PurchaseOrderStatus: string implements HasLabel, HasColor
{
    case Draft              = 'draft';
    case Ordered            = 'ordered';
    case PartiallyReceived  = 'partially_received';
    case Received           = 'received';
    case Cancelled          = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft             => __('Draft'),
            self::Ordered           => __('Ordered'),
            self::PartiallyReceived => __('Partially Received'),
            self::Received          => __('Received'),
            self::Cancelled         => __('Cancelled'),
        };
    }

    public function getColor(): string
    {
        return match ($this) {
            self::Draft             => 'gray',
            self::Ordered           => 'primary',
            self::PartiallyReceived => 'warning',
            self::Received          => 'success',
            self::Cancelled         => 'danger',
        };
    }
}
```

### 4.3 SalesOrderStatus

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;
use Filament\Support\Contracts\HasColor;

enum SalesOrderStatus: string implements HasLabel, HasColor
{
    case Draft                = 'draft';
    case Confirmed            = 'confirmed';
    case PartiallyDispatched  = 'partially_dispatched';
    case Dispatched           = 'dispatched';
    case Cancelled            = 'cancelled';

    public function getLabel(): string
    {
        return match ($this) {
            self::Draft               => __('Draft'),
            self::Confirmed           => __('Confirmed'),
            self::PartiallyDispatched => __('Partially Dispatched'),
            self::Dispatched          => __('Dispatched'),
            self::Cancelled           => __('Cancelled'),
        };
    }

    public function getColor(): string
    {
        return match ($this) {
            self::Draft               => 'gray',
            self::Confirmed           => 'primary',
            self::PartiallyDispatched => 'warning',
            self::Dispatched          => 'success',
            self::Cancelled           => 'danger',
        };
    }
}
```

### 4.4 StockMovementType

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;

enum StockMovementType: string implements HasLabel
{
    case Adjustment        = 'adjustment';
    case TransferOut       = 'transfer_out';
    case TransferIn        = 'transfer_in';
    case Loss              = 'loss';
    case Damage            = 'damage';
    case Purchase          = 'purchase';
    case Sale              = 'sale';
    case SaleReturn        = 'sale_return';
    case PurchaseReturn    = 'purchase_return';

    public function getLabel(): string
    {
        return match ($this) {
            self::Adjustment     => __('Adjustment'),
            self::TransferOut    => __('Transfer Out'),
            self::TransferIn     => __('Transfer In'),
            self::Loss           => __('Loss'),
            self::Damage         => __('Damage'),
            self::Purchase       => __('Purchase'),
            self::Sale           => __('Sale'),
            self::SaleReturn     => __('Sale Return'),
            self::PurchaseReturn => __('Purchase Return'),
        };
    }

    public function isPositive(): bool
    {
        return in_array($this, [self::TransferIn, self::Purchase, self::SaleReturn], true);
    }
}
```

### 4.5 RevisionStatus

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;

enum RevisionStatus: string implements HasLabel
{
    case Pending  = 'pending';
    case Accepted = 'accepted';
    case Rejected = 'rejected';

    public function getLabel(): string
    {
        return match ($this) {
            self::Pending  => __('Pending'),
            self::Accepted => __('Accepted'),
            self::Rejected => __('Rejected'),
        };
    }
}
```

### 4.6 NegotiationSide

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;

enum NegotiationSide: string implements HasLabel
{
    case Fulfiller = 'fulfiller';
    case Requestor = 'requestor';

    public function getLabel(): string
    {
        return match ($this) {
            self::Fulfiller => __('Fulfiller'),
            self::Requestor => __('Requestor'),
        };
    }
}
```

### 4.7 InTransitStatus

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;

enum InTransitStatus: string implements HasLabel
{
    case InTransit = 'in_transit';
    case Cleared   = 'cleared';
    case Lost      = 'lost';

    public function getLabel(): string
    {
        return match ($this) {
            self::InTransit => __('In Transit'),
            self::Cleared   => __('Cleared'),
            self::Lost      => __('Lost'),
        };
    }
}
```

### 4.8 UserRole

```php
namespace App\Enums;

use Filament\Support\Contracts\HasLabel;

enum UserRole: string implements HasLabel
{
    case Admin          = 'admin';
    case Auditor        = 'auditor';
    case WarehouseStaff = 'warehouse_staff';

    public function getLabel(): string
    {
        return match ($this) {
            self::Admin          => __('Admin'),
            self::Auditor        => __('Auditor'),
            self::WarehouseStaff => __('Warehouse Staff'),
        };
    }
}
```

---

## 🏭 Section 5: Model Factories

### 5.1 ProductFactory

```php
namespace Database\Factories;

use App\Models\Product;
use Illuminate\Database\Eloquent\Factories\Factory;

class ProductFactory extends Factory
{
    protected $model = Product::class;

    public function definition(): array
    {
        return [
            'name'     => $this->faker->words(3, true),
            'category' => $this->faker->randomElement(['Electronics', 'Hardware', 'Consumables']),
        ];
    }
}
```

### 5.2 ProductVariantFactory

```php
namespace Database\Factories;

use App\Models\Product;
use App\Models\ProductVariant;
use Illuminate\Database\Eloquent\Factories\Factory;

class ProductVariantFactory extends Factory
{
    protected $model = ProductVariant::class;

    public function definition(): array
    {
        return [
            'product_id'     => Product::factory(),
            'sku'            => strtoupper($this->faker->unique()->bothify('SKU-####-??')),
            'barcode'        => $this->faker->unique()->ean13(),
            'name'           => $this->faker->words(2, true),
            'base_unit_name' => 'pc',
            'reorder_point'  => $this->faker->numberBetween(0, 50),
            'is_active'      => true,
        ];
    }
}
```

### 5.3 ProductVariantPriceFactory

```php
namespace Database\Factories;

use App\Models\ProductVariant;
use App\Models\ProductVariantPrice;
use Illuminate\Database\Eloquent\Factories\Factory;

class ProductVariantPriceFactory extends Factory
{
    protected $model = ProductVariantPrice::class;

    public function definition(): array
    {
        $cost = $this->faker->randomFloat(4, 1, 500);

        return [
            'product_variant_id' => ProductVariant::factory(),
            'cost_price'         => $cost,
            'sale_price'         => $cost * 1.4,
            'effective_from'     => now(),
            'is_current'         => true,
        ];
    }
}
```

### 5.4 ProductVariantUnitConversionFactory

```php
namespace Database\Factories;

use App\Models\ProductVariant;
use App\Models\ProductVariantUnitConversion;
use Illuminate\Database\Eloquent\Factories\Factory;

class ProductVariantUnitConversionFactory extends Factory
{
    protected $model = ProductVariantUnitConversion::class;

    public function definition(): array
    {
        return [
            'product_variant_id'  => ProductVariant::factory(),
            'unit_name'           => $this->faker->randomElement(['box', 'case', 'pallet']),
            'base_unit_ratio'     => $this->faker->randomElement([6, 12, 24, 48]),
            'is_default_purchase' => false,
            'is_default_transfer' => false,
        ];
    }

    public function baseUnit(): static
    {
        return $this->state(fn () => [
            'unit_name'       => 'pc',
            'base_unit_ratio' => 1,
        ]);
    }
}
```

### 5.5 WarehouseFactory

```php
namespace Database\Factories;

use App\Models\Warehouse;
use Illuminate\Database\Eloquent\Factories\Factory;

class WarehouseFactory extends Factory
{
    protected $model = Warehouse::class;

    public function definition(): array
    {
        return [
            'code'      => strtoupper($this->faker->unique()->bothify('WH-####')),
            'name'      => $this->faker->city() . ' Warehouse',
            'location'  => $this->faker->address(),
            'is_active' => true,
        ];
    }
}
```

### 5.6 UserFactory

```php
namespace Database\Factories;

use App\Enums\UserRole;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Facades\Hash;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name'      => $this->faker->name(),
            'email'     => $this->faker->unique()->safeEmail(),
            'password'  => Hash::make('password'),
            'role'      => UserRole::WarehouseStaff,
            'is_active' => true,
        ];
    }

    public function admin(): static
    {
        return $this->state(fn () => ['role' => UserRole::Admin]);
    }

    public function auditor(): static
    {
        return $this->state(fn () => ['role' => UserRole::Auditor]);
    }
}
```

### 5.7 SupplierFactory

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
            'name'           => $this->faker->company(),
            'contact_person' => $this->faker->name(),
            'phone'          => $this->faker->phoneNumber(),
            'email'          => $this->faker->companyEmail(),
            'address'        => $this->faker->address(),
            'is_active'      => true,
        ];
    }
}
```

### 5.8 CustomerFactory

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
            'name'           => $this->faker->company(),
            'contact_person' => $this->faker->name(),
            'phone'          => $this->faker->phoneNumber(),
            'email'          => $this->faker->companyEmail(),
            'address'        => $this->faker->address(),
            'is_active'      => true,
        ];
    }
}
```

### 5.9 TransferRequisitionFactory

```php
namespace Database\Factories;

use App\Enums\TransferRequisitionStatus;
use App\Models\TransferRequisition;
use App\Models\User;
use App\Models\Warehouse;
use Illuminate\Database\Eloquent\Factories\Factory;

class TransferRequisitionFactory extends Factory
{
    protected $model = TransferRequisition::class;

    public function definition(): array
    {
        return [
            'reference_code'   => 'TR-' . now()->format('YmdHis') . '-' . $this->faker->numberBetween(100, 999),
            'from_warehouse_id' => Warehouse::factory(),
            'to_warehouse_id'   => Warehouse::factory(),
            'status'            => TransferRequisitionStatus::Draft,
            'requested_by'      => User::factory(),
        ];
    }

    public function requested(): static
    {
        return $this->state(fn () => [
            'status'       => TransferRequisitionStatus::Requested,
            'requested_at' => now(),
        ]);
    }

    public function confirmed(): static
    {
        return $this->state(fn () => [
            'status'      => TransferRequisitionStatus::Confirmed,
            'approved_at' => now(),
            'approved_by' => User::factory(),
        ]);
    }

    public function dispatched(): static
    {
        return $this->state(fn () => [
            'status'        => TransferRequisitionStatus::Dispatched,
            'dispatched_at' => now(),
            'dispatched_by' => User::factory(),
        ]);
    }
}
```

### 5.10 TransferRequisitionItemFactory

```php
namespace Database\Factories;

use App\Models\ProductVariant;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItem;
use Illuminate\Database\Eloquent\Factories\Factory;

class TransferRequisitionItemFactory extends Factory
{
    protected $model = TransferRequisitionItem::class;

    public function definition(): array
    {
        $qty = $this->faker->numberBetween(1, 20);

        return [
            'transfer_requisition_id' => TransferRequisition::factory(),
            'product_variant_id'      => ProductVariant::factory(),
            'requested_unit_name'     => 'pc',
            'requested_unit_ratio'    => 1,
            'requested_qty'           => $qty,
            'requested_base_qty'      => $qty,
        ];
    }
}
```

### 5.11 InTransitFactory

```php
namespace Database\Factories;

use App\Enums\InTransitStatus;
use App\Models\InTransit;
use App\Models\ProductVariant;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItem;
use Illuminate\Database\Eloquent\Factories\Factory;

class InTransitFactory extends Factory
{
    protected $model = InTransit::class;

    public function definition(): array
    {
        return [
            'transfer_requisition_id'      => TransferRequisition::factory(),
            'transfer_requisition_item_id' => TransferRequisitionItem::factory(),
            'product_variant_id'           => ProductVariant::factory(),
            'dispatched_base_qty'          => $this->faker->numberBetween(1, 50),
            'dispatched_at'                => now(),
            'status'                       => InTransitStatus::InTransit,
        ];
    }
}
```

### 5.12 LossLedgerFactory

```php
namespace Database\Factories;

use App\Models\LossLedger;
use App\Models\ProductVariant;
use App\Models\TransferRequisition;
use App\Models\Warehouse;
use Illuminate\Database\Eloquent\Factories\Factory;

class LossLedgerFactory extends Factory
{
    protected $model = LossLedger::class;

    public function definition(): array
    {
        $lost = $this->faker->numberBetween(0, 10);
        $damaged = $this->faker->numberBetween(0, 5);
        $unitCost = $this->faker->randomFloat(4, 1, 100);

        return [
            'transfer_requisition_id' => TransferRequisition::factory(),
            'product_variant_id'      => ProductVariant::factory(),
            'warehouse_id'            => Warehouse::factory(),
            'lost_base_qty'           => $lost,
            'damaged_base_qty'        => $damaged,
            'unit_cost_price'         => $unitCost,
            'total_financial_loss'    => bcmul((string) $unitCost, (string) ($lost + $damaged), 4),
            'loss_category'           => $this->faker->randomElement(['shortfall', 'damage', 'spoilage', 'theft', 'other']),
            'recorded_at'             => now(),
        ];
    }
}
```

### 5.13 PurchaseOrderFactory

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
            'reference_code' => 'PO-' . now()->format('YmdHis') . '-' . $this->faker->numberBetween(100, 999),
            'supplier_id'    => Supplier::factory(),
            'warehouse_id'   => Warehouse::factory(),
            'status'         => PurchaseOrderStatus::Draft,
            'ordered_by'     => User::factory(),
        ];
    }

    public function ordered(): static
    {
        return $this->state(fn () => [
            'status'     => PurchaseOrderStatus::Ordered,
            'ordered_at' => now(),
        ]);
    }
}
```

### 5.14 PurchaseOrderItemFactory

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
        $qty = $this->faker->numberBetween(1, 20);

        return [
            'purchase_order_id'  => PurchaseOrder::factory(),
            'product_variant_id' => ProductVariant::factory(),
            'ordered_unit_name'  => 'pc',
            'ordered_unit_ratio' => 1,
            'ordered_qty'        => $qty,
            'ordered_base_qty'   => $qty,
            'unit_cost_price'    => $this->faker->randomFloat(4, 1, 500),
        ];
    }
}
```

### 5.15 SalesOrderFactory

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
            'reference_code' => 'SO-' . now()->format('YmdHis') . '-' . $this->faker->numberBetween(100, 999),
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
}
```

### 5.16 SalesOrderItemFactory

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
        $qty = $this->faker->numberBetween(1, 20);

        return [
            'sales_order_id'            => SalesOrder::factory(),
            'product_variant_id'        => ProductVariant::factory(),
            'unit_name'                 => 'pc',
            'unit_ratio'                => 1,
            'qty'                       => $qty,
            'base_qty'                  => $qty,
            'unit_sale_price_snapshot'  => $this->faker->randomFloat(4, 1, 500),
        ];
    }
}
```

---

## ⚙️ Section 6: Transactional Service Layer

### 6.1 GuardsOutstandingQuantity

```php
namespace App\Services;

use App\Models\PurchaseOrderItem;
use App\Models\SalesOrderItem;

class GuardsOutstandingQuantity
{
    public function assertPurchaseNotOverReceived(PurchaseOrderItem $item, int $newReceived): void
    {
        $outstanding = $item->outstandingBaseQty();
        if ($newReceived > $outstanding) {
            throw new \DomainException(
                "Cannot receive {$newReceived} base units; outstanding is {$outstanding}."
            );
        }
    }

    public function assertSaleNotOverDispatched(SalesOrderItem $item, int $newDispatch): void
    {
        $outstanding = $item->outstandingBaseQty();
        if ($newDispatch > $outstanding) {
            throw new \DomainException(
                "Cannot dispatch {$newDispatch} base units; outstanding is {$outstanding}."
            );
        }
    }
}
```

### 6.2 InventoryService

```php
namespace App\Services;

use App\Enums\InTransitStatus;
use App\Enums\StockMovementType;
use App\Models\InTransit;
use App\Models\LossLedger;
use App\Models\ProductVariant;
use App\Models\StockMovement;
use App\Models\StockMovementIdempotencyKey;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItem;
use App\Models\Warehouse;
use Illuminate\Support\Facades\DB;

class InventoryService
{
    public function __construct(
        private readonly GuardsOutstandingQuantity $guards,
    ) {}

    /**
     * Record a signed stock movement inside a lock.
     *
     * Restricted: purchase/sale/sale_return/purchase_return movements must
     * go through PurchaseService / SalesService so their guards apply.
     */
    public function recordMovement(
        int $productVariantId,
        int $warehouseId,
        StockMovementType $type,
        int $baseQuantity,
        string $unitName,
        int $unitRatio,
        ?string $referenceType = null,
        ?string $referenceId = null,
        ?string $referenceCode = null,
        ?string $notes = null,
    ): StockMovement {
        if (in_array($type, [
            StockMovementType::Purchase,
            StockMovementType::Sale,
            StockMovementType::SaleReturn,
            StockMovementType::PurchaseReturn,
        ], true)) {
            throw new \DomainException(
                'Use dedicated service methods for purchase/sale movements.'
            );
        }

        if ($unitRatio < 1) {
            throw new \DomainException('Unit ratio must be >= 1.');
        }

        return DB::transaction(function () use (
            $productVariantId, $warehouseId, $type, $baseQuantity,
            $unitName, $unitRatio, $referenceType, $referenceId, $referenceCode, $notes
        ) {
            ProductVariant::lockForUpdate()->findOrFail($productVariantId);
            Warehouse::lockForUpdate()->findOrFail($warehouseId);

            return StockMovement::create([
                'product_variant_id' => $productVariantId,
                'warehouse_id'       => $warehouseId,
                'type'               => $type,
                'quantity'           => $type->isPositive() ? abs($baseQuantity) : -abs($baseQuantity),
                'unit_name_used'     => $unitName,
                'unit_ratio_used'    => $unitRatio,
                'reference_type'     => $referenceType,
                'reference_id'       => $referenceId,
                'reference_code'     => $referenceCode,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);
        });
    }

    public function directTransfer(
        int $productVariantId,
        int $fromWarehouseId,
        int $toWarehouseId,
        int $baseQuantity,
        string $unitName,
        int $unitRatio,
        string $referenceCode,
        string $notes,
    ): void {
        if ($unitRatio < 1) {
            throw new \DomainException('Unit ratio must be >= 1.');
        }

        DB::transaction(function () use (
            $productVariantId, $fromWarehouseId, $toWarehouseId,
            $baseQuantity, $unitName, $unitRatio, $referenceCode, $notes
        ) {
            // Lock warehouses in sorted-ID order to avoid deadlocks.
            $ids = collect([$fromWarehouseId, $toWarehouseId])->sort()->values()->all();
            Warehouse::whereIn('id', $ids)->orderBy('id')->lockForUpdate()->get();

            ProductVariant::lockForUpdate()->findOrFail($productVariantId);

            $out = StockMovement::create([
                'product_variant_id' => $productVariantId,
                'warehouse_id'       => $fromWarehouseId,
                'type'               => StockMovementType::TransferOut,
                'quantity'           => -abs($baseQuantity),
                'unit_name_used'     => $unitName,
                'unit_ratio_used'    => $unitRatio,
                'reference_code'     => $referenceCode,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);

            StockMovement::create([
                'product_variant_id' => $productVariantId,
                'warehouse_id'       => $toWarehouseId,
                'type'               => StockMovementType::TransferIn,
                'quantity'           => abs($baseQuantity),
                'unit_name_used'     => $unitName,
                'unit_ratio_used'    => $unitRatio,
                'related_movement_id' => $out->id,
                'reference_code'     => $referenceCode,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);
        });
    }

    /**
     * Dispatch a confirmed requisition — materializes in_transits.
     *
     * Re-locks items and variants under the parent transaction and verifies
     * on-hand availability, excluding this requisition's own reservation.
     */
    public function dispatchTransfer(TransferRequisition $requisition): void
    {
        DB::transaction(function () use ($requisition) {
            TransferRequisition::lockForUpdate()->findOrFail($requisition->id);

            $items = TransferRequisitionItem::where('transfer_requisition_id', $requisition->id)
                ->lockForUpdate()
                ->get();

            $variantIds = $items->pluck('product_variant_id')
                ->merge($items->pluck('substitute_product_variant_id'))
                ->filter()
                ->unique()
                ->values()
                ->all();

            if (! empty($variantIds)) {
                ProductVariant::whereIn('id', $variantIds)
                    ->orderBy('id')
                    ->lockForUpdate()
                    ->get();
            }

            Warehouse::lockForUpdate()->findOrFail($requisition->from_warehouse_id);

            $availableByVariant = ProductVariant::batchAvailableQuantity(
                $variantIds,
                $requisition->from_warehouse_id,
                null,
                $requisition->id, // exclude own transfer reservation
            );

            foreach ($items as $item) {
                if ($item->approved_base_qty === null) {
                    throw new \DomainException(
                        "Item {$item->id} has no approved base quantity."
                    );
                }

                $variantId = $item->actualVariantId();
                $available = $availableByVariant[$variantId] ?? 0;

                if ($item->approved_base_qty > $available) {
                    throw new \DomainException(
                        "Insufficient stock to dispatch variant {$variantId}. " .
                        "Required: {$item->approved_base_qty}, available: {$available}."
                    );
                }

                InTransit::create([
                    'transfer_requisition_id'      => $requisition->id,
                    'transfer_requisition_item_id' => $item->id,
                    'product_variant_id'           => $variantId,
                    'dispatched_base_qty'          => $item->approved_base_qty,
                    'dispatched_at'                => now(),
                    'status'                       => InTransitStatus::InTransit,
                ]);

                StockMovement::create([
                    'product_variant_id' => $variantId,
                    'warehouse_id'       => $requisition->from_warehouse_id,
                    'type'               => StockMovementType::TransferOut,
                    'quantity'           => -abs($item->approved_base_qty),
                    'unit_name_used'     => $item->approved_unit_name,
                    'unit_ratio_used'    => $item->approved_unit_ratio,
                    'reference_type'     => TransferRequisition::class,
                    'reference_id'       => (string) $requisition->id,
                    'reference_code'     => $requisition->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                $item->update(['shipped_base_qty' => $item->approved_base_qty]);

                $availableByVariant[$variantId] = $available - $item->approved_base_qty;
            }

            $requisition->update([
                'status'        => \App\Enums\TransferRequisitionStatus::Dispatched,
                'dispatched_at' => now(),
                'dispatched_by' => auth()->id(),
            ]);
        });
    }

    /**
     * Scan-to-receive with idempotency via state-equality check.
     *
     * @param  array<int, array{received_good:int, received_damaged:int}>  $scanPayload
     */
    public function scanToReceive(TransferRequisition $requisition, array $scanPayload): void
    {
        DB::transaction(function () use ($requisition, $scanPayload) {
            TransferRequisition::lockForUpdate()->findOrFail($requisition->id);

            $checksum = hash('sha256', json_encode($scanPayload));

            $alreadyProcessed = StockMovementIdempotencyKey::where('transfer_requisition_id', $requisition->id)
                ->where('payload_checksum', $checksum)
                ->exists();

            if ($alreadyProcessed) {
                return;
            }

            // First-scan detection: no idempotency record exists yet.
            $isFirstScan = ! StockMovementIdempotencyKey::where(
                'transfer_requisition_id',
                $requisition->id
            )->exists();

            $items = TransferRequisitionItem::where('transfer_requisition_id', $requisition->id)
                ->lockForUpdate()
                ->get();

            foreach ($items as $item) {
                $payload = $scanPayload[$item->id] ?? null;

                if ($payload === null && $isFirstScan) {
                    $this->writeOffOmittedItem($requisition, $item);
                    continue;
                }

                if ($payload === null) {
                    continue;
                }

                $good = (int) ($payload['received_good'] ?? 0);
                $damaged = (int) ($payload['received_damaged'] ?? 0);

                if ($good > 0) {
                    StockMovement::create([
                        'product_variant_id' => $item->actualVariantId(),
                        'warehouse_id'       => $requisition->to_warehouse_id,
                        'type'               => StockMovementType::TransferIn,
                        'quantity'           => abs($good),
                        'unit_name_used'     => $item->approved_unit_name,
                        'unit_ratio_used'    => $item->approved_unit_ratio,
                        'reference_type'     => TransferRequisition::class,
                        'reference_id'       => (string) $requisition->id,
                        'reference_code'     => $requisition->reference_code,
                        'created_by'         => auth()->id(),
                    ]);
                }

                $item->update([
                    'received_good_base_qty'    => $item->received_good_base_qty + $good,
                    'received_damaged_base_qty' => $item->received_damaged_base_qty + $damaged,
                    'received_qty'              => $item->received_qty + $good,
                ]);

                $item->refresh();

                if ($item->received_good_base_qty + $item->received_damaged_base_qty >= $item->approved_base_qty) {
                    $this->markInTransit($item, InTransitStatus::Cleared);
                }
            }

            StockMovementIdempotencyKey::create([
                'transfer_requisition_id' => $requisition->id,
                'payload_checksum'        => $checksum,
                'resulting_item_states'   => $requisition->items()->get()->toArray(),
            ]);

            $allReceived = $requisition->items()->get()->every(
                fn ($item) => $item->received_good_base_qty + $item->received_damaged_base_qty >= $item->approved_base_qty
            );

            $requisition->update([
                'status'       => $allReceived
                    ? \App\Enums\TransferRequisitionStatus::Completed
                    : \App\Enums\TransferRequisitionStatus::PartiallyReceived,
                'received_by'  => auth()->id(),
                'completed_at' => $allReceived ? now() : null,
            ]);
        });
    }

    private function writeOffOmittedItem(TransferRequisition $requisition, TransferRequisitionItem $item): void
    {
        $variant = $item->productVariant;
        $unitCost = LossLedger::snapshotUnitCostFrom($variant);
        $totalLoss = LossLedger::calculateTotalFinancialLoss($unitCost, $item->approved_base_qty);

        LossLedger::create([
            'transfer_requisition_id'      => $requisition->id,
            'transfer_requisition_item_id' => $item->id,
            'product_variant_id'           => $item->actualVariantId(),
            'warehouse_id'                 => $requisition->to_warehouse_id,
            'lost_base_qty'                => $item->approved_base_qty,
            'damaged_base_qty'             => 0,
            'unit_cost_price'              => $unitCost,
            'total_financial_loss'         => $totalLoss,
            'loss_category'                => 'shortfall',
            'notes'                        => bccomp($unitCost, '0.0000', 4) === 0
                ? 'Cost price missing or zero at time of write-off.'
                : null,
            'recorded_by'                  => auth()->id(),
            'recorded_at'                  => now(),
        ]);

        $this->markInTransit($item, InTransitStatus::Lost);
    }

    private function markInTransit(TransferRequisitionItem $item, InTransitStatus $status): void
    {
        InTransit::where('transfer_requisition_item_id', $item->id)
            ->where('status', InTransitStatus::InTransit->value)
            ->update([
                'status'     => $status->value,
                'cleared_at' => now(),
            ]);
    }

    public function adjustment(
        int $productVariantId,
        int $warehouseId,
        int $signedBaseQuantity,
        string $notes,
    ): StockMovement {
        return DB::transaction(function () use ($productVariantId, $warehouseId, $signedBaseQuantity, $notes) {
            $variant = ProductVariant::lockForUpdate()->findOrFail($productVariantId);
            Warehouse::lockForUpdate()->findOrFail($warehouseId);

            return StockMovement::create([
                'product_variant_id' => $productVariantId,
                'warehouse_id'       => $warehouseId,
                'type'               => StockMovementType::Adjustment,
                'quantity'           => $signedBaseQuantity,
                'unit_name_used'     => $variant->base_unit_name,
                'unit_ratio_used'    => 1,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);
        });
    }
}
```

### 6.3 NegotiationService

```php
namespace App\Services;

use App\Enums\RevisionStatus;
use App\Enums\TransferRequisitionStatus;
use App\Models\TransferRequisition;
use App\Models\TransferRequisitionItemRevision;

class NegotiationService
{
    public function submitRequest(TransferRequisition $requisition): void
    {
        if ($requisition->status !== TransferRequisitionStatus::Draft) {
            throw new \DomainException('Only draft requisitions can be submitted.');
        }

        $requisition->update([
            'status'       => TransferRequisitionStatus::Requested,
            'requested_at' => now(),
            'requested_by' => $requisition->requested_by ?? auth()->id(),
        ]);
    }

    /**
     * Materialize requested items as approved items on confirm.
     *
     * After materialization, verifies every item has a non-null approved
     * base quantity — a confirm with a null approved qty would silently
     * produce a wrong reservation.
     */
    public function materializeRequestedAsApproved(TransferRequisition $requisition): void
    {
        foreach ($requisition->items as $item) {
            if ($item->approved_base_qty !== null) {
                continue;
            }

            $item->update([
                'approved_unit_name'  => $item->requested_unit_name,
                'approved_unit_ratio' => $item->requested_unit_ratio,
                'approved_qty'        => $item->requested_qty,
                'approved_base_qty'   => $item->requested_base_qty,
            ]);
        }

        if ($requisition->items()->whereNull('approved_base_qty')->exists()) {
            throw new \DomainException(
                'All items must have an approved base quantity before confirmation.'
            );
        }
    }

    public function assertNegotiable(TransferRequisitionItemRevision $revision): void
    {
        $parent = $revision->item->transferRequisition;
        if (! in_array($parent->status, [
            TransferRequisitionStatus::Requested,
            TransferRequisitionStatus::UnderReviewFulfiller,
            TransferRequisitionStatus::UnderReviewRequestor,
        ], true)) {
            throw new \App\Exceptions\NegotiationNotAllowedException(
                'Parent requisition is not in a negotiable status.'
            );
        }
        if ($revision->status !== RevisionStatus::Pending) {
            throw new \App\Exceptions\NegotiationNotAllowedException(
                'Revision is already resolved.'
            );
        }
    }

    public function accept(TransferRequisitionItemRevision $revision): void
    {
        $this->assertNegotiable($revision);
        $revision->ensureCanTransitionTo(RevisionStatus::Accepted);
        $revision->update([
            'status'       => RevisionStatus::Accepted,
            'responded_at' => now(),
        ]);
    }

    public function reject(TransferRequisitionItemRevision $revision): void
    {
        $this->assertNegotiable($revision);
        $revision->ensureCanTransitionTo(RevisionStatus::Rejected);
        $revision->update([
            'status'       => RevisionStatus::Rejected,
            'responded_at' => now(),
        ]);
    }
}
```

### 6.4 PurchaseService

```php
namespace App\Services;

use App\Enums\PurchaseOrderStatus;
use App\Enums\StockMovementType;
use App\Models\ProductVariant;
use App\Models\ProductVariantPrice;
use App\Models\PurchaseOrder;
use App\Models\PurchaseOrderItem;
use App\Models\StockMovement;
use App\Models\Warehouse;
use Illuminate\Support\Facades\DB;

class PurchaseService
{
    public function __construct(
        private readonly GuardsOutstandingQuantity $guards,
    ) {}

    public function orderPurchase(PurchaseOrder $order): void
    {
        if ($order->status !== PurchaseOrderStatus::Draft) {
            throw new \DomainException('Only draft purchase orders can be ordered.');
        }

        $order->update([
            'status'     => PurchaseOrderStatus::Ordered,
            'ordered_at' => now(),
            'ordered_by' => $order->ordered_by ?? auth()->id(),
        ]);
    }

    /**
     * @param  array<int, int>  $receivedByItemId  item_id => base_qty_received
     */
    public function receivePurchase(int $orderId, array $receivedByItemId): void
    {
        DB::transaction(function () use ($orderId, $receivedByItemId) {
            $order = PurchaseOrder::lockForUpdate()->findOrFail($orderId);

            if (! in_array($order->status, [
                PurchaseOrderStatus::Ordered,
                PurchaseOrderStatus::PartiallyReceived,
            ], true)) {
                throw new \DomainException('Purchase order is not in a receivable state.');
            }

            $items = PurchaseOrderItem::where('purchase_order_id', $order->id)
                ->lockForUpdate()
                ->get();

            $variantIds = $items->pluck('product_variant_id')->unique()->values()->all();

            if (! empty($variantIds)) {
                ProductVariant::whereIn('id', $variantIds)
                    ->orderBy('id')
                    ->lockForUpdate()
                    ->get();
            }

            Warehouse::lockForUpdate()->findOrFail($order->warehouse_id);

            foreach ($items as $item) {
                $received = (int) ($receivedByItemId[$item->id] ?? 0);
                if ($received <= 0) {
                    continue;
                }

                $this->guards->assertPurchaseNotOverReceived($item, $received);

                StockMovement::create([
                    'product_variant_id' => $item->product_variant_id,
                    'warehouse_id'       => $order->warehouse_id,
                    'type'               => StockMovementType::Purchase,
                    'quantity'           => abs($received),
                    'unit_name_used'     => $item->ordered_unit_name,
                    'unit_ratio_used'    => $item->ordered_unit_ratio,
                    'reference_type'     => PurchaseOrder::class,
                    'reference_id'       => (string) $order->id,
                    'reference_code'     => $order->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                $item->update(['received_base_qty' => $item->received_base_qty + $received]);

                if ($order->update_cost_price) {
                    $this->updateCurrentCostPrice($item->product_variant_id, $item->unit_cost_price);
                }
            }

            $allReceived = $items->every(
                fn ($item) => $item->fresh()->received_base_qty >= $item->ordered_base_qty
            );

            $order->update([
                'status'      => $allReceived ? PurchaseOrderStatus::Received : PurchaseOrderStatus::PartiallyReceived,
                'received_at' => $allReceived ? now() : null,
                'received_by' => auth()->id(),
            ]);
        });
    }

    public function cancelPurchaseOrder(PurchaseOrder $order): void
    {
        DB::transaction(function () use ($order) {
            $fresh = PurchaseOrder::lockForUpdate()->findOrFail($order->id);

            if (! $fresh->canBeCancelled()) {
                throw new \DomainException('Purchase order cannot be cancelled.');
            }

            $fresh->update([
                'status'       => PurchaseOrderStatus::Cancelled,
                'cancelled_at' => now(),
            ]);
        });
    }

    private function updateCurrentCostPrice(int $variantId, string $newCost): void
    {
        $variant = ProductVariant::lockForUpdate()->findOrFail($variantId);
        $current = $variant->currentPrice;

        if ($current && bccomp($current->cost_price, $newCost, 4) === 0) {
            return;
        }

        if ($current) {
            $current->update(['is_current' => false]);
        }

        ProductVariantPrice::create([
            'product_variant_id' => $variantId,
            'cost_price'         => $newCost,
            'sale_price'         => $current?->sale_price ?? '0.0000',
            'effective_from'     => now(),
            'is_current'         => true,
            'set_by'             => auth()->id(),
        ]);
    }
}
```

### 6.5 SalesService

```php
namespace App\Services;

use App\Enums\SalesOrderStatus;
use App\Enums\StockMovementType;
use App\Models\ProductVariant;
use App\Models\SalesOrder;
use App\Models\SalesOrderItem;
use App\Models\StockMovement;
use App\Models\Warehouse;
use Illuminate\Support\Facades\DB;

class SalesService
{
    public function __construct(
        private readonly GuardsOutstandingQuantity $guards,
    ) {}

    public function confirmSalesOrder(SalesOrder $order): void
    {
        DB::transaction(function () use ($order) {
            $fresh = SalesOrder::lockForUpdate()->findOrFail($order->id);

            if ($fresh->status !== SalesOrderStatus::Draft) {
                throw new \DomainException('Only draft sales orders can be confirmed.');
            }

            $items = SalesOrderItem::where('sales_order_id', $fresh->id)
                ->lockForUpdate()
                ->get();

            foreach ($items as $item) {
                $variant = ProductVariant::with('currentPrice')->find($item->product_variant_id);
                $salePrice = (string) ($variant?->currentPrice?->sale_price ?? '0.0000');
                $item->update(['unit_sale_price_snapshot' => $salePrice]);
            }

            $fresh->update([
                'status'       => SalesOrderStatus::Confirmed,
                'confirmed_at' => now(),
            ]);
        });
    }

    /**
     * @param  array<int, int>  $dispatchByItemId  item_id => base_qty_dispatched
     */
    public function dispatchSale(int $orderId, array $dispatchByItemId): void
    {
        DB::transaction(function () use ($orderId, $dispatchByItemId) {
            $order = SalesOrder::lockForUpdate()->findOrFail($orderId);

            if (! in_array($order->status, [
                SalesOrderStatus::Confirmed,
                SalesOrderStatus::PartiallyDispatched,
            ], true)) {
                throw new \DomainException('Sales order is not in a dispatchable state.');
            }

            $items = SalesOrderItem::where('sales_order_id', $order->id)
                ->lockForUpdate()
                ->get();

            $variantIds = $items->pluck('product_variant_id')->unique()->values()->all();

            if (! empty($variantIds)) {
                ProductVariant::whereIn('id', $variantIds)
                    ->orderBy('id')
                    ->lockForUpdate()
                    ->get();
            }

            Warehouse::lockForUpdate()->findOrFail($order->warehouse_id);

            // Exclude this order's own reservation so its outstanding qty
            // does not count against its own availability.
            $availableByVariant = ProductVariant::batchAvailableQuantity(
                $variantIds,
                $order->warehouse_id,
                $order->id,
            );

            foreach ($items as $item) {
                $dispatched = (int) ($dispatchByItemId[$item->id] ?? 0);
                if ($dispatched <= 0) {
                    continue;
                }

                $this->guards->assertSaleNotOverDispatched($item, $dispatched);

                $available = $availableByVariant[$item->product_variant_id] ?? 0;
                if ($dispatched > $available) {
                    throw new \DomainException(
                        "Insufficient stock for variant {$item->product_variant_id}."
                    );
                }

                StockMovement::create([
                    'product_variant_id' => $item->product_variant_id,
                    'warehouse_id'       => $order->warehouse_id,
                    'type'               => StockMovementType::Sale,
                    'quantity'           => -abs($dispatched),
                    'unit_name_used'     => $item->unit_name,
                    'unit_ratio_used'    => $item->unit_ratio,
                    'reference_type'     => SalesOrder::class,
                    'reference_id'       => (string) $order->id,
                    'reference_code'     => $order->reference_code,
                    'created_by'         => auth()->id(),
                ]);

                $item->update(['dispatched_base_qty' => $item->dispatched_base_qty + $dispatched]);
                $availableByVariant[$item->product_variant_id] = $available - $dispatched;
            }

            $allDispatched = $items->every(
                fn ($item) => $item->fresh()->dispatched_base_qty >= $item->base_qty
            );

            $order->update([
                'status'        => $allDispatched ? SalesOrderStatus::Dispatched : SalesOrderStatus::PartiallyDispatched,
                'dispatched_at' => $allDispatched ? now() : null,
                'dispatched_by' => auth()->id(),
            ]);
        });
    }

    /**
     * Record a sales return with server-side cumulative over-return guard.
     * Locks variant and warehouse to preserve uniform locking discipline.
     */
    public function recordSalesReturn(int $itemId, int $returnedBaseQty, ?string $notes = null): void
    {
        DB::transaction(function () use ($itemId, $returnedBaseQty, $notes) {
            $item = SalesOrderItem::lockForUpdate()->findOrFail($itemId);

            $order = $item->salesOrder;

            ProductVariant::lockForUpdate()->findOrFail($item->product_variant_id);
            Warehouse::lockForUpdate()->findOrFail($order->warehouse_id);

            $alreadyReturned = (int) StockMovement::where('type', StockMovementType::SaleReturn->value)
                ->where('reference_type', SalesOrderItem::class)
                ->where('reference_id', (string) $item->id)
                ->sum('quantity');

            if ($alreadyReturned + $returnedBaseQty > $item->dispatched_base_qty) {
                throw new \DomainException(
                    "Return of {$returnedBaseQty} would exceed dispatched quantity. Already returned: {$alreadyReturned}."
                );
            }

            StockMovement::create([
                'product_variant_id' => $item->product_variant_id,
                'warehouse_id'       => $order->warehouse_id,
                'type'               => StockMovementType::SaleReturn,
                'quantity'           => abs($returnedBaseQty),
                'unit_name_used'     => $item->unit_name,
                'unit_ratio_used'    => $item->unit_ratio,
                'reference_type'     => SalesOrderItem::class,
                'reference_id'       => (string) $item->id,
                'reference_code'     => $order->reference_code,
                'notes'              => $notes,
                'created_by'         => auth()->id(),
            ]);
        });
    }

    public function cancelSalesOrder(SalesOrder $order): void
    {
        DB::transaction(function () use ($order) {
            $fresh = SalesOrder::lockForUpdate()->findOrFail($order->id);

            if (! in_array($fresh->status, [SalesOrderStatus::Draft, SalesOrderStatus::Confirmed], true)) {
                throw new \DomainException('Sales order cannot be cancelled.');
            }

            $fresh->update([
                'status'       => SalesOrderStatus::Cancelled,
                'cancelled_at' => now(),
            ]);
        });
    }
}
```

---

## 🎨 Section 7: Filament Resources — Master Specifications

### 7A. ProductResource

**Model:** `App\Models\ProductVariant` · **Group:** CATALOG · **Sort:** 1 · **Route:** `/admin/products`

#### 7A.1 ProductForm.php

```php
namespace App\Filament\Resources\Products\Schemas;

use Filament\Forms\Components\KeyValue;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Tabs;
use Filament\Schemas\Components\Tabs\Tab;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class ProductForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Tabs::make('Product Variant')
                ->persistTabInQueryString()
                ->columnSpanFull()
                ->tabs([
                    Tab::make('Identity')
                        ->icon(Heroicon::Identification)
                        ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->schema([
                            Select::make('product_id')
                                ->relationship('product', 'name')
                                ->prefixIcon(Heroicon::FolderOpen)
                                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                                ->required()
                                ->createOptionForm(fn (Schema $schema) => $schema->components([
                                    TextInput::make('name')
                                        ->prefixIcon(Heroicon::Identification)
                                        ->columnSpanFull()
                                        ->required()
                                        ->maxLength(255),
                                    TextInput::make('category')
                                        ->prefixIcon(Heroicon::Tag)
                                        ->columnSpanFull(),
                                ])),

                            TextInput::make('sku')
                                ->prefixIcon(Heroicon::Tag)
                                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                                ->required()
                                ->unique(ignoreRecord: true),

                            TextInput::make('barcode')
                                ->prefixIcon(Heroicon::QrCode)
                                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                                ->nullable()
                                ->unique(ignoreRecord: true),

                            TextInput::make('name')
                                ->prefixIcon(Heroicon::Identification)
                                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                                ->required(),
                        ]),

                    Tab::make('Stock & Pricing')
                        ->icon(Heroicon::CurrencyDollar)
                        ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->schema([
                            TextInput::make('base_unit_name')
                                ->prefixIcon(Heroicon::Scale)
                                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                                ->required(),

                            TextInput::make('reorder_point')
                                ->prefixIcon(Heroicon::ExclamationTriangle)
                                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                                ->numeric()
                                ->default(0)
                                ->required(),

                            KeyValue::make('attributes')
                                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2]),
                        ]),

                    Tab::make('Status')
                        ->icon(Heroicon::CheckCircle)
                        ->schema([
                            Toggle::make('is_active')
                                ->onIcon(Heroicon::CheckCircle)
                                ->offIcon(Heroicon::XCircle)
                                ->columnSpanFull()
                                ->default(true),
                        ]),
                ]),
        ]);
    }
}
```

#### 7A.2 ProductsTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\Products\Tables;

use App\Filament\Resources\Products\Actions\EditProductFamilyAction;
use App\Filament\Resources\Products\Actions\ManageUnitConversionsAction;
use App\Filament\Resources\Products\Actions\QuickStockAdjustmentAction;
use App\Filament\Resources\Products\Actions\SetCurrentPriceAction;
use App\Filament\Resources\Products\ProductResource;
use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Actions\RestoreAction;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\IconColumn;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
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
                Stack::make([
                    Split::make([
                        TextColumn::make('sku')
                            ->fontFamily('mono')
                            ->weight(FontWeight::Bold)
                            ->searchable()
                            ->sortable()
                            ->copyable()
                            ->copyMessage('SKU copied'),

                        TextColumn::make('currentPrice.sale_price')
                            ->money(config('app.currency'))
                            ->weight(FontWeight::Bold)
                            ->alignEnd()
                            ->sortable(),
                    ])->from('md'),

                    TextColumn::make('name')
                        ->label('Variant')
                        ->searchable()
                        ->sortable()
                        ->limit(50)
                        ->weight(FontWeight::SemiBold),

                    Split::make([
                        TextColumn::make('product.name')
                            ->label('Family')
                            ->badge()
                            ->color('gray')
                            ->searchable(),

                        TextColumn::make('base_unit_name')
                            ->label('Unit')
                            ->badge()
                            ->color('info'),

                        TextColumn::make('reorder_point')
                            ->label('Reorder')
                            ->badge()
                            ->color(fn ($state) => $state > 0 ? 'warning' : 'gray')
                            ->numeric(),
                    ])->from('md'),

                    IconColumn::make('is_active')
                        ->label('Active')
                        ->boolean()
                        ->trueIcon(Heroicon::CheckCircle)
                        ->falseIcon(Heroicon::XCircle)
                        ->trueColor('success')
                        ->falseColor('danger'),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                TernaryFilter::make('is_active')
                    ->label('Active status')
                    ->placeholder('All variants')
                    ->trueLabel('Active only')
                    ->falseLabel('Inactive only'),

                SelectFilter::make('product_id')
                    ->label('Product family')
                    ->relationship('product', 'name')
                    ->searchable(),

                TrashedFilter::make(),
            ])
            ->defaultSort('sku')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordUrl(fn ($record) => ProductResource::getUrl('view', ['record' => $record]))
            ->recordActions([
                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
                    ->modalWidth(Width::Large),

                SetCurrentPriceAction::make()
                    ->icon(Heroicon::CurrencyDollar)
                    ->color('primary'),

                EditProductFamilyAction::make()
                    ->icon(Heroicon::FolderOpen),

                ManageUnitConversionsAction::make()
                    ->icon(Heroicon::Scale),

                QuickStockAdjustmentAction::make()
                    ->icon(Heroicon::AdjustmentsHorizontal)
                    ->color('warning'),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete'),

                RestoreAction::make()
                    ->icon(Heroicon::ArrowUturnLeft)
                    ->authorize('restore'),
            ]);

        // Bulk actions intentionally omitted (F30). Card layout does not render
        // per-card checkboxes without the mkdev-grid-card-layout plugin.
    }
}
```

#### 7A.3 ProductInfolist.php

```php
namespace App\Filament\Resources\Products\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class ProductInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('Identity')
                    ->icon(Heroicon::Identification)
                    ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('product.name')
                            ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2]),
                        TextEntry::make('sku')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('barcode')
                            ->placeholder('—')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                    ]),

                Section::make('Pricing')
                    ->icon(Heroicon::CurrencyDollar)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->columns(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('currentPrice.cost_price')
                            ->money(config('app.currency')),
                        TextEntry::make('currentPrice.sale_price')
                            ->money(config('app.currency')),
                    ]),

                Section::make('Unit Conversions')
                    ->icon(Heroicon::Scale)
                    ->columnSpanFull()
                    ->schema([
                        RepeatableEntry::make('unitConversions')
                            ->schema([
                                Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                                    TextEntry::make('unit_name')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('base_unit_ratio')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('is_default_purchase')
                                        ->badge()->boolean()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                ]),
                            ]),
                    ]),
            ]),
        ]);
    }
}
```

#### 7A.4 ManageUnitConversionsAction

```php
namespace App\Filament\Resources\Products\Actions;

use Filament\Actions\Action;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;

class ManageUnitConversionsAction
{
    public static function make(): Action
    {
        return Action::make('manageUnitConversions')
            ->label('UNIT CONVERSIONS')
            ->icon(Heroicon::Scale)
            ->modalWidth(Width::SevenExtraLarge)
            ->fillForm(fn ($record) => [
                'unitConversions' => $record->unitConversions
                    ->map->only(['unit_name', 'base_unit_ratio', 'is_default_purchase', 'is_default_transfer'])
                    ->toArray(),
            ])
            ->schema([
                Repeater::make('unitConversions')
                    ->schema([
                        TextInput::make('unit_name')
                            ->required()
                            ->disabled(fn (Get $get, $record) => $get('unit_name') === $record->base_unit_name),

                        TextInput::make('base_unit_ratio')
                            ->numeric()
                            ->required()
                            ->disabled(fn (Get $get, $record) => $get('unit_name') === $record->base_unit_name),

                        Toggle::make('is_default_purchase'),
                        Toggle::make('is_default_transfer'),
                    ])
                    ->columns(4)
                    ->deletable(fn ($record, array $item) => ($item['unit_name'] ?? null) !== $record->base_unit_name),
            ])
            ->action(function (array $data, $record) {
                $baseName = $record->base_unit_name;
                $incoming = collect($data['unitConversions'] ?? []);

                $record->unitConversions()
                    ->where('unit_name', '!=', $baseName)
                    ->delete();

                foreach ($incoming as $conv) {
                    if (($conv['unit_name'] ?? null) === $baseName) {
                        continue;
                    }

                    $record->unitConversions()->create($conv);
                }
            });
    }
}
```

---

### 7B. TransferRequisitionResource

**Model:** `App\Models\TransferRequisition` · **Group:** OPERATIONS · **Sort:** 1 · **Route:** `/admin/transfer-requisitions`

#### 7B.1 TransferRequisitionForm.php

```php
namespace App\Filament\Resources\TransferRequisitions\Schemas;

use App\Models\ProductVariantUnitConversion;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class TransferRequisitionForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            ...self::getRoutingFields(),
            ...self::getMaterialManifestFields(),
        ]);
    }

    public static function getRoutingFields(): array
    {
        return [
            Section::make('Warehouse Routing')
                ->icon(Heroicon::BuildingOffice)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('from_warehouse_id')
                        ->prefixIcon(Heroicon::BuildingOffice)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->default(fn () => auth()->user()->warehouses()->count() === 1
                            ? auth()->user()->warehouses()->first()->id
                            : null)
                        ->required(),

                    Select::make('to_warehouse_id')
                        ->prefixIcon(Heroicon::BuildingOffice2)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->required()
                        ->different('from_warehouse_id'),
                ]),
        ];
    }

    public static function getMaterialManifestFields(): array
    {
        return [
            Repeater::make('items')
                ->relationship()
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 4])
                ->schema([
                    Select::make('product_variant_id')
                        ->label('Variant (SKU)')
                        ->relationship('productVariant', 'sku')
                        ->prefixIcon(Heroicon::Tag)
                        ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->searchable()
                        ->preload()
                        ->required()
                        ->disableOptionsWhenSelectedInSiblingRepeaterItems()
                        ->live()
                        ->afterStateUpdated(function ($set) {
                            $set('requested_unit_name', null);
                            $set('requested_unit_ratio', null);
                        }),

                    Select::make('requested_unit_name')
                        ->label('Unit')
                        ->prefixIcon(Heroicon::Scale)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(function (Get $get) {
                            $variantId = $get('product_variant_id');
                            if (! $variantId) {
                                return [];
                            }
                            return ProductVariantUnitConversion::where('product_variant_id', $variantId)
                                ->orderByDesc('base_unit_ratio')
                                ->pluck('unit_name', 'unit_name')
                                ->toArray();
                        })
                        ->required()
                        ->live()
                        ->afterStateUpdated(function (Get $get, $set, $state) {
                            $ratio = ProductVariantUnitConversion::where('product_variant_id', $get('product_variant_id'))
                                ->where('unit_name', $state)
                                ->value('base_unit_ratio');
                            $set('requested_unit_ratio', $ratio ?? 1);
                        }),

                    TextInput::make('requested_unit_ratio')
                        ->label('Ratio (base)')
                        ->hintIcon(Heroicon::InformationCircle)
                        ->hint('Auto-filled from selected unit')
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->disabled()
                        ->dehydrated()
                        ->required(),

                    TextInput::make('requested_qty')
                        ->label('Qty')
                        ->prefixIcon(Heroicon::Hashtag)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->minValue(1)
                        ->required(),
                ])
                ->minItems(1)
                ->required()
                ->dehydrated()
                ->mutateRelationshipDataBeforeCreateUsing(function (array $data): array {
                    $data['requested_base_qty'] = (int) $data['requested_qty'] * (int) $data['requested_unit_ratio'];
                    return $data;
                })
                ->mutateRelationshipDataBeforeSaveUsing(function (array $data): array {
                    $data['requested_base_qty'] = (int) $data['requested_qty'] * (int) $data['requested_unit_ratio'];
                    return $data;
                }),
        ];
    }
}
```

#### 7B.2 CreateTransferRequisition.php

```php
namespace App\Filament\Resources\TransferRequisitions\Pages;

use App\Filament\Resources\TransferRequisitions\TransferRequisitionResource;
use App\Filament\Resources\TransferRequisitions\Schemas\TransferRequisitionForm;
use Filament\Forms\Components\Placeholder;
use Filament\Resources\Pages\CreateRecord;
use Filament\Resources\Pages\CreateRecord\Concerns\HasWizard;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;

class CreateTransferRequisition extends CreateRecord
{
    use HasWizard;

    protected static string $resource = TransferRequisitionResource::class;

    /**
     * @return array<Step>
     */
    protected function getSteps(): array
    {
        return [
            Step::make('Routing Pathways')
                ->description('Define origin and destination warehouses')
                ->icon(Heroicon::BuildingOffice)
                ->schema(TransferRequisitionForm::getRoutingFields()),

            Step::make('Material Manifest')
                ->description('Add items — pick variant first, then unit')
                ->icon(Heroicon::ClipboardDocumentList)
                ->schema(TransferRequisitionForm::getMaterialManifestFields()),

            Step::make('Review & Verify')
                ->description('Confirm details before submission')
                ->icon(Heroicon::CheckCircle)
                ->schema([
                    Placeholder::make('review_summary')
                        ->columnSpanFull()
                        ->content(fn (Get $get) => view(
                            'filament.wizards.transfer-review',
                            ['state' => $get()],
                        )),
                ]),
        ];
    }

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $data['reference_code'] = $data['reference_code']
            ?? 'TR-' . now()->format('YmdHis') . '-' . random_int(100, 999);
        $data['requested_by'] = auth()->id();
        $data['status'] = \App\Enums\TransferRequisitionStatus::Draft->value;
        return $data;
    }

    public function getMaxContentWidth(): ?string
    {
        return Width::SevenExtraLarge->value;
    }
}
```

#### 7B.3 TransferRequisitionsTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\TransferRequisitions\Tables;

use App\Enums\TransferRequisitionStatus;
use App\Models\TransferRequisition;
use Filament\Actions\Action;
use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Actions\ForceDeleteAction;
use Filament\Actions\RestoreAction;
use Filament\Actions\ViewAction;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;

class TransferRequisitionsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Stack::make([
                    Split::make([
                        TextColumn::make('reference_code')
                            ->label('Reference')
                            ->fontFamily('mono')
                            ->weight(FontWeight::Bold)
                            ->searchable()
                            ->sortable()
                            ->copyable()
                            ->copyMessage('Reference copied'),

                        TextColumn::make('status')
                            ->badge()
                            ->alignEnd()
                            ->sortable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('fromWarehouse.name')
                            ->label('From')
                            ->icon(Heroicon::BuildingOffice)
                            ->iconColor('gray')
                            ->searchable(),

                        TextColumn::make('toWarehouse.name')
                            ->label('To')
                            ->icon(Heroicon::BuildingOffice2)
                            ->iconColor('gray')
                            ->searchable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('items_count')
                            ->label('Items')
                            ->counts('items')
                            ->badge()
                            ->color('gray')
                            ->numeric(),

                        TextColumn::make('requestedBy.name')
                            ->label('Requested by')
                            ->icon(Heroicon::User)
                            ->iconColor('gray')
                            ->placeholder('—'),

                        TextColumn::make('requested_at')
                            ->label('Requested')
                            ->dateTime('M j, Y')
                            ->sortable()
                            ->placeholder('—')
                            ->alignEnd(),
                    ])->from('lg'),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                SelectFilter::make('status')->options(TransferRequisitionStatus::class),
                SelectFilter::make('from_warehouse_id')
                    ->label('From warehouse')
                    ->relationship('fromWarehouse', 'name')
                    ->searchable(),
                SelectFilter::make('to_warehouse_id')
                    ->label('To warehouse')
                    ->relationship('toWarehouse', 'name')
                    ->searchable(),
                TrashedFilter::make(),
                \App\Filament\Support\Filters\AdminReviewFilters::period('requested_at')
                    ->authorize('viewAuditFilters'),
            ])
            ->defaultSort('created_at', 'desc')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordUrl(fn (TransferRequisition $record) => $record->getUrl('view'))
            ->recordActions([
                ViewAction::make(),

                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
                    ->visible(fn (TransferRequisition $record) => $record->status === TransferRequisitionStatus::Draft)
                    ->modalWidth(Width::Large),

                Action::make('submitRequest')
                    ->label('SUBMIT')
                    ->icon(Heroicon::PaperAirplane)
                    ->color('primary')
                    ->authorize('submitRequest')
                    ->visible(fn (TransferRequisition $record) => $record->status === TransferRequisitionStatus::Draft)
                    ->requiresConfirmation()
                    ->action(fn (TransferRequisition $record) => app(\App\Services\NegotiationService::class)->submitRequest($record)),

                Action::make('reviewNegotiate')
                    ->label('REVIEW')
                    ->icon(Heroicon::ChatBubbleLeftRight)
                    ->color('warning')
                    ->visible(fn (TransferRequisition $record) => in_array($record->status, [
                        TransferRequisitionStatus::Requested,
                        TransferRequisitionStatus::UnderReviewFulfiller,
                        TransferRequisitionStatus::UnderReviewRequestor,
                    ], true))
                    ->url(fn (TransferRequisition $record) => $record->getUrl('edit')),

                Action::make('confirm')
                    ->label('CONFIRM')
                    ->icon(Heroicon::CheckBadge)
                    ->color('primary')
                    ->authorize('confirm')
                    ->visible(fn (TransferRequisition $record) => in_array($record->status, [
                        TransferRequisitionStatus::Requested,
                        TransferRequisitionStatus::UnderReviewFulfiller,
                        TransferRequisitionStatus::UnderReviewRequestor,
                    ], true))
                    ->action(function (TransferRequisition $record) {
                        app(\App\Services\NegotiationService::class)->materializeRequestedAsApproved($record);
                        $record->update([
                            'status'      => TransferRequisitionStatus::Confirmed,
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
                    ->visible(fn (TransferRequisition $record) => $record->status === TransferRequisitionStatus::Confirmed)
                    ->action(fn (TransferRequisition $record) => app(\App\Services\InventoryService::class)->dispatchTransfer($record))
                    ->requiresConfirmation(),

                Action::make('scanToReceive')
                    ->label('RECEIVE')
                    ->icon(Heroicon::QrCode)
                    ->color('success')
                    ->authorize('receive')
                    ->visible(fn (TransferRequisition $record) => in_array($record->status, [
                        TransferRequisitionStatus::Dispatched,
                        TransferRequisitionStatus::PartiallyReceived,
                    ], true))
                    ->url(fn (TransferRequisition $record) => route('stn.scan', ['transferRequisition' => $record->id])),

                Action::make('cancel')
                    ->label('CANCEL')
                    ->icon(Heroicon::XMark)
                    ->color('danger')
                    ->authorize('cancel')
                    ->visible(fn (TransferRequisition $record) => $record->canBeCancelled())
                    ->requiresConfirmation(),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete')
                    ->visible(fn (TransferRequisition $record) => in_array($record->status, [
                        TransferRequisitionStatus::Draft,
                        TransferRequisitionStatus::Cancelled,
                    ], true)),

                RestoreAction::make()->icon(Heroicon::ArrowUturnLeft)->authorize('restore'),

                ForceDeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('forceDelete')
                    ->visible(fn () => auth()->user()->isAdmin()),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

#### 7B.4 TransferRequisitionInfolist.php

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
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('REQUISITION PROFILE')
                    ->icon(Heroicon::DocumentText)
                    ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('reference_code')
                            ->label('REFERENCE CODE')
                            ->weight(FontWeight::Bold)
                            ->size('lg')
                            ->copyable()
                            ->color('primary')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                        TextEntry::make('status')
                            ->label('OPERATIONAL STATUS')
                            ->badge()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                        TextEntry::make('fromWarehouse.name')
                            ->label('ORIGIN BRANCH')
                            ->icon(Heroicon::BuildingOffice)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                        TextEntry::make('toWarehouse.name')
                            ->label('RECEIVING BRANCH')
                            ->icon(Heroicon::BuildingOffice2)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                    ]),

                Section::make('AUTHORIZATION SIGN-OFFS')
                    ->icon(Heroicon::ShieldCheck)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('requestedBy.name')->label('REQUESTED BY')->icon(Heroicon::User)->placeholder('System Initialized'),
                        TextEntry::make('approvedBy.name')->label('APPROVED BY')->icon(Heroicon::Check)->placeholder('Pending Approval'),
                        TextEntry::make('dispatchedBy.name')->label('DISPATCHED BY')->icon(Heroicon::Truck)->placeholder('Pending Dispatch'),
                        TextEntry::make('receivedBy.name')->label('RECEIVED BY')->icon(Heroicon::QrCode)->placeholder('Pending Intake'),
                    ]),

                Section::make('MATERIAL MANIFEST ITEMS')
                    ->icon(Heroicon::ClipboardDocumentList)
                    ->columnSpanFull()
                    ->schema([
                        RepeatableEntry::make('items')
                            ->schema([
                                Grid::make(['default' => 1, 'md' => 3, 'xl' => 6])->schema([
                                    TextEntry::make('productVariant.sku')
                                        ->label('ORIGINAL SKU')
                                        ->weight(FontWeight::Bold)
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                                    TextEntry::make('substituteProductVariant.sku')
                                        ->label('PROPOSED SUBSTITUTE')
                                        ->badge()
                                        ->color('warning')
                                        ->placeholder('No Substitute')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                                    TextEntry::make('requested_qty')
                                        ->label('REQUESTED')
                                        ->state(fn ($record) => "{$record->requested_qty} {$record->requested_unit_name}")
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                                    TextEntry::make('approved_qty')
                                        ->label('APPROVED')
                                        ->state(fn ($record) => $record->approved_qty
                                            ? "{$record->approved_qty} {$record->approved_unit_name}"
                                            : 'Pending Verification')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                                    TextEntry::make('shipped_base_qty')
                                        ->label('SHIPPED (BASE)')
                                        ->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),

                                    TextEntry::make('received_good_base_qty')
                                        ->label('RECEIVED GOOD (BASE)')
                                        ->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                ]),
                            ]),
                    ]),
            ]),
        ]);
    }
}
```

---

### 7C. DirectTransferResource

**Model:** `App\Models\StockMovement` · **Group:** OPERATIONS · **Sort:** 2

#### 7C.1 DirectTransferForm.php

```php
namespace App\Filament\Resources\DirectTransfers\Schemas;

use App\Models\ProductVariantUnitConversion;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class DirectTransferForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            ...self::getLocationMappingFields(),
            ...self::getStockAllocationFields(),
        ]);
    }

    public static function getLocationMappingFields(): array
    {
        return [
            Section::make('Warehouse Routing')
                ->icon(Heroicon::BuildingOffice)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('from_warehouse_id')
                        ->prefixIcon(Heroicon::BuildingOffice)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->default(fn () => auth()->user()->warehouses()->count() === 1
                            ? auth()->user()->warehouses()->first()->id
                            : null)
                        ->required(),

                    Select::make('to_warehouse_id')
                        ->prefixIcon(Heroicon::BuildingOffice2)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->required()
                        ->different('from_warehouse_id'),
                ]),
        ];
    }

    public static function getStockAllocationFields(): array
    {
        return [
            Section::make('Stock Allocation')
                ->icon(Heroicon::Cube)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('product_variant_id')
                        ->label('Variant (SKU)')
                        ->relationship('productVariant', 'sku')
                        ->prefixIcon(Heroicon::Tag)
                        ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->searchable()
                        ->preload()
                        ->required()
                        ->live()
                        ->afterStateUpdated(function ($set) {
                            $set('unit_name', null);
                            $set('unit_ratio', null);
                        }),

                    Select::make('unit_name')
                        ->label('Unit')
                        ->prefixIcon(Heroicon::Scale)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(function (Get $get) {
                            $variantId = $get('product_variant_id');
                            if (! $variantId) {
                                return [];
                            }
                            return ProductVariantUnitConversion::where('product_variant_id', $variantId)
                                ->orderByDesc('base_unit_ratio')
                                ->pluck('unit_name', 'unit_name')
                                ->toArray();
                        })
                        ->required()
                        ->live()
                        ->afterStateUpdated(function (Get $get, $set, $state) {
                            $ratio = ProductVariantUnitConversion::where('product_variant_id', $get('product_variant_id'))
                                ->where('unit_name', $state)
                                ->value('base_unit_ratio');
                            $set('unit_ratio', $ratio ?? 1);
                        }),

                    TextInput::make('unit_ratio')
                        ->label('Ratio (base)')
                        ->hintIcon(Heroicon::InformationCircle)
                        ->hint('Auto-filled from selected unit')
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->disabled()
                        ->dehydrated()
                        ->required(),

                    TextInput::make('quantity')
                        ->label('Qty')
                        ->prefixIcon(Heroicon::Hashtag)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->minValue(1)
                        ->required(),
                ]),

            Textarea::make('notes')
                ->prefixIcon(Heroicon::ChatBubbleBottomCenterText)
                ->columnSpanFull()
                ->required()
                ->minLength(15),
        ];
    }
}
```

#### 7C.2 CreateDirectTransfer.php

```php
namespace App\Filament\Resources\DirectTransfers\Pages;

use App\Filament\Resources\DirectTransfers\DirectTransferResource;
use App\Filament\Resources\DirectTransfers\Schemas\DirectTransferForm;
use Filament\Forms\Components\Placeholder;
use Filament\Resources\Pages\CreateRecord;
use Filament\Resources\Pages\CreateRecord\Concerns\HasWizard;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;

class CreateDirectTransfer extends CreateRecord
{
    use HasWizard;

    protected static string $resource = DirectTransferResource::class;

    /**
     * @return array<Step>
     */
    protected function getSteps(): array
    {
        return [
            Step::make('Location Mapping')
                ->description('Select origin and destination warehouses')
                ->icon(Heroicon::BuildingOffice)
                ->schema(DirectTransferForm::getLocationMappingFields()),

            Step::make('Stock Allocation')
                ->description('Pick variant and unit, then quantity')
                ->icon(Heroicon::Cube)
                ->schema(DirectTransferForm::getStockAllocationFields()),

            Step::make('Review & Verify')
                ->description('Confirm transfer details')
                ->icon(Heroicon::CheckCircle)
                ->schema([
                    Placeholder::make('review_summary')
                        ->columnSpanFull()
                        ->content(fn (Get $get) => view(
                            'filament.wizards.direct-transfer-review',
                            ['state' => $get()],
                        )),
                ]),
        ];
    }

    protected function handleRecordCreation(array $data): \Illuminate\Database\Eloquent\Model
    {
        $referenceCode = 'DT-' . now()->format('YmdHis') . '-' . random_int(100, 999);

        app(\App\Services\InventoryService::class)->directTransfer(
            productVariantId: (int) $data['product_variant_id'],
            fromWarehouseId:  (int) $data['from_warehouse_id'],
            toWarehouseId:    (int) $data['to_warehouse_id'],
            baseQuantity:     (int) $data['quantity'] * (int) $data['unit_ratio'],
            unitName:         $data['unit_name'],
            unitRatio:        (int) $data['unit_ratio'],
            referenceCode:    $referenceCode,
            notes:            $data['notes'],
        );

        return \App\Models\StockMovement::query()
            ->where('reference_code', $referenceCode)
            ->where('type', \App\Enums\StockMovementType::TransferOut)
            ->firstOrFail();
    }

    public function getMaxContentWidth(): ?string
    {
        return Width::SevenExtraLarge->value;
    }
}
```

#### 7C.3 DirectTransfersTable.php

```php
namespace App\Filament\Resources\DirectTransfers\Tables;

use App\Enums\StockMovementType;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;
use Illuminate\Database\Eloquent\Builder;

class DirectTransfersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->query(
                \App\Models\StockMovement::query()
                    ->whereIn('type', [
                        StockMovementType::TransferOut->value,
                        StockMovementType::TransferIn->value,
                    ])
            )
            ->columns([
                TextColumn::make('reference_code')
                    ->label('REFERENCE')
                    ->fontFamily('mono')
                    ->weight('bold')
                    ->searchable()
                    ->copyable(),

                TextColumn::make('warehouse.name')
                    ->label('WAREHOUSE')
                    ->sortable(),

                TextColumn::make('productVariant.sku')
                    ->label('SKU')
                    ->fontFamily('mono')
                    ->searchable(),

                TextColumn::make('type')
                    ->badge()
                    ->sortable(),

                TextColumn::make('quantity')
                    ->numeric()
                    ->alignEnd()
                    ->color(fn ($state) => $state >= 0 ? 'success' : 'danger'),

                TextColumn::make('unit_name_used')
                    ->label('UNIT')
                    ->visibleFrom('md'),

                TextColumn::make('created_at')
                    ->label('TIMESTAMP')
                    ->dateTime('M j, Y H:i')
                    ->sortable(),

                TextColumn::make('createdBy.name')
                    ->label('BY')
                    ->visibleFrom('xl'),
            ])
            ->filters([
                SelectFilter::make('warehouse_id')
                    ->relationship('warehouse', 'name')
                    ->label('Warehouse')
                    ->searchable(),
            ])
            ->defaultSort('created_at', 'desc')
            ->stackedOnMobile()
            ->paginated([25, 50, 100])
            ->defaultPaginationPageOption(50);
    }
}
```

---

### 7D. InTransitResource

**Model:** `App\Models\InTransit` · **Group:** OPERATIONS · **Sort:** 3

#### 7D.1 InTransitInfolist.php

```php
namespace App\Filament\Resources\InTransits\Schemas;

use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class InTransitInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('IN-TRANSIT CARGO')
                    ->icon(Heroicon::Truck)
                    ->columnSpanFull()
                    ->columns(['default' => 1, 'md' => 3, 'xl' => 3])
                    ->schema([
                        TextEntry::make('transferRequisition.reference_code')
                            ->label('REQUISITION')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('productVariant.sku')
                            ->label('SKU')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('dispatched_base_qty')
                            ->label('DISPATCHED (BASE)')
                            ->numeric()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('dispatched_at')
                            ->label('DISPATCHED AT')
                            ->dateTime('M j, Y H:i')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('status')
                            ->badge()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('cleared_at')
                            ->label('CLEARED AT')
                            ->dateTime('M j, Y H:i')
                            ->placeholder('—')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                    ]),
            ]),
        ]);
    }
}
```

#### 7D.2 InTransitsTable.php — **Standard Table + `stackedOnMobile()`**

```php
namespace App\Filament\Resources\InTransits\Tables;

use App\Enums\InTransitStatus;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;

class InTransitsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('transferRequisition.reference_code')
                    ->label('REQUISITION')
                    ->fontFamily('mono')
                    ->weight('bold')
                    ->searchable()
                    ->sortable()
                    ->copyable(),

                TextColumn::make('productVariant.sku')
                    ->label('SKU')
                    ->fontFamily('mono')
                    ->searchable()
                    ->sortable(),

                TextColumn::make('dispatched_base_qty')
                    ->label('DISPATCHED')
                    ->numeric()
                    ->alignEnd()
                    ->sortable(),

                TextColumn::make('dispatched_at')
                    ->label('DISPATCHED AT')
                    ->dateTime('M j, Y H:i')
                    ->sortable()
                    ->visibleFrom('md'),

                TextColumn::make('status')
                    ->badge()
                    ->sortable(),
            ])
            ->filters([
                SelectFilter::make('status')->options(InTransitStatus::class),
            ])
            ->defaultSort('dispatched_at', 'desc')
            ->stackedOnMobile()
            ->paginated([25, 50, 100])
            ->defaultPaginationPageOption(50);
    }
}
```

---

### 7E. StockMovementResource

**Model:** `App\Models\StockMovement` · **Group:** AUDIT LEDGERS · **Sort:** 1

#### 7E.1 StockMovementInfolist.php

```php
namespace App\Filament\Resources\StockMovements\Schemas;

use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class StockMovementInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 2, 'xl' => 2])->schema([
                Section::make('MOVEMENT')
                    ->icon(Heroicon::QueueList)
                    ->columnSpanFull()
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('created_at')->label('TIMESTAMP')->dateTime('M j, Y H:i')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('type')->badge()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('productVariant.sku')->label('SKU')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('warehouse.name')->label('WAREHOUSE')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('quantity')->numeric()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('unit_name_used')->label('UNIT')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('reference_code')->label('REFERENCE')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('createdBy.name')->label('BY')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('notes')->columnSpanFull(),
                    ]),
            ]),
        ]);
    }
}
```

#### 7E.2 StockMovementsTable.php — **Standard Table + `stackedOnMobile()`**

```php
namespace App\Filament\Resources\StockMovements\Tables;

use App\Enums\StockMovementType;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;

class StockMovementsTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('created_at')
                    ->label('TIMESTAMP')
                    ->dateTime('M j, Y H:i')
                    ->sortable(),

                TextColumn::make('productVariant.sku')
                    ->label('SKU')
                    ->fontFamily('mono')
                    ->searchable()
                    ->sortable(),

                TextColumn::make('warehouse.code')
                    ->label('WAREHOUSE')
                    ->badge()
                    ->color('gray')
                    ->sortable()
                    ->visibleFrom('md'),

                TextColumn::make('type')
                    ->badge()
                    ->sortable(),

                TextColumn::make('quantity')
                    ->label('QTY')
                    ->numeric()
                    ->alignEnd()
                    ->sortable()
                    ->color(fn ($state) => $state >= 0 ? 'success' : 'danger')
                    ->weight('bold'),

                TextColumn::make('unit_name_used')
                    ->label('UNIT')
                    ->state(fn ($record) => $record->unit_ratio_used > 1
                        ? "{$record->unit_name_used} (×{$record->unit_ratio_used})"
                        : $record->unit_name_used)
                    ->visibleFrom('lg'),

                TextColumn::make('reference_code')
                    ->label('REFERENCE')
                    ->fontFamily('mono')
                    ->copyable()
                    ->searchable()
                    ->visibleFrom('md'),

                TextColumn::make('createdBy.name')
                    ->label('BY')
                    ->visibleFrom('xl'),
            ])
            ->filters([
                SelectFilter::make('type')->options(StockMovementType::class),
                SelectFilter::make('warehouse_id')
                    ->relationship('warehouse', 'name')
                    ->label('Warehouse')
                    ->searchable(),
                SelectFilter::make('product_variant_id')
                    ->label('Variant')
                    ->relationship('productVariant', 'sku')
                    ->searchable(),
                \App\Filament\Support\Filters\AdminReviewFilters::period('created_at')
                    ->authorize('viewAuditFilters'),
            ])
            ->defaultSort('created_at', 'desc')
            ->stackedOnMobile()
            ->paginated([25, 50, 100])
            ->defaultPaginationPageOption(50);
    }
}
```

---

### 7F. LossLedgerResource

**Model:** `App\Models\LossLedger` · **Group:** AUDIT LEDGERS · **Sort:** 2

#### 7F.1 LossLedgerInfolist.php

```php
namespace App\Filament\Resources\LossLedgers\Schemas;

use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class LossLedgerInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 2, 'xl' => 2])->schema([
                Section::make('LOSS RECORD')
                    ->icon(Heroicon::ExclamationTriangle)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('recorded_at')->dateTime('M j, Y H:i')->columnSpanFull(),
                        TextEntry::make('transferRequisition.reference_code')->label('REQUISITION')->columnSpanFull(),
                        TextEntry::make('productVariant.sku')->label('SKU')->columnSpanFull(),
                        TextEntry::make('warehouse.name')->label('WAREHOUSE')->columnSpanFull(),
                        TextEntry::make('loss_category')->badge()->columnSpanFull(),
                    ]),

                Section::make('FINANCIAL IMPACT')
                    ->icon(Heroicon::CurrencyDollar)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('lost_base_qty')->label('LOST (BASE)')->numeric()->columnSpanFull(),
                        TextEntry::make('damaged_base_qty')->label('DAMAGED (BASE)')->numeric()->columnSpanFull(),
                        TextEntry::make('unit_cost_price')
                            ->money(config('app.currency'), decimals: 4)->columnSpanFull(),
                        TextEntry::make('total_financial_loss')
                            ->money(config('app.currency'), decimals: 4)
                            ->weight('bold')->columnSpanFull(),
                    ]),
            ]),
        ]);
    }
}
```

#### 7F.2 LossLedgersTable.php — **Standard Table + `stackedOnMobile()`**

```php
namespace App\Filament\Resources\LossLedgers\Tables;

use Filament\Tables\Columns\Summarizers\Sum;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Table;

class LossLedgersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('recorded_at')
                    ->label('RECORDED')
                    ->dateTime('M j, Y H:i')
                    ->sortable(),

                TextColumn::make('transferRequisition.reference_code')
                    ->label('REQUISITION')
                    ->fontFamily('mono')
                    ->weight('bold')
                    ->searchable()
                    ->copyable(),

                TextColumn::make('productVariant.sku')
                    ->label('SKU')
                    ->fontFamily('mono')
                    ->searchable()
                    ->sortable(),

                TextColumn::make('warehouse.code')
                    ->label('WAREHOUSE')
                    ->badge()
                    ->color('gray')
                    ->sortable()
                    ->visibleFrom('md'),

                TextColumn::make('lost_base_qty')
                    ->label('LOST')
                    ->numeric()->alignEnd()->color('warning'),

                TextColumn::make('damaged_base_qty')
                    ->label('DAMAGED')
                    ->numeric()->alignEnd()->color('danger'),

                TextColumn::make('loss_category')
                    ->label('CATEGORY')
                    ->badge()
                    ->visibleFrom('md'),

                TextColumn::make('unit_cost_price')
                    ->label('UNIT COST')
                    ->money(config('app.currency'), decimals: 4)
                    ->visibleFrom('lg'),

                TextColumn::make('total_financial_loss')
                    ->label('TOTAL LOSS')
                    ->money(config('app.currency'), decimals: 4)
                    ->weight('bold')
                    ->alignEnd()
                    ->summarize(Sum::make()->money(config('app.currency'), decimals: 4)),

                TextColumn::make('recordedBy.name')
                    ->label('BY')
                    ->visibleFrom('xl'),
            ])
            ->filters([
                SelectFilter::make('loss_category')->options([
                    'shortfall' => 'Shortfall',
                    'damage'    => 'Damage',
                    'spoilage'  => 'Spoilage',
                    'theft'     => 'Theft',
                    'other'     => 'Other',
                ]),
                SelectFilter::make('warehouse_id')
                    ->relationship('warehouse', 'name')
                    ->label('Warehouse')
                    ->searchable(),
                \App\Filament\Support\Filters\AdminReviewFilters::period('recorded_at')
                    ->authorize('viewAuditFilters'),
            ])
            ->defaultSort('recorded_at', 'desc')
            ->stackedOnMobile()
            ->paginated([25, 50, 100])
            ->defaultPaginationPageOption(50);
    }
}
```

---

### 7G. PurchaseOrderResource

**Model:** `App\Models\PurchaseOrder` · **Group:** PURCHASING · **Sort:** 1

#### 7G.1 PurchaseOrderForm.php

```php
namespace App\Filament\Resources\PurchaseOrders\Schemas;

use App\Models\ProductVariantUnitConversion;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class PurchaseOrderForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            ...self::getSupplierWarehouseFields(),
            ...self::getLineItemsFields(),
            ...self::getReviewFields(),
        ]);
    }

    public static function getSupplierWarehouseFields(): array
    {
        return [
            Section::make('Supplier & Warehouse')
                ->icon(Heroicon::BuildingStorefront)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('supplier_id')
                        ->relationship('supplier', 'name')
                        ->prefixIcon(Heroicon::BuildingStorefront)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->searchable()
                        ->preload()
                        ->required()
                        ->createOptionForm(fn (Schema $schema) => \App\Filament\Resources\Suppliers\Schemas\SupplierForm::configure($schema)),

                    Select::make('warehouse_id')
                        ->label('Receiving Warehouse')
                        ->prefixIcon(Heroicon::BuildingOffice2)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->default(fn () => auth()->user()->warehouses()->count() === 1
                            ? auth()->user()->warehouses()->first()->id
                            : null)
                        ->required(),
                ]),
        ];
    }

    public static function getLineItemsFields(): array
    {
        return [
            Repeater::make('items')
                ->relationship()
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 4])
                ->schema([
                    Select::make('product_variant_id')
                        ->label('Variant (SKU)')
                        ->relationship('productVariant', 'sku')
                        ->prefixIcon(Heroicon::Tag)
                        ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->searchable()
                        ->preload()
                        ->required()
                        ->disableOptionsWhenSelectedInSiblingRepeaterItems()
                        ->live()
                        ->afterStateUpdated(function ($set) {
                            $set('ordered_unit_name', null);
                            $set('ordered_unit_ratio', null);
                        }),

                    Select::make('ordered_unit_name')
                        ->label('Unit')
                        ->prefixIcon(Heroicon::Scale)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(function (Get $get) {
                            $variantId = $get('product_variant_id');
                            if (! $variantId) {
                                return [];
                            }
                            $query = ProductVariantUnitConversion::where('product_variant_id', $variantId);
                            $flagged = (clone $query)->where('is_default_purchase', true)
                                ->orderByDesc('base_unit_ratio')
                                ->pluck('unit_name', 'unit_name')
                                ->toArray();
                            if (! empty($flagged)) {
                                return $flagged;
                            }
                            return $query->orderByDesc('base_unit_ratio')
                                ->pluck('unit_name', 'unit_name')
                                ->toArray();
                        })
                        ->required()
                        ->live()
                        ->afterStateUpdated(function (Get $get, $set, $state) {
                            $ratio = ProductVariantUnitConversion::where('product_variant_id', $get('product_variant_id'))
                                ->where('unit_name', $state)
                                ->value('base_unit_ratio');
                            $set('ordered_unit_ratio', $ratio ?? 1);
                        }),

                    TextInput::make('ordered_unit_ratio')
                        ->label('Ratio (base)')
                        ->hintIcon(Heroicon::InformationCircle)
                        ->hint('Auto-filled from selected unit')
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->disabled()
                        ->dehydrated()
                        ->required(),

                    TextInput::make('ordered_qty')
                        ->label('Qty')
                        ->prefixIcon(Heroicon::Hashtag)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->minValue(1)
                        ->required(),

                    TextInput::make('unit_cost_price')
                        ->label('Unit Cost')
                        ->prefixIcon(Heroicon::CurrencyDollar)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()
                        ->step(0.0001)
                        ->minValue(0)
                        ->required(),
                ])
                ->minItems(1)
                ->required()
                ->dehydrated()
                ->mutateRelationshipDataBeforeCreateUsing(function (array $data): array {
                    $data['ordered_base_qty'] = (int) $data['ordered_qty'] * (int) $data['ordered_unit_ratio'];
                    return $data;
                })
                ->mutateRelationshipDataBeforeSaveUsing(function (array $data): array {
                    $data['ordered_base_qty'] = (int) $data['ordered_qty'] * (int) $data['ordered_unit_ratio'];
                    return $data;
                }),
        ];
    }

    public static function getReviewFields(): array
    {
        return [
            Toggle::make('update_cost_price')
                ->label('Update catalog cost price on receipt')
                ->helperText('If enabled, receiving this PO will set each variant\'s current cost price to this order\'s unit cost, if different.')
                ->onIcon(Heroicon::CurrencyDollar)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->default(false),
        ];
    }
}
```

#### 7G.2 CreatePurchaseOrder.php

```php
namespace App\Filament\Resources\PurchaseOrders\Pages;

use App\Filament\Resources\PurchaseOrders\PurchaseOrderResource;
use App\Filament\Resources\PurchaseOrders\Schemas\PurchaseOrderForm;
use Filament\Forms\Components\Placeholder;
use Filament\Resources\Pages\CreateRecord;
use Filament\Resources\Pages\CreateRecord\Concerns\HasWizard;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;

class CreatePurchaseOrder extends CreateRecord
{
    use HasWizard;

    protected static string $resource = PurchaseOrderResource::class;

    protected function getSteps(): array
    {
        return [
            Step::make('Supplier & Warehouse')
                ->description('Select supplier and receiving warehouse')
                ->icon(Heroicon::BuildingStorefront)
                ->schema(PurchaseOrderForm::getSupplierWarehouseFields()),

            Step::make('Line Items')
                ->description('Pick variant, then unit, then qty and cost')
                ->icon(Heroicon::ClipboardDocumentList)
                ->schema(PurchaseOrderForm::getLineItemsFields()),

            Step::make('Review & Verify')
                ->description('Confirm order details')
                ->icon(Heroicon::CheckCircle)
                ->schema([
                    ...PurchaseOrderForm::getReviewFields(),
                    Placeholder::make('review_summary')
                        ->columnSpanFull()
                        ->content(fn (Get $get) => view(
                            'filament.wizards.purchase-order-review',
                            ['state' => $get()],
                        )),
                ]),
        ];
    }

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $data['reference_code'] = $data['reference_code']
            ?? 'PO-' . now()->format('YmdHis') . '-' . random_int(100, 999);
        $data['ordered_by'] = auth()->id();
        return $data;
    }

    public function getMaxContentWidth(): ?string
    {
        return Width::SevenExtraLarge->value;
    }
}
```

#### 7G.3 PurchaseOrdersTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\PurchaseOrders\Tables;

use App\Enums\PurchaseOrderStatus;
use App\Models\PurchaseOrder;
use Filament\Actions\Action;
use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Actions\ForceDeleteAction;
use Filament\Actions\RestoreAction;
use Filament\Actions\ViewAction;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
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
                Stack::make([
                    Split::make([
                        TextColumn::make('reference_code')
                            ->label('Reference')
                            ->fontFamily('mono')
                            ->weight(FontWeight::Bold)
                            ->searchable()
                            ->sortable()
                            ->copyable(),

                        TextColumn::make('status')
                            ->badge()
                            ->alignEnd()
                            ->sortable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('supplier.name')
                            ->label('Supplier')
                            ->icon(Heroicon::BuildingStorefront)
                            ->iconColor('gray')
                            ->searchable()
                            ->sortable(),

                        TextColumn::make('warehouse.name')
                            ->label('Warehouse')
                            ->icon(Heroicon::BuildingOffice2)
                            ->iconColor('gray')
                            ->sortable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('items_count')
                            ->label('Items')
                            ->counts('items')
                            ->badge()
                            ->color('gray')
                            ->numeric(),

                        TextColumn::make('ordered_at')
                            ->label('Ordered')
                            ->dateTime('M j, Y')
                            ->sortable()
                            ->placeholder('—'),

                        TextColumn::make('received_at')
                            ->label('Received')
                            ->dateTime('M j, Y')
                            ->sortable()
                            ->placeholder('—')
                            ->alignEnd(),
                    ])->from('lg'),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                SelectFilter::make('status')->options(PurchaseOrderStatus::class),
                SelectFilter::make('supplier_id')->relationship('supplier', 'name')->label('Supplier')->searchable(),
                SelectFilter::make('warehouse_id')->relationship('warehouse', 'name')->label('Warehouse')->searchable(),
                TrashedFilter::make(),
                \App\Filament\Support\Filters\AdminReviewFilters::period('ordered_at')
                    ->authorize('viewAuditFilters'),
            ])
            ->defaultSort('created_at', 'desc')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordUrl(fn (PurchaseOrder $record) => $record->getUrl('view'))
            ->recordActions([
                ViewAction::make(),

                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
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
                    ], true))
                    ->modalWidth(Width::FourExtraLarge)
                    ->schema(fn (PurchaseOrder $record) => collect($record->items)
                        ->map(function ($item) {
                            $outstandingBase = $item->outstandingBaseQty();
                            $outstandingDisplay = $item->ordered_unit_ratio > 1
                                ? round($outstandingBase / $item->ordered_unit_ratio, 2)
                                : $outstandingBase;
                            return TextInput::make("received.{$item->id}")
                                ->label("{$item->productVariant->sku} — outstanding {$outstandingDisplay} {$item->ordered_unit_name} ({$outstandingBase} base)")
                                ->prefixIcon(Heroicon::ArchiveBoxArrowDown)
                                ->columnSpan(['default' => 1, 'md' => 1])
                                ->numeric()
                                ->minValue(0)
                                ->maxValue($outstandingBase)
                                ->default($outstandingBase);
                        })
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
                    ->visible(fn (PurchaseOrder $record) => $record->canBeCancelled())
                    ->requiresConfirmation()
                    ->action(fn (PurchaseOrder $record) => app(\App\Services\PurchaseService::class)->cancelPurchaseOrder($record)),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete')
                    ->visible(fn (PurchaseOrder $record) => in_array($record->status, [
                        PurchaseOrderStatus::Draft,
                        PurchaseOrderStatus::Cancelled,
                    ], true)),

                RestoreAction::make()->icon(Heroicon::ArrowUturnLeft)->authorize('restore'),

                ForceDeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('forceDelete')
                    ->visible(fn () => auth()->user()->isAdmin()),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

#### 7G.4 PurchaseOrderInfolist.php

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
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('PURCHASE ORDER PROFILE')
                    ->icon(Heroicon::DocumentText)
                    ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('reference_code')
                            ->label('REFERENCE CODE')
                            ->weight(FontWeight::Bold)->size('lg')->copyable()->color('primary')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('status')->badge()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('supplier.name')->icon(Heroicon::BuildingStorefront)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('warehouse.name')->icon(Heroicon::BuildingOffice2)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('update_cost_price')
                            ->badge()->boolean()
                            ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2]),
                    ]),

                Section::make('SIGN-OFFS')
                    ->icon(Heroicon::ShieldCheck)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('orderedBy.name')->label('ORDERED BY')->icon(Heroicon::User)->placeholder('—'),
                        TextEntry::make('receivedBy.name')->label('RECEIVED BY')->icon(Heroicon::ArchiveBoxArrowDown)->placeholder('Pending Intake'),
                        TextEntry::make('ordered_at')->label('ORDERED AT')->dateTime('M j, Y H:i')->placeholder('—'),
                        TextEntry::make('received_at')->label('RECEIVED AT')->dateTime('M j, Y H:i')->placeholder('—'),
                    ]),

                Section::make('LINE ITEMS')
                    ->icon(Heroicon::ClipboardDocumentList)
                    ->columnSpanFull()
                    ->schema([
                        RepeatableEntry::make('items')
                            ->schema([
                                Grid::make(['default' => 1, 'md' => 3, 'xl' => 6])->schema([
                                    TextEntry::make('productVariant.sku')->label('SKU')->weight(FontWeight::Bold)
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('productVariant.name')->label('PRODUCT')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 2]),
                                    TextEntry::make('ordered_base_qty')->label('ORDERED (BASE)')->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('received_base_qty')->label('RECEIVED (BASE)')->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('unit_cost_price')->money(config('app.currency'), decimals: 4)
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                ]),
                            ]),
                    ]),
            ]),
        ]);
    }
}
```

---

### 7H. SalesOrderResource

**Model:** `App\Models\SalesOrder` · **Group:** SALES · **Sort:** 1

#### 7H.1 SalesOrderForm.php

```php
namespace App\Filament\Resources\SalesOrders\Schemas;

use App\Models\ProductVariantUnitConversion;
use Filament\Forms\Components\Placeholder;
use Filament\Forms\Components\Repeater;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class SalesOrderForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            ...self::getCustomerWarehouseFields(),
            ...self::getLineItemsFields(),
        ]);
    }

    public static function getCustomerWarehouseFields(): array
    {
        return [
            Section::make('Customer & Warehouse')
                ->icon(Heroicon::UserGroup)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('customer_id')
                        ->relationship('customer', 'name')
                        ->prefixIcon(Heroicon::UserGroup)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->searchable()->preload()->required()
                        ->createOptionForm(fn (Schema $schema) => \App\Filament\Resources\Customers\Schemas\CustomerForm::configure($schema)),

                    Select::make('warehouse_id')
                        ->label('Dispatching Warehouse')
                        ->prefixIcon(Heroicon::BuildingOffice2)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(fn () => auth()->user()->warehouses()->pluck('name', 'id'))
                        ->default(fn () => auth()->user()->warehouses()->count() === 1
                            ? auth()->user()->warehouses()->first()->id
                            : null)
                        ->required(),
                ]),
        ];
    }

    public static function getLineItemsFields(): array
    {
        return [
            Repeater::make('items')
                ->relationship()
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 4])
                ->schema([
                    Select::make('product_variant_id')
                        ->label('Variant (SKU)')
                        ->relationship('productVariant', 'sku')
                        ->prefixIcon(Heroicon::Tag)
                        ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->searchable()->preload()->required()
                        ->disableOptionsWhenSelectedInSiblingRepeaterItems()
                        ->live()
                        ->afterStateUpdated(function (Get $get, $set, $state) {
                            $set('unit_name', null);
                            $set('unit_ratio', null);
                            $variant = \App\Models\ProductVariant::with('currentPrice')->find($state);
                            $set('_current_sale_price_preview', $variant?->currentPrice?->sale_price ?? '0.0000');
                        }),

                    Select::make('unit_name')
                        ->label('Unit')
                        ->prefixIcon(Heroicon::Scale)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->options(function (Get $get) {
                            $variantId = $get('product_variant_id');
                            if (! $variantId) {
                                return [];
                            }
                            return ProductVariantUnitConversion::where('product_variant_id', $variantId)
                                ->orderByDesc('base_unit_ratio')
                                ->pluck('unit_name', 'unit_name')
                                ->toArray();
                        })
                        ->required()
                        ->live()
                        ->afterStateUpdated(function (Get $get, $set, $state) {
                            $ratio = ProductVariantUnitConversion::where('product_variant_id', $get('product_variant_id'))
                                ->where('unit_name', $state)
                                ->value('base_unit_ratio');
                            $set('unit_ratio', $ratio ?? 1);
                        }),

                    TextInput::make('unit_ratio')
                        ->label('Ratio (base)')
                        ->hintIcon(Heroicon::InformationCircle)
                        ->hint('Auto-filled from selected unit')
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()->disabled()->dehydrated()->required(),

                    TextInput::make('qty')
                        ->label('Qty')
                        ->prefixIcon(Heroicon::Hashtag)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->numeric()->minValue(1)->required(),

                    Placeholder::make('_current_sale_price_preview')
                        ->label('Catalog Sale Price')
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->content(fn (Get $get) => $get('_current_sale_price_preview') ?? '—'),
                ])
                ->minItems(1)->required()->dehydrated()
                ->mutateRelationshipDataBeforeCreateUsing(function (array $data): array {
                    $data['base_qty'] = (int) $data['qty'] * (int) $data['unit_ratio'];
                    return $data;
                })
                ->mutateRelationshipDataBeforeSaveUsing(function (array $data): array {
                    $data['base_qty'] = (int) $data['qty'] * (int) $data['unit_ratio'];
                    return $data;
                }),
        ];
    }
}
```

#### 7H.2 CreateSalesOrder.php

```php
namespace App\Filament\Resources\SalesOrders\Pages;

use App\Filament\Resources\SalesOrders\SalesOrderResource;
use App\Filament\Resources\SalesOrders\Schemas\SalesOrderForm;
use Filament\Forms\Components\Placeholder;
use Filament\Resources\Pages\CreateRecord;
use Filament\Resources\Pages\CreateRecord\Concerns\HasWizard;
use Filament\Schemas\Components\Utilities\Get;
use Filament\Schemas\Components\Wizard\Step;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;

class CreateSalesOrder extends CreateRecord
{
    use HasWizard;

    protected static string $resource = SalesOrderResource::class;

    protected function getSteps(): array
    {
        return [
            Step::make('Customer & Warehouse')
                ->description('Select customer and dispatching warehouse')
                ->icon(Heroicon::UserGroup)
                ->schema(SalesOrderForm::getCustomerWarehouseFields()),

            Step::make('Line Items')
                ->description('Pick variant, then unit, then qty')
                ->icon(Heroicon::ClipboardDocumentList)
                ->schema(SalesOrderForm::getLineItemsFields()),

            Step::make('Review & Verify')
                ->description('Confirm order details')
                ->icon(Heroicon::CheckCircle)
                ->schema([
                    Placeholder::make('review_summary')
                        ->columnSpanFull()
                        ->content(fn (Get $get) => view(
                            'filament.wizards.sales-order-review',
                            ['state' => $get()],
                        )),
                ]),
        ];
    }

    protected function mutateFormDataBeforeCreate(array $data): array
    {
        $data['reference_code'] = $data['reference_code']
            ?? 'SO-' . now()->format('YmdHis') . '-' . random_int(100, 999);
        $data['ordered_by'] = auth()->id();
        return $data;
    }

    public function getMaxContentWidth(): ?string
    {
        return Width::SevenExtraLarge->value;
    }
}
```

#### 7H.3 SalesOrdersTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\SalesOrders\Tables;

use App\Enums\SalesOrderStatus;
use App\Models\SalesOrder;
use Filament\Actions\Action;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Notifications\Notification;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Table;

class SalesOrdersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Stack::make([
                    Split::make([
                        TextColumn::make('reference_code')
                            ->label('Reference')
                            ->fontFamily('mono')
                            ->weight(FontWeight::Bold)
                            ->searchable()->sortable()->copyable(),

                        TextColumn::make('status')
                            ->badge()->alignEnd()->sortable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('customer.name')
                            ->label('Customer')
                            ->icon(Heroicon::UserGroup)->iconColor('gray')
                            ->searchable()->sortable(),

                        TextColumn::make('warehouse.name')
                            ->label('Warehouse')
                            ->icon(Heroicon::BuildingOffice2)->iconColor('gray')
                            ->sortable(),
                    ])->from('md'),

                    Split::make([
                        TextColumn::make('items_count')
                            ->label('Items')->counts('items')->badge()->color('gray')->numeric(),

                        TextColumn::make('confirmed_at')
                            ->label('Confirmed')->dateTime('M j, Y')->sortable()
                            ->placeholder('—')->visibleFrom('md'),

                        TextColumn::make('dispatched_at')
                            ->label('Dispatched')->dateTime('M j, Y')->sortable()
                            ->placeholder('—')->alignEnd(),
                    ])->from('lg'),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                \Filament\Tables\Filters\SelectFilter::make('status')->options(SalesOrderStatus::class),
                \Filament\Tables\Filters\SelectFilter::make('customer_id')->relationship('customer', 'name')->label('Customer')->searchable(),
                \Filament\Tables\Filters\SelectFilter::make('warehouse_id')->relationship('warehouse', 'name')->label('Warehouse')->searchable(),
                \Filament\Tables\Filters\TrashedFilter::make(),
                \App\Filament\Support\Filters\AdminReviewFilters::period('confirmed_at')
                    ->authorize('viewAuditFilters'),
            ])
            ->defaultSort('created_at', 'desc')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordUrl(fn (SalesOrder $record) => $record->getUrl('view'))
            ->recordActions([
                \Filament\Actions\ViewAction::make(),

                \Filament\Actions\EditAction::make()
                    ->icon(Heroicon::PencilSquare)
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
                    ], true))
                    ->modalWidth(Width::FourExtraLarge)
                    ->schema(function (SalesOrder $record) {
                        $variantIds = $record->items->pluck('product_variant_id')->unique()->all();

                        // Exclude this order's own reservation.
                        $availableByVariant = \App\Models\ProductVariant::batchAvailableQuantity(
                            $variantIds,
                            $record->warehouse_id,
                            $record->id,
                        );

                        return collect($record->items)
                            ->map(function ($item) use ($availableByVariant) {
                                $available = $availableByVariant[$item->product_variant_id] ?? 0;
                                $safeMax = min($item->outstandingBaseQty(), max(0, $available));

                                $helperText = null;
                                if ($available === 0) {
                                    $helperText = 'No stock available — dispatch blocked for this line.';
                                } elseif ($available < $item->outstandingBaseQty()) {
                                    $helperText = 'Insufficient stock for full dispatch — partial dispatch only.';
                                }

                                return TextInput::make("dispatch.{$item->id}")
                                    ->label("{$item->productVariant->sku} — outstanding {$item->outstandingBaseQty()} {$item->unit_name} (available: {$available})")
                                    ->prefixIcon(Heroicon::Truck)
                                    ->columnSpan(['default' => 1, 'md' => 1])
                                    ->numeric()
                                    ->minValue(0)
                                    ->maxValue($safeMax)
                                    ->default($safeMax)
                                    ->helperText($helperText);
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
                    ->label('RETURN')
                    ->icon(Heroicon::ArrowUturnLeft)
                    ->color('warning')
                    ->authorize('recordSalesReturn')
                    ->visible(fn (SalesOrder $record) => $record->items->contains(fn ($item) => $item->dispatched_base_qty > 0))
                    ->modalWidth(Width::Large)
                    ->schema([
                        Select::make('sales_order_item_id')
                            ->label('Line Item')
                            ->prefixIcon(Heroicon::ClipboardDocumentList)
                            ->columnSpan(['default' => 1, 'md' => 1])
                            ->options(fn (SalesOrder $record) => $record->items
                                ->where('dispatched_base_qty', '>', 0)
                                ->mapWithKeys(fn ($item) => [
                                    $item->id => "{$item->productVariant->sku} (dispatched: {$item->dispatched_base_qty}, already returned: {$item->alreadyReturnedBaseQty()})",
                                ]))
                            ->required()
                            ->live(),

                        TextInput::make('returned_base_qty')
                            ->label('Returned Qty (Base)')
                            ->prefixIcon(Heroicon::Hashtag)
                            ->columnSpan(['default' => 1, 'md' => 1])
                            ->numeric()
                            ->minValue(1)
                            ->maxValue(function (\Filament\Schemas\Components\Utilities\Get $get, SalesOrder $record) {
                                $itemId = $get('sales_order_item_id');
                                if (! $itemId) {
                                    return null;
                                }
                                $item = $record->items->firstWhere('id', (int) $itemId);
                                return $item ? ($item->dispatched_base_qty - $item->alreadyReturnedBaseQty()) : null;
                            })
                            ->required(),

                        Textarea::make('notes')
                            ->prefixIcon(Heroicon::ChatBubbleBottomCenterText)
                            ->columnSpanFull(),
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
                    ], true))
                    ->requiresConfirmation()
                    ->action(fn (SalesOrder $record) => app(\App\Services\SalesService::class)->cancelSalesOrder($record)),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

#### 7H.4 SalesOrderInfolist.php

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
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('SALES ORDER PROFILE')
                    ->icon(Heroicon::DocumentText)
                    ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('reference_code')->label('REFERENCE CODE')
                            ->weight(FontWeight::Bold)->size('lg')->copyable()->color('primary')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('status')->badge()
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('customer.name')->icon(Heroicon::UserGroup)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('warehouse.name')->icon(Heroicon::BuildingOffice2)
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                    ]),

                Section::make('SIGN-OFFS')
                    ->icon(Heroicon::ShieldCheck)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('orderedBy.name')->label('ORDERED BY')->icon(Heroicon::User)->placeholder('—'),
                        TextEntry::make('dispatchedBy.name')->label('DISPATCHED BY')->icon(Heroicon::Truck)->placeholder('Pending Dispatch'),
                        TextEntry::make('confirmed_at')->label('CONFIRMED AT')->dateTime('M j, Y H:i')->placeholder('—'),
                        TextEntry::make('dispatched_at')->label('DISPATCHED AT')->dateTime('M j, Y H:i')->placeholder('—'),
                    ]),

                Section::make('LINE ITEMS')
                    ->icon(Heroicon::ClipboardDocumentList)
                    ->columnSpanFull()
                    ->schema([
                        RepeatableEntry::make('items')
                            ->schema([
                                Grid::make(['default' => 1, 'md' => 3, 'xl' => 6])->schema([
                                    TextEntry::make('productVariant.sku')->label('SKU')->weight(FontWeight::Bold)
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('productVariant.name')->label('PRODUCT')
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 2]),
                                    TextEntry::make('base_qty')->label('ORDERED (BASE)')->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('dispatched_base_qty')->label('DISPATCHED (BASE)')->numeric()
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('unit_sale_price_snapshot')->label('SNAPSHOT PRICE')
                                        ->money(config('app.currency'), decimals: 4)
                                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                ]),
                            ]),
                    ]),
            ]),
        ]);
    }
}
```

---

### 7I. SupplierResource

**Model:** `App\Models\Supplier` · **Group:** PURCHASING · **Sort:** 2

#### 7I.1 SupplierForm.php

```php
namespace App\Filament\Resources\Suppliers\Schemas;

use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class SupplierForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')
                ->prefixIcon(Heroicon::BuildingStorefront)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->required()->maxLength(255),

            TextInput::make('contact_person')
                ->prefixIcon(Heroicon::User)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->maxLength(255),

            TextInput::make('phone')
                ->prefixIcon(Heroicon::Phone)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->tel(),

            TextInput::make('email')
                ->prefixIcon(Heroicon::Envelope)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->email(),

            Textarea::make('address')
                ->prefixIcon(Heroicon::MapPin)
                ->columnSpanFull(),

            Toggle::make('is_active')
                ->onIcon(Heroicon::CheckCircle)
                ->offIcon(Heroicon::XCircle)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->default(true),
        ]);
    }
}
```

#### 7I.2 SuppliersTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\Suppliers\Tables;

use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Actions\ForceDeleteAction;
use Filament\Actions\RestoreAction;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;

class SuppliersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Stack::make([
                    Split::make([
                        TextColumn::make('name')
                            ->weight(FontWeight::Bold)
                            ->searchable()->sortable(),

                        TextColumn::make('is_active')
                            ->label('Status')
                            ->badge()->alignEnd()
                            ->formatStateUsing(fn (bool $state) => $state ? 'Active' : 'Inactive')
                            ->color(fn (bool $state) => $state ? 'success' : 'danger'),
                    ])->from('md'),

                    TextColumn::make('contact_person')
                        ->label('Contact')
                        ->icon(Heroicon::User)->iconColor('gray')
                        ->searchable()->placeholder('—'),

                    Split::make([
                        TextColumn::make('phone')
                            ->icon(Heroicon::Phone)->iconColor('gray')
                            ->copyable()->placeholder('—'),

                        TextColumn::make('email')
                            ->icon(Heroicon::Envelope)->iconColor('gray')
                            ->copyable()->placeholder('—'),
                    ])->from('md'),

                    TextColumn::make('purchase_orders_count')
                        ->label('Purchase Orders')
                        ->counts('purchaseOrders')
                        ->badge()->color('primary')->numeric(),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                TernaryFilter::make('is_active'),
                TrashedFilter::make(),
            ])
            ->defaultSort('name')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordActions([
                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
                    ->modalWidth(Width::Large),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete'),

                RestoreAction::make()
                    ->icon(Heroicon::ArrowUturnLeft)
                    ->authorize('restore'),

                ForceDeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('forceDelete')
                    ->visible(fn () => auth()->user()->isAdmin()),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

---

### 7J. CustomerResource

**Model:** `App\Models\Customer` · **Group:** SALES · **Sort:** 2

#### 7J.1 CustomerForm.php

```php
namespace App\Filament\Resources\Customers\Schemas;

use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class CustomerForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')
                ->prefixIcon(Heroicon::UserGroup)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->required()->maxLength(255),

            TextInput::make('contact_person')
                ->prefixIcon(Heroicon::User)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->maxLength(255),

            TextInput::make('phone')
                ->prefixIcon(Heroicon::Phone)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->tel(),

            TextInput::make('email')
                ->prefixIcon(Heroicon::Envelope)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->email(),

            Textarea::make('address')
                ->prefixIcon(Heroicon::MapPin)
                ->columnSpanFull(),

            Toggle::make('is_active')
                ->onIcon(Heroicon::CheckCircle)
                ->offIcon(Heroicon::XCircle)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->default(true),
        ]);
    }
}
```

#### 7J.2 CustomersTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\Customers\Tables;

use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Actions\ForceDeleteAction;
use Filament\Actions\RestoreAction;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Filters\TrashedFilter;
use Filament\Tables\Table;

class CustomersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Stack::make([
                    Split::make([
                        TextColumn::make('name')->weight(FontWeight::Bold)->searchable()->sortable(),
                        TextColumn::make('is_active')
                            ->label('Status')->badge()->alignEnd()
                            ->formatStateUsing(fn (bool $state) => $state ? 'Active' : 'Inactive')
                            ->color(fn (bool $state) => $state ? 'success' : 'danger'),
                    ])->from('md'),

                    TextColumn::make('contact_person')
                        ->label('Contact')->icon(Heroicon::User)->iconColor('gray')
                        ->searchable()->placeholder('—'),

                    Split::make([
                        TextColumn::make('phone')->icon(Heroicon::Phone)->iconColor('gray')->copyable()->placeholder('—'),
                        TextColumn::make('email')->icon(Heroicon::Envelope)->iconColor('gray')->copyable()->placeholder('—'),
                    ])->from('md'),

                    TextColumn::make('sales_orders_count')
                        ->label('Sales Orders')
                        ->counts('salesOrders')
                        ->badge()->color('primary')->numeric(),
                ])->space(3),
            ])
            ->contentGrid(['md' => 2, 'xl' => 3])
            ->filters([
                TernaryFilter::make('is_active'),
                TrashedFilter::make(),
            ])
            ->defaultSort('name')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordActions([
                EditAction::make()->icon(Heroicon::PencilSquare)->modalWidth(Width::Large),
                DeleteAction::make()->icon(Heroicon::Trash)->authorize('delete'),
                RestoreAction::make()->icon(Heroicon::ArrowUturnLeft)->authorize('restore'),
                ForceDeleteAction::make()
                    ->icon(Heroicon::Trash)->authorize('forceDelete')
                    ->visible(fn () => auth()->user()->isAdmin()),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

---

### 7K. WarehouseResource

**Model:** `App\Models\Warehouse` · **Group:** SYSTEM ADMIN · **Sort:** 1

#### 7K.1 WarehouseForm.php — **Read-Only User Assignments**

```php
namespace App\Filament\Resources\Warehouses\Schemas;

use Filament\Forms\Components\Select;
use Filament\Forms\Components\Textarea;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class WarehouseForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Section::make('Warehouse Profile')
                ->icon(Heroicon::BuildingOffice)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    TextInput::make('code')
                        ->prefixIcon(Heroicon::Tag)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->required()->unique(ignoreRecord: true)->maxLength(50)
                        ->helperText('Short identifier, e.g. WH-CEBU-01'),

                    TextInput::make('name')
                        ->prefixIcon(Heroicon::Identification)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->required()->maxLength(255),

                    Textarea::make('location')
                        ->prefixIcon(Heroicon::MapPin)
                        ->columnSpanFull()->rows(2)->maxLength(500),
                ]),

            Section::make('Access & Status')
                ->icon(Heroicon::ShieldCheck)
                ->columnSpanFull()
                ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                ->schema([
                    Select::make('users')
                        ->relationship('users', 'name')
                        ->prefixIcon(Heroicon::UserGroup)
                        ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                        ->multiple()
                        ->searchable()
                        ->preload()
                        ->disabled()
                        ->dehydrated(false)
                        ->helperText('Manage warehouse assignments from the User resource. This view is read-only.'),

                    Toggle::make('is_active')
                        ->onIcon(Heroicon::CheckCircle)
                        ->offIcon(Heroicon::XCircle)
                        ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                        ->default(true),
                ]),
        ]);
    }
}
```

#### 7K.2 WarehousesTable.php — **Card Layout, No Bulk Actions**

```php
namespace App\Filament\Resources\Warehouses\Tables;

use App\Filament\Resources\Warehouses\WarehouseResource;
use Filament\Actions\DeleteAction;
use Filament\Actions\EditAction;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\Layout\Split;
use Filament\Tables\Columns\Layout\Stack;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Table;

class WarehousesTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                Stack::make([
                    Split::make([
                        TextColumn::make('code')
                            ->fontFamily('mono')
                            ->weight(FontWeight::Bold)
                            ->searchable()->sortable()->copyable()->copyMessage('Code copied'),

                        TextColumn::make('is_active')
                            ->label('Status')
                            ->badge()->alignEnd()
                            ->formatStateUsing(fn (bool $state) => $state ? 'Active' : 'Inactive')
                            ->color(fn (bool $state) => $state ? 'success' : 'danger'),
                    ])->from('md'),

                    TextColumn::make('name')
                        ->searchable()->sortable()->weight(FontWeight::SemiBold),

                    TextColumn::make('location')
                        ->icon(Heroicon::MapPin)->iconColor('gray')
                        ->searchable()->limit(60)->placeholder('—'),

                    Split::make([
                        TextColumn::make('users_count')
                            ->label('Staff')
                            ->counts('users')
                            ->badge()->color('primary')->numeric(),

                        TextColumn::make('stock_movements_count')
                            ->label('Ledger Entries')
                            ->counts('stockMovements')
                            ->badge()->color('gray')->numeric(),
                    ])->from('md'),
                ])->space(3),
            ])
            ->contentGrid([
                'md' => 2,
                'xl' => 3,
            ])
            ->filters([
                TernaryFilter::make('is_active')
                    ->label('Active status')
                    ->placeholder('All warehouses')
                    ->trueLabel('Active only')
                    ->falseLabel('Inactive only'),
            ])
            ->defaultSort('code')
            ->defaultPaginationPageOption(12)
            ->paginated([12, 24, 48])
            ->recordUrl(fn ($record) => WarehouseResource::getUrl('view', ['record' => $record]))
            ->recordActions([
                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
                    ->modalWidth(Width::Large),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete')
                    ->requiresConfirmation()
                    ->modalDescription('Warehouses with stock movement history or referenced documents cannot be deleted.'),
            ]);

        // Bulk actions intentionally omitted (F30).
    }
}
```

#### 7K.3 WarehouseInfolist.php

```php
namespace App\Filament\Resources\Warehouses\Schemas;

use Filament\Infolists\Components\RepeatableEntry;
use Filament\Infolists\Components\TextEntry;
use Filament\Schemas\Components\Grid;
use Filament\Schemas\Components\Section;
use Filament\Schemas\Schema;
use Filament\Support\Enums\FontWeight;
use Filament\Support\Icons\Heroicon;

class WarehouseInfolist
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                Section::make('WAREHOUSE PROFILE')
                    ->icon(Heroicon::BuildingOffice)
                    ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
                    ->schema([
                        TextEntry::make('code')->label('CODE')
                            ->weight(FontWeight::Bold)->size('lg')->copyable()->color('primary')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('name')->label('NAME')
                            ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                        TextEntry::make('location')->label('LOCATION')->placeholder('—')
                            ->columnSpanFull(),
                    ]),

                Section::make('STATUS')
                    ->icon(Heroicon::ShieldCheck)
                    ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                    ->schema([
                        TextEntry::make('is_active')
                            ->label('ACTIVE')->badge()
                            ->color(fn (bool $state) => $state ? 'success' : 'danger')
                            ->formatStateUsing(fn (bool $state) => $state ? 'Active' : 'Inactive'),
                        TextEntry::make('users_count')
                            ->label('ASSIGNED STAFF')
                            ->state(fn ($record) => $record->users()->count()),
                    ]),

                Section::make('ASSIGNED STAFF')
                    ->icon(Heroicon::UserGroup)
                    ->columnSpanFull()
                    ->schema([
                        RepeatableEntry::make('users')
                            ->schema([
                                Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])->schema([
                                    TextEntry::make('name')->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('email')->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                    TextEntry::make('role')->badge()->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1]),
                                ]),
                            ])
                            ->placeholder('No staff assigned to this warehouse.'),
                    ]),
            ]),
        ]);
    }
}
```

#### 7K.4 WarehouseResource.php

```php
namespace App\Filament\Resources\Warehouses;

use App\Filament\Resources\Warehouses\Pages\CreateWarehouse;
use App\Filament\Resources\Warehouses\Pages\EditWarehouse;
use App\Filament\Resources\Warehouses\Pages\ListWarehouses;
use App\Filament\Resources\Warehouses\Pages\ViewWarehouse;
use App\Filament\Resources\Warehouses\Schemas\WarehouseForm;
use App\Filament\Resources\Warehouses\Schemas\WarehouseInfolist;
use App\Filament\Resources\Warehouses\Tables\WarehousesTable;
use App\Models\Warehouse;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Table;

class WarehouseResource extends Resource
{
    protected static ?string $model = Warehouse::class;
    protected static string | \UnitEnum | null $navigationGroup = 'SYSTEM ADMIN';
    protected static ?int $navigationSort = 1;
    protected static ?string $recordTitleAttribute = 'name';
    protected static string | \BackedEnum | null $navigationIcon = Heroicon::OutlinedBuildingOffice;
    protected static string | \BackedEnum | null $activeNavigationIcon = Heroicon::BuildingOffice;

    public static function form(Schema $schema): Schema
    {
        return WarehouseForm::configure($schema);
    }

    public static function table(Table $table): Table
    {
        return WarehousesTable::configure($table);
    }

    public static function infolist(Schema $schema): Schema
    {
        return WarehouseInfolist::configure($schema);
    }

    public static function getPages(): array
    {
        return [
            'index'  => ListWarehouses::route('/'),
            'create' => CreateWarehouse::route('/create'),
            'view'   => ViewWarehouse::route('/{record}'),
            'edit'   => EditWarehouse::route('/{record}/edit'),
        ];
    }
}
```

---

### 7L. UserResource

**Model:** `App\Models\User` · **Group:** SYSTEM ADMIN · **Sort:** 2

#### 7L.1 UserForm.php

```php
namespace App\Filament\Resources\Users\Schemas;

use Filament\Forms\Components\Select;
use Filament\Forms\Components\TextInput;
use Filament\Forms\Components\Toggle;
use Filament\Schemas\Schema;
use Filament\Support\Icons\Heroicon;

class UserForm
{
    public static function configure(Schema $schema): Schema
    {
        return $schema->components([
            TextInput::make('name')
                ->prefixIcon(Heroicon::User)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->required()->maxLength(255),

            TextInput::make('email')
                ->prefixIcon(Heroicon::Envelope)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->email()->required()->unique(ignoreRecord: true),

            TextInput::make('password')
                ->prefixIcon(Heroicon::Key)
                ->password()->revealable()
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->dehydrated(fn ($state) => filled($state))
                ->required(fn (string $operation) => $operation === 'create'),

            Select::make('role')
                ->prefixIcon(Heroicon::ShieldCheck)
                ->columnSpan(['default' => 1, 'md' => 1, 'xl' => 1])
                ->options(\App\Enums\UserRole::class)
                ->required(),

            Select::make('warehouses')
                ->relationship('warehouses', 'name')
                ->prefixIcon(Heroicon::BuildingOffice)
                ->columnSpan(['default' => 1, 'md' => 2, 'xl' => 2])
                ->multiple()->searchable()->preload()
                ->helperText('This is the single source of truth for user-warehouse assignments.'),

            Toggle::make('is_active')
                ->onIcon(Heroicon::CheckCircle)
                ->offIcon(Heroicon::XCircle)
                ->columnSpanFull()
                ->default(true),
        ]);
    }
}
```

#### 7L.2 UsersTable.php — **Standard Table + `stackedOnMobile()`**

```php
namespace App\Filament\Resources\Users\Tables;

use Filament\Actions\BulkActionGroup;
use Filament\Actions\DeleteAction;
use Filament\Actions\DeleteBulkAction;
use Filament\Actions\EditAction;
use Filament\Support\Enums\Width;
use Filament\Support\Icons\Heroicon;
use Filament\Tables\Columns\IconColumn;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Filters\SelectFilter;
use Filament\Tables\Filters\TernaryFilter;
use Filament\Tables\Table;

class UsersTable
{
    public static function configure(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('name')
                    ->searchable()->sortable()->weight('bold'),

                TextColumn::make('email')
                    ->searchable()->copyable()->visibleFrom('md'),

                TextColumn::make('role')
                    ->badge()->sortable(),

                TextColumn::make('warehouses_count')
                    ->label('WAREHOUSES')
                    ->counts('warehouses')
                    ->numeric()->badge()->color('gray')->alignEnd(),

                IconColumn::make('is_active')
                    ->label('ACTIVE')
                    ->boolean()
                    ->trueIcon(Heroicon::CheckCircle)
                    ->falseIcon(Heroicon::XCircle)
                    ->trueColor('success')
                    ->falseColor('danger'),

                TextColumn::make('created_at')
                    ->label('CREATED')
                    ->dateTime('M j, Y')
                    ->sortable()
                    ->visibleFrom('lg'),
            ])
            ->filters([
                SelectFilter::make('role')->options(\App\Enums\UserRole::class),
                SelectFilter::make('warehouse_id')
                    ->label('Warehouse')
                    ->relationship('warehouses', 'name')
                    ->searchable(),
                TernaryFilter::make('is_active'),
            ])
            ->defaultSort('name')
            ->stackedOnMobile()
            ->paginated([25, 50, 100])
            ->defaultPaginationPageOption(50)
            ->recordActions([
                EditAction::make()
                    ->icon(Heroicon::PencilSquare)
                    ->modalWidth(Width::Large)
                    ->authorize('update'),

                DeleteAction::make()
                    ->icon(Heroicon::Trash)
                    ->authorize('delete'),
            ])
            ->toolbarActions([
                BulkActionGroup::make([
                    DeleteBulkAction::make()->icon(Heroicon::Trash)->authorize('deleteAny'),
                ]),
            ]);
    }
}
```

---

## 📐 Section 7M: Filament v5 Layout & Styling Components

### 7M.1 Grid System Fundamentals

```php
Grid::make(2)                                        // 2 columns on lg+
Grid::make(['default' => 1, 'md' => 2, 'xl' => 4])   // breakpoint array

TextInput::make('name')->columnSpan(2)               // 2 cols on lg+
TextInput::make('notes')->columnSpanFull()           // full width all devices
TextInput::make('sku')->columnSpan(['md' => 2, 'xl' => 1])
```

### 7M.2 Section Component

```php
Section::make('Routing Pathways')
    ->description('Define origin and destination warehouses')
    ->icon(Heroicon::BuildingOffice)
    ->columnSpanFull()
    ->columns(['default' => 1, 'md' => 2, 'xl' => 2])
    ->schema([...])
```

### 7M.3 Fieldset Component

```php
Fieldset::make('Line Item Details')
    ->schema([
        Select::make('product_variant_id')->prefixIcon(Heroicon::Tag)->required(),
        TextInput::make('qty')->prefixIcon(Heroicon::Hashtag)->numeric()->required(),
    ])
    ->columns(3)
```

### 7M.4 Tabs Component

```php
Tabs::make('Order Details')
    ->persistTabInQueryString()
    ->tabs([
        Tab::make('Customer')->icon(Heroicon::User)->schema([...]),
        Tab::make('Line Items')->icon(Heroicon::ClipboardDocumentList)->schema([...]),
        Tab::make('Notes')->icon(Heroicon::ChatBubbleBottomCenterText)->schema([...]),
    ])
```

### 7M.5 Flex Component

```php
Flex::make([
    TextInput::make('first_name')->required(),
    TextInput::make('last_name')->required(),
])
    ->from('sm')
    ->justify('between')
    ->gap(4)
```

### 7M.6 Layout Component Selection Guide

| Scenario | Recommended Component |
|---|---|
| Two side-by-side fields with equal weight | `Grid::make(2)` |
| Full-width field below a 2-col row | `TextInput::make(...)->columnSpanFull()` |
| Themed card with heading + description | `Section::make('Title')->icon(Heroicon::...)->description('...')` |
| Lightweight grouping without card chrome | `Fieldset::make('Label')` |
| Reduce visual clutter in long forms | `Tabs::make()->tabs([...])` with `->icon()` on each tab |
| Unequal-width side-by-side fields | `Flex::make([...])->justify('between')` |
| Sign-off rows in infolist | `Flex::make([...])->justify('between')` |
| Repeater item rendered as a card | Wrap item schema in `Section::make()` and set `->columns(1)` on the Repeater |
| Wizard step with internal grouping | `Section` inside each `Step::make(...)->schema([...])` with step `->icon()` |

### 7M.7 Styling Consistency Rules

1. Wizard steps use `Section` with `->icon()`; the `Step` itself carries `->icon()`.
2. Infolists use `Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])` as outer wrapper.
3. Line-item repeaters use `->columns(['default' => 1, 'md' => 2, 'xl' => 4])`.
4. `Flex::justify('between')` for side-by-side key-value pairs in infolists.
5. Tabs use `->persistTabInQueryString()` for 3+ tabs.
6. `columnSpanFull()` for full-width fields on all breakpoints.
7. `Section::make()` with no heading for repeater item wrappers.
8. Heroicons on every Section, Tab, Step, Action, and semantically meaningful form field.

---

## 📐 Section 7N: Table Architecture — Card vs. Standard

### 7N.1 The Card Layout API

```php
// Native card layout
->contentGrid([
    'md' => 2, // 2 cards per row on tablet
    'xl' => 3, // 3 cards per row on desktop
])

// Card internal composition
Stack::make([...])->space(3)      // vertical grouping inside a card
Split::make([...])->from('md')    // side-by-side within a Stack, stacks on mobile
```

### 7N.2 The Standard Table Responsive Primitive

```php
->stackedOnMobile()               // preserves dense table on desktop, stacks on mobile
```

### 7N.3 Decision Matrix

| Table Shape | Primary Use Case | Presentation |
|---|---|---|
| **Document** (requisition, PO, SO) | Browse & action small set of rich records | `->contentGrid(['md' => 2, 'xl' => 3])` |
| **Ledger** (stock movements, loss ledgers) | Scan, sort, filter large set of homogeneous rows | Standard table + `->stackedOnMobile()` |
| **Monitor** (in-transits) | Live operational queue, read-only | Standard table + `->stackedOnMobile()` |
| **Master Data (high cardinality)** (products, warehouses) | Browse with visual richness | `->contentGrid()` |
| **Master Data (low cardinality)** (suppliers, customers) | Browse small set with rich detail | `->contentGrid()` |
| **Comparison Surface** (users) | Compare rows against each other by column | Standard table + `->stackedOnMobile()` |

### 7N.4 Per-Resource Verdict (Council Recorded)

| Resource | Layout | Rationale |
|---|---|---|
| `ProductsTable` | ✅ Card | Visual catalog browse |
| `WarehousesTable` | ✅ Card | Small set, rich detail |
| `TransferRequisitionsTable` | ✅ Card | Document-centric records |
| `PurchaseOrdersTable` | ✅ Card | Document-centric records |
| `SalesOrdersTable` | ✅ Card | Document-centric records |
| `SuppliersTable` | ✅ Card | Small master-data set |
| `CustomersTable` | ✅ Card | Small master-data set |
| `InTransitsTable` | ⚠️ Standard | Read-only monitor; high row count; scanning task |
| `StockMovementsTable` | ❌ Standard | Append-only ledger; hundreds of thousands of rows; signed quantities need column alignment |
| `LossLedgersTable` | ❌ Standard | Financial audit trail; `summarize()` aggregate |
| `UsersTable` | ⚠️ Standard | Comparison task, not browsing task |

### 7N.5 Cross-Cutting Table Rules

1. **Every table declares `->defaultSort()`** (F27).
2. **Every relational column has an eager-loaded relation** in `getEloquentQuery()` (F28).
3. **Card tables declare `->defaultPaginationPageOption(12)` and `->paginated([12, 24, 48])`** (F29).
4. **Ledger tables paginate at 50 with `->paginated([25, 50, 100])`.**
5. **Card tables declare no bulk actions** (F30). The native renderer does not render per-card checkboxes. To enable bulk selection on card tables, install `mkdev-grid-card-layout` and add the corresponding `BulkActionGroup` per resource.
6. **Signed quantity columns are color-coded** (`success` for positive, `danger` for negative).
7. **Audit filters are policy-gated, never visibility-gated.**

### 7N.6 Plugin Option

If per-card bulk selection is required, `mkdev-grid-card-layout` wraps the same `table()` definition and adds checkboxes without rewriting resources. It is the only council-sanctioned plugin for card tables.

---

## 📐 Section 7O: Responsive Column Spans Across All Filament v5 Constructs

### 7O.1 Breakpoint Reference (Tailwind CSS)

| Breakpoint | Min Width | Typical Device |
|---|---|---|
| `default` | 0px | Mobile phone |
| `sm` | 640px | Large phone / small tablet |
| `md` | 768px | Tablet portrait |
| `lg` | 1024px | Tablet landscape / small laptop |
| `xl` | 1280px | Desktop |
| `2xl` | 1536px | Large desktop |

### 7O.2 Forms — Field-Level Responsive Spans

Every form field declares `->columnSpan(['default' => X, 'md' => Y, 'xl' => Z])`. The canonical pattern across all forms is:

- Full-width primary identifiers (`name`, `location`, `address`, `notes`) → `default=1, md=2, xl=2`
- Paired fields (warehouse selects, unit + ratio, quantity + cost) → `default=1, md=1, xl=1`
- Repeater line-item variant selectors → `default=1, md=2, xl=2`
- Repeater numeric fields (unit, ratio, qty, cost, preview) → `default=1, md=1, xl=1`

### 7O.3 Wizards — Step-Level Responsive Configuration

| Wizard Step | Inner Container `columns()` |
|---|---|
| Routing Pathways | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| Material Manifest (req/PO) | `['default' => 1, 'md' => 2, 'xl' => 4]` |
| Review & Verify | `['default' => 1]` |
| Location Mapping | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| Stock Allocation | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| Supplier & Warehouse | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| Customer & Warehouse | `['default' => 1, 'md' => 2, 'xl' => 2]` |

### 7O.4 Infolists

Infolists use `Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])` as outer wrapper. Document profile sections occupy `columnSpan=2`; sign-off sections occupy `columnSpan=1`. Line-item repeatable entries use `Grid::make(['default' => 1, 'md' => 3, 'xl' => 6])`.

### 7O.5 Dashboard Widgets

Dashboard `getColumns()`:
```php
public function getColumns(): int | string | array
{
    return ['default' => 1, 'md' => 2, 'xl' => 4];
}
```

### 7O.6 Tables — Responsive Column Visibility

| Column Type | Mobile | Tablet | Desktop |
|---|---|---|---|
| Primary identifier | ✅ | ✅ | ✅ |
| Status badge | ✅ | ✅ | ✅ |
| Secondary entity name | ❌ `visibleFrom('md')` | ✅ | ✅ |
| Line counts, dates, notes | ❌ `visibleFrom('lg')` | ❌ | ✅ |
| Money amounts | ✅ | ✅ | ✅ |
| Actor names (createdBy) | ❌ `visibleFrom('xl')` | ❌ | ❌ at lg, ✅ at xl |

### 7O.7 Canonical Breakpoint Convention (F24)

| Tier | Breakpoint | Target | Behaviour |
|---|---|---|---|
| Tier 1 | `default` (< 768px) | Mobile | Everything stacks to 1 column |
| Tier 2 | `md` (≥ 768px) | Tablet | Wide components span 2 columns |
| Tier 3 | `xl` (≥ 1280px) | Desktop | Full bento layout |

### 7O.8 Styling Consistency Rules

1. Every field declares `->columnSpan([...])` with an explicit `default` key.
2. Every container declares `->columns([...])` with an explicit `default` key.
3. `columnSpanFull()` for single-field full-width rows.
4. `columnSpan(['md' => X, 'xl' => Y])` is the canonical responsive pattern.
5. Infolists use `Grid::make(['default' => 1, 'md' => 3, 'xl' => 3])` outer wrapper.
6. Widgets use `$columnSpan` as a breakpoint array.
7. Dashboard `getColumns()` returns a breakpoint array.
8. Tables use `visibleFrom()` for mobile hiding.
9. Repeaters declare `->columns()` plus per-field `columnSpan()`.
10. `columnStart()` and `columnOrder()` reserved for advanced asymmetric layouts.

---

## 🛡️ Section 8: Authorization — Policies

### 8.1 ProductPolicy

```php
namespace App\Policies;

use App\Models\Product;
use App\Models\User;

class ProductPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Product $product): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, Product $product): bool { return $user->isAdmin(); }
    public function delete(User $user, Product $product): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function restore(User $user, Product $product): bool { return $user->isAdmin(); }
    public function restoreAny(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user, Product $product): bool { return $user->isAdmin(); }
    public function forceDeleteAny(User $user): bool { return $user->isAdmin(); }
}
```

### 8.2 ProductVariantPolicy

```php
namespace App\Policies;

use App\Models\ProductVariant;
use App\Models\User;

class ProductVariantPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, ProductVariant $variant): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, ProductVariant $variant): bool { return $user->isAdmin(); }
    public function delete(User $user, ProductVariant $variant): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function restore(User $user, ProductVariant $variant): bool { return $user->isAdmin(); }
    public function restoreAny(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user, ProductVariant $variant): bool { return false; }
    public function forceDeleteAny(User $user): bool { return false; }
    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.3 TransferRequisitionPolicy

```php
namespace App\Policies;

use App\Models\TransferRequisition;
use App\Models\User;

class TransferRequisitionPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, TransferRequisition $r): bool
    {
        return $user->isAdmin()
            || $user->warehouses->contains($r->from_warehouse_id)
            || $user->warehouses->contains($r->to_warehouse_id);
    }

    public function create(User $user): bool
    {
        return $user->warehouses()->exists();
    }

    public function update(User $user, TransferRequisition $r): bool
    {
        return $r->status === \App\Enums\TransferRequisitionStatus::Draft
            && $user->warehouses->contains($r->from_warehouse_id);
    }

    public function delete(User $user, TransferRequisition $r): bool
    {
        return $user->isAdmin()
            && in_array($r->status, [
                \App\Enums\TransferRequisitionStatus::Draft,
                \App\Enums\TransferRequisitionStatus::Cancelled,
            ], true);
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function restore(User $user, TransferRequisition $r): bool
    {
        return $user->isAdmin();
    }

    public function restoreAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDelete(User $user, TransferRequisition $r): bool
    {
        return $user->isAdmin();
    }

    public function forceDeleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function submitRequest(User $user, TransferRequisition $r): bool
    {
        return $r->status === \App\Enums\TransferRequisitionStatus::Draft
            && $user->warehouses->contains($r->from_warehouse_id);
    }

    public function confirm(User $user, TransferRequisition $r): bool
    {
        return $user->isAdmin()
            || $user->warehouses->contains($r->to_warehouse_id);
    }

    public function dispatch(User $user, TransferRequisition $r): bool
    {
        return $user->warehouses->contains($r->from_warehouse_id);
    }

    public function receive(User $user, TransferRequisition $r): bool
    {
        return $user->warehouses->contains($r->to_warehouse_id);
    }

    public function recordLoss(User $user, TransferRequisition $r): bool
    {
        return $user->warehouses->contains($r->to_warehouse_id);
    }

    public function cancel(User $user, TransferRequisition $r): bool
    {
        return $r->canBeCancelled()
            && ($user->isAdmin() || $user->warehouses->contains($r->from_warehouse_id));
    }

    public function negotiate(User $user, TransferRequisition $r): bool
    {
        return in_array($r->status, [
            \App\Enums\TransferRequisitionStatus::Requested,
            \App\Enums\TransferRequisitionStatus::UnderReviewFulfiller,
            \App\Enums\TransferRequisitionStatus::UnderReviewRequestor,
        ], true) && (
            $user->warehouses->contains($r->from_warehouse_id)
            || $user->warehouses->contains($r->to_warehouse_id)
        );
    }

    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.4 DirectTransferPolicy

```php
namespace App\Policies;

use App\Models\User;

class DirectTransferPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return $user->warehouses()->count() >= 1;
    }
}
```

### 8.5 InTransitPolicy

```php
namespace App\Policies;

use App\Models\InTransit;
use App\Models\User;

class InTransitPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, InTransit $t): bool
    {
        return true;
    }
}
```

### 8.6 StockMovementPolicy

```php
namespace App\Policies;

use App\Models\StockMovement;
use App\Models\User;

class StockMovementPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, StockMovement $m): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return false;
    }

    public function update(User $user, StockMovement $m): bool
    {
        return false;
    }

    public function delete(User $user, StockMovement $m): bool
    {
        return false;
    }

    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.7 LossLedgerPolicy

```php
namespace App\Policies;

use App\Models\LossLedger;
use App\Models\User;

class LossLedgerPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, LossLedger $l): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return false;
    }

    public function update(User $user, LossLedger $l): bool
    {
        return false;
    }

    public function delete(User $user, LossLedger $l): bool
    {
        return false;
    }

    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.8 PurchaseOrderPolicy

```php
namespace App\Policies;

use App\Models\PurchaseOrder;
use App\Models\User;

class PurchaseOrderPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, PurchaseOrder $o): bool
    {
        return $user->isAdmin() || $user->warehouses->contains($o->warehouse_id);
    }

    public function create(User $user): bool
    {
        return $user->warehouses()->exists();
    }

    public function update(User $user, PurchaseOrder $o): bool
    {
        return $o->status === \App\Enums\PurchaseOrderStatus::Draft
            && $user->warehouses->contains($o->warehouse_id);
    }

    public function delete(User $user, PurchaseOrder $o): bool
    {
        return in_array($o->status, [
            \App\Enums\PurchaseOrderStatus::Draft,
            \App\Enums\PurchaseOrderStatus::Cancelled,
        ], true) && $user->warehouses->contains($o->warehouse_id);
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function restore(User $user, PurchaseOrder $o): bool
    {
        return $user->isAdmin();
    }

    public function restoreAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function forceDelete(User $user, PurchaseOrder $o): bool
    {
        return $user->isAdmin();
    }

    public function forceDeleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function orderPurchase(User $user, PurchaseOrder $o): bool
    {
        return $user->warehouses->contains($o->warehouse_id);
    }

    public function receivePurchase(User $user, PurchaseOrder $o): bool
    {
        return $user->warehouses->contains($o->warehouse_id);
    }

    public function cancelPurchase(User $user, PurchaseOrder $o): bool
    {
        return $o->canBeCancelled()
            && ($user->isAdmin() || $user->warehouses->contains($o->warehouse_id));
    }

    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.9 SalesOrderPolicy

```php
namespace App\Policies;

use App\Models\SalesOrder;
use App\Models\User;

class SalesOrderPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, SalesOrder $o): bool
    {
        return $user->isAdmin() || $user->warehouses->contains($o->warehouse_id);
    }

    public function create(User $user): bool
    {
        return $user->warehouses()->exists();
    }

    public function update(User $user, SalesOrder $o): bool
    {
        return $o->status === \App\Enums\SalesOrderStatus::Draft
            && $user->warehouses->contains($o->warehouse_id);
    }

    public function delete(User $user, SalesOrder $o): bool
    {
        return $user->isAdmin();
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function confirmSalesOrder(User $user, SalesOrder $o): bool
    {
        return $user->warehouses->contains($o->warehouse_id);
    }

    public function dispatchSale(User $user, SalesOrder $o): bool
    {
        return $user->warehouses->contains($o->warehouse_id);
    }

    public function recordSalesReturn(User $user, SalesOrder $o): bool
    {
        return $user->warehouses->contains($o->warehouse_id);
    }

    public function cancelSalesOrder(User $user, SalesOrder $o): bool
    {
        return $user->isAdmin() || $user->warehouses->contains($o->warehouse_id);
    }

    public function viewAuditFilters(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }
}
```

### 8.10 SupplierPolicy

```php
namespace App\Policies;

use App\Models\Supplier;
use App\Models\User;

class SupplierPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Supplier $s): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, Supplier $s): bool { return $user->isAdmin(); }
    public function delete(User $user, Supplier $s): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function restore(User $user, Supplier $s): bool { return $user->isAdmin(); }
    public function restoreAny(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user, Supplier $s): bool { return $user->isAdmin(); }
    public function forceDeleteAny(User $user): bool { return $user->isAdmin(); }
}
```

### 8.11 CustomerPolicy

```php
namespace App\Policies;

use App\Models\Customer;
use App\Models\User;

class CustomerPolicy
{
    public function viewAny(User $user): bool { return true; }
    public function view(User $user, Customer $customer): bool { return true; }
    public function create(User $user): bool { return $user->isAdmin(); }
    public function update(User $user, Customer $customer): bool { return $user->isAdmin(); }
    public function delete(User $user, Customer $customer): bool { return $user->isAdmin(); }
    public function deleteAny(User $user): bool { return $user->isAdmin(); }
    public function restore(User $user, Customer $customer): bool { return $user->isAdmin(); }
    public function restoreAny(User $user): bool { return $user->isAdmin(); }
    public function forceDelete(User $user, Customer $customer): bool { return $user->isAdmin(); }
    public function forceDeleteAny(User $user): bool { return $user->isAdmin(); }
}
```

### 8.12 WarehousePolicy

```php
namespace App\Policies;

use App\Models\User;
use App\Models\Warehouse;

class WarehousePolicy
{
    public function viewAny(User $user): bool
    {
        return $user->isAdmin() || $user->isAuditor();
    }

    public function view(User $user, Warehouse $w): bool
    {
        return $user->isAdmin()
            || $user->isAuditor()
            || $user->warehouses->contains($w->id);
    }

    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    public function update(User $user, Warehouse $w): bool
    {
        return $user->isAdmin();
    }

    /**
     * A warehouse may only be deleted when it has no ledger history and is
     * not referenced by any document (PO, SO, or transfer requisition).
     * This prevents raw FK violations from bubbling up as 500s.
     */
    public function delete(User $user, Warehouse $w): bool
    {
        if (! $user->isAdmin()) {
            return false;
        }

        if ($w->stockMovements()->exists()) {
            return false;
        }

        if ($w->purchaseOrders()->exists()) {
            return false;
        }

        if ($w->salesOrders()->exists()) {
            return false;
        }

        if ($w->transferRequisitionsFrom()->exists()) {
            return false;
        }

        if ($w->transferRequisitionsTo()->exists()) {
            return false;
        }

        return true;
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }
}
```

### 8.13 UserPolicy

```php
namespace App\Policies;

use App\Models\User;

class UserPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function view(User $user, User $model): bool
    {
        return $user->isAdmin() || $user->id === $model->id;
    }

    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    public function update(User $user, User $model): bool
    {
        return $user->isAdmin();
    }

    public function delete(User $user, User $model): bool
    {
        return $user->isAdmin() && $user->id !== $model->id;
    }

    public function deleteAny(User $user): bool
    {
        return $user->isAdmin();
    }
}
```

### 8.14 Policy Registration

In `AuthServiceProvider::boot()`:
```php
Gate::policy(Product::class, ProductPolicy::class);
Gate::policy(ProductVariant::class, ProductVariantPolicy::class);
Gate::policy(TransferRequisition::class, TransferRequisitionPolicy::class);
Gate::policy(InTransit::class, InTransitPolicy::class);
Gate::policy(StockMovement::class, StockMovementPolicy::class);
Gate::policy(LossLedger::class, LossLedgerPolicy::class);
Gate::policy(PurchaseOrder::class, PurchaseOrderPolicy::class);
Gate::policy(SalesOrder::class, SalesOrderPolicy::class);
Gate::policy(Supplier::class, SupplierPolicy::class);
Gate::policy(Customer::class, CustomerPolicy::class);
Gate::policy(Warehouse::class, WarehousePolicy::class);
Gate::policy(User::class, UserPolicy::class);
```

---

## 🔍 Section 9: Shared Filter Architecture

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

class AdminReviewFilters
{
    public static function warehouse(string $relationshipName = 'warehouse'): SelectFilter
    {
        return SelectFilter::make('warehouse_id')
            ->label('Warehouse')
            ->relationship($relationshipName, 'name')
            ->searchable()
            ->preload();
    }

    /**
     * Period filter. Must be policy-gated via ->authorize('viewAuditFilters') at call site.
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
                    ->columnSpan(['default' => 1, 'md' => 1])
                    ->live(),

                DatePicker::make('specific_date')
                    ->label('Date')
                    ->columnSpan(['default' => 1, 'md' => 1])
                    ->visible(fn (Get $get) => $get('preset') === 'specific_date'),

                DatePicker::make('range_from')
                    ->label('From')
                    ->columnSpan(['default' => 1, 'md' => 1])
                    ->visible(fn (Get $get) => $get('preset') === 'custom_range'),

                DatePicker::make('range_until')
                    ->label('Until')
                    ->columnSpan(['default' => 1, 'md' => 1])
                    ->visible(fn (Get $get) => $get('preset') === 'custom_range'),
            ])
            ->columns(['default' => 1, 'md' => 2])
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
            ->indicateUsing(function (array $data): string|array|null {
                return match ($data['preset'] ?? null) {
                    'today'         => 'Today',
                    'this_week'     => 'This week',
                    'this_month'    => 'This month',
                    'this_year'     => 'This year',
                    'specific_date' => isset($data['specific_date'])
                        ? 'On ' . Carbon::parse($data['specific_date'])->toFormattedDateString()
                        : null,
                    'custom_range'  => array_filter([
                        isset($data['range_from'])
                            ? Indicator::make('From ' . Carbon::parse($data['range_from'])->toFormattedDateString())
                                ->removeField('range_from')
                            : Indicator::make('No lower bound')
                                ->removeField('range_from'),
                        isset($data['range_until'])
                            ? Indicator::make('Until ' . Carbon::parse($data['range_until'])->toFormattedDateString())
                                ->removeField('range_until')
                            : Indicator::make('No upper bound')
                                ->removeField('range_until'),
                    ]),
                    default => null,
                };
            });
    }
}
```

---

## 📊 Section 10: Dashboard — Bento Grid & Widgets

### Colour Palette

| Token | Value | Usage |
|---|---|---|
| primary | #3b82f6 | Primary action execution triggers |
| surface | #ffffff | Card wrappers |
| surface-muted | #fafafa | Hover states |
| border | #e4e4e7 | 1px solid borders |
| text-primary | #18181b | Headings |
| text-secondary | #71717a | Labels |
| danger | #ef4444 | Loss, force-delete |
| warning | #f59e0b | Partial intake |
| success | #22c55e | Completed transfers |

### Elevation Rules

- Flat rest states (1px solid zinc-200).
- Shadows only on modal focus (`shadow-lg`).
- No zebra striping — thin dividers + hover highlights.

### Glassmorphic Bento Grid — Full Layout (All 9 Widgets)

```
┌─────────────────────────────┬───────────────┬───────────────┐
│                             │               │               │
│     StatsOverview           │   LowStock    │   Recent      │
│     (2 cols × 1 row)        │   (1 col)     │   Movements   │
│                             │               │   (1 col)     │
├─────────────────────────────┼───────────────┴───────────────┤
│                             │                               │
│     Sales Revenue Trend     │     Active In-Transit         │
│     (2 cols × 1 row)        │     (2 cols × 1 row)          │
├─────────────────────────────┼───────────────┬───────────────┤
│                             │               │               │
│     Sales vs Purchases      │   Top Selling │   Pending     │
│     (2 cols × 1 row)        │   Variants    │   Fulfillment │
├─────────────────────────────┴───────────────┴───────────────┤
│                    Quick Actions (4 cols)                   │
└─────────────────────────────────────────────────────────────┘
```

### Grid Layout Breakdown

| Row | Column Spans | Widgets |
|---|---|---|
| Row 1 | `[2, 1, 1]` | StatsOverview · LowStock · RecentMovements |
| Row 2 | `[2, 2]` | SalesRevenueTrend · ActiveInTransit |
| Row 3 | `[2, 1, 1]` | SalesVsPurchases · TopSellingVariants · PendingFulfillment |
| Row 4 | `[4]` | Quick Actions |

### Widget Column Span Configuration

| Widget | Mobile | `md` | `xl` | `$sort` |
|---|---|---|---|---|
| `StatsOverviewWidget` | 1 | 2 | 2 | 1 |
| `LowStockAlertsWidget` | 1 | 1 | 1 | 2 |
| `RecentMovementsWidget` | 1 | 1 | 1 | 3 |
| `SalesRevenueTrendWidget` | 1 | 2 | 2 | 4 |
| `ActiveInTransitWidget` | 1 | 2 | 2 | 5 |
| `SalesVsPurchasesWidget` | 1 | 2 | 2 | 6 |
| `TopSellingVariantsWidget` | 1 | 1 | 1 | 7 |
| `PendingFulfillmentWidget` | 1 | 1 | 1 | 8 |
| `QuickActionsWidget` | 1 | 2 | 4 | 9 |

### Widget Definitions

| Widget | Data Source | Cache TTL | Type | `$columnSpan` |
|---|---|---|---|---|
| `StatsOverviewWidget` | Total On-Hand, Pending Requisitions, Active In-Transit, Total Write-Off | 300s | TableWidget | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| `LowStockAlertsWidget` | Variants where `availableQuantity <= reorder_point` | 300s | ChartWidget (bar) | `['default' => 1, 'md' => 1, 'xl' => 1]` |
| `RecentMovementsWidget` | Recent `stock_movements` daily buckets, 7 days | 60s | ChartWidget (line) | `['default' => 1, 'md' => 1, 'xl' => 1]` |
| `SalesRevenueTrendWidget` | Daily Sale value, 30 days | 300s | ChartWidget (line) | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| `ActiveInTransitWidget` | InTransit rows where `status != cleared` | 300s | TableWidget | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| `SalesVsPurchasesWidget` | Side-by-side monthly Purchase vs Sale value, 6 months | 300s | ChartWidget (bar, grouped) | `['default' => 1, 'md' => 2, 'xl' => 2]` |
| `TopSellingVariantsWidget` | Top 10 variants by dispatched base qty, current month | 300s | ChartWidget (bar, horizontal) | `['default' => 1, 'md' => 1, 'xl' => 1]` |
| `PendingFulfillmentWidget` | Count of pending SO/PO, scoped | 60s | TableWidget | `['default' => 1, 'md' => 1, 'xl' => 1]` |
| `QuickActionsWidget` | Static shortcut buttons | — | Custom | `['default' => 1, 'md' => 2, 'xl' => 4]` |

### Implementation Examples

```php
class SalesRevenueTrendWidget extends ChartWidget
{
    protected static ?int $sort = 4;
    protected int | string | array $columnSpan = ['default' => 1, 'md' => 2, 'xl' => 2];
    // getType() returns 'line'; getData() scoped by warehouses
}

class QuickActionsWidget extends Widget
{
    protected static ?int $sort = 9;
    protected int | string | array $columnSpan = ['default' => 1, 'md' => 2, 'xl' => 4];
}
```

### Cache Key Reference

| Widget | Cache Key Pattern | TTL |
|---|---|---|
| `StatsOverviewWidget` | `stats_overview_{userId}_{firstWarehouseId}` | 300s |
| `LowStockAlertsWidget` | `low_stock_alerts_chart_{userId}_{firstWarehouseId}` | 300s |
| `RecentMovementsWidget` | `recent_movements_chart_{userId}_{firstWarehouseId}` | 60s |
| `SalesRevenueTrendWidget` | `sales_revenue_trend_{userId}_{firstWarehouseId}` | 300s |
| `ActiveInTransitWidget` | `active_in_transit_{userId}_{firstWarehouseId}` | 300s |
| `SalesVsPurchasesWidget` | `sales_vs_purchases_{userId}_{firstWarehouseId}` | 300s |
| `TopSellingVariantsWidget` | `top_selling_variants_{userId}_{firstWarehouseId}_{month}` | 300s |
| `PendingFulfillmentWidget` | `pending_fulfillment_{userId}_{firstWarehouseId}` | 60s |

### Widget Caching — `[ACCEPTED RISK]`

> **`[ACCEPTED RISK NOTE]`** The `LowStockAlertsWidget` iterates `ProductVariant` records and calls the `availableQuantity()` accessor per row, with the entire widget result wrapped in a single 300-second cache window. Every cache miss still issues 2N queries for N variants. At small-to-medium catalog sizes this is acceptable. At large catalog sizes (tens of thousands of variants), this will produce a spiky cache-refresh moment every 5 minutes and should be revisited.
>
> **Upgrade path:** replace the per-variant loop with a single grouped aggregate query. Red flag threshold: `product_variants` count exceeds ~5,000–10,000 active rows, or cache-miss load exceeds ~1–2 seconds.

### Role-Based Widget Visibility

| Widget | Visible To |
|---|---|
| `StatsOverviewWidget` | All authenticated users (scoped) |
| `LowStockAlertsWidget` | Admin, Auditor, Warehouse Staff |
| `RecentMovementsWidget` | All authenticated users |
| `SalesRevenueTrendWidget` | Admin, Auditor |
| `ActiveInTransitWidget` | Admin, Auditor, Warehouse Staff |
| `SalesVsPurchasesWidget` | Admin, Auditor |
| `TopSellingVariantsWidget` | Admin, Auditor |
| `PendingFulfillmentWidget` | All authenticated users (scoped) |
| `QuickActionsWidget` | All authenticated users |

### Dashboard Registration

```php
->widgets([
    \App\Filament\Widgets\StatsOverviewWidget::class,
    \App\Filament\Widgets\LowStockAlertsWidget::class,
    \App\Filament\Widgets\RecentMovementsWidget::class,
    \App\Filament\Widgets\SalesRevenueTrendWidget::class,
    \App\Filament\Widgets\ActiveInTransitWidget::class,
    \App\Filament\Widgets\SalesVsPurchasesWidget::class,
    \App\Filament\Widgets\TopSellingVariantsWidget::class,
    \App\Filament\Widgets\PendingFulfillmentWidget::class,
    \App\Filament\Widgets\QuickActionsWidget::class,
])
```

---

## 📋 Section 11: Master Execution Sequence (18 Phases)

**Phase 00: Environment & Core Guardrails Setup**

1. Bootstrap Laravel 13 with PostgreSQL.
2. Install FilamentPHP v5.
3. Install Livewire v4.
4. Install `simplesoftwareio/simple-qrcode`.
5. Install `pestphp/pest`.
6. Add `'currency' => env('APP_CURRENCY', 'PHP')` to `config/app.php`.
7. Verify `ext-bcmath` is enabled and declared.
8. Mandate `->strictAuthorization()` in `AdminPanelProvider`.
9. Enumerate every policy method before enabling strict mode.
10. Verify `HasWizard` trait availability on all wizard-based `CreateRecord` page classes.
11. Register `ProductVariantObserver` before any seeder creates variants.

**Phase 01: Relational Schema Migrations** — 20 tables in dependency order, including `in_transits.cleared_at` and `created_at` indexes on document tables.

**Phase 02: Base Seeders & Opening Ledger** — including base-unit self-conversion rows.

**Phase 03: Eloquent Model Projections & Enums** — derived stock methods, eight backed enums, both observers registered.

**Phase 04: Transactional Inventory Engine** — `InventoryService`, `NegotiationService`, `PurchaseService`, `SalesService`, `GuardsOutstandingQuantity`.

**Phase 05: Product Catalog Resource** — `ProductResource` bound to `ProductVariant`, `ManageUnitConversionsAction` guards base-unit row, **card-layout table, no bulk actions**.

**Phase 06: Price Snapshots & Unit Conversions.**

**Phase 07: Warehouses & Manual Adjustments** — `WarehouseResource` with card layout, read-only user pivot, and `WarehousePolicy` blocking referenced warehouses.

**Phase 08: Inter-Warehouse Requisition Wizard** — with responsive `columnSpan` on all fields.

**Phase 09: Negotiation Loop UI** — revision form with substitute-variant unit sourcing.

**Phase 10: Dispatch, In-Transit Monitor & Confirm Materialization** — `InTransitResource` with standard table + `stackedOnMobile()`, and in-transit `Cleared`/`Lost` transitions.

**Phase 11: Printable STN & Signed QR Route.**

**Phase 12: Scan-to-Receive Modal & Multi-Batch Intake** — first-scan detection driven by idempotency table, not `cleared_at`.

**Phase 13: Read-Only Audit Ledgers** — `StockMovementResource` and `LossLedgerResource` with dense tables and `summarize()` on loss.

**Phase 14: Purchases Module** — `PurchaseOrderResource` with card layout, warehouse-scoped badge, over-receive guard, item/variant locks on receive.

**Phase 15: Sales Module** — `SalesOrderResource` with card layout, warehouse-scoped badge, over-dispatch and over-return guards, own-reservation-excluding dispatch availability.

**Phase 16: Glassmorphic Bento Dashboard** — 9 widgets with responsive `$columnSpan`.

**Phase 17: Multi-Language Translation.**

**Phase 18: Automated CI/CD Testing.**

---

## 🧪 Section 12: Automated CI/CD Testing & E2E Validation Strategy

| Test Runner | Environment | Focus Area |
|---|---|---|
| Laravel Pint | Local / CI | Code style compliance |
| Pest PHP | SQLite (`:memory:`) | Unit, Feature, Service & Model tests |
| Playwright | PostgreSQL (Test DB) | Sequential multi-role E2E browser flows |

### Critical Pest Coverage Targets

- `ProductVariant::onHandQuantity()` / `reservedQuantity()` / `reservedForSalesQuantity()` / `availableQuantity()` correctness.
- `reservedQuantity()` and `batchAvailableQuantity()` honor `$excludeTransferRequisitionId`.
- `reservedForSalesQuantity()` and `batchAvailableQuantity()` honor `$excludeSalesOrderId`.
- `batchAvailableQuantity()` issues exactly 3 queries regardless of variant count.
- `batchUnitConversions()` issues exactly 1 query.
- `ProductVariantObserver` materializes base-unit self-conversion row on create.
- `TransferRequisition::canBeCancelled()` returns true only for the five pre-dispatch states.
- `PurchaseOrder::canBeCancelled()` returns false when any item has received quantity.
- `InventoryService::directTransfer()` locks warehouses in sorted-ID order.
- `InventoryService::dispatchTransfer()` locks items and variants; throws on insufficient availability.
- `InventoryService::scanToReceive()` no-ops on duplicate payload via state-equality check.
- `InventoryService::scanToReceive()` uses idempotency presence for first-scan detection, not `cleared_at`.
- `InventoryService::scanToReceive()` transitions `InTransit` rows to `Cleared` or `Lost`.
- `InventoryService::dispatchTransfer()` throws if `approved_base_qty` is null.
- `InventoryService::recordMovement()` throws on purchase/sale/sale_return/purchase_return types.
- `NegotiationService::submitRequest()` transitions Draft → Requested only.
- `NegotiationService::materializeRequestedAsApproved()` throws if any item has null approved qty.
- `NegotiationService::assertNegotiable()` rejects non-negotiable parent statuses.
- `PurchaseService::receivePurchase()` locks items and variants; guards over-receive; updates cost price when `update_cost_price = true`.
- `SalesService::recordSalesReturn()` guards cumulative over-return; locks variant and warehouse.
- `SalesService::dispatchSale()` locks items and variants; excludes own reservation in availability check; rejects dispatch when insufficient.
- Loss ledger `total_financial_loss` uses `bcmul()`, not float cast.
- `LossLedger::snapshotUnitCostFrom()` logs a warning when cost is missing or zero.
- All policies return expected booleans for each role.
- `WarehousePolicy::delete()` blocks warehouses with stock movements, POs, SOs, or TRs.
- QR lifetime = 7 days.

### Navigation Badge & Icon Tests

```
TransferRequisitionResourceTest::navigation_badge_is_warehouse_scoped()
PurchaseOrderResourceTest::navigation_badge_is_warehouse_scoped()
SalesOrderResourceTest::navigation_badge_is_warehouse_scoped()
NavigationBadgeTest::badge_count_is_computed_once_per_request()
AllResourcesTest::every_resource_declares_active_navigation_icon()
AllResourcesTest::no_resource_uses_raw_string_icon()
AllActionsTest::every_action_declares_heroicon_enum_icon()
AllFormFieldsTest::semantically_meaningful_fields_carry_prefix_icons()
```

### Responsive & Table Tests

```
ResponsiveSpanTest::all_form_fields_declare_explicit_default_breakpoint()
ResponsiveSpanTest::all_sections_declare_columns_with_default_key()
ResponsiveSpanTest::all_wizard_steps_declare_responsive_columns()
ResponsiveSpanTest::all_infolist_sections_declare_responsive_column_span()
ResponsiveSpanTest::all_widgets_declare_column_span_as_breakpoint_array()
ResponsiveSpanTest::dashboard_get_columns_returns_breakpoint_array()
ResponsiveSpanTest::all_repeaters_declare_columns_with_default_key()
ResponsiveSpanTest::all_repeater_fields_declare_column_span_with_default_key()
ResponsiveSpanTest::column_span_full_used_for_placeholder_review_summaries()
ResponsiveSpanTest::mobile_breakpoint_collapses_all_forms_to_single_column()
ResponsiveSpanTest::tablet_breakpoint_unstacks_wide_fields_to_two_columns()
ResponsiveSpanTest::desktop_breakpoint_achieves_full_bento_grid()
ResponsiveSpanTest::table_columns_use_visible_from_for_mobile_hiding()
ResponsiveSpanTest::no_raw_integer_column_span_without_breakpoint_array()
```

### Table Architecture Tests

```
TableArchitectureTest::document_tables_declare_content_grid()
TableArchitectureTest::ledger_tables_declare_stacked_on_mobile()
TableArchitectureTest::card_tables_declare_pagination_page_option()
TableArchitectureTest::all_tables_declare_default_sort()
TableArchitectureTest::all_relational_columns_have_eager_loaded_relations()
TableArchitectureTest::signed_quantity_columns_are_color_coded()
TableArchitectureTest::audit_filters_use_authorize_not_visible()
TableArchitectureTest::card_tables_do_not_declare_bulk_actions_without_plugin()
```

### Playwright E2E Scenarios

1. Full Transfer Lifecycle.
2. Direct Transfer.
3. Loss Write-Off.
4. Soft-Delete Guard.
5. Authorization Bypass Attempt.
6. Cancellation Boundary.
7. Duplicate Scan Submission.
8. Negotiation Loop.
9. Purchase Lifecycle (with over-receive attempt).
10. Sales Lifecycle (with over-return attempt).
11. Sales Cancellation Boundary.
12. Wizard Submit Button Visibility.
13. Unit Select Flow.
14. Base-Unit Deletion Guard.
15. Navigation Badge Visibility.
16. **Responsive Layout — Mobile:** resize to 375px → verify forms collapse to single column; card tables show 1 card per row; ledger tables use stacked layout.
17. **Responsive Layout — Tablet:** resize to 768px → verify card tables show 2 cards per row; wide fields span 2 columns.
18. **Responsive Layout — Desktop:** resize to 1440px → verify card tables show 3 cards per row; full bento dashboard; 4-column line-item repeaters.
19. **Sales dispatch self-reservation:** confirm an order with on-hand quantity sufficient for its own outstanding qty dispatches successfully.

---

## 📌 Section 13: Deferred to v13.4

1. **Event + notification layer** — `InventoryBelowReorderPoint`, `TransferDispatched`, `TransferReceived`, `LossRecorded`, `PurchaseOrderReceived`, `SalesOrderDispatched`.
2. **`[P0]` `->form()` vs `->schema()` on actions** — audit all Actions for `->form()` calls; replace with `->schema()`.
3. **`[P0]` Placeholder replacement** — replace `Placeholder` in wizard review steps with `WizardReviewStep` Livewire component.
4. **`[P0]` `createOptionForm` auto-select behaviour** — verify inline Product create auto-selects new record.
5. **Panel `->strictAuthorization()` role coverage** — enumerate every policy method.
6. **Low-stock widget scaling threshold** — revisit at 5,000–10,000 active variants.
7. **Supplier shipment / transit tracking for purchases.**
8. **Purchase-side negotiation.**
9. **FIFO / weighted-average / lot-level COGS costing.**
10. **`PurchaseReturn` full workflow UI.**
11. **Backorder auto-fulfillment.**
12. **Reporting-view decision** — separate Purchases/Sales tab on `StockMovementResource`.
13. **Filter-level authorization bypass hardening.**
14. **Per-card bulk selection** — requires `mkdev-grid-card-layout` plugin.
15. **`AdminReviewFilters::period()` — richer range presets** (last N days, YTD).

---

## ✅ Section 14: Cross-Cutting Verification Checklist

| Check | Status |
|---|---|
| `reservedQuantity()` counts Confirmed only, permanently and by design | ✅ |
| `reservedQuantity()` scope boundary is documented in-code | ✅ |
| `reservedForSalesQuantity()` is separate from `reservedQuantity()` | ✅ |
| `batchAvailableQuantity()` issues exactly 3 queries regardless of variant count | ✅ |
| `batchUnitConversions()` issues exactly 1 query regardless of variant count | ✅ |
| **Sales dispatch excludes own order reservation from availability** | ✅ |
| **Transfer dispatch excludes own requisition reservation from availability** | ✅ |
| **Purchase receive re-locks items and variants under parent transaction** | ✅ |
| **Sales dispatch re-locks items and variants under parent transaction** | ✅ |
| ForceDeleteAction absent from ProductResource | ✅ |
| All ledger product_variant_id FKs are restrictOnDelete | ✅ |
| `stock_movements.notes` column + service param | ✅ |
| `loss_ledgers.transfer_requisition_id` nullable | ✅ |
| `partially_received` has producer and consumer | ✅ |
| ConfirmAction calls `materializeRequestedAsApproved()` | ✅ |
| **`materializeRequestedAsApproved()` throws on null approved qty** | ✅ |
| `dispatchTransfer` / `scanToReceive` free of `??` fallbacks | ✅ |
| `dispatchTransfer` throws if `approved_base_qty` null | ✅ |
| **`InTransit` rows transition to `Cleared` / `Lost`** | ✅ |
| **First-scan detection uses idempotency table, not `cleared_at` on `in_transits`** | ✅ |
| ScanToReceiveAction named `scanToReceive` (camelCase) | ✅ |
| All wizard step-review components are Placeholder | ✅ |
| `RepeatableEntry` (not `RepeatEntry`) in all infolists | ✅ |
| `SoftDeletingScope` imported in `getEloquentQuery()` | ✅ |
| Enums route `getLabel()` through `__()` | ✅ |
| Policies exist and are wired via `->authorize()` | ✅ |
| **`ProductPolicy` exists and is registered** | ✅ |
| **`CustomerPolicy` exists and is registered** | ✅ |
| ProductObserver guards parent soft-delete | ✅ |
| QR lifetime = 7 days | ✅ |
| Direct-transfer list uses type + related_movement_id | ✅ |
| All action namespaces = `Filament\Actions\*` (^5.0) | ✅ |
| `->recordActions()` / `->toolbarActions()` (v5, not v3) | ✅ |
| `BulkActionGroup` wraps multiple bulk actions | ✅ |
| **Bulk actions omitted from all card tables (F30)** | ✅ |
| Section/Grid/Wizard from `Filament\Schemas\Components\*` | ✅ |
| `Get` from `Filament\Schemas\Components\Utilities\Get` | ✅ |
| `->money(config('app.currency'))` on all money columns | ✅ |
| `$navigationGroup` / `$navigationSort` specified per resource | ✅ |
| `->strictAuthorization()` mandated in panel provider | ✅ |
| Resource classes use thin delegation pattern | ✅ |
| Schema classes expose static `configure()` method | ✅ |
| `getRecordRouteBindingEloquentQuery()` overrides for soft-delete | ✅ |
| LossLedger model exists with `snapshotUnitCostFrom()` | ✅ |
| **`snapshotUnitCostFrom()` logs warning on missing/zero cost** | ✅ |
| `directTransfer()` locks warehouses in sorted-ID order | ✅ |
| `recordMovement()` and `directTransfer()` reject `unit_ratio < 1` | ✅ |
| **`recordMovement()` rejects purchase/sale/sale_return/purchase_return types** | ✅ |
| `scanToReceive()` no-ops on duplicate payload via state-equality check | ✅ |
| `total_financial_loss` computed via `bcmul()`, not float cast | ✅ |
| CancelAction restricted to five pre-dispatch states, both `->authorize()` and `->visible()` | ✅ |
| `ext-bcmath` declared as required PHP extension | ✅ |
| LowStockAlertsWidget scaling risk documented as accepted | ✅ |
| All Actions use `->schema()`, zero `->form()` calls | ✅ |
| Wizard review steps use `Placeholder` component | ✅ |
| `createOptionForm` auto-selects new option after save | ✅ |
| PurchaseOrderPolicy, SalesOrderPolicy, SupplierPolicy, CustomerPolicy, **ProductPolicy**, WarehousePolicy, UserPolicy exist | ✅ |
| No `->visible()` closure re-derives a permission decision | ✅ |
| No Service method contains a role check | ✅ |
| PolicyAuditTest exists and passes | ✅ |
| AdminReviewFilters reused across all applicable resources | ✅ |
| **AdminReviewFilters custom-range shows unbounded indicators** | ✅ |
| `batchAvailableQuantity()` used in `dispatchSale` modal | ✅ |
| `HasWizard` trait used on all wizard-based CreateRecord pages | ✅ |
| `getSteps()` returns `array<Step>` on all wizard pages | ✅ |
| Resource `form()` provides flat fields for Edit page | ✅ |
| Public static field helpers extracted on all wizard form classes | ✅ |
| Relationship-bound repeaters marked `->dehydrated()` | ✅ |
| `mutateRelationshipDataBeforeCreateUsing()` fires per item | ✅ |
| `mutateRelationshipDataBeforeSaveUsing()` fires per item on Edit | ✅ |
| `unit_sale_price_snapshot` remains at default until confirm-time | ✅ |
| Submit button only appears on last wizard step | ✅ |
| Unit fields are `Select`, never free-text `TextInput` | ✅ |
| Unit `Select` sources from variant's `product_variant_unit_conversions` | ✅ |
| `*_unit_ratio` is `->disabled()` + `->dehydrated()`, auto-filled via `->live()` | ✅ |
| `ProductVariantObserver` materializes base-unit self-conversion row on create | ✅ |
| Base-unit row has `unit_name = base_unit_name` and `base_unit_ratio = 1` | ✅ |
| `ManageUnitConversionsAction` disallows deletion of base-unit row | ✅ |
| **`ManageUnitConversionsAction` implementation present** | ✅ |
| Purchase unit `Select` prefers `is_default_purchase`, falls back to all | ✅ |
| Changing variant resets unit + ratio fields | ✅ |
| Layout components used per Section 7M rules | ✅ |
| Wizard steps use Section with icon where >2 fields | ✅ |
| Infolists use Grid::make(3) outer wrapper | ✅ |
| Tabs use `->persistTabInQueryString()` for 3+ tab resources | ✅ |
| Navigation groups registered in `->navigationGroups()` with icons and collapsibility | ✅ |
| Every resource declares `$navigationGroup` matching a registered group | ✅ |
| Every resource declares `$navigationSort` unique within its group | ✅ |
| Every resource declares `$navigationIcon` as `Heroicon` enum | ✅ |
| Every resource declares `$activeNavigationIcon` distinct from `$navigationIcon` | ✅ |
| All navigation badges warehouse-scoped | ✅ |
| **Badge counts computed once per request** | ✅ |
| Badge colors switch to `warning` above threshold of 10 | ✅ |
| Every Action declares `->icon()` with a `Heroicon` enum | ✅ |
| Every wizard `Step` declares `->icon()` | ✅ |
| Every infolist `Section` header carries a `Heroicon` | ✅ |
| No raw-string icon declarations anywhere in resources | ✅ |
| Card layout applied to ProductsTable | ✅ |
| Card layout applied to WarehousesTable | ✅ |
| Card layout applied to TransferRequisitionsTable | ✅ |
| Card layout applied to PurchaseOrdersTable | ✅ |
| Card layout applied to SalesOrdersTable | ✅ |
| Card layout applied to SuppliersTable | ✅ |
| Card layout applied to CustomersTable | ✅ |
| `stackedOnMobile()` applied to InTransitsTable | ✅ |
| `stackedOnMobile()` applied to StockMovementsTable | ✅ |
| `stackedOnMobile()` applied to LossLedgersTable | ✅ |
| `stackedOnMobile()` applied to UsersTable | ✅ |
| Every table declares `->defaultSort()` | ✅ |
| Card tables declare `->defaultPaginationPageOption(12)` | ✅ |
| Ledger tables paginate at 50 | ✅ |
| Signed quantity columns color-coded | ✅ |
| `total_financial_loss` summarized with `Sum::make()` | ✅ |
| `AdminReviewFilters::period()` uses `->authorize('viewAuditFilters')` | ✅ |
| All relational columns have eager-loaded relations | ✅ |
| **`WarehousePolicy::delete()` blocks warehouses with stock movements, POs, SOs, or TRs** | ✅ |
| `PurchaseOrder::canBeCancelled()` extracted as model method | ✅ |
| `TransferRequisition::canBeCancelled()` extracted as model method | ✅ |
| `SalesOrderItem::alreadyReturnedBaseQty()` extracted as model method | ✅ |
| `NegotiationService::submitRequest()` extracted from inline action | ✅ |
| **Substitute variants documented as transfer-only (A10)** | ✅ |
| **Document tables index `created_at`** | ✅ |
| **`user_warehouse` pivot edited from UserResource only** | ✅ |

---

## 📊 Section 15: Summary of All Changes (v13.2 → v13.3)

| # | Area | Resolution | Severity |
|---|---|---|---|
| 1 | Sales dispatch self-reservation double-count | `batchAvailableQuantity()` and `availableQuantity()` accept `$excludeSalesOrderId`; dispatch and modal pass order ID | Critical |
| 2 | First-scan detection permanently true | Replaced `cleared_at` existence check with `stock_movement_idempotency_keys` presence | Critical |
| 3 | In-transit rows never clear | `cleared_at` column added; `markInTransit()` transitions to `Cleared`/`Lost` | Critical |
| 4 | Purchase receive stale item race | Items and variants re-locked under parent transaction | High |
| 5 | Sales dispatch stale item race | Items and variants re-locked under parent transaction | High |
| 6 | Nullable `approved_base_qty` race | `materializeRequestedAsApproved()` validates; reservation queries filter `whereNotNull` | High |
| 7 | Missing `ProductPolicy` | Policy added and registered | High |
| 8 | Missing `CustomerPolicy` | Policy added and registered | High |
| 9 | Transfer dispatch no availability guard | `dispatchTransfer()` checks availability excluding own reservation | Medium |
| 10 | Zero-cost loss silent | `snapshotUnitCostFrom()` logs warning; loss note set when cost missing | Medium |
| 11 | Warehouse delete orphaning PO/SO/TR | `WarehousePolicy::delete()` extended | Medium |
| 12 | `recordSalesReturn` lock gap | Variant and warehouse locked | Medium |
| 13 | Substitute variant absence undocumented | Principle A10 added | Medium |
| 14 | Badge query duplication | Cached per-request via `private static ?int` | Medium |
| 15 | `recordMovement()` bypass footgun | Rejects purchase/sale/sale_return/purchase_return types | Medium |
| 16 | Document table `created_at` unindexed | Indexes added to `transfer_requisitions`, `purchase_orders`, `sales_orders` | Low |
| 17 | `ManageUnitConversionsAction` unverifiable | Full implementation provided | Low |
| 18 | Custom-range filter one-sided indicators | `No lower bound` / `No upper bound` indicators | Low |
| 19 | Card tables declared unusable bulk actions | Bulk actions removed; F30 added | Low |
| 20 | `user_warehouse` pivot dual edit surfaces | `WarehouseForm` users field made read-only; editing documented as UserResource-only | Low |

---

*End of blueprint v13.3.*