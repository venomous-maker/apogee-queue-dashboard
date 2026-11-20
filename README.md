# apogee-queue-dashboard

A dashboard for [apogee-queue](https://github.com/venomous-maker/apogee-queue):
what is waiting on each queue, how many jobs finish a minute, how long each
kind of job takes, what failed (retry or forget it from the page), which
workers run, and the latest jobs. One page with its own CSS and script (no
build step, nothing fetched from elsewhere), light and dark, refreshing every
few seconds, over a small JSON API.

```grav
import "std"
import "apogee-queue-dashboard"
use apogee.*

let store = new RedisDashboardStore(RedisConnection.open(env("REDIS_URL")))
let dash = new QueueDashboard(queue, store)

dash.attach(events)                       // in the workers' process: record the queue's events
dash.routes(router, "queue-dashboard")    // in the web app: the page at /queue-dashboard
dash.gate(fn(req: Request) -> bool { return req.user != null && (req.user as User).isAdmin })
```

## What it records

`attach(events)` listens to the queue's own events (the `Events` the
`QueueManager` fires): each job queued, started, finished (and how long it
took), released or failed (and why), and each worker starting and stopping.
Workers usually run in other processes than the web app, so the record lives
in a store they share:

| | |
|---|---|
| `RedisDashboardStore(redis, prefix = "queue-dashboard:", keep = 1000)` | shared by every process; per-minute counts expire after two days |
| `MemoryDashboardStore(keep = 1000)` | one process (development, tests) |

The newest `keep` jobs are kept.

## The page

Jobs per minute, processed and failed in the last hour, failed jobs kept,
workers running; a throughput chart for the hour; each queue's waiting,
delayed and reserved jobs; the latest jobs (filter by status); failed jobs
with their exceptions, to retry (one or all) or forget; totals, average and
slowest run time per job class; the workers.

`watchQueue(connection, queue)` shows a queue before any job has been seen on
it.

## The JSON API

Under the same prefix: `GET /api/stats`, `GET /api/throughput?minutes=60`,
`GET /api/metrics/jobs`, `GET /api/jobs/recent?limit=50&status=failed`,
`GET /api/jobs/failed`, `POST /api/jobs/failed/{id}/retry`,
`POST /api/jobs/failed/retry-all`, `DELETE /api/jobs/failed/{id}`.

## Who may see it

By default, only requests from this machine. `gate(fn(Request) -> bool)`
decides otherwise (it also guards retrying and forgetting).

## Tests

```bash
grav test tests/dashboard.grav
```

They run jobs through a real worker and drive the pages with
[apogee-testing](https://github.com/venomous-maker/apogee-testing); the Redis
store's test needs a server (`REDIS_URL`, default `redis://127.0.0.1:6379/9`).
