# Tasks: Berap / Marap — Bestuursrapportages

## Phase 1: Foundation & schema setup

### 1.1 App scaffolding
- [ ] Create Nextcloud app structure: `apps/financeq/` with standard layout
  - `lib/` — PHP service classes
  - `src/` — Vue components and TypeScript
  - `tests/` — PHPUnit + Vitest
  - `docs/` — OpenSpec specs and ADR references
  - `appinfo/info.xml`, `appinfo/routes.php`, `manifest.json`
- [ ] Set up GitHub Actions CI/CD pipeline (lint, test, coverage, SonarQube)
- [ ] Configure `.claude/settings.json` for hydra-gates: ADR-010 (NL Design System), ADR-005 (security), ADR-023 (authorization)

### 1.2 OpenRegister schema definitions
- [ ] Create `lib/Settings/financeq_register.json` with four main schemas:
  - [ ] **Rapportage** schema: properties {type, peilperiode, peilperiode_jaar, peildatum, status, opgesteld_door, opgesteld_op, vastgesteld_door, vastgesteld_op, programmaregel_ids, toelichting_ids, publiceren, notities_interne}
    - Status enum: concept, intern, bestuurlijk, vastgesteld
    - Type enum: berap, marap
  - [ ] **ProgrammaRegel** schema: properties {rapportage_id, programma_slug, programma_naam, begroting_primitief, begroting_na_wijziging, realisatie_ytd, afwijking_absoluut, afwijking_percentage, drempel_overschreden, drempelwaarde_percentage, drempelwaarde_absoluut, prognose_jaareinde, prognose_berekend, prognose_onderbouwing, toelichting_id}
    - afwijking_absoluut and afwijking_percentage as `x-openregister-calculations` (derived fields)
  - [ ] **Toelichting** schema: properties {rapportage_id, programma_slug, programmaregel_id, oorzaak, maatregel, risico, prognose_onderbouwing, status_review, opgesteld_door, opgesteld_op, reviewed_door, reviewed_op, review_notities}
    - status_review enum: in_process, reviewed_approved, reviewed_requested_changes
  - [ ] **BegrotingswijzigingTrigger** schema: properties {rapportage_id, programma_slug, bedrag, motivatie, status_bw, decidesk_motion_id, gevoegd_op}
    - status_bw enum: voorgesteld, aangenomen, afgewezen
- [ ] Add lifecycle definition to Rapportage schema via `x-openregister-lifecycle`:
  - [ ] States: concept → intern (if marap) or concept → bestuurlijk (if berap) → vastgesteld
  - [ ] Transitions with guards:
    - `concept → intern`: requires type=marap AND all drempel_overschreden Toelichtingen reviewed_approved
    - `concept → bestuurlijk`: requires type=berap AND all drempel_overschreden Toelichtingen reviewed_approved
    - `intern → vastgesteld` / `bestuurlijk → vastgesteld`: requires reviewer approval
  - [ ] Register lifecycle guard class: `\OCA\Financeq\Lifecycle\ToelichtigingRequiredGuard`
- [ ] Add calculated fields via `x-openregister-calculations` for ProgrammaRegel:
  - [ ] `afwijking_absoluut` = realisatie_ytd - begroting_na_wijziging
  - [ ] `afwijking_percentage` = (afwijking_absoluut / begroting_na_wijziging) * 100
  - [ ] Guard logic: if (|afwijking_percentage| >= drempelwaarde_percentage OR |afwijking_absoluut| >= drempelwaarde_absoluut) then drempel_overschreden=true
- [ ] Register four OpenRegister objects/schemas in manifest:
  - financeq-rapportages (register)
  - financeq-programmaregels (register)
  - financeq-toelichtingen (register)
  - financeq-bw-triggers (register)
- [ ] Add seed data to financeq_register.json: 3-5 example Rapportages + ProgrammaRegels + Toelichtingen per register (see design.md Seed data section)
  - [ ] Include Dutch values: gemeente names, programme names (Onderwijs, Sociale Diensten, etc.), realistic amounts, valid BSNs (11-proef), Dutch street names
  - [ ] Mark with `@self` envelope per ADR-001

---

## Phase 2: Backend services

### 2.1 Snapshot job (REQ-001)
- [ ] Create `\OCA\Financeq\Service\FinanceqSnapshotJob extends Job` class
  - [ ] Method: `execute(IJobList $jobList)` — synchronously called on rapportage creation
  - [ ] Implementation:
    - [ ] Accept parameters: {rapportage_id, peilperiode, peildatum}
    - [ ] Query openconnector-synced tables for:
      - [ ] Begrotingsregister: all actieve programmes, begroting_primitief, begroting_na_wijziging at peildatum
      - [ ] Grootboek (YTD): sum of transactions per programme from Jan 1 to peildatum
    - [ ] For each programme, create ProgrammaRegel object via ObjectService.saveObject()
    - [ ] Set begroting_primitief, begroting_na_wijziging, realisatie_ytd as immutable snapshots
    - [ ] Return array of created ProgrammaRegel IDs; update Rapportage.programmaregel_ids
  - [ ] Error handling: if openconnector sync stale (peildatum too far in past), return warning; allow manual re-sync
  - [ ] Performance: bulk insert via ObjectService batch API if available, else optimize query with index on programma_slug + peildatum

### 2.2 Forecast service (REQ-004)
- [ ] Create `\OCA\Financeq\Service\FinanceqForecastService` class
  - [ ] Method: `calculatePrognose(ProgrammaRegel $regel, int $peilperiode_year): float`
    - [ ] Input: ProgrammaRegel with realisatie_ytd, peildatum
    - [ ] Calculate days_elapsed = days from Jan 1 to peildatum
    - [ ] Return prognose_berekend = realisatie_ytd / (days_elapsed / 365)
    - [ ] Example: Q1 (90 days) realisatie_ytd=€1M → prognose_berekend=€4M
  - [ ] Method: `validatePrognoseOverride(float $prognose_jaareinde, string $onderbouwing): bool`
    - [ ] Validate: onderbouwing is non-empty, min 100 chars
    - [ ] Return true if valid
  - [ ] No UI; called from backend endpoint when ProgrammaRegel updated

### 2.3 Deviation flagging (REQ-002)
- [ ] Leverage `x-openregister-calculations` guard in schema (see 1.2)
- [ ] Create `\OCA\Financeq\Lifecycle\ToelichtigingRequiredGuard extends LifecycleGuard` class
  - [ ] Method: `isSatisfied(Object $rapportage, string $transition): bool`
    - [ ] Get all ProgrammaRegels linked to rapportage
    - [ ] For each regel with drempel_overschreden=true:
      - [ ] Check: does a Toelichting exist AND status_review=reviewed_approved?
      - [ ] If any regel fails this check, return false (transition blocked)
    - [ ] Return true (transition allowed)
  - [ ] Called by lifecycle engine on status transitions

### 2.4 Begrotingswijziging trigger (REQ-005)
- [ ] Create `\OCA\Financeq\Service\FinanceqBegrotingswijzigingService` class
  - [ ] Method: `createTrigger(string $rapportage_id, string $programma_slug, float $bedrag, string $motivatie): BegrotingswijzigingTrigger`
    - [ ] Create BegrotingswijzigingTrigger object via ObjectService
    - [ ] Set status_bw=voorgesteld, gevoegd_op=now
    - [ ] Return trigger object
  - [ ] Method: `dispatchToDecidesk(BegrotingswijzigingTrigger $trigger): string`
    - [ ] Construct webhook payload: {rapportage_slug, programma_slug, bedrag, motivatie, bw_trigger_slug}
    - [ ] POST to configured decidesk webhook endpoint (from app config)
    - [ ] Expect response: {motion_id, meeting_id, meeting_date}
    - [ ] Update trigger.decidesk_motion_id with returned motion_id
    - [ ] Save trigger via ObjectService
    - [ ] Return motion_id
  - [ ] Error handling: if decidesk unavailable, queue webhook for retry via `WebhookService.retry()`

### 2.5 Decidesk webhook handler (REQ-005)
- [ ] Create incoming webhook listener: `\OCA\Financeq\WebhookHandler\DecideskBegrotingswijzigingHandler extends WebhookHandler`
  - [ ] Method: `handle(array $payload): void`
    - [ ] Extract motion_id, new status (voorgesteld/aangenomen/afgewezen)
    - [ ] Find BegrotingswijzigingTrigger by decidesk_motion_id
    - [ ] Update status_bw in trigger object
    - [ ] Save via ObjectService
    - [ ] If status_bw=aangenomen, send notification to controller: "Budget amendment accepted in Motion [id]"
  - [ ] Register in openregister webhook subscriptions for Motion state transitions

### 2.6 Trend service (REQ-008)
- [ ] Create `\OCA\Financeq\Service\FinanceqTrendService` class
  - [ ] Method: `getTrendData(string $programma_slug, DateTime $from, DateTime $to): array`
    - [ ] Query all vastgestelde Rapportages for gemeentes matching programma_slug, filtered by peilperiode between $from and $to
    - [ ] For each Rapportage, find linked ProgrammaRegel for programma_slug
    - [ ] Build array: {peilperiode: [{begroting, realisatie, prognose}]}
    - [ ] Query linked BegrotingswijzigingTriggers and add status markers (aangenomen/afgewezen)
    - [ ] Return array for chart rendering
  - [ ] No UI; called from API endpoint

### 2.7 PDF/Excel export (REQ-007)
- [ ] Leverage ExportService (provided by openregister) to generate PDF/Excel
- [ ] Create `\OCA\Financeq\Service\FinanceqExportService extends ExportService`
  - [ ] Override: customize PDF template to include Rapportage header, ProgrammaRegel table, Toelichting narratives, audit footer
  - [ ] Add method: `generateAuditFooter(Rapportage $rapport): string`
    - [ ] Include: rapportage_id, peildatum, vastgesteld_door, vastgesteld_op, SHA256 hash of rapportage
  - [ ] On vastgesteld status transition, auto-call: `exportRapportageToPdf($rapportage_id)` and attach to FileService
  - [ ] Same for Excel via `exportRapportageToExcel()`
- [ ] Register export templates in manifest

### 2.8 Immutability enforcement (REQ-007)
- [ ] Create `\OCA\Financeq\Listener\RapportageVastgesteldListener extends EventListener`
  - [ ] On Rapportage.status→vastgesteld event:
    - [ ] Snapshot all ProgrammaRegel + Toelichting JSON to rapportage.snapshot_json field
    - [ ] Generate and store SHA256 hash
    - [ ] Mark ProgrammaRegels and Toelichtingen as read-only via RBAC (admin only can edit post-vast)
  - [ ] Create middleware/hook to enforce 403 Forbidden on PUT/POST to ProgrammaRegel/Toelichting if parent Rapportage.status=vastgesteld

### 2.9 Notification dispatch (REQ-003, REQ-005, REQ-006)
- [ ] Create `\OCA\Financeq\Service\FinanceqNotificationService extends NotificationService`
  - [ ] Method: `notifyToelichtigngAssigned(Toelichting $t, User $assigned_reviewer): void`
    - [ ] Create NC notification to reviewer: "Toelichting awaiting review: [ProgrammaNaam] in Rapportage [Q1-2026]"
  - [ ] Method: `notifyToelichtigngRejected(Toelichting $t, User $programme_owner, string $feedback): void`
    - [ ] Notify programme-owner of reviewer request-changes
  - [ ] Method: `notifyRapportageReady(Rapportage $r, array $reviewers): void`
    - [ ] Notify assigned reviewers: "Rapportage Q1-2026 ready for review (type=marap/berap)"
  - [ ] Method: `notifyBegrotingswijzigingDispatched(BegrotingswijzigingTrigger $t, string $motion_id): void`
    - [ ] Notify controller: "Budget amendment sent to Motion [id] in meeting [date]"

---

## Phase 3: Frontend components

### 3.1 Rapportage list view
- [ ] Component: `src/components/RapportageList.vue`
  - [ ] Use CnDataTable + useListView composable
  - [ ] Columns: type (icon: berap=building, marap=lock), peilperiode, status (timeline badge), opgesteld_op, # drempel_overschreden, actions
  - [ ] Filters: type, peilperiode, status, opgesteld_door
  - [ ] Row action: click → open RapportageDetail
  - [ ] Toolbar: "Nieuwe rapportage" button → RapportageCreateDialog

### 3.2 Rapportage create dialog
- [ ] Component: `src/components/RapportageCreateDialog.vue`
  - [ ] Form fields:
    - [ ] Dropdown: type (berap/marap, required)
    - [ ] Dropdown: peilperiode (Q1/Q2/Q3/Q4/jaar, required)
    - [ ] Spin box: year (default: current year)
  - [ ] On submit:
    - [ ] POST /api/v1/financeq/rapportages with {type, peilperiode, peilperiode_jaar}
    - [ ] Show loading spinner: "Creating rapportage and loading data..."
    - [ ] On success: navigate to RapportageDetail
    - [ ] On error: show error toast

### 3.3 Rapportage detail view
- [ ] Component: `src/components/RapportageDetail.vue`
  - [ ] Use CnDetailPage + CnDetailGrid + CnTimelineStages
  - [ ] Sections:
    - [ ] **Status timeline**: concept → intern/bestuurlijk → vastgesteld (current state highlighted, unavailable states greyed)
    - [ ] **Summary card**: # programmes total, # with drempel_overschreden, € total afwijking_absoluut
    - [ ] **ProgrammaRegel table**: sortable, filterable by drempel_overschreden / search programma_naam
      - [ ] Columns: programma_naam, begroting_na_wijziging, realisatie_ytd, afwijking_absoluut (red/green highlight), afwijking_percentage, drempel_overschreden (icon), actions
      - [ ] Row action: click → expand inline Toelichting editor OR open ProgrammaRegelDetail modal
      - [ ] Inline action: "Voorstel begrotingswijziging" button (visible if drempel_overschreden=true)
  - [ ] Modals:
    - [ ] Inline Toelichting editor (REQ-003)
    - [ ] Voorstel begrotingswijziging dialog (REQ-005)
  - [ ] Workflow controls:
    - [ ] Button: "Voorleggen intern review" (if type=marap, status=concept) → advances to status=intern (if ToelichtigingRequiredGuard satisfied)
    - [ ] Button: "Voorleggen bestuurlijk" (if type=berap, status=concept) → advances to status=bestuurlijk (if guard satisfied)
    - [ ] Button: "Goedkeuren & vaststellen" (if reviewer, status=intern/bestuurlijk) → advances to status=vastgesteld
  - [ ] Top toolbar:
    - [ ] Download PDF button (calls GET /api/v1/financeq/rapportages/{id}/export/pdf, saves file)
    - [ ] Download Excel button
    - [ ] Share/publish button (if type=berap, status=vastgesteld) → sets publiceren=true, dispatches to opencatalogi

### 3.4 ProgrammaRegel detail modal
- [ ] Component: `src/components/ProgrammaRegelDetail.vue`
  - [ ] Read-only display of: programma_naam, begroting_primitief, begroting_na_wijziging, realisatie_ytd, afwijking_*, prognose_berekend
  - [ ] Editable fields (if permitted by RBAC):
    - [ ] prognose_jaareinde (number, optional)
    - [ ] prognose_onderbouwing (markdown, required if prognose_jaareinde set)
  - [ ] Linked Toelichting display (if exists): summary of oorzaak/maatregel/risico/prognose_onderbouwing with "View full" link
  - [ ] Buttons:
    - [ ] "Edit prognose" → toggle editable fields
    - [ ] "Edit toelichting" → open ToelichtigingEditor modal
    - [ ] "Voorstel begrotingswijziging" (if drempel_overschreden=true) → open BegrotingswijzigingDialog

### 3.5 Toelichting editor form
- [ ] Component: `src/components/ToelichtigingEditor.vue`
  - [ ] Form fields (all required, markdown support):
    - [ ] **Oorzaak** (textarea, min 50 chars, max 2000): "Why did we deviate?"
    - [ ] **Maatregel** (textarea, min 50 chars, max 2000): "What corrective actions?"
    - [ ] **Risico** (textarea, min 50 chars, max 2000): "Forward-looking risks?"
    - [ ] **Prognose onderbouwing** (textarea, min 50 chars, max 2000): "Year-end forecast justification"
  - [ ] Tabs: Edit | Preview
  - [ ] Preview tab: render all four fields as formatted markdown with embedded links/lists
  - [ ] Validation: all fields required, min-char checks, show error messages inline
  - [ ] Buttons: Save | Save as draft | Cancel
  - [ ] On save:
    - [ ] POST /api/v1/financeq/rapportages/{rapportage_id}/toelichtingen with {programmaregel_id, oorzaak, maatregel, risico, prognose_onderbouwing}
    - [ ] Show success toast
    - [ ] Assigned reviewer is notified (backend dispatches notification)
  - [ ] Accessibility: labels properly associated with textareas, keyboard-navigable

### 3.6 Toelichting review view
- [ ] Component: `src/components/ToelichtigingReview.vue`
  - [ ] Read-only display of programme-owner's four fields (formatted markdown)
  - [ ] Reviewer feedback section:
    - [ ] Textarea: review_notities (required if choosing "Request changes")
    - [ ] Buttons: "Approve" | "Request changes" | "Cancel"
  - [ ] On "Approve": PUT /api/v1/financeq/toelichtingen/{id} with {status_review: reviewed_approved, reviewed_door, reviewed_op}
  - [ ] On "Request changes": PUT with {status_review: reviewed_requested_changes, review_notities}
  - [ ] Programme-owner is notified of outcome

### 3.7 Begrotingswijziging dialog
- [ ] Component: `src/components/BegrotingswijzigingDialog.vue`
  - [ ] Pre-filled fields (editable):
    - [ ] Programma: programma_naam (read-only)
    - [ ] Bedrag: afwijking_absoluut (pre-filled, editable number)
    - [ ] Motivatie: summary of linked Toelichting (oorzaak + maatregel, editable markdown)
  - [ ] Validation: bedrag required, motivatie min 50 chars
  - [ ] Buttons: Submit | Cancel
  - [ ] On submit:
    - [ ] POST /api/v1/financeq/bw-triggers with {rapportage_id, programma_slug, bedrag, motivatie}
    - [ ] Show loading: "Sending to decidesk..."
    - [ ] On success: show result toast with Motion ID and meeting date
    - [ ] On error: show error toast with retry option

### 3.8 Trend dashboard / chart
- [ ] Component: `src/components/TrendDashboard.vue`
  - [ ] Controls:
    - [ ] Dropdown: select programme_slug (from all programmes with rapportages)
    - [ ] Date range picker: from/to (defaults: 2 years back, now)
  - [ ] Chart: line chart (ApexCharts via CnChartWidget)
    - [ ] X-axis: peilperiode labels (Q1-2024, Q2-2024, ..., Q4-2026)
    - [ ] Y-axis: EUR (millions, thousands, auto-scale)
    - [ ] Three lines:
      - [ ] Begroting (dotted): begroting_na_wijziging per kwartaal
      - [ ] Realisatie (solid): realisatie_ytd per kwartaal
      - [ ] Prognose (dashed): prognose_jaareinde per kwartaal
    - [ ] Markers: green flag (BW aangenomen), red flag (BW afgewezen)
  - [ ] Hover tooltip: figures + Toelichting oorzaak summary (200 chars)
  - [ ] Buttons: Download as PNG | Download as CSV
  - [ ] Data source: GET /api/v1/financeq/trend/{programma_slug}?from=&to=

### 3.9 Router configuration
- [ ] Add routes in `src/router/index.ts`:
  - [ ] `/financeq` → RapportageList
  - [ ] `/financeq/{id}` → RapportageDetail
  - [ ] `/financeq/trend` → TrendDashboard

### 3.10 Store configuration
- [ ] Use `createObjectStore('financeq-rapportages')` for Rapportage CRUD store
- [ ] Use `createObjectStore('financeq-programmaregels')` for ProgrammaRegel store
- [ ] Use `createObjectStore('financeq-toelichtingen')` for Toelichting store
- [ ] Use `createObjectStore('financeq-bw-triggers')` for BegrotingswijzigingTrigger store
- [ ] No custom Pinia stores; leverage provided schema-driven stores

### 3.11 Manifest configuration
- [ ] Update `manifest.json`:
  - [ ] `name`: "Financeq — Bestuursrapportages"
  - [ ] `apps`: list app routes + icons
  - [ ] Define app navigation entry: "Rapportages" (icon: financial-report) → `/financeq`
  - [ ] Define sidebar apps or external integrations (opencatalogi, docudesk, decidesk links)

---

## Phase 4: Integration & webhooks

### 4.1 Openconnector integration setup
- [ ] Create `lib/Connector/FinanceqOpenconnectorAdapter.php`
  - [ ] Implement connector interface to read realisatie from synced financial tables
  - [ ] Methods:
    - [ ] `pullRealisation(DateTime $peildatum, array $programme_slugs): array` — fetch YTD realisatie per programme
    - [ ] `pullBegroting(DateTime $peildatum, array $programme_slugs): array` — fetch begroting_primitief + begroting_na_wijziging
  - [ ] Configuration: store openconnector endpoint URL in financeq app config
- [ ] Create scheduled sync job: `\OCA\Financeq\BackgroundJob\SyncGrootboekJob` (runs daily at 6am)
  - [ ] Calls FinanceqOpenconnectorAdapter.pullRealisation() and caches results
  - [ ] Updates any open (status ≠ vastgesteld) Rapportages with latest realisatie (for users who refresh)

### 4.2 Decidesk webhook integration
- [ ] Create incoming webhook handler: `src/WebhookHandler/DecideskBegrotingswijzigingHandler.ts` (see 2.5)
- [ ] Create outgoing webhook: `\OCA\Financeq\Service\FinanceqBegrotingswijzigingService::dispatchToDecidesk()` (see 2.4)
- [ ] Configure decidesk integration in app config: decidesk_webhook_url, decidesk_api_key (if needed)
- [ ] Register webhook in openregister subscriptions: subscribe to BegrotingswijzigingTrigger creation + Motion state changes

### 4.3 Opencatalogi integration
- [ ] Create webhook handler: `\OCA\Financeq\WebhookHandler\RapportagePublishHandler`
  - [ ] On Rapportage vastgesteld with type=berap and publiceren=true:
    - [ ] Call `opencatalogi.publish()` with rapportage metadata + PDF URL
- [ ] Configure opencatalogi endpoint in app config

### 4.4 Docudesk integration
- [ ] Create webhook handler: `\OCA\Financeq\WebhookHandler\ArchiveRapportageHandler`
  - [ ] On Rapportage vastgesteld:
    - [ ] Call `docudesk.archive()` with rapportage + PDF + audit trail + BBV retention schedule
- [ ] Configure docudesk endpoint in app config
- [ ] Define retention schedule: 10 years for peilperiode=jaar, 7 years for peilperiode=Q*

---

## Phase 5: Authorization & access control

### 5.1 RBAC roles and permissions
- [ ] Define roles in app config / RBAC system:
  - [ ] `financeq.controller` — can create/initiate rapportages, assign Toelichtingen, propose begrotingswijzigingen
  - [ ] `financeq.programmamanager` — can author Toelichtingen for own programmes, edit prognose_jaareinde
  - [ ] `financeq.reviewer` — can review assigned Toelichtingen, approve/request-changes
  - [ ] `financeq.directie` — can review/approve Marap rapportages to vastgesteld
  - [ ] `financeq.college` — can review/approve Berap rapportages, see only own portefeuille programmes
  - [ ] `financeq.griffie` — read-only access to vastgestelde rapportages
  - [ ] `financeq.accountant` — read-only access, can audit vastgestelde rapportages
- [ ] Implement `\OCA\Financeq\Authorization\FinanceqAuthorizationHandler extends AuthorizationHandler`
  - [ ] Method: `canViewRapportage(User $user, Rapportage $r): bool`
    - [ ] If role=controller/reviewer/directie: true (see all rapportages)
    - [ ] If role=college: filter by portefeuille (user can only see ProgrammaRegels for own programmes)
    - [ ] If role=griffie/accountant: true (read-only)
  - [ ] Method: `canEditRapportage(User $user, Rapportage $r): bool`
    - [ ] Return false if Rapportage.status=vastgesteld (immutable)
    - [ ] Otherwise check role + rapportage.status
  - [ ] Method: `canEditToelichting(User $user, Toelichting $t): bool`
    - [ ] Return false if parent Rapportage.status=vastgesteld
    - [ ] Return true if user is toelichting.opgesteld_door (author) OR role=reviewer assigned
  - [ ] Method: `canReviewToelichting(User $user, Toelichting $t): bool`
    - [ ] Return true if user is assigned reviewer OR role=directie/college

### 5.2 Portefeuille filtering (for College members)
- [ ] Create `\OCA\Financeq\Service\PortefeuilleScopeService`
  - [ ] Method: `getPortefeuilleScopes(User $user): array` — fetch portefeuille programmes for college member
  - [ ] Method: `filterByPortefeuille(array $programmes, User $user): array` — filter array by user's portefeuille
- [ ] In RapportageDetail component, apply filter when role=college
- [ ] In ProgrammaRegel table, filter rows to user's portefeuille programmes

---

## Phase 6: Testing

### 6.1 Backend unit tests
- [ ] `tests/Unit/Service/FinanceqSnapshotJobTest.php`
  - [ ] Test rapportage creation → ProgrammaRegel generation
  - [ ] Test snapshot immutability
  - [ ] Test error handling (openconnector unavailable)
- [ ] `tests/Unit/Service/FinanceqForecastServiceTest.php`
  - [ ] Test prognose calculation: Q1 (90 days) → 4x multiplier
  - [ ] Test Q2 (181 days) → ~2x multiplier
  - [ ] Test prognose_onderbouwing validation
- [ ] `tests/Unit/Lifecycle/ToelichtigingRequiredGuardTest.php`
  - [ ] Test guard blocks transition if drempel_overschreden=true AND no reviewed Toelichting
  - [ ] Test guard allows transition if all Toelichtingen reviewed_approved
- [ ] `tests/Unit/Service/FinanceqBegrotingswijzigingServiceTest.php`
  - [ ] Test BegrotingswijzigingTrigger creation
  - [ ] Test webhook dispatch to decidesk
  - [ ] Test error handling (decidesk unavailable)

### 6.2 Integration tests
- [ ] `tests/Integration/RapportageWorkflowTest.php`
  - [ ] Full workflow: create rapportage → flag deviations → assign Toelichting → review → advance status → vastgesteld
  - [ ] Verify PDF/Excel exports generated
  - [ ] Verify immutability post-vastgesteld
- [ ] `tests/Integration/DecideskIntegrationTest.php`
  - [ ] Create rapportage → propose begrotingswijziging → verify Motion created in decidesk
  - [ ] Verify webhook handler updates BegrotingswijzigingTrigger.status_bw on Motion status change

### 6.3 Frontend unit tests (Vitest)
- [ ] `tests/unit/components/RapportageList.spec.ts`
  - [ ] Test list renders, filtering, sorting
  - [ ] Test "Nieuwe rapportage" button opens dialog
- [ ] `tests/unit/components/ToelichtigingEditor.spec.ts`
  - [ ] Test form validation (all fields required, min chars)
  - [ ] Test markdown preview toggle
  - [ ] Test save dispatches to store
- [ ] `tests/unit/components/TrendDashboard.spec.ts`
  - [ ] Test chart renders with correct data
  - [ ] Test programme filter
  - [ ] Test CSV download

### 6.4 Browser tests (test-app / Cypress)
- [ ] `tests/e2e/rapportage-workflow.spec.ts`
  - [ ] Full user journey:
    - [ ] Controller creates rapportage
    - [ ] System populates ProgrammaRegels (verify in UI)
    - [ ] Programme-owner authors Toelichting
    - [ ] Reviewer approves Toelichting
    - [ ] Controller advances rapportage to bestuurlijk
    - [ ] College member reviews (portefeuille filter works)
    - [ ] College member votes to vastgesteld
    - [ ] Verify PDF export generated + immutability enforced
- [ ] `tests/e2e/begrotingswijziging-workflow.spec.ts`
  - [ ] Controller proposes begrotingswijziging from ProgrammaRegel
  - [ ] Verify Motion created in decidesk
  - [ ] Verify webhook updates status_bw when Motion status changes

### 6.5 Performance tests
- [ ] Load test: create rapportage with 200 programmes, measure snapshot job duration (target: < 2s)
- [ ] Trend chart: render 8 kwartalen of data, measure load time (target: < 1s)

---

## Phase 7: Documentation & deployment

### 7.1 Developer documentation
- [ ] Create `docs/architecture.md`: high-level overview, data flow, schema definitions
- [ ] Create `docs/api.md`: REST endpoint reference with examples
- [ ] Create `docs/extensions.md`: guide for extending schema-declarative features (ADR-031)
- [ ] Create `docs/integration.md`: guide to integrating openconnector, decidesk, opencatalogi, docudesk

### 7.2 User documentation
- [ ] Create `docs/user-guide-controller.md`: how to create rapportage, assign Toelichtingen, propose amendments
- [ ] Create `docs/user-guide-programmer.md`: how to author Toelichting, review Toelichting, edit prognose
- [ ] Create `docs/user-guide-reviewer.md`: how to review and approve Toelichtingen
- [ ] Create `docs/user-guide-college.md`: how to review Berap, vote vaststelling, understand portefeuille filtering

### 7.3 Admin configuration guide
- [ ] Document app settings in financeq app config:
  - [ ] Gemeente drempelwaarde_percentage (default 5%)
  - [ ] Gemeente drempelwaarde_absoluut (default €50k)
  - [ ] openconnector endpoint URL
  - [ ] decidesk webhook URL + API key
  - [ ] opencatalogi endpoint URL
  - [ ] docudesk endpoint URL + API key
  - [ ] Portefeuille mappings (college member → programme slugs)

### 7.4 Regulatory compliance documentation
- [ ] BBV mapping: explain how schema enforces mandatory programme structure
- [ ] Audit trail: explain how AuditTrailService tracks all transitions, edits, approvals
- [ ] Data retention: explain how docudesk archival enforces 10-year/7-year schedules
- [ ] Immutability: explain how vastgesteld rapportages are locked + hashed

### 7.5 Release & deployment
- [ ] Prepare release notes: feature summary, breaking changes (none), migration steps (none)
- [ ] Add app to Nextcloud App Store (if applicable)
- [ ] Create deployment checklist:
  - [ ] Install app in staging
  - [ ] Configure openconnector, decidesk, opencatalogi, docudesk endpoints
  - [ ] Seed test data (rapportages with deviations)
  - [ ] Run browser tests on staging
  - [ ] Deploy to production

---

## Deduplication check

Per ADR-031 + ADR-022, this change leverages existing OpenRegister + Nextcloud Vue abstractions:
- [ ] **CRUD operations**: ObjectService.saveObject/findAll/deleteObject — no custom DAO/Repository needed
- [ ] **List views**: CnDataTable + useListView composable — no custom pagination/sorting logic
- [ ] **Detail views**: CnDetailPage + CnDetailGrid — no custom layout components
- [ ] **Forms**: CnFormDialog (auto-generated from schema) — no custom form builder
- [ ] **Export**: ExportService (PDF/Excel) — no custom export controller
- [ ] **Notifications**: NotificationService — no custom notification queue
- [ ] **RBAC**: AuthorizationService — no custom permission checker
- [ ] **Audit trail**: AuditTrailService — automatic via lifecycle events

**No duplicates detected.** All custom code (FinanceqSnapshotJob, FinanceqForecastService, FinanceqBegrotingswijzigingService, ToelichtigingRequiredGuard) addresses domain-specific logic not covered by OR abstractions.

---

## Acceptance criteria

- [ ] All 4 schemas (Rapportage, ProgrammaRegel, Toelichting, BegrotingswijzigingTrigger) defined in financeq_register.json
- [ ] Seed data included (3-5 example objects per schema)
- [ ] All 8 REQ-* requirements implemented and tested
- [ ] All 6 user stories pass acceptance criteria
- [ ] Frontend components render correctly on 320px–1920px viewports
- [ ] PDF/Excel exports generated on vastgesteld
- [ ] Immutability enforced (403 Forbidden on PUT to vastgestelde objects)
- [ ] Lifecycle guards block invalid transitions
- [ ] All webhook integrations (decidesk, opencatalogi, docudesk) tested
- [ ] RBAC roles enforced (controller, programmer-owner, reviewer, directie, college, griffie, accountant)
- [ ] Portefeuille filtering works for college members
- [ ] Browser tests pass (rapportage workflow, begrotingswijziging workflow)
- [ ] Performance targets met (< 2s rapportage creation, < 1s trend chart)
- [ ] Documentation complete (architecture, API, user guides, admin config)
- [ ] Zero hydra-gate violations (ADR-010 NL Design System, ADR-005 security, ADR-023 authorization)
