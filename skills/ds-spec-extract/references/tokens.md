# Tokens

Rules for producing `spec/tokens.json` (W3C DTCG) and `spec/theme.css` (Tailwind `@theme`).

This is the **stable layer**. DTCG is a real standard with an ecosystem behind it; unlike the
component spec schema, it is not expected to churn. Version `tokens.json` independently of the
component schema — a component-spec revision must not invalidate token work.

---

## Three tiers

Tiers are not decoration. They are what makes the port survivable: codegen binds to
`component.*`, retheming happens at `semantic.*`, and `primitive.*` is the only place raw values
live.

**1. `primitive` — raw values, no meaning.**

```json
"primitive": {
  "gray": {
    "900": { "$type": "color", "$value": "#0a0a0a" },
    "500": { "$type": "color", "$value": "#737373" }
  },
  "space": {
    "2": { "$type": "dimension", "$value": { "value": 8, "unit": "px" } }
  }
}
```

Name by what it *is*. Never by what it is used for. `primitive.gray.900`, never
`primitive.text-color`.

**2. `semantic` — meaning, referencing primitives.**

```json
"semantic": {
  "text": {
    "primary":   { "$type": "color", "$value": "{primitive.gray.900}" },
    "secondary": { "$type": "color", "$value": "{primitive.gray.500}" }
  }
}
```

Name by role. This is the retheming surface — a dark theme swaps `semantic.*` targets and
everything downstream follows.

**3. `component` — per-component, referencing semantics.**

```json
"component": {
  "button": {
    "primary": {
      "bg":       { "$type": "color", "$value": "{semantic.bg.accent}" },
      "bg-hover": { "$type": "color", "$value": "{semantic.bg.accent-hover}" }
    }
  }
}
```

Component specs bind **only** to `component.*` — or to `semantic.*` when the value is genuinely
shared and has no component-specific meaning (a focus ring). Never to `primitive.*`. A component
spec reaching into `primitive` has skipped the meaning layer, which is where the port's
retheming ability lives.

**Never skip a tier upward.** A `component.*` token whose `$value` is a literal rather than a
reference is a bug — it means a value with no semantic identity, and it will not survive
retheming.

---

## DTCG requirements

- Every token: `$type` and `$value`. No exceptions.
- `$type` from the DTCG set: `color`, `dimension`, `fontFamily`, `fontWeight`, `duration`,
  `cubicBezier`, `number`, `shadow`, `border`, `typography`, `transition`.
- References use `{path.to.token}` in `$value`.
- Group-level `$type` is inherited by children — use it, it cuts noise.
- `$description` on anything non-obvious, and on **everything** in `semantic`. A semantic token
  whose meaning is not written down is a naming argument waiting to happen.
- Composite types (`shadow`, `typography`, `border`) take object values. Do not decompose them
  into loose scalars — the composite is the standard's unit and tooling expects it.

---

## Inferred tokens

Exports use raw values. You must invent semantic names for them. **Flag every one:**

```json
"semantic": {
  "bg": {
    "accent": {
      "$type": "color",
      "$value": "{primitive.blue.600}",
      "$description": "Primary action background",
      "$extensions": {
        "dsPort": {
          "inferred": true,
          "evidence": ["src/components/Button.tsx:34", "src/components/Cta.tsx:12"],
          "confidence": "high",
          "note": "Same hex used for all primary CTAs; semantic name invented."
        }
      }
    }
  }
}
```

`confidence`:

- `high` — one value, consistent role, unambiguous. (Same blue on every primary CTA.)
- `medium` — consistent role, some outliers you decided were mistakes. **Say which.**
- `low` — you are guessing. The value appears in contexts that do not share a role, or the
  grouping is aesthetic rather than semantic.

`ds-port-orchestrator` pauses for human review of every `inferred: true` before codegen. That
gate only works if the flag is trustworthy — flag liberally. An over-flagged token costs a
reviewer three seconds; an unflagged wrong one propagates into every component that binds it and
surfaces later as an unexplained screenshot diff.

**Do not invent a semantic tier where none exists.** If two hexes differ by one digit and appear
in unrelated places, they are probably generator noise, not two semantic roles. Collapse them,
flag `confidence: medium`, and note the collapse. Inventing `semantic.bg.accent-alt` to preserve
a rounding artifact is how a token system becomes unusable.

---

## Near-duplicate values

AI exports are full of values that are almost the same: `#0a0a0a` and `#0b0b0b`, `15px` and
`16px`. Decide deliberately:

1. Cluster near-identical values.
2. For each cluster, ask whether the difference is **meaningful** (a real hierarchy) or
   **noise** (the generator drifting).
3. Collapse noise to one primitive. Record the collapse in `SPEC-GAPS.md` if you are not certain
   — this is exactly the kind of call a human should be able to audit later.
4. Keep meaningful differences and name both.

Silently preserving every distinct value produces a token file with 40 grays that is technically
faithful and practically worthless.

---

## Tailwind `@theme`

Emit `spec/theme.css` mapping tokens to Tailwind theme variables. Codegen binds token-derived
classes through this rather than re-deriving values.

```css
@theme {
  --color-text-primary: #0a0a0a;
  --color-bg-accent: #2563eb;
  --radius-control: 0.375rem;
  --spacing-2: 0.5rem;
}
```

Rules:

- Generate it **from `tokens.json`**, never by hand. It is a projection; hand-editing forks the
  source of truth.
- Resolve references — `@theme` takes concrete values, not `{...}` refs.
- Map from `semantic` and `component` tiers. `primitive` generally does not need to reach
  Tailwind; if it does, that suggests a missing semantic name.
- Follow Tailwind's namespace conventions (`--color-*`, `--spacing-*`, `--radius-*`,
  `--font-*`) — that is what makes the utility classes generate.

Note the constraint from `CONTRACTS.md` G-6: if the export assumed Tailwind classes with no
config backing them, those classes silently did nothing in the original. Do not faithfully
reproduce a no-op. Check whether the class was dead before porting it.

---

## Validation

1. Every token has `$type` and `$value`.
2. Every reference resolves; no cycles.
3. No `component.*` token with a literal `$value`.
4. No component spec binds to `primitive.*`.
5. Every `inferred: true` has `evidence` and `confidence`.
6. `theme.css` regenerates byte-identically from `tokens.json` (the emit is idempotent).

Failures → `SPEC-GAPS.md`.
