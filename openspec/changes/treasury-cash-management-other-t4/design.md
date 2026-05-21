# Design: Treasury & Cash Management — Shillinq — Other T4

**Status:** Design  
**Change:** treasury-cash-management-other-t4  
**Last Updated:** 2026-05-21

---

## Overview

This design document specifies the data structures, API endpoints, frontend components, and business logic for Phase 1 treasury features in Shillinq.

---

## Part I: Data Model

All features operate on existing Shillinq entities (no new entity definitions):

### Primary Entities Used

#### BankAccount (existing)
Represents a company bank account with IBAN and bank connection details.

**Key Fields:**
- `id`: UUID  
- `iban`: String (IBAN format, validated per ISO 13616)  
- `accountHolder`: String (company name)  
- `bankName`: String (e.g., "ING Netherlands")  
- `currency`: String (EUR, GBP, USD)  
- `status`: String (active, archived)  
- `balance`: MonetaryAmount (current balance, updated from bank import)  
- `lastReconciled`: DateTime  
- `reconciliationRule`: String (matching algorithm preference)  
- `reserve_minimum`: MonetaryAmount (threshold for alerts)  
- `administration_id`: UUID (foreign key via OpenRegister relation)

#### GeneralLedgerEntry (existing)
Double-entry bookkeeping records linked to transactions.

**Fields Used:**
- `id`: UUID  
- `date`: Date  
- `account`: GeneralLedgerAccount (debit/credit)  
- `amount`: MonetaryAmount  
- `description`: String  
- `reference`: String (invoice number, bank reference)  
- `transaction_id`: UUID (link to Payment or other source)  
- `bank_entry_id`: UUID (link to imported bank statement entry, added for reconciliation)

#### Payment (existing)
Represents a payment transaction (outbound/inbound).

**Fields Used:**
- `id`: UUID  
- `date`: Date  
- `amount`: MonetaryAmount  
- `status`: String (pending, authorized, executed, settled, cancelled)  
- `counterparty`: Supplier or Person  
- `reference`: String (invoice/order number)  
- `iban`: String (destination IBAN for outbound)  
- `bank_account_id`: UUID (source account)  
- `netting_group_id`: UUID (new: for netting analysis)  
- `provider_id`: UUID (new: payment service provider)  
- `approval_chain_id`: UUID (workflow for separation of duties)

#### Supplier (existing)
Vendor/supplier information.

**Fields Used:**
- `id`: UUID  
- `name`: String  
- `iban`: String (primary IBAN for payments)  
- `iban_verified`: Boolean (new: verification status)  
- `iban_verified_date`: DateTime (new: when verified)  
- `iban_verified_by`: User (new: audit trail)  
- `payment_provider_id`: UUID (new: preferred provider for this supplier)

#### Loan (existing)
Debt instrument.

**Fields Used:**
- `id`: UUID  
- `principal`: MonetaryAmount  
- `rate`: Decimal (annual interest rate, e.g., 5.25)  
- `term_months`: Integer  
- `origination_date`: Date  
- `maturity_date`: Date  
- `is_eligible_for_refinancing`: Boolean (new: computed field)  
- `refinancing_score`: Float (new: market rate delta, used for sorting)

#### ScheduledPayment (existing)
Recurring or future payment arrangement.

**Fields Used:**
- `id`: UUID  
- `frequency`: String (once, monthly, quarterly)  
- `next_due`: Date  
- `amount`: MonetaryAmount  
- `counterparty`: Supplier or Person  
- `status`: String (active, paused, completed)

#### Role & Authorization (existing)
Permission definitions for role-based access control.

**Fields Used:**
- `id`: UUID  
- `name`: String (e.g., "requester", "approver", "payer")  
- `permissions`: Array<String>

#### PaymentBatch (new, minimal)
Groups multiple payments for bulk approval/execution.

**Definition:**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "PaymentBatch",
  "description": "Batch of payments awaiting approval and execution",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "administration_id": {
      "type": "string",
      "format": "uuid"
    },
    "created_by": {
      "type": "string",
      "format": "uuid",
      "description": "User who created the batch"
    },
    "created_date": {
      "type": "string",
      "format": "date-time"
    },
    "status": {
      "type": "string",
      "enum": ["draft", "approved", "executed", "cancelled"],
      "default": "draft"
    },
    "total_amount": {
      "$ref": "#/$defs/MonetaryAmount"
    },
    "payment_ids": {
      "type": "array",
      "items": {
        "type": "string",
        "format": "uuid"
      }
    },
    "approval_chain_id": {
      "type": "string",
      "format": "uuid"
    },
    "approved_by": {
      "type": "string",
      "format": "uuid"
    },
    "approved_date": {
      "type": "string",
      "format": "date-time"
    },
    "executed_date": {
      "type": "string",
      "format": "date-time"
    },
    "notes": {
      "type": "string"
    }
  },
  "required": ["administration_id", "created_by", "created_date", "status"],
  "$defs": {
    "MonetaryAmount": {
      "type": "object",
      "properties": {
        "amount": { "type": "number" },
        "currency": { "type": "string", "pattern": "^[A-Z]{3}$" }
      },
      "required": ["amount", "currency"]
    }
  }
}
```

#### PaymentProvider (new, minimal)
Configuration for external payment service (Wise, Stripe, SEPA Core, etc.).

**Definition:**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "PaymentProvider",
  "description": "External payment service configuration",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "name": {
      "type": "string",
      "description": "e.g., Wise, Stripe, SEPA Core, etc."
    },
    "type": {
      "type": "string",
      "enum": ["bank", "fintech", "internal"]
    },
    "administration_id": {
      "type": "string",
      "format": "uuid"
    },
    "supported_currencies": {
      "type": "array",
      "items": { "type": "string" },
      "description": "e.g., [\"EUR\", \"GBP\", \"USD\"]"
    },
    "api_key_encrypted": {
      "type": "string",
      "description": "Encrypted API key/credentials"
    },
    "is_active": {
      "type": "boolean"
    },
    "created_date": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["name", "type", "administration_id", "supported_currencies", "is_active"]
}
```

#### CashFlowForecast (new, minimal)
Computed cash flow projection based on committed transactions.

**Definition:**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "CashFlowForecast",
  "description": "Projected cash position over time",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "administration_id": {
      "type": "string",
      "format": "uuid"
    },
    "base_date": {
      "type": "string",
      "format": "date"
    },
    "horizon_days": {
      "type": "integer",
      "description": "30, 60, or 90"
    },
    "projections": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "date": { "type": "string", "format": "date" },
          "opening_balance": { "$ref": "#/$defs/MonetaryAmount" },
          "inflows": { "$ref": "#/$defs/MonetaryAmount" },
          "outflows": { "$ref": "#/$defs/MonetaryAmount" },
          "closing_balance": { "$ref": "#/$defs/MonetaryAmount" }
        }
      }
    },
    "assumptions": {
      "type": "object",
      "description": "Metadata about forecast (which invoices included, adjustments made, etc.)"
    }
  },
  "required": ["administration_id", "base_date", "horizon_days", "projections"],
  "$defs": {
    "MonetaryAmount": {
      "type": "object",
      "properties": {
        "amount": { "type": "number" },
        "currency": { "type": "string" }
      }
    }
  }
}
```

#### BankStatementEntry (new, minimal)
Imported bank transaction (SEPA MT940 or CAMT.053).

**Definition:**
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "title": "BankStatementEntry",
  "description": "Single transaction from bank statement",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "bank_account_id": {
      "type": "string",
      "format": "uuid"
    },
    "import_date": {
      "type": "string",
      "format": "date-time"
    },
    "statement_date": {
      "type": "string",
      "format": "date"
    },
    "entry_date": {
      "type": "string",
      "format": "date"
    },
    "amount": {
      "$ref": "#/$defs/MonetaryAmount"
    },
    "counterparty_name": {
      "type": "string"
    },
    "counterparty_iban": {
      "type": "string"
    },
    "reference": {
      "type": "string"
    },
    "transaction_code": {
      "type": "string",
      "description": "MT940 transaction code (e.g., 'N005' for SEPA transfer)"
    },
    "matched_ledger_entry_id": {
      "type": "string",
      "format": "uuid"
    },
    "matched_confidence": {
      "type": "number",
      "description": "0.0 to 1.0, confidence of automated match"
    },
    "is_reconciled": {
      "type": "boolean"
    }
  },
  "required": ["bank_account_id", "statement_date", "entry_date", "amount", "reference"],
  "$defs": {
    "MonetaryAmount": {
      "type": "object",
      "properties": {
        "amount": { "type": "number" },
        "currency": { "type": "string" }
      }
    }
  }
}
```

---

## Seed Data (Dutch Examples)

### BankAccount (3 examples)

```json
{
  "id": "b1a1c1d1-1111-1111-1111-111111111111",
  "iban": "NL91ABNA0417164300",
  "accountHolder": "Voorbeeld BV",
  "bankName": "ABN AMRO",
  "currency": "EUR",
  "status": "active",
  "balance": { "amount": 125000.00, "currency": "EUR" },
  "lastReconciled": "2026-05-21T09:00:00Z",
  "reconciliationRule": "automated_match_high_confidence",
  "reserve_minimum": { "amount": 50000.00, "currency": "EUR" },
  "administration_id": "adm1"
}
```

```json
{
  "id": "b1a1c1d1-2222-2222-2222-222222222222",
  "iban": "NL31INGBNL2A08",
  "accountHolder": "Voorbeeld BV",
  "bankName": "ING Netherlands",
  "currency": "EUR",
  "status": "active",
  "balance": { "amount": 45000.00, "currency": "EUR" },
  "lastReconciled": "2026-05-21T09:00:00Z",
  "reconciliationRule": "automated_match_high_confidence",
  "reserve_minimum": { "amount": 20000.00, "currency": "EUR" },
  "administration_id": "adm1"
}
```

```json
{
  "id": "b1a1c1d1-3333-3333-3333-333333333333",
  "iban": "DE89370400440532013000",
  "accountHolder": "Voorbeeld GmbH",
  "bankName": "Deutsche Bank",
  "currency": "EUR",
  "status": "active",
  "balance": { "amount": 32000.00, "currency": "EUR" },
  "lastReconciled": "2026-05-19T09:00:00Z",
  "reconciliationRule": "manual_review",
  "reserve_minimum": { "amount": 15000.00, "currency": "EUR" },
  "administration_id": "adm2"
}
```

### Supplier (4 examples with IBAN verification)

```json
{
  "id": "supp-001",
  "name": "Zorgdiensten de Toekomst",
  "iban": "NL75ABNA0588564432",
  "iban_verified": true,
  "iban_verified_date": "2026-05-15T10:30:00Z",
  "iban_verified_by": "user-controller-001",
  "payment_provider_id": "prov-sepa-core"
}
```

```json
{
  "id": "supp-002",
  "name": "Transportmaatschappij Noord BV",
  "iban": "NL56RABO0123456789",
  "iban_verified": true,
  "iban_verified_date": "2026-05-10T14:00:00Z",
  "iban_verified_by": "user-treasurer-001",
  "payment_provider_id": "prov-sepa-core"
}
```

```json
{
  "id": "supp-003",
  "name": "Europese Distributie Ltd",
  "iban": "GB82WEST12345698765432",
  "iban_verified": true,
  "iban_verified_date": "2026-05-20T09:15:00Z",
  "iban_verified_by": "user-controller-001",
  "payment_provider_id": "prov-wise"
}
```

```json
{
  "id": "supp-004",
  "name": "Kantoorbenodigdheden +Plus",
  "iban": "NL91ABNC0234567890",
  "iban_verified": false,
  "iban_verified_date": null,
  "iban_verified_by": null,
  "payment_provider_id": null
}
```

### Payment (3 examples showing netting and provider usage)

```json
{
  "id": "pay-001",
  "date": "2026-05-21",
  "amount": { "amount": 2500.00, "currency": "EUR" },
  "status": "settled",
  "counterparty": { "type": "Supplier", "id": "supp-001" },
  "reference": "INV-2026-00451",
  "iban": "NL75ABNA0588564432",
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "netting_group_id": "netting-group-042",
  "provider_id": "prov-sepa-core"
}
```

```json
{
  "id": "pay-002",
  "date": "2026-05-21",
  "amount": { "amount": 1800.00, "currency": "EUR" },
  "status": "pending",
  "counterparty": { "type": "Supplier", "id": "supp-002" },
  "reference": "INV-2026-00453",
  "iban": "NL56RABO0123456789",
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "netting_group_id": null,
  "provider_id": "prov-sepa-core"
}
```

```json
{
  "id": "pay-003",
  "date": "2026-05-20",
  "amount": { "amount": 3200.00, "currency": "GBP" },
  "status": "authorized",
  "counterparty": { "type": "Supplier", "id": "supp-003" },
  "reference": "ORDER-EU-789",
  "iban": "GB82WEST12345698765432",
  "bank_account_id": "b1a1c1d1-2222-2222-2222-222222222222",
  "netting_group_id": null,
  "provider_id": "prov-wise"
}
```

### PaymentBatch (1 example)

```json
{
  "id": "batch-2026-05-21-001",
  "administration_id": "adm1",
  "created_by": "user-treasurer-001",
  "created_date": "2026-05-21T08:30:00Z",
  "status": "approved",
  "total_amount": { "amount": 18750.00, "currency": "EUR" },
  "payment_ids": ["pay-001", "pay-002", "pay-004", "pay-005"],
  "approval_chain_id": "approval-chain-001",
  "approved_by": "user-cfo-001",
  "approved_date": "2026-05-21T11:00:00Z",
  "executed_date": null,
  "notes": "Weekly supplier payments, all IBAN verified"
}
```

### PaymentProvider (3 examples)

```json
{
  "id": "prov-sepa-core",
  "name": "SEPA Core Direct Debit",
  "type": "bank",
  "administration_id": "adm1",
  "supported_currencies": ["EUR"],
  "api_key_encrypted": "encrypted::xyz123abc",
  "is_active": true,
  "created_date": "2026-01-15T09:00:00Z"
}
```

```json
{
  "id": "prov-wise",
  "name": "Wise (TransferWise)",
  "type": "fintech",
  "administration_id": "adm1",
  "supported_currencies": ["EUR", "GBP", "USD", "SEK"],
  "api_key_encrypted": "encrypted::abc456def",
  "is_active": true,
  "created_date": "2026-02-01T10:00:00Z"
}
```

```json
{
  "id": "prov-internal",
  "name": "Internal - Manual Processing",
  "type": "internal",
  "administration_id": "adm1",
  "supported_currencies": ["EUR"],
  "api_key_encrypted": null,
  "is_active": true,
  "created_date": "2026-01-01T00:00:00Z"
}
```

### CashFlowForecast (1 example, 30-day)

```json
{
  "id": "forecast-2026-05-21-30d",
  "administration_id": "adm1",
  "base_date": "2026-05-21",
  "horizon_days": 30,
  "projections": [
    {
      "date": "2026-05-21",
      "opening_balance": { "amount": 170000.00, "currency": "EUR" },
      "inflows": { "amount": 15000.00, "currency": "EUR" },
      "outflows": { "amount": 8500.00, "currency": "EUR" },
      "closing_balance": { "amount": 176500.00, "currency": "EUR" }
    },
    {
      "date": "2026-05-22",
      "opening_balance": { "amount": 176500.00, "currency": "EUR" },
      "inflows": { "amount": 0.00, "currency": "EUR" },
      "outflows": { "amount": 5200.00, "currency": "EUR" },
      "closing_balance": { "amount": 171300.00, "currency": "EUR" }
    },
    {
      "date": "2026-05-28",
      "opening_balance": { "amount": 165000.00, "currency": "EUR" },
      "inflows": { "amount": 45000.00, "currency": "EUR" },
      "outflows": { "amount": 12000.00, "currency": "EUR" },
      "closing_balance": { "amount": 198000.00, "currency": "EUR" }
    }
  ],
  "assumptions": {
    "included_invoices": 125,
    "manual_adjustments": 2,
    "method": "committed_transactions",
    "currency_rates_as_of": "2026-05-21"
  }
}
```

### BankStatementEntry (3 examples)

```json
{
  "id": "bse-001",
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "import_date": "2026-05-21T09:30:00Z",
  "statement_date": "2026-05-21",
  "entry_date": "2026-05-21",
  "amount": { "amount": -2500.00, "currency": "EUR" },
  "counterparty_name": "Zorgdiensten de Toekomst",
  "counterparty_iban": "NL75ABNA0588564432",
  "reference": "INV-2026-00451",
  "transaction_code": "N005",
  "matched_ledger_entry_id": "gle-001",
  "matched_confidence": 0.98,
  "is_reconciled": true
}
```

```json
{
  "id": "bse-002",
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "import_date": "2026-05-21T09:30:00Z",
  "statement_date": "2026-05-21",
  "entry_date": "2026-05-21",
  "amount": { "amount": 8750.00, "currency": "EUR" },
  "counterparty_name": "Klant B.V.",
  "counterparty_iban": "NL91RABO0123456789",
  "reference": "ORDER-2026-00123",
  "transaction_code": "N005",
  "matched_ledger_entry_id": null,
  "matched_confidence": 0.0,
  "is_reconciled": false
}
```

```json
{
  "id": "bse-003",
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "import_date": "2026-05-21T09:30:00Z",
  "statement_date": "2026-05-21",
  "entry_date": "2026-05-21",
  "amount": { "amount": -45.00, "currency": "EUR" },
  "counterparty_name": "ABN AMRO",
  "counterparty_iban": "NL91ABNA0417164300",
  "reference": "FEE - Monthly maintenance",
  "transaction_code": "N034",
  "matched_ledger_entry_id": null,
  "matched_confidence": 0.0,
  "is_reconciled": false
}
```

---

## Part II: API Design

### REST Endpoints (OpenAPI 3.0)

All endpoints follow `/index.php/apps/shillinq/api/` pattern.

#### Bank Reconciliation

**GET /api/bank-reconciliation/status**
Fetch reconciliation status for all bank accounts.

```
Response:
{
  "accounts": [
    {
      "account_id": "b1a1c1d1-1111-1111-1111-111111111111",
      "iban": "NL91ABNA0417164300",
      "total_entries_imported": 487,
      "total_matched": 465,
      "total_unmatched": 22,
      "match_rate": 0.955,
      "last_reconciled": "2026-05-21T09:00:00Z"
    }
  ],
  "total": 3,
  "page": 1,
  "pages": 1
}
```

**POST /api/bank-reconciliation/import**
Import bank statement (MT940 or CAMT.053 file).

```
Request:
{
  "bank_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "file": <multipart file>,
  "format": "mt940" | "camt053"
}

Response:
{
  "import_id": "import-2026-05-21-001",
  "entries_imported": 42,
  "entries_matched": 38,
  "entries_unmatched": 4,
  "status": "completed"
}
```

**GET /api/bank-reconciliation/entries**
List bank statement entries with reconciliation status.

```
Query params:
  - bank_account_id (required)
  - is_reconciled: boolean (optional, filter)
  - _page: int (default 1)
  - _limit: int (default 50)

Response:
{
  "entries": [ { BankStatementEntry }, ... ],
  "total": 487,
  "page": 1,
  "pages": 10
}
```

**POST /api/bank-reconciliation/match-manual**
Manually assign a bank entry to a ledger entry.

```
Request:
{
  "bank_entry_id": "bse-002",
  "ledger_entry_id": "gle-123"
}

Response:
{
  "matched": true,
  "confidence": 1.0,
  "bank_entry": { BankStatementEntry },
  "ledger_entry": { GeneralLedgerEntry }
}
```

#### Cash Flow Forecasting

**GET /api/cash-flow/forecast**
Compute or retrieve cash flow forecast.

```
Query params:
  - administration_id (required)
  - horizon_days: 30 | 60 | 90 (default 30)
  - as_of_date: YYYY-MM-DD (default today)

Response:
{
  "forecast": { CashFlowForecast },
  "generated_at": "2026-05-21T10:00:00Z"
}
```

**POST /api/cash-flow/forecast-with-adjustment**
Forecast including manual adjustments (e.g., planned bonus, expected loan disbursement).

```
Request:
{
  "administration_id": "adm1",
  "horizon_days": 90,
  "adjustments": [
    { "date": "2026-06-15", "amount": 30000, "currency": "EUR", "description": "Dividend payout" }
  ]
}

Response:
{
  "forecast": { CashFlowForecast with adjustments },
  "generated_at": "2026-05-21T10:00:00Z"
}
```

#### Loan Refinancing

**GET /api/loans/refinancing-analysis**
List loans eligible for refinancing with market rate comparison.

```
Query params:
  - administration_id (required)
  - _page: int (default 1)
  - _limit: int (default 50)

Response:
{
  "loans": [
    {
      "id": "loan-001",
      "principal": 100000,
      "currency": "EUR",
      "current_rate": 5.5,
      "market_rate": 4.2,
      "is_eligible": true,
      "refinancing_score": 1.3,
      "estimated_annual_savings": 1300,
      "maturity_date": "2030-06-15"
    },
    ...
  ],
  "total": 2,
  "page": 1,
  "pages": 1
}
```

#### IBAN Verification

**POST /api/suppliers/verify-iban**
Validate supplier IBAN and store verification result.

```
Request:
{
  "supplier_id": "supp-004",
  "iban": "NL91ABNC0234567890"
}

Response:
{
  "iban": "NL91ABNC0234567890",
  "is_valid": true,
  "is_sepa_enabled": true,
  "country": "NL",
  "bank_name": "ABN AMRO"
}
```

**GET /api/suppliers/:id/iban-verification-history**
Fetch verification audit trail for a supplier.

```
Response:
{
  "supplier_id": "supp-001",
  "current_iban": "NL75ABNA0588564432",
  "history": [
    {
      "iban": "NL75ABNA0588564432",
      "verified_date": "2026-05-15T10:30:00Z",
      "verified_by": "user-controller-001",
      "status": "verified"
    }
  ]
}
```

#### Payment Batching & Netting

**POST /api/payments/create-batch**
Create a batch from a list of pending payments.

```
Request:
{
  "administration_id": "adm1",
  "payment_ids": ["pay-001", "pay-002", "pay-005"],
  "notes": "Weekly supplier payments"
}

Response:
{
  "batch": { PaymentBatch },
  "total_amount": { "amount": 7300.00, "currency": "EUR" }
}
```

**GET /api/payments/netting-analysis**
Analyze offsetting opportunities between payables and receivables.

```
Query params:
  - administration_id (required)
  - supplier_id (optional)

Response:
{
  "netting_opportunities": [
    {
      "netting_group_id": "netting-group-042",
      "supplier_id": "supp-001",
      "payable_amount": 5000.00,
      "receivable_amount": 3200.00,
      "net_payment_required": 1800.00,
      "potential_savings": 85.00,
      "currency": "EUR"
    }
  ]
}
```

**POST /api/payments/execute-netting**
Execute identified netting transactions.

```
Request:
{
  "netting_group_ids": ["netting-group-042"]
}

Response:
{
  "netting_transactions": [
    {
      "netting_group_id": "netting-group-042",
      "payable_payment_id": "pay-001",
      "receivable_payment_id": "ar-123",
      "offset_amount": 3200.00,
      "net_payment": 1800.00,
      "status": "settled"
    }
  ]
}
```

#### Separation of Duties

**GET /api/compliance/separation-of-duties-violations**
List transactions violating separation-of-duties rules.

```
Query params:
  - administration_id (required)
  - date_from (optional)
  - date_to (optional)
  - _page: int (default 1)
  - _limit: int (default 50)

Response:
{
  "violations": [
    {
      "violation_id": "sodv-001",
      "payment_id": "pay-xyz",
      "rule_violated": "requester_equals_approver",
      "user_id": "user-001",
      "date": "2026-05-20",
      "amount": 5000.00,
      "severity": "critical"
    }
  ],
  "total": 1,
  "page": 1,
  "pages": 1
}
```

**POST /api/compliance/set-separation-of-duties-rules**
Configure role-separation rules (admin only).

```
Request:
{
  "administration_id": "adm1",
  "rules": [
    { "rule": "requester_cannot_be_approver", "enabled": true },
    { "rule": "approver_cannot_be_payer", "enabled": true },
    { "rule": "requester_cannot_be_payer", "enabled": true }
  ]
}

Response:
{
  "rules_updated": 3,
  "administration_id": "adm1"
}
```

#### Payment Providers & Cash Pooling

**GET /api/payment-providers**
List payment providers for the administration.

```
Query params:
  - administration_id (required)

Response:
{
  "providers": [ { PaymentProvider }, ... ],
  "total": 3
}
```

**POST /api/cash-pooling/configure**
Set up cash pooling rules (admin only).

```
Request:
{
  "administration_id": "adm1",
  "primary_account_id": "b1a1c1d1-1111-1111-1111-111111111111",
  "pooled_account_ids": [
    "b1a1c1d1-2222-2222-2222-222222222222",
    "b1a1c1d1-3333-3333-3333-333333333333"
  ],
  "transfer_frequency": "daily",
  "transfer_time": "17:00",
  "minimum_balance_per_account": { "amount": 10000.00, "currency": "EUR" }
}

Response:
{
  "pooling_configuration": {
    "id": "pool-config-001",
    "status": "active"
  }
}
```

**POST /api/cash-pooling/execute**
Manually trigger cash pooling transfers.

```
Request:
{
  "administration_id": "adm1"
}

Response:
{
  "transfers_initiated": 2,
  "total_amount_transferred": 25000.00,
  "details": [
    {
      "from_account": "b1a1c1d1-2222-2222-2222-222222222222",
      "to_account": "b1a1c1d1-1111-1111-1111-111111111111",
      "amount": 25000.00,
      "reference": "Cash pooling transfer"
    }
  ]
}
```

#### Liquid Reserve Monitoring

**GET /api/treasury/liquid-reserves**
Check current vs. required reserves.

```
Query params:
  - administration_id (required)

Response:
{
  "administration_id": "adm1",
  "current_balance": { "amount": 170000.00, "currency": "EUR" },
  "minimum_required": { "amount": 50000.00, "currency": "EUR" },
  "status": "healthy",
  "days_of_reserve": 12.5
}
```

**POST /api/treasury/set-reserve-requirements**
Configure reserve thresholds (admin only).

```
Request:
{
  "administration_id": "adm1",
  "minimum_balance": { "amount": 50000.00, "currency": "EUR" },
  "alert_threshold_percentage": 80
}

Response:
{
  "configuration_id": "reserve-cfg-001",
  "status": "active"
}
```

---

## Part III: Frontend Components & UI Patterns

All components use `@conduction/nextcloud-vue` and follow NL Design System tokens.

### Pages

#### Dashboard (Enhanced)
- **Component:** `CnDashboardPage`  
- **Widgets:**
  1. Liquid reserves card (current vs. minimum, color-coded)  
  2. Cash flow chart (30/60/90-day horizon selector)  
  3. Reconciliation status widget (match rate, unmatched count)  
  4. Loan refinancing opportunities (count + savings estimate)  
  5. Separation-of-duties violations (recent, with severity badges)  
  6. Payment arrangements portfolio (due this week, overdue)

#### Bank Reconciliation Index
- **Component:** `CnIndexPage`  
- **Data source:** `BankStatementEntry`  
- **Features:**
  - Filter by account, reconciliation status, date range  
  - Column for "matched confidence" (visual indicator: green ≥0.9, yellow 0.5-0.9, red <0.5)  
  - Row action: "Match Manual" opens `CnAdvancedFormDialog` with ledger entry search  
  - Bulk reconciliation: select multiple unmatched entries, auto-suggest matches, approve all  

#### Cash Flow Forecast
- **Component:** Custom page with `CnChartWidget`  
- **Features:**
  - Horizon selector (30/60/90 days)  
  - Line chart showing opening balance, inflows, outflows, closing balance by day  
  - Assumptions panel (which invoices included, manual adjustments)  
  - Drill-down: click a date to see transactions for that day  
  - Export button: CSV or PDF forecast report  

#### Loan Refinancing
- **Component:** `CnIndexPage`  
- **Data source:** Computed field from `Loan` entity  
- **Features:**
  - Table: principal, current rate, market rate, annual savings, maturity date  
  - Sort by "refinancing score" (largest savings first)  
  - Row action: "View Refinancing Options" → detail modal with lender recommendations  
  - Filter: only eligible loans, date range for maturity  

#### Payment Batch Manager
- **Component:** Custom page with `CnDetailPage` for active batch  
- **Features:**
  - List of batches (draft/approved/executed/cancelled)  
  - Batch detail: payment list, total amount, approval status  
  - Actions: "Add Payment" (search dialog), "Remove", "Approve" (requires role), "Execute"  
  - Audit trail: show approvals, execution timestamp, rejection comments  

#### Supplier IBAN Verification
- **Component:** Side panel in `CnDetailPage` (supplier detail)  
- **Features:**
  - IBAN input field with real-time validation (format + checksum)  
  - "Verify" button → calls API, shows result with bank name  
  - Verification history timeline  
  - Flag for manual review if IBAN has changed  

#### Separation of Duties Audit Report
- **Component:** Custom page with `CnDataTable` + `CnFacetSidebar`  
- **Features:**
  - Table: violation ID, user, rule violated, date, amount, severity  
  - Facets: rule type, severity, date range  
  - Row detail: transaction audit trail, approvals chain  
  - Export: CSV with all violations and remediation notes  

#### Treasury Settings
- **Component:** `CnSettingsSection` pages  
- **Sections:**
  1. Bank Accounts: add/edit/remove, set reserve thresholds  
  2. Payment Providers: CRUD, test API connectivity  
  3. Separation of Duties: enable/disable rules, set strictness level  
  4. Cash Pooling: configure primary account, pooled accounts, frequency  
  5. Bank Statement Import: supported formats, auto-matching threshold  

---

## Part IV: Business Logic & Algorithms

### Bank Entry Matching Algorithm

**Input:** Bank statement entry, ledger entries (past 30 days)

**Steps:**
1. Exact match: amount + date + reference → high confidence (0.99)  
2. Fuzzy amount match: amount ±5% + date within 3 days → medium (0.7)  
3. Counterparty name fuzzy match + date ±5 days → low (0.5)  
4. Apply user-configured matching rules (e.g., "always match transfer code N005 if amount match")  

**Output:** Matched ledger entry ID + confidence score

### Cash Flow Projection

**Input:** Today's date, horizon (days), adjustments (optional)

**Steps:**
1. Fetch all outstanding invoices (AR): due date ≥ today, due date ≤ today + horizon  
2. Fetch all unpaid bills (AP): due date ≥ today, due date ≤ today + horizon  
3. Fetch scheduled payments: active, due date within horizon  
4. Aggregate by date: sum inflows (AR) and outflows (AP + scheduled) per day  
5. Calculate running balance: opening balance + inflows - outflows  
6. Apply adjustments (manual transactions)  

**Output:** Day-by-day projection table

### Loan Refinancing Score

**Input:** Loan (principal, current rate, maturity date)

**Steps:**
1. Fetch market rate for similar loan (lookup table: term + risk profile)  
2. Calculate rate delta: current rate - market rate  
3. Calculate annual interest savings: principal × rate delta  
4. Calculate refinancing score: savings / refinancing costs (est. 0.5-1% of principal)  
5. Threshold: score > 1.0 = eligible for refinancing  

**Output:** eligibility Boolean + savings estimate + refinancing score

### Netting Analysis

**Input:** Supplier (optional), horizon (days)

**Steps:**
1. Fetch all active payables to supplier: amount, due date  
2. Fetch all active receivables from supplier: amount, due date  
3. Within horizon, match receivables to payables (FIFO by due date)  
4. Calculate offset amount (min of payable, receivable)  
5. Calculate net payment: payable - receivable  
6. Savings: (offset amount × banking cost %) + (offset amount × interest rate % × days / 365)  

**Output:** Netting opportunities by supplier/date

### Separation of Duties Detection

**Input:** Payment (with approval chain, execution user)

**Steps:**
1. Fetch approval chain: requester, approver, payer  
2. Check rules:
   - `requester_cannot_be_approver`: payment.requester_id ≠ approval.approver_id  
   - `approver_cannot_be_payer`: approval.approver_id ≠ payment.executed_by_id  
   - `requester_cannot_be_payer`: payment.requester_id ≠ payment.executed_by_id  
3. Severity: critical (all 3 violated), high (2 violated), medium (1 violated)  

**Output:** Violations list with rule + severity

---

## Part V: Reuse Analysis

**Existing services leveraged (no duplication):**

| Service | Usage | Feature(s) |
|---------|-------|-----------|
| ObjectService | Save/retrieve bank accounts, payments, suppliers | All |
| ImportService | Import bank statements from files | Bank reconciliation |
| ExportService | Export forecasts, reports, audit trails | All reporting |
| IndexService | Full-text search on transactions, suppliers | Discovery |
| AuditTrailService | Track all changes, approval chains | Separation of duties, IBAN verification |
| FileService | Store bank statement files | Bank reconciliation |
| AuthorizationService | Role-based approval workflows | Payment batching, netting execution |
| NotificationService | Alert on reserve thresholds, violations | Monitoring |

**No new services proposed; all business logic fits within existing architectural patterns.**

---

**Design Version:** 1.0  
**Last Updated:** 2026-05-21
