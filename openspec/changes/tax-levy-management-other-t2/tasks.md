# Tasks: Tax & Levy Management — Shillinq — Other T2

**Status:** Ready for Implementation  
**Date:** 2026-05-21  
**Change ID:** tax-levy-management-other-t2  
**Story Points:** 89 (estimated)  

---

## Epic 1: Schema & Data Layer (20 points)

### TASK-001 [Schema] Create TaxExemption Schema
- [ ] Define `TaxExemption` schema (exemptionType, jurisdiction, validFrom, conditions, etc.)
- [ ] Add to `lib/Settings/shillinq_register.json`
- [ ] Mark required fields (exemptionType, jurisdiction, validFrom)
- [ ] Add descriptions and enum values per design.md
- [ ] Create seed data (3–5 exemptions: KOR, B2B, reverse charge)
- [ ] Run `composer check:strict` to validate schema
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-001

**Acceptance:**
- [ ] Schema valid JSON Schema
- [ ] All enum values documented
- [ ] Seed data includes realistic Dutch exemptions (KOR, B2B)
- [ ] Schema loadable without errors

---

### TASK-002 [Schema] Create TaxDetectionRule Schema
- [ ] Define `TaxDetectionRule` schema (priority, name, conditions, outcome)
- [ ] Add to `lib/Settings/shillinq_register.json`
- [ ] Support condition operators (equality, list match, range)
- [ ] Seed with 10 pre-built rules:
  - [ ] NL domestic sale (standard VAT)
  - [ ] NL domestic purchase (standard VAT)
  - [ ] EU B2B reverse charge
  - [ ] Import goods (standard VAT)
  - [ ] B2C exports (zero VAT)
  - [ ] B2B exports (zero VAT)
  - [ ] Small purchase (<€1,000)
  - [ ] Inter-company (for future multi-entity)
  - [ ] Creative services (special rules)
  - [ ] Digital services (cross-border VAT)
- [ ] Validate rule ordering by priority
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-002

**Acceptance:**
- [ ] Schema accepts all condition types (country, amount, type, keywords)
- [ ] 10 seed rules properly ordered by priority
- [ ] Test: Query rules by priority returns correct order

---

### TASK-003 [Schema] Extend TaxRate Schema
- [ ] Add `compoundWith` field (nullable, ref to TaxRate)
- [ ] Add `inclusive` boolean field (default: false)
- [ ] Add `rateType` enum (Standard, Reduced, Zero, ReverseCharge, Exempt)
- [ ] Create seed data (5 rates: NL 21%, 9%, 0%; DE 19%, 0%)
- [ ] Validate: compoundWith must reference existing rate
- [ ] Update existing TaxRate schema if separate file exists
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-003

**Acceptance:**
- [ ] Extended fields present in schema
- [ ] Seed data includes multiple countries
- [ ] Test: Calculate compound rate (€1,000 × 0.21 + compound 0.08 = €1,306.80)

---

### TASK-004 [Schema] Extend TaxConfiguration Schema
- [ ] Add `autodetectionEnabled` boolean
- [ ] Add `detectionRules` array (TaxDetectionRule IDs)
- [ ] Add `multiJurisdictionEnabled` boolean
- [ ] Add `jurisdictions` array
- [ ] Add `exemptionRules` array (TaxExemption IDs)
- [ ] Add `recoveryRules` object (inboundVATRecoveryAllowed, partialRecoveryPercentage, excludedCategories)
- [ ] Create seed configuration with defaults for NL org
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-004

**Acceptance:**
- [ ] All fields present and typed correctly
- [ ] Defaults sensible for NL SMB (auto-detection on, KOR enabled, 100% recovery)
- [ ] Test: Load config, verify rules and exemptions accessible

---

### TASK-005 [Migration] Create Repair Step for Schema Import
- [ ] Implement `IRepairStep` for TaxExemption, TaxDetectionRule schema import
- [ ] Load from `lib/Settings/shillinq_register.json`
- [ ] Use `ConfigurationService::importFromApp()` with idempotency
- [ ] Handle version bumps (re-import skips existing by slug)
- [ ] Log import results (created, skipped, updated)
- [ ] Test: Run repair step, verify schemas available
- [ ] Test: Run repair step again (no duplicates)
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-005

**Acceptance:**
- [ ] Repair step runs without errors
- [ ] Seed data present in OpenRegister after repair
- [ ] Idempotent: second run creates no duplicates
- [ ] Audit trail shows import action

---

## Epic 2: Backend Services & API (32 points)

### TASK-006 [Service] Implement TaxDetectionService
- [ ] Create `lib/Service/TaxDetectionService.php`
- [ ] Implement `determine(array $metadata): TaxDetectionResult`
  - Input: transaction metadata (supplierCountry, customerCountry, transactionType, amount, customerType, keywords)
  - Output: { vatStatus, ratePercentage, jurisdiction, ruleApplied, explanation }
- [ ] Load rules from TaxConfiguration in priority order
- [ ] Match conditions using flexible operator system
- [ ] Check exemptions first (stop on first match)
- [ ] Fallback to default VAT status if no rule matches
- [ ] Log rule evaluation for debugging
- [ ] Inject `ConfigurationService`, `ObjectService`
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-006

**Acceptance:**
- [ ] `determine()` called with metadata returns correct vatStatus and rate
- [ ] Test: Domestic NL sale → Standard, 21%
- [ ] Test: EU B2B reverse charge → ReverseCharge, 0%
- [ ] Test: No matching rule → Fallback to Standard, default rate
- [ ] Test: KOR exemption checked first, blocks further rules
- [ ] Explanation field includes rule name for audit

---

### TASK-007 [Service] Implement TaxCalculationService
- [ ] Create `lib/Service/TaxCalculationService.php`
- [ ] Implement `calculateTax(array $parameters): TaxCalculationResult`
  - Parameters: netAmount, rate, inclusive (bool)
  - Output: { netAmount, taxAmount, grossAmount }
- [ ] Support exclusive rates (net × rate = tax)
- [ ] Support inclusive rates (gross ÷ (1 + rate) = net)
- [ ] Support compound rates (base rate first, then compound rate on total)
- [ ] Handle edge cases (zero rates, very small amounts)
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-007

**Acceptance:**
- [ ] Exclusive: €1,000 net, 0.21 rate → €210 tax, €1,210 gross
- [ ] Inclusive: €1,210 gross, 0.21 rate → €1,000 net, €210 tax
- [ ] Compound: €1,000, VAT 0.21 + sales tax 0.08 → €306.80 total tax
- [ ] Test with decimals (e.g., 0.065) for precision

---

### TASK-008 [Service] Implement TaxExemptionService
- [ ] Create `lib/Service/TaxExemptionService.php`
- [ ] Implement `checkExemptibility(TaxableTransaction $tx): TaxExemption|null`
  - Check all applicable exemptions (jurisdiction, date valid)
  - Match conditions (amount, supplier type, transaction type)
  - Return first matching exemption or null
- [ ] Implement `applyExemption(TaxableTransaction $tx, TaxExemption $exemption): void`
  - Set `tx.vatStatus = Exempt`
  - Set `tx.taxAmount = 0`
  - Store exemption ID for audit
- [ ] Log exemption application with user/timestamp
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-008

**Acceptance:**
- [ ] checkExemptibility returns KOR rule for NL sale under €50k
- [ ] checkExemptibility returns null if threshold exceeded
- [ ] checkExemptibility returns null for expired rule
- [ ] applyExemption sets transaction fields and stores rule ID
- [ ] Audit trail shows exemption with rule name

---

### TASK-009 [Service] Implement TaxRecoveryService
- [ ] Create `lib/Service/TaxRecoveryService.php`
- [ ] Implement `calculateRecoverableVAT(TaxableTransaction $tx): array`
  - Load recovery rules from TaxConfiguration
  - Return { recoverable, nonRecoverable }
- [ ] Check excluded categories
- [ ] Apply partial recovery percentage
- [ ] Handle split transactions (some recoverable, some not)
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-009

**Acceptance:**
- [ ] Office supplies €210 VAT → €210 recoverable
- [ ] Meals €42 VAT → €0 recoverable
- [ ] Partial recovery 80%: €10,500 VAT → €8,400 recoverable
- [ ] Test mixed categories in same transaction

---

### TASK-010 [API] TaxExemptions Controller
- [ ] Create `lib/Controller/TaxExemptionsController.php`
- [ ] `GET /api/exemptions` — list all (filter: jurisdiction, date)
- [ ] `POST /api/exemptions` — create exemption
- [ ] `PUT /api/exemptions/{id}` — update exemption
- [ ] `DELETE /api/exemptions/{id}` — delete exemption
- [ ] Validate: jurisdiction enum, date logic (validFrom ≤ validTo if present)
- [ ] Return HTTP 400 with error message for validation failure
- [ ] Use `ObjectService` for CRUD
- [ ] Inject services: `ObjectService`, `TaxExemptionService`
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-010

**Acceptance:**
- [ ] POST creates exemption, returns ID
- [ ] GET /api/exemptions returns all exemptions
- [ ] GET /api/exemptions?jurisdiction=NL filters correctly
- [ ] PUT updates exemption, logs change
- [ ] DELETE removes exemption

---

### TASK-011 [API] TaxDetectionRules Controller
- [ ] Create `lib/Controller/TaxDetectionRulesController.php`
- [ ] `GET /api/detection-rules` — list all (filter: enabled, priority order)
- [ ] `POST /api/detection-rules` — create rule
- [ ] `PUT /api/detection-rules/{id}` — update rule
- [ ] `DELETE /api/detection-rules/{id}` — delete rule
- [ ] `POST /api/detection-rules/{id}/test` — test rule against sample metadata
- [ ] Validation: priority unique (or allow duplicates with position sorting)
- [ ] Use `ObjectService` for CRUD
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-011

**Acceptance:**
- [ ] Create rule with conditions and outcome
- [ ] List returns rules ordered by priority
- [ ] Test endpoint: pass metadata, get { matches: bool, outcome }
- [ ] PUT reorders if priority changed

---

### TASK-012 [API] Tax Detection Endpoint
- [ ] Create `GET /api/tax-detection/determine`
- [ ] Parameters: transactionType, supplierCountry, customerCountry, amount, customerType (optional), keywords (optional)
- [ ] Call `TaxDetectionService::determine()`
- [ ] Return: { vatStatus, rate, jurisdiction, ruleApplied, explanation }
- [ ] Return HTTP 200 even if no rule matches (fallback applies)
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-012

**Acceptance:**
- [ ] GET with NL→NL sale params returns Standard, 21%
- [ ] GET with DE→NL purchase params returns ReverseCharge, 0%
- [ ] GET with no match returns Standard, default rate, explanation: "no rule matched"

---

### TASK-013 [API] Tax Calculation Endpoint
- [ ] Create `POST /api/tax-calculation`
- [ ] Body: { netAmount, ratePercentage, inclusive }
- [ ] Call `TaxCalculationService::calculateTax()`
- [ ] Return: { netAmount, taxAmount, grossAmount }
- [ ] Handle compound rates if provided
- [ ] Validate: rate ≥ 0, amount > 0
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-013

**Acceptance:**
- [ ] POST with €1,000 net, 0.21, exclusive → { net: 1000, tax: 210, gross: 1210 }
- [ ] POST with €1,210 gross, 0.21, inclusive → { net: 1000, tax: 210, gross: 1210 }

---

### TASK-014 [API] Tax Configuration Controller
- [ ] Create `lib/Controller/TaxConfigurationController.php`
- [ ] `GET /api/tax-configuration` — get current org config
- [ ] `PUT /api/tax-configuration` — update config
- [ ] Load config from TaxConfiguration by org ID
- [ ] Update: autodetectionEnabled, jurisdictions, recoveryRules, etc.
- [ ] Validate: jurisdictions exist as valid enum, recovery % 0–100
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-014

**Acceptance:**
- [ ] GET returns current config
- [ ] PUT updates recoveryRules, auto-detection toggle, jurisdictions
- [ ] PUT persists changes to OpenRegister

---

## Epic 3: Frontend Pages & Components (24 points)

### TASK-015 [Frontend] Tax Exemptions Index Page
- [ ] Create `src/pages/TaxExemptions.vue`
- [ ] Use `CnIndexPage` with `useListView` composable
- [ ] Display columns: exemptionType, jurisdiction, validFrom, validTo, appliesAutomatically
- [ ] Row actions: Edit, Delete, Duplicate
- [ ] Add button: Create new exemption
- [ ] Search/filter: by jurisdiction, exemption type, status (active/expired)
- [ ] Sidebar: show selected exemption details
- [ ] Inject exemption store from parent
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-015

**Acceptance:**
- [ ] Page loads exemptions list
- [ ] Add button opens create dialog
- [ ] Click row → sidebar shows details
- [ ] Edit button opens edit dialog
- [ ] Delete button removes exemption with confirmation
- [ ] Filter by jurisdiction works

---

### TASK-016 [Frontend] Tax Exemption Form Dialog
- [ ] Create `src/components/TaxExemptionDialog.vue`
- [ ] Use `CnFormDialog` (auto-generated from schema)
- [ ] Fields: exemptionType (select), jurisdiction (select), validFrom (date), validTo (date), conditions (JSON), appliesAutomatically (toggle)
- [ ] Validation: exemptionType required, jurisdiction required, validFrom ≤ validTo
- [ ] Save via `exemptionStore.save()`
- [ ] Show success/error notification
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-016

**Acceptance:**
- [ ] Create: fill form, save, exemption appears in list
- [ ] Edit: change field, save, updates reflected
- [ ] Validation: prevent save with validFrom > validTo
- [ ] Conditions: JSON editor for flexible structure

---

### TASK-017 [Frontend] Tax Detection Rules Index Page
- [ ] Create `src/pages/TaxDetectionRules.vue`
- [ ] Use `CnIndexPage` for list
- [ ] Display columns: priority, name, conditions (summary), outcome (summary), enabled
- [ ] Reorder button: drag/drop or priority input
- [ ] Test button: open modal, enter sample metadata, show match result
- [ ] Row actions: Edit, Delete, Duplicate, Enable/Disable
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-017

**Acceptance:**
- [ ] List shows all rules ordered by priority
- [ ] Reorder updates priority
- [ ] Test button shows match result
- [ ] Disable toggle affects rule availability
- [ ] Edit dialog shows complex conditions (country list, amount range)

---

### TASK-018 [Frontend] Tax Detection Rule Form Dialog
- [ ] Create `src/components/TaxDetectionRuleDialog.vue`
- [ ] Fields: priority (number), name (text), conditions (complex object), outcome (object), enabled (toggle)
- [ ] Conditions: dynamic form for supplierCountry (multi-select), customerCountry, transactionType (select), amount range, keywords
- [ ] Outcome: select vatStatus, enter rate %, select jurisdiction
- [ ] Validation: priority unique or allow, all conditions optional except outcome
- [ ] Save via store
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-018

**Acceptance:**
- [ ] Create rule with conditions
- [ ] Conditions include list fields (countries)
- [ ] Outcome shows derived values
- [ ] Save stores rule and updates priority list

---

### TASK-019 [Frontend] Tax Rates Index Page
- [ ] Create `src/pages/TaxRates.vue`
- [ ] Use `CnIndexPage` for list
- [ ] Display columns: country, type, rate (%), inclusive, validFrom, validTo
- [ ] Row actions: Edit, Delete, Copy to new date
- [ ] Filter: by country, type, status (active/expired)
- [ ] Add button: Create new rate
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-019

**Acceptance:**
- [ ] List shows NL 21%, 9%, 0%; DE 19%, 0%
- [ ] Filter by country shows only that country's rates
- [ ] Edit rate, set validTo to date, create new with new rate from next date
- [ ] Add rate with country, type, percentage, dates

---

### TASK-020 [Frontend] Tax Configuration Settings Page
- [ ] Create `src/pages/TaxConfiguration.vue`
- [ ] Sections:
  - [ ] Jurisdiction setup (select active jurisdictions)
  - [ ] Auto-detection (toggle on/off, link to detection rules)
  - [ ] Exemptions (toggle KOR, show applied exemptions)
  - [ ] VAT Recovery (toggle, set partial %, exclude categories)
  - [ ] Filing frequency (select quarterly/annual)
- [ ] Load config via API, display current values
- [ ] Save button: POST updated config
- [ ] Show success message
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-020

**Acceptance:**
- [ ] Load and display current config
- [ ] Toggle auto-detection, save, toggle persists
- [ ] Set recovery %, save, value persists
- [ ] Select jurisdictions, verify detection rules and exemptions updated

---

### TASK-021 [Frontend] Tax Detection Tester Component
- [ ] Create `src/components/TaxDetectionTester.vue`
- [ ] Modal with form: supplierCountry, customerCountry, transactionType, amount, customerType (optional)
- [ ] Call API `GET /api/tax-detection/determine`
- [ ] Display result: vatStatus, rate %, jurisdiction, rule applied, explanation
- [ ] Show previous tests (session storage)
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-021

**Acceptance:**
- [ ] Fill form, click Test
- [ ] Show result with rule name
- [ ] Test multiple scenarios, history shown
- [ ] Used in detection rules page for rule testing

---

### TASK-022 [Frontend] Tax Dashboard Widget
- [ ] Create `src/components/TaxDashboard.vue` (or integrate into app dashboard)
- [ ] Display KPI cards:
  - [ ] YTD VAT Payable (€)
  - [ ] YTD Inbound VAT (€)
  - [ ] VAT Recovery Rate (%)
  - [ ] Pending Returns (count)
- [ ] Chart: VAT by month (line chart)
- [ ] List: Recent exemptions applied
- [ ] Call API to fetch metrics
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-022

**Acceptance:**
- [ ] KPI cards show current values
- [ ] Chart updates as new transactions added
- [ ] Exemptions list shows recent applications
- [ ] Responsive on mobile (≥768px)

---

## Epic 4: Integration & Testing (13 points)

### TASK-023 [Integration] Integrate Tax Detection with Invoice Entry
- [ ] Modify `Invoice.vue` or `TransactionEntry.vue`
- [ ] After user enters supplier country, customer country, amount:
  - [ ] Call `TaxDetectionService::determine()`
  - [ ] Pre-populate VAT status and rate fields
  - [ ] Show detected rule name + explanation below field
  - [ ] Allow user to override (exemption button, manual VAT selection)
- [ ] Store original detected values for audit
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-023

**Acceptance:**
- [ ] Create invoice, enter countries/amount → VAT auto-detected
- [ ] Detected value shown to user for confirmation
- [ ] User can override with exemption
- [ ] Audit shows original + final VAT status

---

### TASK-024 [Testing] PHPUnit Tests for TaxDetectionService
- [ ] Create `tests/Unit/Service/TaxDetectionServiceTest.php`
- [ ] Test cases:
  - [ ] Domestic NL sale → Standard, 21%
  - [ ] EU B2B reverse charge → ReverseCharge, 0%
  - [ ] KOR exemption applied (below threshold)
  - [ ] KOR exemption not applied (above threshold)
  - [ ] No matching rule → default VAT status
  - [ ] Rule priority ordering
  - [ ] Expired exemption ignored
- [ ] Mock: ConfigurationService, ObjectService
- [ ] ≥8 tests, all passing
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-024

**Acceptance:**
- [ ] Run `composer test` — all tests pass
- [ ] Coverage: ≥80% of TaxDetectionService methods

---

### TASK-025 [Testing] PHPUnit Tests for TaxCalculationService
- [ ] Create `tests/Unit/Service/TaxCalculationServiceTest.php`
- [ ] Test cases:
  - [ ] Exclusive rate: €1,000 × 0.21 = €210 tax
  - [ ] Inclusive rate: €1,210 ÷ 1.21 = €1,000 net
  - [ ] Compound rate: VAT + sales tax correct
  - [ ] Zero rate: €1,000 × 0.0 = €0 tax
  - [ ] Decimal rate: €1,000 × 0.065 correct
  - [ ] Edge case: very small amount (€0.01)
- [ ] ≥6 tests, all passing
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-025

**Acceptance:**
- [ ] Run `composer test` — all tests pass
- [ ] Coverage: ≥85% of TaxCalculationService

---

### TASK-026 [Testing] PHPUnit Tests for TaxExemptionService
- [ ] Create `tests/Unit/Service/TaxExemptionServiceTest.php`
- [ ] Test cases:
  - [ ] checkExemptibility: KOR applies (under threshold)
  - [ ] checkExemptibility: KOR rejected (over threshold)
  - [ ] checkExemptibility: expired exemption ignored
  - [ ] applyExemption: sets vatStatus=Exempt, taxAmount=0
  - [ ] Audit trail: exemption logged with rule ID
- [ ] ≥5 tests
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-026

**Acceptance:**
- [ ] All tests pass
- [ ] Coverage: ≥80%

---

### TASK-027 [Testing] API Integration Tests (Newman/Postman)
- [ ] Create `tests/integration/tax-detection.postman_collection.json`
- [ ] Test requests:
  - [ ] POST /api/exemptions (create KOR rule)
  - [ ] GET /api/exemptions (list)
  - [ ] PUT /api/exemptions/{id} (update)
  - [ ] DELETE /api/exemptions/{id}
  - [ ] POST /api/detection-rules (create rule)
  - [ ] GET /api/detection-rules (list)
  - [ ] POST /api/detection-rules/{id}/test (test rule)
  - [ ] GET /api/tax-detection/determine (detect VAT)
  - [ ] POST /api/tax-calculation (calculate tax)
- [ ] Test assertions: response code 200, body schema valid
- [ ] Run via `postman collection run`
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-027

**Acceptance:**
- [ ] All requests return 200 or 201
- [ ] Create then read returns same data
- [ ] Update persists changes
- [ ] Delete removes resource

---

### TASK-028 [Testing] Browser Test: Tax Detection (Playwright)
- [ ] Create `tests/browser/tax-detection.spec.js` (Playwright)
- [ ] Scenario: REQ-TAX-001-A (Domestic Sale, Standard VAT)
  - [ ] Navigate to Invoice creation page
  - [ ] Fill: Supplier country = NL, Customer country = NL, Amount = €1,000
  - [ ] VERIFY: VAT Status pre-populated with "Standard"
  - [ ] VERIFY: Rate shows "21%"
  - [ ] Submit invoice
  - [ ] VERIFY: Invoice created with correct VAT
- [ ] Scenario: REQ-TAX-002-A (KOR Exemption)
  - [ ] Create invoice with KOR-eligible conditions
  - [ ] VERIFY: Exemption suggested or auto-applied
  - [ ] VERIFY: VAT Status = Exempt, Tax Amount = €0
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-028

**Acceptance:**
- [ ] Both scenarios pass
- [ ] No console errors
- [ ] Screenshots of result pages

---

## Epic 5: Documentation & Polish (0 points, non-blocking)

### TASK-029 [Docs] Create User Documentation
- [ ] Docs file: `docs/tax-detection.md`
- [ ] Sections:
  - [ ] Overview: How auto-detection works
  - [ ] Exemptions: When and how they apply
  - [ ] Configuration: Setting up for your jurisdiction
  - [ ] Screenshots: Tax rates, exemptions, detection rules pages
  - [ ] FAQ: Common issues (threshold exceeded, override exemption, etc.)
- [ ] Language: English (primary), Dutch (recommended)
- [ ] Screenshots from running app
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-029

**Acceptance:**
- [ ] Docs exist and are readable
- [ ] Screenshots current
- [ ] No broken links

---

### TASK-030 [Polish] Add i18n Keys for Tax Module
- [ ] Create `translationkeys/tax-detection.php` or add to main translation file
- [ ] Keys:
  - [ ] `tax.detection.title` → "VAT Auto-Detection"
  - [ ] `tax.exemptions.title` → "Tax Exemptions"
  - [ ] `tax.rates.title` → "Tax Rates"
  - [ ] `tax.configuration.title` → "Tax Configuration"
  - [ ] Form labels, button labels, notifications
- [ ] Provide English + Dutch translations
- [ ] Use `t(appName, 'key')` in all Vue templates
- [ ] Use `$this->l->t('key')` in PHP controllers
- [ ] @spec openspec/changes/tax-levy-management-other-t2/tasks.md#task-030

**Acceptance:**
- [ ] No hardcoded strings in Vue/PHP
- [ ] All i18n keys translated to EN + NL

---

## Deduplication Check

**Question:** Are there existing Shillinq or OpenRegister services that duplicate this work?

**Findings:**
- ✅ `ObjectService` — used for CRUD, no duplication
- ✅ `ConfigurationService::importFromApp()` — used for seed data, no duplication
- ✅ `AuditTrailService` — used for change tracking, automatic
- ❌ **No existing VAT determination service** — this is new, domain-specific
- ❌ **No existing exemption rule engine** — this is new, domain-specific
- ❌ **No existing tax calculation service** — formula-based, app-specific

**Conclusion:** Zero overlap with existing services. All tasks are appropriate for Shillinq app.

---

## Task Dependencies

```
TASK-001 ─────────────────────────┐
TASK-002 ──────────────────────┐  │
TASK-003 ────────────────────┐ │  │
TASK-004 ──────────────────┐ │ │  │
TASK-005 ┬─────────────────┼─┼─┼──┘
         │                 │ │ │
TASK-006 ├─┬─────────────┐ │ │ │
TASK-007 │ ├────────┐    │ │ │ │
TASK-008 │ │        │    │ │ │ │
TASK-009 │ │        │    │ │ │ │
TASK-010 │ │        │    │ │ │ │
TASK-011 │ │        │    │ │ │ │
TASK-012 │ │        ├────┼─┘ │ │
TASK-013 │ │        │    │   │ │
TASK-014 │ │        │    │   │ │
         │ │        │    │   │ │
TASK-015 ├─┘        │    │   │ │
TASK-016 │          │    │   │ │
TASK-017 │          │    │   │ │
TASK-018 │          │    │   │ │
TASK-019 │          │    │   │ │
TASK-020 │          │    │   │ │
TASK-021 │          │    │   │ │
TASK-022 │          │    │   │ │
         │          │    │   │ │
TASK-023 │          ├────┼───┘ │
         │          │    │     │
TASK-024 └──────────┼────┘     │
TASK-025 ───────────┤          │
TASK-026 ───────────┤          │
TASK-027 ───────────┤          │
TASK-028 ───────────┴────┐     │
                          │     │
TASK-029 ─────────────────┴─────┴──────→ FINAL
TASK-030 ─────────────────┴─────┴──────→ FINAL
```

**Critical path:**
1. TASK-001–005 (schema) → 4 days
2. TASK-006–009 (core services) → 6 days
3. TASK-010–014 (API) → 5 days
4. TASK-015–022 (frontend) → 5 days
5. TASK-023–028 (integration & testing) → 4 days
6. TASK-029–030 (docs) → 1 day

**Total:** ~25 days (~5 weeks with 1-2 day buffers per epic)

---

## Estimation Summary

| Epic | Tasks | Points | Days |
|------|-------|--------|------|
| 1: Schema & Data Layer | 5 | 20 | 4 |
| 2: Backend Services & API | 9 | 32 | 6 |
| 3: Frontend Pages & Components | 8 | 24 | 5 |
| 4: Integration & Testing | 6 | 13 | 4 |
| 5: Documentation & Polish | 2 | 0 | 1 |
| **TOTAL** | **30** | **89** | **20** |

---

## Success Criteria (All Required)

- [ ] All 30 tasks completed and marked done
- [ ] `composer check:strict` passes (PHP linting, type checking, tests)
- [ ] All PHPUnit tests pass (≥80% coverage per service)
- [ ] All API endpoints respond correctly (Postman tests pass)
- [ ] Browser tests pass (no console errors, scenarios verified)
- [ ] Documentation complete with screenshots
- [ ] i18n complete (EN + NL, no hardcoded strings)
- [ ] No merge conflicts with main
- [ ] PR review approved (code, security, deduplication checks)

---

**Status:** Ready for Sprint Planning  
**Next:** Assign tasks to developers, estimate velocity, schedule sprints

---

## Notes

- Tasks are modular and can be worked in parallel within epics (e.g., TASK-010, TASK-011, TASK-012 can be done simultaneously)
- All classes/methods MUST include `@spec` PHPDoc tags per ADR-003
- All public API endpoints MUST be documented in a REST API spec (OpenAPI 3.0) per ADR-002
- Seed data MUST use Dutch realistic examples (streets, postcodes, company names) per ADR-001
- All Vue components MUST be responsive (320px–1920px) and WCAG AA accessible per ADR-010

---

Generated: 2026-05-21  
Change: `tax-levy-management-other-t2`  
Spec Version: 1.0
