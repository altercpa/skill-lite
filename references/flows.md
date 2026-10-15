# Flows & Ways API Reference

## Overview

The `flows` application manages **flows** (traffic channels) and **ways** (destinations/paths within a flow). All functions support `scope` parameter.

Base URL: `api/flows/{func}.json?id={token}[&scope=user|gang|admin]`

**Important:** `scope` must be in the GET query string even when POSTing JSON.

---

## Scope Behavior

| Scope | Flow ownership check |
|---|---|
| `user` | The flow belongs to the current user |
| `gang` | The flow belongs to the caller's team; optionally filter by `user=N` |
| `admin` | No ownership check |

Error `unpaid` is returned when paid subscriptions are enabled and the subscription is not valid.

---

## Flow Object (full shape)

Returned by `get`, `create`, `clone`, `change`. Key fields:

```json
{
  "id": 12345,
  "user": 7,
  "gang": 0,
  "name": "My Flow",
  "domain": 0,
  "code": "ef1c",
  "url": "https://click.example.com/ef1c",
  "gif": "<img src=\"https://click.example.com/ef1c/pixel.gif\" .../>",
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
    "source": false, "campaign": false, "content": false, "term": false,
    "medium": false, "subid": false, "data1": false, "data2": false,
    "data3": false, "data4": false, "data5": false,
    "hide1": false, "hide2": false, "hide3": false,
    "hide4": false, "hide5": false, "hide6": false,
    "cpc": false, "cur": "usd"
  },
  "stats": { "cr": 0.0, "epc": 0.0, "appr": 0.0, "money": 0.0 },
  "postback": {
    "global": false, "new": false, "hold": false,
    "approve": false, "cancel": false, "trash": false
  },
  "ways": { "<way_id>": { /* way object */ } }
}
```

The `ways` field is only present in `get`, `create`, `clone`, `change` responses.

---

## Field Value Reference

**`algo` — traffic distribution algorithm:**
| Value | Algorithm |
|---|---|
| 0 | Round-robin (equal rotation) |
| 1 | Random |
| 2 | Best CR (highest conversion rate wins more traffic) |
| 3 | Best EPC (highest earnings per click) |
| 4 | Locked CR (same as 2 but threshold-based) |
| 5 | Locked EPC |
| 6 | Weighted (by `pos` field) |
| 7 | Manual (by `prio` field) |
| 8 | Smart (auto-detect best performer) |

**`nostore` (flow level):**
| Value | Meaning |
|---|---|
| 0 | Normal — store click and lead data |
| 1 | Skip click storage (clicks are not stored) |
| 2 | Skip both click and lead storage |

**`nocookie` (flow level):**
| Value | Meaning |
|---|---|
| 0 | Normal — set cookies on click |
| 1 | Skip cookies (stateless mode) |
| 2 | Skip cookies but still track via click ID in URL |

**`limit` (way cap period):**
| Value | Period |
|---|---|
| 0 | No limit |
| 1 | Per day |
| 2 | Per week |
| 3 | Per month |
| 4 | Total (lifetime) |
| 5 | Per hour |
| 6 | Per 15 minutes |
| 7 | Per 30 minutes |

**`goal` (way cap goal — which leads count toward `cap`):** the value is the goal slot number **plus one**, with two values below the slots; it is not a plain slot number.
| Value | Leads counted |
|---|---|
| 0 | Any lead (no goal condition) |
| 1 | Leads with **no** goal |
| n + 1 | Leads tagged with goal slot `n` (`n` ≥ 1, the slot number as keyed in `gclid/goals`) |

The cap counts the leads whose own `goal` equals `max(0, goal - 1)`; a negative value sent to `flows/edit` is stored as `0`. Write the same encoding that `flows/get` returns.

**`mobile` (way level, same as filter campaign):**
| Value | Meaning |
|---|---|
| 0 | Any device |
| 1 | Mobile only |
| 2 | Desktop only |
| 3 | iOS only |
| 4 | Mobile, non-iOS |

**OS IDs** (for `os` filter parameter in `flows/edit`):
| ID | OS |
|---|---|
| 0 | Other |
| 1 | Windows |
| 2 | macOS |
| 3 | Linux |
| 4 | iOS |
| 5 | Android |
| 6 | Windows Mobile |
| 7 | Symbian |
| 8 | BlackBerry |

**Browser IDs** (for `browser` filter parameter):
| ID | Browser |
|---|---|
| 0 | Other |
| 1 | Chrome |
| 2 | Firefox |
| 3 | IE/MSIE |
| 4 | Safari |
| 5 | Opera |
| 6 | Edge |
| 7 | Facebook in-app |
| 8 | Instagram in-app |
| 9 | TikTok in-app |
| 10 | WebView |
| 11 | Bot |

---

## Way Object

Each way inside `ways`:

```json
{
  "id": 678,
  "flow": 12345,
  "name": "Offer 1",
  "active": true,
  "nostore": 0,
  "prio": 0,
  "site": 0,
  "spath": false,
  "url": "https://offer.example.com/?click={click}",
  "method": "http302",
  "nxsite": 0,
  "nxpath": false,
  "nxurl": "",
  "nxmethod": "http302",
  "pos": 100,
  "w1": true, "w2": true, "w3": true, "w4": true, "w5": true, "w6": true, "w7": true,
  "from": "00:00",
  "to": "23:59",
  "tz": "UTC",
  "shift": 0,
  "limit": 0,
  "cap": 0,
  "goal": 0,
  "status": 0,
  "geo": [],
  "nogeo": [],
  "mobile": 0,
  "bots": false,
  "adid": false,
  "filter": 0,
  "filters": false,
  "lang": false,
  "uwl": [],
  "ubl": [],
  "cr": 0.0,
  "epc": 0.0,
  "appr": 0.0,
  "money": 0.0,
  "crg": {},
  "epcg": {},
  "hide1": "", "hide2": "", "hide3": "", "hide4": "", "hide5": "", "hide6": ""
}
```

---

## flows/list — Paginated flow list

```
GET api/flows/list.json?id={token}[&scope=...]
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `scope` | string | GET only. `user`/`gang`/`admin` |
| `user` | int | Filter by user (admin/gang scope) |
| `gang` | int | Filter by gang (admin scope) |
| `flow` | string | Flow symbolic code (e.g. `ef1c`) |
| `fid` / `uid` | int | Numeric flow ID |
| `domain` | int | Filter by parked domain ID |
| `search` | string | Full-text search by flow name |
| `page` | int | Page number (≥1, default 1) |
| `show` | int | Page size (0 = 100, default 100) |
| `short` | bool | Short format: only id, user, gang, name, code, url |

**Response:**
```json
{"status":"ok","total":42,"page":1,"show":100,"data":{"12345":{/* flow */}}}
```

In short mode each entry has only: `id`, `user`, `gang`, `name`, `code`, `url`.

**Errors:** `scope`, `unpaid`

---

## flows/get — Single flow with ways

```
GET api/flows/get.json?id={token}&fid={id}[&scope=...]
```

**Parameters:** `flow` (symbolic code) or `fid` (numeric ID), plus scope params.

**Response:** `{"status":"ok","data":{/* full flow object with ways */}}`

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`

---

## flows/file — Download TDS client PHP

```
GET api/flows/file.raw?id={token}&fid={id}
```

Use `format=raw` (`.raw` extension) to get the file download. Other formats return the file as a string in `data`.

**Response (raw):** `Content-Disposition: attachment; filename=index.php` + PHP file body.

**Response (json):** `{"status":"ok","data":"<?php ..."}`

---

## flows/caps — Current lead caps

```
GET api/flows/caps.json?id={token}&fid={id}
```

**Response:**
```json
{
  "status": "ok",
  "data": {
    "678": {"id":678,"name":"Way #1","active":true,"available":true,"cap":100,"leads":47}
  }
}
```

`available: false` when `cap > 0` and `leads >= cap`.

---

## flows/create — Create flow

```
POST api/flows/create.json?id={token}&scope=user
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Flow name |
| `url` | string | one of | Initial way URL |
| `site` | int | one of | Initial way site ID from storage |
| `method` | int | no | Way method: 0=http302, 1=http301, 2=redirect, 3=iframe, 4=proxy, 5=curl, 6=local, 7=error, 8=blank, 9=jsgo |
| `simple` | bool | no | Simplified mode (single way named "Default") |
| `user` | int | admin/gang | Owner user ID |

**Response:** Full flow object with `ways`.

**Errors:** `scope`, `user-required`, `limit-reached`, `unpaid`, `no-name`, `no-way`, `create`

---

## flows/clone — Clone flow

```
POST api/flows/clone.json?id={token}&fid={id}&name=NewName
```

**Parameters:** `flow`/`fid` (source), `name` (new flow name), scope params.

**Response:** Full flow object of the new clone.

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`, `limit-reached`, `no-name`, `create`

---

## flows/change — Edit flow settings

```
POST api/flows/change.json?id={token}&fid={id}
```

Accepts any subset of flow fields. POST wins over GET on collision.

**Editable fields:**

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Flow name (empty = error) |
| `domain` | int | Parked domain ID |
| `algo` | int | Traffic distribution: 0=Round-robin, 1=Random, 2=CR, 3=EPC, 4=Best CR, 5=Best EPC, 6=Weighted, 7=Manual, 8=Smart |
| `direct` | int | % traffic to best way (0–100) |
| `rotor` | int | Enable rotor (0/1) |
| `template` | int | Template ID |
| `nostore` | int | Skip storage: 0=normal, 1=skip click storage, 2=skip click+lead storage |
| `simple` | bool | Simplified mode |
| `autourl` | bool | Auto-generate flow URL |
| `pixel` | bool | Enable pixel tracking |
| `nocookie` | int | Cookie mode: 0=normal, 1=no cookies, 2=no cookies but track via URL |
| `filter` | int | Filter campaign ID (must be owned by same user; 0 = disable) |
| `bots` | bool | Block bots |
| `filters` | bool | Enable flow-level filter |
| `geo` | bool | Enable geo filter |
| `lang` | bool | Enable language filter |
| `adid` | bool | Require ad click-id |
| `geos` | string/array | ISO country codes. `ww`/`zz` = reset. |
| `back` | string | Traffic-back URL |
| `site` | int | Traffic-back site ID |
| `method` | int | Traffic-back method |
| `utms`, `utmc`, `utmn`, `utmt`, `utmm` | string | UTM tag names |
| `subid` | string | SubID field name |
| `data1`–`data5` | string | Custom data field names |
| `hide1`–`hide6` | string | Hidden field names |
| `cpc` | string | Click cost expression |
| `cur` | string | Currency (3 letters) |
| `pbglobal` | string | Global postback URL |
| `pbnew` | string | New lead postback URL |
| `pbhold` | string | Hold postback URL |
| `pbapprove` | string | Approve postback URL |
| `pbcancel` | string | Cancel postback URL |
| `pbtrash` | string | Trash postback URL |

**Response:** Updated full flow object with `ways`.

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`, `create`

---

## flows/remove — Delete flow

```
POST api/flows/remove.json?id={token}&fid={id}
```

**Response:** `{"status":"ok","data":{/* deleted flow */}}`

**Note:** The click and lead statistics of the flow are NOT deleted (by design, for reports).

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`

---

## flows/add — Add way to flow

```
POST api/flows/add.json?id={token}&fid={flow_id}&name=WayName&url=https://...
```

**Parameters:**

| Name | Required | Description |
|---|---|---|
| `flow`/`fid` | yes | Parent flow ID |
| `name` | yes | Way name |
| `url` | one of | Way destination URL |
| `site` | one of | Site ID from storage |
| `method` | no | Display method (see `flows/create`) |

**Response:** `{"status":"ok","data":{/* way object with flow field */}}`

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`, `no-name`, `no-way`, `add`

---

## flows/copy — Copy a way

```
POST api/flows/copy.json?id={token}&way={way_id}&name=NewName[&fid={target_flow_id}]
```

Copies a way to the same flow or a different flow (must be same user).

**Parameters:** `way` (numeric ID), `name` (new way name), optionally `flow`/`fid` (target flow).

**Response:** `{"status":"ok","data":{/* new way with flow */}}`

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`, `wrong-way`, `no-name`, `create`

---

## flows/edit — Edit way settings

```
POST api/flows/edit.json?id={token}&way={way_id}
```

**Editable way fields:**

| Parameter | Type | Description |
|---|---|---|
| `name` | string | Way name (empty = error) |
| `site` | int | Site ID (validated by access rights) |
| `url` | string | Destination URL |
| `active` | int | 0/1 |
| `method` | int | Display method |
| `nostore` | int | 0=normal, 1=skip storage |
| `pos` | int | Weight (≥0) |
| `prio` | int | Priority (-9 to +9) |
| `wd1`–`wd7` | bool | Days of week (Mon–Sun) |
| `fh`, `fm`, `th`, `tm` | int | Schedule hours/minutes from/to |
| `limit` | int | Cap period: 0=no, 1=day, 2=week, 3=month, 4=total, 5=hour, 6=15min, 7=30min |
| `cap` | int | Lead limit per period |
| `goal` | int | Which leads count toward the cap: `0` = any lead, `1` = leads with no goal, `n + 1` = leads of goal slot `n` (see the `goal` table above); negative = 0 |
| `status` | int | Lead statuses for cap |
| `tz` | string | Timezone identifier |
| `nxsite` | int | Second-step site ID |
| `nxurl` | string | Second-step URL |
| `nxmethod` | int | Second-step method |
| `geo` | string/array | Allowed countries. `ww`/`zz` = reset. |
| `nogeo` | string/array | Blocked countries |
| `mobile` | int | 0=Any, 1=Mobile, 2=Desktop, 3=iOS, 4=Mobile non-iOS |
| `bots` | bool | Pass bots |
| `adid` | bool | Require ad click-id |
| `filters` | bool | Enable way-level filter |
| `lang` | bool | Language filter |
| `uwl` | string | URL whitelist (newline-separated) |
| `ubl` | string | URL blacklist |
| `hide1`–`hide6` | string | Hidden data fields |

**Response:** `{"status":"ok","data":{/* updated way with flow */}}`

**Errors:** `scope`, `unpaid`, `no-id`, `bad-id`, `access-denied`, `wrong-user`, `wrong-way`, `create`

---

## flows/del — Delete a way

```
POST api/flows/del.json?id={token}&way={way_id}
```

**Response:** `{"status":"ok","data":{/* deleted way with flow */}}`

**Note:** Statistics are NOT deleted. Deleting the last way in a flow leaves it without any destinations.

---

## Common Errors Reference

| Error | Meaning |
|---|---|
| `scope` | Invalid scope or insufficient permissions |
| `unpaid` | Active subscription required |
| `no-id` | Flow or way ID not provided |
| `bad-id` | Flow not found |
| `wrong-way` | Way not found or doesn't belong to accessible flow |
| `access-denied` | Flow belongs to different user/gang |
| `wrong-user` | In gang scope, flow belongs to different team member |
| `no-name` | Name parameter missing or empty |
| `no-way` | Neither url nor site provided |
| `limit-reached` | Flow limit per plan is exhausted |
| `user-required` | Admin scope without specifying user |
| `create` | Saving failed, or general operation error |
| `add` | Way could not be created, or site not accessible |
