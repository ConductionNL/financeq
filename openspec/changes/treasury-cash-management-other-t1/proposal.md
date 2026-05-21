# OpenSpec Change: Treasury & Cash Management — Other T1

**Status:** Proposal  
**App:** Shillinq (Complete open-source business administration suite)  
**Change ID:** treasury-cash-management-other-t1  
**Last Updated:** 2026-05-21  

---

## Executive Summary

Implement a unified treasury and cash management suite for Shillinq, enabling small businesses and corporations to consolidate accounts payable, accounts receivable, bank accounts, and cash flow across multiple currencies. This feature cluster addresses 49 treasury-related demands ranging from basic multi-bank account management (demand: 1598) to advanced FX exposure tracking and subscription management (demand: 1604-1672).

**Market validation:** 708 tender mentions across 49 clustered features; 1-40% competitor coverage indicates significant open-source gap.

---

## Features

### Tier 1: Core Treasury (High Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Unified AP/AR/Spend task list for cash flow management** | 2128 | Consolidated inbox for accounts payable, receivable, and spend items with aging, priority, and due-date views |
| **FX exposure management** | 1672 | Track and analyze foreign exchange risks across multi-currency positions |
| **Treasury and cash management** | 1640 | Core dashboard for liquidity, forecasting, and strategic cash positioning |
| **Foreign currency bank account management** | 1605 | Separate currency balances per account with real-time settlement tracking |
| **Subscription management with recurring payment scheduling** | 1604 | Schedule and automate recurring payments with approval workflows |

### Tier 2: Bank & Account Management (High Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Bank account management with multiple account support** | 1598 | Manage 2+ bank accounts with unified transaction overview |
| **Supplier payment management with global payment processing** | 1598 | Process payments to 196+ countries via integrated payment gateways |
| **Cash management with cash flow forecasting** | 1596 | Forecast liquidity based on AR, AP, and scheduled items |
| **Multi-bank account management with consolidated cash view** | 1596 | Single dashboard showing aggregate position across all accounts |
| **Multi-currency account management** | 1594 | Per-account currency assignment with automatic conversion tracking |

### Tier 3: Payment & Liquidity (Medium-High Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Payment method management** | 1595 | Support cards, bank debits, wallets, ACH, SEPA, Trustly, etc. |
| **Mass payment processing with batch scheduling** | 254 | Bulk process 100+ payments with approval and scheduling |
| **Invoice matching with approval and batching** | 32 | Match invoices to POs, batch for payment, batch approval |
| **Advance payment processing** | 14 | Apply advance payments against future invoices |
| **Standing order and direct debit identification** | 11 | Identify and manage recurring debit patterns |

### Tier 4: Forecasting & Analytics (Medium Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Cash flow statement with projection** | 38 | Snapshot + forecast based on outstanding items |
| **Forecast cash flow based on AR, AP, recurring items** | 18 | Multi-scenario forecasting with sensitivity analysis |
| **View cash flow forecast based on open invoices/bills** | 17 | Dashboard widget showing 90-day projection |

### Tier 5: Bank Connections & Integrations (Low-Medium Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **12,000+ financial institution connections** | 86 | Support 12K+ banks via Plaid/TrueLayer/similar |
| **Multi-bank account connection with unified overview** | 95 | Sync transactions from multiple banks in real-time |
| **99% UK bank account coverage for open banking** | 14 | Full UK bank coverage via PSD2/open banking APIs |
| **Request to Pay (R2P) open banking** | 449 | Send payment requests via open banking channels |
| **Instant Bank Pay (IBP) one-off payments** | 193 | One-click payments via open banking APIs |

### Tier 6: Multi-Currency & Regulatory (Low-Medium Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Multi-currency accounting (160+ currencies)** | 20 | Full accounting in 160+ currencies with auto FX rates |
| **Multi-currency support with configurable rates** | 25 | Manual/auto FX rate management |
| **Target Currency Configuration** | 180 | Set primary reporting currency for consolidation |
| **Currency Support** | 23 | Core multi-currency foundation |

### Tier 7: Analytics & Enrichment (Low Demand)

| Feature | Demand | Description |
|---------|--------|-------------|
| **Enrich product for merchant name/category standardization** | 218 | Standardize merchant/category metadata |
| **Cash flow overview with money in/out visualization** | 86 | Visual cash flow funnel |
| **Intuit Assist Finance Agent** | 59 | AI-powered analysis and comparisons |

---

## User Stories (Synthesized from Features)

### Treasury Manager
**Goal:** Maintain optimal liquidity and manage cash flow across multiple accounts and currencies

- **US-TM-001:** As a treasury manager, I want a unified task inbox showing all AP, AR, and spend items so I can prioritize payments and collections.
- **US-TM-002:** As a treasury manager, I want to forecast cash flow for the next 90 days based on open invoices, bills, and scheduled payments so I can plan liquidity needs.
- **US-TM-003:** As a treasury manager, I want to view FX exposure by currency pair and time bucket so I can identify and hedge foreign exchange risks.
- **US-TM-004:** As a treasury manager, I want to see aggregate cash position across all bank accounts and currencies so I can make strategic decisions on liquidity placement.

### CFO / Finance Director
**Goal:** Strategic financial reporting, compliance, and performance visibility

- **US-CFO-001:** As a CFO, I want to see a multi-currency consolidated cash flow statement (actual + forecast) so I can report to the board and stakeholders.
- **US-CFO-002:** As a CFO, I want to configure the target reporting currency so all multi-currency transactions consolidate correctly for financial statements.
- **US-CFO-003:** As a CFO, I want audit trails and approval workflow controls on payments so I can ensure compliance and prevent unauthorized transactions.

### Accountant / AP/AR Specialist
**Goal:** Manage day-to-day transactional flows and reconciliation

- **US-AR-001:** As an accountant, I want to match invoices to purchase orders and batch them for payment approval so I can reduce AP processing time.
- **US-AR-002:** As an accountant, I want to identify and categorize standing orders and direct debits so I can track recurring obligations.
- **US-AR-003:** As an accountant, I want to reconcile bank transactions against ledger entries so I can close the period cleanly.

### Bank Liaison / Operations Manager
**Goal:** Manage bank accounts and payment execution

- **US-BANK-001:** As a bank liaison, I want to connect to our primary bank and sync transactions in real-time so I can monitor cash positions automatically.
- **US-BANK-002:** As a bank liaison, I want to process mass payments in batches with scheduling and approval so I can execute payroll and vendor payments efficiently.
- **US-BANK-003:** As a bank liaison, I want to support multiple payment methods (SEPA, ACH, cards, open banking) so we can pay suppliers across different countries.

---

## Stakeholders

### Treasury Manager
**Responsibilities:** Daily cash position monitoring, liquidity planning, payment prioritization  
**Goals:** Reduce payment friction, prevent cash shortfalls, optimize returns on excess cash  
**Pain Points:** Manual spreadsheets tracking multiple bank accounts, no forecast visibility, FX exposure surprises

### CFO / Finance Director
**Responsibilities:** Strategic cash positioning, financial reporting, regulatory compliance  
**Goals:** Consolidated multi-currency reporting, board-ready dashboards, compliance audit trail  
**Pain Points:** Delayed reports due to manual consolidation, incomplete FX tracking, scattered approval workflows

### Accountant / AP/AR Specialist
**Responsibilities:** Invoice processing, AP/AR aging, bank reconciliation  
**Goals:** Faster invoice-to-pay cycles, accurate reconciliation, period close efficiency  
**Pain Points:** Manual matching of invoices to POs, no batch payment capability, reconciliation errors from data gaps

### Bank Liaison / Operations Manager
**Responsibilities:** Bank relationships, payment execution, account management  
**Goals:** Faster payment processing, multi-bank automation, compliance with payment timelines  
**Pain Points:** Logging into multiple bank portals, manual batch file uploads, limited payment method support

### Company Director / Business Owner
**Responsibilities:** Overall business health, risk management, stakeholder reporting  
**Goals:** Visibility into cash runway, confidence in payment controls, regulatory compliance  
**Pain Points:** Cash position surprises, approval bottlenecks, inability to forecast impact of business decisions

---

## Market Context

**Demand Validation:**
- 708 tender mentions across 49 treasury-related features
- Demand scores: 2128 (top), median 59, range 11–2128
- Competitor coverage: 1–40% (median 3%), indicating significant open-source gap

**Regional Focus:**
- Primary: Netherlands (Dutch regulatory alignment: BBV, IV3, SiSa, DigiInkoop)
- Secondary: EU/UK (PSD2/open banking, SEPA, regulatory standards)
- Global: 196-country payment support

**Related Specs & Integrations:**
- Bookkeeping & general ledger (existing Shillinq module)
- Accounts payable (invoice/PO module)
- Bank reconciliation (existing module)
- Multi-currency accounting (infrastructure)
- Payment gateway integrations (Peppol, Trustly, Plaid, TrueLayer)

---

## Success Criteria

✓ **Functional:** Treasury manager can forecast cash for next 90 days and see consolidated position across 3+ currencies/accounts  
✓ **Adoption:** 80% of target users interact with task inbox daily  
✓ **Compliance:** All payment transactions have full audit trail and approval workflow  
✓ **Performance:** Task inbox loads <2s with 10K+ open items; cash position updates within 5 min of bank sync  
✓ **Coverage:** Support 3+ payment methods; connect to 100+ financial institutions

---

## Open Questions / Dependencies

1. **Bank Connection API:** Which open banking provider (Plaid, TrueLayer, Finery, etc.)? Cost/rate limits?
2. **Payment Gateway:** Which gateways for 196-country support (Stripe, Wise, Remitly, SWIFT, etc.)?
3. **FX Rate Source:** Real-time ECB rates, Xe.com, or manual override capability?
4. **Forecast Assumptions:** Should forecast assume all AP/AR settle on due date, or configurable assumptions?
5. **Regulatory:** Which compliance frameworks (GDPR, PCI-DSS for payment data, Dutch tax filing)?

---

## Not in Scope (This Spec)

- Detailed payroll module (bonus/deduction/compliance per country)
- Contract-driven spend forecasting
- Supply chain finance / invoice financing / factoring
- Intuit Assist AI agent (separate feature, demand: 59)
- Blockchain / crypto payment support
- Multi-company consolidation accounting (separate spec)

---

## Change Artifacts

- **design.md** — Data models, entity schemas, seed data, customer journeys
- **specs.md** — Detailed requirements with GIVEN/WHEN/THEN scenarios
- **tasks.md** — Implementation plan with checkboxes and effort estimates
