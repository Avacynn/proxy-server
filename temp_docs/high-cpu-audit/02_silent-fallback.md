# 02 — Return 503 when no active endpoints

## What

When all 50 VPN endpoints are inactive (e.g. during mass restart), `GetNextEndpoint` and `GetRandomEndpoint` silently return `endpoints[0]` (a dead tunnel). This causes every proxied request to 502 with no diagnostic signal. Change the behaviour so callers receive a clean HTTP 503 instead.

## Scope

**In scope:**
- `main.go` only
- Change `GetNextEndpoint` and `GetRandomEndpoint` to return `(*VPNEndpoint, bool)` where `bool=false` signals no active endpoint
- Update `handleProxy` to check the bool and write HTTP 503 with a descriptive message when false
- No change to `GetEndpointByName` (it already returns nil for unknown names, which `handleProxy` handles correctly)

**Out of scope:**
- Any changes to `docker-compose.yml`, scripts, or CLAUDE.md
- Changing the health-check logic (that's subtask 03)

## Key files / entry points

Read these before implementing:
- `main.go:262-305` — `GetNextEndpoint` and `GetRandomEndpoint` — the two functions to change
- `main.go:382-435` — `handleProxy` — the switch/case that calls these functions and must be updated
- `main.go:117-123` — `VPNEndpoint` struct — for context

## Notes from planning

**Finding resolved:**
- Important: "`GetNextEndpoint` fallback to `endpoints[0]` is silent" — `main.go:282-283` and `main.go:300-301` — *recommendation: return 503 instead of silently proxying through `endpoints[0]`.*

**Signature change:**
```go
func (p *VPNPool) GetNextEndpoint() (*VPNEndpoint, bool)
func (p *VPNPool) GetRandomEndpoint() (*VPNEndpoint, bool)
```

In `handleProxy`, after calling these, check the bool:
```go
endpoint, ok := s.vpnPool.GetNextEndpoint()
if !ok {
    http.Error(w, "No VPN endpoints available", http.StatusServiceUnavailable)
    return
}
```

The `random` branch in `handleProxy` (line ~420) calls `GetRandomEndpoint`; update it the same way. The `specific` branch uses `GetEndpointByName` which returns nil — that path is already handled correctly with a 400.

**Do not** change the `specific` strategy path — it intentionally uses a named endpoint regardless of active status (caller explicitly requested it).

**Log the zero-active event** in `GetNextEndpoint`/`GetRandomEndpoint` (one `log.Printf` when all are down) so it appears in `/status` logs and systemd journal.

## Depends on

None.

## UI / follow-ups deferred

Nothing.
