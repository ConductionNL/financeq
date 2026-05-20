---
kind: code
depends_on: []
---

# Budget Planning & Control — Shillinq — Other T1

**Change:** budget-planning-control-other-t1
**App:** Shillinq
**Platform:** Nextcloud + OpenRegister

## Why

Budget planning and control represents one of the most significant capability gaps in Shillinq. Market research across 48 feature clusters reveals exceptional demand signals with low competitor coverage — a blue-ocean opportunity. The six highest-demand feature areas score between 1661 and 2005 on the composite demand index, covering 549–663 tender mentions with only 7–48% competitor coverage.

Dutch municipalities, SMBs, and corporations manage multi-dimensional budgets spanning departments, projects, cost centers, funding sources, locations, and fiscal years. Without a budget planning layer, Shillinq users must rely on external spreadsheets or separate ERP systems for budget oversight — fragmenting the single-platform promise and creating reconciliation overhead.

| Feature | Demand | Tender Mentions | Competitor Coverage |
|---|---|---|---|
| Location-specific budget management | 2005 | 663 | 8% |
| Budget period management (fiscal year + custom) | 1800 | 594 | 9% |
| Core budget management | 1735 | 545 | 48% |
| Management reporting with comparative analysis | 1706 | 562 | 10% |
| Department and project budget management with roll-up | 1691 | 559 | 7% |
| Funding source management with multi-stream allocation | 1661 | 549 | 7% |

Lower-tier features (demand 63–381) add critical workflow depth: expenditure request approval (1431), pre-purchase budget validation (118), budget overspend alerts (72), and fiscal year rollover (68).

## What Changes

### New Capabilities

**budget-management** — Core budget lifecycle covering annual budget creation, bottom-up budget building from BU submissions, council approval workflows, contract and framework lot ceiling setting, multi-dimensional budget hierarchies (location → department → project → cost center → GL account), and draft begroting workflows for internal review and circulation. Covers 15 features including "Build bottom-up budget", "Draft begroting", "Council budget approval", "Set and approve budget ceiling for a contract", "BU budget submission", and "Enter line-item budget requests".

**budget-period-management** — Fiscal year and custom budget period lifecycle management. Supports period-over-period comparison (proposed budget vs previous year), fiscal year rollover with carryforward and reallocation capabilities, and split contract budget ceilings across fiscal years. Covers 4 features including "Budget period management with fiscal year and custom period support", "Split contract budget ceiling across fiscal years", "Compare proposed budget to previous year", and "Create annual budget".

**expenditure-control** — Pre-purchase budget validation and expenditure approval workflows. Blocks over-budget requisitions at request stage, supports expenditure request submission for budget holder approval, budget override workflows for exceptional purchases requiring additional authorization, and budget enforcement at requisition stage. Covers 12 features including "Submit an expenditure request for budget holder approval", "Pre-purchase budget validation preventing over-commitment at requisition stage", "Budget enforcement at request stage blocking purchases exceeding available funds", "Budget override workflow for exceptional purchases requiring additional authorization", and "Attach supporting documents to budget request".

**budget-monitoring** — Real-time spending vs. budget tracking with configurable alerts. Covers per-user and per-supplier budget limits with instant alerts on threshold breach, department-level allocation monitoring, email notifications at configurable threshold percentages, traffic light status per programme, budget overspend alerts, contract budget exhaustion date forecasting, view real-time spending versus budget, and view budget utilisation across all contracts. Covers 11 features.

**budget-reporting** — Management reporting with comparative period and budget variance analysis. Includes budget creation and budget vs actuals comparison reporting, management reporting with comparative period and budget analysis, budget utilisation reporting with variance analysis and cost-saving identification, balance sheet with comparative period and budget variance analysis, and monitoring budget execution. Covers 6 features.

### Modified Capabilities

None — this is a greenfield capability module.

## Capabilities

### New Capabilities

| Capability | Description |
|---|---|
| `budget-management` | Core budget lifecycle: creation, hierarchy, approval, and BU submissions |
| `budget-period-management` | Fiscal year and period lifecycle with rollover and comparison support |
| `expenditure-control` | Pre-purchase validation, expenditure requests, and override approval workflows |
| `budget-monitoring` | Real-time spend tracking, threshold alerts, and overspend detection |
| `budget-reporting` | Comparative period reporting, variance analysis, and management dashboards |

### Modified Capabilities

None.

## Impact

**Primary users:** Finance managers and controllers, budget holders, department heads, procurement officers, CFOs, and management controllers at Dutch municipalities, SMBs, and corporations running Shillinq.

**Scope:** 48 features across 5 capability areas covering the complete budget planning and control lifecycle — from initial budget creation and approval through period management, real-time expenditure control, monitoring, and management reporting.

**Compliance:** Supports Dutch government budget management practices aligned with BBV (Besluit Begroting en Verantwoording) and IV3 reporting. Enables proper pre-commitment accounting and budget enforcement required for Dutch public procurement compliance.
