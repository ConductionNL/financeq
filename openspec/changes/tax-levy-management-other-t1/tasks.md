---
title: Tax & Levy Management — Tasks
app: shillinq
change: tax-levy-management-other-t1
---

# Tasks: Tax & Levy Management

## Deduplication Check

- [ ] **Task: Verify No Overlap with OpenRegister Services**
  - [ ] Search OpenRegister `lib/Service/` for ObjectService, SchemaService, ConfigurationService, ImportService, ExportService usage
  - [ ] Confirm ObjectService handles CRUD for TaxConfiguration, TaxRate, TaxDeclaration, TaxReturn, ExemptionCertificate
  - [ ] Confirm SchemaService imports tax schemas at app init
  - [ ] Confirm @conduction/nextcloud-vue provides CnIndexPage, CnFormDialog, CnDetailPage for UI (no custom form components)
  - [ ] Confirm AuditTrailService handles audit logging (no custom audit service needed)
  - [ ] Document findings in PR description

## Seed Data Preparation

- [ ] **Task: Create Seed Data for All Tax Schemas**
  - [ ] Generate 3-5 realistic TaxConfiguration objects (consultant, freelancer, retailer, non-profit)
  - [ ] Generate 3-5 realistic TaxRate objects (standard 21%, reduced 9%, zero VAT, exempt)
  - [ ] Generate 3-5 realistic TaxableTransaction objects (invoices, expenses, capital gains)
  - [ ] Generate 3-5 realistic TaxDeclaration objects (draft, submitted, accepted)
  - [ ] Generate 2-3 ExemptionCertificate objects (KOR, reverse charge)
  - [ ] Use Dutch street names, postcodes (`[1-9][0-9]{3}[A-Z]{2}`), valid KVK codes
  - [ ] Ensure cross-register consistency (TaxConfiguration → TaxableTransaction → TaxDeclaration)
  - [ ] Add seed data to design.md Seed Data section ✓ (already done)

## Register & Schema Setup

- [ ] **Task: Define Tax Schemas in `lib/Settings/shillinq_register.json`**
  - [ ] Create schema definitions:
    - [ ] `TaxConfiguration` (string: name, vatNumber, businessJurisdiction, fiscal year, default rates, flags)
    - [ ] `TaxRate` (number: rate, string: jurisdiction/itemType/appliedTo, datetime: validFrom/validUntil, boolean: reverseChargeApplies)
    - [ ] `TaxableTransaction` (string: transactionType/parentObjectId, number: amount/vatAmount/taxableAmount, string: taxRate/taxCategory, datetime: transactionDate)
    - [ ] `TaxDeclaration` (string: declarationType/status/filingReference, datetime: periodStart/periodEnd/submittedAt/submissionDeadline, number: grossSales/vat totals)
    - [ ] `TaxReturn` (number: taxYear, string: returnType/jurisdiction, number: income/deductions/tax, string: filingStatus/assessmentReference)
    - [ ] `ExemptionCertificate` (string: certificateType/exemptionCode, datetime: validFrom/validUntil, string: status)
  - [ ] Use schema.org vocabulary as primary (schema:Organization, schema:MonetaryAmount, etc.)
  - [ ] Include description field for every property
  - [ ] Mark required fields (name, amount, transactionDate, etc.)
  - [ ] Define types explicitly (string, number, boolean, datetime, array, object)
  - [ ] Validate schema.json structure with OpenRegister schema validator

- [ ] **Task: Register Template with Relations**
  - [ ] Define relations in register template:
    - [ ] TaxConfiguration → TaxRate (1:many) via register+schema+objectId
    - [ ] TaxConfiguration → TaxableTransaction (1:many) via foreignKey or relation
    - [ ] TaxableTransaction → TaxDeclaration (many:1) via declarationId
    - [ ] TaxableTransaction → TaxReturn (many:1) via returnId
    - [ ] ExemptionCertificate → TaxConfiguration (1:1) via configurationId
  - [ ] Ensure NO embedded objects; all relations use OpenRegister relation mechanism
  - [ ] Define access control (RBAC) per schema:
    - [ ] TaxConfiguration: viewable by Finance Manager+, editable by Admin only
    - [ ] TaxableTransaction: viewable by Finance Manager+, creatable by AutoProcess+Finance Manager
    - [ ] TaxDeclaration: viewable by Finance Manager+, fileable by Tax Officer (restricted role)
  - [ ] Include seed data in register template (3-5 objects per schema with @self envelope)

- [ ] **Task: Create Migration (Repair Step)**
  - [ ] Create `lib/Migration/{VersionNumber}_AddTaxSchemas.php` repair step
  - [ ] Implements `IRepairStep` interface
  - [ ] Calls `ConfigurationService::importFromApp('shillinq', registerTemplate, version, force: false)`
  - [ ] Idempotent: checks if schemas already exist; skips if present
  - [ ] Logs migration: "Tax schemas imported successfully"
  - [ ] Test: Run repair step twice; confirm no duplicates created

## Lifecycle & State Machine

- [ ] **Task: Define Tax Declaration Lifecycle (Declarative in Register)**
  - [ ] Add `x-openregister-lifecycle` to TaxDeclaration schema:
    ```json
    "x-openregister-lifecycle": {
      "states": [
        {"name": "draft", "label": "Draft", "color": "#FFF3CD"},
        {"name": "in-review", "label": "In Review", "color": "#D1ECF1"},
        {"name": "ready-to-file", "label": "Ready to File", "color": "#D4EDDA"},
        {"name": "submitted", "label": "Submitted", "color": "#CFE2FF"},
        {"name": "accepted", "label": "Accepted", "color": "#D1E7DD"},
        {"name": "rejected", "label": "Rejected", "color": "#F8D7DA"},
        {"name": "objection-filed", "label": "Objection Filed", "color": "#E2E3E5"},
        {"name": "resolved", "label": "Resolved", "color": "#D1E7DD"}
      ],
      "transitions": [
        {
          "from": "draft",
          "to": "in-review",
          "trigger": "user-review",
          "requires": null,
          "emitEvent": "declaration:review-started"
        },
        {
          "from": "in-review",
          "to": "ready-to-file",
          "trigger": "consistency-check-passed",
          "requires": "TaxDeclarationValidationGuard",
          "emitEvent": "declaration:ready-to-file"
        },
        {
          "from": "ready-to-file",
          "to": "submitted",
          "trigger": "user-file",
          "requires": null,
          "emitEvent": "declaration:submitted"
        },
        {
          "from": "submitted",
          "to": "accepted",
          "trigger": "filing:accepted-webhook",
          "requires": null,
          "emitEvent": "declaration:accepted"
        },
        {
          "from": "submitted",
          "to": "rejected",
          "trigger": "filing:rejected-webhook",
          "requires": null,
          "emitEvent": "declaration:rejected"
        },
        {
          "from": "accepted",
          "to": "objection-filed",
          "trigger": "user-file-objection",
          "requires": null,
          "emitEvent": "declaration:objection-filed"
        },
        {
          "from": "objection-filed",
          "to": "resolved",
          "trigger": "objection-response-webhook",
          "requires": null,
          "emitEvent": "declaration:resolved"
        }
      ]
    }
    ```
  - [ ] Document guards in `lib/Lifecycle/TaxDeclarationValidationGuard.php` (see Guard Implementation task below)

- [ ] **Task: Implement Lifecycle Guard: `TaxDeclarationValidationGuard`**
  - [ ] File: `lib/Lifecycle/TaxDeclarationValidationGuard.php`
  - [ ] Implements `LifecycleGuardInterface`
  - [ ] Method: `canTransition(string $fromState, string $toState, object $entity): bool`
  - [ ] Logic for `draft` → `in-review`: always true (user-initiated)
  - [ ] Logic for `in-review` → `ready-to-file`: run consistency checks:
    - [ ] No duplicate transactions
    - [ ] All transactions have tax category assigned
    - [ ] VAT calculations match source transactions
    - [ ] Reverse charge rules applied correctly
    - [ ] Return true if all checks pass; false otherwise
  - [ ] Method: `getReason(): string` — explain why transition was blocked (if applicable)
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#guard-implementation`
  - [ ] Unit test: `tests/Unit/Lifecycle/TaxDeclarationValidationGuardTest.php`

## Aggregations & Calculations

- [ ] **Task: Define Tax Aggregations (Declarative in Register)**
  - [ ] Add `x-openregister-aggregations` to TaxDeclaration schema:
    ```json
    "x-openregister-aggregations": [
      {
        "name": "totalVatCollected",
        "label": "Total VAT Collected",
        "description": "Sum of VAT on all sales invoices in period",
        "query": {
          "register": "shillinq",
          "schema": "TaxableTransaction",
          "filter": {
            "transactionType": "invoice",
            "transactionDate.gte": "@self.periodStart",
            "transactionDate.lte": "@self.periodEnd"
          },
          "group": "taxRate",
          "reduce": {"sum": "vatAmount"}
        }
      },
      {
        "name": "totalVatDeductible",
        "label": "Total VAT Deductible",
        "description": "Sum of VAT on all purchase invoices in period",
        "query": {
          "register": "shillinq",
          "schema": "TaxableTransaction",
          "filter": {
            "transactionType": "expense",
            "transactionDate.gte": "@self.periodStart",
            "transactionDate.lte": "@self.periodEnd"
          },
          "reduce": {"sum": "vatAmount"}
        }
      },
      {
        "name": "netVatOwed",
        "label": "Net VAT Owed",
        "description": "Difference between collected and deductible",
        "formula": "@self.totalVatCollected - @self.totalVatDeductible"
      }
    ]
    ```
  - [ ] Test: Create declaration; verify aggregations calculate correctly
  - [ ] Test: Add transaction to period; verify aggregation updates

- [ ] **Task: Define Tax Calculations (Declarative in Register)**
  - [ ] Add `x-openregister-calculations` to TaxableTransaction schema:
    ```json
    "x-openregister-calculations": [
      {
        "name": "isDeductible",
        "label": "Is Deductible",
        "type": "boolean",
        "formula": "?(@self.taxCategory in ['office-supplies', 'software', 'training', ...] or @self.transactionType == 'expense')"
      },
      {
        "name": "deductibleAmount",
        "label": "Deductible Amount",
        "type": "number",
        "formula": "?(@self.isDeductible ? @self.amount * @self.deductionPercentage : 0)"
      },
      {
        "name": "deductionPercentage",
        "label": "Deduction Percentage",
        "type": "number",
        "formula": "?(@self.taxCategory == 'meals-entertainment' ? 0.5 : 1.0)"
      }
    ]
    ```
  - [ ] Test: Create expense with meals-entertainment category; verify deductibleAmount = amount * 0.5

- [ ] **Task: Define Tax Notifications (Declarative in Register)**
  - [ ] Add `x-openregister-notifications` to TaxDeclaration schema:
    ```json
    "x-openregister-notifications": [
      {
        "event": "declaration:submitted",
        "trigger": "state-change",
        "recipient": {
          "role": "Finance Manager",
          "viaChannel": "nextcloud-notification"
        },
        "template": "Tax declaration {{name}} ({{declarationType}}) has been submitted. Reference: {{filingReference}}"
      },
      {
        "event": "declaration:deadline-approaching",
        "trigger": "schedule",
        "schedule": "-11 days",
        "recipient": {"role": "Tax Officer", "viaChannel": "email"},
        "template": "VAT filing deadline approaching on {{submissionDeadline}}. Prepare {{name}} now."
      },
      {
        "event": "declaration:accepted",
        "trigger": "webhook",
        "recipient": {"role": "Finance Manager", "viaChannel": "nextcloud-notification"},
        "template": "Tax declaration {{name}} accepted by {{jurisdiction}} authority!"
      }
    ]
    ```
  - [ ] Test: Trigger each notification type manually; verify content and recipient

## Integration Services

- [ ] **Task: Create Receipt OCR Adapter Service**
  - [ ] File: `lib/Service/OcrService.php`
  - [ ] Method: `extractReceiptData(File $receipt): OcrExtractionResult`
  - [ ] Calls Scan & Herken API (or Tesseract fallback)
  - [ ] Returns:
    ```php
    {
      "merchant": "string",
      "date": "DateTime",
      "items": [{"description": "string", "amount": float}],
      "subtotal": float,
      "vatAmount": float,
      "totalAmount": float,
      "vatRate": int,
      "confidence": float (0-1)
    }
    ```
  - [ ] Catches API errors; returns confidence = 0 if extraction fails
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#ocr-service`
  - [ ] Unit test: Mock Scan & Herken API; verify extraction result

- [ ] **Task: Create Tax Filing Integration Service**
  - [ ] File: `lib/Service/TaxFilingService.php`
  - [ ] Method: `submitToFiscalAuthority(TaxDeclaration $decl): SubmissionResult`
  - [ ] Determines authority endpoint based on $decl.jurisdiction and declarationType
  - [ ] For VAT → Belastingdienst endpoint
  - [ ] For IV3 → CBS endpoint
  - [ ] Serializes declaration to authority XSD format
  - [ ] Signs with DigiSign service (via separate contract)
  - [ ] POSTs to endpoint
  - [ ] Receives filing reference and stores in $decl.filingReference
  - [ ] Schedules webhook handler to poll for status updates
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#filing-service`
  - [ ] Integration test: Mock Belastingdienst endpoint; verify submission flow

- [ ] **Task: Create Tax Rate Validation Service**
  - [ ] File: `lib/Service/TaxRateValidator.php`
  - [ ] Method: `validateRate(TaxRate $rate): ValidationResult`
  - [ ] Checks jurisdiction rules:
    - [ ] KOR small business: revenue ≤ €20,000
    - [ ] Reverse charge: only for B2B, cross-border
    - [ ] Zero VAT: only for books, newspapers, medicines
  - [ ] If AvaTax is configured, calls `validateAgainstAvaTax(TaxRate $rate)`
  - [ ] Returns ValidationResult with status, warnings, errors
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#validator-service`
  - [ ] Unit test: Test each jurisdiction rule; test AvaTax mocking

- [ ] **Task: Create Webhook Handler for Filing Status Updates**
  - [ ] File: `lib/WebhookHandler/TaxFilingStatusHandler.php`
  - [ ] Method: `handle(WebhookEvent $event): void`
  - [ ] Listens for events:
    - [ ] `filing:accepted` — updates declaration.status = "accepted"
    - [ ] `filing:rejected` — updates declaration.status = "rejected"
    - [ ] `objection:accepted` — updates declaration.status = "resolved"
    - [ ] `objection:rejected` — updates declaration.status = "rejected"
  - [ ] Updates TaxDeclaration via ObjectService
  - [ ] Triggers notifications on state change
  - [ ] Logs to audit trail
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#webhook-handler`
  - [ ] Unit test: Mock webhook events; verify state transitions and notifications

## Frontend Pages

- [ ] **Task: Create Tax Settings Page (Admin)**
  - [ ] File: `src/pages/AdminSettings.vue`
  - [ ] Components:
    - [ ] `CnVersionInfoCard` (ADR-010 requirement) — show app version, OpenRegister status
    - [ ] Tax Configuration form:
      - [ ] Use `CnFormDialog` auto-generated from TaxConfiguration schema
      - [ ] Fields: Business Name, VAT Number, Jurisdiction, Fiscal Year, Default VAT Rate, VAT Scheme
      - [ ] Buttons: Save, Cancel
    - [ ] Tax Rate table:
      - [ ] Use `CnDataTable` populated from ObjectStore(TaxRate)
      - [ ] Columns: Name, Rate %, Jurisdiction, Valid From, Actions
      - [ ] Actions: Edit, Delete, Validate
      - [ ] Button: "Import Rates from CSV"
  - [ ] Store: Use `createObjectStore('TaxRate', 'TaxRate', 'shillinq')` with plugins (auditTrails, lifecycle)
  - [ ] Fetch settings on mount via `GET /api/settings`
  - [ ] Save via `POST /api/settings`
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#settings-page`

- [ ] **Task: Create Tax Dashboard (Finance Manager)**
  - [ ] File: `src/pages/TaxDashboard.vue`
  - [ ] Layout:
    - [ ] `CnPageHeader` — "Tax Dashboard 2026"
    - [ ] KPI section:
      - [ ] `CnStatsBlock` — "Annual Tax Liability" card with amount, trend
      - [ ] `CnStatsBlock` — "Q2 VAT Owed" card
      - [ ] `CnStatsBlock` — "Filing Deadline" card (days remaining)
      - [ ] `CnStatsBlock` — "Deductible Expenses" card (YTD)
    - [ ] Chart section:
      - [ ] `CnChartWidget` (line chart) — "Annual Tax Estimate" over months
      - [ ] `CnChartWidget` (donut chart) — "Income by Category" breakdown
    - [ ] Recent filings table:
      - [ ] `CnDataTable` — list of TaxDeclarations (type, period, status, deadline)
      - [ ] Click row → detail page
    - [ ] Button: "New Declaration"
  - [ ] Fetch data: `Promise.all([...taxConfigStore.fetch(), ...transactionStore.fetch(), ...])`
  - [ ] Real-time update: Refresh every 5 seconds when estimate is shown
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#dashboard-page`

- [ ] **Task: Create Tax Declaration Detail Page (Finance Manager)**
  - [ ] File: `src/pages/TaxDeclarationDetail.vue`
  - [ ] Two modes: edit (form) and view (detail page with cards)
  - [ ] View mode (default for existing declarations):
    - [ ] `CnPageHeader` — "VAT Return Q2 2026"
    - [ ] `CnDetailCard` sections:
      - [ ] Summary: name, period, status, deadline
      - [ ] Calculation breakdown: gross sales, VAT collected, VAT deductible, net owed
      - [ ] Action buttons: Edit, Review, File, Download PDF, Export JSON
    - [ ] Related transactions table (filtered by period + type):
      - [ ] Shows TaxableTransaction list for this declaration
      - [ ] Columns: Date, Amount, Category, VAT, Deductible
      - [ ] Click to drill into detail
    - [ ] `CnObjectSidebar` with tabs:
      - [ ] Files — attach supporting documents
      - [ ] Notes — internal review notes
      - [ ] Audit — change history
  - [ ] Edit mode (for draft declarations):
    - [ ] Form fields via `CnFormDialog` schema
    - [ ] Button: "Save Draft", "Cancel", "Ready for Review"
  - [ ] Fetch declaration: `store.getObjectById(id)`
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#detail-page`

- [ ] **Task: Create Tax Declaration List Page (Finance Manager)**
  - [ ] File: `src/pages/TaxDeclarationList.vue`
  - [ ] Layout:
    - [ ] `CnPageHeader` — "Tax Declarations"
    - [ ] `CnActionsBar` — Add, Search, Filter, Export buttons
    - [ ] `CnFilterBar` — filter by type (VAT, IV3, BCF, ESG), status (draft, submitted, accepted), year
    - [ ] `CnDataTable`:
      - [ ] Columns: Name, Type, Period, Status, Deadline, Filing Ref
      - [ ] Row click → detail page
      - [ ] Column sort: by period (desc), by status
      - [ ] Pagination: 25 rows/page
  - [ ] Store: `useListView('TaxDeclaration', { sidebarState, objectStore })`
  - [ ] Button: "New Declaration" → opens CnFormDialog (TaxDeclaration schema, new mode)
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#list-page`

- [ ] **Task: Create Receipt Upload Dialog**
  - [ ] File: `src/components/ReceiptUploadDialog.vue`
  - [ ] Dialog triggered from TaxableTransaction detail page
  - [ ] Features:
    - [ ] File upload (PDF, image)
    - [ ] Progress bar
    - [ ] On upload, call OcrService to extract data
    - [ ] Show extraction results with confidence
    - [ ] User confirms → auto-populates transaction fields
    - [ ] User can edit before saving
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#receipt-dialog`

## API Endpoints

- [ ] **Task: Create Tax API Endpoints**
  - [ ] File: `lib/Controller/TaxController.php`
  - [ ] Implements standard REST pattern (per ADR-002):
    - [ ] `GET /api/tax-configurations` — list (paginated)
    - [ ] `GET /api/tax-configurations/{id}` — detail
    - [ ] `POST /api/tax-configurations` — create
    - [ ] `PUT /api/tax-configurations/{id}` — update
    - [ ] `DELETE /api/tax-configurations/{id}` — delete
    - [ ] Similar for: `/api/tax-rates`, `/api/tax-declarations`, `/api/taxable-transactions`, `/api/tax-returns`, `/api/exemption-certificates`
  - [ ] Response format:
    ```json
    {
      "data": [...objects...],
      "total": 50,
      "page": 1,
      "pages": 2
    }
    ```
  - [ ] All endpoints use ObjectService (no custom Mapper)
  - [ ] Auth: Standard Nextcloud auth (IUserSession)
  - [ ] Admin-only endpoints: `/api/admin/*` tagged with `#[AdminRequired]`
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#api-endpoints`

- [ ] **Task: Create Filing Submission Endpoint**
  - [ ] Endpoint: `POST /api/tax-declarations/{id}/submit`
  - [ ] Validates declaration is in "ready-to-file" state
  - [ ] Calls TaxFilingService.submitToFiscalAuthority()
  - [ ] Returns filing reference and status
  - [ ] Response:
    ```json
    {
      "success": true,
      "filingReference": "VAT-NL-2026-Q2-98765",
      "message": "Filed successfully"
    }
    ```
  - [ ] Error handling: Returns 400 with validation error message
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#submit-endpoint`

- [ ] **Task: Create Receipt OCR Endpoint**
  - [ ] Endpoint: `POST /api/taxable-transactions/{id}/ocr`
  - [ ] Expects multipart form with file upload
  - [ ] Calls OcrService.extractReceiptData()
  - [ ] Returns extraction result (merchant, date, items, VAT, confidence)
  - [ ] Response:
    ```json
    {
      "merchant": "Restaurant De Groene Molen",
      "date": "2026-05-20",
      "items": [...],
      "totalAmount": 50.82,
      "vatAmount": 8.82,
      "vatRate": 21,
      "confidence": 0.95
    }
    ```
  - [ ] Spec tag: `@spec openspec/changes/tax-levy-management-other-t1/tasks.md#ocr-endpoint`

## Testing

- [ ] **Task: Unit Tests for Lifecycle Guard**
  - [ ] File: `tests/Unit/Lifecycle/TaxDeclarationValidationGuardTest.php`
  - [ ] Test cases:
    - [ ] `testCanTransitionDraftToReview()` — always true
    - [ ] `testCanTransitionReviewToReadyWithoutDuplicates()` — pass validation
    - [ ] `testCanTransitionReviewToReadyWithDuplicates()` — fail validation
    - [ ] `testCanTransitionReviewToReadyWithoutTaxCategories()` — fail validation
    - [ ] `testCanTransitionReviewToReadyReverseChargeIncorrect()` — fail validation
    - [ ] `testGetReasonForBlockedTransition()` — verify error messages
  - [ ] Run: `composer test -- tests/Unit/Lifecycle/TaxDeclarationValidationGuardTest.php`

- [ ] **Task: Unit Tests for Tax Rate Validator**
  - [ ] File: `tests/Unit/Service/TaxRateValidatorTest.php`
  - [ ] Test cases:
    - [ ] `testValidateKorEligible()` — revenue ≤ threshold
    - [ ] `testValidateKorExceeded()` — revenue > threshold fails
    - [ ] `testValidateReverseChargeOnlyB2B()` — non-B2B fails
    - [ ] `testValidateZeroVatOnlyBooks()` — non-book items fail
    - [ ] `testValidateAgainstAvaTax()` — mock AvaTax response
  - [ ] Run: `composer test -- tests/Unit/Service/TaxRateValidatorTest.php`

- [ ] **Task: Unit Tests for OCR Service**
  - [ ] File: `tests/Unit/Service/OcrServiceTest.php`
  - [ ] Test cases:
    - [ ] `testExtractReceiptHighConfidence()` — successful extraction
    - [ ] `testExtractReceiptLowConfidence()` — low confidence returns low score
    - [ ] `testExtractReceiptApiFailure()` — fallback on API error
    - [ ] `testExtractReceiptTimeout()` — handles timeout gracefully
  - [ ] Run: `composer test -- tests/Unit/Service/OcrServiceTest.php`

- [ ] **Task: Integration Tests for Tax Filing Submission**
  - [ ] File: `tests/Integration/TaxFilingSubmissionTest.php`
  - [ ] Test cases:
    - [ ] `testSubmitVatDeclarationSuccess()` — mock Belastingdienst success response
    - [ ] `testSubmitVatDeclarationFailure()` — mock Belastingdienst error response
    - [ ] `testWebhookHandlerAcceptsDeclaration()` — mock webhook payload processes correctly
    - [ ] `testWebhookHandlerRejectsDeclaration()` — mock webhook error payload
  - [ ] Mock Belastingdienst endpoint using GuzzleHttp mock handler
  - [ ] Run: `composer test -- tests/Integration/TaxFilingSubmissionTest.php`

- [ ] **Task: Browser Tests for Tax Dashboard**
  - [ ] File: `tests/Browser/TaxDashboardTest.spec.js` (Playwright)
  - [ ] Test cases:
    - [ ] `test('View tax liability estimate on dashboard')`
      - [ ] Load dashboard
      - [ ] Verify KPI card shows "Annual Tax Liability" with amount
      - [ ] Verify chart shows "Annual Tax Estimate"
    - [ ] `test('Create new VAT declaration from dashboard')`
      - [ ] Click "New Declaration" button
      - [ ] Fill form (type: VAT, period: Q2)
      - [ ] Click "Save"
      - [ ] Verify declaration appears in list
    - [ ] `test('Submit VAT declaration')`
      - [ ] Navigate to declaration detail
      - [ ] Click "Review" button
      - [ ] Wait for consistency checks (mock passing)
      - [ ] Click "File"
      - [ ] Confirm submission
      - [ ] Verify filing reference is shown
  - [ ] Mock external APIs (Belastingdienst, AvaTax) using MSW
  - [ ] Run: `npm run test:browser -- tests/Browser/TaxDashboardTest.spec.js`

- [ ] **Task: Regression Tests for Invoice/Expense Workflows**
  - [ ] File: `tests/Regression/ExistingWorkflowsTest.php` and `.spec.js`
  - [ ] Verify existing invoice and expense creation workflows still work:
    - [ ] Create invoice (no tax change)
    - [ ] Create purchase order (no tax change)
    - [ ] Record expense (no tax change)
  - [ ] Verify no new tax fields break existing forms
  - [ ] Run: `composer test && npm run test:browser`

- [ ] **Task: Run Full Test Suite**
  - [ ] `composer check:strict` (static analysis + unit tests)
  - [ ] `npm run test` (frontend unit tests)
  - [ ] `npm run test:browser` (Playwright browser tests)
  - [ ] All must pass before PR creation
  - [ ] Target: 0 failures, 0 skipped

## Documentation

- [ ] **Task: Create User Documentation**
  - [ ] File: `docs/tax-management.md` (English)
  - [ ] File: `docs/nl/belastingbeheer.md` (Dutch)
  - [ ] Sections:
    - [ ] Overview: What is the Tax module?
    - [ ] Getting Started: Set up tax configuration
    - [ ] Managing Tax Rates: Create, update, validate
    - [ ] Filing VAT Returns: Step-by-step filing process
    - [ ] Filing IV3: Consistency checks and submission
    - [ ] Receipt OCR: Upload and extract
    - [ ] Exemption Certificates: KOR and reverse charge
    - [ ] Troubleshooting: Common issues and solutions
  - [ ] Screenshots from running app (not mocked)
  - [ ] Include real examples (freelancer, consultant, retailer)
  - [ ] ADR-009 requirement: Every user-facing feature needs docs

- [ ] **Task: Create Admin Configuration Guide**
  - [ ] File: `docs/admin/tax-configuration.md`
  - [ ] Sections:
    - [ ] Tax Schema Setup
    - [ ] Configuring Belastingdienst Integration
    - [ ] Configuring AvaTax Integration
    - [ ] Configuring OCR (Scan & Herken)
    - [ ] Seed Data Import
    - [ ] Audit Trail Review

## Migration & Rollout

- [ ] **Task: Create Repair Step for Initial Load**
  - [ ] File: `lib/Migration/{Version}_InitializeTaxModule.php`
  - [ ] On first install:
    - [ ] Imports tax schemas from register template
    - [ ] Imports 3-5 seed data objects per schema
    - [ ] Creates default tax configuration (jurisdiction auto-detected from Nextcloud config)
    - [ ] Logs completion
  - [ ] On upgrade: Idempotent (no-op if already installed)
  - [ ] Test: Run twice; verify second run is no-op

- [ ] **Task: Plan Rollout to Production**
  - [ ] Checklist:
    - [ ] All tests passing (unit, integration, browser)
    - [ ] Security review completed (SPDX headers, auth checks, IDOR)
    - [ ] Documentation written (user + admin)
    - [ ] No TODOs in code
    - [ ] Spec tag (`@spec`) on all new public methods/classes
    - [ ] Compliance audit trail working
    - [ ] External API integrations mocked in tests
  - [ ] Rollout plan:
    - [ ] 1. Merge to main
    - [ ] 2. Tag release (e.g., `v1.0.0`)
    - [ ] 3. Deploy to staging; test all workflows
    - [ ] 4. Deploy to production; monitor for errors
    - [ ] 5. Notify users via release notes

## Final Verification Checklist

- [ ] **Schemas & Migrations**
  - [ ] TaxConfiguration, TaxRate, TaxableTransaction, TaxDeclaration, TaxReturn, ExemptionCertificate defined
  - [ ] All relations use OpenRegister relation mechanism (no foreign keys)
  - [ ] Seed data includes 3-5 realistic objects per schema
  - [ ] Migration is idempotent
  - [ ] `composer check:strict` passes

- [ ] **Lifecycle & State Machine**
  - [ ] TaxDeclaration lifecycle declared in register (draft → submitted → accepted)
  - [ ] TaxDeclarationValidationGuard implemented and tested
  - [ ] Webhooks handle filing status updates (accepted/rejected/objection)

- [ ] **Calculations & Aggregations**
  - [ ] Tax aggregations auto-calculate totals (VAT collected/deductible/owed)
  - [ ] Tax calculations auto-compute derived fields (deductibleAmount, isDeductible)
  - [ ] Notifications trigger on state change and deadline

- [ ] **Integrations**
  - [ ] OcrService calls Scan & Herken API
  - [ ] TaxFilingService submits to Belastingdienst
  - [ ] TaxRateValidator checks AvaTax
  - [ ] All integrations have mock tests

- [ ] **Frontend**
  - [ ] Tax Settings page (Admin): Tax configuration + rate management
  - [ ] Tax Dashboard (Finance Manager): KPIs + charts + filing list
  - [ ] Declaration Detail page: View + edit + review + file
  - [ ] Declaration List page: Filter + search + export
  - [ ] Receipt upload dialog: OCR extraction + confirmation
  - [ ] All pages use schema-driven forms, tables, sidebars (no custom components)

- [ ] **API**
  - [ ] Standard REST endpoints for all entities (/api/tax-*)
  - [ ] Filing submission endpoint (POST /api/tax-declarations/{id}/submit)
  - [ ] Receipt OCR endpoint (POST /api/taxable-transactions/{id}/ocr)
  - [ ] All endpoints tested via Newman/Postman collection
  - [ ] Errors return appropriate HTTP status + message (no stack traces)

- [ ] **Testing**
  - [ ] Unit tests for lifecycle guard, validators, services
  - [ ] Integration tests for filing submission, webhooks, OCR
  - [ ] Browser tests for dashboard, declaration creation, filing
  - [ ] Regression tests for existing invoice/expense workflows
  - [ ] `composer test && npm run test && npm run test:browser` all pass

- [ ] **Documentation**
  - [ ] User docs in docs/tax-management.md (EN + NL)
  - [ ] Admin docs in docs/admin/tax-configuration.md
  - [ ] Screenshots from running app
  - [ ] Every feature documented with examples

- [ ] **Compliance & Security**
  - [ ] `@spec` tags on all public methods/classes
  - [ ] SPDX headers on all PHP files (`@license AGPL-3.0-or-later`)
  - [ ] No PII in logs or error responses
  - [ ] File uploads validated (type + size)
  - [ ] API responses have no stack traces
  - [ ] `hydra-gate-spdx` passes
  - [ ] `hydra-gate-route-auth` passes (if auth-protected endpoints)
  - [ ] `hydra-gate-semantic-auth` passes (RBAC checks)

- [ ] **Specification Traceability**
  - [ ] Code changes link to spec tasks via `@spec` docblock tags
  - [ ] Spec requirements (REQ-XXX-NNN) link to test cases
  - [ ] Every task in tasks.md has corresponding code/test
