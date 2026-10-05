# Design — the Sacred Engine (Modular Rules Engine)

## Overview

The Rules Engine turns "what the world is" (canon) into "how play resolves." It is **modular**:
a GM selects one rules module per deployment, co-located with the canon service and a surface.

The central design choice is a **capability-based contract** rather than a universal mechanics
schema. Rules systems disagree about what a character or a check is, so a one-size schema would
fit none well (the inner-platform trap). Instead, the contract defines a few named
**capabilities** a surface actually needs; each module declares and implements the ones it
supports, and surfaces degrade gracefully when a capability is absent.

```
   ┌──────────────────────────────────────────────────────────┐
   │                     Rules Contract                         │
   │  capabilities: characterModel | resolveCheck |             │
   │                advancement(optional) | canonMapping        │
   └───────────────▲───────────────▲───────────────▲───────────┘
                   │ implements     │ implements    │ implements
            ┌──────┴─────┐   ┌──────┴─────┐  ┌───────┴────────┐
            │ rules-light│   │  tristat   │  │    gumshoe     │
            │ (CC BY-NC) │   │ (OGL 1.0a) │  │ (CC SRD)       │
            └──────┬─────┘   └──────┬─────┘  └───────┬────────┘
                   └──────── reads canon hooks ──────┘
                      (speedRegime, scarcity, strata)
                      via canon service HTTP/S API
```

## Capability contract

Capabilities are named, independently-declarable units. A module advertises a **capability
manifest**; the host (`RulesEngine`) and surfaces negotiate against it.

```ts
type CapabilityId = 'characterModel' | 'resolveCheck' | 'advancement' | 'canonMapping';

interface CapabilityManifest {
  moduleId: string;
  version: string;
  license: { id: string; provenance: 'original' | 'srd'; source?: string; notice?: string };
  capabilities: CapabilityId[];
}

interface RulesModule {
  manifest: CapabilityManifest;
  characterModel?: CharacterModelCapability;
  resolveCheck?: ResolveCapability;
  advancement?: AdvancementCapability;
  canonMapping?: CanonMappingCapability;
}

type CapabilityResult<T> =
  | { supported: true; value: T }
  | { supported: false; reason: string };   // graceful degradation (Req 1.4)
```

The host wraps every capability call so an unsupported capability yields
`{ supported: false, reason }` instead of throwing (Req 1.4, 4.3).

### characterModel (Req 2)

Declares the shape of a character sheet as **data**, so surfaces render generically:

```ts
interface FieldDef {
  id: string;
  label: string;
  type: 'number' | 'text' | 'enum' | 'pool' | 'list';
  constraints?: { min?: number; max?: number; options?: string[] };
  derived?: boolean;          // computed, not edited
}

interface CharacterModelCapability {
  fields(): FieldDef[];                             // the sheet schema
  computeDerived(c: Character): Character;          // fill derived values (Req 2.3)
  validate(c: Character): { ok: boolean; errors: string[] };
  serialize(c: Character): string;                  // store/reload (Req 2.4)
  deserialize(s: string): Character;
}
```

A surface renders `fields()` — three fields for rules-light, Body/Mind/Soul-plus for tristat,
investigative/general ability lists for gumshoe — without any hard-coded assumptions (Req 2.2).

### resolveCheck (Req 3)

```ts
interface ActionRequest {
  actor: Character;
  action: { id: string; label?: string };
  difficulty?: number | string;      // system interprets
  modifiers?: Record<string, number>;
  rng: Rng;                           // injected (Req 3.3 / Req 9.4)
}
interface Outcome {
  result: 'success' | 'failure' | 'partial' | string;   // graded allowed
  explanation: string;                                   // human-readable (Req 3.2)
  detail?: unknown;                                       // opaque system-specific (Req 3.4)
}
interface ResolveCapability { resolve(req: ActionRequest): Outcome; }
```

`Rng` is an injected interface (`next(): number` / `roll(sides, count)`) so tests seed it for
deterministic outcomes (Req 3.3, 9.4). Surfaces read only `result`/`explanation`; `detail`
stays opaque (Req 3.4).

### advancement (Req 4, optional)

```ts
interface AdvancementCapability {
  advance(c: Character, req: AdvanceRequest):
    | { ok: true; character: Character }
    | { ok: false; reason: string };
}
```

Absent on modules that don't support it; requesting it then returns the host's
`{ supported: false }` result.

### canonMapping (Req 5)

```ts
interface CanonHooks {                 // read from canon service (canon-server Req 12)
  speedRegime: { regime: string; characteristics: Record<string, unknown> };
  scarcity: Record<string, string>;    // resourceId -> scarcity
  strata: { id: string; rank: number }[];
}
interface CanonMappingCapability {
  fetchHooks(canonBaseUrl: string, packId: string): Promise<CanonHooks>;
  applyToDifficulty(base: number | string, hooks: CanonHooks): number | string;
  applyToActor(c: Character, hooks: CanonHooks): Character;
  defaults(): CanonHooks;              // used if hooks unavailable (Req 5.4)
}
```

Each module maps the world its own way (Req 5.3); the canon server stays system-agnostic. If
hooks are unreachable, the module uses `defaults()` and still resolves (Req 5.4).

## Host: RulesEngine & deployment composition (Req 6)

```ts
class RulesEngine {
  static load(moduleId: string): RulesEngine;   // throws -> deployment fails loud (Req 6.3)
  activeModule(): CapabilityManifest;           // id/version/license/capabilities (Req 6.4)
  capability<T>(id: CapabilityId): CapabilityResult<T>;
}
```

- The GM sets the active module by **configuration** (env/config file: `SE_RULES_MODULE=rules-light`), not code (Req 6.1).
- Exactly one module is active per deployment, co-located with canon + surface (Req 6.2).
- An unknown/failed module id fails startup with a diagnostic (Req 6.3).
- The active manifest (incl. license id) is exposed for surface display (Req 6.4, 10.4).

### Where it runs

Co-located with the canon service and surface in the GM's deployment. The rules module reads
canon hooks over the **same local HTTP/S canon API** the surfaces use. The rules engine itself
can be an in-process library the surface links, or a thin local service — both satisfy the
contract; the reference implementation ships as a library plus an optional local HTTP wrapper.

## Technology stack

- **TypeScript** host + contract (shared with canon server and JS/TS surfaces).
- **Cucumber.js** behavior tests, parameterized by target (`logic`/`ttrpg`/`godot`),
  consistent with the canon-server spec; injected `Rng` for determinism.
- Unit tests for each capability of each module.
- Godot consumes resolution results over the local HTTP wrapper (or a generated type stub);
  native Godot binding deferred (test/startup evaluation scope, per canon-server decision).

## Repository layout (rules-engine portion)

```
rules-engine/
├─ contract/
│  ├─ src/                      # capability interfaces, RulesEngine host, Rng
│  └─ LICENSE  -> PolyForm NC (code) ; contract docs CC BY-NC
├─ modules/
│  ├─ rules-light/              # original (CC BY-NC / PolyForm NC)
│  │  ├─ src/  manifest.json  LICENSE
│  ├─ tristat/                  # OGL 1.0a
│  │  ├─ src/  manifest.json  LICENSE-OGL  OGL-SECTION-15.txt
│  └─ gumshoe/                  # CC SRD
│     ├─ src/  manifest.json  LICENSE-CC  ATTRIBUTION.md
└─ features/rules-engine/       # executable .feature files (@req-N)
```

(GURPS: no folder — deferred; recorded in `modules/README.md` as "no open SRD, not bundled".)

## Testing strategy

Consistent three-layer model with the canon-server spec:

1. **Cucumber.js (behavior), parameterized** — `logic` drives the host + module directly;
   `ttrpg`/`godot` bind through the surface's view. Tags `@all` / `@logic-only` / `@ttrpg` /
   `@godot` and `@req-N`. Injected `Rng` for deterministic resolution (Req 9.4).
2. **Unit tests** — each capability of each module (character validation, derived computation,
   resolution math, canon mapping, advancement).
3. **Canon integration** — `canonMapping` tested against a canon fixture / the local canon API,
   including the hooks-unavailable `defaults()` path (Req 5.4).

### Feature file layout

```
features/rules-engine/
├─ capability-negotiation.feature  # @req-1  declare + graceful degrade
├─ character-model.feature         # @req-2  declared fields, derived, (de)serialize
├─ resolution.feature              # @req-3  injected rng, outcome shape, opaque detail
├─ advancement.feature             # @req-4  optional capability behavior
├─ canon-mapping.feature           # @req-5  hooks -> effects, defaults path
└─ module-selection.feature        # @req-6  config select, fail-loud, active manifest
```

## Licensing design (Req 8, 10)

- **contract/** and **rules-light/** — project's own work: PolyForm NC (code) + CC BY-NC (text).
- **tristat/** — Tri-Stat dX content under **OGL 1.0a**; ships `LICENSE-OGL` + a correct
  **Section 15** notice listing the Open Game Content used; not relicensed NC.
- **gumshoe/** — GUMSHOE SRD under its **Creative Commons** license; ships `LICENSE-CC` +
  `ATTRIBUTION.md`; described as "GUMSHOE-compatible"; no Pelgrane trademark/logos.
- Every module's `manifest.json` carries `license.{id, provenance, source, notice}` (Req 10.1),
  surfaced via `activeModule()` (Req 10.4). The root `CONTRIBUTING.md` states SRD-derived
  modules keep their own license and cannot be relicensed NC (Req 10.3).

## Error handling

- **Unknown/failed module** → deployment fails to start with a clear message (Req 6.3).
- **Unsupported capability** → `{ supported: false, reason }`, never a throw (Req 1.4).
- **Canon hooks unreachable** → module uses `defaults()`; resolution still proceeds (Req 5.4).
- **Invalid character** → `validate()` returns structured errors; surface shows them.

## Design decisions & rationale

- **Capabilities over a universal schema:** the only honest way to let GUMSHOE, Tri-Stat, and a
  rules-light system coexist without a leaky abstraction; surfaces target capabilities and
  degrade gracefully.
- **Injected RNG:** makes resolution deterministic and testable, and keeps randomness policy
  out of modules.
- **Canon mapping owned by the module:** preserves "canon is world-only"; each system
  interprets the world its own way; the canon server never learns any rules.
- **Per-module licensing with explicit manifests:** keeps a mixed-license repo compliant and
  transparent; the only project-owned, NC-licensed rules content is the original rules-light
  module and the contract.
- **GURPS deferred, recorded:** avoids an incorrect assumption that a free-to-read ruleset is a
  free/open SRD; revisit only with SJGames permission or a clean-room compatible approach.
