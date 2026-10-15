# Parked Domains

The parked domains API lets you manage domains parked on tracker storage: retrieve the list, add domains, trigger a status check, delete domains, and link them to [flows](api-flows.md). Domains are used as the primary link address for a flow and support IDN (internationalized domain names). It also lets you manage confirmation files served at the domain root — small text/markup files (e.g. search-engine ownership verification files, `ads.txt`) that prove ownership of the domain or serve static declarations, without deploying a real web server behind the parked domain.

Each domain belongs either to a specific user (personal domain) or to the team as a whole (team domain). When working with `scope=gang` the API serves exclusively team domains — shared across the entire team. Personal domains of team members are not accessible in gang scope. The same ownership convention applies to confirmation files: a file's owner is copied from its parent domain's owner at creation time and does not change afterwards.

<a id="domainslist"></a>
## Domains List

**Webmaster URL**: `https://domain.com/api/domains/list.json?id={token}`

**Team URL**: `https://domain.com/api/domains/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/list.json?id={token}&scope=admin`

The function returns a list of parked domains with filtering and pagination support. All filters are combined with AND logic. Billing is not checked when `scope=admin`.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `gang` | For `scope=admin`: team ID to filter by. |
| `flow` | Symbolic flow code to filter by binding (format `flow_code`, as returned by the [flows API](api-flows.md)). |
| `fid` | Numeric flow ID to filter by binding. |
| `url` | Substring search by human-readable domain name. |
| `addr` | Substring search by the punycode form of the domain. |
| `ip` | IP address to filter by domain host. |
| `status` | Filter by domain status (see below). |
| `uid` | Exact search by numeric domain ID. |
| `page` | Results page (≥ 1). Default: 1. |
| `show` | Page size. Default: 0 (return all records without limit). |

`status` field values:

- `0` — Waiting (domain added, no check performed yet).
- `1` — Active (HTTP check passed successfully).
- `2` — OK + SSL (HTTPS check passed successfully).
- `3` — Error (domain is not responding).
- `4` — Wrong (domain responds but points to a different server).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of records matching the filters. Calculated by a separate query when `show > 0`. |
| `page` | Current page. |
| `show` | Page size. |
| `data` | Dictionary of domains, keyed by numeric domain ID (see below). |

Error codes:

- `scope` — workspace permission check failed (invalid scope, missing team lead or administrator rights).
- `unpaid` — tracker access has expired or is unpaid (only for `scope=user` and `scope=gang`).

Fields of a record in `data`:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Human-readable domain name (IDN-decoded). |
| `addr` | Punycode form of the domain name. |
| `ip` | IP address recorded at the last check. |
| `user` | ID of the owner user; `0` for a team domain. |
| `gang` | ID of the owner team; `0` for a personal domain. |
| `flow` | Numeric ID of the linked flow; `0` if not linked. |
| `status` | Domain status (see above). |
| `stage` | String status: `wait`, `active`, `ok`, `error`, `wrong`. |
| `time` | Unix timestamp when the domain was added. |
| `created` | WHOIS registration date in `Ymd` format (e.g., `20210914`). |
| `expires` | WHOIS registration expiry date in `Ymd` format. |
| `isok` | WHOIS delegation status: `0` — no data, `1` — delegated, `2` — issues detected. |

Example of a successful server response:

```json
{
    "status": "ok",
    "total": 2,
    "page": 1,
    "show": 0,
    "data": {
        "42": {
            "uid": 42,
            "url": "пример.рф",
            "addr": "xn--e1afmkfd.xn--p1ai",
            "ip": "203.0.113.10",
            "user": 0,
            "gang": 3,
            "flow": 18,
            "status": 2,
            "stage": "ok",
            "time": 1714723200,
            "created": 20210914,
            "expires": 20260914,
            "isok": 1
        }
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "scope"
}
```

<a id="domainsadd"></a>
## Add Domain

**Webmaster URL**: `https://domain.com/api/domains/add.json?id={token}`

**Team URL**: `https://domain.com/api/domains/add.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/add.json?id={token}&scope=admin`

The function adds a single parked domain. The domain name may be supplied with a protocol (`https://example.com`) or with a `www.` prefix — the tracker normalizes it automatically. The added domain is assigned status 0 (Waiting); the actual check is triggered later, either by the CRON schedule or by an explicit call to [domain check](#domainscheck). When working with `scope=gang` the domain is immediately created as a team domain belonging to the team as a whole, not to any individual member.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `url`* | Domain name. Required. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=admin` without `gang`: ID of the owner user for the new domain. |
| `gang` | For `scope=admin`: ID of the owner team for the new domain. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the created domain (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `unpaid` — tracker access has expired or is unpaid (only for `scope=user` and `scope=gang`).
- `no-user` — neither `user` nor `gang` was provided for `scope=admin`.
- `no-url` — the `url` parameter was not provided or is empty.
- `bad-url` — the domain name was empty after normalization.
- `duplicate` — a domain with this name is already registered in the system.
- `bad-domain` — unable to add the domain: an IP address was supplied, it is a system tracker domain, the name failed IDNA validation, or a database error occurred.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric ID of the created domain. |
| `user` | ID of the owner user; `0` for a team domain. |
| `gang` | ID of the owner team; `0` for a personal domain. |
| `url` | Human-readable domain name. |
| `addr` | Punycode form of the domain name. |

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 137,
        "user": 0,
        "gang": 3,
        "url": "пример.рф",
        "addr": "xn--e1afmkfd.xn--p1ai"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "duplicate"
}
```

<a id="domainscheck"></a>
## Check Domain

**Webmaster URL**: `https://domain.com/api/domains/check.json?id={token}`

**Team URL**: `https://domain.com/api/domains/check.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/check.json?id={token}&scope=admin`

The function immediately triggers a domain status check and returns up-to-date data. During the check, the tracker sends HTTP and HTTPS requests to the domain and updates the WHOIS information. The response contains the domain state after the check completes.

The domain is identified by `uid` (numeric ID) or by name (`url` or `domain`). If the domain is not found or the caller lacks sufficient scope permissions, the function returns `not-found` without specifying the reason.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Domain name. Used if `uid` is not provided. |
| `domain` | Alternative field for the domain name. Used if neither `uid` nor `url` is provided. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `uid`, `url`, or `domain` must be provided.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Domain data after the check (field composition is the same as in the [domains list](#domainslist)). |

Error codes:

- `not-found` — domain not found, no identifier provided, or insufficient scope permissions.

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "uid": 42,
        "url": "пример.рф",
        "addr": "xn--e1afmkfd.xn--p1ai",
        "ip": "203.0.113.10",
        "user": 0,
        "gang": 3,
        "flow": 18,
        "status": 2,
        "stage": "ok",
        "time": 1714723200,
        "created": 20210914,
        "expires": 20260914,
        "isok": 1
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="domainsdel"></a>
## Delete Domain

**Webmaster URL**: `https://domain.com/api/domains/del.json?id={token}`

**Team URL**: `https://domain.com/api/domains/del.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/del.json?id={token}&scope=admin`

The function deletes a domain from the system. Along with the domain record, all associated files (verification and other files in the domain root) are deleted. Flows that had this domain selected as their link domain lose the binding — it is recommended to update the affected [flows](api-flows.md) in advance.

The domain is identified by `uid` or by name (`url` or `domain`). When working with `scope=gang` only team domains of that team are accessible — personal domains of team members are not available.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Domain name. Used if `uid` is not provided. |
| `domain` | Alternative field for the domain name. Used if neither `uid` nor `url` is provided. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `uid`, `url`, or `domain` must be provided.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the deleted domain (see below). |

Error codes:

- `not-found` — domain not found, no identifier provided, or insufficient scope permissions.
- `bad-domain` — unable to delete the domain: database error or ownership condition not met.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric ID of the deleted domain. |
| `user` | ID of the owner user; `0` for a team domain. |
| `gang` | ID of the owner team; `0` for a personal domain. |
| `url` | Human-readable domain name. |
| `addr` | Punycode form of the domain name. |

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "user": 0,
        "gang": 3,
        "url": "пример.рф",
        "addr": "xn--e1afmkfd.xn--p1ai"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="domainsglue"></a>
## Link Domain to Flow

**Webmaster URL**: `https://domain.com/api/domains/glue.json?id={token}`

**Team URL**: `https://domain.com/api/domains/glue.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/glue.json?id={token}&scope=admin`

The function links a domain to a flow or unlinks it. The flow is specified by symbolic code (`flow`) or numeric ID (`fid`). To unlink a domain from a flow, pass `flow=0` or provide neither `flow` nor `fid`.

The flow must belong to the same owner as the domain: for `scope=user` — the same user, for `scope=gang` — the same team. Linking to a flow owned by a different party returns a `glue` error.

The domain is identified by `uid` or by name (`url` or `domain`).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Domain name. Used if `uid` is not provided. |
| `domain` | Alternative field for the domain name. Used if neither `uid` nor `url` is provided. |
| `flow` | Symbolic flow code (e.g., `ef1c`). |
| `fid` | Numeric flow ID. Used if `flow` is not provided or cannot be decoded. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `uid`, `url`, or `domain` must be provided.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Domain data after linking (see below). |

Error codes:

- `not-found` — domain not found, no identifier provided, or insufficient scope permissions.
- `glue` — linking failed: the flow belongs to a different owner or a database error occurred.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric domain ID. |
| `user` | ID of the owner user; `0` for a team domain. |
| `gang` | ID of the owner team; `0` for a personal domain. |
| `url` | Human-readable domain name. |
| `addr` | Punycode form of the domain name. |
| `flow` | Numeric ID of the linked flow; `0` when unlinked. |
| `code` | Symbolic code of the linked flow; `false` when unlinked. |

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "user": 0,
        "gang": 3,
        "url": "пример.рф",
        "addr": "xn--e1afmkfd.xn--p1ai",
        "flow": 18,
        "code": "ef1c"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "glue"
}
```

<a id="domainsfiles"></a>
## List Confirmation Files

**Webmaster URL**: `https://domain.com/api/domains/files.json?id={token}`

**Team URL**: `https://domain.com/api/domains/files.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/files.json?id={token}&scope=admin`

The function returns all confirmation files stored at a domain's root, with their full content. A confirmation file is a small text/markup file (e.g. a search-engine ownership verification file, or `ads.txt`) served directly by the tracker's front-end when a request to the parked domain matches a single root-level path with an allowed extension — no real web server is required behind the domain. There is no pagination: the whole file list of the domain is always returned. Unlike [domains list](#domainslist) and [add domain](#domainsadd), this function does not check billing status.

The domain is identified by `uid` or by name (`url` or `domain`) — the same convention as [check domain](#domainscheck), [delete domain](#domainsdel), and [link domain to flow](#domainsglue).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Domain name. Used if `uid` is not provided. |
| `domain` | Alternative field for the domain name. Used if neither `uid` nor `url` is provided. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `uid`, `url`, or `domain` must be provided.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of confirmation files on the domain. |
| `data` | Dictionary of confirmation files, keyed by numeric file ID (see below). |

Error codes:

- `not-found` — domain not found, no identifier provided, or insufficient scope permissions.

Fields of a record in `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric file ID. |
| `dom` | Numeric ID of the parent domain. |
| `user` | ID of the owner user; `0` for a team file. |
| `gang` | ID of the owner team; `0` for a personal file. |
| `name` | File name at the domain root, e.g. `ads.txt`. |
| `data` | Raw file content. |

Example of a successful server response:

```json
{
    "status": "ok",
    "total": 2,
    "data": {
        "7": {
            "id": 7,
            "dom": 42,
            "user": 0,
            "gang": 3,
            "name": "ads.txt",
            "data": "google.com, pub-0000000000000000, DIRECT"
        },
        "8": {
            "id": 8,
            "dom": 42,
            "user": 0,
            "gang": 3,
            "name": "google1234567890.html",
            "data": "google-site-verification: google1234567890.html"
        }
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="domainsconfirm"></a>
## Create Confirmation File

**Webmaster URL**: `https://domain.com/api/domains/confirm.json?id={token}`

**Team URL**: `https://domain.com/api/domains/confirm.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/confirm.json?id={token}&scope=admin`

The function creates a confirmation file at the domain root. Since a file name is unique within a domain, calling this function again with the same `name` for the same domain silently overwrites the existing file's content instead of returning a duplicate error — the function behaves as an upsert (create-or-replace). This function does not check billing status.

The domain (not the file) is identified by `uid` or by name (`url` or `domain`) — the same convention as [check domain](#domainscheck), [delete domain](#domainsdel), and [link domain to flow](#domainsglue).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid` | Numeric domain ID. |
| `url` | Domain name. Used if `uid` is not provided. |
| `domain` | Alternative field for the domain name. Used if neither `uid` nor `url` is provided. |
| `name`* | File name at the domain root, e.g. `ads.txt`, `google123.html`. Required parameter. |
| `data` | File content. Defaults to an empty string. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `uid`, `url`, or `domain` must be provided.

`name` is cleaned before use: any character outside `[A-Za-z0-9._-]` is stripped, and leading/trailing `.`, `-`, `_` are trimmed. The extension (the part after the last `.`, lowercased) must be one of `txt`, `js`, `json`, `htm`, `html`, `css`, `xml` — any other extension, or a name that becomes empty after cleanup, is rejected. `data` is trimmed and truncated to 3500 characters; empty content is allowed.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the created or updated file (see below). |

Error codes:

- `not-found` — domain not found, no identifier provided, or insufficient scope permissions.
- `no-name` — the `name` parameter was not provided or is empty.
- `bad-file` — the file could not be created: invalid file name or extension after cleanup, or a database error occurred.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric file ID. |
| `dom` | Numeric ID of the parent domain. |
| `user` | ID of the owner user; `0` for a team file. |
| `gang` | ID of the owner team; `0` for a personal file. |
| `name` | File name at the domain root. |
| `data` | Raw file content (see above). |

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 8,
        "dom": 42,
        "user": 0,
        "gang": 3,
        "name": "google1234567890.html",
        "data": "google-site-verification: google1234567890.html"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "bad-file"
}
```

<a id="domainschange"></a>
## Change Confirmation File

**Webmaster URL**: `https://domain.com/api/domains/change.json?id={token}`

**Team URL**: `https://domain.com/api/domains/change.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/change.json?id={token}&scope=admin`

The function renames and/or rewrites the content of an existing confirmation file. Unlike [create confirmation file](#domainsconfirm), this is a plain update, not an upsert: if the new `name` collides with another existing file of the same domain, the call fails with `bad-file`. This function does not check billing status.

The file itself (not its parent domain) is identified directly by numeric file ID, via `fid` or `file` (an alias, tried second if `fid` is missing) — unlike [create confirmation file](#domainsconfirm), there is no `uid`/`url`/`domain` parameter here.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `fid` | Numeric file ID. |
| `file` | Alternative field for the numeric file ID. Used if `fid` is not provided. |
| `name`* | New file name at the domain root. Required parameter. |
| `data` | New file content. Defaults to an empty string. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `fid` or `file` must be provided. `name` and `data` are cleaned the same way as in [create confirmation file](#domainsconfirm).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the updated file (see below). |

Error codes:

- `not-found` — file not found, no identifier provided, or insufficient scope permissions.
- `no-name` — the `name` parameter was not provided or is empty.
- `bad-file` — the file could not be updated: invalid file name or extension after cleanup, the new name collides with another file of the same domain, or a database error occurred.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric file ID. |
| `dom` | Numeric ID of the parent domain. |
| `user` | ID of the owner user; `0` for a team file. |
| `gang` | ID of the owner team; `0` for a personal file. |
| `name` | File name at the domain root (see above). |
| `data` | Raw file content (see above). |

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 8,
        "dom": 42,
        "user": 0,
        "gang": 3,
        "name": "google1234567890.html",
        "data": "google-site-verification: google1234567890.html (updated)"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "bad-file"
}
```

<a id="domainsremove"></a>
## Remove Confirmation File

**Webmaster URL**: `https://domain.com/api/domains/remove.json?id={token}`

**Team URL**: `https://domain.com/api/domains/remove.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/domains/remove.json?id={token}&scope=admin`

The function deletes a single confirmation file. Unlike [delete domain](#domainsdel), which deletes all files of a domain as a side effect, this function removes exactly one file and leaves the domain and its other files untouched. This function does not check billing status.

The file is identified directly by numeric file ID, via `fid` or `file` (an alias, tried second if `fid` is missing) — the same convention as [change confirmation file](#domainschange).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `fid` | Numeric file ID. |
| `file` | Alternative field for the numeric file ID. Used if `fid` is not provided. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

At least one of the fields `fid` or `file` must be provided.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the deleted file (see below). |

Error codes:

- `not-found` — file not found, no identifier provided, or insufficient scope permissions.
- `bad-file` — unable to delete the file: ownership condition not met or a database error occurred.

Fields of `data`:

| Field | Description |
| --- | --- |
| `id` | Numeric file ID. |
| `dom` | Numeric ID of the parent domain. |
| `user` | ID of the owner user; `0` for a team file. |
| `gang` | ID of the owner team; `0` for a personal file. |
| `name` | File name at the domain root. |

Unlike the other confirmation-file functions, this record has no `data` sub-field for file content — the deleted content is not echoed back.

Example of a successful server response:

```json
{
    "status": "ok",
    "data": {
        "id": 8,
        "dom": 42,
        "user": 0,
        "gang": 3,
        "name": "google1234567890.html"
    }
}
```

Example of an error server response:

```json
{
    "status": "error",
    "error": "not-found"
}
```
