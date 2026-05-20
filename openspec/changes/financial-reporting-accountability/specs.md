# Specifications: Financial Reporting & Accountability — Shillinq

## ADDED Requirements

---

### REQ-FRA-001: Fiscal Year Creation and Management

The system must allow administrators to define, open, and close fiscal years. Once a fiscal year is closed, no new journal entries or GL postings may be made against it.

#### Scenario: Open a new fiscal year

- **GIVEN** no fiscal year exists for calendar year 2025
- **WHEN** an administrator creates a FiscalYear with year=2025, startDate=2025-01-01, endDate=2025-12-31
- **THEN** the fiscal year is created with isClosed=false
- **AND** journal entries may be posted to this period

#### Scenario: Close a fiscal year

- **GIVEN** a FiscalYear with isClosed=false and all year-end processing complete
- **WHEN** an administrator sets isClosed=true and closingDate to the current date
- **THEN** the fiscal year status is updated and recorded in the audit trail
- **AND** subsequent journal entry creation requests referencing this fiscal year return HTTP 409 with message "Boekjaar is afgesloten"

#### Scenario: Prevent posting to a closed year

- **GIVEN** a FiscalYear with isClosed=true
- **WHEN** any user attempts to create a JournalEntry or GeneralLedgerEntry linked to this fiscal year
- **THEN** the request is rejected with HTTP 409
- **AND** no data is persisted

---

### REQ-FRA-002: Double-Entry Journal Entry with Balance Validation

Every journal entry must be balanced (total debits = total credits) before it can be saved. Unbalanced entries are rejected at the API layer.

#### Scenario: Create a balanced journal entry

- **GIVEN** an open FiscalYear and a valid GeneralLedgerAccount
- **WHEN** a user creates a JournalEntry with debitAmount=5000.00 and creditAmount=5000.00
- **THEN** isBalanced is set to true and the entry is persisted
- **AND** the related GeneralLedgerAccount balance is updated

#### Scenario: Reject an unbalanced journal entry

- **GIVEN** an open FiscalYear
- **WHEN** a user submits a JournalEntry with debitAmount=5000.00 and creditAmount=4000.00
- **THEN** the request is rejected with HTTP 422 and message "Boeking is niet in evenwicht: debet en credit moeten gelijk zijn"
- **AND** no entry is persisted

#### Scenario: Reverse a posted journal entry

- **GIVEN** a JournalEntry with status=posted
- **WHEN** an authorized user initiates a reversal
- **THEN** a new JournalEntry is created with swapped debit/credit amounts and status=posted
- **AND** the original entry status is updated to reversed

---

### REQ-FRA-003: Chart of Accounts Management

The system must maintain a chart of accounts (GeneralLedgerAccount) covering the five standard account types: Asset, Liability, Equity, Revenue, and Expense.

#### Scenario: Add a new account to the chart of accounts

- **GIVEN** an authenticated bookkeeper
- **WHEN** they create a GeneralLedgerAccount with accountNumber="7100", accountName="Marketingkosten", accountType="Expense", currency="EUR"
- **THEN** the account is created and available for journal entry selection
- **AND** its currentBalance defaults to zero

#### Scenario: Prevent duplicate account numbers

- **GIVEN** a GeneralLedgerAccount with accountNumber="1000" already exists
- **WHEN** a user attempts to create another account with accountNumber="1000"
- **THEN** the request is rejected with HTTP 409 and message "Rekeningnummer is al in gebruik"

---

### REQ-FRA-004: Balance Sheet Generation

The system must generate a balance sheet showing total assets, total liabilities, and total equity at any specified date, derived from posted GeneralLedgerEntry records.

#### Scenario: Generate a balance sheet at year-end

- **GIVEN** a closed FiscalYear 2024 with posted GeneralLedgerEntries
- **WHEN** a controller requests a balance sheet for reportDate=2024-12-31
- **THEN** the system generates a BalanceSheet with totalAssets, totalLiabilities, and totalEquity computed from all posted entries up to and including 2024-12-31
- **AND** totalAssets = totalLiabilities + totalEquity (fundamental accounting equation)
- **AND** the balance sheet status is set to draft

#### Scenario: Publish a finalized balance sheet

- **GIVEN** a BalanceSheet with status=final that has been reviewed
- **WHEN** a financial director sets status=published
- **THEN** the balance sheet is marked published and no further edits are permitted
- **AND** the change is recorded in the audit trail

#### Scenario: Balance sheet at any point in time

- **GIVEN** an open FiscalYear with posted entries across multiple months
- **WHEN** a user requests a balance sheet for reportDate=2025-03-31
- **THEN** the system includes only GL entries with entryDate ≤ 2025-03-31
- **AND** returns the balance sheet with status=draft

---

### REQ-FRA-005: Trial Balance Generation and Verification

The system must generate a trial balance listing all GL account balances grouped by account type, with separate debit and credit columns, and a check that total debits equal total credits.

#### Scenario: Generate a balanced trial balance

- **GIVEN** a FiscalYear with posted GeneralLedgerEntries where total debits equal total credits
- **WHEN** a user generates a trial balance for a given reportDate
- **THEN** a TrialBalance is created with totalDebits, totalCredits, and isBalanced=true
- **AND** the status is set to draft

#### Scenario: Detect an out-of-balance trial balance

- **GIVEN** GeneralLedgerEntries where total debits do not equal total credits (data integrity issue)
- **WHEN** a trial balance is generated
- **THEN** the TrialBalance is created with isBalanced=false
- **AND** a Nextcloud notification is sent to the financial controller indicating reconciliation is needed

---

### REQ-FRA-006: Annual Report Export

The system must allow authorized users to export a complete annual financial report for a fiscal year in PDF, Excel, XML, and JSON formats. PDF exports must be board-ready (A4, formatted, with cover page and table of contents). Narrative footnotes added to line items must appear in the export.

#### Scenario: Export annual income overview as PDF (must)

- **GIVEN** a completed FiscalYear with finalized FinancialReport records
- **WHEN** a foundation director requests an annual income export with format=PDF
- **THEN** the system generates a board-ready PDF with income grouped by category
- **AND** totals are provided per quarter and per year
- **AND** income is aggregated across all events/sources in the fiscal year
- **AND** the PDF is available for download and attached to the FinancialReport record via FileService

#### Scenario: Export in CSV and PDF (user story)

- **GIVEN** a completed FiscalYear
- **WHEN** a user requests an annual income export
- **THEN** the export is available in both CSV (via ExportService) and PDF format

#### Scenario: Add narrative footnote to a line item

- **GIVEN** a generated annual report
- **WHEN** a user clicks on a line item and adds a free-text note
- **THEN** the note is saved linked to that line item
- **AND** when the report is exported to PDF, the note appears as a footnote linked to the corresponding item

#### Scenario: Export in multiple formats

- **GIVEN** a FinancialReport with reportType=Annual and reportStatus=Approved
- **WHEN** a user selects reportFormat=Excel
- **THEN** the report is exported as an Excel file via ExportService with all GL category totals
- **AND** when reportFormat=XML is selected, the export follows UBL financial report structure

---

### REQ-FRA-007: ESEF/XBRL Annual Report Filing (should-have)

The system must generate ESEF-compliant annual reports as XHTML documents with inline XBRL tags for listed companies filing with the AFM. The output must validate against the ESEF Conformance Suite.

#### Scenario: Generate ESEF filing from finalized financial statements (user story)

- **GIVEN** the consolidated financial statements are finalized for fiscal year 2024
- **WHEN** a CFO initiates ESEF filing via the FinancialReport export action
- **THEN** the annual report is generated as an XHTML document with inline XBRL tags
- **AND** IFRS taxonomy elements are mapped to the organization's chart-of-accounts line items
- **AND** custom extension elements are created where no standard IFRS tag exists
- **AND** block tagging is applied to notes sections
- **AND** the output validates against the ESEF Conformance Suite
- **AND** the generated filing is attached to the FinancialReport record and submittable to AFM's filing system

---

### REQ-FRA-008: Accountability Report Workflow

The system must support a full accountability report lifecycle: draft → submitted → approved or rejected. Reports may be sent to recipients (e.g., subsidizing authority). Overdue reports must be escalated via notification.

#### Scenario: Submit an accountability report

- **GIVEN** an AccountabilityReport with status=draft and all required content completed
- **WHEN** a foundation director submits the report
- **THEN** the status changes to submitted and submissionDate is recorded
- **AND** the recipient organization is notified via NotificationService

#### Scenario: Review and approve a submitted accountability report

- **GIVEN** an AccountabilityReport with status=submitted
- **WHEN** an authorized reviewer approves the report
- **THEN** the status changes to approved and approvalStatus changes to approved
- **AND** the submitting organization is notified of the approval
- **AND** the full transition history is recorded in the audit trail

#### Scenario: Reject an accountability report with reason

- **GIVEN** an AccountabilityReport with status=submitted
- **WHEN** a reviewer rejects the report with a reason
- **THEN** the status changes to rejected and approvalStatus changes to rejected
- **AND** the submitter is notified with the rejection reason

#### Scenario: Flag and escalate an overdue accountability report

- **GIVEN** an AccountabilityReport with status=submitted that has passed its due date
- **WHEN** the OverdueAccountabilityReportJob runs nightly
- **THEN** a Nextcloud notification is sent to the responsible controller
- **AND** the report is flagged in the accountability report list view

---

### REQ-FRA-009: Consolidated Financial Reporting

The system must consolidate financial data from multiple organizations within a ConsolidationGroup, applying inter-company elimination rules, and generate a ConsolidatedReport.

#### Scenario: Generate a consolidated management reporting package

- **GIVEN** a ConsolidationGroup with multiple member organizations and eliminationRules defined
- **WHEN** a group controller initiates consolidation for FiscalYear 2024
- **THEN** a ConsolidatedReport is created combining BalanceSheet data from all member organizations
- **AND** inter-company transactions matching the eliminationRules are eliminated
- **AND** eliminationsApplied is set to true on the ConsolidatedReport

#### Scenario: Consolidated reporting across administrations

- **GIVEN** a ConsolidationGroup with consolidationMethod=full spanning three municipal administrations
- **WHEN** a municipal controller generates the consolidated report
- **THEN** the system produces a single ConsolidatedReport aggregating all administrations
- **AND** the report includes a breakdown per member organization

#### Scenario: Publish a consolidated report

- **GIVEN** a ConsolidatedReport with status=finalized and eliminationsApplied=true
- **WHEN** the group controller sets isPublished=true
- **THEN** the report status changes to published
- **AND** it becomes read-only

---

### REQ-FRA-010: Revenue Stream Management

The system must allow tracking of revenue streams by category, with annual targets, and aggregate revenue across journal entries per stream and fiscal period.

#### Scenario: Create and activate a revenue stream

- **GIVEN** an authenticated user with bookkeeper role
- **WHEN** they create a RevenueStream with streamName="Legesopbrengsten", category="licensing", currency="EUR", isActive=true
- **THEN** the revenue stream is created and available for linking to JournalEntries

#### Scenario: View consolidated income overview

- **GIVEN** multiple JournalEntries linked to different RevenueStreams across fiscal year 2025
- **WHEN** a foundation director requests the income overview for FiscalYear 2025
- **THEN** the system returns income grouped by RevenueStream category
- **AND** totals are shown per quarter (Q1–Q4) and for the full year
- **AND** the overview is available for export via ExportService

---

### REQ-FRA-011: General Ledger Report

The system must provide a General Ledger Report listing all GeneralLedgerEntry records filtered by account, date range, and status, supporting audit review and reconciliation.

#### Scenario: Filter general ledger by account and period

- **GIVEN** posted GeneralLedgerEntries across multiple accounts and months
- **WHEN** a financial controller filters by accountNumber="6000" and date range 2025-01-01 to 2025-03-31
- **THEN** only entries matching the account and date range are returned
- **AND** the response includes opening balance, period movements, and closing balance for the account

#### Scenario: Export general ledger report for audit

- **GIVEN** a filtered general ledger view
- **WHEN** an external auditor requests an export
- **THEN** the export is generated via ExportService in Excel or CSV format
- **AND** the export includes all columns: date, account number, account name, debit, credit, reference, description, status

---

### REQ-FRA-012: Year-End Processing

The system must support structured year-end processing including accruals, provisions, and the final closing sequence. A background job must handle scheduled year-end tasks.

#### Scenario: Process year-end accruals

- **GIVEN** a FiscalYear 2024 approaching its endDate
- **WHEN** a financial controller initiates year-end accrual processing
- **THEN** the system creates JournalEntries for all pending accruals and provisions
- **AND** each entry is marked with journalCode="general" and memo indicating year-end origin

#### Scenario: Validate year-end closing readiness

- **GIVEN** a FiscalYear 2024 with pending draft JournalEntries
- **WHEN** an administrator attempts to close the fiscal year
- **THEN** the system checks for unposted (draft) entries
- **AND** if any exist, the close is blocked with a message listing the unposted entry count
- **AND** if all entries are posted and the trial balance is balanced, the close proceeds

---

### REQ-FRA-013: Financial Dashboard

The system must provide a dashboard with key financial performance indicators (KPIs) and visual charts for income vs expense by period.

#### Scenario: View financial KPI dashboard

- **GIVEN** an authenticated user with access to financial data
- **WHEN** they open the Shillinq dashboard
- **THEN** the dashboard shows four KPI cards: Total Assets, Total Liabilities, Total Equity (current balance sheet), and Net Revenue (current fiscal year)
- **AND** an income vs expense bar chart is shown per period (month or quarter) using CnChartWidget
- **AND** open accountability reports appear in a "My Work" list grouped by status

---

### REQ-FRA-014: Audit-Ready Financial Reporting

All financial records must maintain a full audit trail. The system must support structured evidence export for external auditors covering all financial record mutations.

#### Scenario: Access audit trail for a journal entry

- **GIVEN** a JournalEntry that has been created and subsequently edited
- **WHEN** an external auditor opens the audit trail tab (CnObjectSidebar → CnAuditTrailTab)
- **THEN** all mutations are shown with before/after snapshots, timestamps, and user identifiers
- **AND** the audit trail is exportable via AuditTrailController

#### Scenario: Export accountability evidence package

- **GIVEN** an AccountabilityReport with status=approved and attached DigitalDocuments
- **WHEN** an internal auditor requests a bulk download
- **THEN** all attached files are packaged as a ZIP via FileService.createObjectFilesZip()
- **AND** the download includes the accountability report PDF and all supporting documents

---

### REQ-FRA-015: Metrics and Health Endpoints

Per ADR-006, the app must expose Prometheus metrics and a public health check endpoint.

#### Scenario: Health check verifies OpenRegister connectivity

- **GIVEN** the Shillinq app is running
- **WHEN** `GET /index.php/apps/shillinq/api/health` is called
- **THEN** the response is HTTP 200 JSON with status "ok" and openregister connectivity confirmed

#### Scenario: Prometheus metrics include financial domain counters

- **GIVEN** an admin user
- **WHEN** `GET /index.php/apps/shillinq/api/metrics` is called with admin authentication
- **THEN** the response is Prometheus text format including:
  - `shillinq_health_status`
  - `shillinq_info`
  - `shillinq_journal_entries_total`
  - `shillinq_accountability_reports_total` (labelled by status)
  - `shillinq_fiscal_years_total` (labelled by isClosed)
