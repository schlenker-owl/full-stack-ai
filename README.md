# AI App Foundry — Full-Stack AI Tech Stack

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
4. [Authentication & Identity (Clerk)](#authentication--identity-clerk)
5. [Environments & Deployment](#environments--deployment)
6. [AWS Infrastructure (at a glance)](#aws-infrastructure-at-a-glance)
7. [Configuration & Secrets](#configuration--secrets)
8. [Observability & SLOs](#observability--slos)
9. [Security & Compliance](#security--compliance)
10. [Cost & Scaling](#cost--scaling)
11. [Operations (Runbooks)](#operations-runbooks)
12. [Roadmap](#roadmap)
13. [Related Documentation](#related-documentation)

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

  %% Edge & Identity
  subgraph Edge
    CF["CloudFront CDN"]
    CLERK["Clerk (Auth)"]
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
  RN -. signup/login .-> CLERK
  WEB -. signup/login .-> CLERK

  CF --> AR --> API
  API -->|Resolvers| ENT --> PG
  API -->|S3 SDK| S3
  API -->|Internal REST| AIAR --> AIFast --> ExtAPIs
  AIFast -->|Pre/Post-process| PyLibs

  %% Token verification path
  API -. verify JWT .-> CLERK
````

### End-to-End Request Sequence (Happy Path)

```mermaid
sequenceDiagram
  participant Client as "Client (Mobile/Web)"
  participant Clerk as "Clerk (Auth)"
  participant API as "GraphQL API (Go)"
  participant DB as "Postgres (RDS)"
  participant AI as "AI Service (FastAPI)"
  participant Ext as "External AI API"

  Client->>Clerk: Sign-up / Sign-in (hosted UI or SDK)
  Clerk-->>Client: Session established (Clerk token)
  Client->>API: GraphQL query/mutation (Authorization: Bearer <Clerk token>)
  API->>Clerk: Verify token / fetch claims (server verify)
  API->>DB: Read/Write via go-ent (tenant-aware)
  API->>AI: POST /v1/generate (+ user/tenant context)
  AI->>Ext: Invoke selected model/API
  Ext-->>AI: Streamed / chunked tokens
  AI-->>API: JSON + metadata (latency, token usage)
  API-->>Client: GraphQL response (Relay store update)
```

---

## Stack Overview

### Frontend

* **Mobile:** React Native (TypeScript) distributed via **TestFlight**.
* **Web:** React (TypeScript) containerized and deployed on **AWS App Runner** (fronted by CloudFront).
* **Auth:** **Clerk** for sign-up/sign-in, session management, and JWTs.
* **API contract:** **GraphQL** with **Relay** conventions (Node IDs, Connections, persisted queries recommended).
* **Files:** Direct S3 uploads via pre-signed URLs (issued by backend after auth).
* **State:** Relay store + minimal local storage.

### Backend

* **Runtime:** Go.
* **Data access:** **go-ent** (schema, migrations, typed queries).
* **API:** **GraphQL**; resolvers orchestrate DB and AI calls.
* **AuthN/Z:** **Clerk** token verification server-side; per-resolver policy checks and rate limits.
* **Storage:** **PostgreSQL (RDS)**; **S3** for file/object storage.
* **Deployment:** Docker container on **AWS App Runner** (VPC connector).
* **Internal traffic:** Private REST to AI service.

### Specialized AI Service

* **Framework:** **FastAPI (Python)**.
* **Responsibilities:** Normalize provider calls, pre/post-process, safety/guardrails, token & latency accounting.
* **Libraries:** Best-of-breed Python AI/ML libraries.
* **Deployment:** Docker on **AWS App Runner** (internal endpoint).
* **Endpoints:** Versioned REST (e.g., `/v1/generate`, `/v1/embed`, `/v1/moderate`).
* **Research mode:** Separate Python-only repos for experiments.

---

## Authentication & Identity (Clerk)

* **User Identity Source of Truth:** Clerk manages sign-up, sign-in, sessions, MFA (if enabled), and user profiles across web and mobile.
* **Frontend Integration:**

  * Use Clerk’s React/React Native SDKs for hosted components and session handling.
  * Client attaches the Clerk session token to every GraphQL request (`Authorization: Bearer …`).
* **Backend Verification:**

  * API verifies incoming Clerk tokens using Clerk’s server SDK/JWKS before executing resolvers.
  * Resolver context includes user id, org/tenant, roles/permissions, and rate-limit keys derived from claims.
* **Service Calls:**

  * API passes minimal user context to the AI service (no raw PII).
  * The AI service trusts the API (private network) and enforces its own auth check (internal allowlist or short-lived service token).
* **Webhooks (optional):**

  * Clerk → Backend webhook can provision app-specific records (e.g., create `User`/`Organization` rows on first sign-in).
* **Logout/Revocation:**

  * Client clears session; server honors revocation by rejecting expired/invalid tokens on each request.

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
* **Auth-related keys:** `CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`, `CLERK_JWT_ISSUER`, `CLERK_JWKS_URL`.
* **Common service keys:** `GRAPHQL_URL`, `AI_SERVICE_URL`, `RDS_SECRET_ARN`, `S3_BUCKET_UPLOADS`, `S3_PUBLIC_BASEURL`.
* **Security:** No secrets in code or Docker build args; rotate on schedule; audit access.

---

## Observability & SLOs

* **Traces:** OTel propagation from client → API → AI; include token/cost and latency metrics.
* **Metrics:** RED/USE per service; cost counters (tokens/$) on AI paths.
* **Logs:** Structured JSON with correlation IDs; Clerk user/org ids recorded as pseudonymous identifiers (no raw PII).
* **Draft SLOs:**

  * API p95 ≤ **800 ms** (no AI) / ≤ **2.5 s** (with AI).
  * AI endpoint success rate ≥ **99%** monthly.
  * Error budget alerts with paging policy.

---

## Security & Compliance

* **AuthN/Z:** Clerk-verified JWTs; per-tenant/org scoping; policy checks in resolvers; request-level rate limits.
* **Data protection:** TLS everywhere; KMS on S3/RDS; pre-signed S3 uploads; clear retention policy.
* **IAM:** Scoped roles per service; no wildcards; separate CI/runtime/read-only roles.
* **Audit:** Record auth events, admin actions, and data export.
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
* **Auth incidents:** Invalidate sessions at Clerk; rotate backend secrets; verify JWKS cache and token TTLs.
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

* `FRONTEND.md` — RN/React conventions, Relay patterns, accessibility, performance.
* `BACKEND.md` — GraphQL schema guidelines, ent patterns, pagination, authorization.
* `AI_SERVICE.md` — API contracts, provider adapters, evaluation, guardrails.
* `AWS_INFRA.md` — AWS topology, App Runner/VPC connectors, IaC structure.
* `OBSERVABILITY.md` — OTel setup, dashboards, SLOs & alerts.
* `SECURITY.md` — Threat model, IAM policy standards, data handling.
* `AUTH.md` — **Clerk setup**, token verification, webhook provisioning, session lifecycles.

