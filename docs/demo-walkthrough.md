# Demo Walkthrough (15 minutes max)

Screen-recording script following the assignment's required sequence (Tasks
28–32). Timings are targets. Have these port-forwards open before recording:
Argo CD (8080), Grafana (3000), Prometheus (9090), api-service (8000),
chaos-service (8002).

Pre-flight:
```bash
kubectl get applications -n gitops      # all Synced + Healthy
kubectl get pods -n app                 # all Running
```

## 0:00–2:00 — Argo CD: all Applications Sync + Healthy (Task 28)
- Open the Argo CD UI (`https://localhost:8080`).
- Show `root-app` and its children: `monitoring-app`, `sealed-secrets-app`,
  `dev-app`, `staging-app`, `prod-app`, `webhook-listener-app` — all green.
- Briefly open `root-app` to show the App-of-Apps tree.
- [Insert Screenshot: Argo CD Applications Synced + Healthy]

## 2:00–5:00 — Canary gate auto-revert (Task 29)
- Explain: push a deliberately broken image → `release-build` runs → bot commit
  updates the dev overlay → Argo CD deploys → canary polls
  `job:http_error_rate:ratio_rate5m` → on breach it pushes a revert commit.
- Show the GitOps repo history with both the deploy and the revert bot commits:
  ```bash
  git -C major-project-gitops log --oneline | grep -E 'chore\(deploy\)|chore\(revert\)'
  ```
- [Insert Screenshot: canary auto-revert commit in GitOps history]

## 5:00–9:00 — Experiment 1: pod kill under load + Grafana (Task 30)
- Start load (`hey -z 3m -c 20 ... /task`).
- Show the Grafana dashboard (API Service + Kubernetes Health rows).
- `kubectl delete pod -n app -l app=api-service` and narrate recovery:
  HPA replica tracking, request/error rate dip and recovery.
- [Insert Screenshot: Grafana during pod kill]

## 9:00–12:00 — Grafana alert firing + resolution for chaos-service (Task 31)
- Trigger chaos errors: `curl -X POST http://localhost:8002/chaos/errors`.
- Show the chaos-service error rate crossing 5% and the Grafana alert firing;
  show the webhook-listener receiving the Firing payload
  (`kubectl logs -n monitoring deploy/webhook-listener`).
- Stop chaos: `curl -X POST http://localhost:8002/chaos/stop`; show the Resolved
  notification.
- Reference the captured payloads in `docs/evidence/`.

## 12:00–15:00 — GitOps structure: App-of-Apps + ApplicationSet (Task 32)
- Walk `apps/`: `root-app.yaml` → children; the sync-wave annotations.
- Open `apps/applicationset.yaml`: one template → dev/staging/prod.
- Show `overlays/dev|staging|prod` differing only by namespace, image pin, and
  the ServiceMonitor namespace patch.
- Close on `kubectl get applications -n gitops` all green.

## Notes for the recording
- Ensure the committed dashboard reorg (`bea136e`) is pushed and synced so
  Grafana shows the 6-row layout.
- If demoing live, narrate the worker RED panels reading zero (queue consumer —
  real signal is "Worker - Tasks Processed").
- Keep each segment tight; 15 minutes is the hard ceiling.
