# Build an agent

Write an agent that prefixes each text prompt with `You said:`. First complete the [quickstart](quickstart.md) so you have a repository checkout and a working client.

## Implement the handlers

Save this as `my_agent.py` in the repository root:

```python
import asyncio
from typing import Any
from uuid import uuid4

from acp import (
    PROTOCOL_VERSION,
    Agent,
    Client,
    InitializeResponse,
    NewSessionResponse,
    PromptResponse,
    run_agent,
)
from acp.schema import AgentMessageChunk, TextContentBlock


class MyAgent(Agent):
    _client: Client

    def on_connect(self, conn: Client) -> None:
        self._client = conn

    async def initialize(
        self, protocol_version: int, **kwargs: Any
    ) -> InitializeResponse:
        return InitializeResponse(protocol_version=PROTOCOL_VERSION)

    async def new_session(self, cwd: str, **kwargs: Any) -> NewSessionResponse:
        return NewSessionResponse(session_id=uuid4().hex)

    async def prompt(
        self, session_id: str, prompt: list[Any], **kwargs: Any
    ) -> PromptResponse:
        for block in prompt:
            if isinstance(block, TextContentBlock):
                await self._client.session_update(
                    session_id=session_id,
                    update=AgentMessageChunk(
                        content=TextContentBlock(text=f"You said: {block.text}")
                    ),
                )
        return PromptResponse(stop_reason="end_turn")


if __name__ == "__main__":
    asyncio.run(run_agent(MyAgent()))
```

`Agent` describes the handler interface. Subclassing it is optional; the SDK can use an object with the matching methods. This tutorial uses `Any` for prompt content because it handles only text and ignores other content types. The [API reference](../reference/api.md#agent-handlers) shows the complete content union.

`on_connect` receives the connection used to send callbacks to the client. `initialize` reports the supported protocol version. `new_session` returns a session ID. During `prompt`, the agent sends text through `session_update`, then returns the turn's stop reason.

## Run it through the client

```bash
uv run --frozen python examples/client.py python my_agent.py
```

Type `Hello ACP`:

```text
> Hello ACP
| Agent: You said: Hello ACP
>
```

The client owns the agent process. Keep agent logs on stderr because stdout carries ACP frames.

## Add your own behavior

Replace the text generation inside `prompt` with your agent's work. Send multiple text chunks as results become available. Implement the agent capabilities you advertise, such as supported prompt content and session persistence. Check the client's advertised filesystem and terminal capabilities before calling those services.

This example creates session IDs without retaining conversation state. Add session storage before advertising support for loading previous sessions. Use the [permission guide](../how-to/permissions.md) when an operation needs user approval. The [integration patterns](../integrations/index.md) show how Bub and PydanticAI adapters map an existing runtime's events and history to these handlers.
