# Spec: Budget Period Management

**Capability:** `budget-period-management`
**Change:** budget-planning-control-other-t1
**Status:** in-progress

## Overview

Fiscal year and custom budget period lifecycle management. Supports period definitions (annual, quarterly, monthly), period-over-period comparison (proposed budget vs previous year), split contract budget ceilings across fiscal years, and fiscal year rollover with carryforward and reallocation.

**Entities:** BudgetPeriod, FiscalYear, Budget, BudgetAllocation, Contract

## Requirements

### REQ-PER-001 — Create and manage budget periods

**As a** finance manager,
**I want to** define budget periods (fiscal years, quarters, custom periods) that budgets and allocations are tied to,
**so that** all financial reporting is anchored to consistent time boundaries.

**Acceptance Criteria:**

**Scenario 1: Create a fiscal year period**
- GIVEN a finance manager is setting up the budget administration for a new year
- WHEN they create a BudgetPeriod with type `annual`, startDate `2026-01-01`, and endDate `2026-12-31`
- THEN the period is stored with status `open`
- AND budgets can be linked to this period immediately

**Scenario 2: Create a quarterly sub-period**
- GIVEN a fiscal year BudgetPeriod exists in status `open`
- WHEN the finance manager creates a BudgetPeriod with type `quarterly` for Q1
- THEN the quarterly period is stored linked to the parent fiscal year
- AND BudgetAllocations can be scoped to either the quarterly or annual period

**Scenario 3: Close a budget period**
- GIVEN a BudgetPeriod is in status `open` and the period end date has passed
- WHEN the finance manager closes the period
- THEN the period status transitions to `gesloten`
- AND no new budget transactions can be posted against this period

---

### REQ-PER-002 — Compare proposed budget to previous year

**As a** controller,
**I want to** compare the proposed budget for the coming year side-by-side with the previous year's budget and actuals,
**so that** the council can evaluate changes in a structured way.

**Acceptance Criteria:**

**Scenario 1: Year-over-year comparison view**
- GIVEN budgets exist for both the current fiscal year (actief) and the proposed next year (ingediend)
- WHEN the controller opens the budget comparison report for the two periods
- THEN the report shows for each budget line: previous year approved, previous year actual, proposed year, and variance (amount and percentage)
- AND the report is sorted by cost center / department hierarchy

**Scenario 2: Export comparison report**
- GIVEN a period comparison report is displayed
- WHEN the controller exports the report
- THEN a CSV or Excel file is generated via `ExportService` containing all comparison columns
- AND the export preserves the hierarchical structure

---

### REQ-PER-003 — Split contract budget ceiling across fiscal years

**As a** procurement officer,
**I want to** split a multi-year contract's budget ceiling across the relevant fiscal years,
**so that** annual budgets accurately reflect the year-specific commitment.

**Acceptance Criteria:**

**Scenario 1: Allocate contract budget per fiscal year**
- GIVEN a multi-year Contract exists with a total budget ceiling
- WHEN the procurement officer creates BudgetAllocation objects for the Contract linked to each relevant FiscalYear BudgetPeriod
- THEN each annual allocation is stored and totals are visible on both the Contract and Budget detail pages
- AND the sum of annual allocations matches the total contract value

**Scenario 2: Validate split does not exceed contract ceiling**
- GIVEN a Contract has a budget ceiling of €500,000
- WHEN the procurement officer assigns annual allocations totalling €550,000
- THEN the system shows a validation warning that the split exceeds the contract ceiling
- AND the over-allocation is highlighted in the budget view

---

### REQ-PER-004 — Fiscal year rollover with carryforward

**As a** finance manager,
**I want to** roll over unspent budget allocations from a closing fiscal year to the new year, with configurable carryforward rules,
**so that** approved budgets that could not be spent due to timing carry forward correctly.

**Acceptance Criteria:**

**Scenario 1: Configure carryforward rules before rollover**
- GIVEN the fiscal year is approaching its end date
- WHEN the finance manager configures which cost centers or budget categories are eligible for carryforward
- THEN the configuration is stored as BudgetAllocation metadata accessible to the rollover workflow

**Scenario 2: Execute fiscal year rollover**
- GIVEN the closing fiscal year BudgetPeriod is being closed
- WHEN the scheduled rollover workflow runs (via OR ScheduledWorkflow + n8n)
- THEN for each eligible BudgetAllocation, a new BudgetAllocation is created in the new fiscal year for the carryforward amount (`resterendBudget`)
- AND the original allocation is marked as rolled over in its metadata
- AND the finance manager receives a rollover completion notification

**Scenario 3: Review rollover results**
- GIVEN the fiscal year rollover has completed
- WHEN the finance manager opens the rollover report
- THEN the report lists all carried-forward allocations with original period, amount, and new allocation reference
