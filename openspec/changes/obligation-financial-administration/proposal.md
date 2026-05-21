# Proposal: Obligation & Financial Administration — Shillinq

## Executive Summary

This specification introduces obligation and financial administration capabilities to Shillinq, Nextcloud's complete open-source business administration suite. The module tracks financial commitments (obligations), automates deadline management with AI-powered task generation, and provides compliance reporting for on-time settlement performance. Built on OpenRegister schemas aligned to Dutch government standards and double-entry accounting principles.

## Market Demand

| Feature | Demand | Mentions | Coverage | Category |
|---------|--------|----------|----------|----------|
| AI obligation task management with automated deadline tracking | 2,484 | 824 | 6% | AI |
| Obligation management | 1,713 | 530 | 57% | Core |
| Issue final settlement decision (vaststelling) | 221 | 73 | — | Core |
| Obligation Analytics (PowerBI) | 124 | 31 | 15% | Analytics |
| 99% On-Time Obligation Compliance | 103 | 29 | 6% | Governance |
| Register business assets and auto-calculate yearly depreciation | 63 | 21 | — | Core |
| Register fixed asset | 63 | 19 | 3% | Core |
| Pre-commitment approval ensuring spending is authorized | 56 | 18 | 1% | Core |
| SiSa-verantwoording | 38 | 12 | 1% | Compliance |
| Fixed Asset Depreciation Boards | 10 | 5 | — | Analytics |

**Total addressable market demand: 5,045+ mentions across 22 features.**

## User Stories

### As a Finance Administrator
I need to register financial obligations from invoices and purchase orders so that I can track all commitments and monitor payment compliance against deadlines.

**Acceptance Criteria:**
- GIVEN an unpaid invoice or PO
- WHEN I create an obligation record
- THEN the system captures obligationNumber, dueDate, amount, creditor, obligationType, and description
- AND creates an automated ObligationTask for the due date
- AND calculates days-to-deadline for priority scoring

### As a Finance Officer
I need to monitor obligations approaching their due date so that I can ensure on-time payment and avoid penalties.

**Acceptance Criteria:**
- GIVEN the obligations list filtered by status (open, overdue, settled)
- WHEN I view the dashboard
- THEN I see obligations grouped by deadline (overdue → due this week → future) with alert colors
- AND each obligation shows dueDate, amount, creditor, and days-remaining
- AND I can filter by obligationType, creditor, or amount range

### As a Compliance Officer
I need to generate compliance reports showing settlement performance metrics so that I can demonstrate 99% on-time compliance to auditors and stakeholders.

**Acceptance Criteria:**
- GIVEN a reporting period (month, quarter, year)
- WHEN I generate a ComplianceReport
- THEN the system calculates: total obligations, on-time settlements, overdue settlements, compliance rate, average payment days
- AND exports the report to PowerBI for dashboard visualization
- AND provides drill-down by obligation type, creditor, or payment method

### As an Asset Manager
I need to register fixed assets, assign depreciation schedules, and track yearly depreciation calculations so that I can maintain accurate financial records for tax and audit purposes.

**Acceptance Criteria:**
- GIVEN a capital asset purchase (equipment, vehicle, property)
- WHEN I register a FixedAsset with purchase cost and date
- THEN the system auto-generates a DepreciationSchedule based on asset type
- AND calculates annual depreciation using linear, declining-balance, or units-of-production method
- AND provides a depreciation board showing yearly amounts and accumulated depreciation

### As a Chief Financial Officer
I need to issue settlement decisions finalizing and marking multiple obligations as settled so that I can authorize payment batches and maintain formal settlement records for compliance.

**Acceptance Criteria:**
- GIVEN a batch of obligations ready for payment
- WHEN I create a SettlementDecision
- THEN the system validates all obligations are properly approved
- AND issues a formal decision document with decision number, date, authorized person, total amount
- AND marks all linked obligations with settledOnTime = true/false based on due date comparison
- AND creates an audit trail for SiSa compliance

## Stakeholders

### Finance & Accounting Team
- **Role:** Obligation creation, tracking, settlement
- **Goals:** Timely payment, compliance with terms, audit trail
- **Constraints:** Dutch government compliance (BBV, IV3, SiSa), tax code mapping
- **Touchpoints:** Obligation list, detail, settlement workflow

### Compliance & Audit Team
- **Role:** Reporting, verification, risk assessment
- **Goals:** 99% on-time settlement, regulatory compliance, audit-ready records
- **Constraints:** GDPR, audit trail requirements, report formats
- **Touchpoints:** ComplianceReport, audit trail, settlement decisions

### Asset Management Team
- **Role:** Fixed asset registration, depreciation tracking
- **Goals:** Accurate financial records, tax compliance, asset lifecycle
- **Constraints:** Depreciation method standards, asset useful-life determination
- **Touchpoints:** FixedAsset registration, depreciation schedules, depreciation boards

### Finance Approval Chain
- **Role:** Authorization of spending, approval of settlements
- **Goals:** Control exposure, prevent unauthorized spending
- **Constraints:** Pre-commitment approval authority levels
- **Touchpoints:** Pre-commitment approval, settlement decision authorization

## Customer Journeys

### Journey: Monitor & Settle Obligations
1. **Trigger:** Invoice received / PO issued
2. **Pain point:** Manual tracking of deadlines and payment status
3. **Solution:** AI-generated ObligationTask with auto-calculated due dates
4. **Outcomes:** Reduced missed payments, improved compliance, automated reminders
5. **Metrics:** % on-time settlements, average payment days, compliance rate

### Journey: Comply with Dutch Government Standards
1. **Trigger:** Quarterly/annual reporting requirement (SiSa, IV3, BBV)
2. **Pain point:** Manual compilation of obligation and settlement records
3. **Solution:** Automated ComplianceReport with audit trail
4. **Outcomes:** Faster reporting, audit-ready records, reduced manual error
5. **Metrics:** Report generation time, audit findings count, compliance pass rate

### Journey: Manage Fixed Assets & Depreciation
1. **Trigger:** Asset acquisition / fiscal year end
2. **Pain point:** Manual depreciation calculations, error-prone asset tracking
3. **Solution:** Auto-generated DepreciationSchedule, calculated yearly depreciation
4. **Outcomes:** Accurate balance sheet, reduced manual workload, tax compliance
5. **Metrics:** Depreciation accuracy, schedule adherence, audit pass rate

### Journey: Authorize Spending Before Commitment
1. **Trigger:** Purchase order creation
2. **Pain point:** Spending can happen without prior authorization
3. **Solution:** Pre-commitment approval workflow with ApprovalChain integration
4. **Outcomes:** Controlled spending, reduced unauthorized commitments
5. **Metrics:** Approval rate, days-to-approval, spending within budget

## Implementation Scope

### Included
- ✅ Obligation lifecycle management (create, read, update, settle)
- ✅ AI-powered task generation for deadline tracking
- ✅ Compliance reporting with PowerBI integration
- ✅ Fixed asset registration with depreciation scheduling
- ✅ Settlement decision workflow with approval chain
- ✅ SiSa compliance audit trail
- ✅ Multi-currency support
- ✅ Dutch government compliance (BBV, IV3, SiSa, DigiInkoop)

### Out of Scope (Future Features)
- ❌ Real-time bank API integration (scheduled for v2)
- ❌ Machine learning for obligation discovery from contracts
- ❌ Obligation extraction from OCR/document parsing
- ❌ Advanced analytics dashboards (initial: PowerBI export)
- ❌ Multi-company consolidation reporting

## Architecture Alignment

- **Data Layer:** All domain data stored in OpenRegister objects (ADR-001)
- **API:** REST endpoints following `/index.php/apps/shillinq/api/{resource}` pattern (ADR-002)
- **Backend:** Controller → Service → Mapper architecture with Spec traceability (ADR-003)
- **Frontend:** Vue 2 + Pinia + @conduction/nextcloud-vue, no custom state management (ADR-004)
- **Security:** Nextcloud auth, field-level RBAC, no PII in logs (ADR-005)
- **Schema Standards:** schema.org vocabulary, vCard for contacts, Dutch government mappings (ADR-011)
- **Testing:** PHPUnit tests for services, Vue tests, browser tests for scenarios (ADR-008)
- **i18n:** Dutch (nl) + English (en) translations (ADR-007)
- **Deduplication:** Reuses ObjectService, RegisterService, WebhookService, NotificationService from OpenRegister core

## Timeline

**Phase 1 (Sprint 1-2):** Core obligation CRUD, dashboard, basic reporting
**Phase 2 (Sprint 3-4):** Fixed assets, depreciation scheduling, settlement decisions
**Phase 3 (Sprint 5-6):** AI task generation, compliance reporting, SiSa export
**Phase 4 (Sprint 7+):** PowerBI integration, multi-currency, advanced analytics
