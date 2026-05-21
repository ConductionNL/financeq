# Proposal: Treasury & Cash Management — Shillinq — Other T3

**Version:** 1.0  
**Spec ID:** treasury-cash-management-other-t3  
**App:** Shillinq  
**Platform:** Nextcloud + OpenRegister  
**Status:** Proposed  

## Executive Summary

This spec introduces **treasury and cash management capabilities** to Shillinq, including multi-bank connectivity, payment orchestration, reconciliation automation, and cash flow forecasting. It completes the financial transaction lifecycle by enabling users to connect bank accounts, schedule payments, optimize cash flow, and maintain real-time visibility across multiple banking relationships.

**Market demand:** 5 features with demand score 5, 20+ features with demand score 4+. Represents 41% of all feature requests in this category.

## Market Context

### Demand Analysis

- **Tender mentions:** 1 (bill payment scheduling)  
- **Competitor coverage:** 2% (short-term cash flow, bank reconciliation)
- **User story requests:** Multi-bank visibility, cash flow forecasting, auto-reconciliation
- **Regional focus:** Dutch/European businesses with multi-currency, SEPA, iDEAL, Giropay requirements

### Core Features (Demand ≥ 4)

| Feature | Demand | Category | Rationale |
|---------|--------|----------|-----------|
| Bill payment scheduling with check printing or online payment | 5 | Payment execution | Eliminate manual payment processing |
| iDEAL Payment with Streamlined Checkout | 5 | Payment channels | Dutch payment standard (98% coverage) |
| Customer object with payment method and subscription state | 5 | Payment routing | Enable flexible customer payment handling |
| Branded payment page | 5 | Customer experience | White-label payment experience |
| Plaid Link secure bank account connection widget | 5 | Bank connectivity | Secure bank account linking |
| Multi-bank cash visibility | 5 | Visibility | Real-time cash position across accounts |
| 13-week rolling cash flow forecast | 4 | Planning | Working capital optimization |
| Short-term cash flow projection | 4 | Planning | Daily/weekly cash planning |
| Configure custom auto-matching rules for bank reconciliation | 4 | Reconciliation | Automation of routine matching |
| Instant bank payment initiation (A2A transfers) | 4 | Payment execution | Same-day payment settlement |
| Currency Conversion | 4 | Multi-currency | Real-time FX rates |
| Credit card acquiring (Visa/Mastercard/Amex) with 3D Secure | 4 | Payment channels | Card payment acceptance |
| Platform fee calculation and deduction from splits | 4 | Payment routing | Marketplace revenue models |
| Bank reconciliation with auto-matching and manual adjustment options | 4 | Reconciliation | Flexible reconciliation workflows |

## Problem Statement

### Pain Points

1. **Fragmented cash visibility** — users maintain separate spreadsheets tracking multiple bank accounts
2. **Manual payment processing** — bill payments require external tools; no scheduling capability
3. **Bank reconciliation bottleneck** — matching transactions manually takes 4-6 hours monthly
4. **Payment channel silos** — customers can't self-serve payment method selection
5. **Cash flow blindness** — no forward-looking visibility into working capital needs

### Scope Boundaries

**In Scope:**
- Bank account connectivity and real-time balance updates
- Payment scheduling (bill pay, bulk transfers, A2A)
- Bank reconciliation with auto-matching rules
- Cash flow forecasting (13-week rolling model)
- Payment page customization and branding
- Customer payment object with method/subscription tracking
- Multi-currency transaction handling
- iDEAL, Giropay, SEPA, credit card payment channels

**Out of Scope:**
- Accounting automation (journal entry generation) — separate spec
- Advanced cash position management (netting, pooling) — future phase
- Derivative hedging or FX trading — out of domain
- POS terminal management — separate payment spec
- Complex supply chain financing — future enhancement

## Stakeholder Goals

### Finance Manager
**Goals:**
- Unified view of cash across all banks daily
- 80% reduction in reconciliation time via auto-matching
- Forward-looking cash position (2-4 weeks ahead)
- Exception-driven workflow (only manual intervention for mismatches)

**Success Metrics:**
- Cash visibility: <5 min to answer "what's our position?"
- Reconciliation: <1 hour monthly end-of-month close
- Forecast accuracy: ±5% for 1-week projection, ±10% for 3-week

### CFO/Controller
**Goals:**
- Audit trail for all payments and reconciliations
- Role-based access (treasury team can initiate, controller approves)
- Compliance with Dutch BBV, IV3, SiSa standards
- Multi-entity rollup (if multi-org future)

**Success Metrics:**
- 100% payment audit trail coverage
- Zero unauthorized payments executed
- Weekly variance reports (actual vs. forecast)

### Sales/Revenue
**Goals:**
- Customers can self-serve payment method selection at checkout
- Accept major payment channels (iDEAL, cards, transfers)
- Recurring payment support for subscriptions
- Reduce payment friction (white-label pages, minimal form fields)

**Success Metrics:**
- <1% checkout abandonment due to payment selection
- 25% of customers opt for recurring payments
- Branded pages reduce support tickets by 30%

### Operations
**Goals:**
- Reduce manual touchpoints in bill payment workflow
- Automate routine reconciliation
- Bank connection lifecycle management (add/remove/rotate)
- Background job resilience for 24/7 sync

**Success Metrics:**
- 95% of routine payments scheduled automatically
- 85%+ of transactions auto-matched without manual review
- <5 min mean time to retry failed bank sync

## MVP Definition

### Tier 1 (MVP)
Focuses on **core visibility + simple payments**:

1. **Bank Account Management** — add/edit/remove bank connections via Plaid Link
2. **Cash Visibility Dashboard** — multi-bank balance widget, transaction list by account
3. **Payment Scheduling** — schedule bill payments (single or bulk), execution via ACH/SEPA
4. **Basic Reconciliation** — import bank statement, auto-match by amount + date + description, mark as reconciled
5. **Customer Payment Method** — object with payment preference (iDEAL, card, transfer), SEPA mandate tracking

### Tier 2 (After MVP)
Adds **automation + forecasting + channels**:

6. **Auto-Matching Rules** — configurable matching strategies (exact amount, amount range, substring matching)
7. **13-Week Cash Flow Forecast** — project based on invoices due, bills due, historical patterns
8. **iDEAL Payment Checkout** — accept iDEAL payments from customer payment page
9. **Branded Payment Page** — customizable branding, logo, colors
10. **Recurring Payments** — subscription/mandate management for auto-debit

### Tier 3 (Roadmap)
Advanced features:

11. **Credit Card Acquiring** — Visa/MC/Amex with 3D Secure and tokenization
12. **Multi-Currency Conversion** — real-time FX rates, historical tracking
13. **Payment Optimization** — routing recommendations, fee comparison across banks
14. **Giropay + others** — regional payment channels (Germany, Austria, Nordics)

## Implementation Approach

### Architecture Philosophy

**Leverage OpenRegister** — all treasury data (accounts, transactions, payments, forecasts) stored as OpenRegister objects with schema-driven validation and audit trails.

**Integrate payment partners** — Plaid Link for bank connectivity, iDEAL provider for payment processing, maintain provider-agnostic abstraction layer.

**Async-first** — bank sync, reconciliation matching, forecasting run in background jobs with webhooks for notifications.

**RBAC-driven** — field-level permissions (treasurer can view, approver can initiate, finance manager can execute).

## Success Criteria

| Metric | Target | Owner | Measurement |
|--------|--------|-------|-------------|
| Cash position visibility time | <5 min to answer query | Ops | Timer from dashboard load to answer |
| Reconciliation effort | <1 hour monthly | Finance | Time tracking in job logs |
| Auto-match rate | ≥85% of transactions | Operations | Metric from reconciliation dashboard |
| Forecast accuracy (1-week) | ±5% | Finance | Actual vs. projection post-close |
| Payment execution uptime | ≥99.5% | Ops | Payment job success rate |
| Feature adoption | ≥50% of users enable bank sync | Product | Feature flag metrics |

## Open Questions & Next Steps

1. **Bank connectivity partner** — Plaid (EU coverage good, cost ~2-3% of ARR) vs. Open Banking APIs directly (lower cost, higher integration effort)?
2. **Payment processing** — single provider (Adyen, Stripe) vs. multi-provider abstraction layer?
3. **Forecast model** — linear regression + seasonality vs. ML models?
4. **Phase 1 payment channels** — start with SEPA/ACH + iDEAL, or add cards in MVP?

**Decision needed by:** Design phase kickoff  
**Owner:** Product Lead (Specter)
