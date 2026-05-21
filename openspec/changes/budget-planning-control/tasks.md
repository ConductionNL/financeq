# Tasks: Budget Planning & Control — Shillinq

## Implementation Checklist

All tasks completed to acceptance criteria defined in specs.md. Each task includes deduplication checks against OpenRegister core services and @conduction/nextcloud-vue components.

---

## Phase 1: Core Budget Entities & CRUD

### Task 1: Define OpenRegister Schemas

- [ ] **Create `lib/Settings/shillinq_budgets.json`**
  - Define Budget schema with properties: budgetName, totalAmount, startDate, endDate, currency, budgetCategory, amountSpent, alertThreshold, budgetType, fiscalYear, costCenter, attachments
  - Add relations: Organization, Location, Person, BudgetPeriod, BudgetAllocation, BudgetAmendment, ExpenditureRequest
  - Include seed data (3-5 examples with Dutch values)
  - Register with OpenRegister via `registerSchema()` in backend

- [ ] **Create `lib/Settings/shillinq_budget_allocations.json`**
  - Define BudgetAllocation schema with properties: allocationNumber, amount, status, description
  - Add relations: Budget, FundingSource, Organization
  - Include seed data

- [ ] **Create `lib/Settings/shillinq_budget_amendments.json`**
  - Define BudgetAmendment schema with properties: amendmentNumber, originalAmount, newAmount, reason, status, effectiveDate
  - Add relations: Budget, ApprovalRequest
  - Include seed data

- [ ] **Create `lib/Settings/shillinq_budget_periods.json`**
  - Define BudgetPeriod schema with properties: name, type, startDate, endDate, fiscalYear
  - Add relations: Budget (one-to-many)
  - Include seed data for FY2026, quarters, months

- [ ] **Create `lib/Settings/shillinq_expenditure_requests.json`**
  - Define ExpenditureRequest schema with properties: requestNumber, amount, purpose, status, requestDate
  - Add relations: Budget, ApprovalRequest, Person
  - Include seed data

- [ ] **Create `lib/Settings/shillinq_funding_sources.json`**
  - Define FundingSource schema with properties: name, totalAmount, status, description
  - Add relations: BudgetAllocation (one-to-many)
  - Include seed data

- [ ] **Create `lib/Settings/shillinq_locations.json`**
  - Define Location schema (schema:Place) with properties: name, code, address, region
  - Add relations: Organization, Budget (one-to-many)
  - Include seed data with Dutch locations

**Deduplication Check:**
  - Schema definitions leverage OpenRegister's schema registry — NO custom Entity/Mapper classes
  - Relations use OpenRegister's relation mechanism (register+schema+objectId)
  - Seed data follows @self envelope format
  - ✅ No duplication with core services

---

### Task 2: Implement Budget CRUD Backend

- [ ] **Create `src/Service/BudgetService.php`**
  - Implement `createBudget(array $data): Budget` with validation
  - Implement `getBudget(string $id): Budget` with relation loading
  - Implement `updateBudget(string $id, array $data): Budget` with audit logging
  - Implement `listBudgets(array $filters = [], int $page = 1, int $limit = 20): array` with pagination
  - Implement `deleteBudget(string $id)` with cascade delete validation
  - Implement `getBudgetsByOrganization(string $orgId): array` for tenant isolation
  - All methods tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-2`
  - Use ObjectService for OpenRegister CRUD, no custom mappers

- [ ] **Create `src/Controller/BudgetController.php`**
  - Implement `POST /api/budgets` (create) — validate via `BudgetService`
  - Implement `GET /api/budgets` (list with pagination, filtering by org/costCenter)
  - Implement `GET /api/budgets/{id}` (detail with relations)
  - Implement `PUT /api/budgets/{id}` (update)
  - Implement `DELETE /api/budgets/{id}` (soft delete, audit log)
  - All methods thin (<10 lines), delegating to service
  - All methods tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-2`
  - Return paginated responses with total/page/pages metadata
  - Error responses: HTTP status + message, NO stack traces

- [ ] **Deduplication Check:**
  - CnIndexPage + CnDetailPage handle UI scaffolding — NO custom list/detail controllers
  - ObjectService handles CRUD — NO custom mapper classes
  - ✅ Service implements only business logic (amountSpent calculation, commitment tracking)

**@spec openspec/changes/budget-planning-control/tasks.md#task-2**

---

### Task 3: Implement Budget List & Detail UI

- [ ] **Create `src/views/BudgetIndexPage.vue`**
  - Use `CnIndexPage` component (schema-driven)
  - Initialize with `useListView('Budget', { sidebarState, objectStore })`
  - Implement columns: budgetName, totalAmount, fiscalYear, costCenter, % Utilized, status
  - Implement filters: by organization, cost center, budget category, fiscal year, date range
  - Row click navigates to detail: `$router.push({ name: 'BudgetDetail', params: { id } })`
  - Add button creates new budget with id='new'
  - Sidebar: filtered sidebar by organization using `sidebarState.filterOrganization()`
  - Load all objects on mount: `await objectStore.getBudgets()`
  - All user-visible strings via `t(appName, 'text')` (Dutch/English)

- [ ] **Create `src/views/BudgetDetailPage.vue`**
  - Use `CnDetailPage` component with `CnDetailCard` sections
  - Display mode shows: budget summary, allocations table, expenditure requests table, amendments table
  - Edit mode: form with budgetName, totalAmount, startDate, endDate, currency, budgetCategory, costCenter fields
  - Implement sections:
    - Budget Summary: total, spent, committed, remaining, % utilized with color indicator
    - Allocations: table with allocationNumber, amount, status, fundingSource
    - Expenditure Requests: table with requestNumber, amount, purpose, status, approver
    - Amendments: table with amendmentNumber, old/new amounts, reason, status, approver
    - Forecast: EAC, exhaustion date, confidence range, burn rate (if available)
  - Sidebar: Files tab, Notes tab, Audit Trail tab, Relations tab
  - Header actions: Edit, Delete, Propose Amendment buttons
  - Load budget and relations on mount
  - Props: `budgetId` from route, `isNew = budgetId === 'new'`
  - All translations via `t(appName, 'text')`

- [ ] **Create `src/store/modules/budgetStore.js`**
  - Use `createObjectStore('budgets', 'Budget', 'budgets')` with plugins
  - Plugins: `auditTrailsPlugin`, `filesPlugin`, `relationsPlugin`
  - Expose store: `export const useBudgetStore = defineStore('budgets', budgetStore)`
  - Define state: budgets[], loading, error, selectedBudgets, filters
  - Define actions: `fetchBudgets(filters)`, `fetchBudget(id)`, `saveBudget(budget)`, `deleteBudget(id)`
  - Use `ObjectService` for API calls, NOT custom axios

- [ ] **Update `src/router.js`**
  - Add route: `{ path: '/budgets', name: 'BudgetIndex', component: BudgetIndexPage }`
  - Add route: `{ path: '/budgets/:id', name: 'BudgetDetail', component: BudgetDetailPage, props: route => ({ budgetId: route.params.id }) }`

**Deduplication Check:**
  - CnIndexPage, CnDetailPage, CnDataTable, CnDetailCard, CnDetailGrid from @conduction/nextcloud-vue — NO custom components
  - useListView, useDetailView composables handle state management — NO custom Pinia stores for list/detail
  - ✅ Store plugins (auditTrails, files, relations) are provided

**@spec openspec/changes/budget-planning-control/tasks.md#task-3**

---

## Phase 2: Real-time Budget Checking & Commitment Tracking

### Task 4: Implement Budget Consumption Calculation

- [ ] **Add method to `BudgetService`:**
  ```php
  public function calculateConsumption(string $budgetId): array {
    // Actual spend: sum invoiced ExpenditureRequests with status='executed'
    // Committed spend: sum approved PurchaseOrders linked to this budget
    // Forecasted spend: sum of uncommitted forecasted amounts
    // Return: ['actual' => X, 'committed' => Y, 'forecasted' => Z, 'total' => X+Y+Z]
  }
  ```
  - Query ExpenditureRequest with status='executed' and budget relation
  - Query PurchaseOrder with status='approved' linked via budget
  - Calculate forecast: (actual / elapsed_months) × remaining_months
  - Return as array with calculated fields
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-4`

- [ ] **Add computed field to Budget schema:**
  - `amountSpent` (number) — sum of all executed expenditure requests
  - `amountCommitted` (number) — sum of all approved POs
  - `amountForecast` (number) — burn rate projection
  - `percentUtilized` (number) — (amountSpent + amountCommitted) / totalAmount × 100
  - `remainingBalance` (number) — totalAmount - amountSpent - amountCommitted
  - Calculated on read, cached 5 minutes

- [ ] **Create `src/components/BudgetConsumptionWidget.vue`**
  - Display three columns: Actual | Committed | Forecasted
  - Progress bar showing total utilization
  - Color indicator: Green <80%, Amber 80-100%, Red >100%
  - Remaining balance highlight
  - Format amounts with currency and 2 decimal places
  - All strings translated

**@spec openspec/changes/budget-planning-control/tasks.md#task-4**

---

### Task 5: Implement Budget Check in Approval Workflow

- [ ] **Create `src/Service/BudgetCheckService.php`**
  - Implement `checkBudgetAvailability(string $budgetId, float $requestAmount): array`
  - Return: `['available' => true/false, 'remaining' => X, 'message' => '...', 'risk_level' => 'green|amber|red']`
  - Consider: actual spent + committed POs + pending approval requests
  - Consider: alert threshold (default 80%) — if total would exceed, flag as amber
  - Validate budget exists and is active (dates within range)
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-5`

- [ ] **Create budget check task in ApprovalRequest workflow:**
  - When ExpenditureRequest is submitted for approval, create ApprovalTask with budget check
  - Call `BudgetCheckService.checkBudgetAvailability()` before approver sees request
  - If check fails (available = false):
    - Block approval workflow
    - Show red banner: "Budget exceeded by €X. Request approval blocked."
    - Allow approver to: 1) Amend request amount, 2) Propose budget amendment, 3) Reject
  - If check warns (amber): show amber banner but allow approval
  - Log budget check result in ApprovalTask audit
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-5`

- [ ] **Create `src/components/BudgetCheckIndicator.vue`**
  - Display in ApprovalTask detail
  - Show: green/amber/red icon + message
  - Show: current budget, requested amount, remaining after approval
  - If red: show action buttons ("Amend Request", "Propose Amendment", "Reject")
  - Update budget data when approver refreshes page

**@spec openspec/changes/budget-planning-control/tasks.md#task-5**

---

## Phase 3: Budget Amendments & Approval Workflow

### Task 6: Implement Budget Amendment CRUD

- [ ] **Add methods to `BudgetService`:**
  ```php
  public function createAmendment(string $budgetId, array $amendmentData): BudgetAmendment
  public function getAmendment(string $amendmentId): BudgetAmendment
  public function listAmendmentsByBudget(string $budgetId): array
  public function calculateAmendmentImpact(string $amendmentId): array
  ```
  - Create amendment with status='proposed', link to Budget
  - Impact calc: return newAmount - originalAmount, % change, cash flow impact
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-6`

- [ ] **Create `src/Service/AmendmentApprovalService.php`**
  - Implement `submitAmendmentForReview(string $amendmentId): ApprovalRequest`
  - Create ApprovalRequest with two stages: Controller review → Director approval
  - Stage 1 (Controller): fiscal impact validation, vote to approve/reject/comment
  - Stage 2 (Director): final approval/rejection
  - Notify recipients via NotificationService
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-6`

- [ ] **Implement amendment status transitions:**
  - proposed → pending_approval (after Controller review passed)
  - pending_approval → approved (Director approval)
  - pending_approval → rejected (either stage rejects)
  - approved → executed (on effectiveDate arrival)
  - Each transition logged in audit trail with actor, timestamp, reason
  - Block invalid transitions (e.g., cannot go from rejected back to proposed)

**@spec openspec/changes/budget-planning-control/tasks.md#task-6**

---

### Task 7: Create Amendment UI & Workflow

- [ ] **Create `src/views/AmendmentProposalPage.vue`**
  - Form to create amendment:
    - Select existing budget (dropdown, search by name)
    - New amount (number input, validate > 0)
    - Reason textarea
    - Effective date picker (default: tomorrow)
    - File upload for supporting documents
  - On save:
    - Create amendment with status='proposed'
    - Display summary: old amount → new amount, % change
    - Show impact analysis: (from BudgetService.calculateAmendmentImpact)
    - Button: "Submit for Review" → creates ApprovalRequest, navigates to workflow
  - All strings translated

- [ ] **Create `src/views/AmendmentDetailPage.vue`**
  - Display amendment detail with status badge
  - If status='proposed': show form fields editable (edit/save/cancel)
  - If status='pending_approval': show read-only fields + reviewer feedback
  - If status='approved': show execution details + confirmation date
  - If status='rejected': show rejection reason + approve/reject comments
  - Sections:
    - Amendment Summary (old/new amount, reason, dates)
    - Impact Analysis (showing fiscal impact)
    - Approval History (table: reviewer, stage, action, date, comments)
    - Audit Trail (who did what when)
  - Header actions:
    - If proposed: "Submit for Review", "Cancel Amendment", "Delete"
    - If pending: (view-only, approvers only see their approval form)
  - Sidebar: Files, Notes, Audit Trail

- [ ] **Create `src/components/AmendmentImpactCard.vue`**
  - Display calculated impact:
    - Amount change (+ or -)
    - % change vs. original
    - Net impact on org budget pool
    - Risk assessment (low/med/high based on variance vs. historical)
    - Cash flow impact (funds freed or absorbed)
  - Color coded: savings=green, increases=amber, conflicts=red

**@spec openspec/changes/budget-planning-control/tasks.md#task-7**

---

## Phase 4: Budget Forecasting & Analytics

### Task 8: Implement EAC (Estimate at Completion) Calculation

- [ ] **Create `src/Service/ForecastService.php`**
  - Implement `calculateEAC(string $budgetId): array`
  - Logic:
    - Get budget total, start date, end date, current date
    - Get actual spend to date (sum ExpenditureRequest.status='executed')
    - Calculate elapsed months: now - startDate
    - Calculate remaining months: endDate - now
    - Calculate burn rate: actual / elapsed
    - Project remaining spend: burn_rate × remaining_months
    - EAC = actual + projected_remaining
    - Confidence range: ±10% (or configurable)
  - Return: `['eac' => X, 'variance' => variance_amount, 'variance_pct' => pct, 'confidence_range' => [min, max], 'confidence_pct' => 85, 'status' => 'green|amber|red', 'exhaustion_date' => '...']`
  - Cache result 1 hour
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-8`

- [ ] **Implement `calculateExhaustionDate(string $budgetId): DateTime`**
  - Using burn rate, calculate when budget will reach 100%
  - remaining_budget = totalAmount - actual_spend - committed_spend
  - months_until_exhaustion = remaining_budget / burn_rate
  - exhaustion_date = now + months_until_exhaustion
  - Return null if burn rate is negative (savings) or 0 (no spending)
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-8`

- [ ] **Create `src/components/ForecastPanel.vue`**
  - Display:
    - EAC amount, variance vs. budget, confidence %
    - Confidence range (min/max)
    - Status indicator (green: under budget, amber: near limit, red: over budget)
    - Exhaustion date (if forecast shows early exhaustion)
    - Recommendation text based on variance
  - If variance > 10%, show link: "Propose Budget Amendment"
  - Interactive: hover over EAC to see calculation breakdown
  - Update on-demand via "Refresh Forecast" button

**@spec openspec/changes/budget-planning-control/tasks.md#task-8**

---

### Task 9: Implement Budget Variance Report

- [ ] **Create `src/Service/VarianceReportService.php`**
  - Implement `generateVarianceReport(array $filters = [], $format = 'array'): array|CSV`
  - Filters: organization, cost center, budget category, fiscal year, date range
  - Return array:
    ```php
    [
      'summary' => [
        'total_budgets' => X,
        'total_budget_amount' => X,
        'total_actual_spend' => X,
        'total_variance' => X,
        'total_variance_pct' => X,
      ],
      'budgets' => [
        ['name' => 'Budget A', 'total' => X, 'actual' => Y, 'forecast' => Z, 'variance' => V, 'status' => 'green'],
        ...
      ]
    ]
    ```
  - Format as CSV if requested
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-9`

- [ ] **Create `src/views/VarianceReportPage.vue`**
  - Use `CnChartWidget` for visualization options:
    - Table: budgets with budget/actual/forecast/variance columns
    - Chart: horizontal bar chart (budget vs. actual vs. forecast) per budget
    - Heatmap: utilization % by cost center
  - Filters: organization, category, fiscal year, date range
  - Actions: Export as CSV, Drill down to cost center, Drill down to cost category
  - Links: click on budget row → BudgetDetailPage
  - Responsive: hide less important columns on mobile

**@spec openspec/changes/budget-planning-control/tasks.md#task-9**

---

## Phase 5: Governance & Compliance

### Task 10: Implement Budget Holder Policy Acknowledgement

- [ ] **Create `BudgetHolderAcknowledgement` OpenRegister schema**
  - Properties:
    - budgetId (relation to Budget)
    - personId (relation to Person)
    - policyVersion (string, e.g., '2026-v1')
    - acknowledgedDate (datetime)
    - signature (string or file reference)
    - notes (string, optional)
  - Immutable record after creation

- [ ] **Create `src/Service/PolicyAcknowledgementService.php`**
  - Implement `createAcknowledgement(string $budgetId, string $personId): BudgetHolderAcknowledgement`
  - Validate person is assigned as budget holder for this budget
  - Record signed acknowledgement with timestamp
  - Send confirmation notification
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-10`

- [ ] **Create `src/views/PolicyAcknowledgementModal.vue`**
  - Display when person is assigned as budget holder
  - Show policy document (embedded, with download link)
  - "I Acknowledge" button with signature
  - On acknowledgement: save record, show confirmation, dismiss modal
  - Record immutable after completion

- [ ] **Add policy acknowledgement check to budget access:**
  - Budget detail page shows warning banner if budget holder has not acknowledged
  - Budget cannot be used for expenditure requests until acknowledged
  - Compliance Officer can view all acknowledgement records

**@spec openspec/changes/budget-planning-control/tasks.md#task-10**

---

### Task 11: Implement Audit Trail & Compliance Logging

- [ ] **Leverage OpenRegister AuditTrailService (do not rebuild)**
  - Ensure Budget schema has audit enabled
  - Ensure BudgetAmendment schema has audit enabled (captures all status changes)
  - AuditTrailTab automatically available in detail sidebar
  - Budget changes logged: who, what, when, before/after snapshots
  - Amendment changes logged: proposal → approval → execution with actor at each stage
  - Tagged with `@spec openspec/changes/budget-planning-control/tasks.md#task-11`

- [ ] **Create amendment-specific audit function:**
  - Implement `logAmendmentAction(string $amendmentId, string $action, string $actor, ?string $notes)` in AmendmentApprovalService
  - Actions: 'proposed', 'reviewed', 'approved', 'rejected', 'executed'
  - Call standard AuditTrailService for immutable logging

- [ ] **Ensure GDPR compliance:**
  - No PII in error responses or logs (only IDs, not names)
  - Data subject access requests via `inzageverzoek()`
  - Budget holder acknowledgements stored per GDPR retention rules
  - DeletedBudget records soft-deleted, not hard-deleted

**@spec openspec/changes/budget-planning-control/tasks.md#task-11**

---

## Phase 6: Advanced Features (Optional / Backlog)

### Task 12: Multi-location Budget Allocation

- [ ] Extend Budget schema with `location` relation
- [ ] Implement location-specific budget breakdown
- [ ] Create allocation view by location
- [ ] Roll-up reporting across locations

**Backlog - Phase 2 or later**

---

### Task 13: Budget vs Actual Comparison (Advanced)

- [ ] Extend VarianceReportService to include forecast data
- [ ] Create comparison view with budget/actual/forecast side-by-side
- [ ] Add drill-down to cost center and cost category level
- [ ] Implement filtering by date range, category, cost center

**Backlog - Phase 2 or later**

---

## Cross-Cutting Implementation Tasks

### Task 14: Translations (i18n)

- [ ] Create `src/locale/budget-planning-control/nl.json` (Dutch)
  - All UI strings: budget, amendment, forecast, consumed, remaining, etc.
  - Button labels: Create Budget, Submit Amendment, Propose, Approve, etc.
  - Error messages and validations
  - Report labels and column headers

- [ ] Create `src/locale/budget-planning-control/en.json` (English)
  - Same keys and structure as Dutch

- [ ] Register locale files in AppComponent via `l10n.register()`

- [ ] Verify all user-visible strings use `t(appName, 'key')`

**@spec openspec/changes/budget-planning-control/tasks.md#task-14**

---

### Task 15: Testing (PHPUnit, Integration, Browser Tests)

- [ ] **Unit Tests:** `tests/Unit/Service/`
  - BudgetServiceTest (≥3 methods: createBudget, calculateConsumption, etc.)
  - ForecastServiceTest (≥3 methods: calculateEAC, calculateExhaustionDate, etc.)
  - BudgetCheckServiceTest (≥3 methods: checkBudgetAvailability variations)
  - AmendmentApprovalServiceTest (≥3 methods: status transitions, approval logic)

- [ ] **Integration Tests:** `tests/Integration/`
  - API tests via Newman/Postman collection
  - POST /api/budgets → verify schema, audit log
  - GET /api/budgets → verify pagination, filtering
  - PUT /api/budgets/{id} → verify status change
  - Amendment workflow: propose → review → approve → execute

- [ ] **Browser Tests:** `tests/Playwright/`
  - Story 1: Create budget, allocate across months, verify display
  - Story 2: Add expenditure request, see budget check warning, propose amendment
  - Story 18: Compare budget vs actual in variance report

**@spec openspec/changes/budget-planning-control/tasks.md#task-15**

---

### Task 16: Documentation

- [ ] Create `docs/budget-planning-control.md`
  - Feature overview with screenshots
  - User guide: create budget, track spend, propose amendment, review approvals
  - Administrator guide: manage budget holders, policy settings
  - API documentation: all endpoints, request/response examples

- [ ] Include screenshots of:
  - Budget detail page
  - Budget variance report
  - Amendment approval workflow
  - Forecast panel

**@spec openspec/changes/budget-planning-control/tasks.md#task-16**

---

### Task 17: Integration with Related Systems

- [ ] **Purchase Order Integration:**
  - PurchaseOrder schema must link to Budget
  - Approved PO status triggers commitment tracking update
  - PO invoice updates amountSpent in budget

- [ ] **Approval Request Integration:**
  - ExpenditureRequest integrates with ApprovalRequest
  - BudgetCheckService called during approval workflow
  - Amendment workflow uses ApprovalRequest multi-stage flow

- [ ] **Notification Integration:**
  - NotificationService alerts on:
    - Budget approaching threshold (80% utilization)
    - Budget amendment proposed (notify CFO)
    - Budget amendment approved (notify department head)
    - Budget exhaustion imminent (notification 30 days before)

**@spec openspec/changes/budget-planning-control/tasks.md#task-17**

---

## Deduplication Summary

| Feature | OpenRegister Service | Reused | Custom Required |
|---------|---------------------|--------|-----------------|
| Budget CRUD | ObjectService | ✅ | Filtering, roll-up calculation |
| Budget search | IndexService | ✅ | None — use FacetBuilder |
| Budget list UI | CnIndexPage | ✅ | Column setup only |
| Budget detail UI | CnDetailPage | ✅ | Sections & sidebar setup |
| Budget import/export | ImportService / ExportService | ✅ | None — use CnMassImportDialog |
| Budget amendments | ObjectService + ApprovalRequest | ✅ | Amendment workflow state machine |
| Amendment approvals | ApprovalRequest | ✅ | Custom approval task types |
| Budget audit | AuditTrailService | ✅ | None — automatic |
| Budget check | None (custom) | — | ✅ Calculate available balance |
| Forecasting | None (custom) | — | ✅ EAC, burn rate, exhaustion |
| Variance reporting | None (custom) | — | ✅ Report generation, aggregation |

**Conclusion:** All CRUD, search, import/export, audit, and multi-tenancy handled by OpenRegister. Custom code limited to budget-specific business logic (commitment tracking, forecasting, amendment approval state machine).

---

## Definition of Done

- [ ] All tasks completed and tested (see Task 15)
- [ ] All user stories from context-brief.md addressed in specs.md and implementation
- [ ] 100% of REQ-* requirements have acceptance criteria verified via browser tests
- [ ] All strings translated (Dutch + English)
- [ ] Audit trail complete for all budget changes and amendments
- [ ] Authorization checks in place (RBAC, multi-tenant isolation)
- [ ] Performance: budget detail loads in <2s, EAC calculates in <5s, variance report generates in <10s
- [ ] Documentation complete with screenshots
- [ ] Code merged to main and deployed to staging environment

