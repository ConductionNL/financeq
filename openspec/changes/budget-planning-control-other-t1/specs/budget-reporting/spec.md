# Spec: Budget Reporting

**Capability:** `budget-reporting`
**Change:** budget-planning-control-other-t1
**Status:** in-progress

## Overview

Management reporting with comparative period and budget variance analysis. Covers budget vs actuals comparison reporting, management reporting with period comparison and budget analysis, budget utilisation reporting with variance analysis and cost-saving identification, balance sheet with comparative period and budget variance analysis, and monitoring budget execution.

**Entities:** Budget, BudgetAllocation, BudgetPeriod, FiscalYear, GeneralLedgerEntry, CostCenter, CostProject

## Requirements

### REQ-REP-001 — Budget vs actuals comparison reporting

**As a** finance manager or controller,
**I want to** compare approved budget amounts against actual spend and committed amounts for any budget period,
**so that** variance is quantified and actionable.

**Acceptance Criteria:**

**Scenario 1: Generate budget vs actuals report**
- GIVEN budgets and expenditure records exist for a fiscal year
- WHEN the controller generates a budget vs actuals report for a selected BudgetPeriod
- THEN the report shows per cost center: budget approved, actual spent, committed, remaining, variance (amount), and variance (%)
- AND totals roll up to department and organizational level

**Scenario 2: Drill down from variance to underlying transactions**
- GIVEN a budget vs actuals report shows a significant variance for a cost center
- WHEN the controller clicks on the variance figure
- THEN a filtered list of ExpenditureRequest and PurchaseRequisition objects for that cost center and period is shown

**Scenario 3: Export the report**
- GIVEN the budget vs actuals report is displayed
- WHEN the controller exports it
- THEN the export contains all columns including hierarchy, period, and variance
- AND is generated via `ExportService` in CSV or Excel format

---

### REQ-REP-002 — Management reporting with comparative period analysis

**As a** CFO or management controller,
**I want to** produce management reports comparing current period performance against the prior year and against the approved budget,
**so that** board and council can make informed financial decisions.

**Acceptance Criteria:**

**Scenario 1: Comparative period management report**
- GIVEN budgets and actuals exist for the current and previous fiscal year
- WHEN the controller generates a management report selecting current and comparison periods
- THEN `BudgetReportService.getVarianceReport()` returns structured data showing: current year budget, current year actual, prior year actual, YoY change (€ and %), and budget deviation (€ and %)
- AND the report is displayed via `CnTableWidget` with sortable columns

**Scenario 2: Budget analysis by department**
- GIVEN the management report is displayed
- WHEN the controller applies a department filter using `CnFilterBar`
- THEN the report re-renders for the selected department hierarchy
- AND charts update to reflect the filtered scope

**Scenario 3: Monitor budget execution over the period**
- GIVEN the fiscal year is in progress
- WHEN the controller opens the budget execution monitor
- THEN a line chart (`CnChartWidget`) shows cumulative actual spend versus the pro-rata budget plan month by month
- AND the projection line extends to year-end based on the current spend rate

---

### REQ-REP-003 — Budget utilisation reporting with variance analysis

**As a** finance manager,
**I want to** identify cost centres where spending is significantly above or below budget,
**so that** I can reallocate budgets or identify cost-saving opportunities.

**Acceptance Criteria:**

**Scenario 1: Variance analysis report**
- GIVEN budget allocations and actual spend data exist
- WHEN the finance manager opens the utilisation report for a period
- THEN `BudgetReportService.getUtilisationReport()` returns all BudgetAllocations sorted by variance (largest underspend and overspend first)
- AND each row shows: allocation name, budget, actual, variance, utilisation %, and a `CnProgressBar` visualization

**Scenario 2: Identify cost-saving opportunities**
- GIVEN the utilisation report shows allocations with < 50% utilisation at mid-year
- WHEN the controller filters for low-utilisation allocations
- THEN allocations with underspend > 30% are highlighted as potential reallocation candidates
- AND the total re-allocatable amount is shown in a summary KPI

**Scenario 3: Export variance analysis for board presentation**
- GIVEN the variance analysis report is complete
- WHEN the controller exports it
- THEN the export includes all variance columns and the summary section
- AND the file is in Excel format suitable for board presentation

---

### REQ-REP-004 — Balance sheet with comparative period and budget variance

**As a** CFO,
**I want to** view the balance sheet with both a prior-period comparison and a budget variance column,
**so that** the financial position is presented in the context of plan and trend.

**Acceptance Criteria:**

**Scenario 1: Balance sheet with three-column layout**
- GIVEN the balance sheet report (BalanceSheet entity) is generated for the current period
- WHEN the CFO requests the comparative view
- THEN the balance sheet shows three value columns: current period actual, prior period actual, and approved budget
- AND variance columns (€ and %) are computed for both current vs. prior and current vs. budget

**Scenario 2: Drill-down from balance sheet line to GL entries**
- GIVEN the balance sheet shows a variance on a specific account
- WHEN the CFO clicks on the account line
- THEN the underlying GeneralLedgerEntry objects for that account and period are shown in a filtered list

**Scenario 3: Accessibility of comparative report**
- GIVEN the balance sheet comparative report is displayed
- WHEN a keyboard user navigates the report table
- THEN all cells are reachable via keyboard (Tab/arrow keys)
- AND variance values use text indicators (e.g. "▲ +€12.400" and "▼ -€3.200") in addition to color, so color is not the sole conveyor of meaning (WCAG AA per ADR-010)
