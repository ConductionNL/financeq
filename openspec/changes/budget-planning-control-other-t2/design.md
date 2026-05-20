# Design: Budget Planning & Control — Other T2

**Change:** budget-planning-control-other-t2
**App:** Shillinq
**Date:** 2026-05-20

## Architecture Overview

This change extends Shillinq's OpenRegister-backed data model with budget planning and control capabilities. No new entity schemas are introduced — all domain data is stored using the pre-existing Shillinq entities (Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CostCenter, FiscalYear, etc.) as declared in the app's register.

The implementation has three layers:

1. **Declarative schema extensions** — register patches to `lib/Settings/shillinq_register.json` using `x-openregister-notifications`, `x-openregister-aggregations`, `x-openregister-calculations`, and `x-openregister-lifecycle`
2. **PHP lifecycle guards and domain services** — `BudgetValidationService` (cross-schema guard at requisition/PO save), `BudgetAmendmentTransitionGuard` (lifecycle guard on approval transition), `TaakveldValidationService` (BBV Bijlage IV code lookup), `StructuralBalanceService` (cross-budget computation)
3. **Frontend** — `CnDashboardPage` budget widgets, `CnChartWidget` actual-vs-budget visualisations, `CnIndexPage` budget list with faceted filtering by BBV programme and taakveld; all via `@conduction/nextcloud-vue`

## Reuse Analysis (ADR-012)

| Requirement | OpenRegister / nextcloud-vue service used | Custom code? |
|---|---|---|
| Budget CRUD | `ObjectService.saveObject()` / `deleteObject()` | No |
| Budget list + filtering | `ObjectService.findAll()` + `CnDataTable` + `CnFacetSidebar` | No |
| Budget detail view | `CnDetailPage` + `CnDetailCard` | No |
| Audit trail on all changes | `AuditTrailService` (automatic) | No |
| Threshold alert notifications | `x-openregister-notifications` (declarative) | No |
| Actual-vs-budget aggregations | `x-openregister-aggregations` (declarative) | No |
| Utilisation %, remaining budget | `x-openregister-calculations` (declarative) | No |
| BudgetAmendment lifecycle | `x-openregister-lifecycle` (declarative) | No |
| Approval request routing | Existing `ApprovalChain` + `ApprovalRequest` schemas | No |
| CSV/Excel export (IV3 report) | `CnMassExportDialog` + `ExportService` | No |
| PDF export (structural balance) | `ExportService` PDF path | No |
| Dashboard widgets | `CnDashboardPage` + `CnChartWidget` + `CnStatsBlock` | No |
| Budget validation at PO/requisition save | `BudgetValidationService` PHP guard | **Yes** — cross-schema guard on save event (see Declarative-vs-Imperative) |
| Taakveld code validation | `TaakveldValidationService` PHP service | **Yes** — static code list lookup not expressible in OR schema |
| Structural balance check | `StructuralBalanceService` PHP service | **Yes** — cross-budget computation OR aggregations cannot span schemas |

**Deduplication findings:**
- Searched `openregister/lib/Service/` and shared `openspec/specs/` — no `BudgetValidationService`, `TaakveldValidationService`, or `StructuralBalanceService` found
- `ObjectService`, `RegisterService`, `SchemaService`, `ConfigurationService` checked — no budget-domain overlap
- `@conduction/nextcloud-vue` covers all dashboard, chart, form, and list requirements — no custom chart or form components needed
- **Finding:** No duplication detected. Custom PHP services justified by cross-schema guard patterns and Dutch regulatory code lists not addressable by OR declarative engine.

## Declarative-vs-Imperative Decisions (ADR-031)

| Behaviour | Decision | Rationale |
|---|---|---|
| **Threshold alert notifications** (warning, critical) | **Declarative** — `x-openregister-notifications` on Budget schema | OR's notification extension supports threshold triggers on calculated fields; recipient resolver targets budgetOwner relation + role:financial-controller |
| **Actual spend aggregation** | **Declarative** — `x-openregister-aggregations` on Budget schema | Declarative sum of SpendingRecord.amount filtered by budgetId — no external data, no side effects |
| **Committed amount aggregation** | **Declarative** — `x-openregister-aggregations` on Budget schema | Declarative sum of non-cancelled PurchaseOrders linked to budget |
| **Utilisation %, remaining budget** | **Declarative** — `x-openregister-calculations` on Budget schema | Pure derived fields from ceiling, actualSpend, committedAmount — expressible as arithmetic calculations |
| **BudgetAmendment approval lifecycle** | **Declarative** — `x-openregister-lifecycle` on BudgetAmendment schema | Standard draft→submitted→approved/rejected lifecycle; guard handles ceiling-update side-effect |
| **Budget validation at PO/requisition save** | **Imperative** — `BudgetValidationService` PHP guard | OR lifecycle guards operate on the object being saved, not on cross-schema commitment totals. Computing remaining budget requires reading the Budget object, summing existing POs, and comparing to the new document amount — a multi-object read that the lifecycle engine cannot express as a precondition on the requisition/PO schema. Exception: cross-schema guard shape, ADR-031 §"PHP guards remain a legitimate seam". |
| **Taakveld code validation** | **Imperative** — `TaakveldValidationService` | Validates against a static official code list (BBV Bijlage IV) stored in `IAppConfig`. No OR schema extension covers static reference-data validation against an external code list. |
| **Structural balance check** | **Imperative** — `StructuralBalanceService` | Requires summing BudgetAllocations across multiple Budget objects, classified by IV3 revenue/expenditure type. OR aggregations are per-schema (within one object's relations) and cannot perform cross-budget joins. Exception: cross-schema aggregation not supported by OR engine. |
| **Indexation copy to new fiscal year** | **Imperative** — `BudgetIndexationService` method | Creating a new Budget and duplicating its BudgetAllocations with adjusted amounts is a bulk object-creation operation; no OR declarative extension covers this pattern. |

## Entity Usage

All entities are pre-existing in the Shillinq data model. This change adds optional, non-breaking schema fields (additions only per ADR-011).

### Budget

Core entity representing a financial budget for a fiscal period.

New fields added (non-breaking additions):

| Field | Type | Default | Description |
|---|---|---|---|
| `alertThreshold` | integer (0–100) | 90 | Critical alert fires when utilisation reaches this % |
| `warningThreshold` | integer (0–100) | 75 | Warning alert fires when utilisation reaches this % |
| `overBudgetPrevention` | boolean | true | Block requisitions/POs that would exhaust the budget |
| `bbvProgramme` | string | — | BBV programme code (Dutch municipalities) |
| `allocationPeriod` | enum: monthly/quarterly/yearly/custom | yearly | Period granularity for auto-generating BudgetPeriods |
| `indexationPercentage` | number | — | % increase applied when copying to a new fiscal year |
| `budgetType` | enum: opex/capex | opex | Capital vs operational budget classification |

Derived via `x-openregister-calculations`: `utilisation`, `remaining`
Aggregated via `x-openregister-aggregations`: `actualSpend`, `committedAmount`

### BudgetAllocation

Links a Budget to spending dimensions (CostCenter, CostProject, GeneralLedgerAccount).

New fields added (non-breaking additions):

| Field | Type | Default | Description |
|---|---|---|---|
| `iv3Category` | string | — | IV3 reporting category code |
| `taakveld` | string | — | Dutch BBV taakveld classification code |
| `structural` | boolean | false | Marks as structurally recurring for balance check |
| `forecastAmount` | number | — | Year-end prognose amount |

### BudgetAmendment

Records requests to change a budget ceiling or reallocate amounts. Existing entity; `x-openregister-lifecycle` added.

New lifecycle states: `draft → submitted → approved | rejected`

### BudgetPeriod

Monthly/quarterly/yearly sub-period of a Budget. Auto-generated when `allocationPeriod` is set. Existing entity; no new fields required.

## Schema Register Patches (lib/Settings/shillinq_register.json)

### Budget schema — new properties

```json
{
  "alertThreshold": {
    "type": "integer", "minimum": 0, "maximum": 100, "default": 90,
    "description": "Percentage at which a critical alert fires"
  },
  "warningThreshold": {
    "type": "integer", "minimum": 0, "maximum": 100, "default": 75,
    "description": "Percentage at which a warning alert fires"
  },
  "overBudgetPrevention": {
    "type": "boolean", "default": true,
    "description": "Block requisitions and POs that would exceed the budget ceiling"
  },
  "bbvProgramme": {
    "type": "string",
    "description": "BBV programme code, e.g. '6 - Sociaal domein'"
  },
  "allocationPeriod": {
    "type": "string", "enum": ["monthly", "quarterly", "yearly", "custom"],
    "default": "yearly"
  },
  "indexationPercentage": {
    "type": "number",
    "description": "Percentage increase applied when copying budget to a new fiscal year"
  },
  "budgetType": {
    "type": "string", "enum": ["opex", "capex"], "default": "opex"
  }
}
```

### BudgetAllocation schema — new properties

```json
{
  "iv3Category": {
    "type": "string",
    "description": "IV3 reporting category code (A=Bestuur, B=Personeel, C=Goederen en diensten, D=Rente, E=Afschrijvingen, F=Overdrachten, G=Grondexploitaties)"
  },
  "taakveld": {
    "type": "string",
    "description": "Dutch BBV taakveld classification code, e.g. '6.72'"
  },
  "structural": {
    "type": "boolean", "default": false,
    "description": "True for structurally recurring allocations used in structural balance check"
  },
  "forecastAmount": {
    "type": "number",
    "description": "Year-end forecast (prognose) amount"
  }
}
```

### x-openregister-notifications on Budget schema

```json
{
  "x-openregister-notifications": [
    {
      "id": "budget-warning",
      "trigger": "calculation",
      "condition": "utilisation >= warningThreshold AND utilisation < alertThreshold",
      "deduplication": "once-per-band",
      "recipients": ["relation:budgetOwner", "role:financial-controller"],
      "template": "Budgetwaarschuwing: {{ utilisation }}% besteed van {{ name }}"
    },
    {
      "id": "budget-critical",
      "trigger": "calculation",
      "condition": "utilisation >= alertThreshold",
      "deduplication": "once-per-band",
      "recipients": ["relation:budgetOwner", "role:financial-controller"],
      "template": "Budget kritisch: {{ utilisation }}% besteed van {{ name }}"
    }
  ]
}
```

### x-openregister-aggregations on Budget schema

```json
{
  "x-openregister-aggregations": [
    {
      "id": "actualSpend",
      "type": "sum",
      "source": "SpendingRecord",
      "filter": "budgetId = @self.id",
      "field": "amount"
    },
    {
      "id": "committedAmount",
      "type": "sum",
      "source": "PurchaseOrder",
      "filter": "budgetId = @self.id AND status != 'cancelled'",
      "field": "totalAmount"
    }
  ]
}
```

### x-openregister-calculations on Budget schema

```json
{
  "x-openregister-calculations": [
    {
      "id": "utilisation",
      "formula": "ceiling > 0 ? (actualSpend / ceiling) * 100 : 0",
      "type": "number",
      "description": "Actual spend as percentage of budget ceiling"
    },
    {
      "id": "remaining",
      "formula": "ceiling - actualSpend - committedAmount",
      "type": "number",
      "description": "Available budget: ceiling minus spent and committed"
    }
  ]
}
```

### x-openregister-lifecycle on BudgetAmendment schema

```json
{
  "x-openregister-lifecycle": {
    "initialState": "draft",
    "states": ["draft", "submitted", "approved", "rejected"],
    "transitions": [
      {
        "from": "draft", "to": "submitted",
        "label": "Indienen",
        "requires": "OCA\\Shillinq\\Lifecycle\\BudgetAmendmentTransitionGuard"
      },
      {
        "from": "submitted", "to": "approved",
        "label": "Goedkeuren",
        "role": "financial-controller"
      },
      {
        "from": "submitted", "to": "rejected",
        "label": "Afwijzen",
        "role": "financial-controller"
      },
      {
        "from": "approved", "to": "draft",
        "label": "Herzien",
        "role": "financial-controller"
      }
    ]
  }
}
```

## PHP Services

### BudgetValidationService (`lib/Service/BudgetValidationService.php`)

Called as an event listener on `PurchaseRequisitionSaveEvent` and `PurchaseOrderSaveEvent` before the object is persisted.

```
validate(string $documentType, array $documentData): void
  1. Locate active Budget via CostCenter + GeneralLedgerAccount from document dimensions
  2. If no Budget found: return (no constraint — requisitions without a budget are allowed unless admin config prohibits it)
  3. Read Budget.remaining (= ceiling - actualSpend - committedAmount, via OR calculations)
  4. If documentData['totalAmount'] > remaining AND budget.overBudgetPrevention == true:
       throw BudgetExceededException($budget, $overage)
     Else if documentData['totalAmount'] > remaining:
       Patch document with overBudget=true, budgetOverage=($documentData['totalAmount'] - remaining)
```

### BudgetAmendmentTransitionGuard (`lib/Lifecycle/BudgetAmendmentTransitionGuard.php`)

Implements OR lifecycle guard interface. Called on `submitted → approved` transition.

On approval: reads the linked Budget via `budgetId` relation, adds `amendment.amount` to `Budget.ceiling`, saves via `ObjectService.saveObject()`.

### TaakveldValidationService (`lib/Service/TaakveldValidationService.php`)

```
validate(string $budgetId): array  // returns list of validation issues
  1. Load all BudgetAllocations where budgetId = $budgetId
  2. Load BBV Bijlage IV code list from IAppConfig
  3. For each allocation: check taakveld is non-empty AND exists in code list
  4. Return [{ allocationId, allocationName, issue: 'missing'|'invalid', value }]
```

### StructuralBalanceService (`lib/Service/StructuralBalanceService.php`)

```
compute(string $budgetId): array  // { revenues, expenditures, balance, balanced, byProgramme[] }
  1. Load all BudgetAllocations where budgetId = $budgetId AND structural = true
  2. Classify each allocation as revenue or expenditure via IV3 category
     (A, B, C, D, E = expenditure; F when negative = revenue; configurable mapping)
  3. revenues = sum of revenue allocations ceiling
  4. expenditures = sum of expenditure allocations ceiling
  5. Return balance = revenues - expenditures, balanced = balance >= 0
  6. Group results by bbvProgramme for programme-level report
```

## Frontend Components

All frontend uses `@conduction/nextcloud-vue` components per ADR-004. No custom chart, form, or pagination components.

| View | Components | Notes |
|---|---|---|
| Budget list | `CnIndexPage` + `CnDataTable` + `CnFacetSidebar` | Facets: BBV programme, taakveld, allocationPeriod, fiscal year, status, budgetType |
| Budget detail | `CnDetailPage` + `CnDetailCard` | Sections: ceiling summary, allocations table, amendments list, spend breakdown |
| Budget dashboard | `CnDashboardPage` + `CnStatsBlock` × 4 + `CnChartWidget` × 2 | KPIs: total budget, total committed, total actuals, over-budget count |
| Budget form | `CnFormDialog` (schema-driven) | New fields auto-appear from schema; Dutch labels via i18n |
| Amendment workflow | `CnTimelineStages` | States: Concept → Ingediend → Goedgekeurd/Afgewezen |
| Taakveld validation result | `CnDetailCard` inline list | Validation issues per allocation with inline navigation |
| Structural balance report | `CnDetailCard` + `CnChartWidget` (bar) | Revenue vs expenditure per BBV programme |
| Kadernota overview | `CnDetailGrid` + `CnMassExportDialog` | Multi-year table; Excel export |

Router entries (flat per ADR-004):
- `/budgets` → `BudgetIndex`
- `/budgets/:id` → `BudgetDetail`
- `/budget-amendments` → `BudgetAmendmentIndex`
- `/budget-amendments/:id` → `BudgetAmendmentDetail`

## Seed Data

Seed objects using `@self` envelope for `lib/Settings/shillinq_register.json`.
General Dutch organisation data — municipality and consultancy contexts per ADR-001.

### Budget (4 objects)

```json
{
  "@self": { "register": "shillinq", "schema": "Budget", "slug": "budget-gemeente-utrecht-ict-2026" },
  "name": "ICT-budget Gemeente Utrecht 2026",
  "ceiling": 450000,
  "fiscalYear": "2026",
  "allocationPeriod": "quarterly",
  "alertThreshold": 90,
  "warningThreshold": 75,
  "overBudgetPrevention": true,
  "bbvProgramme": "0 - Bestuur en ondersteuning",
  "budgetType": "opex",
  "status": "active",
  "currency": "EUR"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "Budget", "slug": "budget-gemeente-rotterdam-sociaal-2026" },
  "name": "Sociaal Domein budget Gemeente Rotterdam 2026",
  "ceiling": 2800000,
  "fiscalYear": "2026",
  "allocationPeriod": "yearly",
  "alertThreshold": 95,
  "warningThreshold": 80,
  "overBudgetPrevention": true,
  "bbvProgramme": "6 - Sociaal domein",
  "budgetType": "opex",
  "status": "active",
  "currency": "EUR"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "Budget", "slug": "budget-conduction-rd-2026" },
  "name": "R&D Budget Conduction BV 2026",
  "ceiling": 180000,
  "fiscalYear": "2026",
  "allocationPeriod": "quarterly",
  "alertThreshold": 85,
  "warningThreshold": 70,
  "overBudgetPrevention": false,
  "budgetType": "opex",
  "status": "active",
  "currency": "EUR"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "Budget", "slug": "budget-gemeente-eindhoven-werken-2026" },
  "name": "Openbare Werken Gemeente Eindhoven 2026",
  "ceiling": 3200000,
  "fiscalYear": "2026",
  "allocationPeriod": "quarterly",
  "alertThreshold": 90,
  "warningThreshold": 80,
  "overBudgetPrevention": true,
  "bbvProgramme": "2 - Openbaar gebied en mobiliteit",
  "budgetType": "opex",
  "status": "active",
  "currency": "EUR"
}
```

### BudgetAllocation (4 objects)

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "balloc-utrecht-ict-infra-2026" },
  "name": "ICT Infrastructuur 2026",
  "ceiling": 180000,
  "iv3Category": "C",
  "taakveld": "0.4",
  "structural": true,
  "allocationPeriod": "quarterly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "balloc-utrecht-ict-licenties-2026" },
  "name": "Softwarelicenties en abonnementen 2026",
  "ceiling": 95000,
  "iv3Category": "C",
  "taakveld": "0.4",
  "structural": true,
  "allocationPeriod": "yearly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "balloc-rotterdam-jeugdzorg-2026" },
  "name": "Jeugdzorg inkoop 2026",
  "ceiling": 1200000,
  "iv3Category": "F",
  "taakveld": "6.72",
  "structural": true,
  "allocationPeriod": "quarterly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "balloc-eindhoven-wegonderhoud-2026" },
  "name": "Wegonderhoud en reparaties 2026",
  "ceiling": 850000,
  "iv3Category": "C",
  "taakveld": "2.1",
  "structural": true,
  "allocationPeriod": "quarterly"
}
```

### BudgetAmendment (3 objects)

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAmendment", "slug": "bamend-utrecht-ict-verhoging-001" },
  "name": "Plafondverhoging ICT Q3 2026",
  "type": "ceilingIncrease",
  "amount": 35000,
  "justification": "Extra serverinvesteringen vanwege capaciteitsuitbreiding cloudopslag",
  "status": "submitted",
  "requestDate": "2026-07-15"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAmendment", "slug": "bamend-rotterdam-sd-herziening-001" },
  "name": "Herziening Sociaal Domein Q2 2026 — Wmo naar Jeugdzorg",
  "type": "reallocation",
  "amount": 75000,
  "justification": "Verschuiving van Wmo-budget naar Jeugdzorg na hogere instroom Q1",
  "status": "approved",
  "requestDate": "2026-04-10",
  "approvalDate": "2026-04-18"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetAmendment", "slug": "bamend-eindhoven-werken-stormschade-001" },
  "name": "Aanvullend budget wegonderhoud stormschade 2026",
  "type": "ceilingIncrease",
  "amount": 120000,
  "justification": "Stormschade reparaties NW woonwijk — onvoorziene herstelkosten na storm van 14 april",
  "status": "draft",
  "requestDate": "2026-05-10"
}
```

### BudgetPeriod (4 objects)

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "bperiod-utrecht-ict-q1-2026" },
  "name": "Q1 2026 — ICT Utrecht",
  "startDate": "2026-01-01",
  "endDate": "2026-03-31",
  "ceiling": 112500,
  "periodType": "quarterly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "bperiod-utrecht-ict-q2-2026" },
  "name": "Q2 2026 — ICT Utrecht",
  "startDate": "2026-04-01",
  "endDate": "2026-06-30",
  "ceiling": 112500,
  "periodType": "quarterly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "bperiod-rotterdam-sd-jaar-2026" },
  "name": "Boekjaar 2026 — Sociaal Domein Rotterdam",
  "startDate": "2026-01-01",
  "endDate": "2026-12-31",
  "ceiling": 2800000,
  "periodType": "yearly"
}
```

```json
{
  "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "bperiod-eindhoven-werken-q1-2026" },
  "name": "Q1 2026 — Openbare Werken Eindhoven",
  "startDate": "2026-01-01",
  "endDate": "2026-03-31",
  "ceiling": 800000,
  "periodType": "quarterly"
}
```
