# Ad Spend Connectors

The ad spend connectors API manages background integrations that periodically pull spend statistics from ad networks and traffic sources (Facebook, TikTok, BIGO Ads, VK, MGid, LuckyAds, Dolphin, Dolphin Cloud, FBtool, MixAdvert, AdPartner) and write the imported cost onto tracked clicks — the same destination used by [manual spend import](api-tracker.md#spend), but running automatically on a schedule. Each integration is created for a single connector type, chosen once and fixed afterwards; its credentials, schedule and connector-specific settings are then configured separately.

Each integration belongs either to a specific user (personal integration) or to the team as a whole (team integration). Ownership is strictly binary: unlike [site storage](api-sites.md) or [parked domains](api-domains.md), there is no per-member visibility marker for team integrations — an integration is either wholly personal or wholly the team's.

<a id="advtypes"></a>
## Connector Types

**Webmaster URL**: `https://domain.com/api/adv/types.json?id={token}`

**Team URL**: `https://domain.com/api/adv/types.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/types.json?id={token}&scope=admin`

The function returns the catalogue of available connector types: the type code used everywhere else in this API (`type`), together with its localized display name. The `scope` parameter only affects authentication — the returned catalogue is identical for every caller regardless of scope.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Dictionary of connector types, keyed by type code (see below). |

Error codes:

- `scope` — workspace permission check failed (invalid scope, missing team lead or administrator rights, or, for `scope=gang`, a `user` parameter that is not a member of your team).

The `data` dictionary maps each connector type code to its localized name; entries are ordered alphabetically by the localized name, not by code. Available type codes:

- `facebook` — FB: Facebook
- `tiktok` — TikTok Ads
- `bigo` — BIGO Ads
- `dolphin` — FB: Dolphin
- `dolphincl` — FB: Dolphin Cloud
- `fbtool` — FB: FBtool
- `mgid` — MGid
- `mixadvert` — MixAdvert
- `adpartner` — AdPartner
- `luckyads` — LuckyAds
- `vk` — VK: VKontakte

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "adpartner": "AdPartner",
        "bigo": "BIGO Ads",
        "dolphin": "FB: Dolphin",
        "dolphincl": "FB: Dolphin Cloud",
        "facebook": "FB: Facebook",
        "fbtool": "FB: FBtool",
        "luckyads": "LuckyAds",
        "mgid": "MGid",
        "mixadvert": "MixAdvert",
        "tiktok": "TikTok Ads",
        "vk": "VK: VKontakte"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "scope"
}
```

<a id="advtags"></a>
## Spend-Matching Tag Labels

**Webmaster URL**: `https://domain.com/api/adv/tags.json?id={token}`

**Team URL**: `https://domain.com/api/adv/tags.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/tags.json?id={token}&scope=admin`

The function returns the ordered label list for the "which click field carries the ad/creative ID" selector, used to render the `field` setting (`dataset.tags` in [get integration edit form](#advform)) without hardcoding labels client-side. As with [connector types](#advtypes), `scope` only affects authentication — the returned list is identical for every caller.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Dictionary of tag labels, keyed by the integer index stored in `field` (see below). |

Error codes:

- `scope` — workspace permission check failed.

Values of `data` (index — label):

- `0` — `utm_content`
- `1` — `utm_term`
- `2` — `utm_medium`
- `3` — `utm_campaign`
- `4` — `utm_source`
- `5` — `domain`
- `6` — `subid`
- `7` — `flow`
- `8` — `way`
- `9` — `customer`

Each index selects which click field (UTM tag, domain, SubID, flow, way or customer, as listed above) is matched against the ad/creative ID when the cost is applied.

**Example of a successful server response:**

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

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "scope"
}
```

<a id="advlist"></a>
## Integrations List

**Webmaster URL**: `https://domain.com/api/adv/list.json?id={token}`

**Team URL**: `https://domain.com/api/adv/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/list.json?id={token}&scope=admin`

The function returns a list of configured ad-spend integrations with filtering and pagination support. All filters are combined using AND. Results are ordered with active integrations first (`status` descending), then alphabetically by name.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=admin`: the ID of a specific user to filter by. |
| `gang` | For `scope=admin`: team ID to filter by. |
| `type` | Filter by connector type code. Unrecognized codes are silently ignored (no filtering is applied). |
| `search` | Full-text search by integration name. |
| `page` | Results page (≥ 1). Default: 1. |
| `show` | Page size. Default: 0 (return all records without limit). |

For `scope=admin`, if neither `user` nor `gang` is provided, every integration in the system is returned.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of records matching the filters. When `show > 0`, computed by a separate query. |
| `page` | Current page. |
| `show` | Page size. |
| `data` | Dictionary of integrations, keyed by numeric integration ID (see below). |

Error codes:

- `scope` — workspace permission check failed (invalid scope, missing team lead or administrator rights, or, for `scope=gang`, a `user` parameter that is not a member of your team).

Fields of a record in `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric integration ID. |
| `name` | Integration name. |
| `status` | Integration status (see below). |
| `type` | Connector type code. |
| `typename` | Localized connector name, or the raw type code if no localized name exists. |
| `last` | Unix timestamp of the last CRON run; `0` if it has never run. |
| `next` | Unix timestamp of the next scheduled CRON run. |

Values of the `status` field:

- `0` — disabled (the integration is never picked up by the CRON scheduler).
- `1` — active.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 1,
    "page": 1,
    "show": 0,
    "data": {
        "12": {
            "id": 12,
            "name": "FB main",
            "status": 1,
            "type": "facebook",
            "typename": "FB: Facebook",
            "last": 1752600000,
            "next": 1752610800
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "scope"
}
```

<a id="advform"></a>
## Get Integration Edit Form

**Webmaster URL**: `https://domain.com/api/adv/form.json?id={token}`

**Team URL**: `https://domain.com/api/adv/form.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/form.json?id={token}&scope=admin`

The function returns everything needed to render a complete edit form for one integration in a single call: every stored field, the connector's own extra field descriptors, and reference data (`dataset`) for populating the form's selects.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `adv`* | Integration ID. Required. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Integration data and connector-specific field descriptors (see below). |
| `dataset` | Reference data for rendering the form (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `no-id` — the `adv` parameter was not provided (or is `0`).
- `not-found` — no integration with this ID exists, or its connector type is no longer available.
- `access` — the integration is not owned under the resolved scope.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric integration ID. |
| `type` | Connector type code. Fixed at creation, not editable afterwards. |
| `typename` | Localized connector name. |
| `status` | Integration status: `1` active, `0` disabled. |
| `name` | Integration name. |
| `url` | Standard credential: URL. Only meaningful if listed in `dataset.auths`. |
| `login` | Standard credential: login. Only meaningful if listed in `dataset.auths`. |
| `pass` | Standard credential: password. Only meaningful if listed in `dataset.auths`. |
| `user` | Standard credential: user/account identifier. Only meaningful if listed in `dataset.auths`. |
| `api` | Standard credential: API key. Only meaningful if listed in `dataset.auths`. |
| `key` | Standard credential: API secret. Only meaningful if listed in `dataset.auths`. |
| `repeat` | Check frequency (see below). |
| `period` | Report period to fetch on each run (see below). |
| `unique` | `1` — only count unique clicks when matching cost; `0` — count all. |
| `valid` | `1` — only count valid (non-filtered) clicks when matching cost; `0` — count all. |
| `source` | Optional `utm_source` value filter; empty means no filter. |
| `field` | Which click field carries the ad/creative ID for matching (see [spend-matching tag labels](#advtags)). |
| `meta` | Connector-specific settings, decoded (an object, not a raw JSON string). |
| `fields` | Connector-specific field descriptors for rendering extra settings (see below); an empty array for connectors with no extra settings. |
| `desc` | Setup instructions for the connector type in your interface language (where to get the credentials, recommended period and frequency); may contain simple HTML such as links. An empty string if the connector has no description. |

All 6 standard credential fields (`url`, `login`, `pass`, `user`, `api`, `key`) are always present in the response regardless of which ones the connector actually uses — consult `dataset.auths` to know which are meaningful for the integration's connector type.

Values of `repeat` (check frequency):

- `0` — Once a day
- `1` — Each hour
- `2` — Once 3 hours
- `3` — Twice a day
- `4` — 3 times per day
- `5` — Once 3 days

Values of `period` (report period to fetch):

- `0` — Today
- `1` — Yesterday
- `2` — Today and yesterday
- `3` — 3 days
- `4` — 1 week

For every value except `1` (yesterday only), the actual fetched date range also includes the current day in addition to the stated number of past days — e.g. `3` ("3 days") really covers the last 3 days plus today.

Each entry of `fields` describes one connector-specific setting:

| Field | Description |
| --- | --- |
| `type` | Input type: `text`, `textarea`, `select`, or `checkbox`. |
| `name` | Parameter name to submit back through [edit integration](#advedit). |
| `head` | Human-readable label. |
| `descr` | Optional help text; omitted if the connector does not provide one. |
| `value` | Current value, read from `meta`. |
| `options` | For `type=select` only: list of option labels, the array index being the value to submit. |

Fields of `dataset`:

| Field | Description |
| --- | --- |
| `auths` | List of the connector's standard credential fields (a subset of `url`, `user`, `login`, `pass`, `api`, `key`) — which of the 6 credential fields in `data` are actually used by this connector type. |
| `tags` | Same label list as [spend-matching tag labels](#advtags). |
| `repeats` | Same label list as the `repeat` values above, indexed identically. |
| `periods` | Same label list as the `period` values above, indexed identically. |

Connector reference (type code, localized name, standard credentials used, connector-specific `fields`):

- `facebook` — FB: Facebook. Credentials: `api`. Fields: `facebook_mode` (select), `facebook_proxy` (text), `facebook_pauth` (text).
- `tiktok` — TikTok Ads. Credentials: `api`. Fields: `tiktok_mode` (select), `tiktok_accs` (text, comma-separated advertiser IDs), `tiktok_proxy` (text), `tiktok_pauth` (text).
- `bigo` — BIGO Ads. Credentials: `api`, `key`. Fields: `bigo_refresh` (textarea, OAuth refresh token seed), `bigo_mode` (select).
- `dolphin` — FB: Dolphin. Credentials: `url`, `api`. Fields: `dolphin_type` (select), `dolphin_cur` (select, currency code).
- `dolphincl` — FB: Dolphin Cloud. Credentials: `api`. Fields: `dolphin_type` (select), `dolphin_cur` (select).
- `fbtool` — FB: FBtool. Credentials: `api`. Fields: `fbtool_mode` (select), `fbtool_ast` (select, campaign-status filter), `fbtool_accs` (text).
- `mgid` — MGid. Credentials: `login`, `pass`. No connector-specific fields.
- `mixadvert` — MixAdvert. Credentials: `login`, `api`. Fields: `mixadvert_mode` (select).
- `adpartner` — AdPartner. Credentials: `api`. No connector-specific fields.
- `luckyads` — LuckyAds. Credentials: `api`. Fields: `luckyads_mode` (select), `luckyads_cur` (select).
- `vk` — VK: VKontakte. Credentials: `login`, `key`. Fields: `vk_mode` (select), `vk_reset` (checkbox, auto-drop overdue OAuth tokens).

For the `bigo` connector, the OAuth access token, its expiry, and — if BIGO rotates it — a new refresh token stored in `meta` can be silently rewritten by the scheduled run itself (token rotation), independently of any [edit integration](#advedit) call. A client polling this function may see `meta` change between calls without having made an edit; resending the connector's current `bigo_refresh` value on [edit](#advedit) preserves this rotated state, a genuinely new value replaces it.

**Example of a successful server response:**

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
            { "type": "select", "name": "facebook_mode", "head": "Contains", "value": 0, "options": [ "ID of ad", "ID of adset", "ID of campaign" ] },
            { "type": "text", "name": "facebook_proxy", "head": "Proxy for requests", "descr": "Protocol, IP address and port, e.g.: socks5://1.2.3.4:56", "value": "" },
            { "type": "text", "name": "facebook_pauth", "head": "Proxy authorization", "descr": "Login and password separated by a colon, e.g.: user:pass", "value": "" }
        ],
        "desc": "To integrate with Facebook directly, provide your API token. Register on developers.facebook.com, create an application like Other - Business. ..."
    },
    "dataset": {
        "auths": [ "api" ],
        "tags": { "0": "utm_content", "1": "utm_term", "2": "utm_medium", "3": "utm_campaign", "4": "utm_source", "5": "domain", "6": "subid", "7": "flow", "8": "way", "9": "customer" },
        "repeats": [ "Once a day", "Each hour", "Once 3 hours", "Twice a day", "3 times per day", "Once 3 days" ],
        "periods": [ "Today", "Yesterday", "Today and yesterday", "3 days", "1 week" ]
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="advadd"></a>
## Create Integration

**Webmaster URL**: `https://domain.com/api/adv/add.json?id={token}`

**Team URL**: `https://domain.com/api/adv/add.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/add.json?id={token}&scope=admin`

The function creates a new integration shell — only the connector type and name are set. Credentials, schedule and connector-specific settings must be set afterwards via [edit integration](#advedit); it is recommended to first call [get integration edit form](#advform) to learn which credential fields and connector-specific settings are relevant for the chosen type.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `type`* | Connector type code (see [connector types](#advtypes)). Required. |
| `name` | Integration name. Practically required — an empty name is stored but not meaningful. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=admin` without `gang`: the ID of the owner user for the new integration. |
| `gang` | For `scope=admin`: the ID of the owner team for the new integration. |

For `scope=gang`, the owning team is always the caller's own team — there is no `gang` request parameter for non-admin scopes.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the created integration (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `bad-type` — `type` is empty or not a recognized connector code.
- `no-user` — for `scope=admin`: neither `user` nor `gang` was provided.
- `db` — database error while creating the integration.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric ID of the created integration. |
| `type` | Connector type code (see above). |
| `name` | Integration name (see above). |

New integrations are created disabled (`status` = `0`) — the CRON scheduler only picks up active integrations, so a freshly created integration will not run until [edit integration](#advedit) sets `status=1`.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "type": "facebook",
        "name": "FB main"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-type"
}
```

<a id="advedit"></a>
## Edit Integration

**Webmaster URL**: `https://domain.com/api/adv/edit.json?id={token}`

**Team URL**: `https://domain.com/api/adv/edit.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/edit.json?id={token}&scope=admin`

The function updates any subset of an integration's standard fields (including activation via `status`) and/or its connector-specific settings. All fields are optional — only the parameters actually present in the request are written; an explicit `0` or empty-string value is still honored as "present". If the request contains no recognized field at all, the function is a no-op and returns `status: ok` with the integration's current, unchanged data.

**Connector settings are updated all-or-nothing.** If the request contains any one of the connector's own field names (as listed in `data.fields` from [get integration edit form](#advform)), the connector rebuilds its *entire* settings blob (`meta`) from the request — it does not merge with the previously stored values. Concretely, sending only one of a connector's several fields (e.g. only `facebook_mode` without `facebook_proxy`/`facebook_pauth`) resets the omitted settings to empty. To change a single connector setting without resetting the others, resend the connector's full field set as returned by [get integration edit form](#advform).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `adv`* | Integration ID. Required. |
| `status` | Activation flag: any truthy value activates the integration, falsy deactivates it. |
| `name` | Integration name. |
| `url` | Standard credential: URL. |
| `login` | Standard credential: login. |
| `pass` | Standard credential: password. |
| `user` | Standard credential: user/account identifier. |
| `api` | Standard credential: API key. |
| `key` | Standard credential: API secret. |
| `period` | Report period to fetch (see above). |
| `repeat` | Check frequency (see above). |
| `unique` | `1` — only count unique clicks when matching cost. |
| `valid` | `1` — only count valid (non-filtered) clicks when matching cost. |
| `source` | Optional `utm_source` value filter. |
| `field` | Which click field carries the ad/creative ID (see [spend-matching tag labels](#advtags)). |
| *(connector-specific fields, e.g. `facebook_mode`)* | Any field name declared by the connector in `data.fields` (see [get integration edit form](#advform)); see note above on all-or-nothing behavior. |

`type` cannot be changed after creation — sending it has no effect. `status` doubles as the activate/deactivate toggle; there is no separate toggle function.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Integration data after the update (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `no-id` — the `adv` parameter was not provided (or is `0`).
- `not-found` — no integration with this ID exists, or its connector type is no longer available.
- `access` — the integration is not owned under the resolved scope.
- `db` — database error while saving changes.

Fields of `data`: same as `data` in [get integration edit form](#advform), except that it never includes the `fields` and `desc` keys (a refreshed connector field descriptor list is not echoed back after a settings update — call [get integration edit form](#advform) again to obtain it).

**Example of a successful server response:**

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

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "access"
}
```

<a id="advupdate"></a>
## Force Integration Run

**Webmaster URL**: `https://domain.com/api/adv/update.json?id={token}`

**Team URL**: `https://domain.com/api/adv/update.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/update.json?id={token}&scope=admin`

The function resets the integration's next-run time to now, so the CRON scheduler picks it up on its next tick instead of waiting for the configured check frequency (`repeat`).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `adv`* | Integration ID. Required. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Updated schedule data (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `no-id` — the `adv` parameter was not provided (or is `0`).
- `not-found` — no integration with this ID exists.
- `access` — the integration is not owned under the resolved scope.
- `db` — database error while saving the change.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric integration ID. |
| `next` | Unix timestamp of the next scheduled run — always the current time on success. |

This function does not change `status`. If the integration is disabled, forcing `next` to now does not activate it — the CRON scheduler only selects active integrations regardless of their next-run time. Set `status=1` via [edit integration](#advedit) first if the integration is currently disabled.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12,
        "next": 1752614400
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="advdel"></a>
## Delete Integration

**Webmaster URL**: `https://domain.com/api/adv/del.json?id={token}`

**Team URL**: `https://domain.com/api/adv/del.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/del.json?id={token}&scope=admin`

The function deletes the integration and its entire request log (see [integration request log](#advlog)) — no log entries are kept after deletion.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `adv`* | Integration ID. Required. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |

There is no `data` field on success.

Error codes:

- `scope` — workspace permission check failed.
- `no-id` — the `adv` parameter was not provided (or is `0`).
- `not-found` — no integration with this ID exists.
- `access` — the integration is not owned under the resolved scope.
- `db` — database error while deleting the integration.

**Example of a successful server response:**

```json
{
    "status": "ok"
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "access"
}
```

<a id="advlog"></a>
## Integration Request Log

**Webmaster URL**: `https://domain.com/api/adv/log.json?id={token}`

**Team URL**: `https://domain.com/api/adv/log.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/adv/log.json?id={token}&scope=admin`

The function returns the journal of outbound HTTP requests made by connectors, for troubleshooting an integration. Results are ordered newest first.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=admin`: the ID of a specific user to filter by. |
| `gang` | For `scope=admin`: team ID to filter by. |
| `adv` | Restrict the log to one integration ID. If omitted, the log of every integration owned under the resolved scope is returned. |
| `error` | Filter by outcome (see below). If omitted, no filter is applied. |
| `page` | Results page (≥ 1). Default: 1. |
| `show` | Page size. Default: 0 (return all records without limit). |

Values of `error` (filter parameter): any truthy value (e.g. `1`) restricts the log to failed requests; `0` (present but falsy) restricts it to successful requests; omitting the parameter applies no filter.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of log records matching the filters. When `show > 0`, computed by a separate query. |
| `page` | Current page. |
| `show` | Page size. |
| `data` | Dictionary of log entries, keyed by numeric log ID (see below). |

Error codes:

- `scope` — workspace permission check failed.

Fields of a record in `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric log entry ID. |
| `adv` | Integration ID that made the request. |
| `time` | Unix timestamp of the request. |
| `url` | Full request URL, not truncated. |
| `post` | Raw POST body sent with the request, if any. |
| `reply` | Raw HTTP response body. |
| `cc` | JSON-encoded cURL request configuration (headers, proxy, etc.). |
| `error` | `1` if the response body was empty (request considered failed), `0` otherwise. |
| `status` | HTTP status code of the outbound request; `0` if the request itself failed to complete. |
| `timer` | Request duration in seconds. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 1,
    "page": 1,
    "show": 0,
    "data": {
        "501": {
            "id": 501,
            "adv": 12,
            "time": 1752610801,
            "url": "https://graph.facebook.com/v25.0/me/adaccounts?...",
            "post": "",
            "reply": "{\"data\":[...]}",
            "cc": "{\"verbose\":true}",
            "error": 0,
            "status": 200,
            "timer": 0.842
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "scope"
}
```
