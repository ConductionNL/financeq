# Tasks: Tax & Levy Management

**Change ID:** tax-levy-management  
**Last Updated:** 2026-05-21

## Phase 1: Data Model & Backend Infrastructure

### Task 1.1: Define OpenRegister Schemas
- [ ] Create `lib/Settings/tax-levy-management_register.json` with 11 schemas
  - [ ] ExemptionCertificate (schema:DigitalDocument)
  - [ ] TaxConfiguration (schema:Thing)
  - [ ] TaxDeclaration (schema:Report)
  - [ ] TaxExemption (schema:Offer)
  - [ ] TaxLot (schema:MonetaryAmount)
  - [ ] TaxRate (schema:Thing)
  - [ ] TaxReturn (schema:Thing)
  - [ ] TaxableTransaction (schema:Thing)
  - [ ] VATReturn (schema:Thing)
  - [ ] XBRLInstance (schema:DigitalDocument)
  - [ ] XBRLTaxonomy (schema:CreativeWork)
- [ ] Validate each schema against JSON Schema draft 2020-12
- [ ] Add schema.org type annotations to all properties
- [ ] Define relations (many-to-one, one-to-many, many-to-many) using OpenRegister vocabulary
- [ ] Test schema rendering in OpenRegister UI

**Acceptance:** All schemas registered and visible in OpenRegister admin UI

---

### Task 1.2: Create PHP Service Layer
- [ ] Create `lib/Service/TaxCalculationService.php`
  - [ ] Method: `calculateVATByCategory()` — aggregate VAT by rate category
  - [ ] Method: `calculateNetVATLiability()` — compute net amount due/refundable
  - [ ] Method: `applyExemptions()` — filter transactions by exemption certificate
  - [ ] @spec openspec/changes/tax-levy-management/tasks.md#task-1.2
- [ ] Create `lib/Service/TaxDeclarationService.php`
  - [ ] Method: `createDeclaration()` — create new VAT/BCF/corporate return
  - [ ] Method: `aggregateLots()` — roll up transactions into declaration
  - [ ] Method: `validateForSubmission()` — pre-submission validation
  - [ ] @spec openspec/changes/tax-levy-management/tasks.md#task-1.2
- [ ] Create `lib/Service/TaxReportingService.php`
  - [ ] Method: `generateVATReport()` — create VAT summary by period/category
  - [ ] Method: `generateIncomeStatement()` — P&L with quarterly breakdowns
  - [ ] Method: `generateJaaropgave()` — annual statement for employees (PDF)
  - [ ] @spec openspec/changes/tax-levy-management/tasks.md#task-1.2
- [ ] All services use DI (inject ObjectService, AuditTrailService, etc.)
- [ ] All services are stateless (no instance variables)
- [ ] Add unit tests (≥3 methods per service, ≥80% coverage)

**Acceptance:** Services pass unit tests and are callable from Controller layer

---

### Task 1.3: Create PHP Controllers & API Routes
- [ ] Create `lib/Controller/TaxDeclarationController.php`
  - [ ] POST `/api/tax-declarations` — create new declaration
  - [ ] GET `/api/tax-declarations/{id}` — fetch declaration details
  - [ ] PUT `/api/tax-declarations/{id}` — update declaration (draft only)
  - [ ] POST `/api/tax-declarations/{id}/submit` — submit to authority
  - [ ] Pagination support: `_page`, `_limit` query params
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001
- [ ] Create `lib/Controller/TaxRateController.php`
  - [ ] GET `/api/tax-rates?jurisdiction=NL&year=2026` — list rates
  - [ ] POST `/api/tax-rates` — create new rate (admin only)
  - [ ] PUT `/api/tax-rates/{id}` — update rate
  - [ ] DELETE `/api/tax-rates/{id}` — retire old rate
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-007
- [ ] Create `lib/Controller/VATReturnController.php`
  - [ ] GET `/api/vat-returns?period=Q1&year=2026` — list VAT returns
  - [ ] POST `/api/vat-returns` — create VAT return
  - [ ] POST `/api/vat-returns/{id}/calculate` — aggregate VAT by category
  - [ ] POST `/api/vat-returns/{id}/approve` — change status to approved
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001
- [ ] Register routes in `appinfo/routes.php`
- [ ] All controllers are thin (<10 lines/method), delegate to services
- [ ] Error responses include HTTP status + `message` field (no stack traces)
- [ ] All endpoints require Nextcloud auth (use `IUserSession`)

**Acceptance:** All endpoints return correct HTTP status and JSON response format

---

### Task 1.4: Repair Step & Database Initialization
- [ ] Create `lib/Migration/TaxLevyManagementRepairStep.php` (IRepairStep)
  - [ ] Detect OpenRegister availability (check for OpenRegister app installed)
  - [ ] Register the 11 tax schemas via RegisterService
  - [ ] Create seed data (3–5 example tax rates, configurations per country)
  - [ ] Log progress to info logger
  - [ ] @spec openspec/changes/tax-levy-management/design.md (seed data)
- [ ] Add migration entry to `appinfo/info.xml` with version number
- [ ] Test repair step on fresh Nextcloud install
- [ ] Test repair step on upgrade from prior version (idempotency check)

**Acceptance:** Repair step executes without errors and schemas are registered

---

## Phase 2: XBRL Export & Belastingdienst Integration

### Task 2.1: XBRL Serialization Service
- [ ] Create `lib/Service/XBRLExportService.php`
  - [ ] Method: `exportDeclarationAsXBRL()` — convert TaxDeclaration to XBRL XML
  - [ ] Method: `mapToXBRLConcepts()` — translate tax data to NTA7 elements
  - [ ] Method: `generateXBRLInstance()` — serialize to XML instance document
  - [ ] Use proper XBRL namespace (`http://www.xbrl.nl/nl/taxonomy/nta7/2026`)
  - [ ] Include contexts (periods, entities) and dimensions (if applicable)
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-010
- [ ] Create `lib/Service/XBRLValidationService.php`
  - [ ] Method: `validateXBRL()` — check against NTA7 schema
  - [ ] Return validation status: valid, invalid, warned
  - [ ] List validation errors/warnings with line numbers
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-010
- [ ] Add unit tests for XBRL generation (sample VAT return → XBRL)

**Acceptance:** XBRL export generates valid NTA7 documents, validation detects schema violations

---

### Task 2.2: Belastingdienst API Integration
- [ ] Create `lib/Service/BelastingdienstService.php`
  - [ ] Method: `submitVATReturn()` — POST XBRL to Belastingdienst API
  - [ ] Method: `submitPayrollDeclaration()` — POST loonaangifte XML
  - [ ] Method: `retrieveSubmissionStatus()` — poll for acknowledgment
  - [ ] Use OAuth credentials from IAppConfig (store in secure config)
  - [ ] Implement retry logic: exponential backoff on temporary failures (≤3 retries)
  - [ ] Timeout: 30 seconds; fallback to manual submission if timeout
  - [ ] Log all API calls (without sensitive data like credentials)
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001, req-tax-008
- [ ] Create `lib/Job/BelastingdienstRetryJob.php` (background job)
  - [ ] Query failed submissions from last 24 hours
  - [ ] Retry each failed submission up to 3 times
  - [ ] Log results and notify administrator if all retries exhausted
- [ ] Add unit tests with mocked Belastingdienst API responses

**Acceptance:** API calls succeed for valid submissions; failed submissions are queued and retried

---

### Task 2.3: Digital Signature (MTD Compliance)
- [ ] Create `lib/Service/DigitalSignatureService.php`
  - [ ] Method: `signXBRL()` — digitally sign XBRL document (if MTD enabled)
  - [ ] Method: `verifySignature()` — validate signature on received documents
  - [ ] Use PKI certificate stored in IAppConfig (load from secure key storage)
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001 (MTD compliance)
- [ ] Store certificate paths in `IAppConfig` (flag as sensitive)
- [ ] Add error handling for missing certificates (graceful fallback)

**Acceptance:** Signed XBRL documents are generated when MTD is enabled; signatures are verifiable

---

## Phase 3: Frontend UI & User Workflows

### Task 3.1: Tax Dashboard Component
- [ ] Create `src/pages/Dashboard.vue`
  - [ ] Use CnDashboardPage with GridStack layout
  - [ ] KPI cards (top 4):
    - [ ] "Pending Returns" — count of VAT returns in draft/approved status
    - [ ] "Collected VAT (This Quarter)" — YTD VAT collected
    - [ ] "Tax Liability" — net VAT payable (current period)
    - [ ] "Submissions This Year" — count of submitted returns
  - [ ] Chart: VAT by jurisdiction (pie chart, ApexCharts)
  - [ ] Chart: VAT collected vs. paid over last 12 months (area chart)
  - [ ] Table: "Upcoming Deadlines" (next 3 filings, jurisdiction, period, due date)
  - [ ] All text translatable: use `t(appName, 'key')`
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001
- [ ] Create Pinia store: `src/store/modules/dashboardStore.js`
  - [ ] Fetch KPI data from `/api/tax-dashboard/kpis`
  - [ ] Use `createObjectStore('taxReturns', 'TaxReturn', 'tax-levy-management')`
  - [ ] Loading state with `NcLoadingIcon` while data fetches

**Acceptance:** Dashboard loads in <2s, displays correct KPI values, responsive at 320px+

---

### Task 3.2: VAT Return Preparation Flow
- [ ] Create `src/pages/VATReturns.vue` (list page)
  - [ ] Use CnIndexPage with `useListView(objectStore)`
  - [ ] Table columns: Period, Jurisdiction, Status, Collected VAT, Paid VAT, Net Amount, Actions
  - [ ] Filter bar: Jurisdiction (dropdown), Status (multi-select), Year (input)
  - [ ] Row click → detail page (`$router.push({ name: 'VATReturnDetail', params: { id } })`)
  - [ ] Action buttons: + Create new, Bulk export, Bulk submit
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001
- [ ] Create `src/pages/VATReturnDetail.vue` (detail page)
  - [ ] Two modes: Edit (CnFormDialog), View (CnDetailPage)
  - [ ] Header: Title "Q1 2026 VAT Return — NL", Status badge
  - [ ] Sections:
    - [ ] **Overview** (CnDetailCard): Collected VAT, Paid VAT, Net Amount, Jurisdiction, Period
    - [ ] **Tax Breakdown** (CnDataTable): By rate category (Standard 21%, Reduced 9%, Zero 0%, Reverse, Exempt)
    - [ ] **Transactions** (CnDataTable): 10 sample transactions contributing to this return
    - [ ] **Exemptions** (CnDetailCard): Applied exemption certificates (if any)
    - [ ] **Audit Trail** (CnObjectSidebar → CnAuditTrailTab): Full change history
    - [ ] **Files** (CnObjectSidebar → CnFilesTab): Downloaded XBRL, submission confirmations
  - [ ] Actions: Edit, Approve, Submit, Download XBRL, View Audit Trail
  - [ ] Submit button → POST `/api/tax-declarations/{id}/submit` → show submission status modal
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-001
- [ ] Create Pinia store: `src/store/modules/vatReturnStore.js`
  - [ ] Use `createObjectStore` with plugins: `auditTrailsPlugin`, `filesPlugin`
  - [ ] Actions: fetchReturn(), updateReturn(), submitReturn()

**Acceptance:** VAT return list displays correctly, detail view shows all sections, submit works

---

### Task 3.3: Tax Rate Management (Municipal)
- [ ] Create `src/pages/TaxRates.vue` (list page)
  - [ ] Use CnIndexPage with table: Jurisdiction, Tax Category, Current Rate, Proposed Rate, Status
  - [ ] Filter: Jurisdiction, Status (draft/adopted)
  - [ ] Action: + Propose New Rates → CnFormDialog
  - [ ] Row click → detail (show full proposal with council notes)
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-007
- [ ] Create `src/pages/TaxRateProposal.vue` (detail page)
  - [ ] Form: Fiscal year, Tax categories (table)
    - [ ] Each row: Category, Current Rate, Proposed Rate, Revenue Change %
  - [ ] Section: "Council Review" (when submitted)
    - [ ] Finance head recommendation (textarea + approve/reject buttons)
    - [ ] Council vote result (date, raadsbesluit number, adopted YES/NO)
  - [ ] Section: "Publication" (when adopted)
    - [ ] Effective date (usually Jan 1 of next year)
    - [ ] Publish button → locks rates and notifies all modules
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-007
- [ ] Create Pinia store: `src/store/modules/taxRateStore.js`

**Acceptance:** Rate proposals can be created, reviewed, voted, and published; UI reflects correct status

---

### Task 3.4: Employee Annual Statements (Jaaropgave)
- [ ] Create `src/pages/AnnualStatements.vue` (employee self-service)
  - [ ] List current year jaaropgave if generated (available after Feb 28)
  - [ ] Link to download as PDF
  - [ ] Show total income, withheld tax, employer details
  - [ ] Filter by year (dropdown)
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-005
- [ ] Create `lib/Service/JaaropgaveGenerationService.php`
  - [ ] Method: `generateJaaropgaveForEmployee()` — create PDF statement
  - [ ] Include: Employee name, BSN, employer name/KVK, total income, withheld tax, period
  - [ ] Use TCPDF or similar for PDF generation
  - [ ] Format matches Belastingdienst IB33 template
  - [ ] @spec openspec/changes/tax-levy-management/specs.md#req-tax-005
- [ ] Create background job: `lib/Job/GenerateJaaropgaveJob.php`
  - [ ] Runs annually on Feb 28, generates jaaropgaves for all employees
  - [ ] Stores PDFs in FileService, linked to employee records

**Acceptance:** Employees can download jaaropgave PDFs in correct format

---

### Task 3.5: Settings & Configuration Page
- [ ] Create `src/pages/Settings.vue`
  - [ ] Use CnSettingsSection pattern (ADR-004)
  - [ ] **First card:** CnVersionInfoCard (app version, Nextcloud version, OpenRegister status)
  - [ ] **Second card:** CnRegisterMapping (show if OpenRegister is configured)
  - [ ] **Section 1: Belastingdienst Integration**
    - [ ] OAuth credentials (securely stored in IAppConfig)
    - [ ] Enable/disable MTD compliance flag
    - [ ] Test connection button → calls BelastingdienstService test endpoint
  - [ ] **Section 2: XBRL Configuration**
    - [ ] Select active taxonomy: NTA7-2026, SBR-NT-2026, etc.
    - [ ] View registered taxonomies and versions
  - [ ] **Section 3: Audit & Retention**
    - [ ] Audit trail export format (CSV, JSON, PDF)
    - [ ] Retention period (default: 7 years per Dutch law)
    - [ ] Legal hold checkbox (prevents deletion)
  - [ ] All text translatable
  - [ ] Save button → POST `/api/settings`, shows success toast
  - [ ] @spec openspec/changes/tax-levy-management/design.md (integration)
- [ ] Create backend: `lib/Controller/SettingsController.php`
  - [ ] GET `/api/settings` — fetch app settings
  - [ ] POST `/api/settings` — update settings (admin only)

**Acceptance:** Settings page loads and saves correctly; Belastingdienst connection test succeeds

---

## Phase 4: Testing & Documentation

### Task 4.1: Unit Tests
- [ ] Services: TaxCalculationService, TaxDeclarationService, TaxReportingService
  - [ ] ≥3 test methods per service
  - [ ] Cover happy path + edge cases (negative VAT, rate changes, exemptions)
  - [ ] Mock ObjectService, AuditTrailService dependencies
  - [ ] Run: `composer check:strict` ✅
  - [ ] Coverage: ≥80%
- [ ] Controllers: TaxDeclarationController, TaxRateController, VATReturnController
  - [ ] Test GET, POST, PUT endpoints with valid/invalid input
  - [ ] Test pagination (_page, _limit)
  - [ ] Test error responses (400, 403, 404, 500)
  - [ ] Mock service layer

**Acceptance:** All unit tests pass, coverage ≥80%

---

### Task 4.2: Integration Tests (Newman/Postman)
- [ ] Create `tests/integration/tax-declarations.postman_collection.json`
  - [ ] Create VAT return (POST /api/tax-declarations)
  - [ ] Fetch VAT return (GET /api/tax-declarations/{id})
  - [ ] Update VAT return (PUT /api/tax-declarations/{id})
  - [ ] Approve VAT return (POST /api/tax-declarations/{id}/approve)
  - [ ] Submit VAT return (POST /api/tax-declarations/{id}/submit)
  - [ ] Verify correct HTTP status and response structure
- [ ] Create `tests/integration/vat-returns.postman_collection.json`
  - [ ] Create VAT return with aggregation
  - [ ] Calculate VAT by category
  - [ ] Export as XBRL
- [ ] Run via Newman (CI/CD integration)

**Acceptance:** All integration tests pass on fresh database

---

### Task 4.3: Browser Tests (Playwright)
- [ ] Create `tests/e2e/vat-return.spec.ts` (GIVEN/WHEN/THEN)
  - [ ] **Story 2 (REQ-TAX-001):** VAT Return Preparation
    - [ ] GIVEN user on Dashboard, WHEN clicks "Prepare VAT Return", THEN form opens
    - [ ] GIVEN period selected (Q1), WHEN clicks Calculate, THEN return shows collected/paid VAT
    - [ ] GIVEN return approved, WHEN clicks Submit, THEN status changes to "submitted"
  - [ ] **Story 10 (REQ-TAX-007):** Tax Rate Management
    - [ ] GIVEN admin on Settings, WHEN proposes new rate, THEN proposal saved
    - [ ] GIVEN council votes, WHEN publishes rates, THEN all modules use new rates
  - [ ] **Story 5 (REQ-TAX-005):** Annual Statements
    - [ ] GIVEN employee after Feb 28, WHEN navigates to Annual Statements, THEN jaaropgave available
    - [ ] GIVEN download PDF, WHEN opened, THEN contains required fields
- [ ] Run via `npm run test:e2e` in CI/CD

**Acceptance:** All browser tests pass; screenshots show correct UI

---

### Task 4.4: User Documentation
- [ ] Create `docs/en/vat-returns.md`
  - [ ] How to prepare a VAT return (step-by-step with screenshots)
  - [ ] How to manage exemptions
  - [ ] How to submit to tax authorities
  - [ ] Troubleshooting (common errors, contact support)
- [ ] Create `docs/en/tax-rates.md`
  - [ ] How to propose new tax rates (municipal admin)
  - [ ] How to approve and publish rates
  - [ ] Effective date management
- [ ] Create `docs/en/annual-statements.md` (employee)
  - [ ] How to download jaaropgave
  - [ ] What the statement includes
  - [ ] Using it for tax return filing
- [ ] Create `docs/nl/` translations (Dutch)
- [ ] Add screenshots from running app (Vue UI)
- [ ] Ensure all docs reference latest UI components

**Acceptance:** Docs are complete, accurate, and include screenshots

---

## Phase 5: Security & Compliance

### Task 5.1: Security Review
- [ ] Verify auth: All endpoints require Nextcloud auth (no public endpoints except health)
- [ ] Verify CSRF: Routes file has CSRF token protection
- [ ] Verify SQL injection: All DB queries use parameterized statements (handled by OpenRegister)
- [ ] Verify XSS: All user input is sanitized in templates (Vue auto-escaping)
- [ ] Verify IDOR: Object access checks via AuthorizationService (multi-tenant isolation)
- [ ] Test with hydra-gates: Run security gate checks before merge
  - [ ] hydra-gate-orphan-auth (unused auth methods)
  - [ ] hydra-gate-unsafe-auth-resolver (catch Throwable patterns)
  - [ ] hydra-gate-no-admin-idor (admin-only endpoints)

**Acceptance:** No security issues found; hydra-gates pass

---

### Task 5.2: Data Protection & Compliance
- [ ] Verify GDPR: Implement data subject access via AuditTrailService
  - [ ] Method: `inzageverzoek()` — export all personal data for a user
  - [ ] Method: `verwerkingsregister()` — processing record for regulatory compliance
- [ ] Verify audit trail: All tax transactions have immutable audit logs (AuditTrailService)
- [ ] Verify retention: Tax records retain for 7 years (ArchivalService legal hold)
- [ ] Verify encryption: Belastingdienst credentials stored in secure IAppConfig (encrypted at rest)
- [ ] Add @license + @copyright PHPDoc tags to all PHP files (hydra-gate-spdx)

**Acceptance:** Data protection measures in place; audit trail is tamper-proof

---

## Phase 6: Performance & Scale

### Task 6.1: Performance Optimization
- [ ] Index tax transactions by:
  - [ ] organization_id, tax_category, transaction_date
  - [ ] For quick aggregation (VAT by category, period)
- [ ] Cache tax rates (Redis):
  - [ ] Key: `tax_rate:{jurisdiction}:{rateType}:{year}`
  - [ ] TTL: 86400 (1 day); invalidate on rate create/update
- [ ] Cache VAT calculations (Redis):
  - [ ] Key: `vat_calc:{organization_id}:{period}:{jurisdiction}`
  - [ ] TTL: 3600 (1 hour); invalidate on transaction changes
- [ ] Batch processing:
  - [ ] XBRL export uses background job for >10K facts
  - [ ] Jaaropgave generation uses job queue (scheduled)
- [ ] Test query performance:
  - [ ] VAT aggregation: <2 seconds for 10K transactions
  - [ ] XBRL generation: <5 seconds for 50K facts
  - [ ] Tax rate lookup: <100ms (cached)

**Acceptance:** Performance tests pass; no N+1 queries in VAT aggregation

---

### Task 6.2: Load Testing
- [ ] Simulate 10 concurrent VAT return submissions
- [ ] Verify no timeout (30s max per API call)
- [ ] Verify Belastingdienst API calls use proper retry logic
- [ ] Verify background jobs process without blocking UI

**Acceptance:** Load test completes without errors; UI remains responsive

---

## Phase 7: Release & Deployment

### Task 7.1: App Store Submission Prep
- [ ] Update `appinfo/info.xml`:
  - [ ] Increment version to 1.0.0
  - [ ] Update app description
  - [ ] List maintainers and contributors
  - [ ] Set Nextcloud compatibility (e.g., Nextcloud 27+)
  - [ ] Declare OpenRegister dependency (required: yes)
- [ ] Create `CHANGELOG.md` (v1.0.0 initial release)
- [ ] License: AGPL-3.0 (or other open-source license)
- [ ] README.md with:
  - [ ] Feature overview
  - [ ] Installation steps
  - [ ] Configuration (Belastingdienst OAuth setup)
  - [ ] Supported jurisdictions & XBRL taxonomies
  - [ ] Known limitations

**Acceptance:** App ready for Nextcloud App Store submission

---

### Task 7.2: Documentation for Admins
- [ ] Create `docs/ADMIN.md`:
  - [ ] System requirements (Nextcloud 27+, OpenRegister enabled)
  - [ ] Configuration: Belastingdienst OAuth credentials, certificate paths
  - [ ] Backup & recovery (audit trails, tax records)
  - [ ] Troubleshooting (API errors, failed submissions, rate issues)
  - [ ] Multi-tenancy setup (organization isolation)
- [ ] Create `docs/OPERATOR.md`:
  - [ ] Daily operations: monitoring submissions, checking failures
  - [ ] Handling failed Belastingdienst submissions (manual retry)
  - [ ] Audit trail export for tax defense
  - [ ] Legal hold procedures (7-year retention)

**Acceptance:** Admin docs are complete and tested against real workflows

---

## Definition of Done

✅ All code passes `composer check:strict`  
✅ All unit tests pass with ≥80% coverage  
✅ All integration tests pass (Newman)  
✅ All browser tests pass (Playwright)  
✅ Security gates pass (hydra-gates)  
✅ GDPR/audit trail requirements met  
✅ Documentation complete (user + admin)  
✅ Performance tests show <2s VAT aggregation, <5s XBRL export  
✅ App info updated with version 1.0.0  
✅ Ready for Nextcloud App Store submission

---

## Notes

- **Multi-jurisdiction Support (Phase 1):** Focus on NL + UK for initial release; extend to DE, BE, etc. in future version
- **Exemption Logic (Phase 1):** Handle research, export, environmental certificates; other types deferred
- **XBRL Taxonomies (Phase 2):** NTA7 (Dutch GL/tax) required; SBR-NT multi-country support deferred
- **Belastingdienst Integration (Phase 2):** Mock for testing; production OAuth credentials in IAppConfig
- **Digital Signature (Phase 2):** MTD compliance deferred to v1.1 if certificate infrastructure not available
- **Employee Statements (Phase 3):** Jaaropgave generation via background job; PDF format matches Belastingdienst IB33 template

---

## Task Dependencies

```
1.1 (Schemas) → 1.2 (Services) → 1.3 (Controllers)
1.1 (Schemas) → 1.4 (Repair step)
1.2 (Services) → 2.1 (XBRL export)
1.2 (Services) → 2.2 (Belastingdienst API)
2.1 (XBRL export) → 2.2 (API integration)
2.2 (API) → 2.3 (Digital signature)
1.3 (Controllers) → 3.1 (Dashboard)
3.1 (Dashboard) → 3.2 (VAT returns)
3.1 (Dashboard) → 3.3 (Tax rates)
3.1 (Dashboard) → 3.4 (Jaaropgave)
3.1 (Dashboard) → 3.5 (Settings)
All phases → 4.1 (Unit tests)
1.3 + 3.* → 4.2 (Integration tests)
3.* → 4.3 (Browser tests)
4.3 → 4.4 (Documentation)
All → 5.1 (Security review)
All → 5.2 (Compliance)
All → 6.1 (Performance)
All → 6.2 (Load testing)
All → 7.1 (App store prep)
4.4 → 7.2 (Admin docs)
```

**Critical Path:** 1.1 → 1.2 → 1.3 → 3.1 → 3.2 → 4.3 → 4.4 → 7.1
