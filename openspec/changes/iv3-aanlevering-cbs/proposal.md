---
status: ready
demandScore: critical
priority: P1
theme: compliance
---

# Proposal: Iv3-Aanlevering CBS

## Problem Statement

Dutch decentrale overheden (gemeenten, provincies, waterschappen, gemeenschappelijke regelingen) are statutorily required by Wet Fido art. 3 lid 4 and the Regeling informatie voor derden (RIV) to deliver quarterly financial reports (Iv3 — Informatie voor derden) to the Centraal Bureau voor de Statistiek. These aanleveringen feed critical national statistical databases (statline), European reporting obligations (EMU-saldi, EDP), and macro-economic forecasts (Houdbaarheidsraming, CPB).

The current process is entirely manual and spreadsheet-driven:
- Most organisations maintain tussenrekening / categorieverdeling spreadsheets that manually map their grootboek-rekeningen to the standardised Iv3-taakvelden
- These mappings drift quarter to quarter and are frequently hand-edited days before the deadline
- The XML serialisation to EDA-XML format is error-prone and often requires rework
- Pre-submission validation is incomplete; many organisations discover errors only after CBS-Kredo rejects their payload
- Late or non-conforming aanleveringen trigger escalations from BZK financieel toezicht and contribute to artikel 12 status risk

The risk is acute: missed deadlines and validation failures are compliance violations with direct consequences for municipal oversight rankings, state budget scrutiny, and European reporting accuracy.

## Scope

The `iv3-aanlevering-cbs` spec owns the complete Iv3 lifecycle from mapping definition through submission and correction:

1. **Taxonomy ingestion** — annual CBS publication of taakvelden, economische categorieën, balansposten, and gemeentecodes
2. **Mapping registers** — canonical, versioned mapping from each organisation's grootboek-rekeningen to Iv3-elements with support for full, percentage, and driver-based splits
3. **Aggregation engine** — per-period computation of ventilated amounts (baten, lasten, mutaties reserves, saldi)
4. **EDA-XML serialisation** — generation of XML conformant to CBS-published XSD schemas
5. **Local validation** — pre-run of the full CBS-Kredo validation suite (including structural, numeric, and cross-foot checks)
6. **Submission tracking** — POST to CBS-Kredo, response capture, and status lifecycle management
7. **Correctie-aanleveringen** — support for corrections against prior submitted payloads with chain maintenance
8. **Audit trail** — append-only capture of every state transition, validation, and submission action
9. **Readiness dashboard** — pre-deadline visibility and escalation for blockers

The spec does NOT own:
- The underlying grootboek (owned by `bookkeeping-bbv-compliance`)
- Budget/forecast figures (owned by `bookkeeping-budget-forecast`)
- Macro-aggregation for country-level EMU-saldo (owned by CBS)
- Deduplication of gemeenschappelijke regelingen (responsibility of the individual GR)

## Value Proposition

**Compliance & Risk Reduction**
- Eliminates spreadsheet drift and manual error as sources of non-conformity
- Pre-deadline validation catches CBS-Kredo rejections locally, reducing deadline-day crises
- Audit trail provides defensible evidence of due diligence for accountant review and toezichthouder inquiries

**Operational Efficiency**
- One-time mapping setup, automatically versioned per boekjaar
- Quarterly aggregations are fully automated and deterministic
- Reduces Iv3-coordinator effort from days to hours per quarter

**Data Quality & Governance**
- Canonical taxonomy prevents outdated reference data from polluting submissions
- Immutable payload storage and response capture enables long-term audit and archival
- Version control of mappings and payload traces enable root-cause analysis of corrections

## Target Users

- **Concerncontroller / hoofd Financiën** — signs off on submissions, monitors deadline readiness, escalates blockers
- **Iv3-coordinator** — maintains mappings, runs aggregations, produces payloads
- **Afdelings-controllers** — reviews their programma's contribution pre-signature
- **Accountant** — uses Y-aanlevering reconciliation for jaarrekening-controle, reviews audit log
- **BZK financieel toezicht** — consumes aanlevering status feed for compliance monitoring
- **CBS / Kredo-team** — receives aanleveringen, returns acknowledgements/rejections

## Success Criteria

1. **First aanlevering** — Q1-2027 aanlevering produced and successfully submitted to CBS-Kredo within 10 working days of Q1 close (target: 2027-04-20)
2. **Zero manual XML editing** — 100% of EDA-XML generated via the spec; zero hand-edited payloads post-generation
3. **Pre-deadline validation catch rate** — local validation catches ≥95% of CBS-Kredo rejection reasons before submission
4. **Reduced Iv3-coordinator time** — per-period effort ≤4 hours (mapping maintenance excluded)
5. **Audit defensibility** — 100% of state transitions captured in immutable audit log; able to reconstruct reason and actor for every submission
6. **Cross-organisation rollout** — 10+ Dutch municipalities/provinces using the spec within 18 months of initial release

## Stakeholders & Responsibilities

- **Specter (Iv3-coordinator)** — context capture, mapping definition, aggregation design, XML schema binding, CBS-Kredo API integration
- **VNG / IPO / UvW** — stakeholder consultation on sector-wide usability, contribution to yearly taxonomy load process
- **CBS / Kredo-team** — API endpoint documentation, XSD schemas, validation rule documentation
- **BZK financieel toezicht** — integration endpoint for aanlevering status notifications
- **Accountant / audit community** — review of Y-aanlevering reconciliation logic and audit trail design

---

## Appendix: Feature Overview

| Feature | Demand | Description |
|---------|--------|-------------|
| Taxonomy Ingestion | MUST | Annual CBS taxonomy load with version tracking and herindeling handling |
| GrootboekMapping | MUST | Per-org, per-boekjaar mapping with full/percentage/driver split methods |
| Aggregation Engine | MUST | Per-period ventilation: baten, lasten, mutaties, saldi |
| EDA-XML Serialisation | MUST | CBS-XSD conformant XML generation per boekjaar schema version |
| Local Validation | MUST | Pre-run of CBS-Kredo checks; block submission on blockers |
| Cross-foot Checks | MUST | Balanstotaal sluitend; baten-lasten-saldi-consistent |
| Quarterly Cumulatief | MUST | Q2 ⊇ Q1, Q3 ⊇ Q1+Q2, Q4+Y reconciliation |
| CBS Submission & Response | MUST | Submit to Kredo, capture responses, update aanlevering status |
| Correctie-aanleveringen | MUST | Support corrections with chain maintenance |
| Audit Trail | MUST | Immutable event log for all state transitions and actions |
| Readiness Dashboard | MUST | Pre-deadline visibility and escalation widget |
| Y-Aanlevering Reconciliation | SHOULD | Match vastgestelde jaarrekening within tolerance |
| Driver-Based Splits | SHOULD | Integration with `driver-based-forecasting` for allocation logic |
| Notification Service | SHOULD | Pre-deadline alerts and escalations via n8n |
