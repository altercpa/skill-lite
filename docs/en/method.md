# Traffic processing methods

The tracker offers several methods for working with links and files. They work the same regardless of how you use the tracker: when working with links to parked domains, when working with site storage, or when using local files.

## Available working methods

These methods work equally well within ways as well as with trafficback. If no site is selected from the repository, the link and one of the methods for displaying it are used. If a site is selected, the "Show site from storage" method is used.

<a id="http302"></a>

### Redirect by URL (302 redirect)

The visitor is redirected to the specified link using the `302 Found` header. This is the standard and most common way to work with redirects. Macros are automatically replaced in the URL.

> It is not recommended to use 301 and 302 redirects in combination with other methods, as some ad networks may consider different delivery methods as site hacking.

<a id="http301"></a>

### Redirect by URL and glue (301 redirect)

The visitor is redirected to the specified link using the `301 Moved Permanently` header. This kind of redirect is cached even if rotation is enabled in the stream settings. Macros are automatically replaced in the URL.

> There is a belief that this redirect method allows you to glue your link to the target domain and the target domain will be shown in advertisements. We don't recommend believing these rumors.

<a id="redirect"></a>

### Redirect via separate page (meta redirect)

The user sees the "Your browser is being checked" page and is redirected to the target site three seconds after it is displayed. In this case, the browser does not receive direct 301 or 302 redirect headers, all redirects are performed using HTML. Macros are automatically replaced in the URL.

> This redirect option is recommended if you're combining different methods of working, such as showing content for some paths and redirecting for others. This will avoid blocking an advertising campaign, for example, in Google with "Malware" mark.

<a id="jsgo"></a>

### Redirect via script (JS redirect)

The user loads a page without content and is immediately redirected to the target site. In this case, the browser does not receive direct 301 or 302 redirect headers, all redirects are performed using JavaScript. Macros are automatically replaced in the URL.

> This option can be used along with meta redirect to avoid blocking for "Hacked site". But this method may sometimes not work due to ad blockers in browsers.

<a id="iframe"></a>

### Show the URL in IFrame

The user sees a page with no content, on top of which an iframe opens in full screen with the site specified by the link. Macros are automatically replaced in the URL.

> Not all sites support iframes due to security restrictions. When working with an iframe, it is recommended to open all links in a new window by adding the `target="_blank"` attribute to them, or allow displaying all sites in frames using titles.

<a id="curl"></a>

### Show the URL contents (curl)

The script loads content from by the link and shows it to the user. The `base` tag is added to the content, which replaces all paths to resources. So all resources (pictures, scripts, styles) are displayed on the site without the need to change its content. GET and POST requests are supported. Macros are automatically replaced both in the URL and the content.

> There is a belief that the presence of the `base` tag in the content of the site negatively affects the approval of the site by search engines. We do not recommend listening to this nonsense. Displaying content via a link is the best way to work!

<a id="proxy"></a>

### Show the URL contents with no changes

The script loads content from by the link without changing paths and shows it to the user. Macros are automatically replaced both in the URL and the content.

> When working with this technique, you must specify absolute paths with a domain for all resources: images, scripts, styles, fonts, otherwise they will not be loaded. The `action` fields in forms must also have absolute paths.

<a id="local"></a>

### Open local file

Instead of a link, set the relative path to the file that will be loaded and display by the script. The file must be in PHP or HTML format. Macros are automatically replaced in the contents of the file. When using files inside folders, the `base` tag will not be added automatically, don't forget to add it manually!

> This method will only work when downloading the client file from the flow or using your own site store. Embedded TDS doesn't support this method because it is not possible to upload the file to the central server.

<a id="site"></a>

### Show site from storage

If you select a site in the path or traffic-back settings, it will be used instead of a link. Sites are located in your local storage. The script will show the file `index.html` or `index.php` from the site folder. Macros are automatically replaced in the contents of the file.

> A local site can contain many files, forms, scripts, thank you pages, and any other content. Some affiliate networks provide ready-made archives for uploading to the tracker.

<a id="error"></a>

### Show error page

Instead of link content, a standard NGinx error page is shown. The specific error is chosen randomly and depends on the visitor's IP address.

> You can ignore the random selection of the error code and set your own code. To do this, enter the HTTP error code in the URL field. The most common option is to simply type in `404`.

<a id="blank"></a>

### Show blank page

Instead of linked content, a blank page with no content is displayed. Not even standard HTML tags for empty markup are sent. The response status is always `200`.

> It is assumed that you use this option only for traffic back and traffic filters. If you want to use this option in paths, specify something in the link, as long as there is some text.

<a id="macros"></a>

## Automatic content replacement

The tracker automatically replaces the link text or content of the downloaded site. The following macros are available to you:

- `{click}` - click ID.
- `{flow}` - flow ID.
- `{host}` - the domain on which the transition was made.
- `{geo}` - two-letter ISO-code of visitor's country.
- `{adid}` - advertising campaign click identifier (`fbclid`, `gclid`, `adclid`, `clid`, `adid`).
- `{data1}` - arbitrary data from the `data1` or `d1` parameter.
- `{data2}` - arbitrary data from the `data2` or `d2` parameter.
- `{subid}` - the value of the `subid` tag.
- `{customer}` - the value of the Customer tag.
- `{utm_source}` - UTM Source tag value.
- `{utm_campaign}` - UTM Campaign tag value.
- `{utm_content}` - UTM Content tag value.
- `{utm_term}` - UTM Term tag value.
- `{utm_medium}` - UTM Medium tag value.
- `{next}` - full link of the second step (landing) with the domain and flow specified.
- `{local}` - relative local link of the second step without the domain and path specified.
- `{direct}` - direct link of the second step without redirect.
- `{esub}` - encrypted SubID tag.
- `{eutms}` - encrypted UTM Source tag.
- `{eutmc}` - encrypted UTM Campaign tag.
- `{eutmn}` - encrypted UTM Content tag.
- `{eutmt}` - encrypted UTM Term tag.
- `{eutmm}` - encrypted UTM Medium tag.
- `{eutmu}` - encrypted Customer tag.
- `{ed1}` - encrypted additional data field 1.
- `{ed2}` - encrypted additional data field 2.

For redirects and iframes, the data is replaced in the link. For content requests, the data is replaced both in the link and content. For sites and local files, the data is replaced only in the content.
