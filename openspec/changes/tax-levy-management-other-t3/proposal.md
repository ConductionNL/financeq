---
kind: config
depends_on: []
chain:
  - tax-levy-management-other-t3
---

# Tax & Levy Management — Shillinq — Other T3

## Executive Summary

Shillinq (complete open-source business administration suite for Nextcloud) requires tax and levy management capabilities to serve freelancers, SMBs, and corporations. This spec implements declarative tax configuration, VAT calculation rules, and compliance reporting foundations via OpenRegister schema extensions and OpenAPI register templates.

**Market demand:** 2 (1% competitor coverage) across 31 feature slots
**Scope:** Schema-only (config kind); no service classes
**Timeline:** Single spec (no chaining needed)

---

## Market Features

Demand distribution and feature clustering from market research:

### High-frequency features (demand: 2)
- Auto-calculate VAT
- Tax Administration
- Corporate income tax (VPB) declaration support
- Automatic VAT calculation with configurable tax codes and rates
- Suppletie-aangifte (supplementary VAT return) for corrections
- Configurable VAT schemes including kleineondernemersregeling (KOR)
- Tax Planner tool for forecasting annual tax liability
- Multi-line invoices with discounts, taxes, and subtotal sections
- Tax-ready income and expense categorization for year-end filing
- VAT Reclaim
- Compound Taxes
- Withholding Tax
- DNI/NIF Validation
- CNPJ/CPF Validation
- Invoice Tax Rounding
- RJ Taxonomy (Richtlijnen)
- Agro Taxonomy (AGT) Support

### Emerging features (demand: 1 or unknown)
- vat-filing
- Estimate tax liability
- Calculate VAT amounts
- Handle VAT on imports
- Generate VAT reports
- Reject application with formal motivation
- Deactivate items linked to expired contracts
- Generate VAT return form
- Distinguish BCF from regular VAT
- Track voorlopige aanslag payments against actual tax liability
- Prepare corporate tax return (VPB-aangifte)
- Prepare income tax return (IB-aangifte) for sole proprietor
- Pillar Two GloBE minimum tax calculation
- Split private/business portion

---

## User Stories (Generated from Features)

### Story 1: Freelancer tracks quarterly VAT liability
**As a** freelancer operating under KOR (small business exemption)
**I want to** automatically calculate VAT on invoices using Dutch rules
**So that** I can forecast my quarterly tax payment and file VAT returns accurately

**Acceptance Criteria:**
- GIVEN a freelancer with KOR-exempt status configured
- WHEN they create an invoice with taxable items
- THEN VAT is NOT calculated (KOR excludes VAT)
- AND the invoice notes KOR exemption status

### Story 2: SMB owner prepares annual income tax filing
**As a** SMB accountant
**I want to** categorize all transactions by tax-relevant categories (revenue, expenses, capital, depreciation)
**So that** we can generate a compliant IB-aangifte (income tax declaration) for year-end filing

**Acceptance Criteria:**
- GIVEN all transactions in a fiscal year
- WHEN grouping by category
- THEN system shows: revenue sum, deductible expenses, capital gains/losses, depreciation
- AND export format matches Dutch tax authorities' expected structure

### Story 3: Corporate treasurer reconciles tax payments
**As a** corporation's financial controller
**I want to** track voorlopige aanslag (provisional tax payment) instalments against actual calculated liability
**So that** we can reconcile and file the final VPB-aangifte (corporate income tax return) accurately

**Acceptance Criteria:**
- GIVEN monthly/quarterly provisional tax payments recorded
- WHEN calculating actual liability at year-end
- THEN system compares provisional-paid vs actual-owed
- AND shows balance (refund or additional payment due)

### Story 4: Multi-currency business applies compound taxes
**As a** goods-importer with EU and non-EU suppliers
**I want to** handle compound tax calculations where one tax base depends on another
**So that** imported goods are priced correctly with all applicable levies

**Acceptance Criteria:**
- GIVEN an imported item with import duty → VAT-on-duty scenario
- WHEN calculating final price
- THEN system applies: import duty on base → VAT on (base + duty)
- AND amount fields show calculation chain with intermediate steps

### Story 5: Supplier withholding tax compliance
**As a** freelancer receiving payments from corporate clients
**I want to** track withholding tax obligations by client (some clients deduct tax at source)
**So that** I can reconcile my net income and verify correctness of withheld amounts

**Acceptance Criteria:**
- GIVEN a supplier payment with withholding tax applied
- WHEN recording the transaction
- THEN system records: gross amount, withholding rate, net amount
- AND provides summary showing total withheld YTD for reconciliation

---

## Stakeholders (Generated from Features & ADRs)

### Shillinq Product Owner
**Responsibilities:** Feature prioritization, market positioning, competitive feature mapping
**Goals:** Close gaps with competitors; support Dutch government compliance (BBV, IV3)
**Constraints:** Limited budget for implementation; must focus on high-demand features (demand: 2)

### Dutch Freelancer (KOR-eligible)
**Responsibilities:** File quarterly VAT returns (or claim exemption); track income for tax filing
**Goals:** Simple, accurate VAT tracking without accounting overhead
**Constraints:** Limited accounting knowledge; works under KOR exemption rules

### SMB Accountant
**Responsibilities:** Prepare year-end tax filings (IB/VPB-aangifte); maintain tax-ready books
**Goals:** Automated categorization; export-ready formats for tax authority submission
**Constraints:** Must comply with Dutch taxation rules; must audit trail all transactions

### Corporate Tax Controller
**Responsibilities:** Manage provisional tax payments; reconcile actual liability; file corporate returns
**Goals:** Accurate forecasting; quick month-end/quarter-end close
**Constraints:** Complex multi-currency scenarios; withholding tax from customers and vendors

### Tax Authority Liaison (Dutch)
**Responsibilities:** Validate that exports match expected formats (IV3, Elias, BBV)
**Goals:** Automated, compliant data formats
**Constraints:** Schema must match official Dutch tax reporting standards

---

## Reuse Analysis

This spec builds on existing OpenRegister capabilities:

| Capability | How Reused | Reference |
|---|---|---|
| Schema-driven declarative calculations | VAT amount calculation; compound tax order-of-operations | `x-openregister-calculations` (ADR-031) |
| Aggregations for summaries | VAT reclaim totals; withholding tax YTD; provisional vs actual reconciliation | `x-openregister-aggregations` (ADR-031) |
| Relations for cross-references | Tax code → rate tables; invoice → tax allocation; supplier → withholding configuration | `x-openregister-relations` (ADR-001) |
| OpenAPI register templates | Tax code library (Dutch standard), VAT scheme definitions, withholding tax rules | Register templates (ADR-001) |
| Existing Invoice/InvoiceLine entities | Multi-line invoice tax allocation; tax rounding per line | Already in OpenRegister |
| Existing TaxRate entity | Base for VAT rate configuration; regional variants | Already in OpenRegister |
| Existing Transaction entity | Foundation for tax categorization by GL account | Already in OpenRegister |
| Field-level RBAC | Restrict tax configuration to admins; allow users to view tax amounts only | `PropertyRbacHandler` (ADR-005) |

No custom service classes required (config-only spec per ADR-032).

---

## Deduplication Check

**Finding:** No overlap with existing OpenRegister services.

- `ConfigurationService` handles import of tax code libraries (reuse)
- `SchemaService` manages register & schema validation (reuse)
- `ObjectService` handles CRUD on tax configurations (reuse)
- No app-local service classes needed; all business logic declared in schema
- Tax calculation rules → `x-openregister-calculations` metadata (not imperative code)

---

## Success Criteria

1. **Schemas defined:** TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration with required fields and relations
2. **Register template created:** `lib/Settings/shillinq_tax_register.json` with seed data (3-5 tax codes, 2-3 VAT schemes)
3. **Integration test:** Verify tax calculation for single-line invoice under KOR exemption
4. **Documentation:** Seed data in design.md; schema definitions with schema.org references
5. **No regressions:** Existing Invoice, TaxRate, Transaction entities unchanged; backward compatible

---

## Implementation Notes

- Tax calculation order matters (import duty → VAT on duty). Declarative `x-openregister-calculations` with `dependsOn` ordering.
- Withholding tax tracking requires supplier-level configuration (link TaxScheme or Supplier entity to withholding rules).
- Dutch government mappings (IV3, BBV, Elias) are outside this spec scope; stored as annotations in schema for future automated export.
- Multi-currency support leverages existing MonetaryAmount entity; no new tax-specific currency logic.

---

## Schedule

**Phase:** Batch 3 (config-only, no chaining)  
**Builder:** Sonnet (config specs at 30-80 turns)  
**Review gates:** Schema validation + ADR-031 declarative-fit + integration test coverage (3 gates, fast)
