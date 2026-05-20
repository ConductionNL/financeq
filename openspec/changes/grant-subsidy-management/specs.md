# Specs: Grant & Subsidy Management — Shillinq

## ADDED Requirements

---

### Subsidy Scheme Management

---

#### REQ-SCH-001: Create and configure a new subsidy scheme

A grant administrator must be able to create a new subsidy scheme defining all rules and parameters required for consistent application of the scheme.

**Scenario: Create scheme with all parameters**

- **GIVEN** a grant administrator opens the new scheme form
- **WHEN** they complete the configuration with scheme name, legal basis, total budget, maximum grant per applicant, target group, purpose, application period, and required documents
- **THEN** the scheme is saved with status `draft` and all configured parameters are persisted

**Scenario: Preview scheme before publishing**

- **GIVEN** a scheme has been saved as `draft`
- **WHEN** the administrator clicks 'Preview portal view'
- **THEN** the scheme is displayed in portal format showing eligibility requirements, maximum grant, deadline, and required documents exactly as they will appear on the public portal

---

#### REQ-SCH-002: Publish subsidy scheme to public portal

An approved subsidy scheme must be publishable to the citizen-facing portal so eligible applicants can find and apply.

**Scenario: Publish approved scheme**

- **GIVEN** a scheme has been approved by the head of finance (status: `approved`)
- **WHEN** the administrator clicks 'Publish'
- **THEN** the scheme status changes to `published`, `isPublished` is set to `true`, `publishedDate` is recorded, and the scheme appears on the public portal with all scheme information, eligibility requirements, required documents, and application deadline

**Scenario: Application form activates on period open**

- **GIVEN** a scheme is published and `applicationPeriodStart` has been reached
- **WHEN** a citizen visits the public portal
- **THEN** the online application form is active and applicants can submit applications

**Scenario: Application form closes on deadline**

- **GIVEN** a published scheme with `applicationPeriodEnd` in the past
- **WHEN** the closing time is reached
- **THEN** the application form is automatically deactivated and any late submission attempt receives an explanation message with the closure time

---

#### REQ-SCH-003: Browse and filter available subsidy schemes (public portal)

A subsidy recipient must be able to browse all open subsidy schemes and filter by category and target group.

**Scenario: Browse published schemes**

- **GIVEN** multiple schemes with `isPublished: true` exist
- **WHEN** a visitor accesses the public subsidy portal (`GET /api/public/subsidy-schemes`)
- **THEN** only schemes with `isPublished: true` and an open application period are returned, each showing scheme name, purpose, maximum grant, deadline, and link to full description and application form

**Scenario: Filter schemes by category and target group**

- **GIVEN** multiple published schemes spanning different categories and target groups
- **WHEN** the visitor applies filters (e.g. category: 'duurzaamheid', targetGroup: 'mkb')
- **THEN** only schemes matching all selected filters are returned, with updated result count

---

### Subsidy Application Management

---

#### REQ-APP-001: Submit a subsidy application with supporting documents

An applicant organisation must be able to submit a subsidy application with required supporting documentation.

**Scenario: Submit application**

- **GIVEN** a published scheme with an open application period
- **WHEN** an applicant completes and submits the application form with all required documents attached
- **THEN** the application is created with status `submitted`, `submissionDate` is recorded, and the applicant receives a confirmation

**Scenario: Automatic sanctions check on submission**

- **GIVEN** a new subsidy application has been submitted
- **WHEN** the application submission is processed
- **THEN** the system automatically checks the applicant name and KvK number against EU Sanctions, OFAC, and the Dutch Rijksoverheid exclusion register; any match flags the application as `sanctions-hold` and notifies the integrity officer for manual review

---

#### REQ-APP-002: View risk-scored application list

An integrity officer must be able to view approved subsidy applications ranked by automated risk score to prioritise spot checks.

**Scenario: Load spot-check dashboard**

- **GIVEN** the integrity officer opens the spot-check dashboard
- **WHEN** the application list loads
- **THEN** applications are sorted descending by risk score with score, scheme name, applicant name, and granted amount visible per row

**Scenario: Risk score tooltip**

- **GIVEN** a risk score is displayed for an application
- **WHEN** the officer hovers over the score
- **THEN** a tooltip explains which risk factors contributed to the score (e.g. high amount, first-time applicant, flagged sector)

---

#### REQ-APP-003: Process grant application (assess and decide)

A subsidy case handler must be able to assess an application and record a decision outcome.

**Scenario: Approve application**

- **GIVEN** an application in status `under-review`
- **WHEN** the case handler sets the outcome to `approved` and confirms
- **THEN** the application status changes to `approved`, `reviewDate` is recorded, and a Grant record is created linked to this application

**Scenario: Reject application**

- **GIVEN** an application in status `under-review`
- **WHEN** the case handler sets the outcome to `rejected` with a reason
- **THEN** the application status changes to `rejected`, `reviewDate` is recorded, and a decision letter can be generated

---

#### REQ-APP-004: Generate decision letter from template

A subsidy case handler must be able to generate a formatted decision letter pre-filled with application data.

**Scenario: Generate pre-filled letter**

- **GIVEN** a case handler selects a decision outcome for an application
- **WHEN** they click 'Generate letter'
- **THEN** a letter is produced with applicant name, address, scheme name, decision, amount (if granted), and legal basis pre-filled from the application record

**Scenario: Edit letter before finalising**

- **GIVEN** a generated decision letter is displayed in preview
- **WHEN** the handler identifies an error in the pre-filled text
- **THEN** the letter text is editable before the handler finalises and sends it

---

#### REQ-APP-005: Track subsidy application status online

An applicant must be able to track the status of their submitted application.

**Scenario: View application status**

- **GIVEN** an applicant has submitted an application
- **WHEN** they open the application detail view in the portal
- **THEN** the current status (`submitted`, `under-review`, `approved`, `rejected`, etc.) and any status-change history is visible

---

#### REQ-APP-006: Check applicant against sanctions and debarment lists

An integrity officer must ensure applicants are automatically checked against sanctions lists before review proceeds.

**Scenario: Clean check allows review**

- **GIVEN** a new application has been submitted
- **WHEN** the automatic sanctions check returns no matches
- **THEN** the application proceeds to `under-review` status without interruption

**Scenario: Hit suspends application**

- **GIVEN** a new application has been submitted
- **WHEN** the automatic sanctions check returns a potential match
- **THEN** the application status is set to `sanctions-hold`, the integrity officer receives a Nextcloud notification, and the application cannot proceed until the integrity officer reviews and clears or confirms the flag

---

### Grant Management

---

#### REQ-GRT-001: Monitor grant spending

A subsidieverlener must be able to monitor how grantees spend their awarded funding.

**Scenario: View disbursement status**

- **GIVEN** an active grant
- **WHEN** the grant administrator views the grant detail
- **THEN** the total awarded amount, total disbursed to date, remaining balance, and a timeline of payment milestones are visible

---

#### REQ-GRT-002: Register and verify auditor statement for large subsidies

A grant administrator must be able to register and verify whether a required auditor statement has been submitted for large subsidies.

**Scenario: Auditor statement required above threshold**

- **GIVEN** a subsidy where `awardedAmount` exceeds the statutory threshold requiring an accountantsverklaring
- **WHEN** the administrator opens the accountability checklist for the grant
- **THEN** the system shows a required auditor statement item that must be completed before the accountability review can be finalised

**Scenario: Mark statement as verified**

- **GIVEN** an auditor statement document has been uploaded to the grant record
- **WHEN** the administrator marks it as verified
- **THEN** `isVerified` is set to `true`, `verificationDate`, auditor registration number, and opinion type are recorded on the AuditorStatement record

---

#### REQ-GRT-003: Block final payment if mandatory conditions are unmet

A financial administrator must have the system prevent a final payment if mandatory conditions are not yet met.

**Scenario: Block payment with outstanding conditions**

- **GIVEN** a subsidy grant where the accountability report is not yet approved
- **WHEN** the administrator attempts to process the final payment
- **THEN** the system blocks the payment and displays a checklist listing each outstanding condition that must be fulfilled before payment can proceed

**Scenario: Allow payment when conditions met**

- **GIVEN** all mandatory conditions for a grant have been fulfilled (accountability report approved, all declarations signed)
- **WHEN** the administrator processes the final payment
- **THEN** the payment is processed without interruption

---

#### REQ-GRT-004: Close subsidy dossier after final payment

A financial administrator must have the subsidy dossier automatically closed and archived after the final payment is confirmed.

**Scenario: Auto-close on payment execution**

- **GIVEN** a grant's final payment has been confirmed by the bank
- **WHEN** the payment status updates to `executed`
- **THEN** the grant status changes to `closed`, a `closureTimestamp` is recorded, and the dossier moves to the read-only archive

---

#### REQ-GRT-005: Identify overpayment after accountability review

A financial administrator must have the system automatically flag a subsidy as overpaid when disbursements exceed the approved final amount.

**Scenario: Detect overpayment**

- **GIVEN** a grant where `totalDisbursed` exceeds the approved final amount determined during accountability review
- **WHEN** the accountability review is finalised
- **THEN** the system sets the grant status to `overpaid`, calculates the `recoveryAmount` (disbursed minus approved), and creates a recovery task linked to the grant

---

#### REQ-GRT-006: Receive grant accountability (send accountability request)

A grant administrator must be able to automatically send an accountability request to subsidy recipients when the project end date is reached.

**Scenario: Accountability request on project end**

- **GIVEN** a subsidy award is active and `projectEndDate` is reached
- **WHEN** the accountability deadline trigger fires
- **THEN** the system sends a formal accountability request to the registered recipient contact person with the deadline and list of required documents

**Scenario: Reminder 14 days before deadline**

- **GIVEN** an accountability request has been sent and 14 days remain before the deadline with no report received
- **WHEN** the 14-day check runs
- **THEN** the system sends a reminder to the recipient and notifies the grant administrator

---

#### REQ-GRT-007: Track Single Information Single Audit (SiSa) grants

A grant administrator must be able to identify and track grants that are eligible for SiSa reporting.

**Scenario: Flag SiSa-eligible grant**

- **GIVEN** a grant is created under a SiSa-eligible scheme (`isSISAEligible: true`)
- **WHEN** the grant is saved
- **THEN** `isSISAEligible` on the grant is set to `true` and the grant appears in the SiSa grants report

**Scenario: SiSa export**

- **GIVEN** multiple grants with `isSISAEligible: true`
- **WHEN** the administrator generates the SiSa export
- **THEN** a structured export is produced containing all required SiSa fields for the current reporting year

---

#### REQ-GRT-008: Identify open commitments at year-end

A municipal controller must be able to see all open subsidy commitments at year-end for accurate jaarrekening reporting.

**Scenario: Open commitments report**

- **GIVEN** the year-end closing date has been reached
- **WHEN** the controller runs the open commitments report
- **THEN** the system lists all grants where `status` is `active` and final payment has not been made, showing committed amounts and expected payment dates

---

#### REQ-GRT-009: Lock financial year after reconciliation sign-off

A municipal controller must be able to lock the financial year after reconciliation sign-off to prevent retrospective changes.

**Scenario: Lock year**

- **GIVEN** a reconciliation has been approved by the controller
- **WHEN** the controller applies the year-end lock for the specified year
- **THEN** all grants and payments in the closed year become read-only, and any attempt to modify them returns an error message referencing the financial year lock

---

### Grant Portfolio Management

---

#### REQ-PTF-001: View real-time subsidy portfolio compliance dashboard

A financial administrator must be able to view a dashboard showing all outstanding advance payments and compliance status.

**Scenario: Advance payments dashboard**

- **GIVEN** multiple subsidy grants with advance payments issued
- **WHEN** the administrator opens the advance payments dashboard
- **THEN** a list sorted by `advancePaymentDueDate` is shown, with overdue items highlighted using a distinct visual indicator

**Scenario: Portfolio compliance overview**

- **GIVEN** multiple GrantPortfolio records with varying `complianceStatus`
- **WHEN** the administrator opens the portfolio overview
- **THEN** each portfolio shows `complianceStatus`, `concentrationRiskLevel`, `totalGrantValue`, and the date of `lastAuditDate`

---

#### REQ-PTF-002: Monitor subsidy concentration risk per recipient

A financial administrator must be able to identify portfolios with high concentration of grants to a single recipient.

**Scenario: High concentration risk flag**

- **GIVEN** a portfolio where a single organisation holds grants exceeding a defined threshold percentage of `totalGrantValue`
- **WHEN** the portfolio is loaded or recalculated
- **THEN** `concentrationRiskLevel` is set to `high` and the administrator is notified

---

#### REQ-PTF-003: Report on decision outcomes by scheme

A subsidy team lead must be able to see grant, rejection, and withdrawal rates per scheme for a reporting period.

**Scenario: Outcomes report by scheme**

- **GIVEN** the team lead opens the outcomes report with a scheme and period selected
- **WHEN** the report is generated
- **THEN** counts and percentages for granted, rejected, withdrawn, and still-open applications are shown for the selected scheme and period

**Scenario: Export outcomes to CSV**

- **GIVEN** the outcomes report is displayed
- **WHEN** the team lead clicks 'Export'
- **THEN** a CSV file is downloaded with one row per application including scheme, applicant, decision, amount, and date

---

### Accountability Management

---

#### REQ-ACT-001: Complete online accountability report form

A subsidy applicant must be able to complete their accountability report online via a structured form.

**Scenario: Step-by-step accountability form**

- **GIVEN** an active subsidy grant with an open accountability period
- **WHEN** the grantee logs in and opens the accountability task
- **THEN** a step-by-step form is displayed pre-filled with grant details, asking for actual costs, activities completed, and outcomes achieved

---

#### REQ-ACT-002: Receive accountability deadline reminder

A subsidy applicant must receive email reminders before their accountability deadline.

**Scenario: 30-day reminder**

- **GIVEN** an active grant with an accountability deadline 30 days away and no report submitted
- **WHEN** the daily reminder job runs
- **THEN** an email is sent to the registered contact person containing the deadline date and a direct link to the accountability form

**Scenario: 7-day reminder**

- **GIVEN** an active grant with an accountability deadline 7 days away and no report submitted
- **WHEN** the daily reminder job runs
- **THEN** an email is sent to the registered contact person containing the deadline date and a direct link to the accountability form

---

### Integrity & Fraud Management

---

#### REQ-INT-001: Revoke delegated access granted by offboarded user

An administrator must be able to revoke all delegated access rights granted by a user who has left the organisation, without affecting other users' permissions.

**Scenario: Revoke offboarded user's delegations**

- **GIVEN** a user account is deactivated or offboarded
- **WHEN** the administrator initiates the delegation revocation for the offboarded user
- **THEN** all access delegations granted by that user are revoked, other users' access is unaffected, and an audit trail entry is created for each revoked delegation

---

#### REQ-INT-002: Grant temporary access for substitute

A user must be able to grant a temporary substitute access to their cases and dossiers during a period of absence.

**Scenario: Grant temporary access**

- **GIVEN** a user needs to assign a substitute during absence
- **WHEN** the user configures temporary access with a start date, end date, and substitute person
- **THEN** the substitute gains access to the specified dossiers for the defined period, and access is automatically revoked when the end date passes

---

#### REQ-INT-003: Open a formal fraud investigation case

An integrity officer must be able to open a formal fraud investigation case linked to a specific grant or applicant.

**Scenario: Create fraud investigation case**

- **GIVEN** a suspected fraud signal (internal report, data anomaly, or external tip)
- **WHEN** the integrity officer opens a fraud investigation case and links it to a grant or application
- **THEN** a confidential case record is created with access restricted to the investigation team, the linked grant(s) have a payment hold applied, and the investigation is tracked separately from the regular dossier workflow

---

#### REQ-INT-004: Assess grant eligibility

A case handler must be able to formally assess and record the eligibility of an applicant against scheme criteria.

**Scenario: Record eligibility assessment**

- **GIVEN** an application in `under-review` status
- **WHEN** the case handler completes the eligibility assessment form
- **THEN** each eligibility criterion is recorded as met or not met, with notes, and the overall assessment result is saved on the application record

---

### Training & Subsidy Registration

---

#### REQ-TRN-001: Register subsidy-funded training (SLIM, ESF)

A Learning & Development Coordinator must be able to tag training activities as subsidy-funded and record the source and amount for reporting.

**Scenario: Tag training as subsidy-funded**

- **GIVEN** a training activity is being added or edited
- **WHEN** the coordinator toggles 'Subsidy funded'
- **THEN** fields for subsidy programme, grant reference, approved amount, and conditions become visible and required

**Scenario: Include in subsidy report**

- **GIVEN** a subsidised training activity has been marked as completed
- **WHEN** the coordinator marks it as complete
- **THEN** the system automatically includes it in the subsidy reporting export for the relevant programme

**Scenario: Generate subsidy report**

- **GIVEN** the subsidy reporting period has closed
- **WHEN** the coordinator generates the subsidy report
- **THEN** the report shows all funded activities, participant counts, actual costs, and subsidy amount claimed for the period

---

### Process Grant Reclaim

---

#### REQ-RCL-001: Process subsidy reclaim and monitor repayment

A financial administrator must be able to initiate and track recovery of overpaid subsidy amounts.

**Scenario: Initiate recovery after overpayment detection**

- **GIVEN** a grant has been flagged as `overpaid` with a calculated `recoveryAmount`
- **WHEN** the administrator initiates the reclaim process
- **THEN** a formal recovery notice is generated, a repayment schedule is created, and repayment tracking is enabled on the grant record

**Scenario: Monitor repayment progress**

- **GIVEN** an active reclaim with a repayment schedule
- **WHEN** the administrator views the grant detail
- **THEN** the amount recovered to date, outstanding balance, and next expected payment date are visible

---

### Misuse & Recovery

---

#### REQ-MSU-001: Recover misused grants

A grant administrator must be able to initiate formal recovery proceedings for grants that have been misused.

**Scenario: Flag grant for misuse recovery**

- **GIVEN** a fraud investigation has concluded that a grant was misused
- **WHEN** the administrator marks the grant for misuse recovery
- **THEN** the grant status changes to `revoked`, a formal recovery order is generated, and the linked fraud investigation case is updated with the recovery decision

---

### Model Algemene Uitkering

---

#### REQ-MAU-001: Model Algemene Uitkering scenarios

A municipal controller must be able to model different Algemene Uitkering distribution scenarios for budget planning.

**Scenario: Create AU scenario**

- **GIVEN** the controller opens the AU scenario modelling tool
- **WHEN** they configure parameters (distribution year, scenario assumptions, population factors)
- **THEN** the modelled distribution amounts are calculated and displayed per cost driver, exportable for use in the municipal budget

---

### System & Compliance

---

#### REQ-SYS-001: Health check and metrics endpoint

The application must expose standard health and metrics endpoints per ADR-006.

**Scenario: Health check**

- **GIVEN** the application is running
- **WHEN** `GET /api/health` is called (public)
- **THEN** a JSON response is returned with `status: ok` and OpenRegister connectivity confirmed

**Scenario: Prometheus metrics**

- **GIVEN** an admin user calls `GET /api/metrics`
- **WHEN** the request is authenticated
- **THEN** Prometheus text-format metrics are returned including `shillinq_health_status`, `shillinq_info`, active grants count, pending applications count, and overdue accountability count

---

#### REQ-SYS-002: Dutch and English translations

All user-visible strings must be available in Dutch and English per ADR-007.

**Scenario: Language switch**

- **GIVEN** a user has Dutch (nl) set as their Nextcloud language
- **WHEN** they view any grant management screen
- **THEN** all labels, button texts, status values, and notifications are displayed in Dutch

---

#### REQ-SYS-003: WCAG AA accessibility

All screens must meet WCAG 2.1 AA accessibility requirements per ADR-010.

**Scenario: Keyboard navigation**

- **GIVEN** a user navigates using keyboard only (Tab, Enter, Escape)
- **WHEN** they open any grant management view
- **THEN** all interactive elements are reachable and operable via keyboard, focus is visible at all times, and no information is conveyed by colour alone
