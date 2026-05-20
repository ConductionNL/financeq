# Design: Budget Planning & Control — Shillinq — Other T3

## Overview

This design describes the implementation of two features — **Consolidate budgets** and **Set savings goals** — within the Shillinq Nextcloud app. Both features are built on existing entities and the OpenRegister + @conduction/nextcloud-vue platform. The design follows ADR-001 (data layer), ADR-003 (backend), ADR-004 (frontend), ADR-011 (schema standards), and ADR-012 (deduplication).

---

## Reuse Analysis

Per ADR-012, the following OpenRegister and platform services are leveraged. No duplication of existing platform capability.

| Capability | Platform service used | Custom code needed? |
|---|---|---|
| CRUD for ConsolidationGroup, ConsolidatedReport, SavingsOpportunity | `ObjectService.saveObject()`, `findObject()`, `findObjects()` | No |
| List views with pagination, filter, sort | `CnIndexPage` + `useListView` composable | No |
| Detail views with related entities | `CnDetailPage` + `CnDetailCard` + `useDetailView` | No |
| Schema-driven create/edit forms | `CnFormDialog` (auto-generated from schema) | No |
| Relation lookups (Budget → ConsolidationGroup) | `fetchUsed` / `fetchUses` via `relationsPlugin` | No |
| Notifications on status change | `NotificationService` | Trigger wiring only |
| Dashboard KPI widgets | `CnStatsBlock` + `CnChartWidget` via `CnDashboardPage` | Widget config only |
| Export (ConsolidatedReport → CSV/Excel) | `CnMassExportDialog` + `ExportService` | No |
| Audit trail | `AuditTrailService` + `CnAuditTrailTab` in `CnObjectSidebar` | No (automatic) |
| Status lifecycle transitions | `WorkflowEngineController` trigger via service | Trigger wiring only |

**Deduplication finding:** No overlap with ObjectService, RegisterService, SchemaService, or ConfigurationService. No @conduction/nextcloud-vue components duplicated. Custom code is limited to notification dispatch on SavingsOpportunity status transition and ConsolidationGroup total calculation service.

---

## Data Model

All entities are pre-existing in Shillinq. This change adds no new schemas. The relevant entity properties for this feature are described below for implementation reference.

### ConsolidationGroup
Schema.org alignment: `schema:ItemList` (a named collection of items)

| Property | Type | Required | Description |
|---|---|---|---|
| name | string | yes | Human-readable name of the consolidation group |
| description | string | no | Purpose or scope of the group |
| fiscalYear | relation → FiscalYear | yes | The fiscal year this group applies to |
| budgets | relation[] → Budget | no | Budgets included in this consolidation |
| status | enum: active, archived | yes | Lifecycle status |
| createdAt | string (date-time) | yes | ISO 8601 creation timestamp |

Cross-entity references use OpenRegister relations (register + schema + objectId). No foreign keys.

### ConsolidatedReport
Schema.org alignment: `schema:Report`

| Property | Type | Required | Description |
|---|---|---|---|
| name | string | yes | Report title (e.g. "Jaarverslag 2024") |
| consolidationGroup | relation → ConsolidationGroup | yes | The group this report covers |
| fiscalYear | relation → FiscalYear | yes | The fiscal year being reported |
| totalBudget | number | yes | Sum of all linked Budget.amount values |
| totalSpend | number | yes | Sum of all linked Budget actual spend values |
| variance | number | yes | totalBudget − totalSpend (positive = underspend) |
| reportDate | string (date) | yes | Date the report was generated |
| status | enum: draft, final | yes | Draft = editable; final = locked for export |
| generatedBy | string | no | UID of the user who generated the report |

### SavingsOpportunity
Schema.org alignment: `schema:PotentialAction` (an action with a goal/target)

| Property | Type | Required | Description |
|---|---|---|---|
| name | string | yes | Name of the savings goal |
| description | string | no | Explanation of how the savings will be realized |
| budget | relation → Budget | yes | Budget this saving is tracked against |
| targetAmount | number | yes | Savings target in base currency |
| currentAmount | number | yes | Savings realized to date |
| targetDate | string (date) | yes | Deadline for achieving the goal |
| status | enum: active, achieved, cancelled | yes | Current lifecycle status |
| responsibleUser | string | no | Nextcloud UID of the responsible person |
| progressPercent | number | no | Computed: (currentAmount / targetAmount) × 100 |

### Referenced entities (no changes made)

- **Budget** — existing entity; used via relation from ConsolidationGroup and SavingsOpportunity
- **BudgetPeriod** — existing entity; used for date-range filtering of ConsolidatedReports
- **FiscalYear** — existing entity; used as grouping dimension on ConsolidationGroup
- **CostCenter** — existing entity; used as filter dimension on ConsolidatedReport list

---

## Seed Data

Per ADR-001, 3–5 realistic Dutch seed objects per entity are required. These use the `@self` envelope and are loaded via `importFromApp()` in the repair step.

### ConsolidationGroup seed objects

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidationGroup",
      "slug": "consolidation-group-gemeentelijke-diensten-2025"
    },
    "name": "Gemeentelijke Diensten 2025",
    "description": "Consolidatie van alle dienstbudgetten voor het boekjaar 2025",
    "status": "active",
    "createdAt": "2025-01-02T09:00:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidationGroup",
      "slug": "consolidation-group-ict-digitalisering-2025"
    },
    "name": "ICT & Digitalisering 2025",
    "description": "Gebundeld overzicht van ICT-projectbudgetten en licentiekosten",
    "status": "active",
    "createdAt": "2025-01-03T10:15:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidationGroup",
      "slug": "consolidation-group-projecten-investeringen-2024"
    },
    "name": "Projecten & Investeringen 2024",
    "description": "Afgerond consolidatiegroep voor investeringsprojecten boekjaar 2024",
    "status": "archived",
    "createdAt": "2024-01-08T08:30:00Z"
  }
]
```

### ConsolidatedReport seed objects

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidatedReport",
      "slug": "consolidated-report-jaarverslag-2024"
    },
    "name": "Geconsolideerd Jaarverslag 2024",
    "totalBudget": 4250000.00,
    "totalSpend": 4087350.00,
    "variance": 162650.00,
    "reportDate": "2025-01-15",
    "status": "final",
    "generatedBy": "jansen"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidatedReport",
      "slug": "consolidated-report-kwartaal-q1-2025"
    },
    "name": "Kwartaalrapport Q1 2025 — Gemeentelijke Diensten",
    "totalBudget": 1062500.00,
    "totalSpend": 978430.00,
    "variance": 84070.00,
    "reportDate": "2025-04-05",
    "status": "final",
    "generatedBy": "devries"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ConsolidatedReport",
      "slug": "consolidated-report-h1-2025-ict"
    },
    "name": "Halfjaaroverzicht H1 2025 — ICT & Digitalisering",
    "totalBudget": 620000.00,
    "totalSpend": 541800.00,
    "variance": 78200.00,
    "reportDate": "2025-07-03",
    "status": "draft",
    "generatedBy": "bakker"
  }
]
```

### SavingsOpportunity seed objects

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "SavingsOpportunity",
      "slug": "savings-inkoop-raamcontract-2025"
    },
    "name": "Besparingen Raamcontract Kantoorartikelen",
    "description": "Door bundeling van inkoopvolumes via nieuw raamcontract verwachten we 12% kostenverlaging op kantoorartikelen.",
    "targetAmount": 48000.00,
    "currentAmount": 31500.00,
    "targetDate": "2025-12-31",
    "status": "active",
    "responsibleUser": "devries",
    "progressPercent": 65.63
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SavingsOpportunity",
      "slug": "savings-energiekosten-reductie-2025"
    },
    "name": "Energiekosten Reductie Gemeentehuis",
    "description": "Installatie LED-verlichting en slimme thermostaten reduceert energieverbruik met circa 18%.",
    "targetAmount": 95000.00,
    "currentAmount": 95000.00,
    "targetDate": "2025-10-31",
    "status": "achieved",
    "responsibleUser": "vandenberg",
    "progressPercent": 100.00
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SavingsOpportunity",
      "slug": "savings-ict-consolidatie-servers-2025"
    },
    "name": "ICT-consolidatie Serverinfrastructuur",
    "description": "Migratie naar hybride cloudoplossing vermindert on-premise serverkosten.",
    "targetAmount": 120000.00,
    "currentAmount": 0.00,
    "targetDate": "2025-12-31",
    "status": "cancelled",
    "responsibleUser": "bakker",
    "progressPercent": 0.00
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "SavingsOpportunity",
      "slug": "savings-reiskosten-thuiswerken-2025"
    },
    "name": "Reiskosten Optimalisatie via Thuiswerken",
    "description": "Verhogen hybride werken percentage van 40% naar 60% geeft structurele reiskostenbesparing.",
    "targetAmount": 36000.00,
    "currentAmount": 22800.00,
    "targetDate": "2025-12-31",
    "status": "active",
    "responsibleUser": "pietersen",
    "progressPercent": 63.33
  }
]
```

---

## API Design

All data access goes through OpenRegister's `ObjectService`. No custom REST endpoints are introduced for CRUD. The following custom endpoint is added for report generation.

### POST /api/consolidated-reports/generate
Generate (or regenerate) a ConsolidatedReport for a given ConsolidationGroup and FiscalYear. Admin or financial-manager role required.

**Request body:**
```json
{
  "consolidationGroupId": "<uuid>",
  "fiscalYearId": "<uuid>"
}
```

**Response 201:**
```json
{
  "id": "<uuid>",
  "name": "Kwartaalrapport Q2 2025 — Gemeentelijke Diensten",
  "totalBudget": 1062500.00,
  "totalSpend": 998200.00,
  "variance": 64300.00,
  "reportDate": "2025-07-01",
  "status": "draft"
}
```

**Response 422:** `{ "message": "ConsolidationGroup has no linked Budgets" }`
**Response 403:** `{ "message": "Insufficient permissions" }`

Backend route: `appinfo/routes.php` → `ConsolidationController::generate()`  
Service: `ConsolidationService::generateReport($groupId, $fiscalYearId)`

---

## Frontend Design

### Navigation additions
Two new entries in `MainMenu.vue`:
- "Consolidatiegroepen" → `/consolidation-groups` (icon: `mdi-group`)
- "Besparingsdoelen" → `/savings-opportunities` (icon: `mdi-piggy-bank`)

### Pages

**ConsolidationGroupIndex** (`src/views/ConsolidationGroupIndex.vue`)
- `CnIndexPage` with `useListView('consolidation-group', ...)`
- Columns: name, fiscalYear, status, budgetCount (count of linked budgets)
- Add button → navigate to `ConsolidationGroupDetail` with id='new'

**ConsolidationGroupDetail** (`src/views/ConsolidationGroupDetail.vue`)
- `CnDetailPage` with `CnDetailCard` sections: Properties, Linked Budgets, Consolidated Reports
- Header action: "Genereer Rapport" button → calls `POST /api/consolidated-reports/generate`
- `CnObjectSidebar` with Files, Notes, Audit Trail tabs

**ConsolidatedReportIndex** (`src/views/ConsolidatedReportIndex.vue`)
- `CnIndexPage` with `useListView('consolidated-report', ...)`
- Columns: name, fiscalYear, totalBudget, totalSpend, variance, status, reportDate
- Filter bar: by fiscalYear, by status (draft/final)

**ConsolidatedReportDetail** (`src/views/ConsolidatedReportDetail.vue`)
- `CnDetailPage` showing totals in `CnDetailGrid`
- Header action: "Exporteren" (`CnMassExportDialog`) when status = final
- "Definitief maken" button → sets status to `final` (admin only)

**SavingsOpportunityIndex** (`src/views/SavingsOpportunityIndex.vue`)
- `CnIndexPage` with `useListView('savings-opportunity', ...)`
- Columns: name, budget, targetAmount, currentAmount, progressPercent, targetDate, status
- Progress column rendered with `CnProgressBar`

**SavingsOpportunityDetail** (`src/views/SavingsOpportunityDetail.vue`)
- `CnDetailPage` + `CnDetailGrid` showing all properties
- Progress bar (`CnProgressBar`) showing currentAmount / targetAmount
- Lifecycle action buttons: "Markeer als behaald" / "Annuleren" (status transitions)
- `CnObjectSidebar` with Audit Trail tab

### Dashboard additions
Two new widgets on the Shillinq dashboard (`CnDashboardPage`):

1. **Consolidatie KPI** (`CnStatsBlock`): Total consolidated budget vs. total spend for current fiscal year; variance amount and percentage
2. **Besparingsdoelen voortgang** (`CnChartWidget`, donut): Distribution of SavingsOpportunity by status (active / achieved / cancelled) with total target amount and total current amount as subtitles

### Store registration
In `src/store/store.js`, register:
```js
objectStore.registerObjectType('consolidation-group', 'ConsolidationGroup', 'shillinq')
objectStore.registerObjectType('consolidated-report', 'ConsolidatedReport', 'shillinq')
objectStore.registerObjectType('savings-opportunity', 'SavingsOpportunity', 'shillinq')
```
Each uses `createObjectStore` with `auditTrails`, `files`, and `relations` plugins.

### Router additions
Flat named routes added to `src/router/index.js`:
```
/consolidation-groups                → ConsolidationGroupIndex
/consolidation-groups/:id            → ConsolidationGroupDetail
/consolidated-reports                → ConsolidatedReportIndex
/consolidated-reports/:id            → ConsolidatedReportDetail
/savings-opportunities               → SavingsOpportunityIndex
/savings-opportunities/:id           → SavingsOpportunityDetail
```
