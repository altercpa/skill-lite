---
name: altercpa-lite-api
description: AlterCPA Lite tracker - the full API reference plus the user guide. Use whenever making calls to the tracker API, building integrations or writing code against it, and whenever explaining how the tracker works (flows and ways, split tests, traffic processing methods, domain parking, postbacks and pixels, goals, leads and statistics, teams, site storage, visit filtering/cloaking, blacklists, spend analytics). Triggers on any mention of tracker API endpoints, postbacks, flows, domains, filter/cloaking, campaign copy, visit journal, statistics, journal clearing, IP/network/header blacklists, click tracking, a single lead with its status history, API-request and postback logs, API key reset, ad-spend connectors (Facebook, TikTok, BIGO Ads, VK, MGid, LuckyAds, Dolphin, FBtool, MixAdvert, AdPartner), and the hack APIs - Facebook Conversions API pixels (fbpix), TikTok Events API pixels (ttpix), Google Ads offline conversions (gclid), Postback 2.0 HTTP/Telegram postbacks (p2b).
---

# AlterCPA Lite — API Reference

Use this skill to make correct API calls to the AlterCPA Lite tracker. Read the relevant reference file for full parameter/response details. If you need to build an integration, always check authentication and scope first. For how the tracker behaves rather than which call to make, see **Tracker Guide** below.

---

## Authentication

**Token format:** `{user_id}-{api_key}` (e.g. `13-4a3bc9d…`)  
**Get your token:** tracker `/profile` page → API key field.

**Pass the token:**
- GET: `?id=13-4a3bc9d…` in the query string
- POST: `id=13-4a3bc9d…` as a form field, or `authid=…` in POST body
- JSON: `"id": "13-4a3bc9d…"` in the JSON body

**HMAC alternative:** Instead of the raw key, you can pass an HMAC-SHA1 of the POST body (encoded as PHP's `http_build_query` does) keyed by your API key. The key in the URL stays `{user_id}-{hmac}`.

**Admin impersonation:** Add `uu={target_user_id}` (GET or POST) to act as another user. Requires an admin account.

---

## Base URLs

```
https://{tracker}/api/{app}/{func}.json?id={token}[&scope=...][&other_params...]
```

The `/fltr/` endpoint is separate — see **Filter Reference**.

Success is HTTP 200 with JSON body `{"status":"ok", ...}`. An error is `{"status":"error","error":"<code>"}` with an HTTP status derived from the code: 401 `key`/`auth`, 402 `unpaid`, 403 `access`/`ban`, 404 `no-id`/`bad-id`/`not-found` and similar, 406 `dublicate`, 418 `bot`, 500 `app`/`func`/`db`, otherwise 400. The headers `X-API-Status` and `X-API-Error` repeat them. The separate `/fltr/` endpoint always answers HTTP 200.  
The `Content-Type` header is `application/json`.

---

## Request Formats

- **GET:** params in query string
- **POST form:** `Content-Type: application/x-www-form-urlencoded`
- **POST JSON:** `Content-Type: application/json` — body decoded into POST

Most endpoints accept both GET and POST. In `change`/`edit` functions POST wins over GET on collision.

**Important:** `scope` must be passed as a GET parameter even when the body is JSON. Example: `?scope=gang` in the URL.

---

## Scopes

Most `/api/` endpoints accept an optional `scope` GET parameter:

| `scope` | Who can use | What it does |
|---|---|---|
| `user` (default) | Any authenticated user | Operates on the current user's data |
| `gang` or `team` | Team leads only (team admin) | Operates on team data; optionally add `user=N` to filter by team member |
| `admin`, `sys`, `system`, `all` | Admin accounts | Full access; add `user=N` and/or `gang=N` to target specific records |

Error `scope` is returned when the caller lacks permission for the requested scope.

**Note:** the `adv` app's ownership is strictly binary (owned by a user XOR by a team) — its `gang` scope does not honor the `user=` sub-filter (no per-member visibility marker). See `references/adv.md`.

---

## Common Framework Errors

These can be returned by any `/api/` endpoint before the function runs:

| `error` | HTTP | Meaning |
|---|---|---|
| `key` | 401 | Bad or missing API token |
| `ban` | 403 | User is banned or inactive |
| `bot` | 418 | Request UA is SkypeUriPreview / TelegramBot / TwitterBot |
| `func` | 500 | Function name not implemented |
| `app` | 500 | App name not registered |
| `db` | 500 | Database error |
| `scope` | 400 | Invalid scope or insufficient permissions |
| `unpaid` | 402 | Active subscription required (when paid subscriptions are enabled) |

`unpaid` is never returned by the `adv` app — it has no billing gate.

---

## Endpoint Index

### Filter & Cloaking
Read `references/filter.md` for full spec.

| Endpoint | Method | Description |
|---|---|---|
| `/fltr/{user}-{key}-{campaign_id}` | POST | **Check visit / cloaking runtime** (separate auth, not /api/) |
| `api/filter/list.json` | GET | List filter campaigns |
| `api/filter/get.json` | GET/POST | Get single campaign |
| `api/filter/set.json` | POST | Create or update campaign (a create can answer `limit`: plan exhausted) |
| `api/filter/del.json` | POST | Delete campaign |
| `api/filter/make.json` | GET | Download integration index.php |
| `api/filter/block.json` | POST | Add IP/network/header rule to a blacklist the scope permits |
| `api/filter/postback.json` | GET | Record lead postback to mini-tracker |
| `api/filter/log.json` | GET/POST | Visit journal: paginated per-visit rows (reason code, geo, IP, UA, headers), filterable by site/list/result/ip/ua32/geo |
| `api/filter/stats.json` | GET/POST | Visit statistics grouped by `item` (`date` or `site`) over a date range, with reason breakdown and percentages; days in the caller's time zone when per-user zones are on (`dataset.tz`) |
| `api/filter/clear.json` | POST | Clear the visit journal: rows matching the `log` filters (`site`, `list`, `result`, `ip`, `ua32`, `geo`), the whole scope when none; `scope=gang` refused (`access`) |
| `api/filter/clone.json` | GET/POST | Copy a campaign (`uid` source + `domain` new name) into the source owner's account, counters zeroed; `limit` when the plan is exhausted |

`list`, `get`, `set`, `del`, `make`, `block`, `log`, `stats` and `clone` accept `scope` / `user` / `gang`; `clear` accepts `user` / `gang` under `scope=admin` and refuses `scope=gang`.

### Blacklists
Read `references/blacklist.md` for full spec. Blacklists are the lists of IPs, networks and header rules that filter campaigns consult; lists belong to a user, a team or the whole tracker.

| Endpoint | Method | Description |
|---|---|---|
| `api/blacklist/avail.json` | GET | Lists a campaign owner may attach (active global + personal + team) with default/selected flags |
| `api/blacklist/list.json` | GET | List block-lists of the scope with kind filter, name search, paging |
| `api/blacklist/show.json` | GET | Entries of one list with search, reason/header filters, paging |
| `api/blacklist/add.json` | POST | Create an empty list (name + kind 1-4); global lists start inactive |
| `api/blacklist/edit.json` | POST | Partial update of list settings: name, status, default, feed key, URL mirror |
| `api/blacklist/del.json` | POST | Delete a list with all its entries |
| `api/blacklist/block.json` | POST | Add one entry (IP, network, IPv6 network or header rule); repeats update the existing one |
| `api/blacklist/remove.json` | POST | Delete one entry by ID |
| `api/blacklist/change.json` | POST | Partial update of one entry (`duplicate` on collision) |
| `api/blacklist/clean.json` | POST | Delete every entry of one list in place (list ID kept, `{id, del}`); `locked` for feed lists and URL mirrors |
| `api/blacklist/import.json` | POST | Merge a CSV file into a list (multipart, merge-only, per-kind columns) |
| `api/blacklist/export.json` | GET | Download a whole list as CSV (not a JSON envelope; same format `import` reads) |

### Flows & Ways
Read `references/flows.md` for full spec.

| Endpoint | Description |
|---|---|
| `api/flows/list.json` | Paginated list of flows |
| `api/flows/get.json` | Full flow card + ways |
| `api/flows/file.json` | Download TDS client PHP file |
| `api/flows/caps.json` | Current lead caps per way |
| `api/flows/create.json` | Create new flow |
| `api/flows/clone.json` | Clone existing flow |
| `api/flows/change.json` | Edit flow settings |
| `api/flows/remove.json` | Delete flow |
| `api/flows/add.json` | Add way to flow |
| `api/flows/copy.json` | Copy a way |
| `api/flows/edit.json` | Edit way settings |
| `api/flows/del.json` | Delete a way |

### Click API (TDS)
Read `references/tracker.md` for full spec.

| Endpoint | Description |
|---|---|
| `GET /track-tds/{flow_code}` | Register click by flow code |
| `GET /track-dom/{domain}` | Register click by parked domain |
| `GET /track-pxl/{flow_code}` | Pixel tracking on event |
| `GET /track-thx/{flow_code}` | Pixel tracking on approval |

These are **not** standard `/api/` routes — they are client-facing routes called by the `server.php` client script installed on your domain.

### Tracker Stats & Integration
Read `references/tracker.md` for full spec.

| Endpoint | Also available as | Description |
|---|---|---|
| `api/tracker/postback.json` | `api/gangs/postback.json` | Receive lead postback |
| `api/tracker/stats.json` | `api/gangs/stats.json` | Click + lead statistics (GET/POST, `show`/`page` pagination, `sort`/`order`) |
| `api/tracker/leads.json` | `api/gangs/leads.json` | Leads list (GET/POST, `show`/`page` pagination, `sort`/`order`; filters include `mobile`, `os`, `browser`, `hour`) |
| `api/tracker/lead.json` | `api/gangs/lead.json` | One lead by `lead` ID with its status-change history (`changes`) |
| `api/tracker/clicks.json` | `api/gangs/clicks.json` | Raw click list / traffic log (GET/POST, `show`/`page` pagination, `sort`/`order`, `hour` filter; response key is `data`, not `leads`) |
| `api/tracker/spend.json` | `api/gangs/spend.json` | Import click spend |
| `api/tracker/templates.json` | — | CPA-network postback templates (reference book) |
| `api/tracker/networks.json` | — | Traffic-source presets (reference book) |
| `api/tracker/settings.json` | — | Update storage/control settings (`scope=gang` in the POST body needs a team admin, else `scope`) |

Short-form postback route: `POST /postbacks/{flow_key}` (positive key = user, negative = gang).

### Sites Storage
Read `references/sites-domains.md` for full spec.

| Endpoint | Description |
|---|---|
| `api/sites/list.json` | List sites |
| `api/sites/add.json` | Upload site (ZIP or mirror URL) |
| `api/sites/edit.json` | Edit site name/content |
| `api/sites/del.json` | Delete site |
| `api/sites/download.json` | Download site as ZIP |

### Parked Domains
Read `references/sites-domains.md` for full spec.

| Endpoint | Description |
|---|---|
| `api/domains/list.json` | List domains |
| `api/domains/add.json` | Add domain |
| `api/domains/check.json` | Manual domain status check |
| `api/domains/del.json` | Delete domain |
| `api/domains/glue.json` | Attach/detach domain to flow |
| `api/domains/files.json` | List domain confirmation files |
| `api/domains/confirm.json` | Create/replace a confirmation file (upsert) |
| `api/domains/change.json` | Rename/rewrite a confirmation file |
| `api/domains/remove.json` | Delete a confirmation file |
| `api/domains/dns.json` | Parking IPs and the DNS records (`A`/`AAAA`) to point a domain at |

### Ad-Spend Connectors (adv)
Read `references/adv.md` for full spec.

Ad-network spend-export integrations (Facebook, TikTok, BIGO Ads, VK, MGid, LuckyAds,
Dolphin/Dolphin Cloud, FBtool, MixAdvert, AdPartner) that periodically write per-click
cost into the tracker. No billing gate. Ownership is strictly binary
(a user XOR a team) — no per-member team visibility marker.

| Endpoint | Description |
|---|---|
| `api/adv/types.json` | List connector types (type code → localized name) |
| `api/adv/tags.json` | List spend-matching tag labels (the `field` selector) |
| `api/adv/list.json` | Paginated list of configured integrations |
| `api/adv/form.json` | Integration + connector field descriptors + dataset + setup text (`desc`) |
| `api/adv/add.json` | Create integration shell (type + name only) |
| `api/adv/edit.json` | Edit integration settings (partial; connector meta all-or-nothing) |
| `api/adv/update.json` | Force an out-of-schedule CRON run |
| `api/adv/del.json` | Delete integration and purge its request log |
| `api/adv/log.json` | Paginated/filtered request journal |

### Profile & System
Read `references/profile-sys.md` for full spec.

| Endpoint | Auth required | Description |
|---|---|---|
| `api/profile/info.json` | Yes | Get profile data (avatar URL, team card, contact fields, time zones) |
| `api/profile/edit.json` | Yes | Edit profile (partial update: only the sent fields change) |
| `api/profile/reset.json` | Yes | Issue a new API key (the old one stops working) |
| `api/profile/login.json` | Session (no token) | Get current session user |
| `api/profile/register.json` | No | Register new user |
| `api/profile/recover.json` | No | Request password recovery email |
| `api/sys/ping.json` | Yes | Liveness check |
| `api/sys/users.json` | Yes (admin accounts only) | User list/search |
| `api/sys/settings.json` | No | Tracker timezone |

### Logs
Read `references/logs.md` for full spec.

Read-only journals, newest first, paginated by `show`/`page`, entries keyed by log ID. `scope` selects whose entries (own / team / admin).

| Endpoint | Description |
|---|---|
| `api/logs/requests.json` | Journal of incoming API requests (any app/function), filter by `from`/`to`, `site`, `click`, `lead`, `flow`, `err` |
| `api/logs/postbacks.json` | Journal of outgoing postbacks (URL, body, remote response, HTTP code, timer), filter by `from`/`to`, `lead`, `site`, `event`, `flow`, `click` |

### Hack APIs — Conversions & Postbacks
Read `references/hacks.md` for the shared Foundation + all four modules.

These four hack apps share one skeleton (auth, `scope`, verbs `list/show/add/edit/clone/del`,
partial edit, response shape). The entity id is passed in an **entity-named** param — `pb`
(fbpix/ttpix/p2b) or `doc` (gclid) — never in `id`.

| App | Endpoints | Description |
|---|---|---|
| `fbpix` | `api/fbpix/{list,show,add,edit,clone,del}.json` + `{events,tags,goals}.json` | Facebook Conversions API (CAPI) server pixels |
| `ttpix` | `api/ttpix/{list,show,add,edit,clone,del}.json` + `{events,tags,goals}.json` | TikTok Events API server pixels (no `data`/`noclick`; `pixel` is a string) |
| `gclid` | `api/gclid/{list,show,add,edit,clone,del}.json` + `{tags,domains,offers,flows,geo,goals}.json` | Google Ads / GA4 offline-conversion CSV export docs (**no `active`**; `domain` optional, `0` = system domain) |
| `p2b` | `api/p2b/{list,show,add,edit,clone,del}.json` + `{events,tags,goals,formats,types}.json` | Postback 2.0 — outbound HTTP call or Telegram message (supports **system** records) |

---

## Reference Files

| File | Contents |
|---|---|
| `references/filter.md` | Filter campaigns CRUD and copy, visit journal, statistics and journal clearing, `/fltr/` cloaking endpoint |
| `references/blacklist.md` | Blacklists: lists and entries CRUD, emptying a list, import/export, header IDs |
| `references/flows.md` | Flow and way CRUD, all parameters, caps |
| `references/tracker.md` | Postback, stats, leads, single lead, clicks, spend, settings, click API (TDS) |
| `references/sites-domains.md` | Sites storage, parked domains, domain confirmation files, parking DNS records |
| `references/adv.md` | Ad-spend connector catalogue, CRUD, scheduling, request log |
| `references/profile-sys.md` | Profile management (info, partial edit, API key reset), system endpoints |
| `references/logs.md` | API-request journal, outgoing-postback journal |
| `references/hacks.md` | Hack APIs: fbpix, ttpix, gclid, p2b — shared Foundation + per-module fields |

When writing code that calls the API, always read the relevant reference file first to get exact parameter names, types, and error codes.

## Tracker Guide

The API reference says *which call*. For *how the tracker behaves* — what a setting does, why traffic went where it did, how a feature fits together — read the matching user-guide page. Pages are short (2–12 KB): open the one or two that fit the question, not the whole set.

- **Exact parameters, types and error codes live in `references/` and `spec/`**; they win when a guide page says otherwise. The guide is written for the web interface ("click…", screens, screenshots); images are not shipped.
- Guide links to `api.md` / `api-*.md` and root-relative links such as `/profile` are not shipped: the first point to material that is in `references/` (`api-tracker.md` → `references/tracker.md`), the second to screens of the user's own tracker.
- **Two things are called filtering.** A flow's own filtering (`filter.md`) is a few switches inside a flow or way, optionally attached to a filtering campaign. A *filtering campaign* (`start.md`, `code.md`, `tracker.md`, `blacklist.md`) is a standalone product with its own domain, file and statistics — the one `api/filter/*` and `/fltr/` manage. Blacklists are shared by both kinds and managed by `api/blacklist/*` (`references/blacklist.md`).

**Tracker**

| Page | Read it when the question is about |
|---|---|
| `docs/en/flows.md` | Flow setup overview: creating or cloning a flow, per-way settings (geo, probability, device, pause), next step, link versus local file, URL macros such as {click}. |
| `docs/en/split.md` | Choosing between ways: probabilities, lead caps per period and status, work schedule and time zone, multi-armed bandit optimization, best-option share, new way on every visit. |
| `docs/en/method.md` | Which working method a way or trafficback uses: 302/301/meta/JS redirects, iframe, curl, local file, stored site, error or blank page, replacement macros ({next}, {local}). |
| `docs/en/domain.md` | Parking own domains: DNS A/AAAA records, Cloudflare modes, status and SSL, where to park, gluing a domain to a flow's main page, expiry alerts. |
| `docs/en/postback.md` | Incoming server-side postbacks (network to tracker): integration templates, query parameters, custom status mapping, meaning of hold, leads by flow ID without click ID, error log. |
| `docs/en/team.md` | Arbitration teams: joining or creating one, leader access to members' clicks, leads, flows and spend, data before joining or after leaving, shared blacklists, storage, API. |
| `docs/en/sites.md` | Own site storage: installing it, uploading ZIP sites, white screen or 403 after upload, adding {click} and {next} to pages, mirroring a site, changing keys. |
| `docs/en/filter.md` | Filtering inside a flow or way: bot, blacklist, country, language, ad click ID checks, trafficback for blocked visits, attaching a filtering campaign to a flow. |
| `docs/en/tags.md` | Outgoing postbacks to ad networks on lead events: source templates, UTM/SubID/data tags and which reach statistics, event types, macros, Facebook events, POST/JSON format, outgoing log. |
| `docs/en/pixel.md` | Client-side pixel tracking instead of postbacks: click ID cookie, img or JSON pixel URL, lead versus success pixel, status/goal/price parameters, cheating risk. |
| `docs/en/stats.md` | Reading reports: lead table, click statistics by date/flow/way/geo/device/tags, metric definitions (CR, EPC, EPL, approve, ROI, CPL), hierarchical analytics, clicks log, filters and Excel export. |
| `docs/en/spend.md` | Getting ad costs into statistics: four import methods (auto loaders, cpc in link, manual, API), UTM tag needed to match spend, per-network tips (Facebook, Dolphin). |
| `docs/en/goal.md` | Up to five goals (registration versus deposit, RevShare): goal code in postbacks, per-goal columns, uid and status=approve rules for new leads versus goal changes. |

**Filtering campaigns**

| Page | Read it when the question is about |
|---|---|
| `docs/en/start.md` | Creating a filtering campaign (not a flow's built-in filter): domain, work modes, countries, target and dummy sites, IP/OS/browser/language checks, when to re-download the file. |
| `docs/en/code.md` | Filtering-campaign file: what target and dummy sites mean, how each shows (redirects, iframe, curl, PHP include, WordPress, blank), {geo} target files, re-download after method change. |
| `docs/en/tracker.md` | Success counting in filtering campaigns: mini-tracker counters and metrics, its uid/status/payout postbacks (no duplicate handling), and what the full Tracker menu section offers. |
| `docs/en/blacklist.md` | The blacklists themselves (not filter switches): system versus personal lists, types, active versus default flags, subnet sizes, regex header rules, team sharing, CSV import/export. |

**Other**

| Page | Read it when the question is about |
|---|---|
| `docs/en/server.md` | Choosing a VPS for the tracker: server size by daily clicks, recommended OS, hosting provider pros and cons (Inferno, Hetzner, BulletProof, Timeweb). |

## OpenAPI Specs

Machine-readable OpenAPI 3.1.0 specs are in `spec/`. Thirteen files, one per domain:

| File | Covers |
|---|---|
| `spec/openapi-filter.json` | `/api/filter/*` + `/fltr/` cloaking |
| `spec/openapi-blacklist.json` | `/api/blacklist/*` — block-lists and entries |
| `spec/openapi-flows.json` | `/api/flows/*` (flows + ways) |
| `spec/openapi-tracker.json` | `/api/tracker/*`, `/api/gangs/*`, `/track-*` (TDS) |
| `spec/openapi-sites.json` | `/api/sites/*` |
| `spec/openapi-domains.json` | `/api/domains/*` |
| `spec/openapi-adv.json` | `/api/adv/*` — ad-spend export connectors |
| `spec/openapi-profile-sys.json` | `/api/profile/*`, `/api/sys/*` |
| `spec/openapi-logs.json` | `/api/logs/*` — API-request and outgoing-postback journals |
| `spec/openapi-fbpix.json` | `/api/fbpix/*` — Facebook Conversions API pixels |
| `spec/openapi-ttpix.json` | `/api/ttpix/*` — TikTok Events API pixels |
| `spec/openapi-gclid.json` | `/api/gclid/*` — Google offline-conversion export |
| `spec/openapi-p2b.json` | `/api/p2b/*` — Postback 2.0 (HTTP / Telegram) |

See `spec/README.md` for viewing instructions.
