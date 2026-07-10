## 1. Register + schema definitions (config head)

- [ ] 1.1 Create the financeq register JSON (slug `financeq`) following the current fleet config-head convention (check pipelinq/procest register JSON location and shape)
- [ ] 1.2 Define the `programma` schema: `code` (string, required, unique within `boekjaar`), `naam` (string, required), `type` (enum `programma`|`doel`|`activiteit`), `parent` (self-relation, UUID, nullable), `boekjaar` (integer, required), `taakveld` (relation to `taakveld`), `actief` (boolean, default true)
- [ ] 1.3 Define the `taakveld` schema: `code` (string, required, CBS taakveldcode), `omschrijving` (string, required), `hoofdtaakveld` (string)
- [ ] 1.4 Verify schema slugs against the global lower(slug) collision rule; namespace if the fleet convention requires it

## 2. Import plumbing

- [ ] 2.1 Register a Repair step that imports the register JSON on install/upgrade (OR does not self-import its own register JSON)
- [ ] 2.2 Guard the Repair step so it works in CLI/repair contexts (no `isEnabledForUser`-based availability check — ADR-042 context)
- [ ] 2.3 Ship the CBS taakvelden lookup as neutral reference seed only; NO gemeente-specific programma seed before the ADR-042 setup gate passes

## 3. Access contract

- [ ] 3.1 Configure OR schema RBAC: write on `programma` restricted to the financeq begroting role; read open to authenticated consumers (planix, Rapportage, launchpad)
- [ ] 3.2 Document the read contract (register/schema slugs, query shapes for tree traversal and programma→doel laddering) in `docs/bbv-programma-tree.md`
- [ ] 3.3 Confirm launchpad's boundary is honoured: launchpad reads via runtime GraphQL only, no install-time dependency on financeq or OR

## 4. Verification

- [ ] 4.1 Clean-install e2e: install financeq on a fresh instance, confirm register + schemas imported, taakvelden lookup present, zero programma objects
- [ ] 4.2 Create a programma via OR API, rename it, and confirm a consumer-side read (plain OR query) reflects the rename with no local copy involved
- [ ] 4.3 Run `openspec validate publish-bbv-programma-tree-schema --strict` and resolve any errors
