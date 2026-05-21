# Specifications: Treasury & Cash Management — Shillinq — Other T4

**Status:** Specification  
**Change:** treasury-cash-management-other-t4  
**Last Updated:** 2026-05-21

---

## Overview

Detailed requirements for 12 treasury features spanning bank reconciliation, cash flow forecasting, loan analysis, payment optimization, and compliance monitoring. All requirements are testable via manual browser tests (feature verification) and API integration tests (automated validation).

---

## Bank Reconciliation

### REQ-001-001: Import bank statements in MT940 format

**Summary:** Users can upload bank statements in SEPA MT940 format and the system automatically parses and stores entries.

**Details:**
- Support MT940 standard (ISO 20022 predecessor)  
- Parse fields: transaction date, amount, currency, counterparty, reference, transaction code  
- Validate IBAN/BIC format in counterparty information  
- Store unparseable lines with error log for manual review  
- Transaction deduplication: skip if entry with same date + amount + reference already exists  

**Test Scenarios:**

**REQ-001-001-A: Successful MT940 import with valid entries**

GIVEN: A bank statement file in MT940 format with 5 valid SEPA transfers  
WHEN: I click "Import" and select the file  
THEN: The system displays "Import complete: 5 entries imported, 0 errors"  
AND: Each entry is visible in the "Unreconciled" section  
AND: Entry fields show date, amount (with sign), counterparty name, reference  

**REQ-001-001-B: MT940 import with unparseable lines**

GIVEN: An MT940 file with 1 malformed line (invalid amount format)  
WHEN: I import the file  
THEN: The system displays "Import complete: 4 entries imported, 1 error"  
AND: The error log shows the malformed line with reason ("Invalid amount format in field 32A")  
AND: Valid entries are still imported and visible  

**REQ-001-001-C: Duplicate transaction detection**

GIVEN: A previous import with entry (2026-05-15, EUR 500, ref "INV-001")  
WHEN: I import a file containing the same entry again  
THEN: The system shows "1 duplicate skipped"  
AND: Only 1 new entry is added (deduplication works)  
AND: Audit trail shows "Duplicate entry skipped"  

---

### REQ-001-002: Automatic bank entry matching to ledger entries

**Summary:** The system automatically matches imported bank entries to ledger entries using configurable matching rules.

**Details:**
- Match strategies (in order of preference):
  1. Exact: amount + date + reference (confidence 0.99)  
  2. Fuzzy amount: amount ±5% + date ±3 days (confidence 0.7)  
  3. Counterparty fuzzy: counterparty name + amount + date ±5 days (confidence 0.5)  
- User-configurable threshold (default 0.7): mark entries below threshold as "manual review required"  
- Learn from manual matches: if user matches entry to ledger repeatedly (3+ times), auto-suggest for future similar entries  
- Cross-check: if multiple ledger entries match a bank entry, prompt user to select  

**Test Scenarios:**

**REQ-001-002-A: Exact match (confidence 0.99)**

GIVEN: Bank entry (2026-05-21, EUR 2500, ref "INV-00451")  
AND: Ledger entry (2026-05-21, EUR 2500, description "Invoice 00451")  
WHEN: I view the reconciliation dashboard  
THEN: The bank entry shows "Matched" with green checkmark  
AND: Confidence badge shows "99%"  
AND: Matched ledger entry is linked and displayed  

**REQ-001-002-B: Fuzzy amount match (confidence 0.7)**

GIVEN: Bank entry (2026-05-21, EUR 2500)  
AND: Ledger entry (2026-05-19, EUR 2550, same reference)  
WHEN: I view unreconciled entries with threshold set to 0.7  
THEN: The system shows "Possible match: 70% confidence"  
AND: I can confirm or reject the match  
AND: If confirmed, the entry is marked reconciled with confidence 0.7  

**REQ-001-002-C: Learning from manual matches**

GIVEN: I manually match supplier "Zorgdiensten X" bank entries to ledger entries 4 times  
WHEN: A new bank entry arrives from the same supplier with same reference pattern  
THEN: The system auto-suggests the match with confidence 0.8  
AND: User can confirm or override  

---

### REQ-001-003: Manual reconciliation UI for unmatched entries

**Summary:** Finance controllers can manually match bank entries to ledger entries via an intuitive search-and-confirm dialog.

**Details:**
- Unmatched entries list: sortable by date, amount, counterparty  
- "Match" button opens dialog with ledger entry search (by date range, amount, reference, account)  
- Show matching score for each candidate  
- Bulk reconciliation: select multiple bank entries, get suggestions, approve all at once  
- Undo reconciliation: user can revert a match (within 7 days) and re-assign  
- Audit trail: record who matched, when, and original confidence  

**Test Scenarios:**

**REQ-001-003-A: Manual match via search dialog**

GIVEN: An unmatched bank entry (EUR 8750, ref "ORDER-123")  
WHEN: I click "Match"  
THEN: A dialog opens with ledger entry search  
AND: I search by reference "ORDER-123"  
AND: System shows 1 matching ledger entry with 95% confidence  
AND: I click "Confirm"  
THEN: Entry is marked reconciled  
AND: Audit trail shows "[user] matched entry at 2026-05-21 10:30 UTC"  

**REQ-001-003-B: Bulk reconciliation of multiple entries**

GIVEN: 10 unmatched bank entries  
AND: 8 of them have high-confidence auto-suggestions (≥0.8)  
WHEN: I click "Auto-Match All"  
THEN: System matches all 8 automatically  
AND: 2 entries remain unmatched (below threshold)  
AND: I can review and confirm the 8 matches in a single dialog  

**REQ-001-003-C: Undo reconciliation within 7 days**

GIVEN: A reconciled entry matched 2 days ago  
WHEN: I click "Undo" on the matched entry  
THEN: The entry reverts to unmatched status  
AND: Audit trail logs "Reconciliation undone by [user] at 2026-05-21 14:00 UTC"  

---

### REQ-001-004: Reconciliation status dashboard

**Summary:** Finance controllers see high-level reconciliation status and drill-down to problem areas.

**Details:**
- Summary card per account: total entries, matched count, match rate %, last reconciled timestamp  
- Trend chart: match rate over past 30 days  
- Unmatched entries breakdown: by amount range, age (0-3 days, 4-7 days, >7 days)  
- Filter options: by account, date range, amount range, counterparty  
- Export: CSV with all entries, reconciliation status, matched ledger entry  

**Test Scenarios:**

**REQ-001-004-A: View reconciliation status for all accounts**

GIVEN: 3 bank accounts with reconciliation data  
WHEN: I view the reconciliation dashboard  
THEN: I see 3 status cards showing:
- Account name and IBAN
- Match rate: 95.5%, 92.3%, 87.8%
- Unmatched count: 22, 12, 35
- Last reconciled timestamp  

**REQ-001-004-B: Filter and drill-down to old unmatched entries**

GIVEN: Unmatched entries >7 days old  
WHEN: I click on "Unmatched >7 days" segment in the breakdown chart  
THEN: System filters the list to show only entries older than 7 days  
AND: I can see counterparty names and amounts to investigate  

**REQ-001-004-C: Export reconciliation report**

GIVEN: Reconciliation data for a 30-day period  
WHEN: I click "Export CSV"  
THEN: A file is downloaded with columns:
- Date, Amount, Counterparty, Reference, Status (matched/unmatched), Matched Entry ID, Match Rate  

---

## Cash Flow Forecasting

### REQ-002-001: Compute 30/60/90-day cash flow forecast

**Summary:** System projects cash position based on committed (outstanding) invoices and bills, with user-adjustable horizon.

**Details:**
- Input: outstanding AR, AP, scheduled payments, current bank balances  
- Projection method: committed transactions only (no probabilistic forecasting at this stage)  
- Daily granularity (not monthly)  
- Assumptions: invoices due on due date, not on issue date; scheduling delays assumed 0 unless specified  
- Export: CSV with opening balance, inflows, outflows, closing balance per day  
- Adjustments: allow manual entry of one-off transactions (e.g., dividend payout, loan disbursement)  

**Test Scenarios:**

**REQ-002-001-A: 30-day forecast with committed transactions**

GIVEN: Current balance EUR 170,000  
AND: Outstanding AR: EUR 50,000 due 2026-05-25  
AND: Outstanding AP: EUR 35,000 due 2026-05-22  
AND: Scheduled payment: EUR 5,000 weekly (next due 2026-05-28)  
WHEN: I view the 30-day cash flow forecast as of 2026-05-21  
THEN: Forecast shows:
- 2026-05-21 opening: EUR 170,000
- 2026-05-22 outflow: EUR 35,000 → closing: EUR 135,000
- 2026-05-25 inflow: EUR 50,000 → closing: EUR 185,000
- 2026-05-28 outflow: EUR 5,000 → closing: EUR 180,000
- ... (continue through 2026-06-20)  

**REQ-001-001-B: 90-day forecast with manual adjustment**

GIVEN: A base 90-day forecast  
AND: A planned dividend payout of EUR 30,000 on 2026-06-15  
WHEN: I add the dividend as a manual adjustment  
THEN: Forecast updates  
AND: 2026-06-15 closing balance is EUR 30,000 lower than before  
AND: Subsequent days reflect the reduced balance  

**REQ-002-001-C: Forecast accuracy vs. actual settlement**

GIVEN: A forecast generated on 2026-05-21 for the following 30 days  
AND: Actual settlements occur as projected  
THEN: At 2026-06-21, compare forecast closing balance vs. actual balance  
AND: Variance should be <5% (or document exceptions)  

---

### REQ-002-002: Visual cash flow chart with horizon selector

**Summary:** Interactive line chart shows cumulative cash position over the selected horizon.

**Details:**
- Chart type: line chart with stacked area for inflows/outflows  
- Horizon selector: buttons for 30, 60, 90 days  
- Interactivity: hover to see exact amount and date; click date to drill down to transactions  
- Anomalies highlighted: if closing balance dips below minimum reserve, show warning zone (red)  
- Print-friendly: chart is embeddable in PDF reports  

**Test Scenarios:**

**REQ-002-002-A: Horizon selector and chart update**

GIVEN: A 30-day forecast is displayed  
WHEN: I click the "90 days" button  
THEN: Chart updates to show 90-day horizon  
AND: X-axis extends to 2026-08-19  
AND: Axis scales adjust accordingly  

**REQ-002-002-B: Drill-down from chart to transactions**

GIVEN: A forecast chart with a dip on 2026-05-28  
WHEN: I click on the 2026-05-28 point  
THEN: A popover or detail panel shows:
- Inflows: 0 (with breakdown by AR)
- Outflows: 5,000 (with breakdown by AP + scheduled payments)
- Transactions: list of the 3 items driving the outflow  

**REQ-002-002-C: Reserve threshold visualization**

GIVEN: Minimum reserve set to EUR 50,000  
AND: Forecast shows balance dipping to EUR 45,000 on 2026-06-10  
WHEN: I view the chart  
THEN: 2026-06-10 is highlighted in red/orange  
AND: Legend shows "Below minimum reserve"  

---

## Loan Refinancing

### REQ-003-001: Identify loans eligible for refinancing with market rate comparison

**Summary:** System analyzes existing debt and compares to current market rates, highlighting refinancing opportunities.

**Details:**
- Data source: Loan entity (principal, current rate, term, maturity)  
- Market rate lookup: configurable table mapping term + risk profile → market rate  
- Eligibility threshold: refinancing score > 1.0 (savings justify costs)  
- Cost assumption: refinancing costs = 0.5% of principal (estimate, configurable)  
- Sorting: by annual savings (highest first)  
- Details view: lender options (placeholder for Phase 2 integration with broker APIs)  

**Test Scenarios:**

**REQ-003-001-A: Identify single eligible loan**

GIVEN: Loan with principal EUR 100,000, current rate 5.5%, term 5 years, maturity 2030-06-15  
AND: Market rate for 5-year term: 4.2%  
AND: Rate delta: 1.3% → annual savings: EUR 1,300  
AND: Refinancing cost (0.5%): EUR 500  
AND: Refinancing score: 1.3 / 0.5 = 2.6 (>1.0, eligible)  
WHEN: I view the refinancing analysis  
THEN: The loan is listed as eligible  
AND: Estimated annual savings shows EUR 1,300  
AND: Green badge indicates "Recommended"  

**REQ-003-001-B: Filter ineligible loans**

GIVEN: 3 loans, 1 with refinancing score 0.8 (ineligible)  
WHEN: I view the analysis with filter "Eligible only"  
THEN: Only 2 loans are shown  
AND: The ineligible loan is hidden (or grayed out with reason "Score 0.8 < 1.0")  

**REQ-003-001-C: Export refinancing report**

GIVEN: A refinancing analysis with 2 eligible loans  
WHEN: I click "Export CSV"  
THEN: Downloaded file contains:
- Loan ID, Principal, Current Rate, Market Rate, Rate Delta, Annual Savings, Refinancing Score, Maturity Date  

---

## Payment Verification & Fraud Prevention

### REQ-004-001: Verify supplier IBAN validity before payment

**Summary:** System validates IBAN format and flags inconsistencies with prior payments to prevent fraud.

**Details:**
- IBAN validation: per ISO 13616 (checksum, length, country code)  
- SEPA eligibility check: is country in SEPA zone (EU27 + others)  
- Consistency check: compare new IBAN vs. supplier's previous payment IBANs  
- Change detection: if IBAN differs from last 3 payments, flag for manual review  
- Audit trail: record verification timestamp, verifier ID, result  

**Test Scenarios:**

**REQ-004-001-A: Valid IBAN passes verification**

GIVEN: Supplier "Zorgdiensten de Toekomst" with IBAN "NL75ABNA0588564432"  
WHEN: I initiate a payment to this supplier  
THEN: System validates the IBAN  
AND: Display shows "IBAN valid ✓"  
AND: Bank name auto-fills: "ABN AMRO"  
AND: Verification badge shows checkmark  

**REQ-004-001-B: Invalid IBAN is rejected**

GIVEN: A supplier IBAN "NL75ABNA058856443X" (invalid checksum)  
WHEN: I try to save the supplier record  
THEN: System shows error: "Invalid IBAN checksum. Please verify."  
AND: Payment cannot be initiated until IBAN is corrected  

**REQ-004-001-C: IBAN change detection flags for review**

GIVEN: Supplier previously paid with IBAN "NL75ABNA0588564432"  
AND: A new payment is initiated with IBAN "NL75ABNA0588564433" (different)  
WHEN: I view the payment approval dialog  
THEN: A warning appears: "IBAN differs from last 3 payments. Please verify with supplier."  
AND: Payment requires manual controller override to proceed  
AND: Override is logged in audit trail  

**REQ-004-001-D: IBAN verification history**

GIVEN: A supplier with multiple verification records  
WHEN: I view the supplier detail page  
THEN: Verification history shows:
- Date verified, IBAN, verified by (user), status (valid/invalid)  
- All historic IBANs are displayed chronologically  

---

## Payment Optimization & Netting

### REQ-005-001: Identify payment netting opportunities

**Summary:** System analyzes mutual payables and receivables with the same counterparty and suggests offsetting them.

**Details:**
- Scope: current administration only (multi-admin netting is Phase 2)  
- Payables + Receivables: within configurable horizon (default 90 days)  
- Matching: group by supplier, amount, currency  
- Offset calculation: min(payable, receivable) = amount to offset  
- Savings estimate: banking fees (SEPA fee %) + interest (daily rate × days / 365)  
- Execution: user confirms, system creates contra-journal entry, marks both transactions settled  

**Test Scenarios:**

**REQ-005-001-A: Identify mutual payables and receivables**

GIVEN: Payable to supplier X: EUR 5,000 (due 2026-05-25)  
AND: Receivable from supplier X: EUR 3,200 (due 2026-05-28)  
WHEN: I view netting analysis  
THEN: System shows:
- Offset amount: EUR 3,200
- Net payment required: EUR 1,800
- Gross cost avoided: EUR 3,200 × 0.5% (SEPA fee) = EUR 16
- Interest savings: EUR 3,200 × 5% annual / 365 × 5 days = EUR 2.19
- Total savings estimate: EUR 18.19  

**REQ-005-001-B: Execute netting transaction**

GIVEN: A netting opportunity identified (EUR 3,200 offset)  
WHEN: I click "Execute Netting"  
THEN: System creates:
- Contra-journal entry: debit AR, credit AP (both EUR 3,200)
- Status updates: both transactions marked "settled via netting"
- Reference links: each transaction references the other
- Audit log: "Netting executed by [user] at 2026-05-21 11:00 UTC, offset EUR 3,200"  

**REQ-005-001-C: Netting excludes certain suppliers**

GIVEN: A netting rule "exclude critical suppliers" (e.g., utilities, insurance)  
WHEN: I run netting analysis  
THEN: Excluded suppliers are not included in opportunities  
AND: System shows count of excluded opportunities  

---

## Bulk Payment Processing

### REQ-006-001: Create and batch multiple payments for approval and execution

**Summary:** Treasurers can group multiple payments, route through an approval workflow, and execute to multiple providers.

**Details:**
- Batch creation: select payments from pending list or create from scratch  
- Batch contents: summary shows payment count, total amount, grouped by provider/currency  
- Approval workflow: assign to approver role, track approval status, capture comments  
- Execution: initiate payments to each provider, track execution status per payment  
- Rollback: if execution fails for some payments, mark batch as "partial" and allow retry  

**Test Scenarios:**

**REQ-006-001-A: Create batch from pending payments**

GIVEN: 25 supplier invoices due this week with payment status "pending"  
WHEN: I click "Create Batch"  
THEN: Dialog shows all 25 pending payments  
AND: I can select/deselect individual payments  
AND: Summary shows "25 payments, EUR 7,350 total"  
AND: Grouping shows "SEPA Core: 23 payments, EUR 6,200 | Wise: 2 payments, EUR 1,150"  

**REQ-006-001-B: Submit batch for approval**

GIVEN: A batch with 25 payments created  
WHEN: I click "Submit for Approval"  
THEN: System routes to CFO role  
AND: Batch status changes to "awaiting_approval"  
AND: CFO receives notification  
AND: Audit trail logs "Batch submitted by [treasurer] for approval at 2026-05-21 09:00 UTC"  

**REQ-006-001-C: Execute batch after approval**

GIVEN: A batch approved by CFO  
WHEN: I click "Execute"  
THEN: System initiates payments to each provider  
AND: Each payment status changes to "executing"  
AND: After provider confirmation, status updates to "executed"  
AND: If 1 payment fails, batch status is "partial_execution", allows retry  

---

## Separation of Duties & Compliance

### REQ-007-001: Detect and report separation-of-duties violations

**Summary:** System monitors transactions for violations of segregated authorization rules (requester ≠ approver ≠ payer).

**Details:**
- Rules (configurable per organization):
  - Requester cannot be approver  
  - Approver cannot be payer  
  - Requester cannot be payer  
- Detection: check on each payment settlement, flag violations immediately  
- Severity levels: critical (all 3 violated), high (2 violated), medium (1 violated)  
- Reporting: daily audit digest with all violations, exportable for external auditors  
- Exception handling: violations can be marked "reviewed and approved" by compliance officer with comment  

**Test Scenarios:**

**REQ-007-001-A: Detect critical violation (same person all 3 roles)**

GIVEN: A payment with requester = approver = payer = "user-001"  
WHEN: The payment is executed  
THEN: System immediately flags violation:
- Severity: "Critical"
- Rules violated: "requester_equals_approver, approver_equals_payer, requester_equals_payer"
- Audit log: "Critical SOD violation detected for payment [id]"
- Notification sent to compliance officer  

**REQ-007-001-B: Detect high-severity violation (2 roles same person)**

GIVEN: A payment with requester = approver (different from payer)  
WHEN: The payment is executed  
THEN: System flags:
- Severity: "High"
- Rules violated: "requester_equals_approver"  

**REQ-007-001-C: View and export violations report**

GIVEN: 5 violations recorded in past 30 days  
WHEN: I view the "Separation of Duties" audit report  
THEN: Table shows:
- Violation ID, User (violator), Rule Violated, Date, Amount, Severity, Status (open/reviewed)
- Filter by severity, date range, rule type
- Export to CSV for auditor review  

**REQ-007-001-D: Mark violation as reviewed**

GIVEN: A critical violation flagged  
WHEN: Compliance officer clicks "Mark as Reviewed"  
THEN: Dialog appears for comment input  
AND: After submitting, violation status changes to "reviewed_and_approved"  
AND: Audit trail records: "[compliance_officer] reviewed violation at 2026-05-21 14:00 UTC with comment: '[comment]'"  

---

## Treasury Administration & Configuration

### REQ-008-001: Configure payment providers per administration

**Summary:** Administrators can assign distinct payment service providers (bank, fintech, internal) to administrations for cost optimization and multi-currency support.

**Details:**
- Provider types: bank (SEPA Core, international wires), fintech (Wise, Stripe), internal (manual processing)  
- Per-administration assignment: each administration can have multiple providers, with preference order  
- Supported currencies: define which currencies each provider supports  
- API credentials: encrypt and store provider API keys securely (never log)  
- Connectivity test: verify API access before activation  
- Fallback: if primary provider is unavailable, route to backup provider  

**Test Scenarios:**

**REQ-008-001-A: Add payment provider to administration**

GIVEN: I am an admin for "Voorbeeld BV"  
WHEN: I go to Settings → Payment Providers → Add Provider  
THEN: Dialog shows fields:
- Provider name: dropdown (SEPA Core, Wise, Stripe, Internal, Custom)
- Supported currencies: multi-select [EUR, GBP, USD, ...]
- API endpoint (if fintech)
- API key: encrypted input with toggle to show/hide
- Test connectivity: button to verify access  
AND: I can save the provider  

**REQ-008-001-B: Test provider connectivity**

GIVEN: A new Wise provider with API key filled in  
WHEN: I click "Test Connectivity"  
THEN: System attempts to authenticate with Wise API  
AND: Shows result: "Connection successful ✓" or error with reason  
AND: I can proceed only if connection succeeds  

**REQ-008-001-C: Route payment to provider based on currency**

GIVEN: Two providers configured:
- SEPA Core: EUR only
- Wise: EUR, GBP, USD  
AND: A payment in GBP is initiated  
WHEN: Payment is approved  
THEN: System routes to Wise (only provider supporting GBP)  
AND: Payment detail shows "Provider: Wise"  

---

### REQ-008-002: Set and monitor liquid reserve requirements

**Summary:** CFOs configure minimum cash reserve thresholds and receive alerts when reserves fall below requirements.

**Details:**
- Threshold definition: minimum balance per account or per administration (EUR, GBP, etc.)  
- Alert triggers: real-time on settlement, daily digest, or weekly report  
- Alert actions: notify CFO/treasurer, suspend non-critical payments, suggest funding  
- Dashboard widget: live reserve status showing current vs. minimum, days of reserve  
- Reporting: historical trend (reserve level over past 90 days)  

**Test Scenarios:**

**REQ-008-002-A: Set reserve minimum and receive alert**

GIVEN: CFO sets minimum reserve for "Voorbeeld BV" to EUR 50,000  
AND: Current balance is EUR 52,000  
WHEN: Payments of EUR 3,500 and EUR 2,000 are executed (total EUR 5,500)  
THEN: Closing balance becomes EUR 46,500 (below minimum)  
AND: Real-time alert is sent to CFO: "Liquid reserves below minimum. Current: EUR 46,500, Minimum: EUR 50,000, Shortfall: EUR 3,500"  

**REQ-008-002-B: View reserve dashboard widget**

GIVEN: Reserve minimum is EUR 50,000, current balance EUR 46,500  
WHEN: I view the treasury dashboard  
THEN: Liquid Reserves widget shows:
- Current balance: EUR 46,500
- Minimum required: EUR 50,000
- Shortfall: EUR 3,500 (in red)
- Status: "Critical"
- Days of reserve: 10.5 (based on avg daily burn)
- Recommended actions: "Defer non-critical payments, increase credit line"  

**REQ-008-002-C: Export reserve trend report**

GIVEN: 90 days of reserve data  
WHEN: I click "Export Trend Report"  
THEN: CSV is downloaded with:
- Date, Opening Balance, Inflows, Outflows, Closing Balance, Min Required, Variance
- Line chart showing trend (opening + closing balance vs. min threshold)  

---

### REQ-008-003: Configure cash pooling rules

**Summary:** Administrators set up automatic daily/weekly transfers from satellite accounts to a primary account for liquidity optimization.

**Details:**
- Primary account: designated collection account (usually head office)  
- Pooled accounts: satellite accounts to drain (branch offices, project accounts)  
- Frequency: daily, weekly, monthly (configurable)  
- Transfer time: specify time of day (e.g., 5 PM UTC) to avoid intra-day settlement issues  
- Minimum balance rule: keep threshold balance in each satellite (e.g., EUR 10k for operational needs)  
- Cost savings: report estimated monthly fee reduction (e.g., EUR X saved via netting vs. multiple account fees)  

**Test Scenarios:**

**REQ-008-003-A: Configure cash pooling**

GIVEN: 3 bank accounts (1 primary, 2 satellites)  
WHEN: I go to Settings → Cash Pooling → Configure  
THEN: Form shows:
- Primary account: dropdown [ABN AMRO NL91...]
- Pooled accounts: multi-select [ING NL31..., Deutsche Bank DE89...]
- Transfer frequency: dropdown [daily, weekly, monthly]
- Transfer time: time picker (default 17:00 UTC)
- Minimum balance per account: currency input (EUR 10,000)  

**REQ-008-003-B: Execute cash pooling manually**

GIVEN: Cash pooling configured but not yet scheduled  
WHEN: I click "Execute Pool Now"  
THEN: System calculates optimal transfers:
- ING account balance: EUR 45,000, min required: EUR 10,000 → transfer EUR 35,000
- Deutsche account balance: EUR 32,000, min required: EUR 10,000 → transfer EUR 22,000
- Total to transfer: EUR 57,000
AND: Initiates transfers
AND: Shows confirmation: "2 transfers initiated, EUR 57,000 total"  

**REQ-008-003-C: Scheduled daily pooling**

GIVEN: Cash pooling configured with frequency "daily" at 17:00 UTC  
WHEN: Daily job runs at 17:00  
THEN: System:
- Fetches balance for each satellite account
- Calculates transfer amounts (balance - minimum)
- Initiates SEPA transfers
- Logs each transfer with timestamp, amounts, status
- Sends daily digest to treasurer with transfer summary  

---

## Bank Fee Analysis

### REQ-009-001: Analyze bank fees by transaction type and account

**Summary:** System breaks down and visualizes bank charges to help identify savings opportunities and support vendor negotiations.

**Details:**
- Fee sources: transaction fees (SEPA, wire, settlement), maintenance, currency conversion, services  
- Grouping: by account, transaction type, time period (month, quarter)  
- Trend analysis: compare month-over-month or year-over-year  
- Opportunity detection: flag accounts with above-average fees for potential renegotiation  
- Export: CSV with detail rows for audit/negotiation with bank  

**Test Scenarios:**

**REQ-009-001-A: View fee breakdown by transaction type**

GIVEN: 6 months of bank statements with various transaction types  
WHEN: I view the "Analyze Bank Fees" report  
THEN: Chart shows breakdown by transaction type:
- SEPA transfers: EUR 125 (45%)
- Currency conversion: EUR 89 (32%)
- Wire transfers: EUR 42 (15%)
- Account maintenance: EUR 15 (5%)
- Other: EUR 9 (3%)
AND: Total fees: EUR 280 over 6 months (EUR 46.67/month average)  

**REQ-009-001-B: Trend analysis and optimization recommendation**

GIVEN: Monthly fee trends for 6 months: [EUR 42, 45, 48, 51, 53, 58]  
WHEN: I view the report  
THEN: System shows:
- Trend line: upward (fees increasing by EUR 3.33/month on average)
- Annualized: EUR 640 (if trend continues)
- Recommendation: "Consider switching payment provider or negotiating maintenance fee"
- Potential savings: "If you reduce wire transfer frequency by 20%, estimated savings: EUR 8.40/month"  

**REQ-009-001-C: Export fees for bank negotiation**

GIVEN: Fee data with detailed transactions  
WHEN: I click "Export for Negotiation"  
THEN: CSV downloads with columns:
- Date, Transaction Type, Amount, Fee Charged, Fee % of Amount
- Sorted by fee amount (highest first)
- Summary row: Total Fees, Avg Fee Rate
- Used by finance team to negotiate better rates with bank  

---

## Payment Arrangements Portfolio

### REQ-010-001: View portfolio of all active payment arrangements

**Summary:** Treasurer sees unified view of standing orders, payment plans, and settlement agreements.

**Details:**
- Data sources: ScheduledPayment, PaymentPlan (invoices on installment), settlement agreements  
- Columns: arrangement type, amount, frequency, next due date, counterparty, status  
- Filtering: by status (active, paused, completed), due date range, counterparty type  
- Sorting: by next due date (overdue first)  
- Alerts: highlight overdue arrangements, approaching maturity, high-value items  
- Actions: skip/defer payment, modify frequency, pause/resume, cancel  

**Test Scenarios:**

**REQ-010-001-A: View all active payment arrangements**

GIVEN: Multiple active arrangements:
- Standing order: EUR 2,000/month to utilities (Essent)
- Settlement agreement: EUR 5,000 quarterly to contractor (Zorgdiensten)
- Invoice installment: EUR 1,200 × 3 months (Office supplies)  
WHEN: I view the portfolio  
THEN: Table shows:
- Standing Order, EUR 2,000, Monthly, Next due 2026-05-28, Essent, Active
- Settlement, EUR 5,000, Quarterly, Next due 2026-06-15, Zorgdiensten, Active
- Installment, EUR 1,200, Monthly, Next due 2026-05-25, Office supplies, Active  

**REQ-010-001-B: Highlight overdue arrangements**

GIVEN: A standing order due 2026-05-15 (today is 2026-05-21)  
WHEN: I view the portfolio  
THEN: The overdue arrangement is highlighted in red  
AND: Row shows "Overdue: 6 days"  
AND: Status badge shows "Action Required"  

**REQ-010-001-C: Defer payment**

GIVEN: A standing order due today (2026-05-21)  
WHEN: I click "Defer" on the arrangement  
THEN: Dialog asks for new due date  
AND: I select 2026-05-28  
AND: Next due date updates, arrangement status remains active  
AND: Audit log records deferral: "[user] deferred payment by 7 days at 2026-05-21 10:00 UTC"  

---

## Testing Requirements

### Manual Browser Tests (Feature Verification)

All 12 features require manual browser testing to verify:
- UI rendering (form fields, tables, charts)  
- Data flow (input → processing → output)  
- Error handling (invalid input, edge cases)  
- Accessibility (keyboard navigation, screen reader compatibility per WCAG AA)  
- Performance (UI responsive <2s load time, <5s for initial data fetch)  

**Test Environment:**
- Browser: Chrome/Firefox (latest 2 versions)  
- Nextcloud instance: v29+ with OpenRegister enabled  
- Test data: seed data from design.md  
- Network: simulated 3G latency to test real-world performance  

### API Integration Tests (Automated)

Newman/Postman collection covering:
- Bank statement import (valid/invalid files, duplicates)  
- Entry matching (exact, fuzzy, edge cases)  
- Cash flow forecast (various time horizons, adjustments)  
- Loan analysis (eligible/ineligible scenarios)  
- IBAN verification (valid/invalid, change detection)  
- Payment batching (create, approve, execute, rollback)  
- Netting analysis (mutual AR/AP, partial offsets)  
- Separation of duties (all violation types, exception handling)  
- Reserve monitoring (threshold breaches, alerts)  

Each test includes:
- Request (method, endpoint, params, auth)  
- Expected response (status 200/400/403, body schema)  
- Assertions (status, content-type, field values)  

---

**Specifications Version:** 1.0  
**Last Updated:** 2026-05-21
