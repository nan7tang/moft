# Data models

**Audience:** Backend engineers building the API.
**Authoritative TypeScript source:** [`src/app/data/api/types.ts`](../src/app/data/api/types.ts) — every interface below mirrors that file.
**Companion:** [`API_SPECIFICATION.yaml`](./API_SPECIFICATION.yaml) — the OpenAPI 3.0 schema with the same constraints.

Every field that the frontend renders has a documented constraint here. If you ship a longer string than the documented `maxLength`, the UI will either truncate, ellipsize, or wrap — see [EDGE_CASES](./EDGE_CASES.md) for the per-field behavior.

---

## Conventions

| Notation | Meaning |
|---|---|
| `?` after a field name | Optional / nullable |
| `@maxLength N` | Frontend truncates or visually clips beyond N characters |
| `@enum a \| b \| c` | Only these literal values are accepted |
| `@pattern <regex>` | Backend should validate before persisting |
| `@minimum`, `@maximum` | Numeric range for `number` fields |
| `*Key` field | Translation-dictionary key — frontend uses it instead of the raw English field if present |

---

## 1. Shared scalars

| Type | Definition | Notes |
|---|---|---|
| `RiskSeverity` | `"Low" \| "Moderate" \| "High" \| "Critical"` | Drives color in the UI (gold / orange / red / red). |
| `InventoryStatus` | `"good" \| "warning" \| "critical"` | Drives the inventory pill color on the Dependencies list. |
| `UtcDisplayTime` | `string`, format `YYYY-MM-DD HH:MM UTC` | Display-ready (note the SPACE, not `T`, and explicit `UTC` suffix). Backend can return ISO-8601 if the frontend is updated to format it client-side. |
| `LonLat` | `[longitude: number, latitude: number]` | WGS84 datum. Longitude first (GeoJSON convention). |
| `NotificationType` | `"high" \| "warning" \| "info" \| "report"` | Drives the notification dot color in the bell panel. |
| `UserRole` | `"viewer" \| "analyst" \| "officer" \| "admin"` | RBAC matrix in [AUTHENTICATION §RBAC](./AUTHENTICATION.md#rbac-matrix). |
| `TranslationKey` | Union of ~330 string literals from `translations.ts` | Backend must use one of these for any `*Key` field. |

---

## 2. `GlobalRisk` — Global Monitor map pin

Mock source: [`src/app/data/mock/global-risks.json`](../src/app/data/mock/global-risks.json)
Endpoint: `GET /risks/global`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | `@minimum 1 @maximum 9999` | Stable risk identifier. Route parameter for the detail page. |
| `title` | `string` | `@maxLength 80` | English fallback title. Truncated in the map popup if longer. |
| `subtitle` | `string` | `@maxLength 120` | Secondary line under the title. |
| `severity` | `RiskSeverity` | required | Drives marker color + halo size + detail-page hero tint. |
| `coordinates` | `LonLat` | required | Markers outside the visible viewport are clipped. |

**Edge cases handled by the UI:**
- Long titles → wraps to a second line in the marker label; truncates with ellipsis in the search results.
- Missing severity → falls back to `"High"` (worst-case visual, so nothing is silently understated).
- Coordinates over the antimeridian → MapTiler handles wrapping; no special backend action needed.

---

## 3. `ActiveRisk` — Risk Intelligence list, "All" tab

Mock source: [`src/app/data/mock/active-risks.json`](../src/app/data/mock/active-risks.json)
Endpoint: `GET /risks?status=active`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | `@minimum 1`, disjoint from `ArchivedRisk.id` | Stable identifier. The backend should keep IDs unique across both tables. |
| `title` | `string` | `@maxLength 80` | English fallback. Used when `titleKey` is missing. |
| `titleKey?` | `TranslationKey` | optional | Preferred — frontend looks this up in the dictionary first. |
| `subtitle` | `string` | `@maxLength 120` | English fallback. |
| `subtitleKey?` | `TranslationKey` | optional | |
| `category` | `string` | `@maxLength 30` | English fallback. |
| `categoryKey?` | `TranslationKey` | optional | |
| `severity` | `RiskSeverity` | required | |
| `firstDetected` | `UtcDisplayTime` | required | |
| `lastUpdated` | `UtcDisplayTime` | required | Should be `>= firstDetected`. |
| `goods` | `number` | `@minimum 0` | Count of affected commodities. |
| `impactedGoods` | `number` | `@minimum 0` | Currently a mirror of `goods`. Backend can collapse to one field once consumers are updated. |

---

## 4. `ArchivedRisk` — Risk Intelligence list, "Archive" tab

Mock source: [`src/app/data/mock/archived-risks.json`](../src/app/data/mock/archived-risks.json)
Endpoint: `GET /risks?status=archived`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | unique across active+archived | |
| `title`, `titleKey?` | as above | | |
| `category`, `categoryKey?` | as above | | |
| `severity` | `RiskSeverity` | required | |
| `dismissedBy` | `string` | `@maxLength 50` | User display name. The backend should also return the user id in a separate field for auditing — not currently consumed by the UI. |
| `dismissedOn` | `UtcDisplayTime` | required | |
| `reason` | `string` | `@maxLength 500` | Free-text reason captured in the dismiss modal. UI wraps freely. |
| `impactedGoods` | `number` | `@minimum 0` | |

---

## 5. `Dependency` — commodity import dependency row

Mock source: [`src/app/data/mock/dependencies.json`](../src/app/data/mock/dependencies.json)
Endpoint: `GET /dependencies`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | `@minimum 1` | |
| `product` | `string` | `@maxLength 30` | English fallback. UI may also resolve a `PRODUCT_KEYS` map in `Dependencies.tsx`. |
| `category` | `string` | `@enum` (see below) | Slug used to look up the category icon. |
| `hsCode` | `string` | `@pattern ^[0-9]{4,6}$` | **String, not number** — preserves leading zeros (e.g. `"0207"` for poultry). |
| `primarySupplier` | `string` | `@maxLength 30` | Must match an English country name in `COUNTRIES` (`src/app/data/countries.ts`). |
| `dependency` | `number` | `@minimum 0 @maximum 100` | Percentage. |
| `alternateSuppliers` | `string` | `@maxLength 200` | Comma-separated `"Country (NN%), Country (NN%), …"`. Parsed and re-translated client-side. **Backend may switch to a structured array** — when it does, update the `tAltSuppliers` helper in `Dependencies.tsx`. |
| `uaeConsumption` | `string` | `@maxLength 10` | Pre-formatted headline (e.g. `"27.5K"`, `"1.2M"`). Frontend does not currently format raw numbers. |
| `uaeConsumptionUnit` | `string` | `@maxLength 6` | Currently always `"MT"`. |
| `inventory` | `number` | `@minimum 0 @maximum 999` | Days of cover. |
| `inventoryStatus` | `InventoryStatus` | required | |
| `topImporters` | `string` | `@maxLength 120` | Comma-separated importer brand names. Each part is looked up in `IMPORTER_KEYS` for translation. |
| `intlPrice` | `number` | `@minimum 0` | USD per MT. |
| `priceChange` | `number` | `@minimum -100 @maximum 100` | MoM percentage. |
| `mfgCapacity` | `string` | `@maxLength 10` | `"—"` when not applicable. |
| `risk` | `RiskSeverity` | required | Drives the risk pill color. |

### Allowed `category` values

Map to icon files in `src/assets/category-icons/`:

```
Beverages, Cleaning__Disinfection, Condiments, Dairy___Eggs, Edible_Oils,
Fruits, Grain, Industrial_Chemicals, Lab, Legumes, Live_Animals,
Measurement_Instruments, Meat, Medical, Medical_Devices, Medical_Supplies,
Prepared_Food, SWEETENERS, Sea_Food, Spices, Vegetables, Water___Infrastruction
```

If the backend introduces a new category, add the corresponding SVG to `src/assets/category-icons/` and the mapping in `Dependencies.tsx`'s `categoryIcons` const. Otherwise the row falls back to the Grain icon.

---

## 6. `CriticalGood` — HS chapter chip row

Source: [`src/app/data/criticalGoodsData.ts`](../src/app/data/criticalGoodsData.ts) (not yet migrated to JSON — see [INTEGRATION_GUIDE §migration map](./INTEGRATION_GUIDE.md))
Endpoint: `GET /critical-goods`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | `@minimum 1` | |
| `chapterKey` | `TranslationKey` | required | E.g. `"chapter.01-05"`. **No English fallback** — every chip must have both EN/AR translations in the dictionary. |
| `categoryKey` | `TranslationKey` | required | |
| `commoditiesKey` | `TranslationKey` | required | Translates to a comma-separated commodities list. |

**Edge case:** if the backend wants to expose chapters not in the current dictionary, the frontend must add the keys to `translations.ts` FIRST or the chips will display the raw key string.

---

## 7. `AppNotification` — header bell panel item

Source: [`src/app/context/NotificationsContext.tsx`](../src/app/context/NotificationsContext.tsx)
Endpoint: `GET /notifications` (paginated when scaled — see `Paginated<T>`)

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `number` | `@minimum 1` | |
| `type` | `NotificationType` | required | `"high" \| "warning" \| "info" \| "report"` — drives the colored dot. |
| `titleKey` | `TranslationKey` | required | |
| `descKey` | `TranslationKey` | required | When rendered, body wraps freely; aim for under ~150 chars to fit two lines in the panel. |
| `timeKey` | `TranslationKey` | required | Frontend currently uses static keys like `"notif.ex.time.12m"`. **Recommended backend change:** return raw ISO timestamps + format relative time client-side. |
| `read` | `boolean` | required | Drives the red badge count + per-row read indicator. |

---

## 8. `UserProfile` — `/auth/me`

| Field | Type | Constraint | Description |
|---|---|---|---|
| `id` | `string` | UUID | |
| `displayName` | `string` | `@maxLength 60` | Shown on the avatar hover tooltip. |
| `email` | `string` | `@pattern ^[^@\s]+@[^@\s]+\.[^@\s]+$` | |
| `avatarUrl` | `string \| null` | URL | Falls back to a generated initials avatar. |
| `roles` | `UserRole[]` | non-empty | |
| `department` | `string \| null` | `@maxLength 60` | |

---

## 9. `ApiError` — error envelope

Every 4xx / 5xx response must use this shape (or a subset). Returned with `Content-Type: application/json`.

| Field | Type | Description |
|---|---|---|
| `code` | `string` | Stable machine-readable code (`AUTH_EXPIRED`, `VALIDATION_FAILED`, `NOT_FOUND`, `RATE_LIMITED`, `INTERNAL_ERROR`). Frontend may dispatch different UI based on this. |
| `message` | `string` | `@maxLength 300`. Display-safe (no sensitive data). |
| `fields?` | `Record<string, string>` | Field-level validation errors. Keyed by JSON path (`"reason"`, `"user.email"`). |
| `traceId?` | `string` | Server correlation id. Frontend includes this in bug reports and Sentry breadcrumbs. |

**Example 422:**
```json
{
  "code": "VALIDATION_FAILED",
  "message": "Dismiss reason is required.",
  "fields": { "reason": "Reason must be at least 10 characters." },
  "traceId": "8f3c…b2a1"
}
```

---

## 10. `Paginated<T>` — list envelope

Reserved for endpoints that may return >50 items. Today every list is small enough to return as a plain array, but the helper exists so the backend can adopt pagination per-endpoint without a frontend rewrite.

| Field | Type | Description |
|---|---|---|
| `items` | `T[]` | The page contents. |
| `total` | `number` | Total available across all pages. |
| `page` | `number` | 1-indexed. |
| `pageSize` | `number` | Capped at 100 by the backend; default 25. |

---

## Field-length cheat sheet

| Entity | Field | Max | Behavior on overflow |
|---|---|---|---|
| `GlobalRisk` | `title` | 80 | Wraps to 2 lines in popup, truncates with `…` in search row |
| `GlobalRisk` | `subtitle` | 120 | Wraps; very long ones clip at popup width |
| `ActiveRisk` | `title` | 80 | Same |
| `ActiveRisk` | `subtitle` | 120 | Wraps |
| `ArchivedRisk` | `reason` | 500 | Wraps in modal; consider scroll if you exceed 500 |
| `Dependency` | `product` | 30 | Truncates with `…` |
| `Dependency` | `alternateSuppliers` | 200 | Joins with locale separator; clips |
| `Dependency` | `topImporters` | 120 | Truncates with `…` in narrow columns |
| `AppNotification` | `descKey` (resolved) | ~150 for clean 2-line fit | Wraps |
| `UserProfile` | `displayName` | 60 | Truncates in tooltip |
| `ApiError` | `message` | 300 | Wraps in toast / inline error |

For any field not listed, assume "as long as makes sense for a government UI — but the frontend won't break."
