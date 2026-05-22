---
status: ready
architecture: spec-driven
---

# Design: Iv3-Aanlevering CBS

## Architecture Overview

The `iv3-aanlevering-cbs` spec implements a four-layer pipeline:

```
┌─────────────────────────────────────────────────────────────┐
│ Taxonomy Layer (CBS Reference Data)                          │
│ Iv3Taxonomy (versioned annually, loaded via openconnector)  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Mapping Layer (Organisation Configuration)                   │
│ GrootboekMapping (per-org, per-boekjaar, audit-versioned)   │
│ Tussenrekening (optional, for cost-driver allocations)      │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Aggregation Layer (Per-Period Computation)                   │
│ Iv3Aanlevering (status lifecycle: voorbereiding → ingediend) │
│ Iv3Payload (immutable, versioned XML)                       │
│ Iv3ReconciliationCheck (derived, one per check-type)        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Submission Layer (CBS Integration)                           │
│ Iv3SubmissionResponse (CBS-Kredo acknowledgement/rejection) │
│ Iv3AuditEvent (immutable state-transition log)              │
└─────────────────────────────────────────────────────────────┘
```

## Entity Definitions

### Iv3Taxonomy

Versioned reference data set published annually by CBS. Loaded once per boekjaar via openconnector source `cbs-iv3-taxonomy`.

**Fields:**
- `id` — UUID, primary key
- `boekjaar` — e.g., "2027"
- `taxonomyVersion` — CBS publication identifier, e.g., "iv3-2027.1"
- `taakvelden[]` — array of Taakveld records:
  - `code` — e.g., "4.1" (Openbare orde en veiligheid)
  - `naam` — e.g., "Openbare orde en veiligheid"
  - `hoofdfunctie` — parent category
  - `beschrijving` — long description
  - `validFrom`, `validTo` — date range
- `economischeCategorieen[]` — array of EconomischeCategorie records:
  - `code` — e.g., "1.1" (Lonen werknemers)
  - `naam` — e.g., "Lonen werknemers"
  - `debet_credit_aard` — "D" (debet) or "C" (credit)
  - `parent` — hierarchical parent code (nullable)
- `balansposten[]` — array with codes:
  - `ACTIVA_VAST`, `ACTIVA_VLOTTEND`
  - `PASSIVA_VERMOGEN`, `PASSIVA_VOORZIENINGEN`, `PASSIVA_SCHULDEN_VAST`, `PASSIVA_SCHULDEN_VLOTTEND`
- `mutatieKenmerken[]` — array: `raming-bijgesteld`, `werkelijk`, `raming-oorspronkelijk`
- `gemeenteCodes[]` — array of gemeente codes with validFrom/validTo for herindelingen
- `createdAt`, `publishedBy` — audit fields

### GrootboekMapping

Per-organisation, per-boekjaar configuration of how each grootboek-rekening maps to Iv3-elements.

**Fields:**
- `id` — UUID
- `organisationRef` — foreign key to organisation record
- `boekjaar` — "2027"
- `grootboekRekeningRef` — e.g., "411000" (salariskosten)
- `taakveldCode` — target Iv3-taakveld, e.g., "1.5" (Onderwijs)
- `economischeCategorieCode` — target economic category, e.g., "1.1" (Lonen)
- `balanspostCode` — nullable for resultaat-rekeningen, required for balance-sheet items
- `splitMethod` — one of: `full`, `percentage`, `driver`
- `splitDetails` — JSON:
  - If `full`: `{}`
  - If `percentage`: `[{target: "1.5", pct: 75}, {target: "1.6", pct: 25}]`
  - If `driver`: `{driverRef: "urenregistratie-driver", formula: "hours_by_taakveld / total_hours"}`
- `effectiveFrom`, `effectiveTo` — date range (e.g., "2027-01-01" to null)
- `mappingRationale` — free text, e.g., "Shared staff across education and welfare; driver is urenregistratie"
- `mappedBy`, `reviewedBy` — actor references
- `reviewedAt` — timestamp
- `version`, `createdAt`, `updatedAt` — audit fields

### Tussenrekening

Optional intermediate aggregation layer for organisations using cost drivers.

**Fields:**
- `id` — UUID
- `organisationRef` — foreign key
- `boekjaar` — "2027"
- `code` — e.g., "TRK-411-SPLIT" (shared salariskosten)
- `naam` — "Gedeelde salariskosten verdeeld via urenregistratie"
- `aggregationFormula` — DSL expression, e.g.:
  ```
  GB411000 * (hours_taakveld_1_5 / total_hours)
  ```
- `outputTargets[]` — array mapping this tussenrekening to Iv3-elements:
  - `{taakveld: "1.5", economischeCategorie: "1.1", percentage: 75}`
  - `{taakveld: "1.6", economischeCategorie: "1.1", percentage: 25}`
- `createdAt`, `updatedAt` — audit fields

### Iv3Aanlevering

One record per (boekjaar, periode, organisatie). Tracks the state and metadata of an aanlevering from creation through submission and potential corrections.

**Fields:**
- `id` — UUID
- `boekjaar` — "2027"
- `periode` — one of: "Q1", "Q2", "Q3", "Q4", "Y", "correctie-1", "correctie-2", etc.
- `organisationRef` — foreign key
- `cbsCode` — gemeentecode, e.g., "0363" (Amsterdam)
- `status` — one of:
  - `in-voorbereiding` — draft, under review
  - `gevalideerd` — passed all local checks, ready to submit
  - `ingediend` — submitted to CBS-Kredo, awaiting response
  - `geaccepteerd` — CBS accepted
  - `teruggewezen` — CBS rejected
  - `gecorrigeerd` — superseded by correctie-aanlevering
- `payloadVersionRef` — foreign key to Iv3Payload (nullable until payload generated)
- `submittedAt` — timestamp when POST to Kredo occurred (nullable)
- `cbsResponseRef` — foreign key to Iv3SubmissionResponse (nullable until response received)
- `taxonomyVersionUsed` — e.g., "iv3-2027.1" (frozen at submission time)
- `mappingSnapshotRef` — frozen view of GrootboekMapping as it stood at submission (blob or jsonb)
- `correctsRef` — if periode starts with "correctie-", foreign key to the original Iv3Aanlevering (nullable)
- `createdAt`, `createdBy`, `updatedAt` — audit fields

### Iv3Payload

Immutable serialised aanlevering payload. One version can be created, validated, and submitted multiple times if earlier submission fails; subsequent versions are created only if underlying data changes.

**Fields:**
- `id` — UUID
- `aanleveringRef` — foreign key to Iv3Aanlevering
- `versionNumber` — starts at 1, increments if data changes post-validation
- `format` — CBS XSD version, e.g., "eda-xml-v4.0" (for boekjaar 2027)
- `xmlBlob` — full XML document (large binary field)
- `contentHash` — SHA256(xmlBlob), used to detect downstream mutations
- `validationReport` — JSON array of validation results:
  ```json
  [
    {
      "checkId": "BAL_001_BALANSTOTAAL_NIET_SLUITEND",
      "severity": "blocker",
      "result": "pass",
      "details": "Activa-passiva delta: EUR 0.00"
    }
  ]
  ```
- `createdAt`, `createdBy` — audit fields
- `readonly` — once created, immutable

### Iv3ReconciliationCheck

Derived register; one row per (aanlevering, checkType) capturing the result of structural and numeric validation.

**Fields:**
- `id` — UUID
- `aanleveringRef` — foreign key to Iv3Aanlevering
- `checkType` — one of:
  - `balanstotaal-sluitend` — Activa balanstotaal == Passiva balanstotaal
  - `baten-lasten-saldi-consistent` — baten - lasten + mutaties = saldo
  - `deelnemingen-apart` — taakveld 0.5 not bundled with other categories
  - `reserves-mutaties-consistent` — opening reserves + mutaties = closing reserves
  - `y-aanlevering-matches-vastgestelde-jaarrekening` — Y total matches within tolerance
  - `quartaal-mutaties-cumulatief` — Q2 ⊇ Q1, Q3 ⊇ Q1+Q2, Q4+Y reconciliation
  - `gemeentecode-actueel` — gemeente code valid for the periode
  - `taxonomy-current-jaar` — taxonomy matches boekjaar
- `result` — one of: `pass`, `warn`, `fail`
- `details` — structured JSON describing the violation (if any)
- `tolerance` — numeric tolerance used (e.g., EUR 1.00 for cumulatief checks)
- `runAt` — timestamp of last evaluation

### Iv3SubmissionResponse

Capture of the CBS-Kredo response to a submission.

**Fields:**
- `id` — UUID
- `payloadRef` — foreign key to Iv3Payload
- `submissionId` — CBS-issued identifier, e.g., "KR-2027-000001-ABC"
- `acknowledgedAt` — timestamp when response received
- `responseCode` — one of: `accepted`, `rejected-validation`, `rejected-format`, `partial`
- `responseDetails` — JSON with CBS's response message
- `validationErrors[]` — array of CBS error codes with affected taakveld/rekening references
- `createdAt` — audit field

### Iv3AuditEvent

Immutable append-only log of every state transition, validation run, payload generation, and submission action.

**Fields:**
- `id` — UUID
- `aanleveringRef` — foreign key to Iv3Aanlevering
- `eventType` — one of:
  - `aanlevering-created`
  - `status-transitioned`
  - `payload-generated`
  - `validation-run`
  - `submitted-to-kredo`
  - `response-received`
  - `correctie-created`
  - `audit-queried`
- `actorId` — user reference or service principal
- `timestamp` — ISO 8601, immutable
- `priorState`, `newState` — JSON snapshots (for status transitions)
- `payloadHash` — SHA256 hash of the payload (if applicable)
- `ipAddress` — source IP (if applicable)
- `reason` — free text justification (e.g., "Material error in taakveld 4.1; recalculated from GL 411000")
- `readonly` — immutable after creation

---

## Seed Data

### Example: Gemeente Amsterdam 2027 Q1

#### Iv3Taxonomy (Excerpt)

```json
{
  "boekjaar": "2027",
  "taxonomyVersion": "iv3-2027.1",
  "taakvelden": [
    {
      "code": "0.1",
      "naam": "Openbare orde en veiligheid",
      "validFrom": "2027-01-01",
      "validTo": null
    },
    {
      "code": "1.5",
      "naam": "Onderwijs",
      "validFrom": "2027-01-01",
      "validTo": null
    },
    {
      "code": "6.3",
      "naam": "Waterhuishouding",
      "validFrom": "2027-01-01",
      "validTo": null
    }
  ],
  "economischeCategorieen": [
    {
      "code": "1.1",
      "naam": "Lonen werknemers",
      "debet_credit_aard": "D"
    },
    {
      "code": "2.1",
      "naam": "Inhuur diensten",
      "debet_credit_aard": "D"
    }
  ],
  "gemeenteCodes": [
    {
      "code": "0363",
      "naam": "Amsterdam",
      "validFrom": "2000-01-01",
      "validTo": null
    }
  ]
}
```

#### GrootboekMapping (Excerpt)

```json
[
  {
    "organisationRef": "gemeente-amsterdam",
    "boekjaar": "2027",
    "grootboekRekeningRef": "411000",
    "taakveldCode": "1.5",
    "economischeCategorieCode": "1.1",
    "balanspostCode": null,
    "splitMethod": "driver",
    "splitDetails": {
      "driverRef": "urenregistratie-driver-2027",
      "formula": "hours_by_taakveld / total_hours"
    },
    "effectiveFrom": "2027-01-01",
    "effectiveTo": null,
    "mappingRationale": "Gedeelde salariskosten verdeeld op basis van uurrooster per taakveld",
    "mappedBy": "iv3-coordinator@amsterdam.nl",
    "reviewedBy": "hoofd-financien@amsterdam.nl",
    "reviewedAt": "2027-03-15T14:30:00Z"
  },
  {
    "organisationRef": "gemeente-amsterdam",
    "boekjaar": "2027",
    "grootboekRekeningRef": "421000",
    "taakveldCode": "1.5",
    "economischeCategorieCode": "2.1",
    "balanspostCode": null,
    "splitMethod": "percentage",
    "splitDetails": [
      {
        "target": "1.5",
        "pct": 60
      },
      {
        "target": "0.1",
        "pct": 40
      }
    ],
    "effectiveFrom": "2027-01-01",
    "effectiveTo": null,
    "mappingRationale": "Inhuurkosten voor onderwijsinstellingen (60%) en politie (40%)",
    "mappedBy": "iv3-coordinator@amsterdam.nl",
    "reviewedBy": "hoofd-financien@amsterdam.nl",
    "reviewedAt": "2027-03-15T14:30:00Z"
  }
]
```

#### Iv3Aanlevering (Excerpt)

```json
{
  "id": "aanlev-2027-q1-ams-001",
  "boekjaar": "2027",
  "periode": "Q1",
  "organisationRef": "gemeente-amsterdam",
  "cbsCode": "0363",
  "status": "in-voorbereiding",
  "payloadVersionRef": null,
  "submittedAt": null,
  "cbsResponseRef": null,
  "taxonomyVersionUsed": null,
  "mappingSnapshotRef": null,
  "correctsRef": null,
  "createdAt": "2027-04-01T09:00:00Z",
  "createdBy": "iv3-coordinator@amsterdam.nl"
}
```

#### Iv3Payload (Excerpt)

```json
{
  "id": "payload-2027-q1-ams-001-v1",
  "aanleveringRef": "aanlev-2027-q1-ams-001",
  "versionNumber": 1,
  "format": "eda-xml-v4.0",
  "xmlBlob": "<?xml version=\"1.0\" encoding=\"UTF-8\"?><CBSData>...</CBSData>",
  "contentHash": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "validationReport": [
    {
      "checkId": "BAL_001_BALANSTOTAAL_NIET_SLUITEND",
      "severity": "blocker",
      "result": "pass",
      "details": "Activa balanstotaal EUR 845,120,503.22 == Passiva balanstotaal EUR 845,120,503.22"
    },
    {
      "checkId": "STR_014_DEELNEMINGEN_APART_VEREIST",
      "severity": "blocker",
      "result": "pass",
      "details": "Taakveld 0.5 (Deelnemingen) appears in separate row; no bundling with category 1.5"
    }
  ],
  "createdAt": "2027-04-10T16:45:00Z",
  "createdBy": "aggregation-engine-v1"
}
```

#### Iv3ReconciliationCheck (Excerpt)

```json
[
  {
    "id": "check-2027-q1-ams-bal-001",
    "aanleveringRef": "aanlev-2027-q1-ams-001",
    "checkType": "balanstotaal-sluitend",
    "result": "pass",
    "details": "Activa EUR 845,120,503.22; Passiva EUR 845,120,503.22; delta EUR 0.00",
    "tolerance": 0.01,
    "runAt": "2027-04-10T16:45:00Z"
  },
  {
    "id": "check-2027-q1-ams-cum-001",
    "aanleveringRef": "aanlev-2027-q1-ams-001",
    "checkType": "quartaal-mutaties-cumulatief",
    "result": "pass",
    "details": "First quarterly aanlevering; cumulative baseline established",
    "tolerance": 100.00,
    "runAt": "2027-04-10T16:45:00Z"
  }
]
```

#### Iv3SubmissionResponse (Excerpt)

```json
{
  "id": "response-2027-q1-ams-001",
  "payloadRef": "payload-2027-q1-ams-001-v1",
  "submissionId": "KR-2027-000042-AMS",
  "acknowledgedAt": "2027-04-27T10:15:30Z",
  "responseCode": "accepted",
  "responseDetails": {
    "message": "Aanlevering ontvangen en gevalideerd",
    "processedAt": "2027-04-27T10:15:30Z"
  },
  "validationErrors": [],
  "createdAt": "2027-04-27T10:15:30Z"
}
```

#### Iv3AuditEvent (Excerpt)

```json
[
  {
    "id": "event-2027-q1-ams-001",
    "aanleveringRef": "aanlev-2027-q1-ams-001",
    "eventType": "aanlevering-created",
    "actorId": "iv3-coordinator@amsterdam.nl",
    "timestamp": "2027-04-01T09:00:00Z",
    "reason": "Q1-2027 aanlevering creation initiated",
    "ipAddress": "203.0.113.45"
  },
  {
    "id": "event-2027-q1-ams-002",
    "aanleveringRef": "aanlev-2027-q1-ams-001",
    "eventType": "payload-generated",
    "actorId": "aggregation-engine-v1",
    "timestamp": "2027-04-10T16:45:00Z",
    "priorState": {"status": "in-voorbereiding"},
    "newState": {"status": "in-voorbereiding", "payloadVersionRef": "payload-2027-q1-ams-001-v1"},
    "payloadHash": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
    "reason": "Aggregation complete; payload v1 generated and validated"
  },
  {
    "id": "event-2027-q1-ams-003",
    "aanleveringRef": "aanlev-2027-q1-ams-001",
    "eventType": "submitted-to-kredo",
    "actorId": "hoofd-financien@amsterdam.nl",
    "timestamp": "2027-04-27T10:15:00Z",
    "priorState": {"status": "gevalideerd"},
    "newState": {"status": "ingediend", "submittedAt": "2027-04-27T10:15:00Z"},
    "payloadHash": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
    "reason": "Ready for submission; all local validations passed",
    "ipAddress": "203.0.113.46"
  }
]
```

---

## Data Integrity Constraints

1. **Immutable payloads** — Iv3Payload records, once created, are read-only. Any subsequent change to underlying data triggers a new version.
2. **Audit-versioned mappings** — GrootboekMapping changes are tracked with effective dates. A snapshot is frozen at aanlevering submission time to enable later reconstruction.
3. **Taxonomy versioning** — Only one Iv3Taxonomy per (boekjaar, taxonomyVersion) is permitted. Updates within a year retain the prior version for audit.
4. **One-way status transitions** — Iv3Aanlevering status transitions only: voorbereiding → gevalideerd → ingediend → geaccepteerd | teruggewezen → gecorrigeerd (if correction issued).
5. **Cumulatief enforcement** — Quarterly aanleveringen MUST form a cumulatief chain Q1 ⊆ Q2 ⊆ Q3 ⊆ Q4 ⊆ Y, enforced by reconciliation checks.
6. **Append-only audit log** — Iv3AuditEvent records are immutable and read-only; no deletion or modification permitted.

---

## Integration Points

- **openconnector** — taxonomy ingestion (source: `cbs-iv3-taxonomy`), submission POST (sink: `cbs-kredo-submission`), status updates (sink: `bzk-financieel-toezicht`)
- **bookkeeping-bbv-compliance** — source of grootboek-rekeningen and grootboek-mutaties
- **driver-based-forecasting** — driver formulas used in GrootboekMapping splitMethod=driver
- **pc-cyclus-workflow** — vastgestelde jaarrekening used for Y-aanlevering reconciliation
- **n8n** — orchestrates taxonomy updates, pre-deadline notifications, daily reconciliation runs
- **mydash** — exposes "Iv3-readiness" widget for concerncontrollers
- **docusaurus journeydoc** — ships yearly how-to guide with step-by-step screenshots
- **docudesk** — long-term archival of submitted payloads (Archiefwet retention rules)
