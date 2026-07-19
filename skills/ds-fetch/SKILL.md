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

**Read-only.** Never call `finalize_plan`, `write_files`, or `delete_files`. If a task seems to
need a write back to Claude Design, stop and ask — this pipeline is one-way by design.

## Output

```
<dest>/
  source/            # verbatim mirror, original paths preserved
  FETCH.json         # machine-readable manifest (hashes, skips, provenance)
  FETCH.md           # same thing, human-readable
```

`<dest>` is caller-supplied. Defaults: `design-export/<project-slug>/` for the port pipeline,
`design-system/` when `ds-sync` calls it (that path is already established in that repo).

`FETCH.json` shape:

```json
{
  "projectId": "…",
  "projectName": "…",
  "fetchedAt": "2026-07-19T15:00:00Z",
  "route": "designsync-tool",
  "capabilityProbe": { "listsAppProjects": true, "evidence": "…" },
  "files": { "components/core/Button.jsx": "sha256:…" },
  "skipped": [ { "path": "uploads/hero.png", "reason": "binary; over get_file cap" } ]
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
2. `ds-sync` documents a `DesignSync` tool with named read methods. Never verified; see Step 1.
3. Autocomplete advertises six subcommands. Five are inert and it omits the two real ones.

The rule that survives all three: **an advertised surface is a claim, not evidence.** Docs,
existing skill files, and autocomplete hints are all leads. Running the thing is evidence. This is
the same distinction `ds-contract-excavation` exists to draw — what code appears to depend on
versus what is actually there — applied to our own tooling.

Re-verify this table by running the commands, not by reading anything. Update it with a date when
behavior moves.

### Flow

`/design consent`, and confirm it actually completed. There is no working status check — `status`
is one of the inert subcommands, so the only signal that consent worked is whether design access
subsequently functions.

If `/design` does not respond at all, the MCP server may need adding. Documented as
`claude mcp add --scope user --transport http claude-design https://api.anthropic.com/v1/design/mcp`
— `[docs]`, unverified, from the article that invented `/design-login`. Treat accordingly.

Headless and scheduled runs have no interactive session, so lapsed consent fails there and only
there — a fetch that works interactively and fails nightly is almost always this. Report it as a
consent problem, not a fetch problem.

**Do not "correct" this section back to what any documentation says.** That regression is one
search away, and the docs will still look authoritative when it happens.

## Step 1 — Find the route that actually works

Read `references/fetch-paths.md` for the routes and their verification status.

**No slash command fetches anything.** `import`, `export`, and `sync` are inert. So the mechanism,
if one exists, is not a command — which leaves one hypothesis worth testing and two routes that
work today regardless.

### The hypothesis to test

`/design consent` plausibly authorizes the **MCP server**, which would expose design access as MCP
*tools* rather than slash commands. That would reconcile everything: the inert subcommands, and
`ds-sync` having somehow mirrored 83 files through something it calls `DesignSync`.

Test it in one step: **after consent, enumerate the available MCP tools and look for design ones.**

- **Design tools present** → that is the route. **Record their real names and signatures** in
  `fetch-paths.md`. They may or may not match `ds-sync`'s `DesignSync` / `list_projects` /
  `list_files` / `get_file`. Do not assume they do — that naming has never been verified against a
  live server, and assuming is what produced the last three errors.
- **No design tools** → there is no programmatic fetch. Say so plainly, mark R-1/R-2 `unavailable`
  in `fetch-paths.md`, and use a UI route. Do not go hunting for an undocumented API.

### The routes that work regardless

Neither depends on anything unverified, so prefer them until the hypothesis is settled:

- **Export from Claude Design, then normalize** (R-3 / R-4). The user clicks Export — "Handoff to
  Claude Code → Send to local coding agent", or "Download as .zip". Skill takes it from there via
  Step 2b. The handoff bundle additionally carries the README and design chat, which is the only
  place design *intent* survives — worth preferring on those grounds alone.
- **Already on disk** (R-5). How the pipeline's one real excavation fixture was actually obtained.

Being honest that step 0 may be irreducibly manual is better than a skill that pretends to
automate it and fails at the seam.

Call `list_projects`. Then answer, **in writing, into `FETCH.json.capabilityProbe`**:

> Does `list_projects` return app/site projects, or only design-system projects?

**This is an open question in this pipeline, not a rhetorical one.** It is confirmed that
`DesignSync` reaches design-system projects — `ds-sync` run 1 mirrored 83 files from one. Whether
the same API reaches app/site projects (the kind the port pipeline actually consumes) has never
been tested. The first real run of this skill settles it.

- **App projects listed** → route `designsync-tool`. Continue to Step 2.
- **Only DS projects listed** → the app export must come from a UI route (handoff bundle,
  save-as-folder, HTML export). See `references/fetch-paths.md`, tell the user which route to use,
  and switch to Step 2b to normalize what they produce.

Either way, **write the finding into `references/fetch-paths.md`** and promote that route's status
from `unknown` to `confirmed` or `unavailable`. Do not leave the next run to rediscover it. This
is the same catalog discipline `known-scaffolds.md` runs on, for the same reason: an answer
learned and not recorded is an answer paid for twice.

## Step 2 — Mirror (tool route) — UNEXECUTED

**This step has never run.** It was written from `ds-sync`'s Phase 1, which was itself written as
though a run had happened when it had not (confirmed with the author, 2026-07-19). No Claude Design
MCP server is registered, so nothing here has ever touched a live server.

An earlier revision of this file said these trip-wires were "confirmed from ds-sync run 1... not
speculative." That was false, and it is the clearest example in this project of the failure this
skill now warns about: an unverified claim in one file laundered into a confirmed claim in another
by citing a run that did not happen.

Keep the steps — they are a **reasonable design** for a mirror, and reasonable is worth something.
Just do not treat any specific number or method name below as observed. On first real execution,
correct them against what the server actually does and re-tag them `[env]`.

1. **`list_files` returns bare directories mixed in with files.** Keep only real files: paths
   that are not a prefix of another path. Directory entries have no extension and fail `get_file`.
2. **`get_file` each kept path.** Write **verbatim** into `<dest>/source/<path>`, preserving the
   original path. No reformatting, no normalization of line endings, no prettifying. This is a
   mirror; anything that edits bytes here corrupts the excavation's file:line evidence downstream.
3. **Skip `uploads/*` and `.thumbnail`.** Binary reference material, and `uploads/*` regularly
   exceeds the 256 KiB `get_file` cap. Record every skip with its reason — a silent skip becomes a
   phantom missing dependency in `CONTRACTS.md`.
4. **Anything else over the 256 KiB cap** — record as skipped with `reason: "over get_file cap"`.
   Do not attempt to chunk or reconstruct it.
5. **Hash each file** (sha256) as you write it.
6. **Delegate the mirror to a subagent when running interactively.** ~80 files burns roughly
   250K tokens of context, and the parent session needs that context for the actual port work.
   Afterwards, independently re-verify a sample of hashes — a subagent reporting success is not
   the same as bytes being on disk.

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
