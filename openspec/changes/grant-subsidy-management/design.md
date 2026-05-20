# Design: Grant & Subsidy Management — Shillinq

## Context

Shillinq is a self-hosted Nextcloud business administration suite targeting Dutch public-sector organisations (municipalities, provinces, ZBO's) and SMBs. This change adds a grant and subsidy management module, the highest-demand unserved feature cluster in the product (demand scores 1675 and 1593 for the top two feature requests).

All domain data is stored in OpenRegister. The module follows the Conduction 3-layer (Controller → Service → Mapper) backend pattern and Vue 2 + Pinia frontend pattern. No custom CRUD, search, pagination, file upload, or audit logging is built — these are provided by the platform for free.

## Goals / Non-Goals

**Goals:**
- Full subsidy lifecycle: scheme creation → application intake → eligibility assessment → award decision → payment milestones → accountability review → dossier closure
- Integrity safeguards: sanctions checking, risk scoring, fraud investigation
- Dutch public-sector compliance: SiSa eligibility tracking, BBV year-end commitments report, financial year lock
- Public portal: published scheme browser with filtering by category and target group
- Automated notifications: accountability deadline reminders at 30 and 7 days
- Auditor statement registration for large subsidies

**Non-Goals:**
- Custom CRUD UI — handled by `CnFormDialog` / `CnIndexPage` / `CnDetailPage`
- Custom file upload — handled by `CnObjectSidebar` → `CnFilesTab` via `FileService`
- Custom audit logging — handled by `CnObjectSidebar` → `CnAuditTrailTab` via `AuditTrailService`
- Peppol/UBL e-invoicing integration (handled by existing Shillinq invoicing module)
- BRP/BAG address verification (out of scope for this change)
- Rebuilding Nextcloud authentication or role management

## Architecture Decisions

### Decision 1: All entities in OpenRegister, no custom Mapper

Following ADR-001, all five entities (SubsidyScheme, SubsidyApplication, Grant, GrantPortfolio, AuditorStatement) are stored as OpenRegister objects. No custom `Entity` or `Mapper` classes. CRUD, search, filtering, pagination, file attachments, and audit trails are provided by the platform.

Services call `ObjectService::findObject($register, $schema, $id)` and `ObjectService::saveObject($register, $schema, $object)` using the 3-positional-argument form.

### Decision 2: Public portal uses #[PublicPage] controller

The scheme browser endpoint (`GET /api/public/subsidy-schemes`) must be publicly accessible without Nextcloud authentication so citizens can browse without an account. This controller is annotated `#[PublicPage]` and `#[NoCSRFRequired]`. A CORS OPTIONS route is registered in `routes.php`. It returns only schemes with `isPublished: true` and `status: open`.

### Decision 3: Sanctions check as background job on submission

Checking EU Sanctions, OFAC, and Rijksoverheid exclusion registers at submission time is done asynchronously via a background job (`SanctionsCheckJob`) to avoid blocking the user's submission flow. The application moves to `under-review` status immediately; if a hit is found, the application is flagged `sanctions-hold` and the integrity officer is notified. This avoids coupling external API latency to the user request.

### Decision 4: Payment block enforced in GrantService, not controller

The rule "block final payment if mandatory conditions are unmet" is business logic that belongs in `GrantService::processPayment()`, not in the controller. The service checks that `accountabilityReport.status === approved` and all required conditions are met before calling the payment disbursement. If blocked, the service returns a structured error listing the outstanding conditions. The controller returns this as a 422 response.

### Decision 5: Year-end lock uses OpenRegister object locking

`ObjectService::lockObject($register, $schema, $id)` is used to lock individual grant records after financial year sign-off. The `GrantService::lockFinancialYear($year)` method locks all grants where `awardDate` falls within the year. Locked records are read-only; attempts to modify them via the API return a 423 Locked response.

### Decision 6: Risk scoring is computed, not stored

Risk scores for the integrity officer dashboard are computed on-the-fly by `RiskScoringService` from existing application and grant data (amount, applicant history, scheme risk profile). Scores are not persisted — they are recalculated on each dashboard load. This avoids stale scores and eliminates a separate migration if the scoring model changes.

### Decision 7: Decision letters via docudesk integration

Decision letter PDF generation reuses the existing Shillinq docudesk integration. `SubsidyApplicationService::generateDecisionLetter($applicationId, $outcome)` calls the docudesk template engine with application data. No custom PDF generation code is written.

## Data Model

All entities match ADR-000 exactly. Relations use OpenRegister relation mechanism (register + schema + objectId). No foreign keys or embedded objects.

### SubsidyScheme (`schema:GovernmentService`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| schemeId | string | Yes | Unique scheme identifier |
| name | string | Yes | Subsidy scheme name |
| description | string | No | Scheme description and purpose |
| maxGrant | number | No | Maximum grant per applicant (€) |
| minGrant | number | No | Minimum grant per applicant (€) |
| isPublished | boolean | No | Published to public portal |
| publishedDate | datetime | No | Date published to portal |
| governmentLevel | string | No | `national`, `provincial`, or `municipal` |

**Relations:**
- → Organization (many-to-one): funding authority
- → Grant (one-to-many): grants awarded under this scheme

**Status lifecycle:** `draft` → `pending-approval` → `approved` → `published` → `closed`

**Additional fields managed via schema metadata:** `legalBasis`, `totalBudget`, `targetGroup`, `applicationPeriodStart`, `applicationPeriodEnd`, `requiredDocuments`, `status`, `category`

### SubsidyApplication (`schema:Application`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| applicationId | string | Yes | Unique application identifier |
| requestedAmount | number | Yes | Requested grant amount (€) |
| status | string | Yes | `draft`, `submitted`, `under-review`, `sanctions-hold`, `approved`, `rejected`, `withdrawn` |
| submissionDate | datetime | No | Date of submission |
| reviewDate | datetime | No | Date review was completed |
| notes | string | No | Internal reviewer notes |

**Relations:**
- → SubsidyScheme (many-to-one): scheme being applied for
- → Organization (many-to-one): applicant organisation
- → Document (one-to-many): supporting documents

**Additional fields:** `riskScore` (computed), `sanctionsCheckResult`, `decisionLetterRef`, `applicantContactPerson`

### Grant (`schema:Grant`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| grantId | string | Yes | Unique grant identifier |
| name | string | Yes | Grant name |
| awardedAmount | number | Yes | Awarded amount (€) |
| awardDate | datetime | Yes | Date of award decision |
| status | string | Yes | `active`, `completed`, `suspended`, `revoked`, `overpaid`, `closed` |
| accountingStandard | string | No | BBV, IV3, SiSa, or other standard applied |
| isSISAEligible | boolean | No | Eligible for Single Information Single Audit |

**Relations:**
- → SubsidyScheme (many-to-one): scheme under which grant was awarded
- → Organization (many-to-one): grantee organisation
- → GrantPortfolio (many-to-one): portfolio this grant belongs to

**Additional fields:** `totalDisbursed`, `accountabilityDeadline`, `projectEndDate`, `finalPaymentDate`, `closureTimestamp`, `recoveryAmount`, `paymentHold`, `financialYearLock`, `advancePaymentAmount`, `advancePaymentDueDate`

### GrantPortfolio (`schema:Collection`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| portfolioId | string | Yes | Unique portfolio identifier |
| name | string | Yes | Portfolio name |
| description | string | No | Portfolio description |
| totalGrantValue | number | No | Total value of all grants (€) |
| complianceStatus | string | No | `compliant`, `non-compliant`, `under-review` |
| concentrationRiskLevel | string | No | `low`, `medium`, `high` |
| lastAuditDate | datetime | No | Date of last portfolio audit |

**Relations:**
- → Organization (many-to-one): owning organisation
- → Grant (one-to-many): grants in portfolio

### AuditorStatement (`schema:Statement`)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| statementId | string | Yes | Unique statement identifier |
| verificationDate | datetime | Yes | Date of auditor verification |
| isVerified | boolean | Yes | Statement has been verified |
| findings | string | No | Audit findings and observations |
| verdict | string | No | `approved`, `rejected`, `conditional` |

**Relations:**
- → Grant (many-to-one): grant this statement covers
- → Person (many-to-one): auditor (registered accountant)
- → DigitalDocument (one-to-one): uploaded statement document

**Additional fields:** `auditorRegistrationNumber`, `opinionType`, `thresholdExceeded`, `statementDate`

## Seed Data

Realistic Dutch seed objects per schema. Loaded via `lib/Settings/shillinq_register.json` using `@self` envelope. Must be idempotent (match by slug).

### SubsidyScheme — 5 objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "subsidy-scheme", "slug": "duurzaamheidssubsidie-utrecht-2026" },
    "schemeId": "SCH-2026-001",
    "name": "Duurzaamheidssubsidie Utrecht 2026",
    "description": "Subsidie voor duurzame maatregelen door MKB-bedrijven in de gemeente Utrecht, zoals zonnepanelen, isolatie en warmtepompen.",
    "maxGrant": 25000,
    "minGrant": 1000,
    "isPublished": true,
    "publishedDate": "2026-01-15T00:00:00Z",
    "governmentLevel": "municipal",
    "status": "published",
    "category": "duurzaamheid",
    "targetGroup": "mkb",
    "legalBasis": "Subsidieverordening gemeente Utrecht 2025",
    "totalBudget": 500000,
    "applicationPeriodStart": "2026-02-01T00:00:00Z",
    "applicationPeriodEnd": "2026-06-30T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-scheme", "slug": "cultuurfonds-amsterdam-2026" },
    "schemeId": "SCH-2026-002",
    "name": "Cultuurfonds Amsterdam 2026",
    "description": "Subsidie voor culturele activiteiten en producties door Amsterdamse culturele instellingen en verenigingen.",
    "maxGrant": 50000,
    "minGrant": 2500,
    "isPublished": true,
    "publishedDate": "2026-01-10T00:00:00Z",
    "governmentLevel": "municipal",
    "status": "published",
    "category": "cultuur",
    "targetGroup": "vereniging",
    "legalBasis": "Subsidieverordening Amateurkunst Amsterdam 2024",
    "totalBudget": 1200000,
    "applicationPeriodStart": "2026-02-15T00:00:00Z",
    "applicationPeriodEnd": "2026-04-30T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-scheme", "slug": "sport-bewegen-eindhoven-2026" },
    "schemeId": "SCH-2026-003",
    "name": "Sport & Bewegen Gemeente Eindhoven 2026",
    "description": "Subsidie voor sportverenigingen die sportdeelname bij jeugd bevorderen in de gemeente Eindhoven.",
    "maxGrant": 10000,
    "minGrant": 500,
    "isPublished": true,
    "publishedDate": "2026-01-20T00:00:00Z",
    "governmentLevel": "municipal",
    "status": "published",
    "category": "sport",
    "targetGroup": "vereniging",
    "legalBasis": "Subsidieregeling Sport Eindhoven 2026",
    "totalBudget": 200000,
    "applicationPeriodStart": "2026-03-01T00:00:00Z",
    "applicationPeriodEnd": "2026-05-15T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-scheme", "slug": "slim-mkb-opleiding-2026" },
    "schemeId": "SCH-2026-004",
    "name": "SLIM Opleiding en Ontwikkeling MKB 2026",
    "description": "Stimuleringsregeling leren en ontwikkelen in MKB-ondernemingen. Nationaal programma voor werknemers- en organisatieontwikkeling.",
    "maxGrant": 5000,
    "minGrant": 1000,
    "isPublished": true,
    "publishedDate": "2026-01-05T00:00:00Z",
    "governmentLevel": "national",
    "status": "published",
    "category": "onderwijs",
    "targetGroup": "mkb",
    "legalBasis": "Regeling SLIM 2020 (Stcrt. 2020, 27819)",
    "totalBudget": 48000000,
    "applicationPeriodStart": "2026-04-01T00:00:00Z",
    "applicationPeriodEnd": "2026-09-30T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-scheme", "slug": "esf-plus-arbeidsmarkt-2026" },
    "schemeId": "SCH-2026-005",
    "name": "ESF+ Arbeidsmarktintegratie 2026-2027",
    "description": "Europees Sociaal Fonds Plus subsidie voor arbeidsmarktintegratie van kwetsbare groepen en langdurig werklozen.",
    "maxGrant": 100000,
    "minGrant": 10000,
    "isPublished": false,
    "governmentLevel": "national",
    "status": "draft",
    "category": "arbeidsmarkt",
    "targetGroup": "gemeente",
    "legalBasis": "ESF+ Verordening (EU) 2021/1057",
    "totalBudget": 350000000,
    "isSISAEligible": true
  }
]
```

### SubsidyApplication — 5 objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "subsidy-application", "slug": "app-2026-001-groene-buurt" },
    "applicationId": "APP-2026-001",
    "requestedAmount": 18500,
    "status": "approved",
    "submissionDate": "2026-02-14T09:30:00Z",
    "reviewDate": "2026-03-20T14:00:00Z",
    "notes": "Aanvraag voldoet aan alle criteria. Businessplan is solide, kostenraming realistisch."
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-application", "slug": "app-2026-002-rode-maskers" },
    "applicationId": "APP-2026-002",
    "requestedAmount": 35000,
    "status": "under-review",
    "submissionDate": "2026-02-28T11:15:00Z",
    "notes": "Aanvraag compleet ontvangen. Accountantsverklaring vereist (>€30.000). Lopende integriteitsscan."
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-application", "slug": "app-2026-003-sv-oranje-wit" },
    "applicationId": "APP-2026-003",
    "requestedAmount": 7500,
    "status": "approved",
    "submissionDate": "2026-03-10T10:00:00Z",
    "reviewDate": "2026-04-02T09:30:00Z",
    "notes": "Sportvereniging actief met jeugdprogramma voor 120 jongeren. Aanvraag goedgekeurd."
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-application", "slug": "app-2026-004-bakkerij-van-der-berg" },
    "applicationId": "APP-2026-004",
    "requestedAmount": 4500,
    "status": "submitted",
    "submissionDate": "2026-04-15T16:45:00Z",
    "notes": "SLIM aanvraag voor vakopleidingstraject drie medewerkers bakkerijsector."
  },
  {
    "@self": { "register": "shillinq", "schema": "subsidy-application", "slug": "app-2026-005-gemeente-haarlem-esf" },
    "applicationId": "APP-2026-005",
    "requestedAmount": 85000,
    "status": "draft",
    "notes": "Concept ESF+ aanvraag voor re-integratieproject arbeidsmarkt Haarlemmermeer. Nog in te dienen."
  }
]
```

### Grant — 4 objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "grant", "slug": "grant-2026-001-groene-buurt" },
    "grantId": "GRT-2026-001",
    "name": "Duurzaamheidssubsidie Stichting De Groene Buurt",
    "awardedAmount": 18500,
    "awardDate": "2026-03-25T00:00:00Z",
    "status": "active",
    "accountingStandard": "BBV",
    "isSISAEligible": false,
    "totalDisbursed": 9250,
    "accountabilityDeadline": "2027-03-31T23:59:59Z",
    "projectEndDate": "2026-12-31T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "grant", "slug": "grant-2026-002-sv-oranje-wit" },
    "grantId": "GRT-2026-002",
    "name": "Sport & Bewegen subsidie SV Oranje Wit",
    "awardedAmount": 7500,
    "awardDate": "2026-04-10T00:00:00Z",
    "status": "active",
    "accountingStandard": "BBV",
    "isSISAEligible": false,
    "totalDisbursed": 7500,
    "accountabilityDeadline": "2027-04-30T23:59:59Z",
    "projectEndDate": "2026-12-31T23:59:59Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "grant", "slug": "grant-2025-015-theaterfonds" },
    "grantId": "GRT-2025-015",
    "name": "Cultuurfonds subsidie Theatergroep Nieuwe Horizon",
    "awardedAmount": 42000,
    "awardDate": "2025-05-01T00:00:00Z",
    "status": "completed",
    "accountingStandard": "BBV",
    "isSISAEligible": false,
    "totalDisbursed": 42000,
    "accountabilityDeadline": "2026-04-30T23:59:59Z",
    "projectEndDate": "2025-12-31T23:59:59Z",
    "closureTimestamp": "2026-04-15T10:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "grant", "slug": "grant-2024-088-esf-re-integratie" },
    "grantId": "GRT-2024-088",
    "name": "ESF Re-integratieproject Gemeente Zaandam 2024",
    "awardedAmount": 95000,
    "awardDate": "2024-06-15T00:00:00Z",
    "status": "overpaid",
    "accountingStandard": "SiSa",
    "isSISAEligible": true,
    "totalDisbursed": 97500,
    "recoveryAmount": 2500,
    "accountabilityDeadline": "2025-06-30T23:59:59Z",
    "projectEndDate": "2024-12-31T23:59:59Z"
  }
]
```

### GrantPortfolio — 3 objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "grant-portfolio", "slug": "portfolio-duurzaamheid-2026" },
    "portfolioId": "PTF-2026-001",
    "name": "Duurzaamheidssubsidies Gemeente Utrecht 2026",
    "description": "Alle duurzaamheidssubsidies verstrekt door gemeente Utrecht in het boekjaar 2026.",
    "totalGrantValue": 487500,
    "complianceStatus": "compliant",
    "concentrationRiskLevel": "low",
    "lastAuditDate": "2026-04-01T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "grant-portfolio", "slug": "portfolio-cultuur-sport-2025-2026" },
    "portfolioId": "PTF-2025-002",
    "name": "Cultuur & Sport Portfolio 2025-2026",
    "description": "Geconsolideerde portfolio cultuur- en sportsubsidies voor de gemeenten Amsterdam en Eindhoven.",
    "totalGrantValue": 1850000,
    "complianceStatus": "under-review",
    "concentrationRiskLevel": "medium",
    "lastAuditDate": "2026-01-15T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "grant-portfolio", "slug": "portfolio-esf-nationaal-2024-2027" },
    "portfolioId": "PTF-2024-003",
    "name": "ESF+ Nationaal Programma 2024-2027",
    "description": "Landelijk ESF+ portfolio voor arbeidsmarktintegratie en sociale inclusie. SiSa-rapportageplicht.",
    "totalGrantValue": 12500000,
    "complianceStatus": "non-compliant",
    "concentrationRiskLevel": "high",
    "lastAuditDate": "2025-11-30T00:00:00Z"
  }
]
```

### AuditorStatement — 3 objects

```json
[
  {
    "@self": { "register": "shillinq", "schema": "auditor-statement", "slug": "aud-stmt-2026-001-theaterfonds" },
    "statementId": "AUD-2026-001",
    "verificationDate": "2026-04-10T00:00:00Z",
    "isVerified": true,
    "findings": "Bestedingen aansluiten op activiteitenverslag. Geen onregelmatigheden geconstateerd.",
    "verdict": "approved",
    "auditorRegistrationNumber": "RA-12345",
    "opinionType": "goedkeurend",
    "statementDate": "2026-04-08T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "auditor-statement", "slug": "aud-stmt-2025-007-esf-zaandam" },
    "statementId": "AUD-2025-007",
    "verificationDate": "2025-08-20T00:00:00Z",
    "isVerified": true,
    "findings": "Overschrijding op kostenpost personeelslasten van €2.500. Terugvordering geadviseerd.",
    "verdict": "conditional",
    "auditorRegistrationNumber": "RA-67890",
    "opinionType": "met beperking",
    "statementDate": "2025-08-18T00:00:00Z"
  },
  {
    "@self": { "register": "shillinq", "schema": "auditor-statement", "slug": "aud-stmt-2026-003-groene-buurt-pending" },
    "statementId": "AUD-2026-003",
    "verificationDate": "2026-12-01T00:00:00Z",
    "isVerified": false,
    "findings": null,
    "verdict": null,
    "statementDate": null
  }
]
```

## Reuse Analysis

This change leverages the following OpenRegister and platform services. No custom reimplementation of these capabilities is permitted.

| Capability | Platform Service | Notes |
|-----------|-----------------|-------|
| CRUD operations | `ObjectService.saveObject()` / `findObject()` / `findObjects()` | All 5 entities |
| List + pagination + filtering | `CnIndexPage` + `useListView` | All entity list views |
| Schema-driven forms | `CnFormDialog` | Create/edit for all entities |
| Detail views | `CnDetailPage` + `CnDetailCard` | All entity detail pages |
| File attachments (supporting documents) | `FileService` + `CnObjectSidebar` → `CnFilesTab` | SubsidyApplication documents, AuditorStatement upload |
| Audit trail | `AuditTrailService` + `CnObjectSidebar` → `CnAuditTrailTab` | Automatic on all entities |
| Notifications | `NotificationService` | Accountability deadline reminders |
| Dashboard KPIs | `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` | Grant portfolio dashboard |
| Object locking | `ObjectService.lockObject()` / `unlockObject()` | Financial year lock |
| Background jobs | `IJobList` + OC job queue | `AccountabilityReminderJob`, `SanctionsCheckJob` |
| Export (CSV/Excel) | `CnMassExportDialog` | Application outcome export per scheme |
| Search | `IndexService` + `CnFilterBar` | Scheme browser search and facets |
| RBAC | `AuthorizationService` | Role checks for admin operations |
| Task management | `TasksController` | Accountability report tasks, recovery tasks |
| Decision letter PDF | docudesk template engine (existing Shillinq module) | `SubsidyApplicationService::generateDecisionLetter()` |

**Custom code required only for:**
- `SanctionsCheckService`: HTTP client calling EU Sanctions API, OFAC API, and Rijksoverheid exclusion register REST endpoint
- `RiskScoringService`: scoring algorithm combining amount, applicant history, scheme risk profile
- `AccountabilityReminderJob`: domain-specific reminder schedule (30d + 7d before deadline, on project end date)
- `GrantService`: business rules for payment blocking, overpayment detection, dossier closure, year-end lock
- `FraudInvestigationService`: confidential case creation, payment hold application, access restriction

## API Surface

All endpoints follow ADR-002 URL pattern: `/index.php/apps/shillinq/api/{resource}`.

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/subsidy-schemes` | Nextcloud | List schemes with pagination and filtering |
| POST | `/api/subsidy-schemes` | Nextcloud admin | Create scheme |
| GET | `/api/subsidy-schemes/{id}` | Nextcloud | Get scheme detail |
| PUT | `/api/subsidy-schemes/{id}` | Nextcloud admin | Update scheme |
| POST | `/api/subsidy-schemes/{id}/publish` | Nextcloud admin | Publish scheme to portal |
| GET | `/api/subsidy-applications` | Nextcloud | List applications |
| POST | `/api/subsidy-applications` | Nextcloud | Submit application |
| GET | `/api/subsidy-applications/{id}` | Nextcloud | Get application detail |
| PUT | `/api/subsidy-applications/{id}` | Nextcloud | Update application |
| POST | `/api/subsidy-applications/{id}/generate-letter` | Nextcloud | Generate decision letter |
| GET | `/api/grants` | Nextcloud | List grants |
| POST | `/api/grants` | Nextcloud admin | Create grant |
| GET | `/api/grants/{id}` | Nextcloud | Get grant detail |
| PUT | `/api/grants/{id}` | Nextcloud admin | Update grant |
| POST | `/api/grants/{id}/process-payment` | Nextcloud admin | Process payment with condition check |
| POST | `/api/grants/{id}/close` | Nextcloud admin | Close and archive dossier |
| POST | `/api/grants/lock-year` | Nextcloud admin | Lock financial year |
| GET | `/api/grant-portfolios` | Nextcloud | List portfolios |
| POST | `/api/grant-portfolios` | Nextcloud admin | Create portfolio |
| GET | `/api/grant-portfolios/{id}` | Nextcloud | Get portfolio detail |
| GET | `/api/auditor-statements` | Nextcloud | List statements |
| POST | `/api/auditor-statements` | Nextcloud | Register statement |
| PUT | `/api/auditor-statements/{id}` | Nextcloud | Update statement |
| POST | `/api/auditor-statements/{id}/verify` | Nextcloud | Mark statement as verified |
| GET | `/api/public/subsidy-schemes` | Public (#[PublicPage]) | Published schemes for citizen portal |
| GET | `/api/metrics` | Nextcloud admin | Prometheus metrics |
| GET | `/api/health` | Public | Health check |

## Frontend Pages

Following ADR-004 page construction patterns.

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | `DashboardView.vue` | `CnDashboardPage` with KPI cards (active grants, pending applications, overdue accountability, total disbursed), scheme status chart, advance payment table |
| `/subsidy-schemes` | `SubsidySchemesView.vue` | `CnIndexPage` with scheme list, status badges, publish action |
| `/subsidy-schemes/:id` | `SubsidySchemeDetailView.vue` | `CnDetailPage` with scheme info, linked applications table, linked grants table |
| `/subsidy-applications` | `SubsidyApplicationsView.vue` | `CnIndexPage` with application list, risk score column, status filter |
| `/subsidy-applications/:id` | `SubsidyApplicationDetailView.vue` | `CnDetailPage` with application info, documents section, integrity check result, decision letter action |
| `/grants` | `GrantsView.vue` | `CnIndexPage` with grant list, SiSa flag, accountability deadline column |
| `/grants/:id` | `GrantDetailView.vue` | `CnDetailPage` with disbursement timeline, accountability checklist, payment hold status, auditor statement section |
| `/grant-portfolios` | `GrantPortfoliosView.vue` | `CnIndexPage` with portfolio list, compliance status badges, risk level indicator |
| `/grant-portfolios/:id` | `GrantPortfolioDetailView.vue` | `CnDetailPage` with portfolio metrics, grants table, concentration risk chart |
| `/auditor-statements` | `AuditorStatementsView.vue` | `CnIndexPage` with statement list, verified status, verdict badges |
| `/auditor-statements/:id` | `AuditorStatementDetailView.vue` | `CnDetailPage` with statement details, linked grant, uploaded document |

**Modal/dialog components** (each in its own `.vue` file per ADR-004):

- `src/modals/PublishSchemeModal.vue` — confirmation + preview before publishing to portal
- `src/modals/GenerateDecisionLetterModal.vue` — outcome selection + letter preview + edit
- `src/modals/OpenFraudInvestigationModal.vue` — investigation case creation with grant link
- `src/dialogs/PaymentBlockDialog.vue` — outstanding conditions checklist before payment
- `src/dialogs/LockFinancialYearDialog.vue` — year selection + confirmation for year-end lock
- `src/dialogs/VerifyAuditorStatementDialog.vue` — statement verification form

## Navigation

MainMenu items:
1. Dashboard (`/`)
2. Subsidy Schemes (`/subsidy-schemes`)
3. Applications (`/subsidy-applications`)
4. Grants (`/grants`)
5. Portfolios (`/grant-portfolios`)
6. Auditor Statements (`/auditor-statements`)
