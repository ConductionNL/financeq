# Design: Berap / Marap — Bestuursrapportages

## Architecture overview

The berap-marap-bestuursrapportages change introduces a new OpenRegister-backed app (`financeq`) that models the quarterly financial reporting workflow. All domain data — Rapportage, ProgrammaRegel, Toelichting, Begrotingswijziging-trigger — are OpenRegister objects following ADR-001 (no custom Entity/Mapper). Lifecycle transitions (rapportage status: concept → intern → bestuurlijk → vastgesteld) are declared via `x-openregister-lifecycle` on the Rapportage schema, enabling audit-trailed state machine, RBAC-per-state, and automatic CloudEvents per ADR-031.

### Integrations (ADR-003 patterns)

- **openconnector** — reads realisatie from external financial systems (Civision, Key2Financiën, AFAS, Unit4); on-demand sync or scheduled pull via a `FinanceqConnectorSyncJob`
- **decidesk** — receives Begrotingswijziging-trigger via webhook + creates Motion agenda item
- **opencatalogi** — receives Berap Rapportage objects marked `publiceren=true` and exposes as Woo-compliant public documents
- **docudesk** — archives vastgestelde Rapportages with 10-year/7-year retention schedules per BBV

### Declarative-vs-imperative decision

Per ADR-031, the following are **declarative (schema register)**:
- **Rapportage lifecycle** (`x-openregister-lifecycle`): status transitions concept → intern → bestuurlijk → vastgesteld with guards (e.g., all Toelichtingen reviewed before → bestuurlijk)
- **Afwijking calculation** (`x-openregister-calculations`): `afwijking_absoluut` and `afwijking_percentage` derived fields on ProgrammaRegel, computed from begroting_na_wijziging and realisatie_ytd
- **Toelichting requirement enforcement** (`x-openregister-lifecycle` guard): when ProgrammaRegel.drempel_overschreden=true, the schema guard prevents status advance until Toelichting is non-empty

The following are **imperative (PHP service code)**:
- **Forecast calculation logic** (`FinanceqForecastService`): lineaire extrapolatie of realisatie-pace is domain-specific heuristic not yet covered by OR's calculation engine; includes optional override with onderbouwing
- **Begrotingswijziging trigger** (`FinanceqBegrotingswijzigingService`): creates Begrotingswijziging-trigger and dispatches webhook to decidesk; integrates with external decidesk state machine
- **Grootboek snapshot ingestion** (`FinanceqSnapshotJob`): reads from openconnector-synced tables, calculates YTD realisatie per programme, creates/updates ProgrammaRegel records on rapportage initiation
- **Trend aggregations** (`FinanceqTrendService`): cross-rapportage comparative analytics (evolution of prognose, cumulative deviation, kwartaal-over-kwartaal deltas) — spans multiple Rapportage objects and includes complex filtering logic

### Reuse analysis

Per ADR-031 and ADR-022, the app leverages:
- **ObjectService** (create/read/update/delete) — all CRUD on Rapportage, ProgrammaRegel, Toelichting, Begrotingswijziging-trigger
- **CnDetailPage** + **CnDetailGrid** — Rapportage detail view with nested ProgrammaRegel list and Toelichting inline editor
- **CnFormDialog** — auto-generated forms for ProgrammaRegel edit (if allowed by RBAC) and Toelichting narrative entry
- **CnDataTable** — list view of Rapportages with filtering by peilperiode, type, status
- **CnTimelineStages** — visual workflow progression (concept → intern → bestuurlijk → vastgesteld)
- **ExportService** — PDF/Excel generation of vastgestelde Rapportage with audit trail and embedded narratives
- **NotificationService** — notify programme-owners of assigned Toelichtingen; notify College of Berap ready for review
- **FileService** — attach signed bestuursbesluit and audit reports to Rapportage
- **AuthorizationService** — RBAC: controller can initiate; programme-owner can edit Toelichting for own programmes; directie can review/approve; College members see only own portefeuille

No duplication with openregister services detected. Forecast and snapshot logic is domain-specific to the BBV/IV3 financial-reporting context.

## Data model

### Rapportage (register: financeq-rapportages, schema: Rapportage)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `@self` | object | ✓ | OpenRegister self-reference (register, schema, slug) |
| `type` | enum | ✓ | `berap` or `marap` — determines review route |
| `peilperiode` | enum | ✓ | `Q1`, `Q2`, `Q3`, `Q4`, `jaar` — quarter or annual |
| `peilperiode_jaar` | integer | ✓ | Year (e.g., 2026) |
| `peildatum` | date | ✓ | Report-as-of date (last day of quarter or year) |
| `status` | enum | ✓ | Lifecycle: `concept`, `intern`, `bestuurlijk`, `vastgesteld` (see `x-openregister-lifecycle`) |
| `opgesteld_door` | relation | ✓ | User/role reference who created rapport |
| `opgesteld_op` | timestamp | ✓ | Creation timestamp |
| `vastgesteld_door` | relation | ✗ | User/role reference who approved to vastgesteld |
| `vastgesteld_op` | timestamp | ✗ | Vaststelling timestamp; null until vastgesteld=true |
| `programmaregel_ids` | array | ✓ | Cross-register relation: array of ProgrammaRegel slugs |
| `toelichting_ids` | array | ✓ | Cross-register relation: array of Toelichting slugs |
| `publiceren` | boolean | ✓ | Flag for opencatalogi publication (default: false for marap, true for vastgestelde berap) |
| `notities_interne` | text | ✗ | Internal controller notes (not part of formal rapportage) |

### ProgrammaRegel (register: financeq-programmaregels, schema: ProgrammaRegel)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `@self` | object | ✓ | OpenRegister self-reference |
| `rapportage_id` | relation | ✓ | Back-reference to parent Rapportage slug |
| `programma_slug` | string | ✓ | Slug/ID of the programme (from begrotingsregister) |
| `programma_naam` | string | ✓ | Display name (denormalized from begrotingsregister for snapshot immutability) |
| `begroting_primitief` | decimal | ✓ | Original approved budget (snapshot from peildatum) |
| `begroting_na_wijziging` | decimal | ✓ | Budget after any amendments (snapshot from peildatum) |
| `realisatie_ytd` | decimal | ✓ | Year-to-date realization from grootboek (snapshot from peildatum) |
| `afwijking_absoluut` | decimal | derived | `realisatie_ytd - begroting_na_wijziging` (calculated field, `x-openregister-calculations`) |
| `afwijking_percentage` | decimal | derived | `afwijking_absoluut / begroting_na_wijziging * 100` (calculated field) |
| `drempel_overschreden` | boolean | ✓ | Set by lifecycle guard: `|afwijking_percentage| >= drempelwaarde OR |afwijking_absoluut| >= absolute_drempel` |
| `drempelwaarde_percentage` | decimal | ✓ | Gemeente-specific threshold (default 5%) — stored on ProgrammaRegel for audit trail |
| `drempelwaarde_absoluut` | decimal | ✓ | Gemeente-specific absolute threshold (default €50k) |
| `prognose_jaareinde` | decimal | ✗ | Manual override forecast; if null, use calculated lineaire extrapolatie |
| `prognose_berekend` | decimal | derived | Lineaire extrapolatie: `realisatie_ytd / (peildatum_doy / 365) * 1` (calculated field) |
| `prognose_onderbouwing` | text | ✗ | Narrative explaining any prognose override |
| `toelichting_id` | relation | ✗ | Back-reference to Toelichting slug (if drempel_overschreden) |

### Toelichting (register: financeq-toelichtingen, schema: Toelichting)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `@self` | object | ✓ | OpenRegister self-reference |
| `rapportage_id` | relation | ✓ | Back-reference to parent Rapportage slug |
| `programma_slug` | string | ✓ | Programme slug (denormalized for clarity) |
| `programmaregel_id` | relation | ✓ | Back-reference to ProgrammaRegel slug |
| `oorzaak` | text | ✓ | Structured field: why did we deviate? (markdown) |
| `maatregel` | text | ✓ | What measures are we taking? (markdown) |
| `risico` | text | ✓ | What are the forward-looking risks? (markdown) |
| `prognose_onderbouwing` | text | ✓ | Year-end forecast justification (markdown) |
| `status_review` | enum | ✓ | `in_process`, `reviewed_approved`, `reviewed_requested_changes` (RBAC: only assigned reviewer can approve) |
| `opgesteld_door` | relation | ✓ | Programme-owner who authored |
| `opgesteld_op` | timestamp | ✓ | Creation timestamp |
| `reviewed_door` | relation | ✗ | Reviewer user/role |
| `reviewed_op` | timestamp | ✗ | Review timestamp |
| `review_notities` | text | ✗ | Reviewer feedback if requested-changes |

### Begrotingswijziging-trigger (register: financeq-bw-triggers, schema: BegrotingswijzigingTrigger)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `@self` | object | ✓ | OpenRegister self-reference |
| `rapportage_id` | relation | ✓ | Source Rapportage slug |
| `programma_slug` | string | ✓ | Programme affected |
| `bedrag` | decimal | ✓ | Amendment amount (can be positive or negative) |
| `motivatie` | text | ✓ | Structured reason (pre-filled from ProgrammaRegel + Toelichting) |
| `status_bw` | enum | ✓ | `voorgesteld`, `aangenomen`, `afgewezen` — synced with decidesk Motion status |
| `decidesk_motion_id` | relation | ✗ | Back-reference to decidesk Motion (created by webhook handler) |
| `gevoegd_op` | timestamp | ✓ | Timestamp when triggered from rapportage |

## Seed data

### Rapportage examples

**rapportage-berap-q1-2026**
```json
{
  "@self": { "register": "financeq-rapportages", "schema": "Rapportage", "slug": "rapportage-berap-q1-2026" },
  "type": "berap",
  "peilperiode": "Q1",
  "peilperiode_jaar": 2026,
  "peildatum": "2026-03-31",
  "status": "concept",
  "opgesteld_door": "john.controller@gemeente.nl",
  "opgesteld_op": "2026-04-01T10:30:00Z",
  "programmaregel_ids": ["regel-q1-001-onderwijs", "regel-q1-002-sociale"],
  "toelichting_ids": [],
  "publiceren": false,
  "notities_interne": "Expected deficit in sociale diensten due to influx of asylum seekers."
}
```

**rapportage-marap-q1-2026**
```json
{
  "@self": { "register": "financeq-rapportages", "schema": "Rapportage", "slug": "rapportage-marap-q1-2026" },
  "type": "marap",
  "peilperiode": "Q1",
  "peilperiode_jaar": 2026,
  "peildatum": "2026-03-31",
  "status": "intern",
  "opgesteld_door": "john.controller@gemeente.nl",
  "opgesteld_op": "2026-04-01T10:30:00Z",
  "vastgesteld_door": "maria.director@gemeente.nl",
  "vastgesteld_op": "2026-04-10T15:00:00Z",
  "programmaregel_ids": ["regel-q1-001-onderwijs", "regel-q1-002-sociale", "regel-q1-003-sport"],
  "toelichting_ids": ["toelichting-q1-001"],
  "publiceren": false
}
```

### ProgrammaRegel examples

**regel-q1-001-onderwijs**
```json
{
  "@self": { "register": "financeq-programmaregels", "schema": "ProgrammaRegel", "slug": "regel-q1-001-onderwijs" },
  "rapportage_id": "rapportage-berap-q1-2026",
  "programma_slug": "prog-onderwijs",
  "programma_naam": "Onderwijs & Kinderopvang",
  "begroting_primitief": 8500000,
  "begroting_na_wijziging": 8500000,
  "realisatie_ytd": 7890000,
  "afwijking_absoluut": -610000,
  "afwijking_percentage": -7.18,
  "drempel_overschreden": true,
  "drempelwaarde_percentage": 5.0,
  "drempelwaarde_absoluut": 50000,
  "prognose_jaareinde": null,
  "prognose_berekend": 33500000,
  "prognose_onderbouwing": null,
  "toelichting_id": null
}
```

**regel-q1-002-sociale**
```json
{
  "@self": { "register": "financeq-programmaregels", "schema": "ProgrammaRegel", "slug": "regel-q1-002-sociale" },
  "rapportage_id": "rapportage-berap-q1-2026",
  "programma_slug": "prog-sociale-diensten",
  "programma_naam": "Sociale Diensten & Maatschappelijke Ondersteuning",
  "begroting_primitief": 12000000,
  "begroting_na_wijziging": 12500000,
  "realisatie_ytd": 13200000,
  "afwijking_absoluut": 700000,
  "afwijking_percentage": 5.6,
  "drempel_overschreden": true,
  "drempelwaarde_percentage": 5.0,
  "drempelwaarde_absoluut": 50000,
  "prognose_jaareinde": 52800000,
  "prognose_berekend": 52800000,
  "prognose_onderbouwing": null,
  "toelichting_id": "toelichting-q1-001"
}
```

### Toelichting example

**toelichting-q1-001**
```json
{
  "@self": { "register": "financeq-toelichtingen", "schema": "Toelichting", "slug": "toelichting-q1-001" },
  "rapportage_id": "rapportage-berap-q1-2026",
  "programma_slug": "prog-sociale-diensten",
  "programmaregel_id": "regel-q1-002-sociale",
  "oorzaak": "Higher-than-expected intake of asylum seekers in Q1 (342 vs 180 budgeted). Integration costs and temporary housing subsidies driving the €700k overage.",
  "maatregel": "Invoiced state government for asylum integration support per §4.82 BBV (expected reimbursement €550k by June). Negotiating volume discount with temporary housing provider.",
  "risico": "If asylum intake continues at Q1 pace, full-year overage could reach €2.8M. State reimbursement may lag or be disputed.",
  "prognose_onderbouwing": "Extrapolated Q1 intake pace: 1,368 asylum seekers by year-end. Assumed €3,870/person average (down from Q1 €4,100 after efficiency gains). Conservative estimate: €5.3M full-year expenditure.",
  "status_review": "reviewed_approved",
  "opgesteld_door": "anna.programmer@gemeente.nl",
  "opgesteld_op": "2026-04-05T14:20:00Z",
  "reviewed_door": "peter.reviewer@gemeente.nl",
  "reviewed_op": "2026-04-08T11:45:00Z",
  "review_notities": null
}
```

### Begrotingswijziging-trigger example

**bw-trigger-q1-2026-001**
```json
{
  "@self": { "register": "financeq-bw-triggers", "schema": "BegrotingswijzigingTrigger", "slug": "bw-trigger-q1-2026-001" },
  "rapportage_id": "rapportage-berap-q1-2026",
  "programma_slug": "prog-sociale-diensten",
  "bedrag": 550000,
  "motivatie": "Reallocation of €550k pending state reimbursement for asylum integration costs. Current deficit: €700k; expected state funding: €550k; remaining municipality gap: €150k to be covered from reserves.",
  "status_bw": "voorgesteld",
  "decidesk_motion_id": null,
  "gevoegd_op": "2026-04-08T16:30:00Z"
}
```

## Frontend flows

### Create rapportage (Controller)
1. Controller clicks "Nieuwe rapportage" on financeq dashboard
2. Dialog opens: select type (berap/marap), peilperiode (Q1/Q2/Q3/Q4/jaar), year
3. On confirm:
   - `RapportageService.initiate(type, peilperiode, peilperiode_jaar)` → calls `FinanceqSnapshotJob` synchronously
   - Job reads live begroting + realisatie from `openconnector`-synced tables
   - Creates Rapportage object, status=concept
   - Generates one ProgrammaRegel per actieve programma
   - Returns list view of ProgrammaRegels with deviation-flags
4. Controller scans deviations (drempel_overschreden=true), notes which ones need Toelichtingen

### Author toelichting (Programme owner)
1. Controller assigns programme-owner to drempel_overschreden=true ProgrammaRegel
2. Programme-owner navigates to Toelichting form
3. Form pre-loads four markdown text areas with labels:
   - **Oorzaak**: Why the deviation?
   - **Maatregel**: What corrective actions?
   - **Risico**: Outlook risks?
   - **Prognose**: Year-end forecast with justification?
4. Programme-owner fills all four fields; can preview as formatted markdown
5. On save: Toelichting object created, status_review=in_process, opgesteld_door=current_user
6. Notification sent to assigned reviewer (e.g., finance director)

### Review toelichting (Reviewer)
1. Reviewer sees inbox notification: "Toelichting awaiting review: prog-sociale-diensten in Rapportage Q1-2026"
2. Opens Toelichting detail page
3. Reads pre-filled oorzaak/maatregel/risico/prognose_onderbouwing in markdown
4. Can add review feedback in review_notities field
5. Clicks "Approve" or "Request changes":
   - Approve: status_review=reviewed_approved, reviewed_door=current_user, reviewed_op=now
   - Request changes: status_review=reviewed_requested_changes, sends notification back to programme-owner
6. Workflow engine checks: when all Toelichtingen → reviewed_approved, allows rapportage to transition to intern (if type=marap) or bestuurlijk (if type=berap)

### Propose begrotingswijziging (Controller)
1. On ProgrammaRegel detail where drempel_overschreden=true, controller sees button: "Voorstel begrotingswijziging"
2. Clicking opens pre-filled form:
   - **Programma**: auto-filled from ProgrammaRegel
   - **Bedrag**: suggested as afwijking_absoluut (user can adjust)
   - **Motivatie**: pre-filled with summary of oorzaak + maatregel from linked Toelichting (user can customize)
   - **Status**: voorgesteld
3. On submit:
   - BegrotingswijzigingTrigger object created
   - `FinanceqBegrotingswijzigingService.dispatchToDecidesk()` sends webhook to decidesk
   - decidesk-listener (webhook handler in decidesk app) creates Motion with standard template
   - Motion linked back to BegrotingswijzigingTrigger via `decidesk_motion_id`
4. UI shows: "Begrotingswijziging sent to agenda item XYZ in meeting YYYY-MM-DD"

### Review rapportage (Directie / College)
1. Rapportage status transitions from concept → intern (Marap) or concept → bestuurlijk (Berap) via workflow
2. Directie (for Marap) or College members (for Berap) receive notification
3. Open rapportage detail page:
   - Kanban-style timeline showing concept → intern → bestuurlijk → vastgesteld with current state highlighted
   - Summary card: totaal actieve programmes, aantal drempel_overschreden, totaal afwijking_absoluut
   - Filterable list of ProgrammaRegels with expand-to-see-Toelichting
   - For College members: portefeuille-filter limits to own programmes (via AuthorizationService)
4. Can drill into each ProgrammaRegel + linked Toelichting
5. Vote to approve (clicks "Goedkeuren") → workflow advances to vastgesteld
6. On vastgesteld:
   - Rapportage marked immutable (status=vastgesteld, vastgesteld_door/vastgesteld_op recorded)
   - PDF export generated with embedded Toelichtingen and audit trail
   - If publiceren=true and type=berap, webhook sent to opencatalogi
   - Notification sent to Griffie + Raad

### Trend dashboard
1. User navigates to "Rapportages" → "Trend" tab
2. Selects programme and date range (e.g., last 2 years = 8 kwartalen)
3. Dashboard renders line chart:
   - X-axis: peilperiode (Q1-2024 → Q4-2026)
   - Y-axis: EUR
   - Three lines:
     - **Begroting**: begroting_na_wijziging per kwartaal
     - **Realisatie**: realisatie_ytd per kwartaal (cumulative)
     - **Prognose**: prognose_jaareinde per kwartaal (showing how outlook evolved)
   - Markers on kwartalen where a BegrotingswijzigingTrigger was aangenomen (green flag) or afgewezen (red flag)
4. Hover on any kwartaal to see exact figures and linked Toelichting summary

## PDF export format (for vastgestelde rapportage)

The PDF export (generated by ExportService) includes:
- **Header**: Gemeente/Provincie name, rapportage type (Berap/Marap), peilperiode, peildatum
- **Executive summary**: totaal begroting, totaal realisatie, totaal afwijking, count of programmes with drempel_overschreden
- **Programme table**: one row per ProgrammaRegel with columns: programma_naam, begroting_primitief, begroting_na_wijziging, realisatie_ytd, afwijking_absoluut, afwijking_percentage, drempel_overschreden, linked-toelichting summary
- **Toelichting narratives**: full markdown-rendered Toelichting per drempel_overschreden programme (oorzaak/maatregel/risico/prognose)
- **Audit footer**: rapportage slug, peildatum, vastgesteld_door, vastgesteld_op, SHA256 hash of immutable snapshot
- **Begrotingswijziging appendix**: list of BegrotingswijzigingTriggers with status (voorgesteld/aangenomen/afgewezen) and link to decidesk Motion

## Integration points

### openconnector (realisatie input)
- **On demand**: Controller clicks "Sync realisatie from financial system" → calls `openconnector.pull(grootboek, [Q1-2026], peildatum=2026-03-31)` → updates synced financial tables
- **Scheduled**: `FinanceqConnectorSyncJob` runs daily at 6am CET, pulls latest realisatie for open rapportages
- **Schema**: openconnector exposes `GrootboekLine` (programma_slug, rekeningcode, amount_ytd, peildatum); financeq aggregates by programma

### decidesk (begrotingswijziging agendaing)
- **Webhook trigger**: `BegrotingswijzigingTrigger.dispatchToDecidesk()` POSTs to decidesk's incoming webhook
- **Payload**: { rapportage_slug, programma_slug, bedrag, motivatie, bw_trigger_slug }
- **Handler in decidesk**: decidesk-listener creates Motion with type=begrotingswijziging and links back to BegrotingswijzigingTrigger via relation
- **Sync back**: When Motion transitions to aangenomen/afgewezen, decidesk POSTs back to financeq; BegrotingswijzigingTrigger.status_bw updated

### opencatalogi (public rapportage publishing)
- **Trigger**: Rapportage transitioned to vastgesteld AND publiceren=true AND type=berap
- **Webhook**: financeq POSTs to opencatalogi.publish() with { rapportage_slug, pdf_url, markdown_content, woo_tags }
- **Result**: OpenCatalogi registers Rapportage as a Woo-compliant public document with publication date and archive link

### docudesk (archive + retention)
- **Trigger**: Rapportage transitioned to vastgesteld
- **Webhook**: financeq POSTs to docudesk.archive() with { rapportage_slug, pdf_attachment, xml_audit_trail, retention_schedule_bbv }
- **Retention schedule**: 10 years for jaarrekening (year=1), 7 years for interim Q rapportages

## Style and accessibility

- **NL Design System tokens** (per ADR-010): All colors, fonts, spacing from `nldesign` token sets
- **WCAG AA**: Toelichting form labels, legend prompts, PDF alt-text on embedded tables
- **Responsive**: List views functional at 320px; detail forms/PDFs optimal at 768px+
- **Keyboard navigation**: Tab through ProgrammaRegel table, Enter to expand/collapse Toelichting, Enter to submit form
