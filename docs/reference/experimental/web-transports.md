# Web Transport (Streamable HTTP & WebSocket)

> **Experimental.** The remote web transports are experimental and may change.
> They ship as an optional extra and are import-guarded.

The SDK can run ACP over two remote connectivity profiles in addition to stdio:

- **Streamable HTTP** — `POST` for client→server messages, long-lived `GET` SSE
  streams for server→client messages (one connection-scoped stream plus one per
  session), and `DELETE` to terminate. `initialize` returns `200 OK` with a JSON
  body; all other POSTs return `202 Accepted`. **Requires HTTP/2.**
- **WebSocket** — a `GET` upgrade on the same endpoint carrying full-duplex
  JSON-RPC text frames, one message per frame. Binary frames are ignored, as
  the transport RFD specifies; the server logs a warning on the first one.

Both reuse the existing JSON-RPC message format and ACP lifecycle
(`initialize` → session methods → close).

## Installation

```bash
pip install "agent-client-protocol[http]"
```

This pulls in `httpx2[http2]` (HTTP/2 + SSE consumption), `websockets`, and
`starlette` (the server application). The core SDK and stdio transport do not
require these optional dependencies.

## Client

Both transports produce a message-level `Transport` that plugs into the existing
`connect_to_agent`:

```python
from acp import connect_to_agent
from acp.http import create_http_stream
from acp.ws import create_websocket_stream

# Streamable HTTP
transport = create_http_stream("http://localhost:8000/acp")
conn = connect_to_agent(my_client, transport)

# ...or WebSocket
transport = await create_websocket_stream("ws://localhost:8000/acp")
conn = connect_to_agent(my_client, transport)

init = await conn.initialize(protocol_version=1)
session = await conn.new_session(cwd="/tmp", mcp_servers=[])
await conn.prompt(session_id=session.session_id, prompt=[...])
await conn.close()
await transport.close()
```

The client sends `initialize` first, reads the `Acp-Connection-Id` response
header, then opens the connection-scoped SSE stream. When a new `sessionId`
appears it opens that session-scoped stream too. A single SSE attempt is made per
stream; reconnect/retry is the caller's responsibility (v1 of the RFD).

### Loading an existing session

Both HTTP and WebSocket support `load_session()` when the agent advertises
`loadSession` and implements session persistence:

```python
init = await conn.initialize(protocol_version=1)
if init.agent_capabilities.load_session:
    await conn.load_session(session_id="saved-session-id", cwd="/workspace", mcp_servers=[])
    await conn.prompt(session_id="saved-session-id", prompt=[...])
```

For HTTP, history replay and the load response use the connection SSE stream.
The client correlates the response with the session ID from the load request,
then opens the session SSE stream for further prompts and agent callbacks. This
also works with an empty history: a load response need not contain `sessionId`.
Replay is consumed as it arrives, so histories larger than the SSE buffer do not
wait for a session stream to open. WebSocket uses its existing bidirectional
connection for both replay and subsequent messages.

`resume_session()` (when the agent advertises `sessionCapabilities.resume`)
follows the same HTTP routing without replaying history: the response uses the
connection SSE stream, and the client then opens the session SSE stream.

A failed load returns its JSON-RPC error on the connection stream and can be
retried. The server removes streams provisioned only for failed loads, while
preserving established sessions and overlapping loads. It does not change the
agent's load response or automatically enable the agent's `loadSession` capability.

## Server

The server uses Starlette for HTTP requests, responses, routing, streaming,
WebSocket handling, and application lifespan:

```python
from acp.http.asgi import create_asgi_app

# One agent instance is created per connection.
app = create_asgi_app(lambda conn: MyAgent())
```

`app` is a `starlette.applications.Starlette` instance handling
`POST`/`GET`/`DELETE` and WebSocket upgrades at `/acp` by default. Set the
keyword-only `path` argument to use a different endpoint for both transports:

```python
app = create_asgi_app(lambda conn: MyAgent(), path="/rpc")
```

Other paths do not serve ACP. Starlette supplies
`Request`, `JSONResponse`, `StreamingResponse`, and `WebSocket`; the SDK keeps
ACP connection and session routing.

### Mounting in another application

Mount the app at the desired prefix. The parent must enter the child lifespan
so HTTP connections are cleaned up during shutdown (mounted application
lifespans are not run automatically):

```python
from contextlib import asynccontextmanager
from starlette.applications import Starlette
from starlette.routing import Mount

acp_app = create_asgi_app(lambda conn: MyAgent())

@asynccontextmanager
async def lifespan(app):
    async with acp_app.router.lifespan_context(acp_app):
        yield

app = Starlette(routes=[Mount("/agents", app=acp_app)], lifespan=lifespan)
# Connect to /agents/acp using either HTTP or WebSocket.
```

With `path="/rpc"`, the mounted endpoint is `/agents/rpc`. Use `path="/"` to
serve ACP at the mount root (`/agents/`).

For the connection registry and stream routing, see [Transport architecture](../../explanation/transports.md).

### Compatibility notes

The experimental server API uses Starlette. `create_asgi_app(agent_factory, *, path="/acp")`
returns a Starlette application, and `AcpServer.handle_post()` and `handle_delete()`
return Starlette response objects. `open_stream()` and `close()` retain their signatures.

Code using the earlier `AcpAsgiApp`, `PostResult`, `ConnectionRegistry`,
`ConnectionState`, `AcpServer.registry`, or `create_websocket_connection()` wrappers
must migrate to these APIs. Direct WebSocket integrations use
`handle_websocket(agent_factory, websocket)` with a Starlette `WebSocket`.
The default route is `/acp`; pass `path="/"` explicitly if you need the mount root.

### HTTP/2 server requirement

> ⚠️ **Uvicorn does not serve HTTP/2.** For a spec-compliant Streamable HTTP
> server, run an HTTP/2-capable ASGI server (**Hypercorn**, Daphne, Granian) or
> terminate HTTP/2 at a proxy. The WebSocket profile works on Uvicorn.

```python
import asyncio
import hypercorn.asyncio
from hypercorn.config import Config

config = Config()
config.bind = ["localhost:8000"]
config.alpn_protocols = ["h2", "http/1.1"]
asyncio.run(hypercorn.asyncio.serve(app, config))
```

## Examples

- [`examples/http_server.py`](https://github.com/agentclientprotocol/python-sdk/blob/main/examples/http_server.py) — serve an agent over HTTP + WS (Hypercorn).
- [`examples/http_client.py`](https://github.com/agentclientprotocol/python-sdk/blob/main/examples/http_client.py) — connect over Streamable HTTP.
- [`examples/ws_client.py`](https://github.com/agentclientprotocol/python-sdk/blob/main/examples/ws_client.py) — connect over WebSocket.

## Identity model

- `Acp-Connection-Id` (HTTP header) — returned by `initialize`; required on all
  post-initialize HTTP requests and every GET stream.
- `Acp-Session-Id` (HTTP header) — required on session-scoped POSTs and the
  session-scoped GET stream.
- `sessionId` (JSON-RPC field) — carried in params/results and used to route
  messages to the correct stream.

## Not yet supported (deferred to a future revision)

- `Last-Event-ID` / SSE resumability and message sequencing.
- Client-side automatic reconnect/backoff.
- Batch JSON-RPC (the server returns `501`).
- `Acp-Protocol-Version` header enforcement.
