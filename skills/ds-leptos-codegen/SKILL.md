---
name: ds-leptos-codegen
description: >
  Generate idiomatic Leptos (Rust) components from a framework-agnostic spec directory
  (tokens.json + component specs + state machines) or, for fidelity ports, directly from an
  audited React/AI-generated export. Use whenever the user wants to port React or AI-generated
  UI to Leptos, says "port to Leptos" or "React to Leptos", mentions Leptos, or asks to
  implement components from a spec/ directory in Rust. Translates reactivity semantics
  (signals, memos, effects), not syntax.
---

# Leptos Codegen

> **Runs in Claude Code, not Cowork.**
>
> This skill needs a Rust toolchain — `cargo check`, `leptosfmt` — against a working copy of the target repo.
>
> **If you are reading this in Cowork:** do not improvise a workaround. Cowork cannot reach
> Claude Code's MCP servers or the user's toolchain, and its sandbox is a separate filesystem from
> the user's machine. Stop and tell the user to run this in Claude Code, in the target repo.
>
> Detect it cheaply: if `mcp__claude-design__*` tools are absent when this skill needs them, or the
> repo's build tooling is missing, you are in the wrong client. Say so plainly rather than
> producing a degraded result that looks like a real one.


You are translating **semantics, not syntax**. React re-renders functions; Leptos runs each
component body once and updates through fine-grained signals. Most `useEffect`s exist only to
patch over React's render model and translate to *nothing*. The failures that matter are not
in the view macro — they are in ownership (who fetches, who holds state), in the server/WASM
boundary, and in a handful of Leptos-specific traps that pass `cargo check` and die at
runtime or in release builds. The references carry those traps; read them before writing code.

## Inputs

1. **`CONTRACTS.md`** from `ds-contract-excavation` — **required**. If it does not exist,
   stop and run that skill first. Every shim or redesign you implement cites its contract ID
   (`C-XX`) in a code comment at the implementation site. This is not decoration: when a
   verify diff fails, the C-ID routes the finding back to its seam.
2. **The spec directory** from `ds-spec-extract`, when it exists. Spec-driven is the default
   mode: read the component's `.spec.yaml` + `tokens.json`, not the React source. Any
   fallback to source is a logged spec gap.
3. **Source-driven fidelity mode.** When no spec exists (a decision on record — the
   library-of-light port ran this way, integration plan §1.4), port directly from the export
   source with `CONTRACTS.md` as the safety net and `docs/verify-manifest.yaml` (or
   `spec/verify.yaml`) as the state enumeration. In this mode the original source is the
   spec: transcribe copy, constants, and inline styles verbatim — see the fidelity ethic
   below.
4. **`references/project-conventions.md`** — the target repo's layout, DS crate, sync
   pattern, CSS mechanism, credential handling. Read it first; seed a new copy per project.

## The fidelity ethic

When the brief is a port (not a redesign), **fidelity is the bar**:

- **Copy is sacred.** Strings, glyphs, alert text, filenames, confirm-dialog copy transfer
  byte-for-byte. `parity/tones/behavior-log.txt` asserts import-confirm copy character-equal.
- **Constants are sacred.** Debounce windows, dimensions, quality factors, clamps
  (`MAX_DIM` 1200, webp q 0.85, `S_MAX` 5 in the image-slot port) transfer exactly, with a
  comment naming the original line.
- **Bugs are preserved and flagged.** The tarot port renders the JS `undefined` literally in
  court continuation folios because the original does
  (`crates/site/src/pages/tarot/spread.rs:157` — "preserved bug-for-bug"). Fix-vs-preserve is
  a user decision; preserve by default and file the bug separately.
- **Storage shapes are frozen.** localStorage prefixes, value JSON shapes, and remote row
  keys are a compatibility surface with live data. Unit-test them
  (`storage_keys_are_byte_identical_to_the_original` in the tones port).
- **Deviations are recorded, never silent.** Every accepted delta (poll replacing realtime,
  800 ms vs 650 ms debounce) goes in a "Known deltas" list the verify pass will read.

## Workflow

### Step 0 — Orient

Read `CONTRACTS.md` end to end, then `references/project-conventions.md`, then the brief's
frozen-file list. In a multi-agent port, shared files (DS crate, sync layer, root manifests,
`app.rs`) are frozen; you work around them locally and file each workaround as a change
request (`docs/ds-change-requests-<page>.md`) naming the workaround so a supervisor can
delete it when the request lands. Do not touch frozen files, even for one-line fixes.

### Step 1 — State model first

Before any `view!`, write the page's state as Rust:

- Each independent state dimension → its own signal. Do not port a React state *object* as
  one signal; split it (`Six Tones`' `{view, tone, bodyEmpty, senseFirst, now}` became five
  independent signals plus a `Screen` enum).
- Finite states → an enum, one signal per parallel region. Data-carrying transitions → enum
  variants with fields (the image-slot's `Drag` enum carries the reframe geometry).
- Derived values → `Memo` or plain closures over signals — **never** an effect writing a
  signal.
- Pure logic (pagination, geometry, formats) → plain functions in a `model.rs`/`pagination.rs`
  module with unit tests against captured fixtures. Keep DOM out of them so tests run native.

### Step 2 — Translate the view

Apply `references/react-to-leptos.md` — the table, then the worked examples. Summary:

| React / export construct | Leptos 0.7 |
|---|---|
| `useState` / `this.state` + `setState` | `RwSignal` / `signal()` |
| derived value, `useMemo` | `Memo::new` (or a closure) |
| `useEffect` (most) | **nothing** — fine-grained reactivity subsumes it |
| `useEffect` (genuine external-world sync) | `Effect::new` (client-only by nature) |
| `componentDidMount` / mount work | `Effect::new` (first run) |
| `componentWillUnmount` / cleanup | `on_cleanup` (or a Drop guard — see anti-patterns) |
| `createContext` / `useContext` | `provide_context` / `use_context` |
| prop callbacks (`onCommit`) | `Callback<T>` props |
| conditional render | `<Show>` or `match` in the view |
| lists | `<For>` keyed; static rosters may `.map().collect_view()` |
| refs | `NodeRef` (never read from `Drop` — see anti-patterns) |
| state machines | enum + signal per parallel region; transitions as methods |

### Step 3 — Styling

- Bind token-derived values through the DS: `var(--token)` strings and DS classes, never
  re-derived literals. In spec-driven mode, bind through the `@theme` mapping from Skill 2.
- In fidelity mode, transcribe inline styles **verbatim** (`pages.rs` copies `pages.jsx`
  property-for-property) so computed styles match; move host-HTML CSS (print rules,
  contenteditable affordances) into the page's scoped CSS file keeping every class pairing
  intact.
- Page CSS is scoped under a `.pg-<page>` root class via the project's page-CSS mechanism
  (see project-conventions). Page overrides of DS components are allowed but each one is a
  filed DS change request.

### Step 4 — Server boundary

All remote data goes behind `#[server]` functions — the component never fetches.

- The page-visible remote surface is a small set of typed server fns (the whole
  library-of-light store is four: `load_tones`/`save_tone`/`load_cards`/`save_card`,
  `crates/site/src/sync/server_fns.rs`). Fixed endpoints (`/api/...`) so they stay curl-able.
- Server-only deps (`reqwest`, `chrono`, `axum`) are `optional = true`, activated by the
  `ssr` feature only — the WASM bundle must not grow.
- **Graceful degradation is a contract**: missing config ⇒ the server fn returns an explicit
  `Offline` envelope variant (not an error), and the page falls back to local-only mode with
  its own status presentation. Credentials live in a gitignored config file with a committed
  `.example`, env vars overriding; the browser never sees a key.

### Step 5 — Preview surface

Every component/state the verify manifest names must be reachable:

- Routes for pages; a kitchen-sink route (`/ds` in library-of-light) exercising every DS
  component and token, including states the pages don't use (light theme).
- Give preview/gallery DOM nodes `id`s equal to the verify-manifest state ids
  (`tarot_book/mod.rs` `Exhibit` does exactly this) — the screenshot harness selects by them.
- States that need data: make them reachable by seeding localStorage or env, not by code
  edits. The verify pass scripts against what ships.

### Step 6 — Check discipline

After every component, before marking anything done:

```sh
cargo check -p <crate> --features ssr --no-default-features   # server half
cargo check -p <crate> --features hydrate --no-default-features  # WASM half
cargo test -p <crate>                                          # fixtures + shape tests
leptosfmt <files>                                              # view! formatting
```

Both feature halves, always — `#[server]` bodies and client-only code fail on opposite sides.

### Step 7 — Release build EARLY, not last

`cargo leptos build --release` compiles a different world than dev and fails in ways dev
never shows (type-layout overflow, wasm-opt corruption — see anti-patterns A1 and A10).
Run it as soon as the first page renders, then keep it green. Re-verify interactive behavior
on the release binary once per package (the book-canvas port caught a broken-on-every-route
release wasm this way).

### Step 8 — Record

Per package, emit:

- `docs/ds-change-requests-<page>.md` — every local workaround of a frozen file, each entry
  naming the workaround location so it can be deleted when the request lands.
- Known-deltas list (in the change-requests doc or the parity manifest) — every accepted
  behavioral deviation with its decision citation.
- C-ID citations in code at every shim/redesign site (grep `C-[0-9]` should hit every
  contract-bearing module).

Then hand off to `ds-visual-verify`.

## Release-build and platform gotchas (headlines)

Full symptom → cause → fix catalog in `references/react-to-leptos.md` §Anti-patterns. The
ones that cost this port real time:

1. **Release SSR overflows rustc's type-layout depth** ("queries overflow the depth limit!").
   cargo-leptos passes `--cfg erase_components` in dev only. Fix: `.into_any()` at component
   tails and large `view!` branches. `#![recursion_limit = "256"]` alone did **not** suffice.
2. **`view!` drops camelCase SVG attributes** (`viewBox` renders literally). Emit static SVG
   via `inner_html`.
3. **`StoredValue::new_local` during SSR deadlocks the Axum render.** Create local-arena
   values inside a client-only `Effect` — never in the component body.
4. **Reading a `NodeRef` from `Drop` during owner disposal aborts the WASM instance.** Cache
   raw `web_sys` handles at mount; unmount paths touch only the cache.
5. **wasm-opt ≤ 108 corrupts the release wasm** (`Table.grow` RangeError at
   `__wbindgen_start` on every route). Require binaryen ≥ 110 on PATH, or remove it so
   cargo-leptos downloads its own.

## Anti-patterns

- **Porting syntax.** A Leptos component with one big state-struct signal and effects
  mirroring every `useEffect` is a React program in Rust clothes. Delete the effects; split
  the state.
- **Effect-writes-signal for derived data.** That is a hand-rolled, glitch-prone `Memo`.
  If a signal's value is a function of other signals, it is a `Memo` or a closure.
- **Fetching from components.** Every remote read/write goes through the `#[server]` surface;
  ownership lives in the sync/engine layer, exactly as `CONTRACTS.md` classified it.
- **"Fixing" the original silently.** Behavior deltas — even improvements — are decisions on
  record or they are bugs. The verify pass diffs against the original, not your taste.
- **Deferring the release build.** Two of this port's five worst traps only exist in release.
- **Editing frozen files because it's faster.** It merges as a conflict for three parallel
  agents. Work around locally, file the change request.
