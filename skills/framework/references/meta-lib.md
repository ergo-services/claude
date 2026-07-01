# WebSocket and SSE Meta Processes

Separate modules for protocols not covered by core `ergo/meta`. Both integrate with the core `meta.WebServer` via handlers.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/meta/websocket` | `websocket` | WebSocket server and client |
| `ergo.services/meta/sse` | `sse` | Server-Sent Events (unidirectional HTTP push) |

Prerequisites: understanding of core `meta` processes (`references/meta.md`) and `meta.WebServer` / `meta.WebHandler` integration.

Neither client auto-reconnects. When the underlying socket closes, the meta process terminates and emits `MessageDisconnect`. Resuming is the caller's job - respawn the connection (see the resume notes per protocol).

## websocket - WebSocket

Each WebSocket connection becomes a meta process. On the server side the connection is routed to a worker from `ProcessPool` (round-robin), or to the parent actor if `ProcessPool` is empty. Actors exchange typed messages for lifecycle and frames.

### HandlerOptions (Server Side)

```go
type HandlerOptions struct {
    ProcessPool       []gen.Atom          // round-robin worker routing; parent if empty
    HandshakeTimeout  time.Duration       // default: 15s
    EnableCompression bool                // per-message-deflate
    CheckOrigin       func(r *http.Request) bool
}
```

Leaving `ProcessPool` empty routes every connection message to the parent actor - the one that spawned the handler. That is the minimal single-worker setup; add `ProcessPool` only to fan connections across dedicated worker actors.

### ConnectionOptions (Client Side)

```go
type ConnectionOptions struct {
    Process           gen.Atom  // receiver (parent if empty)
    URL               url.URL
    HandshakeTimeout  time.Duration  // default: 15s
    EnableCompression bool
}
```

### Messages

| Type | Fields | Direction |
|------|--------|-----------|
| `MessageConnect` | `ID gen.Alias, RemoteAddr, LocalAddr net.Addr` | to worker on connect |
| `MessageDisconnect` | `ID gen.Alias` | to worker on close |
| `Message` | `ID gen.Alias, Type MessageType, Body []byte` | both directions (frames) |

`MessageType` constants: `MessageTypeText` (1), `MessageTypeBinary` (2), `MessageTypeClose` (8), `MessageTypePing` (9), `MessageTypePong` (10).

### Hook-in (Server)

```go
import (
    "ergo.services/ergo/meta"
    "ergo.services/meta/websocket"
)

// In your actor's Init
func (a *Gateway) Init(args ...any) error {
    wsHandler := websocket.CreateHandler(websocket.HandlerOptions{
        ProcessPool:      []gen.Atom{"ws-worker-1", "ws-worker-2"},
        HandshakeTimeout: 15 * time.Second,
    })

    mux := http.NewServeMux()
    mux.Handle("/ws", wsHandler)

    webServer, _ := meta.CreateWebServer(meta.WebServerOptions{
        Host:    "0.0.0.0",
        Port:    8080,
        Handler: mux,
    })
    _, err := a.SpawnMeta(webServer, gen.MetaOptions{})
    return err
}
```

### Worker Actor

```go
func (w *Worker) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case websocket.MessageConnect:
        w.Log().Info("ws client %s", m.RemoteAddr)
    case websocket.Message:
        if m.Type == websocket.MessageTypeText {
            // Echo back. m.ID is the connection alias.
            w.SendAlias(m.ID, websocket.Message{
                ID:   m.ID,
                Type: websocket.MessageTypeText,
                Body: m.Body,
            })
        }
    case websocket.MessageDisconnect:
        w.Log().Info("ws closed %s", m.ID)
    }
    return nil
}
```

`m.ID` is the connection's alias. `SendAlias(connID, ...)` pushes a frame back to that connection; `Send(connID, ...)` also works because `Send` accepts an `Alias`.

### Closing a Connection From the Actor

To tear down a live connection, call `SendExitMeta` with the connection alias:

```go
w.SendExitMeta(connID, errors.New("done"))
```

The meta process closes the socket and the worker then receives `MessageDisconnect`. The reason is only logged - it is not delivered to the peer. This is distinct from sending a `websocket.MessageTypeClose` frame (a protocol-level close over the open socket).

### Hook-in (Client)

`websocket.CreateConnection` dials eagerly - it performs the handshake immediately and returns a dial error before you ever call `SpawnMeta`. Handle that error at creation time:

```go
conn, err := websocket.CreateConnection(websocket.ConnectionOptions{
    Process: "ws-client-handler",
    URL:     parsedURL,
})
if err != nil {
    return err // handshake failed - nothing was spawned
}
connID, _ := actor.SpawnMeta(conn, gen.MetaOptions{})

actor.SendAlias(connID, websocket.Message{
    ID:   connID,
    Type: websocket.MessageTypeText,
    Body: []byte("hello"),
})
```

The client meta process does not reconnect. On any read error it terminates and sends `MessageDisconnect` to its receiver; to resume, call `CreateConnection` again and spawn a fresh connection.

## sse - Server-Sent Events

Unidirectional server-to-client HTTP push. Unlike WebSocket, there is no bidirectional channel - the server writes events, the client only reads. Useful for live feeds, notifications, log streams. The SSE protocol carries a `Last-Event-ID` for resume, but the client meta process does not act on it automatically (see resume notes below).

### HandlerOptions (Server Side)

```go
type HandlerOptions struct {
    ProcessPool []gen.Atom    // round-robin worker routing; parent if empty
    Heartbeat   time.Duration // keepalive ping; default: 30s
    Compression bool          // gzip for supporting clients
}
```

As with WebSocket, an empty `ProcessPool` routes every connection message to the parent actor.

### ConnectionOptions (Client Side)

```go
type ConnectionOptions struct {
    Process           gen.Atom      // receiver (parent if empty)
    URL               url.URL
    Headers           http.Header
    LastEventID       string        // sent as the Last-Event-ID request header
    ReconnectInterval time.Duration // tracked only; default: 3s
}
```

`ReconnectInterval` is only tracked - it is seeded from this option, updated from the server's `retry:` hint as events stream in, and reported via `HandleInspect`. It does not itself trigger a reconnect; the client never redials on its own.

### Messages

| Type | Fields | Direction |
|------|--------|-----------|
| `MessageConnect` | `ID gen.Alias, RemoteAddr, LocalAddr net.Addr, Request *http.Request` | to worker on connect |
| `MessageDisconnect` | `ID gen.Alias` | on close |
| `Message` | `ID gen.Alias, Event string, Data []byte, MsgID string, Retry int` | server to client |
| `MessageLastEventID` | `ID gen.Alias, LastEventID string` | to worker on client reconnect (server side) |

`Event` is the SSE event type (optional). `MsgID` becomes the client's `Last-Event-ID`. `Retry` is a millisecond reconnect hint (server side only).

`MessageConnect` address fields are fully populated only on the server side (from the HTTP request - `RemoteAddr`, `LocalAddr`, and `Request`). The client-side `MessageConnect` carries `ID` and, when the transport exposes them, `Request` and `RemoteAddr`; it never sets `LocalAddr`. `MessageLastEventID` is emitted server side when a client reconnects carrying a `Last-Event-ID` header - the client meta process never sends it.

### Hook-in (Server)

```go
import (
    "ergo.services/ergo/meta"
    "ergo.services/meta/sse"
)

func (a *Feed) Init(args ...any) error {
    sseHandler := sse.CreateHandler(sse.HandlerOptions{
        ProcessPool: []gen.Atom{"feed-writer"},
        Heartbeat:   30 * time.Second,
    })

    mux := http.NewServeMux()
    mux.Handle("/events", sseHandler)

    server, _ := meta.CreateWebServer(meta.WebServerOptions{
        Host: "0.0.0.0", Port: 8080, Handler: mux,
    })
    _, err := a.SpawnMeta(server, gen.MetaOptions{})
    return err
}
```

### Worker Pushing Events

```go
func (w *FeedWriter) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case sse.MessageConnect:
        w.subscribers[m.ID] = struct{}{}
    case sse.MessageDisconnect:
        delete(w.subscribers, m.ID)
    case sse.MessageLastEventID:
        // replay events since m.LastEventID
    case NewsItem:
        for id := range w.subscribers {
            w.SendAlias(id, sse.Message{
                ID:    id,
                Event: "news",
                Data:  mustJSON(m),
                MsgID: m.ID,
            })
        }
    }
    return nil
}
```

`SendAlias(connID, sse.Message{...})` pushes an event to a connection; `Send` with the same alias also works. Only the server-side connection meta consumes `sse.Message` and writes it to the stream. To close a subscriber's stream from the actor, call `SendExitMeta(connID, reason)` - the meta process closes the connection and the worker receives `MessageDisconnect`.

### Hook-in (Client)

`sse.CreateConnection` returns a single value and dials lazily - the HTTP GET happens later, when the meta process starts. Connection failures therefore surface after `SpawnMeta`, as a logged error followed by `MessageDisconnect`, not at creation time:

```go
conn := sse.CreateConnection(sse.ConnectionOptions{
    Process: "news-consumer",
    URL:     parsedURL,
})
connID, _ := actor.SpawnMeta(conn, gen.MetaOptions{})

// Worker receives sse.Message{Event, Data, MsgID} as events stream in,
// then MessageDisconnect when the stream ends.
```

The SSE client is strictly read-only. Sending any message to `connID` is ignored - the client meta process logs it as read-only and drops it. The client only emits `MessageConnect`, `Message`, and `MessageDisconnect` upward to its receiver.

The client does not reconnect. On disconnect it terminates and emits `MessageDisconnect`. To resume, respawn the connection with `ConnectionOptions.LastEventID` set to the last received `MsgID`; that value is sent as the `Last-Event-ID` request header, and the server side receives it as `MessageLastEventID` so it can replay missed events.

## Choosing Between WebSocket and SSE

| Need | Pick |
|------|------|
| Bidirectional messaging | WebSocket |
| Server-push only, HTTP infrastructure-friendly | SSE |
| Browser with proxy/firewall hostility | SSE (plain HTTP) |
| Binary frames | WebSocket |
| Resume from a known point after disconnect | SSE (`Last-Event-ID` on the wire) |
| Many clients, low per-client state | SSE (fewer protocol overheads) |

Neither client auto-reconnects, so "resume semantics" is a property of the SSE wire format, not automatic behavior - your actor decides when to respawn.

## Anti-Patterns

- **Do not forget `CheckOrigin`** in WebSocket HandlerOptions when exposed publicly - default-deny unknown origins.
- **Do not block in the worker actor** when broadcasting to many subscribers - batch or use a pool of writer actors.
- **Do not send `MessageTypePing` manually** - the gorilla implementation handles ping/pong automatically; application-level pings require explicit protocol agreement.
- **Do not forget `Heartbeat`** in SSE; proxies close idle connections.
- **Do not expect the client to reconnect on its own** - both the SSE and WebSocket client meta processes terminate on disconnect and emit `MessageDisconnect`. To resume SSE, respawn with `ConnectionOptions.LastEventID`; `ReconnectInterval` is tracked but never triggers a redial.
- **Do not push to the SSE client connection** - the client meta process is read-only and silently drops any message you send it. Only the server side consumes `sse.Message`.
- **Do not put business logic in the meta process** - meta routes frames; real handling belongs in the worker actor from `ProcessPool`.
