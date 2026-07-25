# Figma MCP → spec/ mapping

Which Figma MCP tool populates which spec artifact, and how far each claim is verified.

**Provenance tags** (same discipline as the `ds-fetch` catalog):

- `[env]` — confirmed against a **real file**, run 2026-07-19 (fileKey `of6sR4JjlHzTFp5m0op6N2`,
  node `177:61`). Return shape observed, not assumed.
- `[figma-doc]` — return shape from the tool's own description, **not** yet run. Inspect on first
  use and promote.

Server prefix is `mcp__b78f5274-7ace-4f4b-a579-25b35966e81c__` (a session-specific id — confirm
yours; do not hardcode it into committed code). Tools referred to below by short name.

**The standing rule:** tool names are solid, return shapes are not. The first real run already
overturned two assumptions (Code Connect is plan-gated; a raster node yields nothing) — see the
findings box below. Probe, then fix this file.

## First-run findings (2026-07-19) — read these before trusting the happy path

1. **A raster / flattened node yields nothing extractable.** The test node was a single 4096×2304
   image ("Ticket"). `get_variable_defs` → `{}`, `get_metadata` → one `<rounded-rectangle>`,
   `get_design_context` → a lone `<img>` with an asset URL. No tokens, no structure, no variants.
   **The extractor must detect this and stop** (Step 1 of the SKILL). Not every Figma URL points at
   native design data; imported/flattened art is common and produces a confidently empty spec if you
   do not guard for it.

2. **Code Connect is plan-gated.** `get_code_connect_map` returned: *"You need a Dev or Full seat on
   an Organization or Enterprise plan."* A Full seat on a **team/student** plan is not enough. So the
   "Code Connect is the highest-value binding" claim holds **only when the plan allows it** — for
   many users these tools error. Treat Code Connect as a bonus with a fallback, never a requirement.
   `list_file_components_for_code_connect` and `get_context_for_code_connect` are almost certainly
   gated the same way (unconfirmed — same subsystem).

3. **`get_variable_defs` empty ≠ "design has no tokens."** Confirmed the per-node warning with a real
   `{}`. This node simply binds no variables. Query native component nodes, and union across them.

---

## The mandatory gate

`get_design_context` **requires** the `figma-design-to-code` skill loaded first, and its name passed
in `skillNames` (prefixed `resource:` when loaded via the MCP resource). This is documented as a
hard prerequisite, not advice. `[env]` — stated on the tool and in the skill index.

Load it once at the start of extraction; it governs every `get_design_context` call.

---

## Tool → artifact table

| Figma MCP tool | Params | Populates | Return shape |
|---|---|---|---|
| `get_metadata` | `fileKey`, optional `nodeId` | node selection, `layout.md` structure | `[env]` "Currently selected nodes:" preamble, then XML elements (`<rounded-rectangle id name x y width height />`), then a "you MUST call get_design_context" nudge. Flat for a leaf node. |
| `get_variable_defs` | `nodeId`, `fileKey` | **`tokens.json`** | `[env]` flat JSON map `{"path/name": value}`. **Returned `{}` for a node with no bound variables.** |
| `get_design_context` | `nodeId`, `fileKey` | component structure + hints | `[env]` a TS module string (React component, Tailwind classes, asset URLs as `const`s, `data-node-id` attrs) + instruction blocks (convert-to-target-stack, node-ids, 7-day asset URLs) + an inline screenshot. |
| `get_code_connect_map` | `nodeId`, `fileKey` | `behavior_contract` bindings | `[env]` **plan-gated** — errors without a Dev/Full seat on Org/Enterprise. When allowed: `{nodeId: {codeConnectSrc, codeConnectName}}` `[figma-doc]`. |
| `list_file_components_for_code_connect` | `fileKey` | component inventory + deps | `[figma-doc]` flat per-component graph with variants. Likely plan-gated (Code Connect subsystem). |
| `get_context_for_code_connect` | `nodeId`, `fileKey` | one component's variant/prop tree | `[figma-doc]` properties, variant options, descendant tree. Likely plan-gated. |
| `get_screenshot` | `nodeId`, `fileKey`, `maxDimension` | `verify.yaml` anchors | `[env]` JSON `{image_url, width, height, format, original_width, original_height}` — a **short-lived URL (~7-day), not inline** unless `enableBase64Response:true`. |
| `get_motion_context` | `nodeId`, `fileKey` | (defer to `figma-implement-motion`) | `[figma-doc]` keyframes, easing, CSS/motion snippets |
| `get_libraries` | `fileKey` | token/component completeness | `[figma-doc]` subscribed + available libraries with keys |
| `whoami` | — | auth/permission debug | `[env]` `{handle, email, plans:[{name, seat, tier, key}]}` |

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
