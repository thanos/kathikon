# Kathikon

[![CI](https://github.com/AnimaLogica/kathikon/actions/workflows/ci.yml/badge.svg)](https://github.com/AnimaLogica/kathikon/actions/workflows/ci.yml)
[![Coverage Status](https://coveralls.io/repos/github/AnimaLogica/kathikon/badge.svg?branch=main)](https://coveralls.io/github/AnimaLogica/kathikon?branch=main)
[![Hex version](https://img.shields.io/hexpm/v/kathikon.svg)](https://hex.pm/packages/kathikon)
[![Hex Docs](https://img.shields.io/badge/hex-docs-8A33A3.svg)](https://hexdocs.pm/kathikon)

**Kathikon** (Greek: καθήκον - duty, obligation) is a BEAM-native durable job queue and task execution platform for Elixir.

Jobs are treated as durable obligations that must eventually be fulfilled: completed, retried, cancelled, or discarded - but never silently lost.

Kathikon uses **Mnesia** as its coordination store and **OTP** as its execution substrate. No PostgreSQL, Redis, RabbitMQ, or external brokers are required.

## Features

### Core queue

1. **Durable jobs in Mnesia.** Jobs are stored in Mnesia, so no PostgreSQL, Redis, or broker is needed.
2. **Explicit job state machine.** There are 11 states: `scheduled`, `available`, `claimed`, `running`, `waiting_for_children`, `retryable`, `completed`, `failed`, `dead`, `discarded`, and `cancelled`.
3. **Workers.** `use Kathikon.Worker` with `perform/1`. Return values are `:ok`, `{:ok, value}`, `{:sleep, seconds}`, `{:discard, reason}`, `{:retry, reason}`, and `{:error, reason}`.
4. **Queues and concurrency.** Each queue has its own concurrency and one dispatcher. You can pause and resume a single queue or all queues.
5. **Priority and retries.** Jobs have a `priority` and a `max_attempts` limit.
6. **Cancel.** Pending jobs can be cancelled with `Kathikon.cancel/1`.
7. **Durable history.** Each job keeps an audit trail of state transitions.
8. **Dead-letter queue.** Failed jobs can be listed, rerun, retried, or discarded (`rerun/2`, `retry_dead/2`, `discard_dead/2`).
9. **Job uniqueness (v0.5.0).** `unique:` on a worker or on `insert/3` coalesces duplicates. The `criteria:` function picks the key, and `on_conflict: :error` can reject duplicates instead. Coalescing emits telemetry, and a warning is logged when the existing job has already completed.

### Bulk and workflows

10. **Bulk insert.** `insert_many/2` writes chunks with partial success and per-index errors.
11. **Streaming batches.** `Kathikon.Batch.open/2`, `append/2`, and `close/1` add children incrementally without holding every spec in memory.
12. **Parent/child batches.** Fan-out and fan-in workflows, with batch telemetry.

### Scheduling

13. **One-time and recurring schedules.** `Kathikon.schedule/3` accepts `at:`, `in:`, or `cron:`.
14. **Cron.** Standard 5-field expressions with timezone support, and schedules can be updated.
15. **Scheduler adapters.** A built-in scheduler, plus an optional Quantum adapter.
16. **Promoter.** Moves due scheduled jobs to `available`.

### Retention and storage

17. **Pruner.** Deletes old `completed`, `cancelled`, and `discarded` jobs after `retention_period`. It also clears unique index rows for old `failed` and `dead` jobs, which it never deletes.
18. **Storage behaviour.** Mnesia is the default. Optional callbacks cover bulk writes (`insert_jobs/2`), unique inserts (`insert_unique/2`), and paginated listing (`list_jobs_page/1`).

### Operations and observability

19. **Dashboard facade.** `Kathikon.Dashboard` gives queue summaries, job lists, and bulk control for UIs and RPC.
20. **Remote ops.** `Kathikon.Dashboard.RPC` makes whitelisted calls over Erlang distribution.
21. **CLI.** `mix kathikon.ops` provides `summary`, `jobs`, `show`, `pause`, `resume`, `cancel`, `retry`, `rerun`, `purge`, and `prune`.
22. **Phoenix LiveDashboard page.** `Kathikon.LiveDashboard.Page` covers queue health, pause and resume, cancel, filter, search, retry, and paginated lists.
23. **Reporting.** `Kathikon.Report` provides queue and job summaries.
24. **Telemetry.** Events are under `[:kathikon, ...]`, with a default logger you can attach.
25. **Examples and a livebook.** Runnable scripts in `examples/` and an interactive demo in `livebooks/`.

### Not built yet

- **Lifeline.** `Kathikon.Lifeline` is a placeholder that returns `{:error, :not_implemented}`.
- **Leases and distributed coordination.** Planned for v0.7.0.
- **Ecto backend.** Planned for v0.6.0.
- **Later:** MongoDB, SQS, workflows/DAGs, dynamic queues, rate limits, and a stable API.

## Status

**v0.5.0 - Job uniqueness**

- `use Kathikon.Worker, unique: [period:, criteria: fun]` and `insert/3` `:unique` opts
- Duplicate enqueues coalesce to the existing in-flight or completed job; `criteria:` picks what counts as a duplicate, and `[:kathikon, :job, :coalesced]` reports each coalesce
- Mnesia `:kathikon_unique` fingerprint index

**v0.4.0 - Bulk job submission**

- `Kathikon.insert_many/2` — chunked enqueue of independent jobs
- `Kathikon.Batch.open/2`, `append/2`, and `close/1` — stream map/reduce children without holding every spec in memory
- Storage `insert_jobs/2`, implemented transactionally by Mnesia

**v0.3.0 - LiveDashboard**

- `Kathikon.LiveDashboard.Page` — Kathikon tab in Phoenix LiveDashboard
- Pause, resume, cancel, filter, search, retry, and paginated job lists
- Phoenix setup guide, Playground demo, and Livebook

![Kathikon LiveDashboard](docs/screenshots/dashboard-screenshot.png)

**v0.2.1 - Operations tooling**

- `Kathikon.Dashboard` - inspect and control facade for UIs and RPC
- `mix kathikon.ops` - terminal CLI (`summary`, `jobs`, pause/resume, retry, purge, …)
- Remote ops over Erlang distribution (`Kathikon.Dashboard.RPC`)

**v0.2.0 - Control, scheduling, batches, and correctness**

- Storage behaviour with atomic lifecycle operations
- Formal job state machine and durable history
- Dead-letter queue with rerun semantics
- `Kathikon.schedule/3` and scheduler adapters (built-in + optional Quantum)
- Management and reporting APIs
- Parent/child batch workflows

Previous: **v0.1.0 - Phase 1: Durable Job Queue**

## Installation

```elixir
def deps do
  [
    {:kathikon, "~> 0.5.0"}
  ]
end
```

The [Quick start](docs/guides/quick-start.md) guide covers configuration, a worker, and the first job (v0.5.0). A short version:

Define a worker:

```elixir
defmodule MyApp.EmailWorker do
  use Kathikon.Worker

  @impl true
  def perform(%Kathikon.Job{args: %{"email" => email}}) do
    MyApp.Mailer.deliver(email)
    :ok
  end
end
```

Configure queues:

```elixir
# config/config.exs
config :kathikon,
  queues: [
    default: [concurrency: 10],
    emails: [concurrency: 5]
  ]
```

Enqueue jobs:

```elixir
{:ok, job} = Kathikon.insert(MyApp.EmailWorker, %{"email" => "user@example.com"})

{:ok, job} =
  Kathikon.insert(MyApp.EmailWorker, %{"email" => "user@example.com"},
    queue: :emails,
    priority: 5,
    schedule_in: 60,
    max_attempts: 10
  )
```

Cancel a pending job:

```elixir
{:ok, job} = Kathikon.cancel(job_id)
```

## Architecture

```
Kathikon.Supervisor
├── Registry
├── Kathikon.Queue (DynamicSupervisor)
│   └── Kathikon.Dispatcher (one per queue)
├── Kathikon.Scheduler.Promoter
└── Kathikon.Pruner
```

| Module | Role |
|--------|------|
| `Kathikon.Job` | Job struct and state machine |
| `Kathikon.Worker` | Worker behaviour (`perform/1`) |
| `Kathikon.Storage` | Storage behaviour (default: `Kathikon.Storage.Mnesia`) |
| `Kathikon.Dashboard` | Ops facade - queue summary, job lists, bulk control |
| `Kathikon.Report` | Queue/job reporting helpers |
| `Kathikon.Batch` | Fan-out/fan-in batch workflows |
| `Kathikon.Scheduler.Promoter` | Promotes scheduled jobs to available |
| `Kathikon.Pruner` | Enforces retention on terminal jobs |
| `Kathikon.Telemetry` | `[:kathikon, ...]` telemetry events |

## Job states

```
scheduled → available → claimed → running → completed
                    ↘           ↘ retryable → available
                      cancelled   failed → dead
```

## Telemetry

Kathikon emits standard telemetry events. See [Telemetry guide](docs/guides/telemetry-and-observability.md) for the full list. Highlights:

- Job: `[:kathikon, :job, :inserted]`, `:coalesced`, `:claimed`, `:started`, `:completed`, `:failed`, `:sleep`, `:retry`, `:discard`, `:cancel`, `:dead`, `:prune`
- Runtime: `[:kathikon, :scheduler, :tick]`, `[:kathikon, :scheduler, :fired]`, `[:kathikon, :pruner, :tick]`, `[:kathikon, :dispatcher, :poll]`

Attach the default logger in development:

```elixir
Kathikon.Telemetry.attach_default_logger()
```

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `:queues` | `[default: [concurrency: 10]]` | Queue names and concurrency |
| `:poll_interval` | `200` | Dispatcher poll interval (ms) |
| `:scheduler_interval` | `1000` | Scheduler tick interval (ms) |
| `:prune_interval` | `60000` | Pruner tick interval (ms) |
| `:retention_period` | `7 days` | How long to keep terminal jobs (ms) |
| `:max_attempts` | `20` | Default retry limit |
| `:storage_backend` | `Kathikon.Storage.Mnesia` | Storage implementation |
| `:mnesia_copies` | `:auto` | Mnesia storage: `:ram`, `:disc`, or `:auto` (`ram` on `nonode@nohost` and Livebook nodes) |

## Examples

Runnable scripts under `examples/` (use `mix run examples/<name>.exs`):

| Script | Demonstrates |
|--------|----------------|
| [basic_worker.exs](examples/basic_worker.exs) | Insert and poll status |
| [scheduled_job.exs](examples/scheduled_job.exs) | `schedule` with `:at` / `:in` |
| [dead_letter_retry.exs](examples/dead_letter_retry.exs) | Failures, dead letter, rerun |
| [batch_fanout_fanin.exs](examples/batch_fanout_fanin.exs) | Parent/child batches |
| [reporting.exs](examples/reporting.exs) | `Kathikon.Report` summaries |
| [quantum_scheduler_adapter.exs](examples/quantum_scheduler_adapter.exs) | Optional Quantum scheduler |

## Operations CLI

Inspect and control a running node from the terminal:

```bash
mix kathikon.ops summary
mix kathikon.ops jobs --queue default --tab completed --limit 20
mix kathikon.ops pause --all
```

Remote (named node + matching cookie on both sides):

```bash
elixir --name ops@127.0.0.1 --cookie SECRET -S mix kathikon.ops --node kathikon@127.0.0.1 summary
```

See [Management API](docs/management_api.md) and `Kathikon.Dashboard` docs.

## Roadmap

| Release | Focus |
|---------|--------|
| v0.1.0 | Durable job queue (done) |
| v0.2.0 | Control, scheduling, batches, and correctness (done) |
| v0.2.1 | Operations tooling: Dashboard facade, CLI, and RPC (done) |
| v0.3.0 | LiveDashboard page for a Phoenix app (done) |
| v0.4.0 | Bulk job submission (chunked insert) |
| v0.5.0 | Job uniqueness (`criteria:` functions, Oban-style default states) |
| v0.6.0 | Ecto storage backend (PostgreSQL) |
| v0.7.0 | Distributed coordination: leases, lifeline, worker ownership |
| later | Workflows/DAGs, MongoDB, SQS, dynamic queues, rate limits, stable API |

## Documentation

Generate HTML docs with [ExDoc](https://github.com/elixir-lang/ex_doc):

```bash
mix docs
open doc/index.html
```

- **[Documentation index](docs/documentation.md)** - guides and module reference (source)
- [CHANGELOG](CHANGELOG.md) - release history
- [Quick start](docs/guides/quick-start.md)
- [LiveDashboard in a Phoenix app](docs/guides/live-dashboard.md)
- [Module reference](docs/reference/modules.md)
- [Configuration](docs/guides/configuration.md)
- [Interactive demo (Livebook)](livebooks/kathikon_demo.livemd)


## License

MIT. See [LICENSE](LICENSE).
