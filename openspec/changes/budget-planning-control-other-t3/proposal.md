---
title: Budget Planning & Control — Shillinq — Other T3
change: budget-planning-control-other-t3
app: shillinq
kind: code
status: proposed
created: 2026-05-20
depends_on: []
chain: []
---

# Budget Planning & Control — Shillinq — Other T3

## Summary

This change adds two budget planning features to Shillinq: **Consolidate budgets** and **Set savings goals**. Together they allow financial managers and controllers to aggregate budgets across cost centers, departments, or legal entities into a unified view, and to define and track savings targets alongside that consolidated position.

Both features rely entirely on existing Shillinq entities — ConsolidationGroup, ConsolidatedReport, SavingsOpportunity, Budget, BudgetPeriod, FiscalYear — and on the OpenRegister + @conduction/nextcloud-vue platform. No new schemas are introduced.

## Features

### 1. Consolidate budgets
**Demand:** unknown | **Category:** other | **Cluster:** 2 user stories

Financial managers need to consolidate multiple individual budgets — from different cost centers, departments, or legal entities — into a single consolidated overview. The consolidated view must show total budget, total spend, and variance in one place, grouped by ConsolidationGroup and filtered by FiscalYear.

Scope:
- ConsolidationGroup list (`CnIndexPage`) and detail page (`CnDetailPage`) for defining which Budgets are grouped together
- ConsolidatedReport list and detail view showing aggregated totals and variance
- Filters by FiscalYear, status, and CostCenter
- Dashboard widget: consolidated budget KPI card (total budget vs. total spend)

### 2. Set savings goals
**Demand:** unknown | **Category:** other | **Cluster:** 1 user story

Controllers and financial managers need to define savings goals linked to specific Budgets or BudgetPeriods. Each goal has a target amount, a current achievement amount, and a deadline. Progress is visualized on the dashboard.

Scope:
- SavingsOpportunity list and detail view for defining and tracking savings goals
- Dashboard KPI widget showing active goals, progress percentage, and achieved vs. target amounts
- Status lifecycle: `active` → `achieved` / `cancelled`

## Stakeholders

### Financieel Beheerder (Financial Manager)
**Responsibilities:** Manages budgets, allocations, and amendments; produces financial reports for management and external parties.
**Goals:** Obtain a real-time consolidated view of the organization's financial position across all cost centers; monitor savings progress without leaving Shillinq.
**Pain points:** Currently aggregates budget data manually from multiple spreadsheets; consolidated reports are produced infrequently and by hand.

### Controller
**Responsibilities:** Sets financial targets, performs variance analysis, ensures budget compliance across departments.
**Goals:** Define savings goals at the start of each fiscal year; track weekly progress; flag overruns early enough to take corrective action.
**Pain points:** No structured savings goal tracking in the system; consolidation is done ad-hoc in Excel; no audit trail on who set which target.

### Directeur (Director/Executive)
**Responsibilities:** Strategic oversight; final approval on major budget amendments and investment decisions.
**Goals:** View the consolidated budget position and savings progress in a single dashboard without drilling into individual cost center budgets.
**Pain points:** Consolidated reports are produced manually and shared as PDFs; no live drill-down from the consolidated view into individual budgets.

## User Stories

### Feature: Consolidate budgets

**US-CON-001** — As a Financieel Beheerder, I want to create a ConsolidationGroup that contains multiple Budgets, so that I can produce a consolidated budget report across all selected budgets.

*Acceptance criteria:*
- GIVEN I am on the ConsolidationGroup detail page  
  WHEN I add one or more Budgets to the group  
  THEN the linked Budgets appear in the Budgets section of the detail view and are stored via the OpenRegister relation mechanism
- GIVEN a ConsolidationGroup has at least one linked Budget  
  WHEN I request a ConsolidatedReport for that group  
  THEN the report aggregates totalBudget, totalSpend, and variance across all linked Budgets for the selected FiscalYear

**US-CON-002** — As a Controller, I want to view a ConsolidatedReport for a ConsolidationGroup and fiscal year, so that I can see the total budget, total spend, and variance in one place.

*Acceptance criteria:*
- GIVEN I open a ConsolidatedReport  
  WHEN the report is in `final` status  
  THEN I see totalBudget, totalSpend, variance, and the report date; all amounts are formatted per user locale
- GIVEN multiple ConsolidatedReports exist  
  WHEN I filter by FiscalYear  
  THEN only reports for that fiscal year are shown

### Feature: Set savings goals

**US-SAV-001** — As a Financieel Beheerder, I want to create a SavingsOpportunity linked to a Budget and a target date, so that I can track progress toward a specific savings target throughout the year.

*Acceptance criteria:*
- GIVEN I am on the SavingsOpportunity detail page  
  WHEN I set targetAmount, currentAmount, targetDate, and link to a Budget  
  THEN the progress percentage is displayed and the SavingsOpportunity is saved via ObjectService
- GIVEN a SavingsOpportunity is in `active` status  
  WHEN currentAmount ≥ targetAmount  
  THEN the status transitions to `achieved` and a Nextcloud notification is dispatched to the responsible user

## Customer Journeys

### Journey: Consolidate year-end budgets
**Trigger:** End of fiscal year; Financieel Beheerder must produce a consolidated budget statement for management.

1. Open Shillinq → Consolidation Groups
2. Create or open the ConsolidationGroup for the current FiscalYear (e.g. "Gemeentelijke Diensten 2025")
3. Add all relevant Budgets to the group via the Budgets relation card
4. Navigate to ConsolidatedReports → generate a new report for the group and fiscal year
5. System aggregates totalBudget, totalSpend, and variance across all linked Budgets
6. Controller reviews and sets status to `final`; report is available for export via `CnMassExportDialog`

**Pain point addressed:** Eliminates manual Excel aggregation; provides a full audit trail of which budgets were consolidated and when.

### Journey: Set and track savings goal
**Trigger:** Start of fiscal year; Controller sets savings targets per department.

1. Open Shillinq → Savings Goals
2. Create SavingsOpportunity: enter name, description, targetAmount, targetDate; link to the relevant Budget
3. Set initial currentAmount (often 0 at start of year)
4. Monitor progress on the dashboard KPI widget as the year progresses; update currentAmount as savings are realized
5. At year-end, mark the SavingsOpportunity as `achieved` (if target met) or `cancelled`

**Pain point addressed:** Replaces spreadsheet-based goal tracking with a structured, auditable system integrated with the budget data already in Shillinq.

## Out of scope

- Automatic calculation of consolidation totals from ledger entries (post-MVP; requires integration with GeneralLedgerEntry)
- Multi-currency consolidation (handled by existing FXExposure and CurrencyBalance entities; not wired to consolidation in this change)
- Public API exposure of consolidated reports (use existing `/api/metrics` for monitoring)
