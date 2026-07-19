# Known Scaffold Contract Patterns

Catalog of contract signatures per generator. `ds-contract-excavation` reads this before every
sweep so it starts from known patterns instead of zero.

**Status legend.** Every entry is marked:

- `confirmed` — observed in a real export, with the fixture named.
- `likely` — strong prior from the generator's architecture, not yet confirmed against a fixture.
- `unfilled` — placeholder; fill from the first real export of this generator.

**Do not treat `likely` entries as fact during a sweep.** Use them as grep hints. If a `likely`
pattern holds in a real export, promote it to `confirmed` and name the fixture. If it does not,
delete it. A catalog that accumulates plausible-sounding fiction is worse than an empty one,
because it manufactures false confidence in exactly the step whose whole job is exhaustiveness.

---

## Identifying the generator

Fingerprint before you sweep. Check, in order:

1. `package.json` — name, scripts, and dependency fingerprints.
2. README / comment headers — generators often stamp provenance.
3. Directory shape — each generator has a characteristic layout.
4. Import graph orphans — exports nobody imports are the scaffold's call surface.

If no fingerprint matches, record the export as an unknown generator, run the generic sweep, and
open a new section here from what you find.

---

## Generic patterns (apply to every export)

These hold across generators and are the backbone of the sweep. All `confirmed` as a class —
these are properties of the "code generated against an absent harness" situation, not of any one
vendor.

### G-1 — Orphan exports (magic names)

**Signature:** an exported symbol that nothing in-repo imports.
**Why it happens:** the harness imports it by convention.
**Find it:** build the import graph; list exports with zero in-repo importers. Grep alone will
not find this — it needs the graph.

### G-2 — Ambient globals

**Signature:** `window.<name>` / `globalThis.<name>` read but never assigned in-repo.
**Find it:** `rg 'window\.|globalThis\.'`, then diff the read set against the assigned set. The
reads-minus-writes difference is the contract surface.

### G-3 — `declare global` / ambient type declarations

**Signature:** a `.d.ts` or inline `declare global` block typing something the repo never
implements. This is the generator writing down the contract explicitly — the highest-value find
in any sweep, because it is the scaffold's own documentation of itself.
**Find it:** `rg 'declare global|declare module|declare const'`.

### G-4 — Types declared at point of use

**Signature:** a component defines `interface Props { user: { name: string } }` inline, and
nothing in-repo ever produces a `user`. The data shape is assumed, not owned.
**Find it:** for each inline type describing domain data, search for a producer. No producer
means a contract with whatever was going to supply it.

### G-5 — CSS custom properties with no definition

**Signature:** `var(--foo)` used, `--foo` never set in-repo.
**Find it:** collect `var\(--[a-z-]+\)` reads, diff against `--[a-z-]+:` definitions.
**Note:** this one matters disproportionately — it silently degrades to the fallback value or to
nothing, so the port looks fine and is subtly wrong. Downstream, `ds-spec-extract` treats every
undefined var as an inferred token needing human review.

### G-6 — Tailwind classes with no config backing

**Signature:** arbitrary or extended class names (`bg-brand-500`, `rounded-4xl`) with no
`tailwind.config` or `@theme` block defining them.
**Find it:** extract the class-name set from `className`/`class` strings, diff against the
resolved Tailwind theme. If there is no config in-repo at all, the entire non-core class set is
one contract.

### G-7 — Listener without dispatcher (or the reverse)

**Signature:** `addEventListener('app:something')` with no `dispatchEvent` in-repo, or a
dispatch nobody hears.
**Find it:** `rg 'addEventListener|dispatchEvent|CustomEvent'`, pair them up, report the
unpaired.

### G-8 — Fetch with unowned loading/error state

**Signature:** a component fetches and renders, but loading/error handling lives somewhere that
is not in the repo — an error boundary, a suspense boundary, a harness-level wrapper.
**Find it:** for each fetch call site, ask who renders the loading state and who catches the
throw. If the answer is "nothing in-repo," that is the contract.
**Note:** record the **ownership**, not the call site. Ownership is what a naive port loses.

---

## Claude Design

**Status:** `unfilled`
**Fingerprint:** _TBD — fill from first real export._

The doc's test-fixture plan calls for a real Claude Design export, ideally one already ported by
hand so there is a known-good reference. Fill this section from that fixture rather than from
priors.

Expected areas to probe (all `likely`, none confirmed — treat as questions, not answers):

- Artifact-style storage contract (`window.storage`-shaped KV, async get/set). The doc's Skill 1
  spec calls this out by name, so it is worth probing first.
- Single-file component conventions and what the harness supplies around them.
- Tailwind core-utilities-only assumption — the artifact runtime historically ships a base
  stylesheet with no compiler, meaning arbitrary values silently do nothing (see G-6).
- Browser storage APIs being unavailable in the artifact runtime, which pushes state into
  in-memory patterns that may look like bugs but are harness accommodations.

Record for each: the exact global/name, its shape, evidence lines, and whether the harness or the
export owns it.

## v0 (Vercel)

**Status:** `unfilled`
**Fingerprint:** _TBD — fill from first real export._

Probe areas (`likely`): Next.js App Router assumptions, shadcn/ui component vendoring and its
`@/components/ui` path alias, `next/*` imports that assume framework-level providers, server/
client component boundary markers (`'use client'`) that encode an ownership contract.

## Lovable

**Status:** `unfilled`
**Fingerprint:** _TBD — fill from first real export._

Probe areas (`likely`): Vite + React + shadcn baseline, Supabase client assumed to be configured
elsewhere, env-var-driven config with no `.env` in the export.

---

## Adding a new entry

Use this shape. Keep the grep in the entry — a pattern you cannot search for is a pattern the
next sweep will miss.

````markdown
### CD-3 — <short name>

**Status:** confirmed (fixture: `fixtures/claude-design-storefront`)
**Kind:** storage | global | magic-name | data-shape | fetch | css | event
**Signature:** <what it looks like in source>
**Grep:** `<ripgrep pattern>`
**Shape:** <the assumed interface, if any>
**Usual decision:** shim | redesign | drop — and why
````
