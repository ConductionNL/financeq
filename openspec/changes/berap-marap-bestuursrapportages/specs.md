# Specs: Berap / Marap — Bestuursrapportages

## Requirements

### REQ-001: Generate report from live grootboek snapshot

**Requirement:** On rapportage initiation, the system must populate ProgrammaRegels from current financial data without manual re-entry.

**GIVEN** a concerned controller with role=financier,
**WHEN** the controller initiates a new Rapportage with type ∈ {berap, marap}, peilperiode ∈ {Q1, Q2, Q3, Q4, jaar}, and peilperiode_jaar=YYYY,
**THEN**
- A new Rapportage object is created with status=concept, opgesteld_door=current_user, opgesteld_op=now
- For each actieve programme in the gemeente's begrotingsregister (excluding archived/inactive programmes):
  - One ProgrammaRegel is created per programme
  - `begroting_primitief` ← snapshot from begrotingsregister at peildatum
  - `begroting_na_wijziging` ← snapshot including approved begrotingswijzigingen effective on or before peildatum
  - `realisatie_ytd` ← sum of all transactions in grootboek for that programme from January 1 to peildatum (via openconnector-synced data)
  - `drempelwaarde_percentage` and `drempelwaarde_absoluut` ← gemeente-configurable defaults (settable in app config)
  - All snapshot fields are immutable post-creation
- The controller is presented with a sorted list of ProgrammaRegels (by programma_naam alphabetically)
- Performance: Rapportage + full ProgrammaRegel population completes in < 2 seconds for a gemeente with 200+ programmes

**Acceptance criteria:**
- [ ] POST /rapportages with {type, peilperiode, peilperiode_jaar} creates Rapportage with status=concept
- [ ] All actieve programmes from begrotingsregister appear as ProgrammaRegels (none missing, none extra unless inactive)
- [ ] begroting_primitief and begroting_na_wijziging match begrotingsregister snapshots within €0.01
- [ ] realisatie_ytd sums correctly from openconnector-synced grootboek lines
- [ ] Response includes programmaregel_ids array; UI renders sorted list
- [ ] Load test: 200 programmes + 365-day GTRY aggregation < 2000ms

---

### REQ-002: Automatic deviation flagging at configurable thresholds

**Requirement:** Flag material deviations for mandatory narrative explanation.

**GIVEN** a ProgrammaRegel with afwijking_absoluut and afwijking_percentage calculated,
**WHEN** the Rapportage is opened OR a ProgrammaRegel is created/updated,
**THEN**
- `drempel_overschreden` ← TRUE if ( |afwijking_percentage| ≥ drempelwaarde_percentage ) OR ( |afwijking_absoluut| ≥ drempelwaarde_absoluut )
- `drempel_overschreden` ← FALSE otherwise
- If drempel_overschreden changed from false → true, the system checks: does a Toelichting already exist for this ProgrammaRegel?
  - If not, the lifecycle guard `ToelichtigingRequiredGuard` prevents rapportage status advancement until a Toelichting is linked and reviewed
- A count badge appears on the rapportage card: "X programmes with deviations exceeding threshold"

**Acceptance criteria:**
- [ ] afwijking_absoluut = realisatie_ytd - begroting_na_wijziging (calculated field, immutable)
- [ ] afwijking_percentage = (afwijking_absoluut / begroting_na_wijziging) * 100 (calculated field)
- [ ] drempel_overschreden correctly set based on gemeente-configurable thresholds
- [ ] Lifecycle guard prevents status advance if drempel_overschreden=true AND Toelichting missing/not-reviewed
- [ ] UI badge shows accurate count of flagged regels
- [ ] Threshold values (5%, €50k default) are editable in app settings and reflect in recalculations

---

### REQ-003: Narrative per programme with template prompts

**Requirement:** Enforce structured, consistent explanations for deviations.

**GIVEN** a ProgrammaRegel with drempel_overschreden=true,
**WHEN** a programme-owner assigned to that programme opens the Toelichting form,
**THEN**
- Four markdown text areas appear with pre-filled placeholder labels:
  1. **Oorzaak** (Why did we deviate?) — min 50 chars, max 2000 chars
  2. **Maatregel** (What corrective actions?) — min 50 chars, max 2000 chars
  3. **Risico** (What are the forward-looking risks?) — min 50 chars, max 2000 chars
  4. **Prognose** (Year-end forecast + justification) — min 50 chars, max 2000 chars
- Each field supports markdown (bold, italics, links, lists); WYSIWYG preview available
- On save:
  - All four fields must be non-empty and meet min-char requirement, else form blocks with validation errors
  - Toelichting object created with status_review=in_process, opgesteld_door=current_user, opgesteld_op=now
  - Assigned reviewer is notified: "Toelichting awaiting review: [ProgrammaName]"
- Programme-owner can save as draft and return later; status_review remains in_process

**Acceptance criteria:**
- [ ] Toelichting form renders four labeled markdown fields with placeholders
- [ ] Client-side validation: all fields required, min 50 chars each
- [ ] Markdown syntax support confirmed (bold, italics, lists render correctly)
- [ ] Save creates Toelichting object linked to ProgrammaRegel
- [ ] Reviewer receives notification with link to review task
- [ ] Draft save-and-return workflow does not require re-entering already-filled fields
- [ ] Form UI labels match Dutch terminology: Oorzaak, Maatregel, Risico, Prognose Jaareinde Onderbouwing

---

### REQ-004: Forecast-to-year-end with override

**Requirement:** Enable informed year-end projections using either automatic extrapolation or domain-specific override.

**GIVEN** a ProgrammaRegel with realisatie_ytd up to peildatum,
**WHEN** the programme-owner views the ProgrammaRegel detail,
**THEN**
- `prognose_berekend` is calculated as: realisatie_ytd / (days_elapsed_since_jan1 / 365) * 1
  - Example: Q1 (90 days elapsed): realisatie_ytd=€1M → prognose_berekend=€4M
  - Example: Q2 (181 days): realisatie_ytd=€2.5M → prognose_berekend=€5.05M
- **If** prognose_jaareinde is null (no override entered):
  - UI displays "Calculated year-end forecast: €X.XXM (based on Q1 pace)" in read-only field
  - This prognose_berekend is used in PDF export and trend dashboard
- **If** programme-owner enters prognose_jaareinde (explicit override):
  - prognose_jaareinde field becomes visible + required prognose_onderbouwing (markdown, min 100 chars)
  - Example: "Q1 pace was anomalous; hiring delays resolved in Q2. Expect catch-up to €3.8M by year-end."
  - On save: prognose_jaareinde supersedes prognose_berekend in calculations and exports
- Programme-owner can toggle between auto-calculated and manual override; toggling back to auto clears override

**Acceptance criteria:**
- [ ] prognose_berekend correctly extrapolates realisatie_ytd to 365-day year
- [ ] UI clearly labels auto-calculated vs override prognose
- [ ] prognose_onderbouwing required only when prognose_jaareinde is non-null
- [ ] Override prognose_jaareinde appears in PDF export and trend charts
- [ ] Clearing override reverts UI to display prognose_berekend
- [ ] Audit trail shows who set override and when (via AuditTrailService)

---

### REQ-005: Trigger begrotingswijziging from rapportage

**Requirement:** Enable seamless budget-amendment proposals directly from deviated programmes.

**GIVEN** a ProgrammaRegel with drempel_overschreden=true,
**WHEN** the concerncontroller clicks "Voorstel begrotingswijziging" on that regel,
**THEN**
- A form modal opens pre-populated with:
  - **Programma**: programme_naam (read-only)
  - **Bedrag**: afwijking_absoluut (editable; user can adjust or add multiple tranches)
  - **Motivatie**: auto-filled from linked Toelichting (oorzaak + maatregel summary, editable)
  - **Status**: voorgesteld (locked, user cannot change)
- User can customize motivatie and bedrag; on submit:
  - BegrotingswijzigingTrigger object created with gevoegd_op=now
  - `FinanceqBegrotingswijzigingService.dispatchToDecidesk()` sends webhook to decidesk with { rapportage_slug, programma_slug, bedrag, motivatie }
  - decidesk webhook handler creates Motion in standard template (governance domain TBD)
  - BegrotingswijzigingTrigger.decidesk_motion_id linked back
  - UI displays success: "Begrotingswijziging sent to Motion [ID] in meeting [date]"
- If Toelichting does not exist or is not-reviewed, form still allows submit but shows warning: "Toelichting should be reviewed first"

**Acceptance criteria:**
- [ ] "Voorstel begrotingswijziging" button visible on drempel_overschreden=true regels
- [ ] Form modal pre-fills programma, bedrag (afwijking_absoluut), motivatie from Toelichting
- [ ] bedrag field is editable; user can adjust
- [ ] On submit: BegrotingswijzigingTrigger created with status=voorgesteld
- [ ] Webhook dispatch to decidesk succeeds and returns Motion ID
- [ ] decidesk_motion_id populated in BegrotingswijzigingTrigger
- [ ] Success message includes Motion ID and meeting date
- [ ] Audit trail logs who initiated, timestamp, and final bedrag/motivatie

---

### REQ-006: Two-track review (Marap internal vs Berap political)

**Requirement:** Route rapportages to appropriate reviewers based on type and portefeuille.

**GIVEN** a Rapportage with type ∈ {marap, berap},
**WHEN** the rapportage transitions from status=concept to status=intern (if marap) or status=bestuurlijk (if berap),
**THEN**
- **For Marap (type=marap, status=intern)**:
  - Lifecycle guard `MarapReviewGuard` checks: all drempel_overschreden Toelichtingen have status_review=reviewed_approved
  - If satisfied, rapportage routed to users with role=directie (Directeur Financiën, Secretaris, etc.)
  - Directie receives notification: "Marap Q1-2026 awaiting internal review"
  - Directie can filter/view only Rapportages assigned to them
  - Directie can approve (→ vastgesteld) or request-changes (→ concept)
- **For Berap (type=berap, status=bestuurlijk)**:
  - Lifecycle guard checks same Toelichting prerequisites
  - Rapportage routed to users with role=collegelid (college members)
  - Each collegelid sees only ProgrammaRegels for which they hold portefeuille (via AuthorizationService)
    - Example: Collegelid A (portefeuille=onderwijs+cultuur) sees only ProgrammaRegels for prog-onderwijs, prog-cultuur
    - Collegelid B (portefeuille=sociale-diensten) sees only those programmes
  - Collegelid can approve-with-comments or request-changes; voting/consensus TBD (out of scope for this spec)
  - Once approved by college (consensus/vote), rapportage transitions to vastgesteld
- Both tracks must update Rapportage.vastgesteld_door and vastgesteld_op upon approval

**Acceptance criteria:**
- [ ] Marap type routes to directie reviewers only; Berap to college members
- [ ] Collegelid views filtered by portefeuille (not all regels visible)
- [ ] Transition to intern/bestuurlijk blocked if drempel_overschreden=true Toelichtingen not reviewed
- [ ] Role-based filter works in list views and detail views
- [ ] vastgesteld_door and vastgesteld_op recorded on approval
- [ ] Notifications routed to correct roles per type

---

### REQ-007: Lock and version on vaststelling

**Requirement:** Immutably snapshot rapportage upon formal adoption.

**GIVEN** a Rapportage with status=vastgesteld (transitioned via lifecycle),
**WHEN** the vaststelling is recorded (vastgesteld_op timestamp set),
**THEN**
- All ProgrammaRegel fields are locked: begroting_primitief, begroting_na_wijziging, realisatie_ytd, afwijking_*, drempel_*, prognose_* become immutable
  - Any POST/PUT to update these fields returns 403 Forbidden with message: "Rapportage vastgesteld; no edits allowed"
- All Toelichting fields (oorzaak, maatregel, risico, prognose_onderbouwing) become immutable
- A PDF export is automatically generated:
  - `ExportService.exportRapportageToPdf(rapportage_id)` called
  - PDF includes: rapportage header, ProgrammaRegel table, all Toelichting narratives, audit footer (hash, vastgesteld_door, vastgesteld_op)
  - PDF stored as attachment in FileService linked to Rapportage
- An Excel export is generated similarly with tabular ProgrammaRegels + separate sheet for Toelichtingen
- If later realisatie data is corrected in the grootboek (e.g., erroneous transaction voided):
  - New Rapportage with same peilperiode can be created (e.g., "Q1-2026 corrected")
  - Original vastgestelde rapportage remains unchanged; audit trail shows two versions
  - Trend dashboard shows both versions; user can compare
- Hash verification: SHA256(rapportage_id + peildatum + programmaregel_json + toelichting_json) is stored and validated on read

**Acceptance criteria:**
- [ ] PUT/POST to vastgestelde ProgrammaRegel/Toelichting returns 403 Forbidden
- [ ] PDF export generated and linked to Rapportage as file attachment
- [ ] Excel export generated and linked as file attachment
- [ ] PDF includes all required sections: header, table, narratives, audit footer
- [ ] Realisatie corrections handled via new (non-overwriting) Rapportage version
- [ ] SHA256 hash stored and validated; audit trail references both versions
- [ ] vastgesteld_door/vastgesteld_op recorded and immutable

---

### REQ-008: Comparative dashboard across rapportages

**Requirement:** Visualize multi-quarter financial trends and amendment impacts.

**GIVEN** multiple vastgestelde Rapportages for the same gemeente over consecutive kwartalen (e.g., Q1-2024 through Q4-2026 = 12 rapportages),
**WHEN** a user navigates to the Rapportages → Trend view and selects a programme,
**THEN**
- A line chart is rendered with:
  - X-axis: peilperiode labels (Q1-2024, Q2-2024, ..., Q4-2026)
  - Y-axis: EUR (millions or thousands, user-selectable)
  - Three lines:
    1. **Begroting** (dotted line): begroting_na_wijziging per kwartaal (shows approved budget post-amendments)
    2. **Realisatie** (solid line): realisatie_ytd per kwartaal (cumulative within year; resets to 0 on Jan 1 of new year)
    3. **Prognose** (dashed line): prognose_jaareinde per kwartaal (shows how year-end forecast evolved across quarter-reports)
- Markers on kwartalen where a BegrotingswijzigingTrigger for this programme is aangenomen (green flag icon) or afgewezen (red flag)
- Hover tooltip shows exact figures + linked Toelichting summary (first 200 chars of oorzaak)
- Download button: export chart as PNG + underlying CSV
- Performance: chart renders for 8 kwartalen (2 years) in < 1000ms

**Acceptance criteria:**
- [ ] Chart loads on Trend view with programme filter
- [ ] Begroting, Realisatie, Prognose lines render correctly
- [ ] Y-axis scale is readable for both small deviations (€50k range) and large (€10M+ range)
- [ ] Markers appear on kwartalen with aangenomen/afgewezen BegrotingswijzigingTriggers
- [ ] Hover tooltip displays programme, kwartaal, begroting/realisatie/prognose figures
- [ ] Hover tooltip includes first 200 chars of Toelichting oorzaak
- [ ] CSV export includes all data points and metadata (rapportage_id, peildatum, etc.)
- [ ] Chart responsive to window resize; accessible at 768px+ viewport

---

## User stories — acceptance criteria

### Story 1: Create rapportage (Controller)
**GIVEN** I'm a concerncontroller,
**WHEN** I click "Nieuwe rapportage" on the financeq dashboard,
**THEN**
- [ ] Modal opens with dropdowns for type (berap/marap), peilperiode (Q1-Q4/jaar), year
- [ ] I can confirm and the system loads a rapportage detail page
- [ ] Within 2 seconds, all actieve programmes appear as ProgrammaRegel rows
- [ ] I can see begroting, realisatie, afwijking, and drempel_overschreden flags
- [ ] Programmes flagged as drempel_overschreden are highlighted (color: attention color from NL Design System)

### Story 2: Author toelichting (Programme owner)
**GIVEN** a ProgrammaRegel is flagged drempel_overschreden=true,
**WHEN** I navigate to the Toelichting form,
**THEN**
- [ ] Four markdown text areas appear with prompts
- [ ] I can fill oorzaak, maatregel, risico, prognose_onderbouwing
- [ ] I can toggle preview to see formatted markdown
- [ ] I can save; form is validated (all fields required, min 50 chars)
- [ ] On save, a reviewer is notified
- [ ] I can return later and see my draft text unchanged

### Story 3: Review toelichting (Reviewer)
**GIVEN** a Toelichting is assigned to me for review,
**WHEN** I open the review task,
**THEN**
- [ ] I see the programme-owner's oorzaak/maatregel/risico/prognose_onderbouwing in formatted markdown
- [ ] I can add review feedback in a review_notities field
- [ ] I can click "Approve" or "Request changes"
- [ ] On "Request changes", the programme-owner is notified and can revise
- [ ] On "Approve", the system checks: can the rapportage now advance to next status? (all Toelichtingen reviewed?)

### Story 4: Propose begrotingswijziging (Controller)
**GIVEN** a ProgrammaRegel is flagged drempel_overschreden=true with a linked Toelichting,
**WHEN** I click "Voorstel begrotingswijziging",
**THEN**
- [ ] Form opens pre-filled with programma, bedrag (afwijking_absoluut), motivatie (from Toelichting)
- [ ] I can customize bedrag and motivatie
- [ ] On submit, a Motion is created in decidesk
- [ ] I see success message with Motion ID and meeting date

### Story 5: Review rapportage (Directie/College)
**GIVEN** a Rapportage is assigned to me for review (Marap → Directie, Berap → College),
**WHEN** I navigate to my inbox,
**THEN**
- [ ] I see the rapportage listed with type, peilperiode, number of drempel_overschreden programmes
- [ ] I can open the rapportage detail and see timeline showing concept → intern/bestuurlijk → vastgesteld
- [ ] I can drill into each ProgrammaRegel and see the linked Toelichting
- [ ] (For College) I see only ProgrammaRegels for my portefeuille
- [ ] I can approve (advancing to vastgesteld) or request changes (reverting to concept)
- [ ] On approval to vastgesteld, PDF/Excel exports are generated

### Story 6: Trend analysis (Financial advisor)
**GIVEN** I want to see how a programme's budget performance evolved over 2 years,
**WHEN** I navigate to Trend view and select a programme,
**THEN**
- [ ] Line chart shows Begroting, Realisatie, Prognose lines across 8 kwartalen
- [ ] I can see where budget amendments (green/red flags) landed
- [ ] I can hover for exact figures and linked Toelichting summary
- [ ] I can download the chart as PNG and data as CSV

---

## API endpoints (reference)

- `POST /api/v1/financeq/rapportages` — create rapportage → triggers snapshot job
- `GET /api/v1/financeq/rapportages?type=&peilperiode=&status=` — list rapportages with filters
- `GET /api/v1/financeq/rapportages/{id}` — read rapportage detail + nested ProgrammaRegels
- `PUT /api/v1/financeq/rapportages/{id}/status` — advance rapportage status (via lifecycle)
- `POST /api/v1/financeq/rapportages/{id}/toelichtingen` — create Toelichting
- `PUT /api/v1/financeq/toelichtingen/{id}` — update Toelichting (if not vastgesteld parent)
- `GET /api/v1/financeq/rapportages/{id}/export/pdf` — generate PDF export
- `GET /api/v1/financeq/rapportages/{id}/export/excel` — generate Excel export
- `POST /api/v1/financeq/bw-triggers` — create BegrotingswijzigingTrigger + dispatch webhook
- `GET /api/v1/financeq/trend/{programma_slug}?from=&to=` — fetch trend data for chart

---

## Non-functional requirements

### Performance
- Rapportage creation + snapshot: < 2 seconds for 200+ programmes
- ProgrammaRegel list render: < 500ms
- Trend chart render (8 kwartalen): < 1000ms
- PDF export: < 5 seconds

### Scalability
- Support 500+ municipalities/provinces in single SaaS instance
- Rapportage object growth: 4 per municipality per year (Q1-Q4) × years
- ProgrammaRegel growth: (200 programmes per gemeente) × (4 rapportages per year) = 800k objects/year fleet-wide

### Security
- RBAC: controller, programme-owner, directie, college, griffie roles with appropriate data visibility
- Immutability: vastgestelde rapportages locked via 403 Forbidden on PUT/POST
- Audit trail: all state transitions, edits, and approvals logged via AuditTrailService
- Encryption: Toelichtingen may contain sensitive financial data; ensure TLS in transit and at-rest encryption if configured

### Compliance
- BBV (Besluit Begroting en Verantwoording) — mandatory structure enforced by schema
- IV3 (Informatie voor Derden) — rapportage data exportable in IV3 XML format (future, out of scope)
- SBR (Standard Business Reporting) — jaarrekening linkage (future, out of scope)
- XBRL-NL Decentrale Overheden — machine-readable export (future, out of scope)
- Wet open overheid (Woo) — Berap published as open document with proper metadata (ADR-010 compliance)
- Data retention: docudesk archives per BBV schedules (10 years jaarrekening, 7 years interim)

### Accessibility (WCAG AA)
- Toelichting form labels properly associated with textareas
- Markdown preview toggle keyboard-accessible
- Colour not sole conveyor: drempel_overschreden flagged with icon + colour + text label
- Charts accessible: alt-text on chart image, CSV download provides data in text form
- Responsive: critical flows work at 768px viewport

---

## Success metrics

- **Adoption**: 80% of municipalities using financeq within 6 months of launch
- **Time to rapportage**: Median time from create → vastgesteld reduced from 15 days (Excel) to 7 days (financeq)
- **Narrative quality**: 95% of Toelichtingen flagged as drempel_overschreden receive all four prompts filled (no empty sections)
- **Amendment traceability**: 100% of BegrotingswijzigingTriggers linked to source Rapportage + Toelichting (audit trail complete)
- **Trend adoption**: 60% of financial advisors use Trend dashboard for comparative analysis
- **Zero immutability breaches**: 0 instances of vastgestelde rapportage edits post-vaststelling (enforced by 403)
