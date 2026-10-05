# Implementation Plan — the Sacred Engine (Modular Rules Engine)

Test-driven, incremental build order. Each task is small, cites the requirements it satisfies,
and writes Gherkin scenarios / unit tests before implementation. Depends on the canon-server
spec for the canon hooks API (speed-regime, scarcity, strata). The reference `rules-light`
module is built first; open-SRD modules (`tristat`, `gumshoe`) come after the contract is
proven.

- [ ] 1. Rules-engine project bootstrap
  - [ ] 1.1 Initialize `rules-engine/contract/` as a TypeScript package; add `@cucumber/cucumber`, ts-node, and a unit test runner
  - [ ] 1.2 Define the capability interfaces (`CapabilityManifest`, `RulesModule`, `CapabilityResult`, `Rng`) and the `RulesEngine` host skeleton
  - [ ] 1.3 Wire Cucumber with the three profiles (`logic`/`ttrpg`/`godot`) and a `World` carrying the active module + injected `Rng`; implement the `logic` driver; stub `ttrpg`/`godot`
  - _Requirements: 1.1, 1.2, 9.1, 9.2, 9.5_

- [ ] 2. Capability negotiation & graceful degradation
  - [ ] 2.1 Write `features/rules-engine/capability-negotiation.feature` (`@req-1`: declare capabilities; requesting an unsupported one returns `{supported:false}` not a throw) before implementing
  - [ ] 2.2 Implement `RulesEngine.capability<T>()` wrapping calls so unsupported yields a defined result; add unit tests
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5_

- [ ] 3. Module selection & composition
  - [ ] 3.1 Write `features/rules-engine/module-selection.feature` (`@req-6`: select by config id; unknown id fails loud; `activeModule()` exposes id/version/license/capabilities) before implementing
  - [ ] 3.2 Implement `RulesEngine.load(moduleId)` reading config (`SE_RULES_MODULE`), failing startup with a diagnostic on unknown/failed module
  - [ ] 3.3 Implement `activeModule()` surfacing the manifest incl. license id; unit tests
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 10.4_

- [ ] 4. Reference module: rules-light — character model
  - [ ] 4.1 Write `features/rules-engine/character-model.feature` (`@req-2`: declared fields, derived computation, (de)serialize round-trip) before implementing
  - [ ] 4.2 Implement rules-light `characterModel` (original minimal stats) with `fields()`, `computeDerived()`, `validate()`, `serialize()/deserialize()`
  - [ ] 4.3 Author `modules/rules-light/manifest.json` (license: original, CC BY-NC / PolyForm NC) and `LICENSE`; unit tests
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 7.1, 7.2, 7.3, 10.1_

- [ ] 5. Reference module: rules-light — resolution
  - [ ] 5.1 Write `features/rules-engine/resolution.feature` (`@req-3`: injected rng determinism, outcome shape, opaque `detail`) before implementing
  - [ ] 5.2 Implement rules-light `resolveCheck` consuming the injected `Rng`; return `{result, explanation, detail?}`
  - [ ] 5.3 Unit tests with a seeded `Rng` asserting deterministic outcomes
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 9.4_

- [ ] 6. Canon mapping (rules-light) against canon hooks
  - [ ] 6.1 Write `features/rules-engine/canon-mapping.feature` (`@req-5`: hooks -> difficulty/actor effects; hooks-unavailable uses `defaults()` and still resolves) before implementing
  - [ ] 6.2 Implement rules-light `canonMapping`: `fetchHooks()` against the canon API (speed-regime, scarcity, strata), `applyToDifficulty`, `applyToActor`, `defaults()`
  - [ ] 6.3 Integration test against a canon fixture / local canon API; test the defaults path when hooks are unreachable
  - _Requirements: 5.1, 5.2, 5.3, 5.4_

- [ ] 7. Advancement (optional) — rules-light
  - [ ] 7.1 Write `features/rules-engine/advancement.feature` (`@req-4`: optional; supported path updates character; unsupported path returns defined result)
  - [ ] 7.2 Decide whether rules-light declares `advancement`; implement if yes, otherwise assert the unsupported result via another (stub) module
  - _Requirements: 4.1, 4.2, 4.3_

- [ ] 8. Open-SRD module: tristat (OGL 1.0a)
  - [ ] 8.1 Create `modules/tristat/` with `manifest.json` (provenance: srd, OGL), `LICENSE-OGL`, and `OGL-SECTION-15.txt` with a correct Section 15 notice
  - [ ] 8.2 Implement tristat `characterModel` + `resolveCheck` (+ `canonMapping`) against the contract, using Tri-Stat dX open content
  - [ ] 8.3 Reuse the existing `.feature` files against the tristat module (same scenarios, different active module); unit tests
  - _Requirements: 8.1, 8.3, 8.4, 10.1, 10.2_

- [ ] 9. Open-SRD module: gumshoe (CC SRD, "GUMSHOE-compatible")
  - [ ] 9.1 Create `modules/gumshoe/` with `manifest.json` (provenance: srd, CC), `LICENSE-CC`, and `ATTRIBUTION.md`; avoid Pelgrane trademark/branding
  - [ ] 9.2 Implement gumshoe `characterModel` (investigative/general abilities, pools) + `resolveCheck` (+ `canonMapping`)
  - [ ] 9.3 Reuse the `.feature` files against the gumshoe module; unit tests
  - _Requirements: 8.2, 8.3, 8.4, 10.1, 10.2_

- [ ] 10. Licensing & provenance surfacing + docs
  - [ ] 10.1 Ensure every module `manifest.json` carries `license.{id, provenance, source, notice}` and that `activeModule()` includes the license id
  - [ ] 10.2 Add `modules/README.md` documenting each module's license and the **GURPS-deferred** rationale (no open SRD)
  - [ ] 10.3 Update root `CONTRIBUTING.md` to state SRD-derived modules keep their own license and cannot be relicensed NC (coordinate with canon-server task 1.3)
  - _Requirements: 10.1, 10.2, 10.3, 10.4_

- [ ] 11. CI wiring (rules-engine)
  - [ ] 11.1 Add rules-engine jobs to CI: unit tests + `cucumber-js -p logic` across all modules; fail the build on failure
  - [ ] 11.2 Confirm the `logic` target blocks the build (Req 9.5); document local run commands in `rules-engine/README.md`
  - _Requirements: 9.5_

- [ ] 12. Consumer driver stubs for parameterized targets
  - [ ] 12.1 Implement minimal `ttrpg`/`godot` drivers so `-p ttrpg` and `-p godot` run `@all` rules scenarios through the surface's view (test/startup evaluation scope)
  - _Requirements: 9.2, 9.3_

## Notes

- Build the **contract + rules-light** fully (tasks 1–7) before the SRD modules; the SRD
  modules then reuse the same scenarios by swapping the active module, which proves the
  capability contract is genuinely system-neutral.
- Depends on canon-server tasks providing the hooks API (speed-regime, scarcity, strata) for
  task 6; until then, task 6 can run against a canon fixture.
- GURPS is intentionally absent (no open SRD); recorded in `modules/README.md`.
- Surfaces (TTRPG web/Foundry, Godot) are their own specs; here they appear only as Pact/
  Gherkin targets.
