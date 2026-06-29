# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

MOFT Supply Chain — a UAE-government-themed supply-chain monitoring SPA bundled from Figma Make. React 18 + Vite 6 + Tailwind v4. **All data is mocked**; there is no backend.

## Commands

- `npm i` — install deps
- `npm run dev` — start Vite dev server (port 5173). Prefer launching via `.claude/launch.json` (`name: "vite-dev"`) when using the Claude Preview MCP.
- `npm run build` — production build via Vite

There are **no tests, no linter, no `tsconfig.json`**. Vite only does TypeScript type-stripping — type errors will not fail the build. Be deliberate about types yourself.

## Architecture

### Entry & top-level state

`src/main.tsx` mounts the app inside two providers (order matters): `LanguageProvider > NotificationsProvider > App`.

`src/app/App.tsx` is the view router. There is no URL routing — view is a single `currentView` union state: `"globalMonitor" | "list" | "detail" | "dependencies" | "criticalGoods" | "dependencyDetail"`. App.tsx also owns the cross-cutting state: `globalSearchQuery`, `pendingCountry`, `selectedRiskId`, `selectedProductId`, `isCollapsed`. Sidebar navigation handlers in App.tsx clear `globalSearchQuery` on every transition.

### `src/imports/` vs `src/app/`

`src/imports/` contains ~50 folders, each a raw Figma export — JSX with inlined SVG paths (`svg-*.ts`) and rasters. **Treat as design source of truth**: don't refactor freely, don't rename, don't merge. Build app logic on top in `src/app/`. When you need to tweak how a Figma-exported component looks/behaves, the cleanest move is to edit the exported file directly (small, surgical change) and add a comment explaining the deviation from Figma.

Folder layout under `src/app/`:
- `components/` — feature components (GlobalMonitor, Dependencies, RiskListScreen, GlobalSearchResults, the two country cards, etc.)
- `pages/DependencyDetail.tsx` — the per-product detail "page"
- `context/` — `LanguageContext`, `NotificationsContext`
- `data/` — mock datasets (`countries.ts`, `criticalGoodsData.ts`, `supplierData.ts`) + the flat translations dictionary

### i18n + RTL (critical — touches everything)

`src/app/data/translations.ts` is a flat `{ key: { en, ar } }` dictionary. Every user-visible string — including mock data values (product names, country names, importer names, risk severities, news headlines) — has a key. The `useT()`/`useLanguage()` hooks from `LanguageContext` translate at render time with `{var}` substitutions.

`LanguageProvider` persists choice to `localStorage["moft.lang"]` and flips `<html lang>` + `<html dir>`. When `dir="rtl"` the entire layout mirrors, so:

- **Always** use logical CSS: `text-start`/`text-end`, `start-[Npx]`/`end-[Npx]`, `insetInlineStart` — never `text-left`, `left-`, etc.
- For HTML strings injected via `innerHTML` (e.g. MapTiler marker labels in `GlobalMonitor.tsx`), tag elements with `data-i18n-key="..."` and add a `useEffect` keyed on `[lang]` that queries those nodes and rewrites `textContent`. The marker re-translate effect is the reference pattern.
- Western Arabic numerals (1, 2, 3) are the convention even in AR — don't substitute Arabic-Indic digits.
- Translation helpers in components follow a stable pattern: `PRODUCT_KEYS`, `RISK_KEYS`, `IMPORTER_KEYS` `Record<string, TranslationKey>` maps plus `tProduct`/`tRisk`/`tCountry` functions that fall back to the raw string when no mapping exists. `Dependencies.tsx` is the reference template; mirror it when wiring a new view.

### Maps (MapTiler SDK)

`GlobalMonitor.tsx` initializes a MapTiler map once in `useEffect`, injects a `<style>` block with `@keyframes pulse-high|moderate|low` + `blink-dot` into `<head>` (once per mount). **HMR will NOT replace the injected `<style>` block** — a hard reload is required after editing the keyframes.

**Risk markers are clustered** (MapLibre's built-in `supercluster`, which ships with the MapTiler SDK). On map `load` the `risks` array is loaded as a clustered GeoJSON source (`cluster: true`, `clusterRadius: 60`, `clusterMaxZoom: 6`), plus a fully-transparent `risks-anchor` circle layer that exists only so the source gets tiled and `querySourceFeatures` returns geometry. `clusterProperties` aggregates per-severity counts (`high`, `moderate`) so a cluster badge can adopt the color of its most-severe member. An `updateMarkers()` reconciliation loop (bound to the map `render` event — the canonical MapLibre "HTML markers + clustering" recipe) diffs source features against two registries (`markers`, `markersOnScreen`) and adds/removes HTML markers:
- **Clusters** → a count-badge element from `buildClusterEl(count, severity)`; clicking calls `getClusterExpansionZoom` and eases the camera in to split the cluster.
- **Individual risks** → `buildRiskMarkerEl(risk)`. High-severity risks listed in `RISK_LABEL_KEY` render the labelled white card (anchor `bottom`); the rest render a bare pulsing dot (anchor `center`) with a hover tooltip carrying the translated risk name. Clicking opens the RiskPopup modal.

Per-severity colors + dot/halo geometry live in one `SEVERITY_STYLE` table. Marker DOM is built imperatively, so it captures translations via `tRef` (a ref holding the latest `t`); the language-change effect rebuilds all markers through `updateMarkersRef.current()` so labels/tooltips re-translate on EN/AR toggle. When adding/removing a risk, update the `risks` array (it also feeds the GlobalSearchResults index — see "Global search").

### Country assets

15 countries in `src/assets/countries/{flags,maps}/`, indexed by `src/app/data/countries.ts` (`COUNTRIES` is keyed by lowercase name, lookup via `getCountry(name)`). Country **maps** are imported with Vite's `?raw` suffix so the SVG markup can be rendered with `dangerouslySetInnerHTML` and themed via `currentColor` (lets the alt-suppliers map outlines pick up a gradient stroke). When adding a country, add both flag + map SVGs, then register in `COUNTRIES`. Use `svgo` to compress map SVGs before committing.

### Z-index conventions (Global Monitor screen)

Stacking order — keep this in mind when adding overlays:
- map markers: 0–10
- GlobalMonitor header (Frame2 search bar): **z-30**
- Search-results overlay (GlobalSearchResults wrapper): **z-20**
- LeftSidebar: **z-40**
- Risk detail modal: **z-50**

### Global search

`GlobalSearchResults.tsx` is mounted by App.tsx as an overlay above the map when `globalSearchQuery.trim()` is non-empty. It holds an in-file search index that **mirrors** subsets of `dependenciesData` (in `Dependencies.tsx`) and the `risks` array (in `GlobalMonitor.tsx`). Both source arrays have "Keep in sync" comments; if you add/remove an item there, update the index in `GlobalSearchResults.tsx`. `CRITICAL_GOODS` and `COUNTRIES` are imported directly — no duplication.

### Tailwind v4

Configured via `@tailwindcss/vite` only — there is **no `tailwind.config.js`** or theme file. Arbitrary value classes are everywhere (`font-['Noto_Sans:Regular',sans-serif]`, `bg-[#cd4747]`, `rounded-[16px]`). The bracketed font names like `Noto_Sans:Regular` come from Figma export and are **not real font families**; `src/styles/fonts.css` globally forces `Noto Sans` + `Noto Sans Arabic` via `*` selector with `!important` to make AR pages render in Noto Sans Arabic instead of the system Arabic font.

### Vite specifics

`vite.config.ts` declares a custom `figmaAssetResolver` plugin that rewrites `figma:asset/<file>` ids to `src/assets/<file>`. `assetsInclude` is set to `['**/*.svg', '**/*.csv']` to enable raw imports. The React + Tailwind plugins are both required by Figma Make tooling — keep them even if Tailwind looks unused.

### Persistent app state

`localStorage` keys in use:
- `moft.lang` — `"en"` or `"ar"`
- (NotificationsContext is in-memory only; reset on reload)

### Notes for common edits

- **Adding a new searchable view**: extend the union in App.tsx, add a navigation handler that clears `globalSearchQuery`, add the sidebar button, then thread `isCollapsed` through layout (other views use `insetInlineStart: isCollapsed ? '136px' : '263px'`).
- **Adding a translation**: add the key to `translations.ts` with both `en` and `ar`. If it appears inside `innerHTML`, also add a `data-i18n-key` attribute and add a re-translate hook (or reuse the existing one in GlobalMonitor).
- **Adding a country**: add flag + map SVGs to `src/assets/countries/`, register in `COUNTRIES` in `countries.ts`, add the `country.<name>` translation key. The country dropdown in Dependencies' "Select country" empty-state reads from a local `countries` array — also update there if it should be selectable.
- **Adding to risk popup or impact tables**: text lives in `translations.ts` under `risk.*`, `riskList.*`, `riskPopup.*`, `riskDetail.*`, `news.r{N}.n{M}` namespaces.
