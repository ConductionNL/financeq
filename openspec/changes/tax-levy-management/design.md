# Design: Tax & Levy Management

**Change ID:** tax-levy-management  
**Last Updated:** 2026-05-21

## Architecture Overview

```
┌─────────────────────────────────────┐
│ UI Layer (Vue 2 + NL Design tokens) │
│  ├─ Tax Dashboard                    │
│  ├─ VAT Return Preparation           │
│  ├─ Tax Rate Management              │
│  ├─ Declaration Workflow             │
│  └─ Audit Trail & Compliance         │
├─────────────────────────────────────┤
│ Service Layer (Stateless Business)   │
│  ├─ TaxCalculationService            │
│  ├─ TaxDeclarationService            │
│  ├─ TaxReportingService              │
│  ├─ XBRLExportService                │
│  └─ TaxComplianceService             │
├─────────────────────────────────────┤
│ Data Layer (OpenRegister)            │
│  ├─ ExemptionCertificate (schema)    │
│  ├─ TaxConfiguration (schema)         │
│  ├─ TaxDeclaration (schema)           │
│  ├─ TaxExemption (schema)             │
│  ├─ TaxLot (schema)                   │
│  ├─ TaxRate (schema)                  │
│  ├─ TaxReturn (schema)                │
│  ├─ TaxableTransaction (schema)       │
│  ├─ VATReturn (schema)                │
│  ├─ XBRLInstance (schema)             │
│  └─ XBRLTaxonomy (schema)             │
└─────────────────────────────────────┘
```

## Data Model & Schemas

All schemas use **schema.org** vocabulary + OpenRegister relations. Stored as `lib/Settings/tax-levy-management_register.json`.

### 1. ExemptionCertificate

**Base Type:** `schema:DigitalDocument`

Tax exemption credential (research, export, environmental, humanitarian). Stores certificate metadata, validity, and linked exemptions for workflow automation.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "ExemptionCertificate",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier" },
    "certificateNumber": { "type": "string", "description": "Official certificate ID from issuing authority" },
    "certificateType": { "type": "string", "enum": ["research", "export", "environmental", "humanitarian", "innovation", "vat-reverse", "other"], "description": "Type of exemption certificate" },
    "issueDate": { "type": "string", "format": "date", "description": "Certificate issuance date" },
    "expiryDate": { "type": "string", "format": "date", "description": "Expiration date; null = perpetual" },
    "exemptionReason": { "type": "string", "description": "Legal basis or reason code" },
    "documentURL": { "type": "string", "format": "uri", "description": "Link to official document or scan" }
  },
  "required": ["certificateNumber", "certificateType", "issueDate", "exemptionReason"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `taxDeclarations` → TaxDeclaration (many-to-many)

---

### 2. TaxConfiguration

**Base Type:** `schema:Thing`

System-wide tax settings, rules, and thresholds for a specific jurisdiction and tax year.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxConfiguration",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "configId": { "type": "string", "description": "Unique configuration identifier" },
    "taxYear": { "type": "integer", "description": "Tax year (e.g., 2026)" },
    "jurisdiction": { "type": "string", "description": "Tax jurisdiction code (NL, UK, US, etc.)" },
    "effectiveDate": { "type": "string", "format": "date-time", "description": "When this configuration becomes effective" },
    "description": { "type": "string", "description": "Configuration description and compliance notes" }
  },
  "required": ["configId", "taxYear", "jurisdiction", "effectiveDate"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `taxRates` → TaxRate (one-to-many)

---

### 3. TaxDeclaration

**Base Type:** `schema:Report`

Primary tax declaration submission (VAT, BCF, exemptions). Aggregates tax lots and manages workflow from draft to submission.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxDeclaration",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "declarationType": { "type": "string", "enum": ["BCF", "VAT-NL", "ICP", "VAT-UK", "GST", "Other"], "description": "Dutch tax form type" },
    "taxYear": { "type": "integer", "description": "Calendar or fiscal year (e.g., 2025)" },
    "declarationStatus": { "type": "string", "enum": ["draft", "approved", "submitted", "acknowledged", "rejected"], "description": "Workflow status" },
    "totalTaxAmount": { "type": "number", "description": "Net tax liability or credit (in EUR)" },
    "submissionDate": { "type": "string", "format": "date", "description": "Actual submission timestamp to authorities" },
    "businessTaxID": { "type": "string", "description": "Taxpayer BSN/KVK or VAT ID" }
  },
  "required": ["declarationType", "taxYear", "declarationStatus", "totalTaxAmount", "businessTaxID"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `taxLots` → TaxLot (one-to-many)
- `exemptionCertificates` → ExemptionCertificate (many-to-many)

---

### 4. TaxExemption

**Base Type:** `schema:Offer`

Reusable exemption rule or policy: qualifies transactions or amounts as exempt. Linked to certificates and applied during tax lot calculation.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxExemption",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "exemptionCode": { "type": "string", "description": "Statutory code (e.g., 021 for research)" },
    "exemptionName": { "type": "string", "description": "Display name (e.g., 'Research & Development Exemption')" },
    "applicableTaxTypes": { "type": "array", "items": { "type": "string" }, "description": "List of tax categories (VAT, profit, withholding, etc.)" },
    "effectiveFrom": { "type": "string", "format": "date", "description": "Start of exemption period" },
    "effectiveUntil": { "type": "string", "format": "date", "description": "End of exemption period; null = ongoing" }
  },
  "required": ["exemptionCode", "exemptionName", "applicableTaxTypes", "effectiveFrom"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `exemptionCertificate` → ExemptionCertificate (many-to-one)

---

### 5. TaxLot

**Base Type:** `schema:MonetaryAmount`

Individual tax line item: single transaction or aggregate category contributing to declaration.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxLot",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "lotNumber": { "type": "string", "description": "Unique identifier within declaration (e.g., VAT-001)" },
    "taxCategory": { "type": "string", "description": "VAT standard/reverse/zero, profit, withholding, excise, etc." },
    "amount": { "type": "number", "description": "Gross or net tax amount" },
    "currency": { "type": "string", "default": "EUR", "description": "Currency code" },
    "transactionDate": { "type": "string", "format": "date", "description": "Date of underlying transaction or period start" },
    "description": { "type": "string", "description": "Narrative or reference (e.g., invoice number, period)" }
  },
  "required": ["lotNumber", "taxCategory", "amount", "currency", "transactionDate"]
}
```

**Relations:**
- `taxDeclaration` → TaxDeclaration (many-to-one)
- `bankAccount` → BankAccount (many-to-one)

---

### 6. TaxRate

**Base Type:** `schema:Thing`

Individual tax rate rules for income, sales, VAT, capital gains, or other tax types with effective date management.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxRate",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "rateId": { "type": "string", "description": "Unique rate identifier" },
    "rateType": { "type": "string", "enum": ["income", "sales", "vat", "capital_gains", "tds", "gst", "other"], "description": "Type of tax" },
    "percentage": { "type": "number", "description": "Tax rate as percentage (e.g., 21 for 21%)" },
    "effectiveDate": { "type": "string", "format": "date-time", "description": "When this rate becomes effective" },
    "expiryDate": { "type": "string", "format": "date-time", "description": "When this rate expires or is superseded" }
  },
  "required": ["rateId", "rateType", "percentage", "effectiveDate"]
}
```

**Relations:**
- `taxConfiguration` → TaxConfiguration (many-to-one)
- `product` → Product (many-to-one)

---

### 7. TaxReturn

**Base Type:** `schema:Thing`

A formal tax return filing for income, VAT, or other tax obligations with workflow management and compliance tracking.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxReturn",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "returnId": { "type": "string", "description": "Unique return identifier" },
    "filingPeriod": { "type": "string", "description": "Period covered (e.g., Q1 2026)" },
    "taxYear": { "type": "integer", "description": "Calendar year for tax reporting" },
    "totalIncome": { "type": "number", "description": "Total income for the period" },
    "totalExpenses": { "type": "number", "description": "Total deductible expenses" },
    "status": { "type": "string", "enum": ["draft", "submitted", "approved", "rejected"], "description": "Current status" },
    "filedDate": { "type": "string", "format": "date-time", "description": "When the return was submitted" }
  },
  "required": ["returnId", "filingPeriod", "taxYear", "status"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `taxConfiguration` → TaxConfiguration (many-to-one)

---

### 8. TaxableTransaction

**Base Type:** `schema:Thing`

Business transaction classified and tracked for tax reporting, audit trail, and automated tax calculation.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "TaxableTransaction",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "transactionId": { "type": "string", "description": "Unique transaction identifier" },
    "amount": { "type": "number", "description": "Transaction amount" },
    "transactionDate": { "type": "string", "format": "date-time", "description": "Date of the transaction" },
    "taxCategory": { "type": "string", "description": "Tax classification for reporting" },
    "taxRate": { "type": "number", "description": "Applied tax rate percentage" },
    "description": { "type": "string", "description": "Transaction description for audit trail" }
  },
  "required": ["transactionId", "amount", "transactionDate", "taxCategory"]
}
```

**Relations:**
- `taxReturn` → TaxReturn (many-to-one)
- `receipt` → Receipt (many-to-one)
- `payment` → Payment (many-to-one)

---

### 9. VATReturn

**Base Type:** `schema:Thing`

VAT-specific tax return showing collected VAT, paid VAT, and net amount due for MTD compliance and electronic filing.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "VATReturn",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "vatReturnId": { "type": "string", "description": "Unique VAT return identifier" },
    "reportingPeriod": { "type": "string", "enum": ["monthly", "quarterly", "annually"], "description": "VAT reporting period" },
    "collectedVAT": { "type": "number", "description": "VAT collected from customers (EUR)" },
    "paidVAT": { "type": "number", "description": "VAT paid on business purchases (EUR)" },
    "netAmount": { "type": "number", "description": "Net VAT payable (positive) or refundable (negative)" },
    "status": { "type": "string", "enum": ["draft", "submitted", "approved", "rejected"], "description": "Workflow status" },
    "submissionDate": { "type": "string", "format": "date-time", "description": "When VAT return was submitted" }
  },
  "required": ["vatReturnId", "reportingPeriod", "collectedVAT", "paidVAT", "netAmount", "status"]
}
```

**Relations:**
- `organization` → Organization (many-to-one)
- `taxReturn` → TaxReturn (many-to-one)

---

### 10. XBRLInstance

**Base Type:** `schema:DigitalDocument`

Structured XBRL instance document for taxonomies (NTA7, SBR-NT). Contains facts, contexts, and dimensions for standardized digital reporting to Dutch authorities.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "XBRLInstance",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "taxonomyVersion": { "type": "string", "description": "e.g., NTA7-2025, SBR-NT-2025" },
    "instanceID": { "type": "string", "description": "Unique document identifier" },
    "reportingPeriod": { "type": "string", "format": "date", "description": "ISO date range (e.g., 2025-01-01/2025-12-31)" },
    "factCount": { "type": "integer", "description": "Number of XBRL facts in instance" },
    "encodingFormat": { "type": "string", "enum": ["application/xbrl+xml", "application/xbrl+json"], "description": "XBRL encoding format" },
    "validationStatus": { "type": "string", "enum": ["valid", "invalid", "warned", "unvalidated"], "description": "XBRL validation status" }
  },
  "required": ["taxonomyVersion", "instanceID", "reportingPeriod", "encodingFormat", "validationStatus"]
}
```

**Relations:**
- `taxDeclaration` → TaxDeclaration (many-to-one)

---

### 11. XBRLTaxonomy

**Base Type:** `schema:CreativeWork`

XBRL taxonomy definitions for structured tax reporting, compliance, and regulatory filing.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "XBRLTaxonomy",
  "$comment": "@spec openspec/changes/tax-levy-management/tasks.md",
  "properties": {
    "id": { "type": "string" },
    "taxonomyId": { "type": "string", "description": "Unique taxonomy identifier" },
    "version": { "type": "string", "description": "Taxonomy version number (e.g., 2025.1)" },
    "effectiveDate": { "type": "string", "format": "date-time", "description": "When taxonomy becomes effective" },
    "namespace": { "type": "string", "format": "uri", "description": "XML namespace URI for the taxonomy" },
    "elements": { "type": "array", "items": { "type": "string" }, "description": "List of XBRL element definitions and mappings" }
  },
  "required": ["taxonomyId", "version", "effectiveDate", "namespace"]
}
```

**Relations:**
- `taxReturns` → TaxReturn (one-to-many)

---

## Seed Data (Example Objects)

### ExemptionCertificate (Research & Development)
```json
{
  "register": "tax-levy-management",
  "schema": "ExemptionCertificate",
  "slug": "cert-rnd-2026-001",
  "certificateNumber": "RND/2026/001",
  "certificateType": "research",
  "issueDate": "2026-01-15",
  "expiryDate": "2027-01-15",
  "exemptionReason": "R&D exemption under Dutch Innovation Act",
  "documentURL": "https://example.com/certs/rnd-2026-001.pdf",
  "organization": { "register": "core", "schema": "Organization", "objectId": "org-acme" }
}
```

### TaxConfiguration (Netherlands 2026)
```json
{
  "register": "tax-levy-management",
  "schema": "TaxConfiguration",
  "slug": "config-nl-2026",
  "configId": "NL-2026",
  "taxYear": 2026,
  "jurisdiction": "NL",
  "effectiveDate": "2026-01-01T00:00:00Z",
  "description": "Dutch tax configuration for 2026 including VAT standard rate 21%, reduced rate 9%, zero rate 0%",
  "organization": { "register": "core", "schema": "Organization", "objectId": "org-acme" }
}
```

### TaxRate (VAT 21% Standard Rate)
```json
{
  "register": "tax-levy-management",
  "schema": "TaxRate",
  "slug": "rate-vat-nl-21",
  "rateId": "VAT-NL-21",
  "rateType": "vat",
  "percentage": 21.0,
  "effectiveDate": "2026-01-01T00:00:00Z",
  "expiryDate": null,
  "taxConfiguration": { "register": "tax-levy-management", "schema": "TaxConfiguration", "objectId": "config-nl-2026" }
}
```

### TaxReturn (Q1 2026 VAT Return)
```json
{
  "register": "tax-levy-management",
  "schema": "TaxReturn",
  "slug": "return-vat-2026-q1",
  "returnId": "VAT-2026-Q1",
  "filingPeriod": "Q1 2026",
  "taxYear": 2026,
  "totalIncome": 45000,
  "totalExpenses": 12000,
  "status": "draft",
  "filedDate": null,
  "organization": { "register": "core", "schema": "Organization", "objectId": "org-acme" },
  "taxConfiguration": { "register": "tax-levy-management", "schema": "TaxConfiguration", "objectId": "config-nl-2026" }
}
```

### TaxableTransaction (Invoice #INV-001)
```json
{
  "register": "tax-levy-management",
  "schema": "TaxableTransaction",
  "slug": "txn-inv-2026-001",
  "transactionId": "INV-2026-001",
  "amount": 5000.00,
  "transactionDate": "2026-03-15T14:30:00Z",
  "taxCategory": "VAT-standard",
  "taxRate": 21.0,
  "description": "Sales invoice for software development services",
  "taxReturn": { "register": "tax-levy-management", "schema": "TaxReturn", "objectId": "return-vat-2026-q1" }
}
```

### VATReturn (Q1 2026)
```json
{
  "register": "tax-levy-management",
  "schema": "VATReturn",
  "slug": "vat-return-2026-q1",
  "vatReturnId": "VAT-2026-Q1",
  "reportingPeriod": "quarterly",
  "collectedVAT": 9450.00,
  "paidVAT": 2100.00,
  "netAmount": 7350.00,
  "status": "draft",
  "submissionDate": null,
  "organization": { "register": "core", "schema": "Organization", "objectId": "org-acme" },
  "taxReturn": { "register": "tax-levy-management", "schema": "TaxReturn", "objectId": "return-vat-2026-q1" }
}
```

### XBRLTaxonomy (NTA7 2026)
```json
{
  "register": "tax-levy-management",
  "schema": "XBRLTaxonomy",
  "slug": "taxonomy-nta7-2026",
  "taxonomyId": "NTA7-2026",
  "version": "2026.1",
  "effectiveDate": "2026-01-01T00:00:00Z",
  "namespace": "http://www.xbrl.nl/nl/taxonomy/nta7/2026",
  "elements": ["Assets", "Liabilities", "Equity", "Revenue", "Expenses", "TaxableIncome"]
}
```

---

## Integration Points

### External Systems
1. **Dutch Tax Authority (Belastingdienst)** — XBRL/SBR submission for VAT, BCF, corporate tax (VPB)
2. **Nextcloud Accounts** — Sync employee/user data for jaaropgave (annual statements)
3. **Banking Systems** — Bank reconciliation and transaction import
4. **ERP/Accounting Systems** — GL account mapping and transaction import
5. **OpenRegister** — Core data persistence, audit trails, file attachments

### APIs & Webhooks
- **VAT Return Submission** → POST `/api/tax-declarations/{id}/submit` (triggers XBRL export + Belastingdienst filing)
- **Tax Rate Changes** → Webhook on TaxRate create/update (notifies billing module)
- **Exemption Certificate Expiry** — Scheduled job to flag expiring certificates (30-day warning)

### Reuse Analysis

**OpenRegister Services Used:**
- `ObjectService` — CRUD for all 11 tax entities
- `AuditTrailService` — Audit trail for VAT/tax transactions (mandatory for tax defense)
- `FileService` — Certificate documents, XBRL submissions, audit reports
- `ArchivalService` — Legal hold for tax records (7-year retention)
- `IndexService` + `FacetSidebar` — Search and filter tax returns by period, type, status
- `CnFormDialog` — Schema-driven create/edit for tax returns, declarations
- `CnDetailPage` — Detail view for tax return with related lots, rates, exemptions
- `CnDataTable` — List VAT returns with sorting/pagination
- `CnDashboardPage` — Tax dashboard with KPI cards (pending returns, collected VAT, tax liability)

**NO DUPLICATION:** Tax calculation, XBRL serialization, and compliance validation are domain-specific and not duplicated in OpenRegister.

---

## Reuse Analysis Checklist

- [x] Searched OpenRegister for existing Tax/VAT/Declaration entities — none exist
- [x] Confirmed ObjectService + AuditTrailService cover CRUD + audit needs
- [x] Confirmed no overlap with Finance/Bookkeeping module schemas
- [x] Confirmed FileService handles certificate/document storage (no custom upload)
- [x] Confirmed CnIndexPage + CnDetailPage cover UI layout (no custom components)
- [x] Domain-specific: XBRL serialization, tax calculation rules, compliance validation

---

## UI Mockups & Workflows

### VAT Return Preparation Workflow
1. **Dashboard** → "Prepare VAT Return" → Select period (Q1, Q2, etc.)
2. **Auto-aggregate** taxable transactions grouped by VAT category (standard 21%, reduced 9%, zero 0%)
3. **Review** collected VAT (sales), paid VAT (expenses), net liability
4. **Optionally apply** exemption certificates if eligible transactions exist
5. **Approve** draft return → change status to "approved"
6. **Submit** → triggers XBRL export + Belastingdienst API submission
7. **Acknowledge** → receipt of acknowledgment from authorities

### Tax Rate Management (Municipal)
1. **Admin** → Tax Rates → + Create new proposal
2. **Finance head** reviews proposal with revenue projections
3. **Council** votes on rates (record date + raadsbesluit number)
4. **Compliance Officer** → Publish rates → effective 2026-01-01
5. **All billing/tax modules** automatically use published rates

---

## Performance & Scalability

- VAT return aggregation caches transaction counts by category (Redis)
- XBRL generation uses background job queue for large submissions (>10K facts)
- Audit trail indexed on transaction date + tax category for reporting queries
- Tax rate lookups cached per fiscal year (2 cache misses/year on average)

---

## Compliance & Audit

- **Dutch Law:** BBV (Boekhoudwet), BTW-richtlijn (VAT), CAO Gemeenten (payroll)
- **Nextcloud:** GDPR data subject access via AuditTrailService, retention policies via ArchivalService
- **XBRL:** NTA7 (Dutch standard GL/tax), SBR-NT (multi-country submissions)
- **Accessibility:** All tax forms WCAG AA compliant (NL Design tokens only, no hardcoded colors)
