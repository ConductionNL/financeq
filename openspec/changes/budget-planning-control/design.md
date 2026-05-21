# Design: Budget Planning & Control — Shillinq

## Data Model & Schema Definitions

### Budget

_A financial plan allocating resources for a specific period, organization, and location_

**Schema:** `Budget` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| budgetName | string | Yes | Name or identifier of the budget (e.g., "Marketing 2026", "IT Infrastructure Q2") |
| totalAmount | number | Yes | Total budgeted amount in the specified currency |
| startDate | datetime | Yes | Date when the budget becomes effective |
| endDate | datetime | Yes | Date when the budget expires |
| description | string | No | Detailed description or purpose of the budget |
| currency | string | Yes | Currency code (ISO 4217), defaults to EUR for Dutch organizations |
| budgetCategory | string | Yes | Category of the budget (operational expenses, capital expenses, revenue, training, etc.) |
| amountSpent | number | No | Current amount spent or committed against this budget (calculated) |
| alertThreshold | number | No | Percentage (0-100) at which to trigger spending alerts (default: 80) |
| budgetType | string | No | Type of budget (fixed, flexible, rolling, zero-based) |
| fiscalYear | integer | Yes | Fiscal year this budget applies to (e.g., 2026) |
| costCenter | string | No | Cost center or department code for budget allocation |
| attachments | array | No | Supporting documents and justification files |

**Relations:**
- → Organization (many-to-one) — Organization owning the budget
- → Location (many-to-one) — Location for multi-site allocation
- → Person (many-to-one) — Budget owner/manager
- → BudgetPeriod (many-to-one) — Period within which budget applies
- → BudgetAllocation (one-to-many) — Subdivisions of this budget
- → BudgetAmendment (one-to-many) — Changes to this budget
- → ExpenditureRequest (one-to-many) — Expenditure requests against this budget

**Seed Data:**

```json
{
  "register": "budgets",
  "schema": "Budget",
  "@self": [
    {
      "budgetName": "Faciliteiten & Huisvesting 2026",
      "totalAmount": 250000.00,
      "startDate": "2026-01-01T00:00:00Z",
      "endDate": "2026-12-31T23:59:59Z",
      "description": "Jaarlijks budget voor kantoorruimte, verwarming, elektriciteit en onderhoudswerk",
      "currency": "EUR",
      "budgetCategory": "operational expenses",
      "amountSpent": 127500.00,
      "alertThreshold": 80,
      "budgetType": "fixed",
      "fiscalYear": 2026,
      "costCenter": "CC-001"
    },
    {
      "budgetName": "Personeelstraining en Ontwikkeling Q2-Q3",
      "totalAmount": 45000.00,
      "startDate": "2026-04-01T00:00:00Z",
      "endDate": "2026-09-30T23:59:59Z",
      "description": "Training programma's voor medewerkers, cursus licenties, conferentie deelname",
      "currency": "EUR",
      "budgetCategory": "training",
      "amountSpent": 18200.50,
      "alertThreshold": 75,
      "budgetType": "flexible",
      "fiscalYear": 2026,
      "costCenter": "CC-HR-001"
    },
    {
      "budgetName": "IT Infrastructuur Upgrade 2026",
      "totalAmount": 180000.00,
      "startDate": "2026-01-01T00:00:00Z",
      "endDate": "2026-12-31T23:59:59Z",
      "description": "Server upgrades, netwerk modernisering, cloud migratie",
      "currency": "EUR",
      "budgetCategory": "capital expenses",
      "amountSpent": 89400.00,
      "alertThreshold": 85,
      "budgetType": "fixed",
      "fiscalYear": 2026,
      "costCenter": "CC-IT-001"
    },
    {
      "budgetName": "Gemeente Participatie-Proces 2026",
      "totalAmount": 75000.00,
      "startDate": "2026-01-01T00:00:00Z",
      "endDate": "2026-12-31T23:59:59Z",
      "description": "Publieke participatie-activiteiten, burgerbetrokking, raadpleeging projecten",
      "currency": "EUR",
      "budgetCategory": "participation",
      "amountSpent": 32150.00,
      "alertThreshold": 80,
      "budgetType": "rolling",
      "fiscalYear": 2026,
      "costCenter": "CC-PART-001"
    },
    {
      "budgetName": "Leveranciers & Aankoop Materiaal",
      "totalAmount": 500000.00,
      "startDate": "2026-01-01T00:00:00Z",
      "endDate": "2026-12-31T23:59:59Z",
      "description": "Direct material procurement, kantoorbenodigdheden, hardware",
      "currency": "EUR",
      "budgetCategory": "operational expenses",
      "amountSpent": 287600.00,
      "alertThreshold": 85,
      "budgetType": "flexible",
      "fiscalYear": 2026,
      "costCenter": "CC-PROC-001"
    }
  ]
}
```

### BudgetAllocation

_A subdivision of budget resources allocated to a specific department, funding source, or purpose_

**Schema:** `BudgetAllocation` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| allocationNumber | string | Yes | Unique identifier (e.g., "ALLOC-2026-001") |
| amount | number | Yes | Allocated amount |
| status | string | Yes | Status: pending, approved, allocated, spent, closed |
| description | string | No | Details about the allocation |

**Relations:**
- → Budget (many-to-one) — Parent budget
- → FundingSource (many-to-one) — Source of funds
- → Organization (many-to-one) — Organization receiving allocation

**Seed Data:**

```json
{
  "register": "budget_allocations",
  "schema": "BudgetAllocation",
  "@self": [
    {
      "allocationNumber": "ALLOC-2026-001",
      "amount": 100000.00,
      "status": "approved",
      "description": "Huurkosten kantoor Amsterdam Noord, maandelijks"
    },
    {
      "allocationNumber": "ALLOC-2026-002",
      "amount": 50000.00,
      "status": "allocated",
      "description": "Personeelstraining programma IT-vaardigheden"
    },
    {
      "allocationNumber": "ALLOC-2026-003",
      "amount": 75000.00,
      "status": "pending",
      "description": "Netwerk hardware upgrade fase 2"
    }
  ]
}
```

### BudgetAmendment

_A proposed or executed change to an approved budget amount_

**Schema:** `BudgetAmendment` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| amendmentNumber | string | Yes | Unique identifier (e.g., "AMEND-2026-001") |
| originalAmount | number | Yes | Original budgeted amount |
| newAmount | number | Yes | Revised budget amount |
| reason | string | Yes | Reason for the amendment (scope change, savings, unforeseen cost) |
| status | string | Yes | Status: proposed, pending_approval, approved, rejected, executed |
| effectiveDate | datetime | No | When amendment takes effect |

**Relations:**
- → Budget (many-to-one) — Budget being amended
- → ApprovalRequest (many-to-one) — Approval workflow for this amendment

**Seed Data:**

```json
{
  "register": "budget_amendments",
  "schema": "BudgetAmendment",
  "@self": [
    {
      "amendmentNumber": "AMEND-2026-001",
      "originalAmount": 45000.00,
      "newAmount": 52500.00,
      "reason": "Scope expansion: additional training modules en certificering programma's",
      "status": "approved",
      "effectiveDate": "2026-05-15T00:00:00Z"
    },
    {
      "amendmentNumber": "AMEND-2026-002",
      "originalAmount": 180000.00,
      "newAmount": 165000.00,
      "reason": "Kostenbesparing door vendor consolidatie en betere contractvoorwaarden",
      "status": "pending_approval",
      "effectiveDate": "2026-06-01T00:00:00Z"
    },
    {
      "amendmentNumber": "AMEND-2026-003",
      "originalAmount": 75000.00,
      "newAmount": 85000.00,
      "reason": "Increased participation activities: extra bijeenkomsten en online platform kosten",
      "status": "proposed",
      "effectiveDate": null
    }
  ]
}
```

### BudgetPeriod

_A defined time period for budget planning, such as fiscal year, quarter, or month_

**Schema:** `BudgetPeriod` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the period (e.g., 'FY2026', 'Q2 2026', 'May 2026') |
| type | string | Yes | Period type: fiscal_year, calendar_year, quarter, month, custom |
| startDate | datetime | Yes | Period start date |
| endDate | datetime | Yes | Period end date |
| fiscalYear | string | No | Associated fiscal year (e.g., '2026') |

**Relations:**
- → Budget (one-to-many) — Budgets within this period

**Seed Data:**

```json
{
  "register": "budget_periods",
  "schema": "BudgetPeriod",
  "@self": [
    {
      "name": "FY2026",
      "type": "fiscal_year",
      "startDate": "2026-01-01T00:00:00Z",
      "endDate": "2026-12-31T23:59:59Z",
      "fiscalYear": "2026"
    },
    {
      "name": "Q2 2026",
      "type": "quarter",
      "startDate": "2026-04-01T00:00:00Z",
      "endDate": "2026-06-30T23:59:59Z",
      "fiscalYear": "2026"
    },
    {
      "name": "May 2026",
      "type": "month",
      "startDate": "2026-05-01T00:00:00Z",
      "endDate": "2026-05-31T23:59:59Z",
      "fiscalYear": "2026"
    },
    {
      "name": "June 2026",
      "type": "month",
      "startDate": "2026-06-01T00:00:00Z",
      "endDate": "2026-06-30T23:59:59Z",
      "fiscalYear": "2026"
    }
  ]
}
```

### ExpenditureRequest

_A request to spend funds from an allocated budget, requiring review and approval_

**Schema:** `ExpenditureRequest` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| requestNumber | string | Yes | Unique identifier (e.g., "EXP-2026-0001") |
| amount | number | Yes | Requested expenditure amount |
| purpose | string | Yes | Purpose or description of the expenditure |
| status | string | Yes | Status: draft, submitted, approved, rejected, executed |
| requestDate | datetime | Yes | Date request was made |

**Relations:**
- → Budget (many-to-one) — Budget being spent from
- → ApprovalRequest (many-to-one) — Approval workflow
- → Person (many-to-one) — Person requesting expenditure

**Seed Data:**

```json
{
  "register": "expenditure_requests",
  "schema": "ExpenditureRequest",
  "@self": [
    {
      "requestNumber": "EXP-2026-0001",
      "amount": 12500.00,
      "purpose": "Jaarlijkse kantoor vernieuwing - nieuwe stoelen en bureaus receptie",
      "status": "approved",
      "requestDate": "2026-04-15T09:00:00Z"
    },
    {
      "requestNumber": "EXP-2026-0002",
      "amount": 8750.00,
      "purpose": "Conference deelname: European HR Summit Amsterdam",
      "status": "submitted",
      "requestDate": "2026-05-10T14:30:00Z"
    },
    {
      "requestNumber": "EXP-2026-0003",
      "amount": 45000.00,
      "purpose": "Server hardware: twee Dell PowerEdge R750 servers",
      "status": "draft",
      "requestDate": "2026-05-20T11:15:00Z"
    }
  ]
}
```

### FundingSource

_A source of funds that can be allocated to budgets and expenditures_

**Schema:** `FundingSource` (custom OpenRegister schema)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the funding source (e.g., "Annual Budget 2026", "EU Grant Program") |
| totalAmount | number | Yes | Total available funds |
| status | string | Yes | Status: active, inactive, depleted |
| description | string | No | Details about the funding source |

**Relations:**
- → BudgetAllocation (one-to-many) — Allocations from this source

**Seed Data:**

```json
{
  "register": "funding_sources",
  "schema": "FundingSource",
  "@self": [
    {
      "name": "Annual Budget 2026",
      "totalAmount": 1500000.00,
      "status": "active",
      "description": "Jaarlijks operationeel budget toegewezen door gemeenteraad"
    },
    {
      "name": "EU Participation Grant 2026-2027",
      "totalAmount": 180000.00,
      "status": "active",
      "description": "Europese subsidie voor burgerbetrokking en democratische innovatie"
    },
    {
      "name": "IT Modernization Fund",
      "totalAmount": 350000.00,
      "status": "active",
      "description": "Meerjarig investeringsfonds voor IT-transformatie"
    },
    {
      "name": "Department Savings Reserve 2025",
      "totalAmount": 45000.00,
      "status": "active",
      "description": "Gerealiseerde besparing vorig jaar, beschikbaar voor nieuwe initiatieven"
    }
  ]
}
```

### Location

_A physical or geographic location for multi-site budget allocation and tracking_

**Schema:** `Location` (schema:Place)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Location name (e.g., "Amsterdam Noord", "Den Haag Central") |
| code | string | No | Location code or identifier (e.g., "LOC-001") |
| address | string | No | Physical address |
| region | string | No | Geographic region or administrative boundary |

**Relations:**
- → Organization (many-to-one) — Organization managing location
- → Budget (one-to-many) — Budgets allocated to this location

**Seed Data:**

```json
{
  "register": "locations",
  "schema": "Location",
  "@self": [
    {
      "name": "Amsterdam Noord",
      "code": "LOC-001",
      "address": "Zaanstraat 100, 1013 HZ Amsterdam",
      "region": "Noord-Holland"
    },
    {
      "name": "Den Haag Centrum",
      "code": "LOC-002",
      "address": "Prinsessegracht 20, 2514 AP Den Haag",
      "region": "Zuid-Holland"
    },
    {
      "name": "Almere Stad",
      "code": "LOC-003",
      "address": "Stadsplein 1, 1315 AL Almere",
      "region": "Flevoland"
    }
  ]
}
```

## User Interface Patterns

### Budget Dashboard
- KPI cards showing: Total Budget, Spent Amount, Remaining Balance, % Utilization
- Line chart: Budget vs Actual vs Forecast over time
- Heatmap: Budget utilization by cost center and department
- Alert banner: Budgets approaching threshold with amber/red indicators

### Budget Detail Page
- Header: Budget name, fiscal year, dates, status badge
- Sections:
  - Budget Overview (total, spent, remaining, allocation breakdown)
  - Allocations (table: allocation number, amount, status, funding source)
  - Expenditure Requests (table: request number, amount, purpose, status, approver)
  - Amendments (table: amendment number, old amount, new amount, reason, status)
  - Commitments (table: purchase order, committed amount, status)
  - Audit Trail (who changed what when)
- Sidebar:
  - Files (supporting documents)
  - Notes (decision notes, rationale)
  - Related Records (projects, cost centers using this budget)

### Amendment Workflow
- Step 1: Propose (form: reason, new amount, effective date, supporting docs)
- Step 2: Review (financial controller validates fiscal impact)
- Step 3: Approve (CFO/Director decision)
- Step 4: Execute (system updates budget allocation, records effective date)
- Status transitions with audit trail at each step

### Real-time Budget Check (Approval Integration)
- Widget in expenditure request approval:
  - Current budget remaining
  - Pending commitments
  - This request impact (new total % utilization)
  - Green/amber/red indicator
  - Approval allowed/blocked based on policy

## Reuse Analysis

**OpenRegister Services Leveraged:**
- `ObjectService.saveObject()` — Budget CRUD
- `ObjectService.findAll()` — Budget listings with pagination/filtering
- `ImportService` / `ExportService` — Budget import/export
- `CnIndexPage` + `CnDetailPage` — UI scaffolding
- `AuditTrailService` — Amendment audit trails
- `ApprovalRequest` — Workflow integration for budget amendments
- `FileService` — Supporting document management
- `IndexService` — Budget search and discovery

**No custom CRUD, search, or import/export controllers needed.**

## Integration Points

### Approval Chain Integration
- Budget check as approval task input (Budget → ExpenditureRequest → ApprovalRequest)
- Block approval if expenditure exceeds remaining budget
- Record budget approval decision in amendment history

### PO & Purchase Order Integration
- Commitment tracking: approved POs update ExpenditureRequest.status
- Forecast integration: approved but unshipped POs appear in forecasted spend
- Budget consumption: invoice matching updates amountSpent

### Financial Reporting Integration
- Budget vs Actual report: summarize budgets by cost center, department, location
- Variance analysis: forecast vs actual with materiality thresholds
- Amendment report: track all budget changes, approvers, effective dates

## Security & Authorization

- Budget visibility: user's organization + delegated locations only (RBAC via AuthorizationService)
- Amendment approval: Controller + Director roles via ApprovalRequest
- Budget holder acknowledgement: electronically signed via FileService + audit trail
- Audit log: all amendments, approvals, expenditure decisions recorded immutably

## Accessibility (WCAG 2.1 AA)

- Budget KPI cards: color not sole conveyor (include numeric values)
- Charts: alternative text table view for budget consumption
- Forms: all fields labeled, error messages associated with inputs
- Amendment workflow: keyboard-navigable step-through, focus management on modal open
- Responsive: budget dashboard tiles stack at 768px, detail page sections at 480px

## Performance Considerations

- Budget summary (total, spent, remaining): calculated field with caching (5-minute TTL)
- Commitment tracking: indexed query on ExpenditureRequest.status = 'approved' + purchase order relation
- Forecast calculations: run async background job nightly, results cached (ETC, exhaustion date)
- Amendment history: paginated (20 items per page, lazy load on scroll)

## Localization

- All user-visible strings: Dutch (nl) + English (en) translations
- Currency formatting: EUR, respect locale for thousands/decimal separator
- Dates: ISO 8601 in API, formatted per user locale in UI (d/m/Y for Dutch)
- Budget categories: enum values translated (operational expenses → operationele kosten)
