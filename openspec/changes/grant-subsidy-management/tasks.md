# Tasks: Grant & Subsidy Management — Shillinq

## Deduplication Check

- [ ] DUP-1 Verify no overlap with `ObjectService` for CRUD operations — use `ObjectService::saveObject($register, $schema, $object)` and `findObject($register, $schema, $id)` exclusively; no custom CRUD
- [ ] DUP-2 Confirm `FileService` + `CnObjectSidebar` handles all document attachments (supporting documents for SubsidyApplication, AuditorStatement upload) — no custom file upload controller
- [ ] DUP-3 Confirm `AuditTrailService` handles all change tracking — no custom audit logging in any service
- [ ] DUP-4 Confirm `NotificationService` handles all email/push notifications — custom code only for trigger scheduling and payload construction
- [ ] DUP-5 Confirm `ObjectService::lockObject()` handles financial year lock — no custom lock mechanism
- [ ] DUP-6 Confirm `TasksController` handles recovery tasks and accountability tasks — no custom task system
- [ ] DUP-7 Confirm `CnFormDialog` auto-generates all create/edit forms from schema — no custom form components
- [ ] DUP-8 Confirm `CnDashboardPage` + `CnStatsBlock` + `CnChartWidget` handles all dashboard widgets — no custom dashboard layout
- [ ] DUP-9 Confirm `CnMassExportDialog` handles CSV export — no custom export controller
- [ ] DUP-10 Confirm docudesk integration handles decision letter PDF generation — no separate PDF library

---

## 1. Schema & Register Configuration

- [ ] 1.1 Add `SubsidyScheme` schema definition to `lib/Settings/shillinq_register.json` with all properties from ADR-000: `schemeId`, `name`, `description`, `maxGrant`, `minGrant`, `isPublished`, `publishedDate`, `governmentLevel`, plus extended metadata fields (`legalBasis`, `totalBudget`, `targetGroup`, `applicationPeriodStart`, `applicationPeriodEnd`, `requiredDocuments`, `status`, `category`)
- [ ] 1.2 Add `SubsidyApplication` schema to register JSON: `applicationId`, `requestedAmount`, `status` (enum: draft, submitted, under-review, sanctions-hold, approved, rejected, withdrawn), `submissionDate`, `reviewDate`, `notes`, `sanctionsCheckResult`, `decisionLetterRef`, `applicantContactPerson`
- [ ] 1.3 Add `Grant` schema to register JSON: `grantId`, `name`, `awardedAmount`, `awardDate`, `status` (enum: active, completed, suspended, revoked, overpaid, closed), `accountingStandard`, `isSISAEligible`, `totalDisbursed`, `accountabilityDeadline`, `projectEndDate`, `finalPaymentDate`, `closureTimestamp`, `recoveryAmount`, `paymentHold`, `advancePaymentAmount`, `advancePaymentDueDate`
- [ ] 1.4 Add `GrantPortfolio` schema to register JSON: `portfolioId`, `name`, `description`, `totalGrantValue`, `complianceStatus` (enum: compliant, non-compliant, under-review), `concentrationRiskLevel` (enum: low, medium, high), `lastAuditDate`
- [ ] 1.5 Add `AuditorStatement` schema to register JSON: `statementId`, `verificationDate`, `isVerified`, `findings`, `verdict` (enum: approved, rejected, conditional), `auditorRegistrationNumber`, `opinionType`, `statementDate`
- [ ] 1.6 Add 5 seed objects for each schema (25 total) per design.md Seed Data section using `@self` envelope (`register`, `schema`, `slug`) with Dutch values — municipality names, real Dutch postcodes, realistic amounts
- [ ] 1.7 Add OpenRegister relations in schema definitions: SubsidyScheme → Organization, SubsidyApplication → SubsidyScheme + Organization, Grant → SubsidyScheme + Organization + GrantPortfolio, GrantPortfolio → Organization, AuditorStatement → Grant + Person + DigitalDocument
- [ ] 1.8 Create repair step `lib/Migration/RepairStep/RegisterSubsidySchemas.php` implementing `IRepairStep` that calls `ConfigurationService::importFromApp('shillinq', $data, $version, false)` — idempotent
- [ ] 1.9 Register repair step in `appinfo/info.xml` under `<repair-steps><post-migration>`

---

## 2. Backend Services

### 2.1 SubsidySchemeService

- [ ] 2.1.1 Create `lib/Service/SubsidySchemeService.php` with `@spec` tag linking to this tasks.md
- [ ] 2.1.2 Implement `createScheme(array $data): array` — validates required fields, sets status to `draft`, saves via `ObjectService::saveObject()`
- [ ] 2.1.3 Implement `publishScheme(string $schemeId): array` — validates status is `approved`, sets `isPublished: true`, `publishedDate` to now, status to `published`
- [ ] 2.1.4 Implement `closeScheme(string $schemeId): array` — sets status to `closed`, deactivates application form
- [ ] 2.1.5 Implement `getPublishedSchemes(array $filters): array` — returns only `isPublished: true` records; used by public portal; filters by `category`, `targetGroup`, `applicationPeriodEnd > now`

### 2.2 SubsidyApplicationService

- [ ] 2.2.1 Create `lib/Service/SubsidyApplicationService.php` with `@spec` tag
- [ ] 2.2.2 Implement `submitApplication(array $data): array` — sets status to `submitted`, records `submissionDate`, dispatches `SanctionsCheckJob` asynchronously
- [ ] 2.2.3 Implement `approveApplication(string $applicationId, float $awardedAmount): array` — sets status to `approved`, calls `GrantService::createGrant()` with linked application data
- [ ] 2.2.4 Implement `rejectApplication(string $applicationId, string $reason): array` — sets status to `rejected`, records `reviewDate`
- [ ] 2.2.5 Implement `generateDecisionLetter(string $applicationId, string $outcome): string` — calls docudesk template engine with pre-filled applicant name, address, scheme name, decision, amount, legal basis; returns document reference
- [ ] 2.2.6 Implement `recordEligibilityAssessment(string $applicationId, array $criteria): array` — records per-criterion met/not-met results and overall assessment on application record

### 2.3 GrantService

- [ ] 2.3.1 Create `lib/Service/GrantService.php` with `@spec` tag
- [ ] 2.3.2 Implement `createGrant(array $data): array` — creates Grant record linked to SubsidyApplication and SubsidyScheme; inherits `isSISAEligible` from scheme
- [ ] 2.3.3 Implement `processPayment(string $grantId, array $paymentData): array` — checks all mandatory conditions before processing; if any condition unmet, returns 422 with structured checklist of outstanding items; blocks payment; if all met, records disbursement and updates `totalDisbursed`
- [ ] 2.3.4 Implement `checkMandatoryConditions(string $grantId): array` — returns list of conditions with met/unmet status: accountability report approved, declarations signed, auditor statement verified (if threshold exceeded)
- [ ] 2.3.5 Implement `closeDossier(string $grantId): array` — sets status to `closed`, records `closureTimestamp`, locks record via `ObjectService::lockObject()`
- [ ] 2.3.6 Implement `detectOverpayment(string $grantId, float $approvedFinalAmount): ?array` — compares `totalDisbursed` with `approvedFinalAmount`; if overpaid, sets status to `overpaid`, calculates `recoveryAmount`, creates recovery task via `TasksController`
- [ ] 2.3.7 Implement `lockFinancialYear(int $year): int` — locks all grants where `awardDate` falls within the year using `ObjectService::lockObject()`; returns count of locked records
- [ ] 2.3.8 Implement `getOpenCommitments(int $year): array` — returns all grants with `status: active` and no final payment recorded within the specified year, with committed amounts and expected payment dates
- [ ] 2.3.9 Implement `sendAccountabilityRequest(string $grantId): void` — sends formal accountability request to recipient contact person via `NotificationService`; records request sent timestamp

### 2.4 GrantPortfolioService

- [ ] 2.4.1 Create `lib/Service/GrantPortfolioService.php` with `@spec` tag
- [ ] 2.4.2 Implement `calculatePortfolioMetrics(string $portfolioId): array` — aggregates `totalGrantValue` from linked grants, recalculates `concentrationRiskLevel` based on largest single-recipient share
- [ ] 2.4.3 Implement `assessConcentrationRisk(string $portfolioId): string` — returns `low`, `medium`, or `high` based on percentage of total portfolio held by largest single grantee
- [ ] 2.4.4 Implement `getDecisionOutcomes(string $schemeId, string $fromDate, string $toDate): array` — returns counts and percentages for granted, rejected, withdrawn, open applications in the period

### 2.5 AuditorStatementService

- [ ] 2.5.1 Create `lib/Service/AuditorStatementService.php` with `@spec` tag
- [ ] 2.5.2 Implement `registerStatement(array $data): array` — creates AuditorStatement record linked to Grant, sets `isVerified: false`
- [ ] 2.5.3 Implement `verifyStatement(string $statementId, string $auditorRegNum, string $opinionType): array` — sets `isVerified: true`, records `verificationDate`, `auditorRegistrationNumber`, `opinionType` on statement record
- [ ] 2.5.4 Implement `isStatementRequired(string $grantId): bool` — checks if `awardedAmount` exceeds statutory threshold requiring accountantsverklaring; threshold configurable via `IAppConfig`

### 2.6 SanctionsCheckService

- [ ] 2.6.1 Create `lib/Service/SanctionsCheckService.php` with `@spec` tag
- [ ] 2.6.2 Implement `checkApplicant(string $organisationName, string $kvkNumber): array` — calls EU Sanctions API endpoint, OFAC API, and Rijksoverheid exclusion register endpoint; returns structured result with `hasMatch: bool`, `matchedLists: array`, `matchDetails: array`
- [ ] 2.6.3 Handle HTTP timeouts and API errors gracefully — log error via `ILogger`, return `checkStatus: error` (do NOT silently treat error as clean check); notify integrity officer on error
- [ ] 2.6.4 Store API endpoint URLs and timeouts in `IAppConfig` (not hardcoded); mark API keys as sensitive

### 2.7 RiskScoringService

- [ ] 2.7.1 Create `lib/Service/RiskScoringService.php` with `@spec` tag
- [ ] 2.7.2 Implement `scoreApplication(string $applicationId): array` — computes risk score (0-100) based on: requested amount vs. scheme average, first-time applicant flag, number of prior applications and outcomes, high-risk sector classification, sanctions check result; returns score and contributing factor breakdown
- [ ] 2.7.3 Implement `scoreApplicationBatch(array $applicationIds): array` — scores multiple applications in one pass for dashboard loading; do NOT store scores — return computed values only

### 2.8 FraudInvestigationService

- [ ] 2.8.1 Create `lib/Service/FraudInvestigationService.php` with `@spec` tag
- [ ] 2.8.2 Implement `openInvestigation(string $grantId, string $signalType, string $description): array` — creates confidential investigation case record linked to grant; applies payment hold via `GrantService`; restricts record access to investigation team role via `AuthorizationService`; returns case reference
- [ ] 2.8.3 Implement `applyPaymentHold(string $grantId): void` — sets `paymentHold: true` on grant; any subsequent payment attempt checks this flag and is blocked
- [ ] 2.8.4 Implement `closeInvestigation(string $caseId, string $outcome, bool $markForRecovery): void` — closes case; if `markForRecovery: true` calls `GrantService` to set status to `revoked` and generate recovery order

### 2.9 Background Jobs

- [ ] 2.9.1 Create `lib/BackgroundJob/AccountabilityReminderJob.php` extending `TimedJob` with `@spec` tag
- [ ] 2.9.2 Set job interval to daily (86400 seconds)
- [ ] 2.9.3 In `run()`: query all active grants; for each grant where `accountabilityDeadline` is exactly 30 days or 7 days away and no accountability report received, send email reminder via `NotificationService` with deadline date and direct link to accountability form
- [ ] 2.9.4 In `run()`: query grants where `projectEndDate` is today and no accountability request has been sent; call `GrantService::sendAccountabilityRequest()` for each
- [ ] 2.9.5 Create `lib/BackgroundJob/SanctionsCheckJob.php` extending `QueuedJob` with `@spec` tag
- [ ] 2.9.6 In `run()`: call `SanctionsCheckService::checkApplicant()` with applicant data from payload; if match found set application status to `sanctions-hold` and notify integrity officer; if no match set status to `under-review`
- [ ] 2.9.7 Register both jobs in `lib/AppInfo/Application.php` via `IJobList::add()`

---

## 3. Controllers

- [ ] 3.1 Create `lib/Controller/SubsidySchemeController.php` — thin controller (<10 lines/method); GET list, GET detail, POST create, PUT update, POST publish; all admin mutations check `IGroupManager::isAdmin()` on backend; `@spec` tag on class and all methods
- [ ] 3.2 Create `lib/Controller/SubsidyPortalController.php` — public portal endpoint; annotate `#[PublicPage]` and `#[NoCSRFRequired]`; GET `/api/public/subsidy-schemes` returns only `isPublished: true` schemes; CORS OPTIONS route registered in `routes.php`
- [ ] 3.3 Create `lib/Controller/SubsidyApplicationController.php` — GET list, GET detail, POST submit, PUT update, POST generate-letter; sanctions check dispatched asynchronously on submit
- [ ] 3.4 Create `lib/Controller/GrantController.php` — GET list, GET detail, POST create, PUT update, POST process-payment (calls `GrantService::processPayment()` returning 422 with checklist if blocked), POST close, POST lock-year (admin only)
- [ ] 3.5 Create `lib/Controller/GrantPortfolioController.php` — GET list, GET detail, POST create, PUT update
- [ ] 3.6 Create `lib/Controller/AuditorStatementController.php` — GET list, GET detail, POST register, PUT update, POST verify
- [ ] 3.7 Create `lib/Controller/GrantMetricsController.php` — GET `/api/metrics` (Prometheus, admin auth) and GET `/api/health` (JSON, public) per ADR-006; metrics include `shillinq_active_grants_total`, `shillinq_pending_applications_total`, `shillinq_overdue_accountability_total`, `shillinq_health_status`, `shillinq_info`
- [ ] 3.8 Register all routes in `appinfo/routes.php` — specific routes before wildcard `{slug}` catch-all; CORS OPTIONS route for public portal

---

## 4. Frontend — Stores

- [ ] 4.1 Create `src/store/modules/subsidyScheme.js` using `createObjectStore('subsidy-scheme')` with plugins: `auditTrailsPlugin`, `relationsPlugin`, `filesPlugin`, `lifecyclePlugin`, `selectionPlugin`
- [ ] 4.2 Create `src/store/modules/subsidyApplication.js` using `createObjectStore('subsidy-application')` with same plugins
- [ ] 4.3 Create `src/store/modules/grant.js` using `createObjectStore('grant')` with same plugins
- [ ] 4.4 Create `src/store/modules/grantPortfolio.js` using `createObjectStore('grant-portfolio')` with same plugins
- [ ] 4.5 Create `src/store/modules/auditorStatement.js` using `createObjectStore('auditor-statement')` with same plugins
- [ ] 4.6 Register all 5 entity types in `src/store/store.js` via `objectStore.registerObjectType(name, schemaSlug, registerSlug)` — kebab-case type names; each registered exactly once

---

## 5. Frontend — Views

- [ ] 5.1 Create `src/views/DashboardView.vue` — `CnDashboardPage` with 4 `CnStatsBlock` KPIs: active grants count, pending applications, overdue accountability items, total grant value disbursed; status distribution `CnChartWidget` (donut); advance payments `CnTableWidget` sorted by due date with overdue items highlighted; fetch all data in parallel via `Promise.all`
- [ ] 5.2 Create `src/views/SubsidySchemesView.vue` — `CnIndexPage` with `useListView('subsidy-scheme', ...)`, status badges via `CnStatusBadge`, publish action in row actions; filter by `status`, `category`, `governmentLevel`
- [ ] 5.3 Create `src/views/SubsidySchemeDetailView.vue` — `CnDetailPage` with scheme info `CnDetailCard`, linked SubsidyApplications table `CnDetailCard`, linked Grants table `CnDetailCard`; `CnObjectSidebar` with files, notes, audit trail tabs
- [ ] 5.4 Create `src/views/SubsidyApplicationsView.vue` — `CnIndexPage` with `useListView('subsidy-application', ...)`, risk score column, sanctions status badge; filter by `status`, scheme
- [ ] 5.5 Create `src/views/SubsidyApplicationDetailView.vue` — `CnDetailPage` with application info, sanctions check result, eligibility assessment section, decision letter action button; `CnObjectSidebar`
- [ ] 5.6 Create `src/views/GrantsView.vue` — `CnIndexPage` with `useListView('grant', ...)`, SiSa flag column, accountability deadline column, overdue indicator; filter by `status`, `isSISAEligible`, year
- [ ] 5.7 Create `src/views/GrantDetailView.vue` — `CnDetailPage` with disbursement timeline (`CnTimelineStages`), accountability checklist section, payment hold indicator, auditor statement `CnDetailCard`; process payment and close dossier header actions; `CnObjectSidebar`
- [ ] 5.8 Create `src/views/GrantPortfoliosView.vue` — `CnIndexPage` with compliance status and risk level badges
- [ ] 5.9 Create `src/views/GrantPortfolioDetailView.vue` — `CnDetailPage` with portfolio metrics `CnStatsPanel`, grants table, concentration risk `CnChartWidget` (bar)
- [ ] 5.10 Create `src/views/AuditorStatementsView.vue` — `CnIndexPage` with verified status badge, verdict badge
- [ ] 5.11 Create `src/views/AuditorStatementDetailView.vue` — `CnDetailPage` with statement details, linked grant, uploaded document via `CnObjectSidebar`; verify action button

---

## 6. Frontend — Modals & Dialogs

Each modal/dialog in its own `.vue` file per ADR-004. No inline modal markup in parent components.

- [ ] 6.1 Create `src/modals/PublishSchemeModal.vue` (NcModal-based) — confirmation dialog with portal preview; confirm button calls `subsidySchemeStore.publishScheme(id)`
- [ ] 6.2 Create `src/modals/GenerateDecisionLetterModal.vue` (NcModal-based) — outcome selector (granted/rejected/withdrawn), letter preview pane, editable text area, finalise button
- [ ] 6.3 Create `src/modals/OpenFraudInvestigationModal.vue` (NcModal-based) — signal type selector, description text area, linked grant lookup; submit calls fraud investigation API
- [ ] 6.4 Create `src/dialogs/PaymentBlockDialog.vue` (NcDialog-based) — shows checklist of outstanding conditions; each condition with met/unmet status; no confirm button if any condition unmet
- [ ] 6.5 Create `src/dialogs/LockFinancialYearDialog.vue` (NcDialog-based) — year selector, confirmation checkbox ("I confirm this action is irreversible"), lock button calls grant year-lock API
- [ ] 6.6 Create `src/dialogs/VerifyAuditorStatementDialog.vue` (NcDialog-based) — auditor registration number field, opinion type selector, verification date; save calls auditor statement verify endpoint

---

## 7. Frontend — Navigation & Router

- [ ] 7.1 Update `src/router/index.js` — add named flat routes: `/` (Dashboard), `/subsidy-schemes` (list), `/subsidy-schemes/:id` (detail), `/subsidy-applications` (list), `/subsidy-applications/:id` (detail), `/grants` (list), `/grants/:id` (detail), `/grant-portfolios` (list), `/grant-portfolios/:id` (detail), `/auditor-statements` (list), `/auditor-statements/:id` (detail); props via arrow function for `:id` params
- [ ] 7.2 Update `src/components/MainMenu.vue` — add `NcAppNavigationItem` for each entity list route; icons from MDI; footer `NcAppNavigationSettings` for in-app settings modal (do NOT add `/settings` route)
- [ ] 7.3 Update `src/App.vue` — register all 5 entity stores in `initializeStores()`; inject `sidebarState` for child components

---

## 8. Translations

- [ ] 8.1 Add Dutch translations (`l10n/nl.json`) for all new user-visible strings: entity labels, status values (draft/ingediend/in-behandeling/etc.), button labels, notification messages, error messages
- [ ] 8.2 Add English translations (`l10n/en.json`) for same strings
- [ ] 8.3 Ensure all `t()` calls in Vue use `this.t('shillinq', 'English key')` — translation keys must be English; Dutch text goes only in `l10n/nl.json`

---

## 9. SPDX License Headers

- [ ] 9.1 Add `// SPDX-License-Identifier: EUPL-1.2` header to every new PHP file
- [ ] 9.2 Add `<!-- SPDX-License-Identifier: EUPL-1.2 -->` as first line in every new Vue file
- [ ] 9.3 Add `// SPDX-License-Identifier: EUPL-1.2` as first line in every new JS file
- [ ] 9.4 Run `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` and fix any missing headers

---

## 10. Tests

- [ ] 10.1 Create `tests/Unit/Service/SubsidySchemeServiceTest.php` — ≥3 test methods: testCreateScheme, testPublishScheme, testGetPublishedSchemesFilter
- [ ] 10.2 Create `tests/Unit/Service/SubsidyApplicationServiceTest.php` — ≥3 test methods: testSubmitApplicationDispatchesSanctionsCheck, testApproveApplicationCreatesGrant, testGenerateDecisionLetterCallsDocudesk
- [ ] 10.3 Create `tests/Unit/Service/GrantServiceTest.php` — ≥5 test methods: testProcessPaymentBlockedWhenConditionsUnmet, testProcessPaymentSucceedsWhenConditionsMet, testDetectOverpaymentFlagsGrant, testCloseDossierLocksRecord, testLockFinancialYearLocksAllGrantsInYear
- [ ] 10.4 Create `tests/Unit/Service/AuditorStatementServiceTest.php` — ≥3 test methods: testRegisterStatement, testVerifyStatement, testIsStatementRequiredAboveThreshold
- [ ] 10.5 Create `tests/Unit/Service/RiskScoringServiceTest.php` — ≥3 test methods: testScoreApplicationHighAmount, testScoreApplicationFirstTimeApplicant, testScoreApplicationBatchReturnsAllScores
- [ ] 10.6 Create `tests/Unit/Service/SanctionsCheckServiceTest.php` — ≥3 test methods: testCheckApplicantReturnsMatchOnHit, testCheckApplicantReturnsCleanOnNoMatch, testCheckApplicantHandlesApiErrorGracefully
- [ ] 10.7 Create `tests/integration/grant-subsidy-management.postman_collection.json` — Newman collection covering: GET /api/public/subsidy-schemes (public), POST /api/subsidy-applications (submit), GET /api/grants (list), POST /api/grants/{id}/process-payment (block scenario + success scenario), GET /api/health, GET /api/metrics (admin)
- [ ] 10.8 Verify all tests pass with `composer check:strict`

---

## 11. Pre-commit Verification

Run all checks before committing:

- [ ] 11.1 SPDX headers: `grep -rL 'SPDX-License-Identifier' src/ lib/ --include='*.php' --include='*.vue' --include='*.js'` → must return zero files
- [ ] 11.2 ObjectService calls: `grep -rn 'findObject\|saveObject\|findObjects' lib/ --include='*.php'` → verify every call has 3 positional args
- [ ] 11.3 Error responses: `grep -rn 'getMessage()' lib/Controller/ --include='*.php'` → must return zero matches (use static error strings)
- [ ] 11.4 Auth checks: every POST/PUT/DELETE controller method for admin operations calls `IGroupManager::isAdmin()` on backend
- [ ] 11.5 Store registration: `grep -rn 'registerObjectType' src/store/store.js` → verify all 5 entity types registered, kebab-case names
- [ ] 11.6 Dependencies: `npm run lint` — no missing `package.json` entries
- [ ] 11.7 No raw fetch: `grep -rn 'fetch(' src/ --include='*.vue' --include='*.js'` → must use `@nextcloud/axios` for mutations
- [ ] 11.8 No direct @nextcloud/vue imports: `grep -rn "from '@nextcloud/vue'" src/` → must return zero matches
- [ ] 11.9 Component imports: every `<NcFoo>` and `<CnFoo>` in templates is imported and in `components: {}`
- [ ] 11.10 Modal isolation: `grep -rn 'NcModal\|NcDialog' src/views/ --include='*.vue'` → must return zero matches (all modals/dialogs in `src/modals/` or `src/dialogs/`)
- [ ] 11.11 Translation keys: all `t()` keys are English strings, not Dutch
- [ ] 11.12 try/catch: every `await store.action()` call is wrapped in `try/catch` with user-facing error feedback
