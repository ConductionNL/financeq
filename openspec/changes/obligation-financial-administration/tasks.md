# Tasks: Obligation & Financial Administration — Shillinq

Implementation checklist for the obligation-financial-administration spec. All tasks are tagged with their implementation section (Backend, Frontend, Testing, etc.) and estimated story points.

## Phase 1: Core Obligation CRUD & Dashboard (Sprint 1-2)

### Backend: Data Model & Services

- [ ] **TASK-OFA-001: Create Obligation OpenRegister schema** [Backend] [3pts]
  - [ ] Define Obligation schema per ADR-011 (schema.org:Order) in `lib/Settings/shillinq_register.json`
  - [ ] Set required fields: obligationNumber, obligationDate, dueDate, amount, creditor
  - [ ] Add relations: invoiceId (many-to-one), paymentIds (one-to-many), settlementDecisionId (many-to-one)
  - [ ] Configure field validation: obligationDate and dueDate are date format, amount is number >= 0
  - [ ] Add seed data (3 example obligations) per ADR-001

- [ ] **TASK-OFA-002: Create Invoice OpenRegister schema** [Backend] [3pts]
  - [ ] Define Invoice schema per ADR-011 (schema.org:DigitalDocument)
  - [ ] Set required fields: invoiceNumber, invoiceDate, dueDate, netAmount, vatAmount, grossAmount, vatRate, creditor, recipient, paymentTerms, documentFormat
  - [ ] Add relations: obligationId (one-to-one), paymentIds (one-to-many), attachmentIds (one-to-many)
  - [ ] Configure VAT rate enum: [0, 6, 9, 21] per Dutch standards
  - [ ] Add seed data (2 example invoices)

- [ ] **TASK-OFA-003: Create ObligationTask OpenRegister schema** [Backend] [2pts]
  - [ ] Define ObligationTask schema per ADR-011 (schema.org:Task)
  - [ ] Set required fields: taskNumber, title, dueDate, status
  - [ ] Add optional fields: description, priority, aiGenerated
  - [ ] Add relations: obligationId (many-to-one), assignedTo (many-to-one)
  - [ ] Configure status enum: [open, in-progress, completed, overdue]
  - [ ] Configure priority enum: [low, medium, high, critical]

- [ ] **TASK-OFA-004: Create SettlementDecision OpenRegister schema** [Backend] [2pts]
  - [ ] Define SettlementDecision schema per ADR-011 (schema.org:DigitalDocument)
  - [ ] Set required fields: decisionNumber, decisionDate, issuedBy, totalSettledAmount
  - [ ] Add optional fields: obligationCount, decisionRationale, documentUrl
  - [ ] Add relations: obligationIds (one-to-many), complianceReportId (many-to-one)

- [ ] **TASK-OFA-005: Create ObligationService class** [Backend] [5pts]
  - [ ] Create `lib/Service/ObligationService.php` per ADR-003 (Controller → Service → Mapper)
  - [ ] Implement `createObligation(obligationData): Obligation` method
    - [ ] Auto-generate obligationNumber if not provided
    - [ ] Validate dueDate >= obligationDate
    - [ ] Validate amount > 0
    - [ ] Call ObjectService.saveObject() to persist in OpenRegister
    - [ ] Trigger ObligationTaskService to create task
    - [ ] Add @spec tag linking to tasks.md#TASK-OFA-005
  - [ ] Implement `getObligation(obligationId): Obligation` method
  - [ ] Implement `updateObligation(obligationId, updates): Obligation` method with audit trail
  - [ ] Implement `settleObligation(obligationId, paymentDate): void` method
    - [ ] Set status="settled"
    - [ ] Calculate settledOnTime = (paymentDate <= dueDate)
  - [ ] Implement `deleteObligation(obligationId): void` method (with validation: not settled)
  - [ ] Add error handling and logging per ADR-005

- [ ] **TASK-OFA-006: Create InvoiceService class** [Backend] [4pts]
  - [ ] Create `lib/Service/InvoiceService.php`
  - [ ] Implement `createInvoice(invoiceData): Invoice` method
    - [ ] Auto-generate invoiceNumber if not provided
    - [ ] Validate invoiceDate and dueDate are valid dates
    - [ ] Validate: netAmount + vatAmount = grossAmount
    - [ ] Validate: vatRate matches vatAmount / netAmount
    - [ ] Call ObjectService.saveObject()
  - [ ] Implement `getInvoice(invoiceId): Invoice` method
  - [ ] Implement `linkToObligation(invoiceId, obligationId): void` method
  - [ ] Implement `deleteInvoice(invoiceId): void` method

- [ ] **TASK-OFA-007: Create ObligationTaskService class** [Backend] [5pts]
  - [ ] Create `lib/Service/ObligationTaskService.php`
  - [ ] Implement `generateTaskForObligation(obligation): ObligationTask` method
    - [ ] Auto-generate taskNumber
    - [ ] Set title = "Pay {invoiceNumber or obligation description}"
    - [ ] Copy dueDate from obligation
    - [ ] Calculate priority based on days-to-deadline
    - [ ] Set aiGenerated = true
    - [ ] Call ObjectService.saveObject()
  - [ ] Implement `recalculateTaskPriority(taskId): void` method
    - [ ] Fetch linked obligation
    - [ ] Recalculate priority based on obligation.dueDate
    - [ ] Update task if priority changed
  - [ ] Implement `completeTask(taskId): void` method
    - [ ] Set status = "completed"
    - [ ] If all tasks for an obligation are complete, mark obligation as "settled"
  - [ ] Implement `markOverdueTask(taskId): void` method (called by daily job)

- [ ] **TASK-OFA-008: Create SettlementService class** [Backend] [6pts]
  - [ ] Create `lib/Service/SettlementService.php`
  - [ ] Implement `createSettlementDecision(decisionData, obligationIds[]): SettlementDecision` method
    - [ ] Auto-generate decisionNumber
    - [ ] Validate all obligations are in "open" or "in-progress" status
    - [ ] Calculate totalSettledAmount as sum of obligation amounts
    - [ ] Set obligationCount = count of obligationIds
    - [ ] Set status = "draft"
    - [ ] Call ObjectService.saveObject()
    - [ ] Trigger ApprovalChain workflow
  - [ ] Implement `approveSettlementDecision(decisionId): void` method
    - [ ] Validate decision status = "draft"
    - [ ] Fetch all linked obligations
    - [ ] For each obligation:
      - [ ] Set status = "settled"
      - [ ] Calculate settledOnTime = (paymentDate <= obligation.dueDate)
    - [ ] Set decision status = "approved"
    - [ ] Generate decision document (PDF)
  - [ ] Implement `finalizeSettlementDecision(decisionId): void` method
    - [ ] Set status = "finalized"
    - [ ] Update ComplianceReport metrics
    - [ ] Log to audit trail
  - [ ] Implement `getSettlementDecision(decisionId): SettlementDecision` method
  - [ ] Implement `listSettlementDecisions(filters): array` method

- [ ] **TASK-OFA-009: Create ObligationMapper class** [Backend] [2pts]
  - [ ] Create `lib/Mapper/ObligationMapper.php` per ADR-003
  - [ ] Implement CRUD methods: findAll(), find(), save(), delete()
  - [ ] NO business logic — only OpenRegister ObjectService calls
  - [ ] Implement relation loading: loadInvoice(), loadPayments(), loadSettlementDecision()

- [ ] **TASK-OFA-010: Create repair step for schema initialization** [Backend] [3pts]
  - [ ] Create `lib/Migration/ObligationSchemaRepair.php` implementing `IRepairStep`
  - [ ] In `repair()` method:
    - [ ] Call ConfigurationService.importFromApp() to load schemas from `lib/Settings/shillinq_register.json`
    - [ ] Call ConfigurationService.importFromApp() to load seed data (3 obligations, 2 invoices)
    - [ ] Use version checking to ensure idempotency
    - [ ] Log success/failure

### Frontend: Obligation List & Detail

- [ ] **TASK-OFA-011: Create ObligationIndex.vue component** [Frontend] [5pts]
  - [ ] Create `src/views/ObligationIndex.vue`
  - [ ] Use `CnIndexPage` component per ADR-004
  - [ ] Use `useListView()` composable for search/filter/sort
  - [ ] Implement filters:
    - [ ] Status (dropdown: open, settled, all)
    - [ ] Creditor (autocomplete search via ObjectService.findAll(creditor schema))
    - [ ] Amount range (two inputs: min/max)
    - [ ] Due date (date picker with presets: "This week", "This month", "Overdue")
  - [ ] Display table columns:
    - [ ] obligationNumber (clickable → detail)
    - [ ] creditor.name
    - [ ] amount (currency formatted)
    - [ ] dueDate
    - [ ] status (badge color: open=blue, settled=green, overdue=red)
    - [ ] daysToDeadline (calculated field)
  - [ ] Actions:
    - [ ] Row click → navigate to ObligationDetail
    - [ ] "Add Obligation" button → navigate to ObligationDetail with id='new'
    - [ ] "Export" button → calls ObligationController.export() with filtered query
  - [ ] Sidebar: CnIndexSidebar for saved filters

- [ ] **TASK-OFA-012: Create ObligationDetail.vue component** [Frontend] [8pts]
  - [ ] Create `src/views/ObligationDetail.vue`
  - [ ] Use `CnDetailPage` for display mode, edit form for edit mode
  - [ ] Implement form mode (id='new' or in edit):
    - [ ] Input fields:
      - [ ] obligationNumber (readonly if not 'new')
      - [ ] obligationDate (date picker, default today)
      - [ ] dueDate (date picker, required)
      - [ ] amount (number input, currency selector EUR/USD/GBP)
      - [ ] creditor (searchable select via objectStore)
      - [ ] obligationType (enum select)
      - [ ] description (textarea)
    - [ ] Form validation: dueDate >= obligationDate, amount > 0
    - [ ] "Save" button calls store action
    - [ ] "Cancel" button returns to list
  - [ ] Implement display mode:
    - [ ] Detail sections with `CnDetailCard`:
      - [ ] "Overview": obligationNumber, status badge, dueDate, daysToDeadline
      - [ ] "Details": obligationDate, amount, creditor (link), obligationType, description
      - [ ] "Invoice": linked invoice (if exists) with clickable link
      - [ ] "Payments": table of linked payments (date, amount, method, status)
      - [ ] "Settlement": linked settlement decision (if exists)
      - [ ] "Tasks": table of linked ObligationTasks (title, dueDate, priority, status)
    - [ ] "Edit" and "Delete" buttons in header
  - [ ] Sidebar with `CnObjectSidebar`:
    - [ ] Files tab (attachments)
    - [ ] Notes tab
    - [ ] Audit trail tab (change history)
  - [ ] use `useDetailView()` for load/edit/delete state management

- [ ] **TASK-OFA-013: Create obligation Pinia store** [Frontend] [4pts]
  - [ ] Create `src/store/modules/obligation.js`
  - [ ] Use `createObjectStore('Obligation')` with:
    - [ ] Schema slug: 'Obligation'
    - [ ] Register slug: 'shillinq'
    - [ ] Plugins: [filesPlugin, auditTrailsPlugin, relationsPlugin, searchPlugin]
  - [ ] Add computed properties:
    - [ ] `overdueObligations()` — filter status="open" and dueDate < today
    - [ ] `upcomingObligations()` — filter status="open" and dueDate within 7 days
    - [ ] `settlementRate()` — count of settled / total obligations
  - [ ] Add actions:
    - [ ] `createObligation(data)` — POST /api/obligations
    - [ ] `updateObligation(id, data)` — PUT /api/obligations/{id}
    - [ ] `settleObligation(id, paymentDate)` — POST /api/obligations/{id}/settle
    - [ ] `fetchObligations(filters)` — GET /api/obligations with query params

### API Controllers

- [ ] **TASK-OFA-014: Create ObligationController class** [Backend] [6pts]
  - [ ] Create `lib/Controller/ObligationController.php` per ADR-002 and ADR-003
  - [ ] Implement REST endpoints:
    - [ ] `GET /api/obligations` — list with pagination, filtering, sorting
      - [ ] Query params: `_page`, `_limit`, `status`, `creditor`, `minAmount`, `maxAmount`, `dueDateStart`, `dueDateEnd`
      - [ ] Response: JSON array + total/page/pages
      - [ ] Authorize via AuthorizationService (field-level RBAC on amounts)
    - [ ] `GET /api/obligations/{id}` — single obligation detail
    - [ ] `POST /api/obligations` — create obligation
      - [ ] Request body: obligationDate, dueDate, amount, creditor, obligationType, description
      - [ ] Response: created obligation with obligationNumber
      - [ ] Authorize: "create" action
    - [ ] `PUT /api/obligations/{id}` — update obligation
      - [ ] Request body: any updatable fields
      - [ ] Validation: no status changes via this endpoint
      - [ ] Authorize: "update" action
    - [ ] `DELETE /api/obligations/{id}` — delete obligation
      - [ ] Validate: not settled (status != "settled")
      - [ ] Authorize: "delete" action
    - [ ] `POST /api/obligations/{id}/settle` — settle obligation
      - [ ] Request body: paymentDate
      - [ ] Calls ObligationService.settleObligation()
      - [ ] Authorize: "settle" action (higher privilege)
    - [ ] `POST /api/obligations/export` — bulk export (CSV/JSON)
      - [ ] Query params: `format` (csv|json), `filters` (same as GET list)
      - [ ] Returns file download
  - [ ] Add @spec tag to each method: `@spec openspec/changes/obligation-financial-administration/tasks.md#TASK-OFA-014`
  - [ ] Error responses: appropriate HTTP status + message field (no stack traces per ADR-005)

- [ ] **TASK-OFA-015: Create InvoiceController class** [Backend] [4pts]
  - [ ] Create `lib/Controller/InvoiceController.php`
  - [ ] Implement REST endpoints:
    - [ ] `GET /api/invoices` — list with pagination
    - [ ] `GET /api/invoices/{id}` — detail
    - [ ] `POST /api/invoices` — create
    - [ ] `PUT /api/invoices/{id}` — update
    - [ ] `DELETE /api/invoices/{id}` — delete
    - [ ] `POST /api/invoices/{id}/link-obligation` — link to obligation (request: obligationId)
  - [ ] All endpoints use InvoiceService and follow ADR-002/ADR-003

- [ ] **TASK-OFA-016: Create ObligationTaskController class** [Backend] [3pts]
  - [ ] Create `lib/Controller/ObligationTaskController.php`
  - [ ] Implement endpoints:
    - [ ] `GET /api/obligation-tasks` — list by obligation or assigned user
    - [ ] `GET /api/obligation-tasks/{id}` — detail
    - [ ] `POST /api/obligation-tasks/{id}/complete` — mark complete
  - [ ] No create/delete endpoints (tasks auto-generated from obligations)

### Frontend: Dashboard

- [ ] **TASK-OFA-017: Create Dashboard.vue component** [Frontend] [8pts]
  - [ ] Create `src/views/Dashboard.vue`
  - [ ] Use `CnDashboardPage` component per ADR-004
  - [ ] Implement KPI section with 4 `CnStatsBlock` cards:
    - [ ] **Open Obligations**: count of obligations where status="open" or "in-progress"
      - [ ] Fetch from store.obligationCount (computed property)
      - [ ] Click → filter list to status="open"
    - [ ] **Overdue Obligations**: count of obligations where dueDate < today and status != "settled"
      - [ ] Badge shows count in red
      - [ ] Click → filter list to overdue
    - [ ] **Total Outstanding Value**: sum of amounts for all open obligations
      - [ ] Currency formatted (€ or selected currency)
      - [ ] Click → total detail breakdown by creditor
    - [ ] **Compliance Rate**: percentage from ComplianceReport for current month
      - [ ] Formula: (onTimeObligations / totalObligations) × 100
      - [ ] Color: red if < 90%, yellow if < 95%, green if >= 95%
      - [ ] Click → ComplianceReport detail
  - [ ] Implement obligation list grouped by deadline:
    - [ ] **Overdue** section (red header)
      - [ ] Table: obligationNumber, creditor, amount, daysOverdue
      - [ ] Sort by daysOverdue descending
    - [ ] **Due This Week** section (yellow header)
      - [ ] Table: obligationNumber, creditor, amount, daysRemaining
    - [ ] **Due Later** section (green header)
      - [ ] Table: obligationNumber, creditor, amount, daysRemaining
    - [ ] Each row clickable → detail view
    - [ ] Max 10 items per section; "View all" link if more
  - [ ] Implement assets depreciation status widget (Phase 2):
    - [ ] Placeholder with "Coming soon" message for now
  - [ ] Fetch data in parallel:
    - [ ] `Promise.all([fetchObligations(), fetchComplianceReport(), fetchAssets()])`
    - [ ] Handle loading state (spinner)
    - [ ] Handle error state (message)

---

## Phase 2: Fixed Assets & Depreciation (Sprint 3-4)

- [ ] **TASK-OFA-018: Create FixedAsset OpenRegister schema** [Backend] [2pts]
  - [ ] Define FixedAsset schema per ADR-011 (schema.org:Thing)
  - [ ] Required fields: assetNumber, name, assetType, purchaseDate, purchaseCost, status
  - [ ] Optional: location
  - [ ] Enum for assetType: [equipment, vehicle, property, building, furniture, it-hardware]
  - [ ] Enum for status: [active, inactive, retired]
  - [ ] Seed data: 2 example assets (vehicle, furniture)

- [ ] **TASK-OFA-019: Create DepreciationSchedule OpenRegister schema** [Backend] [2pts]
  - [ ] Define DepreciationSchedule schema (schema.org:Thing)
  - [ ] Required: scheduleNumber, name, startDate, depreciationMethod, annualRate, status
  - [ ] Optional: endDate, totalDepreciationAmount
  - [ ] Enum for method: [linear, declining-balance, units-of-production]
  - [ ] Enum for status: [planned, active, completed]
  - [ ] Seed data: 1 example schedule (vehicle linear)

- [ ] **TASK-OFA-020: Create AssetService class** [Backend] [5pts]
  - [ ] Create `lib/Service/AssetService.php`
  - [ ] Implement `registerAsset(assetData): FixedAsset` method
    - [ ] Auto-generate assetNumber
    - [ ] Validate purchaseDate is valid date
    - [ ] Validate purchaseCost > 0
    - [ ] Call ObjectService.saveObject()
    - [ ] Trigger DepreciationService to create schedule
  - [ ] Implement `getAsset(assetId): FixedAsset`
  - [ ] Implement `updateAsset(assetId, updates): FixedAsset`
  - [ ] Implement `retireAsset(assetId): void`

- [ ] **TASK-OFA-021: Create DepreciationCalculator service** [Backend] [6pts]
  - [ ] Create `lib/Service/DepreciationCalculator.php`
  - [ ] Implement `generateSchedule(asset): DepreciationSchedule` method
    - [ ] Based on assetType, determine useful life (e.g., vehicles=5yrs, equipment=10yrs)
    - [ ] Based on useful life, calculate annualRate (100 / usefulLifeYears)
    - [ ] Create DepreciationSchedule with:
      - [ ] startDate = asset.purchaseDate
      - [ ] endDate = startDate + usefulLifeYears
      - [ ] depreciationMethod = "linear" (default, configurable per app settings)
      - [ ] annualRate = calculated %
      - [ ] totalDepreciationAmount = asset.purchaseCost
  - [ ] Implement `calculateYearlyDepreciation(schedule, year): Depreciation` method
    - [ ] Returns object: { year, annualDepreciation, accumulatedDepreciation, bookValue }
    - [ ] annualDepreciation = purchaseCost × annualRate / 100
    - [ ] accumulatedDepreciation = sum of depreciation through year
    - [ ] bookValue = purchaseCost - accumulatedDepreciation
  - [ ] Implement `generateDepreciationBoard(schedule): array` method
    - [ ] Returns array of yearly calculations from startDate to endDate
    - [ ] Used by frontend to display depreciation table
  - [ ] Support declining-balance and units-of-production methods (future)

- [ ] **TASK-OFA-022: Create AssetController class** [Backend] [4pts]
  - [ ] Create `lib/Controller/AssetController.php`
  - [ ] Implement endpoints:
    - [ ] `GET /api/assets` — list
    - [ ] `GET /api/assets/{id}` — detail with depreciation schedule
    - [ ] `POST /api/assets` — register new asset
    - [ ] `PUT /api/assets/{id}` — update asset
    - [ ] `DELETE /api/assets/{id}` — retire/delete asset
    - [ ] `GET /api/assets/{id}/depreciation` — get depreciation board

- [ ] **TASK-OFA-023: Create AssetIndex.vue component** [Frontend] [5pts]
  - [ ] Create `src/views/AssetIndex.vue`
  - [ ] Use `CnIndexPage` with useListView
  - [ ] Display table: assetNumber, name, assetType, purchaseDate, purchaseCost, status
  - [ ] Filter: assetType, status, purchaseDateRange
  - [ ] Actions: row click → detail, add new asset

- [ ] **TASK-OFA-024: Create AssetDetail.vue component** [Frontend] [8pts]
  - [ ] Create `src/views/AssetDetail.vue`
  - [ ] Form mode (new/edit):
    - [ ] Inputs: assetNumber, name, assetType, purchaseDate, purchaseCost, status, location
    - [ ] Validation: purchaseDate valid, purchaseCost > 0
    - [ ] Save button creates/updates asset
  - [ ] Display mode:
    - [ ] Overview: assetNumber, name, assetType, purchaseDate, purchaseCost, status
    - [ ] "Depreciation" section:
      - [ ] Table with columns: Year, Annual Depreciation, Accumulated Depreciation, Book Value
      - [ ] Fetched from `/api/assets/{id}/depreciation`
      - [ ] Calculate and display all years from schedule.startDate to endDate
    - [ ] Sidebar with audit trail
  - [ ] Use `useDetailView()` for state

- [ ] **TASK-OFA-025: Create asset Pinia store** [Frontend] [3pts]
  - [ ] Create `src/store/modules/asset.js`
  - [ ] Use `createObjectStore('FixedAsset')`
  - [ ] Add actions: createAsset(), updateAsset(), retireAsset(), fetchAssets()

---

## Phase 3: Settlement Workflow & Compliance (Sprint 5-6)

- [ ] **TASK-OFA-026: Create SettlementDecision workflow** [Backend] [8pts]
  - [ ] Integrate with ApprovalChain service (existing OpenRegister component)
  - [ ] Implement settlement workflow state machine:
    - [ ] draft → pending-approval → approved → finalized
    - [ ] Transitions enforced by SettlementService methods
    - [ ] Only CFO (high privilege) can finalize
  - [ ] Generate settlement decision PDF document:
    - [ ] Create `lib/Service/SettlementDocumentGenerator.php`
    - [ ] Template: decision number, date, issuer, authorized approver, obligation table, legal statement
    - [ ] Use mPDF or similar for PDF generation (existing Nextcloud dependency)
    - [ ] Store PDF URL in SettlementDecision.documentUrl
  - [ ] Update linked obligations on approval:
    - [ ] For each obligation, set status="settled" and settledOnTime boolean
    - [ ] Calculate based on comparison of payment date with dueDate

- [ ] **TASK-OFA-027: Create ComplianceReport schema & service** [Backend] [5pts]
  - [ ] Define ComplianceReport schema per ADR-011
  - [ ] Create ComplianceService class:
    - [ ] `generateReport(reportPeriod): ComplianceReport` method
      - [ ] reportPeriod format: "2026-Q1" or "2026-01"
      - [ ] Query all obligations and payments in period
      - [ ] Calculate: totalObligations, onTimeObligations, overdueObligations, complianceRate, totalAmount, averagePaymentDays
      - [ ] Formulas:
        - [ ] complianceRate = (onTimeObligations / totalObligations) × 100
        - [ ] averagePaymentDays = average(paymentDate - dueDate) for all obligations
      - [ ] Create ComplianceReport object
      - [ ] Return for display or export
    - [ ] `exportToPowerBI(reportId): string` method (if configured)
      - [ ] Call PowerBI API to create dataset
      - [ ] Return powerBiUrl for dashboard access
    - [ ] `listReports(filters)` method

- [ ] **TASK-OFA-028: Create ComplianceController class** [Backend] [4pts]
  - [ ] Create `lib/Controller/ComplianceController.php`
  - [ ] Implement endpoints:
    - [ ] `GET /api/compliance-reports` — list reports
    - [ ] `GET /api/compliance-reports/{id}` — detail
    - [ ] `POST /api/compliance-reports/generate` — trigger generation (request: reportPeriod)
    - [ ] `POST /api/compliance-reports/{id}/export-powerbi` — export to PowerBI
    - [ ] `POST /api/compliance-reports/export-sisa` — export for SiSa compliance

- [ ] **TASK-OFA-029: Create SettlementDecision.vue workflow component** [Frontend] [8pts]
  - [ ] Create `src/views/SettlementDetail.vue`
  - [ ] Form mode (new):
    - [ ] Step 1: Select obligations from list (multi-select with checkboxes)
      - [ ] Table: obligationNumber, creditor, amount, dueDate
      - [ ] Show total amount being settled
      - [ ] Next button validates at least 1 selected
    - [ ] Step 2: Decision details (form)
      - [ ] decisionNumber (auto-generated, readonly)
      - [ ] decisionDate (default today, readonly)
      - [ ] decisionRationale (textarea)
      - [ ] Review button shows summary
    - [ ] Step 3: Review & submit
      - [ ] Summary of selected obligations
      - [ ] Approval chain info
      - [ ] Submit button sends to ApprovalChain
  - [ ] Display mode:
    - [ ] Decision header: number, date, status badge
    - [ ] Details: issuedBy, totalSettledAmount, obligationCount
    - [ ] Obligation table: linked obligations
    - [ ] Decision document link (if approved)
    - [ ] Approval history (from ApprovalChain)
  - [ ] Use `CnTabbedFormDialog` for multi-step workflow

- [ ] **TASK-OFA-030: Create ComplianceReport.vue component** [Frontend] [6pts]
  - [ ] Create `src/views/ComplianceReport.vue`
  - [ ] List view:
    - [ ] Table: reportPeriod, generatedDate, complianceRate, totalObligations, onTimeObligations, actions
    - [ ] Sort by period descending
    - [ ] Generate button → `POST /api/compliance-reports/generate` with period picker
  - [ ] Detail view (modal or page):
    - [ ] KPI cards: complianceRate (large %), totalObligations, onTimeObligations, overdueObligations, averagePaymentDays
    - [ ] Chart: line chart showing complianceRate trend over last 12 months (ApexCharts via CnChartWidget)
    - [ ] Table: drill-down by obligationType or creditor
    - [ ] Export buttons: "Export to PowerBI" (if configured), "Download as CSV"
  - [ ] Use CnChartWidget for compliance trend visualization

- [ ] **TASK-OFA-031: Create settlement & compliance Pinia store** [Frontend] [3pts]
  - [ ] Create `src/store/modules/settlement.js`
  - [ ] Create `src/store/modules/compliance.js`
  - [ ] Actions: createSettlement(), generateReport(), exportPowerBI(), etc.

---

## Phase 4: Testing & Documentation

- [ ] **TASK-OFA-032: Write ObligationService unit tests** [Testing] [6pts]
  - [ ] Create `tests/Unit/Service/ObligationServiceTest.php`
  - [ ] Test cases (≥3 methods per ADR-008):
    - [ ] `testCreateObligation()` — validates required fields, auto-generates obligationNumber, persists
    - [ ] `testCreateObligationValidation()` — validates dueDate >= obligationDate, amount > 0
    - [ ] `testSettleObligation()` — sets status, calculates settledOnTime
    - [ ] `testDeleteObligation()` — prevents deletion if settled
  - [ ] Mock ObjectService and ObligationTaskService
  - [ ] Run via `composer test:unit`

- [ ] **TASK-OFA-033: Write InvoiceService unit tests** [Testing] [4pts]
  - [ ] Create `tests/Unit/Service/InvoiceServiceTest.php`
  - [ ] Test cases:
    - [ ] `testCreateInvoice()` — validates VAT calculation
    - [ ] `testLinkToObligation()` — creates relation
    - [ ] `testDeleteInvoice()` — cascades or prevents

- [ ] **TASK-OFA-034: Write SettlementService unit tests** [Testing] [6pts]
  - [ ] Create `tests/Unit/Service/SettlementServiceTest.php`
  - [ ] Test cases:
    - [ ] `testCreateSettlementDecision()` — validates obligations, calculates total
    - [ ] `testApproveSettlement()` — updates obligations, generates document
    - [ ] `testFinalizeSettlement()` — updates metrics

- [ ] **TASK-OFA-035: Write API integration tests** [Testing] [8pts]
  - [ ] Create Newman/Postman collection: `tests/integration/obligation-financial-administration.postman_collection.json`
  - [ ] Test endpoints:
    - [ ] POST /api/obligations (create)
    - [ ] GET /api/obligations (list with filters)
    - [ ] GET /api/obligations/{id} (detail)
    - [ ] PUT /api/obligations/{id} (update)
    - [ ] POST /api/obligations/{id}/settle (settle)
    - [ ] POST /api/invoices (create)
    - [ ] POST /api/settlement-decisions (create)
    - [ ] GET /api/compliance-reports/generate (generate)
  - [ ] Verify response schemas, pagination, authorization

- [ ] **TASK-OFA-036: Write browser tests for obligation workflow** [Testing] [8pts]
  - [ ] Create Playwright tests: `tests/e2e/obligation-workflow.spec.ts`
  - [ ] Scenario 1: Create obligation from invoice
    - [ ] GIVEN: invoice in system
    - [ ] WHEN: user navigates to Invoices, clicks "Create Obligation"
    - [ ] THEN: obligation form opens, user fills details, saves
    - [ ] AND: obligation appears in list with correct data
  - [ ] Scenario 2: Settle obligation batch
    - [ ] GIVEN: 3 open obligations
    - [ ] WHEN: user creates settlement decision, selects 3, approves
    - [ ] THEN: obligations marked settled, decision document generated
  - [ ] Scenario 3: View compliance report
    - [ ] GIVEN: settled obligations in Q1
    - [ ] WHEN: user generates ComplianceReport for Q1
    - [ ] THEN: compliance rate calculated correctly, chart displays
  - [ ] Run via `npm run test:e2e`

- [ ] **TASK-OFA-037: Authorization tests** [Testing] [4pts]
  - [ ] Create `tests/Unit/Service/AuthorizationTest.php`
  - [ ] Test cases:
    - [ ] Finance Officer can create/edit obligations, cannot settle
    - [ ] Finance Manager can settle up to €10,000
    - [ ] CFO can issue settlement decisions without limit
    - [ ] Field-level RBAC: amount field hidden for non-managers

- [ ] **TASK-OFA-038: Create user documentation** [Documentation] [6pts]
  - [ ] Create `docs/obligation-financial-administration.md` with sections:
    - [ ] **Getting Started**: navigate to Obligations module, add first obligation
    - [ ] **Creating Obligations**: from invoice or manually
    - [ ] **Monitoring Obligations**: dashboard overview, filtering, exporting
    - [ ] **Settlement Workflow**: creating decisions, approval, finalization
    - [ ] **Compliance Reporting**: generating reports, interpreting metrics, PowerBI export
    - [ ] **Fixed Assets**: registering assets, depreciation schedules, board view
    - [ ] **Screenshots**: 6-8 annotated screenshots from running app
    - [ ] **Troubleshooting**: common issues and resolutions
  - [ ] Dutch translation: `docs/nl/obligatie-financiele-administratie.md`

- [ ] **TASK-OFA-039: Create API documentation** [Documentation] [4pts]
  - [ ] Document REST endpoints in OpenAPI 3.0 format
  - [ ] Add to `docs/api/openapi.yaml` or `lib/OpenApi/obligation.yaml`
  - [ ] For each endpoint:
    - [ ] Path, method, parameters, request/response schemas
    - [ ] Example requests and responses
    - [ ] Authorization requirements

- [ ] **TASK-OFA-040: Deduplication check** [Architecture] [3pts]
  - [ ] Verify no overlap with OpenRegister services:
    - [ ] ✅ ObjectService for CRUD (using it)
    - [ ] ✅ AuditTrailService for audit trail (using it)
    - [ ] ✅ RelationService for cross-entity links (using it)
    - [ ] ✅ FileService for attachments (using it)
    - [ ] ❌ No custom permission system (using AuthorizationService)
    - [ ] ❌ No custom search (using IndexService)
    - [ ] ❌ No custom import/export (using ImportService/ExportService)
  - [ ] Document findings in design.md "Reuse Analysis" section

- [ ] **TASK-OFA-041: Seed data generation task** [Data] [3pts]
  - [ ] Verify seed data is loaded via repair step (TASK-OFA-010)
  - [ ] Create 3 example obligations, 2 invoices, 2 assets in JSON format
  - [ ] Use `@self` envelope per ADR-001
  - [ ] Dutch values: Amsterdam streets, valid postcodes, realistic company names
  - [ ] Cross-reference: invoice obligationIds link to obligations, asset depreciation links to asset

---

## Phase 5: Advanced Features (Sprint 7+)

- [ ] **TASK-OFA-042: PowerBI integration** [Backend] [8pts]
  - [ ] Implement PowerBI OAuth flow
  - [ ] Create PowerBI API client service
  - [ ] Dataset schema: ComplianceReport metrics + dimensions (period, type, creditor)
  - [ ] Automatic dataset refresh on report generation

- [ ] **TASK-OFA-043: SiSa export compliance** [Backend] [8pts]
  - [ ] Implement SiSa XML/JSON export per Dutch government spec
  - [ ] Validate exported data against SiSa schema
  - [ ] Create automated testing to verify compliance

- [ ] **TASK-OFA-044: AI obligation discovery** [Backend] [13pts]
  - [ ] Integrate LLM (Claude) to extract obligations from contracts
  - [ ] Parse contract text (uploaded PDFs)
  - [ ] Identify commitment clauses, due dates, amounts
  - [ ] Suggest obligation creation with extracted data
  - [ ] Background job to process contract uploads

- [ ] **TASK-OFA-045: Advanced analytics dashboard** [Frontend] [13pts]
  - [ ] Implement `src/views/Analytics.vue`
  - [ ] Multiple charts:
    - [ ] Compliance trend (line chart)
    - [ ] Obligation distribution by type/creditor (pie/bar)
    - [ ] Payment timing distribution (histogram)
    - [ ] Cash flow forecast (area chart)
  - [ ] Drill-down capabilities
  - [ ] Custom date range picker

---

## Quality Gates

All tasks MUST pass before merge:
- ✅ PHPUnit tests: `composer check:strict`
- ✅ Frontend tests: `npm run test`
- ✅ API tests: Newman/Postman collection
- ✅ Browser tests: Playwright
- ✅ Code review: Spec traceability, ADR compliance, security review
- ✅ Documentation: User guide, API docs, screenshots
- ✅ Audit trail: All changes logged with user/timestamp

---

## Implementation Notes

- **Schema Migrations:** Use repair steps (`IRepairStep`) for schema changes; never modify migrations directly (ADR-001)
- **Spec Traceability:** Every PHP class and public method has `@spec openspec/changes/obligation-financial-administration/tasks.md#TASK-XXX` tag (ADR-003)
- **Authorization:** Use AuthorizationService for RBAC; no hardcoded permission checks (ADR-005)
- **Frontend:** Vue 2 + Pinia + @conduction/nextcloud-vue only; no custom components unless justified (ADR-004)
- **Testing:** Unit → Integration → E2E coverage; all scenarios from specs tested (ADR-008)
- **i18n:** All UI strings via `t()` function; translations in `l10n/nl.json` (ADR-007)
