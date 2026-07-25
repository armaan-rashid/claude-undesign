---
name: ds-figma-extract
description: >
  Extract a framework-agnostic design spec (W3C DTCG tokens, component specs, interaction notes)
  from a Figma file, using the Figma MCP server. Use whenever the user wants to port a Figma
  design to code through the design-port pipeline, says "extract the design system from Figma",
  "Figma to spec", or "port this Figma to Next.js / SwiftUI / any framework". Produces the same
  spec/ directory ds-spec-extract does, so any ds-*-codegen target consumes it unchanged.
---

# Figma → Spec

The Figma counterpart of `ds-spec-extract`. Where that skill recovers a spec from a code export,
this one lifts it from a Figma file — and Figma is a *better* source, because the design data is
already structured: variables are tokens, components have declared variants, Code Connect already
maps components to code. Less reverse-engineering, more faithful mapping.

Output is the **same `spec/` directory** the code-export path produces. That is the whole point:
`ds-nextjs-codegen`, `ds-leptos-codegen`, and any future target read one spec format regardless of
whether it came from Figma or a React export.

> **Runs in Claude Code, not Cowork.**
>
> Needs the Figma MCP server (`mcp__b78f5274-…__*` / tools like `get_design_context`) connected in
> Claude Code, and a target repo on disk to seed conventions from. Cowork cannot reach Claude
> Code's MCP servers.
>
> **If the Figma tools are absent, stop and say so** — do not hand-write a spec from a screenshot.
> Confirm with `whoami` (the Figma MCP's auth check) before concluding they are missing.

## Orchestrate Figma's own skills — do not reinvent them

The Figma MCP ships skills that already do the hard node→code work. **Load them; build on top.**

- **`figma-design-to-code`** is a **mandatory prerequisite before `get_design_context`.** Load it
  (`get_figma_skill` → `skill://figma/figma-design-to-code/SKILL.md`, or the `/figma-design-to-code`
  slash skill) and pass `skillNames: "resource:figma-design-to-code"` on every `get_design_context`
  call. Skipping it is a documented failure mode.
- **`figma-code-connect`** governs the component↔code mappings this skill reads.
- **`figma-implement-motion`** + `get_motion_context` own animation; defer to them.
- A **`figma-swiftui`** skill already exists — relevant when the eventual target is SwiftUI.

This skill owns only what Figma's skills do **not**: lifting their output into the durable,
framework-agnostic `spec/` — token tiering, component-spec schema, verification manifest.

## The honest scope note

For a **one-shot Figma → Next.js** job with no second target, `figma-design-to-code` alone is
lighter — it emits React+Tailwind directly. The spec earns its cost when (a) you have **more than
one** target framework, (b) you want a **durable, tiered token system** rather than inline values,
or (c) you want **verification against a spec** rather than against one implementation. If none of
those hold, say so and use `figma-design-to-code` directly. Do not build a spec nobody consumes.

## The spec format is defined elsewhere — reuse it, do not fork it

The spec schema is the volatile layer and has **one** source of truth: the `ds-spec-extract` skill's
bundled references. Both skills install together in this plugin, so those files sit alongside this
one on disk. Read them from the **sibling `ds-spec-extract` skill directory** and follow them
exactly (find them under the plugin's `skills/ds-spec-extract/references/`):

- `tokens.md` — DTCG three-tier tokens + Tailwind `@theme`.
- `spec-schema.md` — the component `.spec.yaml` schema (`specVersion`, the volatile layer).
- `statechart-subset.md` — the XState-compatible subset for `states.machine`.

Duplicating them here would fork the schema — the exact failure the single-source design prevents.

This skill's own `references/figma-mcp-map.md` is the only Figma-specific reference: which MCP tool
populates which spec artifact, and where the returns are verified vs assumed.

## Inputs

1. A Figma URL with a **node id** (`?node-id=1-2`). A file-only URL is not enough — the tools need
   a node target. Ask for a node-specific URL rather than guessing (`figma-design-to-code` error
   recovery says the same).
2. The **target repo** — to discover existing components worth binding to via Code Connect, and so
   the downstream codegen skill can seed its own `project-conventions.md` from it. (That conventions
   file belongs to the codegen skill, not this one.)

## Outputs

The `spec/` directory per `ds-spec-extract`, plus a Figma-specific gaps file:

```
spec/
  tokens.json          # from get_variable_defs — usually the richest, most faithful artifact
  theme.css            # Tailwind @theme, generated from tokens.json
  components/
    <Name>.spec.yaml
    <Name>.spec.json
  layout.md
  verify.yaml          # screenshot anchors — get_screenshot per node/state
  FIGMA-GAPS.md        # what Figma could not tell us (see Step 5)
```

## Step 1 — Orient without spending the whole file

`get_metadata` on the target node returns structure (ids, types, names, sizes) cheaply. Use it to
pick the real component nodes before pulling full context. Do **not** `get_design_context` the
whole file blind — it is the expensive call.

## Step 2 — Tokens (do this first; it is the strongest artifact)

`get_variable_defs` returns the design variables a node uses — Figma variables **are** design
tokens, so this maps to `tokens.json` more directly than anything in the code-export path.

Read `references/figma-mcp-map.md` for the mapping and this critical constraint: **`get_variable_defs`
is per-node** — it returns variables *used by that node*, not the file's whole collection. To build
complete tokens you union across the key nodes (and consult `get_libraries`). A token tier that
looks empty may just mean you queried the wrong node.

Tier the result per `ds-spec-extract`'s bundled `tokens.md` (primitive → semantic → component).
Figma variable **names often already encode tiers** (`color/text/primary`, `space/2`) — preserve
that structure; it is designer intent, not something to infer. Flag any value you had to name
yourself `inferred: true`.

## Step 3 — Components

For each component node, per `references/figma-mcp-map.md`:

- `get_design_context` (with the mandatory skill loaded) → reference structure and the hint
  bundle. Use hints by the skill's priority: Code Connect > docs > annotations > tokens > raw.
- `get_code_connect_map` / `list_file_components_for_code_connect` → existing code components.
  **A Code Connect mapping is a binding, not a suggestion**: record it in `behavior_contract` so
  codegen reuses the real component instead of regenerating it.
- Figma **variant properties** map to spec `variants` and, where they name interaction/availability
  dimensions (hover, pressed, disabled), to `states.machine` parallel regions.

**On state machines — do not invent them.** Figma encodes *visual* states as variants, not
behavioral transitions. A `State=Hover` variant tells you hover looks different; it does **not**
tell you the transition graph. Derive what the variants and any prototype reactions genuinely
support; mark the rest `TBD-user` in `FIGMA-GAPS.md`. A fabricated statechart is worse than an
honest gap — codegen will implement the fabrication.

## Step 4 — Layout and verification manifest

- `layout.md` — composition and responsive intent. Figma frames give you sizes and constraints;
  translate to intent ("sidebar collapses below `md`"), not pixel positions. Absolute coordinates
  are the weakest hint per the design-to-code priority — lean on structure.
- `verify.yaml` — one screenshot anchor per component key-state, captured with `get_screenshot`
  (note its ~7-day URL expiry; download bytes if the manifest must persist).

## Step 5 — Record what Figma could not say

`FIGMA-GAPS.md` — the honest ledger, and larger than a code export's because a design encodes less
behavior than code does:

- **State transitions** variants imply but do not define.
- **Data ownership** — who fetches, who holds state. Figma has none of this; it is all `TBD-user`
  or comes from the target repo, never invented here.
- **Interaction/async behavior** beyond prototype reactions.
- **Tokens only reachable from nodes you did not query** (the per-node constraint above).

## Anti-patterns

- **Reinventing `get_design_context`.** If this skill is hand-parsing Figma node JSON to produce
  React, it has bypassed the mandatory design-to-code skill. Orchestrate it.
- **Inventing behavior the design does not encode.** Variants are looks, not transitions. Gap it.
- **Binding tokens by value.** Figma hands you `#2563eb`; the spec binds `{color.text.primary}`.
  Tier it, name it, reference it — same rule as every other source.
- **Querying one node and trusting the token set.** `get_variable_defs` is per-node. Union, or
  under-count.
- **Emitting code.** This skill stops at `spec/`. Codegen is a separate, swappable stage.
