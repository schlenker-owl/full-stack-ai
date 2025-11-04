# Universal AI Foundry — Full-Stack AI Tech Stack

> **Purpose:** Internal documentation of our end-to-end stack for AI-centric products.  
> **Scope:** Client apps (React Native + React) → Backend API (Go + GraphQL/Relay + Ent + Postgres + S3) → Specialized AI Service (FastAPI + Python libs) → AWS (App Runner, RDS, S3).  
> **Audience:** Engineers, SRE, product/tech leadership.  
> **Note:** This top-level README is *narrative only*—no code. Deep technical details live under `docs/`.

---

## Table of Contents
1. [Philosophy & Principles](#philosophy--principles)
2. [High-Level Architecture](#high-level-architecture)
3. [Stack Overview](#stack-overview)
   - [Frontend](#frontend)
   - [Backend](#backend)
   - [Specialized AI Service](#specialized-ai-service)
4. [Environments & Deployment](#environments--deployment)
5. [AWS Infrastructure (at a glance)](#aws-infrastructure-at-a-glance)
6. [Configuration & Secrets](#configuration--secrets)
7. [Observability & SLOs](#observability--slos)
8. [Security & Compliance](#security--compliance)
9. [Cost & Scaling](#cost--scaling)
10. [Operations (Runbooks)](#operations-runbooks)
11. [Roadmap](#roadmap)
12. [Related Documentation](#related-documentation)

---

## Philosophy & Principles

- **End-to-end ownership:** One accountable owner per product spanning client → backend → AI.  
- **Production bias:** Prefer managed services (App Runner, RDS, S3) to minimize ops toil.  
- **Clear interfaces:** GraphQL (Relay) for app → API; versioned REST for API → AI.  
- **Privacy by design:** Least-privilege IAM, KMS encryption, strict PII handling.  
- **Observability first:** Traces, metrics, logs from day one; SLOs with error budgets.  
- **Continuous evaluation:** Offline golden sets + online A/B to measure AI quality.  
- **Reproducibility:** Infra as Code, consistent make/task workflows, documented promotion.

---

## High-Level Architecture

```mermaid
flowchart LR
  %% Client
  subgraph Client
    RN["React Native (iOS via TestFlight)"]
    WEB["React (Web)"]
  end

  %% Edge
  subgraph Edge
    CF["CloudFront CDN"]
  end

  %% Backend
  subgraph Backend
    AR["App Runner (Backend Endpoint)"]
    API["Go API (GraphQL + Relay)"]
    ENT["go-ent ORM"]
    PG["RDS PostgreSQL"]
    S3["S3 Buckets"]
  end

  %% AI
  subgraph AI
    AIAR["App Runner (AI Endpoint)"]
    AIFast["FastAPI (REST)"]
    PyLibs["Python AI Libraries"]
    ExtAPIs["SOTA AI APIs"]
  end

  RN -->|HTTPS| CF
  WEB -->|HTTPS| CF
  CF --> AR --> API
  API -->|Resolvers| ENT --> PG
  API -->|S3 SDK| S3
  API -->|Internal REST| AIAR --> AIFast --> ExtAPIs
  AIFast -->|Pre/Post-process| PyLibs
````

### End-to-End Request Sequence (Happy Path)

```mermaid
sequenceDiagram
  participant Client as "Client (Mobile/Web)"
  participant API as "GraphQL API (Go)"
  participant DB as "Postgres (RDS)"
  participant AI as "AI Service (FastAPI)"
  participant Ext as "External AI API"

  Client->>API: GraphQL query/mutation (Relay)
  API->>DB: Read/Write via go-ent
  API->>AI: POST /v1/generate
  AI->>Ext: Invoke selected model/API
  Ext-->>AI: Streamed/chunked tokens
  AI-->>API: JSON + metadata (latency, tokens)
  API-->>Client: GraphQL response (Relay store update)
```

---

## Stack Overview

### Frontend

* **Mobile:** React Native (TypeScript) distributed via **TestFlight**.
* **Web:** React (TypeScript) containerized and deployed on **AWS App Runner** (fronted by CloudFront).
* **API contract:** **GraphQL** with **Relay** conventions (Node IDs, Connections, persisted queries recommended).
* **Files:** Direct S3 uploads via pre-signed URLs (from backend).
* **State:** Relay store + minimal local storage.

### Backend

* **Runtime:** Go.
* **Data access:** **go-ent** (schema, migrations, typed queries).
* **API:** **GraphQL**; resolvers orchestrate DB and AI calls.
* **Storage:** **PostgreSQL (RDS)**; **S3** for file/object storage.
* **Deployment:** Docker container on **AWS App Runner** (VPC connector).
* **Internal traffic:** Private REST to AI service.
* **AuthZ/N:** OIDC/JWT, per-resolver policy checks, rate limiting.

### Specialized AI Service

* **Framework:** **FastAPI (Python)**.
* **Responsibilities:**

  * Normalize requests across SOTA AI providers (adapter layer).
  * Pre/post-processing (chunking, templating, safety/guardrails, scoring).
  * Token/cost accounting and latency metrics.
* **Libraries:** Best-of-breed Python AI/ML libraries.
* **Deployment:** Docker on **AWS App Runner** (internal endpoint).
* **Endpoints:** Versioned REST (e.g., `/v1/generate`, `/v1/embed`, `/v1/moderate`).
* **Research mode:** Separate Python-only repos for experiments.

---

## Environments & Deployment

* **Environments:** `dev`, `staging`, `prod`.
* **Promotion policy:** PR → CI checks → staging deploy → smoke/e2e → manual approval → prod.
* **Images:** Built in CI and pushed to ECR; App Runner services track image tags.
* **Networking:** App Runner VPC Connectors for private access to RDS/S3; CloudFront fronts public endpoints.

---

## AWS Infrastructure (at a glance)

* **Compute:** App Runner services: `web`, `api`, `ai`.
* **Data:** RDS PostgreSQL (encrypted, backups), S3 buckets (uploads/raw/public) with KMS + lifecycle.
* **Edge & DNS:** Route53 → CloudFront → App Runner (web/api).
* **Identity & Secrets:** IAM least-privilege roles; AWS Secrets Manager for DB creds & API keys.
* **Observability:** CloudWatch + OpenTelemetry; optional Grafana for dashboards.

---

## Configuration & Secrets

* **Config order:** Env vars → local `.env` (dev only) → Secrets Manager (cloud).
* **Common keys:** `GRAPHQL_URL`, `AI_SERVICE_URL`, `RDS_SECRET_ARN`, `S3_BUCKET_UPLOADS`, `JWT_ISSUER/JWKS_URL`, provider API keys.
* **Security:** No secrets in code or Docker build args; rotate on schedule; audit access.

---

## Observability & SLOs

* **Traces:** OTel propagation from client → API → AI; include token and latency metrics.
* **Metrics:** RED/USE per service; cost counters (tokens/$) on AI paths.
* **Logs:** Structured JSON with correlation IDs; PII/PHI never logged.
* **Draft SLOs:**

  * API p95 ≤ **800 ms** (no AI) / ≤ **2.5 s** (with AI).
  * AI endpoint success rate ≥ **99%** monthly.
  * Error budget alerts with paging policy.

---

## Security & Compliance

* **AuthN/Z:** OIDC JWT; per-tenant / per-org scoping; rate limits & abuse detection.
* **Data protection:** TLS everywhere; KMS on S3/RDS; pre-signed S3 uploads; clear retention policy.
* **IAM:** Scoped roles per service; no wildcards; separate CI/runtime/read-only roles.
* **Audit:** Record auth events, admin actions, data export.
* **Compliance posture:** SOC2-friendly controls; DSAR/SAR export pathway.

---

## Cost & Scaling

* **Primary drivers:** App Runner compute, RDS size/IO, S3 egress, AI API usage.
* **Controls:** Budgets & anomaly alerts; token caps/throttles; input truncation; caching; nightly scale-down in non-prod; S3 lifecycle to IA/Glacier.
* **Scale plan:** Horizontal scale on App Runner; async offload for long AI tasks; read replicas when needed.

---

## Operations (Runbooks)

* **AI timeouts:** Stream partials; exponential backoff; circuit breakers on upstream.
* **DB pool exhaustion:** Tune pool, fix N+1, consider read replicas.
* **S3 upload issues:** Renew pre-signed URL; verify IAM & clock skew; check bucket policy.
* **App Runner rollback:** Revert to previous image; validate env/secret bindings; compare config drift.
* **Secrets rotation:** Update in Secrets Manager; restart services; verify health probes.

---

## Roadmap

* Persisted GraphQL queries for Relay.
* AI evaluator & guardrails service.
* Optional vector search (pgvector/OpenSearch) for RAG projects.
* Canary deploys with request mirroring.
* Synthetic probes for auth, AI, and uploads.

---

## Related Documentation

This README intentionally avoids code. See:

* `docs/frontend/README.md` — RN/React conventions, Relay patterns, accessibility, performance.
* `docs/backend/README.md` — GraphQL schema guidelines, ent patterns, pagination, auth.
* `docs/ai-service/README.md` — API contracts, provider adapters, evaluation, guardrails.
* `docs/infra/README.md` — AWS topology, App Runner/VPC connectors, IaC structure.
* `docs/observability/README.md` — OTel setup, dashboards, SLOs & alerts.
* `docs/security/README.md` — Threat model, IAM policy standards, data handling.

