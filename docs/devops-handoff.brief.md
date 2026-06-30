# DevOps Handoff — Brief (server + install)

What to provision and install to run the platform. Full detail: [devops-handoff.md](./devops-handoff.md).

---

## What you're running
- **One Docker image** for all app services; a `SERVICES` env var picks which run in each container.
- Phase 1 = **single application host** with Docker Compose. No Kubernetes needed.
- TLS terminates at the **gateway** (only public container); everything else is internal-only.

---

## Servers to get

### 1. Application host (1 server)
Runs the gateway + app containers + stateless engines.

| Resource | Spec |
|---|---|
| vCPU | 8–12 (min 4) |
| RAM | 24–32 GB |
| Disk | 100+ GB SSD |
| OS | Ubuntu LTS (modern Linux) |
| GPU | Not needed (cloud LLMs) |

### 2. Data plane (separate host or managed instances — recommended)

| Component | Spec |
|---|---|
| PostgreSQL 16 | 4 vCPU / 16 GB / 100 GB SSD; backups + PITR. Managed preferred. |
| TimescaleDB (IoT, separate cluster) | Postgres 16 + Timescale ext; 4 vCPU / 16 GB / SSD |
| Redis 7 | 1–2 vCPU / 4 GB; AOF on + replica |
| Qdrant | 2 vCPU / 4–8 GB; SSD volume |
| S3 / MinIO | Managed S3 in prod, or MinIO host with ample disk + versioning |

---

## What to install

### On the application host
- **Docker Engine + Docker Compose v2** (that's effectively all that's needed — the app ships as one image).

### Datastores (managed service or self-installed)
- **PostgreSQL 16** (shared cluster, one DB + role per service)
- **TimescaleDB** (Postgres 16 + Timescale extension) — separate cluster for IoT
- **Redis 7** (AOF enabled, + replica in prod)
- **Qdrant** (vector store)
- **S3 or MinIO** (object storage, bucket-per-service)

### Self-hosted engines (run as containers on the internal network, not public)
- **LiteLLM** (LLM access) · **Carbone** (PDF/DOCX rendering) · **Unstructured** (OCR/parsing) · **Node-RED** (IoT ingestion edge)
- Stateful (need their own pg/redis/volume, back them up): **Nango**, **Documenso**, **Authentik**
- Optional, only if needed: **Ollama** (local LLM, needs GPU host), **n8n**

---

## Must-have config / ops
- **All config via environment variables**; prod secrets from a secret manager (never in code/images).
- Each service uses its **own DB role / `DATABASE_URL`**.
- Health: `GET /health` (liveness), `GET /ready` (gates traffic).
- **Backups:** per-DB Postgres dumps + PITR, Redis AOF + replica, S3/MinIO versioning, and back up Nango/Documenso/Authentik DBs.

---

## Scale-out later (config only, same image)
Add gateway replicas → ai-plane replicas → split hottest service out → split other groups. No re-architecting.
