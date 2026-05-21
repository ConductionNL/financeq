# Specifications: Budget Planning & Control — Shillinq

## Requirements Overview

All requirements follow the REQ-{category}-{number} format with GIVEN/WHEN/THEN acceptance criteria derived from the user stories in context-brief.md.

---

## Core Budget Management

### REQ-BUDGET-001: Create Budget with Allocations

**Story:** Budget creation with monthly allocation and variance tracking (demand: 146)

**Description:**
Budget holders and finance managers can create a budget record with a total amount, time period, cost center, and monthly allocation breakdown. The system tracks the distribution across months and calculates variance against actual spend.

**Acceptance Criteria:**

```gherkin
Scenario: Create a budget and allocate across months
  GIVEN I have Finance Manager role
  WHEN I open the Budget creation form
  THEN I can enter:
    - Budget name (e.g., "Marketing 2026")
    - Total amount (e.g., €50,000)
    - Start date and end date
    - Budget category (operational, capital, training, etc.)
    - Cost center code
    - Currency (defaults to EUR)
    - Budget type (fixed, flexible, rolling, zero-based)
    - Alert threshold percentage (default 80%)

Scenario: System allocates budget across months
  GIVEN I have created a budget "IT Support 2026" with €120,000 from Jan-Dec
  AND I open the allocation details
  WHEN I view the Monthly Breakdown
  THEN the system displays:
    - Total for budget: €120,000
    - Monthly equal split: €10,000/month
    - Month-by-month allocation editable (for flexible budgets)
    - Cumulative remaining after each month

Scenario: Budget is not saved without required fields
  GIVEN the create form is open
  WHEN I try to save without entering budgetName, totalAmount, or startDate
  THEN the system shows validation errors and prevents save
  AND displays clear error messages for each missing field
```

---

### REQ-BUDGET-002: Real-time Budget Consumption Tracking

**Story:** Real-time budget consumption tracking with committed, actual, and forecasted spend views (demand: 185)

**Description:**
The system tracks budget consumption in three states: (1) actual invoiced spend, (2) committed spend from approved purchase orders, (3) forecasted spend based on historical burn rate. Budget holders see all three views simultaneously in the budget detail page.

**Acceptance Criteria:**

```gherkin
Scenario: Budget shows breakdown of spend types
  GIVEN a budget "Facilities 2026" with €250,000
  AND current date is 2026-05-21
  WHEN I open the budget detail page
  THEN I see:
    - Actual Spend (invoiced): €127,500 (51%)
    - Committed Spend (approved POs): €45,000 (18%)
    - Forecasted Spend (remaining 7 months): €95,000 (38%)
    - Total (actual + committed + forecast): €267,500 (107% - FLAG AS RISK)
    - Remaining Available Balance: €-17,500

Scenario: Committed spend updates when PO is approved
  GIVEN an existing budget with €0 committed spend
  WHEN a Purchase Order for €5,000 is approved
  AND the PO is linked to this budget
  THEN within 60 seconds, the budget page shows:
    - Committed Spend increased to €5,000
    - Remaining balance recalculated
    - Warning if total (actual + committed + forecast) exceeds 100%

Scenario: Forecast includes only linked spend items
  GIVEN a budget "Training 2026"
  WHEN I expand the Forecasted Spend detail
  THEN I see:
    - List of committed items (approved POs not yet invoiced)
    - Burn rate calculation (actual spend / elapsed months)
    - Projected spend for remaining months at current rate
    - Projected exhaustion date if rate continues
    - Confidence range (±10% variance historical data)
```

---

### REQ-BUDGET-003: Budget Check Integration with Approval Workflows

**Story:** Budget check integration blocking requisitions that exceed available budget (demand: 245)

**Description:**
When an expenditure request or purchase requisition is submitted for approval, the system checks if the request amount exceeds the remaining available budget. If it does, the approval is blocked with a clear message, and the budget holder receives a notification recommending budget amendment or reallocation.

**Acceptance Criteria:**

```gherkin
Scenario: Approval is blocked when budget is insufficient
  GIVEN a budget "Supplies 2026" with €50,000 remaining balance
  WHEN an expenditure request for €55,000 is submitted
  AND the approval workflow begins
  THEN:
    - Approval task includes budget check result: FAIL
    - Red banner shows: "Budget exceeded by €5,000. Approval blocked."
    - Approver sees two options:
      1. Amend the request amount to ≤€50,000
      2. Propose budget amendment (creates amendment draft)
      3. Reject the request

Scenario: Approval proceeds when budget is available
  GIVEN a budget "Supplies 2026" with €55,000 remaining balance
  WHEN an expenditure request for €50,000 is submitted
  THEN:
    - Budget check result: PASS
    - Green indicator shows: "Available balance: €5,000 after this request"
    - Approval workflow proceeds without budget obstruction

Scenario: Budget check considers pending approvals
  GIVEN a budget with €100,000 total, €70,000 already spent
  AND an existing expenditure request for €20,000 pending approval
  WHEN a second expenditure request for €15,000 is submitted
  THEN the budget check considers both pending requests:
    - Available balance calculated as: €100,000 - €70,000 - €20,000 = €10,000
    - Second request (€15,000) exceeds available (€10,000)
    - Approval blocked with message about pending approvals
```

---

### REQ-BUDGET-004: Department and Project-level Budget Tracking with Roll-up

**Story:** Department and project-level budget tracking with roll-up reporting (demand: 506)

**Description:**
Budgets can be allocated to departments and projects. The system rolls up spend from child entities to parent. A department manager sees the sum of all project budgets within their department. An executive sees the sum across all departments.

**Acceptance Criteria:**

```gherkin
Scenario: Parent budget sums child allocations
  GIVEN an organization structure:
    - Organization: "Gemeente Amsterdam"
      - Department: "Finance" (budget: €500,000)
        - Project: "Tax Collection" (budget: €200,000, spent: €95,000)
        - Project: "Audit Compliance" (budget: €150,000, spent: €75,000)
        - Project: "Grants Management" (budget: €150,000, spent: €60,000)
  WHEN I view the Department budget summary
  THEN I see:
    - Department total budget: €500,000
    - Actual spend (sum): €95,000 + €75,000 + €60,000 = €230,000
    - Committed (sum of all project POs): €X
    - Remaining balance: €500,000 - €230,000 = €270,000
    - Breakdown showing all child projects

Scenario: Roll-up reflects real-time changes
  GIVEN the department budget is displayed
  WHEN a project within the department incurs a new invoice (€10,000)
  AND the invoice is marked as "paid" in the system
  THEN within 60 seconds:
    - Project actual spend increases to €105,000
    - Department actual spend increases to €240,000
    - Remaining balance recalculated and displayed

Scenario: Filter and export department spend by cost category
  GIVEN a department budget with multiple projects
  WHEN I open the breakdown and filter by "Personnel" or "Equipment"
  THEN I see:
    - Only expenses in that category
    - Filtered total
    - Option to export CSV with department, project, category, amount, date
```

---

## Budget Amendments & Governance

### REQ-AMEND-001: Process Budget Amendment with Multi-stage Approval

**Story:** Process budget amendment with fiscal impact review at each decision point (demand: 708)

**Description:**
Budget amendments follow a structured workflow: propose → review fiscal impact → approve → execute. Each stage has defined roles, notifications, and audit logging. The amendment must not become effective until it reaches "approved" status.

**Acceptance Criteria:**

```gherkin
Scenario: Department head proposes a budget amendment
  GIVEN a budget "Training 2026" with €45,000 originally approved
  WHEN a department head submits an amendment proposal:
    - Original amount: €45,000
    - New amount: €52,500
    - Reason: "Scope expansion: additional certification programs"
    - Supporting documents (PDF, docx)
    - Effective date: 2026-06-01
  THEN:
    - Amendment record is created with status "proposed"
    - Amendment number is assigned (e.g., "AMEND-2026-001")
    - Notification sent to Financial Controller for review
    - Amendment saved as draft until submitted

Scenario: Financial Controller reviews fiscal impact
  GIVEN an amendment with status "proposed"
  WHEN the Financial Controller opens the review task
  THEN they see:
    - Current budget balance and utilization
    - Proposed new amount and impact (€7,500 increase)
    - Impact on department cash flow and overall org budget
    - Historical variance for this budget (did similar budgets overshoot?)
    - Option to:
      1. Approve (move to pending Director approval)
      2. Request changes (return to proposer with notes)
      3. Reject (block amendment with reason)
  AND if approved, the amendment moves to status "pending_approval"

Scenario: Director approves or rejects amendment
  GIVEN an amendment with status "pending_approval"
  WHEN the Director receives the approval task
  THEN they see:
    - Full amendment detail
    - Financial Controller's recommendation and notes
    - Budget trend and departmental context
    - Option to Approve (effective immediately or at scheduled date) or Reject
  AND if approved:
    - Amendment status becomes "approved"
    - Effective date is recorded
    - New budget total becomes active
    - Notification sent to department head confirming approval
  AND if rejected:
    - Amendment status becomes "rejected"
    - Department head receives notification with rejection reason
    - Budget remains unchanged

Scenario: Amendment execution is logged in audit trail
  GIVEN an approved amendment with effective date 2026-06-01
  WHEN the effective date arrives
  THEN:
    - Amendment status changes to "executed"
    - Budget total is updated to new amount
    - Entry created in audit log:
      [Amendment AMEND-2026-001 executed: €45,000 → €52,500 by system on 2026-06-01T00:00:00Z]
    - Alert sent to budget manager confirming new budget is in effect
```

---

### REQ-AMEND-002: Review Fiscal Impact of Amendment Before Approval

**Story:** Review fiscal impact of an amendment (demand: 771)

**Description:**
Before approving any amendment, the financial controller sees a clear analysis of how it affects the organization's financial position, including impact on available cash, whether other budgets must be reduced, and risk assessment.

**Acceptance Criteria:**

```gherkin
Scenario: Amendment impact report is auto-generated
  GIVEN a proposed amendment:
    - Budget: "IT Infrastructure"
    - Old amount: €180,000
    - New amount: €165,000 (€15,000 savings)
    - Reason: "Vendor consolidation achieved better pricing"
  WHEN the Financial Controller opens the amendment detail
  THEN they see an Impact Report showing:
    - Current fiscal year budget total: €1,500,000
    - Net impact of this amendment: -€15,000 (savings)
    - Percentage change: -0.83%
    - Cumulative amendments this year: +€10,000 (net)
    - Cash flow impact: "Available fund pool increases by €15,000"
    - Risk assessment: "LOW - Savings realization; no conflict with other budget increases"
    - Funding source impact (if multi-funded): breakdown by source

Scenario: Amendment blocked if conflicting approvals exist
  GIVEN a proposed amendment to increase budget "Marketing 2026" by €25,000
  AND a concurrent amendment to "Facilities 2026" is pending Director approval (requesting €30,000)
  WHEN the Financial Controller reviews total impact
  THEN they see a WARNING: "Organization budget pool already pledged €30,000 pending approval. Available room: €50,000. This amendment uses €25,000 total, leaving €25,000 for other approvals."

Scenario: Amendment impact includes historical baseline
  GIVEN an amendment to "Training" budget
  WHEN the Financial Controller reviews the amendment
  THEN they see comparison to prior year:
    - Last year "Training" budget: €40,000
    - This year approved: €45,000
    - This amendment proposes: €52,500
    - Year-over-year change: +31% vs. +12.5% baseline
    - Trend indicator: "Unusual spike - validate business case"
```

---

## Budget Holder Governance

### REQ-GOV-001: Budget Holder Policy Acknowledgement

**Story:** Record budget holder policy acknowledgement (demand: 531)

**Description:**
When a new budget holder is assigned or when policies change, the budget holder must acknowledge and commit to the budget policy (spending limits, approval requirements, variance thresholds, etc.). This acknowledgement is recorded electronically with date, signature, and is auditable.

**Acceptance Criteria:**

```gherkin
Scenario: Budget holder is assigned to budget
  GIVEN I am a Compliance Officer
  WHEN I create a new budget "Department X 2026" with budget holder "Jane Smith"
  THEN:
    - Jane Smith receives a notification: "You are now budget holder for 'Department X 2026'"
    - Notification includes a link to the Budget Holder Policy
    - Status shows: "Acknowledgement Required"

Scenario: Budget holder reviews and acknowledges policy
  GIVEN Jane Smith receives the notification
  WHEN she clicks "View Policy" and opens the budget detail page
  THEN she sees:
    - Budget policy document (embedded, with download link)
    - Key terms: spending limits, variance thresholds (±5%), approval process
    - Electronic acknowledgement form with fields:
      - "I confirm I have read and understand the budget policy"
      - Date reviewed
      - Signature (click-to-sign via integrated signing service, or name entry if signing unavailable)
      - Optional: Comments or questions
  AND when she clicks "I Acknowledge", then:
    - Record created: [Policy Acknowledgement for Budget XXX, signed by Jane Smith, 2026-05-21 14:35:00]
    - Budget status changes to "Acknowledged - Active"
    - Compliance Officer receives notification confirming acknowledgement

Scenario: Policy acknowledgement is immutable and auditable
  GIVEN a signed policy acknowledgement
  WHEN I access the budget's audit trail
  THEN I see:
    - Policy acknowledgement entry with date, budget holder name, timestamp
    - No edit capability (immutable)
    - Link to view the policy document that was acknowledged at that time
    - If policies are versioned: clear indication of which version was acknowledged
```

---

## Budget Analytics & Forecasting

### REQ-FORECAST-001: Estimate at Completion (EAC) with Confidence Range

**Story:** Forecast budget at completion based on current spend rate and remaining scope (demand: 712; Story 12)

**Description:**
The system calculates Estimate at Completion (EAC) using the burn rate (actual spend ÷ elapsed time) and applies it to the remaining period. A confidence range (±10%) is shown based on historical variance. If EAC exceeds budget, a "Budget Risk" flag is raised.

**Acceptance Criteria:**

```gherkin
Scenario: EAC is auto-calculated and displayed
  GIVEN a budget "Project Alpha":
    - Total budget: €100,000
    - Start date: 2026-01-01
    - End date: 2026-12-31
    - Current date: 2026-05-21 (20% of year elapsed)
    - Actual spend to date: €22,000 (22% of budget)
    - Historical variance for similar projects: 8%
  WHEN I open the Forecast panel
  THEN I see:
    - Burn rate: €22,000 ÷ 5.7 months = €3,860/month
    - Projected spend for remaining 6.3 months: €3,860 × 6.3 = €24,318
    - EAC (Estimate at Completion): €22,000 + €24,318 = €46,318
    - Variance vs. budget: €46,318 - €100,000 = -€53,682 (underrun by 53.7%)
    - Confidence range: ±8% = €42,613 to €50,023
    - Status: GREEN (well under budget)
    - Confidence indicator: 85% (based on historical data)

Scenario: EAC variance triggers budget risk flag
  GIVEN a budget "Project Beta":
    - Total budget: €80,000
    - Actual spend (May 21): €48,000 (60% spent, only 42% of year elapsed)
    - Burn rate: €48,000 ÷ 4.7 months = €10,213/month
    - Remaining 7.3 months projected: €10,213 × 7.3 = €74,555
    - EAC: €48,000 + €74,555 = €122,555
  WHEN I open the Forecast panel
  THEN:
    - EAC: €122,555 (EXCEEDS BUDGET by €42,555 or 53%)
    - RED flag: "Budget Risk - Forecast Over Budget"
    - Status indicator shows: "Over-run Likely by May 2027"
    - Recommendation: "Propose budget amendment or reduce scope by Sept 1"
    - Budget holder notified via alert

Scenario: EAC updates when new spend is recorded
  GIVEN an EAC was calculated on May 15 (showing green status)
  WHEN a large invoice (€15,000) is received and recorded on May 21
  THEN within 5 minutes:
    - Budget manager receives notification: "Budget status changed to AMBER"
    - EAC panel shows updated calculation
    - If EAC now exceeds budget, flag is raised
    - Link to "Propose Amendment" is displayed
```

---

### REQ-FORECAST-002: Forecast Budget Exhaustion Date

**Story:** Forecast contract budget exhaustion date (demand: 503; Story 3)

**Description:**
For budgets linked to contracts, the system projects when the budget will be fully spent based on current burn rate. This projection includes committed (approved but uninvoiced) spend and provides a warning if exhaustion occurs before the budget end date or contract expiration.

**Acceptance Criteria:**

```gherkin
Scenario: System calculates budget exhaustion date
  GIVEN a budget "Maintenance Contract - Building Facilities":
    - Total budget: €240,000
    - Start date: 2026-01-01
    - End date: 2026-12-31
    - Current date: 2026-05-21
    - Actual spend: €127,500
    - Committed (approved POs, not invoiced): €32,000
    - Total consumed: €159,500 (66%)
    - Months elapsed: 4.7
    - Average monthly burn: €159,500 ÷ 4.7 = €33,936/month
  WHEN I open the Exhaustion Forecast panel
  THEN I see:
    - Remaining budget: €80,500
    - Months remaining: 7.3
    - At current burn rate: €80,500 ÷ €33,936 = 2.4 months
    - Projected exhaustion date: 2026-08-01 (September 1)
    - Status: AMBER (budget ends Dec 31; exhaustion Oct 1)
    - Message: "Budget will be exhausted ~4 months early"
    - Recommendation: "Plan contract renewal by July 1 or request budget amendment"

Scenario: Exhaustion date changes when burn rate changes
  GIVEN the above budget showing exhaustion Aug 1
  WHEN the contract service provider implements cost-saving measures
  AND actual spend for May 21-31 is only €2,100 (vs. typical €3,400)
  THEN on June 1:
    - New burn rate calculated: €33,200/month (5% reduction)
    - New exhaustion date: Aug 15 (slightly later)
    - Trend indicator: "Burn rate improving - review monthly"

Scenario: Exhaustion warning when date approaches
  GIVEN projected exhaustion date of Aug 1
  WHEN current date is July 15
  THEN:
    - Budget detail page shows RED banner: "CRITICAL: Budget exhaustion expected in 17 days"
    - Budget holder receives daily alert
    - Recommendation to act: renew contract, amend budget, or reduce spending
    - Link to "Request Amendment" or "View Contract Renewal Options"
```

---

### REQ-REPORT-001: Budget vs Actual Comparison

**Story:** Budget vs forecast comparison (demand: 290; Story 18)

**Description:**
Executive dashboard displays side-by-side comparison of budget baseline, forecast (EAC), and actual spend for all active budgets. Variance is highlighted and drilled into by department, cost center, and budget category.

**Acceptance Criteria:**

```gherkin
Scenario: Budget vs Actual report summarizes all budgets
  GIVEN I am the Finance Director
  WHEN I open the Budget Variance Report
  THEN I see a table with all budgets:
    | Budget | Category | Total Budget | Actual Spend | % Spent | EAC | Variance | Status |
    |--------|----------|--------------|--------------|---------|-----|----------|--------|
    | Facilities | Operational | €250,000 | €127,500 | 51% | €220,000 | -€30,000 | GREEN |
    | Training | Personnel | €45,000 | €18,200 | 40% | €52,500 | +€7,500 | RED |
    | IT Infra | Capital | €180,000 | €89,400 | 50% | €178,000 | -€2,000 | GREEN |
  
  AND summary row:
    | TOTAL | | €1,500,000 | €687,300 | 46% | €1,510,000 | +€10,000 | AMBER |

Scenario: Drill down into budget variance by cost center
  GIVEN the variance report is open
  WHEN I click on "Facilities" budget row
  THEN I see breakdown by cost center:
    - CC-001 (Headquarters): €100,000 budget, €68,000 spent (68%)
    - CC-002 (Amsterdam Office): €100,000 budget, €47,000 spent (47%)
    - CC-003 (Regional): €50,000 budget, €12,500 spent (25%)
  AND drill down one more level to see individual purchase orders and invoices

Scenario: Export variance report
  GIVEN the variance report is displayed
  WHEN I click "Export as CSV"
  THEN a file is downloaded with:
    - One row per budget
    - Columns: Budget Name, Category, Total, Actual, Forecast, Variance, Status
    - Summary totals at bottom
    - Timestamp of when report was generated
```

---

## Authorization & Compliance

### REQ-AUTH-001: Role-based Budget Access Control

**Story:** Implicit in budget holder assignment and amendment approval (demand: 531, 604)

**Description:**
Budget visibility and action permissions are enforced by role: Budget Holders see their budgets, Managers see department budgets, Controllers see all budgets (read-only), Directors/CFO approve amendments.

**Acceptance Criteria:**

```gherkin
Scenario: Budget Holder sees only assigned budgets
  GIVEN Jane Smith is a Budget Holder for "Department X 2026" only
  WHEN she logs in and views the Budget list
  THEN she sees:
    - Only "Department X 2026" in her list
    - Cannot see "Finance 2026" or "IT Infrastructure 2026" (other departments)
    - Can view detail, create expenditure requests, see approval history
    - Cannot create amendments (only propose; CFO approves)

Scenario: Financial Controller sees all budgets (read-only)
  GIVEN I am a Financial Controller
  WHEN I open the Budget list
  THEN I see:
    - All organizational budgets
    - Cannot edit budget totals or create expenditure requests
    - Can review amendments and provide fiscal impact assessment
    - Can export, filter, and search
    - Can view all approval histories and audit trails

Scenario: Director can approve amendments across all departments
  GIVEN I am a Director/CFO
  WHEN an amendment is proposed for any budget
  THEN I receive:
    - Notification of pending approval
    - Access to amendment detail and fiscal impact
    - Option to approve, reject, or request changes
    - All approval decisions logged with my ID and timestamp
```

---

## Data Quality & Audit

### REQ-AUDIT-001: Amendment Audit Trail

**Story:** Implicit in amendment workflow and governance (demand: 531, 604)

**Description:**
Every amendment action (propose, review, approve, reject, execute) is recorded with date, actor, action, and status change. The audit trail is immutable and exportable for compliance.

**Acceptance Criteria:**

```gherkin
Scenario: Amendment actions are logged chronologically
  GIVEN an amendment has been created, reviewed, and approved
  WHEN I open the Audit Trail section of the amendment
  THEN I see:
    - [2026-05-15 10:30] Proposed by Jane Smith (budget holder)
      "Scope expansion: additional certification programs"
    - [2026-05-16 14:00] Reviewed by John Controls (Financial Controller)
      Recommendation: APPROVE - "Funds available, low risk"
    - [2026-05-17 09:15] Approved by Maria Director (CFO)
      Decision: APPROVE - "Endorsed by Finance, proceeding"
    - [2026-06-01 00:00] Executed by System
      "Effective date reached; budget updated to €52,500"
  AND each entry shows:
    - Exact timestamp
    - Actor's name and role
    - Action taken
    - Status before and after
    - Any notes or supporting comments

Scenario: Audit trail is immutable
  GIVEN an amendment audit trail exists
  WHEN I attempt to edit any entry
  THEN the system blocks the action and shows:
    - "Audit trail entries cannot be edited or deleted"
    - Message: "For compliance reasons, all amendment actions are permanently recorded"

Scenario: Export audit trail for external audit
  GIVEN an amendment's audit trail
  WHEN I click "Export Audit Trail"
  THEN I can download:
    - PDF report showing full amendment history
    - Format includes: amendment details, all actions chronologically, actors, timestamps, decisions
    - Digital signature or official seal (if org policy requires)
```

---

## Summary of All Requirements

| Requirement | Demand Score | Priority | Story | Status |
|-------------|--------------|----------|-------|--------|
| REQ-BUDGET-001 | 146 | P0 | Budget Creation | Defined |
| REQ-BUDGET-002 | 185 | P0 | Real-time Tracking | Defined |
| REQ-BUDGET-003 | 245 | P0 | Approval Integration | Defined |
| REQ-BUDGET-004 | 506 | P1 | Roll-up Reporting | Defined |
| REQ-AMEND-001 | 708 | P0 | Amendment Workflow | Defined |
| REQ-AMEND-002 | 771 | P0 | Fiscal Impact Review | Defined |
| REQ-GOV-001 | 531 | P1 | Policy Acknowledgement | Defined |
| REQ-FORECAST-001 | 712 | P1 | EAC Calculation | Defined |
| REQ-FORECAST-002 | 503 | P1 | Exhaustion Date | Defined |
| REQ-REPORT-001 | 290 | P2 | Budget Variance Report | Defined |
| REQ-AUTH-001 | 531+ | P0 | Role-based Access | Defined |
| REQ-AUDIT-001 | 531+ | P0 | Audit Trail | Defined |

---

## Traceability to Context-Brief Stories

- REQ-BUDGET-001 ← Story 1, 18
- REQ-BUDGET-002 ← Story 7, 18, 19
- REQ-BUDGET-003 ← Feature: Budget check integration
- REQ-BUDGET-004 ← Story 6, 9
- REQ-AMEND-001 ← Story 5, 25
- REQ-AMEND-002 ← Story 4, 21
- REQ-GOV-001 ← Story 8, 38
- REQ-FORECAST-001 ← Story 2, 7, 12, 14
- REQ-FORECAST-002 ← Story 3, 15
- REQ-REPORT-001 ← Story 18
- REQ-AUTH-001 ← Stakeholder: Budget Holder, Controller, Director
- REQ-AUDIT-001 ← Governance requirement across all amendments

