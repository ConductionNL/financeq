# Tasks: Treasury & Cash Management — Shillinq — Other T4

**Status:** Tasks  
**Change:** treasury-cash-management-other-t4  
**Effort:** 120 hours (4 weeks, 1 backend + 1 frontend + 1 QA)  
**Last Updated:** 2026-05-21

---

## Overview

Implementation tasks for Phase 1 of treasury and cash management features. Tasks are organized by feature and include:
- Acceptance criteria (reference REQ-XXX-NNN from specs.md)
- Implementation steps (backend/frontend/testing)  
- Dependencies and blockers  
- Spec traceability via `@spec` tags  

---

## Data Model & Schema Setup

### Task 1: Define PaymentBatch schema in OpenRegister
- [ ] **Design:** Review design.md PaymentBatch definition (section Part I, Data Model)  
- [ ] **Backend:** Create `lib/Settings/shillinq_register.json` entry for PaymentBatch schema  
  - [ ] Add properties: id, administration_id, created_by, created_date, status (enum: draft/approved/executed/cancelled)  
  - [ ] Add payment_ids array, approval_chain_id, approved_by, approved_date, executed_date, notes  
  - [ ] Add $defs for MonetaryAmount (amount + currency)  
  - [ ] Validate schema per OpenAPI 3.0 + x-openregister  
  - [ ] Test parsing: POST /api/payment-batches with sample data from design.md  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-1`  
- [ ] **Acceptance:** Schema is registered, sample batch objects can be created/retrieved via ObjectService  

**Dependencies:** None  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 2: Define PaymentProvider schema in OpenRegister
- [ ] **Design:** Review design.md PaymentProvider definition  
- [ ] **Backend:** Create `lib/Settings/shillinq_register.json` entry for PaymentProvider schema  
  - [ ] Add properties: id, name, type (enum: bank/fintech/internal), administration_id  
  - [ ] Add supported_currencies array, api_key_encrypted, is_active, created_date  
  - [ ] Implement encryption for api_key_encrypted field (use IAppConfig::getSecretValue())  
  - [ ] Add validation: at least one supported currency required  
  - [ ] Test: Create sample providers from design.md  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-2`  
- [ ] **Acceptance:** Schema registered, sample providers can be created with encrypted API keys  

**Dependencies:** Task 1 (schema setup pattern)  
**Blockers:** None  
**Effort:** 3 hours

---

### Task 3: Define CashFlowForecast schema in OpenRegister
- [ ] **Design:** Review design.md CashFlowForecast definition  
- [ ] **Backend:** Create `lib/Settings/shillinq_register.json` entry  
  - [ ] Add properties: id, administration_id, base_date, horizon_days  
  - [ ] Add projections array (items: date, opening_balance, inflows, outflows, closing_balance)  
  - [ ] Add assumptions object (metadata for forecast assumptions)  
  - [ ] Test: Create sample forecast from design.md  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-3`  
- [ ] **Acceptance:** Schema registered, forecasts can be stored/retrieved  

**Dependencies:** Task 1  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 4: Define BankStatementEntry schema in OpenRegister
- [ ] **Design:** Review design.md BankStatementEntry definition  
- [ ] **Backend:** Create `lib/Settings/shillinq_register.json` entry  
  - [ ] Add properties: id, bank_account_id, import_date, statement_date, entry_date  
  - [ ] Add amount, counterparty_name, counterparty_iban, reference, transaction_code  
  - [ ] Add matched_ledger_entry_id, matched_confidence, is_reconciled  
  - [ ] Add index on bank_account_id + statement_date for query performance  
  - [ ] Test: Create sample entries from design.md  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-4`  
- [ ] **Acceptance:** Schema registered, entries indexed for reconciliation queries  

**Dependencies:** Task 1  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 5: Extend BankAccount entity with reconciliation fields
- [ ] **Design:** Review design.md BankAccount updates (reconciliationRule, reserve_minimum, lastReconciled)  
- [ ] **Backend:** Add schema migration via repair step (ADR-001-data-layer pattern)  
  - [ ] Add optional fields: reconciliationRule (string, default "automated_match_high_confidence")  
  - [ ] Add reserve_minimum (MonetaryAmount type)  
  - [ ] Add lastReconciled (DateTime, nullable)  
  - [ ] Ensure backward compatibility: existing BankAccount objects have null for new fields  
  - [ ] Test: Fetch existing BankAccount via ObjectService, verify new fields present  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-5`  
- [ ] **Acceptance:** Existing BankAccount objects can be updated with new fields  

**Dependencies:** None (extends existing entity)  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 6: Extend Payment entity with payment optimization fields
- [ ] **Design:** Review design.md Payment updates (netting_group_id, provider_id)  
- [ ] **Backend:** Add schema migration via repair step  
  - [ ] Add optional fields: netting_group_id (UUID, nullable)  
  - [ ] Add provider_id (UUID, nullable, reference to PaymentProvider)  
  - [ ] Ensure backward compatibility  
  - [ ] Test: Existing Payment objects can be fetched and modified  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-6`  
- [ ] **Acceptance:** Payment objects support netting and provider routing  

**Dependencies:** None  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 7: Extend Supplier entity with IBAN verification fields
- [ ] **Design:** Review design.md Supplier updates (iban_verified, iban_verified_date, iban_verified_by)  
- [ ] **Backend:** Add schema migration via repair step  
  - [ ] Add optional fields: iban_verified (boolean, default false)  
  - [ ] Add iban_verified_date (DateTime, nullable)  
  - [ ] Add iban_verified_by (User reference, nullable)  
  - [ ] Add payment_provider_id (UUID, nullable)  
  - [ ] Test: Supplier objects support verification tracking  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-7`  
- [ ] **Acceptance:** Supplier IBAN verification audit trail is recorded  

**Dependencies:** None  
**Blockers:** None  
**Effort:** 2 hours

---

### Task 8: Extend Loan entity with refinancing fields
- [ ] **Design:** Review design.md Loan updates (is_eligible_for_refinancing, refinancing_score)  
- [ ] **Backend:** Add schema migration via repair step  
  - [ ] Add computed fields: is_eligible_for_refinancing (boolean, computed from market rate)  
  - [ ] Add refinancing_score (float, computed)  
  - [ ] Implement computation method in LoanService (see Task 17)  
  - [ ] Test: is_eligible_for_refinancing updates when market rate changes  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-8`  
- [ ] **Acceptance:** Loan objects compute eligibility and score dynamically  

**Dependencies:** Task 1  
**Blockers:** None  
**Effort:** 1 hour

---

### Task 9: Extend GeneralLedgerEntry with bank reconciliation link
- [ ] **Design:** Review design.md GeneralLedgerEntry updates (bank_entry_id field)  
- [ ] **Backend:** Add schema migration via repair step  
  - [ ] Add optional field: bank_entry_id (UUID, reference to BankStatementEntry)  
  - [ ] Add index on bank_entry_id for reconciliation queries  
  - [ ] Test: GLEntry objects link to BankStatementEntry  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/tasks.md#task-9`  
- [ ] **Acceptance:** Bank entries are linked to ledger entries during reconciliation  

**Dependencies:** Task 4  
**Blockers:** None  
**Effort:** 1 hour

---

## Bank Reconciliation (REQ-001)

### Task 10: Implement MT940 bank statement import
- [ ] **Backend:** Create BankStatementImportService  
  - [ ] Method: `importMT940File(BankAccount $account, Stream $fileStream): ImportResult`  
  - [ ] Parse MT940 standard: transaction fields (32A=date/amount, 33B=amount, 60F=opening bal, 86=memo)  
  - [ ] Map to BankStatementEntry schema: statement_date, entry_date, amount, counterparty, reference, transaction_code  
  - [ ] Validation: IBAN/BIC format, amount precision (2 decimals for EUR)  
  - [ ] Deduplication: skip entry if (statement_date, amount, reference) already exists in database  
  - [ ] Error handling: collect unparseable lines in error log, return partial success  
  - [ ] Test: Parse sample MT940 file (from SEPA test files), verify 5 entries imported correctly  
- [ ] **Controller:** Add POST /api/bank-reconciliation/import endpoint  
  - [ ] Parameters: bank_account_id, file (multipart), format ("mt940")  
  - [ ] Response: ImportResult { entries_imported, entries_matched, entries_unmatched, status }  
  - [ ] Auth: require "manage_treasury" role per ADR-001  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-001-001`  
- [ ] **Acceptance:** 
  - Requirement REQ-001-001-A: Import 5 valid MT940 entries successfully  
  - Requirement REQ-001-001-B: Handle malformed line with graceful error logging  
  - Requirement REQ-001-001-C: Detect and skip duplicate entries  

**Dependencies:** Task 4, Task 5  
**Blockers:** None  
**Effort:** 8 hours

---

### Task 11: Implement bank entry matching algorithm (exact, fuzzy, configurable)
- [ ] **Backend:** Create BankEntryMatchingService  
  - [ ] Method: `findMatches(BankStatementEntry $entry, array $ledgerEntries = []): array<Match>`  
  - [ ] Strategy 1 (Exact): amount + date + reference → confidence 0.99  
  - [ ] Strategy 2 (Fuzzy amount): amount ±5% + date ±3 days → confidence 0.7  
  - [ ] Strategy 3 (Fuzzy counterparty): counterparty fuzzy match + amount + date ±5 → confidence 0.5  
  - [ ] Apply user-configured rules (defer to Task 20 for custom rules)  
  - [ ] Return: sorted by confidence descending  
  - [ ] Test: Verify each strategy with sample entries from design.md  
- [ ] **Service:** Store matched confidence score in BankStatementEntry.matched_confidence  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-001-002`  
- [ ] **Acceptance:**
  - Requirement REQ-001-002-A: Exact match with 99% confidence  
  - Requirement REQ-001-002-B: Fuzzy amount match with 70% confidence  
  - Requirement REQ-001-002-C: Learn from manual matches (defer to Task 13)  

**Dependencies:** Task 4, Task 10  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 12: Implement manual reconciliation API
- [ ] **Backend:** Create ReconciliationService.matchManual()  
  - [ ] Method: `matchManual(BankStatementEntry $entry, GeneralLedgerEntry $ledgerEntry): void`  
  - [ ] Logic: update BankStatementEntry.matched_ledger_entry_id, set is_reconciled = true  
  - [ ] Audit trail: log who matched, when, original vs. matched entry IDs  
  - [ ] Undo capability: add `unmatchReconciliation($entry)` to revert within 7 days  
- [ ] **Controller:** Add POST /api/bank-reconciliation/match-manual endpoint  
  - [ ] Parameters: bank_entry_id, ledger_entry_id  
  - [ ] Response: { matched: true, confidence: 1.0, bank_entry, ledger_entry }  
  - [ ] Test: Verify manual match, then undo, then re-match  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-001-003`  
- [ ] **Acceptance:**
  - Requirement REQ-001-003-A: Manual match via search dialog works  
  - Requirement REQ-001-003-C: Undo reconciliation within 7 days  

**Dependencies:** Task 11  
**Blockers:** None  
**Effort:** 4 hours

---

### Task 13: Implement reconciliation status dashboard and reporting
- [ ] **Backend:** Create ReconciliationReportService  
  - [ ] Method: `getReconciliationStatus(BankAccount $account): StatusReport`  
  - [ ] Calculate: total_entries, total_matched, match_rate, last_reconciled_date  
  - [ ] Unmatched breakdown: by amount range (0-100, 100-1k, 1k-10k, >10k), by age (0-3 days, 4-7 days, >7 days)  
  - [ ] Method: `getMatchRateTrend(BankAccount $account, int $days = 30): array<{ date, match_rate }>`  
- [ ] **Controller:** Add GET /api/bank-reconciliation/status endpoint  
  - [ ] Returns: array of status reports (one per account)  
  - [ ] Parameters: _page, _limit  
- [ ] **Controller:** Add GET /api/bank-reconciliation/entries?bank_account_id=...&is_reconciled=...  
  - [ ] Returns paginated list with reconciliation status per entry  
- [ ] **Export:** Add ReconciliationExportService to generate CSV  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-001-004`  
- [ ] **Acceptance:**
  - Requirement REQ-001-004-A: Status cards show match rates for all accounts  
  - Requirement REQ-001-004-B: Filter and drill-down to old unmatched entries  
  - Requirement REQ-001-004-C: Export CSV with reconciliation data  

**Dependencies:** Task 12  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 14: Implement bank reconciliation UI (list, match dialog, status dashboard)
- [ ] **Frontend:** Create pages/BankReconciliation/Index.vue  
  - [ ] Use CnIndexPage + useListView composable  
  - [ ] Columns: statement_date, amount, counterparty_name, reference, status (matched/unmatched), confidence badge  
  - [ ] Filter: by bank account, reconciliation status, date range, amount range  
  - [ ] Row action: "Match" button opens CnAdvancedFormDialog  
  - [ ] Bulk action: "Auto-Match All" with confirmation dialog  
- [ ] **Frontend:** Create pages/BankReconciliation/MatchDialog.vue (child component)  
  - [ ] Search field: ledger entry search (by date, amount, reference)  
  - [ ] Results: table of matching candidates with confidence score  
  - [ ] Confirm button: call matchManual API  
- [ ] **Frontend:** Create widgets/LiquidReservesWidget.vue  
  - [ ] Display: current balance, minimum required, shortfall, status (healthy/warning/critical)  
  - [ ] Color-coded: green (≥110% of min), yellow (100-110%), red (<100%)  
  - [ ] Status text: "Days of reserve: X"  
- [ ] **Frontend:** Create pages/BankReconciliation/StatusDashboard.vue  
  - [ ] Use CnDashboardPage  
  - [ ] Status cards per account: match rate, unmatched count, last reconciled  
  - [ ] Trend chart: 30-day match rate trend  
  - [ ] Export button: calls ReconciliationExportService  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-001-001 to req-001-004`  
- [ ] **Acceptance:**
  - UI loads in <2s  
  - Manual match works end-to-end  
  - Export generates valid CSV  

**Dependencies:** Task 13  
**Blockers:** None  
**Effort:** 12 hours

---

## Cash Flow Forecasting (REQ-002)

### Task 15: Implement cash flow forecast computation engine
- [ ] **Backend:** Create CashFlowForecastService  
  - [ ] Method: `generateForecast(Administration $admin, int $horizonDays = 30, Date $asOfDate = null): CashFlowForecast`  
  - [ ] Logic:
    - [ ] Fetch all outstanding invoices (AR) where due_date >= asOfDate AND due_date <= asOfDate + horizonDays  
    - [ ] Fetch all unpaid bills (AP) where due_date >= asOfDate AND due_date <= asOfDate + horizonDays  
    - [ ] Fetch scheduled payments (active, due within horizon)  
    - [ ] Aggregate by date: sum inflows (AR), sum outflows (AP + scheduled)  
    - [ ] Calculate running balance: opening_balance + inflows - outflows per day  
    - [ ] Apply adjustments (optional manual transactions passed via parameter)  
  - [ ] Method: `generateForecastWithAdjustments(Admin, int, Date, array $adjustments): CashFlowForecast`  
  - [ ] Assumptions: invoices due on due_date, 0 scheduling delays (configurable per Task 20)  
  - [ ] Test: Sample forecast from design.md matches hand-calculated projection  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-002-001`  
- [ ] **Acceptance:**
  - Requirement REQ-002-001-A: 30-day forecast with committed transactions  
  - Requirement REQ-002-001-B: Adjustments update forecast  
  - Requirement REQ-002-001-C: Forecast vs. actual variance <5% (verified in testing)  

**Dependencies:** None (uses existing Invoice, Bill, ScheduledPayment entities)  
**Blockers:** Invoice and Bill entities must have due_date and status fields  
**Effort:** 8 hours

---

### Task 16: Implement cash flow chart widget
- [ ] **Frontend:** Create widgets/CashFlowChartWidget.vue  
  - [ ] Chart library: ApexCharts  
  - [ ] Chart type: stacked area chart (inflows green, outflows red)  
  - [ ] X-axis: date (30/60/90 days)  
  - [ ] Y-axis: balance (EUR)  
  - [ ] Interactivity:
    - [ ] Hover to show exact amount  
    - [ ] Click date to drill down to transactions  
    - [ ] Horizon selector: buttons for 30/60/90 days  
  - [ ] Anomalies: highlight dates where closing balance < minimum reserve (red zone)  
  - [ ] Legend: Opening Balance, Inflows, Outflows, Closing Balance  
  - [ ] Responsiveness: works from 320px to 1920px width  
  - [ ] Print: CSS for print-friendly layout  
- [ ] **Frontend:** Create pages/CashFlow/Forecast.vue  
  - [ ] Use CnDashboardPage  
  - [ ] Widget: CashFlowChartWidget (with horizon selector)  
  - [ ] Assumptions panel: list of invoices/bills included, manual adjustments  
  - [ ] Drill-down modal: click date to see transactions  
  - [ ] Export button: PDF or CSV  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-002-002`  
- [ ] **Acceptance:**
  - Requirement REQ-002-002-A: Horizon selector updates chart  
  - Requirement REQ-002-002-B: Drill-down shows transactions for date  
  - Requirement REQ-002-002-C: Reserve threshold visualized  

**Dependencies:** Task 15  
**Blockers:** None  
**Effort:** 10 hours

---

## Loan Refinancing (REQ-003)

### Task 17: Implement loan refinancing eligibility and scoring logic
- [ ] **Backend:** Create LoanRefinancingService  
  - [ ] Method: `analyzeLoan(Loan $loan): RefinancingAnalysis`  
  - [ ] Logic:
    - [ ] Lookup market rate for loan term (configurable table per Task 20)  
    - [ ] Calculate rate delta: current_rate - market_rate  
    - [ ] Calculate annual savings: principal × rate_delta  
    - [ ] Calculate refinancing cost: 0.5% × principal (configurable)  
    - [ ] Calculate score: annual_savings / refinancing_cost  
    - [ ] is_eligible: score > 1.0  
  - [ ] Method: `getMarketRate(int $termMonths, string $riskProfile = 'standard'): float`  
  - [ ] Market rate table: configurable via IAppConfig (editable in settings)  
  - [ ] Test: Sample loan from design.md (100k, 5.5%, 5-year term) → score 2.6, eligible  
- [ ] **Service:** Implement computed field updating:
  - [ ] Add cron job or repair step to update all Loan.is_eligible_for_refinancing weekly  
  - [ ] Or: compute on-demand during API call  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-003-001`  
- [ ] **Acceptance:**
  - Requirement REQ-003-001-A: Eligible loan flagged with score and savings estimate  
  - Requirement REQ-003-001-B: Ineligible loans filtered  

**Dependencies:** None  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 18: Implement loan refinancing UI and reporting
- [ ] **Frontend:** Create pages/Loans/RefinancingAnalysis.vue  
  - [ ] Use CnIndexPage with Loan entity  
  - [ ] Columns: principal, current_rate, market_rate, rate_delta, annual_savings, refinancing_score, maturity_date  
  - [ ] Filter: "Eligible only" checkbox (filters score > 1.0)  
  - [ ] Sort: by refinancing_score descending (highest savings first)  
  - [ ] Row action: "View Options" opens detail modal with lender recommendations (placeholder for Phase 2)  
  - [ ] Export button: CSV with all analysis columns  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-003-001`  
- [ ] **Acceptance:**
  - Requirement REQ-003-001-A: Eligible loan displayed with savings  
  - Requirement REQ-003-001-B: Eligible filter works  
  - Requirement REQ-003-001-C: Export CSV includes all fields  

**Dependencies:** Task 17  
**Blockers:** None  
**Effort:** 6 hours

---

## IBAN Verification (REQ-004)

### Task 19: Implement IBAN validation and supplier verification workflow
- [ ] **Backend:** Create IBANValidationService  
  - [ ] Method: `validateIBAN(string $iban): ValidationResult { isValid, isSepa, country, bankName }`  
  - [ ] Library: use IBAN.js or similar for checksum validation  
  - [ ] Validation: ISO 13616 checksum + length per country + country code lookup  
  - [ ] SEPA eligibility: check if country in SEPA zone (EU27 + Iceland, Liechtenstein, Norway, Switzerland)  
  - [ ] Test: Valid (NL75ABNA0588564432) and invalid (NL75ABNA058856443X) IBANs  
- [ ] **Backend:** Create SupplierIBANVerificationService  
  - [ ] Method: `verifySupplierIBAN(Supplier $supplier, string $iban, User $verifier): void`  
  - [ ] Logic: update supplier.iban_verified = true, iban_verified_date = now, iban_verified_by = $verifier  
  - [ ] Consistency check: compare to supplier's previous payment IBANs (last 3)  
  - [ ] Flag change: if IBAN differs from all previous, set flag for manual review  
  - [ ] Audit trail: log verification with timestamp + verifier  
  - [ ] Method: `getVerificationHistory(Supplier $supplier): array<VerificationRecord>`  
- [ ] **Controller:** Add POST /api/suppliers/verify-iban endpoint  
  - [ ] Parameters: supplier_id, iban  
  - [ ] Response: ValidationResult  
- [ ] **Controller:** Add GET /api/suppliers/{id}/iban-verification-history endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-004-001`  
- [ ] **Acceptance:**
  - Requirement REQ-004-001-A: Valid IBAN passes validation  
  - Requirement REQ-004-001-B: Invalid IBAN rejected  
  - Requirement REQ-004-001-C: IBAN change detection flags for review  
  - Requirement REQ-004-001-D: Verification history visible  

**Dependencies:** Task 7 (Supplier schema updates)  
**Blockers:** iban.js library integration (add to composer.json)  
**Effort:** 6 hours

---

### Task 20: Implement supplier IBAN verification UI in detail page
- [ ] **Frontend:** Update Supplier detail page (pages/Suppliers/Detail.vue or similar)  
  - [ ] Add IBAN verification section in sidebar or tab  
  - [ ] Current IBAN display with validation status badge  
  - [ ] "Verify" button: calls verifySupplierIBAN API  
  - [ ] Result display: shows isValid, bankName, isSepa status  
  - [ ] If change detected: warning modal "IBAN differs from last 3 payments"  
  - [ ] Verification history timeline: dates, IBANs, verifier name  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-004-001`  
- [ ] **Acceptance:**
  - UI integrates with detail page  
  - Verification modal works end-to-end  

**Dependencies:** Task 19  
**Blockers:** None  
**Effort:** 6 hours

---

## Payment Netting (REQ-005)

### Task 21: Implement netting analysis algorithm
- [ ] **Backend:** Create PaymentNettingService  
  - [ ] Method: `analyzeNetting(Administration $admin, Supplier $supplier = null, int $horizonDays = 90): array<NettingOpportunity>`  
  - [ ] Logic:
    - [ ] Fetch payables to supplier (AP): amount, due_date, status = "outstanding"  
    - [ ] Fetch receivables from supplier (AR): amount, due_date, status = "outstanding"  
    - [ ] Group by supplier (if $supplier = null) or single supplier  
    - [ ] Within horizon, match payables to receivables (FIFO by due date)  
    - [ ] Calculate offset: min(payable, receivable)  
    - [ ] Calculate net payment: payable - receivable  
    - [ ] Savings estimate: offset × SEPA_FEE_RATE + offset × INTEREST_RATE × days / 365  
  - [ ] Return: sorted by potential_savings descending  
  - [ ] Test: Sample netting opportunity from design.md  
- [ ] **Controller:** Add GET /api/payments/netting-analysis?administration_id=...&supplier_id=... endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-005-001`  
- [ ] **Acceptance:**
  - Requirement REQ-005-001-A: Netting opportunity identified with savings estimate  
  - Requirement REQ-005-001-B: Execution creates contra-journal entry  

**Dependencies:** None (uses existing Payment entity)  
**Blockers:** Payment entity must support netting_group_id (Task 6)  
**Effort:** 6 hours

---

### Task 22: Implement netting execution logic
- [ ] **Backend:** Create PaymentNettingExecutionService  
  - [ ] Method: `executeNetting(array $nettingGroupIds): array<SettlementRecord>`  
  - [ ] Logic:
    - [ ] For each netting group: fetch payable + receivable payments  
    - [ ] Create journal entry: debit AR, credit AP (offset amount)  
    - [ ] Update both payments: status = "settled_via_netting", netting_group_id set  
    - [ ] Link payments: store cross-reference  
    - [ ] Audit log: record execution with timestamp, amounts, user  
  - [ ] Error handling: if journal entry creation fails, rollback and report error  
  - [ ] Test: Execute sample netting from design.md, verify journal entries and payment status  
- [ ] **Controller:** Add POST /api/payments/execute-netting endpoint  
  - [ ] Parameters: netting_group_ids array  
  - [ ] Response: array of SettlementRecords with status  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-005-001`  
- [ ] **Acceptance:**
  - Requirement REQ-005-001-B: Execute netting creates contra-journal entry and marks settled  

**Dependencies:** Task 21  
**Blockers:** None  
**Effort:** 4 hours

---

### Task 23: Implement netting UI
- [ ] **Frontend:** Create pages/Payments/NettingAnalysis.vue  
  - [ ] Use CnIndexPage displaying NettingOpportunity list  
  - [ ] Columns: supplier, payable_amount, receivable_amount, net_payment, potential_savings, currency  
  - [ ] Filter: by supplier, currency, min_savings threshold  
  - [ ] Row action: "Execute" button with confirmation modal  
  - [ ] Bulk action: "Execute All" (with validation)  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-005-001`  
- [ ] **Acceptance:**
  - UI loads netting opportunities  
  - Execute action works end-to-end  

**Dependencies:** Task 22  
**Blockers:** None  
**Effort:** 5 hours

---

## Payment Batching & Separation of Duties (REQ-006, REQ-007)

### Task 24: Implement payment batch creation and management service
- [ ] **Backend:** Create PaymentBatchService  
  - [ ] Method: `createBatch(Administration $admin, array $paymentIds, string $notes = ''): PaymentBatch`  
  - [ ] Logic: group payments into batch, calculate total_amount, set status = "draft", record created_by + created_date  
  - [ ] Method: `addPaymentsToBatch(PaymentBatch $batch, array $paymentIds): void`  
  - [ ] Method: `removePaymentsFromBatch(PaymentBatch $batch, array $paymentIds): void`  
  - [ ] Method: `submitBatchForApproval(PaymentBatch $batch): void`  
    - [ ] Update status = "approved_pending" or similar  
    - [ ] Assign to approver role via approval_chain_id  
  - [ ] Method: `executeBatch(PaymentBatch $batch): ExecutionResult`  
    - [ ] Initiate payments to each provider  
    - [ ] Track execution status per payment  
    - [ ] Handle partial failures (if 1 payment fails, mark batch "partial_execution")  
- [ ] **Controller:** Add POST /api/payments/create-batch endpoint  
  - [ ] Parameters: administration_id, payment_ids, notes  
  - [ ] Response: PaymentBatch  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-006-001`  
- [ ] **Acceptance:**
  - Requirement REQ-006-001-A: Batch created from pending payments  
  - Requirement REQ-006-001-B: Batch submitted for approval  
  - Requirement REQ-006-001-C: Batch executed to providers  

**Dependencies:** Task 2 (PaymentProvider), Task 1 (PaymentBatch schema)  
**Blockers:** None  
**Effort:** 8 hours

---

### Task 25: Implement separation-of-duties detection and reporting service
- [ ] **Backend:** Create SeparationOfDutiesService  
  - [ ] Method: `detectViolations(Payment $payment): array<Violation>`  
  - [ ] Rules (per ADR-023 or similar):
    - [ ] `requester_cannot_be_approver`: payment.requester_id ≠ approval.approver_id  
    - [ ] `approver_cannot_be_payer`: approval.approver_id ≠ payment.executed_by_id  
    - [ ] `requester_cannot_be_payer`: payment.requester_id ≠ payment.executed_by_id  
  - [ ] Severity calculation: count rules violated → "critical" (3), "high" (2), "medium" (1)  
  - [ ] Log violations immediately  
  - [ ] Method: `getViolationReport(Administration $admin, Date $dateFrom = null, Date $dateTo = null): ViolationReport`  
  - [ ] Method: `markViolationReviewed(Violation $violation, string $comment): void`  
  - [ ] Audit trail: log who reviewed, when, with comment  
- [ ] **Controller:** Add GET /api/compliance/separation-of-duties-violations endpoint  
  - [ ] Parameters: administration_id, date_from, date_to, _page, _limit  
  - [ ] Response: paginated violation list  
- [ ] **Controller:** Add POST /api/compliance/mark-violation-reviewed endpoint  
  - [ ] Parameters: violation_id, comment  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-007-001`  
- [ ] **Acceptance:**
  - Requirement REQ-007-001-A: Critical violation detected  
  - Requirement REQ-007-001-B: High-severity violation detected  
  - Requirement REQ-007-001-C: Violations report with filtering  
  - Requirement REQ-007-001-D: Mark violation as reviewed  

**Dependencies:** Task 24 (for context)  
**Blockers:** Approval chain and payment executor fields must be populated  
**Effort:** 6 hours

---

### Task 26: Implement separation-of-duties configuration UI
- [ ] **Backend:** Create SeparationOfDutiesConfigService  
  - [ ] Method: `setSeparationRules(Administration $admin, array $rules): void`  
  - [ ] Stores rule preferences (enable/disable per rule)  
- [ ] **Controller:** Add POST /api/compliance/set-separation-of-duties-rules endpoint  
  - [ ] Parameters: administration_id, rules array  
- [ ] **Frontend:** Create pages/Settings/ComplianceSettings.vue  
  - [ ] Section: "Separation of Duties"  
  - [ ] Checkboxes for each rule: requester_cannot_be_approver, etc.  
  - [ ] Save button calls API  
  - [ ] Confirmation: "Rules updated"  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-007-001`  
- [ ] **Acceptance:**
  - Settings page allows rule configuration  
  - Rules are persisted and applied  

**Dependencies:** Task 25  
**Blockers:** None  
**Effort:** 4 hours

---

### Task 27: Implement payment batching and approval workflow UI
- [ ] **Frontend:** Create pages/Payments/BatchManager.vue  
  - [ ] List of batches: status, created_by, created_date, total_amount, payment_count  
  - [ ] Filter: by status (draft/approved/executed/cancelled)  
  - [ ] Row action: "View Details" opens detail page  
- [ ] **Frontend:** Create pages/Payments/BatchDetail.vue  
  - [ ] Header: batch ID, status, total amount, created info  
  - [ ] Payments table: list of payments in batch, with individual status  
  - [ ] Actions (conditional on status):
    - [ ] "Add Payment" button (draft) opens search dialog  
    - [ ] "Remove Payment" button (draft)  
    - [ ] "Submit for Approval" button (draft) → calls submitBatchForApproval API  
    - [ ] "Approve" button (approver role, awaiting_approval status)  
    - [ ] "Execute" button (approved status) → calls executeBatch API  
  - [ ] Audit trail tab: show approval history and execution logs  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-006-001`  
- [ ] **Acceptance:**
  - Batch creation and approval workflow works end-to-end  
  - UI responsive and accessible  

**Dependencies:** Task 24  
**Blockers:** None  
**Effort:** 8 hours

---

## Treasury Administration (REQ-008)

### Task 28: Implement payment provider configuration and management service
- [ ] **Backend:** Create PaymentProviderService  
  - [ ] Method: `createProvider(Administration $admin, array $data): PaymentProvider`  
  - [ ] Method: `testProviderConnectivity(PaymentProvider $provider): ConnectivityResult`  
    - [ ] Attempt API authentication  
    - [ ] Return: { success: bool, message: string }  
  - [ ] Method: `getProviderForPayment(Payment $payment): PaymentProvider`  
    - [ ] Match on currency + payment method  
    - [ ] Return best provider (by priority order)  
  - [ ] Encryption: use IAppConfig::setSecureValue() for API keys  
- [ ] **Controller:** Add GET /api/payment-providers endpoint  
  - [ ] Parameters: administration_id  
- [ ] **Controller:** Add POST /api/payment-providers endpoint  
  - [ ] Create new provider with encrypted key storage  
- [ ] **Controller:** Add POST /api/payment-providers/{id}/test-connectivity endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-001`  
- [ ] **Acceptance:**
  - Provider created and API key encrypted  
  - Connectivity test successful  
  - Payment routed to correct provider by currency  

**Dependencies:** Task 2 (PaymentProvider schema)  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 29: Implement payment provider configuration UI
- [ ] **Frontend:** Create pages/Settings/PaymentProvidersSettings.vue  
  - [ ] List of providers: name, type, supported_currencies, is_active  
  - [ ] Add button: opens form to create new provider  
  - [ ] Row actions: Edit, Test Connectivity, Delete  
  - [ ] Edit form fields:
    - [ ] Provider name (dropdown or text)  
    - [ ] Type (dropdown: bank/fintech/internal)  
    - [ ] Supported currencies (multi-select)  
    - [ ] API endpoint (conditional on type)  
    - [ ] API key (encrypted input)  
  - [ ] Test Connectivity button: shows result message  
  - [ ] Save/Cancel buttons  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-001`  
- [ ] **Acceptance:**
  - Provider settings page functional  
  - Encryption working (key not visible in logs)  

**Dependencies:** Task 28  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 30: Implement cash pooling configuration and execution service
- [ ] **Backend:** Create CashPoolingService  
  - [ ] Method: `configureCashPooling(Administration $admin, array $config): CashPoolingConfiguration`  
    - [ ] Store: primary_account_id, pooled_account_ids, transfer_frequency, transfer_time, minimum_balance_per_account  
  - [ ] Method: `executePooling(Administration $admin): PoolingResult`  
    - [ ] For each pooled account:
      - [ ] Fetch balance  
      - [ ] Calculate transfer: balance - minimum_balance  
      - [ ] Initiate SEPA transfer to primary account  
      - [ ] Log transfer with status  
    - [ ] Return: array of transfers initiated  
  - [ ] Method: `getPoolingConfiguration(Administration $admin): CashPoolingConfiguration`  
- [ ] **Controller:** Add POST /api/cash-pooling/configure endpoint  
- [ ] **Controller:** Add POST /api/cash-pooling/execute endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-003`  
- [ ] **Acceptance:**
  - Requirement REQ-008-003-A: Cash pooling configured  
  - Requirement REQ-008-003-B: Manual pooling execution works  
  - Requirement REQ-008-003-C: Scheduled daily pooling (via background job)  

**Dependencies:** None  
**Blockers:** None  
**Effort:** 6 hours

---

### Task 31: Implement cash pooling configuration UI
- [ ] **Frontend:** Create pages/Settings/CashPoolingSettings.vue  
  - [ ] Form fields:
    - [ ] Primary account (dropdown: list of BankAccounts)  
    - [ ] Pooled accounts (multi-select)  
    - [ ] Transfer frequency (dropdown: daily/weekly/monthly)  
    - [ ] Transfer time (time picker)  
    - [ ] Minimum balance per account (currency input)  
  - [ ] "Save Configuration" button  
  - [ ] "Execute Pool Now" button (triggers immediate pooling)  
  - [ ] Result message: "2 transfers initiated, EUR 57,000 total"  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-003`  
- [ ] **Acceptance:**
  - Settings page saves configuration  
  - Manual execution works  

**Dependencies:** Task 30  
**Blockers:** None  
**Effort:** 4 hours

---

### Task 32: Implement liquid reserve monitoring service
- [ ] **Backend:** Create LiquidReserveService  
  - [ ] Method: `getLiquidReserveStatus(Administration $admin): ReserveStatus`  
    - [ ] Fetch current_balance (sum of all active bank accounts)  
    - [ ] Fetch minimum_required (from BankAccount.reserve_minimum)  
    - [ ] Calculate shortfall: min(0, current_balance - minimum_required)  
    - [ ] Calculate days_of_reserve: current_balance / (avg_daily_burn_rate)  
    - [ ] Status: "healthy" (>110% of min), "warning" (100-110%), "critical" (<100%)  
  - [ ] Method: `setReserveRequirements(Administration $admin, MonetaryAmount $minimum, int $alertThreshold): void`  
  - [ ] Method: `checkAndAlertReserves(Administration $admin): void`  
    - [ ] If below threshold, send notification to CFO/treasurer  
- [ ] **Controller:** Add GET /api/treasury/liquid-reserves endpoint  
- [ ] **Controller:** Add POST /api/treasury/set-reserve-requirements endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-002`  
- [ ] **Acceptance:**
  - Requirement REQ-008-002-A: Alert triggered when reserves fall below minimum  
  - Requirement REQ-008-002-B: Reserve dashboard widget shows status  

**Dependencies:** Task 5 (BankAccount.reserve_minimum)  
**Blockers:** None  
**Effort:** 4 hours

---

### Task 33: Implement liquid reserve monitoring UI
- [ ] **Frontend:** Update Dashboard with CnStatsBlock widget or card  
  - [ ] Widget displays:
    - [ ] Current balance (large number, color-coded)  
    - [ ] Minimum required  
    - [ ] Shortfall (if any)  
    - [ ] Status: "Healthy" / "Warning" / "Critical"  
    - [ ] Days of reserve (calculated)  
  - [ ] Click action: navigate to detailed reserve report  
- [ ] **Frontend:** Create pages/Treasury/ReserveReport.vue  
  - [ ] 90-day historical trend (CSV export)  
  - [ ] Current vs. minimum chart  
  - [ ] Recommended actions (if critical)  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-008-002`  
- [ ] **Acceptance:**
  - Reserve status displayed on dashboard  
  - Report page functional  

**Dependencies:** Task 32  
**Blockers:** None  
**Effort:** 6 hours

---

## Bank Fee Analysis (REQ-009)

### Task 34: Implement bank fee analysis and reporting service
- [ ] **Backend:** Create BankFeeAnalysisService  
  - [ ] Method: `analyzeFees(BankAccount $account, Date $dateFrom, Date $dateTo): FeeReport`  
  - [ ] Logic:
    - [ ] Fetch all BankStatementEntries in date range  
    - [ ] Parse transaction_code to classify type (N005=SEPA, N044=wire, etc.)  
    - [ ] Extract fee amounts from statement (separate fee entries)  
    - [ ] Group by type and aggregate  
    - [ ] Calculate percentage of transaction amount  
  - [ ] Method: `getFeeOpportunities(BankAccount $account): array<Opportunity>`  
    - [ ] Identify above-average fees per transaction type  
    - [ ] Suggest actions (e.g., "Use SEPA instead of wire to save 50%")  
  - [ ] Test: Sample bank statement with various fees  
- [ ] **Controller:** Add GET /api/bank-fees/analysis endpoint  
  - [ ] Parameters: bank_account_id, date_from, date_to  
  - [ ] Response: FeeReport with breakdown and trend  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-009-001`  
- [ ] **Acceptance:**
  - Requirement REQ-009-001-A: Fee breakdown by transaction type  
  - Requirement REQ-009-001-B: Trend analysis and recommendations  
  - Requirement REQ-009-001-C: Export for negotiation  

**Dependencies:** Task 10 (BankStatementEntry import)  
**Blockers:** Transaction code mapping must be accurate  
**Effort:** 6 hours

---

### Task 35: Implement bank fee analysis UI
- [ ] **Frontend:** Create pages/BankFees/Analysis.vue  
  - [ ] Date range picker: from/to  
  - [ ] Chart: pie chart of fees by transaction type  
  - [ ] Table: detailed breakdown (date, type, fee, % of amount)  
  - [ ] Trend section: month-over-month comparison  
  - [ ] Recommendations: list of optimization suggestions  
  - [ ] Export button: "Export for Negotiation" → CSV  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-009-001`  
- [ ] **Acceptance:**
  - Fee analysis page loads and displays data  
  - Export generates CSV  

**Dependencies:** Task 34  
**Blockers:** None  
**Effort:** 6 hours

---

## Payment Arrangements Portfolio (REQ-010)

### Task 36: Implement payment arrangements portfolio service and reporting
- [ ] **Backend:** Create PaymentArrangementsService  
  - [ ] Method: `getActiveArrangements(Administration $admin, Date $asOfDate = null): array<Arrangement>`  
  - [ ] Sources: ScheduledPayment + active invoices on installment plans + settlement agreements  
  - [ ] Fields: type, amount, frequency, next_due_date, counterparty, status  
  - [ ] Sorting: by next_due_date ascending (overdue first)  
  - [ ] Status calculation: overdue (next_due < today), due (next_due = today), pending (next_due > today)  
  - [ ] Method: `deferPayment(Arrangement $arr, Date $newDueDate): void`  
  - [ ] Method: `pauseArrangement(Arrangement $arr): void`  
  - [ ] Method: `resumeArrangement(Arrangement $arr): void`  
  - [ ] Audit trail: log all modifications  
- [ ] **Controller:** Add GET /api/payment-arrangements endpoint  
  - [ ] Parameters: administration_id, status filter  
- [ ] **Controller:** Add POST /api/payment-arrangements/{id}/defer endpoint  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-010-001`  
- [ ] **Acceptance:**
  - Requirement REQ-010-001-A: View all active arrangements  
  - Requirement REQ-010-001-B: Overdue arrangements highlighted  
  - Requirement REQ-010-001-C: Defer payment works  

**Dependencies:** None (uses existing entities)  
**Blockers:** ScheduledPayment and settlement agreement entities must have due_date field  
**Effort:** 4 hours

---

### Task 37: Implement payment arrangements portfolio UI
- [ ] **Frontend:** Create pages/PaymentArrangements/Portfolio.vue  
  - [ ] Table columns: type, amount, frequency, next_due_date, counterparty, status  
  - [ ] Conditional row styling: overdue (red), due today (orange), pending (normal)  
  - [ ] Sorting: by next_due_date ascending  
  - [ ] Filter: by status (all/overdue/due_today/pending)  
  - [ ] Row actions: Defer, Pause, Resume, View Details  
  - [ ] Defer dialog: date picker + confirmation  
  - [ ] Mass action: select multiple, defer all (batch)  
- [ ] **Spec Traceability:** `@spec openspec/changes/treasury-cash-management-other-t4/specs.md#req-010-001`  
- [ ] **Acceptance:**
  - Portfolio page displays arrangements  
  - Defer action works  

**Dependencies:** Task 36  
**Blockers:** None  
**Effort:** 6 hours

---

## Integration & Testing

### Task 38: Create API integration test suite (Newman/Postman)
- [ ] **Testing:** Generate Postman collection covering all 12 features  
  - [ ] Bank reconciliation: import MT940, match entries, reconciliation status  
  - [ ] Cash flow: generate forecast, adjust, export  
  - [ ] Loans: get refinancing analysis  
  - [ ] IBAN verification: validate, check history  
  - [ ] Netting: analyze and execute  
  - [ ] Payment batching: create, approve, execute  
  - [ ] Separation of duties: detect violations, get report  
  - [ ] Providers: configure, test connectivity  
  - [ ] Cash pooling: configure, execute  
  - [ ] Reserves: get status, set requirements  
  - [ ] Bank fees: analyze  
  - [ ] Payment arrangements: list, defer  
- [ ] **Tests per endpoint:**
  - [ ] Successful request (HTTP 200/201)  
  - [ ] Invalid parameters (HTTP 400)  
  - [ ] Unauthorized (HTTP 403 if applicable)  
  - [ ] Schema validation (response body matches spec)  
- [ ] **Run:** Execute via `newman run` in CI/CD  
- [ ] **Acceptance:** All tests pass, coverage >90% of endpoints  

**Dependencies:** All backend tasks (Task 10-37)  
**Blockers:** None  
**Effort:** 10 hours

---

### Task 39: Create end-to-end browser tests (manual or Playwright)
- [ ] **Testing:** Manual test scenarios for each feature (or automate with Playwright)  
  - [ ] Bank reconciliation: import file, view status, match entry  
  - [ ] Cash flow: view chart, adjust, export  
  - [ ] Loans: view eligible loans, export  
  - [ ] IBAN: verify supplier IBAN  
  - [ ] Netting: identify and execute  
  - [ ] Batching: create, submit, approve, execute  
  - [ ] SOD: view violations, mark reviewed  
  - [ ] Providers: add, test, route payment  
  - [ ] Pooling: configure, execute  
  - [ ] Reserves: set minimum, view alert  
  - [ ] Fees: analyze and export  
  - [ ] Arrangements: view, defer  
- [ ] **Environment:** Nextcloud v29+ with test data from design.md  
- [ ] **Browser:** Chrome, Firefox  
- [ ] **Acceptance:** All features work end-to-end, accessible per WCAG AA  

**Dependencies:** Task 14-37 (UI tasks)  
**Blockers:** None  
**Effort:** 15 hours

---

### Task 40: Documentation and release preparation
- [ ] **Docs:** Create user documentation in `docs/` directory  
  - [ ] Bank reconciliation how-to  
  - [ ] Cash flow forecasting guide  
  - [ ] Loan refinancing tutorial  
  - [ ] IBAN verification workflow  
  - [ ] Payment netting guide  
  - [ ] Treasury settings (providers, pooling, reserves)  
  - [ ] Compliance & separation of duties audit guide  
  - [ ] Screenshots from running app  
- [ ] **Translations:** Dutch (nl) + English (en)  
  - [ ] All user-visible strings in frontend via `t(appName, 'key')`  
  - [ ] Translation keys in `l10n/en.js` and `l10n/nl.js`  
- [ ] **Release notes:** Summarize 12 new features, breaking changes (none expected), migration steps  
- [ ] **ADR annotation:** Ensure all PHP classes and Vue components have `@spec` tags  
- [ ] **Acceptance:** Documentation complete, translations done, release notes ready  

**Dependencies:** All tasks  
**Blockers:** None  
**Effort:** 12 hours

---

## Summary

**Total Effort:** 120 hours  
**Team:** 1 backend (60h), 1 frontend (40h), 1 QA (20h)  
**Duration:** 4 weeks (assuming 30h/week available)  

### Deliverables
1. ✓ proposal.md (this change overview)  
2. ✓ design.md (data model, API, UI patterns, seed data)  
3. ✓ specs.md (detailed requirements with test scenarios REQ-XXX-NNN)  
4. ✓ tasks.md (implementation tasks with spec traceability)  

### Acceptance Criteria for Phase 1
- [ ] All 40 tasks completed (marked ✓)  
- [ ] API integration tests pass (Newman: 90%+ endpoint coverage)  
- [ ] Browser tests pass (manual: all 12 features verified)  
- [ ] Documentation complete (user guides + screenshots)  
- [ ] Translations done (en + nl)  
- [ ] Code review passed (all classes tagged with @spec)  
- [ ] CI/CD pipeline green (unit tests, linting, security checks)  

---

**Tasks Version:** 1.0  
**Last Updated:** 2026-05-21  
**Prepared by:** Specter (Intelligence/Research)
