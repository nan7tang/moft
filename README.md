# MOFT Supply Chain

UAE Ministry of Foreign Trade — supply-chain monitoring SPA. React 18 + Vite + Tailwind v4 + MapTiler. All data currently mocked.

> 🚚 **Are you here for the backend handover?** Start at [`handover/README.md`](./handover/README.md). The whole `handover/` folder is your map.

---

## Quick start

```bash
# 1. Install
npm install

# 2. Configure (optional — defaults work for local dev)
cp .env.example .env.local
# Edit .env.local — at minimum set VITE_MAPTILER_KEY if you have a key.
# Without it, the bundled demo key is used (rate-limited; replace before prod).

# 3. Run
npm run dev            # → http://localhost:5173

# 4. Production build
npm run build          # → ./dist
```

---

## Repo map

```
.
├── handover/                ← Backend integration docs (start here if integrating)
├── src/
│   ├── main.tsx             ← Entry: providers + <App />
│   ├── app/
│   │   ├── App.tsx          ← View router + cross-cutting state
│   │   ├── components/      ← Feature components
│   │   ├── pages/           ← Per-product detail page
│   │   ├── context/         ← LanguageContext, NotificationsContext
│   │   └── data/
│   │       ├── api/         ★ Integration seam — accessors, types, client
│   │       ├── mock/        ★ JSON mocks matching API response shape
│   │       └── *.ts         (countries, criticalGoods, supplier, translations)
│   ├── assets/              Country flags + maps, AI orb, category icons
│   ├── imports/             Figma-generated components (regenerate; don't edit)
│   └── styles/              Tailwind + custom CSS
├── .env.example             Template for local secrets
├── .gitignore
├── package.json
├── vite.config.ts
├── index.html
├── README.md                (this file)
└── CLAUDE.md                Guidance for AI agents working in the repo
```

---

## Key documents

| Doc | What it covers |
|---|---|
| [`handover/README.md`](./handover/README.md) | Backend handover entry point |
| [`handover/ARCHITECTURE.md`](./handover/ARCHITECTURE.md) | System overview, conventions, data flow |
| [`handover/API_SPECIFICATION.yaml`](./handover/API_SPECIFICATION.yaml) | OpenAPI 3.0 backend contract |
| [`handover/INTEGRATION_GUIDE.md`](./handover/INTEGRATION_GUIDE.md) | File-by-file mock → real-API swap |
| [`handover/SECURITY_AUDIT.md`](./handover/SECURITY_AUDIT.md) | Pre-production security checklist |
| [`CLAUDE.md`](./CLAUDE.md) | AI-agent guidance (use this with Claude Code, Cursor, etc.) |

---

## Tech stack at a glance

- **Vite 6** — dev server + build. Type-strips TypeScript; no `tsconfig.json`.
- **React 18** — strict mode, function components only.
- **Tailwind CSS v4** via `@tailwindcss/vite` — arbitrary value classes (`bg-[#cd4747]`).
- **MapTiler SDK 4** — global map. Key in `.env`.
- **lucide-react** + bespoke Figma SVGs — icons.
- **jsPDF + jspdf-autotable** — risk report PDF export.

---

## What's working

Open the app and try:
- Click any pin on the world map → risk popup → "View Goods" → Risk Intelligence detail (hero color matches severity).
- Switch language via the globe icon (top right) → entire UI flips to AR + RTL.
- Use the search bar (top right of map) → cross-entity search results.
- Click the AI Assistant orb (bottom right) → mock chat panel.
- Click "Dependencies" in the sidebar → browse commodities → click a row → per-product detail with alternative supplier countries (flags + map outlines).
- Toggle the maximize icon in the map controls → sidebar collapses for a fullscreen map.
- Toggle the globe icon in the map controls → resets to world view + collapses sidebar.

---

## What's not yet wired

| Capability | Status | Plan |
|---|---|---|
| Real backend | Mocked | [`handover/INTEGRATION_GUIDE.md`](./handover/INTEGRATION_GUIDE.md) |
| Auth (sign-in, RBAC) | Stub | [`handover/AUTHENTICATION.md`](./handover/AUTHENTICATION.md) |
| URL routing | None | [`handover/INTEGRATION_GUIDE.md §6`](./handover/INTEGRATION_GUIDE.md#6-add-real-routing) |
| Loading / error states | None | [`handover/EDGE_CASES.md §2-3`](./handover/EDGE_CASES.md) |
| Tests | None | [`handover/INTEGRATION_GUIDE.md §10`](./handover/INTEGRATION_GUIDE.md#10-tests-currently-zero) |

---

## License

Proprietary — MOFT internal use only.

---

## Attribution

UI primitives from [shadcn/ui](https://ui.shadcn.com) (MIT). Map tiles by [MapTiler](https://www.maptiler.com). Fonts: [Noto Sans / Noto Sans Arabic](https://fonts.google.com/noto) (OFL).
