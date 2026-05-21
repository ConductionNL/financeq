# Treasury & Cash Management — Design

**Change:** treasury-cash-management  
**Phase:** design  
**Created:** 2026-05-21

---

## Data Model

### 1. CashAccount (`schema:BankAccount`)

_Primary entity: tracks bank accounts, petty cash, and cash equivalents for liquidity management and multi-account consolidation_

#### OpenRegister Schema

```json
{
  "name": "CashAccount",
  "description": "Bank accounts, petty cash, and cash equivalents for liquidity tracking",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique account identifier"
    },
    "accountName": {
      "type": "string",
      "description": "Account display name (e.g., 'Main Operating Account')"
    },
    "accountType": {
      "type": "string",
      "enum": ["BankAccount", "PettyCash", "CashEquivalent"],
      "description": "Account type: bank, petty cash, or equivalent"
    },
    "accountCode": {
      "type": "string",
      "description": "Internal GL account code for reconciliation"
    },
    "bankName": {
      "type": "string",
      "description": "Bank name (e.g., 'ING Bank')"
    },
    "accountNumber": {
      "type": "string",
      "description": "Bank account number or IBAN (masked for display)"
    },
    "currency": {
      "type": "string",
      "description": "Base currency code (ISO 4217, e.g., 'EUR')"
    },
    "currentBalance": {
      "type": "number",
      "description": "Current balance in base currency"
    },
    "availableBalance": {
      "type": "number",
      "description": "Available balance (excluding holds/pending)"
    },
    "lastBalanceUpdate": {
      "type": "string",
      "format": "date-time",
      "description": "Last balance import timestamp"
    },
    "riskLevel": {
      "type": "string",
      "enum": ["Low", "Medium", "High"],
      "description": "Counterparty credit risk level"
    },
    "isActive": {
      "type": "boolean",
      "description": "Account is actively used"
    },
    "isPrimaryAccount": {
      "type": "boolean",
      "description": "Primary account for cash concentration"
    }
  },
  "required": ["accountName", "accountType", "accountCode", "currency", "currentBalance"],
  "relations": [
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    },
    {
      "property": "currencyBalances",
      "type": "one-to-many",
      "target": "CurrencyBalance",
      "description": "Multi-currency balances"
    },
    {
      "property": "fxExposures",
      "type": "one-to-many",
      "target": "FXExposure",
      "description": "FX exposures for this account"
    },
    {
      "property": "forecasts",
      "type": "one-to-many",
      "target": "LiquidityForecast",
      "description": "Liquidity forecasts"
    },
    {
      "property": "scheduledPayments",
      "type": "one-to-many",
      "target": "ScheduledPayment",
      "description": "Scheduled payments from this account"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "cash-accounts",
    "schema": "CashAccount",
    "slug": "ing-main-eur",
    "data": {
      "accountName": "ING Main Operating Account",
      "accountType": "BankAccount",
      "accountCode": "1200",
      "bankName": "ING Bank",
      "accountNumber": "NL91 ABNA 0417 1643 00",
      "currency": "EUR",
      "currentBalance": 450000.00,
      "availableBalance": 445000.00,
      "lastBalanceUpdate": "2026-05-21T10:15:00Z",
      "riskLevel": "Low",
      "isActive": true,
      "isPrimaryAccount": true
    }
  },
  {
    "register": "cash-accounts",
    "schema": "CashAccount",
    "slug": "rabobank-usd",
    "data": {
      "accountName": "Rabobank USD Investment",
      "accountType": "BankAccount",
      "accountCode": "1201",
      "bankName": "Rabobank",
      "accountNumber": "NL12 RABO 0123 4567 89",
      "currency": "USD",
      "currentBalance": 200000.00,
      "availableBalance": 200000.00,
      "lastBalanceUpdate": "2026-05-21T09:45:00Z",
      "riskLevel": "Low",
      "isActive": true,
      "isPrimaryAccount": false
    }
  },
  {
    "register": "cash-accounts",
    "schema": "CashAccount",
    "slug": "petty-cash-main",
    "data": {
      "accountName": "Petty Cash Box",
      "accountType": "PettyCash",
      "accountCode": "1100",
      "bankName": "N/A",
      "accountNumber": "PETTY-001",
      "currency": "EUR",
      "currentBalance": 5000.00,
      "availableBalance": 5000.00,
      "lastBalanceUpdate": "2026-05-21T17:00:00Z",
      "riskLevel": "Medium",
      "isActive": true,
      "isPrimaryAccount": false
    }
  }
]
```

---

### 2. CurrencyBalance (`schema:Thing`)

_Multi-currency balance tracking per account for foreign currency management and exposure monitoring_

#### OpenRegister Schema

```json
{
  "name": "CurrencyBalance",
  "description": "Multi-currency balance record for FX tracking",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique balance record identifier"
    },
    "balanceId": {
      "type": "string",
      "description": "Business identifier for the balance"
    },
    "currency": {
      "type": "string",
      "description": "Currency code (ISO 4217, e.g., 'GBP', 'USD', 'JPY')"
    },
    "balance": {
      "type": "number",
      "description": "Current balance in foreign currency"
    },
    "previousBalance": {
      "type": "number",
      "description": "Previous balance for variance tracking"
    },
    "balanceDate": {
      "type": "string",
      "format": "date",
      "description": "Date of balance snapshot"
    },
    "lastUpdated": {
      "type": "string",
      "format": "date-time",
      "description": "Last update timestamp"
    }
  },
  "required": ["currency", "balance", "lastUpdated"],
  "relations": [
    {
      "property": "cashAccount",
      "type": "many-to-one",
      "target": "CashAccount",
      "description": "Parent cash account"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "currency-balances",
    "schema": "CurrencyBalance",
    "slug": "usd-balance-20260521",
    "data": {
      "balanceId": "USD-2026-05",
      "currency": "USD",
      "balance": 200000.00,
      "previousBalance": 195000.00,
      "balanceDate": "2026-05-21",
      "lastUpdated": "2026-05-21T10:00:00Z"
    }
  },
  {
    "register": "currency-balances",
    "schema": "CurrencyBalance",
    "slug": "gbp-balance-20260521",
    "data": {
      "balanceId": "GBP-2026-05",
      "currency": "GBP",
      "balance": 85000.00,
      "previousBalance": 82000.00,
      "balanceDate": "2026-05-21",
      "lastUpdated": "2026-05-21T10:00:00Z"
    }
  },
  {
    "register": "currency-balances",
    "schema": "CurrencyBalance",
    "slug": "jpy-balance-20260521",
    "data": {
      "balanceId": "JPY-2026-05",
      "currency": "JPY",
      "balance": 25000000.00,
      "previousBalance": 24500000.00,
      "balanceDate": "2026-05-21",
      "lastUpdated": "2026-05-21T10:00:00Z"
    }
  }
]
```

---

### 3. FXExposure (`schema:MonetaryAmount`)

_Track foreign exchange risk across currencies with current rates, valuations, and unrealized gains/losses_

#### OpenRegister Schema

```json
{
  "name": "FXExposure",
  "description": "FX exposure tracking with risk valuation",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique exposure identifier"
    },
    "baseCurrency": {
      "type": "string",
      "description": "Base reporting currency (typically EUR)"
    },
    "foreignCurrency": {
      "type": "string",
      "description": "Foreign currency code (ISO 4217)"
    },
    "exposureAmount": {
      "type": "number",
      "description": "Amount held in foreign currency"
    },
    "currentExchangeRate": {
      "type": "number",
      "description": "Current exchange rate (foreign/base)"
    },
    "rateSourceDate": {
      "type": "string",
      "format": "date",
      "description": "Date of exchange rate"
    },
    "valuationDate": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 rate snapshot datetime"
    },
    "baseCurrencyValue": {
      "type": "number",
      "description": "Exposure valued in base currency"
    },
    "unrealizedGainLoss": {
      "type": "number",
      "description": "Unrealized P&L in base currency"
    },
    "riskLevel": {
      "type": "string",
      "enum": ["Low", "Medium", "High"],
      "description": "Currency volatility risk level"
    }
  },
  "required": ["baseCurrency", "foreignCurrency", "exposureAmount", "currentExchangeRate"],
  "relations": [
    {
      "property": "cashAccount",
      "type": "many-to-one",
      "target": "CashAccount",
      "description": "Related cash account"
    },
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "fx-exposures",
    "schema": "FXExposure",
    "slug": "usd-eur-exposure",
    "data": {
      "baseCurrency": "EUR",
      "foreignCurrency": "USD",
      "exposureAmount": 200000.00,
      "currentExchangeRate": 1.0850,
      "rateSourceDate": "2026-05-21",
      "valuationDate": "2026-05-21T16:00:00Z",
      "baseCurrencyValue": 217000.00,
      "unrealizedGainLoss": 2500.00,
      "riskLevel": "Medium"
    }
  },
  {
    "register": "fx-exposures",
    "schema": "FXExposure",
    "slug": "gbp-eur-exposure",
    "data": {
      "baseCurrency": "EUR",
      "foreignCurrency": "GBP",
      "exposureAmount": 85000.00,
      "currentExchangeRate": 0.8620,
      "rateSourceDate": "2026-05-21",
      "valuationDate": "2026-05-21T16:00:00Z",
      "baseCurrencyValue": 73270.00,
      "unrealizedGainLoss": -500.00,
      "riskLevel": "Low"
    }
  },
  {
    "register": "fx-exposures",
    "schema": "FXExposure",
    "slug": "jpy-eur-exposure",
    "data": {
      "baseCurrency": "EUR",
      "foreignCurrency": "JPY",
      "exposureAmount": 25000000.00,
      "currentExchangeRate": 0.0065,
      "rateSourceDate": "2026-05-21",
      "valuationDate": "2026-05-21T16:00:00Z",
      "baseCurrencyValue": 162500.00,
      "unrealizedGainLoss": 1200.00,
      "riskLevel": "High"
    }
  }
]
```

---

### 4. LiquidityForecast (`schema:Report`)

_Daily/weekly/monthly cash flow projections for liquidity planning, including inflow/outflow/net position_

#### OpenRegister Schema

```json
{
  "name": "LiquidityForecast",
  "description": "Cash flow projections and liquidity planning",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique forecast identifier"
    },
    "period": {
      "type": "string",
      "enum": ["Daily", "Weekly", "Monthly"],
      "description": "Forecast period granularity"
    },
    "forecastDate": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 forecast generation datetime"
    },
    "projectionStartDate": {
      "type": "string",
      "format": "date",
      "description": "Start date of projection window"
    },
    "projectionEndDate": {
      "type": "string",
      "format": "date",
      "description": "End date of projection window"
    },
    "projectionDays": {
      "type": "integer",
      "description": "Days ahead to forecast (typically 13 weeks)"
    },
    "projectedInflow": {
      "type": "number",
      "description": "Expected cash in over projection period"
    },
    "projectedOutflow": {
      "type": "number",
      "description": "Expected cash out over projection period"
    },
    "netProjection": {
      "type": "number",
      "description": "Net position (inflow - outflow)"
    },
    "openingBalance": {
      "type": "number",
      "description": "Starting cash balance"
    },
    "projectedClosingBalance": {
      "type": "number",
      "description": "Projected ending cash balance"
    },
    "currency": {
      "type": "string",
      "description": "Currency code (ISO 4217)"
    },
    "confidence": {
      "type": "string",
      "enum": ["Low", "Medium", "High"],
      "description": "Forecast confidence level"
    },
    "methodology": {
      "type": "string",
      "description": "Forecasting method (e.g., 'AI', 'historical', 'manual')"
    }
  },
  "required": ["period", "forecastDate", "projectionDays", "projectedInflow", "projectedOutflow", "netProjection", "currency"],
  "relations": [
    {
      "property": "cashAccount",
      "type": "many-to-one",
      "target": "CashAccount",
      "description": "Related cash account"
    },
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "liquidity-forecasts",
    "schema": "LiquidityForecast",
    "slug": "forecast-daily-20260521",
    "data": {
      "period": "Daily",
      "forecastDate": "2026-05-21T10:00:00Z",
      "projectionStartDate": "2026-05-21",
      "projectionEndDate": "2026-09-02",
      "projectionDays": 91,
      "projectedInflow": 2150000.00,
      "projectedOutflow": 1980000.00,
      "netProjection": 170000.00,
      "openingBalance": 450000.00,
      "projectedClosingBalance": 620000.00,
      "currency": "EUR",
      "confidence": "High",
      "methodology": "AI"
    }
  },
  {
    "register": "liquidity-forecasts",
    "schema": "LiquidityForecast",
    "slug": "forecast-weekly-20260518",
    "data": {
      "period": "Weekly",
      "forecastDate": "2026-05-18T08:00:00Z",
      "projectionStartDate": "2026-05-18",
      "projectionEndDate": "2026-09-13",
      "projectionDays": 119,
      "projectedInflow": 2300000.00,
      "projectedOutflow": 2150000.00,
      "netProjection": 150000.00,
      "openingBalance": 450000.00,
      "projectedClosingBalance": 600000.00,
      "currency": "EUR",
      "confidence": "Medium",
      "methodology": "AI"
    }
  },
  {
    "register": "liquidity-forecasts",
    "schema": "LiquidityForecast",
    "slug": "forecast-monthly-202605",
    "data": {
      "period": "Monthly",
      "forecastDate": "2026-05-01T08:00:00Z",
      "projectionStartDate": "2026-05-01",
      "projectionEndDate": "2026-10-31",
      "projectionDays": 184,
      "projectedInflow": 5200000.00,
      "projectedOutflow": 4800000.00,
      "netProjection": 400000.00,
      "openingBalance": 450000.00,
      "projectedClosingBalance": 850000.00,
      "currency": "EUR",
      "confidence": "Medium",
      "methodology": "historical"
    }
  }
]
```

---

### 5. PaymentBatch (`schema:Payment`)

_Batch grouping of multiple payments for mass processing, approval, and scheduled execution_

#### OpenRegister Schema

```json
{
  "name": "PaymentBatch",
  "description": "Batch of payments for grouped processing",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique batch identifier"
    },
    "batchNumber": {
      "type": "string",
      "description": "Business-friendly batch identifier (e.g., 'BATCH-2026-05-001')"
    },
    "description": {
      "type": "string",
      "description": "Batch description or purpose"
    },
    "totalAmount": {
      "type": "number",
      "description": "Sum of all payments in batch"
    },
    "totalPayments": {
      "type": "integer",
      "description": "Count of payment records in batch"
    },
    "currency": {
      "type": "string",
      "description": "Currency code (ISO 4217)"
    },
    "status": {
      "type": "string",
      "enum": ["draft", "pending", "processing", "completed", "failed", "cancelled"],
      "description": "Batch execution status"
    },
    "approvalStatus": {
      "type": "string",
      "enum": ["pending", "approved", "rejected"],
      "description": "Approval workflow status"
    },
    "approvedBy": {
      "type": "string",
      "description": "User who approved the batch"
    },
    "approvalDate": {
      "type": "string",
      "format": "date-time",
      "description": "Date/time of approval"
    },
    "scheduledDate": {
      "type": "string",
      "format": "date-time",
      "description": "Scheduled execution date for batch"
    },
    "executedDate": {
      "type": "string",
      "format": "date-time",
      "description": "Actual execution date"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time",
      "description": "Batch creation timestamp"
    },
    "exportFormat": {
      "type": "string",
      "enum": ["SEPA", "SWIFT", "CSV", "JSON"],
      "description": "Export file format"
    },
    "exportedFile": {
      "type": "string",
      "description": "Reference to exported file"
    }
  },
  "required": ["batchNumber", "totalAmount", "totalPayments", "status", "currency"],
  "relations": [
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    },
    {
      "property": "payments",
      "type": "one-to-many",
      "target": "schema:Payment",
      "description": "Individual payments in batch"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "payment-batches",
    "schema": "PaymentBatch",
    "slug": "batch-2026-05-001",
    "data": {
      "batchNumber": "BATCH-2026-05-001",
      "description": "Payroll May 2026",
      "totalAmount": 125000.00,
      "totalPayments": 48,
      "currency": "EUR",
      "status": "completed",
      "approvalStatus": "approved",
      "approvedBy": "john.doe@org.nl",
      "approvalDate": "2026-05-20T14:30:00Z",
      "scheduledDate": "2026-05-22T00:00:00Z",
      "executedDate": "2026-05-22T08:15:00Z",
      "createdDate": "2026-05-19T09:00:00Z",
      "exportFormat": "SEPA",
      "exportedFile": "payroll-2026-05.xml"
    }
  },
  {
    "register": "payment-batches",
    "schema": "PaymentBatch",
    "slug": "batch-2026-05-002",
    "data": {
      "batchNumber": "BATCH-2026-05-002",
      "description": "Supplier invoices - Week of 26 May",
      "totalAmount": 87500.00,
      "totalPayments": 15,
      "currency": "EUR",
      "status": "pending",
      "approvalStatus": "pending",
      "approvedBy": null,
      "approvalDate": null,
      "scheduledDate": "2026-05-28T00:00:00Z",
      "executedDate": null,
      "createdDate": "2026-05-21T10:00:00Z",
      "exportFormat": "SEPA",
      "exportedFile": null
    }
  },
  {
    "register": "payment-batches",
    "schema": "PaymentBatch",
    "slug": "batch-2026-05-003",
    "data": {
      "batchNumber": "BATCH-2026-05-003",
      "description": "Municipal transfers to gemeenschappelijke regelingen",
      "totalAmount": 250000.00,
      "totalPayments": 8,
      "currency": "EUR",
      "status": "draft",
      "approvalStatus": "pending",
      "approvedBy": null,
      "approvalDate": null,
      "scheduledDate": "2026-06-05T00:00:00Z",
      "executedDate": null,
      "createdDate": "2026-05-21T11:30:00Z",
      "exportFormat": "SEPA",
      "exportedFile": null
    }
  }
]
```

---

### 6. RequestForQuotation (`schema:Quotation`)

_Request for quotation supporting RFx management with templated events, multi-round negotiations, and digital lockbox_

#### OpenRegister Schema

```json
{
  "name": "RequestForQuotation",
  "description": "RFQ with digital lockbox and multi-round support",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique RFQ identifier"
    },
    "rfqNumber": {
      "type": "string",
      "description": "Business RFQ identifier (e.g., 'RFQ-2026-00145')"
    },
    "title": {
      "type": "string",
      "description": "RFQ title or procurement description"
    },
    "description": {
      "type": "string",
      "description": "Detailed RFQ scope and requirements"
    },
    "estimatedValue": {
      "type": "number",
      "description": "Estimated procurement value"
    },
    "currency": {
      "type": "string",
      "description": "Currency for procurement"
    },
    "deadline": {
      "type": "string",
      "format": "date-time",
      "description": "Submission deadline for responses"
    },
    "round": {
      "type": "integer",
      "description": "Negotiation round number (1=initial)"
    },
    "status": {
      "type": "string",
      "enum": ["draft", "published", "closed", "awarded", "cancelled"],
      "description": "RFQ lifecycle status"
    },
    "lockboxEnabled": {
      "type": "boolean",
      "description": "Enable digital lockbox to prevent bid viewing before deadline"
    },
    "lockboxOpensAt": {
      "type": "string",
      "format": "date-time",
      "description": "Datetime when bids become visible"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time",
      "description": "RFQ creation date"
    },
    "publishedDate": {
      "type": "string",
      "format": "date-time",
      "description": "Publication date"
    },
    "awardedSupplier": {
      "type": "string",
      "description": "Selected supplier after award"
    }
  },
  "required": ["rfqNumber", "title", "deadline", "status"],
  "relations": [
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    },
    {
      "property": "suppliers",
      "type": "many-to-many",
      "target": "schema:Supplier",
      "description": "Invited suppliers"
    },
    {
      "property": "offers",
      "type": "one-to-many",
      "target": "schema:Offer",
      "description": "Received offers/bids"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "requests-for-quotation",
    "schema": "RequestForQuotation",
    "slug": "rfq-2026-00145",
    "data": {
      "rfqNumber": "RFQ-2026-00145",
      "title": "Software License Procurement - 5-year agreement",
      "description": "Procurement of enterprise software licenses for financial management suite",
      "estimatedValue": 85000.00,
      "currency": "EUR",
      "deadline": "2026-06-15T17:00:00Z",
      "round": 1,
      "status": "published",
      "lockboxEnabled": true,
      "lockboxOpensAt": "2026-06-16T09:00:00Z",
      "createdDate": "2026-05-10T08:30:00Z",
      "publishedDate": "2026-05-15T10:00:00Z",
      "awardedSupplier": null
    }
  },
  {
    "register": "requests-for-quotation",
    "schema": "RequestForQuotation",
    "slug": "rfq-2026-00146",
    "data": {
      "rfqNumber": "RFQ-2026-00146",
      "title": "Office supplies and stationery - Annual contract",
      "description": "Supply of office materials for all municipal offices",
      "estimatedValue": 35000.00,
      "currency": "EUR",
      "deadline": "2026-06-10T17:00:00Z",
      "round": 1,
      "status": "closed",
      "lockboxEnabled": true,
      "lockboxOpensAt": "2026-06-11T09:00:00Z",
      "createdDate": "2026-05-01T08:00:00Z",
      "publishedDate": "2026-05-05T10:00:00Z",
      "awardedSupplier": "Staples Netherlands"
    }
  },
  {
    "register": "requests-for-quotation",
    "schema": "RequestForQuotation",
    "slug": "rfq-2026-00147",
    "data": {
      "rfqNumber": "RFQ-2026-00147",
      "title": "IT Infrastructure Services - Cloud hosting",
      "description": "Cloud hosting and infrastructure services for municipal applications",
      "estimatedValue": 120000.00,
      "currency": "EUR",
      "deadline": "2026-07-01T17:00:00Z",
      "round": 2,
      "status": "published",
      "lockboxEnabled": true,
      "lockboxOpensAt": "2026-07-02T09:00:00Z",
      "createdDate": "2026-04-15T08:00:00Z",
      "publishedDate": "2026-05-20T10:00:00Z",
      "awardedSupplier": null
    }
  }
]
```

---

### 7. ScheduledPayment (`schema:Payment`)

_Payment scheduled for future execution with support for recurring transactions_

#### OpenRegister Schema

```json
{
  "name": "ScheduledPayment",
  "description": "Future or recurring payment",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique payment identifier"
    },
    "paymentReference": {
      "type": "string",
      "description": "Unique payment reference or confirmation number"
    },
    "description": {
      "type": "string",
      "description": "Payment purpose or description"
    },
    "amount": {
      "type": "number",
      "description": "Payment amount"
    },
    "currency": {
      "type": "string",
      "description": "Currency code (ISO 4217)"
    },
    "payeeId": {
      "type": "string",
      "description": "Payee identifier"
    },
    "payeeName": {
      "type": "string",
      "description": "Payee name or description"
    },
    "payeeAccountNumber": {
      "type": "string",
      "description": "Payee IBAN or account number"
    },
    "scheduledDate": {
      "type": "string",
      "format": "date-time",
      "description": "Date scheduled for execution"
    },
    "frequency": {
      "type": "string",
      "enum": ["once", "daily", "weekly", "bi-weekly", "monthly", "quarterly", "semi-annual", "annual"],
      "description": "Recurrence frequency"
    },
    "recurringStartDate": {
      "type": "string",
      "format": "date",
      "description": "Start date for recurring payments"
    },
    "recurringEndDate": {
      "type": "string",
      "format": "date",
      "description": "End date for recurring payments"
    },
    "occurrenceCount": {
      "type": "integer",
      "description": "Number of occurrences for recurring payment"
    },
    "status": {
      "type": "string",
      "enum": ["pending", "approved", "executed", "failed", "cancelled"],
      "description": "Payment execution status"
    },
    "lastExecutionDate": {
      "type": "string",
      "format": "date-time",
      "description": "Date of last payment execution"
    },
    "nextExecutionDate": {
      "type": "string",
      "format": "date",
      "description": "Next scheduled execution date"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time",
      "description": "Payment creation date"
    }
  },
  "required": ["paymentReference", "amount", "currency", "payeeName", "scheduledDate", "status"],
  "relations": [
    {
      "property": "payee",
      "type": "many-to-one",
      "target": "schema:Payee",
      "description": "Payee organization"
    },
    {
      "property": "cashAccount",
      "type": "many-to-one",
      "target": "CashAccount",
      "description": "Source cash account"
    },
    {
      "property": "payments",
      "type": "one-to-many",
      "target": "schema:Payment",
      "description": "Individual payment records"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "scheduled-payments",
    "schema": "ScheduledPayment",
    "slug": "sp-rent-annual",
    "data": {
      "paymentReference": "SP-2026-RENT-001",
      "description": "Monthly office rent - Building A",
      "amount": 5000.00,
      "currency": "EUR",
      "payeeId": "realestate-corp-nl",
      "payeeName": "Real Estate Corp NL",
      "payeeAccountNumber": "NL91 ABNA 0212 3456 78",
      "scheduledDate": "2026-06-01T00:00:00Z",
      "frequency": "monthly",
      "recurringStartDate": "2026-06-01",
      "recurringEndDate": "2027-05-31",
      "occurrenceCount": 12,
      "status": "approved",
      "lastExecutionDate": null,
      "nextExecutionDate": "2026-06-01",
      "createdDate": "2026-05-01T10:00:00Z"
    }
  },
  {
    "register": "scheduled-payments",
    "schema": "ScheduledPayment",
    "slug": "sp-insurance-quarterly",
    "data": {
      "paymentReference": "SP-2026-INSURE-Q2",
      "description": "Quarterly liability insurance premium",
      "amount": 12500.00,
      "currency": "EUR",
      "payeeId": "insurance-provider-nl",
      "payeeName": "Dutch Insurance Company",
      "payeeAccountNumber": "NL12 RABO 0345 6789 01",
      "scheduledDate": "2026-06-15T00:00:00Z",
      "frequency": "quarterly",
      "recurringStartDate": "2026-06-15",
      "recurringEndDate": "2027-12-15",
      "occurrenceCount": 4,
      "status": "pending",
      "lastExecutionDate": null,
      "nextExecutionDate": "2026-06-15",
      "createdDate": "2026-05-20T14:00:00Z"
    }
  },
  {
    "register": "scheduled-payments",
    "schema": "ScheduledPayment",
    "slug": "sp-intercompany-transfer",
    "data": {
      "paymentReference": "SP-2026-INTER-TRANS",
      "description": "One-time intercompany liquidity transfer to subsidiary",
      "amount": 250000.00,
      "currency": "EUR",
      "payeeId": "subsidiary-bv",
      "payeeName": "Holding Company Subsidiary BV",
      "payeeAccountNumber": "NL88 ABNA 0987 6543 21",
      "scheduledDate": "2026-06-10T00:00:00Z",
      "frequency": "once",
      "recurringStartDate": null,
      "recurringEndDate": null,
      "occurrenceCount": 1,
      "status": "approved",
      "lastExecutionDate": null,
      "nextExecutionDate": "2026-06-10",
      "createdDate": "2026-05-18T09:30:00Z"
    }
  }
]
```

---

### 8. TreasuryTask (`schema:Event`)

_Unified AP/AR/spend task list for cash flow management with due dates and counterparty tracking_

#### OpenRegister Schema

```json
{
  "name": "TreasuryTask",
  "description": "Unified AP/AR/spend task for cash flow tracking",
  "type": "object",
  "properties": {
    "id": {
      "type": "string",
      "description": "Unique task identifier"
    },
    "taskId": {
      "type": "string",
      "description": "Business task identifier (e.g., 'TASK-AP-2026-001')"
    },
    "taskType": {
      "type": "string",
      "enum": ["AccountsPayable", "AccountsReceivable", "CapitalExpenditure", "Loan", "Investment"],
      "description": "Task category"
    },
    "title": {
      "type": "string",
      "description": "Task title or description"
    },
    "amount": {
      "type": "number",
      "description": "Transaction amount"
    },
    "currency": {
      "type": "string",
      "description": "Currency code (ISO 4217)"
    },
    "dueDate": {
      "type": "string",
      "format": "date",
      "description": "ISO 8601 date"
    },
    "counterpartyName": {
      "type": "string",
      "description": "Vendor, customer, or counterparty name"
    },
    "counterpartyId": {
      "type": "string",
      "description": "Counterparty identifier"
    },
    "description": {
      "type": "string",
      "description": "Task details and notes"
    },
    "status": {
      "type": "string",
      "enum": ["pending", "in-progress", "due-soon", "overdue", "completed", "cancelled"],
      "description": "Task status"
    },
    "priority": {
      "type": "string",
      "enum": ["low", "medium", "high", "critical"],
      "description": "Priority level"
    },
    "documentRef": {
      "type": "string",
      "description": "Reference to invoice, PO, or contract"
    },
    "createdDate": {
      "type": "string",
      "format": "date-time",
      "description": "Task creation date"
    },
    "completedDate": {
      "type": "string",
      "format": "date-time",
      "description": "Task completion date"
    }
  },
  "required": ["taskId", "taskType", "amount", "currency", "dueDate", "counterpartyName"],
  "relations": [
    {
      "property": "cashAccount",
      "type": "many-to-one",
      "target": "CashAccount",
      "description": "Related cash account"
    },
    {
      "property": "organization",
      "type": "many-to-one",
      "target": "schema:Organization",
      "description": "Owning organization"
    }
  ]
}
```

#### Seed Data (3 examples)

```json
[
  {
    "register": "treasury-tasks",
    "schema": "TreasuryTask",
    "slug": "task-ap-202605-001",
    "data": {
      "taskId": "TASK-AP-2026-05-001",
      "taskType": "AccountsPayable",
      "title": "Invoice INV-2026-00456 - Office supplies",
      "amount": 8500.00,
      "currency": "EUR",
      "dueDate": "2026-06-10",
      "counterpartyName": "Staples Netherlands",
      "counterpartyId": "staples-nl",
      "description": "Payment for office supplies and stationery delivered on 2026-05-20",
      "status": "pending",
      "priority": "medium",
      "documentRef": "INV-2026-00456",
      "createdDate": "2026-05-20T14:30:00Z",
      "completedDate": null
    }
  },
  {
    "register": "treasury-tasks",
    "schema": "TreasuryTask",
    "slug": "task-ar-202605-001",
    "data": {
      "taskId": "TASK-AR-2026-05-001",
      "taskType": "AccountsReceivable",
      "title": "Invoice INV-OUT-2026-00234 - Consulting services",
      "amount": 45000.00,
      "currency": "EUR",
      "dueDate": "2026-06-20",
      "counterpartyName": "City Council - Department of Finance",
      "counterpartyId": "city-council-finance",
      "description": "Consulting project completion invoice, due 30 days net",
      "status": "in-progress",
      "priority": "high",
      "documentRef": "INV-OUT-2026-00234",
      "createdDate": "2026-05-15T10:00:00Z",
      "completedDate": null
    }
  },
  {
    "register": "treasury-tasks",
    "schema": "TreasuryTask",
    "slug": "task-capex-202605-001",
    "data": {
      "taskId": "TASK-CAPEX-2026-05-001",
      "taskType": "CapitalExpenditure",
      "title": "Server equipment purchase - IT infrastructure upgrade",
      "amount": 125000.00,
      "currency": "EUR",
      "dueDate": "2026-07-15",
      "counterpartyName": "Dell Technologies",
      "counterpartyId": "dell-tech-nl",
      "description": "Planned CapEx for data center server replacement, budgeted for Q3 2026",
      "status": "due-soon",
      "priority": "high",
      "documentRef": "PO-2026-00789",
      "createdDate": "2026-04-01T08:00:00Z",
      "completedDate": null
    }
  }
]
```

---

## Workflows

### Payment Execution Workflow

```
Draft → Pending → Approved → Scheduled → Executed → Completed
         ↓
       Rejected ← Approval Denied
```

### Cash Forecasting Workflow

```
Import Bank Statements → Calculate Opening Balances → Retrieve Scheduled Payments → 
AI Model Prediction → Generate Forecast → Notify Treasury Team → Refresh Daily
```

### Compliance Monitoring Workflow

```
Daily Cash Position → Check Wet Fido Limits → Check schatkistbankieren Rules →
Flag Violations → Alert Treasurer → Recommendation for Transfer/Deposit
```

### Intercompany Transfer Workflow

```
Initiate Transfer → Record Counterparty → Create Offsetting Entries → 
Update Both Entities' Positions → Month-End Reconciliation Check
```

---

## Reuse Analysis

This spec leverages existing OpenRegister + Nextcloud capabilities:

- **ObjectService:** CRUD for all 8 entities via `saveObject()`, `deleteObject()`
- **ObjectStore + Pinia:** Entity state management, schema-driven forms
- **CnIndexPage + CnDetailPage:** List and detail views for all entities
- **ImportService/ExportService:** Bank statement import, SEPA/SWIFT export
- **AuditTrailService:** Full change tracking on payments and forecasts
- **NotificationService:** Treasury alerts (compliance violations, forecast updates)
- **WebhookService:** Bank integration events, payment status callbacks
- **SearchService:** Find payments, tasks, forecasts by amount/date/counterparty
- **TasksController:** Native task management for TreasuryTask

**No duplication:** Each new entity fills a gap (forecasts, FX tracking, scheduled payments, treasury tasks) not covered by existing Order/Invoice/Payment entities in OpenRegister.

---

## Dutch Government Compliance

- **BBV (Besluit Begroting en Verantwoording Decentrale Overheden):** Treasury reporting mandatory for municipal finance
- **Wet Fido (Wet Financiering Decentrale Overheden):** Strict borrowing and interest risk limits — kasgeldlimiet enforcement
- **schatkistbankieren:** Obligation to deposit municipal cash in Rijkshoofdboekhouding when above threshold
- **SiSa (Systeem Informatieverstrekking Subsidies aan Derden):** Subsidy transparency reporting
- **IV3 (Interne Verslaglegging Informatiestandaard):** Municipality reporting standard for CBS

---

## Implementation Notes

- Entities use OpenRegister `@self` envelope (register, schema, slug) for seeding
- Seed data uses Dutch organization names and realistic EUR amounts
- All timestamps in ISO 8601 format for international compatibility
- FX rates updated daily via external rate provider integration (future task)
- Forecasting uses Anthropic API with prompt caching for efficiency
- SEPA export validates IBAN checksums before file generation
- Compliance engines run nightly with alert dispatch at 06:00 CET
