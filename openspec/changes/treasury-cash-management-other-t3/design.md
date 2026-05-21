# Design: Treasury & Cash Management — Shillinq

**Version:** 1.0  
**Spec ID:** treasury-cash-management-other-t3  

## Data Model

### Entity Definitions

All entities use schema.org vocabulary where applicable, stored as OpenRegister objects with `@self` envelope per ADR-001.

#### BankAccount

Represents a connected bank account used for receipts and payments.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "BankAccount",
    "slug": "iban-nl91abna0417164300"
  },
  "id": "uuid",
  "name": "string (required)",
  "description": "string",
  "iban": "string (required, unique within org, format: /^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$/)",
  "bic": "string",
  "currency": "string (ISO 4217, default: EUR)",
  "accountType": "string (enum: checking, savings, investment, other)",
  "bank": {
    "register": "shillinq",
    "schema": "Bank",
    "objectId": "uuid"
  },
  "plaidAccountId": "string (Plaid Link token, encrypted)",
  "syncEnabled": "boolean (default: true)",
  "lastSyncAt": "datetime",
  "syncStatus": "string (enum: active, paused, error, pending_auth)",
  "currentBalance": {
    "currency": "string",
    "amount": "decimal (8,2)"
  },
  "availableBalance": {
    "currency": "string",
    "amount": "decimal (8,2)"
  },
  "reconciliationStatus": "string (enum: unreconciled, partial, reconciled)",
  "reconciliationDate": "date",
  "isActive": "boolean (default: true)",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

#### Transaction (Bank)

Bank transaction imported from connected account.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "BankTransaction",
    "slug": "txn-2026-05-21-NL91-eur-5000-vendor-acme"
  },
  "id": "uuid",
  "bankAccount": {
    "register": "shillinq",
    "schema": "BankAccount",
    "objectId": "uuid"
  },
  "transactionDate": "date",
  "postDate": "date",
  "amount": {
    "currency": "string (ISO 4217)",
    "value": "decimal (12,2)"
  },
  "description": "string",
  "counterpartyName": "string",
  "counterpartyIban": "string",
  "counterpartyBic": "string",
  "referenceNumber": "string (bank reference)",
  "paymentReference": "string (structured like SEPA /USTRD/)",
  "transactionType": "string (enum: credit, debit, internal_transfer, fee, interest, adjustment)",
  "reconciliationStatus": "string (enum: unmatched, pending, matched, excluded)",
  "matchedExpense": {
    "register": "shillinq",
    "schema": "ExpenseLineItem",
    "objectId": "uuid"
  },
  "matchedInvoice": {
    "register": "shillinq",
    "schema": "InvoiceLine",
    "objectId": "uuid"
  },
  "matchedPayment": {
    "register": "shillinq",
    "schema": "Payment",
    "objectId": "uuid"
  },
  "matchRuleName": "string",
  "manuallyMatched": "boolean",
  "manualNoteByUser": {
    "register": "shillinq",
    "schema": "User",
    "objectId": "uuid"
  },
  "manualNoteDate": "datetime",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

#### Payment (Scheduled)

Represents a scheduled or executed payment (bill pay, transfer, etc.).

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "ScheduledPayment",
    "slug": "pay-2026-06-15-vendor-acme-eur-5000"
  },
  "id": "uuid",
  "paymentType": "string (enum: scheduled_bill_pay, immediate_transfer, recurring, internal_transfer, customer_payment)",
  "status": "string (enum: draft, scheduled, submitted, processing, completed, failed, cancelled)",
  "fromAccount": {
    "register": "shillinq",
    "schema": "BankAccount",
    "objectId": "uuid"
  },
  "toCounterparty": {
    "register": "shillinq",
    "schema": "Payee",
    "objectId": "uuid"
  },
  "paymentMethod": "string (enum: sepa_credit_transfer, ach, iDEAL, giropay, card, internal_transfer)",
  "amount": {
    "currency": "string (ISO 4217)",
    "value": "decimal (12,2)"
  },
  "scheduledDate": "date (nullable for immediate)",
  "executionDate": "date (nullable until executed)",
  "description": "string",
  "referenceNumber": "string",
  "linkedExpenses": [
    {
      "register": "shillinq",
      "schema": "ExpenseLineItem",
      "objectId": "uuid"
    }
  ],
  "linkedInvoices": [
    {
      "register": "shillinq",
      "schema": "InvoiceLine",
      "objectId": "uuid"
    }
  ],
  "approvalChain": {
    "register": "shillinq",
    "schema": "ApprovalChain",
    "objectId": "uuid"
  },
  "approvalStatus": "string (enum: pending_approval, approved, rejected, auto_approved)",
  "approvedBy": {
    "register": "shillinq",
    "schema": "User",
    "objectId": "uuid"
  },
  "approvalDate": "datetime",
  "createdBy": {
    "register": "shillinq",
    "schema": "User",
    "objectId": "uuid"
  },
  "createdAt": "datetime",
  "updatedAt": "datetime",
  "deletedAt": "datetime (soft delete)"
}
```

#### CashFlowForecast

13-week rolling cash flow projection.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "CashFlowForecast",
    "slug": "forecast-2026-05-21-13week"
  },
  "id": "uuid",
  "forecastPeriodStart": "date",
  "forecastPeriodEnd": "date",
  "weeksAhead": "integer (13)",
  "byAccount": {
    "register": "shillinq",
    "schema": "BankAccount",
    "objectId": "uuid"
  },
  "weeklyForecasts": [
    {
      "weekStart": "date",
      "weekEnd": "date",
      "openingBalance": "decimal (12,2)",
      "inflows": {
        "invoicesDue": "decimal (12,2)",
        "otherIncome": "decimal (12,2)",
        "total": "decimal (12,2)"
      },
      "outflows": {
        "billsDue": "decimal (12,2)",
        "payroll": "decimal (12,2)",
        "taxes": "decimal (12,2)",
        "otherExpenses": "decimal (12,2)",
        "total": "decimal (12,2)"
      },
      "netChange": "decimal (12,2)",
      "closingBalance": "decimal (12,2)",
      "confidence": "integer (0-100, percentage)"
    }
  ],
  "modelType": "string (enum: linear_regression, seasonality_adjusted, ml_based)",
  "historicalDataPoints": "integer",
  "accuracy": {
    "oneWeekRmse": "decimal (2,2)",
    "threeWeekRmse": "decimal (2,2)",
    "thirteenWeekRmse": "decimal (2,2)"
  },
  "generatedAt": "datetime",
  "updatedAt": "datetime"
}
```

#### CustomerPaymentPreference

Customer object with payment method and subscription state.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "CustomerPaymentPreference",
    "slug": "cust-acme-corp-iDEAL-recurring"
  },
  "id": "uuid",
  "customer": {
    "register": "shillinq",
    "schema": "Customer",
    "objectId": "uuid"
  },
  "primaryPaymentMethod": "string (enum: iDEAL, giropay, sepa_credit_transfer, card, bank_transfer, offline)",
  "mandateAgreement": {
    "register": "shillinq",
    "schema": "Mandate",
    "objectId": "uuid (nullable)"
  },
  "cardToken": "string (PCI-DSS encrypted, nullable)",
  "iban": "string (nullable)",
  "bic": "string (nullable)",
  "subscriptionState": "string (enum: none, active, paused, cancelled, expired)",
  "subscriptionStartDate": "date (nullable)",
  "subscriptionEndDate": "date (nullable)",
  "frequencyDays": "integer (nullable, e.g. 30 for monthly)",
  "nextPaymentDate": "date (nullable)",
  "failureRetryCount": "integer (default: 0)",
  "lastPaymentStatus": "string (enum: success, failed, pending, cancelled)",
  "lastPaymentDate": "datetime (nullable)",
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

#### ReconciliationRule

Auto-matching configuration for bank reconciliation.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "ReconciliationRule",
    "slug": "rule-vendor-acme-exact-amount"
  },
  "id": "uuid",
  "name": "string (required)",
  "description": "string",
  "isActive": "boolean (default: true)",
  "bankAccount": {
    "register": "shillinq",
    "schema": "BankAccount",
    "objectId": "uuid"
  },
  "priority": "integer (1=highest)",
  "matchingCriteria": {
    "amountType": "string (enum: exact, range)",
    "amountMin": "decimal (12,2)",
    "amountMax": "decimal (12,2)",
    "descriptionKeywords": ["string"],
    "counterpartyPattern": "string (regex nullable)",
    "transactionTypeFilter": ["string"]
  },
  "matchTarget": "string (enum: expense, invoice, payment, journal_entry, manual_review)",
  "targetFieldMapping": {
    "expenseCategory": {
      "register": "shillinq",
      "schema": "ExpenseCategory",
      "objectId": "uuid (nullable)"
    },
    "description": "string (nullable)"
  },
  "autoApply": "boolean (default: true)",
  "requiresApproval": "boolean (default: false)",
  "createdBy": {
    "register": "shillinq",
    "schema": "User",
    "objectId": "uuid"
  },
  "createdAt": "datetime",
  "updatedAt": "datetime"
}
```

#### Bank (Reference)

Bank details for lookup/display.

```json
{
  "@self": {
    "register": "shillinq",
    "schema": "Bank",
    "slug": "nl-abn-amro"
  },
  "id": "uuid",
  "name": "string",
  "bic": "string",
  "countryCode": "string (ISO 3166-1 alpha-2)",
  "plaidInstitutionId": "string (nullable)",
  "isSupported": "boolean"
}
```

### Seed Data

**3-5 realistic objects per entity for development/testing:**

#### BankAccount Seed

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "BankAccount",
      "slug": "iban-nl91-abna-0417-1643-00"
    },
    "id": "ba-uuid-1",
    "name": "Operational Main Account",
    "description": "Primary checking account for day-to-day operations",
    "iban": "NL91ABNA0417164300",
    "bic": "ABNANL2A",
    "currency": "EUR",
    "accountType": "checking",
    "bank": { "register": "shillinq", "schema": "Bank", "objectId": "bank-uuid-1" },
    "syncEnabled": true,
    "syncStatus": "active",
    "currentBalance": { "currency": "EUR", "amount": "85000.00" },
    "availableBalance": { "currency": "EUR", "amount": "82500.00" },
    "reconciliationStatus": "reconciled",
    "reconciliationDate": "2026-05-21",
    "isActive": true,
    "createdAt": "2026-01-15T09:00:00Z",
    "updatedAt": "2026-05-21T08:30:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "BankAccount",
      "slug": "iban-nl39-rabo-0195-6969-69"
    },
    "id": "ba-uuid-2",
    "name": "Savings Reserve Account",
    "description": "Tax provision and emergency reserve",
    "iban": "NL39RABO0195696969",
    "bic": "RABONL2U",
    "currency": "EUR",
    "accountType": "savings",
    "bank": { "register": "shillinq", "schema": "Bank", "objectId": "bank-uuid-2" },
    "syncEnabled": true,
    "syncStatus": "active",
    "currentBalance": { "currency": "EUR", "amount": "250000.00" },
    "availableBalance": { "currency": "EUR", "amount": "250000.00" },
    "reconciliationStatus": "reconciled",
    "reconciliationDate": "2026-05-21",
    "isActive": true,
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-05-21T08:30:00Z"
  }
]
```

#### BankTransaction Seed

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "BankTransaction",
      "slug": "txn-2026-05-20-NL91-eur-15000-invoice-acme"
    },
    "id": "txn-uuid-1",
    "bankAccount": { "register": "shillinq", "schema": "BankAccount", "objectId": "ba-uuid-1" },
    "transactionDate": "2026-05-20",
    "postDate": "2026-05-21",
    "amount": { "currency": "EUR", "value": "15000.00" },
    "description": "Invoice INV-2026-001 Acme Corp services",
    "counterpartyName": "Acme Corp B.V.",
    "counterpartyIban": "NL91ABNA0417164300",
    "referenceNumber": "TXN-2026-NL91-1001",
    "paymentReference": "/USTRD/INV-2026-001//Acme services/",
    "transactionType": "credit",
    "reconciliationStatus": "matched",
    "matchedInvoice": { "register": "shillinq", "schema": "InvoiceLine", "objectId": "inv-line-uuid-1" },
    "matchRuleName": "Auto-match: Invoice amount exact",
    "createdAt": "2026-05-21T08:00:00Z",
    "updatedAt": "2026-05-21T08:15:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "BankTransaction",
      "slug": "txn-2026-05-20-NL91-eur-8500-vendor-office"
    },
    "id": "txn-uuid-2",
    "bankAccount": { "register": "shillinq", "schema": "BankAccount", "objectId": "ba-uuid-1" },
    "transactionDate": "2026-05-20",
    "postDate": "2026-05-21",
    "amount": { "currency": "EUR", "value": "-8500.00" },
    "description": "Office Supplies B.V. - Invoice P-2859",
    "counterpartyName": "Office Supplies B.V.",
    "counterpartyIban": "NL39RABO0195696969",
    "referenceNumber": "TXN-2026-NL91-1002",
    "paymentReference": "/USTRD/P-2859/Office supplies/",
    "transactionType": "debit",
    "reconciliationStatus": "matched",
    "matchedExpense": { "register": "shillinq", "schema": "ExpenseLineItem", "objectId": "exp-line-uuid-1" },
    "matchRuleName": "Auto-match: Vendor+amount range",
    "createdAt": "2026-05-21T08:00:00Z",
    "updatedAt": "2026-05-21T08:15:00Z"
  }
]
```

#### ScheduledPayment Seed

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ScheduledPayment",
      "slug": "pay-2026-06-15-vendor-acme-eur-5000"
    },
    "id": "pay-uuid-1",
    "paymentType": "scheduled_bill_pay",
    "status": "scheduled",
    "fromAccount": { "register": "shillinq", "schema": "BankAccount", "objectId": "ba-uuid-1" },
    "toCounterparty": { "register": "shillinq", "schema": "Payee", "objectId": "payee-uuid-1" },
    "paymentMethod": "sepa_credit_transfer",
    "amount": { "currency": "EUR", "value": "5000.00" },
    "scheduledDate": "2026-06-15",
    "description": "Vendor payment - Monthly services",
    "referenceNumber": "REF-2026-0542",
    "linkedExpenses": [
      { "register": "shillinq", "schema": "ExpenseLineItem", "objectId": "exp-line-uuid-2" }
    ],
    "approvalStatus": "approved",
    "approvedBy": { "register": "shillinq", "schema": "User", "objectId": "user-uuid-1" },
    "approvalDate": "2026-05-21T14:30:00Z",
    "createdBy": { "register": "shillinq", "schema": "User", "objectId": "user-uuid-2" },
    "createdAt": "2026-05-21T10:00:00Z",
    "updatedAt": "2026-05-21T14:30:00Z"
  },
  {
    "@self": {
      "register": "shillinq",
      "schema": "ScheduledPayment",
      "slug": "pay-2026-06-01-payroll-eur-45000"
    },
    "id": "pay-uuid-2",
    "paymentType": "scheduled_bill_pay",
    "status": "scheduled",
    "fromAccount": { "register": "shillinq", "schema": "BankAccount", "objectId": "ba-uuid-1" },
    "paymentMethod": "sepa_credit_transfer",
    "amount": { "currency": "EUR", "value": "45000.00" },
    "scheduledDate": "2026-06-01",
    "description": "Monthly payroll - June 2026",
    "referenceNumber": "PAYROLL-2026-06",
    "approvalStatus": "approved",
    "approvedBy": { "register": "shillinq", "schema": "User", "objectId": "user-uuid-1" },
    "approvalDate": "2026-05-21T08:00:00Z",
    "createdBy": { "register": "shillinq", "schema": "User", "objectId": "user-uuid-1" },
    "createdAt": "2026-05-21T07:30:00Z",
    "updatedAt": "2026-05-21T08:00:00Z"
  }
]
```

#### CustomerPaymentPreference Seed

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "CustomerPaymentPreference",
      "slug": "cust-acme-corp-iDEAL-recurring"
    },
    "id": "cust-pref-uuid-1",
    "customer": { "register": "shillinq", "schema": "Customer", "objectId": "customer-uuid-1" },
    "primaryPaymentMethod": "iDEAL",
    "subscriptionState": "active",
    "subscriptionStartDate": "2026-03-15",
    "frequencyDays": 30,
    "nextPaymentDate": "2026-06-15",
    "lastPaymentStatus": "success",
    "lastPaymentDate": "2026-05-15T10:30:00Z",
    "createdAt": "2026-03-15T09:00:00Z",
    "updatedAt": "2026-05-15T10:30:00Z"
  }
]
```

#### ReconciliationRule Seed

```json
[
  {
    "@self": {
      "register": "shillinq",
      "schema": "ReconciliationRule",
      "slug": "rule-vendor-acme-exact-amount"
    },
    "id": "rule-uuid-1",
    "name": "Auto-match Acme Corp by exact amount",
    "description": "Match transactions for Acme Corp B.V. when amount matches invoice exactly",
    "isActive": true,
    "bankAccount": { "register": "shillinq", "schema": "BankAccount", "objectId": "ba-uuid-1" },
    "priority": 1,
    "matchingCriteria": {
      "amountType": "exact",
      "descriptionKeywords": ["Acme", "ACME"],
      "counterpartyPattern": "(?i)acme.*corp",
      "transactionTypeFilter": ["credit", "debit"]
    },
    "matchTarget": "invoice",
    "autoApply": true,
    "requiresApproval": false,
    "createdBy": { "register": "shillinq", "schema": "User", "objectId": "user-uuid-1" },
    "createdAt": "2026-02-01T10:00:00Z",
    "updatedAt": "2026-02-01T10:00:00Z"
  }
]
```

## Workflow Patterns

### Bank Account Sync Workflow

```
User clicks "Connect Bank Account"
  ↓ Opens Plaid Link modal
  ↓ User authorizes bank connection
  ↓ Plaid returns accessToken + accountId
  ↓ Backend stores encrypted token in BankAccount.plaidAccountId
  ↓ Background job: fetch last 90 days transactions
  ↓ Each transaction → BankTransaction object
  ↓ Dashboard shows "New transactions imported: N"
  ↓ User manually reconciles or enables auto-match rules
```

### Payment Scheduling Workflow

```
Accounts manager creates ScheduledPayment
  ↓ Selects: from account, to payee, amount, date, linked expenses
  ↓ System routes to approval chain (if amount > threshold)
  ↓ Approver reviews + clicks "Approve"
  ↓ ScheduledPayment.status = "approved"
  ↓ Background job on scheduled date: submit to bank API (SEPA/ACH)
  ↓ Bank returns confirmation ID
  ↓ ScheduledPayment.status = "submitted", executionDate populated
  ↓ Webhook from bank on settlement: status = "completed"
```

### Auto-Reconciliation Workflow

```
Bank statement imported
  ↓ For each BankTransaction:
    - Load active ReconciliationRules (sorted by priority)
    - For each rule, check if criteria match
    - If match found:
      - Set BankTransaction.matchedInvoice/Expense/Payment
      - Set BankTransaction.matchRuleName
      - Set BankTransaction.reconciliationStatus = "matched"
    - If no match: reconciliationStatus = "pending" (manual review)
  ↓ Finance manager reviews unmatched (5-10 transactions typically)
  ↓ Manager clicks "Mark reconciled" for excluded transactions
  ↓ All BankTransactions now have reconciliationStatus in [matched, excluded]
  ↓ Close out month
```

### Cash Flow Forecast Generation

```
Daily job at 6 AM UTC:
  ↓ For each BankAccount:
    - Load last 90 days of BankTransactions grouped by week
    - Fetch unpaid invoices (Invoice.status != "paid")
    - Fetch unpaid bills (Expense.status != "paid")
    - Fetch scheduled payments (ScheduledPayment.status = "scheduled" within 13 weeks)
    - Apply linear regression model to historical outflows
    - Calculate weekly inflows = sum(unpaid invoices due that week)
    - Calculate weekly outflows = sum(unpaid bills + historical avg + scheduled payments)
    - Generate 13 CashFlowForecast.weeklyForecasts entries
    - Calculate confidence % based on RMSE
  ↓ Store as CashFlowForecast object
  ↓ Dashboard widget reads latest forecast
```

## UI/UX Architecture

### Key Pages

**1. Treasury Dashboard**
- Top cards: total liquidity (sum all accounts), month-end projection, cash runway
- Multi-account balance chart (stacked area)
- Transaction list (filterable by account, date, counterparty)
- Alerts: unmatched transactions, scheduled payments due today, low balance warnings

**2. Bank Accounts Index**
- List view: account name, IBAN, current balance, last sync time, sync status
- Add account button → Plaid Link modal
- Row actions: view, edit, pause sync, remove

**3. Bank Account Detail**
- Balance cards (current + available)
- Transaction import history
- Transaction list with auto-match status
- Reconciliation status % complete

**4. Reconciliation Manager**
- Unmatched transactions table
- For each: edit match, create rule, mark excluded
- Rules list: manage active/inactive, priority order

**5. Payment Scheduler**
- Create button → form: from, to, amount, date, linked expenses
- Draft payments list → submit for approval
- Approved payments list → execution countdown
- Completed payments with bank confirmation

**6. Cash Flow Dashboard**
- 13-week forecast chart (area chart: opening + inflows - outflows = closing)
- Weekly detail table (drill-down per week)
- Sensitivity analysis: ±10% variance bands
- Variance tracking (actual vs. forecast from 4 weeks ago)

## Reuse Analysis

### Leveraged OpenRegister Services

| Service | Purpose | Why Build Custom |
|---------|---------|------------------|
| ObjectService | CRUD for all treasury objects | Standard CRUD, no custom logic |
| SchemaService | Schema validation | Schemas defined in register.json |
| ImportService | Bank statement CSV import | Need custom bank CSV parsing + Plaid Link |
| ExportService | Export reconciliation reports | Standard export (no custom logic) |
| AuditTrailService | Payment audit trail | Automatic per ADR-001 |
| AuthorizationService | Role-based payment approval | RBAC for treasurer/approver roles |
| FileService | Attach bank statement PDFs | Standard file attachment |
| WebhookService | Bank notification webhooks | Receive webhooks from Plaid, payment processor |
| NotificationService | Alerts (low balance, sync errors) | Trigger notifications on BankAccount events |
| TaskService | Tasks for reconciliation follow-up | Create tasks for unmatched items |

### Custom Business Logic Required

1. **Plaid Link Integration** — backend service to exchange auth token for account sync
2. **Bank Transaction Import** — parse bank CSV/OFX, create BankTransaction objects, handle duplicates
3. **Auto-Matching Engine** — evaluate ReconciliationRules, match transactions to invoices/expenses
4. **Payment Submission** — format SEPA/ACH payment, submit to bank API, handle rejections
5. **Cash Flow Forecast Model** — linear regression + seasonality adjustment
6. **Payment Approval Chain** — route to approvers based on amount thresholds
7. **Reconciliation Report** — generate month-end variance report (actual vs. forecast)

## Implementation Phases

### Phase 1 (MVP): Core Visibility + Simple Payments

**Deliverables:**
- BankAccount, BankTransaction, ScheduledPayment entities
- Plaid Link integration for account sync
- Basic reconciliation (manual matching)
- Payment scheduling (single + bulk)
- Dashboard: balance cards, transaction list

**Timeline:** 6 weeks  
**Team:** 1 backend (PHP) + 1 frontend (Vue) + 1 product

### Phase 2: Automation + Forecasting

**Deliverables:**
- ReconciliationRule auto-matching
- CashFlowForecast model
- iDEAL checkout integration
- Branded payment pages
- Customer payment preference management

**Timeline:** 6 weeks  
**Team:** 1 backend + 1 frontend + 1 product

### Phase 3: Advanced Channels + Multi-Currency

**Deliverables:**
- Credit card acquiring (Visa/MC/Amex)
- Multi-currency conversion
- Giropay + other regional channels
- Payment optimization engine

**Timeline:** 8 weeks  
**Team:** 1 backend + 1 frontend + 1 payment ops
