---
name: ergo-framework
description: Build distributed actor-based systems with Ergo Framework in Go. Covers actor lifecycle, supervision, message patterns, meta processes for I/O, cluster configuration, and EDF serialization. Use when implementing Ergo applications or working with actor-based distributed systems.
---

# Ergo Framework

Build distributed, fault-tolerant actor-based systems in Go with Erlang-like reliability. Ergo Framework provides actor model, supervision trees, network transparency, and cluster support.

## When to Use This Skill

- Implementing actors with Ergo Framework
- Designing supervision trees for fault tolerance
- Building distributed systems with cluster support
- Integrating TCP/UDP/HTTP with actor model via meta processes
- Configuring service discovery with etcd or Saturn
- Implementing message patterns (Send, Call, Important Delivery)
- Serializing messages with EDF for cross-node communication

## Core Concepts

### 1. Actor Model

**Process States:**
- ProcessStateInit - initializing, spawning children
- ProcessStateSleep - idle, waiting for messages
- ProcessStateRunning - handling messages
- ProcessStateTerminated - shutting down

**Mailbox Queues (priority order):**
- Urgent (MessagePriorityMax) - processed first
- System (MessagePriorityHigh) - framework messages
- Main (MessagePriorityNormal) - application messages (default)
- Log - logging messages

**Critical Rule:** NEVER use mutexes, goroutines, or blocking operations in actor callbacks.

### 2. Supervision

**Strategies:**
- OneForOne - only failing child restarts (default)
- AllForOne - all children restart
- RestForOne - failed + later siblings restart
- SimpleOneForOne - dynamic worker pool

**Child Restart Policies:**
- Permanent - always restart
- Transient - restart on abnormal termination
- Temporary - never restart

### 3. Message Patterns

**Async (Send):** Fire-and-forget, no response expected
**Sync (Call):** Blocks until response received
**Important Delivery:** Confirms delivery to remote mailbox
**FR-2PC:** Fully-Reliable Two-Phase Commit for distributed transactions

### 4. Application Structure

**ApplicationSpec fields:**
- Name, Description, Version
- Mode (Temporary/Transient/Permanent)
- Group (process members)
- Map (role to process name mapping)
- Tags (blue/green, canary deployment)

## Actor Implementation

### Basic Actor

```go
package main

import (
    "ergo.services/ergo"
    "ergo.services/ergo/act"
    "ergo.services/ergo/gen"
)

type MyActor struct {
    act.Actor
    counter int
}

func createMyActor() gen.ProcessBehavior {
    return &MyActor{}
}

func (a *MyActor) Init(args ...any) error {
    a.Log().Info("actor started")
    return nil
}

func (a *MyActor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case MessageIncrement:
        a.counter += m.Value
        a.Log().Info("counter: %d", a.counter)
    }
    return nil
}

func (a *MyActor) HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) {
    switch r := request.(type) {
    case GetCounterRequest:
        return GetCounterResponse{Value: a.counter}, nil
    }
    return nil, fmt.Errorf("unknown request: %T", request)
}

func (a *MyActor) Terminate(reason error) {
    a.Log().Info("actor terminated: %v", reason)
}

// Messages - use MessageXXX for async, XXXRequest/XXXResponse for sync
type MessageIncrement struct {
    Value int
}

type GetCounterRequest struct{}
type GetCounterResponse struct {
    Value int
}

func main() {
    node, err := ergo.StartNode("mynode@localhost", gen.NodeOptions{})
    if err != nil {
        panic(err)
    }

    pid, err := node.Spawn(createMyActor, gen.ProcessOptions{})
    if err != nil {
        panic(err)
    }

    // Send async message
    node.Send(pid, MessageIncrement{Value: 5})

    // Call sync request
    result, err := node.Call(pid, GetCounterRequest{})
    if err == nil {
        resp := result.(GetCounterResponse)
        fmt.Printf("Counter: %d\n", resp.Value)
    }

    node.Wait()
}
```

### Spawning Children

```go
func (a *ParentActor) Init(args ...any) error {
    // Spawn child actor
    childPID, err := a.Spawn(createChildActor, gen.ProcessOptions{})
    if err != nil {
        return err
    }
    a.childPID = childPID

    // Spawn with registered name
    _, err = a.SpawnRegister("worker", createWorker, gen.ProcessOptions{})
    if err != nil {
        return err
    }

    return nil
}
```

### Links and Monitors

```go
func (a *MyActor) Init(args ...any) error {
    // Link - bidirectional, both terminate together
    a.Link(otherPID)

    // Monitor - unidirectional, receive MessageDownPID on termination
    a.Monitor(otherPID)

    return nil
}

func (a *MyActor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case gen.MessageDownPID:
        a.Log().Info("monitored process %v terminated: %v", m.PID, m.Reason)
    }
    return nil
}
```

## Supervision

### Supervisor Implementation

```go
type MySupervisor struct {
    act.Supervisor
}

func createSupervisor() gen.ProcessBehavior {
    return &MySupervisor{}
}

func (s *MySupervisor) Init(args ...any) (act.SupervisorSpec, error) {
    return act.SupervisorSpec{
        Type: act.SupervisorTypeOneForOne,
        Restart: act.SupervisorRestart{
            Strategy:  act.SupervisorStrategyTransient,
            Intensity: 5,  // max restarts
            Period:    10, // within 10 seconds
        },
        Children: []act.SupervisorChild{
            {
                Name:    "worker1",
                Factory: createWorker,
                Args:    []any{"config1"},
                Restart: act.SupervisorChildRestartPermanent,
            },
            {
                Name:    "worker2",
                Factory: createWorker,
                Args:    []any{"config2"},
                Restart: act.SupervisorChildRestartTransient,
            },
        },
    }, nil
}
```

### Dynamic Child Management

```go
func (a *MyActor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case StartWorkerRequest:
        // Start child via supervisor
        spec := act.SupervisorChild{
            Name:    m.Name,
            Factory: createWorker,
            Restart: act.SupervisorChildRestartTemporary,
        }
        err := a.Send(a.supervisorPID, act.MessageStartChild{Child: spec})

    case StopWorkerRequest:
        err := a.Send(a.supervisorPID, act.MessageStopChild{Name: m.Name})
    }
    return nil
}
```

## Message Patterns

### Send vs Call

```go
// Async - fire and forget
a.Send(targetPID, MessageDoWork{Data: "task1"})

// Async with priority
a.SendWithPriority(targetPID, UrgentMessage{}, gen.MessagePriorityHigh)

// Sync - blocks until response
result, err := a.Call(targetPID, GetStatusRequest{})
if err != nil {
    // Handle timeout or error
}

// Sync with timeout
result, err := a.CallWithTimeout(targetPID, request, 5*time.Second)
```

### Important Delivery (Network Transparency)

```go
// Send with delivery confirmation (blocks until ACK)
err := a.SendImportant(remotePID, CriticalMessage{})
if err != nil {
    // ErrProcessUnknown - process doesn't exist
    // ErrProcessMailboxFull - mailbox is full
    // ErrTimeout - no ACK received
}

// Call with delivery confirmation
result, err := a.CallImportant(remotePID, CriticalRequest{})
// Immediate error if process missing (no ambiguous timeout)

// Process-level flag - all messages use Important Delivery
func (a *MyActor) Init(args ...any) error {
    a.SetImportantDelivery(true)
    return nil
}
```

### FR-2PC (Fully-Reliable Two-Phase Commit)

```go
// Caller side - request confirmed delivered
result, err := process.CallImportant(target, TransactionRequest{})

// Handler side - response confirmed delivered
func (h *Handler) HandleCall(from gen.PID, ref gen.Ref, request any) (any, error) {
    h.SetImportantDelivery(true) // Response uses Important Delivery
    result := h.processTransaction(request)
    return TransactionResponse{Result: result}, nil
}
// Both directions guaranteed - foundation for distributed transactions
```

## Application Structure

### ApplicationSpec

```go
type MyApp struct {
    act.Application
}

func createMyApp() gen.ApplicationBehavior {
    return &MyApp{}
}

func (a *MyApp) Load(args ...any) (gen.ApplicationSpec, error) {
    return gen.ApplicationSpec{
        Name:        "my-service",
        Description: "My service description",
        Version:     "1.0.0",
        Mode:        gen.ApplicationModeTransient,
        Tags:        []gen.Tag{"production"},
        Map: gen.ApplicationMap{
            "api":    "api-handler",
            "worker": "background-worker",
        },
        Group: []gen.ApplicationMemberSpec{
            {
                Factory: createAPIHandler,
                Name:    "api-handler",
            },
            {
                Factory: createWorker,
                Name:    "background-worker",
            },
        },
        Env: map[gen.Env]any{
            "config_key": "config_value",
        },
    }, nil
}

func (a *MyApp) Start(mode gen.ApplicationMode) error {
    a.Log().Info("application started in mode: %s", mode)
    return nil
}

func (a *MyApp) Terminate(reason error) {
    a.Log().Info("application terminated: %v", reason)
}
```

### Starting Application

```go
func main() {
    node, _ := ergo.StartNode("mynode@localhost", gen.NodeOptions{})

    // Load and start application
    err := node.ApplicationLoad(createMyApp)
    if err != nil {
        panic(err)
    }

    err = node.ApplicationStart("my-service")
    if err != nil {
        panic(err)
    }

    node.Wait()
}
```

## Pool (Parallel Processing)

### Pool Implementation

```go
type WorkerPool struct {
    act.Pool
}

func createPool() gen.ProcessBehavior {
    return &WorkerPool{}
}

func (p *WorkerPool) Init(args ...any) (act.PoolOptions, error) {
    return act.PoolOptions{
        PoolSize:          10,           // 10 workers
        WorkerMailboxSize: 20,           // 20 messages per worker
        WorkerFactory:     createWorker, // Worker factory
        WorkerArgs:        []any{"config"},
    }, nil
}
// Capacity = PoolSize * WorkerMailboxSize = 200 concurrent messages

// Pool-level message handling (Urgent/System priority)
func (p *WorkerPool) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case ScaleUpCommand:
        newSize, _ := p.AddWorkers(m.Count)
        p.Log().Info("scaled to %d workers", newSize)
    }
    return nil
}

// Worker implementation
type Worker struct {
    act.Actor
}

func createWorker() gen.ProcessBehavior {
    return &Worker{}
}

func (w *Worker) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case WorkRequest:
        result := w.process(m)
        w.Send(from, WorkResult{Data: result})
    }
    return nil
}
```

### When to Use Pool

- Message rate > 1000 msg/sec per actor
- Independent work items (no ordering dependencies)
- Stateless or cheap-to-reconstruct worker state

**Don't use Pool when:**
- Work items depend on previous items
- Workers need persistent state
- Single actor is fast enough

## Meta Processes (I/O Integration)

### TCP Server

```go
func (a *Actor) Init(args ...any) error {
    options := meta.TCPServerOptions{
        Host: "0.0.0.0",
        Port: 8080,
        ProcessPool: []gen.Atom{"worker1", "worker2"}, // Route connections
    }

    server, err := meta.CreateTCPServer(options)
    if err != nil {
        return err
    }

    serverID, err := a.SpawnMeta(server, gen.MetaOptions{})
    if err != nil {
        server.Terminate(err) // Close socket on failure
        return err
    }

    return nil
}

func (a *Actor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case meta.MessageTCPConnect:
        a.Log().Info("client connected: %s", m.RemoteAddr)
        a.Send(m.ID, meta.MessageTCP{Data: []byte("Welcome!\n")})

    case meta.MessageTCP:
        // Echo received data
        a.Send(m.ID, meta.MessageTCP{Data: m.Data})

    case meta.MessageTCPDisconnect:
        a.Log().Info("client disconnected: %s", m.ID)
    }
    return nil
}
```

### TCP Client

```go
func (a *Actor) Init(args ...any) error {
    options := meta.TCPConnectionOptions{
        Host: "example.com",
        Port: 80,
    }

    conn, err := meta.CreateTCPConnection(options)
    if err != nil {
        return err
    }

    connID, err := a.SpawnMeta(conn, gen.MetaOptions{})
    if err != nil {
        conn.Terminate(err)
        return err
    }

    a.connID = connID
    return nil
}

func (a *Actor) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case meta.MessageTCPConnect:
        // Connection established, send request
        request := "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n"
        a.Send(m.ID, meta.MessageTCP{Data: []byte(request)})

    case meta.MessageTCP:
        a.Log().Info("response: %s", string(m.Data))

    case meta.MessageTCPDisconnect:
        a.Log().Info("disconnected")
    }
    return nil
}
```

### Meta Process Restrictions

- Run 2 goroutines (External Reader for I/O, Actor Handler for messages)
- Cannot make Call() (no sync requests from meta process)
- Cannot create links/monitors (can only receive them)

## EDF Serialization

### Type Registration

```go
import "ergo.services/ergo/net/edf"

// Register in init() BEFORE node starts
func init() {
    // Register nested types first
    edf.RegisterTypeOf(Address{})
    edf.RegisterTypeOf(Person{})

    // Register errors
    edf.RegisterError(ErrInvalidOrder)
}

// All fields MUST be Exported for cross-node messages
type Address struct {
    City   string  // Exported
    Street string  // Exported
}

type Person struct {
    Name    string  // Exported
    Address Address // Exported, nested type registered first
}

// WRONG - unexported fields cannot be serialized
type BadMessage struct {
    Name string
    data []byte // unexported - EDF will fail
}
```

### Type Constraints

| Type | Max Size |
|------|----------|
| Atom | 255 bytes |
| String | 65535 bytes |
| Error | 32767 bytes |
| Binary | 4GB |
| Collections | 2^32 elements |

## Cluster Configuration

### etcd Registrar (50-70 nodes)

```go
import "ergo.services/registrar/etcd"

node, err := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: etcd.Create(etcd.Options{
            Endpoints: []string{"etcd1:2379", "etcd2:2379"},
        }),
    },
})
```

### Saturn Registrar (1000+ nodes)

```go
import "ergo.services/registrar/saturn"

node, err := ergo.StartNode("mynode@localhost", gen.NodeOptions{
    Network: gen.NetworkOptions{
        Registrar: saturn.Create(saturn.Options{
            Nodes: []string{"saturn1:4499", "saturn2:4499"},
        }),
    },
})
```

### Service Discovery

```go
func (a *Gateway) callService(request any) (any, error) {
    registrar, _ := a.Node().Network().Registrar()
    routes, _ := registrar.Resolver().ResolveApplication("my-service")

    for _, r := range routes {
        for _, tag := range r.Tags {
            if tag == "production" {
                return a.Call(
                    gen.ProcessID{Node: r.Node, Name: r.Map.Get("handler")},
                    request,
                )
            }
        }
    }
    return nil, fmt.Errorf("service not found")
}
```

## Best Practices

1. **Message Naming**: MessageXXX for async, XXXRequest/XXXResponse for sync
2. **Actor State**: Keep state in struct fields, never share between actors
3. **Error Handling**: Return errors from Init/HandleMessage to trigger supervision
4. **Pool Justification**: Only use Pool when message rate exceeds single actor capacity
5. **Important Delivery**: Use for critical messages, CallImportant for distributed transactions
6. **EDF Registration**: Register all types in init() before node starts
7. **Registrar Selection**: Embedded for dev, etcd for 50-70 nodes, Saturn for 1000+
8. **Meta Processes**: Use for blocking I/O (TCP, HTTP), not for business logic

## Reference

When uncertain about APIs or behavior, consult the Ergo Framework source code and `/docs` directory in Go module cache:
```bash
ls $(go env GOMODCACHE)/ergo.services/ergo@*/docs/
```

## Common Pitfalls

- **Blocking in Callbacks**: NEVER use mutexes, channels, or blocking I/O in actors
- **Shared State**: Actors must not share state, always copy messages
- **act.Pool in ProcessPool**: Don't use act.Pool in TCPServer.ProcessPool (breaks connection binding)
- **Unexported Fields**: EDF cannot serialize unexported struct fields
- **Late Registration**: EDF types must be registered before node starts
- **Embedded Registrar in Production**: Use etcd or Saturn for production clusters
- **Unnecessary Pool**: Single actor handles 100+ msg/sec easily, profile first
- **Sync in Meta Process**: Meta processes cannot make Call() requests
