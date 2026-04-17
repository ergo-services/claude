# Erlang Protocol Interoperability

`ergo.services/proto/erlang23` implements the Erlang/OTP network stack (EPMD, ETF, DIST, handshake) for OTP 23+. It lets Ergo nodes join Erlang/Elixir clusters and exchange messages with Erlang processes using native protocols.

**Use only when:** you must interoperate with existing Erlang/OTP infrastructure. For Ergo-to-Ergo clusters keep EDF (default) — it is faster and supports Important Delivery and tracing, neither of which cross the Erlang boundary.

## Package Layout

```
ergo.services/proto/erlang23
├── epmd/         EPMD (Erlang Port Mapper Daemon) registrar — gen.Registrar implementation
├── etf/          ETF (External Term Format) codec and Erlang type wrappers (Tuple, List, Port, ...)
├── dist/         DIST (inter-node) protocol — gen.NetworkProto implementation
├── handshake/    Erlang handshake protocol — gen.NetworkHandshake implementation
└── gen_server.go erlang23.GenServer behavior — embed to act as an Erlang gen_server
```

## When To Use

| Scenario | Use |
|----------|-----|
| Ergo ↔ Ergo | EDF (default). Do NOT use Erlang proto. |
| Ergo must call an Erlang/Elixir gen_server | `erlang23` |
| Migration: ex-Erlang team maintains some OTP services | `erlang23` as bridge |
| Compatibility testing | `erlang23` |

What is **not** available over the Erlang protocol:
- Important Delivery (`SendImportant` / `CallImportant` return `gen.ErrUnsupported`).
- Tracing propagation.
- Proxy routing.
- Ergo-style atom cache invariants.

## Node Wiring

Three package-level `Create` functions produce the components that replace EDF:

```go
import (
    "ergo.services/ergo"
    "ergo.services/ergo/gen"
    "ergo.services/proto/erlang23/dist"
    "ergo.services/proto/erlang23/epmd"
    "ergo.services/proto/erlang23/handshake"
)

node, err := ergo.StartNode("ergo_node@host", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Cookie:    "erlangcookie",              // must match the Erlang cluster
        Proto:     dist.Create(dist.Options{}), // gen.NetworkProto
        Handshake: handshake.Create(handshake.Options{}), // gen.NetworkHandshake
        Registrar: epmd.Create(epmd.Options{}), // gen.Registrar replacing Ergo registrars
    },
})
```

After joining, the Ergo node appears in `:erlang.nodes()` on the Erlang side and can receive messages addressed to its registered process names.

## erlang23.GenServer Behavior

To act as an Erlang gen_server (so Erlang processes can `gen_server:call` / `gen_server:cast` into a Go process), embed `erlang23.GenServer` and implement `GenServerBehavior`:

```go
type GenServerBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) error
    HandleInfo(message any) error                                           // for plain sends
    HandleCast(message any) error                                           // for gen_server:cast
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error)         // for gen_server:call
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`GenServer` wraps incoming Erlang tuples (`{$gen_cast, Msg}`, `{$gen_call, From, Req}`, `{$gen_info, ...}`) and dispatches to the corresponding callback. A response returned from `HandleCall` is serialized back into the Erlang reply format.

## Sending to an Erlang gen_server

Send a cast via the embedded `GenServer.Cast` method — it wraps the message in `{'$gen_cast', Msg}` and sends it:

```go
err := gs.Cast(gen.ProcessID{Node: "erlang@host", Name: "my_gen_server"}, myRequest)
```

For a synchronous call, use the normal `Call` from the process interface — the DIST protocol handles the wrapping:

```go
resp, err := gs.Call(gen.ProcessID{Node: "erlang@host", Name: "my_gen_server"}, myRequest)
```

The request payload (`myRequest`) should be built from ETF-compatible Go types (see below) so the Erlang side sees a familiar term.

Ergo uses the standard `gen.ProcessID` / `gen.PID` types — there is no `erlang23.ProcessID`.

## ETF Types

Erlang-specific types live in the `etf` package:

```go
import "ergo.services/proto/erlang23/etf"

t := etf.Tuple{gen.Atom("update"), 42}
l := etf.List{1, 2, 3}
p := etf.Port{Node: "erlang@host", ID: 0x42, Creation: 0}
```

| Erlang | Go |
|--------|-----|
| atom | `gen.Atom` |
| integer | `int64` |
| float | `float64` |
| binary | `[]byte` |
| list | `etf.List` (also `[]any`) |
| improper list | `etf.ListImproper` |
| tuple | `etf.Tuple` |
| pid | `gen.PID` |
| reference | `gen.Ref` |
| port | `etf.Port` |
| map | `map[any]any` |
| string | ambiguous: `[]int` char-list or `[]byte` binary, depending on sender |

Strings in classic Erlang are char-lists of integers; modern OTP code usually sends binaries. Agree on binary encoding with the Erlang side to avoid accidental char-list roundtrips.

## EPMD

EPMD discovers Erlang nodes on a host by name. `epmd.Create()` registers the Ergo node with a local EPMD like a native Erlang node. The package also provides an embedded EPMD server (see `epmd/server.go`) — useful when no host-level EPMD is running.

## Cookie

Erlang clusters authenticate with a shared cookie. Set it on both sides:

- **Erlang side** — by convention read from `~/.erlang.cookie` or passed as `-setcookie cookie_value` at node start. Ergo does **not** read this file; it only uses `NetworkOptions.Cookie`.
- **Ergo side** — `NetworkOptions.Cookie` must match the value Erlang uses.

Mismatched cookies fail the handshake.

## Known Limitations

- **Ergo-exclusive features absent over Erlang protocol**: `SendImportant` / `CallImportant` return `gen.ErrUnsupported` (see `proto/erlang23/dist/connection.go`). Tracing spans and proxy routing do not cross the Erlang boundary.
- **`gen_server.Cast` does not exist as a package-level function** — it is a method on the embedded `erlang23.GenServer` struct. Likewise there is no package-level `gen_server.Call`; use `gs.Call(...)` from the process.
- **Types are in `etf`, not `erlang23`**: `etf.Tuple`, `etf.List`, `etf.Port`.
- **Embedded EPMD is local-host only.** For multi-host Erlang discovery configure static routes or run EPMD on each host.

## Anti-Patterns

- **Do not use `erlang23` for Ergo-to-Ergo** — you lose EDF features for no benefit.
- **Do not assume string/binary parity** — always agree on binary encoding explicitly.
- **Do not combine `epmd.Create(...)` with `etcd` or `saturn` Registrar** — pick one registrar per node.
- **Do not rely on Important Delivery for Erlang peers** — the dist layer refuses it.
- **Do not expose EPMD (default port 4369) to the internet** — it leaks node names and is unauthenticated.
