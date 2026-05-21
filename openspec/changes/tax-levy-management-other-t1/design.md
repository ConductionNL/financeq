---
title: Tax & Levy Management — Design
app: shillinq
change: tax-levy-management-other-t1
---

# Design: Tax & Levy Management

## Architecture Overview

The tax management module extends Shillinq's OpenRegister with:

1. **Tax Schemas** — TaxConfiguration, TaxRate, TaxDeclaration, TaxReturn, TaxableTransaction, ExemptionCertificate
2. **Tax Workflows** — Lifecycle for filing (draft → submitted → accepted → objection → resolved)
3. **Calculations & Aggregations** — Real-time VAT/income tax totals, tax liability estimates
4. **Integrations** — External tax engines (AvaTax, Belastingdienst, Peppol), receipt OCR (Scan & Herken)
5. **Reporting** — VAT returns, IV3, BCF, CSRD/ESRS, capital gains tracking

## Data Model

All schemas use schema.org vocabulary where equivalent exists, with Dutch government extensions.

### TaxConfiguration

Defines tax rules and rates for a business.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxConfiguration",
    "slug": "tax-config-default"
  },
  "name": "Tax Configuration",
  "description": "VAT rates, deduction rules, exemption categories",
  "businessJurisdiction": "NL",
  "vatNumber": "NL123456789B12",
  "fiscalYear": 2026,
  "defaultVatRate": 21,
  "reducedVatRates": [6, 9],
  "vatScheme": "standard",
  "smallBusinessExemption": false,
  "reverseChargeEnabled": true,
  "icpDeclarationRequired": true,
  "createdAt": "2026-01-15T10:00:00Z",
  "modifiedAt": "2026-01-15T10:00:00Z"
}
```

### TaxRate

Individual tax rate per item/service.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxRate",
    "slug": "tax-rate-standard-21"
  },
  "name": "Standard VAT 21%",
  "itemType": "service",
  "taxType": "VAT",
  "rate": 21,
  "jurisdiction": "NL",
  "validFrom": "2026-01-01",
  "validUntil": null,
  "applicableToCategories": ["professional-services", "goods"],
  "reverseChargeApplies": false,
  "exemptionCode": null,
  "description": "Standard Dutch VAT rate for most goods and services"
}
```

### TaxableTransaction

Marks a transaction (Invoice, PurchaseOrder, Expense) as tax-relevant.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxableTransaction",
    "slug": "taxable-inv-2026-001"
  },
  "transactionId": "invoice-2026-001",
  "transactionType": "invoice",
  "amount": 1200.00,
  "currency": "EUR",
  "taxableAmount": 1200.00,
  "vatAmount": 252.00,
  "taxRate": "21%",
  "taxCategory": "professional-services",
  "jurisdiction": "NL",
  "transactionDate": "2026-05-15",
  "parentObjectId": "inv-2026-001",
  "taxReturnId": null,
  "declarationId": null,
  "ocrExtracted": false,
  "ocrConfidence": 0,
  "createdAt": "2026-05-15T14:30:00Z"
}
```

### TaxDeclaration

A filing (VAT, IV3, BCF, ESG) prepared for submission.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxDeclaration",
    "slug": "vat-decl-2026-q2"
  },
  "name": "VAT Return Q2 2026",
  "declarationType": "VAT",
  "jurisdiction": "NL",
  "periodStart": "2026-04-01",
  "periodEnd": "2026-06-30",
  "status": "draft",
  "grossSales": 15000.00,
  "totalVatCollected": 3150.00,
  "totalVatDeductible": 2100.00,
  "netVatOwed": 1050.00,
  "submissionDeadline": "2026-08-20",
  "submittedAt": null,
  "submissionStatus": null,
  "filingReference": null,
  "createdAt": "2026-07-01T09:00:00Z",
  "modifiedAt": "2026-07-01T09:00:00Z"
}
```

### TaxReturn

Historical record of filed/accepted tax returns.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxReturn",
    "slug": "tax-return-2025"
  },
  "taxYear": 2025,
  "returnType": "annual-income-tax",
  "jurisdiction": "NL",
  "grossIncome": 45000.00,
  "deductibleExpenses": 8000.00,
  "taxableIncome": 37000.00,
  "incomeTaxOwed": 9250.00,
  "filingStatus": "accepted",
  "filingDate": "2026-03-31",
  "acceptanceDate": "2026-04-15",
  "assessmentReferenceNumber": "2026/1234567",
  "supportingDocumentsCount": 12,
  "archiveDate": "2026-04-15"
}
```

### ExemptionCertificate

Tax exemption documentation (e.g., EU reverse charge, small business exemption).

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "ExemptionCertificate",
    "slug": "cert-kor-2026"
  },
  "name": "Kleine Ondernemersregeling (KOR) 2026",
  "certificateType": "small-business-exemption",
  "jurisdiction": "NL",
  "validFrom": "2026-01-01",
  "validUntil": "2026-12-31",
  "exemptionCode": "KOR",
  "annualRevenueThreshold": 20000.00,
  "currentRevenue": 18500.00,
  "status": "active",
  "documentReference": "file-id-xyz",
  "createdAt": "2025-12-01",
  "expiresAt": "2026-12-31"
}
```

## Seed Data

### Example 1: Consultancy Firm (Amsterdam)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxConfiguration",
    "slug": "tax-config-consultancy-ams"
  },
  "name": "Adviesbureau Amsterdam",
  "businessJurisdiction": "NL",
  "vatNumber": "NL987654321B01",
  "fiscalYear": 2026,
  "defaultVatRate": 21,
  "reducedVatRates": [],
  "vatScheme": "standard",
  "smallBusinessExemption": false,
  "reverseChargeEnabled": true,
  "icpDeclarationRequired": true
}

{
  "@self": {
    "register": "shillinq",
    "schema": "TaxableTransaction",
    "slug": "taxable-inv-adv-001"
  },
  "transactionId": "invoice-consulting-001",
  "transactionType": "invoice",
  "amount": 5000.00,
  "taxableAmount": 5000.00,
  "vatAmount": 1050.00,
  "taxRate": "21%",
  "taxCategory": "professional-services",
  "jurisdiction": "NL",
  "transactionDate": "2026-05-10"
}

{
  "@self": {
    "register": "shillinq",
    "schema": "TaxDeclaration",
    "slug": "vat-decl-adv-2026-q2"
  },
  "name": "VAT Return Q2 2026 - Adviesbureau",
  "declarationType": "VAT",
  "jurisdiction": "NL",
  "periodStart": "2026-04-01",
  "periodEnd": "2026-06-30",
  "status": "submitted",
  "grossSales": 18000.00,
  "totalVatCollected": 3780.00,
  "totalVatDeductible": 500.00,
  "netVatOwed": 3280.00,
  "submissionDeadline": "2026-08-20",
  "submittedAt": "2026-07-15T11:00:00Z",
  "submissionStatus": "accepted",
  "filingReference": "VAT-NL-2026-Q2-98765"
}
```

### Example 2: Freelancer (Rotterdam) - KOR Eligible

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxConfiguration",
    "slug": "tax-config-freelancer-rdam"
  },
  "name": "Freelance Developer",
  "businessJurisdiction": "NL",
  "vatNumber": "NL456789123B02",
  "fiscalYear": 2026,
  "defaultVatRate": 0,
  "reducedVatRates": [],
  "vatScheme": "kor",
  "smallBusinessExemption": true,
  "reverseChargeEnabled": false,
  "icpDeclarationRequired": false
}

{
  "@self": {
    "register": "shillinq",
    "schema": "ExemptionCertificate",
    "slug": "cert-kor-freelance-2026"
  },
  "name": "KOR Exemption - Freelance Developer 2026",
  "certificateType": "small-business-exemption",
  "jurisdiction": "NL",
  "validFrom": "2026-01-01",
  "validUntil": "2026-12-31",
  "exemptionCode": "KOR",
  "annualRevenueThreshold": 20000.00,
  "currentRevenue": 12000.00,
  "status": "active"
}

{
  "@self": {
    "register": "shillinq",
    "schema": "TaxableTransaction",
    "slug": "taxable-inv-freelance-001"
  },
  "transactionId": "invoice-dev-2026-001",
  "transactionType": "invoice",
  "amount": 2500.00,
  "taxableAmount": 0.00,
  "vatAmount": 0.00,
  "taxRate": "exempt",
  "taxCategory": "professional-services",
  "jurisdiction": "NL",
  "transactionDate": "2026-05-20"
}
```

### Example 3: Retail Store (Utrecht) - Multiple Rates

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxRate",
    "slug": "tax-rate-reduced-9-food"
  },
  "name": "Reduced VAT 9% - Food",
  "itemType": "goods",
  "taxType": "VAT",
  "rate": 9,
  "jurisdiction": "NL",
  "validFrom": "2026-01-01",
  "validUntil": null,
  "applicableToCategories": ["food", "groceries"],
  "reverseChargeApplies": false,
  "description": "Reduced VAT for food and groceries"
}

{
  "@self": {
    "register": "shillinq",
    "schema": "TaxRate",
    "slug": "tax-rate-standard-21-goods"
  },
  "name": "Standard VAT 21% - Goods",
  "itemType": "goods",
  "taxType": "VAT",
  "rate": 21,
  "jurisdiction": "NL",
  "validFrom": "2026-01-01",
  "validUntil": null,
  "applicableToCategories": ["electronics", "clothing", "household"],
  "reverseChargeApplies": false,
  "description": "Standard VAT for most retail goods"
}
```

### Example 4: IV3 Filing

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "TaxDeclaration",
    "slug": "iv3-decl-2025"
  },
  "name": "IV3 Declaration 2025",
  "declarationType": "IV3",
  "jurisdiction": "NL",
  "periodStart": "2025-01-01",
  "periodEnd": "2025-12-31",
  "status": "accepted",
  "grossSales": 150000.00,
  "totalVatCollected": 0.00,
  "totalVatDeductible": 0.00,
  "netVatOwed": 0.00,
  "submissionDeadline": "2026-03-31",
  "submittedAt": "2026-03-20T14:30:00Z",
  "submissionStatus": "accepted",
  "filingReference": "IV3-CBS-2025-987654"
}
```

## Reuse Analysis

### Existing OpenRegister Services Leveraged

| Service | Usage | Notes |
|---------|-------|-------|
| `ObjectService` | Create/read/update TaxConfiguration, TaxRate, TaxDeclaration entities | Standard CRUD operations |
| `SchemaService` | Register tax schemas at app initialization | Via repair step, `importFromApp()` |
| `ImportService` | Import tax rates from CSV/Excel (bulk upload) | Bulk tax rate configuration |
| `ExportService` | Export tax declarations, transaction lists as CSV/JSON | For external audit/archival |
| `IndexService` | Full-text search over tax declarations by type/year/status | Dashboard search, filing history |
| `AuditTrailService` | Auto-track every declaration change + filing submission | Compliance audit trail |
| `NotificationService` | Alert on filing deadlines, objection responses | Nextcloud notifications + email |
| `FileService` | Attach supporting documents to declarations | Receipt scans, audit reports |
| `VectorizationService` | Semantic search: "find all Q2 filings", "show my income tax estimates" | Optional enhancement |

### Custom Logic Required (Service Classes)

| Requirement | Why Service Needed | Scope |
|---|---|---|
| Receipt OCR (Scan & Herken) | External SDK integration; no schema extension covers OCR workflow | Adapter to Scan & Herken API |
| Tax Liability Estimation | Aggregates transactions across multiple tax categories + forecasts annual; calculation logic is domain-specific | Aggregate calculation beyond schema engine |
| Belastingdienst Filing Integration | State-machine triggers that call external Belastingdienst API on submission; OAuth + signing; requires external sealing | External API orchestration |
| AvaTax Lookup | Real-time tax rate validation via AvaTax API when tax rate is created/modified | External validation service |
| Reverse Charge Validation | Complex cross-border B2B rule engine (EU VAT rules, intra-community, country-of-origin logic) | Domain rule engine |

## Lifecycle & Workflows

### Tax Declaration Lifecycle

```
draft → in-review → ready-to-file → submitted → accepted | rejected | objection-filed → resolved
```

**Transitions & Guards:**

| From | To | Guard | Trigger |
|---|---|---|---|
| draft | in-review | - | Manual; user clicks "Review" |
| in-review | ready-to-file | Consistency checks pass | Auto; validates completeness |
| ready-to-file | submitted | - | Manual; user clicks "File" |
| submitted | accepted | External API response (Belastingdienst) | Webhook: `filing:accepted` |
| submitted | rejected | External API response | Webhook: `filing:rejected` |
| accepted | objection-filed | User initiates bezwaar | Manual registration |
| objection-filed | resolved | Assessment authority responds | External status update |

Declared in `x-openregister-lifecycle`.

## Declarative-vs-Imperative Decision

**Schema-engine coverage:**
- ✅ TaxConfiguration, TaxRate, TaxDeclaration, TaxReturn, ExemptionCertificate — **declarative**; CRUD via ObjectService
- ✅ Tax declaration state machine — **declarative**; `x-openregister-lifecycle` (draft → submitted → accepted)
- ✅ Tax calculations (VAT totals, tax-by-category aggregations) — **declarative**; `x-openregister-aggregations` + `x-openregister-calculations`
- ✅ Filing deadline alerts — **declarative**; `x-openregister-notifications`
- ❌ Receipt OCR workflow — **imperative** (external SDK); service adapter
- ❌ Belastingdienst API calls — **imperative** (external state machine); integration service
- ❌ Tax rate validation against external engines (AvaTax) — **imperative**; validation service
- ❌ Reverse charge rule engine — **imperative** (domain-specific); guard service called by lifecycle

## Integration Hooks

### Receipt OCR (Scan & Herken)

When a receipt file is uploaded to a TaxableTransaction:
1. Trigger `FileUploadedHandler`
2. Call Scan & Herken API
3. Extract merchant, date, amount, VAT, line items
4. Auto-populate TaxableTransaction fields
5. Assign tax category based on merchant match

### Belastingdienst Electronic Filing

When a TaxDeclaration transitions to `submitted`:
1. Serialize declaration to Belastingdienst XSD format
2. Sign with business certificate
3. POST to Belastingdienst endpoint
4. Poll for filing status
5. Webhook callback updates TaxDeclaration status

### AvaTax Rate Validation

Before saving a TaxRate:
1. Call AvaTax API: `GetTaxRateByAddress(jurisdiction, itemType)`
2. Compare provided rate against AvaTax rate
3. Warn if mismatch; allow override with justification
4. Log deviation for audit

## Constraints & Assumptions

- All transactions are in EUR (multi-currency support in phase 2)
- Tax year = calendar year (Dutch standard); fiscal year flexibility in future
- VAT filing uses quarterly period (Dutch standard); monthly/annual in future
- OCR confidence > 90% considered high-confidence; manual review required otherwise
- Reverse charge rules are Netherlands-centric (expand to EU in regional specs)
- Belastingdienst uses DigiSign for document signing (integrated via separate service)

## Testing Strategy

1. **Unit:** Tax rate validation, aggregation logic, lifecycle guards
2. **Integration:** Import tax rates, OCR receipt, submit declaration mock, webhook processing
3. **Browser:** VAT return dashboard, filing submission, receipt upload, declaration review
4. **Regression:** Existing invoice/expense workflows unaffected
