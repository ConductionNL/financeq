# Specifications: Treasury & Cash Management — Shillinq

**Version:** 1.0  
**Spec ID:** treasury-cash-management-other-t3  

## User Stories & Acceptance Criteria

### Epic 1: Multi-Bank Visibility

#### US-001: Connect Bank Account via Plaid Link

**As a** treasurer  
**I want to** securely connect a bank account using Plaid Link  
**So that** I can see real-time balances and import transactions automatically

**Acceptance Criteria:**

```gherkin
Scenario: User initiates bank account connection
  GIVEN a user is on the Bank Accounts page
  AND they have permission to manage bank connections
  WHEN they click "Add Bank Account"
  THEN a Plaid Link modal opens
  AND the modal displays supported banks by country

Scenario: User selects bank and authenticates
  GIVEN Plaid Link is open
  WHEN they search for and select their bank (e.g., "ABN AMRO")
  AND they authenticate with their online banking credentials
  THEN Plaid returns a list of available accounts
  AND the user can select which account(s) to connect

Scenario: Account successfully connected
  GIVEN the user has authenticated and selected account(s)
  WHEN they complete the Plaid Link flow
  THEN the system creates a BankAccount object:
    - name = account nickname (user input or bank name)
    - iban = from Plaid
    - bic = from Plaid
    - plaidAccountId = encrypted and stored
    - syncEnabled = true
    - syncStatus = "pending_auth"
  AND the user is shown "Importing transactions..." with a spinner
  AND background job immediately fetches last 90 days of transactions

Scenario: Connection fails due to invalid credentials
  GIVEN the user enters incorrect banking credentials
  WHEN Plaid returns an authentication error
  THEN the system shows error message: "Bank rejected credentials. Try again or contact your bank."
  AND BankAccount is NOT created
  AND user can retry

Scenario: Account already connected (duplicate detection)
  GIVEN a BankAccount with IBAN "NL91ABNA0417164300" already exists
  WHEN the user tries to connect the same account again via Plaid
  THEN the system detects duplicate by IBAN match
  AND shows warning: "This account is already connected. Click [View] or [Cancel]."
  AND does NOT create a duplicate BankAccount
```

**Requirements:**

- **REQ-001-001:** Plaid Link integration MUST securely exchange OAuth token for encrypted storage. NO plaintext credentials stored.
- **REQ-001-002:** BankAccount.plaidAccountId MUST be encrypted at rest (database encryption + field-level encryption).
- **REQ-001-003:** On connection, system MUST immediately fetch last 90 days of transactions via Plaid API.
- **REQ-001-004:** Duplicate detection by IBAN + account type MUST prevent re-connection of existing accounts.
- **REQ-001-005:** Plaid Link SDK MUST support Dutch + European banks (Plaid EU API, not US-only).

---

#### US-002: View Multi-Bank Cash Position Dashboard

**As a** CFO  
**I want to** see my total liquid assets across all connected banks at a glance  
**So that** I can answer "what's our cash position?" in <5 minutes

**Acceptance Criteria:**

```gherkin
Scenario: Dashboard shows current cash position
  GIVEN the user has 2+ bank accounts connected
  AND all accounts have completed initial sync (syncStatus = "active")
  WHEN they navigate to Treasury Dashboard
  THEN they see:
    - Total Liquidity card: sum of all BankAccount.currentBalance values
    - By-account breakdown: each account name + current balance + last sync time
    - Liquidity trend chart (last 30 days): line chart showing total liquidity over time

Scenario: Account sync is in progress
  GIVEN a BankAccount with syncStatus = "pending_auth"
  WHEN the dashboard loads
  THEN that account shows loading spinner instead of balance
  AND displays "Importing transactions..."

Scenario: Account sync failed (temporary error)
  GIVEN a BankAccount with syncStatus = "error"
  WHEN the dashboard loads
  THEN account shows error badge (red)
  AND displays error message: "Last sync failed. Retrying..."
  AND shows "Retry Now" button

Scenario: Filter transactions by account and date range
  GIVEN the user is viewing the dashboard transaction list
  WHEN they select an account from the filter dropdown
  AND they select a date range
  THEN the transaction list updates to show only transactions from that account in that date range
  AND count is displayed: "Showing X of Y transactions"

Scenario: Sort transactions by date, amount, counterparty
  GIVEN the user is viewing the dashboard transaction list
  WHEN they click column headers (Date, Amount, Counterparty)
  THEN transactions are sorted by that column
  AND a sort indicator (↑↓) appears in the header
```

**Requirements:**

- **REQ-002-001:** Dashboard MUST load in <2 seconds (cached balance data, background sync).
- **REQ-002-002:** Balance data MUST refresh no less than every 5 minutes (via background job or webhook from Plaid).
- **REQ-002-003:** Liquidity trend MUST be calculated from daily snapshots (stored in CashFlowForecast or separate TimeSeries entity).
- **REQ-002-004:** Total Liquidity = sum of currentBalance across all isActive=true BankAccounts (exclude closed accounts).
- **REQ-002-005:** Last sync time MUST display in local user timezone (not UTC).

---

### Epic 2: Payment Scheduling & Execution

#### US-003: Schedule Bill Payment

**As an** accounts payable manager  
**I want to** schedule a payment to a vendor for a future date  
**So that** I can plan cash outflows and automate approval workflows

**Acceptance Criteria:**

```gherkin
Scenario: Create a single bill payment
  GIVEN the user is on the Payment Scheduler page
  WHEN they click "Schedule Payment"
  THEN a form appears with fields:
    - From Account (dropdown, required)
    - To Payee (autocomplete/select from supplier list, required)
    - Payment Method (dropdown: SEPA, ACH, iDEAL, etc., required)
    - Amount (number, 2 decimals, required)
    - Scheduled Date (date picker, required, must be future date)
    - Description (text, optional)
    - Link Expenses (multi-select, optional)
    - Link Invoices (multi-select, optional)

Scenario: Link payment to bills/expenses
  GIVEN the user is creating a payment
  WHEN they click "Link Expenses"
  THEN a modal shows unpaid expenses filtered by vendor
  AND they can select multiple expenses
  AND the Amount field auto-populates with sum of selected expenses
  AND the Description field auto-populates with expense references

Scenario: Submit payment for approval
  GIVEN the user has filled in all required fields
  WHEN they click "Submit for Approval"
  AND the amount is > threshold (e.g., €5,000)
  THEN:
    - ScheduledPayment is created with status = "pending_approval"
    - ApprovalChain is evaluated (route to CFO if amount > €10k, to treasurer if < €10k)
    - Approver receives notification: "Payment of €X.XX to [Payee] awaits approval"
    - User sees confirmation: "Payment submitted for approval. Approver will review by [due date]"

Scenario: Auto-approved payment (low amount)
  GIVEN a ScheduledPayment with amount < €1,000
  WHEN the user clicks "Submit for Approval"
  AND their role has "auto_approve_under_1000" permission
  THEN:
    - ScheduledPayment.approvalStatus = "auto_approved"
    - ScheduledPayment.status moves directly to "scheduled"
    - No notification sent to approver

Scenario: Payment scheduled and awaiting execution
  GIVEN a ScheduledPayment with status = "scheduled"
  WHEN the scheduled date arrives
  THEN a background job:
    - Fetches the payment
    - Validates bank account still has sufficient balance
    - Formats payment per SEPA/ACH standards
    - Submits to bank API
    - Updates ScheduledPayment.status = "submitted"
    - Updates ScheduledPayment.executionDate = today
    - Creates audit log entry

Scenario: Payment execution fails (insufficient funds)
  GIVEN a ScheduledPayment is due for execution
  WHEN the payment account has insufficient balance
  THEN:
    - Payment is NOT submitted to bank
    - ScheduledPayment.status = "failed"
    - ScheduledPayment.failureReason = "Insufficient funds"
    - Finance manager receives alert: "Payment to [Payee] failed: insufficient funds. Balance: €X"
    - Manager can click "Retry Tomorrow" to reschedule
```

**Requirements:**

- **REQ-003-001:** Payment amount MUST be validated: >0, ≤ remaining balance + credit limit (if defined).
- **REQ-003-002:** Scheduled date MUST be ≥ today + 1 day (no same-day payments except "Immediate Transfer").
- **REQ-003-003:** Approval chain MUST be configurable by amount threshold (Settings → Approval Rules).
- **REQ-003-004:** Auto-approval MUST require explicit role permission (PropertyRbacHandler: "auto_approve_threshold").
- **REQ-003-005:** SEPA payment format MUST comply with ISO 20022 XML (pain.001.003.02 for CT/SCT).
- **REQ-003-006:** Bank API calls MUST timeout after 30 seconds; retry up to 3 times over 1 hour.
- **REQ-003-007:** All payment submissions MUST be audit-logged with user, timestamp, bank confirmation ID.

---

#### US-004: Bulk Payment Scheduling

**As an** AP manager  
**I want to** upload a CSV of vendor payments and schedule them all at once  
**So that** I don't have to enter each payment individually

**Acceptance Criteria:**

```gherkin
Scenario: Upload payment batch CSV
  GIVEN the user is on Payment Scheduler page
  WHEN they click "Bulk Import"
  THEN a modal allows CSV upload
  AND displays template: "Payee | Amount | Date | Description"
  AND shows example rows

Scenario: Validate and preview batch
  GIVEN the user has uploaded a CSV with 20 payment rows
  WHEN the system processes the file
  THEN:
    - Parse CSV and validate each row (required fields: Payee, Amount, Date)
    - Match "Payee" column to existing Payee objects (fuzzy match or exact)
    - Show preview table with columns: Payee, Amount, Date, Status (Valid/Error)
    - Highlight rows with errors (e.g., "Payee not found", "Invalid date")
    - Show summary: "18 valid, 2 errors. Fix errors or skip invalid rows."

Scenario: Create batch with validation errors
  GIVEN the preview shows some invalid rows
  WHEN the user clicks "Create Batch" without fixing errors
  THEN the system asks for confirmation:
    "2 rows have errors. Continue without them?"
  AND if confirmed, only valid rows are created as ScheduledPayments

Scenario: Batch creation succeeds
  GIVEN the user has confirmed a valid batch upload
  WHEN the system processes all rows
  THEN:
    - ScheduledPayment created for each row
    - All payments show status = "draft" initially
    - A PaymentBatch object is created (groups these payments)
    - User sees confirmation: "18 payments created. [Review] [Submit All]"

Scenario: Submit entire batch for approval
  GIVEN a PaymentBatch with 18 draft payments
  WHEN the user clicks "Submit All"
  THEN:
    - All ScheduledPayments transition to "pending_approval"
    - Total amount calculated (€XXX)
    - Single approval request to CFO: "Batch of 18 payments, €XXX total"
    - (Individual approvals NOT needed; single approval for batch)
```

**Requirements:**

- **REQ-004-001:** CSV upload MUST support encoding: UTF-8, ISO-8859-1.
- **REQ-004-002:** Payee matching MUST use fuzzy string matching (Levenshtein distance ≤ 2 = potential match).
- **REQ-004-003:** For ambiguous payee matches, system MUST prompt user to select from options.
- **REQ-004-004:** PaymentBatch grouping MUST allow partial approval (approve 10 of 18, retry others).
- **REQ-004-005:** Batch CSV import MUST include validation report (downloadable as attachment in email confirmation).

---

### Epic 3: Bank Reconciliation

#### US-005: Auto-Reconcile Bank Transactions

**As a** finance manager  
**I want to** automatically match bank transactions to invoices and expenses  
**So that** I reduce manual reconciliation effort from 4 hours to <1 hour monthly

**Acceptance Criteria:**

```gherkin
Scenario: Bank transactions imported and auto-matching starts
  GIVEN 50 new bank transactions have been imported
  WHEN the background job runs (daily at 6 AM)
  THEN:
    - System loads all active ReconciliationRules (sorted by priority)
    - For each unmatched BankTransaction:
      - Evaluate criteria (amount, description, counterparty) against each rule
      - If rule matches: set BankTransaction.matchedInvoice/Expense and reconciliationStatus = "matched"
      - If no rule matches: set reconciliationStatus = "pending" (manual review)
    - Finance manager sees count: "40 auto-matched, 10 pending your review"

Scenario: View auto-matched transactions
  GIVEN the reconciliation is complete
  WHEN the finance manager opens Reconciliation Dashboard
  THEN they see:
    - Matched count: 40 (with "✓" badge)
    - Pending count: 10 (highlighted, awaiting action)
    - Each matched transaction shows: date, amount, description, matched item reference
    - Can click to view or edit the match

Scenario: Correct an incorrect auto-match
  GIVEN a BankTransaction with an incorrect matched invoice (wrong amount but similar name)
  WHEN the manager clicks "Edit Match"
  THEN:
    - A dialog opens: "Current match: Invoice INV-2026-001"
    - Manager can select a different invoice or mark as "Manual Review Required"
    - Updates BankTransaction.matchedInvoice and manuallyMatched = true
    - Records manager and timestamp

Scenario: Mark transaction as excluded (not a real match)
  GIVEN a BankTransaction that shouldn't be matched (e.g., bank fee, interest)
  WHEN the manager clicks "Mark as Excluded"
  THEN:
    - reconciliationStatus = "excluded"
    - matchTarget = "journal_entry" (auto-creates GLA entry if configured)
    - Transaction is hidden from pending list
    - Can view under "Excluded Transactions"

Scenario: Month-end reconciliation close
  GIVEN it's May 31, 11:59 PM
  AND all May transactions have been reviewed (matched or excluded)
  WHEN the manager clicks "Close May Reconciliation"
  THEN:
    - System locks all May BankTransactions (immutable)
    - Generates reconciliation report: matched %, variance, by-category breakdown
    - Creates snapshot for audit trail
    - Notifies: "May reconciliation closed. Report: [PDF link]"
```

**Requirements:**

- **REQ-005-001:** Auto-matching MUST run on all BankTransactions with reconciliationStatus="unmatched".
- **REQ-005-002:** Matching criteria evaluation MUST respect priority order (ReconciliationRule.priority).
- **REQ-005-003:** First rule match wins (no multiple matches per transaction).
- **REQ-005-004:** Manual adjustments MUST be audit-logged (user, old match, new match, timestamp).
- **REQ-005-005:** Month-end close MUST prevent changes to prior-month transactions (soft immutability via permissions).
- **REQ-005-006:** Reconciliation report MUST include: total transactions, matched count/%, matched amounts, variance (sum of unmatched), by-category breakdown.

---

#### US-006: Configure Auto-Matching Rules

**As a** finance manager  
**I want to** set up custom rules to automatically match transactions for common vendors  
**So that** I don't have to manually review routine vendor payments

**Acceptance Criteria:**

```gherkin
Scenario: Create a new matching rule
  GIVEN the user is in Settings → Reconciliation Rules
  WHEN they click "Add Rule"
  THEN a form appears with fields:
    - Name (required, e.g., "Acme Corp exact amount")
    - Description (optional)
    - Bank Account (required, scope rule to account)
    - Priority (number, lower = higher priority, required)
    - Matching Criteria:
      - Amount Type: Exact / Range
      - Amount Min/Max (if Range)
      - Description Keywords (comma-separated, optional)
      - Counterparty Pattern (regex, optional)
      - Transaction Type Filter (checkboxes: Credit, Debit)
    - Match Target: Invoice / Expense / Payment / Journal Entry / Manual Review
    - For each target, optional field mapping (e.g., expense category)
    - Auto Apply (toggle, default: true)
    - Requires Approval (toggle, default: false for auto rules)

Scenario: Test rule against existing transactions
  GIVEN the user has filled in matching criteria
  WHEN they click "Test Rule"
  THEN:
    - System loads last 30 days of unmatched transactions
    - Evaluates rule against each transaction
    - Shows preview: "This rule would match 7 of 30 transactions"
    - Lists matched transactions: date, amount, description, counterparty
    - User can click on each to review before saving rule

Scenario: Save and activate rule
  GIVEN the user is satisfied with test results
  WHEN they click "Save Rule"
  THEN:
    - ReconciliationRule is created with isActive = true
    - If autoApply = true: system immediately re-evaluates all pending transactions
    - Auto-matched transactions count updates in dashboard
    - Confirmation: "Rule saved. Auto-matched 7 transactions."

Scenario: Disable a rule temporarily
  GIVEN an active ReconciliationRule
  WHEN the user clicks the toggle to disable it
  THEN:
    - isActive = false
    - Rule is no longer evaluated for new transactions
    - Previously matched transactions retain their match (historical data)
    - Confirmation: "Rule disabled. Future transactions won't use this rule."

Scenario: Reorder rule priorities
  GIVEN multiple ReconciliationRules
  WHEN the user drags Rule A above Rule B
  THEN:
    - Priority values are updated (Rule A.priority < Rule B.priority)
    - On next transaction evaluation, Rule A is checked before Rule B
```

**Requirements:**

- **REQ-006-001:** ReconciliationRule.matchingCriteria MUST support: exact amount, amount range, keyword substring, regex pattern on counterparty, transaction type filter.
- **REQ-006-002:** Description keywords MUST be case-insensitive substring match (e.g., "acme" matches "ACME Corp" and "Acme Services").
- **REQ-006-003:** Counterparty pattern MUST support valid regex (e.g., `^Vendor.*LLC$`) with input validation to prevent ReDoS.
- **REQ-006-004:** Rule evaluation MUST short-circuit on first match (priority order enforced).
- **REQ-006-005:** Rule test MUST be non-destructive (preview only, no changes until "Save").
- **REQ-006-006:** Rule deletion MUST be soft-delete (historical rule references preserved for audit, rules not reapplied).

---

### Epic 4: Cash Flow Planning

#### US-007: View 13-Week Cash Flow Forecast

**As a** CFO  
**I want to** see a 13-week projection of cash inflows and outflows  
**So that** I can plan for working capital needs and identify cash shortfalls early

**Acceptance Criteria:**

```gherkin
Scenario: Forecast dashboard displays weekly projections
  GIVEN it's May 21, 2026
  WHEN the user opens the Cash Flow Dashboard
  THEN they see:
    - Forecast period: "May 21 - Aug 16 (13 weeks)"
    - Summary cards: Opening Balance, Total Inflows, Total Outflows, Closing Balance
    - Area chart showing weekly closing balances over 13 weeks
    - Risk indicator: if any week shows negative balance, color the week red
    - Confidence indicator: "Model confidence: 87%" (based on historical RMSE)

Scenario: Drill down into weekly detail
  GIVEN the forecast chart is displayed
  WHEN the user hovers over a week on the chart
  THEN a tooltip shows:
    - Week of: [date range]
    - Opening Balance: €85,000
    - Inflows: Invoices due €45,000, Other €5,000 (total €50,000)
    - Outflows: Bills due €35,000, Payroll €18,000, Tax €2,000 (total €55,000)
    - Closing Balance: €80,000

Scenario: View weekly breakdown table
  GIVEN the user is on the Cash Flow Dashboard
  WHEN they click "View Table" or scroll below the chart
  THEN a detailed table shows each week:
    - Week Start | Opening Bal | Inflows | Outflows | Closing Bal | Variance
    - Each cell can be clicked to see breakdown (invoices, bills, scheduled payments)

Scenario: Forecast based on unpaid invoices
  GIVEN the user has 10 unpaid invoices:
    - INV-001: €15,000 due May 25
    - INV-002: €8,000 due June 1
    - INV-003: €12,000 due June 10
    - etc.
  WHEN the forecast is generated
  THEN the Cash Flow Forecast sums invoices by due week:
    - Week of May 21: +€15,000
    - Week of May 28: +€8,000
    - Week of June 5: +€12,000

Scenario: Forecast factors in scheduled payments
  GIVEN the user has scheduled payments:
    - Payment to Vendor A: €5,000 due June 15
    - Payroll: €45,000 due June 1
    - Tax payment: €20,000 due June 30
  WHEN the forecast is generated
  THEN outflows include these scheduled amounts in their respective weeks

Scenario: Historical variance tracking (actuals vs. forecast from 4 weeks ago)
  GIVEN a forecast was generated 4 weeks ago
  AND actual transactions have now occurred
  WHEN the user views the historical accuracy report
  THEN they see comparison table:
    - Week | Forecast (4 weeks ago) | Actual (now) | Variance | % Error
    - Week A | €50,000 | €48,500 | -€1,500 | -3%
    - Week B | €45,000 | €47,200 | +€2,200 | +4.9%
    - etc.
  AND model accuracy metrics are shown:
    - 1-week RMSE: 2.3%
    - 3-week RMSE: 5.1%
    - 13-week RMSE: 8.9%
```

**Requirements:**

- **REQ-007-001:** CashFlowForecast MUST be generated daily at 6 AM UTC.
- **REQ-007-002:** Forecast model MUST use linear regression on last 90 days of transaction patterns.
- **REQ-007-003:** Inflows = sum of Invoice.totalAmount where Invoice.dueDate is within forecast period AND Invoice.status != "paid".
- **REQ-007-004:** Outflows = sum of ExpenseLineItem.totalAmount where due date in period AND status != "paid" + scheduled payments + historical daily average for non-invoice outflows.
- **REQ-007-005:** Confidence % MUST be calculated as `(1 - RMSE / avg_daily_amount) * 100`, clamped to [0, 100].
- **REQ-007-006:** Negative balance weeks MUST be visually highlighted (red zone) and trigger alert notification.
- **REQ-007-007:** Forecast accuracy comparison MUST compare against same week from previous forecast (1-month-old data, if available).

---

### Epic 5: Customer Payment Preferences

#### US-008: Set Customer Payment Method

**As a** sales manager  
**I want to** configure how each customer prefers to pay (iDEAL, bank transfer, card, etc.)  
**So that** they have a seamless payment experience and we reduce payment friction

**Acceptance Criteria:**

```gherkin
Scenario: Create customer payment preference
  GIVEN a customer exists in the system
  WHEN the user clicks "Payment Settings" on the customer detail page
  THEN a form appears:
    - Primary Payment Method (required dropdown): iDEAL, Giropay, SEPA, Card, Offline
    - SEPA Mandate (if SEPA selected): list existing mandates or "Create Mandate"
    - Card Token (if Card selected): "(Auto-filled from hosted vault or manual entry)"
    - IBAN (if Bank Transfer selected): auto-fill from mandate or manual entry
    - Subscription State (dropdown): None, Active, Paused, Cancelled
    - Frequency (if subscription): Days between payments (e.g., 30 for monthly)
    - Next Payment Date (date picker)

Scenario: Link SEPA mandate to customer
  GIVEN the user has selected "SEPA" as payment method
  WHEN they click "Create Mandate" (or "Link Existing")
  THEN:
    - If creating: a mandate generation workflow begins
    - Mandate document is generated (XML per ISO 20022)
    - Customer is sent mandate for signature (via email or API)
    - Upon signature, Mandate.status = "active"
    - CustomerPaymentPreference.mandateAgreement references the mandate

Scenario: Enable recurring subscription payment
  GIVEN a customer with SEPA mandate
  WHEN the user sets:
    - Subscription State: "Active"
    - Frequency: 30 days
    - Amount: €1,500 per cycle
  AND clicks "Save"
  THEN:
    - CustomerPaymentPreference is saved with subscription settings
    - A background job schedules the first ScheduledPayment for 30 days from now
    - Customer receives notification: "Your subscription is set up for €1,500 every 30 days"

Scenario: Customer payment page uses their preference
  GIVEN a customer with iDEAL as primary payment method
  WHEN they are presented with a payment page (invoice checkout)
  THEN:
    - iDEAL is pre-selected (fastest checkout)
    - Other options available (fallback)
    - Customer clicks "Pay with iDEAL"
    - They are redirected to their bank's iDEAL login
    - Upon success, invoice is marked paid

Scenario: Payment fails and retry is queued
  GIVEN a ScheduledPayment (recurring subscription) fails to execute
  WHEN the payment gateway returns a decline
  THEN:
    - CustomerPaymentPreference.lastPaymentStatus = "failed"
    - failureRetryCount incremented
    - If retryCount < 3: retry is queued for 24 hours later
    - If retryCount >= 3: manager is alerted and subscription is paused
    - Customer receives notification: "Payment failed. Please update your payment method."
```

**Requirements:**

- **REQ-008-001:** CustomerPaymentPreference MUST enforce exactly one primary payment method (enum, required).
- **REQ-008-002:** SEPA mandate MUST be persisted as separate Mandate entity per ADR-001; MUST validate IBAN format before storage.
- **REQ-008-003:** Card tokenization MUST comply with PCI-DSS; NO card data stored directly (use host vault or Token service).
- **REQ-008-004:** Subscription scheduling MUST automatically create ScheduledPayment on a cron job (daily check for due dates).
- **REQ-008-005:** Payment method fallback: if primary method is unavailable (gateway down, mandate expired), system MUST present secondary options.
- **REQ-008-006:** Customer payment page MUST support white-label branding (Organization-specific logo, colors, terms).

---

#### US-009: Branded Customer Payment Page

**As a** vendor/sales lead  
**I want to** customize the payment page with our branding (logo, colors, company name)  
**So that** customers see a professional, branded experience instead of a generic page

**Acceptance Criteria:**

```gherkin
Scenario: Configure brand settings for payment page
  GIVEN a user is in Settings → Brand Configuration
  WHEN they upload/configure:
    - Company Logo (PNG/SVG, max 2 MB)
    - Primary Color (hex #RRGGBB)
    - Secondary Color (hex #RRGGBB)
    - Company Name (text)
    - Support Email (text)
    - Terms URL (optional, links to /terms)
  AND click "Save"
  THEN the configuration is stored in Settings
  AND confirmation: "Brand settings updated. View preview."

Scenario: Preview branded payment page
  GIVEN the user has entered brand settings
  WHEN they click "Preview"
  THEN a modal shows the payment page with:
    - Logo displayed in header
    - Company name below logo
    - Primary color used for button (Pay), highlights
    - Secondary color for accents
    - Support email shown in footer

Scenario: Customer views branded payment page
  GIVEN a customer receives an invoice with a payment link
  WHEN they click the link to pay
  THEN the payment page displays:
    - Vendor's logo and branding (colors, fonts per brand config)
    - Invoice details (amount, due date, line items)
    - Configured payment methods (iDEAL, card, transfer)
    - Footer: "Powered by Shillinq" (small, discreet)

Scenario: Multi-organization branding (if applicable)
  GIVEN a white-label scenario where Organization A and Organization B use the same instance
  WHEN Organization A's customer accesses Organization A's payment page
  THEN they see Organization A's branding
  AND when Organization B's customer accesses Organization B's payment page
  THEN they see Organization B's branding
```

**Requirements:**

- **REQ-009-001:** Brand configuration MUST be scoped to Organization (IAppConfig or Settings object, not global).
- **REQ-009-002:** Logo upload MUST validate format (PNG, SVG, JPEG) and size (<2 MB).
- **REQ-009-003:** Color values MUST be validated as valid hex codes (#RRGGBB format).
- **REQ-009-004:** Payment page MUST respect NL Design System tokens (primary color from component defaults, secondary customizable).
- **REQ-009-005:** Brand settings MUST be applied consistently across all payment surfaces (invoice checkout, subscription management, receipt email).

---

## Non-Functional Requirements

### Performance

- **REQ-NFR-001:** Dashboard load time: <2 seconds (p95).
- **REQ-NFR-002:** Bank transaction import: process 1,000 transactions in <30 seconds.
- **REQ-NFR-003:** Auto-matching: evaluate 100 rules against 500 transactions in <5 seconds.
- **REQ-NFR-004:** Cash flow forecast generation: complete 13-week calculation in <10 seconds.
- **REQ-NFR-005:** API response time: <500ms (p95) for all endpoints.

### Reliability & Availability

- **REQ-NFR-006:** Bank sync uptime: ≥99.5% (max 2 hours downtime/month).
- **REQ-NFR-007:** Payment submission retry: automatic retry up to 3 times, exponential backoff (1 min, 5 min, 15 min).
- **REQ-NFR-008:** Scheduled payment execution: guaranteed execution within 24 hours of scheduled date (or earlier if batch job runs).

### Security & Compliance

- **REQ-NFR-009:** Bank credentials (Plaid tokens): encrypted at rest (AES-256) and in transit (TLS 1.3).
- **REQ-NFR-010:** Payment audit trail: immutable log of all payments, approvals, executions (tamper-evident via crypto signatures).
- **REQ-NFR-011:** Role-based access control: treasurer can initiate, approver can authorize, finance mgr can view all (PropertyRbacHandler).
- **REQ-NFR-012:** Compliance: conform to Dutch BBV, IV3, SiSa standards (audit trail, dual control, segregation of duties).

### Data Integrity

- **REQ-NFR-013:** Duplicate prevention: no two BankAccounts with same IBAN + accountType.
- **REQ-NFR-014:** Transaction idempotency: Plaid sync MUST NOT create duplicate transactions if re-run on same date range.
- **REQ-NFR-015:** Reconciliation locking: past-month transactions MUST be immutable after close (soft delete only, not update).

### Scalability

- **REQ-NFR-016:** Bank accounts: system MUST support ≥100 connected accounts per organization.
- **REQ-NFR-017:** Transactions: system MUST handle ≥10,000 transactions per month without performance degradation.
- **REQ-NFR-018:** Users: system MUST support ≥50 concurrent users viewing dashboards (read-heavy).

## Cross-Cutting Requirements

- **Audit Trail:** every object change (create, update, delete) logged with user, timestamp, before/after snapshots (automatic via AuditTrailService per ADR-001).
- **Notifications:** users notified of: payment approvals, sync errors, reconciliation status, forecast alerts (via NotificationService).
- **Localization:** all user-facing text translated to Dutch (nl) + English (en); date/currency formatting respects user locale per ADR-007.
- **Accessibility:** WCAG AA compliance — keyboard navigation, labeled forms, color not sole conveyor, alt text on images (ADR-010).
