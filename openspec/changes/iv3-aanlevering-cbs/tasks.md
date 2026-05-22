---
status: ready
---

# Implementation Tasks: Iv3-Aanlevering CBS

## Phase 1: Data Model & Schema (Week 1-2)

- [ ] Define Iv3Taxonomy schema in financeq OpenAPI spec
  - [ ] Entity: boekjaar, taxonomyVersion, taakvelden[], economischeCategorieen[], balansposten[], mutatieKenmerken[], gemeenteCodes[]
  - [ ] Validation: ensure taakveld codes match CBS-published format (0.1-8.3, hierarchical parents)
  - [ ] OpenRegister inheritance: audit-trail, versioning, search

- [ ] Define GrootboekMapping schema
  - [ ] Entity: organisationRef, boekjaar, grootboekRekeningRef, taakveldCode, economischeCategorieCode, balanspostCode
  - [ ] Split methods: full, percentage (validation: sum=100%), driver (formula DSL)
  - [ ] Effective date ranges: effectiveFrom, effectiveTo
  - [ ] Audit versioning: mappedBy, reviewedBy, reviewedAt, version, createdAt, updatedAt

- [ ] Define Tussenrekening schema (intermediate aggregation layer)
  - [ ] Entity: code, naam, aggregationFormula (DSL), outputTargets[]
  - [ ] Formula DSL parser: support GB-prefix references and driver invocations

- [ ] Define Iv3Aanlevering schema (state machine)
  - [ ] Entity: boekjaar, periode (Q1-Q4, Y, correctie-n), organisationRef, cbsCode
  - [ ] Status machine: in-voorbereiding → gevalideerd → ingediend → (geaccepteerd | teruggewezen) → gecorrigeerd
  - [ ] Links: payloadVersionRef, cbsResponseRef, correctsRef, mappingSnapshotRef

- [ ] Define Iv3Payload schema (immutable)
  - [ ] Entity: aanleveringRef, versionNumber, format (eda-xml-v3.5, eda-xml-v4.0), xmlBlob, contentHash
  - [ ] ValidationReport: JSON array of check results (id, severity, result, details)
  - [ ] Immutability constraint: readonly=true after creation

- [ ] Define Iv3SubmissionResponse schema
  - [ ] Entity: payloadRef, submissionId (CBS-issued), acknowledgedAt, responseCode, responseDetails
  - [ ] validationErrors[]: array of CBS error codes with taakveld/rekening references

- [ ] Define Iv3ReconciliationCheck schema (derived register)
  - [ ] Entity: aanleveringRef, checkType, result (pass/warn/fail), details, tolerance, runAt
  - [ ] Check types: balanstotaal-sluitend, baten-lasten-saldi, deelnemingen-apart, reserves-mutaties, y-reconciliation, cumulatief, gemeente-code, taxonomy-version

- [ ] Define Iv3AuditEvent schema (immutable append-only)
  - [ ] Entity: aanleveringRef, eventType, actorId, timestamp, priorState, newState, payloadHash, ipAddress, reason
  - [ ] Immutability: readonly=true, no deletes

- [ ] Create migrations for all schemas (PostgreSQL)
  - [ ] Indices: (organisationRef, boekjaar, periode), (aanleveringRef, eventType), contentHash
  - [ ] Constraints: unique (aanleveringRef, versionNumber), foreign keys with cascades

---

## Phase 2: Taxonomy Ingestion (Week 2-3)

- [ ] Integrate with openconnector source `cbs-iv3-taxonomy`
  - [ ] Design openconnector configuration: CBS publication endpoint, yearly load schedule
  - [ ] Implement CBS XML/JSON parser for CBS-published taakvelden and economische categorieën
  - [ ] Support multiple taxonomy versions per boekjaar (retain prior versions)

- [ ] Taxonomy API endpoints
  - [ ] GET /api/iv3/taxonomy/{boekjaar} — retrieve active taxonomy for year
  - [ ] GET /api/iv3/taxonomy/{boekjaar}/{version} — retrieve specific version
  - [ ] POST /api/iv3/taxonomy/load — trigger manual load (admin-only)
  - [ ] GET /api/iv3/taxonomy/versions — list all loaded versions with timestamps

- [ ] Taxonomy validation on load
  - [ ] Validate taakveld codes match expected format (0.1-8.3)
  - [ ] Validate economische categorie codes match format
  - [ ] Detect herindelingen (gemeentecodes with validTo < current date)
  - [ ] Pre-check: ensure new version is truly newer (no downgrade)

- [ ] Notification on taxonomy bump
  - [ ] Detect version change (2027.1 → 2027.2)
  - [ ] Compute and store diff (added/removed/renamed taakvelden)
  - [ ] Trigger n8n workflow: email concerncontroller with diff summary
  - [ ] Dashboard notification: "CBS taxonomy updated; review changes for impact on aanleveringen"

---

## Phase 3: Mapping Configuration (Week 3-4)

- [ ] GrootboekMapping management UI
  - [ ] List view: all mappings for an organisation / boekjaar
  - [ ] Create/edit form:
    - [ ] Dropdown: select grootboek-rekening (pull from bookkeeping-bbv-compliance)
    - [ ] Dropdown: select taakveld (from loaded taxonomy)
    - [ ] Dropdown: select economische categorie
    - [ ] Radio: split method (full | percentage | driver)
    - [ ] Conditional: if percentage, show matrix of targets + percentages (validation: sum=100%)
    - [ ] Conditional: if driver, dropdown of available drivers (from driver-based-forecasting)
    - [ ] Text field: mapping rationale (required)
    - [ ] Date range: effectiveFrom, effectiveTo

- [ ] Split method logic
  - [ ] Full: simple 100% allocation
  - [ ] Percentage: validate sum = 100%, allow fractions (e.g., 33.33%, 66.67%)
  - [ ] Driver: DSL expression parser (support formula syntax: `GB{code} * (var / total)`)
  - [ ] Driver evaluation: fetch driver context (e.g., urenregistratie hours) at aggregation time

- [ ] Mapping audit versioning
  - [ ] Track all changes: createdAt, updatedAt, mappedBy, reviewedBy, reviewedAt
  - [ ] Freeze snapshot at aanlevering submission (mappingSnapshotRef)
  - [ ] Allow retrieval of historical mapping state for audit trail

- [ ] Tussenrekening management (optional, for complex organisations)
  - [ ] Create/edit form: code, naam, aggregationFormula (DSL editor)
  - [ ] Output targets: list of taakveld + percentage pairs
  - [ ] Formula validation: parse formula, ensure all referenced GB-rekeningen and drivers exist

- [ ] Mapping validation
  - [ ] REQ-002-002: Percentage sum validation (must == 100%)
  - [ ] REQ-002-003: Unmapped period detection (if mapping has gap, flag in aggregation)
  - [ ] Cross-check: ensure all taakvelden in a mapping are valid for the boekjaar (per taxonomy)

---

## Phase 4: Aggregation Engine (Week 4-5)

- [ ] Aggregation algorithm design
  - [ ] Input: Iv3Aanlevering (periode, boekjaar), GrootboekMapping (snapshot), grootboek-mutaties
  - [ ] Processing:
    - [ ] For each mapped rekening, retrieve GL mutaties for the periode
    - [ ] Apply split method: full (100%), percentage (multiply), or driver (evaluate formula)
    - [ ] Accumulate per (taakveld, economische categorie, balanspost) intersection
    - [ ] Compute saldo: baten - lasten + mutaties_reserves
    - [ ] Compute balanstotaal: sum of activa/passiva per balanspost
  - [ ] Output: aggregated ventilation structure (ready for XML serialisation)

- [ ] Driver evaluation engine
  - [ ] Load driver context (e.g., urenregistratie hours per taakveld) from driver-based-forecasting
  - [ ] Parse and evaluate DSL formula: `GB411000 * (hours_taakveld_1_5 / total_hours)`
  - [ ] Record inputs and output in splitEvaluationLog (audit trail)
  - [ ] Handle missing drivers: surface DRIVER_NOT_FOUND error (blocker)

- [ ] Aggregation API endpoint
  - [ ] POST /api/iv3/aanleveringen/{id}/aggregate
  - [ ] Input: organisationRef, boekjaar, periode
  - [ ] Processing:
    - [ ] Load taxonomy for boekjaar (or fail with TAXONOMY_MISSING)
    - [ ] Load mapping snapshot (or current mappings if not yet submitted)
    - [ ] Fetch GL mutaties for periode
    - [ ] Run aggregation algorithm
    - [ ] Generate aggregated structure (JSON)
  - [ ] Output: aggregated ventilation, ready for XML or payload display

- [ ] Aggregation caching
  - [ ] Cache aggregation result with contentHash
  - [ ] Invalidate cache if:
    - [ ] Underlying GL mutaties change
    - [ ] Mapping is modified (if not yet submitted)
    - [ ] Driver context changes
  - [ ] Surface "stale" indicator if payload exists but aggregation has changed

---

## Phase 5: EDA-XML Serialisation (Week 5-6)

- [ ] EDA-XML schema support
  - [ ] Bundle CBS-published XSD schemas (eda-xml-v3.5, eda-xml-v4.0) in the app
  - [ ] Map boekjaar → required schema version (e.g., 2027 → eda-xml-v4.0)
  - [ ] Schema lookup: GET endpoint to query required format for a boekjaar

- [ ] XML generation engine
  - [ ] Design serializer: aggregated ventilation → EDA-XML
  - [ ] Structure:
    - [ ] Header: CBSData, aanleveringsId, organisation, periode, boekjaar
    - [ ] Body: Aanleveringen[] with per-taakveld rows (baten, lasten, mutaties, saldi, balansposten)
    - [ ] All monetary amounts: use core-financial-objects.Money (EUR, 2 decimal places)
  - [ ] Namespace and XML declaration: per XSD version
  - [ ] Indentation and formatting: human-readable (for audit review)

- [ ] XML validation
  - [ ] Load XSD for boekjaar
  - [ ] Validate generated XML against XSD
  - [ ] If validation fails: report XPath-level violations (e.g., "/CBSData/Aanlevering/Baten[1]/Bedrag" expected xs:decimal)
  - [ ] Block payload creation if XSD validation fails (REQ-003-002)

- [ ] Payload creation API
  - [ ] POST /api/iv3/aanleveringen/{id}/generate-xml
  - [ ] Input: aanleveringRef
  - [ ] Processing:
    - [ ] Run aggregation (if not cached)
    - [ ] Serialize to XML
    - [ ] Validate XSD
    - [ ] Compute contentHash (SHA256)
    - [ ] Create Iv3Payload record (versionNumber=1)
  - [ ] Output: Iv3Payload with xmlBlob, contentHash, validationReport
  - [ ] On success: update aanlevering.payloadVersionRef

- [ ] XML immutability
  - [ ] Once Iv3Payload is created, xmlBlob is readonly
  - [ ] Detect changes to underlying data (GL mutaties, mappings)
  - [ ] If data changes: mark payload as "stale" (surface indicator)
  - [ ] If user re-aggregates: create Iv3Payload v2 (increment versionNumber)

---

## Phase 6: Local Validation Suite (Week 6-7)

- [ ] CBS-Kredo validation rule codification
  - [ ] Map all CBS error codes to Python/TypeScript validation functions
  - [ ] Classification: blocker (fail submission) vs warn (acknowledge and proceed)
  - [ ] Implement validators:
    - [ ] BAL_001: Balanstotaal sluitend (Activa = Passiva to the cent)
    - [ ] BAL_002: Baten - Lasten + Mutaties = Saldo
    - [ ] STR_014: Deelnemingen apart (taakveld 0.5 in separate row)
    - [ ] CUM_001: Quartaal cumulatief (Q2 ⊇ Q1, Q3 ⊇ Q1+Q2, Q4+Y reconciliation)
    - [ ] OPB_002: Opbrengsten/uitgaven onbeklemtoond (warn)
    - [ ] GEM_001: Gemeentecode actueel (gemeente valid for periode)
    - [ ] TAX_001: Taxonomy matches boekjaar

- [ ] Validation engine
  - [ ] API endpoint: POST /api/iv3/aanleveringen/{id}/validate
  - [ ] Input: aanleveringRef, payloadRef (optional; validate latest)
  - [ ] Processing:
    - [ ] Run all applicable validators against the payload
    - [ ] Aggregate results: blockers[] and warnings[]
    - [ ] Compute overall status: blocker detected? fail : pass
  - [ ] Output: validationReport JSON array:
    ```json
    [
      {
        "checkId": "BAL_001_BALANSTOTAAL_NIET_SLUITEND",
        "severity": "blocker",
        "result": "pass",
        "details": "Activa EUR 100M = Passiva EUR 100M"
      },
      {
        "checkId": "OPB_002_ONBEKLEMTOOND",
        "severity": "warn",
        "result": "warn",
        "details": "Category XXX has zero balance; unusual but permitted"
      }
    ]
    ```

- [ ] Validation state transitions
  - [ ] If all blockers pass: allow transition to status=gevalideerd
  - [ ] If any blocker fails: reject transition (surface blocking issues UI)
  - [ ] If warn-class: require user acknowledgement (acknowledgedWarnings[])

- [ ] Reconciliation checks (derived from validation)
  - [ ] Create Iv3ReconciliationCheck records (one per check type)
  - [ ] Store in database for historical audit trail
  - [ ] Update runAt timestamp on each validation run

---

## Phase 7: CBS Submission & Response Capture (Week 7-8)

- [ ] openconnector integration: `cbs-kredo-submission`
  - [ ] Configure openconnector destination:
    - [ ] CBS-Kredo API endpoint (production URL)
    - [ ] Authentication: SSL cert + API key (stored securely)
    - [ ] Retry policy: exponential backoff, max 3 retries
  - [ ] Payload format: POST application/xml, body = Iv3Payload.xmlBlob

- [ ] Submission API endpoint
  - [ ] POST /api/iv3/aanleveringen/{id}/submit
  - [ ] Pre-checks:
    - [ ] status == gevalideerd
    - [ ] All blockers passed in latest validation
    - [ ] payloadVersionRef is set (XML generated)
  - [ ] Processing:
    - [ ] POST to CBS-Kredo via openconnector
    - [ ] Capture synchronous response: submissionId (CBS-issued)
    - [ ] Update aanlevering:
      - [ ] status = ingediend
      - [ ] submittedAt = now
    - [ ] Record Iv3AuditEvent: eventType=submitted-to-kredo
  - [ ] Output: {status: "ingediend", submissionId: "KR-2027-000001-ABC"}

- [ ] Response polling via openconnector
  - [ ] Design n8n workflow:
    - [ ] Daily poll: check all ingediend aanleveringen for CBS response
    - [ ] Use submissionId to query CBS-Kredo response endpoint
    - [ ] If response is final (accepted/rejected), call webhook to update aanlevering
  - [ ] Webhook endpoint: PATCH /api/iv3/submissions/{submissionId}/response
    - [ ] Input: submissionId, responseCode, responseDetails, validationErrors[]
    - [ ] Processing:
      - [ ] Lookup Iv3Aanlevering by submissionId
      - [ ] Create Iv3SubmissionResponse record
      - [ ] Update aanlevering.status:
        - [ ] responseCode=accepted → status=geaccepteerd
        - [ ] responseCode=rejected-* → status=teruggewezen
      - [ ] Record Iv3AuditEvent: eventType=response-received
    - [ ] Notifications:
      - [ ] Email concerncontroller (acceptance/rejection)
      - [ ] Dashboard alert
      - [ ] If rejection: include CBS error codes + remediation suggestions

- [ ] Error linking (CBS error code → affected taakveld/rekening)
  - [ ] CBS validationErrors[] include code (e.g., "BAL_001") and taakveld (e.g., "4.1")
  - [ ] Reverse-map to affected GrootboekMapping records
  - [ ] Surface in rejection notification: "Error in taakveld 4.1; check mappings for rekeningen [411000, 412000]"

---

## Phase 8: Correctie-Aanleveringen (Week 8-9)

- [ ] Correctie creation workflow
  - [ ] API endpoint: POST /api/iv3/aanleveringen/{id}/create-correctie
  - [ ] Input: originalAanleveringRef, periode (auto-compute correctie-n), reason
  - [ ] Processing:
    - [ ] Create new Iv3Aanlevering:
      - [ ] periode = correctie-1 (or correctie-2, etc., if prior exists)
      - [ ] correctsRef = original aanlevering
      - [ ] status = in-voorbereiding
      - [ ] Copy: boekjaar, organisationRef, cbsCode
    - [ ] Initialize with original's mappings and aggregation
    - [ ] Record Iv3AuditEvent: eventType=correctie-created
  - [ ] Output: new Iv3Aanlevering (ready for modification)

- [ ] Correctie modification
  - [ ] Allow user to override specific taakveld amounts or mappings
  - [ ] Re-aggregate (either delta or full replacement, per CBS schema)
  - [ ] Re-validate before submission
  - [ ] Warn if correctie changes data used in already-submitted Y-aanlevering

- [ ] Correctie chain maintenance
  - [ ] Link original → correctie-1 → correctie-2 via correctsRef
  - [ ] Mark original as status=gecorrigeerd when correctie is submitted
  - [ ] UI: show chain (timeline or breadcrumb)
  - [ ] Reconciliation: update downstream expectations (Q3/Q4/Y cumulative checks)

- [ ] Correctie submission
  - [ ] Follow same path as original (validate, submit, wait for response)
  - [ ] CBS-Kredo: generate delta-XML or full-replacement per schema

---

## Phase 9: Audit Trail & Immutability (Week 9-10)

- [ ] Iv3AuditEvent schema + storage
  - [ ] Implement append-only storage (PostgreSQL: one table, no updates/deletes)
  - [ ] API endpoint: GET /api/iv3/aanleveringen/{id}/audit-trail
    - [ ] Return all events for an aanlevering, chronologically
    - [ ] Include filtering: by eventType, dateRange, actor
    - [ ] Support export: CSV, PDF

- [ ] State snapshot capture
  - [ ] For each event: priorState and newState (JSON snapshots)
  - [ ] Include all fields: status, payloadHash, taxonomy version, etc.
  - [ ] Enable reconstruction: "What was the state at T=2027-04-15?"

- [ ] Immutability enforcement
  - [ ] Database constraints: no UPDATE or DELETE on Iv3AuditEvent table
  - [ ] Application layer: reject any modify/delete attempt with AUDIT_IMMUTABLE error
  - [ ] Monitor: log and alert on attempted violations

---

## Phase 10: Readiness Dashboard & Escalations (Week 10-11)

- [ ] Readiness view component
  - [ ] Display aanleveringen due within 30 days
  - [ ] For each: status, progress bar (validation %), blocking issue count
  - [ ] Colour coding: red (teruggewezen/blockers), yellow (in-voorbereiding/warnings), green (gevalideerd/geaccepteerd)
  - [ ] Expandable: show each blocker with remediation links

- [ ] Deep-linking to mapping issues
  - [ ] When user clicks "Investigate" on a blocker
  - [ ] Link to specific GrootboekMapping record
  - [ ] Pre-apply filter to highlight the failing validation
  - [ ] Suggest remediation (e.g., "Add missing mapping", "Change split method")

- [ ] Escalation workflow
  - [ ] n8n job: daily, check all in-progress aanleveringen
  - [ ] Flag those with blockers + T-7 days to deadline
  - [ ] Send email to concerncontroller: "Q1-2027: 7 days to deadline, 2 unresolved blockers"
  - [ ] Update dashboard alert priority

- [ ] Integration with mydash widget
  - [ ] Expose Iv3-readiness widget for concerncontrollers
  - [ ] Show upcoming deadlines, status, blocker count
  - [ ] Link back to full readiness dashboard

---

## Phase 11: Documentation & Journeydoc (Week 11-12)

- [ ] User journeys (docusaurus journeydoc)
  - [ ] Journey: "Iv3-aanlevering Q1 voorbereiding"
    - [ ] Step 1: Load taxonomy for boekjaar 2027
    - [ ] Step 2: Review/update GrootboekMapping
    - [ ] Step 3: Run aggregation
    - [ ] Step 4: Review validation results
    - [ ] Step 5: Generate XML
    - [ ] Step 6: Submit to CBS-Kredo
  - [ ] Include screenshots, expected outcomes, troubleshooting
  - [ ] Update yearly in lockstep with taxonomy changes

- [ ] Admin guide
  - [ ] Taxonomy loading procedure (manual + automated via openconnector)
  - [ ] Debugging aggregation issues (logs, driver evaluation context)
  - [ ] Handling CBS-Kredo rejections (reverse-map errors to taakvelden)

- [ ] Accountant guide
  - [ ] How to access and review Y-aanlevering reconciliation
  - [ ] How to audit submission history (audit trail queries)
  - [ ] How to verify correcties and chain integrity

---

## Phase 12: Testing & QA (Week 12-13)

- [ ] Unit tests
  - [ ] Aggregation algorithm: various split methods, cumulatief chains
  - [ ] Driver DSL parser: formula evaluation with different inputs
  - [ ] Validation rules: all CBS error codes, edge cases (rounding, zero balance, etc.)

- [ ] Integration tests
  - [ ] End-to-end: create aanlevering, map, aggregate, validate, submit, receive response
  - [ ] Correctie workflow: create correctie, modify, revalidate, resubmit
  - [ ] Taxonomy bump: load new version, detect impact on in-progress aanleveringen

- [ ] Regression tests
  - [ ] Ensure changes to one spec (e.g., driver-based-forecasting) don't break Iv3
  - [ ] openconnector: test submission/polling with mock CBS-Kredo endpoint

- [ ] Performance tests
  - [ ] Large gemeente (500+ rekeningen, 30+ drivers): aggregation <5s
  - [ ] Concurrent submissions: 100+ aanleveringen in parallel, no degradation
  - [ ] Audit trail query: retrieve 1000+ events in <1s

- [ ] Manual testing (UAT)
  - [ ] with real gemeente: load their mapping, aggregate Q1, validate, submit
  - [ ] Test readiness dashboard with Iv3-coordinators and concerncontrollers
  - [ ] Verify notification emails and escalations

---

## Phase 13: Deployment & Rollout (Week 13-14)

- [ ] Production deployment
  - [ ] Database migrations (create schemas, indices)
  - [ ] openconnector configuration (CBS-Kredo endpoints, credentials)
  - [ ] Feature flags (if needed for gradual rollout)
  - [ ] Monitoring: set up alerts for aggregation failures, submission rejections

- [ ] Early adopter rollout
  - [ ] Partner with 2-3 early gemeenten (Amsterdam, Rotterdam, etc.)
  - [ ] Install app, load taxonomy, set up mappings
  - [ ] Support Q1-2027 aanlevering through full lifecycle
  - [ ] Collect feedback, fix any issues

- [ ] General availability
  - [ ] Marketing/communication: VNG, IPO, UvW sector-wide announcement
  - [ ] Training: webinars for Iv3-coordinators, concerncontrollers
  - [ ] Support: establish support channel (email, docs, FAQs)

- [ ] Monitoring & observability
  - [ ] Track: aanleveringen submitted per gemeente, success rate, average time to submission
  - [ ] Metrics: aggregation duration, validation check pass rates, CBS rejection reasons
  - [ ] Dashboard: sector-wide anonymised timeliness telemetry (for VNG/IPO/UvW)

---

## Phase 14: Post-Launch Maintenance (Ongoing)

- [ ] Taxonomy updates
  - [ ] Yearly CBS taxonomy bump (early calendar year)
  - [ ] Load new version, test impact on in-progress aanleveringen
  - [ ] Update journeydoc with any changes

- [ ] Bug fixes & improvements
  - [ ] Monitor for issues from real gemeente use
  - [ ] Improve error messages based on feedback
  - [ ] Optimize performance if needed

- [ ] CBS schema version support
  - [ ] When CBS publishes eda-xml-v4.1 or v5.0, bundle and test
  - [ ] Update format mappings (boekjaar → schema version)

- [ ] Integration improvements
  - [ ] Deepen integration with bookkeeping-bbv-compliance (GL sync, real-time notifications)
  - [ ] Extend driver-based-forecasting DSL if needed
  - [ ] Explore macro-level EMU-saldo aggregation (future)

