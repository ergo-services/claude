# Logging

Every process, meta process, and the node itself emit logs through one central system. A log call produces a `gen.MessageLog` that the node fans out to every registered logger whose level filter matches. Each logger decides what to do with it - write to stdout, append to a rotating file, forward errors to Sentry. This reference covers the developer-facing API (`gen.Log`), log levels, structured fields, per-process routing, the default logger, and how to write your own logger. The pre-built backend modules (colored, rotate, sentry) live in `integrations.md`.

## The gen.Log API

`process.Log()` (inside a callback, usually `a.Log()`) and `node.Log()` both return a `gen.Log`. This is the primary logging interface - every actor uses it.

### Level methods

| Method | Emits when source level is | Notes |
|--------|----------------------------|-------|
| `Trace(format string, args ...any)` | `<= LogLevelTrace` | Framework-internal verbosity; cannot be raised at runtime (set at startup only) |
| `Debug(format string, args ...any)` | `<= LogLevelDebug` | Development detail |
| `Info(format string, args ...any)` | `<= LogLevelInfo` | Normal operations (default level) |
| `Warning(format string, args ...any)` | `<= LogLevelWarning` | Notable but non-fatal |
| `Error(format string, args ...any)` | `<= LogLevelError` | Failed operation, node keeps running |
| `Panic(format string, args ...any)` | always, regardless of level | Logs at panic severity - does NOT trigger a Go panic |

Each method takes a `fmt`-style format string plus args. A message at level N is emitted only if the source's current level is `<= N` (lower numeric value = more verbose). `Panic` is special: it always logs and never panics. Ergo's `Log().Panic()` is unlike the standard library's `log.Panic()`, which does panic.

```go
func (a *Worker) HandleMessage(from gen.PID, message any) error {
    a.Log().Info("received %T from %s", message, from)
    if err := a.process(message); err != nil {
        a.Log().Error("processing failed: %v", err)
    }
    return nil
}
```

You normally never call `Panic` yourself. When an actor built on `act.Actor`, `act.Supervisor`, or `act.Pool` panics inside a callback, the framework recovers it and logs it at panic level for you. It only becomes relevant if you implement the raw `gen.ProcessBehavior` loop and recover panics yourself.

### Level get/set

| Method | Purpose |
|--------|---------|
| `Level() LogLevel` | Current level of this source |
| `SetLevel(level LogLevel) error` | Change the level; returns `gen.ErrIncorrect` for an invalid value |

```go
a.Log().SetLevel(gen.LogLevelDebug) // raise verbosity for this process
```

`LogLevelTrace` cannot be reached through `SetLevel` at runtime - trace is so verbose it must be enabled at startup via `NodeOptions.Log.Level` or `ProcessOptions.LogLevel` to avoid flooding storage by accident.

## Log levels

`gen.LogLevel` is an ordered integer enum. A source at level N emits every message whose level is `>= N`.

| Constant | Value | Meaning |
|----------|-------|---------|
| `LogLevelSystem` | -100 | Framework-internal; part of `gen.DefaultLogLevels`, not `DefaultLogFilter` |
| `LogLevelTrace` | -2 | Most verbose; startup-only |
| `LogLevelDebug` | -1 | Debug detail |
| `LogLevelDefault` | 0 | Sentinel: inherit from node / application / parent process |
| `LogLevelInfo` | 1 | Default effective level |
| `LogLevelWarning` | 2 | |
| `LogLevelError` | 3 | |
| `LogLevelPanic` | 4 | Highest severity |
| `LogLevelDisabled` | 5 | Silences the source - the framework does not even build the message |

`LogLevelDefault` (0) is the inherit sentinel. `ProcessOptions.LogLevel` defaults to it, so a freshly spawned process inherits its parent's / node's effective level. A node configured with `LogLevelDefault` resolves to `LogLevelInfo`. `LogLevelDisabled` fully mutes a source without touching the logger set.

Two package-level slices are referenced by the framework:

- `gen.DefaultLogFilter` - Trace through Panic; the filter applied to a logger registered with no explicit filter.
- `gen.DefaultLogLevels` - System plus Trace through Panic.

## Structured fields

Attach key-value context that rides along with every subsequent message from the source. The unit is `gen.LogField`:

```go
type LogField struct {
    Name  string
    Value any
}
```

| Method | Purpose |
|--------|---------|
| `Fields() []LogField` | Current field set |
| `AddFields(fields ...LogField)` | Accumulate fields (does not replace existing ones) |
| `DeleteFields(names ...string)` | Remove fields by name |

```go
func (a *OrderProcessor) HandleMessage(from gen.PID, message any) error {
    order := message.(Order)

    a.Log().AddFields(
        gen.LogField{Name: "order_id", Value: order.ID},
        gen.LogField{Name: "customer_id", Value: order.CustomerID},
    )

    a.Log().Info("processing order")   // both fields attached
    a.Log().Debug("validating payment")

    a.Log().DeleteFields("customer_id") // drop one field
    return nil
}
```

Fields accumulate across calls, so you build context incrementally (session id at login, transaction id when a transaction opens, and so on). They are always tracked internally but only appear in output when the logger is told to render them - for the default logger set `DefaultLoggerOptions.IncludeFields = true`.

### Scoped fields (push / pop)

For fields that should exist only for the duration of one operation, use a scope instead of manual `DeleteFields`.

| Method | Purpose |
|--------|---------|
| `PushFields() int` | Save the current field set, open a new scope; returns the new stack depth |
| `PopFields() int` | Restore the previous field set; returns the new depth (no-op when nothing was pushed) |

```go
a.Log().AddFields(gen.LogField{Name: "session_id", Value: "abc123"})

a.Log().PushFields()
a.Log().AddFields(gen.LogField{Name: "operation", Value: "payment"})
a.Log().Info("processing payment")  // session_id + operation
a.Log().PopFields()

a.Log().Info("payment complete")    // session_id only
```

Scopes nest - each `PushFields` returns a deeper level, each `PopFields` unwinds one. While the stack has any active frame you cannot delete fields; pop back to the base level first. This keeps the field state consistent so a pending pop never resurrects a field you meant to drop.

## Per-process log routing (fan-out and SetLogger)

By default a log message fans out to every registered non-hidden logger, each filtering by its own level set. A source can opt out of fan-out and pin its logs to one named logger.

| Method | Purpose |
|--------|---------|
| `Logger() string` | Name of the pinned logger; empty means fan-out (default) |
| `SetLogger(name string)` | Send this source's logs ONLY to the named logger; `SetLogger("")` restores fan-out |

The named logger must already be registered via `node.LoggerAdd` / `node.LoggerAddPID`.

### Hidden loggers

A logger whose name begins with `"."` is a hidden logger. It is excluded from fan-out and receives logs only from sources that explicitly `SetLogger` to it. This gives bidirectional isolation: the source's logs go nowhere else, and the hidden logger sees nothing but that source. It is the mechanism for peeling a noisy process off into its own stream.

```go
// Register a dedicated file logger, hidden from fan-out
tradingLog, _ := rotate.CreateLogger(rotate.Options{Path: "/var/log/trading"})
node.LoggerAdd(".trading", tradingLog)

// Pin the noisy trading actor to it (typically from the actor's Init)
a.Log().SetLogger(".trading")
```

Now the trading actor's logs land only in `.trading`, and `.trading` never mixes in anyone else's output. Using a non-dotted name would still isolate the source's output to that logger, but the logger would also keep receiving fan-out from every other process.

## The default logger

Every node starts with a built-in logger that writes to `os.Stdout`. Configure it through `NodeOptions.Log.DefaultLogger` (a `gen.DefaultLoggerOptions`).

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `Disable` | `bool` | `false` | Turn the default logger off entirely (use when only custom loggers are wanted) |
| `TimeFormat` | `string` | `""` | Go time layout (e.g. `time.RFC3339`); empty renders nanoseconds since epoch |
| `IncludeBehavior` | `bool` | `false` | Append the behavior type name to process/meta/app logs |
| `IncludeName` | `bool` | `false` | Append the registered process name (when set) |
| `IncludeFields` | `bool` | `false` | Render structured fields in the output |
| `Filter` | `[]LogLevel` | `nil` | Restrict output levels (empty = all); applied by the node at registration, not inside the logger |
| `Output` | `io.Writer` | `os.Stdout` | Any writer - file, buffer, `net.Conn`, etc. |
| `EnableJSON` | `bool` | `false` | Emit one single-line JSON object per message (for Elasticsearch, Splunk, and similar) |
| `DisableBanner` | `bool` | `false` | Suppress the Ergo logo printed on start |

```go
node, _ := ergo.StartNode("app@localhost", gen.NodeOptions{
    Log: gen.LogOptions{
        Level: gen.LogLevelInfo,
        DefaultLogger: gen.DefaultLoggerOptions{
            TimeFormat:    time.RFC3339,
            IncludeName:   true,
            IncludeFields: true,
            EnableJSON:    true,
        },
    },
})
```

`gen.CreateDefaultLogger(DefaultLoggerOptions) gen.LoggerBehavior` builds the same logger directly - useful when you want a second instance (for example a JSON copy pointed at a file) registered alongside the built-in one via `node.LoggerAdd`.

`NodeOptions.Log` is a `gen.LogOptions`: `Level` (node default level), `DefaultLogger`, and `Loggers []gen.Logger` for custom loggers registered on start. Each `gen.Logger` carries `Name`, `Logger` (a `LoggerBehavior`), and an optional `Filter []LogLevel`.

## Writing a custom logger

A logger implements `gen.LoggerBehavior`, exactly two methods:

```go
type LoggerBehavior interface {
    Log(message MessageLog)
    Terminate()
}
```

`Log` is called for every message that matches the logger's filter. It must not block - if the work is expensive (network, database, compression) queue the message and return immediately, draining on a background goroutine. `Terminate` runs on removal or node shutdown; use it to flush and close. Node shutdown waits for it, so keep it quick.

Every message arrives as a `gen.MessageLog`:

| Field | Type | Purpose |
|-------|------|---------|
| `Time` | `time.Time` | When the log was produced |
| `Level` | `gen.LogLevel` | Severity |
| `Source` | `any` | One of the five source variants below |
| `Format` | `string` | Format string (unformatted) |
| `Args` | `[]any` | Format arguments |
| `Fields` | `[]LogField` | Structured fields attached at the source |
| `StackTrace` | `[]string` | Captured frames, present for recovered panics |

`Source` is a typed value telling you what emitted the message. Switch on it:

| Source variant | Fields |
|----------------|--------|
| `MessageLogNode` | `Node Atom`, `Creation int64` |
| `MessageLogNetwork` | `Node Atom`, `Peer Atom`, `Creation int64` |
| `MessageLogProcess` | `Node Atom`, `PID PID`, `Name Atom`, `Behavior string` |
| `MessageLogMeta` | `Node Atom`, `Parent PID`, `Meta Alias`, `Behavior string` |
| `MessageLogApplication` | `Node Atom`, `Name Atom`, `Mode ApplicationMode`, `Behavior string` |

```go
type stderrLogger struct{}

func (l *stderrLogger) Log(m gen.MessageLog) {
    msg := m.Format
    if len(m.Args) > 0 {
        msg = fmt.Sprintf(m.Format, m.Args...)
    }
    switch src := m.Source.(type) {
    case gen.MessageLogProcess:
        fmt.Fprintf(os.Stderr, "[%s] %s: %s\n", m.Level, src.PID, msg)
    case gen.MessageLogNode:
        fmt.Fprintf(os.Stderr, "[%s] %s: %s\n", m.Level, src.Node, msg)
    default:
        fmt.Fprintf(os.Stderr, "[%s] %s\n", m.Level, msg)
    }
}

func (l *stderrLogger) Terminate() {}
```

Register it:

```go
// Trace..Panic when no filter is passed (gen.DefaultLogFilter)
err := node.LoggerAdd("stderr", &stderrLogger{})

// or restrict the levels this logger sees
err = node.LoggerAdd("errors", &stderrLogger{}, gen.LogLevelError, gen.LogLevelPanic)
```

### Process-based loggers

An actor can also act as a logger: implement the `HandleLog(message gen.MessageLog) error` callback (available on `act.Actor`) and register the process with `node.LoggerAddPID`. Log delivery to a process is queued through its mailbox, so the emitting code never blocks. A logger process terminating is auto-removed from the logging system - no explicit `LoggerDeletePID` needed.

```go
func (l *AuditLogger) HandleLog(message gen.MessageLog) error {
    switch src := message.Source.(type) {
    case gen.MessageLogProcess:
        l.record(src.PID, message)
    }
    return nil
}
```

## Node-side logger management (runtime)

`gen.Node` exposes methods to add, remove, and retune loggers and per-source levels on a running node - no restart required. All are on the node object (`process.Node()` or the value returned by `ergo.StartNode`).

| Method | Purpose |
|--------|---------|
| `LoggerAdd(name string, logger LoggerBehavior, filter ...LogLevel) error` | Register a logger; `gen.ErrTaken` if the name is in use, `gen.ErrNodeTerminated` outside Running |
| `LoggerAddPID(pid PID, name string, filter ...LogLevel) error` | Register a process as a logger |
| `LoggerDelete(name string)` | Remove a logger; calls its `Terminate()` |
| `LoggerDeletePID(pid PID)` | Remove a process logger |
| `Loggers() []string` | Names of all registered loggers |
| `LoggerLevels(name string) []LogLevel` | Levels a named logger is capturing |
| `SetProcessLogLevel(pid PID, level LogLevel) error` | Retune one process's level; `gen.ErrNodeTerminated` outside Running |
| `SetMetaLogLevel(meta Alias, level LogLevel) error` | Retune one meta process's level |

Names must be unique - reusing one returns `gen.ErrTaken`; delete first, then re-add. Omit the filter and the logger captures `gen.DefaultLogFilter` (Trace through Panic).

```go
// Temporarily raise verbosity on a suspect process, then restore
node.SetProcessLogLevel(suspiciousPID, gen.LogLevelDebug)
// ... investigate ...
node.SetProcessLogLevel(suspiciousPID, gen.LogLevelInfo)
```

## Common patterns

- Production: disable the default logger, add a rotating file logger plus (optionally) a Sentry logger for errors. See `integrations.md`.
- Development: a colored console logger (with the default logger disabled to avoid duplicate output). See `integrations.md`.
- Isolate a noisy actor into its own file with a hidden logger and `SetLogger(".name")`.
- Correlate a request across many log lines with `AddFields`, and scope per-operation context with `PushFields` / `PopFields`.

## Anti-patterns

- Never block inside `LoggerBehavior.Log` - queue expensive work and return. It runs on the logging path for synchronous (non-process) loggers.
- Do not disable the default logger without registering a replacement - the node then produces no log output.
- Do not rely on trace being enable-able at runtime - set it at startup or not at all.
- Do not expect fields in output without `IncludeFields` (default logger) or the equivalent option on a backend logger; they are tracked but not rendered.
