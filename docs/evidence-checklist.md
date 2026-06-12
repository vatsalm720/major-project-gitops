# Evidence Checklist

Tracks every submission artifact: what exists, where it is, and what is still
outstanding. Screenshots are placeholders until captured — none are fabricated.

## Repository artifacts (submission checklist)

| Item | Location | Status |
|---|---|---|
| Source code + Dockerfiles (3 services) | `major-project/services/*` | Present |
| docker-compose (3 files) | `major-project/docker-compose*.yml` | Present |
| GitOps repo with App-of-Apps + overlays | `major-project-gitops/{apps,base,overlays}` | Present |
| CI/CD workflow YAML | `major-project/.github/workflows/*` | Present |
| Manifests pass kube-score in CI | `pr-validation.yml` job `manifest-validation` | Present (with documented `--ignore-test` suppressions) |
| SealedSecret YAML, no raw secrets | `overlays/dev/app-credentials-sealed.yaml` | Present |
| Grafana dashboard JSON via ConfigMap | `monitoring/grafana/major-project-dashboard-configmap.yaml` | Present (6-row reorg committed in `bea136e`) |
| PrometheusRule YAML (SLO/error budget) | `monitoring/prometheus-rules/*` | Present |
| Runbooks for 3 experiments | `docs/runbooks/*` | Exp 2 complete; Exp 1 & 3 procedure-only |
| Root README per repo | `major-project/README.md`, `major-project-gitops/README.md` | Present |

## Evidence artifacts

| Evidence | Required by | Status | Location / action |
|---|---|---|---|
| Argo CD OutOfSync (drift) screenshot | Task 21 | **Captured** (held by author) | add to `docs/evidence/` for the bundle |
| CI Trivy CVE fail → fixed pass | Task 15 | **Outstanding** | procedure validated (pyyaml 5.3.1 → 1 CRITICAL); follow Phase 4 steps |
| VPA recommendation after memory chaos | Task 12 | **Captured** (held by author) | add to `docs/evidence/` for the bundle |
| PDB node-drain / NetworkPolicy rejection / HPA scaling | Tasks 10/11/9 | **Captured** (held by author) | add to `docs/evidence/` for the bundle |
| Canary auto-revert bot commit screenshot | Task 2.7 | **Outstanding** | procedure validated; follow Phase 5 steps |
| Grafana Firing payload | Task 2.14 | **Captured** | `docs/evidence/grafana-alert-firing.json` |
| Grafana Resolved payload | Task 2.14 | **Captured** | `docs/evidence/grafana-alert-resolved.json` |
| ResourceQuota rejection events | Task 2.5 | **Captured (live)** | `kubectl get events -n app-staging \| grep 'exceeded quota'` |
| 15-minute demo video | Task 2.17 | **Outstanding** | record per `docs/demo-walkthrough.md` |

## Outstanding work before submission
1. Capture the **Trivy fail → fix** screenshots (Phase 4 procedure — validated).
2. Capture the **canary auto-revert** screenshot (Phase 5 procedure — validated).
3. Collect the already-captured screenshots into `docs/evidence/` (drift, VPA,
   PDB drain, NetworkPolicy rejection, HPA, Experiment 1 Grafana).
4. Transcribe Experiment 1 numeric observations into its runbook.
5. (Optional) Run Experiment 3 (resource starvation) and fill its runbook.
6. Decide the SealedSecrets fresh-cluster approach (key restore vs re-seal).
7. Record the 15-minute demo video.

## Optional bonus (not implemented)
- Loki + trace-correlated logs in Grafana — not deployed.
- External Secrets Operator + Vault — not deployed (SealedSecrets used instead).
