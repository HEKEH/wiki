---
created: 2026-08-11T22:20:00 (UTC +08:00)
tags: []
source: https://openai.github.io/openai-agents-python/
author: OpenAI
---

# OpenAI Agents SDK — Core Documentation (agents, tools, handoffs, guardrails, context)

## Agents

# Agents

Agents are the core building block in your apps. An agent is a large language model (LLM) configured with instructions, tools, and optional runtime behavior such as handoffs, guardrails, and structured outputs.

Use this page when you want to define or customize a single base `Agent` rather than a `SandboxAgent`. If you are deciding how multiple agents should collaborate, read [Agent orchestration](multi_agent.md). If the agent should run inside an isolated workspace with manifest-defined files and sandbox-native capabilities, read [Sandbox agent concepts](sandbox/guide.md).

The SDK uses the Responses API by default for OpenAI models, but the distinction here is orchestration: `Agent` plus `Runner` lets the SDK manage turns, tools, guardrails, handoffs, and sessions for you. If you want to own that loop yourself, use the Responses API directly instead.

## Choose the next guide

Use this page as the hub for agent definition. Jump to the adjacent guide that matches the next decision you need to make.

| If you want to... | Read next |
| --- | --- |
| Choose a model or provider setup | [Models](models/index.md) |
| Add capabilities to the agent | [Tools](tools.md) |
| Run an agent against a real repo, document bundle, or isolated workspace | [Sandbox agents quickstart](sandbox_agents.md) |
| Decide between manager-style orchestration and handoffs | [Agent orchestration](multi_agent.md) |
| Configure handoff behavior | [Handoffs](handoffs.md) |
| Run turns, stream events, or manage conversation state | [Running agents](running_agents.md) |
| Inspect final output, run items, or resumable state | [Results](results.md) |
| Share local dependencies and runtime state | [Context management](context.md) |

## Basic configuration

The most common properties of an agent are:

| Property | Required | Description |
| --- | --- | --- |
| `name` | yes | Human-readable agent name. |
| `instructions` | no | System prompt or dynamic instructions callback. Strongly recommended. See [Dynamic instructions](#dynamic-instructions). |
| `prompt` | no | OpenAI Responses API prompt configuration. Accepts a static prompt object or a function. See [Prompt templates](#prompt-templates). |
| `handoff_description` | no | Short description exposed when this agent is offered as a handoff target. |
| `handoffs` | no | Delegate the conversation to specialist agents. See [handoffs](handoffs.md). |
| `model` | no | Which LLM to use. See [Models](models/index.md). |
| `model_settings` | no | Model tuning parameters such as `temperature`, `top_p`, and `tool_choice`. |
| `tools` | no | Tools the agent can call. See [Tools](tools.md). |
| `mcp_servers` | no | MCP servers that provide MCP-backed tools to the agent. See the [MCP guide](mcp.md). |
| `mcp_config` | no | Fine-tune how MCP tools are prepared, such as converting their schemas to strict mode and formatting MCP failures. See the [MCP guide](mcp.md#agent-level-mcp-configuration). |
| `input_guardrails` | no | Guardrails that run on the first user input for this agent chain. See [Guardrails](guardrails.md). |
| `output_guardrails` | no | Guardrails that run on the final output for this agent. See [Guardrails](guardrails.md). |
| `output_type` | no | Structured output type instead of plain text. See [Output types](#output-types). |
| `hooks` | no | Agent-scoped lifecycle callbacks. See [Lifecycle events (hooks)](#lifecycle-events-hooks). |
| `tool_use_behavior` | no | Control whether tool results loop back to the model or end the run. See [Tool use behavior](#tool-use-behavior). |
| `reset_tool_choice` | no | Reset `tool_choice` after a tool call (default: `True`) to avoid tool-use loops. See [Forcing tool use](#forcing-tool-use). |

```python
from agents import Agent
from agents.decorators import tool

@tool
def get_weather(city: str) -> str:
    """returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Haiku agent",
    instructions="Always respond in haiku form",
    model="gpt-5-nano",
    tools=[get_weather],
)
```

Everything in this section applies to `Agent`. `SandboxAgent` builds on the same ideas, then adds `default_manifest`, `base_instructions`, `capabilities`, and `run_as` for workspace-scoped runs. See [Sandbox agent concepts](sandbox/guide.md).

## Prompt templates

You can reference a prompt template created in the OpenAI platform by setting `prompt`. This works when OpenAI models are accessed through the Responses API.

To use it, please:

1. Go to https://platform.openai.com/playground/prompts
2. Create a new prompt variable, `poem_style`.
3. Create a system prompt with the content:

    ```
    Write a poem in {{poem_style}}
    ```

4. Run the example with the `--prompt-id` flag.

```python
from agents import Agent

agent = Agent(
    name="Prompted assistant",
    prompt={
        "id": "pmpt_123",
        "version": "1",
        "variables": {"poem_style": "haiku"},
    },
)
```

You can also generate the prompt dynamically at run time:

```python
from dataclasses import dataclass

from agents import Agent, GenerateDynamicPromptData, Runner

@dataclass
class PromptContext:
    prompt_id: str
    poem_style: str


async def build_prompt(data: GenerateDynamicPromptData):
    ctx: PromptContext = data.context.context
    return {
        "id": ctx.prompt_id,
        "version": "1",
        "variables": {"poem_style": ctx.poem_style},
    }


agent = Agent(name="Prompted assistant", prompt=build_prompt)
result = await Runner.run(
    agent,
    "Say hello",
    context=PromptContext(prompt_id="pmpt_123", poem_style="limerick"),
)
```

## Context

Agents are generic on their `context` type. Context is a dependency-injection tool: it's an object you create and pass to `Runner.run()`, that is passed to every agent, tool, handoff etc, and it serves as a grab bag of dependencies and state for the agent run. You can provide any Python object as the context.

Read the [context guide](context.md) for the full `RunContextWrapper` surface, shared usage tracking, nested `tool_input`, and serialization caveats.

```python
from dataclasses import dataclass

@dataclass
class Purchase:
    id: str

@dataclass
class UserContext:
    name: str
    uid: str
    is_pro_user: bool

    async def fetch_purchases(self) -> list[Purchase]:
        # implement your logic here
        return []

agent = Agent[UserContext](
    ...,
)
```

## Output types

By default, agents produce plain text (i.e. `str`) outputs. If you want the agent to produce a particular type of output, you can use the `output_type` parameter. A common choice is to use [Pydantic](https://docs.pydantic.dev/) objects, but we support any type that can be wrapped in a Pydantic [TypeAdapter](https://docs.pydantic.dev/latest/api/type_adapter/) - dataclasses, lists, TypedDict, etc.

```python
from pydantic import BaseModel
from agents import Agent


class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

agent = Agent(
    name="Calendar extractor",
    instructions="Extract calendar events from text",
    output_type=CalendarEvent,
)
```

!!! note

    When you pass an `output_type`, that tells the model to use [structured outputs](https://platform.openai.com/docs/guides/structured-outputs) instead of regular plain text responses.

## Multi-agent system design patterns

There are many ways to design multi‑agent systems, but we commonly see two broadly applicable patterns:

1. Manager (agents as tools): A central manager/orchestrator invokes specialized sub‑agents as tools and retains control of the conversation.
2. Handoffs: Peer agents hand off control to a specialized agent that takes over the conversation. This is decentralized.

See [our practical guide to building agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) for more details.

### Manager (agents as tools)

The `customer_facing_agent` handles all user interaction and invokes specialized sub‑agents exposed as tools. Read more in the [tools](tools.md#agents-as-tools) documentation.

```python
from agents import Agent

booking_agent = Agent(...)
refund_agent = Agent(...)

customer_facing_agent = Agent(
    name="Customer-facing agent",
    instructions=(
        "Handle all direct user communication. "
        "Call the relevant tools when specialized expertise is needed."
    ),
    tools=[
        booking_agent.as_tool(
            tool_name="booking_expert",
            tool_description="Handles booking questions and requests.",
        ),
        refund_agent.as_tool(
            tool_name="refund_expert",
            tool_description="Handles refund questions and requests.",
        )
    ],
)
```

### Handoffs

Configured handoff targets are sub‑agents to which the agent can delegate. When a handoff occurs, the delegated agent receives the conversation history and takes over the conversation. This pattern enables modular, specialized agents that excel at a single task. Read more in the [handoffs](handoffs.md) documentation.

```python
from agents import Agent

booking_agent = Agent(...)
refund_agent = Agent(...)

triage_agent = Agent(
    name="Triage agent",
    instructions=(
        "Help the user with their questions. "
        "If they ask about booking, hand off to the booking agent. "
        "If they ask about refunds, hand off to the refund agent."
    ),
    handoffs=[booking_agent, refund_agent],
)
```

## Dynamic instructions

In most cases, you can provide instructions when you create the agent. However, you can also provide dynamic instructions via a function. The function will receive the agent and context, and must return the prompt. Both regular and `async` functions are accepted.

```python
from agents import Agent, RunContextWrapper

def dynamic_instructions(
    context: RunContextWrapper[UserContext], agent: Agent[UserContext]
) -> str:
    return f"The user's name is {context.context.name}. Help them with their questions."


agent = Agent[UserContext](
    name="Triage agent",
    instructions=dynamic_instructions,
)
```

## Lifecycle events (hooks)

Sometimes, you want to observe the lifecycle of an agent. For example, you may want to log events, pre-fetch data, or record usage when certain events occur.

There are two hook scopes:

-   [`RunHooks`][agents.lifecycle.RunHooks] observe the entire `Runner.run(...)` invocation, including handoffs to other agents.
-   [`AgentHooks`][agents.lifecycle.AgentHooks] are attached to a specific agent instance via `agent.hooks`.

The callback context also changes depending on the event:

-   Agent start/end hooks receive [`AgentHookContext`][agents.run_context.AgentHookContext], which wraps your original context and carries the shared run usage state.
-   LLM, tool, and handoff hooks receive [`RunContextWrapper`][agents.run_context.RunContextWrapper].

Typical hook timing:

-   `on_agent_start`: when a specific agent begins running; `on_agent_end`: when that agent finishes producing a final output.
-   `on_llm_start` / `on_llm_end`: immediately around each model call.
- `on_tool_start` / `on_tool_end`: around each local tool invocation. For function tools, the hook `context` is typically a `ToolContext`, so you can inspect tool-call metadata such as `tool_call_id`.
-   `on_handoff`: when control moves from one agent to another.

Use `RunHooks` when you want a single observer for the whole workflow, and `AgentHooks` when you want lifecycle callbacks scoped to a specific agent.

```python
from agents import Agent, RunHooks, Runner


class LoggingHooks(RunHooks):
    async def on_agent_start(self, context, agent):
        print(f"Starting {agent.name}")

    async def on_llm_end(self, context, agent, response):
        print(f"{agent.name} produced {len(response.output)} output items")

    async def on_agent_end(self, context, agent, output):
        print(f"{agent.name} finished with usage: {context.usage}")


agent = Agent(name="Assistant", instructions="Be concise.")
result = await Runner.run(agent, "Explain quines", hooks=LoggingHooks())
print(result.final_output)
```

For the full callback surface, see the [Lifecycle API reference](ref/lifecycle.md).

## Guardrails

Guardrails allow you to run checks/validations on user input in parallel to the agent running, and on the agent's output once it is produced. For example, you could screen the user's input and agent's output for relevance. Read more in the [guardrails](guardrails.md) documentation.

## Cloning/copying agents

By using the `clone()` method on an agent, you can duplicate an Agent, and optionally change any properties you like.

```python
pirate_agent = Agent(
    name="Pirate",
    instructions="Write like a pirate",
    model="gpt-5.6-sol",
)

robot_agent = pirate_agent.clone(
    name="Robot",
    instructions="Write like a robot",
)
```

## Forcing tool use

Supplying a list of tools doesn't always mean the LLM will use a tool. You can force tool use by setting [`ModelSettings.tool_choice`][agents.model_settings.ModelSettings.tool_choice]. Valid values are:

1. `auto`, which allows the LLM to decide whether or not to use a tool.
2. `required`, which requires the LLM to use a tool (but it can intelligently decide which tool).
3. `none`, which requires the LLM to _not_ use a tool.
4. Setting a specific string e.g. `my_tool`, which requires the LLM to use that specific tool.

When you are using OpenAI Responses tool search, named tool choices are more limited: you cannot target bare namespace names or deferred-only tools with `tool_choice`, and `tool_choice="tool_search"` does not target [`ToolSearchTool`][agents.tool.ToolSearchTool]. In those cases, prefer `auto` or `required`. See [Hosted tool search](tools.md#hosted-tool-search) for the Responses-specific constraints.

```python
from agents import Agent, ModelSettings
from agents.decorators import tool

@tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    model_settings=ModelSettings(tool_choice="get_weather")
)
```

## Tool use behavior

The `tool_use_behavior` parameter in the `Agent` configuration controls how tool outputs are handled:

- `"run_llm_again"`: The default. Tools are run, and the LLM processes the results to produce a final response.
- `"stop_on_first_tool"`: The output of the first tool call is used as the final response, without further LLM processing.

```python
from agents import Agent
from agents.decorators import tool

@tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    tool_use_behavior="stop_on_first_tool"
)
```

- `StopAtTools(stop_at_tool_names=[...])`: Stops if any specified tool is called, using its output as the final response.

```python
from agents import Agent
from agents.agent import StopAtTools
from agents.decorators import tool

@tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

@tool
def sum_numbers(a: int, b: int) -> int:
    """Adds two numbers."""
    return a + b

agent = Agent(
    name="Stop At Stock Agent",
    instructions="Get weather or sum numbers.",
    tools=[get_weather, sum_numbers],
    tool_use_behavior=StopAtTools(stop_at_tool_names=["get_weather"])
)
```

- `ToolsToFinalOutputFunction`: A custom function that processes tool results and decides whether to end the run with a final output or continue processing with the LLM.

```python
from agents import Agent, FunctionToolResult, RunContextWrapper
from agents.agent import ToolsToFinalOutputResult
from agents.decorators import tool
from typing import List, Any

@tool
def get_weather(city: str) -> str:
    """Returns weather info for the specified city."""
    return f"The weather in {city} is sunny"

def custom_tool_handler(
    context: RunContextWrapper[Any],
    tool_results: List[FunctionToolResult]
) -> ToolsToFinalOutputResult:
    """Processes tool results to decide final output."""
    for result in tool_results:
        if result.output and "sunny" in result.output:
            return ToolsToFinalOutputResult(
                is_final_output=True,
                final_output=f"Final weather: {result.output}"
            )
    return ToolsToFinalOutputResult(
        is_final_output=False,
        final_output=None
    )

agent = Agent(
    name="Weather Agent",
    instructions="Retrieve weather details.",
    tools=[get_weather],
    tool_use_behavior=custom_tool_handler
)
```

!!! note

    To prevent infinite loops, the framework automatically resets `tool_choice` to "auto" after a tool call. This behavior is configurable via [`agent.reset_tool_choice`][agents.agent.Agent.reset_tool_choice]. The infinite loop is because tool results are sent to the LLM, which then generates another tool call because of `tool_choice`, ad infinitum.

---

## Running agents

# Running agents

You can run agents via the [`Runner`][agents.run.Runner] class. You have 3 options:

1. [`Runner.run()`][agents.run.Runner.run], which runs async and returns a [`RunResult`][agents.result.RunResult].
2. [`Runner.run_sync()`][agents.run.Runner.run_sync], which is a sync method and just runs `.run()` under the hood.
3. [`Runner.run_streamed()`][agents.run.Runner.run_streamed], which runs async and returns a [`RunResultStreaming`][agents.result.RunResultStreaming]. It calls the LLM in streaming mode, and streams those events to you as they are received.

```python
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="You are a helpful assistant")

    result = await Runner.run(agent, "Write a haiku about recursion in programming.")
    print(result.final_output)
    # Code within the code,
    # Functions calling themselves,
    # Infinite loop's dance
```

Read more in the [results guide](results.md).

## Runner lifecycle and configuration

### The agent loop

When you call any of the three `Runner` methods above, you pass in a starting agent and input. The input can be:

-   a string (treated as a user message),
-   a list of input items in the OpenAI Responses API format, or
-   a [`RunState`][agents.run_state.RunState] when resuming a paused run or a run stopped with `cancel(mode="after_turn")`. The state can also carry [input staged for the next resumed model call](results.md#add-input-before-resuming).

The runner then runs a loop:

1. We call the LLM for the current agent, with the current input.
2. The LLM produces its output.
    1. If the runner classifies the LLM's output as final output, the loop ends and we return the result.
    2. If the LLM requests a handoff, we update the current agent and input, and re-run the loop.
    3. If the LLM produces tool calls, we run those tool calls, append the results, and re-run the loop.
3. If we exceed the `max_turns` passed, we raise a [`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded] exception. Pass `max_turns=None` to disable this turn limit.

!!! note

    The rule for whether the LLM output is considered as a "final output" is that it produces text output with the desired type, and there are no tool calls.

### Streaming

Streaming allows you to additionally receive streaming events as the LLM runs. Once the stream is done, the [`RunResultStreaming`][agents.result.RunResultStreaming] will contain the complete information about the run, including all the new outputs produced. You can call `.stream_events()` for the streaming events. Read more in the [streaming guide](streaming.md).

#### Responses WebSocket transport (optional helper)

If you enable the OpenAI Responses websocket transport, you can keep using the normal `Runner` APIs. The websocket session helper is recommended for connection reuse, but it is not required.

This is the Responses API over websocket transport, not the [Realtime API](realtime/guide.md).

For transport-selection rules and caveats around concrete model objects or custom providers, see [Models](models/index.md#responses-websocket-transport).

##### Pattern 1: No session helper (works)

Use this when you just want websocket transport and do not need the SDK to manage a shared provider/session for you.

```python
import asyncio

from agents import Agent, Runner, set_default_openai_responses_transport


async def main():
    set_default_openai_responses_transport("websocket")

    agent = Agent(name="Assistant", instructions="Be concise.")
    result = Runner.run_streamed(agent, "Summarize recursion in one sentence.")

    async for event in result.stream_events():
        if event.type == "raw_response_event":
            continue
        print(event.type)


asyncio.run(main())
```

This pattern is fine for single runs. If you call `Runner.run()` / `Runner.run_streamed()` repeatedly, each run may reconnect unless you manually reuse the same `RunConfig` / provider instance.

##### Pattern 2: Use `responses_websocket_session()` (recommended for multi-turn reuse)

Use [`responses_websocket_session()`][agents.responses_websocket_session] when you want a shared websocket-capable provider and `RunConfig` across multiple runs (including nested agent-as-tool calls that inherit the same `run_config`).

```python
import asyncio

from agents import Agent, responses_websocket_session


async def main():
    agent = Agent(name="Assistant", instructions="Be concise.")

    async with responses_websocket_session(
        responses_websocket_options={"ping_interval": 20.0, "ping_timeout": 60.0},
    ) as ws:
        first = ws.run_streamed(agent, "Say hello in one short sentence.")
        async for _event in first.stream_events():
            pass

        second = ws.run_streamed(
            agent,
            "Now say goodbye.",
            previous_response_id=first.last_response_id,
        )
        async for _event in second.stream_events():
            pass


asyncio.run(main())
```

Finish consuming streamed results before the context exits. Exiting the context while a websocket request is still in flight may force-close the shared connection.

The service processes one response at a time on each websocket connection and limits a connection to 60 minutes. The helper reuses the connection but does not remove those constraints. After a reconnect, `store=False` and ZDR flows cannot recover an uncached `previous_response_id`; start a new chain with full input context or rebuild it from locally managed session state. See the [Responses WebSocket transport notes](models/index.md#responses-websocket-transport) for the full recovery behavior.

If long reasoning turns hit websocket keepalive timeouts, increase `ping_timeout` or set `ping_timeout=None` to disable heartbeat timeouts. Use HTTP/SSE transport for runs where reliability matters more than websocket latency.

### Run config

The `run_config` parameter lets you configure some global settings for the agent run:

#### Common run config categories

Use `RunConfig` to override behavior for a single run without changing each agent definition.

##### Model, provider, and session defaults

-   [`model`][agents.run.RunConfig.model]: Allows setting a global LLM model to use, irrespective of what `model` each Agent has.
-   [`model_provider`][agents.run.RunConfig.model_provider]: A model provider for looking up model names, which defaults to OpenAI.
-   [`model_settings`][agents.run.RunConfig.model_settings]: Overrides agent-specific settings. For example, you can set a global `temperature` or `top_p`.
-   [`session_settings`][agents.run.RunConfig.session_settings]: Overrides session-level defaults (for example, `SessionSettings(limit=...)`) when retrieving history during a run.
-   [`session_input_callback`][agents.run.RunConfig.session_input_callback]: Customize how new user input is merged with session history before each `Runner` run when using Sessions. The callback can be sync or async.

##### Guardrails, handoffs, and model input shaping

-   [`input_guardrails`][agents.run.RunConfig.input_guardrails], [`output_guardrails`][agents.run.RunConfig.output_guardrails]: A list of input or output guardrails to include on all runs.
-   [`handoff_input_filter`][agents.run.RunConfig.handoff_input_filter]: A global input filter to apply to all handoffs, if the handoff doesn't already have one. The input filter allows you to edit the inputs that are sent to the new agent. See the documentation in [`Handoff.input_filter`][agents.handoffs.Handoff.input_filter] for more details.
-   [`nest_handoff_history`][agents.run.RunConfig.nest_handoff_history]: Opt-in beta that compacts summarizable history into ordered assistant summary segments while preserving lossless message items in their original positions before invoking the next agent. This is disabled by default while we stabilize nested handoffs; set to `True` to enable or leave `False` to pass through the raw transcript. Sessions, `RunState`, and `RunResult.to_input_list()` avoid appending an exact message occurrence twice when the SDK-default nested history already owns it, while preserving separate identical messages. All [Runner methods][agents.run.Runner] automatically create a `RunConfig` when you do not pass one, so the quickstarts and examples keep the default off, and any explicit [`Handoff.input_filter`][agents.handoffs.Handoff.input_filter] callbacks continue to override it. Individual handoffs can override this setting via [`Handoff.nest_handoff_history`][agents.handoffs.Handoff.nest_handoff_history].
-   [`handoff_history_mapper`][agents.run.RunConfig.handoff_history_mapper]: Optional callable that receives the normalized transcript (history + handoff items) whenever you opt in to `nest_handoff_history`. It must return the exact list of input items to forward to the next agent, replacing the built-in ordered summary segments without writing a full handoff filter.
-   [`call_model_input_filter`][agents.run.RunConfig.call_model_input_filter]: Hook to edit the fully prepared model input (instructions and input items) immediately before the model call, e.g., to trim history or inject a system prompt.
-   [`reasoning_item_id_policy`][agents.run.RunConfig.reasoning_item_id_policy]: Control whether reasoning item IDs are preserved or omitted when the runner converts prior outputs into next-turn model input.

##### Tracing and observability

-   [`tracing_disabled`][agents.run.RunConfig.tracing_disabled]: Allows you to disable [tracing](tracing.md) for the entire run.
-   [`tracing`][agents.run.RunConfig.tracing]: Pass a [`TracingConfig`][agents.tracing.TracingConfig] to override trace export settings such as the per-run tracing API key.
-   [`trace_include_sensitive_data`][agents.run.RunConfig.trace_include_sensitive_data]: Configures whether traces will include potentially sensitive data, such as LLM and tool call inputs/outputs.
-   [`workflow_name`][agents.run.RunConfig.workflow_name], [`trace_id`][agents.run.RunConfig.trace_id], [`group_id`][agents.run.RunConfig.group_id]: Sets the tracing workflow name, trace ID and trace group ID for the run. We recommend at least setting `workflow_name`. The group ID is an optional field that lets you link traces across multiple runs.
-   [`trace_metadata`][agents.run.RunConfig.trace_metadata]: Metadata to include on all traces.

##### Tool execution, approval, and tool error behavior

-   [`tool_execution`][agents.run.RunConfig.tool_execution]: Configure SDK-side execution behavior for local tool calls, such as limiting how many local function tool calls run at once.
-   [`tool_not_found_behavior`][agents.run.RunConfig.tool_not_found_behavior]: Configure how the runner handles model-emitted function tool calls whose tool name does not match any function tool available to the current agent. The default raises `ModelBehaviorError`; opt in to return a model-visible error output instead.
-   [`tool_name_collision_policy`][agents.run.RunConfig.tool_name_collision_policy]: Configure how the runner handles unnamespaced function-tool and handoff names that collide. The default, `"warn"`, logs an actionable warning and exposes only the current dispatch winner; `"error"` raises `UserError` before the model is called. Strict validation for namespaced and deferred-loading tools is unchanged.
-   [`tool_error_formatter`][agents.run.RunConfig.tool_error_formatter]: Customize model-visible tool error messages, such as approval rejections and opt-in tool-not-found outputs.

Nested handoffs are available as an opt-in beta. Enable ordered transcript compaction by passing `RunConfig(nest_handoff_history=True)` or set `handoff(..., nest_handoff_history=True)` to turn it on for a specific handoff. The built-in mapper places generated assistant summary segments around lossless message items instead of collapsing the whole transcript into one message. If you prefer to keep the raw transcript (the default), leave the flag unset or provide a `handoff_input_filter` (or `handoff_history_mapper`) that forwards the conversation exactly as you need. To change the wrapper text used in generated summary segments without writing a custom mapper, call [`set_conversation_history_wrappers`][agents.handoffs.set_conversation_history_wrappers] (and [`reset_conversation_history_wrappers`][agents.handoffs.reset_conversation_history_wrappers] to restore the defaults).

#### Run config details

##### `tool_execution`

Use `tool_execution` when you want to configure SDK-side behavior for local function tools, such as limiting local function-tool concurrency for a run.

```python
from agents import Agent, RunConfig, Runner, ToolExecutionConfig

agent = Agent(name="Assistant", tools=[...])

result = await Runner.run(
    agent,
    "Run the required tool calls.",
    run_config=RunConfig(
        tool_execution=ToolExecutionConfig(
            max_function_tool_concurrency=2,
            pre_approval_tool_input_guardrails=True,
        ),
    ),
)
```

`max_function_tool_concurrency=None` preserves the default behavior: when a model emits multiple function tool calls in a turn, the SDK starts all emitted local function tool calls. Set an integer value to cap how many of those local function tool calls run at once.

This is separate from provider-side [`ModelSettings.parallel_tool_calls`][agents.model_settings.ModelSettings.parallel_tool_calls]. `parallel_tool_calls` controls whether the model is allowed to emit multiple tool calls in a single response. `tool_execution.max_function_tool_concurrency` controls how the SDK executes local function tool calls after the model has emitted them.

`pre_approval_tool_input_guardrails=False` preserves the default approval flow: if a function tool needs approval, the run pauses first and the tool input guardrails run only after approval, immediately before execution. Set it to `True` when you want function-tool input guardrails to run before the pending approval interruption is emitted. Calls that pass this pre-approval check still run the same input guardrails again after approval, so time-sensitive checks are revalidated before execution.

##### `tool_not_found_behavior`

By default, if the model emits a function tool call that does not match any function tool available to the current agent, the runner raises `ModelBehaviorError`.

Set `tool_not_found_behavior="return_error_to_model"` when you want the run to remain recoverable. In that mode, the SDK appends a `function_call_output` for the unresolved tool call and runs the model again, so the model can choose an available tool or answer without using that tool.

```python
from agents import Agent, RunConfig, Runner

agent = Agent(name="Assistant", tools=[...])

result = await Runner.run(
    agent,
    "Handle this request with the available tools.",
    run_config=RunConfig(tool_not_found_behavior="return_error_to_model"),
)
```

This option currently applies only to function tool calls that fail tool-name lookup. Other invalid tool payloads continue to use their existing error behavior.

##### `tool_error_formatter`

Use `tool_error_formatter` to customize the message that is returned to the model when the SDK creates a model-visible tool error output.

The formatter receives [`ToolErrorFormatterArgs`][agents.run_config.ToolErrorFormatterArgs] with:

-   `kind`: The error category, such as `"approval_rejected"` or `"tool_not_found"`.
-   `tool_type`: The tool runtime (`"function"`, `"computer"`, `"shell"`, `"apply_patch"`, or `"custom"`).
-   `tool_name`: The tool name.
-   `call_id`: The tool call ID.
-   `default_message`: The SDK's default model-visible message.
-   `run_context`: The active run context wrapper.

Return a string to replace the message, or `None` to use the SDK default.

```python
from agents import Agent, RunConfig, Runner, ToolErrorFormatterArgs


def format_rejection(args: ToolErrorFormatterArgs[None]) -> str | None:
    if args.kind == "approval_rejected":
        return (
            f"Tool call '{args.tool_name}' was rejected by a human reviewer. "
            "Ask for confirmation or propose a safer alternative."
        )
    if args.kind == "tool_not_found":
        return f"Tool '{args.tool_name}' is not available. Choose one of the listed tools."
    return None


agent = Agent(name="Assistant")
result = Runner.run_sync(
    agent,
    "Please delete the production database.",
    run_config=RunConfig(tool_error_formatter=format_rejection),
)
```

##### `reasoning_item_id_policy`

`reasoning_item_id_policy` controls how reasoning items are converted into next-turn model input when the runner carries history forward (for example, when using `RunResult.to_input_list()` or session-backed runs).

-   `None` or `"preserve"` (default): Keep reasoning item IDs.
-   `"omit"`: Strip reasoning item IDs from the generated next-turn input.

Use `"omit"` primarily as an opt-in mitigation for a class of Responses API 400 errors where a reasoning item is sent with an `id` but without the required following item (for example, `Item 'rs_...' of type 'reasoning' was provided without its required following item.`).

This can happen in multi-turn agent runs when the SDK constructs follow-up input from prior outputs (including session persistence, server-managed conversation deltas, streamed/non-streamed follow-up turns, and resume paths) and a reasoning item ID is preserved but the provider requires that ID to remain paired with its corresponding following item.

Setting `reasoning_item_id_policy="omit"` keeps the reasoning content but strips the reasoning item `id`, which avoids triggering that API invariant in SDK-generated follow-up inputs.

Scope notes:

-   This only changes reasoning items generated/forwarded by the SDK when it builds follow-up input.
-   It does not rewrite user-supplied initial input items.
-   `call_model_input_filter` can still intentionally reintroduce reasoning IDs after this policy is applied.

## State and conversation management

### Choose a memory strategy

There are four common ways to carry state into the next turn:

| Strategy | Where state lives | Best for | What you pass on the next turn |
| --- | --- | --- | --- |
| `result.to_input_list()` | Your app memory | Small chat loops, full manual control, any provider | The list from `result.to_input_list()` plus the next user message |
| `session` | Your storage plus the SDK | Persistent chat state, resumable runs, custom stores | The same `session` instance or another instance pointed at the same store |
| `conversation_id` | OpenAI Conversations API | A named server-side conversation you want to share across workers or services | The same `conversation_id` plus only the new user turn |
| `previous_response_id` | OpenAI Responses API | Lightweight server-managed continuation without creating a conversation resource | `result.last_response_id` plus only the new user turn |

`result.to_input_list()` and `session` are client-managed. `conversation_id` and `previous_response_id` are OpenAI-managed and only apply when you are using the OpenAI Responses API. In most applications, pick one persistence strategy per conversation. Mixing client-managed history with OpenAI-managed state can duplicate context unless you are deliberately reconciling both layers.

!!! note

    Session persistence cannot be combined with server-managed conversation settings
    (`conversation_id`, `previous_response_id`, or `auto_previous_response_id`) in the
    same run. Choose one approach per call.

### Conversations/chat threads

Calling any of the run methods can result in one or more agents running (and hence one or more LLM calls), but it represents a single logical turn in a chat conversation. For example:

1. User turn: user enters text
2. Runner run: first agent calls LLM, runs tools, does a handoff to a second agent, second agent runs more tools, and then produces an output.

At the end of the agent run, you can choose what to show to the user. For example, you might show the user every new item generated by the agents, or just the final output. Either way, the user might then ask a followup question, in which case you can call the run method again.

#### Manual conversation management

You can manually manage conversation history using the [`RunResultBase.to_input_list()`][agents.result.RunResultBase.to_input_list] method to get the inputs for the next turn:

```python
from agents import Agent, Runner, trace

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    thread_id = "thread_123"  # Example thread ID
    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?")
        print(result.final_output)
        # San Francisco

        # Second turn
        new_input = result.to_input_list() + [{"role": "user", "content": "What state is it in?"}]
        result = await Runner.run(agent, new_input)
        print(result.final_output)
        # California
```

#### Automatic conversation management with sessions

For a simpler approach, you can use [Sessions](sessions/index.md) to automatically handle conversation history without manually calling `.to_input_list()`:

```python
from agents import Agent, Runner, SQLiteSession, trace

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    # Create session instance
    session = SQLiteSession("conversation_123")

    thread_id = "thread_123"  # Example thread ID
    with trace(workflow_name="Conversation", group_id=thread_id):
        # First turn
        result = await Runner.run(agent, "What city is the Golden Gate Bridge in?", session=session)
        print(result.final_output)
        # San Francisco

        # Second turn - agent automatically remembers previous context
        result = await Runner.run(agent, "What state is it in?", session=session)
        print(result.final_output)
        # California
```

Sessions automatically:

-   Retrieves conversation history before each run
-   Stores new messages after each run
-   Maintains separate conversations for different session IDs

See the [Sessions documentation](sessions/index.md) for more details.


#### Server-managed conversations

You can also let the OpenAI conversation state feature manage conversation state on the server side, instead of handling it locally with `to_input_list()` or `Sessions`. This allows you to preserve conversation history without manually resending all past messages. With either server-managed approach below, pass only the new turn's input on each request and reuse the saved ID. See the [OpenAI Conversation state guide](https://platform.openai.com/docs/guides/conversation-state?api-mode=responses) for more details.

OpenAI provides two ways to track state across turns:

##### 1. Using `conversation_id`

You first create a conversation using the OpenAI Conversations API and then reuse its ID for every subsequent call:

```python
from agents import Agent, Runner
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    # Create a server-managed conversation
    conversation = await client.conversations.create()
    conv_id = conversation.id

    while True:
        user_input = input("You: ")
        result = await Runner.run(agent, user_input, conversation_id=conv_id)
        print(f"Assistant: {result.final_output}")
```

##### 2. Using `previous_response_id`

Another option is **response chaining**, where each turn links explicitly to the response ID from the previous turn.

```python
from agents import Agent, Runner

async def main():
    agent = Agent(name="Assistant", instructions="Reply very concisely.")

    previous_response_id = None

    while True:
        user_input = input("You: ")

        # Setting auto_previous_response_id=True enables response chaining automatically
        # for the first turn, even when there's no actual previous response ID yet.
        result = await Runner.run(
            agent,
            user_input,
            previous_response_id=previous_response_id,
            auto_previous_response_id=True,
        )
        previous_response_id = result.last_response_id
        print(f"Assistant: {result.final_output}")
```

If a run pauses for approval and you resume from a [`RunState`][agents.run_state.RunState], the SDK keeps the saved `conversation_id` / `previous_response_id` / `auto_previous_response_id` settings so the resumed turn continues in the same server-managed conversation.

`conversation_id` and `previous_response_id` are mutually exclusive. Use `conversation_id` when you want a named conversation resource that can be shared across systems. Use `previous_response_id` when you want the lightest Responses API continuation primitive from one turn to the next.

!!! note

    The SDK automatically retries `conversation_locked` errors with backoff. In server-managed
    conversation runs, it rewinds the internal conversation-tracker input before retrying so the
    same prepared items can be resent cleanly.

    In local session-based runs (which cannot be combined with `conversation_id`,
    `previous_response_id`, or `auto_previous_response_id`), the SDK also performs a best-effort
    rollback of recently persisted input items to reduce duplicate history entries after a retry.

    This compatibility retry happens even if you do not configure `ModelSettings.retry`. For
    broader opt-in retry behavior on model requests, see [Runner-managed retries](models/index.md#runner-managed-retries).

## Hooks and customization

### Call model input filter

Use `call_model_input_filter` to edit the model input right before the model call. The hook receives the current agent, context, and the combined input items (including session history when present) and returns a new `ModelInputData`.

The return value must be a [`ModelInputData`][agents.run.ModelInputData] object. Its `input` field is required and must be a list of input items. Returning any other shape raises a `UserError`.

```python
from agents import Agent, Runner, RunConfig
from agents.run import CallModelData, ModelInputData

def drop_old_messages(data: CallModelData[None]) -> ModelInputData:
    # Keep only the last 5 items and preserve existing instructions.
    trimmed = data.model_data.input[-5:]
    return ModelInputData(input=trimmed, instructions=data.model_data.instructions)

agent = Agent(name="Assistant", instructions="Answer concisely.")
result = Runner.run_sync(
    agent,
    "Explain quines",
    run_config=RunConfig(call_model_input_filter=drop_old_messages),
)
```

The runner passes a copy of the prepared input list to the hook, so you can trim, replace, or reorder it without mutating the caller's original list in place.

If you are using a session, `call_model_input_filter` runs after session history has already been loaded and merged with the current turn. Use [`session_input_callback`][agents.run.RunConfig.session_input_callback] when you want to customize that earlier merge step itself.

If you are using OpenAI server-managed conversation state with `conversation_id`, `previous_response_id`, or `auto_previous_response_id`, the hook runs on the prepared payload for the next Responses API call. That payload may already represent only the new-turn delta rather than a full replay of earlier history. Only the items you return are marked as sent for that server-managed continuation.

Set the hook per run via `run_config` to redact sensitive data, trim long histories, or inject additional system guidance.

## Errors and recovery

### Error handlers

All `Runner` entry points accept `error_handlers`, a dict keyed by error kind. The supported keys are `"max_turns"`, `"model_refusal"`, and `"invalid_final_output"`. Use them when you want to return a controlled final output instead of ending the run with the corresponding error.

```python
from agents import (
    Agent,
    RunErrorHandlerInput,
    RunErrorHandlerResult,
    Runner,
)

agent = Agent(name="Assistant", instructions="Be concise.")


def on_max_turns(_data: RunErrorHandlerInput[None]) -> RunErrorHandlerResult:
    return RunErrorHandlerResult(
        final_output="I couldn't finish within the turn limit. Please narrow the request.",
        include_in_history=False,
    )


result = Runner.run_sync(
    agent,
    "Analyze this long transcript",
    max_turns=3,
    error_handlers={"max_turns": on_max_turns},
)
print(result.final_output)
```

Use `"invalid_final_output"` when a model message does not validate against the agent's structured `output_type`, or when the model returns no structured final message. The handler can return an application-specific fallback, which the SDK validates against the same `output_type`. It does not retry the model call or replay any tool side effects. Returning `None` declines recovery. Without a fallback, non-empty validation failures continue to raise `ModelBehaviorError`, and empty structured responses retain the existing next-turn behavior.

```python
from pydantic import BaseModel

from agents import Agent, ModelBehaviorError, RunErrorHandlerInput, Runner


class Recipe(BaseModel):
    ingredients: list[str]
    recovered_from_invalid_output: bool = False


def on_invalid_final_output(data: RunErrorHandlerInput[None]) -> Recipe:
    assert isinstance(data.error, ModelBehaviorError)
    return Recipe(ingredients=[], recovered_from_invalid_output=True)


agent = Agent(
    name="Recipe assistant",
    instructions="Return a structured recipe.",
    output_type=Recipe,
)

result = Runner.run_sync(
    agent,
    "Plan tonight's dinner.",
    error_handlers={"invalid_final_output": on_invalid_final_output},
)
print(result.final_output)
```

`RunErrorHandlerResult.include_in_history` defaults to `True`. For a max-turns handler, this appends the synthesized fallback output to conversation history and persists it to the configured session. Set `include_in_history=False` when you want the fallback returned to the caller without adding it to result history or session storage.

Use `"model_refusal"` when a model refusal should produce an application-specific fallback instead of ending the run with `ModelRefusalError`.

```python
from pydantic import BaseModel

from agents import Agent, ModelRefusalError, RunErrorHandlerInput, Runner


class Recipe(BaseModel):
    ingredients: list[str]
    refusal_reason: str | None = None


def on_model_refusal(data: RunErrorHandlerInput[None]) -> Recipe:
    assert isinstance(data.error, ModelRefusalError)
    return Recipe(ingredients=[], refusal_reason=data.error.refusal)


agent = Agent(
    name="Recipe assistant",
    instructions="Return a structured recipe.",
    output_type=Recipe,
)

result = Runner.run_sync(
    agent,
    "Make me something unsafe.",
    error_handlers={"model_refusal": on_model_refusal},
)
print(result.final_output)
```

## Durable execution integrations and human-in-the-loop

For tool approval pause/resume patterns, start with the dedicated [Human-in-the-loop guide](human_in_the_loop.md). The integrations below are for durable orchestration when runs may span long waits, retries, or process restarts.

### Dapr

You can use the Agents SDK [Dapr](https://dapr.io) Diagrid integration to run durable, long-running agents that automatically recover from failures and support human-in-the-loop workflows. Dapr is a vendor-neutral, [CNCF](https://cncf.io) workflow orchestrator. Get started with Dapr and OpenAI agents [here](https://docs.diagrid.io/getting-started/quickstarts/ai-agents/?agentframework=openai).

### Temporal

You can use the Agents SDK [Temporal](https://temporal.io/) integration to run durable, long-running workflows, including human-in-the-loop tasks. View a demo of Temporal and the Agents SDK working in action to complete long-running tasks [in this video](https://www.youtube.com/watch?v=fFBZqzT4DD8), and [view docs here](https://github.com/temporalio/sdk-python/tree/main/temporalio/contrib/openai_agents). 

### Restate

You can use the Agents SDK [Restate](https://restate.dev/) integration for lightweight, durable agents, including human approval, handoffs, and session management. The integration requires Restate's single-binary runtime as a dependency, and supports running agents as processes/containers or serverless functions. Read the [overview](https://www.restate.dev/blog/durable-orchestration-for-ai-agents-with-restate-and-openai-sdk) or view the [docs](https://docs.restate.dev/ai) for more details.

### DBOS

You can use the Agents SDK [DBOS](https://dbos.dev/) integration to run reliable agents that preserve progress across failures and restarts. It supports long-running agents, human-in-the-loop workflows, and handoffs. It supports both sync and async methods. The integration requires only a SQLite or Postgres database. View the integration [repo](https://github.com/dbos-inc/dbos-openai-agents) and the [docs](https://docs.dbos.dev/integrations/openai-agents) for more details.

## Exceptions

The SDK raises exceptions in certain cases. The full list is in [`agents.exceptions`][]. As an overview:

-   [`AgentsException`][agents.exceptions.AgentsException]: This is the base class for all exceptions that the SDK raises. It serves as a generic type from which all other specific exceptions are derived.
-   [`MaxTurnsExceeded`][agents.exceptions.MaxTurnsExceeded]: This exception is raised when the agent's run exceeds the `max_turns` limit passed to the `Runner.run`, `Runner.run_sync`, or `Runner.run_streamed` methods. It indicates that the agent could not complete its task within the specified number of agent-loop turns (LLM calls). Set `max_turns=None` to disable the limit.
-   [`ModelBehaviorError`][agents.exceptions.ModelBehaviorError]: This exception occurs when the underlying model (LLM) produces unexpected or invalid outputs. This can include:
    -   Malformed JSON: When the model provides a malformed JSON structure for tool calls or in its direct output, especially if a specific `output_type` is defined.
    -   Unexpected tool-related failures: When the model fails to use tools in an expected manner
-   [`ToolTimeoutError`][agents.exceptions.ToolTimeoutError]: This exception is raised when a function tool call exceeds its configured timeout and the tool uses `timeout_behavior="raise_exception"`.
-   [`UserError`][agents.exceptions.UserError]: This exception is raised when you (the person writing code using the SDK) make an error while using the SDK. This typically results from incorrect code implementation, invalid configuration, or misuse of the SDK's API.
-   [`InputGuardrailTripwireTriggered`][agents.exceptions.InputGuardrailTripwireTriggered], [`OutputGuardrailTripwireTriggered`][agents.exceptions.OutputGuardrailTripwireTriggered]: `InputGuardrailTripwireTriggered` is raised when an input guardrail's conditions are met, and `OutputGuardrailTripwireTriggered` is raised when an output guardrail's conditions are met. Input guardrails check incoming messages before processing, while output guardrails check the agent's final response before delivery.

---

## Tools

# Tools

Tools let agents take actions: things like fetching data, running code, calling external APIs, and even using a computer. The SDK supports five categories:

-   Hosted OpenAI tools: execute for the model on OpenAI servers.
-   Local/runtime execution tools: `ComputerTool` and `ApplyPatchTool` always run in your environment, while `ShellTool` can run locally or in a hosted container.
-   `FunctionTool` instances: wrap any Python function as a tool.
-   Agents as tools: expose an agent as a callable tool without a full handoff.
-   Experimental: Codex tool: run workspace-scoped Codex tasks from a tool call.

## Choosing a tool type

Use this page as a catalog, then jump to the section that matches the runtime you control.

| If you want to... | Start here |
| --- | --- |
| Use OpenAI-managed tools (web search, file search, code interpreter, hosted MCP, image generation) | [Hosted tools](#hosted-tools) |
| Defer large tool surfaces until runtime with tool search | [Hosted tool search](#hosted-tool-search) |
| Coordinate several tool calls from generated JavaScript | [Programmatic Tool Calling](#programmatic-tool-calling) |
| Run tools in your own process or environment | [Local runtime tools](#local-runtime-tools) |
| Wrap Python functions as tools | [Function tools](#function-tools) |
| Let one agent call another without a handoff | [Agents as tools](#agents-as-tools) |
| Run workspace-scoped Codex tasks from an agent | [Experimental: Codex tool](#experimental-codex-tool) |

## Hosted tools

OpenAI offers a few built-in tools when using the [`OpenAIResponsesModel`][agents.models.openai_responses.OpenAIResponsesModel]:

-   The [`WebSearchTool`][agents.tool.WebSearchTool] lets an agent search the web.
-   The [`FileSearchTool`][agents.tool.FileSearchTool] allows retrieving information from your OpenAI Vector Stores.
-   The [`CodeInterpreterTool`][agents.tool.CodeInterpreterTool] lets the LLM execute code in a sandboxed environment.
-   The [`HostedMCPTool`][agents.tool.HostedMCPTool] exposes a remote MCP server's tools to the model.
-   The [`ImageGenerationTool`][agents.tool.ImageGenerationTool] generates images from a prompt.
-   The [`ToolSearchTool`][agents.tool.ToolSearchTool] lets the model load deferred tools, namespaces, or hosted MCP servers on demand.
-   The [`ProgrammaticToolCallingTool`][agents.tool.ProgrammaticToolCallingTool] lets the model coordinate eligible tools from generated JavaScript.

Advanced hosted search options:

-   `FileSearchTool` supports `filters`, `ranking_options`, and `include_search_results` in addition to `vector_store_ids` and `max_num_results`. Set `max_num_results` to an integer from 1 through 50; `None` or zero uses the provider default.
-   `WebSearchTool` supports `filters`, `user_location`, and `search_context_size`.

```python
from agents import Agent, FileSearchTool, Runner, WebSearchTool

agent = Agent(
    name="Assistant",
    tools=[
        WebSearchTool(),
        FileSearchTool(
            max_num_results=3,
            vector_store_ids=["VECTOR_STORE_ID"],
        ),
    ],
)

async def main():
    result = await Runner.run(agent, "Which coffee shop should I go to, taking into account my preferences and the weather today in SF?")
    print(result.final_output)
```

### Hosted tool search

Tool search lets OpenAI Responses models defer large tool surfaces until runtime, so the model loads only the subset it needs for the current turn. This is useful when you have many function tools, namespace groups, or hosted MCP servers and want to reduce tool-schema tokens without exposing every tool up front.

Start with hosted tool search when the candidate tools are already known when you build the agent. If your application needs to decide what to load dynamically, the Responses API also supports client-executed tool search, but the standard `Runner` does not auto-execute that mode.

```python
from typing import Annotated

from agents import Agent, Runner, ToolSearchTool, tool_namespace
from agents.decorators import tool


@tool(defer_loading=True)
def get_customer_profile(
    customer_id: Annotated[str, "The customer ID to look up."],
) -> str:
    """Fetch a CRM customer profile."""
    return f"profile for {customer_id}"


@tool(defer_loading=True)
def list_open_orders(
    customer_id: Annotated[str, "The customer ID to look up."],
) -> str:
    """List open orders for a customer."""
    return f"open orders for {customer_id}"


crm_tools = tool_namespace(
    name="crm",
    description="CRM tools for customer lookups.",
    tools=[get_customer_profile, list_open_orders],
)


agent = Agent(
    name="Operations assistant",
    model="gpt-5.6-sol",
    instructions="Load the crm namespace before using CRM tools.",
    tools=[*crm_tools, ToolSearchTool()],
)

result = await Runner.run(agent, "Look up customer_42 and list their open orders.")
print(result.final_output)
```

What to know:

-   Hosted tool search is available only with OpenAI Responses models. The current Python SDK support depends on `openai>=2.25.0`.
-   Add exactly one `ToolSearchTool()` when you configure deferred-loading surfaces on an agent.
-   Searchable surfaces include `@function_tool(defer_loading=True)`, `tool_namespace(name=..., description=..., tools=[...])`, and `HostedMCPTool(tool_config={..., "defer_loading": True})`.
-   Deferred-loading function tools must be paired with `ToolSearchTool()`. Namespace-only setups may also use `ToolSearchTool()` to let the model load the right group on demand.
-   `tool_namespace()` groups `FunctionTool` instances under a shared namespace name and description. This is usually the best fit when you have many related tools, such as `crm`, `billing`, or `shipping`.
-   OpenAI's official best-practice guidance is [Use namespaces where possible](https://developers.openai.com/api/docs/guides/tools-tool-search#use-namespaces-where-possible).
-   Prefer namespaces or hosted MCP servers over many individually deferred functions when possible. They usually give the model a better high-level search surface and better token savings.
-   Namespaces can mix immediate and deferred tools. Tools without `defer_loading=True` remain callable immediately, while deferred tools in the same namespace are loaded through tool search.
-   As a rule of thumb, keep each namespace fairly small, ideally fewer than 10 functions.
-   Named `tool_choice` cannot target bare namespace names or deferred-only tools. Prefer `auto`, `required`, or a real top-level callable tool name.
-   `ToolSearchTool(execution="client")` is for manual Responses orchestration. If the model emits a client-executed `tool_search_call`, the standard `Runner` raises instead of executing it for you.
-   Tool search activity appears in [`RunResult.new_items`](results.md#new-items) and in [`RunItemStreamEvent`](streaming.md#run-item-event-names) with dedicated item and event types.
-   See `examples/tools/tool_search.py` for complete runnable examples covering both namespaced loading and top-level deferred tools.
-   Official platform guide: [Tool search](https://developers.openai.com/api/docs/guides/tools-tool-search).

### Programmatic Tool Calling

Programmatic Tool Calling lets a supported OpenAI Responses model generate JavaScript that calls eligible tools, combines their outputs, and returns one result to the model. It is useful for bounded workflows that benefit from loops, branching, parallel calls, or intermediate calculations without a model round trip after every tool call.

The generated program runs in a fresh hosted V8 environment. It does not have Node.js APIs, filesystem or network access, or a persistent process. The program can interact only with tools that you explicitly allow.

```python
from pydantic import BaseModel

from agents import (
    Agent,
    ModelSettings,
    ProgrammaticToolCallingTool,
    Runner,
)
from agents.decorators import tool


class InventoryOutput(BaseModel):
    sku: str
    available_units: int


@tool(allowed_callers=["programmatic"])
def get_inventory(sku: str) -> InventoryOutput:
    return InventoryOutput(sku=sku, available_units=42)


agent = Agent(
    name="Inventory planner",
    model="gpt-5.6",
    model_settings=ModelSettings(tool_choice="programmatic_tool_calling"),
    tools=[get_inventory, ProgrammaticToolCallingTool()],
)

result = Runner.run_sync(agent, "Check inventory for desk-lamp and summarize it.")
print(result.final_output)
```

What to know:

-   Programmatic Tool Calling is available only with supported OpenAI Responses models. `ProgrammaticToolCallingTool()` and `tool_choice="programmatic_tool_calling"` are rejected by Chat Completions models and non-Responses backends.
-   Add at most one `ProgrammaticToolCallingTool()` to an agent. The agent must also expose at least one programmatically callable tool, a `ToolSearchTool()` backed by a namespace, deferred function, or deferred hosted MCP server, or an opaque prompt-managed tool surface. A bare `ToolSearchTool()` without a searchable surface is rejected.
-   `allowed_callers` controls how a tool may be invoked. Omitting it allows direct model calls only. Use `["programmatic"]` for program-only access or `["direct", "programmatic"]` to allow both.
-   SDK tool types that can opt in are `FunctionTool`, `CustomTool`, `ShellTool`, `ApplyPatchTool`, `HostedMCPTool`, and `CodeInterpreterTool`. Function, custom, shell, and apply-patch tools expose `allowed_callers` directly. For hosted MCP and code interpreter, set `allowed_callers` inside `tool_config`.
-   For `@function_tool(allowed_callers=[...])`, a structured return annotation such as a Pydantic model, TypedDict, or dataclass automatically becomes a strict object output schema, and the returned value is validated against that schema before it is returned to the program. Use `output_type=...` when the function has no usable annotation, or the lower-level `output_json_schema={...}` escape hatch when you already have a strict object schema. `output_type` and `output_json_schema` are mutually exclusive. Return annotations of `str`, `Any`, or `None` do not create an output schema. For a schema-backed program-owned call, the default failure formatter is disabled because its free-form text does not satisfy the output schema. A handler exception therefore propagates unless you provide a custom `failure_error_function` that returns schema-conforming JSON.
-   Program-owned SDK tools still use the normal Runner lifecycle. Tool input and output guardrails, hooks, timeouts, concurrency limits, approvals, sessions, and `RunState` pause/resume behavior continue to apply, and the SDK preserves each child call's program caller relationship.
-   Model-request retries use a stricter replay-safety boundary whenever `ProgrammaticToolCallingTool()` is present, even before a program executes. The SDK disables provider-managed retries and WebSocket pre-event retries for these requests. A Runner retry policy retries only when provider advice explicitly marks the replay safe; `retry_policies.network_error()` by itself does not override this boundary.
-   Approval-sensitive or high-impact tools are usually better kept as direct calls so a person can review each action before it becomes part of a larger program. If a program-owned call pauses for approval, resolve the interruption through `RunState` and resume the original run as usual.
-   Programmatic Tool Calling can be combined with [hosted tool search](#hosted-tool-search). The model must load deferred tools before a generated program can call them.
-   A `program` item and its ordinary program-owned child tool calls appear as [`ToolCallItem`][agents.items.ToolCallItem] entries. The matching `program_output` appears as a [`ToolCallOutputItem`][agents.items.ToolCallOutputItem]. Hosted MCP approval requests and tool catalogs use specialized MCP items and stream events instead. See [Results](results.md#new-items) and [Streaming](streaming.md#run-item-event-names) for inspection details.
-   See `examples/tools/programmatic_tool_calling.py` for a complete concurrent inventory-planning example.
-   Official platform guide: [Programmatic Tool Calling](https://developers.openai.com/api/docs/guides/tools-programmatic-tool-calling).

### Hosted container shell + skills

`ShellTool` also supports OpenAI-hosted container execution. Use this mode when you want the model to run shell commands in a managed container instead of your local runtime.

```python
from agents import Agent, Runner, ShellTool, ShellToolSkillReference

csv_skill: ShellToolSkillReference = {
    "type": "skill_reference",
    "skill_id": "skill_698bbe879adc81918725cbc69dcae7960bc5613dadaed377",
    "version": "1",
}

agent = Agent(
    name="Container shell agent",
    model="gpt-5.6-sol",
    instructions="Use the mounted skill when helpful.",
    tools=[
        ShellTool(
            environment={
                "type": "container_auto",
                "network_policy": {"type": "disabled"},
                "skills": [csv_skill],
            }
        )
    ],
)

result = await Runner.run(
    agent,
    "Use the configured skill to analyze CSV files in /mnt/data and summarize totals by region.",
)
print(result.final_output)
```

To reuse an existing container in later runs, set `environment={"type": "container_reference", "container_id": "cntr_..."}`.

What to know:

-   Hosted shell is available through the Responses API shell tool.
-   `container_auto` provisions a container for the request; `container_reference` reuses an existing one.
-   `container_auto` can also include `file_ids` and `memory_limit`.
-   `environment.skills` accepts skill references and inline skill bundles.
-   With hosted environments, do not set `executor`, `needs_approval`, or `on_approval` on `ShellTool`.
-   `network_policy` supports `disabled` and `allowlist` modes.
-   In allowlist mode, `network_policy.domain_secrets` can inject domain-scoped secrets by name.
-   See `examples/tools/container_shell_skill_reference.py` and `examples/tools/container_shell_inline_skill.py` for complete examples.
-   OpenAI platform guides: [Shell](https://platform.openai.com/docs/guides/tools-shell) and [Skills](https://platform.openai.com/docs/guides/tools-skills).

## Local runtime tools

Local runtime tools execute outside the model response itself. The model still decides when to call them, but your application or configured execution environment performs the actual work.

`ComputerTool` and `ApplyPatchTool` always require local implementations that you provide. `ShellTool` spans both modes: use the hosted-container configuration above when you want managed execution, or the local runtime configuration below when you want commands to run in your own process.

Local runtime tools require you to supply implementations:

-   [`ComputerTool`][agents.tool.ComputerTool]: implement the [`Computer`][agents.computer.Computer] or [`AsyncComputer`][agents.computer.AsyncComputer] interface to enable GUI/browser automation.
-   [`ShellTool`][agents.tool.ShellTool]: the latest shell tool for both local execution and hosted container execution.
-   [`LocalShellTool`][agents.tool.LocalShellTool]: legacy local-shell integration.
-   [`ApplyPatchTool`][agents.tool.ApplyPatchTool]: implement [`ApplyPatchEditor`][agents.editor.ApplyPatchEditor] to apply diffs locally.
-   Local shell skills are available with `ShellTool(environment={"type": "local", "skills": [...]})`.

Shell action timeouts use positive integer milliseconds for a finite timeout. The SDK treats both `0` and `None` as no explicit timeout before calling a local `ShellTool` executor because zero does not have a portable meaning across executor implementations; other values are rejected before executor invocation. This is specific to the timeout field: `max_output_length=0` remains a supported request for empty captured output.

### ComputerTool and the Responses computer tool

`ComputerTool` is still a local harness: you provide a [`Computer`][agents.computer.Computer] or [`AsyncComputer`][agents.computer.AsyncComputer] implementation, and the SDK maps that harness onto the OpenAI Responses API computer surface.

For explicit [`gpt-5.5`](https://developers.openai.com/api/docs/models/gpt-5.5) requests, the SDK sends the GA built-in tool payload `{"type": "computer"}`. For requests to the older `computer-use-preview` model, the SDK continues to send the preview payload `{"type": "computer_use_preview", "environment": ..., "display_width": ..., "display_height": ...}`. This mirrors the platform migration described in OpenAI's [Computer use guide](https://developers.openai.com/api/docs/guides/tools-computer-use/):

-   Model: `computer-use-preview` -> `gpt-5.5`
-   Tool selector: `computer_use_preview` -> `computer`
-   Computer call shape: one `action` per `computer_call` -> batched `actions[]` on `computer_call`
-   Truncation: `ModelSettings(truncation="auto")` required on the preview path -> not required on the GA path

The SDK chooses that wire shape from the effective model on the actual Responses request. If you use a prompt template and the request omits `model` because the prompt owns it, the SDK keeps the preview-compatible computer payload unless you either keep `model="gpt-5.5"` explicit or force the GA selector with `ModelSettings(tool_choice="computer")` or `ModelSettings(tool_choice="computer_use")`.

When a [`ComputerTool`][agents.tool.ComputerTool] is present, `tool_choice="computer"`, `"computer_use"`, and `"computer_use_preview"` are all accepted and normalized to the built-in selector that matches the effective request model. Without a `ComputerTool`, those strings still behave like ordinary function names.

This distinction matters when `ComputerTool` is backed by a [`ComputerProvider`][agents.tool.ComputerProvider] factory. The GA `computer` payload does not need `environment` or dimensions at serialization time, so serialization can occur before a factory has produced a `Computer` or `AsyncComputer` instance. Preview-compatible serialization still needs a resolved `Computer` or `AsyncComputer` instance so the SDK can send `environment`, `display_width`, and `display_height`.

At runtime, both paths still use the same local harness. Preview responses emit `computer_call` items with a single `action`; `gpt-5.5` can emit batched `actions[]`, and the SDK executes them in order before producing a `computer_call_output` screenshot item. See `examples/tools/computer_use.py` for a runnable Playwright-based harness.

```python
from agents import Agent, ApplyPatchTool, ShellTool
from agents.computer import AsyncComputer
from agents.editor import ApplyPatchResult, ApplyPatchOperation, ApplyPatchEditor


class NoopComputer(AsyncComputer):
    environment = "browser"
    dimensions = (1024, 768)
    async def screenshot(self): return ""
    async def click(self, x, y, button): ...
    async def double_click(self, x, y): ...
    async def scroll(self, x, y, scroll_x, scroll_y): ...
    async def type(self, text): ...
    async def wait(self): ...
    async def move(self, x, y): ...
    async def keypress(self, keys): ...
    async def drag(self, path): ...


class NoopEditor(ApplyPatchEditor):
    async def create_file(self, op: ApplyPatchOperation): return ApplyPatchResult(status="completed")
    async def update_file(self, op: ApplyPatchOperation): return ApplyPatchResult(status="completed")
    async def delete_file(self, op: ApplyPatchOperation): return ApplyPatchResult(status="completed")


async def run_shell(request):
    return "shell output"


agent = Agent(
    name="Local tools agent",
    tools=[
        ShellTool(executor=run_shell),
        ApplyPatchTool(editor=NoopEditor()),
        # ComputerTool expects a Computer/AsyncComputer implementation; omitted here for brevity.
    ],
)
```

## Function tools

You can use any Python function as a tool. The Agents SDK will set up the tool automatically:

-   The name of the tool will be the name of the Python function (or you can provide a name)
-   Tool description will be taken from the docstring of the function (or you can provide a description)
-   The schema for the function inputs is automatically created from the function's arguments
-   Descriptions for each input are taken from the docstring of the function, unless disabled

Tools created by `@tool` expose the original Python callable through the read-only `__wrapped__` attribute. This is useful for inspection and testing, but calling it directly bypasses the tool runtime pipeline, including schema validation, context injection, guardrails, timeouts, failure handling, and tracing. Hand-built `FunctionTool` instances do not expose `__wrapped__`.

We use Python's `inspect` module to extract the function signature, along with [`griffe`](https://mkdocstrings.github.io/griffe/) to parse docstrings and `pydantic` for schema creation.

When you are using OpenAI Responses models, `@function_tool(defer_loading=True)` hides a function tool until `ToolSearchTool()` loads it. You can also group related function tools with [`tool_namespace()`][agents.tool.tool_namespace]. See [Hosted tool search](#hosted-tool-search) for the full setup and constraints.

```python
import json

from typing_extensions import TypedDict, Any

from agents import Agent, FunctionTool, RunContextWrapper
from agents.decorators import tool


class Location(TypedDict):
    lat: float
    long: float

@tool  # (1)!
async def fetch_weather(location: Location) -> str:
    # (2)!
    """Fetch the weather for a given location.

    Args:
        location: The location to fetch the weather for.
    """
    # In real life, we'd fetch the weather from a weather API
    return "sunny"


@tool(name_override="fetch_data")  # (3)!
def read_file(ctx: RunContextWrapper[Any], path: str, directory: str | None = None) -> str:
    """Read the contents of a file.

    Args:
        path: The path to the file to read.
        directory: The directory to read the file from.
    """
    # In real life, we'd read the file from the file system
    return "<file contents>"


agent = Agent(
    name="Assistant",
    tools=[fetch_weather, read_file],  # (4)!
)

for tool in agent.tools:
    if isinstance(tool, FunctionTool):
        print(tool.name)
        print(tool.description)
        print(json.dumps(tool.params_json_schema, indent=2))
        print()

```

1.  You can use any Python types as arguments to your functions, and the function can be sync or async.
2.  Docstrings, if present, are used to capture descriptions and argument descriptions
3.  Functions can optionally take the run context as their first argument. You can also set overrides, like the name of the tool, description, which docstring style to use, etc.
4.  You can pass the decorated functions to the list of tools.

??? note "Expand to see output"

    ```
    fetch_weather
    Fetch the weather for a given location.
    {
    "$defs": {
      "Location": {
        "properties": {
          "lat": {
            "title": "Lat",
            "type": "number"
          },
          "long": {
            "title": "Long",
            "type": "number"
          }
        },
        "required": [
          "lat",
          "long"
        ],
        "title": "Location",
        "type": "object"
      }
    },
    "properties": {
      "location": {
        "$ref": "#/$defs/Location",
        "description": "The location to fetch the weather for."
      }
    },
    "required": [
      "location"
    ],
    "title": "fetch_weather_args",
    "type": "object"
    }

    fetch_data
    Read the contents of a file.
    {
    "properties": {
      "path": {
        "description": "The path to the file to read.",
        "title": "Path",
        "type": "string"
      },
      "directory": {
        "anyOf": [
          {
            "type": "string"
          },
          {
            "type": "null"
          }
        ],
        "default": null,
        "description": "The directory to read the file from.",
        "title": "Directory"
      }
    },
    "required": [
      "path"
    ],
    "title": "fetch_data_args",
    "type": "object"
    }
    ```

### Returning images or files from function tools

In addition to returning text outputs, you can return one or many images or files as the output of a function tool. To do so, you can return any of:

-   Images: [`ToolOutputImage`][agents.tool.ToolOutputImage] (or the TypedDict version, [`ToolOutputImageDict`][agents.tool.ToolOutputImageDict])
-   Files: [`ToolOutputFileContent`][agents.tool.ToolOutputFileContent] (or the TypedDict version, [`ToolOutputFileContentDict`][agents.tool.ToolOutputFileContentDict])
-   Text: either a string or stringable objects, or [`ToolOutputText`][agents.tool.ToolOutputText] (or the TypedDict version, [`ToolOutputTextDict`][agents.tool.ToolOutputTextDict])

### Custom function tools

Sometimes, you don't want to use a Python function as a tool. You can directly create a [`FunctionTool`][agents.tool.FunctionTool] if you prefer. You'll need to provide:

-   `name`
-   `description`
-   `params_json_schema`, which is the JSON schema for the arguments
-   `on_invoke_tool`, which is an async function that receives a [`ToolContext`][agents.tool_context.ToolContext] and the arguments as a JSON string, and returns tool output (for example, text, structured tool output objects, or a list of outputs).

```python
from typing import Any

from pydantic import BaseModel

from agents import RunContextWrapper, FunctionTool



def do_some_work(data: str) -> str:
    return "done"


class FunctionArgs(BaseModel):
    username: str
    age: int


async def run_function(ctx: RunContextWrapper[Any], args: str) -> str:
    parsed = FunctionArgs.model_validate_json(args)
    return do_some_work(data=f"{parsed.username} is {parsed.age} years old")


tool = FunctionTool(
    name="process_user",
    description="Processes extracted user data",
    params_json_schema=FunctionArgs.model_json_schema(),
    on_invoke_tool=run_function,
)
```

### Automatic argument and docstring parsing

As mentioned before, we automatically parse the function signature to extract the schema for the tool, and we parse the docstring to extract descriptions for the tool and for individual arguments. Some notes on that:

1. The signature parsing is done via the `inspect` module. We use type annotations to understand the types for the arguments, and dynamically build a Pydantic model to represent the overall schema. It supports most types, including Python primitives, Pydantic models, TypedDicts, and more.
2. We use `griffe` to parse docstrings. Supported docstring formats are `google`, `sphinx` and `numpy`. We attempt to automatically detect the docstring format, but this is best-effort and you can explicitly set it when calling `function_tool`. You can also disable docstring parsing by setting `use_docstring_info` to `False`. For Google-style docstrings, the parser also accepts an `Args:`, `Arguments:`, `Params:`, or `Parameters:` section immediately after summary text without an intervening blank line.

The code for the schema extraction lives in [`agents.function_schema`][].

### Constraining and describing arguments with Pydantic Field

You can use Pydantic's [`Field`](https://docs.pydantic.dev/latest/concepts/fields/) to add constraints (e.g. min/max for numbers, length or pattern for strings) and descriptions to tool arguments. As in Pydantic, both forms are supported: default-based (`arg: int = Field(..., ge=1)`) and `Annotated` (`arg: Annotated[int, Field(..., ge=1)]`). The generated JSON schema and validation include these constraints.

```python
from typing import Annotated
from pydantic import Field
from agents.decorators import tool

# Default-based form
@tool
def score_a(score: int = Field(..., ge=0, le=100, description="Score from 0 to 100")) -> str:
    return f"Score recorded: {score}"

# Annotated form
@tool
def score_b(score: Annotated[int, Field(..., ge=0, le=100, description="Score from 0 to 100")]) -> str:
    return f"Score recorded: {score}"
```

### Function tool timeouts

You can set per-call timeouts for async function tools with `@function_tool(timeout=...)`.

```python
import asyncio
from agents import Agent
from agents.decorators import tool


@tool(timeout=2.0)
async def slow_lookup(query: str) -> str:
    await asyncio.sleep(10)
    return f"Result for {query}"


agent = Agent(
    name="Timeout demo",
    instructions="Use tools when helpful.",
    tools=[slow_lookup],
)
```

When a timeout is reached, the default behavior is `timeout_behavior="error_as_result"`, which sends a model-visible timeout message (for example, `Tool 'slow_lookup' timed out after 2 seconds.`).

You can control timeout handling:

-   `timeout_behavior="error_as_result"` (default): return a timeout message to the model so it can recover.
-   `timeout_behavior="raise_exception"`: raise [`ToolTimeoutError`][agents.exceptions.ToolTimeoutError] and fail the run.
-   `timeout_error_function=...`: customize the timeout message when using `error_as_result`.

```python
import asyncio
from agents import Agent, Runner, ToolTimeoutError
from agents.decorators import tool


@tool(timeout=1.5, timeout_behavior="raise_exception")
async def slow_tool() -> str:
    await asyncio.sleep(5)
    return "done"


agent = Agent(name="Timeout hard-fail", tools=[slow_tool])

try:
    await Runner.run(agent, "Run the tool")
except ToolTimeoutError as e:
    print(f"{e.tool_name} timed out in {e.timeout_seconds} seconds")
```

!!! note

    Timeout configuration is supported only for async `@function_tool` handlers.

### Handling errors in function tools

When you create a function tool via `@function_tool`, you can pass a `failure_error_function`. This is a function that provides an error response to the LLM in case the tool call crashes.

-   By default (i.e. if you don't pass anything), it runs a `default_tool_error_function` which tells the LLM an error occurred.
-   If you pass your own error function, it runs that instead, and sends the response to the LLM.
-   If you explicitly pass `None`, then any tool call errors will be re-raised for you to handle. This could be a `ModelBehaviorError` if the model produced invalid JSON, or a `UserError` if your code crashed, etc.

```python
from agents import RunContextWrapper
from agents.decorators import tool
from typing import Any

def my_custom_error_function(context: RunContextWrapper[Any], error: Exception) -> str:
    """A custom function to provide a user-friendly error message."""
    print(f"A tool call failed with the following error: {error}")
    return "An internal server error occurred. Please try again later."

@tool(failure_error_function=my_custom_error_function)
def get_user_profile(user_id: str) -> str:
    """Fetches a user profile from a mock API.
     This function demonstrates a 'flaky' or failing API call.
    """
    if user_id == "user_123":
        return "User profile for user_123 successfully retrieved."
    else:
        raise ValueError(f"Could not retrieve profile for user_id: {user_id}. API returned an error.")

```

If you are manually creating a `FunctionTool` object, then you must handle errors inside the `on_invoke_tool` function.

## Agents as tools

In some workflows, you may want a central agent to orchestrate a network of specialized agents, instead of handing off control. You can do this by modeling agents as tools.

```python
import asyncio

from agents import Agent, Runner

spanish_agent = Agent(
    name="Spanish agent",
    instructions="You translate the user's message to Spanish",
)

french_agent = Agent(
    name="French agent",
    instructions="You translate the user's message to French",
)

orchestrator_agent = Agent(
    name="orchestrator_agent",
    instructions=(
        "You are a translation agent. You use the tools given to you to translate. "
        "If asked for multiple translations, you call the relevant tools."
    ),
    tools=[
        spanish_agent.as_tool(
            tool_name="translate_to_spanish",
            tool_description="Translate the user's message to Spanish",
        ),
        french_agent.as_tool(
            tool_name="translate_to_french",
            tool_description="Translate the user's message to French",
        ),
    ],
)

async def main():
    result = await Runner.run(orchestrator_agent, input="Say 'Hello, how are you?' in Spanish.")
    print(result.final_output)


if __name__ == "__main__":
    asyncio.run(main())
```

### Customizing tool-agents

`agent.as_tool` is a convenience method for turning an agent into a tool. It supports common runtime options such as `max_turns`, `run_config`, `hooks`, `previous_response_id`, `conversation_id`, `session`, and `needs_approval`. It also supports structured input with `parameters`, `input_builder`, and `include_input_schema`.

The state options configure the nested agent run started by the tool call; the parent run's conversation state is not inherited automatically. To share client-managed history between the parent and nested runs, explicitly pass the same `session` to both. As with `Runner.run`, choose one state strategy for the nested run: a client-managed `session`, or server-managed continuation through `previous_response_id` or `conversation_id`.

```python
from agents.decorators import tool


@tool
async def run_my_agent() -> str:
    """A tool that runs the agent with custom configs"""

    agent = Agent(name="My agent", instructions="...")

    result = await Runner.run(
        agent,
        input="...",
        max_turns=5,
        run_config=...
    )

    return str(result.final_output)
```

### Structured input for tool-agents

By default, `Agent.as_tool()` expects an object with one string field, `input` (`{"input": "..."}`), but you can expose a structured schema by passing `parameters` (a Pydantic model type or a dataclass type).

Additional options:

- `include_input_schema=True` includes the full JSON Schema in the generated nested input.
- `input_builder=...` lets you fully customize how structured tool arguments become nested agent input.
- `RunContextWrapper.tool_input` contains the parsed structured payload inside the nested run context.

```python
from pydantic import BaseModel, Field


class TranslationInput(BaseModel):
    text: str = Field(description="Text to translate.")
    source: str = Field(description="Source language.")
    target: str = Field(description="Target language.")


translator_tool = translator_agent.as_tool(
    tool_name="translate_text",
    tool_description="Translate text between languages.",
    parameters=TranslationInput,
    include_input_schema=True,
)
```

See `examples/agent_patterns/agents_as_tools_structured.py` for a complete runnable example.

### Approval gates for tool-agents

`Agent.as_tool(..., needs_approval=...)` uses the same approval flow as `function_tool`. If approval is required, the run pauses and pending items appear in `result.interruptions`; then use `result.to_state()` and resume after calling `state.approve(...)` or `state.reject(...)`. See the [Human-in-the-loop guide](human_in_the_loop.md) for the full pause/resume pattern.

### Custom output extraction

In certain cases, you might want to modify the output of the tool-agents before returning it to the central agent. This may be useful if you want to:

-   Extract a specific piece of information (e.g., a JSON payload) from the sub-agent's chat history.
-   Convert or reformat the agent’s final answer (e.g., transform Markdown into plain text or CSV).
-   Validate the output or provide a fallback value when the agent’s response is missing or malformed.

You can do this by supplying the `custom_output_extractor` argument to the `as_tool` method:

```python
async def extract_json_payload(run_result: RunResult) -> str:
    # Scan the agent’s outputs in reverse order until we find a JSON-like message from a tool call.
    for item in reversed(run_result.new_items):
        if isinstance(item, ToolCallOutputItem) and item.output.strip().startswith("{"):
            return item.output.strip()
    # Fallback to an empty JSON object if nothing was found
    return "{}"


json_tool = data_agent.as_tool(
    tool_name="get_data_json",
    tool_description="Run the data agent and return only its JSON payload",
    custom_output_extractor=extract_json_payload,
)
```

Inside a custom extractor, the nested [`RunResult`][agents.result.RunResult] also exposes [`agent_tool_invocation`][agents.result.RunResultBase.agent_tool_invocation], which is useful when you need the outer tool name, call ID, or raw arguments while post-processing the nested result. See the [Results guide](results.md#agent-as-tool-metadata).

### Streaming nested agent runs

Pass an `on_stream` callback to `as_tool` to listen to streaming events emitted by the nested agent while still returning its final output once the stream completes.

```python
from agents import AgentToolStreamEvent


async def handle_stream(event: AgentToolStreamEvent) -> None:
    # Inspect the underlying StreamEvent along with agent metadata.
    print(f"[stream] {event['agent'].name} :: {event['event'].type}")


billing_agent_tool = billing_agent.as_tool(
    tool_name="billing_helper",
    tool_description="Answer billing questions.",
    on_stream=handle_stream,  # Can be sync or async.
)
```

What to expect:

- Event types mirror `StreamEvent["type"]`: `raw_response_event`, `run_item_stream_event`, `agent_updated_stream_event`.
- Providing `on_stream` automatically runs the nested agent in streaming mode and drains the stream before returning the final output.
- The handler may be synchronous or asynchronous; each event is delivered in order as it arrives.
- `tool_call` is present when the tool is invoked via a model tool call; direct calls may leave it `None`.
- See `examples/agent_patterns/agents_as_tools_streaming.py` for a complete runnable sample.

### Conditional tool enabling

You can conditionally enable or disable agent tools at runtime using the `is_enabled` parameter. This allows you to dynamically filter which tools are available to the LLM based on context, user preferences, or runtime conditions.

```python
import asyncio
from agents import Agent, AgentBase, Runner, RunContextWrapper
from pydantic import BaseModel

class LanguageContext(BaseModel):
    language_preference: str = "french_spanish"

def french_enabled(ctx: RunContextWrapper[LanguageContext], agent: AgentBase) -> bool:
    """Enable French for French+Spanish preference."""
    return ctx.context.language_preference == "french_spanish"

# Create specialized agents
spanish_agent = Agent(
    name="spanish_agent",
    instructions="You respond in Spanish. Always reply to the user's question in Spanish.",
)

french_agent = Agent(
    name="french_agent",
    instructions="You respond in French. Always reply to the user's question in French.",
)

# Create orchestrator with conditional tools
orchestrator = Agent(
    name="orchestrator",
    instructions=(
        "You are a multilingual assistant. You use the tools given to you to respond to users. "
        "You must call ALL available tools to provide responses in different languages. "
        "You never respond in languages yourself, you always use the provided tools."
    ),
    tools=[
        spanish_agent.as_tool(
            tool_name="respond_spanish",
            tool_description="Respond to the user's question in Spanish",
            is_enabled=True,  # Always enabled
        ),
        french_agent.as_tool(
            tool_name="respond_french",
            tool_description="Respond to the user's question in French",
            is_enabled=french_enabled,
        ),
    ],
)

async def main():
    context = LanguageContext(language_preference="french_spanish")
    result = await Runner.run(orchestrator, "How are you?", context=context)
    print(result.final_output)

asyncio.run(main())
```

The `is_enabled` parameter accepts:

-   **Boolean values**: `True` (always enabled) or `False` (always disabled)
-   **Callable functions**: Functions that take `(context, agent)` and return a boolean
-   **Async functions**: Async functions for complex conditional logic

Disabled tools are completely hidden from the LLM at runtime, making this useful for:

-   Feature gating based on user permissions
-   Environment-specific tool availability (dev vs prod)
-   A/B testing different tool configurations
-   Dynamic tool filtering based on runtime state

## Experimental: Codex tool

The `codex_tool` wraps the Codex CLI so an agent can run workspace-scoped tasks (shell, file edits, MCP tools) during a tool call. This surface is experimental and may change.

Use it when you want the main agent to delegate a bounded workspace task to Codex without leaving the current run. By default, the tool name is `codex`. If you set a custom name, it must be `codex` or start with `codex_`. When an agent includes multiple Codex tools, each must use a unique name.

```python
from agents import Agent
from agents.extensions.experimental.codex import ThreadOptions, TurnOptions, codex_tool

agent = Agent(
    name="Codex Agent",
    instructions="Use the codex tool to inspect the workspace and answer the question.",
    tools=[
        codex_tool(
            sandbox_mode="workspace-write",
            working_directory="/path/to/repo",
            default_thread_options=ThreadOptions(
                model="gpt-5.5",
                model_reasoning_effort="low",
                network_access_enabled=True,
                web_search_mode="disabled",
                approval_policy="never",
            ),
            default_turn_options=TurnOptions(
                idle_timeout_seconds=60,
            ),
            persist_session=True,
        )
    ],
)
```

Start with these option groups:

-   Execution surface: `sandbox_mode` and `working_directory` define where Codex can operate. Pair them together, and set `skip_git_repo_check=True` when the working directory is not inside a Git repository.
-   Thread defaults: `default_thread_options=ThreadOptions(...)` configures the model, reasoning effort, approval policy, additional directories, network access, and web search mode. Prefer `web_search_mode` over the legacy `web_search_enabled`.
-   Turn defaults: `default_turn_options=TurnOptions(...)` configures per-turn behavior such as `idle_timeout_seconds` and the optional cancellation `signal`.
-   Tool I/O: tool calls must include at least one `inputs` item with `{ "type": "text", "text": ... }` or `{ "type": "local_image", "path": ... }`. `output_schema` lets you require structured Codex responses.

Thread reuse and persistence are separate controls:

-   `persist_session=True` reuses one Codex thread for repeated calls to the same tool instance.
-   `use_run_context_thread_id=True` stores and reuses the thread ID in run context across runs that share the same mutable context object.
-   Thread ID precedence is: per-call `thread_id`, then run-context thread ID (if enabled), then the configured `thread_id` option.
-   The default run-context key is `codex_thread_id` for `name="codex"` and `codex_thread_id_<suffix>` for `name="codex_<suffix>"`. Override it with `run_context_thread_id_key`.

Runtime configuration:

-   Auth: set `CODEX_API_KEY` (preferred) or `OPENAI_API_KEY`, or pass `codex_options={"api_key": "..."}`.
-   Runtime: `codex_options.base_url` overrides the CLI base URL.
-   Binary resolution: set `codex_options.codex_path_override` (or `CODEX_PATH`) to pin the CLI path. Otherwise the SDK resolves `codex` from `PATH`, then falls back to the bundled vendor binary.
-   Environment: `codex_options.env` fully controls the subprocess environment. When it is provided, the subprocess does not inherit `os.environ`.
-   Stream limits: `codex_options.codex_subprocess_stream_limit_bytes` (or `OPENAI_AGENTS_CODEX_SUBPROCESS_STREAM_LIMIT_BYTES`) controls stdout/stderr reader limits. Valid range is `65536` to `67108864`; default is `8388608`.
-   Streaming: `on_stream` receives thread/turn lifecycle events and item events (`reasoning`, `command_execution`, `mcp_tool_call`, `file_change`, `web_search`, `todo_list`, and `error` item updates).
-   Outputs: results include `response`, `usage`, and `thread_id`; usage is added to `RunContextWrapper.usage`.

Reference:

-   [Codex tool API reference](ref/extensions/experimental/codex/codex_tool.md)
-   [ThreadOptions reference](ref/extensions/experimental/codex/thread_options.md)
-   [TurnOptions reference](ref/extensions/experimental/codex/turn_options.md)
-   See `examples/tools/codex.py` and `examples/tools/codex_same_thread.py` for complete runnable samples.

---

## Handoffs

# Handoffs

Handoffs allow an agent to delegate tasks to another agent. This is particularly useful in scenarios where different agents specialize in distinct areas. For example, a customer support app might have agents that each specifically handle tasks like order status, refunds, FAQs, etc.

Handoffs are represented as tools to the LLM. So if there's a handoff to an agent named `Refund Agent`, the tool would be named `transfer_to_refund_agent`.

## Creating a handoff

All agents have a [`handoffs`][agents.agent.Agent.handoffs] param, which can either take an `Agent` directly, or a `Handoff` object that customizes the Handoff.

If you pass plain `Agent` instances, their [`handoff_description`][agents.agent.Agent.handoff_description] (when set) is appended to the default tool description. Use it to hint when the model should pick that handoff without writing a full `handoff()` object.

You can create a handoff using the [`handoff()`][agents.handoffs.handoff] function provided by the Agents SDK. This function allows you to specify the agent to hand off to, along with optional overrides and input filters.

### Basic usage

Here's how you can create a simple handoff:

```python
from agents import Agent, handoff

billing_agent = Agent(name="Billing agent")
refund_agent = Agent(name="Refund agent")

# (1)!
triage_agent = Agent(name="Triage agent", handoffs=[billing_agent, handoff(refund_agent)])
```

1. You can use the agent directly (as in `billing_agent`), or you can use the `handoff()` function.

### Customizing handoffs via the `handoff()` function

The [`handoff()`][agents.handoffs.handoff] function lets you customize things.

-   `agent`: This is the agent to which things will be handed off.
-   `tool_name_override`: By default, the `Handoff.default_tool_name()` function is used, which resolves to `transfer_to_<agent_name>`. You can override this.
-   `tool_description_override`: Override the default tool description from `Handoff.default_tool_description()`
-   `on_handoff`: A callback function executed when the handoff is invoked. This is useful for things like kicking off some data fetching as soon as you know a handoff is being invoked. This function receives the agent context, and can optionally also receive LLM generated input. The input data is controlled by the `input_type` param.
-   `input_type`: The schema for the handoff tool-call arguments. When set, the parsed payload is passed to `on_handoff`.
-   `input_filter`: This lets you filter the input received by the next agent. See below for more.
-   `is_enabled`: Whether the handoff is enabled. This can be a boolean or a function that returns a boolean, allowing you to dynamically enable or disable the handoff at runtime.
-   `nest_handoff_history`: Optional per-handoff override for the RunConfig-level `nest_handoff_history` setting. If `None`, the value defined in the active run configuration is used instead.

The [`handoff()`][agents.handoffs.handoff] helper always transfers control to the specific `agent` you passed in. If you have multiple possible destinations, register one handoff per destination and let the model choose among them. Use a custom [`Handoff`][agents.handoffs.Handoff] only when your own handoff code must decide which agent to return at invocation time.

```python
from agents import Agent, handoff, RunContextWrapper

def on_handoff(ctx: RunContextWrapper[None]):
    print("Handoff called")

agent = Agent(name="My agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    tool_name_override="custom_handoff_tool",
    tool_description_override="Custom description",
)
```

## Handoff inputs

In certain situations, you want the LLM to provide some data when it calls a handoff. For example, imagine a handoff to an "Escalation agent". You might want the model to provide a reason so you can log it.

```python
from pydantic import BaseModel

from agents import Agent, handoff, RunContextWrapper

class EscalationData(BaseModel):
    reason: str

async def on_handoff(ctx: RunContextWrapper[None], input_data: EscalationData):
    print(f"Escalation agent called with reason: {input_data.reason}")

agent = Agent(name="Escalation agent")

handoff_obj = handoff(
    agent=agent,
    on_handoff=on_handoff,
    input_type=EscalationData,
)
```

`input_type` describes the arguments for the handoff tool call itself. The SDK exposes that schema to the model as the handoff tool's `parameters`, validates the returned JSON locally, and passes the parsed value to `on_handoff`.

It does not replace the next agent's main input, and it does not choose a different destination. The [`handoff()`][agents.handoffs.handoff] helper still transfers to the specific agent you wrapped, and the receiving agent still sees the conversation history unless you change it with an [`input_filter`][agents.handoffs.Handoff.input_filter] or nested handoff history settings.

`input_type` is also separate from [`RunContextWrapper.context`][agents.run_context.RunContextWrapper.context]. Use `input_type` for metadata the model decides at handoff time, not for application state or dependencies you already have locally.

### When to use `input_type`

Use `input_type` when the handoff needs a small piece of model-generated metadata such as `reason`, `language`, `priority`, or `summary`. For example, a triage agent can hand off to a refund agent with `{ "reason": "duplicate_charge", "priority": "high" }`, and `on_handoff` can log or persist that metadata before the refund agent takes over.

Choose a different mechanism when the goal is different:

-   Put existing application state and dependencies in [`RunContextWrapper.context`][agents.run_context.RunContextWrapper.context]. See the [context guide](context.md).
-   Use [`input_filter`][agents.handoffs.Handoff.input_filter], [`RunConfig.nest_handoff_history`][agents.run.RunConfig.nest_handoff_history], or [`RunConfig.handoff_history_mapper`][agents.run.RunConfig.handoff_history_mapper] if you want to change what history the receiving agent sees.
-   Register one handoff per destination if there are multiple possible specialists. `input_type` can add metadata to the chosen handoff, but it does not dispatch between destinations.
-   If you want structured input for a nested specialist without transferring the conversation, prefer [`Agent.as_tool(parameters=...)`][agents.agent.Agent.as_tool]. See [tools](tools.md#structured-input-for-tool-agents).

## Input filters

When a handoff occurs, it's as though the new agent takes over the conversation, and gets to see the entire previous conversation history. If you want to change this, you can set an [`input_filter`][agents.handoffs.Handoff.input_filter]. An input filter is a function that receives the existing input via a [`HandoffInputData`][agents.handoffs.HandoffInputData], and must return a new `HandoffInputData`.

[`HandoffInputData`][agents.handoffs.HandoffInputData] includes:

-   `input_history`: the input history before `Runner.run(...)` started.
-   `pre_handoff_items`: items generated before the agent turn where the handoff was invoked.
-   `new_items`: items generated during the current turn, including the handoff call and handoff output items.
-   `input_items`: optional items to forward to the next agent instead of `new_items`, allowing you to filter model input while keeping `new_items` intact for session history.
-   `run_context`: the active [`RunContextWrapper`][agents.run_context.RunContextWrapper] at the time the handoff was invoked.

Nested handoff history is available as an opt-in beta and is disabled by default while we stabilize it. When you enable [`RunConfig.nest_handoff_history`][agents.run.RunConfig.nest_handoff_history], the runner compacts summarizable history into ordered assistant summary segments while preserving lossless message items in their original positions. Each generated summary segment uses the `<CONVERSATION HISTORY>` wrapper, and later handoffs flatten earlier generated segments before rebuilding the ordered transcript. Sessions, `RunState`, and `RunResult.to_input_list()` track exact message occurrences moved into this SDK-default history so those occurrences are not appended twice; separate identical messages are still preserved. You can provide your own mapping function via [`RunConfig.handoff_history_mapper`][agents.run.RunConfig.handoff_history_mapper] to return the exact list of input items for the next agent instead of using the built-in segmentation. The opt-in applies only when neither the handoff's `input_filter` nor the active run's `RunConfig.handoff_input_filter` is set, so existing code that already customizes the payload (including the examples in this repository) keeps its current behavior without changes. You can override the nesting behaviour for a single handoff by passing `nest_handoff_history=True` or `False` to [`handoff(...)`][agents.handoffs.handoff], which sets [`Handoff.nest_handoff_history`][agents.handoffs.Handoff.nest_handoff_history]. If you just need to change the wrapper text for generated summary segments, call [`set_conversation_history_wrappers`][agents.handoffs.set_conversation_history_wrappers] before running your agents. Call [`reset_conversation_history_wrappers`][agents.handoffs.reset_conversation_history_wrappers] before a later run when you need to restore the default wrappers.

If both the handoff and the active [`RunConfig.handoff_input_filter`][agents.run.RunConfig.handoff_input_filter] define a filter, the per-handoff [`input_filter`][agents.handoffs.Handoff.input_filter] takes precedence for that specific handoff.

!!! note

    Handoffs stay within a single run. Input guardrails still apply only to the first agent in the chain, and output guardrails only to the agent that produces the final output. Use tool guardrails when you need checks around each custom function-tool call inside the workflow.

There are some common patterns (for example removing all tool calls from the history), which are implemented for you in [`agents.extensions.handoff_filters`][]

```python
from agents import Agent, handoff
from agents.extensions import handoff_filters

agent = Agent(name="FAQ agent")

handoff_obj = handoff(
    agent=agent,
    input_filter=handoff_filters.remove_all_tools, # (1)!
)
```

1. This will automatically remove all tool-related items from the history when `FAQ agent` is called.

## Recommended prompts

To make sure that LLMs understand handoffs properly, we recommend including information about handoffs in your agents. We have a suggested prefix in [`agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX`][], or you can call [`agents.extensions.handoff_prompt.prompt_with_handoff_instructions`][] to automatically add recommended data to your prompts.

```python
from agents import Agent
from agents.extensions.handoff_prompt import RECOMMENDED_PROMPT_PREFIX

billing_agent = Agent(
    name="Billing agent",
    instructions=f"""{RECOMMENDED_PROMPT_PREFIX}
    <Fill in the rest of your prompt here>.""",
)
```

---

## Orchestrating multiple agents

# Agent orchestration

Orchestration refers to the flow of agents in your app. Which agents run, in what order, and how is the next step decided? There are two main ways to orchestrate agents:

1. Allowing the LLM to make decisions: this uses the intelligence of an LLM to plan, reason, and decide on what steps to take based on that.
2. Orchestrating via code: determining the flow of agents via your code.

You can mix and match these patterns. Each has their own tradeoffs, described below.

## Orchestrating via LLM

An agent is an LLM equipped with instructions, tools and handoffs. This means that given an open-ended task, the LLM can autonomously plan how it will tackle the task, using tools to take actions and acquire data, and using handoffs to delegate tasks to sub-agents. For example, a research agent could be equipped with capabilities like:

-   Web search to find information online
-   File search and retrieval to search through proprietary data and connected data sources
-   Computer use to take actions on a computer
-   Code execution to do data analysis
-   Handoffs to specialized agents that are great at planning, report writing and more.

### Core SDK patterns

In the Python SDK, two orchestration patterns come up most often:

| Pattern | How it works | Best when |
| --- | --- | --- |
| Agents as tools | A manager agent keeps control of the conversation and calls specialist agents through `Agent.as_tool()`. | You want one agent to own the final answer, combine outputs from multiple specialists, or enforce shared SDK guardrails in one place. |
| Handoffs | A triage agent routes the conversation to a specialist, and that specialist becomes the active agent for the rest of the turn. | You want the specialist to respond directly, keep prompts focused, or have the handoff switch the active instructions without requiring the manager to narrate the result. |

Use **agents as tools** when a specialist should help with a bounded subtask but should not take over the user-facing conversation. Use **handoffs** when routing itself is part of the workflow and you want the chosen specialist to own the remainder of the current turn.

You can also combine the two. A triage agent might hand off to a specialist, and that specialist can still call other agents as tools for narrow subtasks.

This pattern is great when the task is open-ended and you want to rely on the intelligence of an LLM. The most important tactics here are:

1. Invest in good prompts. Make it clear what tools are available, how to use them, and what constraints the agent must follow.
2. Monitor your app and iterate on it. See where things go wrong, and iterate on your prompts.
3. Allow the agent to introspect and improve. For example, run it in a loop, and let it critique itself; or, provide error messages and let it improve.
4. Have specialized agents that excel in one task, rather than having a general purpose agent that is expected to be good at anything.
5. Invest in [evals](https://platform.openai.com/docs/guides/evals). This lets you train your agents to improve and get better at tasks.

If you want the core SDK primitives behind this style of orchestration, start with [tools](tools.md), [handoffs](handoffs.md), and [running agents](running_agents.md).

## Orchestrating via code

While orchestrating via LLM is powerful, orchestrating via code makes tasks more deterministic and predictable, in terms of speed, cost and performance. Common patterns here are:

-   Using [structured outputs](https://platform.openai.com/docs/guides/structured-outputs) to generate well formed data that you can inspect with your code. For example, you might ask an agent to classify the task into a few categories, and then pick the next agent based on the category.
-   Chaining multiple agents by transforming the output of one into the input of the next. You can decompose a task like writing a blog post into a series of steps - do research, write an outline, write the blog post, critique it, and then improve it.
-   In each iteration of a `while` loop, run the task agent to produce an output, then run an evaluator agent to assess that output and provide feedback; stop when the evaluator says the output passes the required criteria.
-   Running multiple agents in parallel, e.g. via Python primitives like `asyncio.gather`. This is useful for speed when you have multiple tasks that don't depend on each other.

We have a number of examples in [`examples/agent_patterns`](https://github.com/openai/openai-agents-python/tree/main/examples/agent_patterns).

## Related guides

-   [Agents](agents.md) for composition patterns and agent configuration.
-   [Tools](tools.md#agents-as-tools) for `Agent.as_tool()` and manager-style orchestration.
-   [Handoffs](handoffs.md) for delegation between specialist agents.
-   [Running agents](running_agents.md) for per-run orchestration controls and conversation state.
-   [Quickstart](quickstart.md) for a minimal end-to-end handoff example.

---

## Guardrails

# Guardrails

Guardrails enable you to do checks and validations of user input and agent output. For example, imagine you have an agent that uses a very smart (and hence slow/expensive) model to help with customer requests. You wouldn't want malicious users to ask the model to help them with their math homework. So, you can run a guardrail with a fast/cheap model. If the guardrail detects malicious usage, it can immediately raise an error, saving time and money. Blocking execution guarantees that the expensive model does not start; with parallel execution, the expensive model may already have started before the guardrail completes. See "Execution modes" below for details.

There are two kinds of guardrails:

1. Input guardrails run on the initial user input
2. Output guardrails run on the final agent output

## Workflow boundaries

Guardrails are attached to agents and tools, but they do not all run at the same points in a workflow:

-   **Input guardrails** run only for the first agent in the chain.
-   **Output guardrails** run only for the agent that produces the final output.
-   **Tool guardrails** run on every custom function-tool invocation, with input guardrails before execution and output guardrails after execution.

If you need checks before and/or after each custom function-tool call in a workflow that includes managers, handoffs, or delegated specialists, use tool guardrails instead of relying only on agent-level input/output guardrails.

## Input guardrails

Input guardrails run in 3 steps:

1. First, the guardrail receives the same input passed to the agent.
2. Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput], which is then wrapped in an [`InputGuardrailResult`][agents.guardrail.InputGuardrailResult]
3. Finally, we check if [`.tripwire_triggered`][agents.guardrail.GuardrailFunctionOutput.tripwire_triggered] is true. If true, an [`InputGuardrailTripwireTriggered`][agents.exceptions.InputGuardrailTripwireTriggered] exception is raised, so you can appropriately respond to the user or handle the exception.

!!! Note

    Input guardrails are intended to run on user input, so an agent's guardrails only run if the agent is the *first* agent. You might wonder, why is the `guardrails` property on the agent instead of passed to `Runner.run`? It's because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

### Execution modes

Input guardrails support two execution modes:

- **Parallel execution** (default, `run_in_parallel=True`): The guardrail runs concurrently with the agent's execution. This provides the best latency since both start at the same time. However, if the guardrail's tripwire is triggered, the agent may have already consumed tokens and executed tools before being cancelled.

- **Blocking execution** (`run_in_parallel=False`): The guardrail runs and completes *before* the agent starts. If the guardrail tripwire is triggered, the agent never executes, preventing token consumption and tool execution. This is ideal for cost optimization and when you want to avoid potential side effects from tool calls.

## Output guardrails

Output guardrails run in 3 steps:

1. First, the guardrail receives the output produced by the agent.
2. Next, the guardrail function runs to produce a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput], which is then wrapped in an [`OutputGuardrailResult`][agents.guardrail.OutputGuardrailResult]
3. Finally, we check if [`.tripwire_triggered`][agents.guardrail.GuardrailFunctionOutput.tripwire_triggered] is true. If true, an [`OutputGuardrailTripwireTriggered`][agents.exceptions.OutputGuardrailTripwireTriggered] exception is raised, so you can appropriately respond to the user or handle the exception.

!!! Note

    Output guardrails are intended to run on the final agent output, so an agent's guardrails only run if the agent is the *last* agent. Similar to the input guardrails, we do this because guardrails tend to be related to the actual Agent - you'd run different guardrails for different agents, so colocating the code is useful for readability.

    Output guardrails always run after the agent completes, so they don't support the `run_in_parallel` parameter.

An output tripwire and an exception raised by the guardrail function have different session behavior. A tripwire rejects the candidate final output. When a tripwire fires, the runner asks the configured session to persist already-completed tool call and tool output items, together with any reasoning context required to replay those calls, while excluding the rejected candidate final output. The runner applies this tripwire rule to both streaming and non-streaming runs. When the guardrail function raises an exception instead of returning a tripwire result, the runner treats the verdict as unknown and asks the configured session to persist the completed final-turn items before surfacing the guardrail exception. If that session write also fails, the session write error takes precedence. Streaming runs use the same persistence ordering as non-streaming runs and raise the terminal exception from `stream_events()`. An immediate [`RunResultStreaming.cancel()`][agents.result.RunResultStreaming.cancel] call while the output guardrail is running cancels the in-flight guardrail and does not start a final-turn session write.

## Tool guardrails

Tool guardrails wrap **`FunctionTool` instances** and let you validate or block calls to those tools before and after execution. They are configured on the tool itself and run every time that tool is invoked.

- Input tool guardrails run before the tool executes and can skip the call, replace the output with a message, or raise a tripwire.
- Output tool guardrails run after the tool executes and can replace the output or raise a tripwire.
- If a function tool requires approval, input tool guardrails normally run after approval and immediately before execution. Set [`RunConfig.tool_execution`][agents.run.RunConfig.tool_execution] to [`ToolExecutionConfig(pre_approval_tool_input_guardrails=True)`][agents.run.ToolExecutionConfig] when you want those input checks to run before the pending approval interruption is emitted. Calls that pass this pre-approval check are still checked again after approval before the tool executes.
- Tool guardrails apply only to function tools created with [`function_tool`][agents.tool.function_tool]. Handoffs run through the SDK's handoff pipeline rather than the normal function-tool pipeline, so tool guardrails do not apply to the handoff call itself. Hosted tools (`WebSearchTool`, `FileSearchTool`, `HostedMCPTool`, `CodeInterpreterTool`, `ImageGenerationTool`) and built-in execution tools (`ComputerTool`, `ShellTool`, `ApplyPatchTool`, `LocalShellTool`) also do not use this guardrail pipeline, and [`Agent.as_tool()`][agents.agent.Agent.as_tool] does not currently expose tool-guardrail options directly.

See the code snippet below for details.

## Tripwires

If an agent input or output fails a guardrail, the guardrail can signal this with a tripwire. The runner immediately raises an `InputGuardrailTripwireTriggered` or `OutputGuardrailTripwireTriggered` exception and halts agent execution. Tool guardrails use the corresponding `ToolInputGuardrailTripwireTriggered` and `ToolOutputGuardrailTripwireTriggered` exceptions.

For agent-level tripwires, the exception's `guardrail_result` identifies the guardrail that triggered the tripwire. For an input tripwire raised by the runner, `exception.run_data.input_guardrail_results` contains every input guardrail result completed before the run stopped, including the result that triggered the tripwire. Output tripwires provide the equivalent accumulated results through `exception.run_data.output_guardrail_results`.

Tool tripwire exceptions instead expose the triggering `guardrail` and `output` directly. Their `run_data.tool_input_guardrail_results` and `run_data.tool_output_guardrail_results` lists preserve results accumulated from completed turns before the failure; the triggering result is available through the exception's `output`. Other runner-managed failures, such as `MaxTurnsExceeded`, also preserve completed tool guardrail results in these lists. After `stream_events()` raises an exception, the streamed result exposes the same accumulated agent and tool guardrail result lists. `run_data` can be `None` when an exception is raised outside a runner-managed execution path.

## Implementing a guardrail

You need to provide a function that receives input, and returns a [`GuardrailFunctionOutput`][agents.guardrail.GuardrailFunctionOutput]. In this example, we'll do this by running an Agent under the hood.

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    InputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
    TResponseInputItem,
)
from agents.decorators import input_guardrail

class MathHomeworkOutput(BaseModel):
    is_math_homework: bool
    reasoning: str

guardrail_agent = Agent( # (1)!
    name="Guardrail check",
    instructions="Check if the user is asking you to do their math homework.",
    output_type=MathHomeworkOutput,
)


@input_guardrail
async def math_guardrail( # (2)!
    ctx: RunContextWrapper[None], agent: Agent, input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, input, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output, # (3)!
        tripwire_triggered=result.final_output.is_math_homework,
    )


agent = Agent(  # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    input_guardrails=[math_guardrail],
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except InputGuardrailTripwireTriggered:
        print("Math homework guardrail tripped")
```

1. We'll use this agent in our guardrail function.
2. This is the guardrail function that receives the agent's input/context, and returns the result.
3. We can include extra information in the guardrail result.
4. This is the actual agent that defines the workflow.

Output guardrails are similar.

```python
from pydantic import BaseModel
from agents import (
    Agent,
    GuardrailFunctionOutput,
    OutputGuardrailTripwireTriggered,
    RunContextWrapper,
    Runner,
)
from agents.decorators import output_guardrail
class MessageOutput(BaseModel): # (1)!
    response: str

class MathOutput(BaseModel): # (2)!
    reasoning: str
    is_math: bool

guardrail_agent = Agent(
    name="Guardrail check",
    instructions="Check if the output includes any math.",
    output_type=MathOutput,
)

@output_guardrail
async def math_guardrail(  # (3)!
    ctx: RunContextWrapper, agent: Agent, output: MessageOutput
) -> GuardrailFunctionOutput:
    result = await Runner.run(guardrail_agent, output.response, context=ctx.context)

    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=result.final_output.is_math,
    )

agent = Agent( # (4)!
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    output_guardrails=[math_guardrail],
    output_type=MessageOutput,
)

async def main():
    # This should trip the guardrail
    try:
        await Runner.run(agent, "Hello, can you help me solve for x: 2x + 3 = 11?")
        print("Guardrail didn't trip - this is unexpected")

    except OutputGuardrailTripwireTriggered:
        print("Math output guardrail tripped")
```

1. This is the actual agent's output type.
2. This is the guardrail's output type.
3. This is the guardrail function that receives the agent's output, and returns the result.
4. This is the actual agent that defines the workflow.

Lastly, here are examples of tool guardrails.

```python
import json
from agents import (
    Agent,
    Runner,
    ToolGuardrailFunctionOutput,
)
from agents.decorators import tool, tool_input_guardrail, tool_output_guardrail

@tool_input_guardrail
def block_secrets(data):
    args = json.loads(data.context.tool_arguments or "{}")
    if "sk-" in json.dumps(args):
        return ToolGuardrailFunctionOutput.reject_content(
            "Remove secrets before calling this tool."
        )
    return ToolGuardrailFunctionOutput.allow()


@tool_output_guardrail
def redact_output(data):
    text = str(data.output or "")
    if "sk-" in text:
        return ToolGuardrailFunctionOutput.reject_content("Output contained sensitive data.")
    return ToolGuardrailFunctionOutput.allow()


@tool(
    tool_input_guardrails=[block_secrets],
    tool_output_guardrails=[redact_output],
)
def classify_text(text: str) -> str:
    """Classify text for internal routing."""
    return f"length:{len(text)}"


agent = Agent(name="Classifier", tools=[classify_text])
result = Runner.run_sync(agent, "hello world")
print(result.final_output)
```

---

## Context management

# Context management

Context is an overloaded term. There are two main classes of context you might care about:

1. Context available locally to your code: this is data and dependencies you might need when tool functions run, during callbacks like `on_handoff`, in lifecycle hooks, etc.
2. Context available to LLMs: this is data the LLM sees when generating a response.

## Local context

This is represented via the [`RunContextWrapper`][agents.run_context.RunContextWrapper] class and the [`context`][agents.run_context.RunContextWrapper.context] property within it. The way this works is:

1. You create any Python object you want. A common pattern is to use a dataclass or a Pydantic object.
2. You pass that object to the various run methods (e.g. `Runner.run(..., context=whatever)`).
3. All your tool calls, lifecycle hooks etc will be passed a wrapper object, `RunContextWrapper[T]`, where `T` represents the type of your context object; the object itself is available via `wrapper.context`.

For some runtime-specific callbacks, the SDK may pass a more specialized subclass of `RunContextWrapper[T]`. For example, lifecycle hooks for `FunctionTool` instances typically receive `ToolContext`, which also exposes tool-call metadata like `tool_call_id`, `tool_name`, and `tool_arguments`.

The **most important** thing to be aware of: every agent, tool function, lifecycle etc for a given agent run must use the same _type_ of context.

You can use the context for things like:

-   Contextual data for your run (e.g. things like a username/uid or other information about the user)
-   Dependencies (e.g. logger objects, data fetchers, etc)
-   Helper functions

!!! danger "Note"

    The context object is **not** sent to the LLM. It is purely a local object that you can read from, write to and call methods on it.

Within a single run, derived wrappers share the same underlying app context, approval state, and usage tracking. Nested [`Agent.as_tool()`][agents.agent.Agent.as_tool] runs may attach a different `tool_input`, but they do not get an isolated copy of your app state by default.

### What `RunContextWrapper` exposes

[`RunContextWrapper`][agents.run_context.RunContextWrapper] is a wrapper around your app-defined context object. In practice you will most often use:

-   [`wrapper.context`][agents.run_context.RunContextWrapper.context] for your own mutable app state and dependencies.
-   [`wrapper.usage`][agents.run_context.RunContextWrapper.usage] for aggregated request and token usage across the current run.
-   [`wrapper.tool_input`][agents.run_context.RunContextWrapper.tool_input] for structured input when the current run is executing inside [`Agent.as_tool()`][agents.agent.Agent.as_tool].
-   [`wrapper.approve_tool(...)`][agents.run_context.RunContextWrapper.approve_tool] / [`wrapper.reject_tool(...)`][agents.run_context.RunContextWrapper.reject_tool] when you need to update approval state programmatically.

Only `wrapper.context` is your app-defined object. The other fields are runtime metadata managed by the SDK.

If you later serialize a [`RunState`][agents.run_state.RunState] for human-in-the-loop or durable job workflows, that runtime metadata is saved with the state. Avoid putting secrets in [`RunContextWrapper.context`][agents.run_context.RunContextWrapper.context] if you intend to persist or transmit serialized state.

Conversation state is a separate concern. Use `result.to_input_list()`, `session`, `conversation_id`, or `previous_response_id` depending on how you want to carry turns forward. See [results](results.md), [running agents](running_agents.md), and [sessions](sessions/index.md) for that decision.

```python
import asyncio
from dataclasses import dataclass

from agents import Agent, RunContextWrapper, Runner
from agents.decorators import tool

@dataclass
class UserInfo:  # (1)!
    name: str
    uid: int

@tool
async def fetch_user_age(wrapper: RunContextWrapper[UserInfo]) -> str:  # (2)!
    """Fetch the age of the user. Call this function to get user's age information."""
    return f"The user {wrapper.context.name} is 47 years old"

async def main():
    user_info = UserInfo(name="John", uid=123)

    agent = Agent[UserInfo](  # (3)!
        name="Assistant",
        tools=[fetch_user_age],
    )

    result = await Runner.run(  # (4)!
        starting_agent=agent,
        input="What is the age of the user?",
        context=user_info,
    )

    print(result.final_output)  # (5)!
    # The user John is 47 years old.

if __name__ == "__main__":
    asyncio.run(main())
```

1. This is the context object. We've used a dataclass here, but you can use any type.
2. This is a tool. You can see it takes a `RunContextWrapper[UserInfo]`. The tool implementation reads from the context.
3. We mark the agent with the generic `UserInfo`, so that the typechecker can catch errors (for example, if we tried to pass a tool that took a different context type).
4. The context is passed to the `run` function.
5. The agent correctly calls the tool and gets the age.

---

### Advanced: `ToolContext`

In some cases, you might want to access extra metadata about the tool being executed — such as its name, call ID, or raw argument string.  
For this, you can use the [`ToolContext`][agents.tool_context.ToolContext] class, which extends `RunContextWrapper`.

```python
from typing import Annotated
from pydantic import BaseModel, Field
from agents import Agent
from agents.decorators import tool
from agents.tool_context import ToolContext

class WeatherContext(BaseModel):
    user_id: str

class Weather(BaseModel):
    city: str = Field(description="The city name")
    temperature_range: str = Field(description="The temperature range in Celsius")
    conditions: str = Field(description="The weather conditions")

@tool
def get_weather(ctx: ToolContext[WeatherContext], city: Annotated[str, "The city to get the weather for"]) -> Weather:
    print(f"[debug] Tool context: (name: {ctx.tool_name}, call_id: {ctx.tool_call_id}, args: {ctx.tool_arguments})")
    return Weather(city=city, temperature_range="14-20C", conditions="Sunny with wind.")

agent = Agent(
    name="Weather Agent",
    instructions="You are a helpful agent that can tell the weather of a given city.",
    tools=[get_weather],
)
```

`ToolContext` provides the same `.context` property as `RunContextWrapper`,  
plus additional fields specific to the current tool call:

- `tool_name` – the name of the tool being invoked  
- `tool_call_id` – a unique identifier for this tool call  
- `tool_arguments` – the raw argument string passed to the tool  
- `tool_namespace` – the Responses namespace for the tool call, when the tool was loaded through `tool_namespace()` or another namespaced surface  
- `qualified_tool_name` – the tool name qualified with the namespace when one is available  

Use `ToolContext` when you need tool-level metadata during execution.  
For general context sharing between agents and tools, `RunContextWrapper` remains sufficient. Because `ToolContext` extends `RunContextWrapper`, it can also expose `.tool_input` when a nested `Agent.as_tool()` run supplied structured input.

---

## Agent/LLM context

When an LLM is called, the **only** data it can see is from the conversation history. This means that if you want to make some new data available to the LLM, you must do it in a way that makes it available in that history. There are a few ways to do this:

1. You can add it to the Agent `instructions`. This is also known as a "system prompt" or "developer message". System prompts can be static strings, or they can be dynamic functions that receive the context and output a string. This is a common tactic for information that is always useful (for example, the user's name or the current date).
2. Add it to the `input` when calling the `Runner.run` functions. This is similar to the `instructions` tactic, but allows you to have messages that are lower in the [chain of command](https://cdn.openai.com/spec/model-spec-2024-05-08.html#follow-the-chain-of-command).
3. Expose it through `FunctionTool` instances. This is useful for _on-demand_ context - the LLM decides when it needs some data, and can call the tool to fetch that data.
4. Use retrieval or web search. These are special tools that are able to fetch relevant data from files or databases (retrieval), or from the web (web search). This is useful for "grounding" the response in relevant contextual data.
