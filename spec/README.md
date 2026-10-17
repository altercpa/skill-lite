# AlterCPA Lite — OpenAPI 3.1.0 Specifications

Thirteen self-contained OpenAPI 3.1.0 files, one per API domain.

## Files

| File | Covers | Endpoints |
|---|---|---|
| `openapi-filter.json` | Filter campaigns (CRUD, copy), visit journal, statistics and journal clearing + /fltr/ cloaking runtime | 11 + 1 |
| `openapi-blacklist.json` | Blacklists: lists and entries, emptying a list, CSV import/export | 12 |
| `openapi-flows.json` | Flows and ways CRUD | 12 |
| `openapi-tracker.json` | Postback (+ short-form `/postbacks/{key}`), stats, leads, single lead, clicks, spend, settings, templates, networks; gangs/* wrappers; click API (TDS) | 10 + 6 + 4 |
| `openapi-sites.json` | Site storage | 5 |
| `openapi-domains.json` | Parked domains + domain confirmation files + parking DNS records | 10 |
| `openapi-adv.json` | adv — ad-spend export connectors (Facebook, TikTok, BIGO Ads, VK, MGid, LuckyAds, Dolphin/Dolphin Cloud, FBtool, MixAdvert, AdPartner) | 9 |
| `openapi-profile-sys.json` | User profile (info, partial edit, API key reset), registration, system | 9 |
| `openapi-logs.json` | logs — API-request and outgoing-postback journals | 2 |
| `openapi-fbpix.json` | fbpix — Facebook Conversions API (CAPI) pixels | 9 |
| `openapi-ttpix.json` | ttpix — TikTok Events API pixels | 9 |
| `openapi-gclid.json` | gclid — Google Ads / GA4 offline-conversion CSV export | 12 |
| `openapi-p2b.json` | p2b — Postback 2.0 outbound HTTP / Telegram | 11 |

## Key conventions

- **Success is HTTP 200** with `status: "ok"`. Errors carry `status: "error"` and an HTTP status derived from the error code (401 `key`/`auth`, 402 `unpaid`, 403 `access`/`ban`, 404 not-found family, 406 `dublicate`, 418 `bot`, 500 `app`/`func`/`db`, otherwise 400); the `X-API-Status` / `X-API-Error` headers repeat them. The `/fltr/` runtime always answers HTTP 200.
- **Standard auth**: `?id={user_id}-{api_key}` query parameter (apiKey scheme named `id`).
- **scope parameter** must always be in the GET query string, even when the body is JSON.
- Endpoints marked `security: []` do not require authentication.
- The `/fltr/` and `/track-*` routes have completely different auth/routing from the `/api/` family.
- The `adv` domain has no billing gate (`unpaid` never appears in its error set) and its
  ownership model is strictly binary (a user XOR a team) — no per-member team glue.

## Viewing

Use any OpenAPI 3.1 viewer. Quick options:

```bash
# Swagger UI via Docker
docker run -p 8080:8080 -e SWAGGER_JSON=/spec/openapi-filter.json \
  -v $(pwd):/spec swaggerapi/swagger-ui

# Redoc CLI
npx @redocly/cli preview-docs openapi-filter.json
```

Or paste into [editor.swagger.io](https://editor.swagger.io) or [Redocly Playground](https://redocly.com/demo/).
