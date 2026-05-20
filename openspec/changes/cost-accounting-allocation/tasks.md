# Tasks: Cost Accounting & Allocation — Shillinq

## Deduplication Check

- [ ] Verify no overlap with `ObjectService`, `RegisterService`, `SchemaService`, `ConfigurationService` for CRUD operations — use platform services, do not rebuild
- [ ] Verify `AllocationService` business logic (allocation run, zero-key guard, version control, reversal) has no equivalent in OpenRegister core or existing Shillinq services
- [ ] Verify `TimesheetService` aggregation logic is not duplicated in existing `TimeEntry` handling elsewhere in the app
- [ ] Verify `PerDiemService` rate lookup is not already implemented in `ExpenseCategory` or `ExpenseClaim` flows
- [ ] Verify `InventoryValuationService` FIFO/average cost logic is not duplicated in `InventoryItem` or `InventoryStock` handling
- [ ] Confirm `WorkflowEngineController` is reused for timesheet and per diem approval chains rather than building custom approval logic
- [ ] Confirm `CnDashboardPage`, `CnChartWidget`, `CnStatsBlock` are reused for cost accounting dashboard — no custom chart components
- [ ] Document findings in design.md "Reuse Analysis" section (already included — verify completeness after implementation)

---

## 1. OpenRegister Schemas and Seed Data

- [ ] Add `CostCenter` schema to `lib/Settings/shillinq_register.json` with all properties from design.md (code, name, description, status, budget, createdDate) and relations to Person and Organization
- [ ] Add `AllocationRule` schema to register JSON with all properties (name, ruleType, percentage, fixedAmount, frequency, isActive, startDate, endDate, description) and source/target CostCenter relations
- [ ] Add `CostAllocation` schema to register JSON with all properties (name, allocationDate, sourceAmount, allocationPercentage, allocationAmount, period, status, version, description) and source/target CostCenter relations
- [ ] Add `CostProject` schema to register JSON with all properties (code, name, description, budget, totalCost, startDate, endDate, status) and Organization/CostCenter relations
- [ ] Add `InventoryValuation` schema to register JSON with all properties (quantity, unitCost, totalValue, valuationMethod, date, warehouse, status) and Product/CostCenter relations
- [ ] Add `PerDiem` schema to register JSON with all properties (date, country, nights, rate, amount, status, approvedDate, description) and Person/CostCenter relations
- [ ] Add `Timesheet` schema to register JSON with all properties (periodStart, periodEnd, totalHours, utilizationPercentage, totalCost, status, submittedDate, approvedDate) and Person/TimeEntry/ApprovalRequest relations
- [ ] Add 5 seed `CostCenter` objects (CC-ICT, CC-HRM, CC-FIN, CC-ADV, CC-FAC) using `@self` envelope with Dutch values as defined in design.md
- [ ] Add 4 seed `AllocationRule` objects (ar-ict-naar-adv, ar-hrm-naar-adv, ar-fac-vast, ar-fin-naar-adv) with Dutch values as defined in design.md
- [ ] Add 4 seed `CostAllocation` objects (ca-jan2026-ict, ca-jan2026-hrm, ca-q12026-fac, ca-feb2026-ict-v2) with Dutch values as defined in design.md
- [ ] Add 4 seed `CostProject` objects (proj-2026-dlo, proj-2026-crm, proj-2025-duu, proj-2026-pro) with Dutch values as defined in design.md
- [ ] Add 4 seed `InventoryValuation` objects (iv-laptop, iv-monitor, iv-kantoor, iv-server) with Dutch values as defined in design.md
- [ ] Add 4 seed `PerDiem` objects (pd-devries-nl, pd-bakker-be, pd-janssen-nl, pd-devries-de) with Dutch values as defined in design.md
- [ ] Add 4 seed `Timesheet` objects (ts-pietersen-jan2026, ts-degroot-jan2026, ts-janssen-feb2026, ts-bakker-jan2026) with Dutch values as defined in design.md
- [ ] Verify register JSON is valid OpenAPI 3.0 + x-openregister format
- [ ] Verify slugs are unique and idempotent (re-import skips existing objects)

---

## 2. Repair Step / Schema Registration

- [ ] Create `lib/Migration/RepairRegisterStep.php` implementing `IRepairStep` that calls `ConfigurationService::importFromApp(appId, data, version, force: false)` to register schemas and seed data on install/upgrade
- [ ] Add `@spec openspec/changes/cost-accounting-allocation/tasks.md#task-2` PHPDoc to `RepairRegisterStep`
- [ ] Add repair step to `appinfo/info.xml` under `<repair-steps><install>` and `<upgrade>`
- [ ] Verify idempotency: running repair step twice does not duplicate schemas or seed objects

---

## 3. Backend Services

### AllocationService

- [ ] Create `lib/Service/AllocationService.php` with `@spec openspec/changes/cost-accounting-allocation/tasks.md#task-3` header docblock
- [ ] Implement `runAllocationForPeriod(string $period, string $register, string $schema): array` — fetches active AllocationRules for the period, evaluates each rule, creates draft CostAllocation records via `ObjectService::saveObject()`
- [ ] Implement zero-key guard: when formula or percentage resolves to zero total, log warning, skip key, include skip notice in run result array
- [ ] Implement version increment: when re-running for a period that already has CostAllocation records, create new records with `version + 1`; retain previous versions
- [ ] Implement `reverseAllocation(string $allocationId, string $register, string $schema): array` — creates offsetting CostAllocation with negative `allocationAmount` and same version; updates original status to `reversed`
- [ ] Implement `getAllocationBreakdown(string $period): array` — returns allocated amount per product/programme with applied key percentage and source cost center
- [ ] Write PHPUnit tests in `tests/Unit/Service/AllocationServiceTest.php` covering: normal run, zero-key warning, version increment, reversal

### TimesheetService

- [ ] Create `lib/Service/TimesheetService.php` with `@spec` header
- [ ] Implement `aggregatePeriod(string $timesheetId): array` — sums `totalHours` from linked TimeEntry objects, calculates `utilizationPercentage`, calculates `totalCost` using employee hourly rate
- [ ] Implement `getTeamSummary(string $costCenterId, string $periodStart, string $periodEnd): array` — returns hours, utilization, and cost aggregated per team member
- [ ] Write PHPUnit tests in `tests/Unit/Service/TimesheetServiceTest.php` (≥3 test methods)

### PerDiemService

- [ ] Create `lib/Service/PerDiemService.php` with `@spec` header
- [ ] Implement `calculateAmount(string $country, int $nights, string $date): float` — looks up rate from config table for country/date combination; applies overnight rate when nights ≥ 1, day-trip rate when nights = 0
- [ ] Implement `getDefaultRate(string $country, string $date): float` — returns configured rate for country or Dutch domestic fallback
- [ ] Write PHPUnit tests in `tests/Unit/Service/PerDiemServiceTest.php` (≥3 test methods: NL overnight, NL day, foreign country)

### InventoryValuationService

- [ ] Create `lib/Service/InventoryValuationService.php` with `@spec` header
- [ ] Implement `recalculateAverage(string $productId): float` — computes weighted average unit cost from all receipt InventoryValuation records for the product
- [ ] Implement `recalculateFIFO(string $productId): float` — computes FIFO cost of on-hand stock from earliest-received unsold units
- [ ] Implement `calculateFullCostPrice(string $productId, string $period): float` — adds overhead allocation per unit from linked CostAllocation to direct unit cost
- [ ] Write PHPUnit tests in `tests/Unit/Service/InventoryValuationServiceTest.php` (≥3 test methods: average cost, FIFO, full cost price)

---

## 4. Backend Controllers

- [ ] Create `lib/Controller/AllocationController.php` implementing `POST /api/allocations/run`, `GET /api/allocations/{id}/results`, `POST /api/allocations/{id}/reverse`
- [ ] Annotate all mutation methods with `IGroupManager::isAdmin()` check — reject with 403 if not admin
- [ ] Add `@spec` PHPDoc to all controller methods
- [ ] Add routes to `appinfo/routes.php` — specific allocation routes BEFORE any wildcard `{slug}` routes
- [ ] Register CORS OPTIONS route for allocation endpoints (ADR-002)
- [ ] Ensure all error responses return static messages, never `$e->getMessage()` (ADR-015)
- [ ] Add `GET /api/metrics` Prometheus endpoint with `shillinq_` prefix, including `shillinq_health_status` and `shillinq_info` metrics (ADR-006)
- [ ] Add `GET /api/health` endpoint verifying OpenRegister connectivity (ADR-006)
- [ ] Write PHPUnit tests in `tests/Unit/Controller/AllocationControllerTest.php` (≥3 methods)

---

## 5. Background Jobs

- [ ] Create `lib/BackgroundJob/RecurringAllocationJob.php` implementing Nextcloud's `TimedJob` with `@spec` header
- [ ] Job runs daily; checks AllocationRule records with `isActive: true` and `frequency: monthly` or `quarterly` — creates draft CostAllocation records when period boundary is crossed
- [ ] Skip expired rules (rules where `endDate` is in the past)
- [ ] Register job in `lib/AppInfo/Application.php` via `IJobList::add()`
- [ ] Write PHPUnit test for job execution logic in `tests/Unit/BackgroundJob/RecurringAllocationJobTest.php`

---

## 6. Frontend Stores

- [ ] Register `cost-center` entity type in `src/store/store.js` via `createObjectStore('cost-center')` with `files`, `auditTrails`, `relations` plugins
- [ ] Register `allocation-rule` entity type in `src/store/store.js`
- [ ] Register `cost-allocation` entity type in `src/store/store.js`
- [ ] Register `cost-project` entity type in `src/store/store.js`
- [ ] Register `inventory-valuation` entity type in `src/store/store.js`
- [ ] Register `per-diem` entity type in `src/store/store.js`
- [ ] Register `timesheet` entity type in `src/store/store.js`
- [ ] Verify all type names are kebab-case; no camelCase type strings anywhere
- [ ] Add settings store entry for Dutch per diem rates and mileage rate configuration

---

## 7. Frontend Router

- [ ] Add named route `CostCenterList` → `/kostenplaatsen` in `src/router/index.js`
- [ ] Add named route `CostCenterDetail` → `/kostenplaatsen/:id`
- [ ] Add named route `AllocationRuleList` → `/verdeelregels`
- [ ] Add named route `AllocationRuleDetail` → `/verdeelregels/:id`
- [ ] Add named route `CostAllocationList` → `/kostenverdelingen`
- [ ] Add named route `CostAllocationDetail` → `/kostenverdelingen/:id`
- [ ] Add named route `CostProjectList` → `/projecten`
- [ ] Add named route `CostProjectDetail` → `/projecten/:id`
- [ ] Add named route `InventoryValuationList` → `/voorraden`
- [ ] Add named route `InventoryValuationDetail` → `/voorraden/:id`
- [ ] Add named route `PerDiemList` → `/dagvergoedingen`
- [ ] Add named route `PerDiemDetail` → `/dagvergoedingen/:id`
- [ ] Add named route `TimesheetList` → `/urenstaten`
- [ ] Add named route `TimesheetDetail` → `/urenstaten/:id`
- [ ] Add `Dashboard` route → `/`
- [ ] Verify all routes are flat (no nesting), all named, props via arrow function for params
- [ ] Verify catch-all `*` redirect to `/` is present

---

## 8. Frontend Pages

### Dashboard

- [ ] Create `src/views/DashboardView.vue` using `CnDashboardPage`
- [ ] Add 4 `CnStatsBlock` KPI cards: open allocations, active cost centers, pending timesheets, pending per diems
- [ ] Add `CnChartWidget` (donut) for allocation status distribution (draft / approved / allocated)
- [ ] Add `CnChartWidget` (bar) for cost center budget vs. actuals
- [ ] Add `CnTableWidget` for recent allocation runs
- [ ] Fetch all data in parallel via `Promise.all` in `created()`
- [ ] Wrap all `await store.action()` calls in `try/catch` with user-facing error feedback

### Cost Center

- [ ] Create `src/views/CostCenterListView.vue` using `CnIndexPage` + `useListView('cost-center', { sidebarState, objectStore })`
- [ ] Create `src/views/CostCenterDetailView.vue` using `CnDetailPage` with `CnDetailCard` for general info
- [ ] Add `CnDetailCard` for linked AllocationRules (forward lookup via `fetchUses`)
- [ ] Add `CnDetailCard` for linked CostAllocations (reverse lookup via `fetchUsed`)
- [ ] Add `CnDetailCard` for linked CostProjects (reverse lookup via `fetchUsed`)
- [ ] Add `CnObjectSidebar` with Files, Notes, Audit tabs

### Allocation Rule

- [ ] Create `src/views/AllocationRuleListView.vue` using `CnIndexPage` + `useListView`
- [ ] Create `src/views/AllocationRuleDetailView.vue` with `CnDetailCard` for rule properties
- [ ] Add `CnDetailCard` for source CostCenter (forward lookup)
- [ ] Add `CnDetailCard` for target CostCenter (forward lookup)
- [ ] Add `CnDetailCard` for generated CostAllocations (reverse lookup via `fetchUsed`)
- [ ] Add `CnObjectSidebar` with Audit tab

### Cost Allocation

- [ ] Create `src/views/CostAllocationListView.vue` using `CnIndexPage` + `useListView`
- [ ] Create `src/views/CostAllocationDetailView.vue` with `CnDetailCard` for allocation properties, version badge, and status timeline using `CnTimelineStages` (draft → approved → allocated)
- [ ] Add "Verdeelrun uitvoeren" action button that calls `POST /api/allocations/run` — only visible to admins
- [ ] Add "Terugboeken" (reverse) action button calling `POST /api/allocations/{id}/reverse` — only visible when status is `allocated`
- [ ] Add `CnDetailCard` for allocation breakdown per product
- [ ] Add `CnObjectSidebar` with Audit tab

### Cost Project

- [ ] Create `src/views/CostProjectListView.vue` using `CnIndexPage` + `useListView`
- [ ] Create `src/views/CostProjectDetailView.vue` with `CnDetailCard` for project info
- [ ] Add `CnDetailCard` for linked CostAllocations (reverse lookup)
- [ ] Add `CnDetailCard` for linked Timesheets (reverse lookup)
- [ ] Add budget utilization progress indicator (`CnProgressBar`) showing `totalCost / budget × 100`
- [ ] Add `CnObjectSidebar` with Files, Notes, Audit tabs

### Inventory Valuation

- [ ] Create `src/views/InventoryValuationListView.vue` using `CnIndexPage` + `useListView`
- [ ] Create `src/views/InventoryValuationDetailView.vue` with `CnDetailCard` for valuation info and method badge
- [ ] Add `CnObjectSidebar` with Audit tab

### Per Diem

- [ ] Create `src/views/PerDiemListView.vue` using `CnIndexPage` + `useListView`
- [ ] Create `src/views/PerDiemDetailView.vue` with `CnDetailCard` and `CnTimelineStages` (draft → approved → paid)
- [ ] Auto-calculate `amount` when `country`, `nights`, or `date` changes — call backend `PerDiemService` or apply rate lookup in Vue computed property
- [ ] Add `CnObjectSidebar` with Audit tab

### Timesheet

- [ ] Create `src/views/TimesheetListView.vue` using `CnIndexPage` + `useListView` with status filter (draft / submitted / approved)
- [ ] Create `src/views/TimesheetDetailView.vue` with `CnDetailCard` and `CnTimelineStages` (draft → submitted → approved)
- [ ] Add `CnDetailCard` for linked TimeEntry objects (reverse lookup via `fetchUsed`)
- [ ] Show utilization percentage as `CnProgressBar`
- [ ] Add `CnObjectSidebar` with Audit tab

### MainMenu

- [ ] Add `NcAppNavigationItem` entries for all 7 entity list routes plus Dashboard
- [ ] Add settings link via `NcAppNavigationSettings` footer

---

## 9. Admin Settings

- [ ] Create `lib/Settings/AdminSettings.php` registering the admin settings panel
- [ ] Create `src/views/AdminRoot.vue` with `CnVersionInfoCard` (first), `CnRegisterMapping`, settings sections for Dutch per diem rates and mileage rate (EUR 0.23/km default)
- [ ] Load settings via `GET /api/settings`; save via `POST /api/settings`
- [ ] Use `IInitialState::provideInitialState` in PHP; use `loadState` from `@nextcloud/initial-state` in Vue — do NOT pass data via DOM attributes
- [ ] Do NOT add `AdminRoot.vue` to the vue-router (ADR-004)

---

## 10. Translations

- [ ] Add Dutch (`nl`) translation strings for all user-visible labels in `l10n/nl.json`
- [ ] Add English (`en`) translation strings as baseline in `l10n/en.json`
- [ ] Verify ALL user-visible strings use `t(appName, 'key')` in Vue and `$this->l->t('key')` in PHP — no hardcoded Dutch or English strings in templates
- [ ] Verify all translation keys are English; Dutch translations go only in `l10n/nl.json`

---

## 11. SPDX License Headers

- [ ] Add `// SPDX-License-Identifier: EUPL-1.2` after `<?php` to every new PHP file
- [ ] Add `<!-- SPDX-License-Identifier: EUPL-1.2 -->` as first line of every new Vue file
- [ ] Add `// SPDX-License-Identifier: EUPL-1.2` as first line of every new JS file
- [ ] Run `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` and fix any missing headers

---

## 12. Tests

- [ ] PHPUnit tests for `AllocationService` — `tests/Unit/Service/AllocationServiceTest.php` (≥3 methods: normal run, zero-key guard, reversal)
- [ ] PHPUnit tests for `TimesheetService` — `tests/Unit/Service/TimesheetServiceTest.php` (≥3 methods)
- [ ] PHPUnit tests for `PerDiemService` — `tests/Unit/Service/PerDiemServiceTest.php` (≥3 methods)
- [ ] PHPUnit tests for `InventoryValuationService` — `tests/Unit/Service/InventoryValuationServiceTest.php` (≥3 methods)
- [ ] PHPUnit tests for `AllocationController` — `tests/Unit/Controller/AllocationControllerTest.php` (≥3 methods)
- [ ] PHPUnit tests for `RecurringAllocationJob` — `tests/Unit/BackgroundJob/RecurringAllocationJobTest.php`
- [ ] Newman/Postman collection for allocation run, get results, and reversal endpoints — `tests/integration/allocation.json`
- [ ] Playwright browser test for US-1: overhead cost allocation run (GIVEN overhead cost centres closed → WHEN accountant triggers run → THEN costs distributed per allocation key)
- [ ] Playwright browser test for REQ-CA-001 zero-key guard: allocation with zero-total key raises warning and skips
- [ ] Playwright browser test for REQ-TS-002: timesheet submit and approve workflow
- [ ] Playwright browser test for REQ-PD-001: per diem auto-calculation for Dutch domestic overnight
- [ ] All tests must pass in `composer check:strict`

---

## 13. Documentation

- [ ] Create `docs/cost-centers.md` with screenshots of cost center list and detail pages
- [ ] Create `docs/allocation-rules.md` explaining percentage, fixed, and formula rule types
- [ ] Create `docs/allocation-run.md` step-by-step guide for running monthly overhead allocation
- [ ] Create `docs/timesheets.md` with weekly/monthly summary and approval workflow
- [ ] Create `docs/per-diem.md` with Dutch per diem rate table and mileage rate
- [ ] Create `docs/inventory-valuation.md` explaining FIFO, average, and weighted average methods

---

## 14. Pre-Commit Verification Checklist

Before committing, verify all items from ADR-015:

- [ ] SPDX headers present in all new files
- [ ] `ObjectService` calls use 3 positional args: `($register, $schema, $idOrParams)`
- [ ] Error responses use static messages, never `$e->getMessage()`
- [ ] All POST/PUT/DELETE methods have `IGroupManager::isAdmin()` check
- [ ] Each entity registered exactly once in store.js with kebab-case names
- [ ] `npm run lint` passes — no extraneous imports
- [ ] All user-visible strings use `this.t()` in Vue; no hardcoded strings
- [ ] Every `await store.action()` is wrapped in `try/catch`
- [ ] No raw `fetch()` — use `@nextcloud/axios` for all API calls
- [ ] No `from '@nextcloud/vue'` imports — use `@conduction/nextcloud-vue`
- [ ] All `<NcFoo>` and `<CnFoo>` components are imported AND listed in `components: {}`
- [ ] All entity type strings are kebab-case (no camelCase) across ALL files
- [ ] All `t()` keys are English; Dutch text is in `l10n/nl.json` only
- [ ] Every route referenced in navigation has a matching named route in `src/router/`
- [ ] All `[x]` tasks are fully implemented (not stubs)
