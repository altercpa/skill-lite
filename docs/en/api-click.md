# Click API — click processing

Click API is a low-level tracker interface for registering clicks and delivering instructions to the client script. It powers flow link processing: the client script (`server.php`), installed on a site or storage, collects visitor data and sends it to one of two endpoints. The tracker analyzes the click, applies filters and TDS logic, records the click in the database, and returns a JSON instruction — what to show and how.

This API does not use the standard user token. Requests are formed by the client script automatically. This documentation is intended for developers writing custom client implementations or integrating the tracker in a non-standard way.

---

<a id="tds"></a>
## Register a click by flow code

**URL**: `https://domain.com/track-tds/{flow}`

Accepts a click by the symbolic flow identifier `{flow}`. The client script sends a POST request containing visitor data. The tracker determines the appropriate path, records the click, and returns a content display instruction.

If the `next` parameter is present in the query string, the request is processed as the **second step** (transition from pre-landing to landing) — see below for details.

The function accepts a POST body consisting of two parts:

- the contents of `$_SERVER` on the client side (server-specific fields are removed beforehand);
- the `cookie` field — the visitor's `$_COOKIE` associative array.

Key fields used by the tracker:

| Field | Description |
| --- | --- |
| `cookie` | Visitor cookie array. |
| `QUERY_STRING` | GET parameter string of the request; labels, identifiers, and control parameters are extracted from it (see below). |
| `HTTP_USER_AGENT` | Visitor's User-Agent for determining browser, OS, and device type. |
| `HTTP_HOST` | Domain where the click was recorded. |
| `HTTP_REFERER` | Referrer; used for bot detection. |
| `REMOTE_ADDR` | Visitor's IP address (also checked: `HTTP_CF_CONNECTING_IP`, `HTTP_X_REAL_IP`, `HTTP_X_CLIENT_IP`). |

Parameters from QUERY_STRING (`{flow}?param=value`):

| Parameter | Description |
| --- | --- |
| `next` | Encoded click ID. If provided, the request is processed as the second step (see below). |
| `utm_source` | UTM Source label. |
| `utm_campaign` | UTM Campaign label. |
| `utm_content` | UTM Content label. |
| `utm_term` | UTM Term label. |
| `utm_medium` | UTM Medium label. |
| `subid` | Sub ID label. |
| `customer` | Customer label (also accepted as `utm_user`). |
| `data1`..`data5` | Custom data fields (also accepted as `d1`..`d5`). |
| `fbclid`, `gclid`, `adclid`, `clid`, `adid` | Ad campaign click identifier. Saved in the `adid` field. |
| `cpc` | Click price. |
| `cur` | Click price currency. Default: `usd`. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below). |
| `uid` | Symbolic flow identifier. |
| `way` | Numeric ID of the selected path. When the rotor is enabled — a string with the history of shown paths separated by commas. |
| `method` | Content display method (see below). |
| `url` | URL for redirect or content loading. |
| `nocookie` | If `true` — the client does not set cookies (e.g., on trafficback). |
| `params` | Object with macros for substitution into URLs and content (see below). |

Error codes:

- `no-id` — flow symbolic identifier not provided.
- `bad-id` — flow identifier cannot be decoded into a numeric ID.
- `bad-flow` — no flow with this ID found in the database.
- `no-ways` — the flow has no active paths.
- `no-data` — POST body is empty or not provided.
- `db` — error writing the click to the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "uid": "ef1c",
    "way": 42,
    "method": "http302",
    "url": "https://offer.example.com/?click={click}",
    "nocookie": false,
    "params": {
        "click": "ca1337",
        "flow": "ef1c",
        "geo": "RU",
        "utm_source": "facebook",
        "utm_campaign": "summer24"
    }
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-flow"
}
```

---

<a id="domain"></a>
## Register a click by domain

**URL**: `https://domain.com/track-dom/{domain}`

Works similarly to `track-tds`, but instead of a flow code accepts the hostname `{domain}`. The tracker looks up the flow bound to that domain and passes control to the same TDS logic.

Used by the client script installed on a parked domain when it is not possible or not necessary to pass the flow code explicitly — the flow is determined automatically by the request domain.

The input format, QUERY_STRING parameters, output fields, error codes, and second-step behavior are identical to `track-tds` (see above). Additional error codes specific to this endpoint:

- `no-domain` — domain hostname not provided.
- `bad-path` — the path in `REQUEST_URI` is invalid.

---

## Second step (pre-landing to landing transition)

If the `next` parameter with an encoded click ID is present in the QUERY_STRING of the first request (`track-tds` or `track-dom`), the tracker processes the request as the second step of the "pre-landing → landing" scheme:

1. Decodes `next` into the numeric click ID.
2. Finds the click in the database and verifies it belongs to the flow.
3. Reads the second-step settings from the path: URL, site, and display method.
4. Updates the click: sets the second-step transition flag.
5. Returns a response with the second-step method and URL.

To use the second step on a pre-landing page, insert the link via the `{next}` macro (full link with domain and flow) or `{local}` (query string only). For more on path configuration, see [Flows and traffic](flows.md).

---

## Display methods (field `method`)

The `method` field in the server response tells the client script how to render the content at the `url` address. Detailed description of each method — in the [Traffic display methods](method.md) section.

| Value | Name | Description |
| --- | --- | --- |
| `http302` | 302 redirect | Standard temporary redirect via the `302 Found` header. |
| `http301` | 301 redirect | Permanent redirect via the `301 Moved Permanently` header; cached by the browser. |
| `redirect` | Meta-redirect | HTML page "Checking your browser" with a redirect after one second. |
| `jsgo` | JS redirect | Blank page with `location.replace()`, redirects after 500 ms. |
| `iframe` | IFrame | Full-screen iframe with the target site. |
| `proxy` | Curl with rebase | Content loaded via cURL; a `<base>` tag is injected into the HTML to rebase paths. |
| `curl` | Curl without rebase | Content loaded via cURL without path modification; all resource paths must be absolute. |
| `local` | Local file | Relative path to a local PHP/HTML file; macros are replaced in the content automatically. |
| `site` | Site from storage | File from the local site storage (`index.html` / `index.php`) with rebase. |
| `error` | Error page | HTTP response with the code from `url`; if `url` is empty — the code is chosen randomly by IP. |
| `blank` | Blank page | Empty `200 OK` response with no content. |
| `raw` | Raw data | Body from the `data` field, Content-Type from the `type` field. |

---

## Macros (field `params`)

The `params` field in the server response contains an object with named values that the client script substitutes for the corresponding macros in `url` and in the content of the loaded site. Full macro list — in the [Flows and traffic](flows.md#macros) section.

| Macro | Value |
| --- | --- |
| `{click}` | Symbolic click identifier. |
| `{flow}` | Symbolic flow identifier. |
| `{user}` | Tracker user identifier. |
| `{geo}` | ISO country code of the click. |
| `{host}` | Domain where the click was recorded. |
| `{query}` | GET parameter string of the request. |
| `{next}` | Full second-step link (with domain and flow code). |
| `{local}` | Relative second-step link (query string only). |
| `{direct}` | Direct second-step link without a tracker redirect. |
| `{adid}` | Ad campaign click identifier (`fbclid`, `gclid`, `adclid`, `clid`, `adid`). |
| `{data1}`..`{data5}` | Custom data fields. |
| `{subid}` | SubID label. |
| `{customer}` | Customer label. |
| `{utm_source}` | UTM Source label. |
| `{utm_campaign}` | UTM Campaign label. |
| `{utm_content}` | UTM Content label. |
| `{utm_term}` | UTM Term label. |
| `{utm_medium}` | UTM Medium label. |
| `{esub}` | Encrypted SubID label. |
| `{eutms}` | Encrypted UTM Source label. |
| `{eutmc}` | Encrypted UTM Campaign label. |
| `{eutmn}` | Encrypted UTM Content label. |
| `{eutmt}` | Encrypted UTM Term label. |
| `{eutmm}` | Encrypted UTM Medium label. |
| `{eutmu}` | Encrypted Customer label. |
| `{ed1}`..`{ed5}` | Encrypted custom data fields. |
| `{hide1}`..`{hide6}` | Hidden data fields of the flow or path. |

---

## Cookies

After a successful response, the client script sets two cookies on the visitor's domain (unless `nocookie` equals `true` in the response):

| Cookie name | Value | Lifetime |
| --- | --- | --- |
| `{uid}` | ID of the selected path (`way`) | 86 400 s (1 day) |
| `x{uid}` | Encoded click ID | 2 592 000 s (30 days) |

Here `{uid}` is the symbolic flow code. Both cookies are set with `SameSite=None; Secure` attributes and are bound to the domain from the `HTTP_HOST` header.

The `x{uid}` cookie is used by pixel endpoints to match a view or conversion to a specific click.
