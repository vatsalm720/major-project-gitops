# Architecture Overview

## System

A task-queue platform of three FastAPI services plus Redis, deployed to a local
`kind` cluster, delivered entirely through Argo CD, observed with
kube-prometheus-stack, and exercised with chaos experiments.

```
                         ┌─────────────── kind cluster (1 cp + 2 workers) ───────────────┐
                         │                                                                │
  developer ── git push ─┼─► GitHub ──► CI (self-hosted runner)                           │
                         │                │ build+push localhost:5000, kind load          │
                         │                │ bot commit image tag ──► gitops repo          │
                         │                ▼                                               │
                         │   Argo CD (ns: gitops) ──watches──► major-project-gitops       │
                         │       │ reconciles                                             │
                         │       ├──► ns: app        api / worker / chaos / redis          │
                         │       ├──► ns: app-staging (same base, staging overlay)         │
                         │       ├──► ns: app-prod    (same base, prod overlay)            │
                         │       └──► ns: monitoring  Prometheus / Grafana / Alertmanager  │
                         │                                                                │
  client ── HTTP ────────┼─► api-service ──enqueue──► Redis ──BLPOP──► worker-service      │
                         │        ▲ /metrics              ▲ /metrics        ▲ /metrics     │
                         │        └──── Prometheus scrapes via ServiceMonitor ────────────┘
                         └────────────────────────────────────────────────────────────────┘
```

## Components

### Application (namespace `app`)
- **api-service** — `Deployment`, 2–6 replicas under HPA. Enqueues tasks to Redis,
  serves results. Exposes RED metrics + `http_requests_per_second` (custom HPA metric).
- **worker-service** — `Deployment`, 2 replicas. `BLPOP` consumer; exposes
  `worker_tasks_processed_total`. (HTTP RED metrics exist but stay 0 — it serves
  no business HTTP traffic.)
- **chaos-service** — `Deployment`, 2 replicas. Failure injector; exposes
  `chaos_active{chaos_mode}`.
- **redis** — `StatefulSet`, 1 replica, PVC `redis-data` (1Gi, `standard` StorageClass).

### Delivery (namespace `gitops`)
- **Argo CD** (Helm) — App-of-Apps rooted at `apps/root-app.yaml`. Auto-sync +
  self-heal + prune on every Application.
- **ApplicationSet** — generates `dev-app`/`staging-app`/`prod-app` from one template.

### Observability (namespace `monitoring`)
- **kube-prometheus-stack** (Helm via `monitoring-app`): Prometheus, Alertmanager,
  Grafana, kube-state-metrics, node-exporter, Prometheus Operator.
- **prometheus-adapter** (Helm): serves `http_requests_per_second` to the HPA via
  `custom.metrics.k8s.io`.
- **ServiceMonitor** CRDs (one per service) drive scraping — no static scrape configs.
- **PrometheusRule** CRDs: SLO recording rules + alerts.
- **Grafana dashboard** provisioned from a ConfigMap by the sidecar.
- **webhook-listener** — receives Grafana alert notifications (Phase 5 twist).

### Security / policy (namespace `app`)
- **NetworkPolicy**: default-deny-ingress + allow-redis (api/worker→6379) +
  allow-prometheus (monitoring→8000). Enforced by kindnet (verified).
- **ResourceQuota**: 2 CPU / 1Gi memory requests.
- **SealedSecrets**: only encrypted secrets in git; controller decrypts in-cluster.

## Namespacing and environments

| Namespace | Owner | Contents |
|---|---|---|
| `app` | `dev-app` (ApplicationSet) | dev workloads + Redis + policies + quota + SealedSecret |
| `app-staging` | `staging-app` | base workloads, staging overlay (image pin + SM namespace patch) |
| `app-prod` | `prod-app` | base workloads, prod overlay |
| `monitoring` | `monitoring-app` | Prometheus/Grafana/adapter/rules/dashboard/webhook |
| `gitops` | bootstrap + Argo CD | Argo CD, all Application objects |

Each overlay patches the base `ServiceMonitor` `namespaceSelector` to its own
namespace and pins images to a SHA tag (`bb225dc`).

## Request and metric flow

1. `POST /task` → api-service increments `http_requests_total`, pushes to Redis.
2. worker-service `BLPOP`s, processes, increments `worker_tasks_processed_total`.
3. `GET /result/{id}` → api-service returns the result.
4. Prometheus scrapes `/metrics` on each pod every 15s via ServiceMonitors.
5. Recording rules compute `job:http_error_rate:ratio_rate{5m,1h,6h}` and the
   30-day error budget; alerts fire on multi-burn-rate, p95 latency, and chaos.
6. Grafana renders the dashboard and (separately from Alertmanager) posts
   firing/resolved notifications to the webhook-listener.

## Key design decisions
- **ApplicationSet instead of a static `app-app.yaml`** — one template yields all
  three environments, satisfying both the App-of-Apps and multi-overlay requirements.
- **`imagePullPolicy: Never` + `kind load`** — avoids needing the nodes to reach a
  registry; images are side-loaded into containerd.
- **Liveness and readiness both probe `/health`** — simple, but means a Redis
  outage can cascade api-service into CrashLoop (documented in the Exp-2 runbook
  as a known tradeoff).
- **Quota sized to 1Gi** — deliberately tight so the Phase 2.5 scale-out rejection
  is demonstrable.
