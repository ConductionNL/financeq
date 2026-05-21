# Tax & Levy Management — Requirements Specification

## REQ-TAX-001: Tax Code Library CRUD

**Title:** System must provide admin interface to create, read, update, delete tax codes

**Description:** Tax codes are the foundation of VAT calculation. Admins must be able to define new codes (when jurisdiction changes tax rates), update existing rates (effective date changes), and deactivate expired codes.

**Acceptance Criteria:**

- GIVEN an admin user accessing Settings → Tax Configuration
- WHEN they click "Add Tax Code"
- THEN a form appears with required fields: code, name, jurisdiction, tax type, rate, effective date, calculation method, applicable-to checkboxes
- AND the form validates: rate ∈ [0, 1], effective date ≤ today or future, expiry date > effective date
- AND on save, the code is stored in OpenRegister TaxCode schema
- AND the code appears in the tax code library list (paginated, sortable by date, jurisdiction)

- GIVEN a tax code with an error in the name
- WHEN an admin edits the code and corrects the name
- THEN the change is saved and all active uses (on invoices, in schemes) see the updated name immediately
- AND an audit log entry records the change with admin user + timestamp

- GIVEN an admin tries to delete an active tax code used by current invoices
- WHEN they click delete
- THEN the system displays: "This code is used by N invoices; deactivate instead (set expiry date)"
- AND the delete is blocked until all uses are migrated or the code is marked inactive

---

## REQ-TAX-002: VAT Calculation — Exclusive Method

**Title:** System must calculate VAT on invoice lines using exclusive method (tax added to base)

**Description:** Standard VAT in Netherlands is exclusive: invoiced amount = base + VAT. Given a line item with base amount and tax code, the system calculates VAT and total.

**Acceptance Criteria:**

- GIVEN an invoice line with:
  - Base amount: €100
  - Tax code: VAT-21% (rate = 0.21, method = EXCLUSIVE)
- WHEN the system calculates tax
- THEN VAT amount = €100 × 0.21 = €21
- AND total = €100 + €21 = €121
- AND the calculation is stored in TaxableTransaction.taxAmount and visible on the invoice

- GIVEN the same line but tax code is VAT-6% (rate = 0.06)
- WHEN the system calculates tax
- THEN VAT = €100 × 0.06 = €6
- AND total = €106

- GIVEN an invoice with multiple lines under different tax codes:
  - Line 1: €100 @ VAT-21% = €121 total
  - Line 2: €50 @ VAT-6% = €53 total
- WHEN the system sums the invoice
- THEN invoice total = €121 + €53 = €174
- AND aggregation shows: VAT 21% total = €21, VAT 6% total = €3

---

## REQ-TAX-003: VAT Calculation — Inclusive Method

**Title:** System must calculate VAT on amounts already including tax (reverse calculation)

**Description:** Some invoices express amounts as inclusive (the €100 you see already contains VAT). The system must reverse-calculate the VAT portion.

**Acceptance Criteria:**

- GIVEN an inclusive invoice line with:
  - Total amount: €121 (includes VAT)
  - Tax code: VAT-21% (rate = 0.21, method = INCLUSIVE)
- WHEN the system calculates VAT (reverse)
- THEN: VAT = €121 - (€121 / 1.21) = €121 - €100 = €21
- AND base amount = €100
- AND TaxableTransaction records both VAT and base correctly

- GIVEN the inclusive amount is wrong (e.g., €115 for 21% inclusive)
- WHEN the system validates the amount
- THEN it either: (a) auto-corrects to nearest valid amount (€121 for base €100), or (b) flags as invalid with reason
- AND the user can override the validation with a note

---

## REQ-TAX-004: Compound Tax Calculation

**Title:** System must handle compound taxes (tax-on-tax) such as VAT on import duty

**Description:** Imported goods incur import duty (on base), then VAT is applied to (base + duty). The calculation order matters.

**Acceptance Criteria:**

- GIVEN an import scenario:
  - Base cost: €1000
  - Import duty: 5% (IMPORT_DUTY code, EXCLUSIVE)
  - VAT: 21% on top (VAT-21, EXCLUSIVE, dependsOnTaxCode = IMPORT_DUTY)
- WHEN the system calculates taxes:
  - Step 1: Import duty = €1000 × 0.05 = €50
  - Step 2: VAT base = €1000 + €50 = €1050
  - Step 3: VAT = €1050 × 0.21 = €220.50
- THEN TaxableTransaction.taxAmount shows the final VAT (€220.50) AND the compound calculation is visible/auditable
- AND invoice total = €1000 + €50 + €220.50 = €1270.50

- GIVEN the same import but with a different duty code (10% instead of 5%)
- WHEN the system recalculates
- THEN the VAT base updates: €1000 + €100 = €1100, VAT = €231
- AND the total updates to €1331

---

## REQ-TAX-005: Tax Code Validity Check

**Title:** System must only apply active tax codes to transactions

**Description:** A tax code may have an effective date in the future or an expiry date in the past. The system must enforce that only active codes can be applied to new transactions.

**Acceptance Criteria:**

- GIVEN a tax code with effectiveDate = 2026-06-01 (future)
- AND today's date is 2026-05-20
- WHEN a user tries to apply this code to an invoice line
- THEN the system blocks the application with message: "This code is not yet active (effective 2026-06-01)"

- GIVEN a tax code with expiryDate = 2026-05-31 (past)
- AND today's date is 2026-06-01
- WHEN a user tries to apply this code
- THEN the system blocks with message: "This code expired on 2026-05-31"

- GIVEN a code with effectiveDate = 2026-01-01, expiryDate = null (indefinite)
- AND today = any date ≥ 2026-01-01
- WHEN applying the code
- THEN the system allows the application

---

## REQ-TAX-006: Tax Scheme Association

**Title:** Organization or supplier can be linked to a tax scheme defining their tax regime

**Description:** A freelancer under KOR exemption, a standard VAT trader, or a supplier subject to reverse-charge must be able to declare their tax scheme in Shillinq. When creating invoices or payments involving that entity, the system applies the appropriate tax codes or exemptions.

**Acceptance Criteria:**

- GIVEN an organization with TaxScheme = "Freelancer KOR"
- AND the scheme specifies: exemptTransactionTypes = ["INVOICE_SALE"], applicableTaxCodes = []
- WHEN creating a sales invoice for this organization
- THEN the system does NOT offer VAT tax codes as options
- AND if a line item is added, it is automatically marked as non-taxable
- AND the invoice shows "KOR exempt" note

- GIVEN an organization with TaxScheme = "Standard VAT Trader"
- AND the scheme specifies: applicableTaxCodes = ["VAT-21", "VAT-6"]
- WHEN creating a sales invoice
- THEN the system offers only VAT-21 and VAT-6 as available codes
- AND applies VAT-21 by default (standard rate)

- GIVEN a supplier with withholding_tax_scheme = "withholding-20pct"
- WHEN recording a payment to that supplier
- THEN the system automatically calculates 20% withholding on the gross payment
- AND stores: gross amount, withholding amount, net amount paid

---

## REQ-TAX-007: KOR Exemption Processing

**Title:** Freelancers under KOR (small business exemption) must be able to operate tax-exempt

**Description:** Dutch KOR scheme allows freelancers with turnover < €50,000 annual to be VAT-exempt. They do not collect VAT (no output) but also cannot reclaim input VAT. Shillinq must support this mode.

**Acceptance Criteria:**

- GIVEN a freelancer with TaxScheme = "KOR"
- AND fiscal year turnover = €45,000
- WHEN generating the annual tax summary
- THEN system shows:
  - Sales invoiced: €45,000 (no VAT collected)
  - No output VAT due
  - No input VAT deductions allowed
  - Note: "KOR exemption active; no VAT return required"

- GIVEN the same freelancer with fiscal year turnover = €52,000 (exceeds €50,000 threshold)
- WHEN the system reviews the threshold
- THEN it flags: "Turnover exceeds €50,000; KOR exemption may be revoked. Consult tax advisor."

- GIVEN a KOR-exempt freelancer with input invoices containing VAT
- WHEN trying to claim deductions in a tax return
- THEN the system prevents it with message: "KOR status does not allow input VAT deduction"

---

## REQ-TAX-008: Provisional vs Actual Tax Reconciliation

**Title:** Organizations must be able to track provisional tax payments and reconcile against actual calculated liability

**Description:** Corporations and some SMBs pay voorlopige aanslag (provisional) tax instalments quarterly or monthly. At year-end, actual liability is calculated, and the balance (refund or additional payment) is determined.

**Acceptance Criteria:**

- GIVEN an organization that paid provisional taxes:
  - Q1 2026: €5,000
  - Q2 2026: €5,000
  - Q3 2026: €5,000
  - Q4 2026: €5,000
  - Total paid: €20,000
- WHEN calculating actual VAT liability for 2026:
  - Output VAT (sales): €25,000
  - Deductible input VAT (purchases): €18,000
  - Net VAT due: €7,000
- THEN TaxDeclaration shows:
  - Calculated liability: €7,000
  - Provisional payments: €20,000
  - Balance: -€13,000 (refund due)

- GIVEN the same org with actual liability = €22,000
- WHEN the balance is calculated
- THEN: balance = €22,000 - €20,000 = €2,000 (additional due)
- AND TaxDeclaration.balanceDue = €2,000
- AND the system can generate a payment notice for €2,000

---

## REQ-TAX-009: VAT Return Drafting

**Title:** System must aggregate quarterly VAT transactions and draft a VAT return ready for filing

**Description:** At quarter-end, the system collects all invoices and payments for that period, calculates net VAT, and drafts a return (aangifte) that can be filed with Dutch tax authorities or reviewed by an accountant.

**Acceptance Criteria:**

- GIVEN a Q2 2026 (Apr-Jun) transaction history:
  - Sales invoices (all VAT-21): €50,000 (VAT collected: €10,500)
  - Purchase invoices (various codes):
    - VAT-21 purchases: €20,000 (input VAT: €4,200)
    - VAT-6 purchases: €5,000 (input VAT: €300)
  - Total input VAT: €4,500
- WHEN creating a Q2 2026 VAT return
- THEN TaxDeclaration shows:
  - Declaration type: VAT_QUARTERLY
  - Period: 2026-04-01 to 2026-06-30
  - Total output VAT: €10,500 (box 1)
  - Total input VAT: €4,500 (box 2)
  - Net VAT due: €6,000
  - Status: DRAFT

- GIVEN the draft return
- WHEN the user clicks "File with Tax Authority"
- THEN the system:
  - Locks the declaration (no further edits without amendment)
  - Sets status to FILED
  - Records filingDate = today
  - Generates a reference number (for tracking with authorities)

---

## REQ-TAX-010: Income Tax Categorization

**Title:** System must categorize transactions for income tax return preparation (IB-aangifte)

**Description:** Sole proprietors file annual income tax returns (IB-aangifte) showing gross income, deductible expenses, capital gains, etc. Shillinq must allow categorization of transactions into these buckets.

**Acceptance Criteria:**

- GIVEN a freelancer's fiscal year transactions:
  - Invoice payments received: €60,000 (revenue)
  - Office rent (12 months): €10,000 (deductible expense)
  - Internet/phone: €1,500 (deductible expense)
  - Laptop purchase: €2,000 (capital asset, depreciation over 5 years = €400/year)
  - Personal drawing: €15,000 (not deductible; private)
  - Interest on business loan: €500 (deductible)
- WHEN creating an income tax return
- THEN TaxDeclaration shows:
  - Gross income: €60,000
  - Deductible business expenses: €10,000 + €1,500 + €400 (depreciation) + €500 = €12,400
  - Taxable income: €60,000 - €12,400 = €47,600
  - Personal drawings and capital purchases listed separately (not reducing taxable income)

- GIVEN the same freelancer with a side rental income (€8,000) in the same year
- WHEN the return is recalculated
- THEN gross income = €60,000 + €8,000 = €68,000
- AND taxable income = €68,000 - €12,400 = €55,600

---

## REQ-TAX-011: Supplementary VAT Return (Suppletie-aangifte)

**Title:** System must support supplementary VAT returns for corrections

**Description:** If errors are discovered in a filed VAT return, a supplementary return (suppletie-aangifte) can be filed to correct the prior period.

**Acceptance Criteria:**

- GIVEN a Q2 2026 VAT return already FILED with net VAT due = €6,000
- AND an error is discovered: a purchase invoice for €2,000 was omitted
- WHEN creating a supplementary return:
  - Declaration type: SUPPLEMENTARY_VAT
  - References prior declaration
  - Adjustment: additional input VAT = €420 (if VAT-21)
  - New net VAT due: €6,000 - €420 = €5,580
- THEN the system shows:
  - Correction reason/notes (for documentation)
  - Status: DRAFT (until filed)
  - When filed, both original and supplementary returns are linked in audit trail

---

## REQ-TAX-012: Withholding Tax Tracking

**Title:** System must track withholding taxes on supplier payments

**Description:** Some organizations are required to withhold tax from supplier payments (e.g., freelancer withholding in certain scenarios). The system must record gross, withheld, and net amounts, and provide a summary for reconciliation.

**Acceptance Criteria:**

- GIVEN a supplier payment scenario:
  - Gross amount due: €1,000
  - Supplier is subject to withholding: 20%
  - Tax withheld: €1,000 × 0.20 = €200
  - Net amount paid to supplier: €800
- WHEN recording this payment
- THEN the system captures:
  - Gross amount: €1,000
  - Withholding rate: 20%
  - Withholding tax amount: €200
  - Net amount paid: €800
  - Supplier's TaxScheme.withholding_tax_scheme = "withholding-20pct"

- GIVEN an end-of-year report on withheld taxes
- WHEN querying all withholding transactions for a fiscal year
- THEN system shows:
  - Total gross amounts: €10,000
  - Total withheld: €2,000 (sum of all individual withholdings)
  - Broken down by supplier and withholding rate
  - Ready for reporting to tax authority or supplier reconciliation

---

## REQ-TAX-013: Invoice Tax Rounding

**Title:** System must apply correct rounding to tax calculations per invoice (not per line)

**Description:** VAT is calculated and rounded per invoice (or per line, depending on jurisdiction rules). Shillinq must apply the appropriate rounding to avoid discrepancies.

**Acceptance Criteria:**

- GIVEN an invoice with three lines:
  - Line 1: €10.00 @ 21% = VAT €2.10
  - Line 2: €10.00 @ 21% = VAT €2.10
  - Line 3: €10.01 @ 21% = VAT €2.10 (exactly)
  - Total base: €30.01
  - Total VAT (exact): €6.30
  - Rounded (to 2 decimals): €6.30
- WHEN the invoice is finalized
- THEN invoice total = €30.01 + €6.30 = €36.31
- AND no rounding discrepancies occur

- GIVEN an invoice where line-by-line rounding differs from invoice-level rounding:
  - Line 1: €10.001 @ 21% = €2.1002 → rounds to €2.10
  - Line 2: €10.001 @ 21% = €2.1002 → rounds to €2.10
  - Total exact: €4.2004 → rounds to €4.20
  - OR line sums: €2.10 + €2.10 = €4.20 (same, no difference)
- THEN the system applies consistent rounding (recommend: per-invoice, not per-line)
- AND documents the rounding method in the invoice schema

---

## REQ-TAX-014: DNI/NIF Validation (ID Number Validation)

**Title:** System must validate Spanish/Portuguese tax ID numbers (DNI/NIF)

**Description:** For organizations in Spain or Portugal, the system should validate the format and check digit of their tax ID (DNI for Spain, NIF for Portugal).

**Acceptance Criteria:**

- GIVEN a Spanish supplier with NIF: 12345678-A
- WHEN the system validates the NIF
- THEN it checks:
  - Format: 8 digits + hyphen + 1 letter
  - Check digit: calculated from modulo-23 algorithm
  - If valid: stored as-is; used in reporting
  - If invalid: system rejects with "Invalid NIF format or check digit"

- GIVEN a Portuguese supplier with NIF: 505196215
- WHEN validating
- THEN it checks: 9-digit format, valid per Portuguese algorithm
- AND accepts if valid; rejects if not

---

## REQ-TAX-015: CNPJ/CPF Validation (Brazilian Tax IDs)

**Title:** System must validate Brazilian tax identification numbers

**Description:** For Brazilian organizations (CNPJ) or individuals (CPF), the system should validate the format and check digits.

**Acceptance Criteria:**

- GIVEN a Brazilian company with CNPJ: 11.222.333/0001-81
- WHEN validating
- THEN system checks: format (##.###.###/####-##), modulo-11 check digits (positions 11 and 12)
- AND accepts if valid per CNPJ algorithm; rejects if not

- GIVEN a Brazilian individual with CPF: 123.456.789-09
- WHEN validating
- THEN system checks: format (###.###.###-##), modulo-11 check digits
- AND accepts if valid per CPF algorithm; rejects if not

---

## REQ-TAX-016: Tax Category Mapping to GL Accounts

**Title:** System must link tax categories to general ledger accounts for journal entry posting

**Description:** When a transaction is categorized as VAT output, VAT deductible, or other tax type, it must be linked to the correct GL account for accurate financial reporting.

**Acceptance Criteria:**

- GIVEN a TaxableTransaction with taxCategory = "VAT_OUTPUT"
- AND organization's GL chart defines account "2001" = "VAT Output (Short-term Liability)"
- WHEN posting the transaction to the journal
- THEN:
  - Sales account (e.g., "3100") is credited €100 (base)
  - VAT Output account (e.g., "2001") is credited €21 (VAT)
  - Total credit: €121

- GIVEN a purchase transaction with taxCategory = "VAT_DEDUCTIBLE"
- AND GL account "1501" = "VAT Deductible (Asset)"
- WHEN posting
- THEN:
  - Expense account is debited €100
  - VAT Deductible account is debited €21
  - Accounts Payable is credited €121

---

## REQ-TAX-017: RJ Taxonomy Support (Dutch Reporting Taxonomy)

**Title:** System must support RJ (Richtlijnen) taxonomy for Dutch financial reporting

**Description:** Dutch accounting standards (RJ = Richtlijnen Jaarverslaglegging) define GL account classifications and reporting templates. Shillinq should map tax transactions to RJ categories where applicable.

**Acceptance Criteria:**

- GIVEN Dutch tax codes (VAT-21, VAT-6, etc.)
- AND RJ taxonomy defines standard GL account structures
- WHEN the system generates a Dutch tax return
- THEN it maps:
  - Sales revenue → RJ account "8001" (Sales of goods) or "8100" (Services)
  - VAT output → RJ "8200" (VAT collected)
  - VAT deductible → RJ "8210" (VAT recoverable)
- AND the export is structured per RJ format

---

## REQ-TAX-018: AGT Taxonomy Support (Agricultural Goods Taxonomy)

**Title:** System must support AGT (Agricultural Goods Taxonomy) for agricultural business classification

**Description:** Agricultural organizations may use a specialized tax regime. Shillinq should support classification of transactions under AGT if applicable.

**Acceptance Criteria:**

- GIVEN an agricultural supplier (farm, dairy, etc.)
- AND associated with TaxScheme with AGT classification
- WHEN creating transactions
- THEN system offers AGT-specific tax codes (if any)
- AND tagging transactions as "agricultural goods" enables specialized VAT reporting
- AND the system can export per agricultural-specific formats

---

## REQ-TAX-019: Corporate Income Tax (VPB) Declaration Support

**Title:** System must support corporate income tax return preparation (VPB-aangifte)

**Description:** Corporations file annual corporate income tax returns (VPB = Vennootschapsbelasting). The system must collect profit/loss and apply corporate tax rates.

**Acceptance Criteria:**

- GIVEN a corporation with fiscal year transactions:
  - Gross revenues: €500,000
  - Deductible business expenses: €350,000
  - Depreciation: €20,000
  - Other deductions: €5,000
  - Net profit: €500,000 - €350,000 - €20,000 - €5,000 = €125,000
- WHEN creating a VPB return
- THEN TaxDeclaration shows:
  - Declaration type: CORPORATE_ANNUAL
  - Taxable profit: €125,000
  - Corporate tax rate: 19% (standard rate as of 2026)
  - Corporate tax due: €125,000 × 0.19 = €23,750
  - Status: DRAFT

- GIVEN the same corp with profit = €50,000 (lower bracket)
- WHEN filing
- THEN system applies lower rate (if applicable): e.g., 15% for first €200,000
- AND shows: tax due = €50,000 × 0.15 = €7,500

---

## REQ-TAX-020: Tax Planning / Forecasting

**Title:** System must provide tax liability forecasting based on YTD results

**Description:** Organizations want to estimate year-end tax liability and plan quarterly payments based on current performance.

**Acceptance Criteria:**

- GIVEN year-to-date data (Jan-May 2026):
  - Revenue: €40,000
  - Expenses: €25,000
  - Net profit (YTD): €15,000
- WHEN the user requests a forecast for full year 2026
- THEN system extrapolates:
  - Projected annual revenue: €40,000 ÷ 5 months × 12 months = €96,000
  - Projected annual expenses: €25,000 ÷ 5 × 12 = €60,000
  - Projected profit: €36,000
  - Projected VAT liability (at 21% standard): €36,000 × 0.21 = €7,560
  - Recommended quarterly provision: €7,560 ÷ 4 = €1,890/quarter

- GIVEN the forecast
- WHEN actual June results come in and profit is higher than forecast
- THEN user can update the forecast and adjust quarterly payment plans

---

## REQ-TAX-021: Private vs Business Expense Splitting

**Title:** System must allow splitting of expenses between private and business portions

**Description:** Some expenses (vehicles, home office, etc.) are partly private and partly deductible. The system must track both portions and exclude the private portion from tax deductions.

**Acceptance Criteria:**

- GIVEN an expense: car lease €500/month
- WHEN recording the transaction
- THEN user can split:
  - Business use: 60% = €300 (deductible)
  - Private use: 40% = €200 (not deductible)
- AND the system records both in TaxableTransaction
- AND year-end report shows only €300/month (€3,600 annual) as deductible

- GIVEN a home-office expense: rent €1,500/month
- WHEN allocating home office use = 25% (1 room out of 4)
- THEN deductible portion = €1,500 × 0.25 = €375/month
- AND system tracks the private-use portion separately
- AND neither enters the tax return

---

## REQ-TAX-022: Pillar Two Global Minimum Tax Calculation

**Title:** System must calculate OECD Pillar Two global minimum tax liability (15%)

**Description:** Multinational enterprises (MNEs) with global revenue > €750M are subject to OECD Pillar Two (GloBE) minimum tax of 15% on global profits. Shillinq should support this calculation for qualifying organizations.

**Acceptance Criteria:**

- GIVEN a multinational organization with:
  - Global revenue: €1,000,000,000 (exceeds €750M threshold)
  - Global profit: €100,000,000
  - Current tax rate (weighted average): 12% (below 15% minimum)
  - Pillar Two top-up tax: (€100,000,000 × 0.15) - (€100,000,000 × 0.12) = €3,000,000
- WHEN calculating Pillar Two liability
- THEN system shows:
  - Global profit: €100M
  - Global effective tax rate: 12%
  - Shortfall vs 15%: 3%
  - Top-up tax required: €3,000,000

- GIVEN a different organization with:
  - Global profit: €100,000,000
  - Current tax rate: 16% (above 15%)
- WHEN calculating Pillar Two
- THEN system shows:
  - Effective rate: 16%
  - No top-up required (compliant with 15% minimum)

---

## REQ-TAX-023: VAT Reclaim Process

**Title:** System must support VAT reclaim (negative VAT return) when input VAT exceeds output VAT

**Description:** If a business incurs more input VAT (purchases, imports) than output VAT (sales), they can claim a refund. The system must identify and facilitate this process.

**Acceptance Criteria:**

- GIVEN Q2 2026 transactions:
  - Output VAT (sales): €2,000
  - Input VAT (purchases): €5,000
  - Net VAT: €2,000 - €5,000 = -€3,000 (reclaim due)
- WHEN creating a VAT return for Q2
- THEN system shows:
  - Status: RECLAIM
  - Reclaim amount: €3,000
  - Filed: Marked as "Reclaim filed" when submitted
  - Subsequent status updates as refund is processed

- GIVEN the reclaim has been filed
- WHEN the tax authority processes it and deposits €3,000
- THEN user records the refund payment
- AND system marks the declaration as CLOSED

---

## REQ-TAX-024: Multi-Jurisdiction Tax Reporting

**Title:** System must handle taxes across multiple jurisdictions (NL, EU, non-EU)

**Description:** Organizations operating in multiple countries may need to track tax obligations in each jurisdiction separately.

**Acceptance Criteria:**

- GIVEN an organization with operations in:
  - Netherlands (NL)
  - Germany (DE)
  - UK (GB)
- WHEN filtering tax codes by jurisdiction
- THEN system shows:
  - NL codes: VAT-21, VAT-6, etc.
  - DE codes: VAT-19, VAT-7, etc.
  - GB codes: VAT-20, VAT-5, etc.
- AND transactions can be tagged by jurisdiction
- AND tax returns can be generated per jurisdiction

---

## REQ-TAX-025: Audit Trail and Immutability

**Title:** All tax transactions and calculations must be immutable and audit-trailed

**Description:** Tax filings are subject to audits. The system must provide a complete, tamper-evident audit trail showing every calculation, change, and filing action.

**Acceptance Criteria:**

- GIVEN a TaxableTransaction created on 2026-01-15
- WHEN viewing the audit trail
- THEN system shows:
  - Created by: user@example.com, 2026-01-15 09:30:00
  - Applied tax code: VAT-21 @ rate 0.21
  - Calculated VAT: €21.00
  - Any edits (e.g., correcting amount) show before/after values and reason

- GIVEN a TaxDeclaration filed on 2026-04-30
- WHEN authorities request audit trail
- THEN system exports:
  - All TaxableTransaction included in the return
  - Calculation methods (INCLUSIVE/EXCLUSIVE/COMPOUND per line)
  - Filing timestamp, person responsible, final state
  - Immutable: any changes post-filing are logged as AMENDMENTS, not overwrites

- GIVEN a change to a TaxCode rate (effective 2026-06-01)
- WHEN past transactions used that code
- THEN the audit shows:
  - Old code version + old rate for past transactions
  - New code version + new rate applied to future transactions
  - Clear separation (not recalculated retroactively unless explicitly amended)
