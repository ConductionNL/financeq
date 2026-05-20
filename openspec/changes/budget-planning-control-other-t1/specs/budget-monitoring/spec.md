# Spec: Budget Monitoring

**Capability:** `budget-monitoring`
**Change:** budget-planning-control-other-t1
**Status:** in-progress

## Overview

Real-time spending vs. budget tracking with configurable threshold alerts. Covers per-user and per-supplier budget limits with instant overspend notifications, department-level allocation monitoring, traffic light status per programme, contract budget exhaustion date forecasting, and a live budget utilisation dashboard.

**Entities:** Budget, BudgetAllocation, ExpenditureRequest, Contract, Supplier, User

## Requirements

### REQ-MON-001 — View real-time spending versus budget

**As a** budget holder or finance manager,
**I want to** see current spending against budget in real time for each budget and allocation,
**so that** I can take corrective action before limits are breached.

**Acceptance Criteria:**

**Scenario 1: View budget utilisation on the dashboard**
- GIVEN budget allocations with `spent`, `committed`, and `amount` values exist
- WHEN the user opens the Budget Dashboard
- THEN a KPI block (`CnStatsBlock`) shows total budget, total spent, total committed, and remaining budget
- AND a bar or donut chart (`CnChartWidget`) shows utilisation percentage per top-level budget

**Scenario 2: View utilisation per allocation**
- GIVEN a budget has multiple BudgetAllocations across departments
- WHEN the finance manager views the Budget detail page
- THEN each allocation row shows `utilisatiePercentage` (calculated field) and a `CnProgressBar`
- AND allocations above the configured `drempelPercentage` are highlighted with a warning badge

**Scenario 3: View budget utilisation across all contracts**
- GIVEN Contracts are linked to Budget objects
- WHEN the procurement officer views the contract budget overview
- THEN each contract line shows: original ceiling, committed, spent, remaining, and forecast exhaustion date
- AND the list is sortable by utilisation percentage

---

### REQ-MON-002 — Configurable threshold alerts with notifications

**As a** finance manager,
**I want to** set threshold percentages per budget and receive notifications when those thresholds are crossed,
**so that** I am informed before budgets are exhausted without manually checking.

**Acceptance Criteria:**

**Scenario 1: Configure a budget threshold alert**
- GIVEN a Budget is in status `actief`
- WHEN the finance manager sets `drempelPercentage` to 80 on the Budget
- THEN the `x-openregister-notifications` declaration on the Budget schema monitors `utilisatiePercentage >= drempelPercentage`
- AND when the threshold is reached, the budget holder and finance manager receive a Nextcloud notification and email

**Scenario 2: Threshold breach notification content**
- GIVEN a Budget's `utilisatiePercentage` calculation crosses `drempelPercentage`
- WHEN the notification fires
- THEN the notification subject reads "Budget drempel bereikt: {name}"
- AND the body shows the current percentage, threshold, and a direct link to the Budget detail page

**Scenario 3: Set consumption threshold alert for a specific allocation**
- GIVEN a BudgetAllocation exists for a department
- WHEN the controller sets `drempelPercentage` on the allocation
- THEN the allocation-level notification triggers independently of the parent budget threshold
- AND the department head is included in the notification recipients

---

### REQ-MON-003 — Budget limits per user and per supplier with instant alerts

**As a** procurement manager,
**I want to** define maximum spend limits per user and per supplier within a budget period,
**so that** no single requester or supplier can exhaust a budget unilaterally.

**Acceptance Criteria:**

**Scenario 1: Set a per-user expenditure limit**
- GIVEN a budget holder is configuring a Budget
- WHEN they set a `limietPerGebruiker` amount on the BudgetAllocation
- THEN each user's cumulative ExpenditureRequest amount is checked against this limit at submission time
- AND users who reach the limit are blocked from submitting additional requests against this allocation

**Scenario 2: Set a per-supplier budget limit**
- GIVEN a procurement manager wants to cap spend with a specific Supplier
- WHEN they configure a per-supplier limit on a Budget or BudgetAllocation
- THEN the system tracks cumulative spend per Supplier within the budget period
- AND an alert is sent when a supplier approaches or reaches the limit

**Scenario 3: Instant alert on threshold breach**
- GIVEN a per-user or per-supplier limit is configured
- WHEN a transaction pushes cumulative spend over the configured limit
- THEN the budget holder receives an immediate Nextcloud notification
- AND the offending transaction is flagged in the expenditure request list with a `CnStatusBadge`

---

### REQ-MON-004 — Traffic light status per programme

**As a** programme manager,
**I want to** see a traffic light (RAG) status for each programme's budget health,
**so that** portfolio oversight is possible at a glance without drilling into individual budgets.

**Acceptance Criteria:**

**Scenario 1: Traffic light status calculated from utilisation**
- GIVEN multiple Budgets are grouped under a programme or department
- WHEN the programme manager views the portfolio dashboard
- THEN each programme shows a traffic light: green (< 70%), amber (70–90%), red (> 90% or overrun)
- AND the status is derived from the `utilisatiePercentage` calculated field

**Scenario 2: Drill-down from traffic light to budget detail**
- GIVEN the programme dashboard shows a red status for a programme
- WHEN the programme manager clicks the red indicator
- THEN they are navigated to the Budget detail page for the overrunning allocation

---

### REQ-MON-005 — Forecast contract budget exhaustion date

**As a** contract manager,
**I want to** see a forecasted date when a contract's budget will be exhausted at the current spend rate,
**so that** re-negotiation or budget amendments can be initiated proactively.

**Acceptance Criteria:**

**Scenario 1: Exhaustion date calculated and displayed**
- GIVEN a Budget linked to a Contract has committed and spent values over multiple months
- WHEN the contract manager views the Budget or Contract detail page
- THEN the `verwachteUitputtingsdatum` calculated field shows the forecasted exhaustion date
- AND the forecast is based on the 30-day moving average spend rate (`burnRate30Dagen`)

**Scenario 2: Alert when exhaustion is imminent**
- GIVEN the `verwachteUitputtingsdatum` is within 60 days
- WHEN the nightly recalculation runs
- THEN the contract manager and budget holder receive a proactive notification
- AND the contract appears in a "Budgets bijna uitgeput" section on the dashboard
