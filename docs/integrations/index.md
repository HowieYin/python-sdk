# Integration patterns

Existing SDK integrations show where the protocol boundary ends and application behavior begins. Use these as source references when adapting your runtime or building client callbacks. The executable starting points are the [agent/client quickstart](../tutorials/quickstart.md) and [agent tutorial](../tutorials/agent.md); the project references below have their own dependencies and are not a tested installation recipe for this checkout.

## Adapt an existing agent runtime: Bub

The [Bub ACP adapter](https://github.com/bubbuild/bub-contrib/tree/5257e58fef85c735001995a6da0186c1b66403dc/packages/bub-acp-server) uses this SDK to expose Bub's runtime. At the linked revision, its [package metadata](https://github.com/bubbuild/bub-contrib/blob/5257e58fef85c735001995a6da0186c1b66403dc/packages/bub-acp-server/pyproject.toml) pins SDK `1.0.0rc2` and requires Python 3.12 or newer.

| Integration responsibility | Implementation to read |
| --- | --- |
| Map initialization, sessions, prompts, and cancellation to the runtime | `BubACPAgent` in [agent.py](https://github.com/bubbuild/bub-contrib/blob/5257e58fef85c735001995a6da0186c1b66403dc/packages/bub-acp-server/src/bub_acp_server/agent.py) |
| Convert model text and tool activity into `AgentMessageChunk`, `ToolCallStart`, and `ToolCallProgress` notifications | `ACPStreamRouter` in the same file |
| Select client-backed file and terminal tools from advertised capabilities | `ACPClientToolRuntime` and `build_client_tools` in [client_tools.py](https://github.com/bubbuild/bub-contrib/blob/5257e58fef85c735001995a6da0186c1b66403dc/packages/bub-acp-server/src/bub_acp_server/client_tools.py) |
| Retain session metadata and replay application history for `session/load` | Session methods in `BubACPAgent` |

The adapter calls SDK `run_agent` to serve stdio. Bub retains responsibility for its execution, tapes, and configuration. The [end-to-end test](https://github.com/bubbuild/bub-contrib/blob/5257e58fef85c735001995a6da0186c1b66403dc/packages/bub-acp-server/tests/test_e2e.py) exercises the adapter's client-backed file and terminal operations with controlled fixtures. This separation is useful when your runtime already has its own event stream and history store.

## Adapt a framework agent: PydanticAI

PydanticAI's [experimental ACP adapter](https://pydantic.dev/docs/ai/harness/acp/) wraps a PydanticAI agent. [`PydanticAIACPAgent`](https://github.com/pydantic/pydantic-ai/blob/0a28fc079668fa03040f0a3024f3bf1f1b63e819/src/pydantic_ai_harness/pydantic_ai_harness/experimental/acp/_adapter.py) implements the SDK's `Agent` interface, maps run events to session updates, and uses `request_permission` for deferred tool approvals. Its [server entry point](https://github.com/pydantic/pydantic-ai/blob/0a28fc079668fa03040f0a3024f3bf1f1b63e819/src/pydantic_ai_harness/pydantic_ai_harness/experimental/acp/_server.py) passes that adapter to `acp.run_agent`.

Read this boundary when deciding which state and hooks belong in your framework adapter. Session history and approval behavior stay with the adapter; protocol framing and callback dispatch come from the SDK. The [stdio test agent](https://github.com/pydantic/pydantic-ai/blob/0a28fc079668fa03040f0a3024f3bf1f1b63e819/tests/harness/experimental/acp/_demo_agent.py) uses PydanticAI's `TestModel` to exercise the integration without a model service.

The adapter is experimental. At the linked revision, its [ACP extra](https://github.com/pydantic/pydantic-ai/blob/0a28fc079668fa03040f0a3024f3bf1f1b63e819/src/pydantic_ai_harness/pyproject.toml) requires SDK `>=0.11,<0.12`. That source is a version-specific reference, not evidence of compatibility with the SDK on current `main`.

## Connect protocol events to notebook output

[`agent-client-kernel`](https://github.com/wiki3-ai/agent-client-kernel/blob/fc10043bf78d3e16af0117417aaf783472a194ef/agent_client_kernel/kernel.py) is an SDK-based client. Its `ACPClientImpl` handles session updates and permission requests. The kernel starts an agent subprocess, passes its pipes to `connect_to_agent`, initializes the connection, and sends cell input as a prompt. Client callbacks translate agent updates into notebook output.

This illustrates the developer boundary between process ownership, protocol callbacks, and your application's UI. Its [metadata](https://github.com/wiki3-ai/agent-client-kernel/blob/fc10043bf78d3e16af0117417aaf783472a194ef/pyproject.toml) labels the project alpha and depends directly on the SDK Git repository. It is a source study, not a compatibility or maturity guarantee. Use the [complete SDK client example](../tutorials/client.md) as the runnable baseline.

## Add a source reference

Include the implementation and tests, the SDK dependency, and a specific release or commit. Describe the interface mapping a developer can learn from it. Verify any proposed runnable recipe separately against its exact dependencies before presenting it as an example.
