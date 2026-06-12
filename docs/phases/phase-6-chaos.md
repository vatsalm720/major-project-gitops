# Phase 6 — Chaos Engineering

## Scope
Three structured chaos experiments, a runbook per experiment, and a 15-minute
demo recording.

## Experiments

| Experiment | Method | Status |
|---|---|---|
| 1 — pod kill under load | load with `hey`/`k6`, delete all api-service pods | **Executed** — evidence captured; numbers to transcribe |
| 2 — network partition | block ingress to Redis for 3 min, then restore | **Executed** — observations recorded |
| 3 — resource starvation | trigger chaos memory mode, watch OOM/VPA/PDB | **Not yet run** |

Runbooks live in `docs/runbooks/`:
- `experiment-1-pod-kill.md` — executed; transcribe numeric observations from captured evidence.
- `experiment-2-network-partition.md` — complete, with real observations.
- `experiment-3-resource-starvation.md` — procedure only (observation fields pending).

### Experiment 2 — what actually happened (summary)
Setting the Redis ingress policy to no-allow (correct method:
`ingress: []`, not `from: []`) genuinely partitioned Redis. Observed cascade:
api-service `/health` failed (it checks Redis) → **liveness** probe killed the
container (Exit 137, graceful shutdown then SIGKILL — not OOM) → CrashLoop;
missing `/metrics` scrapes made the HPA lose its CPU metric (`<unknown>`) and
scale up toward 6, which **exhausted the 1Gi ResourceQuota** (`exceeded quota`
FailedCreate). worker-service failed its Redis calls but **recovered
automatically** once connectivity returned. After restoring the policy and the
HPA scaling back to 2, the platform self-healed end to end; Argo CD returned to
Synced + Healthy. Full detail in the Experiment 2 runbook.

Key finding for the writeup: liveness using the dependency-aware `/health`
endpoint turns a Redis outage into an api-service CrashLoop. Splitting liveness
(process-only) from readiness (dependency-aware) is the recommended fix.

## Demo video (Task 2.17)
Script in `docs/demo-walkthrough.md`. Recording not yet produced.

## Evidence
- Available: Experiment 2 commits (`14a7b63` isolate, `b024be9` restore) and the
  recorded incident analysis.
- Outstanding: Experiments 1 and 3 runs (and their Grafana screenshots / VPA
  output); the 15-minute demo recording.

## Status
Partial — Experiments 1 and 2 executed; Experiment 3 and the demo recording remain.
