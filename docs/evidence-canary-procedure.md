# Evidence Procedure — Canary Auto-Revert (Task 2.7)

Goal: capture the bot **revert commit** in the GitOps repo produced when the
canary gate detects a bad deployment.

## Canary logic (verified against `release-build.yml`)
The `canary-gate` job (after `update-gitops` deploys the new image):
1. waits for Argo CD to roll api-service to the new SHA, then `kubectl rollout status`;
2. warms up 20 `POST /task` requests and waits 15s for a Prometheus scrape;
3. polls `job:http_error_rate:ratio_rate5m` five times at 60s intervals,
   threshold `0.01` (1%); empty/NaN is treated as 0% (denominator is guarded);
4. on any poll > 1% it clones the GitOps repo, rewrites `overlays/dev/kustomization.yaml`
   back to the previous SHA, commits, pushes to `main`, and fails the run.

A clean **pass** run already exists (Release Build #3 — all 5 polls `0 < 0.01`).
To produce the **revert** evidence you must ship an image that makes api-service
return 5xx so the error rate crosses 1%.

## 1. Change to introduce (a deliberately broken api-service)
Edit `major-project/services/api-service/app/main.py` so `POST /task` always
returns 500 while `/health` stays healthy (so the pod still becomes Ready and the
rollout completes — the canary only polls after a successful rollout):

```python
# in the POST /task handler, before the normal logic:
from fastapi import HTTPException
raise HTTPException(status_code=500, detail="canary-break: forced failure")
```
This is counted by the metrics middleware as `http_errors_total{status_code="500"}`,
which feeds `job:http_error_rate:ratio_rate5m`. Background load (~9 req/s already
hitting `/task`) plus the 20 warm-up requests guarantee the rate exceeds 1%.

## 2–4. PR / merge / trigger flow
```bash
cd major-project
git checkout -b chore/canary-break-demo
# apply the edit above
git commit -am "test(ci): force api-service 5xx to trigger canary auto-revert"
git push -u origin chore/canary-break-demo
# open PR -> main ; pr-validation runs (it will pass: unit/integration may still pass,
#   but you can merge regardless — the canary runs on the MAIN push)
# merge the PR to main  ->  triggers release-build (build/push/load/update-gitops/canary-gate)
```
Failure condition: the broken image deploys, the canary warm-up + live traffic
produce 5xx, the first poll reads > 0.01, and the gate triggers.

## 5. Screenshots to capture
- **[Insert Screenshot: canary auto-revert bot commit in GitOps repo]** — the
  GitOps repo commit history showing the revert commit by `GitOps Bot`.
- The failed `Canary Gate` job log line: `CANARY GATE TRIGGERED — error rate ... exceeds 0.01`.
- (Optional) Argo CD rolling api-service back to the previous SHA.

## 6. Expected revert commit message
```
chore(revert): rollback api-service to sha-<PREV_SHA> (canary gate triggered)
```
author `GitOps Bot <bot@major-project.local>`.

## 7. Expected commit location
GitOps repo `major-project-gitops`, branch `main`, file
`overlays/dev/kustomization.yaml` (the three `newTag` values reset to `<PREV_SHA>`).

## 8. Expected Argo CD behaviour
After `update-gitops` the broken SHA deploys (api-service starts returning 5xx).
After the revert commit, Argo CD syncs `overlays/dev` back to `<PREV_SHA>` and
api-service returns to healthy. The workflow run ends red (gate failed).

## Cleanup
```bash
# restore the clean api-service
git checkout main -- services/api-service/app/main.py   # on your demo branch
git push origin --delete chore/canary-break-demo
```
Confirm the live dev overlay is back on the good SHA and api-service is healthy:
```bash
kubectl get deploy api-service -n app -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

> Caution: this deploys a broken image to the live cluster for the ~1–2 minutes
> until the canary reverts. Run it deliberately, not during other evidence capture.
