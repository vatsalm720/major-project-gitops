# Runbook — Experiment 1: api-service Pod Kill Under Load

> Status: **executed.** The experiment was run (load test + simultaneous pod
> delete, recovery observed in Grafana) and evidence captured. Transcribe the
> real numbers into the Observations section below from the captured evidence —
> do not invent values.

## Hypothesis
With a load generator running against api-service, deleting all api-service pods
simultaneously should cause a brief availability dip, after which the Deployment
recreates pods and the HPA returns replicas to the load-appropriate level.
In-flight requests at the moment of deletion are expected to fail.

## How to run
```bash
# 1. drive load (laptop)
hey -z 3m -c 20 -m POST -H 'Content-Type: application/json' \
  -d '{"text":"load"}' http://localhost:8000/task     # via api-service port-forward
# 2. delete all api-service pods at once
kubectl delete pod -n app -l app=api-service --wait=false
# 3. observe recovery
kubectl get pods -n app -l app=api-service -w
```

## Symptom (what the alert/dashboard says)
- Platform Overview: API Request Rate dips; 5m Error Rate spikes briefly.
- Possible `HighP95Latency` warning during recovery.
- Argo CD: dev-app may show Progressing during pod recreation.

## First look (panels, in order)
1. **Kubernetes Health → HPA Replica Tracking** — current vs desired replicas.
2. **API Service → Request Rate / Error Rate** — magnitude and duration of the dip.
3. **Platform Overview → API Request Rate / 5m Error Rate** — user-visible impact.

## Root cause determination (PromQL)
```promql
# available replicas over time
kube_deployment_status_replicas_available{namespace="app", deployment="api-service"}
# targets up
up{namespace="app", job="api-service"}
# error rate during the window
job:http_error_rate:ratio_rate5m
# scrape gaps / recovery
rate(http_requests_total{job="api-service"}[1m])
```

## Remediation if auto-heal fails
- The Deployment recreates pods automatically. If replicas stay at 0:
  - `kubectl describe rs -n app -l app=api-service` — check for `FailedCreate`.
  - If `exceeded quota`, the namespace is at the ResourceQuota ceiling — see the
    Experiment 3 / quota notes; free headroom or let the HPA scale down.
  - `kubectl get events -n app --sort-by=.lastTimestamp | tail`.

## Prevention
- Keep `minReplicas >= 2` and `PodDisruptionBudget minAvailable: 1`.
- Spread replicas across nodes (pod anti-affinity is already set).
- Ensure readiness gating so traffic only routes to ready pods.

## Observations (transcribe from captured evidence)
- Time to first recovery: _transcribe from run_
- In-flight requests dropped: _transcribe from run_
- HPA behaviour vs expectation: _transcribe from run_
- [Insert Screenshot: Grafana during Experiment 1 pod kill] (captured)
