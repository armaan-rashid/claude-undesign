# Proposed "Claude Design" section for `references/known-scaffolds.md`

_Drafted from the first real excavation (fixture: `library-of-light/subwebsites` —
`hd-tones` + `tarot guidebook`, 2026-07-16; full evidence in that run's CONTRACTS.md).
Every entry below is `confirmed` against that fixture. Replaces the current `unfilled`
placeholder section. The old `likely` probes are dispositioned at the bottom — two of the
four were wrong and should not survive._

---

## Claude Design

**Status:** `confirmed` (fixture: `library-of-light/subwebsites`)

**Fingerprint** (check in order — no `package.json` exists in these exports, so the
generic step 1 fingerprint never fires):

1. `*.dc.html` files, plus a `support.js` whose first line reads
   `// GENERATED from dc-runtime/src/*.ts — do not edit.`
2. A `_ds/<system-name>-<uuid>/` folder containing `_ds_manifest.json`, `_ds_bundle.js`,
   `colors_and_type.css`, and `_adherence.oxlintrc.json` — the lint file carries an
   `x-omelette` key ("omelette" is the host's internal name; grep: `x-omelette`).
3. `window.omelette` references and `.state.json` sidecar filenames in source comments.
4. `__dc_`-prefixed postMessage types (`__dc_booted`, `__dc_zoom`, `__dc_probe`, …).
5. Header comments of the form `AUTO-GENERATED from <name>.jsx (do not edit by hand)`.

**One generator, two architectures.** The same project contained (a) a **dc-runtime
single-file app** (`*.dc.html` + `support.js`) and (b) a **plain React 18 UMD app**
(vendored `vendor/react*.js`, precompiled `editor.js`, a web component, raw `.jsx`
siblings). Fingerprint for both before concluding which checks apply — a project can be
both at once.

### CD-1 — dc-runtime single-file component (`<x-dc>` / `DCLogic` / `renderVals`)

**Status:** confirmed (fixture: `hd-tones/Six Tones.dc.html`)
**Kind:** magic-name
**Signature:** an `<x-dc>` template block + `<script type="text/x-dc" data-dc-script
data-props='…'>` defining `class Component extends DCLogic` — `DCLogic` is injected via
`new Function("DCLogic", …)`, not imported. Templates bind `{{ expr }}` against
`vals = { ...props, ...renderVals() }` (a restricted expression grammar: paths, `!`,
equality, literals — no calls). Special tags: `<sc-if value>`, `<sc-for list as>`,
`<helmet>` (hoisted to `<head>`). Design-time-only attrs: `hint-placeholder-val`,
`hint-placeholder-count`, `data-screen-label` — inert at runtime but they enumerate the
intended preview states, which is free input for the verify manifest.
**Grep:** `data-dc-script|<x-dc|renderVals|sc-if|sc-for|hint-placeholder`
**Shape:** logic class gets `props`, `state`, `setState`, React-style lifecycle;
`renderVals()` returns the flat view-model (values, and functions for `onClick`/`ref`).
**Usual decision:** drop the runtime; treat the `renderVals()` return object as the
component's spec surface (props/signals/callbacks) for extraction.

### CD-2 — `window.omelette` sidecar persistence (read-via-fetch, write-via-bridge)

**Status:** confirmed (fixture: `tarot guidebook/image-slot.js`, `design-canvas.jsx`)
**Kind:** storage
**Signature:** state persists to a `.<something>.state.json` file next to the HTML.
Reads are plain `fetch('./x.state.json')` (works anywhere the files are served together);
writes go through `window.omelette.writeFile(name, json)`, which the host allowlists to
`*.state.json` basenames at the project root. Outside the host, components degrade to
read-only and typically gate their editing affordances on
`window.omelette && window.omelette.writeFile`.
**Grep:** `window\.omelette|\.state\.json`
**Shape:** `omelette.writeFile(path: string, contents: string) => Promise`
**Usual decision:** redesign — replace with real persistence (localStorage or a server
boundary). Note: the async-read merge machinery around it (tombstones, echo-write
suppression, race guards) exists only because the read was a fetch; a synchronous store
lets you delete it deliberately rather than port it.

### CD-3 — `__dc_*` postMessage host protocol

**Status:** confirmed (fixture: `hd-tones/support.js`, `tarot guidebook/design-canvas.jsx`)
**Kind:** event
**Signature:** `window.parent.postMessage({ type: '__dc_…' }, '*')` plus a `message`
listener for host-sent types. Observed: `__dc_booted`, `__dc_design_mode`, `__dc_theme`,
`__dc_probe` (runtime side); `__dc_zoom`, `__dc_present`, `__dc_set_zoom` (canvas side).
Guarded by `window.parent === window` checks, so standalone behavior is a no-op. Related
marker: `data-omelette-chrome` attribute on elements the host should exclude from
captures. The dc-runtime also installs a host API on `window`: `__dcUpdate`,
`__dcSetProps`, `__dcBoot`, `__dcRegistry`, `getDC`, `__dcContentKeyed`, ….
**Grep:** `__dc_|data-omelette-chrome|window\.__dc`
**Usual decision:** drop — editor plumbing; verify nothing user-visible rides on it (in
the fixture, the host's zoom toolbar did — the capability moves in-page or dies).

### CD-4 — `_ds/` design-system export with machine-readable manifest (that can be stale)

**Status:** confirmed (fixture: `hd-tones/_ds/library-of-light-design-system-<uuid>/`)
**Kind:** css / data-shape
**Signature:** a versioned DS folder: `colors_and_type.css` (tokens as CSS custom
properties + semantic classes), `README.md` (brand voice/foundations),
`_ds_manifest.json` (JSON list of tokens/themes/fonts/preview cards),
`_adherence.oxlintrc.json` (lint rules banning raw hex/px/fonts — the generator writing
its own token discipline down; highest-value read in the folder after the CSS), and
`_ds_bundle.js` (see CD-5). **The manifest lags the CSS**: in the fixture it was missing
an entire token tier (`--amber*`, `--font-terminal`) that the CSS had and the app used.
Apps may also hold their own *older copies* of the DS CSS — diff every copy; additive
diffs mean the newest wins.
**Grep:** `_ds_manifest|_adherence|x-omelette` ; token diff per G-5.
**Usual decision:** shim the CSS (it is the real token source). Treat the manifest as a
hint, never as truth — `ds-spec-extract`'s "prefer the bundle" rule must reconcile
against the CSS and record disagreements.

### CD-5 — Self-registering window-global component kits (orphan exports without imports)

**Status:** confirmed (fixture: `_ds_bundle.js`, `pages.jsx`, `design-canvas.jsx`)
**Kind:** magic-name / global
**Signature:** no module system at all — files end in `Object.assign(window, { Page,
CoverPlate, DesignCanvas, … })` and a *host HTML file* composes them via bare globals.
G-1's import-graph technique finds nothing here (there are no imports); the orphan test
becomes "which `Object.assign(window, …)` names does no in-repo file reference?" In the
fixture, an entire host (`Guidebook.html`) was missing from the export — the README
mentioned it; the registry calls were the only trace. Raw `.jsx` files with no compiled
twin imply the missing host also loaded Babel.
**Grep:** `Object.assign\(window,` ; then cross-reference each name against the repo.
**Usual decision:** redesign — the composition must be reconstructed (from README/prose
if the host is missing); the global registry itself dies in any module-system target.

### CD-6 — Dependency delivery: CDN-pinned (runtime) and vendored (app), side by side

**Status:** confirmed (fixture: both apps)
**Kind:** fetch / glue
**Signature:** the dc-runtime loads React 18.3.1 UMD from unpkg with SRI hashes and
`@babel/standalone` lazily (grep: `unpkg.com/react@|@babel/standalone`); helmet blocks may
add jsdelivr CDN libs with floating majors (`@supabase/supabase-js@2`). The React-app
flavor instead **vendors** the same libraries locally (`vendor/react.production.min.js`,
`vendor/supabase.js`) so the file works by double-click with no network — the export
states this rationale in HTML comments. Precompiled `editor.js` from `editor.jsx`
(header: `AUTO-GENERATED from editor.jsx`) with the build step *not in the repo* is part
of the same file://-first posture.
**Grep:** `unpkg.com|cdn.jsdelivr.net|AUTO-GENERATED from|vendor/`
**Usual decision:** drop (target has its own toolchain). Port from the `.jsx` source, but
diff-check the compiled `.js` first — it is what actually ran.

### CD-7 — Config-as-global `window.*_CONFIG` script files

**Status:** confirmed (fixture: `hd-supabase-config.js`, `supabase-config.js`)
**Kind:** global / config
**Signature:** a tiny hand-editable `.js` file assigning `window.<NAME>_CONFIG = { url,
key/publishableKey, … }`, loaded before the app scripts (script order is the contract —
comments in the HTML say so). Consumers read it lazily and treat blank/placeholder values
(`/PASTE_/`) as "feature off, degrade to local-only" — the degrade path is deliberate
product behavior, not an error path. Live publishable keys are committed by design.
**Grep:** `window\.[A-Z_]+_CONFIG`
**Usual decision:** redesign — env/server config in the target; **preserve the degrade
path** (missing config ⇒ local-only mode), it is a feature users see.

### CD-8 — Cloud-sync wrapper global with a disabled stub twin

**Status:** confirmed (fixture: `tarot guidebook/cloud-sync.js` → `window.GuidebookSync`)
**Kind:** global / fetch-ownership
**Signature:** one IIFE owns all remote I/O and exposes a single window global with two
shapes: enabled (`{ enabled:true, ready:Promise, get*(), save*(), subscribe*(), getStatus,
onStatus }`) and disabled stub (`{ enabled:false, ready:resolved, getStatus:()=>'local',
onStatus:()=>()=>{} }`). Consumers must handle both. Load-bearing semantics live here,
not in components: field-level partial upserts, echo suppression on subscriber
notification, status values. Supabase Realtime channels (`.channel(…).on(
'postgres_changes', …)`) ride along; the table schema is documented in a header comment
and in a SETUP.md SQL block — schema evidence outside any code file.
**Grep:** `window\.[A-Za-z]+Sync|postgres_changes|enabled: false`
**Usual decision:** redesign into the target's data layer, preserving the ownership
semantics verbatim; realtime usually becomes polling.

### CD-9 — localStorage namespace prefixes as durable data contracts

**Status:** confirmed (fixture: `lol-hd-tones:`, `lol-tarot:`, `dc-viewport:<pathname>`)
**Kind:** storage
**Signature:** `const PREFIX = "<project>:"` + `JSON.parse(localStorage.getItem(PREFIX +
key))`; paint-first-then-reconcile against the remote store; sometimes a full
`localStorage.length` scan for the prefix; meta-keys like `_last`/`_lastCard` (which may
be write-only dead code — check for a matching read). The canvas flavor keys off
`location.pathname`, which was only safe because the omelette sandbox origin was
per-project.
**Grep:** `localStorage\.(get|set)Item|STORAGE_PREFIX|PREFIX`
**Usual decision:** shim — preserve prefixes and value shapes exactly if any user data
exists; note the `file://` → served-origin migration wall (localStorage does not follow).

### CD-10 — `<image-slot>` fillable-image web component

**Status:** confirmed (fixture: `tarot guidebook/image-slot.js`)
**Kind:** magic-name / data-shape
**Signature:** a shadow-DOM custom element for user-supplied images: drag/browse ingest,
canvas downscale to webp data URL (caps: 1200 px / 2× rendered width, q 0.85), dblclick
reframe (pan + corner scale in frame-%), persisted value `{ u: dataURL, s, x, y }` (legacy
bare-string form accepted), `s` clamped [1,5], `data:image/` allowlist on read-back,
editability gated on a persistence runtime being present. Persists via CD-2 sidecar or a
CD-8 sync global when configured. React consumers render the lowercase tag directly and
key it for remount.
**Grep:** `customElements.define\('image-slot'|image-slot`
**Usual decision:** redesign as a native component in the target, keeping the constants,
value shape, and crop math; the chrome uses omelette-brand terracotta `#c96442` — flag
whether to keep or retokenize (product question, not a technical one).

### Dispositions of the old `likely` probes (per this file's own rule: confirm or delete)

- **Artifact-style `window.storage` KV** — **not observed** in either export. Persistence
  was localStorage + Supabase + omelette sidecars. Delete the probe for Claude Design
  *site/app exports*; it may still apply to claude.ai artifacts, which are a different
  runtime.
- **Browser storage unavailable → in-memory patterns** — **contradicted**: localStorage is
  used heavily and deliberately. Delete for this export class.
- **Tailwind core-utilities-only** — **not applicable**: no Tailwind anywhere; styling is
  inline styles + CSS custom properties from the `_ds` token sheet. Replace with CD-4.
- **Single-file component conventions** — **confirmed and specified** as CD-1.
