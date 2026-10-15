# Site Storage

The site storage API allows you to manage local sites in the tracker: upload ZIP archives of landing pages and pre-landing pages, mirror sites by URL, modify their properties, delete them, and download archives. Sites in the storage are used in [flow paths](api-flows.md) as target pages and in trafficback. Each site receives a short obfuscated path in the format `{2hex}/{enc-id}` and is accessible via the storage domain.

The storage can be system-level (shared across the tracker), user-level, or team-level. Operations on sites are always directed to the storage that is active for the request context: team-level takes priority over user-level, user-level takes priority over system-level.

<a id="siteslist"></a>
## Sites List

**Webmaster URL**: `https://domain.com/api/sites/list.json?id={token}`

**Team URL**: `https://domain.com/api/sites/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/sites/list.json?id={token}&scope=admin`

The function returns a list of sites with filtering and pagination support. All filters are combined using AND. When `scope=admin`, billing validation is not performed.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=gang` or `scope=admin`: the ID of a specific team member or user to filter by. |
| `gang` | For `scope=admin`: team ID to filter by. |
| `uid` | Exact match by site ID. |
| `glue` | Filter by visibility marker (for team sites, see below). |
| `offer` | Filter by offer ID. |
| `search` | Search by site name. |
| `page` | Results page (≥ 1). Default: 1. |
| `show` | Page size. Default: 0 (return all records without limit). |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `total` | Total number of records matching the filters. When `show > 0`, computed by a separate query. |
| `page` | Current page. |
| `show` | Page size. |
| `data` | Dictionary of sites, keyed by site ID (see below). |

Error codes:

- `scope` — workspace permission check failed (invalid scope, missing team lead or administrator rights).
- `unpaid` — tracker access has expired or is not paid (only for `scope=user` and `scope=gang`).

Record structure in `data`:

| Field | Description |
| --- | --- |
| `uid` | Numeric site identifier. |
| `user` | Owner user ID for a personal site; `0` for a gang site. |
| `gang` | Team ID; `0` for a personal site. |
| `glue` | Visibility marker for a gang site (see below); `0` for a personal site. |
| `name` | Site name. |
| `path` | Site path in the storage in the format `{2hex}/{enc-id}`. |

Visibility marker (`glue`) values for gang sites:

- `0` — site is hidden from the team (visible to the team administrator only).
- `-1` — public site for the entire team.
- `>0` — site is assigned to a specific team member (visible only to that member and team administrators).

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 2,
    "page": 1,
    "show": 0,
    "data": {
        "42": {
            "uid": 42,
            "user": 7,
            "gang": 0,
            "glue": 0,
            "name": "Landing about insurance",
            "path": "ab/cm6gh1"
        },
        "43": {
            "uid": 43,
            "user": 0,
            "gang": 3,
            "glue": -1,
            "name": "Team public prelanding",
            "path": "f2/dkp8n3"
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

<a id="sitesadd"></a>
## Upload Site

**Webmaster URL**: `https://domain.com/api/sites/add.json?id={token}`

**Team URL**: `https://domain.com/api/sites/add.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/sites/add.json?id={token}&scope=admin`

The function creates a new site in the storage. Depending on the parameters provided, one of three actions is performed on the storage: unpacking a ZIP archive, mirroring a site by URL via httrack, or creating an empty folder. The database operation and the storage operation are independent: if the storage is unavailable, the site record will still be created but files may not appear on disk.

Mirroring (`url`) is performed in the background without notifications. Technical httrack limits: up to 100 MB of data and up to 10 minutes of execution time. If both parameters are provided (`archive` and `url`), `archive` takes priority.

The request must be sent via POST with `Content-Type: multipart/form-data` when uploading a file, or as a standard GET/POST without a file.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `name`* | Site name. Required. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `glue` | Visibility marker for a gang site (see values in [sites list](#siteslist)). Applicable when `scope=gang` or `scope=admin` with `gang` specified. Default: 0 (hidden). |
| `user` | For `scope=admin` without `gang`: the user ID of the new site owner. |
| `gang` | For `scope=admin`: the team ID of the new site owner. |
| `archive` | Site ZIP archive (`archive` field in `multipart/form-data`). After upload, the storage unpacks the archive into the site folder. |
| `url` | Site URL for mirroring via httrack. Ignored if `archive` is provided. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the created site (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `unpaid` — tracker access has expired or is not paid (only for `scope=user` and `scope=gang`).
- `user-required` — for `scope=admin`: neither `user` nor `gang` was provided.
- `wrong-glue` — the provided `glue` value (`>0`) does not belong to the selected team.
- `add` — failed to create the site: empty name or database error.

`data` field structure:

| Field | Description |
| --- | --- |
| `id` | Numeric identifier of the created site. |
| `user` | Owner user ID; `0` for a gang site. |
| `gang` | Team ID; `0` for a personal site. |
| `glue` | Visibility marker (see [sites list](#siteslist)). |
| `name` | Site name. |
| `path` | Site path in the storage in the format `{2hex}/{enc-id}`. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "user": 7,
        "gang": 0,
        "glue": 0,
        "name": "Landing about insurance",
        "path": "ab/cm6gh1"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "add"
}
```

<a id="sitesedit"></a>
## Edit Site

**Webmaster URL**: `https://domain.com/api/sites/edit.json?id={token}`

**Team URL**: `https://domain.com/api/sites/edit.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/sites/edit.json?id={token}&scope=admin`

The function modifies the name and/or visibility marker of a site, and also allows overwriting its content by uploading a new ZIP archive or providing a new URL for mirroring. If neither `archive` nor `url` is provided, no request is sent to the storage — only the database fields are updated. When `archive` or `url` is provided, the site content in the storage is completely overwritten.

The database and storage operations are independent: if the storage is unavailable, the database changes will still be saved.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid`* | Site identifier. Required. Also accepted in the `sid` field. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `name` | New site name. If not provided, the name remains unchanged. |
| `glue` | New visibility marker for a gang site (see values in [sites list](#siteslist)). Applicable when `scope=gang` or `scope=admin`. Only takes effect when the parameter is actually present in the request; if omitted, the current marker is preserved. |
| `archive` | New site ZIP archive (`archive` field in `multipart/form-data`). Overwrites the content. |
| `url` | New URL for mirroring. Ignored if `archive` is provided. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Site data after the update (structure as in [upload site](#sitesadd)). |

Error codes:

- `scope` — workspace permission check failed.
- `unpaid` — tracker access has expired or is not paid (only for `scope=user` and `scope=gang`).
- `no-id` — site identifier (`uid`/`sid`) was not provided.
- `bad-id` — site with the specified identifier was not found.
- `auth` — insufficient permissions: for `scope=user`, the site belongs to another user or team; for `scope=gang`, the site belongs to a different team.
- `wrong-glue` — the provided `glue` value (`>0`) does not belong to the selected team.
- `edit` — failed to save changes: a database error, or the site has no owner.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "user": 7,
        "gang": 0,
        "glue": 0,
        "name": "Updated landing",
        "path": "ab/cm6gh1"
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

<a id="sitesdel"></a>
## Delete Site

**Webmaster URL**: `https://domain.com/api/sites/del.json?id={token}`

**Team URL**: `https://domain.com/api/sites/del.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/sites/del.json?id={token}&scope=admin`

The function removes the site record from the database and deletes the site folder from the storage. The operations are independent: if the storage is unavailable, the database record will still be deleted but the files on disk will remain. After a site is deleted, flow paths associated with it retain references to the non-existent identifier — it is recommended to update the relevant [flows](api-flows.md) in advance.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid`* | Site identifier. Required. Also accepted in the `sid` field. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `data` | Data of the deleted site (structure as in [upload site](#sitesadd)). |

Error codes:

- `scope` — workspace permission check failed.
- `unpaid` — tracker access has expired or is not paid (only for `scope=user` and `scope=gang`).
- `no-id` — site identifier (`uid`/`sid`) was not provided.
- `bad-id` — site with the specified identifier was not found.
- `auth` — insufficient permissions: for `scope=user`, the site belongs to another user or team; for `scope=gang`, the site belongs to a different team.
- `del` — failed to delete the site: database error or no permissions on the site.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 42,
        "user": 7,
        "gang": 0,
        "glue": 0,
        "name": "Landing about insurance",
        "path": "ab/cm6gh1"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "auth"
}
```

<a id="sitesdownload"></a>
## Download Site

**Webmaster URL**: `https://domain.com/api/sites/download.json?id={token}`

**Team URL**: `https://domain.com/api/sites/download.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/sites/download.json?id={token}&scope=admin`

The function returns a ZIP archive containing all files of the site. On success, the response contains a binary data stream rather than JSON. On failure, the response is returned in the standard JSON format.

When `scope=admin`, site ownership is not verified — an administrator can download any site without specifying `user` or `gang`.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `uid`* | Site identifier. Required. Also accepted in the `sid` field. |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |

**Successful response** — HTTP 200 with headers:

```
Content-Type: application/octet-stream
Content-Disposition: attachment; filename=site.zip
```

The response body is a binary ZIP archive of the site files.

**Error response** — JSON:

| Field | Description |
| --- | --- |
| `status` | Operation result: `error`. |
| `error` | Error code (see below). |

Error codes:

- `scope` — workspace permission check failed.
- `unpaid` — tracker access has expired or is not paid (only for `scope=user` and `scope=gang`).
- `no-id` — site identifier (`uid`/`sid`) was not provided.
- `bad-id` — site with the specified identifier was not found.
- `auth` — insufficient permissions: for `scope=user`, the site belongs to another user or team; for `scope=gang`, the site belongs to a different team. Not returned for `scope=admin`.
- `download` — failed to retrieve the archive from the storage (storage unavailable or archive not created).

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "download"
}
```
