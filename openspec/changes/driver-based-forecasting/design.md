---
status: draft
version: 1.0
---

# Driver-Based Forecasting — Design

## Architecture Overview

The `driver-based-forecasting` spec introduces 8 OpenRegister schemas (Driver, DriverForecast, CostRevenueFormula, Calibration, Scenario, ForecastRun, Variance, RollingForecastSchedule) stored under `financeq/openspec/specs/driver-based-forecasting/schemas/`. All data inherits OpenRegister's audit-trail, versioning, search, and RBAC automatically.

### Data Model

The spec reuses two core objects from platform utilities:
- `core-financial-objects.Money` for monetary amounts (EUR, with decimal precision)
- `core-time.PeriodSpec` for time-points (YYYY-QN or YYYY-MM format)
- `core-statistics.ConfidenceInterval` for (value, lowerBound, upperBound, sampleCount, methodology)

### Storage & Lifecycle

All 8 registers are OpenRegister objects, supporting:
- Full CRUD via ObjectService
- Search & filtering via IndexService
- Immutable history via AuditTrailService
- Version control via ObjectService.saveObject() with versioning
- RBAC (see Authorization below)
- Webhooks on create/update/delete
- Related items (files, notes, tasks) linkable via relation system (pending openregister seed-related-items change; currently only object properties supported)

### Reuse Analysis

- **ObjectService + CnIndexPage + CnDetailPage:** All 8 schemas use OpenRegister's CRUD UI automatically; no custom list/detail pages needed.
- **CnFormDialog:** Auto-generated forms from schema definitions; custom DSL editor needed only for CostRevenueFormula.formula field.
- **ImportService / ExportService:** Built-in CSV/Excel import and export; reference packs delivered as JSON seed data.
- **IndexService + FacetBuilder:** Full-text search, faceted navigation on all registers (e.g., search drivers by domain, scenario by tag, etc.)
- **AuditTrailService:** Automatic change tracking; Calibration and Variance records provide business-level audit trail for NBA compliance.
- **FileService:** Attach calibration reports, variance commentaries, policy documentation to relevant objects.
- **NotificationService:** Alerts on stale drivers, high-variance flags, reference-pack updates.
- **WebhookService:** Trigger n8n workflows on ForecastRun completion, Variance flagged, scenario promoted to begroting.

**No custom entity/mapper layers needed.** All domain objects are OpenRegister objects.

### External Integrations

- **openconnector:** Scheduled refreshes of drivers from CBS-StatLine, PBL, DUO, Vektis, BAG, CPB.
- **bookkeeping-bbv-compliance:** Supplies grootboek-rekeningen (cost centers) and realisatiecijfers for calibration and variance.
- **pc-cyclus-workflow:** Receives promoted ForecastRuns to populate begroting-werkbestand; receives raads-amendementen as overrides for next-period variance.
- **decidesk:** Policy decisions linked from Scenario.policyMeasures[]; used to validate scenario scenarios (warn if linked besluit is verworpen).
- **mydash:** Consumes ForecastRun.outputs[] and Variance records to render forecast-vs-actual widgets and scenario-comparison cards.
- **docudesk:** Archives promoted ForecastRun snapshots alongside begroting-werkbestand they substantiate.
- **n8n:** Orchestrates rolling-forecast scheduler (monthly, configurable), variance computations (monthly after period close), promotion workflows (sign-off, begroting update).
- **docusaurus journeydoc:** Ships "Wmo-budget onderbouwen met driver-based" tutorial and "What-if doen: hervormingsagenda jeugd" walkthrough.

## Schemas

### 1. Driver

Represents a forecast driver (inwoners, leerlingen, zorgvragers, square meters, km riool, hectares groen, etc.).

```json
{
  "title": "Driver",
  "type": "object",
  "properties": {
    "code": { "type": "string", "description": "Unique identifier, e.g., inwoners-totaal, wmo-clienten-65plus, leerlingen-po. PascalCase + kebab-case pattern.", "pattern": "^[a-z0-9-]+$" },
    "naam": { "type": "string", "description": "Human-readable name in Dutch." },
    "eenheid": { "type": "string", "enum": ["eenheid", "m2", "km", "fte", "ha"], "description": "Unit of measurement." },
    "domain": { "type": "string", "enum": ["demografie", "onderwijs", "zorg", "fysieke-leefomgeving", "sociaal-domein", "veiligheid", "economie"], "description": "Domain classification." },
    "aggregationLevel": { "type": "string", "enum": ["organisatie", "wijk", "buurt", "gemeente-binnen-gr"], "description": "Geographic granularity at which driver is available." },
    "sourceRef": { "type": "string", "description": "openconnector source identifier, e.g., cbs-statline-37230NED, vektis-wmo-2024." },
    "provenance": { "type": "string", "enum": ["CBS-StatLine-table-X", "BAG", "Vektis", "DUO", "eigen-meting", "expert-judgement"], "description": "Data source." },
    "refreshFrequency": { "type": "string", "enum": ["daily", "weekly", "monthly", "quarterly", "yearly"], "description": "Expected refresh cadence from source." },
    "lastRefreshedAt": { "type": "string", "format": "date-time", "description": "Timestamp of most recent value append." },
    "historicalSeries": { 
      "type": "array",
      "description": "Time-ordered series of (periodSpec, value) pairs, typically ≥ 5 years history.",
      "items": {
        "type": "object",
        "properties": {
          "period": { "$ref": "#/components/schemas/PeriodSpec" },
          "value": { "type": "number" }
        },
        "required": ["period", "value"]
      }
    },
    "baselineForecastRef": { "type": "string", "description": "Reference to a DriverForecast object used as baseline for scenario overrides." },
    "confidenceRationale": { "type": "string", "description": "Free-text explanation of confidence interval sources (e.g., 'CBS PBL prognose uncertainty band', 'Kalman-filter from 2023-2024 actuals')." }
  },
  "required": ["code", "naam", "eenheid", "domain", "sourceRef", "provenance", "refreshFrequency", "historicalSeries"]
}
```

### 2. DriverForecast

Projected values for a Driver over a forecast horizon (next 18 months, 4 years, etc.).

```json
{
  "title": "DriverForecast",
  "type": "object",
  "properties": {
    "driverRef": { "type": "string", "description": "Reference to Driver object." },
    "scenarioRef": { "type": "string", "description": "Reference to Scenario object. One driver may have multiple forecasts for different scenarios." },
    "forecastVersion": { "type": "integer", "description": "Version number if driver forecast is recalculated." },
    "horizon": { "type": "string", "description": "Duration, e.g., '18 months', '4 years'. Free-form string for flexibility." },
    "dataPoints": {
      "type": "array",
      "description": "Time-ordered forecast values, each with central estimate and confidence interval.",
      "items": {
        "type": "object",
        "properties": {
          "period": { "$ref": "#/components/schemas/PeriodSpec" },
          "value": { "type": "number" },
          "lowerCI": { "type": "number", "description": "Lower bound of 90% confidence interval (if applicable)." },
          "upperCI": { "type": "number", "description": "Upper bound of 90% confidence interval (if applicable)." }
        },
        "required": ["period", "value"]
      }
    },
    "method": { "type": "string", "enum": ["cbs-prognose", "trend-extrapolatie", "ar-1", "prophet", "expert-judgement", "policy-override"], "description": "Forecasting method used." },
    "methodParams": { "type": "object", "additionalProperties": true, "description": "Method-specific parameters (e.g., for prophet: seasonality_mode, interval_width; for trend-extrapolatie: lookback_periods)." },
    "confidence": { 
      "type": "object",
      "properties": {
        "level": { "type": "string", "enum": ["high", "medium", "low"] },
        "rationale": { "type": "string", "description": "Why this confidence level (e.g., 'high: CBS official forecast with 95% CI', 'low: expert judgement post-pandemic, regimes uncertain')." }
      },
      "required": ["level", "rationale"]
    },
    "createdAt": { "type": "string", "format": "date-time" },
    "createdBy": { "type": "string", "description": "User ID of forecaster." }
  },
  "required": ["driverRef", "scenarioRef", "horizon", "dataPoints", "method", "confidence"]
}
```

### 3. CostRevenueFormula

Links a financial target (cost center, programma-onderdeel, taakveld) to one or more drivers via a typed DSL.

```json
{
  "title": "CostRevenueFormula",
  "type": "object",
  "properties": {
    "targetType": { "type": "string", "enum": ["grootboek-rekening", "programma-onderdeel", "taakveld", "economische-categorie"], "description": "Type of financial target." },
    "targetRef": { "type": "string", "description": "Reference to the target (e.g., cost-center code, programma-onderdeel slug)." },
    "formula": { "type": "string", "description": "DSL expression: e.g., kosten_per_eenheid('wmo-zorgvraag-zwaar') * volume('wmo-clienten-zwaar') + indexering('cao-vng', 'salaris-component') * 1850000. See DSL spec below." },
    "driverRefs": { 
      "type": "array",
      "description": "Extracted driver references from formula (auto-populated at save time).",
      "items": { "type": "string" }
    },
    "validFrom": { "type": "string", "format": "date", "description": "When this formula becomes active." },
    "validTo": { "type": "string", "format": "date", "description": "When this formula expires (null = ongoing)." },
    "calibrationRef": { "type": "string", "description": "Reference to most recent accepted Calibration record." },
    "ownerRef": { "type": "string", "description": "User ID of programma-controller responsible for this formula." },
    "peerReviewedBy": { "type": "string", "description": "User ID of peer reviewer (required before first use)." },
    "documentation": { "type": "string", "description": "Free-text explanation of formula logic, assumptions, and rationale for constant values." }
  },
  "required": ["targetType", "targetRef", "formula", "validFrom", "ownerRef"]
}
```

**Typed DSL Grammar:**

```
expression := term ('+' | '-') term | term
term := factor ('*' | '/' | '%') factor | factor
factor := primitive | '(' expression ')' | '(' expression ')' '^' NUMBER

primitive := 
    | volume(DRIVER_CODE)
    | kosten_per_eenheid(CONSTANT_NAME)
    | indexering(INDEX_CODE, COMPONENT)
    | policy_override(SCENARIO_PARAMETER_REF)
    | NUMBER

DRIVER_CODE := 'inwoners-totaal', 'wmo-clienten-zwaar', etc. (validated against Driver.code)
CONSTANT_NAME := 'wmo-zorg-zwaar', 'cao-vng-salaris', etc. (validated against Calibration records)
INDEX_CODE := 'cao-vng', 'cpi', 'bbp-deflator', etc.
COMPONENT := 'salaris-component', 'material-cost', etc.
```

Static validation at save:
- All `volume(...)` drivers must exist and not be deprecated.
- All `kosten_per_eenheid(...)` and `indexering(...)` constants must be resolvable.
- Divide-by-zero patterns (`/ volume(...)`) flagged; author must acknowledge.
- Result type checked (should produce Money for financial targets).

### 4. Calibration

Derived from history: given a CostRevenueFormula and N years of realisatie, derive the kosten-per-eenheid that best fits.

```json
{
  "title": "Calibration",
  "type": "object",
  "properties": {
    "formulaRef": { "type": "string", "description": "Reference to CostRevenueFormula being calibrated." },
    "period": { "type": "string", "description": "Historical period used for calibration, e.g., '2020-2024'." },
    "derivedConstants": {
      "type": "array",
      "description": "List of fitted constants and their values.",
      "items": {
        "type": "object",
        "properties": {
          "constantName": { "type": "string" },
          "value": { "$ref": "#/components/schemas/Money" },
          "confidence": { "type": "string", "enum": ["high", "medium", "low"] }
        },
        "required": ["constantName", "value", "confidence"]
      }
    },
    "rmse": { "type": "number", "description": "Root mean squared error (absolute units)." },
    "mape": { "type": "number", "description": "Mean absolute percentage error (0-100%)." },
    "r2": { "type": "number", "description": "R-squared goodness-of-fit (0-1, higher is better)." },
    "notes": { "type": "string", "description": "Free-text notes (e.g., 'regressor stability concerns', 'data quality issues', 'regime change post-2024')." },
    "acceptanceStatus": { "type": "string", "enum": ["pending", "accepted", "rejected"], "description": "Acceptance status by accountant/controller." },
    "acceptanceRationale": { "type": "string", "description": "Justification for acceptance or rejection." },
    "acceptedBy": { "type": "string", "description": "User ID of approver." },
    "acceptedAt": { "type": "string", "format": "date-time" },
    "calibratedAt": { "type": "string", "format": "date-time" },
    "calibratedBy": { "type": "string", "description": "User ID of analist who ran calibration." }
  },
  "required": ["formulaRef", "period", "derivedConstants", "rmse", "mape", "r2", "calibratedAt", "calibratedBy"]
}
```

### 5. Scenario

Named what-if with named parameter overrides and optional parent inheritance.

```json
{
  "title": "Scenario",
  "type": "object",
  "properties": {
    "code": { "type": "string", "description": "Unique identifier, e.g., realistisch-2027, groeigebied-noord-2030, hervormingsagenda-jeugd." },
    "naam": { "type": "string" },
    "description": { "type": "string" },
    "parentScenarioRef": { "type": "string", "description": "Reference to parent scenario (allows inheritance). Null = root scenario." },
    "parameterOverrides": {
      "type": "array",
      "description": "Driver volume/formula constant overrides.",
      "items": {
        "type": "object",
        "oneOf": [
          {
            "properties": {
              "type": { "const": "driver-volume" },
              "driverRef": { "type": "string" },
              "period": { "$ref": "#/components/schemas/PeriodSpec" },
              "overrideType": { "type": "string", "enum": ["multiplier", "absoluteValue", "deltaPct"], "description": "multiplier: factor applied (1.05 = +5%); absoluteValue: replace with number; deltaPct: add percentage points." },
              "value": { "type": "number" }
            },
            "required": ["type", "driverRef", "period", "overrideType", "value"]
          },
          {
            "properties": {
              "type": { "const": "formula-constant" },
              "formulaRef": { "type": "string" },
              "constantName": { "type": "string" },
              "overrideType": { "type": "string", "enum": ["multiplier", "absoluteValue", "deltaPct"] },
              "value": { "type": "number" }
            },
            "required": ["type", "formulaRef", "constantName", "overrideType", "value"]
          }
        ]
      }
    },
    "policyMeasures": {
      "type": "array",
      "description": "Links to raadsbesluiten or other policy documents justifying overrides.",
      "items": {
        "type": "object",
        "properties": {
          "decidesk_besluitRef": { "type": "string", "description": "Reference to decidesk besluit object." },
          "rationale": { "type": "string" }
        }
      }
    },
    "tag": { "type": "string", "enum": ["optimistisch", "realistisch", "pessimistisch", "stress", "beleidsvariant"], "description": "Scenario classification for presentation and filtering." },
    "createdAt": { "type": "string", "format": "date-time" },
    "createdBy": { "type": "string" }
  },
  "required": ["code", "naam", "tag"]
}
```

### 6. ForecastRun

One execution of the forecast engine for a (scenario × horizon). Immutable once created.

```json
{
  "title": "ForecastRun",
  "type": "object",
  "properties": {
    "scenarioRef": { "type": "string", "description": "Reference to Scenario." },
    "horizon": { "type": "string", "description": "Forecast duration, e.g., '18 months', '4 years'." },
    "runType": { "type": "string", "enum": ["rolling", "ad-hoc", "what-if"], "description": "rolling = from scheduler; ad-hoc = manual trigger; what-if = from what-if API (not persisted by default)." },
    "runAt": { "type": "string", "format": "date-time" },
    "runBy": { "type": "string", "description": "User ID or 'system' if triggered by scheduler." },
    "status": { "type": "string", "enum": ["pending", "running", "completed", "failed"], "description": "Execution status." },
    "statusMessage": { "type": "string", "description": "Error message if failed." },
    "inputSnapshots": {
      "type": "array",
      "description": "Frozen state of Drivers, Formulas, Calibrations at run time (immutable for auditability).",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["driver", "formula", "calibration"] },
          "ref": { "type": "string" },
          "snapshot": { "type": "object", "description": "Full JSON representation at time of run." }
        }
      }
    },
    "outputs": {
      "type": "array",
      "description": "Per-target, per-period forecasted values and contributing-driver breakdown.",
      "items": {
        "type": "object",
        "properties": {
          "targetRef": { "type": "string" },
          "period": { "$ref": "#/components/schemas/PeriodSpec" },
          "forecastValue": { "$ref": "#/components/schemas/Money" },
          "driverContribution": {
            "type": "array",
            "description": "Attribution to each driver (for sensitivity analysis).",
            "items": {
              "type": "object",
              "properties": {
                "driverRef": { "type": "string" },
                "contributionAmount": { "$ref": "#/components/schemas/Money" }
              }
            }
          }
        }
      }
    },
    "runtime": { "type": "number", "description": "Milliseconds elapsed." },
    "engineVersion": { "type": "string", "description": "Version of forecast engine used (for reproducibility)." },
    "confidence": {
      "type": "object",
      "properties": {
        "overallLevel": { "type": "string", "enum": ["high", "medium", "low"] },
        "warnings": {
          "type": "array",
          "description": "Warning codes and messages (e.g., DRIVER_STALE, LOW_SAMPLE_COUNT, DIVIDE_BY_ZERO).",
          "items": {
            "type": "object",
            "properties": {
              "code": { "type": "string" },
              "message": { "type": "string" }
            }
          }
        }
      }
    },
    "promotedToBegrotingRef": { "type": "string", "description": "Reference to begroting-werkbestand object if promoted (frozen snapshot)." }
  },
  "required": ["scenarioRef", "horizon", "runAt", "runBy", "status", "outputs"]
}
```

### 7. Variance

Comparison of a ForecastRun against actuals for a closed period.

```json
{
  "title": "Variance",
  "type": "object",
  "properties": {
    "forecastRunRef": { "type": "string", "description": "Reference to most recent ForecastRun made before period close." },
    "actualSource": { "type": "string", "enum": ["grootboek", "realisatiesysteem"], "description": "Where actuals come from." },
    "period": { "$ref": "#/components/schemas/PeriodSpec" },
    "targetRef": { "type": "string", "description": "Cost center or programma-onderdeel." },
    "forecastValue": { "$ref": "#/components/schemas/Money" },
    "actualValue": { "$ref": "#/components/schemas/Money" },
    "delta": { "$ref": "#/components/schemas/Money", "description": "actualValue - forecastValue." },
    "deltaPct": { "type": "number", "description": "delta / forecastValue * 100." },
    "requiresExplanation": { "type": "boolean", "description": "True if |deltaPct| > 5%." },
    "attribution": {
      "type": "array",
      "description": "Decomposition of variance into causal components.",
      "items": {
        "type": "object",
        "properties": {
          "type": { "type": "string", "enum": ["volume-effect", "prijs-effect", "mix-effect", "indexering-effect", "onverklaard-residu"] },
          "amount": { "$ref": "#/components/schemas/Money" },
          "percentOfDelta": { "type": "number" }
        }
      }
    },
    "commentaryByOwner": { "type": "string", "description": "Free-text explanation by formula owner." },
    "commentaryAddedAt": { "type": "string", "format": "date-time" },
    "commentaryAddedBy": { "type": "string" },
    "flaggedFor": { "type": "string", "enum": ["investigation", "formula-recalibration", "resolved"], "description": "Action flag." },
    "computedAt": { "type": "string", "format": "date-time" }
  },
  "required": ["forecastRunRef", "period", "targetRef", "forecastValue", "actualValue", "delta"]
}
```

### 8. RollingForecastSchedule

Configuration of periodic forecast runs.

```json
{
  "title": "RollingForecastSchedule",
  "type": "object",
  "properties": {
    "code": { "type": "string", "description": "Unique identifier, e.g., monthly-standard, quarterly-detail." },
    "naam": { "type": "string" },
    "frequency": { "type": "string", "enum": ["daily", "weekly", "monthly", "quarterly"], "description": "Run cadence." },
    "runDay": { "type": "integer", "description": "Day of month (1-31) or day of week (1-7 for weekly) to run." },
    "runTime": { "type": "string", "description": "Time of day in HH:MM format, e.g., '05:00'." },
    "scenarioRefs": { "type": "array", "items": { "type": "string" }, "description": "Scenarios to include in each run." },
    "horizons": { "type": "array", "items": { "type": "string" }, "description": "Forecast horizons, e.g., ['18 months', '4 years']." },
    "triggers": { 
      "type": "array",
      "items": { "type": "string", "enum": ["calendar", "event-realisatie-closed", "event-driver-refreshed"] },
      "description": "What events trigger runs (in addition to schedule)."
    },
    "notificationRecipients": { "type": "array", "items": { "type": "string" }, "description": "User IDs to notify on completion." },
    "enabled": { "type": "boolean", "default": true }
  },
  "required": ["code", "frequency", "runDay", "runTime", "scenarioRefs", "horizons"]
}
```

## Seed Data

Reference objects per entity for development and testing:

### Drivers (5 examples)
```json
[
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Driver", "slug": "inwoners-totaal" },
    "code": "inwoners-totaal",
    "naam": "Totaal aantal inwonersin Gemeente Amsterdam",
    "eenheid": "eenheid",
    "domain": "demografie",
    "aggregationLevel": "organisatie",
    "sourceRef": "cbs-statline-37230NED",
    "provenance": "CBS-StatLine-table-37230NED",
    "refreshFrequency": "monthly",
    "lastRefreshedAt": "2026-05-15T10:00:00Z",
    "historicalSeries": [
      { "period": "2021-12", "value": 873000 },
      { "period": "2022-12", "value": 877500 },
      { "period": "2023-12", "value": 882300 },
      { "period": "2024-12", "value": 887200 },
      { "period": "2025-12", "value": 892100 }
    ],
    "baselineForecastRef": "driver-forecast-inwoners-totaal-realistisch-2027",
    "confidenceRationale": "CBS official prognose with 95% CI; 5-year historical stability."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Driver", "slug": "wmo-clienten-zwaar" },
    "code": "wmo-clienten-zwaar",
    "naam": "Aantal Wmo-cliënten met zware zorgbehoefte",
    "eenheid": "eenheid",
    "domain": "zorg",
    "aggregationLevel": "organisatie",
    "sourceRef": "vektis-wmo-2025",
    "provenance": "Vektis",
    "refreshFrequency": "quarterly",
    "lastRefreshedAt": "2026-04-30T09:30:00Z",
    "historicalSeries": [
      { "period": "2022-Q4", "value": 2280 },
      { "period": "2023-Q4", "value": 2340 },
      { "period": "2024-Q4", "value": 2410 },
      { "period": "2025-Q4", "value": 2480 }
    ],
    "confidenceRationale": "Vektis Wmo-realisatie data; growth trend 2-3% annually."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Driver", "slug": "leerlingen-primair" },
    "code": "leerlingen-primair",
    "naam": "Leerlingen primair onderwijs",
    "eenheid": "eenheid",
    "domain": "onderwijs",
    "aggregationLevel": "organisatie",
    "sourceRef": "duo-leerlingenprognose-2024",
    "provenance": "DUO",
    "refreshFrequency": "yearly",
    "lastRefreshedAt": "2025-09-15T08:00:00Z",
    "historicalSeries": [
      { "period": "2021", "value": 95400 },
      { "period": "2022", "value": 94200 },
      { "period": "2023", "value": 93100 },
      { "period": "2024", "value": 92200 },
      { "period": "2025", "value": 91500 }
    ],
    "confidenceRationale": "DUO prognose 2024-2070; declining trend due to demografische verschuiving."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Driver", "slug": "oppervlakte-openbare-ruimte" },
    "code": "oppervlakte-openbare-ruimte",
    "naam": "Oppervlakte openbare ruimte (m²)",
    "eenheid": "m2",
    "domain": "fysieke-leefomgeving",
    "aggregationLevel": "organisatie",
    "sourceRef": "bag-geom-amsterdam",
    "provenance": "BAG",
    "refreshFrequency": "quarterly",
    "lastRefreshedAt": "2026-03-20T11:00:00Z",
    "historicalSeries": [
      { "period": "2023-Q1", "value": 48500000 },
      { "period": "2024-Q1", "value": 48750000 },
      { "period": "2025-Q1", "value": 48900000 }
    ],
    "confidenceRationale": "BAG basis + lokale GIS; stable with small growth in new neighborhoods."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Driver", "slug": "cao-loonindexering-vng" },
    "code": "cao-loonindexering-vng",
    "naam": "VNG CAO loonindexering (jaarlijkse %-verandering)",
    "eenheid": "eenheid",
    "domain": "economie",
    "aggregationLevel": "organisatie",
    "sourceRef": "cpb-cea-prognose",
    "provenance": "CPB",
    "refreshFrequency": "yearly",
    "lastRefreshedAt": "2025-12-01T10:00:00Z",
    "historicalSeries": [
      { "period": "2021", "value": 1.5 },
      { "period": "2022", "value": 3.2 },
      { "period": "2023", "value": 4.8 },
      { "period": "2024", "value": 2.1 }
    ],
    "baselineForecastRef": "driver-forecast-cao-realistisch-2027",
    "confidenceRationale": "CPB CEP prognose 2026; indexering-aannames subject to macro-volatility."
  }
]
```

### CostRevenueFormulas (3 examples)
```json
[
  {
    "@self": { "register": "driver-based-forecasting", "schema": "CostRevenueFormula", "slug": "wmo-zorg-zwaar-formula" },
    "targetType": "programma-onderdeel",
    "targetRef": "wmo-zorg-zwaar-2027",
    "formula": "kosten_per_eenheid('wmo-zorg-zwaar') * volume('wmo-clienten-zwaar') + indexering('cao-vng', 'salaris') * 850000",
    "driverRefs": ["wmo-clienten-zwaar", "cao-loonindexering-vng"],
    "validFrom": "2027-01-01",
    "validTo": null,
    "calibrationRef": "calibration-wmo-zorg-zwaar-2024-2025",
    "ownerRef": "programma-controller-wmo",
    "peerReviewedBy": "hoofd-financien",
    "documentation": "Wmo zorg-zwaar kosten worden aangestuurd door (1) aantal cliënten met zware zorgbehoefte, (2) kosten per cliënt (uit Vektis-realisatie), en (3) jaarlijkse cao-loonindexering voor personeelskosten. Formule gekalibreerd op 2024-2025 realisatie."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "CostRevenueFormula", "slug": "onderwijs-lumpsum-formula" },
    "targetType": "programma-onderdeel",
    "targetRef": "onderwijs-lumpsum-2027",
    "formula": "volume('leerlingen-primair') * 6500 + indexering('cpi', 'general') * 2400000",
    "driverRefs": ["leerlingen-primair"],
    "validFrom": "2027-01-01",
    "calibrationRef": "calibration-onderwijs-lumpsum-2023-2025",
    "ownerRef": "programma-controller-onderwijs",
    "peerReviewedBy": "hoofd-financien",
    "documentation": "Onderwijs lumpsum-financiering (DUO-model) gebaseerd op leerlingenaantallen en vaste uitgavenverhoging per leerling (EUR 6.500 base + CPI-indexering). Formule volgt DUO-algoritme."
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "CostRevenueFormula", "slug": "ozb-opbrengst-formula" },
    "targetType": "grootboek-rekening",
    "targetRef": "4100-opbrengsten-ozb",
    "formula": "volume('oppervlakte-openbare-ruimte') * kosten_per_eenheid('ozb-tarief-per-m2') / 1000000",
    "driverRefs": ["oppervlakte-openbare-ruimte"],
    "validFrom": "2027-01-01",
    "calibrationRef": "calibration-ozb-2024",
    "ownerRef": "programma-controller-ozb",
    "peerReviewedBy": "hoofd-financien",
    "documentation": "OZB-opbrengsten berekend als oppervlakte openbare ruimte × tarief per m². Tarief jaarlijks vastgesteld door raad; formule neutraliseert groeien krimpgebieden via volume-aanpassing."
  }
]
```

### Scenarios (3 examples)
```json
[
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Scenario", "slug": "realistisch-2027" },
    "code": "realistisch-2027",
    "naam": "Realistisch scenario 2027-2030",
    "description": "Midpoint scenario voor meerjarenraming 2027; assumes trend extrapolatie van recente jaren, geen grote beleidsveranderingen.",
    "parentScenarioRef": null,
    "parameterOverrides": [],
    "policyMeasures": [],
    "tag": "realistisch",
    "createdAt": "2026-04-01T09:00:00Z",
    "createdBy": "strateeg-financien"
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Scenario", "slug": "groeigebied-noord-2027" },
    "code": "groeigebied-noord-2027",
    "naam": "Groeigebied Noord scenario 2027",
    "description": "Kind van realistisch-2027, met +8% inwoners in 2027-2030 ivm geplande woningbouw Noord.",
    "parentScenarioRef": "realistisch-2027",
    "parameterOverrides": [
      {
        "type": "driver-volume",
        "driverRef": "inwoners-totaal",
        "period": "2027-Q1",
        "overrideType": "deltaPct",
        "value": 2.0
      },
      {
        "type": "driver-volume",
        "driverRef": "inwoners-totaal",
        "period": "2028-Q1",
        "overrideType": "deltaPct",
        "value": 3.5
      }
    ],
    "policyMeasures": [
      {
        "decidesk_besluitRef": "rb-2025-12-woningbouw-noord",
        "rationale": "Gemeenteraad heeft woningbouwproject Noord goed gekeurd, verwachting 2.500 nieuwe bewoners."
      }
    ],
    "tag": "beleidsvariant",
    "createdAt": "2026-04-15T10:30:00Z",
    "createdBy": "strateeg-financien"
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Scenario", "slug": "pessimistisch-2027" },
    "code": "pessimistisch-2027",
    "naam": "Pessimistisch scenario 2027 (reserve-scenario)",
    "description": "Downside scenario: hogere werkloosheid, meer Wmo-aanvragen, minder OZB-inkomsten.",
    "parentScenarioRef": "realistisch-2027",
    "parameterOverrides": [
      {
        "type": "driver-volume",
        "driverRef": "inwoners-totaal",
        "period": "2027-Q1",
        "overrideType": "deltaPct",
        "value": -1.0
      },
      {
        "type": "driver-volume",
        "driverRef": "wmo-clienten-zwaar",
        "period": "2027-Q1",
        "overrideType": "multiplier",
        "value": 1.15
      }
    ],
    "tag": "pessimistisch",
    "createdAt": "2026-04-20T14:00:00Z",
    "createdBy": "strateeg-financien"
  }
]
```

### Calibrations (2 examples)
```json
[
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Calibration", "slug": "calibration-wmo-zorg-zwaar-2024-2025" },
    "formulaRef": "wmo-zorg-zwaar-formula",
    "period": "2023-2025",
    "derivedConstants": [
      {
        "constantName": "wmo-zorg-zwaar",
        "value": { "amount": 23840, "currency": "EUR" },
        "confidence": "high"
      }
    ],
    "rmse": 1200,
    "mape": 4.3,
    "r2": 0.94,
    "notes": "Goede fit over 3-jarige periode; kosten-per-cliënt relatief stabiel ondanks cao-indexering.",
    "acceptanceStatus": "accepted",
    "acceptanceRationale": "Vektis-realisatie data is betrouwbaar; fit >0.94 is excellent. Gebruikt voor 2027-2030 raming.",
    "acceptedBy": "hoofd-financien",
    "acceptedAt": "2026-03-15T11:00:00Z",
    "calibratedAt": "2026-03-10T09:30:00Z",
    "calibratedBy": "analist-financien"
  },
  {
    "@self": { "register": "driver-based-forecasting", "schema": "Calibration", "slug": "calibration-onderwijs-lumpsum-2023-2025" },
    "formulaRef": "onderwijs-lumpsum-formula",
    "period": "2023-2025",
    "derivedConstants": [
      {
        "constantName": "lumpsum-per-leerling",
        "value": { "amount": 6500, "currency": "EUR" },
        "confidence": "high"
      }
    ],
    "rmse": 350,
    "mape": 2.1,
    "r2": 0.97,
    "notes": "DUO-algoritme is stabiel; lokale variaties in voortijdig schoolverlaten hebben beperkte impact.",
    "acceptanceStatus": "accepted",
    "acceptanceRationale": "DUO-ramingen zijn bindend; R² >0.97 confirms model fit.",
    "acceptedBy": "hoofd-financien",
    "acceptedAt": "2026-03-12T14:30:00Z",
    "calibratedAt": "2026-03-08T10:00:00Z",
    "calibratedBy": "analist-financien"
  }
]
```

### RollingForecastSchedule (1 example)
```json
[
  {
    "@self": { "register": "driver-based-forecasting", "schema": "RollingForecastSchedule", "slug": "monthly-standard" },
    "code": "monthly-standard",
    "naam": "Standard monthly rolling forecast (18 months)",
    "frequency": "monthly",
    "runDay": 5,
    "runTime": "05:00",
    "scenarioRefs": ["realistisch-2027", "pessimistisch-2027", "optimistisch-2027"],
    "horizons": ["18 months"],
    "triggers": ["calendar", "event-driver-refreshed"],
    "notificationRecipients": ["strateeg-financien", "hoofd-financien", "analist-financien"],
    "enabled": true
  }
]
```

## Authorization & RBAC

- **Strateeg, Analist:** Create/edit Drivers, DriverForecasts, Scenarios; run ForecastRuns; create Calibrations.
- **Programma-controller:** Create/edit formulas (targetType=programma-onderdeel); review variance; add commentary; approve formulas for use.
- **Hoofd Financiën / Concerncontroller:** Approve calibrations (acceptanceStatus); promote ForecastRuns to begroting-werkbestand; review/approve scenarios.
- **Accountant:** Read-only access to all registers for audit purposes (via OpenRegister RBAC on views).
- **Admin:** Full access; manage reference packs; configure RollingForecastSchedules.

All mutations use OpenRegister's per-object RBAC (ownerRef, reviewedBy fields checked via AuthorizationService).

## Integration Points & Workflows

### Monthly Rolling Forecast Workflow (n8n)
1. RollingForecastSchedule triggers on day 5 at 05:00.
2. openconnector refreshes all active drivers from upstream sources.
3. ForecastRun.runBy = 'system', status = 'running'.
4. Engine executes all Scenarios in RollingForecastSchedule.scenarioRefs with 18-month horizon.
5. Outputs persisted; status = 'completed'.
6. Webhooks notify assigned recipients.
7. Rolling-forecast dashboard updates with new runs.

### Variance Computation Workflow (n8n, post-period-close)
1. Monthly period closes (e.g., 2026-04 closes on 2026-05-01).
2. bookkeeping-bbv-compliance provides realisatiecijfers for Q1-2026.
3. System computes Variance records for each (ForecastRun, target, period).
4. Attribution algorithm decomposes delta into volume, prijs, mix, indexering, residu.
5. If |delta %| > 5%, variance flagged with requiresExplanation = true; assigned to formula owner.
6. Owner adds commentary; commentary linked to next-period ForecastRun notes.

### Scenario Promotion to Begroting Workflow (n8n)
1. Concerncontroller initiates promotion: ForecastRun (realistisch-2027, 18 months) → begroting-werkbestand 2027.
2. System creates frozen snapshot (Driver, Formula, Calibration, Scenario state at run time).
3. pc-cyclus-workflow receives snapshot; populates begroting-regels with ForecastRun.outputs per target.
4. Snapshot linked to begroting via promotedToBegrotingRef.
5. If raad amends a programma-budget (e.g., +EUR 500k to Wmo), override captured as raads-amendement in Scenario; next-period variance attributes delta to this override.

### What-If API Workflow
1. Beleidsadviseur calls POST /api/forecast/what-if with base=realistisch-2027 + overrides.
2. Engine fetches latest ForecastRun for realistisch-2027.
3. Applies overrides (driver volumes, formula constants).
4. Re-executes formulas; returns deltas vs base per-formula, per-programma.
5. Response excludes persisted ForecastRun; short-lived compute result.

## Performance Targets & Caching

- **ForecastRun execution:** ≤ 5 seconds for 50 formulas × 18 months.
- **What-if API:** ≤ 2 seconds (same workload, no DB writes).
- **Variance decomposition:** ≤ 30 seconds for 50 targets per period.
- **Driver lookup & validation:** ≤ 100ms for 1000-row driver register (index on code).

Caching strategy:
- Driver register cached (30-minute TTL) during ForecastRun execution.
- Formula DSL parse tree cached (1-hour TTL) per formula version.
- DriverForecast.dataPoints lazy-loaded per scenario (paginated).

## Documentation & Training

- **journeydoc "Wmo-budget onderbouwen met driver-based":** Step-by-step walkthrough of creating a driver, calibrating a formula, and running a forecast for Wmo domain.
- **journeydoc "What-if doen: hervormingsagenda jeugd":** Policy advisor use case showing how to model budget shifts and present scenarios to portefeuillehouder.
- **Reference documentation** in docusaurus: DSL grammar, calibration methodology, variance attribution algorithm, approval workflows.
