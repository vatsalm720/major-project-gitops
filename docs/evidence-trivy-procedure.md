# Evidence Procedure — Trivy CVE Fail → Fix (Task 15)

Goal: capture one CI run where the Trivy job **fails** on a CRITICAL CVE, and one
run where it **passes** after the fix.

**Validated locally** (2026-06-12): adding `pyyaml==5.3.1` to api-service produces
exactly **1 CRITICAL — CVE-2020-14343 (PyYAML, fixed in 5.4)**, and
`trivy image --exit-code 1 --severity CRITICAL` returns **exit 1**. The base
`python:3.12-alpine` image has **0 CRITICAL**, so this is the only finding and the
result is deterministic. Removing the line returns exit 0.

## 1. File to modify
`major-project/services/api-service/requirements.txt`

Current:
```
fastapi==0.115.12
uvicorn[standard]==0.34.2
redis==5.2.1
prometheus-client==0.21.1
```

## 2. Vulnerable package to introduce
Append:
```
pyyaml==5.3.1
```

## 3. Commit / PR sequence (fail run)
```bash
cd major-project
git checkout -b chore/trivy-cve-demo
printf 'pyyaml==5.3.1\n' >> services/api-service/requirements.txt
git add services/api-service/requirements.txt
git commit -m "test(ci): introduce CVE-2020-14343 (pyyaml 5.3.1) to demonstrate Trivy gate"
git push -u origin chore/trivy-cve-demo
# open a PR: chore/trivy-cve-demo -> main   (triggers pr-validation)
```
PR validation runs on the self-hosted runner: `test` → `build` → **`trivy`**.
The `trivy` job's "Scan api-service" step fails (exit 1).

## 4. Screenshots to capture (fail)
- **[Insert Screenshot: CI pipeline failing due to Trivy CVE]** — the Actions run
  with the `Trivy Scan` job red.
- Open the failed step log showing `CVE-2020-14343 ... CRITICAL ... PyYAML 5.3.1`.

## 5. Fix and re-run (pass run)
```bash
git checkout services/api-service/requirements.txt   # drop the vulnerable line
# (or: edit the file to remove the pyyaml==5.3.1 line)
git commit -am "fix(ci): remove vulnerable pyyaml; Trivy gate passes"
git push
# the same PR re-runs pr-validation automatically
```

## 6. Screenshots to capture (pass)
- **[Insert Screenshot: CI pipeline passing after Trivy fix]** — the same PR's
  later run with `Trivy Scan` green and all jobs passing.

## 7. Expected behaviour
| State | `trivy` job | PR |
|---|---|---|
| pyyaml 5.3.1 present | "Scan api-service" exits 1 (CVE-2020-14343 CRITICAL) | red / blocked |
| pyyaml removed | all three scans exit 0 | green |

## 8. Rollback / cleanup
The fix commit already restores `requirements.txt`. After capturing both runs,
either merge the PR (it's back to the clean state) or close it and delete the
branch:
```bash
git push origin --delete chore/trivy-cve-demo
```

> Note: keep this on a feature branch + PR (it targets `main`, which the PR
> workflow validates). Do not merge the vulnerable commit to `main`.
