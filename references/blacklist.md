# Blacklist API Reference

## Overview

The `blacklist` application manages the traffic-filter **block-lists** ("Black lists" in the UI): named collections of IPv4 addresses, IPv4 networks, IPv6 networks or request-header rules that the cloaking filter (`/fltr/`) and the flow filter consult on every visit. It creates and configures lists, edits entries one by one, empties a list, imports/exports CSV, and shows which lists a filter campaign may use. The campaigns themselves are managed by `/api/filter/*` (`filter/set` chooses lists by ID in its `list` field).

Base URL: `api/blacklist/{func}.json?id={token}[&scope=user|gang|admin]`

Functions: `avail`, `list`, `show`, `add`, `edit`, `del`, `block`, `remove`, `change`, `clean`, `import`, `export`.

### List kinds (`type` of a list)

| `type` | Kind | Entry | Entry identity (duplicate detection) |
|---|---|---|---|
| 1 | `ip` | IPv4 address | address |
| 2 | `net` | IPv4 network, prefix `size` 8–31 | network start + `size` |
| 3 | `ip6` | IPv6 network, prefix `size` 32–128 (a single address is 128) | network start + `size` |
| 4 | `bl` | Header rule: `value` + header ID `type` (+ `regexp`, `white`) | header ID + `value` (case-insensitive unless `regexp`) |

The kind is fixed at creation (`edit` cannot change it).

### Ownership and scopes

Each list belongs to one owner; `role` in the list object shows which:

| `role` | Owner | UI name |
|---|---|---|
| `user` | one webmaster (`user` > 0, `gang` = 0) | My filters |
| `team` | one team (`gang` > 0, `user` = 0) | Team filters |
| `admin` | global manual list (`user` = 0, `gang` = 0) | Traffic filters |
| `feed` | system list updated from the supplier catalog (`feed` non-empty; checked first) | Traffic filters |

| Scope | `list` returns / item functions accept | `add` creates |
|---|---|---|
| `user` (default) | own personal lists | personal list, active, not default |
| `gang` / `team` (team admin only) | the team's lists | team list, active, not default |
| `admin` + `gang=N` | team N's lists | team list (as `gang`) |
| `admin` + `user=N` | user N's personal lists | personal list (as `user`) |
| `admin` (neither) | global lists, supplier feeds included; item functions accept **any** list | global list, **inactive and not default** |

With `admin` and both `gang` and `user`, the team wins. A non-admin never sees global lists through `list`/`show`/`edit`; they are visible read-only through `avail`.

### How filtering uses the lists

- A campaign stores chosen list IDs (`filter/set` keeps only IDs from the owner's active global, personal and team lists). With no selection, the owner's **default** lists are used: active defaults of global + owner + team.
- IPv4 visits are looked up in `ip` and `net` lists, IPv6 in `ip6` lists, request headers against `bl` rules: a black rule (`white: false`) blocks on match (substring, or PCRE when `regexp`); a white rule (`white: true`) blocks when the header is missing or does **not** match.
- The flow filter (tracker) uses only the `ip`, `net` and `ip6` kinds of the default lists.
- A list is **locked** when it is a supplier feed or a URL mirror: entries cannot be changed by hand (`locked` error), the content comes from the feed / file.

### Conventions

- Auth: `?id={user_id}-{api_key}`. `scope` is read from the **GET** query string only, even with a JSON body. `gang` / `user` select the owner in admin scope (POST first, then GET). `uu={user_id}` (key owner is an admin account) acts as another user.
- No method check: GET and POST params are merged, POST wins; JSON body accepted. Use GET for reads and POST for writes; `import` is `multipart/form-data`.
- Success: HTTP 200, `{"status":"ok", ...}`. Error: `{"status":"error","error":"<code>"[,"message":"<localized text>"]}` with HTTP 400 (`db` 500, bad key 401, banned 403). Headers `X-API-Status` / `X-API-Error` repeat them.
- `export` is the only function that does not return JSON. A map keyed by ID that is empty serializes as `[]`.
- IDs (list, entry, campaign) are plain integers. There is no billing gate and no per-user quota in this app.

---

## Objects

### List object

Returned by `list` (in `data`), `add`, `edit`, and as `dataset.list` of `show`.

| Field | Type | Description |
|---|---|---|
| `id` | int | List ID |
| `name` | string | Name |
| `type` | int | Kind 1–4 |
| `role` | string | `feed` / `team` / `user` / `admin` |
| `user` / `gang` | int | Owner user / team ID, 0 if none |
| `status` | bool | Active; only active lists can be chosen in campaigns |
| `default` | bool | Default list; used when a campaign has no explicit selection |
| `size` | int | Entry count |
| `update` | int | Unix time of the last successful update, 0 = never |
| `locked` | bool | Entries cannot be changed by hand (feed or mirror) |
| `mirror` | bool | Mirror of a file at `url` (not a feed, `auto` on, `url` set) |
| `auto` | bool | Automatic update switch (feed refresh or URL mirror) |
| `url` | string | Mirror source address |
| `period` | int | Mirror update period, seconds |
| `periods` | int[] | Allowed periods for this list's role; empty for feed lists |
| `check` | int | Unix time of the last mirror attempt |
| `fail` | int | Consecutive failed mirror attempts |
| `feed` | string | Supplier-feed UID, `""` otherwise |
| `private` | bool | Feed requires an access key |
| `key` | string | Feed access key |

### Entry object

| Kind | Fields |
|---|---|
| `ip` | `id`, `ip`, `comment`, `ttl` |
| `net` | `id`, `ip` (network start), `size`, `to` (last address), `comment`, `ttl` |
| `ip6` | `id`, `ip` (network start, compressed), `size`, `to` (last address), `comment`, `ttl` |
| `bl` | `id`, `value`, `type` (header ID), `regexp` (bool), `white` (bool), `comment`, `ttl` |

### Entry fields and validation

| Field | Kinds | Rule |
|---|---|---|
| `ip` | ip, net, ip6 | Valid IPv4 / IPv6 address; private and reserved ranges and `0.0.0.0` are rejected (`bad-ip`). Networks are masked to the network start. |
| `size` | net, ip6 | Prefix length: net 8–31, ip6 32–128 (`bad-size`) |
| `value` | bl | Header text or PCRE; trimmed, non-empty, valid UTF-8, at most 250 characters (`bad-value`) |
| `type` | bl | Header ID 1–255 (`bad-type`); unknown IDs are stored but never match |
| `regexp` | bl | `1` = `value` is a PCRE with delimiters (not validated on write); `0` = case-insensitive substring |
| `white` | bl | `1` = white rule, `0` = black rule |
| `comment` | all | Reason code, clamped to 0–255; non-numeric = 0 |
| `ttl` | all | Expiry as a **Unix timestamp**, `0` = never; clamped to 0–4294967295 |

### Reason codes (`comment`)

1 Public list, 2 Suspicious, 3 Corporate, 4 Different User-Agent, 5 GEO error, 6 Header error, 7 Multiple errors, 8 Bad request, 9 Bot. (Localized map returned as `dataset.comments` by `show`.)

### Header IDs (`type` of kind `bl`)

1 User-Agent, 2 Accept, 3 Accept-Charset, 4 Accept-Encoding, 5 Accept-Language, 6 Authorization, 7 Cache-Control, 8 Connection, 9 Content-Disposition, 10 Date, 11 Expect, 12 From, 13 Host, 14 If-Match, 15 If-Modified-Since, 16 If-None-Match, 17 If-Range, 18 If-Unmodified-Since, 19 Max-Forwards, 20 Pragma, 21 Proxy-Authorization, 22 Range, 23 Referer, 24 TE, 25 Trailer, 26 Transfer-Encoding, 27 Upgrade, 28 Via, 30 X-Forwarded-For, 31 X-Purpose, 32 X-FB-HTTP-Engine, 33 X-Frame-Options, 34 X-WAP-Profile, 35 X-Real-IP, 36 Client-IP, 37 CF-Connecting-IP, 40 Priority, 41 Sec-Fetch-Mode, 42 Sec-Fetch-Site, 43 Sec-Fetch-Dest, 44 Sec-Fetch-User, 45 Sec-CH-UA, 46 Sec-CH-UA-Mobile, 47 Sec-CH-UA-Platform, 48 DNT, 49 Origin, 50 Upgrade-Insecure-Requests, 51 Save-Data, 52 X-Forwarded-Host, 53 X-Forwarded-Proto, 100 Remote Address, 101 Remote Port, 102 Request Method, 103 Request URI, 104 Request Query String, 106 Request Scheme. IDs 29, 38, 39, 105 do not exist.

### Limits by list role

The role comes from the list row, not from the request scope.

| Limit | `admin` role (global) | `user`, `team`, `feed` roles |
|---|---|---|
| Allowed mirror `period`, seconds | 600, 1800, 3600, 10800, 21600, 43200, 86400, 259200, 604800 | 10800, 21600, 43200, 86400, 259200, 604800 |
| Max `import` file size | 268 435 456 bytes | 8 388 608 bytes |
| Max rows of a mirrored file | unlimited | 200 000 (500 for kind `bl`) |
| Mirrored lists per owner | unlimited | 5 |

The row cap applies to URL mirrors only, not to `import` / `block`. There is no limit on lists per owner or entries per list.

---

## Functions

### blacklist/avail — Lists a campaign may attach

```
GET api/blacklist/avail.json?id={token}[&uid={campaign_id}][&scope=...]
```

Lists the filter-campaign owner may attach, with default / selected flags (data of the campaign "Filters set" picker). Inactive lists are never offered.

| Name | Type | Description |
|---|---|---|
| `uid` | int | Filter campaign ID. When sent (even `0`) the campaign must exist in the scope, else `no-site`. Its owner becomes the list owner and its selection drives `selected`. |
| `gang` / `user` | int | Admin scope: owner selection, must exist. Without `uid` the owner is the scope user: `gang` scope needs `user=<member>`, `admin` scope needs `user=<id>`, `user` scope uses the key owner; none found → `no-user`. |

Order: active global lists, then the owner's personal lists, then the owner's team lists; each part sorted by name; duplicates keep the first occurrence.

```json
{
  "status": "ok",
  "data": {
    "5":  {"id": 5,  "name": "Public proxies", "type": 1, "role": "admin", "default": true,  "selected": false},
    "12": {"id": 12, "name": "My bad UAs",     "type": 4, "role": "user",  "default": false, "selected": true}
  },
  "dataset": {"types": {"1": "IP addresses", "2": "IP networks", "3": "IPv6 addresses", "4": "Headers"}}
}
```

`default` — in the owner's default set (system + own + team). `selected` — chosen in campaign `uid`; always `false` without `uid`, and an inactive list stored in the campaign is not reported. `dataset` text follows the API user's language.

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-site`, `no-user`

---

### blacklist/list — Lists of the scope

```
GET api/blacklist/list.json?id={token}[&scope=...][&type=][&search=][&page=][&show=]
```

| Name | Type | Description |
|---|---|---|
| `type` | int 1–4 | Kind filter. Sent but not 1–4 → `bad-type`; empty string = no filter. |
| `search` | string | Name search: at least 2 characters, split by spaces/commas, **all** words must occur as substrings (case-insensitive). Text yielding no words adds no filter. |
| `page` | int | Page number, min 1 (default 1) |
| `show` | int | Page size 0–1000. Default `0` = return every list, no paging, no `nav`. |
| `gang` / `user` | int | Admin scope owner selection (not checked for existence) |

Order: `status` DESC (active first), then name ASC. A page beyond the last returns empty `data`.

```json
{
  "status": "ok",
  "data": {
    "12": {"id": 12, "name": "My bad UAs", "type": 4, "role": "user", "user": 7, "gang": 0,
           "status": true, "default": false, "size": 42, "update": 0, "locked": false,
           "mirror": false, "auto": false, "url": "", "period": 0, "periods": [10800, 21600, 43200, 86400, 259200, 604800],
           "check": 0, "fail": 0, "feed": "", "private": false, "key": ""}
  },
  "nav": {"total": 1, "page": 1, "pages": 1, "show": 50},
  "dataset": {"types": {"1": "IP addresses", "2": "IP networks", "3": "IPv6 addresses", "4": "Headers"},
              "periods": {"600": "10 minutes", "86400": "1 day"}}
}
```

`nav` appears only when `show > 0`. `dataset.periods` carries all nine period labels (shortened above).

**Errors:** `scope`, `bad-gang`, `bad-user`, `bad-type`

---

### blacklist/show — Entries of a list

```
GET api/blacklist/show.json?id={token}&list={list_id}[&search=][&comment=][&type=][&page=][&show=]
```

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID in the scope |
| `search` | string | By kind: `ip` — exact IPv4 address; `net` — networks containing this IPv4 address; `ip6` — networks containing this IPv6 address; `bl` — substring of `value`. Invalid address → `bad-search`. Empty = no filter. |
| `comment` | int 0–255 | Reason-code filter; `0` = no filter; otherwise `bad-comment` |
| `type` | int 0–255 | **Kind `bl` only:** header-ID filter; `0` = no filter; otherwise `bad-type`. Ignored for other kinds. |
| `page` | int | Page number, min 1 |
| `show` | int | Page size, default 50, max 1000 (`< 1` → 50) |

Order: ascending by address (`ip`, `net`, `ip6`) or by `value` (`bl`). Works for locked lists too.

```json
{
  "status": "ok",
  "data": {
    "301": {"id": 301, "value": "HeadlessChrome", "type": 1, "regexp": false, "white": false, "comment": 9, "ttl": 0}
  },
  "nav": {"total": 1, "page": 1, "pages": 1, "show": 50},
  "dataset": {"list": {/* list object */},
              "comments": {"1": "Public list", "9": "Bot"},
              "headers": {"1": "User-Agent", "2": "Accept"}}
}
```

`dataset.headers` appears for kind `bl` only.

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `bad-search`, `bad-comment`, `bad-type`

---

### blacklist/add — Create a list

```
POST api/blacklist/add.json?id={token}[&scope=...]
```

Creates an empty list owned by the scope (see the ownership table).

| Name | Type | Description |
|---|---|---|
| `name` | string | **Required.** Tags stripped, trimmed; empty → `no-name`. Keep within 100 characters (the API does not truncate). |
| `type` | int 1–4 | **Required.** Kind; otherwise `bad-type`. Cannot be changed later. |
| `gang` / `user` | int | Admin scope: owner, must exist (`bad-gang` / `bad-user`) |

Check order: `scope` → `no-name` → `bad-type` → `bad-gang` / `bad-user`.

**Response:** `{"status":"ok","data":{/* list object */}}`. User and team lists start with `status: true`, `default: false`; a global list (admin scope without `gang`/`user`) starts with both `false`.

**Errors:** `scope`, `no-name`, `bad-type`, `bad-gang`, `bad-user`, `db`

**Side effects:** owner's filter caches cleared; admin log entry only in `admin` scope. Configure mirror / feed fields afterwards with `edit`.

---

### blacklist/edit — Update list settings

```
POST api/blacklist/edit.json?id={token}[&scope=...]
```

Partial update: only fields present in the request change.

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID in the scope |
| `name` | string | New name; empty after cleaning → `bad-name` |
| `status` | bool | Active flag (send `0` / `1`; see Gotchas) |
| `default` | bool | Default flag |
| `key` | string | **Feed lists only:** access key. Ignored for other roles. |
| `auto` | bool | Feed lists: automatic update switch. Other lists: URL mirroring on/off. |
| `url` | string | **Not for feed lists.** Mirror source address; empty string clears it |
| `period` | int | **Not for feed lists.** Mirror update period, seconds |

**Mirror rules** (applied when any of `url`, `period`, `auto` is sent for a non-feed list; missing ones come from the stored row):

- `url` non-empty: at most 500 characters, valid `http`/`https` URL with a host, no user/password, port absent or 80/443, a numeric host must be a public IPv4 → else `bad-url`. `auto` on with empty `url` → `bad-url`.
- `period` must be one of the role's allowed periods (limits table). Not allowed with `auto` on → `bad-period`; not allowed with `auto` off → silently stored as `0`.
- `auto` on while the list is selected as an engine auto-block target in the filter settings → `target`.
- `auto` on while the owner already has 5 other mirrored lists (user/team roles) → `limit`.
- Refusals `bad-url`, `bad-period`, `target`, `limit` carry a localized `message`.
- When the URL changes or `auto` is turned on, the mirror's update state is reset. A mirror list becomes `locked` and the next cron run replaces its entries with the file.

**Response:** `{"status":"ok","data":{/* updated list object */}}`. If no recognized field was sent, the unchanged list is returned without writing.

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `bad-name`, `bad-url`, `bad-period`, `target`, `limit`, `db`

**Side effects:** campaign filter caches of the owner / team cleared; admin log entry only in `admin` scope. Locked and feed lists can be edited (only their entries are protected).

---

### blacklist/del — Delete a list

```
POST api/blacklist/del.json?id={token}&list={list_id}[&scope=...]
```

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID in the scope |

**Response:** `{"status":"ok","data":{"id":12}}`

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `db`

**Side effects:** deletes the list with **all** its entries; caches cleared; admin log entry in every scope. No lock check: feed and mirror lists can be deleted (a supplier feed list reappears with the next feed update). The ID is not removed from campaigns' selections; the runtime ignores IDs that no longer exist.

---

### blacklist/block — Add one entry (upsert)

```
POST api/blacklist/block.json?id={token}[&scope=...]
```

Adds one entry; if an entry with the same identity already exists it is updated instead. The list must be in the scope, have a valid kind and not be locked.

| Name | Kinds | Description |
|---|---|---|
| `list` | all | **Required.** List ID |
| `ip` | ip, net, ip6 | **Required.** Address / network address. Missing → `no-ip`; invalid or private/reserved → `bad-ip` |
| `size` | net, ip6 | **Required.** Prefix length (net 8–31, ip6 32–128), else `bad-size` |
| `value` | bl | **Required.** Header text / PCRE. Missing → `no-value`; empty, non-UTF-8 or > 250 chars → `bad-value` |
| `type` | bl | **Required.** Header ID 1–255, else `bad-type` |
| `regexp` | bl | `1` = PCRE with delimiters (default `0`) |
| `white` | bl | `1` = white rule (default `0`) |
| `comment` | all | Reason code 0–255 (default 0) |
| `ttl` | all | Expiry Unix timestamp, `0` = never (default 0) |

On a match only the fields present in the request are updated (`comment`, `ttl`; for `bl` also `value`, `type`, `regexp`, `white`) and extra copies of the same `bl` key are removed.

**Response:** `{"status":"ok","data":{/* entry object */, "added": true}}` — `added: true` for a new row, `false` for an update.

```json
{"status": "ok", "data": {"id": 301, "value": "HeadlessChrome", "type": 1, "regexp": false, "white": false, "comment": 9, "ttl": 0, "added": true}}
```

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `locked`, `no-ip`, `no-value`, `bad-size`, `bad-type`, `bad-ip`, `bad-value`, `db`

**Side effects:** the list `size` is recounted when an entry was added or twins removed. No cache reset (see Gotchas on network cache).

---

### blacklist/remove — Delete one entry

```
POST api/blacklist/remove.json?id={token}[&scope=...]
```

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID |
| `item` | int | **Required.** Entry ID (`id` of the entry object). Missing/0 → `no-item`; not in this list → `bad-item` |

**Response:** `{"status":"ok","data":{"id":301}}`

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `locked`, `no-item`, `bad-item`, `db`

**Side effects:** entry deleted, the list `size` recounted. To delete every entry of a list at once use `clean`.

---

### blacklist/change — Update one entry

```
POST api/blacklist/change.json?id={token}[&scope=...]
```

Partial update: fields not sent keep their stored values. The list must be in the scope, have a valid kind and not be locked.

| Name | Kinds | Description |
|---|---|---|
| `list` | all | **Required.** List ID |
| `item` | all | **Required.** Entry ID; `no-item` / `bad-item` |
| `ip` | ip, net, ip6 | New address / network start |
| `size` | net, ip6 | New prefix length |
| `value`, `type`, `regexp`, `white` | bl | As in `block` |
| `comment`, `ttl` | all | As in `block` |

If no entry field is present, the entry is returned unchanged without a write. Otherwise the row is built over the stored entry and validated like `block`; if it would duplicate another entry of the same list → `duplicate`.

**Response:** `{"status":"ok","data":{/* entry object */}}` (no `added` field).

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `locked`, `no-item`, `bad-item`, `bad-size`, `bad-type`, `bad-ip`, `bad-value`, `duplicate`, `db`

---

### blacklist/clean — Empty a list

```
POST api/blacklist/clean.json?id={token}&list={list_id}[&scope=...]
```

Deletes every entry of one list in place: the list keeps its ID, name, status, default flag and mirror settings, so the campaigns that selected it keep selecting it. The list must be in the scope (same scope rules as `del`), have a valid kind and not be locked.

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID in the scope. Query string or form field. |
| `gang` / `user` | int | Admin scope: owner selection (non-numeric → `bad-gang` / `bad-user`) |

**Response:** `{"status":"ok","data":{"id":12,"del":4210}}` — `id` is the list, `del` the number of entries deleted (`0` for an empty list). The list `size` is reset to `0`.

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `locked` (feed list or URL mirror — unlike `del`, which has no lock check), `db`

**Side effects:** only the entries of the list's kind are deleted. No cache is cleared and no admin log entry is written (see Gotchas on network cache: an IPv4 network may keep matching for up to 666 seconds).

---

### blacklist/import — Merge a CSV file into a list

```
POST api/blacklist/import.json?id={token}&list={list_id}[&scope=...]
Content-Type: multipart/form-data   (file field: import)
```

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** Target list in the scope, not locked. Query string or form field. |
| `import` | file | **Required.** CSV file |

**CSV format:** UTF-8 (leading BOM skipped), no header line (a header counts as one rejected row), blank lines skipped. Delimiter auto-detected among `,` `;` TAB; quote `"`. Columns by position, missing trailing columns = 0:

| Kind | Columns |
|---|---|
| `ip` | `ip, comment, ttl` |
| `net`, `ip6` | `ip, size, comment, ttl` |
| `bl` | `value, type, regexp, comment, ttl, white` |

Rows are validated like `block`; an invalid row is counted in `bad` and skipped, it does not abort the import. The format is exactly what `export` produces.

**Merge:** matched entries (same identity) are updated with the file's `comment` / `ttl` (for `bl` also `value`, `type`, `regexp`, `white`; columns absent from the row overwrite with 0), others are added. Nothing is deleted, except stored duplicate twins of a `bl` key present in the file (counted in `del`). A duplicate row inside the file counts as an update.

**Response:**

```json
{"status": "ok", "data": {"add": 120, "upd": 5, "bad": 2, "del": 0, "size": 4210}}
```

`add` added, `upd` updated, `bad` rejected, `del` twins removed (always 0 for `ip`/`net`/`ip6`), `size` resulting list size (also stored).

**Errors:** `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`, `locked`, `limit` (upload exceeds the PHP upload limit or the role's size limit), `no-file` (no uploaded file), `bad-file` (cannot be read), `db`

A file without a single valid row is not an error (`add: 0`, `bad: N`).

---

### blacklist/export — Download a list as CSV

```
GET api/blacklist/export.json?id={token}&list={list_id}[&scope=...]
```

| Name | Type | Description |
|---|---|---|
| `list` | int | **Required.** List ID in the scope |

**Response:** not a JSON envelope. `Content-type: text/csv`, `Content-disposition: attachment;filename=filter-{kind}-{list_id}.csv`; one line per entry, comma-separated, no header line, no paging, no ordering. The `.json` suffix is ignored.

| Kind | Line format |
|---|---|
| `ip` | `<ip>,<comment>,<ttl>` |
| `net`, `ip6` | `<ip>,<size>,<comment>,<ttl>` |
| `bl` | `"<value, quotes doubled>",<type>,<regexp 0/1>,<comment>,<ttl>,<white 0/1>` |

The output is exactly what `import` reads and what a URL mirror expects.

**Errors** (JSON, returned before any CSV header): `scope`, `bad-gang`, `bad-user`, `no-list`, `bad-list`. Works for locked and feed lists.

---

## Error codes

| Error | Raised by | Meaning |
|---|---|---|
| `scope` | all | Scope could not be resolved (banned/inactive user, non-admin asked `admin`, non-team-admin asked `gang`, foreign `user`) |
| `bad-gang` / `bad-user` | all, admin scope | `gang` / `user` not numeric, or (in `avail`, `add`) the team / user does not exist |
| `no-site` / `no-user` | `avail` | Campaign `uid` not found in the scope / no `uid` and the scope has no user |
| `no-name` / `bad-name` | `add` / `edit` | Empty name |
| `bad-type` | `list`, `add`, `show`, `block`, `change` | List kind not 1–4 / header ID not 1–255 or not numeric |
| `no-list` | functions with `list` | `list` missing or 0 |
| `bad-list` | functions with `list` | List not found, not in the scope, or unknown kind |
| `locked` | `block`, `remove`, `change`, `clean`, `import` | List is a feed or a URL mirror |
| `bad-search` / `bad-comment` | `show` | Invalid `search` address / `comment` not 0–255 |
| `no-ip` / `no-value` | `block` | Required entry field missing |
| `bad-ip` / `bad-value` / `bad-size` | `block`, `change` | Entry address (private/reserved included), value or prefix length invalid |
| `no-item` / `bad-item` | `remove`, `change` | Entry ID missing / not in the list |
| `duplicate` | `change` | The edit would duplicate another entry |
| `bad-url` / `bad-period` / `target` / `limit` | `edit` (with `message`) | Mirror settings refused |
| `limit` / `no-file` / `bad-file` | `import` | File too large / no uploaded file / unreadable file |
| `db` | add, edit, del, block, change, remove, clean, import | Database write failed (HTTP 500) |

---

## Gotchas

- **Global lists are created inactive.** `add` in `admin` scope without `gang`/`user` stores `status: false`, `default: false`. Turn them on with `edit` (`status=1`, optionally `default=1`); inactive lists are not offered by `avail` and cannot be chosen in campaigns. User and team lists start active.
- **`type` means different things.** In `list`/`add` it is the list kind (1–4); in `show`, `block`, `change` and in `bl` entries it is the header ID (1–255; `0` in `show` = no filter). `show`'s `type` filter is ignored for kinds other than `bl`.
- **`ttl` is an absolute Unix timestamp**, not a duration; `0` = never. Expired entries are deleted by a minute cron, and stay effective until then.
- **Network cache delay.** IPv4 network lookups are cached per /24 and list set for 666 seconds; an entry added through `block`/`import` may take effect with that delay for recently checked addresses. No cache reset happens on entry writes.
- **`import` is merge-only.** It never removes entries (except `bl` twins) and overwrites `comment`/`ttl` of matches with the file's values, absent columns as 0. To replace a list, `clean` it and `import` the new file (the list keeps its ID), or `remove` entries one by one. There is no row limit on import.
- **Size limits depend on the list's role, not the scope:** 8 MiB import files for personal/team lists, 256 MiB for global lists; mirror period set, row cap and 5-mirror quota likewise (limits table).
- **`gang` scope needs a team admin;** a team member who is not an admin gets `scope`. `avail` in `gang` scope without `uid` also needs `user=<member>`. The API ignores the UI-only "shared team blacklists" flag.
- **Boolean quirk in `edit`.** `status`, `default`, `auto` are read as loose booleans: `0`, empty and `"0"` mean off, everything else — including the string `"false"` — means on. Send `0` / `1`.
- **`scope` is read from GET only,** also for JSON POST bodies. Array values in any field count as empty/invalid.
- **`block` is an upsert** (`added: false` on update); `change` refuses duplicates with `duplicate`. For `bl`, identity is `type` + `value`, case-insensitive unless `regexp` is set.
- **Locked lists:** feed and mirror lists reject `block`/`remove`/`change`/`clean`/`import` with `locked`, but `show`, `export`, `edit` and `del` still work.
- **Mirror on = replaced content.** Turning on `auto` with a `url` makes the list `locked`, and the next cron run replaces its entries with the file.
- **`export` ignores the `.json` suffix** and returns CSV; errors come back as JSON before any CSV header.
