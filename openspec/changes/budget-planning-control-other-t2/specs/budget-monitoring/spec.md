# Capability: Budget Monitoring

**Spec:** budget-monitoring
**Change:** budget-planning-control-other-t2
**Status:** proposed

## Description

Provides real-time tracking of actual spending against budget, with configurable warning and critical threshold alerts dispatched via Nextcloud notifications, and automated over-budget prevention that blocks or flags purchase requisitions and purchase orders that would exhaust the available budget.

## Stakeholders

- **Financial Controller (Financieel Controller)** — configures alert thresholds per budget; monitors actual-vs-budget dashboards; reviews over-budget flags and commitment lists
- **Budget Owner (Budgethouder)** — receives notifications when spending in their area approaches or exceeds the budget ceiling
- **Purchasing Manager (Inkoopmanager)** — submits requisitions and purchase orders; encounters budget validation at entry; sees inline warning or block message

## Requirements

### REQ-MON-001: Warning and critical threshold alerts

When actual spending reaches a configurable warning percentage the budget owner and financial controller receive a Nextcloud notification; when it reaches the critical threshold a second notification fires.

**Acceptance criteria:**

GIVEN a Budget "ICT 2026" with `warningThreshold: 75` and `alertThreshold: 90` and ceiling €450,000
AND actual spend reaches €340,000 (75.6%)
WHEN the system evaluates budget utilisation
THEN the budget owner receives a Nextcloud notification with subject "Budgetwaarschuwing: 76% besteed van ICT 2026"
AND the financial controller receives the same notification
AND the Budget record shows `status: warning`
AND no duplicate notification is sent if utilisation remains in the 75–90% band

GIVEN the same Budget
AND actual spend reaches €410,000 (91.1%)
WHEN the system evaluates budget utilisation
THEN the budget owner receives a notification "Budget kritisch: 91% besteed van ICT 2026"
AND the financial controller receives the same notification
AND the Budget record shows `status: critical`

### REQ-MON-002: Over-budget prevention at requisition stage

When a purchase requisition would cause commitments plus actual spend to exceed the budget ceiling, the system blocks submission (if `overBudgetPrevention: true`) or flags the requisition (if `overBudgetPrevention: false`).

**Acceptance criteria:**

GIVEN a Budget for cost centre "ICT" with ceiling €450,000
AND existing actuals of €380,000 and committed POs of €60,000 (total committed: €440,000)
AND `overBudgetPrevention: true`
WHEN a user submits a PurchaseRequisition for €20,000 against that cost centre
THEN the system displays "Budget overschrijding: €10,000 boven het plafond van €450,000"
AND the requisition cannot be submitted without an approved BudgetAmendment
AND the blocked attempt is recorded in the audit trail with timestamp and user

GIVEN the same Budget with `overBudgetPrevention: false`
WHEN a user submits a PurchaseRequisition for €20,000 against that cost centre
THEN the requisition is saved with `overBudget: true` and `budgetOverage: 10000`
AND a warning banner is shown: "Let op: deze aanvraag overschrijdt het budgetplafond"
AND the financial controller receives a notification of the over-budget commitment

### REQ-MON-003: Over-budget prevention at purchase order stage

The same over-budget check that applies to purchase requisitions also applies when a purchase order is created or updated.

**Acceptance criteria:**

GIVEN a Budget with remaining balance €5,000 and `overBudgetPrevention: true`
WHEN a PurchaseOrder of €8,000 is created against this budget
THEN the system rejects the PO with message "Budget overschrijding: €3,000 boven het beschikbare saldo"
AND the PO is not persisted
AND the responsible buyer receives guidance to request a budget ceiling increase

GIVEN the same situation with a PO being updated (line item added, increasing total by €3,000)
WHEN the PO is saved
THEN the same block applies to the updated amount

### REQ-MON-004: Actual-vs-budget dashboard panel

Financial controllers can view a dashboard panel comparing actual spend to budget by cost centre and fiscal period.

**Acceptance criteria:**

GIVEN a Financial Controller is on the Shillinq dashboard
WHEN they open the "Budgetoverzicht" widget
THEN they see a bar chart with one bar per active cost centre for the current fiscal period
AND each bar shows budget ceiling (target) and actual spend (actual) as stacked segments
AND utilisation percentage is shown as a label on each bar
AND bars where utilisation ≥ alertThreshold are coloured red, ≥ warningThreshold amber, otherwise green

### REQ-MON-005: Custom alert thresholds per budget

Each budget can have independently configured warning and critical alert thresholds.

**Acceptance criteria:**

GIVEN a Budget Owner edits the Budget "Sociaal Domein 2026"
WHEN they set `warningThreshold: 80` and `alertThreshold: 95`
THEN the system saves these values
AND subsequent utilisation calculations use 80% as the warning trigger and 95% as the critical trigger for this budget
AND other budgets retain their own threshold settings

### REQ-MON-006: Flag commitment exceeding budget

Purchase orders and commitments that exceed the budget are visually flagged in list views and carry machine-readable `overBudget` and `budgetOverage` fields.

**Acceptance criteria:**

GIVEN a Budget "Renovatie Stadhuis" with remaining balance €2,000 and `overBudgetPrevention: false`
WHEN a PurchaseOrder of €5,000 is created against this budget
THEN the PurchaseOrder record shows `overBudget: true` and `budgetOverage: 3000`
AND the budget list view shows a warning badge next to "Renovatie Stadhuis"
AND the Financial Controller can filter the budget list to show only budgets with over-budget commitments
AND the PurchaseOrder list can be filtered to show only `overBudget: true` records
