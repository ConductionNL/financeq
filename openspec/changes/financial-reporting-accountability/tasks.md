# Tasks: Financial Reporting & Accountability — Shillinq

## 1. Deduplication Check

- [ ] 1.1 Verify no overlap with ObjectService, RegisterService, SchemaService for the 11 new schemas (ADR-012)
- [ ] 1.2 Confirm ExportService + CnMassExportDialog covers CSV/Excel/JSON export — no custom export controllers needed
- [ ] 1.3 Confirm AuditTrailService covers all change tracking — no custom audit logging needed
- [ ] 1.4 Confirm WorkflowEngineController can handle accountability report status transitions — no custom state machine needed
- [ ] 1.5 Document findings in design.md Reuse Analysis table

## 2. Register & Schema Setup

- [ ] 2.1 Create `lib/Settings/shillinq_register.json` with OpenAPI 3.0 + x-openregister definitions for all 11 schemas: FiscalYear, GeneralLedgerAccount, JournalEntry, GeneralLedgerEntry, BalanceSheet, TrialBalance, FinancialReport, AccountabilityReport, ConsolidationGroup, ConsolidatedReport, RevenueStream
- [ ] 2.2 Add `@spec` file-level PHPDoc tag to all generated files linking to `openspec/changes/financial-reporting-accountability/tasks.md`
- [ ] 2.3 Create `lib/Repair/InitializeShillinqRegisterRepairStep.php` implementing `IRepairStep` — imports `shillinq_register.json` and loads seed data from `design.md` on first install
- [ ] 2.4 Register the repair step in `appinfo/info.xml` under `<repair-steps><install-repair-steps>`

## 3. Backend — Fiscal Year Service

- [ ] 3.1 Create `lib/Service/FiscalYearService.php` with methods: `createFiscalYear()`, `closeFiscalYear()`, `isYearClosed(int $year): bool`
- [ ] 3.2 `closeFiscalYear()` checks for unposted (draft) JournalEntries — blocks close if any exist; returns list of unposted entry count (REQ-FRA-012)
- [ ] 3.3 `isYearClosed()` is called by JournalEntryService and GeneralLedgerService as a posting guard — throws a 409 exception when closed (REQ-FRA-001)
- [ ] 3.4 Write PHPUnit tests in `tests/Unit/Service/FiscalYearServiceTest.php` (≥3 methods: testClose, testPostingGuard, testReopenYear)

## 4. Backend — Journal Entry Service

- [ ] 4.1 Create `lib/Service/JournalEntryService.php` with methods: `createEntry()`, `postEntry()`, `reverseEntry()`
- [ ] 4.2 `createEntry()` validates `debitAmount === creditAmount` before calling `ObjectService.saveObject()`; returns HTTP 422 with Dutch message on mismatch (REQ-FRA-002)
- [ ] 4.3 `createEntry()` calls `FiscalYearService.isYearClosed()` and throws HTTP 409 if the referenced fiscal year is closed (REQ-FRA-001)
- [ ] 4.4 `reverseEntry()` creates a new JournalEntry with swapped debit/credit and sets original entry status=reversed (REQ-FRA-002)
- [ ] 4.5 Write PHPUnit tests in `tests/Unit/Service/JournalEntryServiceTest.php` (≥3 methods: testBalancedEntry, testUnbalancedRejection, testReversal)

## 5. Backend — General Ledger Service

- [ ] 5.1 Create `lib/Service/GeneralLedgerService.php` with methods: `createAccount()`, `getAccountBalance(string $accountNumber, \DateTime $asOf): float`, `getGeneralLedgerReport(string $accountNumber, \DateTime $from, \DateTime $to): array`
- [ ] 5.2 `createAccount()` checks for duplicate accountNumber and returns HTTP 409 with Dutch message if duplicate (REQ-FRA-003)
- [ ] 5.3 `getGeneralLedgerReport()` returns opening balance, period movements (debits, credits), and closing balance for the requested account and date range (REQ-FRA-011)
- [ ] 5.4 Write PHPUnit tests in `tests/Unit/Service/GeneralLedgerServiceTest.php` (≥3 methods: testDuplicateAccountRejection, testAccountBalance, testLedgerReport)

## 6. Backend — Financial Statement Service

- [ ] 6.1 Create `lib/Service/FinancialStatementService.php` with methods: `generateBalanceSheet(\DateTime $asOf, string $organizationId): array`, `generateTrialBalance(\DateTime $asOf, string $organizationId): array`
- [ ] 6.2 `generateBalanceSheet()` computes totalAssets, totalLiabilities, totalEquity from posted GL entries up to the given date; validates accounting equation (assets = liabilities + equity); saves a BalanceSheet object via ObjectService (REQ-FRA-004)
- [ ] 6.3 `generateTrialBalance()` sums debit and credit balances per account; sets isBalanced=true/false; sends a Nextcloud notification via NotificationService if isBalanced=false (REQ-FRA-005)
- [ ] 6.4 Write PHPUnit tests in `tests/Unit/Service/FinancialStatementServiceTest.php` (≥3 methods: testBalanceSheetEquation, testTrialBalanceBalanced, testTrialBalanceUnbalancedNotification)

## 7. Backend — Financial Report Service

- [ ] 7.1 Create `lib/Service/FinancialReportService.php` with methods: `generateAnnualReport(string $fiscalYearId, string $format): string`, `addFootnote(string $reportId, string $lineItemRef, string $note): void`
- [ ] 7.2 `generateAnnualReport()` for format=PDF calls docudesk with a Shillinq-specific report template; attaches the file to the FinancialReport record via FileService (REQ-FRA-006)
- [ ] 7.3 `generateAnnualReport()` for format=CSV/Excel/JSON delegates to OpenRegister ExportService — no custom implementation (REQ-FRA-006)
- [ ] 7.4 `addFootnote()` stores the note linked to the line item; PDF re-export reads footnotes and renders them as footnotes in the PDF template (REQ-FRA-006)
- [ ] 7.5 Write PHPUnit tests in `tests/Unit/Service/FinancialReportServiceTest.php` (≥3 methods: testPdfExportAttachment, testCsvDelegatesToExportService, testFootnoteLinkedToLineItem)

## 8. Backend — XBRL Export Service (should-have)

- [ ] 8.1 Create `lib/Service/XbrlExportService.php` with method `generateEsefReport(string $financialReportId): string`
- [ ] 8.2 Map GeneralLedgerAccount entries to IFRS taxonomy elements; create custom extension elements where no standard tag exists
- [ ] 8.3 Apply block tagging to notes sections in the XHTML document
- [ ] 8.4 Validate output against ESEF Conformance Suite validation rules; attach to FinancialReport via FileService (REQ-FRA-007)
- [ ] 8.5 Write PHPUnit tests in `tests/Unit/Service/XbrlExportServiceTest.php` (≥3 methods: testIfrsMapping, testCustomExtensionCreation, testConformanceSuiteValidation)

## 9. Backend — Accountability Report Service

- [ ] 9.1 Create `lib/Service/AccountabilityReportService.php` with methods: `submitReport(string $reportId): void`, `approveReport(string $reportId): void`, `rejectReport(string $reportId, string $reason): void`, `sendReportRequest(string $reportId, string $recipientId): void`
- [ ] 9.2 Register a Shillinq workflow definition with WorkflowEngineController for the draft→submitted→approved/rejected transitions (REQ-FRA-008)
- [ ] 9.3 Each transition dispatches a Nextcloud notification via NotificationService to the relevant party (submitter notified on approve/reject; recipient notified on submit) (REQ-FRA-008)
- [ ] 9.4 Write PHPUnit tests in `tests/Unit/Service/AccountabilityReportServiceTest.php` (≥3 methods: testSubmitTransition, testApproveNotification, testRejectWithReason)

## 10. Backend — Consolidation Service

- [ ] 10.1 Create `lib/Service/ConsolidationService.php` with method `generateConsolidatedReport(string $consolidationGroupId, string $fiscalYearId): array`
- [ ] 10.2 Load all member organizations from the ConsolidationGroup and fetch their BalanceSheet records for the fiscal year
- [ ] 10.3 Apply eliminationRules from ConsolidationGroup to remove inter-company transactions; set eliminationsApplied=true on the ConsolidatedReport (REQ-FRA-009)
- [ ] 10.4 Write PHPUnit tests in `tests/Unit/Service/ConsolidationServiceTest.php` (≥3 methods: testFullConsolidation, testEliminationRulesApplied, testConsolidatedReportCreated)

## 11. Backend — Revenue Stream Service

- [ ] 11.1 Create `lib/Service/RevenueStreamService.php` with methods: `createStream()`, `getIncomeOverview(string $fiscalYearId): array`
- [ ] 11.2 `getIncomeOverview()` aggregates JournalEntry amounts by RevenueStream.category, grouped by quarter (Q1–Q4) and full year (REQ-FRA-010)
- [ ] 11.3 Write PHPUnit tests in `tests/Unit/Service/RevenueStreamServiceTest.php` (≥3 methods: testCreateStream, testIncomeGroupedByCategory, testQuarterlyAggregation)

## 12. Backend — Controllers and Routes

- [ ] 12.1 Create `lib/Controller/FinancialReportController.php` with routes: `POST /api/financial-reports/generate` (trigger report generation), `GET /api/financial-reports/{id}/export` (download), `POST /api/financial-reports/{id}/footnotes` (add footnote)
- [ ] 12.2 Create `lib/Controller/AccountabilityReportController.php` with routes: `POST /api/accountability-reports/{id}/submit`, `POST /api/accountability-reports/{id}/approve`, `POST /api/accountability-reports/{id}/reject`, `POST /api/accountability-reports/{id}/send-request`
- [ ] 12.3 Create `lib/Controller/MetricsController.php` implementing ADR-006: `GET /api/metrics` (Prometheus text, admin auth), `GET /api/health` (JSON, public, verifies OpenRegister connectivity) (REQ-FRA-015)
- [ ] 12.4 Register all routes in `appinfo/routes.php`; place specific routes before wildcard `{slug}` catch-all (ADR-003)
- [ ] 12.5 Controllers must be thin (<10 lines per method): validate input, call service, return JSON response (ADR-003)
- [ ] 12.6 Create Newman/Postman test collection in `tests/integration/financial-reporting.json` covering all new endpoints (ADR-008)

## 13. Background Jobs

- [ ] 13.1 Create `lib/BackgroundJob/YearEndProcessingJob.php` — nightly job that checks for fiscal years approaching endDate; triggers accrual creation via FinancialStatementService; logs via ActivityService (REQ-FRA-012)
- [ ] 13.2 Create `lib/BackgroundJob/OverdueAccountabilityReportJob.php` — nightly job that queries AccountabilityReport objects with status=submitted past due date; sends notifications via NotificationService (REQ-FRA-008)
- [ ] 13.3 Register both jobs in `appinfo/info.xml` under `<background-jobs>`

## 14. Frontend — Object Stores

- [ ] 14.1 Create `src/store/modules/fiscalYears.js` using `createObjectStore('fiscalYears')` with plugins: `auditTrailsPlugin`, `relationsPlugin`
- [ ] 14.2 Create `src/store/modules/journalEntries.js` using `createObjectStore('journalEntries')` with plugins: `auditTrailsPlugin`, `relationsPlugin`, `filesPlugin`
- [ ] 14.3 Create `src/store/modules/generalLedgerAccounts.js` using `createObjectStore('generalLedgerAccounts')` with plugins: `auditTrailsPlugin`
- [ ] 14.4 Create `src/store/modules/balanceSheets.js`, `trialBalances.js`, `financialReports.js`, `accountabilityReports.js`, `consolidatedReports.js`, `consolidationGroups.js`, `revenueStreams.js`, `generalLedgerEntries.js` — each using `createObjectStore` with `auditTrailsPlugin` and `relationsPlugin`
- [ ] 14.5 Register all stores in `src/store/store.js` via `initializeStores()` with `objectStore.registerObjectType(name, schemaSlug, registerSlug)` calls for each entity

## 15. Frontend — Router

- [ ] 15.1 Add flat named routes to `src/router.js` for all entity views: `/fiscal-years`, `/fiscal-years/:id`, `/journal-entries`, `/journal-entries/:id`, `/general-ledger-accounts`, `/general-ledger-accounts/:id`, `/balance-sheets`, `/balance-sheets/:id`, `/trial-balances`, `/trial-balances/:id`, `/financial-reports`, `/financial-reports/:id`, `/accountability-reports`, `/accountability-reports/:id`, `/consolidated-reports`, `/consolidated-reports/:id`, `/revenue-streams`, `/revenue-streams/:id`
- [ ] 15.2 Add `/settings` route; catch-all `*` redirects to `/`

## 16. Frontend — Dashboard

- [ ] 16.1 Implement `src/views/Dashboard.vue` using `CnDashboardPage` with `useDashboardView` composable
- [ ] 16.2 Add four KPI cards via `CnStatsBlock`: Total Assets, Total Liabilities, Total Equity (from current BalanceSheet), and Net Revenue (from RevenueStream income overview) (REQ-FRA-013)
- [ ] 16.3 Add income vs expense bar chart using `CnChartWidget` (ApexCharts bar); period = current fiscal year by month (REQ-FRA-013)
- [ ] 16.4 Add "Mijn werk" list: open AccountabilityReports grouped by status (overdue → submitted → draft) using `CnTableWidget`
- [ ] 16.5 Fetch all data in parallel via `Promise.all` in `created()`; use `try/finally` for loading state

## 17. Frontend — Entity Views

- [ ] 17.1 Create `src/views/FiscalYearsIndex.vue` and `FiscalYearDetail.vue` using `CnIndexPage` / `CnDetailPage` + `useListView` / `useDetailView`; detail shows close/reopen action buttons that call the FiscalYearService endpoints
- [ ] 17.2 Create `src/views/JournalEntriesIndex.vue` and `JournalEntryDetail.vue`; form includes debit/credit fields with real-time balance indicator (debit = credit check); inline balance status badge using `CnStatusBadge`
- [ ] 17.3 Create `src/views/GeneralLedgerAccountsIndex.vue` and `GeneralLedgerAccountDetail.vue`; detail shows linked JournalEntries in a `CnDetailCard` table
- [ ] 17.4 Create `src/views/BalanceSheetsIndex.vue` and `BalanceSheetDetail.vue`; detail includes "Genereer balans" action calling the generate endpoint with a date picker
- [ ] 17.5 Create `src/views/TrialBalancesIndex.vue` and `TrialBalanceDetail.vue`; list shows isBalanced status via `CnStatusBadge` (green=balanced, red=unbalanced)
- [ ] 17.6 Create `src/views/FinancialReportsIndex.vue` and `FinancialReportDetail.vue`; detail has format picker (PDF/Excel/XML/JSON) and "Exporteren" action; footnote editor linked per line item
- [ ] 17.7 Create `src/views/AccountabilityReportsIndex.vue` and `AccountabilityReportDetail.vue`; detail shows `CnTimelineStages` for draft→submitted→approved/rejected progression; action buttons for submit/approve/reject/send-request
- [ ] 17.8 Create `src/views/ConsolidatedReportsIndex.vue` and `ConsolidatedReportDetail.vue`; detail shows member organization breakdown and elimination summary
- [ ] 17.9 Create `src/views/ConsolidationGroupsIndex.vue` and `ConsolidationGroupDetail.vue`; detail shows member organizations list and `eliminationRules` as a `CnJsonViewer`
- [ ] 17.10 Create `src/views/RevenueStreamsIndex.vue` and `RevenueStreamDetail.vue`; detail shows period income chart via `CnChartWidget`

## 18. Frontend — MainMenu and App.vue

- [ ] 18.1 Update `src/App.vue` with `NcContent` wrapper; three states: loading (`NcLoadingIcon`), no-OpenRegister (`NcEmptyContent`), ready (`MainMenu` + `NcAppContent` + `router-view`); `created()` calls `initializeStores()`
- [ ] 18.2 Update `src/views/MainMenu.vue` with `NcAppNavigation`; navigation items: Dashboard, Boekjaren, Grootboek, Journaalposten, Balansrekening, Proefbalans, Financiële rapporten, Verantwoordingsrapportages, Geconsolideerde rapporten, Inkomstenstromen; footer: settings link via `NcAppNavigationSettings`

## 19. Frontend — Settings

- [ ] 19.1 Create `src/views/Settings.vue` using `CnVersionInfoCard` (first, always) → `CnRegisterMapping` → `CnSettingsSection` for each configurable feature
- [ ] 19.2 Load settings from `GET /api/settings`; save via `POST /api/settings`; re-import button calls `POST /api/settings/load`
- [ ] 19.3 Add admin route `/settings` and `NcAppNavigationSettings` link in MainMenu

## 20. Translations (ADR-007)

- [ ] 20.1 Create `l10n/nl.js` with Dutch translations for all user-visible strings (navigation labels, KPI card titles, action buttons, error messages, status labels)
- [ ] 20.2 Create `l10n/en.js` with English translations for the same key set
- [ ] 20.3 Wrap all PHP user-visible strings with `$this->l->t()` in services and controllers
- [ ] 20.4 Wrap all Vue user-visible strings with `t(appName, 'key')` in all view components

## 21. Testing

- [ ] 21.1 Write PHPUnit tests for all services listed in tasks 3–11 (≥3 methods each) — minimum coverage: FiscalYearService, JournalEntryService, GeneralLedgerService, FinancialStatementService, FinancialReportService, XbrlExportService, AccountabilityReportService, ConsolidationService, RevenueStreamService
- [ ] 21.2 Write Playwright browser tests for each GIVEN/WHEN/THEN scenario in specs.md — one test file per requirement REQ-FRA-001 through REQ-FRA-015 in `tests/e2e/`
- [ ] 21.3 Add Newman/Postman integration test collection for all API endpoints in `tests/integration/financial-reporting.json`
- [ ] 21.4 Ensure all tests pass under `composer check:strict` (ADR-008)

## 22. Documentation

- [ ] 22.1 Create `docs/financial-reporting.md` with English documentation covering all user-facing features: fiscal year management, journal entries, balance sheet, trial balance, annual report export, accountability report workflow, consolidated reporting, revenue streams (ADR-009)
- [ ] 22.2 Add screenshots from a running Shillinq instance to the docs folder once the app is deployed
