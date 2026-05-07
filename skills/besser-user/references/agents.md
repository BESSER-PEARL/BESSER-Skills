# Agent (Chatbot) Modeling Reference

Agents extend state machines with NLP — intents, entities, and platform
adapters. The `BAFGenerator` produces a runnable agent script plus config.

## Imports

```python
from besser.BUML.metamodel.state_machine.agent import (
    Agent, AgentReply, LLMReply,
    WebSocketPlatform,
)
from besser.BUML.metamodel.state_machine.state_machine import Body
```

## Minimal helpdesk agent

```python
agent = Agent(name="helpdesk")
agent.use_websocket_platform()

# States
initial  = agent.new_state(name="initial", initial=True)
greeting = agent.new_state(name="greeting")
fallback = agent.new_state(name="fallback")

# Intents — short example utterances train the NLP classifier
hello = agent.new_intent(
    name="hello_intent",
    training_sentences=["Hi", "Hello", "Hey there", "Good morning"],
)

# Transitions
initial.when_intent_matched(hello).go_to(greeting)
initial.when_no_intent_matched().go_to(fallback)

# Behaviors
greeting.set_body(Body(name="greet", actions=[
    AgentReply(message="Hello! How can I help you today?"),
]))
fallback.set_body(Body(name="fallback", actions=[
    AgentReply(message="Sorry, I didn't understand. Can you rephrase?"),
]))

# Auto-return to the initial state after each turn
greeting.go_to(initial)
fallback.go_to(initial)
```

## Generation

```python
from besser.generators.agents import BAFGenerator

gen = BAFGenerator(model=agent, output_dir="./agent_output")
gen.generate()
```

Outputs:

- `{agent_name}.py` — the agent script
- `config.ini` — runtime configuration
- `readme.txt` — quick start instructions
- optional RAG subdirectories if the agent uses retrieval

See the besser-generators skill for `BAFGenerator` options:
`generation_mode` (FULL / PERSONALIZED_ONLY / CODE_ONLY), `config_path`,
`openai_api_key` for LLM-driven personalization.

## Action types

| Action | Effect |
|--------|--------|
| `AgentReply(message=...)` | Send a static reply to the user |
| `LLMReply(...)` | Generate a reply via an LLM call |
| `CustomCodeAction(source="...")` | Run arbitrary Python (advanced) |

## Platforms

`agent.use_websocket_platform()` is the most common — pairs with the React
chat widget produced by `WebAppGenerator`. Other platforms (Telegram, etc.)
are partially implemented; check the BESSER repo for status before relying
on them.
