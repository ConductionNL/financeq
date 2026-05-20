# Capability: Budget Compliance

**Spec:** budget-compliance
**Change:** budget-planning-control-other-t2
**Status:** proposed

## Description

Dutch government compliance layer for Shillinq: links budget lines to BBV programme structure, maps them to IV3 reporting categories, validates taakveld classification against the official BBV Bijlage IV code list, runs structural budget balance checks (structureel sluitend), and supports kadernota (budget policy memorandum) preparation and meerjarenplanning (multi-year planning). Targets Dutch municipalities and public bodies required to report under BBV (Besluit Begroting en Verantwoording).

## Stakeholders

- **Municipality Finance Officer (Gemeentelijk Financieel Medewerker)** — maintains BBV programme and taakveld mapping; runs structural balance checks; prepares kadernota; compiles IV3 export
- **Financial Controller (Financieel Controller)** — validates taakveld classification before budget publication; applies indexation for multi-year planning; generates prognose
- **CFO / Directeur Financiën** — reviews structural balance report; signs off on kadernota; approves structural financial developments

## Requirements

### REQ-COM-001: Link budget lines to BBV programme structure

Each BudgetAllocation can be linked to a BBV programme via the `bbvProgramme` field, enabling programme-level budget reporting as required by BBV Article 8.

**Acceptance criteria:**

GIVEN a Municipality Finance Officer is editing a BudgetAllocation "Jeugdzorg inkoop 2026"
WHEN they select `bbvProgramme: "6 - Sociaal domein"` from a dropdown of standard BBV programmes
THEN the allocation is tagged to that programme
AND the budget list can be filtered and grouped by BBV programme
AND programme subtotals (sum of all allocation ceilings, sum of actuals) are shown in the facet sidebar
AND the programme filter includes "Geen programma" to surface unlinked allocations

GIVEN all BudgetAllocations for fiscal year 2026 are tagged with a BBV programme
WHEN the Finance Officer generates a BBV programme summary
THEN a table is produced showing: programme code, programme name, total ceiling, total actuals, utilisation % — one row per programme, with a grand total row

### REQ-COM-002: Map budget lines to IV3 categories

BudgetAllocations can be tagged with an IV3 category (Informatie voor derden) for financial reporting to the central government.

**Acceptance criteria:**

GIVEN a Financial Controller edits a BudgetAllocation
WHEN they select `iv3Category: "B"` (Personeel) from the IV3 category list
THEN the allocation is tagged with that IV3 category
AND an IV3 summary report groups all budget allocation amounts and actuals by IV3 category
AND allocations without an IV3 category are flagged in the compliance report as "Ontbrekende IV3 categorie"
AND the IV3 summary can be exported to Excel or CSV via `CnMassExportDialog`

### REQ-COM-003: Validate taakveld classification of budget lines

The system validates that all BudgetAllocations carry a valid taakveld code before the budget can be published. Valid codes are drawn from BBV Bijlage IV.

**Acceptance criteria:**

GIVEN a budget "Gemeente Utrecht 2026" has 12 BudgetAllocations, 3 of which have no `taakveld` set and 1 has an invalid code "9.99"
WHEN the Finance Officer runs "Taakveld validatie" on the budget
THEN the system reports 4 issues: 3 missing taakveld codes and 1 invalid code
AND each issue shows the allocation name, the problematic field value, and the expected format (e.g. "0.4 - Overhead")
AND the budget cannot be set to `status: published` until all allocations pass taakveld validation
AND re-running validation after corrections shows 0 issues

GIVEN the Finance Officer enters taakveld "2.1" for an allocation
WHEN the field loses focus (blur validation)
THEN the system immediately validates the code against the BBV Bijlage IV list stored in app configuration
AND if invalid, shows inline error: "Ongeldige taakvelcode — raadpleeg BBV Bijlage IV"

### REQ-COM-004: Run structural budget balance check

The system checks whether the budget is structurally balanced: recurring (structural) revenues must cover recurring expenditures.

**Acceptance criteria:**

GIVEN a budget has BudgetAllocations where some are tagged `structural: true` with revenue or expenditure classification via IV3 category
WHEN the Finance Officer clicks "Structurele balans berekenen"
THEN the system computes: structural revenues = sum of revenue-type allocations where `structural: true`; structural expenditures = sum of expenditure-type allocations where `structural: true`
AND if expenditures > revenues: the result shows "Structureel tekort: €[amount]" in red
AND if revenues ≥ expenditures: the result shows "Structureel sluitend (overschot: €[amount])" in green
AND the result is displayed per BBV programme and in total
AND the balance report can be exported as PDF

### REQ-COM-005: Generate year-end forecast (prognose)

Financial controllers can generate and update a year-end spending forecast at any point during the fiscal year, supporting interim reporting to the council.

**Acceptance criteria:**

GIVEN a Budget is partway through the fiscal year
WHEN a Financial Controller opens the budget and enters a prognose amount of €2,650,000 (against ceiling of €2,800,000)
THEN the prognose is saved with the current date and the user's name
AND the budget detail shows: Begroting €2,800,000 | Prognose €2,650,000 | Werkelijk €1,820,000 | Verwacht eindresultaat: −€150,000 (meevaller)
AND a negative expected result (prognose < actuals trend) is flagged in amber
AND the Finance Officer can add a narrative explanation to the prognose record

GIVEN the Finance Officer updates the prognose three times during the year
WHEN they view the budget audit trail
THEN all three prognose entries are listed with their amounts, dates, and authors
AND only the most recent prognose is used in dashboard calculations

### REQ-COM-006: Prepare kadernota

The system supports preparation of the kadernota by aggregating structural financial developments and multi-year projections per BBV programme.

**Acceptance criteria:**

GIVEN a Municipality Finance Officer is in the "Kadernota" section
WHEN they enter structural financial developments (nieuwe baten and nieuwe lasten per BBV programme for years 2027–2030)
THEN the system stores these as structured entries linked to the relevant BBV programme and fiscal year
AND a kadernota overview is generated showing: per programme, per year — existing structural budget + new developments = updated structural total
AND the overview displays a multi-year balance line (structural revenues − structural expenditures) for each forecast year
AND the kadernota overview can be exported to Excel for further formatting before council submission

GIVEN structural developments have been entered for all programmes
WHEN the Finance Officer runs the structural balance check for year 2028
THEN the check incorporates both existing structural allocations and the entered structural developments

### REQ-COM-007: Enter structural financial developments

Finance officers can register structural financial developments (structurele financiële ontwikkelingen) that affect future budget years.

**Acceptance criteria:**

GIVEN a Finance Officer enters a structural development: "Loonkostenstijging CAO 2027" as a recurring expenditure of €180,000 per year for BBV programme "0 - Bestuur en ondersteuning"
WHEN the entry is saved
THEN it appears in the kadernota and meerjarenraming for years 2027 onward
AND the structural balance check for affected years includes this development
AND the entry can be marked as incidental (eenmalig) or structural (structureel)
AND incidental developments appear only in the year they apply and do not carry forward
