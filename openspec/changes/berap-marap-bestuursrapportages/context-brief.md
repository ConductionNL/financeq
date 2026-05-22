## Status

Draft — financeq spec brief, 2026-05-21.

# Berap / Marap — Bestuursrapportages

## Purpose

Provide a structured workflow for municipal (gemeente) and provincial (provincie) governments to produce the legally required periodic financial reports — the **Bestuursrapportage (Berap)** and **Managementrapportage (Marap)** — directly from financeq's budget and realisation data. Per quarter, each program (and per product/cost-center beneath it) is compared against the approved begroting (budget). Material deviations (drempelwaarde-overschrijdingen) trigger explanatory narrative, a forward-looking outlook (prognose), and, where required, a formal begrotingswijziging (budget amendment) is initiated. The rapportage is reviewed by the directie (Marap, internal) and the College van B&W / Gedeputeerde Staten (Berap, political), then offered to the Raad / Provinciale Staten. The goal is to replace the typical Excel-heavy, copy-paste reporting cycle with a continuously-current, single-source-of-truth report bound to the live grootboek, with audit-traceable narrative per program and per deviation.

## Data Model

- **Rapportage**: type (berap/marap), peilperiode (Q1/Q2/Q3/Q4/jaar), status (concept/intern/bestuurlijk/vastgesteld), opgesteld_door, vastgesteld_op, vastgesteld_door.
- **ProgrammaRegel**: per program, begroting_primitief, begroting_na_wijziging, realisatie_ytd, prognose_jaareinde, afwijking_absoluut, afwijking_percentage, drempel_overschreden (bool).
- **Toelichting**: program/product, narrative-text (markdown), oorzaak, maatregel, risico, opgesteld_door, reviewed_door.
- **Begrotingswijziging-trigger**: source-rapportage, programma, bedrag, motivatie, status (voorgesteld/aangenomen/afgewezen).
- **VooruitblikSectie**: ontwikkelingen, risico's, kansen, voorgestelde maatregelen.
- **Bijlage**: PDF-export, Excel-export, getekend bestuursbesluit.

## Requirements

**REQ-001: Generate report from live grootboek snapshot.** GIVEN a peilperiode (e.g. Q2-2026), WHEN a financial controller starts a new rapportage, THEN financeq creates a Rapportage object pre-filled with one ProgrammaRegel per actieve programma, populated from the grootboek snapshot at the peildatum, with begroting_primitief and begroting_na_wijziging pulled from the begrotingsregister.

**REQ-002: Automatic deviation flagging at configurable thresholds.** GIVEN a ProgrammaRegel where |afwijking_percentage| exceeds the gemeente-specific drempelwaarde (default 5% or €50k absolute, whichever is higher), WHEN the rapportage is opened, THEN the regel is marked drempel_overschreden=true and a Toelichting becomes verplicht.

**REQ-003: Narrative per program with template prompts.** GIVEN a Toelichting field is verplicht, WHEN the assigned program-owner opens it, THEN the editor pre-loads four structured prompts (oorzaak, maatregel, risico, prognose-onderbouwing) so the narrative is consistent across programs and review-ready.

**REQ-004: Forecast-to-year-end with override.** GIVEN realisatie_ytd, WHEN no manual prognose is entered, THEN financeq calculates prognose_jaareinde via lineaire extrapolatie of the realisatie-pace; AND the program-owner can override with an explicit cijfer + onderbouwing.

**REQ-005: Trigger begrotingswijziging from rapportage.** GIVEN a ProgrammaRegel with drempel_overschreden=true that requires a formele begrotingswijziging, WHEN the controller clicks "voorstel begrotingswijziging", THEN a Begrotingswijziging-trigger is created linked to the rapportage and routed to the begrotingswijziging-workflow without re-typing of bedragen of motivatie.

**REQ-006: Two-track review (Marap internal vs Berap political).** GIVEN a rapportage of type=marap, WHEN it transitions to status=intern, THEN it is routed only to directie-reviewers; GIVEN type=berap, WHEN status=bestuurlijk, THEN it is routed to college/GS members with their portefeuille-programmas pre-filtered.

**REQ-007: Lock and version on vaststelling.** GIVEN a rapportage transitions to status=vastgesteld, WHEN the vaststelling is recorded, THEN all ProgrammaRegel-cijfers and Toelichtingen are gesnapshot (immutable), a PDF + Excel-export is generated, and any later realisatie-correcties in the grootboek do not mutate the vastgestelde rapportage.

**REQ-008: Comparative dashboard across rapportages.** GIVEN multiple vastgestelde rapportages over multiple kwartalen, WHEN a user opens the trend-view per programma, THEN realisatie, prognose-evolutie en cumulatieve afwijking are plotted, with annotaties op kwartalen waarin een begrotingswijziging is doorgevoerd.

## Standards

- **BBV (Besluit Begroting en Verantwoording provincies en gemeenten)** — verplichte programmastructuur en taakvelden.
- **IV3 (Informatie voor Derden)** — CBS-rapportage taxonomie waar realisatiecijfers naar geaggregeerd worden.
- **SBR (Standard Business Reporting)** — voor digitale jaarrekening-aansluiting.
- **XBRL-NL Decentrale Overheden** — taxonomie voor machine-leesbare rapportage.

## Cross-app

- **openregister** — Rapportage/ProgrammaRegel/Toelichting schemas; immutable snapshot via vastgesteld-status.
- **decidesk** — begrotingswijzigings-besluiten worden geagendeerd op B&W / GS / Raad / Provinciale Staten vergaderingen.
- **opencatalogi** — vastgestelde Berap als openbaar document publiceren onder Wet open overheid (Woo).
- **docudesk** — DMS-archivering met BBV-bewaarschema (10 jaar voor jaarrekening, 7 jaar voor tussentijdse rapportages).
- **openconnector** — koppelvlak naar grootboeksystemen (Civision, Key2Financiën, AFAS, Unit4) voor realisatie-input.

## Target users

- **Concerncontroller / financieel adviseur** — opstellen rapportage, signaleren materiële afwijkingen, coördineren toelichtingen.
- **Programmamanager / budgethouder** — invullen toelichting per programma, onderbouwen prognose, voorstellen maatregelen.
- **Directie / gemeentesecretaris / provinciesecretaris** — Marap-review intern.
- **College van B&W / Gedeputeerde Staten** — Berap-review bestuurlijk, vaststelling, aanbieding aan Raad/PS.
- **Griffie + raadsleden / statenleden** — kaderstellend lezen, begrotingswijzigingen behandelen.
- **Accountant** — interim- en jaarrekeningcontrole; gebruikt vastgestelde rapportages als interim-bewijslast.
