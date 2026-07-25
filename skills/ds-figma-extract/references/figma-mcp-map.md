# Figma MCP → spec/ mapping

Which Figma MCP tool populates which spec artifact, and how far each claim is verified.

**Provenance tags** (same discipline as the `ds-fetch` catalog):

- `[env]` — the tool **name and parameters** are confirmed present in the connected Figma MCP
  (they are in the loaded tool list, 2026-07-19).
- `[figma-doc]` — the **return shape** described here comes from the tool's own description, **not**
  from a run against a real file. Inspect the actual return on first use and correct this file.

Server prefix is `mcp__b78f5274-7ace-4f4b-a579-25b35966e81c__` (a session-specific id — confirm
yours; do not hardcode it into committed code). Tools referred to below by short name.

**The standing rule:** tool names are solid, return shapes are not. The whole `ds-fetch` saga was
about exactly this gap. Do not assume a field exists because it is drawn below — probe, then fix
this file.

---

## The mandatory gate

`get_design_context` **requires** the `figma-design-to-code` skill loaded first, and its name passed
in `skillNames` (prefixed `resource:` when loaded via the MCP resource). This is documented as a
hard prerequisite, not advice. `[env]` — stated on the tool and in the skill index.

Load it once at the start of extraction; it governs every `get_design_context` call.

---

## Tool → artifact table

| Figma MCP tool | Params `[env]` | Populates | Return shape |
|---|---|---|---|
| `get_metadata` | `fileKey`, optional `nodeId` | node selection, `layout.md` structure | XML of ids/types/names/positions/sizes `[figma-doc]` |
| `get_variable_defs` | `nodeId`, `fileKey` | **`tokens.json`** | flat map `{"path/name": value}`, e.g. `{"icon/default/secondary": "#949494"}` `[figma-doc]` |
| `get_design_context` | `nodeId`, `fileKey` | component structure + hints | React+Tailwind reference code, screenshot, hint bundle `[figma-doc]` |
| `get_code_connect_map` | `nodeId`, `fileKey` | `behavior_contract` bindings | `{nodeId: {codeConnectSrc, codeConnectName}}` `[figma-doc]` |
| `list_file_components_for_code_connect` | `fileKey` | component inventory + deps | flat per-component graph with variants `[figma-doc]` |
| `get_context_for_code_connect` | `nodeId`, `fileKey` | one component's variant/prop tree | properties, variant options, descendant tree `[figma-doc]` |
| `get_screenshot` | `nodeId`, `fileKey`, `maxDimension` | `verify.yaml` anchors | PNG via short-lived URL (**~7-day expiry**) `[figma-doc]` |
| `get_motion_context` | `nodeId`, `fileKey` | (defer to `figma-implement-motion`) | keyframes, easing, CSS/motion snippets `[figma-doc]` |
| `get_libraries` | `fileKey` | token/component completeness | subscribed + available libraries with keys `[figma-doc]` |
| `whoami` | — | auth/permission debug | handle, email, plans `[figma-doc]` |

---

## Tokens — the per-node constraint that will bite

`get_variable_defs` **requires a concrete `nodeId`** and returns the variables **that node uses** —
not the file's whole variable collection. `[env]` on the requirement (the tool description states
"requires a concrete node target"); `[figma-doc]` on the "used-by-node" scoping — **verify it**, it
is the difference between a complete token set and a partial one.

Consequences:

- **Union across nodes.** Query `get_variable_defs` on each key component/frame and merge. One node
  yields one node's palette.
- **An empty tier is ambiguous.** A missing `semantic.*` layer may mean the design has none, or
  that you queried a node that binds only primitives. Do not conclude "no semantic tokens" from a
  single node.
- **`get_libraries` for reach.** If variables live in a shared library, confirm it is subscribed;
  library keys scope a `search_design_system` if you need to hunt specific tokens.

### Tiering Figma variables

Figma variable **names frequently already encode DTCG tiers** via `/` grouping:

```
color/text/primary      → semantic.text.primary
color/blue/600          → primitive.blue.600
space/2                 → primitive.space.2
component/button/bg      → component.button.bg
```

Preserve this. The grouping is **designer intent**, not something to re-derive — mapping it into
`tokens.json`'s three tiers is usually mechanical. Only invent a name (and flag `inferred: true`)
where a raw value appears with no variable behind it. Follow `ds-spec-extract/references/tokens.md`
for the tier rules and the `@theme` emit.

---

## Components — Code Connect is a binding, not a hint

`get_code_connect_map` returns, per node, the **codebase component it is already mapped to**
(`codeConnectSrc`, `codeConnectName`). `[figma-doc]`

When a mapping exists, it is authoritative: the design team has declared *this Figma component is
that code component*. Record it in the spec's `behavior_contract` so `ds-nextjs-codegen` (or any
target) **reuses the real component** instead of regenerating an equivalent. This is the single
highest-value signal Figma gives that a code export cannot.

`list_file_components_for_code_connect` gives the **whole-file** component graph with variant
options and dependencies — use it to plan extraction order (leaf components before composites) and
to see which components already have Code Connect coverage.

`get_context_for_code_connect` drills one component: its property definitions, exhaustive variant
options, defaults. This maps almost directly to spec `props` + `variants`.

---

## Variants → states: the careful part

Figma **variant properties** (e.g. `State: Default|Hover|Pressed|Disabled`, `Size: sm|md|lg`) come
back in the component context. Map them:

- **Presentational axes** (`Size`, `Variant`, `Emphasis`) → spec `variants`. Straightforward.
- **Interaction/availability axes** (`State=Hover/Pressed`, `Disabled=true`) → candidate
  `states.machine` regions per `ds-spec-extract/references/statechart-subset.md`: `interaction`,
  `availability`.

**But a variant is a snapshot, not a transition.** `State=Hover` says hover has a look; it does not
say `idle --POINTER_ENTER--> hover`. The transition graph is conventional (you may fill the standard
interaction region) but anything non-obvious — async states, validation, multi-step flows — is
**not in the variants**. Prototype **reactions** may encode some transitions; where they are not
retrievable or not present, the transition is `TBD-user`.

Never emit a `states.machine` richer than the design supports. Gap it in `FIGMA-GAPS.md`.

---

## What Figma structurally cannot give you

Record these as `TBD-user` every time — they are not extraction failures, they are absent by nature:

- **Data ownership / fetching.** A design has no server boundary. This comes from the target repo
  or the user, never invented in the spec.
- **Loading / error / empty states** unless the designer drew them as explicit frames or variants.
- **Callback semantics** — what fires when. Variants show end states, not event wiring.
- **Copy that is Lorem/placeholder.** Flag it; do not ship placeholder strings as real content.

---

## Assets

`get_design_context` returns icons/images as **exported assets with expiring URLs (~7 days)**
`[figma-doc]`. The `figma-design-to-code` skill's asset-fidelity rules (G5) apply: render every
asset from its export, preserve outer-box and inner-leaf geometry, download bytes for anything
committed. For the spec stage, record asset references and geometry in the component spec; the
codegen stage does the actual download + placement under those rules.
