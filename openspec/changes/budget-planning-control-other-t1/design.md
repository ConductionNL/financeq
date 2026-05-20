# Design: Budget Planning & Control — Shillinq — Other T1

## Architecture Overview

This change implements the budget planning and control module on top of existing Shillinq entities. All domain data flows through OpenRegister objects (ADR-001). Custom PHP code is limited to domain-specific business logic that the OR schema engine cannot express declaratively.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Frontend (Vue 2 + Pinia + @conduction/nextcloud-vue)               │
│                                                                     │
│  BudgetDashboard ─── CnDashboardPage + CnStatsBlock + CnChartWidget │
│  BudgetIndex      ─── CnIndexPage + useListView                     │
│  BudgetDetail     ─── CnDetailPage + CnDetailCard + CnObjectSidebar │
│  ExpenditureIndex ─── CnIndexPage + CnTimelineStages                │
│  BudgetReportPage ─── CnTableWidget + CnChartWidget                 │
└─────────────────────────┬───────────────────────────────────────────┘
                          │ REST /api/{resource}
┌─────────────────────────▼───────────────────────────────────────────┐
│  Controllers (thin — routing + validation + response only)          │
│                                                                     │
│  BudgetController          BudgetPeriodController                   │
│  BudgetAllocationController  ExpenditureRequestController           │
│  BudgetReportController                                             │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────────┐
│  Services (custom business logic — imperative exceptions only)      │
│                                                                     │
│  BudgetValidationService   — pre-purchase guard (lifecycle seam)    │
│  BudgetReportService       — comparative period aggregation         │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────────┐
│  OpenRegister (schema-declarative business logic)                   │
│                                                                     │
│  Budget lifecycle         → x-openregister-lifecycle                │
│  ExpenditureRequest lifecycle → x-openregister-lifecycle            │
│  Budget calculations      → x-openregister-calculations             │
│  Budget notifications     → x-openregister-notifications            │
│  BudgetAllocation notifications → x-openregister-notifications      │
└─────────────────────────────────────────────────────────────────────┘
```

## Declarative-vs-Imperative Decision

Per ADR-031, every behaviour is evaluated for declarative fitness before writing PHP.

| Behaviour | Path | Rationale |
|---|---|---|
| Budget status lifecycle (concept → ingediend → goedgekeurd → actief → gesloten) | **Declarative** — `x-openregister-lifecycle` on `Budget` | Standard lifecycle with state-guarded transitions; no external dependencies; full audit trail + RBAC per state via OR engine |
| ExpenditureRequest status lifecycle (ingediend → in_behandeling → goedgekeurd / afgewezen) | **Declarative** — `x-openregister-lifecycle` on `ExpenditureRequest` | Standard approval lifecycle; declarative guard suffices |
| BudgetAmendment status lifecycle (concept → aangevraagd → goedgekeurd / afgewezen) | **Declarative** — `x-openregister-lifecycle` on `BudgetAmendment` | Amendment request workflow; standard approval shape |
| Budget utilisation percentage | **Declarative** — `x-openregister-calculations` on `Budget` | Derived field: `(committed + spent) / amount * 100`; fresh on every read |
| Remaining budget amount | **Declarative** — `x-openregister-calculations` on `Budget` | Derived field: `amount - committed - spent` |
| Forecast contract budget exhaustion date | **Declarative** — `x-openregister-calculations` on `Budget` | Derived: `today + (remaining / burnRate * 30)` from spend history |
| Budget threshold alert notifications | **Declarative** — `x-openregister-notifications` on `Budget` | Configurable threshold; recipient resolver (budget holder + finance manager); no external system |
| Overspend alert notifications | **Declarative** — `x-openregister-notifications` on `BudgetAllocation` | Trigger when utilisation > 100%; email + NC notification |
| Pre-purchase budget validation blocking over-budget requisitions | **Imperative** — `BudgetValidationService` (lifecycle guard) | ADR-031 exception: domain rule guard. Requires real-time query across multiple PurchaseRequisition and ExpenditureRequest objects to compute committed amounts; multi-object join logic the OR engine cannot express in a single guard field. Called as `requires: OCA\Shillinq\Lifecycle\BudgetAvailabilityGuard` from the ExpenditureRequest lifecycle |
| Fiscal year rollover with carryforward and reallocation | **Imperative** — OR `ScheduledWorkflow` + n8n adapter | ADR-031 exception: scheduled bulk work. Walks all BudgetAllocation objects for a closing FiscalYear, applies carryforward rules, and creates new BudgetAllocation objects for the new year. Expressed as a `ScheduledWorkflow` entity with n8n workflow handling the logic; no per-app Job class |
| Management reporting with comparative period analysis | **Imperative** — `BudgetReportService` | ADR-031 exception: complex cross-schema aggregation spanning Budget, BudgetAllocation, BudgetPeriod, GeneralLedgerEntry, and ExpenditureRequest across multiple periods. The OR aggregation engine cannot yet express multi-period cross-schema variance analysis. Gap filed as feature request against openregister |

## Controllers

All controllers are thin (<10 lines/method), use constructor DI with `private readonly`, and follow ADR-003.

| Controller | Routes | Method |
|---|---|---|
| `BudgetController` | `GET/POST /api/budgets`, `GET/PUT/DELETE /api/budgets/{id}` | index, create, show, update, destroy |
| `BudgetPeriodController` | `GET/POST /api/budget-periods`, `GET/PUT/DELETE /api/budget-periods/{id}` | index, create, show, update, destroy |
| `BudgetAllocationController` | `GET/POST /api/budget-allocations`, `GET/PUT/DELETE /api/budget-allocations/{id}` | index, create, show, update, destroy |
| `ExpenditureRequestController` | `GET/POST /api/expenditure-requests`, `GET/PUT/DELETE /api/expenditure-requests/{id}` | index, create, show, update, destroy |
| `BudgetReportController` | `GET /api/budget-reports/variance`, `GET /api/budget-reports/utilisation` | variance, utilisation |

## Services (Imperative Exceptions)

### BudgetValidationService

Lifecycle guard called by the OR engine when an ExpenditureRequest transitions from `concept` to `ingediend`. Queries all open ExpenditureRequest and PurchaseRequisition objects against the relevant BudgetAllocation to compute the real-time available balance. Returns `true` (proceed) or throws `BudgetExceededException` (block).

Implements `OCA\Shillinq\Lifecycle\BudgetAvailabilityGuard` (single `check(array $context): bool` method).

### BudgetReportService

Generates comparative period reports by querying Budget, BudgetAllocation, BudgetPeriod, and GeneralLedgerEntry objects across two user-specified periods. Returns structured data consumed by `BudgetReportController` for the management reporting API endpoints.

Methods:
- `getVarianceReport(string $periodId, string $comparePeriodId): array`
- `getUtilisationReport(string $periodId): array`

## Frontend Pages

All pages follow ADR-004 patterns. No custom stores — `createObjectStore` with plugins for each entity type.

| Page | Component | Route |
|---|---|---|
| Budget Dashboard | `CnDashboardPage` (4 KPI blocks + utilisation chart + "My Requests" list) | `/` |
| Budget List | `CnIndexPage` with `useListView` | `/budgets` |
| Budget Detail | `CnDetailPage` + `CnObjectSidebar` | `/budgets/:id` |
| Budget Period List | `CnIndexPage` | `/budget-periods` |
| Expenditure Request List | `CnIndexPage` + `CnTimelineStages` | `/expenditure-requests` |
| Expenditure Request Detail | `CnDetailPage` + `CnObjectSidebar` | `/expenditure-requests/:id` |
| Budget Allocation List | `CnIndexPage` | `/budget-allocations` |
| Budget Report | Custom report page using `CnTableWidget` + `CnChartWidget` | `/budget-reports` |
| Settings | `CnVersionInfoCard` → `CnRegisterMapping` → `CnSettingsSection` | `/settings` |

## Schema Register Patches

The following declarative blocks are added to `lib/Settings/shillinq_register.json`:

### Budget — lifecycle

```json
"x-openregister-lifecycle": {
  "property": "status",
  "states": ["concept", "ingediend", "goedgekeurd", "actief", "gesloten"],
  "transitions": [
    { "from": "concept",       "to": "ingediend",    "label": "Indienen" },
    { "from": "ingediend",     "to": "goedgekeurd",  "label": "Goedkeuren",  "requires": "OCA\\Shillinq\\Lifecycle\\BudgetApprovalGuard" },
    { "from": "ingediend",     "to": "concept",      "label": "Terugsturen" },
    { "from": "goedgekeurd",   "to": "actief",       "label": "Activeren" },
    { "from": "actief",        "to": "gesloten",     "label": "Sluiten" }
  ]
}
```

### Budget — calculations

```json
"x-openregister-calculations": [
  {
    "property": "utilisatiePercentage",
    "expression": "(committed + spent) / amount * 100",
    "type": "number"
  },
  {
    "property": "resterendBudget",
    "expression": "amount - committed - spent",
    "type": "number"
  },
  {
    "property": "verwachteUitputtingsdatum",
    "expression": "today + (resterendBudget / burnRate30Dagen * 30)",
    "type": "date"
  }
]
```

### Budget — notifications (threshold breach)

```json
"x-openregister-notifications": [
  {
    "trigger": "utilisatiePercentage >= drempelPercentage",
    "recipients": ["budgethouder", "financieel_beheerder"],
    "channels": ["nextcloud", "email"],
    "subject": "Budget drempel bereikt: {name}",
    "body": "Het budget '{name}' heeft {utilisatiePercentage}% benut (drempel: {drempelPercentage}%)."
  },
  {
    "trigger": "utilisatiePercentage >= 100",
    "recipients": ["budgethouder", "financieel_beheerder", "controller"],
    "channels": ["nextcloud", "email"],
    "subject": "Budget overschreden: {name}",
    "body": "Het budget '{name}' is overschreden. Huidig verbruik: {utilisatiePercentage}%."
  }
]
```

### ExpenditureRequest — lifecycle

```json
"x-openregister-lifecycle": {
  "property": "status",
  "states": ["concept", "ingediend", "in_behandeling", "goedgekeurd", "afgewezen"],
  "transitions": [
    { "from": "concept",         "to": "ingediend",       "label": "Indienen",    "requires": "OCA\\Shillinq\\Lifecycle\\BudgetAvailabilityGuard" },
    { "from": "ingediend",       "to": "in_behandeling",  "label": "In behandeling nemen" },
    { "from": "in_behandeling",  "to": "goedgekeurd",     "label": "Goedkeuren" },
    { "from": "in_behandeling",  "to": "afgewezen",       "label": "Afwijzen" },
    { "from": "ingediend",       "to": "concept",         "label": "Terugsturen" }
  ]
}
```

## Reuse Analysis

Per ADR-012, the following OpenRegister and platform services are leveraged — no overlap with the capabilities below requires new core code:

| Capability | Reused Service / Component |
|---|---|
| All CRUD operations | `ObjectService.saveObject()`, `findAll()`, `deleteObject()` |
| Budget list + search | `CnIndexPage` + `useListView` + `CnDataTable` + `CnFilterBar` |
| Budget detail view | `CnDetailPage` + `CnDetailCard` + `CnDetailGrid` |
| Forms (create/edit) | `CnFormDialog` (schema-driven, auto-generates from OR schema) |
| File attachments on expenditure requests | `FileService` + `CnObjectSidebar` → `CnFilesTab` |
| Audit trail | `AuditTrailService` + `CnObjectSidebar` → `CnAuditTrailTab` (automatic) |
| Budget dashboard KPIs | `CnDashboardPage` + `CnStatsBlock` + `CnKpiGrid` |
| Utilisation charts | `CnChartWidget` (ApexCharts area/bar/donut) |
| Threshold notifications | `NotificationService` via `x-openregister-notifications` |
| Budget lifecycle | `WorkflowEngineRegistry` via `x-openregister-lifecycle` |
| Approval routing | `ApprovalChain` + `ApprovalRequest` entities (existing) |
| RBAC enforcement | `AuthorizationService` + `PropertyRbacHandler` (automatic) |
| Mass export (budget data) | `ExportService` + `CnMassExportDialog` |
| Fiscal year rollover scheduling | `ScheduledWorkflow` entity + OR `ScheduledWorkflowJob` + n8n |
| Pagination + search state | `useListView` composable |
| Delete/confirm dialogs | `CnDeleteDialog` |

No new abstractions are needed. The two imperative services (`BudgetValidationService`, `BudgetReportService`) cover only domain-specific logic with no OR equivalent.

## Seed Data

Seed objects use `@self` envelope per ADR-001. All values are Dutch, general-purpose (municipality + consultancy). Register key: `shillinq`.

### Budget (5 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "Budget", "slug": "gemeente-amstelveen-ict-2025" },
    "name": "ICT Infrastructuur 2025",
    "reference": "BEG-2025-ICT-001",
    "status": "actief",
    "amount": 485000.00,
    "spent": 142300.00,
    "committed": 87500.00,
    "drempelPercentage": 80,
    "omschrijving": "Jaarbudget voor ICT-infrastructuur, licenties en ondersteuning",
    "afdeling": "Informatie & Automatisering",
    "kostenplaats": "KP-7210"
  },
  {
    "@self": { "register": "shillinq", "schema": "Budget", "slug": "gemeente-amstelveen-wmo-2025" },
    "name": "WMO Individuele Begeleiding 2025",
    "reference": "BEG-2025-WMO-002",
    "status": "actief",
    "amount": 2150000.00,
    "spent": 1342000.00,
    "committed": 315000.00,
    "drempelPercentage": 85,
    "omschrijving": "Budget Wet Maatschappelijke Ondersteuning — individuele begeleiding",
    "afdeling": "Maatschappelijke Zaken",
    "kostenplaats": "KP-6510"
  },
  {
    "@self": { "register": "shillinq", "schema": "Budget", "slug": "conduction-projectbudget-2025" },
    "name": "Projectbudget Digitalisering 2025",
    "reference": "BEG-2025-DIG-003",
    "status": "goedgekeurd",
    "amount": 320000.00,
    "spent": 0.00,
    "committed": 45000.00,
    "drempelPercentage": 90,
    "omschrijving": "Budget voor digitaliseringsprojecten en -advies",
    "afdeling": "Business Development",
    "kostenplaats": "KP-4400"
  },
  {
    "@self": { "register": "shillinq", "schema": "Budget", "slug": "gemeente-amstelveen-openbare-ruimte-2025" },
    "name": "Openbare Ruimte — Onderhoud 2025",
    "reference": "BEG-2025-OR-004",
    "status": "actief",
    "amount": 1780000.00,
    "spent": 923400.00,
    "committed": 412000.00,
    "drempelPercentage": 80,
    "omschrijving": "Budget voor onderhoud openbare ruimte, groen en wegen",
    "afdeling": "Beheer Openbare Ruimte",
    "kostenplaats": "KP-5500"
  },
  {
    "@self": { "register": "shillinq", "schema": "Budget", "slug": "stichting-buurtwerk-activiteiten-2025" },
    "name": "Buurtactiviteiten 2025",
    "reference": "BEG-2025-BA-005",
    "status": "ingediend",
    "amount": 48500.00,
    "spent": 0.00,
    "committed": 0.00,
    "drempelPercentage": 75,
    "omschrijving": "Jaarbudget voor wijkactiviteiten en buurtinitiatieven",
    "afdeling": "Programma's & Projecten",
    "kostenplaats": "KP-3200"
  }
]
```

### BudgetPeriod (3 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "boekjaar-2025" },
    "name": "Boekjaar 2025",
    "type": "annual",
    "startDate": "2025-01-01",
    "endDate": "2025-12-31",
    "status": "open",
    "omschrijving": "Standaard gemeentelijk boekjaar 2025"
  },
  {
    "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "boekjaar-2024" },
    "name": "Boekjaar 2024",
    "type": "annual",
    "startDate": "2024-01-01",
    "endDate": "2024-12-31",
    "status": "gesloten",
    "omschrijving": "Afgesloten boekjaar 2024 — jaarrekening vastgesteld"
  },
  {
    "@self": { "register": "shillinq", "schema": "BudgetPeriod", "slug": "q1-2025" },
    "name": "Kwartaal 1 — 2025",
    "type": "quarterly",
    "startDate": "2025-01-01",
    "endDate": "2025-03-31",
    "status": "gesloten",
    "omschrijving": "Eerste kwartaal 2025"
  }
]
```

### FundingSource (3 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "FundingSource", "slug": "gemeentefonds-2025" },
    "name": "Gemeentefonds 2025",
    "type": "government",
    "referentie": "GF-2025",
    "totaalBedrag": 8450000.00,
    "beschikbaarBedrag": 3210000.00,
    "omschrijving": "Rijksbijdrage via het Gemeentefonds — algemene uitkering 2025"
  },
  {
    "@self": { "register": "shillinq", "schema": "FundingSource", "slug": "europees-sociaal-fonds-2025" },
    "name": "Europees Sociaal Fonds — Participatie",
    "type": "grant",
    "referentie": "ESF-2025-NL-003",
    "totaalBedrag": 420000.00,
    "beschikbaarBedrag": 290000.00,
    "omschrijving": "ESF-subsidie voor re-integratietrajecten en participatieprojecten"
  },
  {
    "@self": { "register": "shillinq", "schema": "FundingSource", "slug": "eigen-middelen-conduction-2025" },
    "name": "Eigen Middelen 2025",
    "type": "own",
    "referentie": "EM-2025",
    "totaalBedrag": 650000.00,
    "beschikbaarBedrag": 412000.00,
    "omschrijving": "Intern gefinancierde projecten en bedrijfsvoering"
  }
]
```

### BudgetAllocation (3 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "ict-afd-ia-2025" },
    "naam": "ICT — Afdeling I&A Q1-Q4 2025",
    "toegewezenBedrag": 485000.00,
    "besteedBedrag": 142300.00,
    "vastgelegdBedrag": 87500.00,
    "drempelPercentage": 80,
    "omschrijving": "Jaarallocatie ICT infrastructuurbudget voor afdeling I&A"
  },
  {
    "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "wmo-begeleiding-q1-2025" },
    "naam": "WMO Begeleiding — Q1 2025",
    "toegewezenBedrag": 537500.00,
    "besteedBedrag": 335500.00,
    "vastgelegdBedrag": 78750.00,
    "drempelPercentage": 85,
    "omschrijving": "Kwartaalallocatie WMO individuele begeleiding"
  },
  {
    "@self": { "register": "shillinq", "schema": "BudgetAllocation", "slug": "openbare-ruimte-onderhoud-2025" },
    "naam": "OR Onderhoud — Volledig boekjaar 2025",
    "toegewezenBedrag": 1780000.00,
    "besteedBedrag": 923400.00,
    "vastgelegdBedrag": 412000.00,
    "drempelPercentage": 80,
    "omschrijving": "Jaarallocatie voor onderhoud openbare ruimte"
  }
]
```

### ExpenditureRequest (3 objects)

```json
[
  {
    "@self": { "register": "shillinq", "schema": "ExpenditureRequest", "slug": "laptops-2025-ia-001" },
    "titel": "Aanschaf laptops afdeling I&A",
    "referentie": "UIT-2025-001",
    "status": "in_behandeling",
    "bedrag": 24500.00,
    "omschrijving": "Vervanging 14 laptops voor medewerkers afdeling Informatie & Automatisering",
    "leverancier": "Centraal ICT Depot B.V.",
    "motivering": "Huidige laptops zijn ouder dan 5 jaar en voldoen niet meer aan beveiligingseisen"
  },
  {
    "@self": { "register": "shillinq", "schema": "ExpenditureRequest", "slug": "advies-digitalisering-001" },
    "titel": "Externe adviesopdracht — Digitale Dienstverlening",
    "referentie": "UIT-2025-002",
    "status": "goedgekeurd",
    "bedrag": 45000.00,
    "omschrijving": "Adviesopdracht voor herontwerp digitale dienstverlening gemeente",
    "leverancier": "Conduction B.V.",
    "motivering": "Strategisch advies vereist voor implementatieplan 2025-2027"
  },
  {
    "@self": { "register": "shillinq", "schema": "ExpenditureRequest", "slug": "maaimachines-bor-2025" },
    "titel": "Aanschaf maaimachines BOR",
    "referentie": "UIT-2025-003",
    "status": "concept",
    "bedrag": 18750.00,
    "omschrijving": "Twee nieuwe maaimachines ter vervanging van verouderd materieel",
    "leverancier": "Groenpro Machines B.V.",
    "motivering": "Vervanging gepland in meerjarenonderhoudsprogramma 2024-2026"
  }
]
```
