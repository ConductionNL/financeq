# Proposal: Tax & Levy Management — Shillinq

**Change ID:** tax-levy-management  
**Created:** 2026-05-21  
**Platform:** Nextcloud + OpenRegister  
**Stakeholders:** Tax Director, CFO, Payroll Administrator, Municipal Tax Administrator, Compliance Officer

## Executive Summary

Tax & Levy Management adds comprehensive tax compliance, reporting, and filing capabilities to Shillinq for Dutch freelancers, SMBs, municipalities, and larger organizations. This includes VAT return preparation with MTD compliance, multi-jurisdiction tax tracking, automated tax categorization, and standardized reporting (XBRL/SBR) for Dutch tax authorities.

### Market Demand & Positioning

| Feature | Demand | Coverage | Priority |
|---------|--------|----------|----------|
| VAT return preparation with MTD (Making Tax Digital) compliance for UK | 665 | 1% competitor | must-have |
| Approve or reject amendment with motivation | 651 | — | must-have |
| Income statement with quarterly breakdowns for tax planning | 200 | 1% competitor | must-have |
| Tax report generation based on tagged transactions | 173 | 45% competitor | should-have |
| VAT report overview showing collected and paid VAT by period | 172 | 5% competitor | should-have |
| TDS (Tax Deducted at Source) tracking and reporting | 171 | 12% competitor | should-have |
| Automated tax compliance capturing W-9/W-8 and validating TINs for 1099 reporting | 161 | 4% competitor | nice-to-have |
| Tax summary report for filing preparation by period | 158 | 34% competitor | should-have |
| Quarterly tax report for estimated tax payment preparation | 133 | 26% competitor | should-have |
| Sales tax tracking with configurable rates | 115 | 5% competitor | should-have |

## Epics & User Stories (11 core stories)

### Epic 1: Multi-Jurisdiction VAT & Tax Returns
- **Story 2:** Multi-jurisdiction VAT — VAT returns generated per jurisdiction for all obligations
- **Story 3:** Separate chart of accounts per administration — each with own VAT settings and fiscal year
- **Story 4:** Maintain VAT audit trail — complete audit trail for all VAT transactions

### Epic 2: Annual Tax Statements & Corporate Returns
- **Story 5:** View annual income statement (jaaropgave) — employees access tax statements for annual tax returns
- **Story 6:** Prepare corporate tax return (VPB-aangifte) — accountants prepare and file corporate tax with SBR export

### Epic 3: Tax Rate Management & Council Governance
- **Story 9:** Review tax rate proposal before council vote — visibility into proposed rates with comparisons
- **Story 10:** Publish adopted tax rates — lock rates for new fiscal year and update all tax modules

### Epic 4: Payroll Tax Compliance
- **Story 11:** Submit monthly wage declaration (loonaangifte) — generate and file payroll tax returns to Belastingdienst
- **Story 1:** Fiscal partner allocation (nice-to-have) — allocate items to fiscal partner for joint tax optimization

### Epic 5: Tax Exemptions & Objections (Municipal)
- **Story 7:** Assess objection and revise WOZ value — municipal tax administrators manage tax objections
- **Story 8:** Prepare income tax return (IB-aangifte) for sole proprietor — freelancers file personal tax returns with deductions

## Data Model (11 Entities)

All entities implement OpenRegister schemas. See design.md for schemas and seed data.

1. **ExemptionCertificate** — Tax exemption credentials (research, export, environmental)
2. **TaxConfiguration** — System-wide tax settings per jurisdiction and year
3. **TaxDeclaration** — Primary tax declaration (VAT, BCF, ICP) with submission workflow
4. **TaxExemption** — Reusable exemption rules applied to transactions
5. **TaxLot** — Individual tax line items within declarations
6. **TaxRate** — Tax rate rules with effective date management
7. **TaxReturn** — Formal tax return filings with workflow management
8. **TaxableTransaction** — Business transactions classified and tracked for reporting
9. **VATReturn** — VAT-specific returns showing collected/paid VAT and MTD compliance
10. **XBRLInstance** — Structured XBRL instance documents for Dutch authority filing
11. **XBRLTaxonomy** — XBRL taxonomy definitions (NTA7, SBR-NT) for reporting

## Success Criteria

- [ ] VAT returns can be generated per jurisdiction and period
- [ ] Multi-administration support with separate COA, VAT settings, fiscal years
- [ ] Tax exemption certificates and exemptions apply correctly to transaction categorization
- [ ] XBRL/SBR export for Dutch tax authority submission (NTA7, SBR-NT)
- [ ] Audit trail captures all VAT/tax transaction changes for defense during audits
- [ ] Annual statements available for employee download in Belastingdienst-required format
- [ ] Payroll tax declarations (loonaangifte) can be submitted to Belastingdienst
- [ ] Tax rate management with council voting workflow for municipal administrations

## Related Specs

- `openspec/architecture/adr-000-data-model.md` — Data model patterns
- `openspec/architecture/adr-001-data-layer.md` — OpenRegister implementation
- `openspec/architecture/adr-010-nl-design.md` — NL Design System compliance for tax reporting UI
- `openspec/architecture/adr-011-schema-standards.md` — schema.org vocabulary for tax entities

## Open Questions

1. Which XBRL taxonomies are in scope for the first release? (NTA7 only, or include SBR-NT?)
2. Should tax rate proposals support draft versions before council approval?
3. How are multi-country rates handled (e.g., cross-border transactions)?
4. Who can amend filed declarations, and what triggers the "approve/reject amendment" workflow?

## Notes

- Shillinq is a complete open-source business administration suite covering bookkeeping, invoicing, procurement, and contract management.
- Named after the shilling — one of the oldest coins in European history.
- Platform: Nextcloud self-hosted with OpenRegister for data management.
- Compliance focus: Dutch government standards (BBV, IV3, SiSa, DigiInkoop), VAT/GST/sales tax globally.
