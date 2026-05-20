# Proposal: Financial Reporting & Accountability — Shillinq

## Why

Freelancers, SMBs, foundations, and Dutch public-sector organizations using Shillinq need a complete financial reporting and accountability module to meet statutory obligations: BBV jaarrekening, IV3 rapportages, ESEF/XBRL filing for listed companies, and board-level annual report packages. Without it, users depend on spreadsheets and external software for their most critical year-end compliance tasks, defeating the purpose of a self-hosted business administration suite.

## What Changes

- Fiscal year lifecycle management: open periods, year-end close, re-open with audit trail
- Double-entry general ledger with balanced journal entries and chart of accounts
- Balance sheet and trial balance generation at any point in time with comparative periods
- Annual report export in PDF (board-ready), Excel, XML, and JSON formats
- Accountability report workflow: draft → submit → approve/reject with recipient notification
- Consolidated financial reporting across multiple administrations with inter-company eliminations
- Revenue stream tracking by category with annual targets and actuals
- ESEF/XBRL export with inline tagging for AFM filing (listed companies, should-have)
- Financial KPI dashboard with income vs expense charts and period comparisons

## Capabilities

### New Capabilities

- `fiscal-year-management`: Open, close, and reopen accounting periods; enforce year-end processing; prevent posting to closed years
- `general-ledger`: Double-entry journal entries with automatic balance validation (debits = credits); chart-of-accounts management across asset/liability/equity/revenue/expense types
- `financial-statements`: Generate balance sheets, trial balances, and income summaries at any date with opening/period/closing balances
- `annual-report-export`: Export annual financial reports as board-ready PDF (via docudesk), Excel, XML (UBL), and JSON; narrative footnotes linked to line items
- `accountability-reporting`: Manage accountability report lifecycle from draft through board approval; send report requests to recipients; flag and escalate overdue reports
- `consolidated-reporting`: Consolidate financial data from multiple organizations into a single management reporting package with automatic inter-company eliminations
- `revenue-management`: Categorize and track revenue streams (subsidies, service fees, grants, licensing) with annual targets; aggregate across fiscal periods
- `esef-xbrl-filing`: Generate ESEF-compliant XHTML with inline XBRL tags mapped to IFRS taxonomy; validate against ESEF Conformance Suite for AFM filing

## Impact

- `lib/Settings/shillinq_register.json`: Register template defining all 11 new OpenRegister schemas
- `lib/Repair/InitializeShillinqRegisterRepairStep.php`: Schema initialization and seed data import on first install
- `lib/Service/FiscalYearService.php`: Fiscal year open/close logic; prevent posting to closed periods
- `lib/Service/JournalEntryService.php`: Double-entry validation (isBalanced), posting, reversal
- `lib/Service/GeneralLedgerService.php`: Chart-of-accounts management; balance aggregation per account
- `lib/Service/FinancialStatementService.php`: Balance sheet and trial balance generation at a given date
- `lib/Service/FinancialReportService.php`: Annual report assembly and multi-format export (PDF, Excel, XML, JSON)
- `lib/Service/XbrlExportService.php`: ESEF/XBRL document generation with IFRS taxonomy mapping
- `lib/Service/AccountabilityReportService.php`: Accountability report workflow; overdue escalation; recipient notification
- `lib/Service/ConsolidationService.php`: Multi-entity consolidation with elimination-rule evaluation
- `lib/Service/RevenueStreamService.php`: Revenue categorization and period aggregation
- `lib/Controller/FinancialReportController.php`: Report generation and export endpoints
- `lib/Controller/AccountabilityReportController.php`: Accountability report workflow endpoints
- `lib/BackgroundJob/YearEndProcessingJob.php`: Scheduled year-end accruals and closing validation
- `lib/BackgroundJob/OverdueAccountabilityReportJob.php`: Escalation job for overdue reports
- `appinfo/routes.php`: All new API routes for above controllers
- `src/store/modules/fiscalYears.js`, `journalEntries.js`, `generalLedgerAccounts.js`, `balanceSheets.js`, `trialBalances.js`, `accountabilityReports.js`, `consolidatedReports.js`, `consolidationGroups.js`, `financialReports.js`, `revenueStreams.js`, `generalLedgerEntries.js`: Pinia object stores via `createObjectStore`
- `src/views/Dashboard.vue`: Financial KPI dashboard with charts and "My Work" accountability list
- `src/views/FiscalYears*.vue`, `JournalEntries*.vue`, `GeneralLedgerAccounts*.vue`, `BalanceSheets*.vue`, `TrialBalances*.vue`, `AccountabilityReports*.vue`, `ConsolidatedReports*.vue`, `FinancialReports*.vue`, `RevenueStreams*.vue`: Index and detail pages for all entities
- `src/views/Settings.vue`: Admin settings page with register mapping and version info
- `l10n/nl.js`, `l10n/en.js`: Dutch and English translations for all user-visible strings
