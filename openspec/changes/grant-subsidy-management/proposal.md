---
kind: code
depends_on: []
chain:
  - grant-subsidy-management
---

# Proposal: Grant & Subsidy Management — Shillinq

## Why

Dutch municipalities, provinces, and public-sector organisations manage hundreds of subsidy schemes annually, disbursing public funds under strict accountability obligations (BBV, SiSa, IV3). Today these organisations track grants and applications in spreadsheets or disconnected systems, resulting in missed accountability deadlines, overpayments that are never recovered, and integrity failures that only surface at year-end audit. Market demand analysis identifies **grant and fund management (demand: 1675)** and **governmental accounting standards and grant management (demand: 1593)** as the two highest-demand unserved features in the Shillinq product, covering 530 tender mentions with only 1–41% competitor coverage.

This change adds a complete subsidy lifecycle to Shillinq: from scheme publication and application intake, through eligibility assessment, award decision, advance payments, and accountability review, to dossier closure and year-end reporting. It also covers integrity safeguards (sanctions checking, fraud investigation, risk scoring) required by Dutch public procurement law.

## What Changes

- New **SubsidyScheme** entity: configure funding programmes with budget, eligibility rules, application periods, and portal publication
- New **SubsidyApplication** entity: applicants submit requests with supporting documents; case handlers assess and decide
- New **Grant** entity: tracks awarded amounts, payment milestones, accountability deadlines, SiSa eligibility, and dossier status
- New **GrantPortfolio** entity: aggregate view for concentration-risk analysis and compliance monitoring per organisation
- New **AuditorStatement** entity: register and verify accountantsverklaringen for large subsidies above statutory thresholds
- Background job for accountability deadline reminders (30-day and 7-day warnings, automated on project end date)
- Integrity services: sanctions/debarment list check on application submission, risk scoring for spot-check prioritisation, fraud investigation case creation
- Financial safeguards: block final payment when mandatory conditions are unmet, auto-flag overpayments after accountability review
- Year-end controller views: open commitments report, financial year lock
- Subsidy portal: public scheme browser with category and target-group filters

## Capabilities

### New Capabilities

- `SubsidySchemeService`: create, draft, approve, and publish subsidy schemes with eligibility rules and budget parameters
- `SubsidyApplicationService`: submit applications with documents, run integrity checks, generate decision letters from templates
- `GrantService`: award grants, track disbursements, manage accountability lifecycle, detect overpayments, close and archive dossiers
- `GrantPortfolioService`: aggregate grant portfolios, compute concentration risk, report compliance status
- `AuditorStatementService`: register auditor statements, verify statements for large subsidies, link to grant records
- `SanctionsCheckJob`: on application submission, check applicant against EU Sanctions, OFAC, and Rijksoverheid exclusion register
- `AccountabilityReminderJob`: scheduled job that sends 30-day and 7-day deadline reminders to subsidy recipients
- `RiskScoringService`: compute automated risk score per application for integrity officer spot-check dashboard
- `FraudInvestigationService`: open confidential investigation cases, apply payment holds, restrict access to investigation team
- Public portal API: list published schemes with category/target-group filtering (`#[PublicPage]`)
- Year-end lock: controller locks financial year after reconciliation sign-off; locked records become read-only
- Decision letter generation: produce pre-filled PDF decision letters from configurable templates (docudesk integration)

### Modified Capabilities

- None — this is a new feature module within Shillinq. Existing bookkeeping and invoicing modules are not changed.

## Stakeholders

| Role | Responsibility |
|------|---------------|
| Grant Administrator | Configures subsidy schemes, publishes to portal, sends accountability requests, generates decision letters |
| Financial Administrator | Monitors advance payments, blocks payments on unmet conditions, identifies overpayments, closes dossiers |
| Integrity Officer | Reviews risk-scored application lists, checks sanctions, opens fraud investigation cases |
| Subsidy Case Handler | Processes applications, assesses eligibility, generates and finalises decision letters |
| Municipal Controller | Locks financial year, runs open commitments report for jaarrekening |
| Subsidy Team Lead | Monitors decision outcomes per scheme, exports application data |
| Subsidy Applicant (Organisation) | Browses schemes, submits applications, completes accountability reports online |
| L&D Coordinator | Tags training activities as subsidy-funded (SLIM/ESF), generates subsidy reports |
| Subsidieverlener | Monitors how grantees spend their funding |

## Entities (from ADR-000)

All entities are OpenRegister schemas. No custom Entity/Mapper classes.

| Entity | schema.org type | Purpose |
|--------|----------------|---------|
| `SubsidyScheme` | `schema:GovernmentService` | Funding programme definition, eligibility rules, portal publication |
| `SubsidyApplication` | `schema:Application` | Application for a scheme with documents and review state |
| `Grant` | `schema:Grant` | Awarded subsidy with disbursement tracking and accountability lifecycle |
| `GrantPortfolio` | `schema:Collection` | Managed collection of grants for compliance and risk monitoring |
| `AuditorStatement` | `schema:Statement` | Auditor verification statement for large subsidies |

Cross-references to existing Shillinq entities (do NOT redefine): `Organization`, `Person`, `Document`, `DigitalDocument`, `AccountabilityReport`, `Payment`.

## Impact

- `lib/Settings/shillinq_register.json`: add 5 new schemas + seed data
- `lib/Service/SubsidySchemeService.php`: scheme lifecycle, portal publication
- `lib/Service/SubsidyApplicationService.php`: application intake, integrity checks, letter generation
- `lib/Service/GrantService.php`: award, disbursement, accountability, overpayment, dossier closure
- `lib/Service/GrantPortfolioService.php`: portfolio aggregation, concentration risk
- `lib/Service/AuditorStatementService.php`: statement registration and verification
- `lib/Service/RiskScoringService.php`: automated risk scoring
- `lib/Service/FraudInvestigationService.php`: fraud case management
- `lib/Service/SanctionsCheckService.php`: external API integration for EU Sanctions / OFAC / Rijksoverheid
- `lib/BackgroundJob/AccountabilityReminderJob.php`: scheduled deadline notifications
- `lib/Controller/SubsidyPortalController.php`: public portal API (#[PublicPage])
- `lib/Controller/SubsidySchemeController.php`, `SubsidyApplicationController.php`, `GrantController.php`, `GrantPortfolioController.php`, `AuditorStatementController.php`
- `src/views/`: Dashboard, SubsidySchemes, SubsidyApplications, Grants, GrantPortfolios, AuditorStatements pages
- `src/store/modules/`: Pinia stores for all 5 entity types
- `l10n/nl.json`, `l10n/en.json`: translations for all new strings
- `tests/Unit/`: PHPUnit tests for all new services
- `tests/integration/`: Newman collection for all new endpoints
