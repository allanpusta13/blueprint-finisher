# HRIS — Single-Company Edition — Complete Project Blueprint

> **Scope:** Single-company, private-sector-only Philippine HRIS. One installation serves one business, with any number of its own branches, employees, and admins. No multi-tenancy, no `stancl/tenancy`, no subscription/billing, no Super Admin platform-owner concepts, no Government Agency module.

---

## 📋 AI Governance Reference

AI-assisted project work is governed by the Vibe Coding Standard and project-specific `CLAUDE.md`:

- `docs/00-project/vibe-coding/standard.md` — Vibe Coding development standard: **WHAT + WHEN**
- `docs/00-project/vibe-coding/guideline.md` — Rationale and implementation guidance: **WHY + HOW TO THINK**
- `CLAUDE.md` — AI-facing project contract and enforcement: **HOW THE AI BEHAVES**

These documents define AI behavior, governance, and approval gates. The Blueprint describes the system/product.

## 📋 Reference Documents

The following document has been extracted from the original blueprint and preserved:

- **Historical Audit Results:** `docs/02-research/initial-council-audit-results.md` — Initial architecture and security council review

---

## 📦 Stack

|| Package | Version | Notes |
||---|---|---|
|| PHP | `^8.4` | Property Hooks, Asymmetric Visibility, `array_find()`, `array_first()` |
|| Laravel | `^13.0` | PHP 8.4+, first-party AI SDK, JSON:API Resources, Queue Routing, Semantic/Vector Search, `Cache::touch()`, PHP Attributes |
|| Filament | `^5.6` | Two panels: Admin, Employee Portal. Schema API for forms/infolists, Simple (modal) Resources |
|| Livewire | `^4.0` | Bundled with Filament 5 |
|| Alpine.js | `^3.x` | Bundled with Filament 5 / Livewire 4 |
|| Tailwind CSS | `^4.x` | Default via Laravel 13 Vite pipeline |
|| Spatie Activitylog | `^5.1` | Trait-only logging |
|| maatwebsite/excel | `^4.0` | Excel exports (v3.1 fails on Laravel 13) |
|| barryvdh/laravel-dompdf | `^3.0` | DejaVu Sans configured for Unicode |
|| laravel/sanctum | `^4.0` | Attendance device tokens, portal API tokens, public API tokens |
|| darkclow4/filament-map-picker | `^5.0` | Branch and employee site-location geofence maps |
|| saade/filament-fullcalendar | `^4.0` | Team and personal calendar views |
|| laravel/socialite | `^5.x` | Google Workspace / Microsoft 365 SSO |
|| spatie/laravel-backup | `^10.x` | Automated backup and point-in-time recovery |
|| vinkla/hashids | `^14.0` | Obfuscated public IDs — one salt per model |
|| laravel/ai | `^1.0` | First-party Laravel AI SDK (agents, embeddings, vector stores) |

### Dev Packages

|| Package | Version | Notes |
||---|---|---|
|| pestphp/pest | `^4.0` | Test runner with browser testing, datasets, arch tests |
|| pestphp/pest-plugin-laravel | `^4.0` | Laravel integration |
|| laravel/pint | `^1.0` | Code style |
|| laravel/horizon | `^5.0` | Queue monitoring |

### Optional

|| Package | Version | Notes |
||---|---|---|
|| spatie/laravel-medialibrary | `^11.x` | Richer file management |
|| spatie/laravel-settings | `^3.x` | Company-wide settings singleton |
|| bezhansalleh/filament-shield | `^4.x` | Auto-generated `RoleResource` in the Admin panel |

### Standard testing architecture

- **Pest v4:** Unit and Feature tests.
- **Standalone Playwright:** browser E2E, critical user flows, cross-browser, and responsive checks.
- Do **not** use Pest Browser as a second standard browser-testing architecture.
- Do not duplicate the same flow in Pest Browser and Playwright.

---

## 🔍 Product Discovery

### 0.1 Validate the problem

**Target Users:**
- HR administrators and managers for employee management
- Payroll officers for compensation processing
- Employees for self-service access
- Company leadership for reporting and analytics

**Current Problem:**
- Manual HR processes (paper-based or spreadsheets)
- Inefficient attendance tracking and leave management
- Lack of centralized employee data
- Manual payroll processing
- No self-service access for employees
- Limited reporting capabilities

**Existing Alternatives:**
- Other HRIS systems (SAP, Oracle, etc.)
- Spreadsheets (Excel, Google Sheets)
- Paper-based systems
- Outsourced HR services

**Riskiest Assumption:**
- User adoption of new digital system
- Integration with existing biometric devices
- Data migration from legacy systems
- Compliance with Philippine labor laws

### 0.2 Stack Validation

**Filament Appropriate for:**
- Admin/back-office HR management
- Employee self-service portal
- Reporting and analytics dashboards
- Configuration and settings management

**Filament Appropriate Because:**
- Built-in CRUD operations and resource management
- Powerful form and table components
- Role-based access control
- Responsive design out of the box
- Developer productivity

**Customer-Facing Experience:**
- Employee portal uses Filament (acceptable for internal company use)
- No public-facing customer portal required
- Focus on internal HR operations

### 0.3 Environment Requirements

**Required:**
- PHP 8.4+
- Composer 2.x
- Node 20+
- MySQL/MariaDB database
- Redis for queues and caching
- S3-compatible storage

**Database:**
- MySQL 8.0+ or MariaDB 10.6+
- InnoDB engine
- UTF8MB4 character set
- Collation: utf8mb4_unicode_ci

---

## 🎯 Phase 1: Plan

> Council-audit every individual task before marking it complete.

### 1.1 PRD Summary

**Problem Statement:**
Develop a single-company Philippine HRIS that automates HR processes, provides employee self-service, and ensures compliance with Philippine labor laws.

**Target Users:**
- HR administrators
- Payroll officers
- Department managers
- Employees
- Company leadership

**Core Features (MVP):**
- Employee management (profiles, assignments, documents)
- Time & attendance tracking
- Leave management
- Payroll processing
- Employee self-service portal
- Basic reporting

**User Flow:**
1. HR admin configures company settings, departments, positions
2. HR admin adds employees with basic information
3. Employees access portal to view profile, attendance, leave
4. Employees submit leave requests through portal
5. Managers approve/reject leave requests
6. HR processes payroll runs
7. Employees view payslips in portal
8. HR generates reports

**Success Metrics:**
- Employee data accuracy
- Attendance tracking accuracy
- Leave request processing time
- Payroll processing accuracy
- User adoption rate

**Roadmap:**
- MVP: Core HR, attendance, leave, payroll
- v1: Advanced features (performance, training, recruitment)
- v2: Compliance, analytics, integrations

### 1.2 Create `CLAUDE.md`

**Stack:**
- PHP 8.4+
- Laravel 13
- Filament v5
- Pest v4
- Standalone Playwright

**Filament v5 Rules:**
- Use Schema API for forms and infolists
- Use Simple (modal) Resources for ≤6-input forms
- Implement comprehensive Laravel Policies
- Use enum casting with Filament interfaces
- Implement resource-level authorization

**Directory Conventions:**
- `app/Models/` - Eloquent models
- `app/Services/` - Domain services
- `app/Filament/Resources/` - Filament resources
- `app/Filament/Resources/*/Pages/` - Resource pages
- `app/Filament/Resources/*/RelationManagers/` - Relation managers
- `app/Filament/Schemas/` - Form/infolist schemas
- `app/Filament/Tables/` - Table configurations
- `database/migrations/` - Database migrations
- `database/factories/` - Model factories
- `database/seeders/` - Database seeders
- `tests/Unit/` - Unit tests
- `tests/Feature/` - Feature tests
- `tests/e2e/` - Playwright E2E tests

**Code Style:**
- Follow Laravel 13 conventions
- Use PHP 8.4 features (property hooks, asymmetric visibility)
- Use Laravel AI SDK for ATS features
- Use Pest v4 for testing
- Use standalone Playwright for E2E

**Testing Architecture:**
- Pest v4 for unit and feature tests
- Standalone Playwright for E2E tests
- No Pest Browser duplication
- Test coverage for all critical paths

**Security Rules:**
- Comprehensive Laravel Policies for all domains using App\Enums\UserRole
- Encrypted casts for sensitive PII
- Input validation and sanitization
- File upload security
- PII masking in logs
- Session security with Redis

**Agent Behavior:**
- Follow Vibe Coding standard
- Council review for Phases 0-3
- No-guessing hierarchy
- Decision authority guidelines
- Security-first approach

**Council Rules:**
- Per-task Council review for Phases 0-3
- Five-perspective review (Product, Security, Architecture, QA, Skeptic)
- Council decision workflow
- Stop and ask for ambiguity/conflicts

**Commands:**
- Standard Laravel artisan commands
- Filament make commands
- Pest test commands
- Playwright test commands

**Package Decision Rules:**
- Default answer: no package
- Package Decision Checklist for dependencies
- User approval for material dependencies
- Security review for new packages

### 1.3 Ultra Plan

**Phase Breakdown:**
- Phase 0: Discovery (validation, stack fit, environment)
- Phase 1: Plan (PRD, CLAUDE.md, ultra plan, UI/UX, data model, package decisions)
- Phase 2: Build (foundation, auth, core resources, integration, security, debugging)
- Phase 3: Test & Refine (unit tests, feature tests, E2E, cleanup, git hygiene, hooks)
- Phase 4: Deploy & Monitor (CI/CD, environment, staging, monitoring, rollback)
- Phase 5: Document & Grow (README, architecture, skills, vault, retro)

**Task Structure:**
Each task includes:
- Goal
- Scope
- Inputs
- Expected output
- Verification
- Stop conditions

### 1.4 Spec-Driven Development

**Per Feature Definition:**
- Model + migration
- Filament Schema
- Table
- Authorization
- Edge cases
- Expected behavior
- Test expectations

**Implementation Rule:**
Do not write implementation merely because a spec is incomplete. Stop and ask for clarification.

### 1.5 UI/UX Design Brief

**Navigation:**
- Admin panel: Employee Management, Attendance, Leave, Payroll, Reports, Settings
- Portal panel: Dashboard, Profile, Attendance, Leave, Payslips, Notifications

**Table Layout:**
- Consistent column ordering
- Badge styling for status fields
- Search and filter capabilities
- Bulk actions where appropriate

**Forms:**
- Tabbed organization for complex forms
- Field-level validation
- Helper text and descriptions
- File upload controls

**Status Conventions:**
- Active: green badge
- Inactive: gray badge
- Pending: yellow badge
- Approved: green badge
- Rejected: red badge

**Responsive Behavior:**
- Mobile-friendly tables
- Responsive forms
- Touch-friendly actions

**Empty/Loading/Error States:**
- Empty state illustrations
- Loading spinners
- Error messages with guidance

### 1.6 Data Model & Relationships Map

**Core Entities:**
- User (authentication)
- CompanySettings (singleton)
- Branch (organizational unit)
- Department (hierarchical)
- Position (job level)
- Employee (central entity)
- EmployeeAssignment (work assignments)
- Shift (work schedule)
- AttendanceRecord (time tracking)
- LeaveType (leave policies)
- LeaveBalance (leave accruals)
- LeaveRequest (leave applications)
- PayrollRun (payroll batches)
- PayrollItem (individual payroll)

**Relationships:**
- User → Employee (one-to-one)
- Employee → Department (many-to-one)
- Employee → Position (many-to-one)
- Employee → Branch (many-to-one)
- Employee → EmployeeAssignments (one-to-many)
- Employee → AttendanceRecords (one-to-many)
- Employee → LeaveRequests (one-to-many)
- Employee → LeaveBalances (one-to-many)
- Employee → PayrollItems (one-to-many)

**Indexes and Constraints:**
- Foreign key indexes
- Unique constraints on employee_no, email
- Composite indexes for common queries
- Soft deletes where appropriate

**Data Ownership/Access:**
- HR admin: full access to all employee data
- Manager: access to department employees
- Employee: access to own data only
- Payroll officer: access to compensation data

### 1.7 Package Decision Checklist — HARD GATE

**Default answer: no package.**

**Package Categories:**
- Authorization: Native Laravel Policies with App\Enums\UserRole ✅ (built-in)
- Audit trail: Spatie Activitylog ✅ (approved)
- Excel/CSV exports: maatwebsite/excel ✅ (approved)
- Generated PDFs: barryvdh/laravel-dompdf ✅ (approved)
- API/mobile/device tokens: laravel/sanctum ✅ (approved)
- SSO/OAuth: laravel/socialite ✅ (approved)
- Off-site backups: spatie/laravel-backup ✅ (approved)
- AI/agents/embeddings/search: laravel/ai ✅ (approved)
- Rich media/file library: spatie/laravel-medialibrary (optional)
- Configurable application settings: spatie/laravel-settings (optional)
- Maps/geolocation/calendar: darkclow4/filament-map-picker, saade/filament-fullcalendar ✅ (approved)

**Approved Package List:**
- Core: Laravel 13, Filament v5, Livewire 4, Alpine.js, Tailwind 4
- Authorization: Native Laravel Policies, Spatie Activitylog
- File handling: maatwebsite/excel, barryvdh/laravel-dompdf
- API: laravel/sanctum
- Integrations: laravel/socialite, darkclow4/filament-map-picker, saade/filament-fullcalendar
- Backup: spatie/laravel-backup
- Security: vinkla/hashids
- AI: laravel/ai
- Dev: Pest v4, Laravel Pint, Laravel Horizon
- Optional: spatie/laravel-medialibrary, spatie/laravel-settings, bezhansalleh/filament-shield

---

## 🚀 Phase 2: Build

> Council-audit every individual task before marking it complete.

## 2A. Foundation

### 2.1 Laravel
Scaffold Laravel 13 and confirm the actual installed PHP/Laravel versions.

### 2.2 Filament
Install and configure Filament v5 with two panels (Admin and Portal).

### 2.3 Pest
Install/configure Pest v4 and Laravel integration.

### 2.4 Playwright
Install standalone Playwright and configure `playwright.config.ts`.

### 2.5 Environment/database
Configure `.env`, database, and baseline migrations.

### 2.6 Error Handling
Configure centralized error handling in `bootstrap/app.php` using the `->withExceptions` method to capture all errors in one place.

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->reportable(function (Throwable $e) {
        if ($this->app->environment('production')) {
            // Log to external service (Sentry, Bugsnag, etc.)
            \Log::channel('errors')->error($e->getMessage(), [
                'exception' => $e,
                'url' => request()->url(),
                'user' => auth()->id(),
                'user_role' => auth()->user()?->role ?? null,
            ]);
        }
    });

    $exceptions->renderable(function (Throwable $e, Request $request) {
        return response()->json([
            'error' => $e->getMessage(),
            'trace' => $this->app->environment('local') ? $e->getTrace() : null,
        ], $e instanceof HttpException ? $e->getStatusCode() : 500);
    });
})
```

### 2.7 Synchronize `CLAUDE.md`
Update `CLAUDE.md` to reflect the **actual** installed stack and approved packages.

---

## 2B. Auth & Users

### 2.8 Panel authentication
Configure Filament authentication for both Admin and Portal panels using Laravel session auth.

### 2.9 Panel access
Configure Filament panel access using App\Enums\UserRole enum-based authorization.

### 2.10 Policies
Create and wire a Policy for every Resource that requires authorization.

**Policy Implementation:**
- BasePolicy with common authorization logic
- EmployeePolicy (sensitive data protection, department access)
- LeaveRequestPolicy (approval workflow, self-service)
- PayrollPolicy (processing restrictions, finalization)
- AttendancePolicy (correction workflow, device access)
- DepartmentPolicy (department management, head access)
- BranchPolicy (branch management, geofence access)

---

## 2C. Core Resources

**Repeat per entity:**

### 2.11 Migration + Eloquent Model
Include relationships, casts, factories, indexes, constraints, and mass-assignment protection.

**Foundation Models:**
- CompanySettings (singleton)
- Branch (with geofencing)
- Department (hierarchical)
- Position (with salary levels)
- User (with App\Enums\UserRole enum casting)
- Employee (central entity with encrypted fields)

### 2.12 Filament Resource
Create the Resource using Filament v5 conventions.

**Admin Panel Resources:**
- CompanySettingsResource (simple resource)
- BranchResource (with map picker)
- DepartmentResource (hierarchical)
- PositionResource
- EmployeeResource (comprehensive)
- ShiftResource
- AttendanceRecordResource
- LeaveTypeResource
- LeaveRequestResource
- PayrollRunResource

**Portal Panel Resources:**
- MyProfileResource
- MyAttendanceResource
- MyLeaveResource
- MyPayslipResource

### 2.13 Schema
Use the Filament v5 **Schemas API** with enum integration.

**Schema Organization:**
- Extracted schemas for complex forms
- Inline schemas for simple forms
- Enum-based select fields with descriptions
- Tabbed form organization
- Section-based grouping

### 2.14 Table
Configure columns, filters, sorting, searching, row actions, and bulk actions.

**Table Features:**
- Automatic enum badge integration
- Enum-based filters
- Searchable columns
- Sortable columns
- Bulk actions with authorization
- Header actions

### 2.15 RelationManagers
Add only when the feature requires nested/related management.

**Relation Managers:**
- Employee → EmployeeAssignments
- Employee → EmployeeDocuments
- Employee → EmergencyContacts
- Department → Positions
- LeaveRequest → LeaveApprovals

### 2.16 Custom Pages/actions/widgets
Add only when required by the approved specification.

**Custom Pages:**
- Dashboard widgets
- Company settings page
- Payroll processing pages
- Reporting pages

---

## 2D. Integration

### 2.17 MCP, if required
If MCP is explicitly required, keep MCP changes within the approved MCP scope and stop before adding unapproved dependencies.

### 2.18 Database finalization
Finalize indexes, constraints, seeders, and development/demo data.

**Seeders:**
- CompanySettingsSeeder (singleton)
- BranchSeeder (main office)
- DepartmentSeeder (standard departments)
- PositionSeeder (organizational levels)
- HolidaySeeder (Philippine holidays)
- LeaveTypeSeeder (standard leave types)

### 2.19 Queues/Jobs
Use queues/jobs for work that is explicitly appropriate for asynchronous execution.

**Queue Jobs:**
- Attendance processing
- Leave notification jobs
- Payroll processing jobs
- Email notification jobs
- Webhook delivery jobs

---

## 🔐 2E. Security Pass — Continuous

Repeat after **every feature**, not only at the end.

### 2.20 Authentication
Verify authentication boundaries, session behavior, protected routes, and relevant login/recovery protections.

### 2.21 Authorization
Verify panel access, Resource Policies, record-level access, action/bulk-action authorization, and server-side enforcement.

**UI visibility is not authorization.**

### 2.22 Record-level access
Verify users cannot access, mutate, export, or infer records outside their permitted scope.

### 2.23 Mass assignment
Review `$fillable` / `$guarded` on every Model touched.

### 2.24 Input validation
Review request/form/schema validation, type constraints, ownership checks, and unsafe input paths.

### 2.25 File upload/storage security
Review MIME/type and size validation, storage disks, filenames/paths, visibility, and access controls.

### 2.26 Dependency security
Run `composer audit` and `npm audit` when applicable, especially after dependency changes.

### 2.27 Rate limiting/abuse protection
Review authentication and public-facing endpoints/actions for appropriate rate limiting and abuse controls.

---

## 🐛 2F. Debugging

### 2.28 Debug an Error

When fixing an error:
- Identify the root cause.
- Modify only directly relevant files.
- Do not perform unrelated refactors.
- Verify the fix.
- Stop if the fix requires a new dependency, destructive schema change, architectural change, or other material decision.

---

## 🧪 Phase 3: Test & Refine

> Council-audit every individual task before marking it complete.

### 3.1 Unit tests — Pest v4
`tests/Unit/`

Test non-trivial domain/model/service logic in isolation.

**Unit Test Coverage:**
- Model relationships
- Enum casting and methods
- Service methods
- Helper functions
- Enum Filament interface compliance

### 3.2 Feature tests — Pest v4
`tests/Feature/`

Test application behavior and Filament Resources, including:
- list/rendering behavior
- create
- edit
- delete
- relevant bulk actions
- Policy enforcement
- validation
- important edge cases

Use Filament's Livewire testing helpers where appropriate.

### 3.3 Playwright E2E — critical paths
`tests/e2e/`

Test complete user workflows that genuinely require a browser.

**Critical E2E Paths:**
- Employee creation and management
- Leave request workflow
- Attendance recording
- Payroll processing
- Employee portal access

### 3.4 Playwright — cross-browser/responsive
Run the approved E2E suite against the required browser matrix and responsive viewports.

The matrix may be reduced when the project's actual supported-browser requirements justify it.

### 3.5 Dead-code/quality cleanup
Remove unused imports, unreachable code, and artifacts created by the completed feature. Do not add a dependency solely for cleanup without approval.

### 3.6 Git hygiene
Create focused, meaningful commits appropriate to the project's workflow.

### 3.7 Hooks/guardrails
Configure pre-commit/CI checks only when approved by the project. Do not introduce hook-related packages without the Package Decision Checklist and user approval.

---

## 📦 Phase 4: Deploy & Monitor

Council per-step review is optional.

### 4.1 CI/CD
Run the project's approved test, style, and build checks before deployment.

### 4.2 Environment configuration
Use separate environment configuration and appropriate production caches/build steps.

### 4.3 Staging
Use staging for final production-like verification where appropriate.

### 4.4 Monitoring & logging
Configure error tracking, application logging, queue monitoring, and uptime monitoring according to project requirements.

Do not assume a particular monitoring vendor or package without a project decision.

### 4.5 Rollback plan
Document a practical rollback strategy, including database implications for migrations.

---

## 📚 Phase 5: Document & Grow

Council per-step review is optional.

### 5.1 README
Document setup, environment requirements, database setup, tests, and common development commands.

### 5.2 Architecture documentation
Document how Resources, Schemas, Policies, models, jobs, integrations, and other major components fit together.

### 5.3 Turn a task into a skill
Capture reusable implementation knowledge from important features.

### 5.4 Documentation Library

Store meaningful project documentation in the Tolaria Vault according to its documentation type. Keep the changelog concise and focused on meaningful project-level changes rather than individual code edits.

### 5.5 Retro & next iteration
Record:
- What took longer than expected and why
- What should change next time
- Highest-leverage next improvement

Feed useful lessons into the next project's Phase 0.

---

## ✅ Definition of Done

A Phase 0–3 task is complete only when:

1. The requested scope is implemented.
2. Verification has been performed.
3. The applicable Security Pass has been considered.
4. The Council has reviewed it from all five perspectives.
5. No unresolved ambiguity remains.
6. Required tests pass.
7. No unrelated scope was introduced.
8. `CLAUDE.md` remains consistent with the actual project when the task changes project reality.

**Evidence Before Done:** never claim a task is complete solely because code was written. State what was verified.

---

## 🏗️ Architecture Overview

The following sections contain the detailed HRIS-specific technical specifications for the system architecture, data model, and implementation patterns.

### Security Architecture Principles

The HRIS follows defense-in-depth security with multiple layers of protection:

1. **Authentication**: Laravel session auth + Sanctum tokens + SSO
2. **Authorization**: App\Enums\UserRole enum + Laravel Policies (fine-grained)
3. **Data Protection**: Encryption at rest + PII masking + secure transmission
4. **Input Validation**: Form Requests + sanitization + file upload security
5. **Session Security**: Redis + secure cookies + CSP headers
6. **Audit Trail**: Comprehensive logging with PII masking
7. **Network Security**: HTTPS + rate limiting + CORS policies

```text
Two Filament 5 Panels — both PWA-installable:

  /admin      AdminPanelProvider   → HR admins, managers, payroll officers
  /portal     PortalPanelProvider  → Employees only (self-service)

Attendance API (/api/attendance):
  3 endpoints: time-in, time-out, bulk sync

Biometric Hardware:
  ZKTeco ADMS/iClock HTTP push protocol; standalone Node.js TCP connector
  agent bridges older devices into POST /api/attendance/sync.

Geofencing — two tiers:
  Branch geofence (fixed location) and field-employee geofence (per-employee
  assigned site locations with manager-approval workflow).

Benefits & Compensation (Philippine private-sector labor law):
  13th Month Pay (PD 851), Night Differential (Labor Code Art. 86),
  Holiday Pay + Overtime (OTF/OTRD), De Minimis Benefits (BIR RR) with
  mid-year proration, SIL Cash Conversion, Separation/Final Pay,
  Retirement Pay (RA 7641), BIR 2316 + BIR 1604-C.

Leave Customization:
  Fully configurable leave types: custom names/codes, accrual rules,
  carry-over rules, cash conversion rate, n-level approval chain,
  pro-rating per employment type and tenure.

Employee Lifecycle:
  Recruitment (ATS) → Onboarding → Performance → Training →
  Contracts/Assets → Grievance/Disciplinary → Separation/Clearance → Final Pay

Data Privacy Act (RA 10173) Compliance:
  Consent tracking and Data Subject Access Request handling.

Platform Integrations:
  Outbound webhooks, scoped public read API, SSO (Google Workspace /
  Microsoft 365). AI-powered resume parsing, grievance sentiment analysis,
  and intelligent ticket routing via Laravel AI SDK. Semantic search over
  employee skills, job descriptions, and training content via pgvector.

Laravel 13 Features Utilized:
  PHP Attributes, Laravel AI SDK, JSON:API Resources, Queue Routing,
  Semantic/Vector Search, Cache::touch(), PreventRequestForgery middleware.

Filament 5 Features Utilized:
  Schema API for forms/infolists, Simple (modal) Resources for ≤6-input forms,
  Infolists, Actions, Clusters, unsavedChangesAlerts(), PWA.

Pest 4 Features Utilized:
  Datasets, Architecture tests, Browser tests (Playwright), Smoke testing,
  Sharding, `describe`/`it` grouping.
```

**Soft Delete:** All models use Laravel's `SoftDeletes` trait. Deleted records are preserved with `deleted_at`, restorable, and audited via Activitylog v5.

---

## 🎨 Filament v5 Panel Resource Mapping

### Admin Panel Resources Organization

```text
🏠 Dashboard
├── AdminDashboardPage
│   ├── Stats Overview Widget (StatsOverviewWidget)
│   ├── Branch Filter (SelectFilter for branch scope)
│   ├── Attendance Charts (Chart.js integration)
│   ├── Leave Summary Table (Table widget)
│   ├── Payroll Summary Table (Table widget)
│   └── Global Search (scoped to user's branch and role)
│
🏠 Dashboard (Portal Panel)
├── PortalDashboardPage
│   ├── Personal Stats Overview Widget (StatsOverviewWidget)
│   ├── My Attendance Chart (Chart.js integration)
│   ├── My Leave Balance Table (Table widget)
│   ├── My Payslips Table (Table widget)
│   └── Global Search (scoped to user's own data)

👥 Employee Management (Navigation Group)
├── EmployeeResource (getPages, getRelations, infolist)
├── DepartmentResource (Simple Resource — modal)
├── PositionResource (Simple Resource — modal)
├── BranchResource (Simple Resource — modal)
├── OnboardingTemplateResource (Simple Resource — modal)
├── EmployeeOnboardingResource
├── ContractTemplateResource (Simple Resource — modal)
├── EmployeeContractResource
├── AssetCategoryResource (Simple Resource — modal)
├── AssetResource (Simple Resource — modal)
└── EmployeeAssetRelationManager (on EmployeeResource)

🕐 Attendance & Leave (Navigation Group)
├── ShiftResource (Simple Resource — modal)
├── EmployeeScheduleResource
├── AttendanceLogResource
├── DtrEntryResource
├── DtrCorrectionRequestResource
├── HolidayResource (Simple Resource — modal)
├── LeaveTypeResource (Simple Resource — modal)
├── LeaveCreditResource
├── LeaveRequestResource
├── GeofenceApprovalResource
└── EmployeeSiteLocationsRelationManager (on EmployeeResource)

💰 Payroll (Navigation Group)
├── PayPeriodResource (Simple Resource — modal)
├── SalaryGradeResource (Simple Resource — modal)
├── EmployeeSalaryResource
├── PayrollRunResource
├── PayrollItemResource
├── DeductionTypeResource (Simple Resource — modal)
├── EmployeeStatutoryLoanResource
├── EmployeeCompanyLoanResource
└── PayrollSettingsPage

🎁 Benefits & Compensation (Navigation Group)
├── ThirteenthMonthRecordResource
├── SeparationPayRecordResource
├── FinalPayRecordResource
├── RetirementPayRecordResource
├── DeMinimisBenefitTypeResource (Simple Resource — modal)
├── EmployeeDeMinimisBenefitResource
├── LeaveCashConversionResource
├── Bir2316RecordResource
└── Bir1604cRecordResource

📋 Recruitment (Navigation Group)
├── JobRequisitionResource (Simple Resource — modal)
├── JobPostingResource
├── JobApplicantResource
├── InterviewScheduleResource
└── JobOfferResource

📈 Performance (Navigation Group)
├── PerformanceCycleResource (Simple Resource — modal)
├── PerformanceReviewResource
├── EmployeeGoalResource
└── PerformanceImprovementPlanResource

🤝 Employee Relations (Navigation Group)
├── GrievanceCaseResource
├── DisciplinaryCaseResource
└── HrTicketResource

🎓 Training (Navigation Group)
├── TrainingProgramResource (Simple Resource — modal)
├── EmployeeTrainingRecordResource
└── EmployeeCertificationResource

🔒 Privacy (Navigation Group)
├── DataPrivacyConsentResource
├── DataSubjectRequestResource
├── DataRetentionPolicyResource (Simple Resource — modal)
└── PiiAuditReportPage

📊 Reports (Navigation Group)
├── AttendanceReportPage
├── TardinessReportPage
├── AbsenceReportPage
├── GeofenceViolationReportPage
├── BenefitsReportPage
├── PayrollReportPage
├── BirReportPage
└── GlExportPage

🔑 Access & Roles (Navigation Group)
├── UserResource
├── RoleResource (via Filament Shield)
└── PermissionResource (via Filament Shield)

🔌 Integrations (Navigation Group)
├── WebhookSubscriptionResource (Simple Resource — modal)
├── WebhookDeliveryLogResource
├── PublicApiTokenResource
├── SsoConfigurationPage
├── BiometricDeviceResource (Simple Resource — modal)
├── EmployeeDeviceMappingRelationManager (on EmployeeResource)
└── BiometricUnmappedRecordResource

⚙️ Settings (Navigation Group)
├── CompanySettingsPage
├── AnnouncementResource (Simple Resource — modal)
└── SystemSettingsPage

📋 Audit (Navigation Group)
├── ActivityLogResource
└── SystemLogPage
```

### Portal Panel Resources Organization

```text
🏠 Dashboard
├── PortalDashboardPage

👤 Profile (Navigation Group)
├── MyProfileResource (SingleRecordResource — scoped to auth user)
├── MyDocumentsResource (SingleRecordResource)
└── MySettingsResource (SingleRecordResource)

🕐 Attendance (Navigation Group)
├── MyAttendanceResource (SingleRecordResource)
├── MyDtrResource (SingleRecordResource)
└── AttendanceCorrectionRequestResource

🌴 Leave (Navigation Group)
├── MyLeaveBalancesResource (SingleRecordResource)
├── MyLeaveRequestsResource
├── FileLeaveResource (Simple Resource — modal)
└── LeaveCalendarPage

💸 Payslips & Benefits (Navigation Group)
├── MyPayslipsResource (SingleRecordResource)
├── MyBenefitsResource (SingleRecordResource)
├── My13thMonthResource (SingleRecordResource)
├── MySilConversionResource (SingleRecordResource)
├── MyBirFormsResource (SingleRecordResource)
└── MyLoansResource (SingleRecordResource)

🏢 My HR (Navigation Group)
├── MyOnboardingResource (SingleRecordResource)
├── MyContractsResource (SingleRecordResource)
├── MyAssetsResource (SingleRecordResource)
├── MyClearanceResource (SingleRecordResource)
└── MyTrainingResource (SingleRecordResource)

📈 Performance (Navigation Group)
├── MyPerformanceResource (SingleRecordResource)
├── MyGoalsResource
├── MyReviewsResource (SingleRecordResource)
└── MyPipResource (SingleRecordResource)

🤝 Support (Navigation Group)
├── MyTicketsResource
├── CreateTicketResource (Simple Resource — modal)
└── MyCasesResource (SingleRecordResource — grievance status only)

🔒 Privacy (Navigation Group)
├── MyPrivacyResource (SingleRecordResource)
├── MyConsentsResource (SingleRecordResource)
└── DataSubjectRequestResource (Simple Resource — modal)
```

---

## 📁 Filament v5 Resource Directory Structure

### Standard Resource Structure
```
app/Filament/
├── Resources/
│   ├── EmployeeResource.php
│   ├── Pages/
│   │   ├── ListEmployees.php
│   │   ├── CreateEmployee.php
│   │   ├── EditEmployee.php
   │   └── ViewEmployee.php
│   ├── RelationManagers/
│   │   ├── AssignmentsRelationManager.php
│   │   ├── DocumentsRelationManager.php
   │   └── ContractsRelationManager.php
│   ├── Schemas/
│   │   ├── EmployeeForm.php
│   │   └── EmployeeInfolist.php
│   └── Tables/
│       └── EmployeeTable.php
```

### Resource File Organization Patterns

#### Simple Resource (Inline Form/Infolist)
```php
// EmployeeResource.php
namespace App\Filament\Resources;

use App\Models\Employee;
use Filament\Resources\Resource;
use Filament\Resources\Pages\PageRegistration;
use Filament\Schemas\Schema;
use Filament\Tables\Table;

class EmployeeResource extends Resource
{
    protected static ?string $model = Employee::class;

    public static function form(Schema $schema): Schema
    {
        return $schema->schema([
            // Inline form definition
        ]);
    }

    public static function infolist(Schema $schema): Schema
    {
        return $schema->schema([
            // Inline infolist definition
        ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                // Table columns
            ]);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListEmployees::route('/'),
            'create' => Pages\CreateEmployee::route('/create'),
            'edit' => Pages\EditEmployee::route('/{record}/edit'),
            'view' => Pages\ViewEmployee::route('/{record}'),
        ];
    }
}
```

#### Complex Resource (Extracted Schemas/Tables)
```php
// EmployeeResource.php
namespace App\Filament\Resources;

use App\Filament\Resources\EmployeeResource\Pages;
use App\Filament\Schemas\EmployeeFormSchema;
use App\Filament\Tables\EmployeeTable;
use App\Models\Employee;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables\Table;

class EmployeeResource extends Resource
{
    protected static ?string $model = Employee::class;

    public static function form(Schema $schema): Schema
    {
        return EmployeeFormSchema::make($schema);
    }

    public static function table(Table $table): Table
    {
        return EmployeeTable::make($table);
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ListEmployees::route('/'),
            'create' => Pages\CreateEmployee::route('/create'),
            'edit' => Pages\EditEmployee::route('/{record}/edit'),
        ];
    }
}
```

#### Form Schema with Enum Integration
```php
// Schemas/EmployeeFormSchema.php
namespace App\Filament\Schemas;

use App\Enums\CivilStatus;
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\Gender;
use Filament\Forms;
use Filament\Forms\Form;
use Filament\Schemas\Schema;

class EmployeeFormSchema
{
    public static function make(Schema $schema): Schema
    {
        return $schema->schema([
            Forms\Components\Tabs::make('employee_tabs')
                ->tabs([
                    Forms\Components\Tabs\Tab::make('Personal Information')
                        ->schema([
                            Forms\Components\Section::make('Basic Information')
                                ->schema([
                                    Forms\Components\TextInput::make('first_name')
                                        ->required()
                                        ->maxLength(255),
                                    Forms\Components\TextInput::make('last_name')
                                        ->required()
                                        ->maxLength(255),
                                    Forms\Components\TextInput::make('email')
                                        ->email()
                                        ->required()
                                        ->unique(ignoreRecord: true),
                                ])
                                ->columns(2),
                            
                            Forms\Components\Section::make('Contact Details')
                                ->schema([
                                    Forms\Components\TextInput::make('mobile_no')
                                        ->tel()
                                        ->maxLength(20),
                                    Forms\Components\Textarea::make('address')
                                        ->rows(3),
                                ])
                                ->columns(2),
                            
                            Forms\Components\Section::make('Personal Details')
                                ->schema([
                                    Forms\Components\Select::make('gender')
                                        ->options(Gender::class)
                                        ->required()
                                        ->enum(Gender::class)
                                        ->helperText(fn (Gender $gender): string => $gender->getDescription()),
                                    Forms\Components\Select::make('civil_status')
                                        ->options(CivilStatus::class)
                                        ->required()
                                        ->enum(CivilStatus::class)
                                        ->helperText(fn (CivilStatus $status): string => $status->getDescription()),
                                ])
                                ->columns(2),
                        ]),
                    
                    Forms\Components\Tabs\Tab::make('Employment')
                        ->schema([
                            Forms\Components\Section::make('Position Details')
                                ->schema([
                                    Forms\Components\Select::make('department_id')
                                        ->relationship('department', 'name')
                                        ->searchable()
                                        ->required(),
                                    Forms\Components\Select::make('position_id')
                                        ->relationship('position', 'title')
                                        ->searchable()
                                        ->required(),
                                    Forms\Components\Select::make('employment_type')
                                        ->options(EmploymentType::class)
                                        ->required()
                                        ->enum(EmploymentType::class)
                                        ->helperText(fn (EmploymentType $type): string => $type->getDescription()),
                                    Forms\Components\Select::make('employment_status')
                                        ->options(EmploymentStatus::class)
                                        ->required()
                                        ->enum(EmploymentStatus::class)
                                        ->helperText(fn (EmploymentStatus $status): string => $status->getDescription()),
                                ])
                                ->columns(2),
                            
                            Forms\Components\Section::make('Dates')
                                ->schema([
                                    Forms\Components\DatePicker::make('date_hired')
                                        ->required(),
                                    Forms\Components\DatePicker::make('date_regularized'),
                                    Forms\Components\DatePicker::make('date_separated'),
                                ])
                                ->columns(3),
                        ]),
                    
                    Forms\Components\Tabs\Tab::make('Government IDs')
                        ->schema([
                            Forms\Components\Section::make('Government IDs')
                                ->description('These are encrypted at rest')
                                ->schema([
                                    Forms\Components\TextInput::make('sss_no')
                                        ->mask('999-99-9999-9')
                                        ->label('SSS Number'),
                                    Forms\Components\TextInput::make('philhealth_no')
                                        ->mask('9999-999999-9')
                                        ->label('PhilHealth Number'),
                                    Forms\Components\TextInput::make('pagibig_no')
                                        ->mask('9999-999999-9')
                                        ->label('Pag-IBIG Number'),
                                    Forms\Components\TextInput::make('tin_no')
                                        ->mask('999-999-999-999')
                                        ->label('TIN Number'),
                                ])
                                ->columns(2),
                        ]),
                ])
                ->persistTabs()
                ->columnSpanFull(),
        ]);
    }
}
```

### Form Schema Organization

#### Complex Form with Tabs and Sections
```php
// Schemas/EmployeeFormSchema.php
namespace App\Filament\Schemas;

use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use Filament\Forms;
use Filament\Forms\Form;
use Filament\Schemas\Schema;

class EmployeeFormSchema
{
    public static function make(Schema $schema): Schema
    {
        return $schema->schema([
            Forms\Components\Tabs::make('employee_tabs')
                ->tabs([
                    Forms\Components\Tabs\Tab::make('Personal Information')
                        ->schema([
                            Forms\Components\Section::make('Basic Information')
                                ->schema([
                                    Forms\Components\TextInput::make('first_name')
                                        ->required()
                                        ->maxLength(255),
                                    Forms\Components\TextInput::make('last_name')
                                        ->required()
                                        ->maxLength(255),
                                    Forms\Components\TextInput::make('email')
                                        ->email()
                                        ->required()
                                        ->unique(ignoreRecord: true),
                                ])
                                ->columns(2),
                            
                            Forms\Components\Section::make('Contact Details')
                                ->schema([
                                    Forms\Components\TextInput::make('mobile_no')
                                        ->tel()
                                        ->maxLength(20),
                                    Forms\Components\Textarea::make('address')
                                        ->rows(3),
                                ]),
                        ]),
                    
                    Forms\Components\Tabs\Tab::make('Employment')
                        ->schema([
                            Forms\Components\Section::make('Position Details')
                                ->schema([
                                    Forms\Components\Select::make('department_id')
                                        ->relationship('department', 'name')
                                        ->searchable()
                                        ->required(),
                                    Forms\Components\Select::make('position_id')
                                        ->relationship('position', 'title')
                                        ->searchable()
                                        ->required(),
                                    Forms\Components\Select::make('employment_type')
                                        ->options(EmploymentType::class)
                                        ->required()
                                        ->enum(EmploymentType::class),
                                    Forms\Components\Select::make('employment_status')
                                        ->options(EmploymentStatus::class)
                                        ->required()
                                        ->enum(EmploymentStatus::class),
                                ])
                                ->columns(2),
                            
                            Forms\Components\Section::make('Dates')
                                ->schema([
                                    Forms\Components\DatePicker::make('date_hired')
                                        ->required(),
                                    Forms\Components\DatePicker::make('date_regularized'),
                                    Forms\Components\DatePicker::make('date_separated'),
                                ])
                                ->columns(3),
                        ]),
                    
                    Forms\Components\Tabs\Tab::make('Government IDs')
                        ->schema([
                            Forms\Components\Section::make('Government IDs')
                                ->description('These are encrypted at rest')
                                ->schema([
                                    Forms\Components\TextInput::make('sss_no')
                                        ->mask('999-99-9999-9')
                                        ->label('SSS Number'),
                                    Forms\Components\TextInput::make('philhealth_no')
                                        ->mask('9999-999999-9')
                                        ->label('PhilHealth Number'),
                                    Forms\Components\TextInput::make('pagibig_no')
                                        ->mask('9999-999999-9')
                                        ->label('Pag-IBIG Number'),
                                    Forms\Components\TextInput::make('tin_no')
                                        ->mask('999-999-999-999')
                                        ->label('TIN Number'),
                                ])
                                ->columns(2),
                        ]),
                ])
                ->persistTabs()
                ->columnSpanFull(),
        ]);
    }
}
```

### Infolist Schema Organization

#### Read-Only Record View
```php
// Schemas/EmployeeInfolistSchema.php
namespace App\Filament\Schemas;

use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use Filament\Infolists;
use Filament\Infolists\Infolist;
use Filament\Schemas\Schema;

class EmployeeInfolistSchema
{
    public static function make(Schema $schema): Schema
    {
        return $schema->schema([
            Infolists\Components\Section::make('Personal Information')
                ->schema([
                    Infolists\Components\TextEntry::make('full_name')
                        ->label('Name'),
                    Infolists\Components\TextEntry::make('email'),
                    Infolists\Components\TextEntry::make('mobile_no'),
                    Infolists\Components\TextEntry::make('address')
                        ->columnSpanFull(),
                ])
                ->columns(2),
            
            Infolists\Components\Section::make('Employment Details')
                ->schema([
                    Infolists\Components\TextEntry::make('department.name')
                        ->label('Department'),
                    Infolists\Components\TextEntry::make('position.title')
                        ->label('Position'),
                    // Filament v5 automatic enum badge (uses HasLabel and HasColor from enum)
                    Infolists\Components\TextEntry::make('employment_type')
                        ->badge(),
                    // Filament v5 automatic enum badge (uses HasLabel and HasColor from enum)
                    Infolists\Components\TextEntry::make('employment_status')
                        ->badge(),
                    Infolists\Components\TextEntry::make('date_hired')
                        ->date(),
                    Infolists\Components\TextEntry::make('date_regularized')
                        ->date(),
                ])
                ->columns(3),
            
            Infolists\Components\Section::make('Government IDs')
                ->schema([
                    Infolists\Components\TextEntry::make('sss_no')
                        ->label('SSS Number')
                        ->mask('999-99-9999-9'),
                    Infolists\Components\TextEntry::make('philhealth_no')
                        ->label('PhilHealth Number')
                        ->mask('9999-999999-9'),
                    Infolists\Components\TextEntry::make('pagibig_no')
                        ->label('Pag-IBIG Number')
                        ->mask('9999-999999-9'),
                    Infolists\Components\TextEntry::make('tin_no')
                        ->label('TIN Number')
                        ->mask('999-999-999-999'),
                ])
                ->columns(2),
        ]);
    }
}
```

### Table Organization

#### Advanced Table with Filters and Actions
```php
// Tables/EmployeeTable.php
namespace App\Filament\Tables;

use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use Filament\Tables;
use Filament\Tables\Table;

class EmployeeTable
{
    public static function make(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('employee_no')
                    ->searchable()
                    ->sortable(),
                Tables\Columns\TextColumn::make('full_name')
                    ->searchable(['first_name', 'last_name'])
                    ->sortable()
                    ->description(fn ($record): string => $record->email),
                Tables\Columns\TextColumn::make('department.name')
                    ->searchable()
                    ->sortable(),
                Tables\Columns\TextColumn::make('position.title')
                    ->searchable()
                    ->sortable(),
                // Filament v5 automatic enum badge (uses HasLabel and HasColor from enum)
                Tables\Columns\TextColumn::make('employment_type')
                    ->badge()
                    ->sortable(),
                // Filament v5 automatic enum badge (uses HasLabel and HasColor from enum)
                Tables\Columns\TextColumn::make('employment_status')
                    ->badge()
                    ->sortable(),
                Tables\Columns\IconColumn::make('is_active')
                    ->boolean(),
                Tables\Columns\TextColumn::make('created_at')
                    ->dateTime()
                    ->sortable()
                    ->toggleable(isToggledHiddenByDefault: true),
            ])
            ->filters([
                Tables\Filters\SelectFilter::make('department')
                    ->relationship('department', 'name')
                    ->searchable()
                    ->preload(),
                Tables\Filters\SelectFilter::make('position')
                    ->relationship('position', 'title')
                    ->searchable()
                    ->preload(),
                // Filament v5 automatic enum filter (uses HasLabel from enum)
                Tables\Filters\SelectFilter::make('employment_type')
                    ->options(EmploymentType::class),
                // Filament v5 automatic enum filter (uses HasLabel from enum)
                Tables\Filters\SelectFilter::make('employment_status')
                    ->options(EmploymentStatus::class),
                Tables\Filters\Filter::make('active')
                    ->query(fn ($query) => $query->where('is_active', true)),
                Tables\Filters\Filter::make('created_at')
                    ->form([
                        Forms\Components\DatePicker::make('from'),
                        Forms\Components\DatePicker::make('until'),
                    ])
                    ->query(function ($query, array $data): Builder {
                        return $query
                            ->when($data['from'], fn ($query, $date) => $query->whereDate('created_at', '>=', $date))
                            ->when($data['until'], fn ($query, $date) => $query->whereDate('created_at', '<=', $date));
                    }),
            ])
            ->actions([
                Tables\Actions\ViewAction::make(),
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\DeleteBulkAction::make(),
                    Tables\Actions\BulkAction::make('activate')
                        ->action(fn ($records) => $records->each->update(['is_active' => true]))
                        ->requiresConfirmation()
                        ->color('success'),
                    Tables\Actions\BulkAction::make('deactivate')
                        ->action(fn ($records) => $records->each->update(['is_active' => false]))
                        ->requiresConfirmation()
                        ->color('danger'),
                ]),
            ])
            ->headerActions([
                Tables\Actions\CreateAction::make(),
                Tables\Actions\ImportAction::make()
                    ->importer(EmployeeImporter::class),
            ])
            ->defaultSort('created_at', 'desc')
            ->paginated([25, 50, 100]);
    }
}
```

### Relation Manager Organization

#### Many-to-Many Relation Manager
```php
// RelationManagers/AssignmentsRelationManager.php
namespace App\Filament\Resources\EmployeeResource\RelationManagers;

use App\Models\EmployeeAssignment;
use Filament\Forms;
use Filament\Forms\Form;
use Filament\Resources\RelationManagers\RelationManager;
use Filament\Tables;
use Filament\Tables\Table;

class AssignmentsRelationManager extends RelationManager
{
    protected static string $relationship = 'assignments';

    protected static ?string $title = 'Work Assignments';

    public function form(Form $form): Form
    {
        return $form
            ->schema([
                Forms\Components\Select::make('branch_id')
                    ->relationship('branch', 'name')
                    ->searchable()
                    ->required(),
                Forms\Components\Select::make('department_id')
                    ->relationship('department', 'name')
                    ->searchable()
                    ->required(),
                Forms\Components\Select::make('position_id')
                    ->relationship('position', 'title')
                    ->searchable()
                    ->required(),
                Forms\Components\Select::make('shift_id')
                    ->relationship('shift', 'name')
                    ->searchable(),
                Forms\Components\DatePicker::make('effective_date')
                    ->required(),
                Forms\Components\DatePicker::make('end_date'),
                Forms\Components\Toggle::make('is_primary')
                    ->default(false),
            ]);
    }

    public function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('branch.name'),
                Tables\Columns\TextColumn::make('department.name'),
                Tables\Columns\TextColumn::make('position.title'),
                Tables\Columns\TextColumn::make('shift.name'),
                Tables\Columns\TextColumn::make('effective_date')
                    ->date(),
                Tables\Columns\TextColumn::make('end_date')
                    ->date()
                    ->default('-'),
                Tables\Columns\IconColumn::make('is_primary')
                    ->boolean(),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make(),
            ])
            ->defaultSort('effective_date', 'desc');
    }
}
```

### Page Customization

#### Custom Page Actions and Headers
```php
// Pages/CreateEmployee.php
namespace App\Filament\Resources\EmployeeResource\Pages;

use App\Filament\Resources\EmployeeResource;
use Filament\Actions;
use Filament\Resources\Pages\CreateRecord;

class CreateEmployee extends CreateRecord
{
    protected static string $resource = EmployeeResource::class;

    protected function getHeaderActions(): array
    {
        return [
            Actions\Action::make('bulk_import')
                ->label('Bulk Import')
                ->icon('heroicon-o-document-arrow-up')
                ->url(route('filament.admin.resources.employees.bulk-import'))
                ->color('info'),
            ...parent::getHeaderActions(),
        ];
    }

    protected function getRedirectUrl(): string
    {
        return $this->getResource()::getUrl('view', ['record' => $this->record]);
    }
}
```

#### Custom Edit Page with Additional Actions
```php
// Pages/EditEmployee.php
namespace App\Filament\Resources\EmployeeResource\Pages;

use App\Filament\Resources\EmployeeResource;
use Filament\Actions;
use Filament\Resources\Pages\EditRecord;

class EditEmployee extends EditRecord
{
    protected static string $resource = EmployeeResource::class;

    protected function getHeaderActions(): array
    {
        return [
            Actions\Action::make('deactivate')
                ->label('Deactivate Employee')
                ->icon('heroicon-o-user-minus')
                ->color('danger')
                ->requiresConfirmation()
                ->action(function () {
                    $this->record->update([
                        'employment_status' => EmploymentStatus::INACTIVE,
                        'date_separated' => now(),
                    ]);
                })
                ->visible(fn (): bool => $this->record->employment_status === EmploymentStatus::ACTIVE),
            Actions\Action::make('send_onboarding')
                ->label('Send Onboarding')
                ->icon('heroicon-o-envelope')
                ->action(function () {
                    // Send onboarding notification
                })
                ->visible(fn (): bool => $this->record->onboarding?->progress_percent < 100),
            Actions\DeleteAction::make(),
            ...parent::getHeaderActions(),
        ];
    }
}
```

### Resource Configuration Best Practices

#### Navigation and Grouping
```php
class EmployeeResource extends Resource
{
    protected static ?string $model = Employee::class;

    protected static ?string $navigationIcon = 'heroicon-o-users';

    protected static ?string $navigationGroup = 'Employee Management';

    protected static ?int $navigationSort = 1;

    protected static ?string $recordTitleAttribute = 'full_name';

    protected static int $globalSearchResultLimit = 5;

    public static function getGloballySearchableAttributes(): array
    {
        return ['employee_no', 'first_name', 'last_name', 'email'];
    }

    public static function getNavigationBadge(): ?string
    {
        return static::getModel()::count();
    }

    public static function getNavigationBadgeColor(): ?string
    {
        return 'primary';
    }
}
```

---

## 🔧 Pipeline/Workflow Architecture

**Technology Stack:** Laravel Native (Queues, Events, Listeners, Jobs)
**Implementation Strategy:** Hybrid Approach - Basic queues now, advanced workflows later
**Scope:** Full Scope - HR workflows + System automation
**Complexity:** Simple Linear (sequential with basic branching)

### Pipeline Architecture

**Phase 1-6:** Basic Laravel Queue Jobs
- Simple job chaining for straightforward workflows
- Event-driven architecture for decoupled operations
- Basic retry mechanisms and error handling

**Phase 7-10:** Advanced Workflow Features
- State machine for complex workflow tracking
- Parallel job execution where needed
- Workflow persistence and recovery
- Advanced branching and conditional logic

### Core Pipeline Components

#### 1. Queue System Foundation
```php
// config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),
    'connections' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => true,
        ],
    ],
];
```

#### 2. Event System
```php
// Events/EmployeeEvents.php
namespace App\Events;

use App\Models\Employee;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class EmployeeCreated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Employee $employee
    ) {}
}

class EmployeeDeactivated
{
    use Dispatchable, SerializesModels;

    public function __construct(
        public Employee $employee,
        public string $reason
    ) {}
}
```

#### 3. Event Listeners
```php
// Listeners/EmployeeListeners.php
namespace App\Listeners;

use App\Events\EmployeeCreated;
use App\Events\EmployeeDeactivated;
use App\Services\OnboardingService;
use App\Services\ClearanceService;
use Illuminate\Contracts\Queue\ShouldQueue;

class AssignOnboardingChecklist implements ShouldQueue
{
    public function __construct(
        protected OnboardingService $onboardingService
    ) {}

    public function handle(EmployeeCreated $event): void
    {
        $this->onboardingService->assignChecklist($event->employee);
    }
}

class InitiateClearanceProcess implements ShouldQueue
{
    public function __construct(
        protected ClearanceService $clearanceService
    ) {}

    public function handle(EmployeeDeactivated $event): void
    {
        $this->clearanceService->initiate($event->employee, $event->reason);
    }
}
```

#### 4. Job Classes
```php
// Jobs/Payroll/ProcessPayrollJob.php
namespace App\Jobs\Payroll;

use App\Models\PayrollRun;
use App\Services\PayrollComputationService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessPayrollJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries = 3;
    public int $timeout = 1200; // 20 minutes

    public function __construct(
        public PayrollRun $payrollRun,
        public int $employeeId
    ) {}

    public function handle(PayrollComputationService $service): void
    {
        $service->processEmployee($this->payrollRun, $this->employeeId);
    }

    public function failed(\Throwable $exception): void
    {
        // Handle failure, log error, mark payroll item as failed
        $this->payrollRun->payrollItems()
            ->where('employee_id', $this->employeeId)
            ->update([
                'status' => 'failed',
                'error_details' => [
                    'message' => $exception->getMessage(),
                    'trace' => $exception->getTraceAsString(),
                ]
            ]);
    }
}
```

#### 5. Pipeline Chains
```php
// Services/Pipeline/PayrollPipeline.php
namespace App\Services\Pipeline;

use App\Jobs\Payroll\ProcessPayrollJob;
use App\Jobs\Payroll\GeneratePayslipJob;
use App\Jobs\Payroll\SendNotificationJob;
use App\Models\PayrollRun;
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;
use Throwable;

class PayrollPipeline
{
    public function execute(PayrollRun $payrollRun): Batch
    {
        $jobs = $payrollRun->eligibleEmployees->map(function ($employee) use ($payrollRun) {
            return [
                new ProcessPayrollJob($payrollRun, $employee->id),
                new GeneratePayslipJob($payrollRun, $employee->id),
                new SendNotificationJob($employee->id, 'payslip_available'),
            ];
        }->flatten();

        return Bus::batch($jobs->toArray())
            ->then(function (Batch $batch) use ($payrollRun) {
                $payrollRun->update([
                    'status' => 'processed',
                    'processed_count' => $batch->totalJobs,
                ]);
            })
            ->catch(function (Batch $batch, Throwable $e) use ($payrollRun) {
                $payrollRun->update([
                    'status' => 'partial_failure',
                    'failed_count' => $batch->failedJobs,
                ]);
            })
            ->allowFailures()
            ->onQueue('payroll')
            ->dispatch();
    }
}
```

### Workflow State Tracking

#### Pipeline State Model
```php
// Models/WorkflowExecution.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class WorkflowExecution extends Model
{
    protected $fillable = [
        'workflow_type',
        'status',
        'payload',
        'current_step',
        'total_steps',
        'completed_steps',
        'error_details',
        'started_at',
        'completed_at',
    ];

    protected $casts = [
        'payload' => 'array',
        'error_details' => 'array',
        'started_at' => 'datetime',
        'completed_at' => 'datetime',
    ];

    public function scopeActive($query)
    {
        return $query->where('status', 'running');
    }

    public function scopeCompleted($query)
    {
        return $query->where('status', 'completed');
    }

    public function scopeFailed($query)
    {
        return $query->where('status', 'failed');
    }
}
```

### HR-Specific Workflow Implementations

#### Onboarding Workflow
```php
// Services/Workflow/OnboardingWorkflow.php
namespace App\Services\Workflow;

use App\Jobs\Onboarding\SendWelcomeEmailJob;
use App\Jobs\Onboarding\SetupAccountsJob;
use App\Jobs\Onboarding\ScheduleOrientationJob;
use App\Models\Employee;
use Illuminate\Support\Facades\Bus;

class OnboardingWorkflow
{
    public function execute(Employee $employee): void
    {
        Bus::chain([
            new SendWelcomeEmailJob($employee),
            new SetupAccountsJob($employee),
            new ScheduleOrientationJob($employee),
        ])->dispatch();
    }
}
```

#### Clearance Workflow
```php
// Services/Workflow/ClearanceWorkflow.php
namespace App\Services\Workflow;

use App\Jobs\Clearance\VerifyAssetReturnJob;
use App\Jobs\Clearance\ProcessBenefitsJob;
use App\Jobs\Clearance\ReleaseFinalPayJob;
use App\Models\Employee;
use Illuminate\Support\Facades\Bus;

class ClearanceWorkflow
{
    public function execute(Employee $employee): void
    {
        Bus::chain([
            new VerifyAssetReturnJob($employee),
            new ProcessBenefitsJob($employee),
            new ReleaseFinalPayJob($employee),
        ])->dispatch();
    }
}
```

#### Leave Approval Workflow
```php
// Services/Workflow/LeaveApprovalWorkflow.php
namespace App\Services\Workflow;

use App\Jobs\Leave\UpdateBalanceJob;
use App\Jobs\Leave\SendApprovalNotificationJob;
use App\Jobs\Leave\UpdateCalendarJob;
use App\Models\LeaveRequest;
use Illuminate\Support\Facades\Bus;

class LeaveApprovalWorkflow
{
    public function execute(LeaveRequest $request): void
    {
        Bus::chain([
            new UpdateBalanceJob($request),
            new SendApprovalNotificationJob($request),
            new UpdateCalendarJob($request),
        ])->dispatch();
    }
}
```

### System Automation Pipelines

#### Biometric Sync Pipeline
```php
// Services/Workflow/BiometricSyncPipeline.php
namespace App\Services\Workflow;

use App\Jobs\Biometric\ProcessAttendanceJob;
use App\Jobs\Biometric\ValidateEmployeeMappingJob;
use Illuminate\Support\Facades\Bus;

class BiometricSyncPipeline
{
    public function execute(array $attendanceData): void
    {
        Bus::batch([
            new ValidateEmployeeMappingJob($attendanceData),
            new ProcessAttendanceJob($attendanceData),
        ])->allowFailures()->dispatch();
    }
}
```

#### Webhook Delivery Pipeline
```php
// Services/Workflow/WebhookPipeline.php
namespace App\Services\Workflow;

use App\Jobs\Webhook\DeliverWebhookJob;
use App\Models\WebhookSubscription;
use Illuminate\Support\Facades\Bus;

class WebhookPipeline
{
    public function execute(string $eventType, array $payload): void
    {
        $jobs = WebhookSubscription::where('is_active', true)
            ->whereJsonContains('events', $eventType)
            ->get()
            ->map(function ($subscription) use ($eventType, $payload) {
                return new DeliverWebhookJob($subscription, $eventType, $payload);
            });

        Bus::batch($jobs->toArray())
            ->allowFailures()
            ->dispatch();
    }
}
```

### Pipeline Monitoring and Management

#### Horizon Configuration
```php
// config/horizon.php
return [
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue' => ['default', 'payroll', 'attendance', 'notifications'],
                'balance' => 'auto',
                'maxProcesses' => 10,
                'maxTries' => 3,
                'timeout' => 1200,
            ],
        ],
    ],
];
```

#### Pipeline Dashboard Widget
```php
// Filament/Widgets/PipelineStatusWidget.php
namespace App\Filament\Widgets;

use App\Models\WorkflowExecution;
use Filament\Widgets\StatsOverviewWidget;
use Illuminate\Support\Facades\DB;

class PipelineStatusWidget extends StatsOverviewWidget
{
    protected static ?string $title = 'Pipeline Status';

    protected function getStats(): array
    {
        return [
            Stat::make('Running', WorkflowExecution::active()->count())
                ->description('Currently executing')
                ->descriptionIcon('heroicon-o-arrow-path')
                ->color('info'),
            Stat::make('Completed Today', WorkflowExecution::completed()->whereDate('completed_at', today())->count())
                ->description('Successfully completed')
                ->descriptionIcon('heroicon-o-check-circle')
                ->color('success'),
            Stat::make('Failed Today', WorkflowExecution::failed()->whereDate('completed_at', today())->count())
                ->description('Failed executions')
                ->descriptionIcon('heroicon-o-x-circle')
                ->color('danger'),
            Stat::make('Avg Duration', $this->getAverageDuration())
                ->description('Average completion time')
                ->descriptionIcon('heroicon-o-clock')
                ->color('warning'),
        ];
    }

    private function getAverageDuration(): string
    {
        $avgSeconds = WorkflowExecution::completed()
            ->where('completed_at', '>=', now()->subDay())
            ->selectRaw('AVG(TIMESTAMPDIFF(SECOND, started_at, completed_at)) as avg_duration')
            ->value('avg_duration');

        return gmdate('H:i:s', $avgSeconds);
    }
}
```

#### Admin Dashboard Widgets
```php
// Filament/Widgets/AdminDashboardWidgets.php
namespace App\Filament\Widgets;

use App\Models\Employee;
use App\Models\LeaveRequest;
use App\Models\AttendanceRecord;
use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;
use Filament\Widgets\ChartWidget;
use Illuminate\Support\Facades\DB;

class AdminDashboardStats extends StatsOverviewWidget
{
    protected static ?string $title = 'HR Overview';

    protected function getStats(): array
    {
        return [
            Stat::make('Total Employees', Employee::count())
                ->description('Across all branches')
                ->descriptionIcon('heroicon-o-users')
                ->color('primary'),
            Stat::make('Active Employees', Employee::where('is_active', true)->count())
                ->description('Currently employed')
                ->descriptionIcon('heroicon-o-user-check')
                ->color('success'),
            Stat::make('Pending Leave Requests', LeaveRequest::where('status', 'pending')->count())
                ->description('Awaiting approval')
                ->descriptionIcon('heroicon-o-clock')
                ->color('warning'),
            Stat::make('Attendance Today', AttendanceRecord::whereDate('date', today())->count())
                ->description('Today\'s records')
                ->descriptionIcon('heroicon-o-calendar')
                ->color('info'),
        ];
    }
}

class AttendanceChartWidget extends ChartWidget
{
    protected static ?string $title = 'Attendance Trends';
    
    protected function getData(): array
    {
        return [
            'datasets' => [
                [
                    'label' => 'Present',
                    'data' => [12, 19, 3, 5, 2, 3],
                    'backgroundColor' => 'rgba(16, 185, 129, 0.2)',
                    'borderColor' => 'rgb(16, 185, 129)',
                ],
                [
                    'label' => 'Absent',
                    'data' => [2, 3, 20, 5, 1, 4],
                    'backgroundColor' => 'rgba(239, 68, 68, 0.2)',
                    'borderColor' => 'rgb(239, 68, 68)',
                ],
            ],
            'labels' => ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'],
        ];
    }
}

class LeaveSummaryTableWidget extends ChartWidget
{
    protected static ?string $title = 'Leave Summary by Department';
    
    protected function getData(): array
    {
        return [
            'datasets' => [
                [
                    'label' => 'Approved',
                    'data' => [15, 8, 12, 5],
                    'backgroundColor' => 'rgba(16, 185, 129, 0.8)',
                ],
                [
                    'label' => 'Pending',
                    'data' => [3, 2, 4, 1],
                    'backgroundColor' => 'rgba(234, 179, 8, 0.8)',
                ],
            ],
            'labels' => ['HR', 'Engineering', 'Sales', 'Finance'],
        ];
    }
}
```

#### Dashboard Branch Filtering
```php
// Filament/Pages/AdminDashboardPage.php
namespace App\Filament\Pages;

use App\Filament\Widgets\AdminDashboardStats;
use App\Filament\Widgets\AttendanceChartWidget;
use App\Filament\Widgets\LeaveSummaryTableWidget;
use Filament\Pages\DashboardPage;
use Filament\Forms\Components\Select;
use Filament\Support\Facades\FilamentView;

class AdminDashboardPage extends DashboardPage
{
    protected static string $view = 'filament.pages.dashboard';
    
    protected function getHeaderWidgets(): array
    {
        return [
            AdminDashboardStats::class,
        ];
    }
    
    protected function getFooterWidgets(): array
    {
        return [
            AttendanceChartWidget::class,
            LeaveSummaryTableWidget::class,
        ];
    }
    
    public function getHeaderActions(): array
    {
        return [
            \Filament\Actions\Action::make('refresh')
                ->label('Refresh')
                ->icon('heroicon-o-arrow-path')
                ->action(fn () => $this->redirect()),
        ];
    }
}
```

#### Portal Dashboard Widgets
```php
// Filament/Widgets/PortalDashboardWidgets.php
namespace App\Filament\Widgets;

use App\Models\LeaveRequest;
use App\Models\AttendanceRecord;
use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;
use Filament\Widgets\ChartWidget;

class PortalDashboardStats extends StatsOverviewWidget
{
    protected static ?string $title = 'My Overview';

    protected function getStats(): array
    {
        $user = auth()->user();
        
        return [
            Stat::make('Leave Balance', $user->employee->leaveBalances->sum('remaining_days'))
                ->description('Total available')
                ->descriptionIcon('heroicon-o-calendar')
                ->color('success'),
            Stat::make('Pending Requests', LeaveRequest::where('employee_id', $user->employee->id)
                ->where('status', 'pending')->count())
                ->description('Awaiting approval')
                ->descriptionIcon('heroicon-o-clock')
                ->color('warning'),
            Stat::make('This Month Attendance', AttendanceRecord::where('employee_id', $user->employee->id)
                ->whereMonth('date', now()->month)->count())
                ->description('Days present')
                ->descriptionIcon('heroicon-o-check-circle')
                ->color('info'),
        ];
    }
}

class MyAttendanceChartWidget extends ChartWidget
{
    protected static ?string $title = 'My Attendance This Month';
    
    protected function getData(): array
    {
        return [
            'datasets' => [
                [
                    'label' => 'Hours Worked',
                    'data' => [8, 8, 8, 8, 8, 0, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8, 8],
                    'backgroundColor' => 'rgba(16, 185, 129, 0.2)',
                    'borderColor' => 'rgb(16, 185, 129)',
                ],
            ],
            'labels' => range(1, date('t')),
        ];
    }
}
```

#### Global Search with Branch and Role Scoping
```php
// Filament/Resources/EmployeeResource.php
namespace App\Filament\Resources;

use App\Models\Employee;
use Filament\Resources\Resource;
use Filament\Tables\Table;
use Filament\Tables\Columns\TextColumn;

class EmployeeResource extends Resource
{
    protected static ?string $model = Employee::class;
    
    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                TextColumn::make('employee_no')->searchable(),
                TextColumn::make('first_name')->searchable(['first_name', 'last_name']),
                TextColumn::make('last_name')->searchable(),
                TextColumn::make('email')->searchable(),
            ])
            ->query(function ($query) {
                // Scope to user's branch and role
                $user = auth()->user();
                
                if ($user->role === 'manager') {
                    // Managers see their department only
                    $query->where('department_id', $user->employee->department_id);
                } elseif ($user->role === 'hr_staff') {
                    // HR staff see all branches they have access to
                    $query->whereIn('branch_id', $user->accessibleBranches());
                }
                // Admins see all branches
                
                return $query;
            });
    }
    
    public static function getGloballySearchableAttributes(): array
    {
        return ['employee_no', 'first_name', 'last_name', 'email'];
    }
    
    public static function getGlobalSearchResultLimit(): int
    {
        return 10;
    }
}
```

### Pipeline Error Handling and Recovery

#### Retry Strategies
```php
// Jobs/Payroll/ProcessPayrollJob.php
public int $tries = 3;
public int $backoff = [30, 60, 120]; // seconds

public function retryUntil(): DateTime
{
    return now()->addHours(2);
}
```

#### Dead Letter Queue
```php
// Jobs/FailedJobMonitor.php
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Support\Facades\Log;

class FailedJobMonitor implements ShouldQueue
{
    use Queueable;

    public function __construct(
        public string $jobId,
        public string $queue
    ) {}

    public function handle(): void
    {
        $failedJob = \DB::table('failed_jobs')
            ->where('id', $this->jobId)
            ->first();

        if ($failedJob) {
            Log::channel('pipeline-errors')->error('Pipeline job failed', [
                'job_id' => $this->jobId,
                'queue' => $this->queue,
                'exception' => $failedJob->exception,
                'failed_at' => $failedJob->failed_at,
            ]);

            // Send alert to admin if critical failure
            if ($this->queue === 'payroll') {
                // Send notification
            }
        }
    }
}
```

### Pipeline Database Schema

#### Workflow Executions Table
```php
// Database/migrations/xxxx_create_workflow_executions_table.php
Schema::create('workflow_executions', function (Blueprint $table) {
    $table->id();
    $table->string('workflow_type'); // onboarding, clearance, payroll, leave_approval
    $table->string('status')->default('pending'); // pending, running, completed, failed
    $table->json('payload'); // Input data for the workflow
    $table->string('current_step')->nullable();
    $table->integer('total_steps')->default(0);
    $table->integer('completed_steps')->default(0);
    $table->json('error_details')->nullable();
    $table->timestamp('started_at')->nullable();
    $table->timestamp('completed_at')->nullable();
    $table->timestamps();
    
    $table->index('workflow_type');
    $table->index('status');
    $table->index('started_at');
});
```

### Implementation Timeline

**Phase 1-6:** Basic Queue Infrastructure
- Redis queue configuration
- Basic job classes for common operations
- Event system foundation
- Simple job chains

**Phase 7-10:** Advanced Workflow Features
- Workflow state tracking
- Pipeline monitoring dashboard
- Advanced error handling
- Complex workflow orchestration

---

## 🗄️ Database Schemas & Models

### Phase 1-2: Foundation & Company Structure

#### Company Settings Model
```php
// Models/CompanySettings.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class CompanySettings extends Model
{
    protected $table = 'company_settings';
    
    protected $fillable = [
        'name',
        'logo',
        'address',
        'city',
        'country',
        'timezone',
        'locale',
        'currency',
        'tin_no',
    ];

    protected $casts = [
        'timezone' => 'string',
        'locale' => 'string',
        'currency' => 'string',
    ];

    public static function current(): self
    {
        return static::firstOrFail();
    }
}
```

#### Branch Model
```php
// Models/Branch.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Branch extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'code',
        'address',
        'city',
        'timezone',
        'geofence_lat',
        'geofence_lng',
        'geofence_radius_meters',
        'is_active',
    ];

    protected $casts = [
        'geofence_lat' => 'decimal:7',
        'geofence_lng' => 'decimal:7',
        'geofence_radius_meters' => 'integer',
        'is_active' => 'boolean',
    ];

    public function employees()
    {
        return $this->hasMany(Employee::class);
    }

    public function departments()
    {
        return $this->hasMany(Department::class);
    }

    public function positions()
    {
        return $this->hasMany(Position::class);
    }

    public function biometricDevices()
    {
        return $this->hasMany(BiometricDevice::class);
    }

    public function attendanceRecords()
    {
        return $this->hasMany(AttendanceRecord::class);
    }

    public function hasGeofence(): bool
    {
        return $this->geofence_lat !== null 
            && $this->geofence_lng !== null 
            && $this->geofence_radius_meters !== null;
    }
}
```

#### User Model (Enhanced)
```php
// Models/User.php
namespace App\Models;

use App\Enums\UserRole;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable, SoftDeletes;

    protected $fillable = [
        'name',
        'email',
        'password',
        'employee_id',
        'role',
        'is_active',
        'email_verified_at',
    ];

    protected $hidden = [
        'password',
        'remember_token',
    ];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'role' => UserRole::class,
        'is_active' => 'boolean',
        'password' => 'hashed',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function sessions()
    {
        return $this->hasMany(UserSession::class);
    }
}
```

### Phase 3: Employee Management

#### Employee Model
```php
// Models/Employee.php
namespace App\Models;

use App\Enums\CivilStatus;
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\Gender;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Employee extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_no',
        'user_id',
        'branch_id',
        'first_name',
        'middle_name',
        'last_name',
        'suffix',
        'nickname',
        'birthdate',
        'gender',
        'civil_status',
        'nationality',
        'personal_email',
        'work_email',
        'mobile_no',
        'address_line',
        'city',
        'province',
        'postal_code',
        'department_id',
        'position_id',
        'direct_manager_id',
        'employment_type',
        'employment_status',
        'date_hired',
        'date_regularized',
        'date_separated',
        'sss_no',
        'philhealth_no',
        'pagibig_no',
        'tin_no',
        'bank_name',
        'bank_account_number',
        'bank_account_name',
        'is_field_employee',
        'photo',
    ];

    protected $casts = [
        'gender' => Gender::class,
        'civil_status' => CivilStatus::class,
        'employment_type' => EmploymentType::class,
        'employment_status' => EmploymentStatus::class,
        'birthdate' => 'date',
        'date_hired' => 'date',
        'date_regularized' => 'date',
        'date_separated' => 'date',
        'is_field_employee' => 'boolean',
    ];

    protected $hidden = [
        'sss_no',
        'philhealth_no',
        'pagibig_no',
        'tin_no',
        'bank_account_number',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function position()
    {
        return $this->belongsTo(Position::class);
    }

    public function directManager()
    {
        return $this->belongsTo(Employee::class, 'direct_manager_id');
    }

    public function subordinates()
    {
        return $this->hasMany(Employee::class, 'direct_manager_id');
    }

    public function assignments()
    {
        return $this->hasMany(EmployeeAssignment::class);
    }

    public function primaryAssignment()
    {
        return $this->hasOne(EmployeeAssignment::class)->where('is_primary', true);
    }

    public function emergencyContacts()
    {
        return $this->hasMany(EmergencyContact::class);
    }

    public function documents()
    {
        return $this->hasMany(EmployeeDocument::class);
    }

    public function allowances()
    {
        return $this->hasMany(EmployeeAllowance::class);
    }

    public function schedules()
    {
        return $this->hasMany(EmployeeSchedule::class);
    }

    public function currentSchedule()
    {
        return $this->hasOne(EmployeeSchedule::class)
            ->where('effective_date', '<=', now())
            ->where(function ($query) {
                $query->whereNull('end_date')
                      ->orWhere('end_date', '>=', now());
            })
            ->latest('effective_date');
    }

    public function attendanceRecords()
    {
        return $this->hasMany(AttendanceRecord::class);
    }

    public function leaveBalances()
    {
        return $this->hasMany(LeaveBalance::class);
    }

    public function leaveRequests()
    {
        return $this->hasMany(LeaveRequest::class);
    }

    public function salaries()
    {
        return $this->hasMany(EmployeeSalary::class);
    }

    public function currentSalary()
    {
        return $this->hasOne(EmployeeSalary::class)
            ->where('effective_date', '<=', now())
            ->where(function ($query) {
                $query->whereNull('end_date')
                      ->orWhere('end_date', '>=', now());
            })
            ->latest('effective_date');
    }

    public function statutoryLoans()
    {
        return $this->hasMany(EmployeeStatutoryLoan::class);
    }

    public function companyLoans()
    {
        return $this->hasMany(EmployeeCompanyLoan::class);
    }

    public function payrollItems()
    {
        return $this->hasMany(PayrollItem::class);
    }

    public function onboarding()
    {
        return $this->hasOne(EmployeeOnboarding::class);
    }

    public function contracts()
    {
        return $this->hasMany(EmployeeContract::class);
    }

    public function assetAssignments()
    {
        return $this->hasMany(AssetAssignment::class);
    }

    public function assignedAssets()
    {
        return $this->belongsToMany(Asset::class, 'asset_assignments')
            ->withPivot('assigned_by', 'assigned_at', 'expected_return_date', 'returned_at', 'returned_condition')
            ->wherePivotNull('returned_at');
    }

    public function clearances()
    {
        return $this->hasMany(EmployeeClearance::class);
    }

    public function deviceMappings()
    {
        return $this->hasMany(EmployeeDeviceMapping::class);
    }

    public function siteLocations()
    {
        return $this->hasMany(EmployeeSiteLocation::class);
    }

    public function consents()
    {
        return $this->hasMany(DataPrivacyConsent::class);
    }

    public function dataSubjectRequests()
    {
        return $this->hasMany(DataSubjectRequest::class);
    }

    public function certifications()
    {
        return $this->hasMany(EmployeeCertification::class);
    }

    public function trainingRecords()
    {
        return $this->hasMany(EmployeeTrainingRecord::class);
    }

    public function goals()
    {
        return $this->hasManyThrough(PerformanceReview::class, Goal::class);
    }

    public function performanceReviews()
    {
        return $this->hasMany(PerformanceReview::class);
    }

    public function reviewsGiven()
    {
        return $this->hasMany(PerformanceReview::class, 'reviewer_id');
    }

    public function pips()
    {
        return $this->hasMany(PerformanceImprovementPlan::class);
    }

    public function pipsAssigned()
    {
        return $this->hasMany(PerformanceImprovementPlan::class, 'reviewer_id');
    }

    public function grievanceCasesComplainant()
    {
        return $this->hasMany(GrievanceCase::class, 'complainant_id');
    }

    public function grievanceCasesRespondent()
    {
        return $this->hasMany(GrievanceCase::class, 'respondent_id');
    }

    public function disciplinaryCases()
    {
        return $this->hasMany(DisciplinaryCase::class);
    }

    public function helpdeskTickets()
    {
        return $this->hasMany(HrTicket::class);
    }

    public function assignedTickets()
    {
        return $this->hasMany(HrTicket::class, 'assigned_to');
    }

    public function getFullNameAttribute(): string
    {
        return trim("{$this->first_name} {$this->middle_name} {$this->last_name} {$this->suffix}");
    }
}
```

#### Department Model
```php
// Models/Department.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Department extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'name',
        'code',
        'parent_id',
        'head_employee_id',
        'is_active',
    ];

    protected $casts = [
        'is_active' => 'boolean',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function parent()
    {
        return $this->belongsTo(Department::class, 'parent_id');
    }

    public function children()
    {
        return $this->hasMany(Department::class, 'parent_id');
    }

    public function head()
    {
        return $this->belongsTo(Employee::class, 'head_employee_id');
    }

    public function positions()
    {
        return $this->hasMany(Position::class);
    }

    public function employees()
    {
        return $this->hasMany(Employee::class);
    }

    public function grievanceCases()
    {
        return $this->hasMany(GrievanceCase::class);
    }

    public function disciplinaryCases()
    {
        return $this->hasMany(DisciplinaryCase::class);
    }
}
```

#### Position Model
```php
// Models/Position.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Position extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'department_id',
        'name',
        'code',
        'level',
        'is_active',
    ];

    protected $casts = [
        'level' => 'integer',
        'is_active' => 'boolean',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function employees()
    {
        return $this->hasMany(Employee::class);
    }

    public function salaries()
    {
        return $this->hasMany(SalaryGrade::class);
    }
}
```

#### Holiday Model
```php
// Models/Holiday.php
namespace App\Models;

use App\Enums\HolidayType;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Holiday extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'name',
        'date',
        'type',
        'is_recurring',
    ];

    protected $casts = [
        'type' => HolidayType::class,
        'date' => 'date',
        'is_recurring' => 'boolean',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function isHoliday(Carbon $date): bool
    {
        if ($this->is_recurring) {
            return $date->format('m-d') === $this->date->format('m-d');
        }
        return $date->format('Y-m-d') === $this->date->format('Y-m-d');
    }
}
```

### Phase 4: Time & Attendance

#### Shift Model
```php
// Models/Shift.php
namespace App\Models;

use App\Enums\ShiftType;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Shift extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'name',
        'type',
        'time_in',
        'time_out',
        'break_minutes',
        'grace_period_minutes',
        'days_of_week',
        'is_night_shift',
        'is_active',
        'is_default',
    ];

    protected $casts = [
        'type' => ShiftType::class,
        'time_in' => 'datetime',
        'time_out' => 'datetime',
        'break_minutes' => 'integer',
        'grace_period_minutes' => 'integer',
        'days_of_week' => 'array',
        'is_night_shift' => 'boolean',
        'is_active' => 'boolean',
        'is_default' => 'boolean',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function employeeSchedules()
    {
        return $this->hasMany(EmployeeSchedule::class);
    }
}
```

#### Employee Schedule Model
```php
// Models/EmployeeSchedule.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class EmployeeSchedule extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'shift_id',
        'effective_date',
        'end_date',
    ];

    protected $casts = [
        'effective_date' => 'date',
        'end_date' => 'date',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function shift()
    {
        return $this->belongsTo(Shift::class);
    }
}
```

#### Attendance Record Model
```php
// Models/AttendanceRecord.php
namespace App\Models;

use App\Enums\AttendanceSource;
use App\Enums\GeofenceStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class AttendanceRecord extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'employee_id',
        'device_id',
        'log_date',
        'time_in',
        'time_out',
        'source',
        'raw_payload',
        'geolocation_lat',
        'geolocation_lng',
        'geolocation_accuracy_meters',
        'geofence_status',
        'geofence_approval_decision',
        'synced_at',
    ];

    protected $casts = [
        'source' => AttendanceSource::class,
        'geofence_status' => GeofenceStatus::class,
        'log_date' => 'date',
        'time_in' => 'datetime',
        'time_out' => 'datetime',
        'raw_payload' => 'array',
        'synced_at' => 'datetime',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function device()
    {
        return $this->belongsTo(BiometricDevice::class, 'device_id');
    }

    public function adjustments()
    {
        return $this->hasMany(AttendanceAdjustment::class);
    }
}
```

#### DTR Entry Model
```php
// Models/DtrEntry.php
namespace App\Models;

use App\Enums\AttendanceStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class DtrEntry extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'work_date',
        'expected_in',
        'expected_out',
        'actual_in',
        'actual_out',
        'late_minutes',
        'undertime_minutes',
        'overtime_minutes',
        'night_diff_minutes',
        'work_hours',
        'status',
        'geofence_status',
        'is_adjusted',
    ];

    protected $casts = [
        'status' => AttendanceStatus::class,
        'work_date' => 'date',
        'expected_in' => 'datetime',
        'expected_out' => 'datetime',
        'actual_in' => 'datetime',
        'actual_out' => 'datetime',
        'late_minutes' => 'integer',
        'undertime_minutes' => 'integer',
        'overtime_minutes' => 'integer',
        'night_diff_minutes' => 'integer',
        'work_hours' => 'decimal:2',
        'is_adjusted' => 'boolean',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function correctionRequests()
    {
        return $this->hasMany(DtrCorrectionRequest::class);
    }
}
```

### Phase 5: Leave Management

#### Leave Type Model
```php
// Models/LeaveType.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class LeaveType extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'code',
        'description',
        'applicable_gender',
        'is_paid',
        'requires_document',
        'is_convertible_to_cash',
        'cash_conversion_rate',
        'max_days_per_request',
        'min_days_notice',
        'is_active',
    ];

    protected $casts = [
        'is_paid' => 'boolean',
        'requires_document' => 'boolean',
        'is_convertible_to_cash' => 'boolean',
        'cash_conversion_rate' => 'decimal:2',
        'max_days_per_request' => 'integer',
        'min_days_notice' => 'integer',
        'is_active' => 'boolean',
    ];

    public function accrualRules()
    {
        return $this->hasMany(LeaveAccrualRule::class);
    }

    public function carryoverRules()
    {
        return $this->hasMany(LeaveCarryoverRule::class);
    }

    public function approvalChains()
    {
        return $this->hasMany(LeaveApprovalChain::class);
    }

    public function balances()
    {
        return $this->hasMany(LeaveBalance::class);
    }

    public function requests()
    {
        return $this->hasMany(LeaveRequest::class);
    }
}
```

#### Leave Balance Model
```php
// Models/LeaveBalance.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class LeaveBalance extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'leave_type_id',
        'year',
        'opening_balance',
        'accrued',
        'used',
        'carry_forward',
        'cash_converted',
        'closing_balance',
        'last_accrued_at',
    ];

    protected $casts = [
        'opening_balance' => 'decimal:2',
        'accrued' => 'decimal:2',
        'used' => 'decimal:2',
        'carry_forward' => 'decimal:2',
        'cash_converted' => 'decimal:2',
        'closing_balance' => 'decimal:2',
        'last_accrued_at' => 'datetime',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function leaveType()
    {
        return $this->belongsTo(LeaveType::class);
    }
}
```

#### Leave Request Model
```php
// Models/LeaveRequest.php
namespace App\Models;

use App\Enums\LeaveRequestStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class LeaveRequest extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'leave_type_id',
        'approver_id',
        'start_date',
        'end_date',
        'total_days',
        'reason',
        'attachment_path',
        'status',
    ];

    protected $casts = [
        'status' => LeaveRequestStatus::class,
        'start_date' => 'date',
        'end_date' => 'date',
        'total_days' => 'decimal:2',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function leaveType()
    {
        return $this->belongsTo(LeaveType::class);
    }

    public function approver()
    {
        return $this->belongsTo(Employee::class, 'approver_id');
    }

    public function approvals()
    {
        return $this->hasMany(LeaveApproval::class);
    }
}
```

### Phase 6: Payroll

#### Payroll Run Model
```php
// Models/PayrollRun.php
namespace App\Models;

use App\Enums\PayrollStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class PayrollRun extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'period_start',
        'period_end',
        'run_date',
        'status',
        'total_employees',
        'processed_count',
        'failed_count',
    ];

    protected $casts = [
        'status' => PayrollStatus::class,
        'period_start' => 'date',
        'period_end' => 'date',
        'run_date' => 'datetime',
        'total_employees' => 'integer',
        'processed_count' => 'integer',
        'failed_count' => 'integer',
    ];

    public function payrollItems()
    {
        return $this->hasMany(PayrollItem::class);
    }
}
```

#### Payroll Item Model
```php
// Models/PayrollItem.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class PayrollItem extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'payroll_run_id',
        'employee_id',
        'assignment_id',
        'basic_pay',
        'gross_pay',
        'net_pay',
        'status',
        'error_details',
    ];

    protected $casts = [
        'basic_pay' => 'decimal:2',
        'gross_pay' => 'decimal:2',
        'net_pay' => 'decimal:2',
        'error_details' => 'array',
    ];

    public function payrollRun()
    {
        return $this->belongsTo(PayrollRun::class);
    }

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function assignment()
    {
        return $this->belongsTo(EmployeeAssignment::class);
    }

    public function earnings()
    {
        return $this->hasMany(Earning::class);
    }

    public function deductions()
    {
        return $this->hasMany(Deduction::class);
    }

    public function sssContribution()
    {
        return $this->hasOne(SssContribution::class);
    }

    public function pagibigContribution()
    {
        return $this->hasOne(PagibigContribution::class);
    }

    public function philhealthContribution()
    {
        return $this->hasOne(PhilhealthContribution::class);
    }

    public function bir2316Record()
    {
        return $this->hasOne(Bir2316Record::class);
    }
}
```

### Additional Supporting Models

#### Employee Assignment Model
```php
// Models/EmployeeAssignment.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class EmployeeAssignment extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'branch_id',
        'department_id',
        'position_id',
        'manager_id',
        'work_schedule_id',
        'effective_date',
        'end_date',
        'is_primary',
    ];

    protected $casts = [
        'effective_date' => 'date',
        'end_date' => 'date',
        'is_primary' => 'boolean',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function position()
    {
        return $this->belongsTo(Position::class);
    }

    public function manager()
    {
        return $this->belongsTo(Employee::class, 'manager_id');
    }

    public function workSchedule()
    {
        return $this->belongsTo(WorkSchedule::class);
    }

    public function payrollItems()
    {
        return $this->hasMany(PayrollItem::class);
    }
}
```

#### Biometric Device Model
```php
// Models/BiometricDevice.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class BiometricDevice extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'name',
        'brand',
        'protocol',
        'serial_number',
        'comm_key',
        'last_seen_at',
        'last_sync_status',
        'is_active',
    ];

    protected $casts = [
        'last_seen_at' => 'datetime',
        'is_active' => 'boolean',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function employeeMappings()
    {
        return $this->hasMany(EmployeeDeviceMapping::class);
    }

    public function attendanceRecords()
    {
        return $this->hasMany(AttendanceRecord::class);
    }
}
```

---

## 🏭 Factory Definitions

### Core Factories

#### Employee Factory
```php
// Database/Factories/EmployeeFactory.php
namespace Database\Factories;

use App\Enums\CivilStatus;
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\Gender;
use App\Models\Employee;
use Illuminate\Database\Eloquent\Factories\Factory;

class EmployeeFactory extends Factory
{
    protected $model = Employee::class;

    public function definition(): array
    {
        return [
            'employee_no' => 'EMP-' . now()->format('Ymd') . '-' . fake()->unique()->randomNumber(3, true),
            'first_name' => fake()->firstName(),
            'middle_name' => fake()->optional()->firstName(),
            'last_name' => fake()->lastName(),
            'email' => fake()->unique()->safeEmail(),
            'mobile_no' => fake()->phoneNumber(),
            'address_line' => fake()->streetAddress(),
            'city' => fake()->city(),
            'province' => fake()->state(),
            'postal_code' => fake()->postcode(),
            'gender' => fake()->randomElement(Gender::cases()),
            'civil_status' => fake()->randomElement(CivilStatus::cases()),
            'nationality' => 'Filipino',
            'employment_type' => fake()->randomElement(EmploymentType::cases()),
            'employment_status' => EmploymentStatus::ACTIVE,
            'date_hired' => fake()->dateTimeBetween('-2 years', 'now'),
            'is_field_employee' => false,
            'is_active' => true,
        ];
    }

    public function withUser(): self
    {
        return $this->afterCreating(function (Employee $employee) {
            \App\Models\User::factory()->create([
                'name' => $employee->full_name,
                'email' => $employee->email,
                'employee_id' => $employee->id,
            ]);
        });
    }

    public function regular(): self
    {
        return $this->state(fn (array $attributes) => [
            'employment_type' => EmploymentType::REGULAR,
            'date_regularized' => fake()->dateTimeBetween($attributes['date_hired'], '+6 months'),
        ]);
    }

    public function active(): self
    {
        return $this->state(fn (array $attributes) => [
            'employment_status' => EmploymentStatus::ACTIVE,
        ]);
    }

    public function fieldEmployee(): self
    {
        return $this->state(fn (array $attributes) => [
            'is_field_employee' => true,
        ]);
    }
}
```

#### Branch Factory
```php
// Database/Factories/BranchFactory.php
namespace Database\Factories;

use App\Models\Branch;
use Illuminate\Database\Eloquent\Factories\Factory;

class BranchFactory extends Factory
{
    protected $model = Branch::class;

    public function definition(): array
    {
        return [
            'name' => fake()->company() . ' Branch',
            'code' => strtoupper(fake()->unique()->lexify('BRANCH-????')),
            'address' => fake()->streetAddress(),
            'city' => fake()->city(),
            'timezone' => 'Asia/Manila',
            'is_active' => true,
        ];
    }

    public function withGeofence(): self
    {
        return $this->state(fn (array $attributes) => [
            'geofence_lat' => fake()->latitude(14.5, 14.7),
            'geofence_lng' => fake()->longitude(120.9, 121.1),
            'geofence_radius_meters' => 100,
        ]);
    }
}
```

#### Department Factory
```php
// Database/Factories/DepartmentFactory.php
namespace Database\Factories;

use App\Models\Department;
use Illuminate\Database\Eloquent\Factories\Factory;

class DepartmentFactory extends Factory
{
    protected $model = Department::class;

    public function definition(): array
    {
        return [
            'name' => fake()->word(),
            'code' => strtoupper(fake()->unique()->lexify('DEPT-????')),
            'is_active' => true,
        ];
    }

    public function withParent(): self
    {
        return $this->afterCreating(function (Department $department) {
            $parent = Department::inRandomOrder()->first();
            if ($parent) {
                $department->parent_id = $parent->id;
                $department->save();
            }
        });
    }
}
```

#### Position Factory
```php
// Database/Factories/PositionFactory.php
namespace Database\Factories;

use App\Models\Position;
use Illuminate\Database\Eloquent\Factories\Factory;

class PositionFactory extends Factory
{
    protected $model = Position::class;

    public function definition(): array
    {
        return [
            'name' => fake()->jobTitle(),
            'code' => strtoupper(fake()->unique()->lexify('POS-????')),
            'level' => fake()->numberBetween(1, 10),
            'is_active' => true,
        ];
    }
}
```

#### Holiday Factory
```php
// Database/Factories/HolidayFactory.php
namespace Database\Factories;

use App\Enums\HolidayType;
use App\Models\Holiday;
use Illuminate\Database\Eloquent\Factories\Factory;

class HolidayFactory extends Factory
{
    protected $model = Holiday::class;

    public function definition(): array
    {
        return [
            'name' => fake()->word() . ' Holiday',
            'date' => fake()->dateTimeBetween('+1 month', '+12 months')->format('Y-m-d'),
            'type' => fake()->randomElement(HolidayType::cases()),
            'is_recurring' => fake()->boolean(70), // 70% chance of recurring
        ];
    }

    public function regularHoliday(): self
    {
        return $this->state(fn (array $attributes) => [
            'type' => HolidayType::REGULAR,
        ]);
    }

    public function specialHoliday(): self
    {
        return $this->state(fn (array $attributes) => [
            'type' => HolidayType::SPECIAL_NON_WORKING,
        ]);
    }
}
```

---

## 🌱 Seeder Definitions

### Foundation Seeders

#### CompanySettingsSeeder
```php
// Database/Seeders/CompanySettingsSeeder.php
namespace Database\Seeders;

use App\Models\CompanySettings;
use Illuminate\Database\Seeder;

class CompanySettingsSeeder extends Seeder
{
    public function run(): void
    {
        CompanySettings::firstOrCreate(
            ['id' => 1],
            [
                'name' => config('app.name', 'HRIS System'),
                'address' => '123 Business District',
                'city' => 'Manila',
                'country' => 'Philippines',
                'timezone' => 'Asia/Manila',
                'locale' => 'en',
                'currency' => 'PHP',
                'tin_no' => '000-000-000-000',
            ]
        );
    }
}
```

#### BranchSeeder
```php
// Database/Seeders/BranchSeeder.php
namespace Database\Seeders;

use App\Models\Branch;
use Illuminate\Database\Seeder;

class BranchSeeder extends Seeder
{
    public function run(): void
    {
        Branch::firstOrCreate(
            ['code' => 'MAIN'],
            [
                'name' => 'Main Office',
                'address' => '123 Business District',
                'city' => 'Manila',
                'timezone' => 'Asia/Manila',
                'is_active' => true,
            ]
        );
    }
}
```

#### DepartmentSeeder
```php
// Database/Seeders/DepartmentSeeder.php
namespace Database\Seeders;

use App\Models\Department;
use Illuminate\Database\Seeder;

class DepartmentSeeder extends Seeder
{
    public function run(): void
    {
        $departments = [
            ['name' => 'Human Resources', 'code' => 'HR'],
            ['name' => 'Finance', 'code' => 'FIN'],
            ['name' => 'Information Technology', 'code' => 'IT'],
            ['name' => 'Operations', 'code' => 'OPS'],
            ['name' => 'Sales', 'code' => 'SALES'],
            ['name' => 'Marketing', 'code' => 'MKT'],
        ];

        foreach ($departments as $department) {
            Department::firstOrCreate(
                ['code' => $department['code']],
                [
                    'name' => $department['name'],
                    'is_active' => true,
                ]
            );
        }
    }
}
```

#### PositionSeeder
```php
// Database/Seeders/PositionSeeder.php
namespace Database\Seeders;

use App\Models\Position;
use Illuminate\Database\Seeder;

class PositionSeeder extends Seeder
{
    public function run(): void
    {
        $positions = [
            ['name' => 'Manager', 'code' => 'MGR', 'level' => 8],
            ['name' => 'Senior Associate', 'code' => 'SR_ASSOC', 'level' => 6],
            ['name' => 'Associate', 'code' => 'ASSOC', 'level' => 4],
            ['name' => 'Junior Associate', 'code' => 'JR_ASSOC', 'level' => 2],
            ['name' => 'Intern', 'code' => 'INTERN', 'level' => 1],
        ];

        foreach ($positions as $position) {
            Position::firstOrCreate(
                ['code' => $position['code']],
                [
                    'name' => $position['name'],
                    'level' => $position['level'],
                    'is_active' => true,
                ]
            );
        }
    }
}
```

#### HolidaySeeder
```php
// Database/Seeders/HolidaySeeder.php
namespace Database\Seeders;

use App\Enums\HolidayType;
use App\Models\Holiday;
use Illuminate\Database\Seeder;

class HolidaySeeder extends Seeder
{
    public function run(): void
    {
        $year = now()->year;
        
        $holidays = [
            ['name' => 'New Year\'s Day', 'date' => "{$year}-01-01", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'Maundy Thursday', 'date' => "{$year}-04-02", 'type' => HolidayType::REGULAR, 'is_recurring' => false],
            ['name' => 'Good Friday', 'date' => "{$year}-04-03", 'type' => HolidayType::REGULAR, 'is_recurring' => false],
            ['name' => 'Araw ng Kagitingan', 'date' => "{$year}-04-09", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' 'Labor Day', 'date' => "{$year}-05-01", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'Independence Day', 'date' => "{$year}-06-12", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'National Heroes Day', 'date' => "{$year}-08-27", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'Bonifacio Day', 'date' => "{$year}-11-30", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'Christmas Day', 'date' => "{$year}-12-25", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
            ['name' => 'Rizal Day', 'date' => "{$year}-12-30", 'type' => HolidayType::REGULAR, 'is_recurring' => true],
        ];

        foreach ($holidays as $holiday) {
            Holiday::firstOrCreate(
                ['name' => $holiday['name'], 'date' => $holiday['date']],
                [
                    'type' => $holiday['type'],
                    'is_recurring' => $holiday['is_recurring'],
                ]
            );
        }
    }
}
```

#### LeaveTypeSeeder
```php
// Database/Seeders/LeaveTypeSeeder.php
namespace Database\Seeders;

use App\Models\LeaveType;
use Illuminate\Database\Seeder;

class LeaveTypeSeeder extends Seeder
{
    public function run(): void
    {
        $leaveTypes = [
            [
                'name' => 'Sick Leave',
                'code' => 'SL',
                'description' => 'For employee illness or medical appointments',
                'is_paid' => true,
                'requires_document' => true,
                'is_convertible_to_cash' => true,
                'cash_conversion_rate' => 0.50,
                'max_days_per_request' => 5,
                'min_days_notice' => 1,
            ],
            [
                'name' => 'Vacation Leave',
                'code' => 'VL',
                'description' => 'For personal vacation and rest',
                'is_paid' => true,
                'requires_document' => false,
                'is_convertible_to_cash' => true,
                'cash_conversion_rate' => 0.50,
                'max_days_per_request' => 10,
                'min_days_notice' => 7,
            ],
            [
                'name' => 'Emergency Leave',
                'code' => 'EL',
                'description' => 'For emergency situations',
                'is_paid' => true,
                'requires_document' => true,
                'is_convertible_to_cash' => false,
                'cash_conversion_rate' => 0.00,
                'max_days_per_request' => 3,
                'min_days_notice' => 0,
            ],
            [
                'name' => 'Maternity Leave',
                'code' => 'ML',
                'description' => 'For female employees for childbirth',
                'is_paid' => true,
                'requires_document' => true,
                'is_convertible_to_cash' => false,
                'cash_conversion_rate' => 0.00,
                'max_days_per_request' => 105,
                'min_days_notice' => 30,
            ],
            [
                'name' => 'Paternity Leave',
                'code' => 'PL',
                'description' => 'For male employees for childbirth',
                'is_paid' => true,
                'requires_document' => true,
                'is_convertible_to_cash' => false,
                'cash_conversion_rate' => 0.00,
                'max_days_per_request' => 7,
                'min_days_notice' => 30,
            ],
            [
                'name' => 'Service Incentive Leave',
                'code' => 'SIL',
                'description' => 'For long-serving employees',
                'is_paid' => true,
                'requires_document' => false,
                'is_convertible_to_cash' => true,
                'cash_conversion_rate' => 1.00,
                'max_days_per_request' => 15,
                'min_days_notice' => 30,
            ],
        ];

        foreach ($leaveTypes as $leaveType) {
            LeaveType::firstOrCreate(
                ['code' => $leaveType['code']],
                [
                    'name' => $leaveType['name'],
                    'description' => $leaveType['description'],
                    'is_paid' => $leaveType['is_paid'],
                    'requires_document' => $leaveType['requires_document'],
                    'is_convertible_to_cash' => $leaveType['is_convertible_to_cash'],
                    'cash_conversion_rate' => $leaveType['cash_conversion_rate'],
                    'max_days_per_request' => $leaveType['max_days_per_request'],
                    'min_days_notice' => $leaveType['min_days_notice'],
                    'is_active' => true,
                ]
            );
        }
    }
}
```

### Additional Models for Recruitment

#### Job Requisition Model
```php
// Models/JobRequisition.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class JobRequisition extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'department_id',
        'position_id',
        'requested_by',
        'title',
        'description',
        'requirements',
        'salary_range_min',
        'salary_range_max',
        'vacancies_count',
        'status',
        'requested_date',
        'target_date',
    ];

    protected $casts = [
        'salary_range_min' => 'decimal:2',
        'salary_range_max' => 'decimal:2',
        'vacancies_count' => 'integer',
        'requested_date' => 'date',
        'target_date' => 'date',
    ];

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function position()
    {
        return $this->belongsTo(Position::class);
    }

    public function requestedBy()
    {
        return $this->belongsTo(Employee::class, 'requested_by');
    }

    public function jobPosting()
    {
        return $this->hasOne(JobPosting::class);
    }

    public function applications()
    {
        return $this->hasMany(JobApplication::class);
    }
}
```

#### Job Application Model
```php
// Models/JobApplication.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class JobApplication extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'job_requisition_id',
        'applicant_name',
        'email',
        'phone',
        'resume_path',
        'cover_letter_path',
        'status',
        'applied_date',
    ];

    protected $casts = [
        'applied_date' => 'date',
    ];

    public function jobRequisition()
    {
        return $this->belongsTo(JobRequisition::class);
    }

    public function stages()
    {
        return $this->hasMany(ApplicationStage::class);
    }

    public function jobOffer()
    {
        return $this->hasOne(JobOffer::class);
    }
}
```

### Performance Management Models

#### Performance Review Model
```php
// Models/PerformanceReview.php
namespace App\Models;

use App\Enums\PerformanceRating;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class PerformanceReview extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'review_cycle_id',
        'employee_id',
        'reviewer_id',
        'department_id',
        'overall_rating',
        'comments',
        'status',
        'review_date',
    ];

    protected $casts = [
        'overall_rating' => PerformanceRating::class,
        'review_date' => 'date',
    ];

    public function reviewCycle()
    {
        return $this->belongsTo(ReviewCycle::class);
    }

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function reviewer()
    {
        return $this->belongsTo(Employee::class, 'reviewer_id');
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function goals()
    {
        return $this->hasMany(Goal::class);
    }
}
```

#### Goal Model
```php
// Models/Goal.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Goal extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'performance_review_id',
        'title',
        'description',
        'weight',
        'target_value',
        'actual_value',
        'status',
        'due_date',
    ];

    protected $casts = [
        'weight' => 'decimal:2',
        'target_value' => 'decimal:2',
        'actual_value' => 'decimal:2',
        'due_date' => 'date',
    ];

    public function performanceReview()
    {
        return $this->belongsTo(PerformanceReview::class);
    }
}
```

#### Performance Improvement Plan Model
```php
// Models/PerformanceImprovementPlan.php
namespace App\Models;

use App\Enums\PipStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class PerformanceImprovementPlan extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'reviewer_id',
        'start_date',
        'end_date',
        'status',
        'objectives',
        'outcome',
    ];

    protected $casts = [
        'status' => PipStatus::class,
        'start_date' => 'date',
        'end_date' => 'date',
        'objectives' => 'array',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function reviewer()
    {
        return $this->belongsTo(Employee::class, 'reviewer_id');
    }

    public function milestones()
    {
        return $this->hasMany(PipMilestone::class);
    }
}
```

### Asset Management Models

#### Asset Model
```php
// Models/Asset.php
namespace App\Models;

use App\Enums\AssetCondition;
use App\Enums\AssetStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class Asset extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'branch_id',
        'asset_category_id',
        'name',
        'asset_tag',
        'serial_no',
        'description',
        'condition',
        'status',
        'purchase_date',
        'purchase_cost',
        'warranty_expiry',
    ];

    protected $casts = [
        'condition' => AssetCondition::class,
        'status' => AssetStatus::class,
        'purchase_date' => 'date',
        'purchase_cost' => 'decimal:2',
        'warranty_expiry' => 'date',
    ];

    public function branch()
    {
        return $this->belongsTo(Branch::class);
    }

    public function category()
    {
        return $this->belongsTo(AssetCategory::class, 'asset_category_id');
    }

    public function assignments()
    {
        return $this->hasMany(AssetAssignment::class);
    }
}
```

#### Asset Assignment Model
```php
// Models/AssetAssignment.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class AssetAssignment extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'asset_id',
        'employee_id',
        'assigned_by',
        'assigned_at',
        'expected_return_date',
        'returned_at',
        'returned_condition',
    ];

    protected $casts = [
        'assigned_at' => 'datetime',
        'expected_return_date' => 'date',
        'returned_at' => 'datetime',
        'returned_condition' => AssetCondition::class,
    ];

    public function asset()
    {
        return $this->belongsTo(Asset::class);
    }

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function assignedBy()
    {
        return $this->belongsTo(User::class, 'assigned_by');
    }
}
```

### Employee Relations Models

#### Grievance Case Model
```php
// Models/GrievanceCase.php
namespace App\Models;

use App\Enums\GrievanceStatus;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class GrievanceCase extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'complainant_id',
        'respondent_id',
        'department_id',
        'title',
        'description',
        'severity',
        'status',
        'filed_date',
        'resolution_notes',
        'resolved_date',
        'resolved_by',
    ];

    protected $casts = [
        'status' => GrievanceStatus::class,
        'filed_date' => 'date',
        'resolved_date' => 'date',
    ];

    public function complainant()
    {
        return $this->belongsTo(Employee::class, 'complainant_id');
    }

    public function respondent()
    {
        return $this->belongsTo(Employee::class, 'respondent_id');
    }

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function resolvedBy()
    {
        return $this->belongsTo(Employee::class, 'resolved_by');
    }

    public function steps()
    {
        return $this->hasMany(GrievanceCaseStep::class);
    }
}
```

#### HR Ticket Model
```php
// Models/HrTicket.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class HrTicket extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'category',
        'subject',
        'description',
        'priority',
        'status',
        'assigned_to',
        'resolution',
        'resolved_at',
    ];

    protected $casts = [
        'resolved_at' => 'datetime',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function assignedTo()
    {
        return $this->belongsTo(Employee::class, 'assigned_to');
    }

    public function replies()
    {
        return $this->hasMany(HrTicketReply::class);
    }
}
```

### Training Models

#### Training Program Model
```php
// Models/TrainingProgram.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class TrainingProgram extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'description',
        'provider',
        'cost',
        'validity_period_months',
        'is_active',
    ];

    protected $casts = [
        'cost' => 'decimal:2',
        'validity_period_months' => 'integer',
        'is_active' => 'boolean',
    ];

    public function certifications()
    {
        return $this->hasMany(EmployeeCertification::class);
    }

    public function trainingRecords()
    {
        return $this->hasMany(EmployeeTrainingRecord::class);
    }
}
```

#### Employee Certification Model
```php
// Models/EmployeeCertification.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class EmployeeCertification extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'training_program_id',
        'certification_no',
        'issue_date',
        'expiry_date',
        'status',
        'document_path',
    ];

    protected $casts = [
        'issue_date' => 'date',
        'expiry_date' => 'date',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function trainingProgram()
    {
        return $this->belongsTo(TrainingProgram::class);
    }
}
```

### Privacy & Compliance Models

#### Data Privacy Consent Model
```php
// Models/DataPrivacyConsent.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class DataPrivacyConsent extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'consent_type',
        'consent_version',
        'consent_text',
        'consented_at',
        'ip_address',
        'user_agent',
    ];

    protected $casts = [
        'consented_at' => 'datetime',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
```

#### Data Subject Request Model
```php
// Models/DataSubjectRequest.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class DataSubjectRequest extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'employee_id',
        'request_type',
        'request_details',
        'status',
        'submitted_date',
        'completed_date',
    ];

    protected $casts = [
        'submitted_date' => 'datetime',
        'completed_date' => 'datetime',
    ];

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }
}
```

### Integration Models

#### Webhook Subscription Model
```php
// Models/WebhookSubscription.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class WebhookSubscription extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'name',
        'url',
        'events',
        'secret',
        'is_active',
        'last_triggered_at',
    ];

    protected $casts = [
        'events' => 'array',
        'last_triggered_at' => 'datetime',
        'is_active' => 'boolean',
    ];

    public function deliveries()
    {
        return $this->hasMany(WebhookDelivery::class);
    }
}
```

#### Webhook Delivery Model
```php
// Models/WebhookDelivery.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class WebhookDelivery extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'webhook_id',
        'event_type',
        'payload',
        'response_code',
        'response_body',
        'attempt_count',
        'delivered_at',
    ];

    protected $casts = [
        'payload' => 'array',
        'response_body' => 'text',
        'delivered_at' => 'datetime',
    ];

    public function webhook()
    {
        return $this->belongsTo(WebhookSubscription::class, 'webhook_id');
    }
}
```

---

## 🎯 Additional Enums

#### Attendance Source Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum AttendanceSource: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case API = 'api';
    case MANUAL = 'manual';
    case OFFLINE_SYNC = 'offline_sync';
    case BIOMETRIC_ADMS = 'biometric_adms';
    case FIELD_KIOSK = 'field_kiosk';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::API => 'API',
            self::MANUAL => 'Manual',
            self::OFFLINE_SYNC => 'Offline Sync',
            self::BIOMETRIC_ADMS => 'Biometric Device',
            self::FIELD_KIOSK => 'Field Kiosk',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::API => 'Direct API call with authentication',
            self::MANUAL => 'Manual entry by HR personnel',
            self::OFFLINE_SYNC => 'Synced from offline queue when connection restored',
            self::BIOMETRIC_ADMS => 'Biometric device automatic push via ADMS protocol',
            self::FIELD_KIOSK => 'Field employee kiosk terminal',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::API => 'primary',
            self::MANUAL => 'info',
            self::OFFLINE_SYNC => 'warning',
            self::BIOMETRIC_ADMS => 'success',
            self::FIELD_KIOSK => 'purple',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::API => Heroicon::GlobeAlt,
            self::MANUAL => Heroicon::Pencil,
            self::OFFLINE_SYNC => Heroicon::ArrowPath,
            self::BIOMETRIC_ADMS => Heroicon::Fingerprint,
            self::FIELD_KIOSK => Heroicon::DevicePhoneMobile,
        };
    }
}
```

#### Leave Type Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum LeaveType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case SICK_LEAVE = 'sick_leave';
    case VACATION_LEAVE = 'vacation_leave';
    case EMERGENCY_LEAVE = 'emergency_leave';
    case MATERNITY_LEAVE = 'maternity_leave';
    case PATERNITY_LEAVE = 'paternity_leave';
    case SERVICE_INCENTIVE_LEAVE = 'service_incentive_leave';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::SICK_LEAVE => 'Sick Leave',
            self::VACATION_LEAVE => 'Vacation Leave',
            self::EMERGENCY_LEAVE => 'Emergency Leave',
            self::MATERNITY_LEAVE => 'Maternity Leave',
            self::PATERNITY_LEAVE => 'Paternity Leave',
            self::SERVICE_INCENTIVE_LEAVE => 'Service Incentive Leave',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::SICK_LEAVE => 'For illness, medical appointments, or quarantine',
            self::VACATION_LEAVE => 'For personal vacation, rest, and recreation',
            self::EMERGENCY_LEAVE => 'For urgent family emergencies or personal crises',
            self::MATERNITY_LEAVE => 'For female employees before and after childbirth (105 days)',
            self::PATERNITY_LEAVE => 'For male employees supporting childbirth (7 days)',
            self::SERVICE_INCENTIVE_LEAVE => 'For employees with at least 1 year of service (15 days)',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::SICK_LEAVE => 'danger',
            self::VACATION_LEAVE => 'success',
            self::EMERGENCY_LEAVE => 'warning',
            self::MATERNITY_LEAVE => 'pink',
            self::PATERNITY_LEAVE => 'primary',
            self::SERVICE_INCENTIVE_LEAVE => 'info',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::SICK_LEAVE => Heroicon::HeartPulse,
            self::VACATION_LEAVE => Heroicon::Sun,
            self::EMERGENCY_LEAVE => Heroicon::ExclamationTriangle,
            self::MATERNITY_LEAVE => Heroicon::Users,
            self::PATERNITY_LEAVE => Heroicon::User,
            self::SERVICE_INCENTIVE_LEAVE => Heroicon::Star,
        };
    }
}
```

#### Shift Type Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum ShiftType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case FIXED = 'fixed';
    case FLEXIBLE = 'flexible';
    case ROTATING = 'rotating';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::FIXED => 'Fixed Shift',
            self::FLEXIBLE => 'Flexible Shift',
            self::ROTATING => 'Rotating Shift',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::FIXED => 'Fixed start and end times daily with defined schedule',
            self::FLEXIBLE => 'Flexible working hours within company-defined core hours',
            self::ROTATING => 'Rotating shift schedule with morning, afternoon, and night cycles',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::FIXED => 'primary',
            self::FLEXIBLE => 'success',
            self::ROTATING => 'info',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::FIXED => Heroicon::Clock,
            self::FLEXIBLE => Heroicon::AdjustmentsHorizontal,
            self::ROTATING => Heroicon::ArrowPath,
        };
    }
}
```

#### Employment Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum EmploymentStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
    case ON_LEAVE = 'on_leave';
    case RESIGNED = 'resigned';
    case TERMINATED = 'terminated';
    case RETIRED = 'retired';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::ACTIVE => 'Active',
            self::INACTIVE => 'Inactive',
            self::ON_LEAVE => 'On Leave',
            self::RESIGNED => 'Resigned',
            self::TERMINATED => 'Terminated',
            self::RETIRED => 'Retired',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::ACTIVE => 'Currently employed and actively working',
            self::INACTIVE => 'Employed but not currently active',
            self::ON_LEAVE => 'On approved leave of absence',
            self::RESIGNED => 'Voluntarily resigned from employment',
            self::TERMINATED => 'Employment terminated by company',
            self::RETIRED => 'Retired from service',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::ACTIVE => 'success',
            self::INACTIVE => 'gray',
            self::ON_LEAVE => 'warning',
            self::RESIGNED => 'info',
            self::TERMINATED => 'danger',
            self::RETIRED => 'purple',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::ACTIVE => Heroicon::CheckCircle,
            self::INACTIVE => Heroicon::PauseCircle,
            self::ON_LEAVE => Heroicon::CalendarDays,
            self::RESIGNED => Heroicon::ArrowLeftOnRectangle,
            self::TERMINATED => Heroicon::XCircle,
            self::RETIRED => Heroicon::CpuChip,
        };
    }
}
```

#### Employment Type Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum EmploymentType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case REGULAR = 'regular';
    case PROBATIONARY = 'probationary';
    case CONTRACTUAL = 'contractual';
    case PROJECT_BASED = 'project_based';
    case INTERN = 'intern';
    case CONSULTANT = 'consultant';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::REGULAR => 'Regular',
            self::PROBATIONARY => 'Probationary',
            self::CONTRACTUAL => 'Contractual',
            self::PROJECT_BASED => 'Project-Based',
            self::INTERN => 'Intern',
            self::CONSULTANT => 'Consultant',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::REGULAR => 'Permanent regular employee with full benefits',
            self::PROBATIONARY => 'Probationary period before regularization',
            self::CONTRACTUAL => 'Fixed-term contractual employment',
            self::PROJECT_BASED => 'Employment tied to specific project duration',
            self::INTERN => 'Internship program (paid or unpaid)',
            self::CONSULTANT => 'External consultant or contractor',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::REGULAR => 'success',
            self::PROBATIONARY => 'warning',
            self::CONTRACTUAL => 'info',
            self::PROJECT_BASED => 'purple',
            self::INTERN => 'gray',
            self::CONSULTANT => 'primary',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::REGULAR => Heroicon::BadgeCheck,
            self::PROBATIONARY => Heroicon::ExclamationCircle,
            self::CONTRACTUAL => Heroicon::DocumentText,
            self::PROJECT_BASED => Heroicon::Folder,
            self::INTERN => Heroicon::AcademicCap,
            self::CONSULTANT => Heroicon::Briefcase,
        };
    }
}
```

#### Leave Request Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum LeaveRequestStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PENDING = 'pending';
    case APPROVED = 'approved';
    case REJECTED = 'rejected';
    case CANCELLED = 'cancelled';
    case ON_HOLD = 'on_hold';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::PENDING => 'Pending',
            self::APPROVED => 'Approved',
            self::REJECTED => 'Rejected',
            self::CANCELLED => 'Cancelled',
            self::ON_HOLD => 'On Hold',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::PENDING => 'Awaiting manager approval',
            self::APPROVED => 'Leave request approved and scheduled',
            self::REJECTED => 'Leave request denied by manager',
            self::CANCELLED => 'Leave request cancelled by employee',
            self::ON_HOLD => 'Leave request temporarily suspended',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::PENDING => 'warning',
            self::APPROVED => 'success',
            self::REJECTED => 'danger',
            self::CANCELLED => 'gray',
            self::ON_HOLD => 'info',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::PENDING => Heroicon::Clock,
            self::APPROVED => Heroicon::CheckCircle,
            self::REJECTED => Heroicon::XCircle,
            self::CANCELLED => Heroicon::NoSymbol,
            self::ON_HOLD => Heroicon::PauseCircle,
        };
    }
}
```

#### Payroll Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum PayrollStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case DRAFT = 'draft';
    case IN_PROGRESS = 'in_progress';
    case REVIEW = 'review';
    case APPROVED = 'approved';
    case PROCESSED = 'processed';
    case FAILED = 'failed';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::DRAFT => 'Draft',
            self::IN_PROGRESS => 'In Progress',
            self::REVIEW => 'Under Review',
            self::APPROVED => 'Approved',
            self::PROCESSED => 'Processed',
            self::FAILED => 'Failed',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::DRAFT => 'Payroll run in draft state',
            self::IN_PROGRESS => 'Payroll computation in progress',
            self::REVIEW => 'Payroll pending final review',
            self::APPROVED => 'Payroll approved for processing',
            self::PROCESSED => 'Payroll successfully processed',
            self::FAILED => 'Payroll processing failed',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::DRAFT => 'gray',
            self::IN_PROGRESS => 'info',
            self::REVIEW => 'warning',
            self::APPROVED => 'success',
            self::PROCESSED => 'primary',
            self::FAILED => 'danger',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::DRAFT => Heroicon::DocumentText,
            self::IN_PROGRESS => Heroicon::ArrowPath,
            self::REVIEW => Heroicon::MagnifyingGlass,
            self::APPROVED => Heroicon::CheckCircle,
            self::PROCESSED => Heroicon::CheckBadge,
            self::FAILED => Heroicon::XCircle,
        };
    }
}
```

#### Attendance Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum AttendanceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PRESENT = 'present';
    case ABSENT = 'absent';
    case LATE = 'late';
    case UNDERTIME = 'undertime';
    case OVERTIME = 'overtime';
    case ON_LEAVE = 'on_leave';
    case HOLIDAY = 'holiday';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::PRESENT => 'Present',
            self::ABSENT => 'Absent',
            self::LATE => 'Late',
            self::UNDERTIME => 'Undertime',
            self::OVERTIME => 'Overtime',
            self::ON_LEAVE => 'On Leave',
            self::HOLIDAY => 'Holiday',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::PRESENT => 'Employee was present and on time',
            self::ABSENT => 'Employee was absent without leave',
            self::LATE => 'Employee arrived after scheduled time',
            self::UNDERTIME => 'Employee left before scheduled time',
            self::OVERTIME => 'Employee worked beyond scheduled hours',
            self::ON_LEAVE => 'Employee was on approved leave',
            self::HOLIDAY => 'Day was a company or public holiday',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::PRESENT => 'success',
            self::ABSENT => 'danger',
            self::LATE => 'warning',
            self::UNDERTIME => 'warning',
            self::OVERTIME => 'info',
            self::ON_LEAVE => 'gray',
            self::HOLIDAY => 'purple',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::PRESENT => Heroicon::CheckCircle,
            self::ABSENT => Heroicon::XCircle,
            self::LATE => Heroicon::Clock,
            self::UNDERTIME => Heroicon::ArrowLeft,
            self::OVERTIME => Heroicon::PlusCircle,
            self::ON_LEAVE => Heroicon::CalendarDays,
            self::HOLIDAY => Heroicon::Star,
        };
    }
}
```

#### Geofence Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum GeofenceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case INSIDE = 'inside';
    case OUTSIDE = 'outside';
    case BORDERLINE = 'borderline';
    case NOT_APPLICABLE = 'not_applicable';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::INSIDE => 'Inside Geofence',
            self::OUTSIDE => 'Outside Geofence',
            self::BORDERLINE => 'Borderline',
            self::NOT_APPLICABLE => 'Not Applicable',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::INSIDE => 'Punch was within the approved geofence radius',
            self::OUTSIDE => 'Punch was outside the approved geofence radius',
            self::BORDERLINE => 'Punch was borderline within tolerance range',
            self::NOT_APPLICABLE => 'Geofence not applicable for this employee',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::INSIDE => 'success',
            self::OUTSIDE => 'danger',
            self::BORDERLINE => 'warning',
            self::NOT_APPLICABLE => 'gray',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::INSIDE => Heroicon::MapPin,
            self::OUTSIDE => Heroicon::MapPin,
            self::BORDERLINE => Heroicon::ExclamationTriangle,
            self::NOT_APPLICABLE => Heroicon::MinusCircle,
        };
    }
}
```

#### Holiday Type Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum HolidayType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case REGULAR = 'regular';
    case SPECIAL_NON_WORKING = 'special_non_working';
    case SPECIAL_HOLIDAY = 'special_holiday';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::REGULAR => 'Regular Holiday',
            self::SPECIAL_NON_WORKING => 'Special Non-Working Holiday',
            self::SPECIAL_HOLIDAY => 'Special Holiday',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::REGULAR => 'National regular holiday with full pay benefits',
            self::SPECIAL_NON_WORKING => 'Special non-working holiday with differential pay',
            self::SPECIAL_HOLIDAY => 'Special holiday with limited benefits',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::REGULAR => 'primary',
            self::SPECIAL_NON_WORKING => 'info',
            self::SPECIAL_HOLIDAY => 'success',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::REGULAR => Heroicon::Star,
            self::SPECIAL_NON_WORKING => Heroicon::CalendarDays,
            self::SPECIAL_HOLIDAY => Heroicon::Gift,
        };
    }
}
```

#### Gender Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum Gender: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case MALE = 'male';
    case FEMALE = 'female';
    case OTHER = 'other';
    case PREFER_NOT_TO_SAY = 'prefer_not_to_say';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::MALE => 'Male',
            self::FEMALE => 'Female',
            self::OTHER => 'Other',
            self::PREFER_NOT_TO_SAY => 'Prefer not to say',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::MALE => 'Male gender',
            self::FEMALE => 'Female gender',
            self::OTHER => 'Other gender identity',
            self::PREFER_NOT_TO_SAY => 'Gender information not disclosed',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::MALE => 'primary',
            self::FEMALE => 'pink',
            self::OTHER => 'info',
            self::PREFER_NOT_TO_SAY => 'gray',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::MALE => Heroicon::User,
            self::FEMALE => Heroicon::User,
            self::OTHER => Heroicon::UserGroup,
            self::PREFER_NOT_TO_SAY => Heroicon::EyeSlash,
        };
    }
}
```

#### Civil Status Enum (Advanced Filament v5 Tricks)
```php
namespace App\Enums;

use BackedEnum;
use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;
use Illuminate\Contracts\Support\Htmlable;

enum CivilStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case SINGLE = 'single';
    case MARRIED = 'married';
    case WIDOWED = 'widowed';
    case SEPARATED = 'separated';
    case DIVORCED = 'divorced';

    public function getLabel(): string | Htmlable | null
    {
        return match($this) {
            self::SINGLE => 'Single',
            self::MARRIED => 'Married',
            self::WIDOWED => 'Widowed',
            self::SEPARATED => 'Separated',
            self::DIVORCED => 'Divorced',
        };
    }

    public function getDescription(): string | Htmlable | null
    {
        return match($this) {
            self::SINGLE => 'Never married',
            self::MARRIED => 'Currently married',
            self::WIDOWED => 'Spouse has passed away',
            self::SEPARATED => 'Legally separated',
            self::DIVORCED => 'Divorced',
        };
    }

    public function getColor(): string | array | null
    {
        return match($this) {
            self::SINGLE => 'info',
            self::MARRIED => 'success',
            self::WIDOWED => 'gray',
            self::SEPARATED => 'warning',
            self::DIVORCED => 'danger',
        };
    }

    public function getIcon(): string | BackedEnum | Htmlable | null
    {
        return match($this) {
            self::SINGLE => Heroicon::User,
            self::MARRIED => Heroicon::Heart,
            self::WIDOWED => Heroicon::Flower,
            self::SEPARATED => Heroicon::ArrowLeftOnRectangle,
            self::DIVORCED => Heroicon::Scissors,
        };
    }
}
```

### Filament v5 Enum Tricks Implementation Examples

#### Automatic Label Display in Tables
```php
// When model has enum casting, Filament automatically uses HasLabel
// Employee model has: protected $casts = ['employment_status' => EmploymentStatus::class];

// In Filament table - automatic enum integration
Tables\Columns\TextColumn::make('employment_status')
    ->badge() // Automatically uses getColor() and getLabel()
    ->sortable();
```

#### Automatic Label Display in Infolists
```php
// Filament automatically uses HasLabel for infolist entries
Infolists\Components\TextEntry::make('employment_status')
    ->badge(); // Automatically uses getColor() and getLabel()
```

#### Enum Options in Select Fields with Descriptions
```php
// Directly use enum class for options with helper text
Forms\Components\Select::make('employment_status')
    ->options(EmploymentStatus::class) // Automatically uses getLabel()
    ->enum(EmploymentStatus::class) // Validation uses enum
    ->helperText(fn (EmploymentStatus $status): string => $status->getDescription())
    ->live() // Show description when selection changes
    ->hint(fn (Forms\Components\Select $component): string => 
        $component->getState() 
            ? EmploymentStatus::from($component->getState())->getDescription() 
            : 'Select employment status'
    );
```

#### Enum Filters in Tables
```php
// Use enum for table filters with automatic labels
Tables\Filters\SelectFilter::make('employment_status')
    ->options(EmploymentStatus::class) // Automatically uses getLabel()
    ->query(function (Builder $query, array $data): Builder {
        return $query->when($data['value'], fn ($query, $value) => 
            $query->where('employment_status', $value)
        );
    });
```

#### Enum in Radio Buttons with Icons
```php
// Use enum for radio buttons with icons
Forms\Components\Radio::make('employment_type')
    ->options(EmploymentType::class)
    ->enum(EmploymentType::class)
    ->descriptions(fn (EmploymentType $type): string => $type->getDescription())
    ->icons(fn (EmploymentType $type): string => $type->getIcon());
```

#### Enum in Toggle Buttons with Colors
```php
// Use enum for toggle buttons with automatic colors/icons
Forms\Components\ToggleButtons::make('employment_status')
    ->options(EmploymentStatus::class)
    ->enum(EmploymentStatus::class)
    // Colors and icons automatically used from enum HasColor and HasIcon
```

#### Enum in Checkbox List
```php
// Use enum for checkbox list (for multi-select enums)
Forms\Components\CheckboxList::make('skills')
    ->options(Skill::class)
    ->descriptions(fn (Skill $skill): string => $skill->getDescription());
```

#### Enum Grouping in Tables
```php
// Use enum for table grouping with automatic labels
Tables\Columns\TextColumn::make('employment_status')
    ->badge()
    ->sortable()
    ->group('employment_status'); // Uses getLabel() for group headers
```

#### Enum in Relation Manager Forms
```php
// Use enum in relation manager forms
Forms\Components\Select::make('status')
    ->options(LeaveRequestStatus::class)
    ->enum(LeaveRequestStatus::class)
    ->default(LeaveRequestStatus::PENDING)
    ->live()
    ->afterStateUpdated(function (Forms\Components\Select $component, $state) {
        // Show/hide fields based on enum state
        if ($state === LeaveRequestStatus::APPROVED->value) {
            $component->getContainer()->getComponent('approval_date')->visible(true);
        }
    });
```

#### Dynamic Helper Text Based on Enum Selection
```php
Forms\Components\Select::make('leave_type')
    ->options(LeaveType::class)
    ->enum(LeaveType::class)
    ->live()
    ->hint(fn (Forms\Components\Select $component): string => {
        if (!$component->getState()) {
            return 'Select a leave type to see details';
        }
        
        $leaveType = LeaveType::from($component->getState());
        return sprintf(
            '%s • %s • %s days max',
            $leaveType->getLabel(),
            $leaveType->getDescription(),
            $leaveType->getMaxDaysPerRequest()
        );
    });
```

#### Conditional Actions Based on Enum State
```php
// In resource pages, conditionally show actions based on enum state
protected function getHeaderActions(): array
{
    return [
        Actions\Action::make('approve')
            ->visible(fn (): bool => 
                $this->record->status === LeaveRequestStatus::PENDING
            )
            ->requiresConfirmation()
            ->action(fn () => $this->record->update(['status' => LeaveRequestStatus::APPROVED])),
    ];
}
```

#### Enum Color Arrays for Advanced Styling
```php
// In enum definition, return color arrays for dynamic styling
public function getColor(): string | array | null
{
    return match($this) {
        self::ACTIVE => ['success', '50', '600'], // [color, bg-shade, text-shade]
        self::INACTIVE => 'gray',
    };
}

// In Filament, this automatically applies the color array
Tables\Columns\TextColumn::make('status')
    ->badge(); // Automatically uses the color array
```

#### HTML Descriptions for Rich Content
```php
// In enum definition, return Htmlable for rich content
use Illuminate\Support\HtmlString;

public function getDescription(): string | Htmlable | null
{
    return match($this) {
        self::REGULAR => new HtmlString('<strong>Regular</strong> employee with <em>full benefits</em>'),
        self::PROBATIONARY => 'Probationary period',
    };
}

// In Filament, this renders the HTML content
Forms\Components\Select::make('employment_type')
    ->options(EmploymentType::class)
    ->helperText(fn (EmploymentType $type): string => $type->getDescription());
```

#### Icon Composition with Heroicon
```php
// In enum definition, return BackedEnum for dynamic icons
public function getIcon(): string | BackedEnum | Htmlable | null
{
    return match($this) {
        self::API => Heroicon::GlobeAlt,
        self::MANUAL => 'heroicon-o-pencil', // Alternative string format
        self::OFFLINE_SYNC => new HtmlString('<svg>...</svg>'), // Custom SVG
    };
}

// In Filament, this automatically uses the icon
Tables\Columns\TextColumn::make('source')
    ->icon(fn (AttendanceSource $source): string => $source->getIcon());
```

#### Enum in Action Modals
```php
// Use enum in action modals with automatic labels
Actions\Action::make('change_status')
    ->form([
        Forms\Components\Select::make('new_status')
            ->options(EmploymentStatus::class)
            ->enum(EmploymentStatus::class)
            ->required()
            ->live()
            ->hint(fn (Forms\Components\Select $component): string => 
                $component->getState() 
                    ? EmploymentStatus::from($component->getState())->getDescription() 
                    : 'Select new status'
            ),
    ])
    ->action(function (array $data) {
        $this->record->update(['employment_status' => $data['new_status']]);
    });
```

#### Enum in Resource Headers/Global Search
```php
// Use enum in resource global search configuration
public static function getGloballySearchableAttributes(): array
{
    return [
        'employee_no',
        'first_name',
        'last_name',
        'email',
        'employment_status', // Enum field for global search
    ];
}

// Use enum in resource navigation badge
public static function getNavigationBadge(): ?string
{
    return static::getModel()::where('employment_status', EmploymentStatus::ACTIVE)->count();
}

public static function getNavigationBadgeColor(): ?string
{
    return EmploymentStatus::ACTIVE->getColor();
}
```

#### Enum in Bulk Actions
```php
// Use enum in bulk actions with conditional behavior
Tables\Actions\BulkAction::make('change_status')
    ->form([
        Forms\Components\Select::make('new_status')
            ->options(EmploymentStatus::class)
            ->enum(EmploymentStatus::class)
            ->required()
            ->descriptions(fn (EmploymentStatus $status): string => $status->getDescription()),
    ])
    ->action(function (Collection $records, array $data) {
        $records->each->update(['employment_status' => $data['new_status']]);
    });
```

#### Enum in Page Configuration
```php
// Use enum in page configuration for conditional rendering
use App\Enums\UserRole;

public function canView(): bool
{
    $user = auth()->user();

    if ($user->role === UserRole::COMPANY_ADMIN) {
        return true;
    }

    // Check employee status for self-service access
    if ($user->employee && $user->employee->employment_status === EmploymentStatus::ACTIVE) {
        return true;
    }

    return false;
}
```

#### Enum in Widgets
```php
// Use enum in dashboard widgets
use Filament\Widgets\StatsOverviewWidget;
use Filament\Widgets\StatsOverviewWidget\Stat;

class EmployeeStatusWidget extends StatsOverviewWidget
{
    protected function getStats(): array
    {
        return [
            Stat::make('Active Employees', Employee::where('employment_status', EmploymentStatus::ACTIVE)->count())
                ->description(EmploymentStatus::ACTIVE->getDescription())
                ->descriptionIcon(EmploymentStatus::ACTIVE->getIcon())
                ->color(EmploymentStatus::ACTIVE->getColor()), // Widgets need manual color calls
            Stat::make('On Leave', Employee::where('employment_status', EmploymentStatus::ON_LEAVE)->count())
                ->description(EmploymentStatus::ON_LEAVE->getDescription())
                ->descriptionIcon(EmploymentStatus::ON_LEAVE->getIcon())
                ->color(EmploymentStatus::ON_LEAVE->getColor()), // Widgets need manual color calls
        ];
    }
}
```

### Filament v5 Enum Integration Best Practices

#### When to Use Automatic vs. Manual Enum Handling

**Automatic Enum Handling (Filament v5 Native):**
- Tables with enum casting → use `badge()` only
- Infolists with enum casting → use `badge()` only
- Select fields with enum class → use `options(Enum::class)` only
- Filters with enum class → use `options(Enum::class)` only

**Manual Enum Handling (When Needed):**
- Custom color logic beyond enum definition
- Conditional helper text based on selection
- Dynamic form field visibility
- Custom action behavior based on enum state

#### Complete Enum Integration Checklist

**In Models:**
- ✅ Add enum to `$casts` array
- ✅ Ensure enum implements Filament interfaces (HasLabel, HasColor, HasIcon, HasDescription)
- ✅ Use proper return types (`string | Htmlable | null`, etc.)

**In Forms:**
- ✅ Use `options(Enum::class)` for select fields
- ✅ Add `enum(Enum::class)` for validation
- ✅ Use `helperText(fn (Enum $enum): string => $enum->getDescription())` for descriptions
- ✅ Use `live()` for dynamic helper text updates

**In Tables:**
- ✅ Use `badge()` for enum columns (automatic color/label)
- ✅ Use `options(Enum::class)` for enum filters
- ✅ Use `group('enum_field')` for table grouping

**In Infolists:**
- ✅ Use `badge()` for enum entries (automatic color/label)
- ✅ No manual color/label methods needed

**In Actions:**
- ✅ Use enum state for conditional visibility
- ✅ Use enum values in action forms
- ✅ Implement enum-based business logic

**In Widgets:**
- ✅ Use enum for dynamic statistics
- ✅ Use enum for conditional styling
- ✅ Use enum helper methods for descriptions/icons

#### Common Enum Integration Patterns

**Pattern 1: Standard Select with Enum**
```php
Forms\Components\Select::make('employment_type')
    ->options(EmploymentType::class)
    ->enum(EmploymentType::class)
    ->required();
```

**Pattern 2: Select with Description**
```php
Forms\Components\Select::make('employment_type')
    ->options(EmploymentType::class)
    ->enum(EmploymentType::class)
    ->helperText(fn (EmploymentType $type): string => $type->getDescription());
```

**Pattern 3: Select with Dynamic Helper Text**
```php
Forms\Components\Select::make('employment_type')
    ->options(EmploymentType::class)
    ->enum(EmploymentType::class)
    ->live()
    ->hint(fn (Forms\Components\Select $component): string => 
        $component->getState() 
            ? EmploymentType::from($component->getState())->getDescription() 
            : 'Select employment type'
    );
```

**Pattern 4: Table Badge with Enum**
```php
Tables\Columns\TextColumn::make('employment_status')
    ->badge()
    ->sortable();
```

**Pattern 5: Table Filter with Enum**
```php
Tables\Filters\SelectFilter::make('employment_status')
    ->options(EmploymentStatus::class);
```

**Pattern 6: Infolist Badge with Enum**
```php
Infolists\Components\TextEntry::make('employment_status')
    ->badge();
```

**Pattern 7: Radio Buttons with Enum**
```php
Forms\Components\Radio::make('gender')
    ->options(Gender::class)
    ->enum(Gender::class)
    ->inline();
```

**Pattern 8: Toggle Buttons with Enum**
```php
Forms\Components\ToggleButtons::make('employment_status')
    ->options(EmploymentStatus::class)
    ->enum(EmploymentStatus::class);
```

**Pattern 9: Conditional Actions Based on Enum**
```php
Actions\Action::make('approve')
    ->visible(fn (): bool => 
        $this->record->status === LeaveRequestStatus::PENDING
    );
```

**Pattern 10: Enum in Action Form**
```php
Actions\Action::make('change_status')
    ->form([
        Forms\Components\Select::make('new_status')
            ->options(EmploymentStatus::class)
            ->enum(EmploymentStatus::class)
            ->required(),
    ]);
```

**Pattern 11: Enum in Widgets (Manual Color Required)**
```php
// Widgets require manual color calls (no automatic enum integration)
Stat::make('Active Employees', Employee::where('employment_status', EmploymentStatus::ACTIVE)->count())
    ->color(EmploymentStatus::ACTIVE->getColor()) // Manual color call required
    ->descriptionIcon(EmploymentStatus::ACTIVE->getIcon());
```

**Pattern 12: Enum in Panel Configuration**
```php
public static function getNavigationBadgeColor(): ?string
{
    return EmploymentStatus::ACTIVE->getColor(); // Manual color call required
}
```

#### Enum Integration Summary

**Automatic Integration (No Manual Calls Needed):**
- Tables: `badge()` automatically uses enum colors/labels
- Infolists: `badge()` automatically uses enum colors/labels
- Select fields: `options(Enum::class)` automatically uses enum labels
- Filters: `options(Enum::class)` automatically uses enum labels

**Manual Integration Required:**
- Widgets: Manual `->color()` and `->descriptionIcon()` calls
- Panel configuration: Manual `->color()` calls
- Custom logic: Manual enum method calls as needed

---

## 🎯 Comprehensive Relationship Matrix

### Employee Model Relationships Summary
```php
// Employee hasMany
- assignments (EmployeeAssignment)
- emergencyContacts (EmergencyContact)
- documents (EmployeeDocument)
- allowances (EmployeeAllowance)
- schedules (EmployeeSchedule)
- attendanceRecords (AttendanceRecord)
- leaveBalances (LeaveBalance)
- leaveRequests (LeaveRequest)
- salaries (EmployeeSalary)
- statutoryLoans (EmployeeStatutoryLoan)
- companyLoans (EmployeeCompanyLoan)
- payrollItems (PayrollItem)
- onboarding (EmployeeOnboarding)
- contracts (EmployeeContract)
- assetAssignments (AssetAssignment)
- clearances (EmployeeClearance)
- deviceMappings (EmployeeDeviceMapping)
- siteLocations (EmployeeSiteLocation)
- consents (DataPrivacyConsent)
- dataSubjectRequests (DataSubjectRequest)
- certifications (EmployeeCertification)
- trainingRecords (EmployeeTrainingRecord)
- performanceReviews (PerformanceReview)
- reviewsGiven (PerformanceReview, as reviewer)
- pips (PerformanceImprovementPlan)
- pipsAssigned (PerformanceImprovementPlan, as reviewer)
- grievanceCasesComplainant (GrievanceCase)
- grievanceCasesRespondent (GrievanceCase)
- disciplinaryCases (DisciplinaryCase)
- helpdeskTickets (HrTicket)
- assignedTickets (HrTicket, as assigned_to)
- subordinates (Employee, as directManager)

// Employee belongsTo
- user (User)
- branch (Branch)
- department (Department)
- position (Position)
- directManager (Employee)
```

### Department Model Relationships Summary
```php
// Department hasMany
- positions (Position)
- employees (Employee)
- children (Department, as parent)
- grievanceCases (GrievanceCase)
- disciplinaryCases (DisciplinaryCase)

// Department belongsTo
- branch (Branch)
- parent (Department, as parent)
- head (Employee, as head_employee_id)
```

### Branch Model Relationships Summary
```php
// Branch hasMany
- employees (Employee)
- departments (Department)
- positions (Position)
- biometricDevices (BiometricDevice)
- attendanceRecords (AttendanceRecord)

// Branch belongsTo
- (No parent in single-tenant setup)
```

---

## 🧪 Test Examples

### Model Relationship Tests (Pest v4)
```php
// Tests/Unit/EmployeeRelationshipsTest.php
use App\Models\Employee;
use App\Models\User;
use App\Models\EmployeeAssignment;
use App\Models\LeaveRequest;
use App\Models\PayrollItem;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('employee belongs to user', function () {
    $employee = Employee::factory()->withUser()->create();
    
    expect($employee->user)->toBeInstanceOf(User::class)
        ->and($employee->id)->toBe($employee->user->employee_id);
});

test('employee has many assignments', function () {
    $employee = Employee::factory()->create();
    $assignment = EmployeeAssignment::factory()->create(['employee_id' => $employee->id]);
    
    expect($employee->assignments)->toHaveCount(1)
        ->and($employee->assignments->first()->id)->toBe($assignment->id);
});

test('employee has many leave requests', function () {
    $employee = Employee::factory()->create();
    LeaveRequest::factory()->create(['employee_id' => $employee->id]);
    
    expect($employee->leaveRequests)->toHaveCount(1);
});

test('employee has many payroll items', function () {
    $employee = Employee::factory()->create();
    PayrollItem::factory()->create(['employee_id' => $employee->id]);
    
    expect($employee->payrollItems)->toHaveCount(1);
});
```

### Enum Casting Tests (Pest v4)
```php
// Tests/Unit/EnumCastingTest.php
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Models\Employee;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('employment status enum casting', function () {
    $employee = Employee::factory()->create([
        'employment_status' => EmploymentStatus::ACTIVE,
    ]);

    expect($employee->employment_status)->toBeInstanceOf(EmploymentStatus::class)
        ->toBe(EmploymentStatus::ACTIVE);
});

test('employment type enum casting', function () {
    $employee = Employee::factory()->create([
        'employment_type' => EmploymentType::REGULAR,
    ]);

    expect($employee->employment_type)->toBeInstanceOf(EmploymentType::class)
        ->toBe(EmploymentType::REGULAR);
});

test('enum methods return correct values', function () {
    $status = EmploymentStatus::ACTIVE;
    
    expect($status->getLabel())->toBe('Active')
        ->and($status->getDescription())->toBe('Currently employed and active')
        ->and($status->getColor())->toBe('success')
        ->and($status->getIcon())->not->toBeNull();
});

test('enum automatic filament integration', function () {
    $status = EmploymentStatus::ACTIVE;
    
    // Test that enum provides all required interfaces for Filament automatic integration
    expect($status)->toBeInstanceOf(\Filament\Support\Contracts\HasLabel::class)
        ->and($status)->toBeInstanceOf(\Filament\Support\Contracts\HasColor::class)
        ->and($status)->toBeInstanceOf(\Filament\Support\Contracts\HasIcon::class)
        ->and($status)->toBeInstanceOf(\Filament\Support\Contracts\HasDescription::class);
});
```

### Factory Tests (Pest v4)
```php
// Tests/Unit/EmployeeFactoryTest.php
use App\Models\Employee;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('factory creates employee with unique employee no', function () {
    $employees = Employee::factory()->count(5)->create();
    
    $employeeNos = $employees->pluck('employee_no');
    expect($employeeNos->unique())->toHaveCount(5);
});

test('factory withUser creates associated user', function () {
    $employee = Employee::factory()->withUser()->create();
    
    expect($employee->user)->not->toBeNull()
        ->and($employee->full_name)->toBe($employee->user->name);
});

test('factory regular sets correct employment type', function () {
    $employee = Employee::factory()->regular()->create();
    
    expect($employee->employment_type)->toBe(EmploymentType::REGULAR)
        ->and($employee->date_regularized)->not->toBeNull();
});
```

### Filament Resource Tests (Pest v4)
```php
// Tests/Filament/EmployeeResourceTest.php
use App\Filament\Resources\EmployeeResource;
use App\Models\Employee;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Livewire\Livewire;

uses(RefreshDatabase::class);

test('employee resource can list employees', function () {
    Employee::factory()->count(10)->create();
    
    Livewire::test(EmployeeResource\Pages\ListEmployees::class)
        ->assertCanSeeTableRecords(Employee::all(), 10);
});

test('employee resource can create employee', function () {
    Livewire::test(EmployeeResource\Pages\CreateEmployee::class)
        ->fillForm([
            'first_name' => 'John',
            'last_name' => 'Doe',
            'email' => 'john@example.com',
        ])
        ->call('create')
        ->assertHasNoErrors();
    
    expect(Employee::where('email', 'john@example.com')->exists())->toBeTrue();
});

test('employee resource can edit employee', function () {
    $employee = Employee::factory()->create();
    
    Livewire::test(EmployeeResource\Pages\EditEmployee::class, ['record' => $employee->id])
        ->fillForm([
            'first_name' => 'Jane',
        ])
        ->call('save')
        ->assertHasNoErrors();
    
    expect($employee->refresh()->first_name)->toBe('Jane');
});
```

### Service Layer Tests (Pest v4)
```php
// Tests/Unit/Services/EmployeeServiceTest.php
use App\Services\EmployeeService;
use App\Models\Employee;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('employee service can create employee with unique number', function () {
    $service = new EmployeeService();
    
    $employee = $service->createEmployee([
        'first_name' => 'John',
        'last_name' => 'Doe',
        'email' => 'john@example.com',
    ]);
    
    expect($employee->employee_no)->not->toBeNull()
        ->and($employee->employee_no)->toMatch('/EMP-\d{8}-\d{3}/');
});

test('employee service handles duplicate email', function () {
    $service = new EmployeeService();
    
    Employee::factory()->create(['email' => 'john@example.com']);
    
    expect(fn () => $service->createEmployee([
        'first_name' => 'Jane',
        'last_name' => 'Doe',
        'email' => 'john@example.com',
    ]))->toThrow(\Illuminate\Validation\ValidationException::class);
});
```

### Feature Tests (Pest v4)
```php
// Tests/Feature/EmployeeApiTest.php
use App\Models\Employee;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('authenticated user can access employee api', function () {
    $user = User::factory()->create();
    Employee::factory()->count(5)->create();
    
    $response = $this->actingAs($user)
        ->getJson('/api/employees');
    
    $response->assertStatus(200)
        ->assertJsonCount(5, 'data');
});

test('unauthenticated user cannot access employee api', function () {
    $response = $this->getJson('/api/employees');
    
    $response->assertStatus(401);
});

test('employee api returns paginated results', function () {
    $user = User::factory()->create();
    Employee::factory()->count(25)->create();
    
    $response = $this->actingAs($user)
        ->getJson('/api/employees?page=1&per_page=10');
    
    $response->assertStatus(200)
        ->assertJsonPath('meta.current_page', 1)
        ->assertJsonPath('meta.per_page', 10)
        ->assertJsonPath('meta.total', 25);
});
```

### Data Provider Tests (Pest v4)
```php
// Tests/Unit/EnumDataProviderTest.php
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;

dataset('employment_statuses', [
    EmploymentStatus::ACTIVE,
    EmploymentStatus::INACTIVE,
    EmploymentStatus::ON_LEAVE,
    EmploymentStatus::RESIGNED,
]);

dataset('employment_types', [
    EmploymentType::REGULAR,
    EmploymentType::PROBATIONARY,
    EmploymentType::CONTRACTUAL,
    EmploymentType::INTERN,
]);

test('employment status enums have valid labels', function (EmploymentStatus $status) {
    expect($status->getLabel())->not->toBeEmpty()
        ->and($status->getColor())->not->toBeEmpty();
})->with('employment_statuses');

test('employment type enums have valid labels', function (EmploymentType $type) {
    expect($type->getLabel())->not->toBeEmpty()
        ->and($type->getDescription())->not->toBeEmpty();
})->with('employment_types');
```

### Authorization Tests (Pest v4)
```php
// Tests/Feature/EmployeeAuthorizationTest.php
use App\Models\Employee;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('admin can view all employees', function () {
    $admin = User::factory()->create();
    $admin->assignRole('admin');
    
    Employee::factory()->count(10)->create();
    
    $response = $this->actingAs($admin)
        ->get('/admin/employees');
    
    $response->assertStatus(200);
});

test('regular user cannot view all employees', function () {
    $user = User::factory()->create();
    $user->assignRole('employee');
    
    $response = $this->actingAs($user)
        ->get('/admin/employees');
    
    $response->assertStatus(403);
});

test('employee can view own profile in portal', function () {
    $user = User::factory()->create();
    $employee = Employee::factory()->create(['user_id' => $user->id]);
    
    $response = $this->actingAs($user)
        ->get('/portal/profile');
    
    $response->assertStatus(200);
});

test('employee cannot view other employee profile', function () {
    $user = User::factory()->create();
    $otherEmployee = Employee::factory()->create();
    
    $response = $this->actingAs($user)
        ->get("/portal/employees/{$otherEmployee->id}");
    
    $response->assertStatus(403);
});
```

---

## 🔒 Security Implementation Requirements

### Input Validation & Sanitization

#### Form Request Validation Template
```php
// Requests/StoreEmployeeRequest.php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreEmployeeRequest extends FormRequest
{
    public function authorize(): bool
    {
        return auth()->user()->can('create', \App\Models\Employee::class);
    }

    public function rules(): array
    {
        return [
            'first_name' => 'required|string|max:255|regex:/^[a-zA-Z\s\-\'\.]+$/',
            'last_name' => 'required|string|max:255|regex:/^[a-zA-Z\s\-\'\.]+$/',
            'email' => 'required|email|max:255|unique:employees,email',
            'mobile_no' => 'nullable|string|max:20|regex:/^[0-9\+\-\(\)\s]+$/',
            'birthdate' => 'nullable|date|before:today',
            'gender' => ['required', Rule::enum(\App\Enums\Gender::class)],
            'civil_status' => ['required', Rule::enum(\App\Enums\CivilStatus::class)],
            'employment_type' => ['required', Rule::enum(\App\Enums\EmploymentType::class)],
            'employment_status' => ['required', Rule::enum(\App\Enums\EmploymentStatus::class)],
            'department_id' => 'required|exists:departments,id',
            'position_id' => 'required|exists:positions,id',
            'date_hired' => 'required|date|before_or_equal:today',
        ];
    }

    public function sanitize(): array
    {
        return array_map(function ($value) {
            if (is_string($value)) {
                return trim(strip_tags($value));
            }
            return $value;
        }, $this->all());
    }
}
```

#### File Upload Security
```php
// Services/FileUploadService.php
namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;

class FileUploadService
{
    public function uploadDocument(UploadedFile $file, string $directory): string
    {
        // Validate file type
        $allowedMimes = ['application/pdf', 'image/jpeg', 'image/png', 'application/msword'];
        if (!in_array($file->getMimeType(), $allowedMimes)) {
            throw new \InvalidArgumentException('Invalid file type');
        }

        // Validate file size (max 10MB)
        if ($file->getSize() > 10 * 1024 * 1024) {
            throw new \InvalidArgumentException('File too large');
        }

        // Sanitize filename
        $filename = Str::slug(pathinfo($file->getClientOriginalName(), PATHINFO_FILENAME)) 
                    . '-' . Str::random(8) 
                    . '.' . $file->getClientOriginalExtension();

        // Store in S3
        $path = $file->storeAs($directory, $filename, 's3');

        return $path;
    }

    public function validateResume(UploadedFile $file): bool
    {
        // Additional validation for resumes
        $allowedMimes = ['application/pdf', 'application/msword', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'];
        
        if (!in_array($file->getMimeType(), $allowedMimes)) {
            return false;
        }

        // Scan for malware (if scanning service available)
        // $this->scanForMalware($file);

        return true;
    }
}
```

### Data Encryption Implementation

#### Encrypted Model Attributes
```php
// Models/Employee.php (Enhanced)
namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasAttributes;
use Illuminate\Database\Eloquent\Model;

class Employee extends Model
{
    use SoftDeletes;

    protected $fillable = [
        // ... other fields
        'sss_no',
        'philhealth_no',
        'pagibig_no',
        'tin_no',
        'bank_name',
        'bank_account_number',
        'bank_account_name',
    ];

    protected $casts = [
        // ... other casts
        'sss_no' => 'encrypted',
        'philhealth_no' => 'encrypted',
        'pagibig_no' => 'encrypted',
        'tin_no' => 'encrypted',
        'bank_account_number' => 'encrypted',
        'bank_account_name' => 'encrypted',
    ];

    protected $hidden = [
        'sss_no',
        'philhealth_no',
        'pagibig_no',
        'tin_no',
        'bank_account_number',
        'bank_account_name',
    ];

    public function getMaskedSssNoAttribute(): string
    {
        if (!$this->sss_no) return '';
        return '***-**-****-' . substr($this->sss_no, -1);
    }

    public function getMaskedBankAccountAttribute(): string
    {
        if (!$this->bank_account_number) return '';
        return '****' . substr($this->bank_account_number, -4);
    }
}
```

#### Encryption Database Configuration
```php
// Database/migrations/xxxx_create_employees_table.php
Schema::create('employees', function (Blueprint $table) {
    // ... other columns
    
    // Encrypted columns must be TEXT or larger
    $table->text('sss_no')->nullable();
    $table->text('philhealth_no')->nullable();
    $table->text('pagibig_no')->nullable();
    $table->text('tin_no')->nullable();
    $table->text('bank_account_number')->nullable();
    $table->text('bank_account_name')->nullable();
});
```

### PII Masking in Logs

#### Log Sanitization Middleware
```php
// Middleware/SanitizeLogs.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class SanitizeLogs
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        // Sanitize request data before logging
        if ($request->hasAny(['password', 'sss_no', 'bank_account_number', 'token'])) {
            $sanitized = $request->except(['password', 'sss_no', 'bank_account_number', 'token']);
            Log::info('Sanitized request', ['data' => $sanitized]);
        }

        return $response;
    }
}
```

#### Audit Log PII Masking
```php
// Models/AuditLog.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class AuditLog extends Model
{
    protected $fillable = [
        'user_id',
        'action',
        'model_type',
        'model_id',
        'old_values',
        'new_values',
        'ip_address',
        'user_agent',
    ];

    protected $casts = [
        'old_values' => 'array',
        'new_values' => 'array',
    ];

    protected static function boot()
    {
        parent::boot();

        static::creating(function ($log) {
            // Mask sensitive fields in old_values and new_values
            $sensitiveFields = ['password', 'sss_no', 'philhealth_no', 'pagibig_no', 'tin_no', 'bank_account_number'];
            
            foreach (['old_values', 'new_values'] as $field) {
                if (isset($log->$field) && is_array($log->$field)) {
                    foreach ($sensitiveFields as $sensitiveField) {
                        if (isset($log->$field[$sensitiveField])) {
                            $log->$field[$sensitiveField] = '***MASKED***';
                        }
                    }
                }
            }
        });
    }
}
```

### API Response Field Control

#### API Resource with Field Visibility
```php
// Http/Resources/EmployeeResource.php
namespace App\Http\Resources;

use App\Enums\UserRole;
use Illuminate\Http\Resources\Json\JsonResource;
use Illuminate\Support\Facades\Auth;

class EmployeeResource extends JsonResource
{
    public function toArray($request): array
    {
        $user = Auth::user();
        $isOwner = $this->user_id === $user->id;
        $isAdminOrHr = in_array($user->role, [UserRole::COMPANY_ADMIN, UserRole::HR_ADMIN, UserRole::HR_STAFF]);
        $isPayrollOfficer = $user->role === UserRole::PAYROLL_OFFICER;

        $data = [
            'id' => $this->hashid,
            'employee_no' => $this->employee_no,
            'full_name' => $this->full_name,
            'email' => $this->email,
            'department' => $this->department->name ?? null,
            'position' => $this->position->title ?? null,
            'employment_type' => $this->employment_type->getLabel(),
            'employment_status' => $this->employment_status->getLabel(),
        ];

        // Add sensitive fields only if authorized
        if ($isAdminOrHr || $isOwner) {
            $data['mobile_no'] = $this->mobile_no;
            $data['address'] = $this->address_line;
        }

        // Add compensation data only if authorized
        if ($isAdminOrHr || $isPayrollOfficer || $isOwner) {
            $data['masked_sss_no'] = $this->masked_sss_no;
            $data['masked_bank_account'] = $this->masked_bank_account;
        }

        // Add full sensitive data only for admins and HR
        if ($isAdminOrHr) {
            $data['sss_no'] = $this->sss_no;
            $data['bank_account_number'] = $this->bank_account_number;
        }

        return $data;
    }
}
```

### XSS & CSRF Protection

#### XSS Protection Configuration
```php
// config/app.php
return [
    'security' => [
        'xss_protection' => env('XSS_PROTECTION', '1; mode=block'),
        'content_security_policy' => env('CSP_ENABLED', true),
    ],
];
```

#### Content Security Policy Middleware
```php
// Middleware/ContentSecurityPolicy.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class ContentSecurityPolicy
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        $response->headers->set('Content-Security-Policy', 
            "default-src 'self'; " .
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " .
            "style-src 'self' 'unsafe-inline'; " .
            "img-src 'self' data: https:; " .
            "font-src 'self' data:; " .
            "connect-src 'self'; " .
            "frame-ancestors 'self';"
        );

        return $response;
    }
}
```

#### CSRF Protection (Laravel 13)
```php
// Middleware/PreventRequestForgery.php (Laravel 13 default)
namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\PreventRequestForgery as Middleware;

class PreventRequestForgery extends Middleware
{
    protected $except = [
        'api/*', // Exclude API routes if using token auth
        'webhooks/*', // Exclude webhook endpoints
    ];
}
```

### Session Security

#### Session Configuration
```php
// config/session.php
return [
    'driver' => env('SESSION_DRIVER', 'redis'),
    'lifetime' => env('SESSION_LIFETIME', 120),
    'expire_on_close' => true,
    'encrypt' => true,
    'files' => null,
    'connection' => null,
    'table' => 'sessions',
    'store' => null,
    'lottery' => [2, 100],
    'cookie' => env('SESSION_COOKIE_NAME', 'hris_session'),
    'path' => '/',
    'domain' => env('SESSION_DOMAIN'),
    'secure' => env('SESSION_SECURE_COOKIE', true),
    'http_only' => true,
    'same_site' => 'lax',
];
```

---

## 🔐 Laravel Policies (Authorization)

### Policy Structure Overview

Laravel Policies provide fine-grained authorization for specific model actions, using the App\Enums\UserRole enum for role-based access control.

### Base Policy Template
```php
// Policies/BasePolicy.php
namespace App\Policies;

use App\Enums\UserRole;
use Illuminate\Auth\Access\HandlesAuthorization;

abstract class BasePolicy
{
    use HandlesAuthorization;

    protected function isAdminOrHrAdmin($user): bool
    {
        return in_array($user->role, [UserRole::COMPANY_ADMIN, UserRole::HR_ADMIN]);
    }

    protected function isHrStaff($user): bool
    {
        return $user->role === UserRole::HR_STAFF;
    }

    protected function isManager($user): bool
    {
        return $user->role === UserRole::MANAGER;
    }

    protected function isEmployee($user): bool
    {
        return $user->role === UserRole::EMPLOYEE;
    }

    protected function isRecordOwner($user, $record): bool
    {
        return $record->user_id === $user->id || 
               ($record instanceof \App\Models\Employee && $record->user_id === $user->id);
    }

    protected function isDepartmentManager($user, $record): bool
    {
        if (!$this->isManager($user)) {
            return false;
        }

        $employee = $user->employee;
        if (!$employee) {
            return false;
        }

        if ($record instanceof \App\Models\Employee) {
            return $record->department_id === $employee->department_id;
        }

        if (isset($record->department_id)) {
            return $record->department_id === $employee->department_id;
        }

        return false;
    }
}
```

### Employee Policy
```php
// Policies/EmployeePolicy.php
namespace App\Policies;

use App\Models\Employee;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class EmployeePolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $this->isHrStaff($user) || 
               $this->isManager($user);
    }

    public function view(User $user, Employee $employee): bool
    {
        // Admins and HR can view all employees
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can view employees in their department
        if ($this->isDepartmentManager($user, $employee)) {
            return true;
        }

        // Employees can only view their own record
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $employee)) {
            return true;
        }

        return false;
    }

    public function create(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function update(User $user, Employee $employee): bool
    {
        // Admins and HR can update all employees
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can update limited information for department employees
        if ($this->isDepartmentManager($user, $employee)) {
            return true;
        }

        // Employees can update their own profile
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $employee)) {
            return true;
        }

        return false;
    }

    public function delete(User $user, Employee $employee): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function restore(User $user, Employee $employee): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function forceDelete(User $user, Employee $employee): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function viewSensitiveData(User $user, Employee $employee): bool
    {
        // Only admins and HR can view sensitive data (SSS, bank info, etc.)
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function viewCompensation(User $user, Employee $employee): bool
    {
        // Admins, HR, and payroll officers can view compensation
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        if ($user->role === UserRole::PAYROLL_OFFICER) {
            return true;
        }

        // Employees can view their own compensation
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $employee)) {
            return true;
        }

        return false;
    }
}
```

### Leave Request Policy
```php
// Policies/LeaveRequestPolicy.php
namespace App\Policies;

use App\Models\LeaveRequest;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class LeaveRequestPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $this->isHrStaff($user) || 
               $this->isManager($user);
    }

    public function view(User $user, LeaveRequest $leaveRequest): bool
    {
        // Admins and HR can view all leave requests
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can view department leave requests
        if ($this->isDepartmentManager($user, $leaveRequest->employee)) {
            return true;
        }

        // Employees can view their own leave requests
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $leaveRequest->employee)) {
            return true;
        }

        return false;
    }

    public function create(User $user): bool
    {
        // Employees can create leave requests
        if ($this->isEmployee($user)) {
            return true;
        }

        // HR can create leave requests on behalf of employees
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function update(User $user, LeaveRequest $leaveRequest): bool
    {
        // HR can update leave requests
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Employees can update their own pending requests
        if ($this->isEmployee($user) && 
            $this->isRecordOwner($user, $leaveRequest->employee) &&
            $leaveRequest->status === \App\Enums\LeaveRequestStatus::PENDING) {
            return true;
        }

        return false;
    }

    public function delete(User $user, LeaveRequest $leaveRequest): bool
    {
        // HR can delete leave requests
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Employees can delete their own pending requests
        if ($this->isEmployee($user) && 
            $this->isRecordOwner($user, $leaveRequest->employee) &&
            $leaveRequest->status === \App\Enums\LeaveRequestStatus::PENDING) {
            return true;
        }

        return false;
    }

    public function approve(User $user, LeaveRequest $leaveRequest): bool
    {
        // Managers can approve department leave requests
        if ($this->isDepartmentManager($user, $leaveRequest->employee)) {
            return true;
        }

        // HR can approve any leave request
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function reject(User $user, LeaveRequest $leaveRequest): bool
    {
        return $this->approve($user, $leaveRequest);
    }
}
```

### Payroll Policy
```php
// Policies/PayrollRunPolicy.php
namespace App\Policies;

use App\Enums\UserRole;
use App\Models\PayrollRun;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class PayrollRunPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) ||
               $user->role === UserRole::PAYROLL_OFFICER;
    }

    public function view(User $user, PayrollRun $payrollRun): bool
    {
        return $this->isAdminOrHrAdmin($user) ||
               $user->role === UserRole::PAYROLL_OFFICER;
    }

    public function create(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) ||
               $user->role === UserRole::PAYROLL_OFFICER;
    }

    public function update(User $user, PayrollRun $payrollRun): bool
    {
        return $this->isAdminOrHrAdmin($user) ||
               $user->role === UserRole::PAYROLL_OFFICER;
    }

    public function delete(User $user, PayrollRun $payrollRun): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function process(User $user, PayrollRun $payrollRun): bool
    {
        // Only payroll officers can process payroll
        return $user->role === UserRole::PAYROLL_OFFICER;
    }

    public function finalize(User $user, PayrollRun $payrollRun): bool
    {
        // Only admins can finalize payroll
        return $this->isAdminOrHrAdmin($user);
    }

    public function viewAllPayrollItems(User $user, PayrollRun $payrollRun): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $user->hasRole('payroll-officer');
    }

    public function viewOwnPayrollItems(User $user, PayrollRun $payrollRun): bool
    {
        // Employees can view their own payroll items
        if ($this->isEmployee($user) && $user->employee) {
            return $payrollRun->payrollItems()->where('employee_id', $user->employee->id)->exists();
        }

        return false;
    }
}
```

### Attendance Policy
```php
// Policies/AttendanceRecordPolicy.php
namespace App\Policies;

use App\Models\AttendanceRecord;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class AttendanceRecordPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $this->isHrStaff($user) || 
               $this->isManager($user);
    }

    public function view(User $user, AttendanceRecord $attendanceRecord): bool
    {
        // Admins and HR can view all attendance records
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can view department attendance
        if ($this->isDepartmentManager($user, $attendanceRecord->employee)) {
            return true;
        }

        // Employees can view their own attendance
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $attendanceRecord->employee)) {
            return true;
        }

        return false;
    }

    public function create(User $user): bool
    {
        // HR can create manual attendance records
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function update(User $user, AttendanceRecord $attendanceRecord): bool
    {
        // HR can update attendance records
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function delete(User $user, AttendanceRecord $attendanceRecord): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function adjust(User $user, AttendanceRecord $attendanceRecord): bool
    {
        // HR can adjust attendance records
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function requestCorrection(User $user, AttendanceRecord $attendanceRecord): bool
    {
        // Employees can request corrections to their own attendance
        if ($this->isEmployee($user) && $this->isRecordOwner($user, $attendanceRecord->employee)) {
            return true;
        }

        return false;
    }

    public function approveCorrection(User $user, AttendanceRecord $attendanceRecord): bool
    {
        // Managers can approve corrections for department employees
        if ($this->isDepartmentManager($user, $attendanceRecord->employee)) {
            return true;
        }

        // HR can approve any corrections
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }
}
```

### Department Policy
```php
// Policies/DepartmentPolicy.php
namespace App\Policies;

use App\Models\Department;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class DepartmentPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $this->isHrStaff($user) || 
               $this->isManager($user);
    }

    public function view(User $user, Department $department): bool
    {
        // Admins and HR can view all departments
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can view their department
        if ($this->isManager($user) && $user->employee) {
            return $user->employee->department_id === $department->id;
        }

        return false;
    }

    public function create(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function update(User $user, Department $department): bool
    {
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }

    public function delete(User $user, Department $department): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function manageEmployees(User $user, Department $department): bool
    {
        // Department head can manage employees
        if ($this->isManager($user) && $user->employee) {
            return $user->employee->department_id === $department->id &&
                   $department->head_employee_id === $user->employee->id;
        }

        // HR can manage employees in any department
        return $this->isAdminOrHrAdmin($user) || $this->isHrStaff($user);
    }
}
```

### Branch Policy
```php
// Policies/BranchPolicy.php
namespace App\Policies;

use App\Models\Branch;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class BranchPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user) || 
               $this->isHrStaff($user) || 
               $this->isManager($user);
    }

    public function view(User $user, Branch $branch): bool
    {
        // Admins and HR can view all branches
        if ($this->isAdminOrHrAdmin($user) || $this->isHrStaff($user)) {
            return true;
        }

        // Managers can view their branch
        if ($this->isManager($user) && $user->employee) {
            return $user->employee->branch_id === $branch->id;
        }

        return false;
    }

    public function create(User $user): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function update(User $user, Branch $branch): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function delete(User $user, Branch $branch): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }

    public function manageGeofence(User $user, Branch $branch): bool
    {
        return $this->isAdminOrHrAdmin($user);
    }
}
```

### Policy Registration
```php
// Providers/AuthServiceProvider.php
namespace App\Providers;

use App\Models\AttendanceRecord;
use App\Models\Branch;
use App\Models\Department;
use App\Models\Employee;
use App\Models\LeaveRequest;
use App\Models\PayrollRun;
use App\Policies\AttendanceRecordPolicy;
use App\Policies\BranchPolicy;
use App\Policies\DepartmentPolicy;
use App\Policies\EmployeePolicy;
use App\Policies\LeaveRequestPolicy;
use App\Policies\PayrollRunPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        Employee::class => EmployeePolicy::class,
        LeaveRequest::class => LeaveRequestPolicy::class,
        PayrollRun::class => PayrollRunPolicy::class,
        AttendanceRecord::class => AttendanceRecordPolicy::class,
        Department::class => DepartmentPolicy::class,
        Branch::class => BranchPolicy::class,
    ];

    public function boot(): void
    {
        $this->registerPolicies();
    }
}
```

### Policy Usage in Controllers
```php
// Controllers/EmployeeController.php
namespace App\Http\Controllers;

use App\Models\Employee;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class EmployeeController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        $this->authorize('viewAny', Employee::class);
        
        $employees = Employee::with(['department', 'position'])
            ->when($request->has('department_id'), fn($q) => $q->where('department_id', $request->department_id))
            ->paginate(15);
            
        return response()->json($employees);
    }

    public function show(Employee $employee): JsonResponse
    {
        $this->authorize('view', $employee);
        
        $employee->load(['department', 'position', 'assignments']);
        
        // Only include sensitive data if authorized
        if (auth()->user()->can('viewSensitiveData', $employee)) {
            $employee->makeVisible(['sss_no', 'philhealth_no', 'pagibig_no', 'tin_no', 'bank_account_number']);
        }
        
        return response()->json($employee);
    }

    public function update(Request $request, Employee $employee): JsonResponse
    {
        $this->authorize('update', $employee);
        
        $validated = $request->validate([
            'first_name' => 'required|string|max:255',
            'last_name' => 'required|string|max:255',
            'email' => 'required|email|unique:employees,email,' . $employee->id,
        ]);
        
        $employee->update($validated);
        
        return response()->json($employee);
    }
}
```

### Policy Usage in Filament Resources
```php
// Filament/Resources/EmployeeResource.php
namespace App\Filament\Resources;

use App\Models\Employee;
use Filament\Resources\Resource;
use Filament\Resources\Pages\PageRegistration;

class EmployeeResource extends Resource
{
    protected static ?string $model = Employee::class;

    public static function canViewAny(): bool
    {
        return auth()->user()->can('viewAny', Employee::class);
    }

    public static function canCreate(): bool
    {
        return auth()->user()->can('create', Employee::class);
    }

    public static function canEdit($record): bool
    {
        return auth()->user()->can('update', $record);
    }

    public static function canDelete($record): bool
    {
        return auth()->user()->can('delete', $record);
    }

    public static function canViewAny($record): bool
    {
        return auth()->user()->can('viewAny', Employee::class);
    }
}
```

### Policy Tests (Pest v4)
```php
// Tests/Unit/Policies/EmployeePolicyTest.php
use App\Models\Employee;
use App\Models\User;
use App\Policies\EmployeePolicy;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('admin can view any employee', function () {
    $admin = User::factory()->create();
    $admin->assignRole('company-admin');
    
    $policy = new EmployeePolicy();
    expect($policy->viewAny($admin))->toBeTrue();
});

test('employee can only view own record', function () {
    $employee = User::factory()->create();
    $employee->assignRole('employee');
    $employeeRecord = Employee::factory()->create(['user_id' => $employee->id]);
    
    $otherEmployee = Employee::factory()->create();
    
    $policy = new EmployeePolicy();
    expect($policy->view($employee, $employeeRecord))->toBeTrue()
        ->and($policy->view($employee, $otherEmployee))->toBeFalse();
});

test('manager can view department employees', function () {
    $manager = User::factory()->create();
    $manager->assignRole('manager');
    $managerRecord = Employee::factory()->create(['user_id' => $manager->id, 'department_id' => 1]);
    
    $departmentEmployee = Employee::factory()->create(['department_id' => 1]);
    $otherEmployee = Employee::factory()->create(['department_id' => 2]);
    
    $policy = new EmployeePolicy();
    expect($policy->view($manager, $departmentEmployee))->toBeTrue()
        ->and($policy->view($manager, $otherEmployee))->toBeFalse();
});

test('only admin can delete employees', function () {
    $admin = User::factory()->create();
    $admin->assignRole('company-admin');
    
    $hrStaff = User::factory()->create();
    $hrStaff->assignRole('hr-staff');
    
    $employee = Employee::factory()->create();
    
    $policy = new EmployeePolicy();
    expect($policy->delete($admin, $employee))->toBeTrue()
        ->and($policy->delete($hrStaff, $employee))->toBeFalse();
});
```

### Security Compliance

The policy system fulfills the security architecture requirements:

✅ **Comprehensive Laravel Policies for all sensitive domains**
- Employee policy with sensitive data protection
- Leave request policy with approval workflows
- Payroll policy with processing restrictions
- Attendance policy with correction workflows
- Department/Branch policies with hierarchical access

✅ **Filament panel middleware and role checks**
- Panel-level role restrictions
- Resource-level policy integration
- Action-level authorization

✅ **Department/location-based manager access controls**
- Department manager access validation
- Branch-level restrictions
- Hierarchical data access

✅ **API Resource classes with controlled fields**
- Sensitive data visibility checks
- Role-based field exposure
- Authorization middleware integration

---

## 🤖 Laravel AI Integration for ATS

### Installation & Configuration

#### Package Installation
```bash
composer require laravel/ai
php artisan vendor:publish --provider="Laravel\Ai\AiServiceProvider" --tag="ai-config"
```

#### Environment Configuration
```env
# AI Provider Configuration
AI_DEFAULT_PROVIDER=openai
AI_DEFAULT_IMAGE_PROVIDER=gemini
AI_DEFAULT_EMBEDDING_PROVIDER=openai

# API Keys
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
GEMINI_API_KEY=your-gemini-key

# Vector Storage
AI_VECTOR_STORE_DRIVER=database
CACHE_STORE=database
```

#### AI Configuration File
```php
// config/ai.php
return [
    'default' => env('AI_DEFAULT_PROVIDER', 'openai'),
    'default_for_images' => env('AI_DEFAULT_IMAGE_PROVIDER', 'gemini'),
    'default_for_audio' => env('AI_DEFAULT_AUDIO_PROVIDER', 'openai'),
    'default_for_transcription' => env('AI_DEFAULT_TRANSCRIPTION_PROVIDER', 'openai'),
    'default_for_embeddings' => env('AI_DEFAULT_EMBEDDING_PROVIDER', 'openai'),
    'default_for_reranking' => env('AI_DEFAULT_RERANKING_PROVIDER', 'cohere'),
    
    'providers' => [
        'openai' => [
            'key' => env('OPENAI_API_KEY'),
            'url' => env('OPENAI_URL', 'https://api.openai.com/v1'),
        ],
        'anthropic' => [
            'key' => env('ANTHROPIC_API_KEY'),
        ],
        'gemini' => [
            'key' => env('GEMINI_API_KEY'),
        ],
    ],
];
```

### ATS AI Services

#### Resume Parsing Service
```php
// Services/AI/ResumeParsingService.php
namespace App\Services\AI;

use Laravel\Ai\Contracts\GeneratesText;
use Laravel\Ai\Facades\Ai;
use Illuminate\Support\Facades\Storage;

class ResumeParsingService
{
    public function parseResume(string $filePath): array
    {
        $resumeText = $this->extractTextFromResume($filePath);
        
        $response = Ai::text()
            ->prompt("Extract structured information from this resume in JSON format with these fields: 
                name, email, phone, experience (years, roles, companies), 
                education (degrees, institutions, years), skills (technical and soft),
                certifications, languages, availability, expected_salary range.
                
                Resume text: {$resumeText}")
            ->asJson()
            ->generate();

        return $response->text;
    }

    private function extractTextFromResume(string $filePath): string
    {
        // Use Laravel AI Document processing
        $document = \Laravel\Ai\Files\Document::fromPath($filePath);
        $text = $document->getText();
        
        return $text;
    }

    public function validateCandidateProfile(array $parsedData, array $jobRequirements): array
    {
        $response = Ai::text()
            ->prompt("Analyze this candidate profile against job requirements and provide:
                1. Match percentage (0-100)
                2. Skills gap analysis
                3. Experience relevance
                4. Education match
                5. Overall recommendation (highly_recommended, recommended, consider, not_recommended)
                
                Candidate: " . json_encode($parsedData) . "
                Requirements: " . json_encode($jobRequirements))
            ->asJson()
            ->generate();

        return $response->text;
    }
}
```

#### Candidate Matching Service
```php
// Services/AI/CandidateMatchingService.php
namespace App\Services\AI;

use Laravel\Ai\Facades\Ai;
use Laravel\Ai\Embeddings;
use App\Models\JobApplication;
use App\Models\JobRequisition;

class CandidateMatchingService
{
    public function matchCandidates(JobRequisition $jobRequisition): array
    {
        // Generate embedding for job requirements
        $jobEmbedding = $this->generateJobEmbedding($jobRequisition);
        
        // Get all applications for this requisition
        $applications = $jobRequisition->applications;
        
        $matches = [];
        
        foreach ($applications as $application) {
            // Generate embedding for candidate resume
            $candidateEmbedding = $this->generateCandidateEmbedding($application);
            
            // Calculate similarity
            $similarity = $this->calculateCosineSimilarity($jobEmbedding, $candidateEmbedding);
            
            $matches[] = [
                'application_id' => $application->id,
                'candidate_name' => $application->applicant_name,
                'similarity_score' => $similarity,
                'match_quality' => $this->getMatchQuality($similarity),
            ];
        }
        
        // Sort by similarity score
        usort($matches, fn($a, $b) => $b['similarity_score'] <=> $a['similarity_score']);
        
        return $matches;
    }

    private function generateJobEmbedding(JobRequisition $jobRequisition): array
    {
        $jobText = $this->prepareJobText($jobRequisition);
        
        $response = Embeddings::for([$jobText])->generate();
        
        return $response->embeddings[0];
    }

    private function generateCandidateEmbedding(JobApplication $application): array
    {
        $candidateText = $this->prepareCandidateText($application);
        
        $response = Embeddings::for([$candidateText])->generate();
        
        return $response->embeddings[0];
    }

    private function prepareJobText(JobRequisition $jobRequisition): string
    {
        return sprintf(
            "Position: %s\nDescription: %s\nRequirements: %s\nSkills: %s\nExperience Level: %s",
            $jobRequisition->title,
            $jobRequisition->description,
            $jobRequisition->requirements,
            $jobRequisition->skills ?? '',
            $jobRequisition->experience_level ?? ''
        );
    }

    private function prepareCandidateText(JobApplication $application): string
    {
        $resumeText = $application->resume_text ?? $this->extractResumeText($application);
        
        return sprintf(
            "Name: %s\nExperience: %s\nEducation: %s\nSkills: %s\nResume: %s",
            $application->applicant_name,
            $application->experience_summary ?? '',
            $application->education_summary ?? '',
            $application->skills_summary ?? '',
            $resumeText
        );
    }

    private function calculateCosineSimilarity(array $vector1, array $vector2): float
    {
        $dotProduct = 0;
        $magnitude1 = 0;
        $magnitude2 = 0;

        for ($i = 0; $i < count($vector1); $i++) {
            $dotProduct += $vector1[$i] * $vector2[$i];
            $magnitude1 += $vector1[$i] ** 2;
            $magnitude2 += $vector2[$i] ** 2;
        }

        $magnitude1 = sqrt($magnitude1);
        $magnitude2 = sqrt($magnitude2);

        if ($magnitude1 === 0 || $magnitude2 === 0) {
            return 0;
        }

        return $dotProduct / ($magnitude1 * $magnitude2);
    }

    private function getMatchQuality(float $similarity): string
    {
        if ($similarity >= 0.8) return 'excellent';
        if ($similarity >= 0.6) return 'good';
        if ($similarity >= 0.4) return 'fair';
        return 'poor';
    }
}
```

#### Candidate Screening Agent
```php
// Services/AI/CandidateScreeningAgent.php
namespace App\Services\AI;

use Laravel\Ai\Agent;
use Laravel\Ai\Contracts\GeneratesText;
use Laravel\Ai\Traits\HasStructuredOutput;
use Laravel\Ai\Traits\RemembersConversations;

class CandidateScreeningAgent extends Agent
{
    use HasStructuredOutput, RemembersConversations;

    protected $systemPrompt = "You are an expert HR screening assistant. 
        Your role is to evaluate job applications based on company standards, 
        job requirements, and candidate qualifications. Provide fair, unbiased assessments.";

    public function screenCandidate(array $candidateData, array $jobRequirements): array
    {
        return $this->prompt(
            "Screen this candidate for the position based on these requirements:
            
            Job Requirements: " . json_encode($jobRequirements) . "
            Candidate Data: " . json_encode($candidateData) . "
            
            Provide a structured assessment with:
            1. Overall rating (1-10)
            2. Skills match (percentage)
            3. Experience relevance (high/medium/low)
            4. Education fit (yes/no)
            5. Red flags (if any)
            6. Interview recommendation (yes/no/maybe)
            7. Key strengths
            8. Areas for improvement
            9. Interview questions to ask"
        )->asJson()->generate()->text;
    }

    public function generateInterviewQuestions(array $candidateData, array $jobRequirements): array
    {
        return $this->prompt(
            "Generate 5-7 interview questions for this candidate based on the job requirements.
            Include questions about technical skills, experience, behavioral scenarios, and role-specific challenges.
            
            Candidate: " . json_encode($candidateData) . "
            Requirements: " . json_encode($jobRequirements)
        )->asJson()->generate()->text;
    }
}
```

#### Skills Extraction Service
```php
// Services/AI/SkillsExtractionService.php
namespace App\Services\AI;

use Laravel\Ai\Facades\Ai;

class SkillsExtractionService
{
    public function extractSkills(string $resumeText): array
    {
        $response = Ai::text()
            ->prompt("Extract all skills from this resume and categorize them:
                1. Technical skills (programming languages, frameworks, tools)
                2. Soft skills (communication, leadership, teamwork)
                3. Industry knowledge
                4. Certifications
                5. Languages
                
                Resume: {$resumeText}")
            ->asJson()
            ->generate();

        return $response->text;
    }

    public function assessSkillLevel(string $skill, string $experienceDescription): string
    {
        $response = Ai::text()
            ->prompt("Assess the proficiency level of '{$skill}' based on this experience description.
            Rate as: beginner, intermediate, advanced, expert.
            
            Experience: {$experienceDescription}")
            ->asJson()
            ->generate();

        return $response->text['level'] ?? 'unknown';
    }
}
```

### Vector Store Integration

#### Resume Vector Store Setup
```php
// Services/AI/ResumeVectorStore.php
namespace App\Services\AI;

use Laravel\Ai\Stores;
use Laravel\Ai\Files\Document;
use App\Models\JobApplication;

class ResumeVectorStore
{
    public function __construct()
    {
        $this->store = Stores::create('resumes');
    }

    public function addResume(JobApplication $application): void
    {
        if ($application->resume_path) {
            $document = Document::fromPath($application->resume_path);
            $file = $document->put();
            
            $this->store->add($file->id, [
                'application_id' => $application->id,
                'candidate_name' => $application->applicant_name,
                'position_applied' => $application->jobRequisition->title,
            ]);
        }
    }

    public function searchResumes(string $query, array $filters = []): array
    {
        $results = $this->store->search($query, $filters);
        
        return collect($results)->map(function ($result) {
            return [
                'application_id' => $result['metadata']['application_id'],
                'candidate_name' => $result['metadata']['candidate_name'],
                'relevance_score' => $result['score'],
            ];
        })->toArray();
    }

    public function findSimilarCandidates(JobApplication $referenceApplication): array
    {
        $referenceResume = Document::fromPath($referenceApplication->resume_path);
        
        $results = $this->store->similar($referenceResume, [
            'exclude_application_id' => $referenceApplication->id,
        ]);
        
        return collect($results)->map(function ($result) {
            return [
                'application_id' => $result['metadata']['application_id'],
                'candidate_name' => $result['metadata']['candidate_name'],
                'similarity_score' => $result['score'],
            ];
        })->toArray();
    }
}
```

### ATS AI Models

#### Job Application Model with AI Features
```php
// Models/JobApplication.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class JobApplication extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'job_requisition_id',
        'applicant_name',
        'email',
        'phone',
        'resume_path',
        'resume_text',
        'cover_letter_path',
        'skills_summary',
        'experience_summary',
        'education_summary',
        'ai_match_score',
        'ai_screening_result',
        'ai_interview_questions',
        'status',
        'applied_date',
    ];

    protected $casts = [
        'ai_match_score' => 'decimal:2',
        'ai_screening_result' => 'array',
        'ai_interview_questions' => 'array',
        'applied_date' => 'date',
    ];

    public function jobRequisition()
    {
        return $this->belongsTo(JobRequisition::class);
    }

    public function stages()
    {
        return $this->hasMany(ApplicationStage::class);
    }

    public function jobOffer()
    {
        return $this->hasOne(JobOffer::class);
    }

    public function updateAiMatchScore(float $score): void
    {
        $this->update(['ai_match_score' => $score]);
    }

    public function updateAiScreening(array $result): void
    {
        $this->update(['ai_screening_result' => $result]);
    }

    public function generateAiInterviewQuestions(): array
    {
        $service = new CandidateScreeningAgent();
        $questions = $service->generateInterviewQuestions(
            $this->toArray(),
            $this->jobRequisition->toArray()
        );
        
        $this->update(['ai_interview_questions' => $questions]);
        
        return $questions;
    }
}
```

#### Job Requisition Model with AI Enhancement
```php
// Models/JobRequisition.php
namespace App\Models;

use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Model;

class JobRequisition extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'department_id',
        'position_id',
        'requested_by',
        'title',
        'description',
        'requirements',
        'skills',
        'experience_level',
        'salary_range_min',
        'salary_range_max',
        'vacancies_count',
        'ai_embedding',
        'status',
        'requested_date',
        'target_date',
    ];

    protected $casts = [
        'ai_embedding' => 'array',
        'salary_range_min' => 'decimal:2',
        'salary_range_max' => 'decimal:2',
        'vacancies_count' => 'integer',
        'requested_date' => 'date',
        'target_date' => 'date',
    ];

    public function department()
    {
        return $this->belongsTo(Department::class);
    }

    public function position()
    {
        return $this->belongsTo(Position::class);
    }

    public function requestedBy()
    {
        return $this->belongsTo(Employee::class, 'requested_by');
    }

    public function jobPosting()
    {
        return $this->hasOne(JobPosting::class);
    }

    public function applications()
    {
        return $this->hasMany(JobApplication::class);
    }

    public function generateAiEmbedding(): void
    {
        $service = new CandidateMatchingService();
        $this->update(['ai_embedding' => $service->generateJobEmbedding($this)]);
    }

    public function getTopMatches(int $limit = 10): array
    {
        $service = new CandidateMatchingService();
        return array_slice($service->matchCandidates($this), 0, $limit);
    }
}
```

### ATS AI Filament Integration

#### AI-Powered Candidate Filtering
```php
// Filament/Resources/JobApplicationResource.php
namespace App\Filament\Resources;

use App\Services\AI\CandidateMatchingService;
use Filament\Tables\Filters\Filter;
use Illuminate\Database\Eloquent\Builder;

class JobApplicationResource extends Resource
{
    public function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('applicant_name'),
                Tables\Columns\TextColumn::make('ai_match_score')
                    ->label('AI Match Score')
                    ->badge()
                    ->color(fn ($record): string => 
                        $record->ai_match_score >= 80 ? 'success' :
                        ($record->ai_match_score >= 60 ? 'warning' : 'danger')
                    ),
                Tables\Columns\TextColumn::make('status'),
            ])
            ->filters([
                Filter::make('high_match')
                    ->query(fn (Builder $query): Builder => 
                        $query->where('ai_match_score', '>=', 80)
                    ),
                Filter::make('medium_match')
                    ->query(fn (Builder $query): Builder => 
                        $query->whereBetween('ai_match_score', [60, 79])
                    ),
                Filter::make('ai_recommended')
                    ->query(fn (Builder $query): Builder => 
                        $query->whereJsonContains('ai_screening_result->interview_recommendation', 'yes')
                    ),
            ]);
    }
}
```

#### AI-Enhanced Application View
```php
// Filament/Resources/JobApplicationResource/Pages/ViewApplication.php
namespace App\Filament\Resources\JobApplicationResource\Pages;

use App\Services\AI\CandidateScreeningAgent;
use Filament\Actions;
use Filament\Resources\Pages\ViewRecord;

class ViewApplication extends ViewRecord
{
    protected static string $resource = JobApplicationResource::class;

    protected function getHeaderActions(): array
    {
        return [
            Actions\Action::make('ai_screen')
                ->label('AI Screening')
                ->icon('heroicon-o-sparkles')
                ->action(function (): void {
                    $agent = new CandidateScreeningAgent();
                    $result = $agent->screenCandidate(
                        $this->record->toArray(),
                        $this->record->jobRequisition->toArray()
                    );
                    $this->record->updateAiScreening($result);
                })
                ->visible(fn (): bool => empty($this->record->ai_screening_result)),
            
            Actions\Action::make('generate_questions')
                ->label('Generate Interview Questions')
                ->icon('heroicon-o-question-mark-circle')
                ->action(function (): void {
                    $this->record->generateAiInterviewQuestions();
                })
                ->visible(fn (): bool => empty($this->record->ai_interview_questions)),
            
            Actions\Action::make('recalculate_match')
                ->label('Recalculate Match Score')
                ->icon('heroicon-o-arrow-path')
                ->action(function (): void {
                    $service = new CandidateMatchingService();
                    $matches = $service->matchCandidates($this->record->jobRequisition);
                    $match = collect($matches)->firstWhere('application_id', $this->record->id);
                    if ($match) {
                        $this->record->updateAiMatchScore($match['similarity_score']);
                    }
                }),
        ];
    }
}
```

### ATS AI Testing

#### AI Service Tests (Pest v4)
```php
// Tests/Unit/AI/ResumeParsingServiceTest.php
use App\Services\AI\ResumeParsingService;
use Illuminate\Foundation\Testing\RefreshDatabase;

uses(RefreshDatabase::class);

test('resume parsing service extracts structured data', function () {
    $service = new ResumeParsingService();
    $testResume = storage_path('test-resumes/sample-resume.pdf');
    
    $parsedData = $service->parseResume($testResume);
    
    expect($parsedData)->toHaveKeys(['name', 'email', 'phone', 'experience', 'education', 'skills'])
        ->and($parsedData['name'])->not->toBeEmpty()
        ->and($parsedData['email'])->toContain('@');
});

test('candidate matching service calculates similarity correctly', function () {
    $service = new CandidateMatchingService();
    
    $vector1 = [0.1, 0.2, 0.3];
    $vector2 = [0.1, 0.2, 0.3];
    
    $similarity = $service->calculateCosineSimilarity($vector1, $vector2);
    
    expect($similarity)->toBeCloseTo(1.0, 1);
});

test('candidate screening agent provides structured output', function () {
    $agent = new CandidateScreeningAgent();
    
    $candidateData = [
        'name' => 'John Doe',
        'experience' => '5 years in software development',
        'skills' => ['PHP', 'Laravel', 'MySQL'],
    ];
    
    $jobRequirements = [
        'title' => 'Senior PHP Developer',
        'required_skills' => ['PHP', 'Laravel'],
        'experience_years' => 3,
    ];
    
    $result = $agent->screenCandidate($candidateData, $jobRequirements);
    
    expect($result)->toHaveKeys(['overall_rating', 'skills_match', 'interview_recommendation'])
        ->and($result['overall_rating'])->toBeBetween(1, 10);
});
```

### Implementation Timeline

**Phase 3 (Recruitment Module):**
- Install Laravel AI package
- Configure AI providers and API keys
- Set up vector storage for resumes
- Implement resume parsing service
- Create candidate matching service
- Build candidate screening agent

**Phase 4 (Advanced ATS Features):**
- Skills extraction service
- AI-powered job matching
- Interview question generation
- Resume similarity search
- AI-enhanced candidate ranking
- ATS AI Filament integration

---

## 📋 Development Phases (Updated with Security Requirements)

### Phase 1: Foundation & Core Infrastructure (2-3 weeks)

**Security Integration:**
- ✅ Configure Laravel 13 middleware (PreventRequestForgery)
- ✅ Set up Content Security Policy
- ✅ Configure session security (Redis, secure cookies)
- ✅ Implement encryption for sensitive model attributes
- ✅ Set up CSRF protection
- ✅ Configure XSS protection

**Core Components:**
- Laravel 13 project setup with PHP 8.4
- Filament v5 panel configuration (Admin + Portal)
- Database migrations (Users, CompanySettings, Branches)
- Core models with encrypted casts
- App\Enums\UserRole enum integration
- Laravel AI SDK setup (for later ATS use)
- Redis queue configuration
- S3-compatible storage setup
- Hashid configuration per model

**Policies & Authorization:**
- BasePolicy template
- User and Employee policies
- Department and Branch policies
- App\Enums\UserRole enum integration
- Panel middleware configuration

**Testing:**
- Authentication tests
- Authorization tests
- Security configuration tests
- Policy compliance tests

### Phase 2: Company & Branch Management (1-2 weeks)

**Security Integration:**
- ✅ Input validation for company settings
- ✅ File upload security for company logo
- ✅ Geofence data validation
- ✅ API response field control

**Components:**
- CompanySettings singleton model
- Branch management with geofencing
- Department hierarchical structure
- Position management
- Company settings seeding
- Default branch creation

**Policies & Authorization:**
- CompanySettings policy (admin only)
- Branch policy (admin + branch managers)
- Department policy (admin + department heads)
- Position policy (admin + HR)

### Phase 3: Employee Management (2-3 weeks)

**Security Integration:**
- ✅ Comprehensive input validation for employee data
- ✅ PII encryption implementation
- ✅ File upload security for documents
- ✅ Audit log PII masking
- ✅ Sensitive data visibility control

**Components:**
- Employee model with encrypted fields
- Employee assignments
- Emergency contacts
- Employee documents with secure storage
- Employee allowances
- Department/position assignment
- Employee profile management

**Policies & Authorization:**
- Employee policy (comprehensive)
- Sensitive data access control
- Department manager access
- Employee self-service access
- API resource field-level exposure

### Phase 4: Time & Attendance (2-3 weeks)

**Security Integration:**
- ✅ Biometric device authentication
- ✅ Geofence validation
- ✅ Attendance data validation
- ✅ Location data security
- ✅ API token management

**Components:**
- Shift management
- Employee schedules
- Attendance records
- DTR processing
- Geofence validation
- Biometric device integration
- Field location management

**Policies & Authorization:**
- Attendance policy
- Attendance correction workflow
- Device mapping policy
- Geofence approval policy

### Phase 5: Leave Management (2 weeks)

**Security Integration:**
- ✅ Leave request validation
- ✅ Approval workflow security
- ✅ Balance calculation security
- ✅ Document upload security

**Components:**
- Leave type configuration
- Leave balance management
- Leave request workflow
- Multi-level approval chains
- Leave calendar integration
- Carryover and cash conversion

**Policies & Authorization:**
- Leave request policy
- Approval chain authorization
- Balance view permissions
- Document access control

### Phase 6: Payroll Module (3-4 weeks)

**Security Integration:**
- ✅ Payroll processing authorization
- ✅ Compensation data encryption
- ✅ Statutory calculation security
- ✅ API response masking
- ✅ Transaction security

**Components:**
- Pay period management
- Salary grades
- Payroll runs
- Payroll items
- Earnings and deductions
- Statutory contributions (SSS, PhilHealth, Pag-IBIG)
- BIR 2316 and 1604-C reports
- 13th month pay processing
- Benefits calculation

**Policies & Authorization:**
- Payroll policy (comprehensive)
- Processing restrictions
- Finalization authorization
- Payslip access control
- Compensation visibility

### Phase 7: Recruitment (ATS) with AI (2-3 weeks)

**Security Integration:**
- ✅ Resume parsing data sanitization
- ✅ AI API key security
- ✅ Candidate data privacy
- ✅ File upload validation
- ✅ Vector store security

**Components:**
- Job requisitions
- Job postings
- Job applications
- Laravel AI integration
- Resume parsing service
- Candidate matching service
- Screening agent
- Interview scheduling
- Job offers

**Policies & Authorization:**
- Job requisition policy
- Application policy
- AI service access control
- Candidate data privacy

### Phase 8: Advanced Features (3-4 weeks)

**Security Integration:**
- ✅ Performance data security
- ✅ Training record privacy
- ✅ Grievance confidentiality
- ✅ Asset assignment security
- ✅ Data privacy consent tracking

**Components:**
- Performance management
- Training programs
- Asset management
- Employee relations
- Compliance tracking
- Data privacy management
- Webhook integrations
- SSO integration

**Policies & Authorization:**
- Performance review policy
- Training record policy
- Grievance policy
- Asset policy
- Privacy policy

### Phase 9: Employee Portal & Self-Service (2-3 weeks)

**Security Integration:**
- ✅ Employee-only access enforcement
- ✅ Self-service data validation
- ✅ Personal data exposure control
- ✅ Profile update security

**Components:**
- Employee dashboard
- Personal profile management
- My attendance
- My leave balances
- My payslips
- My benefits
- Onboarding tasks
- My assets
- Notifications

**Policies & Authorization:**
- Portal access control
- Self-service policy
- Personal data access
- Cross-user access prevention

### Phase 10: Production Readiness & Deployment (2-3 weeks)

**Security Integration:**
- ✅ Security audit completion
- ✅ Penetration testing
- ✅ Backup security
- ✅ Monitoring setup
- ✅ Incident response plan

**Components:**
- Security hardening
- Performance optimization
- Backup and recovery
- Monitoring and alerting
- Documentation completion
- User training
- Go-live preparation

**Security Checklist:**
- ✅ All policies implemented and tested
- ✅ Encryption for all sensitive fields
- ✅ PII masking in logs and audit trails
- ✅ Input validation comprehensive
- ✅ File upload security complete
- ✅ API response field control implemented
- ✅ XSS and CSRF protection configured
- ✅ Session security hardened
- ✅ Security audit passed
- ✅ Penetration testing completed

---

## 🔒 Security Audit Checklist

### Data Protection
- [ ] All sensitive fields encrypted at rest
- [ ] PII masking in logs and audit trails
- [ ] Hashid implementation for public IDs
- [ ] API response field-level control
- [ ] Sensitive data access policies implemented

### Input Validation
- [ ] Form Request validation for all inputs
- [ ] File upload security (type, size, validation)
- [ ] XSS protection configured
- [ ] CSRF protection (Laravel 13 PreventRequestForgery)
- [ ] Input sanitization middleware

### Authentication & Authorization
- [ ] Laravel Policies for all domains using App\Enums\UserRole
- [ ] Panel middleware configuration
- [ ] API token management
- [ ] Department/location-based access controls

### Communication Security
- [ ] HTTPS enforcement
- [ ] Secure session configuration
- [ ] Content Security Policy
- [ ] Secure cookie settings
- [ ] API rate limiting

### Infrastructure Security
- [ ] Environment variable protection
- [ ] Database encryption at rest
- [ ] S3 bucket security
- [ ] Redis authentication
- [ ] Backup encryption

### Monitoring & Auditing
- [ ] Comprehensive audit logging
- [ ] Security event monitoring
- [ ] Failed login tracking
- [ ] PII access logging
- [ ] Incident response procedures

---

## 📋 Development Phases

| Phase | Focus | Duration |
|---|---|---|
| Phase 1 | Foundation & Core Infrastructure | 2 weeks |
| Phase 2 | Company & Branch Management | 1 week |
| Phase 3 | Employee Management Core | 2 weeks |
| Phase 4 | Time & Attendance Foundation | 2 weeks |
| Phase 5 | Leave Management System | 1 week |
| Phase 6 | Payroll Foundation | 3 weeks |
| Phase 7 | Employee Self-Service Portal | 2 weeks |
| Phase 8 | Advanced Features (ATS, Performance, Relations) | 3 weeks |
| Phase 9 | Compliance & Integrations | 2 weeks |
| Phase 10 | Reports & Analytics | 1 week |

---

## PHASE 1 — Foundation & Core Infrastructure

### Phase Overview
Establish the project foundation with Laravel 13, Filament 5 two-panel architecture, authentication, and core infrastructure components.

### Foundation Models

#### User Model
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Hidden;
use Illuminate\Database\Eloquent\Attributes\Table;
use App\Enums\UserRole;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

#[Table('users')]
#[Fillable(['name', 'email', 'password', 'employee_id', 'role', 'is_active', 'email_verified_at'])]
#[Hidden(['password', 'remember_token'])]
class User extends Authenticatable
{
    use HasApiTokens, Notifiable, SoftDeletes;

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'role' => UserRole::class,
            'is_active' => 'boolean',
            'password' => 'hashed',
        ];
    }

    public function employee()
    {
        return $this->belongsTo(Employee::class);
    }

    public function canAccessPanel(Panel $panel): bool
    {
        if ($panel->getId() === 'admin') {
            return in_array($this->role, [
                UserRole::COMPANY_ADMIN, 
                UserRole::HR_ADMIN, 
                UserRole::HR_STAFF, 
                UserRole::PAYROLL_OFFICER, 
                UserRole::MANAGER
            ]);
        }

        return $this->role === UserRole::EMPLOYEE;
    }
}
```

#### CompanySettings Model (Singleton)
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;

#[Table('company_settings')]
#[Fillable(['name', 'logo', 'address', 'city', 'country', 'timezone', 'locale', 'currency', 'tin_no'])]
class CompanySettings extends Model
{
    protected function casts(): array
    {
        return [
            'timezone' => 'string',
            'locale' => 'string',
            'currency' => 'string',
        ];
    }

    public static function current(): self
    {
        return static::firstOrFail();
    }
}
```

### Enums

#### UserRole Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum UserRole: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case COMPANY_ADMIN = 'company-admin';
    case HR_ADMIN = 'hr-admin';
    case HR_STAFF = 'hr-staff';
    case PAYROLL_OFFICER = 'payroll-officer';
    case MANAGER = 'manager';
    case EMPLOYEE = 'employee';

    public function getLabel(): string
    {
        return match($this) {
            self::COMPANY_ADMIN => 'Company Admin',
            self::HR_ADMIN => 'HR Admin',
            self::HR_STAFF => 'HR Staff',
            self::PAYROLL_OFFICER => 'Payroll Officer',
            self::MANAGER => 'Manager',
            self::EMPLOYEE => 'Employee',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::COMPANY_ADMIN => 'Full system access with all permissions',
            self::HR_ADMIN => 'HR management with employee and attendance access',
            self::HR_STAFF => 'HR operations with limited administrative access',
            self::PAYROLL_OFFICER => 'Payroll processing and compensation management',
            self::MANAGER => 'Team management and performance oversight',
            self::EMPLOYEE => 'Self-service portal access only',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::COMPANY_ADMIN => 'danger',
            self::HR_ADMIN => 'warning',
            self::HR_STAFF => 'info',
            self::PAYROLL_OFFICER => 'primary',
            self::MANAGER => 'success',
            self::EMPLOYEE => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::COMPANY_ADMIN => Heroicon::ShieldCheck,
            self::HR_ADMIN => Heroicon::Users,
            self::HR_STAFF => Heroicon::UserGroup,
            self::PAYROLL_OFFICER => Heroicon::CurrencyDollar,
            self::MANAGER => Heroicon::Briefcase,
            self::EMPLOYEE => Heroicon::User,
        };
    }

    public function canAccessAdminPanel(): bool
    {
        return $this !== self::EMPLOYEE;
    }

    public function canAccessPortalPanel(): bool
    {
        return $this === self::EMPLOYEE;
    }
}
```

#### EmploymentStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum EmploymentStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case ACTIVE = 'active';
    case INACTIVE = 'inactive';
    case RESIGNED = 'resigned';
    case TERMINATED = 'terminated';
    case RETIRED = 'retired';

    public function getLabel(): string
    {
        return match($this) {
            self::ACTIVE => 'Active',
            self::INACTIVE => 'Inactive',
            self::RESIGNED => 'Resigned',
            self::TERMINATED => 'Terminated',
            self::RETIRED => 'Retired',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::ACTIVE => 'Currently employed and active',
            self::INACTIVE => 'Employed but temporarily inactive',
            self::RESIGNED => 'Voluntarily left the company',
            self::TERMINATED => 'Employment was terminated',
            self::RETIRED => 'Retired from service',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::ACTIVE => 'success',
            self::INACTIVE => 'warning',
            self::RESIGNED => 'info',
            self::TERMINATED => 'danger',
            self::RETIRED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::ACTIVE => Heroicon::CheckCircle,
            self::INACTIVE => Heroicon::Clock,
            self::RESIGNED => Heroicon::ArrowRightOnRectangle,
            self::TERMINATED => Heroicon::XCircle,
            self::RETIRED => Heroicon::Sun,
        };
    }
}
```

#### EmploymentType Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum EmploymentType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case REGULAR = 'regular';
    case CONTRACTUAL = 'contractual';
    case PROBATIONARY = 'probationary';
    case PART_TIME = 'part_time';

    public function getLabel(): string
    {
        return match($this) {
            self::REGULAR => 'Regular',
            self::CONTRACTUAL => 'Contractual',
            self::PROBATIONARY => 'Probationary',
            self::PART_TIME => 'Part-time',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::REGULAR => 'Permanent regular employment',
            self::CONTRACTUAL => 'Fixed-term contract employment',
            self::PROBATIONARY => 'Probationary period before regularization',
            self::PART_TIME => 'Part-time employment arrangement',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::REGULAR => 'success',
            self::CONTRACTUAL => 'warning',
            self::PROBATIONARY => 'info',
            self::PART_TIME => 'primary',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::REGULAR => Heroicon::Briefcase,
            self::CONTRACTUAL => Heroicon::DocumentText,
            self::PROBATIONARY => Heroicon::AcademicCap,
            self::PART_TIME => Heroicon::Clock,
        };
    }
}
```

#### LeaveRequestStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum LeaveRequestStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PENDING = 'pending';
    case APPROVED = 'approved';
    case REJECTED = 'rejected';
    case CANCELLED = 'cancelled';

    public function getLabel(): string
    {
        return match($this) {
            self::PENDING => 'Pending',
            self::APPROVED => 'Approved',
            self::REJECTED => 'Rejected',
            self::CANCELLED => 'Cancelled',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::PENDING => 'Awaiting approval from manager',
            self::APPROVED => 'Leave request has been approved',
            self::REJECTED => 'Leave request was denied',
            self::CANCELLED => 'Leave request cancelled by employee',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::PENDING => 'warning',
            self::APPROVED => 'success',
            self::REJECTED => 'danger',
            self::CANCELLED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::PENDING => Heroicon::Clock,
            self::APPROVED => Heroicon::CheckCircle,
            self::REJECTED => Heroicon::XCircle,
            self::CANCELLED => Heroicon::XMark,
        };
    }
}
```

#### AttendanceStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum AttendanceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PRESENT = 'present';
    case ABSENT = 'absent';
    case LATE = 'late';
    case HALF_DAY = 'half_day';
    case REST_DAY = 'rest_day';
    case HOLIDAY = 'holiday';
    case ON_LEAVE = 'on_leave';

    public function getLabel(): string
    {
        return match($this) {
            self::PRESENT => 'Present',
            self::ABSENT => 'Absent',
            self::LATE => 'Late',
            self::HALF_DAY => 'Half Day',
            self::REST_DAY => 'Rest Day',
            self::HOLIDAY => 'Holiday',
            self::ON_LEAVE => 'On Leave',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::PRESENT => 'Attended work on time',
            self::ABSENT => 'Did not attend work',
            self::LATE => 'Arrived after shift start time',
            self::HALF_DAY => 'Attended half of the work day',
            self::REST_DAY => 'Scheduled rest day',
            self::HOLIDAY => 'Public holiday',
            self::ON_LEAVE => 'On approved leave',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::PRESENT => 'success',
            self::ABSENT => 'danger',
            self::LATE => 'warning',
            self::HALF_DAY => 'info',
            self::REST_DAY => 'gray',
            self::HOLIDAY => 'primary',
            self::ON_LEAVE => 'purple',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::PRESENT => Heroicon::CheckCircle,
            self::ABSENT => Heroicon::XCircle,
            self::LATE => Heroicon::Clock,
            self::HALF_DAY => Heroicon::Moon,
            self::REST_DAY => Heroicon::Sun,
            self::HOLIDAY => Heroicon::CalendarDays,
            self::ON_LEAVE => Heroicon::UmbrellaBeach,
        };
    }
}
```

#### PayrollStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum PayrollStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case DRAFT = 'draft';
    case PROCESSING = 'processing';
    case PROCESSED = 'processed';
    case LOCKED = 'locked';
    case PAID = 'paid';

    public function getLabel(): string
    {
        return match($this) {
            self::DRAFT => 'Draft',
            self::PROCESSING => 'Processing',
            self::PROCESSED => 'Processed',
            self::LOCKED => 'Locked',
            self::PAID => 'Paid',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::DRAFT => 'Payroll run in draft state',
            self::PROCESSING => 'Currently being processed',
            self::PROCESSED => 'Processing completed, ready for review',
            self::LOCKED => 'Locked for review, no further edits allowed',
            self::PAID => 'Payments have been disbursed',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::DRAFT => 'gray',
            self::PROCESSING => 'info',
            self::PROCESSED => 'warning',
            self::LOCKED => 'primary',
            self::PAID => 'success',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::DRAFT => Heroicon::DocumentText,
            self::PROCESSING => Heroicon::Cog,
            self::PROCESSED => Heroicon::Eye,
            self::LOCKED => Heroicon::LockClosed,
            self::PAID => Heroicon::CurrencyDollar,
        };
    }
}
```

#### HolidayType Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum HolidayType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case REGULAR = 'regular';
    case SPECIAL_NON_WORKING = 'special_non_working';
    case SPECIAL_WORKING = 'special_working';

    public function getLabel(): string
    {
        return match($this) {
            self::REGULAR => 'Regular Holiday',
            self::SPECIAL_NON_WORKING => 'Special Non-Working Holiday',
            self::SPECIAL_WORKING => 'Special Working Holiday',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::REGULAR => 'Regular holiday with full pay premium',
            self::SPECIAL_NON_WORKING => 'Special non-working holiday with 30% premium',
            self::SPECIAL_WORKING => 'Special working day, no premium',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::REGULAR => 'danger',
            self::SPECIAL_NON_WORKING => 'warning',
            self::SPECIAL_WORKING => 'info',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::REGULAR => Heroicon::CalendarDays,
            self::SPECIAL_NON_WORKING => Heroicon::Calendar,
            self::SPECIAL_WORKING => Heroicon::Briefcase,
        };
    }
}
```

#### GeofenceStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum GeofenceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case WITHIN = 'within';
    case OUTSIDE = 'outside';
    case WITHIN_SITE = 'within_site';
    case OUTSIDE_PENDING_APPROVAL = 'outside_pending_approval';
    case UNCHECKED = 'unchecked';

    public function getLabel(): string
    {
        return match($this) {
            self::WITHIN => 'Within Geofence',
            self::OUTSIDE => 'Outside Geofence',
            self::WITHIN_SITE => 'Within Site Location',
            self::OUTSIDE_PENDING_APPROVAL => 'Outside - Pending Approval',
            self::UNCHECKED => 'Not Checked',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::WITHIN => 'Attendance within branch geofence',
            self::OUTSIDE => 'Attendance outside branch geofence',
            self::WITHIN_SITE => 'Within assigned field site location',
            self::OUTSIDE_PENDING_APPROVAL => 'Outside field site, awaiting manager approval',
            self::UNCHECKED => 'Geofence not configured for this location',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::WITHIN => 'success',
            self::OUTSIDE => 'danger',
            self::WITHIN_SITE => 'success',
            self::OUTSIDE_PENDING_APPROVAL => 'warning',
            self::UNCHECKED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::WITHIN => Heroicon::MapPin,
            self::OUTSIDE => Heroicon::MapPin,
            self::WITHIN_SITE => Heroicon::MapPin,
            self::OUTSIDE_PENDING_APPROVAL => Heroicon::ExclamationTriangle,
            self::UNCHECKED => Heroicon::QuestionMarkCircle,
        };
    }
}
```

#### ShiftType Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum ShiftType: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case FIXED = 'fixed';
    case FLEXIBLE = 'flexible';
    case ROTATING = 'rotating';

    public function getLabel(): string
    {
        return match($this) {
            self::FIXED => 'Fixed Shift',
            self::FLEXIBLE => 'Flexible Shift',
            self::ROTATING => 'Rotating Shift',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::FIXED => 'Fixed start and end times daily',
            self::FLEXIBLE => 'Flexible working hours within a range',
            self::ROTATING => 'Rotating shift schedule',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::FIXED => 'primary',
            self::FLEXIBLE => 'success',
            self::ROTATING => 'info',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::FIXED => Heroicon::Clock,
            self::FLEXIBLE => Heroicon::AdjustmentsHorizontal,
            self::ROTATING => Heroicon::ArrowPath,
        };
    }
}
```

#### GrievanceStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum GrievanceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case FILED = 'filed';
    case UNDER_INVESTIGATION = 'under_investigation';
    case MEDIATION_SCHEDULED = 'mediation_scheduled';
    case RESOLVED = 'resolved';
    case ESCALATED = 'escalated';
    case CLOSED = 'closed';

    public function getLabel(): string
    {
        return match($this) {
            self::FILED => 'Filed',
            self::UNDER_INVESTIGATION => 'Under Investigation',
            self::MEDIATION_SCHEDULED => 'Mediation Scheduled',
            self::RESOLVED => 'Resolved',
            self::ESCALATED => 'Escalated',
            self::CLOSED => 'Closed',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::FILED => 'Grievance has been filed',
            self::UNDER_INVESTIGATION => 'Currently under investigation',
            self::MEDIATION_SCHEDULED => 'Mediation process scheduled',
            self::RESOLVED => 'Issue has been resolved',
            self::ESCALATED => 'Escalated to higher authority',
            self::CLOSED => 'Case closed',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::FILED => 'info',
            self::UNDER_INVESTIGATION => 'warning',
            self::MEDIATION_SCHEDULED => 'primary',
            self::RESOLVED => 'success',
            self::ESCALATED => 'danger',
            self::CLOSED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::FILED => Heroicon::DocumentText,
            self::UNDER_INVESTIGATION => Heroicon::MagnifyingGlass,
            self::MEDIATION_SCHEDULED => Heroicon::Calendar,
            self::RESOLVED => Heroicon::CheckCircle,
            self::ESCALATED => Heroicon::ExclamationTriangle,
            self::CLOSED => Heroicon::XCircle,
        };
    }
}
```

#### ContractStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum ContractStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case DRAFT = 'draft';
    case SENT = 'sent';
    case ACKNOWLEDGED = 'acknowledged';
    case EXPIRED = 'expired';
    case TERMINATED = 'terminated';

    public function getLabel(): string
    {
        return match($this) {
            self::DRAFT => 'Draft',
            self::SENT => 'Sent',
            self::ACKNOWLEDGED => 'Acknowledged',
            self::EXPIRED => 'Expired',
            self::TERMINATED => 'Terminated',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::DRAFT => 'Contract in draft state',
            self::SENT => 'Contract sent to employee for review',
            self::ACKNOWLEDGED => 'Employee has acknowledged the contract',
            self::EXPIRED => 'Contract has expired',
            self::TERMINATED => 'Contract was terminated',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::DRAFT => 'gray',
            self::SENT => 'info',
            self::ACKNOWLEDGED => 'success',
            self::EXPIRED => 'warning',
            self::TERMINATED => 'danger',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::DRAFT => Heroicon::DocumentText,
            self::SENT => Heroicon::PaperAirplane,
            self::ACKNOWLEDGED => Heroicon::CheckCircle,
            self::EXPIRED => Heroicon::Clock,
            self::TERMINATED => Heroicon::XCircle,
        };
    }
}
```

#### PerformanceRating Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum PerformanceRating: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case OUTSTANDING = 'outstanding';
    case EXCEEDS_EXPECTATIONS = 'exceeds_expectations';
    case MEETS_EXPECTATIONS = 'meets_expectations';
    case NEEDS_IMPROVEMENT = 'needs_improvement';
    case UNSATISFACTORY = 'unsatisfactory';

    public function getLabel(): string
    {
        return match($this) {
            self::OUTSTANDING => 'Outstanding',
            self::EXCEEDS_EXPECTATIONS => 'Exceeds Expectations',
            self::MEETS_EXPECTATIONS => 'Meets Expectations',
            self::NEEDS_IMPROVEMENT => 'Needs Improvement',
            self::UNSATISFACTORY => 'Unsatisfactory',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::OUTSTANDING => 'Consistently exceeds all expectations',
            self::EXCEEDS_EXPECTATIONS => 'Regularly exceeds expectations',
            self::MEETS_EXPECTATIONS => 'Meets all expectations',
            self::NEEDS_IMPROVEMENT => 'Some areas need improvement',
            self::UNSATISFACTORY => 'Does not meet expectations',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::OUTSTANDING => 'success',
            self::EXCEEDS_EXPECTATIONS => 'primary',
            self::MEETS_EXPECTATIONS => 'info',
            self::NEEDS_IMPROVEMENT => 'warning',
            self::UNSATISFACTORY => 'danger',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::OUTSTANDING => Heroicon::Star,
            self::EXCEEDS_EXPECTATIONS => Heroicon::Sparkles,
            self::MEETS_EXPECTATIONS => Heroicon::CheckCircle,
            self::NEEDS_IMPROVEMENT => Heroicon::ExclamationTriangle,
            self::UNSATISFACTORY => Heroicon::XCircle,
        };
    }
}
```

#### AssetStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum AssetStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case AVAILABLE = 'available';
    case ASSIGNED = 'assigned';
    case UNDER_MAINTENANCE = 'under_maintenance';
    case RETIRED = 'retired';

    public function getLabel(): string
    {
        return match($this) {
            self::AVAILABLE => 'Available',
            self::ASSIGNED => 'Assigned',
            self::UNDER_MAINTENANCE => 'Under Maintenance',
            self::RETIRED => 'Retired',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::AVAILABLE => 'Available for assignment',
            self::ASSIGNED => 'Currently assigned to an employee',
            self::UNDER_MAINTENANCE => 'Undergoing maintenance',
            self::RETIRED => 'No longer in use',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::AVAILABLE => 'success',
            self::ASSIGNED => 'info',
            self::UNDER_MAINTENANCE => 'warning',
            self::RETIRED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::AVAILABLE => Heroicon::CheckCircle,
            self::ASSIGNED => Heroicon::User,
            self::UNDER_MAINTENANCE => Heroicon::WrenchScrewdriver,
            self::RETIRED => Heroicon::ArchiveBox,
        };
    }
}
```

#### AssetCondition Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum AssetCondition: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case NEW = 'new';
    case GOOD = 'good';
    case FAIR = 'fair';
    case POOR = 'poor';

    public function getLabel(): string
    {
        return match($this) {
            self::NEW => 'New',
            self::GOOD => 'Good',
            self::FAIR => 'Fair',
            self::POOR => 'Poor',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::NEW => 'Brand new condition',
            self::GOOD => 'Good condition with minimal wear',
            self::FAIR => 'Fair condition with noticeable wear',
            self::POOR => 'Poor condition, needs replacement',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::NEW => 'success',
            self::GOOD => 'primary',
            self::FAIR => 'warning',
            self::POOR => 'danger',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::NEW => Heroicon::Sparkles,
            self::GOOD => Heroicon::CheckCircle,
            self::FAIR => Heroicon::ExclamationTriangle,
            self::POOR => Heroicon::XCircle,
        };
    }
}
```

#### LoanStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum LoanStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PENDING_APPROVAL = 'pending_approval';
    case ACTIVE = 'active';
    case PAID = 'paid';
    case DEFAULTED = 'defaulted';
    case CANCELLED = 'cancelled';

    public function getLabel(): string
    {
        return match($this) {
            self::PENDING_APPROVAL => 'Pending Approval',
            self::ACTIVE => 'Active',
            self::PAID => 'Paid',
            self::DEFAULTED => 'Defaulted',
            self::CANCELLED => 'Cancelled',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::PENDING_APPROVAL => 'Awaiting approval from HR/Finance',
            self::ACTIVE => 'Loan is active with ongoing deductions',
            self::PAID => 'Loan has been fully paid',
            self::DEFAULTED => 'Loan has defaulted',
            self::CANCELLED => 'Loan was cancelled',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::PENDING_APPROVAL => 'warning',
            self::ACTIVE => 'info',
            self::PAID => 'success',
            self::DEFAULTED => 'danger',
            self::CANCELLED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::PENDING_APPROVAL => Heroicon::Clock,
            self::ACTIVE => Heroicon::CurrencyDollar,
            self::PAID => Heroicon::CheckCircle,
            self::DEFAULTED => Heroicon::XCircle,
            self::CANCELLED => Heroicon::XMark,
        };
    }
}
```

#### ClearanceStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum ClearanceStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PENDING = 'pending';
    case IN_PROGRESS = 'in_progress';
    case BLOCKED = 'blocked';
    case COMPLETED = 'completed';

    public function getLabel(): string
    {
        return match($this) {
            self::PENDING => 'Pending',
            self::IN_PROGRESS => 'In Progress',
            self::BLOCKED => 'Blocked',
            self::COMPLETED => 'Completed',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::PENDING => 'Clearance process not started',
            self::IN_PROGRESS => 'Clearance process in progress',
            self::BLOCKED => 'Blocked by pending items',
            self::COMPLETED => 'All clearance items completed',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::PENDING => 'gray',
            self::IN_PROGRESS => 'info',
            self::BLOCKED => 'danger',
            self::COMPLETED => 'success',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::PENDING => Heroicon::Clock,
            self::IN_PROGRESS => Heroicon::ArrowPath,
            self::BLOCKED => Heroicon::LockClosed,
            self::COMPLETED => Heroicon::CheckCircle,
        };
    }
}
```

#### PipStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum PipStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case ACTIVE = 'active';
    case EXTENDED = 'extended';
    case COMPLETED = 'completed';
    case TERMINATED = 'terminated';

    public function getLabel(): string
    {
        return match($this) {
            self::ACTIVE => 'Active',
            self::EXTENDED => 'Extended',
            self::COMPLETED => 'Completed',
            self::TERMINATED => 'Terminated',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::ACTIVE => 'PIP is currently active',
            self::EXTENDED => 'PIP period has been extended',
            self::COMPLETED => 'Employee successfully completed PIP',
            self::TERMINATED => 'Employment terminated due to PIP failure',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::ACTIVE => 'warning',
            self::EXTENDED => 'info',
            self::COMPLETED => 'success',
            self::TERMINATED => 'danger',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::ACTIVE => Heroicon::ExclamationTriangle,
            self::EXTENDED => Heroicon::Clock,
            self::COMPLETED => Heroicon::CheckCircle,
            self::TERMINATED => Heroicon::XCircle,
        };
    }
}
```

#### OnboardingTaskStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum OnboardingTaskStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case PENDING = 'pending';
    case IN_PROGRESS = 'in_progress';
    case COMPLETED = 'completed';
    case SKIPPED = 'skipped';

    public function getLabel(): string
    {
        return match($this) {
            self::PENDING => 'Pending',
            self::IN_PROGRESS => 'In Progress',
            self::COMPLETED => 'Completed',
            self::SKIPPED => 'Skipped',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::PENDING => 'Task not yet started',
            self::IN_PROGRESS => 'Task currently in progress',
            self::COMPLETED => 'Task has been completed',
            self::SKIPPED => 'Task was skipped',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::PENDING => 'gray',
            self::IN_PROGRESS => 'info',
            self::COMPLETED => 'success',
            self::SKIPPED => 'warning',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::PENDING => Heroicon::Clock,
            self::IN_PROGRESS => Heroicon::ArrowPath,
            self::COMPLETED => Heroicon::CheckCircle,
            self::SKIPPED => Heroicon::Forward,
        };
    }
}
```

#### TrainingStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum TrainingStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case SCHEDULED = 'scheduled';
    case IN_PROGRESS = 'in_progress';
    case COMPLETED = 'completed';
    case CANCELLED = 'cancelled';

    public function getLabel(): string
    {
        return match($this) {
            self::SCHEDULED => 'Scheduled',
            self::IN_PROGRESS => 'In Progress',
            self::COMPLETED => 'Completed',
            self::CANCELLED => 'Cancelled',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::SCHEDULED => 'Training is scheduled',
            self::IN_PROGRESS => 'Training currently in progress',
            self::COMPLETED => 'Training has been completed',
            self::CANCELLED => 'Training was cancelled',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::SCHEDULED => 'info',
            self::IN_PROGRESS => 'warning',
            self::COMPLETED => 'success',
            self::CANCELLED => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::SCHEDULED => Heroicon::Calendar,
            self::IN_PROGRESS => Heroicon::AcademicCap,
            self::COMPLETED => Heroicon::CheckCircle,
            self::CANCELLED => Heroicon::XMark,
        };
    }
}
```

#### Gender Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum Gender: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case MALE = 'male';
    case FEMALE = 'female';
    case OTHER = 'other';
    case PREFER_NOT_TO_SAY = 'prefer_not_to_say';

    public function getLabel(): string
    {
        return match($this) {
            self::MALE => 'Male',
            self::FEMALE => 'Female',
            self::OTHER => 'Other',
            self::PREFER_NOT_TO_SAY => 'Prefer not to say',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::MALE => 'Male',
            self::FEMALE => 'Female',
            self::OTHER => 'Other gender identity',
            self::PREFER_NOT_TO_SAY => 'Choose not to disclose',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::MALE => 'primary',
            self::FEMALE => 'pink',
            self::OTHER => 'purple',
            self::PREFER_NOT_TO_SAY => 'gray',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::MALE => Heroicon::User,
            self::FEMALE => Heroicon::User,
            self::OTHER => Heroicon::User,
            self::PREFER_NOT_TO_SAY => Heroicon::QuestionMarkCircle,
        };
    }
}
```

#### CivilStatus Enum
```php
namespace App\Enums;

use Filament\Support\Contracts\HasColor;
use Filament\Support\Contracts\HasDescription;
use Filament\Support\Contracts\HasIcon;
use Filament\Support\Contracts\HasLabel;
use Filament\Support\Icons\Heroicon;

enum CivilStatus: string implements HasLabel, HasDescription, HasColor, HasIcon
{
    case SINGLE = 'single';
    case MARRIED = 'married';
    case WIDOWED = 'widowed';
    case LEGALLY_SEPARATED = 'legally_separated';

    public function getLabel(): string
    {
        return match($this) {
            self::SINGLE => 'Single',
            self::MARRIED => 'Married',
            self::WIDOWED => 'Widowed',
            self::LEGALLY_SEPARATED => 'Legally Separated',
        };
    }

    public function getDescription(): string
    {
        return match($this) {
            self::SINGLE => 'Never married',
            self::MARRIED => 'Currently married',
            self::WIDOWED => 'Spouse has passed away',
            self::LEGALLY_SEPARATED => 'Legally separated from spouse',
        };
    }

    public function getColor(): string
    {
        return match($this) {
            self::SINGLE => 'primary',
            self::MARRIED => 'success',
            self::WIDOWED => 'gray',
            self::LEGALLY_SEPARATED => 'warning',
        };
    }

    public function getIcon(): ?string
    {
        return match($this) {
            self::SINGLE => Heroicon::User,
            self::MARRIED => Heroicon::Heart,
            self::WIDOWED => Heroicon::UserMinus,
            self::LEGALLY_SEPARATED => Heroicon::UserMinus,
        };
    }
}
```

---

## 🎨 Using Filament v5 Enum Features in Resources

### Form Field Integration

```php
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\UserRole;
use Filament\Forms\Components\Select;
use Filament\Forms\Components\Radio;
use Filament\Forms\Components\ToggleButtons;
use Filament\Schemas\Schema;

// In Schema components:
Select::make('employment_status')
    ->options(EmploymentStatus::class)
    ->required(),

Radio::make('employment_type')
    ->options(EmploymentType::class)
    ->required(),

ToggleButtons::make('role')
    ->options(UserRole::class)
    ->inline()
    ->required(),
```

### Table Column Integration

```php
use App\Enums\EmploymentStatus;
use App\Enums\LeaveRequestStatus;
use Filament\Tables\Columns\TextColumn;
use Filament\Tables\Columns\SelectColumn;
use Filament\Tables\Filters\SelectFilter;

// In table columns:
TextColumn::make('employment_status')
    ->badge()
    ->color(fn (EmploymentStatus $status): string => $status->getColor())
    ->icon(fn (EmploymentStatus $status): string => $status->getIcon())
    ->formatStateUsing(fn (EmploymentStatus $status): string => $status->getLabel()),

SelectColumn::make('leave_request_status')
    ->options(LeaveRequestStatus::class)
    ->selectablePlaceholder(false),

// In table filters:
SelectFilter::make('employment_status')
    ->options(EmploymentStatus::class),
```

### Model Casting with Enums

```php
namespace App\Models;

use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use Illuminate\Database\Eloquent\Model;

class Employee extends Model
{
    protected function casts(): array
    {
        return [
            'employment_status' => EmploymentStatus::class,
            'employment_type' => EmploymentType::class,
        ];
    }
}
```

### Enum Validation in Form Requests

```php
use App\Enums\EmploymentStatus;
use Illuminate\Foundation\Http\FormRequest;

class StoreEmployeeRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'employment_status' => ['required', 'enum:' . EmploymentStatus::class],
            'employment_type' => ['required', 'enum:' . EmploymentType::class],
        ];
    }
}
```

### Enum Descriptions in Tooltips

```php
use App\Enums\UserRole;
use Filament\Forms\Components\Select;

Select::make('role')
    ->options(UserRole::class)
    ->helperText(fn (UserRole $role): string => $role->getDescription())
    ->required(),
```

### Migrations

#### Create Users Table
```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->foreignId('employee_id')->nullable()->constrained()->nullOnDelete();
            $table->boolean('is_active')->default(true);
            $table->timestamp('last_login_at')->nullable();
            $table->rememberToken();
            $table->softDeletes();
            $table->timestamps();

            $table->index('employee_id');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('users');
    }
};
```

#### Create Company Settings Table
```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('company_settings', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('logo')->nullable();
            $table->string('address');
            $table->string('city');
            $table->string('country')->default('Philippines');
            $table->string('timezone')->default('Asia/Manila');
            $table->string('locale')->default('en');
            $table->string('currency')->default('PHP');
            $table->string('tin_no')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('company_settings');
    }
};
```

#### Create Branches Table
```php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('branches', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('code')->unique();
            $table->string('address');
            $table->string('city');
            $table->string('timezone')->nullable();
            $table->decimal('geofence_lat', 10, 7)->nullable();
            $table->decimal('geofence_lng', 10, 7)->nullable();
            $table->unsignedInteger('geofence_radius_meters')->default(100)->nullable();
            $table->boolean('is_active')->default(true);
            $table->softDeletes();
            $table->timestamps();

            $table->index('code');
            $table->index('is_active');
            $table->index(['geofence_lat', 'geofence_lng']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('branches');
    }
};
```

### Services

#### AuthService
```php
namespace App\Services;

use App\Models\User;
use Illuminate\Auth\Events\Login;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;

class AuthService
{
    public function login(array $credentials): bool
    {
        if (!Auth::attempt($credentials)) {
            return false;
        }

        $user = Auth::user();

        if (!$user->is_active) {
            Auth::logout();
            return false;
        }

        $user->update(['last_login_at' => now()]);
        event(new Login('web', $user, false));

        return true;
    }

    public function generateDeviceToken(): string
    {
        $user = Auth::user();
        return $user->createToken('device-token', ['attendance'])->plainTextToken;
    }

    public function generatePortalToken(): string
    {
        $user = Auth::user();
        return $user->createToken('portal-token', ['portal'])->plainTextToken;
    }
}
```

#### HashidService
```php
namespace App\Services;

use Vinkla\Hashids\Facades\Hashids;

class HashidService
{
    public function encode(int $id, string $connection): string
    {
        return Hashids::connection($connection)->encode($id);
    }

    public function decode(string $hash, string $connection): ?int
    {
        $decoded = Hashids::connection($connection)->decode($hash);
        return $decoded[0] ?? null;
    }

    public function getConnectionForModel(string $modelClass): string
    {
        return match($modelClass) {
            User::class => 'user',
            Employee::class => 'employee',
            Branch::class => 'branch',
            default => 'main',
        };
    }
}
```

#### GeofenceService
```php
namespace App\Services;

use App\Models\Branch;

class GeofenceService
{
    private const EARTH_RADIUS = 6371000;

    public function check(float $lat, float $lng, Branch $branch): bool
    {
        if (!$branch->hasGeofence()) {
            return true;
        }

        $distance = $this->calculateDistance(
            $lat,
            $lng,
            $branch->geofence_lat,
            $branch->geofence_lng
        );

        return $distance <= $branch->geofence_radius_meters;
    }

    private function calculateDistance(float $lat1, float $lng1, float $lat2, float $lng2): float
    {
        $lat1Rad = deg2rad($lat1);
        $lat2Rad = deg2rad($lat2);
        $deltaLat = deg2rad($lat2 - $lat1);
        $deltaLng = deg2rad($lng2 - $lng1);

        $a = sin($deltaLat / 2) ** 2 +
             cos($lat1Rad) * cos($lat2Rad) *
             sin($deltaLng / 2) ** 2;

        $c = 2 * asin(sqrt($a));

        return self::EARTH_RADIUS * $c;
    }
}
```

#### AiService (Laravel AI SDK)
```php
namespace App\Services;

use App\Models\JobApplicant;
use App\Models\GrievanceCase;
use App\Models\HrTicket;
use Laravel\Ai\Facades\Ai;

class AiService
{
    public function parseResume(string $resumeText): array
    {
        $response = Ai::agent()
            ->withInstructions('Extract structured data from this resume. Return JSON with fields: name, email, phone, skills (array), experience_years, education.')
            ->prompt($resumeText);

        return json_decode($response->text, true);
    }

    public function analyzeGrievanceSentiment(string $description): string
    {
        $response = Ai::agent()
            ->withInstructions('Classify the sentiment of this grievance description as: positive, neutral, negative, or urgent.')
            ->prompt($description);

        return trim($response->text);
    }

    public function suggestTicketRouting(string $ticketSubject, string $ticketBody): string
    {
        $response = Ai::agent()
            ->withInstructions('Suggest the best department to route this HR ticket to. Return only the department name.')
            ->prompt($ticketSubject . ' ' . $ticketBody);

        return trim($response->text);
    }
}
```

### Factories

#### UserFactory
```php
namespace Database\Factories;

use App\Enums\UserRole;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class UserFactory extends Factory
{
    protected $model = User::class;

    public function definition(): array
    {
        return [
            'name' => fake()->name(),
            'email' => fake()->unique()->safeEmail(),
            'password' => bcrypt('password'),
            'role' => UserRole::EMPLOYEE,
            'is_active' => true,
            'email_verified_at' => now(),
        ];
    }

    public function companyAdmin(): self
    {
        return $this->state(fn (array $attributes) => [
            'email' => 'admin@company.com',
            'role' => UserRole::COMPANY_ADMIN,
        ]);
    }

    public function hrAdmin(): self
    {
        return $this->state(fn (array $attributes) => [
            'role' => UserRole::HR_ADMIN,
        ]);
    }

    public function payrollOfficer(): self
    {
        return $this->state(fn (array $attributes) => [
            'role' => UserRole::PAYROLL_OFFICER,
        ]);
    }

    public function manager(): self
    {
        return $this->state(fn (array $attributes) => [
            'role' => UserRole::MANAGER,
        ]);
    }

    public function employee(): self
    {
        return $this->state(fn (array $attributes) => [
            'role' => UserRole::EMPLOYEE,
        ]);
    }
}
```

#### CompanySettingsFactory
```php
namespace Database\Factories;

use App\Models\CompanySettings;
use Illuminate\Database\Eloquent\Factories\Factory;

class CompanySettingsFactory extends Factory
{
    protected $model = CompanySettings::class;

    public function definition(): array
    {
        return [
            'name' => 'Sample Company',
            'address' => fake()->address(),
            'city' => 'Manila',
            'country' => 'Philippines',
            'timezone' => 'Asia/Manila',
            'locale' => 'en',
            'currency' => 'PHP',
            'tin_no' => '123-456-789-000',
        ];
    }
}
```

#### BranchFactory
```php
namespace Database\Factories;

use App\Models\Branch;
use Illuminate\Database\Eloquent\Factories\Factory;

class BranchFactory extends Factory
{
    protected $model = Branch::class;

    public function definition(): array
    {
        return [
            'name' => fake()->company() . ' Branch',
            'code' => strtoupper(fake()->unique()->lexify('BRANCH-????')),
            'address' => fake()->address(),
            'city' => fake()->city(),
            'timezone' => 'Asia/Manila',
            'is_active' => true,
        ];
    }

    public function withGeofence(): self
    {
        return $this->state(fn (array $attributes) => [
            'geofence_lat' => fake()->latitude(14.5, 14.7),
            'geofence_lng' => fake()->longitude(120.9, 121.1),
            'geofence_radius_meters' => 100,
        ]);
    }
}
```

#### DepartmentFactory
```php
namespace Database\Factories;

use App\Models\Department;
use Illuminate\Database\Eloquent\Factories\Factory;

class DepartmentFactory extends Factory
{
    protected $model = Department::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Department',
            'code' => strtoupper(fake()->unique()->lexify('DEPT-????')),
            'description' => fake()->sentence(),
        ];
    }
}
```

#### PositionFactory
```php
namespace Database\Factories;

use App\Models\Department;
use App\Models\Position;
use Illuminate\Database\Eloquent\Factories\Factory;

class PositionFactory extends Factory
{
    protected $model = Position::class;

    public function definition(): array
    {
        return [
            'title' => fake()->jobTitle(),
            'department_id' => Department::factory(),
            'description' => fake()->sentence(),
        ];
    }
}
```

#### EmployeeFactory
```php
namespace Database\Factories;

use App\Enums\CivilStatus;
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\Gender;
use App\Models\Branch;
use App\Models\Department;
use App\Models\Employee;
use App\Models\Position;
use Illuminate\Database\Eloquent\Factories\Factory;

class EmployeeFactory extends Factory
{
    protected $model = Employee::class;

    public function definition(): array
    {
        return [
            'first_name' => fake()->firstName(),
            'last_name' => fake()->lastName(),
            'email' => fake()->unique()->safeEmail(),
            'phone' => fake()->phoneNumber(),
            'birth_date' => fake()->date('Y-m-d', '-18 years'),
            'gender' => fake()->randomElement(Gender::cases()),
            'civil_status' => fake()->randomElement(CivilStatus::cases()),
            'address' => fake()->address(),
            'employment_status' => EmploymentStatus::ACTIVE,
            'employment_type' => fake()->randomElement(EmploymentType::cases()),
            'hire_date' => fake()->date('Y-m-d', '-5 years'),
            'branch_id' => Branch::factory(),
            'department_id' => Department::factory(),
            'position_id' => Position::factory(),
            'is_field_employee' => fake()->boolean(20),
        ];
    }

    public function regular(): self
    {
        return $this->state(fn (array $attributes) => [
            'employment_type' => EmploymentType::REGULAR,
        ]);
    }

    public function probationary(): self
    {
        return $this->state(fn (array $attributes) => [
            'employment_type' => EmploymentType::PROBATIONARY,
        ]);
    }

    public function fieldEmployee(): self
    {
        return $this->state(fn (array $attributes) => [
            'is_field_employee' => true,
        ]);
    }
}
```

#### LeaveTypeFactory
```php
namespace Database\Factories;

use App\Models\LeaveType;
use Illuminate\Database\Eloquent\Factories\Factory;

class LeaveTypeFactory extends Factory
{
    protected $model = LeaveType::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Leave',
            'code' => strtoupper(fake()->unique()->lexify('LV-???')),
            'accrual_type' => 'annual',
            'max_days' => fake()->numberBetween(5, 30),
            'is_paid' => true,
            'carry_over_days' => fake()->numberBetween(0, 5),
        ];
    }

    public function vacation(): self
    {
        return $this->state(fn (array $attributes) => [
            'name' => 'Vacation Leave',
            'code' => 'VL',
            'max_days' => 15,
        ]);
    }

    public function sick(): self
    {
        return $this->state(fn (array $attributes) => [
            'name' => 'Sick Leave',
            'code' => 'SL',
            'max_days' => 15,
        ]);
    }
}
```

#### HolidayFactory
```php
namespace Database\Factories;

use App\Enums\HolidayType;
use App\Models\Holiday;
use Illuminate\Database\Eloquent\Factories\Factory;

class HolidayFactory extends Factory
{
    protected $model = Holiday::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Holiday',
            'date' => fake()->date('Y-m-d'),
            'type' => fake()->randomElement(HolidayType::cases()),
            'is_paid' => true,
        ];
    }

    public function regular(): self
    {
        return $this->state(fn (array $attributes) => [
            'type' => HolidayType::REGULAR,
        ]);
    }

    public function specialNonWorking(): self
    {
        return $this->state(fn (array $attributes) => [
            'type' => HolidayType::SPECIAL_NON_WORKING,
        ]);
    }
}
```

#### SalaryGradeFactory
```php
namespace Database\Factories;

use App\Models\SalaryGrade;
use Illuminate\Database\Eloquent\Factories\Factory;

class SalaryGradeFactory extends Factory
{
    protected $model = SalaryGrade::class;

    public function definition(): array
    {
        return [
            'name' => 'SG-' . fake()->unique()->numberBetween(1, 30),
            'min_salary' => fake()->numberBetween(15000, 30000),
            'max_salary' => fake()->numberBetween(31000, 80000),
            'step_count' => fake()->numberBetween(1, 8),
        ];
    }
}
```

#### DeductionTypeFactory
```php
namespace Database\Factories;

use App\Models\DeductionType;
use Illuminate\Database\Eloquent\Factories\Factory;

class DeductionTypeFactory extends Factory
{
    protected $model = DeductionType::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Deduction',
            'code' => strtoupper(fake()->unique()->lexify('DED-???')),
            'calculation_type' => fake()->randomElement(['fixed', 'percentage']),
            'rate' => fake()->randomFloat(4, 0.01, 0.15),
            'is_mandatory' => fake()->boolean(70),
        ];
    }

    public function sss(): self
    {
        return $this->state(fn (array $attributes) => [
            'name' => 'SSS Contribution',
            'code' => 'SSS',
            'calculation_type' => 'percentage',
            'rate' => 0.045,
            'is_mandatory' => true,
        ]);
    }

    public function philhealth(): self
    {
        return $this->state(fn (array $attributes) => [
            'name' => 'PhilHealth Contribution',
            'code' => 'PHIC',
            'calculation_type' => 'percentage',
            'rate' => 0.025,
            'is_mandatory' => true,
        ]);
    }

    public function pagibig(): self
    {
        return $this->state(fn (array $attributes) => [
            'name' => 'Pag-IBIG Contribution',
            'code' => 'HDMF',
            'calculation_type' => 'fixed',
            'rate' => 100,
            'is_mandatory' => true,
        ]);
    }
}
```

#### ShiftFactory
```php
namespace Database\Factories;

use App\Enums\ShiftType;
use App\Models\Shift;
use Illuminate\Database\Eloquent\Factories\Factory;

class ShiftFactory extends Factory
{
    protected $model = Shift::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Shift',
            'type' => fake()->randomElement(ShiftType::cases()),
            'start_time' => '08:00',
            'end_time' => '17:00',
            'grace_period_minutes' => 15,
        ];
    }
}
```

#### AssetCategoryFactory
```php
namespace Database\Factories;

use App\Models\AssetCategory;
use Illuminate\Database\Eloquent\Factories\Factory;

class AssetCategoryFactory extends Factory
{
    protected $model = AssetCategory::class;

    public function definition(): array
    {
        return [
            'name' => fake()->unique()->word() . ' Category',
            'description' => fake()->sentence(),
        ];
    }
}
```

#### AssetFactory
```php
namespace Database\Factories;

use App\Enums\AssetCondition;
use App\Enums\AssetStatus;
use App\Models\Asset;
use App\Models\AssetCategory;
use Illuminate\Database\Eloquent\Factories\Factory;

class AssetFactory extends Factory
{
    protected $model = Asset::class;

    public function definition(): array
    {
        return [
            'name' => fake()->word() . ' Asset',
            'asset_category_id' => AssetCategory::factory(),
            'serial_number' => strtoupper(fake()->unique()->bothify('AST-####-????')),
            'status' => AssetStatus::AVAILABLE,
            'condition' => fake()->randomElement(AssetCondition::cases()),
            'purchase_date' => fake()->date('Y-m-d', '-3 years'),
            'purchase_cost' => fake()->numberBetween(5000, 100000),
        ];
    }

    public function assigned(): self
    {
        return $this->state(fn (array $attributes) => [
            'status' => AssetStatus::ASSIGNED,
        ]);
    }
}
```

#### TrainingProgramFactory
```php
namespace Database\Factories;

use App\Enums\TrainingStatus;
use App\Models\TrainingProgram;
use Illuminate\Database\Eloquent\Factories\Factory;

class TrainingProgramFactory extends Factory
{
    protected $model = TrainingProgram::class;

    public function definition(): array
    {
        return [
            'title' => fake()->sentence(3),
            'provider' => fake()->company(),
            'start_date' => fake()->date('Y-m-d'),
            'end_date' => fake()->date('Y-m-d', '+1 month'),
            'cost' => fake()->numberBetween(1000, 50000),
            'status' => fake()->randomElement(TrainingStatus::cases()),
        ];
    }
}
```

#### JobRequisitionFactory
```php
namespace Database\Factories;

use App\Models\Department;
use App\Models\JobRequisition;
use App\Models\Position;
use Illuminate\Database\Eloquent\Factories\Factory;

class JobRequisitionFactory extends Factory
{
    protected $model = JobRequisition::class;

    public function definition(): array
    {
        return [
            'title' => fake()->jobTitle(),
            'department_id' => Department::factory(),
            'position_id' => Position::factory(),
            'headcount' => fake()->numberBetween(1, 5),
            'status' => 'open',
            'description' => fake()->paragraph(),
        ];
    }
}
```

#### JobPostingFactory
```php
namespace Database\Factories;

use App\Models\JobPosting;
use App\Models\JobRequisition;
use Illuminate\Database\Eloquent\Factories\Factory;

class JobPostingFactory extends Factory
{
    protected $model = JobPosting::class;

    public function definition(): array
    {
        return [
            'job_requisition_id' => JobRequisition::factory(),
            'platform' => fake()->randomElement(['LinkedIn', 'Indeed', 'JobStreet', 'Company Website']),
            'posted_date' => fake()->date('Y-m-d'),
            'expiry_date' => fake()->date('Y-m-d', '+1 month'),
            'is_active' => true,
        ];
    }
}
```

### Seeders

#### UserRoleSeeder
```php
namespace Database\Seeders;

use App\Enums\UserRole;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class UserRoleSeeder extends Seeder
{
    public function run(): void
    {
        // Create default users for each role
        User::firstOrCreate(
            ['email' => 'admin@company.com'],
            [
                'name' => 'Company Admin',
                'password' => Hash::make('password'),
                'role' => UserRole::COMPANY_ADMIN,
                'is_active' => true,
                'email_verified_at' => now(),
            ]
        );

        User::firstOrCreate(
            ['email' => 'hr-admin@company.com'],
            [
                'name' => 'HR Admin',
                'password' => Hash::make('password'),
                'role' => UserRole::HR_ADMIN,
                'is_active' => true,
                'email_verified_at' => now(),
            ]
        );

        User::firstOrCreate(
            ['email' => 'hr-staff@company.com'],
            [
                'name' => 'HR Staff',
                'password' => Hash::make('password'),
                'role' => UserRole::HR_STAFF,
                'is_active' => true,
                'email_verified_at' => now(),
            ]
        );

        User::firstOrCreate(
            ['email' => 'payroll@company.com'],
            [
                'name' => 'Payroll Officer',
                'password' => Hash::make('password'),
                'role' => UserRole::PAYROLL_OFFICER,
                'is_active' => true,
                'email_verified_at' => now(),
            ]
        );

        User::firstOrCreate(
            ['email' => 'manager@company.com'],
            [
                'name' => 'Manager',
                'password' => Hash::make('password'),
                'role' => UserRole::MANAGER,
                'is_active' => true,
                'email_verified_at' => now(),
            ]
        );
    }
}
```

#### CompanySettingsSeeder
```php
namespace Database\Seeders;

use App\Models\CompanySettings;
use Illuminate\Database\Seeder;

class CompanySettingsSeeder extends Seeder
{
    public function run(): void
    {
        CompanySettings::firstOrCreate(
            ['id' => 1],
            [
                'name' => config('app.name', 'HRIS System'),
                'address' => '123 Business District',
                'city' => 'Manila',
                'country' => 'Philippines',
                'timezone' => 'Asia/Manila',
                'locale' => 'en',
                'currency' => 'PHP',
                'tin_no' => '000-000-000-000',
            ]
        );
    }
}
```

#### BranchSeeder
```php
namespace Database\Seeders;

use App\Models\Branch;
use Illuminate\Database\Seeder;

class BranchSeeder extends Seeder
{
    public function run(): void
    {
        $branches = [
            ['code' => 'MAIN', 'name' => 'Main Office', 'city' => 'Manila'],
            ['code' => 'CEBU', 'name' => 'Cebu Branch', 'city' => 'Cebu'],
            ['code' => 'DAVAO', 'name' => 'Davao Branch', 'city' => 'Davao'],
            ['code' => 'BAGUIO', 'name' => 'Baguio Branch', 'city' => 'Baguio'],
            ['code' => 'ILOILO', 'name' => 'Iloilo Branch', 'city' => 'Iloilo'],
        ];

        foreach ($branches as $branch) {
            Branch::firstOrCreate(
                ['code' => $branch['code']],
                [
                    'name' => $branch['name'],
                    'address' => fake()->address(),
                    'city' => $branch['city'],
                    'timezone' => 'Asia/Manila',
                    'is_active' => true,
                ]
            );
        }
    }
}
```

#### DepartmentSeeder
```php
namespace Database\Seeders;

use App\Models\Department;
use Illuminate\Database\Seeder;

class DepartmentSeeder extends Seeder
{
    public function run(): void
    {
        $departments = [
            ['name' => 'Human Resources', 'code' => 'HR'],
            ['name' => 'Finance', 'code' => 'FIN'],
            ['name' => 'Information Technology', 'code' => 'IT'],
            ['name' => 'Marketing', 'code' => 'MKT'],
            ['name' => 'Operations', 'code' => 'OPS'],
            ['name' => 'Sales', 'code' => 'SLS'],
            ['name' => 'Customer Service', 'code' => 'CS'],
            ['name' => 'Legal', 'code' => 'LGL'],
            ['name' => 'Administration', 'code' => 'ADM'],
            ['name' => 'Research & Development', 'code' => 'RND'],
        ];

        foreach ($departments as $department) {
            Department::firstOrCreate(
                ['code' => $department['code']],
                [
                    'name' => $department['name'],
                    'description' => fake()->sentence(),
                ]
            );
        }
    }
}
```

#### PositionSeeder
```php
namespace Database\Seeders;

use App\Models\Department;
use App\Models\Position;
use Illuminate\Database\Seeder;

class PositionSeeder extends Seeder
{
    public function run(): void
    {
        $positions = [
            ['title' => 'HR Manager', 'department' => 'HR'],
            ['title' => 'HR Specialist', 'department' => 'HR'],
            ['title' => 'Finance Manager', 'department' => 'FIN'],
            ['title' => 'Accountant', 'department' => 'FIN'],
            ['title' => 'IT Manager', 'department' => 'IT'],
            ['title' => 'Software Developer', 'department' => 'IT'],
            ['title' => 'Marketing Manager', 'department' => 'MKT'],
            ['title' => 'Marketing Specialist', 'department' => 'MKT'],
            ['title' => 'Operations Manager', 'department' => 'OPS'],
            ['title' => 'Operations Coordinator', 'department' => 'OPS'],
            ['title' => 'Sales Manager', 'department' => 'SLS'],
            ['title' => 'Sales Representative', 'department' => 'SLS'],
            ['title' => 'Customer Service Lead', 'department' => 'CS'],
            ['title' => 'Customer Service Representative', 'department' => 'CS'],
            ['title' => 'Legal Counsel', 'department' => 'LGL'],
            ['title' => 'Admin Assistant', 'department' => 'ADM'],
            ['title' => 'R&D Lead', 'department' => 'RND'],
            ['title' => 'Research Analyst', 'department' => 'RND'],
        ];

        foreach ($positions as $position) {
            $department = Department::where('code', $position['department'])->first();

            Position::firstOrCreate(
                ['title' => $position['title'], 'department_id' => $department->id],
                [
                    'description' => fake()->sentence(),
                ]
            );
        }
    }
}
```

#### EmployeeSeeder
```php
namespace Database\Seeders;

use App\Enums\CivilStatus;
use App\Enums\EmploymentStatus;
use App\Enums\EmploymentType;
use App\Enums\Gender;
use App\Models\Branch;
use App\Models\Department;
use App\Models\Employee;
use App\Models\Position;
use App\Models\User;
use Illuminate\Database\Seeder;

class EmployeeSeeder extends Seeder
{
    public function run(): void
    {
        $branches = Branch::all();
        $departments = Department::all();
        $positions = Position::all();

        $adminUser = User::factory()->companyAdmin()->create();

        $hrUser = User::factory()->hrAdmin()->create();

        $payrollUser = User::factory()->payrollOfficer()->create();

        $managerUser = User::factory()->manager()->create();

        for ($i = 0; $i < 50; $i++) {
            $branch = $branches->random();
            $department = $departments->random();
            $position = $positions->where('department_id', $department->id)->first() ?? $positions->random();

            Employee::factory()->create([
                'branch_id' => $branch->id,
                'department_id' => $department->id,
                'position_id' => $position->id,
                'employment_status' => EmploymentStatus::ACTIVE,
                'employment_type' => $i < 40 ? EmploymentType::REGULAR : EmploymentType::PROBATIONARY,
                'is_field_employee' => $i % 10 === 0,
            ]);
        }
    }
}
```

#### ShiftSeeder
```php
namespace Database\Seeders;

use App\Enums\ShiftType;
use App\Models\Shift;
use Illuminate\Database\Seeder;

class ShiftSeeder extends Seeder
{
    public function run(): void
    {
        $shifts = [
            ['name' => 'Morning Shift', 'type' => ShiftType::FIXED, 'start_time' => '06:00', 'end_time' => '14:00'],
            ['name' => 'Mid Shift', 'type' => ShiftType::FIXED, 'start_time' => '14:00', 'end_time' => '22:00'],
            ['name' => 'Night Shift', 'type' => ShiftType::FIXED, 'start_time' => '22:00', 'end_time' => '06:00'],
            ['name' => 'Flexi Shift', 'type' => ShiftType::FLEXIBLE, 'start_time' => '08:00', 'end_time' => '17:00'],
            ['name' => 'Rotating Shift A', 'type' => ShiftType::ROTATING, 'start_time' => '08:00', 'end_time' => '17:00'],
        ];

        foreach ($shifts as $shift) {
            Shift::firstOrCreate(
                ['name' => $shift['name']],
                [
                    'type' => $shift['type'],
                    'start_time' => $shift['start_time'],
                    'end_time' => $shift['end_time'],
                    'grace_period_minutes' => 15,
                ]
            );
        }
    }
}
```

#### LeaveTypeSeeder
```php
namespace Database\Seeders;

use App\Models\LeaveType;
use Illuminate\Database\Seeder;

class LeaveTypeSeeder extends Seeder
{
    public function run(): void
    {
        $leaveTypes = [
            ['name' => 'Vacation Leave', 'code' => 'VL', 'accrual_type' => 'annual', 'max_days' => 15, 'is_paid' => true, 'carry_over_days' => 5],
            ['name' => 'Sick Leave', 'code' => 'SL', 'accrual_type' => 'annual', 'max_days' => 15, 'is_paid' => true, 'carry_over_days' => 5],
            ['name' => 'Maternity Leave', 'code' => 'ML', 'accrual_type' => 'on_hire', 'max_days' => 105, 'is_paid' => true, 'carry_over_days' => 0],
            ['name' => 'Paternity Leave', 'code' => 'PL', 'accrual_type' => 'on_hire', 'max_days' => 7, 'is_paid' => true, 'carry_over_days' => 0],
            ['name' => 'Bereavement Leave', 'code' => 'BL', 'accrual_type' => 'annual', 'max_days' => 5, 'is_paid' => true, 'carry_over_days' => 0],
            ['name' => 'Emergency Leave', 'code' => 'EL', 'accrual_type' => 'annual', 'max_days' => 3, 'is_paid' => true, 'carry_over_days' => 0],
        ];

        foreach ($leaveTypes as $leaveType) {
            LeaveType::firstOrCreate(
                ['code' => $leaveType['code']],
                $leaveType
            );
        }
    }
}
```

#### HolidaySeeder
```php
namespace Database\Seeders;

use App\Enums\HolidayType;
use App\Models\Holiday;
use Illuminate\Database\Seeder;

class HolidaySeeder extends Seeder
{
    public function run(): void
    {
        $holidays = [
            ['name' => 'New Year\'s Day', 'date' => '2026-01-01', 'type' => HolidayType::REGULAR],
            ['name' => 'Maundy Thursday', 'date' => '2026-04-02', 'type' => HolidayType::REGULAR],
            ['name' => 'Good Friday', 'date' => '2026-04-03', 'type' => HolidayType::REGULAR],
            ['name' => 'Araw ng Kagitingan', 'date' => '2026-04-09', 'type' => HolidayType::REGULAR],
            ['name' => 'Labor Day', 'date' => '2026-05-01', 'type' => HolidayType::REGULAR],
            ['name' => 'Independence Day', 'date' => '2026-06-12', 'type' => HolidayType::REGULAR],
            ['name' => 'National Heroes Day', 'date' => '2026-08-31', 'type' => HolidayType::REGULAR],
            ['name' => 'Bonifacio Day', 'date' => '2026-11-30', 'type' => HolidayType::REGULAR],
            ['name' => 'Christmas Day', 'date' => '2026-12-25', 'type' => HolidayType::REGULAR],
            ['name' => 'Rizal Day', 'date' => '2026-12-30', 'type' => HolidayType::REGULAR],
            ['name' => 'Chinese New Year', 'date' => '2026-02-17', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'EDSA Revolution Anniversary', 'date' => '2026-02-25', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'Black Saturday', 'date' => '2026-04-04', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'Ninoy Aquino Day', 'date' => '2026-08-21', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'All Saints\' Day', 'date' => '2026-11-01', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'Feast of the Immaculate Conception', 'date' => '2026-12-08', 'type' => HolidayType::SPECIAL_NON_WORKING],
            ['name' => 'Last Day of the Year', 'date' => '2026-12-31', 'type' => HolidayType::SPECIAL_NON_WORKING],
        ];

        foreach ($holidays as $holiday) {
            Holiday::firstOrCreate(
                ['date' => $holiday['date'], 'name' => $holiday['name']],
                $holiday
            );
        }
    }
}
```

#### SalaryGradeSeeder
```php
namespace Database\Seeders;

use App\Models\SalaryGrade;
use Illuminate\Database\Seeder;

class SalaryGradeSeeder extends Seeder
{
    public function run(): void
    {
        for ($i = 1; $i <= 30; $i++) {
            SalaryGrade::firstOrCreate(
                ['name' => 'SG-' . $i],
                [
                    'min_salary' => 15000 + ($i - 1) * 2000,
                    'max_salary' => 30000 + ($i - 1) * 2500,
                    'step_count' => 8,
                ]
            );
        }
    }
}
```

#### DeductionTypeSeeder
```php
namespace Database\Seeders;

use App\Models\DeductionType;
use Illuminate\Database\Seeder;

class DeductionTypeSeeder extends Seeder
{
    public function run(): void
    {
        $deductions = [
            ['name' => 'SSS Contribution', 'code' => 'SSS', 'calculation_type' => 'percentage', 'rate' => 0.045, 'is_mandatory' => true],
            ['name' => 'PhilHealth Contribution', 'code' => 'PHIC', 'calculation_type' => 'percentage', 'rate' => 0.025, 'is_mandatory' => true],
            ['name' => 'Pag-IBIG Contribution', 'code' => 'HDMF', 'calculation_type' => 'fixed', 'rate' => 100, 'is_mandatory' => true],
            ['name' => 'Withholding Tax', 'code' => 'TAX', 'calculation_type' => 'percentage', 'rate' => 0.0, 'is_mandatory' => true],
            ['name' => 'SSS Loan', 'code' => 'SSS-LOAN', 'calculation_type' => 'fixed', 'rate' => 0, 'is_mandatory' => false],
            ['name' => 'Pag-IBIG Loan', 'code' => 'HDMF-LOAN', 'calculation_type' => 'fixed', 'rate' => 0, 'is_mandatory' => false],
            ['name' => 'Company Loan', 'code' => 'COMP-LOAN', 'calculation_type' => 'fixed', 'rate' => 0, 'is_mandatory' => false],
        ];

        foreach ($deductions as $deduction) {
            DeductionType::firstOrCreate(
                ['code' => $deduction['code']],
                $deduction
            );
        }
    }
}
```

#### AssetCategorySeeder
```php
namespace Database\Seeders;

use App\Models\AssetCategory;
use Illuminate\Database\Seeder;

class AssetCategorySeeder extends Seeder
{
    public function run(): void
    {
        $categories = [
            ['name' => 'Laptop', 'description' => 'Company-issued laptops'],
            ['name' => 'Desktop', 'description' => 'Desktop computers'],
            ['name' => 'Monitor', 'description' => 'External monitors'],
            ['name' => 'Keyboard', 'description' => 'Keyboards'],
            ['name' => 'Mouse', 'description' => 'Mice'],
            ['name' => 'Headset', 'description' => 'Headsets and headphones'],
            ['name' => 'Mobile Phone', 'description' => 'Company-issued mobile phones'],
            ['name' => 'Tablet', 'description' => 'Tablets'],
            ['name' => 'Vehicle', 'description' => 'Company vehicles'],
            ['name' => 'Furniture', 'description' => 'Office furniture'],
        ];

        foreach ($categories as $category) {
            AssetCategory::firstOrCreate(
                ['name' => $category['name']],
                $category
            );
        }
    }
}
```

#### AssetSeeder
```php
namespace Database\Seeders;

use App\Models\Asset;
use App\Models\AssetCategory;
use Illuminate\Database\Seeder;

class AssetSeeder extends Seeder
{
    public function run(): void
    {
        $categories = AssetCategory::all();

        foreach ($categories as $category) {
            for ($i = 0; $i < 5; $i++) {
                Asset::create([
                    'name' => $category->name . ' #' . ($i + 1),
                    'asset_category_id' => $category->id,
                    'serial_number' => strtoupper($category->code ?? 'AST') . '-' . fake()->unique()->numerify('####-????'),
                    'status' => 'available',
                    'condition' => fake()->randomElement(['new', 'good', 'fair']),
                    'purchase_date' => fake()->date('Y-m-d', '-3 years'),
                    'purchase_cost' => fake()->numberBetween(5000, 100000),
                ]);
            }
        }
    }
}
```

#### TrainingProgramSeeder
```php
namespace Database\Seeders;

use App\Enums\TrainingStatus;
use App\Models\TrainingProgram;
use Illuminate\Database\Seeder;

class TrainingProgramSeeder extends Seeder
{
    public function run(): void
    {
        $programs = [
            ['title' => 'Leadership Excellence Program', 'provider' => 'Management Institute'],
            ['title' => 'Advanced Excel for HR', 'provider' => 'Excel Experts PH'],
            ['title' => 'Effective Communication Skills', 'provider' => 'SpeakWell Philippines'],
            ['title' => 'Data Privacy Act Compliance', 'provider' => 'Privacy Experts Inc.'],
            ['title' => 'Project Management Fundamentals', 'provider' => 'PMI Philippines'],
            ['title' => 'Customer Service Excellence', 'provider' => 'ServicePro Training'],
            ['title' => 'Cybersecurity Awareness', 'provider' => 'SecureIT Philippines'],
            ['title' => 'Financial Literacy for Employees', 'provider' => 'FinancePH'],
        ];

        foreach ($programs as $program) {
            TrainingProgram::create([
                'title' => $program['title'],
                'provider' => $program['provider'],
                'start_date' => fake()->date('Y-m-d', '+1 month'),
                'end_date' => fake()->date('Y-m-d', '+2 months'),
                'cost' => fake()->numberBetween(1000, 50000),
                'status' => TrainingStatus::SCHEDULED,
            ]);
        }
    }
}
```

#### JobRequisitionSeeder
```php
namespace Database\Seeders;

use App\Models\Department;
use App\Models\JobRequisition;
use App\Models\Position;
use Illuminate\Database\Seeder;

class JobRequisitionSeeder extends Seeder
{
    public function run(): void
    {
        $departments = Department::all();

        foreach ($departments as $department) {
            $position = Position::where('department_id', $department->id)->first();

            if ($position) {
                JobRequisition::create([
                    'title' => $position->title,
                    'department_id' => $department->id,
                    'position_id' => $position->id,
                    'headcount' => fake()->numberBetween(1, 3),
                    'status' => 'open',
                    'description' => fake()->paragraph(),
                ]);
            }
        }
    }
}
```

#### JobPostingSeeder
```php
namespace Database\Seeders;

use App\Models\JobPosting;
use App\Models\JobRequisition;
use Illuminate\Database\Seeder;

class JobPostingSeeder extends Seeder
{
    public function run(): void
    {
        $requisitions = JobRequisition::all();

        foreach ($requisitions as $requisition) {
            JobPosting::create([
                'job_requisition_id' => $requisition->id,
                'platform' => fake()->randomElement(['LinkedIn', 'Indeed', 'JobStreet', 'Company Website']),
                'posted_date' => now()->subDays(fake()->numberBetween(1, 30)),
                'expiry_date' => now()->addDays(30),
                'is_active' => true,
            ]);
        }
    }
}
```

#### DemoSeeder (Master Seeder)
```php
namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DemoSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            UserRoleSeeder::class,
            CompanySettingsSeeder::class,
            BranchSeeder::class,
            DepartmentSeeder::class,
            PositionSeeder::class,
            ShiftSeeder::class,
            LeaveTypeSeeder::class,
            HolidaySeeder::class,
            SalaryGradeSeeder::class,
            DeductionTypeSeeder::class,
            AssetCategorySeeder::class,
            AssetSeeder::class,
            TrainingProgramSeeder::class,
            EmployeeSeeder::class,
            JobRequisitionSeeder::class,
            JobPostingSeeder::class,
        ]);
    }
}
```

### Filament Panel Configuration

#### AdminPanelProvider
```php
namespace App\Providers\Filament;

use App\Filament\Pages\AdminDashboard;
use Filament\Http\Middleware\Authenticate;
use Filament\Http\Middleware\DisableBladeIconComponents;
use Filament\Http\Middleware\DispatchServingFilamentEvent;
use Filament\Panel;
use Filament\PanelProvider;
use Filament\Support\Colors\Color;
use Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse;
use Illuminate\Cookie\Middleware\EncryptCookies;
use Illuminate\Foundation\Http\Middleware\PreventRequestForgery;
use Illuminate\Routing\Middleware\SubstituteBindings;
use Illuminate\Session\Middleware\AuthenticateSession;
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;

class AdminPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->default()
            ->id('admin')
            ->path('admin')
            ->login()
            ->passwordReset()
            ->emailVerification()
            ->profile()
            ->unsavedChangesAlerts()
            ->colors([
                'primary' => Color::Amber,
            ])
            ->discoverResources(in: app_path('Filament/Resources'), for: 'App\\Filament\\Resources')
            ->discoverPages(in: app_path('Filament/Pages'), for: 'App\\Filament\\Pages')
            ->pages([
                AdminDashboard::class,
            ])
            ->middleware([
                EncryptCookies::class,
                AddQueuedCookiesToResponse::class,
                StartSession::class,
                AuthenticateSession::class,
                ShareErrorsFromSession::class,
                PreventRequestForgery::class,
                SubstituteBindings::class,
                DisableBladeIconComponents::class,
                DispatchServingFilamentEvent::class,
            ])
            ->authMiddleware([
                Authenticate::class,
            ]);
    }
}
```

#### PortalPanelProvider
```php
namespace App\Providers\Filament;

use Filament\Http\Middleware\Authenticate;
use Filament\Http\Middleware\DisableBladeIconComponents;
use Filament\Http\Middleware\DispatchServingFilamentEvent;
use Filament\Panel;
use Filament\PanelProvider;
use Filament\Support\Colors\Color;
use Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse;
use Illuminate\Cookie\Middleware\EncryptCookies;
use Illuminate\Foundation\Http\Middleware\PreventRequestForgery;
use Illuminate\Routing\Middleware\SubstituteBindings;
use Illuminate\Session\Middleware\AuthenticateSession;
use Illuminate\Session\Middleware\StartSession;
use Illuminate\View\Middleware\ShareErrorsFromSession;

class PortalPanelProvider extends PanelProvider
{
    public function panel(Panel $panel): Panel
    {
        return $panel
            ->id('portal')
            ->path('portal')
            ->login()
            ->passwordReset()
            ->profile()
            ->unsavedChangesAlerts()
            ->colors([
                'primary' => Color::Blue,
            ])
            ->discoverResources(in: app_path('Filament/Portal/Resources'), for: 'App\\Filament\\Portal\\Resources')
            ->discoverPages(in: app_path('Filament/Portal/Pages'), for: 'App\\Filament\\Portal\\Pages')
            ->middleware([
                EncryptCookies::class,
                AddQueuedCookiesToResponse::class,
                StartSession::class,
                AuthenticateSession::class,
                ShareErrorsFromSession::class,
                PreventRequestForgery::class,
                SubstituteBindings::class,
                DisableBladeIconComponents::class,
                DispatchServingFilamentEvent::class,
            ])
            ->authMiddleware([
                Authenticate::class,
            ]);
    }
}
```

### PWA Configuration

#### Vite Configuration (for both panels)
```javascript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
    plugins: [
        laravel({
            input: [
                'resources/css/admin.css',
                'resources/js/admin.js',
                'resources/css/portal.css',
                'resources/js/portal.js',
            ],
            refresh: true,
        }),
        VitePWA({
            registerType: 'autoUpdate',
            includeAssets: ['favicon.ico', 'apple-touch-icon.png'],
            manifest: {
                name: 'HRIS Admin',
                short_name: 'HRIS Admin',
                description: 'HRIS Admin Panel',
                theme_color: '#f59e0b',
                background_color: '#ffffff',
                display: 'standalone',
                orientation: 'portrait-primary',
                icons: [
                    {
                        src: '/icon-192x192.png',
                        sizes: '192x192',
                        type: 'image/png',
                    },
                    {
                        src: '/icon-512x512.png',
                        sizes: '512x512',
                        type: 'image/png',
                    },
                ],
            },
        }),
    ],
});
```

### Hashid Configuration

#### Config/hashids.php
```php
return [
    'default' => 'main',

    'connections' => [
        'user' => [
            'salt' => env('HASHID_SALT', 'your-secret-salt') . 'user',
            'length' => 6,
            'alphabet' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890',
        ],
        'employee' => [
            'salt' => env('HASHID_SALT', 'your-secret-salt') . 'employee',
            'length' => 8,
            'alphabet' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890',
        ],
        'branch' => [
            'salt' => env('HASHID_SALT', 'your-secret-salt') . 'branch',
            'length' => 6,
            'alphabet' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890',
        ],
    ],
];
```

### Tests (Pest 4)

#### Pest.php (Global Test Configuration)
```php
<?php

use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

uses(TestCase::class, RefreshDatabase::class)->in('Feature', 'Unit');
```

#### FoundationTest.php
```php
<?php

use App\Enums\UserRole;
use App\Models\User;

describe('Admin Panel Access', function () {
    it('allows company admin to access admin panel', function () {
        $user = User::factory()->companyAdmin()->create();
        $this->actingAs($user)->get('/admin')->assertOk();
    });

    it('allows hr admin to access admin panel', function () {
        $user = User::factory()->hrAdmin()->create();
        $this->actingAs($user)->get('/admin')->assertOk();
    });

    it('allows payroll officer to access admin panel', function () {
        $user = User::factory()->payrollOfficer()->create();
        $this->actingAs($user)->get('/admin')->assertOk();
    });

    it('allows manager to access admin panel', function () {
        $user = User::factory()->manager()->create();
        $this->actingAs($user)->get('/admin')->assertOk();
    });

    it('denies employee access to admin panel', function () {
        $user = User::factory()->employee()->create();
        $this->actingAs($user)->get('/admin')->assertRedirect();
    });
});

describe('Portal Panel Access', function () {
    it('allows employee to access portal panel', function () {
        $user = User::factory()->employee()->create();
        $this->actingAs($user)->get('/portal')->assertOk();
    });

    it('denies company admin access to portal panel', function () {
        $user = User::factory()->companyAdmin()->create();
        $this->actingAs($user)->get('/portal')->assertRedirect();
    });
});

describe('Authentication', function () {
    it('denies inactive user login', function () {
        $user = User::factory()->create(['is_active' => false]);

        $this->post('/admin/login', [
            'email' => $user->email,
            'password' => 'password',
        ])->assertSessionHasErrors();

        $this->assertGuest();
    });

    it('allows active user login', function () {
        $user = User::factory()->create(['is_active' => true]);

        $this->post('/admin/login', [
            'email' => $user->email,
            'password' => 'password',
        ])->assertRedirect();

        $this->assertAuthenticated();
    });
});

describe('Company Settings', function () {
    it('enforces singleton pattern', function () {
        $this->seed(\Database\Seeders\CompanySettingsSeeder::class);

        $settings = \App\Models\CompanySettings::current();

        expect($settings)->not->toBeNull()
            ->and($settings->id)->toBe(1);
    });

    it('throws exception when no settings exist', function () {
        expect(fn () => \App\Models\CompanySettings::current())
            ->toThrow(\Illuminate\Database\Eloquent\ModelNotFoundException::class);
    });
});

describe('HashidService', function () {
    it('encodes and decodes correctly', function () {
        $service = app(\App\Services\HashidService::class);
        $id = 123;

        $hash = $service->encode($id, 'user');
        $decoded = $service->decode($hash, 'user');

        expect($decoded)->toBe($id)
            ->and($hash)->not->toBe((string) $id);
    });

    it('returns null for invalid hash', function () {
        $service = app(\App\Services\HashidService::class);

        expect($service->decode('invalid-hash', 'user'))->toBeNull();
    });

    it('returns null for empty string', function () {
        $service = app(\App\Services\HashidService::class);

        expect($service->decode('', 'user'))->toBeNull();
    });

    it('returns null for wrong connection', function () {
        $service = app(\App\Services\HashidService::class);
        $hash = $service->encode(123, 'user');

        expect($service->decode($hash, 'branch'))->not->toBe(123);
    });
});

describe('GeofenceService', function () {
    it('returns true when branch has no geofence', function () {
        $branch = \App\Models\Branch::factory()->create();
        $service = app(\App\Services\GeofenceService::class);

        expect($service->check(14.5995, 120.9842, $branch))->toBeTrue();
    });

    it('returns true when within geofence', function () {
        $branch = \App\Models\Branch::factory()->withGeofence()->create([
            'geofence_lat' => 14.5995,
            'geofence_lng' => 120.9842,
            'geofence_radius_meters' => 100,
        ]);

        $service = app(\App\Services\GeofenceService::class);

        expect($service->check(14.5995, 120.9842, $branch))->toBeTrue();
    });

    it('returns false when outside geofence', function () {
        $branch = \App\Models\Branch::factory()->withGeofence()->create([
            'geofence_lat' => 14.5995,
            'geofence_lng' => 120.9842,
            'geofence_radius_meters' => 100,
        ]);

        $service = app(\App\Services\GeofenceService::class);

        expect($service->check(14.6100, 121.0000, $branch))->toBeFalse();
    });

    it('handles exact boundary', function () {
        $branch = \App\Models\Branch::factory()->withGeofence()->create([
            'geofence_lat' => 14.5995,
            'geofence_lng' => 120.9842,
            'geofence_radius_meters' => 100,
        ]);

        $service = app(\App\Services\GeofenceService::class);

        // Approximately 100 meters away
        expect($service->check(14.6004, 120.9842, $branch))->toBeTrue();
    });

    it('handles zero radius', function () {
        $branch = \App\Models\Branch::factory()->withGeofence()->create([
            'geofence_lat' => 14.5995,
            'geofence_lng' => 120.9842,
            'geofence_radius_meters' => 0,
        ]);

        $service = app(\App\Services\GeofenceService::class);

        expect($service->check(14.5995, 120.9842, $branch))->toBeTrue()
            ->and($service->check(14.5996, 120.9843, $branch))->toBeFalse();
    });
});

describe('AuthService', function () {
    it('logs in active user', function () {
        $user = User::factory()->create(['is_active' => true]);
        $service = app(\App\Services\AuthService::class);

        expect($service->login([
            'email' => $user->email,
            'password' => 'password',
        ]))->toBeTrue();

        $this->assertAuthenticatedAs($user);
    });

    it('rejects inactive user', function () {
        $user = User::factory()->create(['is_active' => false]);
        $service = app(\App\Services\AuthService::class);

        expect($service->login([
            'email' => $user->email,
            'password' => 'password',
        ]))->toBeFalse();

        $this->assertGuest();
    });

    it('rejects invalid credentials', function () {
        $service = app(\App\Services\AuthService::class);

        expect($service->login([
            'email' => 'nonexistent@example.com',
            'password' => 'wrong-password',
        ]))->toBeFalse();
    });
});
```

#### MigrationTest.php
```php
<?php

use Illuminate\Support\Facades\Schema;

describe('Migrations', function () {
    it('creates users table with correct columns', function () {
        expect(Schema::hasTable('users'))->toBeTrue()
            ->and(Schema::hasColumns('users', [
                'id', 'name', 'email', 'password', 'employee_id',
                'is_active', 'last_login_at', 'created_at', 'updated_at', 'deleted_at',
            ]))->toBeTrue();
    });

    it('creates company_settings table with correct columns', function () {
        expect(Schema::hasTable('company_settings'))->toBeTrue()
            ->and(Schema::hasColumns('company_settings', [
                'id', 'name', 'address', 'city', 'country',
                'timezone', 'locale', 'currency', 'tin_no', 'created_at', 'updated_at',
            ]))->toBeTrue();
    });

    it('creates branches table with correct columns', function () {
        expect(Schema::hasTable('branches'))->toBeTrue()
            ->and(Schema::hasColumns('branches', [
                'id', 'name', 'code', 'address', 'city', 'timezone',
                'geofence_lat', 'geofence_lng', 'geofence_radius_meters',
                'is_active', 'created_at', 'updated_at', 'deleted_at',
            ]))->toBeTrue();
    });

    it('enforces unique email on users', function () {
        \App\Models\User::factory()->create(['email' => 'test@example.com']);

        expect(fn () => \App\Models\User::factory()->create(['email' => 'test@example.com']))
            ->toThrow(\Illuminate\Database\QueryException::class);
    });

    it('enforces unique code on branches', function () {
        \App\Models\Branch::factory()->create(['code' => 'TEST']);

        expect(fn () => \App\Models\Branch::factory()->create(['code' => 'TEST']))
            ->toThrow(\Illuminate\Database\QueryException::class);
    });
});
```

#### FactoryTest.php
```php
<?php

use App\Enums\UserRole;
use App\Models\Branch;
use App\Models\CompanySettings;
use App\Models\User;

describe('UserFactory', function () {
    it('creates a user with default state', function () {
        $user = User::factory()->create();

        expect($user)->toBeInstanceOf(User::class)
            ->and($user->name)->not->toBeEmpty()
            ->and($user->email)->not->toBeEmpty()
            ->and($user->is_active)->toBeTrue();
    });

    it('creates a company admin with role', function () {
        $user = User::factory()->companyAdmin()->create();

        expect($user->role)->toBe(UserRole::COMPANY_ADMIN);
    });

    it('creates an employee with role', function () {
        $user = User::factory()->employee()->create();

        expect($user->role)->toBe(UserRole::EMPLOYEE);
    });
});

describe('BranchFactory', function () {
    it('creates a branch with default state', function () {
        $branch = Branch::factory()->create();

        expect($branch)->toBeInstanceOf(Branch::class)
            ->and($branch->name)->not->toBeEmpty()
            ->and($branch->code)->not->toBeEmpty()
            ->and($branch->is_active)->toBeTrue();
    });

    it('creates a branch with geofence', function () {
        $branch = Branch::factory()->withGeofence()->create();

        expect($branch->geofence_lat)->not->toBeNull()
            ->and($branch->geofence_lng)->not->toBeNull()
            ->and($branch->geofence_radius_meters)->toBe(100);
    });
});

describe('CompanySettingsFactory', function () {
    it('creates company settings', function () {
        $settings = CompanySettings::factory()->create();

        expect($settings)->toBeInstanceOf(CompanySettings::class)
            ->and($settings->name)->not->toBeEmpty()
            ->and($settings->country)->toBe('Philippines')
            ->and($settings->timezone)->toBe('Asia/Manila');
    });
});
```

#### ArchTest.php
```php
<?php

arch('models extend base model')
    ->expect('App\Models')
    ->toExtend('Illuminate\Database\Eloquent\Model');

arch('services are in service namespace')
    ->expect('App\Services')
    ->toBeClasses();

arch('enums are in enum namespace')
    ->expect('App\Enums')
    ->toBeEnums();

arch('no debugging functions')
    ->expect(['dd', 'dump', 'ray', 'var_dump'])
    ->not->toBeUsed();

arch('controllers use proper naming')
    ->expect('App\Http\Controllers')
    ->toHaveSuffix('Controller');
```

### Phase 1 Deliverables
- ✅ Laravel 13 project with PHP 8.4+
- ✅ Two Filament 5 panels (Admin + Portal)
- ✅ User authentication with role-based access
- ✅ Company settings singleton
- ✅ Branch management with geofencing support
- ✅ Hashid integration for public ID obfuscation
- ✅ Laravel AI SDK integration (resume parsing, sentiment analysis, ticket routing)
- ✅ Role and permission seeding
- ✅ Demo seeders for all core modules
- ✅ PWA configuration for both panels
- ✅ Comprehensive Pest 4 test coverage (models, migrations, factories, services)
- ✅ Architecture tests
- ✅ CSRF protection using Laravel 13's PreventRequestForgery

---

## PHASE 2 — Company & Branch Management

### Phase Overview
Build out company settings management and branch CRUD with geofencing capabilities using Filament v5 Simple Resources.

### Models

#### Branch Model
```php
namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Attributes\Table;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

#[Table('branches')]
#[Fillable(['name', 'code', 'address', 'city', 'timezone', 'geofence_lat', 'geofence_lng', 'geofence_radius_meters', 'is_active'])]
class Branch extends Model
{
    use SoftDeletes;

    protected function casts(): array
    {
        return [
            'geofence_lat' => 'decimal:7',
            'geofence_lng' => 'decimal:7',
            'geofence_radius_meters' => 'integer',
            'is_active' => 'boolean',
        ];
    }

    public function employees()
    {
        return $this->hasMany(Employee::class);
    }

    public function departments()
    {
        return $this->hasMany(Department::class);
    }

    public function hasGeofence(): bool
    {
        return $this->geofence_lat !== null
            && $this->geofence_lng !== null
            && $this->geofence_radius_meters !== null;
    }
}
```

### Filament Resources

#### BranchResource (Simple Resource — Modal)
```php
namespace App\Filament\Resources;

use App\Filament\Resources\BranchResource\Pages;
use App\Models\Branch;
use Filament\Forms;
use Filament\Resources\Resource;
use Filament\Schemas\Schema;
use Filament\Tables;
use Filament\Tables\Table;
use Illuminate\Validation\Rules\Unique;

class BranchResource extends Resource
{
    protected static ?string $model = Branch::class;

    protected static ?string $navigationIcon = 'heroicon-o-building-office-2';

    protected static ?string $navigationGroup = '👥 Employee Management';

    protected static ?int $navigationSort = 3;

    public static function form(Schema $schema): Schema
    {
        return $schema
            ->components([
                Forms\Components\TextInput::make('name')
                    ->required()
                    ->maxLength(255),
                Forms\Components\TextInput::make('code')
                    ->required()
                    ->unique(ignoreRecord: true)
                    ->maxLength(50),
                Forms\Components\Textarea::make('address')
                    ->required()
                    ->rows(3),
                Forms\Components\TextInput::make('city')
                    ->required()
                    ->maxLength(100),
                Forms\Components\TextInput::make('timezone')
                    ->default('Asia/Manila')
                    ->required(),
                Forms\Components\Toggle::make('is_active')
                    ->default(true),
            ]);
    }

    public static function table(Table $table): Table
    {
        return $table
            ->columns([
                Tables\Columns\TextColumn::make('code')
                    ->searchable(),
                Tables\Columns\TextColumn::make('name')
                    ->searchable(),
                Tables\Columns\TextColumn::make('city')
                    ->searchable(),
                Tables\Columns\IconColumn::make('is_active')
                    ->boolean(),
                Tables\Columns\TextColumn::make('employees_count')
                    ->counts('employees')
                    ->label('Employees'),
            ])
            ->filters([
                Tables\Filters\Filter::make('active')
                    ->query(fn ($query) => $query->where('is_active', true)),
            ])
            ->actions([
                Tables\Actions\EditAction::make(),
                Tables\Actions\DeleteAction::make()
                    ->before(function (Branch $record) {
                        if ($record->is_active && Branch::where('is_active', true)->count() <= 1) {
                            throw new \Exception('At least one active branch must exist');
                        }
                    }),
            ])
            ->bulkActions([
                Tables\Actions\BulkActionGroup::make([
                    Tables\Actions\DeleteBulkAction::make()
                        ->before(function ($records) {
                            $activeCount = $records->where('is_active', true)->count();
                            $totalActive = Branch::where('is_active', true)->count();

                            if ($activeCount >= $totalActive) {
                                throw new \Exception('Cannot delete all active branches');
                            }
                        }),
                ]),
            ]);
    }

    public static function getRelations(): array
    {
        return [];
    }

    public static function getPages(): array
    {
        return [
            'index' => Pages\ManageBranches::route('/'),
        ];
    }
}
```

#### ManageBranches Page (Simple Resource)
```php
namespace App\Filament\Resources\BranchResource\Pages;

use App\Filament\Resources\BranchResource;
use Filament\Resources\Pages\ManageRecords;

class ManageBranches extends ManageRecords
{
    protected static string $resource = BranchResource::class;
}
```

### Phase 2 Deliverables
- ✅ Branch CRUD via Simple Resource (modal)
- ✅ Company settings management page
- ✅ Geofence service with Haversine calculation
- ✅ Branch deletion guards (at least one active branch)
- ✅ Map picker integration for geofence configuration
- ✅ Timezone management per branch
- ✅ Migration tests for branches table
- ✅ Factory tests for BranchFactory states

---

## PHASE 3 — Employee Management Core

### Phase Overview
Departments, positions, employees, holiday calendar, onboarding templates, contracts, assets.

### Key Resources
- `EmployeeResource` (full resource with getPages, getRelations, infolist)
- `DepartmentResource` (Simple Resource — modal)
- `PositionResource` (Simple Resource — modal)
- `OnboardingTemplateResource` (Simple Resource — modal)
- `ContractTemplateResource` (Simple Resource — modal)
- `AssetCategoryResource` (Simple Resource — modal)
- `AssetResource` (Simple Resource — modal)

### Key Services
- `EmployeeService` (onboarding workflow, status transitions)
- `OnboardingService` (checklist generation, task assignment)
- `AssetAssignmentService` (assignment/return workflow)

### Tests
- Employee model relationship tests
- Employee factory states
- Onboarding workflow tests
- Asset assignment edge cases (already assigned, returned with damage)
- Department/Position CRUD tests

### Seeders
- `EmployeeSeeder` (50 employees)
- `OnboardingTemplateSeeder`
- `ContractTemplateSeeder`
- `AssetCategorySeeder`, `AssetSeeder`

---

## PHASE 4 — Time & Attendance Foundation

### Phase Overview
Shifts, attendance API, DTR processing, geofencing integration.

### Key Resources
- `ShiftResource` (Simple Resource — modal)
- `EmployeeScheduleResource`
- `AttendanceLogResource`
- `DtrEntryResource`
- `DtrCorrectionRequestResource`
- `GeofenceApprovalResource`
- `EmployeeSiteLocationsRelationManager`

### Key Services
- `AttendanceService` (time-in/out, bulk sync)
- `DtrService` (DTR generation, correction workflow)
- `GeofenceService` (branch + field employee geofence)
- `BiometricService` (ZKTeco ADMS/iClock protocol)

### API Endpoints
```text
POST /api/attendance/time-in
POST /api/attendance/time-out
POST /api/attendance/sync
```

### Tests
- Attendance API endpoint tests
- DTR calculation edge cases (late, undertime, overtime)
- Geofence boundary tests (exact radius, zero radius, poles)
- Biometric device mapping tests
- Schedule overlap tests

### Seeders
- `ShiftSeeder`
- `AttendanceLogSeeder` (demo attendance data)

---

## PHASE 5 — Leave Management System

### Phase Overview
Leave types, accrual rules, carry-over rules, approval chains, leave requests.

### Key Resources
- `LeaveTypeResource` (Simple Resource — modal)
- `LeaveCreditResource`
- `LeaveRequestResource`

### Key Services
- `LeaveService` (accrual, carry-over, pro-rating)
- `LeaveApprovalService` (n-level approval chain)
- `LeaveCashConversionService` (SIL conversion)

### Tests
- Leave accrual edge cases (monthly, quarterly, annual, on-hire)
- Carry-over expiry tests
- Pro-rating by employment type and tenure
- Approval chain routing tests
- Leave request validation tests

### Seeders
- `LeaveTypeSeeder`
- `LeaveCreditSeeder`
- `HolidaySeeder`

---

## PHASE 6 — Payroll Foundation

### Phase Overview
Payroll configuration, payroll runs, statutory contributions, benefits computation, government reports.

### Key Resources
- `PayPeriodResource` (Simple Resource — modal)
- `SalaryGradeResource` (Simple Resource — modal)
- `EmployeeSalaryResource`
- `PayrollRunResource`
- `PayrollItemResource`
- `DeductionTypeResource` (Simple Resource — modal)
- `EmployeeStatutoryLoanResource`
- `EmployeeCompanyLoanResource`
- `PayrollSettingsPage`
- `ThirteenthMonthRecordResource`
- `SeparationPayRecordResource`
- `FinalPayRecordResource`
- `RetirementPayRecordResource`
- `DeMinimisBenefitTypeResource` (Simple Resource — modal)
- `EmployeeDeMinimisBenefitResource`
- `LeaveCashConversionResource`
- `Bir2316RecordResource`
- `Bir1604cRecordResource`

### Key Services
- `PayrollService` (gross-to-net, government contributions)
- `ThirteenthMonthService` (PD 851)
- `NightDifferentialService` (Labor Code Art. 86)
- `HolidayPayService` (regular/special, OTF/OTRD)
- `DeMinimisService` (BIR RR, mid-year proration)
- `SeparationPayService` (final pay, retirement)
- `BirReportService` (2316, 1604-C)

### Tests
- Payroll computation edge cases (night diff, holiday premiums, OT)
- 13th month proration tests
- De minimis mid-year proration
- Separation pay calculation
- Retirement pay (RA 7641)
- BIR 2316 generation
- Loan deduction priority tests

### Seeders
- `SalaryGradeSeeder`
- `DeductionTypeSeeder`
- `PayPeriodSeeder`
- `EmployeeSalarySeeder`

---

## PHASE 7 — Employee Self-Service Portal

### Phase Overview
Portal resources (SingleRecordResource pattern), PWA features, offline sync, employee dashboard.

### Key Resources (Portal Panel)
- `MyProfileResource` (SingleRecordResource)
- `MyDocumentsResource` (SingleRecordResource)
- `MySettingsResource` (SingleRecordResource)
- `MyAttendanceResource` (SingleRecordResource)
- `MyDtrResource` (SingleRecordResource)
- `AttendanceCorrectionRequestResource`
- `MyLeaveBalancesResource` (SingleRecordResource)
- `MyLeaveRequestsResource`
- `FileLeaveResource` (Simple Resource — modal)
- `LeaveCalendarPage`
- `MyPayslipsResource` (SingleRecordResource)
- `MyBenefitsResource` (SingleRecordResource)
- `My13thMonthResource` (SingleRecordResource)
- `MySilConversionResource` (SingleRecordResource)
- `MyBirFormsResource` (SingleRecordResource)
- `MyLoansResource` (SingleRecordResource)
- `MyOnboardingResource` (SingleRecordResource)
- `MyContractsResource` (SingleRecordResource)
- `MyAssetsResource` (SingleRecordResource)
- `MyClearanceResource` (SingleRecordResource)
- `MyTrainingResource` (SingleRecordResource)
- `MyPerformanceResource` (SingleRecordResource)
- `MyGoalsResource`
- `MyReviewsResource` (SingleRecordResource)
- `MyPipResource` (SingleRecordResource)
- `MyTicketsResource`
- `CreateTicketResource` (Simple Resource — modal)
- `MyCasesResource` (SingleRecordResource)
- `MyPrivacyResource` (SingleRecordResource)
- `MyConsentsResource` (SingleRecordResource)
- `DataSubjectRequestResource` (Simple Resource — modal)

### Tests
- Portal access control (scoped to auth user)
- SingleRecordResource scoping tests
- PWA offline sync tests
- Portal form validation tests

---

## PHASE 8 — Advanced Features (ATS, Performance, Relations)

### Phase Overview
Recruitment (ATS), performance management, employee relations, training.

### Key Resources
- `JobRequisitionResource` (Simple Resource — modal)
- `JobPostingResource`
- `JobApplicantResource`
- `InterviewScheduleResource`
- `JobOfferResource`
- `PerformanceCycleResource` (Simple Resource — modal)
- `PerformanceReviewResource`
- `EmployeeGoalResource`
- `PerformanceImprovementPlanResource`
- `GrievanceCaseResource`
- `DisciplinaryCaseResource`
- `HrTicketResource`
- `TrainingProgramResource` (Simple Resource — modal)
- `EmployeeTrainingRecordResource`
- `EmployeeCertificationResource`

### Key Services
- `RecruitmentService` (applicant pipeline)
- `PerformanceService` (review cycles, scoring)
- `GrievanceService` (case workflow, mediation)
- `TrainingService` (enrollment, completion)
- `AiService` (resume parsing, sentiment analysis, ticket routing)

### Tests
- Recruitment pipeline stage transitions
- Performance scoring edge cases
- Grievance case workflow tests
- Training completion tracking
- AI service mock tests (resume parsing, sentiment, routing)

### Seeders
- `JobRequisitionSeeder`
- `JobPostingSeeder`
- `TrainingProgramSeeder`
- `PerformanceCycleSeeder`

---

## PHASE 9 — Compliance & Integrations

### Phase Overview
Data privacy compliance, webhooks, SSO, public API, biometric integration.

### Key Resources
- `DataPrivacyConsentResource`
- `DataSubjectRequestResource`
- `DataRetentionPolicyResource` (Simple Resource — modal)
- `PiiAuditReportPage`
- `WebhookSubscriptionResource` (Simple Resource — modal)
- `WebhookDeliveryLogResource`
- `PublicApiTokenResource`
- `SsoConfigurationPage`
- `BiometricDeviceResource` (Simple Resource — modal)
- `EmployeeDeviceMappingRelationManager`
- `BiometricUnmappedRecordResource`

### Key Services
- `DataPrivacyService` (consent tracking, DSAR)
- `WebhookService` (dispatch, retry, logging)
- `PublicApiService` (scoped read API)
- `SsoService` (Google Workspace / Microsoft 365)
- `BiometricService` (device mapping, unmapped records)

### Tests
- Consent tracking tests
- DSAR workflow tests
- Webhook dispatch and retry tests
- Public API rate limiting tests
- SSO callback tests
- Biometric device mapping tests

### Seeders
- `DataRetentionPolicySeeder`
- `WebhookSubscriptionSeeder`

---

## PHASE 10 — Reports & Analytics

### Phase Overview
Dashboards, reports, analytics, export functionality.

### Key Resources
- `AttendanceReportPage`
- `TardinessReportPage`
- `AbsenceReportPage`
- `GeofenceViolationReportPage`
- `BenefitsReportPage`
- `PayrollReportPage`
- `BirReportPage`
- `GlExportPage`
- `AnnouncementResource` (Simple Resource — modal)
- `SystemSettingsPage`
- `ActivityLogResource`
- `SystemLogPage`
- `RoleResource` (via Filament Shield)
- `PermissionResource` (via Filament Shield)

### Key Services
- `ReportService` (attendance, tardiness, absence, geofence)
- `PayrollReportService` (payroll summary, BIR)
- `GlExportService` (general ledger export)
- `AnalyticsService` (dashboard metrics)

### Tests
- Report generation tests
- Export format tests (Excel, PDF)
- Analytics aggregation tests
- Activity log filtering tests

### Seeders
- `AnnouncementSeeder`

---

## 🧱 Key Database ERD (Summary)

```text
company_settings (singleton, no FK)

branches ──< users ─(nullable employee_id)──> employees (bank details, is_field_employee)
                                                  >── employee_site_locations
                                                  >── departments >── positions
                                                  >── employee_schedules >── shifts
                                                  >── attendance_logs (geolocation + geofence_status)
                                                  >── dtr_entries >── dtr_correction_requests
                                                  >── leave_credits >── leave_types
                                                  │        >── leave_accrual_rules
                                                  │        >── leave_carryover_rules
                                                  │        >── leave_approval_chains >── leave_approval_steps
                                                  >── leave_requests >── leave_request_approvals
                                                  >── leave_cash_conversions
                                                  >── employee_salaries >── salary_grades
                                                  >── employee_allowances
                                                  >── employee_de_minimis_benefits >── de_minimis_benefit_types
                                                  >── employee_statutory_loans
                                                  >── employee_company_loans
                                                  >── payroll_items >── payroll_runs >── pay_periods
                                                  >── thirteenth_month_records
                                                  >── separation_pay_records >── final_pay_records
                                                  >── retirement_pay_records
                                                  >── employee_onboarding_checklists >── employee_onboarding_items
                                                  >── employee_contracts
                                                  >── asset_assignments >── assets >── asset_categories
                                                  >── employee_clearances >── employee_clearance_items
                                                  >── employee_device_mappings
                                                  >── data_privacy_consents
                                                  >── data_subject_requests
                                                  >── employee_certifications
                                                  >── employee_training_records
                                                  >── employee_goals >── performance_reviews
                                                  >── performance_improvement_plans >── pip_milestones

job_requisitions >── job_postings >── job_applicants >── interview_schedules
                                                       >── offer_letters
grievance_cases >── grievance_case_steps
hr_tickets >── hr_ticket_replies
biometric_devices >── biometric_unmapped_records
webhook_subscriptions >── webhook_delivery_logs
gl_account_mappings, data_retention_policies, training_programs, sso_configuration (singleton)
holidays, deduction_types, onboarding_templates, contract_templates, clearance_templates

activity_log, notifications, push_subscriptions, personal_access_tokens (device/portal tokens)

No company_id anywhere. branch_id is a plain nullable FK, not a tenant boundary.
All tables carry deleted_at (SoftDeletes).
```

---

## 🗺️ Development Timeline

```
PHASE 1 — Foundation & Core Infrastructure
Week 1-2: Project setup, two-panel architecture, authentication, enums,
          models, services, factories, seeders, migrations, Pest 4 tests

PHASE 2 — Company & Branch Management
Week 3: Company settings, branch CRUD (Simple Resource), geofencing, map integration

PHASE 3 — Employee Management Core
Week 4-5: Departments, positions, employees, holiday calendar, onboarding,
          contracts, assets

PHASE 4 — Time & Attendance Foundation
Week 6-7: Shifts, attendance API, DTR processing, geofencing integration,
          biometric devices

PHASE 5 — Leave Management System
Week 8: Leave types, accrual rules, approval chains, leave requests

PHASE 6 — Payroll Foundation
Week 9-11: Payroll configuration, payroll runs, statutory contributions,
            benefits computation, government reports, AI-powered resume parsing

PHASE 7 — Employee Self-Service Portal
Week 12-13: Portal resources (SingleRecordResource), PWA features, offline sync

PHASE 8 — Advanced Features
Week 14-16: ATS, performance management, employee relations, training,
            AI sentiment analysis and ticket routing

PHASE 9 — Compliance & Integrations
Week 17-18: Data privacy compliance, webhooks, SSO, public API,
            semantic/vector search

PHASE 10 — Reports & Analytics
Week 19: Dashboards, reports, analytics, export functionality
```

---

## 📜 Laravel 13 Specific Features Utilized

### PHP Attributes
```php
#[Table('users')]
#[Fillable(['name', 'email', 'password'])]
#[Hidden(['password', 'remember_token'])]
class User extends Authenticatable
{
    // ...
}
```

### Laravel AI SDK
```php
use Laravel\Ai\Facades\Ai;

$response = Ai::agent()
    ->withInstructions('Parse this resume and extract structured data.')
    ->prompt($resumeText);
```

### JSON:API Resources
```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\JsonApi\JsonApiResource;

class EmployeeResource extends JsonApiResource
{
    public function toArray($request): array
    {
        return [
            'first_name' => $this->first_name,
            'last_name' => $this->last_name,
            'email' => $this->email,
        ];
    }
}
```

### Queue Routing
```php
use Illuminate\Support\Facades\Queue;

Queue::route(ProcessPayroll::class, connection: 'redis', queue: 'payroll');
Queue::route(SendPayslipNotification::class, connection: 'redis', queue: 'notifications');
```

### Semantic / Vector Search
```php
$employees = DB::table('employees')
    ->whereVectorSimilarTo('skills_embedding', 'PHP Laravel development')
    ->limit(10)
    ->get();
```

### Cache::touch()
```php
Cache::touch('employee_dashboard_stats', 3600);
Cache::touch('payroll_summary', now()->addHours(6));
```

### PreventRequestForgery Middleware
```php
use Illuminate\Foundation\Http\Middleware\PreventRequestForgery;

->middleware([
    // ...
    PreventRequestForgery::class,
    // ...
])
```

---

*Generated for a Philippines-locale, single-company, private-sector-only HRIS with Laravel 13, Filament 5, and Pest 4.*