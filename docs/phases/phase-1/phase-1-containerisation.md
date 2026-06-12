# Phase 1 — Containerise and Compose

## Scope
Multi-stage Docker builds, image hardening, HEALTHCHECK, Compose dependency
ordering, and the chaos endpoint contract.

## Implementation

### Multi-stage, hardened images (Tasks 1, 2, 5)
`major-project/services/{api,worker,chaos}-service/Dockerfile`, identical pattern:

- **builder stage** (`python:3.12-alpine`): `pip wheel` builds dependency wheels.
- **runtime stage** (`python:3.12-alpine`): installs only the wheels (`--no-index
  --find-links=/wheels`), copies `app/`, then drops to a non-root user created
  with `adduser -S -u 1000` and `USER 1000`.
- **labels**: `version`, `git-sha`, `build-date` from build args.
- **healthcheck**: `HEALTHCHECK --interval=15s --timeout=3s --start-period=10s
  --retries=3 CMD wget -qO- http://127.0.0.1:8000/health || exit 1`.

### Compose stack (Tasks 3, 4)
- `docker-compose.yml` — Redis + 3 services. Redis declares a compose
  healthcheck (`redis-cli ping`); the three services use
  `depends_on: redis: condition: service_healthy` so they start only after Redis
  is healthy.
- `docker-compose.override.yml` — development: bind-mounts source for `--reload`.
- `docker-compose.prod.yml` — production-like: removes mounts, adds
  `mem_limit: 256m` / `cpus: 0.5` per service.

### Chaos endpoint contract (Task 2.2)
`major-project/services/chaos-service/app/main.py`:
`POST /chaos/memory`, `POST /chaos/errors`, `POST /chaos/latency`,
`POST /chaos/stop`, `GET /chaos/status`. Mode is encoded in the path (satisfies
`POST /chaos/{mode}`); `/chaos/status` returns the active mode and elapsed time.

## Commands

```bash
# build with labels
docker build --build-arg VERSION=1.0.0 --build-arg GIT_SHA=$(git rev-parse --short HEAD) \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t localhost:5000/major-project-api-service:dev services/api-service/

# verify hardening + labels + healthcheck
docker inspect --format '{{.Config.User}}' localhost:5000/major-project-api-service:dev
docker inspect --format '{{json .Config.Labels}}' localhost:5000/major-project-api-service:dev

# compose ordering
docker compose -f docker-compose.yml up --build -d --wait
docker compose -f docker-compose.yml ps
```

## Validation
- `docker inspect` shows `User=1000`, all three labels, and the healthcheck.
- `docker compose ps` shows services as `healthy`; bringing Redis down (break its
  healthcheck) prevents the dependents from starting — the cascade.

## Evidence
- Available: Dockerfiles, compose files, chaos routes in source.
- Outstanding: `docker inspect` capture; screenshot of the broken-Redis cascade
  (optional demo for Task 3).

## Status
Done.
