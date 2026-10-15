# Tracking via pixels

Conversions in the AlterCPA Lite tracker can be tracked both through [server-side postbacks](postback.md) and also through client-side pixels. This allows you to install the tracking code on regular HTML sites, such as embedding on "Thanks" pages.

## How does a pixel work?

When you click on your tracking link, a click ID is generated. This identifier is stored in a cookie of the user's browser associated with your flow. A visit by a user with this ID to a page with a pixel will generate a conversion tied to that click.

> **Very important!** This tracking option is unsafe in terms of cheating. Anyone can find the address of your pixel by placing a test order and later catch up with visitors, get click IDs and create false conversions. It is not recommended to use this approach where high accuracy and security of tracking is required.

## How is a pixel inserted?

Requests to a pixel can be made by inserting an image or via AJAX. The simplest option would be to insert a pixel image into the page code. A sample of such an insert is shown in the flow settings.

![Postback settings](assets/img/en-postback.png)

The domain hosting the pixel must match the domain that was navigated to. In case of successful operation of the pixel, the response is given the status `200`, in case of an error - the status `418`.

## General link format

A link for a pixel consists of a domain, a flow code, and a pixel format:

```
https://domain.com/code/pixel.gif
```

Where `domain.com` is your domain, `code` is the character code of the flow, and `pixel` is the pixel type: `pixel` for lead or `success` for approval, format: `json`, `png` or `gif`

## Inserting image

Pixel for creation of new lead in GIF format:

```html
<img src="https://track.altercpa.one/abcd/pixel.gif" alt="" width="1" height="1" />
```

Pixel for creation of new lead in PNG format:

```html
<img src="https://track.altercpa.one/abcd/pixel.png" alt="" width="1" height="1" />
```

Pixel for creation of approved lead in GIF format:

```html
<img src="https://track.altercpa.one/abcd/success.gif" alt="" width="1" height="1" />
```

Pixel for creation of approved lead in PNG format:

```html
<img src="https://track.altercpa.one/abcd/success.png" alt="" width="1" height="1" />
```

## Requesting via JavaScript

Link to create new lead:

```
https://track.altercpa.one/abcd/pixel.json
```

Link to create approved lead:

```
https://track.altercpa.one/abcd/success.json
```

The responses of the script are the same as the responses of the standard [API postback requests](api-tracker.md#postback).

## Pixel parameters

In a pixel link, you can immediately specify additional parameters that will change its behavior. Parameters are specified after the question mark, as in a regular link:

```
https://track.altercpa.one/abcd/pixel.gif?status=approve&goal=dep
```

Available parameters:

- `status` - lead status: `new`, `hold`, `approve`, `cancel` , `trash`. Using the `success` pixel is the same as adding `status=approve`.
- `goal` - character identifier of the goal.
- `uid` - unique lead ID on the advertiser's side.
- `price` - payout amount per lead.
- `currency` - three-letter ISO code of the currency in which the payment is made.
- `name`, `phone`, `email`, `comment`, `geo`, `ip` - lead data.
- `utm_source`, `utm_campaign`, `utm_content`, `utm_term`, `utm_medium`, `subid`, `domain`, `adid`, `data1`, `data2` - lead tags that replace those received from click.

Parameters are always passed only in a GET request. The purpose and composition of the parameters is the same as [sending a regular postback](api-tracker.md#postback).
