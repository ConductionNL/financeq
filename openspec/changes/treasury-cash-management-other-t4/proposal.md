# Proposal: Treasury & Cash Management — Shillinq — Other T4

**Status:** Proposed  
**Change:** treasury-cash-management-other-t4  
**Product:** Shillinq (Complete open-source business administration suite for Nextcloud)  
**Scope:** Treasury, liquidity forecasting, payment optimization, regulatory compliance

---

## Executive Summary

This proposal introduces 12 treasury and cash management features for Shillinq to strengthen liquidity forecasting, payment optimization, bank reconciliation, and regulatory compliance. Features span bank entry matching, loan refinancing identification, cash flow visualization, payment netting, supplier verification, and separation-of-duties detection.

Target users: CFOs, treasurers, finance controllers in SMBs and corporations operating in Dutch regulatory environments.

---

## Market Demand & Segmentation

All features address treasury management gaps in freelancer, sole proprietor, and SMB segments currently not covered by free/open-source alternatives. Demand signals are inferred from Dutch government compliance (BBV, IV3, DigiInkoop) and European banking standards (SEPA, IBAN validation).

| Feature | Demand Signal | User Segment | Compliance Driver |
|---------|--------------|--------------|-------------------|
| Match bank entries | Bank reconciliation automation | All | SEPA, IV3 |
| Identify loans eligible for refinancing | Debt optimization | Corps/SMBs | Financial planning, BBV |
| View cash flow graph | Liquidity forecasting | Corps/SMBs | Budget planning, IV3 |
| Cash pooling | Multi-account optimization | Corps | Liquidity management |
| Analyze bank fees | Cost transparency | All | Financial analysis |
| View portfolio of all active payment arrangements | Settlement tracking | All | Treasury oversight |
| Payment netting | Payment optimization | Corps | Treasury management |
| Verify supplier IBAN bank account | Fraud prevention | All | Supplier verification, DigiInkoop |
| Payment provider per administration | Multi-currency support | Corps | Multi-tenant finance |
| Detect separation of duties violations | Fraud risk mitigation | Orgs | Internal controls, IV3 |
| Centralized payment factory | Bulk payment optimization | Corps | Treasury automation |
| Monitor liquid reserve requirements | Regulatory compliance | All | Liquidity monitoring |

---

## Features (12)

### 1. Match bank entries
Automatically reconcile bank entries with ledger transactions, flagging mismatches and unreconciled items.

**Stakeholder Value:**  
- Finance controller: reconciliation automation, fraud detection  
- Auditor: audit trail completeness, regulatory compliance

### 2. Identify loans eligible for refinancing
Analyze existing debt and suggest refinancing opportunities based on market rates and loan terms.

**Stakeholder Value:**  
- CFO: debt optimization, interest cost reduction  
- Treasurer: financial planning, cash flow improvement

### 3. View cash flow graph
Visualize projected cash flow over configurable horizons (30/60/90 days) based on committed receivables and payables.

**Stakeholder Value:**  
- CFO: liquidity forecasting, strategic planning  
- Treasurer: working capital management

### 4. Cash pooling
Aggregate balances across multiple accounts to optimize liquidity and reduce banking costs.

**Stakeholder Value:**  
- Treasurer: liquidity optimization, cost reduction  
- CFO: working capital efficiency

### 5. Analyze bank fees
Break down and visualize bank charges by transaction type, account, and time period to identify savings opportunities.

**Stakeholder Value:**  
- Finance controller: cost visibility, vendor negotiation  
- CFO: operational expense reduction

### 6. View portfolio of all active payment arrangements
Display current payment plans, standing orders, and settlement agreements in a unified view.

**Stakeholder Value:**  
- Treasurer: payment oversight, obligation tracking  
- CFO: settlement liability visibility

### 7. Payment netting
Identify and execute offsetting payables and receivables to reduce gross payment volume and banking fees.

**Stakeholder Value:**  
- Treasurer: payment optimization, cash flow improvement  
- CFO: liquidity enhancement

### 8. Verify supplier IBAN bank account
Validate supplier IBAN format and cross-check against procurement records to prevent payment fraud.

**Stakeholder Value:**  
- Procurement: fraud prevention, supplier verification  
- Finance controller: payment security

### 9. Payment provider per administration
Assign distinct payment service providers to each administration (multi-currency, multi-geography support).

**Stakeholder Value:**  
- Treasurer: payment flexibility, cost optimization  
- CFO: multi-entity management

### 10. Detect separation of duties violations
Flag transactions and approvals that violate segregated authorization rules (requester ≠ approver ≠ payer).

**Stakeholder Value:**  
- Compliance officer: fraud risk mitigation, audit readiness  
- CFO: internal control strengthening

### 11. Centralized payment factory
Orchestrate bulk payment initiation, approval, and settlement across multiple accounts and providers.

**Stakeholder Value:**  
- Treasurer: payment automation, approval workflow  
- Finance controller: settlement oversight

### 12. Monitor liquid reserve requirements
Track actual vs. required cash reserves (minimum balance, regulatory reserve) and alert when thresholds are breached.

**Stakeholder Value:**  
- CFO: regulatory compliance, liquidity safety  
- Treasurer: reserve monitoring

---

## Stakeholder Profiles

### Chief Financial Officer (CFO)
**Goals:**  
- Forecast cash flow and plan working capital  
- Optimize debt and reduce interest costs  
- Monitor regulatory compliance (reserves, separation of duties)  
- Centralize financial visibility across administrations

**Pain Points:**  
- No integrated cash flow forecast; relies on spreadsheets  
- No automated loan refinancing analysis  
- Manual separation-of-duties checking; audit-time discovery of violations

**Authority:**  
- Approves treasury policies, debt refinancing, and reserve thresholds  
- Reviews dashboard metrics daily/weekly

### Treasurer
**Goals:**  
- Optimize payment timing and settlement costs  
- Aggregate liquidity across accounts  
- Execute netting and bulk payments efficiently  
- Ensure IBAN compliance and fraud prevention

**Pain Points:**  
- Manual payment arrangement tracking  
- No automated netting; relies on external payment systems  
- Limited visibility into bank fees by transaction type

**Authority:**  
- Initiates and approves payment batches  
- Configures payment providers and bank accounts  
- Approves exceptions to reserve thresholds

### Finance Controller
**Goals:**  
- Reconcile bank entries daily without manual effort  
- Verify supplier accounts before payment  
- Track payment obligations and due dates  
- Support audit with complete audit trails

**Pain Points:**  
- Bank reconciliation is manual and error-prone  
- No automated supplier IBAN verification  
- Difficulty identifying fraud patterns across transactions

**Authority:**  
- Approves exceptions to reconciliation rules  
- Configures bank account mappings  
- Reviews separation-of-duties violation reports

### Compliance Officer (Audit/Internal Controls)
**Goals:**  
- Verify separation of duties across all transactions  
- Maintain audit trail for regulatory inspection  
- Flag high-risk payment patterns  
- Report compliance status (reserve requirements, approvals)

**Pain Points:**  
- No automated separation-of-duties checks  
- Manual audit of payment approvals  
- Visibility gaps across multi-administration environments

**Authority:**  
- Configures authorization rules and thresholds  
- Reviews and approves high-risk exceptions  
- Generates compliance reports for external auditors

---

## User Stories (Generated from Features)

### US-001: Match bank entries
**As a** finance controller  
**I want to** automatically match bank statements to ledger transactions  
**So that** I can reconcile accounts without manual effort and detect discrepancies

**Acceptance Criteria:**
- GIVEN: A bank entry with amount, date, counterparty, reference  
WHEN: I view the reconciliation dashboard  
THEN: The system matches the entry to ledger transactions with ≥95% accuracy  
- GIVEN: An unmatched bank entry  
WHEN: I manually assign it to a ledger transaction  
THEN: The system learns the pattern and applies it to future entries

### US-002: Identify loans eligible for refinancing
**As a** CFO  
**I want to** see which existing loans could be refinanced at lower rates  
**So that** I can reduce interest expenses and improve profitability

**Acceptance Criteria:**
- GIVEN: Existing loans in the system with terms and rates  
WHEN: I run the refinancing analysis  
THEN: The system shows loans with market rates ≤ current rate minus 0.5%  
- GIVEN: A loan marked "eligible for refinancing"  
WHEN: I click "View Details"  
THEN: The system shows estimated interest savings, new payment schedule, and lender options

### US-003: View cash flow graph
**As a** treasurer  
**I want to** visualize projected cash flow over 30/60/90 days  
**So that** I can plan payment timing and identify liquidity gaps

**Acceptance Criteria:**
- GIVEN: Outstanding receivables and payables in the system  
WHEN: I view the cash flow dashboard with a 90-day horizon  
THEN: The system displays a line chart with cumulative projected balance by day  
- GIVEN: A manual cash flow adjustment (e.g., expected bonus payment)  
WHEN: I add the adjustment to the forecast  
THEN: The chart updates to reflect the new projection

### US-004: Cash pooling
**As a** treasurer  
**I want to** aggregate balances across my company's 3 bank accounts  
**So that** I can optimize liquidity and reduce fees on idle balances

**Acceptance Criteria:**
- GIVEN: Multiple bank accounts with different balances  
WHEN: I enable cash pooling for a set of accounts  
THEN: The system calculates the optimal transfer amounts to consolidate to the primary account  
- GIVEN: A pooling rule set to run daily at 5 PM  
WHEN: The scheduled job runs  
THEN: The system initiates transfers and logs the action

### US-005: Analyze bank fees
**As a** finance controller  
**I want to** see a breakdown of bank fees by transaction type and account  
**So that** I can identify cost-saving opportunities and negotiate better rates

**Acceptance Criteria:**
- GIVEN: 6 months of bank statements imported  
WHEN: I view the bank fees report  
THEN: The system shows fees aggregated by type (transfer, wire, settlement, maintenance) and account  
- GIVEN: A list of high-fee transaction types  
WHEN: I export the report  
THEN: The system generates a CSV with trend analysis and recommendations

### US-006: View portfolio of all active payment arrangements
**As a** treasurer  
**I want to** see all standing orders, payment plans, and settlement agreements in one view  
**So that** I can track upcoming obligations and avoid missed payments

**Acceptance Criteria:**
- GIVEN: Multiple payment arrangements (standing orders, installment plans, settlements)  
WHEN: I view the portfolio dashboard  
THEN: The system displays a table with arrangement type, amount, due date, and status  
- GIVEN: A standing order due today  
WHEN: I view the portfolio  
THEN: The system highlights the due arrangement and shows pending initiation status

### US-007: Payment netting
**As a** treasurer  
**I want to** identify and execute mutual offsets between payables and receivables  
**So that** I can reduce gross payment volume and associated banking costs

**Acceptance Criteria:**
- GIVEN: Payables to vendor X and receivables from vendor X  
WHEN: I run the netting analysis  
THEN: The system identifies the offset amount and shows net payment required  
- GIVEN: A netting transaction approved  
WHEN: I confirm execution  
THEN: The system creates a contra-journal entry and marks both transactions as settled via netting

### US-008: Verify supplier IBAN bank account
**As a** finance controller  
**I want to** validate that a supplier's IBAN is correct before initiating payment  
**So that** I can prevent fraud and misrouted payments

**Acceptance Criteria:**
- GIVEN: A new supplier with IBAN SE91 3000 0000 0109 4000 0785  
WHEN: I initiate a payment to this supplier  
THEN: The system validates the IBAN format and flags any inconsistencies with prior payments  
- GIVEN: An IBAN format error detected  
WHEN: I view the payment approval dialog  
THEN: The system shows a warning and requires controller override to proceed

### US-009: Payment provider per administration
**As a** treasurer  
**I want to** configure different payment providers (e.g., Wise for EUR, Stripe for GBP) per company administration  
**So that** I can optimize cost and currency support for each entity

**Acceptance Criteria:**
- GIVEN: A company with subsidiaries in multiple currencies  
WHEN: I configure payment providers per administration  
THEN: The system allows me to assign different providers to each and remembers the selection  
- GIVEN: A payment batch to a non-local currency  
WHEN: I initiate the payment  
THEN: The system routes it to the provider configured for that currency/administration

### US-010: Detect separation of duties violations
**As a** compliance officer  
**I want to** receive alerts when the same person requests, approves, and executes a payment  
**So that** I can enforce internal controls and prevent fraud

**Acceptance Criteria:**
- GIVEN: A payment initiated by User A, approved by User A, and executed by User A  
WHEN: The payment is settled  
THEN: The system flags this as a separation-of-duties violation and logs it in the audit report  
- GIVEN: A separation-of-duties rule configured (requester ≠ approver ≠ payer)  
WHEN: I run the compliance audit  
THEN: The system shows all violations from the past 30 days with usernames and transaction details

### US-011: Centralized payment factory
**As a** treasurer  
**I want to** batch multiple payments, get them approved through a workflow, and execute them to multiple providers  
**So that** I can reduce manual processing and ensure consistent approval

**Acceptance Criteria:**
- GIVEN: 25 supplier invoices due this week  
WHEN: I create a payment batch  
THEN: The system collects all invoices, groups by supplier and payment method, and shows the batch summary  
- GIVEN: A batch approved by the CFO  
WHEN: I click "Execute"  
THEN: The system initiates payments to each supplier via their configured provider and logs each transaction

### US-012: Monitor liquid reserve requirements
**As a** CFO  
**I want to** set minimum cash reserve thresholds and receive alerts when reserves fall below requirements  
**So that** I can ensure regulatory compliance and financial stability

**Acceptance Criteria:**
- GIVEN: A minimum reserve requirement of €50k  
WHEN: I set this threshold in the treasury dashboard  
THEN: The system monitors the total cash balance and alerts me if it falls below €50k  
- GIVEN: An alert triggered due to low reserves  
WHEN: I view the alert  
THEN: The system shows the current balance, shortfall amount, and recommended actions (defer payments, increase credit lines)

---

## Reuse Analysis

This proposal leverages existing Shillinq entities (no new core entities proposed):

- **BankAccount:** Used for bank entry matching and cash pooling  
- **ScheduledPayment:** Foundation for payment arrangements and netting  
- **Payment:** Transaction-level fraud detection and fee analysis  
- **Supplier:** IBAN verification and payment provider assignment  
- **Loan:** Refinancing analysis  
- **GeneralLedgerEntry:** Bank reconciliation matching  
- **Role & Authorization:** Separation-of-duties rules  

No new entity definitions needed; all features layer business logic on existing schemas.

---

## Risks & Mitigation

| Risk | Severity | Mitigation |
|------|----------|-----------|
| Bank API integration complexity | High | Phased approach: manual upload → SEPA import → bank API connectors |
| IBAN validation across countries | High | Use IBAN.js library; support EU + select non-EU regions first |
| Separation-of-duties conflicts with small orgs | Medium | Make rules configurable; allow exceptions with audit trail |
| Cash pooling regulatory constraints | Medium | Document per-country rules; flag non-compliant configurations |
| Refinancing data accuracy | Medium | Use real market data; disclose limitations in UI |
| Performance: reconciliation on large volumes | High | Batch processing; async jobs; caching of matching rules |

---

## Success Criteria

- Cash flow forecasts within 5% accuracy vs. actual settlement over 90-day periods  
- Bank reconciliation ≥95% auto-match rate on SEPA-standard entries  
- IBAN validation prevents ≥90% of misrouted payments in pilot  
- Separation-of-duties detection 100% accuracy (rule-based, no ML false positives)  
- Payment netting reduces gross payment volume by ≥10% in pilot  
- Dashboard load time <2s with 50k historical transactions  

---

## Scope Boundaries

**In Scope:**
- Integration with existing BankAccount, Payment, Supplier, and GeneralLedgerEntry entities  
- SEPA bank statement import (MT940, CAMT.053)  
- IBAN validation per ISO 13616  
- Configurable separation-of-duties rules  
- Cash flow forecasting based on committed transactions  
- Admin UI for treasury policies and provider configuration  

**Out of Scope (Phase 2+):**
- Direct bank API connections (Phase 2)  
- Machine learning for fraud detection (Phase 2)  
- Multi-currency fx-exposure modeling (Phase 2)  
- Automated loan origination (Phase 2)  
- Integration with external rating agencies (Phase 2)  

---

## Dependencies

- Shillinq core (bookkeeping, invoicing, procurement, supplier management)  
- OpenRegister (for schema definitions and entity storage)  
- Nextcloud (for user auth, permissions, file storage)  
- IBAN validation library (iban.js or similar)  
- Bank statement parsers (MT940, CAMT.053 libraries)  

---

## Timeline & Estimate

**Phase 1 (This Change):** Core treasury features, SEPA import, separation-of-duties  
- Effort: 120 hours  
- Duration: 4 weeks  
- Team: 1 backend, 1 frontend, 1 QA  

**Delivery Artifacts:**
1. proposal.md (this document)  
2. design.md (data model, API design, UI mockups)  
3. specs.md (detailed requirements with test scenarios)  
4. tasks.md (implementation tasks for Phase 1)  

---

**Prepared by:** Specter (Intelligence/Research)  
**Date:** 2026-05-21  
**Version:** 1.0
