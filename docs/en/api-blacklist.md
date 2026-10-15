# Blacklists

The `blacklist` API section manages the traffic-filter **blacklists**: named lists of blocked IPv4 addresses, IPv4 networks, IPv6 networks, and request-header rules. Every visit that goes through the cloaking filter or the tracker flow filter is checked against these lists. With this API you can create, configure and delete lists, add, change and remove their entries one by one, import and export entries as CSV files, and find out which lists can be attached to a filter campaign. The same lists are managed in the web interface in the "My filters", "Team filters" and (for administrators) "Traffic filters" sections; the meaning of the entry fields is also described in the [Blacklists](blacklist.md) user guide.

A list has one of four kinds that is chosen when the list is created and cannot be changed afterwards: IPv4 addresses, IPv4 networks, IPv6 networks, or request headers (see [List kinds](#blacklisttypes)). An entry of a list is a single address, a network, or a header rule of the corresponding kind.

Request format: parameters are read from the query string and from the POST body (form or JSON); a value from the body takes priority. The `scope` parameter is read from the query string only, even if the request itself is sent by POST. The `import` function is the only one that requires `multipart/form-data`, and `export` returns a CSV file instead of JSON. Entry IDs, list IDs and campaign IDs are plain integers.

Error responses always have the form `{"status": "error", "error": "<code>"}`; refusals of the mirror settings of the [edit](#blacklistedit) function also contain a `message` field with a localized explanation. All error codes are summarized in the [Errors](#blacklisterrors) section, and the numeric limits in the [Limits](#blacklistlimits) section.

<a id="blacklistfilter"></a>
## How lists are used in filtering

- A filter campaign (see [Traffic Filtering](api-filter.md)) checks visits against the lists selected in its `list` field. If no list is selected, the campaign uses all active default lists: the system ones, the owner's own and the owner's team's. The tracker flow filter uses only the default lists of the kinds IP addresses, IP networks and IPv6 addresses; header lists are not used by it.
- Only active lists can be selected in a campaign or work as default lists.
- Added entries start working immediately. The only delay is for lists of the IPv4 networks kind: network lookups are cached for 666 seconds (about 11 minutes) per /24 block, so a newly added network may take that long to take effect for addresses that were checked recently.
- A single address or header rule can also be added with the simplified function [Block IP or header](api-filter.md#filterblock); the functions of this section are the full management counterpart.

<a id="blacklistscopes"></a>
## Scopes and list owners

Every function works in one of three spaces selected by the `scope` parameter (see [API interface](api.md)). The space defines which lists the function sees and where the new lists are created.

| Scope | Description |
| --- | --- |
| `scope=user` | Default. Your personal lists ("My filters"). Available to every active user. New lists are created as your personal lists: active and not default. |
| `scope=gang` | Alias: `team`. Shared lists of your team ("Team filters"). Available only to team leaders, even if shared access to the team lists is enabled for all members in the web interface. The optional `user` parameter must be a member of your team, otherwise the `scope` error is returned; it matters for the [avail](#blacklistavail) function only. New lists are created as team lists: active and not default. |
| `scope=admin` | Administrators only. Without `gang` and `user` the function works with global lists (system lists, including automatically updated supplier lists; "Traffic filters"). With `gang` it works with the lists of that team, with `user` with the personal lists of that user; if both are passed, the team wins. New lists are created in the same space. |

Additional parameters of the scope selection:

| Field | Description |
| --- | --- |
| `gang` | Admin scope only: team ID. Must be a number, otherwise `bad-gang` is returned; the [avail](#blacklistavail) and [add](#blacklistadd) functions also check that the team exists. |
| `user` | Admin scope: user ID. Must be a number, otherwise `bad-user` is returned; the [avail](#blacklistavail) and [add](#blacklistadd) functions also check that the user exists. Team scope: see above. |
| `uu` | Administrator keys only: ID of a user on whose behalf the request is executed. For example, `scope=user&uu=123` works with the personal lists of user 123. |

Notes:

- A regular user cannot read or change global (system) lists with the functions [list](#blacklistlist), [show](#blacklistshow) and [edit](#blacklistedit); such lists are visible to them in read-only form through [avail](#blacklistavail) only.
- A list created in the admin scope without `gang` and `user` is a **global list**. Unlike personal and team lists it is created **inactive** and **not default**: activate it with the [edit](#blacklistedit) function (`status=1`), otherwise it will not be offered in campaigns.
- Functions that take the `list` parameter work only with the lists of the selected space: a list of another owner is reported as `bad-list`.

<a id="blacklistdict"></a>
## Reference values

The values below are used by several functions of this section.

<a id="blacklisttypes"></a>
### List kinds

The kind of a list is passed in the `type` parameter of the [add](#blacklistadd) and [list](#blacklistlist) functions and is returned in the `type` field of a list:

- `1` — IP addresses: single IPv4 addresses.
- `2` — IP networks: IPv4 networks with a prefix length of 8 to 31.
- `3` — IPv6 addresses: IPv6 networks with a prefix length of 32 to 128 (a single address is a network of size 128).
- `4` — Headers: rules for the values of request headers.

> **Attention!** The `type` parameter has two different meanings. In the functions [add](#blacklistadd) and [list](#blacklistlist) it is the **kind of the list** (1–4). In the functions [show](#blacklistshow), [block](#blacklistblock) and [change](#blacklistchange) it is the **header ID** of an entry of a header list (see [Header IDs](#blacklistheaders)).

<a id="blacklistcomments"></a>
### Reason codes

The `comment` field of an entry is a number from 0 to 255 that explains why the entry was blocked. The interface knows the following reasons:

- `1` — Public list
- `2` — Suspicious
- `3` — Corporate
- `4` — Different User-Agent
- `5` — GEO error
- `6` — Header error
- `7` — Multiple errors
- `8` — Bad request
- `9` — Bot

The value `0` means no reason. The `show` function returns the localized names in `dataset.comments`.

<a id="blacklistheaders"></a>
### Header IDs

The `type` field of a header-list entry is the ID of the request header to check:

- `1` — User-Agent
- `2` — Accept
- `3` — Accept-Charset
- `4` — Accept-Encoding
- `5` — Accept-Language
- `6` — Authorization
- `7` — Cache-Control
- `8` — Connection
- `9` — Content-Disposition
- `10` — Date
- `11` — Expect
- `12` — From
- `13` — Host
- `14` — If-Match
- `15` — If-Modified-Since
- `16` — If-None-Match
- `17` — If-Range
- `18` — If-Unmodified-Since
- `19` — Max-Forwards
- `20` — Pragma
- `21` — Proxy-Authorization
- `22` — Range
- `23` — Referer
- `24` — TE
- `25` — Trailer
- `26` — Transfer-Encoding
- `27` — Upgrade
- `28` — Via
- `30` — X-Forwarded-For
- `31` — X-Purpose
- `32` — X-FB-HTTP-Engine
- `33` — X-Frame-Options
- `34` — X-WAP-Profile
- `35` — X-Real-IP
- `36` — Client-IP
- `37` — CF-Connecting-IP
- `40` — Priority
- `41` — Sec-Fetch-Mode
- `42` — Sec-Fetch-Site
- `43` — Sec-Fetch-Dest
- `44` — Sec-Fetch-User
- `45` — Sec-CH-UA
- `46` — Sec-CH-UA-Mobile
- `47` — Sec-CH-UA-Platform
- `48` — DNT
- `49` — Origin
- `50` — Upgrade-Insecure-Requests
- `51` — Save-Data
- `52` — X-Forwarded-Host
- `53` — X-Forwarded-Proto
- `100` — Remote Address
- `101` — Remote Port
- `102` — Request Method
- `103` — Request URI
- `104` — Request Query String
- `106` — Request Scheme

The IDs 29, 38, 39 and 105 do not exist. An entry with any other ID from 1 to 255 is accepted and stored, but it never matches anything during filtering. The `show` function returns the full ID map in `dataset.headers`.

---

<a id="blacklistavail"></a>
## Available lists

**Webmaster URL**: `https://domain.com/api/blacklist/avail.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/avail.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/avail.json?id={token}&scope=admin`

**Method**: `GET` or `POST`.

Returns the lists that the owner of a filter campaign can attach to it: first the active system lists, then the owner's active personal lists, then the active lists of the owner's team; every group is sorted by name. Inactive lists are never returned. For every list the function reports whether it is a default list and, if a campaign is specified, whether it is already selected in that campaign. This is the data of the "Filters set" selector of the campaign screen.

This is the only function through which a regular user sees the system lists (name, kind and flags, read-only).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Filter campaign ID (see [Campaign setup](api-filter.md#filterset)). If the parameter is passed, the campaign must exist in the current scope, otherwise `no-site` is returned. The campaign owner becomes the list owner, and the `selected` flags show the campaign selection. If the parameter is omitted, the owner is determined as described for `user`, and `selected` is always `false`. |
| `gang` | Admin scope only: team ID. The team must exist (`bad-gang` otherwise). Limits the campaign lookup to the campaigns of the team members. |
| `user` | Admin scope: ID of an existing user (`bad-user` otherwise). Team scope: ID of a team member. Without `uid` it selects the list owner and is required in the team and admin scopes (`no-user` otherwise); in the webmaster scope the owner is the key owner. With `uid` it limits the campaign lookup to the campaigns of this user. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Associative array of the offered lists, where the key is the list ID. Composition is described below. |
| `dataset` | Additional data. Contains the `types` field: kind code → localized kind name (see [List kinds](#blacklisttypes)). |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number or the team does not exist.
- `bad-user` — `user` is not a number or the user does not exist.
- `no-site` — the campaign `uid` is not found in the current scope.
- `no-user` — `uid` is omitted and the list owner is not defined (no `user`).

Composition of the `data` element:

| Field | Description |
| --- | --- |
| `id` | List ID. |
| `name` | List name. |
| `type` | List kind (see [List kinds](#blacklisttypes)). |
| `role` | List owner kind: `feed` — supplier list updated automatically, `team` — team list, `user` — personal list, `admin` — global list managed manually. |
| `default` | `true` if the list is a default list for this owner. |
| `selected` | `true` if the list is selected in the campaign `uid`. An inactive list stored in the campaign is not reported. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "3": {
            "id": 3,
            "name": "AlterCPA Lite IPv4",
            "type": 1,
            "role": "feed",
            "default": true,
            "selected": false
        },
        "12": {
            "id": 12,
            "name": "My proxies",
            "type": 2,
            "role": "user",
            "default": false,
            "selected": true
        }
    },
    "dataset": {
        "types": {
            "1": "IP addresses",
            "2": "IP networks",
            "3": "IPv6 addresses",
            "4": "Headers"
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-site"
}
```

<a id="blacklistlist"></a>
## List of blacklists

**Webmaster URL**: `https://domain.com/api/blacklist/list.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/list.json?id={token}&scope=admin`

**Method**: `GET` or `POST`.

Returns the blacklists of the current scope with their settings: active lists first; within each group the lists are sorted by name. In the admin scope without `gang` and `user` the function returns the global lists, including supplier lists (see [Scopes](#blacklistscopes)).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `type` | Filter by list kind, 1 to 4 (see [List kinds](#blacklisttypes)). By default, lists of all kinds are returned. Any other value returns `bad-type`. |
| `search` | Search by list name. The text is split into words by spaces and commas; words shorter than 2 characters are ignored. A list is returned only if its name contains all the words (case-insensitive). If no words remain, the search is not applied. |
| `page` | Page number, starting from 1. Defaults to 1. |
| `show` | Number of lists per page, from 0 to 1000. Defaults to 0: all lists are returned at once, without paging and without the `nav` field. A page beyond the last one returns an empty `data`. |
| `gang` | Admin scope only: return the lists of this team. |
| `user` | Admin scope only: return the personal lists of this user. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Associative array of lists, where the key is the list ID. Composition is described below. An empty result is returned as an empty array. |
| `nav` | Paging data, returned only if `show` is greater than 0: `total` — total number of matching lists, `page` — current page, `pages` — number of pages, `show` — page size. |
| `dataset` | Additional data: `types` — kind code → localized kind name, `periods` — update period in seconds → localized name (all supported periods). |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `bad-type` — `type` is not a number from 1 to 4.

Composition of the `data` element (the same list object is returned by the functions [add](#blacklistadd) and [edit](#blacklistedit) and in `dataset.list` of the function [show](#blacklistshow)):

| Field | Description |
| --- | --- |
| `id` | List ID. |
| `name` | List name. |
| `type` | List kind (see [List kinds](#blacklisttypes)). |
| `role` | List owner kind: `feed` — supplier list updated automatically, `team` — team list, `user` — personal list, `admin` — global list managed manually. |
| `user` | Owner user ID, 0 if the list is not personal. |
| `gang` | Owner team ID, 0 if the list is not a team one. |
| `status` | `true` if the list is active. Only active lists can be selected in campaigns. |
| `default` | `true` if the list is a default list: it is used by campaigns that have no explicit selection. |
| `size` | Number of entries in the list. |
| `update` | Unix time of the last successful automatic update, 0 if there has been none. |
| `locked` | `true` if the entries cannot be changed by hand: the list is a supplier list or a mirror of a file at an address. |
| `mirror` | `true` if the list is a mirror of a file at the address `url` (automatic update is on). |
| `auto` | `true` if automatic update is on (for a supplier list or a mirror). |
| `url` | Address of the mirrored file, empty if the list is not a mirror. |
| `period` | Update period of the mirror, in seconds. |
| `periods` | Array of the update periods allowed for this list, in seconds. Empty for supplier lists. |
| `check` | Unix time of the last attempt to update the mirror. |
| `fail` | Number of consecutive failed update attempts. |
| `feed` | ID of the supplier list, empty string for other lists. |
| `private` | `true` if the supplier list requires an access key. |
| `key` | Access key of the supplier list. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "12": {
            "id": 12,
            "name": "My proxies",
            "type": 2,
            "role": "user",
            "user": 123,
            "gang": 0,
            "status": true,
            "default": true,
            "size": 150,
            "update": 0,
            "locked": false,
            "mirror": false,
            "auto": false,
            "url": "",
            "period": 0,
            "periods": [10800, 21600, 43200, 86400, 259200, 604800],
            "check": 0,
            "fail": 0,
            "feed": "",
            "private": false,
            "key": ""
        }
    },
    "nav": {
        "total": 1,
        "page": 1,
        "pages": 1,
        "show": 20
    },
    "dataset": {
        "types": {
            "1": "IP addresses",
            "2": "IP networks",
            "3": "IPv6 addresses",
            "4": "Headers"
        },
        "periods": {
            "600": "10 minutes",
            "1800": "30 minutes",
            "3600": "1 hour",
            "10800": "3 hours",
            "21600": "6 hours",
            "43200": "12 hours",
            "86400": "1 day",
            "259200": "3 days",
            "604800": "1 week"
        }
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

<a id="blacklistshow"></a>
## List entries

**Webmaster URL**: `https://domain.com/api/blacklist/show.json?id={token}&list={list}`

**Team URL**: `https://domain.com/api/blacklist/show.json?id={token}&list={list}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/show.json?id={token}&list={list}&scope=admin`

**Method**: `GET` or `POST`.

Returns the entries of one list with paging, search and filters. The function works for any list of the current scope, including locked ones (supplier lists and mirrors). Entries are sorted by address (for header lists, by value).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `search` | Search in the entries; the meaning depends on the list kind (see below). An empty string disables the search. |
| `comment` | Filter by reason code, digits only, from 0 to 255 (see [Reason codes](#blacklistcomments)). `0` or omitted: no filter. A value that is not a number or is above 255 returns `bad-comment`. |
| `type` | Only for header lists: filter by header ID, from 0 to 255 (see [Header IDs](#blacklistheaders)). `0` or omitted: no filter. A non-numeric or out-of-range value returns `bad-type`. For other kinds it is ignored. |
| `page` | Page number, starting from 1. Defaults to 1. |
| `show` | Number of entries per page, from 1 to 1000. Defaults to 50. |
| `gang` | Admin scope only: team ID, selects the list owner (see [Scopes](#blacklistscopes)). |
| `user` | Admin scope only: user ID, selects the list owner. |

Meaning of `search` by list kind:

- IP addresses — an exact IPv4 address. An invalid address returns `bad-search`.
- IP networks — returns the networks that contain this IPv4 address. An invalid address returns `bad-search`.
- IPv6 addresses — returns the networks that contain this IPv6 address. An invalid address returns `bad-search`.
- Headers — a substring of the entry value.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Associative array of entries, where the key is the entry ID. Composition is described below. A page beyond the last one returns an empty array. |
| `nav` | Paging data: `total` — number of matching entries, `page` — current page, `pages` — number of pages, `show` — page size. |
| `dataset` | Additional data: `list` — the list object (see [List of blacklists](#blacklistlist)), `comments` — reason code → localized name, `headers` — header ID → header name (header lists only). |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `bad-search` — `search` is not a valid address for the list kinds that require one.
- `bad-comment` — `comment` is not a number from 0 to 255.
- `bad-type` — `type` is not a number from 0 to 255 (header lists).

Composition of the `data` element (the fields depend on the list kind; the same entry object is returned by the functions [block](#blacklistblock) and [change](#blacklistchange)):

| Field | Description |
| --- | --- |
| `id` | Entry ID; used in the `item` parameter of the functions [remove](#blacklistremove) and [change](#blacklistchange). |
| `ip` | IP addresses, IP networks and IPv6 addresses: the address (for networks, the first address of the network). |
| `size` | IP networks and IPv6 addresses: prefix length. |
| `to` | IP networks and IPv6 addresses: the last address of the network. |
| `value` | Headers: header text or regular expression. |
| `type` | Headers: header ID (see [Header IDs](#blacklistheaders)). |
| `regexp` | Headers: `true` if `value` is a regular expression, `false` if it is a substring. |
| `white` | Headers: `true` for a white rule, `false` for a black rule (see [block](#blacklistblock)). |
| `comment` | Reason code (see [Reason codes](#blacklistcomments)). |
| `ttl` | Unix time when the entry expires, 0 if it never expires. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "501": {
            "id": 501,
            "ip": "12.34.56.0",
            "size": 24,
            "to": "12.34.56.255",
            "comment": 3,
            "ttl": 0
        },
        "502": {
            "id": 502,
            "ip": "78.90.0.0",
            "size": 16,
            "to": "78.90.255.255",
            "comment": 2,
            "ttl": 1893456000
        }
    },
    "nav": {
        "total": 2,
        "page": 1,
        "pages": 1,
        "show": 50
    },
    "dataset": {
        "list": {
            "id": 12,
            "name": "My proxies",
            "type": 2,
            "role": "user",
            "user": 123,
            "gang": 0,
            "status": true,
            "default": true,
            "size": 2,
            "update": 0,
            "locked": false,
            "mirror": false,
            "auto": false,
            "url": "",
            "period": 0,
            "periods": [10800, 21600, 43200, 86400, 259200, 604800],
            "check": 0,
            "fail": 0,
            "feed": "",
            "private": false,
            "key": ""
        },
        "comments": {
            "1": "Public list",
            "2": "Suspicious",
            "3": "Corporate",
            "4": "Different User-Agent",
            "5": "GEO error",
            "6": "Header error",
            "7": "Multiple errors",
            "8": "Bad request",
            "9": "Bot"
        }
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-list"
}
```

<a id="blacklistadd"></a>
## Create a list

**Webmaster URL**: `https://domain.com/api/blacklist/add.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/add.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/add.json?id={token}&scope=admin`

**Method**: `POST`.

Creates an empty list with the given name and kind in the current scope. A personal or team list is created active and not default. A **global** list (admin scope without `gang` and `user`) is created **inactive** and **not default**: to make it available in campaigns, activate it with the [edit](#blacklistedit) function. The list kind cannot be changed after creation. Automatic update and the supplier fields are configured with the [edit](#blacklistedit) function.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `name`* | Required parameter. List name, up to 100 characters. HTML tags are removed from the name, quotes are converted to HTML entities, and leading and trailing spaces are trimmed. An empty name returns `no-name`. |
| `type`* | Required parameter. List kind, from 1 to 4 (see [List kinds](#blacklisttypes)). Any other value returns `bad-type`. |
| `gang` | Admin scope only: team ID, the list is created as a team list. The team must exist. |
| `user` | Admin scope only: user ID, the list is created as a personal list of this user. The user must exist. Ignored if `gang` is set. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | The created list object (see [List of blacklists](#blacklistlist)). |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `no-name` — the name is empty.
- `bad-type` — `type` is not a number from 1 to 4.
- `bad-gang` — `gang` is not a number or the team does not exist.
- `bad-user` — `user` is not a number or the user does not exist.
- `db` — the list could not be saved.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 13,
        "name": "Corporate networks",
        "type": 2,
        "role": "user",
        "user": 123,
        "gang": 0,
        "status": true,
        "default": false,
        "size": 0,
        "update": 0,
        "locked": false,
        "mirror": false,
        "auto": false,
        "url": "",
        "period": 0,
        "periods": [10800, 21600, 43200, 86400, 259200, 604800],
        "check": 0,
        "fail": 0,
        "feed": "",
        "private": false,
        "key": ""
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

<a id="blacklistedit"></a>
## Edit a list

**Webmaster URL**: `https://domain.com/api/blacklist/edit.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/edit.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/edit.json?id={token}&scope=admin`

**Method**: `POST`.

Changes the settings of a list. This is a partial update: only the parameters that are present in the request are changed. If no settings are passed, the list is returned unchanged. The kind of a list cannot be changed. The function also works for locked lists: supplier lists and mirrors can be configured, only their entries are locked. Changes of the `status` and `default` flags apply to campaigns immediately.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `name` | New list name (cleaned the same way as in [add](#blacklistadd)). An empty name returns `bad-name`. |
| `status` | Active flag, `1` or `0`. Only active lists can be selected in campaigns. See the note on boolean values below. |
| `default` | Default flag, `1` or `0`. A default list is used by campaigns that have no explicit selection. |
| `key` | Only for supplier lists: access key for receiving updates. Ignored for other lists. |
| `auto` | `1` or `0`. For a supplier list, the automatic update switch. For other lists, turns the mirroring of a file at an address on or off (see below). |
| `url` | Not for supplier lists. Address of the CSV file to mirror. An empty string clears the address. |
| `period` | Not for supplier lists. Update period of the mirror, in seconds (see below). |

> **Attention!** Boolean parameters (`status`, `default`, `auto`) are not interpreted as words: `0`, an empty string and `"0"` turn the flag off, **any other value turns it on, including the string `false`**. Always pass `1` or `0`.

Mirroring of a file (parameters `url`, `period`, `auto`):

A manual list can become an automatic copy of a CSV file published at an address. The file must be in the same format that the [export](#blacklistexport) function produces for this kind of list. At every update the list becomes an exact copy of the file: entries that are absent from the file are removed and manual changes are lost. While mirroring is on, the list is locked and its entries cannot be changed by hand; to edit the entries again, turn `auto` off. If any of `url`, `period` and `auto` is passed, the missing ones are taken from the stored settings, and the following rules are checked:

- The address must be a public `http` or `https` link with a host name, up to 500 characters long, without a login and password, with port 80 or 443 (or without a port). A host given as a number must be a public IPv4 address. Otherwise `bad-url` is returned. Turning `auto` on with an empty address also returns `bad-url`.
- The period must be one of the values allowed for the list: 10800, 21600, 43200, 86400, 259200, 604800 seconds (3 hours, 6 hours, 12 hours, 1 day, 3 days, 1 week) for personal and team lists, and additionally 600, 1800, 3600 seconds (10, 30 minutes, 1 hour) for global lists. The allowed values are also returned in the `periods` field of the list. A wrong period with `auto` on returns `bad-period`; with `auto` off the period is silently stored as 0.
- The list must not be selected in the filtering settings as a list into which the system writes blocked addresses automatically, otherwise `target` is returned: the mirror would delete the entries written by the system.
- A user or a team can have at most 5 mirrored lists, otherwise `limit` is returned. Global lists have no such limit.
- When the address changes or `auto` is turned on, the update state of the list is reset, and the file is loaded again at the next background update.
- A file is applied only if it contains at least one valid row, no more than 10% of rows are invalid, and (for personal and team lists) there are no more than 200 000 rows (500 for header lists) and the size does not exceed 8 388 608 bytes (268 435 456 bytes for global lists). A file that does not pass the checks is not applied; the `check` and `fail` fields of the list show the last attempt and the number of consecutive failures.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `message` | Localized explanation for the errors `bad-url`, `bad-period`, `target` and `limit`. |
| `data` | The updated list object (see [List of blacklists](#blacklistlist)). |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `bad-name` — the name is empty.
- `bad-url` — the file address is invalid, or `auto` is on with an empty address.
- `bad-period` — the update period is not allowed for the list while `auto` is on.
- `target` — the list is selected for automatic writing of blocked addresses in the filtering settings, so mirroring cannot be turned on.
- `limit` — the limit of 5 mirrored lists is reached.
- `db` — the changes could not be saved.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 13,
        "name": "Corporate networks",
        "type": 2,
        "role": "user",
        "user": 123,
        "gang": 0,
        "status": true,
        "default": true,
        "size": 0,
        "update": 0,
        "locked": true,
        "mirror": true,
        "auto": true,
        "url": "https://example.com/corp-nets.csv",
        "period": 86400,
        "periods": [10800, 21600, 43200, 86400, 259200, 604800],
        "check": 0,
        "fail": 0,
        "feed": "",
        "private": false,
        "key": ""
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-period",
    "message": "Choose an update period from the list."
}
```

<a id="blacklistdel"></a>
## Delete a list

**Webmaster URL**: `https://domain.com/api/blacklist/del.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/del.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/del.json?id={token}&scope=admin`

**Method**: `POST`.

Deletes a list together with all its entries. The deletion is irreversible. Locked lists can be deleted too: a deleted supplier list reappears with the next update of the supplier lists (as an inactive one). Campaigns and settings that referenced the list keep its ID, but it is no longer taken into account.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `gang` | Admin scope only: team ID, selects the list owner. |
| `user` | Admin scope only: user ID, selects the list owner. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Result data: the `id` field contains the ID of the deleted list. |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `db` — the list could not be deleted.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 13
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-list"
}
```

<a id="blacklistblock"></a>
## Add an entry

**Webmaster URL**: `https://domain.com/api/blacklist/block.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/block.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/block.json?id={token}&scope=admin`

**Method**: `POST`.

Adds one entry to a list. If the list already contains an entry with the same identity, the entry is updated instead of being duplicated (upsert); the `added` field of the response tells which happened. The function does not work for locked lists (supplier lists and mirrors): `locked` is returned.

The set of required fields depends on the list kind:

- IP addresses: `ip`.
- IP networks and IPv6 addresses: `ip` and `size`.
- Headers: `value` and `type` (optionally `regexp` and `white`).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `ip`* | Required parameter for IP addresses, IP networks and IPv6 addresses. IPv4 address for IP addresses and IP networks, IPv6 address for IPv6 addresses. For networks it may be any address of the network: it is rounded down to the first address of the network. A missing value returns `no-ip`. Invalid, private and reserved addresses (such as `10.0.0.1`, `192.168.1.1`, `127.0.0.1`, `0.0.0.0`) return `bad-ip`. |
| `size`* | Required parameter for IP networks and IPv6 addresses. Prefix length: 8 to 31 for IP networks, 32 to 128 for IPv6 addresses (a single IPv6 address is 128). A missing or out-of-range value returns `bad-size`. |
| `value`* | Required parameter for headers. Header text or regular expression, up to 250 characters, UTF-8. A missing value returns `no-value`; an empty value, invalid UTF-8 or a text longer than 250 characters returns `bad-value`. |
| `type`* | Required parameter for headers. Header ID, from 1 to 255 (see [Header IDs](#blacklistheaders)). A missing or out-of-range value returns `bad-type`. |
| `regexp` | Headers only. `1`: `value` is a regular expression with delimiters (for example `/bot\d+/i`); `0`: `value` is searched as a case-insensitive substring. Defaults to 0. The expression is not checked on saving; a broken expression is skipped during filtering. |
| `white` | Headers only. `1`: white rule, the visit is blocked if the header does **not** match or is missing; `0`: black rule, the visit is blocked if the header matches. Defaults to 0. |
| `comment` | Reason code from 0 to 255 (see [Reason codes](#blacklistcomments)). Defaults to 0. A non-numeric value is treated as 0. |
| `ttl` | Unix time when the entry expires. Defaults to 0: never. A value above 4 294 967 295 is reduced to this maximum. |

Entry expiry (`ttl`): an entry with `ttl` set is deleted automatically after the specified time; the cleaning runs every minute and recounts the size of the list. Until then the entry stays in effect, so an expired entry may keep blocking visits for up to a minute. The entries written by the automatic blocking of the system get their own `ttl` as well.

Identity of an entry (what counts as the "same" entry):

- IP addresses: the address.
- IP networks and IPv6 addresses: the first address of the network together with the prefix length (the address is rounded to the network start before the comparison).
- Headers: the pair `type` and `value`; the value is compared case-insensitively, except for rules with `regexp` set, which are compared exactly.

When the entry is found, only the fields passed in the request are updated: `comment` and `ttl` for all kinds, and also `value`, `type`, `regexp` and `white` for headers; duplicates of the same header rule, if there are any, are deleted. Otherwise a new entry is added. Lists have no limit on the number of entries. For networks, see the delay note in the [introduction](#blacklistfilter).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | The entry object (see [List entries](#blacklistshow)) with the additional field `added`: `true` if a new entry was added, `false` if an existing one was updated. |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `locked` — the list is a supplier list or a mirror and cannot be changed by hand.
- `no-ip` — `ip` is not passed.
- `no-value` — `value` is not passed.
- `bad-size` — the prefix length is out of range.
- `bad-type` — the header ID is not a number from 1 to 255.
- `bad-ip` — the address is invalid, private or reserved.
- `bad-value` — the header value is empty, longer than 250 characters or not UTF-8.
- `db` — the entry could not be saved.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 503,
        "ip": "12.34.56.0",
        "size": 24,
        "to": "12.34.56.255",
        "comment": 3,
        "ttl": 0,
        "added": true
    }
}
```

**Example of a successful server response for a header list:**

```json
{
    "status": "ok",
    "data": {
        "id": 77,
        "value": "HeadlessChrome",
        "type": 1,
        "regexp": false,
        "white": false,
        "comment": 6,
        "ttl": 0,
        "added": false
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "locked"
}
```

<a id="blacklistremove"></a>
## Remove an entry

**Webmaster URL**: `https://domain.com/api/blacklist/remove.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/remove.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/remove.json?id={token}&scope=admin`

**Method**: `POST`.

Removes one entry from a list and recounts the list size. The function does not work for locked lists (supplier lists and mirrors). There is no function that empties a list at once: remove the entries one by one or delete the whole list with the [del](#blacklistdel) function.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `item`* | Required parameter. Entry ID (the `id` field of the [show](#blacklistshow) function). |
| `gang` | Admin scope only: team ID, selects the list owner. |
| `user` | Admin scope only: user ID, selects the list owner. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Result data: the `id` field contains the ID of the removed entry. |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `locked` — the list is a supplier list or a mirror and cannot be changed by hand.
- `no-item` — `item` is not passed or equals 0.
- `bad-item` — there is no entry with this ID in the list.
- `db` — the entry could not be deleted.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 503
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-item"
}
```

<a id="blacklistchange"></a>
## Change an entry

**Webmaster URL**: `https://domain.com/api/blacklist/change.json?id={token}`

**Team URL**: `https://domain.com/api/blacklist/change.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/change.json?id={token}&scope=admin`

**Method**: `POST`.

Partially updates one entry: fields that are not passed keep their stored values. The new values are validated the same way as in the [block](#blacklistblock) function. If none of the entry fields is passed, the entry is returned unchanged. If after the change the entry would duplicate another entry of the same list, the `duplicate` error is returned. The function does not work for locked lists (supplier lists and mirrors).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `item`* | Required parameter. Entry ID (the `id` field of the [show](#blacklistshow) function). |
| `ip` | IP addresses, IP networks and IPv6 addresses: new address or network start (see [block](#blacklistblock)). |
| `size` | IP networks and IPv6 addresses: new prefix length. |
| `value` | Headers: new header text or expression. |
| `type` | Headers: new header ID (see [Header IDs](#blacklistheaders)). |
| `regexp` | Headers: `1` or `0` (see [block](#blacklistblock)). |
| `white` | Headers: `1` or `0` (see [block](#blacklistblock)). |
| `comment` | Reason code from 0 to 255. |
| `ttl` | Unix time when the entry expires, 0 for never. |
| `gang` | Admin scope only: team ID, selects the list owner. |
| `user` | Admin scope only: user ID, selects the list owner. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | The updated entry object (see [List entries](#blacklistshow)); unlike [block](#blacklistblock), it has no `added` field. |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `locked` — the list is a supplier list or a mirror and cannot be changed by hand.
- `no-item` — `item` is not passed or equals 0.
- `bad-item` — there is no entry with this ID in the list.
- `bad-size` — the prefix length is out of range.
- `bad-type` — the header ID is not a number from 1 to 255.
- `bad-ip` — the address is invalid, private or reserved.
- `bad-value` — the header value is empty, longer than 250 characters or not UTF-8.
- `duplicate` — the entry would duplicate another entry of the list.
- `db` — the entry could not be saved.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 503,
        "ip": "12.34.56.0",
        "size": 24,
        "to": "12.34.56.255",
        "comment": 7,
        "ttl": 1893456000
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "duplicate"
}
```

<a id="blacklistimport"></a>
## Import entries

**Webmaster URL**: `https://domain.com/api/blacklist/import.json?id={token}&list={list}`

**Team URL**: `https://domain.com/api/blacklist/import.json?id={token}&list={list}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/import.json?id={token}&list={list}&scope=admin`

**Method**: `POST` with `Content-Type: multipart/form-data`. The CSV file is sent in the `import` field; `list` can be passed in the query string or in the form.

Merges the entries of a CSV file into a list. Import never deletes entries: entries that are present in the file are updated or added, and all other entries of the list stay as they are. The function does not work for locked lists (supplier lists and mirrors). To replace the content of a list completely, remove or recreate the list, or use the mirroring of a file (see [edit](#blacklistedit)).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `import`* | Required parameter. CSV file (see below). |
| `gang` | Admin scope only: team ID, selects the list owner. |
| `user` | Admin scope only: user ID, selects the list owner. |

File format:

- UTF-8 encoding; a BOM at the beginning is skipped. There is no header line: a header line would be counted as a rejected row. Blank lines are skipped.
- The delimiter is detected automatically among comma, semicolon and tab. Values may be enclosed in double quotes. The files produced by [export](#blacklistexport) are always accepted.
- Columns are positional; missing trailing columns are treated as 0.
- Every row is validated the same way as in [block](#blacklistblock); an invalid row is counted as rejected and skipped, it does not stop the import.

Columns of the file by list kind (see [List kinds](#blacklisttypes)):

| List kind | Columns |
| --- | --- |
| IP addresses | `ip`, `comment`, `ttl` |
| IP networks | `ip`, `size`, `comment`, `ttl` |
| IPv6 addresses | `ip`, `size`, `comment`, `ttl` |
| Headers | `value`, `type`, `regexp`, `comment`, `ttl`, `white` |

Merge rules:

- Entries are matched by identity, as in [block](#blacklistblock). A matched entry is **updated** with the `comment` and `ttl` of the file (for headers, also `value`, `type`, `regexp` and `white`); a column that is missing in the file overwrites the stored value with 0, so a row without a `ttl` makes the entry permanent. Other rows are **added**.
- Entries of the list that are absent from the file are not touched. The only deletion is the removal of duplicate copies of a header rule that is present in the file; they are counted in `del`.
- A duplicate row inside the file is counted as an update.
- There is no limit on the number of rows. The size of the file is limited: 8 388 608 bytes (8 MiB) for personal and team lists, 268 435 456 bytes (256 MiB) for global lists, and by the upload limit of the server; a larger file returns `limit`.
- A file without any valid row is not an error: `add` is 0 and `bad` is the number of rows.
- For lists of the IP networks kind, see the delay note in the [introduction](#blacklistfilter).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Import result, composition detailed below. |

Error codes:

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.
- `locked` — the list is a supplier list or a mirror and cannot be changed by hand.
- `limit` — the file is larger than the allowed size or the upload limit.
- `no-file` — the file is not uploaded.
- `bad-file` — the uploaded file cannot be read.
- `db` — the entries could not be saved.

Composition of the `data` element:

| Field | Description |
| --- | --- |
| `add` | Number of added entries. |
| `upd` | Number of updated entries. |
| `bad` | Number of rejected rows (invalid or not saved). |
| `del` | Number of deleted duplicates of header rules. Always 0 for address and network lists. |
| `size` | Resulting number of entries in the list. |

**Example request implementation:**

```bash
curl -F "import=@nets.csv" "https://domain.com/api/blacklist/import.json?id={token}&list=13"
```

Example of the file `nets.csv` for a list of the IP networks kind:

```
12.34.56.0,24,3,0
78.90.0.0,16,2,1893456000
```

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "add": 1,
        "upd": 1,
        "bad": 0,
        "del": 0,
        "size": 151
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "locked"
}
```

<a id="blacklistexport"></a>
## Export entries

**Webmaster URL**: `https://domain.com/api/blacklist/export.json?id={token}&list={list}`

**Team URL**: `https://domain.com/api/blacklist/export.json?id={token}&list={list}&scope=gang`

**Admin URL**: `https://domain.com/api/blacklist/export.json?id={token}&list={list}&scope=admin`

**Method**: `GET` or `POST`.

Downloads all entries of a list as a CSV file. The `.json` extension of the address is required for routing, but it is ignored: the answer is always a file, not JSON. The function works for any list of the current scope, including locked ones. The file has no header line, no paging and no guaranteed order of lines; the format is exactly the one that the [import](#blacklistimport) function reads and that a mirrored file must have.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. ID of the list in the current scope. |
| `gang` | Admin scope only: team ID, selects the list owner. |
| `user` | Admin scope only: user ID, selects the list owner. |

The answer is a file with `Content-Type: text/csv` and the attachment name `filter-<kind>-<list_id>.csv`, where the kind is `ip`, `net`, `ip6` or `bl`. Lines by list kind, separated by commas:

| List kind | Line |
| --- | --- |
| IP addresses | `ip`, `comment`, `ttl` |
| IP networks | `ip`, `size`, `comment`, `ttl` |
| IPv6 addresses | `ip`, `size`, `comment`, `ttl` |
| Headers | `value` (always in double quotes, a quote inside is doubled), `type`, `regexp` (0 or 1), `comment`, `ttl`, `white` (0 or 1) |

Error codes (returned as JSON before any file output):

- `scope` — the scope cannot be resolved or you have no rights for it.
- `bad-gang` — `gang` is not a number.
- `bad-user` — `user` is not a number.
- `no-list` — `list` is not passed or equals 0.
- `bad-list` — the list is not found in the current scope.

**Example of a successful server response** (the file `filter-net-13.csv`):

```
12.34.56.0,24,3,0
78.90.0.0,16,2,1893456000
```

**Example of a successful server response for a header list** (the file `filter-bl-14.csv`):

```
"HeadlessChrome",1,0,6,0,0
"/^python-requests\/\d+/i",1,1,9,0,0
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-list"
}
```

<a id="blacklisterrors"></a>
## Errors

All functions return errors in the `error` field of the response. Codes of this section:

| Code | Description |
| --- | --- |
| `scope` | All functions. The scope cannot be resolved: the user is inactive or banned, the user is not an administrator but requests `admin`, is not a team leader but requests `gang`, or `user` is not a member of the team. |
| `bad-gang` | All functions, admin scope. `gang` is not a number; in [avail](#blacklistavail) and [add](#blacklistadd) also if the team does not exist. |
| `bad-user` | All functions, admin scope. `user` is not a number; in [avail](#blacklistavail) and [add](#blacklistadd) also if the user does not exist. |
| `no-site` | [avail](#blacklistavail). The campaign `uid` is not found in the current scope. |
| `no-user` | [avail](#blacklistavail). `uid` is omitted and the list owner is not defined. |
| `no-name` | [add](#blacklistadd). The list name is empty. |
| `bad-name` | [edit](#blacklistedit). The list name is empty. |
| `bad-type` | [list](#blacklistlist) and [add](#blacklistadd): the list kind is not 1 to 4. [show](#blacklistshow), [block](#blacklistblock) and [change](#blacklistchange): the header ID is incorrect. |
| `no-list` | All functions with the `list` parameter. `list` is not passed or equals 0. |
| `bad-list` | All functions with the `list` parameter. The list is not found in the current scope. |
| `locked` | [block](#blacklistblock), [remove](#blacklistremove), [change](#blacklistchange), [import](#blacklistimport). The list is a supplier list or a mirror. |
| `bad-search` | [show](#blacklistshow). The search text is not a valid address for address and network lists. |
| `bad-comment` | [show](#blacklistshow). `comment` is not a number from 0 to 255. |
| `no-ip` | [block](#blacklistblock). `ip` is not passed. |
| `no-value` | [block](#blacklistblock). `value` is not passed. |
| `bad-ip` | [block](#blacklistblock), [change](#blacklistchange). The address is invalid, private or reserved. |
| `bad-value` | [block](#blacklistblock), [change](#blacklistchange). The header value is empty, longer than 250 characters or not UTF-8. |
| `bad-size` | [block](#blacklistblock), [change](#blacklistchange). The prefix length is out of range. |
| `no-item` | [remove](#blacklistremove), [change](#blacklistchange). `item` is not passed. |
| `bad-item` | [remove](#blacklistremove), [change](#blacklistchange). There is no such entry in the list. |
| `duplicate` | [change](#blacklistchange). The entry would duplicate another entry of the list. |
| `bad-url` | [edit](#blacklistedit). The file address is invalid. Comes with `message`. |
| `bad-period` | [edit](#blacklistedit). The update period is not allowed. Comes with `message`. |
| `target` | [edit](#blacklistedit). The list is used for automatic writing of blocked addresses. Comes with `message`. |
| `limit` | [edit](#blacklistedit): more than 5 mirrored lists. [import](#blacklistimport): the file is too large. |
| `no-file` | [import](#blacklistimport). The file is not uploaded. |
| `bad-file` | [import](#blacklistimport). The uploaded file cannot be read. |
| `db` | [add](#blacklistadd), [edit](#blacklistedit), [del](#blacklistdel), [block](#blacklistblock), [remove](#blacklistremove), [change](#blacklistchange), [import](#blacklistimport). The data could not be saved. |

<a id="blacklistlimits"></a>
## Limits

| Limit | Value |
| --- | --- |
| List name | Up to 100 characters. |
| Number of lists | Not limited. |
| Number of entries in a list | Not limited. |
| Entries per page | [show](#blacklistshow): 1–1000, defaults to 50. [list](#blacklistlist): 0–1000, 0 means all lists. |
| Prefix length | IP networks: 8–31. IPv6 addresses: 32–128. |
| Header rule | Text up to 250 characters, header ID 1–255. |
| Reason code (`comment`) | 0–255. |
| Expiry (`ttl`) | Unix time, from 0 (never) to 4 294 967 295. |
| Import file size | 8 388 608 bytes (8 MiB) for personal and team lists; 268 435 456 bytes (256 MiB) for global lists. The number of rows is not limited. |
| Mirrored lists | 5 per user or team; not limited for global lists. |
| Mirrored file | Link up to 500 characters, port 80 or 443. Same size limit as for import; up to 200 000 rows (500 for header lists) for personal and team lists, not limited for global lists. |
| Mirror update period | 3 hours – 1 week for personal and team lists; 10 minutes – 1 week for global lists. |
| Network cache | The IPv4 network lookups are cached for 666 seconds (see the [introduction](#blacklistfilter)). |
