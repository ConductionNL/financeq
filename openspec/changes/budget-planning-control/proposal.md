# Proposal: Budget Planning & Control — Shillinq

## Overview

Budget Planning & Control enables organizations to create, track, and enforce budget allocations across departments, cost centers, and projects. The feature set provides real-time budget monitoring, commitment tracking, forecasting, and amendment workflows with multi-stage approvals.

**Demand Score:** 12,110 (aggregated from 30 features)  
**Complexity:** High (multiple entities, approval workflows, forecasting logic)  
**Scope:** 49 user stories, 74 stakeholder touch points

## Key Features (Top 10 by Demand)

1. **Multi-location budget management** (demand: 1720)
   - Site-specific allocations and tracking across distributed organizations

2. **Direct material procurement with BOM management** (demand: 1700)
   - Forecast collaboration for material planning

3. **Public sector budgeting** (demand: 897)
   - Compliance with municipal budget cycles and reporting requirements

4. **Review fiscal impact of amendments** (demand: 771)
   - Financial impact analysis before budget changes

5. **Process budget amendment** (demand: 708)
   - Workflow for proposing and approving budget changes

6. **Budget compliance analytics** (demand: 611)
   - Policy adherence reporting across departments

7. **Approval workflow with budget impact review** (demand: 604)
   - Multi-stage approvals with financial validation at each decision point

8. **Record budget holder policy acknowledgement** (demand: 531)
   - Governance and compliance documentation

9. **Department and project-level budget tracking** (demand: 506)
   - Roll-up reporting from project to department to organization

10. **Real-time budget monitoring with commitment tracking** (demand: 428)
    - Track planned, committed, and actual spend by purchase request stage

## Core User Journeys

### Budget Holder Approval Workflow
- **Trigger:** Expenditure request submitted for approval
- **Actors:** Budget holder, Financial controller, Department head
- **Pain Point:** Lack of visibility into available budget and commitment impact
- **Resolution:** Real-time budget check showing available balance, pending commitments, and approval history

### Financial Controller Forecasting
- **Trigger:** Monthly financial review or policy requirement
- **Actors:** Financial controller, Project manager, Department head
- **Pain Point:** Manual calculation of cost-to-completion and variance forecasts
- **Resolution:** Automated EAC (Estimate At Completion) calculation based on historical spend patterns and run rate

### Budget Amendment Proposal
- **Trigger:** Actual costs exceed forecast or scope change mid-year
- **Actors:** Department head, Finance team, CFO/Director
- **Pain Point:** Lengthy approval chain, lack of impact transparency
- **Resolution:** Structured amendment workflow with fiscal impact pre-calculation and audit trail

## Stakeholder Involvement

**Key Stakeholders:**
- CEO / Director (final authority on major budgets)
- MT Member / Manager (departmental budget oversight)
- Department Head (operational budget control)
- Project Manager (project-level budget tracking)
- Controller / Financial Controller (financial validation and reporting)
- Board Treasurer (annual budget and financial statements)
- Compliance Officer (governance and policy adherence)
- Policy Officer (public sector compliance)

**Pain Points Addressed:**
- Unclear budget authority and delegation
- Manual tracking of committed vs. actual spend
- Delayed approval chains causing operational bottlenecks
- Inability to forecast budget exhaustion before it occurs
- Lack of audit trail for budget decisions and amendments

## Data Model

**Core Entities:**
- **Budget** — Financial plan allocating resources for a period, organization, and location
- **BudgetAllocation** — Subdivision of budget resources by department, funding source, or purpose
- **BudgetAmendment** — Proposed or executed change to approved budget amount
- **BudgetPeriod** — Defined time period for budget planning (fiscal year, quarter, month)
- **ExpenditureRequest** — Request to spend funds requiring review and approval
- **FundingSource** — Source of funds allocated to budgets
- **Location** — Physical or geographic location for multi-site budget allocation

## Success Metrics

- Budget amendment approval time < 5 business days (from proposal to decision)
- 95% budget availability visibility within real-time budget check
- 100% commitment tracking at purchase request stage (zero untracked spend)
- Forecast variance (predicted vs. actual) within ±10% by month-end

## Acceptance Criteria (High-Level)

- [ ] Budget creation with monthly allocation and variance tracking
- [ ] Real-time budget consumption tracking (committed, actual, forecasted spend)
- [ ] Amendment workflow with multi-stage approval and fiscal impact review
- [ ] Automatic approval blocking for over-budget requisitions
- [ ] Budget forecasting with EAC and exhaustion date prediction
- [ ] Budget holder policy acknowledgement records
- [ ] Department and cost-center roll-up reporting
- [ ] Export budget data in council-ready format
- [ ] Audit trail for all budget decisions and amendments

## Delivery Approach

1. **Phase 1:** Core budget entities and CRUD operations
2. **Phase 2:** Real-time budget checking in approval workflows
3. **Phase 3:** Forecasting and commitment tracking
4. **Phase 4:** Amendment workflow with multi-stage approvals
5. **Phase 5:** Analytics and reporting (compliance, variance, trends)

## Related Specs

- [Invoice & Expense Tracking](openspec/changes/...)
- [Purchase Order Management](openspec/changes/...)
- [Procurement Approval Workflow](openspec/changes/...)
- [Financial Reporting & Compliance](openspec/changes/...)
