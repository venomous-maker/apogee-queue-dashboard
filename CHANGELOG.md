# Changelog

Every release of apogee-queue-dashboard, newest first. Each is a `vX.Y.Z` tag with a GitHub
release carrying its section; the `grav` line in `grav.toml` says which
compiler versions it builds with.

## Unreleased

## 0.1.0 (2026-11-20)

The first release. Needs grav 1.10.0 or newer.

- `QueueDashboard`: records the queue's events (jobs queued, run, released
  and failed, with run times; workers starting and stopping), and shows
  them: a self-contained page and a JSON API, with failed jobs retried or
  forgotten from it, behind a gate (this machine only by default).
- `RedisDashboardStore` (shared by workers and the web app) and
  `MemoryDashboardStore`.
