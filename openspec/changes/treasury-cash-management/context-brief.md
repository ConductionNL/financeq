# Context Brief: Treasury & Cash Management — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** treasury-cash-management
**Platform:** Nextcloud + OpenRegister

## Placement & Information Architecture

**Placement type:** `TOP_MENU` — Top-level menu entry — this functionality earns its own item in the app's left-nav.

**Lives at:** Treasury

**Rationale:** core treasury  
_Source: /tmp/ia-small5.md_

> **Implementation note for builders:** Respect the placement above. Do not promote this spec to a top-level menu item, sub-page, or new route unless the placement type explicitly says so. If the placement is `DETAIL_TAB`, `WIDGET`, `ACTION`, `SETTING`, or `INFRA`, the feature must NOT introduce a new entry in the app sidebar. When in doubt, ask before creating a new top-level surface.

## Features (27 total, sorted by market demand)

### Cash management with liquidity forecasting and planning
**demand: 1748** (582 tender mentions, 1% competitor coverage) | Category: scheduling
Clustered from 1 mentions: 1 competitor features

### Treasury and cash management integration for payment timing optimization
**demand: 1734** (576 tender mentions, 3% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### Cash account tracking for petty cash management
**demand: 1677** (549 tender mentions, 15% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Digital lockbox technology preventing bid viewing before submission deadline
**demand: 1052** (350 tender mentions, 1% competitor coverage) | Category: scheduling
Clustered from 1 mentions: 1 competitor features

### Export treasury compliance report for council review
**demand: 406** (134 tender mentions, 2% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 user stories

### Business overview dashboard with real-time P&L, expenses, and cash flow
**demand: 218** (70 tender mentions, 4% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### Square payment processing integration
**demand: 216** (64 tender mentions, 12% competitor coverage) | Category: integration
Clustered from 2 mentions: 2 competitor features

### Schedule payment date
**demand: 208** (66 tender mentions, 5% competitor coverage) | Category: scheduling
Clustered from 2 mentions: 2 user stories

### Schedule payments strategically
**demand: 208** (66 tender mentions, 5% competitor coverage) | Category: scheduling
Clustered from 1 mentions: 1 user stories

### Schedule and track loan repayment and interest payments
**demand: 201** (67 tender mentions) | Category: scheduling
Clustered from 1 mentions: 1 user stories

### Cash flow report tracking money in, money out, and net position
**demand: 165** (49 tender mentions, 9% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Analytics Plus: AI cash flow forecasting from real business data
**demand: 134** (44 tender mentions, 1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Real-time dashboard with revenue, expense, and cash flow widgets
**demand: 106** (32 tender mentions, 5% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Base currency and foreign currency balance tracking per account
**demand: 101** (33 tender mentions, 1% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Cash Flow Report
**demand: 98** (26 tender mentions, 9% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### Payment analytics with cash flow forecasting and payment timing optimization
**demand: 97** (31 tender mentions, 2% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Generate monthly liquidity report for treasury committee
**demand: 93** (31 tender mentions) | Category: analytics
Clustered from 1 mentions: 1 user stories

### Cash flow report showing operating, investing, and financing activities
**demand: 87** (27 tender mentions, 3% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Merchant Information feature with brand/logo resolution
**demand: 77** (25 tender mentions, 1% competitor coverage) | Category: core
Clustered from 1 mentions: 1 competitor features

### Payment batch creation with SEPA file generation
**demand: 50** (14 tender mentions, 4% competitor coverage) | Category: document-management
Clustered from 4 mentions: 4 competitor features

### Connect my Dutch bank account via PSD2 for automatic daily import
**demand: 25** (5 tender mentions, 5% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 user stories

### AI-driven cash flow forecasting with real-time accounting sync
**demand: 23** (7 tender mentions, 1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### AI short-term cash flow forecasting with weekly granularity
**demand: 19** (5 tender mentions, 2% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Automated settlement and payout to merchant bank account
**demand: 7** (1 tender mentions, 2% competitor coverage) | Category: ai
Clustered from 2 mentions: 2 competitor features

### AP automation with automated payment processing and reconciliation
**demand: 7** (1 tender mentions, 2% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Send payment reminder
**demand: 6** (2 tender mentions) | Category: scheduling
Clustered from 3 mentions: 3 user stories

### Remittance File Generation
**demand: 2** (1% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 competitor features

## User Stories (21 linked)

### Story 1: In-house bank for intercompany funding
**Priority:** should-have
As a treasurer, I want an in-house bank (IHB) module that acts as the internal bank for all group entities, so that intercompany loans, deposits, and FX transactions are settled internally, reducing external bank dependency and optimizing group-level interest.

**Acceptance Criteria:**
GIVEN entity A has surplus cash and entity B needs funding WHEN an intercompany loan is initiated via the IHB THEN the loan is recorded in both entities' books at arm's-length interest AND a loan amortization schedule is created AND interest accruals are posted automatically AND the IHB consolidation position nets to zero at group level

### Story 2: Schedule payments strategically
**Priority:** should-have
As a controller, I want to schedule payments to optimize cash flow, so that we maintain sufficient liquidity

### Story 3: Support multiple banks
**Priority:** should-have
As a business owner, I want to reconcile multiple bank accounts, so that all cash is accounted for

### Story 4: Offset recovery amount against future subsidy payment
**Priority:** should
As a financial administrator, I want to offset a recovery amount against a future payment to the same recipient, so that we can recover funds without a separate bank transfer when the recipient has another active grant.

**Acceptance Criteria:**
GIVEN a recovery case and an open future payment to the same recipient
WHEN I apply an offset
THEN the future payment is reduced by the recovery amount, both records are linked, and the offset is visible in both dossiers

### Story 5: Centralized payment factory for group entities
**Priority:** must-have
As a treasurer, I want a centralized payment factory that collects approved payment instructions from all group entities and executes them through a single bank connectivity layer (SWIFT/EBICS), so that we reduce bank fees, enforce payment controls, and achieve full visibility over outgoing cash flows.

**Acceptance Criteria:**
GIVEN approved invoices from multiple entities are queued WHEN the daily payment run executes THEN payments are batched per bank/currency AND SEPA PAIN.001 or SWIFT MT101 files are generated AND a payment confirmation (CAMT.054) is matched back to the original invoice AND the entity's books are updated automatically AND dual authorization is enforced for payments above threshold

### Story 6: Import bank statements
**Priority:** must-have
As a controller, I want to import bank statements from multiple banks, so that all cash is tracked

### Story 7: View consolidated cash position
**Priority:** must-have
As a CFO, I want to see the consolidated cash position across all bank accounts, so that I can manage liquidity

### Story 8: Multi-bank cash visibility
**Priority:** must-have
As a treasurer, I want real-time cash positions across all banks, so that I can manage liquidity

### Story 9: Cash flow forecasting
**Priority:** must-have
As a treasurer, I want AI-powered cash forecasting, so that I can plan liquidity accurately

### Story 10: Payment provider per administration
**Priority:** must-have
As a business owner, I want separate payment provider configuration (Mollie/bank) per administration, so each business uses its own bank account.

### Story 11: Record liquidity transfer between municipal entities
**Priority:** must
As a Municipal Treasurer, I want to register internal liquidity transfers between the municipality and its affiliated entities (gemeenschappelijke regelingen), so that intercompany positions are tracked accurately.

**Acceptance Criteria:**
GIVEN a liquidity transfer to a gemeenschappelijke regeling is executed WHEN the treasurer registers it THEN the system records the counterparty, amount, direction, and date and updates both entities' cash positions
GIVEN an intercompany transfer is registered WHEN the month-end intercompany reconciliation runs THEN the system highlights any transfers where the counterparty has not confirmed the matching entry
GIVEN the treasurer needs an overview of intercompany exposures WHEN they open the overview THEN the system shows net positions per entity with interest accrual for any overnight balances

### Story 12: Monitor schatkistbankieren compliance
**Priority:** must
As a Municipal Treasurer, I want to monitor compliance with the schatkistbankieren obligation in real time, so that the municipality does not hold excess liquidity in private banks beyond statutory limits.

**Acceptance Criteria:**
GIVEN the municipality is subject to schatkistbankieren WHEN the treasurer opens the compliance view THEN the system shows the current balance at the Rijkshoofdboekhouding and the permitted maximum at private banks
GIVEN the private bank balance exceeds the allowed maximum WHEN the system detects it THEN it immediately alerts the treasurer with the excess amount and recommends a transfer to the schatkist
GIVEN the treasurer initiates a transfer to the schatkist WHEN it is registered THEN the system updates the compliance position and records the transaction for quarterly reporting to the Ministry

### Story 13: Register short-term cash investment (deposito)
**Priority:** must
As a Municipal Treasurer, I want to register a short-term deposito with a bank, so that surplus cash earns interest and the investment is tracked against the schatkistbankieren rules.

**Acceptance Criteria:**
GIVEN the treasurer has surplus cash WHEN they register a deposito THEN the system records the bank, amount, start date, maturity date, agreed interest rate, and reduces the available liquidity position by the deposited amount
GIVEN a deposito is registered WHEN its maturity date falls within the 13-week forecast window THEN the system includes the principal repayment and interest as an inflow in the forecast
GIVEN schatkistbankieren limits are configured WHEN a deposito would exceed the allowed external placement threshold THEN the system warns the treasurer before saving

### Story 14: View consolidated daily cash position
**Priority:** must
As a Municipal Treasurer, I want to view the consolidated cash position across all municipal bank accounts at the start of each day, so that I can make informed decisions about short-term investments and borrowing.

**Acceptance Criteria:**
GIVEN it is the start of the banking day WHEN the treasurer opens the liquidity dashboard THEN the system displays the opening balance per bank account and the consolidated total in euros
GIVEN bank statements are automatically imported via CAMT.053 WHEN a new statement arrives THEN the system updates the cash position in real time and flags material movements above a configurable threshold
GIVEN the treasurer selects a specific bank account WHEN they drill in THEN the system shows all transactions from the previous banking day with value dates and contra-party information

### Story 15: Generate monthly liquidity report for treasury committee
**Priority:** must
As a Municipal Treasurer, I want to generate a monthly treasury report, so that I can inform the financial controller and head of finance about liquidity positions, investment returns, and financing activity.

**Acceptance Criteria:**
GIVEN the treasurer requests a monthly treasury report WHEN it is generated THEN the system produces a report showing average cash position, interest earned on investments, financing costs, and schatkistbankieren compliance status
GIVEN the report is generated WHEN the treasurer reviews it THEN the system includes a comparison to the liquidity forecast that was made at the start of the month
GIVEN the report is finalised WHEN it is exported to PDF THEN the format follows the municipality's standard treasury reporting template for presentation to the treasury committee

### Story 16: Verify counterparty credit ratings against statute limits
**Priority:** must
As a municipal treasurer, I want to verify that all bank and investment counterparties meet the minimum credit rating requirements set in the treasury statute, so that I can prevent exposure to counterparties that no longer qualify.

**Acceptance Criteria:**
GIVEN counterparties are registered with their ratings in the system WHEN I run the credit rating compliance check THEN the system flags any counterparty whose current rating falls below the statutory minimum
GIVEN a counterparty is flagged as non-compliant WHEN I view its detail THEN I see the current rating, required minimum, rating agency, and date of last rating update

### Story 17: Monitor liquid reserve requirements
**Priority:** must
As a municipal treasurer, I want to monitor the municipality's liquid reserve levels against the minimum required by the treasury statute, so that I can ensure sufficient liquidity at all times.

**Acceptance Criteria:**
GIVEN daily cash position data is imported WHEN I view the liquidity compliance screen THEN the system shows actual liquid reserves versus the statutory minimum for each day in the reporting period
GIVEN a day where reserves fell below the minimum WHEN I select that day THEN the system shows the cash inflows and outflows that caused the shortfall

### Story 18: Export approved payroll as SEPA payment file
**Priority:** must
As a Payroll Administrator, I want to export the approved payroll as a SEPA XML payment file, so that salaries can be transferred via the organization's bank in the correct format.

**Acceptance Criteria:**
GIVEN the payroll is approved by the Financial Controller WHEN I export the payment file THEN a valid SEPA Credit Transfer (pain.001) XML file is generated containing all net salary payments
GIVEN the file is generated WHEN I download it THEN the export is logged with a timestamp and the payroll status is updated to 'Exported'

### Story 19: Export final payments for treasury system
**Priority:** must
As a financial administrator, I want to export confirmed final payment orders in a format compatible with the municipal treasury system (e.g. SEPA XML or Coda), so that payments can be executed without re-keying data.

**Acceptance Criteria:**
GIVEN one or more confirmed final payment orders
WHEN I request a payment export
THEN a SEPA Credit Transfer XML file is generated containing all selected orders, ready for upload to the treasury system

### Story 20: Generate SEPA pain.001 credit transfer files
**Priority:** critical
As a treasury manager, I want to export approved payment batches as ISO 20022 pain.001.001.09 XML files so that I can upload them to my bank portal for execution without re-keying payment details.

## Stakeholders (18 linked)

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

### Audit Committee Member
Member of the kascommissie (audit committee), legally required per BW 2:48. Minimum 2 members who are NOT board members. Examines financial statements and reports to ALV. Has right to inspect all books, documents, and cash.
**Responsibilities:** ["Examine annual financial statements (jaarrekening)", "Verify cash holdings and bank statements", "Inspect books, documents and records", "Report findings to ALV with recommendation for decharge", "Advise ALV on financial soundness", "Flag irregularities or concerns"]
**Pain points:** ["Limited financial expertise (volunteer role)", "Getting timely access to financial records", "Unclear what to check and how deep to audit", "No standardized audit checklist or process", "Time pressure before ALV"]
**Goals:** ["Clear audit process and checklist", "Easy access to all financial documents", "Ability to provide well-founded decharge recommendation", "Transparent reporting to members"]

### Association Administrator
Paid staff or volunteer who handles day-to-day association administration. In larger associations, this is a professional bureau that supports the board with member administration, event organization, and communication.
**Responsibilities:** ["Maintain member database and contact details", "Process membership applications and cancellations", "Prepare meeting logistics (room, tech, catering)", "Send communications on behalf of board", "Manage association website and digital channels", "Support treasurer with financial administration", "Organize events and activities"]
**Pain points:** ["Juggling multiple manual systems", "Member data scattered across spreadsheets/tools", "High administrative burden for ALV preparation", "No integrated system for meetings + members + finances"]
**Goals:** ["Single integrated platform for all association tasks", "Automated member communications", "Efficient ALV preparation workflow", "Self-service portal for members"]

### Treasurer
Corporate treasurer managing cash and treasury
**Responsibilities:** Cash management, FX hedging, bank relations, liquidity planning
**Pain points:** Manual cash position tracking across banks
**Goals:** Real-time cash visibility across all entities

### Alderman (Wethouder)
Elected portfolio holder in the college van B&W accountable to the gemeenteraad for a specific policy domain such as social affairs, spatial planning, or finance. Consumes high-level dashboards to steer and defend policy.
**Responsibilities:** steering departmental performance, defending budget and outcomes in raad sessions, directing policy priorities, signing off on bestuursrapportages
**Pain points:** reports arriving too late to act on, inability to drill down into aggregated figures during council debates, data presented in technical formats not suitable for political communication
**Goals:** access up-to-date portfolio dashboards on any device, receive automated alerts when KPIs deviate from targets, present credible data-backed narratives to the gemeenteraad

### Subsidy Policy Officer
Develops and maintains subsidy regulations, eligibility criteria, and scheme configurations on behalf of the municipality or province. Translates policy goals into operational subsidy schemes.
**Responsibilities:** designing subsidy schemes, setting eligibility rules and budget ceilings, defining application periods, drafting subsidy regulations, aligning schemes with municipal policy goals, coordinating with legal and finance departments
**Pain points:** translating complex legal frameworks into system configuration, keeping scheme definitions synchronized with policy changes mid-cycle, lack of version control for regulation changes, difficulty reusing scheme templates across programs
**Goals:** quickly configure compliant subsidy schemes, reuse and adapt existing scheme templates, track scheme performance against policy objectives, ensure legal correctness of eligibility criteria

### Head of Finance
Senior finance manager accountable for the statutory annual accounts and compliance with BBV and IV3 regulations. Coordinates with external auditors and the provincial supervisor.
**Responsibilities:** year-end close coordination, BBV-compliant annual accounts, IV3 submission to CBS, SiSa accountability reporting, liaison with external auditors
**Pain points:** manual IV3 mapping, tight statutory deadlines, audit queries requiring deep drill-down, reconciliation between subsystems, changing BBV guidelines
**Goals:** automated IV3 export, BBV compliance checks built into workflows, fast audit-trail drill-down, on-time submission without manual rework

### Management Accountant
Designs and maintains the internal cost allocation model, distributing overhead and shared service costs across programs and cost carriers. Provides insight into the true cost of municipal services.
**Responsibilities:** cost center maintenance, allocation key definition, overhead distribution runs, activity-based costing analysis, cost reporting to management
**Pain points:** complex multi-step allocation chains, manual recalculation when keys change, reconciling cost accounting with statutory reporting, explaining allocations to non-finance stakeholders
**Goals:** flexible allocation model, automated distribution runs, what-if scenario modelling, transparent cost breakdown per service

### Municipal Treasurer
Manages the municipality's treasury portfolio, cash position, and financing in strict compliance with the Wet Fido framework. Responsible for limiting financial risk from interest rate and credit exposure.
**Responsibilities:** cash flow forecasting, bank account management, short-term and long-term borrowing, investment of temporary surpluses, Wet Fido compliance reporting, renterisiconorm monitoring
**Pain points:** fragmented bank account data, manual cash flow aggregation, complexity of Wet Fido kasgeldlimiet and renterisiconorm calculations, limited scenario planning tools
**Goals:** integrated bank balance feed, automated Wet Fido compliance dashboard, rolling cash flow forecast, borrowing scenario modelling

## Data Model — Entities for This Spec (8)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### CashAccount (`schema:BankAccount`)
_Track bank accounts, petty cash, and cash equivalents for liquidity management and multi-account consolidation_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| accountType | string | Yes | BankAccount, PettyCash, or CashEquivalent |
| accountCode | string | Yes | Internal GL account code |
| riskLevel | string | No | Low, Medium, High |

**Relations:**
- → Organization (many-to-one)

### CurrencyBalance (`schema:Thing`)
_Multi-currency balance tracking per account for foreign currency management_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| balanceId | string | Yes | Unique balance record identifier |
| currency | string | Yes | Currency code (ISO 4217) |
| balance | number | Yes | Current balance amount |
| previousBalance | number | No | Previous balance for variance tracking |
| lastUpdated | datetime | Yes | Last update timestamp |

**Relations:**
- → BankAccount (many-to-one)

### FXExposure (`schema:MonetaryAmount`)
_Track foreign exchange risk across currencies with current rates, valuations, and unrealized gains/losses_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| baseCurrency | string | Yes | EUR or company base currency |
| foreignCurrency | string | Yes | ISO 4217 code |
| exposureAmount | number | Yes | Amount in foreign currency |
| currentExchangeRate | number | Yes | Foreign/base rate |
| valuationDate | string | Yes | ISO 8601 rate snapshot date |
| unrealizedGainLoss | number | No | P&L in base currency |
| riskLevel | string | No | Low, Medium, High |

**Relations:**
- → CashAccount (many-to-one)
- → Organization (many-to-one)

### LiquidityForecast (`schema:Report`)
_Daily/weekly/monthly cash flow projections for liquidity planning, including inflow/outflow/net position_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| period | string | Yes | Daily, Weekly, or Monthly |
| forecastDate | string | Yes | ISO 8601 generation date |
| projectionDays | integer | Yes | Days ahead to forecast |
| projectedInflow | number | Yes | Expected cash in |
| projectedOutflow | number | Yes | Expected cash out |
| netProjection | number | Yes | Inflow minus outflow |
| currency | string | Yes | ISO 4217 code |
| confidence | string | No | Low, Medium, High |

**Relations:**
- → CashAccount (many-to-one)
- → Organization (many-to-one)

### PaymentBatch (`schema:Payment`)
_Batch grouping of multiple payments for mass processing, approval, and scheduled execution_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| batchNumber | string | Yes | Unique batch identifier |
| totalAmount | number | Yes | Sum of all payments in batch |
| totalPayments | number | Yes | Count of payments in batch |
| status | string | Yes | Status: pending, processing, completed, failed |
| approvalStatus | string | Yes | Approval status: pending, approved, rejected |
| scheduledDate | datetime | No | Scheduled execution date for batch |
| createdDate | datetime | Yes | Date batch was created |

**Relations:**
- → Organization (many-to-one)
- → Payment (one-to-many)

### RequestForQuotation (`schema:Quotation`)
_Request for quotation supporting RFx management with templated events, multi-round negotiations, and digital lockbox_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| rfqNumber | string | Yes | Unique RFQ identifier |
| title | string | Yes | RFQ title or description |
| deadline | datetime | Yes | Submission deadline for responses |
| round | number | Yes | Negotiation round number |
| status | string | Yes | Status: draft, published, closed, awarded, cancelled |
| lockboxEnabled | boolean | Yes | Enable digital lockbox to prevent bid viewing before deadline |
| estimatedValue | number | No | Estimated procurement value |
| createdDate | datetime | Yes | RFQ creation date |

**Relations:**
- → Organization (many-to-one)
- → Payee (many-to-many)
- → Offer (one-to-many)

### ScheduledPayment (`schema:Payment`)
_Payment scheduled for future execution with support for recurring transactions_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| paymentReference | string | Yes | Unique payment reference or confirmation number |
| amount | number | Yes | Payment amount |
| currency | string | Yes | Currency code (ISO 4217) |
| scheduledDate | datetime | Yes | Date payment is scheduled for execution |
| frequency | string | No | Recurrence frequency: once, daily, weekly, monthly, yearly |
| recurringEndDate | datetime | No | End date for recurring payments |
| status | string | Yes | Status: pending, approved, executed, failed, cancelled |
| lastExecutionDate | datetime | No | Date of last payment execution |

**Relations:**
- → Payee (many-to-one)
- → BankAccount (many-to-one)
- → Payment (one-to-many)

### TreasuryTask (`schema:Event`)
_Unified AP/AR/spend task list for cash flow management with due dates and counterparty tracking_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| taskType | string | Yes | AccountsPayable, AccountsReceivable, or CapitalExpenditure |
| amount | number | Yes | Transaction amount |
| currency | string | Yes | ISO 4217 code |
| dueDate | string | Yes | ISO 8601 date |
| counterpartyName | string | No | Vendor, customer, or counterparty |
| description | string | No | Task details and notes |

**Relations:**
- → CashAccount (many-to-one)
- → Organization (many-to-one)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, AccountabilityReport, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AuditorStatement, AwardDecision, AwardNotice, BalanceSheet, BankAccount, Bid, BidEvaluation, BiddingRound, BlanketPurchaseOrder, Branch, Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CallOffOrder, CatalogItem, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, ConsolidatedReport, ConsolidationGroup, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExemptionCertificate, ExpenditureEscalation, ExpenditureRequest, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FinancialDecision, FinancialReport, FiscalYear, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, FundingSource, GeneralLedgerAccount, GeneralLedgerEntry, GoodsReceipt, GovernmentEntity, Grant, GrantPortfolio, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, JournalEntry, Location, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, PricingRule, ProcurementAuditLog, ProcurementCatalog, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, PurchaseRequisition, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RevenueStream, RiskCriteria, Role, SavingsOpportunity, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, StatementOfWork, SubmissionDossier, Subscription, SubsidyApplication, SubsidyScheme, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, TaxConfiguration, TaxDeclaration, TaxExemption, TaxLot, TaxRate, TaxReturn, TaxableTransaction, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TrialBalance, User, UserPreference, VATReturn, VendorBill, WOZAssessment, XBRLInstance, XBRLTaxonomy

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
