# Specifications: Tax & Levy Management — Shillinq — Other T2

**Status:** In Specification  
**Date:** 2026-05-21  
**Change ID:** tax-levy-management-other-t2  

---

## Requirement: REQ-TAX-001 — VAT Auto-detection

**Feature:** VAT Auto-detection (demand: 6)

**Summary:** System automatically classifies invoices by VAT status (Standard, Reverse Charge, Exempt, Zero-Rated) based on transaction context (supplier/customer country, business type, transaction type).

### Scenario: REQ-TAX-001-A — Domestic Sale (Standard VAT)

```gherkin
GIVEN a TaxDetectionRule with:
  | Field            | Value            |
  | supplierCountry  | NL               |
  | customerCountry  | NL               |
  | transactionType  | Sale             |
  | customerType     | Business         |
  | outcome.vatStatus| Standard         |
  | outcome.rate     | 21%              |

AND an Invoice with:
  | Field           | Value            |
  | supplier.country| NL               |
  | customer.country| NL               |
  | type            | Sale             |
  | amount          | €1,000           |

WHEN the system determines VAT for this invoice

THEN the invoice should have:
  | Field       | Value                      |
  | vatStatus   | Standard                   |
  | rate        | 21%                        |
  | taxAmount   | €210 (net: €1000, gross: €1210) |
  | ruleApplied | nl-domestic-sale           |
```

### Scenario: REQ-TAX-001-B — EU B2B Reverse Charge

```gherkin
GIVEN a TaxDetectionRule with:
  | Field              | Value              |
  | supplierCountry    | [DE, BE, FR]       |
  | customerCountry    | NL                 |
  | transactionType    | Purchase           |
  | customerType       | Business           |
  | outcome.vatStatus  | ReverseCharge      |
  | outcome.rate       | 0%                 |

AND an Invoice with:
  | Field            | Value            |
  | supplier.country | DE               |
  | customer.country | NL               |
  | type             | Purchase         |
  | amount           | €2,000           |

WHEN the system determines VAT for this invoice

THEN the invoice should have:
  | Field       | Value                       |
  | vatStatus   | ReverseCharge               |
  | rate        | 0%                          |
  | taxAmount   | €0 (inbound VAT separate)   |
  | ruleApplied | eu-b2b-reverse-charge      |
```

### Scenario: REQ-TAX-001-C — No Rule Match (Fallback to Default)

```gherkin
GIVEN no matching TaxDetectionRule for an invoice

AND TaxConfiguration with defaultVATStatus = Standard, defaultRate = 21%

AND an Invoice with:
  | Field           | Value          |
  | supplier.name   | Unknown Corp   |
  | customer.country| NL             |
  | type            | Service        |
  | amount          | €500           |

WHEN the system determines VAT for this invoice

THEN the invoice should have:
  | Field       | Value                 |
  | vatStatus   | Standard              |
  | rate        | 21%                   |
  | ruleApplied | default               |
  | explanation | "No rule matched"     |
```

---

## Requirement: REQ-TAX-002 — Tax Exemptions

**Feature:** Tax Exemptions (KOR, B2B, Reverse Charge, etc.)

**Summary:** System applies exemption rules automatically or with user confirmation, checking validity date and conditions (amount threshold, supplier type, transaction type).

### Scenario: REQ-TAX-002-A — KOR Small Business Exemption

```gherkin
GIVEN a TaxExemption:
  | Field                 | Value              |
  | exemptionType         | KOR                |
  | jurisdiction          | NL                 |
  | validFrom             | 2024-01-01         |
  | conditions.maxAmount  | €50,000            |
  | appliesAutomatically  | true               |

AND an Organization with:
  | Field             | Value              |
  | jurisdiction      | NL                 |
  | totalAnnualSales  | €30,000            |

AND a Sales Invoice with:
  | Field    | Value   |
  | amount   | €5,000  |

WHEN the system determines VAT for this invoice

THEN the invoice should have:
  | Field          | Value                       |
  | vatStatus      | Exempt                      |
  | exemptionRule  | kor-small-business          |
  | explanation    | "KOR exemption applies"     |
  | taxAmount      | €0                          |

AND the organization's total sales should increase to €35,000
```

### Scenario: REQ-TAX-002-B — KOR Threshold Exceeded (Not Exempt)

```gherkin
GIVEN a TaxExemption with maxAmount = €50,000

AND an Organization with totalAnnualSales = €48,000

AND a Sales Invoice with amount = €5,000

WHEN the system checks if KOR exemption applies

THEN the system should:
  | Action                          |
  | Check: €48,000 + €5,000 = €53,000 > €50,000 threshold |
  | REJECT the exemption            |
  | Apply Standard VAT (21%)         |
  | Notify user of threshold exceeded |
```

### Scenario: REQ-TAX-002-C — Expired Exemption Rule

```gherkin
GIVEN a TaxExemption with validTo = 2025-12-31

AND current date = 2026-05-21

AND a Sales Invoice

WHEN the system checks exemption eligibility

THEN the exemption should be:
  | Status        |
  | Inactive      |
  | Not applied   |
```

---

## Requirement: REQ-TAX-003 — Configurable Tax Rates

**Feature:** Configurable Tax Rates with Compound and Inclusive/Exclusive Options

**Summary:** System supports multiple tax rates per jurisdiction with compound rate support (e.g., VAT + sales tax) and inclusive/exclusive amount handling.

### Scenario: REQ-TAX-003-A — Exclusive Rate (Net-Based)

```gherkin
GIVEN a TaxRate:
  | Field       | Value    |
  | rate        | 0.21     |
  | inclusive   | false    |
  | type        | standard |

AND an Invoice with:
  | Field     | Value  |
  | netAmount | €1,000 |

WHEN tax is calculated

THEN:
  | Field        | Value   |
  | taxAmount    | €210    |
  | grossAmount  | €1,210  |
  | calculation  | €1,000 × 0.21 |
```

### Scenario: REQ-TAX-003-B — Inclusive Rate (Gross-Based)

```gherkin
GIVEN a TaxRate:
  | Field       | Value    |
  | rate        | 0.21     |
  | inclusive   | true     |
  | type        | standard |

AND an Invoice with:
  | Field        | Value  |
  | grossAmount  | €1,210 |

WHEN tax is calculated (reverse calculation)

THEN:
  | Field       | Value   |
  | netAmount   | €1,000  |
  | taxAmount   | €210    |
  | calculation | €1,210 / 1.21 |
```

### Scenario: REQ-TAX-003-C — Compound Rate (VAT + Sales Tax)

```gherkin
GIVEN a TaxRate for Sales Tax:
  | Field        | Value     |
  | type         | salesTax  |
  | rate         | 0.08      |
  | compoundWith | vat-21    |

AND a base TaxRate (vat-21):
  | Field | Value |
  | rate  | 0.21  |

AND an Invoice with netAmount = €1,000

WHEN compound tax is calculated

THEN:
  | Step                         | Value   |
  | 1. Calculate base VAT        | €1,000 × 0.21 = €210 |
  | 2. Add VAT to base           | €1,000 + €210 = €1,210 |
  | 3. Calculate sales tax on total | €1,210 × 0.08 = €96.80 |
  | 4. Final gross amount        | €1,210 + €96.80 = €1,306.80 |
  | 5. Total tax                 | €306.80 |
```

---

## Requirement: REQ-TAX-004 — Multi-Jurisdiction Support

**Feature:** Multi-Jurisdiction VAT and Multi-State Tax Allocation

**Summary:** System supports transactions across multiple jurisdictions (NL, DE, BE, EU, International) with jurisdiction-specific rules, rates, and filing requirements.

### Scenario: REQ-TAX-004-A — Multi-Jurisdiction Transaction (Purchase)

```gherkin
GIVEN TaxConfiguration with:
  | Field                         | Value       |
  | multiJurisdictionEnabled      | true        |
  | jurisdictions                 | [NL, DE, BE]|

AND Purchase Invoices from multiple suppliers:
  | Supplier Country | Amount  | VAT Rate |
  | DE               | €5,000  | 19%      |
  | BE               | €3,000  | 21%      |
  | NL               | €2,000  | 21%      |

WHEN calculating total inbound VAT

THEN:
  | Jurisdiction | Net      | VAT    | Gross    |
  | DE           | €5,000   | €950   | €5,950   |
  | BE           | €3,000   | €630   | €3,630   |
  | NL           | €2,000   | €420   | €2,420   |
  | TOTAL        | €10,000  | €2,000 | €12,000  |

AND each line should be tagged with its jurisdiction for filing
```

### Scenario: REQ-TAX-004-B — Jurisdiction-Specific Filing Requirements

```gherkin
GIVEN a Tax Return spanning multiple jurisdictions:
  | Jurisdiction | Outbound VAT | Inbound VAT | Net Payable |
  | NL           | €5,000       | €2,000      | €3,000      |
  | DE           | €3,000       | €1,500      | €1,500      |

WHEN generating filing exports

THEN the system should create:
  | File               | Content                    |
  | NL-btw-aangifte    | NL-specific lines + €3,000 |
  | DE-tax-return      | DE-specific lines + €1,500 |
```

---

## Requirement: REQ-TAX-005 — VAT Detection & Recovery

**Feature:** VAT Detection & Recovery (Inbound/Outbound VAT Tracking)

**Summary:** System tracks inbound (purchase) and outbound (sales) VAT separately, calculates recoverable VAT, and applies recovery rules (exclusions, partial recovery).

### Scenario: REQ-TAX-005-A — Inbound VAT Recovery

```gherkin
GIVEN an Organization with recoveryRules:
  | Field                      | Value      |
  | inboundVATRecoveryAllowed  | true       |
  | partialRecoveryPercentage  | 100        |
  | excludedCategories         | [Meals, Entertainment] |

AND Purchase Invoices:
  | Category       | Amount | VAT Rate | VAT Amount |
  | Office Supplies| €1,000 | 21%      | €210       |
  | Meals          | €200   | 21%      | €42        |

WHEN calculating recoverable VAT

THEN:
  | Category       | Recoverable |
  | Office Supplies| €210 ✓      |
  | Meals          | €0 ✗        |
  | TOTAL          | €210        |
```

### Scenario: REQ-TAX-005-B — Partial VAT Recovery (Dual-Use Asset)

```gherkin
GIVEN recoveryRules with partialRecoveryPercentage = 80%

AND a Purchase Invoice:
  | Item               | Amount | VAT    |
  | Fleet Vehicle      | €50,000| €10,500|

WHEN calculating recoverable VAT

THEN:
  | Calculation                  | Value    |
  | Inbound VAT                  | €10,500  |
  | Recovery Percentage          | 80%      |
  | Recoverable VAT              | €8,400   |
  | Non-Recoverable VAT          | €2,100   |
```

---

## Requirement: REQ-TAX-006 — Tax Reports

**Feature:** Tax Reports (Compliance, Analytics, VAT Summary)

**Summary:** System generates tax reports showing inbound/outbound VAT, exemptions, recovery, tax liability, and period-over-period trends.

### Scenario: REQ-TAX-006-A — Monthly VAT Report

```gherkin
GIVEN transactions for May 2026:
  | Type      | Amount   | VAT Rate | VAT Amount |
  | Sales     | €50,000  | 21%      | €10,500    |
  | Purchases | €20,000  | 21%      | €4,200     |
  | Exempt    | €5,000   | 0%       | €0         |

WHEN generating May VAT Report

THEN the report should show:
  | Line Item                | Amount    |
  | Outbound VAT (Sales)     | €10,500   |
  | Inbound VAT (Purchases)  | €4,200    |
  | Recoverable Inbound      | €4,200    |
  | VAT Payable / Recoverable| €6,300    |
  | Transactions by Status   | 3 standard, 1 exempt |
```

### Scenario: REQ-TAX-006-B — Tax Summary Dashboard

```gherkin
GIVEN current fiscal year transactions

WHEN viewing Tax Dashboard

THEN display should show:
  | Widget                    | Value              |
  | YTD VAT Payable           | €18,500            |
  | YTD Inbound VAT           | €12,300            |
  | Recovery Rate             | 84%                |
  | Exemptions Applied        | 15 transactions    |
  | Pending Return Periods    | 2 (Apr, May)       |
```

---

## Requirement: REQ-TAX-007 — Auto-Classification

**Feature:** Auto-Classification of Invoices by Country, Language, VAT, Currency

**Summary:** System automatically detects and classifies invoice metadata (supplier country from address, language from OCR, VAT status, currency) to populate transaction fields.

### Scenario: REQ-TAX-007-A — Invoice OCR with Auto-Classification

```gherkin
GIVEN an uploaded invoice image (PDF/JPG):
  | Metadata    | Value                          |
  | Supplier    | "Siemens AG, Nürnberg, Germany"|
  | Language    | German                         |
  | VAT ID      | "DE123456789"                  |
  | Invoice Amount | "2.380,00 EUR"              |

WHEN the system processes the invoice

THEN it should auto-populate:
  | Field            | Detected Value |
  | supplierCountry  | DE             |
  | supplierLanguage | de             |
  | supplierId       | DE123456789    |
  | currency         | EUR            |
  | amount           | 2,380.00       |
```

### Scenario: REQ-TAX-007-B — Currency Conversion

```gherkin
GIVEN auto-classified invoice with currency = GBP, amount = £1,000

AND Organization with baseCurrency = EUR

WHEN invoice is imported

THEN system should:
  | Action                    |
  | 1. Detect currency mismatch |
  | 2. Query daily exchange rate |
  | 3. Convert to base currency |
  | 4. Store both amounts (GBP and EUR) |
  | 5. Tag transaction as "ForeignCurrency" |
```

---

## Requirement: REQ-TAX-008 — Tax Rate Configuration

**Feature:** Configurable Tax Rates with Compound and Inclusive/Exclusive Options

**Summary:** Administrators can create, modify, and maintain tax rates with effective dates, jurisdiction scoping, and compound rate dependencies.

### Scenario: REQ-TAX-008-A — Create New Tax Rate

```gherkin
GIVEN Admin is in Tax Rates section

WHEN creating a new rate:
  | Field       | Value              |
  | Country     | NL                 |
  | Type        | Reduced            |
  | Rate        | 9%                 |
  | Effective   | 2026-01-01         |
  | Inclusive   | false              |
  | Jurisdiction| NL                 |

THEN the system should:
  | Action              |
  | 1. Validate rate format (0.00–1.00) |
  | 2. Check for duplicates in period |
  | 3. Save with auditTrail |
  | 4. Display confirmation |
```

### Scenario: REQ-TAX-008-B — Rate Change with Effective Date

```gherkin
GIVEN a current TaxRate:
  | Field       | Value  |
  | Rate        | 0.21   |
  | ValidFrom   | 2025-01-01 |
  | ValidTo     | null   |

WHEN updating to new rate 0.24 effective 2026-06-01

THEN:
  | Action                        |
  | Set existing rate.validTo = 2026-05-31 |
  | Create new rate with rate=0.24, validFrom=2026-06-01 |
  | Audit: log user, timestamp, before/after values |
  | Notify: system message "Rate updated" |
```

---

## Requirement: REQ-TAX-009 — Multi-Jurisdiction VAT Rules

**Feature:** Multi-Jurisdiction VAT Rules (EU, International)

**Summary:** System applies jurisdiction-specific VAT rules (place of supply, reverse charge, distance selling) based on transaction context.

### Scenario: REQ-TAX-009-A — Place of Supply Rule (Goods)

```gherkin
GIVEN VAT Rule: "Goods are supplied where the buyer is established"

AND a Sales Invoice:
  | Field                  | Value       |
  | Supplier Location      | NL          |
  | Customer Location      | DE          |
  | Goods Type             | Physical    |
  | Transaction Type       | Sale        |

WHEN applying place-of-supply rule

THEN:
  | Determination        | Value   |
  | Place of Supply      | DE      |
  | Applicable VAT Rate  | DE 19%  |
  | Rule Applied         | place-of-supply-goods |
```

### Scenario: REQ-TAX-009-B — Distance Selling Rule

```gherkin
GIVEN VAT Rule: "B2C distance sales taxed where goods are dispatched from"

AND EU Distance Selling Threshold = €10,000 per country

AND Sales Invoices to consumers:
  | Country | Amount    | Count |
  | DE      | €8,000    | 15    |
  | FR      | €9,000    | 12    |
  | IT      | €2,000    | 3     |

WHEN checking distance selling registration requirement

THEN:
  | Country | Status                          |
  | DE      | Below threshold (€8,000)        |
  | FR      | Below threshold (€9,000)        |
  | IT      | Below threshold (€2,000)        |
  | TOTAL   | Registration NOT required       |
```

---

## Requirement: REQ-TAX-010 — Tax Configuration Settings

**Feature:** Tax Configuration Settings per Organization

**Summary:** System allows organizations to configure jurisdiction scope, filing frequency, exemption rules, detection rules, and recovery policies.

### Scenario: REQ-TAX-010-A — Initial Tax Configuration

```gherkin
GIVEN a new Organization with country = NL

WHEN the admin opens Tax Configuration

THEN the system should:
  | Action                    |
  | Display wizard with defaults |
  | Ask: Active jurisdictions (pre-select NL) |
  | Ask: Filing frequency (quarterly/annual) |
  | Ask: Enable auto-detection? (default: yes) |
  | Ask: Apply KOR exemption? |
  | Ask: Inbound VAT recovery (default: yes) |
  | Pre-load NL tax rates |
  | Pre-load NL detection rules |
```

### Scenario: REQ-TAX-010-B — Multi-Jurisdiction Activation

```gherkin
GIVEN Organization with current jurisdiction = NL

WHEN admin activates jurisdiction = DE

THEN system should:
  | Action                        |
  | 1. Load DE tax rates         |
  | 2. Load DE VAT rules         |
  | 3. Add DE to jurisdiction list |
  | 4. Enable DE-specific reporting |
  | 5. Notify: "DE taxes active" |
```

---

## Requirement: REQ-TAX-011 — Detection Rule Engine

**Feature:** Tax Detection Rule Engine with Priority and Condition Matching

**Summary:** Admin can create, order, and test detection rules that automatically classify transactions by country, supplier type, transaction type, and amount.

### Scenario: REQ-TAX-011-A — Create Detection Rule

```gherkin
GIVEN Admin in Detection Rules section

WHEN creating rule:
  | Field               | Value                |
  | Priority            | 5                    |
  | Name                | "Small Purchases"    |
  | supplierCountry     | NL                   |
  | transactionType     | Purchase             |
  | amountRange.max     | €1,000               |
  | outcome.vatStatus   | Standard             |
  | outcome.rate        | 21%                  |

THEN system should:
  | Action                      |
  | 1. Validate conditions      |
  | 2. Save rule with priority  |
  | 3. Display in rule list (ordered by priority) |
```

### Scenario: REQ-TAX-011-B — Test Rule Against Transaction

```gherkin
GIVEN a saved Detection Rule

AND a test transaction:
  | Field            | Value     |
  | supplierCountry  | NL        |
  | transactionType  | Purchase  |
  | amount           | €800      |

WHEN admin clicks "Test Rule"

THEN display:
  | Result                      |
  | Matches: Yes ✓              |
  | Suggested VAT Status: Standard |
  | Suggested Rate: 21%         |
```

---

## Requirement: REQ-TAX-012 — Exemption Management

**Feature:** Tax Exemption Management and Application

**Summary:** Admin can define, enable/disable, and test exemption rules; system automatically checks exemptions and applies them or requests confirmation.

### Scenario: REQ-TAX-012-A — Create Exemption Rule

```gherkin
GIVEN Admin in Tax Exemptions section

WHEN creating exemption:
  | Field               | Value                    |
  | exemptionType       | B2B                      |
  | jurisdiction        | NL                       |
  | validFrom           | 2024-01-01               |
  | appliesAutomatically| true                     |
  | supplierType        | Business                 |
  | transactionType     | Purchase                 |

THEN system should:
  | Action                      |
  | 1. Save exemption rule      |
  | 2. Display in exemptions list |
  | 3. Make available for auto-detection |
```

### Scenario: REQ-TAX-012-B — Manual Exemption Override

```gherkin
GIVEN an Invoice that system classified as "Standard VAT"

AND an Exemption rule that applies but was not triggered

WHEN user clicks "Apply Exemption"

THEN:
  | Action                          |
  | 1. Show available exemptions    |
  | 2. User selects exemption       |
  | 3. Invoice.vatStatus = Exempt   |
  | 4. Log override: user, rule, timestamp |
  | 5. Display: "Exemption applied by user" |
```

---

## User Stories (Derived from Features)

| Story ID | As a | I want to | So that |
|----------|------|-----------|---------|
| USR-TAX-001 | Bookkeeper | invoices to be auto-classified by VAT status | I don't manually enter VAT codes for each invoice |
| USR-TAX-002 | SMB Owner | KOR exemption applied automatically when under threshold | I get a VAT return of €0 when eligible |
| USR-TAX-003 | Accountant | multi-jurisdiction tax summary | I can file in multiple countries from one report |
| USR-TAX-004 | CFO | tax dashboard showing YTD payable and recovery rate | I know our tax position in real-time |
| USR-TAX-005 | Admin | configurable detection rules | rules adapt to our specific business model |
| USR-TAX-006 | Compliance Officer | exemption rules enforced with no manual approval | risk of non-compliance is eliminated |

---

## Acceptance Criteria (Summary)

### REQ-TAX-001 (VAT Auto-detection)
- [ ] At least 10 pre-built detection rules covering common scenarios (domestic, EU B2B, imports)
- [ ] Detection rules ordered by priority; first match applies
- [ ] Fallback to default VAT status if no rule matches
- [ ] Explanation visible on transaction showing which rule was applied

### REQ-TAX-002 (Tax Exemptions)
- [ ] KOR, B2B, ReverseCharge, ZeroRated exemptions supported
- [ ] Exemption conditions (amount threshold, supplier type) enforced
- [ ] Expired exemptions automatically disabled
- [ ] Exemption application logged with user/timestamp

### REQ-TAX-003 (Tax Rates)
- [ ] Exclusive rates (net-based) and inclusive rates (gross-based) both supported
- [ ] Compound rate calculation working (VAT + sales tax)
- [ ] Rate changes effective at specified date, no retroactive changes

### REQ-TAX-004 (Multi-Jurisdiction)
- [ ] At least 5 jurisdictions supported (NL, DE, BE, FR, and 1 international)
- [ ] Transaction tagged with jurisdiction for reporting
- [ ] Each jurisdiction has separate filing output

### REQ-TAX-005 (VAT Recovery)
- [ ] Inbound and outbound VAT tracked separately
- [ ] Recovery rules (exclusions, partial %) applied correctly
- [ ] Non-recoverable VAT clearly marked

### REQ-TAX-006 (Tax Reports)
- [ ] Monthly VAT summary showing outbound, inbound, payable
- [ ] Tax dashboard with YTD metrics and exemption count
- [ ] Report export to Excel/PDF

### REQ-TAX-007 (Auto-Classification)
- [ ] Supplier country detected from address, VAT ID, language
- [ ] Currency auto-detected and converted to base currency
- [ ] Detection shown as "auto" vs "manual" in audit trail

### REQ-TAX-008–012 (Configuration & Management)
- [ ] Tax rates, exemptions, rules manageable via UI (CRUD)
- [ ] Changes logged with before/after values
- [ ] Effective dates enforced (no retroactive changes)

---

**Status:** Ready for Tasks  
**Next:** Break into implementation tasks with test coverage
