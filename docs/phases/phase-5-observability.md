# Phase 5 — Observability

## Scope
Custom metrics, ServiceMonitor-only scraping, SLO/error-budget alerting, the
Grafana dashboard as code, and the Grafana alerting lifecycle.

## Implementation

### Instrumentation (Tasks 23, 24)
`major-project/services/*/app/main.py` expose:
- `request_duration_seconds` histogram, buckets `0.1, 0.5, 1, 2, 5`.
- `http_requests_in_flight` gauge.
- `http_errors_total` counter labelled by `status_code`.
- `http_requests_total` (drives the HPA custom metric).
- chaos-service additionally: `chaos_active` gauge with a `chaos_mode` label.
- worker-service additionally: `worker_tasks_processed_total`.

> worker-service exposes the HTTP RED metrics but they stay at 0 — it is a queue
> consumer with no business HTTP traffic. Its real throughput signal is
> `worker_tasks_processed_total` (surfaced on the dashboard as "Worker - Tasks
> Processed").

### Scraping (Task 25)
`monitoring-app` installs kube-prometheus-stack (86.1.0) and prometheus-adapter
(5.3.0) via Helm, values committed to this repo. Scrape targets are
`ServiceMonitor` CRDs (`base/*/service-monitor.yaml`), 15s interval, each scoped
to its own namespace via an overlay patch. No static `scrape_configs`.

### SLO + alerting (Tasks 26, 27)
`monitoring/prometheus-rules/slo-api-service.yaml`:
- recording rules `job:http_error_rate:ratio_rate{5m,1h,6h}` (denominators
  guarded against 0/0 NaN) and `job:http_error_budget_remaining:30d`.
- `HighErrorBudgetBurnRate` — page severity, fires when both 1h and 6h error
  rates exceed 5× the 1% budget (multi-burn-rate).
- `HighP95Latency` — warning, p95 > 500ms.

`monitoring/prometheus-rules/chaos-active.yaml`:
- `ChaosActive` — warning, `chaos_active == 1` for more than 2 minutes.

### Dashboard as code (Task 2.13)
`monitoring/grafana/major-project-dashboard-configmap.yaml` — a ConfigMap
labelled `grafana_dashboard: "1"`, provisioned by the Grafana sidecar (no manual
UI creation). Organised into six rows: Platform Overview, API Service, Worker
Service, Chaos Service, Kubernetes Health, Reliability & Chaos. Covers RED per
service, HPA current vs desired, pod CPU/memory vs requests/limits, the chaos
timeline overlaid on error rate, and the error-budget gauge + trend.

### Grafana alerting lifecycle (Task 2.14)
`monitoring/kube-prometheus-stack-values.yaml` provisions a Grafana contact point
(webhook) + notification policy + the `ChaosServiceHighErrorRate` rule (>5%
chaos-service error rate). The receiver is
`monitoring/webhook-listener/webhook-listener.yaml` (a small Python listener,
deployed by `webhook-listener-app`). Firing and Resolved payloads were captured.

## Commands / validation
See `docs/validation-guide.md` (Phase 5 section) for the Prometheus queries and
rule/target checks.

## Evidence
- Available: all rules/dashboard/values in repo; **`docs/evidence/grafana-alert-firing.json`** and **`grafana-alert-resolved.json`** captured (Task 2.14).
- Note: the 6-row dashboard reorganisation is committed (`bea136e`); ensure it is
  pushed so Argo CD reloads it before the demo.

## Status
Done.
