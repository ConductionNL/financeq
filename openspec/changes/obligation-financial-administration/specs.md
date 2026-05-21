# Specifications: Obligation & Financial Administration — Shillinq

## REQ-OFA-001: Obligation CRUD Operations

**Category:** Core | **Priority:** P0 | **Complexity:** Medium

### REQ-OFA-001-A: Create Obligation from Invoice

**Scenario:** Finance officer creates an obligation record from an invoice

**GIVEN** an unpaid invoice in the system with invoiceNumber="INV-2026-0001", invoiceDate="2026-05-10", dueDate="2026-06-09", grossAmount=2500.00

**WHEN** the finance officer selects "Create Obligation" from the invoice detail view

**THEN**
- The system creates an Obligation record with:
  - obligationNumber auto-generated (format: OBL-YYYY-NNN)
  - obligationDate set to today's date
  - dueDate copied from invoice.dueDate
  - amount copied from invoice.grossAmount with currency
  - creditor copied from invoice.creditor
  - obligationType set to "invoice"
  - status set to "open"
- An audit trail entry records the creation with user identity and timestamp
- An ObligationTask is auto-generated with title="Pay {invoiceNumber}" and priority calculated based on days-to-deadline

### REQ-OFA-001-B: Create Obligation Manually

**Scenario:** Finance officer creates a purchase order obligation without an invoice

**GIVEN** the obligation creation form is open

**WHEN** the officer enters:
- obligationNumber (manual or auto-generated)
- obligationDate
- dueDate
- amount + currency
- creditor (search via Organization register)
- obligationType="purchase-order"
- description (optional)

**AND** clicks Save

**THEN**
- The obligation is stored in OpenRegister with all required fields
- An ObligationTask is auto-generated with priority based on dueDate
- The obligation appears in the "My Obligations" list with status "open"
- Return-to-form link shows confirmation

### REQ-OFA-001-C: Read Obligation Detail

**Scenario:** Finance officer views a single obligation

**GIVEN** an obligation exists with obligationNumber="OBL-2026-001"

**WHEN** the officer clicks on the obligation in the list

**THEN**
- A detail page loads showing:
  - Obligation header: obligationNumber, status badge, dueDate with days-to-deadline
  - Details section: obligationDate, amount (formatted with currency), creditor link, obligationType, description
  - Invoice link (if linked)
  - Payments section (table: date, amount, method, status)
  - Settlement Decision section (status, decision document link)
  - ObligationTask list (tasks generated for this obligation)
  - Audit trail tab (via CnObjectSidebar)
- All linked objects are clickable (invoice, creditor, tasks)

### REQ-OFA-001-D: Update Obligation

**Scenario:** Finance officer corrects an obligation amount or dueDate

**GIVEN** an obligation in "draft" status

**WHEN** the officer clicks Edit, changes a field (e.g., dueDate from "2026-06-09" to "2026-06-15"), and saves

**THEN**
- The field is updated
- The audit trail records the change with before/after values
- If dueDate changed, the ObligationTask is recalculated for priority
- Status remains "draft" until first save

### REQ-OFA-001-E: Delete Obligation

**Scenario:** Finance officer deletes a draft obligation

**GIVEN** an obligation in "draft" status with no payments or settlement decisions

**WHEN** the officer clicks Delete and confirms

**THEN**
- The obligation is deleted from the register
- The audit trail records the deletion
- Associated ObligationTasks are also deleted
- A confirmation message shows "Obligation deleted"

---

## REQ-OFA-002: Obligation Dashboard & Monitoring

**Category:** Core | **Priority:** P1 | **Complexity:** Medium

### REQ-OFA-002-A: Obligation KPI Cards

**Scenario:** Finance manager views obligation summary on dashboard

**GIVEN** multiple obligations exist with various statuses and due dates

**WHEN** the manager views the dashboard

**THEN** Four KPI cards display:
1. **Open Obligations:** Count of obligations with status="open" or "in-progress"
2. **Overdue Obligations:** Count with dueDate < today and status != "settled"
3. **Total Outstanding Value:** Sum of amount for all open obligations, formatted with currency
4. **Compliance Rate:** Percentage of obligations settled on-time in the current month (from ComplianceReport)

Each card shows:
- Current value (large, color-coded: overdue=red, ok=green)
- Trend arrow (↑/↓) vs. previous month
- Click-through to filtered list

### REQ-OFA-002-B: Obligation List Grouped by Deadline

**Scenario:** Finance officer monitors obligations by deadline urgency

**GIVEN** obligations with various due dates:
- OBL-2026-001: due 2026-05-20 (3 days overdue)
- OBL-2026-002: due 2026-05-28 (overdue by 0 days)
- OBL-2026-003: due 2026-05-25 (5 days from now)
- OBL-2026-004: due 2026-06-15 (21 days from now)

**WHEN** the manager views the dashboard obligation list

**THEN** Obligations are grouped in sections:
1. **Overdue** (red heading)
   - OBL-2026-001: Inkt & Papier BV | €2,500 | 3 days overdue
   - OBL-2026-002: Amsterdam RE | €5,000 | due today
2. **Due This Week** (yellow heading)
   - OBL-2026-003: Office Furniture | €1,200 | 5 days remaining
3. **Due Later** (green heading)
   - OBL-2026-004: Services | €800 | 21 days remaining

Each row shows: obligationNumber, creditor name, amount (currency-formatted), days-to-deadline or overdue indicator.
Clicking a row navigates to obligation detail.

### REQ-OFA-002-C: Obligation Index with Filters

**Scenario:** Finance officer finds obligations matching criteria

**GIVEN** the obligation index page is open with 50+ obligations

**WHEN** the officer applies filters:
- Status: "open" (show only unsettled)
- Creditor: "Amsterdam Real Estate BV" (auto-complete search)
- Amount Range: €1,000 - €10,000
- Due Date: "This month" or specific date range

**THEN**
- The list updates to show 3 obligations matching all filters
- Pagination shows 3 of 50 total
- Columns are sortable: obligationNumber, creditor, amount, dueDate, obligationType, status
- Export button (CSV/JSON) includes filtered results

---

## REQ-OFA-003: Settlement Decision Workflow

**Category:** Core | **Priority:** P1 | **Complexity:** High

### REQ-OFA-003-A: Create Settlement Decision

**Scenario:** Finance director issues a formal settlement decision for a batch of paid obligations

**GIVEN**
- Multiple obligations have been paid (e.g., OBL-2026-001, OBL-2026-002, OBL-2026-003)
- Each obligation is linked to a Payment record with paymentDate < dueDate (on-time)
- The director is authorized to issue settlement decisions

**WHEN** the director navigates to "Settlement Decisions" → "Create New" → selects 3 obligations

**AND** enters:
- decisionNumber (auto-generated, e.g., SETTLEMENT-2026-001)
- decisionDate (today)
- decisionRationale="Approved for payment in batch run May 23"

**AND** clicks Save

**THEN**
- A SettlementDecision record is created with:
  - decisionNumber, decisionDate, issuedBy (current user), totalSettledAmount (sum of selected obligations), obligationCount=3
  - Status="draft"
- An approval chain is triggered (via ApprovalChain integration)
- A notification is sent to the CFO requesting approval
- The decision appears in "My Decisions" with status="pending-approval"

### REQ-OFA-003-B: Approve & Finalize Settlement Decision

**Scenario:** CFO approves a settlement decision

**GIVEN** a SettlementDecision with status="pending-approval" is waiting for CFO sign-off

**WHEN** the CFO views the decision and clicks "Approve"

**THEN**
- Status changes to "approved"
- For each linked obligation:
  - status changes to "settled"
  - settledOnTime boolean is set based on (paymentDate < dueDate)
- A formal decision document is generated (PDF) with:
  - Decision number, date, issuer name, authorized approver name
  - Table of obligations: obligationNumber, creditor, amount, dueDate, paymentDate, on-time status
  - Legal statement: "Finalized under Dutch financial governance standards (BBV)"
- Document URL is stored in documentUrl field
- Audit trail records: decision creation, approval, finalization with user identities and timestamps
- ComplianceReport metrics are updated to include the newly settled obligations

---

## REQ-OFA-004: Obligation Task Management & AI Generation

**Category:** AI | **Priority:** P2 | **Complexity:** High

### REQ-OFA-004-A: Auto-Generate ObligationTask on Creation

**Scenario:** An ObligationTask is automatically generated when an obligation is created

**GIVEN** an obligation is created with dueDate="2026-06-09" and amount=€2,500

**WHEN** the system saves the obligation

**THEN** a background job (ObligationTaskService) triggers and creates an ObligationTask with:
- taskNumber auto-generated (e.g., TASK-OBL-2026-001)
- title="Pay invoice INV-2026-0001 to {creditor}" (derived from obligation)
- description=obligation.description
- dueDate=obligation.dueDate
- priority calculated as:
  - daysToDeadline = dueDate - today
  - if daysToDeadline < 0: priority="critical"
  - if daysToDeadline = 0-3: priority="high"
  - if daysToDeadline = 4-7: priority="medium"
  - if daysToDeadline > 7: priority="low"
- status="open"
- aiGenerated=true
- obligationId linked to the obligation

### REQ-OFA-004-B: Task Priority Recalculation

**Scenario:** An obligation's dueDate is updated, affecting task priority

**GIVEN** an ObligationTask with:
- dueDate="2026-06-09" (5 days away)
- priority="medium"

**WHEN** the linked obligation's dueDate is changed to "2026-05-25" (2 days away)

**AND** the system updates the obligation

**THEN** the ObligationTaskService recalculates:
- daysToDeadline=2
- priority changes to "high"
- Task is updated with new priority
- Audit trail records the priority change

### REQ-OFA-004-C: Task Completion & Obligation Settlement

**Scenario:** Finance officer marks an ObligationTask as completed when payment is made

**GIVEN** an ObligationTask with status="open" is linked to OBL-2026-001

**WHEN** a Payment is recorded against the obligation

**THEN** the ObligationTask status changes to "completed"
AND the obligation status changes to "settled"
AND the system updates settledOnTime based on (paymentDate < dueDate)

---

## REQ-OFA-005: Fixed Asset Management

**Category:** Core | **Priority:** P1 | **Complexity:** Medium

### REQ-OFA-005-A: Register Fixed Asset

**Scenario:** Asset manager registers a newly acquired asset

**GIVEN** the fixed asset registration form is open

**WHEN** the manager enters:
- assetNumber (auto-generated, e.g., FA-2026-001)
- name="Tesla Model 3"
- assetType="vehicle"
- purchaseDate="2026-01-15"
- purchaseCost="€65,000.00"
- status="active"
- location="Amsterdam Office - Parking Lot 2"

**AND** clicks Save

**THEN**
- A FixedAsset record is created in OpenRegister
- An audit trail entry records the creation
- The asset appears in the "Assets" list
- A DepreciationSchedule is auto-generated (see REQ-OFA-005-B)

### REQ-OFA-005-B: Auto-Generate Depreciation Schedule

**Scenario:** A depreciation schedule is automatically created for a registered asset

**GIVEN** a FixedAsset is created with:
- assetType="vehicle"
- purchaseDate="2026-01-15"
- purchaseCost="€65,000"

**WHEN** the system processes the asset registration

**THEN** a DepreciationSchedule is auto-generated with:
- scheduleNumber auto-generated (e.g., DS-2026-001)
- name derived from asset (e.g., "Tesla Model 3 - Linear Depreciation")
- startDate=purchaseDate
- endDate=startDate + useful-life-years (e.g., 5 years for vehicles → 2031-01-15)
- depreciationMethod="linear" (default; can be configured by asset type)
- annualRate calculated (useful-life-based; vehicles=20%, equipment=10%)
- totalDepreciationAmount=purchaseCost
- status="active"

### REQ-OFA-005-C: View Depreciation Board

**Scenario:** Finance officer views the depreciation schedule for an asset

**GIVEN** a FixedAsset with an active DepreciationSchedule

**WHEN** the officer views the asset detail page

**THEN** a "Depreciation" section shows a table:
| Year | Annual Depreciation | Accumulated Depreciation | Book Value |
|------|---------------------|--------------------------|------------|
| 2026 | €13,000 | €13,000 | €52,000 |
| 2027 | €13,000 | €26,000 | €39,000 |
| 2028 | €13,000 | €39,000 | €26,000 |
| 2029 | €13,000 | €52,000 | €13,000 |
| 2030 | €13,000 | €65,000 | €0 |

The table is auto-calculated based on depreciationMethod and annualRate.

---

## REQ-OFA-006: Compliance Reporting

**Category:** Analytics | **Priority:** P2 | **Complexity:** High

### REQ-OFA-006-A: Generate Compliance Report

**Scenario:** Compliance officer generates a quarterly compliance report

**GIVEN** the "Compliance Reports" view is open and user has report-generation permission

**WHEN** the officer selects:
- reportPeriod="2026-Q1" (April 1 - June 30, 2026)
- clicks "Generate Report"

**THEN** a background job (ComplianceService) calculates:
- totalObligations=count of all obligations with obligationDate in Q1 (15)
- onTimeObligations=count of obligations where paymentDate <= dueDate (14)
- overdueObligations=count of obligations where paymentDate > dueDate (1)
- complianceRate=(14/15) × 100 = 93.3%
- totalAmount=sum of all obligation amounts (€125,000)
- averagePaymentDays=average of (paymentDate - dueDate) for all obligations = -2.5 days (early)
- A ComplianceReport record is created with:
  - reportPeriod="2026-Q1"
  - generatedDate=today
  - All calculated metrics
  - powerBiUrl (initially empty; populated if integration configured)

### REQ-OFA-006-B: Export Compliance Report to PowerBI

**Scenario:** Compliance officer exports a report to Power BI for dashboard visualization

**GIVEN** a ComplianceReport exists for 2026-Q1

**WHEN** the officer clicks "Export to PowerBI"

**AND** PowerBI integration is configured (PowerBI API token in settings)

**THEN**
- The system calls PowerBI API to create a dataset with:
  - Metric headers: reportPeriod, complianceRate, onTimeObligations, overdueObligations
  - Drill-down dimensions: obligationType (invoice, PO, contract), creditor, paymentMethod
- A powerBiUrl is generated (link to PowerBI dashboard)
- The URL is stored in ComplianceReport.powerBiUrl
- A confirmation message shows "Exported to PowerBI"
- Officer can click powerBiUrl to view the dashboard

### REQ-OFA-006-C: Compliance Report History

**Scenario:** Finance manager views trends in compliance performance

**GIVEN** multiple ComplianceReports exist for previous periods (2025-Q4, 2026-Q1, 2026-Q2)

**WHEN** the manager views the "Compliance Reports" list

**THEN** a table shows:
| Period | Compliance Rate | Total Obligations | On-Time | Overdue | Avg Days | Action |
|--------|-----------------|-------------------|---------|---------|----------|--------|
| 2026-Q2 | 95% | 18 | 17 | 1 | -1.2 | View |
| 2026-Q1 | 93.3% | 15 | 14 | 1 | -2.5 | View |
| 2025-Q4 | 91% | 22 | 20 | 2 | +0.8 | View |

A line chart visualizes complianceRate trend over time.
Clicking "View" loads the detailed report with drill-downs.

---

## REQ-OFA-007: Invoice Integration

**Category:** Core | **Priority:** P1 | **Complexity:** Medium

### REQ-OFA-007-A: Link Invoice to Obligation

**Scenario:** Finance officer links an existing invoice to an obligation

**GIVEN** an invoice (INV-2026-0001) and an obligation (OBL-2026-001) both exist

**WHEN** the officer clicks "Link Invoice" in the obligation detail and selects the invoice

**THEN**
- The obligation is linked to the invoice via invoiceId relation
- The invoice detail page shows a linked obligation section
- The obligation detail page shows the linked invoice with clickable link
- Audit trail records the link creation

---

## REQ-OFA-008: Authorization & RBAC

**Category:** Security | **Priority:** P0 | **Complexity:** Medium

### REQ-OFA-008-A: Role-Based Access Control

**Scenario:** Different users have different obligations-related permissions

**GIVEN** three users:
- alice (Finance Officer): can view/create/edit obligations, no settlement authority
- bob (Finance Manager): can view/create/edit/settle obligations, approve up to €10,000
- charlie (CFO): can view/create/edit/settle/issue decisions, no approval limit

**WHEN** each user logs in and accesses the obligation module

**THEN** the UI shows:
- **alice:** View list, create form, edit own obligations, no settle/decision buttons
- **bob:** View list, create form, edit all obligations, settle button (up to €10,000), no decision button
- **charlie:** Full access including decision issuance

Authorization is enforced server-side via AuthorizationService; UI buttons hidden/shown based on returned permissions.

---

## REQ-OFA-009: Audit Trail & Compliance

**Category:** Compliance | **Priority:** P1 | **Complexity:** Low

### REQ-OFA-009-A: Automatic Audit Trail

**Scenario:** Every change to an obligation is logged for compliance

**GIVEN** an obligation is created and then updated

**WHEN** the finance officer views the audit trail tab

**THEN** entries show:
1. Created 2026-05-10 14:22 by alice: obligationNumber="OBL-2026-001", dueDate="2026-06-09"
2. Updated 2026-05-11 09:15 by alice: dueDate changed from "2026-06-09" to "2026-06-15"
3. Settled 2026-05-20 16:45 by bob: status changed from "open" to "settled", settledOnTime=true

Each entry shows: timestamp, user identity, operation type, before/after values for changed fields.

---

## REQ-OFA-010: Multi-Currency Support

**Category:** Core | **Priority:** P2 | **Complexity:** Medium

### REQ-OFA-010-A: Obligation Amount Normalization

**Scenario:** Obligations in different currencies are tracked and reported

**GIVEN** obligations in multiple currencies:
- OBL-EUR-001: €2,500
- OBL-USD-001: $2,500 (≈ €2,250)
- OBL-GBP-001: £2,500 (≈ €2,950)

**WHEN** generating a ComplianceReport

**THEN** the system:
- Displays each obligation with its original currency (€, $, £)
- Stores totalAmount in both original currencies and normalized EUR (for reporting)
- Compliance calculations use original amounts (no currency conversion for settlement metrics)
- Reports clearly label currency for each obligation

---

## REQ-OFA-011: Import & Export

**Category:** Bulk Operations | **Priority:** P2 | **Complexity:** Medium

### REQ-OFA-011-A: Bulk Import Obligations (CSV)

**Scenario:** Finance team imports 100 obligations from a legacy system via CSV

**GIVEN** a CSV file with columns: obligationNumber, obligationDate, dueDate, amount, currency, creditorName, obligationType, description

**WHEN** the officer clicks "Mass Import" → selects the CSV → reviews field mapping → clicks "Import"

**THEN**
- ImportService validates each row:
  - obligationNumber is unique
  - obligationDate and dueDate are valid dates
  - amount is numeric and positive
  - creditorName matches an existing Organization (or creates new if allowed)
- Progress bar shows import status
- 98 obligations are imported successfully
- 2 rows with errors (invalid creditor) are shown in error summary
- Officer can download error report (CSV) and retry
- A notification confirms "98 obligations imported"

### REQ-OFA-011-B: Bulk Export Obligations (CSV/JSON)

**Scenario:** Finance officer exports filtered obligations for analysis

**GIVEN** the obligation list with 50 items filtered by status="open" and amount > €1,000

**WHEN** the officer clicks "Export" → selects "CSV" → clicks "Download"

**THEN**
- ExportService exports only the filtered 12 obligations to CSV with columns:
  - obligationNumber, obligationDate, dueDate, amount, currency, creditor, obligationType, status, daysToDeadline
- Numeric columns are formatted with thousands separator and currency symbol
- The file is downloaded as "obligations-2026-05-21.csv"

---

## REQ-OFA-012: Search & Discovery

**Category:** UX | **Priority:** P2 | **Complexity:** Medium

### REQ-OFA-012-A: Full-Text Search Obligations

**Scenario:** Finance officer searches for obligations by invoice number or creditor

**GIVEN** the obligation list is open and indexed via IndexService

**WHEN** the officer types "INV-2026" in the search box

**THEN**
- The system performs full-text search across obligationNumber, description, invoiceNumber, creditor.name
- Results include OBL-2026-001 linked to INV-2026-0001
- Search results are ranked by relevance (exact obligationNumber > description > creditor name)
- Results update in real-time as the officer types (debounced)

---

## REQ-OFA-013: Dutch Government Compliance

**Category:** Compliance | **Priority:** P1 | **Complexity:** High

### REQ-OFA-013-A: SiSa Compliance Export

**Scenario:** Finance officer exports obligations for SiSa (Systeem Informatievoorziening Subsidieverstrekking) compliance

**GIVEN** a ComplianceReport for Q1 2026 with settlement decisions

**WHEN** the officer clicks "Export for SiSa" → selects reporting format

**THEN**
- A structured XML/JSON export is generated per SiSa schema:
  - Organization tax ID
  - Obligation records: obligationNumber, obligationDate, dueDate, amount, creditor tax ID
  - Settlement decision records: decisionNumber, decisionDate, authorized person
  - Audit trail records: change log with timestamp, user, operation
- File is downloaded as "sisa-export-Q1-2026.xml"
- System logs the export (audit trail)

---

## REQ-OFA-014: Dashboard Widgets

**Category:** UX | **Priority:** P2 | **Complexity:** Medium

### REQ-OFA-014-A: Obligation Overview Widget

**Scenario:** Finance manager adds obligation status widget to home dashboard

**GIVEN** the Nextcloud dashboard is open

**WHEN** the manager clicks "Add Widget" → searches "Obligations" → selects "Obligation Overview"

**THEN** a draggable widget appears showing:
- Pie chart: % on-time vs. overdue
- KPI: Compliance rate (93%)
- Mini list: Top 3 urgent obligations (due today/overdue)
- Link to full obligation list

Widget updates every 15 minutes (configurable).

---

## Test Scenarios

All GIVEN/WHEN/THEN scenarios above are verified via:
- **Unit tests:** ObligationService, SettlementService, ComplianceService (PHPUnit)
- **Integration tests:** API endpoints (Newman/Postman)
- **Browser tests:** Obligation workflows end-to-end (Playwright)
- **Security tests:** Authorization enforcement, audit trail integrity
