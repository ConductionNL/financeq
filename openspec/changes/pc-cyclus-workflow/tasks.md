# Implementation Tasks: P&C-Cyclus Workflow

## Phase 1: Core Data Model & Schema (Week 1-2)

- [ ] **TASK-001: Define OpenRegister Schemas**
  - Create JSON Schema files for CyclusInstance, CyclusTemplate, Stage, StageTransition, Deliverable, DeliverableVersion, Signoff, Deadline
  - Location: `financeq/openspec/specs/pc-cyclus-workflow/schemas/`
  - Validate against OpenRegister naming conventions
  - Include examples and cross-references
  - Implementation steps:
    1. Define CyclusInstance schema with enums for status, organisation refs
    2. Define CyclusTemplate schema with sector types (gemeente|provincie|waterschap)
    3. Define Stage schema with stageType enum and deadline derivation rules
    4. Define StageTransition schema with gating logic (gateCriteria, requiredSignoffs)
    5. Define Deliverable and DeliverableVersion schemas with versioning rules
    6. Define Signoff schema with cryptographic fields (signatureBlob, contentHashSigned)
    7. Define Deadline schema with escalation policies
    8. Register all schemas in OpenRegister

- [ ] **TASK-002: Implement Statutory Deadline Derivation Engine (REQ-001)**
  - Create deadline calculator service with sector-specific rules
  - Location: `financeq/src/deadline-calculator/`
  - Implementation steps:
    1. Build decision tree for gemeente deadlines (BBV articles)
    2. Build decision tree for provincie deadlines (Provinciewet articles)
    3. Build decision tree for waterschap deadlines (Waterschapsbesluit articles)
    4. Implement override validation with approver role checks
    5. Add tolerance/waiver logic for acknowledged overrides
    6. Unit tests for each sector and edge cases (leap years, holidays)

- [ ] **TASK-003: Create CyclusTemplate Reference Implementations**
  - Build three default templates: gemeente, provincie, waterschap
  - Location: `financeq/openspec/specs/pc-cyclus-workflow/seed-data/templates/`
  - Implementation steps:
    1. Create gemeente template with 6 stages + BBV deadline rules
    2. Create provincie template with equivalent stages from Provinciewet
    3. Create waterschap template with burap, kwartaalrapportage, and Waterschapsbesluit rules
    4. Seed database with these templates on deployment
    5. Add version tracking (2026.1, 2027.1, etc.)

- [ ] **TASK-004: Set Up Database Migrations**
  - Create migration scripts for CyclusInstance, Stage, Deliverable tables
  - Location: `financeq/migrations/`
  - Implementation steps:
    1. Create table schemas with required indices (cyclusInstanceRef, stageType, status)
    2. Add audit log table with immutable insert-only design
    3. Create versioning tables for DeliverableVersion with parent refs
    4. Add foreign key constraints between registers
    5. Create trigger-based audit log inserts for compliance

## Phase 2: Workflow Engine & Stage Transitions (Week 3-6)

- [ ] **TASK-005: Implement Stage Transition State Machine (REQ-002)**
  - Build transition gating logic with validation checks and signoff requirements
  - Location: `financeq/src/stage-transitions/`
  - Implementation steps:
    1. Create StageTransitionService with pre-flight checks
    2. Implement gateCriteria evaluation (financial-saldi-sluitend, paragraaf checks, etc.)
    3. Implement requiredSignoffs validation (must be non-revoked)
    4. Add atomic transition execution with source stage → target stage updates
    5. Emit stage.transitioned domain events
    6. Handle upstreamSignoffRevoked warning and persist in stage.warnings
    7. Unit tests for sequential, parallel, and conditional transitions

- [ ] **TASK-006: Build Deelnemer Assignment & Role-Based Visibility (REQ-006)**
  - Implement stage deelnemer roster and access control
  - Location: `financeq/src/stage-deelnemers/`, `financeq/src/visibility-control/`
  - Implementation steps:
    1. Create DeelnemerService for assigning users to stages with roles
    2. Define role enum (trekker, mede-trekker, reviewer, signoff-holder, viewer)
    3. Implement VisibilityControl middleware that checks stage deelnemers + org-wide roles
    4. Restrict concept-state deliverables to deelnemers (warn on public deliverables)
    5. Allow definitief deliverables to any authenticated org user
    6. Integrate with Nextcloud user directory for role resolution
    7. API endpoints: POST /api/stages/{id}/deelnemers, PATCH, DELETE

- [ ] **TASK-007: Implement Document Versioning with Content Hashing (REQ-003)**
  - Build immutable version system with sha256 content hashing
  - Location: `financeq/src/deliverable-versioning/`
  - Implementation steps:
    1. Create DeliverableVersionService with immutability enforcement
    2. Implement canonical JSON serialization (sorted keys, normalized whitespace)
    3. Add sha256 content hash computation and storage
    4. Prevent mutations on existing versions (reject PATCH)
    5. Enforce monotonic version numbers (reject backwards versions)
    6. Detect NO_EFFECTIVE_CHANGE and reject version inflation
    7. Store parent version reference for audit trail
    8. Unit tests for version chain validation

- [ ] **TASK-008: Create CyclusInstance Lifecycle Manager**
  - Manage creation, activation, closure, and reopening of cyclus instances
  - Location: `financeq/src/cyclus-instance/`
  - Implementation steps:
    1. Implement CyclusInstanceService for CRUD operations
    2. Add status transitions: planning → actief → afgerond (or heropend)
    3. Create kickoff workflow (initialize stages from template, assign defaults)
    4. Implement customization delta application (forked templates)
    5. Add automatic stage instantiation from template
    6. Support currentStageRefs array for parallel stage execution
    7. Emit cyclus.created, cyclus.started, cyclus.closed events

## Phase 3: Cryptographic Signoff & Versioning (Week 7-8)

- [ ] **TASK-009: Implement Cryptographic Signoff Engine (REQ-004)**
  - Support DigiD-Sign, qualified eIDAS, and ad-hoc OTP sign-off
  - Location: `financeq/src/signoff-engine/`
  - Implementation steps:
    1. Create SignoffRequestService to generate sign-off links with JWT tokens
    2. Implement DigiD-Sign integration (redirect to DigiD auth, receive PKCS#7 blob)
    3. Implement qualified eIDAS integration if applicable (regional connector)
    4. Implement ad-hoc OTP flow (SMS generation, validation, hashed storage)
    5. Bind signature to contentHash of specific DeliverableVersion
    6. Create Signoff records with signature blob and verification fields
    7. Emit signoff.requested, signoff.completed events
    8. Integration tests with mock DigiD responses

- [ ] **TASK-010: Implement Signoff Revocation & Verification (REQ-004)**
  - Support signoff revocation and external verification of signatures
  - Location: `financeq/src/signoff-verification/`
  - Implementation steps:
    1. Create SignoffRevocationService with motivation and revoker tracking
    2. Implement revocation audit trail (revokedAt, revokedBy, revokedReason)
    3. Add GET /api/signoffs/{id}/verify endpoint for external parties
    4. Return contentHashSigned vs. contentHashCurrent comparison
    5. Surface valid|false flag with reasoning
    6. Create test vectors with known DigiD PKCS#7 blobs
    7. Unit tests for revocation scenarios

- [ ] **TASK-011: Build Financial Linkage & Reconciliation (REQ-005)**
  - Link deliverables to begroting-regels and grootboek-mutaties
  - Location: `financeq/src/financial-linkage/`
  - Implementation steps:
    1. Create DeliverableFinancialLinksService to store regel/mutatie refs
    2. Implement recompute-financials endpoint with per-programma rollup
    3. Add out-of-sync detection logic (delta > tolerance)
    4. Block definitief transitions when out-of-sync (unless waived)
    5. Handle frozen raads-vastgesteld totals (store delta as discrepancy)
    6. Create discrepancy records for post-closure changes
    7. Unit tests with 412 regels × 2,847 mutaties dataset

- [ ] **TASK-012: Integrate with Begroting API**
  - Fetch begroting-regels and grootboek-mutaties from bookkeeping-bbv-compliance
  - Location: `financeq/src/external-integrations/bookkeeping-bridge/`
  - Implementation steps:
    1. Create BegrotingBridge service to query begroting API
    2. Implement caching strategy (invalidate on begroting updates)
    3. Add error handling for unavailable API (graceful degradation)
    4. Fetch and sum by programma and taakveld
    5. Performance optimization for large datasets (pagination, batching)

## Phase 4: Notifications & Escalations (Week 9-10)

- [ ] **TASK-013: Implement Deadline Escalation Policy Engine (REQ-007)**
  - Evaluate deadlines daily and emit escalation events at T-30, T-14, T-7, T-0, T+7
  - Location: `financeq/src/deadline-escalation/`
  - Implementation steps:
    1. Create DailyDeadlineEvaluator scheduled service (runs 00:15 UTC)
    2. Implement escalation lead calculation (daysRemaining → lead timing)
    3. Route notifications to responsibleAfdeling, portefeuillehouder, gemeentesecretaris
    4. Create notification templates for each escalation lead
    5. Mark escalationPolicy.escalations[].notificationSent = true after send
    6. Emit deadline.escalated events
    7. Handle waived deadlines (skip notifications)
    8. Unit tests with mock clock for timing

- [ ] **TASK-014: Build n8n Workflow Integration for Escalations**
  - Create n8n workflows to consume deadline.escalated events and send emails
  - Location: `financeq/seed-data/n8n-flows/`
  - Implementation steps:
    1. Create n8n flow: deadline-escalation-listener
    2. Trigger: Subscribe to deadline.escalated event
    3. Action: Build email template with stage info, deadline, deliverable status
    4. Action: Fetch recipient emails from Nextcloud user directory
    5. Action: Send via organisational email service
    6. Action: Log delivery in financeq events table
    7. Handle retry logic for failed deliveries

- [ ] **TASK-015: Implement Deadline Breached Event for Rechtmatigheid**
  - Publish deadline-breached events to rechtmatigheid-rapportage consumer
  - Location: `financeq/src/events/`
  - Implementation steps:
    1. Create deadline-breached domain event schema
    2. Emit on T+7 escalation when stage is still not afgerond
    3. Include stage ref, deadline, days overdue, responsible party
    4. Subscribe rechtmatigheid-rapportage to this event
    5. Add rechtmatigheid-rapportage event tests

## Phase 5: Public Publication & Woo Integration (Week 11-12)

- [ ] **TASK-016: Implement Woo Publication Scheduler (REQ-010)**
  - Publish definitief deliverables to Open Overheid feed within 14 days
  - Location: `financeq/src/woo-publication/`
  - Implementation steps:
    1. Create WooPublicationService to prepare metadata
    2. Implement redaction logic for bevat-bedrijfsgevoelige-info
    3. Store Woo exception records with art. 5.1 basis
    4. Build versioned URL structure (no link rot on supersession)
    5. Create supersession relationship tracking
    6. Integration with openoverheid API for publishing
    7. Handle retry logic and publication verification

- [ ] **TASK-017: Build n8n Workflow for Woo Publication**
  - Create n8n workflow to consume deliverable.published events and push to Open Overheid
  - Location: `financeq/seed-data/n8n-flows/`
  - Implementation steps:
    1. Create n8n flow: woo-publication-scheduler
    2. Trigger: Subscribe to deliverable.published event (state=definitief)
    3. Wait: 14 days before actual publication
    4. Check: If bevat-bedrijfsgevoelige-info, redact before publishing
    5. Transform: Create Open Overheid metadata envelope
    6. Action: POST to openoverheid API
    7. Log: Record publication date and URL in financeq
    8. Emit: publication.confirmed event for auditLog

- [ ] **TASK-018: Implement Heropening Support (REQ-009)**
  - Support reopening of closed cycles with full audit trail
  - Location: `financeq/src/cyclus-heropening/`
  - Implementation steps:
    1. Create ReopeningService with authorization checks (concerncontroller only)
    2. Implement heropening initiation with motivation and error references
    3. Create new Stage (heropening-jaarrekening) with sequenceNumber=7
    4. Create new Deliverable linked to heropening stage
    5. Persist original deliverable with superseded-by reference
    6. Emit notification to toezichthouder with heropening reason
    7. Build audit trail endpoint returning full chronology
    8. Unit tests for heropening scenarios

- [ ] **TASK-019: Integrate with decidesk for Raadsbesluit Chain (REQ-010)**
  - Trigger decidesk raadsstuk creation on raads-versie transition
  - Location: `financeq/src/external-integrations/decidesk-bridge/`
  - Implementation steps:
    1. Subscribe to deliverable state changes (concept → raads-versie)
    2. Create decidesk.raadsstuk via webhook with deliverable ref
    3. Consume decidesk raadsbesluit outcome (aangenomen|verworpen|aangehouden)
    4. Trigger definitief transition only on aangenomen outcome
    5. Handle aangehouden and verworpen outcomes (pause or reject)
    6. Implement error handling if decidesk unavailable
    7. Unit tests with mock decidesk responses

- [ ] **TASK-020: Integrate with docudesk for Archival (REQ-010)**
  - Mirror definitief deliverables to docudesk with retention policies
  - Location: `financeq/src/external-integrations/docudesk-bridge/`
  - Implementation steps:
    1. Subscribe to deliverable state changes (state=definitief)
    2. Create docudesk AIP with OAIS metadata
    3. Set retention policy: archiefwet-7-jaar for ondersteunende, archiefwet-permanent for vastgestelde
    4. Receive AIP identifier and store in Deliverable.docudesk-aip-id
    5. Handle versioning (heropened versions create new AIPs)
    6. Error handling if docudesk unavailable
    7. Integration tests with mock docudesk

## Phase 6: Testing & Documentation (Week 13-14)

- [ ] **TASK-021: Write Comprehensive Unit Tests**
  - Unit test coverage for all business logic
  - Location: `financeq/tests/unit/`
  - Implementation steps:
    1. Test deadline derivation for all sectors and edge cases
    2. Test stage transition gating (pass, fail, partial gate scenarios)
    3. Test content hashing and version immutability
    4. Test signoff creation and verification workflows
    5. Test financial linkage and out-of-sync detection
    6. Test deelnemer assignment and visibility rules
    7. Test escalation policy evaluation
    8. Test heropening audit trail
    9. Achieve 85%+ code coverage on core services

- [ ] **TASK-022: Write Integration Tests**
  - Integration tests for end-to-end workflows
  - Location: `financeq/tests/integration/`
  - Implementation steps:
    1. Full cyclus creation and kickoff workflow
    2. Sequential stage transitions with all gates passing
    3. Multiple signoff chain (portefeuillehouder → concerncontroller → gemeentesecretaris)
    4. Versioning and content hash chain
    5. Financial recompute and out-of-sync detection
    6. Deadline escalations across all leads (T-30 to T+7)
    7. Heropening workflow with supersession
    8. Woo publication after definitief transition

- [ ] **TASK-023: Create API Documentation**
  - OpenAPI/Swagger documentation for all REST endpoints
  - Location: `financeq/openspec/specs/pc-cyclus-workflow/api-docs/`
  - Implementation steps:
    1. Document all GET, POST, PATCH, DELETE endpoints
    2. Include request/response examples with real data
    3. Document error codes and status transitions
    4. Include authentication requirements (role-based)
    5. Document rate limits and pagination
    6. Create tutorial for each major workflow (cyclus creation, transition, signoff)
    7. Include curl examples and SDK usage

- [ ] **TASK-024: Build Journey Documentation (Docusaurus)**
  - Create user-facing journey docs for each cyclus type
  - Location: `financeq/openspec/specs/pc-cyclus-workflow/journeys/`
  - Implementation steps:
    1. Create kadernota-traject.md (timeline, roles, deliverables)
    2. Create programmabegroting-traject.md (process from voorjaarsnota to college approval)
    3. Create jaarrekening-traject.md (full cycle closure process)
    4. Create heropening-traject.md (error correction workflow)
    5. Use Docusaurus storytelling format (actors, decisions, pain points)
    6. Include "what's expected of me" checklist for each role
    7. Link to statutory references (BBV, Gemeentewet, etc.)

- [ ] **TASK-025: Create Admin Dashboard & Configuration UI**
  - Build UI for template management, deadline customization, escalation policies
  - Location: `financeq/src/admin-ui/`
  - Implementation steps:
    1. Create UI for viewing and forking CyclusTemplates
    2. Implement deadline override workflow (UI for approver)
    3. Create escalation policy editor (lead times, recipient roles)
    4. Build cycle health dashboard (deadline status, gate failures, bottlenecks)
    5. Create audit log viewer with filters (date range, actor, entity type)
    6. Implement heropening request form (with motivation and supporting docs)
    7. Add role-based access control to all admin features

- [ ] **TASK-026: Performance & Load Testing**
  - Verify system performance under load
  - Location: `financeq/tests/load/`
  - Implementation steps:
    1. Create load test for 10,000 organisations × 8 stages each
    2. Verify deadline evaluation completes in <500ms (p95)
    3. Verify stage transitions complete in <1s (p95) with gate checks
    4. Test versioning and signoff flows under concurrent writes
    5. Verify financial recompute for 412 regels × 2,847 mutaties completes in <5s
    6. Test Woo publication scheduler with 100+ publications in queue
    7. Document scaling recommendations

- [ ] **TASK-027: Security Review & Penetration Testing**
  - Security assessment of cryptographic signoff, access control, and data protection
  - Location: `financeq/security/`
  - Implementation steps:
    1. Review signoff implementation against NIST guidelines
    2. Verify PKCS#7 signature blob storage and validation
    3. Test access control enforcement (deelnemer roles, org-wide roles)
    4. Test visibility boundaries (concept vs. definitief deliverables)
    5. Verify audit log integrity and tamper detection
    6. Test error handling for secrets (OTP, signature keys)
    7. Run OWASP Top 10 assessment

- [ ] **TASK-028: Deployment & Ops Runbooks**
  - Create deployment scripts, ops documentation, and monitoring setup
  - Location: `financeq/ops/`
  - Implementation steps:
    1. Create Kubernetes manifests for pc-cyclus-workflow service
    2. Create database migration runbook
    3. Create backup strategy for immutable audit logs
    4. Set up monitoring alerts for deadline escalations, gate failures
    5. Create runbook for heropening process (manual approval workflow)
    6. Document incident response for signoff verification failures
    7. Create troubleshooting guide for common issues

---

## Success Criteria per Phase

### Phase 1 Complete
- [ ] All eight register schemas defined and registered in OpenRegister
- [ ] Deadline calculator working for all three sectors
- [ ] Three reference templates seeded in database
- [ ] Database migrations applied successfully
- [ ] All unit tests for deadline logic passing (100% coverage)

### Phase 2 Complete
- [ ] Stage transitions gating enforced (REQ-002 passing all scenarios)
- [ ] Deelnemer assignment working (REQ-006 access control verified)
- [ ] Version immutability enforced (REQ-003 edge cases handled)
- [ ] CyclusInstance lifecycle fully functional
- [ ] Integration tests for basic workflows passing

### Phase 3 Complete
- [ ] DigiD-Sign and OTP signoff flows working end-to-end
- [ ] Signoff verification endpoint live (external callers can verify)
- [ ] Financial linkage to begroting API operational
- [ ] Out-of-sync detection and blocking working
- [ ] Content hash verification tests passing

### Phase 4 Complete
- [ ] Daily deadline evaluator running and emitting correct leads
- [ ] n8n workflows for escalation notifications deployed
- [ ] Deadline-breached events flowing to rechtmatigheid-rapportage
- [ ] Escalation routing to correct roles verified in e2e test

### Phase 5 Complete
- [ ] Woo publication scheduler deployed and publishing within 14 days
- [ ] Redaction logic for bedrijfsgevoelige info working
- [ ] No link rot on heropening/supersession
- [ ] decidesk and docudesk integrations live
- [ ] E2E test from cyclus creation to published Woo feed

### Phase 6 Complete
- [ ] All unit tests passing (85%+ coverage)
- [ ] All integration tests passing
- [ ] API documentation generated and reviewed
- [ ] Journey docs published in Docusaurus
- [ ] Admin UI functional for template management
- [ ] Load tests show <500ms p95 for deadline queries
- [ ] Security review completed and issues remediated
- [ ] Deployment runbooks signed off by ops team

---

## Dependencies & Blockers

- **Blocker:** Must wait for OpenRegister schema registration process (if not yet defined)
- **Dependency:** Begroting API from bookkeeping-bbv-compliance must be available for financial linkage
- **Dependency:** Nextcloud user directory integration for deelnemer assignment
- **Dependency:** DigiD integration framework (may be org-specific)
- **Dependency:** n8n deployment and event bus setup (must be in place before Phase 4)

---

## Definition of Done

Each task is considered complete when:
- [ ] Code changes merged and reviewed
- [ ] Unit/integration tests passing
- [ ] Documentation updated
- [ ] No security warnings or code quality issues
- [ ] Accepted in refinement review by product owner
