# OpenSpec Proposal: Treasury & Cash Management — Shillinq — Other T2

**Change:** treasury-cash-management-other-t2  
**App:** Shillinq — Complete open-source business administration suite  
**Platform:** Nextcloud + OpenRegister  
**Date:** 2026-05-21  
**Status:** Proposed

## Executive Summary

Treasury & Cash Management (Other T2) consolidates 49 features spanning payment flexibility, cash flow forecasting, multi-currency support, and reconciliation workflows. This tier-2 module extends Shillinq's core accounting capabilities by enabling sophisticated payment orchestration, real-time cash visibility, and compliance-ready settlement workflows for freelancers, SMBs, and corporations.

**Primary demand drivers:** Payment method flexibility (demand: 11), personal expense tagging (11), payout reconciliation (10), and cash flow forecasting (10).

## Market Demand & Feature Summary

### Tier 1: Highest Demand (score 10–11)

| Feature | Demand | Driver | Category |
|---------|--------|--------|----------|
| **Payment method flexibility** | 11 | 3 tender mentions, 1% competitor coverage | Payment Methods |
| **Personal expense tagging** | 11 | 3 tender mentions, 1% competitor coverage | Expense Management |
| **Payout reconciliation** | 10 | 5% competitor coverage | Reconciliation |
| **Cash flow forecast** | 10 | 5% competitor coverage | Forecasting |

### Tier 2: High Demand (score 8–9)

| Feature | Demand | Category |
|---------|--------|----------|
| Credit card processing (Visa/Mastercard/Amex/Maestro) | 9 | Payment Methods |
| 40+ payment method support | 9 | Payment Methods |
| Manage foreign currency bank accounts | 9 | Multi-Currency |
| Saved Payment Method Charging | 9 | Payment Methods |
| Terminate payment arrangement on persistent default | 9 | Collection Management |
| Gift card payment acceptance | 8 | Payment Methods |
| Payment initiation via bank transfer | 8 | Payment Methods |
| Bank reconciliation | 8 | Reconciliation |
| Single-use Virtual Cards | 8 | Payment Methods |
| Cash flow statement (direct method) | 8 | Reporting |
| Multi-currency payment acceptance | 8 | Multi-Currency |
| Batch Payment Processing | 8 | Payment Methods |

### Tier 3: Moderate Demand (score 5–7)

40 features spanning: global payments (120+ currencies), multi-bank support, regional payment methods (EPS, Giropay, Klarna, Bancontact), POS solutions, QR code payments, physical cards, payment reminders, and online gateways.

## Architecture Alignment

### Compliance with Shillinq ADRs

- **ADR-001 (Data Layer):** All domain data via OpenRegister objects. Seed data includes: BankAccount, Payment, PaymentBatch, CashFlowForecast, ReconciliationRule, PaymentMethod entities.
- **ADR-003 (Backend):** Controller → Service → Mapper pattern. Every service includes `@spec` tags for traceability.
- **ADR-004 (Frontend):** Vue 2 + Pinia + @conduction/nextcloud-vue. Schema-driven forms, CRUD via CnIndexPage/CnDetailPage.
- **ADR-010 (NL Design):** All UI elements use NL Design System tokens. WCAG AA mandatory.
- **ADR-011 (Schema Standards):** Entities based on schema.org (Payment, Organization, MonetaryAmount) with OpenRegister relation mechanism.

### Key Entities (Reference from ADR-000)

- `BankAccount` (account details, connection status, sync schedule)
- `Payment` (amount, method, status, currency, party references)
- `PaymentBatch` (grouped transactions, settlement status)
- `CashFlowForecast` (projections based on GL entries and scheduled payments)
- `ReconciliationRule` (auto-matching rules for bank statement reconciliation)
- `PaymentMethod` (configuration per method: card types, gateway integration, limits)
- `CashAccount` (subsidiary ledger for treasury operations)

## Stakeholder Profiles

### CFO / Treasury Manager
**Goal:** Real-time cash visibility, predictable payouts, regulatory compliance.  
**Pain Point:** Manual reconciliation, fragmented payment data, currency exposure risk.  
**Success:** Forecast accuracy within 5%, 100% reconciliation match rate, <2hr settlement.

### Accountant
**Goal:** Audit trail, segregation of personal/business expenses, complete GL posting.  
**Pain Point:** Mismatched payments, missing documentation, month-end rework.  
**Success:** Zero reconciliation errors, <1hr GL mapping, NDAR/audit-ready export.

### Business Owner (SMB)
**Goal:** Simplify payment workflows, avoid fees, reduce manual data entry.  
**Pain Point:** Multiple payment provider logins, fraud risk, complicated setup.  
**Success:** Single dashboard, <5min payment initiation, fraud detection enabled.

### Compliance Officer
**Goal:** Segregation of duties, transaction audit trail, regulatory reporting.  
**Pain Point:** No role-based controls, incomplete transaction logs.  
**Success:** Role-based payment authorization, tamper-proof audit log, SiSa-ready export.

## Phase & Dependencies

**Phase:** T2 (secondary tier, depends on core GL + AP/AR modules)  
**Dependencies:**
- Core accounting module (GL, trial balance, financial statements)
- Accounts payable (vendor management, invoice matching)
- Accounts receivable (customer management, invoice tracking)
- Bank account integration (OpenRegister connection, sync service)

## Success Metrics

1. **Adoption:** ≥50% of Shillinq users activate at least one payment feature within 6mo.
2. **Reconciliation Accuracy:** ≥99% of transactions matched automatically (vs. manual).
3. **Cash Forecast Confidence:** Forecast variance <10% vs. actual (rolling 4-week).
4. **Payment Velocity:** Average payment cycle reduced by ≥25% (automated vs. manual baseline).
5. **Compliance:** 100% audit trail, zero unauthorized transactions detected in UAT.
6. **UX:** Task completion rate >80% for self-service payment workflows (no phone support needed).

## Deliverables

1. **Design Document** — entity schemas, workflows, UI mockups (5 key user journeys).
2. **Technical Specs** — 60+ REQ-* requirements (features, validation, error handling).
3. **Tasks & Subtasks** — backend services, frontend components, integration tests, documentation.
4. **Acceptance Criteria** — GIVEN/WHEN/THEN scenarios for each requirement.

---

**Next Steps:**
1. Design phase — entity definitions, workflow diagrams, schema-driven forms.
2. Community feedback — gather input on regional payment method prioritization.
3. Backend implementation — services, mappers, API endpoints.
4. Frontend & integration — Vue components, CnIndexPage/CnDetailPage, payment gateway APIs.
