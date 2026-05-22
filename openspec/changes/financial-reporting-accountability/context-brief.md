# Context Brief: Financial Reporting & Accountability — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** financial-reporting-accountability
**Platform:** Nextcloud + OpenRegister

## Placement & Information Architecture

**Placement type:** `TOP_MENU` — Top-level menu entry — this functionality earns its own item in the app's left-nav.

**Lives at:** Rapportage

**Rationale:** parent of all reports  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Features (90 total, sorted by market demand)

### Export annual report
**demand: 2523** (841 tender mentions) | Category: document-management
Clustered from 103 mentions: 32 user stories, 61 competitor features, 10 external mentions

### Generate consolidated management reporting packages
**demand: 1684** (547 tender mentions, 20% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Fiscal Year Management
**demand: 1683** (533 tender mentions, 41% competitor coverage) | Category: other
Clustered from 2 mentions: 2 competitor features

### General Ledger and Financial Management
**demand: 1616** (538 tender mentions) | Category: other
Clustered from 1 mentions: 1 external mentions

### Revenue management
**demand: 1602** (530 tender mentions, 6% competitor coverage) | Category: other
Clustered from 4 mentions: 4 competitor features

### Review submitted accountability report
**demand: 240** (76 tender mentions, 5% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 user stories

### Audit-Ready Financial Reporting for Public Sector
**demand: 214** (71 tender mentions) | Category: governance
Clustered from 1 mentions: 1 external mentions

### Consolidated reporting across administrations
**demand: 160** (26 tender mentions, 37% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Send accountability report request to recipient
**demand: 154** (37 tender mentions, 20% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 user stories

### Export annual report as board-ready PDF
**demand: 149** (49 tender mentions, 1% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 user stories

### Group consolidation with real-time consolidated financial reporting
**demand: 134** (38 tender mentions, 8% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Revenue Recognition Tracking
**demand: 131** (23 tender mentions, 28% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Balance sheet generation at any point in time
**demand: 121** (39 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### General Ledger Report
**demand: 119** (37 tender mentions, 4% competitor coverage) | Category: analytics
Clustered from 5 mentions: 5 competitor features

### AI-powered financial report analysis for trends and misclassifications
**demand: 101** (27 tender mentions, 9% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### XBRL/ESEF annual report filing for listed companies
**demand: 94** (26 tender mentions, 7% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 user stories

### Location tracking for multi-site revenue and expense analysis
**demand: 92** (26 tender mentions, 7% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Record asset disposal
**demand: 90** (30 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Balance sheet with current financial position overview
**demand: 90** (28 tender mentions, 3% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Record accruals and adjustments
**demand: 90** (30 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Income vs expense overview charts by period
**demand: 88** (28 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### View deduction overview
**demand: 87** (28 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### View consolidated debt portfolio overview
**demand: 84** (28 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Consolidated domestic and global payments dashboard
**demand: 77** (25 tender mentions, 1% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Inter-company posting with automatic consolidation eliminations
**demand: 46** (14 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Flag and escalate overdue accountability reports
**demand: 36** (12 tender mentions) | Category: governance
Clustered from 1 mentions: 1 user stories

### Write programme accountability texts
**demand: 36** (12 tender mentions) | Category: governance
Clustered from 1 mentions: 1 user stories

### Trial balance with opening, period movement, and closing balances
**demand: 32** (10 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### General ledger with automatic double-entry posting from all modules
**demand: 24** (1 tender mentions, 10% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Statement of changes in equity with period comparison
**demand: 23** (7 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Trial Balance Import
**demand: 21** (4 tender mentions, 4% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 competitor features

### Process year-end accruals and provisions
**demand: 18** (6 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Balance sheet generation with comparative period analysis
**demand: 17** (3 tender mentions, 4% competitor coverage) | Category: other
Clustered from 3 mentions: 3 competitor features

### General ledger & chart of accounts
**demand: 15** (7% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### IFRS 15 multi-element revenue recognition with POC
**demand: 15** (5 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Financial statements with consolidation across multiple companies
**demand: 14** (4 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Tail spend identification and consolidation opportunity analysis
**demand: 10** (2 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### IFRS 15 revenue recognition
**demand: 10** (2 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Propose OZB rates for next fiscal year
**demand: 9** (3 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Combined Balance Sheet Generation
**demand: 8** (4% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Year-End Processing
**demand: 8** (2 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Payroll journal entry auto-posting to general ledger
**demand: 7** (1 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### W-2 and 1099 year-end form generation and filing
**demand: 7** (1 tender mentions, 2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Multi-Ledger Consolidation
**demand: 6** (3% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Financial Reports
**demand: 6** (3% competitor coverage) | Category: other
Clustered from 2 mentions: 2 competitor features

### Profit & Loss with multi-dimensional analysis and drill-down
**demand: 6** (3% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Advanced financial close
**demand: 5** (1 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Year-end payroll statements
**demand: 5** (1 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### GL Outlier Detection: ML anomaly detection in general ledger at point of entry
**demand: 5** (1 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Consolidated Trial Balance
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Generate balance sheet
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 2 mentions: 1 user stories, 1 competitor features

### Annual accounts preparation with balance sheet and P&L
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Generate financial statements
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 3 mentions: 1 user stories, 2 competitor features

### Generate trial balance
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Consolidation
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Consolidation Journals
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Period-Based Consolidation
**demand: 4** (2% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Consolidate year-end actual figures
**demand: 3** (1 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Consolidate verbonden partijen
**demand: 3** (1 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Calculate asset depreciation
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 2 mentions: 2 user stories

### Universal Journal
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Horizontal Group Analysis
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Produce balance sheet and results account
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Consolidated Return Support
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Inline XBRL (iXBRL) Generation
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Jaarrekening generatie
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Inter-Company Matching
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Minority interest calculation
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 3 mentions: 1 user stories, 2 competitor features

### Combined Income Statement
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Year-end checklist
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Consolidation adjustments
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 user stories

### Revenue Recognition
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Elimination Entries
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Period Alignment
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Deferred Revenue/Expense
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Generate Turap
**demand: 1** | Category: other
Clustered from 1 mentions: 1 user stories

### triple-entry
**demand: 1** | Category: other
Clustered from 1 mentions: 1 external mentions

### month-end-close
**demand: 1** | Category: other
Clustered from 1 mentions: 1 external mentions

### View consolidated cash position
**demand: unknown** | Category: other
Clustered from 2 mentions: 2 user stories

### Recognize revenue
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Multi-currency consolidation with CTA and temporal method
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Year-over-year comparison
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Generate income summary
**demand: unknown** | Category: ai
Clustered from 1 mentions: 1 user stories

### Confirm reconciliation
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Post opening balance
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Prepare jaarrekening
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Prepare verplichte paragrafen
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### View explanatory notes per programme
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Monitor reserve and provision mutation disclosures
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Entity-level close
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

## User Stories (3 linked)

### Story 1: XBRL/ESEF annual report filing for listed companies
**Priority:** should-have
As a CFO, I want to generate our annual financial report in ESEF (European Single Electronic Format) with inline XBRL tagging, so that we comply with EU Transparency Directive requirements for listed companies and can file with the AFM.

**Acceptance Criteria:**
GIVEN the consolidated financial statements are finalized WHEN I initiate ESEF filing THEN the annual report is generated as an XHTML document with inline XBRL tags AND IFRS taxonomy elements are mapped to our line items AND custom extensions are created where no standard tag exists AND block tagging is applied to notes AND the output validates against the ESEF Conformance Suite AND the filing is submittable to AFM's filing system

### Story 2: Add narrative notes to the financial report
**Priority:** should
As a foundation director, I want to add explanatory notes to specific line items in the annual report, so that auditors and board members understand exceptional costs or income fluctuations.

**Acceptance Criteria:**
GIVEN a generated annual report WHEN I click on a line item THEN I can add a free-text note
GIVEN a note added to a line item WHEN the report is exported to PDF THEN the note appears as a footnote linked to that item

### Story 3: Export annual income overview
**Priority:** must
As a foundation director, I want to export a consolidated overview of all income sources (registrations, grants, sponsorships) for the fiscal year, so that I can prepare the annual financial report for the board.

**Acceptance Criteria:**
GIVEN a completed fiscal year WHEN I request an annual income export THEN the system generates a structured overview grouped by income category with totals per quarter and year
GIVEN the export WHEN downloaded THEN it is available in both CSV and PDF format
GIVEN multiple events in one year WHEN viewing the report THEN income is aggregated across all events

## Stakeholders (38 linked)

### CFO / Financial Director
Chief Financial Officer responsible for financial reporting, internal controls, SOX compliance (if applicable), and financial governance. Interfaces with audit committee and external auditors.
**Responsibilities:** Financial reporting and annual accounts, internal controls (SOX Section 302/404 certification), risk management, treasury, tax compliance, audit coordination, dividend proposals
**Pain points:** SOX compliance documentation burden, coordinating with external auditors, ensuring internal control effectiveness, managing financial reporting deadlines, audit committee preparation workload
**Goals:** Automated internal control documentation, streamlined audit processes, real-time financial governance dashboards, efficient committee reporting

### External Auditor
Independent auditor who audits annual accounts and reports to shareholders. In Dutch governance, appointed by AGM and reports to audit committee. Subject to auditor rotation requirements.
**Responsibilities:** Auditing annual financial statements, reporting to audit committee, attending AGM for shareholder questions, assessing internal controls (SOX 404 if applicable), issuing management letter, evaluating going concern
**Pain points:** Limited digital access to governance documentation, manual evidence collection for audit procedures, difficulty tracking management representations, coordinating with internal audit and audit committee
**Goals:** Digital audit evidence repository, secure access to board minutes and resolutions, automated management representation tracking, integrated communication with audit committee

### Controller / Financial Controller
Responsible for financial oversight, budget control, and financial approval in decision chains. Reviews business cases and validates financial impact of proposals.
**Responsibilities:** ["Reviewing and approving financial aspects of proposals", "Budget monitoring and variance reporting", "Financial validation in procurement approval chains", "Preparing financial reports for MT and board", "Ensuring compliance with financial policies and mandates", "Cost-benefit analysis for investment decisions"]
**Pain points:** ["Approval requests arriving without proper financial substantiation", "No visibility into committed vs actual spend across approvals", "Manual tracking of budget approvals across departments", "Difficulty enforcing financial policies consistently", "Last-minute approval requests bypassing normal process"]
**Goals:** ["Automated budget check in approval workflows", "Real-time budget commitment tracking", "Standardized business case template for proposals", "Financial approval audit trail for compliance"]

### External Accountant
Professional accountant engaged by larger associations or when statutes require it. Provides independent audit opinion on financial statements. May replace or supplement the kascommissie for larger organizations.
**Responsibilities:** ["Audit annual financial statements", "Provide accountant's report (accountantsverklaring)", "Advise on financial controls and compliance", "Report to board and/or ALV", "Verify compliance with legal requirements"]
**Pain points:** ["Incomplete or unstructured financial data from association", "Tight timeline between financial year end and ALV", "Association boards unfamiliar with audit requirements", "Accessing supporting documentation"]
**Goals:** ["Timely access to complete financial records", "Structured financial data for efficient audit", "Clear communication channel with board"]

### Group Controller
Controller responsible for group-level reporting
**Responsibilities:** Consolidation, intercompany elimination, group reporting
**Pain points:** Manual consolidation in spreadsheets; IFRS complexity
**Goals:** Automated consolidation with IFRS/GAAP compliance

### External Auditor
Big-4 or mid-tier audit firm
**Responsibilities:** Statutory audit, internal controls review
**Pain points:** Complex audit trail across systems
**Goals:** Complete digital audit trail; SOX compliance

### Internal Auditor
Periodically audits document management and archiving practices to verify compliance with the Archiefwet, Baseline Informatiebeveiliging Overheid (BIO), and internal policies.
**Responsibilities:** conducting audits of document registration and archiving processes, assessing retention schedule adherence, reviewing access control logs, reporting non-conformities, following up on remediation actions
**Pain points:** incomplete audit trails in legacy DMS, difficulty extracting structured compliance evidence, no automated alerting on policy deviations, manual cross-referencing between systems
**Goals:** exportable audit logs with full document lifecycle evidence, automated compliance dashboards, policy deviation alerts, standardised audit report generation

### Municipal Controller
Financial and operational controller responsible for the planning-and-control cycle, including P&C rapportages, jaarrekening inputs, and budget monitoring for the college van B&W. Owns the authoritative management reporting output.
**Responsibilities:** coordinating periodic P&C rapportages, monitoring budget versus realisation, producing bestuursrapportages, ensuring consistency between financial and operational reporting, liaising with accountants
**Pain points:** reconciling data from multiple financial systems, narrative writing being manual and time-consuming, last-minute changes invalidating already-approved figures, difficulty distributing live dashboards to councillors securely
**Goals:** automate recurring reporting cycles, provide the college with real-time budget dashboards, reduce the production lead time of the jaarverslag

### Financial Controller
Monitors personnel costs against budget and ensures payroll data reconciles with the financial administration. Prepares workforce cost forecasts for P&C cycles.
**Responsibilities:** reconciling payroll totals with financial ledger, monitoring personnel budget variances, providing workforce cost forecasts for planning and control cycles, auditing payroll mutations for correctness
**Pain points:** inconsistent data between HR and financial systems, manual reconciliation processes, late or incomplete payroll exports, difficulty attributing costs to correct cost centers
**Goals:** real-time payroll-to-ledger reconciliation, automated cost center allocation, reliable personnel cost forecasting, reduced manual reconciliation effort

### Learning & Development Coordinator
Plans and administers employee training programs, mandatory government compliance courses, and certifications. Ensures regulatory training requirements for specific public sector roles are met and evidenced.
**Responsibilities:** cataloguing and enrolling employees in training programs, tracking completion of mandatory compliance courses (e.g. integrity, information security), managing training budgets, producing learning reports, coordinating external training providers
**Pain points:** no central overview of mandatory certification expiry dates, manual enrollment and completion tracking across multiple learning platforms, difficulty proving compliance to auditors, training budgets not linked to HR data
**Goals:** automated alerts for expiring certifications, centralized learning record per employee, compliance reporting dashboards for auditors, integrated training catalog with self-enrollment, budget tracking per department

## Data Model — Entities for This Spec (11)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### AccountabilityReport (`schema:Report`)
_An official accountability report submitted by an organization for a fiscal period covering financial position and transactions_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportNumber | string | Yes | Unique identifier for the accountability report |
| reportDate | datetime | Yes | Date the report was generated |
| submissionDate | datetime | No | Date the report was submitted to relevant authority |
| status | string | Yes | Status (draft, submitted, approved, rejected) |
| content | string | No | Full text content of the accountability report |
| approvalStatus | string | Yes | Approval status (pending, approved, rejected) |

**Relations:**
- → FiscalYear (many-to-one)
- → Organization (many-to-one)
- → Person (many-to-one)
- → DigitalDocument (one-to-many)

### BalanceSheet (`schema:Table`)
_A financial statement showing assets, liabilities, and equity at a specific point in time_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportDate | datetime | Yes | Date of the balance sheet snapshot |
| totalAssets | number | No | Total assets in base currency |
| totalLiabilities | number | No | Total liabilities in base currency |
| totalEquity | number | No | Total equity in base currency |
| currency | string | Yes | Currency code for amounts |
| status | string | Yes | Status (draft, final, published) |

**Relations:**
- → FiscalYear (many-to-one)
- → Organization (many-to-one)
- → GeneralLedgerEntry (one-to-many)

### ConsolidatedReport (`schema:Report`)
_A consolidated financial report combining data from multiple organizations with automatic inter-company eliminations_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportNumber | string | Yes | Unique identifier for the consolidated report |
| reportDate | datetime | Yes | Date of the consolidated report |
| consolidationMethod | string | Yes | Method used for consolidation |
| status | string | Yes | Status (draft, finalized, published, archived) |
| eliminationsApplied | boolean | No | Whether inter-company eliminations have been applied |
| isPublished | boolean | No | Whether the consolidated report is published |

**Relations:**
- → ConsolidationGroup (many-to-one)
- → FiscalYear (many-to-one)
- → BalanceSheet (one-to-many)

### ConsolidationGroup (`schema:Organization`)
_A group of organizations consolidated together for consolidated financial reporting across administrations_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the consolidation group |
| consolidationMethod | string | Yes | Method used for consolidation (full, proportional, equity) |
| status | string | Yes | Status of the consolidation group |
| parentOrganization | string | No | Parent organization identifier |
| eliminationRules | object | No | Consolidation elimination rules for inter-company transactions |

**Relations:**
- → Organization (one-to-many)
- → ConsolidatedReport (one-to-many)

### FinancialReport (`schema:Report`)
_Exported financial statements (annual, management, or consolidated) generated for a fiscal year._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportType | string | Yes | Annual, Management, or Consolidated |
| reportFormat | string | Yes | Export format: PDF, Excel, XML, or JSON |
| reportStatus | string | No | Draft, Approved, or Published |
| generatedAt | dateTime | Yes | Timestamp of report generation |

**Relations:**
- → FiscalYear (many-to-one)

### FiscalYear (`schema:Event`)
_An accounting period representing a fiscal year for financial reporting and regulatory compliance._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| year | integer | Yes | The fiscal year number (e.g., 2024) |
| startDate | date | Yes | The first day of the fiscal period |
| endDate | date | Yes | The last day of the fiscal period |
| isClosed | boolean | No | Whether the fiscal year is closed for amendments |
| closingDate | date | No | Date when the fiscal year was officially closed |

**Relations:**
- → FinancialReport (one-to-many)
- → JournalEntry (one-to-many)

### GeneralLedgerAccount (`schema:Product`)
_A chart-of-accounts entry for tracking debits, credits, and account balances across asset, liability, equity, revenue, and expense categories._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| accountNumber | string | Yes | The unique account code (e.g., 1000, 4100) |
| accountName | string | Yes | The descriptive account name |
| accountType | string | Yes | Account classification: Asset, Liability, Equity, Revenue, or Expense |
| currency | string | Yes | ISO 4217 currency code for the account |
| currentBalance | object | No | Current balance as {value, currency} following MonetaryAmount schema |

**Relations:**
- → JournalEntry (one-to-many)

### GeneralLedgerEntry (`schema:Thing`)
_An individual entry in the general ledger representing a financial transaction with debit and credit amounts_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| entryDate | datetime | Yes | Date of the GL entry |
| accountNumber | string | Yes | General ledger account code |
| accountName | string | Yes | Name of the GL account |
| debitAmount | number | No | Debit amount in base currency |
| creditAmount | number | No | Credit amount in base currency |
| description | string | Yes | Description of the transaction |
| reference | string | No | Reference document number or transaction ID |
| status | string | Yes | Status (draft, posted, reversed) |

**Relations:**
- → FiscalYear (many-to-one)
- → Organization (many-to-one)
- → APTransaction (many-to-one)

### JournalEntry (`custom`)
_A balanced transaction record affecting two or more GL accounts (debits equal credits)._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| entryDate | datetime | Yes | Date of the journal entry |
| entryNumber | string | Yes | Unique sequential journal entry number |
| description | string | Yes | Transaction description |
| debitAmount | number | Yes | Debit amount in EUR |
| creditAmount | number | Yes | Credit amount in EUR |
| isBalanced | boolean | Yes | Whether debits equal credits |
| accountCode | string | Yes | General ledger account number |
| journalCode | string | Yes | Journal type (sales, bank, cash, general, etc.) |
| reference | string | No | External reference (invoice, check, or document number) |
| vatAmount | number | No | VAT/BTW amount (21% standard, 9% reduced, etc.) |
| departmentCode | string | No | Cost center or department code |
| memo | string | No | Additional notes or clarification |

**Relations:**
- → GeneralLedgerAccount (many-to-many)
- → FiscalYear (many-to-one)

### RevenueStream (`schema:Offer`)
_A categorized source or type of revenue for tracking income by origin and supporting revenue management analysis._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| streamName | string | Yes | The name of the revenue source |
| category | string | Yes | Revenue classification (e.g., product sales, service fees, licensing) |
| currency | string | Yes | ISO 4217 currency code |
| annualTarget | object | No | Target revenue as {value, currency} following MonetaryAmount schema |
| isActive | boolean | No | Whether this revenue stream is currently active |

**Relations:**
- → JournalEntry (one-to-many)

### TrialBalance (`schema:Table`)
_A report listing all general ledger accounts with debit or credit balances for verification and audit purposes_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportDate | datetime | Yes | Date of the trial balance |
| totalDebits | number | No | Total of all debit balances |
| totalCredits | number | No | Total of all credit balances |
| isBalanced | boolean | No | Whether debits equal credits |
| status | string | Yes | Status (draft, verified, final) |
| preparedBy | string | No | Name or identifier of person who prepared the trial balance |

**Relations:**
- → FiscalYear (many-to-one)
- → Organization (many-to-one)
- → GeneralLedgerEntry (one-to-many)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AuditorStatement, AwardDecision, AwardNotice, BankAccount, Bid, BidEvaluation, BiddingRound, BlanketPurchaseOrder, Branch, Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CallOffOrder, CashAccount, CatalogItem, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, CurrencyBalance, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExemptionCertificate, ExpenditureEscalation, ExpenditureRequest, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FXExposure, FinancialDecision, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, FundingSource, GoodsReceipt, GovernmentEntity, Grant, GrantPortfolio, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, LiquidityForecast, Location, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentBatch, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, PricingRule, ProcurementAuditLog, ProcurementCatalog, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, PurchaseRequisition, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RequestForQuotation, RiskCriteria, Role, SavingsOpportunity, ScheduledPayment, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, StatementOfWork, SubmissionDossier, Subscription, SubsidyApplication, SubsidyScheme, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, TaxConfiguration, TaxDeclaration, TaxExemption, TaxLot, TaxRate, TaxReturn, TaxableTransaction, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TreasuryTask, User, UserPreference, VATReturn, VendorBill, WOZAssessment, XBRLInstance, XBRLTaxonomy

## Company-Wide Architecture Rules (13 ADRs)

These rules are MANDATORY for all Conduction apps.

### ADR-001-data-layer
- ALL domain data → OpenRegister objects. NO custom Entity/Mapper for domain data.
- App config → `IAppConfig`. NOT OpenRegister.
- Schemas: PascalCase, schema.org vocabulary where equivalent exists, explicit types.
- Cross-entity references: OpenRegister relations (register+schema+objectId). NO foreign keys.
- Register templates: `lib/Settings/{app}_register.json` (OpenAPI 3.0 + x-openregister).
- Seed data: 3-5 realistic objects per schema using `@self` envelope (`register`, `schema`, `slug`).
  Use general org data (municipality/consultancy), NOT context-specific. Include in design.md.
- Breaking schema changes → new migration in repair step. NEVER modify existing migrations.

### OpenRegister + @conduction/nextcloud-vue — DO NOT REBUILD

The platform provides 258+ backend methods and 69+ frontend components. Apps ONLY build
custom logic for domain-specific business rules. Everything below is provided for FREE.

**CRUD & Data Management** (use ObjectService + CnIndexPage + CnDetailPage):
- Single & bulk create, read, update, delete — `ObjectService.saveObject()`, `deleteObject()`
- List with pagination, sorting, filtering — `ObjectService.findAll()` + `CnDataTable`
- Schema-driven forms — `CnFormDialog` (auto-generates from schema) or `CnAdvancedFormDialog`
- Detail views — `CnDetailPage` with `CnDetailGrid`, `CnDetailCard` sections
- Record merging/deduplication — `ObjectService.mergeObjects()`
- Object locking — `ObjectService.lockObject()` / `unlockObject()`

**Import & Export** (use ImportService/ExportService + CnMassImportDialog/CnMassExportDialog):
- CSV, Excel, JSON import with intelligent field mapping — `ImportService`
- CSV, Excel, JSON export with column selection — `ExportService`
- Bulk import with validation and progress — `CnMassImportDialog`
- Filtered export with format picker — `CnMassExportDialog`
- NO custom import dialogs, parsers, upload handlers, or export controllers

**Search & Discovery** (use IndexService + CnFilterBar + CnFacetSidebar):
- Full-text search with field weighting — `IndexService`
- Faceted navigation with counts — `FacetBuilder` + `CnFacetSidebar`
- Semantic search with embeddings — `VectorizationService`
- Hybrid search (keyword + semantic) — automatic
- Search analytics — `SearchTrailService` (popular terms, activity)
- NO custom search endpoints, query builders, or search pages

**File Management** (use FileService + CnObjectSidebar):
- Upload (single/multipart), download, share links — `FileService`
- File tagging, public/private toggle — `FileService`
- Bulk download as ZIP — `createObjectFilesZip()`
- Text extraction from PDFs/Office docs — `TextExtractionService`
- File tab in object sidebar — `CnObjectSidebar` → `CnFilesTab`
- NO custom file upload components, file controllers, or download handlers

**Audit & Compliance** (use AuditTrailService + CnObjectSidebar):
- Full change tracking with before/after snapshots — automatic
- Audit trail tab — `CnObjectSidebar` → `CnAuditTrailTab`
- GDPR data subject access requests — `inzageverzoek()`, `verwerkingsregister()`
- Audit export and analytics — `AuditTrailController`
- NO custom audit logging, change tracking, or compliance controllers

**Dashboard & Analytics** (use CnDashboardPage + CnChartWidget + CnStatsBlock):
- Drag-drop widget dashboard — `CnDashboardPage` with GridStack
- KPI cards — `CnKpiGrid`, `CnStatsBlock`, `CnStatsPanel`
- Charts (line/bar/pie/donut) — `CnChartWidget` (ApexCharts)
- Data tables as widgets — `CnTableWidget`
- Editable data grids — `CnObjectDataWidget`
- NO custom dashboard layouts, chart components, or KPI cards

**Forms & Dialogs** (use CnFormDialog + schema-driven generation):
- Auto-generated create/edit forms — `CnFormDialog` reads schema → generates fields
- JSON/metadata editing — `CnAdvancedFormDialog` with Properties/Data/Metadata tabs
- Schema editor — `CnSchemaFormDialog`
- Delete/Copy/Mass operations — `CnDeleteDialog`, `CnCopyDialog`, `CnMassDeleteDialog`
- NO custom form components, validation logic, or dialog wrappers

**Navigation & Pagination** (use CnPagination + CnActionsBar + useListView):
- Pagination control with size selector — `CnPagination`
- Action bar (add, search, toggle views) — `CnActionsBar`
- List state management — `useListView` composable (handles search, filter, sort, page)
- Detail state management — `useDetailView` composable
- NO custom pagination logic, debounced search, or list state management

**Authorization & RBAC** (use AuthorizationService + PropertyRbacHandler):
- Role-based access control — `AuthorizationService`
- Field-level permissions — `PropertyRbacHandler`
- Object-level restrictions — `PermissionHandler`
- Authorization audit — `AuthorizationAuditService`
- NO custom permission checks, role systems, or access control middleware

**Webhooks & Events** (use WebhookService):
- Create, test, retry webhooks — `WebhookService`
- CloudEvents format — automatic
- Event subscriptions — selective per schema/action
- NO custom webhook controllers or event dispatchers

**Notifications & Activity** (use NotificationService + ActivityService):
- Nextcloud notifications — `NotificationService`
- Activity feed — `ActivityService`
- Calendar events — `CalendarEventService`
- Deck/Kanban cards — `DeckCardService`

**Store & State** (use createObjectStore + plugins):
- Object stores — `createObjectStore(name)` generates Pinia CRUD store
- Store plugins: `auditTrails`, `files`, `lifecycle`, `relations`, `search`, `selection`
- Column/field/filter generation from schema — `columnsFromSchema()`, `fieldsFromSchema()`
- NO custom Pinia stores for CRUD, Vuex, or manual API call management

**Chat & AI** (use ChatService):
- Multi-turn conversation — `ChatService`
- RAG-based knowledge retrieval — `ContextRetrievalHandler`
- LLM response generation — `ResponseGenerationHandler`

**Data Retention & Archival** (use ArchivalService):
- Legal hold — `LegalHoldService`
- Destruction schedules — `DestructionService`
- Retention policies — `RetentionService`

**Semantic & Hybrid Search** (use SolrController + SettingsController):
- Semantic search via vector embeddings — `SettingsController.semanticSearch()`
- Hybrid search (keyword + semantic combined) — `SolrController.hybridSearch()`
- Vector embedding generation — `VectorizationService`
- NO custom search algorithms — configure via OpenRegister settings

**GraphQL API** (use GraphQLController):
- Query objects across schemas via GraphQL — `GraphQLController.execute()`
- Alternative to REST for complex cross-entity queries

**Organization / Multi-Tenancy** (use OrganisationController):
- Organization CRUD — `OrganisationController`
- Tenant-scoped data isolation — automatic via `TenantLifecycleService`
- NO custom multi-tenancy logic

**Task & Workflow Management** (use TasksController + WorkflowEngineController):
- Task creation and tracking — `TasksController`
- Workflow orchestration — `WorkflowEngineRegistry`
- Scheduled workflows — `ScheduledWorkflowController`
- NO custom task/workflow systems

**Text Extraction** (use FileTextController):
- Extract text from PDFs and Office docs — `TextExtractionService`
- Entity recognition (PII detection) — `EntityRecognitionHandler`
- Content anonymization — automatic

**Timeline & Stages** (use CnTimelineStages):
- Workflow progression visualization — `CnTimelineStages` component
- Stage tracking with status colors

### What apps SHOULD build (custom business logic only):
- External API integrations (SAP, Peppol, TenderNed, etc.)
- PDF/document generation with business-specific templates
- Workflow triggers and business rules specific to the domain
- Notification dispatch with app-specific event types
- Custom settings pages with app-specific configuration
- Background jobs for domain-specific processing

### ADR-002-api
- URL pattern: `/index.php/apps/{app}/api/{resource}` — lowercase plural, hyphens.
- Methods: GET=read, POST=create, PUT=update, DELETE=remove. No custom methods.
- Pagination: support `_page` + `_limit`. Response includes `total`, `page`, `pages`.
- Errors: appropriate HTTP status + `message` field. NO stack traces in responses.
- Auth: Nextcloud built-in only. NO custom login/session/token flows.
- Public endpoints: annotate `#[PublicPage]` + `#[NoCSRFRequired]`. Register CORS OPTIONS route.

### ADR-003-backend
- **Controller → Service → Mapper** (strict 3-layer). Controllers NEVER call mappers directly.
- Controllers: thin (<10 lines/method). Routing + validation + response only.
- Services: ALL business logic. Stateless — no instance state between requests.
- Mappers: DB CRUD only. No business logic.
- DI: constructor injection with `private readonly`. NO `\OC::$server` or static locators.
- Entity setters: POSITIONAL args only. `$e->setName('val')` — NEVER `$e->setName(name: 'val')`.
  (`__call` passes `['name' => val]` but `setter()` uses `$args[0]`.)
- Routes: `appinfo/routes.php`. Specific routes BEFORE wildcard `{slug}` routes.
- Config: `IAppConfig` with sensitive flag for secrets. NEVER read DB directly.
- Lifecycle: schema init via repair steps (`IRepairStep`), background via job queue, events via dispatcher.
- **Spec traceability**: every class and public method MUST have `@spec` PHPDoc tag(s) linking to
  the OpenSpec change that caused it: `@spec openspec/changes/{name}/tasks.md#task-N`.
  Multiple `@spec` tags allowed (code touched by multiple changes). File-level `@spec` in header docblock.
  This enables: code → docblock → spec traceability alongside code → git blame → commit → issue → spec.

### ADR-004-frontend
- **Vue 2 + Pinia + @nextcloud/vue + @conduction/nextcloud-vue**. NO Vuex. Options API only.
- State: Pinia stores in `src/store/modules/`. Use `createObjectStore` for OpenRegister CRUD.
- `fetch()` for API calls — NOT axios. Loading state with `try/finally`.
- Translations: ALL user-visible strings via `t(appName, 'text')`. NO hardcoded strings.
- CSS: ONLY Nextcloud CSS variables. NO hardcoded colors. NEVER reference `--nldesign-*`.
- Router: history mode, base `/index.php/apps/{app}/`, catch-all `*` redirects to `/`.
- OpenRegister dependency: settings returns `openRegisters` (bool) + `isAdmin`.
  Show empty state if OR missing. NEVER use `OC.isAdmin` — get from backend.

### @conduction/nextcloud-vue — ALWAYS check before building custom

**Pages & Layout:**
  `CnIndexPage` (schema-driven list+CRUD) | `CnDetailPage` (detail+sidebar) |
  `CnPageHeader` (title+icon) | `CnActionsBar` (add+search+toggle)

**Data Display:**
  `CnDataTable` (sortable+paginated) | `CnCardGrid` + `CnObjectCard` (card views) |
  `CnDetailGrid` (label-value pairs) | `CnFilterBar` (search+filters) |
  `CnFacetSidebar` (faceted filters) | `CnPagination` | `CnCellRenderer` (type-aware)

**Forms & Dialogs:**
  `CnFormDialog` (schema-driven create/edit) | `CnAdvancedFormDialog` (properties+JSON+metadata) |
  `CnSchemaFormDialog` (JSON Schema editor) | `CnTabbedFormDialog` (tabbed form framework) |
  `CnDeleteDialog` | `CnCopyDialog`

**Mass Actions:**
  `CnMassDeleteDialog` | `CnMassCopyDialog` | `CnMassExportDialog` (CSV/JSON/XML) |
  `CnMassImportDialog` (upload+summary) | `CnMassActionBar` (floating selection bar)

**Dashboard & Widgets:**
  `CnDashboardPage` (GridStack drag-drop layout) | `CnDashboardGrid` (layout engine) |
  `CnWidgetWrapper` (widget shell) | `CnWidgetRenderer` (NC Dashboard API v1/v2) |
  `CnChartWidget` (ApexCharts: area/line/bar/pie/donut/radial) |
  `CnTableWidget` (data table widget) | `CnTileWidget` (quick-access tile) |
  `CnInfoWidget` (label-value grid) | `CnKpiGrid` (responsive KPI layout) |
  `CnStatsBlock` (metric card) | `CnStatsPanel` (stats sections) | `CnProgressBar` |
  `CnObjectDataWidget` (schema-driven editable data grid, inline edit + save via objectStore) |
  `CnObjectMetadataWidget` (read-only object metadata display)

**UI Elements:**
  `CnStatusBadge` | `CnEmptyState` | `CnIcon` (MDI) | `CnCard` | `CnDetailCard` |
  `CnRowActions` | `CnTimelineStages` (workflow progression) |
  `CnUserActionMenu` (user context menu) | `CnJsonViewer` (CodeMirror)

**Detail Sidebar:**
  `CnObjectSidebar` (Files/Notes/Tags/Tasks/Audit tabs) | `CnIndexSidebar` |
  `CnNotesCard` (inline notes) | `CnTasksCard` (inline tasks)

**Settings:**
  `CnSettingsSection` + `CnVersionInfoCard` (MUST be first on admin pages) |
  `CnSettingsCard` | `CnConfigurationCard` | `CnRegisterMapping`
  User settings: `NcAppSettingsDialog` (NOT `NcDialog`)

**Composables:**
  `useListView` (search/filter/sort/pagination) | `useDetailView` (load/edit/delete) |
  `useSubResource` (related items) | `useDashboardView` (widgets/layout/edit)

**Store Plugins:**
  `auditTrailsPlugin` | `relationsPlugin` | `filesPlugin` | `lifecyclePlugin` |
  `selectionPlugin` | `searchPlugin` | `registerMappingPlugin`

**Utilities:**
  `columnsFromSchema()` | `filtersFromSchema()` | `fieldsFromSchema()` |
  `formatValue()` | `buildHeaders()` | `buildQueryString()`

### Page Construction Patterns (follow these recipes)

**App.vue:** `NcContent` → 3 states: loading (`NcLoadingIcon`), no-OpenRegister (`NcEmptyContent`),
  ready (`MainMenu` + `NcAppContent` + `router-view` + optional `CnIndexSidebar`).
  Inject `sidebarState` for child components. `created()` calls `initializeStores()`.

**MainMenu:** `NcAppNavigation` with `NcAppNavigationItem` per route (icon + name + `:to`).
  Footer: settings link via `NcAppNavigationSettings`.

**Dashboard:** `CnDashboardPage` with `CnStatsBlock` KPIs (4 cards: open/overdue/value/completed),
  status distribution chart, "My Work" list (grouped: overdue → due this week → rest).
  Fetch all collections in parallel via `Promise.all`. Widget templates via `#widget-{id}` slots.

**Index page:** `CnIndexPage` with `useListView(entityType, { sidebarState, objectStore })`.
  Inject sidebarState. Row click → `$router.push({ name: 'EntityDetail', params: { id } })`.
  Add button → new entity detail with id='new'.

**Detail page:** Two modes — edit (form component) / view (`CnDetailPage` + `CnDetailCard` sections).
  Header actions: Edit + Delete buttons. Related entities in table inside `CnDetailCard`.
  Props: `entityId` from route. `isNew = entityId === 'new'`. Sidebar via `CnObjectSidebar`.

**Settings:** `CnVersionInfoCard` (FIRST, always) → `CnRegisterMapping` → `CnSettingsSection` per feature.
  Load settings from `GET /api/settings`. Save via `POST /api/settings`.
  Re-import button calls `POST /api/settings/load`.

**Router:** Flat routes (no nesting), all named, props via arrow function for params.
  Routes: `/` (Dashboard), `/{entities}` (list), `/{entities}/:id` (detail), `/settings`.

**Store init:** `initializeStores()` in `store/store.js` — fetches settings, then calls
  `objectStore.registerObjectType(name, schemaSlug, registerSlug)` for each entity.
  Object store uses `createObjectStore` with plugins (files, auditTrails, relations).
  Settings store: Pinia `defineStore` with `fetchSettings()` and `saveSettings()`.

### ADR-005-security
- Auth: Nextcloud built-in ONLY. NO custom login, sessions, tokens, password storage.
- Admin check: `IGroupManager::isAdmin()` on BACKEND. Frontend-only checks = vulnerability.
- Multi-tenant isolation: enforce at API/service level, not UI only.
- NO PII in logs, error responses, or debug output.
- File uploads: validate type + size before storage.
- API responses: NO stack traces, SQL, or internal paths.

### ADR-006-metrics
- Every app: `GET /api/metrics` (Prometheus text, admin auth) + `GET /api/health` (JSON, public).
- Metric names: `{app}_` prefix. MUST include `{app}_health_status` and `{app}_info`.
- Health check MUST verify OpenRegister connectivity (for apps that depend on it).

### ADR-007-i18n
- Minimum: Dutch (nl) + English (en) translations.
- PHP: `$this->l->t('key')`. JS: `t(appName, 'key')`.
- API field names: English. Date/number formatting: respect user locale.
- Each app with OpenRegister: define `register-i18n` spec listing translatable fields.

### ADR-008-testing
- Every new PHP service/controller → PHPUnit tests in `tests/Unit/` (≥3 methods).
- Every new Vue component → test file (if test framework exists).
- Every new API endpoint → Newman/Postman collection in `tests/integration/`.
- Every spec scenario → browser test (GIVEN/WHEN/THEN verified via Playwright).
- All tests MUST pass in `composer check:strict`.

### ADR-009-docs
- Every user-facing feature → docs in `docs/` with screenshots from running app.
- English primary, Dutch recommended. Update docs when behavior changes.

### ADR-010-nl-design
- ALL UI: CSS custom properties from NL Design System tokens. NO hardcoded colors, fonts, spacing.
- Theme switching: support `nldesign` app's token sets (Rijkshuisstijl, Utrecht, municipality-specific).
- Components: `@nextcloud/vue` primary. Custom components styled via NL Design tokens only.
- Scoped styles: ALL `<style>` blocks MUST use `scoped` attribute.
- WCAG AA mandatory: keyboard-navigable, labelled forms, color not sole conveyor, alt text on images.
- Responsive: work from 320px to 1920px. Critical features accessible at 768px.
- Specs: reference token names ("primary action color") NOT hex values. Include a11y verification in ACs.
- Exception: PDF generation (docudesk) may use fixed dimensions. Admin screens MAY simplify but MUST meet WCAG AA.

### ADR-011-schema-standards
- schema.org types/properties as primary vocabulary (`schema:Person`, `schema:Organization`, `schema:Event`).
- Contact schemas: align with vCard properties (`fn`, `email`, `tel`, `adr`).
- Dutch government fields: mapping layer translating between international standards and Dutch APIs (VNG, ZGW).
- NO custom property names when schema.org equivalent exists.
- Relations: OpenRegister relation mechanism (register + schema + objectId). NO foreign keys or embedded objects.
- Versioning: removing/renaming properties = BREAKING → migration via repair step. Adding optional = non-breaking.
- Specs MUST define data models using schema.org vocabulary; design docs MUST include schema definitions with types, required flags, relations.
- Exception: app-specific workflow states (pipeline stages, process statuses) MAY use custom vocabularies.

### ADR-012-deduplication
- Before proposing new capability: search OpenRegister specs + services for overlap. Reference + justify if similar exists.
- Design docs MUST include "Reuse Analysis" listing which OpenRegister services are leveraged.
- If logic could benefit other apps → propose adding to OpenRegister core, not app-specific.
- Tasks MUST include "Deduplication Check" verifying no overlap with:
  ObjectService, RegisterService, SchemaService, ConfigurationService, shared specs, @conduction/nextcloud-vue.
- Document findings even if "no overlap found".
- Exception: OpenRegister checks internal duplication only. nldesign checks token sets. nextcloud-vue checks own components.

### ADR-013-container-pool
# ADR-013: Unified Container Pool

**Status:** accepted
**Date:** 2026-04-12

## Context

Specter (intelligence/research) and Hydra (build/review/merge) both run LLM workloads in Docker containers. Today they operate independently: Hydra spins up builder/reviewer/security containers on demand, Specter has a separate `run_llm_containers.sh` wrapper. Both compete for the same Claude Max rate limits.

We want to unify these into a **single priority-scheduled container pool** so that:
- Critical work (bugfixes, reviews) preempts lower-priority work (discovery, research)
- A fixed number of containers (e.g. 10) run continuously, pulling from a shared queue
- Token rotation and rate limit recovery happen at the pool level, not per-script
- Adding a new workload type (audit, spec generation, test) is just a new queue entry

## Decision

### Container types (priority order)

| Priority | Type | Source | Container image | Model |
|----------|------|--------|-----------------|-------|
| 1 | **bugfix** | Hydra: fix iteration after review failure | `hydra-builder` | opus |
| 2 | **code-review** | Hydra: PR code review | `hydra-reviewer` | sonnet |
| 3 | **security-review** | Hydra: PR security review | `hydra-security` | sonnet |
| 4 | **build** | Hydra: initial spec build | `hydra-builder` | opus |
| 5 | **audit** | Hydra: codebase audit | `hydra-builder` | sonnet |
| 6 | **spec-generation** | Specter: push_spec_pipeline | `specter-llm-worker` | sonnet |
| 7 | **schema-synthesis** | Specter: generate/dedup schemas | `specter-llm-worker` | haiku |
| 8 | **classification** | Specter: classify/redistribute features | `specter-llm-worker` | haiku |
| 9 | **translation** | Specter: translate requirements | `specter-llm-worker` | haiku |
| 10 | **discovery** | Specter: research, feature extraction | `specter-llm-worker` | haiku |

### Architecture

```
┌─────────────────────────────────────────────────────┐
│  Scheduler (cron or daemon)                         │
│                                                     │
│  reads: queue table (postgres)                      │
│  writes: container assignments, status updates      │
│                                                     │
│  ┌──────────────────────────────────────────┐       │
│  │ Pool: 10 container slots                 │       │
│  │                                          │       │
│  │  slot-1: [bugfix]     ← highest prio     │       │
│  │  slot-2: [code-review]                   │       │
│  │  slot-3: [build]                         │       │
│  │  slot-4: [build]                         │       │
│  │  slot-5: [classify]                      │       │
│  │  slot-6: [classify]                      │       │
│  │  slot-7: [translate]                     │       │
│  │  slot-8: [discovery]                     │       │
│  │  slot-9: [idle]       ← waiting for work │       │
│  │  slot-10: [idle]                         │       │
│  └──────────────────────────────────────────┘       │
│                                                     │
│  Token rotation: credentials.json (work → private)  │
│  Rate limit: pool-level tracking per account        │
│  Preemption: low-prio containers stopped when       │
│              high-prio work arrives and pool is full │
└─────────────────────────────────────────────────────┘
```

### Queue table (future)

```sql
CREATE TABLE container_queue (
    id SERIAL PRIMARY KEY,
    type VARCHAR(50) NOT NULL,        -- bugfix, code-review, build, classify, etc.
    priority INTEGER NOT NULL,         -- 1=highest
    payload JSONB NOT NULL,            -- script args, spec slug, issue URL, etc.
    status VARCHAR(20) DEFAULT 'pending', -- pending, running, completed, failed
    container_id VARCHAR(100),         -- docker container name when running
    token_account VARCHAR(50),         -- which OAuth account is assigned
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    exit_code INTEGER,
    error_message TEXT
);
```

### Phased rollout

**Phase 1 (now):** All LLM calls containerized. Specter scripts run via `run_llm_containers.sh`. Hydra containers use `run_container_with_fallback`. Both read from `credentials.json`. No shared queue yet — each system schedules its own containers.

**Phase 2:** Shared queue table. A single scheduler script replaces both `cron-hydra.sh` dispatch and `run_llm_containers.sh`. Pool size configurable. Priority enforcement by not starting low-prio work when high-prio is queued.

**Phase 3:** Preemption. Running low-priority containers can be stopped (gracefully, with checkpoint) when high-priority work arrives and all slots are occupied. Container images support checkpoint/resume via DB state.

### Current state (Phase 1)

Both systems already containerize LLM calls:
- **Hydra:** `builder`, `reviewer`, `security` images in `hydra/images/`
- **Specter:** `specter-llm-worker` image via `Dockerfile.llm-worker`
- **Shared credentials:** `hydra/secrets/credentials.json` with priority-ordered OAuth tokens
- **Token fallback:** Hydra via `credentials.sh`, Specter via `credentials.py`

## Consequences

- All LLM calls go through containers — no direct `claude -p` from host scripts
- Token management is centralized in `credentials.json`
- Future pool scheduler can enforce rate limits across both systems
- Container images are the unit of deployment — version, test, rollback independently
