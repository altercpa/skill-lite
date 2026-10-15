# Integration with tracker

Integration functions are responsible for connecting third-party services and transmitting conversion data.

<a id="postback"></a>
## Postback

**URL**: `https://domain.com/postbacks/{key}?click={click}`

**API URL for user**: `https://domain.com/api/tracker/postback.json?id={token}&click={click}`

**API URL for team**: `https://domain.com/api/gangs/postback.json?id={token}&click={click}`

The most important function of interaction with the tracker. Creates a new lead or modifies an existing lead based on the click or flow IDs sent. You can use two calling options:

- **Standard operation**: with the `click` parameter. The data for the lead is taken from the click ID. Used by default, this is the recommended approach.
- **Fallback operation**: with the `flow` and `subid` parameters. The data for the lead is taken from the stream ID, the uniqueness of the request is determined by the `subid` parameter. Suitable for cases where there is no concept of a click and a click on a link, for example, for application integration.

The function can take the following parameters as input:

| Field | Description |
| --- | --- |
| `click` | Character click ID. |
| `flow` | Character flow ID. |
| `uid` | Unique lead identifier on the advertiser's side. |
| `status` | Lead status, optimal: `new`, `hold`, `approve`, `cancel`, `trash`. |
| `goal` | Character ID of [goal](goal.md). |
| `create` | Create new lead on each postback received. Value must be `1`. |
| `price` | Lead price (commissiom payment). |
| `addprice` | Add amount to existing lead price. |
| `currency` | Three-letter ISO code for the lead currency, by default `usd`. |
| `convert` | The ISO code of the currency to convert the payout amount to is not used by default. |
| `cost` | Lead cost (not commission!). |
| `addcost` | Add amount to existing lead cost. |
| `costcur` | Three-letter ISO code for the lead cost currency, by default `usd`. |
| `costconv` | The ISO code of the currency to convert the cost to is not used by default. |
| `name` | Customer's name. |
| `phone` | Customer's phone (without validation and correction). |
| `email` | Customer's email (without validation and correction). |
| `comment` | Custom comment to the lead. |
| `meta[xxx]` | Custom lead field with name `xxx`. |
| `utm_source` | Tag **UTM Source** to replace the one received from the click. |
| `utm_campaign` | Tag **UTM Campaign** to replace the one received from the click. |
| `utm_content` | Tag **UTM Content** to replace the one received from the click. |
| `utm_term` | Tag **UTM Term** to replace the one received from the click. |
| `utm_medium` | Tag **UTM Medium** to replace the one received from the click. |
| `subid` | Tag **SubID** to replace the one received from the click. |
| `customer` | Tag **Customer** to track the customer or player IDs. |
| `adid` | Ad click ID to replace the one received from the click. |
| `data1` | Custom **data 1** to replace the one received from the click. |
| `data2` | Custom **data 2** to replace the one received from the click. |
| `data3` | Custom **data 3** to replace the one received from the click. |
| `data4` | Custom **data 4** to replace the one received from the click. |
| `data5` | Custom **data 5** to replace the one received from the click. |
| `stn` | Status or list of statuses that are recognized as **new** lead. |
| `sth` | Status or list of statuses that are recognized as lead in **hold**. |
| `sta` | Status or list of statuses that are recognized as **approved** lead. |
| `stc` | Status or list of statuses that are recognized as **cancelled** lead. |
| `stt` | Status or list of statuses that are recognized as **trash**. |
| `ip` | Lead IP address. |
| `geo` | Two-letter ISO code of lead country. |
| `ua` | User-Agent of the lead to determine the OS and browser. |

The result of the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of success, `error` in case of an error. |
| `error` | Error code for status `error` (see below) |
| `id` | The numeric ID of the lead that the action was taken on. |
| `create` | If a new lead was created - `true`, if an existing one was changed - `false`. |
| `stage` | The status the lead is in: `new`, `hold`, `approve`, `cancel`, `trash`. |

Error codes:

- `access` - the specified click or flow does not belong to you.
- `bad-click` - no click found with this ID.
- `bad-flow` - no flow found with this ID.
- `db` - a system error occurred while working with the database.
- `no-changes` - no changes were required, the lead is already in the right state.
- `no-click` - no click ID or flow specified. |

**Example of a successful server response:**

```json
{
    "status": "ok",
    "id": 1337,
    "create": true,
    "stage": "new"
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "no-changes"
}
```

<a id="spend"></a>
## Spend import

**URL for user**: `https://domain.com/api/tracker/spend.json?id={token}`

**URL for team**: `https://domain.com/api/gangs/spend.json?id={token}`

The function sets the click price for traffic according to the specified conditions. Allows you to use the ROI calculation functionality. Data can be transmitted in both GET and POST requests. The request must contain the `cpc` or `cost` parameters and at least one condition (dates, flow, UTM tags).

The function can take the following parameters as input:

| Field | Description |
| --- | --- |
| `cost` | The total price for all clicks during this period. |
| `cpc` | Price per one click. |
| `from` | The start date of the period in the format `YYYY-MM-DD`, Unix Timestamp, or any other format recognized by the `strtotime` function. |
| `to` | The end date of the period in the format `YYYY-MM-DD`, Unix Timestamp, or any other format recognized by the `strtotime` function. |
| `flow` | Character identifier of the flow. You can pass an array of identifiers. |
| `way` | Numeric identifier of the way within the flow. You can pass an array of identifiers. |
| `currency` | Three-letter ISO code for the click currency, by default `usd`. |
| `geo` | Two-letter ISO code of click counrty. |
| `domain` | The domain where click appeared. |
| `utm_source` | The **UTM Source** tag by which the click was registered. |
| `utm_campaign` | The **UTM Campaign** tag by which the click was registered. |
| `utm_content` | The **UTM Content** tag by which the click was registered. |
| `utm_term` | The **UTM Term** tag by which the click was registered. |
| `utm_medium` | The **UTM Medium** tag by which the click was registered. |
| `subid` | The **SubID** tag by which the click was registered. |
| `customer` | The **Customer** tag by which the click was registered. |
| `valid` | Set the price only for valid clicks: targeted clicks that pass the filter. |
| `unique` | Set the price only for unique clicks. |

You can also simultaneously update several price tags with one request; to do this, combine all requests into a `batch` array and send it via POST, for example:

```json
{
    "batch": [
        {
            "flow": "face",
            "cost": 1984,
            "currency": "usd"
        },
        {
            "from": "2020-04-04 00:00:00",
            "to": "2020-04-07 23:59:59",
            "cpc": 1.337,
            "currency": "eur"
        }
    ]
}
```

The result of the function is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: `ok` in case of success, `error` in case of an error. |
| `error` | Error code for status `error` with value from the list:
| `ok` | Number of successful price setting requests. Usually contains the response `ok=1` |
| `bad` | Number of unsuccessful price setting requests. |

Error codes: 

- `empty` - no data to put into price tags.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "ok": 1,
    "bad": 0
}
```

**Example of a server response with an error:**

```json
{
    "status": "error",
    "error": "empty"
}
```

<a id="templates"></a>
## Postback templates

**URL**: `https://domain.com/api/tracker/templates.json?id={token}`

The function returns the reference book of **postback templates** — presets for CPA networks and affiliate programs. Each template holds the offer URL and ready-made postback query strings that the partner network should call your tracker back with, one per conversion event. These are the same presets shown in the flow editor's "template" selector. The function takes no parameters.

The list combines the built-in reference book (synced from the AlterCPA network) with your own templates created in the tracker settings; both share the same set of fields.

The result is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: always `ok`. |
| `data` | Object of templates keyed by numeric template ID (see below). |

Each template in `data` is an associative array:

| Field | Description |
| --- | --- |
| `name` | Display name of the CPA network / affiliate program. |
| `url` | Offer or landing URL of the template. |
| `param` | Array of placeholder names (`[[name]]`) used inside the postback query strings. |
| `pb` | **Global** postback query string. |
| `pbr` | **Registration** postback query string. |
| `pbd` | **Deposit** postback query string. |
| `pbs` | **RevShare payment** postback query string. |
| `pbn` | **New lead** postback query string. |
| `pbh` | **Hold** postback query string. |
| `pba` | **Approve** postback query string. |
| `pbc` | **Cancel** postback query string. |
| `pbt` | **Trash** postback query string. |

Every `pb*` value is a query string without the leading `?`. To build the final URL you hand to the partner network, append it to your postback base: `{postback}?{pb*}`, substituting the `[[placeholder]]` macros listed in `param`. Only the fields that make sense for a given template are present — a simple template may carry just `pb`.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "12": {
            "name": "Acme CPA",
            "url": "https://acme.example/offer?click=[[click]]",
            "param": [ "click", "status" ],
            "pb": "click=[[click]]&event=global",
            "pba": "click=[[click]]&event=approve",
            "pbc": "click=[[click]]&event=cancel"
        },
        "10001": {
            "name": "My own network",
            "url": "https://my.example/?c=[[click]]",
            "param": [ "click" ],
            "pb": "c=[[click]]"
        }
    }
}
```

<a id="networks"></a>
## Traffic sources

**URL**: `https://domain.com/api/tracker/networks.json?id={token}`

The function returns the reference book of **traffic sources** (ad networks). Each preset pre-fills a flow's UTM/SubID/data tags, the default click cost and currency, and the outbound postback URLs the tracker fires back to the ad network on each lead-status change. These are the same presets shown in the flow editor's "source" selector. The function takes no parameters.

The list combines the built-in reference book (synced from the AlterCPA network) with your own sources created in the tracker settings; both share the same set of fields.

The result is an associative array:

| Field | Description |
| --- | --- |
| `status` | The result of the operation: always `ok`. |
| `data` | Object of traffic sources keyed by numeric source ID (see below). |

Each source in `data` is an associative array:

| Field | Description |
| --- | --- |
| `name` | Display name of the traffic source / ad network. |
| `utms` | **UTM Source** value template. |
| `utmc` | **UTM Campaign** value template. |
| `utmn` | **UTM Content** value template. |
| `utmt` | **UTM Term** value template. |
| `utmm` | **UTM Medium** value template. |
| `subid` | **SubID** value template. |
| `data1`–`data5` | Custom data-slot value templates. |
| `hide1`–`hide6` | Hidden data-field value templates. |
| `cpc` | Default click cost. |
| `cur` | Default currency, by default `usd`. |
| `pbglobal` | **Global** outbound postback URL. |
| `pbnew` | Postback URL fired on a **new** lead. |
| `pbhold` | Postback URL fired on **hold**. |
| `pbapprove` | Postback URL fired on **approve**. |
| `pbcancel` | Postback URL fired on **cancel**. |
| `pbtrash` | Postback URL fired on **trash**. |
| `param` | Array of placeholder names (`[[name]]`) used inside the value templates. |

All value templates may contain `[[placeholder]]` macros; the tracker substitutes them when the source is applied to a flow. Only the fields configured for a given source are present.

**Example of a successful server response:**

```json
{
    "status": "ok",
    "data": {
        "5": {
            "name": "Facebook Ads",
            "utms": "[[campaign]]",
            "subid": "[[adset]]",
            "cpc": "0.15",
            "cur": "usd",
            "pbglobal": "https://graph.facebook.com/...",
            "pbapprove": "https://graph.facebook.com/...?ev=Purchase",
            "param": [ "campaign", "adset" ]
        },
        "10002": {
            "name": "My own source",
            "cpc": "0.1",
            "cur": "usd"
        }
    }
}
```
