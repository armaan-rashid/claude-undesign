---
name: ds-spec-extract
description: >
  Extract a framework-agnostic design spec (W3C DTCG design tokens, component specs, and
  interaction state machines) from a frontend codebase or a Claude Design handoff bundle. Use
  whenever the user wants to port UI between frameworks, "extract the design system," produce
  tokens.json, generate component specs, or create any intermediate representation of a UI.
  Trigger even if the user only says "port this to Leptos" or names any other target framework —
  extraction is step one.
---

# Spec Extraction

The durable interchange format between a design and an implementation is **not code**. It is a
layered spec: tokens, component specs, interaction contracts. The React export is one *rendering*
of that spec. This skill recovers the spec so it can be re-rendered elsewhere.

The discipline that makes this work: **codegen never reads the React source.** If the spec is
ambiguous, that is a bug in the spec, and the fix belongs here — not a lookup downstream. Every
codegen fallback to React source is a logged spec gap that comes back to this skill.

## Inputs

1. The export source tree.
2. `CONTRACTS.md` from `ds-contract-excavation` — **required**. If it does not exist, stop and
   run that skill first. Its contract IDs (`C-01`, …) are cited throughout the spec you emit.
3. The Claude Design handoff bundle, if one exists. **Prefer the bundle's machine-readable spec
   over re-deriving from React source.** If both exist, reconcile them and record every
   disagreement — a bundle/source disagreement is a real finding, usually meaning the source
   drifted from the design.

## Outputs

```
spec/
  tokens.json              # W3C DTCG, three tiers
  theme.css                # Tailwind @theme mapping, generated from tokens.json
  components/
    <Name>.spec.yaml       # authored/emitted source of truth (human-reviewable)
    <Name>.spec.json       # generated from the YAML, for tools
  layout.md
  verify.yaml
  SPEC-GAPS.md             # anything you could not resolve
```

**YAML is source; JSON is emitted.** Never hand-edit the `.json` — regenerate it. The emit step
is mechanical (`yq`, or any YAML→JSON pass) and must be idempotent.

**Quote `"on"` in every statechart.** YAML 1.1 resolves bare `on` to `true`, so an unquoted
transition key emits as `{"true": {...}}` — well-formed YAML, invalid XState, no error. Verify no
boolean keys survive the emit. See `references/statechart-subset.md`.

**Every spec file carries `specVersion`.** The component spec schema is expected to change; the
tokens layer is not (DTCG is a real standard). Version them separately so a component-schema
revision does not invalidate the token work.

The component spec schema lives in `references/spec-schema.md`, not in this file. **Read it
before emitting any component spec.** It is the volatile layer — revising the format means
editing that reference, not rewriting this skill.

## Step 1 — Tokens

Collect every styling value in the export → `spec/tokens.json` in W3C DTCG format, three tiers:
primitive → semantic → component. Read `references/tokens.md` for the tier rules, DTCG type
requirements, and the `@theme` mapping procedure.

Where the export used raw values with no semantic name, you must invent one. Flag every invented
name:

```json
"$extensions": { "dsPort": { "inferred": true, "evidence": ["Button.tsx:34"] } }
```

Inferred names are the highest-risk output of this skill — a wrong semantic name propagates into
every component that binds it, and it is far cheaper to fix here than after codegen. They gate
the pipeline: `ds-port-orchestrator` pauses for human review of every `inferred: true` before
codegen runs.

Emit `spec/theme.css` (Tailwind `@theme`) alongside. Codegen binds token-derived classes through
this mapping rather than re-deriving values.

## Step 2 — Component inventory

One `spec/components/<Name>.spec.yaml` per component. Per `references/spec-schema.md`, each
carries:

- `anatomy` — the part structure. Named parts, nesting, what is required vs optional.
- `props` — typed, with defaults. Types are spec-level, not TypeScript.
- `variants` — the axes and their values.
- `tokens` — bindings into `tokens.json`. Bind by token path, never by literal value.
- `states.machine` — statechart. **Read `references/statechart-subset.md` before writing one.**
  The notation is an XState-compatible subset; using constructs outside the subset produces specs
  codegen cannot lower.
- `behavior_contract` — ownership. Who fetches, who holds state, what callbacks fire when.
  Sourced largely from `CONTRACTS.md`; cite contract IDs.
- `a11y` — roles, aria, focus behavior, keyboard interaction.

**On `states.machine` and orthogonality:** most components have several independent state
dimensions (interaction: idle/hover/pressed; availability: enabled/disabled; async:
idle/loading/error). These are parallel regions, not one flat enum. Model them as parallel — do
not multiply them into a combinatorial state set, and do not silently drop a dimension because it
did not fit. If a dimension genuinely belongs in props or CSS rather than the machine, say so in
the spec and why. The reference covers where to draw the line.

## Step 3 — Layout semantics

`spec/layout.md` — page composition, responsive breakpoints and **what changes at each**,
grid/flex intent.

Describe **intent, not implementation.** "Sidebar collapses to a drawer below `md`; main content
goes full-bleed" is intent. "`flex-col md:flex-row`" is implementation, and it is already in the
source — restating it here buys nothing and quietly re-couples the spec to React's rendering.

## Step 4 — Verification manifest

`spec/verify.yaml` — the routes/states `ds-visual-verify` will screenshot.

Enumerate each component's **key states**, not its state space. The cross-product of every
parallel region is unreviewable; pick the states that carry design meaning (each variant at rest,
each async state, disabled, focus-visible) and the breakpoints where layout actually changes per
`layout.md`.

The behavior pass asserts against **this spec**, not against the React implementation. React is
the visual reference only. This means a spec bug passes verification silently — which is exactly
why the human review gates in Steps 1 and 2 are load-bearing rather than ceremonial.

## Step 5 — Record gaps

`spec/SPEC-GAPS.md`. Anything you could not resolve from source + `CONTRACTS.md` + bundle:

````markdown
### SG-02 — Toast dismissal timing unspecified

**Component:** Toast
**Question:** Auto-dismiss delay is `3000` in source (`Toast.tsx:22`) with no token or config
backing it. Is this a design decision or an arbitrary generator default?
**Blocked:** `states.machine` for Toast — the `after` transition needs a duration token.
**Proposed default:** promote to `duration.toast-dismiss` token, flagged inferred.
````

A gap is not a failure. An unrecorded gap is — it becomes a codegen agent guessing in a worktree
three steps downstream, where the guess is invisible and expensive.

## Anti-patterns

- **Transcribing React into YAML.** If the spec has a `useState` in it, or a `className` string,
  or a hook name, the extraction failed. The spec describes what the component *is*, not how this
  export happened to build it.
- **Binding tokens by value.** `color: "#0A0A0A"` in a component spec defeats the entire layered
  design. Bind the path: `color: "{semantic.text.primary}"`.
- **Inventing semantic names silently.** Every invented name is `inferred: true` with evidence.
  No exceptions — the review gate depends on the flag being trustworthy.
- **Flattening parallel state.** A `Button` with nine states is a modelling failure, not a
  thorough spec.
- **Skipping CONTRACTS.md.** `behavior_contract` without it is invention. Ownership is not
  recoverable from component source alone — that is the whole reason Skill 1 runs first.
