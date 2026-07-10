# App Shell — Installable Scaffold Delta

**Spec refs**: financeq ADR-001 (information architecture), hydra ADR-024 (app manifest),
hydra ADR-042 (first-time setup wizard), hydra ADR-044 (menu architecture)

## ADDED Requirements

### Requirement: Installable Nextcloud App

financeq MUST be an installable Nextcloud app: the repository MUST contain a valid
`appinfo/info.xml` (id `financeq`, EUPL-1.2), a bootable `lib/AppInfo/Application.php`,
and a frontend build producing the app bundle, such that the existing release workflows
(`.forgejo/workflows/release-beta.yml`, `release-stable.yml`) package a signed, installable
artifact.

**Feature tier**: MVP

#### Scenario: Fresh install boots the shell

- GIVEN a Nextcloud instance with financeq installed and enabled
- WHEN a user opens the financeq navigation entry
- THEN the manifest-rendered app shell MUST load without PHP or JS errors
- AND the app icon and name MUST appear in the Nextcloud app menu

### Requirement: Six-Menu Manifest Contract

The app shell MUST be rendered from an ADR-024 manifest (`src/manifest.json`) that encodes
exactly the six top-level menus of financeq ADR-001 §1 — Begroting, Treasury, Subsidies,
Belastingen, Rapportage, Beheer — in that order. Tier-suffixed module surfaces MUST appear
only as sub-pages under their ADR-001 §2 assigned parent, never as top-level menus. The
manifest MUST pass the `@conduction/nextcloud-vue` manifest validator.

**Feature tier**: MVP

#### Scenario: Navigation shows exactly six top-level menus

- GIVEN financeq is installed and set up
- WHEN a concerncontroller opens the app
- THEN the navigation MUST show exactly six top-level menus in the ADR-001 §1 order
- AND no tier-suffixed module (e.g. Beleggingen, WOZ) MUST appear at the top level

#### Scenario: Manifest validation gates the build

- GIVEN the manifest is edited to add a seventh top-level menu
- WHEN the manifest check runs in CI
- THEN the check MUST fail, requiring an ADR-001 amendment before merge

### Requirement: First-Time Setup Gate

Per hydra ADR-042, financeq MUST gate all module surfaces behind a first-time-setup state
covering at minimum: OpenRegister available, financeq register imported, gemeente and
boekjaar selected. financeq MUST NOT write tenant-keyed seed data before the gate passes.

**Feature tier**: MVP

#### Scenario: Unconfigured install shows the setup surface

- GIVEN a fresh financeq install where no gemeente or boekjaar is configured
- WHEN a user opens any financeq menu
- THEN the setup surface MUST be shown instead of an empty module page
- AND no seed objects MUST have been written to OpenRegister

### Requirement: Persona-Driven Menu Visibility

Per financeq ADR-001 §7, menu visibility MUST follow persona: reguliere medewerkers see the
five operational menus and MUST NOT see Beheer; concerncontrollers see all six.

**Feature tier**: MVP

#### Scenario: Medewerker does not see Beheer

- GIVEN a user in the reguliere-medewerker persona group
- WHEN they open financeq
- THEN the navigation MUST show Begroting, Treasury, Subsidies, Belastingen, Rapportage
- AND MUST NOT show Beheer
