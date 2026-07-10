## 1. App skeleton

- [ ] 1.1 Create `appinfo/info.xml` — id `financeq`, name "FinanceQ", EUPL-1.2 license, category `office`, NC version range matching the current fleet (copy from pipelinq/procest), `img/app.svg` icon already present
- [ ] 1.2 Create `lib/AppInfo/Application.php` with SPDX `@license`/`@copyright` docblock tags; register navigation in PHP (not in `info.xml` routes — info.xsd nav-route gotcha)
- [ ] 1.3 Create `composer.json` with `check:strict` scripts (PHPCS, PHPMD, Psalm, PHPStan) matching peer apps; run `composer check:strict` clean
- [ ] 1.4 Create `package.json` + webpack config consuming `@conduction/nextcloud-vue` (all peers in ONE install, no `--legacy-peer-deps`); `src/main.js` mounting `CnAppRoot`

## 2. ADR-024 manifest with the six-menu contract

- [ ] 2.1 Create `src/manifest.json` with exactly six top-level menus in ADR-001 §1 order: Begroting, Treasury, Subsidies, Belastingen, Rapportage, Beheer (menu labels via i18n keys, English source keys)
- [ ] 2.2 Add placeholder sub-pages per the ADR-001 §2 placement table (Begroting > Productenraming / Investeringen / Mutaties; Treasury > Liquiditeitsprognose / Leningenportefeuille / Beleggingen / Schatkistbankieren; Belastingen > WOZ / Bezwaren / Heffingsverordeningen) so module specs land in reserved slots
- [ ] 2.3 Run the manifest validator (`validateManifest` / `npm run check:manifest`) clean
- [ ] 2.4 Adopt ADR-044: use the shared `buildManifest` pipeline from `@conduction/nextcloud-vue` in `src/main.js`; create empty `src/manifest.d/` fragment dir; mark Beheer config entries as settings-foldout candidates

## 3. First-time setup (ADR-042)

- [ ] 3.1 Declare setup preconditions: OpenRegister enabled, financeq register imported, gemeente (organisation) + boekjaar selected
- [ ] 3.2 Gate all six menus behind the setup state — fresh install shows the setup wizard surface, not empty module pages
- [ ] 3.3 No tenant-keyed seed data written before the preconditions pass (ADR-042 hard constraint)

## 4. Persona-driven menu visibility (ADR-001 §7)

- [ ] 4.1 Hide Beheer for reguliere medewerkers; show all six menus for concerncontrollers — wire via manifest visibility + NC group mapping, not a raw permission bitmask
- [ ] 4.2 Document the persona → menu mapping in `docs/`

## 5. Docs + CI truthfulness

- [ ] 5.1 Restore minimal `docs/` (landing page + one page per top-level menu) so `.forgejo/workflows/documentation.yml` (`source-folder: docs`) publishes real content
- [ ] 5.2 Verify `release-beta.yml` / `release-stable.yml` can package the app (build produces `js/`, `appinfo/info.xml` valid against info.xsd)
- [ ] 5.3 Run `openspec validate bootstrap-installable-app-scaffold --strict` and resolve any errors
