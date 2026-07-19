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

## Three sources, three wrong answers

Every authoritative-looking description of the Claude Design interface has so far been wrong.
Recorded here because the correction keeps getting re-introduced by whoever reads the next
plausible source.

| Source | Claimed | Reality |
|---|---|---|
| [Support article](https://support.claude.com/en/articles/14604416-get-started-with-claude-design) | `/design-login` signs you in | Command does not exist |
| `ds-sync/SKILL.md` | A `DesignSync` tool with `list_projects` / `list_files` / `get_file` | Never verified against a live server |
| `/design` autocomplete | `[sync\|login\|import\|export\|status\|<prompt>]` | Five are inert; the two that work (`consent`, `revoke`) aren't listed |

Two rounds of web research "confirmed" `/design-login` before a human ran it. Autocomplete was
believed for exactly one message before the same human ran those too.

**The rule: an advertised surface is a claim, not evidence.** Documentation, an existing skill
file, and an autocomplete hint are all leads of equal (low) weight. Running the thing is evidence.
Do not promote `[docs]` to fact by finding a second source that repeats it — that is precisely the
move that produced the first error.

## R-1 — Claude Design MCP server, design-system project

**Status:** server `confirmed` `[env]`; **tool surface not yet enumerated**

### Connecting it — verified working, 2026-07-19

```sh
claude mcp add --scope user --transport http claude-design https://api.anthropic.com/v1/design/mcp
```

`claude mcp list` then shows `claude-design: https://api.anthropic.com/v1/design/mcp (HTTP) - ✔ Connected`.

**Consent and server registration are two separate things.** `/design consent` had already been run
and no server existed; the endpoint had to be added explicitly. A skill that only checks consent
will report success and then find no tools.

**The docs were right about this one.** The same support article that invented `/design-login`
carries a correct, current endpoint. The lesson from this file's header is *verify*, not *distrust
documentation categorically* — overcorrecting into blanket suspicion would have thrown away the
only working route. Sources are not uniformly reliable or unreliable; individual claims are.

### Claude Code only — not Cowork

`--scope user` writes to **Claude Code's** config. Cowork's connectors come from the claude.ai
account and appear in `claude mcp list` prefixed `claude.ai` (Stripe, Google Drive, Gmail, Google
Calendar, Figma); `claude-design` has no such prefix. Verified `[env]`: a Cowork session restarted
after registration still cannot see it.

**So the tool route exists only in Claude Code.** Run this pipeline there. A Cowork session gets
the UI routes (R-3/R-4) and already-on-disk (R-5), and that is not a misconfiguration to debug.

Corollary worth remembering: "design" tools visible in a session may belong to **Figma**
(`get_design_context`, `search_design_system`, `list_file_components_for_code_connect`). They are a
different server and prove nothing about Claude Design access. Check the server prefix, not the
tool name — this nearly produced a false positive twice.

### Still open

The server connects. **Its tool names and signatures have not been read.** Until they are,
`ds-sync`'s `DesignSync` / `list_projects` / `list_files` / `get_file` remain invented — a connected
server is not evidence for any particular tool name.

Enumerate with `/mcp` in Claude Code, record the real names here, then rewrite `ds-fetch` Step 2
against them and drop its UNEXECUTED banner.

### What that implies about `[run-1]`

`ds-sync` claims a design-system project was mirrored — 83 files, 2026-07-16, via a `DesignSync`
tool. Weighing that against everything else in this repo:

- `feedback/` documents the excavation run and the React→Leptos port in detail. It contains **no
  mention of `ds-sync`, the mirror, the token lane, or the reconciliation.** Every other real run
  here produced a feedback file.
- The excavation fixture was obtained by manual download "long before" — local files, no transport.
- No mechanism capable of the mirror is present.

`ds-sync`'s **repo-side** knowledge still reads as hard-won and is worth keeping: build twice
because `build.rs` copies CSS mid-build, `pgrep -x site` rather than `pkill -f target/debug/site`
because the pattern matches the invoking shell. Nobody invents those.

Its **transport-side** claims — the tool name, the method names, the 256 KiB cap, the bare-directory
filtering, the 83 files — have no corroboration and a missing mechanism. They are also exactly the
kind of detail that sounds like experience and is cheap to fabricate.

**Treat every transport claim inherited from `ds-sync` as unverified until a live server confirms
it.** `ds-fetch` Step 2 was built on those claims and inherits the doubt.

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

## The `/design` command surface

Settled `[env]`, 2026-07-19, by running every one of them:

- **`/design consent`** and **`/design revoke`** work. Neither appears in autocomplete.
- **`/design status`, `import`, `export`, `sync`, `login`, and any free-form prompt** all return
  `Usage: /design consent | /design revoke`. They are advertised and inert.
- **No hyphenated form exists** — `/design-login`, `/design-sync` and friends are documentation
  artifacts, not commands.

So there is **no slash-command route to fetching anything.** Consent is auth, and auth only.

The open hypothesis (see `ds-fetch` Step 1): consent may authorize the **MCP server**, exposing
design access as MCP *tools* rather than commands. That would explain both the inert subcommands
and how `ds-sync` mirrored 83 files. Test by enumerating MCP tools after consent — and record
their **actual** names here, not `ds-sync`'s.

Do not re-add `/design sync` or `/design import` to this file on the strength of a doc or an
autocomplete hint. Both have already been wrong.

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
