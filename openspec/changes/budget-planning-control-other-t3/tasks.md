# Tasks: Budget Planning & Control — Shillinq — Other T3

All tasks implement requirements from `specs.md`. Every new PHP class and public method must carry `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-N` PHPDoc tags (ADR-003).

---

## Deduplication Check

- [ ] **Task 0: Verify no overlap with platform services**
  - Grep `lib/Service/` for any existing consolidation or savings-goal service: `grep -rn "Consolidat\|SavingsOpportunity" lib/`
  - Confirm ObjectService, RegisterService, and SchemaService already cover all CRUD — no custom mappers needed
  - Confirm `CnIndexPage`, `CnDetailPage`, `CnFormDialog`, `CnProgressBar`, `CnStatsBlock`, `CnChartWidget` cover all UI — no custom components needed
  - Confirm `ExportService` + `CnMassExportDialog` covers export — no custom export controller needed
  - Document finding: "No overlap found. Custom code limited to `ConsolidationService::generateReport()` and `SavingsOpportunityService::transitionStatus()`."

---

## Backend

- [ ] **Task 1: Register object types in store init** (`src/store/store.js`)
  - Add `objectStore.registerObjectType('consolidation-group', 'ConsolidationGroup', 'shillinq')`
  - Add `objectStore.registerObjectType('consolidated-report', 'ConsolidatedReport', 'shillinq')`
  - Add `objectStore.registerObjectType('savings-opportunity', 'SavingsOpportunity', 'shillinq')`
  - Verify each is registered exactly once, kebab-case name (ADR-015 pre-commit check #5)

- [ ] **Task 2: Add generate-report route** (`appinfo/routes.php`)
  - Add: `['name' => 'consolidation#generate', 'url' => '/api/consolidated-reports/generate', 'verb' => 'POST']`
  - Ensure this route appears BEFORE any wildcard `{slug}` routes (ADR-003)

- [ ] **Task 3: Implement ConsolidationController** (`lib/Controller/ConsolidationController.php`)
  - Thin controller (<10 lines per method); inject `ConsolidationService` and `IGroupManager`
  - `generate(string $consolidationGroupId, string $fiscalYearId): JSONResponse`
    - Check `IGroupManager::isAdmin()` → return 403 if unauthorized
    - Delegate to `ConsolidationService::generateReport()`
    - Return 201 with report data on success; 422 on validation error; 500 with generic message on exception
    - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-3`
  - No `$e->getMessage()` in JSONResponse — use static messages (ADR-015 pre-commit check #3)

- [ ] **Task 4: Implement ConsolidationService** (`lib/Service/ConsolidationService.php`)
  - Inject `ObjectService` (3-arg API: `findObjects($register, $schema, $params)`)
  - `generateReport(string $groupId, string $fiscalYearId): array`
    - Fetch ConsolidationGroup via `findObject('shillinq', 'ConsolidationGroup', $groupId)`
    - Resolve linked Budget relations; throw `\InvalidArgumentException` if none linked
    - Sum `budget.amount` → totalBudget; sum `budget.spend` → totalSpend; compute variance
    - Upsert ConsolidatedReport via `saveObject('shillinq', 'ConsolidatedReport', $data)` (replace existing draft)
    - Return hydrated report array
    - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-4`
  - All positional ObjectService calls — never named args (ADR-015 pre-commit check #2)

- [ ] **Task 5: Implement SavingsOpportunityService** (`lib/Service/SavingsOpportunityService.php`)
  - Inject `ObjectService`, `NotificationService`, `IUserSession`
  - `transitionStatus(string $id, string $newStatus): array`
    - Fetch SavingsOpportunity; validate allowed transition (active→achieved, active→cancelled)
    - Reject achieved→active with `\LogicException('Achieved goals cannot be reopened')`
    - Persist status change via `saveObject()`
    - Dispatch `NotificationService` notification to `responsibleUser` on transition to `achieved`
    - Return updated object array
    - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-5`
  - Use `$user->getUID()` for audit identity — NEVER `getDisplayName()` (ADR-015)

- [ ] **Task 6: Add status finalization guard to ConsolidationService**
  - In `ConsolidationService`, before updating a ConsolidatedReport via `saveObject()`, verify status ≠ `final`
  - Return HTTP 422 with message "Finalized reports cannot be edited" if guard fails
  - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-6`

- [ ] **Task 7: Update metrics endpoint** (`lib/Controller/MetricsController.php`)
  - Add `shillinq_consolidation_groups_total` gauge
  - Add `shillinq_consolidated_reports_total{status="draft"}` and `{status="final"}` gauges
  - Add `shillinq_savings_opportunities_total{status="active|achieved|cancelled"}` gauge
  - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-7`

---

## Seed Data

- [ ] **Task 8: Add seed objects to register template** (`lib/Settings/shillinq_register.json`)
  - Add 3 ConsolidationGroup seed objects (see design.md Seed Data section)
  - Add 3 ConsolidatedReport seed objects
  - Add 4 SavingsOpportunity seed objects
  - All use `@self` envelope with `register: shillinq`, correct schema name, unique slug
  - Import is idempotent — match by slug via `ObjectService::searchObjects` with `_rbac: false`

---

## Frontend

- [ ] **Task 9: Add router routes** (`src/router/index.js`)
  - `/consolidation-groups` → `ConsolidationGroupIndex` (named: `ConsolidationGroupIndex`)
  - `/consolidation-groups/:id` → `ConsolidationGroupDetail` (named: `ConsolidationGroupDetail`, props: `id`)
  - `/consolidated-reports` → `ConsolidatedReportIndex` (named: `ConsolidatedReportIndex`)
  - `/consolidated-reports/:id` → `ConsolidatedReportDetail` (named: `ConsolidatedReportDetail`, props: `id`)
  - `/savings-opportunities` → `SavingsOpportunityIndex` (named: `SavingsOpportunityIndex`)
  - `/savings-opportunities/:id` → `SavingsOpportunityDetail` (named: `SavingsOpportunityDetail`, props: `id`)
  - All flat, named, no nesting (ADR-004); verify pre-commit check #14 (route consistency)

- [ ] **Task 10: Add navigation items** (`src/components/MainMenu.vue`)
  - Add `NcAppNavigationItem` for "Consolidatiegroepen" → `{ name: 'ConsolidationGroupIndex' }`
  - Add `NcAppNavigationItem` for "Besparingsdoelen" → `{ name: 'SavingsOpportunityIndex' }`
  - Import items from `@conduction/nextcloud-vue` (NEVER `@nextcloud/vue` — ADR-015 check #10)
  - All label strings via `t(appName, 'key')` — no hardcoded Dutch (ADR-007)

- [ ] **Task 11: Implement ConsolidationGroupIndex** (`src/views/ConsolidationGroupIndex.vue`)
  - `CnIndexPage` with `useListView('consolidation-group', { sidebarState, objectStore })`
  - Inject `sidebarState`; row click → `$router.push({ name: 'ConsolidationGroupDetail', params: { id } })`
  - Add button → navigate to `ConsolidationGroupDetail` with id='new'
  - SPDX header as first line: `<!-- SPDX-License-Identifier: EUPL-1.2 -->` (ADR-014)

- [ ] **Task 12: Implement ConsolidationGroupDetail** (`src/views/ConsolidationGroupDetail.vue`)
  - Two modes: edit (`CnFormDialog`) / view (`CnDetailPage` + `CnDetailCard`)
  - `CnDetailCard` sections: Properties, Gekoppelde budgetten (relation), Geconsolideerde rapporten (relation)
  - "Genereer Rapport" button → `axios.post('/api/consolidated-reports/generate', ...)` in `try/catch` with user notification (ADR-015 check #8)
  - `CnObjectSidebar` with Files, Notes, Audit Trail tabs
  - Props: `entityId` from route; `isNew = entityId === 'new'`
  - SPDX header

- [ ] **Task 13: Implement ConsolidatedReportIndex** (`src/views/ConsolidatedReportIndex.vue`)
  - `CnIndexPage` with `useListView('consolidated-report', ...)`
  - Columns: name, fiscalYear, totalBudget, totalSpend, variance, status, reportDate
  - Filter by fiscalYear and status via `CnFilterBar`
  - SPDX header

- [ ] **Task 14: Implement ConsolidatedReportDetail** (`src/views/ConsolidatedReportDetail.vue`)
  - `CnDetailPage` with `CnDetailGrid` showing all properties
  - "Definitief maken" button (admin only): calls `objectStore.saveObject({ ...report, status: 'final' })` in `try/catch`
  - "Exporteren" button (final status only): opens `CnMassExportDialog`
  - Hide "Bewerken" when status = `final`
  - SPDX header

- [ ] **Task 15: Implement SavingsOpportunityIndex** (`src/views/SavingsOpportunityIndex.vue`)
  - `CnIndexPage` with `useListView('savings-opportunity', ...)`
  - Columns: name, budget, targetAmount, currentAmount, progressPercent, targetDate, status
  - Progress column: `CnProgressBar` component rendering `progressPercent`
  - SPDX header

- [ ] **Task 16: Implement SavingsOpportunityDetail** (`src/views/SavingsOpportunityDetail.vue`)
  - `CnDetailPage` with `CnDetailGrid`
  - `CnProgressBar` showing currentAmount / targetAmount
  - "Markeer als behaald" button → `axios.post('/api/savings-opportunities/{id}/transition', { status: 'achieved' })` in `try/catch`
  - "Annuleren" button → `NcDialog` confirmation → transition to `cancelled`; NEVER `window.confirm()` (ADR-015)
  - Hide transition buttons when status = `achieved` or `cancelled`
  - `CnObjectSidebar` with Audit Trail tab
  - SPDX header

- [ ] **Task 17: Add dashboard widgets** (`src/views/Dashboard.vue` or equivalent)
  - Import `CnStatsBlock` and `CnChartWidget` from `@conduction/nextcloud-vue`
  - Consolidatie KPI widget: fetch latest final ConsolidatedReport for current FiscalYear; display totalBudget, totalSpend, variance; `CnEmptyState` if none found
  - Besparingsdoelen voortgang widget: fetch all SavingsOpportunities; `CnChartWidget` (donut) by status; subtitle: total target vs. total current; `CnEmptyState` if none found
  - Fetch in `Promise.all` parallel (ADR-004 dashboard pattern)
  - Color is NOT sole status distinguisher — include text labels (ADR-010 WCAG AA)
  - SPDX header

---

## Translations

- [ ] **Task 18: Add translation keys** (`l10n/nl.json`, `l10n/en.json`)
  - All strings from Tasks 10–17 collected and added
  - Dutch: "Consolidatiegroepen", "Besparingsdoelen", "Genereer Rapport", "Definitief maken", "Exporteren", "Markeer als behaald", "Annuleren", "Geen definitief rapport beschikbaar voor dit boekjaar", "Behaald", "ConsolidationGroup has no linked Budgets", etc.
  - English equivalents in `en.json`
  - Run `grep -rn "'" src/ --include='*.vue' | grep -v "this\.t\|import\|//\|console"` → must return zero hardcoded UI strings (ADR-015 check #7)

---

## Tests

- [ ] **Task 19: PHPUnit tests for ConsolidationService** (`tests/Unit/Service/ConsolidationServiceTest.php`)
  - Test: `generateReport()` with 3 linked budgets → correct totalBudget, totalSpend, variance
  - Test: `generateReport()` with no linked budgets → throws `\InvalidArgumentException`
  - Test: `generateReport()` on existing draft report → replaces draft (not duplicated)
  - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-19`

- [ ] **Task 20: PHPUnit tests for SavingsOpportunityService** (`tests/Unit/Service/SavingsOpportunityServiceTest.php`)
  - Test: `transitionStatus('active', 'achieved')` → status updated, notification dispatched
  - Test: `transitionStatus('active', 'cancelled')` → status updated, no notification
  - Test: `transitionStatus('achieved', 'active')` → throws `\LogicException`
  - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-20`

- [ ] **Task 21: PHPUnit tests for ConsolidationController** (`tests/Unit/Controller/ConsolidationControllerTest.php`)
  - Test: non-admin call to `generate()` → HTTP 403
  - Test: admin call with valid groupId and fiscalYearId → HTTP 201 with report data
  - Test: admin call with empty group → HTTP 422
  - `@spec openspec/changes/budget-planning-control-other-t3/tasks.md#task-21`

- [ ] **Task 22: API integration tests** (`tests/integration/consolidation.postman_collection.json`)
  - Collection: POST `/api/consolidated-reports/generate` (auth: admin, non-admin)
  - Collection: verify ConsolidatedReport finalization blocks editing
  - Collection: verify SavingsOpportunity status transitions

- [ ] **Task 23: Playwright browser tests** (`tests/e2e/consolidation.spec.js`, `tests/e2e/savings.spec.js`)
  - REQ-CON-001 Scenario 1: create ConsolidationGroup → appears in list
  - REQ-CON-003 Scenario 1: generate report → totalBudget, totalSpend, variance shown
  - REQ-SAV-001 Scenario 1: create SavingsOpportunity → appears in list
  - REQ-SAV-002 Scenario 2: update currentAmount to target → status transitions to achieved
  - REQ-SAV-004 Scenario 1: dashboard widget shows active goal count and progress

---

## Pre-commit verification

- [ ] **Task 24: Run all pre-commit checks** (ADR-015)
  1. SPDX headers: `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` → zero results
  2. ObjectService calls: all use 3 positional args
  3. Error responses: zero `$e->getMessage()` in JSONResponse
  4. Auth checks: every POST/PUT/DELETE controller method has `IGroupManager::isAdmin()` check
  5. Store registration: each entity registered once, kebab-case
  6. `npm run lint` → zero errors
  7. Hardcoded strings: zero non-translated UI strings
  8. try/catch: every `await store.action()` is wrapped
  9. No raw `fetch()` for mutations — use `@nextcloud/axios`
  10. Zero imports from `@nextcloud/vue` — use `@conduction/nextcloud-vue`
  11. Every `<NcFoo>` / `<CnFoo>` in templates is imported and registered in `components: {}`
  12. Type slug consistency: `consolidation-group`, `consolidated-report`, `savings-opportunity` — kebab-case everywhere
  13. Translation keys: all English in t() calls; Dutch in `l10n/nl.json`
  14. Route consistency: all entity types have matching named routes
  15. All tasks above are fully implemented, not stubbed
