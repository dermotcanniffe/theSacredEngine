# Design — the Sacred Engine (Canon Server)

## Overview

The Canon Server is an HTTP/S service that loads **canon packs** and serves them to multiple
consumers (a TTRPG web app, a Foundry VTT module, and a Godot game). A pack has two layers:

- **Core** — the structural "physics" of the world (vessel/sections, strata, factions,
  resources, speed regime). Validated **strictly** against a JSON Schema. Invalid core is
  rejected and never served.
- **Lore** — the optional narrative layer (history, places, events, flavor). Validated by an
  **advisory** linter that emits warnings and a coherence score but never blocks loading.

The setting is **pluggable**: the server and consumers depend only on a setting-neutral
*contract*, while the concrete setting lives inside a swappable pack. The default pack is
`dustline` (a slow desert convoy, `halting` speed regime).

Three enforcement mechanisms, kept deliberately separate:

| Mechanism | Governs | Verdict | Tooling |
|---|---|---|---|
| **Pact** | HTTP API shape (consumer ↔ server) | binary pass/fail | `@pact-foundation/pact` |
| **Core JSON Schema** | structural validity of core | binary pass/fail (reject) | `ajv` |
| **Lore lint** | narrative coherence of lore | advisory warnings + score | custom validator |

Behavior is specified once in **Gherkin** and executed against parameterized **targets**
(`logic` / `ttrpg` / `godot`) with **Cucumber.js**.

## Roles & deployment model

The word "consumer" is a **Pact/contract-testing term for a software component**, not a
person. Three roles are kept distinct:

- **GM / operator** — deploys and composes a *campaign instance*: picks a canon pack, a rules
  module, and a surface, and runs them **co-located** as one deployment. An operator, not just
  an end-user.
- **Player** — an end-user at the table, interacting only through the surface; players never
  call the canon API directly.
- **Pact consumer** — the web app, Foundry module, or Godot client *software* that calls the
  canon API.

```
   GM composes a deployment (one campaign):
   ┌───────────────────────────────────────────────┐
   │  Canon service   (serves the chosen pack)      │
   │  Rules module    (chosen system — see          │
   │                   rules-engine spec)           │
   │  Game surface    (web app / Foundry / Godot)   │
   └───────────────────────────────────────────────┘
         ▲                                   ▲
         │ GM selects via config             │ players connect (end-users)
```

The **deployment / campaign instance** = `{ canon pack + rules module + surface }`,
co-located and selected by the GM. The Canon Server is one service in that bundle. It stays
**world-only** — it contains no rules-system mechanics — but it exposes **rules-readable
hooks** (speed-regime pressure, resource scarcity, strata ranks) that a co-located rules
module reads via the same local HTTP/S contract. The modular rules engine itself is designed
in the separate `rules-engine` spec.

## Architecture

```
          core files ─┐
          lore files ─┤          ┌───────────────────────────────────┐
   pack manifest ─────┼─ load ──▶ │             Canon Server           │
                      │          │  ┌─────────────┐  ┌──────────────┐ │
                      │          │  │ Core loader  │  │ Lore loader  │ │
                      │          │  │ + ajv (hard) │  │ + lint (soft)│ │
                      │          │  └──────┬───────┘  └──────┬───────┘ │
                      │          │         ▼                 ▼         │
                      │          │     Pack Registry (valid packs)     │
                      │          │         │                           │
                      │          │     Fastify HTTP/S  /v1/...          │
                      │          └─────────────────┬──────────────────┘
                      │                            │ HTTP/S (JSON)
                      │        ┌───────────────────┼───────────────────┐
                      ▼        ▼                   ▼                   ▼
                 (contracts)  TTRPG web app   Foundry VTT module   Godot game
                   Pact ◀──── each consumer publishes a pact; server verified against all
```

### Request flow

1. On startup the **Pack Loader** scans the packs directory.
2. For each pack: **Core Loader** validates `core/*` against `core.schema.json` via `ajv`.
   - Fail → pack rejected, error recorded, pack excluded from the registry.
   - Pass → core added to the registry.
3. If `lore/*` exists: **Lore Loader** parses it and runs the **Lore Linter** against the
   already-validated core (so it can resolve references). Warnings + score are attached; the
   pack is served regardless.
4. The **Pack Registry** holds only servable packs. If no pack is servable, startup fails
   with a non-zero exit and a diagnostic.
5. **Fastify** exposes versioned read routes over the registry.

## Technology stack

- **Runtime/language:** Node.js + **TypeScript** (shared types with JS/TS consumers).
- **HTTP framework:** **Fastify** (fast, schema-friendly, `inject()` for in-process tests).
- **Core validation:** **ajv** (JSON Schema 2020-12).
- **Lore validation:** custom TypeScript module (pure functions, unit-testable).
- **Interface contracts:** **Pact** (`@pact-foundation/pact`) for JS consumers; a committed
  JSON pact for Godot (no mature native binding yet — see Testing Strategy).
- **Behavioral tests:** **Cucumber.js** (`@cucumber/cucumber`) with parameterized targets.
- **Pack content:** JSON (and/or Markdown front-matter for lore prose) under `canon/packs/`.

Godot consumes plain HTTP/JSON, so the server language is irrelevant to it. TypeScript is
chosen because the Foundry module and web app are already JS/TS and can share generated types.

## Data model & pack format

A pack is a self-contained directory identified by a stable id:

```
canon/packs/dustline/
├─ pack.json              # manifest: id, name, contractVersion, speedRegime
├─ core/
│  ├─ vessel.json         # ordered sections
│  ├─ strata.json         # social strata (ordered rank)
│  ├─ factions.json
│  └─ resources.json
└─ lore/                  # OPTIONAL
   ├─ history.json
   ├─ places.json
   └─ events.json
```

### `pack.json` (manifest)

```jsonc
{
  "id": "dustline",
  "name": "Dustline",
  "contractVersion": "1.0.0",
  "speedRegime": "halting",
  "summary": "A slow convoy crossing an endless salt-and-dust waste."
}
```

### Core shapes (summarized; full schema in `canon/schema/core.schema.json`)

- **vessel**: `{ id, name, sections: [{ id, name, order, role }] }` — `order` is a unique
  integer; sections are the ordered cars/segments.
- **strata**: `[{ id, name, rank, description? }]` — `rank` unique integer, lower = higher status (or documented convention).
- **factions**: `[{ id, name, stratumId?, disposition? }]` — `stratumId` must reference a stratum.
- **resources**: `[{ id, name, scarcity }]` — `scarcity` in an enum.
- **speedRegime**: one of `halting | crawling | running`.
- **regime characteristics** (per Requirement 3): `{ egress: boolean, resupply: "stops"|"closed"|"mixed", threatSource: "external"|"internal"|"both", spatialModel: "regions"|"sections" }`.

### Speed-regime defaults (guidance, pack may override within schema bounds)

| Regime | egress | resupply | threatSource | spatialModel | Play shape |
|---|---|---|---|---|---|
| `halting` | true | stops | external | regions | hub-and-spoke exploration/survival (Metro-like) |
| `crawling` | occasional | mixed | both | mixed | timed excursions, rising internal pressure |
| `running` | false | closed | internal | sections | sealed social intrigue (Snowpiercer-like) |

### Lore shapes (advisory)

Lore entries are narrative objects that *reference* core ids (e.g. an event referencing a
`factionId` or `sectionId`). The linter checks those references resolve against the pack's
core. Unresolved references → warnings, not errors.

## HTTP API (v1)

All routes are prefixed `/v1`. Responses are JSON. Errors use a consistent body
`{ "error": { "code": string, "message": string } }`.

| Method | Route | Purpose |
|---|---|---|
| GET | `/v1` | contract version(s) + list of available pack ids |
| GET | `/v1/packs` | list packs with manifest summaries |
| GET | `/v1/packs/:id` | full pack (core; lore included unless `?lore=false`) |
| GET | `/v1/packs/:id/core` | core only |
| GET | `/v1/packs/:id/lore` | lore + warnings/score (404 if pack has no lore) |
| GET | `/v1/packs/:id/speed-regime` | speed regime + characteristics |
| GET | `/v1/health` | liveness + loaded/rejected pack counts |

- `GET /v1/packs/:id?lore=false` satisfies "core-only" opt-out (Req 4.6, 7.2).
- Lore responses carry `{ lore: {...}, validation: { score, warnings: [...] } }` (Req 4.4).
- Versioning: incompatible changes introduce `/v2` while `/v1` continues (Req 7.3).

### Canon → rules hooks (Req 12)

The canon API exposes world-state values a co-located rules module may read, but the server
has no knowledge of any rules system. The "hooks" are simply existing core fields with a
stable, system-neutral shape:

- `speedRegime` + characteristics (from `/v1/packs/:id/speed-regime`) — a module may derive a
  "pressure" modifier from the regime.
- resource `scarcity` (from core) — a module may map scarcity to resolution difficulty.
- `strata` ranks (from core) — a module may map social position to mechanical advantage.

How a module *interprets* these is defined in the `rules-engine` spec; the Canon Server only
guarantees the data is present and setting/system-neutral.

### Contract types (single source of truth)

Core shapes are defined as JSON Schema in `canon/schema/`. TypeScript types for JS/TS
consumers are **generated** from those schemas (e.g. `json-schema-to-typescript`), so the
server, the web app, and the Foundry module cannot disagree on shape. Godot reads the same
JSON at runtime and (optionally) a generated GDScript type stub.

## Validation design

### Core (strict) — `ajv`

- One compiled schema per core file + a top-level pack schema.
- `contractVersion` is required and checked against supported versions.
- Failure → `PackRejected` with the ajv error list; pack excluded; recorded for `/v1/health`
  and startup diagnostics.

### Lore (advisory) — custom linter

Pure functions returning `{ score: number, warnings: Warning[] }` where
`Warning = { code, message, ref? }`. Initial checks:

- **Unresolved reference** — lore points to a `factionId`/`sectionId`/`stratumId` not in core.
- **Orphan** — a core faction/section never referenced by any lore (informational).
- **Field conventions** — missing recommended narrative fields (e.g. empty description).

Score is a simple normalized ratio (resolved refs / total refs, adjusted by convention hits).
The linter never throws on advisory issues; it only reports.

## Testing strategy

Three complementary layers (no overlap in responsibility):

1. **Pact (interface):** each consumer (`web`, `foundry`, `godot`) declares expected
   request/response shapes; the server runs **provider verification** in CI. Contracts are
   committed under `contracts/`. A hosted broker is deliberately deferred; a documented seam
   exists to adopt PactFlow later (Req 6.4).
2. **Cucumber.js (behavior), parameterized by target (Req 11):** behavior written once in
   `.feature` files; a **World** object constructs a **driver** per target:
   - `logic` — drives the Fastify app via `inject()` and calls validators directly. Default,
     runs in CI, blocks the build.
   - `ttrpg` — binds through the TTRPG client adapter's view of the API.
   - `godot` — binds through the Godot HTTP access pattern (test/startup evaluation only for now).
   Tags scope scenarios (`@all`, `@logic-only`, `@ttrpg`, `@godot`) and trace to requirements (`@req-N`).
3. **Unit tests:** validators, loaders, and pure functions (lore scoring, reference
   resolution) tested directly.

### Cucumber profiles

```js
// cucumber.js
const common =
  'features/**/*.feature --require-module ts-node/register --require src/**/*.steps.ts';
module.exports = {
  logic: `${common} --tags "not @ttrpg and not @godot" --world-parameters '{"target":"logic"}'`,
  ttrpg: `${common} --tags "@all or @ttrpg" --world-parameters '{"target":"ttrpg"}'`,
  godot: `${common} --tags "@all or @godot and not @logic-only" --world-parameters '{"target":"godot"}'`,
};
```

Run: `cucumber-js -p logic` (also `-p ttrpg`, `-p godot`).

### World / driver abstraction

```ts
interface CanonDriver {
  listPacks(): Promise<Response>;
  getPack(id: string, opts?: { lore?: boolean }): Promise<Response>;
  getLore(id: string): Promise<Response>;
  getSpeedRegime(id: string): Promise<Response>;
}
// logic -> InjectDriver (fastify.inject)
// ttrpg -> TtrpgClientDriver
// godot -> GodotHttpDriver
```

Steps call `this.driver.getPack(...)`; only the driver changes per target, so one scenario
proves behavior across surfaces.

### Feature file layout

```
features/canon-server/
├─ http-api.feature          # @req-1  routing, versioning, 404s
├─ core-validation.feature   # @req-2  strict schema, rejection, fail-loud startup
├─ speed-regime.feature      # @req-3  regime enum + characteristics exposure
├─ lore-advisory.feature     # @req-4  advisory warnings, non-blocking, core-only opt-out
└─ pack-pluggability.feature # @req-5  multiple packs, select by id, swap without code
```

## Repository layout

```
theSacredEngine/
├─ canon-server/                 # provider (TS + Fastify)
│  ├─ src/
│  │  ├─ app.ts                  # buildApp() -> Fastify instance (injectable)
│  │  ├─ server.ts               # start()/TLS/exit codes
│  │  ├─ loader/                 # pack discovery, core loader, lore loader
│  │  ├─ validation/             # ajv wiring + lore linter
│  │  ├─ routes/                 # /v1 route handlers
│  │  └─ steps/                  # cucumber step defs + drivers + world
│  ├─ cucumber.js
│  ├─ package.json
│  └─ README.md                  # start / validate / test commands (Req 10)
├─ canon/
│  ├─ CONTRACT.md                # setting-neutral contract (core + lore)
│  ├─ schema/
│  │  ├─ core.schema.json        # STRICT
│  │  └─ lore.schema.json        # SOFT (advisory shape)
│  ├─ validators/lore-lint/      # (shared lint rules if extracted)
│  └─ packs/dustline/            # default pack (core + lore)
├─ features/canon-server/        # executable .feature files
├─ contracts/                    # web.pact.json, foundry.pact.json, godot.pact.json
├─ ttrpg/                        # DESIGN.md, web/, foundry/, srd/ (future specs)
├─ videogame/                    # Godot 4 consumer (future spec)
├─ shared-assets/
├─ LICENSE / LICENSE-CODE / LICENSE-ASSETS
├─ CONTRIBUTING.md
├─ README.md
└─ .github/workflows/            # cla.yml, ci.yml (schema+cucumber logic+pact verify)
```

## Error handling

- **Pack load errors (core):** caught per pack, do not crash the server unless *no* pack is
  servable; surfaced via logs, `/v1/health`, and non-zero startup exit when total.
- **Lore errors:** parse failures degrade to "lore unavailable + warning"; advisory warnings
  never fail a request.
- **HTTP errors:** consistent `{ error: { code, message } }`; 404 for unknown pack/lore,
  400 for bad query params, 500 only for unexpected faults.
- **TLS:** HTTPS when configured; HTTP permitted for local dev only (documented).

## Security & licensing considerations

- Read-only public API in this phase; no auth required for serving canon. Any future write
  path (community pack submission) is out of scope and would need auth + review.
- Treat pack files as trusted repo content (reviewed via PR + CI validation), not arbitrary
  user input, in this phase.
- **Dual license:** server code under PolyForm Noncommercial 1.0 (`LICENSE-CODE`); canon
  content, lore, SRD text, and assets under CC BY-NC 4.0 (`LICENSE-ASSETS`); root `LICENSE`
  explains the split. Externally-licensed rules (e.g. a future GUMSHOE module) keep their own
  license and are not relicensed as NC.

## IP-safety rationale

- The contract and server are **setting-neutral**; no protected names/expression from
  Snowpiercer/Le Transperceneige or Metro appear in code or schema.
- Only the non-protectable genre premise is used (survivors on a perpetual vessel crossing a
  ruined world). The concrete flavor lives in a swappable pack (`dustline`) that can be
  replaced wholesale if challenged, with no change to server, schema, contracts, or consumers.
- This rationale is recorded here and summarized in `README.md`/`CONTRIBUTING.md`.

## Design decisions & rationale

- **Two layers, three mechanisms:** keeps "world must be structurally sound" (hard) separate
  from "story should be coherent" (soft) and from "the wire must match" (Pact). Each failure
  mode is handled by the tool suited to it.
- **Static files now, server-ready contract:** packs are files validated in CI and served by
  the running server; the contract is written as resource definitions so a future hosted/live
  canon service and Pact Broker slot in without a rewrite.
- **Parameterized Gherkin:** one behavioral truth proven against logic/ttrpg/godot avoids
  per-surface test drift while matching the "test/startup evaluation is enough for now" scope
  for Godot.
- **TypeScript + Fastify:** shared types with JS/TS consumers, first-class Pact + ajv tooling,
  `inject()` for fast in-process behavior tests.
```
