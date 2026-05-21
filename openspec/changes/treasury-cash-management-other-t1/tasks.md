# Implementation Tasks: Treasury & Cash Management

**Change ID:** treasury-cash-management-other-t1  
**Last Updated:** 2026-05-21  

---

## Overview

This document breaks the treasury and cash management feature set into implementation tasks with clear acceptance criteria and estimates. Tasks are organized by feature area and dependency chain.

**Total Estimated Effort:** ~380 hours (12 weeks for full-stack team)  
**Dependency Order:** Follow the checklist top-to-bottom; dependencies noted where critical.

---

## Phase 1: Backend Infrastructure (Weeks 1-3)

### Task T1.1: Deduplication Check & Entity Mapping
**Effort:** 8 hours | **Dependencies:** None  
**Status:** ⬜ Pending

Verify that all treasury entities (BankAccount, CurrencyBalance, FXExposure, etc.) already exist in OpenRegister and do not require new schema definitions.

- [ ] Search openregister/lib/Schema/ for existing BankAccount schema
- [ ] Confirm CurrencyBalance, FXExposure, LiquidityForecast exist
- [ ] Document any gaps or naming inconsistencies
- [ ] Map Shillinq entities to OpenRegister object types (register + schema names)
- [ ] Create entity mapping reference in design.md § 2.1
- [ ] Verify seed data format matches OpenRegister @self envelope convention
- [ ] Sign off: No new schemas required, existing entities sufficient

**Acceptance:** Mapping document complete, zero custom schema definitions needed, seed data format validated.

---

### Task T1.2: Register Template & Seed Data
**Effort:** 12 hours | **Dependencies:** T1.1  
**Status:** ⬜ Pending

Create lib/Settings/shillinq_register.json with seed data for treasury entities.

- [ ] Create lib/Settings/shillinq_register.json with OpenAPI 3.0 + x-openregister structure
- [ ] Define x-openregister.type: "application" for schema definitions
- [ ] Add components.schemas sections for:
  - BankAccount
  - CurrencyBalance
  - ScheduledPayment
  - FXExposure
  - LiquidityForecast
  - TreasuryTask
- [ ] Add components.objects[] with @self envelope seed data:
  - 3 BankAccount objects (EUR, USD, GBP) with Dutch realistic values
  - 3 ScheduledPayment objects (rent, insurance, SaaS)
  - 3 CurrencyBalance objects per account
  - 1 FXExposure (EUR/USD)
  - 1 LiquidityForecast (base scenario)
  - 3 TreasuryTask objects (AP, AR, scheduled)
- [ ] Validate JSON schema (via jsonschema validator or openapi-spec-validator)
- [ ] Test idempotency: Run import twice, verify no duplicates created
- [ ] Document seed data in design.md § 4
- [ ] Sign off: Register imports cleanly, seed data loads without errors

**Acceptance:** Register template passes validation, seed data idempotent, all 6 entity types present with realistic Dutch examples.

---

### Task T1.3: Create Treasury Service Classes (Skeleton)
**Effort:** 16 hours | **Dependencies:** T1.2  
**Status:** ⬜ Pending

Define stateless service classes in src/Services/Treasury/ with method stubs and PHPDoc.

- [ ] Create src/Services/Treasury/ directory structure:
  - [ ] CashFlowForecastingService.php
  - [ ] LiquidityAnalysisService.php
  - [ ] TreasuryTaskService.php
  - [ ] FXExposureCalculationService.php
  - [ ] PaymentBatchingService.php
  - [ ] ApprovalWorkflowService.php
  - [ ] BankSyncService.php
- [ ] For each service class:
  - [ ] Inject dependencies: ObjectService, RegisterService, AuthorizationService, LoggerInterface
  - [ ] Add method stubs with PHPDoc comments including @spec tags (e.g., @spec openspec/changes/treasury-cash-management-other-t1/specs.md#req-cf-001)
  - [ ] Define public methods matching spec requirements (e.g., forecast(), listTasks(), calculateExposure())
  - [ ] Add type hints for all params and returns
  - [ ] Implement DI via constructor injection with `private readonly`
- [ ] Create tests/Unit/Services/Treasury/ directory structure
- [ ] For each service: Create TreasuryServiceNameTest.php stub
- [ ] Sign off: Services compile, no undefined dependencies, PHPDoc complete

**Acceptance:** 7 service classes created, all method signatures match spec, tests directory ready, no compilation errors.

---

### Task T1.4: CashFlowForecastingService Implementation
**Effort:** 24 hours | **Dependencies:** T1.3  
**Status:** ⬜ Pending

Implement cash flow forecasting with multi-scenario support (REQ-CF-001 to REQ-CF-004).

- [ ] Implement `forecast(DateTime $start, DateTime $end, array $assumptions, string $bucketSize): LiquidityForecast`
  - [ ] Query open invoices (AR) by due date using ObjectService.findAll(Invoice)
  - [ ] Query unpaid bills (AP) by due date using ObjectService.findAll(VendorBill)
  - [ ] Query scheduled payments using ObjectService.findAll(ScheduledPayment)
  - [ ] Query bank balances from BankAccount objects
  - [ ] Group transactions by bucket (week/month/quarter) based on bucketSize param
  - [ ] For each bucket:
    - [ ] Sum inflows = (AR × arCollectionRate) + (deposits)
    - [ ] Sum outflows = AP + scheduled payments
    - [ ] Calculate net = inflows - outflows
    - [ ] Calculate cumulative balance = previous cumulative + net
  - [ ] Return LiquidityForecast object with scenarios[base] populated
  - [ ] Store forecast in OpenRegister via ObjectService.saveObject(LiquidityForecast)
- [ ] Implement scenario generation:
  - [ ] Pessimistic: arCollectionRate * 0.70
  - [ ] Base: assumptions['arCollectionRate'] (default 0.90)
  - [ ] Optimistic: arCollectionRate * 1.0
- [ ] Add assumption validation:
  - [ ] Check arCollectionRate 0-1.0
  - [ ] Check apPaymentRate 0-1.0
  - [ ] Check date range (end > start, max 365 days)
- [ ] Implement caching: Cache forecast for 24h, invalidate on new invoice/bill/payment
- [ ] Add comprehensive unit tests:
  - [ ] Test basic forecast calculation (5 invoices, 3 bills, 2 scheduled payments)
  - [ ] Test multi-scenario output
  - [ ] Test bucket aggregation (week, month, quarter)
  - [ ] Test edge case: no invoices (zero inflows)
  - [ ] Test assumption override
- [ ] Sign off: All spec scenarios covered, tests pass, performance <10s for 90-day forecast

**Acceptance:** forecast() method handles all GIVEN/WHEN/THEN scenarios from REQ-CF-001 to CF-004, ≥3 unit tests pass, caching implemented.

---

### Task T1.5: TreasuryTaskService Implementation
**Effort:** 20 hours | **Dependencies:** T1.3  
**Status:** ⬜ Pending

Aggregate AP/AR/scheduled items into unified task inbox (REQ-TI-001 to REQ-TI-004).

- [ ] Implement `listTasks(array $filters, int $limit, int $offset): array[TreasuryTask]`
  - [ ] Query TreasuryTask objects from OpenRegister
  - [ ] Apply filters: status, category, priority, assignee, dueDate range
  - [ ] Apply pagination: limit + offset
  - [ ] Return paginated result with total count
- [ ] Implement task auto-generation methods:
  - [ ] `createFromInvoice(Invoice $invoice): TreasuryTask`
    - [ ] Extract amount, dueDate, customer from invoice
    - [ ] Calculate priority: urgent if >60 days overdue, high if >30 days
    - [ ] Set category = 'ar'
    - [ ] Create TreasuryTask via ObjectService.saveObject()
  - [ ] `createFromBill(VendorBill $bill): TreasuryTask`
    - [ ] Extract amount, dueDate, supplier
    - [ ] Check for early payment discount deadline
    - [ ] Set priority = 'high' if discount deadline within 3 days
    - [ ] Set category = 'ap'
    - [ ] Create TreasuryTask via ObjectService.saveObject()
  - [ ] `createFromScheduledPayment(ScheduledPayment $scheduled): TreasuryTask`
    - [ ] Extract nextDueDate, amount, description
    - [ ] Set priority = 'normal'
    - [ ] Set category = 'payment'
    - [ ] Create TreasuryTask
- [ ] Implement background job to periodically create/update tasks:
  - [ ] Register background job: TreasuryTaskSyncJob
  - [ ] Job runs every 1 hour
  - [ ] Scans all open invoices/bills/scheduled payments
  - [ ] Creates/updates corresponding TreasuryTask records
  - [ ] Handles deletions: Remove TreasuryTask if related invoice paid/closed
- [ ] Add unit tests:
  - [ ] Test listTasks() with various filters
  - [ ] Test task creation from invoice (priority calculation)
  - [ ] Test task creation from bill (discount deadline)
  - [ ] Test background job sync
- [ ] Sign off: Task creation logic matches spec, background job integrated, tests pass

**Acceptance:** createFromInvoice/Bill/ScheduledPayment methods implement REQ-TI-002, listTasks() returns paginated results, background job syncs hourly.

---

### Task T1.6: FXExposureCalculationService Implementation
**Effort:** 18 hours | **Dependencies:** T1.3  
**Status:** ⬜ Pending

Calculate and track foreign exchange exposure by currency pair (REQ-FX-001 to REQ-FX-003).

- [ ] Implement `calculateExposure(string $baseCurrency, string $foreignCurrency): FXExposure`
  - [ ] Query open AR invoices in foreignCurrency
  - [ ] Query unpaid AP bills in foreignCurrency
  - [ ] Query bank balances in foreignCurrency
  - [ ] Query scheduled payments in foreignCurrency
  - [ ] Sum all: totalExposure = sum of all amounts
  - [ ] Bucket by maturity: dueIn0to30Days, dueIn31to90Days, etc.
  - [ ] Look up current market rate (from FX feed)
  - [ ] Calculate unrealizedGain = (marketRate - entryRate) × exposure
  - [ ] Create/update FXExposure object
  - [ ] Return FXExposure object
- [ ] Implement hedge tracking:
  - [ ] Add hedge relationship to FXExposure: hedges[] array
  - [ ] When hedge added, recalculate uncovered portion
  - [ ] Update unrealizedGain for hedged portion vs. unhedged
- [ ] Implement caching:
  - [ ] Cache exposure calculations for 1 hour
  - [ ] Invalidate on new invoice/payment/bank sync
- [ ] Add unit tests:
  - [ ] Test exposure calculation (5 USD invoices + 3 bills + balance)
  - [ ] Test maturity bucketing
  - [ ] Test unrealized gain calculation
  - [ ] Test hedge coverage
- [ ] Sign off: Exposure calculation matches spec, bucketing correct, tests pass

**Acceptance:** calculateExposure() method returns correct totalExposure and buckets, hedge tracking accurate, caching implemented.

---

### Task T1.7: BankSyncService Implementation (Core Skeleton)
**Effort:** 16 hours | **Dependencies:** T1.3  
**Status:** ⬜ Pending

Implement bank transaction import and matching (REQ-BA-002 to REQ-BA-003).

- [ ] Implement `sync(BankAccount $account, DateTime $since = null): SyncResult`
  - [ ] Check if bank API provider configured (TBD: Plaid, TrueLayer, etc.)
  - [ ] Call provider API to fetch transactions since $since or last 90 days
  - [ ] Handle API errors, rate limits, timeouts with retry logic
  - [ ] For each transaction returned:
    - [ ] Create Transaction object (if not already exists by bankTxnId)
    - [ ] Extract: amount, currency, date, description, merchant
    - [ ] Set status = 'unmatched' initially
  - [ ] Update BankAccount.balance from API response
  - [ ] Set BankAccount.syncStatus = 'synced'
  - [ ] Set BankAccount.lastSyncAt = now
  - [ ] Return SyncResult with counts: synced, matched, unmatched
- [ ] Implement `matchToLedger(Transaction $txn): ?JournalEntry`
  - [ ] Query JournalEntries by:
    - [ ] Amount within tolerance (±0.01)
    - [ ] Date within 3-day window
    - [ ] GL account matches BankAccount.generalLedgerAccount
  - [ ] If exact match found: return JournalEntry
  - [ ] Otherwise: return null
- [ ] Add background job: BankSyncJob
  - [ ] Register job with frequency: daily at 06:00
  - [ ] Job iterates all BankAccounts with syncStatus.enabled
  - [ ] Calls sync() for each
  - [ ] Logs results (counts, errors)
  - [ ] Notifies user if unmatched transactions exist
- [ ] Add unit tests:
  - [ ] Test sync() with mock API response
  - [ ] Test transaction creation and deduplication
  - [ ] Test matchToLedger() with exact and near-match scenarios
  - [ ] Test background job execution
- [ ] Sign off: Sync logic matches spec, matching ≥90% accuracy, job integrated

**Acceptance:** sync() imports transactions correctly, matchToLedger() finds matches with >90% accuracy, background job registered and tested.

---

## Phase 2: API & Approval Workflows (Weeks 4-6)

### Task T2.1: REST API Endpoints - Treasury Tasks
**Effort:** 12 hours | **Dependencies:** T1.5  
**Status:** ⬜ Pending

Implement API endpoints for task inbox (REQ-TI-001 to REQ-TI-003).

**Endpoints:**

- [ ] `GET /index.php/apps/shillinq/api/treasury/tasks`
  - [ ] Query params: status, category, priority, assignee, dueDate_from, dueDate_to, page, limit
  - [ ] Return: { data: [TreasuryTask], total, page, pages }
  - [ ] Controller: TreasuryController::getTasks()
  - [ ] Calls: TreasuryTaskService::listTasks()
  - [ ] Authorization: Check Treasury role

- [ ] `PATCH /index.php/apps/shillinq/api/treasury/tasks/{taskId}`
  - [ ] Payload: { status, priority, assignedTo, notes }
  - [ ] Updates TreasuryTask via ObjectService.saveObject()
  - [ ] Return updated task
  - [ ] Authorization: Owner or manager can edit

- [ ] `POST /index.php/apps/shillinq/api/treasury/tasks/{taskId}/complete`
  - [ ] Payload: { paymentDate, paymentMethod, bankAccountId, reference }
  - [ ] Creates Payment object (REQ-TI-003)
  - [ ] Updates TreasuryTask.status = 'completed'
  - [ ] Posts JournalEntry to ledger
  - [ ] Return: { success, paymentId, taskId }

**Implementation:**

- [ ] Create src/Controller/TreasuryController.php
  - [ ] Inject TreasuryTaskService, ObjectService, AuthorizationService, LoggerInterface
  - [ ] Implement 3 methods above with thin routing logic
  - [ ] Add PHPDoc with @spec tags
  - [ ] Input validation: Enum checks (status, priority), date format validation
  - [ ] Error handling: Return appropriate HTTP status codes (200, 400, 403, 500)
- [ ] Register routes in appinfo/routes.php:
  - [ ] GET /treasury/tasks → getTasks()
  - [ ] PATCH /treasury/tasks/{taskId} → updateTask()
  - [ ] POST /treasury/tasks/{taskId}/complete → completeTask()
- [ ] Add request/response logging (no PII)
- [ ] Create tests/Integration/TreasuryControllerTest.php
  - [ ] Test getTasks() with various filters
  - [ ] Test updateTask() authorization checks
  - [ ] Test completeTask() with payment creation
- [ ] Sign off: All endpoints tested, authorization enforced, spec compliant

**Acceptance:** 3 endpoints implemented, ≥3 integration tests pass, authorization checks present, no PII in logs.

---

### Task T2.2: REST API Endpoints - Cash Flow Forecast
**Effort:** 10 hours | **Dependencies:** T1.4  
**Status:** ⬜ Pending

Implement cash flow forecast API (REQ-CF-001 to REQ-CF-004).

**Endpoints:**

- [ ] `POST /index.php/apps/shillinq/api/treasury/cash-flow/forecast`
  - [ ] Payload: { startDate, endDate, bucketSize, assumptions, scenario }
  - [ ] Calls: CashFlowForecastingService::forecast()
  - [ ] Returns: LiquidityForecast object
  - [ ] Cache result for 24h (Redis or file-based)

- [ ] `GET /index.php/apps/shillinq/api/treasury/cash-flow/forecast?startDate=...&endDate=...`
  - [ ] Query params: startDate, endDate, bucketSize (week|month|quarter), scenario (base|pessimistic|optimistic)
  - [ ] Check cache, return cached if fresh
  - [ ] Otherwise call forecast() endpoint above
  - [ ] Returns: { periods: [...], assumptions: {...}, confidence }

- [ ] `POST /index.php/apps/shillinq/api/treasury/cash-flow/forecast/export`
  - [ ] Payload: { forecastId, format }
  - [ ] Format: csv | excel | pdf
  - [ ] Generate file and return download link
  - [ ] For PDF: Include chart visualization

**Implementation:**

- [ ] Extend TreasuryController with forecast methods
  - [ ] postForecast(): Generate and cache
  - [ ] getForecast(): Retrieve with caching
  - [ ] exportForecast(): Generate file
- [ ] Add validation:
  - [ ] endDate > startDate
  - [ ] Max range: 365 days
  - [ ] assumptions values: 0-1.0
- [ ] Add export logic:
  - [ ] CSV: Pipe-delimited with headers
  - [ ] Excel: XLSX with chart (uses PHPOffice\PhpSpreadsheet)
  - [ ] PDF: HTML-to-PDF (uses DOMPDF or similar)
- [ ] Create tests/Integration/ForecastEndpointsTest.php
  - [ ] Test forecast generation
  - [ ] Test caching behavior
  - [ ] Test export formats
- [ ] Sign off: Forecast API working, caching functional, export formats correct

**Acceptance:** Forecast endpoints return correct scenarios, caching works, export formats generate valid files.

---

### Task T2.3: REST API Endpoints - FX Exposure
**Effort:** 8 hours | **Dependencies:** T1.6  
**Status:** ⬜ Pending

Implement FX exposure tracking API (REQ-FX-001 to REQ-FX-004).

**Endpoints:**

- [ ] `GET /index.php/apps/shillinq/api/treasury/fx-exposure`
  - [ ] Query params: baseCurrency, foreignCurrency (optional; if omitted, return all pairs)
  - [ ] Returns: array[FXExposure] with exposure, buckets, hedge status, unrealizedGain

- [ ] `POST /index.php/apps/shillinq/api/treasury/fx-exposure/{baseCurrency}/{foreignCurrency}/hedge`
  - [ ] Payload: { hedgeType, amount, rate, settlementDate }
  - [ ] Calls: FXExposureCalculationService to record hedge
  - [ ] Updates FXExposure.hedges array
  - [ ] Returns updated FXExposure

- [ ] `GET /index.php/apps/shillinq/api/treasury/accounts`
  - [ ] Returns: array[BankAccount] with balance, currency, GL account, lastSync
  - [ ] Calculates consolidated balance in base currency

**Implementation:**

- [ ] Extend TreasuryController with exposure methods
  - [ ] getExposures(): Query FXExposure objects
  - [ ] createHedge(): Add hedge to exposure
  - [ ] getAccounts(): List bank accounts with balances
- [ ] Add FX rate lookup:
  - [ ] Call FX rate feed service (TBD: ECB API, Xe.com, etc.)
  - [ ] Use current market rate for calculations
  - [ ] Cache rates for 24h
- [ ] Create tests/Integration/FXExposureTest.php
  - [ ] Test exposure calculation
  - [ ] Test hedge creation and coverage %
  - [ ] Test consolidated balance calculation
- [ ] Sign off: Exposure API returns correct data, hedges tracked, balance consolidation accurate

**Acceptance:** getExposures() returns all pairs with correct buckets, hedges trackable, account consolidation accurate.

---

### Task T2.4: Approval Workflow Service & API
**Effort:** 20 hours | **Dependencies:** T1.3, T2.1  
**Status:** ⬜ Pending

Implement payment approval chains and notification workflow (REQ-PW-001 to REQ-PW-003).

**Service Implementation:**

- [ ] Implement ApprovalWorkflowService:
  - [ ] `getApprovalChain(PaymentBatch $batch): ApprovalChain`
    - [ ] Query ApprovalChain rules
    - [ ] Match trigger conditions (amount ≥ threshold)
    - [ ] Return applicable chain
  - [ ] `initializeApproval(PaymentBatch $batch): ApprovalRequest`
    - [ ] Get approval chain
    - [ ] Create ApprovalRequest for first approver
    - [ ] Set status = 'pending'
    - [ ] Return request
  - [ ] `approveRequest(ApprovalRequest $request, User $approver): ApprovalRequest`
    - [ ] Verify approver in chain
    - [ ] Mark approval with timestamp
    - [ ] Move to next approver or approve
    - [ ] Return updated request
  - [ ] `rejectRequest(ApprovalRequest $request, string $reason): ApprovalRequest`
    - [ ] Mark rejected with reason
    - [ ] Return to initiator for editing
    - [ ] Return updated request

**API Endpoints:**

- [ ] `GET /index.php/apps/shillinq/api/treasury/approval-chains`
  - [ ] Returns: array[ApprovalChain] (admin only)

- [ ] `POST /index.php/apps/shillinq/api/treasury/approval-chains`
  - [ ] Payload: { name, triggerCondition, approvers, requireAll }
  - [ ] Creates ApprovalChain
  - [ ] Admin only

- [ ] `GET /index.php/apps/shillinq/api/treasury/approvals/pending`
  - [ ] Returns: array[ApprovalRequest] awaiting current user's approval

- [ ] `POST /index.php/apps/shillinq/api/treasury/approvals/{requestId}/approve`
  - [ ] Payload: (optional) { comments }
  - [ ] Calls ApprovalWorkflowService::approveRequest()
  - [ ] Sends notification to next approver (if sequential)
  - [ ] Returns updated request

- [ ] `POST /index.php/apps/shillinq/api/treasury/approvals/{requestId}/reject`
  - [ ] Payload: { reason, comments }
  - [ ] Calls ApprovalWorkflowService::rejectRequest()
  - [ ] Sends notification to initiator
  - [ ] Returns updated request

**Implementation:**

- [ ] Create src/Services/Treasury/ApprovalWorkflowService.php with above methods
- [ ] Extend TreasuryController with approval endpoints
- [ ] Integrate NotificationService:
  - [ ] Send notification to next approver: "Approval needed: {amount} {currency}"
  - [ ] Send notification to initiator on rejection: "Batch rejected: {reason}"
- [ ] Add unit tests:
  - [ ] Test chain initialization
  - [ ] Test sequential approval flow
  - [ ] Test rejection and return to draft
- [ ] Sign off: Approval workflow enforced, notifications sent, tests pass

**Acceptance:** ApprovalWorkflowService implements REQ-PW-001 to PW-003, notifications trigger correctly, authorization enforced.

---

### Task T2.5: Payment Batching & Submission API
**Effort:** 16 hours | **Dependencies:** T2.1, T2.4  
**Status:** ⬜ Pending

Implement payment batch creation and bank submission (REQ-BP-001 to REQ-BP-003).

**Service Implementation:**

- [ ] Implement PaymentBatchingService:
  - [ ] `createBatch(array $paymentIds, ApprovalChain $approvalChain, DateTime $executionDate): PaymentBatch`
    - [ ] Aggregate selected payments
    - [ ] Validate: All same currency? All from same account? Total amount reasonable?
    - [ ] Create PaymentBatch object
    - [ ] Initialize approval workflow
    - [ ] Return batch
  - [ ] `generateBankFile(PaymentBatch $batch, string $format): string`
    - [ ] Format: sepa | ach | swift (TBD: implement SEPA first)
    - [ ] For SEPA: Generate pain.001 XML using Omnipay or similar library
    - [ ] Return file content (XML string)
  - [ ] `submitBatch(PaymentBatch $batch): SubmissionResponse`
    - [ ] Verify approval complete
    - [ ] Call payment gateway API (TBD provider)
    - [ ] Receive confirmation and batch reference
    - [ ] Update PaymentBatch.status = 'submitted'
    - [ ] Update Payment.status = 'initiated' for each payment
    - [ ] Store bank reference
    - [ ] Return response with reference number

**API Endpoints:**

- [ ] `POST /index.php/apps/shillinq/api/treasury/payment-batches`
  - [ ] Payload: { paymentIds: [...], approvalChainId, executionDate }
  - [ ] Calls PaymentBatchingService::createBatch()
  - [ ] Returns created batch with status

- [ ] `POST /index.php/apps/shillinq/api/treasury/payment-batches/{batchId}/bank-file`
  - [ ] Query param: format (sepa|ach|swift)
  - [ ] Calls PaymentBatchingService::generateBankFile()
  - [ ] Returns file as download

- [ ] `POST /index.php/apps/shillinq/api/treasury/payment-batches/{batchId}/submit`
  - [ ] Payload: (empty, or optional comments)
  - [ ] Verifies approval complete
  - [ ] Calls PaymentBatchingService::submitBatch()
  - [ ] Returns submission confirmation

- [ ] `GET /index.php/apps/shillinq/api/treasury/payment-batches/{batchId}`
  - [ ] Returns: PaymentBatch with detailed payment list and status

- [ ] `GET /index.php/apps/shillinq/api/treasury/payment-batches`
  - [ ] Query params: status, dateFrom, dateTo
  - [ ] Returns: array[PaymentBatch] paginated

**Implementation:**

- [ ] Create PaymentBatchingService with above methods
- [ ] Implement SEPA pain.001 XML generation using Omnipay\Omnipay or symfony/serializer
- [ ] Add bank submission:
  - [ ] TBD: Which payment gateway (Stripe, Wise, direct bank API)?
  - [ ] Create adapter interface: PaymentGatewayAdapter
  - [ ] Implement stub for TBD provider
- [ ] Add error handling:
  - [ ] API failures → queue for retry
  - [ ] Validation errors → return 400 with details
  - [ ] Authorization failures → return 403
- [ ] Create tests/Integration/PaymentBatchTest.php
  - [ ] Test batch creation from multiple payments
  - [ ] Test SEPA file generation
  - [ ] Test submission (mock API)
  - [ ] Test failed submission handling
- [ ] Sign off: Batching logic correct, SEPA generation valid, API working

**Acceptance:** Batch creation, bank file generation, and submission all working per REQ-BP-001 to BP-003, tests pass.

---

## Phase 3: Frontend & Dashboard (Weeks 7-9)

### Task T3.1: Task Inbox Page UI
**Effort:** 20 hours | **Dependencies:** T2.1  
**Status:** ⬜ Pending

Build Treasury Task inbox page with filtering, sorting, and bulk actions.

**Components:**

- [ ] Create src/views/pages/TreasuryTaskInbox.vue
  - [ ] Page structure:
    - [ ] Header: "Treasury Tasks" + Stats block (4 KPI cards)
    - [ ] Filter bar: status, category, priority, assignee, dueDate range
    - [ ] Data table: columns = Title, Amount, Due Date, Category, Priority, Status, Assignee
    - [ ] Pagination: 50 rows/page, navigate between pages
  - [ ] Composition:
    - [ ] `<script>` uses useListView(entityType, filters) + objectStore
    - [ ] `<template>` uses CnActionsBar + CnDataTable + CnPagination
    - [ ] Filterable & sortable by: status, category, priority, dueDate
    - [ ] Row click → navigates to TreasuryTaskDetail page
    - [ ] Checkbox select → enables bulk action buttons (Mark Complete, Batch, Reassign)

- [ ] Create src/views/pages/TreasuryTaskDetail.vue
  - [ ] Detail page structure:
    - [ ] Header: Task title + amount + due date
    - [ ] Left panel: CnDetailGrid with task details
      - [ ] Title, Category, Amount, Currency, Due Date, Priority, Status, Assigned To
      - [ ] Edit button → edit form
    - [ ] Right panel: CnObjectSidebar with tabs
      - [ ] Related Invoice/Bill preview
      - [ ] Notes (editable)
      - [ ] Approval Chain (if applicable)
      - [ ] Comments/Activity (ActivityService)
    - [ ] Action buttons:
      - [ ] "Mark Complete" → Opens payment dialog
      - [ ] "Reassign" → User selector dialog
      - [ ] "Add to Batch" (for AP items) → Batch creation dialog
      - [ ] "Edit" / "Delete" (if allowed)

**Implementation:**

- [ ] Create src/store/modules/treasuryTasks.js (Pinia store)
  - [ ] Use createObjectStore('treasuryTasks', 'TreasuryTask', 'shillinq-treasury-tasks')
  - [ ] Add plugins: auditTrails, relations, files, selection
  - [ ] Export objectStore ready for CnIndexPage/CnDetailPage

- [ ] Add route handlers in src/router/index.js
  - [ ] Route: { path: '/treasury/tasks', component: TreasuryTaskInbox }
  - [ ] Route: { path: '/treasury/tasks/:id', component: TreasuryTaskDetail }

- [ ] Add i18n strings in src/locales/en.json and nl.json
  - [ ] All user-visible text via t(appName, 'key')
  - [ ] Keys: task_inbox_title, mark_complete, batch_payment, reassign, etc.

- [ ] Create tests/Unit/views/TreasuryTaskInbox.test.js
  - [ ] Test page loads with mock data
  - [ ] Test filtering (click filter button, verify rows updated)
  - [ ] Test sorting (click column header, verify order changed)
  - [ ] Test row click navigation
  - [ ] Test bulk select + action buttons

- [ ] Create tests/Unit/views/TreasuryTaskDetail.test.js
  - [ ] Test detail loads for task
  - [ ] Test edit form opens/closes
  - [ ] Test reassign button opens dialog
  - [ ] Test mark complete button

- [ ] Sign off: Page renders correctly, filters work, API calls made on user action

**Acceptance:** Task inbox page loads, filters/sorts, table paginated; detail page shows task + related objects; ≥3 unit tests pass.

---

### Task T3.2: Cash Flow Forecast Page UI
**Effort:** 18 hours | **Dependencies:** T2.2  
**Status:** ⬜ Pending

Build cash flow forecast visualization and export page.

**Components:**

- [ ] Create src/views/pages/CashFlowForecast.vue
  - [ ] Page structure:
    - [ ] Header: "Cash Flow Forecast"
    - [ ] Input controls:
      - [ ] Start Date picker
      - [ ] End Date picker
      - [ ] Bucket Size selector: Week | Month | Quarter
      - [ ] Scenario selector: Pessimistic | Base | Optimistic
      - [ ] "Generate Forecast" button
    - [ ] Display area:
      - [ ] CnChartWidget (line chart): Cumulative balance projection
        - [ ] X-axis: periods (dates)
        - [ ] Y-axis: balance (EUR or target currency)
        - [ ] Legend: Pessimistic, Base, Optimistic (color-coded)
        - [ ] Tooltip: Show exact values on hover
      - [ ] CnTableWidget: Forecast periods
        - [ ] Columns: Period | Inflows | Outflows | Net Cash | Cumulative Balance
        - [ ] Each row = one period
      - [ ] Assumptions panel (collapsible):
        - [ ] Display assumptions used (AR collection %, AP payment %, etc.)
        - [ ] Link to edit/override assumptions

- [ ] Create src/views/pages/ForecastAssumptions.vue (modal or detail page)
  - [ ] Form inputs:
    - [ ] AR Collection Rate: slider 0-100% (default 90%)
    - [ ] AP Payment Rate: slider 0-100% (default 100%)
    - [ ] Include Other Income: toggle
    - [ ] FX Rate Source: dropdown (ECB, manual, etc.)
    - [ ] Notes: text area
  - [ ] Buttons: [Apply] [Cancel]
  - [ ] On Apply: Regenerate forecast with new assumptions, show comparison

- [ ] Create src/views/components/ForecastExportDialog.vue
  - [ ] Dialog content:
    - [ ] Format selector: CSV | Excel | PDF
    - [ ] Include options: Periods, Assumptions, Chart (if PDF)
    - [ ] Buttons: [Download] [Cancel]
  - [ ] On Download: Call export API, download file

**Implementation:**

- [ ] Create src/store/modules/forecast.js (Pinia store)
  - [ ] State: { forecast, assumptions, loading, error }
  - [ ] Actions:
    - [ ] generateForecast(startDate, endDate, bucketSize, scenario)
    - [ ] updateAssumptions(newAssumptions)
    - [ ] exportForecast(format)
  - [ ] Getters: scenarioData, chartsData, forecastPeriods

- [ ] Chart implementation (using CnChartWidget which wraps ApexCharts):
  - [ ] Line chart with 3 series (Pessimistic, Base, Optimistic)
  - [ ] Shaded confidence band between series
  - [ ] Cursor crosshair for value inspection

- [ ] Add i18n strings: forecast_title, generate, assumptions, export, etc.

- [ ] Create tests/Unit/views/CashFlowForecast.test.js
  - [ ] Test page loads
  - [ ] Test input controls (date picker, bucket selector)
  - [ ] Test forecast generation button calls API
  - [ ] Test chart renders with data
  - [ ] Test assumptions panel opens/closes
  - [ ] Test export dialog opens

- [ ] Sign off: Forecast page renders, inputs work, chart displays, export triggers

**Acceptance:** Forecast page loads data, chart displays 3 scenarios, assumptions editable, export formats available.

---

### Task T3.3: FX Exposure Dashboard UI
**Effort:** 16 hours | **Dependencies:** T2.3  
**Status:** ⬜ Pending

Build FX exposure tracking and hedging page.

**Components:**

- [ ] Create src/views/pages/FXExposure.vue
  - [ ] Page structure:
    - [ ] Header: "FX Exposure Management"
    - [ ] Filter controls:
      - [ ] Base Currency selector (default: EUR)
      - [ ] Foreign Currency selector (optional; if blank, show all pairs)
    - [ ] Display sections:
      - [ ] Exposure Heatmap (CnChartWidget type: heatmap or table)
        - [ ] Rows: currency pairs (EUR/USD, EUR/GBP, etc.)
        - [ ] Columns: maturity buckets (0-30, 31-90, 91-180, 180+)
        - [ ] Cell values: exposure amounts with color coding (red = high)
      - [ ] Hedging Status Table:
        - [ ] Columns: Pair | Total Exposure | Hedged Amount | Uncovered | Hedge Type | Rate | P&L
        - [ ] Rows: Each active pair with exposure ≥ threshold
      - [ ] Add Hedge button → Opens hedge dialog

- [ ] Create src/views/components/AddHedgeDialog.vue
  - [ ] Dialog form:
    - [ ] Base Currency: read-only (from context)
    - [ ] Foreign Currency: read-only
    - [ ] Hedge Type: radio (Forward | Option | Swap)
    - [ ] Amount: input (default = total exposure)
    - [ ] Rate: input (spot rate pre-filled)
    - [ ] Settlement Date: date picker
  - [ ] Buttons: [Create Hedge] [Cancel]
  - [ ] On Create: Call API, update exposure page

- [ ] Create src/views/pages/AccountBalances.vue (part of Cash Position)
  - [ ] Table: Bank Account | Balance | Currency | Equivalent EUR | Last Sync
  - [ ] Footer: Consolidated total in EUR + list of all currencies held
  - [ ] Refresh button: Trigger bank sync on demand
  - [ ] Sync status indicator: "Last synced 5 minutes ago"

**Implementation:**

- [ ] Create src/store/modules/fxExposure.js (Pinia store)
  - [ ] State: { exposures, accountBalances, hedges, loading, error }
  - [ ] Actions:
    - [ ] loadExposures(baseCurrency, foreignCurrency)
    - [ ] addHedge(baseCurrency, foreignCurrency, hedgeData)
    - [ ] loadAccountBalances()
  - [ ] Getters: exposureByPair, totalExposureEur, hedgeCoverage%, etc.

- [ ] Heatmap component: Use CnChartWidget with custom configuration for heatmap
  - [ ] Rows = currency pairs
  - [ ] Columns = maturity buckets
  - [ ] Cell color = risk level (green <25K, yellow 25-50K, red >50K)

- [ ] Add i18n strings: fx_exposure, hedged, uncovered, add_hedge, etc.

- [ ] Create tests/Unit/views/FXExposure.test.js
  - [ ] Test page loads with mock exposures
  - [ ] Test filter by currency pair
  - [ ] Test hedge dialog opens/closes
  - [ ] Test add hedge calls API

- [ ] Sign off: Exposure heatmap displays, hedge creation works, account balances show consolidated view

**Acceptance:** FX exposure page loads, heatmap displays exposure by maturity, hedge creation functional, account balances consolidated.

---

### Task T3.4: Dashboard Widget Implementation
**Effort:** 14 hours | **Dependencies:** T2.1, T2.2, T2.3  
**Status:** ⬜ Pending

Build treasury dashboard with KPI cards and task list widget.

**Components:**

- [ ] Create src/views/pages/TreasuryDashboard.vue (main dashboard)
  - [ ] Page structure (using CnDashboardPage + GridStack):
    - [ ] Top row: 4 KPI cards (CnStatsBlock)
      - [ ] Card 1: Consolidated Cash Balance (EUR)
      - [ ] Card 2: AR Aging (days overdue)
      - [ ] Card 3: AP Aging (days until due)
      - [ ] Card 4: 30-Day Liquidity Forecast (balance projection)
    - [ ] Middle row: Charts section (CnChartWidget)
      - [ ] Cash Flow Funnel: Inflows vs. Outflows by bucket (pie or funnel chart)
      - [ ] FX Exposure: Top 5 currency pairs (bar chart)
    - [ ] Bottom row: "My Tasks" widget (CnTableWidget)
      - [ ] Show top 5 high-priority tasks from task inbox
      - [ ] Columns: Title, Amount, Due Date, Priority
      - [ ] Link to full task inbox

- [ ] KPI Cards styling:
  - [ ] Use NL Design System tokens for colors (primary, success, warning, danger)
  - [ ] Display metric + trend indicator (↑↓)
  - [ ] Show last updated timestamp
  - [ ] Click to drill into detail page (e.g., click balance → accounts page)

**Implementation:**

- [ ] Create src/views/widgets/TreasuryStatsBlock.vue (custom widget wrapper)
  - [ ] Props: { metric, value, unit, trend, updated }
  - [ ] Uses CnStatsBlock internally
  - [ ] Responsive layout

- [ ] Create src/views/widgets/CashFlowFunnelWidget.vue
  - [ ] Fetches forecast data
  - [ ] Renders funnel chart (inflows → net cash → ending balance)

- [ ] Create src/views/widgets/MyTasksWidget.vue
  - [ ] Fetches top 5 open tasks
  - [ ] Table display
  - [ ] Click row → navigate to task detail

- [ ] Update src/App.vue with dashboard layout:
  - [ ] Check if OpenRegister available (from settings)
  - [ ] If available: load dashboard
  - [ ] If not: show empty state with setup instructions

- [ ] Add dashboard configuration (user can customize widget positions):
  - [ ] Store layout in user preferences via UserPreferenceService
  - [ ] Allow drag-drop reordering (GridStack built into CnDashboardPage)

- [ ] Create tests/Unit/views/TreasuryDashboard.test.js
  - [ ] Test page loads
  - [ ] Test KPI cards render with data
  - [ ] Test charts render
  - [ ] Test task widget shows top tasks

- [ ] Sign off: Dashboard renders, KPIs update, widgets interactive

**Acceptance:** Dashboard page loads with all widgets, KPI cards show correct data, charts render, task widget functional.

---

## Phase 4: Integration & Testing (Weeks 10-12)

### Task T4.1: Bank API Integration (TBD Provider)
**Effort:** 20 hours | **Dependencies:** T1.7, T2.5  
**Status:** ⬜ Pending

Integrate with bank API provider (Plaid, TrueLayer, Finery, etc.) — **TBD: provider selection**.

**Prerequisites:**
- [ ] Product decision: Which provider? (Cost, coverage, rate limits, features)
- [ ] Accounts: API keys, sandbox credentials obtained
- [ ] Documentation: Read provider SDK/API docs

**Implementation:**

- [ ] Create src/Services/BankIntegration/ directory
- [ ] Create src/Services/BankIntegration/BankApiClient.php (adapter pattern)
  - [ ] Abstract interface: BankApiClientInterface
  - [ ] Methods:
    - [ ] authenticate(): Get access token
    - [ ] getAccounts(): List connected accounts
    - [ ] getTransactions(accountId, since): Fetch transaction list
    - [ ] getBalance(accountId): Current balance
  - [ ] Error handling: Retry logic, rate limiting, timeouts
- [ ] Create src/Services/BankIntegration/PlaidClient.php (example: if Plaid selected)
  - [ ] Extends BankApiClient
  - [ ] Implements Plaid SDK calls:
    - [ ] `$client->itemPublicTokenExchange()`
    - [ ] `$client->accountsGet()`
    - [ ] `$client->transactionsGet()`
  - [ ] Map Plaid responses to Transaction objects
- [ ] Integrate with BankSyncService:
  - [ ] Inject BankApiClient
  - [ ] Call getTransactions() in sync() method
  - [ ] Handle pagination, errors, rate limits
- [ ] Add configuration:
  - [ ] Store API keys in IAppConfig (marked sensitive: true)
  - [ ] Add settings page for bank connection setup:
    - [ ] "Connect Bank" button → OAuth flow (or API key entry)
    - [ ] List connected accounts
    - [ ] Toggle sync on/off per account
- [ ] Create tests/Unit/Services/BankIntegration/PlaidClientTest.php
  - [ ] Mock Plaid API responses
  - [ ] Test transaction import
  - [ ] Test error handling
- [ ] Sign off: Bank sync working with real API (sandbox), transactions imported

**Acceptance:** Bank connection established, transactions importing, sync job working, ≥3 unit tests pass.

---

### Task T4.2: FX Rate Feed Integration (TBD Provider)
**Effort:** 10 hours | **Dependencies:** T1.6, T2.3  
**Status:** ⬜ Pending

Integrate FX rate updates (ECB, Xe.com, OpenExchangeRates, etc.) — **TBD: provider selection**.

**Prerequisites:**
- [ ] Product decision: Which provider? (Cost, coverage, update frequency)
- [ ] Account: API key obtained (if applicable)

**Implementation:**

- [ ] Create src/Services/FxRate/FxRateProvider.php (adapter pattern)
  - [ ] Interface: FxRateProviderInterface
  - [ ] Methods:
    - [ ] getRate(baseCurrency, targetCurrency): decimal
    - [ ] getRates(baseCurrency, targetCurrencies[]): array[targetCurrency => rate]
    - [ ] getTimestamp(): datetime (rate validity)
- [ ] Create src/Services/FxRate/EcbProvider.php (example: if ECB selected)
  - [ ] Calls ECB API daily (free, daily rates)
  - [ ] Caches rates in file or Redis (24h TTL)
  - [ ] Fallback to previous day if feed unavailable
- [ ] Integrate with FXExposureCalculationService:
  - [ ] Inject FxRateProvider
  - [ ] Call getRate() when calculating unrealizedGain
- [ ] Add background job: FxRateUpdateJob
  - [ ] Runs daily at 09:00 (after ECB publishes)
  - [ ] Fetches all major pairs (EUR/USD, EUR/GBP, EUR/CHF, etc.)
  - [ ] Caches rates
  - [ ] Logs result
- [ ] Add admin settings:
  - [ ] Provider selector (ECB, Xe.com, OpenExchangeRates, manual)
  - [ ] API key field (if applicable, encrypted)
  - [ ] Manual rate override per date
- [ ] Create tests/Unit/Services/FxRate/EcbProviderTest.php
  - [ ] Mock ECB API response
  - [ ] Test rate retrieval
  - [ ] Test caching
- [ ] Sign off: FX rates updating daily, rates used in exposure calculations

**Acceptance:** FX rate provider integrated, rates updating daily, caching working, ≥2 unit tests pass.

---

### Task T4.3: Payment Gateway Integration (SEPA Starter)
**Effort:** 16 hours | **Dependencies:** T2.5  
**Status:** ⬜ Pending

Integrate SEPA payment processing (start with SEPA; ACH/Wise TBD).

**Scope:** SEPA Credit Transfer (SCT) for intra-EU payments.

**Implementation:**

- [ ] Create src/Services/Payment/PaymentGateway.php (adapter pattern)
  - [ ] Interface: PaymentGatewayInterface
  - [ ] Methods:
    - [ ] submitBatch(PaymentBatch): SubmissionResponse
    - [ ] getStatus(batchReference): BatchStatus
    - [ ] getDetails(paymentReference): PaymentDetails

- [ ] Create src/Services/Payment/SepaCreditTransferService.php
  - [ ] Generates pain.001 SEPA XML using a library (e.g., Omnipay\Omnipay, `mdomke/sepa-xml`)
  - [ ] Methods:
    - [ ] generateXml(PaymentBatch): string (XML content)
    - [ ] validateXml(string $xml): bool
  - [ ] Includes:
    - [ ] Message/Payment group structure
    - [ ] Debtor info (company details from IAppConfig)
    - [ ] Creditor info (payee details)
    - [ ] Amount, reference, execution date

- [ ] Integration options (TBD: choose one):
  - [ ] Option A: Direct bank API (if bank provides SEPA endpoint)
    - [ ] Post XML directly to bank
    - [ ] Get confirmation + batch reference
  - [ ] Option B: Payment processor (Stripe, Wise, etc.)
    - [ ] Upload XML to processor
    - [ ] Processor validates and submits
  - [ ] Option C: File download (user uploads manually to online banking)
    - [ ] Generate XML file, offer download
    - [ ] User logs into bank portal and uploads
    - [ ] Track manually via bank statement reconciliation

- [ ] Implement option C (file download) for MVP:
  - [ ] generateBankFile() returns XML file
  - [ ] User downloads, uploads manually to bank portal
  - [ ] BankSyncService matches posted transactions back to batch

- [ ] Add error handling:
  - [ ] Validation: All required fields present? Amounts valid? Dates in future?
  - [ ] Generation: XML well-formed?
  - [ ] Submission: HTTP errors, timeouts, rate limits

- [ ] Create tests/Unit/Services/Payment/SepaCreditTransferTest.php
  - [ ] Test XML generation (valid pain.001 schema)
  - [ ] Test with sample batch of 5 payments
  - [ ] Test validation

- [ ] Sign off: SEPA XML generation working, file downloadable, validation passes

**Acceptance:** PaymentBatching creates valid SEPA pain.001 XML files, validated against schema, ≥2 unit tests pass.

---

### Task T4.4: End-to-End Integration Test
**Effort:** 20 hours | **Dependencies:** T4.1, T4.2, T4.3, T3.1, T3.2, T3.3, T3.4  
**Status:** ⬜ Pending

Test complete workflow: task creation → forecasting → payment batching → submission.

**Test Scenarios:**

- [ ] **Scenario 1: AP Task Creation & Batch Payment**
  - [ ] Create VendorBill (via API or UI)
  - [ ] Verify TreasuryTask auto-created
  - [ ] Open task inbox, see task
  - [ ] Select 3 AP tasks, click "Batch Payment"
  - [ ] Configure batch: 3 invoices, EUR amount, execution date
  - [ ] Generate SEPA file
  - [ ] Approve batch (via approval workflow)
  - [ ] Submit to bank (mock endpoint)
  - [ ] Verify Payment objects created with status 'initiated'
  - [ ] Verify audit trail complete

- [ ] **Scenario 2: Cash Flow Forecast with Defaults**
  - [ ] Navigate to forecast page
  - [ ] Select date range (next 90 days)
  - [ ] Click "Generate"
  - [ ] Verify 3 scenarios displayed
  - [ ] Verify cumulative balance projection
  - [ ] Verify assumptions visible
  - [ ] Export to CSV
  - [ ] Verify file downloadable and readable

- [ ] **Scenario 3: FX Exposure Management**
  - [ ] Create AR invoice in USD (50,000)
  - [ ] Create AP bill in GBP (20,000)
  - [ ] Create scheduled payment in USD (900)
  - [ ] Navigate to FX Exposure page
  - [ ] Verify EUR/USD exposure calculated (50,900 USD)
  - [ ] Verify EUR/GBP exposure calculated (20,000 GBP)
  - [ ] Add hedge: 50,000 USD forward at 0.92
  - [ ] Verify coverage % updated
  - [ ] Verify P&L calculated

- [ ] **Scenario 4: Bank Sync & Reconciliation**
  - [ ] Trigger bank sync (mock API)
  - [ ] Verify 5 transactions imported
  - [ ] Verify 4 auto-matched to JournalEntries
  - [ ] Verify 1 marked as unmatched
  - [ ] Navigate to reconciliation UI
  - [ ] Manually match unmatched transaction
  - [ ] Verify reconciliation complete (0 unmatched)

**Implementation:**

- [ ] Create tests/Integration/TreasuryWorkflowTest.php
  - [ ] PHPUnit test class
  - [ ] setUp(): Create test user, set permissions, load fixtures
  - [ ] testApBatchPaymentWorkflow(): Scenario 1 above
  - [ ] testCashFlowForecastWorkflow(): Scenario 2 above
  - [ ] testFxExposureWorkflow(): Scenario 3 above
  - [ ] testBankSyncReconciliation(): Scenario 4 above
  - [ ] tearDown(): Clean up test data

- [ ] Create tests/Browser/TreasuryEndToEndTest.php (Playwright/Selenium)
  - [ ] Browser-based end-to-end test
  - [ ] Navigate pages, click buttons, fill forms
  - [ ] Verify UI updates (tasks appear, charts render, etc.)
  - [ ] Focus on user journey, not implementation details

- [ ] Sign off: All scenarios pass, API + UI integrated, no data loss

**Acceptance:** 4 complete workflows tested (backend + frontend), no data loss, audit trail correct throughout.

---

### Task T4.5: Performance & Load Testing
**Effort:** 12 hours | **Dependencies:** T4.4  
**Status:** ⬜ Pending

Verify performance targets and identify bottlenecks.

**Test Cases:**

- [ ] **Task Inbox Load Test:**
  - [ ] Create 10,000 TreasuryTask records
  - [ ] Query GET /treasury/tasks?page=1&limit=50
  - [ ] Measure response time: <2 seconds ✓
  - [ ] Verify pagination works

- [ ] **Forecast Generation Load Test:**
  - [ ] Generate forecast with 1,000+ open invoices/bills
  - [ ] Measure generation time: <10 seconds ✓
  - [ ] Verify caching prevents repeated calculations

- [ ] **Batch Payment Processing Load Test:**
  - [ ] Create 100-item batch
  - [ ] Generate SEPA XML: <5 seconds ✓
  - [ ] Submit batch (mock): <2 seconds ✓

- [ ] **Concurrent User Test:**
  - [ ] Simulate 10 concurrent users accessing dashboard
  - [ ] Verify no race conditions, no data corruption
  - [ ] Verify response times remain acceptable

**Implementation:**

- [ ] Use Apache JMeter or similar load testing tool
- [ ] Create test plans for above scenarios
- [ ] Run tests, measure response times
- [ ] Identify slow queries, optimize if needed:
  - [ ] Add database indexes on frequently filtered columns
  - [ ] Implement caching for expensive calculations
  - [ ] Optimize N+1 queries in services
- [ ] Document results in performance report
- [ ] Sign off: All targets met

**Acceptance:** Task inbox <2s, forecast <10s, batch submission <2s under load; concurrent users handle 10 without degradation.

---

### Task T4.6: Security & Compliance Audit
**Effort:** 16 hours | **Dependencies:** T4.4  
**Status:** ⬜ Pending

Verify security, auth, and compliance requirements.

**Security Checks:**

- [ ] **Authorization (REQ-SEC-001, REQ-SEC-002):**
  - [ ] Verify user without Treasury role cannot access /treasury/tasks → 403
  - [ ] Verify CFO can approve payments, Accountant cannot
  - [ ] Verify approval workflow enforced (cannot bypass)
  - [ ] Test role-based access per feature (AP, AR, FX, etc.)

- [ ] **Data Privacy (REQ-NFR-004, ADR-005):**
  - [ ] Scan logs for PII (account numbers, names, email)
  - [ ] Verify account numbers masked in logs (last 4 digits visible)
  - [ ] Verify no stack traces in API responses
  - [ ] Verify no sensitive data in error messages
  - [ ] Test GDPR right-to-delete: User requests deletion → data removed from logs + objects

- [ ] **Audit Trail (REQ-NFR-002):**
  - [ ] Verify every payment has complete before/after snapshots
  - [ ] Verify timestamps accurate
  - [ ] Verify user attribution correct
  - [ ] Verify cannot edit/delete audit trail
  - [ ] Test export audit trail for compliance reporting

- [ ] **Approval Workflows (REQ-SEC-002):**
  - [ ] Verify payment cannot post without approval
  - [ ] Verify approver cannot be bypassed
  - [ ] Verify rejection returns to draft state
  - [ ] Test escalation timeouts (if configured)

- [ ] **Input Validation:**
  - [ ] SQL injection: Test SQL-like strings in text inputs → sanitized
  - [ ] XSS: Test script tags in notes/comments → escaped
  - [ ] CSRF: Verify POST/PUT/DELETE require CSRF token
  - [ ] Date injection: Test invalid dates → rejected

- [ ] **API Security (ADR-002):**
  - [ ] Verify API errors return appropriate HTTP status (400, 403, 500)
  - [ ] Verify no stack traces in responses
  - [ ] Verify authentication required (no public endpoints)
  - [ ] Verify pagination prevents excessive data retrieval (_limit enforced)

**Implementation:**

- [ ] Create tests/Security/AuthorizationTest.php
  - [ ] Test role checks for each API endpoint
  - [ ] Test field-level permissions (if applicable)
  - [ ] Test multi-tenancy isolation (if applicable)

- [ ] Create tests/Security/InputValidationTest.php
  - [ ] Test sanitization of text inputs
  - [ ] Test date format validation
  - [ ] Test amount validation (no negative, reasonable limits)

- [ ] Run OWASP ZAP or similar scanner on running app
  - [ ] Identify vulnerabilities
  - [ ] Fix or document as acceptable risk

- [ ] Manual audit:
  - [ ] Review code for crypto (no hardcoded passwords)
  - [ ] Review config for secrets (API keys encrypted)
  - [ ] Review SQL queries for injection vulnerability
  - [ ] Review file uploads (if applicable)

- [ ] Sign off: Security checklist complete, vulnerabilities addressed

**Acceptance:** Authorization enforced, no PII in logs, audit trail immutable, input validation working, ≥5 security tests pass.

---

### Task T4.7: Documentation & Help Content
**Effort:** 16 hours | **Dependencies:** All previous tasks  
**Status:** ⬜ Pending

Create user documentation and admin guides.

**Documentation Sections:**

- [ ] **User Guide** (docs/USER_GUIDE.md):
  - [ ] Treasury Dashboard overview
  - [ ] Task Inbox: How to filter, sort, mark complete
  - [ ] Cash Flow Forecast: How to run, interpret scenarios, export
  - [ ] FX Exposure: How to view, add hedges
  - [ ] Payment Processing: How to create batch, approve, submit

- [ ] **Admin Guide** (docs/ADMIN_GUIDE.md):
  - [ ] Initial setup: Bank connection, FX rate feed, payment gateway
  - [ ] User roles & permissions: Create Treasury Manager, CFO, Accountant roles
  - [ ] Approval workflows: Configure chains, set thresholds
  - [ ] Settings: Configure assumptions (AR/AP rates), currencies, GL accounts
  - [ ] Troubleshooting: Bank sync failures, failed payments, reconciliation issues

- [ ] **API Reference** (docs/API.md):
  - [ ] Endpoint documentation: All REST endpoints with request/response examples
  - [ ] Authentication & authorization
  - [ ] Error codes & handling
  - [ ] Rate limits

- [ ] **Screenshots & Videos:**
  - [ ] Screenshots of each major page (Task Inbox, Forecast, FX, Dashboard)
  - [ ] Screen recording of common workflows (create batch payment, generate forecast)
  - [ ] Annotated with callouts for key features

**Implementation:**

- [ ] Create docs/ directory structure:
  - [ ] docs/USER_GUIDE.md
  - [ ] docs/ADMIN_GUIDE.md
  - [ ] docs/API.md
  - [ ] docs/TROUBLESHOOTING.md
  - [ ] docs/images/ (screenshots)
  - [ ] docs/videos/ (links or embedded)

- [ ] For each page/feature:
  - [ ] Write 2-3 paragraph description
  - [ ] Include screenshot
  - [ ] List key actions with step-by-step instructions
  - [ ] Document keyboard shortcuts (if any)
  - [ ] Provide example scenarios

- [ ] Add in-app help:
  - [ ] Context-sensitive help: "?" icon on each page
  - [ ] Tooltips on form fields
  - [ ] Empty state messages with helpful links

- [ ] Create glossary of terms (docs/GLOSSARY.md):
  - [ ] Treasury Task, Liquidity Forecast, FX Exposure, Hedge, SEPA, etc.
  - [ ] Links to full documentation

- [ ] Create CHANGELOG.md documenting this release:
  - [ ] Features: List all 49 features + implementation status
  - [ ] Improvements: Performance, security, UX
  - [ ] Known issues: Any limitations or TODOs
  - [ ] Migration: Steps for upgrading from previous version (if any)

- [ ] Sign off: Documentation complete, all pages covered, screenshots current

**Acceptance:** User guide covers all major features, admin guide covers setup, API documented, screenshots present, ≥500 words of documentation.

---

### Task T4.8: Release & Deployment Prep
**Effort:** 8 hours | **Dependencies:** All previous tasks  
**Status:** ⬜ Pending

Prepare app for release (version bump, migration, testing, release notes).

**Release Checklist:**

- [ ] **Version Bump:**
  - [ ] Update version in appinfo/info.xml: e.g., 1.0.0
  - [ ] Update CHANGELOG.md with release date
  - [ ] Create git tag: `git tag v1.0.0`

- [ ] **Database Migrations:**
  - [ ] Run all repair steps (register import, schema creation)
  - [ ] Test migration on fresh install
  - [ ] Test migration on upgrade from previous version
  - [ ] Verify seed data loads (3-5 objects per schema)

- [ ] **Testing Checklist:**
  - [ ] All unit tests pass: `composer check:strict`
  - [ ] All integration tests pass
  - [ ] Browser tests pass (UI interactions)
  - [ ] Load tests pass (performance targets)
  - [ ] Security audit passed
  - [ ] Manual smoke test: Key workflows work end-to-end

- [ ] **Build Verification:**
  - [ ] No console errors/warnings
  - [ ] No 404s on asset loading
  - [ ] JavaScript minified, CSS compiled
  - [ ] Translations complete (en, nl)
  - [ ] App installs cleanly on fresh Nextcloud

- [ ] **Release Notes:**
  - [ ] Write release summary (1 paragraph, highlights)
  - [ ] List major features
  - [ ] Document breaking changes (if any)
  - [ ] List known issues / limitations
  - [ ] Credit contributors

- [ ] **Communication:**
  - [ ] Post to Nextcloud App Store (if applicable)
  - [ ] Announce in forum/Discord
  - [ ] Create PR with release notes
  - [ ] Tag team for final sign-off

- [ ] Sign off: App ready for production, all tests passing, release notes published

**Acceptance:** Version bumped, migrations tested, all tests pass, release notes published, tagged in git.

---

## Deduplication Check

**Summary:**
- ✓ All entities reuse existing OpenRegister objects (BankAccount, CurrencyBalance, etc.)
- ✓ No new custom schemas required
- ✓ Services leverage ObjectService, RegisterService, AuthorizationService from OpenRegister
- ✓ Frontend components use @conduction/nextcloud-vue (CnIndexPage, CnDetailPage, CnDashboardPage, etc.)
- ✓ No duplication of existing functionality (search, import/export, audit, webhooks all provided by platform)

**Result:** Deduplication check complete — no overlap with OpenRegister or nextcloud-vue. Spec focuses on domain-specific business logic (forecasting, approvals, bank sync integration).

---

## Summary

**Total Effort:** ~380 hours (12 weeks for full-stack team of 2-3 engineers)

**Key Milestones:**
- Week 3: Backend infrastructure complete (services, register template, seed data)
- Week 6: API endpoints complete (task inbox, forecasting, FX, approval workflow)
- Week 9: Frontend pages complete (task inbox, forecast, FX, dashboard)
- Week 12: Integration, testing, documentation, release

**Success Criteria:**
✓ Treasury manager can view unified task inbox (AP/AR/scheduled items)
✓ Cash flow forecast generated for 90 days with 3 scenarios
✓ FX exposure tracked by currency pair with hedging capability
✓ Payment batches created, approved, and submitted to bank
✓ All workflows have complete audit trail
✓ Performance targets met (<2s task load, <10s forecast)
✓ Security & compliance verified (RBAC, PII protection, audit immutable)
✓ Documentation complete (user guide, admin guide, API reference)

