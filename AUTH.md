# AUTH — Identity & Access (Clerk + Backend + Services)

> **Scope:** End-to-end authentication and authorization across mobile/web (Clerk), the Go GraphQL API, and internal services.  
> **Design goals:** Simple sign-in UX, strict server verification, clear tenant/role context for resolvers, and private service-to-service auth (no end-user tokens beyond the API boundary).

---

## 1) Mental model

Two compatible modes are supported:

- **Mode A — Pass-through Clerk JWT:** Client attaches a **Clerk session token** (`Authorization: Bearer …`) to GraphQL; API verifies with Clerk’s JWKS. This is the simplest path and is fully supported in the backend. :contentReference[oaicite:0]{index=0}  
- **Mode B — Exchange for App Token:** Client signs in with Clerk, then calls a backend **exchange** endpoint to mint a short-lived **app JWT**. GraphQL thereafter uses this app token. This adds control over claims/TTL and allows bespoke per-app audiences. :contentReference[oaicite:1]{index=1} :contentReference[oaicite:2]{index=2}

Both modes are wired in your code paths (mobile exchange helper + backend verification).

---

## 2) Frontend (Clerk) → Token flow

### 2.1 RN deep-link sign-in (hosted page → app)
- App opens Clerk’s hosted sign-in in a secure browser, deep-linking back to a native scheme. If Clerk returns `session_token`, the app can immediately proceed; if it returns `session_id`, the app calls the backend exchange endpoint. :contentReference[oaicite:3]{index=3} :contentReference[oaicite:4]{index=4}

### 2.2 Clerk provider & hydration
- The app mounts `ClerkProvider` and uses the SDK’s token cache, with a small “reload” context to force re-hydration after session changes. :contentReference[oaicite:5]{index=5} :contentReference[oaicite:6]{index=6} :contentReference[oaicite:7]{index=7}

### 2.3 Auto-exchange on existing sessions
- After sign-in (or if a session already exists), the app ensures an app session by performing the exchange and then refreshing `me`. :contentReference[oaicite:8]{index=8}

**Sanitized fetch header (client → GraphQL):**
```text
Authorization: Bearer <clerk-or-app-token>
````

---

## 3) Backend (Go) — Verify, upsert, and context

### 3.1 Clerk verification (server-side)

* The Go API uses Clerk’s SDK with a **per-app JWKS client** and verifies the `Authorization: Bearer` token on each request. This is the default path for Mode A; it’s also how you verify Clerk tokens during Mode B’s exchange.   

### 3.2 Per-app configuration

* The verifier supports multiple logical apps/audiences via env-namespaced keys (e.g., `_APP`, `_APP2`) and falls back to generic variables when unspecified.   

### 3.3 Optional Clerk profile fetch

* When needed (e.g., first sign-in), the server calls Clerk’s Users API with the secret key to populate local profile fields and then upserts user/org rows.  

**Resolver context (recommended fields):**

```
ctx.user_id        # required
ctx.org_id         # tenant (nullable)
ctx.roles[]        # derived from claims and/or DB
ctx.rate_limit_key # user_id or (org_id,user_id) composite
```

---

## 4) The exchange pattern (Mode B)

**Intent:** Keep Clerk as identity, but have the backend mint a short-lived **app JWT** with the exact claims/TTL you want (e.g., per-product audience, org scoping). Your RN app already performs this when Clerk returns a `session_id` on redirect.  

**Contract (sanitized):**

```
POST /auth/clerk/exchange
Body:   { "session_id": "<from-clerk>" }   # or "session_token"
Reply:  { "token": "<app-jwt>" }           # short TTL; includes org/roles if available
```

**Where it’s used:** RN `ensureAppSessionFromClerk()` path to finalize sign-in and then `refreshMe`. 

---

## 5) GraphQL authorization & tenancy

* **Resolver guardrails:** Authorize early per resolver/mutation using `ctx.user_id`, `ctx.org_id`, and `ctx.roles`. Deny before DB work when possible (fail fast).
* **Tenant scoping:** Centralize org-scoped query helpers; default all reads/writes to the current org unless the role is elevated.
* **Rate limits:** Use `ctx.rate_limit_key` (user or (org,user)) for token-bucket quotas; apply stricter limits on mutations.
* **Audit:** Record auth events and sensitive operations (exports/admin actions).

---

## 6) Service-to-service (API → AI service)

* The **AI service does not accept Clerk tokens**. It expects a private **API key** header and optional context headers (e.g., `X-User-ID`, `X-Org-ID`) containing only non-PII identifiers. Keep the AI service on private networking (VPC/VPC Connector / SG allowlist).
* Prefer **short-lived service tokens** or an **HMAC** header if you need request integrity; rotate keys regularly.
* Propagate `traceparent` and an `Idempotency-Key` for safe retries; never forward raw user PII.

---

## 7) Webhooks (optional but recommended)

* **User provision:** On `user.created` / `session.created`, upsert `User` and (if relevant) `Organization`/membership rows.
* **Profile sync:** On `user.updated`, refresh name/image/email.
* **Revocation:** Honor session revocations and token invalidations immediately.
* Secure webhooks with shared secret verification; store secrets in your secret manager.

---

## 8) Environment & secrets

### 8.1 Frontend (RN/React)

```
CLERK_PUBLISHABLE_KEY=pk_...               # front-end key
CLERK_FRONTEND_URL=https://<clerk-instance>
CLERK_SIGN_IN_PATH=/sign-in
OAUTH_REDIRECT_URI=app://oauth-native-callback
```

(Your RN config and docs emphasize the deep-link and constructed `GRAPHQL_URL`.) 

### 8.2 Backend (Go)

```
# Generic (fallback)
CLERK_SECRET_KEY=sk_...
CLERK_JWT_ISSUER=https://<your-issuer>
CLERK_JWT_AUDIENCE=<audience>

# Per-app overrides (examples)
CLERK_SECRET_KEY_APP=sk_...
CLERK_JWT_ISSUER_APP=...
CLERK_JWT_AUDIENCE_APP=...
CLERK_SECRET_KEY_APP2=...
...
```

(Loaded via `loadClerkConfig`, with per-app suffixes and sensible fallbacks.)  

---

## 9) Operational checks (“done means”)

* **Client → API:** A GraphQL `me` query with a valid Clerk JWT returns `200`; the server verifies via JWKS and sets a resolver context. 
* **Exchange:** When Clerk returns a `session_id`, `/auth/clerk/exchange` responds with an app token; the app then loads user data.  
* **Profile upsert:** First-time sign-ins populate local user profile from Clerk Users API without logging PII. 
* **Service calls:** API → AI includes `X-API-Key`; calls succeed without end-user tokens.

---

## 10) Troubleshooting

* **401 / Missing Bearer:** Ensure `Authorization: Bearer` prefix is present; the verifier logs a specific warning. 
* **State/redirect mismatches on mobile:** Verify deep-link host and allowlist; the helper checks callback host and state. 
* **Exchange fails:** Ensure `/auth/clerk/exchange` is reachable; confirm Clerk returned `session_token` **or** `session_id`. 

---

## 11) Minimal reference snippets (sanitized)

> These are illustrative; actual code lives in the repos.

**RN: ensure app session after Clerk sign-in**

```ts
// ensures app token exists; called after any successful Clerk session
await ensureAppSessionFromClerk(); // then refreshMe()
```

(Your Login flow already does this.) 

**Go: Authorization middleware skeleton**

```go
// read Authorization, verify via Clerk JWKS, build resolver context {user, org, roles}
// deny on missing/invalid; rate-limit key = user or (org,user)
```

(The concrete implementation uses `jwt.Verify` with a per-app JWKS client.) 

---

## 12) Security notes

* Cache JWKS briefly and refresh on `kid` changes.
* Rotate **Clerk secret keys** and any **app-JWT signing keys** on a schedule.
* Never log tokens or raw PII; use pseudonymous IDs in logs.
* Keep the AI service dark to the internet; authenticate with a private key and allowlist.

---

## 13) References (internal)

* **Mobile Clerk flow:** deep-link + exchange helper `src/auth/clerkNative.ts`. 
* **RN provider wiring:** `src/context/ClerkProvider.tsx`. 
* **Auth context & acceptance:** `docs/AGENT_CONTEXT.md`. 
* **Backend verifier:** `internal/auth/clerk.go`. 

