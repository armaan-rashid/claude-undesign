# SKILL-FEEDBACK — ds-contract-excavation, first real run

Fixture: the two Claude Design exports under `library-of-light/subwebsites/`. The skill
worked — the sweep categories mapped cleanly onto what was actually there, and the
"exhaustiveness, not insight" framing produced the right output. Friction points, in
rough priority order:

1. **No guidance for multi-export repos.** SKILL.md assumes one export ⇒ one CONTRACTS.md
   at repo root. This repo has two exports sharing contracts (DS CSS, Supabase project,
   localStorage conventions). The run brief had to invent the policy (one ID sequence,
   per-export sections plus a shared section). Suggest a short paragraph: number one
   sequence across the repo, group by export, add a shared section, and state which
   export a contract's evidence lives in.

2. **Output location is rigid.** "A single CONTRACTS.md at the repo root" collides with
   read-only reference trees (here `subwebsites/` is gitignored reference material and
   the excavation had to write elsewhere). Suggest: default to repo root, but allow the
   caller to designate an output dir; what must stay fixed is that `ds-spec-extract` can
   find it.

3. **The inventory bucket set is incomplete.** Two real cases had no bucket: (a) docs
   (`README.md`, `SETUP.md`, `DC-NOTES.md`, the DS brand bible — the brand README is
   *load-bearing* for copy rules, not cruft); (b) vendored third-party runtimes
   (`vendor/react*.js`) — not really `glue` (they're not scaffold-specific) and calling
   them `asset` hides them. Suggest adding `doc` and `vendor` buckets, or explicitly
   blessing "unclassifiable → finding" for docs (which is what I did, but it reads as a
   workaround).

4. **G-1 (orphan exports via import graph) misses the no-module-system case.** Both
   exports use script-tag globals; there are zero imports to graph. The real orphan
   detector here was `Object.assign(window, {…})` names with no in-repo reference — that
   found the biggest contract in the repo (the missing `Guidebook.html` host). Suggest
   G-1 gain a second signature: "global registration with no in-repo reader," grep
   `Object.assign(window|window\.[A-Z][A-Za-z]+ *=`.

5. **Granularity guidance is thin.** "Enumerate, don't summarize" pushes toward
   micro-contracts; with no counter-pressure a run like this could emit 150 one-liners
   or 8 blobs. I converged on ~40 by merging things that share one producer and one
   decision (e.g. all dc-runtime DSL tags = one contract; the GuidebookSync interface =
   one contract with its semantics inside). A sentence like "one contract per
   producer/decision seam; list co-owned details inside it" would make runs consistent.

6. **Decisions-on-record have no slot.** The user had pre-decided five classifications.
   The schema only has shim/redesign/drop/TBD-user, so I wrote "_(decision on record)_"
   into Rationale ad hoc. A `Decided-by:` field (inferred | user | project-doc + citation)
   would make the human-review pass faster and keep agents from silently overriding user
   decisions on later runs.

7. **Step 5 (grow the catalog) vs. this run's reality.** The skill says to edit
   `references/known-scaffolds.md` in place; the brief redirected the new section to a
   proposal file. Fine, but the skill should say what to do when the catalog and the
   excavation disagree mid-run — the `likely` Claude Design probes were 2-for-4 wrong
   (no `window.storage`, localStorage very much available), and the "delete wrong
   `likely` entries" instruction lives in known-scaffolds.md, not SKILL.md. A pointer in
   SKILL.md ("disposition every `likely` probe explicitly: confirm, or delete with a
   note") would have made that step self-executing.

8. **Evidence lines in generated bundles.** Much of the dc-runtime evidence is in
   `support.js`, a generated esbuild bundle — line numbers there are stable only until a
   rebuild (which can't happen here, since the source isn't in the repo, but the skill
   should note: cite generated files freely when they're checked in and frozen; prefer
   the source file when both exist, e.g. `editor.jsx` over `editor.js`).

9. **Small wins worth keeping:** the known-scaffolds "do not treat `likely` as fact"
   warning directly prevented a wasted `window.storage` hunt; the "contracts live in the
   glue" anti-pattern pointed straight at `cloud-sync.js` and the HTML entry points,
   which held ~half the contracts; the file:line requirement caught a real one — the
   stale `_ds_manifest.json` would have been recorded as "manifest describes the tokens"
   without the line-level diff against the CSS.

10. **One cross-skill note:** `ds-spec-extract` says "prefer the bundle's
    machine-readable spec over re-deriving from source." This fixture's machine-readable
    DS manifest is *stale* (missing the amber/terminal token tier the app uses). The
    excavation now flags it (C-37), but spec-extract's instruction should soften to
    "prefer the bundle *after* reconciling against the CSS; source wins on additive
    drift."
