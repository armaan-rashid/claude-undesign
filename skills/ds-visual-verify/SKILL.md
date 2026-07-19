---
name: ds-visual-verify
description: >
  Adversarially verify a ported UI against its reference implementation via screenshot,
  print-PDF, fixture, and behavior comparison. Use after any design port or codegen pass,
  whenever the user asks "does the port match," wants pixel/behavior fidelity checks, or
  mentions visual regression between two implementations of the same design. Drives both
  sides from a verification manifest and routes every finding to its cause.
---

# Visual Verify

You are the adversary. The codegen agent believes the port matches; your job is to make the
strongest possible case that it does not, and to document precisely what survives that
attack. Verification that only confirms is worthless — a MATCH verdict is only credible if
the rig could have caught a failure.

Two calibration facts from the first real run (library-of-light, 66 manifest states):

1. **Pixel-near-identity is achievable and should be demanded.** With identical fallback
   fonts, frozen clocks, and identical viewports, most states diffed **0 px**; the worst
   accepted residual was ~2% single-channel jitter, and every non-zero residual was
   explained (sub-glyph antialiasing, caret, backdrop-blur sampling). "Design fidelity, not
   pixel identity" governs *font rendering* differences — it is not license for layout
   drift. An unexplained 40 px cluster is a finding.
2. **Every verdict names its evidence.** State → files → diff number → verdict, in a
   manifest a human can re-derive. "Looks the same" is not a verdict.

Recipes (commands, harness code, flags) live in `references/harness-recipes.md`. Read it
before building the rig.

## Inputs

- The verification manifest (`spec/verify.yaml` or `docs/verify-manifest.yaml`): states,
  setup steps, breakpoints, `contracts:` citations, `output: pdf` markers, `behavior-`
  states.
- `CONTRACTS.md` — findings cite contract IDs; decisions on record separate accepted deltas
  from bugs.
- The port (built and served) and the original export (files on disk).
- The codegen package's known-deltas list (`docs/ds-change-requests-*.md`) — these are
  pre-accepted divergences; verify they hold, don't re-report them.

## Step 1 — Build both sides

**Port:** serve the real build (`cargo leptos serve` / the release binary on localhost).
Run the screenshot pass on debug if release is slow, but re-verify interaction on the
**release** build at least once — release-only corruption is real (wasm-opt A10 was caught
exactly this way).

**Reference — three escalating cases, all used in this port:**

1. *Self-contained export:* open via `file://` or a static server (a server is required the
   moment the original `fetch()`es anything, e.g. sidecar state files).
2. *Export with CDN dependencies, sandboxed network:* byte-copy the original into a scratch
   dir and satisfy the CDN loads locally — the tones harness loaded the sibling export's
   vendored React UMD before `support.js` (its loader skips the CDN when `window.React`
   exists) and duplicated the dynamically-injected CSS `<link>` statically. **Modify the
   copy, never the original; document every harness edit in the manifest header** so a
   reviewer can separate harness artifacts from port deltas.
3. *Orphaned components with a lost host* (contract-excavation finding): build a scratch
   reference host in `/tmp` — precompile the JSX with esbuild, load vendored React UMD +
   the canonical token CSS, render each component into a `.state`-classed div whose `id` is
   the manifest state id. Composition fidelity is unverifiable (the host is lost — say so);
   *component* fidelity is fully verifiable. Recipe in harness-recipes.md §3.

## Step 2 — Make both sides deterministic

Same viewport sizes (this port: 1440×900 desktop, 900×700 compact, device scale 1), same
browser binary, `document.fonts.ready` awaited plus a settle delay, scrollbars hidden.
**Freeze the clock on both sides** (Date override init script — the tones pass froze both to
the same instant so the footer clock diffs 0 px). Seed or clear localStorage per the
manifest's `setup:` lines. Disable or stall live data identically on both sides; states that
*need* live data are run separately and labeled.

## Step 3 — Screenshot pass

For each manifest state: apply setup, drive the UI to the state (clicks/hover/keyboard via
the harness — capture *settled* hover states), screenshot each breakpoint on both sides,
pixel-diff identical-size frames. Record: state id, file pair, diff (px count or channel-%),
verdict. Element-level screenshots (per `.state` div) beat viewport screenshots for
component galleries; viewport screenshots for full pages.

## Step 4 — Verdicts and findings

Verdict vocabulary from the real manifests (`parity/*/MANIFEST.md`):

| Verdict | Meaning |
|---|---|
| `MATCH` / `PASS` | Pair compared; residual zero or explained (name the explanation) |
| `SOURCE-VERIFIED` | Reference cannot reach the state in this environment; port screenshot + transcription of the original source (labels/colors/logic) + a unit test stand in |
| `PORT-VERIFIED` | Same, where the port-side evidence is a live screenshot |
| `PROOF` | Port-only evidence of a capability (e.g. real remote rows rendering) |
| finding | Anything else — file it, don't average it away |

Every non-MATCH difference becomes a structured finding:

```json
{ "component": "...", "state": "...", "breakpoint": "...", "description": "...",
  "severity": "high|medium|low",
  "suspected_cause": "spec-gap | codegen-bug | token-mismatch | acceptable-divergence",
  "contracts": ["C-XX"] }
```

Routing: `spec-gap` → back to ds-spec-extract; `codegen-bug` → back to ds-leptos-codegen;
`token-mismatch` → the tokens/DS layer; `acceptable-divergence` → requires a decision
citation (a CONTRACTS.md decision or known-deltas entry) — without one it is not acceptable,
it is unreviewed.

Adversarial discipline: magnify residual-diff crops before accepting them as antialiasing;
check that a 0 px diff is not two identically-broken screenshots (blank page diffs 0 px —
eyeball every pair once); verify *both* sides changed when you drove a state transition.

## Step 5 — Behavior pass

Scripted interaction replay with **logged asserts** (`PASS — <claim>` lines into a
behavior-log kept next to the screenshots). Assert against the spec/manifest description —
the original implementation is the visual reference, not the behavioral oracle; where the
original is also scriptable, run the same asserts on both sides and report pairs.

The classes of assert that caught real issues here:

- **Copy equality:** button labels after toggles, confirm-dialog text, export filenames and
  payload shapes, alert copy — character-identical.
- **Guards:** with the editor focused, a remote change to the open row must NOT replace the
  content; unfocused, it must (mock the server fns to inject the remote change).
- **Persistence round-trips:** type → reload → content persists (localStorage paint);
  state-shape asserted **verbatim** against the contract shape (the canvas pass string-compared
  the persisted JSON against the C-29 shape, `\x1f` separators and all).
- **Two-client convergence:** an edit in session A appears in session B within one poll
  interval; field-level merge means concurrent edits to different fields never clobber.
- **Numeric invariants:** the canvas pass compared *byte-identical* transform strings after
  identical wheel-step scripts (`translate3d(427.27px, 202.958px, 0px) scale(0.40657)`)
  and identical clamp values at both zoom limits. When both sides compute, compare the
  computation, not the pixels.
- **Console hygiene:** capture console errors and pageerrors on every run; hydration
  mismatches and panics are findings even when pixels match.

## Step 6 — Fixture capture for pure functions

When the port reimplements pure logic (pagination, formatting, geometry): transcribe the
original JS functions **verbatim** into an HTML harness, execute in headless Chromium over
curated edge-case inputs, dump the input→output pairs as a base64 JSON fixture file, freeze
it into the port's source tree, and have `cargo test` assert byte-identical outputs. The
tarot pagination shipped 48 fixture cases this way
(`crates/site/src/pages/tarot/pagination_fixtures.json`: 18 bodyToBlocks, 18 blockToWords,
6 wordsToHtml, 4 pageWordsToBlocks, 2 full roundtrips), and the DOM-dependent half was closed
end-to-end: the same seeded body paginates to the same page count and same last-page words
in the same browser. Recipe in harness-recipes.md §6.

## Step 7 — Print pass

For `output: pdf` states: `--print-to-pdf` on both sides (headless Chromium), then compare
**page count, page box dimensions, and break behavior** — including negatives (no trailing
blank page, no toolbar/sidebar artifacts, no editing tints). This port asserted 500×700 px
`@page` boxes emerge as 375.12 pt and print-all breaks after every page except the last.
Flag emulation caveats instead of chasing them: headless print emulation collapsed the
screen stage to one page *identically on both sides* — recorded as MATCH with a
real-browser spot-check deferred.

## Step 8 — Sandbox caveats: flag, never chase

Environment limits must be **declared in the manifest header** and excluded from judgment —
not silently absorbed, not endlessly debugged:

- **Google Fonts do not render under TLS-intercepting proxies.** Both sides then render
  identical fallback stacks: compare structure/spacing/color, **never glyph shapes**, and
  defer every font-metric judgment to a real-browser pass (list it as remaining work).
  Re-measure-on-`fonts.ready` behavior still has to exist in the port; it just can't be
  exercised here.
- **Remote-store states the original cannot reach in a sandbox** (its CDN/supabase calls
  never settle) get `SOURCE-VERIFIED`/`PORT-VERIFIED` instead of a fake pair.
- **Glyphs outside the font stack** (the U+1F70x alchemical suit glyphs) tofu identically on
  both sides on Linux — compare anyway, note the caveat, defer to the target platform.
- **Live-data hygiene:** write-path tests against a real store use throwaway keys outside
  the roster, and the manifest **names them for deletion** (`_agent_test`, `TEST-AGENT`).

## Step 9 — Outputs

```
parity/<target>/MANIFEST.md    # header: harness description, caveats; table: state → files → diff → verdict;
                               # behavior section; fixture methodology; known deviations
parity/<target>/*.png|pdf      # ref-*/port-* (or *-orig/*-port), plus sxs-* side-by-sides and diff-* masks
parity/<target>/behavior-log.txt
verify/findings.json           # structured findings (Step 4), empty array if clean
verify/REPORT.md               # human summary: verdict counts, findings by cause, remaining work
```

The MANIFEST header states the whole rig: reference construction and any harness edits,
capture tool + browser + viewports, clock freeze value, diff metric + threshold, font
caveat. A verdict table nobody can reproduce is an opinion.

## Anti-patterns

- **Confirming instead of attacking.** If every state is MATCH and no finding, no caveat,
  and no deferred item appears anywhere, the pass proved the rig runs — not that the port
  matches.
- **Chasing sandbox artifacts.** Hours spent making Google Fonts load through a TLS proxy
  buy nothing; the flag-and-defer protocol exists so environmental noise doesn't consume
  the budget for real comparison.
- **Screenshotting transitions.** Capture settled states; mid-animation frames diff
  nondeterministically. (Deliberate mid-*gesture* states — a drag in progress — are fine
  when both sides are scripted to the identical pointer position.)
- **Treating the original as the behavioral oracle.** The spec/manifest is truth; the
  original is the visual reference. Where they disagree, that is a finding for the spec,
  not a port bug.
- **Unexplained residuals.** "18 px, probably antialiasing" without a magnified crop is a
  guess. Either explain it or file it.
- **Verifying only the debug build.** At least one full interactive pass on the release
  artifact.
