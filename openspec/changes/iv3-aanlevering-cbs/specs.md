---
status: ready
---

# Technical Specifications: Iv3-Aanlevering CBS

## REQ-001: Taxonomy Ingestion Per Boekjaar

**Requirement:** The system SHALL ingest the CBS-published Iv3-taxonomie at least annually and SHALL refuse to produce aanleveringen for a boekjaar where no matching taxonomy is loaded.

### REQ-001-001: Taxonomy Load on Demand

```gherkin
GIVEN no Iv3Taxonomy exists for boekjaar 2027
WHEN a user attempts to generate Q1-2027 aanlevering
THEN the system rejects the request with error `TAXONOMY_MISSING`
  AND responds with HTTP 400 Bad Request
  AND includes a remediation link to the openconnector source `cbs-iv3-taxonomy`
  AND surfaces a UI prompt to trigger taxonomy load
```

### REQ-001-002: Taxonomy Version Bumps During Year

```gherkin
GIVEN Iv3Taxonomy 2027.1 is loaded in the system
AND CBS publishes 2027.2 mid-year
WHEN the openconnector sync runs (daily, scheduled)
THEN the system loads 2027.2 as a new version
  AND retains 2027.1 (no deletion)
  AND surfaces a `taxonomy-version-bump` notification
  AND includes a structured diff showing:
    - Taakvelden added: list with codes and names
    - Taakvelden removed: list with codes and names (if any)
    - Taakvelden renamed: list with old → new names
  AND notifies concerncontroller via email/n8n workflow
  AND existing aanleveringen in progress continue using 2027.1
```

### REQ-001-003: Gemeente Herindelingen

```gherkin
GIVEN a herindeling whereby gemeente X (CBS-code 1234) opgaat in gemeente Y (CBS-code 5678) per 2027-01-01
WHEN Iv3Taxonomy 2027.1 is loaded
THEN the system records gemeente X with validTo=2026-12-31
  AND records gemeente Y with validFrom=2027-01-01
  AND aanleveringen for boekjaar 2026 Q4 and earlier continue using X's cbsCode=1234
  AND aanleveringen for boekjaar 2027 Q1 and later must use Y's cbsCode=5678
  AND validation checks reference the correct gemeente-code based on the aanlevering's periode
```

---

## REQ-002: Grootboek-to-Iv3 Mapping With Split Methods

**Requirement:** The system SHALL allow each grootboek-rekening to be mapped to one or more (taakveld, economische categorie) pairs using one of three split methods: full (100% to one target), percentage (fixed splits summing to 100%), or driver (formula evaluated per period).

### REQ-002-001: Driver-Based Allocation

```gherkin
GIVEN a salariskosten grootboek-rekening 411000
  AND the organisation uses splitMethod=driver
  AND the driver references urenregistratie-driver
WHEN the aanlevering aggregator runs for Q1-2027
THEN the system:
  1. Retrieves the driver evaluation context (total Q1 hours by taakveld)
  2. Evaluates the formula: GB411000 * (hours_taakveld / total_hours)
  3. Produces the per-taakveld split based on actual hours worked
  4. Records the driver inputs and output in `splitEvaluationLog` (JSON audit trail)
  5. Generates a report showing allocation: e.g., "Taakveld 1.5: EUR 500,000 (50% of 1,000,000), based on 5000/10000 hours"
```

### REQ-002-002: Percentage Validation

```gherkin
GIVEN a grootboek-rekening with splitMethod=percentage
  AND the split list specifies: 60% to taakveld 1.5, 39.5% to taakveld 0.1
  (summing to 99.5%, NOT 100%)
WHEN a user attempts to save the mapping
THEN the system rejects with HTTP 400
  AND error code `SPLIT_TOTAL_NOT_100`
  AND response includes the actual sum: 99.5%
  AND suggests correction: "Add 0.5% or adjust existing splits"
  AND the mapping is not persisted
```

### REQ-002-003: Unmapped Period Detection

```gherkin
GIVEN a grootboek-rekening for boekjaar 2027
  AND GrootboekMapping specifies effectiveFrom=2027-04-01 (no mapping for Q1-Q3)
WHEN Q1-2027 aggregation runs
THEN the system:
  1. Identifies that the rekening has no mapping for the Q1 periode (2027-01-01 to 2027-03-31)
  2. Includes the gap in the GROOTBOEK_UNMAPPED validation check
  3. Fails the check with result=fail
  4. Reports: "Rekening 411000: unmapped from 2027-01-01 to 2027-03-31; mapping effective from 2027-04-01"
  5. Blocks aggregation and aanlevering generation until the gap is resolved
```

---

## REQ-003: EDA-XML Serialisation Conformant To CBS Schema

**Requirement:** The system SHALL serialise each Iv3Aanlevering to EDA-XML using the exact schema version mandated by CBS for the boekjaar and SHALL validate the XML locally against the published XSD before allowing submission.

### REQ-003-001: Schema Version Selection

```gherkin
GIVEN an Iv3Aanlevering for boekjaar 2027
WHEN the user triggers the `generate-xml` action
THEN the system:
  1. Looks up the correct CBS schema version for boekjaar 2027 (from Iv3Taxonomy.format field)
  2. Uses eda-xml-v4.0 (the CBS-2027 schema, bundled with the app)
  3. Serialises the aggregated data to XML according to v4.0 namespace and structure
  4. Validates the generated XML against the bundled XSD schema
  5. If validation succeeds, creates an Iv3Payload record with format=eda-xml-v4.0
  6. If validation fails, blocks creation and reports XPath-level violations
```

### REQ-003-002: XSD Validation Blocking Submission

```gherkin
GIVEN a generated XML payload that fails local XSD validation
  (e.g., a required element is missing, or a numeric value exceeds allowed range)
WHEN the user attempts to submit to CBS-Kredo
THEN the system blocks submission with HTTP 400
  AND error code `XSD_VALIDATION_FAILED`
  AND includes a structured list of violations:
    - XPath to failing element (e.g., `/CBSData/Aanlevering/Baten[1]/Bedrag`)
    - Expected type (e.g., "xs:decimal with max 2 decimal places")
    - Actual value and reason for rejection
  AND surfaces a remediation workflow (e.g., "Fix the aggregation, regenerate XML, revalidate")
  AND the user cannot force-submit a non-validating payload
```

### REQ-003-003: Immutability After Generation

```gherkin
GIVEN a successfully generated Iv3Payload with format=eda-xml-v4.0
  AND the payload has been validated and its contentHash recorded
WHEN any subsequent change occurs to the underlying GrootboekMapping (e.g., a split % is adjusted)
  OR any change to the aggregated grootboek-mutaties (e.g., a GL entry is reversed)
THEN the existing Iv3Payload remains unchanged (immutable)
  AND the system:
    1. Detects the mismatch between payload contentHash and current data
    2. Surfaces a `payload-stale` indicator on the aanlevering UI
    3. Warns the user: "Underlying data has changed since payload generation; re-aggregate and regenerate XML to reflect current state"
    4. Blocks submission if the aanlevering is still in status=ingediend
    5. If the user proceeds, a new Iv3Payload v2 is created (versionNumber incremented)
```

---

## REQ-004: Local Pre-Run Of CBS-Kredo Validations

**Requirement:** The system SHALL execute the full CBS-Kredo validation suite locally before any submission and SHALL block submission unless every blocker-class validation passes.

### REQ-004-001: Blocker-Class Validation Enforcement

```gherkin
GIVEN an aanlevering with a balanstotaal mismatch of EUR 1.23
  (Activa EUR 100,000.00, Passiva EUR 100,001.23)
WHEN the local validator runs (during payload generation or pre-submission check)
THEN the validation check `BAL_001_BALANSTOTAAL_NIET_SLUITEND` is triggered
  AND severity=blocker
  AND result=fail
  AND details include:
    - "Activa total: EUR 100,000.00"
    - "Passiva total: EUR 100,001.23"
    - "Delta: EUR 1.23 (balance does not close)"
  AND submission is blocked until the delta is resolved
  AND a UI dialog suggests actions: "Review balanstotaal computation, adjust if needed, regenerate XML"
```

### REQ-004-002: Deelnemingen Separation Check

```gherkin
GIVEN an aanlevering where deelnemingen (taakveld 0.5) appear bundled with overige financiele baten (taakveld 1.3)
  (i.e., both are in the same economic-category / balanspost cross-foot row)
WHEN the local validator runs
THEN the validation check `STR_014_DEELNEMINGEN_APART_VEREIST` is triggered
  AND severity=blocker
  AND result=fail
  AND details include:
    - "Taakveld 0.5 (Deelnemingen) is bundled with taakveld 1.3"
    - References the offending grootboek-mutaties (e.g., GL entry 520000)
    - Suggests: "Create a separate allocation for taakveld 0.5"
  AND submission is blocked
```

### REQ-004-003: Warning-Class Validation Acknowledgement

```gherkin
GIVEN an aanlevering with a warn-class issue
  (e.g., opbrengsten/uitgaven onbeklemtoond — unusual revenue/expense category with zero balance)
WHEN the user triggers `validate` or `submit`
THEN the validation check (e.g., `OPB_002_ONBEKLEMTOOND`) is triggered
  AND severity=warn
  AND result=warn (not fail)
  AND a UI dialog surfaces:
    - The warning message
    - Option A: "Acknowledge and proceed (records warning in acknowledgedWarnings[])"
    - Option B: "Investigate (returns to mapping/aggregation review)"
  AND if the user selects "Acknowledge", the system records the warning in `acknowledgedWarnings`
  AND the aanlevering is allowed to proceed to status=gevalideerd and submission
  AND the warning is included in the Iv3Payload.validationReport
```

---

## REQ-005: Balanstotaal En Saldi Cross-Foot Checks

**Requirement:** The system SHALL enforce that, for every quarterly aanlevering, baten minus lasten plus mutaties reserves equals the period saldo and that the balanstotaal of activa equals the balanstotaal of passiva to the cent.

### REQ-005-001: Saldo Formula Validation

```gherkin
GIVEN aggregated results for Q1-2027:
  - Baten (all taakvelden): EUR 184,592,103.22
  - Lasten (all taakvelden): EUR 178,401,876.41
  - Mutaties reserves: EUR -3,190,226.81
  - Expected saldo: EUR 184,592,103.22 - EUR 178,401,876.41 + (-EUR 3,190,226.81) = EUR 3,000,000.00
WHEN the cross-foot check runs
THEN the system computes:
  saldo = SUM(baten) - SUM(lasten) + SUM(mutaties_reserves)
  saldo = EUR 3,000,000.00
  AND asserts saldo matches the reported period-saldo to the cent
  AND if mismatch > EUR 0.01, the check fails with detailed breakdown
```

### REQ-005-002: Balance-Sheet Closure

```gherkin
GIVEN aggregated balance-sheet totals for Q1-2027:
  - Activa balanstotaal: EUR 412,778,901.05
  - Passiva balanstotaal: EUR 412,778,901.50
  - Delta: EUR 0.45
WHEN the balanstotaal-sluitend check runs
THEN the check fails
  AND result=fail, severity=blocker
  AND details include:
    - "Activa: EUR 412,778,901.05"
    - "Passiva: EUR 412,778,901.50"
    - "Delta: EUR 0.45 (balance does not close)"
  AND references the contributing balansposten with amounts:
    - Activa breakdown: ACTIVA_VAST EUR X, ACTIVA_VLOTTEND EUR Y, ...
    - Passiva breakdown: PASSIVA_VERMOGEN EUR A, PASSIVA_SCHULDEN_VAST EUR B, ...
  AND submission is blocked
```

### REQ-005-003: Y-Aanlevering Jaarrekening Reconciliation

```gherkin
GIVEN:
  - Y-2027 aanlevering with aggregated taakveld totals from Q1+Q2+Q3+Q4
  - Vastgestelde jaarrekening (from pc-cyclus-workflow) with final approved amounts per taakveld
WHEN the Y-aanlevering is validated
THEN the system runs the reconciliation:
  FOR EACH taakveld:
    delta = ABS(aanlevering_amount - jaarrekening_amount)
    IF delta > EUR 0.01 per regel (single line)
      THEN flag as warning/fail, depending on configuration
    IF SUM(deltas per programma) > EUR 1.00 cumulatief
      THEN fail the check with severity=blocker
  AND if all taakveld deltas <= tolerance, check passes
  AND if any taakveld delta > tolerance, check fails with:
    - List of mismatched taakvelden with amounts and deltas
    - Suggested next steps: "Reconcile with accounting department, investigate GL postings"
```

---

## REQ-006: Quartaal-Mutaties Cumulatief

**Requirement:** The system SHALL enforce that Q2 figures include Q1 cumulatief, Q3 includes Q1+Q2, and Q4 + Y reconcile; deviations SHALL fail the `quartaal-mutaties-cumulatief` check.

### REQ-006-001: Cumulative Chain for Quarterly Data

```gherkin
GIVEN:
  - Q1-2027 aanlevering with cumulatieve lasten taakveld 4.1 = EUR 5,200,000
  - Q2-2027 aanlevering generated with cumulatieve lasten taakveld 4.1 = EUR 9,800,000
WHEN Q3-2027 is generated and validated
THEN the system asserts:
  Q3_lasten_cumulative >= EUR 9,800,000 (Q1 + Q2 baseline)
  OR Q3_lasten_cumulative >= EUR 9,800,000 - reversals
    (if explicit reversal entries are found, they are subtracted from expected baseline)
  IF Q3_lasten_cumulative < expected_baseline (without reversals)
    THEN check fails with:
      - "Q3 cumulative lasten taakveld 4.1: EUR X, expected >= EUR 9,800,000"
      - "Delta from Q2: EUR Y (negative, indicates reversal or error)"
      - "Clarification needed: Are there reversals? If so, please document."
```

### REQ-006-002: Correctie-Aanlevering Chain Updates

```gherkin
GIVEN:
  - Q2-2027 aanlevering with cumulatieve lasten taakveld 4.1 = EUR 9,800,000 (geaccepteerd)
  - A material error is discovered: actual should be EUR 9,700,000
  - User creates correctie-1 against Q2-2027
WHEN correctie-1 is submitted and accepted
THEN the system:
  1. Sets the original Q2 status=gecorrigeerd, links to correctie-1 via correctsRef
  2. Treats correctie-1 as the new "effective" Q2 for the cumulative chain
  3. Recomputes downstream consistency:
     - Q3 cumulative lasten must now be >= EUR 9,700,000 (the new Q2 baseline)
     - Q4 cumulative lasten must be >= EUR 9,700,000 + Q3_increment
  4. If existing Q3/Q4 aanleveringen violate the new chain, surfaces warnings and suggests Q3/Q4 correcties
```

### REQ-006-003: Y-Aanlevering Consistency

```gherkin
GIVEN:
  - Y-2027 aanlevering with totaal taakveld 4.1 = EUR 20,100,000
  - Q4-2027 aanlevering with cumulatief EUR 20,050,000
  - Reconciliation logic: Q4 cumulatief should == Y totaal (or within rounding tolerance)
WHEN the reconciliation runs
THEN the system:
  1. Detects delta: EUR 20,100,000 - EUR 20,050,000 = EUR 50,000
  2. If delta > EUR 1.00, warn and flag as issue
  3. Notifies: "Y-aanlevering does not reconcile with Q4 cumulative; delta EUR 50,000"
  4. Suggests remediation:
     A. Investigate whether Q4 aanlevering needs correction
     B. Create a correctie-Q4 if error found
     C. Or verify Y-aanlevering calculation and document the expected difference
  5. If the user chooses to proceed, records the acknowledged discrepancy in audit trail
```

---

## REQ-007: Submission To CBS-Kredo With Response Capture

**Requirement:** The system SHALL submit Iv3Payloads to the CBS-Kredo portaal via the official submission API, SHALL capture the full response, and SHALL update aanlevering.status based on the response.

### REQ-007-001: Submission Initiation

```gherkin
GIVEN an aanlevering in status=gevalideerd
  AND all blocking validations have passed
WHEN the user (concerncontroller or Iv3-coordinator) triggers `submit` action
THEN the system:
  1. Retrieves the latest Iv3Payload for this aanlevering
  2. POSTs the xmlBlob to CBS-Kredo API endpoint (via openconnector destination `cbs-kredo-submission`)
  3. Captures the synchronous response with CBS-issued submissionId (e.g., "KR-2027-000001-ABC")
  4. Sets aanlevering.status=ingediend
  5. Records aanlevering.submittedAt = current timestamp
  6. Creates Iv3AuditEvent with eventType=submitted-to-kredo, captures user and IP
  7. Returns success response to user: "Aanlevering submitted with ID: KR-2027-000001-ABC"
```

### REQ-007-002: Asynchronous Response Polling

```gherkin
GIVEN an aanlevering in status=ingediend with submissionId=KR-2027-000001-ABC
  AND CBS-Kredo is processing the aanlevering
WHEN openconnector polls the CBS-Kredo response endpoint (daily, per openconnector schedule)
THEN the system:
  1. Checks the response status using the submissionId
  2. If response is still pending, re-polls after delay (configurable, e.g., 6 hours)
  3. If response is final (accepted/rejected), creates Iv3SubmissionResponse record:
     - Captures responseCode (e.g., "accepted")
     - Captures responseDetails (CBS message)
     - Captures validationErrors[] (if rejected)
  4. Updates aanlevering.status:
     - If responseCode=accepted → status=geaccepteerd
     - If responseCode=rejected-validation or rejected-format → status=teruggewezen
     - If responseCode=partial → status=geaccepteerd (with warning)
  5. Notifies the concerncontroller via email/dashboard widget
  6. Creates Iv3AuditEvent with eventType=response-received
```

### REQ-007-003: Rejection Workflow

```gherkin
GIVEN CBS-Kredo asynchronously returns responseCode=rejected-validation
  WITH validationErrors = [
    {code: "BAL_001", taakveld: "4.1", message: "Saldo mismatch"},
    {code: "STR_014", taakveld: "0.5", message: "Deelnemingen not separate"}
  ]
WHEN the response is received by the system
THEN the system:
  1. Sets aanlevering.status=teruggewezen
  2. Stores validationErrors in Iv3SubmissionResponse
  3. Performs reverse-mapping to link each CBS error to the offending:
     - Taakveld (4.1, 0.5, etc.)
     - Grootboek-rekening(s) contributing to that taakveld
     - Mapping rule (if applicable)
  4. Notifies the responsible afdelings-controller and concerncontroller:
     - Email with subject: "Iv3-aanlevering Q1-2027 rejected by CBS"
     - Body includes: CBS error codes, affected taakvelden, suggested actions
  5. Creates Iv3AuditEvent with eventType=response-received, captures rejection reasons
  6. Surfaces a remediation workflow: "Create correctie-aanlevering or adjust mappings and resubmit"
```

---

## REQ-008: Correctie-Aanleveringen

**Requirement:** The system SHALL support correctie-aanleveringen against any previously submitted aanlevering and SHALL maintain a chain from correctie to original.

### REQ-008-001: Correctie Creation

```gherkin
GIVEN a Q1-2027 aanlevering in status=geaccepteerd
  AND a material error is discovered (e.g., taakveld 4.1 was calculated 10% too high)
WHEN a user with appropriate permissions (Iv3-coordinator, concerncontroller) creates a correctie
THEN the system:
  1. Creates a new Iv3Aanlevering with:
     - periode=correctie-1 (auto-incremented if correctie-1 already exists)
     - links to original Q1 via correctsRef
     - status=in-voorbereiding (starts fresh workflow)
     - boekjaar=2027, organisationRef, cbsCode copied from original
  2. Initializes the correctie aanlevering with the same mappings and aggregation as the original
  3. Allows the user to specify the delta or corrected values for affected taakvelden
  4. Regenerates XML (delta-XML or full-replacement, per CBS schema rules)
  5. Runs the full validation suite on the correctie payload
  6. Records in Iv3AuditEvent with eventType=correctie-created, capturing reason
```

### REQ-008-002: Correctie Chain Visibility

```gherkin
GIVEN three correcties against Q1-2027:
  - Original Q1-2027 aanlevering (geaccepteerd)
  - correctie-1 (geaccepteerd)
  - correctie-2 (geaccepteerd)
  - correctie-3 (ingediend, awaiting CBS response)
WHEN the user views the aanlevering history / details page
THEN the system displays:
  1. A chain diagram or timeline showing:
     Q1 → correctie-1 → correctie-2 → correctie-3
  2. For each entry in the chain:
     - Status (geaccepteerd, ingediend, etc.)
     - Submission date
     - CBS submissionId
     - Brief reason/description (from audit trail)
  3. Highlights the "currently effective" payload (correctie-3 if submitted; otherwise, the last geaccepteerd)
  4. Allows deep-linking to any aanlevering in the chain for detailed audit trail review
```

### REQ-008-003: Y-Aanlevering Consistency Warning

```gherkin
GIVEN:
  - Q1-2027 aanlevering submitted with taakveld 4.1 = EUR 5,000,000
  - Y-2027 aanlevering already submitted with total taakveld 4.1 = EUR 20,100,000
  - A correctie-1 against Q1-2027 is created, correcting taakveld 4.1 to EUR 5,200,000
WHEN correctie-1 is validated
THEN the system:
  1. Detects that the original Q1 data was included in the Y-aanlevering calculation
  2. Computes the impact: Y-total should now be EUR 20,100,000 + EUR 200,000 = EUR 20,300,000
  3. Surfaces a warning: "This correctie changes Q1 by EUR 200,000, which affects the Y-total. Consider creating a Y-correctie to maintain consistency."
  4. Suggests: "Create correctie-Y with delta EUR 200,000 for taakveld 4.1"
  5. Allows the user to proceed or cancel
```

---

## REQ-009: Audit Log Of Every Aanlevering Action

**Requirement:** Every state transition, payload generation, validation run, and submission SHALL be captured in an append-only audit log with actor, timestamp, contentHash, and reason.

### REQ-009-001: State Transition Logging

```gherkin
GIVEN an Iv3Aanlevering in status=in-voorbereiding
WHEN a state transition occurs (e.g., status changes to gevalideerd)
THEN the system writes an Iv3AuditEvent record with:
  - eventType=status-transitioned
  - actorId=user who initiated the transition
  - timestamp=ISO 8601 with millisecond precision
  - priorState={status: in-voorbereiding, ...prior snapshot}
  - newState={status: gevalideerd, ...new snapshot}
  - reason=user-provided justification or default reason
  - ipAddress=source IP address (if from HTTP request)
  AND the record is immutable (readonly=true) once created
```

### REQ-009-002: Audit Trail Query

```gherkin
GIVEN an Iv3Aanlevering (aanlevering-id=aanlev-2027-q1-ams-001)
  AND an auditor with appropriate permissions
WHEN the auditor calls GET /api/iv3/aanleveringen/{id}/audit-trail
THEN the system:
  1. Validates the request is authorised (checked against user role / organisation)
  2. Returns a JSON array of Iv3AuditEvent records in chronological order:
     - All events from creation to current state
     - Complete event details including actor, timestamp, state snapshots, reason
  3. Includes human-readable descriptions for each event
  4. Allows filtering by eventType, dateRange, or actor
  5. Enables export to CSV/PDF for external audit review
```

### REQ-009-003: Audit Immutability

```gherkin
GIVEN an Iv3AuditEvent record already persisted
WHEN an attempt is made to delete, modify, or overwrite the record
  (via any interface: API, database, admin tool)
THEN the system:
  1. Rejects the attempt with HTTP 403 Forbidden
  2. Error code: AUDIT_IMMUTABLE
  3. Response: "Audit records are immutable and cannot be modified or deleted"
  4. Logs the attempted modification attempt in a separate security log
  5. Alerts security/compliance team if multiple failed attempts are detected
```

---

## REQ-010: Pre-Deadline Readiness Dashboard

**Requirement:** The system SHALL expose a per-organisation readiness view that, at any moment, summarises which aanleveringen are due within the next 30 days, the current validation status of each, and the list of blocking issues per aanlevering.

### REQ-010-001: Readiness View

```gherkin
GIVEN the current date is 2027-04-01
  AND Q1-2027 deadline is 2027-04-30
WHEN the readiness dashboard is rendered (by a concerncontroller)
THEN the system displays:
  1. A list of due aanleveringen within 30 days:
     - Q1-2027: due 2027-04-30 (29 days remaining)
     - Status: in-voorbereiding
     - Blocking issues count: 2 blockers
  2. For each aanlevering:
     - Progress bar: "Validation 50% complete (3/6 checks passed)"
     - Current status with colour indicator:
       - Red: status=teruggewezen or has blockers
       - Yellow: status=in-voorbereiding with warnings
       - Green: status=gevalideerd or geaccepteerd
  3. Expandable section showing every blocking issue:
     - Issue: "BAL_001_BALANSTOTAAL_NIET_SLUITEND: delta EUR 1.23"
     - Issue: "STR_014_DEELNEMINGEN_APART_VEREIST: taakveld 0.5 bundled"
  4. Action buttons:
     - "Investigate" (deep-link to remediation)
     - "Mark as acknowledged" (if warn-class)
     - "Submit" (if all blockers cleared)
```

### REQ-010-002: Deep-Link to Mapping Issue

```gherkin
GIVEN a blocker on Q1-2027 traced to GrootboekMapping for rekening 411000
WHEN a concerncontroller clicks "Investigate" on the readiness dashboard
THEN the system:
  1. Deep-links to the specific GrootboekMapping record for rekening 411000
  2. Pre-applies the failing validation filter to highlight the issue
  3. Displays the mapping details:
     - Mapping rule: 411000 → taakveld 1.5, splitMethod=driver
     - Current driver evaluation: "Driver urenregistratie-driver not found"
     - Suggested fix: "Load driver data or change splitMethod"
  4. Allows inline editing of the mapping
  5. Triggers a quick re-validation to confirm fix before returning to dashboard
```

### REQ-010-003: Escalation Workflow

```gherkin
GIVEN an aanlevering (Q1-2027) with unresolved blockers
  AND the current date is 2027-04-23 (T-7 days from 2027-04-30 deadline)
WHEN the daily escalation evaluator runs (scheduled job, nightly)
THEN the system:
  1. Identifies all unresolved blockers on Q1-2027
  2. Escalates to the concerncontroller via:
     - Email notification with blocking issue summary
     - Dashboard alert widget: "Q1-2027: Escalation required (7 days to deadline, 2 blockers unresolved)"
     - Integration with pc-cyclus-workflow escalation framework (same mechanism as Budget Cycle escalations)
  3. Records escalation in audit trail
  4. Notifies BZK financieel toezicht if deadline passes with unresolved blockers (optional sink)
```

---

## Cross-Cutting Requirements

### REQ-100: Data Retention & Archival

```gherkin
GIVEN an Iv3Aanlevering and its associated Iv3Payload (submitted and geaccepteerd)
WHEN the aanlevering lifecycle is complete (status=geaccepteerd, no pending correcties)
THEN the system:
  1. Retains all records in live database for 5 years (Archiefwet requirements)
  2. Marks records as "archival-eligible" after 18 months
  3. Periodically exports archival-eligible records to docudesk (long-term archival service)
  4. Maintains read-only access to archived records via API / audit trail queries
  5. Deletes records only after legal retention period expires (7+ years)
```

### REQ-101: Accessibility & Localisation

```gherkin
GIVEN the Iv3-readiness widget and aanlevering management interface
WHEN rendered in a Dutch organisation's environment
THEN the system:
  1. Uses Dutch (nl-NL) for all UI labels, messages, and validation error texts
  2. Formats currency as EUR with Dutch locale (e.g., "€ 1.234.567,89" not "€1,234,567.89")
  3. Formats dates as DD-MM-YYYY (e.g., "01-04-2027")
  4. Uses Dutch terminology per Iv3-richtlijnen (taakveld, economische categorie, etc.)
  5. Provides docusaurus journeydoc in Dutch with step-by-step screenshots
```

### REQ-102: Performance & Scalability

```gherkin
GIVEN a large gemeente (>1M residents) with complex mapping (500+ rekeningen, 30+ drivers)
WHEN Q1 aggregation runs for the entire year
THEN the system:
  1. Completes aggregation in <5 seconds
  2. Completes XML generation in <2 seconds
  3. Completes full validation suite in <3 seconds
  4. Supports concurrent submission of 100+ aanleveringen without degradation
  5. Uses database indices on frequently-queried fields:
     - (organisationRef, boekjaar, periode) for aanlevering lookups
     - (aanleveringRef, eventType) for audit trail queries
```

---

## Appendix: Validation Checklist

All validations SHALL conform to the CBS-Kredo validation rules as documented in the Iv3-richtlijnen (current 2026 edition). The system maintains a mapping of CBS error codes to remediation workflows:

| CBS Error Code | Severity | Description | Remediation |
|---|---|---|---|
| BAL_001_BALANSTOTAAL_NIET_SLUITEND | blocker | Balance sheet does not close | Investigate GL entries, adjust balansposten |
| BAL_002_BATEN_LASTEN_SALDI_INCONSISTENT | blocker | Saldo formula mismatch | Recompute saldo or mutaties |
| STR_014_DEELNEMINGEN_APART_VEREIST | blocker | Deelnemingen bundled with other categories | Separate taakveld 0.5 into own row |
| OPB_002_ONBEKLEMTOOND | warn | Unusual revenue/expense with zero balance | Document reason or investigate |
| CUM_001_QUARTAAL_NICHT_CUMULATIEF | blocker | Quarterly cumulative chain broken | Investigate previous quarter, create correctie if needed |

