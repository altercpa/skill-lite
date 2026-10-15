# Filter & Cloaking API Reference

## Overview

Two distinct subsystems:

1. **`/api/filter/*`** — manage filter campaigns (CRUD, copy, integration file, postbacks, adding blacklist items), read the visit journal and its statistics and clear the journal. Eleven functions: `list`, `get`, `set`, `del`, `make`, `block`, `postback`, `log`, `stats`, `clear`, `clone`. Standard `/api/` family, uses `?id=` auth and `scope`.
2. **`/fltr/{user}-{key}-{campaign_id}`** — runtime cloaking endpoint. **Completely separate routing**, own auth embedded in URL, always returns JSON, only accepts `x-www-form-urlencoded` POST.

Blacklists themselves (create, configure, edit entries, import/export CSV) are managed by the separate `blacklist` API — see `references/blacklist.md`. Here, `filter/set` attaches lists to a campaign by ID and `filter/block` appends one item to an existing list.

---

## /api/filter — Campaign Management

### Auth & conventions
- Token: `?id={user_id}-{api_key}`
- Admin override: `uu={user_id}` in GET/POST (requires an admin account); every function then acts as that user.
- `scope` — **GET query string only** (even with a JSON body). Applies to `list`, `get`, `set`, `del`, `make`, `block`, `log`, `stats`, `clear`, `clone`; `postback` always works on the authenticated user. `clear` refuses `scope=gang` with `access`, even for a team admin.

| `scope` | Requirement | Campaigns / visits addressed |
|---|---|---|
| `user` (default; any unknown value) | none | Own |
| `gang` / `team` | key owner is the admin of a gang | Campaigns of all gang members; `user={member id}` (POST first, then GET) narrows to one member — a non-member gives error `scope` |
| `admin` / `sys` / `system` / `all` | admin account | All; `user={id}` and/or `gang={id}` (POST first, then GET) narrow the owner |

- Error `scope` is returned when the scope cannot be used (not a gang admin, not an admin account, banned/inactive user, `user` outside the gang).
- Gang/admin scope returns **other users' campaigns**; `api` and `postback.*` URLs in `list`/`get`/`set` always carry the **campaign owner's** API key.
- `list` and `get` return a bare array/object (no `status`); `set`, `del`, `block`, `postback`, `log`, `stats`, `clear`, `clone` return `{"status":"ok", …}`; `make` streams a file. Errors: `{"status":"error","error":"<code>"}`.
- HTTP status follows the error code; headers `X-API-Status` (`ok`/`error`) and `X-API-Error` (code) repeat the result:

| HTTP | `error` |
|---|---|
| 200 | success |
| 400 | `request`, `scope`, and every other function-level code (`no-site`, `no-domain`, `user-required`, `limit`, `no-list`, `bad-list`, `bad-ip`, `bad-size`, `bad-type`, `bad-value`, `failed`, `no-item`, `bad-item`, `bad-from`, `bad-to`, `bad-user`, `bad-gang`, `bad-site`, `bad-result`, `bad-ua32`, `bad-geo`) |
| 401 | `key`, `auth` |
| 402 | `unpaid` (when paid subscriptions are enabled) |
| 403 | `access` (`clear` under `scope=gang`), `ban` |
| 404 | `no-id`, `bad-id`, `bad-status` (postback) |
| 406 | `dublicate` (framework mapping; no filter function returns it) |
| 418 | `bot` (suppressed bot User-Agent) |
| 500 | `func`, `app`, `db` |

---

### filter/list — List campaigns

```
GET api/filter/list.json?id={token}[&scope=user|gang|admin][&user={id}][&gang={id}]
```

Only the scope parameters (`scope`, `user`, `gang`; see Auth & conventions). Returns all campaigns in the scope: own in `user` scope, gang members' in `gang` scope, everyone's (optionally narrowed by `user`/`gang`) in `admin` scope. No pagination, no search, no ordering guarantee; an empty result is `[]`.

**Response (success):** JSON array (no envelope), one object per campaign:

| Field | Type | Description |
|---|---|---|
| `id` | int | Campaign ID |
| `user` | int | Campaign owner user ID |
| `api` | string | Cloaking URL: `https://{host}/fltr/{owner}-{owner key}-{id}` (contains the owner's API key) |
| `domain` | string | Campaign domain |
| `mode` | int | 0=Auto, 1=Paused, 2=Disabled, 3=Train-white, 4=Train-black |
| `stats.click` | int | Total visits (white + black) |
| `stats.black` | int | Allowed visits (target shown) |
| `stats.white` | int | Blocked visits (dummy shown) |
| `stats.leads` | int | Leads count |
| `stats.approve` | int | Approved leads |
| `stats.cancel` | int | Cancelled leads |
| `stats.cash` | float | Earnings |
| `stats.cr` | float | Conversion rate % |
| `stats.epc` | float | Earnings per click (cash is truncated to an integer before the division) |
| `stats.appr` | float | Approval rate % |

**Errors:** `scope`.

---

### filter/get — Get campaign details

```
GET api/filter/get.json?id={token}&uid={campaign_id}
GET api/filter/get.json?id={token}&domain={host}
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `uid` | int | one of | Campaign ID. When non-zero the lookup is by ID only and `domain` is ignored. |
| `domain` | string | one of | Campaign name, compared for **exact equality** with the stored `domain` (not parsed as a URL). If several campaigns share the name, the highest ID wins. POST takes priority over GET. |
| `scope`, `user`, `gang` | — | no | Scope parameters. |

**Response (success):** Single object with all editable fields + stats + postback URLs:

- Identity: `id`, `user` (owner ID), `api` (with the owner's key), `domain`, `mode`
- Filtering: `geo[]`, `nogeo[]`, `langs[]`, `nolangs[]`, `os[]`, `browser[]`, `mobile` (int 0–4), `proto` (int 0–2), `ipmain` (bool), `lang` (bool), `adid` (bool), `trash` (int)
- Blacklists: `list[]` (list IDs), `wl[]` (IP whitelist, one entry per line), `uwl[]` (URL whitelist), `ubl[]` (URL blacklist)
- URLs: `white` (dummy URL/file), `black` (target URL/file), `mwhite` (int 0–9), `mblack` (int 0–9)
- Journal: `log` (int 0–3: 0 log all visits, 1 only target visits, 2 only dummy visits, 3 none)
- Schedule: `wd1`..`wd7` (bool, Monday..Sunday), `fh`, `fm` (int, window start hour/minute), `th`, `tm` (int, window end hour/minute), `tz` (string, IANA time-zone name or `""`)
- Stats: `stats` (same shape as in `list`)
- Postbacks: `postback.new`, `postback.approve`, `postback.paid`, `postback.cancel` — URL templates for `filter/postback` (contain the owner's key)

Empty list-type fields are returned as `[]`.

**Errors:** `scope`, `no-site` (campaign not found in the scope), `no-domain` (neither uid nor domain provided)

---

### filter/set — Create or update campaign

```
POST api/filter/set.json?id={token}[&scope=user|gang|admin][&user={id}]
```

Parameters merged from GET + POST (POST wins). If `uid` is absent and `domain` doesn't match any existing campaign in the scope, a new one is created (and the other supplied fields are applied to it in the same call). A new campaign is owned by the scope's `user`; in `gang`/`admin` scope pass `user={id}` or the call fails with `user-required`. A create is also refused with `limit` when the owner's plan allows no more campaigns (checked after `user-required`). Only the fields present in the request change.

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `uid` | int | Existing campaign ID (required for update if using uid); not found in the scope → `no-site` |
| `domain` | string | Campaign name (not parsed as a URL). Required when `uid` absent. POST-first. When it differs from the stored name it renames the campaign, also when `uid` is given. |
| `mode` | int | 0=Auto, 1=Paused, 2=Disabled, 3=Train-white, 4=Train-black |
| `ipmain` | bool | Check only primary IP |
| `lang` | bool | Enable language vs country check |
| `adid` | bool | Require ad click-id parameter |
| `trash` | int | First N visits forced to white (warm-up) |
| `log` | int | Visit-journal mode: 0=log all visits, 1=only target visits, 2=only dummy visits, 3=do not log |
| `mobile` | int | 0=Any, 1=Mobile only, 2=Desktop only, 3=iOS only, 4=Mobile non-iOS |
| `proto` | int | 0=Any, 1=IPv4 only, 2=IPv6 only |
| `geo` | array/string | ISO country codes allowlist. `ww`/`zz` = reset to empty. |
| `nogeo` | array/string | ISO country codes denylist (`ww`/`zz` = reset to empty) |
| `langs` | array/string | Accept-Language codes allowlist |
| `nolangs` | array/string | Accept-Language codes denylist |
| `os` | array/int | OS IDs (non-zero integers kept): 1=Win, 2=macOS, 3=Linux, 4=iOS, 5=Android, 6=WinMobile, 7=Symbian, 8=BlackBerry |
| `browser` | array/int | Browser IDs (non-zero integers kept): 1=Chrome, 2=Firefox, 3=IE, 4=Safari, 5=Opera, 6=Edge, 7=Facebook in-app, 8=Instagram in-app, 9=TikTok in-app, 10=WebView, 11=Bot |
| `wl` | text | IP whitelist (multiline: one address or `a.b.c.d/N` network per line) |
| `uwl` | text | URL whitelist (one substring per line; comma = AND) |
| `ubl` | text | URL blacklist (same format as `uwl`) |
| `white` | string | Dummy URL or filename |
| `black` | string | Target URL or filename |
| `mwhite` | int | Dummy display mode: 0=302, 1=include, 2=iframe, 3=curl, 4=WordPress, 5=301, 6=meta-refresh, 7=curl+base, 8=blank, 9=JS |
| `mblack` | int | Target display mode (same values) |
| `list` | array/csv | Blacklist IDs to attach; IDs outside the campaign owner's global, personal and gang lists are dropped silently; empty clears the selection (default lists apply) |
| `wd1`..`wd7` | bool | Schedule weekday flags, Monday..Sunday |
| `fh`, `fm` | int | Schedule window start hour (0–23) and minute (0–59) |
| `th`, `tm` | int | Schedule window end hour (0–23) and minute (0–59); the window is active only when an hour or minute is non-zero |
| `tz` | string | Schedule time zone (IANA name); an unknown name resets to `""` (server time) |

Bool-ish fields (`ipmain`, `lang`, `adid`, `wd1`..`wd7`) are saved as 1 for any truthy value, otherwise 0 (`0`, `"0"`, `""`). Integer fields are cast without range validation. There are no `ignore` or `bwl` parameters.

**Response (success):** `{"status":"ok","id":{campaign_id},"api":"{cloaking_url}"}` (the same body when the call carried no editable fields).

**Errors:** `scope`, `no-site` (uid provided but not found in the scope), `no-domain` (neither uid nor domain), `user-required` (create without a scope `user`), `limit` (create only: the owner's plan allows no more campaigns), `db`

---

### filter/del — Delete campaign

```
POST api/filter/del.json?id={token}&uid={campaign_id}[&scope=user|gang|admin]
```

**Parameters:** `uid` or `domain` (same lookup as `get`); plus the scope parameters. Only campaigns inside the scope can be deleted.

**Response (success):** `{"status":"ok","id":{campaign_id}}`

**Errors:** `scope`, `no-site` (not found in the scope), `no-domain`, `db`

**Side effects:** Deletes campaign + purges its visit journal (the raw visit rows and the condensed journal rows), which `filter/log` and `filter/stats` read.

---

### filter/make — Download integration index.php

```
GET api/filter/make.json?id={token}&uid={campaign_id}[&scope=user|gang|admin]
```

**Parameters:** `uid` or `domain` (same lookup as `get`); plus the scope parameters.

**Response (success):** File download (`Content-Disposition: attachment;filename=index.php`; no explicit `Content-Type`). The PHP file POSTs `$_SERVER` to the `/fltr/` endpoint and dispatches visitors. The body is the file itself whatever format suffix was requested.

**Note:** The campaign **owner's** API key is embedded in plaintext in the downloaded file, also when a gang admin or an admin downloads it.

**Errors (JSON, HTTP 400):** `scope`, `no-site`, `no-domain` — returned as JSON before file streaming.

---

### filter/block — Add item to blacklist

```
POST api/filter/block.json?id={token}[&scope=user|gang|admin][&user={id}][&gang={id}]
```

Appends one item to an existing blacklist. Which lists the caller may modify depends on the scope:

| Scope | Lists allowed |
|---|---|
| `user` | Personal lists of the key owner (system and gang lists are refused) |
| `gang` | The gang's lists, plus the personal lists of the member given in `user` |
| `admin` | Any list; with `user` and/or `gang` the list owner must match |

A missing, not permitted or wrong-kind list gives `bad-list`. There is no check of list status or of locked (feed/mirror) lists. To create, configure or edit lists and entries, and to get list IDs, use the `blacklist` API (`references/blacklist.md`); `filter/get` shows the lists attached to a campaign in `list[]`.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `list` | int | yes | Target list ID |
| `ip` | string | for types 1/2/3 | IPv4 or IPv6 address. Must be public (no private/reserved). |
| `size` | int | for types 2/3 | Network mask. IPv4: 8–31. IPv6: 32–128. |
| `value` | string | for type 4 | Header value or regular expression; non-empty, valid UTF-8, at most 250 characters |
| `regexp` | bool | no | Treat `value` as a regular expression (type 4 only); an invalid pattern never matches at runtime |
| `white` | bool | no | Type 4 only: header **white-list** rule — the visit is blocked when the header does not match or is not sent (default: black rule, blocked on match) |
| `type` | int | for type 4 | Header ID, 1–255 (names: "Header IDs" in `references/blacklist.md`); an unknown ID is accepted but never matches |
| `comment` | int | no | Tag ID (1–9, not range-checked) |
| `scope`, `user`, `gang` | — | no | Scope parameters |

The list kind is the `type` of the list (see `references/blacklist.md`):
- `1` — IPv4 single address
- `2` — IPv4 network
- `3` — IPv6 network
- `4` — Header/UA signature

**Responses (success):**
- Type 1: `{"status":"ok","id":N,"ip":"1.2.3.4"}`
- Type 2: `{"status":"ok","id":N,"ip":12345678,"size":24}` (ip is the integer network start, masked by `size`)
- Type 3: `{"status":"ok","id":N,"ip":"2001:db8::","size":48}` (address masked by `size`)
- Type 4: `{"status":"ok","id":N,"value":"...","type":1,"regexp":false,"white":false}`

**Errors:** `scope`, `no-list` (`list` missing or 0), `bad-list` (not found, not permitted in the scope, or unsupported list type), `bad-ip`, `bad-size`, `bad-type` (type 4: `type` outside 1–255; checked first), `bad-value` (type 4: empty, non-UTF-8 or over-long `value`), `failed` (the item could not be stored, e.g. the same item is already in the list)

**Side effects:** inserts the item (no expiry) and recounts the list size.

---

### filter/postback — Record postback to mini-tracker

```
GET api/filter/postback.json?id={token}&uid={campaign_id}&status={status}[&payout={amount}]
```

Increments counters on a campaign. No lead row is stored and there is no deduplication. Use the URL templates from `filter/get` response. Works on the authenticated user's own campaigns only (or the `uu` user's); `scope` has no effect.

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `uid` | int | Campaign ID |
| `status` | string | See mapping below |
| `payout` | numeric | Earnings delta (positive adds, negative subtracts; approve group only; comma accepted as decimal separator) |

**Status mapping:**
- New: `lead`, `wait`, `process`, `hold`, `new` → `stats.leads` +1
- Approve: `approve`, `sale`, `approved`, `confirm`, `confirmed` → `stats.approve` +1, `stats.cash` + `payout`
- Cancel: `cancel`, `trash`, `cancelled`, `reject`, `fake` → `stats.cancel` +1

**Response (success):** `{"status":"ok","filter":{uid}}`

**Errors (HTTP 404):** `no-id` (`uid` missing or 0), `bad-id` (campaign not found or not owned), `bad-status`

---

### filter/log — Visit journal

```
GET api/filter/log.json?id={token}[&site={id}][&result={code}][&page=1][&show=50]
```

One page of the visit journal, newest first: one row per checked visit with the reason code, matched blacklist, geo, IP, device data, raw User-Agent, discovered IP list and captured request headers. Read-only; never triggers a check and never changes counters.

**Parameters** (GET or POST, POST wins; `scope` GET only):

| Name | Type | Default | Description |
|---|---|---|---|
| `page` | int | 1 | Page number (minimum 1) |
| `show` | int | 50 | Rows per page; below 1 or non-numeric → 50, above 1000 → 1000 |
| `site` | int | — | Only visits of this campaign (ownership of the scope still applies) |
| `list` | int | — | Only visits blocked by this blacklist |
| `result` | int | — | `-2` = target shown (`reason` 0), `-1` = any block (`reason` ≠ 0), any other number = exactly that reason code (see Filter reason codes). Non-numeric is ignored |
| `ip` | string | — | Visitor IP; matched through a 32-bit hash (IPv4 exact, IPv6 hash — different addresses can collide). Invalid IP is ignored |
| `ua32` | int | — | `crc32` of the User-Agent, as returned in `ua32`; 0 is ignored |
| `geo` | string | — | Country code (lower-cased) |
| `scope`, `user`, `gang` | — | `user` | Scope parameters (visits are filtered by the campaign owner) |

**Response (success):**

```json
{
  "status": "ok",
  "data": {
    "15093": {
      "id": 15093, "user": 13, "site": 7, "time": 1760000000, "result": 3, "list": 0,
      "geo": "cn", "ip": "1.2.3.4", "mobile": false, "os": 1, "browser": 1,
      "ua": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...", "ua32": 3141592653,
      "ips": ["1.2.3.4"], "headers": { "Accept-Language": "zh-CN,zh;q=0.9" }
    }
  },
  "nav": { "total": 1, "page": 1, "pages": 1, "show": 50 }
}
```

`data` is an object keyed by row ID, in descending ID order; an empty page is `[]`. Row fields:

| Field | Type | Description |
|---|---|---|
| `id` | int | Journal row ID |
| `user` | int | Campaign owner |
| `site` | int | Campaign ID |
| `time` | int | Unix timestamp of the visit |
| `result` | int | Reason code 0–14; `0` = target shown (same as `reason` of `/fltr/`) |
| `list` | int | Blacklist that caused the block; `0` when none |
| `geo` | string | Country code, lower case; `zz` when unknown |
| `ip` | string | Visitor IP (IPv4 or IPv6) |
| `mobile` | bool | Mobile User-Agent |
| `os` | int | 0 unknown, 1 Windows, 2 macOS, 3 Linux, 4 iOS, 5 Android, 6 Windows Mobile, 7 Symbian, 8 BlackBerry |
| `browser` | int | 0 unknown, 1 Chrome, 2 Firefox, 3 MSIE, 4 Safari, 5 Opera, 6 Edge, 7 Facebook, 8 Instagram, 9 TikTok, 10 WebView, 11 bot |
| `ua` | string | Raw User-Agent; `""` once the raw row is purged |
| `ua32` | int | `crc32` of the User-Agent |
| `ips` | array | IP addresses found in the request headers; `[]` when none or purged |
| `headers` | object | Captured request headers `{ "<header name>": "<value>" }` (names as in "Header IDs" of `references/blacklist.md`; numeric key when unnamed; User-Agent is in `ua`); `[]` when none or purged |

`nav`: `total` rows matching the filters, `page`, `pages` = ceil(total / show), `show` effective page size. A page beyond the last returns empty `data` with the same `nav`.

**Errors (HTTP 400):** `scope`. Invalid filter values are ignored, not rejected (`filter/clear` takes the same filters and rejects an invalid one).

**Notes:**
- A visit appears only after the background job has condensed it into the journal; visits of paused/disabled campaigns and visits skipped by the campaign `log` mode are never present.
- Raw fields (`ua`, `ips`, `headers`) are purged after 30 days, the rows after 90 days (defaults), so old rows can come back with empty `ua`/`ips`/`headers`.

---

### filter/stats — Visit statistics

```
GET api/filter/stats.json?id={token}&item=date|site[&from=YYYY-MM-DD][&to=YYYY-MM-DD][&site={id}][&page=1][&show=50]
```

Aggregates the visit journal by day or by campaign over a date range: total, target (`black`) and dummy (`white`) visits and a per-reason breakdown with percentages. Read-only.

**Parameters** (GET or POST, POST wins; `scope` GET only):

| Name | Type | Required | Default | Description |
|---|---|---|---|---|
| `item` | string | yes | — | `date` (one row per calendar day) or `site` (one row per campaign) |
| `from` | string | no | `to` minus 6 days | First day, `YYYY-MM-DD` (real calendar date, else `bad-from`); after `to` → set to `to`. In the time zone of `dataset.tz` |
| `to` | string | no | today | Last day, inclusive, `YYYY-MM-DD` (else `bad-to`); empty or in the future → today (in the time zone of `dataset.tz`) |
| `site` | int | no | — | Only this campaign |
| `page` | int | no | 1 | Page number over the aggregated rows |
| `show` | int | no | 50 | Rows per page; below 1 → 50, above 1000 → 1000 |
| `scope`, `user`, `gang` | — | no | `user` | Scope parameters |

**Response (success):**

```json
{
  "status": "ok",
  "data": {
    "20260314": {
      "id": 20260314, "name": "2026-03-14",
      "click": 200, "black": 150, "black_pct": 75.0, "white": 50, "white_pct": 25.0,
      "reason": { "1": 0, "2": 10, "3": 30, "4": 5, "5": 0, "6": 0, "7": 0, "8": 5, "9": 0, "10": 0, "11": 0, "12": 0, "13": 0, "14": 0 },
      "reason_pct": { "1": 0, "2": 5.0, "3": 15.0, "4": 2.5, "5": 0, "6": 0, "7": 0, "8": 2.5, "9": 0, "10": 0, "11": 0, "12": 0, "13": 0, "14": 0 }
    }
  },
  "nav": { "total": 1, "page": 1, "pages": 1, "show": 50 },
  "dataset": { "results": { "0": "Site shown successfully", "1": "Request to invalid path or domain", "...": "..." }, "tz": "Europe/Moscow" }
}
```

| Field | Type | Description |
|---|---|---|
| `id` | int | `item=date`: day as integer `YYYYMMDD`; `item=site`: campaign ID |
| `name` | string | `item=date`: `YYYY-MM-DD`; `item=site`: campaign `domain`, or `"ID <id>"` when the campaign is gone or outside the scope |
| `user` | int | `item=site` only: campaign owner (`0` when the campaign was not found) |
| `click` | int | All logged visits of the item in the range |
| `black` | int | Visits with `reason` 0 (target shown) |
| `black_pct` | float | `black / click * 100`, one decimal |
| `white` | int | Visits with a non-zero reason (`click - black`) |
| `white_pct` | float | `white / click * 100`, one decimal |
| `reason` | object | Visit counts per reason code; keys `1`..`14` always present, zero-filled |
| `reason_pct` | object | Same keys: `count / click * 100`, one decimal |

Rows are keyed by `id`; `item=date` newest day first, `item=site` by campaign name ascending; days/campaigns without visits in the range are absent; a page beyond the end gives `data` = `[]`. `nav` has the same shape as in `filter/log`. `dataset.results` maps reason codes `0`..`14` to labels in the authenticated user's language; `dataset.tz` is the IANA name of the time zone the days are in (see Time zone below).

**Errors (HTTP 400):** `scope`, `no-item` (`item` missing/empty), `bad-item` (not `date`/`site`), `bad-from`, `bad-to`.

**Notes:**
- Counts come from the visit journal, so they differ from the campaign counters of `list`/`get`: the journal honours the campaign `log` mode, omits visits of paused/disabled campaigns (the counters include them), includes checks made by a flow bound to the campaign (the counters do not), and is cut by retention (90 days by default).
- **Time zone.** Days, `from`, `to` and «today» are computed in the calling user's own time zone (`tz` of the profile, `profile/edit`) when the install's per-user time zones setting is on, the stored zone is a valid IANA name and the database server knows it; otherwise in the install's time zone. `dataset.tz` always names the zone used. With `uu` the zone is the acting user's.
- The date range length is not limited.

---

### filter/clear — Clear the visit journal

```
POST api/filter/clear.json?id={token}[&scope=user|admin][&user={id}][&gang={id}][&site=][&list=][&result=][&ip=][&ua32=][&geo=]
```

Deletes the rows of the visit journal that `filter/log` lists for the same filters (`site`, `list`, `result`, `ip`, `ua32`, `geo`), so a client clears exactly what it can read. **No filter at all clears the whole scope.** Only the journal rows are deleted: campaign counters of `list`/`get` are not changed, and a cleared visit does not come back (a visit still in the background queue shows up afterwards).

**Parameters** (GET or POST, POST wins; `scope` GET only). A filter that is sent must be read exactly as sent, otherwise the call is refused and nothing is deleted:

| Name | Type | Description |
|---|---|---|
| `site` | int | Only visits of this campaign (ownership of the scope still applies, so a foreign campaign matches nothing). Digits only, else `bad-site`; `0` or empty = no filter |
| `list` | int | Only visits blocked by this blacklist. Digits only, else `bad-list`; `0` or empty = no filter |
| `result` | int | `-2` = target shown (reason 0), `-1` = any block (reason ≠ 0), any other integer = exactly that reason code. Not an integer → `bad-result` |
| `ip` | string | Visitor IP, matched through a 32-bit hash as in `log` (IPv4 exact, IPv6 hash — a colliding address is deleted too). Not an IP → `bad-ip`; empty = no filter |
| `ua32` | int | `crc32` of the User-Agent. Digits only, else `bad-ua32`; `0` or empty = no filter |
| `geo` | string | Two-letter country code, case-insensitive. Else `bad-geo`; empty = no filter |
| `scope`, `user`, `gang` | — | Scope parameters (see below) |

An empty value counts as not sent; an array value is refused with the `bad-*` code of its field.

| Scope | Deletes |
|---|---|
| `user` (default) | The caller's visits |
| `gang` / `team` | Nothing: `access` for a team admin (the team journal is not clearable — use `scope=user` for one's own visits), `scope` for anyone who is not a team admin |
| `admin` alone | Every visit of the install |
| `admin` + `user` / `gang` | That user's visits / the visits of the team's members (both: only the intersection) |

Under `scope=admin`, `user` and `gang` must be numbers, otherwise `bad-user` / `bad-gang` (a malformed value would otherwise read as «no owner» and widen the delete).

**Response (success):** `{"status":"ok","data":{"del":120}}` — `del` is the number of rows deleted (`0` when nothing matched).

**Errors:** `scope` (including a non-admin team member under `scope=gang`), `access` (HTTP 403: `scope=gang` by a team admin), `bad-user`, `bad-gang`, `bad-site`, `bad-list`, `bad-result`, `bad-ip`, `bad-ua32`, `bad-geo`, `db` (HTTP 500). The checks run before the delete, in this order.

---

### filter/clone — Copy a campaign

```
GET|POST api/filter/clone.json?id={token}&uid={campaign_id}&domain={new_name}[&scope=user|gang|admin][&user={id}][&gang={id}]
```

Makes a new campaign from an existing one: the source's settings under a new name, with the counters at zero. Unlike `filter/set`, it always creates a new campaign and never updates an existing campaign, so an already used name is allowed — keep the returned `id` to address the copy.

**Parameters** (GET or POST, POST wins; `scope` GET only):

| Name | Type | Required | Description |
|---|---|---|---|
| `uid` | int | yes | Source campaign ID, found in the scope. Missing, 0, unknown and foreign all give `no-site` |
| `domain` | string | yes | Name of the copy: plain text, tags stripped, **never parsed as a URL** (`https://a.b/c` is stored whole). Empty → `no-domain` |
| `scope`, `user`, `gang` | — | no | Scope parameters (they only select which sources are reachable) |

**The copy belongs to the source's owner in every scope**, and that owner's plan is counted for `limit` — whoever calls (a team admin or an administrator copying a member's campaign uses the member's plan). There is no `user-required`.

What is copied: all settings of the source, the schedule (`wd1`..`wd7`, `fh`, `fm`, `th`, `tm`, `tz`) and the visit-journal mode `log` included; `trash` is copied as the source's current value (it is a countdown of the warm-up visits left). The six counters (`stats.click`/`black`/`white`/`leads`/`approve`/`cancel`/`cash`) start at zero. The visits of the source stay with the source.

**Response (success):** `{"status":"ok","data":{…}}` — `data` is the new campaign exactly as `filter/get` returns it (wrapped in the envelope, unlike `get`), with `api` and `postback.*` built for the new ID and the owner's key:

```json
{
  "status": "ok",
  "data": {
    "id": 15, "user": 13, "api": "https://example.com/fltr/13-4a3bc9d-15", "domain": "landing-copy", "mode": 0,
    "geo": ["us", "ca"], "mblack": 0, "black": "https://offer.example.com/", "mwhite": 0, "white": "https://white-page.example.com/",
    "trash": 0, "proto": 0, "mobile": 0, "os": [], "browser": [], "log": 0, "ipmain": false, "lang": false, "adid": false,
    "wd1": true, "wd2": true, "wd3": true, "wd4": true, "wd5": true, "wd6": true, "wd7": true,
    "fh": 0, "fm": 0, "th": 0, "tm": 0, "tz": "", "list": [5, 12], "langs": [], "nolangs": [], "nogeo": [],
    "uwl": [], "ubl": [], "wl": [],
    "stats": { "click": 0, "black": 0, "white": 0, "leads": 0, "approve": 0, "cancel": 0, "cash": 0, "cr": 0, "epc": 0, "appr": 0 },
    "postback": {
      "new": "https://example.com/api/filter/postback.json?id=13-4a3bc9d&uid=15&status=new",
      "approve": "https://example.com/api/filter/postback.json?id=13-4a3bc9d&uid=15&status=approve",
      "paid": "https://example.com/api/filter/postback.json?id=13-4a3bc9d&uid=15&status=approve&payout={payout}",
      "cancel": "https://example.com/api/filter/postback.json?id=13-4a3bc9d&uid=15&status=cancel"
    }
  }
}
```

**Errors:** `scope`, `no-site` (`uid` missing or not a campaign of the scope), `no-domain` (`domain` missing or empty), `limit` (the owner's plan allows no more campaigns: a lapsed plan or the campaign count at the plan's limit; an install without paid subscriptions has no limit), `db` (HTTP 500). Checked in this order, before anything is written.

---

## /fltr/ — Runtime Cloaking Endpoint

**COMPLETELY DIFFERENT** from `/api/filter/`. This is the endpoint your `index.php` wrapper calls to check each visitor.

### URL format

```
POST /fltr/{user_id}-{api_key}-{campaign_id}
```

- `user_id` — numeric user ID
- `api_key` — user's API token (hex, lowercase, no special chars)
- `campaign_id` — numeric campaign ID
- All three parts separated by single dashes; the API key must not contain dashes
- No format suffix (`.json`); anything after the token is ignored. `/clo/{user_id}-{api_key}-{campaign_id}` is an equivalent alias of the same endpoint.

### Auth

- Authentication is embedded in the URL path — **no `?id=` parameter**.
- Banned user → `access-denied`.
- Wrong API key → `access-denied`.
- Campaign owned by a different user → `access-denied`.

### Request body

**MUST be `x-www-form-urlencoded`** — JSON body is NOT supported here. POST the entire `$_SERVER` superglobal as form-encoded data.

```php
curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($_SERVER));
```

### Response (success)

```json
{
  "status": "ok",
  "result": "allow",
  "action": "allow",
  "type": "black",
  "url": "https://offer.example.com/",
  "ip": "1.2.3.4",
  "geo": "us",
  "reason": 0,
  "target": "https://offer.example.com/",
  "dummy": "https://white-page.example.com/"
}
```

| Field | Description |
|---|---|
| `result` | Real filter decision: `allow` (target) or `deny` (dummy). In training modes may differ from `action`. |
| `action` | Decision after training mode override — what the wrapper should use. |
| `type` | `black` (target) or `white` (dummy). Use this to dispatch in your PHP wrapper. |
| `url` | URL to serve (macro-expanded). |
| `ip` | Resolved visitor IP (`false` when none could be resolved). |
| `geo` | 2-letter country code (lowercase), `zz` when unknown. |
| `reason` | Code of the real check (integer), also in training modes. 0 = allowed (also in modes 1/2, where no check runs). See reason codes below. |
| `target` | Target URL regardless of decision (macro-expanded). |
| `dummy` | Dummy URL regardless of decision (macro-expanded). |

**URL macros:** `{geo}`, `{geoup}` (uppercase), `{ip}` are replaced in both `target` and `dummy` before returning.

**Headers:** `Content-Type: application/json` and `X-Action: allow|deny` (the `action` value; `deny` on errors). The body is pretty-printed JSON.

### Response errors

| `error` | Meaning |
|---|---|
| `bad-request` | URL token malformed (any of the three parts missing/zero) |
| `access-denied` | Banned user, wrong API key, or site belongs to different user |
| `bad-site` | Campaign not found |
| `no-data` | POST body is empty |

All `/fltr/` errors use HTTP 200 (this endpoint only; `/api/filter/*` maps errors to HTTP statuses). No standard `/api/` error codes apply here.

### Filter reason codes

The same codes appear as `reason` here and as `result`/`reason` in `filter/log` and `filter/stats`.

| Code | Meaning | Counter |
|---|---|---|
| 0 | Allowed | `stats.black` +1 |
| 1 | Invalid path (junk request path such as favicon, source maps, `ads.txt`); the host is not compared with the campaign `domain` | `stats.white` +1 |
| 2 | Wrong device type (device, OS or browser mismatch) | `stats.white` +1 |
| 3 | Blocked country | `stats.white` +1 |
| 4 | IPv4 blacklisted | `stats.white` +1 |
| 5 | IPv4 network blacklisted | `stats.white` +1 |
| 6 | IPv6 blacklisted | `stats.white` +1 |
| 7 | Header: black-list rule matched, white-list rule not matched or header missing, or no User-Agent | `stats.white` +1 |
| 8 | Bot / preview / spy tool / no usable IP | `stats.white` +1 |
| 9 | Wrong IP protocol | `stats.white` +1 |
| 10 | Wrong language | `stats.white` +1 |
| 11 | Trash: warm-up visit (`trash` counter) or probing path (`robots.txt`, `.env`, `/.well-known/`, …) | `stats.white` +1 |
| 12 | URL blacklisted (or URL whitelist miss) | `stats.white` +1 |
| 13 | Missing ad click-id | `stats.white` +1 |
| 14 | Visit outside the campaign schedule (`wd1`..`wd7`, `fh`/`fm`–`th`/`tm`, `tz`) | `stats.white` +1 |

### Check order (first match wins)

1. Mode 1 (Paused) → deny, mode 2 (Disabled) → allow; neither runs any check or writes a journal row.
2. IP whitelist (`wl`) match → allow (reason 0).
3. Preview/translator/spy markers (`HTTP_X_PURPOSE`, `HTTP_X_FB_HTTP_ENGINE`, `HTTP_VIA`, `HTTP_REFERER`), no IP or `0.0.0.0` → reason 8.
4. Junk path → reason 1; probing path → reason 11.
5. No User-Agent → 7; unrecognised OS or bot User-Agent → 8; device (`mobile`), OS, browser allowlists → 2.
6. Protocol (`proto`) → 9.
7. `nolangs` → 10; geo allowlist/denylist → 3; language (`lang`, `langs`) → 10.
8. Ad click-id (`adid`) → 13.
9. URL whitelist/blacklist (`uwl`, `ubl`) → 12.
10. Blacklists: IPv4 → 4, IPv4 network → 5, IPv6 → 6, header rules → 7.
11. Trash counter (`trash`) → 11.
12. Schedule → 14.
13. Otherwise allow (reason 0).

Header rules (list kind 4): a black rule blocks when the header matches (case-insensitive substring, or PCRE when `regexp`); a white rule (`white`) blocks when the header is missing or does not match. An invalid regular expression is skipped (a white regexp rule with an invalid pattern never blocks).

Schedule: weekday flags `wd1`..`wd7` (1 = Monday), window `from`–`to` as hour/minute in the campaign time zone `tz` (server time when empty); `from` later than `to` is an overnight window. No schedule is active when all seven days are on and neither `from` nor `to` is set.

### Visit journal

Each check writes one journal row (what `filter/log` and `filter/stats` return) unless the campaign `log` mode suppresses it: `0` log all visits, `1` only target (allowed) visits, `2` only dummy (blocked) visits, `3` nothing. Modes 1 and 2 (Paused/Disabled) never write a row. Rows become visible in the API after a background job condenses them.

### Campaign modes

| Mode | Runs filter? | `action` result | Counter effect |
|---|---|---|---|
| 0 (Auto) | Yes | Real decision | Real decision |
| 1 (Paused) | No | deny | `stats.white` +1 |
| 2 (Disabled) | No | allow | `stats.black` +1 |
| 3 (Train-white) | Yes | forced deny | Based on real check |
| 4 (Train-black) | Yes | forced allow | Based on real check |

In training modes, `result` and `reason` reflect the real check, `action`/`type`/`url` reflect the training override. Counters are incremented in every mode (modes 1/2 included); the journal row is written only for checks that actually ran and pass the campaign `log` mode.

---

## Key Headers Checked by the Filter

The POST body may include any `$_SERVER` key. The filter specifically reads:

`HTTP_USER_AGENT`, `HTTP_ACCEPT_LANGUAGE`, `HTTP_REFERER`, `HTTP_HOST`, `HTTP_X_FORWARDED_FOR`, `HTTP_CF_CONNECTING_IP`, `HTTP_CF_CONNECTING_IPV6`, `HTTP_CLIENT_IP`, `HTTP_X_REAL_IP`, `HTTP_X_PURPOSE`, `HTTP_X_FB_HTTP_ENGINE`, `HTTP_VIA`, `REMOTE_ADDR`, `REQUEST_URI`, `QUERY_STRING`, `REQUEST_SCHEME`

Plus all `HTTP_SEC_FETCH_*`, `HTTP_PRIORITY`, `HTTP_X_FRAME_OPTIONS`, `HTTP_X_WAP_PROFILE` and other headers (the full name ↔ header ID list is "Header IDs" in `references/blacklist.md`). Hyphenated names such as `user-agent` are remapped to `HTTP_USER_AGENT`.

`HTTP_HOST` (or `HTTP_X_FORWARDED_HOST`) with `REQUEST_URI` forms the URL matched against the URL whitelist/blacklist; the host is **not** compared with the campaign `domain`.
