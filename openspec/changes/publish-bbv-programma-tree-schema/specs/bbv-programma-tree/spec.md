# BBV Programma Tree — Canonical Schema Delta

**Spec refs**: financeq ADR-001 §3 (single source of truth), hydra ADR-022 (apps consume OR
abstractions), hydra ADR-031 (schema-declarative business logic)
**Standards**: BBV (Besluit Begroting en Verantwoording), Iv3 taakvelden (CBS)

## ADDED Requirements

### Requirement: Canonical Programma Schema in OpenRegister

financeq MUST publish the BBV-programma-tree as a `programma` schema in the shared
OpenRegister `financeq` register, imported via a Repair step on install/upgrade. Each node
MUST carry `code`, `naam`, `type` (`programma`|`doel`|`activiteit`), a nullable `parent`
self-relation (UUID), `boekjaar`, and an optional `taakveld` relation. The hierarchy MUST
be parent-linked; no consumer or surface may store a denormalised copy of the tree.

**Feature tier**: MVP

#### Scenario: Register import on clean install

- GIVEN a fresh Nextcloud instance with OpenRegister enabled
- WHEN financeq is installed
- THEN the `financeq` register with the `programma` and `taakveld` schemas MUST exist in
  OpenRegister
- AND zero `programma` objects MUST exist (no gemeente-specific seed before setup)

#### Scenario: Rename propagates to all consumers

- GIVEN a `programma` node "Sociaal Domein" referenced by a planix project and a Rapportage
  view
- WHEN the node's `naam` is updated via the Begroting surface
- THEN any subsequent OR read by planix, Rapportage, or launchpad MUST return the new name
- AND no consumer-side copy MUST require a separate update

### Requirement: Single-Writer Access Contract

Only financeq (Begroting role) MUST be able to create, update, or delete `programma`
objects, enforced via OpenRegister schema RBAC. Consumers (planix, Rapportage surfaces,
launchpad) MUST read the tree through standard OpenRegister object queries or aggregations;
financeq MUST NOT expose a bespoke tree endpoint and consumers MUST NOT write to the schema.

**Feature tier**: MVP

#### Scenario: Consumer write is rejected

- GIVEN an authenticated user whose roles grant planix access but not the financeq
  begroting role
- WHEN they attempt to update a `programma` object via the OR API
- THEN OpenRegister MUST reject the write with a permission error

#### Scenario: Consumer read uses plain OR queries

- GIVEN planix needs the programma→doel ladder for a project
- WHEN it fetches the tree
- THEN it MUST use a standard OpenRegister object query on `financeq`/`programma`
- AND MUST NOT call any financeq-specific endpoint

### Requirement: Taakveld Lookup as Reference Configuration

financeq MUST ship the Iv3 taakvelden lookup as a `taakveld` schema with neutral
CBS reference data seeded at install time. Taakvelden MUST be manageable only under the
Beheer surface (ADR-001 §5); aanlevering screens MUST NOT allow editing taakvelden.

**Feature tier**: MVP

#### Scenario: Taakvelden present after install

- GIVEN a fresh financeq install
- WHEN an admin opens Beheer > Taakvelden
- THEN the CBS taakvelden reference list MUST be present and editable there
- AND no other surface MUST offer taakveld editing
