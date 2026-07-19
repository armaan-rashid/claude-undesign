# Design Port Pipeline — Skill Definitions for Cowork

**Goal:** A set of Claude Code skills that take a Claude Design export (React) and port it to
another framework — first target: **Leptos (Rust)** — via an explicit intermediate spec rather
than direct code translation. The pipeline generalizes to other targets (SwiftUI, GPUI) by
swapping the codegen skill.

**Core principle:** The durable interchange format is not code. It is a layered spec:
1. Design tokens (W3C DTCG JSON)
2. Component specs (anatomy, props/variants, token bindings)
3. Interaction contracts (state machines + ownership rules)
The React output is one *rendering* of this spec. We extract the spec, then re-render it in the
target framework, then verify visually against the original.

**Pipeline:** `excavate → extract → codegen → verify`, with an orchestrator skill coordinating
across git worktrees.

---

## Skill 1: `ds-contract-excavation`

```yaml
name: ds-contract-excavation
description: >
  Enumerate every implicit contract and ambient dependency in an AI-generated frontend export
  (Claude Design, v0, Lovable, or similar). Use this skill whenever the user wants to port,
  audit, refactor, or "understand the structure of" an AI-generated codebase, mentions "hidden
  dependencies," "implicit contracts," or asks what a generated export assumes about its
  environment. Always run this BEFORE any porting or spec-extraction work on an AI-generated
  export.
```

**Workflow:**
1. Inventory all files; classify: components, data, glue/runtime, assets, config.
2. Sweep for ambient dependencies, each recorded with file:line evidence:
   - Globals read or written (`window.*`, module-level singletons, injected runtime objects)
   - Functions/exports assumed to exist by name convention (magic names the scaffold's
     runtime calls)
   - Data-shape assumptions (component X reads field Y that nothing in-repo defines)
   - Storage/persistence conventions (e.g., artifact-style `window.storage` contracts)
   - Fetch/data-flow conventions (who fetches, who owns loading/error state)
   - CSS/theming assumptions (expected CSS vars, Tailwind config the export presumes)
   - Event conventions (custom events, pub/sub patterns, callback signatures)
3. For each contract, classify: **shim** (reproduce in target), **redesign** (replace with an
   explicit interface), or **drop** (dead in target context).
4. Output `CONTRACTS.md`: one section per contract with evidence, consumers, producers, and
   the shim/redesign/drop decision (decisions may be marked `TBD-user` for human review).

**Output:** `CONTRACTS.md` at repo root. This file is a required input to `ds-spec-extract`.

**Bundled resources:** `references/known-scaffolds.md` — catalog of known contract patterns per
generator (Claude Design, v0, Lovable), so the sweep starts from known signatures instead of
zero. Grow this file as new patterns are discovered.

---

## Skill 2: `ds-spec-extract`

```yaml
name: ds-spec-extract
description: >
  Extract a framework-agnostic design spec (W3C DTCG design tokens, component specs, and
  interaction state machines) from a frontend codebase or a Claude Design handoff bundle. Use
  whenever the user wants to port UI between frameworks, "extract the design system," produce
  tokens.json, generate component specs, or create any intermediate representation of a UI.
  Trigger even if the user only says "port this to <framework>" — extraction is step one.
```

**Inputs:** the export source tree, `CONTRACTS.md` (from Skill 1), and — if available — the
Claude Design handoff bundle (prefer the bundle's machine-readable spec over re-deriving from
React source; reconcile the two if both exist).

**Workflow:**
1. **Tokens:** collect all styling values → produce `spec/tokens.json` in W3C DTCG format with
   three tiers (primitive → semantic → component). Where the export used raw values, invent
   semantic names and flag them `"$extensions": {"inferred": true}` for human review. Emit a
   Tailwind `@theme` mapping alongside.
2. **Component inventory:** one `spec/components/<Name>.spec.yaml` per component:
   `anatomy`, `props` (typed, with defaults), `variants`, `tokens` (bindings into tokens.json),
   `states.machine` (statechart: initial state + event transitions), `behavior_contract`
   (ownership: who fetches, who holds state, what callbacks fire when — sourced largely from
   CONTRACTS.md), `a11y` (roles, aria, focus behavior).
3. **Layout semantics:** `spec/layout.md` — page composition, responsive breakpoints and what
   changes at each, grid/flex intent (describe intent, not implementation).
4. **Verification manifest:** `spec/verify.yaml` — list of routes/states to screenshot, used
   later by `ds-visual-verify` (each component's key states enumerated).

**Output:** `spec/` directory. This is the single source of truth for codegen; codegen agents
must not read the React source except when the spec is ambiguous, and any such fallback must be
logged as a spec gap.

---

## Skill 3: `ds-leptos-codegen`

```yaml
name: ds-leptos-codegen
description: >
  Generate idiomatic Leptos (Rust) components from a framework-agnostic spec directory
  (tokens.json + component specs + state machines). Use whenever the user wants to port React
  or AI-generated UI to Leptos, mentions Leptos, or asks to implement components from a spec/
  directory in Rust. Translates reactivity semantics (signals, memos, effects), not syntax.
```

**Workflow (per component, parallelizable across worktrees):**
1. Read the component's `.spec.yaml` + `tokens.json`. Do not read React source.
2. Apply the semantic translation table (bundled reference, summarized):
   - `useState` → `create_signal` / `RwSignal`
   - derived values → `Memo` (never effect-writes-signal)
   - `useEffect` → usually **nothing** (fine-grained reactivity subsumes most effects);
     genuine external-world sync → `Effect::new`
   - context → `provide_context` / `use_context`
   - props callbacks → `Callback<T>`
   - conditional render → `<Show>` / `match` in view; lists → `<For>` keyed properly
   - state machines → Rust enum + signal; transitions as methods; illegal transitions
     unrepresentable
3. Styling: port Tailwind class strings verbatim into `view!` macros; bind token-derived
   classes through the `@theme` mapping from Skill 2.
4. Server boundary: anything CONTRACTS.md classified as data-fetch conventions becomes explicit
   `#[server]` functions; component code never fetches directly (matches existing project
   conventions from the ecommerce site).
5. Emit component + a Storybook-style preview route per component/state (used by verify).
6. `cargo check` + `leptosfmt` before marking done.

**Bundled resources:** `references/react-to-leptos.md` (full translation table with
anti-patterns), `references/project-conventions.md` (import style, module layout, server-fn
patterns — seed from the existing Leptos ecommerce repo).

**Generalization note:** this skill is the swappable stage. `ds-swiftui-codegen` /
`ds-gpui-codegen` follow the same shape: read spec, apply that framework's semantic table, emit
preview surfaces. The spec format never changes.

---

## Skill 4: `ds-visual-verify`

```yaml
name: ds-visual-verify
description: >
  Adversarially verify a ported UI against its reference implementation via screenshot
  comparison. Use after any design port or codegen pass, whenever the user asks "does the port
  match," wants pixel/behavior fidelity checks, or mentions visual regression between two
  implementations of the same design.
```

**Workflow:**
1. Build both: reference (React export, `vite dev`) and target (Leptos, `trunk serve` or
   `cargo leptos watch`).
2. Drive both with Playwright using `spec/verify.yaml`: for each route/component/state, force
   the state (props harness or preview route), screenshot at each breakpoint.
3. **Adversarial verifier agent:** compares screenshot pairs. Reports discrepancies as
   structured findings: `{component, state, breakpoint, description, severity, suspected_cause:
   spec-gap | codegen-bug | token-mismatch | acceptable-divergence}`.
4. Behavior pass: replay each state machine's event sequence in the target; assert the
   resulting states/DOM match the spec (not the React implementation — the spec is truth).
5. Output `verify/findings.json` + human-readable `verify/REPORT.md`. Findings routed by
   suspected cause: spec-gap → back to Skill 2; codegen-bug → back to Skill 3.

**Note:** pixel-identical is a non-goal (font rendering, sub-pixel layout differ). The verifier
judges *design fidelity*: token values, spacing rhythm, hierarchy, state presentation.

---

## Skill 5: `ds-port-orchestrator`

```yaml
name: ds-port-orchestrator
description: >
  Orchestrate the full design-port pipeline (contract excavation → spec extraction → codegen →
  visual verification) across git worktrees with parallel per-component agents. Use whenever
  the user wants to port an entire Claude Design export or multi-component UI to another
  framework end to end, rather than a single component.
```

**Workflow:**
1. Session start: read pipeline state file (`.port-state.json`) — which components are
   extracted / generated / verified / signed-off.
2. Run Skills 1–2 once, serially (they're whole-repo). Pause for human review of `CONTRACTS.md`
   `TBD-user` decisions and inferred token names before codegen.
3. Fan out Skill 3 per component across worktrees (one branch per component or per component
   cluster). Merge order: tokens/theme first, leaf components, then composites.
4. Run Skill 4 after each merge wave; feed findings back as tasks; loop until findings are
   empty or marked acceptable-divergence.
5. Maintain `.port-state.json` and a CHANGELOG so any future agent (or session) can resume.

---

## Notes for Cowork (building these skills)

1. Build order: 1 → 2 → 3 → 4 → 5. Skills 1–2 are useful standalone even before codegen exists.
2. Per skill-creator methodology: draft each SKILL.md, then run test prompts and generate the
   eval viewer for human review BEFORE self-evaluating. Suggested test fixtures:
   - A real Claude Design export (user will supply one; ideally one already ported by hand so
     there's a known-good reference).
   - A deliberately contract-heavy synthetic export (hidden globals, magic function names) as
     the excavation stress test.
   - Trigger tests: "port this to Leptos", "what does this export assume about its
     environment", "extract the design system from this repo", "does my port match the
     original" should each trigger the right skill.
3. Keep each SKILL.md under ~500 lines; push translation tables and scaffold catalogs into
   `references/`.
4. The user's existing conventions to honor in Skill 3: Leptos `#[server]` functions for all
   data access (pattern already used with async-stripe in the ecommerce repo); Tailwind for
   styling.
5. Description optimization (run_loop) last, after skills are functionally validated.

## Open decisions (ask the user)

- Component spec format: YAML (proposed above) vs JSON — YAML reads better for humans, JSON
  round-trips better for tools. Proposal: YAML source, JSON emitted.
- State machine notation: bespoke minimal statechart (above) vs XState-compatible JSON.
  XState-compat buys existing visualizers; bespoke stays smaller.
- Whether Skill 4's behavior pass should run against the React reference at all, or purely
  against the spec (proposal: spec only; React is just the visual reference).
