---
status: draft
---
# Iv3-Aanlevering CBS (Informatie voor Derden)

## Purpose

Iv3 — Informatie voor derden — is the statutory quarterly financial reporting that every Dutch decentrale overheid (gemeenten, provincies, waterschappen, gemeenschappelijke regelingen) must deliver to the Centraal Bureau voor de Statistiek. The duty is anchored in the Wet financiering decentrale overheden (Wet Fido) art. 3 lid 4 and the Regeling informatie voor derden (RIV) and operationalised by the Iv3-richtlijnen issued jointly by CBS, ministerie van BZK, ministerie van Financiën, en commissie BBV. The data feeds into CBS-statistieken (statline), the EMU-saldi reports to the European Commission (ESA 2010 / EDP — excessive deficit procedure), and the macro-economic Houdbaarheidsraming of the CPB. Late or non-conforming aanleveringen trigger correspondence from BZK financieel toezicht and, in extremis, contribute to artikel 12 status escalations.

The aanlevering itself is technically demanding. The XML payload follows the EDA-XML (Electronic Data Adjustment) format, with strict schema validation, totals and sub-totals that must cross-foot exactly, and reference data (taakvelden, economische categorieën, gemeentecodes) that updates yearly. Each kwartaal-aanlevering ventilates four amounts per intersection of programma × taakveld × economische categorie × balanspost: baten, lasten, mutaties reserves, and saldi. Yearly Iv3-Y aanleveringen additionally reconcile to the vastgestelde jaarrekening. The mapping from an organisation's own grootboek-rekeningen to the standardised Iv3-taakvelden is the operational pain point: most organisations maintain a tussenrekening / categorieverdeling spreadsheet that drifts from quarter to quarter, often hand-edited the week before the deadline.

`iv3-aanlevering-cbs` makes the entire Iv3 lifecycle reproducible, validatable, and audit-traceable. It owns: the canonical taxonomy of Iv3-taakvelden and economische categorieën (versioned per jaar); the mapping registers (grootboek → tussenrekening → Iv3-element); the aggregation engine that produces the per-aanlevering ventilation; the EDA-XML serialisation conformant to the current Iv3-schema; the validatie-suite that pre-runs CBS-Kredo's checks locally so deadline-day surprises are rare; the submission record that captures the exact payload sent, the CBS-Kredo response, and any correction-aanleveringen.

The spec does not own the underlying grootboek (that is `bookkeeping-bbv-compliance`) or the budget/forecast figures (that is `bookkeeping-budget-forecast`). It does not perform the macro-aggregation that produces the EMU-saldo at country level (that is CBS). It does not deduplicate gemeenschappelijke regelingen — that responsibility lies with the individual GR.

## Data Model

The spec introduces seven registers under `financeq/openspec/specs/iv3-aanlevering-cbs/schemas/`:

**Iv3Taxonomy** — versioned reference data set published yearly. Fields: `boekjaar`, `taxonomyVersion` (CBS publication identifier, e.g. `iv3-2027.1`), `taakvelden[]` (list of Taakveld records: code 0.1-8.3, naam, hoofdfunctie, beschrijving, validFrom, validTo), `economischeCategorieen[]` (code 1.x-9.x, naam, debet/credit-aard, hierarchical parent), `balansposten[]` (vaste activa, vlottende activa, eigen vermogen, voorzieningen, vaste schulden, vlottende schulden, with codes), `mutatieKenmerken[]` (raming-bijgesteld, werkelijk, raming-oorspronkelijk), `gemeenteCodes[]` (CBS-code, naam, validFrom, validTo — captures herindelingen). Loaded annually from CBS publications via openconnector source.

**GrootboekMapping** — per organisation, per boekjaar: how each grootboek-rekening maps to (taakveld, economische categorie, balanspost). Fields: `grootboekRekeningRef`, `boekjaar`, `taakveldCode`, `economischeCategorieCode`, `balanspostCode` (nullable for resultaat-rekeningen), `splitMethod` (full|percentage|driver), `splitDetails` (when percentage: list of (target, pct); when driver: ref to driver-formula), `effectiveFrom`, `effectiveTo`, `mappingRationale` (free text), `mappedBy`, `reviewedBy`, `reviewedAt`. Audit-versioned.

**Tussenrekening** — optional intermediate aggregation layer used by organisations that split grootboek-rekeningen across multiple taakvelden via cost drivers (e.g. salariskosten verdeeld over taakvelden op basis van urenregistratie). Fields: `code`, `naam`, `aggregationFormula` (DSL expression referencing grootboek-rekeningen, drivers, or other tussenrekeningen), `outputTargets[]` (list of (taakveld, economische categorie, percentage|driverRef)).

**Iv3Aanlevering** — one record per (boekjaar, kwartaal-of-jaar, organisatie). Fields: `boekjaar`, `periode` (Q1|Q2|Q3|Q4|Y|correctie-{n}), `organisationRef`, `cbsCode`, `status` (in-voorbereiding|gevalideerd|ingediend|geaccepteerd|teruggewezen|gecorrigeerd), `payloadVersionRef`, `submittedAt` (nullable), `cbsResponseRef` (nullable), `taxonomyVersionUsed`, `mappingSnapshotRef` (frozen view of GrootboekMapping as it stood at submission).

**Iv3Payload** — immutable serialised aanlevering. Fields: `aanleveringRef`, `versionNumber`, `format` (eda-xml-v3.5|eda-xml-v4.0 depending on CBS-published schema), `xmlBlob` (large), `contentHash`, `validationReport` (list of validation results), `createdAt`, `createdBy`.

**Iv3SubmissionResponse** — CBS-Kredo response capture. Fields: `payloadRef`, `submissionId` (CBS-issued), `acknowledgedAt`, `responseCode` (accepted|rejected-validation|rejected-format|partial), `responseDetails`, `validationErrors[]` (with codes per CBS error-catalogue).

**Iv3ReconciliationCheck** — derived register, one row per (aanlevering, checkType). Fields: `aanleveringRef`, `checkType` (balanstotaal-sluitend|baten-lasten-saldi-consistent|deelnemingen-apart|reserves-mutaties-consistent|y-aanlevering-matches-vastgestelde-jaarrekening|quartaal-mutaties-cumulatief|gemeentecode-actueel|taxonomy-current-jaar), `result` (pass|warn|fail), `details`, `tolerance`, `runAt`.

All monetary amounts use `core-financial-objects.Money`. Tijds-references use ISO 8601 with kwartaal helpers. Validation results conform to a shared `core-validation.Result` schema for fleet-wide diagnostics.

## Requirements

### REQ-001: Taxonomy ingestion per boekjaar

The system SHALL ingest the CBS-published Iv3-taxonomie at least annually and SHALL refuse to produce aanleveringen for a boekjaar where no matching taxonomy is loaded.

- GIVEN no Iv3Taxonomy exists for boekjaar 2027 WHEN a user attempts to generate Q1-2027 aanlevering THEN the system rejects with error `TAXONOMY_MISSING` and a remediation link to the openconnector source `cbs-iv3-taxonomy`.
- GIVEN Iv3Taxonomy 2027.1 is loaded and CBS publishes 2027.2 mid-year WHEN the openconnector sync runs THEN the system loads 2027.2 as a new version, retains 2027.1, and surfaces a `taxonomy-version-bump` notification with a diff (added/removed taakvelden, renamed labels).
- GIVEN a herindeling waarbij gemeente X opgaat in gemeente Y per 2027-01-01 WHEN taxonomy 2027.1 is loaded THEN the system records X with validTo=2026-12-31 and Y with validFrom=2027-01-01; aanleveringen for boekjaar 2026 still use X's cbsCode.

### REQ-002: Grootboek-to-Iv3 mapping with split methods

The system SHALL allow each grootboek-rekening to be mapped to one or more (taakveld, economische categorie) pairs using one of three split methods: full (100% to one target), percentage (fixed splits summing to 100%), or driver (formula evaluated per period).

- GIVEN a salariskosten grootboek-rekening 411000 with splitMethod=driver referencing the urenregistratie-driver WHEN the aanlevering aggregator runs for Q1-2027 THEN the system evaluates the driver against actual Q1 hours and produces the per-taakveld split, recording the inputs in `splitEvaluationLog`.
- GIVEN a grootboek-rekening with splitMethod=percentage and a list summing to 99.5% WHEN a user saves the mapping THEN the system rejects with `SPLIT_TOTAL_NOT_100` and surfaces the actual sum.
- GIVEN a grootboek-rekening for boekjaar 2027 with effectiveFrom=2027-04-01 and no mapping for 2027-01-01 to 2027-03-31 WHEN Q1-2027 aggregation runs THEN the system fails the GROOTBOEK_UNMAPPED check and surfaces the gap-period explicitly.

### REQ-003: EDA-XML serialisation conformant to CBS schema

The system SHALL serialise each Iv3Aanlevering to EDA-XML using the exact schema version mandated by CBS for the boekjaar and SHALL validate the XML locally against the published XSD before allowing submission.

- GIVEN an Iv3Aanlevering for boekjaar 2027 WHEN the user triggers `generate-xml` THEN the system uses `eda-xml-v4.0` (the CBS-2027 schema) and produces an XML document that validates clean against the bundled XSD.
- GIVEN a generated XML payload that fails local XSD validation WHEN the user attempts to submit THEN the system blocks submission with error `XSD_VALIDATION_FAILED` and a structured list of XPath + violation.
- GIVEN a successfully generated payload WHEN any subsequent change occurs to the underlying GrootboekMapping or grootboek-mutaties THEN the existing Iv3Payload remains unchanged (immutable) and the system surfaces a `payload-stale` indicator on the aanlevering.

### REQ-004: Local pre-run of CBS-Kredo validations

The system SHALL execute the full CBS-Kredo validation suite locally before any submission and SHALL block submission unless every blocker-class validation passes.

- GIVEN an aanlevering with a balanstotaal mismatch of EUR 1.23 WHEN the local validator runs THEN it returns `BAL_001_BALANSTOTAAL_NIET_SLUITEND` (blocker) and submission is blocked.
- GIVEN an aanlevering where deelnemingen (taakveld 0.5) appear bundled with overige financiele baten WHEN the local validator runs THEN it returns `STR_014_DEELNEMINGEN_APART_VEREIST` (blocker) referencing the offending grootboek-mutaties.
- GIVEN an aanlevering with a warn-class issue (e.g. opbrengsten/uitgaven onbeklemtoond) WHEN the user submits anyway THEN the system records the warning in `acknowledgedWarnings[]` and proceeds.

### REQ-005: Balanstotaal en saldi cross-foot checks

The system SHALL enforce that, for every quarterly aanlevering, baten minus lasten plus mutaties reserves equals the period saldo and that the balanstotaal of activa equals the balanstotaal of passiva to the cent.

- GIVEN aggregated baten EUR 184,592,103.22, lasten EUR 178,401,876.41, mutaties reserves EUR -3,190,226.81 WHEN the cross-foot check runs THEN it asserts saldo = EUR 3,000,000.00 to the cent.
- GIVEN balanstotaal activa EUR 412,778,901.05 and passiva EUR 412,778,901.50 WHEN the check runs THEN it fails with delta EUR 0.45 reported and links to the contributing balansposten.
- GIVEN a Y-aanlevering WHEN reconciliation against the vastgestelde jaarrekening from `pc-cyclus-workflow` runs THEN it asserts every taakveld total matches the jaarrekening within a EUR 0.01 tolerance per regel and EUR 1.00 cumulatief per programma.

### REQ-006: Quartaal-mutaties cumulatief

The system SHALL enforce that Q2 figures include Q1 cumulatief, Q3 includes Q1+Q2, and Q4 + Y reconcile; deviations SHALL fail the `quartaal-mutaties-cumulatief` check.

- GIVEN Q1-2027 reports cumulatieve lasten taakveld 4.1 = EUR 5,200,000 and Q2-2027 reports cumulatieve lasten taakveld 4.1 = EUR 9,800,000 WHEN Q3-2027 is generated THEN the system asserts Q3-cumulatief >= EUR 9,800,000 minus any explicit reversal.
- GIVEN a Y-aanlevering with totaal taakveld 4.1 = EUR 20,100,000 and Q4-aanlevering with cumulatief EUR 20,050,000 WHEN the reconciliation runs THEN the system warns with delta EUR 50,000 and requires a `correctie-aanlevering` to be either generated or explicitly waived.
- GIVEN a correctie-aanlevering against Q2-2027 WHEN it is submitted THEN the system treats it as superseding the original Q2 for the cumulatief chain and recomputes downstream consistency.

### REQ-007: Submission to CBS-Kredo with response capture

The system SHALL submit Iv3Payloads to the CBS-Kredo portaal via the official submission API, SHALL capture the full response, and SHALL update aanlevering.status based on the response.

- GIVEN an aanlevering in status=gevalideerd WHEN the user triggers submit THEN the system POSTs the payload to CBS-Kredo, captures the synchronous submissionId, and sets status=ingediend.
- GIVEN CBS-Kredo asynchronously returns `accepted` with an acknowledgement document WHEN openconnector polls the response endpoint THEN the system updates status=geaccepteerd and stores the acknowledgement in Iv3SubmissionResponse.
- GIVEN CBS-Kredo returns `rejected-validation` with a list of error codes WHEN the response is received THEN the system sets status=teruggewezen, links every error code to the offending taakveld/grootboek-rekening, and notifies the responsible afdelings-controller.

### REQ-008: Correctie-aanleveringen

The system SHALL support correctie-aanleveringen against any previously submitted aanlevering and SHALL maintain a chain from correctie to original.

- GIVEN a Q1-2027 aanlevering in status=geaccepteerd WHEN a material error is discovered and a user creates a correctie THEN the system creates a new Iv3Aanlevering with periode=correctie-1, links it to the original via `correctsRef`, and generates a delta-XML or full-replacement-XML per CBS schema rules.
- GIVEN three correcties against Q1-2027 WHEN the user views the aanlevering history THEN the system surfaces the chain Q1 → correctie-1 → correctie-2 → correctie-3 with the currently effective payload highlighted.
- GIVEN a correctie that introduces inconsistency with the Y-aanlevering already submitted WHEN it is validated THEN the system warns and recommends a Y-correctie.

### REQ-009: Audit log of every aanlevering action

Every state transition, payload generation, validation run, and submission SHALL be captured in an append-only audit log with actor, timestamp, contentHash, and reason.

- GIVEN any state transition on an Iv3Aanlevering WHEN it occurs THEN the system writes an Iv3AuditEvent record capturing the prior and new state, actor, IP, timestamp, and (where applicable) the payloadHash.
- GIVEN an auditor calls `GET /api/iv3/aanleveringen/{id}/audit-trail` WHEN the request is authorised THEN the system returns the complete sequence from creation to current state.
- GIVEN an attempt to delete or modify an audit record WHEN it is made via any interface THEN the system rejects with `AUDIT_IMMUTABLE`.

### REQ-010: Pre-deadline readiness dashboard

The system SHALL expose a per-organisation readiness view that, at any moment, summarises which aanleveringen are due within the next 30 days, the current validation status of each, and the list of blocking issues per aanlevering.

- GIVEN the current date is 2027-04-01 WHEN the readiness dashboard is rendered THEN the system shows Q1-2027 as `due 2027-04-30`, lists current validation status, and surfaces every blocker.
- GIVEN a blocker on Q1-2027 traced to GrootboekMapping for rekening 411000 WHEN a user clicks through THEN the system deep-links to the mapping record with the failing validation pre-applied.
- GIVEN an aanlevering reaches T-7 days with unresolved blockers WHEN the daily evaluator runs THEN the system escalates to the concerncontroller via the same escalation framework defined in `pc-cyclus-workflow`.

## Standards & Sources

- **Iv3-richtlijnen** — joint publication CBS / BZK / Financiën / commissie BBV, current 2026 edition; defines taakvelden, economische categorieën, balansposten, and aanleveringsregels.
- **EDA-XML** — Electronic Data Adjustment XML, schema versions per boekjaar; XSDs published by CBS.
- **Wet financiering decentrale overheden (Wet Fido)** art. 3 lid 4 — statutory basis for Iv3-plicht.
- **Regeling informatie voor derden (RIV)** — operationalises Wet Fido obligations.
- **ESA 2010** — European System of Accounts; defines EMU-saldo and EDP reporting derived from Iv3 data.
- **BBV** — taakvelden taxonomy is shared with begroting and jaarrekening structure.
- **CBS-Kredo** — submission portal; documentation at cbs.nl/nl-nl/onze-diensten/methoden/dataverzameling/decentrale-overheden.
- **NBA-handreiking 1108** — accountantsprotocol references Iv3 reconciliation in jaarrekening controle.
- **Reference systems** — Cognos Controller (IBM) and SAP S/4HANA Public Sector both implement EDA-XML output; community open-source: none mature as of 2026.

## Cross-app integration

- **bookkeeping-bbv-compliance** (financeq) — supplies grootboek-rekeningen and grootboek-mutaties; defines the BBV-taakveld linkage that mostly aligns with Iv3-taakvelden (with documented deltas captured in the GrootboekMapping).
- **pc-cyclus-workflow** (financeq) — Y-aanlevering reconciliation depends on the vastgestelde jaarrekening Deliverable; quarterly aanleveringen feed BERAP analytics.
- **bookkeeping-budget-forecast** (financeq) — raming-aanleveringen consume the latest forecast version.
- **openconnector** — outbound source `cbs-iv3-taxonomy` (annual taxonomy pull), outbound destination `cbs-kredo-submission` (aanlevering POST + polling), outbound destination `bzk-financieel-toezicht` (status updates to toezichthouder). Credentials, retries, and rate-limits live in openconnector.
- **openregister** — every register inherits OR audit-trail, versioning, search, and access control.
- **n8n** — pre-deadline notifications (REQ-010), daily reconciliation runs, taxonomy update flows.
- **mydash** — exposes a "Iv3-readiness" widget consumed by concerncontrollers.
- **docusaurus journeydoc** — ships a "Iv3-aanlevering Q1 doen" how-to with step-by-step screenshots; updated yearly in lockstep with the taxonomy bump.
- **docudesk** — long-term archival of submitted Iv3Payloads (retention: Archiefwet, decentrale-overheden categorie financiële verantwoording).
- **driver-based-forecasting** (financeq) — drivers used by GrootboekMapping splitMethod=driver are sourced from this spec; the two specs share the driver-formula DSL.

## Target users

- **Concerncontroller / hoofd Financiën** — owns Iv3-readiness, signs off on submissions, escalates issues. Daily-during-deadline-week user.
- **Iv3-coordinator** (in larger gemeenten and provincies a dedicated role) — maintains the GrootboekMapping, runs the aggregations, produces and submits the payloads.
- **Afdelings-controllers** — review their programma's contribution to the aanlevering before sign-off.
- **Accountant** — uses the Y-aanlevering reconciliation as part of jaarrekening-controle; reads audit log.
- **BZK financieel toezicht en provincie financieel toezicht** — consumes the aanlevering status feed to monitor non-compliance.
- **CBS / Kredo-team** — receives the aanleveringen, returns acknowledgements and rejections.
- **CPB / Ministerie van Financiën** — downstream consumer for macro-economic projections.
- **Burger / journalist / onderzoeker** — consumes the eventual statline-publicaties derived from the aggregated Iv3-data.
- **VNG / IPO / UvW** — sector-wide consumption of anonymised submission-timeliness telemetry.
