# Capability: Budget Management

**Spec:** budget-management
**Change:** budget-planning-control-other-t2
**Status:** proposed

## Description

Enables creation and management of multi-dimensional budgets with flexible allocation periods (monthly, quarterly, yearly, custom), cost centre/project/GL account linkage, indexation percentages for year-over-year carry-forward, departmental budget ceiling views, and budget-vs-forecast comparison. Covers both private-sector (SMB/freelancer) and Dutch government budget structures.

## Stakeholders

- **Financial Controller (Financieel Controller)** — creates and maintains budgets; defines allocation periods; applies indexation; runs budget-vs-forecast analysis
- **Budget Owner (Budgethouder)** — views their departmental budget ceiling and per-period breakdown; cannot modify ceiling without an approved BudgetAmendment
- **CFO / Directeur Financiën** — reviews the overall budget structure; approves high-level allocations; views departmental ceilings across all departments

## Requirements

### REQ-MGT-001: Budget creation with allocation periods

Users can create a Budget with a designated allocation period; the system automatically generates the corresponding BudgetPeriod sub-records.

**Acceptance criteria:**

GIVEN a Financial Controller creates a new Budget "ICT 2026" with `allocationPeriod: quarterly` and `fiscalYear: 2026`
WHEN the budget is saved
THEN the system creates four BudgetPeriod records (Q1 2026-01-01–2026-03-31, Q2 2026-04-01–2026-06-30, Q3 2026-07-01–2026-09-30, Q4 2026-10-01–2026-12-31)
AND each BudgetPeriod is linked to the Budget
AND the Financial Controller can assign individual ceiling amounts to each period
AND the Budget total ceiling equals the sum of all period ceilings

GIVEN a Budget with `allocationPeriod: monthly`
WHEN it is saved for fiscal year 2026
THEN twelve BudgetPeriod records are created (January through December)

### REQ-MGT-002: Multi-dimensional budget allocations

BudgetAllocations can be assigned across multiple dimensions: cost centre, project, and GL account. Actual spend is counted against an allocation when all specified dimensions match the transaction.

**Acceptance criteria:**

GIVEN a Financial Controller creates a BudgetAllocation
WHEN they link it to CostCenter "Openbare Werken", CostProject "WBS-2026-045", and GeneralLedgerAccount "63010" with ceiling €850,000
THEN the BudgetAllocation is saved with all three dimension references
AND actual transactions carrying any combination of these dimension values are counted against this allocation's utilisation
AND the budget utilisation view supports filtering independently by each dimension
AND dimensions are optional — a BudgetAllocation may cover only cost centre, only GL account, or any combination

### REQ-MGT-003: Expense-by-category breakdown with percentage distribution

The budget detail page shows actual spend distributed across expense categories as percentages of the total budget ceiling.

**Acceptance criteria:**

GIVEN a Budget has active BudgetAllocations across multiple ExpenseCategories (e.g. Personeel, Huisvesting, ICT, Overig)
WHEN a Financial Controller views the budget detail page
THEN they see a pie chart showing percentage distribution per ExpenseCategory
AND each slice shows the category name, allocated ceiling, actual spend, and utilisation %
AND they can click a slice to drill down and see the underlying SpendingRecords for that category
AND the chart uses NL Design System colour tokens (not hardcoded hex values) per ADR-010

### REQ-MGT-004: Budget-vs-forecast comparison

Users can enter a year-end forecast (prognose) on a Budget and compare it to the original budget alongside actual spend.

**Acceptance criteria:**

GIVEN a Financial Controller has entered a year-end forecast of €420,000 on Budget "ICT 2026" (ceiling €450,000)
AND actual spend to date is €310,000
WHEN they view the budget dashboard
THEN they see: Original budget €450,000 | Prognose €420,000 | Actuals €310,000 side by side
AND the variance (prognose − ceiling) is shown as "−€30,000" in green (under budget)
AND if prognose > ceiling the variance is shown in red
AND they can update the forecast by entering a new value; all historical prognose values are retained in the audit trail

### REQ-MGT-005: Departmental budget ceiling view

Budget owners and managers can view the budget ceiling for their assigned departments without accessing the full financial configuration.

**Acceptance criteria:**

GIVEN a Budget Owner is logged in with access to CostCenter "Juridische Zaken"
WHEN they navigate to the budget overview
THEN they see only the budgets and BudgetAllocations assigned to their cost centre(s)
AND for each budget they see: ceiling, committed amount, actual spend, remaining balance, utilisation %
AND they cannot modify the ceiling or allocation amounts — these require an approved BudgetAmendment
AND the view is updated in real time as new commitments or transactions are recorded

### REQ-MGT-006: Apply indexation percentages to budget lines

Financial controllers can apply an indexation percentage to carry all BudgetAllocations forward to a new fiscal year with adjusted amounts.

**Acceptance criteria:**

GIVEN a Budget "ICT 2025" with total ceiling €500,000 and four quarterly BudgetPeriods
WHEN a Financial Controller clicks "Kopiëren naar nieuw boekjaar" and enters `indexationPercentage: 3.5` for fiscal year 2026
THEN the system creates a new Budget "ICT 2026" with ceiling €517,500 (500,000 × 1.035)
AND all BudgetAllocations from 2025 are copied to the new budget with amounts increased by 3.5%
AND four new BudgetPeriod records are created for 2026 with proportionally adjusted ceilings
AND the original 2025 Budget and its records remain unchanged
AND the new budget starts in `status: draft` pending review

### REQ-MGT-007: Browse and filter budgets by programme and taakveld

Users can browse the budget list filtered and grouped by BBV programme code and taakveld classification.

**Acceptance criteria:**

GIVEN multiple Budgets with different `bbvProgramme` values (e.g. "0 - Bestuur en ondersteuning", "6 - Sociaal domein")
WHEN a user opens the budget list and selects "Programma: 6 - Sociaal domein" in the facet sidebar
THEN only budgets tagged with that programme are shown
AND a secondary taakveld filter is available within the selected programme
AND programme subtotals (total ceiling, total actuals) are displayed in the facet sidebar
AND filtering to "Geen programma" shows budgets with no `bbvProgramme` set

### REQ-MGT-008: Update year-end forecast (rolling)

Financial controllers can update the year-end forecast at any point during the fiscal year; the latest prognose is displayed prominently.

**Acceptance criteria:**

GIVEN a Budget is in an active fiscal year
WHEN a Financial Controller clicks "Prognose bijwerken" and enters a new forecast amount
THEN the new prognose is saved with a timestamp
AND the budget detail page shows the updated prognose alongside the original ceiling and actuals
AND the previous prognose values remain visible in the audit trail tab
AND a bar chart widget on the dashboard updates to reflect the new prognose

### REQ-MGT-009: CapEx budget separation

Budgets can be tagged as `budgetType: capex` or `budgetType: opex` to support capital expenditure planning separate from operational spend.

**Acceptance criteria:**

GIVEN a Financial Controller creates a Budget with `budgetType: capex`
WHEN they view the budget list
THEN they can filter by budget type (CapEx / OpEx / All)
AND CapEx budgets are included in a separate CapEx summary on the dashboard
AND depreciation schedules linked via DepreciationSchedule can be associated with CapEx budgets
