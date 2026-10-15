# Flows and Ways

The `flows` API section provides a complete set of operations for managing **flows** and **ways** — the key entities of the tracker's TDS module. Through this API you can retrieve lists and details of flows, create and clone them, edit settings, add and copy ways, and delete flows and ways. Functions operate in three scopes: webmaster (`user`), team (`gang`), and administrator (`admin`).

The `scope` parameter is passed **via GET only** — even when the main request is sent as POST or in JSON format. The `user` and `gang` parameters for gang and admin scope can be passed in either GET or POST.

Click and lead statistics accumulated for a flow or way are **not deleted** when they are removed — this is intentional behavior to preserve financial reports.

To retrieve statistics on flows, use the functions in the [Tracker Statistics](api-stats.md) section. Managing cloaking filters that are linked to flows is described in the [Traffic Filtering](api-filter.md) section. Managing storage sites used in ways is described in the [Site Storage](api-sites.md) section. Managing domains linked to flows is described in the [Parked Domains](api-domains.md) section.

---

<a id="flowslist"></a>
## Flow List

**Webmaster URL**: `https://domain.com/api/flows/list.json?id={token}`

**Team URL**: `https://domain.com/api/flows/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/list.json?id={token}&scope=admin`

Returns a paginated list of flows with filtering by various criteria. In administrator mode without specifying `user` or `gang` filters, the function returns all flows in the system.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow` | Flow symbolic code — filter by a specific flow. |
| `fid` | Numeric flow ID — filter by a specific flow. Used if `flow` is not set. |
| `uid` | Alias for `fid` — numeric flow ID. |
| `user` | Filter by owner (for `admin` scope — filter by any user; for `gang` scope — filter within team members). |
| `gang` | Filter by team. Available for `admin` scope only. |
| `domain` | Numeric ID of a parked domain — filter by domain linked to the flow. |
| `search` | Full-text search by flow name. |
| `page` | Page number (≥ 1). Default: 1. |
| `show` | Number of flows per page. If 0 is passed, the value 100 will be used. Default: 100. |
| `short` | Shortened response format — only the main fields (see below). Default: false. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of flows matching the filter. |
| `page` | Current page. |
| `show` | Page size. |
| `data` | Associative array of flows, where the key is the numeric flow ID. Field composition is described below. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions for the specified scope.
- `unpaid` — no active subscription (for `user` and `gang` scope only).

Composition of the `data` element when `short=false` (full format):

| Field | Description |
| --- | --- |
| `id` | Numeric flow ID. |
| `user` | Flow owner ID. |
| `gang` | Team ID (0 if the flow is personal). |
| `name` | Flow name. |
| `domain` | Parked domain ID (0 if not linked). |
| `code` | Symbolic hex code of the flow. |
| `url` | Full flow URL. If a domain is linked to this flow, the code is omitted from the URL. |
| `algo` | Traffic distribution optimization mode (0..8). |
| `direct` | Share of traffic to the best way, in percent (0..100). |
| `rotor` | Rotor mode (0 or 1). |
| `template` | Tracker template ID (0 if not set). |
| `nostore` | Click data storage mode (0/1/2). |
| `simple` | Simplified flow mode (true/false). |
| `autourl` | Automatic URL construction (true/false). |
| `nocookie` | Cookie-free mode (0/1/2). |
| `pixel` | Pixel tracking enabled (true/false). |
| `filter` | Flow filter settings (object, see below). |
| `tags` | UTM tags and additional fields (object, see below). |
| `stats` | Aggregated statistics (object, see below). |
| `postback` | Postback settings (object, see below). |

Composition of the `filter` object:

| Field | Description |
| --- | --- |
| `filter` | Cloaking filter ID (0 if not set). |
| `bots` | Bot filtering enabled (true/false). |
| `filters` | Extended filtering enabled (true/false). |
| `geo` | Geo filtering enabled (true/false). |
| `lang` | Browser language filtering enabled (true/false). |
| `adid` | Ad identifier filtering enabled (true/false). |
| `geos` | Array of ISO country codes for the geo filter. |
| `url` | URL filtering enabled (true/false). |
| `site` | Site ID for trafficback (0 if not set). |
| `spath` | Path to the white site in storage (false if not set). |
| `method` | Trafficback method. |

Composition of the `tags` object:

| Field | Description |
| --- | --- |
| `source` | UTM source tag (false if not set). |
| `campaign` | UTM campaign tag (false if not set). |
| `content` | UTM content tag (false if not set). |
| `term` | UTM term tag (false if not set). |
| `medium` | UTM medium tag (false if not set). |
| `subid` | SubID (false if not set). |
| `data1` | Additional field data1 (false if not set). |
| `data2` | Additional field data2 (false if not set). |
| `cpc` | Cost per click (false if not set). |
| `cur` | Currency, three-letter code (e.g., `usd`). |

Composition of the `stats` object:

| Field | Description |
| --- | --- |
| `cr` | Conversion rate. |
| `epc` | Earnings per click. |
| `appr` | Percentage of approved leads. |
| `money` | Total earnings. |

Composition of the `postback` object:

| Field | Description |
| --- | --- |
| `global` | Postback URL for any status change (false if not set). |
| `new` | Postback URL when a new lead is created (false if not set). |
| `hold` | Postback URL on transition to "hold" status (false if not set). |
| `approve` | Postback URL on lead approval (false if not set). |
| `cancel` | Postback URL on lead cancellation (false if not set). |
| `trash` | Postback URL on sending to trash (false if not set). |

When `short=true` the composition of the `data` element is shortened:

| Field | Description |
| --- | --- |
| `id` | Numeric flow ID. |
| `user` | Flow owner ID. |
| `gang` | Team ID (0 if the flow is personal). |
| `name` | Flow name. |
| `code` | Symbolic hex code of the flow. |
| `url` | Full flow URL. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 2,
    "page": 1,
    "show": 100,
    "data": {
        "12345": {
            "id": 12345,
            "user": 7,
            "gang": 0,
            "name": "My Flow",
            "domain": 0,
            "code": "ef1c",
            "url": "https://click.example.com/ef1c",
            "algo": 0,
            "direct": 0,
            "rotor": 0,
            "template": 0,
            "nostore": 0,
            "simple": false,
            "autourl": true,
            "nocookie": 0,
            "pixel": false,
            "filter": {
                "filter": 0,
                "bots": false,
                "filters": false,
                "geo": false,
                "lang": false,
                "adid": false,
                "geos": [],
                "url": false,
                "site": 0,
                "spath": false,
                "method": 0
            },
            "tags": {
                "source": false,
                "campaign": false,
                "content": false,
                "term": false,
                "medium": false,
                "subid": false,
                "data1": false,
                "data2": false,
                "cpc": false,
                "cur": "usd"
            },
            "stats": {
                "cr": 0.0,
                "epc": 0.0,
                "appr": 0.0,
                "money": 0.0
            },
            "postback": {
                "global": false,
                "new": false,
                "hold": false,
                "approve": false,
                "cancel": false,
                "trash": false
            }
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

---

<a id="flowsget"></a>
## Get Flow

**Webmaster URL**: `https://domain.com/api/flows/get.json?id={token}`

**Team URL**: `https://domain.com/api/flows/get.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/get.json?id={token}&scope=admin`

Returns a detailed flow record together with the list of all its ways.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic flow code. Required if `fid` is not provided. |
| `fid`* | Numeric flow ID. Required if `flow` is not provided. |
| `user` | User ID for `gang` scope (filter within team) or `admin`. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Object with flow data. Field composition is described below. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — flow with the specified identifier not found.
- `access-denied` — the flow belongs to another user (`user` scope) or another team (`gang` scope).
- `wrong-user` — `gang` scope, `user` parameter was provided, but the flow owner is a different user.

Composition of the `data` field:

The `data` field contains the full flow structure (all fields described in [flows/list](#flowslist), including the extended set in `tags`: `data1..data5`, `hide1..hide6`) plus the additional `ways` field.

| Field | Description |
| --- | --- |
| `id` | Numeric flow ID. |
| `user` | Flow owner ID. |
| `gang` | Team ID (0 if the flow is personal). |
| `name` | Flow name. |
| `domain` | Parked domain ID (0 if not linked). |
| `code` | Symbolic hex code of the flow. |
| `url` | Full flow URL. |
| `gif` | Pixel image HTML tag for tracking (`<img ...>`). |
| `algo` | Optimization mode (0..8). |
| `direct` | Share of traffic to the best way (0..100). |
| `rotor` | Rotor mode (0 or 1). |
| `template` | Template ID (0 if not set). |
| `nostore` | Click storage mode (0/1/2). |
| `simple` | Simplified mode (true/false). |
| `autourl` | Auto URL (true/false). |
| `nocookie` | Cookie-free mode (0/1/2). |
| `pixel` | Pixel tracking (true/false). |
| `filter` | Filter settings (object, see [flows/list](#flowslist)). |
| `tags` | UTM tags and additional fields, including `data1..data5`, `hide1..hide6`, `cpc`, `cur`. |
| `stats` | Aggregated statistics: `cr`, `epc`, `appr`, `money`. |
| `postback` | Postback settings (see [flows/list](#flowslist)). |
| `ways` | Associative array of flow ways, where the key is the numeric way ID (see below). |

Composition of an element in the `ways` field:

| Field | Description |
| --- | --- |
| `id` | Numeric way ID. |
| `name` | Way name. |
| `active` | Way is active (true/false). |
| `nostore` | Click storage mode for the way (0/1). |
| `prio` | Way priority. |
| `site` | Storage site ID (0 if not set). |
| `spath` | Path to the site in storage (false if not set). |
| `url` | URL of the first step of the way. |
| `method` | Way operation method (0..9, see [flows/create](#flowscreate)). |
| `nxsite` | Storage site ID for the second step (0 if not set). |
| `nxpath` | Path to the storage site for the second step. |
| `nxurl` | URL of the second step. |
| `nxmethod` | Second step method. |
| `pos` | Way weight (≥ 0). |
| `w1`..`w7` | Activity by day of week: w1 — Monday, w7 — Sunday (true/false). |
| `from` | Start of the active period (time). |
| `to` | End of the active period (time). |
| `tz` | Time zone for the schedule. |
| `shift` | Schedule shift. |
| `limit` | Cap period (0..7). |
| `cap` | Lead limit (0 — unlimited). |
| `goal` | Goal for cap calculation. |
| `status` | Lead statuses counted toward the cap. |
| `geo` | Array of ISO codes of allowed countries. |
| `nogeo` | Array of ISO codes of blocked countries. |
| `mobile` | Device type filter (0..4). |
| `bots` | Pass bots through (true/false). |
| `adid` | Ad identifier filter (true/false). |
| `filter` | Cloaking filter ID for the way (0 if not set). |
| `filters` | Extended filtering for the way (true/false). |
| `lang` | Language filtering (true/false). |
| `uwl` | Referrer URL whitelist of words (array). |
| `ubl` | Referrer URL blacklist of words (array). |
| `cr` | Way conversion rate. |
| `epc` | Earnings per click for the way. |
| `appr` | Approval rate for the way. |
| `money` | Total earnings for the way. |
| `crg` | JSON map "country → conversion rate". |
| `epcg` | JSON map "country → EPC". |
| `hide1`..`hide6` | Hidden way fields. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12345,
        "user": 7,
        "gang": 0,
        "name": "My Flow",
        "domain": 0,
        "code": "ef1c",
        "url": "https://click.example.com/ef1c",
        "gif": "<img src=\"https://click.example.com/ef1c/pixel.gif\" width=\"1\" height=\"1\" />",
        "algo": 0,
        "direct": 0,
        "rotor": 0,
        "template": 0,
        "nostore": 0,
        "simple": false,
        "autourl": true,
        "nocookie": 0,
        "pixel": false,
        "filter": { },
        "tags": { },
        "stats": { "cr": 0.0, "epc": 0.0, "appr": 0.0, "money": 0.0 },
        "postback": { },
        "ways": {
            "678": {
                "id": 678,
                "name": "Main",
                "active": true,
                "url": "https://lp.example.com",
                "method": 0,
                "cap": 0,
                "leads": 0
            }
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-id"
}
```

---

<a id="flowsfile"></a>
## TDS Client File

**Webmaster URL**: `https://domain.com/api/flows/file.json?id={token}`

**Team URL**: `https://domain.com/api/flows/file.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/file.json?id={token}&scope=admin`

Generates a ready-to-use TDS client PHP file for embedding on a third-party server. The resulting file (`index.php`) contains the client library and a call to the `tds()` function with your flow's parameters.

To download the file as an attachment, pass the `format=raw` parameter directly in the URL (`api/flows/file.raw`). In this case, the request terminates immediately after the file is served and is not logged. In the standard JSON format, the file contents are returned as a string in the `data` field.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic flow code. Required if `fid` is not provided. |
| `fid`* | Numeric flow ID. Required if `flow` is not provided. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Contents of the PHP client file as a string (JSON format only). |

When requested with `format=raw`, the response is a file attachment:

```
Content-type: text/plain
Content-Disposition: attachment; filename=index.php
```

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — flow with the specified identifier not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": "<?php\n/* AlterCPA TDS client */\n..."
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-id"
}
```

---

<a id="flowscaps"></a>
## Current Way Caps

**Webmaster URL**: `https://domain.com/api/flows/caps.json?id={token}`

**Team URL**: `https://domain.com/api/flows/caps.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/caps.json?id={token}&scope=admin`

Returns the lead limit (cap) state for all ways of a flow. Data is cached for 60 seconds — this allows the function to be used for frequent monitoring without excessive database load.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic flow code. Required if `fid` is not provided. |
| `fid`* | Numeric flow ID. Required if `flow` is not provided. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Associative array of ways, where the key is the numeric way ID. Field composition is described below. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — flow with the specified identifier not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).

Composition of an element in the `data` field:

| Field | Description |
| --- | --- |
| `id` | Numeric way ID. |
| `name` | Way name. |
| `active` | Way is active (true/false). |
| `available` | Way is available to receive traffic: false if a cap is set and has already been reached. |
| `cap` | Configured lead limit (0 — unlimited). |
| `leads` | Number of leads in the current period (0 if no cap is set). |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "678": {
            "id": 678,
            "name": "Main",
            "active": true,
            "available": true,
            "cap": 100,
            "leads": 47
        },
        "679": {
            "id": 679,
            "name": "Reserve",
            "active": true,
            "available": false,
            "cap": 50,
            "leads": 50
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-id"
}
```

---

<a id="flowscreate"></a>
## Create Flow

**Webmaster URL**: `https://domain.com/api/flows/create.json?id={token}`

**Team URL**: `https://domain.com/api/flows/create.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/create.json?id={token}&scope=admin`

Creates a new flow with one initial way. For `gang` scope the `user` parameter is required — the flow is created on behalf of the specified team member. In `user` scope the owner is determined automatically from the token.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `name`* | Flow name. Required. |
| `url`* | URL of the first way. Required if `site` is not provided. |
| `site`* | Storage site ID for the first way. Required if `url` is not provided. |
| `user` | Owner ID. Required for `gang` and `admin` scope. In `user` scope it is set automatically. |
| `gang` | Team ID for `admin` scope. |
| `method` | Operation method for the first way (see below). Default: 0. |
| `simple` | Simplified mode: 1 — enable. In this mode the only way is named `Default`. Default: 0. |

Description of method values (`method`):

- `0` — HTTP 302 (temporary redirect).
- `1` — HTTP 301 (permanent redirect).
- `2` — JavaScript redirect.
- `3` — Iframe.
- `4` — Proxy (server-side proxying request).
- `5` — cURL (server-side page download).
- `6` — Local (local file in storage).
- `7` — Error (return an error).
- `8` — Blank (empty page).
- `9` — JS Go (JavaScript-based redirect).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Full structure of the created flow, including the `ways` field with the first way (see [flows/get](#flowsget)). |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `user-required` — the `user` parameter was not provided (required for `admin` and `gang` scope).
- `limit-reached` — the flow limit under the current plan has been reached.
- `no-name` — flow name was not specified.
- `no-way` — neither `url` nor `site` was provided for the first way.
- `create` — error writing to the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12346,
        "user": 7,
        "gang": 0,
        "name": "New Flow",
        "code": "ab3d",
        "url": "https://click.example.com/ab3d",
        "ways": {
            "680": {
                "id": 680,
                "name": "Default",
                "active": true,
                "url": "https://lp.example.com",
                "method": 0
            }
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-name"
}
```

---

<a id="flowsclone"></a>
## Clone Flow

**Webmaster URL**: `https://domain.com/api/flows/clone.json?id={token}`

**Team URL**: `https://domain.com/api/flows/clone.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/clone.json?id={token}&scope=admin`

Creates a full copy of an existing flow together with all its ways. The statistics of the new flow and its ways are reset to zero.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic code of the flow to clone. Required if `fid` is not provided. |
| `fid`* | Numeric ID of the flow to clone. Required if `flow` is not provided. |
| `name`* | Name of the new flow. Required. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Full structure of the new flow, including the `ways` field (see [flows/get](#flowsget)). |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — source flow not found.
- `access-denied` — the source flow belongs to another user or team.
- `wrong-user` — the source flow owner does not match the provided `user` (`gang` scope).
- `limit-reached` — the flow limit under the current plan has been reached.
- `no-name` — name for the new flow was not specified.
- `create` — error writing to the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12347,
        "user": 7,
        "gang": 0,
        "name": "My Flow (copy)",
        "code": "cc7f",
        "url": "https://click.example.com/cc7f",
        "ways": { }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "limit-reached"
}
```

---

<a id="flowschange"></a>
## Edit Flow Settings

**Webmaster URL**: `https://domain.com/api/flows/change.json?id={token}`

**Team URL**: `https://domain.com/api/flows/change.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/change.json?id={token}&scope=admin`

Updates flow fields. Partial updates are supported: pass only the fields you want to change — the rest will remain unchanged. The `name` field cannot be explicitly cleared: if an empty string is passed, the operation will be rejected.

Cloaking filters (`filter`) are personal — there are no team-level filters. If a filter ID belonging to another user is provided, the field will be automatically reset to zero.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic flow code. Required if `fid` is not provided. |
| `fid`* | Numeric flow ID. Required if `flow` is not provided. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |
| `name` | New flow name. Cannot be empty. |
| `domain` | Parked domain ID (0 — unlink). |
| `algo` | Optimization mode (0..8). |
| `direct` | Share of traffic to the best way (0..100). |
| `rotor` | Rotor mode (0 or 1). |
| `template` | Tracker template ID (0 — disable). |
| `nostore` | Click storage mode (0/1/2). |
| `simple` | Simplified mode (0 or 1). |
| `autourl` | Auto URL (0 or 1). |
| `pixel` | Pixel tracking (0 or 1). |
| `nocookie` | Cookie-free mode (0/1/2). |
| `filter` | Cloaking filter ID from the [Traffic Filtering](api-filter.md) section. Must belong to the flow owner; otherwise it is reset to zero. |
| `bots` | Bot filtering (0 or 1). |
| `filters` | Extended filtering (0 or 1). |
| `geo` | Geo filtering enabled (0 or 1). |
| `lang` | Browser language filtering (0 or 1). |
| `adid` | Ad identifier filtering (0 or 1). |
| `geos` | List of ISO country codes for the geo filter (string or array). Values `ww` and `zz` reset the list. |
| `back` | White page URL for trafficback. |
| `site` | Storage site ID for trafficback. |
| `method` | Trafficback method (see [flows/create](#flowscreate)). |
| `utms` | UTM source tag. |
| `utmc` | UTM campaign tag. |
| `utmn` | UTM content tag. |
| `utmt` | UTM term tag. |
| `utmm` | UTM medium tag. |
| `subid` | SubID. |
| `data1`..`data5` | Additional data fields. |
| `hide1`..`hide6` | Hidden fields. |
| `cpc` | Cost per click. |
| `cur` | Currency, three-letter code. |
| `pbglobal` | Postback URL for any lead status change. |
| `pbnew` | Postback URL when a new lead is created. |
| `pbhold` | Postback URL when a lead transitions to "hold" status. |
| `pbapprove` | Postback URL when a lead is approved. |
| `pbcancel` | Postback URL when a lead is cancelled. |
| `pbtrash` | Postback URL when a lead is sent to trash. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Updated flow structure (see [flows/get](#flowsget)). |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — flow not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `create` — operation error: either no fields were provided for updating, or the `name` field was explicitly cleared, or an error occurred writing to the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12345,
        "name": "Updated Flow Name",
        "algo": 2
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "create"
}
```

---

<a id="flowsremove"></a>
## Delete Flow

**Webmaster URL**: `https://domain.com/api/flows/remove.json?id={token}`

**Team URL**: `https://domain.com/api/flows/remove.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/remove.json?id={token}&scope=admin`

Deletes a flow together with all its ways. Click and lead statistics accumulated for this flow and its ways are **not deleted** and remain available in reports.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic flow code. Required if `fid` is not provided. |
| `fid`* | Numeric flow ID. Required if `flow` is not provided. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Structure of the deleted flow at the time of deletion (see [flows/get](#flowsget)). |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — neither `flow` nor `fid` was provided.
- `bad-id` — flow not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `create` — error performing the delete operation.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 12345,
        "name": "My Flow",
        "code": "ef1c"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "access-denied"
}
```

---

<a id="flowsadd"></a>
## Add Way to Flow

**Webmaster URL**: `https://domain.com/api/flows/add.json?id={token}`

**Team URL**: `https://domain.com/api/flows/add.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/add.json?id={token}&scope=admin`

Creates a new way inside an existing flow. The storage site (`site`), if provided, is validated for ownership by the current user or team.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `flow`* | Symbolic code of the parent flow. Required if `fid` is not provided. |
| `fid`* | Numeric ID of the parent flow. Required if `flow` is not provided. |
| `name`* | Way name. Required. |
| `url`* | Way URL. Required if `site` is not provided. |
| `site`* | Storage site ID. Required if `url` is not provided. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |
| `method` | Way operation method (see [flows/create](#flowscreate)). Default: 0. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Structure of the new way (see [flows/get](#flowsget) section `ways`) plus the `flow` field — numeric ID of the parent flow. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — parent flow identifier was not provided.
- `bad-id` — parent flow not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `no-name` — way name was not specified.
- `no-way` — neither `url` nor `site` was provided.
- `add` — error writing to the database or the specified site is not accessible.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 681,
        "flow": 12345,
        "name": "New Way",
        "active": true,
        "url": "https://lp.example.com",
        "method": 0
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-way"
}
```

---

<a id="flowscopy"></a>
## Copy Way

**Webmaster URL**: `https://domain.com/api/flows/copy.json?id={token}`

**Team URL**: `https://domain.com/api/flows/copy.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/copy.json?id={token}&scope=admin`

Copies an existing way — into the same flow or into another flow belonging to the same owner. The statistics of the new way are reset to zero. Copying a way between flows of different users — including between members of the same team — is not allowed.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `way`* | Numeric ID of the way to copy. Required. |
| `name`* | Name of the new way. Required. |
| `flow` | Symbolic code of the target flow. If not provided, the copy is created in the source flow. |
| `fid` | Numeric ID of the target flow. Used if `flow` is not set. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Structure of the new way plus the `flow` field — numeric ID of the target flow. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — `way` was not provided.
- `bad-id` — way not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `wrong-way` — the way does not belong to an accessible flow.
- `no-name` — name for the new way was not specified.
- `create` — error writing to the database or the owners of the source way and the target flow differ.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 682,
        "flow": 12345,
        "name": "Way Copy",
        "active": true,
        "url": "https://lp.example.com",
        "method": 0
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "create"
}
```

---

<a id="flowsedit"></a>
## Edit Way Settings

**Webmaster URL**: `https://domain.com/api/flows/edit.json?id={token}`

**Team URL**: `https://domain.com/api/flows/edit.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/edit.json?id={token}&scope=admin`

Partially updates way fields. Pass only the fields you want to change — the rest will remain unchanged. The `name` field cannot be explicitly cleared. Storage sites (`site`, `nxsite`) are validated for ownership by the current user or team; if access is denied, the field is silently skipped.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `way`* | Numeric way ID. Required. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |
| `name` | New way name. Cannot be empty. |
| `site` | Storage site ID for the first step. Validated for accessibility. |
| `url` | URL of the first step. |
| `active` | Way activity (0 or 1). |
| `method` | Operation method for the first step (see [flows/create](#flowscreate)). |
| `nostore` | Click storage mode for the way (0 or 1). |
| `pos` | Way weight (≥ 0). |
| `prio` | Way priority (from -9 to 9). |
| `wd1`..`wd7` | Activity by day of week (0 or 1). `wd1` — Monday, `wd7` — Sunday. |
| `fh` | Start hour of the active period (0..23). |
| `fm` | Start minute of the active period (0..59). |
| `th` | End hour of the active period (0..23). |
| `tm` | End minute of the active period (0..59). |
| `tz` | Schedule time zone (e.g., `Europe/Moscow`). |
| `limit` | Cap period (0..7). |
| `cap` | Lead limit (0 — unlimited). |
| `goal` | Goal for cap calculation. |
| `status` | Lead statuses counted toward the cap. |
| `nxsite` | Storage site ID for the second step. Validated for accessibility. |
| `nxurl` | URL of the second step. |
| `nxmethod` | Second step method (see [flows/create](#flowscreate)). |
| `geo` | Allowed countries: ISO codes as comma-separated string or array. Values `ww` and `zz` reset the list. |
| `nogeo` | Blocked countries (same format as `geo`). |
| `mobile` | Device type filter (0..4). |
| `bots` | Pass bots through (0 or 1). |
| `adid` | Ad identifier filter (0 or 1). |
| `filters` | Extended filtering (0 or 1). |
| `lang` | Browser language filtering (0 or 1). |
| `uwl` | Referrer URL whitelist of words (string, words separated by newline). |
| `ubl` | Referrer URL blacklist of words (string, words separated by newline). |
| `hide1`..`hide6` | Hidden way fields. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Updated way structure plus the `flow` field — numeric ID of the parent flow. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — `way` was not provided.
- `bad-id` — way not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `wrong-way` — the way does not belong to an accessible flow.
- `create` — operation error: either no fields were provided for updating, or the `name` field was explicitly cleared, or an error occurred writing to the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 678,
        "flow": 12345,
        "name": "Main",
        "active": false,
        "url": "https://lp.example.com",
        "method": 0,
        "cap": 200
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "create"
}
```

---

<a id="flowsdel"></a>
## Delete Way

**Webmaster URL**: `https://domain.com/api/flows/del.json?id={token}`

**Team URL**: `https://domain.com/api/flows/del.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/flows/del.json?id={token}&scope=admin`

Deletes an individual way from a flow. Click and lead statistics accumulated for this way are **not deleted** and remain available in reports. Deletion of the last way in a flow is not blocked — a flow with no ways will be unable to receive traffic.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `way`* | Numeric way ID. Required. |
| `user` | User ID for `gang` or `admin` scope. |
| `gang` | Team ID for `admin` scope. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Structure of the deleted way at the time of deletion plus the `flow` field — numeric ID of the parent flow. |

Error codes:

- `scope` — invalid value of the `scope` parameter or insufficient permissions.
- `unpaid` — no active subscription.
- `no-id` — `way` was not provided.
- `bad-id` — way not found.
- `access-denied` — the flow belongs to another user or team.
- `wrong-user` — the flow owner does not match the provided `user` (`gang` scope).
- `wrong-way` — the way does not belong to an accessible flow.
- `create` — error performing the delete operation.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 678,
        "flow": 12345,
        "name": "Main",
        "active": true,
        "url": "https://lp.example.com"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "wrong-way"
}
```
