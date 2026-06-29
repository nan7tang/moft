# Architecture

**Audience:** Backend engineers, integrators, and any developer onboarding to the codebase.
**Scope:** What the SPA is, how data flows, what is mocked, and where to plug a real backend in.

---

## 1. Stack at a glance

| Layer | Technology | Notes |
|---|---|---|
| Build / dev server | **Vite 6** | TypeScript is type-stripped only — there is no `tsconfig.json` and type errors do **not** fail the build. See [QA_REPORT](./QA_REPORT.md) for the rationale and recommended hardening. |
| UI framework | **React 18** (strict mode) | Function components + hooks. No class components anywhere. |
| Styling | **Tailwind CSS v4** (via `@tailwindcss/vite`) | No `tailwind.config.js` — arbitrary value classes are used everywhere (`font-['Noto_Sans:Regular',sans-serif]`, `bg-[#cd4747]`). |
| Map | **MapTiler SDK 4** (MapLibre under the hood) | Custom markers built as `document.createElement("div")` then `new maptilersdk.Marker({ element })`. |
| Icons | **lucide-react** + bespoke Figma SVGs | The Figma-exported globe + sparkles icons live in `src/app/components/icons/` and `src/assets/ai/`. |
| State | **React Context** (no Redux, no Zustand) | Two providers: `LanguageContext`, `NotificationsContext`. Per-view state is local `useState`. |
| Routing | **No router yet** (single `currentView` union in App.tsx) | Switching to React Router is a 1-2 day task — see [INTEGRATION_GUIDE §6](./INTEGRATION_GUIDE.md#6-add-real-routing). |
| Data layer | **Mock JSON + thin accessor functions** (now) → **REST + OpenAPI** (target) | Every accessor in `src/app/data/api/*.ts` has a `@integration` JSDoc block with the exact one-line swap to a real `apiFetch()` call. |
| PDF export | **jsPDF + jspdf-autotable** | Used by the "Generate Report" button on the Risk detail page. |

---

## 2. Top-level directory map

```
.
├── handover/                ← THIS FOLDER — backend / handover docs
├── public/                  ← (empty; Vite serves `src/` directly)
├── src/
│   ├── main.tsx             ← Entry: <LanguageProvider><NotificationsProvider><App/></...></...>
│   ├── app/
│   │   ├── App.tsx          ← View router + cross-cutting state + risk-detail page body
│   │   ├── components/      ← Feature components (GlobalMonitor, Dependencies, …)
│   │   ├── pages/           ← Per-product DependencyDetail page
│   │   ├── context/         ← LanguageContext, NotificationsContext
│   │   └── data/
│   │       ├── api/         ← ★ Integration seam (types.ts, client.ts, risks.ts, …)
│   │       ├── mock/        ← ★ JSON mocks matching API response shape
│   │       ├── countries.ts ← Country asset registry (flag + map SVGs)
│   │       ├── criticalGoodsData.ts
│   │       ├── supplierData.ts
│   │       └── translations.ts
│   ├── assets/              ← Country flags + map SVGs, AI orb PNG, sparkles SVG, category icons
│   ├── imports/             ← ★ Figma-generated components (do not edit, regenerate from Figma)
│   └── styles/              ← fonts.css, tailwind.css, theme.css, index.css
├── .env.example             ← Template for local secrets
├── .gitignore
├── package.json
├── vite.config.ts
├── index.html
└── CLAUDE.md                ← Guidance for AI agents working in the repo
```

★ = key directories for the backend handover.

---

## 3. View routing model

There is **no URL router** today. `App.tsx` holds a single state:

```ts
const [currentView, setCurrentView] = useState<
  | "globalMonitor"    // World map (default landing)
  | "list"             // Risk Intelligence list
  | "detail"           // Risk Intelligence detail
  | "dependencies"     // Dependencies (By Goods + By Country)
  | "criticalGoods"    // Critical Goods chapter list
  | "dependencyDetail" // Per-product supplier detail
>("globalMonitor");
```

Cross-cutting state lifted to App.tsx:

| State | Purpose |
|---|---|
| `globalSearchQuery` | Drives the Global Monitor header search overlay |
| `pendingCountry` | Pre-selects a country when navigating from a Country search result |
| `selectedRiskId`, `selectedProductId` | Detail-page subjects |
| `isCollapsed` | Sidebar collapse state (also driven by the map's maximize button) |
| `activeRisks`, `archivedRisks` | Loaded once on mount from `getActiveRisks()` / `getArchivedRisks()` |

### Why no router?

The app was bundled from Figma Make, which generates a single-screen SPA. Adding React Router is a quick win for deep-linking and back-button support — see [INTEGRATION_GUIDE §6](./INTEGRATION_GUIDE.md#6-add-real-routing).

---

## 4. Data flow

```
                            ┌──────────────────────┐
                            │  REST backend        │  ←  Future (not built yet)
                            │  (OpenAPI spec in    │
                            │   handover/)         │
                            └──────────┬───────────┘
                                       │ fetch (when wired)
                                       ▼
┌────────────────────────────────────────────────────────────┐
│  src/app/data/api/*.ts   ← Integration seam (this is the   │
│                            ONLY file backend devs need     │
│                            to edit per resource)           │
│                                                            │
│   • types.ts     — TypeScript interfaces + constraints     │
│   • client.ts    — apiFetch() with auth/timeout/errors     │
│   • risks.ts     — getActiveRisks / getArchivedRisks / …   │
│   • dependencies.ts                                        │
└──────────────────────┬─────────────────────────────────────┘
                       │ returns typed mock data today,
                       │ returns typed promises tomorrow
                       ▼
┌────────────────────────────────────────────────────────────┐
│  Components consume via plain imports                      │
│   const risks = getActiveRisks();                          │
│                                                            │
│  After backend wiring:                                     │
│   const [risks, setRisks] = useState<ActiveRisk[]>([]);    │
│   useEffect(() => {                                        │
│     getActiveRisks().then(setRisks);                       │
│   }, []);                                                  │
└────────────────────────────────────────────────────────────┘
```

The integration seam is **the only contract** between the frontend and the backend. Everything else (component rendering, translations, RTL, animations) is independent of the data source.

---

## 5. i18n + RTL — the cross-cutting concern

This is the single most pervasive thing in the codebase. Every user-visible string — including mock data — has a translation key.

- **Dictionary:** `src/app/data/translations.ts` — a flat `{ key: { en, ar } }` map, ~330 keys.
- **Hooks:** `useT()` returns `t(key, vars?)`. `useLanguage()` adds `lang` + `isRTL` + `setLang`.
- **RTL toggle:** `LanguageProvider` sets `<html dir="rtl">` on AR; flips logical CSS (`text-start`, `start-`, `insetInlineStart`) automatically.
- **Persistence:** `localStorage["moft.lang"]` (only key the app writes).
- **innerHTML markers:** Map markers built imperatively. Each translatable text node gets a `data-i18n-key` attribute; a `useEffect` keyed on `[lang]` re-translates them in place.

**Backend constraint:** Every text field returned from an API must either (a) be a known translation key, or (b) have a `*Key` companion field on the same record. See [DATA_MODELS](./DATA_MODELS.md) for which fields are translatable.

---

## 6. Auth seam

There is **no auth today** — every view is open. The hooks are in place:

- `src/app/data/api/client.ts` calls `readAuthToken()` on every request and includes it as `Authorization: Bearer <token>`.
- `readAuthToken()` reads from `sessionStorage["moft.authToken"]`. Replace with your IdP-specific token retrieval.
- `src/app/data/api/types.ts` defines `UserProfile` and `UserRole` (`viewer | analyst | officer | admin`).

See [AUTHENTICATION.md](./AUTHENTICATION.md) for the full integration recipe.

---

## 7. Map architecture

The Global Monitor map is the most complex screen. Quick mental model:

1. **`useEffect` mounts MapTiler once** with `center: [15, 15]`, `zoom: 1.3`.
2. **Markers are built imperatively** — each risk gets a `div` with custom inner HTML (label + animated halo). Markers are attached via `new maptilersdk.Marker({ element }).setLngLat([lon, lat]).addTo(map)`.
3. **A `<style>` block is injected into `<head>` once** with `@keyframes` for the breathing-halo animations. **HMR does NOT replace this block** — a hard reload is required after editing the keyframes.
4. **The MapTiler default zoom/compass controls are hidden** via CSS; we render our own custom stack (maximize/restore, globe-reset, zoom +/−) in React for design fidelity.
5. **Maximize button** toggles the parent's `isCollapsed` state via the `onToggleCollapse` prop, which collapses the left sidebar.

---

## 8. Z-index ladder (Global Monitor screen)

Document this before adding new overlays — it's easy to introduce a stacking bug.

| Layer | z-index |
|---|---|
| Map markers | 0–10 |
| GlobalMonitor header (Frame2 search bar) | **z-30** |
| Search-results overlay (`GlobalSearchResults` wrapper) | **z-20** |
| LeftSidebar | **z-40** |
| AI Assistant trigger + panel | **z-[45]** |
| Risk detail modal (`RiskPopup`) | **z-50** |

---

## 9. Performance notes

- **Bundle size:** ~1.5 MB unminified (mostly MapTiler, Recharts, jsPDF). Production minified ~450 KB gzipped. Map tiles + fonts are external requests.
- **Country map SVGs** are imported with Vite's `?raw` suffix and rendered via `dangerouslySetInnerHTML` so the outline strokes can use `currentColor` (lets the gradient theme apply). Optimised with `svgo` (~196 KB total).
- **No code splitting** is configured. The whole app loads on first paint. See [QA_REPORT §performance](./QA_REPORT.md#performance) for split recommendations.
- **No memoization of expensive renders.** Sort + filter logic in `Dependencies.tsx` uses `useMemo`, but `GlobalSearchResults` does not. Acceptable today because all lists are small (<50 items).

---

## 10. Things that look weird but are intentional

| Thing | Why |
|---|---|
| `src/imports/` has ~50 folders of Figma-exported `.tsx` | Bundled from Figma Make. Treat as the design source of truth — regenerate from Figma rather than hand-edit. |
| Font family strings like `font-['Noto_Sans:Regular',sans-serif]` | Figma export idiom. `Noto_Sans:Regular` is **not a real font family** — `src/styles/fonts.css` overrides globally with `* { font-family: 'Noto Sans', 'Noto Sans Arabic', system-ui, sans-serif !important }`. |
| 9 risks on the map vs 6 in Risk Intelligence list | Pre-existing inconsistency from two separate Figma sources. The map's risks are the "monitored chokepoints"; the list is the "active investigations". The backend should unify these or document the distinction. See [QA_REPORT B2](./QA_REPORT.md#b2-data-inconsistencies). |
| No URL routing | See §3. |
| MapTiler key has a fallback hardcoded in `GlobalMonitor.tsx` | Demo key for the bundled prototype. Replace before production — see [SECURITY_AUDIT S-01](./SECURITY_AUDIT.md). |
