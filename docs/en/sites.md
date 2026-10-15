# Site storage

Your own site storage greatly expands the possibilities of working with the tracker. It opens access to the use of local sites and takes care of the whole routine of domain parking and tracking. The use of storage significantly reduces the likelihood of advertising campaigns being banned in some networks.

::: card
## Site storage benefits

1. Download ready-made archives of sites from affiliate networks with all customized logic.
2. Use the same site in multiple flows or ways.
3. Collect separate statistics for each site from all flows at once.
4. Minimize the risk of blocking due to bad server neighbors.

[Jump to setup](/track-sites)
:::

<a id="setup"></a>

## Storage settings

Site storage is installed on a clean server running Debian 11 without a control panel. When choosing a server, you can [use our tips](server.md).

1. Go to the "[Sites](/track-sites/setup)" section and find the "Setup command" field in the form that opens. If you don't see the forms with this command, click on "Settings".
2. SSH into your server as user `root`. We recommend that you first follow the requirements in "[Installation tips](#tips)" below in this article.
3. Copy the install command from the form and run it on the server. Installation will take a few minutes.
4. At the end of the installation, the script will show you the data for accessing the server via FTP and the necessary storage settings. Save them, they will be useful to you in the future. If you did not have time to copy them, they are in the file `/root/config.txt`
5. Update the "Sites" page in the tracker without saving anything - the script will fill in all the data automatically, you don't need to specify anything. If you accidentally saved empty settings, just fill them in with the data from the previous step.
6. You can't just change this data after installation. If you change the user's API key or storage settings, you need to perform a couple of actions described below in the section "[Changing keys](#change)".

The storage is installed and ready to go. Your sites will be located in the `/var/www/web` folder. Storage domains are automatically parked and receive their SSL certificate.

After installing the storage, it is recommended to park some domain to it and specify it in the settings. It will be used to preview HTTPS sites instead of simply connecting by IP address.

::: card
<a id="sitesbreif"></a>

## Preparing websites briefly

1. Upload as a ZIP archive. The main files are in the root of the archive, and not in a subfolder, otherwise there will be a white screen. Call the main file `index.php` or `index.html`.
2. On the **landing** add `subid` in a hidden field with the value `{click}` like this:

<input type="hidden" name="sub1" value="{click}" />
3. On the **prelanding** change the links to the macro `{next}` like this:

<a href="{next}"> ... </a>
4. Surely clean up the site: replace #-links with JS scrolling, add pixels, check AJAX forms and API files sending the subid correctly.

Important!

The site will not work well by itself, it

necessarily

should be checked and prepared!
:::

<a id="sites"></a>

## Preparing websites

To create a new site, click on the "Upload site" button, enter its name and select the archive you plan to upload. It is not necessary to select an archive, without it the tracker will create an empty folder on the server, where you can upload the site manually via FTP.

The site archive must be uploaded in ZIP format. The root of the archive must contain the file `index.php` or `index.html`, which will be the main page of the site.

> **Common error**: a white screen or `403 Forbidden` error is displayed at the site address after loading. This means that your site was not located in the root of the archive, but was placed in a subfolder. Unzip the archive, go inside the folder with its contents, select all files and click "Compress" or "Add to archive". Such an archive will contain all files directly in the root directory, and not in a subfolder.

For tracking to work correctly, you need to add a click ID to all forms and links on your site. The macro `{click}` is responsible for it. For example, in a form, you need to create a hidden field like this:

```html
<input type="hidden" name="sub1" value="{click}" />
```

If the site is used as prelanding combined with the "next step", replace the links with the macro `{next}` like this:

```html
<a href="{next}"> ... </a>
```

The following macros are available for you to automatically replace on the site:

- `{click}` - character identifier of the click.
- `{flow}` - character identifier of the flow.
- `{user}` - user ID.
- `{geo}` - ISO code of the country of the click.
- `{query}` - the line of GET-parameters.
- `{next}` - the link to the second step of the flow.
- `{utm_source}` - UTM Source tag.
- `{utm_campaign}` - UTM Campaign tag.
- `{utm_content}` - UTM Content tag.
- `{utm_term}` - UTM Term tag.
- `{utm_medium}` - UTM Medium tag.
- `{host}` - domain.
- `{subid}` - SubID tag.
- `{customer}` - Customer tag.
- `{adid}` - click ID in the ad network.
- `{data1}` - custom data field 1.
- `{data2}` - custom data field 2.
- `{data3}` - custom data field 3.
- `{data4}` - custom data field 4.
- `{data5}` - custom data field 5.
- `{esub}` - encrypted SubID tag.
- `{eutms}` - encrypted UTM Source tag.
- `{eutmc}` - encrypted UTM Campaign tag.
- `{eutmn}` - encrypted UTM Content tag.
- `{eutmt}` - encrypted UTM Term tag.
- `{eutmm}` - encrypted UTM Medium tag.
- `{eutmu}` - encrypted Customer tag.
- `{ed1}` - encrypted custom data field 1.
- `{ed2}` - encrypted custom data field 2.
- `{ed3}` - encrypted custom data field 3.
- `{ed4}` - encrypted custom data field 4.
- `{ed5}` - encrypted custom data field 5.
- `{hide1}` - hidden flow or way data field 1.
- `{hide2}` - hidden flow or way data field 2.
- `{hide3}` - hidden flow or way data field 3.
- `{hide4}` - hidden flow or way data field 4.
- `{hide5}` - hidden flow or way data field 5.
- `{hide6}` - hidden flow or way data field 6.

The data is replaced after the execution of all PHP scripts of the site, so you just need to insert the macro itself into the site content and it will be filled with the desired value.

<a id="mirror"></a>

## Downloading sites

Sometimes you may need to mirror download an existing website from a link and manually configure it to work. You can start downloading using the link both at the time of creating the site and when editing it.

> **Important!** The server will try to download the entire site. It will not limit itself to the one page you give it. Under no circumstances should you provide links to large sites, such as news resources.

In the form for creating or editing a site, provide a link to the target site. Ideally, this link should not contain redirects and lead directly to the resource you want to download. The mirroring process may take up to 10 minutes.

<a id="tips"></a>

## Installation tips

It is highly recommended that you update your system before starting the installation. Yes, this is built into the script itself, but some vendors provide poor quality builds of Debian 11, which can fail when installing an update.

To update the system, run:

```
apt update -y && apt upgrade -y
```

Most likely, the script will work, showing a few messages about successful installation, and will not require any additional actions. But a number of problems are possible:

- When updating, the version of the system changes a lot. The OS will simply notify you of this and ask you to press `Y` to confirm your consent. There are still no other options.
- New software versions want to overwrite existing configuration files. It is recommended to leave the original configs and choose the answer `N`.
- Sometimes the server needs to update the GRUB bootloader (hello [Inferno](https://cp.inferno.name/aff.php?aff=3858)). When updating, you need to select the disk on which to install it. Always choose the disc with the shortest title. For example, if you are prompted with `/dev/sda` and `/dev/sda1`, your choice is `/dev/sda`. Most often, it comes first in the list.

<a id="change"></a>

## Changing keys

After installing the storage, your API key is indicated in its settings. If you change the API key in the profile settings or the storage key in the sites settings, you must manually make the same changes on the storage server.

- The user's API key from the "[Profile](/profile)" section is used to download a list of domains. If you reset the API key in your profile, you need to replace it in the store in the file `/root/webssl` as user `root` (for example, `nano /root/webssl`). The key is indicated in the link in the second line of the file.
- The storage key is used to work with sites - download new ones, delete or change existing ones. It is listed in the settings in the "[Sites](/track-sites/setup)" section. If you change the key in the settings, change it also in the file `/var/www/web/index.php` on the third line.

> The storage key and the user's API key may not match, this is normal. Just make sure that the domain upload contains the user's API key, and that the storage settings are as in the `index.php` file.
