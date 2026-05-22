# Proposal: Berap / Marap — Bestuursrapportages

## Vision

Eliminate the Excel-heavy, copy-paste quarterly financial reporting cycle. Enable municipalities and provinces to produce legally-required periodic reports (Bestuursrapportage/Managementrapportage) in a single-source-of-truth workflow bound directly to the live financial data. Automate deviation flagging, enforce structured narrative, and streamline the multi-stage review and approval chain from financial controller → programme owner → directie → College/Gedeputeerde Staten → Raad/Provinciale Staten.

## Target users

- **Concerncontroller / financieel adviseur** — compose and coordinate multi-programme reports; flag material deviations; route for review
- **Programmamanager / budgethouder** — author narrative explanations per programme; forecast year-end; propose budget amendments
- **Directie / secretarissen** — review Marap internal approvals
- **College van B&W / Gedeputeerde Staten** — review Berap political approvals; vaststelling (formal adoption)
- **Griffie + Raad / Provinciale Staten** — read and debate; track budget amendments
- **Accountant** — audit-traceable narrative; use vastgestelde rapportages as interim proof

## Feature set (demand scoring)

### Core features — Rapportage lifecycle

| Feature | Demand | Description |
|---------|--------|-------------|
| **Generate report from live grootboek snapshot** | 5 | GIVEN a peilperiode (Q1/Q2/Q3/Q4), WHEN controller initiates new rapportage, THEN system pre-fills one ProgrammaRegel per actieve programme from live grootboek, pulling begroting_primitief and begroting_na_wijziging from begrotingsregister |
| **Automatic deviation flagging** | 5 | GIVEN |afwijking_percentage\| exceeds gemeenten-specific threshold (default 5% or €50k absolute), WHEN rapportage is opened, THEN ProgrammaRegel marked drempel_overschreden=true and Toelichting becomes verplicht |
| **Structured narrative per programme** | 4 | GIVEN Toelichting is verplicht, WHEN programme-owner opens editor, THEN template prompts load (oorzaak/maatregel/risico/prognose) ensuring consistency and compliance |
| **Year-end forecast with override** | 4 | GIVEN realisatie_ytd, WHEN no prognose entered, THEN lineaire extrapolatie applied; programme-owner can override with explicit cijfer + onderbouwing |
| **Trigger begrotingswijziging from deviation** | 4 | GIVEN drempel_overschreden=true with formele wijziging needed, WHEN controller clicks "voorstel", THEN Begrotingswijziging-trigger created and routed to begrotingswijziging-workflow (decidesk) |
| **Two-track review (Marap vs Berap)** | 5 | GIVEN rapportage type (marap=internal directie, berap=political college/GS), WHEN status transitions, THEN routed to correct reviewers with portefeuille-filtering |
| **Lock and snapshot on vaststelling** | 5 | GIVEN rapportage transitions to vastgesteld, THEN ProgrammaRegel-cijfers and Toelichtingen immutable, PDF/Excel exports generated, later realisatie-correcties do not mutate the snapshot |
| **Comparative trend dashboard** | 3 | GIVEN multiple vastgestelde rapportages over kwartalen, WHEN user opens trend-view, THEN realisatie, prognose-evolutie, cumulatieve afwijking, and begrotingswijziging markers plotted |

### Integration features — Cross-app workflows

| Feature | Demand | Description |
|---------|--------|-------------|
| **Begrotingswijziging integration (decidesk)** | 4 | Begrotingswijziging-triggers routed to decidesk for agendaing on B&W/GS/Raad/PS meetings |
| **Public rapportage publishing (opencatalogi)** | 3 | Vastgestelde Berap published as openbaar document under Wet open overheid (Woo) |
| **Archive and retention (docudesk)** | 4 | Vastgestelde rapportages archived per BBV-bewaarschema (10 jaar jaarrekening, 7 jaar interim) |
| **Grootboek integration (openconnector)** | 5 | Realisatie-input from financial systems (Civision, Key2Financiën, AFAS, Unit4) via openconnector |

## User stories

### Controller: Initiate quarterly rapportage
- **As a** concerncontroller
- **I want to** create a new quarterly rapportage with a single click
- **So that** programme data is pre-populated from the live grootboek and I can focus on deviations
- **Acceptance:** Rapportage created with type=berap/marap, peilperiode=Q1/Q2/Q3/Q4, status=concept; ProgrammaRegel records auto-generated per actieve programme; begroting and realisatie fields populated from live data

### Programme owner: Explain material deviations
- **As a** programmamanager
- **I want to** author a structured narrative explaining why my programme's realisatie diverged from begroting
- **So that** the College understands the underlying causes and can make informed decisions
- **Acceptance:** Toelichting form pre-loads four prompts (oorzaak/maatregel/risico/prognose); markdown support; author can save, review reviewers can approve/request-changes

### Controller: Propose budget amendment from rapportage
- **As a** concerncontroller
- **I want to** propose a begrotingswijziging directly from a flagged programme deviation
- **So that** I don't re-type amounts and motivatie, and the amendment flows directly to the College's agenda
- **Acceptance:** "Voorstel begrotingswijziging" button visible on drempel_overschreden=true rules; clicking creates Begrotingswijziging-trigger pre-filled with programma/bedrag/motivatie; trigger routed to decidesk

### Directie: Review internal management report
- **As a** directeur
- **I want to** review the Marap and approve/reject at status=intern level before it goes to the College
- **So that** we catch issues internally and ensure consistent messaging
- **Acceptance:** Marap rapportages appear in my inbox; I can read ProgrammaRegel summaries and Toelichting narratives; approve/request-changes via workflow

### College: Vote to adopt annual report
- **As a** collegelid
- **I want to** see only the programmes for which I'm portefeuille-holder
- **So that** I don't drown in unrelated deviations
- **Acceptance:** Berap Rapportage-read filtered by my portefeuille; I can drill into programmes under my purview; vaststelling transitions rapportage to immutable

## Stakeholder profiles

| Role | Responsibility | Goal | Pain Point |
|------|-----------------|------|-----------|
| Concerncontroller | Initiate rapportage; flag deviations; coordinate routes | Single-source-of-truth; audit trail | Currently manual Excel aggregation from multiple systems |
| Programmamanager | Explain deviations; forecast; propose measures | Clear narrative; data-backed decisions | Narrative re-entry across Marap/Berap/audit follow-ups |
| Directie (secretarissen) | Review Marap; ensure internal alignment | Timely visibility; consistent quality | Email chains and attachment versioning |
| College van B&W | Review Berap; vote vaststelling; agenda-set begrotingswijzigingen | Legal compliance; transparency to Raad | Long approval cycles; lost amendments |
| Griffie / Raad | Track published Berap and amendments; inform debates | Accessibility; traceable history | PDF archives; missing links to decisions |
| Accountant | Audit rapportages; use as interim proof; validate calculations | Audit trail; immutable snapshots | Manual re-keying; no linkage to source data |

## Standards and regulations

- **BBV (Besluit Begroting en Verantwoording)** — mandatory programme structure and taakvelden
- **IV3 (Informatie voor Derden)** — CBS reporting taxonomy
- **SBR (Standard Business Reporting)** — digital annual-report alignment
- **XBRL-NL Decentrale Overheden** — machine-readable taxonomy
- **Wet open overheid (Woo)** — public document publication

## Success criteria

- [ ] Rapportage creation → ProgrammaRegel population in < 2 seconds
- [ ] Deviation-flagged narrations complete within 2 working days (target SLA)
- [ ] Two-level review (Marap → Berap) without document re-entry
- [ ] Vastgestelde rapportage exports (PDF/Excel) with embedded audit trail and narrative
- [ ] Begrotingswijziging proposals auto-route to decidesk without manual re-entry
- [ ] Trend dashboard renders across 8 kwartalen (2 years) without performance degradation
- [ ] Accountants report zero audit findings re: immutability post-vaststelling
