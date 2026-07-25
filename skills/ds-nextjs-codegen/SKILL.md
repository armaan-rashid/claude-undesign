---
name: ds-nextjs-codegen
description: >
  Generate idiomatic Next.js (App Router) + Tailwind components from a framework-agnostic spec
  directory (tokens.json + component specs + state machines), or directly from a Figma design via
  the design-to-code path. Use whenever the user wants to port a spec or a Figma design to Next.js,
  says "port to Next.js", "Figma to Next.js", "build this in React/Next", or asks to implement
  components from a spec/ directory in Next.js. The swappable codegen stage for the React target.
---

# Next.js Codegen

The swappable codegen stage, React target. Same shape as `ds-leptos-codegen`: read the spec, apply
this framework's rules, emit components plus a preview surface for `ds-visual-verify`. The spec
format never changes; only this skill does.

**Source and target share a paradigm — which makes the real work easy to miss.** Figma's
`figma-design-to-code` emits React+Tailwind and the spec's `theme.css` already *is* a Tailwind theme,
so it is tempting to think this is a copy job. It is not. Four things carry the weight:

1. **De-absoluting the layout.** Verified on a real file: the reference is *entirely* absolutely
   positioned with percentage insets. Converting that to flow layout is this target's equivalent of
   translating reactivity semantics — the biggest single task (Step 3).
2. **The App Router Server/Client boundary** (Step 1).
3. **Binding the theme and reusing existing components** instead of shipping literals (Step 4).
4. **Not pasting the reference verbatim.** It is a starting point, by its own instructions.

> **Runs in Claude Code, not Cowork.**
>
> Needs a Node toolchain — `pnpm`/`npm`, `next build`, `tsc`, `eslint` — against a working copy of
> the target repo, and (for the Figma-direct path) the Figma MCP.
>
> **If the toolchain or repo is not reachable, stop and say so.** Do not emit components you cannot
> type-check; a build that does not run is not a deliverable.

## Two input modes

1. **Spec-driven (default).** Read the component's `.spec.yaml` + `tokens.json` from
   `ds-figma-extract` or `ds-spec-extract`. Do not read the Figma file or the React export except
   where the spec is ambiguous — and log every such fallback as a spec gap. This is what makes the
   spec worth having.
2. **Figma-direct fidelity mode.** No spec, single target: drive `figma-design-to-code` yourself
   (load it, `get_design_context`, adapt). Faster for one-offs. You lose the durable token system
   and spec-based verification — an explicit trade, recorded, not a default.

Read `references/project-conventions.md` first, always. Seed a fresh copy per repo — the rules are
what this skill enforces; the instance values are what a fresh agent needs to not re-derive the repo.

## Do not fight `figma-design-to-code`

When the source is Figma, its skill already produced a good React+Tailwind reference with a hint
priority (Code Connect > docs > annotations > tokens > raw). Your job is to **adapt**, per that
skill's step 2:

- **Reference, never paste.** The output is a starting point; conform it to this repo's component
  library, file layout, and conventions.
- **Reuse before generating.** A Code Connect mapping means the component already exists — import it,
  do not regenerate it. Same for the project's tokens and layout primitives.
- **Assets by its G5 rules.** Render every icon/image from its export, preserve outer+leaf geometry,
  download bytes for committed code (the ~7-day URLs expire). Do not placeholder.

## Workflow

### Step 0 — Orient

Read `references/project-conventions.md`, then the spec's `CONTRACTS.md`/`FIGMA-GAPS.md` (whichever
the source produced) end to end. Note the frozen files (shared theme, `layout.tsx`, providers) you
must not edit — file a change request instead, exactly as the Leptos skill does.

### Step 1 — The Server/Client boundary FIRST

This is the decision the whole port hinges on, and the one most easily gotten wrong. Before writing
a component, classify it (see `references/react-to-nextjs.md`):

- **Server Component (default in App Router).** No hooks, no event handlers, no browser APIs. Can be
  `async` and fetch directly. Most presentational and layout components are these.
- **Client Component (`"use client"`).** Anything with `useState`/`useEffect`, an `onClick`, a
  browser API, or a Context provider. The directive marks a **boundary**: everything imported into a
  client component is bundled client-side.

The spec's `behavior_contract` tells you which is which — a component that `owns_state` or has
`callbacks` is a client component; a pure presentational one is a server component. **Push `"use
client"` as far down the tree as possible.** A `"use client"` on a layout drags the whole subtree
into the client bundle — the single most common Next.js performance mistake.

### Step 2 — State model

For each client component, from the spec's `states.machine`:

- Each parallel region → its own `useState` (or a `useReducer` when transitions are non-trivial).
  Do not collapse orthogonal regions into one state object — same anti-pattern the Leptos skill
  warns about, opposite framework.
- Model the machine explicitly: a discriminated union type per region and a reducer that makes
  illegal transitions unrepresentable. The statechart subset maps directly to a TS union.
- Derived values are computed in render (React re-renders; no memo needed unless measured hot).
- `TBD-user`/`FIGMA-GAPS` items — do not invent the missing transition. Stub it, cite the gap ID in
  a comment, and surface it.

### Step 3 — De-absolute the layout (the biggest job from a Figma source)

Verified 2026-07-19: `get_design_context` emits **every layer absolutely positioned** with
percentage insets — `absolute inset-[12.96%_0_12.96%_22.5%]`, fixed `h-[54px] w-[360px]` on the
root. That is a faithful transcription of a Figma canvas and it is **not** usable React: it does not
reflow, does not respond, and breaks the moment content length changes.

**Converting absolute positioning to flow layout is the primary translation task here** — the
analogue of "translate reactivity semantics" in the Leptos port. Work from the spec's `layout.md`
intent and the screenshot, not the inset percentages:

- A row of items → `flex` with a `gap`, not three absolutely-placed children.
- Label beside an icon → `inline-flex items-center gap-2`.
- Fixed root dimensions → intrinsic sizing plus constraints (`min-h`, `max-w`) unless the element is
  genuinely fixed (an icon, an avatar).
- Percentage insets → padding, gap, or grid tracks that express the *intent* the percentages encode.

Keep absolute positioning only where the design truly means it: an overlay, a badge pinned to a
corner, a decorative layer. Then it is `relative` parent + `absolute` child, not a whole subtree.

### Step 4 — Styling: bind the theme, do not ship literals

- Emit Tailwind utility classes bound to the project's theme. The spec ships `theme.css` (`@theme`,
  Tailwind v4) generated from `tokens.json` — **install it as the project's theme**, then reference
  its tokens by class (`bg-accent`, `text-primary`), never re-derived hex.
- **Expect the reference to be entirely arbitrary values** (`text-[36px]`, `text-black`,
  `font-['Philosopher:Regular']`) — confirmed on a real file. That is normal for the *reference*;
  the design-to-code skill hands you literals and expects conversion. It is a defect only in your
  **final output**: a spec token binding `{component.button.bg}` must become a themed utility, not
  `bg-[#2563eb]`.
- **Fonts come back as `font-['Family:Weight']`** (e.g. `font-['PingFang_HK:Ultralight']`). Map to
  `next/font` and reference through the theme's `--font-*` namespace — see
  `references/react-to-nextjs.md`.
- Where the design had **no token layer** (common — see `ds-figma-extract` Step 2 case B), the spec's
  tokens are all `inferred: true`. They are still the binding surface: bind to them, and surface that
  they need designer review rather than silently shipping invented names as fact.
- See `references/react-to-nextjs.md` for Tailwind v4 specifics (CSS-first `@theme`, no
  `tailwind.config.js` by default) and the dark-mode mechanism.

### Step 5 — Data boundary

The spec's ownership lives in `behavior_contract`. In Next.js:

- Data fetching belongs in **Server Components** (`async` components) or **Server Actions**
  (`"use server"`) / route handlers — never fetched ad hoc inside a client component.
- A `CONTRACTS.md` data-fetch contract (from a code-export source) becomes a Server Action or a
  `fetch` in a server component; cite the `C-XX` id at the implementation site.
- From Figma, there is **no** data ownership in the design — it is `TBD-user`, resolved from the
  target repo's existing patterns, not invented.

### Step 6 — Preview surface

Every component/state the verify manifest names must be reachable without code edits:

- A route or a dedicated preview page rendering each component in each key state (a Storybook story
  or a `/preview` route). Give preview DOM nodes `id`s equal to the `verify.yaml` state ids — the
  screenshot harness selects by them, same contract as the Leptos `/ds` route.
- States needing data: seed via props on the preview route, not by editing the component.

### Step 7 — Check discipline

After each component, before marking done:

```sh
pnpm exec tsc --noEmit          # types (or: npx tsc --noEmit)
pnpm exec next lint             # eslint incl. next/core-web-vitals
pnpm exec next build            # the real gate: RSC boundary + serialization errors surface here
```

`next build` is the equivalent of the Leptos "release build early" rule — RSC boundary violations
(passing a function prop from a server to a client component, importing server-only code into a
client bundle) frequently pass `tsc` and `next lint` and fail only at build. Run it as soon as the
first component renders; keep it green.

### Step 8 — Record

Per component or wave:

- Change requests for any frozen-file workaround (shared theme, `layout.tsx`, providers).
- Known deltas — every accepted deviation from the design, with its decision.
- Gap-id citations in code (`grep` for `C-[0-9]` or the `FIGMA-GAPS` id) at every contract/gap site.

Then hand off to `ds-visual-verify`.

## Anti-patterns

- **`"use client"` at the top of the tree.** Drags everything into the client bundle. Push it to the
  leaves that actually need interactivity.
- **Passing non-serializable props across the boundary.** A server component cannot hand a function
  or a class instance to a client component. `next build` catches it; design for it up front.
- **Inlining token values.** `bg-[#2563eb]` in the output means the theme binding was skipped. Bind
  through `@theme`.
- **Pasting `get_design_context` output verbatim.** It is a reference. Conform it to the repo, reuse
  existing components, or you have shipped generic code with the project's data in it.
- **Inventing state Figma did not encode.** If the spec says `TBD-user`, the component says `TBD-user`
  — a stub and a surfaced question, not a guess.
- **Skipping `next build` until the end.** RSC boundary errors are the release-build traps of this
  target; two-thirds of them never show in `tsc`.
