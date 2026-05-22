# Context Brief: Tax & Levy Management — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** tax-levy-management
**Platform:** Nextcloud + OpenRegister

## Placement & Information Architecture

**Placement type:** `TOP_MENU` — Top-level menu entry — this functionality earns its own item in the app's left-nav.

**Lives at:** Belastingen

**Rationale:** core tax workflow  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Features (46 total, sorted by market demand)

### VAT return preparation with MTD (Making Tax Digital) compliance for UK
**demand: 665** (221 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Approve or reject amendment with motivation
**demand: 651** (217 tender mentions) | Category: core
Clustered from 1 mentions: 1 user stories

### Stripe Tax integration for automatic tax calculation
**demand: 203** (62 tender mentions, 8% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### Income statement with quarterly breakdowns for tax planning
**demand: 200** (66 tender mentions, 1% competitor coverage) | Category: scheduling
Clustered from 1 mentions: 1 competitor features

### Tax report generation based on tagged transactions
**demand: 173** (26 tender mentions, 45% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### VAT report overview showing collected and paid VAT by period
**demand: 172** (54 tender mentions, 5% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### TDS (Tax Deducted at Source) tracking and reporting
**demand: 171** (49 tender mentions, 12% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Automated tax compliance capturing W-9/W-8 and validating TINs for 1099 reporting
**demand: 161** (51 tender mentions, 4% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Tax summary report for filing preparation by period
**demand: 158** (29 tender mentions, 34% competitor coverage) | Category: ai
Clustered from 2 mentions: 2 competitor features

### Tax summary report for income and expenses by tax category
**demand: 143** (28 tender mentions, 28% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Quarterly tax report for estimated tax payment preparation
**demand: 133** (26 tender mentions, 26% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Economic Nexus Tracking
**demand: 129** (23 tender mentions, 27% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Tax Audit Reports
**demand: 121** (30 tender mentions, 15% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Sales tax tracking with configurable rates
**demand: 115** (35 tender mentions, 5% competitor coverage) | Category: analytics
Clustered from 5 mentions: 5 competitor features

### CCH Integrator Tax Compliance
**demand: 111** (29 tender mentions, 10% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### GST/tax compliance
**demand: 111** (29 tender mentions, 10% competitor coverage) | Category: governance
Clustered from 2 mentions: 2 competitor features

### Cross-Border Tax Compliance
**demand: 111** (29 tender mentions, 10% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Multi-country Tax Compliance
**demand: 111** (29 tender mentions, 10% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Touchless Tax Compliance (CoCounsel)
**demand: 111** (29 tender mentions, 10% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Tax report generation with filing-ready summaries by period
**demand: 106** (26 tender mentions, 13% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Automated tax report generation for VAT/GST/Sales tax filing
**demand: 98** (26 tender mentions, 10% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Tax Tracking for Self-employed
**demand: 98** (23 tender mentions, 13% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Sales tax report summarizing collected and owed tax amounts
**demand: 98** (26 tender mentions, 9% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Agentic Tax Compliance (ALFA Framework)
**demand: 93** (29 tender mentions, 3% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Payroll processing with automatic tax calculation and compliance
**demand: 93** (29 tender mentions, 3% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Supplier collaboration workspace for joint forecasting, innovation, and issue resolution
**demand: 92** (30 tender mentions, 1% competitor coverage) | Category: core
Clustered from 1 mentions: 1 competitor features

### Tax-exempt transaction handling with proper audit trail
**demand: 92** (30 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Supplier tax compliance with automated W-9/W-8 collection and TIN validation
**demand: 89** (29 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Withholding tax tracking on vendor payments
**demand: 81** (23 tender mentions, 6% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Quarterly estimated tax payment tracking and reminders
**demand: 79** (23 tender mentions, 5% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Comprehensive tax template system supporting GST/VAT/sales tax globally
**demand: 77** (25 tender mentions, 1% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 competitor features

### Mileage tracking with distance calculation and tax deduction support
**demand: 77** (23 tender mentions, 4% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Sales tax/VAT tracking with automatic rate calculation
**demand: 75** (23 tender mentions, 3% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### AI auto-categorization of expenses with tax-efficient category suggestions
**demand: 35** (11 tender mentions, 1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### AI spend classification with custom taxonomies and business-owned rules
**demand: 20** (6 tender mentions, 1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### AI Tax Determination Engine
**demand: 10** (5% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### File VAT return
**demand: 9** (3 tender mentions) | Category: document-management
Clustered from 1 mentions: 1 user stories

### File wage tax return
**demand: 9** (3 tender mentions) | Category: document-management
Clustered from 1 mentions: 1 user stories

### Receive VAT filing reminder
**demand: 7** (2 tender mentions) | Category: scheduling
Clustered from 1 mentions: 1 user stories

### Sales tax summary and tax liability reports
**demand: 6** (3% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Year-end tax summary with categorized income and deductions
**demand: 4** (2% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Automated tax provision (current + deferred tax)
**demand: 3** (1 tender mentions) | Category: ai
Clustered from 1 mentions: 1 user stories

### Automate data transfer to eliminate manual entry and reduce errors. This connector maps taxes, accounts, paym
**demand: 2** (1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### ONESOURCE+ AI Tax Returns
**demand: 2** (1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Export data for tax advisor
**demand: unknown** | Category: document-management
Clustered from 1 mentions: 1 user stories

### File payroll taxes
**demand: unknown** | Category: document-management
Clustered from 1 mentions: 1 user stories

## User Stories (11 linked)

### Story 1: Fiscal partner allocation
**Priority:** nice-to-have
As a freelancer, I want to allocate certain items to my fiscal partner, so that we optimize our joint tax position

### Story 2: Multi-jurisdiction VAT
**Priority:** must-have
As a tax director, I want VAT returns generated per jurisdiction, so that all obligations are met

### Story 3: Separate chart of accounts per administration
**Priority:** must-have
As a business owner, I want each administration to have its own chart of accounts, VAT settings, and fiscal year.

### Story 4: Maintain VAT audit trail
**Priority:** must-have
As a tax advisor, I want a complete audit trail for all VAT transactions, so that I can defend the return if audited

### Story 5: View annual income statement (jaaropgave)
**Priority:** must
As an employee, I want to view and download my annual income statement (jaaropgave), so that I can complete my annual tax return (aangifte inkomstenbelasting) without requesting the document from HR.

**Acceptance Criteria:**
GIVEN it is after the jaaropgave generation date in February WHEN I navigate to 'Annual Statements' THEN the current year's jaaropgave is available for download.
GIVEN I download the jaaropgave WHEN I open it THEN it contains total fiscal income, total wage tax withheld, and employer details matching the format required by the Belastingdienst.

### Story 6: Prepare corporate tax return (VPB-aangifte)
**Priority:** must
As an accountant for a BV, I want to prepare the VPB (vennootschapsbelasting) return with fiscal adjustments, so that I can file accurately via SBR

**Acceptance Criteria:**
GIVEN the commercial annual accounts are finalized WHEN I prepare the VPB return THEN the system calculates taxable profit with fiscal adjustments AND generates an SBR/XBRL submission file

### Story 7: Assess objection and revise WOZ value if warranted
**Priority:** must
As a municipal tax administrator, I want to assess the grounds of an objection and revise the WOZ value if the objection is (partially) founded, so that the corrected value is used for all linked tax calculations.

**Acceptance Criteria:**
GIVEN an objection is in assessment WHEN I review the property and the grounds stated THEN I can record a revised WOZ value, the reason for the revision, and reference comparable sales used as evidence
GIVEN the WOZ value is revised WHEN I save the revision THEN all linked local taxes (OZB, water board levy if applicable) are automatically recalculated based on the revised value
GIVEN the objection is assessed as unfounded WHEN I record this conclusion THEN the original value is confirmed and the decision letter template is pre-filled with the original value and grounds for rejection

### Story 8: Prepare income tax return (IB-aangifte) for sole proprietor
**Priority:** must
As a freelancer (zzp'er/eenmanszaak), I want the system to calculate my taxable profit including zelfstandigenaftrek, startersaftrek, and MKB-winstvrijstelling, so that I can file my IB-aangifte accurately

**Acceptance Criteria:**
GIVEN my bookkeeping is complete for the fiscal year WHEN I generate the IB preparation report THEN it shows fiscal profit after deductions (zelfstandigenaftrek, startersaftrek, MKB-winstvrijstelling) AND generates SBR export data

### Story 9: Review tax rate proposal before council vote
**Priority:** must
As a city council member, I want to view a summary of proposed tax rates with revenue projections and a comparison to prior years, so that I can make an informed decision during the council meeting.

**Acceptance Criteria:**
GIVEN a rate proposal has been approved by the head of finance WHEN a council member opens the proposal summary THEN they see a table with all local taxes, current rates, proposed rates, % change, and projected annual revenue per category
GIVEN the council vote is recorded as adopted WHEN the compliance officer confirms adoption THEN the system locks the rates for the next fiscal year and records the raadsbesluit number and date

### Story 10: Publish adopted tax rates in the system
**Priority:** must
As a compliance officer, I want to mark tax rates as officially adopted after the council vote and publish them to all relevant modules (billing, reporting), so that the correct rates are applied from 1 January of the new fiscal year.

**Acceptance Criteria:**
GIVEN a raadsbesluit number and adoption date are recorded WHEN I publish the rates THEN all tax modules are updated and the effective date is set to 1 January of the new fiscal year
GIVEN the rates are published WHEN the tax administrator generates tax assessments THEN the system uses the published rates and not any draft values

### Story 11: Submit monthly wage declaration to Belastingdienst
**Priority:** must
As a Payroll Administrator, I want to generate and submit the monthly wage declaration (loonaangifte) to the Belastingdienst, so that the organization meets its statutory payroll tax filing obligations.

**Acceptance Criteria:**
GIVEN the payroll run is finalized WHEN I generate the loonaangifte THEN an XML file in the Belastingdienst-specified format is produced containing all employee wage and withholding data
GIVEN the file is submitted THEN the submission timestamp and a confirmation reference number are stored in the payroll run record

## Stakeholders (24 linked)

### CFO / Financial Director
Chief Financial Officer responsible for financial reporting, internal controls, SOX compliance (if applicable), and financial governance. Interfaces with audit committee and external auditors.
**Responsibilities:** Financial reporting and annual accounts, internal controls (SOX Section 302/404 certification), risk management, treasury, tax compliance, audit coordination, dividend proposals
**Pain points:** SOX compliance documentation burden, coordinating with external auditors, ensuring internal control effectiveness, managing financial reporting deadlines, audit committee preparation workload
**Goals:** Automated internal control documentation, streamlined audit processes, real-time financial governance dashboards, efficient committee reporting

### MT Member / Manager
Member of the management team responsible for a functional area (finance, operations, HR, IT, etc.). Participates in collective MT decision-making while managing own department.
**Responsibilities:** ["Participating in MT decision-making on strategic matters", "Translating MT decisions into departmental actions", "Preparing proposals and business cases for MT agenda", "Managing departmental budget and resources", "Escalating issues that exceed departmental authority", "Cross-functional coordination with other MT members"]
**Pain points:** ["Dual role tension: MT interest vs department interest", "Decisions revisited repeatedly without clear closure", "No single source of truth for what was decided", "Action items from meetings lost or not tracked", "Difficulty coordinating cross-departmental decisions"]
**Goals:** ["Structured agenda and decision log for MT meetings", "Clear action tracking with ownership and deadlines", "Efficient preparation workflow for meeting items", "Visibility into decisions affecting own department"]

### Board Treasurer
Manages association finances, prepares annual financial statements (jaarrekening), budget proposals, and presents financial reports to the ALV. Works with kascommissie for audit. Under WBTR, increased personal liability for financial mismanagement.
**Responsibilities:** ["Prepare annual financial statements (jaarrekening)", "Present financial report at ALV", "Prepare annual budget (begroting)", "Cooperate with kascommissie audit", "Manage daily finances and payments", "Request decharge (discharge) at ALV", "Ensure financial transparency to members"]
**Pain points:** ["Preparing financial documents for non-financial audience", "Coordinating with kascommissie on audit timeline", "Explaining budget variances at ALV", "Personal liability under WBTR for financial decisions", "Handover complexity when treasurer changes"]
**Goals:** ["Clear financial reporting to members", "Smooth audit process with kascommissie", "Approved budget and discharge at ALV", "Digital financial transparency"]

### External Accountant
Professional accountant engaged by larger associations or when statutes require it. Provides independent audit opinion on financial statements. May replace or supplement the kascommissie for larger organizations.
**Responsibilities:** ["Audit annual financial statements", "Provide accountant's report (accountantsverklaring)", "Advise on financial controls and compliance", "Report to board and/or ALV", "Verify compliance with legal requirements"]
**Pain points:** ["Incomplete or unstructured financial data from association", "Tight timeline between financial year end and ALV", "Association boards unfamiliar with audit requirements", "Accessing supporting documentation"]
**Goals:** ["Timely access to complete financial records", "Structured financial data for efficient audit", "Clear communication channel with board"]

### Association Administrator
Paid staff or volunteer who handles day-to-day association administration. In larger associations, this is a professional bureau that supports the board with member administration, event organization, and communication.
**Responsibilities:** ["Maintain member database and contact details", "Process membership applications and cancellations", "Prepare meeting logistics (room, tech, catering)", "Send communications on behalf of board", "Manage association website and digital channels", "Support treasurer with financial administration", "Organize events and activities"]
**Pain points:** ["Juggling multiple manual systems", "Member data scattered across spreadsheets/tools", "High administrative burden for ALV preparation", "No integrated system for meetings + members + finances"]
**Goals:** ["Single integrated platform for all association tasks", "Automated member communications", "Efficient ALV preparation workflow", "Self-service portal for members"]

### Tax Director
Head of corporate tax
**Responsibilities:** Transfer pricing, international tax, compliance across jurisdictions
**Pain points:** Multiple tax jurisdictions; complex transfer pricing
**Goals:** Multi-jurisdiction compliance; automated TP documentation

### Information Architect
Designs and maintains the metadata schema, taxonomy, and classification structures used across the DMS. Ensures interoperability with national standards such as MDTO and Gemeentelijke Model Architectuur (GEMMA).
**Responsibilities:** defining and maintaining metadata schemas, aligning with MDTO and TMLO standards, governing taxonomy changes, advising on interoperability with other systems, auditing metadata quality
**Pain points:** fragmented metadata standards across departments, no enforced mandatory fields at ingestion, schema drift between DMS and zaaksysteem, difficulty mapping to MDTO upon archiving
**Goals:** enforced metadata schema at document creation, automated MDTO mapping, consistent taxonomy across all integrated systems, measurable metadata quality dashboards

### Municipal Controller
Financial and operational controller responsible for the planning-and-control cycle, including P&C rapportages, jaarrekening inputs, and budget monitoring for the college van B&W. Owns the authoritative management reporting output.
**Responsibilities:** coordinating periodic P&C rapportages, monitoring budget versus realisation, producing bestuursrapportages, ensuring consistency between financial and operational reporting, liaising with accountants
**Pain points:** reconciling data from multiple financial systems, narrative writing being manual and time-consuming, last-minute changes invalidating already-approved figures, difficulty distributing live dashboards to councillors securely
**Goals:** automate recurring reporting cycles, provide the college with real-time budget dashboards, reduce the production lead time of the jaarverslag

### Alderman (Wethouder)
Elected portfolio holder in the college van B&W accountable to the gemeenteraad for a specific policy domain such as social affairs, spatial planning, or finance. Consumes high-level dashboards to steer and defend policy.
**Responsibilities:** steering departmental performance, defending budget and outcomes in raad sessions, directing policy priorities, signing off on bestuursrapportages
**Pain points:** reports arriving too late to act on, inability to drill down into aggregated figures during council debates, data presented in technical formats not suitable for political communication
**Goals:** access up-to-date portfolio dashboards on any device, receive automated alerts when KPIs deviate from targets, present credible data-backed narratives to the gemeenteraad

### Payroll Administrator
Responsible for accurate and timely payroll processing in compliance with the applicable CAO (e.g. CAO Gemeenten, CAO Rijk) and Dutch tax law. Manages salary mutations, produces loonstroken, and prepares jaaropgaven.
**Responsibilities:** processing monthly payroll mutations, applying CAO salary scales and periodic increases, producing legally required loonstroken and jaaropgaven, filing payroll tax returns (loonheffingen), reconciling payroll with financial administration
**Pain points:** manual re-entry of mutations from HR into payroll system, CAO changes requiring bulk salary recalculations, errors in loonstroken due to late mutation submissions, complex integration with financial ERP systems, audit trail gaps
**Goals:** automated mutation flow from HR events to payroll, reliable CAO scale management with scheduled updates, zero-error loonstroken, seamless financial system integration, compliant and timely Belastingdienst filings

## Data Model — Entities for This Spec (11)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### ExemptionCertificate (`schema:DigitalDocument`)
_Tax exemption credential (research, export, environmental, humanitarian). Stores certificate metadata, validity, and linked exemptions for workflow automation._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| certificateNumber | string | Yes | Official certificate ID from issuing authority |
| certificateType | enum | Yes | research, export, environmental, humanitarian, innovation, vat-reverse, other |
| issueDate | date | Yes | Certificate issuance date |
| expiryDate | date | No | Expiration date; null = perpetual |
| exemptionReason | string | Yes | Legal basis or reason code |
| documentURL | uri | No | Link to official document or scan |

**Relations:**
- → Organization (many-to-one)
- → TaxDeclaration (many-to-many)

### TaxConfiguration (`schema:Thing`)
_System-wide tax settings, rules, and thresholds for a specific jurisdiction and tax year_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| configId | string | Yes | Unique configuration identifier |
| taxYear | number | Yes | Tax year this configuration applies to |
| jurisdiction | string | Yes | Tax jurisdiction code (NL, UK, US, etc.) |
| effectiveDate | datetime | Yes | Date when this configuration becomes effective |
| description | string | No | Configuration description and compliance notes |

**Relations:**
- → Organization (many-to-one)
- → TaxRate (one-to-many)

### TaxDeclaration (`schema:Report`)
_Primary tax declaration submission (VAT, BCF, exemptions). Aggregates tax lots and manages workflow from draft to submission._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| declarationType | enum | Yes | BCF, VAT-NL, ICP, or other Dutch tax form type |
| taxYear | integer | Yes | Calendar or fiscal year (e.g. 2025) |
| declarationStatus | enum | Yes | draft, approved, submitted, acknowledged, rejected |
| totalTaxAmount | MonetaryAmount | Yes | Net tax liability or credit |
| submissionDate | date | No | Actual submission timestamp to authorities |
| businessTaxID | string | Yes | Taxpayer BSN/KVK or VAT ID |

**Relations:**
- → Organization (many-to-one)
- → TaxLot (one-to-many)
- → ExemptionCertificate (many-to-many)

### TaxExemption (`schema:Offer`)
_Reusable exemption rule or policy: qualifies transactions or amounts as exempt. Linked to certificates and applied during tax lot calculation._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| exemptionCode | string | Yes | Statutory code (e.g. 021 for research) |
| exemptionName | string | Yes | Display name (e.g. 'Research & Development Exemption') |
| applicableTaxTypes | array | Yes | List of tax categories this exemption applies to (VAT, profit, withholding, etc.) |
| effectiveFrom | date | Yes | Start of exemption period |
| effectiveUntil | date | No | End of exemption period; null = ongoing |

**Relations:**
- → Organization (many-to-one)
- → ExemptionCertificate (many-to-one)

### TaxLot (`schema:MonetaryAmount`)
_Individual tax line item: single transaction or aggregate category contributing to declaration. Tracks category, amount, and justification._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| lotNumber | string | Yes | Unique identifier within declaration (e.g. VAT-001) |
| taxCategory | string | Yes | VAT standard/reverse/zero rate, profit, withholding, excise, etc. |
| amount | decimal | Yes | Gross or net tax amount |
| currency | string | Yes | EUR or other currency code |
| transactionDate | date | Yes | Date of underlying transaction or period start |
| description | string | No | Narrative or reference (e.g. invoice number, period) |

**Relations:**
- → TaxDeclaration (many-to-one)
- → BankAccount (many-to-one)

### TaxRate (`schema:Thing`)
_Individual tax rate rules for income, sales, VAT, capital gains, or other tax types with effective date management_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| rateId | string | Yes | Unique rate identifier |
| rateType | string | Yes | Type of tax: income, sales, vat, capital_gains, tds, gst, or other |
| percentage | number | Yes | Tax rate as percentage |
| effectiveDate | datetime | Yes | Date when this rate becomes effective |
| expiryDate | datetime | No | Date when this rate expires or is superseded |

**Relations:**
- → TaxConfiguration (many-to-one)
- → Product (many-to-one)

### TaxReturn (`schema:Thing`)
_A formal tax return filing for income, VAT, or other tax obligations with workflow management and compliance tracking_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| returnId | string | Yes | Unique identifier for the tax return |
| filingPeriod | string | Yes | Period covered by this return (e.g., Q1 2026) |
| taxYear | number | Yes | Calendar year for tax reporting |
| totalIncome | number | No | Total income for the period |
| totalExpenses | number | No | Total deductible expenses |
| status | string | Yes | Current status: draft, submitted, approved, or rejected |
| filedDate | datetime | No | Date when the return was submitted |

**Relations:**
- → Organization (many-to-one)
- → TaxConfiguration (many-to-one)

### TaxableTransaction (`schema:Thing`)
_Business transaction classified and tracked for tax reporting, audit trail, and automated tax calculation with receipt scanning support_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| transactionId | string | Yes | Unique transaction identifier |
| amount | number | Yes | Transaction amount |
| transactionDate | datetime | Yes | Date of the transaction |
| taxCategory | string | Yes | Tax classification category for reporting |
| taxRate | number | No | Applied tax rate percentage |
| description | string | No | Transaction description for audit trail |

**Relations:**
- → TaxReturn (many-to-one)
- → Receipt (many-to-one)
- → Payment (many-to-one)

### VATReturn (`schema:Thing`)
_VAT-specific tax return showing collected VAT, paid VAT, and net amount due for MTD compliance and electronic filing_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| vatReturnId | string | Yes | Unique VAT return identifier |
| reportingPeriod | string | Yes | VAT reporting period: monthly, quarterly, or annually |
| collectedVAT | number | Yes | VAT collected from customers |
| paidVAT | number | Yes | VAT paid on business purchases and expenses |
| netAmount | number | Yes | Net VAT payable (positive) or refundable (negative) |
| status | string | Yes | Status: draft, submitted, approved, or rejected |
| submissionDate | datetime | No | Date when VAT return was submitted to authorities |

**Relations:**
- → Organization (many-to-one)
- → TaxReturn (many-to-one)

### XBRLInstance (`schema:DigitalDocument`)
_Structured XBRL instance document for taxonomies (NTA7, SBR-NT). Contains facts, contexts, and dimensions for standardized digital reporting to Dutch authorities._

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| taxonomyVersion | string | Yes | e.g. NTA7-2025, SBR-NT-2025 |
| instanceID | string | Yes | Unique document identifier |
| reportingPeriod | string | Yes | ISO date range (e.g. 2025-01-01/2025-12-31) |
| factCount | integer | No | Number of XBRL facts in instance |
| encodingFormat | enum | Yes | application/xbrl+xml or application/xbrl+json |
| validationStatus | enum | Yes | valid, invalid, warned, unvalidated |

**Relations:**
- → TaxDeclaration (many-to-one)

### XBRLTaxonomy (`schema:CreativeWork`)
_XBRL (eXtensible Business Reporting Language) taxonomy definitions for structured tax reporting, compliance, and regulatory filing_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| taxonomyId | string | Yes | Unique taxonomy identifier |
| version | string | Yes | Taxonomy version number |
| effectiveDate | datetime | Yes | Date when taxonomy becomes effective |
| namespace | string | Yes | XML namespace URI for the taxonomy |
| elements | array | No | List of XBRL element definitions and mappings |

**Relations:**
- → TaxReturn (one-to-many)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, AccountabilityReport, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AuditorStatement, AwardDecision, AwardNotice, BalanceSheet, BankAccount, Bid, BidEvaluation, BiddingRound, BlanketPurchaseOrder, Branch, Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CallOffOrder, CashAccount, CatalogItem, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, ConsolidatedReport, ConsolidationGroup, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, CurrencyBalance, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExpenditureEscalation, ExpenditureRequest, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FXExposure, FinancialDecision, FinancialReport, FiscalYear, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, FundingSource, GeneralLedgerAccount, GeneralLedgerEntry, GoodsReceipt, GovernmentEntity, Grant, GrantPortfolio, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, JournalEntry, LiquidityForecast, Location, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentBatch, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, PricingRule, ProcurementAuditLog, ProcurementCatalog, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, PurchaseRequisition, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RequestForQuotation, RevenueStream, RiskCriteria, Role, SavingsOpportunity, ScheduledPayment, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, StatementOfWork, SubmissionDossier, Subscription, SubsidyApplication, SubsidyScheme, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TreasuryTask, TrialBalance, User, UserPreference, VendorBill, WOZAssessment

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
