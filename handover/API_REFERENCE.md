# API reference

Human-readable companion to [`API_SPECIFICATION.yaml`](./API_SPECIFICATION.yaml). Examples assume `Authorization: Bearer <token>` and `Accept: application/json` are present on every request.

Base URL: `${VITE_API_BASE_URL}` — set per environment in `.env.local`.

---

## Conventions

- **Method + path** is the contract — query params modify behavior.
- **Status codes:** `200` OK / `204` No Content / `400` Bad Request / `401` Auth missing or invalid / `403` Wrong role / `404` Not found / `422` Validation failed / `429` Rate limited / `5xx` Server error.
- **Error body:** always `{ code, message, fields?, traceId? }` (see [DATA_MODELS §9](./DATA_MODELS.md#9-apierror--error-envelope)).
- **Timestamps:** display-ready `"YYYY-MM-DD HH:MM UTC"`. (The frontend does not currently parse ISO; backend can switch later — see [INTEGRATION_GUIDE](./INTEGRATION_GUIDE.md).)
- **IDs:** integers for risks/dependencies/critical-goods/notifications; UUIDs for users.

---

## Auth

### `GET /auth/me`

Return the current user. Called once after login.

**Response 200**
```json
{
  "id": "8f3c1b2a-2f43-4c0e-9a18-1ce6b8b22013",
  "displayName": "Sarah Johnson",
  "email": "sarah.johnson@moft.gov.ae",
  "avatarUrl": null,
  "roles": ["analyst"],
  "department": "Strategic Sourcing"
}
```

### `POST /auth/logout`

Revoke the session. Returns `204`.

---

## Risks — global map

### `GET /risks/global`

Returns the array of monitored chokepoints rendered as pins on the world map.

**Response 200**
```json
[
  { "id": 1, "title": "Strait of Hormuz Closure", "subtitle": "IRGC Naval Exercises", "severity": "High", "coordinates": [56.3, 26.5] },
  { "id": 2, "title": "Red Sea Shipping Threat", "subtitle": "Security Concerns", "severity": "High", "coordinates": [43.3, 12.5] }
]
```

### `GET /risks/global/{id}`

Returns the popup body for a single map pin (title + commodities + news + description). Used by `RiskPopup.tsx`.

---

## Risks — Risk Intelligence list

### `GET /risks?status=active`
### `GET /risks?status=archived`

| Query param | Type | Required | Default |
|---|---|---|---|
| `status` | `"active"` \| `"archived"` | yes | — |
| `page` | integer ≥ 1 | no | `1` |
| `pageSize` | integer 1–100 | no | `25` |

**Response 200 (active example)**
```json
[
  {
    "id": 1,
    "title": "Strait of Hormuz Blockade",
    "titleKey": "riskList.straitOfHormuzBlockade",
    "subtitle": "IRGC Naval Exercises",
    "subtitleKey": "riskList.straitOfHormuzBlockade.sub",
    "category": "Supply Chain",
    "categoryKey": "risk.category.supplyChain",
    "severity": "High",
    "firstDetected": "2026-02-28 14:23 UTC",
    "lastUpdated": "2026-03-02 09:15 UTC",
    "goods": 23,
    "impactedGoods": 23
  }
]
```

### `GET /risks/{id}/detail`

Returns the breadcrumb title + impact-chain table for the Risk Intelligence detail page.

**Response 200**
```json
{
  "id": 1,
  "title": "Strait of Hormuz Blockade",
  "breadcrumb": "Strait of Hormuz Closure",
  "impactChain": [
    { "product": "Petroleum Products", "hsCode": "2710", "impactType": "Direct", "dependency": 32, "stock": 45, "inTransit": "12.0K MT" }
  ]
}
```

### `POST /risks/{id}/dismiss`

**Auth:** requires `officer` or `admin` role.

**Request body**
```json
{ "reason": "Risk has been mitigated. Naval exercise concluded." }
```

**Validation**
- `reason`: 10–500 chars, required.

**Response 200** — the freshly archived risk (matches `ArchivedRisk` schema).

**Response 422** when `reason` is missing or too short:
```json
{
  "code": "VALIDATION_FAILED",
  "message": "Dismiss reason is required.",
  "fields": { "reason": "Reason must be at least 10 characters." }
}
```

### `POST /risks/{id}/restore`

**Auth:** `analyst | officer | admin`.

**Body** same as dismiss (`{ reason }`). **Response 200**: `ActiveRisk`.

---

## Dependencies

### `GET /dependencies`

| Query param | Type | Required | Description |
|---|---|---|---|
| `country` | string ≤30 chars | no | Filter to commodities whose `primarySupplier` matches this English country name. |

**Response 200**
```json
[
  {
    "id": 1,
    "product": "Rice",
    "category": "Legumes",
    "hsCode": "1006",
    "primarySupplier": "India",
    "dependency": 50,
    "alternateSuppliers": "Pakistan (31%), Thailand (8%), Vietnam (4%)",
    "uaeConsumption": "27.5K",
    "uaeConsumptionUnit": "MT",
    "inventory": 82,
    "inventoryStatus": "good",
    "topImporters": "Al Dahra, KRBL UAE, LuLu Group",
    "intlPrice": 528,
    "priceChange": 4.8,
    "mfgCapacity": "—",
    "risk": "High"
  }
]
```

### `GET /dependencies/{id}`

Returns the dependency + its structured `alternateSupplierCountries` array for the per-product detail page (`DependencyDetail.tsx`).

---

## Critical Goods

### `GET /critical-goods`

Returns the array of monitored HS chapters. **All fields are translation keys** — no English fallbacks. Backend changes that add new chapters require an accompanying `translations.ts` update.

**Response 200**
```json
[
  { "id": 1, "chapterKey": "chapter.01-05", "categoryKey": "cg.cat.liveAnimals", "commoditiesKey": "cg.com.liveAnimals" }
]
```

---

## Notifications

### `GET /notifications`

**Response 200**
```json
{
  "unreadCount": 3,
  "items": [
    { "id": 1, "type": "high", "titleKey": "notif.ex.hormuz.title", "descKey": "notif.ex.hormuz.desc", "timeKey": "notif.ex.time.12m", "read": false }
  ]
}
```

### `POST /notifications/{id}/read` — mark one as read. `204`.
### `POST /notifications/read-all` — mark all as read. `204`.

---

## Search

### `GET /search?q={query}&limit={limit}`

| Query param | Type | Required | Notes |
|---|---|---|---|
| `q` | string 1–100 | yes | The user's search query. |
| `limit` | integer 1–50 | no | Default `20`. |

Returns matches grouped by entity. Frontend renders each non-empty group as a section.

**Response 200**
```json
{
  "total": 5,
  "risks": [ { "id": 1, "title": "Strait of Hormuz Closure", "subtitle": "IRGC Naval Exercises", "severity": "High", "coordinates": [56.3, 26.5] } ],
  "dependencies": [ { "id": 1, "product": "Rice", "hsCode": "1006", "primarySupplier": "India", "risk": "High", "category": "Legumes" /* + the rest of Dependency */ } ],
  "criticalGoods": [ { "id": 3, "chapterKey": "chapter.10", "categoryKey": "cg.cat.cereals", "commoditiesKey": "cg.com.cereals" } ],
  "countries": [ { "name": "India", "nameKey": "country.india", "code": "IN" } ]
}
```

**Match rules (backend):**
- Case-insensitive substring match.
- Match across the English field, the translated label (resolved server-side or accept both EN and AR query strings).
- Risks: match `title` + `subtitle` + `severity` (so `"high"` returns all High risks).
- Dependencies: match `product` + `hsCode` + `primarySupplier`.
- Critical Goods: match translated `chapterKey` + `categoryKey` + `commoditiesKey`.
- Countries: match `name` + translated `nameKey`.

---

## Rate limiting

Recommended baseline (adjust per endpoint):

| Endpoint family | Limit |
|---|---|
| Read endpoints (`GET`) | 60 req/min/user |
| Write endpoints (`POST`/`PATCH`/`DELETE`) | 20 req/min/user |
| `/search` | 30 req/min/user |
| Anonymous (no token) | 10 req/min/IP |

When rate-limited, return `429` with `Retry-After` header (seconds) and:
```json
{ "code": "RATE_LIMITED", "message": "Too many requests. Try again in 30 seconds.", "traceId": "..." }
```

---

## CORS

Allow the SPA origin only:

```
Access-Control-Allow-Origin: https://moft.gov.ae
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: GET, POST, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Authorization, Content-Type, X-Trace-Id
```

The SPA sends `credentials: "include"` so cookie-based session refresh (if you use it alongside JWT) works without CORS preflight failures.

---

## Caching

| Endpoint | Cache hint |
|---|---|
| `/risks/global`, `/critical-goods` | `Cache-Control: private, max-age=60` — small, infrequently updated |
| `/risks?status=*`, `/dependencies`, `/notifications` | `no-store` — should always be fresh |
| `/auth/me` | `private, max-age=300` — refresh every 5 min |
| Static assets (flags, maps) | Cached on CDN with hashed filenames; immutable |
