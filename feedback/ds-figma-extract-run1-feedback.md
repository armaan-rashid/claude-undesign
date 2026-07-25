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

## Not yet exercised (still `[figma-doc]`)

- A **native component with variants** — the whole variants→`states.machine` mapping is still
  unconfirmed against real return data. This is the highest-value next probe: point at a real
  component set (e.g. a Button with a `State` variant property) and confirm how variant properties
  come back and whether `get_context_for_code_connect` is also plan-gated.
- `get_libraries`, `get_motion_context` — unrun.
- Token tiering from real Figma variable names (`color/text/primary` → `semantic.text.primary`) —
  needs a node that actually binds variables.
