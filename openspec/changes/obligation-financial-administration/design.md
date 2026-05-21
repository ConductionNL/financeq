# Design: Obligation & Financial Administration — Shillinq

## Design Principles

1. **Domain-Driven Design:** Obligation, Settlement, and Asset entities are first-class domain objects, not workflow state machines
2. **Audit-First:** Every obligation and settlement decision maintains a complete audit trail (automated via OpenRegister)
3. **Compliance-Native:** Dutch government standards (BBV, IV3, SiSa) baked into schema design, not added as features
4. **AI-Ready:** Obligation task generation designed for future ML-based deadline prediction and anomaly detection
5. **OpenRegister-First:** All data stored as OpenRegister objects; no custom database schema

## Data Model & Schemas

### 1. Obligation (schema:Order)

_A financial commitment that must be fulfilled by a specific due date, with tracking for AI task automation and compliance reporting._

```json
{
  "name": "Obligation",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "obligationNumber": {
      "type": "string",
      "description": "Unique reference number for the obligation (e.g., OBL-2026-001)"
    },
    "obligationDate": {
      "type": "string",
      "format": "date",
      "description": "Date the obligation was created"
    },
    "dueDate": {
      "type": "string",
      "format": "date",
      "description": "Date by which the obligation must be settled"
    },
    "amount": {
      "$ref": "https://openregister.app/schemas/MonetaryAmount",
      "description": "Financial amount owed"
    },
    "creditor": {
      "$ref": "https://openregister.app/schemas/Organization",
      "description": "Organization to whom the obligation is owed"
    },
    "obligationType": {
      "type": "string",
      "enum": ["invoice", "purchase-order", "contract", "standing-order"],
      "description": "Type of obligation source"
    },
    "description": {
      "type": "string",
      "description": "Details or reason for the obligation"
    },
    "settledOnTime": {
      "type": "boolean",
      "description": "Whether obligation was settled by due date"
    }
  },
  "required": ["obligationNumber", "obligationDate", "dueDate", "amount", "creditor"]
}
```

**Relations:**
- `invoiceId` (many-to-one) → Invoice
- `paymentIds` (one-to-many) → Payment
- `settlementDecisionId` (many-to-one) → SettlementDecision

---

### 2. Invoice (schema:DigitalDocument)

_Financial document detailing goods/services provided and creating an obligation for payment._

```json
{
  "name": "Invoice",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "invoiceNumber": {
      "type": "string",
      "description": "Unique invoice identifier (Dutch: factuurnummer)"
    },
    "invoiceDate": {
      "type": "string",
      "format": "date",
      "description": "Date the invoice was issued (Dutch law requires this)"
    },
    "dueDate": {
      "type": "string",
      "format": "date",
      "description": "Payment deadline date"
    },
    "grossAmount": {
      "type": "number",
      "description": "Total amount including VAT"
    },
    "vatAmount": {
      "type": "number",
      "description": "Value Added Tax amount"
    },
    "netAmount": {
      "type": "number",
      "description": "Amount excluding VAT (gross - vat)"
    },
    "vatRate": {
      "type": "number",
      "enum": [0, 6, 9, 21],
      "description": "VAT percentage (Dutch standard rates)"
    },
    "currency": {
      "type": "string",
      "default": "EUR",
      "description": "ISO 4217 currency code"
    },
    "creditor": {
      "$ref": "https://openregister.app/schemas/Organization",
      "description": "Issuing company (supplier/seller)"
    },
    "recipient": {
      "$ref": "https://openregister.app/schemas/Organization",
      "description": "Receiving company (customer/debtor)"
    },
    "lineItems": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "description": { "type": "string" },
          "quantity": { "type": "number" },
          "unitPrice": { "type": "number" },
          "amount": { "type": "number" }
        }
      },
      "description": "Invoice line items"
    },
    "paymentTerms": {
      "type": "string",
      "enum": ["net-30", "net-15", "net-60", "prepayment"],
      "description": "Payment conditions"
    },
    "documentFormat": {
      "type": "string",
      "enum": ["pdf", "xml", "ubl"],
      "description": "File format (UBL for e-invoicing)"
    },
    "paymentMethod": {
      "type": "string",
      "enum": ["sepa-transfer", "bank-transfer", "direct-debit", "cash"],
      "description": "Payment method"
    },
    "reference": {
      "type": "string",
      "description": "Purchase order number or reference number"
    }
  },
  "required": ["invoiceNumber", "invoiceDate", "dueDate", "grossAmount", "vatAmount", "netAmount", "creditor", "recipient", "paymentTerms", "documentFormat"]
}
```

**Relations:**
- `obligationId` (one-to-one) → Obligation
- `paymentIds` (one-to-many) → Payment
- `attachmentIds` (one-to-many) → File

---

### 3. SettlementDecision (schema:DigitalDocument)

_Formal decision to finalize and mark one or more obligations as settled, issued by authorized personnel._

```json
{
  "name": "SettlementDecision",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "decisionNumber": {
      "type": "string",
      "description": "Unique decision identifier (e.g., SETTLEMENT-2026-001)"
    },
    "decisionDate": {
      "type": "string",
      "format": "date",
      "description": "Date decision was issued"
    },
    "issuedBy": {
      "$ref": "https://openregister.app/schemas/Person",
      "description": "Authorized person who issued the decision"
    },
    "totalSettledAmount": {
      "$ref": "https://openregister.app/schemas/MonetaryAmount",
      "description": "Total financial amount being settled"
    },
    "obligationCount": {
      "type": "integer",
      "description": "Number of obligations included in settlement"
    },
    "decisionRationale": {
      "type": "string",
      "description": "Reason or basis for settlement decision"
    },
    "documentUrl": {
      "type": "string",
      "description": "Reference to decision document or file"
    }
  },
  "required": ["decisionNumber", "decisionDate", "issuedBy", "totalSettledAmount"]
}
```

**Relations:**
- `obligationIds` (one-to-many) → Obligation
- `complianceReportId` (many-to-one) → ComplianceReport

---

### 4. ObligationSettlement (schema:Thing)

_A formal decision record to settle and finalize an obligation, including verification of completion and approval of final amounts._

```json
{
  "name": "ObligationSettlement",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "settlementNumber": {
      "type": "string",
      "description": "Unique identifier for the settlement (e.g., SETTLE-2026-001)"
    },
    "settlementDate": {
      "type": "string",
      "format": "date-time",
      "description": "Date when the settlement was finalized"
    },
    "settledAmount": {
      "type": "number",
      "description": "Final amount settled"
    },
    "status": {
      "type": "string",
      "enum": ["draft", "approved", "finalized"],
      "description": "Current status"
    },
    "settlementType": {
      "type": "string",
      "enum": ["full", "partial", "amended"],
      "description": "Type of settlement"
    },
    "notes": {
      "type": "string",
      "description": "Additional notes or remarks"
    }
  },
  "required": ["settlementNumber", "settlementDate", "settledAmount", "status"]
}
```

**Relations:**
- `obligationId` (many-to-one) → Obligation
- `approvalRequestId` (many-to-one) → ApprovalRequest

---

### 5. ObligationTask (schema:Task)

_An automated task for managing obligation lifecycle, including AI-generated deadline tracking and compliance monitoring._

```json
{
  "name": "ObligationTask",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "taskNumber": {
      "type": "string",
      "description": "Unique identifier for the task"
    },
    "title": {
      "type": "string",
      "description": "Title of the task (e.g., 'Pay invoice INV-2026-001')"
    },
    "description": {
      "type": "string",
      "description": "Detailed description of the task"
    },
    "dueDate": {
      "type": "string",
      "format": "date-time",
      "description": "Calculated or assigned due date with deadline tracking"
    },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high", "critical"],
      "description": "Priority level based on days-to-deadline"
    },
    "status": {
      "type": "string",
      "enum": ["open", "in-progress", "completed", "overdue"],
      "description": "Current status"
    },
    "aiGenerated": {
      "type": "boolean",
      "description": "Indicates if the task was automatically generated by AI"
    }
  },
  "required": ["taskNumber", "title", "dueDate", "status"]
}
```

**Relations:**
- `obligationId` (many-to-one) → Obligation
- `assignedTo` (many-to-one) → Person

---

### 6. ComplianceReport (schema:Report)

_Analytics report tracking obligation and payment compliance metrics, supporting 99% on-time settlement performance goal and PowerBI dashboards._

```json
{
  "name": "ComplianceReport",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "reportPeriod": {
      "type": "string",
      "description": "Reporting period (e.g., 2026-Q1 or 2026-01)"
    },
    "generatedDate": {
      "type": "string",
      "format": "date",
      "description": "Date report was generated"
    },
    "complianceRate": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Percentage of obligations settled on-time (0-100)"
    },
    "totalObligations": {
      "type": "integer",
      "description": "Total obligations in period"
    },
    "onTimeObligations": {
      "type": "integer",
      "description": "Obligations settled by due date"
    },
    "overdueObligations": {
      "type": "integer",
      "description": "Obligations settled after due date"
    },
    "totalAmount": {
      "$ref": "https://openregister.app/schemas/MonetaryAmount",
      "description": "Total financial value of all obligations"
    },
    "averagePaymentDays": {
      "type": "number",
      "description": "Average days to payment after due date (negative = early)"
    },
    "powerBiUrl": {
      "type": "string",
      "description": "URL to Power BI dashboard for this report"
    }
  },
  "required": ["reportPeriod", "generatedDate", "complianceRate", "totalObligations", "onTimeObligations"]
}
```

**Relations:**
- `obligationIds` (one-to-many) → Obligation
- `paymentIds` (one-to-many) → Payment
- `settlementDecisionIds` (one-to-many) → SettlementDecision

---

### 7. FixedAsset (schema:Thing)

_A tangible business asset with long-term value subject to annual depreciation calculation and tracking._

```json
{
  "name": "FixedAsset",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "assetNumber": {
      "type": "string",
      "description": "Unique identifier for the fixed asset"
    },
    "name": {
      "type": "string",
      "description": "Name of the fixed asset"
    },
    "assetType": {
      "type": "string",
      "enum": ["equipment", "vehicle", "property", "building", "furniture", "it-hardware"],
      "description": "Type of asset"
    },
    "purchaseDate": {
      "type": "string",
      "format": "date",
      "description": "Date when the asset was purchased"
    },
    "purchaseCost": {
      "type": "number",
      "description": "Original acquisition cost of the asset"
    },
    "status": {
      "type": "string",
      "enum": ["active", "inactive", "retired"],
      "description": "Current status"
    },
    "location": {
      "type": "string",
      "description": "Physical location of the asset"
    }
  },
  "required": ["assetNumber", "name", "assetType", "purchaseDate", "purchaseCost", "status"]
}
```

**Relations:**
- `organizationId` (many-to-one) → Organization
- `deprecationScheduleIds` (one-to-many) → DepreciationSchedule

---

### 8. DepreciationSchedule (schema:Thing)

_A detailed schedule defining depreciation method, rate, and yearly calculations for a fixed asset with automated computation._

```json
{
  "name": "DepreciationSchedule",
  "context": "https://schema.org",
  "type": "object",
  "properties": {
    "scheduleNumber": {
      "type": "string",
      "description": "Unique identifier for the depreciation schedule"
    },
    "name": {
      "type": "string",
      "description": "Name or description of the depreciation schedule"
    },
    "startDate": {
      "type": "string",
      "format": "date",
      "description": "Start date of the depreciation period"
    },
    "endDate": {
      "type": "string",
      "format": "date",
      "description": "End date of the depreciation period"
    },
    "depreciationMethod": {
      "type": "string",
      "enum": ["linear", "declining-balance", "units-of-production"],
      "description": "Method used for depreciation calculation"
    },
    "annualRate": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Annual depreciation rate as a percentage"
    },
    "totalDepreciationAmount": {
      "type": "number",
      "description": "Total depreciation amount over the schedule period"
    },
    "status": {
      "type": "string",
      "enum": ["planned", "active", "completed"],
      "description": "Current status"
    }
  },
  "required": ["scheduleNumber", "name", "startDate", "depreciationMethod", "annualRate", "status"]
}
```

**Relations:**
- `fixedAssetId` (many-to-one) → FixedAsset

---

## Seed Data

Seed data loaded on app install for demo/testing purposes.

### Obligation (3 examples)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "Obligation",
    "slug": "obl-2026-inkt-supplies"
  },
  "obligationNumber": "OBL-2026-001",
  "obligationDate": "2026-05-01",
  "dueDate": "2026-06-01",
  "amount": {
    "amount": "2500.00",
    "currency": "EUR"
  },
  "creditor": {
    "name": "Inkt & Papier BV",
    "taxId": "NL123456789B01"
  },
  "obligationType": "invoice",
  "description": "Office supplies and printing materials"
}
```

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "Obligation",
    "slug": "obl-2026-office-rent"
  },
  "obligationNumber": "OBL-2026-002",
  "obligationDate": "2026-05-01",
  "dueDate": "2026-05-15",
  "amount": {
    "amount": "5000.00",
    "currency": "EUR"
  },
  "creditor": {
    "name": "Amsterdam Real Estate BV",
    "taxId": "NL987654321B01"
  },
  "obligationType": "standing-order",
  "description": "Monthly office rent - Herengracht 42, Amsterdam"
}
```

### Invoice (2 examples)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "Invoice",
    "slug": "inv-2026-services"
  },
  "invoiceNumber": "INV-2026-0001",
  "invoiceDate": "2026-05-10",
  "dueDate": "2026-06-09",
  "netAmount": "2066.12",
  "vatAmount": "433.88",
  "grossAmount": "2500.00",
  "vatRate": 21,
  "currency": "EUR",
  "creditor": {
    "name": "IT Services Utrecht",
    "taxId": "NL123456789B01"
  },
  "recipient": {
    "name": "Gemeente Amsterdam",
    "taxId": "NL654321987B02"
  },
  "paymentTerms": "net-30",
  "documentFormat": "pdf",
  "lineItems": [
    {
      "description": "Cloud hosting services (May 2026)",
      "quantity": 1,
      "unitPrice": 2066.12,
      "amount": 2066.12
    }
  ]
}
```

### FixedAsset (2 examples)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "FixedAsset",
    "slug": "fa-2026-company-vehicle"
  },
  "assetNumber": "FA-2026-001",
  "name": "Company Vehicle - Tesla Model 3",
  "assetType": "vehicle",
  "purchaseDate": "2026-01-15",
  "purchaseCost": "65000.00",
  "status": "active",
  "location": "Amsterdam Office - Parking Lot 2"
}
```

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "FixedAsset",
    "slug": "fa-2026-office-furniture"
  },
  "assetNumber": "FA-2026-002",
  "name": "Office Furniture - Workstations (10x)",
  "assetType": "furniture",
  "purchaseDate": "2025-09-01",
  "purchaseCost": "15000.00",
  "status": "active",
  "location": "Amsterdam Office - Floors 1-3"
}
```

### DepreciationSchedule (2 examples)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "DepreciationSchedule",
    "slug": "ds-2026-vehicle-linear"
  },
  "scheduleNumber": "DS-2026-001",
  "name": "Tesla Model 3 - Linear Depreciation",
  "startDate": "2026-01-15",
  "endDate": "2031-01-15",
  "depreciationMethod": "linear",
  "annualRate": 20,
  "totalDepreciationAmount": "65000.00",
  "status": "active"
}
```

### ComplianceReport (1 example)

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "ComplianceReport",
    "slug": "cr-2026-q1"
  },
  "reportPeriod": "2026-Q1",
  "generatedDate": "2026-04-01",
  "totalObligations": 15,
  "onTimeObligations": 14,
  "overdueObligations": 1,
  "complianceRate": 93.3,
  "totalAmount": {
    "amount": "125000.00",
    "currency": "EUR"
  },
  "averagePaymentDays": -2.5
}
```

## Reuse Analysis

This change leverages the following OpenRegister core services and does NOT duplicate existing functionality:

| Service | Usage | Rationale |
|---------|-------|-----------|
| **ObjectService** | Obligation/Invoice/Settlement CRUD | Standard register object lifecycle |
| **RegisterService** | Schema import and validation | Standard schema management |
| **AuditTrailService** | Automatic audit trail on all entities | Compliance requirement for financial records |
| **RelationService** | Cross-entity references (Obligation→Invoice, Settlement→Obligation) | Standard OpenRegister relation pattern |
| **FileService** | Invoice attachments, settlement documents | Standard document attachment pattern |
| **WebhookService** | Event dispatch on obligation settlement | Trigger downstream workflows (payment systems, ERP) |
| **NotificationService** | Task due date reminders, settlement approvals | Standard notification delivery |
| **AuthorizationService** | Field-level RBAC for sensitive amounts/decisions | Standard permission enforcement |
| **ImportService** / **ExportService** | Bulk obligation import (via CSV), compliance report export | Standard bulk operations |

No custom database schema, no duplicate CRUD logic, no custom permission system. All data lives in OpenRegister.

## Frontend Architecture

- **Dashboard:** `CnDashboardPage` with KPI cards (open obligations, overdue count, total value, compliance rate) + obligation list grouped by deadline + asset depreciation status
- **Obligation Index:** `CnIndexPage` with `useListView`, filter by status/creditor/dueDate, sortable by amount/days-remaining
- **Obligation Detail:** `CnDetailPage` with sections: invoice, payments, settlement decision, tasks, audit trail (via `CnObjectSidebar`)
- **Settlement Decision Workflow:** `CnFormDialog` for decision creation → `CnDetailPage` for review → approval chain integration
- **Fixed Asset Management:** Separate module with asset index, detail view, depreciation schedule editor
- **Compliance Report:** `CnChartWidget` for compliance rate trend, data table export to PowerBI

## Backend Architecture

- **ObligationController** → ObligationService → ObligationMapper (REST CRUD endpoints)
- **InvoiceController** → InvoiceService → InvoiceMapper (REST CRUD endpoints)
- **SettlementController** → SettlementService → SettlementMapper (Settlement decision workflow)
- **AssetController** → AssetService → AssetMapper (Fixed asset management)
- **ComplianceController** → ComplianceService (Report generation, export to PowerBI)
- **ObligationTaskService** → AI task generation on obligation creation (background job)
- **DepreciationCalculator** → Automatic depreciation calculation service

All controllers export REST API endpoints per ADR-002. All services include Spec traceability via `@spec` PHPDoc tags per ADR-003. All sensitive operations log to audit trail automatically.

## Next Steps

1. Implementation: Core obligation CRUD, dashboard, basic reporting (Phase 1)
2. Review: Feedback on schema design, compliance requirements
3. Testing: Unit tests for services, browser tests for workflows
4. Documentation: User guides, API documentation, compliance mapping
