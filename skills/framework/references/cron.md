# Cron

Ergo ships a node-level scheduler that runs tasks on a crontab schedule. You declare *what* runs and *when*; the framework owns timing and execution. Access it through `node.Cron()`, or declare jobs at boot in `NodeOptions.Cron`.

A job is a `gen.CronJob`: a unique name, a five-field crontab `Spec`, an optional timezone, an `Action` to perform when the schedule fires, and an optional `Fallback` process that is notified when the action fails.

## Defining a Job

```go
cron := node.Cron()

err := cron.AddJob(gen.CronJob{
    Name:     "midnight-sweep",
    Spec:     "0 0 * * *",
    Location: nil, // nil = node local time
    Action:   gen.CreateCronActionMessage("scheduler", gen.MessagePriorityNormal),
})
```

`CronJob` fields:

| Field | Type | Purpose |
|-------|------|---------|
| `Name` | `gen.Atom` | Unique job identifier within the node |
| `Spec` | `string` | Five-field crontab schedule (see below) |
| `Location` | `*time.Location` | Timezone the `Spec` is interpreted in; `nil` means the node's local timezone (`time.Local`) |
| `Action` | `gen.CronAction` | What runs when the schedule fires |
| `Fallback` | `gen.ProcessFallback` | Process notified on action failure (see Failure Handling) |

## Crontab Format

`Spec` is the standard five-field crontab string: minute, hour, day-of-month, month, day-of-week.

| Spec | Meaning |
|------|---------|
| `* * * * *` | Every minute |
| `0 * * * *` | Every hour (top of the hour) |
| `*/5 * * * *` | Every 5 minutes |
| `0 0 * * *` | Every day at midnight |

Per field the parser accepts `*`, a single value, a comma list (`0,30`), a range (`10-13`), a step (`*/5`), and a range with step (`10-13/3`). Mixing `*` with any other token in the same field is rejected. Field bounds: minute `0-59`, hour `0-23`, day-of-month `1-31`, month `1-12`, day-of-week `1-7` (Monday to Sunday; `0` is not accepted).

The day-of-month and day-of-week fields also accept these extended tokens:

| Token | Field | Meaning |
|-------|-------|---------|
| `L` | day-of-month | Last day of the month (leap-year aware) |
| `nL` | day-of-week | Last given weekday of the month (e.g. `7L` = last Sunday) |
| `n#k` | day-of-week | The k-th given weekday of the month (e.g. `1#2` = second Monday, k is `1-5`) |

When both day-of-month and day-of-week are constrained (neither is `*`), a firing occurs if either field matches (OR), following standard cron behavior.

Four macros are accepted in place of a full spec. They expand to fixed times (not the usual "top of the period"):

| Macro | Expands to | Meaning |
|-------|-----------|---------|
| `@hourly` | `1 * * * *` | Minute 1 of every hour |
| `@daily` | `10 3 * * *` | Every day at 03:10 |
| `@weekly` | `30 5 * * 1` | Mondays at 05:30 |
| `@monthly` | `20 4 1 * *` | Day 1 of the month at 04:20 |

Fields are evaluated in the job's `Location`. A job with a `Spec` of `0 0 * * *` and a New York `Location` fires at New York midnight; the same `Spec` with `Location: nil` fires at the node's local midnight (`time.Local`). Timezone is per-job, so one node can schedule tasks for multiple regions correctly.

## Actions

An action is a `gen.CronAction`. Three builtin constructors cover the common cases, and you can implement the interface yourself.

| Constructor | Effect |
|-------------|--------|
| `CreateCronActionMessage(to, priority)` | Sends `gen.MessageCron` to `to` at the given priority |
| `CreateCronActionSpawn(factory, options)` | Spawns a local process |
| `CreateCronActionRemoteSpawn(node, name, options)` | Spawns a process on a remote node |

There is no "call" action - the message action is fire-and-forget.

### Message Action

The most common action. When the schedule fires, the scheduler sends `gen.MessageCron` to the target and the receiving actor handles it like any other message. `to` may be a name (`gen.Atom`), `gen.PID`, `gen.ProcessID`, or `gen.Alias`.

```go
action := gen.CreateCronActionMessage("worker", gen.MessagePriorityNormal)
```

The target receives:

```go
type MessageCron struct {
    Node gen.Atom  // node the job ran on
    Job  gen.Atom  // job name
    Time time.Time // scheduled action time (node local wall-clock, truncated to the minute)
}
```

Match on it in the receiver's `HandleMessage`:

```go
func (w *worker) HandleMessage(from gen.PID, message any) error {
    switch m := message.(type) {
    case gen.MessageCron:
        w.Log().Info("cron job %s fired at %s", m.Job, m.Time)
        // do the scheduled work
    }
    return nil
}
```

Because scheduled work runs inside the target's mailbox, a message action serializes naturally - one actor processing `gen.MessageCron` one at a time will never overlap runs, no matter how tight the schedule.

### Spawn Action

Spawns a fresh process per firing. Use it when each run should be isolated - a crash in one run does not affect the next.

```go
action := gen.CreateCronActionSpawn(createReportWorker, gen.CronActionSpawnOptions{
    Register: "report-worker",           // optional registered name
    ProcessOptions: gen.ProcessOptions{}, // spawn options
    Args:           []any{"weekly"},      // args passed to the process
})
```

`CronActionSpawnOptions`:

| Field | Type | Purpose |
|-------|------|---------|
| `Register` | `gen.Atom` | Register the spawned process under this name; empty means unregistered |
| `ProcessOptions` | `gen.ProcessOptions` | Spawn options (env, mailbox, priority, ...) |
| `Args` | `[]any` | Arguments passed to the process factory |

The scheduler injects three environment variables into every spawned process so it can tell which job spawned it and when. Read them with `process.Env(name)`:

| Env key | Value |
|---------|-------|
| `gen.CronEnvNodeName` | `gen.Atom` - node the job ran on |
| `gen.CronEnvJobName` | `gen.Atom` - job name |
| `gen.CronEnvJobActionTime` | `time.Time` - scheduled action time |

```go
func (w *reportWorker) Init(args ...any) error {
    if v, ok := w.Env(gen.CronEnvJobName); ok {
        w.job = v.(gen.Atom)
    }
    return nil
}
```

These are merged with any env you set in `ProcessOptions.Env` (the injected keys take precedence).

### Remote Spawn Action

Spawns on a remote node by process factory name. Centralize scheduling on one node while distributing execution.

```go
action := gen.CreateCronActionRemoteSpawn("worker@datanode", "report_worker",
    gen.CronActionSpawnOptions{})
```

The same `CronActionSpawnOptions` and injected env keys apply. The remote node must have enabled the named factory for remote spawn (`Network().EnableSpawn(name, factory, ...)`) - see `cluster.md`.

### Custom Action

Implement `gen.CronAction` for anything else:

```go
type CronAction interface {
    Do(job gen.Atom, node gen.Node, atime time.Time) error
    Info() string
}
```

`Do` runs when the schedule fires; `atime` is the scheduled action time (node local wall-clock, truncated to the minute). The `Location` only governs when the `Spec` matches (schedule evaluation), not the timestamp handed to the action. Returning a non-nil error triggers the job's `Fallback`. `Info` returns a human-readable description shown in `CronJobInfo.ActionInfo`.

## Declarative Jobs at Node Start

Instead of calling `AddJob` imperatively, declare jobs in `NodeOptions.Cron`. They are registered automatically when the node starts.

```go
node, err := ergo.StartNode("app@localhost", gen.NodeOptions{
    Cron: gen.CronOptions{
        Jobs: []gen.CronJob{
            {
                Name:   "midnight-sweep",
                Spec:   "0 0 * * *",
                Action: gen.CreateCronActionMessage("scheduler", gen.MessagePriorityNormal),
            },
        },
    },
})
```

`CronOptions` has a single field, `Jobs []CronJob`.

## Managing Jobs

The `gen.Cron` interface (from `node.Cron()`) manages jobs at runtime:

| Method | Purpose | Error on failure |
|--------|---------|------------------|
| `AddJob(job CronJob) error` | Register a job | `gen.ErrTaken` if name exists |
| `RemoveJob(name Atom) error` | Unregister a job | `gen.ErrUnknown` if unknown |
| `EnableJob(name Atom) error` | Resume a disabled job | `gen.ErrUnknown` if unknown |
| `DisableJob(name Atom) error` | Keep registered but stop firing | `gen.ErrUnknown` if unknown |
| `Info() CronInfo` | Scheduler overview | - |
| `JobInfo(name Atom) (CronJobInfo, error)` | One job's details | `gen.ErrUnknown` if unknown |
| `Schedule(since time.Time, dur time.Duration) []CronSchedule` | Preview which jobs fire at each upcoming time | - |
| `JobSchedule(job Atom, since time.Time, dur time.Duration) ([]time.Time, error)` | Upcoming run times for one job | `gen.ErrUnknown` if unknown |

`DisableJob` is useful for maintenance windows: the job stays registered (and visible in `Info`) but does not fire until `EnableJob`.

## Introspection

`Info()` returns scheduler state:

```go
type CronInfo struct {
    Next  time.Time     // time of the next scheduled execution
    Spool []gen.Atom    // job names queued for immediate execution
    Jobs  []CronJobInfo // all registered jobs
}
```

`JobInfo(name)` (and each entry in `CronInfo.Jobs`) returns `CronJobInfo`:

| Field | Type | Meaning |
|-------|------|---------|
| `Disabled` | `bool` | Job is registered but not firing |
| `Name` | `gen.Atom` | Job identifier |
| `Spec` | `string` | Crontab schedule |
| `Location` | `string` | Timezone name (e.g. `"UTC"`) |
| `ActionInfo` | `string` | Human-readable action description (from the action's `Info()`) |
| `LastRun` | `time.Time` | Last execution time; zero if it has never run |
| `LastErr` | `string` | Error from the last failed run; empty on success |
| `Fallback` | `gen.ProcessFallback` | The job's failure-notification target |

`Schedule` and `JobSchedule` preview future firings. They run the same schedule evaluation the scheduler uses, so they are handy to verify a `Spec` is correct or to spot two jobs colliding on the same minute.

```go
// which jobs fire in the next 24 hours, grouped by time
for _, s := range cron.Schedule(time.Now(), 24*time.Hour) {
    fmt.Println(s.Time, s.Jobs) // CronSchedule{Time time.Time; Jobs []gen.Atom}
}

// next 5 run times for one job over the next week
times, err := cron.JobSchedule("midnight-sweep", time.Now(), 7*24*time.Hour)
```

## Failure Handling

When an action's `Do` returns an error and the job has a `Fallback` configured with `Enable: true`, the scheduler forwards the failure to the fallback process as `gen.MessageCronFallback`. The error is also recorded in `CronJobInfo.LastErr`.

```go
type MessageCronFallback struct {
    Job  gen.Atom  // job that failed
    Tag  string    // Fallback.Tag, to identify the source
    Time time.Time // scheduled action time
    Err  error     // the action error
}
```

Configure the fallback on the job:

```go
job := gen.CronJob{
    Name:   "billing-run",
    Spec:   "0 2 * * *",
    Action: gen.CreateCronActionSpawn(createBillingWorker, gen.CronActionSpawnOptions{}),
    Fallback: gen.ProcessFallback{
        Enable: true,
        Name:   "cron-alerts",
        Tag:    "billing",
    },
}
```

A single fallback process can collect failures from many jobs (distinguished by `MessageCronFallback.Tag`) to log, alert, or retry. `gen.ProcessFallback` is the same struct used for mailbox overflow (see `messages.md`); here it targets action failures rather than a full mailbox.

## Notes

- Job names are unique per node - `AddJob` with an existing name returns `gen.ErrTaken`.
- `RemoveJob`, `EnableJob`, `DisableJob`, and `JobInfo` return `gen.ErrUnknown` for a name that is not registered.
- The message action serializes runs through the target's mailbox; the spawn actions do not - a slow spawn action whose schedule fires again before it finishes can run concurrently. Prefer a message action to a named process when runs must not overlap.
- The scheduler wakes every wall-clock minute. Before running a job it re-checks that the current minute (in the job's `Location`) still equals the minute the run was scheduled for; if they differ - for example because the clock was adjusted or a DST transition moved the wall-clock time - that job is skipped for this tick rather than fired at the wrong minute.

For the exhaustive interface and struct definitions, see the godoc for `gen.Cron`, `gen.CronJob`, and `gen.CronAction`.
