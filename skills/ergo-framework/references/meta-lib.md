# WebSocket and SSE Meta Processes

Separate modules for protocols not covered by core `ergo/meta`. Both integrate with the core `meta.WebServer` via handlers.

| Module | Package | Purpose |
|--------|---------|---------|
| `ergo.services/meta/websocket` | `websocket` | WebSocket server and client |
| `ergo.services/meta/sse` | `sse` | Server-Sent Events (unidirectional HTTP push) |

Prerequisites: understanding of core `meta` processes (`references/meta.md`) and `meta.WebServer` / `meta.WebHandler` integration.

## websocket — WebSocket

Each WebSocket connection becomes a meta process. The connection is routed to a worker from `ProcessPool` (round-robin). Actors exchange typed messages for lifecycle and frames.

### HandlerOptions (Server Side)

```go
type HandlerOptions struct {
    ProcessPool       []gen.Atom          // round-robin worker routing
    HandshakeTimeout  time.Duration       // default: 15s
    EnableCompression bool                // per-message-deflate
    CheckOrigin       func(*http.Request) bool
}
```

### ConnectionOptions (Client Side)

```go
type ConnectionOptions struct {
    Process           gen.Atom  // receiver (parent if empty)
    URL               url.URL
    HandshakeTimeout  time.Duration
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
            // Echo back
            w.Send(m.ID, websocket.Message{
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

### Hook-in (Client)

```go
conn, _ := websocket.CreateConnection(websocket.ConnectionOptions{
    Process: "ws-client-handler",
    URL:     parsedURL,
})
connID, _ := actor.SpawnMeta(conn, gen.MetaOptions{})

actor.Send(connID, websocket.Message{
    ID:   connID,
    Type: websocket.MessageTypeText,
    Body: []byte("hello"),
})
```

## sse — Server-Sent Events

Unidirectional server→client HTTP push. Unlike WebSocket, no bidirectional channel — clients use Last-Event-ID for resume semantics. Useful for live feeds, notifications, log streams.

### HandlerOptions (Server Side)

```go
type HandlerOptions struct {
    ProcessPool []gen.Atom    // round-robin
    Heartbeat   time.Duration // keepalive ping
    Compression bool          // gzip for supporting clients
}
```

### ConnectionOptions (Client Side)

```go
type ConnectionOptions struct {
    Process           gen.Atom      // receiver
    URL               url.URL
    Headers           http.Header
    LastEventID       string        // for resume
    ReconnectInterval time.Duration
}
```

### Messages

| Type | Fields | Direction |
|------|--------|-----------|
| `MessageConnect` | `ID gen.Alias, RemoteAddr, LocalAddr net.Addr, Request *http.Request` | to worker on connect |
| `MessageDisconnect` | `ID gen.Alias` | on close |
| `Message` | `ID gen.Alias, Event string, Data []byte, MsgID string, Retry int` | server → client |
| `MessageLastEventID` | `ID gen.Alias, LastEventID string` | on client reconnect |

`Event` is the SSE event type (optional). `MsgID` becomes the client's `Last-Event-ID`. `Retry` is a millisecond reconnect hint.

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
            w.Send(id, sse.Message{
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

### Hook-in (Client)

```go
conn, _ := sse.CreateConnection(sse.ConnectionOptions{
    Process: "news-consumer",
    URL:     parsedURL,
})
connID, _ := actor.SpawnMeta(conn, gen.MetaOptions{})

// Worker receives sse.Message{Event, Data, MsgID} as events stream in.
```

## Choosing Between WebSocket and SSE

| Need | Pick |
|------|------|
| Bidirectional messaging | WebSocket |
| Server-push only, HTTP infrastructure-friendly | SSE |
| Browser with proxy/firewall hostility | SSE (plain HTTP) |
| Binary frames | WebSocket |
| Simple retry/resume semantics | SSE (Last-Event-ID built in) |
| Many clients, low per-client state | SSE (fewer protocol overheads) |

## Anti-Patterns

- **Do not forget `CheckOrigin`** in WebSocket HandlerOptions when exposed publicly — default-deny unknown origins.
- **Do not block in the worker actor** when broadcasting to many subscribers — batch or use a pool of writer actors.
- **Do not send `MessageTypePing` manually** — the gorilla implementation handles ping/pong automatically; application-level pings require explicit protocol agreement.
- **Do not forget `Heartbeat`** in SSE; proxies close idle connections.
- **Do not rely on HTTP keep-alive for SSE resume** — use `Last-Event-ID` / `MsgID` so clients resume from the right point after disconnect.
- **Do not put business logic in the meta process** — meta routes frames; real handling belongs in the worker actor from `ProcessPool`.
