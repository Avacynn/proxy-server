# proxy-server — Claude Code Guide

VPN-farm HTTP proxy. A Go service (`main.go`) round-robins outbound requests
across a farm of [gluetun](https://github.com/qdm12/gluetun) (PIA) Docker
containers so consumers get rotating egress IPs.

- **Service:** `go-proxy-server` (systemd), binds `os.Getenv("PORT")` → `9001`
- **Public:** `https://proxy.jeab.dev` · **Internal:** `http://localhost:9001`
- **API:** `GET /proxy?url=<encoded>&strategy=roundrobin|random|specific&api_key=…`,
  `GET /status` (auth). Full spec: [.github/PROXY_API_SPEC.md](.github/PROXY_API_SPEC.md)
- **Main consumer:** `ks-redeemer` (routes Kingshot game-server traffic through here).

## Deployment Constraints

- **Port:** always `os.Getenv("PORT")`. Never hardcode.
- **Binary:** build output named `main` in app root. Never commit `main`, `.env`.
- **Creds:** PIA login lives in `.env` on the VPS (`PIA_USERNAME`/`PIA_PASSWORD`),
  gitignored. `setup-vpn.sh` writes it. The 50 containers read it via `${…}`.
- **Deploy:** push to `main` → VPS auto-pulls + rebuilds + restarts.
  Run runtime/docker commands over SSH on the Main VPS (`root@213.199.48.131`);
  never hand-edit files under `/opt/reverse-proxy/apps/` on the VPS.

## The 50-Endpoint Invariant (read before touching the farm)

The endpoint set is defined in **two places that MUST stay 1:1**:

1. `main.go` → `NewVPNPool()` hardcodes **50** `VPNEndpoint`s, each pointing at
   `http://127.0.0.1:88xx` (host ports **8881–8930**).
2. `docker-compose.yml` defines **50** `vpn-*` services, each publishing
   `88xx:8888` on a unique static IP `172.22.0.x`.

`main.go` round-robins over the endpoints, but **`StartHealthChecks` probes each
tunnel every 60s** (`GET https://api.ipify.org` through the endpoint) and flips
`Active` — so `GetNextEndpoint`/`GetRandomEndpoint` automatically skip tunnels
that can't currently reach the internet (e.g. dead PIA regions, see below).
`/status` reflects the live result. Without this a dead slot returns `502`
(`proxyconnect …`) straight to the caller (e.g. ks-redeemer "all 5 attempts
blocked by game server"). Endpoints start optimistically `Active` and converge
~10s after boot, so the first few seconds post-restart may still 502.

**Verify correspondence after any change:**
```bash
grep -c '{Name: "' main.go            # → 50  (endpoints)
grep -cE '^  vpn-' docker-compose.yml # → 50  (services)
```
If you add/remove an endpoint, change BOTH files (port + static IP must be unique).

## Drift: why the farm shrank, and how it's prevented

**Do NOT trim the live set with a `profiles:` key.** From inception until
2026-06-05, 30 of the 50 services carried `profiles: ["disabled"]`, so a plain
`docker compose up -d` started only the 20 un-profiled ones. Combined with the
old weekly cron (below), the running set became a one-way ratchet that only
shrank — leaving `main.go` round-robining over 30 dead ports (mostly `502`).
The profiles were removed so **all 50 start by default**. If you ever need to
reduce capacity, drop the endpoints from `main.go` too (keep them 1:1) rather
than disabling containers behind the running proxy.

**Bring up / reconcile the full farm (over SSH):**
```bash
cd /opt/reverse-proxy/apps/proxy-server && docker compose up -d   # → all 50
docker ps --filter name=vpn- --format '{{.Names}}' | wc -l        # → 50
```

**Weekly maintenance cron** (root crontab on the Main VPS) restarts the farm
Sunday 3am to clear gluetun memory leaks. It is **reconcile-based** so it also
revives any stopped container (the old version restarted only already-running
containers — that was the ratchet):
```cron
0 3 * * 0 cd /opt/reverse-proxy/apps/proxy-server && docker compose up -d --force-recreate --remove-orphans >> /var/log/vpn-restart.log 2>&1
```

**Resource note:** idle tunnels use ~85 MiB each (cap 256 MiB); 50 ≈ ~4 GiB on
the 47 GiB host. CPU caps are `cpus: 0.50` per container (raised from `0.10` to
stop CFS throttling — see infra `VPS_PERFORMANCE_INVESTIGATION.md`).
