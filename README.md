# Design Port Pipeline

Claude Code skills that port a Claude Design export (React) to another framework — first target
**Leptos (Rust)** — via an explicit intermediate spec rather than direct code translation.

Design doc: [`design-port-pipeline-skills.md`](./design-port-pipeline-skills.md).

Pipeline: `excavate → extract → codegen → verify`, orchestrated across git worktrees.

## Status

| # | Skill | State |
|---|---|---|
| 1 | `ds-contract-excavation` | drafted |
| 2 | `ds-spec-extract` | drafted |
| 3 | `ds-leptos-codegen` | not started |
| 4 | `ds-visual-verify` | not started |
| 5 | `ds-port-orchestrator` | not started |

Skills 1–2 are useful standalone before codegen exists. Neither has been run against a real
export yet — see Next.

## Layout

```
skills/
  ds-contract-excavation/
    SKILL.md
    references/known-scaffolds.md      # per-generator contract catalog; grows with use
  ds-spec-extract/
    SKILL.md
    references/spec-schema.md          # component spec schema (volatile layer)
    references/statechart-subset.md    # XState-compatible subset for states.machine
    references/tokens.md               # DTCG three-tier rules + Tailwind @theme mapping
```

To use in a target repo, symlink or copy into `.claude/skills/`:

```sh
ln -s "$PWD/skills/ds-contract-excavation" /path/to/repo/.claude/skills/
ln -s "$PWD/skills/ds-spec-extract"        /path/to/repo/.claude/skills/
```

## Decisions

Resolving the design doc's "Open decisions" section.

**Component spec format — YAML source, JSON emitted.** Humans review and edit YAML; tools consume
generated JSON. Never hand-edit the JSON. Because the component spec standard is expected to
change, the schema lives in `references/spec-schema.md` rather than in `SKILL.md` prose — a format
revision is one file edit, not a skill rewrite. Every spec file carries `specVersion`, versioned
independently of `tokens.json`.

**Stable vs. volatile layers.** Tokens rest on W3C DTCG, a real standard — stable. Component specs
are a local invention with no winning standard — volatile. Versioned separately so churn in the
latter does not invalidate work in the former.

**State machines — XState-compatible subset.** XState's key names and semantics, restricted to
states, events, transitions, guards, and parallel regions. Rationale: component state is not one
dimension (a button is `idle|hover|pressed` **and** `enabled|disabled` **and**
`idle|loading|error`), and a flat enum either multiplies those into meaningless states or drops a
dimension. Parallel regions solve it. Taking XState's shape means machines validate in Stately's
visualizer for human review, at no cost — but the subset stays small because every construct
`ds-spec-extract` can emit is one some codegen skill must lower to a Rust enum. Codegen lowers
each region to its own signal. See `references/statechart-subset.md` for the growth procedure.

**Verification asserts against the spec, not React.** React is the visual reference only; behavior
truth lives in the spec. Consequence: a spec bug passes verification silently, which is why the
human review gates (inferred tokens, `TBD-user` contracts) are load-bearing rather than
ceremonial.

## Design notes

**Contract IDs are a public interface.** `CONTRACTS.md` numbers contracts `C-01`, `C-02`, … and
spec files cite them. Append across runs; never renumber.

**Codegen never reads React source.** If the spec is ambiguous that is a spec bug, fixed in Skill
2. Any codegen fallback to source is logged as a spec gap and routed back.

**Human gates before codegen.** `TBD-user` contract decisions and `inferred: true` token names both
pause the pipeline. They are cheap to review here and expensive to discover after a fan-out.

## Next

1. Get a real Claude Design export as a fixture — ideally one already ported by hand, so there is
   a known-good reference. Fill in the Claude Design section of `known-scaffolds.md` from it; it is
   currently `unfilled` placeholders rather than guesses.
2. Build a deliberately contract-heavy synthetic export (hidden globals, magic names, undefined
   CSS vars) as the excavation stress test.
3. Trigger tests — these should each fire the right skill:
   - "port this to Leptos" → `ds-spec-extract`
   - "what does this export assume about its environment" → `ds-contract-excavation`
   - "extract the design system from this repo" → `ds-spec-extract`
   - "does my port match the original" → `ds-visual-verify` (once built)
4. Per skill-creator methodology: run test prompts and generate the eval viewer for human review
   **before** self-evaluating. Description optimization (`run_loop`) last, after the skills are
   functionally validated.
5. Then Skill 3 (`ds-leptos-codegen`), seeding `references/project-conventions.md` from the
   existing Leptos ecommerce repo (`#[server]` functions for all data access, Tailwind styling).
