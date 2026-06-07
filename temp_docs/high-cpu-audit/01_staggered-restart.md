# 01 — Staggered maintenance restart script

## What

Replace the weekly cron's `docker compose up -d --force-recreate` (which restarts all 50 containers simultaneously) with a batched script that recreates containers in groups, keeping the majority of tunnels live throughout maintenance.

## Scope

**In scope:**
- Create `scripts/staggered-restart.sh` in the repo root
- Update the cron documentation in `CLAUDE.md` to replace the old one-liner with the new script invocation
- The script handles: `docker compose pull`, then batched recreation in groups of 10 with a sleep between batches

**Out of scope:**
- Actually updating the VPS crontab (operator manual step — CLAUDE.md update is sufficient as the instruction)
- Changing `docker-compose.yml` service definitions
- Any Go code changes

## Key files / entry points

- `CLAUDE.md` — the existing cron entry is in the "Weekly maintenance cron" section; replace the `0 3 * * 0` line with the new invocation
- `docker-compose.yml` — read to understand service naming (`vpn-florida`, `vpn-california`, …) so the script can target them; services are named `vpn-*`
- No existing `scripts/` directory — create it

## Notes from planning

**Findings resolved:**
- Critical: "Mass simultaneous container recreation causes complete proxy outage" — `docker-compose.yml` + cron — *recommendation: replace `docker compose up -d --force-recreate` with a loop that recreates containers in batches of 5-10 with a `sleep 10` between batches. This keeps ~40 tunnels live at all times during maintenance.*
- Critical: "`cpus: 0.50` × 50 containers = 25 CPU budget on 12-core host" — `docker-compose.yml:23` — *recommendation: addressed by batching (only ~10 containers init at once instead of 50).*
- Important: "`docker compose pull` before `--force-recreate` compounds the spike" — *recommendation: pull the image first (one `docker compose pull`), then batch-recreate; pull itself is sequential and doesn't cause the thundering herd.*

**Script design:**
- Pull the image first (`docker compose pull`), then loop over service names in groups of 10
- Get the service list dynamically: `docker compose config --services | grep '^vpn-'`
- Recreate each batch: `docker compose up -d --force-recreate --no-deps <svc1> <svc2> ...`
- Sleep 20s between batches (gives containers time to bring VPN tunnels up before the next batch tears down)
- Log batch progress with timestamps to stdout (the cron already redirects to `/var/log/vpn-restart.log`)
- Script must be executable (`chmod +x`) — note this in the file header comment, and ensure the file is created with execute permission

**CLAUDE.md update:** Replace the existing cron line with:
```
0 3 * * 0 cd /opt/reverse-proxy/apps/proxy-server && bash scripts/staggered-restart.sh >> /var/log/vpn-restart.log 2>&1
```
And add a brief note that the script batches 10 containers at a time with 20s gaps, keeping ~40 tunnels live during maintenance.

## Depends on

None.

## UI / follow-ups deferred

- The operator must manually update the VPS root crontab to point at the new script. CLAUDE.md will document the exact line to use.
