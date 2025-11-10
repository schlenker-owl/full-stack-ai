# Observability — AI App Foundry

> **Scope:** A complete telemetry spec and runbooks for our stack: **Client (RN/React) → API (Go/GraphQL/ent) → AI service (FastAPI) → AWS (CloudFront, App Runner, RDS, S3, NAT)**.  
> **Signals:** **Metrics, Logs, Traces, Events (Audit), Synthetics**.  
> **Destinations (default profile):** CloudWatch Logs & Metrics, AWS X-Ray for traces. *(Alt profile: AMP/Tempo/Grafana — see §12)*

---

## 0) Objectives

- **See every request** end-to-end with **p95 latency + errors** by operation and by tenant.
- **Find cost drivers** (tokens, egress, DB) within minutes.
- **Page humans only** on symptoms that matter to users (SLO-based).
- **Keep it cheap**: low-cardinality labels, tail sampling for traces, short log retention in non-prod.

---

## 1) Signal Overview

| Signal | Tooling | Purpose | Retention (prod) |
|---|---|---|---|
| **Metrics** | OTel → CloudWatch Metrics | RED/USE KPIs, SLOs, cost meters | 15 months (CW standard) |
| **Logs** | stdout → CloudWatch Logs | Forensics, structured context | 14–30 days app; 90 days audit |
| **Traces** | OTel → AWS X-Ray | Causal latency & error analysis | 30 days (X-Ray) |
| **Events** | App audit table + CW Logs | Security/audit trail | 1 year |
| **Synthetics** | Canaries (Lambda/Actions) | Catch user-visible breaks | 90 days |

---

## 2) Canonical resource names

```

service: web | api | ai
env: dev | staging | prod
region: us-east-1 (etc)
version: git SHA or semver

````

---

## 3) Required labels / attributes

> **Keep cardinality low.** Hash or bucket where needed.

| Name | Applies | Notes |
|---|---|---|
| `service`, `env`, `version`, `region` | all signals | required |
| `request_id`, `trace_id` | logs/traces | UUIDs; correlate across services |
| `tenant_id` | all | pseudonymous (hash/org UUID), **not PII** |
| `user_id` | logs/traces | pseudonymous; omit for anonymous |
| `graphql.operation`, `graphql.type` | API | e.g., `Query.me`, `Mutation.createDocument` |
| `http.method`, `http.route`, `http.status_code` | API/AI | route = templated path |
| `provider`, `model` | AI | e.g., `openai`, `gpt-4o-mini` |
| `tokens_in`, `tokens_out`, `cost_usd` | AI | numbers; optional in logs; counters in metrics |
| `cache_hit` | API/AI | boolean |
| `db.pool_in_use`, `db.pool_wait_ms` | API | gauges for DB pool pressure |

---

## 4) Metric inventory (names, types, units)

### 4.1 API (Go/GraphQL)

| Metric | Type | Unit | Labels |
|---|---|---|---|
| `api_request_total` | counter | requests | `route`, `method`, `status_code` |
| `api_request_duration_ms` | histogram | ms | `route`, `method`, `status_code` |
| `graphql_operation_total` | counter | ops | `operation`, `type`, `status`(ok/error) |
| `graphql_operation_duration_ms` | histogram | ms | `operation`, `type` |
| `rate_limit_hits_total` | counter | hits | `key_type`(user/org/ip) |
| `db_pool_in_use` | gauge | conns | — |
| `db_pool_wait_ms` | histogram | ms | — |
| `s3_presign_total` | counter | ops | `kind`(put/get) |

**Derivations:** API **availability** = 1 − (5xx / all). “With AI” latency = GraphQL ops that call AI (tag via span attribute).

### 4.2 AI service (FastAPI)

| Metric | Type | Unit | Labels |
|---|---|---|---|
| `ai_request_total` | counter | requests | `endpoint`, `status_code` |
| `ai_request_duration_ms` | histogram | ms | `endpoint` |
| `ai_upstream_calls_total` | counter | calls | `provider`, `model`, `status` |
| `ai_upstream_latency_ms` | histogram | ms | `provider`, `model` |
| `ai_tokens_in_total` | counter | tokens | `provider`, `model` |
| `ai_tokens_out_total` | counter | tokens | `provider`, `model` |
| `ai_cost_usd_total` | counter | USD | `provider`, `model` |
| `ai_circuit_open` | gauge | 0/1 | `provider` |

### 4.3 Edge / Infra

| Metric | Source | Why |
|---|---|---|
| CloudFront `5xxErrorRate`, `Requests`, `BytesDownloaded` | CloudFront | Edge health & egress |
| App Runner `CPUUtilization`, `MemoryUtilization`, `Concurrency` | App Runner | Right-sizing |
| RDS `CPUUtilization`, `FreeableMemory`, `DatabaseConnections`, `ReadIOPS/WriteIOPS`, `Deadlocks` | RDS | DB health |
| NAT `BytesOutToDestination`, `ActiveConnections` | NAT | Egress & cost |
| S3 `4xxErrors`, `5xxErrors`, `FirstByteLatency` | S3 (CloudWatch/S3 Storage Lens) | Anomalies on object ops |

---

## 5) Logs (contract & examples)

**Format:** JSON, one line per event. Required envelope:

```jsonc
{
  "ts": "2025-11-04T16:00:00.123Z",
  "level": "info|warn|error",
  "service": "api",
  "env": "prod",
  "version": "b4f1c42",
  "region": "us-east-1",
  "request_id": "a3c7...",
  "trace_id": "94f2...",
  "tenant_id": "org_7f9c...",
  "user_id": "usr_1a2b...", // pseudonymous
  "msg": "graphql operation",
  "graphql": { "operation": "Query.me", "type": "Query" },
  "http": { "method": "POST", "route": "/graphql", "status": 200, "latency_ms": 184 }
}
````

**Redaction rules**

* Never log tokens, secrets, emails, phone numbers, raw prompts/content.
* Log **decisions** (e.g., “moderation_blocked=true”) and **metrics** (latency/tokens/cost), not payloads.
* Sample error payloads only in **dev/staging** with explicit flags.

---

## 6) Traces

* **Propagation:** W3C `traceparent` and `tracestate` across client → API → AI; include `tenant_id`, `provider`, `model` as span attributes.
* **Span names:** `HTTP POST /graphql`, `GraphQL Query.me`, `AI POST /v1/generate`, `SQL SELECT document_by_id`.
* **Sampling (prod):** Tail-based: **10%** default, **100%** for errors, and **100%** for slow (> p95) or AI upstream errors.
* **Destination:** AWS X-Ray (default profile).

---

## 7) SLOs & error budgets

> Measure over rolling **30 days**, with weekly guardrails.

### 7.1 API (GraphQL)

* **Availability:** 99.9% (3 nines). SLI = `1 − 5xx_rate` at the edge origin.
* **Latency:** p95 ≤ **800 ms** (no AI), p95 ≤ **2.5 s** (with AI). Determined via span attribute `uses_ai=true`.
* **Error rate:** < **1%** of requests over 10 minutes.

### 7.2 AI service

* **Upstream success:** ≥ **99%** per provider over 1 hour.
* **Upstream latency:** p95 ≤ **2.0 s** per provider over 1 hour.
* **Cost guardrail:** 95th percentile **cost_usd/request** ≤ policy threshold (per env).

**When an SLO is breached**

* Declare severity, start budget burn review.
* Enact predefined **degradations**: switch to cheaper/faster model, reduce context, enable caching, rate-limit heavy ops.

---

## 8) Alert policy (page vs. notify)

> **Page** = wake someone (on-call). **Notify** = async (Slack/email).

| Condition (prod)                     | Threshold / Window            | Action                             |
| ------------------------------------ | ----------------------------- | ---------------------------------- |
| API error rate (5xx)                 | > **1%** 10m                  | **Page**                           |
| API p95 (with AI)                    | > **2.5s** 15m                | **Page**                           |
| AI upstream error rate (by provider) | > **5%** 10m                  | **Page**                           |
| NAT egress surge                     | > **2×** 7-day median for 60m | **Notify**                         |
| RDS connections                      | > **80%** max for 15m         | **Notify** (page if sustained 30m) |
| App Runner CPU                       | > **85%** 10m with p95 > SLO  | **Notify**                         |
| CloudFront 5xx                       | > **0.5%** 10m                | **Page**                           |
| Token spend per tenant               | > policy limit in 1h          | **Notify** owner                   |

**Escalation:** page on-call → platform secondary → engineering manager at 30m.

---

## 9) Dashboards (one per service)

### 9.1 API (GraphQL)

* Requests & error rate (stacked by 4xx/5xx)
* p50/p95/p99 latency (overall; with vs. without AI)
* Top 10 operations by time & errors
* Rate limit hits
* DB pool in-use & wait ms
* S3 presign successes/failures

### 9.2 AI

* Request rate & 4xx/5xx
* Upstream error rate by **provider**/**model**
* Upstream latency p50/p95/p99
* Tokens in/out per minute
* `cost_usd_total` and per-request percentile
* Circuit breaker state

### 9.3 Infra

* CloudFront requests & 5xx, egress bytes
* App Runner CPU/mem/concurrency
* NAT egress and active connections
* RDS CPU/conns/IOPS; storage growth
* S3 4xx/5xx; first-byte latency

*(Add a small **Business** dashboard with daily active users, AI requests per tenant, top models by cost.)*

---

## 10) Synthetics (canaries)

* **Edge health:** `GET /healthz` (API), `GET /readyz` (AI) from 2+ regions.
* **GraphQL E2E:** a minimal query that triggers an **AI call** to a harmless, quota-safe model; assert JSON shape and `latency_ms < 2500`.
* **S3 flow:** obtain pre-signed PUT, upload small object, finalize metadata.
* **Auth:** JWKS fetch & a signed JWT verification exercise (no real PII).
* **Budget:** run every 5 minutes; alert if 2 consecutive failures.

---

## 11) Implementation notes

### 11.1 App Runner & OTel

* Both **api** (Go) and **ai** (FastAPI) use OTel SDKs.
* **Traces:** OTLP → OTel Collector → X-Ray.
* **Metrics:** OTel SDK → Collector → CloudWatch (EMF).
* **Logs:** `stdout` → CloudWatch Logs (via App Runner).
* **Collector placement:** run one **“observability” collector** as an internal service (ECS Fargate or App Runner private). Allow traffic from services via SG/VPC Connector.

### 11.2 Sampling & cost

* 10% tail sampling saves ~90% trace cost while keeping error/slow traces.
* Keep label sets tight: avoid high-cardinality labels like raw `user_id` (use hashed pseudonyms) or free-text routes.

---

## 12) Alt profile (OSS/Grafana)

* **Metrics:** OTel → **Amazon Managed Service for Prometheus (AMP)**
* **Traces:** OTel → **Tempo** (via Grafana Cloud or self-hosted)
* **Logs:** **CloudWatch Logs** (ship to Loki if desired)
* **Dashboards:** **Managed Grafana** with service folders and templated variables
* Keep the same metric names/labels as §4.

---

## 13) On-call quickstart (first 10 queries)

1. **“What spiked?”** — API 5xx by route & operation (10m).
2. **“Is it AI?”** — AI upstream errors by provider/model (10m).
3. **“Why slow?”** — API p95 with vs. without AI (15m).
4. **“DB hot?”** — RDS CPU & connections; API DB wait ms.
5. **“Egress costly?”** — NAT BytesOut (1h vs. 7-day median).
6. **“Edge failing?”** — CloudFront 5xx & origin status.
7. **“Rate-limited?”** — Rate limit hits by key type.
8. **“Which tenants?”** — AI requests & cost per tenant (top 10).
9. **“Which operations?”** — Slowest GraphQL ops today.
10. **“Regression?”** — Compare current vs. last week (same window).

---

## 14) Acceptance checklist

* [ ] Logs include required fields; no PII or secrets are emitted.
* [ ] API & AI export traces to X-Ray; spans show provider/model where relevant.
* [ ] Metrics exist for all tables in §4 with expected labels.
* [ ] Dashboards created for API, AI, and Infra; E2E canary green.
* [ ] SLOs codified; alerts wired per §8; on-call rota linked.
* [ ] Sampling, retention, and cost budgets documented and reviewed quarterly.

---

## 15) Ownership

* **Signals & dashboards:** SRE/Platform
* **API metrics & spans:** Backend team
* **AI metrics & spans:** AI team
* **Edge/Infra metrics:** Platform
* **Business KPIs:** Product/Analytics

