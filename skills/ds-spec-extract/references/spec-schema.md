# Component Spec Schema

**Current `specVersion`: `0.1.0`**

This is the **volatile layer** of the pipeline. The token layer rests on W3C DTCG, a real
standard that will outlive this project. Nothing comparable has won for component specs, so this
schema is a local invention and should be expected to change.

That expectation is designed for:

- The schema lives **here**, not in `SKILL.md` prose. Revising the format means editing this
  file, not rewriting the skill.
- Every spec file carries `specVersion` at its root. Version this schema independently of
  `tokens.json` — a component-schema revision must not invalidate token work.
- Downstream skills (`ds-*-codegen`, `ds-visual-verify`) read `specVersion` and may refuse a
  major version they do not know. Failing loudly on an unknown version beats misreading it.

**Versioning rule (semver):** additive optional field → minor. Renamed/removed/retyped field, or
changed meaning of an existing field → major. On a major bump, add a migration note at the bottom
of this file.

---

## File shape

```yaml
specVersion: 0.1.0
name: Button
kind: component        # component | composite | page
source:
  evidence: [src/components/Button.tsx]
  bundle: null         # handoff bundle node id, if the spec came from a bundle
```

`source.evidence` is provenance, not an instruction to read those files. Codegen must not open
them. It exists so a human reviewing the spec can check the extraction, and so a spec gap can be
traced back.

---

## `anatomy`

The part structure. Named parts, nesting, required vs optional.

```yaml
anatomy:
  root:
    required: true
    parts:
      - name: leadingIcon
        required: false
        note: renders before label; hidden when loading
      - name: label
        required: true
      - name: spinner
        required: false
        note: replaces leadingIcon slot when async.loading
```

Parts are **named surfaces**, not DOM elements. Do not write `div` / `span` here — the target may
not have them. If a part's element type is semantically load-bearing (a `button` must be a
button), that belongs in `a11y`, not `anatomy`.

---

## `props`

Typed, with defaults. **Spec-level types, not TypeScript.**

```yaml
props:
  - name: variant
    type: enum
    values: [primary, secondary, ghost]
    default: primary
  - name: size
    type: enum
    values: [sm, md, lg]
    default: md
  - name: disabled
    type: boolean
    default: false
  - name: onPress
    type: callback
    signature: "() -> void"
    note: not fired when disabled or loading
```

Allowed `type`: `enum`, `boolean`, `string`, `number`, `callback`, `node`, `ref`, `object`.
For `object`, give a `shape`. For `callback`, give a `signature` in arrow notation — it is
target-agnostic and reads the same in Rust, Swift, and TS.

`node` means "renderable content supplied by the caller" (React `children`, Leptos `Children`,
SwiftUI `@ViewBuilder`). Do not call it `children` unless the caller-supplied slot is genuinely
singular and unnamed.

---

## `variants`

The axes and their values, and what each axis *means*. If an axis is purely a token swap, say so
— codegen can then generate it mechanically instead of branching.

```yaml
variants:
  - axis: variant
    values: [primary, secondary, ghost]
    affects: tokens-only     # tokens-only | anatomy | behavior
  - axis: size
    values: [sm, md, lg]
    affects: tokens-only
```

`affects: anatomy` or `affects: behavior` is a warning sign worth a second look — a variant that
changes structure or behavior is often two components wearing a trenchcoat.

---

## `tokens`

Bindings into `tokens.json`. **Bind by path. Never by literal value.** A literal here defeats the
entire layered design.

```yaml
tokens:
  root:
    background: "{component.button.primary.bg}"
    color: "{component.button.primary.fg}"
    radius: "{semantic.radius.control}"
    paddingX: "{component.button.md.padding-x}"
  root@hover:
    background: "{component.button.primary.bg-hover}"
  root@disabled:
    opacity: "{semantic.opacity.disabled}"
```

Key syntax: `<part>` or `<part>@<state>`. The `@state` must name a state that exists in
`states.machine` (or a parallel region's state). A binding referencing a state the machine does
not define is a validation error — that mismatch is the most common way a spec goes quietly
wrong.

---

## `states.machine`

See `statechart-subset.md`. It is an XState-compatible subset — states, events, transitions,
guards, and parallel regions, and nothing else.

Read that file before writing a machine. Constructs outside the subset produce specs codegen
cannot lower.

---

## `behavior_contract`

Ownership. Who fetches, who holds state, what callbacks fire when. Sourced largely from
`CONTRACTS.md` — **cite the contract IDs.**

```yaml
behavior_contract:
  owns_state: [async]           # which machine regions this component owns
  delegates_state: [availability]  # controlled by caller via props
  fetches: false
  contracts:
    - id: C-07
      relation: consumes
      note: cart persistence; redesigned to explicit server boundary in target
  callbacks:
    - name: onPress
      fires_when: "interaction.pressed -> interaction.idle AND availability.enabled AND async.idle"
      note: suppressed in all other combinations
```

This section is the part a naive port loses. Syntax translates; ownership does not. Be specific:
"the parent holds `disabled`" is ownership, "the button can be disabled" is not.

---

## `a11y`

```yaml
a11y:
  role: button
  element: button        # when the semantic element is load-bearing
  aria:
    - attr: aria-busy
      when: "async.loading"
    - attr: aria-disabled
      when: "availability.disabled"
  focus:
    visible: "{semantic.focus.ring}"
    trap: false
  keyboard:
    - key: Enter
      sends: PRESS
    - key: Space
      sends: PRESS
```

`sends` names an **event in the machine** — that is the link between keyboard handling and
`states.machine`, and it is what lets `ds-visual-verify` replay keyboard interaction against the
spec rather than against the React implementation.

---

## Validation

Before emitting JSON, check:

0. **No boolean keys in the emitted JSON.** YAML 1.1 resolves bare `on`/`off`/`yes`/`no` to
   booleans, so an unquoted `on:` in a machine emits as `"true"`. See `statechart-subset.md`.
   Cheap to check, silent when missed.
1. `specVersion` present and known.
2. Every `tokens` binding resolves to a path in `tokens.json`.
3. Every `@state` in `tokens` names a real state in `states.machine`.
4. Every `sends` in `a11y.keyboard` names a real event in `states.machine`.
5. Every `contracts[].id` exists in `CONTRACTS.md`.
6. No literal color/size/spacing values anywhere in the file.
7. Machine validates against the subset (`statechart-subset.md`).

A failure here is a spec gap → `SPEC-GAPS.md`. Do not emit a spec that fails validation with the
failure unrecorded.

---

## Migration notes

_None yet — `0.1.0` is the initial schema._

When bumping major: record what changed, why, and the mechanical migration if one exists.
Downstream skills read this section to decide whether they can auto-upgrade an old spec.
