# Tax & Levy Management — Design & Specification

## Overview

Shillinq's tax management layer declares VAT calculation rules, tax code libraries, withholding tax configurations, and compliance tracking via OpenRegister schema extensions. All business logic is expressed as schema metadata (`x-openregister-calculations`, `x-openregister-aggregations`, `x-openregister-relations`) — no custom PHP service classes.

This design follows ADR-031 (schema-declarative business logic) and ADR-011 (schema.org vocabulary). Tax-specific Dutch mappings are documented but not automated in this release.

---

## Declarative-vs-Imperative Decisions

### Decision 1: VAT Calculation
**Scope:** Compute VAT amount on invoice line items given tax code, amount, and quantity

**Option A (Imperative):** PHP `TaxCalculationService::calculateVAT()`
- Pros: Explicit control, testable service
- Cons: Not audit-trailed; not replayable; not RBAC-aware

**Option B (Declarative):** `x-openregister-calculations` on InvoiceLine
- Pros: Audit trail on every calculation; replayable; automatic CloudEvent; available to dashboard aggregations
- Cons: Complex formulas need `@formula` syntax validation

**Decision:** **Declarative (Option B)**
- VAT = (lineAmount × taxRate) / (1 + taxRate) [inclusive] OR lineAmount × taxRate [exclusive]
- Formula encoded in schema; guard checks tax-code-compatible-with-invoice-type

### Decision 2: Provisional vs Actual Tax Reconciliation
**Scope:** Compare year-to-date provisional tax payments against calculated liability; show balance

**Option A (Imperative):** `TaxReconciliationService::calculateBalance()`
- Pros: One-off reporting; custom SQL query per jurisdiction
- Cons: Not reusable; not GraphQL-discoverable

**Option B (Declarative):** `x-openregister-aggregations` on TaxPayment + TaxDeclaration
- Pros: Same aggregation serves dashboard, API, GraphQL, MCP; jurisdiction-neutral definition
- Cons: Complex WHERE clauses need parameter pass-through

**Decision:** **Declarative (Option B)**
- Aggregation: `sum(all payments YTD) - calculateLiability()` grouped by jurisdiction
- Filters: date range, payment type (provisional/final), jurisdiction

### Decision 3: Withholding Tax Tracking
**Scope:** Record supplier-specific withholding rates; track withheld amounts YTD

**Option A (Imperative):** Supplier portal plugin with custom withheld-amount journal
- Pros: Supplier-self-service; custom audit log
- Cons: Not federated; supplier data not in Shillinq schema; duplicate reconciliation

**Option B (Declarative):** Link Supplier → TaxScheme (via relation); track withheld amounts on payment record
- Pros: Single source of truth; relation-based (cross-register); RBAC-aware
- Cons: Supplier entity lives in openregister; requires TaxScheme read-permission on supplier detail

**Decision:** **Declarative (Option B)**
- Relation: Supplier.withholding_tax_scheme_id → TaxScheme.id
- PaymentRecord.withholding_tax_amount and PaymentRecord.withholding_tax_basis recorded as fields

---

## Entity Schemas

All schemas align with schema.org vocabulary where applicable. Dutch-specific fields use mapping layers rather than hardcoded names (see Mapping Rules section).

### 1. TaxCode

**Purpose:** Library of tax codes (VAT, import duty, withholding rates, etc.) used in calculations.

**Scope:** Global configuration; read by invoicing, procurement, payment modules. CRUD operations restricted to admin.

**Schema definition:**
```json
{
  "type": "object",
  "title": "Tax Code",
  "description": "A tax code defining rate, jurisdiction, applicability rules, and calculation method",
  "properties": {
    "id": {
      "type": "string",
      "description": "UUID or slug (e.g., 'NL-VAT-21-standard')",
      "format": "uuid"
    },
    "code": {
      "type": "string",
      "description": "Human-readable code (e.g., 'VAT-21', 'IMPORT-DUTY-MACHINERY')",
      "minLength": 1,
      "maxLength": 20
    },
    "name": {
      "type": "string",
      "description": "Display name (e.g., 'VAT 21% — Standard rate')",
      "minLength": 1,
      "maxLength": 255
    },
    "jurisdiction": {
      "type": "string",
      "enum": ["NL", "EU", "GLOBAL"],
      "description": "Jurisdiction this code applies to; GLOBAL for universal rates"
    },
    "taxType": {
      "type": "string",
      "enum": ["VAT", "IMPORT_DUTY", "WITHHOLDING", "EXCISE", "OTHER"],
      "description": "Category of tax"
    },
    "rate": {
      "type": "number",
      "description": "Tax rate as decimal (e.g., 0.21 for 21%)",
      "minimum": 0,
      "maximum": 1
    },
    "effectiveDate": {
      "type": "string",
      "format": "date",
      "description": "Date this rate becomes effective (ISO 8601)"
    },
    "expiryDate": {
      "type": "string",
      "format": "date",
      "description": "Date this rate expires; null if indefinite",
      "nullable": true
    },
    "calculationMethod": {
      "type": "string",
      "enum": ["INCLUSIVE", "EXCLUSIVE", "COMPOUND"],
      "description": "How to apply tax: inclusive of base, exclusive (added), or dependent on another tax"
    },
    "dependsOnTaxCode": {
      "type": "string",
      "description": "For COMPOUND: reference to TaxCode this tax is calculated on (e.g., VAT on top of import duty)",
      "nullable": true
    },
    "applicableTo": {
      "type": "array",
      "description": "Transaction types this code applies to (INVOICE_SALE, INVOICE_PURCHASE, IMPORT, etc.)",
      "items": {
        "type": "string",
        "enum": ["INVOICE_SALE", "INVOICE_PURCHASE", "IMPORT", "EXPORT", "PAYMENT", "REFUND"]
      }
    },
    "description": {
      "type": "string",
      "description": "Regulatory reference or notes (e.g., 'Dutch Tariff 87.02 — Auto parts')",
      "nullable": true
    },
    "dutchTaxId": {
      "type": "string",
      "description": "Dutch tax system identifier (IV3 code, tariff number, etc.) for mapping to government reports",
      "nullable": true
    }
  },
  "required": ["code", "name", "jurisdiction", "taxType", "rate", "effectiveDate", "calculationMethod", "applicableTo"]
}
```

**Calculations (x-openregister-calculations):**
- `isActive`: `effectiveDate <= today AND (expiryDate IS NULL OR expiryDate >= today)`
- `displayRate`: `rate * 100` (for UI display: "21%")

**Relations:**
- None (TaxCode is a reference-only entity)

**RBAC:**
- Create/edit/delete: Admin only
- Read: All authenticated users

---

### 2. TaxScheme

**Purpose:** Bundled tax configuration for a given entity (freelancer under KOR, company subject to VAT, importer with duty exemptions).

**Scope:** Associated with Organization, Freelancer, or Supplier. Determines which tax codes apply, and any exemptions.

**Schema definition:**
```json
{
  "type": "object",
  "title": "Tax Scheme",
  "description": "A tax regime applicable to an organization or supplier (e.g., KOR exemption, reverse charge, import VAT suspension)",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "description": "Display name (e.g., 'Freelancer KOR', 'EU Reverse Charge')",
      "minLength": 1,
      "maxLength": 255
    },
    "schemeType": {
      "type": "string",
      "enum": ["STANDARD", "EXEMPTED", "SIMPLIFIED", "SPECIAL_REGIME"],
      "description": "VAT scheme category per Dutch rules"
    },
    "exemptionCode": {
      "type": "string",
      "description": "Code for exemption (e.g., 'KOR', 'EXPORT_EXEMPT', 'CHARITY')",
      "enum": ["KOR", "EXPORT_EXEMPT", "CHARITY_MEDICAL", "FINANCIAL_SERVICES", "REVERSE_CHARGE", "SMALL_TRADER"],
      "nullable": true
    },
    "effectiveDate": {
      "type": "string",
      "format": "date"
    },
    "expiryDate": {
      "type": "string",
      "format": "date",
      "nullable": true
    },
    "applicableTaxCodes": {
      "type": "array",
      "description": "Array of TaxCode IDs that apply under this scheme (e.g., [VAT-6-reduced, VAT-21-standard] for normal trader; [] for KOR-exempt)",
      "items": {
        "type": "string"
      }
    },
    "exemptTransactionTypes": {
      "type": "array",
      "description": "Transaction types exempt from tax under this scheme (e.g., EXPORT for Export Scheme)",
      "items": {
        "type": "string",
        "enum": ["INVOICE_SALE", "INVOICE_PURCHASE", "IMPORT", "EXPORT", "PAYMENT"]
      },
      "nullable": true
    },
    "withholding_tax_scheme": {
      "type": "string",
      "description": "Reference to TaxCode for withholding-tax configuration (supplier-specific rate)",
      "nullable": true
    },
    "notes": {
      "type": "string",
      "description": "Regulatory or contract notes",
      "nullable": true
    }
  },
  "required": ["name", "schemeType", "effectiveDate", "applicableTaxCodes"]
}
```

**Calculations:**
- `isActive`: `effectiveDate <= today AND (expiryDate IS NULL OR expiryDate >= today)`

**Relations:**
- `Supplier.withholding_tax_scheme_id → TaxScheme.id` (for supplier-specific withholding)
- `Organization.tax_scheme_id → TaxScheme.id` (org-wide tax regime)

**RBAC:**
- Create/edit/delete: Admin + tax admin role
- Read: All authenticated users (own schemes visible to user)

---

### 3. TaxableTransaction

**Purpose:** Tracks which transactions have been tax-categorized and prepared for reporting.

**Scope:** Materialized view or query-filtered subset of all Transactions (GL entries, invoice lines, etc.).

**Schema definition:**
```json
{
  "type": "object",
  "title": "Taxable Transaction",
  "description": "A transaction (invoice line, GL entry, payment) categorized for tax reporting with calculated tax amounts",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "transactionId": {
      "type": "string",
      "description": "Reference to underlying Transaction (GL entry) ID",
      "format": "uuid"
    },
    "type": {
      "type": "string",
      "enum": ["INVOICE_SALE_LINE", "INVOICE_PURCHASE_LINE", "PAYMENT", "IMPORT", "EXPENSE"],
      "description": "Category of transaction"
    },
    "date": {
      "type": "string",
      "format": "date",
      "description": "Transaction date (invoice date, payment date, posting date)"
    },
    "description": {
      "type": "string",
      "description": "Line item or transaction description"
    },
    "lineAmount": {
      "type": "number",
      "description": "Base amount before tax (in EUR or transaction currency)",
      "minimum": 0
    },
    "currency": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "ISO 4217 currency code"
    },
    "appliedTaxCode": {
      "type": "string",
      "description": "TaxCode ID applied to this transaction",
      "nullable": true
    },
    "taxAmount": {
      "type": "number",
      "description": "Calculated tax (computed via x-openregister-calculation)",
      "minimum": 0
    },
    "totalAmount": {
      "type": "number",
      "description": "lineAmount + taxAmount (if EXCLUSIVE) or lineAmount (if INCLUSIVE)",
      "minimum": 0
    },
    "taxCategory": {
      "type": "string",
      "enum": ["VAT_DEDUCTIBLE", "VAT_OUTPUT", "IMPORT_DUTY", "WITHHOLDING", "EXCISE", "OTHER"],
      "description": "Tax reporting category (input VAT, output VAT, etc.)"
    },
    "fiscalYear": {
      "type": "integer",
      "description": "Calendar year for tax reporting (e.g., 2026)"
    },
    "reportingStatus": {
      "type": "string",
      "enum": ["PENDING", "FILED", "AMENDED", "RECONCILED"],
      "description": "Whether included in a VAT return or tax filing"
    }
  },
  "required": ["transactionId", "type", "date", "lineAmount", "currency", "taxCategory", "fiscalYear"]
}
```

**Calculations:**
- `taxAmount`: Conditional on appliedTaxCode.calculationMethod
  - If INCLUSIVE: `taxAmount = lineAmount - (lineAmount / (1 + taxCode.rate))`
  - If EXCLUSIVE: `taxAmount = lineAmount * taxCode.rate`
  - If COMPOUND: `taxAmount = (lineAmount + baseTaxAmount) * compoundRate`
- `totalAmount`: Derived from lineAmount + taxAmount

**Aggregations (x-openregister-aggregations):**
- Sum all VAT_OUTPUT for a fiscal year (sales tax collected)
- Sum all VAT_DEDUCTIBLE for a fiscal year (input VAT deducted)
- Sum all IMPORT_DUTY by tariff code
- Count by reportingStatus (pending, filed, amended)

**Relations:**
- TaxableTransaction.appliedTaxCode → TaxCode.id
- TaxableTransaction.transactionId → Transaction.id (or JournalEntry.id)

---

### 4. TaxDeclaration

**Purpose:** Represents a submitted or draft tax filing (VAT return, income tax return, corporate return).

**Scope:** Once-per-period (quarterly VAT return, annual income/corporate return).

**Schema definition:**
```json
{
  "type": "object",
  "title": "Tax Declaration",
  "description": "A tax return filed with authorities (VAT-aangifte, IB-aangifte, VPB-aangifte, suppletie) with calculated obligations and payment status",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "declarationType": {
      "type": "string",
      "enum": ["VAT_QUARTERLY", "INCOME_ANNUAL", "CORPORATE_ANNUAL", "SUPPLEMENTARY_VAT"],
      "description": "Type of tax filing"
    },
    "jurisdiction": {
      "type": "string",
      "enum": ["NL"],
      "description": "Jurisdiction (initially NL only)"
    },
    "periodStart": {
      "type": "string",
      "format": "date",
      "description": "Start of reporting period"
    },
    "periodEnd": {
      "type": "string",
      "format": "date",
      "description": "End of reporting period"
    },
    "filingDate": {
      "type": "string",
      "format": "date",
      "description": "Date filed with authorities (null if draft)",
      "nullable": true
    },
    "status": {
      "type": "string",
      "enum": ["DRAFT", "FILED", "ACCEPTED", "AMENDED", "CLOSED"],
      "description": "Filing status"
    },
    "totalIncome": {
      "type": "number",
      "description": "Total sales/revenue"
    },
    "totalDeductible": {
      "type": "number",
      "description": "Total deductible expenses or input VAT"
    },
    "calculatedLiability": {
      "type": "number",
      "description": "Calculated tax due (from x-openregister-calculation)"
    },
    "provisionalPaymentsMade": {
      "type": "number",
      "description": "Sum of voorlopige aanslag (provisional) payments made in period",
      "default": 0
    },
    "balanceDue": {
      "type": "number",
      "description": "Calculated liability - provisional payments (negative = refund due)",
      "nullable": true
    },
    "declarationUrl": {
      "type": "string",
      "format": "uri",
      "description": "Government filing reference or URL (e.g., Elias submission ID)",
      "nullable": true
    },
    "attachedDocuments": {
      "type": "array",
      "description": "File IDs of supporting documents (ledger excerpts, receipts, etc.)",
      "items": {
        "type": "string"
      },
      "nullable": true
    },
    "notes": {
      "type": "string",
      "description": "Internal notes (amendments, auditor comments, etc.)",
      "nullable": true
    }
  },
  "required": ["declarationType", "jurisdiction", "periodStart", "periodEnd", "status", "calculatedLiability"]
}
```

**Calculations:**
- `calculatedLiability`: Aggregation over all TaxableTransaction in period, grouped by taxCategory
  - VAT: `sumVATOutput - sumVATDeductible` (or VAT reclaim if negative)
  - Income: `totalIncome - totalExpenses`
  - Corporate: `corporateProfits * corporateTaxRate`
- `balanceDue`: `calculatedLiability - provisionalPaymentsMade`

**Aggregations:**
- Per declaration: sum all TaxableTransaction.taxAmount by taxCategory
- Comparative: current vs prior year (for amended filings)

**Relations:**
- TaxDeclaration ← TaxableTransaction[] (filtered by periodStart, periodEnd, taxCategory)
- TaxDeclaration → TaxPayment[] (provisional payments linked for balance calc)

---

### 5. TaxRate (Existing Entity — Enhanced)

**Note:** TaxRate already exists in OpenRegister core. This spec does NOT redefine it; instead, we reference it and recommend seed data.

**Enhancements for this spec:**
- Add `dutchTaxId` field (IV3 code, tariff code) for government mapping
- Add `applicableTo` enum array (INVOICE_SALE, IMPORT, etc.) to restrict application

Handled as a schema patch (non-breaking field additions).

---

## Seed Data

### TaxCode seeds (3 example entries; actual import from Dutch tariff library)

```json
{
  "@self": {
    "register": "shillinq_tax",
    "schema": "TaxCode",
    "slug": "nl-vat-21-standard"
  },
  "code": "VAT-21",
  "name": "VAT 21% — Standard rate",
  "jurisdiction": "NL",
  "taxType": "VAT",
  "rate": 0.21,
  "effectiveDate": "2019-01-01",
  "expiryDate": null,
  "calculationMethod": "EXCLUSIVE",
  "applicableTo": ["INVOICE_SALE", "INVOICE_PURCHASE"],
  "dutchTaxId": "VAT-STD"
}
```

```json
{
  "@self": {
    "register": "shillinq_tax",
    "schema": "TaxCode",
    "slug": "nl-vat-6-reduced"
  },
  "code": "VAT-6",
  "name": "VAT 6% — Reduced rate",
  "jurisdiction": "NL",
  "taxType": "VAT",
  "rate": 0.06,
  "effectiveDate": "2019-01-01",
  "expiryDate": null,
  "calculationMethod": "EXCLUSIVE",
  "applicableTo": ["INVOICE_SALE", "INVOICE_PURCHASE"],
  "dutchTaxId": "VAT-RED"
}
```

```json
{
  "@self": {
    "register": "shillinq_tax",
    "schema": "TaxCode",
    "slug": "nl-import-duty-machinery"
  },
  "code": "IMPORT-87",
  "name": "Import Duty — Machinery (HS 87.02)",
  "jurisdiction": "NL",
  "taxType": "IMPORT_DUTY",
  "rate": 0.05,
  "effectiveDate": "2019-01-01",
  "expiryDate": null,
  "calculationMethod": "EXCLUSIVE",
  "dependsOnTaxCode": null,
  "applicableTo": ["IMPORT"],
  "dutchTaxId": "HS-8702"
}
```

### TaxScheme seeds (2 example entries)

```json
{
  "@self": {
    "register": "shillinq_tax",
    "schema": "TaxScheme",
    "slug": "freelancer-kor"
  },
  "name": "Freelancer KOR (Small Business Exemption)",
  "schemeType": "EXEMPTED",
  "exemptionCode": "KOR",
  "effectiveDate": "2026-01-01",
  "expiryDate": null,
  "applicableTaxCodes": [],
  "exemptTransactionTypes": ["INVOICE_SALE"],
  "notes": "Freelancer earning < €50,000 annual turnover. KOR exempts from VAT collection but also restricts deductible input VAT."
}
```

```json
{
  "@self": {
    "register": "shillinq_tax",
    "schema": "TaxScheme",
    "slug": "standard-vat-trader"
  },
  "name": "Standard VAT Trader",
  "schemeType": "STANDARD",
  "exemptionCode": null,
  "effectiveDate": "2026-01-01",
  "expiryDate": null,
  "applicableTaxCodes": ["nl-vat-21-standard", "nl-vat-6-reduced"],
  "exemptTransactionTypes": null,
  "notes": "Normal VAT trader. Subject to quarterly VAT return obligation."
}
```

---

## Register Template

File: `lib/Settings/shillinq_tax_register.json`

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "Shillinq Tax Management",
    "version": "1.0.0",
    "description": "Tax codes, schemes, and reporting for VAT, income tax, and levy management"
  },
  "x-openregister": {
    "type": "application",
    "version": "1.0.0"
  },
  "paths": {},
  "components": {
    "schemas": {
      "TaxCode": {
        "type": "object",
        "title": "Tax Code",
        "properties": {
          "id": {"type": "string", "format": "uuid"},
          "code": {"type": "string"},
          "name": {"type": "string"},
          "jurisdiction": {"type": "string", "enum": ["NL", "EU", "GLOBAL"]},
          "taxType": {"type": "string", "enum": ["VAT", "IMPORT_DUTY", "WITHHOLDING", "EXCISE", "OTHER"]},
          "rate": {"type": "number", "minimum": 0, "maximum": 1},
          "effectiveDate": {"type": "string", "format": "date"},
          "expiryDate": {"type": "string", "format": "date", "nullable": true},
          "calculationMethod": {"type": "string", "enum": ["INCLUSIVE", "EXCLUSIVE", "COMPOUND"]},
          "dependsOnTaxCode": {"type": "string", "nullable": true},
          "applicableTo": {"type": "array", "items": {"type": "string"}},
          "description": {"type": "string", "nullable": true},
          "dutchTaxId": {"type": "string", "nullable": true}
        },
        "required": ["code", "name", "jurisdiction", "taxType", "rate", "effectiveDate", "calculationMethod", "applicableTo"],
        "x-openregister-lifecycle": null,
        "x-openregister-calculations": {
          "isActive": "effectiveDate <= @now AND (expiryDate IS NULL OR expiryDate >= @now)",
          "displayRate": "rate * 100"
        }
      },
      "TaxScheme": {
        "type": "object",
        "title": "Tax Scheme",
        "properties": {
          "id": {"type": "string", "format": "uuid"},
          "name": {"type": "string"},
          "schemeType": {"type": "string", "enum": ["STANDARD", "EXEMPTED", "SIMPLIFIED", "SPECIAL_REGIME"]},
          "exemptionCode": {"type": "string", "nullable": true},
          "effectiveDate": {"type": "string", "format": "date"},
          "expiryDate": {"type": "string", "format": "date", "nullable": true},
          "applicableTaxCodes": {"type": "array", "items": {"type": "string"}},
          "exemptTransactionTypes": {"type": "array", "items": {"type": "string"}, "nullable": true},
          "withholding_tax_scheme": {"type": "string", "nullable": true},
          "notes": {"type": "string", "nullable": true}
        },
        "required": ["name", "schemeType", "effectiveDate", "applicableTaxCodes"],
        "x-openregister-calculations": {
          "isActive": "effectiveDate <= @now AND (expiryDate IS NULL OR expiryDate >= @now)"
        }
      },
      "TaxableTransaction": {
        "type": "object",
        "title": "Taxable Transaction",
        "properties": {
          "id": {"type": "string", "format": "uuid"},
          "transactionId": {"type": "string", "format": "uuid"},
          "type": {"type": "string", "enum": ["INVOICE_SALE_LINE", "INVOICE_PURCHASE_LINE", "PAYMENT", "IMPORT", "EXPENSE"]},
          "date": {"type": "string", "format": "date"},
          "description": {"type": "string"},
          "lineAmount": {"type": "number", "minimum": 0},
          "currency": {"type": "string", "pattern": "^[A-Z]{3}$"},
          "appliedTaxCode": {"type": "string", "nullable": true},
          "taxAmount": {"type": "number", "minimum": 0},
          "totalAmount": {"type": "number", "minimum": 0},
          "taxCategory": {"type": "string", "enum": ["VAT_DEDUCTIBLE", "VAT_OUTPUT", "IMPORT_DUTY", "WITHHOLDING", "EXCISE", "OTHER"]},
          "fiscalYear": {"type": "integer"},
          "reportingStatus": {"type": "string", "enum": ["PENDING", "FILED", "AMENDED", "RECONCILED"]}
        },
        "required": ["transactionId", "type", "date", "lineAmount", "currency", "taxCategory", "fiscalYear"],
        "x-openregister-calculations": {
          "taxAmount": "(CASE WHEN @.appliedTaxCode.calculationMethod = 'EXCLUSIVE' THEN @.lineAmount * @.appliedTaxCode.rate WHEN @.appliedTaxCode.calculationMethod = 'INCLUSIVE' THEN @.lineAmount - (@.lineAmount / (1 + @.appliedTaxCode.rate)) ELSE 0 END)",
          "totalAmount": "@.lineAmount + @.taxAmount"
        }
      },
      "TaxDeclaration": {
        "type": "object",
        "title": "Tax Declaration",
        "properties": {
          "id": {"type": "string", "format": "uuid"},
          "declarationType": {"type": "string", "enum": ["VAT_QUARTERLY", "INCOME_ANNUAL", "CORPORATE_ANNUAL", "SUPPLEMENTARY_VAT"]},
          "jurisdiction": {"type": "string", "enum": ["NL"]},
          "periodStart": {"type": "string", "format": "date"},
          "periodEnd": {"type": "string", "format": "date"},
          "filingDate": {"type": "string", "format": "date", "nullable": true},
          "status": {"type": "string", "enum": ["DRAFT", "FILED", "ACCEPTED", "AMENDED", "CLOSED"]},
          "totalIncome": {"type": "number"},
          "totalDeductible": {"type": "number"},
          "calculatedLiability": {"type": "number"},
          "provisionalPaymentsMade": {"type": "number", "default": 0},
          "balanceDue": {"type": "number", "nullable": true},
          "declarationUrl": {"type": "string", "format": "uri", "nullable": true},
          "attachedDocuments": {"type": "array", "items": {"type": "string"}, "nullable": true},
          "notes": {"type": "string", "nullable": true}
        },
        "required": ["declarationType", "jurisdiction", "periodStart", "periodEnd", "status", "calculatedLiability"],
        "x-openregister-calculations": {
          "balanceDue": "@.calculatedLiability - @.provisionalPaymentsMade"
        },
        "x-openregister-aggregations": {
          "totalVATOutput": "SUM(TaxableTransaction.taxAmount WHERE taxCategory = 'VAT_OUTPUT' AND date BETWEEN @.periodStart AND @.periodEnd)",
          "totalVATDeductible": "SUM(TaxableTransaction.taxAmount WHERE taxCategory = 'VAT_DEDUCTIBLE' AND date BETWEEN @.periodStart AND @.periodEnd)"
        }
      }
    }
  }
}
```

---

## Reuse Analysis Summary

| OpenRegister capability | How used | Benefit |
|---|---|---|
| `ConfigurationService::importFromApp()` | Import seed TaxCode and TaxScheme from register | Automatic idempotent loading on install |
| `SchemaService` | Validate register template syntax | Pre-flight schema validation |
| `ObjectService::saveObject()` | Create/update TaxCode, TaxScheme, TaxableTransaction, TaxDeclaration | No custom mapper needed |
| `ObjectService::findAll()` + filters | Query TaxableTransaction by date range, taxCategory, fiscal year | Reusable for reporting queries |
| `x-openregister-calculations` | VAT amount, balance due, display rate | Audit-trailed, replayable, RBAC-aware |
| `x-openregister-aggregations` | VAT output/deductible sums, reclaim totals | Available to dashboard, GraphQL, API |
| `PropertyRbacHandler` | Restrict TaxCode editing to admin; read-only for users | Field-level access control |
| `@conduction/nextcloud-vue` CnIndexPage + CnDetailPage | List and detail views for TaxCode, TaxScheme, TaxDeclaration | Standard UI, no custom components |
| `FileService` | Attach supporting documents to TaxDeclaration | Standard file management in sidebar |

---

## Mapping Rules: Dutch Government Standards

**Scope:** This section defines the translation layer between Shillinq internal schemas and Dutch tax authority formats. Automation is out of scope for this release; mappings are documented for future export automation.

### IV3 (Elektronische Aangifte - Electronic VAT Return)

| Field | Shillinq schema | IV3 mapping |
|---|---|---|
| Declaration.totalVATOutput | Aggregation: SUM(TaxableTransaction WHERE taxCategory=VAT_OUTPUT) | Box 1 (VAT due on sales) |
| Declaration.totalVATDeductible | Aggregation: SUM(TaxableTransaction WHERE taxCategory=VAT_DEDUCTIBLE) | Box 2 (Input VAT deductible) |
| TaxCode.dutchTaxId = "VAT-STD" | rate = 0.21 | Standard rate (21%) |
| TaxCode.dutchTaxId = "VAT-RED" | rate = 0.06 | Reduced rate (6%) |
| TaxCode.dutchTaxId = "VAT-ZERO" | rate = 0 | Zero-rated (exports) |
| TaxScheme.exemptionCode = "KOR" | applicableTaxCodes = [] | Box 2 = 0 (no input VAT deduction) |
| TaxScheme.exemptionCode = "EXPORT_EXEMPT" | exemptTransactionTypes = ["EXPORT"] | Box 3 (zero-rated supplies) |

### Elias (Customs & Excise Portal)

| Field | Shillinq schema | Elias mapping |
|---|---|---|
| TaxableTransaction.type = "IMPORT" | appliedTaxCode.taxType = "IMPORT_DUTY" | Import duty schedule |
| TaxCode.dutchTaxId = "HS-8702" | applicableTo = ["IMPORT"] | HS tariff classification |
| TaxableTransaction.taxAmount | Calculated via x-openregister-calculation | Duty amount in €|

### BBV (Dutch Bookkeeping Standard)

- All TaxableTransaction entries logged with: date, amount, tax code, GL account
- TaxDeclaration period aligns with fiscal year
- TaxCode.dutchTaxId feeds GL chart-of-accounts mapping for audit trail

---

## No Custom Service Classes

Per ADR-031, **all business logic is schema metadata**. No PHP service classes for:
- Tax calculation (uses `x-openregister-calculations`)
- Tax aggregation / summary reports (uses `x-openregister-aggregations`)
- Scheme switching logic (uses relations + schema queries)
- Withholding tax tracking (uses schema fields + calculations)

Exceptions: None yet (all requirements fit declarative model).

---

## Integration Points

### Internal
- **Invoice module:** InvoiceLine → appliedTaxCode → calculation of lineAmount × taxRate
- **Payment module:** Supplier payment → withholding_tax_scheme → withheld amount tracking
- **Reporting module:** TaxableTransaction queries → TaxDeclaration aggregations

### External (Future Releases)
- **IV3 Export:** TaxDeclaration → Dutch tax authorities (Elias)
- **BBV Sync:** TaxableTransaction GL mapping for Dutch bookkeeping compliance
- **Supplier Portal:** Supplier view of withheld-tax-amount for reconciliation

---

## Performance Notes

- TaxCode (read-heavy): Index by `code`, `jurisdiction`, `isActive`
- TaxableTransaction (write-heavy): Index by `date`, `fiscalYear`, `taxCategory` for quarterly aggregations
- TaxDeclaration (read-heavy): Index by `periodStart`, `periodEnd`, `status`
- Aggregations (sum by taxCategory) pre-cached daily if > 100K rows per fiscal year

---

## Testing Strategy

1. **Unit / Integration:** Verify x-openregister-calculations correctness for each tax method (INCLUSIVE, EXCLUSIVE, COMPOUND)
2. **Scenario testing:** VAT return for KOR-exempted freelancer; standard trader; importer with compound duty+VAT
3. **Reconciliation:** Year-to-date provisional vs calculated liability balance (+ test refund scenarios)
4. **Mapping validation:** Sample tax return export against IV3 template (manual spot-check; automation TBD)
