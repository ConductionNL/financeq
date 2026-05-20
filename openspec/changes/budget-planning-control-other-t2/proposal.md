---
kind: code
depends_on: []
chain: []
---

# Budget Planning & Control — Other T2

**App:** Shillinq
**Change:** budget-planning-control-other-t2
**Status:** proposed
**Date:** 2026-05-20

## Why

Market research across 50 feature clusters identifies budget planning and control as the highest-demand unmet area in Shillinq, with demand scores of 48–63 (up to 19 tender mentions, up to 3% competitor coverage). The top unmet clusters are:

- **Threshold-based budget alerts** (demand 63) — organisations lose financial control when spending silently crosses plan without any warning system; competitors cover this at only 3% indicating a market gap
- **Budget monitoring with over-budget prevention** (demand 63) — purchase requisitions and orders committed against exhausted budgets cause compliance failures and audit findings
- **Budget creation with allocation periods** (demand 63) — without structured monthly/quarterly/yearly budgets, managers cannot compare actual spend to plan at the correct granularity
- **Expense-by-category breakdown** (demand 62) — financial controllers need spending distribution by cost centre, project, and GL account to identify drift early
- **Budget owner approvals** (demand 62) — approval authority over budget ceiling adjustments must be delegated to named holders with an auditable trail

Dutch municipalities additionally require compliance with BBV programme structures, IV3 reporting categories, taakveld classification, and structural balance checks — all scoring demand 57 from public-sector tender mentions.

## What Changes

### New Capabilities

| Capability | Demand | Description |
|---|---|---|
| **budget-monitoring** | 63–59 | Threshold-based spend alerts (warning + critical), actual-vs-budget tracking, over-budget prevention at requisition and PO stages, commitment flagging |
| **budget-management** | 63–48 | Multi-dimensional budget creation with monthly/quarterly/yearly allocation periods; cost centre, project, and GL account dimensions; indexation percentages; departmental ceiling views; forecast comparison |
| **budget-approval** | 62–57 | Budget holder profile management, cost centre assignment, delegation, ceiling increase request/approval workflow, variation order budget impact assessment |
| **budget-compliance** | 57 | BBV programme linkage, IV3 category mapping, taakveld classification and validation, structural budget balance check, kadernota preparation, meerjarenplanning |
| **budget-contracts** | 58–57 | Contract and framework agreement budget utilisation tracking, ceiling breach alerts, cost centre and WBS element validation at requisition entry |

### Modified Capabilities

None — this is a net-new feature area for Shillinq.

## Impact

**Entities (all pre-existing in Shillinq data model — no new entities required):**
Budget, BudgetAllocation, BudgetAmendment, BudgetPeriod, CostCenter, FiscalYear, ExpenseCategory, PurchaseRequisition, PurchaseOrder, Contract, ContractSpendRecord, FrameworkAgreement, CallOffOrder, GeneralLedgerAccount, FundAllocation, ApprovalRequest, ApprovalChain

**Schema register patches (non-breaking additions per ADR-011):**
New fields added to Budget schema (`alertThreshold`, `warningThreshold`, `overBudgetPrevention`, `bbvProgramme`, `allocationPeriod`, `indexationPercentage`) and BudgetAllocation schema (`iv3Category`, `taakveld`, `structural`, `forecastAmount`).

**Declarative behaviours (x-openregister-* register patches):**
`x-openregister-notifications` for threshold alert dispatching; `x-openregister-aggregations` for actual-vs-budget summaries; `x-openregister-calculations` for utilisation %, remaining budget; `x-openregister-lifecycle` for BudgetAmendment approval workflow.

**PHP services (imperative — justified, see design.md):**
`BudgetValidationService` (cross-schema guard at requisition/PO save); `BudgetAmendmentTransitionGuard` (lifecycle guard updating budget ceiling on approval); `TaakveldValidationService` (BBV Bijlage IV code lookup); `StructuralBalanceService` (cross-budget structural balance computation).

**ADRs applicable:** ADR-001 (data layer), ADR-003 (backend), ADR-004 (frontend), ADR-007 (i18n nl+en), ADR-008 (testing), ADR-010 (NL Design), ADR-011 (schema standards), ADR-012 (deduplication), ADR-031 (declarative-first), ADR-032 (kind: code)

**Effort:** Large — multiple PHP services, schema register patches across two schemas, five specification capabilities, dashboard widgets
**Risk:** Medium — budget validation blocks requisition/PO submission; false-positive prevention requires careful testing of edge cases (zero ceiling, missing budget, overBudgetPrevention=false)
