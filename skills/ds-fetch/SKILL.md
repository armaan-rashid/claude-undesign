---
name: ds-fetch
description: >
  Fetch a Claude Design project or handoff bundle onto disk as a faithful, hash-manifested
  mirror, ready for the port pipeline. Use whenever the user wants to pull, download, fetch, or
  sync source from Claude Design, says "get the export", "pull the design", "grab the bundle",
  or starts a port with no export already on disk. Runs before ds-contract-excavation.
  Read-only against Claude Design — this pipeline never writes back.
---

# Fetch from Claude Design

Step 0 of the port pipeline. Gets bytes onto disk faithfully and tells you exactly what landed,
so `ds-contract-excavation` has something to excavate.

This skill does **one** job: mirror + manifest. It does not diff, interpret, or apply anything.
`ds-sync` layers change-detection on top of it; the port pipeline consumes its output directly.
Keeping it dumb is what lets both callers share it.

**Read-only.** The never-call list is in `references/fetch-paths.md` and is curated by hand —
**the server's own tool annotations are self-contradictory and cannot be used for this.** If a task
seems to need a write back to Claude Design, stop and ask; this pipeline is one-way by design.

## Output

```
<dest>/
  source/            # verbatim mirror, original paths preserved
  intent/            # design chat, prompt, comments — why, not what
  FETCH.json         # machine-readable manifest (hashes, skips, provenance)
  FETCH.md           # same thing, human-readable
```

`intent/` is not decoration. `ds-spec-extract` must invent semantic token names wherever the export
carries only raw values, and intent is the difference between a name that is right and one that is
merely plausible. Fetch it whenever the route allows.

`<dest>` is caller-supplied. Defaults: `design-export/<project-slug>/` for the port pipeline,
`design-system/` when `ds-sync` calls it (that path is already established in that repo).

`FETCH.json` shape:

```json
{
  "projectId": "…",
  "projectName": "…",
  "fetchedAt": "2026-07-19T15:00:00Z",
  "route": "mcp-tool",
  "capabilityProbe": { "listsAppProjects": true, "evidence": "…" },
  "files": { "components/core/Button.jsx": "sha256:…" },
  "skipped": [ { "path": "uploads/hero.png", "reason": "binary; over file-read size cap" } ]
}
```

**Emit a sha256 per file.** This is the seam that makes the skill reusable: `ds-sync`'s entire
diff reduces to comparing two `FETCH.json` files, and the port pipeline gets provenance for free.
A mirror without hashes forces every caller to re-walk the tree.

## Step 0 — Connect and consent

**Verify the command surface in the environment before relying on it. The published docs are
stale here, and have already sent this skill down a wrong path once.**

### Advertised surface is not working surface

Autocomplete offers this:

```
/design [sync|login|import|export|status|<prompt>]
```

**Only two of those do anything.** Verified by running them, 2026-07-19:

| Command | Actual behavior |
|---|---|
| `/design consent` | **Works.** Grants access. Not advertised in autocomplete. |
| `/design revoke` | **Works.** Withdraws access. Not advertised either. |
| `/design status` | Prints `Usage: /design consent \| /design revoke` |
| `/design import` | Same usage message |
| `/design export` | Same usage message |
| `/design sync` | Same usage message |
| `/design login` | Same usage message |
| `/design <any prompt>` | Same usage message — even though the hint claims to take a prompt |
| `/design-login` and every other hyphenated form | Does not exist at all |

So the two commands that work are the two the hint omits, and the six it advertises are inert.

**This is the single most important thing in this skill.** Three separate authoritative-looking
sources each described a Claude Design interface that does not behave as described:

1. Anthropic's support article documents `/design-login`. It does not exist.
2. `ds-sync` documents a `DesignSync` tool. No tool by that name exists — though five of its six
   *method* names turned out real, so the source was neither reliable nor worthless.
3. Autocomplete advertises six subcommands. Five are inert and it omits the two real ones.
4. The server's own tool annotations mark `list_files` both `read-only` and `destructive`, and
   `finalize_plan` as neither.

The rule that survives all three: **an advertised surface is a claim, not evidence.** Docs,
existing skill files, and autocomplete hints are all leads. Running the thing is evidence. This is
the same distinction `ds-contract-excavation` exists to draw — what code appears to depend on
versus what is actually there — applied to our own tooling.

Re-verify this table by running the commands, not by reading anything. Update it with a date when
behavior moves.

Note also that the docs were **right** about the MCP endpoint (Step 1) while wrong about
`/design-login`. The rule is *verify*, not *distrust documentation categorically* — blanket
suspicion would have discarded the only working route. Sources are not uniformly reliable; claims
are.

### Flow

Full setup is in Step 1 — `mcp add`, then `/design consent`, then restart. There is **no working
status check** (`status` is inert), so the only signal that consent worked is whether the tools
subsequently function.

Headless and scheduled runs have no interactive session, so lapsed consent fails there and only
there — a fetch that works interactively and fails nightly is almost always this. Report it as a
consent problem, not a fetch problem.

**Do not "correct" this section back to what any documentation says.** That regression is one
search away, and the docs will still look authoritative when it happens.

## Step 1 — Find the route that actually works

Read `references/fetch-paths.md` for the routes and their verification status.

**No slash command fetches anything** — `import`, `export`, and `sync` are all inert. The mechanism
is not a command. It is MCP tools.

### The route is MCP tools

`/design consent` authorizes the **MCP server**, which exposes design access as MCP *tools*, not
slash commands. Verified `[env]` 2026-07-19: 22 tools on server `claude-design`.

Setup, in order — each step is separate and skipping one fails silently:

```sh
claude mcp add --scope user --transport http claude-design https://api.anthropic.com/v1/design/mcp
```

then `/design consent`, then **restart Claude Code**.

Three distinct trip-wires, each of which looks like the others:

- **Consent is not registration.** Consent can be granted with no server registered. Check both.
- **`claude mcp list` reads config; `/mcp` reads the running session.** `list` will report
  `✔ Connected` while the current session has no such tools, because it loaded before the server
  was added. A skill that verifies via `mcp list` and concludes availability will find nothing at
  runtime. Restart, then confirm with `/mcp`.
- **Claude Code only.** `--scope user` writes Claude Code's config. Cowork sees claude.ai account
  connectors, not this. A Cowork session cannot use this route, restart or no restart.

Go to Step 2 once `/mcp` lists `claude-design`.

### Fallbacks, when the tools are unavailable

Neither needs the MCP server, so both work when it is missing, unregistered, or the session is
Cowork:

- **Export from Claude Design, then normalize** (R-3 / R-4). The user clicks Export — "Handoff to
  Claude Code → Send to local coding agent", or "Download as .zip". Step 2b takes it from there.
- **Already on disk** (R-5). How the pipeline's one real excavation fixture was actually obtained,
  and the only route that never breaks.

The tool route used to be preferred purely on convenience. It now also wins on **intent**:
`get_conversation`, `get_claude_design_prompt`, and `list_comments` retrieve the design chat and
rationale programmatically — the thing a bare file mirror loses, and the thing that decides whether
`ds-spec-extract`'s invented token names are right or merely plausible.

Where the tools are unavailable, say step 0 is manual and mean it. That beats a skill that claims
automation and fails at the seam.

### The one probe left

`list_projects` exists and is generic. **What it returns has not been observed.** Call it, then
record, in writing, into `FETCH.json.capabilityProbe`:

> Does `list_projects` return app/site projects, or only design-system projects?

The port pipeline consumes app/site projects; every verified fact so far concerns a design-system
project. Note also `list_design_systems` as a separate tool — that the two are distinct hints the
server may treat them as different kinds, which is exactly what this probe is asking about.

- **App projects listed** → route `mcp-tool`. Continue to Step 2.
- **Only design systems** → the app export needs a UI route. Tell the user which, then Step 2b.

Either way, **write the finding into `references/fetch-paths.md`** and move R-2 off `unknown`. An
answer learned and not recorded is an answer paid for twice — the same catalog discipline
`known-scaffolds.md` runs on.

## Step 2 — Mirror (tool route)

**Tool names verified `[env]` 2026-07-19; the mirror itself has still never executed.** Names are
solid. Behavioral details — the size cap, what `list_files` returns — are untested.

Server `claude-design`, tools fully qualified `mcp__claude-design__<name>`. Full catalog and the
never-call list in `references/fetch-paths.md`. Read it before calling anything.

**The annotations lie.** `list_files` and `read_file` are tagged `read-only, destructive` at once;
`finalize_plan` — which writes — is not tagged destructive at all. Never decide safety from an
annotation here. Use the curated lists in the reference.

1. **`list_projects`** → find the target. If several match, ask; do not guess.
2. **`get_project`** → metadata. Compare against prior state before mirroring; a cheap no-op beats
   a needless 250K-token walk.
3. **`list_files`** → the file listing. `[run-1]`, untested: entries may include bare directories
   mixed with files. Filter to paths that are not a prefix of another path, and treat a failed read
   on an extensionless path as confirmation rather than an error.
4. **`read_file`** each kept path — **not `get_file`, which does not exist.** Write **verbatim**
   into `<dest>/source/<path>`. No reformatting, no line-ending normalization. This is a mirror;
   editing bytes here corrupts the excavation's file:line evidence downstream.
5. **Skip `uploads/*` and `.thumbnail`** `[run-1]`, untested: binary reference material, and a size
   cap around 256 KiB is claimed. Record every skip with its reason — a silent skip becomes a
   phantom missing dependency in `CONTRACTS.md`. **If the real cap differs, correct the reference.**
6. **Hash each file** (sha256) as written.
7. **Also fetch intent**, which a plain file mirror misses entirely: `get_conversation`,
   `get_claude_design_prompt`, `list_comments`. Land them under `<dest>/intent/` and list them in
   `FETCH.json`. `ds-spec-extract` invents semantic token names from raw values, and this is the
   only thing that tells it *why* a value was chosen.
8. **Delegate to a subagent when interactive** — a large mirror burns context the parent needs for
   the actual port. Re-verify a hash sample afterwards; a subagent reporting success is not the
   same as bytes on disk.

On first real execution, correct every `[run-1]` detail above against observed behavior and re-tag
it `[env]`.

## Step 2b — Normalize (UI route)

When the export arrived via a UI route, the user has a folder or archive somewhere. The skill's
job shifts from fetching to **verifying and normalizing**:

1. Locate it (ask for the path; do not guess).
2. Copy — do not move — into `<dest>/source/`. The user's download stays untouched, so a botched
   normalize is recoverable.
3. Hash everything and build the same `FETCH.json`, with `route` set to the UI route used.
4. **Flag what a UI export includes that a tool mirror would not**: a handoff bundle carries a
   README and the design chat alongside the design files. Those are not noise — the README states
   how the generator intends the design to be interpreted, and it is the closest thing to design
   intent the pipeline ever gets. Record them in the manifest and point `ds-spec-extract` at them.
5. **Flag what it might be missing**: UI exports can omit files that only exist server-side. If
   the excavation later finds a reference to something absent, suspect the export boundary before
   suspecting a broken contract.

## Step 3 — Report and hand off

Report: file count, total bytes, skip list with reasons, the capability probe result, and the
`<dest>` path.

Then name the next step explicitly — `ds-contract-excavation` against `<dest>/source/`. Do not
run it automatically. The excavation is a whole-repo pass with its own review gate, and silently
chaining into it hides the boundary between "we have the bytes" and "we have understood them."

**Do not interpret the contents here.** Not a token, not a component, not a manifest. The
temptation is strong when you have just read every file — resist it. The interpretation skills
have review gates and evidence rules this one does not, and a fetch that also editorializes is a
fetch nobody can trust as a baseline.

## Anti-patterns

- **Editing bytes in transit.** Reformatting, re-indenting, or "fixing" a file breaks every
  `file:line` citation the excavation will produce. Verbatim means verbatim.
- **Silent skips.** Every skipped file is in the manifest with a reason, or the excavation invents
  a missing dependency to explain the gap.
- **Trusting a subagent's report.** Re-verify a hash sample. The failure mode is a subagent that
  says "mirrored 83 files" having written 40.
- **Mirroring into the repo proper.** `<dest>` is a staging area. Nothing here lands in `crates/`,
  `src/`, or anywhere the target build sees — that is `ds-sync`'s job, behind its lanes and gates.
- **Guessing the project.** If `list_projects` returns several and the target is ambiguous, ask.
  Mirroring the wrong project wastes the context budget and produces a confidently wrong baseline.
- **Leaving the probe unrecorded.** The whole point of Step 1 is that it stops being a question.
