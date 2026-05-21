# Design: Tax & Levy Management — Shillinq — Other T2

**Status:** In Design  
**Date:** 2026-05-21  
**Change ID:** tax-levy-management-other-t2  

## Architecture Overview

### Core Components

```
┌─────────────────────────────────────────┐
│ Transaction Entry (Invoice/Expense)     │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Auto-Detection Service          │   │
│  │  - Country/Language             │   │
│  │  - VAT status (subject/reverse) │   │
│  │  - Supplier type (B2B/B2C/etc)  │   │
│  └────────────────┬────────────────┘   │
├────────────────────┼────────────────────┤
│                    │                    │
│  VAT Determination │  Tax Rate Engine   │
│  (Detection Rules)  │  (Calc & Storage)  │
│                    │                    │
│  - Standard        │  - Compound rates  │
│  - Reverse Charge  │  - Incl/Excl logic │
│  - Exemption       │  - Multi-jurisd    │
│  - Inbound/Outbound │                    │
│                    │                    │
├────────────────────┴────────────────────┤
│ Tax Position Aggregation                │
│  (Return & Report Generation)           │
│  - BTW-aangifte prep                    │
│  - Tax liability calc                   │
│  - Exemption reconciliation             │
└─────────────────────────────────────────┘
```

### Entity Diagram

```
Organization
  └─ TaxConfiguration (1:1)
      ├─ jurisdiction (NL, DE, etc.)
      ├─ filingFrequency (quarterly/annual)
      ├─ exemptionStatus (KOR, B2B, etc.)
      └─ registeredOffice (ref to Location/BAG)

TaxableTransaction
  ├─ TaxRate (many:1) [country, type, rate]
  ├─ TaxableAmount (derived)
  ├─ VATStatus (Standard/Reverse/Exempt)
  ├─ TaxableCategory (sales/purchase/payroll)
  └─ JurisdictionTag (NL, EU, International)

TaxExemption (NEW)
  ├─ exemptionType (KOR, B2B, reverse, etc.)
  ├─ jurisdiction
  ├─ validFrom / validTo
  ├─ conditions (JSON: threshold, supplier type, etc.)
  └─ appliedTransactions (ref list)

TaxReturn (existing, extended)
  ├─ returnType (btw-aangifte, income, payroll)
  ├─ period (period ID + dates)
  ├─ status (draft/filed/amended)
  ├─ lines (TaxReturnLine[])
  └─ auditTrail (full change history)

TaxRate
  ├─ country
  ├─ type (standard/reduced/zero/reverse)
  ├─ rate (decimal: 0.21, 0.09, 0.0)
  ├─ compoundWith (ref to other rate, optional)
  ├─ inclusive (bool: gross includes tax)
  ├─ validFrom / validTo
  └─ jurisdiction
```

## Data Model

### New Entities

#### TaxExemption
```json
{
  "slug": "TaxExemption",
  "title": "Tax Exemption",
  "description": "Tax exemption rule (KOR, B2B reverse charge, etc.)",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "UUID"
    },
    "exemptionType": {
      "type": "string",
      "enum": ["KOR", "B2B", "ReverseCharge", "SupplyOfServices", "ArtisticWorks", "SecondHandGoods", "Custom"],
      "description": "Exemption classification per Dutch/EU VAT rules"
    },
    "jurisdiction": {
      "type": "string",
      "enum": ["NL", "DE", "BE", "FR", "EU", "International"],
      "description": "Geographic scope"
    },
    "validFrom": {
      "type": "string",
      "format": "date",
      "description": "Start date of validity"
    },
    "validTo": {
      "type": "string",
      "format": "date",
      "nullable": true,
      "description": "End date (null = ongoing)"
    },
    "conditions": {
      "type": "object",
      "description": "Exemption conditions (threshold, supplier type, etc.)",
      "properties": {
        "minAmount": { "type": "number", "nullable": true },
        "maxAmount": { "type": "number", "nullable": true },
        "supplierCountries": { "type": "array", "items": { "type": "string" } },
        "supplierType": { "type": "string", "enum": ["Individual", "Business", "Government", "NPO", "Any"], "nullable": true },
        "transactionType": { "type": "string", "enum": ["Sale", "Purchase", "Service", "Any"], "nullable": true }
      }
    },
    "appliesAutomatically": {
      "type": "boolean",
      "description": "If true, system applies exemption without user confirmation"
    },
    "notes": {
      "type": "string",
      "nullable": true
    },
    "createdBy": {
      "type": "string",
      "description": "User ID"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["exemptionType", "jurisdiction", "validFrom", "appliesAutomatically"]
}
```

#### TaxDetectionRule (NEW)
```json
{
  "slug": "TaxDetectionRule",
  "title": "Tax Detection Rule",
  "description": "Auto-detection rule for VAT classification and rate selection",
  "type": "object",
  "properties": {
    "id": {
      "type": "string"
    },
    "priority": {
      "type": "integer",
      "description": "Evaluation order (0=highest)"
    },
    "name": {
      "type": "string",
      "description": "Rule name (e.g., 'NL Domestic Sale')"
    },
    "conditions": {
      "type": "object",
      "description": "Matching conditions",
      "properties": {
        "supplierCountry": { "type": "string", "nullable": true },
        "customerCountry": { "type": "string", "nullable": true },
        "transactionType": { "type": "string", "enum": ["Sale", "Purchase", "Service"] },
        "amountRange": { "type": "object", "properties": { "min": { "type": "number" }, "max": { "type": "number" } } },
        "customerType": { "type": "string", "enum": ["Individual", "Business", "Government"], "nullable": true },
        "keywords": { "type": "array", "items": { "type": "string" }, "description": "Match invoice description/notes" }
      }
    },
    "outcome": {
      "type": "object",
      "description": "Determined values if rule matches",
      "properties": {
        "vatStatus": { "type": "string", "enum": ["Standard", "ReverseCharge", "Exempt", "ZeroRated"] },
        "ratePercentage": { "type": "number" },
        "jurisdiction": { "type": "string" }
      }
    },
    "enabled": {
      "type": "boolean"
    }
  },
  "required": ["priority", "name", "conditions", "outcome"]
}
```

### Extended Entities

#### TaxConfiguration (Extended)
Add fields to existing TaxConfiguration:
```json
{
  "autodetectionEnabled": {
    "type": "boolean",
    "description": "Enable automatic VAT/tax detection"
  },
  "detectionRules": {
    "type": "array",
    "items": { "type": "string" },
    "description": "Array of TaxDetectionRule IDs"
  },
  "multiJurisdictionEnabled": {
    "type": "boolean",
    "description": "Support multiple jurisdictions in single period"
  },
  "jurisdictions": {
    "type": "array",
    "items": { "type": "string" },
    "description": "Active jurisdictions (NL, DE, BE, etc.)"
  },
  "exemptionRules": {
    "type": "array",
    "items": { "type": "string" },
    "description": "Array of TaxExemption IDs"
  },
  "recoveryRules": {
    "type": "object",
    "description": "Inbound VAT recovery settings",
    "properties": {
      "inboundVATRecoveryAllowed": { "type": "boolean" },
      "partialRecoveryPercentage": { "type": "number", "minimum": 0, "maximum": 100 },
      "excludedCategories": { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

#### TaxRate (Extended, if needed)
```json
{
  "compoundWith": {
    "type": "string",
    "nullable": true,
    "description": "UUID of base rate if this is a compound rate"
  },
  "inclusive": {
    "type": "boolean",
    "description": "True if rate is included in net amount (gross basis)"
  },
  "rateType": {
    "type": "string",
    "enum": ["Standard", "Reduced", "Zero", "ReverseCharge", "Exempt"],
    "description": "Classification of rate"
  }
}
```

## Tax Determination Logic

### VAT Detection Algorithm

```
DETERMINE_VAT(invoice):
  1. Extract transaction metadata:
     - supplier_country = invoice.supplier.country
     - customer_country = invoice.customer.country
     - transaction_type = invoice.type (Sale/Purchase/Service)
     - amount = invoice.total_amount
     - customer_type = infer(invoice.customer.id, invoice.customer.type)

  2. Check exemptions first:
     FOR each applicable TaxExemption (jurisdiction match, date valid):
       IF conditions_match(invoice, exemption):
         RETURN VATStatus.EXEMPT

  3. Apply detection rules (priority order):
     FOR each enabled TaxDetectionRule (sorted by priority):
       IF all conditions_match(metadata, rule.conditions):
         RETURN rule.outcome (VATStatus, rate %, jurisdiction)

  4. Default fallback:
     IF no rule matched:
       RETURN Configuration.defaultVATStatus (usually Standard + standard_rate)

  5. Apply compound rates (if configured):
     IF tax_rate.compoundWith != null:
       base_tax = calculate(amount, tax_rate.compoundWith.percentage)
       total_tax = calculate(amount + base_tax, tax_rate.percentage)
       RETURN {tax_amount: total_tax, explanation: "Compound rate"}
```

### Recovery Rules

```
CALCULATE_RECOVERABLE_VAT(inbound_vat, category):
  config = TaxConfiguration.recoveryRules

  IF NOT config.inboundVATRecoveryAllowed:
    RETURN 0

  IF category IN config.excludedCategories:
    RETURN 0

  partial_percent = config.partialRecoveryPercentage || 100
  RETURN inbound_vat * (partial_percent / 100)
```

## Seed Data

### TaxRate Examples

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxRate",
      "slug": "nl-standard-vat-21"
    },
    "country": "NL",
    "type": "standard",
    "rate": 0.21,
    "jurisdiction": "NL",
    "inclusive": false,
    "validFrom": "2024-01-01"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxRate",
      "slug": "nl-reduced-vat-9"
    },
    "country": "NL",
    "type": "reduced",
    "rate": 0.09,
    "jurisdiction": "NL",
    "inclusive": false,
    "validFrom": "2024-01-01"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxRate",
      "slug": "nl-zero-vat"
    },
    "country": "NL",
    "type": "zero",
    "rate": 0.0,
    "jurisdiction": "NL",
    "inclusive": false,
    "validFrom": "2024-01-01"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxRate",
      "slug": "de-standard-vat-19"
    },
    "country": "DE",
    "type": "standard",
    "rate": 0.19,
    "jurisdiction": "DE",
    "inclusive": false,
    "validFrom": "2024-01-01"
  }
]
```

### TaxExemption Examples

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxExemption",
      "slug": "kor-small-business"
    },
    "exemptionType": "KOR",
    "jurisdiction": "NL",
    "validFrom": "2024-01-01",
    "validTo": null,
    "conditions": {
      "maxAmount": 50000,
      "transactionType": "Sale"
    },
    "appliesAutomatically": true,
    "createdBy": "admin",
    "createdAt": "2026-05-21T10:00:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxExemption",
      "slug": "b2b-reverse-charge"
    },
    "exemptionType": "B2B",
    "jurisdiction": "NL",
    "validFrom": "2024-01-01",
    "validTo": null,
    "conditions": {
      "supplierCountries": ["NL", "DE", "BE"],
      "supplierType": "Business",
      "transactionType": "Purchase"
    },
    "appliesAutomatically": true,
    "createdBy": "admin",
    "createdAt": "2026-05-21T10:00:00Z"
  }
]
```

### TaxDetectionRule Examples

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxDetectionRule",
      "slug": "nl-domestic-sale"
    },
    "priority": 1,
    "name": "NL Domestic Sale (Standard VAT)",
    "conditions": {
      "supplierCountry": "NL",
      "customerCountry": "NL",
      "transactionType": "Sale",
      "customerType": "Business"
    },
    "outcome": {
      "vatStatus": "Standard",
      "ratePercentage": 21,
      "jurisdiction": "NL"
    },
    "enabled": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxDetectionRule",
      "slug": "eu-b2b-reverse-charge"
    },
    "priority": 2,
    "name": "EU B2B Reverse Charge",
    "conditions": {
      "supplierCountry": ["DE", "BE", "FR"],
      "customerCountry": "NL",
      "transactionType": "Purchase",
      "customerType": "Business"
    },
    "outcome": {
      "vatStatus": "ReverseCharge",
      "ratePercentage": 0,
      "jurisdiction": "NL"
    },
    "enabled": true
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxDetectionRule",
      "slug": "nl-goods-import"
    },
    "priority": 3,
    "name": "Imported Goods (Standard VAT)",
    "conditions": {
      "transactionType": "Purchase",
      "keywords": ["import", "customs", "duty"]
    },
    "outcome": {
      "vatStatus": "Standard",
      "ratePercentage": 21,
      "jurisdiction": "NL"
    },
    "enabled": true
  }
]
```

### TaxConfiguration Example

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "TaxConfiguration",
      "slug": "default-org-tax-config"
    },
    "organization": "org-123",
    "jurisdiction": "NL",
    "filingFrequency": "quarterly",
    "autodetectionEnabled": true,
    "detectionRules": [
      "nl-domestic-sale",
      "eu-b2b-reverse-charge",
      "nl-goods-import"
    ],
    "exemptionRules": [
      "kor-small-business",
      "b2b-reverse-charge"
    ],
    "multiJurisdictionEnabled": true,
    "jurisdictions": ["NL", "DE", "BE"],
    "recoveryRules": {
      "inboundVATRecoveryAllowed": true,
      "partialRecoveryPercentage": 100,
      "excludedCategories": ["PersonalExpense", "Meals"]
    }
  }
]
```

## API Endpoints

### Tax Detection

```
GET /api/tax-detection/determine
  ?transactionType=Sale
  &supplierCountry=NL
  &customerCountry=NL
  &amount=1000
  &customerType=Business
→ { vatStatus: "Standard", rate: 0.21, explanation: "nl-domestic-sale rule" }
```

### Tax Exemptions

```
GET /api/tax-exemptions
  ?jurisdiction=NL
  &validAt=2026-05-21
→ [ { id, exemptionType, conditions, ... }, ... ]

POST /api/tax-exemptions
  { exemptionType, jurisdiction, conditions, ... }
→ { id, ... }
```

### Tax Rates

```
GET /api/tax-rates
  ?country=NL
  &validAt=2026-05-21
  &type=standard
→ [ { id, rate, jurisdiction, ... }, ... ]
```

### Tax Returns

```
GET /api/tax-returns/{id}/calculate
→ { lines, totalTax, period, status, ... }

POST /api/tax-returns/{id}/finalize
  { approvedBy, notes }
→ { status: "filed", ... }
```

## Reuse Analysis

**OpenRegister services leveraged:**
- `ObjectService` — CRUD for TaxRate, TaxExemption, TaxDetectionRule, TaxConfiguration
- `SearchService` — Find applicable rules by jurisdiction/date/transaction type
- `RelationService` — Link exemptions and rates to transactions
- `AuditTrailService` — Track all tax determination and override history
- `NotificationService` — Alert on filing deadlines, compliance issues
- `ExportService` — Export tax returns to e-filing formats (future)

**NO custom logic needed for:**
- Permission/RBAC (use `AuthorizationService`)
- CRUD UI (use `CnIndexPage`, `CnFormDialog`)
- Change tracking (automatic via `AuditTrailService`)

## Frontend Architecture

**Pages:**
- `/tax-rates` — List + CRUD tax rates (CnIndexPage)
- `/tax-exemptions` — List + CRUD exemptions (CnIndexPage)
- `/tax-detection-rules` — Configure auto-detection (CnIndexPage + rule editor)
- `/tax-configuration` — Settings (CnSettingsSection)
- `/tax-returns` — List returns, view details, finalize (CnIndexPage + CnDetailPage)
- `/tax-dashboard` — Overview + key metrics (CnDashboardPage)

**Stores:**
- `taxRateStore` — `createObjectStore('taxRate', 'TaxRate', 'shillinq')`
- `taxExemptionStore` — `createObjectStore('taxExemption', 'TaxExemption', 'shillinq')`
- `taxConfigStore` — `createObjectStore('taxConfig', 'TaxConfiguration', 'shillinq')`

## Deduplication Check

**Findings:**
- ✅ No overlap with OpenRegister core services
- ✅ VAT determination logic is Shillinq-specific (not a general OpenRegister service)
- ✅ Rule engine follows existing pattern (similar to workflow rules in workflowengineregistry)
- ✅ Exemption handling is domain-specific, not duplicated elsewhere
- ✅ TaxableTransaction, TaxRate, TaxReturn are existing OpenRegister entities

**Conclusion:** All new entities are Shillinq-specific and appropriate for app-level data storage.

## Implementation Notes

1. **VAT Detection:** Use priority-based rule engine, with exemptions checked first
2. **Compound Rates:** Support multi-level tax (e.g., VAT + sales tax) via compoundWith reference
3. **Inclusive Rates:** Store both gross and net in transaction, calculate tax as difference
4. **Jurisdiction Isolation:** Filter rules by active jurisdictions per TaxConfiguration
5. **Audit Trail:** Track every tax determination (auto-detected or overridden) with rule ID and user

---

**Status:** Ready for Specs  
**Next:** Create GIVEN/WHEN/THEN scenarios for each feature
