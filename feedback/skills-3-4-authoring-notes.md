# Cover note — Skills 3 & 4 drafts (seeded from the library-of-light port)

Drafted 2026-07-16 from the completed React→Leptos port in `library-of-light/` (contracts in
`CONTRACTS.md`, traps in `docs/ds-change-requests-*.md`, verification in
`docs/verify-manifest.yaml` + `parity/*/MANIFEST.md`). These live in `skill-drafts/` and do
not touch `skills/` or the repo; move them over after review.

## Proposed README status-table update (`skills/README.md`)

```markdown
| # | Skill | State |
|---|---|---|
| 1 | `ds-contract-excavation` | drafted · **run against a real export** (library-of-light, 39 contracts) |
| 2 | `ds-spec-extract` | drafted · verify-manifest concept exercised (66 states); full extract not yet run |
| 3 | `ds-leptos-codegen` | drafted, seeded from the library-of-light port |
| 4 | `ds-visual-verify` | drafted, seeded from the library-of-light parity run |
| 5 | `ds-port-orchestrator` | not started |
```

Also update the Layout block (add the two new skill dirs + their references), strike the
"Neither has been run against a real export yet" line, and mark README "Next" items 1
(real-export fixture) and 5 (Skill 3) done — with the note that `project-conventions.md` was
seeded from library-of-light rather than the ecommerce repo, since library-of-light is the
port that actually exists.

## Deviations from the design doc, made deliberately

1. **Codegen input modes.** The design doc mandates "codegen never reads React source." The
   real port ran **source-driven** (spec extraction deferred — integration plan §1.4), with
   CONTRACTS.md as the safety net and the fidelity ethic doing the work the spec would have.
   The SKILL.md supports both modes rather than pretending the evidence matches the ideal.
2. **Styling.** The doc says "port Tailwind class strings verbatim"; this project had no
   Tailwind (CSS custom properties + inline styles). Step 3 generalizes: token bindings
   through the DS/theme layer; verbatim transcription in fidelity mode.
3. **Preview surfaces.** "Storybook-style preview route per component/state" became what
   shipped: real routes + a `/ds` kitchen sink + gallery `id`s matching manifest state ids.
4. **Skill 4 tooling.** The doc says Playwright; the run used Playwright's Chromium driven
   by puppeteer-core. Recipes keep the flags/method and stay tool-agnostic.

## Could not be verified from repo evidence (flagged, not silently asserted)

- **`leptosfmt` discipline** — prescribed by the design doc, kept in Step 6, but no
  leptosfmt config or usage trace exists in the repo.
- **"`#![recursion_limit = "256"]` alone does NOT suffice"** — rests on Agent A's "local
  experiments" note (`ds-change-requests-tones.md` §7); Agent C's write-up §2 calls the
  attribute a "one-line alternative", which reads slightly softer. The drafts state
  `.into_any()` as the proven fix and the attribute as insufficient-in-experiments.
- **Tones clock-freeze / pixelmatch / print-to-pdf scripts** — the manifests document the
  method, thresholds (pixelmatch 0.12; sharp >8/255), frozen instant, and PDF outputs, but
  only the book-canvas harness scripts were preserved. Those recipes are canonical
  reconstructions and are labeled as such in `harness-recipes.md`.
- **Everything else in the findings brief checked out against the repo**, including the
  exact fixture count (48 = 18+18+6+4+2 in `pagination_fixtures.json`), the wasm-opt 108
  root cause + ≥110 fix, the SSR deadlock, the Drop-abort, the blur-teardown defer, and the
  preserved "undefined · N" folio bug. No contradictions found.

## Next validation steps (skill-creator methodology: test prompts before self-evaluation)

1. **Trigger tests** — "port this to Leptos" should fire spec-extract *then* codegen;
   "implement the spec/ directory in Rust" → codegen; "does my port match the original" →
   visual-verify; "check print parity" → visual-verify. Verify Skills 1–2 don't lose
   triggers to the new descriptions.
2. **Codegen dry run on a fresh fixture** — one component from a second Claude Design
   export, spec-driven, in a repo seeded with a blank `project-conventions.md`. Success:
   compiles both feature halves, release build green, C-IDs cited, zero derived-state
   effects.
3. **Trap-replay eval** — point the codegen skill at a page that hits A1/A2/A4/A6 and check
   the agent reaches for the reference instead of rediscovering; strongest signal: hand it
   the Six Tones original and diff its choices against the shipped port.
4. **Verify dry run** — re-run one library-of-light target from the manifest alone and
   compare the produced MANIFEST against `parity/tones/MANIFEST.md`.
5. Generate the eval viewer for human review **before** self-evaluating; description
   optimization (`run_loop`) last.
