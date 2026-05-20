# Design: Financial Reporting & Accountability — Shillinq

## Context

Shillinq is a self-hosted Nextcloud business administration suite for freelancers, SMBs, and Dutch public-sector organizations. The `financial-reporting-accountability` spec introduces 11 new OpenRegister schemas covering the full financial reporting stack: fiscal year lifecycle, double-entry general ledger, balance sheets, trial balances, accountability report workflows, consolidated management reporting, and ESEF/XBRL export for listed-company compliance.

All data lives in OpenRegister (ADR-001). All UI uses `@conduction/nextcloud-vue` components (ADR-004). Custom code is limited to domain-specific business rules not provided by the platform.

## Goals / Non-Goals

**Goals:**

- Fiscal year lifecycle: open, close, and reopen accounting periods with audit trail
- Double-entry general ledger: balanced journal entries (debits = credits enforced at service layer)
- Financial statements at any date: balance sheet, trial balance, general ledger report
- Annual report export: PDF (board-ready via docudesk), Excel, XML, JSON; narrative footnotes per line item
- Accountability report workflow: draft → submitted → approved/rejected; overdue escalation; recipient notification
- Consolidated reporting: combine multiple organizations with inter-company elimination rules
- Revenue stream categorization with annual targets and period aggregation
- ESEF/XBRL generation with IFRS taxonomy mapping for AFM filing (should-have)

**Non-Goals:**

- Bank reconciliation and payment matching (separate spec)
- VAT/BTW calculation engine (separate tax spec)
- Payroll integration (separate HR spec)
- Procurement and purchase order management (separate procurement spec)
- Custom authentication or session management (Nextcloud built-in only, ADR-005)

## Architecture Decisions

### Decision 1: OpenRegister for all 11 entities

All financial data is stored as OpenRegister objects. No custom Doctrine entities, database tables, or custom mappers. Schema initialization via a repair step (`IRepairStep`) that imports `shillinq_register.json` on first install. Breaking schema changes go through a new repair step; existing migrations are never modified (ADR-001).

### Decision 2: Balance validation at service layer only

`JournalEntryService` enforces `isBalanced` (debitAmount = creditAmount) before persisting any journal entry. The platform's `ObjectService.saveObject()` is not extended — validation is a pre-save check in the service method. Unbalanced entries return HTTP 422 with a `message` field (ADR-002).

### Decision 3: Annual report export uses platform for CSV/Excel/JSON; custom service for PDF and XBRL

- CSV/Excel/JSON export → OpenRegister `ExportService` + `CnMassExportDialog` (zero custom code)
- PDF → `FinancialReportService` calls docudesk PDF generation with a Shillinq-specific template; attached via `FileService`
- XBRL/ESEF → `XbrlExportService` custom service; only triggered for "Annual" report type when ESEF flag is set

### Decision 4: Consolidation elimination rules stored as structured JSON on ConsolidationGroup

`eliminationRules` is an object property on `ConsolidationGroup`. `ConsolidationService` reads these rules at report-generation time and applies inter-company eliminations to `ConsolidatedReport`. No separate elimination-entry schema needed.

### Decision 5: Accountability report workflow via OpenRegister WorkflowEngineController

Status transitions (`draft → submitted → approved/rejected`) use the platform's `WorkflowEngineController` with a Shillinq-registered workflow definition. `AccountabilityReportService` triggers `NotificationService` on each transition. `OverdueAccountabilityReportJob` runs nightly and calls `NotificationService.notify()` for reports past their submission deadline.

### Decision 6: Fiscal year posting guard at service layer

`JournalEntryService` and `GeneralLedgerService` check `FiscalYear.isClosed` before accepting any new entries. Posting to a closed fiscal year returns HTTP 409 with message `"Boekjaar is afgesloten"`.

## Data Model

All 11 entities are OpenRegister schemas. Schema types follow schema.org vocabulary (ADR-011). Relations use OpenRegister relation mechanism — no foreign keys.

### FiscalYear (`schema:Event`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| year | integer | Yes | Fiscal year number (e.g., 2025) |
| startDate | date | Yes | First day of the fiscal period |
| endDate | date | Yes | Last day of the fiscal period |
| isClosed | boolean | No | Whether the year is closed for amendments |
| closingDate | date | No | Date the year was officially closed |

Relations: → FinancialReport (one-to-many), → JournalEntry (one-to-many)

### GeneralLedgerAccount (`schema:Product`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| accountNumber | string | Yes | Unique account code (e.g., 1000, 4100) |
| accountName | string | Yes | Descriptive account name |
| accountType | string | Yes | Asset, Liability, Equity, Revenue, or Expense |
| currency | string | Yes | ISO 4217 currency code |
| currentBalance | object | No | MonetaryAmount {value, currency} |

Relations: → JournalEntry (one-to-many)

### JournalEntry (`custom`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| entryDate | datetime | Yes | Date of the journal entry |
| entryNumber | string | Yes | Unique sequential journal entry number |
| description | string | Yes | Transaction description |
| debitAmount | number | Yes | Debit amount in EUR |
| creditAmount | number | Yes | Credit amount in EUR |
| isBalanced | boolean | Yes | Whether debits equal credits |
| accountCode | string | Yes | General ledger account number |
| journalCode | string | Yes | Journal type (sales, bank, cash, general, etc.) |
| reference | string | No | External reference (invoice, check, or document number) |
| vatAmount | number | No | VAT/BTW amount |
| departmentCode | string | No | Cost center or department code |
| memo | string | No | Additional notes or clarification |

Relations: → GeneralLedgerAccount (many-to-many), → FiscalYear (many-to-one)

### GeneralLedgerEntry (`schema:Thing`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| entryDate | datetime | Yes | Date of the GL entry |
| accountNumber | string | Yes | GL account code |
| accountName | string | Yes | Name of the GL account |
| debitAmount | number | No | Debit amount in base currency |
| creditAmount | number | No | Credit amount in base currency |
| description | string | Yes | Description of the transaction |
| reference | string | No | Reference document number or transaction ID |
| status | string | Yes | draft, posted, or reversed |

Relations: → FiscalYear (many-to-one), → Organization (many-to-one), → APTransaction (many-to-one)

### BalanceSheet (`schema:Table`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportDate | datetime | Yes | Date of the balance sheet snapshot |
| totalAssets | number | No | Total assets in base currency |
| totalLiabilities | number | No | Total liabilities in base currency |
| totalEquity | number | No | Total equity in base currency |
| currency | string | Yes | Currency code for amounts |
| status | string | Yes | draft, final, or published |

Relations: → FiscalYear (many-to-one), → Organization (many-to-one), → GeneralLedgerEntry (one-to-many)

### TrialBalance (`schema:Table`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportDate | datetime | Yes | Date of the trial balance |
| totalDebits | number | No | Total of all debit balances |
| totalCredits | number | No | Total of all credit balances |
| isBalanced | boolean | No | Whether debits equal credits |
| status | string | Yes | draft, verified, or final |
| preparedBy | string | No | Name or identifier of preparer |

Relations: → FiscalYear (many-to-one), → Organization (many-to-one), → GeneralLedgerEntry (one-to-many)

### FinancialReport (`schema:Report`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportType | string | Yes | Annual, Management, or Consolidated |
| reportFormat | string | Yes | PDF, Excel, XML, or JSON |
| reportStatus | string | No | Draft, Approved, or Published |
| generatedAt | dateTime | Yes | Timestamp of report generation |

Relations: → FiscalYear (many-to-one)

### AccountabilityReport (`schema:Report`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportNumber | string | Yes | Unique identifier for the accountability report |
| reportDate | datetime | Yes | Date the report was generated |
| submissionDate | datetime | No | Date the report was submitted to the relevant authority |
| status | string | Yes | draft, submitted, approved, or rejected |
| content | string | No | Full text content of the accountability report |
| approvalStatus | string | Yes | pending, approved, or rejected |

Relations: → FiscalYear (many-to-one), → Organization (many-to-one), → Person (many-to-one), → DigitalDocument (one-to-many)

### ConsolidationGroup (`schema:Organization`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| name | string | Yes | Name of the consolidation group |
| consolidationMethod | string | Yes | full, proportional, or equity |
| status | string | Yes | Status of the consolidation group |
| parentOrganization | string | No | Parent organization identifier |
| eliminationRules | object | No | Consolidation elimination rules for inter-company transactions |

Relations: → Organization (one-to-many), → ConsolidatedReport (one-to-many)

### ConsolidatedReport (`schema:Report`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| reportNumber | string | Yes | Unique identifier for the consolidated report |
| reportDate | datetime | Yes | Date of the consolidated report |
| consolidationMethod | string | Yes | Method used for consolidation |
| status | string | Yes | draft, finalized, published, or archived |
| eliminationsApplied | boolean | No | Whether inter-company eliminations have been applied |
| isPublished | boolean | No | Whether the consolidated report is published |

Relations: → ConsolidationGroup (many-to-one), → FiscalYear (many-to-one), → BalanceSheet (one-to-many)

### RevenueStream (`schema:Offer`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| streamName | string | Yes | Name of the revenue source |
| category | string | Yes | product sales, service fees, grants, licensing, rental income, interest income |
| currency | string | Yes | ISO 4217 currency code |
| annualTarget | object | No | MonetaryAmount {value, currency} |
| isActive | boolean | No | Whether this revenue stream is currently active |

Relations: → JournalEntry (one-to-many)

## Seed Data

Seed data follows `@self` envelope format (ADR-001). Values use general Dutch municipality/consultancy context.

### FiscalYear

```json
[
  { "@self": { "register": "shillinq", "schema": "fiscal-year", "slug": "fy-2024" },
    "year": 2024, "startDate": "2024-01-01", "endDate": "2024-12-31",
    "isClosed": true, "closingDate": "2025-03-31" },
  { "@self": { "register": "shillinq", "schema": "fiscal-year", "slug": "fy-2025" },
    "year": 2025, "startDate": "2025-01-01", "endDate": "2025-12-31",
    "isClosed": false },
  { "@self": { "register": "shillinq", "schema": "fiscal-year", "slug": "fy-2023" },
    "year": 2023, "startDate": "2023-01-01", "endDate": "2023-12-31",
    "isClosed": true, "closingDate": "2024-04-15" }
]
```

### GeneralLedgerAccount

```json
[
  { "@self": { "register": "shillinq", "schema": "general-ledger-account", "slug": "gl-1000" },
    "accountNumber": "1000", "accountName": "Kas", "accountType": "Asset", "currency": "EUR" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-account", "slug": "gl-1200" },
    "accountNumber": "1200", "accountName": "Debiteuren", "accountType": "Asset", "currency": "EUR" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-account", "slug": "gl-4000" },
    "accountNumber": "4000", "accountName": "Omzet dienstverlening", "accountType": "Revenue", "currency": "EUR" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-account", "slug": "gl-6000" },
    "accountNumber": "6000", "accountName": "Personeelskosten", "accountType": "Expense", "currency": "EUR" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-account", "slug": "gl-8000" },
    "accountNumber": "8000", "accountName": "Eigen vermogen", "accountType": "Equity", "currency": "EUR" }
]
```

### JournalEntry

```json
[
  { "@self": { "register": "shillinq", "schema": "journal-entry", "slug": "je-2025-001" },
    "entryDate": "2025-01-15T00:00:00Z", "entryNumber": "JE-2025-001",
    "description": "Factuur ontvangen — IT leverancier Atos", "debitAmount": 5000.00,
    "creditAmount": 5000.00, "isBalanced": true, "accountCode": "1200",
    "journalCode": "purchase", "reference": "INV-2025-001", "vatAmount": 1050.00,
    "departmentCode": "ICT-01" },
  { "@self": { "register": "shillinq", "schema": "journal-entry", "slug": "je-2025-002" },
    "entryDate": "2025-02-01T00:00:00Z", "entryNumber": "JE-2025-002",
    "description": "Salariskosten januari 2025", "debitAmount": 45000.00,
    "creditAmount": 45000.00, "isBalanced": true, "accountCode": "6000",
    "journalCode": "general", "departmentCode": "HR-01" },
  { "@self": { "register": "shillinq", "schema": "journal-entry", "slug": "je-2025-003" },
    "entryDate": "2025-03-31T00:00:00Z", "entryNumber": "JE-2025-003",
    "description": "Kwartaalafsluiting Q1 — omzetboeking", "debitAmount": 120000.00,
    "creditAmount": 120000.00, "isBalanced": true, "accountCode": "4000",
    "journalCode": "sales" }
]
```

### BalanceSheet

```json
[
  { "@self": { "register": "shillinq", "schema": "balance-sheet", "slug": "bs-2024-q4" },
    "reportDate": "2024-12-31T00:00:00Z", "totalAssets": 780000.00,
    "totalLiabilities": 290000.00, "totalEquity": 490000.00, "currency": "EUR", "status": "published" },
  { "@self": { "register": "shillinq", "schema": "balance-sheet", "slug": "bs-2025-q1" },
    "reportDate": "2025-03-31T00:00:00Z", "totalAssets": 850000.00,
    "totalLiabilities": 320000.00, "totalEquity": 530000.00, "currency": "EUR", "status": "final" },
  { "@self": { "register": "shillinq", "schema": "balance-sheet", "slug": "bs-2025-q2" },
    "reportDate": "2025-06-30T00:00:00Z", "totalAssets": 920000.00,
    "totalLiabilities": 380000.00, "totalEquity": 540000.00, "currency": "EUR", "status": "draft" }
]
```

### TrialBalance

```json
[
  { "@self": { "register": "shillinq", "schema": "trial-balance", "slug": "tb-2024-q4" },
    "reportDate": "2024-12-31T00:00:00Z", "totalDebits": 1250000.00,
    "totalCredits": 1250000.00, "isBalanced": true, "status": "final", "preparedBy": "M. Jansen" },
  { "@self": { "register": "shillinq", "schema": "trial-balance", "slug": "tb-2025-q1" },
    "reportDate": "2025-03-31T00:00:00Z", "totalDebits": 850000.00,
    "totalCredits": 850000.00, "isBalanced": true, "status": "verified", "preparedBy": "J. de Vries" },
  { "@self": { "register": "shillinq", "schema": "trial-balance", "slug": "tb-2025-q2" },
    "reportDate": "2025-06-30T00:00:00Z", "totalDebits": 920000.00,
    "totalCredits": 920000.00, "isBalanced": true, "status": "draft", "preparedBy": "A. van den Berg" }
]
```

### AccountabilityReport

```json
[
  { "@self": { "register": "shillinq", "schema": "accountability-report", "slug": "ver-2023-001" },
    "reportNumber": "VER-2023-001", "reportDate": "2024-03-10T00:00:00Z",
    "submissionDate": "2024-03-15T00:00:00Z", "status": "approved", "approvalStatus": "approved" },
  { "@self": { "register": "shillinq", "schema": "accountability-report", "slug": "ver-2024-001" },
    "reportNumber": "VER-2024-001", "reportDate": "2025-03-15T00:00:00Z",
    "submissionDate": "2025-03-20T00:00:00Z", "status": "submitted", "approvalStatus": "pending" },
  { "@self": { "register": "shillinq", "schema": "accountability-report", "slug": "ver-2024-002" },
    "reportNumber": "VER-2024-002", "reportDate": "2025-04-01T00:00:00Z",
    "status": "draft", "approvalStatus": "pending" }
]
```

### ConsolidationGroup

```json
[
  { "@self": { "register": "shillinq", "schema": "consolidation-group", "slug": "cg-utrecht" },
    "name": "Gemeente Utrecht Groep", "consolidationMethod": "full", "status": "active" },
  { "@self": { "register": "shillinq", "schema": "consolidation-group", "slug": "cg-deltawerken" },
    "name": "Stichting Deltawerken Holding", "consolidationMethod": "proportional",
    "status": "active", "parentOrganization": "Deltawerken BV" },
  { "@self": { "register": "shillinq", "schema": "consolidation-group", "slug": "cg-vng" },
    "name": "VNG Gemeentefonds Groep", "consolidationMethod": "equity", "status": "draft" }
]
```

### ConsolidatedReport

```json
[
  { "@self": { "register": "shillinq", "schema": "consolidated-report", "slug": "cons-2023-001" },
    "reportNumber": "CONS-2023-001", "reportDate": "2024-04-20T00:00:00Z",
    "consolidationMethod": "proportional", "status": "published",
    "eliminationsApplied": true, "isPublished": true },
  { "@self": { "register": "shillinq", "schema": "consolidated-report", "slug": "cons-2024-001" },
    "reportNumber": "CONS-2024-001", "reportDate": "2025-04-15T00:00:00Z",
    "consolidationMethod": "full", "status": "finalized",
    "eliminationsApplied": true, "isPublished": false },
  { "@self": { "register": "shillinq", "schema": "consolidated-report", "slug": "cons-2025-q1" },
    "reportNumber": "CONS-2025-Q1", "reportDate": "2025-04-30T00:00:00Z",
    "consolidationMethod": "full", "status": "draft",
    "eliminationsApplied": false, "isPublished": false }
]
```

### FinancialReport

```json
[
  { "@self": { "register": "shillinq", "schema": "financial-report", "slug": "fr-2024-annual-pdf" },
    "reportType": "Annual", "reportFormat": "PDF", "reportStatus": "Published",
    "generatedAt": "2025-03-31T10:00:00Z" },
  { "@self": { "register": "shillinq", "schema": "financial-report", "slug": "fr-2025-management-excel" },
    "reportType": "Management", "reportFormat": "Excel", "reportStatus": "Approved",
    "generatedAt": "2025-04-15T14:30:00Z" },
  { "@self": { "register": "shillinq", "schema": "financial-report", "slug": "fr-2025-cons-xml" },
    "reportType": "Consolidated", "reportFormat": "XML", "reportStatus": "Draft",
    "generatedAt": "2025-05-01T09:00:00Z" }
]
```

### RevenueStream

```json
[
  { "@self": { "register": "shillinq", "schema": "revenue-stream", "slug": "rs-subsidies" },
    "streamName": "Gemeentelijke subsidies", "category": "grants", "currency": "EUR", "isActive": true },
  { "@self": { "register": "shillinq", "schema": "revenue-stream", "slug": "rs-ict-dienstverlening" },
    "streamName": "Dienstverlening ICT", "category": "service fees", "currency": "EUR", "isActive": true },
  { "@self": { "register": "shillinq", "schema": "revenue-stream", "slug": "rs-zaalhuur" },
    "streamName": "Zaalhuur en evenementen", "category": "rental income", "currency": "EUR", "isActive": true },
  { "@self": { "register": "shillinq", "schema": "revenue-stream", "slug": "rs-leges" },
    "streamName": "Legesopbrengsten", "category": "licensing", "currency": "EUR", "isActive": true },
  { "@self": { "register": "shillinq", "schema": "revenue-stream", "slug": "rs-rente" },
    "streamName": "Rente-inkomsten", "category": "interest income", "currency": "EUR", "isActive": false }
]
```

### GeneralLedgerEntry

```json
[
  { "@self": { "register": "shillinq", "schema": "general-ledger-entry", "slug": "gle-2025-001" },
    "entryDate": "2025-01-15T00:00:00Z", "accountNumber": "1200",
    "accountName": "Debiteuren", "debitAmount": 5000.00,
    "description": "Factuur IT leverancier Atos", "reference": "INV-2025-001", "status": "posted" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-entry", "slug": "gle-2025-002" },
    "entryDate": "2025-02-01T00:00:00Z", "accountNumber": "6000",
    "accountName": "Personeelskosten", "debitAmount": 45000.00,
    "description": "Salariskosten januari 2025", "status": "posted" },
  { "@self": { "register": "shillinq", "schema": "general-ledger-entry", "slug": "gle-2025-003" },
    "entryDate": "2025-03-31T00:00:00Z", "accountNumber": "4000",
    "accountName": "Omzet dienstverlening", "creditAmount": 120000.00,
    "description": "Kwartaalomzet Q1 2025", "status": "posted" }
]
```

## Reuse Analysis

Per ADR-012, the following OpenRegister platform services are leveraged — no custom equivalents will be built:

| Need | Platform Service | Custom code |
|------|-----------------|-------------|
| CRUD for all 11 entities | `ObjectService.saveObject()` / `findAll()` / `deleteObject()` | None |
| List views with pagination and filtering | `CnIndexPage` + `useListView` + `CnDataTable` | None |
| Detail views with sidebar | `CnDetailPage` + `CnObjectSidebar` | None |
| Create/edit forms | `CnFormDialog` (schema-driven) | None |
| CSV/Excel/JSON export | `ExportService` + `CnMassExportDialog` | None |
| Bulk import (trial balance, chart of accounts) | `ImportService` + `CnMassImportDialog` | None |
| Full change tracking and audit trail | `AuditTrailService` (automatic) | None |
| RBAC and field-level permissions | `AuthorizationService` + `PropertyRbacHandler` | None |
| Notification dispatch | `NotificationService` | App-specific event types |
| Workflow status transitions | `WorkflowEngineController` | Shillinq workflow definition |
| Dashboard KPIs and charts | `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` | None |
| File attachment for PDF reports | `FileService` | None |
| Full-text search across GL entries | `IndexService` | None |
| Scheduled background processing | Nextcloud job queue | Year-end and overdue-escalation job logic |
| PDF generation | docudesk integration | Shillinq-specific report template |
| XBRL/ESEF export | None (not in platform) | `XbrlExportService` (custom) |
| Balance validation (debit = credit) | None (domain rule) | `JournalEntryService` (custom) |
| Posting guard for closed fiscal years | None (domain rule) | `FiscalYearService` (custom) |
| Multi-entity consolidation | None (domain rule) | `ConsolidationService` (custom) |

**No overlap found** with existing OpenRegister core services for the custom logic items above. The custom services implement domain-specific financial accounting rules that are not general-purpose platform capabilities.
