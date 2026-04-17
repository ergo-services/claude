# Messages

Ergo distinguishes async (`Send`) and sync (`Call`) messaging, with optional **Important Delivery** semantics for cross-node reliability. All process methods are available on `gen.Process`.

## Send Family (async, no response)

| Method | Signature | Notes |
|--------|-----------|-------|
| `Send` | `Send(to any, message any) error` | Generic — `to` can be PID, ProcessID, Atom (name), or Alias |
| `SendPID` | `SendPID(to gen.PID, message any) error` | Typed, same-node shortcut |
| `SendProcessID` | `SendProcessID(to gen.ProcessID, message any) error` | Cross-node by `{Node, Name}` |
| `SendAlias` | `SendAlias(to gen.Alias, message any) error` | To alias |
| `SendWithPriority` | `SendWithPriority(to, message any, priority MessagePriority) error` | Override default priority |
| `SendImportant` | `SendImportant(to any, message any) error` | Confirmed delivery (see below) |
| `SendAfter` | `SendAfter(to, message any, after time.Duration) (CancelFunc, error)` | Delayed delivery |
| `SendWithPriorityAfter` | `SendWithPriorityAfter(to, message any, priority MessagePriority, after time.Duration) (CancelFunc, error)` | Delayed + priority |
| `SendEvent` | `SendEvent(name gen.Atom, token gen.Ref, message any) error` | Publish event |

## Call Family (sync, blocking)

| Method | Signature | Notes |
|--------|-----------|-------|
| `Call` | `Call(to, request any) (any, error)` | Default timeout (5s) |
| `CallWithTimeout` | `CallWithTimeout(to, request any, timeout int) (any, error)` | Timeout in seconds |
| `CallWithPriority` | `CallWithPriority(to, request any, priority MessagePriority) (any, error)` | Override priority |
| `CallImportant` | `CallImportant(to, request any) (any, error)` | Confirmed delivery (see below) |
| `CallPID` | `CallPID(to gen.PID, request any, timeout int) (any, error)` | Typed |
| `CallProcessID` | `CallProcessID(to gen.ProcessID, request any, timeout int) (any, error)` | Cross-node |
| `CallAlias` | `CallAlias(to gen.Alias, request any, timeout int) (any, error)` | To alias |

Caller enters `ProcessStateWaitResponse` until response arrives or timeout (`gen.ErrTimeout`).

## Response (inside HandleCall)

`HandleCall` can reply two ways:

1. **Synchronous return**: return `(response, nil)` from `HandleCall` — framework delivers automatically.
2. **Deferred**: return `(nil, nil)` and later call `SendResponse(from, ref, response)` with the stored `from` PID and `ref`. Use when reply depends on another async event.

| Method | Purpose |
|--------|---------|
| `SendResponse(to PID, ref Ref, msg any) error` | Normal response |
| `SendResponseImportant(to PID, ref Ref, msg any) error` | Confirmed response (for FR-2PC) |
| `SendResponseError(to PID, ref Ref, err error) error` | Error response |
| `SendResponseErrorImportant(to PID, ref Ref, err error) error` | Confirmed error response |

## Message Priority

```go
const (
    MessagePriorityNormal MessagePriority = 0  // Mailbox.Main (default)
    MessagePriorityHigh   MessagePriority = 1  // Mailbox.System
    MessagePriorityMax    MessagePriority = 2  // Mailbox.Urgent
)
```

Set per-process default: `ProcessOptions.SendPriority` or runtime `SetSendPriority(...)`.

## Important Delivery

With normal `Send`, remote failures are silent: if the target does not exist or its mailbox is full, the sender is not notified. Important Delivery fixes this by requiring an ACK from the target node.

**Enable per call:**
```go
err := p.SendImportant(targetPID, message)
err := p.CallImportant(targetPID, request)
```

**Enable for every message from the process** (`ProcessOptions.ImportantDelivery = true` or `SetImportantDelivery(true)`):

| Error | Meaning |
|-------|---------|
| `gen.ErrProcessUnknown` | Target process does not exist on remote node |
| `gen.ErrProcessMailboxFull` | Target mailbox saturated |
| `gen.ErrTimeout` | Remote received but ACK not delivered within 5s |
| `gen.ErrNoConnection` | Cannot reach remote node |

**Requires** `NetworkFlags.EnableImportantDelivery = true` on both sides.

## FR-2PC (Fully-Reliable Two-Phase Commit)

Both request and response use Important Delivery. Foundation for distributed transactions where both parties must confirm.

```go
// Caller
response, err := p.CallImportant(targetPID, TransactionRequest{...})

// Handler
func (h *Handler) HandleCall(from gen.PID, ref gen.Ref, req any) (any, error) {
    switch r := req.(type) {
    case TransactionRequest:
        result := h.commit(r)
        // Use SendResponseImportant explicitly if returning deferred.
        // Or set process-level important delivery so the framework picks the right path for the synchronous return.
        h.SetImportantDelivery(true)
        return TransactionResponse{Result: result}, nil
    }
    return nil, gen.ErrUnknown
}
```

## Compression

Per-process compression for outgoing messages:

```go
type Compression struct {
    Enable    bool
    Type      CompressionType   // GZIP (default), ZLIB, LZW
    Level     CompressionLevel  // Default, BestSpeed, BestSize
    Threshold int               // min bytes to trigger; default 1024
}
```

```go
// In ProcessOptions at spawn
opts := gen.ProcessOptions{
    Compression: gen.Compression{
        Enable:    true,
        Type:      gen.CompressionTypeGZIP,
        Threshold: 4096,
    },
}

// At runtime
p.SetCompression(true)
p.SetCompressionType(gen.CompressionTypeZLIB)
p.SetCompressionLevel(gen.CompressionBestSize)
p.SetCompressionThreshold(8192)
```

Compression applies only to **cross-node** messages. Same-node sends never compress.

## Process-Level Send Flags

| Method | Effect |
|--------|--------|
| `SetImportantDelivery(bool)` / `ImportantDelivery() bool` | All messages from this process use Important Delivery |
| `SetKeepNetworkOrder(bool)` / `KeepNetworkOrder() bool` | Preserve message order across network reconnects |
| `SetSendPriority(MessagePriority)` / `SendPriority() MessagePriority` | Default priority for `Send` |

## Mailbox Overflow (ProcessFallback)

When `ProcessOptions.MailboxSize > 0` and full, senders get `ErrProcessMailboxFull` by default. Configure a fallback process to receive overflow instead:

```go
opts := gen.ProcessOptions{
    MailboxSize: 1000,
    Fallback: gen.ProcessFallback{
        Enable: true,                 // required to activate forwarding
        Name:   "overflow-queue",
        Tag:    "worker-pool",
    },
}
```

Overflow arrives at `overflow-queue` as `gen.MessageFallback{PID, Tag, Target, Message}`.

## Exit Signals

```go
err := p.SendExit(targetPID, reason)          // request target to terminate
err := p.SendExitAfter(targetPID, reason, 5*time.Second)
err := p.SendExitMeta(alias, reason)          // to meta process
err := p.SendExitMetaAfter(alias, reason, 1*time.Second)
```

Target receives a `gen.MessageExitPID` if trapping exits; otherwise terminates with the given reason.

## Minimal Message Flow

```go
type MessageTick struct{ Time time.Time }
type GetStateRequest struct{}
type GetStateResponse struct{ Running bool }

// Fire-and-forget
a.Send(workerPID, MessageTick{Time: time.Now()})

// Sync call
resp, err := a.Call(workerPID, GetStateRequest{})
if err != nil {
    a.Log().Error("call failed: %v", err)
    return nil
}
state := resp.(GetStateResponse)
```

## Anti-Patterns

- **Never mix `Call` with cross-node without Important Delivery** — silent failures become timeouts with ambiguous causes.
- **Never compress small messages** — set `Threshold: 1024` or higher; below that the overhead dominates.
- **Never call `Send` from inside `HandleCall` to the caller** — use `SendResponse` with the ref.
- **Never send unexported-field structs across nodes** — EDF cannot serialize them; see `edf.md`.
- **Never use `time.After` or a timer in a callback to delay work** — use `SendAfter` and let the mailbox resume on message arrival.
