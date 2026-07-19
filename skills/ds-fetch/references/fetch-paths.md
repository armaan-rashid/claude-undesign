# Routes Out of Claude Design

What each route produces on disk, and whether it is confirmed.

**Status legend** — same discipline as `known-scaffolds.md`:

- `confirmed` — observed working, with the run named.
- `unknown` — plausible, never tested. **Treat as a question, not a fact.**
- `unavailable` — tested and does not work for this purpose. Say why, so nobody retries it.

Promote or delete an entry the first time a real run touches it. Never let an `unknown` sit
through two runs — that means the probe in Step 1 isn't being recorded.

---

## R-1 — `DesignSync` tool, design-system project

**Status:** `confirmed` (ds-sync run 1, 2026-07-16, 83 files)

The tool exposed to Claude Code after `/design-login`. Read methods: `list_projects`,
`list_files`, `get_file`. Write methods exist (`finalize_plan`, `write_files`, `delete_files`) —
**this pipeline never calls them.**

Produces the design-system project's file tree:

```
colors_and_type.css          # tokens as CSS custom properties + semantic classes — the real token source
README.md                    # brand voice, foundations
_ds_manifest.json            # JSON list of tokens/themes/fonts/preview cards — CAN BE STALE
_adherence.oxlintrc.json     # lint rules banning raw hex/px/fonts; carries an x-omelette key
_ds_bundle.js                # generated component kit; self-registers onto window
components/core/*            # component sources
assets/*                     # svg, favicons, brand marks
preview/*                    # reference renders — the expected look
ui_kits/library/*            # future page; mirror only
uploads/*                    # binary reference material — ALWAYS SKIP
```

Confirmed constraints:

- `list_files` mixes bare directory entries in with files. Filter by "path is not a prefix of
  another path" before calling `get_file`.
- `get_file` caps at 256 KiB. `uploads/*` routinely exceeds it.
- Mirroring ~80 files costs roughly 250K tokens of context — delegate to a subagent.
- Freshness is checkable cheaply: `list_projects` exposes `updatedAt`. Compare before mirroring.

**Note for the port pipeline:** `_ds_manifest.json` lags `colors_and_type.css`. In the excavation
fixture it was missing an entire token tier (`--amber*`, `--font-terminal`) that the CSS had and
the app used. `ds-spec-extract`'s "prefer the bundle's machine-readable spec" rule must reconcile
against the CSS, and record disagreements rather than trusting the manifest. The CSS wins on
additive diffs.

## R-2 — `DesignSync` tool, app/site project

**Status:** `unknown` — **this is the open question `ds-fetch` Step 1 exists to settle.**

`list_projects` plainly returns *projects*; whether that set includes app/site projects (the kind
the port pipeline consumes) or only design-system projects has never been tested. Everything
known about `DesignSync` comes from one run against a DS project.

If it works, R-2 is the whole answer for the port pipeline and every UI route below becomes a
fallback. If it does not, the port pipeline's step 0 is irreducibly manual and this skill's job is
normalization, not fetching.

**First run must record the result here.** Replace this section with `confirmed` (and the shape of
what lands) or `unavailable` (and the error).

## R-3 — Export → Hand off to Claude Code

**Status:** `unknown` (documented by Anthropic; never exercised by this pipeline)

The documented handoff flow. Claude packages the project into a bundle intended to be passed to
Claude Code with a single instruction. Per the docs, the bundle carries **the project's design
files, the chat, and a README** telling the model how to interpret the designs.

Why this matters more than it sounds: the README and chat are *design intent*, which no amount of
reading React source recovers. `ds-spec-extract` has to invent semantic token names when the
export only carries raw values — intent is exactly the input that makes those names right rather
than plausible. If this route is available, prefer it over a bare file mirror even if R-2 works.

Unknown: whether it can be triggered non-interactively, and where the bundle lands on disk.

## R-4 — Export → save as folder / standalone HTML

**Status:** `unknown` (documented; never exercised here)

Export menu also offers: internal share URL, save as folder, standalone HTML, PDF, PPTX, and
handoff to third parties (Figma, Canva, Replit, Vercel, and others).

For porting, only "save as folder" and "standalone HTML" are relevant — PDF/PPTX discard the
structure the pipeline needs. Standalone HTML in particular is likely to be the `*.dc.html` +
`support.js` single-file shape catalogued as CD-1 in `known-scaffolds.md`.

## R-5 — Already on disk

**Status:** `confirmed` (how the excavation fixture `library-of-light/subwebsites` was obtained —
downloaded manually, well before the pipeline existed)

Always valid. The user points at a path; `ds-fetch` copies, hashes, and manifests it. No fetching.

Worth keeping as a first-class route rather than treating it as the degenerate case: it is the
only route guaranteed to work, it is how the pipeline's one real excavation actually happened, and
it keeps the skill useful when Claude Design is unreachable or the export predates the tooling.

---

## `/design-sync`

A Claude Code slash command that pulls a design system in from a GitHub repo, design files, raw
uploads, or a local codebase, so work in Claude Design starts from existing components.

Note the direction: this is for getting a design system **into** Claude Design. R-1 is the reverse.
Do not confuse them — the names are nearly identical and the data flows opposite ways.

---

## Recording a new route

```markdown
### R-n — <route name>

**Status:** confirmed (run: <which>) | unknown | unavailable (<why>)
**Trigger:** how it is invoked
**Produces:** the shape that lands on disk
**Constraints:** caps, auth, interactivity requirements
**Carries intent?** does it include README/chat/prose, or only files
```

"Carries intent?" is worth asking of every route. Files tell you what the design *is*; the README
and chat tell you what it *means*, and the meaning layer is the expensive thing to reconstruct.
