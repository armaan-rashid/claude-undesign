# Project conventions — seed file

This file is **per-project state**. At the start of a port, copy it into the target repo (or
into the skill invocation context), keep every "Rule" line, and replace every "This project"
block with the target's values. The rules are what the skill enforces; the instance values
are what a fresh agent needs to not re-derive the repo. The first concrete instance below is
**library-of-light** (the Claude Design → Leptos port completed 2026-07-16).

## 0. Identity

| Field | This project |
|---|---|
| Repo | `library-of-light` |
| Target framework | Leptos 0.7 (SSR via Axum + WASM hydrate), cargo-leptos 0.3.x |
| Rust toolchain | pinned `1.94.1` + `wasm32-unknown-unknown` (`rust-toolchain.toml`) |
| wasm-bindgen | pinned `=0.2.118` (workspace) |
| Environment requirement | binaryen/wasm-opt **≥ 110** on PATH, or none (wasm-opt 108 corrupts release wasm — react-to-leptos.md A10) |
| Contracts | `CONTRACTS.md` at repo root, C-01…C-39 |
| Verify manifest | `docs/verify-manifest.yaml` |

## 1. Workspace layout

**Rule:** one crate owns the design system; one crate owns the site; pages are self-contained
modules under `pages/<page>/` that never edit shared files.

This project:

```
Cargo.toml                  # workspace + [[workspace.metadata.leptos]] config
crates/lol_ds/              # design system: canonical CSS + structural components
  css/colors_and_type.css   #   tokens — BYTE-COPIED from the export's _ds bundle; never hand-edit
  css/system.css            #   system motion (lol-fade, lol-breathe) + component chrome
  src/components/           #   Frame, TopRule, BottomRule, StatusPill, Terminal, Grain,
                            #   Glyph, Button, EditableText, SyncDot (+ SyncStatus enum)
crates/site/
  src/app.rs                # shell + route table (FROZEN in phase 2)
  src/sync/                 # engine.rs, rows.rs, server_fns.rs, postgrest.rs (FROZEN)
  src/pages/<page>/         # one module per route; mod.rs + model/logic submodules
  public/css/<page>.css     # page-scoped CSS (see §3)
  style/main.css            # site sheet; page-CSS @import slots at top
subwebsites/                # the original exports — READ-ONLY reference material
docs/                       # integration plan, verify manifest, ds-change-requests-*.md
```

Routes: `/` home, `/ds` kitchen sink, `/tones`, `/tarot`, `/tarot/book`, `/tarot/canvas`.

## 2. Design-system reuse

**Rule:** pages never redraw system chrome (frames, rules, pills, dots, editable text) by
hand — they compose the DS crate. Page-level overrides of DS internals are allowed only as
scoped CSS, and every override is a filed change request (see §6).

This project:

- Import surface: `lol_ds::{Frame, TopRule, BottomRule, StatusPill, Terminal, TermCursor,
  Grain, Glyph, Button, ButtonVariant, EditableText, SyncDot, SyncStatus, strip_html}`.
- `/ds` is the living contract: every component, token swatch, and type class on one page
  with a dark/light toggle. A DS change request's effect must be visible there.
- Corner-mark chrome (`PageCorners`/`PlateCorners` in the tarot book) reuses
  `lol_ds::Frame` with `hairline=false` + `inset`/`inset_x`/`mark_size`/`color` — wrapped in
  a pointer-events-none absolute div so DOM paint order matches the original's bare spans.
- Precedent for "close but not identical" DS components: the canvas needed a chrome-less
  editable and shipped a ~60-line page-local `DcEditable` instead of forcing a prop into
  `EditableText` (`ds-change-requests-book-canvas.md` §3). Local copy + filed request beats
  speculative DS API.

## 3. Page CSS mechanism

**Rule:** one CSS file per page, scoped under a `.pg-<page>` root class; activation is a
single pre-allocated line the page owner uncomments — no shared-file merge conflicts.

This project (`crates/site/style/main.css` header documents it):

- cargo-leptos 0.3.x does **not** bundle CSS `@import`s, so DS stylesheets ship as separate
  assets: `crates/site/public/css/{colors_and_type,system}.css` (copies of
  `crates/lol_ds/css/`, the single source of truth) loaded by `<Stylesheet>` tags in
  `app.rs` **before** the site sheet. Do not add relative `@import`s to `main.css`.
- Page files: `crates/site/public/css/<page>.css` (`tones.css`, `tarot.css`,
  `tarot-book.css`, `tarot-canvas.css`), activated by `@import "/css/<page>.css";` slots
  pre-created at the top of `main.css` — these resolve in the *browser* against the copied
  asset.
- Page markup wraps in the scope class (`<Grain class="pg-tones">`, `.pg-tarot-book`), and
  every page rule is written under it. DS overrides look like:
  `.pg-tones .tones-status .lol-sync-dot { width: 6px; height: 6px; }`.
- Host-HTML CSS from the original (print rules, contenteditable affordances) moves into the
  page file with class pairings intact — print CSS ↔ emitted class names is a contract
  (C-30).

## 4. Sync wiring pattern

**Rule:** all remote data behind typed `#[server]` fns; a shared client engine owns
paint-first / merge / debounce / poll / focused-guard; pages provide only a thin backend
adapter and preserve the originals' storage shapes byte-identically.

This project (`crates/site/src/sync/`):

- `SyncBackend` trait: row type, `STORAGE_PREFIX` (byte-identical to the original page —
  `lol-hd-tones:`, `lol-tarot:`), `decode_local`/`encode_local` (the *original's* value
  JSON, so pre-port browser data still loads), and `load`/`save` futures over the server fns.
- `SyncEngine<B>` behavior: localStorage paint first → one remote reconcile → 800 ms
  debounced write queue (blur flushes immediately) → failed flush re-queues + retries in 4 s
  → 5 s poll (skipped while `document.hidden`) → `focused_key` guard: the poll never
  overwrites the row being edited nor rows with unflushed local edits.
- Server surface: exactly four fns (`load_tones`, `save_tone`, `load_cards`, `save_card`)
  with fixed endpoints under `/api/`; the backend (`postgrest.rs` today, Axum+SQLite later)
  swaps behind them without pages noticing.
- `Remote<T>` envelope: `Data(T) | Offline`. `Offline` = "not configured", a *normal* state
  pages present as Local mode — never an error.
- Field-level merge (tarot): partial `CardRow` upserts omit `None` fields so two editors on
  different fields never clobber (`pages/tarot/store.rs` layers a dirty-field set over the
  frozen engine).
- Page-private localStorage keys (`_last`, `_lastCard`) start with `_` — the engine skips
  them; pages own them directly.

## 5. Credentials / config

**Rule:** secrets in a gitignored config file with a committed `.example`; env vars
override; absence degrades gracefully to local-only. The browser never sees a key.

This project: `supabase.toml` (gitignored) / `supabase.toml.example` (committed, points at
where the original exports keep the values); `SUPABASE_URL`/`SUPABASE_KEY` env override;
`Postgrest::from_env()` returning `None` ⇒ every server fn answers `Remote::Offline` ⇒ pages
show "Local" status and keep working against localStorage. Use publishable keys only.

## 6. Multi-agent protocol: frozen files + change requests

**Rule:** shared files are frozen while page agents run in parallel worktrees. Workarounds
are local; every workaround is filed as a change request naming its own location so the
supervisor can delete it when the request lands.

This project: frozen set = `lol_ds/*`, `crates/site/Cargo.toml`, `src/app.rs`, `src/lib.rs`,
`src/sync/*` (integration plan §5). Change requests: `docs/ds-change-requests-<page>.md`,
one numbered entry per item with **Request** and **Workaround** fields (see the three
existing files for the format). Branches: one per package (`agent-tarot`,
`agent-book-canvas`), merged by the supervisor; merge order tokens/DS first, then pages.

## 7. Contract citations

**Rule:** every implementation of a `CONTRACTS.md` shim/redesign/drop decision cites its
C-ID in a comment at the site of implementation; module docs list the contract IDs the
module discharges.

This project: `grep -rn "C-[0-9]" crates/site/src/pages` hits every contract-bearing module
(the tones page header alone discharges C-01…C-06, C-08…C-15). The pattern:

```rust
// C-11 (shim, decision on record): keep writing the original's
// `_last` key on every select. Nothing reads it — the write is
// kept for byte-level parity (and a future "resume where I was").
```

Tests that pin contract surfaces name them too
(`storage_keys_are_byte_identical_to_the_original`, the 78-key roster test per C-22).

## 8. Build & test commands

**Rule:** check both feature halves; test native; build release early; format `view!` code.

This project:

```sh
cargo check -p site --no-default-features --features ssr
cargo check -p site --no-default-features --features hydrate
cargo test  -p site                     # 31 tests at merge: fixtures, shapes, geometry
cargo leptos build --release            # requires wasm-opt ≥110 or none on PATH
cargo leptos watch                      # dev server on 127.0.0.1:3030
leptosfmt crates/site/src crates/lol_ds/src
```

Copy/text written into the UI follows the DS brand README (in the export's `_ds` bundle):
no emoji, no exclamation marks, the export's ID casing.
