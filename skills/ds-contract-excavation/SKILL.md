---
name: ds-contract-excavation
description: >
  Enumerate every implicit contract and ambient dependency in an AI-generated frontend export
  (Claude Design, v0, Lovable, or similar). Use this skill whenever the user wants to port,
  audit, refactor, or "understand the structure of" an AI-generated codebase, mentions "hidden
  dependencies," "implicit contracts," or asks what a generated export assumes about its
  environment. Always run this BEFORE any porting or spec-extraction work on an AI-generated
  export.
---

# Contract Excavation

> **Mostly a Claude Code skill.**
>
> This skill needs a checked-out export tree on disk — nothing client-specific, so it **will** run in Cowork provided the
> files are in a connected folder. Say which environment you ran in when you record results.
>
> But the pipeline it feeds is Claude Code-only: `ds-fetch` and `ds-sync` need the `claude-design`
> MCP server (registered in Claude Code's config, which Cowork does not read), and the codegen and
> verify skills need the Rust toolchain and dev servers on the user's machine. Cowork's sandbox is
> a separate filesystem from the user's computer.
>
> **If the inputs are not reachable, stop and say so.** Do not produce a partial result that looks
> complete — tell the user to run this in Claude Code, in the target repo.


An AI-generated frontend export is written against a scaffold that is not in the repo. The
generator assumed a runtime — a preview harness, an injected storage object, a Tailwind config,
a set of magic export names — and that scaffold does not ship with the export. Every one of
those assumptions is a **contract**: something the code depends on but does not define.

Ports fail at these seams, not at the syntax. Syntax translation is mechanical and a compiler
will catch its mistakes. A dropped contract compiles fine and is wrong at runtime, or worse, is
silently missing a feature nobody notices until production. Find them all before anyone writes a
line of target-framework code.

## Output

A single `CONTRACTS.md` at the repo root. It is a required input to `ds-spec-extract`. Do not
proceed to spec extraction or codegen until it exists and its `TBD-user` decisions are resolved
by a human.

## Step 1 — Inventory

List every file. Classify each into exactly one bucket:

| Bucket | Meaning |
|---|---|
| `component` | Renders UI |
| `data` | Fixtures, mocks, seed data, type definitions of data shapes |
| `glue` | Runtime scaffolding: entry points, providers, routers, storage wrappers |
| `asset` | Images, fonts, icons, media |
| `config` | Build, lint, framework, styling config |

Report counts per bucket. **Anything you cannot classify is itself a finding** — an
unclassifiable file is usually a contract with the missing scaffold, not a mystery to shrug at.

## Step 2 — Sweep

Read `references/known-scaffolds.md` **first**. It catalogs contract signatures per generator so
the sweep starts from known patterns instead of zero. Identify which generator produced this
export (that file lists fingerprints), run that generator's checks, then run the generic sweep
below regardless — generators drift, and the catalog is always behind reality.

Every contract must carry **file:line evidence**. A contract without a citation is a guess.
Do not record guesses; either find the line or drop the claim.

Sweep for:

1. **Globals** — read or written. `window.*`, `globalThis.*`, module-level mutable singletons,
   injected runtime objects. Grep: `window\.`, `globalThis`, `declare global`, `as any`.
2. **Magic names** — functions or exports assumed to exist by naming convention, called by a
   runtime that isn't here. Signature: an export nothing in-repo imports. Build the import graph
   and look at the orphans.
3. **Data-shape assumptions** — a component reads field `y` that nothing in-repo produces or
   types. Signature: property access on `any`, optional chaining into shapes with no definition,
   types declared inline at the point of use rather than at the point of production.
4. **Storage/persistence** — artifact-style `window.storage`, `localStorage`/`sessionStorage`
   conventions, IndexedDB wrappers, anything treating a global as a database.
5. **Fetch/data flow** — who fetches, who owns loading state, who owns error state. Record the
   **ownership**, not just the call site. This becomes `behavior_contract` in `ds-spec-extract`,
   and ownership is the part that does not survive a naive port.
6. **CSS/theming** — expected CSS custom properties, Tailwind config the export presumes, class
   names with no definition in-repo, `:root` vars set by something outside the export.
7. **Event conventions** — custom events, pub/sub, callback signatures, `addEventListener` with
   no dispatcher in-repo (or a dispatcher with no listener).

## Step 3 — Classify

Each contract gets exactly one decision:

- **shim** — reproduce it in the target. The contract is real and load-bearing.
- **redesign** — replace with an explicit interface. The contract is real but implicit; the port
  is the moment to make it explicit.
- **drop** — dead in the target context. The scaffold needed it; the target does not.

When the decision depends on product intent you cannot infer from code, mark it `TBD-user` and
state the specific question. Do not guess. A wrong `drop` is silent feature loss downstream, and
it will be discovered late, in `ds-visual-verify`, as a confusing screenshot diff.

## Step 4 — Write CONTRACTS.md

One section per contract. Number them `C-01`, `C-02`, … — `ds-spec-extract` and the codegen
skills cite these IDs, so the numbers are a public interface. Do not renumber across runs;
append.

````markdown
### C-07 — `window.storage` treated as persistent KV

**Kind:** storage
**Evidence:** `src/hooks/useCart.ts:14`, `src/hooks/useCart.ts:31`, `src/App.tsx:8`
**Producers:** none in repo — assumed injected by the preview harness
**Consumers:** `useCart`, `useWishlist`
**Assumed shape:** `{ get(k: string): Promise<unknown>; set(k: string, v: unknown): Promise<void> }`
**Decision:** redesign
**Rationale:** Target has a real server boundary; this becomes explicit persistence behind a
server function rather than an ambient global.
**Open question:** —
````

End the file with:

- a **Decision summary** table (ID, kind, decision) — the at-a-glance view;
- an **Open questions** section listing every `TBD-user` in one place, so the human review pass
  is a single sitting rather than a scavenger hunt through the document.

## Step 5 — Grow the catalog

If you found a contract pattern not in `references/known-scaffolds.md`, add it there with a
signature others can grep for. The catalog is this skill's memory. An excavation that discovers a
new pattern and does not record it has thrown away its most durable output — the next run starts
from zero again.

## Anti-patterns

- **Summarizing instead of enumerating.** "The export assumes a storage layer" is not a contract;
  `window.storage.get` at `useCart.ts:14` with shape `{get, set}` is. The value here is
  exhaustiveness, not insight.
- **Deciding what you should ask.** `TBD-user` is a first-class outcome, not a failure. Three
  honest `TBD-user`s beat three confident wrong guesses.
- **Reading only the components.** Contracts live in the glue. The entry point, the providers,
  and the config files are where the scaffold's assumptions are densest.
- **Stopping at the first generator match.** Run the generic sweep even when the fingerprint is a
  confident match for a catalogued generator.
