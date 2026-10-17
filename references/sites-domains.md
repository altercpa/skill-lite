# Sites & Domains API Reference

## Sites Storage

The `sites` app manages landing page storage: listing, uploading, editing, deleting, and downloading sites. The API performs scope checks and ownership validation and returns unified JSON responses.

Base URL: `api/sites/{func}.json?id={token}[&scope=...]`

**Billing:** every `sites/*` function except `scope=admin` requires the caller's plan to allow it (checked against the caller, not the impersonated team member) — otherwise `unpaid`. For `scope=admin` no billing check is performed at all.

### Site Object

```json
{
  "id": 42,
  "user": 7,
  "gang": 0,
  "glue": 0,
  "name": "Insurance landing",
  "path": "ab/cm6gh1"
}
```

| Field | Description |
|---|---|
| `id` (or `uid` in list) | Site ID |
| `user` | Owner user ID (0 for gang sites) |
| `gang` | Owner gang ID (0 for personal sites) |
| `glue` | For gang sites: visibility marker. 0=hidden, -1=public, >0=specific member ID |
| `name` | Site name |
| `path` | Path on storage server in format `{2hex}/{enc-id}` |

---

### sites/list — List sites

```
GET api/sites/list.json?id={token}[&scope=...]
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `scope` | string | `user`/`gang`/`admin` |
| `user` | int | Filter by user (gang/admin scope) |
| `gang` | int | Filter by gang (admin scope) |
| `uid` | int | Exact site ID match |
| `glue` | int | Filter by the visibility marker (team sites) |
| `offer` | int | Filter by offer ID (reserved, rarely populated) |
| `search` | string | Search in site name |
| `page` | int | Page number (≥1) |
| `show` | int | Page size (0 = all) |

**Response:**
```json
{
  "status": "ok",
  "total": 5,
  "page": 1,
  "show": 0,
  "data": {
    "42": { "uid": 42, "user": 7, "gang": 0, "glue": 0, "name": "Site", "path": "ab/cm6gh1" }
  }
}
```

Note: `data` uses `uid` (not `id`) as the per-record ID key.

**Errors:** `scope`, `unpaid`

**Note:** with `scope=admin` and neither `user` nor `gang` supplied, there is no owner filter at all — the list spans every user's and team's sites. Unlike `domains/list`, admin scope here does not implicitly restrict by team.

---

### sites/add — Upload site

```
POST api/sites/add.json?id={token}&scope=user
```

**Content-Type:** `multipart/form-data` for ZIP upload; otherwise `application/x-www-form-urlencoded` or JSON.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Site name |
| `archive` | file | no | ZIP archive (field `archive` in multipart) |
| `url` | string | no | URL for mirror download via httrack |
| `scope` | string | no | `user`/`gang`/`admin` |
| `user` | int | admin | Owner user ID |
| `gang` | int | admin | Owner gang ID |
| `glue` | int | gang/admin | Visibility marker for gang site (0=hidden, -1=public, >0=member ID) |

**Priority:** If both `archive` and `url` provided, `archive` wins. If neither — empty folder created on storage.

**Scope rules:**
- `user`: creates personal site for current user (cannot create a gang site this way, even for team leads)
- `gang`: creates team site (without `glue` the visibility marker is 0)
- `admin` with `gang`: creates gang site; `glue` sets visibility member
- `admin` with `user`: creates personal site for that user
- `admin` without either → `user-required`

**Response:** `{"status":"ok","data":{"id":42,"user":7,"gang":0,"glue":0,"name":"...","path":"..."}}`

Note: `id` (not `uid`) in response of `add`.

**Errors:** `scope`, `unpaid`, `user-required`, `wrong-glue` (glue member not in gang), `add` (empty name, or neither user nor gang resolved)

**Important:** The storage upload result is NOT checked — the site record is always created on success, regardless of whether storage is reachable. If storage is unavailable or misconfigured, the record exists but no files appear on the storage server.

---

### sites/edit — Edit site

```
POST api/sites/edit.json?id={token}&uid={site}
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `uid` or `sid` | int | yes | Site ID (POST uid > GET uid > POST sid > GET sid) |
| `name` | string | no | New site name |
| `glue` | int | no | New visibility marker (gang/admin only) |
| `archive` | file | no | New ZIP archive (replaces content) |
| `url` | string | no | New mirror URL (replaces content) |

**Scope checks:**
- `user`: site must be personal (not a team site) and owned by current user; otherwise `auth`
- `gang`: site must be team site of same gang; otherwise `auth`. If `glue` is passed, the member must belong to the same gang, otherwise `wrong-glue`.
- `admin`: no ownership check; `glue` (if a gang site) updates the visibility marker, checked with `wrong-glue`

**Response:** Same object as `add`.

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `auth`, `wrong-glue`, `edit` (the save failed: a database error, or the site has no owner)

**Note:** Cannot change owner of a personal site. For team sites the owner is the team and cannot change — only the visibility marker can change via `glue`. If `archive`/`url` is present, storage content is completely overwritten (folder is cleaned and refilled); if neither is set, only `name`/`glue` change and no storage request is sent. A call without any editable field still succeeds and changes nothing; an empty `name` is ignored.

---

### sites/del — Delete site

```
POST api/sites/del.json?id={token}&uid={site}
```

**Parameters:** `uid` or `sid`.

**Scope checks:** Same as `edit`. In admin scope, ownership is taken from the record itself (no separate check).

**Response:** Same site object.

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `auth`, `del`

**Side effects:**
- Deletes the site record
- Sends `remove` request to storage server (folder deleted)
- **Does NOT clean up** the references to the site held by ways (`site`, second-step `nxsite`), flows (traffic-back `site`) and the stored clicks and leads — they may keep pointing at the deleted site ID

**Note:** Storage removal result is not checked — the site record is deleted even if storage is unavailable, leaving orphaned files on the storage server.

---

### sites/download — Download site as ZIP

```
GET api/sites/download.json?id={token}&uid={site}
```

**Parameters:** `uid` or `sid`.

**Scope checks:**
- `admin`: no checks (can download any site)
- `gang`: must be gang site of same gang
- `user`: must be personal site

**Response (success):** NOT JSON. Binary ZIP file with headers:
```
Content-type: application/octet-stream
Content-Disposition: attachment; filename=site.zip
```

**Response (error):** JSON as usual.

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `auth`, `download` (storage unreachable / ZIP not produced)

---

## Parked Domains

The `domains` app manages parked domains for flows: listing, adding, manual status checks, deletion, attaching a domain to a flow, managing per-domain confirmation files served at the domain root, and getting the parking IPs and DNS records to point a domain at.

Base URL: `api/domains/{func}.json?id={token}[&scope=...]`

**Billing:** `domains/list` and `domains/add` require the caller's plan to allow it (except `scope=admin`) — otherwise `unpaid`. `check`/`del`/`glue` do not check billing at all (only the ownership check). The four confirmation-file functions (`files`/`confirm`/`change`/`remove`) and `dns` also do **not** check billing.

### Domain Object (list/check responses)

```json
{
  "uid": 42,
  "url": "example.com",
  "addr": "example.com",
  "ip": "203.0.113.10",
  "user": 7,
  "gang": 0,
  "flow": 18,
  "status": 2,
  "stage": "ok",
  "time": 1714723200,
  "created": 20210914,
  "expires": 20260914,
  "isok": 1
}
```

| Field | Description |
|---|---|
| `uid` | Domain ID |
| `url` | Human-readable (after IDNA decode) |
| `addr` | Punycode form |
| `ip` | Current IP string |
| `user` | Owner user ID (0 for team domain) |
| `gang` | Owner gang ID (0 for personal) |
| `flow` | Attached flow ID (0 = not attached) |
| `status` | 0=Waiting, 1=Active, 2=OK+SSL, 3=Error, 4=Wrong! |
| `stage` | String: `wait`, `active`, `ok`, `error`, `wrong` |
| `time` | Unix timestamp when added |
| `created` | WHOIS registration date (YYYYMMDD int) |
| `expires` | WHOIS expiration date (YYYYMMDD int) |
| `isok` | 0=no data, 1=delegated, 2=has issues |

**Team vs personal domains:**
- Personal: `user` > 0, `gang` = 0
- Team (shared): `user` = 0, `gang` > 0

In `gang` scope, only team domains are accessible — personal domains of team members are NOT accessible. A team domain is shared by the whole team, not tied to a specific member.

### File Object (files/confirm/change/remove responses)

A confirmation file is a small text/markup file served at a parked domain's root (e.g. `google1234567890.html`, `ads.txt`) to prove domain ownership to search engines/ad networks, or to serve a static declaration file. A file belongs to a user (`user` > 0, `gang` = 0) or a team (`user` = 0, `gang` > 0), copied from the parent domain's owner **at creation time** — it does not track later domain re-glue.

```json
{
  "id": 8,
  "dom": 42,
  "user": 0,
  "gang": 3,
  "name": "google1234567890.html",
  "data": "google-site-verification: google1234567890.html"
}
```

| Field | Description |
|---|---|
| `id` | File ID |
| `dom` | Parent domain ID |
| `user` | Owner user ID (0 for team file) |
| `gang` | Owner gang ID (0 for personal file) |
| `name` | File name at domain root, e.g. `ads.txt` |
| `data` | Raw file content (max ~3500 chars, trimmed/truncated server-side). Omitted from `domains/remove`'s response. |

Allowed file-name extensions: `txt`, `js`, `json`, `htm`, `html`, `css`, `xml` (case-insensitive, lowercased). Any other extension — or a name that becomes empty after stripping characters outside `[A-Za-z0-9._-]` and trimming leading/trailing `.`/`-`/`_` — is rejected.

A file name is unique per domain: `domains/confirm` therefore behaves as an **upsert** (posting the same `name` for the same domain again overwrites the existing file instead of erroring or duplicating).

---

### domains/list — List domains

```
GET api/domains/list.json?id={token}[&scope=...]
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `scope` | string | `user`/`gang`/`admin` |
| `gang` | int | Filter by gang (admin scope) |
| `flow` | string | Symbolic flow code |
| `fid` | int | Numeric flow ID |
| `url` | string | Substring in human-readable domain |
| `addr` | string | Substring in punycode domain |
| `ip` | string | IP address |
| `status` | int | Status code (0–4) |
| `uid` | int | Exact domain ID match |
| `page` | int | Page (≥1) |
| `show` | int | Page size (0 = all) |

`scope=admin` with `user` filters by the owner user; with `gang` filters by the owner team; with neither, no owner filter is applied at all.

**Response:**
```json
{
  "status": "ok",
  "total": 12,
  "page": 1,
  "show": 0,
  "data": { "42": { /* domain object */ } }
}
```

**Errors:** `scope`, `unpaid`

**Note:** the client must convert `flow` (numeric flow ID) to a symbolic code itself if needed; `domains/glue` does this conversion server-side (`code` field).

---

### domains/add — Add domain

```
POST api/domains/add.json?id={token}&url=example.com
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `url` | string | yes | Domain name (protocol stripped, `www.` removed) |
| `scope` | string | no | `user`/`gang`/`admin` |
| `user` | int | admin | Owner user ID |
| `gang` | int | admin | Owner gang ID |

**Scope rules:**
- `gang` (or `admin`+`gang`): domain created as team domain (owned by the team, `user` = 0)
- `admin` with `user` (no `gang`): personal domain for that user
- `admin` without user/gang → `no-user`

**Response:**
```json
{
  "status": "ok",
  "data": {
    "id": 137,
    "user": 0,
    "gang": 3,
    "url": "example.com",
    "addr": "example.com"
  }
}
```

**Errors:** `scope`, `no-user`, `unpaid`, `no-url`, `bad-url` (invalid name after cleanup), `duplicate` (a domain with the same `addr` already exists), `bad-domain` (IP address, system domain, invalid name after IDNA, or saving failed)

**Note:** Domain check is NOT launched immediately — waits for the scheduled check (~11.7h cycle) or an explicit `domains/check` call. Only one domain per call (bulk add exists in the UI only).

---

### domains/check — Manual domain status check

```
GET api/domains/check.json?id={token}&uid={dom_id}
GET api/domains/check.json?id={token}&domain=example.com
```

**Parameters:** `uid` (domain ID) or `url`/`domain` (domain name).

**Response:** Full domain object (same shape as a `list` entry).

**Side effects:**
- HTTPS then HTTP request to the domain (cURL, 5s timeout, up to a few attempts)
- Runs a `whois` lookup
- Updates the domain's status and check times and (manual check only) its WHOIS data (`created`, `expires`, `isok`, raw WHOIS text)
- Notifications are **not** sent for manual checks

**Errors:** `not-found` (domain not found OR no permission for the given scope — indistinguishable to the client)

**Note:** Can take up to ~30 seconds if the domain doesn't respond on either protocol.

---

### domains/del — Delete domain

```
POST api/domains/del.json?id={token}&uid={dom_id}
```

**Parameters:** `uid` or `url`/`domain`.

**Response:**
```json
{
  "status": "ok",
  "data": { "id": 42, "user": 0, "gang": 3, "url": "example.com", "addr": "example.com" }
}
```

**Side effects:**
- Deletes the domain record
- Deletes the domain's confirmation files — **all** files of the domain, not just some
- Detaches the domain from all flows that had it attached (their `domain` becomes 0)

**Errors:** `not-found`, `bad-domain`

---

### domains/glue — Attach domain to flow

```
POST api/domains/glue.json?id={token}&uid={dom_id}&flow={flow_code}
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `uid` | int | Domain ID |
| `url`/`domain` | string | Domain name (alternative to uid) |
| `flow` | string | Symbolic flow code to attach |
| `fid` | int | Numeric flow ID (used if `flow` is empty or fails to decode) |

To detach: pass `flow=0` or omit both `flow` and `fid`.

**Ownership rules:**
- Personal domain: flow must belong to the same user
- Team domain: flow must belong to the same team

**Response:**
```json
{
  "status": "ok",
  "data": {
    "id": 42,
    "user": 0,
    "gang": 3,
    "url": "example.com",
    "addr": "example.com",
    "flow": 18,
    "code": "ef1c"
  }
}
```

On detachment: `flow: 0`, `code: false`.

**Errors:** `not-found`, `glue` (flow belongs to a different owner, or flow not found)

**Note:** `check`, `del`, `glue`, `files` and `confirm` answer `not-found` in all three cases: the domain does not exist, it is outside the caller's scope, or the scope is invalid.

---

### domains/files — List confirmation files

```
GET api/domains/files.json?id={token}&uid={dom_id}
GET api/domains/files.json?id={token}&domain=example.com
```

Returns every confirmation file stored at a domain's root, with full content. No pagination — the entire list is always returned (files are capped at 3500 chars and an allowed extension, so counts stay small in practice).

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `uid` | int | one of | Domain ID |
| `url`/`domain` | string | one of | Domain name |

Domain identification/ownership — same as `check`/`del`/`glue`.

**Response:**
```json
{
  "status": "ok",
  "total": 2,
  "data": {
    "7": { "id": 7, "dom": 42, "user": 0, "gang": 3, "name": "ads.txt", "data": "google.com, pub-0000000000000000, DIRECT" },
    "8": { "id": 8, "dom": 42, "user": 0, "gang": 3, "name": "google1234567890.html", "data": "google-site-verification: google1234567890.html" }
  }
}
```

`data` is keyed by file ID.

**Errors:** `not-found` (domain not found or no permission for the given scope)

**Note:** No billing check.

---

### domains/confirm — Create/replace a confirmation file

```
POST api/domains/confirm.json?id={token}&uid={dom_id}&name=ads.txt&data=...
```

Creates a new confirmation file at the domain root, or **silently overwrites** an existing file of the same `name` on the same domain (upsert on domain + file name).

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `uid` | int | one of | Domain ID (identifies the **domain**, same convention as `check`/`del`/`glue`) |
| `url`/`domain` | string | one of | Domain name |
| `name` | string | yes | File name at domain root, e.g. `ads.txt`. Allowed extensions: `txt`/`js`/`json`/`htm`/`html`/`css`/`xml` |
| `data` | string | no | File content (trimmed, truncated to 3500 chars; empty allowed) |

**Response:**
```json
{
  "status": "ok",
  "data": { "id": 8, "dom": 42, "user": 0, "gang": 3, "name": "google1234567890.html", "data": "google-site-verification: google1234567890.html" }
}
```

**Errors:**
- `not-found` — domain not found or no permission for the given scope
- `no-name` — `name` parameter missing/empty (raw check, before extension validation)
- `bad-file` — invalid file name/extension after cleanup, or saving failed

**Note:** No `duplicate` error on repeat `name` — unlike `domains/add`, `confirm` overwrites silently. To get strict create-only semantics, check `domains/files` first. No billing check. File ownership (`user`/`gang`) is copied from the domain's current owner at creation time.

---

### domains/change — Modify an existing confirmation file

```
POST api/domains/change.json?id={token}&fid={file_id}&name=ads.txt&data=...
```

Renames and/or rewrites the content of an existing file, identified **directly by its numeric file ID** (`fid`) — not by domain.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `fid` | int | one of | File ID |
| `file` | int | one of | Alias for `fid` (tried second; numeric ID here, unlike `url`/`domain`'s name aliases elsewhere) |
| `name` | string | yes | New file name |
| `data` | string | no | New file content |

**Response:**
```json
{
  "status": "ok",
  "data": { "id": 8, "dom": 42, "user": 0, "gang": 3, "name": "google1234567890.html", "data": "google-site-verification: google1234567890.html (updated)" }
}
```

**Errors:**
- `not-found` — file not found (bad `fid`/`file`) or no permission for the given scope (checked against the **file's own** owner (`user`/`gang`), not the domain's)
- `no-name` — `name` parameter missing/empty
- `bad-file` — invalid file name/extension, a rename that collides with another file of the same domain, or a save error

**Note:** A plain update, not an upsert — a rename colliding with an existing file name on the same domain fails with `bad-file` rather than merging. No billing check.

---

### domains/remove — Delete a confirmation file

```
POST api/domains/remove.json?id={token}&fid={file_id}
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `fid` | int | one of | File ID |
| `file` | int | one of | Alias for `fid` (tried second) |

**Response:**
```json
{
  "status": "ok",
  "data": { "id": 8, "dom": 42, "user": 0, "gang": 3, "name": "google1234567890.html" }
}
```

Note: unlike `files`/`confirm`/`change`, the response has **no `data` field for file content** — the deleted content is not echoed back.

**Errors:**
- `not-found` — file not found or no permission for the given scope
- `bad-file` — the deletion failed (ownership-mismatch race or a save error; rare since existence and ownership were already validated)

**Side effects:** Deletes exactly one file; does not touch the domain or its flow attachment, and does not affect the domain's other files. Unlike `domains/del` (which cascades to all files of a domain), `remove` is single-file.

**Note:** No billing check.

---

### domains/dns — Parking IPs and DNS records

```
GET api/domains/dns.json?id={token}[&scope=gang|admin][&user={user_id}]
```

Returns the parking IP addresses and the ready DNS records to create at the registrar so that a domain points to the tracker. Read-only: no domain is added or changed; no billing check.

The addresses depend on the parking setup of the user: a user who belongs to a team gets the addresses of the team setup.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | `user` (default), `gang`/`team`, `admin`/`sys`. `gang`: the setup of the caller's own account is used; a `user` that is given must be a member of your team (else `scope`) and does not change the result. |
| `user` | int | for `scope=admin` | User whose parking addresses are returned. Without it `scope=admin` answers `no-user`. |

**Response:**
```json
{
  "status": "ok",
  "data": {
    "ips": ["203.0.113.10", "2001:db8::10"],
    "records": [
      { "name": "@", "type": "A", "value": "203.0.113.10" },
      { "name": "@", "type": "AAAA", "value": "2001:db8::10" }
    ]
  }
}
```

- `ips` — list of parking IP addresses (IPv4 and IPv6) as strings; `[]` when none.
- `records` — DNS records to create; `[]` when there are no addresses. Each record is `{name, type, value}`: `name` is always `@` (domain root), `type` is `A` (IPv4) or `AAAA` (IPv6). At most one `A` and one `AAAA` record are returned; when the setup holds several addresses a random one is picked on every call.

**Errors:** `scope` (workspace not available: not a team leader, not an administrator, the account is blocked, or with `scope=gang` a `user` who is not a member of your team; HTTP 400), `no-user` (`scope=admin` without `user`; HTTP 400).

---

## See also

Storage-facing utility endpoints `api/tracker/checkdom` and `api/tracker/domains` (used by the storage server for Nginx/certbot domain sync, no `/api/` auth) are documented in `references/tracker.md`.
