# Implementation Tasks: Treasury & Cash Management — Shillinq

**Change:** treasury-cash-management-other-t2  
**Version:** 1.0  
**Date:** 2026-05-21  
**Status:** Task Breakdown

---

## Phase 1: Data Model & Backend Services (Weeks 1–3)

### Task 1: OpenRegister Schema Registration

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-001**

- [ ] Create `register.shillinq_treasury.json` in `lib/Settings/` with schemas:
  - [ ] BankAccount (properties: accountHolder, accountNumber, currency, bankName, bic, accountType, syncStatus, lastSyncDate)
  - [ ] Payment (properties: amount, currency, paymentMethod, status, payer, payee, bankAccount, reference, dates)
  - [ ] PaymentBatch (properties: payments, totalAmount, status, approver)
  - [ ] CashFlowForecast (properties: forecastDate, projections, riskFlags)
  - [ ] ReconciliationRule (properties: matchCriteria, isActive, priority)
  - [ ] PaymentMethod (properties: methodType, processorName, fee, limits, currencies)
- [ ] Register all schemas with OpenRegister via `RegisterService`
- [ ] Create migration: `register_schemas_treasury.php` in `lib/Migration/`
- [ ] Add PHP Schema classes (in `lib/Schemas/`) for type hints + validation
- [ ] Test: `tests/Unit/Service/SchemaServiceTest.php` — verify schema registration
- [ ] Test: `tests/Integration/RegisterTest.php` — verify OpenRegister connectivity

**Deliverable:** OpenRegister schemas accessible via API `GET /register/shillinq_treasury/schemas/{name}`

---

### Task 2: Database Mappers (CRUD Layer)

**Complexity:** Low | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-001**

- [ ] Create mappers in `lib/Db/Mapper/`:
  - [ ] `BankAccountMapper` — create, read, update, delete, findBySyncStatus()
  - [ ] `PaymentMapper` — create, read, update, delete, findByStatus(), findByPayee()
  - [ ] `PaymentBatchMapper` — create, read, update, delete, findByStatus()
  - [ ] `CashFlowForecastMapper` — create, read, update, delete, findLatest()
  - [ ] `ReconciliationRuleMapper` — create, read, update, delete, findByBankAccount()
  - [ ] `PaymentMethodMapper` — create, read, update, delete, findEnabled()
- [ ] Database tables: `openregister_objects` (managed by OR) — NO custom tables for domain data
- [ ] Test: `tests/Unit/Db/MapperTest.php` — full CRUD coverage per mapper

**Deliverable:** All mappers support CRUD operations, tested, ready for services to use

---

### Task 3: Backend Service Layer — Payment Operations

**Complexity:** High | **Owner:** Senior Backend  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-003, REQ-TCM-004**

- [ ] Create `lib/Service/PaymentService.php`:
  - [ ] `createPayment(amount, payee, method, bankAccount, reference, notes)` — validates, creates draft
  - [ ] `updatePayment(paymentId, fields)` — edit only if draft (guards status)
  - [ ] `deletePayment(paymentId)` — only if draft (audit trail)
  - [ ] `schedulePayment(paymentId)` — transitions draft → scheduled, validates future date
  - [ ] `getPaymentStatus(paymentId)` — fetch current status
  - [ ] `calculateFee(amount, paymentMethod)` — applies fee structure
  - [ ] `validatePayment(payment)` — amount > 0, payee required, dates valid
  - [ ] `retryPayment(paymentId)` — resubmit failed payment
  - All methods: log `@spec` tags for traceability
- [ ] Exception handling: `PaymentValidationException`, `PaymentStateException`
- [ ] Integration:
  - [ ] `PaymentService` injects `PaymentMapper`, `PaymentMethodService`, `IAppConfig`
  - [ ] DI in `appinfo/routes.php` (constructor injection, NOT `OC::$server`)
- [ ] Test: `tests/Unit/Service/PaymentServiceTest.php` — 15+ test methods
  - [ ] Test validation (amount > 0, payee required, future date)
  - [ ] Test state transitions (draft → scheduled, validate guards)
  - [ ] Test fee calculation (fixed, percentage, tiered)

**Deliverable:** `PaymentService` fully tested, handles REQ-TCM-003, REQ-TCM-004

---

### Task 4: Backend Service Layer — Bank Account & Sync

**Complexity:** High | **Owner:** Senior Backend  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-001, REQ-TCM-002**

- [ ] Create `lib/Service/BankAccountService.php`:
  - [ ] `createBankAccount(accountHolder, accountNumber, currency, ...)` — validates IBAN, creates, sets sync pending
  - [ ] `updateBankAccount(accountId, fields)` — edit account details
  - [ ] `deleteBankAccount(accountId)` — soft-delete (mark inactive, don't remove GL history)
  - [ ] `registerWithOpenRegister(bankAccount)` — async background job to sync
  - [ ] `validateIban(iban)` — checksum validation, format check
  - [ ] `lookupBankFromBic(bic)` — returns bank name + details
  - All methods: log `@spec` tags
- [ ] Create `lib/Service/BankSyncService.php`:
  - [ ] `syncBankStatement(bankAccountId)` — queries OpenRegister, creates Payment records
  - [ ] `applyReconciliationRules(transactions)` — matches bank txns to GL via rules
  - [ ] `createPaymentsFromStatement(statement)` — batch creation of Payment objects
  - [ ] `handleDuplicateDetection(transaction)` — hash(amount + date + reference)
  - [ ] `updateSyncStatus(bankAccountId, status)` — pending, connected, error
- [ ] Background job: `lib/Job/BankAccountSyncJob.php`
  - [ ] Triggered daily (6 AM UTC)
  - [ ] Iterates all BankAccounts with syncStatus="connected"
  - [ ] Calls `BankSyncService.syncBankStatement()`
  - [ ] On error: retry up to 3 times, then notify admin
  - [ ] Logging: all syncs logged to AuditTrail
- [ ] Integration:
  - [ ] Injects: `BankAccountMapper`, `PaymentMapper`, `RegisterService`, `ReconciliationService`
  - [ ] DI in routes.php + job registration in `appinfo/app.php`
- [ ] Test: `tests/Unit/Service/BankAccountServiceTest.php`, `BankSyncServiceTest.php`
  - [ ] Test IBAN validation (valid/invalid formats)
  - [ ] Test BIC lookup (mocked responses)
  - [ ] Test duplicate detection (same txn twice = skipped)
  - [ ] Test reconciliation rule application (3 rules, 95% match)

**Deliverable:** Bank account management + daily sync job, handles REQ-TCM-001, REQ-TCM-002

---

### Task 5: Backend Service Layer — Reconciliation

**Complexity:** High | **Owner:** Senior Backend  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-006, REQ-TCM-009**

- [ ] Create `lib/Service/ReconciliationService.php`:
  - [ ] `createReconciliationRule(ruleName, bankAccount, matchCriteria, priority)` — stores rule
  - [ ] `applyReconciliationRules(transactions, rules)` — matches bank txns to GL
  - [ ] `matchTransaction(bankTxn, glEntries, rule)` — fuzzy match (amount, date, reference, party)
  - [ ] `createMatch(bankTransaction, glEntry)` — creates Match record
  - [ ] `approveReconciliation(matches)` — posts all matches to ReconciliationMatch GL entries
  - [ ] `updateRuleSuccessRate(rule)` — tracks historical accuracy (for feedback)
  - All methods: log `@spec` tags
- [ ] Matching algorithm:
  - [ ] Amount tolerance: configurable (default €1.00)
  - [ ] Date tolerance: configurable (default 3 days)
  - [ ] Reference pattern: regex matching (e.g., `INV-\d{4}-\d{3}`)
  - [ ] Party name: fuzzy match (Levenshtein distance, threshold 80%)
- [ ] Integration:
  - [ ] Injects: `ReconciliationRuleMapper`, `GeneralLedgerService`, `PaymentMapper`
- [ ] Test: `tests/Unit/Service/ReconciliationServiceTest.php`
  - [ ] Test rule creation + retrieval
  - [ ] Test amount matching (within tolerance)
  - [ ] Test date matching (within days)
  - [ ] Test reference pattern (regex)
  - [ ] Test party fuzzy matching (80%+ similarity)
  - [ ] Test success rate tracking

**Deliverable:** Reconciliation rules + matching engine, handles REQ-TCM-006, REQ-TCM-009

---

### Task 6: Backend Service Layer — Cash Flow Forecast

**Complexity:** High | **Owner:** Senior Backend  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-007**

- [ ] Create `lib/Service/CashFlowForecastService.php`:
  - [ ] `generateForecast(forecastDate, projectionDays=90)` — computes 90-day projection
  - [ ] `detectRecurringPatterns(glEntries, window=90)` — analyzes historical GL for recurrence
  - [ ] `projectInflows(date, patterns, scheduledPayments)` — sum AR + deposits expected
  - [ ] `projectOutflows(date, patterns, scheduledPayments)` — sum AP + payments due
  - [ ] `calculateNetPosition(baseBalance, inflows, outflows)` — cumulative balance
  - [ ] `calculateConfidence(daysAhead, historicalVariance)` — decreases with horizon
  - [ ] `detectRiskFlags(projections)` — identifies negative positions, large spikes
  - All methods: log `@spec` tags
- [ ] Forecast algorithm:
  - [ ] Historical analysis: last 90 GL entries for recurring patterns (daily, weekly, monthly)
  - [ ] Confidence: base 95%, decreased by 1% per day (at day 90: 5%)
  - [ ] Risk detection: negative balance, outflow >€50k in single day, high variance
- [ ] Background job: `lib/Job/CashFlowForecastJob.php`
  - [ ] Triggered daily (5 AM UTC, before bank sync)
  - [ ] Calls `CashFlowForecastService.generateForecast()` for all orgs
  - [ ] Stores result in OpenRegister (CashFlowForecast object)
- [ ] Integration:
  - [ ] Injects: `GeneralLedgerService`, `PaymentMapper`, `PaymentBatchMapper`
- [ ] Test: `tests/Unit/Service/CashFlowForecastServiceTest.php`
  - [ ] Test pattern detection (daily/weekly/monthly)
  - [ ] Test projection calculation (inflows + outflows = position)
  - [ ] Test confidence decay (95% → 5% over 90 days)
  - [ ] Test risk flag detection (negative, large outflow)

**Deliverable:** Daily cash flow forecast generation, handles REQ-TCM-007

---

### Task 7: Backend Service Layer — Multi-Currency & FX

**Complexity:** High | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-012, REQ-TCM-019**

- [ ] Create `lib/Service/FxService.php`:
  - [ ] `getExchangeRate(fromCurrency, toCurrency, date)` — fetches from ECB API, caches 1hr
  - [ ] `convertAmount(amount, fromCurrency, toCurrency, date)` — applies FX rate
  - [ ] `calculateFxGainLoss(originalAmount, originalRate, settledAmount, settledRate)` — realized gain/loss
  - [ ] `calculateUnrealizedExposure(accountBalance, accountCurrency, baseCurrency, currentRate)` — unrealized
  - All methods: log `@spec` tags
- [ ] Create `lib/Service/MultiCurrencyService.php`:
  - [ ] `validateCurrencyForPaymentMethod(paymentMethod, currency)` — checks if method supports currency
  - [ ] `aggregateBalances(bankAccounts)` — converts all to base currency for reporting
  - [ ] `getSubsidiaryGlAccount(currency)` — returns GL account for currency (1000-EUR, 1001-USD, etc.)
- [ ] Integration:
  - [ ] ECB API connector: `lib/Connector/EcbApi.php` (fetch daily rates, cache in `IAppConfig`)
  - [ ] Fallback: memoized rates if API unavailable (max 24hr stale)
- [ ] Test: `tests/Unit/Service/FxServiceTest.php`
  - [ ] Test FX rate fetch (mocked ECB)
  - [ ] Test conversion (amount × rate)
  - [ ] Test realized/unrealized gain/loss
  - [ ] Test caching (1hr)

**Deliverable:** FX management + multi-currency support, handles REQ-TCM-012, REQ-TCM-019

---

### Task 8: Backend Service Layer — Payment Method Configuration

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-005**

- [ ] Create `lib/Service/PaymentMethodService.php`:
  - [ ] `getPaymentMethod(methodType)` — fetch from OpenRegister
  - [ ] `getEnabledMethods(baseCurrency=null)` — filter by enabled status + currency support
  - [ ] `validateMethodForPayment(method, amount, currency, payee)` — checks limits, currency, payee type
  - [ ] `calculateFee(amount, paymentMethod)` — applies fee structure (fixed, %, tiered)
  - [ ] `updatePaymentMethodConfig(method, config)` — stores API credentials via IAppConfig
  - All methods: log `@spec` tags
- [ ] Fee structures:
  - [ ] Fixed fee: amount in EUR
  - [ ] Percentage: % of transaction amount
  - [ ] Tiered: different % based on amount ranges
- [ ] Integration:
  - [ ] Stores sensitive data (API keys, webhooks) in `IAppConfig` with `sensitive=true`
  - [ ] Injects: `IAppConfig`, `PaymentMethodMapper`
- [ ] Test: `tests/Unit/Service/PaymentMethodServiceTest.php`
  - [ ] Test fee calculation (fixed, %, tiered)
  - [ ] Test method filtering (enabled, currency match)
  - [ ] Test validation (amount limits, currency)

**Deliverable:** Payment method management + configuration, handles REQ-TCM-005

---

### Task 9: Backend Controllers & API Endpoints

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-001 through REQ-TCM-005**

- [ ] Create `lib/Controller/BankAccountController.php`:
  - [ ] `GET /api/bank-accounts` — list all accounts (with pagination)
  - [ ] `POST /api/bank-accounts` — create new account
  - [ ] `GET /api/bank-accounts/{id}` — fetch account detail
  - [ ] `PUT /api/bank-accounts/{id}` — update account
  - [ ] `DELETE /api/bank-accounts/{id}` — soft-delete (mark inactive)
  - [ ] `POST /api/bank-accounts/{id}/sync` — trigger manual sync
  - All endpoints: validate auth (admin or treasurer role), log `@spec`
- [ ] Create `lib/Controller/PaymentController.php`:
  - [ ] `GET /api/payments` — list payments (with filters: status, payee, date range)
  - [ ] `POST /api/payments` — create payment
  - [ ] `GET /api/payments/{id}` — fetch payment detail
  - [ ] `PUT /api/payments/{id}` — update (only if draft)
  - [ ] `DELETE /api/payments/{id}` — delete (only if draft)
  - [ ] `POST /api/payments/{id}/schedule` — transition to scheduled
  - [ ] `POST /api/payments/{id}/retry` — retry failed payment
- [ ] Create `lib/Controller/PaymentBatchController.php`:
  - [ ] `GET /api/payment-batches` — list batches
  - [ ] `POST /api/payment-batches` — create batch (from draft payments)
  - [ ] `GET /api/payment-batches/{id}` — fetch batch
  - [ ] `PUT /api/payment-batches/{id}` — update (before approval)
  - [ ] `POST /api/payment-batches/{id}/approve` — submit for approval
  - [ ] `POST /api/payment-batches/{id}/execute` — execute after approval
- [ ] Create `lib/Controller/ReconciliationController.php`:
  - [ ] `GET /api/reconciliation/rules` — list rules
  - [ ] `POST /api/reconciliation/rules` — create rule
  - [ ] `POST /api/reconciliation/match` — manual match bank txn to GL
  - [ ] `POST /api/reconciliation/approve` — approve all matches, post GL
  - [ ] `GET /api/reconciliation/status/{bankAccountId}` — fetch matching status (% matched)
- [ ] Create `lib/Controller/ForecastController.php`:
  - [ ] `GET /api/forecasts/latest` — fetch latest cash flow forecast
  - [ ] `POST /api/forecasts/generate` — trigger manual generation
- [ ] Thin controllers (<10 lines per method):
  - [ ] Validation in service layer
  - [ ] Response formatting: status code + message + data
- [ ] Error handling: appropriate HTTP status (400 validation, 401 auth, 403 permission, 500 server)
- [ ] Test: `tests/Integration/ApiTest.php` — Postman/Newman collections
  - [ ] Test all CRUD endpoints
  - [ ] Test state transitions
  - [ ] Test error cases (invalid input, auth failure)

**Deliverable:** RESTful API for all treasury operations, HTTP tested

---

### Task 10: Background Jobs (Cron-Triggered)

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-002, REQ-TCM-007**

- [ ] Create `lib/Job/BankAccountSyncJob.php`:
  - [ ] Trigger: daily at 6 AM UTC
  - [ ] Iterate all BankAccounts with syncStatus="connected"
  - [ ] Call `BankSyncService.syncBankStatement()`
  - [ ] On error: retry up to 3 times, then mark syncStatus="error", notify admin
- [ ] Create `lib/Job/CashFlowForecastJob.php`:
  - [ ] Trigger: daily at 5 AM UTC (before bank sync)
  - [ ] Call `CashFlowForecastService.generateForecast()`
  - [ ] Store result in OpenRegister (CashFlowForecast object)
- [ ] Create `lib/Job/ScheduledPaymentReminderJob.php`:
  - [ ] Trigger: daily at 8 AM UTC
  - [ ] Find all Payments with status="scheduled" AND requestedDate=tomorrow
  - [ ] Send notification to Payment.payer
- [ ] Create `lib/Job/ScheduledPaymentSubmitJob.php`:
  - [ ] Trigger: daily at 6 AM UTC (for each payment's requestedDate)
  - [ ] Find all Payments with status="scheduled" AND requestedDate=today
  - [ ] Submit to payment processor (SEPA, Stripe, etc.)
  - [ ] Update status → "initiated" → "pending"
- [ ] Job registration: `appinfo/app.php` (register with JobRegistry)
- [ ] Error handling: catch exceptions, log to AuditTrail, notify admin on critical failures
- [ ] Test: `tests/Unit/Job/JobTest.php` — mock timing, verify job execution
  - [ ] Test bank sync job execution
  - [ ] Test forecast generation
  - [ ] Test payment submission on due date

**Deliverable:** All background jobs registered + tested

---

### Task 11: Webhook Handlers for Payment Processor Events

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-010**

- [ ] Create `lib/Controller/PaymentProcessorWebhookController.php`:
  - [ ] `POST /api/payment-method-webhook` — receives CloudEvents from processors
  - [ ] Parse webhook payload (signature verification per processor)
  - [ ] Handle events:
    - [ ] `payment.authorized` — update status → "authorized", send notification
    - [ ] `payment.captured` — update status → "captured", send notification
    - [ ] `payment.settled` — update status → "settled", post GL entries, notify payee
    - [ ] `payment.failed` — update status → "failed", send alert to creator, log reason
  - [ ] Idempotency: track event IDs (prevent double-processing)
  - All handlers: log `@spec` tags
- [ ] Webhook signature verification:
  - [ ] Stripe: `Stripe-Signature` header + HMAC-SHA256
  - [ ] Mollie: `X-Mollie-Signature` header + HMAC-SHA256
  - [ ] Generic: webhook secret from PaymentMethod config
- [ ] Integration:
  - [ ] Injects: `PaymentService`, `NotificationService`, `GeneralLedgerService`
- [ ] Test: `tests/Unit/Controller/WebhookControllerTest.php`
  - [ ] Test webhook signature verification
  - [ ] Test event parsing (different processor formats)
  - [ ] Test status updates + notifications
  - [ ] Test idempotency (duplicate event = no double-post)

**Deliverable:** Webhook handlers for processor events

---

## Phase 2: Frontend Components (Weeks 2–3)

### Task 12: Frontend Store Setup

**Complexity:** Low | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-001 through REQ-TCM-020**

- [ ] Create `src/store/store.js` — initialize all object stores:
  - [ ] `createObjectStore('bankAccount', 'BankAccount', 'shillinq_treasury')`
  - [ ] `createObjectStore('payment', 'Payment', 'shillinq_treasury')`
  - [ ] `createObjectStore('paymentBatch', 'PaymentBatch', 'shillinq_treasury')`
  - [ ] `createObjectStore('cashFlowForecast', 'CashFlowForecast', 'shillinq_treasury')`
  - [ ] `createObjectStore('reconciliationRule', 'ReconciliationRule', 'shillinq_treasury')`
  - [ ] `createObjectStore('paymentMethod', 'PaymentMethod', 'shillinq_treasury')`
- [ ] Register store plugins:
  - [ ] `auditTrailsPlugin` — track all changes
  - [ ] `filesPlugin` — attach documents (statements, receipts)
  - [ ] `relationsPlugin` — resolve payee → Supplier, etc.
  - [ ] `searchPlugin` — full-text search
- [ ] Store initialization: called in `App.vue` `created()` hook
- [ ] Settings store: Pinia `defineStore` with `fetchSettings()`, `saveSettings()`
- [ ] Test: `tests/Frontend/store.test.js` (if test framework present)

**Deliverable:** All stores initialized, plugins ready

---

### Task 13: Main App Layout & Navigation

**Complexity:** Medium | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture**

- [ ] Create `src/App.vue`:
  - [ ] `NcContent` → 3 states:
    - [ ] Loading: `NcLoadingIcon`
    - [ ] No OpenRegister: `NcEmptyContent` ("Please enable OpenRegister")
    - [ ] Ready: `MainMenu` + `NcAppContent` + `router-view`
  - [ ] Inject `sidebarState` for child components (for detail pages)
  - [ ] `created()` hook: call `initializeStores()` (load settings, register object types)
  - [ ] Watch `$route` to update active menu item
- [ ] Create `src/components/MainMenu.vue`:
  - [ ] `NcAppNavigation` with `NcAppNavigationItem` per route:
    - [ ] Dashboard (icon: bar-chart-2)
    - [ ] Bank Accounts (icon: bank)
    - [ ] Payments (icon: send)
    - [ ] Payment Batches (icon: layers)
    - [ ] Reconciliation (icon: check-circle)
    - [ ] Forecasts (icon: trending-up)
    - [ ] Payment Methods (settings, icon: credit-card)
    - [ ] Reports (icon: file-text)
  - [ ] Footer: Settings link via `NcAppNavigationSettings`
- [ ] Create `src/router/index.js`:
  - [ ] Routes (flat, named, props via arrow function):
    - [ ] `/` → Dashboard
    - [ ] `/bank-accounts` → BankAccountList
    - [ ] `/bank-accounts/:id` → BankAccountDetail
    - [ ] `/payments` → PaymentList
    - [ ] `/payments/:id` → PaymentDetail
    - [ ] `/payment-batches` → PaymentBatchList
    - [ ] `/payment-batches/:id` → PaymentBatchDetail
    - [ ] `/reconciliation` → ReconciliationHub
    - [ ] `/settings` → Settings
    - [ ] `*` → redirect to `/`
  - [ ] History mode: `base: '/index.php/apps/shillinq/'`

**Deliverable:** App layout + main menu + router

---

### Task 14: Dashboard & Analytics

**Complexity:** High | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture, Page 1**

- [ ] Create `src/pages/Dashboard.vue`:
  - [ ] Component: `CnDashboardPage` (GridStack layout, editable)
  - [ ] Widgets:
    - [ ] **KPI Block 1:** Cash Position
      - [ ] Fetch latest BankAccount balances
      - [ ] Convert all to base currency (via FxService on backend)
      - [ ] Display: "€125,000" with trend sparkline (7-day)
    - [ ] **KPI Block 2:** Forecast Variance
      - [ ] Fetch latest CashFlowForecast
      - [ ] Compare forecasted (90 days) vs actual (today)
      - [ ] Display: "±3%" with color coding (green <5%, yellow 5–10%, red >10%)
    - [ ] **KPI Block 3:** Payment Velocity
      - [ ] Count payments settled in <2 days (last 30 days)
      - [ ] Calculate %: (settled <2 days) / (total settled)
      - [ ] Display: "95%" with benchmark (target: 90%)
    - [ ] **KPI Block 4:** Reconciliation Rate
      - [ ] Last 7 days: % of transactions successfully matched
      - [ ] Display: "99%" (green if >95%, yellow if 80–95%)
    - [ ] **Chart 1:** 90-Day Cash Flow Projection (area chart)
      - [ ] X: date (every 7 days labeled)
      - [ ] Y: cumulative cash position
      - [ ] Shaded area: confidence band (±confidence%)
      - [ ] Line color: green if always positive, red if dips negative
      - [ ] Tooltip: hover day → shows inflows/outflows/position
      - [ ] Risk flags: markers on dates with high-risk alerts
    - [ ] **Chart 2:** Payment Status Distribution (donut chart)
      - [ ] Segments: draft, scheduled, pending, settled, failed
      - [ ] Counts + % per segment
      - [ ] Click segment → filter PaymentList by status
    - [ ] **Alert Panel:** Risk Flags
      - [ ] Fetch CashFlowForecast.riskFlags
      - [ ] Display: icon (high=🔴, medium=🟡), message, recommendation
      - [ ] Max 3 visible (older alerts below fold)
    - [ ] **Quick Actions:** Buttons
      - [ ] "+ New Payment" → PaymentDetail (id='new')
      - [ ] "Reconcile" → ReconciliationHub
      - [ ] "View Forecast" → ForecastDetail
      - [ ] "Settings" → Settings page
  - [ ] Data loading: `Promise.all()` for all widgets (parallel)
  - [ ] Error handling: empty state if no OpenRegister, loading skeleton while fetching
  - [ ] Refresh: auto-refresh every 5 minutes (via `setInterval`)
- [ ] Test: `tests/Frontend/pages/Dashboard.test.js`
  - [ ] Test widget rendering (all KPIs visible)
  - [ ] Test data fetching (API calls)
  - [ ] Test chart rendering (ApexCharts)

**Deliverable:** Interactive dashboard with KPIs, charts, alerts

---

### Task 15: Bank Accounts List & Detail Pages

**Complexity:** High | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture, Pages 2–3**

- [ ] Create `src/pages/BankAccountList.vue`:
  - [ ] Component: `CnIndexPage` + `CnDataTable`
  - [ ] Columns:
    - [ ] Account Holder (text)
    - [ ] IBAN (text, masked: NL91****1300)
    - [ ] Currency (badge: EUR, USD, GBP)
    - [ ] Balance (number, formatted with currency)
    - [ ] Sync Status (badge: connected=green, pending=yellow, error=red, disconnected=gray)
    - [ ] Last Sync (date, relative: "2 hours ago")
    - [ ] Reconciliation Date (date)
  - [ ] Actions (per row):
    - [ ] View (→ BankAccountDetail)
    - [ ] Edit (→ BankAccountDetail in edit mode)
    - [ ] Sync (trigger manual sync)
    - [ ] Reconcile (→ ReconciliationHub with account pre-selected)
    - [ ] Download Statement (fetch recent statement PDF)
  - [ ] Filters (left sidebar via `CnFilterBar`):
    - [ ] Sync Status (multi-select)
    - [ ] Currency (multi-select)
    - [ ] Account Type (checking, savings, credit_card, virtual)
    - [ ] Active/Inactive toggle
  - [ ] Sorting: by balance (desc), last sync (desc), account holder (asc)
  - [ ] Pagination: 25 per page (configurable)
  - [ ] Add button: "+ New Bank Account" → BankAccountDetail (id='new')

- [ ] Create `src/pages/BankAccountDetail.vue`:
  - [ ] Component: `CnDetailPage` + `CnDetailGrid` + `CnObjectSidebar`
  - [ ] Sections:
    - [ ] **Account Info** (edit mode: `CnFormDialog`, view mode: `CnDetailGrid`):
      - [ ] Account Holder (text)
      - [ ] IBAN (text, validated, masked in view)
      - [ ] Currency (dropdown: EUR, USD, GBP, CHF, SEK, DKK, NOK)
      - [ ] Bank Name (text, auto-filled from BIC if available)
      - [ ] BIC (text, lookup enabled)
      - [ ] Account Type (dropdown)
      - [ ] Is Active (toggle)
    - [ ] **Sync Status** (read-only):
      - [ ] Status badge (connected/pending/error/disconnected)
      - [ ] Last Sync date + time
      - [ ] Sync error message (if status="error")
      - [ ] "+ Sync Now" button (trigger BankSyncJob manually)
    - [ ] **Recent Transactions** (table of last 10 payments):
      - [ ] Date, Amount, Payee, Reference, Status
      - [ ] Click row → PaymentDetail
    - [ ] **Sidebar Tabs** (via `CnObjectSidebar`):
      - [ ] Files: uploaded statements (PDFs), download link
      - [ ] Notes: internal notes on account
      - [ ] Audit Trail: sync history, reconciliation events
      - [ ] Tasks: related tasks (sync failures, reconciliation overdue)
  - [ ] Edit mode:
    - [ ] Save button → POST/PUT to API
    - [ ] Cancel button → discard changes
    - [ ] Delete button → soft-delete (mark inactive)
  - [ ] New mode (id='new'):
    - [ ] All fields editable, empty
    - [ ] Save → create account, set sync pending
  - [ ] Error handling: validation messages inline (IBAN format, required fields)
  - [ ] Refresh: manual "Reload" button

**Deliverable:** Bank account management pages

---

### Task 16: Payments List & Detail Pages

**Complexity:** High | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture, Page 2-3 + REQ-TCM-003, REQ-TCM-004**

- [ ] Create `src/pages/PaymentList.vue`:
  - [ ] Component: `CnIndexPage` + `CnDataTable`
  - [ ] Columns:
    - [ ] Date (request date)
    - [ ] Amount (currency badge EUR, USD, etc.)
    - [ ] Payee (name, abbreviated if long)
    - [ ] Method (badge: bank_transfer, credit_card, etc.)
    - [ ] Status (badge: draft=gray, scheduled=blue, pending=yellow, settled=green, failed=red)
    - [ ] Reference (text, truncated)
    - [ ] Approver (if awaiting approval)
  - [ ] Filters:
    - [ ] Status (draft, scheduled, pending, settled, failed, rejected)
    - [ ] Payment Method (dropdown)
    - [ ] Date Range (from/to picker)
    - [ ] Payee (search/autocomplete)
    - [ ] Amount Range (min/max)
    - [ ] Is Personal (toggle)
  - [ ] Sorting: by date (desc), amount (desc), status (asc)
  - [ ] Bulk actions (multi-select):
    - [ ] Delete (only if draft)
    - [ ] Schedule (only if draft)
    - [ ] Export (CSV: all visible columns)
    - [ ] Add to Batch (only if status=draft)
  - [ ] Add button: "+ New Payment" → PaymentDetail (id='new')
  - [ ] Mass action bar (floating): "X selected" + actions

- [ ] Create `src/pages/PaymentDetail.vue`:
  - [ ] Component: `CnDetailPage` + `CnFormDialog` (schema-driven) + `CnObjectSidebar`
  - [ ] Edit form fields (validate in real-time):
    - [ ] Amount (number, >0, currency picker)
    - [ ] Currency (dropdown, defaults to account currency)
    - [ ] Payment Method (dropdown, filtered by enabled + currency)
    - [ ] Payee (autocomplete: Suppliers, Employees, Organizations)
    - [ ] Bank Account (dropdown: filter by currency + active status)
    - [ ] Requested Date (date picker, can be future for scheduled)
    - [ ] Reference (text, max 140 chars, counter)
    - [ ] Is Personal (toggle)
    - [ ] Personal Category (conditional dropdown: travel, meals, office, other)
    - [ ] Notes (textarea)
    - [ ] Fee (read-only, calculated from method + amount)
    - [ ] Total Debit (read-only, amount + fee)
  - [ ] View mode:
    - [ ] All fields read-only as `CnDetailGrid`
    - [ ] Status badge (prominently displayed)
    - [ ] Approval info (if awaiting approval): "Awaiting approval from Jane Smith" + approve/reject buttons (if user is approver)
  - [ ] Actions (per status):
    - [ ] draft: Save, Schedule, Delete, Edit
    - [ ] scheduled: Cancel, Edit (locked unless draft), View
    - [ ] pending: View, Retry (if failed)
    - [ ] settled: View (locked), Export, Print
    - [ ] failed: Retry, Edit (create new), View
  - [ ] Sidebar tabs (via `CnObjectSidebar`):
    - [ ] Files: attachments (invoice, receipt, PO)
    - [ ] Notes: internal notes + approval comments
    - [ ] Related: linked documents (Invoice, PO, Supplier)
    - [ ] Audit Trail: creation, edits, approvals, status changes
  - [ ] New mode (id='new'):
    - [ ] All fields editable, empty
    - [ ] Payee pre-filled if coming from Supplier detail (via route params)
    - [ ] Amount pre-filled if coming from Invoice (via route params)
    - [ ] Save → create draft, show confirmation
  - [ ] Validation:
    - [ ] Amount > 0
    - [ ] Payee required
    - [ ] Bank Account required
    - [ ] If scheduled: requestedDate >= today
    - [ ] If personal: personalCategory required
  - [ ] Real-time: as user types, calculate fee + total (no API call until save)
  - [ ] Error handling: inline messages + toast notifications

**Deliverable:** Payment management pages with full CRUD

---

### Task 17: Payment Batch Pages

**Complexity:** High | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture + REQ-TCM-004**

- [ ] Create `src/pages/PaymentBatchList.vue`:
  - [ ] Component: `CnIndexPage` + `CnDataTable`
  - [ ] Columns:
    - [ ] Batch Number (text, link to detail)
    - [ ] Payment Count (number)
    - [ ] Total Amount (currency badge)
    - [ ] Currency (badge)
    - [ ] Payment Method (badge)
    - [ ] Status (badge: draft, approved, submitted, settled, failed)
    - [ ] Created Date (date)
    - [ ] Approver (name, if approved)
  - [ ] Filters: status, date range, currency, method
  - [ ] Add button: "+ New Batch" → PaymentBatchDetail (create mode, select payments)

- [ ] Create `src/pages/PaymentBatchDetail.vue`:
  - [ ] Component: `CnDetailPage` + custom batch table
  - [ ] View mode:
    - [ ] Batch Number (text)
    - [ ] Status (badge)
    - [ ] Created Date, Submitted Date, Settled Date (readonly)
    - [ ] **Payments Table:**
      - [ ] Columns: Amount, Payee, Method, Reference, Status
      - [ ] Click row → PaymentDetail in new tab
      - [ ] Totals row: sum of amounts, count
    - [ ] Total Amount (large, highlighted)
    - [ ] Total Fee (large)
    - [ ] Grand Total (amount + fee)
    - [ ] Approver info (name, date) if status >= approved
    - [ ] Sidebar tabs: Files, Audit Trail
  - [ ] Edit mode (if draft):
    - [ ] Multi-select payments from PaymentList (filtered: status=draft, same currency)
    - [ ] Add/Remove buttons to alter batch composition
    - [ ] Recalculate total + fee on change
  - [ ] Approval flow (if approver role):
    - [ ] If status="submitted": Show "Approve" + "Reject" buttons
    - [ ] Approve → status → "approved", trigger background job to submit to processor
    - [ ] Reject → status → "rejected", notify creator with reason field
  - [ ] Actions:
    - [ ] Edit (only if draft)
    - [ ] Download SEPA/ACH File (if approved + method allows)
    - [ ] Download Receipt (if settled)

**Deliverable:** Batch payment management pages

---

### Task 18: Reconciliation Hub

**Complexity:** High | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture, Page 5 + REQ-TCM-006, REQ-TCM-009**

- [ ] Create `src/pages/ReconciliationHub.vue`:
  - [ ] Component: custom two-column layout (no template library component)
  - [ ] Header:
    - [ ] BankAccount dropdown (select which account to reconcile)
    - [ ] Date range picker (from/to)
    - [ ] "Fetch Statement" button
    - [ ] Match status counter: "47 of 50 matched (94%)"
  - [ ] Main layout (two columns, draggable):
    - [ ] **Left Column: Bank Transactions**
      - [ ] Table: Date, Amount, Payee, Reference, Match Status (icon: ✓/⚠/✗)
      - [ ] Rows: clickable, draggable
      - [ ] Filter/search: by amount, reference, payee
      - [ ] Unmatched transactions highlighted (pale red background)
    - [ ] **Right Column: GL Entries**
      - [ ] Table: Date, Account, Debit/Credit, Reference, Match Status
      - [ ] Rows: clickable, droppable
      - [ ] Filter/search: by account, amount, reference
      - [ ] Unmatched entries highlighted
  - [ ] **Drag-Drop Matching:**
    - [ ] Drag bank transaction → drop on GL entry
    - [ ] On drop: validate (amount within tolerance, date within tolerance)
    - [ ] If valid: create Match, both items show ✓
    - [ ] If invalid: show error toast
  - [ ] **Manual Match Button** (alternative to drag-drop):
    - [ ] Select bank txn + GL entry
    - [ ] Click "Link" button
    - [ ] Same validation + match creation
  - [ ] **Unmatched Handler:**
    - [ ] Right-click unmatched bank txn → "Create Missing GL Entry"
    - [ ] Opens GL entry form: debit/credit picker, account, amount (pre-filled)
    - [ ] Save → creates GL entry, auto-matches
  - [ ] **Bulk Actions:**
    - [ ] "Match All Suggested" button: applies all high-confidence matches (>90% confidence)
  - [ ] **Approve Button:**
    - [ ] Bottom: "Approve Reconciliation" (enabled only if all matched)
    - [ ] On click: POST to API, status → "approved"
    - [ ] GL entries posted: reconciliation match records
    - [ ] Notify: "Reconciliation complete. 50 transactions matched."
  - [ ] Real-time updates: subscribe to WebSocket (or poll every 10s) for concurrent users
  - [ ] Error handling: validation messages, API errors shown as toasts

**Deliverable:** Interactive reconciliation interface

---

### Task 19: Cash Flow Forecast Display

**Complexity:** Medium | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#UI-Architecture + REQ-TCM-007**

- [ ] Create `src/pages/ForecastDetail.vue`:
  - [ ] Component: `CnDetailPage` + `CnChartWidget`
  - [ ] Header:
    - [ ] Title: "90-Day Cash Flow Forecast"
    - [ ] As-of Date (when forecast was generated)
    - [ ] Baseline Position (€125,000) in large text
  - [ ] **Main Chart:**
    - [ ] ApexCharts area chart
    - [ ] X-axis: dates (every 7 days labeled)
    - [ ] Y-axis: cumulative cash position (EUR)
    - [ ] Area: blue-filled with gradient, confidence band as lighter shade (±band%)
    - [ ] Risk flag markers: red X on dates with high-severity flags
    - [ ] Tooltip: hover day → shows inflows, outflows, position, confidence%
    - [ ] Legend: "Projected Position", "Confidence Band"
    - [ ] Interactive: can zoom/pan, export as PNG
  - [ ] **Daily Details Table:**
    - [ ] Columns: Date, Inflows, Outflows, Net Position, Confidence
    - [ ] Rows: every day or every 7 days (togglable)
    - [ ] Sortable by column
    - [ ] Filterable: show only days with large movements (>€10k)
  - [ ] **Risk Flags Section:**
    - [ ] List of all flags (sortable by severity)
    - [ ] Per flag:
      - [ ] Date, Severity (icon), Message, Recommendation
      - [ ] Collapsible (can expand for full details)
    - [ ] High-severity flags: sticky at top
  - [ ] **Sidebar:**
    - [ ] Forecast Inputs (read-only):
      - [ ] Projection Period: 90 days
      - [ ] Generation Method: rolling_average / trend_analysis / hybrid
      - [ ] Last Updated: timestamp
    - [ ] Notes: internal notes on forecast assumptions
  - [ ] **Export Button:**
    - [ ] Downloads PDF: chart + narrative + recommendations
    - [ ] Suitable for board presentation
  - [ ] Real-time: updates automatically every 5min (polls latest forecast)

**Deliverable:** Forecast visualization + details page

---

### Task 20: Settings Page

**Complexity:** Medium | **Owner:** Frontend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-005**

- [ ] Create `src/pages/Settings.vue`:
  - [ ] Component: `CnSettingsSection` + `CnVersionInfoCard` (MUST be first)
  - [ ] Sections:
    - [ ] **Version & Health:**
      - [ ] `CnVersionInfoCard`: app version, last update, OpenRegister status (connected/error)
    - [ ] **Payment Methods:**
      - [ ] `CnRegisterMapping`: manage PaymentMethod objects
      - [ ] Table: Method Type, Display Name, Processor, Fee, Enabled toggle
      - [ ] Per method: edit configuration (API keys, fee structure, limits)
      - [ ] Sensitive data (API keys): shown only once, then masked
    - [ ] **Reconciliation Rules:**
      - [ ] Table: Rule Name, Bank Account, Priority, Success Rate, Active toggle
      - [ ] Add rule: `CnFormDialog` (ruleName, bankAccount, matchCriteria)
      - [ ] Edit rule: update criteria, priority
    - [ ] **FX Rates:**
      - [ ] Base currency (default EUR)
      - [ ] Fallback rate source (ECB or manual)
      - [ ] Manual rate table (for testing/override): from currency, to currency, rate, valid date
    - [ ] **Notifications:**
      - [ ] Toggle: payment reminders (1 day before scheduled)
      - [ ] Toggle: failed payment alerts (to creator + approver)
      - [ ] Toggle: reconciliation alerts (when discrepancies found)
      - [ ] Email notification opt-in
    - [ ] **Data Retention:**
      - [ ] Archive old payments (default: after 1 year)
      - [ ] Purge personal expense records (GDPR: default after 3 years)
      - [ ] Manual export: download all payment data (CSV)
  - [ ] Load settings: `GET /api/settings` on mount
  - [ ] Save settings: `POST /api/settings` on change
  - [ ] Re-import button: `POST /api/settings/load` (force refresh from OpenRegister)
  - [ ] Success toast: "Settings saved"

**Deliverable:** Admin settings page

---

## Phase 3: Integration & Testing (Weeks 3–4)

### Task 21: API Integration Tests

**Complexity:** Medium | **Owner:** QA Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-* (all)**

- [ ] Create Postman/Newman collection: `tests/integration/treasury.postman.json`
  - [ ] Environment setup: base URL, auth token, test data
  - [ ] Test suites per endpoint group:
    - [ ] Bank Accounts (GET list, POST create, GET detail, PUT update, DELETE soft-delete)
    - [ ] Payments (GET list, POST create, GET detail, PUT update, DELETE, POST schedule)
    - [ ] Payment Batches (GET list, POST create, POST approve, POST execute)
    - [ ] Reconciliation (GET rules, POST create rule, POST match, POST approve)
    - [ ] Forecasts (GET latest, POST generate)
  - [ ] Tests per endpoint:
    - [ ] Happy path (valid input, expected response)
    - [ ] Validation failures (invalid input, error messages)
    - [ ] Auth failures (missing token, insufficient role)
    - [ ] Edge cases (boundary values, empty lists)
  - [ ] Assertions: response status, body structure, header values
- [ ] Run via CI: `newman run tests/integration/treasury.postman.json`
- [ ] Coverage target: ≥90% of endpoints tested

**Deliverable:** Postman collection, all endpoints tested

---

### Task 22: Browser Tests (Playwright)

**Complexity:** High | **Owner:** QA Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#User-Journeys + specs.md (GIVEN/WHEN/THEN)**

- [ ] Create Playwright test file: `tests/browser/treasury.spec.ts`
  - [ ] Test suites per user journey:
    - [ ] **Journey 1: Bank Reconciliation**
      - [ ] GIVEN: BankAccount with synced transactions
      - [ ] WHEN: CFO opens Reconciliation Hub
      - [ ] THEN: bank transactions + GL entries displayed
      - [ ] AND: drag-drop matching works
      - [ ] AND: approve completes reconciliation
    - [ ] **Journey 2: Batch Payment**
      - [ ] GIVEN: 5 draft supplier payments
      - [ ] WHEN: AP creates batch
      - [ ] THEN: payments grouped, total calculated
      - [ ] AND: CFO approves
      - [ ] AND: submission to processor succeeds
    - [ ] **Journey 3: Cash Flow Forecast**
      - [ ] GIVEN: GL entries + scheduled payments
      - [ ] WHEN: dashboard loads
      - [ ] THEN: forecast chart displays with risk flags
      - [ ] AND: clicking flag shows details
    - [ ] **Journey 4: Personal Expense**
      - [ ] GIVEN: transaction from bank statement
      - [ ] WHEN: owner marks as personal
      - [ ] THEN: GL adjustment posted
      - [ ] AND: personal expense report available
  - [ ] Test helpers:
    - [ ] `createTestBankAccount()` — fixture for bank account
    - [ ] `createTestPayment()` — fixture for payment
    - [ ] `loginAs(role)` — authenticate as specific role
  - [ ] Run via CI: `playwright test tests/browser/treasury.spec.ts`
  - [ ] Screenshots on failure: stored in `tests/browser/screenshots/`
  - [ ] Coverage target: ≥80% of user journeys

**Deliverable:** Playwright test suite, all journeys tested

---

### Task 23: PHPUnit Tests (Backend Services)

**Complexity:** Medium | **Owner:** Backend Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-* (all)**

- [ ] Create test files (Unit):
  - [ ] `tests/Unit/Service/PaymentServiceTest.php` — 20+ test methods
  - [ ] `tests/Unit/Service/BankAccountServiceTest.php` — 15+ test methods
  - [ ] `tests/Unit/Service/ReconciliationServiceTest.php` — 15+ test methods
  - [ ] `tests/Unit/Service/CashFlowForecastServiceTest.php` — 10+ test methods
  - [ ] `tests/Unit/Service/FxServiceTest.php` — 10+ test methods
  - [ ] `tests/Unit/Controller/PaymentControllerTest.php` — 10+ test methods
- [ ] Test coverage per service:
  - [ ] Happy path (valid input → success)
  - [ ] Validation failures (invalid input → exception)
  - [ ] Edge cases (boundary values, empty, null)
  - [ ] State guards (can only transition from draft → scheduled, etc.)
  - [ ] Integration (service calls mapper, injects dependencies correctly)
- [ ] Run via CI: `composer test:unit`
- [ ] Coverage target: ≥90% (measured via PHPUnit coverage report)
- [ ] All tests must pass: `composer check:strict`

**Deliverable:** PHPUnit test suite, 100+ tests, ≥90% coverage

---

### Task 24: Documentation & User Guides

**Complexity:** Low | **Owner:** Product/Docs Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md**

- [ ] Create user guides in `docs/`:
  - [ ] `docs/bank-accounts.md` — register, sync, manage accounts (with screenshots)
  - [ ] `docs/payments.md` — create, schedule, approve, track payments (with screenshots)
  - [ ] `docs/payment-batches.md` — batch creation, approval, settlement (with screenshots)
  - [ ] `docs/reconciliation.md` — reconcile bank statements, handle discrepancies
  - [ ] `docs/forecasting.md` — understand cash flow projections, risk flags
  - [ ] `docs/multi-currency.md` — foreign accounts, FX rates, gain/loss
  - [ ] `docs/settings.md` — configure payment methods, reconciliation rules, notifications
  - [ ] `docs/troubleshooting.md` — common issues (sync failures, reconciliation errors)
- [ ] Format: Markdown with screenshots (from running app)
- [ ] Language: English (primary) + Dutch (recommended)
- [ ] Update when behavior changes (future maintenance)

**Deliverable:** User documentation with screenshots

---

### Task 25: Deduplication Verification

**Complexity:** Low | **Owner:** Architect  
**@spec openspec/changes/treasury-cash-management-other-t2/design.md#Reuse-Analysis**

- [ ] Verify no overlap with:
  - [ ] `ObjectService` (CRUD) — Treasury uses it ✓ (no duplication)
  - [ ] `RegisterService` (schema mgmt) — Treasury uses it ✓
  - [ ] `AuditTrailService` (change tracking) — Treasury uses it ✓
  - [ ] `NotificationService` (alerts) — Treasury uses it ✓
  - [ ] `FileService` (documents) — Treasury uses it ✓
  - [ ] `GeneralLedgerService` (GL posting) — Treasury depends on it ✓
  - [ ] Payment processing libraries (`Stripe`, `Mollie`, `Wise`) — new integrations, no duplication
  - [ ] @conduction/nextcloud-vue components — Treasury uses standard components (CnIndexPage, CnDetailPage, CnDashboardPage)
- [ ] Document findings in task completion

**Deliverable:** Deduplication verification completed, no overlaps found

---

## Phase 4: Deployment & Launch (Week 4)

### Task 26: Deployment Preparation

**Complexity:** Medium | **Owner:** DevOps/Release Lead  
**@spec openspec/changes/treasury-cash-management-other-t2/proposal.md#Deliverables**

- [ ] Database migrations:
  - [ ] All migrations in `lib/Migration/` follow repair step pattern
  - [ ] `composer migrate` executes all steps
  - [ ] Rollback tested (can revert to previous version)
- [ ] Feature flags (if any):
  - [ ] Config: `OCP\IAppConfig` with `treasury-cash-management-enabled` (default: false for gradual rollout)
- [ ] Configuration defaults:
  - [ ] `lib/AppInfo/defaults.json` — default settings (payment method fees, reconciliation thresholds)
- [ ] Health check:
  - [ ] `GET /api/health` returns 200 if OpenRegister connected
  - [ ] Metrics: `GET /api/metrics` exposes Prometheus metrics
- [ ] Nextcloud app metadata:
  - [ ] `appinfo/info.xml` — version, description, licenses, authors
  - [ ] Screenshots: 3–5 from running app (dashboard, payments, reconciliation)
  - [ ] Changelog: `CHANGELOG.md` with version history
- [ ] Documentation:
  - [ ] README.md — what is Treasury & Cash Management, installation, configuration
  - [ ] ARCHITECTURE.md — system design, data model, key services
  - [ ] API.md — endpoint reference (auto-generated from controllers)

**Deliverable:** Deployment-ready codebase

---

### Task 27: Launch & Monitoring

**Complexity:** Low | **Owner:** DevOps/Product Lead  

- [ ] Launch checklist:
  - [ ] All tests passing: `composer check:strict`
  - [ ] API documented: `docs/api.md` (or Swagger/OpenAPI spec)
  - [ ] User guide published: `docs/` directory
  - [ ] Screenshots taken: from running app (all key pages)
  - [ ] Community communication: changelog, migration guide
- [ ] Monitoring setup:
  - [ ] Prometheus metrics: `GET /api/metrics` (for alerting)
  - [ ] Log aggregation: all app logs include `@spec` tags (for traceability)
  - [ ] Alerting rules: page admin if bank sync fails 3x in a row
- [ ] Rollout strategy:
  - [ ] Phase 1: Early adopters (feature flag: enable for specific orgs)
  - [ ] Phase 2: Gradual rollout (25%, 50%, 100% of Nextcloud instances)
  - [ ] Phase 3: General availability (feature flag removed)

**Deliverable:** App launched, monitored, documented

---

## Success Criteria

All tasks completed when:

✅ **Backend:**
- [ ] All 11 backend services implemented + tested (≥90% unit test coverage)
- [ ] All API endpoints functional + API tests passing
- [ ] All background jobs scheduled + verified
- [ ] Webhook handlers for processor events working
- [ ] Database migrations reversible

✅ **Frontend:**
- [ ] All 8 Vue pages functional + interactive
- [ ] All forms validating in real-time
- [ ] All charts rendering (ApexCharts)
- [ ] All components using @conduction/nextcloud-vue
- [ ] Responsive design (320px to 1920px)

✅ **Integration:**
- [ ] ≥90% API endpoints tested (Postman)
- [ ] ≥80% user journeys tested (Playwright)
- [ ] All 5 user journeys verified end-to-end
- [ ] Multi-currency flows tested
- [ ] Reconciliation rules tested

✅ **Quality:**
- [ ] Zero undefined behavior / error cases
- [ ] All logs include `@spec` tags for traceability
- [ ] Zero security vulnerabilities (ADR-005 compliance)
- [ ] All UI follows NL Design System (ADR-010)
- [ ] All code follows ADR-003 (backend) + ADR-004 (frontend)

✅ **Documentation:**
- [ ] User guides complete (5+ pages with screenshots)
- [ ] API documented (Swagger or manual)
- [ ] Architecture documented (ARCHITECTURE.md)
- [ ] Changelog updated (version + breaking changes)

---

**Estimated Timeline:**
- Phase 1 (Backend): Weeks 1–3 (11 tasks, parallel)
- Phase 2 (Frontend): Weeks 2–3 (9 tasks, parallel after Phase 1 partial completion)
- Phase 3 (Testing): Week 3–4 (5 tasks)
- Phase 4 (Deployment): Week 4 (2 tasks)
- **Total: 4 weeks** (with parallel work)

---

**Next Steps:**
1. Backend team: Begin Task 1 (OpenRegister schema registration)
2. Frontend team: Begin Task 12 (store setup) once backend schemas registered
3. QA team: Begin Task 21 (prepare Postman collection) in parallel
4. Docs team: Begin Task 24 (documentation stubs) after design finalized

---

**Deduplication Check:** ✅ Completed
- No overlap with ObjectService, RegisterService, AuditTrailService, NotificationService
- No custom implementations of provided @conduction/nextcloud-vue components
- Treasury-specific services (PaymentService, BankSyncService, ReconciliationService) are domain-specific
- No redundancy with core accounting module (GL, AP, AR)
