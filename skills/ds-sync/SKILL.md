---
name: ds-sync
description: >
  Pull the Library of Light design system from Claude Design and land changes in this Leptos
  repo through two lanes: tokens/assets apply automatically, component-source changes become a
  gated reconciliation proposal for human review. Use whenever the user says "sync the design
  system", "pull from Claude Design", "is the design system up to date", or a scheduled
  nightly run fires. Read-only against Claude Design — this pipeline never writes back.
---

# Design-System Sync (Claude Design → Leptos)

One-way sync. Source of truth for the *design* is the Claude Design project; source of truth
for the *Rust rendering* is this repo. The committed mirror in `design-system/` and the hash
state in `.ds-sync-state.json` connect the two.

Project: `Library of Light — Design System`, projectId `2035e932-f292-4d0a-af54-37ded1de013c`.
Access via the `DesignSync` tool (read methods only: `list_projects`, `list_files`,
`get_file`). NEVER call `finalize_plan` / `write_files` / `delete_files` here.

**The mirror itself lives in `ds-fetch`.** That skill owns auth, `list_files` filtering, the
`get_file` cap, skip rules, verbatim writes, and hashing — all of it learned on run 1 of this
skill. This skill owns everything downstream: freshness, diff, the two lanes, screenshots,
commits. Do not reimplement the mirror here; fix it in `ds-fetch` and both callers get the fix.

## Cheap freshness check

`list_projects` → compare the project's `updatedAt` to `.ds-sync-state.json.syncedAt`. If it
hasn't moved, report "in sync" and stop. (Scheduled runs must be a cheap no-op most nights.)

## Phase 0 — BEFORE screenshots

Before touching anything: build if stale, serve, screenshot `/ds /tones /tarot /tarot/book`
at 1440×900 dark. These anchor the visual diff. (Check binary freshness with `find -newer`
before rebuilding; don't `pkill -f target/debug/site` — the pattern matches the invoking
shell; use `pgrep -x site`.)

## Phase 1 — Mirror + diff

1. **Follow the `ds-fetch` skill**, with `dest` = repo root so the mirror lands in
   `design-system/` as before. It handles bare-directory filtering, the 256 KiB `get_file` cap,
   the `uploads/*` and `.thumbnail` skips, verbatim writes, subagent delegation for the ~250K
   token mirror, and the hash sample re-verification. It returns `FETCH.json` with a sha256 per
   file and a reasoned skip list.
2. **Diff `FETCH.json.files` against `.ds-sync-state.json.files`.** Both are `{path: "sha256:…"}`,
   so this is a map comparison — no re-walking the tree. The changed-set drives everything below.
3. Rewrite the state file: `{projectId, syncedAt, files, skipped}`, carrying `ds-fetch`'s skip
   list through verbatim. A file that moved from mirrored to skipped is a **change**, not a
   deletion — check before treating an absence as upstream removal.
4. Commit the mirror + state: `ds-sync: mirror <date>`.

## Phase 2 — Token lane (auto-apply)

Trigger: `colors_and_type.css` or `assets/*` in the changed-set.

1. **Safety diff first.** Extract every `--var:` definition from the old and new css. For any
   variable removed upstream but still used anywhere in `crates/` — add a compat shim block
   at the end of the copied file (`:root { --removed-var: <old value>; }`) and flag it in the
   report. Name-set-identical diffs (values only) need no shim.
2. Overwrite `crates/lol_ds/css/colors_and_type.css` with the mirror copy (+ shim).
3. Changed `assets/*.svg`: favicons compare-and-update in `crates/site/public/`; brand/sigil
   SVGs land in `crates/site/public/assets/`.
4. `cargo leptos build` **twice** (build.rs copies css during the build; the site-root
   assembly of the first build predates the copy). `cargo test` — token *values* must not be
   asserted anywhere; fix any test that does and note it.
5. AFTER screenshots of the same four routes. Apply the rubric: **missing-var symptoms
   (invisible text, unstyled block, blank page) → revert the lane, report FAILED**;
   **cosmetic drift (overflow, spacing) → keep, file as an open question**.
6. Commit: `ds-sync: token lane <date>`.

## Phase 3 — Component lane (gated — never auto-apply)

Trigger: `components/core/*` (or `_ds_manifest.json` component entries) in the changed-set.

1. Read the changed component sources + `.d.ts` contracts.
2. Follow the `ds-leptos-codegen` skill (conventions + anti-pattern catalog) to draft revised
   or new `lol_ds` component sources — into
   `docs/ds-reconciliation-<date>/{REPORT.md, proposed/}`, typechecked in a scratch crate,
   **not** wired into `lib.rs`, no page or `crates/lol_ds/src` modification.
3. REPORT.md: per component — matches / diverges-how / new, severity, whether any current
   page visibly depends on the divergence, and the decision list for the user.
4. Commit the proposal; **notify and stop**. A human merges after review (repo policy:
   tokens auto, components gated — decided 2026-07-16).

## Phase 4 — Ship

Interactive session: report lanes + screenshots, surface open questions.
Scheduled (headless) run: commit to a `ds-sync/<date>` branch, push, notify with the summary
(push notification on: token lane applied / proposal awaiting review / lane FAILED; silent
no-op otherwise). Prerequisites for headless runs: the GitHub remote cloneable from the run
environment, and a standing Claude Design authorization (`/design-login`) since there is no
interactive claude.ai session.

## Known trip-wires (learned on run 1, 2026-07-16)

- `preview/*` changes are reference-only: no code action, but they are the *expected look* —
  cite them in verify judgments.
- `ui_kits/library/*` is a future page, not part of the site: mirror it, act on nothing.
- `_ds_bundle.js` / `_adherence.oxlintrc.json` are generated artifacts: mirror-only.
- Run 1 baseline: 83 files, token lane applied cleanly (variable name-sets identical), 8
  component proposals awaiting review in `docs/ds-reconciliation-2026-07-16/`.

The transport-level trip-wires from run 1 — bare directory entries in `list_files`, the 256 KiB
`get_file` cap, `uploads/*` skipping, subagent delegation, hash re-verification — now live in
`ds-fetch` and are enforced there. They are not repeated here on purpose: two copies drift, and
the copy that drifts is always the one nobody ran.
