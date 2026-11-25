# apogee-queue-dashboard

A dashboard for [apogee-queue](https://github.com/venomous-maker/apogee-queue),
in the manner of Laravel Horizon: what is waiting on each queue and how long
it waits, how many jobs finish a minute, how long each kind of job takes, what
failed (look at it, retry it, forget it), batches and their progress, jobs by
tag (monitored tags keep theirs), every job's payload, which workers run
(pause, continue or restart them from the page), and alerts when a queue's
wait grows too long. One page with its own CSS and script (no build step,
nothing fetched from elsewhere), light and dark, refreshing every few seconds,
over a JSON API. Its record lives in memory, in Redis, or in a database.

```grav
import "std"
import "apogee-queue-dashboard"
use apogee.*

let store = new RedisDashboardStore(RedisConnection.open(env("REDIS_URL")))
// or: let store = new DatabaseDashboardStore(pool); store.createTables()
let dash = new QueueDashboard(queue, store)

dash.attach(events)                       // in every process that dispatches or works jobs
dash.routes(router, "queue-dashboard")    // in the web app: the page at /queue-dashboard
dash.gate(fn(req: Request) -> bool { return req.user != null && (req.user as User).isAdmin })
dash.silence("Heartbeat")                 // counted, but kept out of the job list
dash.alertWhenWaitExceeds(60, fn(w: LongWait) { notify("${w.queue} waits ${w.seconds}s") })
dash.monitor(60)                          // check the waits every minute
```

## In an apogee-framework app

Importing the module is all it takes: `QueueDashboardPackage` (apogee-package) connects it when
the app is created. When `queue_dashboard.enabled`, the queue's jobs, waits, failures and batches are recorded and shown at `/queue-dashboard`. The page is open locally; elsewhere only to a signed-in user the gate allows `viewQueueDashboard`. The `QueueDashboard` is bound in the container.

**Migration:** `CreateQueueDashboardTables` makes the eight `queue_dashboard_*` tables. Run `migrate` after adding the module.

| Setting | Default |
|---|---|
| `queue_dashboard.enabled` | `QUEUE_DASHBOARD_ENABLED`, off |
| `queue_dashboard.driver` | `QUEUE_DASHBOARD_DRIVER`: `database` (default), `redis` (`redis.url`) or `memory` |
| `queue_dashboard.path` | `QUEUE_DASHBOARD_PATH`, `queue-dashboard` |

Your own configuration wins over these defaults.

## What it records

`attach(events)` listens to the queue's own events (the `Events` the
`QueueManager` fires, which carry each job's payload): every job queued,
started, finished, released or failed, with its tags, batch, attempts, how
long it waited (from when it was pushed) and how long it ran; and every worker
starting and stopping. Workers usually run in other processes than the web
app, so the record lives in a store they share:

| | |
|---|---|
| `RedisDashboardStore(redis, prefix = "queue-dashboard:", keep = 1000)` | shared by every process. Each job event is **one round trip**: a Lua script merges the job's record and updates every count on the server; reads are pipelined. Per-minute counts expire after two days. |
| `DatabaseDashboardStore(pool, prefix = "queue_dashboard_", keep = 1000)` | SQLite, MySQL or PostgreSQL through [grav-db](https://github.com/venomous-maker/grav-db); `createTables()` makes its tables. Each job event is one transaction of upserts; old rows are pruned every hundred writes. |
| `MemoryDashboardStore(keep = 1000)` | one process (development, tests) |

The newest `keep` jobs are kept, and `keep` more for each monitored tag.

## The page

- **Headline**: jobs per minute, processed and failed in the last hour, failed
  jobs kept, the longest estimated wait (and its queue), workers running, and
  whether the workers are paused.
- **Throughput** for the last hour, processed and failed per minute.
- **Queues**: waiting, delayed and reserved jobs, average wait, average run
  time, estimated wait for a job queued now (the waiting jobs times the average
  run time, shared among the running workers that take the queue), workers,
  processed and failed.
- **Jobs**: the latest (all, queued, processing, processed, failed), or every
  job with a tag (search it, or click one); click a job for its detail: status,
  timings, tags, batch, exception and its payload.
- **Monitored tags**: monitor a tag to keep its jobs past the recent ones; stop
  when done.
- **Failed jobs**: exception, tags and payload; retry one or all, or forget it.
- **Batches**: progress, pending and failed jobs, state.
- **Jobs by class**: processed, failed, average and slowest run time.
- **Workers**: name, connection, queues, started, running or paused or stopped.
- **Pause / Continue / Restart workers**: through the queue's shared cache
  (apogee-queue's `Worker.signalPause` and `signalRestart`), so it reaches
  workers in every process that shares it.

## Alerts

`alertWhenWaitExceeds(seconds, f)` calls `f(LongWait)` (connection, queue,
estimated seconds, waiting jobs) when a queue's estimated wait passes
`seconds`, at most once per queue every five minutes. `checkWaits()` checks
now (from a scheduled task, say); `monitor(everySeconds)` checks on a
background thread.

## The JSON API

Under the same prefix:

| | |
|---|---|
| `GET /api/stats` | the headline, the queues and the workers |
| `GET /api/throughput?minutes=60` | per minute |
| `GET /api/metrics/jobs`, `GET /api/metrics/queues` | per job class, per queue |
| `GET /api/jobs/recent?limit=50&status=failed` | the latest jobs |
| `GET /api/jobs/tagged/{tag}` | jobs with a tag |
| `GET /api/jobs/{id}` | one job, with its payload |
| `GET /api/jobs/failed`, `GET /api/jobs/failed/{id}` | failed jobs |
| `POST /api/jobs/failed/{id}/retry`, `POST /api/jobs/failed/retry-all`, `DELETE /api/jobs/failed/{id}` | |
| `GET /api/batches` | batches seen lately |
| `GET /api/tags`, `POST /api/tags` (`{"tag": ...}`), `DELETE /api/tags/{tag}` | monitored tags |
| `POST /api/workers/pause`, `/continue`, `/restart` | |

## Who may see it

By default, only requests from this machine. `gate(fn(Request) -> bool)`
decides otherwise; it also guards every action.

## Tests

```bash
grav test tests/dashboard.grav
grav test tests/package.grav
```

They run jobs through a real worker and check that the memory, Redis and
database (SQLite) stores record the same; they drive the pages with
[apogee-testing](https://github.com/venomous-maker/apogee-testing). The Redis
store's test needs a server (`REDIS_URL`, default `redis://127.0.0.1:6379/9`).
