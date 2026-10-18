# Transport behavior

Stdio uses byte streams between a client and an agent process. The experimental
HTTP and WebSocket adapters expose message transports to the same ACP connection
layer. See the [web transport guide](../reference/experimental/web-transports.md) for setup and APIs.

## HTTP and WebSocket

HTTP uses POST requests to send messages and SSE streams to receive them.
After initialization, accepted POSTs return `202 Accepted`; agent responses
arrive on the appropriate SSE stream.

HTTP output follows these routing rules:

| Message | Destination | Why |
| --- | --- | --- |
| `initialize` response | POST body | Establishes the connection before GET streams open |
| Response containing a new `sessionId` | Connection SSE stream | The client needs the ID before it can open the session stream |
| `session/load` replay and response, `session/resume` response | Connection SSE stream | Replay precedes the response; the client gets the session ID from the original request |
| Other messages | Session SSE stream when known, otherwise connection stream | Responses use their request's recorded session; requests/notifications carry `sessionId` |

Idle SSE streams emit keepalives. `DELETE` and server shutdown close the HTTP
connections and cancel their agent work.

The transport retains connection and stream state, not conversation history.
Reopening a transport creates a new connection; loading or resuming a session
depends on the agent's storage and advertised capabilities. A dropped
connection-scoped SSE stream ends the client's receive loop and rejects pending
requests. A session stream ending does not close the whole connection. The SDK
does not retry either stream automatically.

WebSocket uses one bidirectional connection for all messages. Disconnecting
closes the agent connection and cancels its work.
