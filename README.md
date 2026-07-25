# Design Port Pipeline

Claude Code skills that port a UI design to a target framework via an explicit intermediate spec
rather than direct code translation. The durable interchange is the spec (DTCG tokens + component
specs + statecharts); sources and targets plug into it.

Design doc: [`design-port-pipeline-skills.md`](./design-port-pipeline-skills.md).

Pipeline shape — **two source branches feed one spec feed many targets:**

```
  SOURCES                     SPEC                  TARGETS
  Claude Design ─ ds-fetch ─┐
                            ├─ ds-contract-excavation → ds-spec-extract ─┐
  (code export)  ───────────┘                                           │
                                                          spec/  ────────┼─ ds-leptos-codegen  → Leptos
  Figma ──────── ds-figma-extract ──────────────────────────────────────┤─ ds-nextjs-codegen  → Next.js
                                                                         └─ (ds-swiftui, … )
                                                                            ↓
                                                                     ds-visual-verify
```

The spec never changes; only the source adapter and the codegen target do. That is what makes a new
framework cheap — add one codegen skill, reuse extract + verify.

## Status

| Stage | Skill | State |
|---|---|---|
| source (Claude Design) | `ds-fetch` | route confirmed `[env]`; mirror still unrun |
| source (Claude Design) | `ds-sync` | one real run (repo-side); transport unrun |
| source (Figma) | `ds-figma-extract` | MCP run once `[env]`; returns confirmed, variants path still unrun |
| excavate | `ds-contract-excavation` | drafted, one real run |
| extract | `ds-spec-extract` | drafted |
| codegen | `ds-leptos-codegen` | drafted |
| codegen | `ds-nextjs-codegen` | drafted, unrun |
| verify | `ds-visual-verify` | drafted |

`ds-figma-extract` produces the **same `spec/`** as `ds-spec-extract`, so both codegen targets
consume it unchanged. See `feedback/` for run notes.

**Figma branch — orchestrates, does not reinvent.** The Figma MCP ships its own design-to-code
skills (`figma-design-to-code` is a mandatory prerequisite before `get_design_context`;
`figma-code-connect`, `figma-swiftui`, `figma-implement-motion` exist too). `ds-figma-extract` loads
and builds on them, adding only the spec layer they skip.

The first real MCP run (2026-07-19, notes in `feedback/`) already overturned two doc-based
assumptions: **Code Connect is plan-gated** (needs a Dev/Full seat on Org/Enterprise — a
team/student seat errors), and a **raster/flattened node yields nothing extractable** (empty
variables, a lone `<img>`) — now a hard guardrail in the skill's Step 1. Return shapes are confirmed
`[env]` in `figma-mcp-map.md`; the variants→statechart path is the next probe.

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
  ds-figma-extract/
    SKILL.md
    references/figma-mcp-map.md         # Figma MCP tool → spec artifact; reuses ds-spec-extract's format refs
  ds-leptos-codegen/
    SKILL.md
    references/react-to-leptos.md
    references/project-conventions.md
  ds-nextjs-codegen/
    SKILL.md
    references/react-to-nextjs.md       # RSC boundary traps, Tailwind v4, assets/fonts
    references/project-conventions.md   # App Router / RSC / Tailwind v4 seed
  ds-visual-verify/
    SKILL.md
    references/harness-recipes.md
  ds-sync/
    SKILL.md
feedback/                              # run notes, not shipped as skill content
templates/                             # .claude/settings.json template for distribution
```

`skills/`, `commands/`, `agents/` must sit at plugin root — **not** inside `.claude-plugin/`.
Each skill's `references/` ships automatically with it; no registration needed in `plugin.json`.

## Install

Three routes. Pick by how much you care about updates propagating without anyone doing anything.

| Route | Updates on push? | Works in |
|---|---|---|
| A. Claude Code plugin + `autoUpdate` | Yes, next session | Claude Code |
| B. Cowork org marketplace + GitHub sync | Yes, on PR merge | Cowork, **Team/Enterprise only** |
| C. Skills committed to `.claude/skills/` | Yes, **mid-session** | Both |

### A — Claude Code plugin, auto-updating

Copy [`templates/claude-settings.json`](./templates/claude-settings.json) to the *consuming* repo's
`.claude/settings.json`, set `OWNER/REPO`, and commit. Anyone who clones and trusts the folder is
prompted to install; they are never auto-installed without consent.

**`autoUpdate` must be explicit** — third-party marketplaces default to disabled. With it on,
Claude Code checks after session start (random delay up to ~10 min) and applies **on the next
session**; plugin skills load at session start only. `/reload-plugins` applies them sooner.

Manual equivalent, per person:

```
/plugin marketplace add OWNER/REPO
/plugin install design-port-pipeline@design-port
/plugin marketplace update          # to refresh later
```

### B — Cowork organization marketplace

**Team/Enterprise plan owners only.** On Pro/Max there is no auto-sync; manual update is the
ceiling.

Organization settings → Plugins → Add plugin → GitHub → `owner/repo`. Then open the marketplace's
menu and toggle **Sync automatically** — it re-syncs whenever a PR merges. Installation preference
can be set to *Installed by default* or *Required* so members get it with no action.

**The repo must be private or internal.** Public repos are not permitted for org marketplaces, and
must be on github.com (no GitHub Enterprise Server). The Claude GitHub App has to be installed on
the repo. Note this conflicts with route A, where public is fine — if you need both, private plus
granting collaborators access covers it.

Only relative `source` paths (`"./"`, `"./plugins/x"`) are fully supported; `github`, `url`, and
`git-subdir` sources work only against public targets, and `npm`/`pip` are unsupported. This repo
uses `"source": "./"`, so it is fine.

A failed sync can temporarily remove plugins from members. If that happens, fix, push, re-sync, and
**re-check installation preferences** — they can reset.

### C — Skills committed to `.claude/skills/` (the only live one)

Skills in `.claude/skills/` are **live-watched**: adding, editing, or removing one takes effect
inside a running session, no restart. Plugin skills are not. So if you want `git pull` to update an
in-flight session, copy `skills/*` into the consuming repo's `.claude/skills/` and commit them.

Trade-off: you are syncing copies rather than referencing one source, so they drift unless a
subtree/submodule/CI step keeps them current — and submodule behavior under `.claude/skills/` is
undocumented.

### Single skill, no plugin

```sh
ln -s "$PWD/skills/ds-contract-excavation" /path/to/repo/.claude/skills/
```

Symlink handling under `.claude/skills/` is undocumented (the documented rules cover plugins).
Works locally; verify before depending on it.

### Local-path marketplace

Known issue: registers but loads **0 skills**
([#54967](https://github.com/anthropics/claude-code/issues/54967)). Symlink into
`~/.claude/plugins/marketplaces/` first, or just use a GitHub route.

### `.skill` bundles

`dist/*.skill` are zipped skill directories with their `references/` intact — the plain "save
skill" button takes only `SKILL.md` and silently drops the bundled files. Rebuild after any change:

```sh
for d in skills/*/; do n=$(basename "$d"); \
  (cd skills && zip -rq "../dist/$n.skill" "$n" -x '*.DS_Store'); done
```

The `-x '*.DS_Store'` is not optional on macOS — without it Finder metadata ships inside the
bundle. (Verified: a bare `zip -r` picks it up.) Directory entries also get included; harmless,
but it means archive listings differ from a scripted build. Confirm a rebuild with:

```sh
python3 -c "import zipfile,glob;[print(f, zipfile.ZipFile(f).namelist()) for f in sorted(glob.glob('dist/*.skill'))]"
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
