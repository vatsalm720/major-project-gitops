# Phase 3 — CI/CD Pipeline

## Scope
Two GitHub Actions workflows on a self-hosted runner: PR validation and the
release build with a bot commit and a Prometheus-gated canary.

Both live in `major-project/.github/workflows/`.

## PR workflow — `pr-validation.yml` (PRs targeting `release`)

| Job | What it does |
|---|---|
| `test` | venv + `ruff` lint, `pytest tests/unit`, then `docker compose up --wait` and `pytest tests/test_integration.py` |
| `build` | builds all three images with `VERSION`/`GIT_SHA`/`BUILD_DATE` (no push) |
| `trivy` | `trivy image --exit-code 1 --severity CRITICAL` on each image — fails the PR on any CRITICAL CVE (Task 15) |
| `manifest-validation` | `kube-score score --exit-one-on-warning` on `deploy/k8s/app` (Task 16) |
| `cleanup` | removes built images from the runner |

`kube-score` runs with explicit `--ignore-test` flags
(`container-image-tag`, `container-image-pull-policy`, `pod-networkpolicy`,
`container-security-context-user-group-id`, `pod-probes`). These suppress checks
that conflict with the assignment's own design choices (e.g. `imagePullPolicy:
Never`, shared `/health` probe, UID 1000). The gate therefore means
**zero warnings except those explicitly justified** — documented honestly here so
the "zero warnings" claim is not overstated.

## Release workflow — `release-build.yml` (push to `release`)

| Job | What it does |
|---|---|
| `build-and-push` | builds + tags with the short SHA + pushes to `localhost:5000` |
| `load-images` | re-pulls and `kind load`s each image into the cluster nodes |
| `update-gitops` | clones the GitOps repo, records the previous SHA, rewrites `overlays/dev/kustomization.yaml`, commits `chore(deploy): update api-service to sha-<SHORT_SHA>` and pushes (Task 18) |
| `canary-gate` | waits for Argo CD to roll the new image, then polls `job:http_error_rate:ratio_rate5m` every 60s ×5; if it exceeds 1% it pushes a revert commit restoring the previous SHA and fails the run (Task 2.7) |

The canary poll treats an empty/NaN Prometheus result as 0% (the recording rule's
denominator is guarded), so a zero-traffic window cannot trip a false revert.

## Commands / validation
```bash
# bot commits land in the GitOps repo
git -C ../major-project-gitops log --oneline | grep -E 'chore\(deploy\)|chore\(revert\)'
```

## Evidence
- Available: both workflow files; the bot-commit format in GitOps history; a
  passing Canary Gate run (Release Build #3, all 5 polls under threshold).
- Trivy fail→fix **procedure validated locally**: adding `pyyaml==5.3.1` to
  `services/api-service/requirements.txt` yields exactly 1 CRITICAL
  (CVE-2020-14343) and `trivy image --exit-code 1 --severity CRITICAL` returns
  exit 1; removing it returns 0. Capture steps in the Phase 4 evidence procedure.
- Outstanding: the two screenshots (Trivy fail/fix run; canary auto-revert).

## Status
Pipelines implemented and functional; the two evidence artifacts above are the
remaining gaps.
