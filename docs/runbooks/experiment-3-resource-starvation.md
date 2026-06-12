# Runbook — Experiment 3: Resource Starvation (chaos memory mode)

> Status: **procedure defined; experiment not yet executed.** Observation fields
> are PENDING and must be filled from a real run.

## Hypothesis
Triggering chaos-service memory mode (allocates 50 MB every 10 s) should drive the
pod past its 256Mi memory limit, causing the kubelet OOMKiller to terminate it.
The Deployment restarts it; the PodDisruptionBudget keeps at least one replica
available throughout; and the VPA (recommendation mode) raises its suggested
memory after the event.

## How to run
```bash
# trigger memory chaos on a chaos-service pod
kubectl port-forward -n app svc/chaos-service 8002:8000 &
curl -s -X POST http://localhost:8002/chaos/memory
# let it run ~5 minutes, then stop
curl -s -X POST http://localhost:8002/chaos/stop
```

## Symptom (what the alert/dashboard says)
- `ChaosActive` warning alert fires (chaos_active == 1 for > 2 min).
- Chaos Service panels show rising memory then a restart; Platform Overview
  "Chaos Active" card flips to ACTIVE.

## First look (panels, in order)
1. **Kubernetes Health → Pod Memory Usage** — chaos-service climbing toward its limit.
2. **Chaos Service → Request Rate / Error Rate** — restart gap.
3. **Platform Overview → Chaos Active**.

## Root cause determination (PromQL)
```promql
container_memory_working_set_bytes{namespace="app", pod=~"chaos-service.*"}
kube_pod_container_status_restarts_total{namespace="app", pod=~"chaos-service.*"}
kube_pod_container_status_last_terminated_reason{namespace="app", reason="OOMKilled"}
chaos_active{job="chaos-service"}
```
VPA recommendation after the event:
```bash
kubectl describe vpa chaos-service -n app        # Target / Lower Bound / Upper Bound
```

## Remediation if auto-heal fails
- Stop the chaos mode: `POST /chaos/stop` (or delete the affected pod; the
  Deployment recreates it).
- Confirm the PDB kept availability: `kubectl get pdb chaos-service -n app`
  (`ALLOWED DISRUPTIONS` should remain ≥ 0 and at least one pod Running).
- If both replicas OOM simultaneously, check the memory limit and the node's
  available memory; consider applying the VPA's recommended request/limit.

## Prevention
- Right-size memory requests/limits using the VPA recommendation.
- Keep `minAvailable: 1` PDB and ≥2 replicas so one OOM never takes the service down.
- Bound the failure: the 50MB/10s allocator is intentional; in real services cap
  caches and add memory backpressure.

## Observations (PENDING — fill from a real run)
- OOMKilled observed (yes/no, time to kill): _PENDING_
- Restart time: _PENDING_
- VPA recommendation (Target memory): _PENDING_
- PDB prevented total unavailability (yes/no): _PENDING_
- [Insert Screenshot: VPA recommendation output after memory chaos]
- [Insert Screenshot: Grafana Pod Memory Usage during Experiment 3]
