# Agent (Chatbot) Modeling Reference

Agents extend state machines with NLP — intents, entities, and platform
adapters. The `BAFGenerator` produces a runnable agent script plus config.
For plain state machines (no NLP), see `state-machines.md`.

## Table of contents

- [Imports](#imports)
- [Creating an agent](#creating-an-agent)
- [Platforms](#platforms)
- [States](#states)
- [Intents and entities](#intents-and-entities)
- [Transitions](#transitions)
- [State behavior / actions](#state-behavior--actions)
- [LLM integration](#llm-integration)
- [RAG (retrieval-augmented generation)](#rag-retrieval-augmented-generation)
- [Database replies](#database-replies)
- [Intent classifier configuration](#intent-classifier-configuration)
- [ReasoningState](#reasoningstate)
- [Tools, skills, and workspaces](#tools-skills-and-workspaces)
- [Session object in body code](#session-object-in-body-code)
- [Validation](#validation)
- [Generation](#generation)
- [Complete example](#complete-example)

## Imports

```python
from besser.BUML.metamodel.state_machine.agent import (
    Agent,
    # Actions
    AgentReply, LLMReply, RAGReply, DBReply,
    # Platforms
    WebSocketPlatform, TelegramPlatform,
    # Intents & entities
    Intent, IntentParameter,
    BaseEntity, BaseEntityImpl, CustomEntity, EntityEntry,
    # Intent classifier configuration
    SimpleIntentClassifierConfiguration,
    LLMIntentClassifierConfiguration, LLMSuite,
    # Agentic primitives
    ReasoningState, Tool, Skill, Workspace,
    # File handling
    File,
)
from besser.BUML.metamodel.state_machine.state_machine import Body, CustomCodeAction
```

## Creating an agent

```python
agent = Agent(name="helpdesk")
```

`Agent` is a `StateMachine` — all state machine concepts (states, events,
conditions, bodies, fallback bodies, config properties) apply. See
`state-machines.md` for those building blocks.

## Platforms

Register at least one platform before generating:

```python
# WebSocket — pairs with the React chat widget from WebAppGenerator
ws = agent.use_websocket_platform()

# Telegram
tg = agent.use_telegram_platform()
```

Both methods are idempotent — calling them twice returns `None` the second
time without duplicating the platform.

**WebSocket platform — rich reply methods** (call from body code via `session`):

```python
ws.reply_options(session, ["Option A", "Option B", "Option C"])
ws.reply_file(session, file_obj)
ws.reply_dataframe(session, df)
ws.reply_plotly(session, fig)
ws.reply_location(session, latitude=48.85, longitude=2.35)
```

**Telegram platform — rich reply methods**:

```python
tg.reply_file(session, file_obj, message="Here is the file")
tg.reply_image(session, file_obj, message="Here is the image")
tg.reply_location(session, latitude=48.85, longitude=2.35)
```

## States

```python
initial  = agent.new_state(name="initial",  initial=True)
greeting = agent.new_state(name="greeting")
fallback = agent.new_state(name="fallback")
```

The first state must be `initial=True` — the metamodel raises `ValueError`
otherwise. State names must not collide with intent names (case-insensitive
— validation catches this).

To override the intent classifier for a specific state, pass `ic_config=`:

```python
expert_state = agent.new_state(
    name="expert",
    ic_config=SimpleIntentClassifierConfiguration(num_epochs=100),
)
```

A state can also be registered as a global entry point for an intent — the
agent jumps to it from any state when that intent matches:

```python
help_state.set_global(help_intent)   # from any state, "help" goes here
```

## Intents and entities

### Intents

```python
hello = agent.new_intent(
    name="hello_intent",
    training_sentences=["Hi", "Hello", "Hey there", "Good morning"],
    description="User greets the agent",   # optional; helps LLM classifiers
)
```

### Base entities (built-in types)

```python
number_entity   = agent.new_entity(name=BaseEntityImpl.number,   base_entity=True)
datetime_entity = agent.new_entity(name=BaseEntityImpl.datetime, base_entity=True)
any_entity      = agent.new_entity(name=BaseEntityImpl.any,      base_entity=True)
```

`BaseEntityImpl` values: `number`, `datetime`, `any`.

### Custom entities (enumerated values + synonyms)

```python
size_entity = agent.new_entity(
    name="size",
    description="Pizza size",
    entries={
        "small":  ["S", "petit"],
        "medium": ["M"],
        "large":  ["L", "big"],
    },
)
```

### Slot-filling — adding parameters to an intent

Use `intent.parameter()` to attach named entity slots to an intent. These
map fragments of the user's utterance to entity values at runtime.

```python
order_pizza = agent.new_intent(
    name="order_pizza",
    training_sentences=["I want a pizza", "order me a pizza"],
)
order_pizza.parameter(name="size_param",    fragment="size",    entity=size_entity)
order_pizza.parameter(name="topping_param", fragment="topping", entity=topping_entity)
```

At runtime, `session.predicted_intent` holds the matched intent; read
extracted slot values via `session.predicted_intent.get_parameter(name)`:

```python
def show_order(session):
    pred = session.predicted_intent
    size_match = pred.get_parameter("size_param")
    if size_match:
        session.reply(f"Size: {size_match.value}")
```

## Transitions

### Intent-based transitions

```python
initial.when_intent_matched(hello).go_to(greeting)
initial.when_no_intent_matched().go_to(fallback)
```

`when_intent_matched` raises `ValueError` if the intent is not registered on
the agent or if the same intent is added twice to the same state.

### Session variable transitions

Transition when a stored session variable satisfies a predicate:

```python
import operator

routing.when_variable_matches_operation(
    var_name="mode",
    operation=operator.eq,
    target="expert",
).go_to(expert_state)

routing.when_variable_matches_operation(
    var_name="score",
    operation=operator.ge,
    target=80,
).go_to(pass_state)
```

### File-received transitions

```python
# Accept any file type
initial.when_file_received().go_to(file_handler)

# Accept only specific types
initial.when_file_received(allowed_types=["pdf", "txt"]).go_to(doc_handler)
initial.when_file_received(allowed_types="image").go_to(image_handler)
```

### Auto (unconditional) transition

```python
greeting.go_to(initial)   # immediately moves to initial after greeting body runs
```

## State behavior / actions

```python
greeting.set_body(Body(name="greet", actions=[
    AgentReply(message="Hello! How can I help you today?"),
]))
```

Actions available in a body:

| Action | Effect |
|--------|--------|
| `AgentReply(message=...)` | Send a static text reply |
| `LLMReply(prompt=..., llm_name=...)` | Generate a reply via a registered LLM |
| `RAGReply(rag_db_name=..., prompt=...)` | Reply using a registered RAG pipeline |
| `DBReply(...)` | Query a database and reply with the result |
| `CustomCodeAction(source="...")` | Run arbitrary Python (advanced) |

Both `prompt=` and `llm_name=` on `LLMReply` are optional — omitting
`llm_name` uses the agent's `default_llm_name`.

## LLM integration

Register one or more LLMs on the agent. The first one registered
automatically becomes the default.

```python
agent.new_llm(
    name="gpt4o",
    provider="openai",
    parameters={"model_name": "gpt-4o"},
    num_previous_messages=5,          # how many prior turns to include
    global_context="You are a helpful support agent.",
)
```

Supported `provider` values: `"openai"`, `"huggingface"`, `"huggingface_api"`,
`"replicate"`.

Change the default after registering multiple LLMs:

```python
agent.new_llm(name="fast_model", provider="openai", parameters={"model_name": "gpt-4o-mini"})
agent.new_llm(name="smart_model", provider="openai", parameters={"model_name": "gpt-4o"})
agent.set_default_llm("smart_model")
```

Use an LLM in a body via `LLMReply`:

```python
support.set_body(Body(name="answer", actions=[
    LLMReply(
        prompt="Answer the user's support question concisely.",
        llm_name="smart_model",   # omit to use default
    )
]))
```

Validation catches references to unregistered LLM names (in `LLMReply`,
`DBReply`, `RAGReply`, `ReasoningState`, and `default_llm_name`).

## RAG (retrieval-augmented generation)

RAG replies retrieve documents from a vector store and pass them to an LLM.

```python
from besser.BUML.metamodel.state_machine.agent import (
    RAGVectorStore, RAGTextSplitter, RAGReply,
)

vector_store = RAGVectorStore(
    embedding_provider="openai",
    embedding_parameters={"model": "text-embedding-3-small"},
    persist_directory="./rag_db",
)

splitter = RAGTextSplitter(
    splitter_type="recursive",
    chunk_size=500,
    chunk_overlap=50,
)

rag = agent.new_rag(
    name="company_docs",
    vector_store=vector_store,
    splitter=splitter,
    llm_name="gpt4o",   # must be a registered LLM name
    k=4,                # number of retrieved chunks
    num_previous_messages=0,
)
```

Use in a body via `RAGReply`:

```python
knowledge_state.set_body(Body(name="rag_answer", actions=[
    RAGReply(
        rag_db_name="company_docs",
        prompt="Answer based only on the retrieved context.",
    )
]))
```

## Database replies

`DBReply` queries a database and sends the result as a reply.

```python
query_state.set_body(Body(name="db_query", actions=[
    DBReply(
        db_selection_type="default",   # "default" | "custom"
        db_custom_name=None,           # required when db_selection_type="custom"
        db_query_mode="llm_query",     # "llm_query" | "sql"
        db_operation="select",         # "any" | "select" | "insert" | "update" | "delete"
        db_sql_query=None,             # raw SQL string; used when db_query_mode="sql"
        llm_name="gpt4o",             # required when db_query_mode="llm_query"
    )
]))
```

## Intent classifier configuration

The intent classifier maps free-text input to intents. Set a default for the
whole agent, or override per state.

### SimpleIntentClassifierConfiguration (local neural network)

```python
ic = SimpleIntentClassifierConfiguration(
    num_words=1000,
    num_epochs=300,
    embedding_dim=128,
    input_max_num_tokens=15,
    discard_oov_sentences=True,
    check_exact_prediction_match=True,
    activation_last_layer='sigmoid',
    activation_hidden_layers='tanh',
    lr=0.001,
)
agent.default_ic_config = ic
```

### LLMIntentClassifierConfiguration (LLM-based)

```python
llm_ic = LLMIntentClassifierConfiguration(
    llm_suite=LLMSuite.openai,
    parameters={"model_name": "gpt-4o"},
    use_intent_descriptions=True,    # use intent description= field
    use_training_sentences=False,
    use_entity_descriptions=True,
    use_entity_synonyms=True,
    llm_name="gpt4o",               # must be a registered LLM name
)
agent.default_ic_config = llm_ic
```

`LLMSuite` values: `openai`, `huggingface`, `huggingface_inference_api`,
`replicate`.

Override for a single state:

```python
specialized = agent.new_state(
    name="specialized",
    ic_config=SimpleIntentClassifierConfiguration(num_epochs=100),
)
```

## ReasoningState

A `ReasoningState` is an autonomous LLM loop that can use tools, skills, and
workspaces to complete multi-step tasks — no hand-written body needed.

```python
reasoning = agent.new_reasoning_state(
    name="reasoning",
    llm="gpt4o",                  # registered LLM name
    initial=False,
    max_steps=8,                  # maximum tool-call iterations
    enable_task_planning=True,    # let the LLM plan sub-tasks
    stream_steps=True,            # stream intermediate steps to the user
    system_prompt="You are a helpful company support agent.",
    fallback_message="I could not complete the task. Please try again.",
)
```

`ReasoningState` does not accept `set_body()` or `set_fallback_body()` —
configure behavior via the constructor parameters instead.

Transition to and from a reasoning state like any other state:

```python
routing.go_to(reasoning)
reasoning.go_to(initial)
```

## Tools, skills, and workspaces

These primitives give a `ReasoningState` callable functions, domain
knowledge, and file-system access.

### Tools — callable Python functions

```python
agent.new_tool(
    name="search_docs",
    description="Search company docs for an answer. Returns a string.",
    code="""def search_docs(query: str) -> str:
    # implementation here
    return f"Results for: {query}"
""",
)
```

`name` must be a valid Python identifier and must match the function name in
`code`. Validation raises an error if `code` is empty or contains no `def`.

### Skills — text knowledge injected into the LLM context

```python
agent.new_skill(
    name="support_guidelines",
    content="Always be polite. Escalate billing issues to a human agent.",
    description="Customer support behavioral guidelines",
)
```

### Workspaces — file-system paths the agent can browse

```python
agent.new_workspace(
    name="knowledge_base",
    path="./docs",
    description="Company knowledge base with product manuals.",
    writable=False,
    max_read_bytes=200_000,
)
```

Validation warns when `description` is empty (the LLM may not know to use
the workspace) and errors when `path` is empty or `max_read_bytes <= 0`.

## Session object in body code

`CustomCodeAction` and `callable=` bodies receive a `session` object
(`AgentSession`). Key attributes and methods:

```python
# Standard (inherited from Session)
session.get("key")              # read a per-user variable
session.set("key", value)       # write a per-user variable
session.reply("message")        # send a text reply to the user

# Agent-specific
session.message                 # the raw text the user sent
session.predicted_intent        # IntentClassifierPrediction or None
session.file                    # File object if a file was received
session.chat_history            # typed list[tuple[str, int]] in the metamodel
```

Reading a matched intent's extracted parameter values:

```python
def handle_order(session):
    pred = session.predicted_intent
    size = pred.get_parameter("size_param")
    if size:
        session.reply(f"Ordered size: {size.value}")
```

## Validation

`agent.validate()` raises `ValueError` by default. Pass
`raise_exception=False` to inspect safely:

```python
result = agent.validate(raise_exception=False)
print("valid:", result["success"])
for w in result["warnings"]:
    print("warning:", w)
for e in result["errors"]:
    print("error:", e)
```

What it checks:

- **State/intent name collisions** (error) — a state and an intent share the
  same name (case-insensitive)
- **Unregistered intent in transition** (error) — `when_intent_matched`
  references an intent not added to the agent
- **Unregistered LLM references** (error) — `LLMReply`, `DBReply`,
  `RAGReply`, `ReasoningState`, or `default_llm_name` names an LLM that was
  never registered via `new_llm()`
- **Empty tool code / missing def** (error) — `Tool.code` is blank or has no
  top-level `def`
- **Invalid tool name** (error) — tool name is not a valid Python identifier
- **Empty skill content** (error) — `Skill.content` is blank
- **Empty workspace path** (error) — `Workspace.path` is blank
- **Non-positive max_read_bytes** (error) — `Workspace.max_read_bytes <= 0`
- **Missing workspace description** (warning) — LLM may not know to use it
- **Missing tool description** (warning) — LLM may not pick it up reliably

## Generation

```python
from besser.generators.agents import BAFGenerator

gen = BAFGenerator(model=agent, output_dir="./agent_output")
gen.generate()
```

Outputs:

- `{agent_name}.py` — the agent script
- `config.yaml` — runtime configuration
- `readme.txt` — quick start instructions
- optional RAG subdirectories if the agent uses retrieval

See the besser-generators skill for `BAFGenerator` options:
`generation_mode` (FULL / PERSONALIZED_ONLY / CODE_ONLY), `config_path`,
`openai_api_key` for LLM-driven personalization.

## Complete example

```python
from besser.BUML.metamodel.state_machine.agent import (
    Agent, AgentReply, LLMReply,
    WebSocketPlatform,
    SimpleIntentClassifierConfiguration,
    BaseEntityImpl,
)
from besser.BUML.metamodel.state_machine.state_machine import Body, CustomCodeAction

agent = Agent(name="helpdesk")
agent.use_websocket_platform()

# Default intent classifier
agent.default_ic_config = SimpleIntentClassifierConfiguration(num_epochs=200)

# Entities
product_entity = agent.new_entity(
    name="product",
    description="Product name",
    entries={"laptop": ["notebook", "computer"], "phone": ["mobile", "cell"]},
)

# Intents
hello = agent.new_intent(
    name="hello_intent",
    training_sentences=["Hi", "Hello", "Hey there", "Good morning"],
)
buy = agent.new_intent(
    name="buy_intent",
    training_sentences=["I want to buy", "purchase", "order"],
)
buy.parameter(name="product_param", fragment="product", entity=product_entity)

# States
initial   = agent.new_state(name="initial",   initial=True)
greeting  = agent.new_state(name="greeting")
buying    = agent.new_state(name="buying")
fallback  = agent.new_state(name="fallback")

# Transitions
initial.when_intent_matched(hello).go_to(greeting)
initial.when_intent_matched(buy).go_to(buying)
initial.when_no_intent_matched().go_to(fallback)

# Bodies
greeting.set_body(Body(name="greet", actions=[
    AgentReply(message="Hello! How can I help you?"),
]))

def show_product(session):
    pred = session.predicted_intent
    p = pred.get_parameter("product_param")
    session.reply(f"Adding {p.value if p else 'item'} to your cart!")

buying.set_body(Body(name="show_product", callable=show_product))

fallback.set_body(Body(name="fallback_body", actions=[
    AgentReply(message="Sorry, I didn't understand. Can you rephrase?"),
]))

# Return to initial after each turn
greeting.go_to(initial)
buying.go_to(initial)
fallback.go_to(initial)

result = agent.validate(raise_exception=False)
assert result["success"], result["errors"]

from besser.generators.agents import BAFGenerator
gen = BAFGenerator(model=agent, output_dir="./agent_output")
gen.generate()
```
