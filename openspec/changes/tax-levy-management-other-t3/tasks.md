# Tax & Levy Management — Implementation Tasks

## Task 1: Define TaxCode Schema

**Scope:** Create OpenRegister schema definition for TaxCode entity

- [ ] Add TaxCode schema to `lib/Settings/shillinq_tax_register.json`:
  - Properties: id (UUID), code (string), name (string), jurisdiction (enum), taxType (enum), rate (decimal), effectiveDate (date), expiryDate (date, nullable), calculationMethod (enum), dependsOnTaxCode (string, nullable), applicableTo (array), description (string, nullable), dutchTaxId (string, nullable)
  - Required fields: code, name, jurisdiction, taxType, rate, effectiveDate, calculationMethod, applicableTo
  - Add x-openregister-calculations: isActive (effectiveDate <= today AND expiryDate >= today), displayRate (rate * 100)
  
- [ ] Define register template section for TaxCode:
  - OpenAPI 3.0 format with x-openregister extensions
  - Mark as `x-openregister.type: "application"`
  
- [ ] Create seed data (3 examples in @self envelope format):
  - NL-VAT-21-standard (21%, effective indefinite)
  - NL-VAT-6-reduced (6%, effective indefinite)
  - NL-IMPORT-87 (5% machinery duty)
  
- [ ] Validate schema syntax via SchemaService mock test (≥1 test)

**Dependencies:** None

---

## Task 2: Define TaxScheme Schema

**Scope:** Create OpenRegister schema for TaxScheme (tax regime configuration)

- [ ] Add TaxScheme schema to register:
  - Properties: id (UUID), name (string), schemeType (enum: STANDARD/EXEMPTED/SIMPLIFIED/SPECIAL_REGIME), exemptionCode (enum, nullable), effectiveDate (date), expiryDate (date, nullable), applicableTaxCodes (array of TaxCode IDs), exemptTransactionTypes (array, nullable), withholding_tax_scheme (string, nullable), notes (string, nullable)
  - Required: name, schemeType, effectiveDate, applicableTaxCodes
  - Add x-openregister-calculations: isActive (as per TaxCode)
  
- [ ] Create seed data (2 examples):
  - "Freelancer KOR" (EXEMPTED, exemptionCode=KOR, applicableTaxCodes=[], exemptTransactionTypes=["INVOICE_SALE"])
  - "Standard VAT Trader" (STANDARD, applicableTaxCodes=["nl-vat-21-standard", "nl-vat-6-reduced"])
  
- [ ] Add relation rules:
  - Supplier.withholding_tax_scheme_id → TaxScheme.id (nullable)
  - Organization.tax_scheme_id → TaxScheme.id (nullable)
  
- [ ] Validate schema + seed data (≥1 integration test)

**Dependencies:** Task 1

---

## Task 3: Define TaxableTransaction Schema

**Scope:** Create OpenRegister schema for tax-categorized transactions

- [ ] Add TaxableTransaction schema:
  - Properties: id (UUID), transactionId (UUID), type (enum: INVOICE_SALE_LINE/INVOICE_PURCHASE_LINE/PAYMENT/IMPORT/EXPENSE), date (date), description (string), lineAmount (decimal), currency (string, ISO 4217), appliedTaxCode (string, nullable), taxAmount (decimal), totalAmount (decimal), taxCategory (enum: VAT_DEDUCTIBLE/VAT_OUTPUT/IMPORT_DUTY/WITHHOLDING/EXCISE/OTHER), fiscalYear (integer), reportingStatus (enum: PENDING/FILED/AMENDED/RECONCILED)
  - Required: transactionId, type, date, lineAmount, currency, taxCategory, fiscalYear
  - Default reportingStatus: PENDING
  
- [ ] Add x-openregister-calculations:
  - `taxAmount`: IF appliedTaxCode.calculationMethod = 'EXCLUSIVE' THEN lineAmount * rate ELSIF 'INCLUSIVE' THEN lineAmount - (lineAmount / (1 + rate)) ELSE 0
  - `totalAmount`: lineAmount + taxAmount (for EXCLUSIVE); lineAmount (for INCLUSIVE with gross amount)
  
- [ ] Add x-openregister-aggregations:
  - Sum all VAT_OUTPUT by taxCategory filtered by dateRange, fiscalYear
  - Sum all VAT_DEDUCTIBLE by taxCategory filtered by dateRange
  - Count by reportingStatus (pending, filed, amended, reconciled)
  
- [ ] Validate calculation formulas with test cases (3+ test scenarios: EXCLUSIVE, INCLUSIVE, compound)

**Dependencies:** Task 1

---

## Task 4: Define TaxDeclaration Schema

**Scope:** Create OpenRegister schema for tax returns (VAT-aangifte, IB-aangifte, VPB-aangifte)

- [ ] Add TaxDeclaration schema:
  - Properties: id (UUID), declarationType (enum: VAT_QUARTERLY/INCOME_ANNUAL/CORPORATE_ANNUAL/SUPPLEMENTARY_VAT), jurisdiction (enum: NL), periodStart (date), periodEnd (date), filingDate (date, nullable), status (enum: DRAFT/FILED/ACCEPTED/AMENDED/CLOSED), totalIncome (decimal), totalDeductible (decimal), calculatedLiability (decimal), provisionalPaymentsMade (decimal, default=0), balanceDue (decimal, nullable), declarationUrl (URI, nullable), attachedDocuments (array of file IDs, nullable), notes (string, nullable)
  - Required: declarationType, jurisdiction, periodStart, periodEnd, status, calculatedLiability
  
- [ ] Add x-openregister-calculations:
  - `balanceDue`: calculatedLiability - provisionalPaymentsMade
  
- [ ] Add x-openregister-aggregations:
  - `totalVATOutput`: SUM(TaxableTransaction.taxAmount WHERE taxCategory='VAT_OUTPUT' AND date BETWEEN periodStart AND periodEnd)
  - `totalVATDeductible`: SUM(TaxableTransaction.taxAmount WHERE taxCategory='VAT_DEDUCTIBLE' AND date BETWEEN periodStart AND periodEnd)
  - `netVAT`: totalVATOutput - totalVATDeductible
  
- [ ] Test aggregation scenarios (≥2: positive VAT due, negative VAT reclaim)

**Dependencies:** Task 3

---

## Task 5: Create Register Template Import Handler

**Scope:** Implement ConfigurationService integration for loading seed TaxCode + TaxScheme data

- [ ] Add `ImportHandler::importTaxCodesAndSchemes()` listener to repair step or settings-load endpoint:
  - Read `lib/Settings/shillinq_tax_register.json` component.objects
  - For each object: extract @self envelope (register, schema, slug)
  - Call `ObjectService::searchObjects()` with slug to check if already exists (idempotent)
  - If not exists: call `ObjectService::saveObject()` to create
  - If exists: skip (version_compare could check if seed version is newer, but not required for v1)
  
- [ ] Verify seed data is loaded on fresh install (test via integration test)

- [ ] Handle edge cases:
  - Malformed JSON in register → log error, do not halt
  - Duplicate slug entries → use first, log warning
  - Missing required fields → skip object, log error

**Dependencies:** Tasks 1, 2

---

## Task 6: Implement TaxCode Calculations (x-openregister-calculations)

**Scope:** Validate and register x-openregister-calculations metadata on TaxCode schema

- [ ] In OpenRegister SchemaService (or via register patch):
  - Register calculation `isActive`: (effectiveDate <= @now) AND (expiryDate IS NULL OR expiryDate >= @now)
  - Register calculation `displayRate`: rate * 100 (for UI: "21%")
  
- [ ] Test calculations:
  - TaxCode effective 2026-01-01, no expiry → isActive = true on any future date
  - TaxCode effective 2026-06-01, checked on 2026-05-20 → isActive = false
  - TaxCode effective 2026-01-01, expiry 2026-05-31, checked on 2026-06-01 → isActive = false
  - displayRate for rate=0.21 → returns 21
  
- [ ] Ensure calculations are available for queries via ObjectService (no custom service method)

**Dependencies:** Task 1

---

## Task 7: Implement TaxableTransaction Calculations (Tax Amount & Total)

**Scope:** Register and test x-openregister-calculations for VAT computation

- [ ] In register, define calculation `taxAmount` with formula:
  - EXCLUSIVE: `lineAmount * appliedTaxCode.rate`
  - INCLUSIVE: `lineAmount - (lineAmount / (1 + appliedTaxCode.rate))`
  - COMPOUND: `(lineAmount + baseTaxAmount) * compoundRate` (requires @reference to dependent TaxCode)
  
- [ ] Define calculation `totalAmount`:
  - EXCLUSIVE: `lineAmount + taxAmount`
  - INCLUSIVE: assume lineAmount is the total (no addition)
  
- [ ] Test scenarios (≥5 test cases):
  1. Simple EXCLUSIVE: €100 @ 21% → VAT €21, total €121
  2. Simple EXCLUSIVE: €100 @ 6% → VAT €6, total €106
  3. INCLUSIVE: €121 @ 21% → VAT €21, base €100
  4. COMPOUND: €1000 base, import duty 5% (€50), VAT 21% on (1000+50) = €220.50
  5. Multiple lines with different rates
  
- [ ] Ensure calculations are persisted on TaxableTransaction.save() (or via trigger)

**Dependencies:** Task 3, Task 1

---

## Task 8: Implement TaxDeclaration Aggregations

**Scope:** Register x-openregister-aggregations for VAT return summary

- [ ] Define aggregation `totalVATOutput`:
  - Query: SUM(TaxableTransaction.taxAmount WHERE taxCategory = 'VAT_OUTPUT' AND date BETWEEN @.periodStart AND @.periodEnd)
  
- [ ] Define aggregation `totalVATDeductible`:
  - Query: SUM(TaxableTransaction.taxAmount WHERE taxCategory = 'VAT_DEDUCTIBLE' AND date BETWEEN @.periodStart AND @.periodEnd)
  
- [ ] Define aggregation `netVAT`:
  - Derived: totalVATOutput - totalVATDeductible
  
- [ ] Test with sample data:
  - Q2 2026: output VAT €10,500, deductible €4,500, net €6,000
  - Verify aggregations return correct sums
  - Verify availability via ObjectService.getAggregate() (no custom method)
  
- [ ] Test reclaim scenario: output €2,000, deductible €5,000, net = -€3,000

**Dependencies:** Task 4, Task 3

---

## Task 9: Implement TaxCode Validity Enforcement

**Scope:** Add validation rules to prevent applying inactive tax codes

- [ ] Create lifecycle guard or pre-save validator:
  - When TaxableTransaction.appliedTaxCode is set, check TaxCode.isActive calculation
  - If inactive: reject with message "This code is not active. Effective: {effectiveDate}, Expiry: {expiryDate}"
  
- [ ] Scenarios to test:
  - Code with future effective date → reject
  - Code with past expiry date → reject
  - Code with no expiry (indefinite) → accept
  - Code exactly at effective date → accept
  - Code exactly at expiry date → accept (edge case; depends on <=/>= rules)
  
- [ ] Implement in OpenRegister via schema-level guard or PHP guard class:
  - Register in TaxableTransaction schema: `x-openregister-lifecycle.requires: ["TaxCodeValidityGuard"]`
  - Guard checks appliedTaxCode.isActive
  
- [ ] Test with integration test (≥3 scenarios)

**Dependencies:** Task 1, Task 3, Task 6

---

## Task 10: Create Integration Test Suite for VAT Calculations

**Scope:** PHPUnit integration tests verifying end-to-end VAT scenarios

- [ ] Create `tests/Integration/TaxCalculationTest.php`:
  - Test 1: EXCLUSIVE VAT (€100 @ 21% = €121)
  - Test 2: INCLUSIVE VAT reverse calc (€121 @ 21% = €100 base + €21 VAT)
  - Test 3: COMPOUND VAT (import duty + VAT on duty)
  - Test 4: KOR exemption (no VAT applied)
  - Test 5: Multiple lines, different rates
  
- [ ] Verify calculations are stored correctly in TaxableTransaction
  
- [ ] Verify calculations are available via ObjectService queries (no custom service call)
  
- [ ] Run test suite: `composer run test:integration`

**Dependencies:** Tasks 1, 3, 6, 7

---

## Task 11: Create Integration Test for VAT Return Aggregation

**Scope:** PHPUnit integration tests verifying VAT return drafting and aggregations

- [ ] Create `tests/Integration/TaxReturnAggregationTest.php`:
  - Scenario 1: Q2 2026 VAT return with output €10,500, deductible €4,500
    - Verify: totalVATOutput = €10,500
    - Verify: totalVATDeductible = €4,500
    - Verify: netVAT = €6,000
  
  - Scenario 2: Reclaim scenario (output €2,000, deductible €5,000)
    - Verify: netVAT = -€3,000 (reclaim due)
  
  - Scenario 3: Provisional payment reconciliation
    - Paid: €20,000 provisional
    - Calculated liability: €7,000
    - Verify: balanceDue = €7,000 - €20,000 = -€13,000 (refund)
  
- [ ] Verify aggregations are accessible via ObjectService.getAggregate() or direct query

**Dependencies:** Task 4, Task 8

---

## Task 12: Create UI / API Endpoint for Tax Code Library

**Scope:** Expose TaxCode CRUD via OpenRegister API

- [ ] Verify default OpenRegister API works for TaxCode:
  - GET `/index.php/apps/shillinq/api/tax-codes` (list, paginated)
  - GET `/index.php/apps/shillinq/api/tax-codes/{id}` (detail)
  - POST `/index.php/apps/shillinq/api/tax-codes` (create, admin-only)
  - PUT `/index.php/apps/shillinq/api/tax-codes/{id}` (update, admin-only)
  - DELETE `/index.php/apps/shillinq/api/tax-codes/{id}` (delete, admin-only)
  
- [ ] Add RBAC checks:
  - Create/update/delete → admin role (via PropertyRbacHandler)
  - Read → all authenticated users
  
- [ ] Test via Postman/Newman (≥3 test cases):
  - List tax codes (auth required)
  - Create new code (admin required, non-admin blocked)
  - Update code (admin required)
  
- [ ] No custom controller needed (ObjectService default CRUD + RBAC)

**Dependencies:** Task 1, Task 5

---

## Task 13: Create UI / API for Tax Scheme CRUD

**Scope:** Expose TaxScheme CRUD via OpenRegister API

- [ ] Verify default API routes:
  - GET `/index.php/apps/shillinq/api/tax-schemes` (list)
  - POST `/index.php/apps/shillinq/api/tax-schemes` (create)
  - PUT `/index.php/apps/shillinq/api/tax-schemes/{id}` (update)
  - DELETE `/index.php/apps/shillinq/api/tax-schemes/{id}` (delete)
  
- [ ] Add RBAC:
  - Create/update/delete → admin + tax admin role
  - Read → all authenticated users
  
- [ ] Add relation linking when creating TaxScheme:
  - Supplier.withholding_tax_scheme_id can reference TaxScheme
  - Organization.tax_scheme_id can reference TaxScheme
  
- [ ] Test create/read/update (≥2 scenarios)

**Dependencies:** Task 2, Task 5

---

## Task 14: Create UI / API for TaxableTransaction Query & Reporting

**Scope:** Expose TaxableTransaction queries (with filtering) for tax category reporting

- [ ] Verify ObjectService.findAll() supports filters:
  - GET `/index.php/apps/shillinq/api/taxable-transactions?date_gte=2026-04-01&date_lte=2026-06-30&taxCategory=VAT_OUTPUT`
  - GET `/index.php/apps/shillinq/api/taxable-transactions?fiscalYear=2026&reportingStatus=PENDING`
  
- [ ] Test aggregation endpoint (if custom endpoint needed):
  - GET `/index.php/apps/shillinq/api/taxable-transactions/aggregate?date_gte=2026-04-01&date_lte=2026-06-30&groupBy=taxCategory`
  - Response: { "VAT_OUTPUT": 10500, "VAT_DEDUCTIBLE": 4500, "net": 6000 }
  
- [ ] Verify no custom service class needed (default aggregation via schema engine)

**Dependencies:** Task 3, Task 7

---

## Task 15: Create UI / API for Tax Declaration Drafting & Filing

**Scope:** Expose TaxDeclaration CRUD and filing workflow

- [ ] Verify default API:
  - GET `/index.php/apps/shillinq/api/tax-declarations` (list)
  - POST `/index.php/apps/shillinq/api/tax-declarations` (create draft)
  - PUT `/index.php/apps/shillinq/api/tax-declarations/{id}` (update, only DRAFT status)
  - PUT `/index.php/apps/shillinq/api/tax-declarations/{id}/file` (transition to FILED, lock for editing)
  
- [ ] Implement filing action (custom endpoint):
  - Check status = DRAFT
  - Set status = FILED, filingDate = today
  - Lock object (via ObjectService.lockObject())
  - Return confirmation with reference number
  
- [ ] Test scenarios:
  - Create draft VAT return
  - Update draft (add notes)
  - File declaration (transitions to FILED, locked)
  - Try to edit filed declaration → error
  
- [ ] RBAC: allow tax admin + owner to create/file; view available to all

**Dependencies:** Task 4, Task 8

---

## Task 16: Implement Provisional Payment Reconciliation Logic

**Scope:** Add custom logic (PHP) to reconcile provisional taxes against calculated liability

- [ ] Create `src/Service/TaxReconciliationService.php`:
  - Method: `calculateReconciliation(organizationId, fiscalYear) → array`
  - Logic:
    - Query all TaxPayment records for org + fiscal year with type='provisional'
    - Sum provisional payments
    - Query TaxDeclaration for same period, get calculatedLiability
    - Return: { provisionalPaid, calculatedLiability, balanceDue, isRefundDue }
  
- [ ] Add lifecycle event handler to TaxDeclaration on update:
  - When status changes to FILED, call TaxReconciliationService to populate balanceDue
  
- [ ] Test with scenarios:
  - Provisional €20,000, liability €7,000 → refund €13,000
  - Provisional €5,000, liability €8,000 → additional €3,000
  
- [ ] Add method to @spec tags:
  - `@spec openspec/changes/tax-levy-management-other-t3/specs.md#REQ-TAX-008`

**Dependencies:** Task 4

---

## Task 17: Implement Withholding Tax Configuration on Supplier

**Scope:** Link Supplier entity to TaxScheme for withholding tax tracking

- [ ] Extend Supplier schema (register patch):
  - Add field: `withholding_tax_scheme_id` (string, nullable, relation to TaxScheme.id)
  
- [ ] Create migration (repair step):
  - No data migration needed (adding nullable field is non-breaking)
  
- [ ] Implement payment recording logic:
  - When recording a payment to supplier with withholding_tax_scheme_id set
  - Calculate: withholding_amount = gross_amount * taxScheme.withholding_tax_scheme.rate
  - Record: PaymentRecord.withholding_tax_amount, withholding_tax_basis
  - Create TaxableTransaction: type=PAYMENT, taxCategory=WITHHOLDING, taxAmount=withholding_amount
  
- [ ] Add test:
  - Supplier with 20% withholding
  - Payment €1,000 gross → €200 withheld, €800 net
  - Verify TaxableTransaction created with WITHHOLDING category

**Dependencies:** Task 2, Task 3

---

## Task 18: Create Admin Settings Page for Tax Configuration

**Scope:** Build admin UI for managing TaxCode library and import/export

- [ ] No custom Vue component needed (use CnIndexPage for TaxCode list):
  - List view: columns for code, name, rate, jurisdiction, active status
  - Detail view: edit form with all fields
  - Action buttons: add code, edit, delete (with safeguards per REQ-TAX-001)
  
- [ ] Add import button:
  - "Re-import seed data" button → calls POST `/api/settings/load` to reload from register
  - Confirm: "This will not overwrite existing codes"
  
- [ ] Add settings page link from main menu (if needed for Shillinq app layout)

**Dependencies:** Task 12

---

## Task 19: Deduplication Check

**Scope:** Verify no overlap with existing OpenRegister services

- [ ] Review existing OpenRegister services:
  - [ ] ConfigurationService (handles import of register templates) ✓ used for seed data load
  - [ ] ObjectService (CRUD on OR objects) ✓ used for TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration
  - [ ] SchemaService (schema validation) ✓ used for register validation
  - [ ] AuditTrailService (auto-tracks changes) ✓ used for tax transaction audit
  - [ ] PropertyRbacHandler (field-level RBAC) ✓ used for admin-only edit of TaxCode
  - [ ] CnIndexPage + CnDetailPage (UI components) ✓ used for CRUD views
  
- [ ] Verify NO custom service classes needed for:
  - [ ] Tax calculation (uses x-openregister-calculations)
  - [ ] Tax aggregation (uses x-openregister-aggregations)
  - [ ] Tax scheme application (uses schema queries + relations)
  - [ ] Notification on tax filing (deferred; no x-openregister-notifications yet)
  
- [ ] Exception: TaxReconciliationService (Task 16) is JUSTIFIED because:
  - Orchestrates multiple objects (TaxPayment + TaxDeclaration)
  - Not yet supported by schema engine extensions
  - Document in design.md under "Exceptions"
  
- [ ] Document findings in this task checklist ✓

**Dependencies:** All schema tasks (1-4)

---

## Task 20: Create Seed Data Load Test

**Scope:** Integration test verifying seed TaxCode + TaxScheme are loaded on install

- [ ] Create `tests/Integration/SeedDataLoadTest.php`:
  - Mock installation scenario
  - Call repair step (or import handler)
  - Verify TaxCode objects created:
    - "NL-VAT-21-standard" exists with rate=0.21
    - "NL-VAT-6-reduced" exists with rate=0.06
    - "NL-IMPORT-87" exists with rate=0.05
  - Verify TaxScheme objects created:
    - "Freelancer KOR" exists with exemptionCode=KOR
    - "Standard VAT Trader" exists with applicableTaxCodes array
  
- [ ] Test idempotency:
  - Run import twice
  - Verify no duplicate objects created
  - Verify second run is no-op (matched by slug)
  
- [ ] Run test: `composer run test:integration`

**Dependencies:** Task 5

---

## Task 21: End-to-End Scenario Test: KOR Exemption

**Scope:** Integration test verifying freelancer under KOR exemption

- [ ] Test scenario:
  - Create Organization with TaxScheme="Freelancer KOR"
  - Create invoice for that org with:
    - 3 line items (services, goods, supplies)
    - Attempt to apply VAT code → system prevents (KOR has no applicable codes)
  - Create TaxableTransaction for each line with taxCategory=NONE or taxCategory=EXEMPT
  - Generate tax return (Q2 2026):
    - totalIncome: €45,000
    - totalDeductible: €0 (input VAT not deductible under KOR)
    - calculatedLiability: €0
    - Status: no VAT return needed (KOR exemption)
  
- [ ] Verify:
  - No VAT amounts calculated on any line
  - Tax return shows exemption reason
  - No tax payment due
  
- [ ] File test at: `tests/Integration/KORExemptionTest.php`

**Dependencies:** Tasks 2, 3, 4, 7, 8

---

## Task 22: End-to-End Scenario Test: Standard Trader with Reclaim

**Scope:** Integration test for normal VAT trader with input VAT reclaim

- [ ] Test scenario (Q2 2026):
  - Organization: Standard VAT Trader
  - Sales invoices (VAT-21):
    - Invoice 1: €10,000 + VAT €2,100
    - Invoice 2: €5,000 + VAT €1,050
    - Total output: €2,100 + €1,050 = €3,150
  - Purchase invoices (mixed):
    - Purchases @ VAT-21: €15,000 + VAT €3,150
    - Purchases @ VAT-6: €10,000 + VAT €600
    - Total input: €3,150 + €600 = €3,750
  - Create TaxableTransaction for each invoice line
  - Generate VAT return (Q2):
    - totalVATOutput: €3,150
    - totalVATDeductible: €3,750
    - Net VAT: -€600 (reclaim due)
    - Status: DRAFT → FILED → RECLAIM_PENDING
  
- [ ] Verify:
  - Reclaim amount (€600) correctly calculated
  - Return marked as RECLAIM (not standard return)
  - Audit trail shows all transactions + aggregation
  
- [ ] File test at: `tests/Integration/VATReclaimTest.php`

**Dependencies:** Tasks 3, 4, 7, 8

---

## Task 23: End-to-End Scenario Test: Compound Import Duty + VAT

**Scope:** Integration test for import with compound duty calculation

- [ ] Test scenario:
  - Create invoice for imported machinery:
    - Base cost: €5,000
    - Tax codes applied: IMPORT-87 (5%, EXCLUSIVE), then VAT-21 on (base + duty)
  
- [ ] Verify calculations:
  - Import duty: €5,000 × 0.05 = €250
  - VAT base: €5,000 + €250 = €5,250
  - VAT: €5,250 × 0.21 = €1,102.50
  - Total: €5,000 + €250 + €1,102.50 = €6,352.50
  
- [ ] Verify TaxableTransaction:
  - First line: lineAmount=5000, appliedTaxCode=IMPORT-87, taxAmount=250, totalAmount=5250
  - OR second line: lineAmount=5250 (cumulative), appliedTaxCode=VAT-21, taxAmount=1102.50, totalAmount=6352.50
  - (Exact structure depends on whether compound is modeled as multi-line or single calculation)
  
- [ ] Test recalculation if duty rate changes mid-invoice
  
- [ ] File test at: `tests/Integration/CompoundTaxTest.php`

**Dependencies:** Tasks 1, 3, 7

---

## Task 24: Add @spec Tags to PHPUnit Tests

**Scope:** Document test traceability to OpenSpec

- [ ] Add @spec PHPDoc tags to all test files:
  - Class-level: `@spec openspec/changes/tax-levy-management-other-t3/specs.md#REQ-TAX-001` (primary)
  - Method-level: additional REQ-* references per test coverage
  
- [ ] Example format:
  ```php
  /**
   * @spec openspec/changes/tax-levy-management-other-t3/specs.md#REQ-TAX-001
   * @spec openspec/changes/tax-levy-management-other-t3/specs.md#REQ-TAX-003
   */
  public function testExclusiveVATCalculation() { ... }
  ```
  
- [ ] Update test files:
  - Task 10 tests (VAT calculations) → REQ-TAX-002, REQ-TAX-003, REQ-TAX-004
  - Task 11 tests (aggregations) → REQ-TAX-008, REQ-TAX-009, REQ-TAX-023
  - Task 20 (seed data) → REQ-TAX-001
  - Task 21 (KOR) → REQ-TAX-007
  - Task 22 (reclaim) → REQ-TAX-023
  - Task 23 (compound) → REQ-TAX-004

**Dependencies:** All previous test tasks (10, 11, 20-23)

---

## Task 25: Verify Register Template Syntax & Completeness

**Scope:** Final validation of `lib/Settings/shillinq_tax_register.json`

- [ ] Checklist:
  - [ ] Valid OpenAPI 3.0 structure (info, paths, components.schemas)
  - [ ] `x-openregister` metadata present (type: "application", version: "1.0.0")
  - [ ] All 5 schemas defined (TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration, plus reference to TaxRate)
  - [ ] All properties have type, description, required flags
  - [ ] x-openregister-calculations present on TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration
  - [ ] x-openregister-aggregations present on TaxDeclaration and TaxableTransaction
  - [ ] Seed data includes 3 TaxCode + 2 TaxScheme examples in @self format
  - [ ] Seed data uses realistic Dutch values (street names, postcodes, municipalities)
  - [ ] JSON is valid (no syntax errors)
  
- [ ] Run syntax check via SchemaService (or JSON validator)
  
- [ ] Verify no hardcoded foreign keys; use relations instead

**Dependencies:** Tasks 1-4

---

## Task 26: Create Feature Documentation

**Scope:** Write user-facing docs for tax management features

- [ ] Create `docs/tax-management.md`:
  - Overview: what is tax management in Shillinq?
  - Getting started: setting up tax schemes, importing tax codes
  - VAT return workflow: creating a quarterly return, filing
  - KOR exemption guide: how to configure, what to file
  - Withholding tax setup: configuring suppliers with withholding rules
  - Screenshots: tax code list, invoice with VAT, return draft, filed declaration
  
- [ ] Create `docs/tax-codes.md`:
  - List of pre-loaded Dutch tax codes (VAT-21, VAT-6, IMPORT codes, etc.)
  - How to add custom tax codes
  - Effective date and expiry date rules
  
- [ ] Translate: English primary; Dutch translations (docs/nl/tax-management.md)

**Dependencies:** All implementation tasks (1-25)

---

## Task 27: Commit All Changes

**Scope:** Stage and commit the OpenSpec change artifacts

- [ ] Stage OpenSpec files:
  - `openspec/changes/tax-levy-management-other-t3/proposal.md`
  - `openspec/changes/tax-levy-management-other-t3/design.md`
  - `openspec/changes/tax-levy-management-other-t3/specs.md`
  - `openspec/changes/tax-levy-management-other-t3/tasks.md`
  
- [ ] Commit message:
  ```
  feat: Add OpenSpec change tax-levy-management-other-t3 from Specter
  
  Tax & Levy Management spec for Shillinq:
  - 31 market features related to VAT, income tax, corporate tax
  - Schema-only config (kind: config) per ADR-032
  - 5 OpenRegister schemas: TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration, TaxRate (enhanced)
  - Seed data: 3 tax codes, 2 schemes
  - 25 implementation tasks covering schema definition, calculations, aggregations, UI, and testing
  - Zero custom service classes (declarative-first per ADR-031)
  
  Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>
  ```
  
- [ ] Verify all 4 artifacts exist and are valid Markdown

**Dependencies:** All previous tasks (1-26)
