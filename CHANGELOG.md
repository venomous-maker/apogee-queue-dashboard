# Changelog

Every release of apogee-queue-dashboard, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

Needs grav 1.12.0 or newer.

- `QueueDashboardPackage`: the module connects itself to an apogee-framework app (apogee-package):
  its settings, wiring moved here from the framework.
- Its tables as a migration (`CreateQueueDashboardTables`, run by `migrate`); the store's own
  table-making method uses the same definition.

### Dependencies

- Uses apogee-package 0.1.0, grav-orbit 0.3.0, apogee-http 0.3.0, apogee-routing 0.3.0, apogee-queue 0.3.1, apogee-testing 0.1.1 (from their GitHub tags).

## 0.2.0 (2026-11-21)

### Requirements

- Needs grav 1.10.1 or later (its cycle collector fix) and apogee-queue 0.3.0
  (job events carry their payloads; workers pause through the shared cache);
  uses grav-db.

### Added

- Wait times: each job's (from when it was pushed), each queue's average and
  longest, and the estimated wait for a job queued now; the longest one in the
  headline.
- Per-queue totals (processed, failed, average run time) beside the sizes.
- Tags: jobs carry their tags; search jobs by tag; monitored tags keep their
  jobs past the recent ones.
- Job detail with the payload, for recent and failed jobs.
- Batches with their progress.
- Pause, continue and restart workers from the page.
- Alerts when a queue's estimated wait passes a threshold
  (`alertWhenWaitExceeds`, `checkWaits`, `monitor`).
- Silenced job classes: counted, kept out of the job list.
- `DatabaseDashboardStore`: the record in SQLite, MySQL or PostgreSQL.

### Performance

- The Redis store applies each job event in one round trip (a Lua script that
  merges the record and updates every count) instead of six to eight, and
  reads in pipelines: 2,000 jobs recorded while they run took about 920 ms
  instead of 1,620 ms (while recording more), and the page's stats about a
  millisecond instead of two.

### Changed

- `DashboardStore` is one `record(DashboardEvent)` per job event, plus the
  reads; a custom store implements the new interface.

## 0.1.0 (2026-11-20)

The first release. Needs grav 1.10.0 or newer.

- `QueueDashboard`: records the queue's events (jobs queued, run, released
  and failed, with run times; workers starting and stopping), and shows
  them: a self-contained page and a JSON API, with failed jobs retried or
  forgotten from it, behind a gate (this machine only by default).
- `RedisDashboardStore` (shared by workers and the web app) and
  `MemoryDashboardStore`.
