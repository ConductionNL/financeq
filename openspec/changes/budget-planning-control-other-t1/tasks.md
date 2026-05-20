# Tasks: Budget Planning & Control — Shillinq — Other T1

**Change:** budget-planning-control-other-t1
**Kind:** code
**Spec:** openspec/changes/budget-planning-control-other-t1/

## Deduplication Check

Per ADR-012, verify no overlap with OpenRegister core or existing Shillinq features before implementing:

- [ ] **Task 0 — Deduplication Check**
  - Confirm `ObjectService`, `WorkflowEngineRegistry`, `AuthorizationService`, `NotificationService`, `ExportService`, `FileService`, and `AuditTrailService` cover all generic CRUD and workflow needs (no custom equivalents needed)
  - Confirm `CnIndexPage`, `CnDetailPage`, `CnDashboardPage`, `CnFormDialog`, `CnChartWidget`, `CnStatsBlock`, `CnObjectSidebar`, `CnTimelineStages` cover all UI patterns (no custom components needed)
  - Confirm `x-openregister-lifecycle`, `x-openregister-calculations`, and `x-openregister-notifications` are available and stable for the declarative behaviours listed in design.md
  - Document findings (even "no overlap found") in a comment on the tracking issue
  - Custom code is limited to: `BudgetValidationService` (domain rule guard), `BudgetReportService` (complex cross-schema aggregation), and `BudgetAvailabilityGuard` (lifecycle guard interface)

---

## Phase 1 — Schema Register (Declarative)

- [ ] **Task 1 — Patch `lib/Settings/shillinq_register.json` — Budget lifecycle**
  - Add `x-openregister-lifecycle` to the `Budget` schema with states: `concept`, `ingediend`, `goedgekeurd`, `actief`, `gesloten`
  - Define transitions per design.md including `BudgetApprovalGuard` reference on `ingediend → goedgekeurd`
  - Run integration test: verify OR engine recognises the lifecycle block and all transitions are reachable

- [ ] **Task 2 — Patch `lib/Settings/shillinq_register.json` — Budget calculations**
  - Add `x-openregister-calculations` to `Budget` schema: `utilisatiePercentage`, `resterendBudget`, `verwachteUitputtingsdatum`
  - Verify calculated fields appear on Budget objects via `ObjectService.findObjects()`
  - Write integration test asserting correct calculation values for known input data

- [ ] **Task 3 — Patch `lib/Settings/shillinq_register.json` — Budget notifications**
  - Add `x-openregister-notifications` to `Budget` schema: drempel-breach notification (configurable %) and overspend notification (≥ 100%)
  - Add `x-openregister-notifications` to `BudgetAllocation` schema for allocation-level threshold breach
  - Verify notification triggers fire correctly in integration test with a mock notification sink

- [ ] **Task 4 — Patch `lib/Settings/shillinq_register.json` — ExpenditureRequest lifecycle**
  - Add `x-openregister-lifecycle` to `ExpenditureRequest` schema with states: `concept`, `ingediend`, `in_behandeling`, `goedgekeurd`, `afgewezen`
  - Wire `BudgetAvailabilityGuard` on the `concept → ingediend` transition
  - Write integration test covering both the guard-passes and guard-blocks paths

- [ ] **Task 5 — Patch `lib/Settings/shillinq_register.json` — BudgetAmendment lifecycle**
  - Add `x-openregister-lifecycle` to `BudgetAmendment` schema with states: `concept`, `aangevraagd`, `goedgekeurd`, `afgewezen`
  - Write integration test for the full amendment approval flow

- [ ] **Task 6 — Generate seed data in `lib/Settings/shillinq_register.json`**
  - Add `components.objects[]` entries for the 5 Budget seed objects, 3 BudgetPeriod objects, 3 FundingSource objects, 3 BudgetAllocation objects, and 3 ExpenditureRequest objects from design.md
  - Use `@self` envelope with `register: "shillinq"`, correct schema names, and unique slugs
  - Verify idempotency: re-importing with `force: false` does not duplicate objects (matched by slug)

---

## Phase 2 — Backend (Lifecycle Guard + Report Service)

- [ ] **Task 7 — Implement `BudgetAvailabilityGuard`**
  - Create `lib/Lifecycle/BudgetAvailabilityGuard.php` implementing the OR lifecycle guard interface
  - `check(array $context): bool` — queries committed + open ExpenditureRequest amounts against `BudgetAllocation.resterendBudget`
  - Throws `BudgetExceededException` with the shortfall amount when insufficient
  - Add `@spec openspec/changes/budget-planning-control-other-t1/tasks.md#task-7` PHPDoc tag (ADR-003)
  - Write PHPUnit tests: `testGuardPassesWhenBudgetSufficient()`, `testGuardBlocksWhenBudgetInsufficient()`, `testGuardHandlesConcurrentCommitments()`

- [ ] **Task 8 — Implement `BudgetApprovalGuard`**
  - Create `lib/Lifecycle/BudgetApprovalGuard.php`
  - `check(array $context): bool` — verifies the current user has the `budget-approve` permission via `AuthorizationService`
  - Add `@spec` PHPDoc tag
  - Write PHPUnit tests: `testApproverCanApprove()`, `testNonApproverIsBlocked()`

- [ ] **Task 9 — Implement `BudgetValidationService`**
  - Create `lib/Service/BudgetValidationService.php`
  - `getAvailableBalance(string $allocationId): float` — returns `resterendBudget` from the calculated field
  - `validateExpenditureRequest(string $requestId): ValidationResult` — wraps the guard logic for controller use
  - Stateless, constructor-injected dependencies only (ADR-003)
  - Add `@spec` PHPDoc tag
  - Write PHPUnit tests: ≥ 3 test methods

- [ ] **Task 10 — Implement `BudgetReportService`**
  - Create `lib/Service/BudgetReportService.php`
  - `getVarianceReport(string $periodId, string $comparePeriodId): array` — queries Budget, BudgetAllocation, and GeneralLedgerEntry objects for both periods; computes variance columns
  - `getUtilisationReport(string $periodId): array` — returns allocations sorted by variance, with reallocation candidates flagged
  - Add `@spec` PHPDoc tag
  - Write PHPUnit tests: `testVarianceReportStructure()`, `testUtilisationReportSorting()`, `testEmptyPeriodReturnsEmptyReport()`

---

## Phase 3 — API Controllers

- [ ] **Task 11 — Implement `BudgetController`**
  - Create `lib/Controller/BudgetController.php` — thin controller (<10 lines/method, ADR-003)
  - Routes: `GET /api/budgets`, `POST /api/budgets`, `GET /api/budgets/{id}`, `PUT /api/budgets/{id}`, `DELETE /api/budgets/{id}`
  - Register routes in `appinfo/routes.php` before any wildcard `{slug}` routes
  - Add `@spec` PHPDoc tags on class and each public method
  - Write Newman/Postman collection test for each endpoint

- [ ] **Task 12 — Implement `BudgetPeriodController`**
  - Create `lib/Controller/BudgetPeriodController.php`
  - Routes: `GET/POST /api/budget-periods`, `GET/PUT/DELETE /api/budget-periods/{id}`
  - Register routes in `appinfo/routes.php`
  - Add `@spec` PHPDoc tags

- [ ] **Task 13 — Implement `BudgetAllocationController`**
  - Create `lib/Controller/BudgetAllocationController.php`
  - Routes: `GET/POST /api/budget-allocations`, `GET/PUT/DELETE /api/budget-allocations/{id}`
  - Register routes in `appinfo/routes.php`
  - Add `@spec` PHPDoc tags

- [ ] **Task 14 — Implement `ExpenditureRequestController`**
  - Create `lib/Controller/ExpenditureRequestController.php`
  - Routes: `GET/POST /api/expenditure-requests`, `GET/PUT/DELETE /api/expenditure-requests/{id}`
  - Register routes in `appinfo/routes.php`
  - Inject `BudgetValidationService` for pre-submission checks surfaced in API response
  - Add `@spec` PHPDoc tags

- [ ] **Task 15 — Implement `BudgetReportController`**
  - Create `lib/Controller/BudgetReportController.php`
  - `GET /api/budget-reports/variance?period={id}&comparePeriod={id}` — delegates to `BudgetReportService.getVarianceReport()`
  - `GET /api/budget-reports/utilisation?period={id}` — delegates to `BudgetReportService.getUtilisationReport()`
  - Admin auth required; return `403` for non-finance-manager roles
  - Add `@spec` PHPDoc tags

---

## Phase 4 — Frontend

- [ ] **Task 16 — Create Budget Pinia store**
  - `src/store/modules/budget.js` using `createObjectStore('budget')` with plugins: `auditTrails`, `files`, `relations`, `selection`
  - Register in `store/store.js` via `objectStore.registerObjectType('budget', 'Budget', 'shillinq')`

- [ ] **Task 17 — Create BudgetPeriod, BudgetAllocation, ExpenditureRequest stores**
  - One `createObjectStore` per entity in `src/store/modules/`
  - Register all in `store/store.js`

- [ ] **Task 18 — Budget Dashboard page**
  - `src/views/BudgetDashboard.vue` using `CnDashboardPage`
  - 4 KPI blocks (`CnStatsBlock`): totaal budget, besteed, vastgelegd, resterend
  - Utilisation donut chart (`CnChartWidget`) per top-level budget
  - "Budgets bijna uitgeput" list (< 60 days to exhaustion)
  - Fetch all stores in parallel via `Promise.all` in `useDashboardView`
  - Route: `/` → `BudgetDashboard`

- [ ] **Task 19 — Budget list and detail pages**
  - `src/views/BudgetIndex.vue` — `CnIndexPage` + `useListView`; row click → `/budgets/:id`
  - `src/views/BudgetDetail.vue` — `CnDetailPage` with sections: details, allocations table, amendment history; `CnObjectSidebar` (files, notes, audit trail)
  - Forms use `CnFormDialog` (schema-driven); no custom form components

- [ ] **Task 20 — ExpenditureRequest list and detail pages**
  - `src/views/ExpenditureRequestIndex.vue` — `CnIndexPage` + `useListView`; `CnTimelineStages` for lifecycle visualization
  - `src/views/ExpenditureRequestDetail.vue` — `CnDetailPage` + `CnObjectSidebar`
  - Show `resterendBudget` from linked BudgetAllocation calculated field in the request form

- [ ] **Task 21 — Budget Report page**
  - `src/views/BudgetReport.vue`
  - Period selector dropdowns (BudgetPeriod objects via store)
  - Variance report table via `CnTableWidget` fed from `BudgetReportController`
  - Utilisation chart via `CnChartWidget` (horizontal bar, sorted by variance)
  - Export button using `CnMassExportDialog`

- [ ] **Task 22 — Router and navigation**
  - Add named routes to `src/router/index.js`: `BudgetDashboard`, `BudgetIndex`, `BudgetDetail`, `BudgetPeriodIndex`, `ExpenditureRequestIndex`, `ExpenditureRequestDetail`, `BudgetAllocationIndex`, `BudgetReport`, `Settings`
  - Add `NcAppNavigationItem` entries to `MainMenu.vue` for each main section
  - All routes follow history mode with base `/index.php/apps/shillinq/` (ADR-004)

- [ ] **Task 23 — Settings page**
  - `src/views/Settings.vue` — `CnVersionInfoCard` (first, always) → `CnRegisterMapping` → `CnSettingsSection` for budget alert configuration
  - Load settings from `GET /api/settings`; save via `POST /api/settings`
  - Re-import seed data button: `POST /api/settings/load`

---

## Phase 5 — Internationalisation, Tests, and Docs

- [ ] **Task 24 — Dutch and English translations**
  - Add all user-visible strings to `l10n/nl.js` and `l10n/en.js` (ADR-007)
  - Strings include: lifecycle state labels, budget status badges, notification messages, form field labels, report column headers
  - No hardcoded strings anywhere in Vue components — all via `t(appName, 'key')`

- [ ] **Task 25 — PHPUnit tests for all services and controllers**
  - Minimum 3 test methods per new PHP class (ADR-008)
  - Tests cover: happy path, edge cases (zero budget, concurrent requests), authorization failures
  - All tests pass under `composer check:strict`

- [ ] **Task 26 — Newman/Postman API integration tests**
  - `tests/integration/budget-planning.postman_collection.json`
  - Cover all 5 controllers with create, read, update, delete, and auth-failure cases
  - Include variance report and utilisation report endpoints

- [ ] **Task 27 — Playwright browser tests (GIVEN/WHEN/THEN)**
  - One Playwright test per GIVEN/WHEN/THEN scenario in the 5 spec files
  - Priority scenarios: budget creation → approval → expenditure request → guard block
  - Tests use seed data loaded via `POST /api/settings/load`

- [ ] **Task 28 — Documentation**
  - `docs/budget-management.md` — user guide for creating and approving budgets
  - `docs/expenditure-control.md` — user guide for submitting and reviewing expenditure requests
  - `docs/budget-reporting.md` — guide to running comparative period reports
  - English primary, Dutch sections for BBV/IV3 compliance context (ADR-009)

---

## Phase 6 — Metrics and Health

- [ ] **Task 29 — Budget metrics endpoint**
  - Expose budget-specific Prometheus metrics via `GET /api/metrics` (ADR-006): `shillinq_budget_total`, `shillinq_budget_active_count`, `shillinq_expenditure_requests_pending`, `shillinq_budget_overspend_count`
  - Health check (`GET /api/health`) includes OpenRegister connectivity verification
