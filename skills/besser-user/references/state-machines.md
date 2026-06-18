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

Exactly one state should be marked `initial=True`.

## Transitions

```python
confirm_event = Event(name="confirm")
initial.when_event(confirm_event).go_to(confirmed)

ship_event = Event(name="ship")
confirmed.when_event(ship_event).go_to(shipped)
```

The fluent API reads as: *when this event fires in this state, go to that
state*. You can also gate transitions on a `Condition`.

## State behavior

```python
confirmed.set_body(Body(
    name="send_email",
    actions=[CustomCodeAction(source="session.reply('Order confirmed!')")]
))
```

`CustomCodeAction(source=...)` lets you embed Python that runs when the
state is entered. The string is templated into the generated runner.

## Generation

State machines without NLP are typically embedded inside a larger
application — there is no dedicated generator for plain state machines.
Use the BAFGenerator (see `agents.md`) when you want a runnable
chatbot/agent. For embedding in your own code, walk the `sm.states`
collection and each state's `state.transitions` (transitions live on the
individual `State` objects, not on the `StateMachine`).
