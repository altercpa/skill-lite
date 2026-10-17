# Statistics of tracker

Statistics functions provide access to reports on leads and clicks.

<a id="leads"></a>
## Leads list

**URL for user**: `https://domain.com/api/tracker/leads.json?id={token}`

**URL for team**: `https://domain.com/api/gangs/leads.json?id={token}`

The function retrieves the list of leads and returns it as an associative array in the `leads` field. Parameters may be passed via GET or POST (`form-urlencoded` or JSON). Use the `show` and `page` parameters for page navigation; without `show`, all matching leads are returned at once (capped at 10000 rows). It is not recommended to load more than a thousand leads at a time — if you get a 500 error, there was not enough memory for your request, so reduce the request period or use pagination. Use the `sort` and `order` parameters to control the order of returned leads; without `sort`, leads are returned in natural (insertion) order.

The function can accept the following GET or POST filtering parameters as input:

| Field | Description |
| --- | --- |
| `from` | Report start date in `YYYY-MM-DD` format and tracker timezone. |
| `to` | Report end date in `YYYY-MM-DD` format and tracker timezone. |
| `ufrom` | Report start date in Unix Timestamp format. |
| `uto` | Report end date in Unix Timestamp format. |
| `status` | Character status of the lead (see below) |
| `geo` | Two-letter ISO code of lead country. |
| `flow` | Character flow ID. |
| `way` | Numeric way ID. |
| `click` | Character click ID. |
| `site` | Numeric site ID. |
| `goal` | Character goal ID. |
| `filter` | The result of the filters (see below) |
| `mobile` | Traffic type: `0` for desktop traffic or `1` for mobile traffic. |
| `os` | Operating system numeric ID (see the list below). |
| `browser` | Browser numeric ID (see the list below). |
| `hour` | Hour of the day when the lead arrived (number from 0 to 23), counted in the tracker time zone, the same as `from` and `to`. |
| `domain` | The domain from which the lead was sent. |
| `ip` | Lead IP address. |
| `utm_source` | The value of **UTM Source** tag. |
| `utm_campaign` | The value of **UTM Campaign** tag. |
| `utm_content` | The value of **UTM Content** tag. |
| `utm_term` | The value of **UTM Term** tag. |
| `utm_medium` | The value of **UTM Medium** tag. |
| `subid` | The value of **SubID** tag. |
| `customer` | The value of **Customer** tag. |
| `sort` | Column to sort leads by (see below). Without it, leads are returned in natural (insertion) order. |
| `order` | Sort direction: `asc` or `desc`. By default `desc`. |
| `user` | User ID in team. |
| `show` | Number of leads per page, at most `1000`: a larger value is reduced to `1000`, and `1000` is returned in the response. Enables pagination. Without it, or with a value below `1`, all matching leads are returned (capped at 10000 rows) and `show` is `-1` in the response. |
| `page` | Page number, starting from `1` (used together with `show`). |

Character statuses of the lead (`status`):

- `new` - new leads.
- `hold` - leads in the hold.
- `approve` - approved leads.
- `cancel` - cancelled leads.
- `trash` - leads in trash.
- `valid` - valid leads (virtual status, everything except trash).
- `wait` - pending leads (virtual status, new and hold).
- `good` - good leads (virtual status, hold and approve).
- `bad` - bad leads (virtual status, cancelled and trash). 

The result of the filters (`filter`):

- `0` - target visits.
- `1` - blocked: bot.
- `2` - blocked: blacklist.
- `3` - blocked: country.
- `4` - blocked: language.
- `5` - blocked: filtering campaign.

Columns available for `sort` (`sort`):

- `id` — lead ID (default; also used when an unrecognized value is passed).
- `time` / `date` — lead arrival time.
- `status` — lead status.
- `filter` — filter result.
- `goal` — goal ID.
- `price` — lead price.
- `cost` — lead cost.
- `geo` — lead country.
- `flow` — flow ID.
- `way` — way ID.
- `site` — site ID.

The result of the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of success, `error` in case of an error. |
| `leads` | An array of leads, the composition of the fields is analyzed further. |
| `page` | Current page number (echoes the request, `1` by default). |
| `show` | Page size actually used (not more than `1000`), or `-1` when pagination was not requested. |
| `total` | Total number of matching leads. When paginated it is the full count; otherwise it equals the number of returned leads. |
| `error` | Error code for status `error` with value from the list:
						
`access` - you are trying to access data that does not belong to you.
`payment-required` - your subscription has expired, [renew your subscription](/payments). |

Inside the `leads` field, leads are represented by an associative array with fields:

| Field | Description |
| --- | --- |
| `id` | Lead numeric ID. |
| `user` | User ID in team. |
| `time` | Lead arrival time in UNIX Timestamp format. |
| `click` | Character click ID. |
| `uid` | Unique lead identifier on the advertiser's side. |
| `flow` | Character flow ID. |
| `way` | Numeric way ID. |
| `site` | Numeric site ID. |
| `siteurl` | Site URL. |
| `filter` | The result of the filters (see above) |
| `status` | Character status of the lead (see above) |
| `custom` | Real statustext of the lead sent by the advertiser. |
| `goal` | Character goal ID. |
| `price` | Lead price (always available regardless of status). |
| `currency` | Lead price currency, by default `usd`. |
| `cost` | Lead cost (not the commission price, it's above). |
| `costcur` | Lead cost currency, by default `usd`. |
| `ip` | Lead IP address. |
| `geo` | Two-letter ISO code of lead country. |
| `name` | Customer's name. |
| `phone` | Customer's phone (without validation and correction). |
| `email` | Customer's email (without validation and correction). |
| `comment` | Lead comment. |
| `meta` | Array of additional lead fields. |
| `os` | Operating system numeric ID, detected from the `ua` field (see the list below). `0` if no user-agent was stored. |
| `browser` | Browser numeric ID, detected from the `ua` field (see the list below). `0` if no user-agent was stored. |
| `ua` | Browser User Agent |
| `adid` | Ad click ID (`fbclid`, `gclid`, `adclid`, `clid`, `adid`). |
| `data1` | Custom data 1. |
| `data2` | Custom data 2. |
| `data3` | Custom data 3. |
| `data4` | Custom data 4. |
| `data5` | Custom data 5. |
| `domain` | The domain from which the lead was sent. |
| `utm_source` | The value of UTM Source tag. |
| `utm_campaign` | The value of UTM Campaign tag. |
| `utm_content` | The value of UTM Content tag. |
| `utm_term` | The value of UTM Term tag. |
| `utm_medium` | The value of UTM Medium tag. |
| `customer` | The value of Customer tag. |
| `subid` | The value of `subid` tag. |

Operating system (`os`):

- `0` - Other
- `1` - Windows
- `2` - MacOS
- `3` - Linux
- `4` - iOS
- `5` - Android
- `6` - Windows Mobile
- `7` - Symbian
- `8` - BlackBerry

Browsers (`browser`):

- `0` - Other
- `1` - Chrome
- `2` - Firefox
- `3` - MSIE
- `4` - Safari
- `5` - Opera
- `6` - Edge
- `7` - Facebook
- `8` - Instagram
- `9` - Tik-Tok
- `10` - WebView
- `11` - Bot

**Example of a successful server response:**

```json
{
    "status": "ok",
    "page": 1,
    "show": 100,
    "total": 512,
    "leads": [
        {
            "id": 1,
            "user": 13,
            "time": 1691303223,
            "click": "ac13",
            "uid": "c1vvkg272m16",
            "flow": "ecb4",
            "way": 123,
            "site": 42,
            "siteurl": "https://meow.pro/c8/cfb72fe561f0/",
            "filter": 0,
            "status": "approve",
            "custom": "Confirmed",
            "goal": "paid",
            "price": 18,
            "currency": "usd",
            "cost": 35,
            "costcur": "eur",
            "ip": "12.34.56.78",
            "geo": "de",
            "name": "Hans",
            "phone": "+49 (30) 226-511-83",
            "email": "noreply@gmx.de",
            "comment": "Confirmed on sunday",
            "meta": {
                "foo": "bar",
                "bar": "foo"
            },
            "os": 2,
            "browser": 4,
            "ua": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Safari\605.1.15",
            "adid": "bf9d4aa8-4979-11ee-94c6-005056aec456",
            "data1": "wj80b89eit8epaarikhe2oge",
            "data2": false,
            "utm_source": "fb",
            "utm_campaign": "1013444211662",
            "utm_content": "432211555332166",
            "utm_term": false,
            "utm_medium": false,
            "domain": "meow.pro",
            "customer": "vendoso",
            "subid": "purr"
        },
        ...
    ]
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "payment-required"
}
```

<a id="lead"></a>
## Lead details

**URL for user**: `https://domain.com/api/tracker/lead.json?id={token}&lead={lead_id}`

**URL for team**: `https://domain.com/api/gangs/lead.json?id={token}&lead={lead_id}`

The function returns one lead by its numeric ID together with the history of its status changes. Use it when you need the full card of a single lead, for example to see how its status, price and cost changed over time. Parameters may be passed via GET or POST (`form-urlencoded` or JSON).

The function accepts the following input parameters:

| Field | Description |
| --- | --- |
| `lead`* | Numeric lead ID. Required parameter. |
| `user` | User ID in team. Used with the team URL: the lead must belong to this team member. |

The function returns an associative array:

| Field | Description |
| --- | --- |
| `status` | Operation result: `ok` on success, `error` on failure. |
| `error` | Error code for `error` status (see below) |
| `data` | The lead, the composition of the fields is analyzed further. |

Error codes:

- `payment-required` - your subscription has expired, [renew your subscription](/payments).
- `no-lead` - the `lead` parameter is missing or empty.
- `not-found` - there is no such lead, or it does not belong to you (with the team URL — to your team).
- `no-gang` - you are not a member of a team (team URL only).
- `access` - you are not a leader of the team (team URL only).

Inside the `data` field, the lead is represented by an associative array with fields:

| Field | Description |
| --- | --- |
| `id` | Lead numeric ID. |
| `user` | User ID in team. |
| `gang` | ID of the team the lead belongs to, `0` if there is none. |
| `time` | Lead arrival time in UNIX Timestamp format. |
| `date` | Lead arrival date as a number in `YYYYMMDD` format. |
| `status` | Character status of the lead (see the [leads list](#leads)) |
| `custom` | Real statustext of the lead sent by the advertiser. |
| `goal` | Character goal ID. |
| `price` | Lead price (always available regardless of status). |
| `currency` | Lead price currency, by default `usd`. |
| `source` | Where the lead came from, an array described below. |
| `buyer` | Customer data, an array described below. |
| `visitor` | Visitor data, an array described below. |
| `tags` | Tags of the lead, an array described below. |
| `meta` | Array of additional lead fields. |
| `changes` | List of the lead's status changes, oldest first; an empty list if there were none. Composition detailed below. |

Fields of `source`:

| Field | Description |
| --- | --- |
| `click` | Character click ID. |
| `uid` | Unique lead identifier on the advertiser's side. |
| `flow` | Character flow ID. |
| `way` | Numeric way ID. |
| `site` | Numeric site ID. |
| `siteurl` | Site URL. |
| `filter` | The result of the filters (see the [leads list](#leads)) |

Fields of `buyer`:

| Field | Description |
| --- | --- |
| `name` | Customer's name. |
| `phone` | Customer's phone (without validation and correction). |
| `email` | Customer's email (without validation and correction). |
| `comment` | Lead comment. |
| `cost` | Lead cost (not the commission price, it's above). |
| `costcur` | Lead cost currency, by default `usd`. |

Fields of `visitor`:

| Field | Description |
| --- | --- |
| `ip` | Lead IP address. |
| `geo` | Two-letter ISO code of lead country. |
| `mobile` | Traffic type: `0` for desktop traffic or `1` for mobile traffic. |
| `os` | Operating system numeric ID, detected from the `ua` field (see the [leads list](#leads)). `0` if no user-agent was stored. |
| `browser` | Browser numeric ID, detected from the `ua` field (see the [leads list](#leads)). `0` if no user-agent was stored. |
| `ua` | Browser User Agent |

Fields of `tags`:

| Field | Description |
| --- | --- |
| `adid` | Ad click ID (`fbclid`, `gclid`, `adclid`, `clid`, `adid`). |
| `data1` | Custom data 1. |
| `data2` | Custom data 2. |
| `data3` | Custom data 3. |
| `data4` | Custom data 4. |
| `data5` | Custom data 5. |
| `utm_source` | The value of UTM Source tag. |
| `utm_campaign` | The value of UTM Campaign tag. |
| `utm_content` | The value of UTM Content tag. |
| `utm_term` | The value of UTM Term tag. |
| `utm_medium` | The value of UTM Medium tag. |
| `domain` | The domain from which the lead was sent. |
| `subid` | The value of `subid` tag. |
| `customer` | The value of Customer tag. |

A change is recorded when the status, the price or the cost of the lead changes. Each element of `changes` is an associative array with fields:

| Field | Description |
| --- | --- |
| `id` | Change numeric ID. |
| `time` | Time of the change in UNIX Timestamp format. |
| `from` | Status number before the change (see below). |
| `to` | Status number after the change (see below). It equals `from` if only the price or the cost changed. |
| `from_status` | Character status before the change (see the [leads list](#leads)), `false` if unknown. |
| `to_status` | Character status after the change (see the [leads list](#leads)), `false` if unknown. |
| `price` | Lead price after the change. |
| `cost` | Lead cost after the change. |

Status numbers (`from`, `to`):

- `1` - new.
- `2` - hold.
- `3` - approve.
- `4` - cancel.
- `5` - trash.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "id": 77,
        "user": 13,
        "gang": 0,
        "time": 1760698200,
        "date": 20261017,
        "status": "approve",
        "custom": "Confirmed",
        "goal": "paid",
        "price": 18,
        "currency": "usd",
        "source": {
            "click": "ca1337",
            "uid": "c1vvkg272m16",
            "flow": "ef1c",
            "way": 123,
            "site": 42,
            "siteurl": "https://meow.pro/c8/cfb72fe561f0/",
            "filter": 0
        },
        "buyer": {
            "name": "Hans",
            "phone": "+49 (30) 226-511-83",
            "email": "noreply@gmx.de",
            "comment": "Confirmed on sunday",
            "cost": 35,
            "costcur": "eur"
        },
        "visitor": {
            "ip": "12.34.56.78",
            "geo": "de",
            "mobile": 0,
            "os": 2,
            "browser": 4,
            "ua": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Safari/605.1.15"
        },
        "tags": {
            "adid": "bf9d4aa8-4979-11ee-94c6-005056aec456",
            "data1": "wj80b89eit8epaarikhe2oge",
            "data2": false,
            "data3": false,
            "data4": false,
            "data5": false,
            "utm_source": "fb",
            "utm_campaign": "1013444211662",
            "utm_content": "432211555332166",
            "utm_term": false,
            "utm_medium": false,
            "domain": "meow.pro",
            "subid": "purr",
            "customer": "vendoso"
        },
        "meta": {
            "foo": "bar"
        },
        "changes": [
            {
                "id": 301,
                "time": 1760698260,
                "from": 1,
                "to": 2,
                "from_status": "new",
                "to_status": "hold",
                "price": 0,
                "cost": 35
            },
            {
                "id": 307,
                "time": 1760698500,
                "from": 2,
                "to": 3,
                "from_status": "hold",
                "to_status": "approve",
                "price": 18,
                "cost": 35
            }
        ]
    }
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "not-found"
}
```

<a id="stats"></a>
## Click statistics

**URL for user**: `https://domain.com/api/tracker/stats.json?id={token}`

**URL for team**: `https://domain.com/api/gangs/stats.json?id={token}`

The function collects general statistics on the number of clicks and leads, broken down by goals for the selected period according to the specified criterion. Parameters may be passed via GET or POST (`form-urlencoded` or JSON). Use the `show` and `page` parameters for page navigation. Use the `sort` and `order` parameters to control the order of the grouping rows; sorting is applied to the entire result set before it is split into pages, so paginated results stay correctly ordered across page boundaries.

The function can accept the following GET or POST filtering parameters as input:

| Field | Description |
| --- | --- |
| `from`* | Report start date in `YYYY-MM-DD` format (required). |
| `to`* | Report end date in `YYYY-MM-DD` format (required). |
| `item`* | Element by which statistics are grouped (required, see below) |
| `geo` | Two-letter ISO code of lead country. |
| `flow` | Character flow ID. |
| `way` | Numeric way ID. |
| `site` | Numeric site ID. |
| `filter` | The result of the filters (see below) |
| `hour` | Lead arrival hour (number from 0 to 23). |
| `mobile` | Traffic type: `0` for desktop traffic or `1` for mobile traffic. |
| `os` | Operating system (see below) |
| `browser` | Browser (see below) |
| `domain` | The domain from which the lead was sent. |
| `ip` | Lead IP address. |
| `utm_source` | The value of UTM Source tag. |
| `utm_campaign` | The value of UTM Campaign tag. |
| `utm_content` | The value of UTM Content tag. |
| `utm_term` | The value of UTM Term tag. |
| `utm_medium` | The value of UTM Medium tag. |
| `customer` | The value of Customer tag. |
| `subid` | The value of `subid` tag. |
| `sort` | Column to sort the grouping rows by: `name`, `id`, `click`, `unique`, `spend`, `next`, `cn`, `ch`, `ca`, `cc`, `ct`, `cv`, `total`, `mt`, `mh`, `ma`, `mmt`, `mmh`, `mma`, `mct`, `mch`, `mca`, `ur`, `cr`, `crt`, `epc`, `epl`, `appr`, `app`, `per`, `cpc`, `cpa`, `cpv`, `cpl`, `profit`, or `roi` — the same names as the fields of the `stats`/`total` object below (see below). Any other value falls back to `name`. Without `sort`, rows are returned in natural (insertion) order. |
| `order` | Sort direction: `asc` or `desc`. By default `desc`. |
| `show` | Number of grouping rows per page. Enables pagination. Without it, all rows are returned and `show` is `-1` in the response. |
| `page` | Page number, starting from `1` (used together with `show`). |

Statistic groupping elements (`item`):

- `date` - the date the click and lead were received.
- `hour` - hour of the click and lead.
- `flow` - flow ID.
- `way` - way ID.
- `site` - site ID.
- `filter` - filtering results.
- `geo` - country.
- `mobile` - device type.
- `os` - operating system.
- `browser` - browser.
- `ip` - IP address.
- `utmd` - domain.
- `utmu` - customer.
- `subid` - SubID tag.
- `utms` - UTM Source tag.
- `utmc` - UTM Campaign tag.
- `utmn` - UTM Content tag.
- `utmt` - UTM Term tag.
- `utmm` - UTM Medium tag.
- `user` - user ID in team.

The result of the filters (`filter`):

- `0` - target visits.
- `1` - blocked: bot.
- `2` - blocked: blacklist.
- `3` - blocked: country.
- `4` - blocked: language.
- `5` - blocked: filtering campaign.

Operating system (`os`):

- `0` - Other
- `1` - Windows
- `2` - MacOS
- `3` - Linux
- `4` - iOS
- `5` - Android
- `6` - Windows Mobile
- `7` - Symbian
- `8` - BlackBerry

Browsers (`browser`):

- `0` - Other
- `1` - Chrome
- `2` - Firefox
- `3` - MSIE
- `4` - Safari
- `5` - Opera
- `6` - Edge
- `7` - Facebook
- `8` - Instagram
- `9` - Tik-Tok
- `10` - WebView
- `11` - Bot

The result of the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of success, `error` in case of an error. |
| `stats` | The array of statistics results, the composition of the fields is analyzed further. |
| `total` | The total results for the entire query. |
| `page` | Current page number (echoes the request, `1` by default). |
| `show` | Page size, or `-1` when pagination was not requested. |
| `items` | Total number of grouping rows before pagination (use it to calculate the number of pages). |
| `error` | Error code for status `error` (see below) |

Error codes: 

- `access` - you are trying to access data that does not belong to you.
- `no-item` - the criterion by which statistics should be groupped is not specified.
- `no-from` - start date of statistics period is not specified.
- `no-to` - end date of statistics period is not specified.
- `payment-required` - your subscription has expired, [renew your subscription](/payments).

The `stats` array and the `total` field are an associative array:

| Field | Description |
| --- | --- |
| `id` | Statistics element ID. |
| `name` | Statistics element name. |
| `click` | Total clicks amount. |
| `unique` | Unique clicks amount. |
| `next` | Amount of clicks moved to next step. |
| `per` | Percent of clicks moved to next step. |
| `ur` | Unique clicks ratio: the ratio of unique clicks to all clicks, as a percentage. |
| `total` | Total leads number. |
| `cv` | Valid leads number. |
| `cn` | New leads number. |
| `ch` | Hold leads number. |
| `ca` | Approved leads number. |
| `cc` | Cancelled leads number. |
| `ct` | Trash leads number. |
| `mt` | Total earnings. |
| `mh` | Holded leads earnings. |
| `ma` | Approved leads earnings. |
| `mct` | Total cost. |
| `mch` | Holded leads cost. |
| `mca` | Approved leads cost. |
| `mmt` | Avarage (mean) cost. |
| `mmh` | Holded leads avarage (mean) cost. |
| `mma` | Approved leads avarage (mean) cost. |
| `cr` | Conversion: the ratio of valid leads to unique clicks. |
| `crt` | Total conversion: the ratio of all the leads to unique clicks. |
| `epc` | EPC: the ratio of all earnings to unique clicks. |
| `epl` | EPL: earn per approved and holded leads. |
| `appr` | Approve: the ratio of approved leads to valid ones. |
| `app` | Total approve: the ratio of approved leads to all leads. |
| `cpc` | Average cost per click. |
| `cpl` | Average cost per lead in any status. |
| `cpv` | Average cost per valid lead. |
| `cpa` | Average cost per approved lead. |
| `spend` | Total cost for all clicks. |
| `profit` | Earnings: amount for approved leads minus expenses. |
| `roi` | ROI as a percentage, where a positive number represents profit, zero represents a full return on investment, and -100 represents a complete loss. |
| `goal` | The array with statistics for each individual goal. Contains the same fields as the rest of the statistics. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "stats": {
        "20230805": {
            "id": 20230805,
            "name": "2023-08-05",
            "click": 44,
            "unique": 22,
            "total": 6,
            "cv": 4,
            "cn": 1,
            "ch": 1,
            "ca": 2,
            "cc": 0,
            "ct": 2,
            "mh": 20.0,
            "ma": 135.0,
            "mt": 155.0,
            "cr": 18.2,
            "epc": 6.75,
            "appr": 66.7,
            "app": 33.3
        },
        ...
    },
    "total": {
        "click": 175,
        "unique": 58,
        "total": 25,
        "cv": 23,
        "cn": 11,
        "ch": 8,
        "ca": 9,
        "cc": 0,
        "ct": 2,
        "mh": 160.0,
        "ma": 1200.0,
        "mt": 1360.0,
        "cr": 5.17,
        "epc": 12.3,
        "appr": 46.7,
        "app": 31.0
    },
    "page": 1,
    "show": 100,
    "items": 37
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "no-item"
}
```

<a id="clicks"></a>
## Click list

**URL for user**: `https://domain.com/api/tracker/clicks.json?id={token}`

**URL for team**: `https://domain.com/api/gangs/clicks.json?id={token}`

The function retrieves the list of raw clicks — every visit that hit a flow, whether or not it produced a lead — and returns it as an associative array in the `data` field (note: `data`, not `leads`, unlike the leads list function above). Parameters may be passed via GET or POST (`form-urlencoded` or JSON). Use the `show` and `page` parameters for page navigation; without `show`, all matching clicks are returned at once (capped at 10000 rows). Use the `sort` and `order` parameters to control the order of returned clicks; without `sort`, clicks are returned in natural (insertion) order.

The function can accept the following GET or POST filtering parameters as input:

| Field | Description |
| --- | --- |
| `from` | Report start date in `YYYY-MM-DD` format and tracker timezone. |
| `to` | Report end date in `YYYY-MM-DD` format and tracker timezone. |
| `ufrom` | Report start date in Unix Timestamp format. |
| `uto` | Report end date in Unix Timestamp format. |
| `geo` | Two-letter ISO code of click country. |
| `ip` | Click IP address. |
| `flow` | Character flow ID. |
| `way` | Numeric way ID. |
| `site` | Numeric site ID. |
| `offer` | Numeric offer ID. |
| `click` | Character click ID. |
| `filter` | The result of the filters (see below) |
| `mobile` | Traffic type: `0` for desktop traffic or `1` for mobile traffic. |
| `os` | Operating system (see below) |
| `browser` | Browser (see below) |
| `unique` | Numeric flag: `1` for unique clicks only, `0` for repeated clicks only. |
| `next` | Numeric flag: `1` for clicks that moved to the second step, `0` for clicks that did not. |
| `hour` | Hour of the day when the click arrived (number from 0 to 23), counted in the tracker time zone, the same as `from` and `to`. |
| `domain` | The domain from which the click arrived. |
| `utm_source` | The value of **UTM Source** tag. |
| `utm_campaign` | The value of **UTM Campaign** tag. |
| `utm_content` | The value of **UTM Content** tag. |
| `utm_term` | The value of **UTM Term** tag. |
| `utm_medium` | The value of **UTM Medium** tag. |
| `subid` | The value of **SubID** tag. |
| `customer` | The value of **Customer** tag. |
| `sort` | Column to sort clicks by (see below). Without it, clicks are returned in natural (insertion) order. |
| `order` | Sort direction: `asc` or `desc`. By default `desc`. |
| `user` | User ID in team. |
| `show` | Number of clicks per page, at most `1000`: a larger value is reduced to `1000`, and `1000` is returned in the response. Enables pagination. Without it, or with a value below `1`, all matching clicks are returned (capped at 10000 rows) and `show` is `-1` in the response. |
| `page` | Page number, starting from `1` (used together with `show`). |

The result of the filters (`filter`):

- `0` - target visits.
- `1` - blocked: bot.
- `2` - blocked: blacklist.
- `3` - blocked: country.
- `4` - blocked: language.
- `5` - blocked: filtering campaign.

Operating system (`os`):

- `0` - Other
- `1` - Windows
- `2` - MacOS
- `3` - Linux
- `4` - iOS
- `5` - Android
- `6` - Windows Mobile
- `7` - Symbian
- `8` - BlackBerry

Browsers (`browser`):

- `0` - Other
- `1` - Chrome
- `2` - Firefox
- `3` - MSIE
- `4` - Safari
- `5` - Opera
- `6` - Edge
- `7` - Facebook
- `8` - Instagram
- `9` - Tik-Tok
- `10` - WebView
- `11` - Bot

Columns available for `sort` (`sort`):

- `id` — click ID (default; also used when an unrecognized value is passed).
- `time` / `date` — click time.
- `filter` — filter result.
- `price` — click price.
- `geo` — click country.
- `mobile` — device type.
- `os` — operating system.
- `browser` — browser.
- `flow` — flow ID.
- `way` — way ID.
- `site` — site ID.

The result of the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of success, `error` in case of an error. |
| `data` | An array of clicks, the composition of the fields is analyzed further. |
| `page` | Current page number (echoes the request, `1` by default). |
| `show` | Page size actually used (not more than `1000`), or `-1` when pagination was not requested. |
| `total` | Total number of matching clicks. When paginated it is the full count; otherwise it equals the number of returned clicks. |
| `error` | Error code for status `error` with value from the list:

`payment-required` - your subscription has expired, [renew your subscription](/payments). |

Inside the `data` field, clicks are represented by an associative array with fields:

| Field | Description |
| --- | --- |
| `id` | Click numeric ID. |
| `user` | User ID in team. |
| `time` | Click arrival time in UNIX Timestamp format. |
| `click` | Character click ID. |
| `flow` | Character flow ID. |
| `offer` | Numeric offer ID. |
| `way` | Numeric way ID. |
| `site` | Numeric site ID. |
| `sitenext` | Numeric ID of the site shown on the second step, when applicable. |
| `siteurl` | Site URL, or `false` when the click has no site. |
| `filter` | The result of the filters (see above) |
| `unique` | Whether this is a unique click. |
| `next` | Whether this click moved to the second step. |
| `price` | Click price. |
| `currency` | Click price currency, by default `usd`. |
| `ip` | Click IP address. |
| `geo` | Two-letter ISO code of click country. |
| `mobile` | Whether the click came from a mobile device. |
| `os` | Operating system numeric ID, detected from the `ua` field (see the list above). `0` if no user-agent was stored. |
| `browser` | Browser numeric ID, detected from the `ua` field (see the list above). `0` if no user-agent was stored. |
| `ua` | Browser User Agent, or `false` if not stored. |
| `adid` | Ad click ID (`fbclid`, `gclid`, `adclid`, `clid`, `adid`). |
| `data1` | Custom data 1. |
| `data2` | Custom data 2. |
| `data3` | Custom data 3. |
| `data4` | Custom data 4. |
| `data5` | Custom data 5. |
| `domain` | The domain from which the click arrived. |
| `utm_source` | The value of UTM Source tag. |
| `utm_campaign` | The value of UTM Campaign tag. |
| `utm_content` | The value of UTM Content tag. |
| `utm_term` | The value of UTM Term tag. |
| `utm_medium` | The value of UTM Medium tag. |
| `customer` | The value of Customer tag. |
| `subid` | The value of `subid` tag. |

Unlike the leads list, click objects carry no lead-status, custom-status, goal, or customer-contact fields (`name`/`phone`/`email`/`comment`/`meta`) — this is raw traffic, not a converted lead.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "page": 1,
    "show": 100,
    "total": 2048,
    "data": [
        {
            "id": 4821,
            "user": 13,
            "time": 1691303001,
            "click": "ac13",
            "flow": "ecb4",
            "offer": 7,
            "way": 123,
            "site": 42,
            "sitenext": 0,
            "siteurl": "https://meow.pro/c8/cfb72fe561f0/",
            "filter": 0,
            "unique": true,
            "next": false,
            "price": 0.15,
            "currency": "usd",
            "ip": "12.34.56.78",
            "geo": "de",
            "mobile": false,
            "os": 1,
            "browser": 1,
            "ua": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "adid": false,
            "data1": "wj80b89eit8epaarikhe2oge",
            "data2": false,
            "data3": false,
            "data4": false,
            "data5": false,
            "utm_source": "fb",
            "utm_campaign": "1013444211662",
            "utm_content": false,
            "utm_term": false,
            "utm_medium": false,
            "domain": "meow.pro",
            "customer": false,
            "subid": "purr"
        },
        ...
    ]
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "payment-required"
}
```
