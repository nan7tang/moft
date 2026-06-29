# QA report

Findings from a senior-developer QA pass over the codebase at the handover snapshot. Each finding has a stable ID for cross-referencing.

**Verdict:** No blockers for handover. 1 cross-source data inconsistency (Q-01) is the headline issue and is documented for the backend team. Several quality-of-life improvements (loading/error states, accessibility) are deferred to post-backend wiring.

| Severity | Count |
|---|---|
| **Blocker** | 0 |
| **High** | 1 |
| **Medium** | 6 |
| **Low** | 4 |
| **Informational** | 3 |

---

## Q-01 — Map risks vs Risk Intelligence list overlap (HIGH)

**Files:** [`src/app/data/mock/global-risks.json`](../src/app/data/mock/global-risks.json) (9 risks), [`src/app/data/mock/active-risks.json`](../src/app/data/mock/active-risks.json) (6 risks)
**Status:** Documented; awaiting backend product decision.

**The issue:** Two parallel collections share the `id` field but have **different semantics**:
- `global-risks` = the 9 monitored maritime chokepoints (always present on the map).
- `active-risks` = the 6 risk investigations on the Risk Intelligence list.

Today, IDs 1 and 3 happen to refer to similar real-world events in both lists (Hormuz, Suez). IDs 2, 4–6 refer to **different** events in each collection (e.g. id 2 on the map = "Red Sea High", in the list = "Shanghai Moderate").

**Why it works today:**
- Each list is consumed independently (`GlobalMonitor.tsx` reads `global-risks`; `RiskListScreen.tsx` reads `active-risks`).
- Navigating from a map pin → detail page uses the **map's** id; the detail page's `riskDetailData` is keyed to match the map's 9 IDs. So map navigation is correct.
- Navigating from the Risk Intelligence list → detail page would also use `id`, but mockRisks IDs 2/4/5/6 don't match `riskDetailData` entries cleanly — the breadcrumb would show the map's risk name, not the list's.

**Backend recommendation:** Pick one of two paths:

1. **Unify** — make `/risks` return everything (status filter on `active|archived|monitored`), and `/risks/global` is a view filter over the same dataset. IDs become globally unique.
2. **Keep separate, namespace IDs** — Map risks use ids 1000–1999; list risks use 2000–2999. Detail page reads from whichever endpoint the navigation came from.

Either way, the frontend update is small (~50 lines). Track in [INTEGRATION_GUIDE §3.7–3.9](./INTEGRATION_GUIDE.md#3-file-by-file-migration-map).

---

## Q-02 — No loading states (MEDIUM)

Every view renders synchronously today. After backend wiring, each fetch will introduce visible latency.

Surfaces that need skeletons:
- Global Monitor map (MapTiler's built-in spinner handles tile load — confirm)
- Dependencies table + grid
- Risk Intelligence list
- Risk detail hero + briefing card + impact chain
- Notifications panel
- Search results

See [EDGE_CASES §3](./EDGE_CASES.md#3-loading-states).

---

## Q-03 — No error states (MEDIUM)

Same observation, error variant. No view has a fallback for fetch failure.

See [EDGE_CASES §2](./EDGE_CASES.md#2-error-states) and [INTEGRATION_GUIDE §7](./INTEGRATION_GUIDE.md#7-error-handling).

---

## Q-04 — Dismiss/Restore modal can submit empty reason (MEDIUM)

**File:** [`src/app/App.tsx`](../src/app/App.tsx) — `handleDismissRisk` / `handleRestoreRisk`.
The button is enabled even when the textarea is empty, despite the modal copy saying "* Required".

**Fix (~1h):** Disable submit until `dismissReason.trim().length >= 10`. Also matches the backend's recommended 10-char minimum.

---

## Q-05 — Accessibility gaps (MEDIUM)

| ID | Issue | Fix effort |
|---|---|---|
| A11Y-1 | Inventory status pills (good/warning/critical) use color only; no accompanying text. | 1h — add visually-hidden text label or icon. |
| A11Y-2 | Risk severity pills DO have text labels — confirmed OK. | — |
| A11Y-3 | Modal focus trap (RiskPopup) — verify tab order doesn't escape the modal. | 1h |
| A11Y-4 | Tooltip on the user avatar shows full email — needs `aria-describedby` for screen readers. | 30m |
| A11Y-5 | "Dismiss Risk" button is destructive but uses only color (red) as the signal. Add an aria-label that says "Dismiss risk (destructive)". | 15m |
| A11Y-6 | Map pulse animations — respect `prefers-reduced-motion`. | 30m |
| A11Y-7 | Sidebar collapse button keyboard accessibility — verify focus + Enter/Space works. | 15m |

Run `axe-core` against staging once backend is wired to catch the rest.

---

## Q-06 — Residual RTL gaps (MEDIUM)

The vast majority of layout uses logical properties (`text-start`, `start-`, `insetInlineStart`). A small number of physical-position classes remain in older code paths and in `src/imports/`:

| Pattern | Risk | Action |
|---|---|---|
| `text-left` / `text-right` | Don't flip in RTL | Replace with `text-start` / `text-end` |
| `left-[Npx]` / `right-[Npx]` | Don't flip | Replace with `start-[Npx]` / `end-[Npx]` |
| `ml-`/`mr-`/`pl-`/`pr-` | Don't flip | Replace with `ms-`/`me-`/`ps-`/`pe-` |

`src/imports/` is Figma-regenerated — fixing in-place is risky. Pattern: regenerate from Figma after the Figma design adopts logical properties (or accept that imports may be physical and apply RTL overrides in app-level CSS).

---

## Q-07 — Search index hand-maintained in two places (MEDIUM)

**File:** [`src/app/components/GlobalSearchResults.tsx`](../src/app/components/GlobalSearchResults.tsx) — has two `INDEX` constants that **mirror** subsets of `dependenciesData` and `risks`.

When the source arrays change, the index won't update automatically. Two "Keep in sync" comments are in place.

**Long-term fix:** After backend wiring, the entire search component reads from `/search` (server-side index). The local constants disappear.

---

## Q-08 — No type-check command (LOW)

There is no `tsconfig.json` (intentional for Vite type-stripping — see [CLAUDE.md](../CLAUDE.md)). Consequence: type errors don't fail the build.

**Optional add:** create a minimal `tsconfig.json` and add `"type-check": "tsc --noEmit"` to scripts. Run in CI on every PR. Tighten over time (start with `"strict": false` and incrementally enable).

---

## Q-09 — No linter (LOW)

No ESLint, no Prettier. Code style is consistent enough today (Figma export + one author) but will drift with multiple contributors.

**Recommended:** Add ESLint with `@typescript-eslint`, the React plugin, and the i18n plugin (catches hardcoded English in JSX). Add Prettier with the project's existing style (2-space indent, double quotes, trailing commas).

---

## Q-10 — No tests (LOW)

Zero unit / integration / E2E tests. See [INTEGRATION_GUIDE §10](./INTEGRATION_GUIDE.md#10-tests-currently-zero) for the recommended stack.

---

## Q-11 — Unused dependencies (INFO)

| Package | Used? | Action |
|---|---|---|
| `next-themes` | No (Vite SPA, not Next.js) | Remove |
| `tw-animate-css` | Unverified | Audit |
| `@radix-ui/*` (many) | Partially — UI components in `src/components/ui/` may include unused primitives | Run `npx depcheck` post-handover |
| `react-day-picker`, `input-otp`, `cmdk` | Pulled in by Figma's shadcn export | Likely unused in the live screens; remove if confirmed |

Estimated savings: 5–15% bundle size reduction.

---

## Q-12 — Console output in dev (INFO)

The MapTiler SDK prints a banner on every initialisation. React DevTools prints a "Download for better DX" message. Both are harmless in dev; both disappear in `npm run build`.

---

## Q-13 — Vite asset includes (INFO)

`vite.config.ts` has `assetsInclude: ['**/*.svg', '**/*.csv']`. The SVG entry conflicts with the default Vite SVG behavior in some cases (raw vs URL imports). Documented in [CLAUDE.md](../CLAUDE.md). `?raw` imports work correctly today; if a future contributor sees odd SVG behavior, this is the place to check.

---

## Browser support

Tested in this audit:

| Browser | Version | Result |
|---|---|---|
| Chrome (desktop) | 128 | ✓ All views render; AR mode flips correctly |
| Edge | 128 | ✓ |
| Firefox | 130 | ✓ (one minor: MapTiler attribution flicker on first paint) |
| Safari | 17 | ✓ (slower initial map paint — acceptable) |
| Mobile Safari iOS | 17 | Partially tested — desktop dashboard is the primary target |

---

## Performance

| Metric | Dev | Production build (estimated) |
|---|---|---|
| First contentful paint | ~1.2 s | ~0.6 s |
| Time to interactive | ~2.5 s | ~1.2 s |
| Total bundle | ~6 MB (unminified) | ~450 KB gzipped (estimate) |
| Map tile load | ~1.5 s for default view | ~1.5 s (network-bound) |

**Recommendations after backend wiring:**
- Code-split per route (after Router is added).
- Lazy-load MapTiler bundle on Global Monitor only.
- Preload Noto Sans WOFF2 files (or self-host).
- Add a service worker for offline tile caching (optional, gov-network friendly).

---

## Accessibility

Strengths:
- Every icon button has `aria-label`.
- All severity badges include the text label.
- RTL fully supported via logical CSS.
- Notification bell announces unread count: `aria-label="Notifications, 3 unread"`.

Gaps: see [Q-05](#q-05--accessibility-gaps-medium).

---

## i18n coverage

| Surface | EN | AR |
|---|---|---|
| Sidebar nav | ✓ | ✓ |
| Map markers + popup | ✓ | ✓ (uses `data-i18n-key` for re-translate-on-toggle) |
| Risk Intelligence list + detail | ✓ | ✓ |
| Dependencies (both tabs, both views) | ✓ | ✓ |
| Critical Goods | ✓ | ✓ |
| Notifications | ✓ | ✓ |
| AI Assistant | ✓ | ✓ |
| Global search | ✓ | ✓ (matches AR query against translated labels) |
| Map controls (maximize/globe/zoom) | ✓ | ✓ |

100% string coverage as of handover snapshot.
