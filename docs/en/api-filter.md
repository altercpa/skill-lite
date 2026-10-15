# Filtering

Filtering functions allow you to check visits to your sites as quickly as possible. They are placed in a separate interface, which has the minimum response time and maximum performance.

The same section contains the functions for managing filter campaigns, the visit log and the statistics of checked visits, and for adding entries to blacklists. The blacklists themselves (creating lists, editing, importing and exporting entries) are managed with the separate [blacklist API](api-blacklist.md).

<a id="check"></a>
## Check the visit

URL: `https://domain.com/fltr/{user}-{key}-{id}`

In the function address, instead of `{user}` and `{key}`, specify the API ID and key from the [profile](/profile), instead of `{id}` specify the identifier of the filter site you added. You can get the specific URL by clicking on the "Manual" button of the corresponding site in your [your personal account](/filter).

To check your visit, you need to send a POST request to the above URL. Pass the fields of your array `$_SERVER` as the body of the request - we will parse them and show the response in JSON format. A successful response includes the `status`, `action` and `url` fields. The `status` field always contains the value `ok`. The `action` field of the response will contain `allow` or `deny` depending on the verification results, the `url` field will contain the corresponding URL. In the error response, the `status` field will contain `error`, the error code will be indicated in the `error` field. The response always has HTTP status 200 and is always in JSON format; the URL has no format extension. The decision is also duplicated in the `X-Action` response header: `allow` or `deny` (in an error response it is always `deny`).

The POST part of the request must be in `x-www-form-urlencoded` format, JSON format is not supported: a request with an empty body (including a JSON body) returns the `no-data` error. We parse the following fields from the `$_SERVER` array:

HTTP_USER_AGENT, HTTP_ACCEPT, HTTP_ACCEPT_CHARSET, HTTP_ACCEPT_ENCODING, HTTP_ACCEPT_LANGUAGE, HTTP_AUTHORIZATION, HTTP_CACHE_CONTROL, HTTP_CONNECTION, HTTP_CONTENT_DISPOSITION, HTTP_DATE, HTTP_EXPECT, HTTP_FROM, HTTP_HOST, HTTP_IF_MATCH, HTTP_IF_MODIFIED_SINCE, HTTP_IF_NONE_MATCH, HTTP_IF_RANGE, HTTP_IF_UNMODIFIED_SINCE, HTTP_MAX_FORWARDS, HTTP_PRAGMA, HTTP_PROXY_AUTHORIZATION, HTTP_RANGE, HTTP_REFERER, HTTP_TE, HTTP_TRAILER, HTTP_TRANSFER_ENCODING, HTTP_UPGRADE, HTTP_VIA, HTTP_X_FORWARDED_FOR, REMOTE_ADDR, REMOTE_PORT, REQUEST_METHOD, REQUEST_URI, QUERY_STRING, HTTP_CF_CONNECTING_IP, HTTP_CLIENT_IP, HTTP_X_REAL_IP, HTTP_X_PURPOSE, HTTP_X_FB_HTTP_ENGINE, HTTP_CF_CONNECTING_IPV6, HTTP_X_FRAME_OPTIONS, HTTP_X_WAP_PROFILE, HTTP_PRIORITY, HTTP_SEC_FETCH_MODE, HTTP_SEC_FETCH_SITE, HTTP_SEC_FETCH_DEST, HTTP_SEC_FETCH_USER, HTTP_SEC_CH_UA, HTTP_SEC_CH_UA_MOBILE, HTTP_SEC_CH_UA_PLATFORM, HTTP_DNT, HTTP_ORIGIN, HTTP_UPGRADE_INSECURE_REQUESTS, HTTP_SAVE_DATA, HTTP_X_FORWARDED_HOST, HTTP_X_FORWARDED_PROTO, REQUEST_SCHEME

The result of executing the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of successful execution, `error` in case of an error |
| `action` | **Action** required based on verification results: `allow` for legitimate visit (show target site) or `deny` for suspicious visit (show dummy site) |
| `result` | The real **result** of testing traffic, which does not depend on the enabled training mode: `allow` for legitimate visit (show target site) or `deny` for suspicious visit (show dummy site). In training mode, the value of `result` may differ from the value of `action`. |
| `url` | The site URL for the request depends on the results of the check. |
| `target` | The URL of the target site, if available, is always shown regardless of the check results. |
| `dummy` | The URL of the dummy site, if available, is always shown regardless of the check results. |
| `type` | Type of the site to be shown: `black` for the target site, `white` for the dummy site. Always matches `action` (`allow` is `black`, `deny` is `white`); the file produced by [Download filtering file](#filtermake) is guided by this field. |
| `ip` | Visitor IP address determined from the request headers (`false` if it could not be determined). |
| `geo` | Two-letter country code of the visitor in lower case (`zz` if the country is not determined). |
| `reason` | Check result code (see [check result codes](#filterreasons) below): `0` if the visit passed all checks. In training mode it is the real check result, like `result`. For paused and disabled campaigns it is `0`. |
| `error` | Error ID: `access-denied` if there is no access to the verification system (the user is banned, the key is wrong or the campaign belongs to another user), `bad-request` if the verification URL is incorrectly specified, `bad-site` if specified site not found, `no-data` if the request has an empty body. |

In `url`, `target` and `dummy`, the macros `{geo}` (country code in lower case), `{geoup}` (country code in upper case) and `{ip}` (visitor IP address) are replaced with their values.

**An example of the implementation of sending a request:**

```php
$curl = curl_init( 'https://domain.com/fltr/123-abcd-45' );
curl_setopt( $curl, CURLOPT_POST, true );
curl_setopt( $curl, CURLOPT_POSTFIELDS, $_SERVER );
curl_setopt( $curl, CURLOPT_RETURNTRANSFER, true );
$result = curl_exec( $curl );
curl_close( $curl );
$result = $result ? json_decode( $result, true ) : [];
if ( isset( $result['action'] ) && $result['action'] == 'allow' ) {
    include( 'target-index.php' );
} else include( 'dummy-index.php' );
```

**Example of a successful server response:**

```json
{
   "status": "ok",
   "result": "allow",
   "action": "allow",
   "type": "black",
   "url": "http://black.site/url",
   "ip": "91.203.45.12",
   "geo": "ru",
   "reason": 0,
   "target": "http://black.site/url",
   "dummy": "http://white.site/"
}
```

**Example of an error server response:**

```json
{
   "status": "error",
   "error": "access-denied"
}
```

<a id="filterreasons"></a>
### Check result codes and order of checks

The `reason` field of the response and the `result` field of the [visit log](#filterlog) contain one of the following codes:

- `0` - the visit passed all checks (the target site is shown) or the visitor IP address is in the whitelist `wl`.
- `1` - request to an invalid path: service files such as favicon, source maps, manifests, `ads.txt`.
- `2` - traffic from a wrong device: the device type (`mobile`), the operating system (`os`) or the browser (`browser`) is not allowed.
- `3` - request from a blocked country (the country is not in `geo` or is in `nogeo`).
- `4` - IP address is blacklisted.
- `5` - IP network is blacklisted.
- `6` - IPv6 address is blacklisted.
- `7` - one of the request headers is blacklisted, a header required by a white rule is missing or does not match it, or the User-Agent is empty.
- `8` - bot or site preview: preview requests of browsers and messengers, translator proxies, referrers of spy services, no IP address, a bot or an unrecognized User-Agent.
- `9` - traffic via a wrong protocol (`proto`).
- `10` - wrong visitor language (`langs`, `nolangs`, `lang`).
- `11` - first trash visits (the `trash` counter, it decreases by one with each such visit) or a request to a probing path (configuration files, `robots.txt`, `/.env` and so on).
- `12` - URL is blacklisted (no match with `uwl` or a match with `ubl`).
- `13` - no ad network click ID (`adid`).
- `14` - visit outside the campaign schedule (`wd1`..`wd7`, `fh`, `fm`, `th`, `tm`, `tz`).

Checks are performed in the following order, the first one that fires decides the result: IP whitelist `wl` (`0`) → previews, translators and missing IP (`8`) → invalid and probing paths (`1`, `11`) → User-Agent (`7`, `8`) → device, operating system and browser (`2`) → protocol (`9`) → country and language (`3`, `10`) → ad click ID (`13`) → URL lists (`12`) → IP address and network blacklists (`4`, `5`, `6`) → header rules (`7`) → first trash visits (`11`) → schedule (`14`).

Header rules of blacklists are of two kinds: a black rule blocks the visit if the header matches the value, a white rule blocks the visit if the header is missing or does not match the value (see [Block IP address or header](#filterblock)).

Operating modes of the campaign (`mode`):

- In the paused mode (`1`) the answer is always `deny`, in the disabled mode (`2`) it is always `allow`; no checks are performed.
- In training modes (`3`, `4`) all checks are performed, but `action`, `type` and `url` are forced to the dummy (`3`) or the target (`4`) site; `result` and `reason` contain the real check result.

Every checked visit is recorded in the visit log of the campaign (see [Visit log](#filterlog)), unless the visit is excluded by the log mode of the campaign (`log`, see [Campaign setup](#filterset)). Visits of paused and disabled campaigns are not recorded. The campaign counters (`stats` in [Campaigns list](#filterlist)) are incremented for each visit according to the real check result: `black` for allowed visits, `white` for blocked ones (in the paused mode every visit counts as blocked, in the disabled mode as allowed).

<a id="filterscopes"></a>
## Scopes and campaign owners

The functions for managing campaigns and for viewing the visit log work in one of three spaces selected by the `scope` parameter (see [API interface](api.md)). These are [Campaign setup](#filterset), [Campaigns list](#filterlist), [Get campaign info](#filterget), [Delete campaign](#filterdel), [Download filtering file](#filtermake), [Block IP address or header](#filterblock), [Visit log](#filterlog) and [Visit statistics](#filterstats). The space defines whose campaigns the function can see and change. The `scope` parameter is read from the query string only, even if the request itself is sent by POST. The visit check and the [postback](#filterpostback) do not use scopes.

| Scope | Description |
| --- | --- |
| `scope=user` | Default. Your own campaigns. Available to every active user. |
| `scope=gang` | Alias: `team`. The campaigns of all members of your team. Available only to team leaders. The optional `user` parameter limits the function to one member of your team; if the user is not a member of your team, the `scope` error is returned. |
| `scope=admin` | Aliases: `sys`, `system`, `all`. Administrators only. Without `gang` and `user` the function works with the campaigns of all users. The `user` parameter limits it to the campaigns of one user, `gang` to the campaigns of the members of one team; if both are passed, a campaign must satisfy both conditions. |

Additional parameters of the scope selection (read from the query string or from the POST body):

| Field | Description |
| --- | --- |
| `gang` | Admin scope only: team ID. |
| `user` | Team scope: ID of a team member. Admin scope: user ID. In both scopes the parameter is required when a new campaign is created with the [Campaign setup](#filterset) function: the campaign is created for this user. |
| `uu` | Administrator keys only: ID of a user on whose behalf the request is executed. For example, `scope=user&uu=123` works with the campaigns of user 123. |

Notes:

- If the scope cannot be resolved (you are not a team leader, you have no administrator rights, the `user` is not a member of your team, the account is banned or inactive), the `scope` error is returned.
- A campaign outside the current scope is treated as non-existent: for example, `no-site` is returned.
- The `api` links (and the postback links of [Get campaign info](#filterget) and the file of [Download filtering file](#filtermake)) always contain the API key of the campaign **owner**, not the key of the caller. In the team and admin scopes they reveal the keys of other users, so keep such responses and files private.
- The blacklists that are attached to campaigns and receive new entries are managed with the separate [blacklist API](api-blacklist.md).

<a id="filterset"></a>
## Campaign setup

**Webmaster URL**: `https://domain.com/api/filter/set.json?id={token}&domain={host}`

**Team URL**: `https://domain.com/api/filter/set.json?id={token}&domain={host}&scope=gang&user={user}`

**Admin URL**: `https://domain.com/api/filter/set.json?id={token}&domain={host}&scope=admin&user={user}`

The function modifies the filtering campaign parameters or creates a new campaign. If a campaign identifier is passed in the `uid` field, the function will modify the existing campaign by this identifier. If the identifier is not passed, but the campaign domain is provided in the `domain` field, the function will attempt to find a campaign with the specified domain, and if it does not exist, it will create a new one. A new campaign is configured with the other passed parameters in the same call. Parameters that are not passed keep their values. If both `uid` and `domain` are passed and the `domain` differs from the stored name, the campaign is renamed.

The function works in the [scope](#filterscopes) of the request. In the team and admin scopes, the `user` parameter (the owner of the new campaign) is required to create a campaign.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `uid` | Identifier of the filtering campaign (recommended). |
| `domain` | Domain or name of the filtering campaign (use unique names). The name is used only to find the campaign: the domain of the visit is not compared with it. |
| `mode` | Campaign operation mode (see below) |
| `geo` | List of two-letter ISO country codes where the filter works. Provided as an array or a string separated by commas, for example: `ru,kz,by`. If the list contains `ww` or `zz`, the whole list is saved as empty. |
| `mblack` | Operation mode of the target site (see below) |
| `mwhite` | Operation mode of the dummy site (see below) |
| `black` | Target site: link or file name. |
| `white` | Dummy site: link or file name. |
| `trash` | Number of trash visits, integer, default is 0. |
| `proto` | IP protocol version (see below), default is 0. |
| `mobile` | Type of allowed traffic (see below), default is 0. |
| `os` | Allowed operating systems (see below), array or list of IDs separated by commas |
| `browser` | Allowed browsers (see below), array or list of IDs separated by commas  |
| `ipmain` | Check only the main IP address of the request against the blacklist, 0 or 1 (default). |
| `lang` | Check the visitor's language against the list of allowed countries, 0 or 1 (default). |
| `adid` | Check the ad network click identifier, 0 (default) or 1. |
| `list` | List of IDs of traffic filters (blacklists), array or list separated by commas, empty by default. Only the lists available to the campaign owner are accepted (system, personal and team lists, see [blacklist API](api-blacklist.md)); other IDs are silently dropped. If the list is empty, the campaign uses all active default lists. |
| `langs` | List of two-letter ISO codes of allowed languages, array or list separated by commas, empty by default. |
| `nogeo` | List of two-letter ISO codes of countries from which access is prohibited. Specify if the list of allowed countries is empty. If the list contains `ww` or `zz`, the whole list is saved as empty. |
| `uwl` | Whitelist of words in URL, specified as text one per line. Words separated by commas inside one line must all be present in the URL. |
| `ubl` | Blacklist of words in URL, specified as text one per line, in the same format as `uwl`. |
| `wl` | Whitelist of IP addresses, specified as text one per line, can contain IP addresses and subnets. |
| `log` | Visit log mode (see below). |
| `wd1`..`wd7` | Weekdays on which visits are allowed, 0 or 1: `wd1` is Monday, `wd7` is Sunday. Visits on other days are blocked (reason `14`). |
| `fh`, `fm` | Start of the daily time window: hour (0-23) and minute (0-59). |
| `th`, `tm` | End of the daily time window: hour (0-23) and minute (0-59). If the start is later than the end, the window passes through midnight. |
| `tz` | Time zone of the schedule in the IANA format, for example `Europe/Berlin`. An unknown identifier is saved as an empty value (the server time zone is used). |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). In the team and admin scopes, `user` is required to create a campaign. |

Campaign operation modes (`mode`):

- `0` - Automatic: filter enabled (default)
- `1` - Pause: always show the dummy site
- `2` - Disabled: always show the target site
- `3` - Training: always show the dummy site
- `4` - Training: always show the target site

Operation modes of the target site (`mblack`) and dummy site (`mwhite`):

- `0` - Redirect by link (302)
- `5` - Redirect and glue domain (301)
- `6` - Redirect via separate page (meta)
- `9` - Redirect via script (JS)
- `1` - Open file with this name
- `2` - Show this link in iframe
- `7` - Show content by link (curl)
- `3` - Show content by link without changes
- `4` - Embed to WordPress
- `8` - Show blank page

IP protocol versions (`proto`):

- `0` - Accept traffic of any protocol (default)
- `1` - Accept only IPv4 traffic
- `2` - Accept only IPv6 traffic

Types of allowed traffic (`mobile`):

- `0` - Any traffic (default)
- `1` - Mobile traffic only
- `2` - Desktop traffic only
- `3` - Mobile traffic: iOS only
- `4` - Mobile traffic: excluding iOS

Allowed operating systems (`os`):

- `5` - Android
- `8` - BlackBerry
- `3` - Linux
- `2` - MacOS
- `7` - Symbian
- `1` - Windows
- `6` - Windows Mobile
- `4` - iOS

Allowed browsers (`browser`):

- `1` - Chrome
- `6` - Edge
- `7` - Facebook
- `2` - Firefox
- `8` - Instagram
- `3` - MSIE
- `5` - Opera
- `4` - Safari (detected on iOS and macOS only)
- `9` - Tik-Tok
- `10` - WebView (detected on iOS and macOS only)

Visit log modes (`log`):

- `0` - Log all visits
- `1` - Log only target visits (allowed)
- `2` - Log only dummy visits (blocked)
- `3` - Do not log visits

Campaign schedule (`wd1`..`wd7`, `fh`, `fm`, `th`, `tm`, `tz`): the visit is allowed only on the selected weekdays and within the daily time window, calculated in the time zone `tz`. The time window is applied if the hour or the minute of its start or end is non-zero; if all four values are zero, there is no time limit. A visit outside the schedule is blocked with reason `14`.

The result of the function execution is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of successful execution, `error` in case of an error. |
| `error` | Error code for status `error` (see below) |
| `id` | Numeric identifier of the campaign. |
| `api` | Link to the API function for checking visits by this campaign. |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-site` - the campaign with the specified `uid` was not found in the current scope.
- `no-domain` - neither the campaign identifier nor the domain or campaign name was specified.
- `user-required` - a new campaign has to be created, but the `user` parameter is not passed (team and admin scopes).
- `db` - database system error occurred.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "id": 4567,
    "api": "https://domain.com/fltr/123-654b807372f614e64b9-4567"
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-domain"
}
```

<a id="filterlist"></a>
## Campaigns list

**Webmaster URL**: `https://domain.com/api/filter/list.json?id={token}`

**Team URL**: `https://domain.com/api/filter/list.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/list.json?id={token}&scope=admin`

The function returns a list of campaigns created in the traffic filtering section. In the team scope it returns the campaigns of all members of the team, in the admin scope the campaigns of all users (see [Scopes](#filterscopes)). The function does not have its own filtering parameters; the list is limited only by the scope parameters:

| Field | Description |
| --- | --- |
| `user` | Team scope: return the campaigns of this team member only. Admin scope: return the campaigns of this user only. |
| `gang` | Admin scope only: return the campaigns of the members of this team only. |

The result of the function is a list of associative arrays with the following fields (an empty list is `[]`):

| Field | Description |
| --- | --- |
| `id` | Filter campaign identifier. |
| `user` | Identifier of the campaign owner. |
| `api` | API link for checking visits for this campaign. It contains the API key of the campaign owner. |
| `domain` | Filter campaign name. |
| `mode` | Campaign mode (see [campaign settings](#filterset)). |
| `stats` | Filter campaign statistics. Associative array (see below) |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)). The error response is an associative array with `status` and `error` fields instead of a list.

Filter campaign statistics fields (the counters are the campaign totals, see [Visit statistics](#filterstats) for figures from the visit log):

- `click` - total number of visits.
- `black` - number of target visits.
- `white` - number of blocked visits.
- `leads` - total number of leads.
- `approve` - number of approved leads.
- `cancel` - number of rejected leads.
- `cash` - total earnings, rounded to two decimals.
- `cr` - conversion rate for target visits, percent, two decimals.
- `epc` - earnings per target visit, two decimals. The earnings are rounded down to a whole number before the division.
- `appr` - approval rate, percent, two decimals.

**Example of a successful server response:**

```json
[
    {
        "id": 4567,
        "user": 123,
        "api": "https://domain.com/fltr/123-654b807372f614e64b9-4567",
        "domain": "pretty-domain.com",
        "mode": 0,
        "stats": {
            "click": 180,
            "black": 123,
            "white": 57,
            "leads": 11,
            "approve": 6,
            "cancel": 1,
            "cash": 387.44,
            "cr": 8.94,
            "epc": 3.15,
            "appr": 54.55
        }
    },
    ...
]
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "scope"
}
```

<a id="filterget"></a>
## Get campaign info

**Webmaster URL**: `https://domain.com/api/filter/get.json?id={token}&domain={host}`

**Team URL**: `https://domain.com/api/filter/get.json?id={token}&domain={host}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/get.json?id={token}&domain={host}&scope=admin`

The function returns filtering campaign data. If a campaign identifier is passed in the `uid` field, the function will retrieve the campaign through this identifier. If a campaign domain is passed in the `domain` field, the function will attempt to find a campaign with the specified domain. The domain is compared with the campaign name as an exact string (a URL is not parsed); if there are several campaigns with the same name, the one with the highest identifier is returned. The campaign is searched for in the [scope](#filterscopes) of the request.

The function can take the following parameters as input:

| Field | Description |
| --- | --- |
| `uid` | Filter campaign identifier (recommended). If it is passed, `domain` is ignored. |
| `domain` | Domain or name of the filtering campaign (use unique names). |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

The result of the function is an associative array with the following fields:

| Field | Description |
| --- | --- |
| `id` | Identifier of the filtering campaign. |
| `user` | Identifier of the campaign owner. |
| `api` | API link for checking visits by this campaign. It contains the API key of the campaign owner. |
| `domain` | Name of the filtering campaign. |
| `mode` | Campaign operation mode (see [campaign settings](#filterset)). |
| `geo` | List of two-letter ISO country codes where the filter is active (array). |
| `mblack` | Target site operation mode (see [campaign settings](#filterset)). |
| `black` | Target site: link or file name. |
| `mwhite` | Dummy site operation mode (see [campaign settings](#filterset)). |
| `white` | Dummy site: link or file name. |
| `trash` | Number of trash visits. |
| `proto` | IP protocol version (see [campaign settings](#filterset)). |
| `mobile` | Type of acceptable traffic (see [campaign settings](#filterset)). |
| `os` | Acceptable operating systems (array, see [campaign settings](#filterset)). |
| `browser` | Acceptable browsers (array, see [campaign settings](#filterset)). |
| `ipmain` | Check only the main IP address of the request against the blacklist. |
| `lang` | Check the visitor's language against the list of allowed countries. |
| `adid` | Check the ad network click identifier. |
| `list` | Traffic filter list (array). |
| `langs` | List of two-letter ISO codes of acceptable languages (array). |
| `nogeo` | List of two-letter ISO country codes from which access is prohibited (array). |
| `uwl` | Whitelist of words in URL (array). |
| `ubl` | Blacklist of words in URL (array). |
| `wl` | Whitelist of IP addresses (array). |
| `log` | Visit log mode: `0` log all visits, `1` only target visits, `2` only dummy visits, `3` do not log (see [campaign settings](#filterset)). |
| `wd1`..`wd7` | Weekdays on which visits are allowed: `wd1` is Monday, `wd7` is Sunday (boolean). |
| `fh`, `fm` | Start of the daily time window: hour and minute. |
| `th`, `tm` | End of the daily time window: hour and minute. |
| `tz` | Time zone of the schedule in the IANA format, or an empty string (the server time zone). |
| `stats` | Filtering campaign statistics. Similar to the [campaign list](#filterlist) function. |
| `postback` | Postback samples for mini-tracker. Associative array (see below) |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-site` - the campaign was not found in the current scope.
- `no-domain` - neither the campaign identifier nor the domain or campaign name was specified.

Postback samples: 

- `new` - new lead.
- `approve` - approved lead.
- `paid` - approved lead with payout in macro `{payout}`.
- `cancel` - rejected lead. 

**Example of a successful server response:**

```json
{
    "id": 4567,
    "user": 123,
    "api": "https://domain.com/fltr/123-654b807372f614e64b9-4567",
    "domain": "pretty-domain.com",
    "mode": 0,
    "geo": [ "ru", "ua", "by" ],
    "mblack": 1,
    "black": "https://my-main-page.com/",
    "mwhite": 0,
    "white": "http://natribu.org/",
    "trash": 0,
    "proto": 0,
    "mobile": 0,
    "os": [],
    "browser": [],
    "ipmain": false,
    "lang": false,
    "adid": false,
    "list": [],
    "langs": [],
    "nogeo": [],
    "uwl": [],
    "ubl": [],
    "wl": [],
    "log": 0,
    "wd1": true,
    "wd2": true,
    "wd3": true,
    "wd4": true,
    "wd5": true,
    "wd6": true,
    "wd7": true,
    "fh": 0,
    "fm": 0,
    "th": 0,
    "tm": 0,
    "tz": "",
    "stats": {
        "click": 180,
        "black": 123,
        "white": 57,
        "leads": 11,
        "approve": 6,
        "cancel": 1,
        "cash": 387.44,
        "cr": 8.94,
        "epc": 3.15,
        "appr": 54.55
    },
    "postback": {
        "new": "https://domain.com/api/filter/postback.json?id=123-654b807372f614e64b9&uid=11&status=new",
        "approve": "https://domain.com/api/filter/postback.json?id=123-654b807372f614e64b9&uid=11&status=approve",
        "paid": "https://domain.com/api/filter/postback.json?id=123-654b807372f614e64b9&uid=11&status=approve&payout={payout}",
        "cancel": "https://domain.com/api/filter/postback.json?id=123-654b807372f614e64b9&uid=11&status=cancel"
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

<a id="filterdel"></a>
## Delete campaign

**Webmaster URL**: `https://domain.com/api/filter/del.json?id={token}&domain={host}`

**Team URL**: `https://domain.com/api/filter/del.json?id={token}&domain={host}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/del.json?id={token}&domain={host}&scope=admin`

The function deletes a filtering campaign together with its visit log records. If the campaign identifier is passed in the `uid` field, the function will delete the campaign by this identifier. If the campaign domain is passed in the `domain` field, the function will attempt to find the campaign with the specified domain; if there are several campaigns with this name, the one with the highest identifier is deleted. Only the campaigns of the current [scope](#filterscopes) can be deleted. The deletion cannot be undone.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `uid` | Filtering campaign identifier (recommended). |
| `domain` | Campaign domain or name (use unique names). |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

The result of the function execution is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of successful execution, `error` in case of an error. |
| `error` | Error code for status `error` (see below) |
| `id` | Numeric campaign identifier. |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-site` - the campaign was not found in the current scope.
- `no-domain` - neither the campaign identifier nor the domain or campaign name was specified.
- `db` - a system error occurred with the database.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "id": 4567
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-site"
}
```

<a id="filtermake"></a>
## Download filtering file

**Webmaster URL**: `https://domain.com/api/filter/make.json?id={token}&domain={host}`

**Team URL**: `https://domain.com/api/filter/make.json?id={token}&domain={host}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/make.json?id={token}&domain={host}&scope=admin`

This function downloads a prepared file for the filter campaign. If the campaign identifier is passed in the `uid` field, the function will extract the campaign by this identifier. If the campaign domain is passed in the `domain` field, the function will attempt to find the campaign with the specified domain. The campaign is searched for in the [scope](#filterscopes) of the request. The response of the function is the `index.php` file.

The file contains the API key of the campaign **owner** in plain text, also when it is downloaded by a team leader or an administrator, so do not publish it anywhere except the server of the campaign.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `uid` | The identifier of the filtering campaign (recommended). |
| `domain` | The domain or name of the filtering campaign (use unique names). |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

The result of the function execution is the `index.php` file in case of success, or an associative array in case of error:

| Field | Description |
| --- | --- |
| `status` | Result of the operation: `error` in case of error. |
| `error` | Error code for status `error` (see below) |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-site` - the campaign was not found in the current scope.
- `no-domain` - neither the campaign identifier nor the domain or campaign name is specified.

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "no-site"
}
```

<a id="filterblock"></a>
## Block IP address or header

**Webmaster URL**: `https://domain.com/api/filter/block.json?id={token}&list={list}&ip={ip}`

**Team URL**: `https://domain.com/api/filter/block.json?id={token}&list={list}&ip={ip}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/block.json?id={token}&list={list}&ip={ip}&scope=admin`

This function adds one entry to the selected blacklist. It can block both IP addresses and ranges, as well as headers. The entry kind is determined by the kind of the list (IP addresses, IP networks, IPv6 addresses or headers), not by the parameters. The lists that can be changed depend on the [scope](#filterscopes): in the webmaster scope these are your personal lists, in the team scope the lists of your team and the personal lists of the member passed in `user`, in the admin scope any list (or the lists of the passed `user` or `gang`). A list that is not available returns `bad-list`. The contents of the fields and the used identifiers are described in the [blacklist documentation](blacklist.md).

The blacklists themselves are created, configured and cleaned with the [blacklist API](api-blacklist.md); its function [Add an entry](api-blacklist.md#blacklistblock) offers more options (entry expiry, updating an existing entry). The entries added by this function have no expiry; an entry that is already in the list returns `failed`. The lookup of IPv4 networks is cached for about 11 minutes per /24 block, so a new network may take that long to take effect for the addresses that were checked recently.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `list`* | Required parameter. Blacklist identifier. |
| `ip` | IP address or subnet to be blocked: IPv4 for the lists of IP addresses and IP networks, IPv6 for the lists of IPv6 addresses. Required for these list kinds. Private and reserved addresses are rejected (`bad-ip`). |
| `size` | Subnet size to block (recommended 24 for IPv4 and 64 for IPv6). Required for the lists of networks: from 8 to 31 for IPv4, from 32 to 128 for IPv6. The address is rounded down to the beginning of the network. |
| `value` | Header content to be blocked. Required for header lists: text or a regular expression, up to 250 characters, UTF-8. |
| `regexp` | Use regular expression check instead of substring search, 0 or 1. The expression is not validated when saved, a broken expression is skipped during filtering. |
| `white` | Header lists only: create a white rule, 0 or 1. A visit is blocked if the header does **not** match `value` or is not sent at all. By default the rule is a black one: a visit is blocked if the header matches. |
| `type` | Type of header to block, required for header lists: header ID from 1 to 255 (see [Header IDs](api-blacklist.md#blacklistheaders)). A rule with an unknown ID is saved but never fires. |
| `comment` | Comment on the blocked entity: a number (see [Reason codes](api-blacklist.md#blacklistcomments) in the blacklist API). |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

The result of the function execution is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of successful execution, `error` in case of an error. |
| `error` | Error code for status `error` (see below) |
| `id` | Identifier of the added entry. |
| `ip` | IP address lists: the address as it was passed. IP network lists: the first address of the network as an integer. IPv6 lists: the first address of the network as a string. Not returned for header lists. |
| `size` | Network lists only: subnet size. |
| `value` | Header lists only: header content. |
| `type` | Header lists only: header ID. |
| `regexp` | Header lists only: the regular expression flag (boolean). |
| `white` | Header lists only: the white rule flag (boolean). |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-list` - the `list` parameter is not passed or equals 0.
- `bad-list` - the blacklist is not found or is not available in the current scope.
- `bad-ip` - invalid, private or reserved IP address.
- `bad-size` - incorrect subnet size.
- `bad-type` - incorrect type of header to block (not a number from 1 to 255).
- `bad-value` - incorrect header content to block (empty, longer than 250 characters or not UTF-8).
- `failed` - the entry could not be saved, for example it is already in the list.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "id": 1234567,
    "ip": "12.34.56.78"
}
```

**Example of a successful server response for a header list:**

```json
{
    "status": "ok",
    "id": 77,
    "value": "HeadlessChrome",
    "type": 1,
    "regexp": false,
    "white": false
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-list"
}
```

<a id="filterpostback"></a>
## Send postback to mini-tracker

**URL**: `https://domain.com/api/filter/postback.json?id={token}&uid={uid}&status={status}`

This function is responsible for operating the [built-in mini-tracker](tracker.md). It increases the lead counters with statuses like waiting, approval, and cancellation, as well as increments the earnings amount. It does not support any event deduplication.

The function does not use [scopes](#filterscopes): it works only with the campaigns of the owner of the key (an administrator key can act on behalf of another user with the `uu` parameter), and the `scope` parameter has no effect.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `uid` | Filtering campaign identifier (required parameter). |
| `payout` | Payout amount for approved lead. A comma is accepted as the decimal separator; a negative value decreases the earnings. |
| `status` | Lead status (required parameter, see below) |

Lead statuses:

- **New** - `lead`, `wait`, `process`, `hold`, `new`
- **Approved** - `approve`, `sale`, `approved`, `confirm`, `confirmed`
- **Rejected** - `cancel`, `trash`, `cancelled`, `reject`, `fake`

The result of the function execution is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of successful execution, `error` in case of an error. |
| `error` | Error code for status `error` (see below) |
| `filter` | Identifier of the filtering campaign. |

Error codes:

- `no-id` - filtering campaign ID not provided.
- `bad-id` - invalid filtering campaign ID: the campaign is not found or belongs to another user.
- `bad-status` - unable to recognize status.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "filter": 4567
}
```

**Example of an error server response:**

```json
{
    "status": "error",
    "error": "bad-status"
}
```

<a id="filterlog"></a>
## Visit log

**Webmaster URL**: `https://domain.com/api/filter/log.json?id={token}`

**Team URL**: `https://domain.com/api/filter/log.json?id={token}&scope=gang`

**Admin URL**: `https://domain.com/api/filter/log.json?id={token}&scope=admin`

The function returns one page of the visit log of the filter campaigns, the newest visits first. A record describes one checked visit: the check result, the blacklist that blocked it, the country, the IP address, the device data, the User-Agent, the discovered IP addresses and the captured request headers. The records are limited by the [scope](#filterscopes) of the request: your own visits, the visits of your team members or, for an administrator, all visits.

The function only reads the log: it does not run any checks and does not change the counters. A visit appears in the log with a delay, after the background processing of the log. A visit is not recorded if the campaign is paused or disabled, or if it is excluded by the log mode of the campaign (`log`, see [Campaign setup](#filterset)). The visits that are checked by a flow bound to a campaign are recorded in the log too, but they are not added to the campaign counters of [Campaigns list](#filterlist). Therefore the number of records can differ from the counters of a campaign.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `page` | Page number, defaults to 1. |
| `show` | Number of records per page, defaults to 50, the maximum is 1000. |
| `site` | Show only the visits of this campaign (identifier). |
| `list` | Show only the visits blocked by this blacklist (identifier). |
| `result` | Filter by the check result (see below). |
| `ip` | Show only the visits from this IP address (IPv4 or IPv6). An invalid address is ignored. |
| `ua32` | Show only the visits with this User-Agent: the `ua32` value of a record. The value `0` is ignored. |
| `geo` | Show only the visits from this country, two-letter code. |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

Values of the `result` filter:

- `-2` - only allowed visits (the target site was shown, result `0`).
- `-1` - only blocked visits (any non-zero result).
- `0`..`14` - exactly this [check result code](#filterreasons).

Invalid values of the filters are ignored. The `ip` and `ua32` filters compare 32-bit hashes, so a match is not a guarantee of the exact address or User-Agent (different IPv6 addresses may coincide).

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Records of the page: an object whose keys are the record identifiers, the newest first (an empty page is an empty array). The fields of a record are described below. |
| `nav` | Paging: `total` is the number of records matching the filters, `page` is the current page, `pages` is the number of pages, `show` is the page size. A page beyond the last one returns empty `data`. |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).

Fields of a `data` record:

| Field | Description |
| --- | --- |
| `id` | Record identifier. |
| `user` | Identifier of the campaign owner. |
| `site` | Campaign identifier. |
| `time` | Visit time, Unix timestamp. |
| `result` | Check result code from 0 to 14 (see [check result codes](#filterreasons)); `0` - the target site was shown. |
| `list` | Identifier of the blacklist that blocked the visit, `0` if there is none. |
| `geo` | Two-letter country code in lower case, `zz` if the country is not determined. |
| `ip` | Visitor IP address (IPv4 or IPv6). |
| `mobile` | `true` for a mobile User-Agent. |
| `os` | Operating system ID (see [campaign settings](#filterset)), `0` - not determined. |
| `browser` | Browser ID (see [campaign settings](#filterset)), `0` - not determined, `11` - bot. |
| `ua` | User-Agent of the visitor. |
| `ua32` | 32-bit hash (CRC32) of the User-Agent, can be used in the `ua32` filter. |
| `ips` | IP addresses found in the request headers (array). |
| `headers` | Captured request headers: an object `{ "header name": "value" }` without the User-Agent. If a header has no known name, its numeric ID is used as the key. |

Retention: by default the records are kept for 90 days, and the raw data (`ua`, `ips`, `headers`) for 30 days. After that the record stays in the log, but `ua` is an empty string and `ips` and `headers` are empty arrays.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "8452107": {
            "id": 8452107,
            "user": 123,
            "site": 4567,
            "time": 1791979245,
            "result": 0,
            "list": 0,
            "geo": "ru",
            "ip": "91.203.45.12",
            "mobile": true,
            "os": 5,
            "browser": 1,
            "ua": "Mozilla/5.0 (Linux; Android 13; Pixel 7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Mobile Safari/537.36",
            "ua32": 2903417785,
            "ips": [ "91.203.45.12" ],
            "headers": {
                "Accept-Language": "ru-RU,ru;q=0.9,en-US;q=0.8"
            }
        },
        "8452106": {
            "id": 8452106,
            "user": 123,
            "site": 4567,
            "time": 1791979231,
            "result": 8,
            "list": 0,
            "geo": "us",
            "ip": "66.249.66.1",
            "mobile": false,
            "os": 0,
            "browser": 11,
            "ua": "Googlebot/2.1 (+http://www.google.com/bot.html)",
            "ua32": 1187530224,
            "ips": [ "66.249.66.1" ],
            "headers": []
        }
    },
    "nav": {
        "total": 2,
        "page": 1,
        "pages": 1,
        "show": 50
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

<a id="filterstats"></a>
## Visit statistics

**Webmaster URL**: `https://domain.com/api/filter/stats.json?id={token}&item=date`

**Team URL**: `https://domain.com/api/filter/stats.json?id={token}&item=date&scope=gang`

**Admin URL**: `https://domain.com/api/filter/stats.json?id={token}&item=date&scope=admin`

The function returns the statistics of the checked visits, grouped by days or by campaigns for the selected period: the total number of visits, the number of target ("black") and dummy ("white") visits, and the distribution of the block reasons with percentages. The campaigns are limited by the [scope](#filterscopes) of the request.

The statistics is calculated from the [visit log](#filterlog), so the same restrictions apply: the visits excluded by the log mode of the campaign are absent, and the data are available for the retention period of the log (90 days by default). Therefore the numbers can differ from the campaign counters of [Campaigns list](#filterlist), which are cumulative, also count the visits of paused and disabled campaigns and do not include the checks of flows.

The function can accept the following parameters:

| Field | Description |
| --- | --- |
| `item`* | Required parameter. Grouping: `date` - one row per calendar day, `site` - one row per campaign. |
| `from` | First day of the period in the `YYYY-MM-DD` format. Defaults to 6 days before `to` (a period of 7 days). If it is later than `to`, it is replaced with `to`. |
| `to` | Last day of the period (inclusive) in the `YYYY-MM-DD` format. Defaults to today; a later date is replaced with today. |
| `site` | Count only the visits of this campaign (identifier). |
| `page` | Page number of the rows, defaults to 1. |
| `show` | Number of rows per page, defaults to 50, the maximum is 1000. |
| `user`, `gang` | Scope parameters, see [Scopes](#filterscopes). |

The days are counted in the server time zone. The length of the period is not limited.

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | Rows of the page: an object whose keys are the row identifiers (`id`). With `item=date` the rows go from the newest day to the oldest, with `item=site` in alphabetical order of campaign names. The days and campaigns without visits in the period are absent. An empty page is an empty array. The fields of a row are described below. |
| `nav` | Paging, same as in [Visit log](#filterlog); `total` is the number of rows (days or campaigns). |
| `dataset` | Reference data: `results` is an object with the labels of the check result codes `0`..`14` in the language of the user account. |

Error codes:

- `scope` - the scope cannot be resolved or you have no rights for it (see [Scopes](#filterscopes)).
- `no-item` - the `item` parameter is not passed.
- `bad-item` - `item` is neither `date` nor `site`.
- `bad-from` - `from` is not a valid date in the `YYYY-MM-DD` format.
- `bad-to` - `to` is not a valid date in the `YYYY-MM-DD` format.

Fields of a `data` row:

| Field | Description |
| --- | --- |
| `id` | With `item=date`: the day as the integer `YYYYMMDD`. With `item=site`: campaign identifier. |
| `name` | With `item=date`: the day in the `YYYY-MM-DD` format. With `item=site`: campaign name (`ID <id>` if the campaign is not found or not available in the scope). |
| `user` | Only with `item=site`: identifier of the campaign owner (`0` if the campaign is not found). |
| `click` | Total number of logged visits. |
| `black` | Number of target visits (result `0`). |
| `black_pct` | Share of target visits in `click`, percent, one decimal. |
| `white` | Number of blocked visits (any non-zero result). |
| `white_pct` | Share of blocked visits in `click`, percent, one decimal. |
| `reason` | Object: the number of visits for each check result code from `1` to `14` (all keys are present, zero if there were no such visits). |
| `reason_pct` | Object with the same keys: the share of each code in `click`, percent, one decimal. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "20261014": {
            "id": 20261014,
            "name": "2026-10-14",
            "click": 180,
            "black": 123,
            "black_pct": 68.3,
            "white": 57,
            "white_pct": 31.7,
            "reason": { "1": 2, "2": 14, "3": 9, "4": 6, "5": 3, "6": 1, "7": 2, "8": 8, "9": 0, "10": 4, "11": 3, "12": 0, "13": 5, "14": 0 },
            "reason_pct": { "1": 1.1, "2": 7.8, "3": 5, "4": 3.3, "5": 1.7, "6": 0.6, "7": 1.1, "8": 4.4, "9": 0, "10": 2.2, "11": 1.7, "12": 0, "13": 2.8, "14": 0 }
        },
        "20261013": {
            "id": 20261013,
            "name": "2026-10-13",
            "click": 40,
            "black": 31,
            "black_pct": 77.5,
            "white": 9,
            "white_pct": 22.5,
            "reason": { "1": 0, "2": 3, "3": 2, "4": 1, "5": 0, "6": 0, "7": 0, "8": 2, "9": 0, "10": 0, "11": 0, "12": 0, "13": 1, "14": 0 },
            "reason_pct": { "1": 0, "2": 7.5, "3": 5, "4": 2.5, "5": 0, "6": 0, "7": 0, "8": 5, "9": 0, "10": 0, "11": 0, "12": 0, "13": 2.5, "14": 0 }
        }
    },
    "nav": {
        "total": 2,
        "page": 1,
        "pages": 1,
        "show": 50
    },
    "dataset": {
        "results": {
            "0": "Site shown successfully",
            "1": "Request to invalid path or domain",
            "2": "Traffic from wrong device",
            "3": "Request from blocked country",
            "4": "IP address blacklisted",
            "5": "IP network blacklisted",
            "6": "IPv6 address blacklisted",
            "7": "One of request headers blacklisted",
            "8": "Bot or site preview",
            "9": "Traffic via wrong protocol",
            "10": "Wrong visitor language",
            "11": "First trash visits",
            "12": "URL blacklisted",
            "13": "No AD click ID",
            "14": "Visit outside the campaign timetable"
        }
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
