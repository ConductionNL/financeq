# Design: P&C-Cyclus Workflow

## Architecture

The P&C-Cyclus Workflow is a state-machine-driven system built around eight core registers and a directed graph of stage transitions. Each CyclusInstance is a stateful entity representing a single organisation-boekjaar pair; stages are nodes with explicit entry/exit conditions; StageTransitions are directed edges with gating logic (validations, signoffs, role checks).

### Register Structure

All registers are JSON Schema under `financeq/openspec/specs/pc-cyclus-workflow/schemas/` and registered in openregister for auditing, versioning, search, and access control.

```
CyclusInstance
├── CyclusTemplate
├── Stages[]
│   ├── StageTransition[]
│   ├── Deelnemers[]
│   └── Deadlines[]
├── Deliverables[]
│   ├── DeliverableVersion[]
│   │   └── Signoffs[]
│   └── linkedFinancials
└── auditLog[]
```

### Entity Definitions

#### CyclusInstance

One record per organisation-year (e.g., "Gemeente Zeist 2027").

```json
{
  "id": "uuid",
  "boekjaar": 2027,
  "organisationRef": {
    "type": "organisationRef",
    "code": "GM0371",
    "name": "Gemeente Zeist"
  },
  "status": "planning|actief|afgerond|heropend",
  "kickoffDate": "2026-09-01",
  "closureDate": null,
  "templateRef": "cyclus-template-gemeente@2026.1",
  "customisations": {
    "stages": [
      {
        "stageType": "concernberaad",
        "insertAfter": "voorjaarsnota",
        "reason": "organisational custom"
      }
    ]
  },
  "currentStageRefs": [
    "stage-uuid-berap-q1",
    "stage-uuid-voorjaarsnota-in-raad"
  ],
  "createdAt": "2026-08-15T09:30:00Z",
  "createdBy": {"type": "userRef", "id": "u-alice"},
  "updatedAt": "2026-11-20T14:45:00Z"
}
```

#### CyclusTemplate

Reusable definition of stages, transitions, and roles for a sector.

```json
{
  "id": "uuid",
  "name": "Gemeente Standaard 2026",
  "templateType": "gemeente|provincie|waterschap",
  "version": "2026.1",
  "parentTemplateRef": null,
  "parentTemplateVersion": null,
  "stages": [
    {
      "stageType": "kadernota",
      "sequenceNumber": 1,
      "statutoryDeadlineRule": "BBV-art-191-lid-1",
      "deliverables": [
        "kadernota-document"
      ],
      "requiredRoles": ["portefeuillehouder-financien", "concerncontroller"]
    },
    {
      "stageType": "voorjaarsnota",
      "sequenceNumber": 2,
      "statutoryDeadlineRule": "BBV-art-191-lid-3",
      "deliverables": ["voorjaarsnota-document"]
    }
  ],
  "transitions": [
    {
      "from": "kadernota",
      "to": "voorjaarsnota",
      "transitionType": "sequential",
      "gateCriteria": ["financiele-saldi-sluitend"],
      "requiredSignoffs": ["portefeuillehouder-financien"]
    }
  ],
  "createdAt": "2026-01-01T00:00:00Z",
  "createdBy": {"type": "userRef", "id": "u-system"},
  "legalBasis": "BBV, Gemeentewet art. 186-213"
}
```

#### Stage

One record per stage occurrence within a CyclusInstance.

```json
{
  "id": "uuid",
  "cyclusInstanceRef": "cyclusinstance-uuid",
  "stageType": "programmabegroting",
  "sequenceNumber": 3,
  "plannedStart": "2026-10-01",
  "plannedEnd": "2026-11-01",
  "statutoryDeadline": "2026-11-15",
  "statutoryDeadlineReason": "BBV art. 191 lid 2 - 15 november van het jaar voorafgaand",
  "actualStart": "2026-10-03",
  "actualEnd": null,
  "status": "in-voorbereiding",
  "responsibleAfdeling": {
    "type": "afdelingRef",
    "code": "FIN",
    "name": "Afdeling Financiën"
  },
  "portefeuillehouderRef": {
    "type": "userRef",
    "id": "u-bob",
    "name": "Bob van den Berg",
    "role": "wethouder financien"
  },
  "accountableOfficerRef": {
    "type": "userRef",
    "id": "u-carol",
    "name": "Carol Jansen",
    "role": "concerncontroller"
  },
  "deelnemers": [
    {
      "userRef": {"type": "userRef", "id": "u-dave"},
      "role": "trekker",
      "assignedAt": "2026-09-20",
      "assignedBy": {"type": "userRef", "id": "u-carol"}
    },
    {
      "userRef": {"type": "userRef", "id": "u-eve"},
      "role": "reviewer",
      "assignedAt": "2026-09-20"
    }
  ],
  "createdAt": "2026-09-15T10:00:00Z"
}
```

#### StageTransition

Directed edge between stages.

```json
{
  "id": "uuid",
  "sourceStageRef": "stage-uuid-voorjaarsnota",
  "targetStageRef": "stage-uuid-programmabegroting",
  "transitionType": "sequential|parallel|conditional",
  "gateCriteria": [
    {
      "criterion": "financiele-saldi-sluitend",
      "currentStatus": "pass",
      "evaluatedAt": "2026-11-01T15:30:00Z"
    },
    {
      "criterion": "paragraaf-weerstandsvermogen-aanwezig",
      "currentStatus": "pass",
      "evaluatedAt": "2026-11-01T15:30:00Z"
    }
  ],
  "requiredSignoffs": [
    {
      "role": "portefeuillehouder-financien",
      "status": "signed",
      "signoffRef": "signoff-uuid-1",
      "signedAt": "2026-11-05T09:00:00Z"
    },
    {
      "role": "concerncontroller",
      "status": "pending",
      "signoffRef": null
    }
  ],
  "status": "gate-not-met",
  "actualTransitionDate": null,
  "actualTransitionBy": null,
  "createdAt": "2026-11-01T10:00:00Z"
}
```

#### Deliverable

Typed document produced within a stage.

```json
{
  "id": "uuid",
  "stageRef": "stage-uuid-programmabegroting",
  "deliverableType": "programmabegroting-document",
  "title": "Programmabegroting 2027",
  "currentVersionRef": "deliverable-version-uuid-v1-0-0",
  "versionHistory": [
    "deliverable-version-uuid-v0-1-0",
    "deliverable-version-uuid-v0-2-0",
    "deliverable-version-uuid-v1-0-0"
  ],
  "linkedFinancials": {
    "begrotingRegels": [
      {
        "regelRef": "regel-uuid-001",
        "bedrag": {
          "amount": 1250000.00,
          "currency": "EUR",
          "scale": 2
        }
      }
    ],
    "grootboekMutaties": [
      {
        "mutatieRef": "mutatie-uuid-001",
        "bedrag": {
          "amount": 1250000.00,
          "currency": "EUR",
          "scale": 2
        }
      }
    ],
    "totalCalculated": {
      "amount": 12450000.00,
      "currency": "EUR",
      "scale": 2
    }
  },
  "state": "college-versie",
  "financialsOutOfSync": false,
  "supportsWooPublication": true,
  "createdAt": "2026-10-15T08:00:00Z"
}
```

#### DeliverableVersion

Immutable snapshot.

```json
{
  "id": "uuid",
  "deliverableRef": "deliverable-uuid",
  "versionNumber": "1.0.0",
  "versionLabel": "college-akkoord",
  "createdAt": "2026-11-10T14:30:00Z",
  "createdBy": {
    "type": "userRef",
    "id": "u-dave",
    "name": "Dave Hendriks"
  },
  "contentHash": "sha256:abc123def456...",
  "changeNote": "Finalisatie na college-behandeling 10 november",
  "parentVersionRef": "deliverable-version-uuid-v0-2-0",
  "attachments": [
    {
      "filename": "programmabegroting-2027.pdf",
      "contentType": "application/pdf",
      "size": 2458624,
      "hash": "sha256:xyz789...",
      "uploadedAt": "2026-11-10T14:30:00Z"
    }
  ],
  "signoffs": [
    "signoff-uuid-1"
  ]
}
```

#### Signoff

Cryptographic acknowledgement of a DeliverableVersion.

```json
{
  "id": "uuid",
  "signedVersionRef": "deliverable-version-uuid-v1-0-0",
  "signerRef": {
    "type": "userRef",
    "id": "u-bob",
    "name": "Bob van den Berg"
  },
  "signerRole": "portefeuillehouder-financien",
  "signedAt": "2026-11-12T10:45:00Z",
  "signatureMethod": "digid-sign|qualified-eidas|ad-hoc-otp|wet-handtekening",
  "signatureBlob": "-----BEGIN PKCS7-----\nMIIEnAYJKoZIhvcNAQcCoII...\n-----END PKCS7-----",
  "contentHashSigned": "sha256:abc123def456...",
  "revokedAt": null,
  "revokedReason": null,
  "revokedBy": null,
  "auditLog": [
    {
      "timestamp": "2026-11-12T10:45:00Z",
      "event": "signed",
      "method": "digid-sign",
      "ipAddress": "203.0.113.42"
    }
  ]
}
```

#### Deadline

Derived view; one record per Stage holding statutory and internal deadlines.

```json
{
  "id": "uuid",
  "stageRef": "stage-uuid-programmabegroting",
  "deadlineType": "statutoir",
  "dueDate": "2026-11-15",
  "daysRemaining": 14,
  "legalBasis": "BBV art. 191 lid 2",
  "escalationPolicy": {
    "enabled": true,
    "escalations": [
      {
        "lead": "T-30",
        "roles": ["responsibleAfdeling"],
        "notificationSent": false
      },
      {
        "lead": "T-14",
        "roles": ["responsibleAfdeling", "portefeuillehouder"],
        "notificationSent": false
      },
      {
        "lead": "T-7",
        "roles": ["portefeuillehouder", "concerncontroller"],
        "notificationSent": false
      },
      {
        "lead": "T-0",
        "roles": ["concerncontroller", "gemeentesecretaris"],
        "notificationSent": false
      },
      {
        "lead": "T+7",
        "roles": ["gemeentesecretaris"],
        "notificationSent": false,
        "severity": "escalated"
      }
    ]
  },
  "currentStatus": "ok|at-risk|overdue|waived",
  "waivedAt": null,
  "waivedBy": null,
  "waiverReason": null,
  "createdAt": "2026-09-15T10:00:00Z"
}
```

## Seed Data

### CyclusTemplate: Gemeente Standaard 2026

```json
{
  "id": "template-gemeente-2026.1",
  "name": "Gemeente Standaard 2026",
  "templateType": "gemeente",
  "version": "2026.1",
  "parentTemplateRef": null,
  "stages": [
    {
      "stageType": "kadernota",
      "sequenceNumber": 1,
      "statutoryDeadlineRule": "BBV-art-191-lid-1",
      "deadline": "2026-04-15"
    },
    {
      "stageType": "voorjaarsnota",
      "sequenceNumber": 2,
      "statutoryDeadlineRule": "BBV-art-191-lid-3",
      "deadline": "2026-06-01"
    },
    {
      "stageType": "programmabegroting",
      "sequenceNumber": 3,
      "statutoryDeadlineRule": "BBV-art-191-lid-2",
      "deadline": "2026-11-15"
    },
    {
      "stageType": "berap",
      "sequenceNumber": 4,
      "recurringPattern": "quarterly"
    },
    {
      "stageType": "najaarsnota",
      "sequenceNumber": 5,
      "statutoryDeadlineRule": "BBV-art-195-lid-1",
      "deadline": "2026-10-15"
    },
    {
      "stageType": "jaarrekening",
      "sequenceNumber": 6,
      "statutoryDeadlineRule": "Gemw-art-200-lid-2",
      "deadline": "2027-07-15"
    }
  ]
}
```

### CyclusInstance: Gemeente Zeist 2027

```json
{
  "id": "cyclusinstance-gemeente-zeist-2027",
  "boekjaar": 2027,
  "organisationRef": {
    "code": "GM0371",
    "name": "Gemeente Zeist"
  },
  "status": "planning",
  "kickoffDate": "2026-09-01",
  "closureDate": null,
  "templateRef": "template-gemeente-2026.1",
  "customisations": {},
  "currentStageRefs": []
}
```

### Stage: Programmabegroting 2027 (Gemeente Zeist)

```json
{
  "id": "stage-zeist-pb-2027",
  "cyclusInstanceRef": "cyclusinstance-gemeente-zeist-2027",
  "stageType": "programmabegroting",
  "sequenceNumber": 3,
  "plannedStart": "2026-10-01",
  "plannedEnd": "2026-11-15",
  "statutoryDeadline": "2026-11-15",
  "status": "niet-gestart",
  "responsibleAfdeling": {
    "code": "FIN",
    "name": "Afdeling Financiën"
  },
  "portefeuillehouderRef": {
    "id": "u-bob-vandenberg",
    "name": "Bob van den Berg",
    "role": "wethouder financien"
  }
}
```

### Deliverable: Programmabegroting 2027 Document

```json
{
  "id": "deliverable-zeist-pb-2027",
  "stageRef": "stage-zeist-pb-2027",
  "deliverableType": "programmabegroting-document",
  "title": "Programmabegroting 2027 - Gemeente Zeist",
  "currentVersionRef": null,
  "versionHistory": [],
  "state": "niet-gestart",
  "createdAt": "2026-09-01T09:00:00Z"
}
```

## Architectural Decisions

### Immutable DeliverableVersion

Each DeliverableVersion is immutable once created. When content changes, a new version is created rather than modifying the existing version. This ensures signoff integrity and audit trail completeness.

### Content Hash for Verification

Every DeliverableVersion carries a sha256 contentHash of its canonical JSON serialisation (sorted keys, normalised whitespace). This allows external parties (accountants, toezichthouder) to independently verify that the document they're reviewing matches the signed version.

### Stateless Signoff Validation

Signoffs are checked at transition time. If a signoff is later revoked, downstream stages are marked with `upstreamSignoffRevoked` warning but NOT automatically rolled back. This preserves the audit trail and requires explicit human re-evaluation.

### Sector-Specific Templates

The system ships three reference CyclusTemplates (gemeente, provincie, waterschap) derived directly from BBV/Waterschapsbesluit. Organisations can fork these templates and customise stages while retaining parentTemplateRef for traceability.

### n8n Integration for Escalations

Deadline escalations and Woo publication are implemented as n8n workflows triggered by financeq events. This decouples timing logic from the core state machine and allows operational tuning without code changes.

## Integration Points

- **decidesk** — raadsbesluit chain via webhook
- **docudesk** — archival and OAIS AIP on definitief transition
- **bookkeeping-bbv-compliance** — begroting-regels and grootboek-mutaties refs
- **openregister** — all eight registers are OR schemas
- **openconnector** — outbound to toezichthouder and CBS
- **openoverheid** — Woo publication feed
- **n8n** — escalation notifications and publication scheduler
- **mydash** — GraphQL consumption of registers for deadline-risk dashboard
