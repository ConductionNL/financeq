# Technical Specifications: Treasury & Cash Management — Shillinq

**Change:** treasury-cash-management-other-t2  
**Version:** 1.0  
**Date:** 2026-05-21  
**Status:** Specifications

## Overview

This document defines 60+ functional and technical requirements for Treasury & Cash Management features. Each requirement follows REQ-XXX-NNN format with GIVEN/WHEN/THEN acceptance criteria.

---

## REQ-TCM-001: Bank Account Registration

**Title:** Register new bank account with validation  
**Priority:** Critical  
**Complexity:** Medium

**Description:**
User can register a new bank account (checking, savings, credit card, virtual) with automatic IBAN validation and optional OpenRegister sync.

**Acceptance Criteria:**

```gherkin
SCENARIO: Register valid IBAN account
GIVEN I am a CFO with Admin role
WHEN I navigate to Treasury → Bank Accounts → "+ New"
AND I fill: accountHolder="Contoso B.V.", accountNumber="NL91ABNA0417164300", 
    currency="EUR", bankName="ABN AMRO", accountType="business"
AND I click "Save"
THEN the account is created with status syncStatus="pending"
AND a background job triggers to connect via OpenRegister
AND the account appears in Bank Accounts list

SCENARIO: Reject invalid IBAN
GIVEN I am creating a bank account
WHEN I enter accountNumber="XX123456789" (invalid format)
AND I click "Save"
THEN the UI shows validation error "Invalid IBAN format"
AND the form prevents submission

SCENARIO: Auto-detect bank name from BIC
GIVEN I am creating a new account
WHEN I enter BIC="ABNANL2A"
THEN the UI auto-fills bankName="ABN AMRO Bank"
```

**Technical Notes:**
- IBAN validation via `IbanValidator` (lib/Validator/)
- BIC lookup via `BankRegistry` (lib/Service/)
- OpenRegister registration: `register.shillinq_treasury`, schema `BankAccount`
- Background job: `BankAccountSyncJob` scheduled upon creation

---

## REQ-TCM-002: OpenRegister Bank Sync

**Title:** Automatic bank statement sync via OpenRegister  
**Priority:** Critical  
**Complexity:** High

**Description:**
Daily background job syncs bank statements from OpenRegister, creates Payment objects, and flags unmatched transactions for reconciliation.

**Acceptance Criteria:**

```gherkin
SCENARIO: Sync successful with new transactions
GIVEN a BankAccount with syncStatus="connected"
WHEN the daily sync job runs (e.g., 6 AM UTC)
THEN the system queries OpenRegister for new statements
AND newly synced transactions create Payment objects (status="pending")
AND existing GL entries are matched via ReconciliationRules
AND unmatched transactions are flagged (status="unmatched")
AND sync completes with status="completed" in AuditTrail

SCENARIO: Sync failure with retry
GIVEN OpenRegister is temporarily unavailable
WHEN the sync job attempts to fetch statements
THEN the job catches exception and updates syncStatus="error"
AND a retry is scheduled for 30min later (up to 3 retries)
AND a notification is sent to admin: "Bank sync failed for account NL91..."

SCENARIO: Sync with duplicate detection
GIVEN a Payment was already synced yesterday
WHEN the same transaction appears in today's statement
THEN the system detects duplicate (via amount + date + reference)
AND the duplicate is skipped (not creating a second Payment record)
```

**Technical Notes:**
- Cron job: `BankStatementSyncJob` (lib/Job/)
- OpenRegister query: `GET /register/shillinq_treasury/openbanking/{account-id}/transactions`
- Duplicate detection: hash(amount + date + reference)
- Exception handling: max 3 retries, notify admin after failure

---

## REQ-TCM-003: Payment Creation & Validation

**Title:** Create payment with full validation  
**Priority:** Critical  
**Complexity:** Medium

**Description:**
User can create a payment (outbound or inbound) with amount, payee, method, and date validation.

**Acceptance Criteria:**

```gherkin
SCENARIO: Create outbound payment to supplier
GIVEN I am an AP Clerk
WHEN I navigate to Payments → "+ New"
AND I fill:
  - amount=2500.00, currency="EUR"
  - paymentMethod="bank_transfer"
  - payee (search & select from Suppliers)
  - bankAccount (select from active accounts)
  - requestedDate="2026-05-25" (future)
  - reference="INV-2026-001"
AND I click "Create"
THEN the Payment is created with status="draft"
AND the amount is reserved in cash forecast (not posted to GL yet)
AND a form confirmation shows: "Payment created. Next step: Submit for approval"

SCENARIO: Validate payment amount > 0
GIVEN I am creating a payment
WHEN I enter amount=0 or amount=-100
THEN validation error: "Payment amount must be > 0"
AND form is locked until corrected

SCENARIO: Reject payment without payee
GIVEN I am creating a payment
WHEN I leave payee blank
AND I click "Create"
THEN validation error: "Payee is required"
AND form is locked

SCENARIO: Schedule payment for future date
GIVEN I am creating a payment
WHEN I set requestedDate="2026-06-15" (14 days in future)
AND I click "Create"
THEN Payment is created with status="scheduled"
AND a background job sets reminder for 1 day before execution
AND the payment does NOT post to GL until requestedDate
```

**Technical Notes:**
- Schema validation via `SchemaService`
- Amount validation: `amount > 0 && amount < 999999.99`
- Payee resolution: fuzzy match against Suppliers, Users, Organizations
- Date validation: `requestedDate >= today || requestedDate <= today + 180 days`
- GL post timing: manual for draft, automatic for scheduled on requestedDate

---

## REQ-TCM-004: Payment Batch Processing

**Title:** Group and submit multiple payments in batch  
**Priority:** High  
**Complexity:** Medium

**Description:**
User can select multiple payments, group into batch, review collective amount and fees, and submit for approval.

**Acceptance Criteria:**

```gherkin
SCENARIO: Create batch from draft payments
GIVEN I am an AP Clerk with 5 draft supplier payments due
WHEN I navigate to Payment Batches → "+ New Batch"
AND the system auto-populates due payments (multi-select)
AND I review: 5 payments, total €12,500, fee €0 (SEPA)
AND I click "Submit for Approval"
THEN PaymentBatch is created with:
  - batchNumber="BATCH-2026-05-W2"
  - status="submitted"
  - approver field empty (awaiting CFO)
AND a notification is sent to CFO: "5 payments in batch ready for approval"

SCENARIO: Approve and execute batch
GIVEN a PaymentBatch with status="submitted"
WHEN CFO opens batch detail
AND reviews all 5 payments and total amount
AND clicks "Approve"
THEN status → "approved"
AND a background job immediately submits payments to processor (SEPA)
AND status → "processing"
AND GL entries are posted (Debit AP Liability, Credit Cash Account)
AND notifications sent to all payees: "Your payment is processing"

SCENARIO: Batch with mixed currencies fails
GIVEN I am creating a PaymentBatch
WHEN I select payments with currencies EUR, USD, GBP
AND I click "Submit"
THEN validation error: "All payments in batch must use same currency"
AND batch creation is blocked
```

**Technical Notes:**
- Batch CRUD via `ObjectService`
- Batch approval workflow: clerk → CFO (role-based)
- Payment processor submission: call processor API (Stripe/Mollie) per payment
- GL posting: `GeneralLedgerService.postEntry()` per payment
- Notification: via `NotificationService` to users + batch approver

---

## REQ-TCM-005: Payment Method Configuration

**Title:** Configure supported payment methods and processor integration  
**Priority:** High  
**Complexity:** High

**Description:**
Admin can enable/disable payment methods, set processor credentials, define fee structure, and transaction limits.

**Acceptance Criteria:**

```gherkin
SCENARIO: Enable Stripe credit card processing
GIVEN I am an Admin
WHEN I navigate to Settings → Payment Methods
AND I find "Visa Credit Card (Stripe)" in list
AND I toggle isEnabled=true
AND I fill API credentials: apiKey="sk_live_...", webhookSecret="whsec_..."
AND I set fee: 1.5% per transaction
AND I set limits: min=€0.50, max=€50,000, max per day=€500,000
AND I click "Save"
THEN PaymentMethod is saved with all config
AND Stripe webhook is registered: POST /index.php/apps/shillinq/api/payment-method-webhook
AND next credit card payments can be processed

SCENARIO: Disable payment method in use
GIVEN isEnabled=true for "Klarna Pay Later"
WHEN I navigate to Settings and toggle isEnabled=false
AND I click "Save"
THEN isEnabled → false
AND new payments cannot use this method (blocked in form)
AND existing scheduled payments keep their method (not revoked)
AND notification to CFO: "Klarna method disabled. 2 pending payments still use this method."

SCENARIO: Fee calculation on payment
GIVEN a PaymentMethod with fee: 1.5% (percentage)
WHEN a Payment is created: amount=€10,000
THEN feeAmount = 10,000 * 1.5% = €150
AND the fee is added to total outflow (both shown to user)
AND GL posting includes fee as separate line (Debit Fee Expense, Credit Cash)
```

**Technical Notes:**
- PaymentMethod stored in OpenRegister (register: shillinq_treasury)
- Processor credentials: stored via `IAppConfig` with `sensitive=true` flag (encrypted)
- Webhook registration: done in `PaymentMethodService.enablePaymentMethod()`
- Fee calculation: `FeeCalculationService.calculateFee(amount, paymentMethod)`

---

## REQ-TCM-006: Automatic Reconciliation Rules

**Title:** Define and apply automatic reconciliation rules  
**Priority:** High  
**Complexity:** Medium

**Description:**
Admin can define rules for auto-matching bank transactions to GL entries based on amount, date, reference, and party.

**Acceptance Criteria:**

```gherkin
SCENARIO: Create supplier invoice matching rule
GIVEN I am an Admin
WHEN I navigate to Settings → Reconciliation Rules → "+ New Rule"
AND I fill:
  - ruleName="Supplier Invoice Auto-Match"
  - bankAccount=(select ABN AMRO)
  - matchCriteria:
    - amountTolerance=€1.00
    - dateTolerance=3 days
    - referencePattern="INV-\\d{4}-\\d{3}"
    - partyMatch="fuzzy"
  - priority=1 (apply first)
AND I click "Save"
THEN ReconciliationRule is created
AND the rule is activated immediately

SCENARIO: Apply rules during bank sync
GIVEN 50 bank transactions synced today
AND ReconciliationRule "Supplier Invoice Auto-Match" is active
WHEN reconciliation rules engine runs
THEN for each transaction:
  - Check if reference matches pattern "INV-..."
  - Search GL for invoice with same reference
  - If GL entry found AND amount within €1.00 AND date within 3 days:
    - Create Match record (transaction ↔ GL entry)
    - Update Payment.status="matched"
RESULT: 47 of 50 transactions auto-matched (94%)

SCENARIO: Manual match for unmatched transactions
GIVEN 3 unmatched transactions after rule application
WHEN I navigate to Reconciliation Hub
AND I select a bank transaction (€2,500) and GL entry (€2,500)
AND I click "Link"
THEN a Match record is created manually
AND transaction status → "matched"
AND I can now approve reconciliation (all 50 matched)
```

**Technical Notes:**
- ReconciliationRule stored in OpenRegister
- Rule engine: `ReconciliationService.applyRules(transactions)` iterates priority-ordered rules
- Matching algorithm: amount tolerance, date tolerance, regex reference, fuzzy party name (Levenshtein distance)
- Success rate tracked (historical for feedback to users)

---

## REQ-TCM-007: Cash Flow Forecasting

**Title:** Generate and display 90-day cash flow projections  
**Priority:** High  
**Complexity:** High

**Description:**
System auto-generates daily cash flow projections based on GL recurring entries and scheduled payments, with confidence intervals.

**Acceptance Criteria:**

```gherkin
SCENARIO: Generate baseline forecast
GIVEN today is 2026-05-21
AND BankAccount balance (as of today)=€125,000
WHEN I navigate to Dashboard → Cash Flow Forecast widget
AND the system queries:
  - Historical GL entries (last 90 days) for recurring patterns
  - Scheduled payments (next 90 days)
  - Recurring AR invoicing (daily/weekly/monthly)
THEN CashFlowForecast is generated:
  - forecastDate="2026-05-21"
  - projectionPeriodDays=90
  - baselineCashPosition=€125,000
  - 90 daily projections (2026-05-22 to 2026-08-20)
  - Each day shows: inflows, outflows, netPosition, confidence%

SCENARIO: Identify liquidity risk
GIVEN the forecast projects negative balance on 2026-05-28
WHEN forecast calculates netPosition for that day
AND netPosition < 0
THEN a riskFlag is created:
  - date="2026-05-28"
  - severity="high"
  - message="Negative cash position (€-2,500) due to payroll outflow"
  - recommendation="Defer non-critical expenses or increase credit lines"
AND the risk is highlighted in UI (red flag)

SCENARIO: Confidence decreases with horizon
GIVEN today is 2026-05-21
WHEN forecast calculates confidence:
  - 7 days ahead: 95% (based on recent trends + scheduled payments)
  - 30 days ahead: 85% (GL patterns + assumptions)
  - 60+ days ahead: 70% (assumptions only)
THEN confidence values reflect data quality and prediction risk
AND widget shows confidence bands (shaded area around projection line)
```

**Technical Notes:**
- CashFlowForecast generated daily via `CashFlowForecastJob`
- Inflow sources: AR GL entries (revenue), bank deposits
- Outflow sources: AP GL entries (expenses), scheduled payments, payroll
- Recurring pattern detection: `RecurrenceDetectionService` (analyze GL last 90 days)
- Confidence formula: `base_confidence * (1 - horizon_days/90)`

---

## REQ-TCM-008: Personal Expense Tagging

**Title:** Mark expenses as personal to segregate from business GL  
**Priority:** High  
**Complexity:** Low

**Description:**
User can mark a payment/transaction as personal (with category) to exclude from business GL and generate personal expense reports.

**Acceptance Criteria:**

```gherkin
SCENARIO: Tag transaction as personal meal
GIVEN a Payment exists: amount=€87.50, reference="Local Café"
WHEN I open Payment detail
AND I toggle isPersonal=true
AND I select personalCategory="meals"
AND I click "Save"
THEN Payment is updated:
  - isPersonal=true
  - personalCategory="meals"
AND the GL posting is reversed (if already posted)
AND the transaction is added to Personal Expense Report
AND next GL reconciliation excludes this transaction

SCENARIO: Generate personal expense report
GIVEN multiple payments with isPersonal=true in May 2026
WHEN I navigate to Reports → Personal Expenses
AND I select date range="2026-05-01 to 2026-05-31"
THEN a report is generated:
  - Meals: €342.50 (5 transactions)
  - Travel: €125.00 (2 transactions)
  - Office: €87.50 (1 transaction)
  - Total: €555.00
AND I can export as CSV (suitable for tax software)

SCENARIO: Prevent double-counting in GL
GIVEN a Payment marked as personal
WHEN GL reconciliation runs
THEN personal payments are excluded from:
  - Cash account balance verification
  - GL posting validation
  - Audit trail (marked as "non-business")
```

**Technical Notes:**
- isPersonal flag on Payment schema (boolean, default false)
- personalCategory enum: travel, meals, office, other
- GL reversal: auto-post reverse entry if Payment was already in GL
- Personal Expense Report: simple GROUP BY personalCategory, SUM(amount)

---

## REQ-TCM-009: Bank Reconciliation UI

**Title:** User-friendly reconciliation interface with auto-matching  
**Priority:** High  
**Complexity:** Medium

**Description:**
Dedicated reconciliation page with two-column layout for bank transactions vs. GL entries, with drag-drop matching and bulk approve.

**Acceptance Criteria:**

```gherkin
SCENARIO: Open reconciliation hub
GIVEN I am a CFO
WHEN I navigate to Treasury → Reconciliation
AND I select BankAccount="NL91ABNA0417164300"
AND I select date range="2026-05-01 to 2026-05-21"
AND I click "Fetch Statement"
THEN the system:
  - Queries bank statement (50 transactions)
  - Queries GL cash entries (52 entries)
  - Applies ReconciliationRules → 47 auto-matched
  - Displays:
    - Left column: 50 bank transactions (3 unmatched highlighted)
    - Right column: 52 GL entries (5 unmatched highlighted)
    - Match status indicators (✓, ⚠, ✗)

SCENARIO: Manual match and approve
GIVEN 3 unmatched bank transactions visible
WHEN I drag "€2,500 - ACME Logistics" from bank column
AND I drop onto "€2,500 - Invoice INV-2026-001" in GL column
THEN a Match is created
AND both items show ✓ (matched)
AFTER matching all 50 transactions
WHEN I click "Approve Reconciliation"
THEN:
  - All matched pairs are posted to ReconciliationMatch GL entries
  - BankAccount.reconciliationDate=2026-05-21
  - ReconciliationReport is generated (with audit trail)
  - Notification to auditor: "May 2026 reconciliation complete, 50/50 matched"

SCENARIO: Flag discrepancy for investigation
GIVEN an unmatched transaction: €2,500 from supplier
AND no matching GL entry exists
WHEN I click "Create Missing GL Entry"
THEN a manual GL entry dialog opens (create cash receipt)
AND I fill: debit=2500, credit=?, date=reconcile date
AND I click "Post"
THEN GL entry is created and matched
AND the transaction is marked as "reconciled with note"
```

**Technical Notes:**
- UI component: custom `ReconciliationHub.vue` (two-column layout)
- Drag-drop: via `vue-draggable-next`
- Matching: stores Match objects linking Payment ↔ GeneralLedgerEntry
- Bulk approve: `ReconciliationService.approveReconciliation()` posts all Match records

---

## REQ-TCM-010: Payment Status Lifecycle

**Title:** Track payment status through complete workflow  
**Priority:** Critical  
**Complexity:** Low

**Description:**
Payment moves through states: draft → scheduled/submitted → authorized → captured → settled. Each state has guards and triggers.

**Acceptance Criteria:**

```gherkin
SCENARIO: Draft → Scheduled workflow
GIVEN a Payment with status="draft"
WHEN I navigate to Payment detail
AND I review: amount, payee, method, date, fees
AND I click "Schedule"
THEN status → "scheduled"
AND a reminder job is scheduled for 1 day before execution
AND Payment can no longer be edited (read-only)

SCENARIO: Scheduled → Initiated on due date
GIVEN a Payment with status="scheduled" AND requestedDate="2026-05-28"
WHEN scheduled payment processing job runs on 2026-05-28 at 6 AM
THEN status → "initiated"
AND system calls payment processor API (submit for processing)
AND external transaction ID is recorded in metadata.externalId
AND status → "pending" (waiting for processor confirmation)

SCENARIO: Pending → Authorized/Captured/Settled
GIVEN a Payment with status="pending"
WHEN payment processor webhook sends:
  - Event: "payment.authorized" → status → "authorized"
  - Event: "payment.captured" → status → "captured"
  - Event: "payment.settled" → status → "settled"
THEN GL entries are posted at "settled" state
AND Payment cannot be cancelled (locked for audit trail)

SCENARIO: Failed payment retry
GIVEN a Payment with status="failed"
WHEN I click "Retry"
AND the system resubmits to processor
AND processor approves
THEN status → "pending" → "authorized" → ... → "settled"
AND retryCount is incremented in metadata
AND notification sent: "Payment retry #1 succeeded"
```

**Technical Notes:**
- Status field is read-only after creation (enforced in backend)
- State transitions: guarded by `PaymentStateTransitionService`
- Reminders: scheduled via `ScheduledPaymentReminderJob`
- Webhook handling: `PaymentProcessorWebhookController` updates status based on event type
- GL posting: triggered by status transition to "settled" only

---

## REQ-TCM-011: Payment Method Flexibility

**Title:** Support 40+ payment methods with flexible routing  
**Priority:** High  
**Complexity:** High

**Description:**
System supports diverse payment methods (bank transfer, cards, digital wallets, regional methods) with per-method configuration.

**Acceptance Criteria:**

```gherkin
SCENARIO: Select payment method during creation
GIVEN I am creating a Payment
WHEN I click on paymentMethod field
THEN UI shows available methods:
  - SEPA Bank Transfer (€0 fee, 2-day settlement)
  - Visa Credit Card (1.5% fee, 1-day settlement)
  - iDEAL (Netherlands, €0.50 fee, 1-day settlement)
  - Klarna Pay Later (3% fee, 30-day invoice)
  - ACH (US, $0.25 fee, 1-2 day settlement)
  - Giropay (Germany, €0 fee, instant)
  - (36 more...)
AND supported currencies are filtered per method
AND fee structure is displayed

SCENARIO: International payment routing
GIVEN I want to pay a supplier in Switzerland (CHF 10,000)
WHEN I create a Payment with:
  - amount=10,000, currency="CHF"
  - payee in Switzerland
  - paymentMethod auto-suggests "Wise (multi-currency)" or "Standard SEPA to CHF account"
THEN system calculates FX rate and total fee (1.2% + CHF 30 bank fee)
AND confirms: "Total debit: €9,200 (at FX 1.08)"

SCENARIO: Method-specific validation
GIVEN I select paymentMethod="gift_card"
WHEN I create a Payment
AND the payee is a Supplier (not matching gift card network)
THEN validation error: "This supplier does not accept gift cards"
AND method is removed from available options
```

**Technical Notes:**
- PaymentMethod per-processor: Stripe (cards), Mollie (iDEAL/Bancontact/SEPA), Wise (FX), PayPal, etc.
- Method availability: filtered by currency, payee type, amount limits
- FX rates: fetched from ECB or Wise API, cached for 1 hour
- Validation rules: per-method in `PaymentMethodValidator`

---

## REQ-TCM-012: Multi-Currency Support

**Title:** Manage accounts and payments in multiple currencies  
**Priority:** High  
**Complexity:** High

**Description:**
System supports multi-currency bank accounts, FX conversion, and multi-currency forecasts.

**Acceptance Criteria:**

```gherkin
SCENARIO: Create USD bank account
GIVEN I am a CFO
WHEN I create a BankAccount with:
  - accountNumber="US0000123456789012" (valid US format)
  - currency="USD"
  - bankName="JP Morgan"
THEN the account is created with:
  - currency="USD"
  - balance tracked in USD (not EUR-converted)
  - Cash account in GL: USD subsidiary ledger account

SCENARIO: Payment in foreign currency
GIVEN I have a payment due to a US Supplier: $5,000 USD
AND I debit from my EUR bank account
WHEN I create Payment with:
  - amount=5000, currency="USD"
  - bankAccount=€ account (NL91ABNA...)
THEN system fetches FX rate (ECB or processor): €/$ = 1.10
AND confirms total debit: €4,545.45
AND GL posting shows:
  - Debit: USD Accounts Payable (FX gain/loss account)
  - Credit: EUR Cash Account -4,545.45
  - Gain/Loss: €4.55 (rounding)

SCENARIO: Multi-currency forecast
GIVEN cash positions in EUR (€125k), USD ($50k), GBP (£30k)
WHEN I view Cash Flow Forecast widget
THEN the system converts all to base currency (EUR):
  - EUR: €125,000
  - USD: $50k × 1.10 = €45,454
  - GBP: £30k × 1.17 = €35,100
  - Total: €205,554
AND the forecast is displayed in base currency
```

**Technical Notes:**
- FX rates: cached from ECB API (source-of-truth) or processor-provided rates
- GL accounts: multi-currency subsidiary accounts per currency (GL 1000-EUR, 1001-USD, 1002-GBP)
- FX gain/loss: auto-calculated and posted on settlement
- Forecast aggregation: all currencies converted to base currency at latest ECB rate

---

## REQ-TCM-013: Batch Payment File Generation

**Title:** Generate SEPA XML / ACH file for batch submission  
**Priority:** High  
**Complexity:** Medium

**Description:**
System generates standard file formats (SEPA XML, ACH file) from payment batch for submission to processor or bank.

**Acceptance Criteria:**

```gherkin
SCENARIO: Generate SEPA XML from batch
GIVEN a PaymentBatch with 5 SEPA payments:
  - Payee 1: €2,500 to IBAN NL91ABNA0417164301
  - Payee 2: €1,200 to IBAN NL91ABNA0417164302
  - ... (3 more)
  - Total: €12,500
WHEN I click "Generate SEPA File" in batch detail
THEN system generates SEPA XML (ISO 20022 XML 3.02):
  - Message ID, Creation DateTime, Batch numbers
  - Payment instructions (one per transaction)
  - Debtor account: NL91ABNA0417164300
  - Creditor accounts: individual IBANs
  - Amounts in cents, reference info
AND a file "SEPA-BATCH-2026-05-W2.xml" is created
AND user can download or submit directly to processor

SCENARIO: Generate ACH file
GIVEN a PaymentBatch with 10 ACH payments (US)
WHEN I click "Generate ACH File"
THEN system generates NACHA format:
  - File header + batch header
  - Individual entry details (routing number, account, amount, etc.)
  - Batch control + file control
  - File "ACH-BATCH-2026-05-W2.txt" is created

SCENARIO: Validate file before generation
GIVEN a batch with invalid IBAN or missing account
WHEN system validates before file generation
THEN validation errors are shown:
  - "IBAN NL91ABNA0417164301 invalid (check digit mismatch)"
AND file generation is blocked until corrected
```

**Technical Notes:**
- SEPA XML: use `symfony/sepa` library (ISO 20022 XML 3.02 compliant)
- ACH file: use `nacha` library (NACHA standard 94-byte records)
- File validation: checksum, field lengths, required fields
- File storage: uploaded to OpenRegister (FileService), indexed for retrieval

---

## REQ-TCM-014: Payment Reminders & Notifications

**Title:** Send scheduled payment reminders and status updates  
**Priority:** Medium  
**Complexity:** Low

**Description:**
System sends notifications at key payment milestones: 1 day before scheduled, failed attempt, settlement confirmation.

**Acceptance Criteria:**

```gherkin
SCENARIO: Reminder 1 day before scheduled payment
GIVEN a Payment with status="scheduled" AND requestedDate="2026-05-28"
WHEN a scheduled job runs on 2026-05-27 at 9 AM
THEN system sends notification to Payment.payer:
  - Title: "Payment reminder"
  - Message: "Payment of €2,500 to Supplier ABC is scheduled for tomorrow (2026-05-28)"
  - Action button: "View Payment"
AND the notification is sent via:
  - Nextcloud in-app notification
  - Email (if user has email enabled)

SCENARIO: Failed payment notification
GIVEN a Payment with status="failed"
WHEN the processor webhook sends failure event
THEN system sends notification to Payment creator + Approver:
  - Title: "Payment failed"
  - Message: "Payment of €1,200 to Payee XYZ failed: Insufficient funds"
  - Action: "Retry" or "Edit payment"

SCENARIO: Settlement confirmation
GIVEN a Payment with status="pending"
WHEN processor webhook sends "payment.settled"
THEN system sends notification to Payee (via AR):
  - Title: "Payment received"
  - Message: "Your payment of €5,000 has been settled (ref: INV-2026-001)"
AND notification to Payer (Finance team):
  - Title: "Payment settled"
  - Message: "€5,000 paid to Payee ABC on 2026-05-21"
```

**Technical Notes:**
- Notifications via `NotificationService`
- Scheduled reminders: `ScheduledPaymentReminderJob`
- Webhooks: async event handlers in `PaymentProcessorWebhookController`
- Email opt-in: user preference via `UserPreference`

---

## REQ-TCM-015: Payment Authorization Workflows

**Title:** Role-based approval workflow for payments  
**Priority:** High  
**Complexity:** Medium

**Description:**
Payments above threshold require approval from CFO or Director before submission to processor.

**Acceptance Criteria:**

```gherkin
SCENARIO: Small payment auto-approves
GIVEN a Payment with amount=€500
AND PaymentMethod="bank_transfer" (low-risk)
WHEN created by AP Clerk
THEN status → "draft" → immediately → "scheduled"
AND no approval required (auto-submit on requestedDate)

SCENARIO: Large payment requires approval
GIVEN a Payment with amount=€15,000
WHEN created by AP Clerk
THEN status → "draft" (locked for editing)
AND an ApprovalRequest is sent to CFO
AND APClerk sees: "Awaiting approval from CFO Jane Smith"
WHEN CFO receives notification and opens Payment detail
AND reviews amount, payee, reference
AND clicks "Approve"
THEN status → "approved" → "scheduled"
AND AP Clerk is notified: "Your payment was approved"

SCENARIO: Rejection workflow
GIVEN a Payment awaiting approval
WHEN CFO clicks "Reject"
AND provides reason: "Payee name doesn't match invoice header - correct before resubmitting"
THEN status → "rejected"
AND Payment is unlocked for editing
AND AP Clerk is notified: "Payment rejected: Payee name doesn't match invoice header..."
WHEN AP Clerk corrects the payee name
AND resubmits
THEN a new ApprovalRequest is created (cycle repeats)
```

**Technical Notes:**
- Thresholds: configurable per PaymentMethod in Settings
- Approval roles: CFO (all), Director (up to €10k), AP Supervisor (up to €5k)
- ApprovalRequest stored in OpenRegister (uses standard Approval/Task services)
- Delegation: CFO can delegate approval to Director (via IAppConfig)

---

## REQ-TCM-016: Payment Fraud Detection

**Title:** Monitor and flag suspicious payment activity  
**Priority:** High  
**Complexity:** Medium

**Description:**
System monitors for payment anomalies: unusual amounts, new payees, high frequency, and external alerts.

**Acceptance Criteria:**

```gherkin
SCENARIO: Unusual amount flag
GIVEN historical supplier payments: avg €2,000/month, max €5,000
WHEN a new Payment is created: payee=Supplier ABC, amount=€50,000
THEN FraudDetection service analyzes:
  - Historical avg (€2,000) vs. current (€50,000) = 25× spike
  - Confidence: 85% (high anomaly risk)
THEN system flags: ⚠️ "Unusual payment amount"
AND sends notification to CFO with:
  - Message: "Payment to Supplier ABC is 25× your typical amount. Verify before approval."
  - Options: "Approve anyway", "Edit", "Cancel"

SCENARIO: New supplier first payment
GIVEN a new Supplier (created today)
WHEN Payment is created to this supplier: amount=€10,000
THEN system flags: ⚠️ "First payment to new supplier"
AND enrichment via Dun & Bradstreet / Company Registry:
  - Company registration valid? ✓
  - Payment history available? (none for new supplier)
THEN notification to CFO: "First payment to new supplier Acme Corp (NL registration verified)"

SCENARIO: High-frequency payee alert
GIVEN 5 payments to Payee XYZ already submitted today
WHEN a 6th Payment to Payee XYZ is created on same day
THEN system flags: ⚠️ "High frequency to same payee"
AND sends alert: "6 payments to Payee XYZ detected today (total €18,500). Possible duplicate or fraud?"

SCENARIO: Disable payment on high-risk
GIVEN a Payment flagged with severity="critical"
AND fraud confidence > 90%
WHEN AP Clerk tries to submit for approval
THEN system blocks submission:
  - "This payment has been flagged for security review. Contact admin."
AND alerts sent to: Security Officer, CFO, Compliance
```

**Technical Notes:**
- FraudDetection service analyzes: historical average, frequency, new payee, amount variance
- Enrichment: Dun & Bradstreet API lookup (optional, configured in Settings)
- Anomaly thresholds: configurable (e.g., >5× average = flag)
- Critical flags: auto-escalate to Security Officer approval (additional workflow)

---

## REQ-TCM-017: Virtual Cards & Controlled Spending

**Title:** Create and manage virtual cards for controlled spending  
**Priority:** Medium  
**Complexity:** High

**Description:**
Admins can provision virtual cards (per-employee, per-project) with spending limits and transaction controls.

**Acceptance Criteria:**

```gherkin
SCENARIO: Create single-use virtual card
GIVEN I am a CFO
WHEN I navigate to Payments → Virtual Cards → "+ Issue Card"
AND I fill:
  - cardType="single-use"
  - amount=€500 (spending limit)
  - assignee=Employee John Smith
  - validUntil=2026-05-25 (3 days)
  - description="Conference registration"
AND I click "Issue"
THEN a virtual card is created:
  - Card number: 4242 4242 4242 4242 (masked in UI)
  - CVV: masked (shown once, recorded only)
  - Expiry: 05/27
  - Spending limit: €500
AND a notification is sent to Employee John with card details (encrypted email)
AND John can view the card in his Shillinq app (masked)

SCENARIO: Transaction via virtual card
GIVEN Employee John has a virtual card (€500 limit)
WHEN he makes a purchase: €150 at conference vendor
THEN:
  - Transaction is processed (VirtualCard processor approves, amount within limit)
  - Payment record is created: status="captured", reference="Conference vendor"
  - Remaining balance: €350
  - Balance visible to John in his app

SCENARIO: Exceed spending limit
GIVEN a virtual card with €500 limit AND remaining balance €100
WHEN Employee John attempts to spend €150
THEN:
  - Transaction is declined (insufficient balance)
  - Employee is notified: "Card declined. Remaining: €100"
  - Alert sent to CFO: "Virtual card declined - employee over limit"

SCENARIO: Card reuse prevention
GIVEN a virtual card marked as single-use
WHEN Employee John's transaction is settled (status="settled")
THEN the virtual card is automatically revoked:
  - Card is disabled (no further transactions allowed)
  - Expiry is set to transaction date
  - John is notified: "Virtual card deactivated after use"
```

**Technical Notes:**
- Virtual card provider: Stripe Issuing or Contour (depends on processor)
- Single-use: revoked after first successful settlement
- Per-employee limit: tracked per card + aggregate limit per employee
- Controls: MCC restrictions (can limit to specific merchant categories)

---

## REQ-TCM-018: Payout Reconciliation

**Title:** Match payouts to GL entries and settlement reports  
**Priority:** High  
**Complexity:** Medium

**Description:**
System automatically matches payout transactions (from bank statements) to GL revenue entries and settlement reports.

**Acceptance Criteria:**

```gherkin
SCENARIO: Match payout to AR invoice
GIVEN an AR Invoice:
  - amount=€3,000
  - paymentTerms="Net 30" (due 2026-05-21)
  - customer=Acme Corp
WHEN bank statement sync fetches:
  - Deposit from Acme Corp: €3,000 on 2026-05-20
THEN system:
  - Searches AR for matching invoice (reference, amount, date within tolerance)
  - Finds invoice match
  - Creates Payment record: status="matched"
  - GL posts: Debit Cash (€3,000), Credit AR Receivable (€3,000)

SCENARIO: Multi-payment payout
GIVEN 3 AR Invoices (€1,000, €1,500, €500) from same customer
WHEN bank statement shows single deposit: €3,000 on 2026-05-20
THEN system:
  - Matches deposit to all 3 invoices (grouped)
  - GL posts: Debit Cash (€3,000), Credit AR Receivable (€3,000)
  - Individual invoice status → "paid"

SCENARIO: Partial payout
GIVEN an AR Invoice: €5,000 (due date passed)
WHEN bank statement shows deposit: €3,000 on 2026-05-20
AND reference matches invoice (e.g., "INV-2026-001 partial")
THEN system:
  - Creates Payment: amount=€3,000, reference="partial payment"
  - GL posts: Debit Cash (€3,000), Credit AR Receivable (€3,000)
  - Invoice status → "partially paid" (balance €2,000 remaining)
  - System flags: "Partial payment received. Follow up for remaining balance."

SCENARIO: Payout with fee deduction
GIVEN a PaymentMethod: "Stripe" with 2.9% + $0.30 fee
WHEN customer pays €3,000 via Stripe
THEN bank deposits: €3,000 - (€3,000 × 2.9% + €0.30) = €2,901.70
AND system:
  - Matches payout €2,901.70 to AR invoice €3,000
  - Calculates fee: €98.30
  - GL posts: Debit Cash (€2,901.70), Debit Fee Expense (€98.30), Credit AR (€3,000)
```

**Technical Notes:**
- Payout matching: reverse of payment matching (bank deposit ↔ AR invoice)
- Tolerance: configurable amount (€1) and date (3 days)
- Fee calculation: per PaymentMethod configuration
- GL accounts: AR Receivable + Cash + Fee Expense

---

## REQ-TCM-019: Currency Exchange Management

**Title:** Track and manage FX exposure and gains/losses  
**Priority:** Medium  
**Complexity:** High

**Description:**
System tracks FX exposure across multi-currency accounts, calculates realized and unrealized gains/losses.

**Acceptance Criteria:**

```gherkin
SCENARIO: Unrealized FX gain
GIVEN:
  - USD Bank Account: $100,000 (opened at FX €/$ = 1.10)
  - GL: USD Cash (€90,909 at purchase date)
WHEN current date: 2026-05-21, FX rate: €/$ = 1.12
THEN:
  - Current value: $100,000 × 1.12 = €89,286
  - Unrealized loss: €90,909 - €89,286 = €1,623 (adverse move)
  - FX Exposure report shows: USD account = -€1,623 unrealized loss
  - GL shows: FX Adjustment account (not posted until settled)

SCENARIO: Realized FX gain on payment
GIVEN:
  - Payment due: €5,000 to EUR supplier
  - Original amount: $5,500 USD (FX rate 1.10)
  - GL entry: Debit Expense €5,000, Credit USD Cash €5,000
  - Current FX rate: 1.12
WHEN Payment is settled: $5,500 × 1.12 = €4,911
THEN:
  - Realized gain: €5,000 - €4,911 = €89
  - GL posts: Debit Cash (€4,911), Credit USD Liability (€5,000), Credit FX Gain (€89)
  - Payment.metadata.fxGain = €89

SCENARIO: FX exposure report
GIVEN multiple accounts in EUR, USD, GBP
WHEN I navigate to Reports → FX Exposure
AND select date="2026-05-21"
THEN report shows:
  - EUR account: €125,000 (base, no exposure)
  - USD account: $50,000 (exposed -€1,500 at 1.12 rate)
  - GBP account: £30,000 (exposed +€2,100 at 1.17 rate)
  - Total unrealized: +€600 (net gain if rates hold)
  - Hedging recommendations (if used)
```

**Technical Notes:**
- FX rates: fetched daily from ECB (source-of-truth)
- Unrealized gain/loss: calculated daily, not posted until settlement
- Realized gain/loss: posted at payment settlement (automatic GL entry)
- Exposure report: aggregates all accounts, shows per-currency and net

---

## REQ-TCM-020: Compliance & Audit Trail

**Title:** Full audit trail for all payments and reconciliations  
**Priority:** Critical  
**Complexity:** Low

**Description:**
All payment and reconciliation actions are logged with user, timestamp, before/after values. Audit trail is tamper-proof and exportable.

**Acceptance Criteria:**

```gherkin
SCENARIO: Audit trail on payment creation
GIVEN a Payment is created with:
  - amount=€5,000
  - payee=Supplier ABC
  - reference=INV-2026-001
WHEN system records creation
THEN AuditTrail entry is created:
  - timestamp: 2026-05-21T10:30:00Z
  - user: John (AP Clerk)
  - action: "create"
  - object: "Payment"
  - objectId: UUID
  - changes: {amount: "null → 5000", payee: "null → Supplier ABC", ...}
  - ipAddress: 192.168.1.100
  - userAgent: "Mozilla/5.0..."

SCENARIO: Audit trail on status transition
GIVEN Payment with status="draft"
WHEN status is updated to "scheduled"
THEN AuditTrail entry:
  - action: "update"
  - changes: {status: "draft → scheduled", requestedDate: "2026-05-28"}
  - user: Jane (CFO)
  - timestamp: 2026-05-21T14:00:00Z

SCENARIO: Reconciliation audit trail
GIVEN 50 bank transactions being reconciled
WHEN reconciliation is approved
THEN AuditTrail entries created (one per matched pair):
  - action: "match"
  - object: "ReconciliationMatch"
  - changes: {bankTransaction: "...", glEntry: "...", matchedBy: "John", matchedDate: "2026-05-21"}

SCENARIO: Export audit trail
GIVEN I am an Auditor
WHEN I navigate to Reports → Audit Trail
AND I filter: dateRange="2026-05-01 to 2026-05-31", objectType="Payment"
AND I click "Export"
THEN a CSV/Excel file is downloaded:
  - Columns: Timestamp, User, Action, Object, ObjectId, Field, OldValue, NewValue, IPAddress
  - 500+ rows of payment audit trail
  - SHA256 hash appended (for tamper detection)
```

**Technical Notes:**
- Audit trail: provided by OpenRegister `AuditTrailService` (automatic)
- Immutable: cannot be edited (append-only log)
- Retention: minimum 7 years (GDPR, legal hold via `RetentionService`)
- Export: filtered CSV/Excel via `ExportService`

---

## Integration Scenarios

### Scenario A: Supplier Invoice → Payment → Settlement (End-to-End)

```
1. AP receives Invoice from Supplier (€2,500)
2. Invoice recorded in AP (status="due")
3. AP Clerk creates Payment from Invoice
   - amount=€2,500, payee=Supplier, reference=INV-2026-001
   - status → draft
4. Payment scheduled for requestedDate=2026-05-28
   - status → scheduled
5. On 2026-05-28, scheduled job submits to SEPA processor
   - status → initiated → pending
6. Processor webhook: payment authorized
   - status → authorized
7. Processor webhook: payment settled
   - status → settled
   - GL post: Debit AP Payable (€2,500), Credit Cash (€2,500)
8. Bank statement sync on 2026-05-30 fetches debit
   - Creates Payment record (for reconciliation)
   - Reconciliation rule matches to GL entry (✓ matched)
9. Weekly reconciliation: CFO approves
   - Reconciliation complete

Audit trail tracks all steps, all users, all changes.
```

### Scenario B: Multi-Currency Payout from AR

```
1. AR Invoice: customer=Acme Corp (US), amount=$5,000 (due 2026-05-28)
2. GL accounts: AR in USD subsidiary account
3. Customer pays via Stripe: deposits $4,850 (after 1.5% fee: $150)
4. Bank sync (USD account) fetches deposit on 2026-05-28
   - Creates Payment: amount=$4,850, status="pending"
5. Reconciliation rule matches to AR invoice
   - FX: $4,850 × 1.10 = €4,409
   - Fee: 1.5% of $5,000 = $75 = €68.18
   - Realized FX: $5,000 expected @ 1.10 = €4,545; received @ 1.10 = €4,545
   - (FX realized at settlement rate, not original rate)
6. GL posts:
   - Debit Cash EUR (€4,409)
   - Credit AR USD (€4,545 @ settlement rate)
   - Credit FX Gain (€136 rate movement)
7. Reconciliation matches payment to AR
   - Invoice status → "paid"
```

---

**Traceability:** All requirements reference relevant ADRs:
- REQ-TCM-001 to REQ-TCM-020: traceable to `@spec openspec/changes/treasury-cash-management-other-t2/specs.md#REQ-TCM-NNN`
- Backend services: `@spec openspec/changes/treasury-cash-management-other-t2/tasks.md#task-N`
- Frontend components: `@spec openspec/changes/treasury-cash-management-other-t2/tasks.md#task-N`

---

**Next Steps:**
1. Backend team: Implement services for each REQ-* in tasks.md
2. Frontend team: Vue components matching each REQ-* scenario
3. Integration tests: Newman/Postman for each scenario
4. Browser tests: Playwright for GIVEN/WHEN/THEN per spec
