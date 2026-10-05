# Requirements — the Sacred Engine (Modular Rules Engine)

## Introduction

The Rules Engine is the second independently pluggable axis of the Sacred Engine (the first
being the canon pack; the third being the game surface). It defines **how play resolves** and
is deliberately **modular**: a GM composing a deployment selects one rules module, which is
co-located with the canon service and the surface for that campaign.

Rules systems disagree at a deep level about what a "character" or a "check" even is, so this
spec does **not** attempt a single universal schema of all TTRPG mechanics (which would be a
leaky, all-fitting-none abstraction). Instead it defines a **capability-based rules
contract**: a small set of capabilities the surfaces actually need, which each module
declares and implements. Surfaces negotiate against declared capabilities and degrade
gracefully when a module does not support one.

### Module roster (initial)

- **rules-light** — the project's **original** reference module. License: CC BY-NC 4.0 (text) /
  PolyForm Noncommercial 1.0 (code). Always shipped; the default.
- **tristat** — a module implementing **Tri-Stat dX**, whose SRD is open under the **OGL 1.0a**.
  Bundled under the OGL with a required Section 15 notice; not relicensed as NonCommercial.
- **gumshoe** — a **GUMSHOE-compatible** module built on the GUMSHOE SRD, which is open under a
  **Creative Commons** license. Bundled under that CC license with attribution; the Pelgrane
  brand/trademark and logos are not used.
- **GURPS — deferred.** GURPS has **no open/forkable SRD** (GURPS Lite is share-only under
  Steve Jackson Games' own terms, not an OGL/CC SRD). It is explicitly out of scope and is
  recorded as deferred so it is not mistaken for an oversight.

### Licensing principle

Each rules module declares its own license and provenance in its metadata. The **capability
contract itself** is the project's own work (CC BY-NC / PolyForm NC). Modules derived from
open SRDs retain **their** licenses and required notices and are **not** relicensed as
NonCommercial. This produces a deliberate mixed-license repository with per-module license
files.

### Relationship to the canon-server spec

The Canon Server stays world-only and exposes **rules-readable hooks** (speed-regime +
characteristics, resource scarcity, strata ranks) over its HTTP/S API (canon-server
Requirement 12). A rules module *reads* those hooks and maps them into mechanics; the canon
server has no knowledge of any rules system.

### Roles (consistent with canon-server spec)

- **GM / operator** — selects and composes the rules module into a deployment.
- **Player** — end-user; interacts through the surface.
- **Pact consumer** — software component calling an API.

### Working defaults (overridable)

- Default module: **rules-light**.
- Contract/host language: **TypeScript** (shared with canon server and JS/TS surfaces).
- Behavior tests: **Cucumber.js**, parameterized by target (`logic` / `ttrpg` / `godot`),
  consistent with the canon-server spec.

---

## Requirements

### Requirement 1 — Capability-based rules contract

**User Story:** As a surface developer, I want to query what a rules module can do rather than
assume a fixed mechanics model, so that different systems can plug in without rewriting the
surface.

#### Acceptance Criteria

1. The project SHALL define a **rules contract** that enumerates a small, named set of
   capabilities (for example `characterModel`, `resolveCheck`, `advancement`, `canonMapping`).
2. A rules module SHALL declare which capabilities it supports via machine-readable metadata.
3. WHEN a surface queries a module THEN the module SHALL report its supported capabilities and
   the data each capability requires or produces.
4. IF a surface requests a capability a module does not support THEN the system SHALL respond
   with a defined "unsupported" result and the surface SHALL degrade gracefully (no crash).
5. The rules contract SHALL NOT encode any single system's mechanics as the universal model.

### Requirement 2 — Character model capability

**User Story:** As a surface, I want a module to describe the shape of a character sheet, so
that I can render and edit characters generically for any system.

#### Acceptance Criteria

1. A module supporting `characterModel` SHALL declare the fields that define a character
   (for example attributes, skills/abilities, pools, derived values) with types and
   constraints.
2. The surface SHALL render and validate characters from the **declared** field definitions
   rather than any hard-coded system assumption.
3. WHEN a module declares derived values THEN the module SHALL provide the means to compute
   them from base fields.
4. The character model SHALL be serializable so a character can be stored and reloaded.

### Requirement 3 — Resolution capability

**User Story:** As a surface, I want to ask a module to resolve an action, so that I do not
embed any system's dice math.

#### Acceptance Criteria

1. A module supporting `resolveCheck` SHALL accept an action request (actor character, action
   descriptor, difficulty/opposition, optional modifiers) and return a structured outcome.
2. The outcome SHALL include at least a success/failure (or graded result) and a
   human-readable explanation; it MAY include system-specific detail.
3. The resolution capability SHALL be deterministic given an injected randomness source, so
   that tests can assert outcomes.
4. The surface SHALL treat the outcome structure as opaque beyond the common fields, so that
   system-specific detail does not leak into surface logic.

### Requirement 4 — Advancement capability (optional)

**User Story:** As a GM, I want modules to optionally support character advancement, so that
systems with XP/point-spend can grow characters while simpler systems can omit it.

#### Acceptance Criteria

1. `advancement` SHALL be an **optional** capability a module may or may not declare.
2. WHEN a module supports `advancement` THEN it SHALL accept a character plus an advancement
   request and return an updated character or a defined rejection with a reason.
3. IF a module does not support `advancement` THEN requesting it SHALL yield the defined
   "unsupported" result per Requirement 1.4.

### Requirement 5 — Canon mapping capability

**User Story:** As a GM, I want the chosen rules module to reflect the world (speed-regime
pressure, scarcity, strata) in its mechanics, so that play feels like *this* setting and not
a generic system.

#### Acceptance Criteria

1. A module supporting `canonMapping` SHALL read canon hooks (speed-regime + characteristics,
   resource scarcity, strata ranks) from the canon service's HTTP/S API.
2. The module SHALL translate those hooks into system-appropriate effects (for example a
   difficulty modifier derived from scarcity, or an advantage derived from stratum rank).
3. The mapping SHALL be defined by the module (each system maps the world its own way) and
   SHALL NOT require the canon server to know anything about the system.
4. IF canon hooks are unavailable THEN the module SHALL apply documented defaults and SHALL
   NOT fail resolution solely due to missing hooks.

### Requirement 6 — GM deployment-time selection & composition

**User Story:** As a GM/operator, I want to choose a rules module by configuration and run it
co-located with canon and a surface, so that assembling a campaign is configuration, not code.

#### Acceptance Criteria

1. The system SHALL allow a GM to select the active rules module for a deployment via
   configuration (module id), not code changes.
2. WHEN a deployment starts THEN exactly one rules module SHALL be active and SHALL be
   co-located with the canon service and the surface.
3. IF a configured module id is unknown or fails to load THEN the deployment SHALL fail to
   start with a clear diagnostic rather than silently running without rules.
4. The system SHALL expose the active module's id, version, license, and supported
   capabilities for display by the surface.

### Requirement 7 — Reference module: rules-light

**User Story:** As the project owner, I want an original rules-light module shipped by
default, so that the system is immediately playable with clean, non-commercial licensing.

#### Acceptance Criteria

1. The project SHALL ship an original **rules-light** module as the default.
2. The rules-light module SHALL support `characterModel`, `resolveCheck`, and `canonMapping`,
   and MAY support `advancement`.
3. The rules-light module SHALL be entirely original work licensed under CC BY-NC 4.0 (content)
   and PolyForm Noncommercial 1.0 (code), with no dependence on any third-party SRD.

### Requirement 8 — Open-SRD modules: Tri-Stat and GUMSHOE

**User Story:** As a GM, I want to run the game under a familiar open system, so that I can use
Tri-Stat or a GUMSHOE-compatible ruleset within the same world and surfaces.

#### Acceptance Criteria

1. The **tristat** module SHALL implement the rules contract using Tri-Stat dX content under
   the **OGL 1.0a**, and SHALL include the OGL text and a correct Section 15 notice.
2. The **gumshoe** module SHALL implement the rules contract using the GUMSHOE SRD under its
   **Creative Commons** license, SHALL include the required attribution, and SHALL avoid the
   Pelgrane trademark/branding (described as "GUMSHOE-compatible").
3. Each SRD-derived module SHALL carry its own license file and SHALL NOT be relicensed as
   NonCommercial.
4. WHERE a module's SRD requires specific notices THE module SHALL include those notices such
   that a distribution of the module remains compliant.

### Requirement 9 — Behavior-driven, parameterized tests (consistent with canon-server)

**User Story:** As a maintainer, I want rules behavior specified in Gherkin and runnable
against targets, so that modules are verified consistently with the rest of the project.

#### Acceptance Criteria

1. Rules-engine behavior SHALL be expressed as Gherkin `.feature` files executed with
   Cucumber.js.
2. The suite SHALL be parameterizable by target (`logic` / `ttrpg` / `godot`) consistent with
   the canon-server spec, selecting the binding layer for the steps.
3. Scenarios SHALL be tagged for traceability (for example `@req-3`) and for scope
   (`@all`, `@logic-only`, `@ttrpg`, `@godot`).
4. Resolution scenarios SHALL use an injected randomness source so outcomes are deterministic
   and assertable (supports Requirement 3.3).
5. WHEN CI runs THEN at minimum the `logic` target SHALL execute and SHALL block the build on
   failure.

### Requirement 10 — Licensing & provenance surfacing

**User Story:** As a contributor and as a GM, I want each module's license and provenance to be
explicit, so that mixed licensing stays compliant and transparent.

#### Acceptance Criteria

1. Every rules module SHALL declare its license id and provenance (original, or SRD source) in
   machine-readable metadata.
2. The repository SHALL contain a per-module license file for any module derived from an
   external SRD (OGL text for tristat; CC license + attribution for gumshoe).
3. The project `CONTRIBUTING.md` SHALL state that SRD-derived modules keep their own license and
   cannot be relicensed as NonCommercial (consistent with the canon-server community scaffolding).
4. WHEN the active module is displayed (Requirement 6.4) THEN its license id SHALL be included.

### Requirement validation note

EARS statements above are the normative source and are validated by Gherkin scenarios tagged
with the corresponding requirement id. Scenarios live as executable `.feature` files and are
cross-referenced from the design document so prose and executable behavior do not drift.
