# Backend — Go + GraphQL (Relay) + ent + Postgres + S3

> **Scope:** The backend service that exposes a Relay-compatible GraphQL API, uses **ent** for data access, authenticates with **Clerk**, stores objects in **S3**, and calls the internal **AI** service.  
> **Audience:** Backend engineers, SRE, and code agents.

---

## 1) At-a-glance

- **Language/Runtime:** Go (1.22+)  
- **API:** GraphQL via **gqlgen**, Relay-compatible schema & pagination  
- **ORM:** **ent** (code-gen schema, migrations, typed queries)  
- **DB:** PostgreSQL (RDS in cloud; local Postgres in dev)  
- **Auth:** **Clerk** (frontend sessions → backend verifies JWT / org claims)  
- **Storage:** **S3** (pre-signed PUT/GET flows)  
- **AI:** Internal **FastAPI** service (REST) for generation/embeddings/filters  
- **Deploy:** Docker image to **AWS App Runner** (with VPC connector for RDS/S3)  
- **Obs:** OpenTelemetry traces/metrics/logs → CloudWatch/Grafana  
- **Security:** Least-priv IAM, KMS, rate limits, GraphQL complexity/depth limits

```mermaid
flowchart LR
  Client["Web / RN Client (Relay)"]
  Clerk["Clerk (Auth)"]
  API["Go GraphQL (gqlgen)"]
  ENT["ent (ORM)"]
  PG["Postgres (RDS)"]
  S3["S3 (pre-signed)"]
  AI["AI Service (FastAPI)"]

  Client -- Bearer <Clerk JWT> --> API
  API -. verify .-> Clerk
  API --> ENT --> PG
  API --> S3
  API --> AI
````

---

## 2) Service surface

* **HTTP**: `POST /graphql` (JSON), `GET /healthz`, `GET /readyz`, `GET /metrics`
* **Auth**: `Authorization: Bearer <Clerk JWT>` required on GraphQL
* **CORS**: Locked to known frontends; credentials disabled
* **Compression**: gzip/br for responses
* **Rate limits**: per user/org + IP (token bucket); stricter on mutations

---

## 3) Configuration (env)

Minimal set (dev example):

```
PORT=8080
GRAPHQL_PLAYGROUND=true

# Database
DATABASE_URL=postgres://app:app@localhost:5432/app?sslmode=disable

# AWS
AWS_REGION=us-east-1
S3_BUCKET_UPLOADS=uaif-uploads-dev

# Clerk
CLERK_JWKS_URL=https://<your-clerk>.clerk.accounts.dev/.well-known/jwks.json
CLERK_ISSUER=https://<your-clerk>.clerk.accounts.dev
CLERK_AUDIENCE=<your-aud>            # optional, if you scope tokens by aud

# Downstream services
AI_SERVICE_URL=http://localhost:8081
```

> In cloud, secrets/creds come from **AWS Secrets Manager**, not `.env`.

---

## 4) Project structure (recommended)

```
api/
  cmd/server/main.go           # wire config, http, gql, otel
  graph/
    schema.graphqls            # Relay-style SDL
    generated.go               # gqlgen output
    resolver.go                # root resolver
    loaders/                   # DataLoader bundle (per request)
    middlewares/               # auth, tracing, complexity/depth limits
  internal/
    auth/clerk.go              # JWT verify, claims → Context
    db/ent/                    # ent client + generated code
    db/migrate/                # migration helpers
    s3/store.go                # pre-signed URLs
    ai/client.go               # AI HTTP client
    otel/otel.go               # OTel setup
    ids/ids.go                 # global Relay ID helpers
  pkg/
    errorsx/                   # error types + mapping to GraphQL
    ratelimit/                 # limiter interface + impl
```

---

## 5) Authentication & Context (Clerk)

### 5.1 Verify tokens

* Accept **Clerk** JWTs from the client.
* Verify using JWKS (cache for ~5–15 min; rotate on `kid` change).
* Validate `iss` (and optionally `aud`) and `exp`.
* Extract **userId**, **orgId**, **roles/permissions** into a request **Context**.

```go
// internal/auth/clerk.go (sanitized)
type Claims struct {
  UserID string
  OrgID  string
  Roles  []string
  Email  string
}

func VerifyAndExtract(ctx context.Context, token string, cfg Config) (*Claims, error) {
  // 1) Parse + verify with JWKS (cached)
  // 2) Validate iss/aud/exp
  // 3) Map into Claims; return typed error on failure
  return &Claims{UserID: "...", OrgID: "...", Roles: []string{"user"}}, nil
}
```

### 5.2 Inject into GraphQL

Attach a middleware on `/graphql` that:

1. Reads `Authorization` header
2. Calls `VerifyAndExtract`
3. Stores `Claims` in context for resolvers & DataLoaders

```go
// graph/middlewares/auth.go (sanitized)
type ctxKey int
const claimsKey ctxKey = 1

func WithAuth(next http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    token := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
    if token == "" { http.Error(w, "missing auth", http.StatusUnauthorized); return }
    claims, err := auth.VerifyAndExtract(r.Context(), token, cfg)
    if err != nil { http.Error(w, "invalid token", http.StatusUnauthorized); return }
    next.ServeHTTP(w, r.WithContext(context.WithValue(r.Context(), claimsKey, claims)))
  })
}

func ClaimsFrom(ctx context.Context) *auth.Claims {
  c, _ := ctx.Value(claimsKey).(*auth.Claims); return c
}
```

---

## 6) GraphQL schema guidelines (Relay)

### 6.1 Core conventions

* Implement `Node` interface with **global IDs**.
* Use **Connections** (edges/node/pageInfo) for lists.
* Return resource in mutation payloads (and common errors where helpful).
* Avoid raw timestamps on edges—keep them on nodes; add stable sort keys.

```graphql
# graph/schema.graphqls (sanitized)
interface Node { id: ID! }

type User implements Node {
  id: ID!
  email: String!
  createdAt: Time!
}

type Document implements Node {
  id: ID!
  owner: User!
  title: String!
  createdAt: Time!
  updatedAt: Time!
}

type DocumentEdge { node: Document! cursor: String! }
type DocumentConnection { edges: [DocumentEdge!]! pageInfo: PageInfo! }
type PageInfo { hasNextPage: Boolean! endCursor: String }

type Query {
  me: User
  documents(first: Int!, after: String): DocumentConnection!
  document(id: ID!): Document
}

type Mutation {
  createDocument(title: String!): CreateDocumentPayload!
}

type CreateDocumentPayload { document: Document! }
```

### 6.2 gqlgen config (key points)

* Map custom scalars (`Time`, etc.)
* Enable **complexity** function & **depth** limit
* Use `omit_slice_element_pointers: true` to reduce pointer churn

```yaml
# gqlgen.yml (sanitized)
schema:
  - graph/schema.graphqls
exec:
  filename: graph/generated.go
model:
  filename: graph/models_gen.go
resolver:
  layout: follow-schema
  dir: graph
autobind:
  - internal/db/ent
omit_slice_element_pointers: true
```

---

## 7) ent patterns

### 7.1 Schema & edges

* Use UUIDs at the **DB level**, derive Relay global IDs at the API boundary.
* Track `created_at` / `updated_at` in a mixin.
* Add composite indexes for common filters (e.g., `org_id`, `owner_id`).

```go
// internal/db/ent/schema/document.go (sanitized)
func (Document) Fields() []ent.Field {
  return []ent.Field{
    field.UUID("id", uuid.UUID{}).Default(uuid.New),
    field.UUID("owner_id", uuid.UUID{}),
    field.String("title"),
    field.Time("created_at").Default(time.Now),
    field.Time("updated_at").UpdateDefault(time.Now),
  }
}
func (Document) Edges() []ent.Edge {
  return []ent.Edge{
    edge.From("owner", User.Type).Ref("documents").Field("owner_id").Unique().Required(),
  }
}
```

### 7.2 Pagination

Prefer ent’s cursor helpers or a custom paginator backed by stable `(created_at, id)` ordering.

```go
// graph/resolver_document.go (sanitized)
func (r *queryResolver) Documents(ctx context.Context, first int, after *string) (*DocumentConnection, error) {
  q := r.DB.Document.Query().Order(ent.Desc(document.FieldCreatedAt), ent.Desc(document.FieldID))
  if c := decodeAfter(after); c != nil {
    q = q.Where(predicate.Document(
      document.Or(
        document.CreatedAtLT(c.CreatedAt),
        document.And(document.CreatedAtEQ(c.CreatedAt), document.IDLT(c.ID)),
      ),
    ))
  }
  items, err := q.Limit(first + 1).All(ctx)
  if err != nil { return nil, err }

  edges := toEdges(items, first)
  return &DocumentConnection{
    Edges: edges,
    PageInfo: PageInfo{
      HasNextPage: len(items) > first,
      EndCursor:   lastCursor(edges),
    },
  }, nil
}
```

### 7.3 N+1 avoidance

Bundle **DataLoaders** per request (userByID, docByID, etc.) and fetch in batches.

```go
// graph/loaders/loaders.go (sanitized)
type Loaders struct {
  Users *dataloader.Loader[string, *ent.User]
}

func NewLoaders(db *ent.Client) *Loaders {
  batchFn := func(ctx context.Context, keys []string) ([]*ent.User, []error) {
    // 1 query IN (keys) → map back to order
  }
  return &Loaders{ Users: dataloader.New(batchFn, dataloader.WithBatchCapacity(100)) }
}
```

---

## 8) Authorization & policies

* Derive **policy inputs** from context claims (user/org/roles).
* **Check early** in resolvers before DB work (fail fast).
* For multi-tenant data, scope queries by `org_id` in a central helper.
* Log **authz decisions** (debug) and redact PII.

```go
// internal/auth/policy.go (sanitized)
func CanReadDoc(ctx context.Context, d *ent.Document) bool {
  c := ClaimsFrom(ctx)
  return c != nil && (c.OrgID == d.OrgID || hasRole(c, "admin"))
}
```

---

## 9) S3 pre-signed URLs

* **Uploads:** client asks GraphQL for a pre-signed PUT → uploads directly to S3 → backend validates on finalize.
* **Downloads:** pre-signed GET with short TTL when needed.

```go
// internal/s3/store.go (sanitized)
func (s *Store) PresignPut(ctx context.Context, key, contentType string, ttl time.Duration) (string, error) {
  // s3manager / v2 sdk: s.Client.PresignPutObject(ctx, &s3.PutObjectInput{ Bucket: s.bucket, Key: &key, ContentType: &contentType }, s3.WithPresignExpires(ttl))
  return url, nil
}
```

---

## 10) AI service client

* **Contract:** internal REST (e.g., `POST /v1/generate`) with user/org context headers (no PII).
* **Timeouts & CB:** 95th ≤ 2.5s; use timeouts, retries (idempotent ops), and a circuit breaker.

```go
// internal/ai/client.go (sanitized)
type GenerateReq struct { Prompt string; MaxTokens int }
type GenerateResp struct { Text string; LatencyMS int; Provider string }

func (c *Client) Generate(ctx context.Context, req GenerateReq) (GenerateResp, error) {
  ctx, cancel := context.WithTimeout(ctx, 6*time.Second); defer cancel()
  // POST to AI_SERVICE_URL/v1/generate with JSON body
  // propagate traceparent; include X-Org-ID, X-User-ID headers
  // parse JSON; map provider errors → typed errors
  return resp, nil
}
```

---

## 11) Observability

* **Tracing:** Wrap HTTP server and GraphQL resolvers with OTel; propagate `traceparent` to AI/S3/DB.
* **Metrics:** RED (Rate, Errors, Duration) per operation; DB pool stats; AI token/latency histograms.
* **Logs:** Structured JSON with request IDs and pseudonymous user/org IDs.
* **Dashboards:** p50/p95/p99 latency, error rates by operation, AI cost/time, DB saturation.

---

## 12) Security

* **GraphQL hardening:** query depth, complexity, and max input size; deny introspection in prod (or restrict).
* **Validation:** strict types + input validation on mutations.
* **PII:** never log PII; use audit trail for admin actions and exports.
* **IAM:** least-priv roles for App Runner tasks; scoped S3 access to upload bucket paths.
* **Secrets:** AWS Secrets Manager (DB creds, Clerk keys); rotate regularly.
* **Backups:** RDS PITR enabled; S3 lifecycle policies (IA/Glacier).

---

## 13) Local development

* **DB:** `docker run` local Postgres or `docker compose up db`.
* **Migrations:** ent migration at boot (dev only) or via a `migrate` subcommand.
* **Seed:** idempotent seeds behind a `SEED=true` guard.
* **Running:** `go run cmd/server/main.go` (loads `.env`); GraphQL playground if enabled.

---

## 14) Testing

* **Unit:** ent model logic, policy checks, and small resolvers.
* **Integration:** GraphQL queries/mutations against a test DB (transaction per test).
* **Contract:** golden GraphQL responses for key flows; snapshot tokens redacted.
* **Load:** k6/vegeta smoke on critical paths (auth → query → AI call).

---

## 15) Runbook (abridged)

* **401s suddenly everywhere** → JWKS cache stale or Clerk outage → refresh JWKS, verify `iss`; switch to cached allow-last-good for <5m if needed.
* **DB pool exhaustion** → check max conns, slow queries, N+1; enable query logging sample; add loader/batching.
* **AI 5xx or timeouts** → trip CB, degrade features (cached/tiny model), notify provider; re-enable after recovery.
* **S3 upload failures** → check clock skew, content-type mismatch, or expired pre-sign TTL; re-issue.

---

## 16) Acceptance checks

* GraphQL responds with **200** to a simple `me` query with a valid Clerk JWT.
* Documents list paginates with Relay **Connections** and stable cursors.
* S3 **pre-sign** mutation returns a working PUT URL; upload succeeds; finalize records metadata.
* AI **generate** mutation returns text and latency metadata; traces span API→AI→provider.

