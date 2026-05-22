# Specifications: P&C-Cyclus Workflow

## REQ-001: Statutory Deadline Derivation

The system SHALL derive statutory deadlines automatically from stageType and boekjaar, using a sector-specific deadline calendar (gemeente, provincie, waterschap), and SHALL prevent users from setting a `plannedEnd` later than the statutory deadline without an explicit acknowledged override.

### REQ-001-001: Gemeente Programmabegroting Deadline

**GIVEN** a CyclusInstance for Gemeente Zeist boekjaar 2027 with templateRef=`cyclus-template-gemeente@2026.1`
**WHEN** a Stage of type `programmabegroting` is instantiated
**THEN** the system sets `statutoryDeadline` to 2026-11-15 (BBV art. 191 lid 2 — 15 november van het jaar voorafgaand)
**AND** exposes `legalBasis = "BBV art. 191 lid 2"`
**AND** the Stage is created with status=niet-gestart

### REQ-001-002: Override Validation with Acknowledgement

**GIVEN** a Stage of type `jaarrekening` for boekjaar 2026 with statutoryDeadline=2027-07-15
**WHEN** a planner attempts to set `plannedEnd` to 2027-07-31 (16 days after deadline)
**THEN** the system rejects with error `STATUTORY_DEADLINE_EXCEEDED`
**AND** returns error message referencing Gemeentewet art. 200 lid 2
**UNLESS** an `acknowledgedOverride` object is supplied with:
- `motivation`: string explaining the reason (e.g., "delayed audit completion")
- `approverRef`: userRef of the approving officer (gemeentesecretaris or hoger)
- `approverRole`: confirmed as gemeentesecretaris or concerncontroller
**THEN** the system accepts `plannedEnd = 2027-07-31`
**AND** creates an audit log entry with the override motivation

### REQ-001-003: Waterschap Template Deadline

**GIVEN** a Waterschap CyclusInstance with templateRef=`cyclus-template-waterschap@2026.1`
**WHEN** a Stage of type `programmabegroting` is created
**THEN** the system derives `statutoryDeadline` from Waterschapsbesluit (art. 98, 15 november aan provincie)
**AND** exposes `legalBasis = "Waterschapsbesluit art. 98"`
**NOT** from BBV

### REQ-001-004: Deadline Visibility in Response

**GIVEN** a GET request to `/api/stages/{stageId}`
**WHEN** the Stage has a statutoryDeadline
**THEN** the response includes:
```json
{
  "id": "stage-id",
  "stageType": "programmabegroting",
  "plannedEnd": "2026-11-01",
  "statutoryDeadline": "2026-11-15",
  "daysToDeadline": 14,
  "legalBasis": "BBV art. 191 lid 2",
  "deadlineStatus": "ok|at-risk|overdue"
}
```

---

## REQ-002: Stage Transition Gating

The system SHALL enforce that a StageTransition can only be marked `completed` when every `gateCriteria` validation evaluates true and every `requiredSignoffs` entry has a non-revoked Signoff against the deliverable's current version.

### REQ-002-001: Transition with Multiple Gate Criteria

**GIVEN** a Stage `voorjaarsnota` with:
- `gateCriteria = [financiele-saldi-sluitend, paragraaf-weerstandsvermogen-aanwezig]`
- `requiredSignoffs = [portefeuillehouder-financien, concerncontroller]`
- `currentStageRefs = ["stage-voorjaarsnota"]`

**WHEN** a user attempts to POST to `/api/stage-transitions` with sourceStageRef=stage-voorjaarsnota, targetStageRef=stage-programmabegroting
**AND** only the `portefeuillehouder-financien` signoff is present
**AND** the `paragraaf-weerstandsvermogen-aanwezig` validation fails

**THEN** the system rejects with HTTP 422:
```json
{
  "error": "TRANSITION_GATE_FAILED",
  "failedCriteria": [
    {
      "type": "validation",
      "name": "paragraaf-weerstandsvermogen-aanwezig",
      "status": "fail",
      "message": "Paragraaf weerstandsvermogen not found in document"
    }
  ],
  "missingSignoffs": [
    {
      "role": "concerncontroller",
      "requiredAt": "stage-programmabegroting",
      "currentStatus": "pending"
    }
  ],
  "nextCheckableAt": "2026-11-05T09:00:00Z"
}
```

### REQ-002-002: Atomic Transition Execution

**GIVEN** a StageTransition where every gate passes (all criteria=pass, all signoffs present and non-revoked)
**WHEN** a user with role `gemeentesecretaris` POSTs to `/api/stage-transitions/{transitionId}/complete`
**THEN** the system executes atomically:
1. Sets StageTransition.status=completed
2. Sets StageTransition.actualTransitionDate=now (ISO 8601)
3. Sets StageTransition.actualTransitionBy={userRef}
4. Sets source Stage.status=afgerond
5. Sets target Stage.status=in-voorbereiding
6. Emits a `stage.transitioned` event consumed by n8n and external subscribers

**AND** returns HTTP 200 with the updated StageTransition

### REQ-002-003: Revoked Signoff Warning

**GIVEN** a StageTransition that was marked completed at T-10
**AND** a Signoff attached to the deliverable is later revoked (revokedAt set)
**WHEN** the system re-evaluates the transition (either on-demand or daily audit)
**THEN** the system sets the target Stage with flag `upstreamSignoffRevoked=true`
**AND** exposes a `upstreamSignoffRevoked` warning in the Stage's API response:
```json
{
  "id": "stage-id",
  "status": "in-voorbereiding",
  "warnings": [
    {
      "type": "upstreamSignoffRevoked",
      "sourceStageRef": "stage-voorjaarsnota",
      "revokedSignoffRef": "signoff-uuid",
      "revokedBy": "u-bob",
      "revokedAt": "2026-11-12T15:30:00Z",
      "revokedReason": "incorrect data detected"
    }
  ]
}
```

**BUT** does NOT automatically roll back the transition
**INSTEAD** requires explicit human decision to either:
- Accept the revocation and re-validate gate criteria
- Revert to source stage (manual action)

---

## REQ-003: Document Versioning with Content Hashing

Every DeliverableVersion SHALL be immutable once created and SHALL carry a sha256 contentHash of its canonical serialisation; the system SHALL reject any attempt to mutate an existing version and SHALL require a new version for any change.

### REQ-003-001: Version Immutability

**GIVEN** a DeliverableVersion with id=`v1.2.0`, versionNumber=1.2.0, state=college-versie
**AND** the version contains a PDF attachment with filename=programmabegroting.pdf

**WHEN** a user with any role attempts to PATCH `/api/deliverable-versions/{versionId}` with any field change
**THEN** the system rejects with HTTP 409:
```json
{
  "error": "VERSION_IMMUTABLE",
  "versionId": "v1.2.0",
  "versionNumber": "1.2.0",
  "message": "This version cannot be modified. Create a successor version.",
  "nextVersionNumber": "1.2.1",
  "createVersionUrl": "/api/deliverables/{deliverableId}/versions"
}
```

### REQ-003-002: Content Hash Computation

**GIVEN** a new DeliverableVersion is being created via POST `/api/deliverables/{deliverableId}/versions`
**WITH** body containing:
```json
{
  "versionNumber": "1.0.0",
  "changeNote": "Final version after college approval",
  "attachments": [
    {
      "filename": "programmabegroting-2027.pdf",
      "contentHash": "sha256:xyz789..."
    }
  ]
}
```

**WHEN** the system processes the request
**THEN** it computes contentHash by:
1. Creating a canonical JSON representation with:
   - All keys in sorted (lexicographic) order
   - Whitespace normalised (no excess spaces/newlines)
   - Numeric values consistent format (e.g., 1250000.00 not 1.25e6)
2. Applying sha256 hash algorithm
3. Storing as contentHash field

**EXAMPLE:**
```
{
  "attachments": [{"contentHash": "sha256:xyz789...", "filename": "programmabegroting-2027.pdf"}],
  "changeNote": "Final version after college approval",
  "versionNumber": "1.0.0"
}
→ sha256:abc123def456...
```

### REQ-003-003: No Effective Change Detection

**GIVEN** a new DeliverableVersion is being created
**WITH** versionNumber=1.2.1, parentVersionRef=1.2.0
**WHEN** the system computes contentHash and the result matches parent's contentHash
**THEN** the system rejects with HTTP 422:
```json
{
  "error": "NO_EFFECTIVE_CHANGE",
  "parentVersionRef": "1.2.0",
  "parentContentHash": "sha256:abc123...",
  "message": "This version is identical to the parent. No new version created."
}
```

**EXCEPT** if changeNote explicitly signals "formatting fix" or "metadata update" (admin override allowed)

### REQ-003-004: Monotonic Version Numbers

**GIVEN** a Deliverable with existing versions: [0.1.0, 0.2.0, 1.0.0]
**WHEN** a user attempts to create version 0.3.0 (lower than current 1.0.0)
**THEN** the system rejects with HTTP 422:
```json
{
  "error": "VERSION_NOT_MONOTONIC",
  "currentVersion": "1.0.0",
  "attemptedVersion": "0.3.0",
  "message": "Version numbers must be monotonically increasing (e.g., 1.0.1, 1.1.0, 2.0.0)"
}
```

---

## REQ-004: Portefeuillehouder Sign-Off

The system SHALL support cryptographic sign-off by named role-holders against a specific DeliverableVersion using DigiD-Sign, qualified eIDAS, or an ad-hoc OTP method; the signature SHALL bind to the contentHash of that exact version and SHALL be independently verifiable.

### REQ-004-001: DigiD-Sign Signoff Request

**GIVEN** a portefeuillehouder (`u-bob-vandenberg`, role=wethouder-financien) receives a sign-off request
**WITH** email: "Programmabegroting 2027 requires your sign-off"
**AND** the Deliverable has currentVersionRef=v1.0.0
**AND** v1.0.0 has contentHash=sha256:abc123def456...

**WHEN** the portefeuillehouder clicks the sign-off link
**AND** is redirected to `/api/signoffs/request/{requestId}/digid-sign`
**AND** completes DigiD authentication with credentials
**THEN** the system:
1. Retrieves the contentHash from the pinned DeliverableVersion
2. Creates a PKCS#7 signature envelope containing the contentHash
3. Creates a Signoff record with:
   - `signedVersionRef = v1.0.0`
   - `signerRef = u-bob-vandenberg`
   - `signerRole = wethouder-financien`
   - `signedAt = now`
   - `signatureMethod = digid-sign`
   - `signatureBlob = -----BEGIN PKCS7-----...-----END PKCS7-----`
   - `contentHashSigned = sha256:abc123def456...`
4. Sends confirmation email to portefeuillehouder
5. Returns HTTP 200 with Signoff JSON

### REQ-004-002: Ad-Hoc OTP Signoff

**GIVEN** a Signoff request for a user without DigiD credentials
**WHEN** they choose "Sign with OTP" on the signoff request page
**THEN** the system:
1. Generates a 6-digit OTP code
2. Sends OTP via SMS (mobile number from Nextcloud userRef)
3. Prompts user to enter OTP
4. Validates OTP (time window: 10 minutes)
5. Creates Signoff record with `signatureMethod = ad-hoc-otp`
6. Stores hashed OTP in auditLog for compliance

### REQ-004-003: Stale Signoff Indicator

**GIVEN** a Signoff for begrotingsboek v1.0.0 (hash `abc123…`)
**WHEN** the Deliverable progresses to v1.1.0 (typo fix)
**THEN** the existing Signoff remains attached to v1.0.0
**AND** the new version v1.1.0 starts unsigned
**AND** the Deliverable's state transitions expose `signoff-stale` indicator:
```json
{
  "id": "deliverable-id",
  "currentVersionRef": "v1.1.0",
  "currentVersionState": "unsigned",
  "previousVersionRef": "v1.0.0",
  "previousVersionSignoffs": [
    {
      "signoffRef": "signoff-uuid",
      "signerRef": "u-bob-vandenberg",
      "signerRole": "wethouder-financien",
      "signedAt": "2026-11-12T10:45:00Z"
    }
  ],
  "indicators": [
    {
      "type": "signoff-stale",
      "severity": "warning",
      "message": "Current version (1.1.0) is not signed. Previous version (1.0.0) was signed by wethouder-financien on 2026-11-12."
    }
  ]
}
```

### REQ-004-004: Verification Endpoint

**GIVEN** any Signoff record with id=`signoff-uuid`
**WHEN** an external party (accountant, toezichthouder) calls GET `/api/signoffs/{id}/verify`
**THEN** the system returns:
```json
{
  "signoffId": "signoff-uuid",
  "signerRole": "wethouder-financien",
  "signedAt": "2026-11-12T10:45:00Z",
  "signatureMethod": "digid-sign",
  "deliverableRef": "deliverable-id",
  "versionNumber": "1.0.0",
  "contentHashSigned": "sha256:abc123def456...",
  "contentHashCurrent": "sha256:abc123def456...",
  "valid": true,
  "validityReason": "Signature matches current version content hash. Version is immutable."
}
```

**AND** if signature is revoked:
```json
{
  "valid": false,
  "revokedAt": "2026-11-15T14:30:00Z",
  "revokedBy": "u-bob-vandenberg",
  "revokedReason": "incorrect data detected in appendix B"
}
```

---

## REQ-005: Linkage to Begroting and Werkelijke Uitgaven

Every Deliverable that contains financial figures SHALL maintain explicit refs to the begroting-regels and grootboek-mutaties from which those figures are derived; the system SHALL recompute the totals on demand and SHALL warn when the deliverable's displayed totals diverge from the live source totals by more than a configurable tolerance (default 0.01 EUR).

### REQ-005-001: Recompute on Demand

**GIVEN** a BERAP Deliverable with versionRef=v0.3.0
**AND** linkedFinancials containing:
- 412 begroting-regel references
- 2,847 grootboek-mutatie references

**WHEN** a user calls POST `/api/deliverables/{deliverableId}/recompute-financials`
**THEN** the system:
1. Fetches all referenced begroting-regels and recomputes per-programma totals
2. Fetches all referenced grootboek-mutaties and recomputes actuals per-taakveld
3. Returns a financial summary:
```json
{
  "deliverableId": "deliverable-id",
  "versionRef": "v0.3.0",
  "recomputedAt": "2026-11-20T15:30:00Z",
  "byProgramma": [
    {
      "programmaRef": "prog-001",
      "programmaName": "Onderwijs",
      "begroting": {"amount": 5250000.00, "currency": "EUR"},
      "actuals": {"amount": 5249875.50, "currency": "EUR"},
      "delta": {"amount": -124.50, "currency": "EUR"},
      "deltaPercent": -0.0024
    },
    {
      "programmaRef": "prog-002",
      "programmaName": "Infrastructuur",
      "begroting": {"amount": 7200000.00, "currency": "EUR"},
      "actuals": {"amount": 7199999.50, "currency": "EUR"},
      "delta": {"amount": -0.50, "currency": "EUR"},
      "deltaPercent": -0.000007
    }
  ],
  "totalBegroting": {"amount": 12450000.00, "currency": "EUR"},
  "totalActuals": {"amount": 12449875.00, "currency": "EUR"},
  "totalDelta": {"amount": -125.00, "currency": "EUR"},
  "tolerance": {"amount": 0.01, "currency": "EUR"},
  "outOfSync": false
}
```

### REQ-005-002: Out-of-Sync Detection

**GIVEN** a Deliverable with a stored programma-total of EUR 12,450,000
**AND** a live recompute returns EUR 12,450,123.45
**AND** tolerance is default 0.01 EUR
**WHEN** delta (123.45) exceeds tolerance
**THEN** the system:
1. Marks the Deliverable with `financials-out-of-sync = true`
2. Blocks any transition to `definitief` state:
```json
{
  "error": "TRANSITION_BLOCKED",
  "reason": "financials-out-of-sync",
  "message": "Deliverable totals diverge from source data by EUR 123.45. Recompute and reconcile before marking definitief."
}
```

**UNLESS** an explicit waiver is supplied:
```json
{
  "waiverReason": "Difference due to timing of Q4 accruals; approved by concerncontroller",
  "waiverApproverRef": "u-carol",
  "waiverApproverRole": "concerncontroller"
}
```

**THEN** transition is allowed and waiver is recorded in auditLog

### REQ-005-003: Frozen Raads-Vastgesteld Totals

**GIVEN** a Deliverable in state `definitief` (raads-vastgesteld) with linkedFinancials totals
**AND** underlying grootboek-mutaties subsequently change (e.g., Q4 correction entry)
**WHEN** the recompute endpoint is called
**THEN** the deliverable's stored totals do NOT change
**BUT** the system creates a discrepancy record:
```json
{
  "discrepancyId": "disc-uuid",
  "deliverableRef": "deliverable-id",
  "deliverableState": "definitief",
  "storedTotal": {"amount": 12450000.00, "currency": "EUR"},
  "liveTotal": {"amount": 12450125.00, "currency": "EUR"},
  "delta": {"amount": 125.00, "currency": "EUR"},
  "discoveredAt": "2026-12-10T10:00:00Z",
  "reason": "Post-raads-vaststelling correction entries"
}
```

**AND** this discrepancy is visible in the next BERAP as a contextual note

---

## REQ-006: Deelnemer Assignment and Role-Based Visibility

The system SHALL allow assignment of named deelnemers (users) to each Stage with explicit roles (trekker, mede-trekker, reviewer, signoff-holder, viewer) and SHALL restrict visibility of unfinalised deliverables to assigned deelnemers + organisation-wide finance roles.

### REQ-006-001: Deelnemer Access Control

**GIVEN** a Stage with deelnemers:
```json
{
  "deelnemers": [
    {"userRef": "u-alice", "role": "trekker"},
    {"userRef": "u-bob", "role": "reviewer"},
    {"userRef": "u-claire", "role": "signoff-holder", "signerRole": "portefeuillehouder"}
  ]
}
```

**WHEN** user `u-dave` (not a deelnemer, no organisation-wide finance role) calls GET `/api/stages/{stageId}`
**THEN** the system returns HTTP 403:
```json
{
  "error": "FORBIDDEN",
  "reason": "not-a-deelnemer",
  "message": "You are not assigned to this stage. Contact the stage trekker (u-alice) to be added.",
  "deelnemerRequestUrl": "/api/stages/{stageId}/request-access"
}
```

### REQ-006-002: Viewer Role in Draft Deliverables

**GIVEN** a Deliverable in state `concept` (draft, not yet reviewed)
**AND** the parent Stage has deelnemers including `u-eve` with role=viewer

**WHEN** user `u-eve` calls GET `/api/deliverables/{deliverableId}`
**THEN** the system returns the deliverable with a warning banner:
```json
{
  "id": "deliverable-id",
  "state": "concept",
  "currentVersionRef": "v0.1.0",
  "warningBanner": {
    "type": "draft",
    "severity": "info",
    "message": "This is a draft deliverable. Content may change during review."
  },
  "content": { ... }
}
```

### REQ-006-003: Publicised Deliverables

**GIVEN** a Deliverable in state `definitief` (raads-vastgesteld)
**WHEN** any authenticated organisation user (with or without deelnemer role) calls GET `/api/deliverables/{deliverableId}`
**THEN** the system returns it without role check:
```json
{
  "id": "deliverable-id",
  "state": "definitief",
  "visibility": "public-within-organisation",
  "content": { ... }
}
```

**WITH** no access restriction (rechtmatigheid-publicatieplicht per Gemeentewet art. 191)

---

## REQ-007: Escalation Policy Execution

The system SHALL evaluate every active Deadline daily and SHALL emit notifications to the configured escalation chain (T-30, T-14, T-7, T-0, T+7 by default) routing first to the responsible afdeling, then portefeuillehouder, then gemeentesecretaris/algemeen directeur.

### REQ-007-001: T-30 Escalation

**GIVEN** a Stage with `statutoryDeadline = 2027-07-15` and default escalation policy
**WHEN** the daily deadline evaluator runs on 2027-06-15 (T-30)
**THEN** the system:
1. Identifies all Deadlines with daysRemaining=30
2. Creates a `deadline-escalation` event with lead=T-30
3. Routes notification to responsibleAfdeling (e.g., Afdeling Financiën)
4. Sends email:
   - To: afdeling head (responsibleAfdeling.headRef or delegated)
   - Subject: "Deadline 30 dagen: Programmabegroting 2027"
   - Body includes: stage name, deadline date, deliverable status, next required action
5. Records notification in Deadline.escalationPolicy.escalations[0].notificationSent=true

### REQ-007-002: T-7 Escalation to Leadership

**GIVEN** a Stage that remains in status `in-voorbereiding` on 2027-07-08 (T-7 days before deadline)
**WHEN** the evaluator runs
**THEN** the system escalates to:
1. portefeuillehouder (wethouder-financien)
2. concerncontroller
3. Sends email to both with subject: "URGENT: 7 dagen tot deadline Programmabegroting 2027"
4. Includes action items: current blockers, required signoffs, gate failures

### REQ-007-003: T+7 Overdue Escalation

**GIVEN** a Stage with `statutoryDeadline = 2027-07-15` that is still in-voorbereiding or in-behandeling on 2027-07-22 (T+7)
**WHEN** the evaluator runs
**THEN** the system:
1. Sets Deadline.currentStatus=overdue
2. Escalates to gemeentesecretaris/algemeen directeur
3. Creates a `deadline-breached` event that feeds into rechtmatigheid-rapportage
4. Sends email with severity=escalated
5. Optionally creates a task/reminder in the organizational calendar

### REQ-007-004: Waived Deadlines

**GIVEN** a Deadline with `currentStatus = waived` and waiverReason set
**WHEN** the evaluator runs
**THEN** no notifications are emitted
**AND** the deadline does not contribute to rechtmatigheid-rapportage breaches
**BUT** the waiver is visible in audit logs and the stage's public history

---

## REQ-008: Sector-Specific Cyclus Templates

The system SHALL ship three reference CyclusTemplates (gemeente, provincie, waterschap) reflecting current BBV/Waterschapsbesluit obligations and SHALL allow organisations to fork and customise these templates while retaining traceability to the source template version.

### REQ-008-001: Default Gemeente Template

**GIVEN** the system is deployed
**WHEN** a new CyclusInstance is created with organisationType=gemeente
**AND** no explicit templateRef is provided
**THEN** the system uses `cyclus-template-gemeente@2026.1` with stages:
1. kadernota (BBV art. 191 lid 1, deadline 15 april)
2. voorjaarsnota (BBV art. 191 lid 3, deadline 1 juni)
3. programmabegroting (BBV art. 191 lid 2, deadline 15 november)
4. berap (quarterly: Q1 mei, Q2 aug, Q3 nov)
5. najaarsnota (BBV art. 195 lid 1, deadline 15 oktober)
6. jaarrekening (Gemw art. 200 lid 2, deadline 15 juli)

### REQ-008-002: Template Forking

**GIVEN** the gemeente Middelburg wants to insert a custom `concernberaad` stage between voorjaarsnota and programmabegroting
**WHEN** they call POST `/api/cyclus-templates` with:
```json
{
  "name": "Gemeente Middelburg 2027 Variant",
  "parentTemplateRef": "cyclus-template-gemeente@2026.1",
  "parentTemplateVersion": "2026.1",
  "customisations": [
    {
      "type": "insert-stage",
      "stageType": "concernberaad",
      "insertAfter": "voorjaarsnota",
      "plannedDurationDays": 14,
      "reason": "organisational custom: college-wide concern review"
    }
  ]
}
```

**THEN** the system:
1. Creates new CyclusTemplate with parentTemplateRef="cyclus-template-gemeente@2026.1"
2. Stores structured diff in customisations
3. Validates that stage type exists in stageType enum
4. Returns template with id="cyclus-template-middelburg-2027-variant@1.0"

### REQ-008-003: Parent Template Update Notification

**GIVEN** a forked template (Gemeente Middelburg variant) based on `cyclus-template-gemeente@2026.1`
**WHEN** the parent template publishes version 2027.1 with a new statutory deadline for jaarrekening
**THEN** the system:
1. Detects the parent version change
2. Emits a `parent-template-updated` event
3. Surfaces a notification to Middelburg admins: "Template cyclus-template-gemeente has been updated to 2027.1"
4. Provides a diff showing the changes (deadline shifts, new stage types, etc.)

### REQ-008-004: Waterschap Template

**GIVEN** a Waterschap CyclusTemplate
**WHEN** a stage `burap` (bestuursrapportage waterschap-specific) is added
**THEN** the system:
1. Accepts stage types that are absent from the gemeente template (e.g., burap, waterkwantiteit-monitoring)
2. Validates against a waterschap-specific stageType enum
3. Does NOT reject as "unknown stage type"
4. Stores separately from gemeente templates (no confusion)

---

## REQ-009: Heropening of Closed Cycle

The system SHALL support `heropening` of a closed CyclusInstance (e.g. for material corrections discovered post-jaarrekening) with full audit trail and automatic creation of a `heropende-jaarrekening` deliverable that supersedes but does not destroy the original.

### REQ-009-001: Heropening Initiation

**GIVEN** a CyclusInstance for boekjaar 2025 in state `afgerond` (jaarrekening was signed off and published)
**AND** a material error is discovered in the subsidie-verantwoording

**WHEN** a user with role `concerncontroller` POSTs to `/api/cyclus-instances/{cyclusId}/reopen` with:
```json
{
  "reason": "heropening",
  "motivation": "materiële fout in subsidieafrekening artikel 4 - EUR 45,000 overstatement",
  "referencedError": "fout-uuid-001",
  "affectedDeliverables": ["jaarrekening-uuid"],
  "plannedResolutionDate": "2027-03-15"
}
```

**THEN** the system:
1. Sets CyclusInstance.status = heropend
2. Creates a new Stage `heropening-jaarrekening` with:
   - stageType = heropening-jaarrekening
   - sequenceNumber = 7 (after original jaarrekening)
   - status = in-voorbereiding
   - responsibleAfdeling = same as original
3. Creates a new Deliverable `heropende-jaarrekening` linked to this stage
4. Sends notification to toezichthouder with heropening motivation
5. Records audit log entry with full details

### REQ-009-002: Supersession Without Destruction

**GIVEN** a heropende cycle is itself closed
**AND** the resulting `heropende-jaarrekening v1.0.0` is signed off

**WHEN** the system finalises the heropening
**THEN** it:
1. Sets original `jaarrekening v1.0.0` with `superseded-by = heropende-jaarrekening-v1.0.0`
2. Marks original as `state = definitief` still (immutable)
3. Returns in audit trail: original → superseded chain
4. Makes both versions accessible but clearly marks the supersession relationship

### REQ-009-003: Audit Trail for Supervisors

**GIVEN** a heropening event for CyclusInstance
**WHEN** the toezichthouder calls the audit API at `/api/cyclus-instances/{cyclusId}/audit-trail`
**THEN** the system returns the full chronology:
```json
{
  "cyclusId": "cyclusinstance-id",
  "boekjaar": 2025,
  "events": [
    {
      "eventType": "cyclus-closed",
      "closedAt": "2026-01-20",
      "closedBy": "u-carol",
      "deliverables": ["jaarrekening-v1.0.0"]
    },
    {
      "eventType": "heropening-initiated",
      "initiatedAt": "2027-02-15",
      "initiatedBy": "u-carol",
      "reason": "materiële fout in subsidieafrekening artikel 4 - EUR 45,000 overstatement",
      "affectedDeliverables": ["jaarrekening-v1.0.0"]
    },
    {
      "eventType": "heropening-jaarrekening-created",
      "createdAt": "2027-02-15",
      "stageRef": "stage-heropening-jaarrekening"
    },
    {
      "eventType": "heropening-closed",
      "closedAt": "2027-03-15",
      "closedBy": "u-carol",
      "newDeliverables": ["heropende-jaarrekening-v1.0.0"],
      "supersessions": [
        {
          "originalRef": "jaarrekening-v1.0.0",
          "supersededByRef": "heropende-jaarrekening-v1.0.0"
        }
      ]
    }
  ]
}
```

---

## REQ-010: Public Publication of Vastgestelde Stukken

The system SHALL automatically publish every Deliverable that transitions to state `definitief` to the organisation's public-facing channel (default: Open Overheid metadata feed at `/api/public/p-en-c/{boekjaar}`) within 14 days, per Wet open overheid (Woo) art. 3.3 lid 2 sub c.

### REQ-010-001: Woo Publication on Vaststelling

**GIVEN** a Deliverable transitioning to `definitief` on 2027-11-10
**AND** the system has a scheduled n8n workflow `woo-publication-scheduler` with lead time = 14 days

**WHEN** the Deliverable.state is set to definitief
**THEN** the system:
1. Emits a `deliverable.published` event
2. The n8n workflow subscribes to this event
3. Schedules publication no later than 2027-11-24 (14 days)
4. Publishes to Open Overheid metadata feed at `/api/public/p-en-c/{boekjaar}` with:
```json
{
  "informationCategory": "p-en-c",
  "documentType": "programmabegroting",
  "boekjaar": 2027,
  "organisationRef": {"code": "GM0371", "name": "Gemeente Zeist"},
  "deliverableRef": "deliverable-id",
  "versionNumber": "2.0.0",
  "contentHash": "sha256:abc123...",
  "publicUrl": "https://openoverheid.nl/p-en-c/GM0371-2027-programmabegroting",
  "publishedAt": "2027-11-20",
  "legalBasis": "Woo art. 3.3 lid 2 sub c"
}
```

### REQ-010-002: Redaction for Bevat Bedrijfsgevoelige Info

**GIVEN** a Deliverable flagged `bevat-bedrijfsgevoelige-info = true`
**AND** transitioning to definitief
**WHEN** the publication scheduler runs
**THEN** the system:
1. Publishes only the metadata stub (no full document content)
2. Marks `redacted = true` in the Woo feed
3. Records a Woo art. 5.1 exception with reason:
```json
{
  "wooExceptionType": "art-5.1-bedrijfsgevoeligheid",
  "reason": "Bevat bedrijfsgevoelige informatie van de zwemacademie 'Het Groene Zwanstaart'",
  "requestAccessUrl": "/api/woo-requests/new?deliverableId=...",
  "redemptionDate": "2029-01-01"
}
```

### REQ-010-003: Version Continuity on Supersession

**GIVEN** a published Deliverable (programmabegroting v2.0.0, definitief)
**AND** it is later superseded by a heropende version (heropende-programmabegroting v1.0.0)

**WHEN** the new version is published
**THEN** the system:
1. Updates the Woo feed with the new version metadata
2. Keeps original version accessible at versioned URL: `https://openoverheid.nl/p-en-c/GM0371-2027-programmabegroting/v2.0.0`
3. Points primary URL to new version: `https://openoverheid.nl/p-en-c/GM0371-2027-programmabegroting` → v1.0.0 (superseded)
4. Includes supersession relationship in both feed entries:
```json
{
  "versionNumber": "2.0.0",
  "superseded": true,
  "supersededBy": {"versionNumber": "1.0.0", "url": "..."},
  "archivedUrl": "https://openoverheid.nl/p-en-c/GM0371-2027-programmabegroting/v2.0.0"
}
```

**WITH** no link rot — original version remains accessible at its versioned URL

---

## Cross-Cutting Requirements

### REQ-AUDIT: Full Audit Trail

Every mutation (stage transition, signoff, version creation, deelnemer assignment, deadline waiver) SHALL be recorded in an immutable auditLog with:
- eventType (enum)
- timestamp (ISO 8601)
- actor (userRef)
- subject (entity reference)
- details (JSON object with before/after state)
- ipAddress
- userAgent

### REQ-PERF: Query Performance

The system SHALL support queries like "all deliverables in state concept for this cyclusInstance" and "all stages with breached deadlines across all organisations" in <500ms (p95) on datasets up to 10,000 organisations and 100,000 stages.

### REQ-SECURITY: Secrets Management

Signature blobs and OTP hashes SHALL be stored encrypted at rest. DigiD credentials and eIDAS keys are never stored; only signed attestations are persisted.

### REQ-INTEGRATION: Event Publishing

All state mutations emit domain events (`deliverable.version-created`, `stage.transitioned`, `deadline.escalated`, etc.) published to an event bus consumable by n8n, external integrators, and internal subscribers (decidesk webhook bridge, docudesk mirroring, rightmatigheid-rapportage event consumer).
