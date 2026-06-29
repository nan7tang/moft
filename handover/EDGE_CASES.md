# Edge cases

Backend behavior that the frontend handles gracefully — and the cases it does NOT (so the backend can compensate).

Read alongside [DATA_MODELS](./DATA_MODELS.md) for per-field constraints.

---

## 1. Empty states

| Surface | Empty trigger | UI behavior |
|---|---|---|
| Dependencies "By Critical Goods" list | `searchQuery` filters to 0 results | `NoResultFound` illustration + "No results found" copy |
| Dependencies "By Critical Goods" grid | Same | Same |
| Dependencies "By Country" (no country picked) | `selectedCountry === ""` | "Select country from list" empty state with country dropdown |
| Dependencies "By Country" (country has no data) | `getDependenciesByCountry(country).length === 0` | Renders the page chrome with empty tables (current behavior). **Backend should return `[]`, not 404.** |
| Critical Goods | Search returns 0 | Empty state with the same `NoResultFound` graphic |
| Risk Intelligence list | Search returns 0 | Empty state |
| Global search (header) | `q` returns 0 across all 4 entity types | Empty state with "Try a different keyword" copy |

**Backend contract:** When a collection is empty, return an empty array (`[]`), never `null`, never `404`. `404` is reserved for unknown route params (e.g. `/dependencies/9999`).

---

## 2. Error states

The frontend has **no built-in error UX** today (everything is mocked, nothing can fail). After backend wiring:

| Failure | Recommended UI |
|---|---|
| Fetch fails (network / 5xx) | Inline error block in the affected card + "Retry" button + Sentry breadcrumb |
| 401 expired token | Silent refresh attempt; if it fails, redirect to login |
| 403 wrong role | Hide the offending button **before** the click; if reached anyway, toast "You don't have permission for this action" |
| 404 on detail route | Show "Risk / product not found" page with "Back to list" button |
| 422 validation failure | Inline field error using `ApiError.fields` map |
| 429 rate limit | Toast "Too many requests, wait X seconds" using the `Retry-After` header |

See [INTEGRATION_GUIDE §error-handling](./INTEGRATION_GUIDE.md#7-error-handling) for the recommended hook.

---

## 3. Loading states

Also **not yet implemented**. Today everything renders synchronously.

After backend wiring, the patterns to add:

| Surface | Skeleton |
|---|---|
| Global Monitor map | Fullscreen MapTiler loader (built into the SDK) — already shows during initial style load |
| Dependencies list | Shimmer rows (same dimensions as real rows) |
| Risk detail hero | Skeleton title + skeleton badge (greyscale) |
| Notifications panel | Skeleton list of 3 grey rows |

**Bias to "show skeleton fast, fetch fast"** — the data payloads are tiny (<10 KB each) so the perceived latency is mostly network round-trip, not parse time.

---

## 4. Text length / overflow

Field-by-field max lengths are in [DATA_MODELS §cheat sheet](./DATA_MODELS.md#field-length-cheat-sheet). Overflow behavior:

| Surface | Overflow mode | Notes |
|---|---|---|
| Map popup title | Wraps to 2 lines, then clips | Re-render is cheap; no performance impact |
| Search result row | Single-line truncation with `…` | `truncate` Tailwind class |
| Risk detail hero title | Wraps freely | Page can grow vertically |
| Dismiss reason in archive table | Wraps | Long reasons compress the row height — acceptable today; consider line-clamp-3 if it becomes ugly |
| Country names | Truncate in tight columns | All known country names fit; protected by `@maxLength 30` |
| Alternate suppliers ("Pakistan (31%), Thailand (8%), …") | Truncate with `…` if it overflows the column | Backend should keep this string ≤200 chars |

**Risk:** If the backend returns text longer than the documented `maxLength`, the UI doesn't break, but specific edge cases (e.g. a country name >40 chars in the search result row) may look misaligned. Recommendation: backend validates lengths against the OpenAPI spec.

---

## 5. RTL / Arabic edge cases

The frontend has end-to-end RTL support. Watch for these in backend data:

| Case | Behavior |
|---|---|
| Translation key missing in dictionary | Frontend renders the raw key (e.g. `"riskList.foo"`) as the label. Always catch these in QA. |
| Mixed-direction text (e.g. an English HS code inside an Arabic sentence) | Renders correctly thanks to Unicode bidi algorithm; no special markup needed. |
| Arabic numerals (٠ ١ ٢) vs Western (0 1 2) | UI uses **Western numerals everywhere**, even in AR mode. Backend should also send Western numerals — this is the standard for modern Arabic UIs. |
| Arabic comma `،` | The `tAltSuppliers` and `tImporters` helpers use the locale-appropriate separator automatically — backend can send English commas. |
| User-typed text in search (Arabic) | Search index matches translated labels, so users find risks by their AR name (e.g. `أرز` returns Rice). Backend `/search` must also match across translated labels. |

---

## 6. Numbers / dates / units

| Field | Format | Locale-sensitive? |
|---|---|---|
| Percentages (`dependency`, `priceChange`) | Integer or fixed-1 number + literal `%` | No |
| Money (`intlPrice`) | Raw number, displayed as `$528 /MT` | Currency symbol hardcoded; backend in USD/MT only |
| UAE consumption (`27.5K`, `46.0K`) | **Pre-formatted string** | Backend formats; frontend does not parse |
| Dates (`firstDetected`, `lastUpdated`, `dismissedOn`) | `"YYYY-MM-DD HH:MM UTC"` string | No; rendered verbatim |
| Coordinates | `[lon, lat]` array of numbers | No |
| HS codes | String, leading-zero preserved | No |

**If you switch to ISO-8601 dates:** the frontend must format them client-side. Today the strings are rendered verbatim, which is why they include the `UTC` suffix in the data.

---

## 7. Data uniqueness

Frontend assumes:

- Risk IDs are unique **across both** `active` and `archived` collections.
- Dependency IDs are unique within `/dependencies`.
- Notification IDs are unique per-user.
- Country names match the keys in [`countries.ts`](../src/app/data/countries.ts) (case-insensitive). New countries require adding flag + map SVGs.

If the backend introduces collisions, multiple risks will share the same React key → expect rendering glitches and lost state in lists.

---

## 8. Pagination / scale

Today every list is small (<50 items). When data grows:

- **Risks list** (`/risks?status=active`): page beyond ~50.
- **Dependencies**: page beyond ~50 (current 8 is fine for years).
- **Notifications**: page beyond ~50; consider a "Load older" button at the bottom.
- **Search**: hard-cap at 50 results total (5 per group default) to keep the overlay readable.

The frontend has zero pagination UI built. Add when the backend can no longer return everything in one shot.

---

## 9. Optimistic UI

The Risk Intelligence dismiss/restore actions update local state immediately, **before** the backend confirms. If the backend rejects the action (`422`, `403`, network), the UI is now out of sync.

**Recommended pattern post-integration:**
1. Snapshot state before the optimistic update.
2. On error: revert + show toast.
3. On success: replace the optimistic record with the server's authoritative response.

Look at the `handleRestoreRisk` function in [`App.tsx`](../src/app/App.tsx) for the current behavior and the comment block describing the recommended pattern.

---

## 10. Concurrency / multi-tab

- **Notifications**: marking as read in one tab does not propagate to other open tabs. Acceptable today; future improvement is a `BroadcastChannel` or WebSocket push.
- **Language**: changing language in one tab updates `localStorage` but does NOT live-update other open tabs until they reload. (Acceptable; users rarely have two tabs in different languages.)
- **Auth**: when refresh happens in tab A, tab B still has the old token. The 401 handler should silently refresh and retry.

---

## 11. Browser support

| Browser | Status |
|---|---|
| Chrome / Edge ≥ 110 | Fully supported |
| Firefox ≥ 110 | Fully supported |
| Safari ≥ 16 | Fully supported (test MapTiler tile loading; some older Safaris have webgl quirks) |
| Internet Explorer | **Not supported.** Tailwind v4 + Vite + ES2020 target rule out IE entirely. |
| iOS Safari ≥ 16 | Should work; not regularly tested. AI chat panel sizing may need a mobile breakpoint. |
| Android Chrome | Same |

The platform is designed for desktop dashboards (>= 1280px wide). Mobile is out of scope today — sidebar + map controls assume hover; AI chat overlay assumes desktop space.

---

## 12. Accessibility considerations

Documented in [QA_REPORT §a11y](./QA_REPORT.md#accessibility). Edge cases backend should know about:

- **Color-only state indicators** (severity badges, inventory pills) also include the textual label. Backend just needs to send the label.
- **Empty avatars** for users without a photo: frontend generates initials from `displayName`. Backend should never send a broken `avatarUrl` — set it to `null` instead.
