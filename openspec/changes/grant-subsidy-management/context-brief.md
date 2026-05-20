# Context Brief: Grant & Subsidy Management — Shillinq

**App:** Shillinq — Complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations. Combines bookkeeping, invoicing, procurement, and contract management into one self-hosted solution on Nextcloud.Named after the shilling — one of the oldest and most widely used coins in European history, from the Roman solidus to the British shilling to the East African shilling still in use today.Shillinq covers:- Bookkeeping & general ledger (double-entry accounting)- Accounts payable & receivable- Sales invoicing & e-invoicing (UBL/Peppol)- Purchase orders & procurement workflows- Supplier management & approval chains- Contract lifecycle management (creation, renewal, obligations)- Bank reconciliation & payment matching- VAT/tax reporting & compliance- Financial statements (P&L, balance sheet, cash flow)- Budget planning & forecasting- Multi-currency support- Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)
**Spec:** grant-subsidy-management
**Platform:** Nextcloud + OpenRegister

## Features (23 total, sorted by market demand)

### Grant and fund management
**demand: 1675** (530 tender mentions, 41% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### Governmental Accounting Standards and Grant Management
**demand: 1593** (530 tender mentions, 1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 external mentions

### Submit a subsidy application with supporting documents
**demand: 1356** (452 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Revoke delegated access granted by offboarded user
**demand: 437** (143 tender mentions, 4% competitor coverage) | Category: security
Clustered from 1 mentions: 1 user stories

### Publish subsidy scheme to public portal
**demand: 354** (118 tender mentions) | Category: participation
Clustered from 1 mentions: 1 user stories

### View real-time subsidy portfolio compliance dashboard
**demand: 197** (63 tender mentions, 4% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 user stories

### Grant temporary access for substitute
**demand: 175** (57 tender mentions, 2% competitor coverage) | Category: security
Clustered from 1 mentions: 1 user stories

### Track Single Information Single Audit (SISA) grants
**demand: 96** (32 tender mentions) | Category: governance
Clustered from 1 mentions: 1 user stories

### Register and verify auditor statement for large subsidies
**demand: 57** (19 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Receive grant accountability
**demand: 38** (12 tender mentions, 1% competitor coverage) | Category: governance
Clustered from 1 mentions: 1 user stories

### Browse and filter available subsidy schemes
**demand: 12** (4 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Process grant application
**demand: 6** (2 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Track subsidy application status online
**demand: 3** (1 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Model Algemene Uitkering scenarios
**demand: 3** (1 tender mentions) | Category: other
Clustered from 1 mentions: 1 user stories

### Subsidieregistratie
**demand: 2** (1% competitor coverage) | Category: other
Clustered from 1 mentions: 1 competitor features

### efficient handling of grants
**demand: 1** | Category: other
Clustered from 1 mentions: 1 external mentions

### Monitor subsidy concentration risk per recipient
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Create and configure a new subsidy scheme
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Process subsidy reclaim and monitor repayment
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Identify SiSa grants
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Assess grant eligibility
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Monitor grant spending
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

### Recover misused grants
**demand: unknown** | Category: other
Clustered from 1 mentions: 1 user stories

## User Stories (22 linked)

### Story 1: Register subsidy-funded training (SLIM, ESF)
**Priority:** should
As a Learning & Development Coordinator, I want to tag training activities as subsidy-funded and record the subsidy source and amount, so that I can report correctly to subsidy providers such as SLIM or ESF.

**Acceptance Criteria:**
GIVEN I am adding or editing a training activity WHEN I toggle 'subsidy funded' THEN fields for subsidy programme, grant reference, approved amount, and conditions appear
GIVEN a subsidised training activity is completed WHEN I mark it as complete THEN the system automatically includes it in the subsidy reporting export
GIVEN the subsidy reporting period closes WHEN I generate the subsidy report THEN it shows all funded activities, participant counts, actual costs, and subsidy amount claimed

### Story 2: View risk-scored application list
**Priority:** should
As an integrity officer, I want to view a list of approved subsidy applications ranked by automated risk score, so that I can prioritise which applications to select for a spot check.

**Acceptance Criteria:**
GIVEN the integrity officer opens the spot-check dashboard WHEN the list loads THEN applications are sorted descending by risk score with score, scheme name, applicant name, and granted amount visible per row
GIVEN a risk score is displayed WHEN the officer hovers over the score THEN a tooltip explains which risk factors contributed to the score

### Story 3: View dashboard of outstanding advance payments
**Priority:** should
As a financial administrator, I want a dashboard showing all outstanding advance payments with their due accountability dates, so that I can proactively follow up before deadlines are missed.

**Acceptance Criteria:**
GIVEN multiple subsidy grants with advance payments issued
WHEN I open the advance payments dashboard
THEN I see a list sorted by accountability due date, with overdue items highlighted in red

### Story 4: Report on decision outcomes by scheme
**Priority:** should
As a subsidy team lead, I want to see grant, rejection, and withdrawal rates per scheme for a period, so that I can identify schemes that may need process adjustments or clearer eligibility criteria.

**Acceptance Criteria:**
GIVEN the team lead opens the outcomes report WHEN a scheme and period are selected THEN the report shows counts and percentages for granted, rejected, withdrawn, and still open applications
GIVEN the data is displayed WHEN the team lead clicks 'Export' THEN a CSV file is downloaded with one row per application

### Story 5: Monitor grant spending
**Priority:** must-have
As a subsidieverlener, I want to monitor how grantees spend their funding, so that public money is used as intended

### Story 6: Register and verify auditor statement for large subsidies
**Priority:** must
As a grant administrator, I want to register and verify whether a required auditor statement (accountantsverklaring) has been submitted for large subsidies, so that high-value awards are properly accounted for.

**Acceptance Criteria:**
GIVEN a subsidy exceeds the threshold requiring an auditor statement WHEN I open the accountability checklist THEN the system requires an auditor statement to be uploaded before the review can be completed
GIVEN an auditor statement is uploaded WHEN I mark it as verified THEN the statement date, auditor registration number, and opinion type are recorded on the award record

### Story 7: Block final payment if conditions are unmet
**Priority:** must
As a financial administrator, I want the system to prevent a final payment if mandatory conditions (such as approved accountability report or signed declaration) are not yet completed, so that we do not pay out without proper justification.

**Acceptance Criteria:**
GIVEN a subsidy grant where the accountability report is not yet approved
WHEN I attempt to process the final payment
THEN the system blocks the payment and shows a checklist of outstanding conditions

### Story 8: Close subsidy dossier after final payment
**Priority:** must
As a financial administrator, I want the subsidy dossier to be automatically closed and archived after the final payment is confirmed, so that the grant lifecycle is formally completed and the dossier is available for audit.

**Acceptance Criteria:**
GIVEN a final payment that has been confirmed by the bank
WHEN the payment status updates to 'executed'
THEN the subsidy dossier status changes to 'closed', a closure timestamp is recorded, and the dossier moves to the read-only archive

### Story 9: Identify overpayment after accountability review
**Priority:** must
As a financial administrator, I want the system to automatically flag a subsidy as overpaid when the approved final amount is lower than the total already disbursed, so that I can initiate recovery without manual calculations.

**Acceptance Criteria:**
GIVEN a subsidy grant where total disbursements exceed the approved final amount
WHEN the accountability review is finalised
THEN the system marks the grant as 'overpaid', calculates the recovery amount, and creates a recovery task

### Story 10: Create and configure a new subsidy scheme
**Priority:** must
As a grant administrator, I want to create a new subsidy scheme with all its rules and parameters, so that the scheme is consistently applied and automatically guides both applicants and assessors.

**Acceptance Criteria:**
GIVEN I start a new scheme WHEN I complete the configuration form THEN I can define scheme name, legal basis, total budget, maximum grant per applicant, target group, purpose, application period, and required documents
GIVEN the scheme is configured WHEN I save it as a draft THEN I can preview how it will appear on the public portal before publishing

### Story 11: Lock financial year after reconciliation sign-off
**Priority:** must
As a municipal controller, I want to lock the financial year in the subsidy system after I have signed off on the reconciliation, so that no retrospective changes can be made to the closed year's data.

**Acceptance Criteria:**
GIVEN a reconciliation that has been approved by the controller
WHEN I apply the year-end lock
THEN all grants and payments in the closed year become read-only, and any attempt to modify them produces a warning referring to the lock

### Story 12: Complete online accountability report form
**Priority:** must
As a subsidy applicant (organisation), I want to complete my accountability report online via a structured form, so that I can submit the required information without needing to understand complex templates or formats.

**Acceptance Criteria:**
GIVEN an active subsidy grant with an open accountability period
WHEN I log in and open the accountability task
THEN I see a step-by-step form pre-filled with my grant details, asking for actual costs, activities, and outcomes

### Story 13: Receive accountability deadline reminder
**Priority:** must
As a subsidy applicant (organisation), I want to receive an email reminder 30 and 7 days before my accountability deadline, so that I do not miss the submission date and risk losing part of my subsidy.

**Acceptance Criteria:**
GIVEN an active grant with an upcoming accountability deadline
WHEN 30 or 7 days remain before the deadline
THEN the system sends an email reminder to the registered contact person with the deadline date and a direct link to the accountability form

### Story 14: Check applicant against sanctions and debarment lists
**Priority:** must
As an integrity officer, I want to automatically check applicants against EU sanctions lists and Dutch debarment registers, so that we do not grant subsidies to sanctioned entities.

**Acceptance Criteria:**
GIVEN a new subsidy application
WHEN the application is submitted
THEN the system automatically checks the applicant name and KvK number against EU Sanctions, OFAC, and the Dutch Rijksoverheid exclusion register, flagging any matches for manual review

### Story 15: Open a formal fraud investigation case
**Priority:** must
As an integrity officer, I want to open a formal fraud investigation case linked to a specific subsidy grant or applicant, so that the investigation is tracked, documented, and kept separate from the regular dossier workflow.

**Acceptance Criteria:**
GIVEN a suspected fraud signal (internal report, data anomaly, or external tip)
WHEN I open a fraud investigation case
THEN a confidential case record is created with access restricted to the investigation team, linked to the relevant grant(s), and the grant is placed under a payment hold

### Story 16: Generate decision letter from template
**Priority:** must
As a subsidy case handler, I want to generate a formatted decision letter from a template pre-filled with application data, so that I do not have to manually enter applicant details and reduce the risk of errors.

**Acceptance Criteria:**
GIVEN a case handler selects a decision outcome for an application WHEN they click 'Generate letter' THEN a letter is produced with applicant name, address, scheme name, decision, amount (if granted), and legal basis pre-filled from the application record
GIVEN the generated letter is previewed WHEN the handler finds an error THEN they can edit the letter text before finalising

### Story 17: Identify open commitments at year-end
**Priority:** must
As a municipal controller, I want to see all open subsidy commitments (verplichtingen) at year-end that have not yet been paid, so that I can accurately report liabilities in the annual accounts (jaarrekening).

**Acceptance Criteria:**
GIVEN the year-end closing date
WHEN I run the open commitments report
THEN the system lists all approved grants where final payment has not been made, with the committed amounts and expected payment dates

### Story 18: Publish subsidy scheme to public portal
**Priority:** must
As a grant administrator, I want to publish an approved subsidy scheme to the citizen-facing portal, so that eligible applicants can find and apply for the scheme.

**Acceptance Criteria:**
GIVEN a scheme has been approved by the head of finance WHEN I publish it THEN it becomes visible on the public portal with all scheme information, eligibility requirements, required documents, and the application deadline
GIVEN the scheme is published WHEN the application period opens THEN the online application form becomes active and applicants can submit
GIVEN the application deadline passes WHEN the closing time is reached THEN the application form is automatically deactivated and late submissions are rejected with an explanation

### Story 19: Browse and filter available subsidy schemes
**Priority:** must
As a subsidy recipient, I want to browse all open subsidy schemes and filter by category and target group, so that I can quickly find schemes I may be eligible for.

**Acceptance Criteria:**
GIVEN multiple schemes are published WHEN I visit the subsidy portal THEN I can filter by category (e.g. sustainability, culture, sports), target group (individual citizen, association, SME), and application status
GIVEN I apply a filter WHEN results update THEN each scheme shows the scheme name, purpose, maximum grant, deadline, and a link to the full description and application form

### Story 20: Send accountability report request to recipient
**Priority:** must
As a grant administrator, I want to automatically send an accountability request to subsidy recipients when the project end date is reached, so that recipients are reminded to submit their accountability report on time.

**Acceptance Criteria:**
GIVEN a subsidy award is active and the project end date is reached WHEN the accountability deadline trigger fires THEN the system sends a formal accountability request to the recipient with the deadline and required documents
GIVEN an accountability request has been sent WHEN the deadline is 14 days away and no report has been received THEN the system sends a reminder to the recipient and notifies the grant administrator

## Stakeholders

(No stakeholders linked. Infer from the features and user stories above.)

## Data Model — Entities for This Spec (5)

These entities MUST be implemented as OpenRegister schemas.
OpenRegister provides: CRUD, REST API, search, import/export, audit trails, file attachments.
Do NOT rebuild these platform capabilities.

### AuditorStatement (`schema:Statement`)
_An auditor statement registering and verifying grant compliance and authenticity for large subsidies_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| statementId | string | Yes | Unique statement identifier |
| verificationDate | datetime | Yes | Date of auditor verification |
| isVerified | boolean | Yes |  |
| findings | string | No | Audit findings and observations |
| verdict | string | No | Audit verdict: approved, rejected, conditional |

**Relations:**
- → Grant (many-to-one)
- → Person (many-to-one)
- → DigitalDocument (one-to-one)

### Grant (`schema:Grant`)
_A financial grant or subsidy awarded to an organization for specified purposes under a subsidy scheme_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| grantId | string | Yes | Unique grant identifier |
| name | string | Yes | Grant name |
| awardedAmount | number | Yes | Amount awarded |
| awardDate | datetime | Yes | Date grant was awarded |
| status | string | Yes | Grant status: active, completed, suspended, revoked |
| accountingStandard | string | No | Governmental accounting standard applied |
| isSISAEligible | boolean | No | Eligible for Single Information Single Audit |

**Relations:**
- → SubsidyScheme (many-to-one)
- → Organization (many-to-one)
- → GrantPortfolio (many-to-one)

### GrantPortfolio (`schema:Collection`)
_A managed collection of grants for organizational tracking, compliance monitoring, and concentration risk analysis_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| portfolioId | string | Yes | Unique portfolio identifier |
| name | string | Yes | Portfolio name |
| description | string | No |  |
| totalGrantValue | number | No | Total value of all grants |
| complianceStatus | string | No | Compliance status: compliant, non-compliant, under-review |
| concentrationRiskLevel | string | No | Risk level: low, medium, high |
| lastAuditDate | datetime | No |  |

**Relations:**
- → Organization (many-to-one)
- → Grant (one-to-many)

### SubsidyApplication (`schema:Application`)
_An application for a subsidy or grant under a specific subsidy scheme with supporting documentation_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| applicationId | string | Yes | Unique application identifier |
| requestedAmount | number | Yes | Requested grant amount |
| status | string | Yes | Application status: draft, submitted, under-review, approved, rejected |
| submissionDate | datetime | No |  |
| reviewDate | datetime | No |  |
| notes | string | No |  |

**Relations:**
- → SubsidyScheme (many-to-one)
- → Organization (many-to-one)
- → Document (one-to-many)

### SubsidyScheme (`schema:GovernmentService`)
_A government subsidy program defining eligibility criteria, award conditions, and funding framework_

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| schemeId | string | Yes | Unique scheme identifier |
| name | string | Yes | Subsidy scheme name |
| description | string | No |  |
| maxGrant | number | No | Maximum grant amount |
| minGrant | number | No | Minimum grant amount |
| isPublished | boolean | No | Published to public portal |
| publishedDate | datetime | No |  |
| governmentLevel | string | No | national, provincial, or municipal |

**Relations:**
- → Organization (many-to-one)
- → Grant (one-to-many)

## Other App Entities (do NOT redefine, reference only)

APTransaction, Account, AccountabilityReport, Administration, AllocationRule, ApprovalChain, ApprovalRequest, ApprovalRoute, ApprovalTask, AssessmentCriteria, Assignment, Auction, AuditFinding, AwardDecision, AwardNotice, BalanceSheet, BankAccount, Bid, BidEvaluation, BiddingRound, BlanketPurchaseOrder, Branch, Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CallOffOrder, CashAccount, CatalogItem, ChargebackDispute, ComplianceAssessment, ComplianceAudit, ComplianceDocument, ComplianceReport, ComplianceRisk, ConsentRecord, ConsolidatedReport, ConsolidationGroup, Contract, ContractClause, ContractMilestone, ContractModification, ContractObligation, ContractParty, ContractPerformance, ContractRedline, ContractRenewal, ContractSpendRecord, ContractTemplate, Corporation, CostAllocation, CostCenter, CostProject, CreditNote, CurrencyBalance, DebitNote, Deduction, Delegation, DelegationRule, DepreciationSchedule, DigitalDocument, Dividend, Document, DunningNotice, Entitlement, Entity, EvaluationCriterion, Event, ExemptionCertificate, ExpenditureEscalation, ExpenditureRequest, Expense, ExpenseCategory, ExpenseClaim, ExpenseLineItem, ExpenseReport, FXExposure, FinancialDecision, FinancialReport, FiscalYear, FixedAsset, FrameworkAgreement, Freelancer, FundAllocation, FundingSource, GeneralLedgerAccount, GeneralLedgerEntry, GoodsReceipt, GovernmentEntity, IntercompanyTransaction, InventoryItem, InventoryStock, InventoryValuation, Investment, Invoice, InvoiceLine, JointVenture, JournalEntry, LiquidityForecast, Location, Lot, ManagementLetter, Mandate, MandateAuditLog, MandateRequest, MandateScheme, MandateViolation, MarketplaceApp, MarketplaceIntegration, MaverickSpendAlert, MonetaryAmount, OAuthIntegration, Obligation, ObligationSettlement, ObligationTask, Offer, Order, Organization, Payee, Payment, PaymentBatch, PaymentFraudAssessment, PaymentRiskScore, Payroll, PeppolAccessPoint, PeppolParticipant, PerDiem, PerformanceImprovementAction, PerformanceScore, Permission, Person, PolicyRule, PolicyViolation, PricingRule, ProcurementAuditLog, ProcurementCatalog, ProcurementCategory, ProcurementComplianceReport, ProcurementOrder, ProcurementProcedure, ProcurementQuote, Product, Project, ProjectTask, ProofOfDelivery, Property, PropertyAssessment, PublicProcurement, PublicationAmendment, PublicationLog, PublicationNotice, PurchaseOrder, PurchaseOrderChange, PurchaseOrderRevision, PurchaseRequisition, QualificationDeclaration, QualityManagementSystem, Quote, RateCard, Receipt, Report, RequestForQuotation, RevenueStream, RiskCriteria, Role, SavingsOpportunity, ScheduledPayment, ServiceLevelAgreement, SettlementDecision, Share, Shareholder, SigningAuthority, SourcingEvent, SpendCategory, SpendTransaction, SpendingRecord, StatementOfWork, SubmissionDossier, Subscription, Supplier, SupplierBid, SupplierCertificate, SupplierDocument, SupplierKPI, SupplierPerformanceReport, SupplierPerformanceScore, SupplierPerformanceScorecard, SupplierPortalAccount, SupplierPortalUser, SupplierQualification, SupplierRiskProfile, SupplierSLA, SupplierSurvey, SupplyChainRisk, TaxConfiguration, TaxDeclaration, TaxExemption, TaxLot, TaxRate, TaxReturn, TaxableTransaction, Team, Tender, TenderAmendment, TenderDocument, TenderLineItem, TenderLot, TenderNotice, TimeEntry, Timesheet, Transaction, TreasuryTask, TrialBalance, User, UserPreference, VATReturn, VendorBill, WOZAssessment, XBRLInstance, XBRLTaxonomy

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
