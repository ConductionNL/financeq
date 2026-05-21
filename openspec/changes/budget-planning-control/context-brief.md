# Context Brief: Budget Planning & Control — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** budget-planning-control
**Platform:** Nextcloud + OpenRegister

## Features (30 total, sorted by market demand)

### Multi-location budget management with site-specific allocations and tracking
**demand: 1720** (568 tender mentions, 8% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Direct material procurement with BOM management and forecast collaboration
**demand: 1700** (566 tender mentions, 1% competitor coverage) | Category: collaboration
Clustered from 1 mentions: 1 competitor features

### Public sector budgeting
**demand: 897** (297 tender mentions, 3% competitor coverage) | Category: participation
Clustered from 3 mentions: 3 competitor features

### Review fiscal impact of an amendment
**demand: 771** (257 tender mentions) | Category: core
Clustered from 1 mentions: 1 user stories

### Process budget amendment
**demand: 708** (236 tender mentions) | Category: core
Clustered from 1 mentions: 1 user stories

### Budget compliance analytics showing policy adherence across departments
**demand: 611** (203 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 competitor features

### Approval workflow with budget impact review at each decision point
**demand: 604** (200 tender mentions, 2% competitor coverage) | Category: core
Clustered from 2 mentions: 2 competitor features

### Record budget holder policy acknowledgement
**demand: 531** (177 tender mentions) | Category: governance
Clustered from 1 mentions: 1 user stories

### Department and project-level budget tracking with roll-up reporting
**demand: 506** (164 tender mentions, 7% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### Real-time budget monitoring with commitment tracking at purchase request stage
**demand: 428** (134 tender mentions, 13% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Department-level budget allocation with spend tracking and alert thresholds
**demand: 417** (137 tender mentions, 3% competitor coverage) | Category: analytics
Clustered from 2 mentions: 2 competitor features

### Flag lines as new policy initiatives
**demand: 393** (131 tender mentions) | Category: governance
Clustered from 1 mentions: 1 user stories

### Predictive analytics for sourcing timing, inventory planning, and budget forecasting
**demand: 331** (109 tender mentions, 2% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Multi-year planning
**demand: 279** (93 tender mentions) | Category: scheduling
Clustered from 6 mentions: 1 user stories, 5 competitor features

### Budget check integration blocking requisitions that exceed available budget
**demand: 245** (81 tender mentions, 1% competitor coverage) | Category: integration
Clustered from 1 mentions: 1 competitor features

### Report budget variances
**demand: 233** (45 tender mentions, 45% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 user stories

### Export budget proposal in council-ready format
**demand: 198** (66 tender mentions) | Category: document-management
Clustered from 1 mentions: 1 user stories

### Real-time budget consumption tracking with committed, actual, and forecasted spend views
**demand: 185** (55 tender mentions, 10% competitor coverage) | Category: analytics
Clustered from 10 mentions: 10 competitor features

### Project budget tracking with estimated vs actual hours and costs
**demand: 166** (50 tender mentions, 8% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### View budget commitment report by cost centre
**demand: 163** (45 tender mentions, 13% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 user stories

### Export contract budget consumption report
**demand: 161** (45 tender mentions, 12% competitor coverage) | Category: document-management
Clustered from 1 mentions: 1 user stories

### Budget tracking against purchase orders
**demand: 159** (43 tender mentions, 15% competitor coverage) | Category: analytics
Clustered from 5 mentions: 5 competitor features

### Budget creation with monthly allocation and variance tracking
**demand: 146** (42 tender mentions, 10% competitor coverage) | Category: analytics
Clustered from 3 mentions: 3 competitor features

### Commitment tracking showing encumbered funds from approved POs against budget
**demand: 138** (42 tender mentions, 6% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### Budget progress tracking with remaining amount and overspend alerts
**demand: 136** (42 tender mentions, 5% competitor coverage) | Category: analytics
Clustered from 1 mentions: 1 competitor features

### AI Budget Variance Intelligence
**demand: 78** (25 tender mentions, 1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Import previous year budget as baseline
**demand: 69** (23 tender mentions) | Category: document-management
Clustered from 1 mentions: 1 user stories

### Receive automated alerts on budget variance thresholds
**demand: 57** (19 tender mentions) | Category: ai
Clustered from 1 mentions: 1 user stories

### Spend forecasting using historical patterns and machine learning predictive models
**demand: 5** (1 tender mentions, 1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

### Debtor Agent - AI Payment Forecasting
**demand: 2** (1% competitor coverage) | Category: ai
Clustered from 1 mentions: 1 competitor features

## User Stories (49 linked)

### Story 1: Project budget forecast based on historical trend
**Priority:** should
As a policy officer, I want to see a projected cost forecast for the next 12 months based on current trend data, so that I can signal budget risks to the finance department in advance.

**Acceptance Criteria:**
GIVEN sufficient historical expenditure data is available
WHEN I request a forecast for the next four quarters
THEN the system generates a simple trend projection displayed as a line chart alongside actual historical spend, with the forecast confidence range clearly indicated

### Story 2: Set a cost forecast warning threshold
**Priority:** should
As a project-manager, I want to set a threshold for when EAC exceeds the approved budget by a certain percentage, so that I am automatically alerted before a formal budget breach occurs.

**Acceptance Criteria:**
GIVEN I configure a warning threshold of 5% overspend on a project WHEN the EAC reaches 105% of the approved budget THEN I receive a notification prompting me to take corrective action
GIVEN the threshold is exceeded WHEN the project appears in the portfolio overview THEN it shows an amber 'Forecast Warning' indicator

### Story 3: Forecast contract budget exhaustion date
**Priority:** should
As a financial controller, I want to see a projected budget exhaustion date based on current spend run rate, so that I can proactively plan for contract renewal or additional budget allocation.

**Acceptance Criteria:**
GIVEN a contract with at least 3 months of invoicing history WHEN I view the forecast panel THEN the system displays the projected exhaustion date based on average monthly spend
GIVEN the projected exhaustion date falls before the contract end date WHEN the contract manager views the contract THEN a warning indicator is shown prompting early renewal planning

### Story 4: View historical reallocation decisions for a budget line
**Priority:** should
As a manager-openbare-ruimte, I want to view all past reallocation decisions for a specific budget line including dates, amounts, and approvers, so that I can assess cumulative impact before approving a new request.

**Acceptance Criteria:**
GIVEN I am on a budget line detail page WHEN I open the Reallocation History tab THEN I see a chronological list of past reallocations with amount, date, approver, and reason
GIVEN more than ten reallocations exist WHEN I scroll THEN results are paginated and I can export the list to CSV

### Story 5: View reallocation history
**Priority:** should
As a project-manager, I want to view a chronological history of all approved budget reallocations on my project, so that I can explain budget movements during reviews and audits.

**Acceptance Criteria:**
GIVEN one or more reallocations have been approved on my project
WHEN I open the reallocation history tab
THEN I see a list ordered by date showing reference, source category, destination category, amount, approver, and approval date

### Story 6: View my approval history for a cost centre
**Priority:** should
As a budget holder, I want to view a history of all expenditures I have approved for a given cost centre and fiscal year, so that I can monitor spending trends and prepare for budget reviews.

**Acceptance Criteria:**
GIVEN I open the approval history for a cost centre WHEN I filter by fiscal year THEN I see a chronological list of approved expenditures with totals per cost category
GIVEN the approval history is displayed WHEN approved totals approach 80% of the available budget THEN a visual indicator signals the remaining headroom

### Story 7: View forecast cost to completion
**Priority:** should
As a financial controller, I want to see a forecast cost-to-completion figure based on current burn rate, so that I can signal potential overruns to the project manager before they materialise.

**Acceptance Criteria:**
GIVEN actual costs and a project end date are recorded
WHEN I open the financial forecast panel
THEN the system displays the projected total cost at completion, the expected variance against budget, and the assumptions used in the calculation

### Story 8: Create a new budget holder profile
**Priority:** should
As a compliance officer, I want to create a new budget holder profile by linking a named employee to one or more cost centres and assigning a mandate entry, so that the new budget holder can start approving expenditures from their first working day.

**Acceptance Criteria:**
GIVEN I open the budget holder onboarding form WHEN I save a profile with employee name, employee number, cost centres, mandate reference, and start date THEN the profile is created and the employee can log in and access their approval queue from the start date
GIVEN the profile is saved WHEN the associated mandate entry does not yet exist in the mandate register THEN the system blocks saving and prompts me to create the mandate entry first

### Story 9: Split contract budget ceiling across fiscal years
**Priority:** should
As a financial controller, I want to split a multi-year contract's budget ceiling across fiscal years, so that each year's spend is allocated to the correct annual budget and reported correctly to management.

**Acceptance Criteria:**
GIVEN a contract spans multiple fiscal years
WHEN I enter a per-year budget allocation
THEN the system stores the annual split and reports spend against each year's allocation separately

GIVEN we are in a new fiscal year
WHEN I view the contract budget
THEN the current year's allocation and its utilisation are displayed prominently, with previous years' actuals shown for reference

### Story 10: Compare planned versus actual training spend
**Priority:** should
As a Learning & Development Coordinator, I want to compare the approved training budget against actual expenditure throughout the year, so that I can manage spend and reallocate budget where needed.

**Acceptance Criteria:**
GIVEN I open the budget monitoring view WHEN I select a budget year THEN I see each budget line with approved amount, committed spend, actual invoiced amount, and remaining balance
GIVEN actual spend exceeds 90% of a budget line WHEN the threshold is crossed THEN the line is highlighted and I receive an email alert
GIVEN I want to reallocate budget WHEN I adjust a budget line THEN the change is logged with a timestamp, my user ID, and a mandatory reason field

### Story 11: Set and track budget for a participation process
**Priority:** should
As a municipal programme manager, I want to assign a budget to a participation process and track expenditure against it, so that I can ensure the process stays within its allocated resources.

**Acceptance Criteria:**
GIVEN I am reviewing a participation plan
WHEN I enter an approved budget amount
THEN the budget is recorded on the plan
GIVEN costs are logged against the process
WHEN I view the budget panel
THEN I see approved budget, spent amount, and remaining balance

### Story 12: Forecast budget at completion
**Priority:** should
As a financial-controller, I want to calculate and record a forecast-at-completion for each cost category based on current spend rate and remaining scope, so that I can proactively signal likely overruns to the project manager.

**Acceptance Criteria:**
GIVEN actual spend and remaining committed costs are recorded
WHEN I request a forecast-at-completion calculation
THEN the system calculates EAC using actual spend plus estimated remaining costs, and the result is stored with a note and submission date

### Story 13: Identify training gaps by department
**Priority:** should
As a Learning & Development Coordinator, I want to view which departments have the largest gap between training needs and current provision, so that I can prioritise budget allocation to underserved areas.

**Acceptance Criteria:**
GIVEN I open the gap analysis view WHEN I select the current plan year THEN I see a heatmap of departments against competency areas showing coverage percentage
GIVEN a department has less than 50% coverage for a competency area WHEN I hover or click the cell THEN I see the count of employees with that need versus those enrolled in a relevant programme
GIVEN I click 'create programme' from the gap view WHEN the form opens THEN the department and competency area are pre-filled

### Story 14: Track the history of cost forecast revisions
**Priority:** should
As a project-manager, I want to see a history of all ETC revisions with the date, author, and change note, so that I can explain forecast movements to the steering committee.

**Acceptance Criteria:**
GIVEN I have updated the ETC multiple times WHEN I open the forecast revision history THEN I see a chronological list of each revision with the old value, new value, author, date, and optional change note
GIVEN I click a revision entry WHEN the detail view opens THEN I see the full budget state as it was at that revision point

### Story 15: Submit a PGB expenditure declaration
**Priority:** should
As a citizen, I want to submit a PGB expenditure declaration by uploading an invoice and filling in the care provider details, so that the municipality can process reimbursement in time.

**Acceptance Criteria:**
GIVEN I have received an invoice from my informal care provider
WHEN I submit the declaration with the provider's name, bank account, period, and uploaded invoice
THEN a declaration record is created, linked to my PGB, and I receive a confirmation with a processing deadline

GIVEN the declaration amount exceeds the remaining budget
WHEN I submit
THEN the system warns me and blocks submission until the amount is corrected or a budget increase is requested

### Story 16: Request an increase to my PGB budget
**Priority:** should
As a citizen, I want to submit a request to increase my PGB allocation when my care needs have grown, so that I can continue receiving adequate support without interruption.

**Acceptance Criteria:**
GIVEN my current PGB is insufficient for my care needs
WHEN I submit a budget increase request with a description of the changed need and supporting documentation
THEN a new assessment application is created and linked to my existing PGB record, and the case manager is notified

### Story 17: Download annual PGB accountability statement
**Priority:** should
As a citizen, I want to download a pre-filled annual accountability statement (verantwoording) for my PGB, so that I can review, complete, and submit it to the municipality by the statutory deadline.

**Acceptance Criteria:**
GIVEN the verantwoording period has opened
WHEN I navigate to the PGB accountability section
THEN I can download a PDF pre-filled with my approved budget, recorded expenditures, and provider details, along with instructions and the submission deadline

### Story 18: Budget vs forecast comparison
**Priority:** must-have
As a board member, I want to compare budget, forecast, and actual, so that I understand financial performance

### Story 19: Review forecast cost against the approved budget
**Priority:** must
As a financial-controller, I want to compare the current EAC forecast against the approved budget baseline, so that I can identify projects at risk of a budget overrun.

**Acceptance Criteria:**
GIVEN I open the project financials screen WHEN the page loads THEN I see the approved budget baseline, current EAC, and the variance amount and percentage
GIVEN the EAC exceeds the approved budget WHEN I view the project list THEN the project is flagged with a 'Budget Risk' indicator

### Story 20: Aggregate cost forecasts across the portfolio
**Priority:** must
As a portfolio-manager, I want to see aggregated EAC and budget variance across all projects in my portfolio, so that I can manage total portfolio spend and contingency.

**Acceptance Criteria:**
GIVEN multiple projects exist in a portfolio WHEN I view the portfolio financial summary THEN I see total approved budget, total actuals, total ETC, total EAC, and total variance for all active projects
GIVEN a project's EAC is updated WHEN the portfolio summary is refreshed THEN the aggregate figures reflect the latest forecasts

## Stakeholders (74 linked)

### CEO / Director
Top executive responsible for overall organizational strategy, final decision authority on major matters, and accountability to the board of directors.
**Responsibilities:** ["Setting organizational strategy and vision", "Final authority on major investment and policy decisions", "Chairing management team meetings", "Reporting to board of directors / supervisory board", "Approving budgets above delegation thresholds", "Crisis decision-making and escalation endpoint"]
**Pain points:** ["Decisions bottleneck at the top due to unclear delegation", "Lack of visibility into decision status across layers", "Too many items escalated that should be handled lower", "Difficulty tracking whether MT decisions are actually implemented", "Information overload from multiple reporting channels"]
**Goals:** ["Clear delegation of authority matrix", "Real-time dashboard of organizational decision status", "Efficient MT meeting cycle with tracked outcomes", "Audit trail for governance and compliance"]

### MT Member / Manager
Member of the management team responsible for a functional area (finance, operations, HR, IT, etc.). Participates in collective MT decision-making while managing own department.
**Responsibilities:** ["Participating in MT decision-making on strategic matters", "Translating MT decisions into departmental actions", "Preparing proposals and business cases for MT agenda", "Managing departmental budget and resources", "Escalating issues that exceed departmental authority", "Cross-functional coordination with other MT members"]
**Pain points:** ["Dual role tension: MT interest vs department interest", "Decisions revisited repeatedly without clear closure", "No single source of truth for what was decided", "Action items from meetings lost or not tracked", "Difficulty coordinating cross-departmental decisions"]
**Goals:** ["Structured agenda and decision log for MT meetings", "Clear action tracking with ownership and deadlines", "Efficient preparation workflow for meeting items", "Visibility into decisions affecting own department"]

### Department Head
Leads a department within the organization. Translates management decisions into departmental plans, manages team leads, and handles operational decisions within delegated authority.
**Responsibilities:** ["Leading departmental meetings and decision-making", "Implementing MT decisions within the department", "Managing departmental budget within approved limits", "Approving procurement and hiring within delegation", "Escalating decisions beyond authority to MT member", "Coordinating with other departments on shared matters"]
**Pain points:** ["Unclear boundaries of decision authority", "Waiting for approvals from MT that delay operations", "No structured way to propose items to MT agenda", "Difficulty cascading decisions to teams consistently", "Cross-department dependencies causing bottlenecks"]
**Goals:** ["Clear delegation of authority documentation", "Streamlined approval workflows for routine decisions", "Efficient escalation path to management team", "Tool to cascade decisions to team leads and staff"]

### Project Manager
Manages projects that often span multiple departments. Requires decisions from steering committees, handles change requests, and manages go/no-go gates.
**Responsibilities:** ["Preparing steering committee meetings and agendas", "Managing project scope, budget, and timeline decisions", "Processing change requests through approval workflow", "Organizing go/no-go decision gates at project milestones", "Reporting project status to steering committee and sponsor", "Coordinating resources across departments"]
**Pain points:** ["Steering committee decisions delayed by member availability", "Change requests stuck in approval queues", "No audit trail of who decided what and when", "Difficulty getting cross-departmental commitment", "Go/no-go decisions postponed due to incomplete information"]
**Goals:** ["Structured steering committee meeting workflow", "Digital change request approval process", "Decision log with rationale and voting record", "Milestone-based go/no-go decision framework"]

### Controller / Financial Controller
Responsible for financial oversight, budget control, and financial approval in decision chains. Reviews business cases and validates financial impact of proposals.
**Responsibilities:** ["Reviewing and approving financial aspects of proposals", "Budget monitoring and variance reporting", "Financial validation in procurement approval chains", "Preparing financial reports for MT and board", "Ensuring compliance with financial policies and mandates", "Cost-benefit analysis for investment decisions"]
**Pain points:** ["Approval requests arriving without proper financial substantiation", "No visibility into committed vs actual spend across approvals", "Manual tracking of budget approvals across departments", "Difficulty enforcing financial policies consistently", "Last-minute approval requests bypassing normal process"]
**Goals:** ["Automated budget check in approval workflows", "Real-time budget commitment tracking", "Standardized business case template for proposals", "Financial approval audit trail for compliance"]

### Board Treasurer
Manages association finances, prepares annual financial statements (jaarrekening), budget proposals, and presents financial reports to the ALV. Works with kascommissie for audit. Under WBTR, increased personal liability for financial mismanagement.
**Responsibilities:** ["Prepare annual financial statements (jaarrekening)", "Present financial report at ALV", "Prepare annual budget (begroting)", "Cooperate with kascommissie audit", "Manage daily finances and payments", "Request decharge (discharge) at ALV", "Ensure financial transparency to members"]
**Pain points:** ["Preparing financial documents for non-financial audience", "Coordinating with kascommissie on audit timeline", "Explaining budget variances at ALV", "Personal liability under WBTR for financial decisions", "Handover complexity when treasurer changes"]
**Goals:** ["Clear financial reporting to members", "Smooth audit process with kascommissie", "Approved budget and discharge at ALV", "Digital financial transparency"]

### ICT Administrator / System Manager
Technical staff responsible for managing the raadsinformatiesysteem (RIS), bestuursinformatiesysteem (BIS), audiovisual systems, webcasting, and related infrastructure. Often also manages integrations with zaaksystemen, document management, and archiving systems.
**Responsibilities:** Managing and maintaining the RIS/BIS platform; Configuring integrations with other systems (zaaksysteem, DMS, archief); Managing user accounts and permissions; Ensuring system availability and performance; Managing AV equipment in council chamber and committee rooms; Supporting webcasting and live streaming; Ensuring WCAG accessibility compliance; Managing data migration during system transitions
**Pain points:** Legacy systems with poor APIs and integration capabilities; Multiple vendor dependencies; Complex AV infrastructure in council chambers; Tight SLA requirements during meeting nights; Limited budget for system upgrades; Vendor lock-in with proprietary RIS platforms
**Goals:** Maintain stable and performant systems; Enable seamless integrations; Minimize vendor lock-in; Ensure security and compliance; Support smooth meeting operations

### Accessibility Officer / Communications
Responsible for ensuring democratic information is accessible to all citizens, including those with disabilities. Manages the public-facing council website, ensures WCAG compliance, and coordinates communication about council activities.
**Responsibilities:** Ensuring WCAG 2.1 AA compliance of council publications; Managing public council website and RIS portal; Creating accessible versions of documents; Coordinating press releases about council decisions; Managing social media communication about meetings; Ensuring sign language interpretation when needed; Publishing meeting calendars and alerts
**Pain points:** PDF documents often not accessible; Video recordings lacking captions or transcripts; Complex RIS interfaces not meeting accessibility standards; Limited budget for accessibility improvements; Difficulty making historical archives accessible; Multiple platforms to manage consistently
**Goals:** Make all democratic information fully accessible; Reach all citizens regardless of ability; Ensure compliance with Digitoegankelijk requirements; Increase public engagement with council work

### Citizen / Resident
Any person living in the municipality who participates in democratic processes — voting, submitting ideas, attending hearings, joining assemblies, or challenging municipal tasks.
**Responsibilities:** Inform themselves about participation opportunities; submit views, ideas, or proposals; attend hearings and assemblies; vote in referenda and participatory budgets; hold government accountable.
**Pain points:** Difficulty finding participation opportunities; complex bureaucratic language; unclear impact of contributions; lack of feedback on submitted views; digital literacy barriers; time constraints for evening meetings; feeling unheard.
**Goals:** Have genuine influence on decisions affecting their neighbourhood and city; receive clear feedback on contributions; access participation processes digitally at convenient times; understand how their input shaped outcomes.

### Participation Coordinator
Municipal official responsible for designing, organizing, and managing participation processes — from simple consultations to complex citizens assemblies. Ensures legal compliance with the Gemeentewet and Omgevingswet participation requirements.
**Responsibilities:** Design participation trajectories; select appropriate methods (inspraak, burgerberaad, participatory budgeting); manage timelines and budgets; coordinate with policy departments; ensure inclusivity and representativeness; report results to council; comply with participatieverordening.
**Pain points:** Managing multiple participation tracks simultaneously; ensuring representative turnout; translating citizen input into actionable policy advice; coordinating across departments; meeting Omgevingswet participation requirements; limited tooling for hybrid (online+offline) processes.
**Goals:** Run efficient, inclusive participation processes; demonstrate measurable impact of participation; meet legal requirements; build trust between citizens and government; scale digital participation without losing quality.

## Data Model — Entities for This Spec (7)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### Budget (`custom`)
_A financial plan allocating resources for a specific period, organization, and location_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| budgetName | string | Yes | Name or identifier of the budget |
| totalAmount | number | Yes | Total budgeted amount in the specified currency |
| startDate | datetime | Yes | Date when the budget becomes effective |
| endDate | datetime | Yes | Date when the budget expires |
| description | string | No | Detailed description or purpose of the budget |
| currency | string | Yes | Currency code (ISO 4217), defaults to EUR for Dutch organizations |
| budgetCategory | string | Yes | Category of the budget (e.g., operational expenses, capital expenses, revenue) |
| amountSpent | number | No | Current amount spent or committed against this budget |
| alertThreshold | number | No | Percentage (0-100) at which to trigger spending alerts |
| budgetType | string | No | Type of budget (fixed, flexible, rolling, zero-based) |
| fiscalYear | integer | Yes | Fiscal year this budget applies to (e.g., 2026) |
| costCenter | string | No | Cost center or department code for budget allocation |
| attachments | array | No | Supporting documents and justification files |

**Relations:**
- → Organization (many-to-one)
- → Location (many-to-one)
- → Person (many-to-one)
- → BudgetPeriod (many-to-one)
- → BudgetAllocation (one-to-many)
- → BudgetAmendment (one-to-many)
- → ExpenditureRequest (one-to-many)

### BudgetAllocation (`custom`)
_A subdivision of budget resources allocated to a specific department, funding source, or purpose_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| allocationNumber | string | Yes | Unique identifier for the allocation |
| amount | number | Yes | Allocated amount |
| status | string | Yes | Status: pending, approved, allocated, spent, closed |
| description | string | No | Details about the allocation |

**Relations:**
- → Budget (many-to-one)
- → FundingSource (many-to-one)
- → Organization (many-to-one)

### BudgetAmendment (`custom`)
_A proposed or executed change to an approved budget amount_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| amendmentNumber | string | Yes | Unique identifier for the amendment |
| originalAmount | number | Yes | Original budgeted amount |
| newAmount | number | Yes | Revised budget amount |
| reason | string | Yes | Reason for the amendment |
| status | string | Yes | Status: proposed, pending_approval, approved, rejected, executed |
| effectiveDate | datetime | No | When amendment takes effect |

**Relations:**
- → Budget (many-to-one)
- → ApprovalRequest (many-to-one)

### BudgetPeriod (`custom`)
_A defined time period for budget planning, such as fiscal year, calendar year, or quarter_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the period (e.g., 'FY2024', 'Q1 2024') |
| type | string | Yes | Period type: fiscal_year, calendar_year, quarter, month, or custom |
| startDate | datetime | Yes | Period start date |
| endDate | datetime | Yes | Period end date |
| fiscalYear | string | No | Associated fiscal year (e.g., '2024') |

**Relations:**
- → Budget (one-to-many)

### ExpenditureRequest (`custom`)
_A request to spend funds from an allocated budget, requiring review and approval_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| requestNumber | string | Yes | Unique identifier for the request |
| amount | number | Yes | Requested expenditure amount |
| purpose | string | Yes | Purpose or description of the expenditure |
| status | string | Yes | Status: draft, submitted, approved, rejected, executed |
| requestDate | datetime | Yes | Date request was made |

**Relations:**
- → Budget (many-to-one)
- → ApprovalRequest (many-to-one)
- → Person (many-to-one)

### FundingSource (`custom`)
_A source of funds that can be allocated to budgets and expenditures_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the funding source |
| totalAmount | number | Yes | Total available funds |
| status | string | Yes | Status: active, inactive, depleted |
| description | string | No | Details about the funding source |

**Relations:**
- → BudgetAllocation (one-to-many)

### Location (`schema:Place`)
_A physical or geographic location for multi-site budget allocation and tracking_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Location name |
| code | string | No | Location code or identifier |
| address | string | No | Physical address |
| region | string | No | Geographic region |

**Relations:**
- → Organization (many-to-one)
- → Budget (one-to-many)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, AccountabilityReport, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AuditorStatement, AwardDecision, AwardNotice, BalanceSheet, BankAccount, Bid, BidEvaluation, BiddingRound, BlanketPurchaseOrder, Branch, CallOffOrder, CashAccount, CatalogItem, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, ConsolidatedReport, ConsolidationGroup, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, CurrencyBalance, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExemptionCertificate, ExpenditureEscalation, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FXExposure, FinancialDecision, FinancialReport, FiscalYear, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, GeneralLedgerAccount, GeneralLedgerEntry, GoodsReceipt, GovernmentEntity, Grant, GrantPortfolio, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, JournalEntry, LiquidityForecast, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentBatch, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, PricingRule, ProcurementAuditLog, ProcurementCatalog, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, PurchaseRequisition, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RequestForQuotation, RevenueStream, RiskCriteria, Role, SavingsOpportunity, ScheduledPayment, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, StatementOfWork, SubmissionDossier, Subscription, SubsidyApplication, SubsidyScheme, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, TaxConfiguration, TaxDeclaration, TaxExemption, TaxLot, TaxRate, TaxReturn, TaxableTransaction, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TreasuryTask, TrialBalance, User, UserPreference, VATReturn, VendorBill, WOZAssessment, XBRLInstance, XBRLTaxonomy

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
