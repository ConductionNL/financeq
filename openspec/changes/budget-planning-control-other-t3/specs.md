# Specs: Budget Planning & Control — Shillinq — Other T3

Requirements follow the `REQ-XXX-NNN` format. Each requirement has at least one GIVEN/WHEN/THEN scenario.

---

## Feature: Consolidate budgets

### REQ-CON-001: Create and manage ConsolidationGroups

A user with financial-manager or admin role must be able to create, edit, and delete ConsolidationGroups.

**Scenario 1 — Create a ConsolidationGroup**

GIVEN I am logged in as a user with financial-manager role  
WHEN I navigate to Consolidatiegroepen and click "Toevoegen"  
THEN a `CnFormDialog` opens pre-populated from the ConsolidationGroup schema  
AND I can enter name, description, fiscalYear, and status  
AND saving creates a new ConsolidationGroup object via `ObjectService.saveObject()`  
AND the new group appears in the list immediately

**Scenario 2 — Edit a ConsolidationGroup**

GIVEN a ConsolidationGroup exists in status `active`  
WHEN I open its detail page and click "Bewerken"  
THEN I can modify name, description, and status  
AND changes are persisted via `ObjectService.saveObject()`  
AND the audit trail records the before/after snapshots automatically

**Scenario 3 — Accessibility**

GIVEN the ConsolidationGroup form is open  
WHEN I navigate using only keyboard (Tab, Enter, Escape)  
THEN all form fields are reachable and operable  
AND the form meets WCAG AA contrast requirements (ADR-010)

---

### REQ-CON-002: Link Budgets to a ConsolidationGroup

A user must be able to add and remove Budgets from a ConsolidationGroup via the relation mechanism.

**Scenario 1 — Add a Budget**

GIVEN I am on the ConsolidationGroupDetail page  
WHEN I open the "Gekoppelde budgetten" card and add a Budget via the relation picker  
THEN the Budget is linked using the OpenRegister relation mechanism (register + schema + objectId)  
AND NO foreign key is stored  
AND the Budget appears in the linked budgets table

**Scenario 2 — Remove a Budget**

GIVEN a Budget is linked to a ConsolidationGroup  
WHEN I remove the relation from the detail page  
THEN the relation is deleted but the Budget object itself is unaffected  
AND the group's ConsolidatedReports are NOT automatically regenerated (stale indicator shown)

**Scenario 3 — Empty group validation**

GIVEN a ConsolidationGroup has no linked Budgets  
WHEN I click "Genereer Rapport"  
THEN the backend returns HTTP 422 with message "ConsolidationGroup has no linked Budgets"  
AND the frontend displays an error notification (no `window.alert()`)

---

### REQ-CON-003: Generate a ConsolidatedReport

The system must aggregate totals across all Budgets linked to a ConsolidationGroup and produce a ConsolidatedReport.

**Scenario 1 — Successful generation**

GIVEN a ConsolidationGroup with three linked Budgets (amounts: 100 000, 200 000, 150 000; spend: 95 000, 185 000, 140 000)  
WHEN I click "Genereer Rapport" on the ConsolidationGroupDetail page  
THEN the system calls `POST /api/consolidated-reports/generate` with the groupId and fiscalYearId  
AND `ConsolidationService::generateReport()` sums: totalBudget = 450 000, totalSpend = 420 000, variance = 30 000  
AND a ConsolidatedReport is created with status `draft` and reportDate = today  
AND the report appears in the "Geconsolideerde rapporten" section of the detail page

**Scenario 2 — Authorization**

GIVEN I am logged in as a regular user without financial-manager or admin role  
WHEN I call `POST /api/consolidated-reports/generate`  
THEN the backend returns HTTP 403  
AND `IGroupManager::isAdmin()` is the enforcement mechanism (ADR-005, ADR-015)  
AND no ConsolidatedReport is created

**Scenario 3 — Idempotency**

GIVEN a ConsolidatedReport in status `draft` already exists for a group and fiscal year  
WHEN I click "Genereer Rapport" again  
THEN the existing draft is replaced (not duplicated) by the new calculation  
AND the audit trail records both versions

---

### REQ-CON-004: Finalize a ConsolidatedReport

An admin or financial-manager must be able to set a ConsolidatedReport to `final` status, after which it is locked for editing.

**Scenario 1 — Finalize**

GIVEN a ConsolidatedReport is in status `draft`  
WHEN I click "Definitief maken"  
THEN the status transitions to `final` via `ObjectService.saveObject()`  
AND the "Bewerken" button is hidden on the detail page  
AND the "Exporteren" button becomes active

**Scenario 2 — No editing after finalization**

GIVEN a ConsolidatedReport is in status `final`  
WHEN I attempt to edit it via the API (`PUT /api/consolidated-reports/{id}`)  
THEN the backend returns HTTP 422 with message "Finalized reports cannot be edited"

---

### REQ-CON-005: Filter and search ConsolidatedReports

Users must be able to filter the ConsolidatedReport list by FiscalYear and status.

**Scenario 1 — Filter by FiscalYear**

GIVEN multiple ConsolidatedReports exist across two fiscal years  
WHEN I select a FiscalYear in the filter bar  
THEN only ConsolidatedReports linked to that fiscal year are shown  
AND the total count updates accordingly

**Scenario 2 — Filter by status**

GIVEN ConsolidatedReports in both `draft` and `final` status exist  
WHEN I filter by status = `final`  
THEN only finalized reports are shown  
AND draft reports are hidden

**Scenario 3 — Full-text search**

GIVEN ConsolidatedReports exist with different names  
WHEN I type part of a report name in the search bar  
THEN only matching reports are returned  
AND the search is handled by `IndexService` (no custom search endpoint)

---

### REQ-CON-006: Export a ConsolidatedReport

A finalized ConsolidatedReport must be exportable as CSV or Excel.

**Scenario 1 — Export**

GIVEN a ConsolidatedReport is in status `final`  
WHEN I click "Exporteren" and select CSV format  
THEN `CnMassExportDialog` is shown with format selector  
AND the export is produced by `ExportService` (no custom export controller)  
AND the downloaded file contains: name, totalBudget, totalSpend, variance, reportDate, status

---

## Feature: Set savings goals

### REQ-SAV-001: Create and manage SavingsOpportunities

A user with financial-manager or admin role must be able to create, edit, and delete SavingsOpportunities.

**Scenario 1 — Create a SavingsOpportunity**

GIVEN I am on the Besparingsdoelen list page  
WHEN I click "Toevoegen"  
THEN a `CnFormDialog` opens with fields: name, description, budget (relation picker), targetAmount, currentAmount, targetDate, status, responsibleUser  
AND all fields are labelled and keyboard-navigable (ADR-010)  
AND saving creates the object via `ObjectService.saveObject()`

**Scenario 2 — Required fields validation**

GIVEN I open the SavingsOpportunity create form  
WHEN I submit without filling name, targetAmount, targetDate, or budget  
THEN the form shows validation errors per field  
AND no object is created

**Scenario 3 — Translations**

GIVEN the user's Nextcloud language is set to Dutch  
WHEN the SavingsOpportunity form is displayed  
THEN all labels, placeholder texts, and button captions are rendered in Dutch via `t(appName, 'key')`  
AND no Dutch strings are hardcoded in the Vue template (ADR-007)

---

### REQ-SAV-002: Track progress on a SavingsOpportunity

The system must compute and display progress toward the savings target.

**Scenario 1 — Progress calculation**

GIVEN a SavingsOpportunity with targetAmount = 48 000 and currentAmount = 31 500  
WHEN I open the detail page  
THEN progressPercent is displayed as 65.6% (rounded to one decimal)  
AND a `CnProgressBar` shows the visual representation  
AND amounts are formatted per user locale (Dutch: "€ 31.500,00")

**Scenario 2 — Update currentAmount**

GIVEN a SavingsOpportunity in status `active`  
WHEN I edit currentAmount from 31 500 to 48 000  
THEN progressPercent updates to 100%  
AND the status automatically transitions to `achieved`  
AND a Nextcloud notification is sent to responsibleUser via `NotificationService`

**Scenario 3 — Overshoot**

GIVEN a SavingsOpportunity with targetAmount = 48 000  
WHEN currentAmount is set to 52 000 (overshoot)  
THEN progressPercent is capped at 100% in the display  
AND status is set to `achieved`  
AND the raw values (targetAmount, currentAmount) are stored as entered

---

### REQ-SAV-003: Lifecycle transitions for SavingsOpportunity

SavingsOpportunities must support explicit status transitions.

**Scenario 1 — Mark as achieved manually**

GIVEN a SavingsOpportunity in status `active`  
WHEN I click "Markeer als behaald"  
THEN the status transitions to `achieved`  
AND a Nextcloud notification is dispatched to responsibleUser  
AND the button is replaced by a `CnStatusBadge` showing "Behaald"

**Scenario 2 — Cancel a goal**

GIVEN a SavingsOpportunity in status `active`  
WHEN I click "Annuleren" and confirm in the `NcDialog` confirmation dialog  
THEN the status transitions to `cancelled`  
AND the goal no longer appears in the "Actieve doelen" dashboard widget

**Scenario 3 — Achieved goals are immutable**

GIVEN a SavingsOpportunity in status `achieved`  
WHEN I attempt to set status back to `active` via the API  
THEN the backend returns HTTP 422 with message "Achieved goals cannot be reopened"  
AND no status change is persisted

---

### REQ-SAV-004: Dashboard widget — savings goals progress

The Shillinq dashboard must show a savings goals summary widget.

**Scenario 1 — Widget displays**

GIVEN at least one SavingsOpportunity exists  
WHEN I open the Shillinq dashboard  
THEN the "Besparingsdoelen voortgang" widget shows:
- Count of active goals
- Total targetAmount across active goals
- Total currentAmount realized across active goals
- Overall progress percentage

**Scenario 2 — Status distribution chart**

GIVEN SavingsOpportunities exist in active, achieved, and cancelled statuses  
WHEN the widget renders  
THEN a `CnChartWidget` (donut chart) shows the distribution by status with counts  
AND color is not the sole means of distinguishing statuses (WCAG AA, ADR-010)

**Scenario 3 — Empty state**

GIVEN no SavingsOpportunities exist  
WHEN the widget renders  
THEN `CnEmptyState` is shown with a call-to-action to create the first savings goal  
AND no JavaScript error is thrown

---

### REQ-SAV-005: Dashboard widget — consolidated budget KPI

The dashboard must show a consolidated budget KPI for the current fiscal year.

**Scenario 1 — KPI renders**

GIVEN at least one final ConsolidatedReport exists for the current FiscalYear  
WHEN I open the Shillinq dashboard  
THEN the "Consolidatie KPI" `CnStatsBlock` shows: totalBudget, totalSpend, variance amount, variance percentage  
AND amounts are formatted per user locale

**Scenario 2 — No data state**

GIVEN no final ConsolidatedReport exists for the current FiscalYear  
WHEN the widget renders  
THEN `CnEmptyState` is shown with message "Geen definitief rapport beschikbaar voor dit boekjaar"

---

## General requirements

### REQ-GEN-001: Authorization enforcement

All mutation operations (create, update, delete, generate, transition) must be protected by backend authorization checks.

**Scenario 1 — Admin check on backend**

GIVEN any POST, PUT, or DELETE request to a Shillinq endpoint for ConsolidationGroup, ConsolidatedReport, or SavingsOpportunity  
WHEN the request is made by a user without financial-manager or admin role  
THEN the backend returns HTTP 403  
AND `IGroupManager::isAdmin()` is called on the backend (not just frontend)  
AND no data mutation occurs

---

### REQ-GEN-002: Audit trail

All changes to ConsolidationGroup, ConsolidatedReport, and SavingsOpportunity must be fully auditable.

**Scenario 1 — Change recorded**

GIVEN a ConsolidatedReport is updated (e.g. status draft → final)  
WHEN I open the "Auditspoor" tab in `CnObjectSidebar`  
THEN the change is listed with: timestamp, user UID, before-value, after-value  
AND this is handled by `AuditTrailService` automatically (no custom logging code)

---

### REQ-GEN-003: Metrics endpoint

The `/api/metrics` endpoint must include counters for the new entities.

**Scenario 1 — Metrics include new entities**

GIVEN the app is running  
WHEN an admin calls `GET /api/metrics`  
THEN the response includes:
- `shillinq_consolidation_groups_total` (count)
- `shillinq_consolidated_reports_total` (count, labeled by status)
- `shillinq_savings_opportunities_total` (count, labeled by status)

---

### REQ-GEN-004: Internationalisation

All user-visible strings must be translatable. Dutch (nl) and English (en) translations must be provided.

**Scenario 1 — No hardcoded strings**

GIVEN any Vue template or PHP response message in this change  
WHEN a developer runs the pre-commit check (ADR-015)  
THEN zero hardcoded Dutch or English strings are found outside `l10n/*.json` and `t()` calls

**Scenario 2 — Dutch locale**

GIVEN the user's Nextcloud language is `nl`  
WHEN they use the consolidation or savings goal features  
THEN all labels, notifications, status badges, and error messages appear in Dutch
