# Harness recipes

Copy-pasteable rigs, seeded from the library-of-light parity run (2026-07-16). Scripts
marked *preserved* exist verbatim in `parity/book-canvas/harness/`; the rest are canonical
reconstructions of methods documented in `parity/*/MANIFEST.md`. Adjust paths; keep the
flags — each one is there because its absence produced diff noise.

## 1. Browser launch + capture skeleton (*preserved*: `harness/shot.js`)

Works with puppeteer-core against any Chromium (this run: the Playwright-installed
`chromium-1194` binary; plain Playwright is equivalent).

```js
// shot.js — Usage: node shot.js <url> <outdir> <prefix> [mode]
const puppeteer = require('puppeteer-core');
const [, , url, outdir, prefix, mode = 'book'] = process.argv;

(async () => {
  const browser = await puppeteer.launch({
    executablePath: '/opt/pw-browsers/chromium-1194/chrome-linux/chrome',
    args: ['--no-sandbox', '--disable-dev-shm-usage',
           '--force-device-scale-factor=1',   // DPR 1 or nothing lines up
           '--hide-scrollbars'],               // scrollbars diff between hosts
  });
  const page = await browser.newPage();
  await page.setViewport({ width: 1440, height: 900 });   // manifest breakpoint
  await page.goto(url, { waitUntil: 'networkidle0' });
  await page.evaluate(() => document.fonts.ready.then(() => {}));  // metrics settle
  await new Promise((r) => setTimeout(r, 800));                    // paint settle

  if (mode === 'book') {
    // Element-level capture: every .state div, id = manifest state id.
    const ids = await page.$$eval('.state', (els) => els.map((e) => e.id));
    for (const id of ids) {
      const el = await page.$('#' + id);
      await el.screenshot({ path: `${outdir}/${prefix}-${id}.png` });
    }
  }
  await browser.close();
})().catch((e) => { console.error(e); process.exit(1); });
```

Console hygiene — attach before `goto`, report at the end, treat as findings:

```js
const errors = [];
page.on('console', (m) => { if (m.type() === 'error') errors.push(m.text().slice(0, 300)); });
page.on('pageerror', (e) => errors.push('PAGEERROR: ' + String(e).slice(0, 300)));
```

## 2. Determinism prelude (clock freeze, storage seed)

Run on **both** sides before any capture. Clock freeze (the tones pass froze both sides to
`2026-07-16T12:34:00Z` so live clocks diff 0 px) — install before any page script runs:

```js
await page.evaluateOnNewDocument(() => {
  const FROZEN = new Date('2026-07-16T12:34:00Z').getTime();
  const RealDate = Date;
  // Enough for toLocaleTimeString/toISOString/now consumers; extend if the app does date math.
  class FrozenDate extends RealDate {
    constructor(...a) { a.length ? super(...a) : super(FROZEN); }
    static now() { return FROZEN; }
  }
  window.Date = FrozenDate;
});
```

localStorage seeding per the manifest's `setup:` lines (shape must be the *original's*
value encoding — that is itself a contract under test):

```js
await page.evaluateOnNewDocument(() => localStorage.clear());          // atlas-idle-empty
// atlas-written-dots: seed 1, 3, sound with body HTML
await page.evaluateOnNewDocument(() => {
  localStorage.setItem('lol-hd-tones:1', JSON.stringify({ body: '<div>first reading</div>' }));
  localStorage.setItem('lol-hd-tones:3', JSON.stringify({ body: '<div>third reading</div>' }));
  localStorage.setItem('lol-hd-tones:sound', JSON.stringify({ body: '<div>the throat</div>' }));
});
```

Settled hover / active states:

```js
await page.mouse.move(x, y); await sleep(700);        // outlast the transition (600ms ease)
await page.mouse.move(x, y); await page.mouse.down(); // active-press: hold, shoot, release
```

## 3. Reference host from orphaned components (*preserved*: `harness/book.html`)

For components whose composing host is lost (excavation finding — pages.jsx / lost
`Guidebook.html`): precompile the JSX, load vendored React UMD + the canonical token CSS,
render one `.state` div per manifest state.

Precompile (network-free once esbuild exists; classic JSX runtime → global React):

```sh
esbuild pages.jsx --jsx=transform --outfile=/tmp/reference-host/pages.js
cp vendor/react.production.min.js vendor/react-dom.production.min.js colors_and_type.css \
   /tmp/reference-host/
```

Host document:

```html
<!doctype html>
<html data-theme="dark" lang="en">
<head>
  <meta charset="utf-8" />
  <link rel="stylesheet" href="./colors_and_type.css" />
  <style>
    html, body { margin: 0; background: var(--bg); }
    .gallery { display: flex; flex-direction: column; gap: 40px; padding: 40px; align-items: flex-start; }
    .state { position: relative; }
  </style>
</head>
<body>
  <div id="root"></div>
  <script src="./vendor/react.production.min.js"></script>
  <script src="./vendor/react-dom.production.min.js"></script>
  <script src="./pages.js"></script>
  <script>
    const h = React.createElement;
    const states = [                 // id = verify-manifest state id, exactly
      ["cover-plate", h(CoverPlate)],
      ["cover-tablet", h(CoverTablet)],
      ["template-court", h(MinorCourtPage)],
      // …one entry per manifest state; pass props to force variants
    ];
    ReactDOM.createRoot(document.getElementById("root")).render(
      h("div", { className: "gallery" },
        states.map(([id, el]) => h("div", { className: "state", id, key: id }, el)))
    );
  </script>
</body>
</html>
```

Serve with any static server (`python3 -m http.server`) — required if anything `fetch()`es.
State the reconstruction in the manifest header: component parity is verifiable;
composition parity is not (the host is lost) — say so instead of inventing a reference.

## 4. Offline reference host for a CDN-dependent export (tones pattern)

When the original loads runtime deps from CDNs and the sandbox intercepts TLS: byte-copy the
export to a scratch dir and satisfy the loads locally. Two edits that were enough for a
dc-runtime export, both declared in the manifest header:

1. **Pre-satisfy the CDN check.** The runtime skips its React CDN fetch when `window.React`
   exists — inject a vendored React UMD (borrowed from the sibling export) via `<script>`
   tags *before* the runtime bundle.
2. **Duplicate dynamically-injected CSS statically.** The runtime hoisted a
   `<link rel="stylesheet">` at boot which never applied under the CDP session; adding the
   same `<link>` statically to `<head>` is a no-op visually and makes capture reliable.

Rules: modify only the copy; smallest possible edit set; every edit listed in the manifest
header; identical fallback behavior on both sides (if the reference can't reach the remote
store, run the port's equivalent state with sync disabled — or use SOURCE-VERIFIED).

## 5. Pixel diff

Two metrics used, both reported per state in the manifests:

**pixelmatch** (tones/tarot: absolute differing-pixel count, threshold 0.12):

```js
const { PNG } = require('pngjs'); const pixelmatch = require('pixelmatch');
const a = PNG.sync.read(fs.readFileSync(ref)), b = PNG.sync.read(fs.readFileSync(port));
if (a.width !== b.width || a.height !== b.height) throw new Error('size mismatch — fix the rig, never rescale');
const diff = new PNG({ width: a.width, height: a.height });
const n = pixelmatch(a.data, b.data, diff.data, a.width, a.height, { threshold: 0.12 });
fs.writeFileSync(`diff-${state}.png`, PNG.sync.write(diff));   // keep the mask as evidence
console.log(state, n, 'px');
```

**sharp channel-%** (book-canvas: % of RGBA channel values differing by >8/255):

```js
const sharp = require('sharp');
const [ra, rb] = await Promise.all([ref, port].map(f => sharp(f).raw().toBuffer()));
let d = 0; for (let i = 0; i < ra.length; i++) if (Math.abs(ra[i] - rb[i]) > 8) d++;
console.log(state, (100 * d / ra.length).toFixed(3) + '%');
```

Calibration from the run: 0 px / 0.000% is the normal result for structural states;
≤ ~2% with **no positional shift** is text-antialiasing jitter (verify with a magnified
crop of the diff mask); anything localized or shifted is a finding. Never rescale to make
sizes match — a size mismatch means the rig differs, and the comparison is void.

## 6. Fixture capture for pure functions

Freeze the original's pure logic as input→output fixtures the port's native tests replay
(48 cases shipped in `crates/site/src/pages/tarot/pagination_fixtures.json`).

`capture.html` — transcribe the functions **verbatim** (no reformat, no cleanup; you are
capturing behavior, bugs included):

```html
<script>
/* ==== BEGIN verbatim transcription from editor.jsx:237-313 ==== */
function bodyToBlocks(html) { /* …unchanged… */ }
function blockToWords(block) { /* …unchanged… */ }
function wordsToHtml(words) { /* …unchanged… */ }
function pageWordsToBlocks(words) { /* …unchanged… */ }
/* ==== END transcription ==== */

const CASES = {
  body_to_blocks: ["", "hello world", "<div>para one</div><div>para two</div>",
    "<div>one</div><div><br></div><div>two</div>", "<div> <br/> </div>",
    "line<br>break<br/>third<BR>fourth", "<DIV CLASS=\"y\">upper</DIV>lower",
    "a &amp; b", /* …curate edge cases: entities, nested tags, whitespace, case… */],
  block_to_words: ["x<i>y</i>z", "<b>bold <i>both</i></b> plain", /* … */],
};
const out = {};
out.body_to_blocks = CASES.body_to_blocks.map(input => ({ input, output: bodyToBlocks(input) }));
out.block_to_words = CASES.block_to_words.map(input => ({ input, output: blockToWords(input) }));
// roundtrips: body → blocks → words → pages, with a chosen cut index
document.title = 'FIXTURES:' + btoa(unescape(encodeURIComponent(JSON.stringify(out))));
</script>
```

Dump and freeze:

```js
await page.goto('file:///tmp/capture.html');
const b64 = (await page.title()).replace(/^FIXTURES:/, '');
fs.writeFileSync('pagination_fixtures.json', Buffer.from(b64, 'base64'));
```

Port-side test: `include_str!("pagination_fixtures.json")`, iterate every case, `assert_eq!`
the Rust function's output — **byte-identical**, no tolerance. Base64-through-title avoids
console truncation and encoding mangling of HTML-bearing JSON.

For the DOM-*dependent* remainder (measurement): close it end-to-end instead — seed the same
long body on both sides, assert same page count and same last-page words in the same
browser/fonts.

## 7. Print-to-PDF comparison

Capture (either form):

```sh
chromium --headless --no-sandbox --print-to-pdf=/out/print-one-orig.pdf "$REF_URL"
```

```js
await page.pdf({ path: 'print-all-port.pdf', preferCSSPageSize: true, printBackground: true });
```

If printing is app-triggered (readiness polls, print-all preparation), script the app's own
print path but stub the dialog: `page.evaluateOnNewDocument(() => { window.print = () =>
window.__printed = true; })`, wait for the flag, then call `page.pdf(...)` — this exercises
the readiness logic (blur-commit, no page taller than the sheet) without a dialog.

Compare — page count, page boxes, break behavior:

```sh
pdfinfo -box print-one-orig.pdf   # Pages + MediaBox; 500×700 CSS px ⇒ 375×525 pt (this run reported 375.12)
```

Assert: identical page counts (written-only filtering, continuation pages), identical boxes,
no trailing blank page, screen-only chrome absent (render each PDF page to PNG and reuse §5
if box/count parity isn't convincing enough). Note emulation caveats *when identical on both
sides* as MATCH-with-caveat + a real-browser deferral, not as findings.

## 8. Interaction replay with logged asserts (*preserved*: `harness/canvas-states.js`)

One script drives both hosts (pass the URL); every claim is a logged `PASS`/`FAIL` line into
`behavior-log.txt`; numeric state is *read out and compared*, not eyeballed:

```js
const worldTf = () => page.evaluate(() =>
  document.querySelector('.design-canvas > div').style.transform || '(none)');

// behavior-pan-zoom: 5 notched wheel steps at centre, then compare transforms verbatim
for (let i = 0; i < 5; i++) { await page.mouse.wheel({ deltaY: 120 }); await sleep(60); }
log(`transform after 5 steps: ${await worldTf()}`);   // ref vs port: byte-identical string

// persistence: mutate, read the stored JSON verbatim, reload, re-assert
const stored = await page.evaluate(() => localStorage.getItem('lol-tarot-canvas:state'));
assert(stored === EXPECTED_C29_SHAPE, 'C-29 shape verbatim');
await page.reload({ waitUntil: 'networkidle0' });
```

Focused-guard assert (mock the server surface, then poke it):

```js
// With the editor focused, a remote change must NOT clobber; unfocused, it must.
await page.evaluate(() => fetchMock.setRow('3', 'REMOTE CHANGE B'));  // or route-intercept /api/load_*
await page.focus('[contenteditable]'); await sleep(POLL_MS + 1000);
assert((await editorText()).includes('ORIGINAL BODY'), 'focused editor not clobbered by poll');
await page.click('body'); await sleep(POLL_MS + 1000);
assert((await editorText()).includes('REMOTE CHANGE B'), 'unfocused editor refreshed');
```

Two-client convergence: two browser contexts (`browser.createBrowserContext()`) on the same
port URL; edit in A, assert it appears in B within one poll interval; for field-level merge,
edit *different fields of the same row* in both and assert neither is lost.

Live write-path tests: throwaway keys outside the real roster only, and the manifest names
every row left behind for deletion.

## 9. Manifest skeleton

```markdown
# Parity manifest — <package> (<route>)

_<date> · reference = <exact construction + every harness edit> · port = <url, build type>.
Capture: <tool, browser, viewports, device scale>. Clock frozen to <instant> on both sides.
Diff: <metric + threshold>. Fonts: <caveat — compare structure/spacing/color only>._

## Screenshot pairs
| State (manifest id) | Files | Diff | Verdict |
|---|---|---|---|

## Behavior (behavior-log.txt has the full PASS list)
- ref & port: <paired asserts>
- port: <port-only asserts, with why the ref could not run them>

## Known deltas (accepted / flagged)
1. <delta> — <decision citation (C-XX / plan §)>
```
