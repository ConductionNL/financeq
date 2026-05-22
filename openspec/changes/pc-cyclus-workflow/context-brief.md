---
status: draft
---
# P&C-Cyclus Workflow (Planning & Control)

## Purpose

The Planning & Control cyclus (P&C-cyclus) is the statutory annual rhythm by which every Dutch decentrale overheid — gemeenten, provincies, waterschappen, and gemeenschappelijke regelingen — plans, authorises, executes, monitors, and accounts for its finances. The cycle is anchored in the Gemeentewet (art. 186-213), the Provinciewet (art. 190-217), the Waterschapswet (art. 98-109a) and operationalised by the Besluit Begroting en Verantwoording provincies en gemeenten (BBV) and, for waterschappen, the Waterschapsbesluit. It produces a chain of documents — kadernota, voorjaarsnota, programmabegroting, bestuursrapportages (BERAP/MARAP), najaarsnota, and jaarrekening — each with its own statutory deadlines, mandatory deelnemers, and binding handoffs to college, raad, accountant, and toezichthouder (provincie or BZK).

In practice every organisation reinvents this cycle in Word, Excel, SharePoint, and the occasional purpose-built tool (Pepperflow, Lias, iBabs Begroting). The workflow lives in email and project plans rather than the financial system; document versions drift between college-versie, raads-versie, and accountants-versie; portefeuillehouder sign-off is captured in margin comments; statutory deadlines (15 juli kadernota, 15 november begroting bij toezichthouder, 15 juli jaarrekening) are tracked in personal agendas. When a deadline slips or a version conflict surfaces, the organisation has no audit trail of who agreed to what, when.

`pc-cyclus-workflow` makes the P&C-cyclus a first-class workflow inside financeq. The spec defines a process engine where each cycle stage is a typed node with explicit triggers, deelnemers, deliverables, validations, and handoff rules; where every deliverable is a versioned document linked to the underlying begroting and werkelijke uitgaven; where portefeuillehouder sign-off produces a cryptographic signature on a specific document version; and where statutory deadlines drive automated reminders and escalations. The result is a single source of truth for the financial year, queryable by date, stage, portefeuille, programma, or accountable officer, that satisfies both internal control (verbijzonderde interne controle, art. 213a Gemeentewet) and external accountability (rechtmatigheidsverantwoording from boekjaar 2023 onwards).

The spec is deliberately narrow in scope. It owns the *workflow* — stages, transitions, deelnemers, sign-off, deadlines, versions — not the *content* of begroting or jaarrekening, which lives in `bookkeeping-bbv-compliance` and `programma-begroting`. It does not replace decidesk (raadsbesluit chain) or docudesk (long-term archival); it integrates with both. It does not implement OAB (overzicht algemene baten en lasten) or paragrafen content; those are produced by adjacent specs and merely referenced as deliverables here.

## Data Model

The spec introduces eight registers, each a separate JSON Schema under `financeq/openspec/specs/pc-cyclus-workflow/schemas/`:

**CyclusInstance** — one record per organisation-year (e.g. "Gemeente Zeist 2027"). Fields: `boekjaar`, `organisationRef` (BAG/CBS code), `status` (planning|actief|afgerond|heropend), `kickoffDate`, `closureDate`, `templateRef` (which CyclusTemplate was used to instantiate this year), `customisations` (delta from template), `currentStageRefs[]` (stages currently in flight — can be more than one, e.g. BERAP-Q1 while voorjaarsnota is in raad-behandeling).

**CyclusTemplate** — reusable definition of stages, transitions, and roles for a sector (gemeente, provincie, waterschap) or a custom variant. Ships with three default templates derived from BBV and Waterschapsbesluit. Versioned; templates can be forked per organisation.

**Stage** — one record per stage occurrence within a CyclusInstance. Fields: `stageType` (kadernota|voorjaarsnota|programmabegroting|berap|najaarsnota|jaarrekening|burap (waterschap)|kwartaalrapportage), `sequenceNumber` (BERAP-1, BERAP-2…), `plannedStart`, `plannedEnd`, `statutoryDeadline` (auto-derived from stageType + boekjaar), `actualStart`, `actualEnd`, `status` (niet-gestart|in-voorbereiding|in-behandeling|in-besluitvorming|afgerond|overschreden), `responsibleAfdeling`, `portefeuillehouderRef`, `accountableOfficerRef` (concerncontroller, gemeentesecretaris, etc.).

**StageTransition** — directed edge between stages. Captures the handoff: source Stage → target Stage, `transitionType` (sequential|parallel|conditional), `gateCriteria` (list of validations that must pass), `requiredSignoffs[]` (list of role references), `actualTransitionDate`, `actualTransitionBy` (user reference).

**Deliverable** — typed document produced within a stage. Fields: `deliverableType` (kadernota-document|begrotingsboek|berap-rapport|paragraaf-X|jaarrekening|controleverklaring|raadsbesluit-concept), `currentVersionRef`, `versionHistory[]` (chain of DeliverableVersion refs), `linkedFinancials` (refs to begroting-regels and grootboek-mutaties), `state` (concept|college-versie|raads-versie|definitief|gearchiveerd).

**DeliverableVersion** — immutable snapshot. Fields: `versionNumber` (semver-style: 0.1.0 concept → 1.0.0 college-akkoord → 2.0.0 raads-vastgesteld), `createdAt`, `createdBy`, `contentHash` (sha256 of canonical serialisation), `changeNote`, `parentVersionRef`, `attachments[]` (PDF, XLSX, supporting docs).

**Signoff** — cryptographic acknowledgement of a DeliverableVersion by a named role-holder. Fields: `signedVersionRef`, `signerRef`, `signerRole` (portefeuillehouder|wethouder|gedeputeerde|dijkgraaf|concerncontroller|gemeentesecretaris), `signedAt`, `signatureMethod` (digid-sign|qualified-eidas|ad-hoc-otp|wet-handtekening), `signatureBlob`, `revokedAt` (nullable), `revokedReason`.

**Deadline** — derived view; one record per Stage holding statutory and internal deadlines. Fields: `deadlineType` (statutoir|intern|college-aanlevering|raads-aanlevering|toezichthouder-aanlevering|cbs-aanlevering), `dueDate`, `legalBasis` (BBV-art-X / Gemw-art-Y), `escalationPolicy` (list of role refs + lead times: T-30, T-14, T-7, T-0, T+7), `currentStatus` (ok|at-risk|overdue|waived).

All registers reuse `core-financial-objects` for monetary amounts (Money type with currency + scale) and `core-organisation` for organisation and afdeling references. Deelnemers reference Nextcloud users via the standard `userRef` pattern from `core-identity`.

## Requirements

### REQ-001: Statutory deadline derivation

The system SHALL derive statutory deadlines automatically from stageType and boekjaar, using a sector-specific deadline calendar (gemeente, provincie, waterschap), and SHALL prevent users from setting a `plannedEnd` later than the statutory deadline without an explicit acknowledged override.

- GIVEN a CyclusInstance for Gemeente Zeist boekjaar 2027 WHEN a Stage of type `programmabegroting` is instantiated THEN the system sets `statutoryDeadline` to 2026-11-15 (BBV art. 191 lid 2 — 15 november van het jaar voorafgaand) and exposes `legalBasis = "Gemw art. 191 lid 2"`.
- GIVEN a Stage of type `jaarrekening` for boekjaar 2026 WHEN a planner sets `plannedEnd` to 2027-07-31 THEN the system rejects the change with error `STATUTORY_DEADLINE_EXCEEDED` referencing Gemw art. 200 lid 2 (15 juli) unless an `acknowledgedOverride` with motivation and approver is supplied.
- GIVEN a Waterschap CyclusInstance WHEN a Stage of type `programmabegroting` is created THEN the system derives `statutoryDeadline` from Waterschapsbesluit (15 november aan provincie) rather than BBV.

### REQ-002: Stage transition gating

The system SHALL enforce that a StageTransition can only be marked `completed` when every `gateCriteria` validation evaluates true and every `requiredSignoffs` entry has a non-revoked Signoff against the deliverable's current version.

- GIVEN a Stage `voorjaarsnota` with gateCriteria `[financiele-saldi-sluitend, paragraaf-weerstandsvermogen-aanwezig]` and requiredSignoffs `[portefeuillehouder-financien, concerncontroller]` WHEN a user attempts to transition to `programmabegroting` and only one signoff is present THEN the system rejects with `TRANSITION_GATE_FAILED` listing the missing signoff and any failing validations.
- GIVEN a transition where every gate passes WHEN a user with role `gemeentesecretaris` triggers the transition THEN the system creates a StageTransition record with `actualTransitionDate=now`, sets source.status=afgerond and target.status=in-voorbereiding atomically.
- GIVEN a Signoff that is later revoked (revokedAt set) WHEN the system re-evaluates a previously completed transition THEN it flags the downstream stages with `upstreamSignoffRevoked` warning but does NOT auto-rollback.

### REQ-003: Document versioning with content hashing

Every DeliverableVersion SHALL be immutable once created and SHALL carry a sha256 contentHash of its canonical serialisation; the system SHALL reject any attempt to mutate an existing version and SHALL require a new version for any change.

- GIVEN a DeliverableVersion 1.2.0 of begrotingsboek WHEN any field of that record is edited via API or UI THEN the system rejects with `VERSION_IMMUTABLE` and prompts the caller to create a successor version.
- GIVEN a new DeliverableVersion is being created WHEN the system computes contentHash from a canonical JSON serialisation (sorted keys, normalised whitespace) THEN if the resulting hash matches the parent's hash the system rejects with `NO_EFFECTIVE_CHANGE` to prevent version inflation.
- GIVEN a Deliverable with versions 0.1.0 → 0.2.0 → 1.0.0 (college-akkoord) WHEN a user attempts to create version 0.3.0 (lower than current) THEN the system rejects with `VERSION_NOT_MONOTONIC`.

### REQ-004: Portefeuillehouder sign-off

The system SHALL support cryptographic sign-off by named role-holders against a specific DeliverableVersion using DigiD-Sign, qualified eIDAS, or an ad-hoc OTP method; the signature SHALL bind to the contentHash of that exact version and SHALL be independently verifiable.

- GIVEN a portefeuillehouder receives a sign-off request for begrotingsboek v1.0.0 (hash `abc123…`) WHEN they sign via DigiD-Sign THEN the system creates a Signoff record with `signatureMethod=digid-sign`, `signatureBlob` containing the PKCS#7 envelope, and `signedVersionRef` pinned to v1.0.0.
- GIVEN a Signoff exists for begrotingsboek v1.0.0 WHEN the deliverable progresses to v1.1.0 (typo fix) THEN the existing Signoff stays attached to v1.0.0 and the new version starts unsigned; the system surfaces a `signoff-stale` indicator on the workflow board.
- GIVEN any Signoff record WHEN an external party calls `GET /api/signoffs/{id}/verify` THEN the system returns the signed contentHash, the recomputed hash of the linked DeliverableVersion, and a boolean `valid` flag.

### REQ-005: Linkage to begroting and werkelijke uitgaven

Every Deliverable that contains financial figures SHALL maintain explicit refs to the begroting-regels and grootboek-mutaties from which those figures are derived; the system SHALL recompute the totals on demand and SHALL warn when the deliverable's displayed totals diverge from the live source totals by more than a configurable tolerance (default 0.01 EUR).

- GIVEN a BERAP deliverable v0.3.0 referencing 412 begroting-regels and 2,847 grootboek-mutaties WHEN a user requests recompute THEN the system returns per-programma actuals and the delta against the deliverable's stored totals.
- GIVEN a deliverable with a stored programma-total of EUR 12,450,000 and a live recompute of EUR 12,450,123.45 WHEN delta exceeds tolerance THEN the system marks the deliverable with `financials-out-of-sync` and blocks transition to `definitief` until reconciled or explicitly waived.
- GIVEN a Deliverable in state `definitief` (raads-vastgesteld) WHEN underlying grootboek-mutaties change THEN the deliverable's stored totals do NOT change (raads-besluit is frozen) but the system creates a discrepancy record visible in the next BERAP.

### REQ-006: Deelnemer assignment and role-based visibility

The system SHALL allow assignment of named deelnemers (users) to each Stage with explicit roles (trekker, mede-trekker, reviewer, signoff-holder, viewer) and SHALL restrict visibility of unfinalised deliverables to assigned deelnemers + organisation-wide finance roles.

- GIVEN a Stage with deelnemers `[alice:trekker, bob:reviewer, claire:portefeuillehouder]` WHEN user `dave` (no role on this stage, no organisation-wide finance role) calls `GET /api/stages/{id}` THEN the system returns 403 with reason `not-a-deelnemer`.
- GIVEN a Deliverable in state `concept` WHEN a user with `viewer` role on the parent Stage requests it THEN the system returns the deliverable with a `state=concept` warning banner.
- GIVEN a Deliverable in state `definitief` (raads-vastgesteld) WHEN any authenticated organisation user requests it THEN the system returns it without role check (rechtmatigheid-publicatieplicht).

### REQ-007: Escalation policy execution

The system SHALL evaluate every active Deadline daily and SHALL emit notifications to the configured escalation chain (T-30, T-14, T-7, T-0, T+7 by default) routing first to the responsible afdeling, then portefeuillehouder, then gemeentesecretaris/algemeen directeur.

- GIVEN a Stage with `statutoryDeadline = 2027-07-15` and default escalation policy WHEN the daily evaluator runs on 2027-06-15 THEN the system creates a T-30 notification routed to the responsible afdeling.
- GIVEN a Stage that is `overdue` (T+7) WHEN the evaluator runs THEN the system escalates to gemeentesecretaris and creates a `deadline-breached` event consumed by the rechtmatigheid-rapportage.
- GIVEN a Deadline with `currentStatus = waived` WHEN the evaluator runs THEN no notifications are emitted and the deadline does not contribute to rechtmatigheid-rapportage breaches.

### REQ-008: Sector-specific cyclus templates

The system SHALL ship three reference CyclusTemplates (gemeente, provincie, waterschap) reflecting current BBV/Waterschapsbesluit obligations and SHALL allow organisations to fork and customise these templates while retaining traceability to the source template version.

- GIVEN the system ships `cyclus-template-gemeente@2026.1` with stages kadernota → voorjaarsnota → programmabegroting → BERAP-Q1 → BERAP-Q2 → najaarsnota → BERAP-Q3 → jaarrekening WHEN an organisation forks it THEN the fork stores `parentTemplateRef=cyclus-template-gemeente@2026.1` and a structured diff.
- GIVEN a forked template with custom stage `concernberaad` inserted between voorjaarsnota and programmabegroting WHEN the parent template publishes version 2027.1 with a new statutory deadline THEN the system surfaces a `parent-template-updated` notification with a diff against the fork.
- GIVEN a waterschap CyclusTemplate WHEN a stage `burap` (bestuursrapportage) is added THEN the system accepts stage types that are absent from the gemeente template, validated against the waterschap stageType enum.

### REQ-009: Heropening of closed cycle

The system SHALL support `heropening` of a closed CyclusInstance (e.g. for material corrections discovered post-jaarrekening) with full audit trail and automatic creation of a `heropende-jaarrekening` deliverable that supersedes but does not destroy the original.

- GIVEN a CyclusInstance for boekjaar 2025 in state `afgerond` WHEN a user with role `concerncontroller` initiates heropening with motivation "materiële fout subsidieverantwoording" THEN the system sets status=heropend, creates a new Stage `heropening-jaarrekening`, and notifies the toezichthouder.
- GIVEN a heropende cycle is itself closed WHEN the resulting `heropende-jaarrekening v1.0.0` is signed off THEN the system marks the original `jaarrekening v1.0.0` with `superseded-by` reference but retains it as immutable history.
- GIVEN any heropening event WHEN the toezichthouder calls the audit API THEN the system returns the full chronology: original closure, heropening trigger, motivation, signer, new deliverables.

### REQ-010: Public publication of vastgestelde stukken

The system SHALL automatically publish every Deliverable that transitions to state `definitief` to the organisation's public-facing channel (default: Open Overheid metadata feed at `/api/public/p-en-c/{boekjaar}`) within 14 days, per Wet open overheid (Woo) art. 3.3 lid 2 sub c.

- GIVEN a Deliverable transitioning to `definitief` on 2027-11-10 WHEN the publication scheduler runs THEN the system publishes it to the Woo feed no later than 2027-11-24 with metadata fields: `informationCategory=p-en-c`, `documentType=programmabegroting`, `boekjaar=2027`.
- GIVEN a Deliverable that is `definitief` but flagged `bevat-bedrijfsgevoelige-info` WHEN the scheduler runs THEN the system publishes only the metadata stub with redactions and records a Woo art. 5.1 exception with reason.
- GIVEN a published Deliverable WHEN it is later superseded by a heropende version THEN the public feed is updated with the new version and the original remains accessible at its versioned URL (no link rot).

## Standards & Sources

- **BBV** — Besluit Begroting en Verantwoording provincies en gemeenten (commissie-BBV, current version with all notitia 2014-2026).
- **Gemw art. 186-213** — Gemeentewet, financiële bepalingen including statutory deadlines.
- **Provinciewet art. 190-217** — equivalent for provincies.
- **Waterschapswet art. 98-109a** + **Waterschapsbesluit** — equivalent for waterschappen, with separate stage taxonomy.
- **Wet Fido** + **Ruddo** — treasury rules referenced by paragraaf financiering.
- **Rechtmatigheidsverantwoording** — boekjaar 2023 onwards, college signs off on rechtmatigheid of all financial transactions.
- **Wet open overheid (Woo)** — art. 3.3 lid 2 sub c mandates active publication of begroting and verantwoording within 14 days of vaststelling.
- **NBA-handreiking 1108 / 1141** — accountantsprotocol jaarrekening decentrale overheden.
- **VNG Modelverordening 212/213/213a** — financial regulation, control regulation, internal audit regulation.
- **Existing tools** — Pepperflow (Vellekoop), Lias (Lias Informatiemanagement), iBabs Begroting (BCT), Cognos Planning. The spec is interoperable but does not depend on any.
- **Reference implementations** — VNG Realisatie Common Ground patterns, BZK Open Overheid metadata schema.

## Cross-app integration

- **decidesk** (raadsbesluit chain) — every Deliverable transitioning to state `raads-versie` triggers creation of a decidesk raadsstuk; raadsbesluit outcome (aangenomen/verworpen/aangehouden) is consumed back as the trigger for `definitief` state. The integration uses decidesk's `raadsstuk-event` webhook (ADR aligned with decidesk-bridge-spec).
- **docudesk** (long-term archival) — every Deliverable transitioning to `definitief` is mirrored to docudesk with retention policy `archiefwet-7-jaar` for ondersteunende stukken and `archiefwet-permanent` for vastgestelde stukken; docudesk returns an OAIS-compliant AIP identifier stored in the Deliverable.
- **bookkeeping-bbv-compliance** (financeq) — supplies the begroting-regels and grootboek-mutaties referenced by REQ-005; defines the BBV taakveld and economische categorie taxonomies.
- **iv3-aanlevering-cbs** (financeq) — consumes the jaarrekening Deliverable to generate the Iv3-Y aanlevering; subscribes to `deliverable.published` events.
- **openconnector** — outbound integrations to toezichthouder portals (provincie financieel toezicht, BZK ARTHURUS), CBS-Kredo, Pepperflow imports, iBabs councillor agendas. All credentials and rate-limits live in openconnector source records, not in financeq.
- **openregister** — every register defined in this spec is an OR schema; auditing, versioning of records (separate from DeliverableVersion which is a domain concept), search, and access control are inherited.
- **n8n** — escalation notifications (REQ-007) and Woo-publication scheduler (REQ-010) are n8n workflows triggered by financeq events; the workflows ship as part of this spec's `seed-data/n8n-flows/` directory.
- **mydash** — exposes a default dashboard "P&C-Cyclus 2027" with deadline-risk traffic light per stage; consumes the same registers via GraphQL.
- **docusaurus journeydoc** — every CyclusTemplate ships with a user-facing journey-doc (kadernota-traject, begrotings-traject, jaarrekening-traject) so politicians and ambtenaren can self-serve "wat komt er nu op mij af".

## Target users

- **Concerncontroller / hoofd Financiën** — owns the cycle, signs off on transitions, escalates breached deadlines. Primary daily user.
- **Afdelings-controllers** — produce programma-specific deliverable sections, request peer review, route to portefeuillehouder.
- **Portefeuillehouders (wethouders, gedeputeerden, dagelijks bestuur waterschap)** — review and sign off on deliverables within their portefeuille; consume the workflow board to see "wat ligt er op mijn bord".
- **Gemeentesecretaris / algemeen directeur / dijkgraaf** — final escalation, gatekeeper of college-versie transitions.
- **College / dagelijks bestuur** — collective signoff on college-versies; consumes the dashboard.
- **Raad / provinciale staten / algemeen bestuur** — consume `definitief` deliverables via the public Woo feed and the politician-facing journey-doc.
- **Accountant** — read-only access to the full version history of jaarrekening + supporting deliverables via the audit API; uses the contentHash chain to verify nothing changed between accountantscontrole and vaststelling.
- **Toezichthouder (provincie financieel toezicht, BZK)** — read-only access to programmabegroting and jaarrekening with the public audit endpoint; receives `deadline-breached` notifications.
- **Burger / journalist / open-data consument** — consumes published deliverables via the Woo feed.
- **VNG/IPO/UvW** — sector-wide consumption of anonymised cycle-health telemetry (opt-in) for benchmarking.
