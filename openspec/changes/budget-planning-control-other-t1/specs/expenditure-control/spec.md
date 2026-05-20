# Spec: Expenditure Control

**Capability:** `expenditure-control`
**Change:** budget-planning-control-other-t1
**Status:** in-progress

## Overview

Pre-purchase budget validation and expenditure approval workflows. Blocks over-budget requisitions at the request stage, supports expenditure request submission for budget holder approval, budget override workflows for exceptional purchases requiring additional authorization, and document attachment support.

**Entities:** ExpenditureRequest, ExpenditureEscalation, Budget, BudgetAllocation, PurchaseRequisition, ApprovalRequest, ApprovalChain

## Requirements

### REQ-EXP-001 — Submit expenditure request for budget holder approval

**As a** employee or department manager,
**I want to** submit an expenditure request describing the purchase, amount, and justification,
**so that** the budget holder can review and authorize the spend before a purchase order is raised.

**Acceptance Criteria:**

**Scenario 1: Create and submit an expenditure request**
- GIVEN an employee needs to purchase goods or services
- WHEN they create an ExpenditureRequest with title, amount, budget allocation reference, supplier, and motivering
- AND submit it (lifecycle transition `concept` → `ingediend`)
- THEN the `BudgetAvailabilityGuard` checks that `resterendBudget` on the linked BudgetAllocation is ≥ the requested amount
- AND if funds are available, the request transitions to `ingediend`
- AND the budget holder linked via the BudgetAllocation receives a notification

**Scenario 2: Attach supporting documents to an expenditure request**
- GIVEN an employee is preparing an expenditure request
- WHEN they attach supporting documents (quotations, technical specifications, board decisions)
- THEN documents are uploaded via `FileService` and linked to the ExpenditureRequest object
- AND the Files tab in `CnObjectSidebar` shows all attachments

**Scenario 3: Budget holder approves an expenditure request**
- GIVEN an ExpenditureRequest is in status `in_behandeling`
- WHEN the budget holder approves it
- THEN the request transitions to `goedgekeurd`
- AND the `committed` amount on the linked BudgetAllocation is increased by the request amount
- AND the requester receives an approval notification

**Scenario 4: Budget holder rejects an expenditure request**
- GIVEN an ExpenditureRequest is in status `in_behandeling`
- WHEN the budget holder rejects it with a reason
- THEN the request transitions to `afgewezen`
- AND the committed amount on the BudgetAllocation is not affected
- AND the requester receives a rejection notification with the reason

---

### REQ-EXP-002 — Pre-purchase budget validation preventing over-commitment

**As a** budget holder,
**I want** the system to block expenditure requests that would exceed the available budget,
**so that** over-commitment is prevented at source rather than discovered after the fact.

**Acceptance Criteria:**

**Scenario 1: Block submission when budget is insufficient**
- GIVEN a BudgetAllocation has `resterendBudget` of €5,000
- WHEN an employee attempts to submit an ExpenditureRequest for €8,000
- THEN the `BudgetAvailabilityGuard` blocks the lifecycle transition
- AND the system returns an error message showing the available balance and the shortfall
- AND the ExpenditureRequest remains in status `concept`

**Scenario 2: Allow submission when budget is sufficient**
- GIVEN a BudgetAllocation has `resterendBudget` of €15,000
- WHEN an employee submits an ExpenditureRequest for €8,000
- THEN the guard allows the transition
- AND the request moves to `ingediend`

**Scenario 3: Real-time available balance shown in form**
- GIVEN an employee is filling in the ExpenditureRequest form
- WHEN they select a BudgetAllocation
- THEN the form displays the current `resterendBudget` (calculated field from Budget schema) in real time
- AND a warning badge appears if the requested amount approaches or exceeds the available balance

---

### REQ-EXP-003 — Budget override workflow for exceptional purchases

**As a** budget holder or director,
**I want** an override workflow for purchases that must proceed despite exceeding the available budget,
**so that** exceptional cases are handled with additional authorization rather than system bypass.

**Acceptance Criteria:**

**Scenario 1: Initiate a budget override request**
- GIVEN an ExpenditureRequest was blocked by `BudgetAvailabilityGuard`
- WHEN the requester or budget holder explicitly initiates a budget override
- THEN an ExpenditureEscalation is created linked to the original ExpenditureRequest with status `aangevraagd`
- AND the override is routed to the configured higher-level approver (director or controller) via the ApprovalChain

**Scenario 2: Approve a budget override**
- GIVEN an ExpenditureEscalation is pending review by the director
- WHEN the director approves the override with documented justification
- THEN the original ExpenditureRequest bypasses the budget guard and proceeds to `goedgekeurd`
- AND the BudgetAllocation records the override as a note in its audit trail
- AND all approval steps are captured in the AuditTrail for compliance purposes

**Scenario 3: Reject a budget override**
- GIVEN an ExpenditureEscalation is pending review
- WHEN the director rejects the override
- THEN the ExpenditureRequest remains in status `concept` and cannot proceed
- AND both the requester and original budget holder are notified

---

### REQ-EXP-004 — Budget enforcement at requisition stage

**As a** procurement officer,
**I want** purchase requisitions to be checked against available budget before they can be submitted,
**so that** procurement workflows enforce fiscal discipline from the very start.

**Acceptance Criteria:**

**Scenario 1: Block a PurchaseRequisition that would exceed budget**
- GIVEN a BudgetAllocation has insufficient `resterendBudget` for a requested PurchaseRequisition line
- WHEN the requester attempts to submit the PurchaseRequisition
- THEN the system blocks the submission and shows the budget shortfall per line item
- AND the requester is prompted to either reduce the amount or initiate an override request

**Scenario 2: Display budget availability in requisition form**
- GIVEN a procurement officer is creating a PurchaseRequisition
- WHEN they link it to a BudgetAllocation
- THEN the form shows the available budget balance and the utilisation percentage in real time
- AND lines that would exceed the balance are highlighted in a warning color using `CnStatusBadge`

**Scenario 3: Multiple requisitions competing for the same budget**
- GIVEN two employees simultaneously submit ExpenditureRequests against the same BudgetAllocation with only enough budget for one
- WHEN both requests pass the guard concurrently
- THEN the `BudgetAvailabilityGuard` uses a database-level check (committed amount updated atomically) to ensure only one succeeds
- AND the second request receives a clear insufficient-budget error
