---
title: Tax & Levy Management — Shillinq — Other T1
app: shillinq
kind: config
depends_on: []
change: tax-levy-management-other-t1
created: 2026-05-21
---

# Proposal: Tax & Levy Management — Shillinq — Other T1

## Executive Summary

Shillinq is a complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations on Nextcloud. This spec focuses on tax and levy management—the highest-demand cluster (1,775 market mentions) covering workflow automation, compliance reporting, and multi-jurisdiction support.

The 50 features in this scope span:
- Tax compliance workflows (VAT, income tax, ESG reporting)
- Automated reporting (BCF, IV3, BTW-aangifte, Peppol)
- Tax rate and exemption management
- Real-time tax liability and capital gains tracking
- Receipt and invoice OCR with tax line-item extraction

## Market Demand

| Feature | Demand | Competitor Coverage | Category |
|---------|--------|-------------------|----------|
| Tax Workflow Management | 1,775 | 12% | primary |
| XBRL Taxonomy Management | 1,674 | 41% | support |
| Exemption Certificate Management | 1,674 | 41% | support |
| Global tax management | 1,606 | 5% | primary |
| Submit BCF declaration | 1,350 | 0% | compliance |
| Submit VAT electronically | 1,350 | 0% | compliance |
| Submit IV3 to CBS | 1,350 | 0% | compliance |
| Publish adopted tax rates | 348 | 0% | configuration |
| Tax config with line-item breakdown | 341 | 5% | configuration |

*Full feature list (50 total) in `context-brief.md`.*

## Stakeholders

### Role: Tax Compliance Officer
**Goal:** Ensure timely, accurate tax filings and minimize compliance risk.
**Context:** Manages VAT returns, income tax estimates, ESG disclosures; reconciles bank statements to tax liabilities.
**Pain Point:** Manual extraction of tax line items from invoices; error-prone aggregation for multi-jurisdictional filings.

### Role: Finance Manager
**Goal:** Automate tax calculations and reporting; reduce manual effort.
**Context:** Oversees bookkeeping, prepares year-end tax packages, reconciles with accounting records.
**Pain Point:** No real-time visibility into annual income tax (IB) liability; tax rate changes require manual update across line items.

### Role: Bookkeeper
**Goal:** Track deductible expenses and tax categories with minimal re-entry.
**Context:** Records purchase invoices, categorizes expenses, flags tax-deductible items.
**Pain Point:** Receipt scanning is manual; tax rates per item are not standardized; no OCR auto-extraction.

### Role: Freelancer / Sole Proprietor
**Goal:** File taxes efficiently without external accountant; understand tax obligation in real-time.
**Context:** Works part-time on compliance; uses Shillinq for invoicing + expense tracking.
**Pain Point:** Year-end scramble to collect receipts; no estimate of annual tax liability until filing season.

## User Stories

### Story 1: Estimate Annual Tax Liability
```
GIVEN a freelancer with invoices and expenses in a fiscal year
WHEN they open the Tax Liability Dashboard
THEN they see a real-time estimate of their annual income tax (IB) liability
  AND the estimate updates when new transactions are recorded
  AND they can export the estimate as a PDF for their accountant
```

### Story 2: Submit VAT Return Electronically
```
GIVEN a VAT-liable business with sales and purchase invoices
WHEN they initiate a VAT filing for a quarter
THEN the system auto-calculates VAT owed from transactions
  AND they can review the draft BTW-aangifte before submission
  AND they can file directly to Belastingdienst with a single click
  AND filing status is tracked in real-time
```

### Story 3: Scan & Recognize Receipt Tax Items
```
GIVEN a PDF receipt for a business expense
WHEN they upload the receipt to Shillinq
THEN the system extracts merchant, date, amount, VAT, and line items
  AND the OCR tags the transaction with tax category
  AND the expense is automatically categorized
```

### Story 4: Manage Tax Rates per Item
```
GIVEN a business selling taxable and tax-exempt items
WHEN they configure product/service tax rates
THEN they can set multiple tax rates for a single item
  AND the system validates rates against jurisdiction rules (KOR, reverse charge, etc.)
  AND rates auto-apply to new invoices
```

### Story 5: File IV3 to CBS (Dutch Statistics)
```
GIVEN an eligible business
WHEN they prepare their IV3 filing
THEN the system auto-populates IV3 fields from transaction data
  AND they can run consistency checks before submission
  AND they can file directly to CBS with compliance confirmation
```

### Story 6: Review VAT Return Before Filing
```
GIVEN a completed VAT return draft
WHEN they open the Review screen
THEN they see line-by-line VAT calculations
  AND they can drill into transactions backing each line
  AND they can adjust entries before final submission
```

### Story 7: ESG Reporting Under CSRD
```
GIVEN a corporation with sustainability data
WHEN they prepare ESG reporting
THEN the system maps transactions to ESRS taxonomy
  AND they can generate CSRD-compliant ESG reports
  AND reports reference supporting transaction data
```

### Story 8: Register Tax Objection (Bezwaar)
```
GIVEN a disagreement with a tax assessment
WHEN they initiate a formal objection
THEN the system guides them through bezwaar filing steps
  AND supporting documents are linked to the objection record
  AND filing deadline and status are tracked
```

## What's In Scope (This Spec)

✅ Tax data schemas and OpenRegister configuration
✅ Tax workflow definitions (state machine for filing, objection, assessment)
✅ Compliance report templates (VAT, IV3, BCF, ESG)
✅ Integration hooks for external tax engines (AvaTax, Vertex, Peppol)
✅ Tax rate and exemption certificate management
✅ Real-time tax liability calculations
✅ Receipt OCR integration (Scan & Herken)

## What's Out of Scope

❌ Full implementation of external APIs (AvaTax, Belastingdienst real-time filing) — handled in separate integration specs
❌ Payroll tax calculations — covered in a separate payroll spec
❌ Jurisdiction-specific business rules beyond Dutch BBV/IV3/DigiInkoop — addressed in future regional specs
❌ Document generation (e.g., PDF tax packages) — in a separate document-generation spec

## Success Criteria

1. Tax-compliant workflows modeled in OpenRegister lifecycle
2. Real-time tax liability estimate available on dashboard
3. VAT/IV3/BCF filing preparation automated (80%+ pre-filled)
4. Receipt OCR extraction tested with 5 sample Dutch invoices
5. Tax rate management prevents invalid jurisdiction-rule combinations
6. Audit trail records every tax calculation and filing step

## Timeline & Dependencies

- **Predecessor:** None (greenfield tax module)
- **Successor:** Integration specs (Belastingdienst, AvaTax, Peppol)
- **Estimated effort:** 3-4 sprints (config + code)
- **Kind:** config (schema register patches + integration hooks, minimal custom service code)

## References

- ADR-031: Schema-declarative business logic (lifecycles, aggregations, calculations)
- ADR-032: Spec sizing (this is config-only; no code/PHP in scope)
- ADR-011: Schema standards (schema.org vocabulary for tax entities)
- Shillinq feature brief: `/openspec/changes/tax-levy-management-other-t1/context-brief.md`
