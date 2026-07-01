# Erlang Protocol Interoperability

`ergo.services/proto/erlang23` implements the Erlang/OTP network stack (EPMD, ETF, DIST, handshake) so an Ergo node can join an Erlang/Elixir cluster and exchange messages with Erlang processes using their native protocols. The handshake speaks version 6 by default (OTP 23 and later) and can drop to version 5 for OTP 22 and earlier.

**Use only when** you must interoperate with existing Erlang/OTP infrastructure. For Ergo-to-Ergo clusters keep EDF (the default) - it is faster and supports Important Delivery and tracing, neither of which cross the Erlang boundary.

The module is distributed under the Business Source License 1.1 and needs a license for production or commercial use.

## Package Layout

```
ergo.services/proto/erlang23
├── epmd/         EPMD (Erlang Port Mapper Daemon) - gen.Registrar implementation
├── etf/          ETF (External Term Format) codec plus Erlang type wrappers
├── dist/         DIST (inter-node) protocol - gen.NetworkProto implementation
├── handshake/    Erlang handshake - gen.NetworkHandshake implementation
└── gen_server.go erlang23.GenServer behavior - embed to act as an Erlang gen_server
```

## When To Use

| Scenario | Use |
|----------|-----|
| Ergo to Ergo | EDF (default). Do NOT use the Erlang proto. |
| Ergo must call an Erlang/Elixir gen_server | `erlang23` |
| Migration: an ex-Erlang team maintains some OTP services | `erlang23` as a bridge |
| Compatibility testing | `erlang23` |

What is **not** available over the Erlang protocol:
- Important Delivery. `SendImportant` / `CallImportant` return `gen.ErrUnsupported`.
- Tracing propagation.
- Proxy routing.
- Ergo-style atom cache invariants.

## Node Wiring

Three package-level `Create` functions produce the components that replace the default EDF stack in `NetworkOptions`:

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
        Cookie:    "erlangcookie",                    // must match the Erlang cluster
        Proto:     dist.Create(dist.Options{}),        // gen.NetworkProto
        Handshake: handshake.Create(handshake.Options{}), // gen.NetworkHandshake
        Registrar: epmd.Create(epmd.Options{}),        // gen.Registrar replacing Ergo registrars
    },
})
```

After joining, the Ergo node appears in `erlang:nodes()` on the Erlang side and can receive messages addressed to its registered process names. Ergo uses the standard `gen.ProcessID` / `gen.PID` types - there is no `erlang23.ProcessID`.

### handshake.Options

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `Flags` | `[]erlang23.Flag` | `handshake.DefaultFlags()` when empty | Distribution flags negotiated at handshake |
| `UseVersion5` | `bool` | `false` (version 6) | Set `true` to connect to Erlang/OTP 22 and earlier (handshake version 5) |

Leaving `Flags` empty applies the full default set. To tune it, start from the defaults and adjust:

```go
flags := handshake.DefaultFlags()
// add or drop individual erlang23.Flag values, then:
handshake.Create(handshake.Options{Flags: flags, UseVersion5: false})
```

`handshake.NetworkFlags()` derives the node's `EnableRemoteSpawn` from whether `erlang23.FlagSpawn` is present, so remote spawn over the Erlang connection is on only when that flag is enabled (it is part of `DefaultFlags()`).

### epmd.Options

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `Port` | `uint16` | `4369` when `0` | EPMD discovery port |
| `EnableRouteTLS` | `bool` | `false` | Mark resolved routes `TLS`, so DIST connections to peers are established over TLS. Set this when the Erlang cluster uses TLS |
| `DisableServer` | `bool` | `false` | Run purely as an EPMD client; never start the embedded server |

By default the node tries to start an embedded EPMD server. If `DisableServer` is set, or the port is already bound (a host EPMD is already running), it falls back to acting as an EPMD client against that server.

### dist.Options

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `FragmentationUnit` | `int` | `65000` | Fragment size in bytes for large messages. Values below the default are raised to it |

## erlang23.GenServer Behavior

To act as an Erlang gen_server (so Erlang processes can `gen_server:call` / `gen_server:cast` into a Go process), embed `erlang23.GenServer` and implement `GenServerBehavior`. All callbacks are optional - defaults log an "unhandled" warning.

```go
type GenServerBehavior interface {
    gen.ProcessBehavior

    Init(args ...any) error
    HandleInfo(message any) error                                   // plain erlang:send / Send*
    HandleCast(message any) error                                   // gen_server:cast
    HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) // gen_server:call
    Terminate(reason error)
    HandleEvent(message gen.MessageEvent) error
    HandleInspect(from gen.PID, item ...string) map[string]string
}
```

`GenServer` unwraps the incoming Erlang tuples (`{'$gen_cast', Msg}`, `{'$gen_call', From, Req}`) and dispatches to the matching callback. A non-nil value returned from `HandleCall` is sent back as the Erlang reply; returning `nil` defers the reply so it can be sent later with `SendResponse`.

An actor:

```go
import "ergo.services/proto/erlang23"

func factory_MyActor() gen.ProcessBehavior {
    return &MyActor{}
}

type MyActor struct {
    erlang23.GenServer
}

func (a *MyActor) HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) {
    return etf.Tuple{gen.Atom("ok"), request}, nil
}
```

If a process only needs plain messages from Erlang, a normal `act.Actor` with `HandleMessage` is enough; `erlang23.GenServer` is only required for `gen_server:cast` / `gen_server:call` semantics.

### Trapping exits

`erlang23.GenServer` supports the same exit-trapping as `act.Actor`:

| Method | Purpose |
|--------|---------|
| `SetTrapExit(trap bool)` | Enable/disable exit trapping |
| `TrapExit() bool` | Report the current setting |

With trapping on, an incoming exit signal from a non-parent process is redelivered to `HandleInfo` as an ordinary message instead of terminating the process. The exception is the **parent** process: a parent exit still terminates the GenServer even with trapping enabled.

## Sending to an Erlang gen_server

Send a cast via the embedded `Cast` method - it wraps the message in `{'$gen_cast', Msg}` and sends it:

```go
err := gs.Cast(gen.ProcessID{Node: "erlang@host", Name: "my_gen_server"}, myRequest)
```

For a synchronous call, use the normal `Call` from the process interface - the DIST protocol handles the `gen_server:call` wrapping:

```go
resp, err := gs.Call(gen.ProcessID{Node: "erlang@host", Name: "my_gen_server"}, myRequest)
```

Build `myRequest` from ETF-compatible Go types (see below) so the Erlang side sees a familiar term.

## ETF Types

Erlang-specific wrapper types live in the `etf` package:

```go
import "ergo.services/proto/erlang23/etf"

t := etf.Tuple{gen.Atom("update"), 42}
l := etf.List{1, 2, 3}
p := etf.Port{Node: "erlang@host", ID: 0x42, Creation: 0}
```

| Erlang term | Go (decoded) |
|-------------|--------------|
| atom | `gen.Atom` |
| atom `true`/`false` | `bool` |
| integer | `int64` |
| big integer | `*big.Int`, or `int64`/`uint64` when it fits |
| float | `float64` |
| binary | `[]byte` |
| list (`LIST_EXT`) | `etf.List` (also `[]any`); a char-list of integers converts with `etf.TermToString` |
| improper list | `etf.ListImproper` |
| tuple | `etf.Tuple`, or a registered struct type |
| string (compact `STRING_EXT`) | Go `string` directly - no conversion needed |
| map | `map[any]any` |
| pid | `gen.PID` |
| reference | `gen.Ref` |
| reference (alias) | `gen.Alias` |
| port | `etf.Port` |

### Strings: char-list vs binary

Go and Erlang disagree on what a "string" is, so text needs care on the wire:

| Go value (encoded) | Erlang term |
|--------------------|-------------|
| `string` | char-list `"..."` (a list of integers) |
| `etf.String` | binary `<<...>>` |
| `etf.Charlist` | UTF-8 char-list `[...]` (a list of runes) |

A bare Go `string` is sent as an Erlang **char-list**, not a binary. When the Erlang side expects a binary (modern OTP and Elixir code usually does), wrap outgoing text in `etf.String`:

```go
gs.Send(to, etf.String("hello"))   // Erlang sees <<"hello">>
gs.Send(to, "hello")               // Erlang sees "hello" (char-list of integers)
gs.Send(to, etf.Charlist("héllo")) // Erlang sees a UTF-8 list of runes
```

Agree with the Erlang side which form it expects, and pick the matching wrapper.

## ETF Codec and Struct Helpers

The `etf` package exposes the codec and reflection helpers you use to build and parse Erlang terms.

| Function / type | Purpose |
|-----------------|---------|
| `Encode(term any, b *lib.Buffer, opts EncodeOptions) error` | Encode a Go term to ETF bytes |
| `Decode(packet []byte, cache []gen.Atom, opts DecodeOptions) (term any, tail []byte, err error)` | Decode ETF bytes to a Go term |
| `RegisterTypeOf(t any, opts RegisterTypeOptions) (gen.Atom, error)` | Map a named Go struct/slice/array/map for wire encoding and auto-decoding |
| `TermIntoStruct(term, dest any) error` | Decode a received `etf.List` / `etf.Tuple` / `etf.Map` into a Go struct, map, slice, or array |
| `TermProplistIntoStruct(term, dest any) error` | Decode an Erlang proplist (list of `{Name, Value}` tuples, or `[]etf.ProplistElement`) into a struct |
| `TermToString(t any) (string, bool)` | Convert an atom, `[]byte`, or char-list `etf.List` to a Go string |
| `Marshaler` / `Unmarshaler` | Custom (un)marshalling: `MarshalETF() ([]byte, error)` and `UnmarshalETF([]byte) error` |

`RegisterTypeOptions` has two fields:

| Field | Purpose |
|-------|---------|
| `Name` (`gen.Atom`) | Wire name for the type. Empty means auto-generated as `#pkgpath/Type` |
| `Strict` (`bool`) | When `true`, decoding panics (returned as an error) if the incoming data does not fit the destination |

A registered struct is encoded as an Erlang **tuple** whose first element is the registered name atom, followed by the field values in order. Register once at startup, then Erlang can address the type by that atom:

```go
type MyValue struct {
    MyString string
    MyInt    int32
}

etf.RegisterTypeOf(MyValue{}, etf.RegisterTypeOptions{Name: "myvalue", Strict: true})
```

```erlang
%% Erlang side sends a tuple tagged with the registered name:
erlang:send(Pid, {myvalue, "hello", 123}).
```

The Erlang DIST proto deliberately does **not** implement `gen.TypeRegistry` - ETF carries atoms, tuples, lists, and binaries directly, with no separate type-registration handshake. In a multi-proto node, `node.Network().RegisterType(...)` skips the Erlang proto; use `etf.RegisterTypeOf` to teach the Erlang decoder your Go types.

`TermIntoStruct` and `TermProplistIntoStruct` honor `etf:"<name>"` struct tags when matching Erlang keys to Go fields. Both use reflection and are relatively expensive - prefer manual type casting on hot paths.

## Distribution Flags

`erlang23.Flag` constants select which distribution features the handshake advertises. `handshake.DefaultFlags()` returns the recommended set; pass a custom `[]erlang23.Flag` in `handshake.Options.Flags` to change it.

Commonly relevant flags:

| Flag | Effect |
|------|--------|
| `erlang23.FlagSpawn` | Enables remote spawn over the connection; drives `NetworkFlags.EnableRemoteSpawn` |
| `erlang23.FlagAlias` | Process aliases |
| `erlang23.FlagFragments` | Large-message fragmentation |
| `erlang23.FlagBigCreation` | Big node-creation tags (newer pid/ref formats) |
| `erlang23.FlagHandshake23` | Version 6 connection setup (OTP 23+) |

See godoc for the full constant list. The `erlang23.Flags` bitset offers `IsEnabled`, `Enable`, and `Disable` helpers if you build a set programmatically.

## Cookie

Erlang clusters authenticate with a shared cookie. Set it on both sides:

- **Erlang side** - by convention read from `~/.erlang.cookie` or passed as `-setcookie cookie_value` at node start. Ergo does **not** read that file; it uses `NetworkOptions.Cookie` only.
- **Ergo side** - `NetworkOptions.Cookie` must match the value Erlang uses.

Mismatched cookies fail the handshake.

## Mixed Stacks (Ergo and Erlang peers together)

To keep accepting Ergo (EDF) connections while running the Erlang stack as the default, configure explicit acceptors in `NetworkOptions.Acceptors` - one per stack. An empty `gen.AcceptorOptions{}` inherits the stack from `NetworkOptions`; a fully specified one names its own `Registrar`, `Handshake`, and `Proto`. Outgoing connections on the non-default stack are then reached via a static route (`node.Network().AddRoute(match, route, weight)`) whose `Resolver` is that stack's registrar resolver. See `cluster.md` and the framework static-routes docs for the routing details.

## Known Limitations

- **Ergo-exclusive features absent over the Erlang protocol**: `SendImportant` / `CallImportant` return `gen.ErrUnsupported`. Tracing spans and proxy routing do not cross the Erlang boundary.
- **`Cast` is a method, not a package function** - it lives on the embedded `erlang23.GenServer`. There is no package-level `gen_server.Call` either; use `gs.Call(...)` from the process interface.
- **Wrapper types are in `etf`, not `erlang23`**: `etf.Tuple`, `etf.List`, `etf.Port`, `etf.String`, `etf.Charlist`.
- **Embedded EPMD is local-host only.** For multi-host Erlang discovery run an EPMD on each host or use static routes.

## Anti-Patterns

- **Do not use `erlang23` for Ergo-to-Ergo** - you lose EDF features for no benefit.
- **Do not send a bare Go `string` when Erlang expects a binary** - it arrives as a char-list. Wrap it in `etf.String`.
- **Do not combine `epmd.Create(...)` with an `etcd` or `saturn` registrar on the same stack** - pick one registrar per stack.
- **Do not rely on Important Delivery for Erlang peers** - the DIST layer refuses it with `gen.ErrUnsupported`.
- **Do not expose EPMD (default port 4369) to the internet** - it leaks node names and is unauthenticated.
- **Do not forget `UseVersion5` for old peers** - OTP 22 and earlier reject the version 6 handshake.
