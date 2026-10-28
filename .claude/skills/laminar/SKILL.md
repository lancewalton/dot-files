---
name: Laminar UI Development
description: Laminar reactive UI patterns for Scala.js frontend. Use when implementing or modifying frontend components, form fields, or reactive list rendering in Laminar.
---

# Laminar UI Development

## Testing reactive logic

Extract the pure decision from the stream wiring so it unit-tests without a DOM (e.g. an
`event => Option[State]` mapping), and cover the Airstream/subscription/navigation wiring with a Cypress
E2E. Don't try to unit-test `EventStream`/`Var` plumbing directly.

## Reactive List Rendering

**ALWAYS** use the `split` operator with stable keys when rendering lists from signals. **NEVER** recreate DOM elements on signal updates.

### Why

When a `Signal[List[T]]` is mapped directly to DOM elements, Laminar recreates the entire DOM subtree on every signal emission. For form fields, this causes **focus loss on every keystroke** — the input element is destroyed and a new one is created, so the cursor disappears.

### Wrong — recreates DOM on every update

```scala
// BAD: entire list re-rendered on any change, inputs lose focus
itemsSignal.map { items =>
  items.map { item =>
    input(
      value := item.name,
      onInput.mapToValue --> { v => updateItem(item.id, v) }
    )
  }
}
```

### Correct — stable elements via split

```scala
// GOOD: split tracks items by stable key, only updates changed elements
children <-- itemsSignal.split(_.id) { (id, initial, itemSignal) =>
  input(
    controlled(
      value <-- itemSignal.map(_.name),
      onInput.mapToValue --> { v => updateItem(id, v) }
    )
  )
}
```

### Rules

1. **Every list signal rendering MUST use `split` with a stable key** (e.g. an ID field, not an index)
2. **Use `controlled`** for form inputs bound to signals — this pairs `value <-- signal` with an `onInput` handler
3. **Never use `.map` on a `Signal[List[T]]` to produce child elements** — this is always a bug for interactive content
4. **If items lack a natural key**, add a `UUID` or incrementing `ID` field when creating them

## Accessing Signal State in Click Handlers

Use `onClick.compose(_.withCurrentValueOf(signal))` to read the current value of a signal at click time. This avoids stale closures and removes the need for auxiliary `Var`s.

### Wrong — stale closure captures initial value

```scala
// BAD: signal value captured once at element creation, never updated
val currentItem = itemSignal.now()
onClick.mapTo(doSomething(currentItem)) --> observer
```

### Correct — reads signal at click time

```scala
// GOOD: withCurrentValueOf samples the signal on each click
onClick.compose(_.withCurrentValueOf(itemSignal).map { case (_, item) =>
  doSomething(item)
}) --> observer
```

### Common use cases

- **Conditional actions**: decide which event to emit based on current state (e.g. toggle selection)
- **Dynamic payloads**: include current signal values in the emitted event
- **Multiple signals**: chain `.withCurrentValueOf(s1).withCurrentValueOf(s2)` to sample several signals

## Controlled Inputs for Programmatic Updates

**ALWAYS** use `controlled` when an input's value can be updated programmatically (e.g. via a button click that calls `myVar.set(...)`). An uncontrolled input with `value := initial` only sets the DOM value once at creation — subsequent `Var.set()` calls update the `Var` but the DOM input remains unchanged.

### Wrong — DOM ignores programmatic Var updates

```scala
// BAD: value := sets DOM once; clicking "Suggest" updates the Var but the input still shows the old text
val pattern: Var[String] = Var("")
input(
  value := "",
  onInput.mapToValue --> pattern
)
button("Suggest", onClick --> { _ => pattern.set("inferred_value") }) // input doesn't change
```

### Correct — controlled input reflects Var changes

```scala
// GOOD: controlled binds the DOM input to the signal, so programmatic .set() is visible
val pattern: Var[String] = Var("")
input(
  controlled(
    value <-- pattern.signal,
    onInput.mapToValue --> pattern
  )
)
button("Suggest", onClick --> { _ => pattern.set("inferred_value") }) // input updates
```

### Rule

If a `Var` backing an input can be modified from **anywhere other than the input itself** (buttons, external events, signal updates), the input **must** use `controlled`. Reserve `value := initial` only for truly static initial values with no programmatic updates.

## Prefer Stream Transformation + `-->` Over `Var.set` in Event Handlers

When an event handler's only purpose is to transform a value and write it to a `Var`, lift the transformation into the stream instead of calling `Var.set` inside `-->`. Since `Var[A]` is an `Observer[A]`, `-->` can write directly to it.

### Common variants

| Smell | Refactored |
|---|---|
| `onChange.mapToValue --> { v => myVar.set(f(v)) }` | `onChange.mapToValue.map(f) --> myVar` |
| `onClick --> { _ => myVar.set(constant) }` | `onClick.mapTo(constant) --> myVar` |
| `stream --> { event => myVar.set(Some(event)) }` | `stream.map(Some(_)) --> myVar` |

### Wrong — imperative `Var.set` inside handlers

```scala
// BAD: transformation buried in handler
onChange.mapToValue --> { v =>
  if (v.isEmpty) selectedId.set(None)
  else selectedId.set(Some(ID(v)))
}

// BAD: ignoring event just to set a constant
onClick --> { _ => suggestResults.set(None) }

// BAD: wrapping in Some for no reason
inferenceEvents --> { event => fetchResult.set(Some(event)) }
```

### Correct — declarative stream transformation

```scala
// GOOD: map in the stream, --> writes to the Var
onChange.mapToValue.map(ID.fromString[MyType]) --> selectedId

// GOOD: mapTo for constant values
onClick.mapTo(None) --> suggestResults

// GOOD: map for simple wrapping
inferenceEvents.map(Some(_)) --> fetchResult
```

### Rule

If the right-hand side of `-->` contains only a single `Var.set` call, refactor to stream transformation + `-->`. Reserve `Var.set` for cases where multiple `Var`s must be updated, side effects beyond a single write are needed, or conditional logic determines whether to write at all.

## Never Use `Var.now()` Inside Observer Callbacks

`Var.set`/`Var.update` inside an Airstream transaction (any Observer callback) **queues** the new value — it is not committed until the transaction ends. Calling `Var.now()` after a `set`/`update` in the same callback returns the **pre-transaction** value, silently producing stale reads.

### Wrong — stale state from `now()` inside transaction

```scala
// BAD: pendingVar.now() returns pre-update value after set
val handler: Observer[Event] = Observer { event =>
  val current = pendingVar.now()
  val updated = current - event.id
  pendingVar.set(updated)
  if pendingVar.now().isEmpty then  // STILL SEES OLD VALUE — never true!
    complete()
}
```

### Correct — use `withCurrentValueOf` at the stream pipeline

```scala
// GOOD: state sampled before the transaction begins
eventStream
  .withCurrentValueOf(pendingVar.signal)
  --> { (event, current) =>
    val updated = current - event.id
    pendingVar.set(updated)
    if updated.isEmpty then complete()
  }
```

### Rules

1. **Never read `Var.now()` after `set`/`update` in the same Observer** — the value hasn't committed yet
2. **Use `withCurrentValueOf`** to sample signal state at event time, before the transaction starts
3. **If you must read state inside an Observer**, read it **once at the top** before any writes, compute the new state locally, then write once — never read back after writing
4. **`combineWith` + `withCurrentValueOf` flattens tuples** — `stream.withCurrentValueOf(sig1.combineWith(sig2))` produces `EventStream[(Event, A, B)]`, not `EventStream[(Event, (A, B))]`

## Read a Control's Change ONCE — a Second Listener Sees the Reset Value

Two `-->` bindings off the same `onChange` run as two DOM listeners, in order. If the first writes a Var that feeds
the control's own `value <--`, that write runs a transaction synchronously and puts the DOM value back **before** the
second listener reads `mapToValue`. The second then acts on the old value — usually by doing nothing. (2026-10-28: a
"Skills asked" select sent no PATCH; pure-decision unit tests were green, only Cypress showed it.)

```scala
// Wrong — the commit listener reads the select after `picking` reset it
chosen.map(_ == Mode.Specific) --> picking,
chosen.compose(_.withCurrentValueOf(filter$).flatMapSwitch(commitFor)) --> ()

// Correct — one observer reads the choice, sets local state, and emits the request onto a bus
onChange.mapToValue.compose(_.withCurrentValueOf(filter$)) --> { case (v, filter) =>
  requestFor(filter, v).foreach(requests.emit)
  picking.set(v == "specific")
},
requests.events.flatMapSwitch(commit) --> ()
```

## An EventStream Has No Current Value — Share a Signal, Never a Late Subscription

An Airstream `EventStream` carries no current value and does **not** replay. A second subscriber that arrives
after the stream has already been started by a first one receives **nothing** until the next emission — which,
for a mount-time feed or a GET-and-refresh, may never come.

This fails **silently**. No error, no log, no exception: the feature simply does nothing, which reads to the
user as "the app ignored me". It is one of the hardest classes of bug to spot by reading, because both
subscriptions look perfectly reasonable in isolation.

### Wrong — the second consumer is a late subscriber

```scala
// BAD: `children <--` starts the stream; the per-row handler subscribes AFTER and never sees a value,
// so `combineWith` never produces and the drop silently does nothing.
tbody(
  children <-- sortedData.map(...).split(...) { ... =>
    tr(
      onDrop.preventDefault
        .compose(_.combineWith(sortedData.take(1)).flatMapSwitch { ... }) --> ()
    )
  }
)
```

### Correct — one Signal, shared by every consumer

```scala
// GOOD: a Signal memoises its current value, so whichever consumer starts it, both read the same list.
val rows: Signal[List[T]] = sortedData.startWith(Nil)

tbody(
  children <-- rows.map(...).split(...) { ... =>
    tr(
      onDrop.preventDefault
        .compose(_.withCurrentValueOf(rows).flatMapSwitch { ... }) --> ()
    )
  }
)
```

### Rules

1. **If two consumers need the same stream, convert once to a `Signal` and share that one instance.** Do not
   subscribe to the stream twice.
2. **Do not "fix" it by adding a second binder** (`stream --> someVar.writer` alongside `children <-- stream`).
   That only moves the problem: Laminar mounts binders in the order they are added, so whichever mounts first
   starts the stream and gets the value, and the other is now the late subscriber — which can leave a table
   rendering *empty* instead of merely failing to reorder.
3. **`.take(1)` on a shared stream is a red flag.** It reads as "just give me the current value", which is
   exactly what an EventStream cannot do. Reach for a `Signal` and `withCurrentValueOf`.
4. **A local `Var` mirror is a legitimate cure only for the consumer that owns it** — it does not help a
   *different* consumer further down the tree that is still subscribing late.

### The `split` companion trap

A row body built by `.split` closes over the item value captured **when that row first rendered**. Anything
that locates by that captured value (`list.indexOf(item)`) breaks the moment the row is edited — the lookup
returns `-1` and the operation silently misbehaves. **Locate by id against the list as it stands now**, and
take the element from that list rather than from the closure.

The same applies to **attributes set from `initial`**. `attr := f(initial)` is frozen at first render, while
`child <-- item$.map(...)` beside it keeps moving, so the row can show one thing and carry another. On
2026-10-24 a filter pill's `dataName := name(initial)` stayed at a bare "Analysed" after its label gained
"— Solo", because the derived value (here, ambiguity across the whole list) changed under a stable key. A nightly
Cypress lookup then timed out on about half of runs. Anything derived from more than the item's own stable
identity, **test handles included**, must be `attr <-- item$.map(f)`. A Cypress flake that "never finds" an
element the screenshot plainly shows is this trap until proven otherwise.

## Consolidate Related Vars Into Case Classes

When multiple `Var`s represent a single state machine (e.g. `Var[Set[X]]` for pending items, `Var[Map[X, Y]]` for results, `Var[Boolean]` for in-progress), consolidate them into a single `Var[StateCase]`. This prevents cross-Var consistency bugs and eliminates multiple `now()` calls.

### Wrong — scattered state across multiple Vars

```scala
// BAD: four Vars for one state machine, must coordinate updates
private val pending: Var[Set[ID]]           = Var(Set.empty)
private val results: Var[Map[ID, Result]]   = Var(Map.empty)
private val total: Var[Int]                 = Var(0)
private val inProgress: Var[Boolean]        = Var(false)
```

### Correct — single Var with case class

```scala
// GOOD: atomic state, one set() updates everything consistently
case class Progress(
  inProgress: Boolean,
  total: Int,
  pending: Set[ID],
  results: Map[ID, Result]
)
object Progress:
  val idle: Progress = Progress(false, 0, Set.empty, Map.empty)

private val progress: Var[Progress] = Var(Progress.idle)

// Derive individual signals for the UI
val pendingSignal: Signal[Set[ID]]   = progress.signal.map(_.pending)
val isActiveSignal: Signal[Boolean]  = progress.signal.map(_.inProgress)
```