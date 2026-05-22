---
status: draft
version: 1.0
---

# Driver-Based Forecasting — Implementation Tasks

## Phase 1: Foundation & Data Layer

### Task 1.1: Schema Definitions & OpenRegister Setup
- [ ] Create OpenRegister schema files:
  - `financeq/openspec/specs/driver-based-forecasting/schemas/Driver.json` (with validation rules for provenance, refreshFrequency)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/DriverForecast.json` (with dataPoints structure)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/CostRevenueFormula.json` (with formula DSL field)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/Calibration.json` (with metrics: rmse, mape, r2)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/Scenario.json` (with parameterOverrides structure)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/ForecastRun.json` (with outputs, inputSnapshots)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/Variance.json` (with attribution decomposition)
  - `financeq/openspec/specs/driver-based-forecasting/schemas/RollingForecastSchedule.json` (with schedule config)
- [ ] Register all 8 schemas in OpenRegister's central registry
- [ ] Implement schema migrations (if upgrading from prior version)
- [ ] Create seed data file: `lib/Settings/driver_based_forecasting_register.json` with 5 drivers, 3 formulas, 3 scenarios, 2 calibrations, 1 schedule
  - Use `x-openregister.type: "mock"` for seed objects
  - Use `@self` envelope for all seed objects (register, schema, slug)
- [ ] Test: seed data import is idempotent (re-importing does not create duplicates)

### Task 1.2: Core Backend Services - Formula DSL Parser & Validator
- [ ] Implement DSL tokenizer: `FormulaLexer` (tokenize volume, kosten_per_eenheid, indexering, policy_override, operators, numbers, parentheses)
- [ ] Implement DSL parser: `FormulaParser` (recursive descent parser, build AST, respect operator precedence)
- [ ] Implement DSL type checker: `FormulaTypeChecker`
  - Validate all `volume(driverCode)` references against existing Driver registry
  - Validate `kosten_per_eenheid(constantName)` against Calibration constants (or formula definition history)
  - Validate `indexering(indexCode, component)` against known index table
  - Detect `policy_override(...)` references in scope
  - Check result type compatibility with targetType
  - Flag divide-by-zero patterns
- [ ] Implement DSL suggestion engine for unknown drivers (Levenshtein distance, return top 3 suggestions)
- [ ] Test suite:
  - Parse and validate 50+ formula examples (covering all DSL primitives, arithmetic, nesting)
  - Verify error messages for unknown drivers, type mismatches, divide-by-zero
  - Benchmark: tokenize + parse + validate 1000 formulas in < 100ms

### Task 1.3: Core Backend Services - Formula Engine (ForecastRun Executor)
- [ ] Implement `ForecastEngine` service:
  - Execute a ForecastRun: fetch Scenario, Formulas, Drivers; resolve inheritance; merge overrides
  - For each formula, for each period in horizon: evaluate DSL, produce forecast value
  - Capture driver contributions (sensitivity analysis)
  - Record input snapshots (frozen Driver, Formula, Calibration state)
  - Populate ForecastRun.outputs
- [ ] Implement override resolution:
  - Flatten scenario inheritance tree
  - Detect and report conflicts
  - Merge driver-volume and formula-constant overrides
- [ ] Implement confidence propagation (see Task 1.5)
- [ ] Error handling: catch formula evaluation errors, timeouts, null references; report with helpful messages
- [ ] Logging: structured logs for each step (scenario resolved, X formulas evaluated, Y periods processed, runtime)
- [ ] Performance targets:
  - 50 formulas × 18 months in < 5 seconds (dev) / < 2 seconds (optimized)
  - Unit tests with mock data; integration tests with seed data

### Task 1.4: Core Backend Services - Calibration Engine
- [ ] Implement `CalibrationEngine` service:
  - Given CostRevenueFormula, date range, and realisatiedata from bookkeeping
  - Extract historical (period, actual) pairs from bookkeeping-bbv-compliance API
  - Fit formula to actuals (least squares or similar regression)
  - Derive constants (`kosten_per_eenheid`, `indexering` adjustments, etc.)
  - Compute metrics: RMSE, MAPE, R²
- [ ] Implement goodness-of-fit interpretation:
  - MAPE < 10%: high confidence
  - 10% ≤ MAPE ≤ 25%: medium confidence
  - MAPE > 25%: low confidence (flag for manual review)
- [ ] Implement `CalibrationService.createCalibration(formulaRef, period, overrideCalibrationMethod)`:
  - Create Calibration object with acceptanceStatus='pending'
  - Return to caller (not auto-accepted)
- [ ] Acceptance workflow: `CalibrationService.acceptCalibration(calibrationId, rationale)` or `rejectCalibration(calibrationId, rationale)`
- [ ] Idempotency: re-running calibration for same formula+period returns same derived constants (within tolerance)
- [ ] Test suite:
  - Mock realisatiedata: 4 years × 12 periods = 48 data points per formula
  - Verify fitted constants match expected values (within tolerance)
  - Verify metrics (RMSE, MAPE, R²) are computed correctly

### Task 1.5: Core Backend Services - Confidence Interval Propagation (Monte Carlo)
- [ ] Implement `MonteCarloSimulator` service:
  - Given formula and driver forecasts with CI bounds
  - Sample each driver N times (default N=5000) from confidence interval distribution
  - For each sample, evaluate formula
  - Extract central (50th percentile), lower (5th percentile), upper (95th percentile)
  - Return confidence interval for output
- [ ] Distribution assumptions:
  - If driver has lowerCI and upperCI, assume uniform or normal (configurable per driver)
  - Bounds are 90% CI by convention (adjust if needed)
- [ ] Deterministic drivers (no CI): treated as fixed value (sample count = 1)
- [ ] Low sample count handling:
  - Monitor execution time
  - If approaching time limit, reduce sample count to minimum (e.g., 1000)
  - Log warning: `LOW_SAMPLE_COUNT` with actual sample count
- [ ] Performance:
  - 5000 samples × 50 drivers in < 1 second (use vectorized/SIMD operations if possible)
  - Test with Monte Carlo on 5+ formula combinations

### Task 1.6: Database Migrations & OpenRegister Integration
- [ ] Verify schema migrations run successfully on deploy
- [ ] Test: CREATE, READ, UPDATE, DELETE operations on all 8 entity types via ObjectService
- [ ] Audit trails: verify AuditTrailService captures all mutations with before/after snapshots
- [ ] Search indexing: verify IndexService indexes all entity types and supports filtering by common fields
- [ ] Webhooks: configure OpenRegister to emit webhooks on:
  - Driver created/updated/deleted
  - CostRevenueFormula created/updated/deleted
  - ForecastRun completed
  - Variance flagged (requiresExplanation=true)

---

## Phase 2: API & Integration

### Task 2.1: REST API Endpoints
- [ ] Driver CRUD endpoints:
  - `GET /api/drivers` (list, paginated, filterable by domain/provenance/code)
  - `GET /api/drivers/{code}` (detail)
  - `POST /api/drivers` (create)
  - `PUT /api/drivers/{code}` (update)
  - `DELETE /api/drivers/{code}` (soft-delete or mark deprecated)
- [ ] CostRevenueFormula CRUD:
  - `GET /api/formulas` (list, filterable by targetType/targetRef)
  - `GET /api/formulas/{id}` (detail)
  - `POST /api/formulas` (create, with DSL parsing & validation)
  - `PUT /api/formulas/{id}` (update)
- [ ] Scenario CRUD:
  - `GET /api/scenarios` (list, filterable by tag, parent)
  - `GET /api/scenarios/{code}` (detail, with resolved parameters)
  - `POST /api/scenarios` (create)
  - `PUT /api/scenarios/{code}` (update)
- [ ] ForecastRun endpoints:
  - `GET /api/forecast-runs` (list, filterable by scenario, horizon, status)
  - `GET /api/forecast-runs/{id}` (detail with outputs)
  - `POST /api/forecast-runs` (trigger manual ad-hoc run)
  - `POST /api/forecast/what-if` (synchronous what-if, see Task 2.2)
  - `POST /api/forecast-runs/{id}/promote-to-begroting` (promotion workflow)
- [ ] Calibration endpoints:
  - `POST /api/formulas/{id}/calibrate` (run calibration, returns pending Calibration)
  - `POST /api/calibrations/{id}/accept` (accept calibration, update formula)
  - `POST /api/calibrations/{id}/reject` (reject, with rationale)
- [ ] Variance endpoints:
  - `GET /api/variances` (list, filterable by period, target, flaggedFor status)
  - `GET /api/variances/{id}` (detail with attribution)
  - `PUT /api/variances/{id}/commentary` (add owner commentary)
- [ ] All endpoints use proper HTTP status codes (200, 201, 400, 403, 404, 500, 503)
- [ ] All endpoints validate input and return structured error responses
- [ ] API documentation (OpenAPI 3.0 spec with examples)

### Task 2.2: What-If API Endpoint
- [ ] Implement `POST /api/forecast/what-if`:
  - Input: baseScenarioRef, overrides[]
  - Output: per-target per-period deltas vs base (not persisted)
  - Performance target: < 2 seconds
- [ ] Caching strategy:
  - Cache base scenario's ForecastRun in memory (30-second TTL)
  - Cache Driver lookups (60-second TTL)
- [ ] Error handling:
  - Deprecated driver: return 400 with suggestion
  - Complex scenario timeout: return 503 with async alternative
  - Invalid overrides: return 400 with validation errors
- [ ] Test:
  - 50 formulas × 18 months in < 2 seconds
  - Multiple override combinations
  - Deprecated driver detection

### Task 2.3: Driver Refresh Integration with openconnector
- [ ] Implement `DriverRefreshService`:
  - Query openconnector for available sources (CBS-StatLine, PBL, DUO, Vektis, BAG, CPB)
  - For each Driver with sourceRef, fetch latest value from openconnector
  - Append (period, value) to historicalSeries
  - Update lastRefreshedAt
- [ ] Schedule via RollingForecastSchedule triggers:
  - `triggers: ["calendar", "event-driver-refreshed"]`
  - Calendar trigger: run on configured frequency (daily/weekly/monthly/quarterly)
  - Event trigger: run immediately when openconnector emits driver-refreshed event
- [ ] Error handling:
  - If openconnector unavailable, log error and skip (don't fail entire refresh)
  - Mark driver as stale if refresh fails (update lastRefreshedAt to null or flag)
- [ ] Logging: structured log per driver (source, value fetched, append to history, timestamp)

### Task 2.4: Variance Computation Integration with bookkeeping-bbv-compliance
- [ ] Implement `VarianceComputationService`:
  - After period close (event-driven or scheduled), fetch realisatiecijfers from bookkeeping API
  - For each ForecastRun (or most recent for given scenario), for each target, for each period:
    - Fetch actual from bookkeeping
    - Compute delta = actual - forecast
  - Create Variance records
- [ ] Implement attribution algorithm:
  - Decompose delta into: volume-effect, prijs-effect, mix-effect, indexering-effect, onverklaard-residu
  - Use formula constants and driver contributions (captured in ForecastRun) to attribute
  - Example: Wmo cost variance = (Δ volume) × (cost per unit) + (Δ cost per unit) × (volume) + mix + indexering + residu
- [ ] High-variance flagging:
  - If |deltaPct| > 5%, set requiresExplanation=true
  - Notify formula owner: send Nextcloud notification + create task (if task system available)
- [ ] Integration with begroting-werkbestand:
  - If ForecastRun was promoted to begroting, check for raads-amendementen
  - Attribute raads-amendement to separate component in attribution
- [ ] Test:
  - Mock realisatiedata: 10–20 periods × 5 targets = 50–100 variance records
  - Verify attribution sums to delta
  - Verify flagging logic (deltaPct > 5%)

### Task 2.5: Integration with pc-cyclus-workflow (Begroting Promotion)
- [ ] Implement `BegrotingPromotionService`:
  - Accept ForecastRun ID to be promoted
  - Create frozen snapshot: serialize all referenced Drivers, Formulas, Calibrations, Scenarios
  - Call pc-cyclus-workflow API to create/update begroting-werkbestand with ForecastRun.outputs
  - Set ForecastRun.promotedToBegrotingRef
- [ ] Webhook handling:
  - pc-cyclus-workflow emits event when raads-amendement is recorded
  - Extract raads-amendement details (target, amount, rationale)
  - Create related Scenario with tag='raads-amendement' (for next-period variance attribution)
- [ ] Frozen snapshot immutability:
  - When computing variance against promoted ForecastRun, use snapshot (not current state of drivers/formulas)
  - Prevent accidental overwrites of begroting by protecting snapshot
- [ ] Test:
  - Mock begroting-werkbestand API
  - Verify snapshot contains complete object graphs
  - Verify subsequent driver refreshes don't affect begroting snapshot

### Task 2.6: Integration with decidesk (Policy Measures)
- [ ] Implement policy measure validation:
  - When creating/updating Scenario with policyMeasures[].decidesk_besluitRef
  - Query decidesk API to fetch status of besluit
  - Warn if status is verworpen or ingetrokken
- [ ] Implement scheduled validation:
  - Background job (n8n or scheduler): daily, check all scenarios' policy measures for status changes
  - If besluit status changes to verworpen, flag scenario for archival or re-tagging
- [ ] Error handling:
  - If decidesk unavailable, continue without validation (warn in logs)
  - If besluit not found, warn and request correction

---

## Phase 3: Frontend & UI

### Task 3.1: Driver Management UI
- [ ] Create Driver list page (auto-generated via CnIndexPage + OpenRegister)
  - Display: code, naam, domain, eenheid, provenance, lastRefreshedAt
  - Search & filter by: domain, provenance, source, staleness status
  - Sort by: code, lastRefreshedAt, refreshFrequency
- [ ] Create Driver detail page (auto-generated via CnDetailPage)
  - Display all fields
  - Editable form for: code, naam, eenheid, domain, sourceRef, provenance, refreshFrequency, confidenceRationale
  - Read-only display of: historicalSeries (as chart or table), baselineForecastRef
  - Action buttons: "Refresh Now" (manual trigger), "View Forecast", "Delete"
  - History tab (via CnAuditTrailTab): show all mutations
- [ ] Manual refresh action: `POST /api/drivers/{code}/refresh-now`
  - Trigger DriverRefreshService for single driver
  - Display progress spinner
  - Show result: fetched value, new period appended, lastRefreshedAt updated
  - Display staleness warning if applicable
- [ ] Seed data: load 5 example drivers on install

### Task 3.2: Formula Management UI
- [ ] Create CostRevenueFormula list page
  - Display: targetType, targetRef, formula (first 100 chars), owner, peerReviewedBy, validFrom, validTo
  - Filter by: targetType, owner, calibration status, validity (active/expired)
  - Sort by: targetRef, validFrom
- [ ] Create formula detail page
  - Display all fields
  - Formula field rendered with syntax highlighting + validation indicator (✓ valid, ✗ error)
  - Editable form for: formula, ownerRef, documentation, validFrom, validTo
  - On formula change: trigger DSL validation in real-time (debounced), display errors with suggestions
  - Sections:
    - Formula (DSL code + validation)
    - Metadata (target, owner, reviewer, dates, documentation)
    - Drivers (extracted from formula, linked to Driver detail pages)
    - Calibration (most recent calibration: status, metrics, fitted constants, actions to accept/reject)
  - Action buttons: "Run Calibration", "View Forecast Runs", "Delete"
  - History tab: audit trail
- [ ] Calibration workflow:
  - "Run Calibration" button opens modal:
    - Select date range for calibration (default last 4 years)
    - Click "Start Calibration"
    - Spinner shows progress: "Fetching realisatiedata... Fitting constants... Computing metrics..."
    - Result: show RMSE, MAPE, R² (color-coded: green MAPE<10%, yellow 10-25%, red >25%)
    - Derived constants displayed with values
    - "Accept" button (requires rationale input if MAPE>25%) or "Reject"
- [ ] Seed data: load 3 example formulas on install

### Task 3.3: Scenario Management UI
- [ ] Create Scenario list page
  - Display: code, naam, tag, parentScenarioRef, createdAt, createdBy
  - Filter by: tag (optimistisch/realistisch/pessimistisch/beleidsvariant), parent, creator
  - Color-code tags (realistisch=blue, pessimistisch=red, optimistisch=green, beleidsvariant=orange)
- [ ] Create Scenario detail page
  - Display all fields
  - Scenario inheritance tree visualization (parent → child, multi-level support)
  - Resolved parameters section: show merged parameters (parent + child overrides)
  - Parameter overrides editable form:
    - Add override: select driver or formula, select period, select operation (multiplier/absolute/deltaPct), enter value
    - List overrides with "Edit" and "Delete" per override
    - Validate: referenced drivers exist, deltas are numeric
  - Policy measures section:
    - Link to decidesk besluiten (show status: aangenomen/verworpen/etc)
    - Warn if linked besluit is verworpen (offer auto-disable)
  - Action buttons: "Create Child Scenario", "Run Forecast", "Promote to Begroting", "Delete"
  - History tab: audit trail
- [ ] Conflict detection UI:
  - When creating child scenario with multi-parent merge, show conflict summary
  - Offer resolution options (e.g., "Choose Parent A's value", "Choose Parent B's value", "Custom override")
- [ ] Seed data: load 3 example scenarios (root + 2 children)

### Task 3.4: ForecastRun & Results UI
- [ ] Create ForecastRun list page
  - Display: scenarioRef, runType, runAt, runBy, status, runtime, outputs summary
  - Filter by: scenario, status, runType (rolling/ad-hoc/what-if), date range
  - Distinguish rolling runs from ad-hoc (icon or badge)
- [ ] Create ForecastRun detail page
  - Scenario info (with link to scenario detail)
  - Results table: target, period, forecastValue, [confidence interval if applicable], driver contributions (expandable)
  - Input snapshot section: show frozen state of drivers/formulas/calibrations used (read-only)
  - Warnings/errors section (if any): display DRIVER_STALE, ASSUMED_DETERMINISTIC, LOW_SAMPLE_COUNT, etc.
  - Action buttons (if not promoted): "Download CSV", "Promote to Begroting", "Delete"
  - Action buttons (if promoted): "View Promoted Begroting", "View Divergence" (compare to rolling forecast)
- [ ] What-if sandbox page:
  - Base scenario selector (dropdown)
  - Overrides form (add driver volume +/-, add formula constant ±)
  - Live output preview: per-target per-period deltas vs base
  - Export options: CSV, JSON, embed in slide deck
- [ ] Manual ForecastRun trigger:
  - Page with scenario selector, horizon selector, "Run Now" button
  - Spinner showing progress: "Resolving scenario... Validating formulas... Executing..."
  - Result page displayed on completion (auto-navigates to detail view)
- [ ] Rolling forecast dashboard (main page):
  - Latest rolling runs for each scenario (realistisch, pessimistisch, optimistisch)
  - Key-value cards: total revenue/cost forecast, per-domain breakdown
  - Trend mini-chart: forecast over time (rolling runs sequentially)
  - "Run What-If" quick-link button
- [ ] No seed data needed (runs generated on demand)

### Task 3.5: Variance Review UI
- [ ] Create Variance list page
  - Display: period, targetRef, forecastValue, actualValue, delta, deltaPct, requiresExplanation flag, commentaryByOwner status
  - Filter by: period, target, requiresExplanation, flaggedFor status
  - Highlight high-variance rows (|deltaPct| > 5%)
  - Color-coding: green < 2%, yellow 2-5%, orange 5-10%, red > 10%
- [ ] Create Variance detail page
  - Forecast vs Actual side-by-side (values, delta, deltaPct)
  - Attribution breakdown: table with (type, amount, % of delta)
  - Attribution chart: stacked bar or waterfall showing composition
  - Commentary section:
    - Display existing commentary (if any) with author, timestamp
    - Form to add commentary (if not yet added and requiresExplanation=true)
    - "Mark Resolved" button after commentary added
  - Related objects: link to ForecastRun, link to CostRevenueFormula
- [ ] Variance dashboard (summary page):
  - Cards per period: # of variances, # flagged high, # requiring explanation, % explained
  - List of unexplained high-variance items (sorted by magnitude)
  - "Action Items" for current user (variances assigned to them for commentary)
- [ ] No seed data needed (variances computed on demand)

### Task 3.6: Reference Packs UI (Admin)
- [ ] Create reference pack list page (admin only)
  - Display: pack name, sector, version, import status, creation date
  - Action: "View Pack Details", "Import", "Update", "Delete"
- [ ] Create reference pack detail page
  - Show manifest: # drivers, # formulas, # calibrations
  - Drivers list (preview first 10)
  - Formulas list (preview first 10)
  - Calibrations list (preview first 10)
  - Import/Update workflow:
    - Show diff (new items, renamed items, updated items)
    - Checkboxes to select which changes to apply
    - "Apply" button to execute import
    - Result page: "Imported 28 drivers, 12 formulas, 15 calibrations"
- [ ] Seed data: pre-create gemeente pack with 5 drivers (for auto-testing)

---

## Phase 4: Scheduling & Automation (n8n)

### Task 4.1: Rolling Forecast Scheduler Workflow
- [ ] Create n8n workflow: "Driver-Based Forecasting: Rolling Forecast Monthly"
  - Trigger: Schedule (cron: monthly on day 5 at 05:00 UTC)
  - Step 1: Query all enabled RollingForecastSchedules
  - Step 2: For each schedule:
    - Step 2a: Trigger Driver Refresh (openconnector integration)
    - Step 2b: For each scenario in schedule.scenarioRefs:
      - Call ForecastEngine API: POST /api/forecast-runs (trigger manual run)
      - Wait for completion (poll status or webhook)
      - Log runtime, status, any warnings
    - Step 2c: Send notification to notificationRecipients (Nextcloud API)
  - Error handling: log errors, don't fail workflow if one scenario fails
- [ ] Implement alternative trigger: `event-driver-refreshed`
  - openconnector emits webhook when driver is refreshed
  - n8n workflow listens and immediately triggers dependent ForecastRuns (those using this driver)
- [ ] Logging & monitoring:
  - Record workflow execution time, # scenarios run, # warnings, # failures
  - Dashboard accessible at n8n console

### Task 4.2: Variance Computation Scheduled Workflow
- [ ] Create n8n workflow: "Driver-Based Forecasting: Variance Computation Monthly"
  - Trigger: Schedule (monthly on day 1 at 08:00, after period close)
  - Step 1: Query bookkeeping API for newly closed periods
  - Step 2: For each closed period, for each active scenario:
    - Call VarianceComputationService API: POST /api/variances (compute variances)
    - Fetch computed Variance records
  - Step 3: For each variance with requiresExplanation=true:
    - Fetch formula owner
    - Create Nextcloud notification: "Variance [target] [period] requires explanation (delta [amount])"
    - Optionally: create task in task-management system (if available)
  - Error handling: log and continue

### Task 4.3: Policy Measure Validation Scheduled Workflow
- [ ] Create n8n workflow: "Driver-Based Forecasting: Policy Measure Validation"
  - Trigger: Schedule (daily at 06:00)
  - Step 1: Query all scenarios with policyMeasures
  - Step 2: For each policy measure, call decidesk API to fetch latest decision status
  - Step 3: If status changed to verworpen/ingetrokken:
    - Flag scenario for review
    - Notify scenario creator: "Policy measure [besluit] status changed to verworpen; review scenario [code]"
  - Error handling: if decidesk unavailable, skip (don't fail)

### Task 4.4: Begroting Promotion Workflow
- [ ] Create n8n workflow: "Driver-Based Forecasting: Promote to Begroting"
  - Triggered manually by Concerncontroller via UI action
  - Step 1: Receive ForecastRun ID + approval decision
  - Step 2: Call BegrotingPromotionService API: POST /api/forecast-runs/{id}/promote-to-begroting
  - Step 3: Call pc-cyclus-workflow API to receive promoted ForecastRun
  - Step 4: Send confirmation email to concerncontroller + finance team lead
  - Error handling: return error to UI with user-friendly message

---

## Phase 5: Documentation & Training

### Task 5.1: Technical Documentation
- [ ] Write API documentation (OpenAPI 3.0 spec + examples)
  - Driver endpoints
  - Formula endpoints (including DSL specification)
  - Scenario endpoints
  - ForecastRun endpoints
  - Calibration endpoints
  - Variance endpoints
  - What-If API
- [ ] Write DSL specification document:
  - Grammar (EBNF notation)
  - Primitives: volume, kosten_per_eenheid, indexering, policy_override
  - Examples of correct and incorrect formulas
  - Validation rules
- [ ] Write database schema documentation:
  - Entity-relationship diagram (8 schemas)
  - Field descriptions, constraints, relationships
  - Seed data specification
- [ ] Write integration guide:
  - Integration points with openconnector, bookkeeping-bbv-compliance, pc-cyclus-workflow, decidesk
  - Webhook specifications
  - n8n workflow architecture
- [ ] Deployment guide:
  - System requirements (memory, storage, compute for Monte Carlo)
  - Configuration (RollingForecastSchedule defaults, Monte Carlo sample count, caching TTLs)
  - Migration path (upgrading from prior version)

### Task 5.2: User Documentation & Journeys
- [ ] Create journeydoc: "Wmo-budget onderbouwen met driver-based forecasting"
  - User personas: Programma-controller Wmo, Strateeg
  - Workflow: Create driver (inwoners, wmo-clients) → Create formula → Calibrate → Run forecast → Variance review
  - Screenshots and UI walkthrough
  - Calibration interpretation (MAPE, R², confidence)
  - Troubleshooting: stale drivers, poor-fit calibrations
- [ ] Create journeydoc: "What-if doen: hervormingsagenda jeugd"
  - User personas: Beleidsadviseur, Strateeg, Portefeuillehouder
  - Workflow: Load realistisch scenario → Create what-if variant (jeugdbudget -15%) → Compare deltas → Export scenario → Present to raad
  - Visual: scenario comparison card (old vs new budget per programma)
  - Export: slides, Excel, PDF
- [ ] Create video tutorials (optional):
  - 2–3 min each: Creating a driver, Running calibration, What-if sandbox
  - Publish on company intranet or docusaurus
- [ ] Create FAQs:
  - "Why is my forecast high-variance?" (staleness, calibration issues, regime changes)
  - "How do I interpret MAPE > 25%?" (when to accept, when to reject)
  - "How do I link a policy decision to a scenario?" (decidesk integration)
  - "How do I promote a scenario to begroting?" (workflow and approvals)

### Task 5.3: Admin Documentation
- [ ] Administrator manual:
  - Setup: reference pack import (gemeente/provincie/waterschap)
  - Configuration: RollingForecastSchedule, Monte Carlo sample count, refresh frequencies
  - Monitoring: n8n workflow execution, API performance, database size
  - Backup & restore: snapshots for disaster recovery
  - Troubleshooting: common errors, log analysis
- [ ] Reference pack management guide:
  - How to create a new reference pack (for DSOs extending the system)
  - Version management and update workflows
  - Calibration starting points (sourcing from NBA, Vektis, CBS)

---

## Phase 6: Testing & QA

### Task 6.1: Unit Tests
- [ ] Formula DSL parser & validator:
  - 50+ test cases (valid formulas, syntax errors, semantic errors, suggestions)
  - Performance: 1000 parses in < 100ms
- [ ] Forecast engine:
  - 10+ test cases (scenario inheritance, override merging, formula evaluation)
  - Correctness: compare engine output to hand-calculated expected values
- [ ] Calibration engine:
  - 5+ test cases (different formulas, different data sizes, goodness-of-fit metrics)
  - Correctness: RMSE, MAPE, R² computed correctly
- [ ] Monte Carlo simulator:
  - 3+ test cases (single driver, multiple drivers, deterministic+stochastic mix)
  - Statistical validation: sample distribution matches expected CI bounds
- [ ] Coverage target: ≥ 85% for backend logic

### Task 6.2: Integration Tests
- [ ] API endpoint tests (via HTTP client):
  - CRUD operations on all 8 entity types
  - DSL validation in formula POST/PUT
  - Calibration acceptance workflow
  - What-if API: standard and edge cases
  - Error responses: 400, 403, 404, 500, 503
- [ ] Database tests:
  - Seed data import (idempotent)
  - Audit trail (before/after snapshots)
  - Search & filtering
- [ ] n8n workflow tests:
  - Mock openconnector, bookkeeping, pc-cyclus-workflow APIs
  - Trigger rolling-forecast workflow; verify ForecastRuns created
  - Trigger variance-computation workflow; verify Variance objects created
- [ ] Coverage target: ≥ 80% for API/integration logic

### Task 6.3: Browser/UI Tests
- [ ] Manual smoke tests (or automated browser tests if test-app skill available):
  - Create driver, verify form validation (code unique, provenance required)
  - Create formula, verify DSL validation in real-time (syntax error, unknown driver suggestion)
  - Create scenario, verify inheritance and parameter merging
  - Run forecast, verify results display
  - Run what-if, verify < 2 second response
  - Add variance commentary, verify saved and propagated to next run
  - Promote forecast to begroting, verify frozen snapshot created
- [ ] Edge cases:
  - Stale driver (> 2× refreshFrequency): verify warning displayed
  - High-variance formula (MAPE > 25%): verify low-confidence flag and required rationale
  - Conflicting scenario parameters: verify error message and resolution options
- [ ] Performance smoke tests:
  - 50 formulas × 18 months: measure runtime (target < 5 sec)
  - What-if sandbox: measure API response time (target < 2 sec)

### Task 6.4: Data Quality & Audit Tests
- [ ] Seed data quality:
  - Verify all seed objects are valid per schema (no missing required fields)
  - Verify cross-references are consistent (formulaRef points to existing formula, driverRef points to existing driver)
  - Verify monetary amounts are realistic (not 0, not negative, in expected range)
- [ ] Audit trail tests:
  - Verify all mutations logged (create, update, delete)
  - Verify before/after snapshots captured
  - Verify user ID recorded (not display name)
- [ ] Regression suite:
  - Re-run all tests on every PR (CI/CD pipeline)
  - Nightly regression suite: extended tests covering edge cases, performance, data integrity

---

## Phase 7: Deduplication & Architecture Review

### Task 7.1: Reuse Analysis & Deduplication Check
- [ ] Audit existing OpenRegister capabilities:
  - Verify no duplication with ObjectService (CRUD), IndexService (search), AuditTrailService (history)
  - Verify CnIndexPage, CnDetailPage, CnFormDialog are reused (not reimplemented)
  - Verify ImportService/ExportService are leveraged for seed data & reference packs
- [ ] Audit existing openspec specs:
  - Check for similar driver registry patterns (e.g., openconnector's own driver model)
  - Check for similar formula DSL patterns (accounting, planning tools)
  - Verify no overlap; if overlap found, document why new implementation is needed
- [ ] Document findings:
  - Create task 7.1-findings.md listing existing capabilities reused and any intentional divergences
  - Justifications for new code (e.g., DSL parser is domain-specific; no existing implementation suitable)
- [ ] Post-implementation: if new overlaps discovered, refactor to reuse existing utilities

### Task 7.2: Architecture Review
- [ ] Review data model:
  - 8 schemas are well-defined, no redundancy
  - Cross-entity references use OpenRegister relations (not foreign keys)
  - No embedded objects (all separate schemas)
- [ ] Review API design:
  - REST endpoints follow OpenRegister patterns (CRUD, search, filtering)
  - What-if API is synchronous and appropriately scoped (not persisted)
  - Promotion workflow is asynchronous (via n8n, not blocking)
- [ ] Review performance:
  - ForecastEngine < 5 sec: check algorithm, caching, indexing
  - What-if API < 2 sec: check caching, sampling, optimization
  - Monte Carlo: check vector/SIMD optimizations, sample count tuning
- [ ] Document architectural decisions:
  - Why OpenRegister (instead of custom ORM)
  - Why synchronous what-if (instead of async only)
  - Why n8n scheduling (instead of in-app scheduler)
  - Why 18-month default horizon (BBV standard)

---

## Checklist: Pre-Release Validation

Before marking this spec as ready for implementation:

- [ ] All 10 requirements (REQ-001 to REQ-010) have detailed acceptance criteria (GIVEN/WHEN/THEN)
- [ ] All 8 OpenRegister schemas are defined with examples in design.md
- [ ] All seed data includes realistic Dutch values (names, addresses, codes)
- [ ] All tasks are actionable with clear success criteria
- [ ] Integration points with 7 external apps are documented (openconnector, bookkeeping, pc-cyclus, decidesk, mydash, docudesk, n8n)
- [ ] n8n workflows are outlined with steps, error handling, and event triggers
- [ ] User journeys (proposal.md) cover all 10 user personas
- [ ] Reuse analysis (Task 7.1) confirms no unnecessary custom code
- [ ] Performance targets are specified and testable
- [ ] Security considerations (RBAC, input validation) are addressed in design
- [ ] Deduplication check (Task 7.1) and architecture review (Task 7.2) tasks are scheduled

