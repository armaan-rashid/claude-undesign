# Design Port Pipeline

Claude Code skills that port a Claude Design export (React) to another framework — first target
**Leptos (Rust)** — via an explicit intermediate spec rather than direct code translation.

Design doc: [`design-port-pipeline-skills.md`](./design-port-pipeline-skills.md).

Pipeline: `fetch → excavate → extract → codegen → verify`, orchestrated across git worktrees.

Two skills share the fetch: `ds-fetch` mirrors and hashes, `ds-sync` diffs the result and runs
its lanes. Transport-level knowledge (the `get_file` cap, bare-directory filtering, skip rules,
subagent delegation) lives in `ds-fetch` only.

## Status

| # | Skill | State |
|---|---|---|
| 0 | `ds-fetch` | route confirmed `[env]`; mirror still unrun |
| 1 | `ds-contract-excavation` | drafted, one real run |
| 2 | `ds-spec-extract` | drafted |
| 3 | `ds-leptos-codegen` | drafted |
| 4 | `ds-visual-verify` | drafted |
| — | `ds-sync` | drafted, one real run; Phase 1 delegates to `ds-fetch` |

Skills 1–2 are useful standalone before codegen exists. See `feedback/` for run notes.

**Step 0 is automatable.** The Claude Design MCP server (`claude-design`, 22 tools) reaches both
design systems and app/site projects — confirmed 2026-07-19. Setup has three separate failure
points that all look identical: consent is not server registration, `claude mcp list` reads config
while `/mcp` reads the running session, and `--scope user` is Claude Code only. See `ds-fetch`
Step 0.

**What is verified vs. merely written down** is tracked per claim, with `[env]` / `[run-1]` /
`[docs]` tags in `ds-fetch/references/fetch-paths.md`. This matters more than it sounds: four
separate authoritative-looking sources described interfaces that do not behave as described — the
support docs (`/design-login`), an earlier skill file (`get_file`), the `/design` autocomplete, and
the server's own tool annotations. Running the thing is the only evidence that counts.

## Layout

This repo is both a plugin marketplace and the plugin itself.

```
.claude-plugin/
  marketplace.json                     # marketplace "design-port", lists one plugin
  plugin.json                          # plugin "design-port-pipeline"
skills/
  ds-fetch/
    SKILL.md
    references/fetch-paths.md          # routes out of Claude Design; confirmed/unknown status
  ds-contract-excavation/
    SKILL.md
    references/known-scaffolds.md      # per-generator contract catalog; grows with use
  ds-spec-extract/
    SKILL.md
    references/spec-schema.md          # component spec schema (volatile layer)
    references/statechart-subset.md    # XState-compatible subset for states.machine
    references/tokens.md               # DTCG three-tier rules + Tailwind @theme mapping
  ds-leptos-codegen/
    SKILL.md
    references/react-to-leptos.md
    references/project-conventions.md
  ds-visual-verify/
    SKILL.md
    references/harness-recipes.md
  ds-sync/
    SKILL.md
feedback/                              # run notes, not shipped as skill content
```

`skills/`, `commands/`, `agents/` must sit at plugin root — **not** inside `.claude-plugin/`.
Each skill's `references/` ships automatically with it; no registration needed in `plugin.json`.

## Install

### For another Cowork or Claude Code session (recommended)

Push this repo to GitHub, then in the other session:

```
/plugin marketplace add <your-github-user>/<repo-name>
/plugin install design-port-pipeline@design-port
```

All five skills load together. To pick up later changes: `/plugin marketplace update`.

### From a local path

```
/plugin marketplace add "/Users/Armaan/Desktop/claude undesign"
/plugin install design-port-pipeline@design-port
```

Local-path marketplaces have a known issue where the plugin registers but loads **0 skills**
([#54967](https://github.com/anthropics/claude-code/issues/54967)). If that happens, symlink the
directory into `~/.claude/plugins/marketplaces/` before running `marketplace add`, which keeps the
relative `source: "./"` resolvable. Pushing to GitHub avoids this entirely.

### Single skill, no plugin

To use one skill in a target repo without installing the plugin, symlink it into `.claude/skills/`:

```sh
ln -s "$PWD/skills/ds-contract-excavation" /path/to/repo/.claude/skills/
```

### Validating before publishing

```sh
claude plugin validate --strict
```

Note what `validate` does **not** check: that each `SKILL.md` frontmatter `name` matches its
directory name. Mismatches load fine in Claude Code but fail silently in VS Code. They currently
all match — keep it that way when adding skills.

Also: **no angle brackets in frontmatter.** `<framework>` in a description parses as an XML tag and
blocks installation. This already bit `ds-spec-extract` once. Use a concrete example instead of a
placeholder.

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
