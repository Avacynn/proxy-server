# 03 — Health-check concurrency limit + startup jitter

## What

`runHealthCheck` spawns all 50 probe goroutines simultaneously with no concurrency cap. During a mass restart this fires a thundering herd of HTTPS connections on top of an already-overloaded host. Additionally, the health-check ticker fires every 60s on the dot with no per-endpoint jitter, creating a recurring 50-goroutine wave. Fix both.

## Scope

**In scope:**
- `main.go` only
- Add a semaphore in `runHealthCheck` to limit concurrent probes to 10 at a time
- Add per-endpoint startup jitter in `StartHealthChecks` so probes stagger across the 60s window rather than all firing at once

**Out of scope:**
- Changing the 60s interval value (leave it as-is; just add jitter)
- Any changes to `docker-compose.yml`, scripts, or CLAUDE.md
- Changing the fallback/503 logic (that's subtask 02)

## Key files / entry points

Read these before implementing:
- `main.go:333-370` — `StartHealthChecks` and `runHealthCheck` — the two functions to change
- `main.go:566-573` — `main()` call site: `server.vpnPool.StartHealthChecks(60 * time.Second)`

## Notes from planning

**Findings resolved:**
- Important: "Health-check thundering herd fires on top of startup storm" — `main.go:345-370` — *recommendation: add a semaphore to limit concurrent probes so at most ~10 run at once instead of all 50.*
- Nice-to-have: "Health-check interval has no jitter" — `main.go:573` — *recommendation: add `time.Duration(rand.Intn(15)) * time.Second` jitter to the initial offset per-endpoint.*

**Semaphore pattern** — limit to 10 concurrent probes inside `runHealthCheck`:
```go
sem := make(chan struct{}, 10)
for _, ep := range p.endpoints {
    wg.Add(1)
    go func(ep *VPNEndpoint) {
        sem <- struct{}{}
        defer func() { <-sem }()
        defer wg.Done()
        // ... existing probe logic
    }(ep)
}
```

**Startup jitter** — in `StartHealthChecks`, before the immediate `p.runHealthCheck()` call, optionally sleep a small random offset. However, the existing design runs one immediate pass on startup (by design — "endpoints start optimistically Active and converge after the first pass ~10s", per CLAUDE.md). Do not add jitter to the immediate startup pass. Add jitter only to the ticker-based subsequent passes.

A simpler and equally effective approach: add a small random sleep inside `runHealthCheck` per goroutine *before* the probe, spread uniformly over 0–15s. This staggers the actual network calls within each cycle without changing the cycle timing:
```go
time.Sleep(time.Duration(rand.Intn(15)) * time.Second)
```
Place this after acquiring the semaphore slot so the jitter doesn't defeat the concurrency cap.

**`rand.Seed` is already called** in `main()` at line 567 — no need to add it.

**Verify `go build ./...` passes** after the change.

## Depends on

None.

## UI / follow-ups deferred

Nothing.
