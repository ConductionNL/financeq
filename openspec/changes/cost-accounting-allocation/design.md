# Design: Cost Accounting & Allocation — Shillinq

## Architecture

This change follows the OpenRegister + `@conduction/nextcloud-vue` platform model. All domain data lives in OpenRegister objects; no custom Entity/Mapper classes are written. Business logic specific to overhead allocation (running an allocation cycle, zero-key detection, version incrementing, reversal) is implemented in a dedicated `AllocationService`. Recurring allocation runs are scheduled via Nextcloud's background job queue (`IJobList`).

### Layers

| Layer | Responsibility |
|-------|---------------|
| OpenRegister schemas | Seven new schemas defined in `lib/Settings/shillinq_register.json` |
| `AllocationService` | Overhead allocation run logic, zero-key guard, version control, reversal |
| `TimesheetService` | Period aggregation, utilization calculation |
| `PerDiemService` | Dutch rate lookup, amount calculation |
| `InventoryValuationService` | FIFO and average cost recalculation |
| `AllocationController` | Thin REST endpoint: trigger run, get results, reverse |
| Frontend stores | One `createObjectStore` per schema, registered in `store/store.js` |
| Frontend pages | Dashboard, index pages, detail pages per entity |

## Reuse Analysis

| Capability | OpenRegister / Platform service used | Custom code needed? |
|-----------|--------------------------------------|---------------------|
| CRUD for all 7 entities | `ObjectService.saveObject()`, `findObject()`, `findObjects()` | No |
| List, filter, paginate | `CnIndexPage` + `useListView` + `CnDataTable` | No |
| Schema-driven create/edit forms | `CnFormDialog` auto-generated from schema | No |
| Audit trail on all allocation objects | `AuditTrailService` (automatic) | No |
| File attachments (e.g. allocation reports) | `FileService` + `CnObjectSidebar` | No |
| Approval workflow routing | `WorkflowEngineController` + `ApprovalRequest` (existing entity) | No |
| Background scheduled allocation | `IJobList` scheduled job | Yes — domain trigger |
| Overhead allocation run logic | — | Yes — custom `AllocationService` |
| Zero-key guard | — | Yes — part of `AllocationService` |
| Version increment on allocation change | — | Yes — part of `AllocationService` |
| Reversal of incorrect allocation | — | Yes — part of `AllocationService` |
| FIFO / average cost recalculation | — | Yes — `InventoryValuationService` |
| Dutch per diem rate lookup | — | Yes — `PerDiemService` with rate table |
| Timesheet period aggregation | — | Yes — `TimesheetService` |
| Multi-dimensional P&L dashboard | `CnDashboardPage` + `CnChartWidget` + `CnStatsBlock` | No |
| Export allocation results (CSV/Excel) | `ExportService` + `CnMassExportDialog` | No |
| Import allocation keys | `ImportService` + `CnMassImportDialog` | No |

No overlap found with existing ObjectService, RegisterService, SchemaService, ConfigurationService, or shared Vue components for the custom business logic above.

## Data Model

All entities are implemented as OpenRegister schemas using schema.org vocabulary. Relations use the OpenRegister relation mechanism (register + schema + objectId). No foreign keys.

### CostCenter (`schema:Organization`)

Tracks departmental or functional cost centers for expense allocation and analysis.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| code | string | Yes | Unique cost center identifier (e.g. CC-IT) |
| name | string | Yes | Name of the cost center |
| description | string | No | Responsibilities and scope |
| status | string | Yes | `active` or `inactive` |
| budget | number | No | Annual or periodic budget |
| createdDate | datetime | Yes | Date when cost center was created |

Relations: → Person (manager, many-to-one), → Organization (many-to-one)

### AllocationRule (`schema:Thing`)

Defines how overhead costs are automatically distributed between cost centers.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the allocation rule |
| ruleType | string | Yes | `percentage`, `fixed_amount`, or `formula` |
| percentage | number | No | Percentage to allocate (if percentage-based) |
| fixedAmount | number | No | Fixed amount per period (if fixed-based) |
| frequency | string | Yes | `monthly`, `quarterly`, or `yearly` |
| isActive | boolean | Yes | Whether rule is currently active |
| startDate | datetime | Yes | Effective date |
| endDate | datetime | No | Expiry date |
| description | string | No | Rule description |

Relations: → CostCenter (source, many-to-one), → CostCenter (target, many-to-one)

### CostAllocation (`schema:Offer`)

Records a cost distribution transaction with version control for model changes.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Description of the allocation |
| allocationDate | datetime | Yes | Effective date of the allocation |
| sourceAmount | number | Yes | Total amount to allocate |
| allocationPercentage | number | No | Percentage applied |
| allocationAmount | number | No | Calculated allocated amount |
| period | string | Yes | `monthly`, `quarterly`, or `yearly` |
| status | string | Yes | `draft`, `approved`, or `allocated` |
| version | number | Yes | Version number for change tracking and rollback |
| description | string | No | Allocation notes |

Relations: → CostCenter (source, many-to-one), → CostCenter (target, many-to-one)

### CostProject (`schema:Project`)

Tracks time, materials, and costs on a project basis with budget monitoring.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| code | string | Yes | Unique project cost code |
| name | string | Yes | Project name |
| description | string | No | Project scope |
| budget | number | No | Total project budget |
| totalCost | number | No | Total costs incurred to date |
| startDate | datetime | Yes | Project start date |
| endDate | datetime | No | Planned or actual end date |
| status | string | Yes | `active`, `closed`, or `archived` |

Relations: → Organization (many-to-one), → CostCenter (many-to-one)

### InventoryValuation (`schema:Product`)

Valuation of on-hand inventory using FIFO or average cost for P&L and balance sheet.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| quantity | number | Yes | Quantity currently in stock |
| unitCost | number | Yes | Cost per unit under selected valuation method |
| totalValue | number | Yes | Total inventory value (quantity × unitCost) |
| valuationMethod | string | Yes | `FIFO`, `average`, `specific`, or `weighted_average` |
| date | datetime | Yes | Date of valuation or inventory count |
| warehouse | string | No | Warehouse or storage location identifier |
| status | string | Yes | `active`, `adjusted`, or `obsolete` |

Relations: → Product (many-to-one), → CostCenter (many-to-one)

### PerDiem (`schema:Offer`)

Daily allowance for employees on business travel, calculated on Dutch rates.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| date | datetime | Yes | Date for which per diem is claimed |
| country | string | Yes | Country where travel occurred (ISO 3166-1 alpha-2) |
| nights | number | No | Number of nights away from home base |
| rate | number | Yes | Per diem rate applicable for the country/date |
| amount | number | Yes | Total per diem allowance |
| status | string | Yes | `draft`, `approved`, or `paid` |
| approvedDate | datetime | No | Date when approved |
| description | string | No | Travel purpose or notes |

Relations: → Person (many-to-one), → CostCenter (many-to-one)

### Timesheet (`schema:Report`)

Periodic summary of time entries for an employee with utilization metrics.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| periodStart | datetime | Yes | Start date of the reporting period |
| periodEnd | datetime | Yes | End date of the reporting period |
| totalHours | number | Yes | Total hours logged in period |
| utilizationPercentage | number | No | Utilization as percentage of available hours |
| totalCost | number | No | Total cost based on hourly rates |
| status | string | Yes | `draft`, `submitted`, or `approved` |
| submittedDate | datetime | No | Date when submitted |
| approvedDate | datetime | No | Date when approved |

Relations: → Person (many-to-one), → TimeEntry (one-to-many), → ApprovalRequest (many-to-one)

## Seed Data

Seed data uses Dutch values for a fictional consultancy (Adviesbureau De Zilveren Penning B.V., Amsterdam). Loaded via `lib/Settings/shillinq_register.json` using the `@self` envelope, idempotent re-imports matched by slug.

### CostCenter Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "cost-center", "slug": "cc-ict" },
    "code": "CC-ICT",
    "name": "ICT & Digitalisering",
    "description": "Beheer en ontwikkeling van ICT-infrastructuur en digitale dienstverlening",
    "status": "active",
    "budget": 185000.00,
    "createdDate": "2024-01-01T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-center", "slug": "cc-hrm" },
    "code": "CC-HRM",
    "name": "Human Resources & Management",
    "description": "Personeelsbeheer, werving, opleiding en organisatieontwikkeling",
    "status": "active",
    "budget": 92000.00,
    "createdDate": "2024-01-01T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-center", "slug": "cc-fin" },
    "code": "CC-FIN",
    "name": "Financiën & Control",
    "description": "Financiële administratie, rapportage en interne controle",
    "status": "active",
    "budget": 74000.00,
    "createdDate": "2024-01-01T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-center", "slug": "cc-adv" },
    "code": "CC-ADV",
    "name": "Advies & Projecten",
    "description": "Primair proces: klantadvies en projectuitvoering",
    "status": "active",
    "budget": 420000.00,
    "createdDate": "2024-01-01T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-center", "slug": "cc-fac" },
    "code": "CC-FAC",
    "name": "Facilitaire Diensten",
    "description": "Huisvesting, schoonmaak, beveiliging en facilitaire ondersteuning",
    "status": "active",
    "budget": 56000.00,
    "createdDate": "2024-01-01T00:00:00Z"
  }
]
```

### AllocationRule Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "allocation-rule", "slug": "ar-ict-naar-adv" },
    "name": "ICT overhead naar Advies & Projecten",
    "ruleType": "percentage",
    "percentage": 60.00,
    "frequency": "monthly",
    "isActive": true,
    "startDate": "2024-01-01T00:00:00Z",
    "description": "Verdeling ICT-kosten naar primair proces op basis van gebruikersaantal"
  },
  {
    "@self": { "register": "shillinq", "schema": "allocation-rule", "slug": "ar-hrm-naar-adv" },
    "name": "HRM overhead naar Advies & Projecten",
    "ruleType": "percentage",
    "percentage": 70.00,
    "frequency": "monthly",
    "isActive": true,
    "startDate": "2024-01-01T00:00:00Z",
    "description": "Verdeling personeelskosten naar uitvoerend personeel"
  },
  {
    "@self": { "register": "shillinq", "schema": "allocation-rule", "slug": "ar-fac-vast" },
    "name": "Facilitaire vaste bijdrage per kwartaal",
    "ruleType": "fixed_amount",
    "fixedAmount": 14500.00,
    "frequency": "quarterly",
    "isActive": true,
    "startDate": "2024-01-01T00:00:00Z",
    "description": "Vaste kwartaalbijdrage huurlasten en energiekosten"
  },
  {
    "@self": { "register": "shillinq", "schema": "allocation-rule", "slug": "ar-fin-naar-adv" },
    "name": "Financiën overhead naar Advies",
    "ruleType": "percentage",
    "percentage": 50.00,
    "frequency": "monthly",
    "isActive": false,
    "startDate": "2024-01-01T00:00:00Z",
    "endDate": "2024-12-31T23:59:59Z",
    "description": "Historische verdeelsleutel — vervangen door formulebasis per 2025"
  }
]
```

### CostAllocation Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "cost-allocation", "slug": "ca-jan2026-ict" },
    "name": "Januari 2026 — ICT overhead verdeling",
    "allocationDate": "2026-01-31T00:00:00Z",
    "sourceAmount": 15420.00,
    "allocationPercentage": 60.00,
    "allocationAmount": 9252.00,
    "period": "monthly",
    "status": "allocated",
    "version": 1,
    "description": "Maandelijkse verdeling ICT-overheadkosten januari 2026"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-allocation", "slug": "ca-jan2026-hrm" },
    "name": "Januari 2026 — HRM overhead verdeling",
    "allocationDate": "2026-01-31T00:00:00Z",
    "sourceAmount": 7680.00,
    "allocationPercentage": 70.00,
    "allocationAmount": 5376.00,
    "period": "monthly",
    "status": "allocated",
    "version": 1,
    "description": "Maandelijkse verdeling HRM-overheadkosten januari 2026"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-allocation", "slug": "ca-q12026-fac" },
    "name": "Q1 2026 — Facilitaire vaste bijdrage",
    "allocationDate": "2026-03-31T00:00:00Z",
    "sourceAmount": 14500.00,
    "allocationAmount": 14500.00,
    "period": "quarterly",
    "status": "draft",
    "version": 1,
    "description": "Kwartaalverdeling huurlasten en energiekosten Q1 2026"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-allocation", "slug": "ca-feb2026-ict-v2" },
    "name": "Februari 2026 — ICT overhead verdeling (herziening)",
    "allocationDate": "2026-02-28T00:00:00Z",
    "sourceAmount": 16100.00,
    "allocationPercentage": 60.00,
    "allocationAmount": 9660.00,
    "period": "monthly",
    "status": "approved",
    "version": 2,
    "description": "Herziene verdeling na correctie op bronstotaal februari 2026"
  }
]
```

### CostProject Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "cost-project", "slug": "proj-2026-dlo" },
    "code": "PROJ-2026-DLO",
    "name": "Digitaal Loket Herimplementatie",
    "description": "Vervanging van het gemeentelijk digitaal loket met open-source alternatief",
    "budget": 68000.00,
    "totalCost": 24350.00,
    "startDate": "2026-01-15T00:00:00Z",
    "endDate": "2026-09-30T00:00:00Z",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-project", "slug": "proj-2026-crm" },
    "code": "PROJ-2026-CRM",
    "name": "CRM Systeem Implementatie",
    "description": "Implementatie en inrichting nieuw relatiebeheerssysteem",
    "budget": 42000.00,
    "totalCost": 38750.00,
    "startDate": "2025-09-01T00:00:00Z",
    "endDate": "2026-03-31T00:00:00Z",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-project", "slug": "proj-2025-duu" },
    "code": "PROJ-2025-DUU",
    "name": "Duurzaamheid Transitieplan",
    "description": "Adviestraject verduurzaming bedrijfsvoering en energietransitie",
    "budget": 31500.00,
    "totalCost": 31500.00,
    "startDate": "2025-03-01T00:00:00Z",
    "endDate": "2025-11-30T00:00:00Z",
    "status": "closed"
  },
  {
    "@self": { "register": "shillinq", "schema": "cost-project", "slug": "proj-2026-pro" },
    "code": "PROJ-2026-PRO",
    "name": "Procesoptimalisatie Financieel Beheer",
    "description": "Herinrichting financiële processen en automatisering rapportages",
    "budget": 25000.00,
    "totalCost": 4800.00,
    "startDate": "2026-02-01T00:00:00Z",
    "status": "active"
  }
]
```

### InventoryValuation Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "inventory-valuation", "slug": "iv-laptop-2026-01" },
    "quantity": 12,
    "unitCost": 1249.00,
    "totalValue": 14988.00,
    "valuationMethod": "average",
    "date": "2026-01-31T00:00:00Z",
    "warehouse": "Magazijn Amsterdam Keizersgracht",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "inventory-valuation", "slug": "iv-monitor-2026-01" },
    "quantity": 18,
    "unitCost": 384.50,
    "totalValue": 6921.00,
    "valuationMethod": "FIFO",
    "date": "2026-01-31T00:00:00Z",
    "warehouse": "Magazijn Amsterdam Keizersgracht",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "inventory-valuation", "slug": "iv-kantoor-2026-01" },
    "quantity": 450,
    "unitCost": 1.75,
    "totalValue": 787.50,
    "valuationMethod": "weighted_average",
    "date": "2026-01-31T00:00:00Z",
    "warehouse": "Kantoorvoorraad",
    "status": "active"
  },
  {
    "@self": { "register": "shillinq", "schema": "inventory-valuation", "slug": "iv-server-2025-12" },
    "quantity": 2,
    "unitCost": 4875.00,
    "totalValue": 9750.00,
    "valuationMethod": "specific",
    "date": "2025-12-31T00:00:00Z",
    "warehouse": "Serverruimte",
    "status": "adjusted"
  }
]
```

### PerDiem Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "per-diem", "slug": "pd-devries-nl-jan2026" },
    "date": "2026-01-14T00:00:00Z",
    "country": "NL",
    "nights": 1,
    "rate": 37.00,
    "amount": 37.00,
    "status": "approved",
    "approvedDate": "2026-01-17T00:00:00Z",
    "description": "Klantbezoek Den Haag — beleidsadvies gemeentelijke financiën"
  },
  {
    "@self": { "register": "shillinq", "schema": "per-diem", "slug": "pd-bakker-be-jan2026" },
    "date": "2026-01-21T00:00:00Z",
    "country": "BE",
    "nights": 2,
    "rate": 62.50,
    "amount": 125.00,
    "status": "approved",
    "approvedDate": "2026-01-24T00:00:00Z",
    "description": "EU-conferentie open source overheid Brussel"
  },
  {
    "@self": { "register": "shillinq", "schema": "per-diem", "slug": "pd-janssen-nl-feb2026" },
    "date": "2026-02-04T00:00:00Z",
    "country": "NL",
    "nights": 0,
    "rate": 17.50,
    "amount": 17.50,
    "status": "draft",
    "description": "Training Rotterdam — dagvergoeding zonder overnachting"
  },
  {
    "@self": { "register": "shillinq", "schema": "per-diem", "slug": "pd-devries-de-feb2026" },
    "date": "2026-02-11T00:00:00Z",
    "country": "DE",
    "nights": 3,
    "rate": 58.00,
    "amount": 174.00,
    "status": "approved",
    "approvedDate": "2026-02-14T00:00:00Z",
    "description": "CeBIT-bezoek Hannover — marktverkenning bedrijfssoftware"
  }
]
```

### Timesheet Seed Objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "timesheet", "slug": "ts-pietersen-jan2026" },
    "periodStart": "2026-01-01T00:00:00Z",
    "periodEnd": "2026-01-31T23:59:59Z",
    "totalHours": 160.0,
    "utilizationPercentage": 82.5,
    "totalCost": 12800.00,
    "status": "approved",
    "submittedDate": "2026-02-02T09:00:00Z",
    "approvedDate": "2026-02-03T14:30:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "timesheet", "slug": "ts-degroot-jan2026" },
    "periodStart": "2026-01-01T00:00:00Z",
    "periodEnd": "2026-01-31T23:59:59Z",
    "totalHours": 144.0,
    "utilizationPercentage": 90.0,
    "totalCost": 11520.00,
    "status": "submitted",
    "submittedDate": "2026-02-01T17:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "timesheet", "slug": "ts-janssen-feb2026" },
    "periodStart": "2026-02-01T00:00:00Z",
    "periodEnd": "2026-02-28T23:59:59Z",
    "totalHours": 128.5,
    "utilizationPercentage": 74.0,
    "totalCost": 10280.00,
    "status": "draft"
  },
  {
    "@self": { "register": "shillinq", "schema": "timesheet", "slug": "ts-bakker-jan2026" },
    "periodStart": "2026-01-01T00:00:00Z",
    "periodEnd": "2026-01-31T23:59:59Z",
    "totalHours": 168.0,
    "utilizationPercentage": 87.5,
    "totalCost": 13440.00,
    "status": "approved",
    "submittedDate": "2026-02-02T10:15:00Z",
    "approvedDate": "2026-02-03T11:00:00Z"
  }
]
```

## API Design

Custom endpoints for business logic not covered by OpenRegister REST. All follow ADR-002 URL pattern.

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/index.php/apps/shillinq/api/allocations/run` | Trigger overhead allocation run for a period |
| GET | `/index.php/apps/shillinq/api/allocations/{id}/results` | Get allocation results per product/programme |
| POST | `/index.php/apps/shillinq/api/allocations/{id}/reverse` | Reverse an incorrect allocation |
| GET | `/index.php/apps/shillinq/api/metrics` | Prometheus metrics (ADR-006) |
| GET | `/index.php/apps/shillinq/api/health` | Health check (ADR-006) |

All other CRUD is handled by OpenRegister's built-in REST API.

## Frontend Design

### Navigation (MainMenu)
- Dashboard
- Kostenplaatsen (CostCenter list)
- Verdeelregels (AllocationRule list)
- Kostenverdelingen (CostAllocation list)
- Projecten (CostProject list)
- Voorraden (InventoryValuation list)
- Dagvergoedingen (PerDiem list)
- Urenstaten (Timesheet list)

### Dashboard Widgets
- KPI cards: `CnStatsBlock` × 4 (open allocations, active cost centers, pending timesheets, pending per diems)
- Allocation status distribution: `CnChartWidget` (donut — draft / approved / allocated)
- Cost center budget vs. actuals: `CnChartWidget` (bar)
- Recent allocation runs: `CnTableWidget`

### Index Pages
All entity list pages use `CnIndexPage` with `useListView`. Standard pattern per ADR-004.

### Detail Pages
All entity detail pages use `CnDetailPage` with `CnDetailCard` sections. Related entities per spec displayed via `fetchUsed` / `fetchUses`.

### Settings (Admin)
- `CnVersionInfoCard` (first)
- `CnRegisterMapping` for schema/register assignment
- Dutch per diem rate table configuration section
- Mileage rate configuration (EUR 0.23/km default)

## Dutch Compliance

- Mileage: EUR 0.23/km as the default Dutch rate (Belastingdienst 2024 standard)
- Per diem rates: configurable per country with Dutch domestic rate (EUR 37/day with overnight) as default
- All amounts in EUR with 2 decimal precision
- Period boundaries aligned to Dutch fiscal calendar (calendar year)
