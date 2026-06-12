# Requirement-to-Implementation Matrix

This matrix maps every numbered task and hard constraint in the assignment to the
actual files in the two repositories, the implementation status, and the evidence
state. It is the audit that the rest of the documentation is built on.

Repositories:
- **app repo** = `major-project` (service source, Dockerfiles, K8s reference manifests, CI)
- **gitops repo** = `major-project-gitops` (Argo CD source of truth)

Status legend: **Done** (implemented and verified) · **Partial** (implemented, caveat noted) · **Not done** (missing) · **Pending evidence** (implemented, artifact not yet captured).

---

## Phase 1 — Containerise and compose

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| 1 | Multi-stage Dockerfile per service; final stage alpine, non-root UID 1000 | `major-project/services/{api,worker,chaos}-service/Dockerfile` | Done | Dockerfiles use `python:3.12-alpine` builder → runtime, `adduser -S -u 1000`, `USER 1000` | — |
| 2 | HEALTHCHECK calling `/health` via wget | same Dockerfiles | Done | `HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 CMD wget -qO- .../health` | — |
| 3 | docker-compose with Redis + `depends_on: service_healthy` | `major-project/docker-compose.yml` | Done | redis has compose healthcheck; api/worker/chaos `depends_on: redis: condition: service_healthy` | Screenshot of the broken-Redis cascade (optional demo) |
| 4 | `docker-compose.override.yml` (dev bind-mounts) + `docker-compose.prod.yml` (limits 256m/0.5cpu) | `major-project/docker-compose.override.yml`, `docker-compose.prod.yml` | Done | override mounts source for `--reload`; prod sets `mem_limit`/`cpus` | — |
| 5 | LABEL version/git-sha/build-date; verify with `docker inspect` | same Dockerfiles (`LABEL version/git-sha/build-date`, `ARG`s) | Done | build args wired in both CI workflows | `docker inspect` output capture |
| 2.2 | chaos `POST /chaos/{mode}` + `GET /chaos/status` | `major-project/services/chaos-service/app/main.py` | Done | routes `POST /chaos/memory`, `/chaos/errors`, `/chaos/latency`, `/chaos/stop`, `GET /chaos/status` (mode-in-path) | — |

## Phase 2 — Kubernetes beyond basics

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| 6 | kind cluster, ≥2 worker nodes | `major-project/deploy/kind/kind-cluster.yml` | Done | 1 control-plane + 2 workers; `kubectl get nodes` shows 3 Ready | — |
| 7 | Namespaces app/monitoring/gitops | gitops `platform/namespaces/app.yaml`, `bootstrap/namespaces/{gitops,monitoring}.yaml` | Done | `kubectl get ns` | — |
| 8 | Redis StatefulSet + PVC | gitops `base/redis/statefulset.yaml`, `service.yaml`, `service-headless.yaml` | Done | `volumeClaimTemplates` 1Gi `standard`; PVC `redis-data-redis-0` Bound | — |
| 9 | HPA api 2–6, CPU 60% + custom `http_requests_per_second` | gitops `base/api-service/hpa.yaml`; adapter `monitoring/prometheus-adapter-values.yaml` | Done | `kubectl describe hpa` shows `cpu …/60%` + `http_requests_per_second …/10` (verified non-`<unknown>`) | — |
| 10 | PDB minAvailable 1 per service | gitops `base/{api,worker,chaos}-service/pdb.yaml` | Done | `ALLOWED DISRUPTIONS=1` verified; node-drain tested with screenshots | — |
| 11 | NetworkPolicy deny-all + allow api→Redis; reject from outside | gitops `base/netpol-deny-all.yaml`, `netpol-allow-redis.yaml`, `netpol-allow-prometheus.yaml` | Done | enforcement verified live (chaos-svc→redis times out; worker→redis `+PONG`); rejection screenshot captured | — |
| 12 | VPA recommendation-only on chaos | gitops `base/chaos-service/vpa.yaml` (updateMode Off); controller `major-project/deploy/k8s/vpa/*` | Done | `updateMode: "Off"`; VPA recommendation captured after memory chaos | — |
| 2.5 | ResourceQuota 2 CPU / 1Gi; HPA fails gracefully past quota | gitops `base/resourcequota.yaml` | Done | quota `requests.cpu:2, requests.memory:1Gi`; **real `exceeded quota` FailedCreate events captured** during staging/prod rollout | Screenshot of the event in `kubectl get events` for submission |

## Phase 3 — CI/CD pipeline

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| 13 | Lint + unit-test all services (PR) | `major-project/.github/workflows/pr-validation.yml` (job `test`) | Done | ruff + pytest unit + integration via compose | — |
| 14 | Build images, no push (PR) | same (job `build`) | Done | builds 3 images with VERSION/GIT_SHA/BUILD_DATE | — |
| 15 | Trivy scan, fail on CRITICAL; one failed + one fixed in history | same (job `trivy`, `--exit-code 1 --severity CRITICAL`) | Partial | Trivy job present and enforcing | **Commit history showing a failed scan then a fixed scan** not confirmed |
| 16 | kube-score/kube-linter, fail on warning (PR) | same (job `manifest-validation`, `--exit-one-on-warning`) | Partial | kube-score runs in CI | Runs with documented `--ignore-test` flags → "zero warnings *except* the listed suppressions" (see Phase 3 doc) |
| 17 | Build, tag git-SHA, push to localhost:5000 (main/release) | `major-project/.github/workflows/release-build.yml` (job `build-and-push`, `load-images`) | Done | pushes to `localhost:5000`, `kind load` into nodes | — |
| 18 | Bot commit updates image tag in GitOps overlay | same (job `update-gitops`) | Done | commit msg `chore(deploy): update api-service to sha-<SHORT_SHA>`; edits `overlays/dev/kustomization.yaml` | — |
| 2.7 | Canary gate: poll Prometheus 60s×5; auto-revert if error rate >1% | same (job `canary-gate`) | Partial | polls `job:http_error_rate:ratio_rate5m`, pushes revert commit on breach | **Screenshot of a real auto-revert** triggered by a broken image not captured |

## Phase 4 — GitOps with Argo CD

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| 2.8 | Repo layout apps/base/overlays/monitoring | gitops repo root tree | Done | layout matches (uses ApplicationSet in place of a static `app-app.yaml`) | — |
| 19 | App-of-Apps root managing ≥2 children | gitops `apps/root-app.yaml` → `apps/` (monitoring-app, sealed-secrets-app, webhook-listener-app, applicationset) | Done | `kubectl get applications -n gitops` shows all Synced+Healthy | — |
| 20 | Sync waves -1 CRDs/secrets, 0 ns, 1 workloads | wave annotations: monitoring-app/-1, sealed-secrets-app/-1, applicationset/0, base workloads/1, hpa/2 | Done | annotations present in the files | — |
| 21 | Drift detection demo + auto-heal | `automated.selfHeal: true` on all apps | Done | OutOfSync + auto-heal demonstrated; screenshots captured | — |
| 22 | ApplicationSet generating dev/staging/prod | gitops `apps/applicationset.yaml` | Done | generates `dev-app`/`staging-app`/`prod-app`; all Synced+Healthy | — |
| 2.10 | SealedSecrets; no raw secrets; fresh-cluster decrypt | gitops `apps/sealed-secrets-app.yaml`, `overlays/dev/app-credentials-sealed.yaml` | Partial | controller installed via Helm; only encrypted SealedSecret in repo | **Fresh-cluster decrypt requires backing up/restoring the controller key** — not yet validated (see caveat in Deployment Guide) |

## Phase 5 — Observability

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| 23 | Client libs + `request_duration_seconds` (0.1/0.5/1/2/5), `http_requests_in_flight`, `http_errors_total{status_code}` | `major-project/services/*/app/main.py` | Partial | api + chaos expose all with data; **worker exposes them but they stay 0** (queue consumer, no business HTTP) | — |
| 24 | `chaos_active` gauge + `chaos_mode` label | `major-project/services/chaos-service/app/main.py` | Done | `chaos_active{chaos_mode=...}` scraped | — |
| 25 | kube-prometheus-stack via Helm; all scrape via ServiceMonitor CRDs | gitops `monitoring/kube-prometheus-stack-values.yaml`; `base/*/service-monitor.yaml` | Done | Helm via `monitoring-app`; ServiceMonitors per service, 15s, namespace-scoped; no static scrape_configs | — |
| 26 | SLO rules: 99% / p95<500ms; error-budget + multi-burn-rate page alert | gitops `monitoring/prometheus-rules/slo-api-service.yaml` | Done | recording rules `ratio_rate5m/1h/6h`, `budget_remaining:30d`; alerts `HighErrorBudgetBurnRate` (1h&6h>5×), `HighP95Latency` | — |
| 27 | Warning alert chaos_active==1 >2m | gitops `monitoring/prometheus-rules/chaos-active.yaml` | Done | `ChaosActive` alert `for: 2m` | — |
| 2.13 | Grafana dashboard as code (ConfigMap) with RED/HPA/saturation/chaos timeline/error-budget | gitops `monitoring/grafana/major-project-dashboard-configmap.yaml` | Done | 6-row dashboard, sidecar label `grafana_dashboard:"1"`, provisioned via `monitoring-app` | Dashboard reorg currently **uncommitted** in working tree |
| 2.14 | Grafana (not Alertmanager) → webhook on chaos err >5%; capture Firing + Resolved | gitops `monitoring/kube-prometheus-stack-values.yaml` (grafana.alerting), `monitoring/webhook-listener/webhook-listener.yaml` | Done | **`docs/evidence/grafana-alert-firing.json` + `grafana-alert-resolved.json` captured** | — |

## Phase 6 — Chaos engineering

| # | Requirement | File(s) | Status | Evidence available | Evidence missing |
|---|---|---|---|---|---|
| Exp 1 | Pod kill under load | runbook `docs/runbooks/experiment-1-pod-kill.md` | Done | executed (load + pod delete, recovery in Grafana); evidence captured | transcribe numeric observations into the runbook |
| Exp 2 | Network partition to Redis | runbook `docs/runbooks/experiment-2-network-partition.md` | Done | **executed**; real cascade + recovery observed (gitops commits `14a7b63`/`b024be9`) | Grafana panel screenshots during the window |
| Exp 3 | Resource starvation (memory chaos) | runbook `docs/runbooks/experiment-3-resource-starvation.md` | Not done | procedure documented | **Experiment not yet run** — OOM/VPA/PDB observations pending |
| 2.16 | Runbooks for all 3 in `docs/runbooks/` | `docs/runbooks/*.md` | Partial | Exp 2 complete; Exp 1 & 3 are procedure-only until executed | — |
| 2.17 | 15-min demo video | — | Not done | walkthrough script written (`docs/demo-walkthrough.md`) | Recording |

## Hard constraints

| Constraint | Status | Notes |
|---|---|---|
| GitOps only (no `kubectl apply -f`) | Done | Cluster state from Argo CD; `kubectl` used read-only + drift demo + chaos experiments only |
| No hardcoded secrets | Done | Only `SealedSecret` (encrypted) committed; no plaintext secret in either repo |
| Manifest quality gate (kube-score zero warnings in CI) | Partial | Enforced via `--exit-one-on-warning`, with documented `--ignore-test` suppressions |
| Dashboard as code | Done | JSON in ConfigMap, provisioned by sidecar; no manual UI creation |

## Consolidated gaps (honest)

**Implementation gaps**
- Phase 6 Experiment 3 (resource starvation) not yet run as a structured experiment; its runbook is procedure-only until executed. Experiments 1 and 2 are done.
- Worker HTTP RED metrics exist but never populate (worker serves no business HTTP) — by design, documented.
- VPA recommender controller is installed from `major-project/deploy/k8s/vpa/` (not Argo-managed) — minor deviation from GitOps-only.
- SealedSecrets fresh-cluster decryption is unproven without controller-key backup/restore.

**Evidence gaps (submission checklist) — remaining**
- Trivy CVE fail → fix screenshots (procedure validated; see Phase 4 section / `docs/phases/phase-3-cicd.md`).
- Canary auto-revert bot commit screenshot (procedure validated; see Phase 5 section).
- 15-minute demo video.

**Present evidence (held by author / in repo)**
- Grafana Firing + Resolved alert payloads (`docs/evidence/`).
- Real `exceeded quota` events from the staging/prod rollout (Phase 2.5).
- Argo CD OutOfSync drift, VPA recommendation, PDB node-drain, NetworkPolicy
  rejection, HPA scaling, Experiment 1 Grafana — screenshots captured (held by
  author; add to `docs/evidence/` for the submission bundle).
