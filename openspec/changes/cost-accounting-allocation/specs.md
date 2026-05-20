# Specifications: Cost Accounting & Allocation — Shillinq

Requirements follow the REQ-XXX-NNN format. Each requirement is traceable to one or more features from the context brief. Acceptance criteria use GIVEN/WHEN/THEN notation.

---

## REQ-CC: Cost Center Management

Covers features: *Cost center tracking for departmental expense analysis* (demand: 83), *Kostenplaats- en kostendrager administratie* (demand: 73), *Cost Center Allocation* (demand: 13), *Cost center and profit center allocation for departmental accounting* (demand: 8), *Cost Center Accounting* (demand: 4), *Dimension-based accounting* (demand: 49).

### REQ-CC-001: Create and manage cost centers

GIVEN an administrator or finance manager is logged in WHEN they open the Kostenplaatsen index page THEN they see all cost centers with code, name, status, and budget in a sortable, filterable list.

GIVEN a user clicks "Nieuw" on the cost center list WHEN the schema-driven form opens THEN it requires `code`, `name`, and `status`; `budget`, `description`, and `createdDate` are optional.

GIVEN a user enters a duplicate `code` WHEN they save the cost center THEN the system rejects the save with a validation error indicating the code must be unique.

### REQ-CC-002: Activate and deactivate cost centers

GIVEN a cost center has `status: active` WHEN a manager sets it to `inactive` THEN the cost center no longer appears as a selectable dimension in new allocation rules or transactions.

GIVEN a cost center has `status: inactive` WHEN a manager reactivates it THEN it becomes available again as a dimension.

### REQ-CC-003: Cost center budget tracking

GIVEN a cost center has a `budget` set WHEN the finance manager opens the cost center detail page THEN the page shows the budget, total allocated costs to date, and the remaining budget as a derived value.

### REQ-CC-004: Cost center reporting dimensions

GIVEN cost centers exist in the system WHEN a finance manager runs a report THEN the report supports filtering and grouping by cost center code, department, and project dimension simultaneously.

---

## REQ-AR: Allocation Rule Management

Covers features: *Schedule recurring monthly overhead allocation* (demand: 200), *Cost center and project-based accounting with allocation rules* (demand: 19), *Automated account assignment with pre-configured cost center and GL mappings* (demand: 31), *Define a new overhead allocation key* (demand: unknown), *Version control the cost allocation model* (demand: 94).

### REQ-AR-001: Create percentage-based allocation rules

GIVEN a management accountant opens the allocation rule creation form WHEN they select `ruleType: percentage` THEN the form shows the `percentage` field and hides `fixedAmount`.

GIVEN a management accountant fills in source cost center, target cost center, percentage, frequency, and startDate WHEN they save THEN the allocation rule is created with `isActive: true` by default.

### REQ-AR-002: Create fixed-amount allocation rules

GIVEN a management accountant selects `ruleType: fixed_amount` WHEN they fill in `fixedAmount` and frequency THEN the system stores the rule and uses the fixed amount regardless of the source cost center balance.

### REQ-AR-003: Activate and deactivate rules

GIVEN an allocation rule exists WHEN a manager sets `isActive: false` THEN the rule is excluded from the next allocation run without being deleted.

GIVEN an allocation rule has an `endDate` in the past WHEN the allocation run executes THEN the system automatically skips expired rules and logs a skip notice.

### REQ-AR-004: Allocation key (verdeelsleutel) management

GIVEN a management accountant needs a formula-based allocation key WHEN they select `ruleType: formula` THEN the system stores the formula expression and evaluates it at run time using the source cost center's period balance.

GIVEN an allocation key total evaluates to zero WHEN the allocation run executes THEN the system raises a warning for that key, skips it, and continues processing remaining keys without dividing by zero.

---

## REQ-CA: Cost Allocation Execution and Version Control

Covers features: *Run overhead cost allocation to products* (user story US-1, demand: 2), *Reverse an incorrect overhead allocation* (demand: unknown), *View overhead allocation breakdown per product* (demand: 15), *Assign overhead cost pools to products and programmes* (demand: unknown), *Version control the cost allocation model* (demand: 94), *Compare calculated cost price to current tariff* (demand: unknown).

### REQ-CA-001: Trigger allocation run

GIVEN the overhead cost centres have been closed for the period WHEN the accountant triggers the allocation run THEN the system distributes costs across products using the configured allocation keys.

GIVEN the allocation run completes WHEN the accountant reviews the results THEN the system shows the allocated amount per product with the applied basis and key percentage.

GIVEN an allocation key total is zero WHEN the run executes THEN the system raises a warning and skips that key rather than dividing by zero.

### REQ-CA-002: Allocation run creates versioned CostAllocation records

GIVEN an allocation run executes for a period WHEN it creates CostAllocation records THEN each record has `version: 1` and `status: draft`.

GIVEN a CostAllocation record has `status: approved` WHEN the management accountant modifies the allocation model and re-runs THEN a new CostAllocation with incremented `version` is created; the previous version is retained for audit.

### REQ-CA-003: Approve allocation

GIVEN CostAllocation records exist with `status: draft` WHEN the finance manager reviews and approves them THEN their status changes to `approved`.

GIVEN a CostAllocation has `status: approved` WHEN the management accountant finalizes the period THEN the status changes to `allocated` and the records become read-only.

### REQ-CA-004: Reverse incorrect allocation

GIVEN a CostAllocation has `status: allocated` WHEN the management accountant triggers reversal THEN the system creates an offsetting CostAllocation with negative `allocationAmount` and the same `version`, and updates the original's status to indicate reversal.

GIVEN an allocation has been reversed WHEN the accountant views the allocation history THEN both the original and reversal entries are visible with their respective versions.

### REQ-CA-005: Allocation breakdown per product

GIVEN products and programmes are linked to cost centers WHEN the management accountant views the allocation breakdown THEN the system displays the allocated amount per product, the applied allocation key percentage, and the source cost center.

### REQ-CA-006: Recurring allocation scheduling

GIVEN an allocation rule has `frequency: monthly` WHEN the last day of the month passes THEN the system automatically creates a draft CostAllocation record for review by the management accountant.

GIVEN an allocation rule has `frequency: quarterly` WHEN the quarter end date passes THEN the system creates a draft CostAllocation for Q-end allocation.

---

## REQ-CP: Cost Project Management

Covers features: *Project accounting with time and cost allocation* (demand: 70), *Report by cost center, project and department dimensions* (demand: 240), *Profit & Loss with multi-dimensional analysis* (demand: 178).

### REQ-CP-001: Create and manage cost projects

GIVEN a project manager opens the project list WHEN they create a new project THEN the form requires `code`, `name`, `startDate`, and `status`; `budget`, `description`, and `endDate` are optional.

GIVEN a cost project exists WHEN time entries, allocations, or inventory valuations are linked to it THEN the project's `totalCost` reflects the sum of linked cost transactions.

### REQ-CP-002: Project status lifecycle

GIVEN a project has `status: active` WHEN the project manager closes it THEN the status changes to `closed` and no new cost entries can be linked to it.

GIVEN a project has `status: closed` WHEN an administrator archives it THEN the status changes to `archived` and the project is excluded from active reporting views.

### REQ-CP-003: Project budget monitoring

GIVEN a cost project has a `budget` set WHEN `totalCost` exceeds 90% of `budget` THEN the system raises a Nextcloud notification to the project manager and linked cost center manager.

### REQ-CP-004: Multi-dimensional reporting

GIVEN cost centers and projects exist WHEN a finance manager opens the reporting view THEN they can group P&L by cost center, project, department, or any combination of these dimensions.

---

## REQ-IV: Inventory Valuation

Covers features: *Inventory management with average cost valuation* (demand: 1596), *Inventory tracking with FIFO cost valuation method* (demand: 77), *Calculate full cost price per product* (demand: 39), *Define volume drivers for product cost price calculation* (demand: unknown).

### REQ-IV-001: Record inventory valuation

GIVEN a finance manager opens the inventory valuation form WHEN they select a product, enter quantity and unit cost THEN they must also select a `valuationMethod` (FIFO, average, specific, or weighted_average) before saving.

GIVEN an inventory valuation record is saved WHEN the system calculates `totalValue` THEN it multiplies `quantity` × `unitCost` and stores the result.

### REQ-IV-002: Average cost valuation

GIVEN multiple inventory receipts exist for a product with different unit costs WHEN the valuation method is `average` THEN the system calculates the weighted average unit cost across all receipt quantities and updates `unitCost` on the valuation record.

### REQ-IV-003: FIFO cost valuation

GIVEN inventory receipts exist for a product in chronological order WHEN the valuation method is `FIFO` THEN the system values on-hand stock using the cost of the earliest-received unsold units first.

### REQ-IV-004: Inventory status management

GIVEN an inventory valuation record has `status: active` WHEN a count adjustment is made THEN a new valuation record with `status: adjusted` is created; the previous record is retained.

GIVEN a product line is discontinued WHEN the inventory valuation status is set to `obsolete` THEN the record is excluded from active inventory totals on the balance sheet.

### REQ-IV-005: Full cost price calculation

GIVEN inventory items have an overhead allocation linked to the cost center WHEN the finance manager calculates the full cost price THEN the system adds the overhead allocation per unit to the direct unit cost to produce the full cost price per product.

---

## REQ-PD: Per Diem Management

Covers features: *Auto Per Diem Calculation* (demand: 11), *Dutch Per Diem Calculations* (demand: 8), *Dutch Mileage Rate (EUR 0.23/km)* (demand: 5), *Custom Expense Policies* (demand: 9).

### REQ-PD-001: Create per diem claim

GIVEN an employee opens the per diem claim form WHEN they enter `date`, `country`, and `nights` THEN the system automatically populates `rate` based on the configured rate table for that country and date.

GIVEN the country is `NL` and `nights >= 1` WHEN the system calculates the amount THEN it applies the Dutch domestic overnight per diem rate (default: EUR 37.00/night).

GIVEN the country is `NL` and `nights = 0` WHEN the system calculates the amount THEN it applies the Dutch day-trip rate (default: EUR 17.50).

### REQ-PD-002: Dutch mileage rate

GIVEN an employee logs a business trip with mileage WHEN they submit the claim THEN the system calculates reimbursement at EUR 0.23/km (configurable in admin settings).

### REQ-PD-003: Per diem approval workflow

GIVEN a per diem claim has `status: draft` WHEN the employee submits it THEN the status changes and the linked cost center manager receives a Nextcloud notification for approval.

GIVEN a per diem claim is submitted for approval WHEN the manager approves it THEN `status` changes to `approved`, `approvedDate` is set to today, and the employee is notified.

GIVEN a per diem claim has `status: approved` WHEN the payroll run processes it THEN `status` changes to `paid`.

### REQ-PD-004: Configurable per diem rates

GIVEN an administrator opens the admin settings WHEN they configure the Dutch domestic per diem rate THEN all new per diem claims default to that rate; existing approved claims are not retroactively changed.

---

## REQ-TS: Timesheet Management

Covers features: *Team time tracking with per-member hour logging* (demand: 384), *Built-in time tracker with start/stop timer and manual entry* (demand: 124), *Timesheet overview with weekly/monthly summary and utilization metrics* (demand: 100).

### REQ-TS-001: Create and manage timesheets

GIVEN an employee opens the timesheet list WHEN they create a new timesheet THEN they specify `periodStart`, `periodEnd`, and the system links it to their Person record.

GIVEN a timesheet covers a period WHEN TimeEntry objects are linked to the timesheet THEN the timesheet's `totalHours` aggregates the sum of all linked time entry hours.

### REQ-TS-002: Timesheet submission and approval

GIVEN a timesheet has `status: draft` WHEN the employee submits it THEN `status` changes to `submitted`, `submittedDate` is set, and the manager receives a Nextcloud notification.

GIVEN a submitted timesheet WHEN the manager approves it THEN `status` changes to `approved`, `approvedDate` is set, and the employee receives a confirmation notification.

GIVEN a submitted timesheet WHEN the manager rejects it THEN `status` returns to `draft` and the employee receives a notification with the rejection reason.

### REQ-TS-003: Utilization metrics

GIVEN a timesheet period spans N working days WHEN `totalHours` is known THEN `utilizationPercentage` is calculated as `(totalHours / (N × 8)) × 100`.

GIVEN a manager opens the timesheet overview WHEN they select a team or cost center THEN they see total hours, utilization percentage, and total cost aggregated per member for the selected period.

### REQ-TS-004: Labor cost calculation

GIVEN a timesheet has `totalHours` and the employee has an hourly rate on record WHEN the timesheet is approved THEN `totalCost` is calculated as `totalHours × hourly_rate` and allocated to the employee's primary cost center.

### REQ-TS-005: Weekly and monthly summary

GIVEN an employee opens the timesheet overview WHEN they select the weekly view THEN the system shows hours per day, grouped by week, with a weekly total.

GIVEN an employee selects the monthly view WHEN viewing the timesheet THEN the system shows total hours per week within the month and the monthly total with utilization percentage.

---

## REQ-RPT: Reporting and Dashboard

Covers features: *Report by cost center, project and department dimensions* (demand: 240), *Profit & Loss with multi-dimensional analysis (cost center, project, department)* (demand: 178), *Cost center and GL account allocation with automated coding* (demand: 9).

### REQ-RPT-001: Cost center expense report

GIVEN a finance manager opens the cost center report WHEN they select a cost center and period THEN the report shows all direct costs, received allocations, sent allocations, and net position for that center.

### REQ-RPT-002: Multi-dimensional P&L analysis

GIVEN P&L data exists across cost centers and projects WHEN a finance manager opens the P&L report THEN they can pivot by cost center, project, department, or any combination.

### REQ-RPT-003: Allocation audit trail

GIVEN a CostAllocation record exists WHEN the management accountant opens the detail page THEN the audit trail tab shows every version, who created it, when it was approved, and any reversal events.

---

## Non-Functional Requirements

### REQ-NFR-001: Dutch locale compliance
The system MUST present all amounts in EUR with 2 decimal precision and format dates according to Dutch locale (dd-mm-yyyy) respecting the user's Nextcloud locale setting.

### REQ-NFR-002: Zero-division guard
The allocation engine MUST never divide by zero. When an allocation key total is zero, the engine logs a warning, skips the key, and reports the skip in the run results.

### REQ-NFR-003: Idempotent seed data import
The seed data import MUST be idempotent: re-importing skips objects that already exist, matched by slug, without creating duplicates.

### REQ-NFR-004: Accessibility
All UI components MUST meet WCAG AA. Forms MUST have associated labels. Color MUST NOT be the sole method of conveying status information (use text labels alongside status badges).

### REQ-NFR-005: Authorization
All mutation endpoints (POST, PUT, DELETE) MUST enforce `IGroupManager::isAdmin()` on the backend. Frontend-only auth checks are insufficient.
