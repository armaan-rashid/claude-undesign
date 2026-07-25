# SKILL-FEEDBACK — ds-figma-extract, first real MCP run

Run 2026-07-19 against a live Figma file to promote the Figma MCP return shapes from `[figma-doc]`
to `[env]`. File `of6sR4JjlHzTFp5m0op6N2` ("Ticket"), node `177:61`. Auth: `whoami` = Full seat,
tier `student`, team plan.

## What the node turned out to be

A single **4096×2304 flattened raster** — the whole "Ticket" mockup as one PNG, not native Figma
layers. This was accidental (the user grabbed whatever was selected) but turned out to be the most
useful possible first case, because it exercised the empty/degenerate path the skill had no guard
for.

## Confirmed return shapes (now `[env]` in figma-mcp-map.md)

- `get_metadata` → "Currently selected nodes:" preamble, then XML elements, then a hard nudge to
  call `get_design_context`. Flat single element for a leaf node.
- `get_variable_defs` → `{}`. Real confirmation of the per-node under-count risk.
- `get_design_context` → a TS React module string + instruction blocks (convert-to-target-stack,
  node-ids as data attrs, 7-day asset URLs) + an inline screenshot. Tailwind classes in the code.
- `get_screenshot` → JSON `{image_url, width, height, format, original_*}`; URL is short-lived
  (~7 day), not inline unless `enableBase64Response:true`.
- `get_code_connect_map` → **plan-gated error**: "You need a Dev or Full seat on an Organization or
  Enterprise plan." A team/student Full seat is insufficient.

## Two assumptions overturned → two skill changes

1. **Raster/flattened nodes yield nothing** — added a Step 1 guardrail: metadata-leaf +
   `get_variable_defs {}` + design-context-lone-`<img>` ⇒ stop, ask for native layers. Without it
   the skill would emit a confidently empty spec.

2. **Code Connect is not universally available** — it is plan-gated. The skill had treated a Code
   Connect mapping as the central, highest-value binding. Demoted to a bonus-with-fallback: try it,
   fall back to design-context hints + target-repo components on a plan error.

## Round 2 — native components probed (same session)

Walked the file tree (`get_metadata` with no nodeId → pages; then the page) and found real main
components: `16:27` "Component 1" and `5:16` "Stub Counter", both `<symbol>` nodes with instances.

**Node types are the finding:** `<symbol>` = main component, `<instance>` = instance, `<frame>` /
`<canvas>` = containers. Scan for `symbol` to locate components.

### The big one: no design system, and that is normal

- `get_variable_defs` → `{}` on **all three** nodes probed.
- `get_libraries` → `libraries_added_to_file: []`.

The skill's premise — "Figma variables *are* tokens, so this maps cleanly" — only holds for
design-system files. A file with no variables is the common case outside mature design teams.
**Added a fallback path** (Step 2 case B): harvest raw values from the design-context output,
cluster/tier/name them per `tokens.md`, flag every one `inferred: true`, and say plainly in
`FIGMA-GAPS.md` that the tiering is a proposal, not extracted intent.

### The reference output is absolute-positioned, arbitrary-value Tailwind

Every layer `absolute` with percentage insets; every value arbitrary (`h-[54px]`, `text-[36px]`,
`font-['Philosopher:Regular']`, `text-black`); copy hardcoded; a `className` override prop.

Two corrections to `ds-nextjs-codegen`:

1. **The arbitrary-values rule was wrong as written.** It said "an arbitrary-value class means a
   token was skipped — treat it as a defect." But that is Figma's *default reference output*. Fixed:
   arbitrary values are expected in the reference and are a defect only in the **final** output.
2. **De-absoluting the layout was missing entirely** — and it is the biggest task for this target,
   the analogue of translating reactivity semantics in the Leptos port. Added as Step 3 with a
   worked icon+label example, plus a conversion table in `react-to-nextjs.md`.

Also: fonts arrive as `font-['Family:Weight']` (fused, spaces→underscores) and one of them
(`PingFang HK`) is a system font not on Google Fonts — needs `next/font/local` or an explicit
fallback. Documented.

## Not yet exercised (still `[figma-doc]`)

- **A component with variant properties** — this file's components have none, so the
  variants→`states.machine` mapping remains the one unconfirmed core path. Needs a file with a real
  component set (e.g. a Button with `State=Default|Hover|Disabled`).
- `get_context_for_code_connect`, `list_file_components_for_code_connect` — presumed plan-gated like
  `get_code_connect_map`, unconfirmed.
- `get_motion_context` — unrun.
- Token tiering from real Figma variable names — needs a file that actually has variables.
