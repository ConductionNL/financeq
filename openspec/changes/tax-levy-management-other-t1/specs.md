---
title: Tax & Levy Management — Requirements
app: shillinq
change: tax-levy-management-other-t1
---

# Specifications: Tax & Levy Management

## REQ-001: Tax Configuration Setup

**Scope:** Admin configures business tax rules (jurisdiction, VAT scheme, exemptions).

### REQ-001-001: Create Tax Configuration

```
GIVEN an administrator in the Tax Settings page
WHEN they fill in the Tax Configuration form:
  - Business Name: "Adviesbureau Amsterdam"
  - VAT Number: "NL987654321B01"
  - Jurisdiction: "NL"
  - Fiscal Year: 2026
  - VAT Scheme: "standard" | "kor" | "margin"
  - Default VAT Rate: 21%
  - Reverse Charge Enabled: true/false
  - ICP Declaration Required: true/false
AND they click "Save"
THEN a TaxConfiguration object is created in OpenRegister
  AND the slug is auto-generated (e.g., "tax-config-1")
  AND audit trail records the creation
  AND confirmation message: "Tax configuration saved"
```

### REQ-001-002: Update Tax Configuration

```
GIVEN an existing TaxConfiguration
WHEN an admin modifies:
  - Default VAT Rate: 21% → 19%
  - VAT Scheme: "standard" → "kor"
  - Reverse Charge: false → true
AND clicks "Save"
THEN the object is updated in OpenRegister
  AND audit trail records each field change with before/after value
  AND a notification is sent to all Finance Manager users
  AND historic VAT calculations are NOT retroactively changed
```

### REQ-001-003: Validate Jurisdiction Rules

```
GIVEN a VAT Scheme = "kor" (small business exemption)
  AND current revenue = €12,000
  AND annual revenue threshold = €20,000
WHEN tax configuration is saved
THEN system validates: currentRevenue ≤ annualRevenueThreshold
  AND validation passes
  AND ExemptionCertificate is auto-created with status "active"
ELSE validation fails with error: "KOR threshold exceeded; cannot use KOR scheme"
```

---

## REQ-002: Tax Rate Management

**Scope:** Create, update, validate tax rates per item/category.

### REQ-002-001: Create Multiple Tax Rates

```
GIVEN a Finance Manager in the Tax Rates page
WHEN they create three tax rates:
  1. "Standard VAT 21%" → rate: 21%, applies to: "professional-services, goods"
  2. "Reduced VAT 9%" → rate: 9%, applies to: "food, groceries"
  3. "Zero VAT 0%" → rate: 0%, applies to: "books, newspapers"
AND each rate includes:
  - Valid From: 2026-01-01
  - Valid Until: null (ongoing)
  - Reverse Charge: true/false
  - Exemption Code: null or "EU-reverse-charge" | "KOR" | "exempt"
AND they click "Save All"
THEN each TaxRate object is created
  AND all three rates are queryable via IndexService
  AND audit trail records creation of 3 objects
```

### REQ-002-002: Validate Tax Rate Against External Engine (AvaTax)

```
GIVEN a TaxRate creation form with:
  - rate: 21%
  - jurisdiction: "NL"
  - itemType: "professional-services"
AND AvaTax is configured
WHEN they click "Save"
THEN system calls AvaTax API: GetTaxRateByAddress()
  AND AvaTax returns: expectedRate = 21%
  AND validation passes
  AND TaxRate is saved
  AND no warning is shown

ALTERNATIVELY, if AvaTax returns: expectedRate = 17%
THEN validation shows WARNING:
  "Tax rate differs from AvaTax reference (21% vs 17%). Override?"
  AND user can click "Confirm" to proceed or "Cancel"
  AND if "Confirm", reason text is logged to audit trail
```

### REQ-002-003: Prevent Invalid Rate Combinations

```
GIVEN a TaxRate for "professional-services" with:
  - rate: 21%
  - reverseChargeApplies: false
  - jurisdiction: "NL"
WHEN an admin tries to set reverseChargeApplies: true
  AND itemType is NOT "B2B service" | "EU cross-border"
THEN validation error:
  "Reverse charge only applies to B2B services and EU trade; cannot enable for professional-services"
  AND save is blocked
```

---

## REQ-003: Taxable Transaction Recording

**Scope:** Mark transactions (invoices, expenses) as tax-relevant; extract tax data.

### REQ-003-001: Record Tax Data from Invoice

```
GIVEN an Invoice created in Shillinq:
  - Invoice #INV-2026-001
  - Amount: €5,000
  - Line items: [3x €1,000 consulting (21% VAT) + 2x €500 training (6% VAT)]
WHEN the invoice is finalized
THEN system auto-creates TaxableTransaction objects:
  1. transactionId: "invoice-2026-001"
     taxableAmount: €5,000
     vatAmount: €(5000 * 0.21 * 3/5) + (5000 * 0.06 * 2/5) = €610 + €120 = €730
     taxRate: "mixed 21% + 6%"
     taxCategory: "mixed-services"
  2. Child records for line-item detail (optional, via aggregation)
AND audit trail records the transaction recording
```

### REQ-003-002: Upload Receipt & Auto-Extract Tax Data (OCR)

```
GIVEN a PDF receipt for a business expense:
  - File: "Restaurant-Invoice-2026-05-20.pdf"
  - File size: 500 KB
WHEN user uploads to a TaxableTransaction form via "Attach Receipt" button
THEN system:
  1. Stores file via FileService
  2. Calls Scan & Herken API
  3. Returns extraction:
     {
       "merchant": "Restaurant De Groene Molen",
       "address": "Voetboogstraat 20, Amsterdam",
       "date": "2026-05-20",
       "items": [
         { "description": "Lunch", "amount": 25.00, "quantity": 1 },
         { "description": "Beverages", "amount": 8.50, "quantity": 2 }
       ],
       "subtotal": 42.00,
       "vatAmount": 8.82,
       "totalAmount": 50.82,
       "vatRate": 21,
       "confidence": 0.95
     }
  4. Auto-populates TaxableTransaction fields:
     - transactionDate: 2026-05-20
     - amount: 50.82
     - vatAmount: 8.82
     - taxRate: "21%"
     - ocrExtracted: true
     - ocrConfidence: 0.95
  5. Shows: "OCR extraction (95% confidence). Review and confirm?"
  6. User reviews and clicks "Confirm" → TaxableTransaction saved
ELSE if ocrConfidence < 0.90:
  Shows: "OCR confidence low (78%). Please manually enter data."
  AND requires manual entry
```

### REQ-003-003: Tag Transaction with Tax Category

```
GIVEN a recorded TaxableTransaction
WHEN a user clicks the "Tax Category" field
THEN a dropdown shows:
  [office-supplies, meals-entertainment, travel, professional-services,
   software-licenses, training, research-development, ...other-expenses]
WHEN user selects "meals-entertainment"
  AND that category is marked "Non-deductible (50% only)" in tax rules
THEN system flags:
  "This category allows only 50% deduction. Deductible amount: €25.41 / €50.82"
WHEN user clicks "Save"
THEN TaxableTransaction.taxCategory = "meals-entertainment"
  AND a Note is added: "50% deduction rule applied"
  AND audit trail records the categorization
```

---

## REQ-004: Tax Declaration Filing

**Scope:** Prepare, review, and file tax declarations (VAT, IV3, BCF, ESG).

### REQ-004-001: Create VAT Declaration (Quarter or Month)

```
GIVEN a Finance Manager in the Tax Declarations page
WHEN they click "New Declaration" and select:
  - Declaration Type: "VAT Return"
  - Period: "Q2 2026" (Apr 1 - Jun 30)
  - Jurisdiction: "NL"
THEN system:
  1. Creates a TaxDeclaration object with:
     - status: "draft"
     - periodStart: 2026-04-01
     - periodEnd: 2026-06-30
     - createdAt: now
  2. Auto-populates from TaxableTransaction records:
     - grossSales: sum of all sales invoices in Q2
     - totalVatCollected: sum of VAT on sales
     - totalVatDeductible: sum of VAT on purchases
     - netVatOwed: totalVatCollected - totalVatDeductible
  3. Shows: "Draft VAT Return Q2 2026 created. Review before filing."
  4. Displays summary: "Gross Sales: €18,000 | VAT Owed: €3,280"
  AND user can proceed to "Review" or "Edit"
```

### REQ-004-002: Review Declaration Before Filing

```
GIVEN a TaxDeclaration in "draft" status
  AND declaration data is:
    - grossSales: €18,000
    - totalVatCollected: €3,780
    - totalVatDeductible: €500
    - netVatOwed: €3,280
WHEN user clicks "Review"
THEN system transitions to "in-review" status
  AND displays:
    - Summary card: "VAT Return Q2 2026"
    - Line-by-line breakdown:
      * Sales with 21% VAT: €15,000 → VAT €3,150
      * Sales with 9% VAT: €3,000 → VAT €270
      * Professional services (0% VAT): €0 → VAT €0
      * Purchases: €2,500 VAT deductible
    - Button: "Drill into Q2 Transactions" → filters TaxableTransaction list to Q2
    - Button: "Run Consistency Checks"
    - Button: "Ready to File" | "Edit Draft"
WHEN user clicks "Run Consistency Checks":
THEN system validates:
  ✓ No duplicate transactions in period
  ✓ All line items have assigned tax category
  ✓ VAT calculations match transaction records
  ✓ Reverse charge applied correctly (if applicable)
THEN shows: "✓ All checks passed. Ready to file."
ELSE shows: "⚠ 2 issues found" with list and "Edit Draft" button
```

### REQ-004-003: Submit VAT Declaration to Belastingdienst

```
GIVEN a TaxDeclaration in "ready-to-file" status
  AND Belastingdienst integration is configured
  AND user has valid DigiSign certificate
WHEN user clicks "Submit to Belastingdienst"
  AND confirms: "Submit VAT Q2 2026 for €3,280?"
THEN system:
  1. Serializes declaration to Belastingdienst XSD format
  2. Calls DigiSign service to sign document
  3. POSTs to Belastingdienst API endpoint
  4. Receives filing reference: "VAT-NL-2026-Q2-98765"
  5. Updates TaxDeclaration:
     - status: "submitted"
     - submittedAt: now
     - filingReference: "VAT-NL-2026-Q2-98765"
     - submissionStatus: "pending"
  6. Shows: "VAT Return submitted! Reference: VAT-NL-2026-Q2-98765"
  7. Sends notification: "VAT return Q2 2026 submitted to Belastingdienst"
  8. Audit trail records submission with timestamp + reference
ELSE if API call fails:
  Shows: "Submission failed: [error message]. Retry?"
  AND TaxDeclaration remains in "ready-to-file" status
  AND user can retry after fixing issue
```

### REQ-004-004: Track Filing Status via Webhook

```
GIVEN a submitted TaxDeclaration with:
  - status: "submitted"
  - filingReference: "VAT-NL-2026-Q2-98765"
WHEN Belastingdienst accepts the filing:
THEN system receives webhook:
  {
    "event": "filing:accepted",
    "reference": "VAT-NL-2026-Q2-98765",
    "status": "accepted",
    "assessmentDate": "2026-07-10",
    "assessmentAmount": 3280.00
  }
AND updates TaxDeclaration:
  - status: "accepted"
  - submissionStatus: "accepted"
  - acceptanceDate: 2026-07-10
AND notifies user: "VAT Return accepted by Belastingdienst"
AND audit trail: "Filing accepted by Belastingdienst"
```

---

## REQ-005: Real-Time Tax Liability

**Scope:** Dashboard showing current-year tax estimate.

### REQ-005-001: Display Annual Income Tax Estimate

```
GIVEN a freelancer/sole proprietor in the Tax Dashboard
  AND current fiscal year: 2026
  AND current date: 2026-05-21
WHEN dashboard loads
THEN system displays "Tax Liability 2026 (to date)":
  - Gross Income (Jan-May): €23,500
  - Deductible Expenses (Jan-May): €3,200
  - Taxable Income (Jan-May): €20,300
  - Estimated Annual Taxable Income (extrapolated): €49,000
  - Estimated Annual Income Tax (21% rate): €10,290
  - Monthly Installment Recommended: €860
WHEN new invoice is recorded:
THEN estimate updates automatically within 5 seconds
WHEN new expense is categorized:
THEN deductible amount updates, tax estimate recalculates
```

### REQ-005-002: Tax Breakdown by Category

```
GIVEN the annual tax estimate showing:
  - Gross Income: €49,000
WHEN user clicks "Breakdown by Category"
THEN shows chart:
  - Consulting Services: 60% (€29,400)
  - Training & Workshops: 25% (€12,250)
  - Other Income: 15% (€7,350)
WHEN user clicks a category:
THEN filters transactions list to that category
  AND shows: "Training Income (Jan-May): €6,125 | Annualized: €14,700"
```

---

## REQ-006: Capital Gains & Investment Tax

**Scope:** Track investment income and capital gains.

### REQ-006-001: Record Capital Gains from Investment Sale

```
GIVEN an investor with:
  - Stock purchase: 100 shares @ €50 = €5,000 (Jan 2025)
  - Stock sale: 100 shares @ €65 = €6,500 (May 2026)
WHEN they record the sale as a TaxableTransaction with:
  - transactionType: "investment-sale"
  - purchasePrice: €5,000
  - salePrice: €6,500
  - saleDate: 2026-05-20
THEN system:
  1. Calculates capital gain: €6,500 - €5,000 = €1,500
  2. Determines holding period: >1 year (qualifies for reduced rate)
  3. Creates TaxableTransaction:
     - taxCategory: "long-term-capital-gain"
     - taxableAmount: €1,500
     - taxRate: "15%" (preferential long-term rate)
     - capitalGainsTax: €225
  4. Adds to annual tax estimate
  5. Audit trail: "Capital gain recorded: €1,500 sale (€5k cost basis)"
```

---

## REQ-007: Exemption Certificates

**Scope:** Manage tax exemptions (KOR, EU reverse charge, other).

### REQ-007-001: Create & Validate Exemption Certificate

```
GIVEN a small business with:
  - VAT Configuration: vatScheme = "kor"
  - Projected annual revenue: €18,000
  - Threshold for KOR: €20,000
WHEN admin clicks "Create Exemption Certificate"
  AND selects: "Kleine Ondernemersregeling (KOR) 2026"
THEN system:
  1. Validates: currentRevenue ≤ threshold (€18,000 ≤ €20,000) ✓
  2. Creates ExemptionCertificate:
     - certificateType: "small-business-exemption"
     - exemptionCode: "KOR"
     - validFrom: 2026-01-01
     - validUntil: 2026-12-31
     - status: "active"
     - currentRevenue: €18,000
  3. Shows: "KOR Certificate created. VAT filings exempt for 2026."
  4. Applies exemption to all invoices going forward (VAT rate = 0%)
ELSE if projected revenue exceeds threshold:
  Shows: "KOR threshold exceeded. Revenue forecast is €22,000; threshold is €20,000."
  AND exemption cannot be created
```

### REQ-007-002: Monitor Exemption Status

```
GIVEN a KOR exemption certificate (active, until 2026-12-31)
  AND current revenue tracked: €19,500
WHEN monthly tax report is generated
THEN shows:
  - KOR Certificate Status: "Active (expires 2026-12-31)"
  - Current Revenue: €19,500
  - Remaining Budget (before revocation): €500
  - Warning (if applicable): "⚠ Revenue approaching KOR threshold"
WHEN revenue exceeds threshold:
THEN system:
  1. Auto-transitions certificate status: "exceeded"
  2. Notifies user: "KOR threshold exceeded. Standard VAT (21%) applies going forward."
  3. Sets flag: VAT applies to future invoices
  4. Retroactive recalculation: None (KOR protection applies to year filed under)
```

---

## REQ-008: ESG Reporting (CSRD)

**Scope:** Map transactions to ESRS taxonomy; generate ESG reports.

### REQ-008-001: Map Transactions to ESRS Taxonomy

```
GIVEN a corporation preparing ESG report under CSRD
WHEN they initiate ESG Reporting workflow:
  - Reporting Year: 2025
  - Standard: "ESRS (Double Materiality)"
THEN system:
  1. Reviews all 2025 transactions for ESG relevance
  2. Maps expenses to ESRS categories:
     - Energy consumption: utilities invoices → "GRI 302: Energy"
     - Emissions: travel expenses → "GRI 305: Emissions"
     - Employee training: training invoices → "GRI 404: Training"
     - Waste: disposal invoices → "GRI 306: Waste"
  3. Aggregates by category:
     - Energy: €12,000 (80% of total utilities)
     - Emissions: €5,600 (travel by car/flight)
     - Training: €8,900
  4. Shows: "ESG mapping complete. 24 transactions mapped to 4 ESRS topics."
```

### REQ-008-002: Generate CSRD ESG Report

```
GIVEN mapped ESG transactions for 2025
WHEN user clicks "Generate CSRD Report"
THEN system:
  1. Compiles report structure:
     - Double Materiality Assessment
     - Material Topics (with transaction evidence)
     - ESRS-compliant disclosures
  2. Includes transaction detail:
     - Energy: "Utilities (€12,000): per GRI 302, Scope 2 emissions"
     - Travel: "Transport expenses (€5,600): per GRI 305, Scope 3 emissions"
  3. Exports as:
     - HTML report (viewable in browser)
     - PDF (downloadable, signed)
     - XBRL (for regulatory submission)
  4. Links supporting documents: invoices, receipts, sustainability statements
  5. Shows: "CSRD Report 2025 generated. Download or submit?"
```

---

## REQ-009: Tax Objection (Bezwaar)

**Scope:** Register and track formal tax objections.

### REQ-009-001: File Formal Tax Objection

```
GIVEN a business that disagrees with a tax assessment:
  - Assessment Reference: "2026/1234567"
  - Assessment Amount: €15,000
  - Disagreement: "VAT on professional services incorrectly assessed as 21% instead of 6%"
WHEN they click "File Objection (Bezwaar)"
  AND fill form:
    - Assessment Reference: [required]
    - Grounds for Objection: [multiline text]
    - Supporting Documents: [upload files]
    - Requested Outcome: "Reduce tax by €2,250 (difference between 21% and 6%)"
THEN system:
  1. Creates TaxDeclaration with:
     - declarationType: "tax-objection"
     - status: "objection-filed"
     - filedAt: now
     - objectionReference: "BEZWAAR-NL-2026-789"
  2. Submits to Belastingdienst via secure channel
  3. Receives confirmation: "Objection BEZWAAR-NL-2026-789 filed on 2026-05-21"
  4. Sets deadline: objectionResponseDeadline = now + 30 days
  5. Creates task: "Monitor objection response (due 2026-06-20)"
  6. Sends notification: "Objection filed. Reference: BEZWAAR-NL-2026-789"
```

### REQ-009-002: Track Objection Status

```
GIVEN a filed tax objection (BEZWAAR-NL-2026-789)
WHEN status query is made
THEN shows:
  - Status: "pending-assessment-authority-response"
  - Filed: 2026-05-21
  - Deadline: 2026-06-20
  - Days Remaining: 30
WHEN assessment authority responds (via webhook):
  {
    "event": "objection:accepted",
    "reference": "BEZWAAR-NL-2026-789",
    "status": "partially-granted",
    "newAssessmentAmount": 12750.00,
    "reasoning": "VAT rate reduced to 12% per EU ruling"
  }
THEN system:
  1. Updates TaxDeclaration:
     - status: "resolved"
     - objectionOutcome: "partially-granted"
     - newAssessmentAmount: €12,750
     - savedAmount: €2,250
  2. Notifies user: "Objection BEZWAAR-NL-2026-789 accepted. Tax reduced to €12,750."
  3. Audit trail: "Objection resolved; savings of €2,250"
```

---

## REQ-010: Tax Consistency & Validation

**Scope:** Pre-filing checks to prevent submission errors.

### REQ-010-001: IV3 Pre-Submission Consistency Checks

```
GIVEN a completed IV3 declaration for tax year 2025
WHEN user clicks "Run Consistency Checks" before submission
THEN system validates:
  ✓ CHECK-1: No duplicate transactions detected
    - Result: ✓ All 47 transactions are unique by date+amount+reference
  ✓ CHECK-2: All transactions have assigned tax category
    - Result: ✓ 100% (47/47 have category)
  ✓ CHECK-3: Gross income matches bank deposits
    - Result: ⚠ Bank deposits total €150,500; IV3 gross income €151,000 (Δ €500)
      → Suggestion: "Review 5 invoices from December; possible duplicate deposit?"
  ✓ CHECK-4: Deductible expenses are valid per tax code
    - Result: ⚠ Travel expenses €5,600 (meals included €1,200 at 50% deduction)
      → Suggestion: "Meals deductible at 50%; confirm manually"
  ✓ CHECK-5: No outstanding invoices in prior year
    - Result: ✓ No unbilled services from 2024
THEN shows summary:
  "5/5 checks complete. ⚠ 2 warnings (review recommended). Proceed to file?"
```

---

## REQ-011: Tax Data Import & Export

**Scope:** Bulk import/export tax configuration and transaction data.

### REQ-011-001: Import Tax Rates from CSV

```
GIVEN an Excel file with tax rates:
  | Name | Rate | Jurisdiction | Valid From | Valid Until |
  | Standard VAT | 21% | NL | 2026-01-01 | null |
  | Reduced VAT | 9% | NL | 2026-01-01 | null |
  | Zero VAT | 0% | NL | 2026-01-01 | null |
WHEN Finance Manager clicks "Import Tax Rates"
  AND selects the CSV file
THEN system:
  1. Parses and validates each row
  2. Checks for duplicates (by name + jurisdiction)
  3. Shows preview: "3 tax rates ready to import"
  4. On confirmation, creates 3 TaxRate objects
  5. Shows: "✓ 3 tax rates imported successfully"
  AND audit trail records bulk import
```

### REQ-011-002: Export Tax Declaration as PDF/JSON

```
GIVEN a completed TaxDeclaration (VAT Q2 2026, status: "accepted")
WHEN user clicks "Export"
  AND selects format: "PDF" | "JSON" | "XBRL"
THEN:
  - **PDF:** Renders filing summary with Belastingdienst reference, line items, signature block
  - **JSON:** Exports full object structure (for audit archival)
  - **XBRL:** Generates taxonomy-compliant XML for regulatory submission
WHEN download completes:
THEN file is saved: "TAX-VAT-Q2-2026-ACCEPTED.pdf"
```

---

## REQ-012: Notifications & Alerts

**Scope:** Deadline reminders, status updates, validation warnings.

### REQ-012-001: Filing Deadline Alerts

```
GIVEN a TaxConfiguration with fiscalYear = 2026
WHEN each quarter completes:
THEN system schedules notification:
  - Q1 deadline: Mar 31 → alert sent Mar 20 (11 days before)
  - Q2 deadline: Jun 30 → alert sent Jun 19 (11 days before)
  - Q3 deadline: Sep 30 → alert sent Sep 19 (11 days before)
  - Q4 deadline: Dec 31 → alert sent Dec 20 (11 days before)
WHEN alert is triggered:
THEN notification sent to all Finance Manager users:
  "📋 VAT Return deadline approaching: Q2 filing due Jun 30 (11 days remaining)"
  AND includes link: "Prepare VAT Return →"
```

### REQ-012-002: Validation Warnings on Declaration Creation

```
GIVEN a TaxDeclaration being created with:
  - grossSales: €0 (no transactions recorded)
  - period: "Q2 2026"
WHEN system calculates totals
THEN shows warning:
  "⚠ Warning: No sales transactions recorded for Q2. Proceed?"
  AND user can:
    - "Proceed" → creates empty declaration (correct for quarter with no activity)
    - "Cancel" → returns to form
```

---

## Acceptance Criteria Summary

| Req | Criterion | Pass |
|-----|-----------|------|
| REQ-001 | Tax configuration created, updated, validated | - |
| REQ-002 | Multiple tax rates created, external validation works | - |
| REQ-003 | Transactions recorded; OCR extraction ≥90% confidence | - |
| REQ-004 | VAT/IV3 declarations filed to Belastingdienst; webhooks process status | - |
| REQ-005 | Annual tax estimate updates in real-time on dashboard | - |
| REQ-006 | Capital gains calculated; held >1 year qualifies for reduced rate | - |
| REQ-007 | KOR exemption created; monitored; revoked if threshold exceeded | - |
| REQ-008 | Transactions mapped to ESRS; CSRD report generated | - |
| REQ-009 | Objection filed; status tracked; webhook response processed | - |
| REQ-010 | IV3 pre-submission checks detect issues; allow/block filing | - |
| REQ-011 | Tax rates imported via CSV; declarations exported as PDF/JSON/XBRL | - |
| REQ-012 | Filing deadline notifications sent 11 days before due date | - |
