# Design: Treasury & Cash Management

**Change ID:** treasury-cash-management-other-t1  
**Last Updated:** 2026-05-21  

---

## Overview

This design document defines the data models, API structures, and frontend architecture for treasury and cash management in Shillinq. The implementation leverages OpenRegister for all domain data, following ADR-001 (data-layer) and ADR-004 (frontend).

---

## 1. Entity & Schema Definitions

### Reuse Analysis

The following existing entities are **reused** from the Shillinq entity model (no new entities required):

- **BankAccount** — Multi-account support (existing)
- **CashAccount** — Cash holdings (existing)
- **CurrencyBalance** — Per-account currency tracking (existing)
- **FXExposure** — Foreign exchange exposure tracking (existing)
- **LiquidityForecast** — Cash flow forecasting (existing)
- **Payment** — Payment transactions (existing)
- **ScheduledPayment** — Recurring payment scheduling (existing)
- **Account** — General ledger accounts (existing)
- **Invoice** — Accounts receivable (existing)
- **VendorBill** — Accounts payable (existing)
- **Subscription** — Recurring subscriptions (existing)

**Deduplication Check:** All core treasury entities already exist in Shillinq's entity model. This spec focuses on:
- **Business logic workflows** (task lists, approvals, forecasting rules)
- **API integrations** (bank sync, payment gateways, FX rate feeds)
- **Frontend views** (unified task inbox, cash dashboard, FX exposure reports)
- **Backend services** (cash forecasting engine, payment batching, bank reconciliation)

No new OpenRegister schemas are required; this spec implements domain services leveraging existing OpenRegister objects.

### Service Layer Architecture

The treasury module adds stateless services in the `src/Services/Treasury/` directory:

```
src/Services/Treasury/
├── CashFlowForecastingService.php       // Multi-scenario forecasting
├── LiquidityAnalysisService.php         // Liquidity ratios, KPIs
├── PaymentBatchingService.php           // Batch aggregation & splitting
├── ApprovalWorkflowService.php          // Payment approval chains
├── BankSyncService.php                  // Bank transaction import
├── FXExposureCalculationService.php     // FX exposure aggregation
├── TreasuryTaskService.php              // Task inbox aggregation
└── NotificationService.php              // Treasury-specific notifications
```

---

## 2. Data Model & Relationships

### Core Entities (OpenRegister objects)

#### BankAccount
Links physical bank accounts to the ledger.

```yaml
schema: BankAccount
register: shillinq-accounts
properties:
  name: string (required)          # "Checking Account", "Reserve Account"
  accountNumber: string (required)  # IBAN or local account format
  bankName: string (required)       # "ABN AMRO", "ING"
  country: string                   # ISO 3166 country code
  currency: string (required)       # ISO 4217 (EUR, USD, GBP)
  balance: decimal                  # Current balance (denormalized, synced from bank)
  status: enum                      # active | inactive | closed
  syncStatus: enum                  # synced | pending | error
  lastSyncAt: datetime              # Last bank sync timestamp
  generalLedgerAccount: relation    # → Account (for posting transactions)
  createdAt: datetime
  updatedAt: datetime
```

#### CurrencyBalance
Tracks balance in each currency per bank account.

```yaml
schema: CurrencyBalance
register: shillinq-balances
properties:
  bankAccount: relation (required)  # → BankAccount
  currency: string (required)       # ISO 4217
  balance: decimal (required)       # Amount in that currency
  equivalentInBase: decimal         # Balance in target currency
  exchangeRateUsed: decimal         # FX rate applied for conversion
  valuedAt: datetime                # Timestamp of FX rate
  createdAt: datetime
  updatedAt: datetime
```

#### ScheduledPayment
Recurring payment definition.

```yaml
schema: ScheduledPayment
register: shillinq-payments
properties:
  description: string (required)    # "Rent", "Software subscription"
  payee: relation (required)        # → Supplier or Person
  amount: decimal (required)        # Payment amount
  currency: string (required)       # ISO 4217
  schedule: enum (required)         # monthly | weekly | daily | quarterly | annual
  nextDueDate: date (required)      # Next execution date
  lastExecutedAt: datetime          # Previous execution
  status: enum (required)           # active | paused | completed
  approvalRequired: boolean         # Needs approval before execution?
  bankAccount: relation             # → BankAccount (payment source)
  createdAt: datetime
  updatedAt: datetime
```

#### FXExposure
Tracks foreign exchange risk.

```yaml
schema: FXExposure
register: shillinq-fx
properties:
  baseCurrency: string (required)   # EUR
  foreignCurrency: string (required) # USD
  totalExposure: decimal (required) # Sum of all outstanding amounts
  buckets:                          # Time buckets for maturity analysis
    - dueIn0to30Days: decimal
    - dueIn31to90Days: decimal
    - dueIn91to180Days: decimal
    - dueIn180plusDays: decimal
  hedge: object                     # Optional hedging info
    - hedgeType: enum               # forward | option | swap
    - coveredAmount: decimal
    - hedgeRate: decimal
  marketRate: decimal               # Current spot rate
  unrealizedGain: decimal           # Mark-to-market P&L
  calculatedAt: datetime            # Timestamp of calculation
  createdAt: datetime
  updatedAt: datetime
```

#### LiquidityForecast
Cash flow projection.

```yaml
schema: LiquidityForecast
register: shillinq-forecasts
properties:
  forecastDate: date (required)     # Date this forecast was created
  scenarios:                        # Multiple scenarios
    - base:                         # Pessimistic | Base | Optimistic
        - period: date              # Week/month/quarter
        - inflows: decimal          # Collections expected
        - outflows: decimal         # Payments due
        - netCash: decimal          # Inflows - outflows
        - cumulativeBalance: decimal # Running total
  assumptions:
    - arCollectionRate: decimal     # % of AR collected by due date
    - apPaymentRate: decimal        # % of AP paid by due date
    - otherAssumptions: array
  confidence: enum                  # low | medium | high (based on historical accuracy)
  createdBy: relation               # → User
  createdAt: datetime
```

#### TreasuryTask
Unified inbox item for AP, AR, and payment actions.

```yaml
schema: TreasuryTask
register: shillinq-treasury-tasks
properties:
  title: string (required)          # "Pay Invoice #INV-2026-001"
  category: enum (required)         # ap | ar | payment | liquidity
  relatedObject: relation           # → Invoice, VendorBill, ScheduledPayment, etc.
  dueDate: date (required)          # Action due date
  priority: enum                    # low | normal | high | urgent
  status: enum (required)           # open | in_progress | overdue | completed
  amount: decimal                   # Amount in question (for AP/AR)
  currency: string                  # ISO 4217
  assignedTo: relation              # → User
  approvalRequired: boolean         # Needs approval?
  approvalChain: array              # [User, User, ...] approval sequence
  notes: string                     # Internal notes
  createdAt: datetime
  updatedAt: datetime
  completedAt: datetime
```

---

## 3. Key Workflows

### 3.1 Unified Task Inbox

**Data Flow:**
1. System aggregates open invoices, unpaid bills, scheduled payments → TreasuryTask records
2. TreasuryTaskService reads from:
   - Invoice (AR not yet collected)
   - VendorBill (AP not yet paid)
   - ScheduledPayment (upcoming, awaiting execution)
   - Payment (pending, failed)
3. Frontend queries `/api/treasury/tasks?status=open&priority=high` with pagination
4. User marks task complete → Payment execution or reconciliation

**Business Rules:**
- AP task created when VendorBill status = `approved` and not yet paid
- AR task created when Invoice status = `sent` and dueDate passed or approaching
- Overdue flag set when dueDate < today
- Priority auto-set based on:
  - AR: Higher if invoice aged 60+ days
  - AP: Higher if invoice nearing early-pay discount deadline
  - Scheduled: Normal for routine, Urgent if missed payment date

### 3.2 Cash Flow Forecasting

**Calculation Engine:**
1. Query outstanding AP, AR, scheduled payments by due date
2. Apply collection/payment assumptions:
   - AR: Default 90% collected by due date (configurable)
   - AP: Default 100% paid by due date
   - Scheduled: 100% on schedule date
3. Aggregate by time bucket:
   - 0–30 days
   - 31–90 days
   - 91–180 days
   - 180+ days
4. Calculate cumulative position: Starting balance + inflows - outflows
5. Generate three scenarios: Pessimistic (70% collection), Base (90%), Optimistic (100%)

**Data Model:**
- Stored in `LiquidityForecast.scenarios[base].periods[]`
- Recalculated daily or on-demand
- Historical accuracy tracked for confidence scoring

**API Endpoint:**
```
GET /api/treasury/cash-flow/forecast
  ?startDate=2026-05-21
  &endDate=2026-08-19
  &bucketSize=week|month|quarter
  &scenario=base|pessimistic|optimistic
  Response: { periods: [...], assumptions: {...} }
```

### 3.3 Multi-Currency & FX Exposure

**Architecture:**
1. Each BankAccount has a primary `currency`
2. CurrencyBalance tracks amount in each currency
3. FXExposure aggregates by currency pair

**Calculation:**
```
For each foreign currency:
  exposure = SUM(
    AR invoiced in currency (unpaid),
    AP invoices in currency (unpaid),
    Bank balance in currency,
    Scheduled payments in currency
  )
  buckets = Group by maturity date
  hedge = Look up hedges from hedge register
  unrealizedGain = (marketRate - entryRate) * exposure
```

**Dashboard Views:**
- **By Currency Pair:** EUR/USD, EUR/GBP, etc., with exposure heat map
- **By Maturity:** Which currency pairs mature when?
- **Hedge Status:** What's hedged vs. exposed?
- **Unrealized P&L:** Current mark-to-market gains/losses

### 3.4 Payment Batching & Approval

**Process:**
1. Treasury manager selects 5+ AP invoices from task inbox
2. Clicks "Batch & Schedule"
3. System creates PaymentBatch object
4. Generates batch file (SEPA XML, ACH, etc.)
5. Routes to approval chain (per workflow config)
6. Once approved, schedules for next bank connectivity window
7. Posts to bank via Plaid/TrueLayer API
8. Updates Payment status to `initiated`

**Data Entities:**
- **PaymentBatch:** Groups 2+ Payment records with single approval
- **ApprovalChain:** Defines sequence of approvers and conditions
- **ApprovalRequest:** Tracks each step in approval process

---

## 4. Seed Data

### Sample BankAccount Objects

```json
[
  {
    "@self": {
      "register": "shillinq-accounts",
      "schema": "BankAccount",
      "slug": "abnamro-checking-eur"
    },
    "name": "ABN AMRO Checking",
    "accountNumber": "NL91ABNA0417164300",
    "bankName": "ABN AMRO",
    "country": "NL",
    "currency": "EUR",
    "balance": 245000.00,
    "status": "active",
    "syncStatus": "synced",
    "lastSyncAt": "2026-05-21T14:30:00Z"
  },
  {
    "@self": {
      "register": "shillinq-accounts",
      "schema": "BankAccount",
      "slug": "ing-usd-account"
    },
    "name": "ING USD Reserve",
    "accountNumber": "NL89INGB0002822854",
    "bankName": "ING",
    "country": "NL",
    "currency": "USD",
    "balance": 85000.00,
    "status": "active",
    "syncStatus": "synced",
    "lastSyncAt": "2026-05-21T13:45:00Z"
  },
  {
    "@self": {
      "register": "shillinq-accounts",
      "schema": "BankAccount",
      "slug": "triodos-gbp-account"
    },
    "name": "Triodos GBP UK",
    "accountNumber": "GB82TRIOOS212159",
    "bankName": "Triodos",
    "country": "GB",
    "currency": "GBP",
    "balance": 42500.00,
    "status": "active",
    "syncStatus": "synced",
    "lastSyncAt": "2026-05-21T12:00:00Z"
  }
]
```

### Sample ScheduledPayment Objects

```json
[
  {
    "@self": {
      "register": "shillinq-payments",
      "schema": "ScheduledPayment",
      "slug": "rent-monthly-amsterdam"
    },
    "description": "Office Rent - Amsterdam Headquarters",
    "payee": "Real Estate Partner BV",
    "amount": 8500.00,
    "currency": "EUR",
    "schedule": "monthly",
    "nextDueDate": "2026-06-01",
    "lastExecutedAt": "2026-05-01T09:15:00Z",
    "status": "active",
    "approvalRequired": false,
    "bankAccount": "abnamro-checking-eur"
  },
  {
    "@self": {
      "register": "shillinq-payments",
      "schema": "ScheduledPayment",
      "slug": "saas-monthly-slack"
    },
    "description": "Slack Enterprise Subscription",
    "payee": "Slack Technologies",
    "amount": 899.00,
    "currency": "USD",
    "schedule": "monthly",
    "nextDueDate": "2026-06-05",
    "lastExecutedAt": "2026-05-05T08:00:00Z",
    "status": "active",
    "approvalRequired": true,
    "bankAccount": "ing-usd-account"
  },
  {
    "@self": {
      "register": "shillinq-payments",
      "schema": "ScheduledPayment",
      "slug": "insurance-quarterly-allianz"
    },
    "description": "Business Liability Insurance - Allianz",
    "payee": "Allianz Nederland",
    "amount": 4200.00,
    "currency": "EUR",
    "schedule": "quarterly",
    "nextDueDate": "2026-06-15",
    "lastExecutedAt": "2026-03-15T10:30:00Z",
    "status": "active",
    "approvalRequired": false,
    "bankAccount": "abnamro-checking-eur"
  }
]
```

### Sample CurrencyBalance Objects

```json
[
  {
    "@self": {
      "register": "shillinq-balances",
      "schema": "CurrencyBalance",
      "slug": "abnamro-eur-balance"
    },
    "bankAccount": "abnamro-checking-eur",
    "currency": "EUR",
    "balance": 245000.00,
    "equivalentInBase": 245000.00,
    "exchangeRateUsed": 1.0,
    "valuedAt": "2026-05-21T14:30:00Z"
  },
  {
    "@self": {
      "register": "shillinq-balances",
      "schema": "CurrencyBalance",
      "slug": "ing-usd-balance"
    },
    "bankAccount": "ing-usd-account",
    "currency": "USD",
    "balance": 85000.00,
    "equivalentInBase": 78450.00,
    "exchangeRateUsed": 0.9229,
    "valuedAt": "2026-05-21T13:45:00Z"
  },
  {
    "@self": {
      "register": "shillinq-balances",
      "schema": "CurrencyBalance",
      "slug": "triodos-gbp-balance"
    },
    "bankAccount": "triodos-gbp-account",
    "currency": "GBP",
    "balance": 42500.00,
    "equivalentInBase": 50300.00,
    "exchangeRateUsed": 1.1835,
    "valuedAt": "2026-05-21T12:00:00Z"
  }
]
```

### Sample FXExposure Objects

```json
[
  {
    "@self": {
      "register": "shillinq-fx",
      "schema": "FXExposure",
      "slug": "eur-usd-exposure-may2026"
    },
    "baseCurrency": "EUR",
    "foreignCurrency": "USD",
    "totalExposure": 185000.00,
    "buckets": {
      "dueIn0to30Days": 45000.00,
      "dueIn31to90Days": 95000.00,
      "dueIn91to180Days": 35000.00,
      "dueIn180plusDays": 10000.00
    },
    "hedge": {
      "hedgeType": "forward",
      "coveredAmount": 50000.00,
      "hedgeRate": 0.9150
    },
    "marketRate": 0.9229,
    "unrealizedGain": 3651.50,
    "calculatedAt": "2026-05-21T15:00:00Z"
  }
]
```

### Sample LiquidityForecast Objects

```json
[
  {
    "@self": {
      "register": "shillinq-forecasts",
      "schema": "LiquidityForecast",
      "slug": "forecast-2026-05-21"
    },
    "forecastDate": "2026-05-21",
    "scenarios": {
      "base": [
        {
          "period": "2026-05-21",
          "inflows": 34500.00,
          "outflows": 18200.00,
          "netCash": 16300.00,
          "cumulativeBalance": 372750.00
        },
        {
          "period": "2026-05-28",
          "inflows": 52100.00,
          "outflows": 28500.00,
          "netCash": 23600.00,
          "cumulativeBalance": 396350.00
        },
        {
          "period": "2026-06-04",
          "inflows": 18900.00,
          "outflows": 8900.00,
          "netCash": 10000.00,
          "cumulativeBalance": 406350.00
        }
      ],
      "pessimistic": [
        {
          "period": "2026-05-21",
          "inflows": 24150.00,
          "outflows": 18200.00,
          "netCash": 5950.00,
          "cumulativeBalance": 362400.00
        }
      ],
      "optimistic": [
        {
          "period": "2026-05-21",
          "inflows": 41400.00,
          "outflows": 18200.00,
          "netCash": 23200.00,
          "cumulativeBalance": 379100.00
        }
      ]
    },
    "assumptions": {
      "arCollectionRate": 0.90,
      "apPaymentRate": 1.0,
      "otherAssumptions": ["Scheduled payments execute on time"]
    },
    "confidence": "medium",
    "createdBy": "treasury-manager-001"
  }
]
```

### Sample TreasuryTask Objects

```json
[
  {
    "@self": {
      "register": "shillinq-treasury-tasks",
      "schema": "TreasuryTask",
      "slug": "task-inv-2026-001-payment"
    },
    "title": "Pay Invoice #INV-2026-001 - TechVendor Ltd",
    "category": "ap",
    "relatedObject": "vendor-bill-2026-00124",
    "dueDate": "2026-05-28",
    "priority": "high",
    "status": "open",
    "amount": 12500.00,
    "currency": "EUR",
    "assignedTo": "ap-specialist-001",
    "approvalRequired": false,
    "notes": "Early payment discount available until 2026-05-25 (2%)"
  },
  {
    "@self": {
      "register": "shillinq-treasury-tasks",
      "schema": "TreasuryTask",
      "slug": "task-ar-2026-085-collection"
    },
    "title": "Follow up: Invoice #AR-2026-085 - Client Services",
    "category": "ar",
    "relatedObject": "invoice-2026-00085",
    "dueDate": "2026-05-21",
    "priority": "urgent",
    "status": "overdue",
    "amount": 18750.00,
    "currency": "EUR",
    "assignedTo": "ar-specialist-001",
    "approvalRequired": false,
    "notes": "Overdue 15 days; client contact unresponsive"
  },
  {
    "@self": {
      "register": "shillinq-treasury-tasks",
      "schema": "TreasuryTask",
      "slug": "task-scheduled-rent-june"
    },
    "title": "Execute: Office Rent Payment - June 2026",
    "category": "payment",
    "relatedObject": "scheduled-payment-rent-monthly",
    "dueDate": "2026-06-01",
    "priority": "normal",
    "status": "open",
    "amount": 8500.00,
    "currency": "EUR",
    "assignedTo": "treasury-manager-001",
    "approvalRequired": false,
    "notes": ""
  }
]
```

---

## 5. Frontend Architecture

### Pages & Components

#### Dashboard: Treasury Overview
- `CnStatsBlock` × 4: Cash balance (consolidated), AR aging, AP aging, liquidity (30-day forecast)
- `CnChartWidget`: Cash flow funnel (inflows vs. outflows by bucket)
- `CnChartWidget`: FX exposure by currency pair
- `CnTableWidget`: "My Tasks" (high-priority, overdue items)
- `CnTimelineStages`: Multi-currency position timeline

#### Page: Task Inbox
- `CnIndexPage` → `CnDataTable` with columns:
  - Task title | Amount | Due Date | Category (AP/AR) | Priority | Status | Assignee
- Filterable by: status, category, priority, assignee, dueDate range
- Row click → detail panel with:
  - Related invoice/bill preview
  - Notes, approval chain (if required)
  - Action buttons: Mark Complete, Reassign, Batch (for AP)

#### Page: Cash Position
- `CnDetailCard` with sections:
  - **By Account:** Table of BankAccount with balance, currency, lastSync
  - **By Currency:** Pivot table showing EUR/USD/GBP totals
  - **Consolidated:** Total in reporting currency (target currency config)

#### Page: Cash Flow Forecast
- Date range picker + scenario selector (Pessimistic | Base | Optimistic)
- `CnChartWidget` (line chart): Cumulative balance projection
- Table: Periods with inflows, outflows, net cash
- Assumptions display: Collection rate, payment assumptions

#### Page: FX Exposure
- Filter by: currency pair, maturity bucket
- Heatmap: Exposure by pair × maturity
- Hedge status table: Current hedges, coverage %, unrealized P&L
- Time series chart: Spot rate vs. hedge rate

#### Page: Payments
- Two tabs:
  1. **Scheduled Payments:** List with next execution date, frequency, status
  2. **Batch Payments:** Create batch from task inbox, show staging/submitted/confirmed
- Action: Execute scheduled, create batch, view payment files

---

## 6. Backend Services (High-Level)

### CashFlowForecastingService

```php
// Generate forecast for date range with scenarios
forecast(
  DateTime $startDate,
  DateTime $endDate,
  array $assumptions = [
    'arCollectionRate' => 0.90,
    'apPaymentRate' => 1.0,
    ...
  ],
  string $bucketSize = 'week' // week|month|quarter
): LiquidityForecast

// Calculate based on:
// - Open invoices (AR) by due date
// - Unpaid bills (AP) by due date
// - Scheduled payments by next execution
// - Bank balances by currency
```

### TreasuryTaskService

```php
// Aggregate tasks from AR, AP, scheduled payments
listTasks(
  array $filters = ['status' => 'open', 'priority' => 'high'],
  int $limit = 50,
  int $offset = 0
): array[TreasuryTask]

// Auto-create task from invoice/bill
createFromInvoice(Invoice $invoice): TreasuryTask
createFromBill(VendorBill $bill): TreasuryTask
createFromScheduledPayment(ScheduledPayment $scheduled): TreasuryTask
```

### FXExposureCalculationService

```php
// Aggregate exposure by currency pair
calculateExposure(
  string $baseCurrency,
  string $foreignCurrency
): FXExposure

// Apply market rates and mark-to-market
applyMarketRate(
  FXExposure $exposure,
  decimal $spotRate,
  DateTime $asOf
): FXExposure

// Calculate unrealized P&L for hedges
calculateHedgeGain(
  FXExposure $exposure
): decimal
```

### PaymentBatchingService

```php
// Group invoices into batch
createBatch(
  array $paymentIds, // Payment PKs
  array $approvalChain // [User, User, ...] approvers
): PaymentBatch

// Generate bank file (SEPA XML, ACH, etc.)
generateBankFile(
  PaymentBatch $batch,
  string $format = 'sepa' // sepa|ach|swift
): string (file content)

// Post batch to bank via API
submitBatch(
  PaymentBatch $batch
): BatchSubmissionResponse
```

### BankSyncService

```php
// Sync transactions from bank API
sync(
  BankAccount $account,
  DateTime $since = null
): SyncResult

// Match transactions to ledger entries
matchToLedger(
  Transaction $bankTxn
): ?JournalEntry

// Handle unmatched transactions
handleUnmatched(
  Transaction $bankTxn
): ReconciliationTask
```

---

## 7. Integration Points

### Bank APIs (TBD: Provider Selection)
- **Provider Options:** Plaid, TrueLayer, Finery, Yodlee, etc.
- **Functionality:** Real-time transaction sync, balance verification, payment posting
- **Rate Limits:** Determine per provider (e.g., Plaid: 100 requests/min)

### FX Rate Feed (TBD)
- **Options:** ECB (daily, free), Xe.com API, OpenExchangeRates, etc.
- **Update Frequency:** Real-time (paid) or daily (free)
- **Fallback:** Manual override capability

### Payment Gateways (TBD)
- **SEPA/SCT:** Direct SEPA processing or via gateway (Cfonb, etc.)
- **ACH (US):** Via Stripe, Wise, or direct ACH file submission
- **International:** Wise, Remitly, or SWIFT channel (expensive)
- **Cards:** Stripe, Square for card payments

---

## 8. Non-Functional Requirements

### Performance
- Task inbox loads <2 sec with 10K+ items (paginated query)
- Cash position updates within 5 min of bank sync completion
- FX exposure recalculation <30 sec (nightly, or on-demand)
- Forecast generation <10 sec for 90-day projection

### Compliance
- All payments have immutable audit trail (via AuditTrailService)
- Approval workflow enforced before payment posting
- PCI-DSS compliance for payment data (tokenize account numbers in logs)
- GDPR compliance: Right to delete/export for personal data

### Reliability
- Bank sync failures → graceful degradation (cache last sync)
- FX rate feed failure → fallback to prior day rate + warning
- Payment API failure → queue for retry + notification to user
- Data integrity: Reconciliation between ledger and bank

---

## 9. Deployment & Data Migration

### Register Template (lib/Settings/shillinq_register.json)

Seed data per entity:
- **BankAccount:** 3 examples (EUR, USD, GBP)
- **ScheduledPayment:** 3 examples (rent, SaaS, insurance)
- **CurrencyBalance:** 3 examples (per account)
- **FXExposure:** 1 example (EUR/USD)
- **LiquidityForecast:** 1 example (base scenario)
- **TreasuryTask:** 3 examples (AP, AR, scheduled)

### Idempotency
- Import via `ConfigurationService::importFromApp()` matches by slug
- Re-import with `force: false` skips existing objects
- Version compare: Skip if seed version ≤ existing object version

---

## 10. Open Questions

1. **Bank Sync Timing:** Real-time (polling every 5 min?) or daily batch?
2. **FX Rates:** Real-time vs. daily? Manual override capability?
3. **Approval Workflow:** Sequential (A → B → C) or parallel (A & B)?
4. **Forecast Assumptions:** Hardcoded defaults or configurable per org?
5. **Payment Gateway Priority:** Start with SEPA only, or multi-currency from v1?
6. **Dashboard Refresh:** Real-time (WebSocket) or poll every 30 sec?
7. **Audit Trail Retention:** Forever, or per-country retention rules?
8. **Multi-company:** Single co. per instance, or support multi-company from v1?

---

## 11. Success Metrics

✓ **Functional Coverage:**
- [ ] Task inbox shows all open AP/AR items with <2s load time
- [ ] Cash flow forecast displays 3 scenarios (pessimistic/base/optimistic)
- [ ] FX exposure aggregates by currency pair with hedge status
- [ ] Cash position consolidates across 3+ currencies
- [ ] Scheduled payments execute on time with audit trail

✓ **Adoption:**
- [ ] 80%+ of treasury users interact with dashboard daily
- [ ] 50%+ of invoice payments use task inbox (vs. manual)
- [ ] Forecast used for 3+ monthly cash planning decisions

✓ **Data Quality:**
- [ ] 95%+ bank transaction auto-match rate
- [ ] 0 reconciliation discrepancies >€100
- [ ] 100% audit trail completeness for payments

---

