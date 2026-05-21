# Treasury & Cash Management — Implementation Tasks

**Change:** treasury-cash-management  
**Phase:** tasks  
**Created:** 2026-05-21

---

## Overview

Implementation organized into 7 sprints with clear dependencies. Total effort: 5-7 sprints for a 4-person team (1 backend architect, 2 PHP/API engineers, 1 frontend developer).

---

## Sprint 1: Data Model & Core Infrastructure

### Task 1.1: Create OpenRegister Schemas (Backend)
**Story:** ADR-001, REQ-001, REQ-002, REQ-003  
**Effort:** 3 days  

- [ ] Define CashAccount schema in `lib/Settings/treasury_register.json`
  - Properties: accountName, accountType, accountCode, bankName, accountNumber, currency, currentBalance, availableBalance, lastBalanceUpdate, riskLevel, isActive, isPrimaryAccount
  - Relations: organization (many-to-one)
  - Validations: accountNumber IBAN checksum, amount >= 0, currency in ISO 4217 list

- [ ] Define CurrencyBalance schema
  - Properties: balanceId, currency, balance, previousBalance, lastUpdated
  - Relations: cashAccount (many-to-one)
  - Seed data: 3 examples (EUR/USD/GBP/JPY)

- [ ] Define FXExposure schema
  - Properties: baseCurrency, foreignCurrency, exposureAmount, currentExchangeRate, valuationDate, unrealizedGainLoss, riskLevel
  - Relations: cashAccount, organization (many-to-one)
  - Seed data: 3 examples (USD/EUR, GBP/EUR, JPY/EUR)

- [ ] Define LiquidityForecast schema
  - Properties: period, forecastDate, projectionDays, projectedInflow, projectedOutflow, netProjection, currency, confidence, methodology
  - Relations: cashAccount, organization (many-to-one)
  - Seed data: 3 examples (daily, weekly, monthly forecasts)

- [ ] Define PaymentBatch schema
  - Properties: batchNumber, description, totalAmount, totalPayments, status, approvalStatus, approvedBy, approvalDate, scheduledDate, executedDate, createdDate, exportFormat, exportedFile
  - Relations: organization (many-to-one), payments (one-to-many)
  - Seed data: 3 examples (completed, pending, draft batches)

- [ ] Define RequestForQuotation schema
  - Properties: rfqNumber, title, description, estimatedValue, deadline, round, status, lockboxEnabled, lockboxOpensAt, createdDate, publishedDate, awardedSupplier
  - Relations: organization (many-to-one), suppliers (many-to-many), offers (one-to-many)
  - Seed data: 3 examples (published, closed, awarded RFQs)

- [ ] Define ScheduledPayment schema
  - Properties: paymentReference, description, amount, currency, payeeId, payeeName, payeeAccountNumber, scheduledDate, frequency, recurringStartDate, recurringEndDate, occurrenceCount, status, lastExecutionDate, nextExecutionDate, createdDate
  - Relations: payee (many-to-one), cashAccount (many-to-one), payments (one-to-many)
  - Seed data: 3 examples (rent, insurance, intercompany transfer)

- [ ] Define TreasuryTask schema
  - Properties: taskId, taskType, title, amount, currency, dueDate, counterpartyName, counterpartyId, description, status, priority, documentRef, createdDate, completedDate
  - Relations: cashAccount (many-to-one), organization (many-to-one)
  - Seed data: 3 examples (AP, AR, CapEx tasks)

- [ ] Run migration via `IRepairStep` to initialize all schemas in OpenRegister

**Tests:**
- Unit: Schema definitions parse correctly, relations are valid
- Integration: Can create/read/update/delete objects via ObjectService for each entity

---

### Task 1.2: Implement Bank Statement Import Service (Backend)
**Story:** REQ-002  
**Effort:** 4 days  

- [ ] Create `BankStatementImportService` class
  - Public method: `importCAMT053(UploadedFile $file): BankStatementImportResult`
  - Parse XML envelope per ISO 20022 CAMT.053 spec
  - Extract: sourceBank, sourceAccount, statementPeriod, transactions
  - Validate: well-formed XML, required fields present

- [ ] Implement transaction parser
  - Extract debit/credit transactions
  - Map to internal Payment entity properties
  - Handle SEPA remittance information
  - Support multiple transaction details per entry (amount, date, counterparty)

- [ ] Implement duplicate detection
  - Query existing transactions for same (account, amount, date, counterparty)
  - Log warning if duplicate found
  - Skip duplicate, continue with next transaction

- [ ] Implement balance update logic
  - Calculate opening balance from previous statement's closing
  - Apply all transactions (debit-credit)
  - Validate calculated balance matches statement closing balance
  - Update CashAccount.currentBalance atomically

- [ ] Implement import summary report
  - Count total transactions processed
  - Count duplicates skipped
  - Count validation errors
  - Generate summary: "[N] transactions imported, [M] duplicates skipped, [K] errors"

- [ ] Create API controller: `POST /api/treasury/bank-statements/import`
  - Accept multipart file upload
  - Validate file type (application/xml)
  - Call BankStatementImportService
  - Return: importResult (success/error), transactionCount, newBalance

**Tests:**
- Unit: XML parsing, transaction extraction, duplicate detection
- Integration: Upload real CAMT.053 test file, verify CashAccount balance updated

---

### Task 1.3: Set Up Frontend Store and Components (Frontend)
**Story:** ADR-004, All entities  
**Effort:** 2 days  

- [ ] Create Pinia store in `src/store/modules/treasuryStore.ts`
  - Register object stores for all 8 entities (CashAccount, CurrencyBalance, FXExposure, etc.)
  - Use `createObjectStore` with plugins: auditTrails, files, relations, search
  - Expose methods: `getCashAccounts()`, `savePaymentBatch()`, `deleteTask()`, etc.
  - Handle loading state and error notifications

- [ ] Create main Treasury dashboard component `src/pages/Treasury/Dashboard.vue`
  - Displays 4 KPI cards (Total Cash, Open Payments, Forecast, Compliance)
  - Uses `CnStatsBlock` for each KPI
  - Fetches data from treasuryStore on mount
  - Implements real-time updates via store subscriptions

- [ ] Create Cash Accounts index page `src/pages/Treasury/Accounts/Index.vue`
  - Uses `CnIndexPage` + `useListView` composable
  - Displays table: Account Name, Type, Balance, Currency, Risk Level, Last Update
  - Row click → Detail view
  - Actions: New Account, Import Statement, Refresh Balance
  - Sort by: Account Name, Balance (desc), Risk Level

- [ ] Create Cash Account detail page `src/pages/Treasury/Accounts/Detail.vue`
  - Edit mode: form with schema-driven fields (CnFormDialog)
  - View mode: detail cards showing account info, balance history, related items
  - Tabs: Details, FX Exposures, Forecasts, Scheduled Payments, Audit Trail
  - Actions: Edit, Delete, Import Statement (if BankAccount type)

- [ ] Create router configuration in `src/router/index.ts`
  - Routes: `/` (Dashboard), `/accounts` (list), `/accounts/:id` (detail), `/settings`
  - Lazy-load components
  - Inject sidebarState

- [ ] Create i18n translations for Treasury module
  - English: `public/l10n/en/treasury.json`
  - Dutch: `public/l10n/nl/treasury.json`
  - Keys: module title, page titles, form labels, validation messages

**Tests:**
- Unit: Store mutations, computed properties
- Visual: Dashboard renders KPIs, Accounts page shows table, Detail page shows form

---

## Sprint 2: Cash Account Management & Bank Statement Import

### Task 2.1: Implement Cash Account CRUD (Full Stack)
**Story:** REQ-001  
**Effort:** 3 days (1.5 backend + 1.5 frontend)  

**Backend:**
- [ ] Create `CashAccountController` with methods:
  - `POST /api/treasury/cash-accounts` → `CashAccountService.create()`
  - `GET /api/treasury/cash-accounts` → `CashAccountService.findAll()` with pagination, filtering
  - `GET /api/treasury/cash-accounts/{id}` → `CashAccountService.getById()`
  - `PUT /api/treasury/cash-accounts/{id}` → `CashAccountService.update()`
  - `DELETE /api/treasury/cash-accounts/{id}` → `CashAccountService.softDelete()`
  - All endpoints require `@spec openspec/changes/treasury-cash-management/tasks.md#task-2.1` PHPDoc

- [ ] Create `CashAccountService` with business logic:
  - Validate accountCode uniqueness per organization
  - Validate IBAN checksum (mod-97) for bank accounts
  - Calculate availableBalance = currentBalance - holds (future feature)
  - Audit logging for create/update/delete

- [ ] Create `CashAccountMapper` for DB CRUD
  - Map between OpenRegister objects and database rows
  - NO business logic in mapper

**Frontend:**
- [ ] Create CashAccount form component with validation
  - Fields: accountName, type, bankName, accountNumber, GL code, currency, riskLevel
  - Validation: accountName required, accountNumber IBAN format, currency dropdown, riskLevel enum
  - Error display with inline messages

- [ ] Create "New Cash Account" workflow
  - Button in dashboard → Dialog with form
  - Save calls treasuryStore.saveCashAccount()
  - Success: navigate to detail view
  - Error: show toast message

- [ ] Create "Edit Cash Account" workflow
  - Edit button in detail view
  - Populate form with current values
  - Disable: type, accountNumber (immutable after creation)
  - Save calls treasuryStore.updateCashAccount()

**Tests:**
- Unit: Validator (IBAN, GL code), Service (create, update)
- Integration: Create account via API, verify in OpenRegister
- E2E (Playwright): Create form → submit → verify in list

---

### Task 2.2: Implement Bank Statement Import UI (Frontend)
**Story:** REQ-002  
**Effort:** 2 days  

- [ ] Create "Import Statement" dialog component `ImportStatementDialog.vue`
  - File upload input (drag-drop + click)
  - File preview: bank name, account, statement period, transaction count
  - Validation: file type = .xml, file size <= 50MB
  - Buttons: Cancel, Import

- [ ] Create import result notification
  - Toast: "[N] transactions imported successfully"
  - Link to view transaction list
  - Warnings if duplicates skipped

- [ ] Add import action to detail view
  - Button "Import Statement" visible only for BankAccount type
  - Opens ImportStatementDialog
  - On success: refresh balance display, show transaction list

- [ ] Create transaction list component
  - Display imported transactions in a table
  - Columns: Date, Amount, Counterparty, Reference, Status
  - Color-code: green (inflow), red (outflow)
  - Show "Duplicates skipped: [N]" if any

**Tests:**
- Unit: Form validation
- E2E: Upload test CAMT.053 file, verify transactions appear

---

### Task 2.3: Implement Real-Time Cash Position Dashboard (Frontend)
**Story:** REQ-003, REQ-015, REQ-023  
**Effort:** 3 days  

- [ ] Update Treasury Dashboard with KPI cards
  - Card 1: TOTAL CASH
    - Value: sum of all accounts in EUR
    - Change: percentage vs. yesterday
    - Icon: money bag
    - Click → drill-down to accounts list

  - Card 2: OPEN PAYMENTS
    - Value: count of pending/approved payment batches
    - Amount: sum of open payment amounts
    - Icon: envelope (outgoing)
    - Click → drill-down to Payment Batches list

  - Card 3: FORECAST CLOSING BALANCE (13W)
    - Value: projected balance from LiquidityForecast
    - Confidence: "High" / "Medium" / "Low"
    - Icon: chart
    - Click → drill-down to Forecast detail

  - Card 4: COMPLIANCE STATUS
    - Status: Green (OK) / Yellow (Warning) / Red (Alert)
    - Message: e.g., "All limits met" or "Wet Fido: 95% utilized"
    - Icon: checkmark / warning
    - Click → drill-down to Compliance dashboard

- [ ] Implement real-time update mechanism
  - Subscribe to treasuryStore changes
  - Throttle updates to max 1/minute
  - Show "Updated 3 min ago" timestamp
  - Add refresh button for manual update

- [ ] Create account breakdown modal
  - Table: Account Name, Type, Balance, Currency, % of Total
  - Pie chart: Account composition by balance
  - 30-day trend chart: Daily balances, min/max/average
  - Currency breakdown: Show multi-currency summary

- [ ] Create Daily Cash Position page
  - Shows opening balances per account (as of 08:00)
  - Consolidated total
  - Change from previous day
  - Transaction feed for selected account

**Tests:**
- Unit: KPI calculations (sum, percentages, aggregations)
- Visual: Dashboard renders without errors, KPIs display correct values

---

## Sprint 3: Multi-Currency & FX Management

### Task 3.1: Implement Multi-Currency Balance Tracking (Full Stack)
**Story:** REQ-004  
**Effort:** 3 days  

**Backend:**
- [ ] Create `CurrencyBalanceService`
  - Method: `recordBalance(CashAccount $account, string $currency, float $amount): CurrencyBalance`
  - Stores balance record with timestamp
  - Retrieves previous balance for variance calculation
  - Called daily after bank statement import

- [ ] Create daily exchange rate update job
  - Fetch rates from ECB API (or fallback provider)
  - For each CurrencyBalance, create new record with updated rate
  - Calculate EUR valuation: balance × rate
  - Log update timestamp

- [ ] Implement `CurrencyBalanceController` with methods:
  - `GET /api/treasury/currency-balances` → by cash account or currency
  - `GET /api/treasury/currency-balances/{id}` → detail with trend

**Frontend:**
- [ ] Create Currency Balance view component
  - Table: Currency, Amount, Rate, EUR Value, Change %, Last Update
  - Columns sortable: by currency, amount, value
  - Row click → detail with trend chart

- [ ] Create currency balance detail page
  - Display: Foreign currency amount, exchange rate, EUR valuation
  - Trend chart (30-day): Balance in original currency + EUR line
  - Rate history: Date, source, official rate
  - Variance analysis: Month-over-month change

**Tests:**
- Unit: Balance recording, rate calculation
- Integration: Record balance, verify in database

---

### Task 3.2: Implement FX Exposure Tracking (Full Stack)
**Story:** REQ-005  
**Effort:** 4 days  

**Backend:**
- [ ] Create `FXExposureService`
  - Method: `calculateExposure(CashAccount $account): FXExposure[]`
  - For each CurrencyBalance with currency != baseCurrency
  - Calculate: exposureAmount, currentExchangeRate, valuationDate, baseCurrencyValue, unrealizedGainLoss
  - Determine riskLevel based on: volatility, concentration (% of total cash)

- [ ] Implement risk level calculation
  - High: > 15% of total cash OR volatility > 5% (30-day)
  - Medium: 5-15% of total cash OR volatility 2-5%
  - Low: < 5% of total cash AND volatility < 2%

- [ ] Create `FXExposureController` with methods:
  - `GET /api/treasury/fx-exposures` → all exposures summary
  - `GET /api/treasury/fx-exposures/{id}` → detail with P&L history
  - `GET /api/treasury/fx-exposures/risk-summary` → aggregated risk report

- [ ] Implement FX exposure dashboard job
  - Runs daily at 16:00 CET (market close)
  - Recalculates all exposures
  - Generates risk report (email to treasurer if High risk detected)

**Frontend:**
- [ ] Create FX Exposure summary page
  - Table: Foreign Currency, Exposure Amount, Rate, Base Value, Unrealized P&L, Risk Level
  - Color-code risk: Low (green), Medium (yellow), High (red)
  - Chart: Exposure composition pie chart
  - Summary KPI: Total exposure, Total unrealized P&L

- [ ] Create FX Exposure detail page
  - Display all exposure details
  - P&L history: 30-day trend chart
  - Rate history: Date, rate, source
  - Hedging recommendations (future feature placeholder)

- [ ] Create "What-If" scenario tool
  - Input: new exchange rate for a currency
  - Output: recalculated FX positions, P&L impact, risk level change
  - Example: "If EUR/USD moves to 1.15, total exposure would be EUR 225k (now EUR 217k)"

**Tests:**
- Unit: Risk level calculation, P&L computation
- Integration: Calculate exposure, verify against manual calculation

---

### Task 3.3: Implement Liquidity Forecasting (Full Stack)
**Story:** REQ-006  
**Effort:** 5 days  

**Backend:**
- [ ] Create `LiquidityForecastService`
  - Method: `generateForecast(CashAccount $account, int $days = 91): LiquidityForecast`
  - Collects 12+ months historical transaction data
  - Filters ScheduledPayments for forecast window
  - Calls Anthropic API with prompt caching for AI prediction

- [ ] Implement forecast prompt with caching
  - System prompt: "You are a liquidity forecasting AI. Given historical cash flow data and scheduled payments, predict daily cash balances for the next N days. Provide confidence bands (80% CI)."
  - Context: historical inflows/outflows per day of week/season, ScheduledPayments, special events (payroll, quarterly taxes)
  - Cache: historical data (static), model instructions (static)
  - Request: specific date range, account details (non-static)

- [ ] Implement response parsing
  - Extract: daily projected balance, confidence bands, seasonal notes, risk factors
  - Store in LiquidityForecast entity
  - Calculate: confidence level (High/Medium/Low based on variance)

- [ ] Create `ForecastController` with methods:
  - `POST /api/treasury/liquidity-forecasts/generate` → trigger forecast generation
  - `GET /api/treasury/liquidity-forecasts/{accountId}` → latest forecast
  - `GET /api/treasury/liquidity-forecasts/{accountId}/history` → forecast revisions

- [ ] Implement daily forecast refresh job
  - Runs at 07:00 CET
  - Regenerates forecasts for all active accounts
  - Sends alert to treasurer if lowest balance drops below threshold

**Frontend:**
- [ ] Create Liquidity Forecast page
  - Display forecast summary: Period, confidence, lowest balance, risk flags
  - Line chart: Projected daily balance (green) + confidence band (light green shading)
  - Area chart overlay: Scheduled payments (orange dots), inflows/outflows (blue/red)
  - Table: Daily projections (Date, Opening, Inflows, Outflows, Closing, Confidence)

- [ ] Implement forecast detail view
  - Toggle: Daily / Weekly / Monthly view
  - Zoom: Expand/collapse specific date ranges
  - Drill-down: Click a day → see contributing factors (scheduled payments, expected collections)
  - Flags: Highlight days when balance falls below threshold

- [ ] Create forecast comparison: Forecast vs. Actuals
  - Show historical forecast from start of month vs. actual balances
  - Calculate accuracy: % variance
  - Adjust confidence for next forecast based on accuracy

- [ ] Implement forecast refresh trigger
  - "Refresh Forecast" button on page
  - Calls service, shows loading state, updates page on completion
  - Auto-refresh: every 24 hours

**Tests:**
- Unit: Prompt construction, response parsing
- Integration: Call Anthropic API with test data, verify forecast structure

---

## Sprint 4: Payment Scheduling & Batching

### Task 4.1: Implement Scheduled Payments (Full Stack)
**Story:** REQ-007  
**Effort:** 3 days  

**Backend:**
- [ ] Create `ScheduledPaymentService`
  - Method: `createScheduledPayment(array $data): ScheduledPayment`
    - Create single ScheduledPayment if frequency = "once"
    - Create multiple ScheduledPayment records (one per occurrence) if frequency = "recurring"
    - Validate: scheduledDate in future, amount > 0, payeeAccountNumber IBAN format

  - Method: `executeScheduledPayments(DateTime $date): ExecutionResult`
    - Filter: scheduledDate = $date AND status = "pending" or "approved"
    - Check: CashAccount.availableBalance >= payment.amount
    - If sufficient: move to "executing" → "executed"
    - If insufficient: set to "paused", notify treasurer

  - Method: `updateNextExecution(ScheduledPayment $payment): void`
    - Calculate next occurrence based on frequency
    - Update nextExecutionDate

- [ ] Create `ScheduledPaymentController` with methods:
  - `POST /api/treasury/scheduled-payments` → create
  - `GET /api/treasury/scheduled-payments` → list (upcoming, recurring summary)
  - `GET /api/treasury/scheduled-payments/{id}` → detail with all occurrences
  - `PUT /api/treasury/scheduled-payments/{id}` → update
  - `DELETE /api/treasury/scheduled-payments/{id}` → cancel

- [ ] Implement background job: ScheduledPaymentExecutionJob
  - Runs daily at 06:00 CET
  - Calls executeScheduledPayments(today)
  - Logs execution result
  - Sends notification to treasurer if any payments failed

**Frontend:**
- [ ] Create ScheduledPayment creation dialog
  - Triggered from invoice detail or standalone
  - Form fields: Payee, Amount, Currency, Scheduled Date, Frequency, Recurring Until
  - Validation: date in future, amount > 0
  - Preview: If recurring, show "12 payments scheduled, total EUR 60,000"
  - Buttons: Cancel, Schedule

- [ ] Create ScheduledPayment list page
  - Table: Payee, Amount, Scheduled Date, Frequency, Status, Next Execution
  - Filters: Status (upcoming, recurring, failed), Frequency (one-off, monthly, etc.)
  - Actions: Edit, Delete, View Detail

- [ ] Create ScheduledPayment detail page
  - Display: Full payment details, recurring schedule, execution history
  - If recurring: Table of all occurrences (date, status, execution date)
  - Edit button: Allow change of amount, date (future occurrences only)
  - Cancel button: Stop recurring payment (prompts for date)

**Tests:**
- Unit: Validation, recurring schedule generation
- Integration: Create recurring payment, verify 12 instances created in database

---

### Task 4.2: Implement Payment Batching & Approval (Full Stack)
**Story:** REQ-008  
**Effort:** 4 days  

**Backend:**
- [ ] Create `PaymentBatchService`
  - Method: `createBatch(array $paymentIds): PaymentBatch`
    - Sum amounts, generate batch number
    - Create PaymentBatch with status = "draft"

  - Method: `submitForApproval(PaymentBatch $batch): void`
    - Update status = "pending"
    - Lock batch (no edits allowed)
    - Determine approver based on amount:
      - < EUR 50k: Financial Controller
      - >= EUR 50k: Financial Controller + CFO (dual approval)
    - Send approval request emails

  - Method: `approvePaymentBatch(PaymentBatch $batch, User $approver): void`
    - Record approval (user, timestamp)
    - If dual approval required: send to second approver
    - If all approvals received: move to "approved"

  - Method: `executePaymentBatch(PaymentBatch $batch, DateTime $executionDate): void`
    - Validate all payments have sufficient balance
    - Group by bank/currency
    - Generate SEPA files (one per bank/currency combo)
    - Update batch status = "processing"
    - Log file generation

  - Method: `completeBatchExecution(PaymentBatch $batch, array $results): void`
    - Mark batch as "completed" or "failed"
    - Record individual payment statuses
    - Notify approvers of completion

- [ ] Create `PaymentBatchController` with methods:
  - `POST /api/treasury/payment-batches` → create batch
  - `POST /api/treasury/payment-batches/{id}/submit-approval` → submit for approval
  - `POST /api/treasury/payment-batches/{id}/approve` → approve batch
  - `POST /api/treasury/payment-batches/{id}/schedule` → schedule for execution
  - `GET /api/treasury/payment-batches` → list with filtering
  - `GET /api/treasury/payment-batches/{id}` → detail
  - `POST /api/treasury/payment-batches/{id}/execute` → execute batch (admin/scheduler only)

- [ ] Create `ApprovalChainService` (custom or use OpenRegister if available)
  - Track approvals per batch
  - Enforce dual sign-off rules
  - Send approval request/reminder emails

**Frontend:**
- [ ] Create PaymentBatch creation workflow
  - Step 1: Select payments (multi-select from open invoices list)
  - Summary: Total amount, payment count
  - Step 2: Review batch details
  - Step 3: Confirm → create batch

- [ ] Create PaymentBatch list page
  - Table: Batch #, Payments, Total Amount, Status, Approval Status, Execution Date
  - Filters: Status, approval status, date range
  - Color-code: draft (gray), pending approval (yellow), approved (green), completed (blue)
  - Actions: View, Submit for Approval, Schedule (if approved)

- [ ] Create PaymentBatch detail page
  - Header: Batch number, status, total amount, creation date
  - Approval section: Show approvers and approval status
  - If awaiting approval: Show message "Awaiting approval from John Doe (Controller)"
  - If approved: Show schedule execution button
  - Payments table: List of individual payments with status
  - Execution history: If executed, show confirmation numbers, file reference

- [ ] Create approval notification email
  - Subject: "Approval Required: Payment Batch BATCH-YYYY-MM-001"
  - Content: Batch summary, total amount, payment count, link to approval portal
  - Call-to-action: Button to "Review & Approve"

**Tests:**
- Unit: Amount-based approver logic, approval state machine
- Integration: Create batch → submit → approve → schedule → verify SEPA file generated

---

### Task 4.3: Implement SEPA pain.001 Export (Backend)
**Story:** REQ-009, REQ-021  
**Effort:** 4 days  

- [ ] Create `SEPAExportService`
  - Method: `generatePain001(PaymentBatch $batch): SEPAFile`
    - Validate all payments:
      - IBAN format and checksum (mod-97 algorithm)
      - Creditor name (non-empty, <= 70 chars, no forbidden chars)
      - Amount > 0 and < 1,000,000,000
      - Creditor country (extract from IBAN)
    - Group by currency and destination bank
    - Generate pain.001.001.09 XML per group:
      - MessageID: format "MSG[Timestamp]"
      - GroupHeader with InitiatingParty (organization details from config)
      - PaymentInformation block per bank/currency
      - CreditTransferTransactionInformation per payment
      - Payment method: Credit Transfer (CT)
      - Instruction priority: Normal
    - Add signature block (optional, depends on bank requirement)

- [ ] Implement IBAN validation
  - Check format: 2-letter country + 2 digits + alphanumeric
  - Mod-97 checksum: rearrange IBAN, convert to numeric (A=10, B=11, etc.), compute mod 97
  - Should equal 1 for valid IBAN
  - Return detailed error if invalid

- [ ] Implement pain.001 XML generation
  - Use PHP XML library (SimpleXML or DOMDocument)
  - Schema: pain.001.001.09 XSD (validate before return)
  - Envelope structure per ISO 20022 spec
  - Character set: UTF-8 with allowed special chars only

- [ ] Create file handling
  - Save generated XML to Files app (OpenRegister FileService)
  - Generate checksum (SHA-256)
  - Set expiration (e.g., 30 days for bank upload requirement)
  - Return file reference for download

- [ ] Create `SEPAExportController` with methods:
  - `POST /api/treasury/payment-batches/{id}/export-sepa` → generate and return file

- [ ] Implement payment-by-payment SEPA export
  - Method: `exportScheduledPaymentAsSEPA(ScheduledPayment $payment): SEPAFile`
  - For one-off SEPA export from detail page

**Tests:**
- Unit: IBAN validator, pain.001 XML structure validation
- Integration: Create batch with 5 payments, export SEPA, validate XML schema
- Schema validation: Use `xmllint` command or PHP XMLSchemaValidator

---

## Sprint 5: Compliance & Treasury Reporting

### Task 5.1: Implement Wet Fido Compliance Tracking (Full Stack)
**Story:** REQ-012  
**Effort:** 4 days  

**Backend:**
- [ ] Create `WetFidoComplianceService`
  - Load Wet Fido limits from IAppConfig:
    - kasgeldlimiet (max cash at private banks)
    - renterisiconorm (max interest rate exposure)
    - borrowingLimit (max total debt)

  - Method: `calculateCompliance(): WetFidoComplianceReport`
    - Sum: Total cash at private banks
    - Sum: Total interest-bearing debt with > 1 year maturity
    - Sum: Total outstanding loans
    - Compare to limits
    - Calculate: Usage % per limit
    - Determine status: Green/Yellow/Red

  - Method: `getAlert()` if status = Red or Yellow
    - Generate recommendation (e.g., "Deposit EUR 5M to Rijkshoofdboekhouding")

- [ ] Create daily compliance check job
  - Runs at 06:00 CET
  - Calls calculateCompliance()
  - If Red: sends urgent email to treasurer with action items
  - If Yellow: sends advisory email with recommendations

- [ ] Create `WetFidoComplianceController` with methods:
  - `GET /api/treasury/compliance/wet-fido` → current compliance status
  - `GET /api/treasury/compliance/wet-fido/history` → daily history (30-day lookback)
  - `GET /api/treasury/compliance/wet-fido/report` → generate PDF report

**Frontend:**
- [ ] Create Compliance Dashboard page
  - Display 3 Wet Fido KPI cards:
    - kasgeldlimiet: [usage %] of limit
    - renterisiconorm: [usage %] of limit
    - borrowingLimit: [usage %] of limit
  - Color-code: Green < 75%, Yellow 75-90%, Red > 90%
  - Trend chart: 30-day compliance history

- [ ] Create compliance detail pages (one per limit)
  - Show current usage, limit, recommendation
  - Daily trend: [Date, Amount, % of Limit, Status]
  - Drill-down: Which accounts/investments contribute to usage
  - Action items: List recommended actions and their impact

- [ ] Create alert notification
  - Banner on dashboard if Red status
  - Alert text with action recommendation
  - Link to detail page

**Tests:**
- Unit: Limit calculation, status determination (Green/Yellow/Red)
- Integration: Set up compliance limits, verify calculation

---

### Task 5.2: Implement schatkistbankieren Compliance (Full Stack)
**Story:** REQ-013  
**Effort:** 3 days  

**Backend:**
- [ ] Create `SchatkistbankierenComplianceService`
  - Load limit: Max balance at private banks
  - Method: `calculateCompliance(): SchatkistbankierenReport`
    - Sum: Total cash at private banks (excluding Rijkshoofdboekhouding)
    - Calculate: Usage % of limit
    - Determine status: OK / OUT_OF_COMPLIANCE

  - Method: `getRemediationRecommendation()` if out of compliance
    - Recommend transfer amount to Rijkshoofdboekhouding
    - Show calculation: "Current [X]M, Limit [Y]M, Transfer [X-Y]M to comply"

- [ ] Create daily compliance check job
  - Runs at 07:00 CET
  - If out of compliance: sends urgent alert with transfer recommendation
  - Logs status daily

- [ ] Create `SchatkistbankierenComplianceController` with methods:
  - `GET /api/treasury/compliance/schatkistbankieren` → current status
  - `GET /api/treasury/compliance/schatkistbankieren/history` → daily history
  - `GET /api/treasury/compliance/schatkistbankieren/report` → generate quarterly PDF report

**Frontend:**
- [ ] Create schatkistbankieren Compliance page
  - Status card: Balance at Rijkshoofdboekhouding, Balance at private banks, Limit, Utilization %
  - Alert banner if out of compliance with recommended transfer amount
  - Daily history chart: 30-day compliance status (green/red)
  - Recommended action: "Transfer EUR X to Rijkshoofdboekhouding"

**Tests:**
- Unit: Compliance calculation
- Integration: Verify status changes when new deposits are recorded

---

### Task 5.3: Implement Treasury Reporting & PDFs (Full Stack)
**Story:** REQ-016  
**Effort:** 4 days  

**Backend:**
- [ ] Create `TreasuryReportService`
  - Method: `generateMonthlyReport(YearMonth $month): TreasuryReportPDF`
    - Collects data for the month:
      - Average daily balance (simple average)
      - Investment summary (deposito: bank, amount, rate, interest earned)
      - Financing activity (loans, repayments)
      - Wet Fido/schatkistbankieren compliance status
      - Liquidity forecast vs. actuals (if forecast from month start)
      - Bank fees summary
      - FX gains/losses
    - Calculates variance to budget (if budget is configured)

  - Method: `generateWetFidoReport(YearMonth $month): WetFidoReportPDF`
    - Daily compliance status for month
    - Any breaches and remediation taken
    - Financing schedule
    - Signature blocks

  - Method: `generateSchatkistbankierenReport(Quarter $quarter): SchatkistbankierenReportPDF`
    - Daily compliance status for quarter
    - Transfers to Rijkshoofdboekhouding
    - Ministry reporting format

- [ ] Implement PDF generation
  - Use PDF library (e.g., mPDF, TCPDF, or Dompdf)
  - Template: Treasury report layout with logo, tables, charts, signatures
  - Charts: Use image generation library (e.g., GD, ImageMagick) for trend charts
  - Locale: Format dates/numbers per user locale (NL: DD-MM-YYYY, decimal comma)

- [ ] Create `TreasuryReportController` with methods:
  - `POST /api/treasury/reports/monthly` → generate monthly report PDF
  - `POST /api/treasury/reports/wet-fido` → generate Wet Fido report PDF
  - `POST /api/treasury/reports/schatkistbankieren` → generate schatkistbankieren report PDF
  - `GET /api/treasury/reports` → list generated reports

**Frontend:**
- [ ] Create Reports page
  - Form: Select report type (Monthly, Wet Fido, schatkistbankieren)
  - Date picker: Select month/quarter
  - Button: "Generate Report"
  - On completion: Download PDF link + email option

- [ ] Create report archive/history
  - Table: Report type, period, generation date, file size, actions (download, email)
  - Filters: Type, date range

**Tests:**
- Unit: Report data collection (aggregations, calculations)
- Integration: Generate sample report, verify PDF structure and data

---

## Sprint 6: Advanced Features & Compliance

### Task 6.1: Implement Deposito (Short-Term Investment) Management (Full Stack)
**Story:** REQ-014  
**Effort:** 3 days  

**Backend:**
- [ ] Extend `ScheduledPaymentService` to support investment type
  - Create deposito as ScheduledPayment with taskType = "Investment"
  - Record: bank, amount, interest rate, maturity date
  - On creation: reduce available cash balance by deposit amount
  - Check: Wet Fido kasgeldlimiet usage (warn if approaching limit)

- [ ] Implement maturity reminder job
  - Runs daily
  - For deposits maturing within 7 days: send email to treasurer
  - Content: "Deposito at [Bank] for EUR [Amount] matures on [Date]"

- [ ] Implement maturity settlement
  - When deposito matures: add principal + interest back to cash account
  - Create GL entry: Debit Cash, Credit Interest Income

**Frontend:**
- [ ] Create Deposito creation workflow
  - Form: Bank (dropdown of authorized counterparties), Amount, Interest Rate %, Start Date, Maturity Date
  - Validation: Maturity > start date, rate >= 0
  - Check: Warn if Wet Fido kasgeldlimiet usage would exceed 80%
  - Buttons: Cancel, Register Investment

- [ ] Create Deposito list/detail pages
  - List: Bank, Amount, Rate, Start Date, Maturity Date, Days Remaining, Status
  - Detail: Full info, interest accrual calculation, maturity date countdown
  - Status: Active, Maturing Soon (< 7 days), Matured

**Tests:**
- Unit: Interest calculation, Wet Fido impact calculation
- Integration: Register deposito, verify cash balance reduced

---

### Task 6.2: Implement Credit Rating Verification (Full Stack)
**Story:** REQ-17  
**Effort:** 3 days  

**Backend:**
- [ ] Create `CounterpartyCreditRatingService`
  - Load credit rating requirements from IAppConfig (per counterparty type)
  - Method: `verifyRating(Counterparty $counterparty, string $type): RatingVerification`
    - Look up current rating (from third-party data provider or manual input)
    - Compare to minimum requirement
    - Return: compliant (true/false), current rating, required minimum, last update

  - Method: `runComplianceCheck(): RatingComplianceReport`
    - For all registered counterparties
    - Fetch latest ratings
    - Compare to requirements
    - Flag non-compliant counterparties

- [ ] Create daily rating update job
  - Fetches latest ratings from data provider
  - Stores new ratings with timestamp
  - Triggers alerts for any non-compliances

- [ ] Create `CounterpartyCreditRatingController` with methods:
  - `GET /api/treasury/counterparties/{id}/credit-rating` → current rating
  - `GET /api/treasury/counterparties/compliance-check` → all counterparties compliance

**Frontend:**
- [ ] Create Counterparty Credit Rating page
  - Table: Counterparty, Type (Bank/Supplier), Current Rating, Required Minimum, Status, Last Update
  - Color-code: Green (compliant), Red (non-compliant)
  - For non-compliant: Show time to action (e.g., "Transfer by [maturity date]")
  - Actions: View detail, initiate remediation

**Tests:**
- Unit: Rating lookup, compliance check
- Integration: Register counterparty, verify rating check runs daily

---

### Task 6.3: Implement Request for Quotation (RFQ) with Digital Lockbox (Full Stack)
**Story:** REQ-011  
**Effort:** 3 days  

**Backend:**
- [ ] Extend existing RFQ/Bid system with lockbox
  - Method: `createRFQ(array $data): RequestForQuotation`
    - Create RFQ with lockboxEnabled flag
    - If enabled: Set lockboxOpensAt = deadline + 1 hour

  - Method: `submitBid(RequestForQuotation $rfq, Bid $bid): void`
    - Record bid submission (timestamp, bidder, amount, terms)
    - If lockbox enabled: Store bid but DO NOT display to other bidders

  - Method: `openLockbox(RequestForQuotation $rfq): void`
    - Triggered at lockboxOpensAt time
    - Make all bids visible to RFQ owner
    - Send notification: "Lockbox opened, [N] bids available for review"

- [ ] Create RFQ lifecycle job
  - Runs every hour
  - For RFQs with lockboxOpensAt = now: calls openLockbox()

**Frontend:**
- [ ] Create RFQ creation dialog
  - Form: Title, Description, Estimated Value, Currency, Deadline, Lockbox toggle
  - Validation: Deadline in future
  - On save: Create RFQ with auto-generated number

- [ ] Create RFQ list page (buyer view)
  - Table: RFQ #, Title, Estimated Value, Deadline, Status (Published/Closed/Awarded), Bid Count, Lockbox Status
  - Filters: Status, deadline date range
  - Actions: View, Close, Award

- [ ] Create RFQ detail page (buyer view)
  - Lockbox status: If enabled and deadline not reached, show "Lockbox active. Bids hidden until [datetime]"
  - Bids table (if lockbox opened): Amount, Bidder, Key Terms, Submission Date
  - Award button (if closed): Select winning bid, generate award notice

- [ ] Create RFQ page (supplier view)
  - Display: RFQ title, description, requirements, deadline
  - Submit bid button: Form for amount, terms, attachments
  - Confirmation: "Your bid submitted. Bids will be visible [date/time]"

**Tests:**
- E2E: Create RFQ with lockbox, submit bid, verify not visible to others, open lockbox at deadline, verify visible

---

## Sprint 7: Testing, Documentation, Deployment

### Task 7.1: Implement Automated Tests (All layers)
**Effort:** 4 days  

- [ ] Unit tests (PHP)
  - `tests/Unit/Service/CashAccountServiceTest.php` (IBAN validation, create, update)
  - `tests/Unit/Service/SEPAExportServiceTest.php` (IBAN validation, pain.001 structure)
  - `tests/Unit/Service/WetFidoComplianceServiceTest.php` (limit calculations)
  - Coverage target: > 80% for services

- [ ] Unit tests (Frontend - Vue components)
  - Treasury Dashboard KPI calculations
  - Form validation (amounts, dates, IBANs)
  - Store mutations

- [ ] Integration tests (API)
  - Create CashAccount via API, verify in OpenRegister
  - Import CAMT.053 file, verify transactions and balance update
  - Create PaymentBatch, submit for approval, verify approval chain
  - Generate pain.001 SEPA file, validate XML schema

- [ ] Browser tests (Playwright)
  - Dashboard loads and displays KPIs
  - Create new CashAccount (full workflow)
  - Import bank statement (upload, preview, import)
  - Schedule payment (form fill, preview, create)
  - Approve payment batch (view, approve, schedule, export)
  - Verify Wet Fido compliance dashboard

- [ ] Test data setup
  - Create fixture: test CashAccounts, Payees, ScheduledPayments
  - Create CAMT.053 test file for import testing

**Tests:** Run all in `composer check:strict`

---

### Task 7.2: Create User Documentation (Technical Writers)
**Effort:** 2 days  

- [ ] Create feature documentation
  - `docs/treasury-overview.md` - Module overview and architecture
  - `docs/cash-accounts.md` - Creating, managing, importing statements
  - `docs/payment-management.md` - Scheduling, batching, approval, SEPA export
  - `docs/forecasting.md` - How forecasting works, interpreting results
  - `docs/compliance.md` - Wet Fido, schatkistbankieren monitoring
  - `docs/reporting.md` - Monthly reports, audit trails

- [ ] Create administrator guides
  - `docs/admin/setup.md` - Configuration, Wet Fido limits, credit rating sources
  - `docs/admin/integration.md` - Bank connections, API setup, background jobs

- [ ] Include screenshots from running app
  - Dashboard, account list, payment approval flow, SEPA export, compliance dashboard

---

### Task 7.3: Deployment & Go-Live (DevOps)
**Effort:** 2 days  

- [ ] Prepare deployment package
  - Database migrations (OpenRegister schemas)
  - Configuration (Wet Fido limits, compliance thresholds, background job schedule)
  - Background jobs setup (bank import, payment execution, compliance checks, forecast refresh)

- [ ] Deploy to staging
  - Run migrations
  - Test all end-to-end workflows
  - Performance validation (large payment batches, forecast generation)
  - Load testing (concurrent users, API throughput)

- [ ] Prepare go-live
  - Finalize limits and thresholds with customer
  - Train end-users (treasurers, controllers, CFO)
  - Set up monitoring and alerting
  - Backup and disaster recovery plan

- [ ] Go-live (production deployment)
  - Deploy during maintenance window
  - Verify all functions working
  - Monitor system for errors/performance issues

---

## Sprint 8: Polish & Optimization (Optional)

### Task 8.1: Performance Optimization
- [ ] Optimize forecast generation (caching, pagination for large datasets)
- [ ] Optimize SEPA export (streaming for large batches)
- [ ] Add database indexes on frequently-queried columns (accountId, status, dueDate)
- [ ] Frontend: Lazy-load pages, virtual scrolling for large lists

### Task 8.2: UX Improvements
- [ ] Add real-time notifications for approval requests
- [ ] Implement search across all entities (payments, accounts, tasks)
- [ ] Add bulk actions (delete multiple, reschedule multiple)
- [ ] Keyboard shortcuts for frequent actions

### Task 8.3: Advanced Features
- [ ] Multi-currency forecasting (project balance in each currency)
- [ ] Hedging recommendations engine
- [ ] Integration with bank's eBICS connectivity (for automatic import)
- [ ] Mobile app for treasurer notifications

---

## Summary

**Total Effort:** 5-7 sprints (180-250 person-days for 4-person team)

**Key Dependencies:**
1. Sprints 1-2: Foundation (schema, import, dashboard)
2. Sprint 3: Builds on Sprint 2 (forecasting uses historical data from imports)
3. Sprint 4: Builds on Sprints 1-2 (payment batching uses accounts, imports)
4. Sprint 5: Builds on Sprints 1-4 (compliance tracking uses accounts, imports)
5. Sprints 6-7: Polish and testing

**Deployment Ready:** End of Sprint 7 (production release)

---

## Notes

- **Spec Traceability:** Every PHP class and method MUST have `@spec openspec/changes/treasury-cash-management/tasks.md#task-X.Y` PHPDoc tags per ADR-003.
- **Testing:** All code MUST pass `composer check:strict` (lint, type checking, tests).
- **Documentation:** Update docs when behavior changes. Screenshots should reflect actual UI.
- **Compliance:** Wet Fido, schatkistbankieren, BBV, IV3 requirements are mandatory for go-live in Dutch municipalities.
