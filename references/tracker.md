# Tracker, Gangs & Click API Reference

## Table of Contents
1. Click API (TDS) — `/track-tds/`, `/track-dom/`, `/track-pxl/`, `/track-thx/`
2. tracker/postback — Receive lead postback
2a. Owner scope — `scope` of stats, leads, lead, clicks, spend
3. tracker/stats — Click + lead statistics
4. tracker/leads — Leads list
4a. tracker/lead — One lead with its status changes
5. tracker/clicks — Raw click list (traffic log)
6. tracker/spend — Import click spend
7. tracker/templates — CPA-network postback templates
8. tracker/networks — Traffic-source presets
9. tracker/settings — Tracker configuration
10. gangs/* — Team-scope wrappers
11. Internal endpoints — checkdom, domains, client

---

## 1. Click API (TDS)

The click API is a two-tier system:
- **`server.php`** (installed on client's domain) receives visitor HTTP requests and proxies them to the tracker via `cURL POST`.
- **Tracker** receives the POST, runs TDS logic, returns JSON with display instructions.
- The **client library** included by `server.php` provides the `tds()` and `api()` functions.

### Endpoint: GET /track-tds/{flow_code}

Called by `server.php` to process a visit via flow symbolic code.

**POST body** (sent by server.php): all `$_SERVER` keys (form-encoded) + `cookie` = `$_COOKIE`.

Removed from `$_SERVER` before sending: `PHP_SELF`, `SCRIPT_FILENAME`, `GATEWAY_INTERFACE`, `SERVER_ADDR`, `SERVER_NAME`, `SERVER_SOFTWARE`, `SERVER_PROTOCOL`, `SERVER_SIGNATURE`, `SERVER_ADMIN`, `SERVER_PORT`, `REQUEST_TIME`, `REQUEST_TIME_FLOAT`, `SCRIPT_NAME`, `PATH_TRANSLATED`, `PATH_INFO`, `ORIG_PATH_INFO`

**Key server fields used by TDS:**

| Field | Usage |
|---|---|
| `cookie` | Visitor cookies array |
| `QUERY_STRING` | UTM tags, subid, cpc, next (encoded click ID), data fields |
| `HTTP_USER_AGENT` | Browser, OS, mobile detection, bot check |
| `HTTP_HOST` / `HTTP_X_HOST` / `HTTP_X_FORWARDED_HOST` | Hostname |
| `HTTP_REFERER` | Bot detection (toloka, adheart, yandex-team, modadvert) |
| `HTTP_CF_CONNECTING_IP` / `HTTP_X_REAL_IP` / `HTTP_CLIENT_IP` / `REMOTE_ADDR` | Client IP |
| `HTTP_X_PURPOSE` | If "preview" → bot |
| `HTTP_X_FB_HTTP_ENGINE` | If "liger" → bot |
| `HTTP_VIA` | If "translate.google.com" → bot |

**QUERY_STRING parameters extracted:**

| Param | Description |
|---|---|
| `next` | Encoded click ID — second step |
| `utm_source` | UTM Source |
| `utm_campaign` | UTM Campaign |
| `utm_content` | UTM Content |
| `utm_term` | UTM Term |
| `utm_medium` | UTM Medium |
| `subid` | Sub ID |
| `customer` / `utm_user` | Customer tag |
| `data1`–`data5` / `d1`–`d5` | Custom data |
| `fbclid`, `gclid`, `adclid`, `clid`, `adid` | Ad click ID |
| `cpc` | Click cost |
| `cur` | Currency (default `usd`) |

**Success response:**

```json
{
  "status": "ok",
  "uid": "ef1c",
  "way": 42,
  "method": "http302",
  "url": "https://offer.example.com/?click={click}",
  "nocookie": false,
  "params": {
    "click": "ca1337",
    "flow": "ef1c",
    "user": "7",
    "host": "example.com",
    "geo": "us",
    "query": "utm_source=fb",
    "next": "https://example.com/ef1c?next=abc",
    "local": "?next=abc",
    "direct": "https://offer.com/step2",
    "adid": "fbclid123",
    "data1": "", "data2": "", "data3": "", "data4": "", "data5": "",
    "subid": "",
    "customer": "",
    "utm_source": "fb",
    "utm_campaign": "",
    "utm_content": "",
    "utm_term": "",
    "utm_medium": "",
    "hide1": "", "hide2": "", "hide3": "", "hide4": "", "hide5": "", "hide6": ""
  }
}
```

**Display methods** (`method` field):

| Value | Description |
|---|---|
| `http302` | 302 temporary redirect |
| `http301` | 301 permanent redirect (browser-cached) |
| `redirect` | Meta-redirect HTML page with 1s delay |
| `jsgo` | JS `location.replace()` after 500ms |
| `iframe` | Full-screen iframe |
| `proxy` | cURL fetch + `<base>` tag rebasing |
| `curl` | cURL fetch without rebasing |
| `local` | Include local PHP/HTML file |
| `site` | File from site storage (`index.html`/`index.php`) with rebasing |
| `error` | HTTP error page (code from `url`) |
| `blank` | Empty 200 response |
| `raw` | Body from `data` field, Content-type from `type` field |

**Client sets cookies** (if `nocookie != true`):
- `{uid}` = way ID, 86400s (1 day)
- `x{uid}` = encoded click ID, 2592000s (30 days). SameSite=None; Secure.

**Error response:**

```json
{"status":"error","error":"bad-flow"}
```

| Error | Reason |
|---|---|
| `no-id` | Flow UID not provided |
| `bad-id` | UID doesn't decode to numeric ID |
| `bad-flow` | Flow not found |
| `no-ways` | Flow has no active ways |
| `no-data` | POST body empty |
| `db` | Error writing click to DB |

**Click filter codes** (the `filter` field of a stored click, as returned by `tracker/clicks`):

| Code | Reason |
|---|---|
| 1 | Bot (by User-Agent) |
| 2 | IP/UA blacklisted |
| 3 | Geo not in allowed list |
| 4 | Browser language doesn't match geo |
| 5 | Extended flow filter |
| 6 | No suitable way found |
| 7 | Missing adid |

When filtered, trafficback URL is returned (flow's "dump" setting).

### Endpoint: GET /track-dom/{domain}

Same as `track-tds` but looks up the flow by parked domain. If the URL path contains a filename, serves domain verification files instead.

**Additional errors:** `no-domain`, `bad-path`

### Endpoint: GET /track-pxl/{flow_code} — Pixel on event
### Endpoint: GET /track-thx/{flow_code} — Pixel on approval

Called via `/{uid}/pixel`, `/{uid}/pixel.gif`, `/{uid}/pixel.png`, `/{uid}/pixel.json`, or `/{uid}/thanks.*`, `/{uid}/success.*`.

`track-thx` automatically sets `status=approve`.

**POST body:** `cookie` (visitor cookies) + `get` (QUERY_STRING value).

Cookie `x{uid}` contains the encoded click ID.

Postback fields from `get`: `uid`, `status`, `goal`, `price`, `currency`, `convert`, `name`, `phone`, `email`, `comment`, `meta`, UTM tags, `subid`, `customer`, `way`, `site`, `ip`, `geo`, `ua`, `stn`, `sth`, `sta`, `stc`, `stt`, `adid`, `data1`–`data2`.

**Response:** 1×1 GIF (`.gif`), 1×1 PNG (`.png`), or JSON (`.json`). HTTP 200 on success, 418 on lead error, 404 on critical error.

### Macros Available in URLs and Content

| Macro | Value |
|---|---|
| `{click}` | Alphanumeric click identifier |
| `{flow}` | Flow code |
| `{user}` | User ID |
| `{geo}` | ISO country code |
| `{host}` | Domain the visitor came from |
| `{query}` | GET parameter string |
| `{next}` | Full second-step link |
| `{local}` | Relative second-step link |
| `{direct}` | Direct second-step link without redirect |
| `{adid}` | Ad click ID |
| `{data1}`–`{data5}` | Custom data fields |
| `{subid}` | SubID tag |
| `{customer}` | Customer tag |
| `{utm_source}`, `{utm_campaign}`, `{utm_content}`, `{utm_term}`, `{utm_medium}` | UTM tags |
| `{esub}` | Encrypted SubID |
| `{eutms}`, `{eutmc}`, `{eutmn}`, `{eutmt}`, `{eutmm}`, `{eutmu}` | Encrypted UTM tags |
| `{ed1}`–`{ed5}` | Encrypted data fields |
| `{hide1}`–`{hide6}` | Hidden flow/way data fields |

---

## 2. tracker/postback — Receive lead postback

```
GET|POST api/tracker/postback.json?id={token}
GET|POST api/gangs/postback.json?id={token}
POST /postbacks/{flow_key}
```

The `/postbacks/{flow_key}` route decodes the key: positive value = user postback, negative = gang postback. The `{flow_key}` is the value from `flow.postback.*` URL templates.

**Method:** GET or POST (parameters are merged; POST wins).

**Required fields (at least one identification combo):**

| Combo | Fields needed |
|---|---|
| By click | `click` (symbolic click code) |
| By subid+flow | `flow` (symbolic) + `subid` |
| By uid+flow | `flow` (symbolic) + `uid` (advertiser's lead ID) |

**All parameters:**

| Name | Type | Description |
|---|---|---|
| `click` | string | Click code. If negative, treated as way ID. |
| `flow` | string | Flow code. Fallback when no click. |
| `uid` | string | Advertiser-side lead ID (dedup key) |
| `subid` | string | SubID (dedup key) |
| `way` | int | Way ID (also can be `click=-{way}`) |
| `site` | int | Site ID (flow-fallback create only) |
| `ip` | string | Lead IP. Falls back to click IP. |
| `geo` | string | 2-letter country code |
| `ua` | string | User-Agent |
| `status` | string | Advertiser status string. Mapped to internal stages. |
| `goal` | string | Goal code (configured in tracker settings) |
| `convert` | string | ISO currency to convert `price` to |
| `costconv` | string | ISO currency to convert `cost` to |
| `currency` | string | Price currency (default `usd`) |
| `costcur` | string | Cost currency (default `usd`) |
| `price` | decimal | Lead payout (overrides existing) |
| `addprice` | decimal | Add to existing lead price |
| `cost` | decimal | Lead cost (overrides existing) |
| `addcost` | decimal | Add to existing lead cost |
| `name`, `phone`, `email`, `comment` | string | Customer fields |
| `meta` | object | Custom meta fields (POST only, must be object) |
| `adid` | string | Ad-network click ID |
| `data1`–`data5` | string | Custom data slots |
| `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer` | string | Override inherited UTM/tag values |
| `stn`, `sth`, `sta`, `stc`, `stt` | string (csv) | Map advertiser status strings to new/hold/approve/cancel/trash |
| `create` | "1" | Force creation of new lead instead of dedup-update |

**Success response:**
```json
{"status":"ok","id":1337,"create":true,"stage":"new"}
```

`stage` ∈ `fail`, `new`, `hold`, `approve`, `cancel`, `trash`

**Errors:** `no-click`, `bad-click`, `bad-flow`, `access`, `db`, `no-changes`

---

## 2a. Owner scope — stats, leads, lead, clicks, spend

`tracker/stats`, `tracker/leads`, `tracker/lead`, `tracker/clicks` and `tracker/spend` take whose data they work with from the `scope` parameter, like the platform's other API apps. `postback`, `settings` and the other tracker functions do not use it. The `gangs/*` versions of these five functions are the same functions with `scope=gang` forced (section 10).

```
api/tracker/{func}.json?id={token}[&scope=user|gang|admin][&user={id}][&gang={id}]
```

| `scope` | Who may use it | Whose data | `user` | `gang` |
|---|---|---|---|---|
| `user` (default) | any authenticated caller | the caller's own | ignored | ignored |
| `gang` (alias `team`) | admin of the caller's own team | the whole team | optional: narrows to one member of the team; `0` or empty = the whole team | ignored — the team is always the caller's own |
| `admin` (aliases `sys`, `system`, `all`) | admin accounts only | every owner | optional: that user | optional: that team; combined with `user` by AND |

- `scope` is read from the **query string only**; one sent in a POST or JSON body is ignored (the request is then a user-scope one). The word is case-insensitive.
- An absent, empty or **unknown** `scope` word means `user`: a mistyped word is not an error, it returns the caller's own data. A team admin who sends no `scope` gets their own data, not the team's.
- `user` and `gang` are read from POST first, then GET, as integers; absent, empty, non-numeric and `0` all mean "not given". They count only in `gang` and `admin` scope (`tracker/spend` has its own owner rules, section 6).
- Error `scope` (HTTP 400) is answered when: `scope=admin` and the caller is not an admin account; `scope=gang` and the caller has no team or is not an admin of it; `scope=gang` with a `user` that is not currently a member of the caller's team (an unknown user, a user of another team, a user without a team, a former member) — the answer is `scope`, not an empty list, and not `not-found` for `tracker/lead`; the effective caller (see `uu` below) is missing, banned or inactive.
- `uu`: a caller with an admin account who authenticates with `uu={id}` works as that user, and the scope is judged against that user: `scope=admin` needs the target to be an admin account, `scope=gang` needs the target to be a team admin.
- Plan check: after the scope check, `stats`, `leads`, `lead` and `clicks` check the plan of the effective caller and answer `payment-required` (HTTP 400, not 402) when it does not allow them — **not applied when the resolved scope is `admin`**. In `gang` scope the caller's own plan is checked. `spend` has no plan check.
- Order of answers: framework (`key`, `ban`) → `gangs/*` guard (`no-gang`, `access`) → `scope` → `payment-required` → parameter errors (`no-item`, `no-from`, `no-to`, `no-lead`) → data.

---

## 3. tracker/stats — Statistics

```
GET|POST api/tracker/stats.json?id={token}&item=date&from=2025-01-01&to=2025-01-31[&scope=user|gang|admin]
GET|POST api/gangs/stats.json?id={token}&item=date&from=2025-01-01&to=2025-01-31
```

**Method:** GET or POST (parameters are merged; POST wins). Accepts `form-urlencoded` or JSON body. `scope` itself is read from the query string only.

**Owner scope** (section 2a): `scope`, `user` and `gang` choose whose numbers are returned. `user` (default) — the caller's; `gang` — the whole team, `user=M` narrows to one member (`0`/empty = the whole team, a user outside the team → error `scope`); `admin` (admin accounts) — every owner, narrowed by `user` and/or `gang`. `api/gangs/stats` is the same function with `scope=gang` forced (team admins only; a `scope` sent on that URL is overridden). The plan check (`payment-required`) comes after the scope check and is skipped for `scope=admin`.

**`goal` and `event` objects need a single owner:** a `stats` row (and `total`) carries `goal` and `event` objects keyed by slot number `"1"`–`"5"` — there are no flat `goal1`/`event1` keys. The slots come from the goal and event settings of one user, so the objects exist only in user scope and with `user=M` in gang/admin scope; for the whole team or every owner (and for a user with no active goal or event) the key is absent from the rows and from `total`. Only the slots active in that user's settings are listed, each of them in every row (zeros when the row has no matching leads or events); leads on a switched-off goal are not shown. In `total` the objects are summed over the rows (`part` is not recalculated there).

**Pagination:** pass `show` (rows per page; `0`/empty → 100) and `page` (1-based). When `show` is present the `stats` map is cut to the requested page (string keys preserved); when absent, all rows are returned and `show = -1`. The response always echoes `page`, `show`, and `items` (total row count before slicing).

**Sorting:** runs after the stats map is loaded/totaled but **before** pagination slicing, so the full result set is sorted first and pages come out correctly ordered.

| Name | Description |
|---|---|
| `sort` | Column to sort by. Allowed values: `name`, `id`, `click`, `unique`, `spend`, `next`, `cn`, `ch`, `ca`, `cc`, `ct`, `cv`, `total`, `mt`, `mh`, `ma`, `mmt`, `mmh`, `mma`, `mct`, `mch`, `mca`, `ur`, `cr`, `crt`, `epc`, `epl`, `appr`, `app`, `per`, `cpc`, `cpa`, `cpv`, `cpl`, `profit`, `roi`. Unknown value → falls back to `name`. Absent → no sorting (natural insertion order from the loader). |
| `order` | `asc` or `desc`. Default `desc`; only the literal `asc` (case-insensitive) reverses it. |

Natural/string comparison is used only when sorting by `name`; all other columns compare numerically.

**Required parameters:**

| Name | Description |
|---|---|
| `item` | Grouping key: `date`, `hour`, `flow`, `way`, `site`, `filter`, `geo`, `mobile`, `os`, `browser`, `ip`, `utmd`, `subid`, `utms`, `utmc`, `utmn`, `utmt`, `utmm`, `utmu`, `user` |
| `from` | Date range start `YYYY-MM-DD` |
| `to` | Date range end `YYYY-MM-DD` |

**Optional filters:**

| Name | Description |
|---|---|
| `flow` | Symbolic flow code |
| `f` | Numeric flow ID |
| `way` | Way ID |
| `site` | Site ID |
| `filter` | Click filter result: 0=target, 1=bot, 2=blacklist, 3=country, 4=language, 5=filtering campaign. Use -1 for "any non-zero". |
| `hour` | Hour of day (0–23), timezone-aware |
| `mobile` | **Binary**: 0=desktop traffic, 1=mobile traffic. Different from the 0–4 scale used in campaign/way filter settings. |
| `os` | OS: 0=Other, 1=Windows, 2=macOS, 3=Linux, 4=iOS, 5=Android, 6=WindowsMobile, 7=Symbian, 8=BlackBerry |
| `browser` | Browser: 0=Other, 1=Chrome, 2=Firefox, 3=IE, 4=Safari, 5=Opera, 6=Edge, 7=Facebook, 8=Instagram, 9=TikTok, 10=WebView, 11=Bot |
| `geo` | Country code |
| `domain` | Domain (the `utmd` grouping value) |
| `subid` | SubID |
| `customer` | Customer (the `utmu` grouping value) |
| `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium` | UTM filters |
| `ip` | IP address |
| `cur`/`curr`/`currency` | Currency for money columns |
| `scope` | `user` (default) / `gang` (`team`) / `admin` (`sys`, `system`, `all`), query string only — see section 2a. Absent or unknown → `user`. A refused scope → error `scope` (HTTP 400). Forced to `gang` on `gangs/stats` |
| `user` | `scope=gang`: narrow to one member of the caller's team (a user outside the team → error `scope`; `0`/empty → the whole team). `scope=admin` (admin accounts): narrow to that user. `scope=user`: ignored |
| `gang` | `scope=admin` (admin accounts) only: narrow to that team, combined with `user` by AND. Ignored in `user` and `gang` scope |
| `sort` | Column to sort by (see the allowed values above); unknown → `name`. Absent → no sorting. |
| `order` | Sort direction `asc`/`desc`; default `desc`. |
| `show` | Rows per page (enables pagination; `0`/empty → 100) |
| `page` | 1-based page number (with `show`) |

**Success response** (pagination echo fields `page`/`show`/`items` always present):
```json
{
  "status": "ok",
  "page": 1, "show": 100, "items": 37,
  "stats": {
    "20250101": {
      "click": 100, "unique": 80, "next": 10, "spend": 5.0,
      "cn": 10, "ch": 2, "ca": 7, "cc": 1, "ct": 0,
      "mh": 0.0, "ma": 35.0, "mt": 0.0, "mch": 0.0, "mca": 35.0, "mct": 0.0,
      "mmt": 0.0, "mmh": 0.0, "mma": 5.0,
      "cv": 9, "total": 10, "ur": 80.0, "cr": 10.0, "crt": 10.0,
      "epc": 0.35, "epl": 3.5, "appr": 70.0, "app": 7, "per": 7.0,
      "cpc": 0.05, "cpa": 5.0, "cpv": 3.89, "cpl": 0.5, "profit": 30.0, "roi": 600.0, "roic": "green",
      "goal": {
        "1": { "code": "main", "name": "Purchase", "cn": 0, "ch": 1, "ca": 6, "cc": 0, "ct": 0,
               "mh": 5.0, "ma": 30.0, "mt": 35.0, "cv": 7, "total": 7, "cr": 0, "part": 70 }
      },
      "event": {
        "1": { "code": "form", "name": "Form opened", "total": 5, "cr": 6.25, "crt": 5.0, "crn": 50.0,
               "dr": 93.75, "drt": 95.0, "drn": 50.0 }
      }
    }
  },
  "total": { /* same shape, aggregated */ }
}
```

**Stats field decoder:**

| Field | Meaning |
|---|---|
| `id` | Grouping key as numeric/string ID (e.g. date as integer `20230805`, flow ID as int) |
| `name` | Grouping key as human-readable string (e.g. `"2023-08-05"`, flow name) |
| `click` | Total clicks |
| `unique` | Unique clicks |
| `next` | Second-step clicks |
| `spend` | Total click cost (from spend import) |
| `cn` | New leads (status 1) |
| `ch` | Hold leads (status 2) |
| `ca` | Approved leads (status 3) |
| `cc` | Cancelled leads (status 4) |
| `ct` | Trash leads (status 5) |
| `ma` | Total revenue from approved leads |
| `mh` | Total revenue from hold leads |
| `mt` | Total revenue from all leads |
| `mca` | Total cost of approved leads |
| `mch` | Total cost of hold leads |
| `mct` | Total cost of all leads |
| `mma` | Mean (average) revenue per approved lead |
| `mmh` | Mean revenue per hold lead |
| `mmt` | Mean revenue per all leads |
| `cv` | Valid leads (non-trash: cn+ch+ca+cc) |
| `total` | All leads |
| `ur` | Unique ratio % (unique / click × 100) |
| `cr` | Conversion rate % (valid leads / unique clicks) |
| `crt` | Total CR (all leads / unique clicks) |
| `epc` | Earnings per click |
| `epl` | Earnings per lead |
| `appr` | Approval rate % (approved / valid leads) |
| `app` | Total approval rate % (approved / all leads) |
| `per` | Valid leads % |
| `cpc` | Average cost per click |
| `cpa` | Average cost per approved lead |
| `cpv` | Average cost per valid lead. |
| `cpl` | Average cost per lead (any status) |
| `profit` | Revenue minus cost (approved leads) |
| `roi` | ROI % (positive = profit, 0 = breakeven, -100 = total loss) |
| `roic` | ROI color label: `green` (profit), `yellow` (near-zero), `red` (loss) |
| `goal` | Object keyed by the active goal slot (`"1"`–`"5"`): `code`, `name`, `part` (the goal's leads as % of the row's `total`), the lead counters and money fields `cn`, `ch`, `ca`, `cc`, `ct`, `mh`, `ma`, `mt`, `mch`, `mca`, `mct`, `mmt`, `mmh`, `mma`, `cv`, `total`, `epl`, `appr`, `app`; the click-based ratios `ur`, `cr`, `crt`, `epc`, `per` are always `0` here. Only when the numbers have a single owner and that user has active goals — see Owner scope above |
| `event` | Object keyed by the active event slot (`"1"`–`"5"`): `{total, code, name, cr, crt, crn, dr, drt, drn}` (same condition) |

**Errors:** `scope` (HTTP 400), `payment-required` (HTTP 400, not 402!; never for `scope=admin`), `no-item`, `no-from`, `no-to`. `gangs/stats` adds `no-gang` (HTTP 400) and `access` (HTTP 403) before all of these.

---

## 4. tracker/leads — Leads list

```
GET|POST api/tracker/leads.json?id={token}&from=2025-01-01&to=2025-01-31[&scope=user|gang|admin]
GET|POST api/gangs/leads.json?id={token}&from=2025-01-01&to=2025-01-31
```

**Method:** GET or POST (parameters are merged; POST wins). Accepts `form-urlencoded` or JSON body. `scope` itself is read from the query string only.

**Owner scope** (section 2a): `user` (default) returns the caller's own leads; `gang` the whole team's leads (`user=M` narrows to one member; `0`/empty = the whole team; a user outside the team → error `scope`); `admin` (admin accounts) every owner's leads, narrowed by `user` and/or `gang`. `api/gangs/leads` is the same function with `scope=gang` forced (team admins only; a `scope` sent on that URL is overridden). The plan check (`payment-required`) comes after the scope check and is skipped for `scope=admin`. Every returned lead carries its owner in `user`; its `siteurl` is built for the lead's own team when the lead has one (so a team member's lead resolves to the team's site address), otherwise for its user.

**Goal filter and goal label:** the `goal` filter and the `goal` code of every returned lead are looked up in the goal settings of one user — the user named by `user=M` (gang/admin scope), otherwise the caller. The `goal` **filter** knows only goals that are active in those settings; the returned label is the code configured for the lead's slot, even if that goal has since been switched off. A lead stores its goal as a slot number set from its owner's goal settings when the lead was saved. In user scope the settings are the owner's own, so the label matches what the owner sees; with `user=M` it matches for M's leads. In `gang` or `admin` scope **without** `user`, the caller's settings are used, not the lead owners': the `goal` code is turned into a slot number in the caller's settings and matched against every owner's leads (ambiguous when members configured different codes), and the returned label is the caller's code for the lead's slot. Pass `user=M` for exact results on one member's leads.

**Pagination:** pass `show` (rows per page) and `page` (1-based). With `show > 0` only that page is read and `total` is the full number of matching leads. When `show` is absent or below `1`, **all** matching leads are returned unpaginated (`show = -1`, hard-capped at 10000 rows) — use narrow date ranges or `show` to avoid hitting the cap on unbounded pulls.

**Sorting:** applied by the database on both the paginated and unpaginated requests.

| Name | Description |
|---|---|
| `sort` | Column to sort by. Accepted keys: `id`/`lead_id` (lead ID), `time`/`date` (lead time), `status`, `filter`, `goal`, `price`, `cost`, `geo`, `flow`, `way`, `site`. Unknown key → sorted by lead ID. Absent → no ordering at all (natural row order). |
| `order` | `asc` or `desc`; default `desc` (only the literal `asc`, case-insensitive, gives `ASC`). |

**Filters:**

| Name | Description |
|---|---|
| `from`/`to` | Filter by lead date (YYYY-MM-DD, tracker time zone) |
| `ufrom`/`uto` | Filter by lead time (UNIX timestamp) |
| `geo` | Country code |
| `ip` | IP address |
| `f` | Numeric flow ID |
| `way` | Way ID |
| `site` | Site ID |
| `c` | Numeric click ID |
| `st` | Lead status: 1=new, 2=hold, 3=approve, 4=cancel, 5=trash |
| `filter` | Filter step code |
| `mobile` | Numeric device flag: `0` desktop, `1` mobile |
| `os` | Numeric OS ID (same enumeration as the `os` response field below) |
| `browser` | Numeric browser ID (same enumeration as the `browser` response field below) |
| `hour` | Hour of the day of the lead, `0`–`23`, in the tracker time zone (the same as `from`/`to`) |
| `gl` | Numeric goal ID |
| `goal` | Symbolic goal code (e.g. `"main"`), resolved to the numeric goal ID in the goal settings of the user named by `user` (gang/admin scope), otherwise of the caller — see Goal filter and goal label above. Inactive and unknown codes are not found: the filter is then silently ignored (no error). |
| `click` | Symbolic click code |
| `flow` | Symbolic flow code |
| `status` | Status group: `new`, `hold`, `approve`, `cancel`, `trash`, `valid` (all non-trash), `wait` (new+hold), `good` (hold+approve), `bad` (cancel+trash) |
| `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer` | Tag filters |
| `scope` | `user` (default) / `gang` (`team`) / `admin` (`sys`, `system`, `all`), query string only — see section 2a. Absent or unknown → `user`. A refused scope → error `scope` (HTTP 400). Forced to `gang` on `gangs/leads` |
| `user` | `scope=gang`: narrow to one member of the caller's team (a user outside the team → error `scope`; `0`/empty → the whole team). `scope=admin` (admin accounts): narrow to that user. `scope=user`: ignored. Also selects the goal settings used for `goal` |
| `gang` | `scope=admin` (admin accounts) only: narrow to that team, combined with `user` by AND. Ignored in `user` and `gang` scope |
| `sort` | Column to sort by (see list above); unknown → lead ID. Absent → no ordering. |
| `order` | Sort direction; only `asc` gives ascending. Default `desc`. |
| `show` | Rows per page, at most `1000` (a larger value is reduced to `1000`, and `1000` is echoed back); enables paging. Absent or below `1` → all rows, capped at 10000 |
| `page` | 1-based page number (with `show`) |

**Success response** (`page`/`show`/`total` always present; `total` = full number of matching leads when paginated, else the number of returned rows):
```json
{
  "status": "ok",
  "page": 1, "show": 100, "total": 512,
  "leads": [
    {
      "id": 1337, "user": 7, "time": 1714723200,
      "click": "ca1337", "uid": "adv-lead-id",
      "flow": "ef1c", "way": 42, "site": 0,
      "siteurl": false,
      "filter": 0,
      "status": "approve",
      "custom": "approved",
      "goal": false,
      "price": 5.0, "currency": "usd",
      "cost": 0.5, "costcur": "usd",
      "ip": "1.2.3.4", "geo": "us",
      "name": "", "phone": "", "email": "", "comment": "",
      "os": 1, "browser": 1, "ua": "Mozilla/...",
      "meta": {},
      "adid": "", "data1": "", "data2": "", "data3": "", "data4": "", "data5": "",
      "utm_source": "fb", "utm_campaign": "", "utm_content": "", "utm_term": "",
      "utm_medium": "", "domain": "", "subid": "", "customer": ""
    }
  ]
}
```

Field notes:
- `os` — numeric OS ID detected from the stored user-agent (same enumeration as the `os` filter above: `0`=Other, `1`=Windows, `2`=macOS, `3`=Linux, `4`=iOS, `5`=Android, `6`=WindowsMobile, `7`=Symbian, `8`=BlackBerry); `0` when no UA was stored on the lead.
- `browser` — numeric browser ID detected from the stored user-agent (same enumeration as the `browser` filter above: `0`=Other, `1`=Chrome, `2`=Firefox, `3`=IE, `4`=Safari, `5`=Opera, `6`=Edge, `7`=Facebook, `8`=Instagram, `9`=TikTok, `10`=WebView, `11`=Bot); `0` when no UA was stored.
- `user` — owner of the lead. `siteurl` — the site address built for the lead's own team when it has one (a team member's lead resolves to the team's site address), otherwise for its user; `false` when the lead has no site.
- `goal` — the code of the lead's goal slot in the goal settings used for the request (see Goal filter and goal label above), or `false` when the lead has no goal.

**Note:** The `status` field in each lead object is the character status code — one of `new`, `hold`, `approve`, `cancel`, `trash`. The codes are not translated, so the value can be parsed programmatically; it is the same set the `status` filter accepts (the numeric form 1–5 is used by the `st` filter).

**Errors:** `scope` (HTTP 400), `payment-required` (HTTP 400; never for `scope=admin`). `gangs/leads` adds `no-gang` (HTTP 400) and `access` (HTTP 403) before them.

---

## 4a. tracker/lead — One lead with its status changes

```
GET|POST api/tracker/lead.json?id={token}&lead={lead_id}[&scope=user|gang|admin]
GET|POST api/gangs/lead.json?id={token}&lead={lead_id}
```

Returns one lead by its numeric ID together with the history of its status changes. Same owner scope as `tracker/leads` (section 2a): `user` (default) — the caller's own leads; `gang` — the team's leads, `user=M` requires the lead to belong to that member; `admin` (admin accounts) — any owner's lead, narrowed by `user` and/or `gang`. `api/gangs/lead` is the same function with `scope=gang` forced (team admins only; a `scope` sent on that URL is overridden). Parameters may be sent via GET or POST (`form-urlencoded` or JSON; POST wins); `scope` itself is read from the query string only.

**Scope / auth:** API key. Order: `scope`, then the plan check (`payment-required`, HTTP 400; skipped for `scope=admin`; the effective caller's plan, also in gang scope), then `no-lead`, then the lookup.

**Lead outside the scope:** a lead that exists but lies outside the scope (another user's lead in user scope, another team's or another member's lead in gang scope) is `not-found` (HTTP 404) — never `access`. A `user` that is not in the caller's team is not `not-found` but error `scope` (HTTP 400): it is refused before the lookup. The `changes` list is read within the same scope.

**Goal label:** `goal` is always labelled with the goal settings of the lead's owner (unlike `tracker/leads`), so it equals what the owner sees; in user scope the owner is the caller.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `lead` | int | yes | Numeric lead ID. Empty/`0` → `no-lead`. |
| `scope` | enum | no | `user` (default) / `gang` (`team`) / `admin` (`sys`, `system`, `all`), query string only — see section 2a. Absent or unknown → `user`. A refused scope → error `scope` (HTTP 400). Forced to `gang` on `gangs/lead`. |
| `user` | int | no | `scope=gang`: the lead must belong to this member of the caller's team (a user outside the team → error `scope`; `0`/empty → any member). `scope=admin` (admin accounts): the lead must belong to this user. `scope=user`: ignored. |
| `gang` | int | no | `scope=admin` (admin accounts) only: the lead must belong to this team, combined with `user` by AND. Ignored in `user` and `gang` scope. |

**Success response:**
```json
{
  "status": "ok",
  "data": {
    "id": 77, "user": 13, "gang": 0,
    "time": 1760698200, "date": 20261017,
    "status": "approve", "custom": "Confirmed",
    "goal": "paid", "price": 18, "currency": "usd",
    "source": {
      "click": "ca1337", "uid": "c1vvkg272m16", "flow": "ef1c",
      "way": 123, "site": 42, "siteurl": "https://meow.pro/c8/cfb72fe561f0/", "filter": 0
    },
    "buyer": {
      "name": "Hans", "phone": "+49 (30) 226-511-83", "email": "noreply@gmx.de",
      "comment": false, "cost": 35, "costcur": "eur"
    },
    "visitor": {
      "ip": "12.34.56.78", "geo": "de", "mobile": 0, "os": 2, "browser": 4,
      "ua": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ... Safari/605.1.15"
    },
    "tags": {
      "adid": "bf9d4aa8-4979-11ee-94c6-005056aec456",
      "data1": false, "data2": false, "data3": false, "data4": false, "data5": false,
      "utm_source": "fb", "utm_campaign": false, "utm_content": false, "utm_term": false,
      "utm_medium": false, "domain": "meow.pro", "subid": "purr", "customer": false
    },
    "meta": {},
    "changes": [
      { "id": 301, "time": 1760698260, "from": 1, "to": 2, "from_status": "new", "to_status": "hold", "price": 0, "cost": 35 },
      { "id": 307, "time": 1760698500, "from": 2, "to": 3, "from_status": "hold", "to_status": "approve", "price": 18, "cost": 35 }
    ]
  }
}
```

Field notes:
- Top level — `id`, `user` (owner), `gang` (team the lead belongs to, `0` if none), `time` (UNIX timestamp), `date` (arrival day as the integer `YYYYMMDD`), `status` (character code `new`/`hold`/`approve`/`cancel`/`trash`), `custom` (advertiser's own status text or `false`), `goal` (goal code of the lead owner's settings, or `false`), `price`, `currency`, `meta` (extra fields, empty when none).
- `source` — `click` (character ID or `false`), `uid` (lead ID on the advertiser's side or `false`), `flow` (character ID), `way`, `site`, `siteurl` (built as in the leads list, or `false`), `filter` (filter result code, same list as in the leads list).
- `buyer` — `name`, `phone`, `email`, `comment` (each `false` when empty), `cost` and `costcur` (cost and its currency).
- `visitor` — `ip`, `geo`, `mobile` (`0` desktop / `1` mobile), `os`, `browser` (numeric IDs, same enumerations as in the leads list), `ua` (or `false`).
- `tags` — `adid`, `data1`–`data5`, `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer`; each a string or `false`.
- `changes` — status history, oldest first, empty when none. A row is written whenever the status, the price or the cost of the lead changes. `from`/`to` are status numbers (`1` new, `2` hold, `3` approve, `4` cancel, `5` trash; `to` equals `from` for a price/cost-only change), `from_status`/`to_status` the character codes (or `false`), `price`/`cost` the values after the change, `time` the moment of the change.
- Compared with the leads list the fields are grouped into `source`, `buyer`, `visitor` and `tags`; `time`, `date`, `gang` and `changes` are extra.

**Errors:** `scope` (HTTP 400), `payment-required` (HTTP 400; never for `scope=admin`), `no-lead` (HTTP 400, `lead` missing), `not-found` (HTTP 404, no such lead or not in the caller's scope); `gangs/lead` adds `no-gang` (HTTP 400) and `access` (HTTP 403) before all of these.

---

## 5. tracker/clicks — Raw click list (traffic log)

```
GET|POST api/tracker/clicks.json?id={token}&from=2025-01-01&to=2025-01-31[&scope=user|gang|admin]
GET|POST api/gangs/clicks.json?id={token}&from=2025-01-01&to=2025-01-31
```

The click-level analogue of `tracker/leads` — same filter/pagination/sorting mechanics, but reports every recorded click regardless of whether it produced a lead. Reads from the click log (not from the leads).

**Method:** GET or POST (parameters are merged; POST wins). Accepts `form-urlencoded` or JSON body. `scope` itself is read from the query string only.

**Scope / auth:** API key. Order: `scope`, then the plan check (`payment-required`, HTTP 400; skipped for `scope=admin`; the effective caller's plan, also in gang scope), then the query.

**Owner scope** (section 2a): `user` (default) returns the caller's own clicks; `gang` the whole team's clicks (`user=M` narrows to one member; `0`/empty = the whole team; a user outside the team → error `scope`); `admin` (admin accounts) every owner's clicks, narrowed by `user` and/or `gang`. `api/gangs/clicks` is the same function with `scope=gang` forced (team admins only; a `scope` sent on that URL is overridden). Every returned click carries its owner in `user`; its `siteurl` is built for the click's own team when the click has one (a team member's click resolves to the team's site address), otherwise for its user. Clicks have no goal filter or goal label.

**Pagination:** identical mechanics to `tracker/leads`: pass `show` (rows per page) and `page` (1-based). With `show > 0` only that page is read and `total` is the full number of matching clicks. When `show` is absent or below `1`, all matching clicks are returned unpaginated (`show = -1`, hard-capped at 10000 rows).

**Sorting:** applied by the database.

| Name | Description |
|---|---|
| `sort` | Column to sort by. Allowed values: `id`/`click_id` (click ID), `time`/`date` (click time), `filter`, `price`, `geo`, `mobile`, `os`, `browser`, `flow`, `way`, `site`. Unknown key → sorted by click ID. Absent → no ordering. |
| `order` | `asc` or `desc`; default `desc` (only literal `asc` gives ascending). |

**Filters:**

| Name | Description |
|---|---|
| `from`/`to` | Filter by click date (YYYY-MM-DD, tracker time zone) |
| `ufrom`/`uto` | Filter by click time (UNIX timestamp) |
| `geo` | Country code |
| `ip` | IP address (matched through a 32-bit hash) |
| `f` | Numeric flow ID |
| `way` | Way ID |
| `site` | Site ID |
| `offer` | Offer ID |
| `filter` | Numeric click filter code (see **Click filter codes** above) |
| `mobile` | Numeric device flag |
| `os` | Numeric OS ID |
| `browser` | Numeric browser ID |
| `unique` | Numeric unique flag |
| `hour` | Hour of the day of the click, `0`–`23`, in the tracker time zone (the same as `from`/`to`) |
| `next` | Numeric second-step flag |
| `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer` | Tag filters (matched through 32-bit hashes of the values) |
| `click` | Symbolic click code |
| `flow` | Symbolic flow code |
| `scope` | `user` (default) / `gang` (`team`) / `admin` (`sys`, `system`, `all`), query string only — see section 2a. Absent or unknown → `user`. A refused scope → error `scope` (HTTP 400). Forced to `gang` on `gangs/clicks` |
| `user` | `scope=gang`: narrow to one member of the caller's team (a user outside the team → error `scope`; `0`/empty → the whole team). `scope=admin` (admin accounts): narrow to that user. `scope=user`: ignored |
| `gang` | `scope=admin` (admin accounts) only: narrow to that team, combined with `user` by AND. Ignored in `user` and `gang` scope |
| `sort` | Column to sort by (see the allowed values above); unknown → click ID. Absent → no ordering. |
| `order` | Sort direction; only `asc` gives ascending. Default `desc`. |
| `show` | Rows per page, at most `1000` (a larger value is reduced to `1000`, and `1000` is echoed back); enables paging. Absent or below `1` → all rows, capped at 10000 |
| `page` | 1-based page number (with `show`) |

**Success response** (`page`/`show`/`total` always present):
```json
{
  "status": "ok",
  "page": 1, "show": 100, "total": 2048,
  "data": [
    {
      "id": 90210, "user": 7, "time": 1714723200,
      "click": "ca1337", "flow": "ef1c",
      "offer": 3, "way": 42, "site": 0, "sitenext": 0,
      "siteurl": false,
      "filter": 0,
      "unique": true, "next": false,
      "price": 0.05, "currency": "usd",
      "ip": "1.2.3.4", "geo": "us",
      "mobile": false,
      "os": 1, "browser": 1, "ua": "Mozilla/...",
      "adid": "", "data1": "", "data2": "", "data3": "", "data4": "", "data5": "",
      "utm_source": "fb", "utm_campaign": "", "utm_content": "", "utm_term": "",
      "utm_medium": "", "domain": "", "subid": "", "customer": ""
    }
  ]
}
```

**IMPORTANT:** the array key is **`data`**, not `leads` (contrast with `tracker/leads`, which uses `leads`).

Each click object:
- `id`, `user`, `time` — numeric IDs / UNIX ts.
- `click` — always the symbolic click code; unlike leads, always present since every row here *is* a click.
- `flow` — symbolic flow code.
- `offer`, `way`, `site` — numeric IDs. `sitenext` — numeric ID of the second-step site.
- `siteurl` — site address built for the click's own team when it has one, otherwise for its user; `false` when the click has no site.
- `filter` — numeric click filter code.
- `unique`, `next` — booleans.
- `price`, `currency`.
- `ip`, `geo`.
- `mobile` — boolean.
- `os`, `browser` — numeric IDs detected from the stored user-agent (same enumerations as the `os`/`browser` filters above), or `0` when no UA was stored.
- `ua` — raw user-agent, or `false`.
- `adid`, `data1`–`data5`.
- `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer`.

Fields empty in storage are coerced to `false` (not `""`/null).

**Differences vs `tracker/leads`:**
- Response envelope key is `data`, not `leads`.
- No lead-status/custom/goal/name/phone/email/comment/meta fields — this is raw traffic, not converted leads.
- Adds click-specific fields: `offer`, `sitenext`, `unique`, `next`, `mobile`.
- Filter set adds `offer`/`mobile`/`os`/`browser`/`unique`/`next` and drops the lead-only `c`/`st`/`gl`/`goal`/`status` filters.

**Errors:** `scope` (HTTP 400), `payment-required` (HTTP 400; never for `scope=admin`). `gangs/clicks` adds `no-gang` (HTTP 400) and `access` (HTTP 403) before them.

---

## 6. tracker/spend — Import click spend

```
POST api/tracker/spend.json?id={token}[&scope=user|gang|admin]
POST api/gangs/spend.json?id={token}
```

Bulk-updates the price (cost) of the matching clicks. Supports batch mode. All items of one request share one owner, chosen as below; the owner (user and/or team) is added to the filters of every item. There is **no plan check** (`payment-required`) in `spend`.

**Which items count:** an item is dropped unless it has a price (`cpc` / `cost` / `zero`) and at least one filter field present in the request — `flow` (with a known flow code), `way`, `geo`, `from`, `to` or a tag filter; `valid`, `unique` and `filter` do not count. If every item is dropped the answer is `empty`. A filter field sent empty still counts as present: without an owner condition such an item matches nothing and is counted in `bad` (the answer is not `empty`); with an owner condition the empty field leaves only that condition, so all of the owner's clicks are repriced.

**Method:** POST recommended (`batch` needs a POST/JSON body). `scope` is read from the query string only; `wm`, `gang` and `user` are read POST first, then GET.

**Order of answers:** framework (`key`, `ban`) → `gangs/spend` guard (`no-gang`, `access`) → `scope` → owner choice → `empty`.

**Owner choice by scope** (section 2a):

- **`gang`** (`scope=gang` / `team`, and every `gangs/spend` call; a `scope` sent on that URL is overridden): the team is the caller's own. The user is `wm` if given, else the `user` (which must be a member of the team), else none — then the whole team's clicks are repriced. `wm` wins over `user`. `user` outside the team (or of a former member) → error `scope`; `user=0` / empty = the whole team. `wm` is **not** checked against the team: a `wm` outside the team raises no `scope` error, the team condition is still applied, so it can match only clicks that carry both that user and the team.
- **`admin`** (`scope=admin` / `sys` / `system` / `all`, admin accounts only): the user is `user`, the team is `gang` (integers; `0`/empty = not given). `wm` and the word `any` are not special (`user=any` counts as not given). **A write with neither `user` nor `gang` goes through** and reprices the matching clicks of **every owner**. The item's own filters are then the only conditions (see **Which items count**).
- **`user`** (no `scope`, `scope=user`, or an unknown word): with no `gang` opt-in a caller without an admin account always writes to their own clicks, and `user` is ignored. Admin accounts: a `user` replaces the caller as the owner — an id picks that user; `any`, `0`, empty or a non-numeric value means no user condition (every user's clicks). **Team-admin opt-in:** a caller who has a team and is its admin and sends a truthy `gang` (a flag, not an id — the team is always the caller's own) works for the team: the user is `wm`, else `user`, else none (the whole team). This replaces the owner above, also for admin accounts, and neither `wm` nor `user` is checked against the team (the team condition is still applied). Without a truthy `gang` the team is not set.

**Single item parameters:**

| Name | Type | Description |
|---|---|---|
| `cpc` | float | Price per click |
| `cost` | float | Total cost (divided by click count to get CPC) |
| `zero` | flag | Set price to 0 |
| `currency`/`curr`/`cur` | string | ISO currency (default `usd`) |
| `from`/`to` | string | Date range (YYYY-MM-DD, UNIX timestamp, or any date string PHP `strtotime` understands) |
| `flow` | string or array | Symbolic flow code(s) |
| `way` | int or array | Way ID(s) |
| `geo` | string | Country code |
| `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `domain`, `subid`, `customer` | string | Tag filters |
| `filter` | int | Click filter value |
| `valid` | flag | Only clicks that passed all filters (click filter code `0`) |
| `unique` | flag | Only unique clicks |
| `batch` | array | Array of item objects (POST/JSON only) |
| `scope` | enum | `user` (default) / `gang` (`team`) / `admin` (`sys`, `system`, `all`), query string only — see Owner choice above. Absent or unknown → `user`. A refused scope → error `scope` (HTTP 400). Forced to `gang` on `gangs/spend` |
| `user` | int or "any" | `scope=gang` / `gangs/spend`: pins one member of the caller's team (a user outside the team → error `scope`; `0`/empty = the whole team; overridden by `wm`). `scope=admin`: pins that user (`0` / `any` / empty = no user condition). No `scope`: admin accounts only — replaces the caller (`any` / `0` / empty = every user), ignored for others; with the team-admin opt-in it pins a team member when `wm` is absent |
| `wm` | int | `scope=gang` / `gangs/spend` and the team-admin opt-in: pins one team member; wins over `user`; not checked against the team. Ignored in `admin` scope and in the plain user path |
| `gang` | truthy / int | No `scope`: a team admin sends a truthy value to opt into the team (the team is always the caller's own). `scope=admin`: the team id to narrow to. Ignored in `gang` scope |

**Batch example:**
```json
{
  "batch": [
    {"flow": "ef1c", "from": "2025-01-01", "to": "2025-01-31", "cost": 100.0},
    {"flow": "ab2d", "from": "2025-01-01", "to": "2025-01-31", "cpc": 0.05}
  ]
}
```

**Success response:** `{"status":"ok","ok":1,"bad":0}`

`ok`/`bad` = counts of successful/failed items (relevant for batch).

**Errors:** `scope` (HTTP 400; includes `user` outside the team on every `gangs/spend` call, and — the only way a request without `scope` can get it — a missing, banned or inactive effective caller via `uu`); `empty` (HTTP 404) — every item was dropped (see **Which items count**). `gangs/spend` adds `no-gang` (HTTP 400) and `access` (HTTP 403) before them.

---

## 7. tracker/templates — CPA-network postback templates

```
GET api/tracker/templates.json?id={token}
```

Read-only dump of the **postback template** reference book — presets for CPA networks / affiliate programs shown in the flow editor's "template" selector. No parameters, no tariff gate. The body merges two parts into one `data` object:
- the **built-in** book, refreshed by the tracker's scheduled job every 10 minutes;
- the **user-generated** book, rebuilt by the same job (its IDs are offset by `+10000`, so they never collide with the built-in IDs).

Both parts share the same field set.

**Response:** `{ "status": "ok", "data": { "<id>": {…}, … } }`, keyed by numeric template ID. Each template:

| Field | Type | Description |
|---|---|---|
| `name` | string | CPA network / affiliate program name (always present) |
| `url` | string | Offer/landing URL (also used by the TDS fast-redirect lookup) |
| `param` | string[] | Placeholder names (`[[name]]`) used in the postback query strings |
| `pb` | string | **Global** postback query string |
| `pbr` | string | **Registration** postback query string |
| `pbd` | string | **Deposit** postback query string |
| `pbs` | string | **RevShare payment** postback query string |
| `pbn` | string | **New lead** postback query string |
| `pbh` | string | **Hold** postback query string |
| `pba` | string | **Approve** postback query string |
| `pbc` | string | **Cancel** postback query string |
| `pbt` | string | **Trash** postback query string |

Each `pb*` value is a query string **without** the leading `?`. Final network callback URL = `{postback_base}?{pb*}`, with `[[placeholder]]` macros (from `param`) substituted. Only the relevant fields are present per template.

```json
{
  "status": "ok",
  "data": {
    "12": {
      "name": "Acme CPA",
      "url": "https://acme.example/offer?click=[[click]]",
      "param": ["click","status"],
      "pb": "click=[[click]]&event=global",
      "pba": "click=[[click]]&event=approve"
    }
  }
}
```

---

## 8. tracker/networks — Traffic-source presets

```
GET api/tracker/networks.json?id={token}
```

Read-only dump of the **traffic-source** (ad network) reference book shown in the flow editor's "source" selector. No parameters, no tariff gate. The body merges two parts into one `data` object: the **built-in** book (refreshed by the tracker's scheduled job every 10 minutes) and the **user-generated** book (rebuilt by the same job; its IDs are offset by `+10000`). Both parts share the same field set.

**Response:** `{ "status": "ok", "data": { "<id>": {…}, … } }`, keyed by numeric source ID. Each source:

| Field | Type | Description |
|---|---|---|
| `name` | string | Traffic source / ad network name (always present) |
| `utms`, `utmc`, `utmn`, `utmt`, `utmm` | string | UTM Source/Campaign/Content/Term/Medium value templates |
| `subid` | string | SubID value template |
| `data1`–`data5` | string | Custom data-slot value templates |
| `hide1`–`hide6` | string | Hidden data-field value templates |
| `cpc` | string | Default click cost |
| `cur` | string | Default currency (default `usd`) |
| `pbglobal` | string | **Global** outbound postback URL |
| `pbnew` | string | Outbound postback URL fired on **new** lead |
| `pbhold` | string | Outbound postback URL fired on **hold** |
| `pbapprove` | string | Outbound postback URL fired on **approve** |
| `pbcancel` | string | Outbound postback URL fired on **cancel** |
| `pbtrash` | string | Outbound postback URL fired on **trash** |
| `param` | string[] | Placeholder names (`[[name]]`) used in the value templates |

Value templates may embed `[[placeholder]]` macros, substituted when the source is applied to a flow. Only configured fields are present.

**Note on naming:** networks use the long postback keys `pbglobal`/`pbnew`/`pbhold`/`pbapprove`/`pbcancel`/`pbtrash`, whereas `tracker/templates` uses the short `pb`/`pbn`/… codes.

```json
{
  "status": "ok",
  "data": {
    "5": {
      "name": "Facebook Ads",
      "utms": "[[campaign]]",
      "cpc": "0.15", "cur": "usd",
      "pbapprove": "https://graph.facebook.com/...?ev=Purchase",
      "param": ["campaign"]
    }
  }
}
```

---

## 9. tracker/settings — Update tracker config

```
POST api/tracker/settings.json?id={token}
```

Saves per-user (or per-team) tracker storage configuration. At least one setting parameter is required. The settings go to the caller's own block (the effective user after `uu`), or with `scope=gang` to the team's block. The function has no plan check and does not use the owner scope of section 2a: `scope` is a field of the **POST body** here, a `scope` in the query string is ignored, and only the literal `gang` is special (`team` and the other owner-scope words are not aliases; any other value, or none, writes the caller's own block). With `scope=gang` the caller must have a team and be one of its admins; otherwise the call is refused with `scope` and nothing is written.

**Parameters (all POST):**

| Name | Type | Description |
|---|---|---|
| `scope` | string | `gang` — save the team's config; the caller must have a team and be its admin, else error `scope` (HTTP 400). Any other value, or absent: the caller's own block. Read from the POST body, not from the query string |
| `storage` | truthy | Enable external storage server |
| `domain` | string | Storage domain name |
| `control` | string | Control script URL |
| `key` | string | Control script key |
| `ips` | string | Allowed control IPs |

**Success response:** `{"status":"ok"}`

**Errors:** `scope` (HTTP 400) — `scope=gang` by a caller who has no team or is not its admin (nothing is written; checked before `no-params`); `no-params` (HTTP 400) — no recognized keys provided. There is no `access` error here.

**Note:** No `gangs` alias. Team admins use `scope=gang` in the POST body of this endpoint.

---

## 10. gangs/* — Team-scope wrappers

**Available gang functions:** `postback`, `stats`, `leads`, `lead`, `clicks`, `spend`

**Requirements (checked first, before the tracker function runs):**
- Caller must be assigned to a team (gang) — else `no-gang` (HTTP 400)
- Caller must be the team admin — else `access` (HTTP 403)

**`gangs/X` = `tracker/X?scope=gang` plus this guard** for `stats`, `leads`, `lead`, `clicks` and `spend` (section 2a). The scope is forced after the guard: a `scope` sent on a `gangs/*` URL (for instance `scope=admin`) is overridden with `gang`, and `gang=` cannot redirect it — the team is always the caller's own. Order of answers: `no-gang` → `access` → `scope` (a `user` outside the team, or a banned/inactive effective caller who is still a team admin) → `payment-required` (not for `spend`) → parameter errors → data. `user=0` or empty means the whole team.

**Differences from user scope:**
- `postback`: the lead is looked up among the team's leads instead of the user's. It does not use `scope`
- `stats`: the whole team's numbers; optional `?user=N` (a team member) narrows to that member and, only then, adds the `goal` / `event` objects
- `leads`: the team's leads; optional `?user=M` narrows to a member. The `goal` filter and labels use the settings of `user=M` when given, otherwise the caller's (see section 4)
- `lead`: looks the lead up among the team's leads; optional `?user=M` requires the lead to belong to that member; a lead of another team is `not-found` (HTTP 404). The `goal` label is the lead owner's
- `clicks`: the team's clicks; optional `?user=M` narrows to a member (identical structure to `gangs/leads`)
- `spend`: the team is the caller's own; `wm` (wins) or a validated `user` pins one member, otherwise the whole team's clicks are repriced; no plan check (see section 6)

For `stats`, `leads`, `lead`, `clicks` and `spend` the parameter lists and responses are those of the `tracker` functions; `scope` is not needed.

---

## 11. Internal Endpoints

These are NOT standard API endpoints — no `?id=` auth required.

### tracker/checkdom

```
GET api/tracker/checkdom.raw?host={dom_addr}&code={hmac}[&ip={ip}]
```

Used by storage servers to verify parked domains. Compares an HMAC-SHA256 of `"{host}-{user API key}"`, keyed with the tracker's installation secret, against the `code` value. For a domain that belongs to a team, the API key in the HMAC is the team leader's.

**Response:** plaintext `ok`, `wrong`, or `error`.

**Side effects:** Updates the stored IP address(es) of the domain when the IP changes.

### tracker/domains

```
GET api/tracker/domains.raw
```

Returns domains parked for the caller's IP (status 1, 2, or 4). Used by storage servers for Nginx config.

**Response:** Plaintext one domain per line, or JSON array (no `status` envelope).

**Error:** `bad-ip` when caller IP can't be determined.

### tracker/client

```
GET api/tracker/client.json
```

Downloads `server.php` (storage server installer). No authentication required. Returns file with `Content-Disposition: attachment; filename=server.php`.
