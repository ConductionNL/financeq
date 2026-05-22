# Proposal: P&C-Cyclus Workflow (Planning & Control)

## Executive Summary

The Planning & Control cyclus (P&C-cyclus) is the statutory annual rhythm by which every Dutch decentrale overheid — gemeenten, provincies, waterschappen, and gemeenschappelijke regelingen — plans, authorises, executes, monitors, and accounts for its finances. Currently this cycle is reinvented in Word, Excel, SharePoint, and scattered tools; document versions drift between college-versie, raads-versie, and accountants-versie; portefeuillehouder sign-off is captured in margin comments; and statutory deadlines are tracked in personal agendas.

This proposal makes the P&C-cyclus a first-class workflow inside financeq with explicit stages, transitions, deelnemers, deliverables, validations, handoff rules, and cryptographic sign-off. Every deliverable is versioned and linked to the underlying begroting and werkelijke uitgaven. Statutory deadlines drive automated reminders and escalations. The result is a single source of truth for the financial year, queryable by date, stage, portefeuille, programma, or accountable officer, that satisfies both internal control and external accountability.

## Business Goals

1. **Eliminate version drift** — Replace email and margin comments with a versioned, auditable document chain.
2. **Enforce statutory deadlines** — Automate derivation of deadlines and escalate breaches to the concerncontroller immediately.
3. **Reduce coordinator overhead** — Stage transitions, deelnemer assignments, and signoff workflows are system-driven rather than manual.
4. **Enable audit trail** — Every transition, signoff, and document change is recorded with timestamp, actor, and motivation.
5. **Support heropening** — Allow material corrections post-jaarrekening without destroying original history.
6. **Comply with Woo** — Automatically publish definitief deliverables to the public Woo feed within 14 days.

## Scope

The spec owns the *workflow* — stages, transitions, deelnemers, sign-off, deadlines, versions — not the *content* of begroting or jaarrekening, which lives in `bookkeeping-bbv-compliance` and `programma-begroting`. It does not replace decidesk (raadsbesluit chain) or docudesk (long-term archival); it integrates with both.

## Features

### F-001: Statutory Deadline Derivation

Automatically derive deadlines from stageType and boekjaar using sector-specific calendars (gemeente, provincie, waterschap). Prevent users from setting `plannedEnd` later than the statutory deadline without explicit acknowledged override.

**Demand: 5/5** (mandated by BBV art. 191, Gemeentewet art. 200)

### F-002: Stage Transition Gating

Enforce that a StageTransition can only be marked completed when every gateCriteria validation evaluates true and every requiredSignoffs entry has a non-revoked Signoff against the deliverable's current version.

**Demand: 5/5** (core control requirement)

### F-003: Document Versioning with Content Hashing

Every DeliverableVersion is immutable once created and carries a sha256 contentHash of its canonical serialisation. The system rejects any attempt to mutate an existing version and requires a new version for any change.

**Demand: 5/5** (accountability requirement)

### F-004: Portefeuillehouder Sign-off

Support cryptographic sign-off by named role-holders against a specific DeliverableVersion using DigiD-Sign, qualified eIDAS, or an ad-hoc OTP method. The signature binds to the contentHash of that exact version and is independently verifiable.

**Demand: 5/5** (compliance requirement)

### F-005: Linkage to Financial Data

Every Deliverable containing financial figures maintains explicit refs to the begroting-regels and grootboek-mutaties from which those figures are derived. The system recomputes totals on demand and warns when displayed totals diverge from live source totals.

**Demand: 5/5** (audit requirement)

### F-006: Deelnemer Assignment and Role-Based Visibility

Allow assignment of named deelnemers (users) to each Stage with explicit roles (trekker, mede-trekker, reviewer, signoff-holder, viewer) and restrict visibility of unfinalised deliverables to assigned deelnemers + organisation-wide finance roles.

**Demand: 4/5** (control and usability)

### F-007: Escalation Policy Execution

Evaluate every active Deadline daily and emit notifications to the configured escalation chain (T-30, T-14, T-7, T-0, T+7 by default) routing first to the responsible afdeling, then portefeuillehouder, then gemeentesecretaris.

**Demand: 4/5** (operational requirement)

### F-008: Sector-Specific Cyclus Templates

Ship three reference CyclusTemplates (gemeente, provincie, waterschap) reflecting current BBV/Waterschapsbesluit obligations and allow organisations to fork and customise these templates while retaining traceability to the source template version.

**Demand: 4/5** (operational requirement)

### F-009: Heropening of Closed Cycle

Support heropening of a closed CyclusInstance (e.g. for material corrections discovered post-jaarrekening) with full audit trail and automatic creation of a heropende-jaarrekening deliverable that supersedes but does not destroy the original.

**Demand: 3/5** (occasional, compliance-driven)

### F-010: Public Publication of Vastgestelde Stukken

Automatically publish every Deliverable that transitions to state definitief to the organisation's public-facing channel (Open Overheid metadata feed at `/api/public/p-en-c/{boekjaar}`) within 14 days, per Wet open overheid art. 3.3 lid 2 sub c.

**Demand: 5/5** (legal requirement)

## Success Criteria

- [ ] Every statutory deadline is automatically derived and enforced.
- [ ] 100% of stage transitions are gated by validations and signoffs.
- [ ] Every deliverable version is immutable and carries a verifiable content hash.
- [ ] Cryptographic signoff is supported and independently verifiable.
- [ ] All definitief deliverables are published to Woo feed within 14 days.
- [ ] Organisations can fork and customize cyclus templates.
- [ ] Deadline breaches are escalated to concerncontroller within T-7 days.
- [ ] Full audit trail of every transition, signoff, and version change.

## Stakeholders

- **Concerncontroller / hoofd Financiën** — owns the cycle, signs off on transitions, escalates breached deadlines. Primary daily user.
- **Afdelings-controllers** — produce programma-specific deliverable sections, request peer review, route to portefeuillehouder.
- **Portefeuillehouders (wethouders, gedeputeerden, dagelijks bestuur waterschap)** — review and sign off on deliverables within their portefeuille.
- **Gemeentesecretaris / algemeen directeur / dijkgraaf** — final escalation, gatekeeper of college-versie transitions.
- **College / dagelijks bestuur** — collective signoff on college-versies.
- **Accountant** — read-only access to version history and audit API.
- **Toezichthouder** — read-only access to programmabegroting and jaarrekening.

## Timeline

- **Week 1-2:** Design and schema validation
- **Week 3-6:** Core workflow engine and stage/transition logic
- **Week 7-8:** Signoff and versioning subsystem
- **Week 9-10:** Escalation and notification workflows
- **Week 11-12:** Woo publication and integration with external systems
- **Week 13-14:** Testing, templates, and documentation
