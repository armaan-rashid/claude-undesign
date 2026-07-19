# Statechart Subset

The notation for `states.machine` in a component spec.

**The rule: it is XState's shape, restricted to a subset.** Use XState's key names and semantics
so a machine validates in existing XState tooling (Stately's visualizer is genuinely useful for
human review of extracted specs — a reviewer can *see* the machine instead of parsing YAML). But
emit only the constructs listed below.

**Why a subset and not all of XState:** codegen lowers each machine to a target-native
representation (a Rust enum + signal, a Swift enum, etc.). Constructs outside the subset —
`invoke`, actors, `after`/delays, history states, `spawn`, action side-effects — have no clean
lowering and would either be silently ignored or force codegen to invent behavior. If `ds-spec-extract`
can express it, some codegen skill has to implement it. Keep the surface small.

**Why not a flat enum:** because component state is not one dimension. A button is
`idle | hover | pressed` **and independently** `enabled | disabled` **and independently**
`idle | loading | error`. Flattening those gives 18 states, most meaningless, and the meaningless
ones are unreachable in ways nothing checks. Parallel regions are the whole reason to take
XState's shape rather than invent one.

---

## In the subset

- `id`, `initial`, `states`
- `type: parallel` (and `type: final`)
- `on` — event → transition
- `target`, `cond` (guard, by name)
- Nested `states` (one level of hierarchy inside a region; deeper needs a good reason)

## Out of the subset

`invoke` · `actions` / `entry` / `exit` with side-effects · `after` / delayed transitions ·
`history` · `spawn` / actors · `context` (see below) · `always` / eventless transitions ·
internal vs. external transition distinction

If you need one of these, that is a signal — either the behavior belongs in `behavior_contract`
(ownership, callbacks) rather than the machine, or the subset genuinely needs to grow. **Do not
smuggle it in.** Record it in `SPEC-GAPS.md` and let the subset expand deliberately.

### On `context`

XState's `context` is extended state — arbitrary data alongside the finite state. It is out of
the subset because component data belongs in `props` (caller-owned) or is derived. If a machine
seems to need `context`, the data is almost always a prop you have not written down yet.

The exception worth watching: an error message string attached to an `error` state. Put it in
`props` as caller-owned, or note it in `behavior_contract`. Do not add `context`.

---

## `"on"` must be quoted

**In YAML, `on` is a boolean.** YAML 1.1 — which PyYAML and many other parsers implement —
resolves the bare token `on` to `true`. So this:

```yaml
idle:
  on:
    POINTER_ENTER: { target: hover }
```

parses to `{"idle": {True: {...}}}` and emits to JSON as `{"idle": {"true": {...}}}`. The machine
is now invalid XState, the transition is silently unreachable, and nothing complains — the YAML
was well-formed, it just meant something else.

**Always write `"on"` quoted.** This is the [Norway problem](https://hitchdev.com/strictyaml/why/implicit-typing-removed/)
(`NO` → `false`) in the one place it does maximum damage: the key that carries every transition in
every machine.

Related YAML 1.1 booleans to quote if they ever appear as keys or string values: `on`, `off`,
`yes`, `no`, `y`, `n`, `true`, `false`. State and event names in this pipeline are lowercase and
SCREAMING_CASE respectively, so `on` is the realistic collision — but if a component ever has an
`off` state, quote it too.

The emit step must use a parser whose behavior you have checked. Do not assume YAML 1.2 semantics.

## Shape

```yaml
states:
  machine:
    id: button
    type: parallel
    states:

      interaction:
        initial: idle
        states:
          idle:
            "on":
              POINTER_ENTER: { target: hover }
          hover:
            "on":
              POINTER_LEAVE: { target: idle }
              POINTER_DOWN:  { target: pressed }
          pressed:
            "on":
              POINTER_UP:    { target: hover, cond: actionable }
              POINTER_LEAVE: { target: idle }

      availability:
        initial: enabled
        states:
          enabled:
            "on":
              DISABLE: { target: disabled }
          disabled:
            "on":
              ENABLE: { target: enabled }

      async:
        initial: idle
        states:
          idle:
            "on":
              SUBMIT: { target: loading }
          loading:
            "on":
              RESOLVE: { target: idle }
              REJECT:  { target: error }
          error:
            "on":
              RETRY:   { target: loading }
              DISMISS: { target: idle }

    guards:
      actionable: "availability.enabled AND async.idle"
```

## Guards

Guards are **named and declared** in a `guards` block, referenced by name via `cond`. Never
inline a predicate.

Guard expressions are a deliberately tiny language: cross-region state references
(`region.state`), `AND` / `OR` / `NOT`, parens. Nothing else — no arithmetic, no prop access, no
function calls. A guard that needs more than this is describing ownership, and belongs in
`behavior_contract`.

Cross-region references are the point. `actionable: "availability.enabled AND async.idle"` is
exactly the constraint a flat enum cannot express without duplicating every transition.

---

## Region naming

Use these names when the dimension is present; consistency across specs lets codegen and verify
share machinery:

| Region | States | Meaning |
|---|---|---|
| `interaction` | `idle`, `hover`, `pressed` | Pointer state |
| `availability` | `enabled`, `disabled` | Whether the control accepts input |
| `async` | `idle`, `loading`, `error` | In-flight operation |
| `validity` | `unvalidated`, `valid`, `invalid` | Form field validation |
| `expansion` | `collapsed`, `expanded` | Disclosure |
| `selection` | `unselected`, `selected`, `indeterminate` | Selection |

Non-standard regions are fine — name them for the dimension, not the component.

---

## Where to draw the line

Not everything belongs in the machine. Ask: **does this dimension have transitions the component
mediates?**

- `interaction` — yes. The component decides `hover → pressed`.
- `availability` — usually **no**. `disabled` is typically a caller-owned prop with no
  component-mediated transition. Model it as a region only when the component can disable itself
  (e.g. it disables during its own `async.loading`). Otherwise it is a prop, and guards can still
  reference it — declare it in `behavior_contract.delegates_state`.
- Pure visual states with no transitions (`focus-visible` driven entirely by the platform) — no.
  Those are `tokens` bindings and `a11y`, not machine states.

The test: if a region has exactly two states and every transition into them comes from a prop
change rather than an event, it is a prop.

---

## Validation

1. **No boolean keys anywhere in the emitted JSON.** A `"true"` key means an unquoted `on` slipped
   through. Check this first — it is silent, and it invalidates every machine it touches.
2. Every `target` names a state in the same region.
3. Every `cond` names a declared guard.
4. Every guard expression references only real `region.state` pairs.
5. Every event named in `a11y.keyboard[].sends` appears in some `"on"`.
6. Every state is reachable: it is a region's `initial`, or some transition targets it.
7. No constructs from the out-of-subset list.
8. Machine parses as XState JSON. Emit the JSON and check it — this is the payoff for taking
   XState's shape, so do not skip it.

Failures → `SPEC-GAPS.md`.

---

## Growing the subset

The subset is expected to grow — it is sized for what codegen can lower **today**. To add a
construct: confirm every codegen skill can lower it, add it here with its lowering, and bump the
component spec `specVersion` minor (the schema gained expressiveness; old specs stay valid).

The failure mode to avoid is the subset growing implicitly, one spec at a time, because an
extraction run needed something and reached for it. That is how a small notation becomes all of
XState with none of the tooling.
