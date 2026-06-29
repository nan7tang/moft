# Security audit

Findings from a senior-developer cyber-security review of the codebase as of the handover snapshot. Each finding has a stable ID for cross-referencing in commits and tickets.

| Severity | Count |
|---|---|
| **Critical** | 0 |
| **High** | 2 (fixed / partially fixed) |
| **Medium** | 4 |
| **Low** | 3 |
| **Informational** | 2 |

---

## S-01 — Hardcoded MapTiler API key (HIGH → mitigated)

**File:** [`src/app/components/GlobalMonitor.tsx:332-336`](../src/app/components/GlobalMonitor.tsx)
**Status:** ✅ Mitigated. Key is now read from `import.meta.env.VITE_MAPTILER_KEY` with a fallback to the demo key so the prototype still runs without a `.env` file.

**Why it matters:** Even after env wiring, the fallback string still embeds the demo key in the bundle. Demo keys are rate-limited and revocable by MapTiler at any time.

**Remaining action (backend team must do):**
1. Generate a new MapTiler key under the production MOFT account (https://cloud.maptiler.com/account/keys).
2. Add it to your production secret manager and inject as `VITE_MAPTILER_KEY` during `vite build`.
3. **Remove the fallback string** from `GlobalMonitor.tsx` once the env var is reliably present:
   ```ts
   // Before:
   const mapTilerKey = import.meta.env.VITE_MAPTILER_KEY || "npfqIoL83A1XBKCbI1TA";
   // After:
   const mapTilerKey = import.meta.env.VITE_MAPTILER_KEY;
   if (!mapTilerKey) throw new Error("VITE_MAPTILER_KEY missing");
   ```
4. Revoke the old demo key in MapTiler's dashboard.

---

## S-02 — No Content Security Policy (HIGH)

**File:** [`index.html`](../index.html)
**Status:** Not fixed (low-priority for a static demo; required before production).

**Why it matters:** Without a CSP, an attacker who lands a single XSS payload anywhere in the app can exfiltrate session tokens or load arbitrary scripts. Even though the current XSS attack surface is small (see S-05), defence in depth is required for a government dashboard.

**Recommended CSP** for production (add as a `<meta>` tag or — better — as an HTTP header from your edge/CDN):

```http
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com data:;
  img-src 'self' data: https://api.maptiler.com;
  connect-src 'self' https://api.maptiler.com https://api.moft.gov.ae;
  worker-src 'self' blob:;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

Notes:
- `style-src` needs `unsafe-inline` because the Figma export uses inline `style={...}` extensively. Tightening this is a bigger refactor — track as tech debt.
- `worker-src 'self' blob:` is required for MapTiler's web workers.
- Update `connect-src` to your real API origin once known.

---

## S-03 — `dangerouslySetInnerHTML` usage (MEDIUM)

The codebase uses `dangerouslySetInnerHTML` in 4 places. All inputs are static assets — there is no current XSS vector — but the pattern is risky as the app grows.

| File:Line | Source | Risk |
|---|---|---|
| [`src/app/components/AIChat.tsx`](../src/app/components/AIChat.tsx) | `?raw` SVG import (sparkles icon, static) | **Low** — build-time asset |
| [`src/app/components/CountryMap.tsx`](../src/app/components/CountryMap.tsx) | `?raw` SVG import (country outlines, static) | **Low** — build-time asset |
| [`src/app/components/GlobalMonitor.tsx`](../src/app/components/GlobalMonitor.tsx) (~lines 412–610) | Marker `innerHTML` built via template literals with `t()` translations interpolated | **Low** today, **Medium** if a translation key ever contains user-generated content |
| [`src/app/components/ui/chart.tsx`](../src/app/components/ui/chart.tsx) | Recharts-generated SVG | **Low** — vendor library |

**Action:**
1. Add a unit test / lint rule preventing future `dangerouslySetInnerHTML` without a comment explaining the trust boundary.
2. If the backend ever surfaces user-generated text into translations, those values **must** be sanitized server-side before being added to the dictionary.

---

## S-04 — Token storage in `sessionStorage` (MEDIUM)

**File:** [`src/app/data/api/client.ts:31`](../src/app/data/api/client.ts) (`readAuthToken()`)
**Status:** Stub, awaiting real auth wiring.

**Why it matters:** `sessionStorage` is readable by any JavaScript on the same origin. A single XSS payload steals the token.

**Recommended fix:** Once auth is wired, move the refresh token to an `httpOnly`, `Secure`, `SameSite=Strict` cookie. Keep the short-lived access token in memory (not storage) and refresh via the cookie. See [AUTHENTICATION §Token strategy](./AUTHENTICATION.md#token-strategy-decision).

---

## S-05 — No fetch error / abort handling visible to the user (MEDIUM)

**File:** N/A — currently no fetches happen.
**Status:** Will surface once backend is wired.

**Action when wiring backend:**
1. Wrap each accessor call in a try/catch (or React Query / SWR with `onError`).
2. Surface errors as in-app toasts using a shared `<Toaster />` component (consider `sonner` — already a dep).
3. Log `traceId` from the error envelope to Sentry breadcrumbs (see `VITE_SENTRY_DSN`).
4. On `401 AUTH_EXPIRED`, redirect to login transparently — don't crash the UI.

---

## S-06 — No input validation on dismiss/restore reason (MEDIUM)

**File:** [`src/app/App.tsx`](../src/app/App.tsx) — `handleDismissRisk` / `handleRestoreRisk` modals.
**Status:** Frontend allows empty submission of the reason field even though the modal copy says "* Required".

**Fix:**
1. Disable the submit button until `dismissReason.trim().length >= 10` (per backend constraint).
2. Backend still validates (defense in depth) and returns `422` if invalid.
3. Backend should also rate-limit dismiss actions per user (e.g. 10/min) to prevent automation.

---

## S-07 — `localStorage` usage scope (LOW)

**File:** [`src/app/context/LanguageContext.tsx`](../src/app/context/LanguageContext.tsx) — writes `moft.lang`.
**Status:** Low-sensitivity (language preference only). No fix needed; documented for completeness.

---

## S-08 — Console logs in production builds (LOW)

The dev build outputs MapTiler SDK banner + React DevTools hint. These are removed by Vite's production build.

**Action:** Verify after `npm run build` that the production bundle has no `console.log` or sensitive debug output. Add `terserOptions.compress.drop_console: true` to `vite.config.ts` if you want a hard guarantee.

---

## S-09 — Outdated transitive dependencies (LOW)

Run `npm audit` regularly. No known critical vulnerabilities at handover snapshot (May 2026), but the lockfile will age.

**Action:**
1. Add a `npm audit` step to CI; fail the build on high/critical advisories.
2. Use Dependabot or Renovate for monthly bumps.
3. Pin React, React DOM, and MapTiler to known-good versions; allow caret ranges only for utility libraries.

---

## S-10 — Sensitive data in logs (INFORMATIONAL)

When integrating Sentry / GA: do not include `Authorization` headers, request bodies, or user emails in breadcrumbs. Sentry's default `Integrations.Breadcrumbs` config is fine; verify after wiring.

---

## S-11 — Subresource Integrity on external fonts (INFORMATIONAL)

[`src/styles/fonts.css`](../src/styles/fonts.css) imports Noto Sans from Google Fonts at runtime. For high-assurance deployments, self-host the font files (the WOFF2 files are ~150 KB total) and remove the external dependency. Also closes a CSP allowlist entry.

---

## Threat model summary

| Threat | Today | Recommended mitigation |
|---|---|---|
| XSS via untrusted text | Low (no user-generated content) | CSP (S-02), input sanitization at backend |
| Stolen credentials | Low (no auth yet) | OIDC + short-lived tokens + httpOnly cookie (S-04) |
| API abuse / scraping | Medium (no rate limit) | Backend rate-limits per [`API_REFERENCE`](./API_REFERENCE.md#rate-limiting) |
| Map tile abuse | Low (key in env) | Replace demo key (S-01); restrict by referrer in MapTiler dashboard |
| Supply-chain attack on npm | Low | Pinned lockfile + Dependabot + `npm audit` in CI (S-09) |
| Open redirect via search/router | N/A | No router in place; revisit when one is added |
| CSRF | N/A | No state-mutating cookies in use yet; revisit when auth lands |
| Clickjacking | Low | `frame-ancestors 'none'` in CSP (S-02) |

---

## Pre-production checklist

- [ ] S-01: Replace MapTiler demo key + remove fallback
- [ ] S-02: Set CSP header at the edge
- [ ] S-04: Implement httpOnly cookie + in-memory token pattern
- [ ] S-05: Wrap every `apiFetch` call site with error UX
- [ ] S-06: Disable submit until reason ≥10 chars
- [ ] S-09: Add `npm audit` to CI
- [ ] S-11: Self-host Noto Sans fonts (optional)
- [ ] Add Sentry / equivalent error tracking
- [ ] Run a third-party pentest against staging
