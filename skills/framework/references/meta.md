# Meta Processes

Meta processes bridge blocking I/O (sockets, subprocesses, HTTP) into the actor model. They run **two goroutines**: an external reader (your `Start()` method, usually blocked on I/O) and an on-demand message handler that runs `HandleMessage` / `HandleCall` sequentially. They are spawned by an actor via `SpawnMeta`, get an `Alias` (not a PID), and are owned by the spawning process - when the parent terminates, its metas terminate with it.

## What a meta can and cannot do

| Capability | Meta process |
|-----------|--------------|
| Address type | `gen.Alias` (never a PID) |
| Be the CALLER of `Call` | No - `MetaProcess` has no `Call` method |
| RECEIVE a `Call` | Yes - via `HandleCall`, reply with `SendResponse` / a non-nil return |
| Create a Link or Monitor | No - `MetaProcess` has no link/monitor API |
| Receive a Link or Monitor | Yes - actors can link/monitor a meta by its alias |
| Spawn children | Meta processes only (via `MetaProcess.Spawn`), never regular processes |
| Message queues | 2 (system + main); no urgent, no log queue |

The `Call` restriction is directional. A meta cannot originate a synchronous request (there is no goroutine that can safely block for the reply), but it can receive one: an actor's `Call` to the meta's alias is delivered to `HandleCall`, and the meta replies. See [Receiving Call](#receiving-call).

## MetaBehavior

The interface you implement:

```go
type MetaBehavior interface {
    Init(process gen.MetaProcess) error              // store the handle, prepare resources
    Start() error                                    // external reader - blocking I/O loop
    HandleMessage(from gen.PID, message any) error   // actor sent async message
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) // actor's Call
    Terminate(reason error)                          // cleanup; do not block or panic
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`Init` stores the `gen.MetaProcess` handle. `Start` runs in the external-reader goroutine and typically loops forever on `Accept`/`Read`; when it returns the meta terminates. `HandleMessage`/`HandleCall` run in the handler goroutine, one message at a time.

## The MetaProcess handle

`Init` receives a `gen.MetaProcess`. This handle is the entire API a meta implementation uses to interact with the actor system. Which methods are callable depends on the current state (see [States](#states)).

| Method | States | Purpose |
|--------|--------|---------|
| `ID() gen.Alias` | all | This meta's alias |
| `Parent() gen.PID` | all | PID of the spawning process |
| `Send(to, message any) error` | all | Async send; `to` may be PID/ProcessID/Alias/Atom name |
| `SendWithPriority(to, message any, p MessagePriority) error` | all | Async send at a priority |
| `Spawn(b MetaBehavior, o MetaOptions) (Alias, error)` | Sleep, Running | Spawn a child meta (e.g. per connection) |
| `SendResponse(to PID, ref Ref, message any) error` | Running only | Reply to a received `Call` |
| `SendResponseError(to PID, ref Ref, err error) error` | Running only | Reply to a received `Call` with an error |
| `SetSendPriority(p MessagePriority) error` | all | Set default send priority |
| `SetCompression(enabled bool) error` | Running only | Toggle compression for messages this meta sends |
| `SendPriority() MessagePriority`, `Compression() bool` | all | Read current settings |
| `Env`, `EnvList`, `EnvDefault` | all | Environment inherited from the parent |
| `Log() gen.Log` | all | Logger |

`Send` / `SendWithPriority` / `Spawn` are the load-bearing part: they are callable from the `Start()` goroutine while the meta is still in **Sleep**. This is exactly how a TCP server pushes accepted-connection data and spawns a per-connection child meta directly from its blocking accept loop, without ever entering a message-handler callback.

Methods marked **Running only** return `gen.ErrNotAllowed` when called outside the handler goroutine, because they depend on the `gen.Ref` of an in-flight `HandleCall` (`SendResponse*`) or on handler-goroutine ownership (`SetCompression`). `SetSendPriority` stores unconditionally in the current implementation (its godoc still claims Running-only); do not rely on the guard.

## Receiving Call

An actor may `Call` a meta by its alias. The request lands in `HandleCall`:

```go
func (h *Handler) HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) {
    switch r := request.(type) {
    case StatusRequest:
        return StatusResponse{Open: h.open}, nil   // non-nil result -> auto-replied
    case SlowRequest:
        h.pending[ref] = from                       // store from+ref, reply later
        return nil, nil                              // nil -> no auto-reply yet
    }
    return nil, gen.ErrUnsupported
}
```

Returning a non-nil result sends the response automatically. Returning `(nil, nil)` sends no response - to defer, store `from` and `ref`, then call `SendResponse(from, ref, result)` (or `SendResponseError`) later from the handler goroutine. Only `Call` a meta whose `HandleCall` you control: the stock metas do not usefully reply. TCP connection/server-child, UDP, Web server, and Port return `(nil, nil)`, so a synchronous `Call` to them blocks until timeout (the TCP server and Web handler instead auto-reply with `gen.ErrUnsupported` as the result value). The deferred path matters only for custom metas.

## States

| State | External reader | Handler goroutine |
|-------|-----------------|-------------------|
| `MetaStateSleep` | running (blocked on I/O) | idle / not spawned |
| `MetaStateRunning` | running | active in `HandleMessage`/`HandleCall` |
| `MetaStateTerminated` | stopped | stopped |

Transitions are automatic: a message arrives and the handler spawns (Sleep to Running); the mailbox empties and it exits (Running to Sleep); `Start()` returns and the meta terminates.

## Spawning

```go
alias, err := actor.SpawnMeta(metaBehavior, gen.MetaOptions{
    MailboxSize:  1000,                          // bounds the meta mailbox (0 = unbounded)
    SendPriority: gen.MessagePriorityNormal,     // default priority for this meta's sends
    Compression:  false,                         // compress cross-node messages this meta sends
    LogLevel:     gen.LogLevelInfo,
})
```

`SpawnMeta` returns an `Alias`; that is how other processes address the meta. Terminate a meta from its parent with `process.SendExitMeta(alias, reason)`.

Meta processes appear in node inspection as `gen.MetaInfo` (ID, Parent, Application, Behavior, mailbox size/queues, MessagesIn/Out, Uptime, State). `State` is one of the `MetaState` values above.

## Default routing target: Parent

For every built-in meta with a `Process` / `Worker` option, leaving it empty (`""`) delivers all messages to the **spawning actor** (`Parent()`). Set it to a registered `gen.Atom` name only to route elsewhere. This applies to the TCP connection, UDP server, Port, and Web handler.

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

The server accepts connections and spawns one child meta per connection. Each connection is assigned to a member of `ProcessPool` by round-robin. If `ProcessPool` is empty the connection routes to the parent. The chosen process receives `MessageTCPConnect`, then `MessageTCP` data frames, then `MessageTCPDisconnect`.

| Message | Fields |
|---------|--------|
| `MessageTCPConnect` | `ID gen.Alias, RemoteAddr, LocalAddr net.Addr` |
| `MessageTCP` | `ID gen.Alias, Data []byte` |
| `MessageTCPDisconnect` | `ID gen.Alias` |

Reply on the connection by sending `MessageTCP{ID: connID, Data: response}` to the connection alias (`m.ID`).

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

### TCP options (server and connection)

`TCPServerOptions` and `TCPConnectionOptions` share the same tuning fields:

| Field | Effect |
|-------|--------|
| `CertManager gen.CertManager` | Enables TLS. Build one with `gen.CreateCertManager(cert)` |
| `InsecureSkipVerify bool` | Skip peer certificate verification |
| `ReadBufferSize int` | Per-read buffer; defaults to `gen.DefaultTCPBufferSize` (65535) |
| `ReadBufferPool *sync.Pool` | Optional pool of `[]byte` for zero-alloc reads; `CreateTCPServer`/`CreateTCPConnection` error if it is not a pool of `[]byte`. The consumer that receives `MessageTCP.Data` owns recycling |
| `ReadChunk ChunkOptions` | Length-framed reads (see [Message framing](#message-framing-chunkoptions)) |
| `WriteBufferKeepAlive []byte` / `WriteBufferKeepAlivePeriod time.Duration` | Periodic keepalive bytes written to the socket |
| `Advanced.KeepAlivePeriod time.Duration` | TCP keepalive period on the underlying socket |

TLS example:

```go
cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
server, _ := meta.CreateTCPServer(meta.TCPServerOptions{
    Host:        "0.0.0.0",
    Port:        8443,
    ProcessPool: []gen.Atom{"tls-handler"},
    CertManager: gen.CreateCertManager(cert),
})
```

## TCP Connection (Client)

```go
conn, err := meta.CreateTCPConnection(meta.TCPConnectionOptions{
    Host:        "example.com",
    Port:        443,
    Process:     "response-handler",             // empty -> parent
    CertManager: gen.CreateCertManager(cert),    // enables TLS
})
connID, err := actor.SpawnMeta(conn, gen.MetaOptions{})

actor.Send(connID, meta.MessageTCP{ID: connID, Data: requestBytes})
```

Uses the same `MessageTCPConnect` / `MessageTCP` / `MessageTCPDisconnect` types and the same option fields as the server.

## Message framing (ChunkOptions)

By default a TCP connection (and a binary Port) emits whatever bytes each `Read` returns - one `MessageTCP`/`MessagePortData` per read, with no message boundaries. `ChunkOptions` (the `ReadChunk` field) turns the reader into a length-framer: it reassembles bytes into complete frames and emits exactly one message per frame, regardless of how the OS split the stream.

```go
ReadChunk: meta.ChunkOptions{
    Enable:                     true,
    HeaderSize:                 4,      // bytes to buffer before the length is known
    HeaderLengthPosition:       0,      // offset of the length field within the header
    HeaderLengthSize:           4,      // 1, 2, or 4 bytes, big-endian
    HeaderLengthIncludesHeader: false,  // false -> length counts payload only, header added on top
    MaxLength:                  1 << 20, // reject frames over this size
}
```

| Field | Meaning |
|-------|---------|
| `Enable` | Turn framing on |
| `FixedLength` | If > 0, every frame is exactly this many bytes (header fields ignored) |
| `HeaderSize` | Header byte count; required for dynamic (non-fixed) framing |
| `HeaderLengthPosition` | Byte offset of the length field inside the header |
| `HeaderLengthSize` | Width of the length field: 1, 2, or 4 bytes, big-endian |
| `HeaderLengthIncludesHeader` | If false, the decoded length is payload-only and `HeaderSize` is added |
| `MaxLength` | If > 0, a frame exceeding this fails the reader with `gen.ErrTooLarge` |

`CreateTCPServer` / `CreateTCPConnection` / `CreatePort` validate `ReadChunk` at creation and return an error if the configuration is inconsistent (for example a dynamic frame with `HeaderSize == 0`, or a `HeaderLengthSize` other than 1/2/4).

## UDP Server

```go
server, _ := meta.CreateUDPServer(meta.UDPServerOptions{
    Host:       "0.0.0.0",
    Port:       5353,
    Process:    "udp-handler",   // empty -> parent
    BufferSize: 2048,            // per-read buffer; default 65000
    BufferPool: nil,             // optional *sync.Pool of []byte
})
alias, _ := actor.SpawnMeta(server, gen.MetaOptions{})
```

All datagrams are delivered to the single process in `Process` (or the parent when empty):

```go
type MessageUDP struct {
    ID   gen.Alias
    Addr net.Addr
    Data []byte
}
```

Reply by sending `MessageUDP{ID: serverAlias, Addr: peerAddr, Data: response}` back to the server alias.

## Web Server

```go
server, _ := meta.CreateWebServer(meta.WebServerOptions{
    Host:        "0.0.0.0",
    Port:        8080,
    Handler:     http.DefaultServeMux,
    CertManager: nil,   // set to enable TLS
})
alias, _ := actor.SpawnMeta(server, gen.MetaOptions{})
```

`Handler` is any `http.Handler`. To bridge HTTP requests into actor messages, use `meta.CreateWebHandler`, which returns a `WebHandler` - it is both an `http.Handler` (usable directly in a mux) and a `gen.MetaBehavior` (spawnable via `SpawnMeta`).

```go
handler := meta.CreateWebHandler(meta.WebHandlerOptions{
    Worker:         "web-worker",           // empty -> parent
    RequestTimeout: 30 * time.Second,       // defaults to 5s when zero
})
http.Handle("/api/", handler)
```

The worker receives:

```go
type MessageWebRequest struct {
    Response http.ResponseWriter
    Request  *http.Request
    Done     func()  // signals the request is complete
}
```

**Call `Done()` after writing the response.** `ServeHTTP` blocks the HTTP goroutine until `Done()` is called or `RequestTimeout` elapses. Client-visible outcomes:

| Situation | HTTP status |
|-----------|-------------|
| Handler not yet initialized, terminated, or worker not ready | 503 Service Unavailable |
| Request cannot be delivered to the worker | 502 Bad Gateway |
| `RequestTimeout` elapses before `Done()` | 504 Gateway Timeout |
| `Done()` called | request completes, no error status written |

### Web Handler with act.WebWorker

The framework ships `act.WebWorker`, a first-class actor behavior built to be the `CreateWebHandler` `Worker`. Instead of a raw `MessageWebRequest` switch, it dispatches by HTTP method and calls `Done()` for you:

```go
type API struct {
    act.WebWorker
}

func (a *API) HandleGet(from gen.PID, w http.ResponseWriter, r *http.Request) error {
    w.Write([]byte("ok"))
    return nil   // do NOT call r.Done() - the worker already does
}

func (a *API) HandlePost(from gen.PID, w http.ResponseWriter, r *http.Request) error {
    // ...
    return nil
}
```

`WebWorkerBehavior` provides `HandleGet` / `HandlePost` / `HandlePut` / `HandlePatch` / `HandleDelete` / `HandleHead` / `HandleOptions`, each `(from gen.PID, writer http.ResponseWriter, request *http.Request) error`. The worker calls `Done()` automatically after the handler returns, so never call it there. See `pool.md` for how `act.WebWorker` relates to the wider worker model. Reach for the raw `MessageWebRequest` path in a plain actor only when you need manual control over `Done()`.

## Port (Subprocess)

```go
port, _ := meta.CreatePort(meta.PortOptions{
    Cmd:     "/usr/bin/wc",
    Args:    []string{"-l"},
    Tag:     "line-counter",
    Process: "port-owner",   // empty -> parent
})
alias, _ := actor.SpawnMeta(port, gen.MetaOptions{})
```

Exchange data with the subprocess via messages:

| Message | Direction | Purpose |
|---------|-----------|---------|
| `MessagePortStart{ID, Tag}` | from port | Process started |
| `MessagePortTerminate{ID, Tag}` | from port | Process exited |
| `MessagePortText{ID, Tag, Text}` | both | stdout line / write to stdin (text mode) |
| `MessagePortData{ID, Tag, Data}` | both | stdout chunk / write to stdin (binary mode) |
| `MessagePortError{ID, Tag, Error}` | from port | stderr output |

Send `MessagePortText` / `MessagePortData` back to the port alias to write to the subprocess stdin.

Binary mode via `PortOptions.Binary.Enable = true`. In text mode, custom line framing comes from `SplitFuncStdout` / `SplitFuncStderr` (`bufio.ScanLines`, `bufio.ScanBytes`, or a custom `bufio.SplitFunc`). In binary mode, `Binary.ReadChunk` provides length framing (see [Message framing](#message-framing-chunkoptions)); `Binary.ReadBufferSize` defaults to 8192 and `Binary.ReadBufferPool` accepts a `*sync.Pool` of `[]byte`.

### Subprocess environment

| Field | Effect |
|-------|--------|
| `EnableEnvOS bool` | Inherit the node's OS environment (`os.Environ()`) |
| `EnableEnvMeta bool` | Inject the meta's inherited env (`EnvList()`) |
| `Env map[gen.Env]string` | Explicit variables for this subprocess |

They are assembled in order OS, then meta env, then `Env`, so later entries append after earlier ones. `Binary.WriteBufferKeepAlive` writes periodic keepalive bytes to stdin and requires a non-zero `WriteBufferKeepAlivePeriod` (`CreatePort` errors otherwise).

## ProcessPool Routing Semantics

Applies to `TCPServerOptions.ProcessPool`. A connection picks member `ProcessPool[i % len(ProcessPool)]` on accept. The same member handles all frames for that connection in order. Different connections may land on different members in parallel.

**Do not** use `act.Pool` here - see `pool.md` for why.

## When to Use Meta vs Actor

| Need | Use |
|------|-----|
| Blocking syscall (`read`, `accept`) | Meta |
| Fork a subprocess | Meta (Port) |
| Serve HTTP and route to business logic | Meta (WebServer/Handler) + `act.WebWorker` or actor |
| In-memory message routing | Actor or Pool |
| Computation with external state | Actor |
| Per-connection protocol state | Meta (reader) + actor (handler via ProcessPool) |

## Anti-Patterns

- **Never originate `Call()` from a meta process** - the handle has no `Call`; a meta can only receive `Call` via `HandleCall`.
- **Never link/monitor from a meta process** - only actors can initiate; a meta can only receive links/monitors on its alias.
- **Do not forget `Done()`** for a raw `MessageWebRequest` - the HTTP goroutine blocks until `Done()` or timeout (`act.WebWorker` calls it for you).
- **Do not call Running-only handle methods from `Start()`** - `SendResponse` / `SendResponseError` / `SetCompression` return `gen.ErrNotAllowed` outside the handler goroutine.
- **Do not use `act.Pool` in `TCPServerOptions.ProcessPool`** - breaks per-connection routing.
- **Do not block inside `HandleMessage`** of the process receiving meta events - the actor-callback rule applies.
- **Do not share a `ReadBufferPool` []byte without recycling it** - when a pool is configured the consumer that receives the buffer owns returning it.
