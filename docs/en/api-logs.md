# Logs

The logs API gives read access to two journals of the tracker: the journal of incoming API requests and the journal of outgoing postbacks. Both functions only read data. Entries are returned newest first, in pages, as an associative array keyed by the entry ID. The journal of outgoing postbacks is the one shown in the "Outgoing postbacks" subsection of the "Logs" section; the "Incoming postbacks" subsection shows the postback requests from the journal of API requests.

Whose entries you see depends on the workspace (`scope`):

- `user` (default) — your own entries.
- `team` or `gang` — entries of all members of your team; the team leader only. Pass `user` to narrow the list to one team member.
- `admin` — entries of all users; the administrator only. Pass `user` to select one user, or `gang` to select all members of one team.

<a id="logsrequests"></a>
## API Requests Log

**Webmaster URL**: `https://domain.com/api/logs/requests.json?id={token}`

**Team URL**: `https://domain.com/api/logs/requests.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/logs/requests.json?id={token}&scope=admin`

The function returns the journal of incoming API requests: any application and any function, not only postbacks. A request is in the journal only if request logging is enabled on the tracker; `ping` requests are never logged. For each request the journal keeps the parameters, the response, the result and the client data. Parameters may be passed via GET or POST (`form-urlencoded` or JSON); the `scope` parameter is read from the query string only, even if the request itself is sent by POST.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=gang` — team member ID; for `scope=admin` — user ID. |
| `gang` | For `scope=admin` — team ID: entries of all members of the team (ignored when `user` is set). |
| `from` | Start date in `YYYY-MM-DD` format, from 00:00:00 inclusive. |
| `to` | End date in `YYYY-MM-DD` format, until 23:59:59 inclusive. |
| `site` | Numeric site ID. |
| `click` | Character click ID (or numeric ID). |
| `lead` | Numeric lead ID. |
| `flow` | Character flow ID (or numeric ID). |
| `err` | Result filter (see below). |
| `page` | Page number, starting from `1`. Defaults to `1`. |
| `show` | Number of entries per page, from `1` to `1000`. Defaults to `30`. |

Values of `err`:

- `-1` — only requests that finished without an error.
- `-2` — only requests that finished with any error.
- any other value — only requests with exactly this error code (for example `access`).

The `site`, `click`, `lead` and `flow` filters match requests that carried these identifiers, for example postbacks. For other requests these fields are empty.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `total` | Total number of entries matching the filters. |
| `page` | Current page number. |
| `show` | Number of entries per page. |
| `data` | Entries of the page keyed by entry ID, newest first; an empty list when nothing is found. Composition detailed below. |

Error codes:

- `scope` — the requested workspace is not available to you (not a team leader, not an administrator, the account is blocked, or, with `scope=gang`, the `user` parameter names someone who is not a member of your team).

Each entry of the `data` field is an associative array:

| Field | Description |
| --- | --- |
| `id` | Entry numeric ID. |
| `user` | ID of the user the request was made as. |
| `time` | Request time in UNIX Timestamp format. |
| `app` | Called application. |
| `func` | Called function. |
| `ok` | `true` if the response contained no error, `false` otherwise. |
| `error` | Error code of the response, `false` if there was none. |
| `get` | GET parameters of the request as an associative array. |
| `post` | POST parameters of the request as an associative array; an empty list if there were none. |
| `files` | Information about the uploaded files; an empty list if there were none. |
| `result` | Response to the request. |
| `stage` | Lead stage from the response of a postback request, `false` if the response has none. |
| `ip` | Client IP address, `false` if unknown. |
| `geo` | Two-letter ISO code of the client country, `false` if unknown. |
| `ua` | Client User Agent, `false` if unknown. |
| `os` | Client operating system name, `false` if unknown. |
| `browser` | Client browser name, `false` if unknown. |
| `site` | Numeric site ID, `0` if the request had none. |
| `click` | Character click ID, `false` if the request had none. |
| `lead` | Numeric lead ID, `0` if the request had none. |
| `flow` | Character flow ID, `false` if the request had none. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 1,
    "page": 1,
    "show": 30,
    "data": {
        "1042": {
            "id": 1042,
            "user": 13,
            "time": 1760698500,
            "app": "tracker",
            "func": "postback",
            "ok": true,
            "error": false,
            "get": {
                "click": "ca1337",
                "status": "approve"
            },
            "post": [],
            "files": [],
            "result": {
                "status": "ok",
                "stage": "approve"
            },
            "stage": "approve",
            "ip": "203.0.113.7",
            "geo": "de",
            "ua": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0 Safari/537.36",
            "os": "Linux",
            "browser": "Chrome",
            "site": 0,
            "click": "ca1337",
            "lead": 77,
            "flow": "ef1c"
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

<a id="logspostbacks"></a>
## Outgoing Postbacks Log

**Webmaster URL**: `https://domain.com/api/logs/postbacks.json?id={token}`

**Team URL**: `https://domain.com/api/logs/postbacks.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/logs/postbacks.json?id={token}&scope=admin`

The function returns the journal of outgoing postbacks — requests that the tracker sent to external addresses when the status of a lead changed. For each postback the journal keeps the address, the request body, the response of the remote server and the request time. Parameters may be passed via GET or POST (`form-urlencoded` or JSON); the `scope` parameter is read from the query string only, even if the request itself is sent by POST.

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `scope` | Workspace: `user` (default), `gang`/`team`, `admin`/`sys`. |
| `user` | For `scope=gang` — team member ID; for `scope=admin` — user ID. |
| `gang` | For `scope=admin` — team ID: entries of all members of the team (ignored when `user` is set). |
| `from` | Start date in `YYYY-MM-DD` format, from 00:00:00 inclusive. |
| `to` | End date in `YYYY-MM-DD` format, until 23:59:59 inclusive. |
| `lead` | Numeric lead ID. |
| `site` | Numeric site ID. |
| `event` | Postback event (see below) |
| `flow` | Character flow ID. |
| `click` | Character click ID. |
| `page` | Page number, starting from `1`. Defaults to `1`. |
| `show` | Number of entries per page, from `1` to `1000`. Defaults to `30`. |

Values of `event`:

- `1` — Global.
- `2` — New.
- `3` — Hold.
- `4` — Approve.
- `5` — Cancel.
- `6` — Trash.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `total` | Total number of entries matching the filters. |
| `page` | Current page number. |
| `show` | Number of entries per page. |
| `data` | Entries of the page keyed by entry ID, newest first; an empty list when nothing is found. Composition detailed below. |

Error codes:

- `scope` — the requested workspace is not available to you (not a team leader, not an administrator, the account is blocked, or, with `scope=gang`, the `user` parameter names someone who is not a member of your team).

Each entry of the `data` field is an associative array:

| Field | Description |
| --- | --- |
| `id` | Entry numeric ID. |
| `user` | ID of the lead owner. |
| `time` | Time the postback was sent in UNIX Timestamp format. |
| `flow` | Character flow ID, `false` if there is none. |
| `click` | Character click ID, `false` if there is none. |
| `lead` | Numeric lead ID. |
| `site` | Numeric site ID. |
| `event` | Postback event (see above) |
| `eventname` | Event name in your interface language, `false` if unknown. |
| `url` | Address the postback was sent to. |
| `post` | Request body as it was sent, `false` for a request without a body. |
| `header` | Additional request headers, one per line; `false` if there were none. |
| `result` | Response of the remote server. |
| `code` | HTTP status of the response of the remote server; `0` if there was no response. |
| `ok` | `true` if the status is from `200` to `399`, `false` otherwise. |
| `timer` | Request duration in seconds. |
| `json` | `true` if the request body was sent as JSON. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "total": 1,
    "page": 1,
    "show": 30,
    "data": {
        "518": {
            "id": 518,
            "user": 13,
            "time": 1760698510,
            "flow": "ef1c",
            "click": "ca1337",
            "lead": 77,
            "site": 0,
            "event": 4,
            "eventname": "Approve",
            "url": "https://example.com/pb?click=ca1337&status=approve",
            "post": false,
            "header": false,
            "result": "OK",
            "code": 200,
            "ok": true,
            "timer": 0.412,
            "json": false
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
