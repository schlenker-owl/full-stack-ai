# Universal AI Foundry — Full-Stack AI Tech Stack

> **Purpose:** Internal documentation of our end-to-end stack for AI-centric products.  
> **Scope:** Client apps (React Native + React) → Backend API (Go + GraphQL/Relay + Ent + Postgres + S3) → Specialized AI Service (FastAPI + Python libs) → AWS (App Runner, RDS, S3).  
> **Audience:** Engineers, SRE, product/tech leadership.

---

## Table of Contents
1. [Philosophy & Principles](#philosophy--principles)
2. [High-Level Architecture](#high-level-architecture)
3. [Repositories](#repositories)
4. [Frontend Stack](#frontend-stack)
5. [Backend Stack](#backend-stack)
6. [Specialized AI Service](#specialized-ai-service)
7. [AWS Infrastructure](#aws-infrastructure)
8. [Local Dev & Tooling](#local-dev--tooling)
9. [Environments & Config](#environments--config)
10. [Data & GraphQL Contract](#data--graphql-contract)
11. [CI/CD](#cicd)
12. [Observability](#observability)
13. [Security & Compliance](#security--compliance)
14. [Cost Awareness](#cost-awareness)
15. [Runbooks (Ops)](#runbooks-ops)
16. [Roadmap](#roadmap)
17. [Appendix (Snippets & Templates)](#appendix-snippets--templates)

---

## Philosophy & Principles

- **End-to-end ownership:** Each app spans client → backend → AI with single-owner responsibility.
- **Production bias:** Prefer managed services (App Runner, RDS, S3) to reduce ops toil.
- **Clear interfaces:** Typed GraphQL schema (Relay), stable REST contracts for AI.
- **Privacy by design:** Least-privilege IAM, KMS encryption, strict PII handling.
- **Observability first:** Traces/metrics/logs everywhere; budgets & SLOs are enforced.
- **Evaluate continuously:** Offline golden sets + online A/B for AI quality.
- **Reproducibility:** IaC + Taskfile/Make targets to bootstrap and ship consistently.

---

## High-Level Architecture

```mermaid
flowchart LR
  subgraph Client
    RN[React Native (iOS via TestFlight)]
    WEB[React Web]
  end

  subgraph Edge
    CF[CloudFront CDN]
  end

  subgraph Backend
    ALB[App Runner Endpoint]
    API[Go API (GraphQL + Relay)]
    ENT[go-ent ORM]
    PG[(RDS PostgreSQL)]
    S3[(S3 Buckets)]
  end

  subgraph AI
    AIAR[App Runner Endpoint]
    AIFast[FastAPI (REST)]
    PyLibs[Python AI libs]
    ExtAPIs[[SOTA AI APIs]]
  end

  RN -->|HTTPS| CF --> ALB --> API
  WEB -->|HTTPS| CF
  API -->|GraphQL resolvers| ENT --> PG
  API -->|S3 SDK| S3
  API -->|Internal REST| AIAR --> AIFast --> ExtAPIs
  AIFast -->|pre/post-process| PyLibs
````

### End-to-End Request Sequence (Happy Path)

```mermaid
sequenceDiagram
  participant Mobile/Web
  participant GraphQL API
  participant Postgres
  participant AI Service
  participant External AI API

  Mobile/Web->>GraphQL API: GraphQL mutation/query (Relay)
  GraphQL API->>Postgres: Read/write via go-ent
  GraphQL API->>AI Service: POST /v1/generate (payload)
  AI Service->>External AI API: Invoke model (SOTA API)
  External AI API-->>AI Service: Streamed/Chunked response
  AI Service-->>GraphQL API: JSON result + metadata (latency, tokens)
  GraphQL API-->>Mobile/Web: GraphQL response (Relay store update)
```

---

## Repositories

* **`mobile/`** — React Native app (iOS via TestFlight).
* **`web/`** — React web app (Dockerized; deployed to AWS App Runner behind CloudFront).
* **`api/`** — Go service: GraphQL (Relay compliant) + go-ent + S3 integration.
* **`ai/`** — FastAPI microservice orchestrating external AI APIs + Python ML libs.
* **`infra/`** — IaC (Terraform/CDK) for App Runner services, RDS, S3, VPC, IAM.

> Each repo ships a container image; App Runner consumes images from ECR. VPC Connectors enable private RDS/S3 access.

---

## Frontend Stack

### React Native (Mobile)

* **Tech:** React Native, Expo (optional), Relay, TypeScript.
* **Distribution:** TestFlight for iOS.
* **State:** Relay store + lightweight local storage.
* **Networking:** Relay environment → GraphQL endpoint; file uploads direct to S3 pre-signed URLs.
* **Edge:** Static assets via EAS/Apple infra; runtime API via CloudFront → App Runner.

### React (Web)

* **Tech:** React + TypeScript, Relay, Vite/Next-style bundling (project-dependent).
* **Deployment:** Docker image → App Runner; fronted by CloudFront.
* **Performance:** Code-split, HTTP/2, compression, image optimization.

### GraphQL + Relay

* Relay requires:

  * Stable node IDs (`id: ID!`) and `Node` interface.
  * Connections (`edges/node/pageInfo`) for pagination.
  * Persisted queries (recommended) to reduce payload size and lock schema.
  * Typed artifacts generated at build time.

---

## Backend Stack

* **Language/Runtime:** Go.
* **ORM & Schema:** **go-ent** — code-generated schema, migrations, typed queries.
* **API:** **GraphQL** with resolvers calling ent; schema first or code first acceptable.
* **Storage:** **PostgreSQL (RDS)**; S3 for files, pre-signed URL flow.
* **Container:** Docker; **AWS App Runner** deployment with VPC access.
* **Internal Calls:** REST to AI service (App Runner internal or private URL).
* **Auth:** JWT/OIDC (Cognito/Auth0), per-request auth context & policy checks.
* **Observability:** OpenTelemetry (traces/metrics/logs) exported to CloudWatch/OTel collector.

---

## Specialized AI Service

* **Framework:** FastAPI (Python 3.11+).
* **Responsibilities:**

  * Normalize requests to SOTA AI APIs (provider-agnostic adapter layer).
  * Pre/post-processing (chunking, prompt templates, scoring, safety filters).
  * Token usage accounting and latency measurement.
  * Optional RAG hooks (embedding, retrieve, re-rank) if/when needed.
* **Deployment:** Docker → AWS App Runner (VPC; restricted SG; no public DB access).
* **Contracts:** Versioned REST endpoints (`/v1/generate`, `/v1/embed`, `/v1/moderate`).
* **Python libs:** Best-in-class tokenizers, text splitters, evaluators, etc.

---

## AWS Infrastructure

* **Compute:** AWS App Runner for `web`, `api`, and `ai` services (separate).
* **Network:** VPC with private subnets; App Runner VPC Connectors for egress to RDS/S3; security groups locked down.
* **Data:** RDS PostgreSQL (encrypted), S3 buckets (raw/uploads/public) with KMS & lifecycle rules.
* **Edge:** Route53 → CloudFront → App Runner endpoints (web/api).
* **Images:** ECR for container images.
* **Secrets:** AWS Secrets Manager (DB creds, API keys).
* **IAM:** Least-privilege roles per service; scoped S3 access; no wildcard policies.
* **Artifacts:** Terraform stacks and GitHub OIDC for deploy-time credentials.

---

## Local Dev & Tooling

* **Prereqs:** Docker, Make/Task, Go, Node, Python 3.11, Poetry/uv (optional), `aws` CLI.
* **Databases:** Local Postgres via Docker; Local S3 via `minio` (optional).
* **`Taskfile.yml`/`Makefile` Targets:**

  * `task dev` — start api + ai + web with hot-reload.
  * `task db:up`/`db:migrate` — boot Postgres & apply ent migrations.
  * `task relay` — generate Relay artifacts.
  * `task test` — run all tests.
  * `task lint` — lint (golangci-lint, eslint, ruff).
  * `task docker:build` — build images for api/ai/web.
  * `task deploy:*` — trigger GitHub Actions or local TF apply.

---

## Environments & Config

* **Envs:** `dev`, `staging`, `prod`.
* **Config loading order:** env vars → `.env` (local only) → Secrets Manager in cloud.
* **Key Vars (examples):**

  * `GRAPHQL_URL`, `AI_SERVICE_URL`
  * `POSTGRES_DSN` (local), `RDS_SECRET_ARN` (cloud)
  * `S3_BUCKET_UPLOADS`, `S3_PUBLIC_BASEURL`
  * `JWT_ISSUER`, `JWT_AUDIENCE`, `JWKS_URL`
  * Provider keys: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. (in Secrets Manager)

---

## Data & GraphQL Contract

* **Core Entities:** `User`, `Organization`, `Project`, `Document`, `Job`, `AuditEvent`.
* **GraphQL Conventions:**

  * `Node` interface & global `id`.
  * `createdAt/updatedAt`, `ownerId`, `orgId`.
  * Connections for lists; `first/after` + `pageInfo`.
  * `mutation` results return the mutated node and an `AuditEvent` id.

---

## CI/CD

* **CI:** GitHub Actions:

  * Lint & tests (unit/integration/e2e).
  * Build images → push to ECR.
  * Generate and upload SBOM; sign images (cosign).
  * Terraform plan (PR) → apply on `main` with manual approval.
* **CD:** App Runner service updates from ECR image tags; blue/green or rolling.
* **Schema/Migrations:** ent migration step gates deploy; `dbmate`/`atlas` optional.

---

## Observability

* **Traces:** OpenTelemetry SDKs in `api`/`ai`; propagate `traceparent` to AI calls.
* **Metrics:** RED/USE per service; token spend & model latency histograms.
* **Logs:** Structured JSON; request ID; user/tenant ids (pseudonymous).
* **Dashboards:** CloudWatch + (optional) Grafana.
* **SLOs:**

  * API p95 latency ≤ 800ms (no AI call) / ≤ 2500ms (with AI).
  * AI endpoint success rate ≥ 99%.
  * Error budget alerting (Pager/SMS/Email).

---

## Security & Compliance

* **AuthN/Z:** OIDC JWT; per-resolver auth; role/tenant context; rate limits.
* **Data Protection:** TLS everywhere; KMS on S3/RDS; pre-signed S3 uploads; PII redaction.
* **IAM:** No wildcard principals; separate roles for CI, runtime, read-only dashboards.
* **Secrets:** Only in Secrets Manager; never in code or Docker build args.
* **Audit:** `AuditEvent` on auth, data export, admin actions.
* **Compliance Aids:** Data retention policy; DSAR/SAR export pipeline.

---

## Cost Awareness

* **Biggest drivers:** App Runner compute hours, RDS instance size/storage/IO, egress, AI API usage.
* **Controls:**

  * Budgets + alerts; anomaly detection.
  * Nightly scale-down in low-traffic envs.
  * Token caps per user/tenant; caching; smaller input prompts; streaming first byte ASAP.
  * Lifecycle rules for S3 (move to IA/Glacier).

---

## Runbooks (Ops)

* **AI Timeouts:** Return partial/streamed results; retry with exponential backoff; open circuit after p95>threshold for N mins.
* **DB Connection Exhaustion:** Validate pool settings; bump max conns; investigate N+1; use read-only replicas (future).
* **S3 Upload Failures:** Re-issue pre-signed URL; verify clock skew; check IAM boundary and bucket policy.
* **App Runner CrashLoop:** Inspect last image tag; roll back; compare config drift; check env/secret bindings.
* **Secrets Rotation:** Update in Secrets Manager; restart services; verify health probes.

---

## Roadmap

* [ ] Persisted queries for Relay.
* [ ] AI evaluator service (quality metrics, guardrails).
* [ ] Optional vector search (pgvector/OpenSearch) for RAG projects.
* [ ] Canary deploys with request-mirroring to staging.
* [ ] Synthetic canaries on critical paths (auth, AI, upload).

---

## Appendix (Snippets & Templates)

### 1) Relay Environment (web/mobile)

```ts
// web/src/relay/environment.ts
import { Environment, Network, RecordSource, Store } from 'relay-runtime';

function fetchGraphQL(text: string | null | undefined, variables: any) {
  return fetch(process.env.GRAPHQL_URL!, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      authorization: `Bearer ${localStorage.getItem('token') ?? ''}`,
    },
    body: JSON.stringify({ query: text, variables }),
  }).then(res => res.json());
}

export const relayEnv = new Environment({
  network: Network.create(fetchGraphQL),
  store: new Store(new RecordSource()),
});
```

### 2) GraphQL Server (Go) — minimal

```go
// api/cmd/server/main.go
func main() {
  // init config, logger, otel
  // connect Postgres via ent
  // attach GraphQL handler + playground (dev)
  // healthz/readyz; metrics endpoint
}
```

### 3) ent Schema Example

```go
// api/ent/schema/document.go
type Document struct { 
  ent.Schema 
}
func (Document) Fields() []ent.Field {
  return []ent.Field{
    field.UUID("id", uuid.UUID{}).Default(uuid.New),
    field.UUID("owner_id", uuid.UUID{}),
    field.String("title"),
    field.Time("created_at").Default(time.Now),
    field.Time("updated_at").UpdateDefault(time.Now),
  }
}
```

### 4) FastAPI AI Endpoint

```py
# ai/app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import httpx, time

app = FastAPI()

class GenRequest(BaseModel):
    prompt: str
    model: str | None = None
    max_tokens: int = 512

@app.post("/v1/generate")
async def generate(req: GenRequest):
    t0 = time.time()
    async with httpx.AsyncClient(timeout=30) as client:
        # call chosen SOTA API; abstract behind adapter
        r = await client.post("https://provider/api", json={"prompt": req.prompt, "max_tokens": req.max_tokens})
    if r.status_code != 200:
        raise HTTPException(502, "upstream error")
    elapsed_ms = int((time.time()-t0)*1000)
    return {"text": r.json().get("text",""), "latency_ms": elapsed_ms}
```

### 5) S3 Pre-Signed Upload (Go)

```go
// api/internal/storage/s3.go
// func GeneratePresignedPut(ctx, key string, contentType string, ttl time.Duration) (url string, err error)
```

### 6) Dockerfiles (skeletons)

```dockerfile
# api/Dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o /out/api ./cmd/server

FROM gcr.io/distroless/base-debian12
COPY --from=build /out/api /api
USER nonroot:nonroot
ENTRYPOINT ["/api"]
```

```dockerfile
# ai/Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY pyproject.toml poetry.lock* /app/
RUN pip install --no-cache-dir -U pip && pip install --no-cache-dir fastapi uvicorn httpx
COPY ./app /app/app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

```dockerfile
# web/Dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

### 7) Terraform (App Runner + VPC Connector) — outline

```hcl
# infra/terraform/apprunner.tf
# - aws_apprunner_service for api/web/ai
# - aws_apprunner_vpc_connector for RDS/S3 private access
# - aws_iam_role for ECR + Secrets Manager
# - aws_rds_cluster / instance (Aurora Postgres or RDS Postgres)
# - aws_s3_bucket with KMS + lifecycle
```

### 8) Taskfile

```yaml
# Taskfile.yml
version: '3'
tasks:
  dev:
    cmds:
      - docker compose up --build
  relay:
    cmds:
      - npm run relay
  test:
    cmds:
      - go test ./...
      - pytest -q
```

### 9) Env Templates

```bash
# .env.example (local only)
GRAPHQL_URL=http://localhost:8080/query
AI_SERVICE_URL=http://localhost:8081
POSTGRES_DSN=postgres://user:pass@localhost:5432/app?sslmode=disable
S3_BUCKET_UPLOADS=local-uploads
JWT_ISSUER=http://localhost:8080
```

---

## Research Projects (Python-Only)

When the goal is experimentation (papers, benchmarks, prototypes):

* Use standalone Python repos with notebooks/scripts, FastAPI optional for demo UIs.
* Keep dependencies in `pyproject.toml` or `requirements.txt`; use `uv`/Poetry.
* Include `experiments/` (configs, seeds, metrics) and `results/` (artifacts, charts).
* Optionally mirror serving patterns from `ai/` to ease path to production.


