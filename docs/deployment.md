---
title: Deployment
order: 2
group: Get started
description: Getting access, the image, running one container, Docker Compose profiles, Kubernetes manifests, clusters and upgrades.
---

## Getting access

ZAANIX is available on request. [Talk to us]({{mailto}}) and we will give you access to the private image and the deployment files (`docker-compose.yml`, the Kubernetes manifests), and help you set it up.

## The image

The image is multi-architecture (`linux/amd64`, `linux/arm64`), built for every release with SBOM and provenance attestations. Each release is smoke-tested end to end (probes, auth, sandbox, upload, HITL, a full MCP handshake) before it is published.

| Tag | Meaning |
|---|---|
| `latest` | newest release |
| `1`, `1.2` | floating major / minor |
| `{{version}}` | pinned release — use this in production |

The image is `node:20-bookworm-slim`, runs as the non-root `duckuser:duckgroup`, uses `tini` as PID 1, declares volumes for `/data` and `/app/meta`, and ships the `httpfs`, `azure`, `arrow`, `iceberg`, `delta` and `excel` DuckDB extensions pre-installed under `/app/duckdb-extensions` so no network is needed at runtime. It also carries `python3` + `venv` and the ZAANIX Python SDK for [data apps](apps.html); the apps' virtualenv is created under `/data/.zaanix/apps` on first use (network access to PyPI needed once).

## One container

Run the image with the two volumes and port 4200 published, and set `JWT_SECRET`, `ENCRYPTION_KEY` and, to create the first admin at start, `ZAANIX_ADMIN_EMAIL` and `ZAANIX_ADMIN_PASSWORD`. Useful settings:

- `DUCKDB_MEMORY_LIMIT=12GB` — engine ceiling (percentage of host RAM or absolute)
- `ZAANIX_FILESYSTEM_MODE=sandboxed` — confine every user to `/data` (multi-tenant)
- `ZAANIX_ENABLE_EXTERNAL_ACCESS=true` — allow `s3://`, `gcs://`, `https://` and MotherDuck in sandboxed mode
- `ZAANIX_PUBLIC_URL=https://zaanix.example.com` — used for OIDC redirects and MCP snippets
- a container memory cap (for example 8 GB) — DuckDB spills to the temp directory when its own limit is reached

## Docker Compose

The `docker-compose.yml` that comes with access runs ZAANIX with persistent volumes and four optional profiles:

| Profile | What it adds |
|---|---|
| (none) | ZAANIX with SQLite metadata, `./data` mounted at `/data` |
| `postgres` | PostgreSQL 16 for metadata (set `DATABASE_URL` in `.env`) |
| `ollama` | Ollama, a local model for ZAANIX Bot (`COPILOT_PROVIDER=ollama`) |
| `observability` | Prometheus on :9090 |
| `agent` | ZAANIX Agent on :4300 (`AGENT_MODEL_*` in `.env`) |

`.env` holds `JWT_SECRET`, `ENCRYPTION_KEY`, the admin credentials and `POSTGRES_PASSWORD`.

Spill space is a tmpfs (`ZAANIX_SPILL_SIZE`, default 4g) and the container memory limit is `ZAANIX_MEMORY` (default 8g) with `DUCKDB_MEMORY_LIMIT` at 70% so the engine never fights the OS for the last page.

## ZAANIX Agent

The agent app is its own server and web app on port 4300, next to ZAANIX; the `agent` profile runs it. Its settings — where it finds ZAANIX, its secret, the model — are in [ZAANIX Agent](agent-app.html#configuration). The agent server is one client to ZAANIX for everyone who uses it, so raise ZAANIX's per-client rate limit for a team (`ZAANIX__server__rate_limit_per_minute=6000`).

## Kubernetes

The Kustomize manifests that come with access create a `zaanix` namespace, a ConfigMap with `zaanix.config.yaml`, a Deployment (liveness `/healthz`, readiness and startup `/readyz`, `runAsNonRoot`, read-only root filesystem, `emptyDir` spill, PVC for `/data`), a Service with `ClientIP` session affinity so SSE streams stay pinned, and Prometheus scrape annotations (`servicemonitor.yaml` for the Operator).

DuckDB engines are process-local — in-memory workspaces live in the pod. Scale vertically first; for more than one replica use PostgreSQL metadata, a RWX volume for `/data` and keep session affinity. The server-side result cache is per pod (still correct, just less warm).

## Cluster mode

For more than one node, run ZAANIX with PostgreSQL metadata, a ReadWriteMany data volume and `cluster.enabled`: each workspace's engine lives on one node and the others forward to it; scheduled work runs once. Details in [Running ZAANIX for a team](operations.html#cluster-mode).

## Reverse proxies and TLS

Terminate TLS in front (Caddy, nginx, an ingress) and set `ZAANIX_TRUST_PROXY=true` so client IPs in the audit log and rate limiter are right. WebSocket upgrades (`/api/ws/*`) and long-lived SSE responses (`/mcp/sse`, `/api/copilot/chat`) must be allowed through; keep proxy read timeouts above your `DUCKDB_QUERY_TIMEOUT_SECONDS`.

## Upgrading

Metadata migrations are additive and run automatically on start for both SQLite and PostgreSQL. Back up `/app/meta` (or the Postgres database) before a major upgrade. The data directory and any `.duckdb` workspace files are never touched by migrations.

