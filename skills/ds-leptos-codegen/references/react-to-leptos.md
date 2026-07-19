# React → Leptos: semantic translation table, worked examples, anti-patterns

Every example below is real: the "before" is from a Claude Design export in
`subwebsites/` of the library-of-light repo, the "after" is the shipped Leptos 0.7 port in
`crates/`. File paths cite that repo. Treat the paths as evidence, not dependencies — the
patterns stand alone.

## 1. The translation table

| # | React / export construct | Leptos 0.7 translation | Notes |
|---|---|---|---|
| T1 | `useState` / class `this.state` + `setState` | `RwSignal::new` / `signal()` | One signal **per independent dimension**, never one state-object signal |
| T2 | derived value, `useMemo` | `Memo::new` or a plain closure | Never an effect that writes a signal |
| T3 | `useEffect` (sync state → state) | **nothing** | Fine-grained reactivity subsumes it; deleting is the correct port |
| T4 | `useEffect` / `componentDidMount` (external world: timers, DOM, network kick-off) | `Effect::new` | Effects run client-only — this is also the SSR escape hatch |
| T5 | `componentWillUnmount` / effect cleanup | `on_cleanup` | `on_cleanup` demands `Send + Sync`; for JS handles see A5 |
| T6 | `createContext` / `useContext` | `provide_context` / `use_context` | Context value = a small cloneable API struct over signals |
| T7 | prop callbacks (`onCommit`, `onSave`) | `Callback<T>` props, `cb.run(v)` | |
| T8 | conditional render (`cond && <X/>`, ternary, `<sc-if>`) | `<Show when=…>` or `match` in the view | `match` arms of different types need `.into_any()` |
| T9 | lists (`.map`, `<sc-for>`) | `<For each=… key=…>` | Static compile-time rosters may `.map().collect_view()` |
| T10 | refs (`useRef`, `ref={{ setEditorRef }}`) | `NodeRef` | **Never read from `Drop`** — see A4 |
| T11 | implicit state machine (string unions, boolean soup) | enum + one signal per parallel region | Data-carrying transitions = enum variants with fields |
| T12 | fetch / storage wrappers (`window.GuidebookSync`, supabase-js) | `#[server]` fns behind a sync-engine layer | Ownership per `CONTRACTS.md`; component code never fetches |
| T13 | inline `style={{ … }}` objects | `style="…"` strings, transcribed verbatim | Fidelity ports copy property-for-property |
| T14 | `dangerouslySetInnerHTML` | `inner_html=` | Also the workaround for static SVG (A2) |
| T15 | `document.execCommand` bold/italic/insertText | same, via `web_sys::HtmlDocument::exec_command…` | Deprecated but Chrome-baseline (C-38); keep for fidelity |

## 2. Worked examples

### E1 — State object → independent signals + an enum

Before (`Six Tones.dc.html:1029`, a dc-runtime class):

```js
class Component extends DCLogic {
  state = { view: "atlas", tone: 1, bodyEmpty: true, data: {}, now: "", sync: "init", senseFirst: false };
```

After (`crates/site/src/pages/tones/mod.rs`):

```rust
#[derive(Clone, Copy, PartialEq, Eq)]
enum Screen { Atlas, Editor }

let screen = RwSignal::new(Screen::Atlas);
let tone = RwSignal::new(ToneKey::Tone(1));
let sense_first = RwSignal::new(false);
let now = RwSignal::new(String::new());
// data / sync-status / pending live on the shared SyncEngine, not the page:
let eng = SyncEngine::new(TonesBackend);
```

Three moves in one: string unions become enums (`"atlas"|"editor"` → `Screen`;
`tone: number | "sound"` → `ToneKey::{Tone(u8), Sound}` with a `storage_key()` that stays
byte-identical to the original keys), independent dimensions become independent signals, and
ownership that `CONTRACTS.md` classified as redesign (sync) moves out of the page entirely.

### E2 — Hand-maintained derived state → `Memo`

Before (`Six Tones.dc.html`, `onInput`): the original recomputes and conditionally stores
`bodyEmpty` on every keystroke:

```js
const empty = !stripHtml(html);
if (empty !== this.state.bodyEmpty) this.setState({ bodyEmpty: empty });
```

After (`tones/mod.rs`):

```rust
let body_empty = Memo::new(move |_| !has_text(&tone.get().storage_key()));
```

The manual dirty-check disappears; `Memo` only notifies on change. This is the T2/T3 rule in
one line — the `setState` bookkeeping was React's problem, not the design's.

### E3 — Lifecycle + interval → `Effect` + `on_cleanup`

Before (`Six Tones.dc.html:1032-1046`):

```js
componentDidMount() {
  /* paint from localStorage … */
  this._clock = setInterval(() => this.setState({ now: this.fmt() }), 30000);
  this.initSync();
}
componentWillUnmount() { clearInterval(this._clock); clearTimeout(this._syncTimer); }
```

After (`tones/mod.rs`):

```rust
Effect::new(move |_| engine.with_value(|e| e.start()));   // client-only: touches localStorage

Effect::new(move |_| {
    now.set(fmt_time());
    if let Ok(handle) = set_interval_with_handle(
        move || now.set(fmt_time()),
        Duration::from_millis(30_000),
    ) {
        on_cleanup(move || handle.clear());
    }
});
```

`Effect::new` is both the mount hook and the SSR guard: effects never run during server
rendering, so anything touching `window`/`document`/localStorage belongs inside one.

### E4 — Conditional → `<Show>`

Before (`Six Tones.dc.html:829`):

```html
<sc-if value="{{ bodyEmpty }}">
  <div style="position:absolute; inset:0; pointer-events:none; …">
    Begin the reading. Write as much as you like — the page scrolls. …
  </div>
</sc-if>
```

After (`tones/mod.rs`):

```rust
<Show when=move || body_empty.get()>
    <div class="tones-placeholder">
        "Begin the reading. Write as much as you like — the page scrolls. ⌘B for bold, ⌘I for italics. Saved to this browser as you type."
    </div>
</Show>
```

Copy transfers byte-for-byte (entities decoded: `&#8984;` → `⌘`).

### E5 — Lists: keyed `<For>` vs static `collect_view`

Dynamic, reorderable list (`design-canvas.jsx` artboard rows → `tarot_canvas/canvas.rs:683`):

```rust
<For
    each=move || meta.get().slot_ids
    key=|k| k.clone()
    children=move |k: String| { /* find + render artboard */ }
/>
```

Static compile-time roster (`<sc-for list="{{ tones }}">` with exactly six tones →
`tones/mod.rs`):

```rust
{node_positions().into_iter().map(|p| { /* … */ view! { <div class="tone-node" …/> } }).collect_view()}
```

Rule: if membership or order can change at runtime, `<For>` with a stable key. If the list is
a constant, `.map().collect_view()` is simpler and avoids keying overhead.

### E6 — Context → `provide_context` with an API struct

Before (`design-canvas.jsx:110`): `const DCCtx = React.createContext(null);` carrying the
canvas's mutation API down to nested artboards.

After (`tarot_canvas/canvas.rs:140-166`):

```rust
#[derive(Clone, Copy)]
struct DcApi { state: RwSignal<CanvasState>, focus: RwSignal<Option<String>> }
impl DcApi {
    fn patch_section(&self, id: &str, patch: impl FnOnce(&mut SectionState)) { /* … */ }
    fn set_focus(&self, v: Option<String>) { self.focus.set(v); }
}
// in the root component body:
provide_context(api);
// in any descendant:
let api = use_context::<DcApi>().expect("inside DesignCanvas");
```

Signals are `Copy` handles, so the context value is a cheap `Clone`/`Copy` struct of them
plus methods — the Leptos equivalent of a context holding `{state, dispatch}`.

### E7 — Prop callbacks → `Callback<T>`

Before (`editor.jsx:121`):

```jsx
onBlur={(e) => { const v = html ? e.currentTarget.innerHTML : e.currentTarget.textContent; onCommit(v); }}
```

After (`crates/lol_ds/src/components/editable.rs`):

```rust
#[component]
pub fn EditableText(
    #[prop(into)] value: Signal<String>,
    #[prop(into)] on_commit: Callback<String>,
    /* … */
) -> impl IntoView {
    let commit_now = move || { if let Some(v) = read_content() { on_commit.run(v); } };
```

`#[prop(into)]` lets callers pass closures; `Callback` is the clonable, `'static` shape a
`view!` handler needs.

### E8 — Inline styles verbatim (fidelity mode)

Before (`pages.jsx:9-21`):

```jsx
function Page({ children, style }) {
  return (
    <div className="grain" style={{ width: PAGE.width, height: PAGE.height,
      background: "var(--bg)", color: "var(--fg)", position: "relative",
      overflow: "hidden", fontFamily: "var(--font-display)", ...style }}>{children}</div>
  );
}
```

After (`tarot_book/pages.rs`):

```rust
#[component]
pub fn Page(#[prop(optional)] style: &'static str, children: Children) -> impl IntoView {
    view! {
        <div class="grain" style=format!(
            "width:500px;height:700px;background:var(--bg);color:var(--fg);\
             position:relative;overflow:hidden;font-family:var(--font-display);{style}"
        )>
            {children()}
        </div>
    }
}
```

The `...style` spread becomes a trailing `{style}` append — same override order. The
book-canvas parity run diffed 0.000–0.7% per state on this transcription discipline.

### E9 — Fetch/sync ownership → engine + `#[server]` fns

Before (`Six Tones.dc.html:1080-1106`): the component class owns a debounced queue against
supabase-js in the browser:

```js
queueSync(key, body) {
  if (!this.sb) return;
  this._pending[String(key)] = body;
  this.setState({ sync: "saving" });
  clearTimeout(this._syncTimer);
  this._syncTimer = setTimeout(() => this.flushSync(), 650);
}
async flushSync() { /* upsert; on error re-queue + retry in 4000ms */ }
```

After: the page calls a shared engine (`crates/site/src/sync/engine.rs`) —

```rust
let write_body = move |html: String| {
    engine.with_value(|e| e.write(ToneRow { key: tone.get_untracked().storage_key(),
                                            body: Some(html), updated_at: None }))
};
let commit = move |html: String| { write_body(html); engine.with_value(|e| e.flush()); };
```

— and the engine's `flush()` calls a typed server function
(`crates/site/src/sync/server_fns.rs`):

```rust
#[server(prefix = "/api", endpoint = "save_tone", input = Json)]
pub async fn save_tone(key: String, body: String) -> Result<Remote<()>, ServerFnError> {
    let Some(pg) = Postgrest::from_env() else { return Ok(Remote::Offline) };
    /* upsert via reqwest, ssr-only */
}
```

Preserved from the original: immediate localStorage write, debounce-then-flush, blur flushes
now, failed flush re-queues (newer local edits win) and retries in 4 s, and the
focused-editor guard (poll never overwrites the row being typed —
`engine.rs::apply_remote`). Redesigned per contract decisions: browser-side supabase-js and
its key are gone; `Remote::Offline` carries the graceful local-only degrade.

### E10 — Data-carrying state machine

The image-slot reframe interaction (`image-slot.js` drag/pan/corner-resize) ports as an enum
whose variants carry the gesture geometry (`tarot/image_slot.rs:55`):

```rust
enum Drag {
    Pan { /* pointer + frame origin */ },
    Corner { fw: f64, fh: f64, ox: f64, oy: f64, ux: f64, uy: f64, diag0: f64, s0: f64 },
}
// held as RefCell<Option<(i32 /*pointer id*/, Drag)>> — one region, explicit states
```

Illegal states (corner-drag without geometry) are unrepresentable. This is the T11 rule:
don't port `this._drag = {mode:"corner", ...}` bags as structs of `Option`s.

### E11 — Refs + the don't-clobber-typing guard

The original guards contenteditable content against remote refresh with an activeElement
check (`Six Tones.dc.html:1069-1073`) and only writes `innerHTML` on tone change
(`loadedTone` memo). The port centralizes both in `EditableText`
(`lol_ds/src/components/editable.rs`):

```rust
// value -> DOM, with the don't-clobber-typing guard.
Effect::new(move |_| {
    let v = value.get();
    let Some(el) = node.get() else { return };
    let focused = document().active_element().map(|a| &a == el_elem).unwrap_or(false);
    if focused { return; }                    // never write into a focused editor
    if current != v { el.set_inner_html(&v); }
});
```

### E12 — Number formatting parity

JS `String(number)` and Rust's `Display` for `f64` are both shortest-round-trip, so
`format!("{v}")` reproduces the original's `Math.round(v*100)/100` position strings exactly
(`tones/model.rs::fmt_num`, unit-tested against values read out of the running original:
`"14.58"`, `"73.75"`, …). Do not use `{:.2}` — trailing zeros would diff every inline style.

## 3. Anti-patterns: symptom → cause → fix

### A1 — Release SSR build dies: "queries overflow the depth limit!"

- **Symptom:** `cargo leptos build --release` fails in rustc's type-layout query on a
  page-sized `view!` tree. Debug builds are fine.
- **Cause:** cargo-leptos passes `--cfg erase_components` **in dev only**; release compiles
  the fully-typed view tree, whose nesting exceeds rustc's type-layout recursion depth.
- **Fix:** erase the big branches with `.into_any()` — at component tails, at each arm of a
  large `match`/screen split, and at inner chunks if the combined branch still overflows
  (`tones/mod.rs` erases `atlas_view`, `editor_view`, *and* the editor's `editor_bar` /
  `tone_header` / `sound_header` / `reading_body` individually; the canvas erases
  DcViewport/DcSection/DcArtboardFrame/DcFocusOverlay roots).
  `#![recursion_limit = "256"]` at the crate root was **not sufficient** on its own in this
  port's experiments (`docs/ds-change-requests-tones.md` §7) — use `.into_any()`.
- **Cost note:** `.into_any()` at component roots is cheap; don't sprinkle it on every leaf.

### A2 — `viewBox` (any camelCase SVG attribute) renders literally

- **Symptom:** leptos 0.7 `view!` emits `attr:viewBox` as a literal attribute name; the SVG
  gets no viewBox and paints wrong (leptos 0.7.8).
- **Fix:** build fully-static SVG as a markup string and render it with `inner_html` on a
  host `<div>`, deriving coordinates from the same constants as the rest of the layout so
  there is one source of truth — then unit-test the string
  (`tones/model.rs::svg_frame` + `svg_frame_uses_the_same_triangle_in_viewbox_units`):

  ```rust
  <div class="atlas-svg" inner_html=svg_frame()></div>
  ```

  For *dynamic* SVG, either regenerate the string reactively
  (`move || svg_string(sig.get())`) or restructure so the dynamic part is not an SVG attr.

### A3 — A route hangs forever in SSR

- **Symptom:** the Axum render of one route never completes; other routes fine.
- **Cause:** `StoredValue::new_local` (or any `LocalStorage`-arena creation) executed while
  the request is being server-rendered — the local arena must not be touched during SSR;
  doing so wedges the render (`docs/ds-change-requests-tarot.md` §4).
- **Fix:** create local-arena values inside a client-only `Effect::new`, never in the
  component body (`tarot/image_slot.rs` creates its unmount guard inside the mount effect).

### A4 — WASM instance aborts on unmount/navigation

- **Symptom:** navigating away from a page kills the whole app; console shows an abort /
  `RwLock` re-entrancy; every signal after it is dead ("arena poisoned").
- **Cause:** a `Drop` impl (or `on_cleanup` body) reads a `NodeRef` or any reactive-arena
  value while the owner is being disposed — the arena is write-locked during disposal.
- **Fix:** resolve and cache raw DOM handles **at mount**; teardown paths touch only the
  cache (`tarot/image_slot.rs::Els` — "Methods (including the unmount Drop path) read these
  cached handles instead of NodeRefs").

### A5 — `on_cleanup` won't take your JS handles (`Send + Sync` bound)

- **Symptom:** cleanup closure holding `Rc`/`web_sys` handles fails the `Send + Sync` bound.
- **Fix:** the Drop-guard idiom — a struct owning the handles with a `Drop` impl (obeying
  A4), stored via `StoredValue::new_local` **inside the mount effect** (A3); the effect's
  per-run owner drops the guard at component unmount (`tarot/image_slot.rs:458-487`). A
  `ResizeObserver` wrapper that disconnects in its own `Drop` composes with this
  (`tarot/dom_ext.rs::ResizeObserver`).

### A6 — "closure invoked recursively or after being dropped" on blur

- **Symptom:** committing a contenteditable field that closes itself (blur → `on_commit` →
  `editing.set(false)` → unmount) panics with the dropped-closure error.
- **Cause:** the unmount drops the element's handler closures while blur/focusout are still
  dispatching. **A microtask is still too early** — focusout follows blur within the same
  task.
- **Fix:** defer the teardown one macrotask (`tarot/spread.rs:568`):

  ```rust
  on_commit=Callback::new(move |v| {
      ctx.commit(Field::Body, v);
      leptos::leptos_dom::helpers::set_timeout(
          move || { editing.try_set(false); },
          std::time::Duration::ZERO,
      );
  })
  ```

  (Alternative: the component itself dispatches `on_commit` in a fresh task — filed as a DS
  change request in this port.)

### A7 — contenteditable discipline (the whole checklist)

The `EditableText` rules that survived parity testing (`lol_ds/components/editable.rs`):

1. **value → DOM only while unfocused** (E11). A remote poll or signal echo writing
   `inner_html` into a focused editor destroys the caret and the entry.
2. **Placeholder = `data-placeholder` + `data-empty="true"` + CSS `::before`**, hidden on
   focus (contract C-31). But check the original first: Six Tones' placeholder is a separate
   absolute layer that *stays visible while the empty editor is focused* — the port
   replicated that by not using the DS placeholder and rendering the page's own layer
   (`docs/ds-change-requests-tones.md` §4). Placeholder semantics are per-original, not
   universal.
3. **Paste is always plain text**: `preventDefault` + `execCommand("insertText", text)`.
4. **Commit granularity is a per-callsite choice**: `debounce_ms = 0` = commit on blur only
   (editor.jsx behavior); `debounce_ms > 0` = debounced while-typing commits. A page that
   needs *per-keystroke* writes (Six Tones writes localStorage on every input) listens for
   the bubbled `input` event on a wrapper div rather than demanding a new prop
   (`tones/mod.rs::on_body_input`, `ds-change-requests-tones.md` §5).
5. **`Enter` blurs single-line fields; `Escape` always blurs; `cmd/ctrl+B/I` only in html
   mode.** Render initial content into the SSR markup (escaped in text mode) so the server
   paint is correct before hydration.

### A8 — Effect writes a signal that other things read

- **Symptom:** double renders, stale intermediate states, effect ordering bugs.
- **Fix:** it's a `Memo`. If the value is a pure function of signals, `Memo::new` (or a
  closure if cheap). Reserve `Effect` for the external world: DOM measurement, timers,
  storage, network kick-off. This port shipped zero derived-state effects.

### A9 — Missing web-sys features (frozen manifest)

- **Symptom:** `web_sys::Element::get_bounding_client_rect` (etc.) doesn't exist — the
  feature (`DomRect`, `NodeList`, `FileList`, `ResizeObserver`, `FontFaceSet`,
  `ImageBitmap`, …) isn't enabled, and the manifest is frozen.
- **Fix, two escape hatches, both shipped here:**
  - **Structural `#[wasm_bindgen]` externs** via the `web_sys::wasm_bindgen` re-export (no
    manifest change) — duck-typed `Rect`/`NodeList`/`DomQuery` types
    (`tarot_canvas/dom.rs`). Also the only option for APIs web-sys lacks entirely (Safari
    `GestureEvent`).
  - **`js_sys::Reflect` helpers** — `get`/`call`/`apply` wrappers
    (`tarot/dom_ext.rs`: `bounding_rect`, `query_selector_all`, `first_file`,
    `on_fonts_ready`, promise combinators).
- When the manifest is *not* frozen, just add the features — it's additive and cheaper than
  either hatch. File the feature list as a change request regardless
  (`docs/ds-change-requests-tarot.md` §1).

### A10 — Release wasm dead on every route: `Table.grow` RangeError

- **Symptom:** `cargo leptos build --release` succeeds, but the optimized wasm dies at
  `__wbindgen_start` with `RangeError: WebAssembly.Table.grow(): failed to grow table by 4`
  on **every** route.
- **Cause:** binaryen/wasm-opt **108** rewrites the `__wbindgen_externrefs` table export
  onto the sealed (min=max) funcref table — wasm-bindgen 0.2.118 emits two tables and
  wasm-opt 108 reassigns the export to table 0, with or without `--enable-reference-types`.
  cargo-leptos prefers the PATH binary.
- **Fix (environment, no repo change):** binaryen **≥ 110** on PATH (verified fixed with
  131 via `npm i -g binaryen`), or remove wasm-opt from PATH so cargo-leptos downloads its
  own pinned version (`docs/ds-change-requests-book-canvas.md` §4). Diagnose by
  section-dumping the artifact and checking which table `__wbindgen_externrefs` exports.

### A11 — `el.style()` resolves to the wrong method

- **Symptom:** calling `.style(…)` on a `web_sys::HtmlElement` hits tachys's
  `ElementExt::style(self, style)` (in scope via the leptos prelude), not web-sys's zero-arg
  getter.
- **Fix:** UFCS — `web_sys::HtmlElement::style(el).set_property(prop, val)`
  (`tarot/dom_ext.rs::set_style`, `tarot_canvas/dom.rs::css`).

### A12 — Image ingest / webp encode in WASM

The browser is still the codec. Reproduce the original pipeline with its constants
(`tarot/dom_ext.rs::file_to_webp_data_url`, from `image-slot.js:167-181`):
`createImageBitmap(file)` → scale cap `min(1200, round(rendered_width × 2))` → draw to a
canvas → `canvas.to_data_url_with_type_and_encoder_options("image/webp", &0.85.into())`.
Keep the original's generation counter so a stale async encode can't overwrite a newer drop
(`image_slot.rs` `gen: Cell<u64>`), and keep the persisted-value guard
(`/^data:image\//`).

### A13 — Anything touching `window` runs during SSR

- **Symptom:** server panic or hang on first request; or hydration mismatch.
- **Fix:** client-only work (localStorage paint, engine start, measurement) lives in
  `Effect::new`; the component body and initial `view!` must render meaningfully on the
  server with no browser APIs. The sync engine documents this at its boundary: "`start()`
  must be called from a client-only context (inside `Effect::new`) … Everything else is safe
  to construct during SSR" (`sync/engine.rs`).
