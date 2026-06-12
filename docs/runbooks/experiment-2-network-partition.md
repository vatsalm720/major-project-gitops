# Runbook — Experiment 2: Redis Network Partition

> Status: **executed.** Observations below are from a real run.

## Hypothesis
Blocking ingress to Redis for ~3 minutes will break the worker-service Redis loop
and api-service Redis access. worker-service should recover automatically when
connectivity returns. The Grafana error-rate panel should show the disruption.

## How to run (correct method)
NetworkPolicy is **additive-allow only** — there is no deny rule. To block, remove
the allow, do not add a "deny". Set the Redis policy to no allowed ingress:

```yaml
# base/netpol-allow-redis.yaml (temporary, committed via GitOps)
spec:
  podSelector: { matchLabels: { app: redis } }
  policyTypes: [Ingress]
  ingress: []        # <-- no allow rules => all ingress to Redis denied
```
Commit + push; Argo CD syncs. After ~3 minutes, restore the original ingress rule
and push again. `from: []` is **wrong** — an empty `from` means "all sources",
which opens Redis instead of closing it.

> selfHeal note: because the block goes through git, Argo CD will not fight it.
> A `kubectl`-only edit would be reverted by selfHeal within minutes.

## Symptom (what the alert/dashboard says)
- worker logs stop emitting `Processed task_id=...`.
- api-service `/health` returns `redis: down`; readiness/liveness fail.
- Error-rate panel rises; api-service pods begin restarting.

## First look (panels, in order)
1. **API Service → Error Rate** and **Platform Overview → 5m Error Rate**.
2. **Kubernetes Health → HPA Replica Tracking** (the HPA reacts to lost metrics).
3. **Worker Service → Tasks Processed** (drops to zero during the partition).

## Root cause determination (PromQL)
```promql
up{namespace="app"}                                   # which targets are down
job:http_error_rate:ratio_rate5m                      # api error rate
rate(worker_tasks_processed_total[1m])                # worker throughput -> 0
kube_pod_container_status_restarts_total{namespace="app"}   # api restarts climbing
```
Connectivity probe from inside the namespace:
```bash
WK=$(kubectl get pod -n app -l app=worker-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n app "$WK" -- python3 -c \
 'import socket;socket.create_connection(("redis.app.svc.cluster.local",6379),3)'
```

## What was observed (real run)
- **api-service**: `/health` failed (it checks Redis) → **liveness** probe killed
  the container → **Exit 137** (graceful "Shutting down" then SIGKILL; *not* OOM)
  → CrashLoopBackOff.
- **Secondary cascade**: CrashLooping pods stopped serving `/metrics` → the HPA
  lost its CPU metric (`<unknown>`, `FailedGetResourceMetric`) and scaled up
  toward 6 → that exhausted the 1Gi ResourceQuota → `exceeded quota` FailedCreate.
- **worker-service**: Redis calls failed during the window and it **recovered
  automatically** once connectivity returned (reconnect + drain backlog).
- **Recovery**: after restoring the policy, Redis became reachable, api-service
  stopped crashlooping, the HPA scaled back to 2 (freeing quota), the pending
  rollout completed, and Argo CD returned to Synced + Healthy. No manual data fix
  was required.

## Remediation if auto-heal fails
- Restore the original `allow-redis` ingress via git; confirm Argo CD synced.
- If api-service stays CrashLooping after Redis is back, it is usually the quota
  deadlock: let the HPA scale to 2 (frees 128Mi) so the rollout's surge pod can
  schedule, or remove any stray pods consuming quota.
- Verify: `kubectl exec` Redis PING from a worker pod returns `+PONG`.

## Prevention
- **Split probes**: liveness should check only process liveness; readiness should
  check Redis. This stops a dependency outage from killing api-service pods.
- Add a Redis client retry/backoff in api-service so transient loss degrades
  gracefully instead of failing `/health`.
- Leave quota headroom (or raise the quota) so an HPA scale-up during an incident
  does not wedge rollouts.

## Evidence
- GitOps commits: `14a7b63` (isolate), `b024be9` (restore).
- [Insert Screenshot: Grafana error-rate panel during the partition window]
