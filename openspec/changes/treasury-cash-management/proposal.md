# Treasury & Cash Management — Shillinq

**Change:** treasury-cash-management  
**Spec:** treasury-cash-management  
**Status:** proposed  
**Created:** 2026-05-21  
**Platform:** Nextcloud + OpenRegister

---

## Executive Summary

Treasury & Cash Management is a comprehensive treasury portfolio and cash flow management module for Shillinq that enables corporate treasurers, municipal treasurers, and CFOs to achieve real-time visibility across multi-account cash positions, forecast liquidity accurately, manage payments strategically, and ensure compliance with statutory treasury regulations (Wet Fido, schatkistbankieren).

The module provides:
- **Multi-bank cash visibility** with consolidated real-time dashboard
- **Liquidity forecasting** with AI-powered projections
- **Payment scheduling & batching** with SEPA/SWIFT export
- **FX risk management** with exposure tracking
- **Compliance monitoring** (Wet Fido, schatkistbankieren, credit ratings)
- **Intercompany transfers** and in-house banking
- **Treasury reporting** for committees and auditors

---

## Market Demand & Scope

| Feature | Demand | Category | Type |
|---------|--------|----------|------|
| Cash management with liquidity forecasting | 1748 | scheduling | core |
| Treasury & cash integration for payment timing | 1734 | integration | core |
| Cash account tracking (petty cash) | 1677 | analytics | core |
| Digital lockbox for procurement | 1052 | scheduling | stretch |
| Export treasury compliance report | 406 | governance | core |
| Business dashboard (P&L, expenses, cash flow) | 218 | analytics | nice-to-have |
| Square payment processing | 216 | integration | stretch |
| Schedule payment date | 208 | scheduling | core |
| Payment batch creation with SEPA | 50 | document-mgmt | core |
| Connect Dutch bank via PSD2 | 25 | document-mgmt | stretch |
| Multi-currency balance tracking | 101 | analytics | core |

**Total feature count:** 27  
**Core features:** 11  
**Estimated effort:** 5-7 sprints

---

## User Stories (21 total)

### Must-Have Stories (11)
1. **Centralized payment factory** — Collect & execute payments from all group entities with SEPA/SWIFT generation
2. **Import bank statements** — Multi-bank statement import for cash tracking
3. **View consolidated cash position** — See aggregate cash across all bank accounts
4. **Multi-bank cash visibility** — Real-time positions for liquidity management
5. **Cash flow forecasting** — AI-powered liquidity projections
6. **Payment provider per administration** — Separate bank/Mollie config per business
7. **Record intercompany transfers** — Register liquidity moves between municipal entities
8. **Monitor schatkistbankieren compliance** — Track treasury law compliance in real-time
9. **Register deposito investments** — Track short-term cash placements
10. **View daily cash position** — Dashboard with opening balances per bank
11. **Generate monthly treasury report** — Committee reporting with analytics

### Should-Have Stories (7)
- In-house bank for intercompany funding
- Schedule payments strategically
- Support multiple banks
- Offset recovery against future payments
- Verify counterparty credit ratings
- Monitor liquid reserve requirements
- Export payroll as SEPA

### Nice-to-Have Stories (3)
- AI-driven cash flow forecasting (advanced)
- Generate SEPA pain.001 files (advanced export)
- Verify counterparty credit ratings (governance)

---

## Stakeholders (18 linked)

| Role | Key Responsibility | Demand |
|------|-------------------|--------|
| **Treasurer** | Cash management, FX hedging, bank relations | real-time cash visibility |
| **CFO** | Consolidated cash position for liquidity management | strategic visibility |
| **Municipal Treasurer** | Wet Fido compliance, schatkistbankieren tracking | compliance dashboard |
| **Controller** | Payment scheduling, cash flow optimization | payment scheduling |
| **Head of Finance** | BBV/IV3 compliance, external audit support | automated IV3 export |
| **Financial Administrator** | Payment execution, recovery offsets | payment workflows |
| **Payroll Administrator** | Export approved payroll as SEPA | SEPA export |
| **Business Owner** | Multi-account reconciliation | cash consolidation |
| **Alderman (Wethouder)** | Portfolio dashboard, budget defense | real-time KPI dashboard |
| **Management Accountant** | Cost allocation, overhead distribution | cost reporting |
| **Audit Committee Member** | Financial statement verification, audit | audit trails |
| **Board Treasurer** | Annual financial statements, ALV reporting | financial reporting |
| **MT Member** | Cross-functional decision making | structured MT meetings |

---

## Data Entities (8 new OpenRegister schemas)

1. **CashAccount** — Bank accounts, petty cash, cash equivalents
2. **CurrencyBalance** — Multi-currency balance per account
3. **FXExposure** — Foreign exchange risk tracking
4. **LiquidityForecast** — Daily/weekly/monthly cash projections
5. **PaymentBatch** — Grouped payments for mass processing
6. **RequestForQuotation** — RFx with digital lockbox
7. **ScheduledPayment** — Future payments with recurrence
8. **TreasuryTask** — Unified AP/AR/spend task list

---

## Success Criteria

| Criterion | Target | Metric |
|-----------|--------|--------|
| Cash visibility | <5 min update latency | bank import frequency |
| Compliance | 100% Wet Fido coverage | compliance dashboard |
| Payment efficiency | 90% batch adoption | payment automation rate |
| Forecast accuracy | 85% within 5% variance | AI model validation |
| User adoption | 70% active monthly | feature usage analytics |

---

## Architecture Decisions

- **OpenRegister-first:** All domain data as schemas (CashAccount, PaymentBatch, etc.)
- **Multi-bank connectors:** PSD2 integration (Dutch banks), fallback manual import
- **AI forecasting:** Anthropic API for liquidity predictions
- **Compliance engines:** Rules-based (Wet Fido kasgeldlimiet, schatkistbankieren thresholds)
- **SEPA/SWIFT:** ISO 20022 pain.001 & CAMT.053/054 support

---

## Timeline

- **Sprint 1-2:** Data model, bank import, basic cash dashboard
- **Sprint 3:** SEPA export, payment scheduling
- **Sprint 4-5:** Forecasting, compliance engines, treasury reporting
- **Sprint 6:** Advanced features (FX exposure, IHB), testing
- **Sprint 7:** Deployment, documentation, training

---

## Handoff

Spec is ready for detailed design phase with full entity schemas, seed data, workflows, and acceptance criteria.
