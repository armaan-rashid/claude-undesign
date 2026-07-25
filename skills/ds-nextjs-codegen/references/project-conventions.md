# Project conventions — Next.js seed

Per-project state. At the start of a port, copy this into the target repo (or the skill context),
keep every **Rule** line, and replace every **This project** block with the target's real values.
The rules are what the skill enforces; the instance values are what a fresh agent needs to not
re-derive the repo.

This seed carries **current-idiom defaults** (App Router, RSC, Tailwind v4, TypeScript). No real
Next.js reference repo has seeded it yet — unlike the Leptos conventions, which came from the
library-of-light port. So every value below is a **default to confirm against the actual repo**, not
an observed fact. Confirm before relying.

## 0. Identity

| Field | Default — confirm per repo |
|---|---|
| Target framework | Next.js App Router (14/15-era) |
| React | 18 or 19 |
| Language | TypeScript, `strict: true` |
| Styling | Tailwind v4 (CSS-first `@theme`) |
| Package manager | detect from lockfile (`pnpm-lock.yaml` → pnpm, `package-lock.json` → npm, `yarn.lock` → yarn) — **use the repo's**, do not assume |
| Spec source | `spec/` from `ds-figma-extract` or `ds-spec-extract` |
| Verify manifest | `spec/verify.yaml` |

## 1. Workspace layout

**Rule:** App Router structure; shared shell and providers are frozen during parallel work; each
route/component is self-contained and does not edit shared files.

Default (confirm):

```
app/
  layout.tsx            # root shell — Server Component; FROZEN in parallel waves
  page.tsx              # home
  <route>/page.tsx      # one dir per route
  globals.css           # @import "tailwindcss" + @theme (the spec's theme.css lands here)
components/
  ui/                   # design-system components (bind Code Connect mappings here)
lib/                    # server-only utils (mark `import "server-only"`), data access
public/                 # committed assets (downloaded from Figma exports)
```

## 2. Design-system reuse

**Rule:** components never re-implement a design-system primitive that already exists; reuse it. A
Code Connect mapping is binding — import the mapped component, do not regenerate it.

Default: check `components/ui/` and any path named in a `get_code_connect_map` result before
generating. If a close-but-not-identical variant is needed, prefer a small local component + a filed
change request over forcing a prop into the shared one (the Leptos port's `DcEditable` precedent).

## 3. Server/Client boundary

**Rule:** Server Component by default; `"use client"` only on interactive leaves; never on `layout`
or a whole page. Providers are small client wrappers, not client-ifying their subtree's authorship.

Default: see `references/react-to-nextjs.md`. The `behavior_contract` in each spec decides the kind.
`next build` is the gate that enforces it.

## 4. Data boundary

**Rule:** all data access in Server Components (`async`) or Server Actions (`"use server"`) / route
handlers; client components receive data as serializable props, never fetch ad hoc.

Default: a `CONTRACTS.md` data contract → a Server Action in `lib/` or a fetch in the server
component, cited by `C-XX` at the site. From Figma, data ownership is `TBD-user` — resolve from the
repo's existing patterns (an ORM, a fetch wrapper, an API route), never invent one.

## 5. Credentials / config

**Rule:** secrets in env (`.env.local`, gitignored) with a committed `.env.example`; server-only
modules marked `import "server-only"` so a secret can never reach the client bundle; missing config
degrades gracefully where the design has an offline/empty state.

Default: `process.env.*` read only in server code; `NEXT_PUBLIC_*` is the only client-visible prefix
and holds nothing secret.

## 6. Multi-agent protocol: frozen files + change requests

**Rule:** shared files frozen while component agents run in parallel; workarounds are local and each
is filed as a change request naming its own location so a supervisor can delete it when the request
lands. (Same protocol as the Leptos skill.)

Default frozen set: `app/layout.tsx`, `app/globals.css` (the theme), any `components/ui/*` shared
primitive, providers. Change requests: `docs/ds-change-requests-<component>.md`.

## 7. Gap/contract citations

**Rule:** every implementation of a `CONTRACTS.md` decision or a `FIGMA-GAPS` resolution cites its
id in a comment at the implementation site; a stubbed `TBD-user` is a visible comment, never a
silent guess.

Default: `grep -rn "C-[0-9]\|FIGMA-GAP" app components lib` hits every contract/gap-bearing file.

## 8. Build & test commands

**Rule:** type-check, lint, and **build** every component before marking done; build early and keep
it green — RSC boundary errors surface only at build.

Default (swap `pnpm` for the repo's manager):

```sh
pnpm exec tsc --noEmit
pnpm exec next lint
pnpm exec next build            # the real gate
pnpm dev                        # local server, default :3000
```

If the repo has a test runner (Vitest/Jest/Playwright), run it too; pin any spec-surface behavior
(token values must not be asserted as literals — bind through the theme) the same way the Leptos
port unit-tested storage shapes.
