# Spec: Budget Management

**Capability:** `budget-management`
**Change:** budget-planning-control-other-t1
**Status:** in-progress

## Overview

Core budget lifecycle management for Shillinq. Covers annual budget creation, bottom-up budget building from department (BU) submissions, council approval workflows, contract and framework lot ceiling setting, multi-dimensional budget hierarchies (location → department → project → cost center → GL account), and draft begroting workflows for internal review.

**Entities:** Budget, BudgetAllocation, BudgetAmendment, CostCenter, CostProject, Location, FiscalYear, FundingSource, FundAllocation, ApprovalRequest, ApprovalChain

## Requirements

### REQ-BUD-001 — Create and manage budgets

**As a** finance manager,
**I want to** create annual and project budgets with a name, reference, amount, period, and responsible department,
**so that** the organization has a formal budget record tied to the fiscal year.

**Acceptance Criteria:**

**Scenario 1: Create a new annual budget**
- GIVEN a finance manager is logged in with budget creation permission
- WHEN they submit a new Budget with name, reference, amount, fiscalYear, and department
- THEN a Budget object is created in OpenRegister with status `concept`
- AND the budget appears in the budget list

**Scenario 2: Set budget ceiling for a contract**
- GIVEN a budget holder is viewing a Contract record
- WHEN they link a Budget and set a `contractPlafond` amount
- THEN the Budget is linked to the Contract via OpenRegister relation
- AND the contract ceiling is visible on both the Budget and Contract detail pages

**Scenario 3: Set budget ceiling per framework lot**
- GIVEN a procurement officer is managing a FrameworkAgreement with Lot records
- WHEN they assign a Budget with a `lotPlafond` to a specific Lot
- THEN the allocation is stored and visible on the Lot detail page

---

### REQ-BUD-002 — Multi-dimensional budget hierarchy

**As a** controller,
**I want to** organize budgets across a hierarchy of location → department → project → cost center → GL account,
**so that** roll-up reporting works correctly at any level of the hierarchy.

**Acceptance Criteria:**

**Scenario 1: Assign budget to department and cost center**
- GIVEN a budget exists in status `actief`
- WHEN the controller creates a BudgetAllocation linking the Budget to a CostCenter and CostProject
- THEN the allocation is stored with the full dimensional path
- AND the Budget detail page shows a breakdown by dimension

**Scenario 2: Hierarchical roll-up in reporting**
- GIVEN budget allocations exist across multiple cost centers under one department
- WHEN the controller views the department-level budget summary
- THEN the total shows the sum of all child allocations
- AND drill-down to individual cost centers is available

**Scenario 3: Location-specific budget management**
- GIVEN an organization has multiple Locations
- WHEN a budget holder creates a BudgetAllocation scoped to a specific Location
- THEN the allocation is filterable by Location in lists and reports
- AND location budgets aggregate correctly to the parent budget

---

### REQ-BUD-003 — Bottom-up budget building from department submissions

**As a** department head,
**I want to** submit line-item budget requests for my department that feed into the overall budget,
**so that** the annual budget reflects actual operational needs.

**Acceptance Criteria:**

**Scenario 1: Enter line-item budget requests**
- GIVEN a department head is preparing a budget submission for the next fiscal year
- WHEN they create multiple BudgetAllocation objects linked to a draft Budget with status `concept`
- THEN each line item is stored with description, amount, cost center, and justification
- AND the parent Budget's total amount updates to reflect all line items

**Scenario 2: Submit BU budget for consolidation**
- GIVEN a department head has entered all line-item requests
- WHEN they submit the budget for review (lifecycle transition `concept` → `ingediend`)
- THEN the budget holder receives a notification
- AND the budget is locked for editing until reviewed

**Scenario 3: Circulate draft begroting for internal review**
- GIVEN a finance manager has consolidated BU submissions into a draft begroting
- WHEN they circulate the draft for internal review
- THEN all configured reviewers receive a notification with a link to the budget
- AND review comments can be attached via the standard notes feature (CnObjectSidebar)

---

### REQ-BUD-004 — Council budget approval

**As a** CFO or authorised approver,
**I want to** approve or reject budget proposals submitted for council review,
**so that** budgets are formally authorized before becoming active.

**Acceptance Criteria:**

**Scenario 1: Approve a submitted budget**
- GIVEN a Budget is in status `ingediend`
- WHEN an authorized approver (linked via ApprovalChain) transitions the Budget to `goedgekeurd`
- THEN the lifecycle guard `BudgetApprovalGuard` confirms the approver has budget approval permission
- AND the budget moves to `goedgekeurd` with an audit trail entry
- AND the budget holder receives a notification

**Scenario 2: Reject a budget with feedback**
- GIVEN a Budget is in status `ingediend`
- WHEN the approver returns the budget to `concept` with a remark
- THEN the budget holder receives a notification with the rejection reason
- AND the budget is editable again

**Scenario 3: Activate an approved budget**
- GIVEN a Budget is in status `goedgekeurd`
- WHEN the finance manager transitions it to `actief`
- THEN expenditure requests can be submitted against it
- AND the budget appears in active budget reporting

---

### REQ-BUD-005 — Budget amendment management

**As a** budget holder,
**I want to** request amendments to an active budget when needs change,
**so that** budget adjustments are formally tracked and approved.

**Acceptance Criteria:**

**Scenario 1: Request a budget amendment**
- GIVEN a Budget is in status `actief`
- WHEN the budget holder creates a BudgetAmendment with a requested amount change and justification
- THEN the amendment is stored in status `concept` linked to the parent Budget
- AND the amendment can be submitted for approval

**Scenario 2: Approve a budget amendment**
- GIVEN a BudgetAmendment is in status `aangevraagd`
- WHEN the authorized approver approves the amendment
- THEN the parent Budget's `amount` is updated
- AND an audit trail entry records the original and new amounts

**Scenario 3: Attach supporting documents**
- GIVEN a budget holder is preparing an expenditure request or amendment
- WHEN they attach supporting documents (quotes, invoices, decisions)
- THEN files are stored via `FileService` linked to the relevant Budget or BudgetAmendment object
- AND files are accessible from the `CnObjectSidebar` Files tab
