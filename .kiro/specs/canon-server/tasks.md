# Implementation Plan — the Sacred Engine (Canon Server)

Test-driven, incremental build order. Each task is small, references the requirements it
satisfies, and (where it adds behavior) writes Gherkin scenarios or unit tests before the
implementation. The `logic` Cucumber target is wired early because every later slice depends
on it. Consumer (`ttrpg`/`godot`) drivers are stubbed; full consumer apps are separate specs.

- [ ] 1. Community & licensing scaffolding
  - [ ] 1.1 Finish `LICENSE-ASSETS` by appending the full CC BY-NC 4.0 legal text (header already present)
  - [ ] 1.2 Create root `LICENSE` explaining the dual-license model and pointing to `LICENSE-CODE` and `LICENSE-ASSETS`
  - [ ] 1.3 Write `CONTRIBUTING.md` with contribution lanes (canon/core, canon/lore, ttrpg, videogame, server), the license that applies to each, the mandatory-CLA section, and the externally-licensed-rules note (no NC relicensing)
  - [ ] 1.4 Create `.github/workflows/cla.yml` using `cla-assistant/github-action`, triggered on `issue_comment` and `pull_request_target`, with placeholder `GITHUB_TOKEN` and `PERSONAL_ACCESS_TOKEN`
  - [ ] 1.5 Write professional root `README.md` with the shared-world + two-games vision, a Licensing section (dual model), the IP-safety note, and a How to Contribute section linking `CONTRIBUTING.md`
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 9.4_

- [ ] 2. Setting-neutral contract & pack schema
  - [ ] 2.1 Write `canon/CONTRACT.md` describing the setting-neutral contract (vessel/sections, strata, factions, resources, speed regime, optional lore) as resource definitions, server-ready in style
  - [ ] 2.2 Author `canon/schema/core.schema.json` (JSON Schema 2020-12): manifest fields, required core structures, `speedRegime` enum (`halting|crawling|running`), regime characteristics, required `contractVersion`
  - [ ] 2.3 Author `canon/schema/lore.schema.json` as the soft/advisory shape for lore entries and references
  - [ ] 2.4 Add unit tests asserting the schemas accept a known-good fixture and reject representative bad core (missing stratum field, bad `speedRegime`)
  - _Requirements: 2.1, 2.4, 3.1, 3.2, 5.1_

- [ ] 3. Canon-server project bootstrap
  - [ ] 3.1 Initialize `canon-server/` as a TypeScript Node project (tsconfig, scripts), add Fastify, ajv, `@cucumber/cucumber`, ts-node, and a unit test runner
  - [ ] 3.2 Create `src/app.ts` exporting `buildApp()` returning a Fastify instance (no network), and `src/server.ts` with `start()` handling TLS/HTTP and exit codes
  - [ ] 3.3 Add `/v1/health` returning liveness plus loaded/rejected pack counts (seeded empty for now)
  - [ ] 3.4 Wire Cucumber with the three profiles (`logic`/`ttrpg`/`godot`), the `World`, and the `CanonDriver` interface; implement the `logic` `InjectDriver` via `fastify.inject()`; stub `ttrpg`/`godot` drivers (pending)
  - [ ] 3.5 Write a smoke `.feature` (`@all`) hitting `/v1/health` to prove the harness runs under `-p logic`
  - _Requirements: 1.1, 1.5, 10.1, 11.1, 11.2, 11.3, 11.6, 11.8_

- [ ] 4. Pack loading & strict core validation
  - [ ] 4.1 Write `features/canon-server/core-validation.feature` (`@req-2`, incl. `@logic-only` rejection + fail-loud-startup scenarios) before implementing
  - [ ] 4.2 Implement pack discovery + `CoreLoader` validating `core/*` against `core.schema.json` with ajv; build the `PackRegistry` of servable packs only
  - [ ] 4.3 On validation failure: reject the pack, record a descriptive error, exclude from registry; if no pack is servable, fail startup non-zero with diagnostic
  - [ ] 4.4 Add unit tests for the loader/registry; make the `@logic-only` scenarios pass
  - _Requirements: 2.2, 2.3, 2.5, 5.2, 10.1_

- [ ] 5. Core read API & versioning
  - [ ] 5.1 Write `features/canon-server/http-api.feature` (`@req-1`: `/v1` root, `/v1/packs`, `/v1/packs/:id`, `/v1/packs/:id/core`, 404s, JSON content type) before implementing
  - [ ] 5.2 Implement the `/v1` routes over the registry with the consistent error body `{ error: { code, message } }`
  - [ ] 5.3 Make the `http-api` scenarios pass under `-p logic`
  - _Requirements: 1.2, 1.3, 1.4, 1.5, 7.1_

- [ ] 6. Speed regime endpoint
  - [ ] 6.1 Write `features/canon-server/speed-regime.feature` (`@req-3`) before implementing
  - [ ] 6.2 Implement `/v1/packs/:id/speed-regime` returning regime + characteristics; ensure invalid regime was already rejected at load (cross-check with task 4)
  - [ ] 6.3 Make the speed-regime scenarios pass
  - _Requirements: 3.3, 3.4_

- [ ] 7. Optional lore layer & advisory linter
  - [ ] 7.1 Write `features/canon-server/lore-advisory.feature` (`@req-4`: non-blocking warnings, unresolved-reference warning, core-only opt-out, no-lore pack still loads) before implementing
  - [ ] 7.2 Implement `LoreLoader` + pure-function `LoreLinter` returning `{ score, warnings[] }`; resolve lore refs against validated core; never throw/block on advisory issues
  - [ ] 7.3 Implement `/v1/packs/:id/lore` (404 if no lore) and `?lore=false` opt-out on `/v1/packs/:id`; attach `validation` block to lore responses
  - [ ] 7.4 Add unit tests for linter scoring and reference resolution; make the lore scenarios pass
  - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 7.2_

- [ ] 8. Pack pluggability
  - [ ] 8.1 Write `features/canon-server/pack-pluggability.feature` (`@req-5`: multiple packs, select by id, swap/remove without code changes given same contract version) before implementing
  - [ ] 8.2 Add a second minimal fixture pack to prove multi-pack serving and selection; confirm removing a pack needs no code change
  - [ ] 8.3 Make the pluggability scenarios pass
  - _Requirements: 5.3, 5.4_

- [ ] 9. Default pack: Dustline (copyright-safe)
  - [ ] 9.1 Author `canon/packs/dustline/pack.json` (`id: dustline`, `speedRegime: halting`, `contractVersion`) and `core/*` (vessel/sections, strata, factions, resources) with original names only
  - [ ] 9.2 Author a starter `lore/*` for Dustline with at least one intentionally-coherent set of references (and a test fixture variant with an unresolved reference for the linter)
  - [ ] 9.3 Validate Dustline against the core schema and run the linter; confirm it loads and serves
  - _Requirements: 5.5, 9.1, 9.2, 9.3_

- [ ] 10. Contract types from a single source
  - [ ] 10.1 Generate TypeScript types from `canon/schema/*.json` and consume them in the server routes/handlers
  - [ ] 10.2 Document (and script) regeneration; note the Godot type-stub approach for later
  - _Requirements: 7.4_

- [ ] 11. Pact consumer contracts & provider verification
  - [ ] 11.1 Add committed Pact consumer contracts under `contracts/` for `web`, `foundry`, and `godot` (hand-authored for godot) covering the core read endpoints they rely on
  - [ ] 11.2 Implement provider verification against all contracts in the server test suite
  - [ ] 11.3 Document the broker-adoption seam (PactFlow) without requiring it now
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [ ] 12. CI wiring
  - [ ] 12.1 Add `.github/workflows/ci.yml` running: schema/unit tests, `cucumber-js -p logic`, and Pact provider verification; fail the build on any failure
  - [ ] 12.2 Ensure the `logic` Cucumber target blocks the build (Req 11.8) and document local `start`/`validate`/`test` commands in `canon-server/README.md`
  - _Requirements: 6.2, 6.3, 10.2, 10.3, 10.4, 11.8_

- [ ] 13. Consumer driver stubs for parameterized targets
  - [ ] 13.1 Implement minimal `ttrpg` and `godot` drivers so `-p ttrpg` and `-p godot` run the `@all` scenarios against the running/injected server (test/startup evaluation scope)
  - [ ] 13.2 Confirm the same scenarios pass across `logic` and the two consumer targets where applicable
  - _Requirements: 11.4, 11.5, 7.2_

## Notes

- Tasks 2 and 3 can proceed in parallel after task 1; task 4 depends on both.
- Dustline (task 9) is intentionally late so it is authored against finished schema +
  validation, but a tiny fixture pack is used earlier (tasks 3–8) for test setup.
- Consumer apps (TTRPG web/Foundry, Godot game) and the TTRPG SRD are **out of scope** here
  and will each get their own spec; this plan only guarantees the server honors their
  contracts via Pact + parameterized Gherkin.
