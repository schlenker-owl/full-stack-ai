# AWS Infrastructure — End-to-End Architecture for Universal AI Foundry

> **Goal:** Document how AWS is used to serve our applications end-to-end — from **client → backend → AI service** — with clear guidance on **VPCs, IAM, NAT, ALB, App Runner, RDS, ECS/EC2, and S3**.  
> **Bias:** Managed, low-ops, cost-aware. Public-facing entrypoints are minimized; internal services run private by default.

---

## 0) TL;DR (Service Matrix)

| Layer | Service | Purpose | Exposure | Notes |
|---|---|---|---|---|
| Edge/CDN | **CloudFront** | TLS, caching, WAF, geo | Public | Origins: `web` (static), `api` (GraphQL). Custom domains via Route 53 + ACM. |
| Web | **App Runner** (`web`) | Serve React build/static | Public (via CloudFront) | No DB access. Cache-first; long TTLs for assets. |
| API | **App Runner** (`api`) | Go GraphQL (Relay) | Public (via CloudFront) | VPC Connector → RDS/S3 endpoints/**AI private**. Verifies Clerk JWTs. |
| AI | **App Runner** (`ai`) | FastAPI AI microservice | **Private (VPC Ingress)** | Reached only from `api` via VPC DNS/Security Groups. Calls external AI APIs via NAT. |
| Database | **RDS Postgres** | App data | Private | Multi-AZ, encrypted, parameter group; optional RDS Proxy. |
| Object Storage | **S3** | Uploads & public assets | Private/Public | Private bucket for uploads (pre-signed). Public origin bucket (via CloudFront OAC). |
| Containers | **ECR** | Image registry | Private | Enhanced scanning (Inspector). |
| Optional GPU path | **ECS on EC2 (GPU)** or **EKS** | vLLM / heavy inference | Private + **ALB** | Only when we outgrow App Runner for AI. ALB becomes CloudFront origin. |

---

## 1) DNS, TLS & Edge

- **Route 53** hosts public zones; `A/AAAA` records point CloudFront distributions.  
- **ACM (us-east-1)** issues certificates for CloudFront.  
- **CloudFront** has two origins:  
  - **`web`** (static artifacts from S3 or App Runner static)  
  - **`api`** (App Runner `api` HTTPS origin)  
- Optional **AWS WAF** rules: rate limit, GraphQL introspection restrictions (stage/prod), IP reputation lists.  
- Behaviors route `/graphql` and `/api/*` to the API origin; everything else to static.

---

## 2) VPC Topology

- **1 VPC / 2 AZs**, three subnet tiers:
  - **Public subnets:** IGW + NAT Gateways (one per AZ in prod; single in dev).  
  - **Private app subnets:** For App Runner **VPC Connectors** and (optionally) ECS tasks.  
  - **Private data subnets:** RDS and VPC **Interface/Gateway Endpoints**.
- **Route tables:**  
  - Private app → NAT for internet egress (AI providers, Clerk JWKS).  
  - Private data → **S3 Gateway Endpoint** (no NAT charges for S3) and **Interface Endpoints** for Secrets Manager/STS/ECR API.  
- **VPC Endpoints:**  
  - **Gateway:** S3  
  - **Interface:** Secrets Manager, ECR API, ECR DKR, CloudWatch Logs, STS (as needed)

> **Why this split?** App subnets can talk out (via NAT) *without* exposing data tier; data subnets never require public routes.

---

## 3) App Runner Services

### 3.1 `web` (public)
- Serves built React assets.  
- **No VPC Connector** required.  
- Origin for CloudFront static; optionally backed by S3+OAC if we prefer S3 as origin.

### 3.2 `api` (public + VPC egress)
- Public HTTPS endpoint (CloudFront origin).  
- **VPC Connector** attaches to *private app subnets* to reach:
  - **RDS** (via SG allowlist)  
  - **S3** (via Gateway Endpoint)  
  - **AI (private App Runner)** via **VPC Ingress** and private DNS  
  - **Secrets Manager/Logs/ECR** via Interface Endpoints  
- Outbound internet (external AI APIs, Clerk JWKS) via **NAT Gateway**.

### 3.3 `ai` (private)
- **Private Ingress** (VPC-only). No public endpoint.  
- Runs FastAPI; authenticates **only** with an internal API key (header).  
- Accesses external AI APIs via **NAT**; reads secrets from Secrets Manager; optional S3 access for media.

> **Security Groups:**  
> - `sg-api` → egress to `sg-ai`, `sg-rds`, endpoints; inbound only from CloudFront (public) and health checks.  
> - `sg-ai` → inbound only from `sg-api`.  
> - `sg-rds` → inbound from `sg-api` (and optionally `sg-ai` if needed).

---

## 4) Database — RDS Postgres

- **Multi-AZ**, encryption at rest (KMS), storage autoscaling.  
- **Parameter group** tuned for connection limits and statement timeout.  
- Optional **RDS Proxy** if connection churn occurs (helps with spiky App Runner tasks).  
- **Backups:** Daily automated snapshots; PITR enabled; weekly snapshot retention.  
- **Connectivity:** Only via `sg-rds` from `sg-api` (and `sg-ai` if direct reads required). No public access.

---

## 5) S3 Buckets & Access Patterns

- **`uploads` bucket (private)**  
  - Client uploads via **pre-signed PUT** returned by `api`.  
  - Server fetches via SDK; cross-account prevented by bucket policy.  
  - Lifecycle: transition older objects to IA/Glacier.

- **`public` bucket (origin) or `web` App Runner**  
  - If using S3 for static site, front with **CloudFront OAC** (not public).  
  - If using App Runner for static, CloudFront caches `/assets/*`.

- **VPC Gateway Endpoint** to S3 cuts NAT traffic/egress costs for server access.

---

## 6) IAM & Secrets

- **Per-service roles** with least privilege:
  - **`role-web`**: read-only to logs/metrics.  
  - **`role-api`**: `rds-db:connect` via SG, S3 object RW (restricted prefix), Secrets Manager read, Logs write.  
  - **`role-ai`**: Secrets read, S3 read/write (strict prefixes), Logs write, outbound via NAT.  
- **GitHub OIDC**: CI assumes a deploy role with limited permissions (ECR push, App Runner update, read/write Terraform state).  
- **Secrets Manager** for DB creds, API keys (Clerk secret, AI providers, internal API key).  
- **KMS** customer-managed keys for RDS, S3 (bucket key), and Secrets (auto-rotation where supported).

---

## 7) NAT Gateways (Cost & Reliability)

- **Prod:** One NAT per AZ for HA (2x NAT).  
- **Dev/Staging:** Single NAT to reduce cost; accept single-AZ egress risk.  
- Prefer **Gateway Endpoints** (S3) and **Interface Endpoints** to avoid NAT egress where possible.  
- Monitor `BytesOutToDestination` to track egress spend.

---

## 8) Load Balancing & ALB (When We Need It)

Although App Runner has managed ingress, we use **ALB** when:
- Migrating the **AI** service to **ECS/EKS** (GPU workloads, vLLM).  
- Requiring **path-based routing** across many internal services in VPC.  
- Needing **advanced stickiness** or custom health behavior not available in App Runner.

**Pattern:** CloudFront → **ALB** (public) → **ECS/EKS services** (private) → RDS/S3 via VPC endpoints.  
ALB security group only allows CloudFront; targets are private.

---

## 9) ECS / EC2 (GPU) — Optional Heavy Inference Path

Use when the AI service needs **GPU** or custom runtimes:
- **ECS on EC2 (GPU instances)** for vLLM/TGI; AMI with NVIDIA drivers + CUDA.  
- **Task roles** mirror `role-ai` but add ECR pulls and CloudWatch logs.  
- **ALB target group** (HTTP/HTTPS) per service; health checks to `/healthz`.  
- Optional **NLB** for gRPC or high-throughput needs.  
- If scaling fast, consider **Spot** with capacity rebalancing for stateless inference.

---

## 10) Observability (Infra)

- **CloudWatch Logs**: one log group per service; structured JSON.  
- **Metrics & Traces**: OTel Collector (agent or sidecar) shipping to CloudWatch/Grafana/OTLP.  
- **Dashboards**: p50/95/99 latency; 4xx/5xx; NAT bytes; DB CPU/IO; Postgres connections; App Runner CPU/mem concurrency.  
- **Alarms**: elevated 5xx on API, failing health checks, DB CPU > 80%, free storage low, NAT egress spikes, AI upstream error rate.

---

## 11) Networking: How Traffic Flows

### 11.1 Public request path (client → backend)
```mermaid
sequenceDiagram
  participant User
  participant CF as CloudFront (TLS/WAF)
  participant API as App Runner (api)
  participant RDS as RDS (Postgres)
  participant AI as App Runner (ai, Private)
  participant Ext as External AI APIs

  User->>CF: HTTPS /graphql
  CF->>API: HTTPS (origin)
  API->>RDS: VPC (SG-to-SG)
  API->>AI: VPC private DNS (SG-to-SG)
  AI->>Ext: egress via NAT (timeouts/retries)
  API-->>CF: 200 JSON
  CF-->>User: Cached (as allowed)
````

### 11.2 Network layout (two AZs)

```mermaid
flowchart LR
  IGW["Internet Gateway"]
  CF["CloudFront (global)"]
  NATa["NAT-A"]:::nat
  NATb["NAT-B"]:::nat

  subgraph VPC
    subgraph Public-AZ-A
      PubA["Public Subnet A"]
    end
    subgraph Public-AZ-B
      PubB["Public Subnet B"]
    end

    subgraph PrivateApp-AZ-A
      AppA["Private App Subnet A"]
    end
    subgraph PrivateApp-AZ-B
      AppB["Private App Subnet B"]
    end

    subgraph PrivateData-AZ-A
      DataA["Private Data Subnet A"]
    end
    subgraph PrivateData-AZ-B
      DataB["Private Data Subnet B"]
    end

    RDS["RDS (Multi-AZ)"]:::db
    S3EP["S3 Gateway Endpoint"]:::ep
    IFCEPs["Interface Endpoints (SM/ECR/Logs)"]:::ep
  end

  CF -->|HTTPS| IGW
  PubA --> NATa
  PubB --> NATb
  AppA --> NATa
  AppB --> NATb
  AppA --> IFCEPs & S3EP
  AppB --> IFCEPs & S3EP
  DataA --> S3EP
  DataB --> S3EP
  AppA --> RDS
  AppB --> RDS

  classDef nat fill:#fde68a,stroke:#b45309,color:#111;
  classDef db fill:#bfdbfe,stroke:#1d4ed8,color:#111;
  classDef ep fill:#d1fae5,stroke:#065f46,color:#111;
```

---

## 12) CI/CD & Images

* **Build** on GitHub Actions → push to **ECR** (scanned).
* **Deploy**:

  * App Runner services auto-update from ECR image tag (environment-scoped).
  * IaC (Terraform/CDK) applies VPC/Endpoints/SGs/NAT/RDS/CloudFront/distributions.
* **Permissions**: OIDC role limited to ECR push, App Runner update, state bucket access.
* **Rollbacks**: App Runner supports previous image rollback; DB migrations gated by plan/apply review.

---

## 13) Security Posture

* **Network:** Only CloudFront can reach public services; AI/private services are VPC-only.
* **Auth:** Clerk JWTs verified at API; AI service requires internal API key (no end-user tokens).
* **Data:** KMS everywhere (RDS, S3, Secrets).
* **IAM:** No wildcards; resource-level conditions; minimal S3 prefixes per service.
* **WAF:** Block abusive traffic; GraphQL depth/complexity enforced at API.
* **Backups & DR:** RDS PITR; S3 versioning + lifecycle; infra codified (rebuildable).

---

## 14) Cost Controls

* **NAT optimization:** 1x NAT in dev; 2x in prod; prefer VPC endpoints to avoid NAT egress.
* **CloudFront caching:** Aggressive for static; cache GraphQL OPTIONS and schema if safe.
* **RDS sizing:** Right-size instances; enable storage autoscaling; consider RDS Proxy.
* **App Runner**: Tune concurrency and CPU/mem; scale to zero in dev if acceptable.
* **Budgets & Alerts:** Account budgets; anomaly detection on NAT egress and DataTransfer-Out.

---

## 15) Acceptance Checklist

* [ ] CloudFront serves `web` and proxies `api` under custom domain + TLS.
* [ ] `api` reaches RDS/S3/AI via **VPC Connector**; `ai` is not publicly reachable.
* [ ] RDS is private, encrypted, Multi-AZ, with automated backups enabled.
* [ ] S3 uploads succeed via **pre-signed PUT**; bucket policies restrict direct public access (OAC for public assets).
* [ ] NAT egress works for external AI APIs; S3 traffic uses Gateway Endpoint (no NAT).
* [ ] Alarms in place: API 5xx, AI upstream failures, NAT egress spikes, RDS CPU/conn, disk space.
* [ ] IAM roles are least-privilege; Secrets Manager holds all sensitive keys; KMS key policies reviewed.

---

## 16) When to use ECS/EC2 instead of App Runner

Choose **ECS on EC2 (GPU)** (or EKS) for the AI service if:

* You need **GPUs**, custom CUDA/driver control, or very low-level inference knobs.
* You require **model parallelism** (e.g., vLLM tensor-parallel across GPUs).
* You need **sticky sessions** or **gRPC** with NLB.

Migration path: stand up ECS service (private), attach **ALB** (internal), switch CloudFront API origin to the **ALB** or keep API on App Runner and call ECS AI privately.

---

## 17) Runbooks (infra)

* **API can’t reach DB:** Check SG rules (api→rds), route tables, and DNS resolution inside VPC Connector.
* **High NAT bill:** Add/check S3 Gateway Endpoint; move secrets/logs to Interface Endpoints; consolidate NATs (dev).
* **AI private DNS not resolving:** Ensure `ai` service is **private ingress** and `api` VPC Connector subnets share the same VPC; check Route 53 private hosted zone if used.
* **CloudFront 502s:** Validate origin TLS certs and headers; check App Runner health and throttling.
* **RDS connection exhaustion:** Add RDS Proxy or pool at app; tune `max_connections`; audit N+1 queries.

---

## 18) Future Enhancements

* **WAF managed rules** tuned for GraphQL.
* **Private App Runner for API** behind **VPC Lattice** (full private ingress), then expose via ALB if needed.
* **Cross-region DR**: daily snapshot replicate + IaC to spin infra in secondary region.
* **Centralized OTEL** collector with tail-based sampling to cut tracing cost.

