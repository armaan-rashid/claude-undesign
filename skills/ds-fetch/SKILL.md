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

> **Runs in Claude Code, not Cowork.**
>
> This skill needs the `claude-design` MCP server, which is registered with `claude mcp add --scope user` — **Claude Code's config, which Cowork does not read**.
>
> **If you are reading this in Cowork:** do not improvise a workaround. Cowork cannot reach
> Claude Code's MCP servers or the user's toolchain, and its sandbox is a separate filesystem from
> the user's machine. Stop and tell the user to run this in Claude Code, in the target repo.
>
> Detect it cheaply: if `mcp__claude-design__*` tools are absent when this skill needs them, or the
> repo's build tooling is missing, you are in the wrong client. Say so plainly rather than
> producing a degraded result that looks like a real one.


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

### The command surface differs by client

**`/design-sync` and `/design-login` exist in Claude Code. They do not exist in Cowork.** Same
feature, different surface per client — which means "does this command exist?" has no
environment-independent answer.

**In Cowork** `[env]` 2026-07-19 — autocomplete offers
`/design [sync|login|import|export|status|<prompt>]`, and five of six are inert:

| Command | Behavior in Cowork |
|---|---|
| `/design consent` | **Works.** Grants access. Not in autocomplete. |
| `/design revoke` | **Works.** Withdraws access. Not in autocomplete either. |
| `/design status` · `import` · `export` · `sync` · `login` · any prompt | All print `Usage: /design consent \| /design revoke` |
| `/design-login`, `/design-sync` | Not available in this environment |

**In Claude Code** — `/design-sync` and `/design-login` exist, as the support docs describe.
Behavior not yet catalogued here; see `references/fetch-paths.md`.

### What the confusion actually taught

Four authoritative-looking sources described interfaces that did not behave as described *in the
environment where they were tested*:

1. `ds-sync` names a `DesignSync` tool. No tool by that name exists — though five of its six
   *method* names are real, so the source was neither reliable nor worthless.
2. Cowork's `/design` autocomplete advertises six subcommands; five are inert and it omits the two
   that work.
3. The server's tool annotations mark `list_files` both `read-only` and `destructive`, and
   `finalize_plan` as neither.
4. **This skill** previously asserted `/design-login` "does not exist," generalizing from Cowork to
   everywhere. The docs were right; the test was run in the wrong client.

Two rules, and the second was learned the hard way:

- **An advertised surface is a claim, not evidence.** Docs, skill files, and autocomplete hints are
  leads. Running the thing is evidence.
- **A negative result is scoped to where you ran it.** "Command X does not exist" is only ever
  "does not exist *in client Y*." Record the client with every observation — an unscoped negative
  is how correct documentation gets overwritten with a local quirk.

Re-verify by running commands, and write down **which client** you ran them in.

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

### Resolving the target

`list_projects` returns **both** design systems and app/site projects — settled `[env]` 2026-07-19.
Step 0 is automatable; go to Step 2.

Two things to handle when picking a project:

- **There is no reliable type signal.** Records are `{id, name, url}` with no `type`; names are
  convention, not contract; and `list_design_systems` **omits projects created as design systems**,
  so set-difference does not work either (all three verified `[env]`). Determine kind from
  **contents** via `get_project` / `list_files`: tokens CSS + `_ds_manifest.json` +
  `components/core/*` is a design system; page sources and a host HTML are an app.
- **No pagination signal.** Bare array, no cursor or total. If an expected project is absent,
  suspect truncation before permissions.

Ambiguous match → ask. Mirroring the wrong project burns the context budget and yields a
confidently wrong baseline. Known IDs are in `references/fetch-paths.md`, including both excavation
fixtures — useful as regression targets.

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
