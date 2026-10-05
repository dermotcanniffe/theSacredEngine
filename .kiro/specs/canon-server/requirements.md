# Requirements — the Sacred Engine (Canon Server)

## Introduction

The Sacred Engine is a community-driven, open-source, non-commercial project for building
games set aboard a perpetual vessel crossing a hostile, ruined world. The project
deliberately separates three independently swappable concerns so the community can
evolve each one without breaking the others, and so the specific setting flavor can be
replaced if intellectual-property pressure ever requires it:

1. **Canon** — the world: structural *core* rules plus optional narrative *lore*.
2. **Rules engine** — how play resolves. Modular and pluggable: a reference original
   rules-light system by default, with additional modules for systems that have free/open
   SRDs (Tri-Stat dX under OGL, GUMSHOE-compatible under its CC SRD). The modular rules
   engine is designed in its own spec (`rules-engine`); this spec only notes where canon
   exposes data the rules engine reads.
3. **Game surface** — what a player touches: a TTRPG (web app and/or Foundry VTT module)
   and a Godot video game.

### Roles & deployment model

Three distinct roles must not be conflated:

- **GM / operator** — the person who *deploys and composes* the services for a single
  game/campaign. The GM selects a canon pack, a rules module, and a surface, and runs them
  **co-located** as one deployment ("campaign instance"). The GM is an operator, not merely
  an end-user.
- **Player** — an end-user at the table. Players interact through the surface; they do not
  call the canon API directly.
- **Pact consumer** — a *software component* in the deployment that calls the canon API (the
  web app, the Foundry module, the Godot client). "Consumer" is a contract-testing term for a
  piece of software, not a person.

A **deployment / campaign instance** is the unit of operation: `{ canon pack + rules module +
surface }`, selected by the GM via configuration and co-located. The canon server is one
service within that bundle. It stays world-only (it does not contain rules) but exposes
**rules-readable hooks** (e.g. speed-regime pressure, resource scarcity, strata) that a
co-located rules module reads via the same local HTTP/S contract.

At the center is the **Canon Server**: an HTTP/S service that loads canon packs, validates
them against contracts, and serves them to multiple consumers. The canon is **pluggable**:
the server exposes a setting-neutral contract, and the concrete setting lives inside a
swappable *pack*. The default pack is **Dustline**, a slow desert convoy.

A core design axis is the **speed regime** of the vessel, because speed changes the whole
shape of play: a slow, frequently-stopping vessel (Metro-like, "halting") is an
exploration/survival experience, while a fast, never-stopping vessel (Snowpiercer-like,
"running") is a sealed social-intrigue experience.

Three distinct enforcement mechanisms keep the system honest:

- **Pact contracts** govern the HTTP API boundary (consumer ↔ server). Binary pass/fail.
- **Core schema** (JSON Schema) governs structural validity of core canon. Binary pass/fail; invalid core is rejected.
- **Lore validation** is an *advisory* linter over the lore layer. It *helps* keep lore
  coherent and sensible but does **not** guarantee it and does **not** block loading.

### Scope of this spec

This spec covers the **Canon Server** and the **contract/validation system** (core schema,
lore lint, Pact strategy), the **pack format**, the **default Dustline pack skeleton**, and
the **OSS community scaffolding** (dual licensing, CLA, CONTRIBUTING, README). It also
defines the **consumer-facing contract** that the TTRPG (web + Foundry VTT) and the Godot
game depend on. Full implementation of the consumers, the TTRPG SRD, and the Godot game are
out of scope for this spec and will be addressed in their own specs; here we only guarantee
the server honors their contracts via **test/startup-time evaluation**.

### Working defaults (overridable)

- Default canon pack: **Dustline** (slow desert convoy).
- Default speed regime: **halting**.
- Canon server stack: **TypeScript + Fastify**, `ajv` for JSON Schema, `@pact-foundation/pact` for JS consumers.
- Licenses: code under **PolyForm Noncommercial 1.0**; assets/content under **CC BY-NC 4.0**.

---

## Requirements

### Requirement 1 — Canon Server serves canon over HTTP/S

**User Story:** As a Pact consumer (the web app, Foundry module, or Godot client software in a
GM's deployment), I want to fetch canon data from a single HTTP/S service, so that all
surfaces in the campaign instance share one source of truth.

#### Acceptance Criteria

1. WHEN the server starts THEN the Canon Server SHALL load all available canon packs from a configured location and expose them over HTTP.
2. WHEN a client requests the API version root THEN the Canon Server SHALL respond with the supported contract version(s) and a list of available packs.
3. The Canon Server SHALL expose versioned routes under a version prefix (for example `/v1/...`) so that contract changes do not silently break existing consumers.
4. WHEN a client requests a resource that does not exist THEN the Canon Server SHALL respond with an HTTP 404 and a machine-readable error body.
5. The Canon Server SHALL serve responses as JSON with an explicit content type.
6. WHERE TLS is configured THEN the Canon Server SHALL serve over HTTPS; WHERE TLS is not configured THEN the server SHALL serve over HTTP for local development only.

### Requirement 2 — Core layer with strict validation

**User Story:** As a maintainer, I want core canon validated strictly, so that structurally
invalid worlds can never be served to a game.

#### Acceptance Criteria

1. The Canon Server SHALL define a strict JSON Schema (the "core schema") describing required core structures: vessel/sections, strata, factions, resources, and speed regime.
2. WHEN a pack's core data is loaded THEN the Canon Server SHALL validate it against the core schema.
3. IF a pack's core data fails core-schema validation THEN the Canon Server SHALL reject that pack, exclude it from being served, and record a descriptive error.
4. The core schema SHALL require every pack to declare the contract version it implements.
5. WHEN core-schema validation fails at startup for all packs THEN the Canon Server SHALL fail startup with a non-zero exit and a clear diagnostic rather than serving nothing silently.

### Requirement 3 — Speed regime as a first-class core concept

**User Story:** As a designer, I want the vessel's speed regime to be an explicit core
property, so that consumers can switch their core gameplay loop based on it.

#### Acceptance Criteria

1. The core schema SHALL define a `speedRegime` enumeration with at least the values `halting`, `crawling`, and `running`.
2. The core schema SHALL allow a pack to declare, per regime, whether egress from the vessel is permitted, the resupply model, the primary threat source (external/internal), and the spatial model (stops/regions vs. ordered sections).
3. WHEN a consumer reads a pack THEN the Canon Server SHALL make the pack's `speedRegime` and its associated characteristics available in the response.
4. IF a pack declares a `speedRegime` outside the enumeration THEN the Canon Server SHALL treat it as a core-schema validation failure per Requirement 2.

### Requirement 4 — Optional lore layer with advisory validation

**User Story:** As a worldbuilder, I want to attach a narrative lore layer that is checked
for coherence but not rigidly enforced, so that storytelling stays flexible while still
getting helpful guidance.

#### Acceptance Criteria

1. The Canon Server SHALL treat the lore layer as **optional**: a pack with no lore SHALL still load and serve its core.
2. WHEN lore is present THEN the Canon Server SHALL run an **advisory** lore validator that produces warnings and/or a coherence score.
3. The lore validator SHALL check referential coherence (for example, lore that references a faction, section, or stratum which exists in the core) and SHALL emit a warning for unresolved references.
4. IF the lore validator produces warnings THEN the Canon Server SHALL still serve the pack and SHALL expose the warnings/score alongside the lore so consumers can decide how to treat them.
5. The lore validator SHALL NOT block loading or serving on the basis of advisory warnings (it "helps, not guarantees").
6. The Canon Server SHALL allow a consumer to request core-only (excluding lore) so a surface may opt out of lore entirely.

### Requirement 5 — Pluggable, swappable canon packs

**User Story:** As a maintainer, I want the setting to live in a self-contained swappable
pack behind a setting-neutral contract, so that the specific flavor can be replaced without
rewriting the server or the consumers.

#### Acceptance Criteria

1. The Canon Server SHALL define a setting-neutral contract (documented in `canon/CONTRACT.md`) that no concrete setting detail leaks into.
2. The Canon Server SHALL load each pack as a self-contained unit identified by a stable pack id.
3. WHEN multiple packs are present THEN the Canon Server SHALL serve each independently and SHALL allow a consumer to select a pack by id.
4. A pack SHALL be removable or replaceable without code changes to the server, provided the replacement satisfies the same contract version.
5. The Canon Server SHALL ship a default pack with id `dustline` that satisfies the contract and declares the `halting` speed regime.

### Requirement 6 — Consumer-driven contracts via Pact

**User Story:** As a consumer developer, I want to declare exactly what I need from the
canon API and have the server verified against it, so that server changes cannot silently
break my surface.

#### Acceptance Criteria

1. The project SHALL maintain Pact consumer contracts for the identified consumers: the TTRPG web app, the Foundry VTT module, and the Godot game.
2. WHEN the Canon Server is built in CI THEN the build SHALL run Pact provider verification against all available consumer contracts.
3. IF provider verification fails against any consumer contract THEN the CI build SHALL fail.
4. The project SHALL store Pact contracts in the repository (no hosted broker required initially) AND SHALL document a seam for adopting a Pact Broker/PactFlow later.
5. WHERE a consumer lacks mature native Pact tooling (the Godot game) THE project SHALL represent that consumer's expectations as a committed contract that the server is still verified against, satisfying evaluation at test/startup time.

### Requirement 7 — Multiple consumers share one contract

**User Story:** As a player, I want the TTRPG and the video game to reflect the same world,
so that lore and structure stay consistent across surfaces.

#### Acceptance Criteria

1. The Canon Server SHALL serve the same core and lore data to all consumers through the same versioned API.
2. The TTRPG web app, the Foundry VTT module, and the Godot game SHALL each be able to consume core alone or core plus lore.
3. WHEN the contract version is incremented in an incompatible way THEN the Canon Server SHALL continue to serve the prior version until consumers migrate, OR SHALL document the migration path.
4. The project SHALL define shared contract types from a single source so JS/TS consumers and the Godot consumer agree on the same shapes.

### Requirement 8 — Dual-license community scaffolding

**User Story:** As a contributor, I want clear licensing and contribution rules, so that I
know how my code and creative content may be used and how to contribute them.

#### Acceptance Criteria

1. The repository SHALL contain `LICENSE-CODE` with the full PolyForm Noncommercial 1.0 text applying to source code.
2. The repository SHALL contain `LICENSE-ASSETS` with the full CC BY-NC 4.0 text applying to creative content (art, audio, writing, lore, SRD text, 3D models).
3. The repository SHALL contain a root `LICENSE` that explains the dual-license model and points to the two specific files.
4. The repository SHALL contain `CONTRIBUTING.md` that explains the distinct contribution lanes (canon/core, canon/lore, ttrpg, videogame, server) and states which license applies to each.
5. `CONTRIBUTING.md` SHALL state that all contributors MUST sign the CLA, which is checked automatically on pull requests.
6. The repository SHALL contain `.github/workflows/cla.yml` using the `cla-assistant/github-action`, triggered on `issue_comment` and `pull_request_target`, with placeholder `GITHUB_TOKEN` and `PERSONAL_ACCESS_TOKEN` environment variables.
7. The repository SHALL contain a professional `README.md` with a Licensing section explaining the dual model and a How to Contribute section linking to `CONTRIBUTING.md`.
8. WHERE a contribution incorporates externally-licensed rules (for example a future GUMSHOE module or an open SRD) THE `CONTRIBUTING.md` SHALL require that material to retain its own license and SHALL note that it cannot be relicensed as non-commercial.

### Requirement 9 — Copyright-safe original setting

**User Story:** As the project owner, I want the shipped setting to avoid infringing
existing works (Snowpiercer, Metro), so that the project is legally defensible.

#### Acceptance Criteria

1. The default pack SHALL use original names for the setting, places, factions, and characters, and SHALL NOT reuse protected names or specific expression from Snowpiercer/Le Transperceneige or Metro.
2. The setting SHALL rely only on the non-protectable genre premise (survivors aboard a perpetual vessel crossing a ruined world).
3. The contract and server SHALL remain setting-neutral so the concrete pack can be swapped if IP concerns arise.
4. Documentation SHALL record the IP-safety rationale so contributors understand the boundaries.

### Requirement 10 — Local developer startup & evaluation

**User Story:** As a developer, I want to run the canon server and validate packs locally,
so that I can evaluate correctness before any consumer integration.

#### Acceptance Criteria

1. WHEN a developer runs the documented start command THEN the Canon Server SHALL start locally over HTTP and report which packs loaded and which (if any) were rejected.
2. The project SHALL provide a command to validate a pack's core against the core schema and run the lore linter, printing errors (core) and warnings (lore) separately.
3. WHEN a developer runs the test command THEN the project SHALL execute Pact provider verification and schema/lore validation tests.
4. The project SHALL document the start, validate, and test commands in the server README.

### Requirement 11 — Behavior-driven, parameterized test execution (Gherkin)

**User Story:** As a maintainer of a system with diverse consumers, I want behavior specified
once in Gherkin and executable against different targets, so that the server logic, the TTRPG
surface, and the Godot surface are all verified against the same behavioral truth.

#### Acceptance Criteria

1. The project SHALL express canon-server behavior as Gherkin `.feature` files executed with Cucumber.js.
2. The Gherkin suite SHALL be parameterizable by a **target** with at least the values `logic`, `ttrpg`, and `godot`, where the target selects the binding layer (driver) used by the step definitions.
3. WHEN the suite runs with target `logic` THEN steps SHALL drive the server/validation logic directly in-process (for example via `fastify.inject()`), requiring no network and no consumer.
4. WHEN the suite runs with target `ttrpg` or `godot` THEN steps SHALL bind through that consumer's view of the API while reusing the same scenarios where applicable.
5. The suite SHALL use tags to scope scenarios to targets (for example `@all`, `@logic-only`, `@ttrpg`, `@godot`) and to trace scenarios to requirements (for example `@req-4`).
6. The project SHALL provide Cucumber profiles (for example `logic`, `ttrpg`, `godot`) that combine the appropriate tag filter and target world-parameter.
7. The Gherkin behavioral suite SHALL complement, not duplicate, the Pact interface contracts: Pact verifies wire-shape agreement; Gherkin verifies behavioral and content rules.
8. WHEN CI runs THEN at minimum the `logic` target SHALL execute and SHALL block the build on failure.

### Requirement validation note

Each requirement above is normative (EARS is the source of truth) and is validated by one or
more Gherkin scenarios tagged with the corresponding requirement id (for example `@req-4`).
Scenarios live as real, executable `.feature` files under a `features/` directory and are
cross-referenced from the design document, so prose requirements and executable behavior do
not drift.

### Requirement 12 — Canon exposes rules-readable hooks (canon ↔ rules boundary)

**User Story:** As a GM composing a deployment, I want the canon service to expose the
world-state values a rules module needs, so that a co-located rules module can map the world
into mechanics without the canon server knowing anything about any specific rules system.

#### Acceptance Criteria

1. The Canon Server SHALL remain **world-only**: it SHALL NOT contain or serve any
   rules-system mechanics (no dice math, no character models, no system-specific stats).
2. The Canon Server SHALL expose, through the existing read API, the world-state values a
   rules module may read as **hooks**: at minimum `speedRegime` + characteristics, resource
   `scarcity`, and `strata` ranks.
3. The hook data SHALL be setting-neutral and system-neutral (the same shape regardless of
   which rules module, if any, is deployed alongside it).
4. The detailed capability-based rules contract, the rules modules themselves (rules-light,
   Tri-Stat, GUMSHOE-compatible), their per-module licensing, and the GM composition model
   are specified in the separate `rules-engine` spec and are **out of scope** here.
