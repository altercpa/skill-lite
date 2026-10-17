# Profile & System API Reference

## profile/* — User Account

The `profile` app covers user authentication, profile management, API key reset, registration, and password recovery.

Base URL: `api/profile/{func}.json`

**Auth split:**
- `profile/info`, `profile/edit` and `profile/reset` require `?id={token}`
- `profile/login`, `profile/register`, `profile/recover` do NOT require a token

---

### profile/login — Get current session user

```
GET api/profile/login.json
```

Returns the user from the current session (cookie-based). No API token required.

**Special:** POSTing `in_user=email&in_pass=password` to this endpoint triggers a credential login — establishes a session as a side effect.

**Success response:**
```json
{
  "status": "ok",
  "id": 13,
  "api": "4a3bc9d...",
  "name": "John",
  "email": "john@example.com",
  "ban": false
}
```

Note: `ban` is boolean (not int like in other endpoints).

**Errors:** `bad-auth` (no active session)

---

### profile/info — Get profile data

```
GET api/profile/info.json?id={token}
```

No parameters beyond auth. Admin can use `uu=N` to read another user's profile.

**Success response:**
```json
{
  "status": "ok",
  "id": 13,
  "api": "4a3bc9d...",
  "name": "John",
  "email": "john@example.com",
  "access": {
    "level": 0,
    "vip": 0,
    "ban": 0,
    "warn": 0
  },
  "info": {
    "reg": 1690000000,
    "tz": "Europe/Moscow",
    "lang": "en",
    "avatar": "https://tracker.example.com/data/avatar/13.jpg"
  },
  "meta": {
    "lang": "en",
    "skin": "default",
    "avatar": true,
    "telegram": "john"
  },
  "gang": {
    "id": 3,
    "name": "Meow team",
    "admin": true,
    "boss": false,
    "make": true
  },
  "contacts": [
    { "code": "telegram", "name": "Telegram", "icon": "telegram", "type": 1, "url": "https://t.me/%s", "reg": false }
  ],
  "tz": {
    "site": "Europe/Moscow",
    "user": "Europe/Berlin",
    "utz": true
  }
}
```

Field notes:
- `access.level > 0` = admin user.
- `info.tz` — the user's personal time zone, `false` when unset. `info.lang` — language code, `false` when unset.
- `info.avatar` — absolute URL of the 100×100 avatar when the user uploaded one, `false` otherwise (no default-avatar URL is returned).
- `meta` — all stored user meta keys (lang, skin, avatar flag, contact fields).
- `gang` — the user's team card. `id` is `0` (and `name`/`admin`/`boss` are `false`) when the user has no team. `admin` — the user is a team admin; `boss` — the user is the team owner (its creator); `make` — `false` when team creation is disabled on this tracker.
- `contacts` — the contact fields configured on this tracker (the same codes that appear as keys in `meta` and that `profile/edit` accepts). Entry: `code`; `name` (label in the caller's language); `icon`; `type` (`0` simple value, `1` social profile, `2` URL link, `3` value with label); `url` (link template where `%s` stands for the stored value, `false` when the field is not a link); `reg` (required at registration).
- `tz` — `site` is the tracker time zone (always a valid identifier); `user` is the user's personal zone or `false`; `utz` is `true` when personal time zones are enabled on this tracker (only then `profile/edit` saves `tz`).

**Errors:** None from the function. Non-existent `uu` target returns zeros silently.

---

### profile/edit — Edit profile

```
POST api/profile/edit.json?id={token}
```

A **partial update**: only the parameters you send are changed; everything omitted stays as it is. Send the request as form-encoded or JSON; for an avatar upload use `Content-Type: multipart/form-data`.

**Parameters (POST wins over GET):**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | no | Display name. If sent, it must not be empty (else `info`). |
| `email` | string | no | New email (validated, must be unique). |
| `pass` | string | no | New plaintext password. Applied only together with a matching `conf`. |
| `conf` | string | no | Password confirmation (must match `pass` exactly). |
| `tz` | string | no | Time-zone identifier; an empty string clears the personal zone. Considered only when personal time zones are enabled (`tz.utz` in `profile/info`); otherwise silently ignored. An unknown identifier → `bad-tz`. |
| `lang` | string | no | Language code. |
| `skin` | string | no | Skin code. |
| contact fields | string | no | Any contact code from `profile/info` → `contacts[].code` (e.g. `telegram`). An empty value clears the field. |
| `unavatar` | any | no | Truthy = clear avatar |
| `avatar` | file | no | Avatar file (jpg/jpeg/png/gif/webm/webp/svg/avif). Saved as 100×100 and 40×40 JPG. |

**Warnings:**
- Omitting `lang`/`skin`/`tz`/`name`/contact fields leaves them unchanged.
- A duplicate or invalid `email` is silently ignored (no error, no update — `status: ok` without `data.mail`). A `pass` without a matching `conf` is silently ignored too.
- Avatar upload requires `multipart/form-data`; upload failures are silently ignored.
- The password (hash) is never returned.

**Success response:**
```json
{
  "status": "ok",
  "data": { "name": "John" },
  "meta": { "lang": "en", "telegram": "john" }
}
```

`data` contains the user-row fields (`name`, `tz`, `mail`) and `meta` the meta keys (`lang`, `skin`, contact codes — `false` when cleared, `avatar` — `true`/`false`) that were accepted from this request, each with the value now stored — even when it equals the value the account already had. Only `mail` is compared with the current address: it is returned only when it is a new, unused address. A new password is saved but never echoed, so a request that sets only the password also returns empty `data` and `meta`. When nothing valid was sent, both are empty (`[]`) and nothing is saved.

**Errors:** `info` (`name` sent but empty), `bad-tz` (unknown time-zone identifier), `bad-id` (user not found, e.g. a bad `uu`), `save` (update failed)

---

### profile/reset — Reset the API key

```
GET api/profile/reset.json?id={token}
```

Generates a new API key for the account. The old key stops working immediately, so every later request must use the new token. Admin can use `uu=N` to reset another user's key.

**Success response:**
```json
{
  "status": "ok",
  "id": 13,
  "api": "9f1c2e7a4b6d4e0f8a3b5c7d9e1f2a4b",
  "token": "13-9f1c2e7a4b6d4e0f8a3b5c7d9e1f2a4b"
}
```

`token` is the ready-to-use value of the `id` request parameter (`{id}-{api}`).

**Errors:** `bad-id` (user not found; HTTP 404), `save` (update failed; HTTP 400)

---

### profile/register — Register new user

```
POST api/profile/register.json
```

No token required. Admin caller (passing a valid `?id=` token of an admin account) bypasses registration gates.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `email` | string | yes | Email address (must be unique) |
| `name` | string | yes | Display name (minimum 4 characters) |
| `pass` | string | yes | Password (minimum 4 characters) |
| `ref` | int | no | Referrer user ID (validated; not stored by default) |
| `man` | int | no | Manager user ID. The user must have the Manager or Business staff role, otherwise the value is ignored; not stored by default |

Contact meta fields configured in the tracker's registration settings may also be required.

**Errors:**
- `disabled` — registration disabled or captcha required (both collapse to same code)
- `email-exist` — email already used (checked before the password and name length checks)
- `short-pass` — password under 4 characters
- `short-name` — name under 4 characters
- `required` — required contact meta field missing
- `db` — saving failed

**Success response:**
```json
{
  "status": "ok",
  "id": 42,
  "api": "4a3bc9d...",
  "name": "John",
  "email": "john@example.com"
}
```

**Notes:**
- No session is created — caller must call `profile/login` (with credentials) or use the returned `id`/`api` directly.
- No welcome email is sent.

---

### profile/recover — Request password recovery

```
POST api/profile/recover.json
Body: email=john@example.com
```

No token required.

**Parameters:** `email` (POST first, then GET).

**Success response:** `{"status":"ok"}`

**Errors:**
- `no-email` — empty or invalid email
- `bad-email` — no user found with this email

**Side effects:** Sends a recovery email with a link `/?recoverpass={token}`. Mail failures are silently ignored (`status: ok` regardless).

---

## sys/* — System Endpoints

The `sys` app provides system-level introspection. `ping` and `users` require a token; `settings` needs none.

`sys/users` is available to admin accounts only: any other account gets `auth` (HTTP 401).

---

### sys/ping — Liveness check

```
GET api/sys/ping.json?id={token}
```

**Success response:** `{"status":"ok","user":13}`

**Note:** This call is NOT written to the API request log (intentional, by design).

**Errors:** Framework only (`key`, `ban`, `bot`).

---

### sys/users — User list/search

```
GET api/sys/users.json?id={token}[&filters...]
```

**Parameters (GET only):**

| Name | Type | Description |
|---|---|---|
| `page` | int | Page (default 1) |
| `show` | int | Page size (default 100, minimum 1, no upper limit) |
| `uid` | int | Filter by exact user ID |
| `gang` | int | Filter by gang ID |
| `mail` | string | Filter by exact email |
| `level` | int | Filter by admin level (0=regular, 1=admin) |
| `vip` | int | Filter by VIP flag |
| `ban` | int | Filter by banned flag |
| `warn` | int | Filter by warn flag |
| `search` | string | Full-text search in name and email (min 2 chars) |
| `ms` | string | Meta-search key |
| `mv` | string | Meta-search value (requires `ms`; substring match then exact post-filter) |
| `short` | any | Short shape: only id, name, mail, api |

**Full shape response:**
```json
{
  "status": "ok",
  "page": 1,
  "show": 100,
  "total": 234,
  "data": [
    {
      "id": 13,
      "name": "Anton",
      "mail": "user@example.com",
      "api": "654b807372f614e64b9...",
      "lang": "ru",
      "signs": {
        "register": 1690000000,
        "ip": "12.34.56.78",
        "last": "2024-08-21"
      },
      "auth": {
        "admin": false,
        "vip": false,
        "ban": false,
        "warn": false,
        "gang": 0
      },
      "meta": { "custom_key": "value" }
    }
  ]
}
```

**Short shape** (when `short` param present): each entry has only `id`, `name`, `mail`, `api`.

**Notes:**
- `total` is computed before meta-search post-filtering — may be higher than actual returned count when `ms`/`mv` used.
- `meta` excludes `lang`, `graph`, `nograph`, `pbs` keys (housekeeping). Does NOT include the password hash.

**Errors:** `auth` (HTTP 401) — the account behind the token is not an admin account; plus the framework errors (`key`, `ban`, `bot`).

---

### sys/settings — Tracker timezone

```
GET api/sys/settings.json
```

**No token required.**

**Success response (no status envelope):**
```json
{"timezone":"Europe/Moscow"}
```

Note: Response has NO `status` field — just the raw object. Do not expect `{"status":"ok",...}` shape here.

**Errors:** None from the function (framework `bot`/`request` codes still possible).
