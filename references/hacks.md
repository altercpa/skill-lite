# AlterCPA Lite — Hack APIs (fbpix · ttpix · gclid · p2b)

Four JSON CRUD APIs added by tracker hacks. They share one **Foundation** (auth, scope,
verbs, partial edit, clone, response shape) and differ only in entity, fields and reference
lists. Read the Foundation once, then the section for your module.

| App | Entity | Id param | Purpose | Fires to |
|---|---|---|---|---|
| `fbpix` | pixel | `pb` | Facebook Conversions API postback rules | `graph.facebook.com` |
| `ttpix` | pixel | `pb` | TikTok Events API postback rules | `business-api.tiktok.com` |
| `gclid` | doc | `doc` | Google Ads / GA4 offline-conversion CSV export | Google pulls the CSV |
| `p2b` | postback | `pb` | Postback 2.0 — outbound HTTP call / Telegram message | any URL / Telegram |

---

## 1. Foundation (shared by all four)

### Base URL & format
```
https://{tracker}/api/{app}/{func}.json?id={token}[&scope=...][&params...]
```
- Success is HTTP **200** with body `{"status":"ok", ...}`; an error is `{"status":"error","error":"<code>"}` with the HTTP
  status derived from the code (see `SKILL.md` "Base URLs"), repeated in the `X-API-Status` / `X-API-Error` headers.
  `Content-Type: application/json`.
- Request formats: GET, POST `form-urlencoded`, or POST JSON (`Content-Type: application/json`).
- On collision POST wins over GET.

### Authentication
- Credential in **`id`** = `{user_id}-{api_key}` (e.g. `id=123-abcde`), from the tracker
  `/profile` page. Also accepted as POST `authid`, or the HMAC-SHA1 signature form.
- **`id` is the credential only.** The entity id goes in the entity-named param —
  `pb` (fbpix/ttpix/p2b) or `doc` (gclid). Never read the entity id from `id`.

### Scopes (ownership)
Every entity is owned as **user XOR gang** — a personal record has `user>0, gang=0`; a team
record has `user=0, gang>0`. **p2b additionally allows a system record** (`user=0, gang=0`).

| `?scope=` | Who | Owner targeted |
|---|---|---|
| `user` (default) | any authenticated user | the caller |
| `gang` (or `team`) | team lead (team admin) | the team |
| `admin` (`sys`/`system`/`all`) | an admin account | any `?user=` / `?gang=` |

- **`scope` must be a GET parameter**, even for POST/JSON requests. `user` / `gang` may be
  GET or POST.
- p2b **system** records: `scope=admin&user=0&gang=0`.
- Every function re-checks the addressed record belongs to the scope → `access` otherwise.

### Verbs (uniform)
| Verb | Method | Purpose |
|---|---|---|
| `list` | GET | paginated list for the scope |
| `show` | GET | one entity, full projection |
| `add` | POST | create (created active where an active flag exists) |
| `edit` | POST | **partial** update (incl. the `active` flag) |
| `clone` | POST | duplicate under a new name |
| `del` | POST | delete |
| *reference verbs* | GET | `events` / `tags` / `goals` / … — see each module |

### Partial edit, activation, clone
- **`edit` is partial**: only the fields present in the request are written; the rest are
  untouched. Sending no recognized field returns the record unchanged.
- **Activation/deactivation** = `edit` with the single `active` field (`1`/`0`). Modules
  without an active field (gclid) have no `active` field at all.
- Owner fields and runtime/read-only fields are never written by `edit`.
- **`clone`** copies a scope-owned source under a new **`name`** (required), created active.
  fbpix/ttpix write the name to `comment` (there is no separate name field); gclid also resets export
  progress (`last=0`, `lead=0`).

### Response shape
- `status` `ok`/`error`; on error a machine `error` code.
- `list` → `data` keyed by entity id (short row) + `nav {total,page,pages,show}`;
  `nav` omitted when `show=0` (returns all). `page` default 1, `show` default **50**.
- `show`/`add`/`edit`/`clone` → `data` = full row.
- `del` → `data: {id}`.

### Common error codes
`scope` (bad/insufficient), `access` (record outside scope), `no-id` (missing entity id
param), `no-name`, `no-<param>` (missing required field), `bad-<param>` (value not among the allowed ones), `not-found`, `db`. HTTP status: `access` 403, `no-id` / `not-found` 404, `db` 500, every
other code 400 (an unpaid or key/ban failure from the framework uses 402 / 401 / 403). Writes take effect
immediately.

---

## 2. fbpix — Facebook Pixel

**App** `fbpix` · **entity** pixel · **id param** `pb`. A pixel matches leads where field
`sub` equals `value` and fires Facebook events (`new`/`hold`/`approve`) to
`graph.facebook.com` with the pixel's CAPI `token`.

**Verbs:** `list · show · add · edit · clone · del` + reference `events · tags · goals`.

**Short row** (list): `id, active, status, pixel, sub, value, new, hold, approve, data,
noclick, proxy(bool), domain, comment`.
**Full row** (show/add/edit/clone): short + `user, gang, goal[], token`.

| Field | Type | On add | Notes |
|---|---|---|---|
| `pb` | int | — | entity id (in query), required by show/edit/clone/del |
| `active` | 0/1 | add sets 1 | toggle via `edit` |
| `status` | int 0–4 | — | **read-only** runtime health: 0 new · 1 ok · 2 bad params · 3 token expired · 4 network error |
| `pixel` | int | **required** | Facebook Pixel ID; `list` filter |
| `sub` | code | **required** | matched lead field; must be a `tags` code → else `bad-sub` |
| `value` | line | no | value to match in `sub`; empty ⇒ match on the Pixel ID |
| `token` | line | **required** | Facebook CAPI access token |
| `goal` | int or int[] | no | goal-id filter; empty = any goal |
| `proxy` | line | no | `host:port` or `scheme://user:pass@host:port` |
| `data` | 0/1 | default 1 | send hashed lead PII as `user_data` |
| `noclick` | 0/1 | default 0 | fire even without a click identifier |
| `new` | event | default `Lead` | FB event for "new"; must be an `events` value → else `bad-event` |
| `hold` | event | no | FB event for "hold" (empty clears) |
| `approve` | event | no | FB event for "approve" (empty clears) |
| `domain` | line | no | `event_source_url` template; macros `{host}`, `{utm_source}`, … |
| `comment` / `name` | line | no | free-text label — the pixel's de-facto **name** |

`list` filters: `pixel`, `sub`, `value`, `status` (+ `page`, `show`). Ordered by Pixel ID asc.
`add` required → else `no-pixel` / `no-sub` / `no-token`.

**Reference:** `events` → array of 17 FB event names (static). `tags` → `{code: label}` of
bindable lead fields. `goals` → `{goalId: name}` for the scope, `0: "No goal"` prepended
(scope-aware).

---

## 3. ttpix — TikTok Pixel

The TikTok twin of fbpix. **App** `ttpix` · **entity** pixel · **id param** `pb`. Fires TikTok
events to `business-api.tiktok.com` with the pixel's Access-Token.

**Diff vs fbpix:** `pixel` is a **string** (TikTok Pixel code), and there are **no `data` /
`noclick` fields**. Everything else — verbs, scope, partial edit, clone, errors, reference
endpoints — is identical.

**Short row:** `id, active, status, pixel, sub, value, new, hold, approve, proxy(bool),
domain, comment`. **Full row:** + `user, gang, goal[], token`.

Required on `add`: `pixel`, `sub`, `token`. `sub` must be a `tags` code (`bad-sub`);
`new`/`hold`/`approve` must be `events` values (`bad-event`) — the TikTok event set differs
from Facebook's (e.g. `Donate` is FB-only and rejected here). `events` returns ~19 TikTok
event names.

---

## 4. gclid — Google Conversions Export

**App** `gclid` · **entity** doc · **id param** `doc`. A doc is a lead filter + export config
that exposes a live CSV `url` which Google Ads / GA4 pulls on a schedule. **No Google API is
called.** **No active/status field** → there is **no `active` field and no activation**.

**Verbs:** `list · show · add · edit · clone · del` + reference `tags · domains · offers ·
flows · geo · goals`. `clone` resets export progress (`last`/`lead` → 0).

**Short row** (list): `id, name, item, domain, geo[], last(ro), url(ro)`.
**Full row:** short + `user, gang, offers[], flows[], utms, utmc, utmn, utmt, utmm, utmd,
trash, cancel, lead(ro)`.

| Field | Type | On add | Notes |
|---|---|---|---|
| `doc` | int | — | entity id (in query) |
| `name` | line | **required** | doc name |
| `item` | code | **required** | lead field holding the gclid to export; must be a `tags` code → else `bad-item` |
| `domain` | int | default 0 | parked-domain id used to build the export URL; omitted or `0` = the system domain (see `domains`) |
| `geo` | iso or iso[] | no | ISO country filter; empty = any |
| `offers` | int or int[] | no | offer filter — **Pro only; ignored / empty on Lite** |
| `flows` | int or int[] | no | flow filter |
| `utms`/`utmc`/`utmn`/`utmt`/`utmm`/`utmd` | line | no | utm_source/campaign/content/term/medium/domain filters (empty clears) |
| `trash` | 0/1 | default 1 | exclude leads marked trash |
| `cancel` | 0/1 | default 0 | exclude cancelled leads |
| `last` | ts | — | **read-only** last export run (0 = never) |
| `lead` | int | — | **read-only** high-water mark of exported lead id |
| `url` | line | — | **read-only, computed** CSV export URL on the doc's domain; on the system domain when `domain` is `0` or not one of the owner's domains |

`list` filters: `search` (by name), `domain`, `item` (+ `page`, `show`). Ordered by doc id
desc. `add` requires `name` and `item` → else `no-name` / `no-item`; an item that cannot be resolved → `bad-item`.

**Reference:** `tags` → `{code: label}` exportable fields. `domains` → `{domainId: host}`
(scope-aware; key `0` is the system domain and comes first). `offers` → `{offerId: name}` (Pro only; **`{}` on Lite**). `flows` →
`{flowId: name}` (scope-aware). `geo` → `{iso: country}` (~253 static). `goals` →
`{goalId: code}` (scope-aware, **informational only** — gclid has no per-doc goal selection).

---

## 5. p2b — Postback 2.0

**App** `p2b` · **entity** postback · **id param** `pb`. An outbound rule that, when a lead
changes state, fires an **HTTP call** (`type=0`) or a **Telegram message** (`type=1`),
filtered by an optional field/value tag, a goal, and the enabled status events. Sends run
synchronously or through a queue. Supports **system** postbacks (`scope=admin&user=0&gang=0`).
Deleting a postback also **purges its queued sends**.

**Verbs:** `list · show · add · edit · clone · del` + reference `events · tags · goals ·
formats · types`.

**Short row** (list): `id, active, name, type, filter, value, global, new, hold, approve,
cancel, trash, async, nolog, auth`.
**Full row:** short + `user, gang, goal[], url, post, format, header, proxy, aurl, apost,
aformat, aheader, apath, avalid`.

| Field | Type | On add | Notes |
|---|---|---|---|
| `pb` | int | — | entity id (in query) |
| `name` | line | **required** | postback name |
| `type` | 0/1 | default 0 | 0 = HTTP postback, 1 = Telegram message |
| `url` | line | **required** | request URL; **for Telegram = the bot token** |
| `post` | text | req. if `type=1` | POST body; **for Telegram = the message text** → else `no-post` |
| `header` | text | req. if `type=1` | request headers; **for Telegram = the chat_id** → else `no-header` |
| `format` | 0–3 | default 0 | body format (see `formats`) |
| `filter` | code | no | lead field "tag" to match; must be a `tags` code (`bad-filter`); empty clears |
| `value` | line | no | value `filter` must equal; empty = any |
| `goal` | int or int[] | no | goal filter; empty = any |
| `global` | 0/1 | default 0 | fire on any status change |
| `new`/`hold`/`approve`/`cancel`/`trash` | 0/1 | default 0 | fire on that lead-status event |
| `active` | 0/1 | add sets 1 | toggle via `edit` |
| `async` | 0/1 | default 1 | send through the queue vs synchronously |
| `nolog` | 0/1 | default 0 | skip the request log |
| `auth` | 0/1 | default 0 | enable the pre-auth handshake |
| `aurl`/`apost`/`aformat`/`aheader`/`apath`/`avalid` | — | `avalid` def. 60 | pre-auth handshake: URL / body / format(0–3) / headers / dotted JSON path to the token / token lifetime (min) |

> **Events note:** on `add`, every unsent event flag stores 0, so a postback with all six
> events 0 never fires — send at least one of `global`/`new`/`hold`/`approve`/`cancel`/`trash`.
> `edit` writes only the event flags present in the request.

`list` filters: `search` (name), `filter`, `value`, `event` (0–5), `type` (+ `page`, `show`).
Ordered by name asc. `add` required → else `no-name` / `no-url` (+ `no-post` / `no-header`
for Telegram).

**Reference:** `events` → array of `{id,key,name}` for the six events (0 global · 1 new ·
2 hold · 3 approve · 4 cancel · 5 trash). `tags` → `{code: label}` filterable fields. `goals`
→ `{goalId: name}`, `0: "No goal"` prepended (scope-aware). `formats` → `{0..3: label}` body
formats. `types` → `{0:"PB", 1:"TG"}`.

---

## OpenAPI

Machine-readable OpenAPI 3.1.0 per module: `spec/openapi-fbpix.json`,
`spec/openapi-ttpix.json`, `spec/openapi-gclid.json`, `spec/openapi-p2b.json`.
