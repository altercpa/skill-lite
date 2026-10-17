# Logs API Reference (`logs`)

## Overview

The `logs` app gives read access to two journals of the tracker:

- `logs/requests` — the journal of **incoming API requests** (any app and function, not only postbacks).
- `logs/postbacks` — the journal of **outgoing postbacks** the tracker sent to external URLs when a lead changed status (the "Outgoing postbacks" subsection of the "Logs" section of the interface).

Base URL: `api/logs/{func}.json?id={token}[&scope=...]`

Both functions are read-only, return entries **newest first**, paginated by `show`/`page`, as an **object keyed by the entry ID**. Both accept GET or POST (`form-urlencoded` or JSON; POST wins); `scope` is read from the query string only. There is no billing gate (no `unpaid` / `payment-required`).

A request is in the `requests` journal only if request logging is enabled on the tracker; `sys/ping` is never logged.

### Scope — whose entries you see

| `scope` | Who can use | Entries returned |
|---|---|---|
| `user` (default) | Any authenticated user | The caller's own entries |
| `gang` / `team` | Team leads only | All members of the caller's team; `user=N` narrows to one team member |
| `admin` / `sys` | Admin accounts | `user=N` → that user; else `gang=N` → all members of that team; neither → every user |

`scope` goes in the GET query string, even with a JSON body. An unavailable scope (not a team lead, not an admin, blocked account) answers `scope`; so does `scope=gang&user=N` when N is not a member of your team.

---

## logs/requests — API requests journal

```
GET|POST api/logs/requests.json?id={token}[&scope=gang|admin][&from=2026-10-01&to=2026-10-17&err=-2&show=50&page=1]
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `from` | date `YYYY-MM-DD` | Entries from 00:00:00 of this day (inclusive) |
| `to` | date `YYYY-MM-DD` | Entries until 23:59:59 of this day (inclusive) |
| `site` | int | Site ID |
| `click` | string or int | Character click ID, or a numeric click ID |
| `lead` | int | Lead ID |
| `flow` | string or int | Character flow ID, or a numeric flow ID |
| `err` | int or string | `-1` = only requests without an error; `-2` = only requests with any error; any other value = exactly this error code (e.g. `access`) |
| `page` | int | 1-based page number, default `1` |
| `show` | int | Entries per page, clamped to `1`–`1000`, default `30` |
| `user`, `gang` | int | Owner narrowing, see **Scope** |

`site`, `click`, `lead` and `flow` match requests that carried these identifiers (postbacks); for other requests these fields are empty.

**Success response:**
```json
{
  "status": "ok",
  "total": 1, "page": 1, "show": 30,
  "data": {
    "1042": {
      "id": 1042, "user": 13, "time": 1760698500,
      "app": "tracker", "func": "postback",
      "ok": true, "error": false,
      "get": { "click": "ca1337", "status": "approve" },
      "post": [], "files": [],
      "result": { "status": "ok", "stage": "approve" },
      "stage": "approve",
      "ip": "203.0.113.7", "geo": "de",
      "ua": "Mozilla/5.0 (X11; Linux x86_64) ... Chrome/130.0 Safari/537.36",
      "os": "Linux", "browser": "Chrome",
      "site": 0, "click": "ca1337", "lead": 77, "flow": "ef1c"
    }
  }
}
```

Envelope: `total` — count of all matching entries; `page`, `show` — echo of the effective values; `data` — object keyed by entry ID (an empty list `[]` when nothing matches).

Entry fields:
- `id`, `user` (the user the request was made as), `time` (UNIX timestamp), `app`, `func`.
- `ok` — `false` when the response carried an error, otherwise `true`; `error` — the error code of the response or `false`.
- `get`, `post`, `files` — request parameters and uploaded-file info; `post` and `files` are `[]` when there were none.
- `result` — the response given to the request. `stage` — the lead stage from the response of a postback request, `false` when the response has none.
- `ip`, `geo` (country code), `ua`, `os`, `browser` (names such as `Linux`, `Chrome`) — client data; each `false` when unknown.
- `site` (number, `0` if none), `click` (character ID or `false`), `lead` (number, `0` if none), `flow` (character ID or `false`).

**Errors:** `scope`

---

## logs/postbacks — Outgoing postbacks journal

```
GET|POST api/logs/postbacks.json?id={token}[&scope=gang|admin][&lead=77&event=4&from=2026-10-01]
```

**Parameters:**

| Name | Type | Description |
|---|---|---|
| `from`, `to` | date `YYYY-MM-DD` | Period, same as in `logs/requests` |
| `lead` | int | Lead ID |
| `site` | int | Site ID |
| `event` | int | Postback event: `1` Global, `2` New, `3` Hold, `4` Approve, `5` Cancel, `6` Trash |
| `flow` | string | Character flow ID (an unrecognised value is ignored) |
| `click` | string | Character click ID (an unrecognised value is ignored) |
| `page` | int | 1-based page number, default `1` |
| `show` | int | Entries per page, clamped to `1`–`1000`, default `30` |
| `user`, `gang` | int | Owner narrowing, see **Scope** |

**Success response:**
```json
{
  "status": "ok",
  "total": 1, "page": 1, "show": 30,
  "data": {
    "518": {
      "id": 518, "user": 13, "time": 1760698510,
      "flow": "ef1c", "click": "ca1337", "lead": 77, "site": 0,
      "event": 4, "eventname": "Approve",
      "url": "https://example.com/pb?click=ca1337&status=approve",
      "post": false, "header": false,
      "result": "OK", "code": 200, "ok": true,
      "timer": 0.412, "json": false
    }
  }
}
```

Entry fields:
- `id`, `user` (lead owner), `time` (UNIX timestamp the postback was sent), `flow`/`click` (character IDs or `false`), `lead`, `site` (numbers).
- `event` — event number (see the filter), `eventname` — its name in the caller's interface language (`false` if unknown).
- `url` — address the postback was sent to. `post` — request body as sent (`false` for a request without a body). `header` — extra request headers, one per line (`false` if none).
- `result` — response body of the remote server. `code` — HTTP status of the remote response (`0` — no response). `ok` — `true` when `code` is `200`–`399`.
- `timer` — request duration in seconds. `json` — `true` when the body was sent as JSON.

**Errors:** `scope`
