# Ad-Spend Connectors API Reference (`adv`)

## Overview

The `adv` app manages **ad-spend export connectors** — background integrations that
periodically pull ad-network spend statistics (Facebook, TikTok, BIGO Ads, VK, MGid,
LuckyAds, Dolphin/Dolphin Cloud, FBtool, MixAdvert, AdPartner) and write per-click cost
into the clicks stored by the tracker.

Base URL: `api/adv/{func}.json?id={token}[&scope=...]`

**No billing gate:** unlike `sites`/`flows`, no `adv` function calls the paywall check —
there is no `unpaid` error anywhere in this app.

**No file uploads:** connector credentials are text fields only.

### Ownership model

Ownership is strictly binary (owned by a user **XOR** owned by a team) — there is no
per-member team visibility marker (`glue`) like `sites`/`domains` have. A team-owned
integration has no personal owner. The `?user=` sub-filter that `scope=gang`
otherwise accepts (to select one team member) is **not consulted** anywhere in `adv` —
an integration is either wholly personal or wholly the team's.

| Scope | List/log filter | Single-row ownership check |
|---|---|---|
| `user` (default) | Integrations owned by the caller personally | The integration is not team-owned and the caller is its owner |
| `gang` / `team` | Integrations owned by the caller's team | The integration is owned by the caller's team |
| `admin` / `sys` / `system` / `all` | Optional `user`/`gang` filter; neither given → **no filter, every integration** | Always allowed (any row) |
| anything else | matches nothing | denied |

`admin` scope grants unconditional access to any row in `form`/`edit`/`update`/`del`,
even if the request's `?user=`/`?gang=` doesn't match the loaded row's actual owner.

---

## Connector Catalogue

Every connector is identified by its type code (the `type` value). A connector uses some of
six standard credential slots (`url`, `login`, `pass`, `user`, `api`, `key`) — the
`dataset.auths` list of `adv/form` shows which. Connectors with no extra settings return an
empty `fields` array on `adv/form`.

| Type code | Name (EN) | Name (RU) | Credential slots (`dataset.auths`) | Connector-specific fields (`fields` of `adv/form`) |
|---|---|---|---|---|
| `facebook` | FB: Facebook | FB: Facebook | `api` | `facebook_mode` (select), `facebook_proxy` (text), `facebook_pauth` (text) |
| `tiktok` | TikTok Ads | TikTok Ads | `api` | `tiktok_mode` (select), `tiktok_accs` (text, comma-separated advertiser IDs), `tiktok_proxy` (text), `tiktok_pauth` (text) |
| `bigo` | BIGO Ads | BIGO Ads | `api`, `key` | `bigo_refresh` (textarea, OAuth refresh token seed), `bigo_mode` (select) |
| `dolphin` | FB: Dolphin | FB: Dolphin | `url`, `api` | `dolphin_type` (select), `dolphin_cur` (select, currency code) |
| `dolphincl` | FB: Dolphin Cloud | FB: Dolphin Cloud | `api` | `dolphin_type` (select), `dolphin_cur` (select) |
| `fbtool` | FB: FBtool | FB: FBtool | `api` | `fbtool_mode` (select), `fbtool_ast` (select, campaign-status filter), `fbtool_accs` (text) |
| `mgid` | MGid | MGid | `login`, `pass` | *(none)* |
| `mixadvert` | MixAdvert | MixAdvert | `login`, `api` | `mixadvert_mode` (select) |
| `adpartner` | AdPartner | AdPartner | `api` | *(none)* |
| `luckyads` | LuckyAds | LuckyAds | `api` | `luckyads_mode` (select), `luckyads_cur` (select) |
| `vk` | VK: VKontakte | VK: ВКонтакте | `login`, `key` | `vk_mode` (select), `vk_reset` (checkbox, auto-drop overdue OAuth tokens) |

All 6 standard credential fields are always present in the integration object
regardless of which the connector actually uses — consult `dataset.auths` (returned by
`adv/form`) to know which ones are meaningful for a given `type`.

`adv/edit` accepts exactly the connector-specific field names that `adv/form` returns in
`fields` and **rebuilds the whole `meta` object from the request**; it does not merge with
existing meta (see `adv/edit` below).

`bigo` is the one connector with token-rotation side effects: its scheduled run can
silently rewrite its `meta` (access token, expiry, and possibly a new refresh token)
outside of any `adv/edit` API call. `adv/edit` preserves this rotated state unless the
client submits a genuinely new `bigo_refresh` value.

---

## Schedule / Period / Tag Reference

**`repeat`** — index into the "check frequency" label list:

| Index | EN | RU | Actual interval |
|---|---|---|---|
| 0 | Once a day | Раз в день | 86400s |
| 1 | Each hour | Каждый час | 3600s |
| 2 | Once 3 hours | Раз в три часа | 10800s |
| 3 | Twice a day | Дважды в день | 43200s |
| 4 | 3 times per day | Трижды в день | 28800s |
| 5 | Once 3 days | Раз в три дня | 259200s |

Actual scheduler intervals (±10% jitter) do not follow the label order (e.g. index 0 =
86400s sits between index 2 = 10800s and index 3 = 43200s in real duration).

**`period`** — index into the "report period to fetch" label list:

| Index | EN | RU | Actual date range |
|---|---|---|---|
| 0 | Today | Текущий день | midnight today → now |
| 1 | Yesterday | Вчерашний день | yesterday only |
| 2 | Today and yesterday | Сегодня и вчера | -1 day midnight → now |
| 3 | 3 days | Три дня | -3 days midnight → now |
| 4 | 1 week | Неделя | -6 days midnight → now |

Every index except `1` includes today in the fetched range (not just its literal label).

**`field`** — index into the "which field carries the ad/creative ID" label list,
returned by `adv/tags`:

| Index | Label |
|---|---|
| 0 | `utm_content` |
| 1 | `utm_term` |
| 2 | `utm_medium` |
| 3 | `utm_campaign` |
| 4 | `utm_source` |
| 5 | `domain` |
| 6 | `subid` |
| 7 | `flow` |
| 8 | `way` |
| 9 | `customer` |

These are technical parameter names, identical in EN and RU.

---

## Integration Object

Returned (with variations) by `list`, `form`, `add`, `edit`:

```json
{
  "id": 12,
  "type": "facebook",
  "typename": "FB: Facebook",
  "status": 1,
  "name": "FB main",
  "url": "", "login": "", "pass": "", "user": "", "api": "EAAB...", "key": "",
  "repeat": 4, "period": 1, "unique": 1, "valid": 1,
  "source": "", "field": 0,
  "meta": { "mode": 0, "proxy": "", "pauth": "" }
}
```

| Field | Type | Description |
|---|---|---|
| `id` | int | integration ID |
| `type` | string | connector type code, fixed at creation — not editable |
| `typename` | string | localized connector name, or raw code if no label exists |
| `status` | int | 1 = active (picked up by CRON), 0 = disabled |
| `name` | string | integration name |
| `url`, `login`, `pass`, `user`, `api`, `key` | string | standard credential slots (see `dataset.auths` for which are used by `type`) |
| `repeat` | int | index into the check-frequency table above |
| `period` | int | index into the report-period table above |
| `unique` | int | 1 = only count unique clicks |
| `valid` | int | 1 = only count non-filtered (valid) clicks |
| `source` | string | optional `utm_source` value filter |
| `field` | int | index into the tag-label table above |
| `meta` | object | connector-specific settings, JSON-decoded |
| `last` | int | (list only) unix, last CRON run |
| `next` | int | (list only / `update`) unix, next scheduled run |

---

## adv/types — Connector catalogue

```
GET api/adv/types.json?id={token}[&scope=...]
```

Returns the type-code → localized-name map for every connector, identical regardless
of `scope` (scope is only resolved to authenticate the call).

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard, default `user` |

**Response:**
```json
{
  "status": "ok",
  "data": {
    "facebook": "FB: Facebook",
    "tiktok": "TikTok Ads",
    "bigo": "BIGO Ads",
    "vk": "VK: VKontakte",
    "mgid": "MGid"
  }
}
```
Sorted alphabetically by localized value, not by key.

**Errors:** `scope`

---

## adv/tags — UTM-tag label list

```
GET api/adv/tags.json?id={token}[&scope=...]
```

Returns the same label list as the `tags` block of `adv/form`'s `dataset`, so a client
can render the `field` selector standalone (see the `field` table above).

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard, default `user` |

**Response:**
```json
{
  "status": "ok",
  "data": {
    "0": "utm_content", "1": "utm_term", "2": "utm_medium",
    "3": "utm_campaign", "4": "utm_source", "5": "domain",
    "6": "subid", "7": "flow", "8": "way", "9": "customer"
  }
}
```

**Errors:** `scope`

---

## adv/list — Existing integrations

```
GET api/adv/list.json?id={token}[&scope=...]
```

**Parameters:**

| Name | Type | Required | Description | Default |
|---|---|---|---|---|
| `scope` | string | no | standard | `user` |
| `user` | int | no | admin scope: filter to one user | — |
| `gang` | int | no | admin scope: filter to one team | — |
| `type` | string | no | connector type code; silently ignored if not a recognized type | — |
| `search` | string | no | full-text search on integration name | — |
| `page` | int | no | page (≥1) | 1 |
| `show` | int | no | page size; `0` = all rows | 0 |

Ordered by status (active integrations first), then by name ascending.

**Response:**
```json
{
  "status": "ok",
  "total": 3,
  "page": 1,
  "show": 0,
  "data": {
    "12": {
      "id": 12, "name": "FB main", "status": 1,
      "type": "facebook", "typename": "FB: Facebook",
      "last": 1752600000, "next": 1752610800
    }
  }
}
```
`data` is keyed by integration ID. List entries carry only `id`/`name`/`status`/`type`/
`typename`/`last`/`next` — not the full projection (no credentials/meta).

**Errors:** `scope`

---

## adv/form — Integration + editable field set

```
GET api/adv/form.json?id={token}&adv={id}[&scope=...]
```

The API analogue of "render the edit screen for one integration" — returns the stored
integration plus the connector's own extra field descriptors and all reference data
(`dataset`) needed to render a complete edit form in one call.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `adv` | int | **yes** | integration ID |

**Response:**
```json
{
  "status": "ok",
  "data": {
    "id": 12, "type": "facebook", "typename": "FB: Facebook",
    "status": 1, "name": "FB main",
    "url": "", "login": "", "pass": "", "user": "", "api": "EAAB...", "key": "",
    "repeat": 4, "period": 1, "unique": 1, "valid": 1,
    "source": "", "field": 0,
    "meta": { "mode": 0, "proxy": "", "pauth": "" },
    "fields": [
      { "type": "select", "name": "facebook_mode", "head": "Contains", "value": 0, "options": ["ID of ad", "ID of adset", "ID of campaign"] },
      { "type": "text", "name": "facebook_proxy", "head": "Proxy for requests", "descr": "...", "value": "" },
      { "type": "text", "name": "facebook_pauth", "head": "Proxy authorization", "descr": "...", "value": "" }
    ]
  },
  "dataset": {
    "auths": ["api"],
    "tags": { "0": "utm_content", "1": "utm_term" },
    "repeats": ["Once a day", "Each hour", "Once 3 hours", "Twice a day", "3 times per day", "Once 3 days"],
    "periods": ["Today", "Yesterday", "Today and yesterday", "3 days", "1 week"]
  }
}
```

`data` = the Integration Object plus `fields` (the connector's own field descriptors,
`[]` for connectors with no extra settings). `dataset.auths` lists which of the six
standard credential slots this connector's `type` actually uses; `dataset.tags`/
`repeats`/`periods` are the same label lists documented above, provided inline so a
client can render the whole form without a second call.

**Errors:**

| `error` | Condition |
|---|---|
| `scope` | scope is invalid or not permitted |
| `no-id` | `adv` not passed (or `0`) |
| `not-found` | integration id does not exist, or its connector type is no longer available |
| `access` | loaded integration not owned under the resolved scope |

---

## adv/add — Create an integration

```
POST api/adv/add.json?id={token}&type=facebook&name=FB+main
```

Creates a new integration **shell** — type + name only. All credentials, schedule, and
connector-specific fields are set afterwards via `adv/edit`.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `type` | string | **yes** | connector type code; must be a recognized type |
| `name` | string | practically yes | integration name |
| `user` | int | admin scope, if no `gang` | owner user |
| `gang` | int | admin scope, alternative to `user` | owner team |

For `scope=gang`, the owning team is always the caller's own resolved gang — there is
no `gang` request parameter honored for non-admin scopes.

**Response:**
```json
{ "status": "ok", "data": { "id": 42, "type": "facebook", "name": "FB main" } }
```
`type`/`name` are the sanitized values that were saved.

**Errors:**

| `error` | Condition |
|---|---|
| `scope` | scope is invalid or not permitted |
| `bad-type` | `type` empty or not a recognized connector code |
| `no-user` | `scope=admin` with neither `user` nor `gang` set |
| `db` | saving failed |

**Side effects:** every setting other than `name`/`type`/owner keeps its default
— **new integrations are created disabled** (`status = 0`); the scheduler only
picks up integrations with `status = 1`, so a freshly created integration will not run until an
`adv/edit` call sets `status=1`. No credentials or schedule are set at creation.

---

## adv/edit — Edit integration settings (partial)

```
POST api/adv/edit.json?id={token}&adv={id}&status=1&api=EAAB...
```

Updates any subset of an integration's standard fields (including activation via
`status`) and/or its connector-specific settings blob. There is no separate
activate/deactivate endpoint — `status` doubles as the toggle.

**Parameters:**

All parameters are optional — only keys actually present in the request are written
(explicit `0`/`""` is honored as "present", not skipped).

| Name | Type | Notes |
|---|---|---|
| `adv` | int | **required** — integration ID |
| `status` | any | truthy → 1, else 0 |
| `name` | string | |
| `url`, `login`, `pass`, `user`, `api`, `key` | string | standard credentials |
| `repeat` | int | index into check-frequency table |
| `period` | int | index into report-period table |
| `unique` | any | truthy → 1, else 0 |
| `valid` | any | truthy → 1, else 0 |
| `source` | string | |
| `field` | int | index into tag-label table |
| *(connector fields, e.g. `facebook_mode`)* | varies | whole `meta` object is rebuilt, see below |

`type` is **not** editable — the connector type is fixed at `adv/add`; sending `type`
on `edit` has no effect.

**Connector meta — all-or-nothing.** If **any** of the connector's own field names (as
listed in `adv/form`'s `data.fields`) is present in the request, the **entire** meta
blob is rebuilt from the request — it does not merge with existing meta. Sending only
one of a connector's several fields resets the omitted ones to empty. To change one
connector setting without resetting the others, resend the connector's full field set
(as returned by `adv/form`).

If the request contains no recognized key at all, the call is a no-op: `status: ok`
with the current, unchanged integration returned (not an error).

**Response:**
```json
{
  "status": "ok",
  "data": {
    "id": 12, "type": "facebook", "typename": "FB: Facebook",
    "status": 1, "name": "FB main",
    "url": "", "login": "", "pass": "", "user": "", "api": "EAAB...", "key": "",
    "repeat": 4, "period": 1, "unique": 1, "valid": 1,
    "source": "", "field": 0,
    "meta": { "mode": 0, "proxy": "", "pauth": "" }
  }
}
```
Same shape as `adv/form`'s `data`, **minus** `fields` (call `adv/form` again for a
refreshed connector field list).

**Errors:**

| `error` | Condition |
|---|---|
| `scope` | scope is invalid or not permitted |
| `no-id` | `adv` not passed |
| `not-found` | integration id does not exist, or its connector type is no longer available |
| `access` | not owned under the resolved scope |
| `db` | update failed |

---

## adv/update — Force an out-of-schedule run

```
POST api/adv/update.json?id={token}&adv={id}
```

Resets the "next run" timestamp to now so the CRON scheduler picks the integration up
on its next tick, without waiting for the configured `repeat` interval. Not wired into
the UI — API-only capability.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `adv` | int | **yes** | integration ID |

**Response:**
```json
{ "status": "ok", "data": { "id": 12, "next": 1752614400 } }
```

**Errors:**

| `error` | Condition |
|---|---|
| `scope` | scope is invalid or not permitted |
| `no-id` | `adv` not passed |
| `not-found` | integration id does not exist |
| `access` | not owned under the resolved scope |
| `db` | update failed |

**Note:** does **not** touch `status`. If the integration is disabled, forcing `next`
to now does not activate it — the scheduler still skips disabled integrations. A
client must set `status=1` via `adv/edit` first.

---

## adv/del — Delete an integration and purge its log

```
POST api/adv/del.json?id={token}&adv={id}
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `scope` | string | no | standard |
| `adv` | int | **yes** | integration ID |

**Response:**
```json
{ "status": "ok" }
```
No `data` key on success.

**Errors:**

| `error` | Condition |
|---|---|
| `scope` | scope is invalid or not permitted |
| `no-id` | `adv` not passed |
| `not-found` | integration id does not exist |
| `access` | not owned under the resolved scope |
| `db` | deleting the integration failed |

**Side effects:**
- Deletes the integration row.
- Deletes **all** request-log rows for that integration (the purge is limited to the same
  owner scope as `list`/`log`; its own failure is not reported as an API
  error since the integration is already gone).

---

## adv/log — Request journal (paginated + filtered)

```
GET api/adv/log.json?id={token}[&adv={id}][&error=1][&scope=...]
```

Reads the outbound-HTTP-request journal written whenever a connector makes a call, for
troubleshooting.

**Parameters:**

| Name | Type | Required | Description | Default |
|---|---|---|---|---|
| `scope` | string | no | standard | `user` |
| `adv` | int | no | restrict to one integration | — (all owned integrations) |
| `error` | any | no | present+truthy → errors only; present+falsy → successes only; absent → no filter | — |
| `page` | int | no | page (≥1) | 1 |
| `show` | int | no | page size; `0` = all rows | 0 |

Ordered newest first.

**Response:**
```json
{
  "status": "ok",
  "total": 2,
  "page": 1,
  "show": 0,
  "data": {
    "501": {
      "id": 501, "adv": 12, "time": 1752610801,
      "url": "https://graph.facebook.com/v25.0/me/adaccounts?...",
      "post": "", "reply": "{\"data\":[...]}",
      "cc": "{\"verbose\":true}",
      "error": 0, "status": 200, "timer": 0.842
    }
  }
}
```
`data` is keyed by log row ID.

| Field | Type | Description |
|---|---|---|
| `id` | int | log row ID |
| `adv` | int | which integration made the request |
| `time` | int | unix |
| `url` | string | raw request URL, not truncated |
| `post` | string | raw POST body sent |
| `reply` | string | raw HTTP response body |
| `cc` | string | JSON-encoded cURL config (headers, proxy, …) |
| `error` | int | 1 if the response body was empty, else 0 |
| `status` | int | HTTP status of the outbound request (0 if the request itself failed) |
| `timer` | float | request duration in seconds |

**Errors:** `scope`

---

## Common Errors Reference

| Error | Meaning |
|---|---|
| `scope` | Invalid scope or insufficient permissions |
| `no-id` | Integration ID not provided |
| `not-found` | Integration id does not exist (or, for `form`/`edit`, its connector type is no longer available) |
| `access` | Integration not owned under the resolved scope |
| `bad-type` | `type` missing or not a recognized connector code (`adv/add` only) |
| `no-user` | `scope=admin` with neither `user` nor `gang` set (`adv/add` only) |
| `db` | Saving or deleting failed |
