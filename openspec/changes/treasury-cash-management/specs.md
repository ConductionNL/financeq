# Treasury & Cash Management — Specifications

**Change:** treasury-cash-management  
**Phase:** specifications  
**Created:** 2026-05-21

---

## Requirements Overview

This spec defines 28 detailed requirements across 8 key feature areas: cash account management, multi-currency tracking, FX exposure, liquidity forecasting, payment batching, RFQ management, scheduled payments, and treasury task tracking.

Each requirement uses GIVEN/WHEN/THEN acceptance criteria for clear verification during testing.

---

## REQ-001: Create and Manage Cash Accounts

**Description:**  
Users can create, update, and delete bank accounts, petty cash holdings, and cash equivalents. Each account tracks current and available balances, risk level, and currency.

**Acceptance Criteria:**

```
GIVEN a user with Treasury role
WHEN they click "New Cash Account" in the dashboard
THEN a form appears with fields: Account Name, Type (BankAccount/PettyCash/CashEquivalent), 
     Bank Name, Account Number, GL Code, Currency, Current Balance, Risk Level (Low/Medium/High)

GIVEN the form is filled with valid data
WHEN the user clicks "Save"
THEN the account is created in OpenRegister, appears in the account list with 
     immediate balance visibility

GIVEN an existing cash account
WHEN the user clicks "Edit"
THEN they can modify: Account Name, Risk Level, is-active flag, is-primary flag
AND the change is logged in the audit trail

GIVEN a cash account with no recent transactions
WHEN the user clicks "Delete"
THEN a confirmation dialog warns that balance history will be lost
AND if confirmed, the account is soft-deleted (marked inactive, retained for audit)
```

**Story Links:** Story 3, Story 6, Story 7, Story 8, Story 14

---

## REQ-002: Import Bank Statements via CAMT.053

**Description:**  
Treasurers import bank statements in CAMT.053 format (ISO 20022) from multiple banks. The system parses transactions, matches them to pending payments, and updates cash positions.

**Acceptance Criteria:**

```
GIVEN a user on the Cash Accounts page
WHEN they click "Import Statement" and select a .xml file
THEN the system reads the CAMT.053 envelope and displays:
     - Source bank and account
     - Statement period (start-end date)
     - Transaction count
     - Opening and closing balances

GIVEN the preview is accepted
WHEN the user clicks "Import"
THEN each transaction is parsed:
     - Debit/credit direction
     - Amount, currency, date
     - Counterparty (creditor/debtor name)
     - Transaction reference and details
AND the cash account's current balance is updated to the statement closing balance

GIVEN imported transactions exist
WHEN the payment status engine runs
THEN it automatically marks scheduled payments as "executed" if their 
     reference matches a CAMT.054 confirmation

GIVEN a duplicate transaction (same amount, date, account)
WHEN import detects it
THEN it logs a warning and skips the transaction
AND the user is notified in an import summary report
```

**Story Links:** Story 6

**Technical Notes:**
- Use `ImportService` from OpenRegister for file upload
- Map CAMT.053 fields to OpenRegister Payment entity
- Support .xml (CAMT.053) and CSV fallback format

---

## REQ-003: View Consolidated Cash Position

**Description:**  
CFOs and treasurers see a real-time dashboard showing the sum of all cash accounts in the base currency (EUR), grouped by account type and risk level.

**Acceptance Criteria:**

```
GIVEN a user opens the Treasury Dashboard
WHEN the page loads
THEN the system displays a primary KPI card showing:
     - Total consolidated balance in EUR
     - Last update timestamp
     - Change since yesterday (amount and %)

GIVEN multiple bank accounts with different currencies
WHEN the dashboard renders
THEN it shows:
     - Subtotals per account (BankAccount, PettyCash, CashEquivalent)
     - Subtotal per currency with EUR conversion rate and date
     - Grand total in EUR

GIVEN the user clicks on a currency balance
WHEN the detail view opens
THEN they see the full FX conversion trail:
     - Amount in original currency
     - Exchange rate used
     - Rate source (ECB, bank, internal)
     - Valuation timestamp

GIVEN the dashboard is open for more than 5 minutes
WHEN new transactions are imported
THEN the balance KPI updates automatically (no page refresh needed)
AND a notification badge shows "Updated 3 min ago"
```

**Story Links:** Story 3, Story 7, Story 8, Story 14

**Technical Notes:**
- Build with `CnStatsBlock` for KPI card
- Use `CnDataTable` for account list
- Leverage ObjectStore subscriptions for real-time updates

---

## REQ-004: Track Multi-Currency Balances

**Description:**  
System maintains separate balance records per currency per account, enabling FX exposure tracking and multi-currency forecasting.

**Acceptance Criteria:**

```
GIVEN a cash account holds USD 200,000 and GBP 85,000
WHEN the currency balance view loads
THEN the system shows:
     - BankAccount: [EUR 450,000], [USD 200,000 @ 1.0850 = EUR 217,000], 
       [GBP 85,000 @ 0.8620 = EUR 73,270]
     - Total consolidated balance: EUR 740,270

GIVEN exchange rates fluctuate daily
WHEN the rate update job runs at market close (16:00 CET)
THEN the system:
     - Fetches latest rates from ECB API
     - Recalculates all FX positions
     - Records new CurrencyBalance records with timestamp
     - Stores previous balance for variance analysis

GIVEN the user views the monthly currency trend
WHEN they select "View Trend" on a currency
THEN they see a line chart:
     - X-axis: dates (daily)
     - Y-axis: balance in original currency
     - Overlay: EUR valuation line
     - Variance annotation: "↑ 5% this month"
```

**Story Links:** Story 4, Story 9, Story 62

---

## REQ-005: Track Foreign Exchange Exposure and Risk

**Description:**  
System monitors FX exposure across all currencies, calculates unrealized gains/losses, and flags high-risk positions.

**Acceptance Criteria:**

```
GIVEN a cash account holds USD 200,000 while EUR base currency
WHEN the FX exposure report loads
THEN it displays:
     - Foreign Currency: USD
     - Exposure Amount: 200,000 USD
     - Current Rate: 1.0850 EUR/USD
     - Base Currency Value: 217,000 EUR
     - Unrealized Gain/Loss: +2,500 EUR (spot bought at 1.0800)
     - Risk Level: Medium (based on volatility)

GIVEN the USD/EUR exposure exceeds 15% of total cash
WHEN the risk assessment runs
THEN the system:
     - Calculates concentration risk as (USD balance / total EUR equivalent) × 100
     - Flags exposure as "High" if > 15%
     - Recommends hedging via forward contracts (future feature)

GIVEN multiple FX exposures
WHEN the user runs a VaR (Value at Risk) report
THEN the system shows:
     - 95% confidence interval: worst-case daily loss
     - Historical volatility per currency pair
     - Diversification benefit (correlation matrix)

GIVEN an FX rate snapshot exists
WHEN the treasurer runs a "what-if" scenario (e.g., EUR weakens to 1.15)
THEN the system recalculates all FX positions with the new rate
AND displays the impact on total cash position
```

**Story Links:** Story 4, Story 62

---

## REQ-006: Generate Liquidity Forecast (AI-Powered)

**Description:**  
System uses historical transaction data and scheduled payments to project cash flow for 13 weeks ahead. AI model provides High/Medium/Low confidence levels.

**Acceptance Criteria:**

```
GIVEN a cash account with 12+ months of transaction history
WHEN the treasurer clicks "Generate 13-Week Forecast"
THEN the system:
     1. Retrieves historical daily inflows/outflows
     2. Filters scheduled payments for the 13-week window
     3. Passes data to Anthropic API with prompt caching
     4. Generates projections per day with confidence bands

GIVEN the forecast is generated
WHEN the user views the "Liquidity Forecast" page
THEN they see:
     - Line chart: projected daily balance (green line)
     - 80% confidence band (light green shading)
     - Scheduled payment markers (orange dots)
     - Expected inflow/outflow (blue/red area chart)
     - Lowest projected balance and when it occurs

GIVEN the forecast includes seasonal patterns
WHEN the model detects Q2 historical surpluses
THEN the forecast confidence is "High" for Q2 2027 projection
AND if recent patterns changed, confidence drops to "Medium"

GIVEN the forecast is 5+ days old
WHEN new transactions or scheduled payments are added
THEN the system shows a "Refresh" button
AND the user can regenerate with latest data

GIVEN the daily projected balance falls below a threshold
WHEN the threshold is set by the treasurer (e.g., EUR 100k minimum)
THEN the system highlights those days in red
AND recommends a short-term loan or accelerated AR collection
```

**Story Links:** Story 1, Story 9, Story 52, Story 73

**Technical Notes:**
- Use `ChatService` from OpenRegister for AI calls
- Implement prompt caching to reduce token usage
- Store forecasts in `LiquidityForecast` entity
- Calculate confidence via model's uncertainty bands

---

## REQ-007: Schedule Payments for Future Execution

**Description:**  
Treasurers create one-off or recurring payments scheduled for specific dates. The system validates dates, checks liquidity, and batches for execution.

**Acceptance Criteria:**

```
GIVEN a payee and due date for a payment
WHEN the treasurer clicks "Schedule Payment" in an invoice detail
THEN a dialog opens with:
     - Payee: [auto-filled from invoice]
     - Amount: [auto-filled from invoice]
     - Currency: [auto-filled]
     - Scheduled Date: [editable, default = due date]
     - Payment Method: Bank Transfer / Check / Other
     - Frequency: Once / Daily / Weekly / Monthly / Quarterly / Annual
     - Recurring Until: [end date for recurring payments]

GIVEN a monthly rent payment (EUR 5,000)
WHEN the user sets Frequency = "Monthly" and Recurring Until = "2027-05-31"
THEN the system creates 12 ScheduledPayment records (one per month)
AND shows a preview: "12 payments scheduled, total EUR 60,000"

GIVEN the scheduled date is in the past
WHEN the user tries to save
THEN the system shows an error: "Scheduled date must be in the future"

GIVEN scheduled payments exist
WHEN the daily payment execution job runs at 06:00 CET
THEN for each "scheduled date = today" payment:
     1. Check if cash account has sufficient balance (available balance ≥ payment amount)
     2. If YES: move payment to "approved" → "processing" → "executed"
     3. If NO: pause payment and notify treasurer "Insufficient liquidity"
     4. Record execution date and confirmation reference

GIVEN a recurring payment's next execution date
WHEN it is executed successfully
THEN the system:
     - Records the execution date
     - Calculates the next occurrence
     - Updates nextExecutionDate for the next cycle
```

**Story Links:** Story 2, Story 5, Story 38, Story 40

---

## REQ-008: Create and Approve Payment Batches

**Description:**  
Treasurers group multiple payments into batches for mass approval, export, and scheduled execution. Approval chain enforces dual sign-off for large amounts.

**Acceptance Criteria:**

```
GIVEN the treasurer wants to pay 15 supplier invoices
WHEN they select all invoices and click "Batch Payment"
THEN the system:
     - Displays a summary: "15 payments, total EUR 87,500"
     - Creates a PaymentBatch entity with status = "draft"
     - Calculates total amount and transaction count
     - Generates batch number (BATCH-YYYY-MM-NNN)

GIVEN a batch is in "draft" status
WHEN the treasurer clicks "Submit for Approval"
THEN the batch moves to "pending" status
AND an approval request is sent to the Financial Controller
AND the batch is locked (no more edits until decision)

GIVEN a batch total exceeds EUR 50,000
WHEN approval is requested
THEN the system enforces dual approval:
     - First approval: Financial Controller
     - Second approval: CFO or Treasurer
AND an email is sent to each approver with the batch summary

GIVEN a batch is "approved"
WHEN the treasurer clicks "Schedule Batch"
THEN they pick an execution date (next 1-30 days)
AND the batch moves to "processing" → "completed" after execution

GIVEN a batch contains payments to multiple banks
WHEN the execution job runs
THEN the system:
     - Groups payments by bank
     - Groups by currency
     - Generates separate SEPA files per bank/currency combo
     - Logs file generation timestamp and reference

GIVEN a batch is partially failed (e.g., 2 of 15 payments failed due to insufficient balance)
WHEN the execution completes
THEN the batch status = "failed"
AND a report shows:
     - Successful: 13 payments, EUR 85,500
     - Failed: 2 payments, EUR 2,000 (with reason)
     - Recommended action: retry after next cash inflow
```

**Story Links:** Story 1, Story 5, Story 18, Story 19

**Technical Notes:**
- Implement approval chain using `ApprovalService` (if available) or custom workflow
- Use `PaymentBatch` entity to group payments
- Enforce amount thresholds and dual sign-off

---

## REQ-009: Export Payments as SEPA pain.001 Files

**Description:**  
System generates ISO 20022 SEPA Credit Transfer (pain.001.001.09) XML files ready for bank upload. Validates IBAN checksums, BIC routing, and compliance rules.

**Acceptance Criteria:**

```
GIVEN a payment batch is approved and scheduled for 2026-06-10
WHEN the execution date arrives
THEN the system:
     1. Validates all payment records:
        - IBAN checksum (mod-97 algorithm)
        - Payee name present and non-empty
        - Amount > 0 and <= 999,999,999.99
     2. Generates pain.001 XML envelope:
        - MessageID (unique per bank/day)
        - CreationDateTime
        - InitiatingParty (organization details)
        - PaymentInformation block per currency
        - CreditTransferTransactionInformation per payment
     3. Signs file (if PAdES required by bank)
     4. Saves to Files app with timestamp

GIVEN the pain.001 file is generated
WHEN the user clicks "Download"
THEN they receive: SEPA_[BatchNumber]_[Date].xml
AND a checksum and certificate info are displayed

GIVEN the system detects an IBAN syntax error
WHEN generating pain.001
THEN it:
     - Stops file generation
     - Reports which payments have invalid IBANs
     - Allows user to correct and retry

GIVEN a large batch (>100 payments)
WHEN pain.001 file is generated
THEN the system splits it into multiple files (max 100 per file per SEPA rules)
AND generates an index file listing all output files

GIVEN multiple currencies in a batch (EUR + GBP)
WHEN pain.001 is generated
THEN the system creates separate payment information blocks per currency
AND notes in the batch report: "3 EUR payments, 2 GBP payments → 2 SEPA files"
```

**Story Links:** Story 5, Story 18, Story 19, Story 50, Story 249

**Technical Notes:**
- Use XML schema per ISO 20022 pain.001.001.09
- Implement IBAN validator using mod-97 algorithm
- Support both unsigned and signed (PAdES) output
- Leverage `FileService` for upload/download

---

## REQ-010: Support Multiple Payment Methods

**Description:**  
System tracks payment method per scheduled payment (bank transfer, check, ACH, card, internal transfer). Different methods have different execution timelines and compliance rules.

**Acceptance Criteria:**

```
GIVEN a scheduled payment for a small vendor
WHEN the treasurer selects Payment Method = "Check"
THEN the system:
     - Allows check number entry (optional, auto-generate if blank)
     - Sets execution date to T+3 (accounting for mail delay)
     - Moves funds to "pending check issuance"

GIVEN a scheduled payment to a municipal entity
WHEN the treasurer selects Payment Method = "Internal Transfer"
THEN the system:
     - Bypasses bank routing (no IBAN validation needed)
     - Updates both entities' positions immediately
     - Logs as zero-cost intercompany settlement

GIVEN multiple payment methods exist
WHEN the daily execution job runs
THEN it:
     1. Filters payments by method
     2. Executes bank transfers first (SEPA batching)
     3. Queues checks for printing
     4. Processes internal transfers synchronously

GIVEN a check is printed
WHEN the treasurer scans the check image
THEN the system:
     - Records check image attachment
     - Marks as "mailed" (triggers T+3 clearing assumption)
     - Prompts for mailing confirmation when scan completes
```

**Story Links:** Story 2, Story 5, Story 40

---

## REQ-011: Manage Request for Quotation with Digital Lockbox

**Description:**  
Procurement officers create RFQs with digital lockbox to prevent bids from being viewed before the deadline. System enforces lockbox constraints and manages multi-round negotiations.

**Acceptance Criteria:**

```
GIVEN a procurement officer creating an RFQ for a software contract
WHEN they fill in: Title, Description, Estimated Value, Deadline, and toggle "Enable Digital Lockbox"
THEN the system:
     - Shows "Lockbox will open at [deadline + 1 hour]"
     - Creates RequestForQuotation entity with lockboxEnabled = true, lockboxOpensAt timestamp
     - Generates RFQ number (RFQ-YYYY-MM-NNN)

GIVEN the RFQ is published with lockbox enabled
WHEN suppliers visit the RFQ to submit bids
THEN they see:
     - RFQ title, description, requirements
     - Deadline (countdown timer)
     - A "Submit Bid" button
     - NO visibility of other bids (lockbox prevents viewing)

GIVEN a supplier submits a bid before the deadline
WHEN the submission is recorded
THEN:
     - The bid is stored but NOT displayed to other suppliers or the RFQ owner
     - Submission timestamp and bidder name are logged privately
     - Bidder receives confirmation of receipt

GIVEN it is exactly the deadline + 1 hour
WHEN the system runs the "Open Lockbox" job
THEN:
     - All bids become visible to authorized buyers
     - An email is sent to RFQ owner: "[N] bids received, now open for evaluation"
     - Bids are displayed in a comparison table (amount, key terms, dates)

GIVEN an RFQ is in "published" status
WHEN the procurement officer clicks "Multi-Round Negotiation"
THEN they can:
     - Create Round 2 (round = 2)
     - Set a new deadline
     - Invite suppliers to refine bids
AND the system tracks round history

GIVEN the procurement officer selects a winning bid
WHEN they click "Award"
THEN the RFQ moves to "awarded" status
AND an award notice email is sent to all bidders (winner + losers)
```

**Story Links:** Story 21

---

## REQ-012: Track and Report Cash Flow Compliance (Wet Fido)

**Description:**  
For municipal treasurers: system monitors Wet Fido compliance by tracking borrowing limits, interest rate risk (renterisiconorm), and kasgeldlimiet rules for temporary cash placements.

**Acceptance Criteria:**

```
GIVEN a municipality with Wet Fido limits configured:
     - Maximum kasgeldlimiet (temporary cash placement at private banks): EUR 50M
     - Maximum renterisiconorm (interest rate exposure): EUR 200M
     - Borrowing limit (based on annual revenue): EUR 800M

WHEN the treasurer opens the "Compliance Dashboard"
THEN they see:
     - Wet Fido Status Card showing:
       * Current kasgeldlimiet usage: EUR 42M of EUR 50M (84%) [GREEN]
       * Current renterisiconorm: EUR 180M of EUR 200M (90%) [YELLOW]
       * Current borrowing: EUR 650M of EUR 800M (81%) [GREEN]
     - Trend charts (30-day view) for each metric

GIVEN the municipality places EUR 8M in a deposito (short-term deposit) with a private bank
WHEN the treasurer registers it via "New Investment - Deposito"
THEN the system:
     - Reduces available kasgeldlimiet: EUR 50M - EUR 8M = EUR 42M
     - Records the investment: amount, bank, interest rate, maturity date
     - Recalculates compliance position immediately

GIVEN the kasgeldlimiet usage exceeds 90%
WHEN the compliance monitor runs
THEN it:
     - Flags the position as "WARNING"
     - Sends an alert email: "Kasgeldlimiet at 95%, recommend deposit transfer to Rijkshoofdboekhouding"
     - Provides recommendation: "Deposit EUR 5M with Rijkshoofdboekhouding to comply"

GIVEN the renterisiconorm limit is breached
WHEN a new loan is requested
THEN the system shows: "Interest rate exposure at limit. Current: EUR 205M, Limit: EUR 200M.
     Approve new loan would require reducing existing positions."

GIVEN monthly reporting is due
WHEN the treasurer clicks "Generate Wet Fido Report"
THEN the system produces a PDF showing:
     - Compliance per day (30-day lookback)
     - Any breaches and remediation taken
     - Borrowing schedule and interest exposure
     - Signature blocks for treasurer and financial controller
```

**Story Links:** Story 12, Story 177, Story 182

**Technical Notes:**
- Store Wet Fido thresholds in `IAppConfig` (admin settings)
- Create compliance check job running daily at 06:00
- Generate Wet Fido report via PDF library

---

## REQ-013: Monitor schatkistbankieren Compliance

**Description:**  
For Dutch municipalities: track mandatory deposits with Rijkshoofdboekhouding (national treasury) and enforce limits on private bank holdings.

**Acceptance Criteria:**

```
GIVEN a Dutch municipality subject to schatkistbankieren rules
WHEN the treasurer opens the "schatkistbankieren Compliance" page
THEN they see:
     - Balance at Rijkshoofdboekhouding: EUR 150M (safe, no limits)
     - Balance at private banks: EUR 45M
     - Permitted maximum at private banks: EUR 50M
     - Utilization: 45M / 50M = 90% [YELLOW]

GIVEN the private bank balance exceeds the allowed maximum
WHEN the system detects it (daily check at 07:00 CET)
THEN it:
     - Sets status to "OUT OF COMPLIANCE"
     - Sends urgent alert: "Private bank balance EUR 52M exceeds EUR 50M limit.
       Immediate action required: transfer excess to Rijkshoofdboekhouding"
     - Recommends: "Transfer EUR 3M to Rijkshoofdboekhouding (reference: SGB-[date])"

GIVEN the treasurer initiates a transfer to Rijkshoofdboekhouding
WHEN they record it in the system
THEN the system:
     - Creates an intercompany transfer record
     - Reduces private bank balance by transfer amount
     - Logs it for quarterly reporting to Ministry of Interior

GIVEN the system needs to generate a quarterly compliance report
WHEN the treasurer clicks "Generate schatkistbankieren Report"
THEN the system produces:
     - Daily compliance status for 13 weeks (4 months lookback)
     - Any days out of compliance with remediation taken
     - List of transfers to Rijkshoofdboekhouding (amount, date, reference)
     - Signature blocks for treasurer and mayor (burgemeester)

GIVEN a day with a schatkistbankieren violation
WHEN the treasurer views that day's detail
THEN they see:
     - Opening balance at private bank
     - Inflows/outflows during the day
     - Closing balance (and by how much it exceeded limit)
     - Recommended remediation amount
```

**Story Links:** Story 12, Story 177, Story 182

---

## REQ-014: Register and Track Short-Term Cash Investments (Deposito)

**Description:**  
Treasurers register short-term cash placements (deposito) with banks, tracking maturity, interest rate, and Wet Fido impact.

**Acceptance Criteria:**

```
GIVEN the treasurer has EUR 10M in surplus cash
WHEN they click "New Investment - Deposito"
THEN a form appears with:
     - Bank: [dropdown of authorized counterparties]
     - Amount: [input]
     - Currency: [EUR]
     - Start Date: [today]
     - Maturity Date: [future date, e.g., 2026-08-21]
     - Agreed Interest Rate: [e.g., 3.75% p.a.]
     - Risk Level: [Low/Medium/High]

GIVEN the form is filled with valid data
WHEN the user clicks "Save"
THEN the system:
     1. Creates a ScheduledPayment record (type: Investment)
     2. Reduces available cash balance by the deposit amount (EUR 10M)
     3. Updates Wet Fido kasgeldlimiet position
     4. Recalculates compliance dashboard

GIVEN the deposito maturity date is within the 13-week forecast window
WHEN the liquidity forecast is generated
THEN the system includes:
     - Principal repayment (EUR 10M) as an inflow on maturity date
     - Interest accrual (calculated as: principal × rate × days/365)
     - Example: EUR 10M @ 3.75% for 91 days = EUR 94,315 interest inflow

GIVEN a deposito is registered
WHEN the monthly treasury report is generated
THEN it shows:
     - Deposito details: bank, amount, rate, maturity
     - Interest earned this month
     - Interest earned year-to-date
     - Opportunity cost vs. yield (comparison to ECB rates)

GIVEN the deposito maturity date arrives
WHEN the treasurer marks it as "Matured"
THEN the system:
     - Restores principal + interest to cash balance
     - Moves funds back to active cash account
     - Records interest income entry (GL account 4010 - Interest Income)
     - Closes the investment record
```

**Story Links:** Story 13, Story 189

---

## REQ-015: View Daily Cash Position at Day Start

**Description:**  
Municipal treasurers see a consolidated view of cash at the start of each banking day, with opening balance per bank and consolidated total.

**Acceptance Criteria:**

```
GIVEN it is the start of a banking day (08:00 CET)
WHEN the treasurer opens the "Daily Cash Position" dashboard
THEN they see a snapshot showing:
     - Timestamp: "As of 2026-05-21, 08:00 CET"
     - Opening Balance (per bank account):
       * ING NL91 ABNA...: EUR 450,000
       * Rabobank NL12 RABO...: EUR 200,000
       * Petty Cash: EUR 5,000
     - Consolidated Total: EUR 655,000
     - Change from yesterday: +EUR 15,000 (↑ 2.3%)

GIVEN bank statements have not yet arrived
WHEN the dashboard loads
THEN it shows:
     - Previous closing balance (from yesterday's import)
     - A note: "Waiting for bank statement import from [Bank Name]"
     - Last import timestamp: "Yesterday at 17:15 CET"

GIVEN a new bank statement arrives via automatic import
WHEN the CAMT.053 file is imported
THEN the dashboard:
     - Automatically updates the cash position
     - Shows "Updated 2 min ago"
     - Flags transactions > threshold (e.g., EUR 50k) as "Material Movement"

GIVEN the treasurer clicks on a specific bank account
WHEN the detail view opens
THEN they see:
     - Opening balance
     - All transactions from previous banking day:
       * Amount, date, time, counterparty, reference
       * Color-coded: green (inflow), red (outflow)
     - Closing balance = opening + inflows - outflows

GIVEN it is mid-day and new transactions have posted
WHEN the treasurer refreshes the dashboard
THEN the balance updates in real-time
AND they can see the live intraday position
```

**Story Links:** Story 14, Story 204

---

## REQ-016: Generate and Export Monthly Treasury Report

**Description:**  
Treasurers generate PDF reports for the treasury committee showing cash position, investments, compliance, and forecast vs. actuals.

**Acceptance Criteria:**

```
GIVEN a treasurer requests a monthly treasury report
WHEN they click "Generate Report" and select month (e.g., May 2026)
THEN the system collects:
     1. Average daily cash balance (simple average of month-end closing balances)
     2. Investment summary: depositos, interest earned
     3. Financing activity: new loans, repayments
     4. Wet Fido/schatkistbankieren compliance status
     5. Liquidity forecast variance (forecast vs. actual)
     6. Bank fees summary
     7. FX gains/losses for the month

GIVEN the data is collected
WHEN the system generates the report PDF
THEN it includes sections:
     - Executive Summary (1 page)
     - Cash Position Trend (30-day chart)
     - Investment Portfolio (table: bank, amount, rate, interest earned)
     - Compliance Dashboard (Wet Fido, schatkistbankieren, credit ratings)
     - Liquidity Forecast vs. Actuals (chart + table)
     - Bank Reconciliation Summary
     - Signature block for Treasurer and CFO

GIVEN the treasurer has set a budget target for average balance (e.g., EUR 500k)
WHEN the report is generated
THEN it shows:
     - Actual average balance: EUR 485k
     - Variance: -EUR 15k (-3%)
     - Explanation: "Lower than budgeted due to [reason]"

GIVEN a forecast was made at start of month
WHEN the month ends
THEN the report compares:
     - Forecast closing balance (made 2026-05-01): EUR 620k
     - Actual closing balance (2026-05-31): EUR 625k
     - Variance: +EUR 5k (99% accuracy)
     - Confidence band: forecast was "High" confidence

GIVEN the report is complete
WHEN the treasurer clicks "Export"
THEN they download: TREASURY_REPORT_MAY_2026.pdf
AND a summary email is sent to the committee chair with key metrics
```

**Story Links:** Story 15, Story 207, Story 212

---

## REQ-017: Verify Counterparty Credit Ratings

**Description:**  
System enforces minimum credit ratings for banks and investment counterparties per statute rules. Flags non-compliant counterparties and recommends actions.

**Acceptance Criteria:**

```
GIVEN the municipality has defined statutory credit rating requirements:
     - Banks (deposits, short-term borrowing): Min. A- (S&P) or A3 (Moody's)
     - Investment counterparties: Min. BBB (S&P) or Baa2 (Moody's)

WHEN the treasurer registers a new bank for deposito placement
THEN the system:
     1. Looks up the bank's current credit rating (from third-party source)
     2. Compares to minimum requirement
     3. If compliant: allows registration, logs rating and date
     4. If non-compliant: blocks registration and shows error:
        "Bank XYZ has rating BB (below minimum A-). Not eligible for investment."

GIVEN a counterparty's credit rating is downgraded
WHEN the treasurer opens the "Counterparty Compliance Check"
THEN the system shows:
     - Bank: ING Nederland
     - Current Rating: A+ (compliant)
     - Last Update: 2026-05-10
     - Exposure: EUR 50M (deposito, matures 2026-08-21)
     - Status: [GREEN] Compliant

GIVEN a bank's rating falls below minimum after registration
WHEN the compliance monitor runs
THEN it:
     - Sets status to "WARNING"
     - Sends alert: "ING rating downgraded to A-, no longer eligible per statute.
       Recommend moving EUR 50M to compliant counterparty by [maturity date]"
     - Displays countdown: "Time to action: 87 days"

GIVEN multiple counterparties exist
WHEN the treasurer runs "Credit Rating Audit"
THEN the system displays:
     - Counterparty, Current Rating, Minimum Required, Compliance Status
     - Non-compliant: [list with remediation deadline]
     - Summary: "14 of 15 counterparties compliant, 1 flagged for action"

GIVEN the rating update job runs daily
WHEN ratings are fetched from the data provider
THEN the system:
     - Stores new ratings with timestamp
     - Compares to registered minimum
     - Triggers alerts for any non-compliances
     - Logs all rating changes for audit trail
```

**Story Links:** Story 16, Story 216

---

## REQ-018: Monitor Liquid Reserve Requirements

**Description:**  
Municipal treasurers track liquid reserve levels (cash + available credit) against statutory minimums and monitor compliance daily.

**Acceptance Criteria:**

```
GIVEN a municipality with a statutory liquid reserve requirement:
     - Minimum: 25% of annual operating expenditure (e.g., EUR 200M if annual spend = EUR 800M)
     - Actual liquid assets: Cash + liquid deposits + available credit lines

WHEN the treasurer opens the "Liquidity Compliance" dashboard
THEN they see:
     - Required liquid reserves: EUR 200M (25% of budget)
     - Current liquid assets: EUR 245M (cash + accessible deposits + credit)
     - Compliance status: [GREEN] 122% of minimum
     - Trend: 30-day chart showing daily reserves

GIVEN the daily cash import runs
WHEN new balances are recorded
THEN the system:
     1. Sums all available cash per account
     2. Adds committed credit lines (e.g., overdraft facilities)
     3. Deducts restricted funds (encumbered for specific purposes)
     4. Calculates total liquid reserves
     5. Compares to statutory minimum
     6. Updates compliance dashboard

GIVEN a day where reserves fell below 25% minimum
WHEN the treasurer selects that day
THEN they see:
     - Opening liquid reserves: EUR 215M
     - Material outflows:
       * Payroll: EUR 45M
       * Debt service: EUR 32M
       * Capital projects: EUR 25M
     - Closing reserves: EUR 113M (insufficient by EUR 87M)
     - Days to recovery: [chart showing when inflows will restore compliance]

GIVEN reserves are trending toward non-compliance
WHEN the compliance monitor runs daily at 08:00
THEN if reserves fall below 25% + safety margin (20%), it:
     - Sends alert to Treasurer: "Liquid reserves projected to drop below minimum in 3 days"
     - Recommends: "Draw on EUR 50M credit line" or "Accelerate AR collections"

GIVEN the end of reporting period arrives
WHEN the treasurer exports the "Liquidity Compliance Report"
THEN it includes:
     - Daily compliance status for 3-month rolling window
     - Any breaches and remediation taken
     - Projections for next 3 months
     - Signed certification by Treasurer and CFO
```

**Story Links:** Story 17, Story 223

---

## REQ-019: Export Payroll as SEPA Payment File

**Description:**  
Payroll administrator can export approved payroll runs as SEPA payment files for bank upload without re-keying employee salary data.

**Acceptance Criteria:**

```
GIVEN payroll for May 2026 is entered and approved by Financial Controller
WHEN the payroll administrator clicks "Export SEPA"
THEN a dialog appears:
     - Payroll period: May 2026 (reads from payroll data)
     - Payment date: [editable, default = next business day]
     - Total amount: EUR 125,000 (48 employees)
     - Preview: "48 salary payments to employee bank accounts"

GIVEN the export parameters are confirmed
WHEN the user clicks "Generate SEPA File"
THEN the system:
     1. Retrieves all approved salary payments for the period
     2. Validates each employee's IBAN and name
     3. Generates a pain.001 XML file
     4. Saves with filename: PAYROLL_MAY_2026_[timestamp].xml
     5. Logs export: timestamp, filename, payment count, total amount

GIVEN the SEPA file is generated
WHEN the user opens it in their bank's portal
THEN they can upload it directly without re-keying
AND the bank parses it and executes payments per scheduled date

GIVEN export occurs
WHEN payroll status is updated
THEN the payroll status changes from "Approved" → "Exported"
AND a notification is sent to all approvers: "Payroll exported to bank"

GIVEN an employee has an invalid IBAN
WHEN the system tries to generate the SEPA file
THEN it:
     - Stops file generation
     - Reports: "Employee John Doe: IBAN NL91... invalid (checksum failed)"
     - Allows user to correct and retry
```

**Story Links:** Story 18, Story 232

---

## REQ-020: Export Final Payments to Municipal Treasury System

**Description:**  
Financial administrators export confirmed final payment orders in a format compatible with the municipal treasury system (e.g., SEPA or Coda format).

**Acceptance Criteria:**

```
GIVEN one or more confirmed final payment orders in the system
WHEN the administrator clicks "Export for Treasury"
THEN a dialog shows:
     - Selected payments: [list with amounts, dates, payees]
     - Total amount: [sum]
     - Format options: [SEPA XML / CSV / Coda format]
     - Destination: [Municipal Treasury System / Bank Portal / Save locally]

GIVEN SEPA format is selected
WHEN the user clicks "Export"
THEN a pain.001 XML file is generated (as per REQ-009)
AND saved with filename: TREASURY_EXPORT_[date]_[count].xml

GIVEN Coda format is selected (for municipal accounting systems using Coda standard)
WHEN the user clicks "Export"
THEN the system generates a fixed-width ASCII file per Coda specification:
     - Header: file type, date, organization, total amount
     - Detail rows: per payment with position codes and GL codes
     - Trailer: record count and hash total
AND saved with filename: TREASURY_[date].COD

GIVEN payments are exported
WHEN the file is successfully transmitted
THEN the system:
     - Records export date/time
     - Updates payment status to "Exported"
     - Creates audit trail entry
     - Sends confirmation email to Financial Controller

GIVEN a payment export fails
WHEN the system detects a transmission error
THEN it:
     - Retains the file for manual upload
     - Sends alert: "Export failed. File saved to [location] for manual upload."
     - Allows retry after user investigation
```

**Story Links:** Story 19, Story 239

---

## REQ-021: Generate and Validate SEPA pain.001 Files

**Description:**  
Comprehensive requirement for SEPA pain.001 file generation with full validation and compliance checks per ISO 20022.

**Acceptance Criteria:**

```
GIVEN a treasury manager with a batch of approved payments
WHEN they request "Export as pain.001"
THEN the system:
     1. Validates all payments meet SEPA requirements:
        - IBAN format and checksum (mod-97)
        - BIC (optional in modern SEPA, but checked if present)
        - Creditor name (non-empty, ≤70 chars)
        - Amount > 0 and < 1,000,000,000
        - Currency must be EUR (domestic SEPA) or multi-currency (international SEPA)
     2. Generates pain.001.001.09 XML envelope:
        - XML declaration and namespace
        - GroupHeader: MessageID, CreationDateTime, Initiating Party
        - PaymentInformation (one or more per currency/bank):
           * ControlSum (total amount in payment group)
           * RejectSequenceIndicator (if retrying after failure)
        - CreditTransferTransactionInformation per payment:
           * PaymentIdentification (unique reference per payment)
           * InstructedAmount (amount and currency)
           * Creditor info (name, IBAN, BIC)
           * RemittanceInformation (invoice reference, etc.)
     3. Digitally signs the XML (optional, depends on bank requirement)
     4. Returns file ready for download and bank upload

GIVEN the pain.001 file is generated
WHEN the system validates it against schema
THEN it checks:
     - Well-formed XML (parseable)
     - Schema compliance (pain.001.001.09 XSD)
     - All required fields present
     - No prohibited characters in free-text fields
     - Total amount checksum matches

GIVEN multiple currencies or multiple banks
WHEN pain.001 is generated
THEN the system:
     - Creates separate PaymentInformation blocks per currency+bank combination
     - Documents in the file: "Section 1: EUR to ING, Section 2: GBP to Rabobank"
     - Totals per section visible in the export report

GIVEN the file is ready
WHEN the user clicks "Download"
THEN they receive:
     - Filename: SEPA_PAIN001_[Organization]_[Date]_[Sequence].xml
     - Download size: [shown in MB]
     - Checksum (SHA-256) and expiration warning (if required by law)

GIVEN the file is downloaded
WHEN the user logs into their bank portal
THEN they can:
     1. Select "Import pain.001 file"
     2. Upload the .xml file
     3. Bank validates and displays summary: "[N] payments, total EUR [amount]"
     4. User confirms → bank submits to clearing system
```

**Story Links:** Story 5, Story 18, Story 19, Story 50, Story 249

---

## REQ-022: Unified Treasury Task Tracking (AP/AR/CapEx)

**Description:**  
Treasurers see a single task list combining accounts payable (invoices), accounts receivable (customer invoices), and capital expenditure items, all prioritized by due date and amount.

**Acceptance Criteria:**

```
GIVEN a treasurer opens the "Treasury Tasks" page
WHEN the page loads
THEN they see all tasks (AP, AR, CapEx, Loan, Investment) sorted by due date:
     - OVERDUE (red): [tasks with due date < today]
     - DUE THIS WEEK (orange): [due date between today and today+7]
     - DUE NEXT MONTH (yellow): [due date > today+7]
     - COMPLETED (green): [checked-off tasks]

GIVEN multiple task types exist
WHEN the task list displays
THEN each row shows:
     - Task type icon (AP/AR/CapEx) and color
     - Description: "Invoice INV-2026-001 - Acme Corp"
     - Amount and currency
     - Due date with countdown ("Due in 5 days")
     - Counterparty name
     - Status badge

GIVEN the treasurer clicks a task
WHEN the detail view opens
THEN they see:
     - Full task details (type, amount, counterparty, due date)
     - Related document (invoice, PO, contract) as attachment
     - Linked payment (if already scheduled)
     - Notes field for free-form comments
     - Action buttons: "Mark Complete", "Schedule Payment", "Delete"

GIVEN an AP task is marked "Complete"
WHEN the user saves
THEN the system:
     - Moves task to "Completed" section
     - Records completion date
     - Logs in audit trail

GIVEN the treasurer filters by task type
WHEN they select "Accounts Payable Only"
THEN the list shows only AP tasks
AND the count updates: "[45 AP tasks, 8 overdue, 12 due this week]"

GIVEN daily job runs at 08:00
WHEN due dates change (e.g., a task is now "due today" instead of "due tomorrow")
THEN the system:
     - Recalculates task status
     - Moves tasks between sections (e.g., from "Due Next Month" to "Due This Week")
     - Sends notification to treasurer if a task becomes "OVERDUE"
```

**Story Links:** Story 1, Story 4, Story 5, Story 30, Story 162

---

## REQ-023: Dashboard KPIs and Status Indicators

**Description:**  
Treasury dashboard displays key performance indicators (KPIs) with real-time updates, color-coded status, and drill-down capability.

**Acceptance Criteria:**

```
GIVEN a treasurer opens the Treasury Dashboard
WHEN the page fully loads
THEN the top section displays 4 KPI cards:
     - TOTAL CASH: EUR 655,000 [GREEN] (change: +EUR 15,000 / +2.3% from yesterday)
     - OPEN PAYMENTS: EUR 87,500 (15 payments pending approval) [YELLOW]
     - FORECAST CLOSING BALANCE (13 weeks): EUR 620,000 [GREEN] (high confidence)
     - COMPLIANCE STATUS: [GREEN] All limits met, no warnings

GIVEN market data has updated
WHEN the KPI is > 5 minutes old
THEN the card shows a refresh timestamp: "Updated 3 min ago"
AND a "Refresh" button allows manual update

GIVEN the treasurer clicks on "TOTAL CASH" KPI
WHEN the detail view opens
THEN they see:
     - Breakdown by account (ING: EUR 450k, Rabobank: EUR 200k, Petty Cash: EUR 5k)
     - Breakdown by currency (EUR 655k base)
     - Pie chart: account composition
     - 30-day trend: line chart showing daily balances

GIVEN the "COMPLIANCE STATUS" shows a warning
WHEN the card is clicked
THEN it drills into the compliance detail:
     - Which limits are violated
     - By how much
     - Recommended remediation
     - Timeline for action

GIVEN status changes dynamically
WHEN new transactions post
THEN the KPI updates in real-time
AND the color changes (e.g., from GREEN to YELLOW if a limit is approaching)

GIVEN historical KPI data exists
WHEN the user selects a date range (e.g., "Last 30 Days")
THEN the system displays:
     - KPI values on selected dates
     - Trend line showing progression
     - Min/max/average for the period
```

**Story Links:** Story 29, Story 57, Story 106

---

## REQ-024: Role-Based Access Control

**Description:**  
System enforces role-based permissions for treasury functions: view, create, approve, and execute payments based on user role and amount thresholds.

**Acceptance Criteria:**

```
GIVEN user roles defined:
     - Treasurer (all permissions)
     - Financial Controller (approve, execute)
     - Treasury Analyst (view, create payments)
     - CFO (approve large payments > EUR 100k)
     - Auditor (view-only)

WHEN a Treasury Analyst logs in
THEN they can:
     - View all cash accounts and balances
     - Create payment requests and schedules
     - View forecasts and reports
BUT cannot:
     - Approve payments
     - Execute bank exports
     - Delete accounts or payments

GIVEN a payment for EUR 50,000
WHEN a Financial Controller requests approval
THEN they can approve (no escalation needed)

GIVEN a payment for EUR 200,000
WHEN a Financial Controller requests approval
THEN the system:
     - Requires a second approver (CFO or Treasurer)
     - Sends approval request to CFO
     - Locks the payment until CFO approval received

GIVEN an auditor logs in
WHEN they try to create or modify a payment
THEN the system blocks the action: "Read-only access. Contact Treasurer to modify."

GIVEN a user's role is changed
WHEN the change is saved
THEN all open approval requests are reevaluated
AND if the user no longer has permission, they are removed from the approver list
```

**Story Links:** All stories (cross-cutting)

---

## REQ-025: Audit Trail and Change Tracking

**Description:**  
All treasury transactions (payments, forecasts, compliance decisions) are recorded in an audit trail with before/after snapshots, user, timestamp, and reason.

**Acceptance Criteria:**

```
GIVEN a payment is created, modified, or deleted
WHEN the change is saved
THEN the system records in AuditTrail:
     - Entity: Payment
     - Action: create / update / delete
     - User: [who made the change]
     - Timestamp: [ISO 8601]
     - Before snapshot: [previous state, if update]
     - After snapshot: [new state]
     - Change reason/comment (optional)

GIVEN the treasurer modifies a scheduled payment amount
WHEN the update is saved
THEN the audit log shows:
     - "Field 'amount' changed from 5000 to 5500"
     - User: john.doe@org.nl
     - Time: 2026-05-21 14:30:00 CET
     - Reason: "Invoice corrected by vendor"

GIVEN the auditor needs to verify payment history
WHEN they open the "Audit Trail" tab for a payment
THEN they see a timeline:
     - 2026-05-20 09:00: Created by Jane Smith
     - 2026-05-20 14:00: Approved by John Doe
     - 2026-05-21 08:15: Exported to bank
     - 2026-05-22 10:30: Marked executed

GIVEN sensitive changes exist (e.g., payee bank account change)
WHEN a user tries to change a payee IBAN
THEN the system:
     - Requires approval from a higher role (Treasurer)
     - Logs the attempt with old and new IBANs
     - Sends notification to Treasurer for confirmation

GIVEN a compliance violation is detected and remediated
WHEN the treasurer marks it as resolved
THEN the audit trail shows:
     - Violation: "kasgeldlimiet exceeded by EUR 2M on 2026-05-21"
     - Action taken: "Deposit transfer of EUR 3M to Rijkshoofdboekhouding"
     - Status: "Resolved"
     - Timestamp and user
```

**Story Links:** All stories (cross-cutting)

---

## Testing Strategy

- **Unit tests:** Validation logic (IBAN checksum, amount limits, date rules)
- **Integration tests:** Bank statement import, SEPA file generation, compliance checks
- **Browser tests (Playwright):** Dashboard KPIs, multi-step workflows (schedule → approve → export)
- **Persona tests:** Treasurer daily flow, controller approval, auditor verification
- **Compliance tests:** Wet Fido thresholds, schatkistbankieren limits per test data

---

## Acceptance & Sign-Off

All requirements must be verified against user acceptance criteria (GIVEN/WHEN/THEN) before moving to production.

**Verified by:** Treasurer, Financial Controller, CFO, Auditor (per persona testing)

**Date completed:** [Pending implementation]
