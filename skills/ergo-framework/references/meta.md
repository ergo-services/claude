# Meta Processes

Meta processes bridge blocking I/O (sockets, subprocesses, HTTP) into the actor model. They run **two goroutines**: an external reader (blocked on I/O) and a message dispatcher. They are spawned by an actor via `SpawnMeta`, get an `Alias` (not a PID), and communicate only via async messages.

## Restrictions

| Limitation | Why |
|-----------|-----|
| Cannot `Call` | No sync request semantics from a meta process |
| Cannot create Link or Monitor | Can only **receive** links/monitors initiated by actors |
| Mailbox available only via `Send` | Meta runs alongside a blocking I/O loop |

## MetaBehavior

```go
type MetaBehavior interface {
    Init(process gen.MetaProcess) error
    Start() error                                  // long-running I/O loop
    HandleMessage(from gen.PID, message any) error
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)
    Terminate(reason error)
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

## Spawning

```go
alias, err := actor.SpawnMeta(metaBehavior, gen.MetaOptions{
    MailboxSize:  1000,
    SendPriority: gen.MessagePriorityNormal,
    Compression:  false,
    LogLevel:     gen.LogLevelInfo,
})
```

The returned `Alias` is how other processes address this meta process.

## TCP Server

```go
server, err := meta.CreateTCPServer(meta.TCPServerOptions{
    Host:        "0.0.0.0",
    Port:        8080,
    ProcessPool: []gen.Atom{"handler-1", "handler-2"}, // round-robin routing
})
if err != nil {
    return err
}
alias, err := actor.SpawnMeta(server, gen.MetaOptions{})
```

Each incoming connection is assigned to one of the processes in `ProcessPool` via round-robin. The chosen process receives `MessageTCPConnect`, subsequent `MessageTCP` data frames, and finally `MessageTCPDisconnect`.

| Message | Fields |
|---------|--------|
| `MessageTCPConnect` | `ID gen.Alias, RemoteAddr, LocalAddr net.Addr` |
| `MessageTCP` | `ID gen.Alias, Data []byte` |
| `MessageTCPDisconnect` | `ID gen.Alias` |

Reply on the connection by sending `MessageTCP{ID: connID, Data: response}` to the connection's alias.

```go
func (h *Handler) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case meta.MessageTCPConnect:
        h.Log().Info("client %s", m.RemoteAddr)
    case meta.MessageTCP:
        h.Send(m.ID, meta.MessageTCP{ID: m.ID, Data: []byte("ack\n")})
    case meta.MessageTCPDisconnect:
        h.Log().Info("disconnected %s", m.ID)
    }
    return nil
}
```

## TCP Connection (Client)

```go
conn, err := meta.CreateTCPConnection(meta.TCPConnectionOptions{
    Host:    "example.com",
    Port:    443,
    Process: "response-handler",
    CertManager: tlsCerts,
})
connID, err := actor.SpawnMeta(conn, gen.MetaOptions{})

actor.Send(connID, meta.MessageTCP{ID: connID, Data: requestBytes})
```

Uses the same `MessageTCPConnect/MessageTCP/MessageTCPDisconnect` types as the server.

## UDP Server

```go
server, _ := meta.CreateUDPServer(meta.UDPServerOptions{
    Host:       "0.0.0.0",
    Port:       5353,
    Process:    "udp-handler",
    BufferSize: 2048,
})
alias, _ := actor.SpawnMeta(server, gen.MetaOptions{})
```

All incoming datagrams are delivered to the single process named in `Process`:

```go
type MessageUDP struct {
    ID   gen.Alias
    Addr net.Addr
    Data []byte
}
```

Reply by sending `MessageUDP{ID: serverAlias, Addr: peerAddr, Data: response}`.

## Web Server

```go
server, _ := meta.CreateWebServer(meta.WebServerOptions{
    Host:    "0.0.0.0",
    Port:    8080,
    Handler: http.DefaultServeMux,
})
alias, _ := actor.SpawnMeta(server, gen.MetaOptions{})
```

Handler is any `http.Handler`. Use `meta.CreateWebHandler` to bridge HTTP requests to actor messages:

```go
handler := meta.CreateWebHandler(meta.WebHandlerOptions{
    Worker:         "web-worker",
    RequestTimeout: 30 * time.Second,
})
http.Handle("/api/", handler)
```

The worker receives:

```go
type MessageWebRequest struct {
    Response http.ResponseWriter
    Request  *http.Request
    Done     func()  // call when response is complete
}
```

**Important:** call `Done()` after writing the response; otherwise the HTTP handler goroutine leaks until timeout.

## Port (Subprocess)

```go
port, _ := meta.CreatePort(meta.PortOptions{
    Cmd:     "/usr/bin/wc",
    Args:    []string{"-l"},
    Tag:     "line-counter",
    Process: "port-owner",
})
alias, _ := actor.SpawnMeta(port, gen.MetaOptions{})
```

Exchange data with the subprocess via messages:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `MessagePortStart{ID, Tag}` | from port | Process started |
| `MessagePortTerminate{ID, Tag}` | from port | Process exited |
| `MessagePortText{ID, Tag, Text}` | from port | stdout line (text mode) |
| `MessagePortData{ID, Tag, Data}` | from port | stdout chunk (binary mode) |
| `MessagePortError{ID, Tag, Error}` | from port | stderr output or error |

Binary mode via `PortOptions.Binary.Enable = true`. Custom framing via `SplitFuncStdout` / `SplitFuncStderr` (use `bufio.ScanLines`, `bufio.ScanBytes`, or a custom `bufio.SplitFunc`).

## ProcessPool Routing Semantics

Applies to `TCPServerOptions.ProcessPool`. A connection picks member `ProcessPool[i % len(ProcessPool)]` on accept. Same member handles all frames for that connection (ordered). Different connections may land on different members in parallel.

**Do not** use `act.Pool` here — see `pool.md` for why.

## When to Use Meta vs Actor

| Need | Use |
|------|-----|
| Blocking syscall (`read`, `accept`) | Meta |
| Fork a subprocess | Meta (Port) |
| Serve HTTP and route to business logic | Meta (WebServer/Handler) + actor worker |
| In-memory message routing | Actor or Pool |
| Computation with external state | Actor |
| Per-connection protocol state | Meta (reader) + actor (handler via ProcessPool) |

## Anti-Patterns

- **Never call `Call()` from a meta process** — the API does not expose it.
- **Never link/monitor from a meta process** — only actors can initiate; meta can receive.
- **Do not forget to call `Done()`** in `MessageWebRequest` — HTTP goroutines leak otherwise.
- **Do not use `act.Pool` in `TCPServerOptions.ProcessPool`** — breaks per-connection routing.
- **Do not block inside `HandleMessage`** of the process receiving meta events — same actor-callback rule applies.
