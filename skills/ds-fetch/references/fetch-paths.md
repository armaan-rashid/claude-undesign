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

### The tool surface — enumerated `[env]` 2026-07-19

22 tools on server `claude-design`. Fully qualified as `mcp__claude-design__<name>`.

**Read — the pipeline uses these:**

| Tool | Use |
|---|---|
| `list_projects` | Enumerate projects. Entry point. |
| `get_project` | Project metadata. Freshness check before mirroring. |
| `list_files` | File listing for a project. |
| `read_file` | Read one file. **Not `get_file`** — that name does not exist. |
| `list_design_systems` | Enumerate design systems. |
| `get_conversation` | **The design chat.** See below — this matters more than it looks. |
| `get_claude_design_prompt` | The prompt behind a design. Also intent. |
| `list_comments` | Inline comments on a design. Intent again. |
| `list_members` | Not needed here. |

**Never call — mutating:**

`write_files` · `delete_files` · `copy_files` · `finalize_plan` · `put_conversation` ·
`create_project` · `create_support_js` · `add_member` · `remove_member` · `update_member_role` ·
`update_sharing` · `ack_comments`

**Unclear, so don't:** `render_preview` may have server-side effects. Not needed for a fetch.

### Trip-wire: the annotations are incoherent — do not filter on them

Read the annotations carefully and they contradict themselves:

- `list_files`, `read_file`, `list_projects`, `get_project` are all tagged **`read-only, destructive`
  simultaneously.** Those are mutually exclusive.
- `finalize_plan`, which commits a plan, is tagged only `open-world` — **not** destructive.

So a skill that allowlists by `read-only` gets nothing usable, and one that blocklists by
`destructive` sails straight into `finalize_plan`. **Decide from the tool's name and semantics,
never from its annotation.** The read/never lists above are curated by hand for exactly this reason.

### How to actually call these

**MCP tools are model-invoked only.** There is no `/mcp call`, no `@server:tool`, no CLI flag. The
`/mcp` → "Tools for claude-design" list is **reference-only** — it says "Enter to select" and
selecting only shows detail. (MCP *prompts* do become `/mcp__server__prompt` slash commands; tools
never do.)

To exercise one, ask in natural language and name it: *"Use the claude-design `list_projects` tool
and show me the raw result."* First call prompts for permission; `allowedTools` can pre-approve
`mcp__claude-design__*` for unattended runs.

Worth noting the TUI's "Enter to select" is itself another advertised-surface-that-isn't — the
fourth of the day. Do not lose time trying to invoke tools from that menu.

### `get_conversation` — the intent source

`ds-spec-extract` has to invent semantic token names when an export carries only raw values, and
its output quality turns on whether it knows *why* a value was chosen. R-3 was preferred partly
because a handoff bundle "carries the chat."

`get_conversation`, `get_claude_design_prompt`, and `list_comments` make that intent reachable
**programmatically, without the UI route at all.** That is a better answer than R-3, and nothing in
this pipeline knew it existed an hour ago. Wire them into `ds-fetch`'s output so extraction gets
intent alongside the file mirror.

### What this settles about `ds-sync`

Five of its six method names are real. That is not invention — someone had genuine exposure to this
server. My earlier conclusion that the transport half was fabricated **was too strong, and is
retracted.**

What stands: `get_file` is wrong (it is `read_file`), so that call would have failed on contact —
which is consistent with the author's own account that the fetch never ran. Real names, unexecuted
code. The remaining `[run-1]` details — the 256 KiB cap, bare-directory entries, the 83-file count —
are still unverified, but they are now better described as **plausible and untested** rather than
invented.

The correction runs both ways: over-trusting a source and over-correcting into blanket suspicion
are the same error with opposite signs. Tag each claim, test what you can, and let the evidence
move individual claims rather than whole files.

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

Its **transport-side** claims split by evidence. The method names turned out **mostly right** once
enumerated (see the tool surface above): `list_projects`, `list_files`, `finalize_plan`,
`write_files`, `delete_files` all exist. `get_file` does not — it is `read_file`, and that single
wrong name would have failed on first contact.

Still untested `[run-1]`: the 256 KiB cap, bare-directory entries in `list_files`, the 83-file
count, the `updatedAt` freshness field. Plausible and unconfirmed — **not** invented. Correct them
on first execution.

Expected file tree for a design-system project `[run-1]`, unverified:

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

Claimed constraints — all `[run-1]`, **none verified**. Test each on first execution:

- `list_files` mixes bare directory entries in with files. Filter by "path is not a prefix of
  another path" before calling `read_file`.
- A size cap around 256 KiB on file reads; `uploads/*` routinely exceeds it.
- Mirroring ~80 files costs roughly 250K tokens of context — delegate to a subagent.
- ~~`list_projects` exposes `updatedAt` for a cheap freshness check.~~ **Disproven `[env]`** — the
  response is `{id, name, url}` only. Try `get_project`.

**Note for the port pipeline:** `_ds_manifest.json` lags `colors_and_type.css`. In the excavation
fixture it was missing an entire token tier (`--amber*`, `--font-terminal`) that the CSS had and
the app used. `ds-spec-extract`'s "prefer the bundle's machine-readable spec" rule must reconcile
against the CSS, and record disagreements rather than trusting the manifest. The CSS wins on
additive diffs.

## R-2 — Claude Design MCP server, app/site project

**Status:** `confirmed` `[env]` 2026-07-19 — `list_projects` is **not** design-system-only.

10 projects returned, mixing design systems with things that read as apps/sites: *Pentacle Finance
Dashboard*, *Tarot Guidebook*, *Human Design Business Site*, *Human Design Tone Guide*, *Tones for
Ticket*, alongside five design systems.

**The port pipeline's step 0 is automatable.** This was the question the whole skill was built to
answer.

### Response shape `[env]`

```json
{ "id": "2035e932-…", "name": "Library of Light — Design System",
  "url": "https://claude.ai/design/p/2035e932-…" }
```

Three fields. Note what is **absent**:

- **No `type` field.** App/site vs design system is inference from the *name*, not something the
  API asserts. A project called "Human Design Business Site" could hold nothing but tokens. To
  establish kind, call `get_project` or `list_files` on it. **Do not branch on name heuristics.**
- **No `updatedAt`.** This kills the freshness check `ds-sync` describes — see the correction below.
- **No pagination signal.** No cursor, total, or `has_more`; the tool takes no parameters and
  returns a bare array. Whether 10 is everything or the first page is **unknown**. If a project you
  expect is missing, suspect truncation before suspecting permissions.

### Confirmed project IDs `[env]`

| Project | ID |
|---|---|
| Library of Light — Design System | `2035e932-f292-4d0a-af54-37ded1de013c` |
| Tarot Guidebook | `12661eb4-6be3-49ce-b247-73301fc06e37` |
| Human Design Tone Guide | `645c05c4-1efd-4101-a173-28d510d64504` |

The first **confirms `ds-sync`'s hardcoded projectId**, previously unverified. The other two are the
excavation fixtures (`tarot guidebook`, `hd-tones` in `known-scaffolds.md`) — so the pipeline can
re-fetch the very export it was validated against, which is the ideal regression fixture.

### Correction: `list_projects` has no `updatedAt`

`ds-sync`'s cheap freshness check reads `list_projects` → compare `updatedAt` to
`.ds-sync-state.json.syncedAt`. **That field does not exist in the response.** Another `[run-1]`
detail that was plausible and wrong. Try `get_project`; if it has no timestamp either, freshness
must come from hashing, and the "cheap no-op" premise for nightly runs needs rethinking.

### Still open

Whether the *kind* distinction is real server-side. `list_design_systems` exists as a separate tool
— comparing its output against `list_projects` settles typing by set difference, without relying on
names. Run it.

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
