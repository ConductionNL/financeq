# Tasks: Treasury & Cash Management — Shillinq

**Version:** 1.0  
**Spec ID:** treasury-cash-management-other-t3  
**Owner:** Specter (Shillinq Product)  

## Deduplication Check

**Status:** COMPLETE ✓

### Findings

| Service/Component | Purpose | Reuse Decision | Rationale |
|------------------|---------|----------------|-----------|
| ObjectService | CRUD for BankAccount, BankTransaction, etc. | **USE** | Standard entity CRUD, no custom logic needed. Schema-driven validation via register. |
| SchemaService | Validate entities against schemas | **USE** | All entities defined in register.json, schemas auto-validated. |
| ImportService | Import bank statement CSV | **USE FOR FRAMEWORK** | Core uses ImportService framework, but custom bank CSV parser needed (bank-specific formats). |
| ExportService | Export reconciliation reports | **USE** | Standard CSV/PDF export, no custom logic. |
| AuditTrailService | Log all changes (auto) | **USE** | Automatic via ADR-001; all treasury objects audited by default. |
| AuthorizationService | Role-based payment approval | **USE** | RBAC for treasurer/approver roles via PropertyRbacHandler. |
| FileService | Attach bank statements, documents | **USE** | Standard file attachment to BankAccount/BankTransaction objects. |
| WebhookService | Receive bank notifications | **USE** | Standard webhook creation/registration for Plaid, payment processor callbacks. |
| NotificationService | Alerts, payment status updates | **USE** | Standard Nextcloud notifications for low balance, sync errors, payment approvals. |
| TaskService | Create follow-up tasks for unmatched items | **USE** | Standard task creation (create tasks for manual reconciliation). |
| CnIndexPage, CnDetailPage | List/detail UI | **USE** | Standard @conduction/nextcloud-vue components for bank accounts, payments, transactions. |
| CnFormDialog | Create/edit forms | **USE** | Schema-driven form generation for payment creation (no custom form logic). |
| CnDashboardPage, CnChartWidget | Dashboard and charts | **USE** | GridStack dashboard for Treasury (balance cards, transaction list, forecast chart). |
| createObjectStore | Pinia store creation | **USE** | Standard store pattern for BankAccount, BankTransaction, ScheduledPayment. |

### Custom Business Logic Required (NOT in OpenRegister/nextcloud-vue)

1. **Plaid Link Integration**
   - `PlaidService`: exchange OAuth token for account sync
   - Manage account lifecycle (add, remove, rotate Plaid token)
   - Resolve Plaid API rate limits

2. **Bank Transaction Import**
   - Custom bank CSV/OFX parsers (ABN AMRO, Rabobank, others)
   - Duplicate detection (transaction already imported)
   - Foreign amount handling (multi-currency)

3. **Auto-Matching Engine**
   - ReconciliationRuleEvaluator: evaluate rules against transactions
   - Fuzzy string matching for payee names
   - Regex evaluation with ReDoS protection

4. **Payment Submission**
   - SEPA XML generator (ISO 20022 pain.001.003.02)
   - ACH file format generator (US, if applicable)
   - Bank API client (Plaid, Adyen, stripe, etc.)
   - Payment retry logic with exponential backoff

5. **Cash Flow Forecast Model**
   - Linear regression implementation (or use npm package)
   - Seasonality adjustment
   - Confidence calculation from RMSE
   - Historical variance tracking

6. **Payment Approval Workflow**
   - Threshold-based routing to approvers
   - Dual-control enforcement (initiate != approve)
   - SEPA/ACH bank integration for submission

7. **Reconciliation Rules Management**
   - Rule priority ordering
   - Regex input validation (ReDoS prevention)
   - Test rule against historical transactions

### Conclusion

**No overlap found with existing OpenRegister services.** All custom logic is domain-specific (bank APIs, matching algorithms, SEPA formatting) and justified. Framework used pervasively (ObjectService, AuthorizationService, AuditTrailService, etc.); custom code limited to treasury-specific business logic.

---

## Phase 1: MVP (Core Visibility + Simple Payments) — 6 weeks

### Backend Tasks

#### Task 1.1: Set up Bank Account entity and Plaid integration

- [ ] Create `src/Schema/BankAccount.php` or register in `lib/Settings/shillinq_register.json`
  - IBAN (required, unique within org)
  - BIC (nullable)
  - Currency (ISO 4217, default EUR)
  - Account type (checking, savings, investment)
  - Plaid account ID (encrypted)
  - Sync enabled/status
  - Current + available balance
  - Last sync time
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-001, #req-001-002**

- [ ] Create `lib/Service/PlaidService.php`
  - `exchangePublicToken()` → get Plaid access token (called after Plaid Link success)
  - `syncTransactions(BankAccount $account)` → fetch last 90 days from Plaid API
  - Encrypt and store Plaid token (use `IAppConfig::sensitive()` or encryption service)
  - Handle Plaid API errors (rate limits, auth failures)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-001, #req-001-003**

- [ ] Create `lib/Service/BankAccountService.php` (Controller → Service → Mapper pattern)
  - `addAccount(array $plaidData)` → create BankAccount + store encrypted token
  - `removeAccount(BankAccount $account)` → soft delete + notify
  - `updateBalance(BankAccount $account, decimal $balance, decimal $available)` → called by sync job
  - Duplicate detection by IBAN + accountType
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-004, #req-001-005**

- [ ] Create `src/Controller/BankAccountController.php` (thin, 10 lines/method)
  - `POST /api/bank-accounts` → BankAccountService::addAccount()
  - `GET /api/bank-accounts` → ObjectService::findAll() with filters
  - `DELETE /api/bank-accounts/{id}` → BankAccountService::removeAccount()
  - `POST /api/bank-accounts/{id}/sync` → trigger manual sync (background job)

- [ ] Create test: `tests/Unit/Service/PlaidServiceTest.php`
  - Mock Plaid API
  - Test token exchange
  - Test transaction fetch
  - Test error handling (invalid token, rate limit)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

- [ ] Database migration (or register definition)
  - BankAccount schema with encrypted Plaid token column
  - Indexes: IBAN (unique within org), syncStatus

---

#### Task 1.2: Bank Transaction import and storage

- [ ] Create `src/Schema/BankTransaction.php` or extend register
  - Bank account reference (relation)
  - Transaction date + post date
  - Amount (decimal, currency)
  - Description, counterparty name/IBAN/BIC
  - Reference number + payment reference
  - Transaction type (credit, debit, fee, interest, transfer)
  - Reconciliation status (unmatched, matched, excluded)
  - Matched invoice/expense/payment relations
  - Match rule name, manual match flag
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#entity-transaction**

- [ ] Create `lib/Service/BankTransactionService.php`
  - `importTransactions(BankAccount $account, array $plaidTransactions)` → create BankTransaction objects
  - Duplicate detection: check if transaction already exists (date + amount + counterparty)
  - Currency conversion: if account currency != transaction currency, store original + converted
  - Multi-currency handling (store amount + currency pair)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-003**

- [ ] Create `lib/Job/BankTransactionSyncJob.php`
  - Background job: fetch latest transactions from Plaid for all active accounts
  - Run daily at 6 AM UTC (scheduled via `IJobList`)
  - For each account: PlaidService::syncTransactions() → BankTransactionService::importTransactions()
  - Error handling: log errors, notify admin if sync fails
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-003**

- [ ] Test: `tests/Unit/Service/BankTransactionServiceTest.php`
  - Test duplicate detection (same date + amount + counterparty)
  - Test multi-currency handling
  - Test import with missing fields

---

#### Task 1.3: Payment scheduling entity and service

- [ ] Create `src/Schema/ScheduledPayment.php` or register
  - Payment type (bill_pay, immediate_transfer, recurring, internal_transfer)
  - Status (draft, scheduled, submitted, processing, completed, failed, cancelled)
  - From/to accounts (from BankAccount, to Payee)
  - Payment method (SEPA, ACH, iDEAL, Giropay, card, internal_transfer)
  - Amount + currency
  - Scheduled + execution dates
  - Description + reference number
  - Linked expenses + invoices (array of relations)
  - Approval chain reference
  - Approval status + approver + approval date
  - Created by + timestamps
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#entity-scheduledpayment**

- [ ] Create `lib/Service/PaymentService.php`
  - `createPayment(array $data)` → create ScheduledPayment (draft status)
  - `submitForApproval(ScheduledPayment $payment)` → route to approval chain
  - `approve(ScheduledPayment $payment, User $approver)` → move to scheduled
  - `reject(ScheduledPayment $payment, User $approver, string $reason)` → cancel + notify
  - Validation: amount > 0, sufficient balance, scheduled date is future (or immediate), required fields
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-003-001, #req-003-002**

- [ ] Create approval chain resolution
  - Load ApprovalChain for payment amount threshold
  - Route to appropriate approvers (treasurer < €5k, CFO > €5k, etc.)
  - Configurable thresholds via Settings
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-003-003, #req-003-004**

- [ ] Create `lib/Job/PaymentExecutionJob.php`
  - Background job: run daily, fetch ScheduledPayments with scheduledDate = today
  - For each payment: validate balance, format payment (SEPA/ACH), submit to bank API
  - Update ScheduledPayment.status = submitted, executionDate = today
  - Log bank confirmation ID in audit trail
  - Handle errors (insufficient funds) → set status = failed, notify manager
  - Retry logic: exponential backoff (1 min, 5 min, 15 min)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-003-005, #req-003-006, #req-003-007**

- [ ] Test: `tests/Unit/Service/PaymentServiceTest.php`
  - Test payment creation (valid amounts, dates)
  - Test approval routing (different thresholds)
  - Test balance validation
  - Test payment submission (mock bank API)

---

#### Task 1.4: SEPA payment formatting

- [ ] Create `lib/Service/SepaPaymentFormatter.php`
  - `format(ScheduledPayment $payment)` → return ISO 20022 pain.001.003.02 XML
  - Generate XML per SEPA standards:
    - Payment initiation header (MsgId, CreDtTm)
    - Group information (GrpInfo)
    - Payment information (PmtInf) per account
    - Credit transfer transaction details (CdtTrfTxInf)
  - IBAN validation: checksum (mod-97)
  - BIC resolution (if not provided, lookup from bank)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-003-005**

- [ ] Create `lib/Service/BankPaymentApiClient.php`
  - Abstraction for payment submission (Plaid Payments API, open banking APIs, or bank-specific APIs)
  - `submitPayment(string $paymentXml, BankAccount $fromAccount)` → returns confirmation ID
  - HTTP client with timeout, retry logic
  - Error handling: decode bank error responses, log for diagnosis
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-003-005, #req-003-006**

- [ ] Test: `tests/Unit/Service/SepaPaymentFormatterTest.php`
  - Test SEPA XML generation (sample payment)
  - Test IBAN checksum validation
  - Test XML schema compliance (validate against pain.001 XSD if available)

---

#### Task 1.5: Dashboard and API endpoints

- [ ] Create `src/Controller/TreasuryDashboardController.php`
  - `GET /api/treasury/dashboard` → summary data
    - Total liquidity (sum of all account balances)
    - By-account breakdown (name, balance, last sync, status)
    - Recent transaction list (last 10, with filters)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-002-001, #req-002-004, #req-002-005**

- [ ] Create `src/Controller/TransactionController.php`
  - `GET /api/bank-transactions` → list with pagination, filtering (account, date range, counterparty)
  - `GET /api/bank-transactions/{id}` → detail view
  - No POST/PUT/DELETE (transactions are immutable once imported)

- [ ] Optimize queries
  - Index BankTransaction by bankAccountId, transactionDate, reconciliationStatus
  - Lazy load relations (bankAccount, matchedInvoice) to avoid N+1
  - Cache dashboard summary (5-min TTL)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-nfr-001, #req-nfr-002**

---

### Frontend Tasks

#### Task 1.6: UI components and pages (Vue)

- [ ] Create `src/components/BankAccountList.vue`
  - Use CnIndexPage + CnDataTable for list
  - Columns: account name, IBAN (last 4), balance, last sync, status badge
  - Add button → trigger Plaid Link modal
  - Row actions: view detail, remove account
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-004**

- [ ] Create `src/components/PlaidLinkModal.vue`
  - Embed Plaid Link SDK (JavaScript)
  - Handle onSuccess callback → POST /api/bank-accounts
  - Error handling (user cancelled, auth failed)
  - Show spinner during account import
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-001-005, #adr-004**

- [ ] Create `src/pages/TreasuryDashboard.vue`
  - Use CnDashboardPage or custom layout
  - Top section: total liquidity card, month-end projection card, cash runway card
  - Middle: multi-account balance chart (stacked area, last 30 days)
  - Bottom: recent transactions table (CnDataTable)
  - Filters: account dropdown, date range picker
  - Real-time updates (poll API every 5 min or use WebSocket if available)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-002-002, #req-002-003**

- [ ] Create `src/pages/BankAccountDetail.vue`
  - CnDetailPage layout
  - Balance cards (current, available)
  - Transaction list (paginated, sortable)
  - Last sync time + status
  - Action: "Sync Now" button (trigger manual sync)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-004**

- [ ] Create `src/pages/PaymentScheduler.vue`
  - CnFormDialog or custom form for creating payment
  - Fields: from account, to payee (autocomplete), amount, scheduled date, description, linked expenses
  - Submit → POST /api/scheduled-payments
  - Draft/submitted payment list below
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-004**

- [ ] Router setup
  - Routes: `/`, `/bank-accounts`, `/bank-accounts/:id`, `/payments`, `/settings`
  - Base path: `/index.php/apps/shillinq/`
  - Use router history mode with catch-all `*` route
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-004**

- [ ] Store (Pinia)
  - `src/store/modules/bankAccounts.ts` → createObjectStore('bankAccounts', 'BankAccount', 'shillinq')
  - `src/store/modules/bankTransactions.ts` → createObjectStore('bankTransactions', 'BankTransaction', 'shillinq')
  - `src/store/modules/payments.ts` → createObjectStore('payments', 'ScheduledPayment', 'shillinq')
  - Add store plugins: auditTrails, files, relations
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-004**

---

#### Task 1.7: Styling and accessibility

- [ ] Apply NL Design System tokens
  - Use CSS custom properties (--nldesign-primary, --nldesign-accent, etc.)
  - Nextcloud component library colors
  - Responsive design (320px to 1920px)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-010**

- [ ] Accessibility (WCAG AA)
  - Keyboard navigation (tab, arrow keys, enter)
  - Form labels + aria-labels
  - Color not sole conveyor (use icons + text for status)
  - Alt text on charts, images
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-010**

---

#### Task 1.8: Translation (i18n)

- [ ] Create translation files
  - `src/translations/nl.json` (Dutch)
  - `src/translations/en.json` (English)
  - Translate all user-facing strings (buttons, labels, messages, errors)
  - No hardcoded text in Vue components (use `t(appName, 'key')`)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-007, #adr-010**

---

### Integration Tasks

#### Task 1.9: Register template and schema import

- [ ] Create `lib/Settings/shillinq_register.json`
  - Define schemas (OpenAPI 3.0 format):
    - BankAccount
    - BankTransaction
    - ScheduledPayment
    - Bank (reference)
    - Payee (reference)
  - Include seed data (3-5 example objects per entity, Dutch values)
  - Mark as `x-openregister.type: "application"`
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#seed-data**

- [ ] Create repair step `lib/RepairSteps/ImportTreasuryRegister.php`
  - Implements `IRepairStep`
  - Called on app install/update
  - Uses `ConfigurationService::importFromApp()` to load register template
  - Idempotent (re-run doesn't create duplicates)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-001**

---

#### Task 1.10: Database migration

- [ ] Create migration file(s)
  - BankAccount table (if not using pure OpenRegister)
  - BankTransaction table
  - ScheduledPayment table
  - ReconciliationRule table (for Phase 2, but define now)
  - Indexes: IBAN, account GUID, sync status, reconciliation status
  - Encrypted columns for Plaid token

---

### Testing Tasks

#### Task 1.11: Unit tests

- [ ] Test PlaidService (mock Plaid API)
- [ ] Test BankAccountService (account creation, duplicate detection)
- [ ] Test BankTransactionService (import, deduplication)
- [ ] Test PaymentService (creation, approval routing, validation)
- [ ] Test SepaPaymentFormatter (XML generation, IBAN checksum)
- [ ] Coverage target: ≥80% for Service classes
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

#### Task 1.12: Integration tests

- [ ] Newman/Postman collection: `tests/integration/treasury.postman_collection.json`
  - Test flow:
    1. POST /api/bank-accounts (add account)
    2. GET /api/bank-accounts (list)
    3. GET /api/treasury/dashboard (summary)
    4. POST /api/scheduled-payments (create payment)
    5. POST /api/scheduled-payments/{id}/approve (approval)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

#### Task 1.13: Browser tests (Playwright)

- [ ] Test scenario: Add Bank Account
  - User navigates to Bank Accounts page
  - Clicks "Add Account"
  - Plaid modal opens
  - User selects bank and authenticates (mock)
  - Account appears in list
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

- [ ] Test scenario: View Dashboard
  - Dashboard loads in <2 seconds
  - Balance cards display correct totals
  - Recent transactions list is populated
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

- [ ] Test scenario: Schedule Payment
  - User creates payment (form filled)
  - Payment appears in draft list
  - User submits for approval
  - Status changes to pending_approval
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-008**

---

### Documentation Tasks

#### Task 1.14: API documentation

- [ ] Document OpenAPI 3.0 spec for all endpoints
  - Paths, methods, request/response schemas
  - Error responses (400, 401, 403, 404, 500)
  - Examples
  - Authentication (Nextcloud user)

#### Task 1.15: User documentation

- [ ] Create `docs/bank-accounts.md` (with screenshots from running app)
  - How to add a bank account
  - How to view transactions
  - Sync troubleshooting

- [ ] Create `docs/payment-scheduling.md`
  - How to schedule a payment
  - Approval workflow
  - Execution and confirmation

---

## Phase 2: Automation + Forecasting — 6 weeks

### Backend Tasks

#### Task 2.1: Reconciliation rule engine

- [ ] Create `src/Schema/ReconciliationRule.php` or register
  - Name, description, priority
  - Matching criteria (amount, description, counterparty pattern)
  - Match target (invoice, expense, payment)
  - Auto-apply flag
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#entity-reconciliationrule**

- [ ] Create `lib/Service/ReconciliationRuleEvaluator.php`
  - `evaluateRules(BankTransaction $txn)` → iterate active rules (sorted by priority), check if criteria match
  - Criteria matching:
    - Amount: exact or range
    - Description: substring (case-insensitive)
    - Counterparty: regex pattern with ReDoS validation
  - Return first matching rule + target object
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-006-001, #req-006-004**

- [ ] Create `lib/Service/FuzzyMatchService.php`
  - Levenshtein distance for payee name matching
  - For ambiguous matches, return top 3 candidates (confidence > threshold)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-006-002**

- [ ] Create `lib/Service/ReconciliationService.php`
  - `autoReconcile()` → fetch all unmatched transactions, apply rules
  - `matchTransaction(BankTransaction $txn, object $target)` → set matched relation
  - `excludeTransaction(BankTransaction $txn, string $reason)` → mark as excluded
  - Generate reconciliation report (matched count, %, variance)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-005-001, #req-005-005, #req-005-006**

- [ ] Create `lib/Job/ReconciliationJob.php`
  - Run daily at 6 AM UTC
  - For all unmatched BankTransactions: apply rules, update reconciliation status
  - Generate report, notify manager
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-005-001**

- [ ] Test: `tests/Unit/Service/ReconciliationRuleEvaluatorTest.php`
  - Test amount matching (exact, range)
  - Test description keyword matching (case-insensitive)
  - Test regex evaluation (prevent ReDoS)
  - Test rule priority order

---

#### Task 2.2: Cash flow forecast model

- [ ] Create `src/Schema/CashFlowForecast.php` or register
  - Forecast period (13 weeks)
  - Weekly forecasts (opening balance, inflows, outflows, closing balance, confidence)
  - Model type (linear regression, seasonality, ML)
  - Accuracy metrics (1-week, 3-week, 13-week RMSE)
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#entity-cashflowforecast**

- [ ] Create `lib/Service/CashFlowForecastService.php`
  - `generateForecast(BankAccount $account)` → return CashFlowForecast object
  - Data collection:
    - Last 90 days of transactions (group by week, calculate daily avg outflow)
    - Unpaid invoices (sum by due date)
    - Unpaid bills/expenses (sum by due date)
    - Scheduled payments (sum by scheduled date)
  - Forecast calculation:
    - Opening balance = current account balance
    - Weekly inflows = sum of invoices due that week
    - Weekly outflows = bills due + historical daily avg * days in week + scheduled payments
    - Closing balance = opening + inflows - outflows
    - Confidence = (1 - RMSE / avg_daily_amount) * 100
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-007-002, #req-007-003, #req-007-004, #req-007-005**

- [ ] Implement linear regression model (use npm library like `regression` or `ml-regression`)
  - Load historical transaction amounts
  - Fit line to predict future daily outflows
  - Calculate RMSE for accuracy

- [ ] Create `lib/Job/CashFlowForecastJob.php`
  - Run daily at 6 AM UTC
  - For each BankAccount: generate 13-week forecast
  - Store as CashFlowForecast object
  - Notify if any week shows negative balance (alert)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-007-001**

- [ ] Test: `tests/Unit/Service/CashFlowForecastServiceTest.php`
  - Test forecast calculation with mock invoice/bill data
  - Test confidence score calculation
  - Test negative balance detection

---

#### Task 2.3: iDEAL payment processing (basic)

- [ ] Research iDEAL payment providers (Adyen, Mollie, etc.)
  - For MVP, can start with Mollie (good Dutch coverage)
  - OR use Adyen for broader support

- [ ] Create `lib/Service/IdealPaymentService.php`
  - `initiatePayment(Payment $payment)` → create iDEAL payment request, return redirect URL
  - Handle webhook callback from payment provider (payment completed)
  - Update Payment.status = "completed"
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-008-001**

- [ ] Create webhook controller `src/Controller/PaymentWebhookController.php`
  - `POST /api/webhooks/ideal` (or provider-specific)
  - Verify webhook signature (provider-specific)
  - Fetch payment provider status
  - Update local Payment object
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-008-001**

- [ ] Note: Full iDEAL + card support is Phase 3; Phase 2 focuses on infrastructure

---

#### Task 2.4: Customer payment preference entity

- [ ] Create `src/Schema/CustomerPaymentPreference.php` or register
  - Customer reference
  - Primary payment method (enum)
  - Mandate reference (for SEPA)
  - Card token (nullable, encrypted)
  - Subscription state + frequency
  - Next payment date
  - Last payment status + date
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#entity-customerpaymentpreference**

- [ ] Create `lib/Service/CustomerPaymentPreferenceService.php`
  - `setPaymentMethod(Customer $c, string $method)` → create/update preference
  - `enableSubscription(Customer $c, int $frequencyDays)` → set up recurring payment
  - `disableSubscription(Customer $c)` → pause subscription
  - Schedule recurring ScheduledPayments based on preference

---

### Frontend Tasks

#### Task 2.5: Reconciliation dashboard UI

- [ ] Create `src/pages/ReconciliationDashboard.vue`
  - Counts: auto-matched, pending, excluded
  - Pending transaction list (CnDataTable)
    - Columns: date, amount, description, counterparty, proposed match, actions
  - Actions per transaction: "Accept Match" / "Edit" / "Exclude" / "Manual Review"
  - Rules list (manage rules)
  - Generate reconciliation report (download PDF/CSV)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-005-001**

- [ ] Create `src/components/ReconciliationRuleForm.vue`
  - Form for creating/editing ReconciliationRule
  - Criteria inputs: amount type, keywords, pattern (regex), filters
  - Test button → preview matching transactions
  - Save/Delete actions
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-006-005, #req-006-006**

- [ ] Create `src/pages/CashFlowDashboard.vue`
  - 13-week forecast area chart
  - Summary cards: opening, inflows, outflows, closing, confidence
  - Weekly detail table (drill-down)
  - Historical variance chart (forecast vs. actual)
  - Negative balance warning (red highlight)
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-007-001**

- [ ] Update MainMenu navigation
  - Add "Reconciliation" and "Cash Flow" pages

---

#### Task 2.6: Brand configuration settings UI

- [ ] Create `src/pages/BrandingSettings.vue`
  - Logo upload
  - Color picker (primary, secondary)
  - Company name + support email
  - Preview payment page
  - Save/reset buttons
  - **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-009-001, #req-009-002, #req-009-003**

---

### Integration Tasks

#### Task 2.7: Seed data + register updates

- [ ] Update `lib/Settings/shillinq_register.json`
  - Add ReconciliationRule schema
  - Add CashFlowForecast schema
  - Add CustomerPaymentPreference schema
  - Include seed data objects (3-5 per entity)
  - **@spec openspec/changes/treasury-cash-management-other-t3/design.md#seed-data**

---

### Testing Tasks

#### Task 2.8: Reconciliation + forecast tests

- [ ] Test ReconciliationRuleEvaluator with various rule types
- [ ] Test CashFlowForecastService with mock data
- [ ] Test iDEAL webhook handling
- [ ] Integration test: create rule, apply to transactions, verify matches

---

## Phase 3: Advanced Channels + Multi-Currency — 8 weeks

### Backend Tasks

#### Task 3.1: Credit card acquiring

- [ ] Integrate payment processor (e.g., Adyen, Stripe)
- [ ] Support Visa, Mastercard, Amex
- [ ] 3D Secure (3DS) authentication
- [ ] Tokenization for recurring payments
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-008-001**

#### Task 3.2: Multi-currency conversion

- [ ] Integrate FX rate provider (ECB, Fixer.io, etc.)
- [ ] Store historical FX rates in TimeSeries table
- [ ] Convert foreign currency amounts to reporting currency
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-007-003**

#### Task 3.3: Regional payment channels (Giropay, etc.)

- [ ] Add Giropay payment support (Germany)
- [ ] Add other regional channels as needed
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md (phase 3)**

---

### Frontend Tasks

#### Task 3.4: Multi-currency UI updates

- [ ] Update dashboard to show multi-currency balances
- [ ] Currency selection on payment creation
- [ ] FX rate display + historical chart

---

### Testing Tasks

#### Task 3.5: Card payment + multi-currency tests

- [ ] Integration test: create card payment, verify 3DS flow
- [ ] Test FX conversion with mock rates
- [ ] Test multi-currency forecast

---

## Cross-Phase Tasks

### Task A: API Documentation (continuous)

- [ ] Update OpenAPI spec as endpoints are added
- [ ] Document error codes, rate limits, pagination
- [ ] Example requests/responses
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-002**

### Task B: Security Review (continuous)

- [ ] Code review: check for SQL injection, XSS, CSRF vulnerabilities
- [ ] Plaid token encryption (TLS 1.3 + AES-256)
- [ ] Payment data: SEPA/card number validation, no logging of sensitive data
- [ ] RBAC enforcement: verify role-based payment approval
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-005, #req-nfr-009, #req-nfr-010, #req-nfr-011**

### Task C: Performance Optimization (per phase)

- [ ] Dashboard query optimization (caching, indexes)
- [ ] Background job tuning (parallel processing, batch sizes)
- [ ] Frontend bundle size (lazy load components)
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#req-nfr-001, #req-nfr-002, #req-nfr-003, #req-nfr-004**

### Task D: Localization (per phase)

- [ ] Dutch (nl) + English (en) translations
- [ ] Date/currency formatting per user locale
- [ ] Error messages translated
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-007**

### Task E: User Documentation (per phase)

- [ ] Screenshots from running app
- [ ] Step-by-step guides
- [ ] Troubleshooting
- [ ] **@spec openspec/changes/treasury-cash-management-other-t3/specs.md#adr-009**

---

## Estimate Summary

| Phase | Duration | Team | Effort |
|-------|----------|------|--------|
| Phase 1 (MVP) | 6 weeks | 1 BE (PHP) + 1 FE (Vue) + 1 Product | 240 hours |
| Phase 2 (Automation) | 6 weeks | 1 BE + 1 FE + 1 Product | 240 hours |
| Phase 3 (Advanced) | 8 weeks | 1 BE + 1 FE + 1 Ops | 320 hours |
| **Total** | **20 weeks** | **3-4 people** | **800 hours** |

---

## Definition of Done (per task)

✓ Code follows ADR patterns (Controller → Service → Mapper)  
✓ @spec PHPDoc tag linking to this spec  
✓ Unit tests with ≥80% coverage (Services)  
✓ Integration test (API endpoint or workflow)  
✓ Browser test for user-facing feature  
✓ Audit trail log entry (automatic via AuditTrailService)  
✓ Error handling + user-facing messages  
✓ Translations (Dutch + English)  
✓ Documentation (docblock + user guide)  
✓ Security review (no injection, encryption, RBAC)  
✓ Performance verified (meet NFR targets)  
✓ Code review + approval (2+ reviewers)
