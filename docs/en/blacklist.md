# Blacklists

We provide ready-made blacklists of suspicious addresses and headers. You can create your own blacklists and use them in combination with the system ones or instead of them. Suspicious bots and requests with incomplete headers are always blocked at the system level.

::: card
<a id="filter"></a>
## System black lists

There are a number of system filters available to you that you can use on your sites.

- **AlterCPA Lite** of types **IPv4**, **IPv6** and **Net** - static blacklists of addresses of proxy servers and suspicious providers. They are generated several times per hour based on scanning of World Wide Web addresses and blacklists from our suppliers.
- **Smart** of types **IPv4**, **IPv6** and **Net** - smart blacklists, which are automatically generated in real time based on incoming traffic from all sites of the service.

For optimal protection of your site from spurious traffic, we recommend using all available filters, that is, do not configure the "Filters" field at all.
:::

## Personal blacklists

You can create your own blacklists in the "[My filters](/filter-bl)" section. To create a new blacklist, give it a name and select a type:

- **IP addresses** - used for single IPv4 address blocking.
- **IP subnets** - used to block IPv4 address ranges.
- **IPv6 addresses** - used to block IPv6 subnets and addresses.
- **Headers** - used to block stop words in request headers.

Blacklists are divided by type to improve performance. You cannot add an IPv6 address to a classic IPv4 list. After creating a list, you need to fill it in and activate it:

- **Status: active or hidden** - determines whether the list will be available for use at all. Hidden lists are not displayed in the filter selection field.
- **Default** - the list will be automatically active in all campaigns that do not have any filters selected. Default lists are also active in the tracker.

Most often, you will need lists with both checkboxes - active and default. Do not forget to enable them.

After creating a blacklist, you need to fill it with addresses. Open the blacklist by name or by clicking the "View" button. Above the table there will be a form for adding a new element:

- **IP address** - a single address or the address of the subnet you want to block. It is the starting address that is specified, not the range.
- **Size** - for IPv6 addresses and IP subnets, you need to specify the size. The standard size of an IPv4 subnet is 24, and for an IPv6 subnet - 64 (it is not recommended to specify more, this is optimal).
- **Comment** - a small note to choose from, which will help you understand in the future why you blocked this or that address. Selected from the list.

Header blacklists have slightly different fields than address lists:

- **Text** - the keyword by which blocking is performed. If the selected header contains this word, the visit will be blocked. Can be a regular expression.
- **Header** - select from the drop-down list which header we should check.
- **Comment** - used for convenience, just like the mark in the address list.
- **Magic checkbox without signatures** - check it so that blocking works in regular expression mode, and not by exact match.

After adding an item to the blacklist, it immediately starts working.

> **Important!** Working with blacklists requires some technical knowledge. You need to understand what subnet size is in order to block addresses. You need to understand how regular expressions work in order to block headers.

## Teamwork

When working as part of an arbitration team, you can use shared team blacklists. By default, only team leads have access to edit them, but you can enable shared access for the entire team in the settings.

Active team lists are available for selection by all team members. Default lists are automatically activated for the entire team, just like system lists.

## Import and export black lists

You can export and import black lists in CSV format. File encoding - UTF-8, separator - commas, framing values - quotes. Each of the types of black lists works with its format. All lists use the same comments identifiers:

1. Public list
2. Suspicious
3. Corporate
4. Different User-Agent
5. GEO error
6. Header error
7. Multiple errors
8. Bad request
9. Bot

### IPv4 addresses

The line of this list contains two fields: IP address and comment identifier.

```
12.34.56.78,2
```

### IPv4 subnets

The line of this list contains three fields: IP address, subnet capacity and comment identifier.

```
12.34.56.0,24,3
```

### IPv6 addresses and subters

All IPv6 addresses are added as subnets. It is recommended to use subnet / 64. The address may be indicated both in full and in abbreviated form. The line of this list contains three fields: IPv6 address, subnet capacity and comment ID.

```
2001:abcd::1,64,5
```

### Headers filter

The line of this list contains three fields: the contents of the header, the header ID and the comment identifier.

```
"facebook",1,6
```

The header identifier can take one of the following values:

- `1` - User-Agent
- `2` - Accept
- `3` - Accept-Charset
- `4` - Accept-Encoding
- `5` - Accept-Language
- `6` - Authorization
- `7` - Cache-Control
- `8` - Connection
- `9` - Content-Disposition
- `10` - Date
- `11` - Expect
- `12` - From
- `13` - Host
- `14` - If-Match
- `15` - If-Modified-Since
- `16` - If-None-Match
- `17` - If-Range
- `18` - If-Unmodified-Since
- `19` - Max-Forwards
- `20` - Pragma
- `21` - Proxy-Authorization
- `22` - Range
- `23` - Referer
- `24` - TE
- `25` - Trailer
- `26` - Transfer-Encoding
- `27` - Upgrade
- `28` - Via
- `30` - X-Forwarded-For
- `31` - X-Purpose
- `32` - X-FB-HTTP-Engine
- `33` - X-Frame-Options
- `34` - X-WAP-Profile
- `35` - X-Real-IP
- `36` - Client-IP
- `37` - CF-Connecting-IP
- `100` - Remote Address
- `101` - Remote Port
- `102` - Request Method
- `103` - Request URI
- `104` - Request Query String
- `106` - Request Scheme
