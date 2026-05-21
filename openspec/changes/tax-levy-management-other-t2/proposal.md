# Proposal: Tax & Levy Management — Shillinq — Other T2

**Status:** Proposed  
**Date:** 2026-05-21  
**Change ID:** tax-levy-management-other-t2  
**Platform:** Shillinq on Nextcloud + OpenRegister  

## Executive Summary

Shillinq's Tax & Levy Management suite adds comprehensive tax compliance, automated tax calculation, multi-jurisdiction support, and electronic filing capabilities for freelancers, SMBs, and corporations. This change (T2: secondary features) covers VAT automation, tax exemptions, compliance reporting, multi-state/multi-jurisdiction support, and integration-ready tax determination logic.

**Demand:** 118 total points across 50 features (demand range 2–7)  
**Category:** Tax & Compliance Infrastructure  
**Scope:** Core tax calculation, VAT detection, exemption handling, multi-jurisdiction support, and tax reporting

## Problem Statement

Dutch and international businesses face:
- **Complexity:** Multiple tax jurisdictions (NL VAT, international VAT, wage tax, BTW)
- **Manual burden:** Invoice-by-invoice VAT classification, rate selection, return preparation
- **Compliance risk:** Missed exemptions (KOR, B2B, cross-border), incorrect categorization, filing errors
- **Integration fragmentation:** Standalone tax software, external filing services, no unified workflow
- **Time-to-compliance:** Days of reconciliation before each return filing

Shillinq aims to **eliminate manual tax work** through:
- Automatic tax determination from transaction context
- Built-in VAT rules and exemption logic
- Multi-jurisdiction support (NL, EU, international)
- Electronic filing preparation (Belastingdienst, e-filing, etc.)
- Real-time tax position visibility

## Success Criteria

1. **Auto-detection:** VAT classification and rate from invoice context ≥80% accuracy
2. **Exemption handling:** KOR, B2B, cross-border rules applied without manual override
3. **Multi-jurisdiction:** ≥5 jurisdictions supported (NL VAT, BTW, wage tax, international VAT)
4. **Return filing:** Automated btw-aangifte or equivalent ≥90% pre-filled
5. **Compliance:** Zero manual entry errors in automated fields; audit trail 100%

## Features (Sorted by Demand)

| Demand | Feature | Rationale |
|--------|---------|-----------|
| 7 | Multi-state Return Filing | 1 tender, 2% competitor coverage |
| 6 | VAT Auto-detection | Context-aware VAT classification |
| 6 | Configurable Tax Rates (Compound, Incl/Excl) | Flexible rate configuration |
| 6 | Tax Reports | Compliance reporting and analytics |
| 6 | Indirect Tax Determination | Rooftop-level tax logic |
| 6 | VAT Detection & Recovery | Inbound/outbound VAT tracking |
| 6 | Payroll Tax Calculation & Filing | Wage tax (loonheffing) automation |
| 5 | Auto-Classification (Country/Language/VAT/Currency) | ML-driven invoice classification |
| 5 | Tax Registration Service | KvK/BTW lookup and validation |
| 5 | International Tax Modules | Multi-country tax support |
| 5 | Multi-state Tax Allocation | Geographic tax distribution |
| 5 | BTW-aangifte Automation | Dutch VAT return automation |
| 4 | Multiple Tax Rates | Tiered rate support |
| 4 | Tax Provision Calculation | Estimated tax liability |
| 4 | Multi-jurisdiction VAT | Cross-border VAT rules |
| 4 | Wage Tax Filing (Belastingdienst) | Direct wage tax submission |
| 4 | Auto-filled BTW-aangifte | Pre-populated return forms |
| 4 | 300M+ Tax Rules Database | Comprehensive rule coverage |
| 4 | Electronic Tax Return Filing | Submission to authorities |
| 4 | Sales Tax Engine Integration | CCH SureTax or equivalent |
| 4 | Automatic BTW Calculation | Return line auto-population |
| 2 | KOR Exemption (Kleineondernemersregeling) | Small business VAT exemption |
| 2 | Multiple VAT Rates (High/Low/Reverse) | Tiered rate handling |
| (Other 27 features at demand 2–4) | ... | See context-brief.md for full list |

## Implementation Phases

**Phase 1 (This change — T2):** Core infrastructure
- TaxRate management (compound, inclusive/exclusive)
- VAT Detection & Recovery (inbound/outbound)
- Tax Exemption rules (KOR, B2B, reverse charge)
- Auto-detection pipeline (country, language, VAT status)
- Multi-jurisdiction support framework

**Phase 2 (T1 — pending):** Automated filing
- BTW-aangifte generation (NL)
- Belastingdienst integration
- Electronic return submission
- Wage tax (loonheffing) filing

**Phase 3 (Future):** Advanced analytics
- Tax provision forecasting
- Multi-state allocation
- AI-powered exception alerts

## Stakeholders

| Role | Responsibility | Goals |
|------|-----------------|-------|
| **CFO / Accountant** | Tax strategy, compliance certification | 100% compliant returns, zero penalties, audit readiness |
| **Bookkeeper** | Daily transaction entry, invoice processing | VAT auto-detection saves 2–4 hrs/week; exempt transactions flagged |
| **Freelancer/SMB Owner** | Full administration (invoice + tax) | Self-serve VAT returns, no accountant needed for routine filings |
| **Tax Consultant** | External tax planning, return review | Access to transaction detail; override capability; audit logs |
| **Compliance Officer** | Regulatory adherence, policy enforcement | Exemption rules enforced at data entry; no manual approval needed |
| **System Admin** | Config, user access, integrations | Multi-tenancy, permission control, seamless OR integration |

## Out of Scope

- Payroll module (separate app planned)
- Income tax calculations (1040, CT600)
- Automatic filing to external services (manual export → service)
- Custom rule creation by end-users (rules via API/config only)
- Historical data migration from legacy systems

## Dependencies

- **OpenRegister:** TaxRate, TaxConfiguration, TaxableTransaction, TaxReturn entities
- **Nextcloud:** User authentication, multi-tenancy, notification framework
- **External APIs:** OpenRegister (data layer), KvK/Belastingdienst validation (future)

## Next Steps

1. Finalize design (data models, VAT rules engine, exemption logic)
2. Create detailed specifications (GIVEN/WHEN/THEN scenarios)
3. Break into implementation tasks (schema, API, UI, tests)
4. Estimate story points and schedule

---

**Change:** `tax-levy-management-other-t2`  
**Branch:** `spec/tax-levy-management-other-t2`  
**Author:** Specter Intelligence  
**Approvals:** Pending
