# Integration guide

How a backend developer (or an AI coding agent) wires a real backend into this codebase. Estimated effort: **1–2 days** for a developer working in parallel with the backend team.

---

## 0. Pre-flight

1. Read [ARCHITECTURE.md](./ARCHITECTURE.md) end-to-end. The most important section is §4 (Data flow).
2. Skim [DATA_MODELS.md](./DATA_MODELS.md) for type shapes.
3. Have [API_SPECIFICATION.yaml](./API_SPECIFICATION.yaml) open in a viewer (Swagger UI, Insomnia, etc.). This is the contract.
4. Copy `.env.example` → `.env.local` and fill in real values.
5. `npm install && npm run dev` — confirm the mock app boots cleanly at `http://localhost:5173`.

---

## 1. The integration seam

Every backend call funnels through `src/app/data/api/`. There are three layers:

```
src/app/data/api/
├── client.ts        ← apiFetch() — auth header, timeout, error envelope
├── types.ts         ← TypeScript types matching the OpenAPI schemas
├── risks.ts         ← getActiveRisks / getArchivedRisks / getGlobalRisks
├── dependencies.ts  ← getDependencies / getDependenciesByCountry
└── … (add more per resource as needed)
```

**Rule of thumb:** components do not call `fetch` directly. They only call accessors from this folder. When the backend is wired, the accessor swap is contained — components stay the same shape.

---

## 2. Swap one accessor at a time (the safe path)

The pattern, demonstrated for `getActiveRisks()`:

### Before (current — mock)
```ts
// src/app/data/api/risks.ts
import activeRisksJson from "../mock/active-risks.json";

export function getActiveRisks(): ActiveRisk[] {
  return activeRisksJson as ActiveRisk[];
}
```

### After (backend)
```ts
// src/app/data/api/risks.ts
import { apiFetch } from "./client";

export async function getActiveRisks(): Promise<ActiveRisk[]> {
  return apiFetch<ActiveRisk[]>("/risks?status=active");
}
```

### Update the consumer
```tsx
// Before
const [activeRisks] = useState(() => getActiveRisks());

// After
const [activeRisks, setActiveRisks] = useState<ActiveRisk[]>([]);
const [error, setError] = useState<Error | null>(null);
useEffect(() => {
  let mounted = true;
  getActiveRisks()
    .then((data) => mounted && setActiveRisks(data))
    .catch((e) => mounted && setError(e));
  return () => { mounted = false; };
}, []);
```

**Better:** introduce React Query (`@tanstack/react-query`) and replace the manual `useEffect` boilerplate with `useQuery({ queryKey: ["risks", "active"], queryFn: getActiveRisks })`. Adds ~12 KB to the bundle, saves ~3 lines per call site, and gives you caching + retries for free.

---

## 3. File-by-file migration map

Use this as a checklist. Each row is one "swap" — typically 10–30 min of work.

| Step | Component | Accessor today | Backend endpoint | Notes |
|---|---|---|---|---|
| 3.1 | `App.tsx` `useState(() => getActiveRisks())` | [`api/risks.ts#getActiveRisks`](../src/app/data/api/risks.ts) | `GET /risks?status=active` | Pair with `setActiveRisks`; remove the JSON import once backend confirmed. |
| 3.2 | `App.tsx` `useState(() => getArchivedRisks())` | [`api/risks.ts#getArchivedRisks`](../src/app/data/api/risks.ts) | `GET /risks?status=archived` | |
| 3.3 | `App.tsx` `handleRestoreRisk` | [`api/risks.ts#restoreRisk`](../src/app/data/api/risks.ts) | `POST /risks/{id}/restore` | Add optimistic update + error revert. |
| 3.4 | `App.tsx` `handleDismissRisk` | `api/risks.ts#dismissRisk` (add) | `POST /risks/{id}/dismiss` | |
| 3.5 | `Dependencies.tsx` `dependenciesData = getDependencies()` | [`api/dependencies.ts#getDependencies`](../src/app/data/api/dependencies.ts) | `GET /dependencies` | |
| 3.6 | `Dependencies.tsx` `getCountryDependencies` (still inline) | new `api/dependencies.ts#getDependenciesByCountry` | `GET /dependencies?country={name}` | Lifts the inline `countryData` map out of the component. |
| 3.7 | `GlobalMonitor.tsx` `risks` array (still inline, 9 entries) | new `api/risks.ts#getGlobalRisks` (already exists) | `GET /risks/global` | Replace the inline array with `const risks = getGlobalRisks();`. Re-test marker rendering. |
| 3.8 | `App.tsx` `riskDetailData` (still inline, 9 entries) | new `api/risks.ts#getRiskDetail(id)` | `GET /risks/{id}/detail` | Lazy-fetch on detail page mount; cache by id. |
| 3.9 | `App.tsx` `impactChainData` (still inline) | folded into `riskDetailData.impactChain` (already is in the API spec) | (same as 3.8) | |
| 3.10 | `CriticalGoods.tsx` `CRITICAL_GOODS` | new `api/criticalGoods.ts#getCriticalGoods` | `GET /critical-goods` | Currently in `data/criticalGoodsData.ts` — extract to JSON first. |
| 3.11 | `NotificationsContext.tsx` `INITIAL_NOTIFICATIONS` | new `api/notifications.ts#getNotifications` | `GET /notifications` | Includes unread count. |
| 3.12 | `NotificationsContext.tsx` mark-read actions | `api/notifications.ts#markRead` / `markAllRead` | `POST /notifications/{id}/read` / `POST /notifications/read-all` | |
| 3.13 | `GlobalSearchResults.tsx` in-file index | new `api/search.ts#search(q)` | `GET /search?q={q}` | Replace the four hardcoded `*_INDEX` constants with a single fetch. |
| 3.14 | New: AuthProvider | new `api/auth.ts#getCurrentUser` | `GET /auth/me` | See [AUTHENTICATION](./AUTHENTICATION.md). |

---

## 4. Add `apiFetch()` to the bundle

The helper at [`src/app/data/api/client.ts`](../src/app/data/api/client.ts) is already written and exported. Each accessor currently imports it commented-out — uncomment when wiring:

```ts
// import { apiFetch } from "./client";  ← uncomment me
```

No other changes are needed for the helper itself.

---

## 5. Wire authentication

See [AUTHENTICATION.md](./AUTHENTICATION.md) for the full recipe. The 30-second version:

1. Add an `<AuthProvider>` at the top of `main.tsx`.
2. Inside `AuthProvider`, on mount: read `sessionStorage["moft.authToken"]`. If missing, redirect to `/auth/login`. If present, fetch `/auth/me`.
3. Optionally: read `currentUser.roles` in `AuthProvider` and pass via context.
4. Replace `readAuthToken()` in [`client.ts`](../src/app/data/api/client.ts) if you store the token differently (e.g. in-memory).

---

## 6. Add real routing

The app has no URL router today (see [ARCHITECTURE §3](./ARCHITECTURE.md#3-view-routing-model)). To add deep-linking:

```bash
npm install react-router-dom@^7
```

Replace the `currentView` switch in `App.tsx` with `<Routes>`:

```tsx
<Routes>
  <Route path="/" element={<GlobalMonitor />} />
  <Route path="/risks" element={<RiskListScreen />} />
  <Route path="/risks/:id" element={<RiskDetailPage />} />
  <Route path="/dependencies" element={<Dependencies />} />
  <Route path="/dependencies/:id" element={<DependencyDetail />} />
  <Route path="/critical-goods" element={<CriticalGoods />} />
</Routes>
```

Replace `setCurrentView("…")` calls with `navigate("/…")`. Replace `selectedRiskId` state with `useParams()`.

Effort: **0.5–1 day** including testing each navigation path.

---

## 7. Error handling

Recommended pattern using React Query:

```tsx
const { data: risks, isLoading, error } = useQuery({
  queryKey: ["risks", "active"],
  queryFn: getActiveRisks,
  staleTime: 60_000,
  retry: 1,
});

if (isLoading) return <RiskListSkeleton />;
if (error) return <ErrorBlock onRetry={refetch}>{error.message}</ErrorBlock>;
```

Manual pattern (without React Query):

```tsx
const [risks, setRisks] = useState<ActiveRisk[]>([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState<Error | null>(null);

const load = useCallback(() => {
  setLoading(true);
  setError(null);
  getActiveRisks()
    .then(setRisks)
    .catch(setError)
    .finally(() => setLoading(false));
}, []);

useEffect(load, [load]);
```

Either way, add `<Toaster />` from `sonner` (already a dep) to `main.tsx` and surface non-fatal errors as toasts.

---

## 8. RBAC feature-flagging

Once auth is wired, hide officer/admin-only actions for lower roles:

```tsx
const { user } = useAuth();
const canDismiss = user.roles.some((r) => r === "officer" || r === "admin");

{canDismiss && (
  <button onClick={openDismissModal}>Dismiss Risk</button>
)}
```

Backend re-checks on every action — never trust the client.

---

## 9. Internationalization scaling

Today the `translations.ts` dictionary is hand-edited (~330 keys). When the volume grows:

- Move to per-namespace JSON files (`translations/en/risks.json`, `translations/ar/risks.json`).
- Add a translation-management tool (Lokalise, Phrase, Crowdin) — export to JSON.
- Update `LanguageContext.tsx` to merge the namespaced files at startup.

Backend integration is **independent** of this — translation keys live entirely in the frontend repo.

---

## 10. Tests (currently zero)

Recommended additions:

| Layer | Tool | Coverage target |
|---|---|---|
| Unit (utilities, hooks, accessors) | **Vitest** | ≥ 80% of `src/app/data/` and `src/app/context/` |
| Component (rendering, interactions) | **React Testing Library** | All views in `src/app/components/` |
| E2E happy paths | **Playwright** | Login → see dashboard → dismiss risk → see archive |
| Visual regression | **Chromatic / Percy** | All major views in EN + AR |
| Contract tests | **Pact** (or schema-driven) | Every accessor against `API_SPECIFICATION.yaml` |

Suggested commands to add to `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:e2e": "playwright test",
    "type-check": "tsc --noEmit",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

---

## 11. Production checklist

Before flipping to live data:

- [ ] All accessors swapped (use this guide §3 as the checklist)
- [ ] `<AuthProvider>` mounted in `main.tsx`
- [ ] React Router added (or document why not)
- [ ] Error handling on every fetch (toast or inline)
- [ ] Loading skeletons on every async surface
- [ ] [SECURITY_AUDIT pre-prod checklist](./SECURITY_AUDIT.md#pre-production-checklist) green
- [ ] `npm run build` produces a clean `dist/` (no warnings)
- [ ] CSP header set at the edge
- [ ] Sentry / error tracking wired (`VITE_SENTRY_DSN`)
- [ ] A11y audit with axe-core, contrast verified
- [ ] Pentest against staging

---

## Appendix: AI-agent prompts

To accelerate the swap, you can paste these prompts into Claude Code, Cursor, etc.

**Swap one accessor:**
> Read `src/app/data/api/risks.ts`. Convert `getActiveRisks()` to call `apiFetch<ActiveRisk[]>("/risks?status=active")`. Then find every call site in the codebase and update it to use the new async signature (wrap in a `useState + useEffect` pair as shown in `handover/INTEGRATION_GUIDE.md §2`).

**Add error handling to a view:**
> Wrap every `getXxx()` call in `<ComponentName>.tsx` with try/catch. On error, render an inline retry block. Use the `sonner` toast for transient failures. Follow the pattern in `handover/INTEGRATION_GUIDE.md §7`.

**Add a new resource:**
> Add a new accessor `getXxx()` in `src/app/data/api/xxx.ts`. The TypeScript type goes in `src/app/data/api/types.ts`. The mock JSON goes in `src/app/data/mock/xxx.json`. Update `handover/API_SPECIFICATION.yaml` to add the new endpoint and `handover/DATA_MODELS.md` to document the type.
