# Spec → Next.js: translation notes and traps

The spec-to-React mapping is nearly identity — React is React. The traps are all in the **App
Router / Server Component model** and in **Tailwind v4**, not in the view syntax. This file is the
catalog; read it before writing components.

Versions this targets: **Next.js App Router (14/15-era), React 18/19, Tailwind v4.** If the target
repo differs (Pages Router, Tailwind v3), see the notes at the end and update
`project-conventions.md`.

---

## The Server/Client boundary — the whole game

App Router components are **Server Components by default.** A component becomes a Client Component
only with a `"use client"` directive at the top of its file.

| Spec signal | Component kind |
|---|---|
| pure presentation, no `behavior_contract` state/callbacks | Server Component |
| `owns_state`, has `callbacks`, or a `states.machine` with events | Client Component (`"use client"`) |
| fetches data (`behavior_contract.fetches: true`) | Server Component (`async`) or Server Action |
| Context provider | Client Component |

### Rules that pass `tsc` and fail `next build`

These are the RSC equivalents of the Leptos release-build traps — invisible until `next build`:

1. **Non-serializable props across the boundary.** A Server Component may pass a Client Component
   only serializable props — no functions, no class instances, no `Date` in some setups. Passing an
   `onClick` down from a server parent is the classic failure. Fix: the interactive piece owns its
   own handler as a client component, or the handler is a Server Action (which *is* passable).
2. **Server-only import in a client bundle.** Importing a module that touches `fs`, `process.env`
   secrets, or a DB client into anything reachable from `"use client"` leaks it (or errors). Mark
   server-only modules with `import "server-only"` so the mistake fails loudly at build.
3. **Hooks/handlers in a Server Component.** `useState`, `useEffect`, `onClick` in a file without
   `"use client"`. The error is clear at build; the fix is the directive — but only on the leaf that
   needs it.
4. **`"use client"` too high.** Legal, builds fine, and silently bundles the entire subtree to the
   client. Not an error — a performance defect. Push the directive to the interactive leaf.

**Design rule:** push `"use client"` as far down as possible. A page is a server component that
composes mostly server components and drops to client components only at the interactive leaves
(a button with state, a form, a disclosure).

---

## State — statechart subset → TypeScript

The spec's `states.machine` (XState-compatible subset) lowers cleanly:

- **One parallel region → one `useState`**, or the whole machine → one `useReducer` when transitions
  have guards. Do not merge orthogonal regions into a single state object.
- **A region's states → a discriminated union**, so illegal states are unrepresentable:

```ts
type Async =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "error"; message: string };
```

- **Transitions → a reducer** whose cases mirror the machine's events. Guards (`cond`) become
  conditions in the reducer. This is the same "illegal transitions unrepresentable" goal the Leptos
  skill lowers to a Rust enum — TS unions get you most of the way.
- **`sends` from `a11y.keyboard`** → the keyboard handler dispatches that event, so keyboard and
  pointer paths share the reducer. Keeps behavior verifiable against the spec, not the reference.

Derived values: compute in render. React re-renders the component; a `useMemo` is warranted only for
a measured-expensive computation, not by default. (Opposite instinct from Leptos, where derived
values are always `Memo`s — because the models differ. Do not port the Leptos habit here.)

---

## Absolute → flow: what the Figma reference actually looks like

Real, unedited `get_design_context` output (2026-07-19):

```tsx
export default function Component1({ className }: { className?: string }) {
  return (
    <div className={className || "h-[54px] relative w-[360px]"} data-node-id="16:27">
      <div className="absolute inset-[0_82.5%_0_0]" data-node-id="16:12">
        <div className="absolute inset-[-2.78%_-2.38%_0_-2.38%]">
          <img alt="" className="block max-w-none size-full" src={imgTicket1Traced} />
        </div>
      </div>
      <p className="absolute font-['Philosopher:Regular'] inset-[12.96%_0_12.96%_22.5%] text-[36px] text-black whitespace-nowrap">
        Idea and Roadmap
      </p>
    </div>
  );
}
```

Everything to notice, because all of it recurs:

| Reference emits | Convert to |
|---|---|
| `absolute inset-[12.96%_0_12.96%_22.5%]` on every layer | flow layout — `flex`/`grid` with `gap` |
| `h-[54px] w-[360px]` fixed root | intrinsic size + `min-h`/`max-w`, unless truly fixed |
| `text-[36px]`, `text-black` | themed utilities from `@theme` |
| `font-['Philosopher:Regular']` | `next/font` + `--font-*` (see below) |
| `whitespace-nowrap` on text | usually drop — it exists to stop reflow in a fixed box |
| hardcoded copy ("Idea and Roadmap") | a prop, per the spec's `props` |
| `data-node-id="16:27"` | keep during development (traceability), strip for production |
| `{ className }: { className?: string }` | keep — a reasonable override convention |

**The icon-beside-label case above** is the canonical example: an icon at `inset-[0_82.5%_0_0]` and
a label at `inset-[…_22.5%]` is a two-item row. It becomes:

```tsx
<div className="inline-flex items-center gap-3">
  <TicketIcon className="size-[54px] shrink-0" />
  <span className="text-heading text-primary">{label}</span>
</div>
```

Percentages that encode "the icon takes the left 17.5%" are expressing a *gap and an intrinsic
width*. Recover the intent; do not transcribe the arithmetic.

**When to keep `absolute`:** genuine overlays — a badge pinned to a corner, a decorative layer, a
dropdown panel. Then it is one `relative` parent with one `absolute` child, not a whole subtree.

## Tailwind v4 — CSS-first, and the spec already speaks it

Tailwind v4 configures in **CSS, not `tailwind.config.js`.** The spec's `theme.css` is a v4
`@theme` block — so it drops straight in:

```css
@import "tailwindcss";

@theme {
  --color-text-primary: #0a0a0a;
  --color-bg-accent: #2563eb;
  --radius-control: 0.375rem;
  --spacing-2: 0.5rem;
}
```

- **Install the spec's `theme.css` as the project theme** (or merge into the existing one). Then a
  spec token `{color.text.primary}` is the utility `text-primary` — generated by the `@theme` var
  `--color-text-primary`. Namespaces matter: `--color-*` → color utilities, `--spacing-*` → spacing,
  `--radius-*` → radius, `--font-*` → font.
- **Never emit arbitrary values for tokened properties.** `bg-[#2563eb]` is a skipped binding; the
  themed `bg-accent` is correct. Arbitrary values are for genuinely one-off, un-tokened cases only.
- **Dark mode:** v4 uses a `@custom-variant dark` (commonly `class`-based). Confirm the repo's
  choice; the spec's semantic tier is what swaps, exactly as in the DTCG token design.
- If the repo is **Tailwind v3**, there is no `@theme` — the tokens go into `tailwind.config.js`
  `theme.extend` and CSS variables in `:root`. Note it in `project-conventions.md`; the binding
  discipline is identical, only the mechanism differs.

---

## Assets — the design-to-code G5 rules still hold

`get_design_context` returns icons/images as exported assets with **~7-day URLs.** For committed
Next.js code:

- Download the bytes; do not reference the expiring URL. Place under `public/` or import as a static
  asset.
- Use `next/image` for raster images (it wants explicit `width`/`height` — the spec/design geometry
  supplies them; do not `auto` a fixed-size leaf).
- SVG icons: inline as components or import; preserve the outer box and inner-leaf geometry
  separately, per G5. Do not apply one global icon size across unlike glyphs.

---

## Fonts

Use `next/font` (`next/font/google` or `next/font/local`) for any custom font — it self-hosts and
eliminates layout shift. Map the spec's `fontFamily` tokens to the loaded font's CSS variable, then
reference it through the `@theme` `--font-*` namespace. Do not `<link>` a font CDN in `head`; that
reintroduces the shift `next/font` exists to remove.

**Figma emits `font-['Family:Weight']`** — verified: `font-['Philosopher:Regular']`,
`font-['PingFang_HK:Ultralight']`. Note the family/weight are fused in one token and spaces become
underscores. Split them:

```tsx
import { Philosopher } from "next/font/google";
const philosopher = Philosopher({ subsets: ["latin"], weight: ["400"], variable: "--font-display" });
```

then `@theme { --font-display: …; }` and use `font-display`. Two cautions: the weight word must map
to a numeric weight (Regular→400, Ultralight→200), and a font that is **not** on Google Fonts
(`PingFang HK` is a system font) needs `next/font/local` with the file, or an explicit fallback
stack. A missing font silently falls back and the design drifts — flag it rather than guessing.

---

## Routing and layout

- Pages are `app/<route>/page.tsx`; shared shell is `app/layout.tsx` (a Server Component — keep it
  server unless it truly must be client, e.g. a top-level provider, which should itself be a small
  `"use client"` wrapper rather than making the whole layout client).
- The spec's `layout.md` responsive intent → Tailwind responsive prefixes (`md:`, `lg:`). "Sidebar
  collapses below `md`" is `hidden md:block` / a drawer, not absolute positioning.
- Metadata via the `metadata` export or `generateMetadata`, not `<head>` tags.

---

## Pages Router / other setups

If the repo is Pages Router (`pages/` not `app/`): no Server Components, no `"use client"` — every
component is client, data comes from `getServerSideProps`/`getStaticProps`, not `async` components.
Most of the boundary section above does not apply; the state and Tailwind sections still do. Record
the setup in `project-conventions.md` and translate accordingly.
