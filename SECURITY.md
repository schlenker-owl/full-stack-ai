# Security — AI App Foundry

> **Scope:** Security policy, controls, and runbooks for our stack: **Clerk (auth)** → **Go GraphQL API (Relay, ent, Postgres, S3)** → **AI service (FastAPI)** on **AWS (CloudFront, App Runner, RDS, S3, ECR, VPC, NAT, ALB/ECS optional)**.  
> **Audience:** Engineers, SRE/on-call, and reviewers.  
> **Goal:** Make safe the default. Convert guidance into enforceable checks and incident-ready playbooks.

---

## 0) Trust zones & threat model

```mermaid
flowchart LR
  Internet["Clients / Internet"]:::ext
  CF["CloudFront + TLS + (WAF)"]:::edge
  API["App Runner: api (public)"]:::app
  AI["App Runner: ai (private)"]:::app
  RDS["RDS Postgres (private)"]:::data
  S3["S3 Buckets (uploads/public)"]:::data
  Clerk["Clerk (IdP)"]:::ext
  ExtAI["External AI APIs"]:::ext

  Internet --> CF --> API
  API --> AI
  API --> RDS
  API --> S3
  AI --> ExtAI
  API -. verify JWT .-> Clerk

  classDef ext fill:#eee,stroke:#666,color:#111;
  classDef edge fill:#e0f2fe,stroke:#0284c7,color:#111;
  classDef app fill:#ede9fe,stroke:#6d28d9,color:#111;
  classDef data fill:#dcfce7,stroke:#16a34a,color:#111;
````

**Top risks & mitigations (summary)**

| Risk                                   | Control (primary)                                                                                                  | Notes                                           |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------- |
| Token theft / forged JWT               | Verify Clerk JWT via JWKS (`iss/aud/exp/nbf`) on every call; short-TTL app token if using exchange mode            | Cache JWKS briefly; refresh on `kid` change     |
| Multi-tenant data leak                 | Resolver-level authz by `org_id` + role; default queries scoped to tenant                                          | Central helper for org scoping; deny-by-default |
| GraphQL abuse (DoS, deep queries)      | Depth/complexity limits, input size limits, rate limiting per user/org/IP, optional persisted queries              | Disable/limit introspection in prod             |
| Prompt injection / unsafe model output | Input/output filters; allow-list tool use; strip URLs/PII where required; model/content moderation for risky flows | Log *decisions*, never raw prompts by default   |
| S3 misuse (oversized/unsafe uploads)   | Pre-signed PUT with strict content-type, size caps, key scoping, short TTL; server-side validation on “finalize”   | Consider AV scanning or media probing           |
| Secrets leakage                        | Secrets Manager + IAM role-based access; no secrets in code/build args; redaction in logs                          | Rotate on schedule; break-glass is audited      |
| Supply chain compromise                | SBOM + image signing (cosign); vuln scan gates; pinned base images; CI OIDC with least-priv                        | Block deploy on critical vulns (policy)         |
| Egress cost/data exfil                 | VPC Endpoints for S3/SM/Logs; NAT monitoring/alerts; deny unknown egress from private subnets                      | AI calls only from `ai` service                 |

---

## 1) Data classification & handling

**Classes:** Public · Internal · Confidential (default app data) · **Restricted** (PII, auth tokens, provider keys).
**Storage/transit:** TLS 1.2+ end-to-end; KMS encryption at rest (RDS, S3, Secrets).
**Logs:** No PII, no tokens, no raw prompts. Use pseudonymous IDs (`user_id`, `org_id`); mask emails.
**Retention:**

* App logs: 14–30d (env-dependent).
* Audit events (auth, admin, export): ≥ 1 year.
* User-initiated deletes honored within 30 days (hard-delete if required).
  **Exports/DSAR:** Implement export endpoints; emit an **AuditEvent**.

---

## 2) Identity & access

### 2.1 Authentication (Clerk)

* Clients authenticate with Clerk; GraphQL receives **`Authorization: Bearer <Clerk JWT>`**.
* API verifies JWT each request (`iss`, `aud`, `exp`, `nbf`, signature).
* Optional **exchange mode**: API mints short-TTL **app JWT** with exact claims/TTL/audience; client then uses app token.

### 2.2 Authorization (API)

* Resolver context includes: `user_id`, `org_id` (tenant), `roles[]`, `rate_limit_key`.
* **Deny by default**: every resolver checks policy helper (role + tenant).
* **Rate limits**: token bucket per user/org/IP; stricter on mutations.
* **Admin**: only via server-side policy; never trust client-sent roles.

### 2.3 Service-to-service

* **AI service**: private network only; requires **internal API key** (or short-lived HMAC). **Never** forwards end-user tokens.
* Context headers allowed: `X-User-ID`, `X-Org-ID` (opaque IDs only).

---

## 3) AWS controls

### 3.1 Network

* **VPC** with public (IGW/NAT), private-app (App Runner VPC Connectors), private-data (RDS, endpoints).
* **Endpoints**: S3 (Gateway), Secrets Manager/ECR/Logs (Interface).
* **Security Groups**:

  * `api` → inbound only via CloudFront; egress to RDS/S3/ai/endpoints.
  * `ai` → inbound only from `api`.
  * `rds` → inbound from `api` (and `ai` if needed).
* **CloudFront**: TLS + optional **WAF**; ALB used only for ECS/EKS scenarios.

### 3.2 RDS Postgres

* Private, Multi-AZ, KMS encryption, TLS required.
* Parameter group: sane `statement_timeout`; limit `max_connections`.
* Optional **RDS Proxy** for connection churn.

### 3.3 S3

* **Uploads bucket**: private, Block Public Access ON, bucket KMS key, lifecycle to IA/Glacier, object ownership enforced.
* **Public assets**: S3 + CloudFront **OAC** (no public ACL).
* Pre-signed PUT/GET only via API; keys scoped to tenant prefix.

### 3.4 IAM & Secrets

* Per-service roles (least-priv). No wildcards. Prefix-scoped S3 permissions.
* **Secrets Manager** for DB creds, Clerk secret, provider keys, internal API key; rotation schedule.
* **KMS**: CMKs for RDS/S3/secrets; key policies reviewed; rotation enabled when supported.

### 3.5 Container supply chain

* **ECR**: enhanced scanning (Inspector).
* **Images**: SBOM generated; **cosign** signature/verification; pinned base images.
* **CI OIDC**: deploy role limited to ECR push, App Runner update, TF state.

---

## 4) Application hardening

### 4.1 GraphQL (Go)

* Disable or restrict **introspection** in prod; allow in dev/staging.
* Enforce **depth** and **complexity** limits; cap input size.
* Use **persisted queries** when feasible.
* Consistent error mapping; avoid leaking stack traces.

### 4.2 HTTP & CORS

* Only expected methods/paths; `Content-Type: application/json`.
* CORS allow-list: web origins only; credentials off (use bearer tokens).
* Security headers: HSTS at CloudFront, `X-Content-Type-Options: nosniff`.

### 4.3 File uploads (S3 pre-signed)

* Server issues pre-signed PUT with: content-type allow-list, **max size**, short TTL, and **tenant-scoped key**.
* Server finalizes metadata **after** validating the uploaded object (type/size).
* Optionally scan or probe media; strip metadata for public renderings.

### 4.4 AI calls

* Input/output filters for PII; optional moderation.
* Timeouts, retries with jitter, and a **circuit breaker** on provider errors.
* Log **latency and outcome**, not content.

---

## 5) Logging, audit, and privacy

* **Structured logs** (JSON) with `request_id`, `trace_id`, `user_id`, `org_id`, `operation`, `status`, `latency_ms`.
* **No** raw PII, secrets, tokens, prompts, or provider payloads.
* **Audit events** for auth, admin actions, exports, data deletes.
* CloudFront access logs enabled (to S3) with limited retention.

---

## 6) Vulnerability & dependency management

* **Automated scanning**: containers (ECR/Inspector) & deps (Go modules, npm, Python).
* **Blocking policy**: fail build on Criticals, warn on High with SLA.
* **SLA**: Critical 24–72h, High 7d, Medium 30d, Low 90d.
* Renovate/Dependabot for pinning and updates; manual review for transitive risk.

---

## 7) Key & secret rotation

* **Clerk Secret**, **app-JWT signing key**, **provider API keys**, **internal API key**: rotate on schedule, on-demand for incidents.
* RDS creds from Secrets Manager; prefer IAM auth where viable.
* Rotation playbook updates deployments via App Runner configuration / environment referencing secrets by ARN.

---

## 8) Incident response (IR)

**Severities**

| Sev   | User impact                               | Examples                                                      | Target response                           |
| ----- | ----------------------------------------- | ------------------------------------------------------------- | ----------------------------------------- |
| SEV-1 | Widespread outage or confirmed data exfil | Compromised key with active abuse, RDS exposure               | page immediately; IC in 15m; public comms |
| SEV-2 | Degraded or contained security issue      | Token verification outage, privilege bypass blocked by policy | page; IC in 30m                           |
| SEV-3 | Low impact or near-miss                   | WAF block spike, scan indicates vulnerable base image         | ticket within business day                |

**First hour checklist (condensed)**

1. **Stabilize**: block/limit traffic (WAF/rate limits), rotate affected secrets/keys.
2. **Contain**: disable affected features, revoke tokens/sessions if needed.
3. **Preserve evidence**: snapshot logs, CloudFront access logs, relevant buckets/DB snapshots.
4. **Comms**: internal update; decide on external notice if user data impacted.
5. **Eradicate/Recover**: patch, redeploy signed images, restore from backups if required.
6. **Postmortem**: within 72h—timeline, root cause, user impact, action items.

---

## 9) Compliance mapping (mini)

| Control              | Where addressed                                           |
| -------------------- | --------------------------------------------------------- |
| Access control       | IAM least-priv roles; Clerk auth; resolver policies       |
| Change management    | CI/CD with reviews; signed images; TF plans               |
| Logging & monitoring | Structured logs, OTel traces/metrics; alerting            |
| Backups & DR         | RDS PITR; S3 versioning/lifecycle; IaC rebuild            |
| Vendor management    | Provider keys in Secrets Manager; limited egress; SLAs    |
| Security testing     | Dependency/container scans; alert on Criticals; IR drills |

---

## 10) Security checklists

### 10.1 Pre-prod readiness

* [ ] All S3 buckets **Block Public Access**; public assets via CloudFront **OAC** only.
* [ ] RDS private, encrypted, TLS required; automated backups enabled.
* [ ] App Runner roles least-priv; no wildcards; secrets only in **Secrets Manager**.
* [ ] GraphQL depth/complexity limits; introspection off (prod).
* [ ] API enforces auth on every request; **Clerk JWT verified**; resolver authz in place.
* [ ] File uploads use pre-signed PUT with size/type limits and tenant-scoped keys.
* [ ] Images are **signed** (cosign); SBOM attached; Critical vulns gate deploy.

### 10.2 Release gate

* [ ] Change reviewed; TF plan approved; provenance artifact present.
* [ ] Alarms configured (API 5xx, AI upstream errors, NAT egress, RDS CPU/conn).
* [ ] Secrets recently rotated or validated; keys not embedded in images.
* [ ] WAF rules active on CloudFront (rate limit at minimum).

### 10.3 Runtime hygiene

* [ ] Quarterly IAM access review; remove unused roles/keys.
* [ ] Key rotation cadence met; verify JWKS caching behavior.
* [ ] Audit sample shows no PII in logs; redaction working.
* [ ] Restore drill (RDS snapshot → test env) performed within last quarter.

---

## 11) Acceptance criteria (security)

* Clerk → API auth path verified end-to-end; invalid tokens rejected with 401.
* Tenant scoping enforced; cross-org read/write attempts fail.
* Uploads are constrained (type/size/TTL) and validated post-upload; S3 buckets non-public.
* AI service unreachable from the public internet; requires internal API key.
* All deployable images are signed and pass vulnerability gates.
* IR playbooks are discoverable; on-call can execute first-hour checklist without guesswork.

---

## 12) Ownership & reviews

* **Control owners:** Platform (VPC/IAM/secrets), Backend (authz/GraphQL), AI (e2e moderation/filters), Web/Mobile (Clerk integration), SRE (observability, response).
* **Review cadence:** Security review on major features; quarterly control audit; annual IR drill.

