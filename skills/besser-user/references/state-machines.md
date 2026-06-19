# State Machine Modeling Reference

For behavioral modeling — workflows, order processing, anything with
discrete states and transitions. For NLP-driven chatbots specifically, see
the related `agents.md` reference.

## Imports

```python
from besser.BUML.metamodel.state_machine.state_machine import (
    StateMachine, Body, Event, Condition, CustomCodeAction
)
```

## Building a state machine

```python
sm = StateMachine(name="OrderProcess")
initial   = sm.new_state(name="pending",   initial=True)
confirmed = sm.new_state(name="confirmed")
shipped   = sm.new_state(name="shipped")
```

Exactly one state must be marked `initial=True` — and it must be the
**first** state created on the machine (the metamodel enforces this with a
`ValueError`). A state machine can have multiple final states:

```python
sm = StateMachine(name="OrderProcess")
initial   = sm.new_state(name="pending",   initial=True)   # must come first
confirmed = sm.new_state(name="confirmed")
shipped   = sm.new_state(name="shipped")
delivered = sm.new_state(name="delivered", final=True)      # terminal — no outgoing transitions allowed
cancelled = sm.new_state(name="cancelled", final=True)      # another terminal state
```

## Transitions

```python
confirm_event = Event(name="confirm")
initial.when_event(confirm_event).go_to(confirmed)

ship_event = Event(name="ship")
confirmed.when_event(ship_event).go_to(shipped)
```

The fluent API reads as: *when this event fires in this state, go to that
state*. The same `Event` instance can be reused across multiple source
states — create it once and reference it in as many `when_event()` calls
as needed:

```python
cancel_event = Event(name="cancel")
pending.when_event(cancel_event).go_to(cancelled)
confirmed.when_event(cancel_event).go_to(cancelled)   # same event, different source state
```

Transitions can also be gated on a `Condition` (a boolean function).
Use `when_condition()` for condition-only transitions:

```python
# From a source string (use when condition is defined inline or from serialised data)
payment_ok = Condition(
    name="payment_sufficient",
    source="def payment_sufficient(session, params):\n    return session.get('amount') >= 10"
)

# From a callable (the function's source is extracted automatically)
def payment_sufficient(session, params):
    return session.get('amount') >= 10

payment_ok = Condition(name="payment_sufficient", callable=payment_sufficient)
```

```python
item_selected.when_condition(payment_ok).go_to(payment_received)
```

To require **both** an event and a condition, chain `.with_condition()`:

```python
alarm_enabled = Condition(
    name="alarm_enabled",
    source="def alarm_enabled(session, params):\n    return session.get('armed') is True"
)
armed.when_event(motion_detected).with_condition(alarm_enabled).go_to(triggered)
```

Every `Condition` source function must have the signature
`(session, params)`:

- `session` — the `Session` object for the current user; use
  `session.get(key)` / `session.set(key, value)` to read and write
  per-user data
- `params` — a `dict` of extra parameters passed by the runtime at
  transition time (often empty `{}`); read with `params.get(key)`

## State behavior

Three equivalent ways to define a body:

```python
# 1 — inline actions list (most common)
confirmed.set_body(Body(
    name="send_email",
    actions=[CustomCodeAction(source="session.reply('Order confirmed!')")]
))

# 2 — fluent builder with add_custom_code()
confirmed.set_body(
    Body(name="send_email")
    .add_custom_code("session.reply('Order confirmed!')")
)

# 3 — from a callable (source is extracted automatically)
def send_email(session):
    session.reply('Order confirmed!')

confirmed.set_body(Body(name="send_email", callable=send_email))
```

`CustomCodeAction(source=...)` embeds Python that runs when the state is
entered. The `session` object gives access to per-user state at runtime.

## Configuration properties

State machines can carry runtime configuration as `ConfigProperty` objects,
grouped by section — useful for platform settings like ports or API keys:

```python
sm.new_property(section="websocket", name="port", value=8765)
sm.new_property(section="nlp",       name="language", value="en")
```

Properties are accessible at runtime via `sm.properties`.

## Fallback bodies

Each state has an optional `fallback_body` that runs when its main body
raises an error. Set it per state or for all states at once:

```python
# Per state
error_body = Body(name="on_error", actions=[CustomCodeAction(source="session.reply('Something went wrong')")])
confirmed.set_fallback_body(error_body)

# Global — applies to every state in the machine
sm.set_global_fallback_body(error_body)
```

`set_global_fallback_body()` overwrites any previously set `fallback_body`
on all states.

## Validation

`StateMachine.validate()` follows the same pattern as `DomainModel.validate()` —
raises `ValueError` by default; pass `raise_exception=False` to inspect safely:

```python
result = sm.validate(raise_exception=False)
for warning in result["warnings"]:
    print("Warning:", warning)
for error in result["errors"]:
    print("Error:", error)
```

What it checks:

- **Unreachable states** (warning) — a non-initial state with no incoming transitions
- **Final states with outgoing transitions** (error) — violates UML semantics

## Embedding in a domain model

A `StateMachine` can be attached to a `Method` in a class diagram as its
implementation, using `MethodImplementationType.STATE_MACHINE`:

```python
from besser.BUML.metamodel.structural import (
    Class, Method, MethodImplementationType
)
from besser.BUML.metamodel.state_machine.state_machine import StateMachine

sm = StateMachine(name="OrderProcess")
# ... define states and transitions ...

process_order = Method(
    name="process_order",
    implementation_type=MethodImplementationType.STATE_MACHINE,
    state_machine=sm,
)
order = Class(name="Order")
order.add_method(process_order)
```

## Generation

State machines without NLP are typically embedded inside a larger
application — there is no dedicated generator for plain state machines.
Use the BAFGenerator (see `agents.md`) when you want a runnable
chatbot/agent. For embedding in your own code, walk the `sm.states`
collection and each state's `state.transitions` (transitions live on the
individual `State` objects, not on the `StateMachine`).
