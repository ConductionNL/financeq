# Design Document: Treasury & Cash Management — Shillinq

**Change:** treasury-cash-management-other-t2  
**Version:** 1.0  
**Date:** 2026-05-21  
**Status:** Design

## Table of Contents

1. [Entity Data Model](#entity-data-model)
2. [Seed Data](#seed-data)
3. [User Journeys & Workflows](#user-journeys--workflows)
4. [UI Architecture](#ui-architecture)
5. [Integration Points](#integration-points)
6. [Reuse Analysis](#reuse-analysis)

---

## Entity Data Model

All entities conform to **ADR-001** (OpenRegister data layer) and **ADR-011** (schema.org vocabulary). Relations use OpenRegister mechanism (register + schema + objectId), NOT foreign keys.

### 1. BankAccount

Entity for managing organization's bank and financial accounts.

```json
{
  "schema": "BankAccount",
  "type": "object",
  "title": "Bank Account",
  "description": "Financial institution account for payments, receipts, and reconciliation.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid",
      "description": "Unique identifier"
    },
    "accountHolder": {
      "type": "string",
      "description": "Legal account holder name (individual or organization)"
    },
    "accountNumber": {
      "type": "string",
      "pattern": "^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$",
      "description": "IBAN format (required for EU)"
    },
    "currency": {
      "type": "string",
      "enum": ["EUR", "USD", "GBP", "CHF", "SEK", "DKK", "NOK"],
      "default": "EUR",
      "description": "Account currency ISO 4217"
    },
    "bankName": {
      "type": "string",
      "description": "Financial institution name"
    },
    "bic": {
      "type": "string",
      "pattern": "^[A-Z0-9]{8}([A-Z0-9]{3})?$",
      "description": "Bank Identifier Code (SWIFT)"
    },
    "accountType": {
      "type": "string",
      "enum": ["checking", "savings", "business", "credit_card", "virtual"],
      "description": "Type of account"
    },
    "isActive": {
      "type": "boolean",
      "default": true,
      "description": "Whether account is actively used"
    },
    "syncStatus": {
      "type": "string",
      "enum": ["connected", "pending", "error", "disconnected"],
      "default": "pending",
      "description": "OpenRegister sync status"
    },
    "lastSyncDate": {
      "type": "string",
      "format": "date-time",
      "description": "Last successful data sync timestamp"
    },
    "reconciliationDate": {
      "type": "string",
      "format": "date",
      "description": "Latest reconciled statement date"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "tags": {
          "type": "array",
          "items": { "type": "string" },
          "description": "User-defined tags (operational, backup, etc.)"
        }
      }
    }
  },
  "required": ["accountNumber", "accountHolder", "currency", "accountType"]
}
```

### 2. Payment

Represents a single payment transaction (outbound or inbound).

```json
{
  "schema": "Payment",
  "type": "object",
  "title": "Payment",
  "description": "Individual payment transaction with method, status, and settlement details.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "amount": {
      "type": "number",
      "description": "Payment amount in transaction currency"
    },
    "currency": {
      "type": "string",
      "enum": ["EUR", "USD", "GBP", "CHF", "SEK", "DKK", "NOK"],
      "description": "Transaction currency ISO 4217"
    },
    "paymentMethod": {
      "type": "string",
      "enum": [
        "bank_transfer", "credit_card", "debit_card", "ach", "sepa_direct_debit",
        "gift_card", "virtual_card", "check", "cash", "digital_wallet", "other"
      ],
      "description": "Payment method type"
    },
    "status": {
      "type": "string",
      "enum": [
        "draft", "scheduled", "initiated", "pending", "authorized", "captured",
        "rejected", "failed", "settled", "reversed", "disputed"
      ],
      "description": "Payment status in workflow"
    },
    "payer": {
      "type": "object",
      "description": "Party initiating payment (OpenRegister relation: Party)",
      "properties": {
        "register": { "type": "string" },
        "schema": { "type": "string" },
        "objectId": { "type": "string" }
      }
    },
    "payee": {
      "type": "object",
      "description": "Party receiving payment (OpenRegister relation: Payee)",
      "properties": {
        "register": { "type": "string" },
        "schema": { "type": "string" },
        "objectId": { "type": "string" }
      }
    },
    "bankAccount": {
      "type": "object",
      "description": "Source/destination bank account",
      "properties": {
        "register": { "type": "string" },
        "schema": { "type": "string" },
        "objectId": { "type": "string" }
      }
    },
    "reference": {
      "type": "string",
      "maxLength": 140,
      "description": "Payment reference (remittance info for bank transfer)"
    },
    "requestedDate": {
      "type": "string",
      "format": "date",
      "description": "Payment initiation date (can be in future for scheduled)"
    },
    "executedDate": {
      "type": "string",
      "format": "date-time",
      "description": "Actual execution timestamp"
    },
    "settledDate": {
      "type": "string",
      "format": "date",
      "description": "Date funds cleared (final settlement)"
    },
    "feeAmount": {
      "type": "number",
      "description": "Payment processing fee"
    },
    "isPersonal": {
      "type": "boolean",
      "default": false,
      "description": "Personal expense marker (for business/personal segregation)"
    },
    "personalCategory": {
      "type": "string",
      "enum": ["travel", "meals", "office", "other"],
      "description": "Category if marked as personal expense"
    },
    "documentRef": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "register": { "type": "string" },
          "schema": { "type": "string" },
          "objectId": { "type": "string" }
        }
      },
      "description": "Related documents (Invoice, PurchaseOrder, Receipt)"
    },
    "notes": {
      "type": "string",
      "description": "Internal notes on payment"
    },
    "metadata": {
      "type": "object",
      "properties": {
        "externalId": {
          "type": "string",
          "description": "Payment ID from external gateway"
        },
        "retryCount": {
          "type": "integer",
          "default": 0
        }
      }
    }
  },
  "required": ["amount", "currency", "paymentMethod", "status", "payee"]
}
```

### 3. PaymentBatch

Groups multiple payments for collective processing and settlement.

```json
{
  "schema": "PaymentBatch",
  "type": "object",
  "title": "Payment Batch",
  "description": "Collection of payments processed together for settlement efficiency.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "batchNumber": {
      "type": "string",
      "description": "Human-readable batch identifier (e.g., 'BATCH-2026-05-001')"
    },
    "payments": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "register": { "type": "string" },
          "schema": { "type": "string" },
          "objectId": { "type": "string" }
        }
      },
      "description": "OpenRegister Payment objects in batch"
    },
    "totalAmount": {
      "type": "number",
      "description": "Sum of all payment amounts"
    },
    "currency": {
      "type": "string",
      "description": "Batch currency (all payments must be same currency)"
    },
    "paymentCount": {
      "type": "integer",
      "description": "Number of payments in batch"
    },
    "status": {
      "type": "string",
      "enum": ["draft", "approved", "submitted", "settled", "cancelled", "failed"],
      "description": "Batch processing status"
    },
    "paymentMethod": {
      "type": "string",
      "enum": ["bank_transfer", "credit_card", "ach", "sepa_debit", "other"],
      "description": "Collective payment method (all payments must match)"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time",
      "description": "Batch creation timestamp"
    },
    "submittedDate": {
      "type": "string",
      "format": "date-time",
      "description": "Submission to payment processor"
    },
    "settledDate": {
      "type": "string",
      "format": "date",
      "description": "Final settlement date"
    },
    "approver": {
      "type": "object",
      "description": "User who approved batch (OpenRegister relation: User)",
      "properties": {
        "register": { "type": "string" },
        "schema": { "type": "string" },
        "objectId": { "type": "string" }
      }
    },
    "totalFeeAmount": {
      "type": "number",
      "description": "Aggregate processing fees"
    },
    "notes": {
      "type": "string",
      "description": "Batch-level notes"
    }
  },
  "required": ["payments", "totalAmount", "status", "paymentMethod"]
}
```

### 4. CashFlowForecast

Projections of cash movements based on GL entries and scheduled payments.

```json
{
  "schema": "CashFlowForecast",
  "type": "object",
  "title": "Cash Flow Forecast",
  "description": "Projected cash position based on historical patterns and scheduled transactions.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "forecastDate": {
      "type": "string",
      "format": "date",
      "description": "As-of date for forecast generation"
    },
    "projectionPeriodDays": {
      "type": "integer",
      "default": 90,
      "description": "Number of days ahead to project"
    },
    "baselineCashPosition": {
      "type": "number",
      "description": "Current cash balance as of forecastDate"
    },
    "currency": {
      "type": "string",
      "enum": ["EUR", "USD", "GBP", "CHF"],
      "description": "Forecast currency"
    },
    "projections": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "date": {
            "type": "string",
            "format": "date",
            "description": "Projection date"
          },
          "inflows": {
            "type": "number",
            "description": "Expected inbound cash (revenues, collections)"
          },
          "outflows": {
            "type": "number",
            "description": "Expected outbound cash (expenses, payables)"
          },
          "netPosition": {
            "type": "number",
            "description": "Cumulative cash position"
          },
          "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 100,
            "description": "Confidence percentage (based on recurrence history)"
          }
        },
        "required": ["date", "inflows", "outflows", "netPosition"]
      },
      "description": "Daily projection entries"
    },
    "riskFlags": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "date": { "type": "string", "format": "date" },
          "severity": { "enum": ["low", "medium", "high"] },
          "message": { "type": "string" },
          "recommendation": { "type": "string" }
        }
      },
      "description": "Liquidity alerts (negative position, large outflows, etc.)"
    },
    "generationMethod": {
      "type": "string",
      "enum": ["rolling_average", "trend_analysis", "manual_input", "hybrid"],
      "description": "Forecast calculation methodology"
    },
    "lastUpdated": {
      "type": "string",
      "format": "date-time",
      "description": "Forecast recalculation timestamp"
    }
  },
  "required": ["forecastDate", "baselineCashPosition", "currency", "projections"]
}
```

### 5. ReconciliationRule

Auto-matching rules for bank statement reconciliation.

```json
{
  "schema": "ReconciliationRule",
  "type": "object",
  "title": "Reconciliation Rule",
  "description": "Rules for automatic matching of payments to bank statement transactions.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "ruleName": {
      "type": "string",
      "description": "Human-readable rule name"
    },
    "bankAccount": {
      "type": "object",
      "description": "Associated bank account",
      "properties": {
        "register": { "type": "string" },
        "schema": { "type": "string" },
        "objectId": { "type": "string" }
      }
    },
    "matchCriteria": {
      "type": "object",
      "properties": {
        "amountTolerance": {
          "type": "number",
          "description": "Max variance in amount (euros or percentage)"
        },
        "dateTolerance": {
          "type": "integer",
          "description": "Max days difference for date matching"
        },
        "referencePattern": {
          "type": "string",
          "description": "Regex pattern for reference field matching"
        },
        "partyMatch": {
          "type": "string",
          "enum": ["exact", "partial", "fuzzy"],
          "description": "Party name matching algorithm"
        }
      },
      "required": ["amountTolerance", "dateTolerance"]
    },
    "isActive": {
      "type": "boolean",
      "default": true
    },
    "priority": {
      "type": "integer",
      "description": "Rule priority (lower = applied first)"
    },
    "successRate": {
      "type": "number",
      "minimum": 0,
      "maximum": 100,
      "description": "Historical match success percentage"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": ["ruleName", "bankAccount", "matchCriteria"]
}
```

### 6. PaymentMethod

Configuration for supported payment methods and gateway integration.

```json
{
  "schema": "PaymentMethod",
  "type": "object",
  "title": "Payment Method",
  "description": "Supported payment method configuration and processor integration.",
  "properties": {
    "id": {
      "type": "string",
      "format": "uuid"
    },
    "methodType": {
      "type": "string",
      "enum": [
        "bank_transfer", "credit_card", "debit_card", "ach", "sepa_direct_debit",
        "gift_card", "virtual_card", "check", "cash", "digital_wallet", "qr_code",
        "klarna", "paypal", "stripe", "adyen", "mollie", "other"
      ],
      "description": "Payment method category"
    },
    "displayName": {
      "type": "string",
      "description": "User-facing method name"
    },
    "processorName": {
      "type": "string",
      "description": "Payment processor/gateway name"
    },
    "isEnabled": {
      "type": "boolean",
      "default": true
    },
    "processingFee": {
      "type": "object",
      "properties": {
        "feeType": {
          "enum": ["fixed", "percentage", "tiered"],
          "description": "Fee calculation type"
        },
        "feeAmount": {
          "type": "number",
          "description": "Fixed fee (euros) or percentage"
        },
        "minimumFee": {
          "type": "number"
        },
        "maximumFee": {
          "type": "number"
        }
      }
    },
    "supportedCurrencies": {
      "type": "array",
      "items": { "type": "string" },
      "description": "ISO 4217 currency codes supported"
    },
    "transactionLimits": {
      "type": "object",
      "properties": {
        "minAmount": { "type": "number" },
        "maxAmount": { "type": "number" },
        "maxPerDay": { "type": "number" }
      }
    },
    "requiresApproval": {
      "type": "boolean",
      "default": false,
      "description": "Whether transactions require approval before settlement"
    },
    "settlementTimedays": {
      "type": "integer",
      "description": "Days to settlement after execution"
    },
    "apiConfiguration": {
      "type": "object",
      "description": "Processor API credentials (stored securely via IAppConfig)",
      "properties": {
        "apiKey": { "type": "string" },
        "webhookSecret": { "type": "string" },
        "apiEndpoint": { "type": "string" }
      }
    }
  },
  "required": ["methodType", "displayName", "processorName", "isEnabled"]
}
```

---

## Seed Data

### BankAccount Seeds

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "slug": "main-bank-account-amsterdam"
    },
    "accountHolder": "Contoso Netherlands B.V.",
    "accountNumber": "NL91ABNA0417164300",
    "currency": "EUR",
    "bankName": "ABN AMRO Bank",
    "bic": "ABNANL2A",
    "accountType": "business",
    "isActive": true,
    "syncStatus": "connected",
    "lastSyncDate": "2026-05-21T10:30:00Z",
    "reconciliationDate": "2026-05-20",
    "metadata": {
      "tags": ["operational", "primary"]
    }
  },
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "slug": "savings-account-amsterdam"
    },
    "accountHolder": "Contoso Netherlands B.V.",
    "accountNumber": "NL39RABO0300065264",
    "currency": "EUR",
    "bankName": "Rabobank",
    "bic": "RABONL2U",
    "accountType": "savings",
    "isActive": true,
    "syncStatus": "connected",
    "lastSyncDate": "2026-05-21T09:15:00Z",
    "reconciliationDate": "2026-05-20",
    "metadata": {
      "tags": ["reserves", "backup"]
    }
  },
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "slug": "usd-account-amsterdam"
    },
    "accountHolder": "Contoso Netherlands B.V.",
    "accountNumber": "US0000123456789012",
    "currency": "USD",
    "bankName": "ING",
    "bic": "INGSNL2A",
    "accountType": "business",
    "isActive": true,
    "syncStatus": "pending",
    "lastSyncDate": "2026-05-21T08:00:00Z",
    "reconciliationDate": "2026-05-19",
    "metadata": {
      "tags": ["multi-currency", "active"]
    }
  }
]
```

### Payment Seeds

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "Payment",
      "slug": "payment-invoice-2026-05-001"
    },
    "amount": 2500.00,
    "currency": "EUR",
    "paymentMethod": "bank_transfer",
    "status": "settled",
    "payer": {
      "register": "shillinq_core",
      "schema": "Organization",
      "objectId": "org-contoso-nl"
    },
    "payee": {
      "register": "shillinq_core",
      "schema": "Supplier",
      "objectId": "supplier-acme-logistics"
    },
    "bankAccount": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "objectId": "main-bank-account-amsterdam"
    },
    "reference": "INV-2026-001",
    "requestedDate": "2026-05-15",
    "executedDate": "2026-05-15T09:30:00Z",
    "settledDate": "2026-05-17",
    "feeAmount": 0.00,
    "isPersonal": false,
    "documentRef": [
      {
        "register": "shillinq_core",
        "schema": "Invoice",
        "objectId": "inv-2026-001"
      }
    ],
    "notes": "Logistics for Q2 shipment"
  },
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "Payment",
      "slug": "payment-petty-cash-2026-05-002"
    },
    "amount": 87.50,
    "currency": "EUR",
    "paymentMethod": "cash",
    "status": "settled",
    "payer": {
      "register": "shillinq_core",
      "schema": "Organization",
      "objectId": "org-contoso-nl"
    },
    "payee": {
      "register": "shillinq_core",
      "schema": "Supplier",
      "objectId": "supplier-local-office"
    },
    "reference": "PETTY-2026-05-002",
    "requestedDate": "2026-05-18",
    "executedDate": "2026-05-18T14:00:00Z",
    "settledDate": "2026-05-18",
    "feeAmount": 0.00,
    "isPersonal": true,
    "personalCategory": "office",
    "notes": "Office supplies restock"
  },
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "Payment",
      "slug": "payment-salary-batch-2026-05"
    },
    "amount": 45000.00,
    "currency": "EUR",
    "paymentMethod": "sepa_direct_debit",
    "status": "scheduled",
    "payer": {
      "register": "shillinq_core",
      "schema": "Organization",
      "objectId": "org-contoso-nl"
    },
    "payee": {
      "register": "shillinq_core",
      "schema": "User",
      "objectId": "employee-batch-may-2026"
    },
    "bankAccount": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "objectId": "main-bank-account-amsterdam"
    },
    "reference": "PAYROLL-2026-05",
    "requestedDate": "2026-05-28",
    "feeAmount": 12.50,
    "isPersonal": false,
    "notes": "May 2026 payroll"
  }
]
```

### PaymentBatch Seed

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "PaymentBatch",
      "slug": "batch-suppliers-2026-05-weekly"
    },
    "batchNumber": "BATCH-2026-05-W2",
    "payments": [
      {
        "register": "shillinq_treasury",
        "schema": "Payment",
        "objectId": "payment-invoice-2026-05-001"
      }
    ],
    "totalAmount": 2500.00,
    "currency": "EUR",
    "paymentCount": 1,
    "status": "settled",
    "paymentMethod": "bank_transfer",
    "createdDate": "2026-05-14T08:00:00Z",
    "submittedDate": "2026-05-15T10:00:00Z",
    "settledDate": "2026-05-17",
    "approver": {
      "register": "shillinq_core",
      "schema": "User",
      "objectId": "user-cfo-john"
    },
    "totalFeeAmount": 0.00,
    "notes": "Weekly supplier payments - Q2"
  }
]
```

### CashFlowForecast Seed

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "CashFlowForecast",
      "slug": "forecast-2026-05-q2"
    },
    "forecastDate": "2026-05-21",
    "projectionPeriodDays": 90,
    "baselineCashPosition": 125000.00,
    "currency": "EUR",
    "projections": [
      {
        "date": "2026-05-22",
        "inflows": 15000.00,
        "outflows": 8500.00,
        "netPosition": 131500.00,
        "confidence": 92
      },
      {
        "date": "2026-05-23",
        "inflows": 5000.00,
        "outflows": 25000.00,
        "netPosition": 111500.00,
        "confidence": 88
      },
      {
        "date": "2026-05-28",
        "inflows": 8000.00,
        "outflows": 45000.00,
        "netPosition": 74500.00,
        "confidence": 85
      }
    ],
    "riskFlags": [
      {
        "date": "2026-05-28",
        "severity": "medium",
        "message": "Large payroll outflow anticipated",
        "recommendation": "Ensure sufficient reserves or defer non-critical expenses"
      }
    ],
    "generationMethod": "rolling_average",
    "lastUpdated": "2026-05-21T12:00:00Z"
  }
]
```

### ReconciliationRule Seed

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "ReconciliationRule",
      "slug": "rule-supplier-invoices"
    },
    "ruleName": "Supplier Invoice Auto-Match",
    "bankAccount": {
      "register": "shillinq_treasury",
      "schema": "BankAccount",
      "objectId": "main-bank-account-amsterdam"
    },
    "matchCriteria": {
      "amountTolerance": 1.0,
      "dateTolerance": 3,
      "referencePattern": "INV-\\d{4}-\\d{3}",
      "partyMatch": "fuzzy"
    },
    "isActive": true,
    "priority": 1,
    "successRate": 96.5,
    "createdDate": "2026-04-01T08:00:00Z"
  }
]
```

### PaymentMethod Seeds

```json
[
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "PaymentMethod",
      "slug": "method-bank-transfer-sepa"
    },
    "methodType": "bank_transfer",
    "displayName": "SEPA Bank Transfer",
    "processorName": "OpenRegister-SEPA",
    "isEnabled": true,
    "processingFee": {
      "feeType": "fixed",
      "feeAmount": 0.00
    },
    "supportedCurrencies": ["EUR"],
    "transactionLimits": {
      "minAmount": 0.01,
      "maxAmount": 999999.99,
      "maxPerDay": 2000000.00
    },
    "requiresApproval": false,
    "settlementTimedays": 2
  },
  {
    "@self": {
      "register": "shillinq_treasury",
      "schema": "PaymentMethod",
      "slug": "method-credit-card-visa"
    },
    "methodType": "credit_card",
    "displayName": "Visa Credit Card",
    "processorName": "Stripe",
    "isEnabled": true,
    "processingFee": {
      "feeType": "percentage",
      "feeAmount": 1.5
    },
    "supportedCurrencies": ["EUR", "USD", "GBP"],
    "transactionLimits": {
      "minAmount": 0.50,
      "maxAmount": 50000.00
    },
    "requiresApproval": true,
    "settlementTimedays": 1
  }
]
```

---

## User Journeys & Workflows

### Journey 1: Bank Account Reconciliation (CFO)

**Actor:** CFO / Treasury Manager  
**Goal:** Reconcile bank statement with GL entries  
**Trigger:** Weekly statement arrives from bank

**Steps:**
1. Open Shillinq → Treasury → Accounts → "Reconciliation Hub"
2. Select bank account (NL91ABNA0417164300)
3. System auto-fetches latest bank statement via OpenRegister
4. System applies ReconciliationRules, matches 95% of transactions
5. CFO reviews unmatched items (5 transactions)
6. Manual match 3 transactions; 2 flagged as discrepancies
7. Approve reconciliation → GL reconciliation entries posted
8. Export reconciliation report (audit trail, timestamps)

**Acceptance Criteria:**
- Reconciliation completes in <5min for 50+ transactions
- Unmatched transactions clearly highlighted with suggested matches
- Audit log shows who reconciled, when, which rules applied

---

### Journey 2: Batch Payment Processing (AP Clerk)

**Actor:** Accounts Payable Clerk  
**Goal:** Pay 20 supplier invoices in single batch  
**Trigger:** Weekly payment run

**Steps:**
1. Open Shillinq → Payables → Payment Batches → "+ New Batch"
2. System populates due invoices (drag-drop or multi-select)
3. Review: total amount (€47,500), payment method (SEPA), fees
4. Add notes ("Weekly run - Q2 suppliers")
5. Submit for approval (route to CFO)
6. CFO reviews, approves
7. System submits to processor → status: "submitted"
8. Next day: statement sync updates status to "settled"
9. GL entries auto-post (Debit AP Liability, Credit Cash)

**Acceptance Criteria:**
- Batch creation from GL due items in <2min
- Approval workflow enforces roles (AP clerk cannot approve)
- Fee calculation accurate (per PaymentMethod config)
- GL reconciliation automatic post-settlement

---

### Journey 3: Cash Flow Forecast Review (CFO)

**Actor:** CFO  
**Goal:** Monitor projected liquidity for Q2  
**Trigger:** Weekly dashboard view

**Steps:**
1. Dashboard loads Cash Flow Forecast widget
2. Shows: baseline €125k, 90-day projection, risk flags
3. CFO clicks detail → line chart (inflows/outflows by day)
4. Identifies May 28 payroll spike (€45k outflow)
5. System suggests mitigation: defer non-critical expense, or tap savings account
6. CFO exports forecast as PDF → board presentation

**Acceptance Criteria:**
- Forecast updates daily without manual intervention
- Confidence percentages show accuracy (based on recurrence)
- Risk flags (negative position) calculated and highlighted
- Export includes 5-slide PDF with charts, narrative, recommendations

---

### Journey 4: Personal Expense Tagging (SMB Owner)

**Actor:** Business Owner using shared bank account  
**Goal:** Separate personal meal expenses from business spending  
**Trigger:** Monthly review before tax accountant visit

**Steps:**
1. Bank statement sync pulls €87.50 "Local Café" transaction
2. Owner views transaction → marks as "Personal" → "Meals"
3. System: removes from GL (Cash Sales), logs in Personal Expense Report
4. At tax time: Personal Expense Report exported (meals: €450/month)
5. Accountant deducts personal amount from business income

**Acceptance Criteria:**
- Personal expense toggle fast (<1 click)
- GL automatically adjusted (not double-counted as business)
- Monthly report groups expenses by category
- Export format suitable for tax software (CSV with categories)

---

## UI Architecture

### Page 1: Treasury Dashboard

**Component:** `CnDashboardPage`  
**Widgets:**
- **KPI Block:** Cash Position (€125k), Forecast Variance (±3%), Payment Velocity (95% settled <2 days)
- **Chart:** 90-day cash flow projection (area chart with confidence bands)
- **Alert Panel:** Risk flags (negative position, large payables, forecast alerts)
- **Quick Actions:** "New Payment", "Reconcile", "View Forecast"

### Page 2: Bank Accounts List

**Component:** `CnIndexPage` + `CnDataTable`  
**Columns:** Account Holder, IBAN, Currency, Balance, Sync Status, Last Sync, Reconciliation Date  
**Actions:** View, Edit, Sync, Reconcile, Download Statement  
**Filters:** Sync Status, Currency, Account Type, Active/Inactive

### Page 3: Bank Account Detail

**Component:** `CnDetailPage` + `CnDetailGrid` + `CnObjectSidebar`  
**Sections:**
- Account Info (IBAN, BIC, Bank Name, Holder)
- Sync Status & Configuration
- Recent Transactions (table of last 10 payments)
- Files Tab (uploaded statements)
- Audit Trail Tab (sync history, reconciliation events)

### Page 4: Payment Form (Create/Edit)

**Component:** `CnFormDialog` (schema-driven)  
**Fields:**
- Amount (number, currency picker)
- Payee (autocomplete from Suppliers/Employees)
- Payment Method (dropdown)
- Bank Account (dropdown)
- Requested Date (date picker, future dates for scheduled)
- Reference (text, max 140 chars)
- Is Personal? (toggle) → Personal Category (conditional dropdown)
- Notes (textarea)

**Validation:**
- Amount > 0
- Payee required
- If scheduled: date >= today
- Personal category required if isPersonal=true

### Page 5: Reconciliation Hub

**Component:** `CnDetailPage` with custom reconciliation widget  
**Layout:**
- Bank account selector (dropdown)
- Bank statement date range picker
- Two-column table:
  - Left: Bank transactions (from statement)
  - Right: GL cash entries
  - Match indicator (✓ matched, ⚠ unmatched, ✗ discrepancy)
- Match action buttons: "Link", "Unlink", "Create Missing GL Entry"
- Approve button (bottom) → posts reconciliation entries

---

## Integration Points

### OpenRegister Integration

**Data Flow:**
1. App reads BankAccount schema from OpenRegister (`GET /register/shillinq_treasury/schemas/BankAccount`)
2. App creates Payment objects via `ObjectService.saveObject()`
3. Bank sync service retrieves payment data and merges with GL entries
4. Relations (payer → Organization, payee → Supplier) via OpenRegister schema

**Settings:**
- Register name: `shillinq_treasury`
- Schemas: BankAccount, Payment, PaymentBatch, CashFlowForecast, ReconciliationRule, PaymentMethod

### External Payment Processor APIs

**Stripe API** (credit card processing):
- `POST /v1/payment_intents` — initiate payment
- Webhook: payment.succeeded → update Payment.status

**Mollie API** (iDEAL, Bancontact, SEPA):
- `POST /v2/payments` — create payment
- Poll `/v2/payments/{id}` for status updates

**OpenRegister Sync** (bank statement feeds):
- Scheduled job queries OpenRegister banks endpoint
- Parses transactions, creates unmatched Payment records
- Triggers ReconciliationRule engine

### GL Integration

**Post-Settlement GL Entry:** When Payment.status = "settled":
- Debit: Cash Account (GL 1000)
- Credit: Payable/Receivable Account (GL 2100 or 4100)
- Reference: Payment.reference
- Date: Payment.settledDate

---

## Reuse Analysis

**OpenRegister Services Used:**
- `ObjectService` — Payment, PaymentBatch CRUD
- `RegisterService` — fetch bank data
- `SchemaService` — validate payment fields against schema
- `AuditTrailService` — track payment lifecycle (draft → scheduled → settled)
- `NotificationService` — alert on large payments, reconciliation exceptions
- `FileService` — upload bank statements, export reports
- `CalendarEventService` — schedule payment reminders

**@conduction/nextcloud-vue Components Used:**
- `CnIndexPage` — bank account list, payment list
- `CnDetailPage` — account detail, payment detail
- `CnDashboardPage` — cash flow forecast, KPIs
- `CnDataTable` — payment history, transaction list
- `CnFormDialog` — payment entry, reconciliation rules
- `CnObjectSidebar` — files, audit trail, notes
- `CnChartWidget` — cash flow projection (area chart)

**NO Duplication Found:** Treasury Cash Management features align with OpenRegister Payment lifecycle, bank sync, and GL posting. No overlap with core accounting module.

---

**Next Steps:**
1. Frontend team: Vue component stubs for pages 1–5
2. Backend team: Service layer (PaymentService, ReconciliationService, ForecastService)
3. Integration team: Stripe/Mollie connector services
4. QA: Write GIVEN/WHEN/THEN test scenarios (60+ spec scenarios)
