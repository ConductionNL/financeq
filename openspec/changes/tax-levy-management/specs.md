# Specifications: Tax & Levy Management

**Change ID:** tax-levy-management  
**Last Updated:** 2026-05-21

## Functional Requirements

### REQ-TAX-001: VAT Return Preparation with MTD Compliance

**Story:** VAT return preparation with MTD (Making Tax Digital) compliance for UK

**Description:** System must generate and validate VAT returns for submission to tax authorities. Returns aggregate transaction data by VAT category (standard, reduced, zero, reverse, exempt) and calculate net liability.

**Acceptance Criteria:**

**GIVEN** a fiscal period (month or quarter)  
**WHEN** a user navigates to "VAT Returns" and selects the period  
**THEN** the system displays a draft VAT return showing:
  - Collected VAT (sum of sales at each rate category)
  - Paid VAT (sum of deductible expenses at each rate category)
  - Net VAT (payable if positive, refundable if negative)
  - Count of transactions contributing to each category

**GIVEN** a VAT return in draft status  
**WHEN** the user clicks "Approve"  
**THEN** the status changes to "approved" and the return is locked for editing

**GIVEN** an approved VAT return  
**WHEN** the user clicks "Submit"  
**THEN** the system:
  - Generates an XBRL instance document (NTA7 format)
  - Validates the XBRL against NTA7 schema rules
  - Calls the Dutch Tax Authority API to submit
  - Records the submission timestamp and authority acknowledgment
  - Changes status to "submitted"

**GIVEN** MTD compliance is enabled  
**WHEN** a VAT return is submitted  
**THEN** the system enforces that:
  - All transactions are digitally signed
  - Audit trail is complete and tamper-proof
  - No amendments can be made to submitted returns (only new amendments filed)

---

### REQ-TAX-002: Multi-Jurisdiction Tax Compliance

**Story:** Multi-jurisdiction VAT — VAT returns generated per jurisdiction for all obligations

**Description:** System supports separate VAT return workflows for each jurisdiction (NL, UK, DE, BE, etc.). Each jurisdiction may have different reporting periods, rates, and submission requirements.

**Acceptance Criteria:**

**GIVEN** an organization operating in multiple jurisdictions  
**WHEN** a user navigates to "VAT Returns"  
**THEN** the system displays VAT returns grouped by jurisdiction with filters for:
  - Jurisdiction (dropdown: NL, UK, DE, BE, etc.)
  - Reporting period (monthly, quarterly, annually per jurisdiction)
  - Status (draft, approved, submitted, acknowledged)

**GIVEN** a VAT return is created for jurisdiction "UK"  
**WHEN** the system generates the return  
**THEN** it applies UK-specific:
  - VAT standard rate (20% for UK vs. 21% for NL)
  - Reporting period (quarterly for UK MTD)
  - XBRL taxonomy (UK-specific if applicable)
  - Submission endpoint (HMRC vs. Belastingdienst)

**GIVEN** a transaction is tagged with jurisdiction "DE"  
**WHEN** a VAT return is generated  
**THEN** the transaction is included only in the DE jurisdiction VAT return

---

### REQ-TAX-003: Separate Chart of Accounts per Administration

**Story:** Separate chart of accounts per administration — each with own VAT settings and fiscal year.

**Description:** Organizations with multiple administrations (brands, divisions, subsidiaries) can maintain separate tax configurations, COAs, and VAT settings for each.

**Acceptance Criteria:**

**GIVEN** an organization with two administrations: "Shillinq BV" and "Shillinq Consulting"  
**WHEN** a user creates a VAT return  
**THEN** the user must select which administration the return belongs to

**GIVEN** administration "Shillinq Consulting" is selected  
**WHEN** the VAT return is generated  
**THEN** it includes only transactions tagged to that administration and applies that administration's:
  - Fiscal year (e.g., calendar year vs. April–March)
  - VAT settings (standard rate, exemptions, reverse charge rules)
  - Chart of accounts (GL account mappings)

**GIVEN** two administrations each with different fiscal years  
**WHEN** year-end close occurs  
**THEN** each administration can close independently with its own fiscal year end date

---

### REQ-TAX-004: VAT Audit Trail for Tax Defense

**Story:** Maintain VAT audit trail — complete audit trail for all VAT transactions.

**Description:** Every transaction contributing to a VAT return must have a complete, immutable audit trail recording who changed what and when. This is mandatory for defending a return if audited.

**Acceptance Criteria:**

**GIVEN** a VAT transaction is created or modified  
**WHEN** the change is saved  
**THEN** the audit trail records:
  - User who made the change
  - Timestamp of change
  - Before/after values (if modified)
  - System-generated reason (if automated)

**GIVEN** a VAT return is submitted to authorities  
**WHEN** an auditor requests the audit trail  
**THEN** the system exports:
  - All transactions included in the return
  - Full change history for each transaction
  - PDF summary showing data integrity signatures (if MTD enabled)
  - Export is certified/signed by the organization

**GIVEN** a transaction is disputed in an audit  
**WHEN** the user opens the transaction detail  
**THEN** the audit trail tab shows:
  - Original creation date and user
  - All modifications with timestamps and users
  - Linked supporting documents (invoices, receipts)
  - Any exemption certificates applied

---

### REQ-TAX-005: Annual Income Statement (Jaaropgave) for Employees

**Story:** View annual income statement (jaaropgave) — employees access statements for annual tax returns.

**Description:** Employees can download their annual income statement (jaaropgave) showing total fiscal income, withheld wage tax, and employer details in the Belastingdienst-required format.

**Acceptance Criteria:**

**GIVEN** it is after the jaaropgave generation date (February 28 in the current year)  
**WHEN** an employee navigates to "Annual Statements"  
**THEN** the current year's jaaropgave is available for download as PDF

**GIVEN** an employee downloads the jaaropgave  
**WHEN** they open the PDF  
**THEN** it contains:
  - Total fiscal income (sum of salary payments for the year)
  - Total wage tax withheld
  - Employee name, BSN, employer name, and KVK
  - Period covered (January 1 – December 31)
  - Employer signature or digital stamp
  - Format matches Belastingdienst requirements (IB33 or loonaangifte format)

**GIVEN** an employee's salary changed mid-year (e.g., promotion on June 1)  
**WHEN** the jaaropgave is generated  
**THEN** it reflects:
  - Gross salary from January–May at old rate
  - Gross salary from June–December at new rate
  - Proportional wage tax withheld for each period

---

### REQ-TAX-006: Corporate Tax Return Preparation (VPB)

**Story:** Prepare corporate tax return (VPB-aangifte) for BV — accountant prepares return with fiscal adjustments.

**Description:** Accountants can prepare corporate income tax returns (VPB) with fiscal adjustments (depreciation, intercompany transactions, R&D credits) and export to SBR format for electronic filing.

**Acceptance Criteria:**

**GIVEN** the commercial annual accounts are finalized  
**WHEN** an accountant navigates to "Tax Returns" and selects "VPB Preparation"  
**THEN** the system displays:
  - Commercial profit from the P&L
  - Input fields for fiscal adjustments (depreciation, intercompany, credits)
  - Calculated taxable profit = commercial profit ± adjustments
  - Effective tax rate preview

**GIVEN** fiscal adjustments are entered (e.g., depreciation add-back 50,000 EUR)  
**WHEN** the accountant clicks "Calculate Taxable Profit"  
**THEN** the system:
  - Applies each adjustment
  - Calculates cumulative taxable income
  - Applies deductions (loss carryforward, R&D credit)
  - Shows final taxable profit and estimated tax liability

**GIVEN** the VPB return is approved  
**WHEN** the accountant clicks "Export for Filing"  
**THEN** the system generates:
  - SBR/XBRL XML file in NTA7 format
  - PDF summary for manual review
  - Both are packaged for submission to Belastingdienst
  - Download link provided

---

### REQ-TAX-007: Tax Rate Management with Council Approval

**Story:** Review tax rate proposal before council vote — visibility into proposed rates with comparisons.

**Description:** Municipal administrators can propose new tax rates, review them with revenue projections, submit to council for voting, and publish approved rates.

**Acceptance Criteria:**

**GIVEN** a municipal tax administrator accesses "Tax Rate Management"  
**WHEN** they click "+ Propose New Rates"  
**THEN** they see a form for:
  - Fiscal year (e.g., 2027)
  - Tax categories (OZB, water board levy, etc.)
  - Proposed rate for each category
  - Optional revenue projection (based on historical properties)

**GIVEN** a rate proposal is submitted  
**WHEN** a council member navigates to "Rate Proposals"  
**THEN** they see a summary table with:
  - Tax category name
  - Current rate (previous year)
  - Proposed rate (new proposal)
  - % change (e.g., +2%)
  - Projected annual revenue impact
  - Head of Finance's recommendation (approve/reject/modify)

**GIVEN** the council votes during a meeting  
**WHEN** the compliance officer records the vote outcome as "Adopted"  
**THEN** the system requests:
  - Raadsbesluit number (council decision number)
  - Adoption date
  - Effective date (usually January 1 of new fiscal year)

**GIVEN** the rates are recorded as adopted  
**WHEN** the compliance officer clicks "Publish"  
**THEN** the system:
  - Locks the rates for editing
  - Updates all tax modules to use the new rates
  - Sets effective date (January 1 of new year)
  - Logs the change with raadsbesluit reference

---

### REQ-TAX-008: Monthly Payroll Tax Declaration (Loonaangifte)

**Story:** Submit monthly wage declaration to Belastingdienst — generate and file loonaangifte.

**Description:** Payroll administrators can generate monthly wage declarations and submit them to the Dutch Tax Authority (Belastingdienst) in the required XML format.

**Acceptance Criteria:**

**GIVEN** a monthly payroll run is finalized  
**WHEN** the payroll administrator navigates to "Payroll Tax" and selects the month  
**THEN** the system displays:
  - Employee count in payroll
  - Total gross wages
  - Total wage tax withheld
  - Social security contributions (if applicable)
  - One-time/extra payments (if any)

**GIVEN** the payroll is verified and complete  
**WHEN** the administrator clicks "Generate Loonaangifte"  
**THEN** the system creates:
  - XML file in Belastingdienst-specified format
  - File contains: employee names, BSNs, gross wages, withheld tax, social security per employee
  - File is timestamped and ready for submission

**GIVEN** the loonaangifte file is ready  
**WHEN** the administrator clicks "Submit to Belastingdienst"  
**THEN** the system:
  - Submits the XML via Belastingdienst API (with digital signature if required)
  - Receives submission acknowledgment with reference number
  - Records the submission timestamp and reference in the payroll run record
  - Changes loonaangifte status to "submitted"

**GIVEN** a payroll correction is needed after submission  
**WHEN** the administrator files an amendment  
**THEN** a new loonaangifte is generated with amended employee data marked as corrections

---

### REQ-TAX-009: Tax Exemption Certificate Management

**Story:** Tax exemption certificate validation and application.

**Description:** Organizations can upload and manage tax exemption certificates (research, export, environmental) and apply them to transaction categorization.

**Acceptance Criteria:**

**GIVEN** an organization has a research exemption certificate  
**WHEN** a user navigates to "Exemptions" and uploads the certificate (PDF)  
**THEN** the system records:
  - Certificate number (from document or manual entry)
  - Certificate type (research, export, environmental, etc.)
  - Issue date and expiry date
  - Link to uploaded document

**GIVEN** an exemption certificate is stored  
**WHEN** a transaction is classified as exempt  
**THEN** the system:
  - Links the transaction to the applicable exemption certificate
  - Tags the transaction as exempt
  - Excludes it from standard VAT calculation (0% VAT)

**GIVEN** an exemption certificate is about to expire (30 days before)  
**WHEN** the system runs a daily audit  
**THEN** a notification is sent to tax administrator:
  - Certificate expiring soon
  - Current linked transactions (will lose exemption if not renewed)
  - Renewal instructions

**GIVEN** an exemption certificate expires  
**WHEN** a new VAT return is generated  
**THEN** transactions previously linked to that certificate:
  - Revert to standard VAT calculation (21%)
  - Trigger a review flag (audit trail notes the change)

---

### REQ-TAX-010: XBRL Export for Dutch Tax Authority

**Story:** Generate XBRL/SBR documents for Belastingdienst submission.

**Description:** System can export tax data (VAT, BCF, corporate) to XBRL format (NTA7 or SBR-NT) for submission to Dutch tax authorities.

**Acceptance Criteria:**

**GIVEN** a VAT return is approved and ready for submission  
**WHEN** the user clicks "Export as XBRL"  
**THEN** the system:
  - Selects the appropriate taxonomy (NTA7-2026 for VAT)
  - Maps tax data to XBRL concepts (Assets, Liabilities, Revenue, etc.)
  - Generates XML instance document with facts and dimensions
  - Validates against taxonomy rules

**GIVEN** an XBRL instance is generated  
**WHEN** validation is performed  
**THEN** the system reports:
  - Valid: XBRL can be submitted
  - Invalid: List of validation errors (missing facts, schema violations)
  - Warned: Warnings that don't block submission (e.g., unusual values)

**GIVEN** the XBRL is valid  
**WHEN** the user submits to Belastingdienst  
**THEN** the system:
  - Signs the XBRL digitally (if MTD enabled)
  - Transmits via secure Belastingdienst API endpoint
  - Stores a copy of the submitted XBRL in audit files
  - Records the submission timestamp and response code

---

## Non-Functional Requirements

### REQ-TAX-NFR-001: Performance

- VAT return aggregation completes in <2 seconds for up to 10,000 transactions per period
- XBRL generation completes in <5 seconds for documents with up to 50,000 facts
- Tax rate lookups use caching (Redis) for <100ms response time

### REQ-TAX-NFR-002: Audit & Compliance

- All tax transactions are immutably logged with before/after snapshots (AuditTrailService)
- Audit trail exports are cryptographically signed if MTD compliance is enabled
- Tax records retained for minimum 7 years (ArchivalService with legal hold)

### REQ-TAX-NFR-003: Accessibility

- All tax forms and dashboards are WCAG AA compliant
- Color not used as sole conveyor of information (e.g., red for overdue, green for on-time)
- Forms are keyboard-navigable with proper ARIA labels
- Responsive design works from 320px to 1920px

### REQ-TAX-NFR-004: Data Isolation

- Multi-tenancy: each organization's tax data is isolated by organization_id
- Multi-administration: transactions tagged to specific administrations within an organization
- Jurisdiction isolation: VAT returns scoped to jurisdiction and are not cross-contaminated

### REQ-TAX-NFR-005: API Reliability

- Belastingdienst API calls use exponential backoff retry on temporary failures
- Failed submissions are queued and retried every hour for 24 hours
- Timeout: 30 seconds for external API calls; fallback to manual submission if needed

---

## Edge Cases & Constraints

### EC-TAX-001: Negative VAT (Refund Scenarios)

**GIVEN** paid VAT exceeds collected VAT in a period  
**WHEN** the VAT return is generated  
**THEN** netAmount is negative (refund amount)
- System allows marking as "refund requested" (status = pending-refund)
- Belastingdienst may take 4–8 weeks to process refund
- Audit trail tracks when refund is received

### EC-TAX-002: Mid-Year Tax Rate Change

**GIVEN** VAT standard rate changes from 21% to 22% on July 1, 2026  
**WHEN** a half-year VAT return is generated (Jan–June)  
**THEN** transactions are split:
- Jan–June: 21% standard rate
- July–Dec: 22% standard rate
- Return aggregates correctly using effective date of each rate

### EC-TAX-003: Fiscal Year Mismatch

**GIVEN** an organization with fiscal year April–March  
**WHEN** a calendar-year VAT return (Jan–Dec) is requested  
**THEN** system allows both:
- Fiscal-year VAT return (Apr 2025–Mar 2026)
- Calendar-year VAT return (Jan–Dec 2026)
- Transactions are tagged with their calendar date; user selects period

### EC-TAX-004: Amendment After Submission

**GIVEN** a VAT return is submitted and acknowledged  
**WHEN** an error is discovered (missing transaction, incorrect rate)  
**THEN** system prevents editing the submitted return:
- User must file a formal "amendment return" (aanvulling/correctie)
- Amendment return references original return number
- Original return remains immutable in audit trail

### EC-TAX-005: Exemption Certificate Overlap

**GIVEN** two exemption certificates cover overlapping dates  
**WHEN** a transaction falls within both periods  
**THEN** system allows user to:
- Select which certificate to apply, or
- Apply neither (use standard VAT), or
- Flag for manual review

---

## Deduplication Check

**Verified no overlap with:**
- ✅ ObjectService (CRUD handled by OpenRegister)
- ✅ AuditTrailService (mandatory for tax defense — using platform service)
- ✅ FileService (certificate documents — using platform service)
- ✅ ArchivalService (7-year retention — using platform service)
- ✅ CnFormDialog (schema-driven forms — using platform component)
- ✅ CnDetailPage (detail views — using platform component)
- ✅ CnDashboardPage (dashboard widgets — using platform component)

**Domain-specific logic (not duplicated):**
- Tax calculation (VAT aggregation by category)
- XBRL serialization and validation
- Belastingdienst API integration
- Tax rate effective date management
- Multi-jurisdiction VAT logic

**Conclusion:** No duplication found. Implementation uses OpenRegister core; custom code is scoped to tax-specific business logic.

---

## Spec Traceability

- @spec openspec/changes/tax-levy-management/proposal.md (features, demand, stakeholders)
- @spec openspec/changes/tax-levy-management/design.md (schemas, architecture, seed data)
- @spec openspec/changes/tax-levy-management/tasks.md (implementation tasks)

All PHPDoc tags in code will reference: `@spec openspec/changes/tax-levy-management/tasks.md#task-N`
