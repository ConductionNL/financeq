---
kind: code
depends_on: []
---

# Proposal: Cost Accounting & Allocation — Shillinq

## Summary

Shillinq is a complete open-source business administration suite for freelancers, sole proprietors, SMBs, and corporations on Nextcloud. This change introduces cost accounting and allocation capabilities, covering cost center management, overhead allocation, project cost tracking, time tracking, inventory valuation, and per diem management. It adds seven new OpenRegister schemas and the custom business logic required for automated cost distribution, version-controlled allocation models, and multi-dimensional financial reporting.

## Problem Statement

Organisations using Shillinq for bookkeeping and invoicing have no built-in mechanism to:

- Track indirect costs (overhead) across cost centers and distribute them to products and projects
- Run and version-control recurring overhead allocation cycles
- Log employee hours against projects and cost centers for labor cost analysis
- Manage inventory at cost using FIFO or average cost methods for accurate P&L
- Calculate and claim Dutch per diem allowances in compliance with Dutch tax regulations

These gaps force organisations to maintain parallel spreadsheets or purchase separate cost accounting software, undermining the self-hosted, all-in-one premise of Shillinq.

## Goals

1. Enable management accountants to run monthly overhead allocation runs that distribute indirect costs to products and programmes using configurable allocation keys
2. Provide cost center and project dimensions for multi-dimensional P&L and expense analysis
3. Support time tracking at the team and individual level with timesheet approval workflows
4. Implement inventory valuation using FIFO and average cost methods
5. Automate Dutch per diem calculation and approval following Dutch tax regulations (EUR 0.23/km mileage, country-specific daily rates)
6. Version-control the cost allocation model so changes can be audited and incorrect allocations reversed

## Stakeholders

### Management Accountant
**Responsibilities:** Monthly overhead allocation runs, cost price calculations, financial reporting per dimension (cost center, project, department), allocation model version control, reversal of incorrect allocations.
**Goals:** Accurate indirect cost distribution, regulatory-compliant cost pricing, audit-ready allocation history.

### Finance Manager / Controller
**Responsibilities:** Oversight and approval of cost centers and allocation rules, departmental budget vs. actuals, multi-dimensional P&L review.
**Goals:** Real-time insight into departmental expense performance, automated allocation reducing manual journal entries.

### Project Manager
**Responsibilities:** Project budget management, time and material cost tracking per project, project profitability reporting.
**Goals:** Up-to-date project cost totals, clear split between direct and allocated overhead costs per project.

### HR / Payroll Manager
**Responsibilities:** Timesheet approval, per diem claim validation, labor cost allocation to cost centers.
**Goals:** Correct labor cost distribution, compliance with Dutch per diem and mileage regulations, efficient approval cycle.

### Employee / Team Member
**Responsibilities:** Logging time entries against projects and cost centers, submitting travel expense and per diem claims.
**Goals:** Easy start/stop time tracking, accurate travel expense reimbursement with minimal manual calculation.

## Features in Scope

The following features are included in this change, ordered by market demand:

| Demand | Feature |
|--------|---------|
| 1596 | Inventory management with average cost valuation |
| 384 | Team time tracking with per-member hour logging |
| 240 | Report by cost center, project and department dimensions |
| 200 | Schedule recurring monthly overhead allocation |
| 178 | Profit & Loss with multi-dimensional analysis (cost center, project, department) |
| 124 | Built-in time tracker with start/stop timer and manual entry |
| 100 | Timesheet overview with weekly/monthly summary and utilization metrics |
| 94 | Version control the cost allocation model |
| 88 | Configurable approval chains based on spend amount, category, cost center |
| 83 | Cost center tracking for departmental expense analysis |
| 79 | Approval routing by location, department, and cost center with escalation rules |
| 77 | Inventory tracking with FIFO cost valuation method |
| 73 | Kostenplaats- en kostendrager administratie |
| 70 | Project accounting with time and cost allocation |
| 49 | Dimension-based accounting (cost center, project, activity, free fields) |
| 39 | Calculate full cost price per product |
| 31 | Automated account assignment with pre-configured cost center and GL mappings |
| 19 | Cost center and project-based accounting with allocation rules |
| 15 | View overhead allocation breakdown per product |
| 13 | Cost Center Allocation |
| 11 | Auto Per Diem Calculation |
| 9 | Custom Expense Policies |
| 8 | Dutch Per Diem Calculations |
| 8 | Cost center and profit center allocation for departmental accounting |
| 5 | Dutch Mileage Rate (EUR 0.23/km) |
| 2 | Run overhead cost allocation to products |
| unknown | Define a new overhead allocation key |
| unknown | Define volume drivers for product cost price calculation |
| unknown | View overhead allocation breakdown per product |
| unknown | Reverse an incorrect overhead allocation |
| unknown | Assign overhead cost pools to products and programmes |
| unknown | Compare calculated cost price to current tariff |

## User Stories

### US-1: Run Overhead Cost Allocation to Products (priority: must)

> As a Management Accountant, I want to run the monthly overhead cost allocation to products and programmes, so that each product carries its fair share of indirect costs for accurate cost price calculation.

**Acceptance Criteria:**

- GIVEN the overhead cost centres have been closed for the period WHEN the accountant triggers the allocation run THEN the system distributes costs across products using the configured allocation keys
- GIVEN the allocation run completes WHEN the accountant reviews the results THEN the system shows the allocated amount per product with the applied basis and key percentage
- GIVEN an allocation key total is zero WHEN the run executes THEN the system raises a warning and skips that key rather than dividing by zero

## Out of Scope

The following features are deferred to future changes:

- GPS-based automatic business trip logging (mobile integration not yet available)
- Google Maps mileage calculator integration (external API dependency)
- Country-specific per diem and mileage rates beyond Dutch standard (EUR 0.23/km)
- Synergy realization tracking
- Fiscal partner allocation
- Payroll cost distribution
- Purchase landed cost calculation
- Track synergy realization
- Per diem deduction rules for partial days
- Commute vs. business trip split automation
- Spread costs/revenue across periods
- Category split automation
