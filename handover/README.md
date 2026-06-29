# MOFT Supply Chain — backend handover package

> **TL;DR for the recipient:** This is a fully-functional React SPA with mock data. Everything the UI renders is documented as an API contract. Pick up where the frontend ends (the integration seam at `src/app/data/api/`) and you can ship a real backend without touching component code.

**Snapshot date:** 19 May 2026
**Status:** Frontend complete, mocked. Backend not built. Auth not wired. Ready for parallel backend development.

---

## What's in this package

```
handover/
├── README.md                ← You are here. Open this first.
├── ARCHITECTURE.md          ← System overview: stack, data flow, conventions
├── DATA_MODELS.md           ← Every TypeScript interface + field constraints
├── API_SPECIFICATION.yaml   ← OpenAPI 3.0 — machine-readable contract
├── API_REFERENCE.md         ← Same endpoints, human-readable, with examples
├── AUTHENTICATION.md        ← JWT + RBAC baseline + UAE PASS / Entra recipes
├── SECURITY_AUDIT.md        ← Cyber-security review with severity-graded fixes
├── EDGE_CASES.md            ← Empty / error / overflow / RTL / browser cases
├── INTEGRATION_GUIDE.md     ← File-by-file: how to swap mock for real APIs
└── QA_REPORT.md             ← Known issues, browser support, perf, a11y
```

The codebase itself is in the **parent directory** — the handover docs sit alongside the source so they ship together in one zip.

---

## How to read this package

| Your role | Read in this order |
|---|---|
| **Backend engineer** building the API | `ARCHITECTURE.md` → `DATA_MODELS.md` → `API_SPECIFICATION.yaml` → `API_REFERENCE.md` → `EDGE_CASES.md` |
| **Frontend engineer** integrating real APIs | `ARCHITECTURE.md` → `INTEGRATION_GUIDE.md` → `AUTHENTICATION.md` → `QA_REPORT.md` |
| **Security reviewer** | `SECURITY_AUDIT.md` → `AUTHENTICATION.md` → `INTEGRATION_GUIDE.md` |
| **QA engineer** | `QA_REPORT.md` → `EDGE_CASES.md` → `DATA_MODELS.md` |
| **AI coding agent** | `INTEGRATION_GUIDE.md` (esp. §appendix) + `API_SPECIFICATION.yaml` are designed for machine consumption |
| **Product / decision-maker** | `README.md` (this doc) + `QA_REPORT.md` summary |

---

## Running the app locally

```bash
# 1. Install
npm install

# 2. Configure
cp .env.example .env.local
# Edit .env.local — at minimum set VITE_MAPTILER_KEY (free tier at https://cloud.maptiler.com)

# 3. Run
npm run dev            # → http://localhost:5173

# 4. Build for production
npm run build          # → ./dist
```

No backend is needed. The app loads with mock data and works end-to-end.

---

## Where the integration seam is

Everything backend-related funnels through one directory:

```
src/app/data/
├── api/
│   ├── types.ts           ← TypeScript interfaces matching every API response
│   ├── client.ts          ← apiFetch() — Bearer auth, timeout, error envelope
│   ├── risks.ts           ← getActiveRisks / getArchivedRisks / dismissRisk / …
│   └── dependencies.ts    ← getDependencies / getDependenciesByCountry
└── mock/
    ├── active-risks.json
    ├── archived-risks.json
    ├── dependencies.json
    └── global-risks.json
```

Each accessor function in `api/` has a **`@integration` JSDoc block** with the exact one-line snippet to swap from mock to real fetch. Components don't call `fetch` directly — they only call accessors. **One file changes per resource.**

---

## What's been done in this handover snapshot

✅ **Functional UI.** Every view renders with realistic mock data.
✅ **Full Arabic + RTL support.** Every string has both EN/AR. Layout flips on language toggle.
✅ **Global search.** Cross-entity search wired up (mocks indexed in-component; backend should provide `/search`).
✅ **AI chat widget.** Floating orb trigger; panel matches the Figma design. Mock placeholder reply.
✅ **Notifications.** Global header bell with badge count + dropdown panel.
✅ **Per-product detail page.** Alternative supplier countries with flags + map outlines.
✅ **Risk Intelligence detail page.** Hero/badge dynamically colored by severity (High → red, Moderate → orange, Low → gold).
✅ **Map controls.** Custom maximize / globe / zoom +/− stack matching the Figma design.
✅ **Code organization for backend swap.** Mock data extracted to JSON; accessor layer in place.
✅ **`.env.example`, `.env`, `.gitignore`.** API key moved out of source.
✅ **All 10 handover docs (this folder).**

## What's not done (and where it's tracked)

⏸ **Real backend.** Mock today; integration guide lays out the path.
⏸ **Real auth.** Stub in place; see `AUTHENTICATION.md`.
⏸ **URL routing.** Single `currentView` state; React Router migration is a half-day task. See `INTEGRATION_GUIDE §6`.
⏸ **Loading / error states.** Required after backend wiring. See `EDGE_CASES §2-3`.
⏸ **Tests.** None today. Recommended stack in `INTEGRATION_GUIDE §10`.
⏸ **CSP header.** See `SECURITY_AUDIT S-02`.
⏸ **Risk-data dual-collection unification.** Documented in `QA_REPORT Q-01`.

None of these block the handover. They are the explicit "what's next" for the receiving team.

---

## Key conventions (read once, save time later)

| | |
|---|---|
| **Stack** | Vite 6 + React 18 + TypeScript (type-stripped only — no `tsconfig.json`) + Tailwind v4 + MapTiler |
| **Routing** | None. Single `currentView` union state in `App.tsx`. Add React Router post-handover. |
| **State** | Local `useState` per view + 2 React contexts (`LanguageContext`, `NotificationsContext`). No Redux/Zustand. |
| **Styling** | Tailwind arbitrary values everywhere (`bg-[#cd4747]`). No `tailwind.config.js`. |
| **i18n** | Flat `translations.ts` dictionary, `useT()` hook, `data-i18n-key` for imperatively-injected text |
| **RTL** | Logical CSS (`text-start`, `start-`, `insetInlineStart`) flips automatically on AR toggle |
| **Mock data** | JSON files in `src/app/data/mock/` — shape matches future API responses 1:1 |
| **Z-index ladder** | Map 0–10 / search overlay 20 / map header 30 / sidebar 40 / AI chat 45 / modal 50 |
| **API key** | Read from `import.meta.env.VITE_MAPTILER_KEY`; demo fallback in code (REMOVE before prod) |

---

## Estimated effort to production

| Phase | Effort |
|---|---|
| **Backend implementation** (per the OpenAPI spec) | 2–4 weeks, 1–2 devs |
| **Frontend integration** (per `INTEGRATION_GUIDE`) | 1–2 days |
| **Auth wiring** | 1–2 days (depending on IdP) |
| **Add React Router** | 0.5 day |
| **Add loading / error states** | 1 day |
| **Add tests** | 2–4 days (initial coverage) |
| **Pre-prod security checklist** (`SECURITY_AUDIT`) | 1 day |
| **Pentest + fixes** | 1 week (external) |

Realistic ship-to-staging timeline with a parallel team: **~4 weeks.**

---

## Contact

Questions about the codebase or this handover package: see git history for the author(s).
Questions about Figma source: the original is at [Figma → MOFT-Supply-chain](https://www.figma.com/design/MdU52Dnow3Q2uDsK1HM8uQ/MOFT-Supply-chain) (access required).
