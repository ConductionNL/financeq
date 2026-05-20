# Capability: Budget Approval

**Spec:** budget-approval
**Change:** budget-planning-control-other-t2
**Status:** proposed

## Description

Manages budget holder (budgethouder) profiles, cost centre assignment, and the formal workflow for requesting and approving budget ceiling increases via BudgetAmendments. Includes assessment of budget impact from variation orders (contractwijzigingen) and delegation of approval authority.

## Stakeholders

- **Budget Owner (Budgethouder)** — manages their assigned cost centres; requests ceiling increases; assesses variation order budget impact; approves requisitions within delegated authority
- **CFO / Directeur Financiën** — approves or rejects ceiling increase requests; sets top-down budget targets
- **Financial Controller (Financieel Controller)** — administers budget holder profiles; assigns cost centres to holders; monitors the amendment pipeline

## Requirements

### REQ-APR-001: Budget holder profile management

Financial controllers can create and maintain budget holder profiles that link a Person to one or more CostCentres, establishing their budget authority.

**Acceptance criteria:**

GIVEN a Financial Controller creates a budget holder profile
WHEN they assign Person "J.H. van der Berg" as budget holder with access to CostCenter "ICT" and CostCenter "Facilitair"
THEN a configuration object is saved linking the person to both cost centres
AND J.H. van der Berg receives a Nextcloud notification: "U bent aangesteld als budgethouder voor: ICT, Facilitair"
AND they can see their assigned budgets in the budget overview upon next login
AND the assignment is recorded in the audit trail

GIVEN the budget holder profile already exists
WHEN the Financial Controller adds a third cost centre "Huisvesting"
THEN the assignment takes effect immediately
AND J.H. van der Berg's budget overview expands to include the new cost centre

### REQ-APR-002: Assign and remove cost centres from a budget holder

Cost centres can be added to or removed from a budget holder's profile at any time.

**Acceptance criteria:**

GIVEN a budget holder profile for "A.M. de Vries" covering CostCenter "Ruimtelijke Ordening" and CostCenter "Groen"
WHEN a Financial Controller removes "Groen" from her profile
THEN A.M. de Vries no longer sees "Groen" budgets in her overview
AND existing approved commitments she made against "Groen" are preserved
AND the removal is recorded in the audit trail with the administrator's name and timestamp
AND A.M. de Vries receives a notification of the change

### REQ-APR-003: Budget owner approval for amendments

Budget amendments (ceiling increases, reallocations) require approval from the budget owner before taking effect. The approval is routed via the existing ApprovalChain mechanism.

**Acceptance criteria:**

GIVEN a Financial Controller proposes a BudgetAmendment "Verhoging ICT Q3" increasing the ICT budget ceiling by €35,000
WHEN the amendment is submitted
THEN an ApprovalRequest is created and routed to the ICT Budget Owner via the ApprovalChain
AND the budget ceiling remains at the original value until the amendment reaches `status: approved`
AND the Budget Owner can approve or reject with a mandatory comment (minimum 10 characters)
AND upon approval: the Budget ceiling is updated, the BudgetAmendment status becomes `approved`, and the Financial Controller receives a notification

GIVEN the Budget Owner rejects the amendment
WHEN they submit the rejection
THEN the BudgetAmendment status becomes `rejected`
AND the ceiling remains unchanged
AND the proposing Financial Controller receives a notification with the rejection comment

### REQ-APR-004: Request and approve budget ceiling increase

Budget owners can formally request a ceiling increase through a self-service workflow that routes to the CFO for approval.

**Acceptance criteria:**

GIVEN a Budget Owner is on the budget detail page for "Openbare Werken 2026"
WHEN they click "Plafondverhoging aanvragen" and enter amount €120,000 with justification "Stormschade reparaties — onvoorziene kosten"
THEN a BudgetAmendment record is created with `type: ceilingIncrease` and `status: submitted`
AND the CFO receives an ApprovalRequest notification: "Plafondverhoging aangevraagd voor Openbare Werken 2026 (+€120,000)"
AND the CFO can approve or reject from the notification or from the amendments list view
AND the Budget Owner receives a notification of the decision with the CFO's comment

GIVEN the CFO approves the request
WHEN the approval is saved
THEN the Budget "Openbare Werken 2026" ceiling increases by €120,000
AND the BudgetAmendment shows `approvalDate` and `approvedBy`
AND the change is visible in the budget audit trail

### REQ-APR-005: Budget impact assessment of variation orders

When a variation order (ContractModification) is created on a Contract that has an associated Budget, the system calculates and displays the budget impact.

**Acceptance criteria:**

GIVEN a Contract "Schoonmaakdiensten 2026" linked to a Budget "Facilitair 2026" with remaining balance €8,000
WHEN a ContractModification (variation order) is created for an additional scope of €12,000
THEN the ContractModification record shows `budgetImpact: 12000` and `budgetImpactFlag: overBudget`
AND the detail page displays: "Resterend budget voor aanvang: €8,000 | Impact: +€12,000 | Verwacht saldo na wijziging: −€4,000"
AND the Contract's Budget Owner is notified: "Contractwijziging overschrijdt budgetplafond voor Facilitair 2026"
AND the variation order can still be saved but requires a BudgetAmendment to be submitted before PO creation

GIVEN a ContractModification within the remaining budget balance
WHEN it is created
THEN `budgetImpactFlag: withinBudget` is set
AND no notification is sent

### REQ-APR-006: Set top-down budget targets

The CFO can set top-down budget targets per department or BBV programme that constrain how budget owners may allocate their individual budgets.

**Acceptance criteria:**

GIVEN the CFO sets a top-down target of €2,800,000 for BBV programme "6 - Sociaal domein"
WHEN a Financial Controller creates or adjusts BudgetAllocations within that programme
THEN the system validates that the total allocation ceiling for the programme does not exceed the top-down target
AND if the total would exceed the target, an inline warning is shown: "Programmatotaal overschrijdt de kaderstellende norm van €2,800,000"
AND the warning does not block saving (it is advisory, not blocking)
