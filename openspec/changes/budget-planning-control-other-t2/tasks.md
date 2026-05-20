# Tasks: Budget Planning & Control — Other T2

**Change:** budget-planning-control-other-t2
**App:** Shillinq
**Date:** 2026-05-20

---

## Deduplication Check (ADR-012)

- [ ] **TASK-000: Deduplication check** — Before starting implementation, verify no overlap with OpenRegister core or shared specs.
  - Search `openregister/lib/Service/` for any existing budget validation, threshold alert, or structural balance logic → **Finding: none found**
  - Search `openspec/specs/` for budget-monitoring, budget-compliance, budget-contracts specs → **Finding: none found**
  - Check `ObjectService`, `RegisterService`, `SchemaService`, `ConfigurationService` for budget-domain overlap → **Finding: none found**
  - Confirm `@conduction/nextcloud-vue` covers all dashboard/chart/form/list requirements (CnDashboardPage, CnChartWidget, CnStatsBlock, CnIndexPage, CnDetailPage, CnFormDialog) → **Finding: confirmed, no custom components needed**
  - Document: custom PHP services (`BudgetValidationService`, `TaakveldValidationService`, `StructuralBalanceService`) are justified by cross-schema guard patterns not expressible in OR declarative engine (see design.md §Declarative-vs-Imperative)

---

## Schema Register Patches (lib/Settings/shillinq_register.json)

- [ ] **TASK-001: Add alert threshold and period fields to Budget schema**
  - Add `alertThreshold` (integer, min 0, max 100, default 90)
  - Add `warningThreshold` (integer, min 0, max 100, default 75)
  - Add `overBudgetPrevention` (boolean, default true)
  - Add `bbvProgramme` (string)
  - Add `allocationPeriod` (enum: monthly/quarterly/yearly/custom, default yearly)
  - Add `indexationPercentage` (number)
  - Add `budgetType` (enum: opex/capex, default opex)
  - Verify additions are non-breaking (no removed or renamed properties per ADR-011)

- [ ] **TASK-002: Add Dutch compliance fields to BudgetAllocation schema**
  - Add `iv3Category` (string — IV3 category code)
  - Add `taakveld` (string — BBV Bijlage IV code)
  - Add `structural` (boolean, default false)
  - Add `forecastAmount` (number)
  - Verify non-breaking additions only

- [ ] **TASK-003: Add `x-openregister-notifications` to Budget schema**
  - Define `budget-warning` notification: fires when `utilisation >= warningThreshold AND utilisation < alertThreshold`
  - Define `budget-critical` notification: fires when `utilisation >= alertThreshold`
  - Recipients: `relation:budgetOwner` + `role:financial-controller`
  - Dutch message templates per ADR-007: "Budgetwaarschuwing: {{ utilisation }}% besteed van {{ name }}" / "Budget kritisch: ..."
  - Set `deduplication: once-per-band` to prevent duplicate notifications within the same utilisation band

- [ ] **TASK-004: Add `x-openregister-aggregations` to Budget schema**
  - `actualSpend` — sum of SpendingRecord.amount where budgetId = @self.id
  - `committedAmount` — sum of PurchaseOrder.totalAmount where budgetId = @self.id AND status != 'cancelled'
  - Verify aggregation triggers on SpendingRecord and PurchaseOrder save events

- [ ] **TASK-005: Add `x-openregister-calculations` to Budget schema**
  - `utilisation` — `ceiling > 0 ? (actualSpend / ceiling) * 100 : 0` (type: number)
  - `remaining` — `ceiling - actualSpend - committedAmount` (type: number)
  - Verify calculated fields are available on Budget objects without additional API calls

- [ ] **TASK-006: Add `x-openregister-lifecycle` to BudgetAmendment schema**
  - Initial state: `draft`
  - States: `draft`, `submitted`, `approved`, `rejected`
  - Transitions: draft→submitted (guard: `BudgetAmendmentTransitionGuard`), submitted→approved (role: financial-controller), submitted→rejected (role: financial-controller), approved→draft (role: financial-controller)
  - Verify lifecycle transitions are audit-trailed automatically by OR engine

- [ ] **TASK-007: Generate seed data in `lib/Settings/shillinq_register.json`**
  - 4 Budget objects: Gemeente Utrecht ICT 2026, Gemeente Rotterdam Sociaal Domein 2026, Conduction BV R&D 2026, Gemeente Eindhoven Openbare Werken 2026
  - 4 BudgetAllocation objects: ICT Infrastructuur, Softwarelicenties, Jeugdzorg inkoop, Wegonderhoud
  - 3 BudgetAmendment objects: submitted ceiling increase (Utrecht ICT), approved reallocation (Rotterdam SD), draft ceiling increase (Eindhoven Werken)
  - 4 BudgetPeriod objects: Q1+Q2 Utrecht ICT, jaarbudget Rotterdam SD, Q1 Eindhoven Werken
  - All using `@self` envelope with register/schema/slug per ADR-001
  - Import is idempotent — re-import skips existing slugs per ADR-001 §Seed data

---

## Backend Implementation

- [ ] **TASK-008: Implement `BudgetValidationService`**
  - File: `lib/Service/BudgetValidationService.php`
  - Method: `validate(string $documentType, array $documentData): void`
  - Logic: locate Budget via cost centre + GL account dimensions from document → read `remaining` calculation → if `totalAmount > remaining` and `overBudgetPrevention`: throw `BudgetExceededException` → else flag document with `overBudget=true`, `budgetOverage=(totalAmount - remaining)`
  - DI: constructor inject `ObjectService` (read Budget, PurchaseOrder aggregates); `private readonly` per ADR-003
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-8`

- [ ] **TASK-009: Implement `BudgetExceededException`**
  - File: `lib/Exception/BudgetExceededException.php`
  - Carries: budget name, ceiling, remaining, attempted amount, overage
  - Used by `BudgetValidationService` and caught in controller to return HTTP 422 with structured `message` field (no stack trace per ADR-002)
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-9`

- [ ] **TASK-010: Implement `BudgetAmendmentTransitionGuard`**
  - File: `lib/Lifecycle/BudgetAmendmentTransitionGuard.php`
  - Implements OR lifecycle guard interface
  - On `submitted → approved` transition: read linked Budget via `budgetId` relation → add `amendment.amount` to `Budget.ceiling` → save via `ObjectService.saveObject()`
  - Guard must be idempotent — if ceiling already updated (e.g. on retry), skip duplicate increment
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-10`

- [ ] **TASK-011: Implement `TaakveldValidationService`**
  - File: `lib/Service/TaakveldValidationService.php`
  - Method: `validate(string $budgetId): array` — returns list of `{ allocationId, allocationName, issue: 'missing'|'invalid', value }`
  - Load BBV Bijlage IV code list from `IAppConfig` key `shillinq.taakveld_codes`
  - Validates each BudgetAllocation linked to the budget: `taakveld` must be non-empty and in the code list
  - Used by budget publication check: blocks `status: published` if any allocation fails
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-11`

- [ ] **TASK-012: Implement `StructuralBalanceService`**
  - File: `lib/Service/StructuralBalanceService.php`
  - Method: `compute(string $budgetId): array` — returns `{ revenues, expenditures, balance, balanced, byProgramme[] }`
  - Load all BudgetAllocations where `budgetId = $budgetId AND structural = true`
  - Classify as revenue/expenditure via IV3 category (mapping stored in `IAppConfig`)
  - Sum revenues and expenditures; group by `bbvProgramme` for programme-level breakdown
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-12`

- [ ] **TASK-013: Implement `BudgetIndexationService`**
  - File: `lib/Service/BudgetIndexationService.php`
  - Method: `copyToNewFiscalYear(string $budgetId, string $newFiscalYear, float $indexationPercentage): string` — returns new budget ID
  - Clone Budget, adjust ceiling by percentage, set new fiscal year, status: draft
  - Clone all BudgetAllocations with adjusted ceiling amounts
  - Auto-generate BudgetPeriod records from the new budget's `allocationPeriod` setting
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-13`

- [ ] **TASK-014: Register event listeners on PurchaseRequisition and PurchaseOrder save**
  - Register `BudgetValidationService::validate()` as a listener on `PurchaseRequisitionSaveEvent` and `PurchaseOrderSaveEvent`
  - Catch `BudgetExceededException` and return structured HTTP 422 response
  - On non-blocking over-budget (prevention=false): patch document object with `overBudget=true` and `budgetOverage` before persisting
  - Add listener registration in `lib/AppInfo/Application.php` via event dispatcher per ADR-003

- [ ] **TASK-015: Implement `BudgetController` API endpoints**
  - File: `lib/Controller/BudgetController.php`
  - `GET /api/budgets` — list with cost centre / BBV programme / taakveld / fiscal year filters (pagination: `_page` + `_limit` per ADR-002)
  - `POST /api/budgets/{id}/validate-taakveld` — run `TaakveldValidationService::validate()`, return issues array
  - `POST /api/budgets/{id}/structural-balance` — run `StructuralBalanceService::compute()`, return balance report
  - `POST /api/budgets/{id}/copy-to-fiscal-year` — run `BudgetIndexationService::copyToNewFiscalYear()`, return new budget ID
  - `GET /api/budgets/{id}/utilisation` — return utilisation summary per dimension
  - Controller is thin (<10 lines/method): routing + validation + response only, all logic in services per ADR-003
  - `@spec openspec/changes/budget-planning-control-other-t2/tasks.md#task-15`

- [ ] **TASK-016: Register routes in `appinfo/routes.php`**
  - Add all BudgetController routes (specific routes before wildcard `{slug}` per ADR-003)
  - Include CORS OPTIONS route for any public endpoints
  - Follow URL pattern `/index.php/apps/shillinq/api/budgets` per ADR-002

- [ ] **TASK-017: Store BBV Bijlage IV taakveld code list in app configuration**
  - File: `lib/Settings/ShillinqSettings.php` or equivalent settings provider
  - Load taakveld codes via `IAppConfig` key `shillinq.taakveld_codes` (default: standard BBV Bijlage IV list)
  - Admin UI: display the code list in Settings page with documentation link to official BBV reference
  - Allow admin override for custom/extended code lists

- [ ] **TASK-018: Add Dutch i18n translations**
  - `l10n/nl.js` and `l10n/en.js`: all Budget Monitoring, Management, Approval, Compliance, Contracts UI strings
  - PHP: `$this->l->t()` for all `BudgetController` response messages
  - Required strings include: alert notification templates, validation error messages, form field labels, BBV programme names, IV3 category labels
  - Per ADR-007: minimum nl + en; date/number formatting per user locale

---

## Frontend Implementation

- [ ] **TASK-019: Register budget entity types in `store/store.js`**
  - Call `objectStore.registerObjectType('budget', 'Budget', 'shillinq')` and equivalents for BudgetAllocation, BudgetAmendment, BudgetPeriod
  - Use `createObjectStore` with `auditTrails`, `files`, `relations`, `selection` plugins

- [ ] **TASK-020: Budget list page (`src/views/BudgetIndex.vue`)**
  - Use `CnIndexPage` + `useListView('budget', { sidebarState, objectStore })`
  - Inject `sidebarState` from `App.vue`
  - Facet sidebar (`CnFacetSidebar`): filter by bbvProgramme, taakveld, allocationPeriod, fiscal year, status, budgetType
  - Show utilisation % badge per row using `CnStatusBadge`: red ≥ alertThreshold, amber ≥ warningThreshold, green otherwise
  - Row click → `$router.push({ name: 'BudgetDetail', params: { id } })`
  - Add button → `$router.push({ name: 'BudgetDetail', params: { id: 'new' } })`
  - All user-visible strings via `t(appName, 'text')` per ADR-007

- [ ] **TASK-021: Budget detail page (`src/views/BudgetDetail.vue`)**
  - Use `CnDetailPage` + `CnDetailCard` sections per ADR-004 patterns
  - Sections: overview (ceiling / committed / actuals / remaining / utilisation), allocations table, amendments list, spend breakdown by category
  - `CnTimelineStages` for BudgetAmendment lifecycle visualisation (Concept → Ingediend → Goedgekeurd)
  - Action buttons: Edit (→ CnFormDialog), Delete (→ CnDeleteDialog), "Plafondverhoging aanvragen", "Taakveld valideren", "Structurele balans", "Kopiëren naar nieuw boekjaar"
  - Props: `budgetId` from route params. `isNew = budgetId === 'new'`
  - Sidebar: `CnObjectSidebar` (files, notes, tasks, audit trail tabs)

- [ ] **TASK-022: Budget dashboard widgets (`src/views/DashboardView.vue` update)**
  - Use `CnDashboardPage` with drag-drop layout
  - 4 `CnStatsBlock` KPIs: Totaal begroot (sum active ceilings), Totaal vastgelegd (committed), Totaal besteed (actuals), Aantal overschreden (over-budget count)
  - `CnChartWidget` (bar): actual vs budget per cost centre for current fiscal period
  - `CnChartWidget` (pie): budget ceiling distribution by BBV programme
  - Fetch data via budget store's `fetchAll()` with fiscal year filter; use `Promise.all` for parallel fetching

- [ ] **TASK-023: Budget amendment workflow UI**
  - "Plafondverhoging aanvragen" dialog: `CnFormDialog` with amount (number) + justification (textarea, min 10 chars)
  - Amendments list in BudgetDetail showing status badges (`CnStatusBadge`) and approval dates
  - `CnTimelineStages`: stages derived from BudgetAmendment `x-openregister-lifecycle` states
  - Submitted amendments show "Wachten op goedkeuring" interim state

- [ ] **TASK-024: Add router entries in `src/router/index.js`**
  - Flat routes (no nesting) per ADR-004:
    - `/budgets` → `BudgetIndex` (named: `BudgetList`)
    - `/budgets/:id` → `BudgetDetail` (named: `BudgetDetail`, props: route => `{ budgetId: route.params.id }`)
    - `/budget-amendments` → `BudgetAmendmentIndex` (named: `BudgetAmendmentList`)
    - `/budget-amendments/:id` → `BudgetAmendmentDetail` (named: `BudgetAmendmentDetail`)
  - Add "Budgetten" and "Budgetwijzigingen" navigation entries in `MainMenu.vue`

---

## Testing

- [ ] **TASK-025: PHPUnit tests — `BudgetValidationServiceTest`**
  - File: `tests/Unit/Service/BudgetValidationServiceTest.php`
  - `testValidatePassesWhenSufficientBudgetRemaining` — document within remaining: no exception, no flag
  - `testValidateThrowsWhenOverBudgetPreventionEnabled` — over-limit with prevention=true: `BudgetExceededException` thrown
  - `testValidateFlagsWhenPreventionDisabled` — over-limit with prevention=false: document flagged, no exception
  - `testValidateIgnoresWhenNoBudgetFound` — no matching budget: passes without error
  - `testValidateAccountsForExistingCommittedAmount` — existing POs counted in remaining calculation
  - ≥ 5 test methods per ADR-008

- [ ] **TASK-026: PHPUnit tests — `TaakveldValidationServiceTest`**
  - File: `tests/Unit/Service/TaakveldValidationServiceTest.php`
  - `testValidateReturnsEmptyArrayWhenAllAllocationsValid`
  - `testValidateReturnsMissingIssueForEmptyTaakveld`
  - `testValidateReturnsInvalidIssueForUnknownCode`
  - `testValidateHandlesBudgetWithNoAllocations`
  - ≥ 4 test methods per ADR-008

- [ ] **TASK-027: PHPUnit tests — `StructuralBalanceServiceTest`**
  - File: `tests/Unit/Service/StructuralBalanceServiceTest.php`
  - `testComputeReturnsBalancedTrueWhenRevenuesExceedExpenditures`
  - `testComputeReturnsBalancedFalseForStructuralDeficit`
  - `testComputeGroupsByBbvProgramme`
  - `testComputeIgnoresNonStructuralAllocations`
  - `testComputeHandlesBudgetWithNoStructuralAllocations`
  - ≥ 5 test methods per ADR-008

- [ ] **TASK-028: PHPUnit tests — `BudgetAmendmentTransitionGuardTest`**
  - File: `tests/Unit/Lifecycle/BudgetAmendmentTransitionGuardTest.php`
  - `testGuardUpdatesBudgetCeilingOnApproval`
  - `testGuardIsIdempotentOnRetry`
  - `testGuardHandlesMissingBudgetRelation`
  - ≥ 3 test methods per ADR-008

- [ ] **TASK-029: PHPUnit tests — `BudgetIndexationServiceTest`**
  - File: `tests/Unit/Service/BudgetIndexationServiceTest.php`
  - `testCopyToNewFiscalYearAppliesIndexation`
  - `testCopyCreatesCorrectNumberOfPeriods`
  - `testCopyLeavesOriginalBudgetUnchanged`
  - ≥ 3 test methods per ADR-008

- [ ] **TASK-030: Playwright browser tests — budget monitoring**
  - REQ-MON-001: Warning notification sent when utilisation reaches warningThreshold
  - REQ-MON-002: Requisition submission blocked when overBudgetPrevention=true and budget exhausted
  - REQ-MON-002: Requisition flagged (not blocked) when overBudgetPrevention=false
  - REQ-MON-005: Over-budget badge visible in budget list view

- [ ] **TASK-031: Playwright browser tests — budget management**
  - REQ-MGT-001: Four quarterly BudgetPeriods auto-created when allocationPeriod=quarterly
  - REQ-MGT-003: Category pie chart renders with correct percentages
  - REQ-MGT-004: Prognose value shown alongside original budget and actuals
  - REQ-MGT-006: Indexation copy creates new budget with adjusted ceiling

- [ ] **TASK-032: Playwright browser tests — budget approval**
  - REQ-APR-003: Amendment approval lifecycle: draft → submitted → approved updates budget ceiling
  - REQ-APR-004: "Plafondverhoging aanvragen" dialog submits BudgetAmendment and notifies CFO
  - REQ-APR-005: ContractModification with over-budget scope shows budgetImpactFlag warning

- [ ] **TASK-033: Playwright browser tests — budget compliance**
  - REQ-COM-003: Taakveld validation reports missing/invalid codes; budget cannot be published
  - REQ-COM-004: Structural balance check shows "Structureel sluitend" or "Structureel tekort"
  - REQ-COM-001: Budget list filtered by BBV programme shows only matching budgets

- [ ] **TASK-034: Playwright browser tests — budget contracts**
  - REQ-CTR-001: Contract utilisation percentage shown on contract detail
  - REQ-CTR-003: Contract ceiling alert notification sent at alertThreshold
  - REQ-CTR-004: Requisition blocked when cost centre + WBS has no active BudgetAllocation

- [ ] **TASK-035: Newman/Postman collection for BudgetController**
  - File: `tests/integration/budget-planning.postman_collection.json`
  - Cover: GET /api/budgets (list + filters), POST /api/budgets/{id}/validate-taakveld, POST /api/budgets/{id}/structural-balance, POST /api/budgets/{id}/copy-to-fiscal-year, GET /api/budgets/{id}/utilisation
  - Include error cases: over-budget prevention block (422), missing budget (404), invalid taakveld code
  - Per ADR-008

---

## Documentation

- [ ] **TASK-036: Update user docs in `docs/`**
  - Create `docs/budget-monitoring.md` with screenshots of alert thresholds, dashboard widget, and over-budget flag
  - Create `docs/budget-compliance.md` explaining BBV programme setup, IV3 mapping, taakveld validation, and structural balance
  - Update `docs/FEATURES.md` with new budget planning and control section
  - English primary, Dutch recommended per ADR-009
