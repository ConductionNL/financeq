# Capability: Budget Contracts

**Spec:** budget-contracts
**Change:** budget-planning-control-other-t2
**Status:** proposed

## Description

Tracks budget utilisation for contracts and framework agreements (raamovereenkomsten), alerts managers when spend approaches the budget ceiling, validates cost centre and WBS element assignments at requisition entry for budget alignment, and assesses budget impact when call-off orders are placed against framework agreements.

## Stakeholders

- **Contract Manager** — monitors contract spend vs budget ceiling; receives ceiling breach alerts; manages budget amendments for over-budget contracts
- **Purchasing Manager (Inkoopmanager)** — places requisitions and call-off orders against framework budgets; receives WBS validation feedback at entry
- **Financial Controller (Financieel Controller)** — views framework agreement utilisation across the portfolio; validates cost centre alignment

## Requirements

### REQ-CTR-001: Track contract budget utilisation

Each Contract can be linked to a Budget, and actual spend tracked via ContractSpendRecords is aggregated against the contract's budget ceiling.

**Acceptance criteria:**

GIVEN a Contract "Schoonmaakdiensten 2026" is linked to Budget "Facilitair 2026" with ceiling €120,000
WHEN ContractSpendRecords are recorded totalling €95,000
THEN the contract detail page shows: `utilisation: 79%`, `spent: €95,000`, `remaining: €25,000`
AND the utilisation badge updates within one page refresh after a new ContractSpendRecord is added
AND the contract list view shows a utilisation percentage column sortable from highest to lowest
AND the linked Budget's aggregated actuals include the contract's spend

### REQ-CTR-002: Track framework agreement budget utilisation

FrameworkAgreements can be linked to a Budget; call-off orders (CallOffOrders) placed against the framework are counted toward the framework budget ceiling.

**Acceptance criteria:**

GIVEN a FrameworkAgreement "Raamovereenkomst ICT Hardware" linked to Budget "ICT Inkoop 2026" with ceiling €500,000
WHEN CallOffOrders totalling €380,000 are placed against the framework
THEN the framework detail page shows: `utilisation: 76%`, `committed: €380,000`, `remaining: €120,000`
AND each CallOffOrder is listed in the framework's spend breakdown
AND the Budget "ICT Inkoop 2026" reflects the framework's committed amount in its `committedAmount` aggregation

GIVEN a CallOffOrder is cancelled
WHEN the cancellation is saved
THEN the framework's `committed` amount decreases by the CallOffOrder value
AND the remaining balance increases accordingly

### REQ-CTR-003: Alert when contract spend approaches budget ceiling

When contract or framework utilisation reaches configured thresholds, the Contract Manager and Budget Owner receive Nextcloud notifications.

**Acceptance criteria:**

GIVEN a Contract Budget "Facilitair 2026" has `alertThreshold: 90` and `warningThreshold: 75`
AND current contract utilisation is 87%
WHEN a new ContractSpendRecord brings utilisation to 91%
THEN the Contract Manager receives notification: "Contractbudget kritisch: 91% van €120,000 besteed voor Schoonmaakdiensten 2026"
AND the Budget Owner receives the same notification
AND the contract list view shows the contract with a red utilisation badge (≥ alertThreshold)

GIVEN utilisation reaches exactly the `warningThreshold` of 75%
WHEN a ContractSpendRecord is added
THEN a warning notification is sent: "Contractbudget waarschuwing: 75% besteed van €120,000 voor Schoonmaakdiensten 2026"
AND the contract list view shows an amber badge

GIVEN the same contract already sent a warning notification at 75%
WHEN utilisation moves from 76% to 77% (still within the warning band)
THEN no duplicate notification is sent

### REQ-CTR-004: Cost centre and WBS element validation at requisition entry

When a PurchaseRequisition is entered, the system validates that the specified cost centre and WBS element (project code) are covered by an active, non-exhausted BudgetAllocation.

**Acceptance criteria:**

GIVEN a PurchaseRequisition is being created with CostCenter "Openbare Werken" and CostProject "WBS-2026-045"
WHEN the user enters these dimension values
THEN the system checks whether an active BudgetAllocation covers this combination
AND if no active allocation exists: inline validation error "Geen actieve begroting voor kostenplaats 'Openbare Werken' en WBS-element 'WBS-2026-045'"
AND if an allocation exists but is exhausted (remaining ≤ 0): inline warning "Budget voor dit kostenplaats/WBS-element is volledig uitgeput"
AND if the allocation exists and has remaining balance: a green indicator shows "Budget beschikbaar: €[remaining]"
AND the user cannot submit the requisition if cost centre or WBS element validation returns an error

GIVEN a PurchaseRequisition passes cost centre validation at creation
WHEN the budget for that cost centre is subsequently exhausted by another commitment
THEN re-validation on the requisition re-checks the current balance
AND if now exhausted, the requisition is flagged `budgetValidationStatus: warning`

### REQ-CTR-005: Budget alignment when placing call-off orders

When a CallOffOrder is placed against a FrameworkAgreement, the system validates the remaining framework budget before allowing the order to proceed.

**Acceptance criteria:**

GIVEN a FrameworkAgreement with remaining budget €20,000
WHEN a CallOffOrder of €25,000 is placed and the framework's linked Budget has `overBudgetPrevention: true`
THEN the system blocks the CallOffOrder with message: "Raamovereenkomst budget overschreden: €5,000 boven het resterende saldo"
AND a BudgetAmendment must be submitted and approved before the order can proceed

GIVEN the same situation with `overBudgetPrevention: false`
WHEN the CallOffOrder is placed
THEN the order is saved with `overBudget: true` and `budgetOverage: 5000`
AND a notification is sent to the framework Budget Owner

### REQ-CTR-006: Contract budget ceiling increase workflow

When a contract's budget is exhausted or a variation order exceeds the remaining balance, the contract manager can trigger a budget ceiling increase request directly from the contract record.

**Acceptance criteria:**

GIVEN a Contract Manager is on the contract detail page for "Schoonmaakdiensten 2026"
AND the contract's budget remaining balance is €2,000 while a new ContractSpendRecord of €8,000 is pending
WHEN they click "Budgetverhoging aanvragen" from the contract page
THEN a BudgetAmendment is pre-filled with the contract name, the required additional amount (€6,000), and the contract as justification context
AND the standard budget amendment approval workflow applies (see REQ-APR-003 / REQ-APR-004)
AND the contract spend record can be saved as a draft pending budget approval
