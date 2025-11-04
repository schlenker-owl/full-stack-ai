# Frontend (Mobile) — React Native + Relay + Clerk

> **Scope:** Patterns and conventions for our React Native apps (Cura) that talk to the Go/GraphQL backend via **Relay**, authenticate with **Clerk**, and follow accessibility & performance best practices.  
> **Audience:** Mobile engineers & code agents.  
> **This doc avoids app-specific secrets and keeps examples sanitized.**

---

## 1) Tech & Repo Conventions

- **React Native + React 19, Relay runtime & compiler.** We keep a `relay` npm script and colocate fragments/queries next to screens/components. Relay artifacts live in `__generated__/` and are generated from a vendored SDL (`src/graphql/schema.graphql`).  
- **Clerk (Expo)** for auth; app uses Clerk’s hosted pages + native deep link → exchanges token/session for an app token; GraphQL requests carry `Authorization: Bearer <token>`.  
- **Config** via `react-native-config` and a single `api.ts` module that builds `API_BASE` → `GRAPHQL_URL` and exposes Clerk keys/redirect URI.  
- **Per-session Relay environment** with `QueryResponseCache` (QRC) and AsyncStorage persistence. Swap environments on session change to prevent data bleed.

> Quick pointers (actual paths in repo):  
> `src/graphql/schema.graphql` (vendored SDL) · `src/relay/environment.js` (session/QRC/persist) · `src/context/ClerkProvider.tsx` (Clerk) · `src/auth/clerkNative.ts` (hosted OAuth → deep link) · `src/config/api.ts` (runtime endpoints & Clerk keys)

---

## 2) Runtime Configuration

**Required env (dev):**
```

API_BASE=[http://localhost:8080](http://localhost:8080)     # GRAPHQL_URL is derived as ${API_BASE}/graphql
CLERK_PUBLISHABLE_KEY=pk_test_xxx  # or EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY
CLERK_FRONTEND_URL=https://<your-clerk-instance>
CLERK_SIGN_IN_PATH=/sign-in        # or your custom path
OAUTH_REDIRECT_URI=cura://oauth-native-callback

````

**Notes**
- Android emulator uses `10.0.2.2` if you point the app at a host-machine backend; iOS simulator uses `localhost`.  
- The app builds `GRAPHQL_URL` from `API_BASE` and exposes a *single* source of truth through `api.ts`.

---

## 3) Authentication (Clerk → App Token)

**Flow**
1) Open Clerk hosted sign-in (Google/email) in a secure web view; deep-link back to the app.  
2) Clerk returns either a `session_token` (best) or `session_id`.  
3) If `session_id`, call backend `/auth/clerk/exchange` to get an **app token (JWT)** for GraphQL.  
4) Persist token; Relay fetcher adds `Authorization: Bearer <token>` automatically.

**Sanitized example (native OAuth helper)**
```ts
// auth/openClerk.ts (sanitized)
export async function openClerkHostedSignIn(mode: 'signin'|'signup'='signin') {
  const base = process.env.CLERK_FRONTEND_URL!;
  const redirect = process.env.OAUTH_REDIRECT_URI!;     // e.g., cura://oauth-native-callback
  const path = mode === 'signup' ? '/sign-up' : (process.env.CLERK_SIGN_IN_PATH ?? '/sign-in');
  const url = `${base.replace(/\/+$/, '')}${path}?redirect_url=${encodeURIComponent(redirect)}`;

  // Use InAppBrowser if available; otherwise fall back to Linking.
  const { type, url: callback } = await openAuthBrowser(url, redirect);
  if (type !== 'success' || !callback) throw new Error('OAuth canceled');

  // If Clerk returned a session_token, you’re done; otherwise exchange session_id on the API
  const token = extractParam(callback, 'session_token')
            ?? await exchangeSessionIdForAppToken(extractParam(callback, 'session_id'));
  return token;
}
````

**Clerk Provider (sanitized)**

```tsx
// AppClerkProvider.tsx (sanitized)
import { ClerkProvider, ClerkLoaded } from '@clerk/clerk-expo';
export default function AppClerkProvider({ children }) {
  const publishableKey = process.env.EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY!;
  return (
    <ClerkProvider publishableKey={publishableKey}>
      <ClerkLoaded>{children}</ClerkLoaded>
    </ClerkProvider>
  );
}
```

---

## 4) Relay Patterns

### 4.1 Environment (session-scoped + QRC)

We maintain **one Relay Environment per session** (`'anon'` or the authenticated user id), with:

* Response cache: `QueryResponseCache` (≈5 min TTL)
* Store snapshot persisted to AsyncStorage on background/inactive
* Swappable “active” session so the app can log out/in without data bleeding

**Sanitized fetcher**

```ts
// relay/fetcher.ts (sanitized)
export const fetchGraphQL = async (operation, variables, cacheConfig, token: string) => {
  const isMutation =
    operation.operationKind === 'mutation' ||
    (operation.text && /^mutation\s/i.test(operation.text || ''));

  // Try QRC for queries unless force = true
  if (!isMutation && !cacheConfig?.force) {
    const cached = qrc.get(operation.name, variables);
    if (cached) return cached;
  }

  const res = await fetch(process.env.GRAPHQL_URL!, {
    method: 'POST',
    headers: {
      'content-type': 'application/json',
      'authorization': token ? `Bearer ${token}` : '',
      'accept': 'application/json',
    },
    body: JSON.stringify({ query: operation.text, variables, operationName: operation.name }),
  });

  const json = await res.json();
  if (!isMutation) qrc.set(operation.name, variables, json);
  if (json?.errors?.length) throw new Error(json.errors.map((e:any)=>e.message).join('; '));
  return json;
};
```

### 4.2 Queries & Mutations

* **Colocate** fragments with components. Use `useLazyLoadQuery` for screens and `useMutation` for actions.
* For list updates, prefer **Relay Connections** with updaters; for immediate UX, add **optimistic** payloads.

**Sanitized examples**

```tsx
// Screen query
const data = useLazyLoadQuery<MeQuery>(
  graphql`query MeQuery { curaMe { id onboardedAt } }`,
  {},
  { fetchPolicy: 'store-and-network' }
);

// Mutation with optimistic updater
const [commit] = useMutation(graphql`
  mutation SetIntentionMutation($text: String!) {
    setCuraIntention(text: $text) { intention { text } }
  }
`);
commit({
  variables: { text },
  optimisticResponse: { setCuraIntention: { intention: { text } } },
});
```

---

## 5) Accessibility

* **Color & contrast:** follow MD3 guidance; test light/dark themes for ≥ WCAG AA contrast.
* **Dynamic type:** use Paper’s typography tokens and avoid hard-coded font sizes.
* **Semantics:** add accessible labels/roles; respect `accessibilityHint` on actionable elements.
* **Touch targets:** ≥ 44×44 points; avoid tiny tap areas inside dense lists.
* **Motion & feedback:** keep animations subtle; provide haptics or progress for long tasks.

---

## 6) Performance

* **Lists:** Prefer **FlashList** for large/variable data; provide stable `keyExtractor`, fixed item heights when possible.
* **Relay:** Default to `store-and-network` and keep artifacts small; paginate with Connections.
* **Memoization:** `useMemo`/`useCallback` around expensive renders & handlers; avoid prop thrash.
* **Images/Video:** Use appropriately sized thumbnails; defer heavy media until on-screen.
* **Backgrounding:** Persist Relay store snapshot on app background; clear per-session caches on logout.

---

## 7) Testing (what “working” looks like)

* **Auth exchange path:** after Google sign-in, backend exchange returns an app token; `curaMe` loads and the shell routes to Onboarding/App Tabs.
* **Relay artifacts:** repository builds artifacts cleanly from the vendored SDL.
* **Navigation gate:** not-authed → Auth stack; authed with `onboardedAt` → App Tabs; missing `onboardedAt` → Onboarding.

---

## 8) FAQ / Troubleshooting

* **GraphQL 401:** missing/expired token; ensure Clerk session exchange succeeded; verify `Authorization` header is attached.
* **Relay compile fails:** SDL drift; export fresh `schema.graphql` from backend and re-run `npm run relay`.
* **Deep link doesn’t return:** Clerk allowlist/redirect mismatch; confirm `OAUTH_REDIRECT_URI` and hosted paths.
* **Android can’t reach backend on localhost:** use `10.0.2.2:PORT`.

---

## 9) What to copy when starting a new RN app

* The **session-scoped Relay environment** pattern (QRC + AsyncStorage + swap on session).
* The **Clerk hosted-page + deep-link** flow and **exchange endpoint** contract.
* The **colocation** discipline and **Connection-based** pagination patterns.
