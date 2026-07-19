# Routes Out of Claude Design

What each route produces on disk, and whether it is confirmed.

**Status legend** — same discipline as `known-scaffolds.md`:

- `confirmed` — observed working, with the run named.
- `unknown` — plausible, never tested. **Treat as a question, not a fact.**
- `unavailable` — tested and does not work for this purpose. Say why, so nobody retries it.

Promote or delete an entry the first time a real run touches it. Never let an `unknown` sit
through two runs — that means the probe in Step 1 isn't being recorded.

---

## Provenance convention

Every factual claim below is tagged by **where it came from**, because they are not equally
trustworthy and this file has already been wrong once:

- `[env]` — observed in a real Claude Code environment. **Strongest.**
- `[run-1]` — from the ds-sync run of 2026-07-16. Real, but a single run, and possibly stale now.
- `[docs]` — from Anthropic support articles. **Weakest for command surfaces** — see the staleness
  note below.

## Staleness warning — docs lost to environment once already

On 2026-07-19, the support article ["Get started with Claude Design"](https://support.claude.com/en/articles/14604416-get-started-with-claude-design)
documented `/design-login` as the sign-in step. In the actual environment that command **does not
exist**; the surface is `/design` with subcommands, and auth is `/design consent`.

Two rounds of web research "confirmed" the wrong answer before a human ran the command.

The lesson is not "that one line was wrong." It is that **for command surfaces, documentation is a
lead and the environment is the evidence.** Anything tagged `[docs]` here is unverified until
someone runs it. Do not promote `[docs]` to fact by finding a second article that repeats it —
that is the exact move that produced the error.

## R-1 — Claude Design MCP server, design-system project

**Status:** capability `confirmed` `[run-1]`; tool naming **unverified**

A design-system project was successfully mirrored — 83 files, 2026-07-16 `[run-1]`. That the
capability exists is not in doubt.

What **is** in doubt is the interface. `ds-sync` names the tool `DesignSync` with read methods
`list_projects`, `list_files`, `get_file`, and write methods `finalize_plan`, `write_files`,
`delete_files`. Those names entered this pipeline through `ds-sync` and have never been checked
against the MCP server's actual tool list. Given the `/design-login` episode, treat them as
`[run-1]` recollection rather than API documentation.

**First run: enumerate the server's real tools and record them here.** Then this section can be
promoted to `[env]`.

**The read-only policy stands regardless of naming.** Whatever the write methods turn out to be
called, this pipeline never calls them.

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

## R-2 — Claude Design MCP server, app/site project

**Status:** `unknown` — **this is the open question `ds-fetch` Step 1 exists to settle.**

Whether the server reaches app/site projects (the kind the port pipeline consumes) or only
design-system projects has never been tested. Everything known comes from one run against a DS
project `[run-1]`.

**Lead, not evidence** `[docs]`: the support article says that once connected you can "import a
design into your codebase, export your code as a live prototype, or let Claude build the whole
thing from start to finish." Importing a design into a codebase is exactly R-2. That is
encouraging and it is still `[docs]` — the same source that invented `/design-login`. Run it.

If it works, R-2 is the whole answer for the port pipeline and every UI route below becomes a
fallback. If it does not, the port pipeline's step 0 is irreducibly manual and this skill's job is
normalization, not fetching.

**First run must record the result here.** Replace this section with `confirmed` (and the shape of
what lands) or `unavailable` (and the error).

## R-3 — Export → Handoff to Claude Code

**Status:** `unknown` `[docs]` — never exercised by this pipeline

Two sub-options under "Handoff to Claude Code" in the Export menu `[docs]`:

- **Send to local coding agent** — the relevant one; lands in a local Claude Code session.
- **Send to Claude Code Web**

Per the docs the bundle carries **the project's design files, the chat, and a README** telling the
model how to interpret the designs.

Why this matters more than it sounds: the README and chat are *design intent*, which no amount of
reading React source recovers. `ds-spec-extract` has to invent semantic token names when the
export only carries raw values — intent is exactly the input that makes those names right rather
than plausible. If this route is available, prefer it over a bare file mirror even if R-2 works.

Unknown: whether it can be triggered non-interactively, and where the bundle lands on disk.

## R-4 — Export → Download as .zip / standalone HTML

**Status:** `unknown` `[docs]` — never exercised here

The full Export menu `[docs]`, verbatim: Download as .zip · Export as PDF · Export as PPTX · Send
to Canva · Export as standalone HTML · send to Adobe, Base44, Canva, Gamma, Lovable, Miro, Replit,
Vercel, Wix · Handoff to Claude Code (R-3). Projects can also be shared inside the org by link
with view / comment / edit access.

For porting, only **Download as .zip** and **Export as standalone HTML** are relevant — PDF and
PPTX discard the structure the pipeline needs. Standalone HTML is likely the `*.dc.html` +
`support.js` single-file shape catalogued as CD-1 in `known-scaffolds.md`.

(An earlier revision of this file called this "save as folder." That was a paraphrase of a search
summary, not the menu. The menu says .zip.)

## R-5 — Already on disk

**Status:** `confirmed` (how the excavation fixture `library-of-light/subwebsites` was obtained —
downloaded manually, well before the pipeline existed)

Always valid. The user points at a path; `ds-fetch` copies, hashes, and manifests it. No fetching.

Worth keeping as a first-class route rather than treating it as the degenerate case: it is the
only route guaranteed to work, it is how the pipeline's one real excavation actually happened, and
it keeps the skill useful when Claude Design is unreachable or the export predates the tooling.

---

## Adjacent commands

`[docs]`, and therefore unverified — check against `/design` subcommands in the environment before
relying on any of it.

- **`/design-sync`** — pulls a design system in from a GitHub repo, design files, raw uploads, or a
  local codebase. Described as **bidirectional**: pull a system in, or push code changes back to
  Claude Design. An earlier revision of this file called it one-way into Claude Design; that was
  wrong.
- **`/design`** — the actual command surface `[env]`, with subcommands. `consent` and `revoke` are
  confirmed. **The full subcommand list has not been enumerated** — do that on the first run and
  record it here, since it likely supersedes the hyphenated forms in the docs.

Whether `/design-sync` still exists as a hyphenated command, or has become `/design sync`, is
exactly the kind of thing the `/design-login` episode says not to assume. Check.

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
