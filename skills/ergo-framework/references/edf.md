# EDF Serialization

Ergo Data Format (EDF) is the native binary protocol for cross-node messages. Every type sent across the network must be registered in EDF before the node starts. Same-node messages bypass EDF entirely.

## Core API

```go
import "ergo.services/ergo/net/edf"

func RegisterTypeOf(v any) error   // register a struct value (not a pointer)
func RegisterError(e error) error  // register an error value for compact transport
func RegisterAtom(a gen.Atom) error // pre-cache an atom
```

## Registration Rules

- Registration must happen in `init()`, **before** the node starts.
- Register nested types **before** their parents.
- `RegisterTypeOf` accepts a value — **not** a pointer. Pointer types are rejected: `pointer type is not supported`.
- All fields crossing a node boundary must be **Exported** (start with uppercase).
- Primitives (bool, int, string, float, []byte) and framework types (Atom, PID, ProcessID, Event, Ref, Alias, time.Time) are built-in — do not re-register.

## Custom Serialization (Optional)

For types where default reflection-based marshaling is insufficient (performance, custom framing), implement:

```go
type Marshaler interface {
    MarshalEDF(io.Writer) error
}

type Unmarshaler interface {
    UnmarshalEDF([]byte) error
}
```

Or use the standard library pair:

```go
encoding.BinaryMarshaler
encoding.BinaryUnmarshaler
```

`RegisterTypeOf` detects these interfaces and wires them; otherwise it generates a reflection-based encoder/decoder.

## Type Constraints

| Type | Maximum |
|------|---------|
| Atom | 255 bytes |
| String | 65535 bytes |
| Error | 32767 bytes |
| Binary (`[]byte`) | 4 GB |
| Collections (slice, map) | 2³² elements |

Messages exceeding network `MaxMessageSize` (in `NetworkOptions`) are rejected with `ErrTooLarge`.

## Visibility Matrix

| Type | Fields | Serializable | Scope |
|------|--------|--------------|-------|
| unexported | unexported | No | Same node only, within the defining application |
| unexported | Exported | Yes | Same application, across nodes |
| Exported | Exported | Yes | Cross-application, cross-node |

**Design principle:** pick the tightest scope that works. Widening visibility later is cheap; narrowing it breaks callers.

## Registration Pattern

```go
package messaging

import (
    "ergo.services/ergo/net/edf"
)

type Address struct {
    City   string
    Street string
}

type Person struct {
    Name    string
    Address Address  // nested struct
}

// Register in init() — nested types first
func init() {
    if err := edf.RegisterTypeOf(Address{}); err != nil {
        panic(err)
    }
    if err := edf.RegisterTypeOf(Person{}); err != nil {
        panic(err)
    }

    // Errors for compact transport
    if err := edf.RegisterError(ErrInvalidOrder); err != nil {
        panic(err)
    }
}

var ErrInvalidOrder = errors.New("invalid order")
```

## Do Not Send Across Nodes

```go
// WRONG: unexported field
type BadMessage struct {
    Name string
    data []byte  // cannot serialize
}

// WRONG: pointer type
var msg *Person
edf.RegisterTypeOf(msg)  // error: pointer type is not supported

// CORRECT
edf.RegisterTypeOf(Person{})
```

## Pre-Caching Atoms

Registration with `RegisterAtom` inserts the atom into the atom cache before first use. Reduces handshake-time discovery latency for atoms known in advance (application names, process names, tag values).

```go
edf.RegisterAtom("orders")
edf.RegisterAtom("production")
edf.RegisterAtom("api-handler")
```

Not required for correctness — atoms are auto-cached on first encounter — but helpful for predictable startup cost.

## Error Registration

Register errors for compact cross-node transport (errors become short references rather than full strings):

```go
var (
    ErrInvalidOrder   = errors.New("invalid order")
    ErrOrderNotFound  = errors.New("order not found")
    ErrDuplicateOrder = errors.New("duplicate order")
)

func init() {
    edf.RegisterError(ErrInvalidOrder)
    edf.RegisterError(ErrOrderNotFound)
    edf.RegisterError(ErrDuplicateOrder)
}
```

Unregistered errors still travel across nodes (as strings), but with larger payload.

## Compression

Compression is configured at the **process** or **message** level, not EDF. See `messages.md` for `Compression` struct and `SetCompression*` methods. EDF handles only type encoding.

## Anti-Patterns

- **Never register after node start** — types not in the registry at handshake time cannot be exchanged.
- **Never register a pointer type** — `RegisterTypeOf(&Foo{})` returns an error. Use `RegisterTypeOf(Foo{})`.
- **Never send unexported-field structs cross-node** — they serialize to zero values silently (data loss).
- **Do not forget nested type registration** — a `Person` with `Address Address` needs `Address` registered first.
- **Do not rely on same-node struct sharing to mean cross-node works** — same-node skips EDF entirely, so unexported fields appear to work locally but fail remotely.
- **Do not send enormous `[]byte` values** — the 4GB limit is a theoretical cap; typical `MaxMessageSize` is much lower.
