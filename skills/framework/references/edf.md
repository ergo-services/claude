# EDF Serialization

Ergo Data Format (EDF) is the native binary protocol for cross-node messages. Every type sent across the network must be registered with the node's network stack before a connection is formed. Same-node messages bypass EDF entirely, so an unregistered or unexported-field struct works locally and only fails once it crosses a node boundary.

Registration is per node, not per process. Types are collected during node startup, and during handshake each pair of nodes exchanges its registered set to build the encoding dictionaries. Registering after a connection is established has no effect on that connection.

## Registration API

Registration goes through `node.Network()`, which exposes the current registration methods:

| Method | Purpose |
|--------|---------|
| `RegisterType(v any) error` | Register one struct/type value (not a pointer) |
| `RegisterTypes(types []any) error` | Register a batch; order-agnostic |
| `RegisterError(e error) error` | Register one sentinel error for compact transport |
| `RegisterErrors(errs []error) error` | Register a batch of sentinel errors |
| `RegisterAtom(a gen.Atom) error` | Pre-cache one atom |
| `RegisterAtoms(atoms []gen.Atom) error` | Pre-cache a batch of atoms |

`RegisterTypes` resolves inter-type dependencies internally, so you may pass a `Person` that contains an `Address` field before or after `Address` in the same slice. Manual "nested types first" ordering is only relevant if you call `RegisterType` one type at a time.

```go
func (a *MyApp) Load(args ...any) (gen.ApplicationSpec, error) {
    err := a.Node().Network().RegisterTypes([]any{
        Person{},   // contains an Address field
        Address{},  // order does not matter for the batch API
    })
    if err != nil {
        return gen.ApplicationSpec{}, err
    }
    // ...
}
```

### Declarative registration (recommended)

The cleanest place to register an application's wire types is `ApplicationSpec.Network` (a `gen.ApplicationNetwork`). Entries there are processed during `ApplicationLoad`, before any process in the application is spawned, and are silently skipped when the node's network mode is `NetworkModeDisabled`.

```go
func (a *MyApp) Load(args ...any) (gen.ApplicationSpec, error) {
    return gen.ApplicationSpec{
        Name: "orders",
        Network: gen.ApplicationNetwork{
            RegisterTypes:  []any{Person{}, Address{}},
            RegisterErrors: []error{ErrInvalidOrder, ErrOrderNotFound},
            RegisterAtoms:  []gen.Atom{"production", "api-handler"},
        },
        Group: []gen.ApplicationMemberSpec{ /* ... */ },
    }, nil
}
```

`ApplicationNetwork` has three fields - `RegisterTypes []any`, `RegisterErrors []error`, `RegisterAtoms []gen.Atom` - all order-agnostic like the imperative batch calls.

### Migration note

The package-level `edf.RegisterTypeOf`, `edf.RegisterError`, and `edf.RegisterAtom` functions still exist but are deprecated: calling them from application code emits a one-time deprecation warning to stderr. Use `node.Network().RegisterType(s)` / `RegisterError(s)` / `RegisterAtom(s)` (or the declarative `ApplicationSpec.Network`) instead.

## Registration Rules

- Register **before** any connection is formed - typically from `ApplicationSpec.Network` or an application's `Load` callback.
- `RegisterType` accepts a value, **not** a pointer. `RegisterType(&Foo{})` is rejected with `pointer type is not supported`. Pointer **fields** inside a registered struct are fully supported (see below).
- Every field that crosses a node boundary must be **exported** (start with uppercase). A struct with an unexported field is rejected at registration with `struct <Name> has unexported field(s)`.
- Primitives (bool, the int/uint family, float32/64, string, `[]byte`) and framework types (`gen.Atom`, `gen.PID`, `gen.ProcessID`, `gen.Event`, `gen.Ref`, `gen.Alias`, `time.Time`, `time.Duration`) are built-in - do not re-register them. `time.Duration` is encoded as an int64.

## Excluding a Field

Tag an exported field with `edf:"-"` to keep it out of the wire format entirely. It is skipped on encode and left at its zero value on decode. Use this for local-only or non-serializable state on an otherwise-registered struct, without having to make the field unexported.

```go
type Job struct {
    ID     string
    Args   []string
    Result *Result `edf:"-"` // exported but excluded from the wire
}
```

The framework itself uses this: `gen.Error` carries a `Mailbox *ProcessMailbox` field tagged `edf:"-"` so a captured mailbox never travels across the network.

## Pointers

The value you pass to `RegisterType` must not itself be a pointer, but pointer **fields** inside a registered struct are encoded correctly:

```go
type Order struct {
    ID       int64
    Discount *int    // pointer field - supported, nil-safe
}
```

- A nil pointer encodes as a nil marker and decodes back to nil.
- Nested pointers (`**T`) are not supported (`nested pointer type is not supported`).
- Cyclic pointer graphs are bounded by `MaxDepth` (default 100). Exceeding it returns `ErrMaxDepthExceeded`.

## Custom Serialization (Optional)

For types where reflection-based marshaling is not enough (performance, custom framing), implement one of these interface pairs. `RegisterType` detects them and wires the custom encoder/decoder; otherwise it generates a reflection-based one.

```go
// EDF-native pair
type Marshaler interface {
    MarshalEDF(io.Writer) error
}
type Unmarshaler interface {
    UnmarshalEDF([]byte) error
}
```

Or the standard-library pair `encoding.BinaryMarshaler` / `encoding.BinaryUnmarshaler`.

`UnmarshalEDF` / `UnmarshalBinary` must be defined on a pointer receiver (`*T`). If only the marshaler half is present, registration returns an error telling you the unmarshaler must be a method of `*T`.

## Type Constraints

| Type | Maximum | Error on overflow |
|------|---------|-------------------|
| Atom | 255 bytes | `edf.ErrAtomTooLong` |
| String | 65535 bytes | `edf.ErrStringTooLong` |
| Error message | 32767 bytes | `edf.ErrErrorTooLong` |
| Binary (`[]byte`) | ~4 GB (2^32-1) | `edf.ErrBinaryTooLong` |
| Struct body (schema evolution on) | ~4 GB (2^32-1) | `edf.ErrStructTooLong` |
| Collections (slice, map) | 2^32 elements | - |

Messages exceeding the connection's `MaxMessageSize` (in `NetworkOptions`) are rejected with `gen.ErrTooLarge`, which is far below the theoretical 4 GB binary cap in any real deployment.

## Visibility Matrix

| Type | Fields | Serializable | Scope |
|------|--------|--------------|-------|
| unexported | unexported | No | Same node only, within the defining application |
| unexported | Exported | Yes | Same application, across nodes |
| Exported | Exported | Yes | Cross-application, cross-node |

**Design principle:** pick the tightest scope that works. Widening visibility later is cheap; narrowing it breaks callers.

## Error Registration

Registering an error lets it cross a node boundary as a short reference instead of a full string. Register the **sentinel markers** (values created with `errors.New`):

```go
var (
    ErrInvalidOrder   = errors.New("invalid order")
    ErrOrderNotFound  = errors.New("order not found")
    ErrDuplicateOrder = errors.New("duplicate order")
)

func (a *MyApp) Load(args ...any) (gen.ApplicationSpec, error) {
    return gen.ApplicationSpec{
        Name: "orders",
        Network: gen.ApplicationNetwork{
            RegisterErrors: []error{ErrInvalidOrder, ErrOrderNotFound, ErrDuplicateOrder},
        },
        // ...
    }, nil
}
```

An unregistered error still travels (as a string), just with a larger payload.

### Wrapped errors (`gen.Errorf` / `gen.Error`)

Do not register a `*gen.Error` value. `RegisterError` rejects it: register the `errors.New` markers, then build wrap chains at runtime with `gen.Errorf`:

```go
err := gen.Errorf("processing order %d: %w", id, ErrInvalidOrder)
```

`gen.Errorf` mirrors `fmt.Errorf` and stores the `%w` operands in a `*gen.Error` so `errors.Is` and `errors.Unwrap` keep working. The wrap chain survives a network hop - `errors.Is(err, ErrInvalidOrder)` still holds on the far node - but only when `NetworkFlags.EnableWrappedErrors` is on at **both** ends. It defaults to `true` (part of `gen.DefaultNetworkFlags`); if either side turns it off, the `*gen.Error` degrades to a flat `.Error()` string and the chain is lost.

## Pre-Registering Atoms

`Network().RegisterAtom(name)` (and `RegisterAtoms`) add an atom to the atom cache that nodes exchange during the connection handshake. A registered atom then crosses the network as a compact 2-byte id instead of its full string, starting with the first message that carries it. Register the atoms you send often across nodes: application names, process names, tag values, recurring message discriminators.

This is purely a wire-size optimization; correctness never depends on it. But there is no auto-caching - an unregistered atom is encoded in full every time it crosses a node, so registration is the only way to get the compact form. Register in `init()` (before the node starts), since a connection's atom cache is fixed at handshake time.

```go
Network: gen.ApplicationNetwork{
    RegisterAtoms: []gen.Atom{"orders", "production", "api-handler"},
},
```

## Schema Evolution

By default EDF is strict: a struct is its exact Go type, encoded positionally with every field always present. Adding or removing a field creates a new, incompatible type. That is deliberate change control, and it is the right default when every contract change must be a visible, compile-time decision.

Schema evolution relaxes this for **trailing** fields. Enable `NetworkFlags.EnableSchemaEvolution` on **both** nodes (it is negotiated at handshake; a connection where either side leaves it off stays strict). With it on, EDF length-prefixes each encoded struct body:

- A node that does not know an appended field **skips** it (new sender, old receiver).
- A node expecting a field the sender did not include reads it as its **zero value** (old sender, new receiver).

```go
// Start from the defaults so you do not turn off the other capabilities.
flags := gen.DefaultNetworkFlags
flags.EnableSchemaEvolution = true

node, err := ergo.StartNode("orders@host", gen.NodeOptions{
    Network: gen.NetworkOptions{Flags: flags},
})
```

Constraints to know:

- **Trailing fields only.** Shared leading fields must match in type and order; versions may differ only at the end. Inserting, reordering, retyping, or removing a non-trailing field is still a breaking change.
- **No mismatch detection.** Evolution tolerates a different trailing field count; it does not verify that the shared leading fields actually match. A mistaken reorder or type change on the leading fields silently misreads.
- **4 GB per-struct cap.** With evolution on, one encoded struct body is bounded at 2^32-1 bytes; overflow returns `edf.ErrStructTooLong`. The strict default has no such per-struct cap.

## Introspection

The registry is inspectable at runtime, useful for diagnostics and dynamic decode:

- `node.Network().RegisteredTypes()` returns `[]gen.RegisteredTypeInfo` - one entry per registered type carrying `Name`, `Kind` (bool, int, struct, marshaler, ...), `Schema` (Go-syntax shape), `MinSize`, `SizeVariable`, and usage `Stats`.
- `node.Network().LookupType(name)` resolves a name to a `reflect.Type`. It accepts the canonical form (`#pkgpath/Type`) or a short name (`Type`, matched by suffix).

## Compression

Compression is configured at the **process** or **message** level, not in EDF. See `messages.md` for the `Compression` struct and `SetCompression*` methods. EDF handles only type encoding.

## Anti-Patterns

- **Never register after a connection is formed** - types not in the registry at handshake time cannot be exchanged on that connection.
- **Never pass a pointer to `RegisterType`** - `RegisterType(&Foo{})` is rejected. Use `RegisterType(Foo{})`. Pointer *fields* inside the struct are fine.
- **Never send unexported-field structs cross-node** - registration rejects them, and same-node sends bypass EDF so the gap is invisible until the message crosses a node boundary.
- **Do not rely on same-node behavior to validate cross-node** - same-node skips EDF entirely; test the actual network path.
- **Do not register `*gen.Error`** - register `errors.New` sentinels and wrap with `gen.Errorf`.
- **Do not turn on schema evolution and treat it as free** - it only covers trailing fields, does not detect leading-field mismatches, and must be enabled on both nodes.
- **Do not send enormous `[]byte` values** - the 4 GB cap is theoretical; the connection's `MaxMessageSize` is the real limit.
