# Specifications: Treasury & Cash Management

**Change ID:** treasury-cash-management-other-t1  
**Last Updated:** 2026-05-21  

---

## Overview

This document specifies functional and non-functional requirements for treasury and cash management features in Shillinq. All requirements follow the REQ-XXX-NNN format with acceptance criteria in GIVEN/WHEN/THEN format (Gherkin).

---

## 1. Unified Task Inbox

### REQ-TI-001: Display Open Tasks

**Description:** Treasury manager can view all open AP, AR, and payment items in a unified task inbox.

**Acceptance Criteria:**

```gherkin
GIVEN a user with Treasury role
AND the system contains:
  - 5 open invoices (AR) awaiting collection
  - 3 unpaid vendor bills (AP) awaiting payment
  - 2 scheduled payments due in next 7 days
WHEN user navigates to /treasury/tasks
THEN page loads within 2 seconds
AND displays table with columns: Title, Amount, Due Date, Category, Priority, Status
AND shows 10 rows (pagination)
AND each row represents one task aggregated from AR/AP/scheduled payments
AND filtering is available by: status, category, priority, assignee, dueDate range
```

### REQ-TI-002: Task Priority Calculation

**Description:** System automatically prioritizes tasks based on business rules.

**Acceptance Criteria:**

```gherkin
GIVEN an accounts receivable invoice with:
  - dueDate: 60 days ago
  - amount: 10,000 EUR
WHEN system generates TreasuryTask from invoice
THEN task.priority = 'urgent'
AND task.status = 'overdue'

GIVEN a vendor bill with:
  - dueDate: 2026-05-25 (in 4 days)
  - earlyPaymentDiscountDeadline: 2026-05-25 (2% discount)
  - status: approved, unpaid
WHEN system generates TreasuryTask
THEN task.priority = 'high'
AND task.notes include early payment discount details

GIVEN a scheduled payment with:
  - nextDueDate: 2026-06-01 (10 days from now)
  - status: active
WHEN system generates TreasuryTask
THEN task.priority = 'normal'
```

### REQ-TI-003: Task Completion & Reconciliation

**Description:** User marks task complete; system records transaction or updates status.

**Acceptance Criteria:**

```gherkin
GIVEN a TreasuryTask of category 'ap' related to VendorBill #BILL-2026-001
AND amount: 5,000 EUR
WHEN user clicks "Mark Complete" → "Record Payment"
THEN dialog appears with fields:
  - Payment Date
  - Payment Method (SEPA, ACH, Card, etc.)
  - From Account (dropdown of BankAccounts)
  - Reference (pre-filled with bill number)
AND user selects values and clicks "Post Payment"
THEN system creates Payment object:
  - status: posted
  - amount: 5,000 EUR
  - relatedObject: Bill #BILL-2026-001
  - postedAt: today
AND VendorBill.status updates to 'paid'
AND TreasuryTask.status updates to 'completed'
AND entry posted to ledger against VendorBill's GL account
```

### REQ-TI-004: Batch Payment Preparation

**Description:** User can select multiple AP tasks and prepare for batch payment.

**Acceptance Criteria:**

```gherkin
GIVEN task inbox with 15 open AP tasks
WHEN user selects 5 tasks (checkboxes) with total 45,000 EUR
AND clicks "Batch & Schedule"
THEN dialog appears with:
  - Summary: "5 invoices, 45,000 EUR total"
  - Execution date picker
  - Approval required toggle
  - From Account selector
AND user configures values
THEN system creates PaymentBatch object:
  - 5 Payment records aggregated
  - batch.status: 'staged'
  - batch.approvalRequired: true if toggled
  - batch.scheduledDate: selected date
AND dialog shows "Bank file ready to download" (SEPA XML)
AND user can download or proceed to approval workflow
```

---

## 2. Cash Flow Forecasting

### REQ-CF-001: Generate Forecast for Date Range

**Description:** Treasury manager generates cash flow projection for specified period.

**Acceptance Criteria:**

```gherkin
GIVEN a user navigates to /treasury/cash-flow
AND the system has:
  - 20 open invoices (AR) ranging from 1,000-50,000 EUR, due dates 2026-05-25 to 2026-07-31
  - 15 unpaid bills (AP) ranging from 2,000-30,000 EUR, due dates 2026-05-24 to 2026-06-30
  - 3 scheduled payments: Rent (8,500 EUR, 2026-06-01), Insurance (4,200 EUR, 2026-06-15), SaaS (900 USD, 2026-06-05)
  - Starting cash balance: 250,000 EUR
WHEN user selects:
  - Start Date: 2026-05-21
  - End Date: 2026-08-19 (90 days)
  - Bucket Size: Week
  - Scenario: Base
THEN page displays forecast with:
  - 13 rows (13 weeks)
  - Each row shows: Period, Expected Inflows, Expected Outflows, Net Cash, Cumulative Balance
  - Calculations apply 90% AR collection rate (default)
  - Calculations apply 100% AP payment rate on due date
  - Each scheduled payment included on schedule date
  - Cumulative balance starts at 250,000 and adjusts with each period
```

### REQ-CF-002: Multi-Scenario Forecasting

**Description:** System displays pessimistic, base, and optimistic scenarios.

**Acceptance Criteria:**

```gherkin
GIVEN a forecast scenario selection with 3 options: Pessimistic, Base, Optimistic
WHEN user selects each scenario
THEN Base scenario displays with:
  - 90% AR collection rate
  - 100% AP payment on due date
AND Pessimistic scenario displays with:
  - 70% AR collection rate (20% reduction)
  - 100% AP payment on due date
AND Optimistic scenario displays with:
  - 100% AR collection rate
  - 100% AP payment on due date
AND all three scenarios show cumulative balance projection
AND variance between scenarios visible on chart (shaded confidence band)
```

### REQ-CF-003: Forecast Assumptions Display

**Description:** User can see and override forecast assumptions.

**Acceptance Criteria:**

```gherkin
GIVEN forecast page displays in Base scenario
WHEN user clicks "Assumptions" or "Advanced"
THEN panel displays:
  - AR Collection Rate: 90% (with edit button)
  - AP Payment Rate: 100% (with edit button)
  - Include Scheduled Payments: checked
  - Include Other Income: unchecked
  - FX Rate Source: ECB (with dropdown for manual)
  - Notes: (configurable assumptions)
AND user can edit each field
AND clicking "Apply" regenerates forecast with new assumptions
AND previous forecast remains available for comparison
```

### REQ-CF-004: Export Forecast

**Description:** User exports forecast data for reporting or external analysis.

**Acceptance Criteria:**

```gherkin
GIVEN forecast page is displayed
WHEN user clicks "Export" or "Download"
THEN dialog appears with options:
  - Format: CSV | Excel | PDF
  - Include: Periods | Assumptions | Notes
WHEN user selects options and clicks "Download"
THEN file is generated and downloaded with:
  - File name: cash_flow_forecast_2026-05-21.xlsx (or .csv, .pdf)
  - Content: Forecast table with all periods and scenarios
  - Metadata: Generated date, assumptions, confidence level
  - PDF includes chart visualization
```

---

## 3. Multi-Currency & FX Exposure

### REQ-FX-001: Calculate & Display FX Exposure

**Description:** System aggregates foreign exchange exposure by currency pair.

**Acceptance Criteria:**

```gherkin
GIVEN system contains:
  - AR invoice in USD: 50,000 (due 2026-06-30)
  - AR invoice in USD: 25,000 (due 2026-07-15)
  - AP bill in USD: 20,000 (due 2026-06-15)
  - Bank balance in USD: 85,000 (in ING USD account)
  - Scheduled payment in USD: 900 (due 2026-06-05, SaaS subscription)
WHEN user navigates to /treasury/fx-exposure
THEN page displays FX Exposure for EUR/USD pair:
  - Total Exposure: 140,900 USD (50k + 25k + 20k + 900 + 85k balance)
  - Maturity Buckets:
    - 0-30 days: 105,900 USD
    - 31-90 days: 25,000 USD
    - 91+ days: 10,000 USD
  - Current Market Rate: 0.9229
  - Unrealized Gain/Loss: calculated at current rate
  - Hedge Status: 0 hedges currently active
```

### REQ-FX-002: Hedge Management

**Description:** User can record FX hedges (forwards, options) against exposure.

**Acceptance Criteria:**

```gherkin
GIVEN FX Exposure for EUR/USD showing 100,000 USD exposure
WHEN user clicks "Add Hedge"
THEN dialog appears with fields:
  - Hedge Type: Forward | Option | Swap
  - Amount: (input field, e.g., 50,000 USD)
  - Rate: (input field, e.g., 0.9150)
  - Settlement Date: (date picker)
WHEN user fills and clicks "Create Hedge"
THEN Hedge object created and FX Exposure updates:
  - coveredAmount: 50,000 USD
  - uncovered portion: 50,000 USD (100k - 50k)
  - unrealizedGain recalculates: (0.9229 - 0.9150) * 50,000 = €397.50
AND Hedge Status table displays new hedge with coverage %
```

### REQ-FX-003: Currency Balance by Account

**Description:** Dashboard shows balance in each currency per bank account.

**Acceptance Criteria:**

```gherkin
GIVEN user navigates to /treasury/accounts
THEN table displays:
  - Account Name | Balance | Currency | Equivalent EUR | Last Sync
  - Row 1: ABN AMRO Checking | 245,000 | EUR | 245,000 EUR | 2026-05-21 14:30
  - Row 2: ING USD Reserve | 85,000 | USD | 78,450 EUR | 2026-05-21 13:45
  - Row 3: Triodos GBP | 42,500 | GBP | 50,300 EUR | 2026-05-21 12:00
AND footer shows:
  - Total Consolidated (EUR): 373,750 EUR
  - Total USD Exposure: 85,000 USD
  - Total GBP Exposure: 42,500 GBP
AND FX rates displayed: 1 EUR = 0.9229 USD, 1 GBP = 1.1835 EUR
AND rates sourced from ECB updated at [timestamp]
```

### REQ-FX-004: Multi-Currency Ledger Posting

**Description:** When posting payment in foreign currency, system handles conversion.

**Acceptance Criteria:**

```gherkin
GIVEN a Payment to be posted:
  - Amount: 25,000 USD
  - From Account: ING USD Reserve (currency: USD)
  - Invoice: AR invoice in USD
WHEN user posts payment
THEN system:
  - Posts Payment as 25,000 USD
  - Looks up FX rate (e.g., 0.9229 from market)
  - Calculates EUR equivalent: 25,000 * 0.9229 = 23,072.50 EUR
  - Posts dual-currency JournalEntry:
    - Debit: Bank Account (USD) 25,000 USD
    - Debit: FX Gain/Loss (if applicable)
    - Credit: AR account 23,072.50 EUR (or AUD, original currency)
  - Records exchange rate in Payment.exchangeRateUsed
  - Updates CurrencyBalance for USD account
```

---

## 4. Bank Account Management

### REQ-BA-001: Add Bank Account

**Description:** User registers a new bank account in the system.

**Acceptance Criteria:**

```gherkin
GIVEN user is Treasury Manager with permissions to manage accounts
AND user navigates to /treasury/accounts
WHEN user clicks "Add Account"
THEN form appears with fields:
  - Account Name (text, required)
  - IBAN (text, required, validated against IBAN checksum)
  - Bank Name (text, required)
  - Country (dropdown)
  - Currency (dropdown, ISO 4217)
  - GL Account (relationship, required)
  - Sync Options (toggle: "Connect to bank sync")
WHEN user fills form and clicks "Save"
THEN BankAccount object created:
  - status: active
  - syncStatus: pending (if sync enabled) or synced (manual)
  - balance: 0 (until first sync)
THEN account appears in accounts list
AND if sync enabled, background job initiates first sync
```

### REQ-BA-002: Sync Transactions from Bank

**Description:** System retrieves transactions from bank API and matches to ledger.

**Acceptance Criteria:**

```gherkin
GIVEN BankAccount with syncStatus: pending
WHEN background job runs (e.g., daily at 06:00)
AND bank API (e.g., Plaid) is available
THEN system:
  - Fetches transactions since lastSyncAt or last 90 days
  - For each transaction:
    - Creates/updates Transaction object with:
      - bankTxnId (unique ID from bank)
      - amount, currency, date, description, merchant
      - status: unmatched (initially)
  - Attempts to auto-match each transaction:
    - Looks for JournalEntry with same amount ± 0.01, date within 3 days
    - If exact match found: marks Transaction.status = matched
    - If no match: status = unmatched (requires manual reconciliation)
  - Updates BankAccount.balance from bank response
  - Updates BankAccount.syncStatus = synced
  - Updates BankAccount.lastSyncAt = today
AND notifies user if unmatched transactions exist
```

### REQ-BA-003: Bank Reconciliation

**Description:** User manually matches unmatched transactions.

**Acceptance Criteria:**

```gherkin
GIVEN /treasury/reconciliation page displays unmatched transactions
AND 5 unmatched bank transactions displayed:
  - Transaction 1: 150 EUR (2026-05-20), "COFFEE SHOP"
  - Transaction 2: 5,000 USD (2026-05-19), "SUPPLIER XYZ"
  - Transaction 3: 2,500 EUR (2026-05-20), "INSURANCE"
  - [...]
WHEN user clicks transaction 2 → 5,000 USD
THEN side panel shows:
  - Transaction details
  - "Unmatched JournalEntries" list showing similar amount/date range:
    - JE: AP Bill #2026-001, 5,000 USD, 2026-05-19, Supplier XYZ
    - JE: Expense, 4,980 USD, 2026-05-21, Unknown Vendor
WHEN user clicks "Match" on first JE
THEN system:
  - Links Transaction to JournalEntry
  - Updates Transaction.status = matched
  - Updates reconciliation status
AND page refreshes, transaction no longer in unmatched list
```

---

## 5. Subscription & Recurring Payments

### REQ-SR-001: Create Scheduled Payment

**Description:** User sets up recurring payment (monthly, quarterly, annual, etc.).

**Acceptance Criteria:**

```gherkin
GIVEN user navigates to /treasury/payments → "Scheduled"
WHEN user clicks "Add Scheduled Payment"
THEN form appears with fields:
  - Description: (text, e.g., "Office Rent")
  - Payee: (relationship, required)
  - Amount: (decimal, required)
  - Currency: (dropdown)
  - Schedule: (radio: weekly | monthly | quarterly | annual)
  - First Payment Date: (date)
  - Approval Required: (checkbox)
  - From Account: (dropdown)
  - End Date: (optional date, for finite schedules)
WHEN user fills and clicks "Create"
THEN ScheduledPayment created with status: active
AND nextDueDate calculated based on schedule
AND system stores in CRON/background job queue
```

### REQ-SR-002: Execute Scheduled Payment

**Description:** System executes payment on scheduled date.

**Acceptance Criteria:**

```gherkin
GIVEN ScheduledPayment:
  - Description: "Rent"
  - Amount: 8,500 EUR
  - Schedule: monthly
  - nextDueDate: 2026-06-01
  - status: active
WHEN background job runs on 2026-06-01 at 09:00
THEN system:
  - Creates Payment object:
    - amount: 8,500 EUR
    - relatedObject: ScheduledPayment
    - status: initiated (if approvalRequired: false) or pending_approval (if true)
    - executionDate: 2026-06-01
  - Submits to bank (if no approval required) or routes to approval chain
  - Updates ScheduledPayment.lastExecutedAt = 2026-06-01
  - Calculates nextDueDate: 2026-07-01
AND user receives notification of payment execution
AND audit trail records payment with full trace
```

### REQ-SR-003: Pause / Resume / Cancel Scheduled Payment

**Description:** User can manage lifecycle of scheduled payments.

**Acceptance Criteria:**

```gherkin
GIVEN ScheduledPayment is active
WHEN user navigates to scheduled payment detail
AND clicks "Pause"
THEN dialog asks "Pause until when?" with date picker
WHEN user selects date and confirms
THEN ScheduledPayment.status = paused
AND nextDueDate unchanged but execution skipped until unpause
AND background job ignores paused payments

WHEN user clicks "Resume"
THEN ScheduledPayment.status = active
AND nextDueDate recalculated (if past)

WHEN user clicks "Cancel"
THEN dialog asks "Confirm cancellation?"
WHEN confirmed:
  - ScheduledPayment.status = cancelled
  - endDate set to today
  - Audit trail records cancellation reason
AND payment no longer executes
```

---

## 6. Payment Approval Workflow

### REQ-PW-001: Configure Approval Chain

**Description:** Admin can define approval sequences for payments.

**Acceptance Criteria:**

```gherkin
GIVEN user is admin (has treasury settings permission)
AND navigates to /settings/treasury → "Approval Workflows"
WHEN user clicks "New Approval Chain"
THEN form appears:
  - Chain Name: (text, e.g., "Large Payments")
  - Trigger Conditions: (e.g., "amount >= 10,000 EUR")
  - Approvers: (ordered list of Users/Roles)
    - Approver 1: Treasury Manager
    - Approver 2: CFO
    - [+ Add Approver]
  - Required Approvals: (radio: Sequential | All Required)
  - Escalation: (time in hours after which escalate to next approver)
WHEN user clicks "Save"
THEN ApprovalChain stored
AND applies to PaymentBatch or Payment when trigger matches
```

### REQ-PW-002: Approval Request Notification

**Description:** Approver receives notification and can approve/reject payment.

**Acceptance Criteria:**

```gherkin
GIVEN PaymentBatch created with approvalRequired: true
AND Approval Chain configured: [Treasury Manager → CFO]
WHEN PaymentBatch.status = pending_approval
THEN system sends notification to Treasury Manager:
  - Subject: "Approval Needed: 5 Invoices, 45,000 EUR"
  - Link to approval page with details:
    - List of 5 invoices with amounts
    - Total: 45,000 EUR
    - Approval buttons: [Approve] [Reject]
WHEN Treasury Manager clicks "Approve"
THEN:
  - ApprovalRequest.status = approved_by_treasury_manager
  - timestamp = now
  - Forwarded to CFO (next in chain)
  - CFO receives notification
WHEN CFO clicks "Approve"
THEN:
  - PaymentBatch.status = approved
  - All Payment.status in batch = approved
  - System can now submit to bank
AND notification sent to initiator: "Batch approved, submitting to bank"
```

### REQ-PW-003: Rejection & Return to Draft

**Description:** Approver can reject payment with reason; returns to initiator.

**Acceptance Criteria:**

```gherkin
GIVEN PaymentBatch pending approval
WHEN Approver clicks "Reject"
THEN dialog appears with:
  - Reason: (required text field)
  - Comments: (optional)
WHEN approver enters reason and clicks "Reject"
THEN:
  - PaymentBatch.status = rejected
  - ApprovalRequest.status = rejected_reason = [text]
  - Notification sent to initiator with reason
  - Initiator can edit batch (remove/add payments, change date)
  - And resubmit for approval
```

---

## 7. Bank Connectivity & Payment Submission

### REQ-BP-001: Submit Payment to Bank

**Description:** Approved payment batch submitted to bank for posting.

**Acceptance Criteria:**

```gherkin
GIVEN PaymentBatch:
  - status: approved
  - 3 payments: 5,000 EUR (SEPA), 10,000 EUR (SEPA), 2,500 EUR (SEPA)
  - total: 17,500 EUR
  - executionDate: 2026-05-25
WHEN user clicks "Submit to Bank"
THEN system:
  - Generates SEPA XML file (pain.001 format) with:
    - All 3 payments as CTTransfer entries
    - Debtor: Company details
    - Creditor: Each payee details
    - Reference: Payment ID
  - Submits to bank API (e.g., Plaid) or downloads for manual upload
  - Updates Payment.status = initiated
  - Updates PaymentBatch.status = submitted
  - Records submission timestamp and batch reference from bank
AND user sees confirmation: "Batch submitted successfully. Bank Reference: [ref]"
AND audit trail records submission with file content hash
```

### REQ-BP-002: Track Payment Status

**Description:** User can monitor payment progress through bank.

**Acceptance Criteria:**

```gherkin
GIVEN PaymentBatch submitted to bank on 2026-05-25
WHEN user navigates to /treasury/payments → "Submitted"
THEN table shows:
  - Batch Date | Payments | Amount | Status | Bank Reference
  - Row: 2026-05-25 | 3 | 17,500 EUR | In Transit | [BANK-REF-123]
WHEN user clicks row
THEN detail panel displays:
  - Individual payment status:
    - Payment 1: 5,000 EUR → Status: Posted (2026-05-25 18:00)
    - Payment 2: 10,000 EUR → Status: Posted (2026-05-25 18:05)
    - Payment 3: 2,500 EUR → Status: Posted (2026-05-25 18:10)
  - Batch status: 3/3 posted successfully
  - Download receipt/confirmation from bank if available
```

### REQ-BP-003: Failed Payment Handling

**Description:** System handles payment rejections or errors from bank.

**Acceptance Criteria:**

```gherkin
GIVEN PaymentBatch submitted to bank
AND one payment fails due to:
  - Insufficient funds at payee's bank
  - Invalid account number
  - Fraud detection hold
WHEN bank API returns error response
THEN system:
  - Updates Payment.status = failed
  - Payment.failureReason = error message from bank
  - Creates TreasuryTask: "Investigate Failed Payment: [Payment ID]"
  - Assigns to original submitter
  - Notifies treasury team
WHEN treasury team investigates and resolves issue:
  - Edits Payment details (if account number incorrect)
  - OR confirms account has funds
  - Clicks "Resubmit"
THEN system resubmits Payment to bank
AND audit trail records all attempts and resolutions
```

---

## 8. Non-Functional Requirements

### REQ-NFR-001: Performance

**Requirement:** Task inbox loads with <2s response time for up to 10,000 items (paginated).

```gherkin
GIVEN 10,000 TreasuryTask records in database
WHEN user navigates to /treasury/tasks?page=1&limit=50
AND network latency: 50ms, database on same region
THEN page loads within 2 seconds
AND initial table displays 50 rows
AND pagination controls show total count
```

### REQ-NFR-002: Data Integrity

**Requirement:** All payment transactions have immutable, complete audit trail.

```gherkin
GIVEN PaymentBatch submitted and executed
WHEN user navigates to audit trail for that batch
THEN complete history visible:
  - Created: 2026-05-21 10:30 by User A (add 5 payments)
  - Modified: 2026-05-21 11:15 by User B (remove 1 payment, edit amount)
  - Submitted: 2026-05-21 15:00 by User A (submit to bank)
  - Approved: 2026-05-21 15:30 by CFO
  - Confirmed Posted: 2026-05-25 18:00 (from bank sync)
AND before/after snapshot captured for each change
AND user cannot delete or edit audit trail
```

### REQ-NFR-003: Reconciliation Accuracy

**Requirement:** Bank statement reconciliation must achieve 95%+ auto-match rate.

```gherkin
GIVEN 1,000 bank transactions from recent 30 days
WHEN bank sync completes and auto-matching runs
THEN 950+ transactions auto-matched to JournalEntries
AND <50 transactions require manual reconciliation
AND unmatched transactions visible in reconciliation UI for user review
```

### REQ-NFR-004: PCI-DSS Compliance

**Requirement:** Payment/bank data must not be stored in logs or responses.

```gherkin
GIVEN Payment with sensitive data:
  - Payee account number: NL91ABNA0417164300
  - Bank routing code
WHEN payment processed and logged
THEN logs contain:
  - Payment ID, amount, currency, status
  - Masked account: NL91****7164300 (last 4 visible)
  - NO full account number, routing code, or clearinghouse info
AND API responses (even error responses) contain no PII
```

---

## 9. Security & Authorization

### REQ-SEC-001: Role-Based Access Control

**Requirement:** Features restricted by role (Treasury Manager, CFO, Accountant, etc.).

```gherkin
GIVEN roles defined:
  - Treasury Manager: Read/Write tasks, view forecasts, submit payments
  - CFO: Read all, approve payments >10K EUR, configure workflows
  - Accountant: Read/Write AP/AR, reconciliation
  - Bank Liaison: Read/Write bank accounts, bank sync
WHEN user without Treasury Manager role attempts to submit payment
THEN system:
  - Returns 403 Forbidden
  - Logs unauthorized access attempt
  - Notifies admin

WHEN user with Treasury Manager role accesses /treasury/payments
THEN can view and submit
```

### REQ-SEC-002: Approval Authority Enforcement

**Requirement:** Payment approval chain cannot be bypassed.

```gherkin
GIVEN PaymentBatch with approvalRequired: true
AND Approval Chain: [Treasury Manager → CFO]
WHEN PaymentBatch.status = pending_approval (awaiting Treasury Manager)
THEN:
  - Only Treasury Manager (or higher privilege) can change status
  - Submitting to bank without approval rejected with error
  - Audit trail records all access attempts
```

---

## 10. Integration Requirements

### REQ-INT-001: Bank API Integration (TBD Provider)

**Requirement:** System must support bank transaction import and payment submission.

```gherkin
GIVEN BankAccount configured with sync enabled
AND bank API provider selected (e.g., Plaid, TrueLayer)
WHEN background sync job runs
THEN system calls provider API:
  - POST /auth/login with credentials (encrypted)
  - GET /accounts/{accountId}/transactions?since=[date]
  - Handles pagination for large result sets
  - Handles rate limits (retry with backoff)
  - Processes response and creates Transaction objects
```

### REQ-INT-002: FX Rate Feed Integration (TBD Provider)

**Requirement:** Daily FX rates updated for multi-currency reporting.

```gherkin
GIVEN FX Rate Feed configured (ECB, Xe.com API, etc.)
WHEN daily scheduler runs at 08:00
THEN system:
  - Calls rate provider API for major pairs (EUR/USD, EUR/GBP, etc.)
  - Stores rates with timestamp
  - Uses rates for auto-conversion in forecasts and reporting
  - Falls back to previous day's rate if feed unavailable
  - Allows manual rate override for specific dates
```

### REQ-INT-003: Payment Gateway Integration (TBD Provider)

**Requirement:** Support multiple payment methods across geographies.

```gherkin
GIVEN supported payment methods configured:
  - SEPA Credit Transfer (Europe)
  - ACH (USA)
  - Wise API (International)
WHEN PaymentBatch submitted with method: 'SEPA'
THEN system:
  - Generates SEPA XML (pain.001)
  - Submits to configured payment gateway
  - Tracks submission and settlement
  - Receives confirmation and reference number

WHEN method: 'ACH'
THEN system:
  - Generates ACH file (Nacha format)
  - Submits to ACH provider
  - [similar flow]
```

---

## 11. Acceptance Criteria Summary

All requirements must satisfy:

✓ Functional correctness per GIVEN/WHEN/THEN scenarios  
✓ Performance SLAs (2s task load, <30s forecast generation)  
✓ Data integrity (100% audit trail, zero orphaned records)  
✓ Security (RBAC, approval enforced, no PII in logs)  
✓ Compliance (GDPR, PCI-DSS, audit trail retention)  
✓ Usability (task inbox primary workflow, <3 clicks to approve payment)  

---

