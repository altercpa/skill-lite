# API interface

The filter and tracker APIs allow you to work with visit checking, traffic and statistics. To get an API token, go to your [profile](/profile) page in the system. Interaction with the service is carried out via the HTTP protocol. A successful response has status 200, and an error is returned with the matching HTTP status (for example 400, 401, 402, 403, 404 or 500) and a JSON body with the `status` and `error` fields. The standard request format is GET or "pure" POST (`form-urlencoded`). When submitting in JSON, don't forget the `Content-type: application/json` header. The result format is JSON. There is no limit on the number of requests.

Futher in the text, important parameters are shown in curly brackets:

- `{token}` - your API key from [personal account](/profile), for example `123-abdce`.
- `{key}` - your postback key from [any flow](/track-flows), for example `666c93ba7cebe113`.
- `{flow}` - symbolic identifier of a flow from the list, for example `ef1c`.
- `{click}` - character ID of a click from the tracker, for example `ca1337`.

**Important!** When using links and data, don't use curly braces, only the values themselves!

> The tracker functions can work for both the webmaster and the team. For regular work, use the URL for yser. For team-level work, use the URL for team with the key of any of the team leads.

Most of the API can be run in either user, team, or administrator space. By default, work is performed in user space. To specify the workspace, use the `scope` field:

- `user` - standard request, run as the user, accessible to all users.
- `team` or `gang` - run as the team leader, accessible only to team leaders.
- `admin` - run as the tracker administrator, accessible only to administrators.

Some functions have separate URLs for team work. Using the `scope` parameter is not required for these functions. The documentation provides sample URLs for different access levels, if supported.

::: card
## Filtering

  1. [Check the visit](api-filter.md#check)
  2. [Campaign setup](api-filter.md#filterset)
  3. [Campaigns list](api-filter.md#filterlist)
  4. [Get campaign info](api-filter.md#filterget)
  5. [Delete campaign](api-filter.md#filterdel)
  6. [Download filtering file](api-filter.md#filtermake)
  7. [Block IP or header](api-filter.md#filterblock)
  8. [Postback for mini-tracker](api-filter.md#filterpostback)
  9. [Visit log](api-filter.md#filterlog)
  10. [Visit statistics](api-filter.md#filterstats)
:::

::: card
## Blacklists

  1. [Available lists](api-blacklist.md#blacklistavail)
  2. [List of blacklists](api-blacklist.md#blacklistlist)
  3. [List entries](api-blacklist.md#blacklistshow)
  4. [Create a list](api-blacklist.md#blacklistadd)
  5. [Edit a list](api-blacklist.md#blacklistedit)
  6. [Delete a list](api-blacklist.md#blacklistdel)
  7. [Add an entry](api-blacklist.md#blacklistblock)
  8. [Remove an entry](api-blacklist.md#blacklistremove)
  9. [Change an entry](api-blacklist.md#blacklistchange)
  10. [Import entries](api-blacklist.md#blacklistimport)
  11. [Export entries](api-blacklist.md#blacklistexport)
:::

::: card
## Integration with tracker

  1. [Sending postback](api-tracker.md#postback)
  2. [Import spend statistics](api-tracker.md#spend)
  3. [Postback templates](api-tracker.md#templates)
  4. [Traffic sources](api-tracker.md#networks)
:::

::: card
## Tracker statistics

  1. [Leads list](api-stats.md#leads)
  2. [Click statistics](api-stats.md#stats)
  3. [Click list](api-stats.md#clicks)
:::

::: card
## Click API — click processing

1. [Register a click by flow code](api-click.md#tds)
2. [Register a click by domain](api-click.md#domain)
:::

::: card
## Site Storage

1. [Sites list](api-sites.md#siteslist)
2. [Upload site](api-sites.md#sitesadd)
3. [Edit site](api-sites.md#sitesedit)
4. [Delete site](api-sites.md#sitesdel)
5. [Download site](api-sites.md#sitesdownload)
:::

::: card
## Parked Domains

1. [Domains list](api-domains.md#domainslist)
2. [Add domain](api-domains.md#domainsadd)
3. [Check domain](api-domains.md#domainscheck)
4. [Delete domain](api-domains.md#domainsdel)
5. [Link domain to flow](api-domains.md#domainsglue)
6. [List confirmation files](api-domains.md#domainsfiles)
7. [Create confirmation file](api-domains.md#domainsconfirm)
8. [Change confirmation file](api-domains.md#domainschange)
9. [Remove confirmation file](api-domains.md#domainsremove)
:::

::: card
## Flows and Ways

1. [Flow list](api-flows.md#flowslist)
2. [Get flow](api-flows.md#flowsget)
3. [TDS client file](api-flows.md#flowsfile)
4. [Current way caps](api-flows.md#flowscaps)
5. [Create flow](api-flows.md#flowscreate)
6. [Clone flow](api-flows.md#flowsclone)
7. [Edit flow settings](api-flows.md#flowschange)
8. [Delete flow](api-flows.md#flowsremove)
9. [Add way to flow](api-flows.md#flowsadd)
10. [Copy way](api-flows.md#flowscopy)
11. [Edit way settings](api-flows.md#flowsedit)
12. [Delete way](api-flows.md#flowsdel)
:::

::: card
## Ad Spend Connectors

1. [Connector types](api-adv.md#advtypes)
2. [Spend-matching tag labels](api-adv.md#advtags)
3. [Integrations list](api-adv.md#advlist)
4. [Get integration edit form](api-adv.md#advform)
5. [Create integration](api-adv.md#advadd)
6. [Edit integration](api-adv.md#advedit)
7. [Force integration run](api-adv.md#advupdate)
8. [Delete integration](api-adv.md#advdel)
9. [Integration request log](api-adv.md#advlog)
:::
