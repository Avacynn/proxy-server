# high-cpu-audit — Overview

**Parent audit:** in-conversation audit on 2026-06-07 (Sunday 03:01 high-CPU alert).
**Branch:** `feature/high-cpu-audit`
**Repo root:** `/home/jeabs/Projects/proxy-server/`

## Root cause summary

The Sunday 3am weekly maintenance cron (`docker compose pull && docker compose up -d --force-recreate`) recreates all 50 gluetun containers simultaneously. Each container runs OpenVPN init (TLS + iptables) at startup — 50 concurrent inits on a 12-core host drove load to 50.24. During the ~60s window until tunnels come up, the Go proxy silently falls back to `endpoints[0]` (a dead tunnel), returning 502 to all callers instead of 503.

## Key shared files

- `main.go` — Go proxy; `VPNPool`, `StartHealthChecks`, `runHealthCheck`, `GetNextEndpoint`, `GetRandomEndpoint`, `handleProxy`
- `docker-compose.yml` — 50 vpn-* services, each `cpus: 0.50`, `mem_limit: 256m`
- `CLAUDE.md` — documents the weekly cron entry (root crontab on VPS)
- `scripts/staggered-restart.sh` — **does not yet exist**; subtask 01 creates it

## Subtask build order (all independent — run in parallel)

| File | Subtask | Findings resolved |
|------|---------|-------------------|
| `01_staggered-restart.md` | Create batched restart script + update CLAUDE.md cron docs | Critical: mass simultaneous recreation; Critical: cpus thundering herd; Important: pull compounds spike |
| `02_silent-fallback.md` | Return 503 when no active endpoints | Important: `GetNextEndpoint`/`GetRandomEndpoint` silent fallback at `main.go:282-283` and `main.go:300-301` |
| `03_healthcheck-tuning.md` | Limit concurrent health probes + add startup jitter | Important: thundering herd in `runHealthCheck` at `main.go:345-370`; Nice-to-have: no jitter at `main.go:573` |

## Findings coverage map

| Finding | Severity | Home subtask |
|---------|----------|--------------|
| Mass simultaneous recreation → complete proxy outage | Critical | `01_staggered-restart.md` |
| `cpus:0.50` × 50 > 12 cores during mass restart | Critical | `01_staggered-restart.md` (addressed by batching) |
| Health-check thundering herd on startup | Important | `03_healthcheck-tuning.md` |
| `docker compose pull` before `--force-recreate` compounds spike | Important | `01_staggered-restart.md` |
| `GetNextEndpoint`/`GetRandomEndpoint` silent fallback | Important | `02_silent-fallback.md` |
| Health-check no jitter | Nice-to-have | `03_healthcheck-tuning.md` |

No findings pulled for decision or held for dedicated session.

## Cross-cutting constraints

- Port is always `os.Getenv("PORT")` — never hardcode.
- Binary output is `main` in app root — never commit it.
- `go build ./...` must pass cleanly after any Go change.
- The 50-endpoint invariant must remain 1:1 between `main.go` and `docker-compose.yml` — subtask 01 does not change the endpoint count.
- The VPS cron lives in root's crontab on `root@213.199.48.131`. Subtask 01 creates the script in-repo and updates CLAUDE.md documentation; the operator must update the crontab manually (out of scope for automated deploy).
