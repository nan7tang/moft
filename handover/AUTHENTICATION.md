# Authentication

**Status today:** No auth. Every view is open. The integration seam (token retrieval + `Authorization` header injection) is wired and tested with a stub.

**Recommended baseline:** **JWT + RBAC.** The frontend treats the token as opaque; the backend issues it (typically via OIDC against UAE PASS or Microsoft Entra).

---

## Architecture

```
        ┌──────────────┐          ┌──────────────────┐
        │  IdP         │          │  Backend         │
        │ (UAE PASS,   │   OIDC   │  /auth/callback  │
        │  Entra, …)   │ ───────► │  → issues JWT    │
        └──────────────┘          └────────┬─────────┘
                                           │ 302 to SPA with token
                                           ▼
                                  ┌──────────────────────────┐
                                  │  Frontend SPA            │
                                  │  - sessionStorage.set    │
                                  │    ("moft.authToken", t) │
                                  │  - apiFetch() reads it   │
                                  └──────────────────────────┘
```

The frontend **never sees the IdP** directly in this model. It hits backend endpoints; the backend handles the OIDC dance and gives the SPA a JWT it can present on subsequent requests.

---

## Where the integration seam lives

| File | What it does | What to change |
|---|---|---|
| [`src/app/data/api/client.ts`](../src/app/data/api/client.ts) | Reads token via `readAuthToken()`, adds `Authorization: Bearer <token>` to every request | Swap `readAuthToken()` body to read from your auth library (MSAL / Auth0 / custom) |
| [`src/app/data/api/types.ts`](../src/app/data/api/types.ts) | `UserProfile`, `UserRole` types | Add/remove roles to match your backend |

The current stub:
```ts
export function readAuthToken(): string | null {
  try { return window.sessionStorage.getItem("moft.authToken"); } catch { return null; }
}
```

For a JWT issued by your backend after OIDC: keep `sessionStorage` (or upgrade to an `httpOnly` cookie — see security trade-offs below).

---

## RBAC matrix

| Role | Read risks | Dismiss risk | Restore risk | Modify dependencies | Admin settings |
|---|:-:|:-:|:-:|:-:|:-:|
| `viewer`   | ✓ | ✗ | ✗ | ✗ | ✗ |
| `analyst`  | ✓ | ✗ | ✓ | ✗ | ✗ |
| `officer`  | ✓ | ✓ | ✓ | ✓ | ✗ |
| `admin`    | ✓ | ✓ | ✓ | ✓ | ✓ |

The backend enforces these on every endpoint. The frontend currently shows every button regardless of role — a follow-up task is to read `currentUser.roles` and hide forbidden actions (see [INTEGRATION_GUIDE §RBAC](./INTEGRATION_GUIDE.md#rbac-feature-flagging)).

---

## Recipe 1 — Generic OIDC (recommended)

Works for any provider that supports OIDC Authorization Code + PKCE (Auth0, Keycloak, Okta, Cognito, generic OIDC).

### Backend
1. Implement `GET /auth/login` — redirects to IdP with PKCE challenge.
2. Implement `GET /auth/callback` — exchanges code + verifier for IdP tokens, mints your own JWT with `{ sub, email, roles, exp }`, sets it as response body or `Set-Cookie`, redirects to `/`.
3. Implement `POST /auth/logout` — clears the session.

### Frontend
1. Add a top-level `<AuthProvider>` in `main.tsx` that:
   - On mount: read token from `sessionStorage`.
   - If missing: redirect to `/auth/login`.
   - If present: fetch `/auth/me` → render `<App />` with the user in context.
2. `readAuthToken()` already pulls from `sessionStorage["moft.authToken"]` — set it after `/auth/callback` parses the token from the URL fragment or response.

### Env vars
```
VITE_AUTH_PROVIDER=oidc
VITE_AUTH_ISSUER=https://your-idp.example.com/realms/moft
VITE_AUTH_CLIENT_ID=moft-spa
VITE_AUTH_REDIRECT_URI=https://moft.gov.ae/auth/callback
VITE_AUTH_SCOPES=openid profile email
```

---

## Recipe 2 — UAE PASS

UAE's national digital identity service (https://uaepass.ae). Same OIDC pattern as Recipe 1 with provider-specific quirks:

### Backend
1. Register the SPA with UAE PASS (production + staging URLs).
2. UAE PASS scopes: `openid profile email urn:uae:digitalid:profile:general` (depending on the level you need).
3. Validate `aud` against your client_id, `iss` against `https://id.uaepass.ae`.
4. UAE PASS returns Emirates ID details — store `eid`, `nameEn`, `nameAr`, and map to your `roles` table.

### Frontend
- Same as Recipe 1 — the SPA doesn't change.
- Optional: handle the `acr_values` claim (assurance level: SOP1/2/3) to gate sensitive actions.

### Caveats
- UAE PASS production requires NDA + government project approval.
- Test against the sandbox first.

---

## Recipe 3 — Microsoft Entra ID (Azure AD)

For UAE federal agencies on Microsoft 365.

### Backend
- Register an app in Entra, capture `tenantId` + `clientId`.
- Use the official `microsoft-identity-web` SDK or any OIDC library.
- Validate tokens against `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration`.

### Frontend
- Same backend-mediated pattern as Recipe 1.
- **Alternative:** use MSAL.js directly in the SPA (browser-side OIDC). Trades simpler backend for more SPA-side complexity. Not recommended unless your backend can't easily host an auth proxy.

---

## Token strategy decision

|  | `sessionStorage` (current) | `localStorage` | `httpOnly` cookie |
|---|---|---|---|
| **Survives tab close** | ✗ | ✓ | depends on `Max-Age` |
| **XSS-protected** | ✗ | ✗ | ✓ |
| **CSRF-protected** | ✓ | ✓ | ✗ (needs CSRF token) |
| **Easy SPA access** | ✓ | ✓ | ✗ (need parallel non-cookie session) |

**Recommendation:** Use `httpOnly` cookies for the long-lived refresh token, and ephemeral `sessionStorage` for the short-lived access token (≤15 min). The SPA reads the access token to attach `Authorization`; when it expires, a silent refresh against `/auth/refresh` mints a new one using the cookie. This is the OAuth 2.0 BFF pattern.

---

## Token shape (JWT claims)

The backend can choose, but at minimum:

```json
{
  "iss": "https://api.moft.gov.ae",
  "aud": "moft-spa",
  "sub": "8f3c1b2a-2f43-4c0e-9a18-1ce6b8b22013",
  "exp": 1748694000,
  "iat": 1748693100,
  "email": "sarah.johnson@moft.gov.ae",
  "name": "Sarah Johnson",
  "roles": ["analyst"],
  "dept": "Strategic Sourcing"
}
```

Frontend reads `roles` for client-side button hiding; backend re-checks `roles` on every request (never trust client-side decisions).

---

## Session lifecycle

| Event | What the SPA does |
|---|---|
| App loads, no token | Redirect to `/auth/login` (handled by `<AuthProvider>`) |
| Token expires mid-session | `apiFetch()` gets `401` → Provider attempts silent refresh; if that fails, redirect to login |
| User clicks "Sign out" | `POST /auth/logout`, clear `sessionStorage`, redirect to `/auth/login` |
| Token tampering (signature invalid) | Backend returns `401 AUTH_INVALID`; SPA treats as expired |
| Role demotion | `/auth/me` returns the new roles on next refresh; UI re-renders accordingly |

---

## Frontend audit checklist (post-integration)

- [ ] Every `apiFetch` call hits an authenticated endpoint (no anonymous endpoints unless intentional).
- [ ] `<AuthProvider>` redirects to login on missing/expired token.
- [ ] `currentUser.roles` is read from context, never hardcoded.
- [ ] Buttons that map to officer/admin endpoints are hidden for lower roles.
- [ ] Token is never logged (verify `console.log` / Sentry breadcrumbs).
- [ ] No tokens in URL fragments after callback completes.

---

## See also

- [SECURITY_AUDIT.md](./SECURITY_AUDIT.md) — broader cyber-security findings
- [INTEGRATION_GUIDE.md](./INTEGRATION_GUIDE.md) — step-by-step backend swap
- OWASP ASVS auth controls: https://owasp.org/www-project-application-security-verification-standard/
