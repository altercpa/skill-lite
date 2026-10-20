# Built-in tracker

Our traffic filtering service allows you to track the success of your filters and actions on target sites. Two tracking options are available to you - a simplified mini-tracker inside the campaigns themselves and a powerful tracker tracker in a separate section.

## Mini-tracker

Each filtering campaign can count the number of clicks and leads that have passed through it. This is the simplest counter that does not take into account changes in leads, does not save their data, only counting the number.

The following columns are used in the filter section:

- **Total number of visits** that were processed by the filter.
- **Number of targeted visits**, used for further calculations.
- **Number of junk visits** rejected by the filter.
- **Total number of leads** or actions on the target site.
- **Number of approved leads** or successful actions.
- **Number of rejected leads** or failed actions.
- **Total earnings** on approved leads.
- **CR**: conversion - the ratio of total leads to target visits.
- **EPC**: earnings per click - the ratio of earnings to target visits.
- **Approve**: The ratio of the number of approved leads to their total number.

In order to connect the mini-tracker to the affiliate network, you need to set up postbacks for lead creation, lead approval and lead rejection. Ready-made postbacks can be found by clicking on the "Settings" of your filtering campaign and scrolling to the bottom of the page.

The postback function has three parameters:

- `uid` - ID of your filtering campaign.
- `status` - lead status (`new`, `approve`, `cancel`).
- `payout` - earnings per lead (taken into account only for `approve` status).

> **Very important!** These postbacks can't handle duplicates or incorrect submissions. Each time a postback is called, it increments the counter for the number of leads in the selected status. Add postbacks only to those statuses on the side of the affiliate network, which will not be sent several times.

## Powerful tracker

You can use the full AlterCPA Lite tracker right inside our traffic filtering service. It is located in the "Tracker" section of the main menu:

- **[Flows](/track-flows)**: this is where you set up your tracking campaigns and split tests.
- **[Leads](/track-leads)**: this section collects data about your leads.
- **[Statistics](/track-stats)**: this section collects summary statistics for leads and clicks.
- **[Domains](/track-domain)**: this tool parks domains for the traffic.
- **[Sites](/track-sites)**: this section enables working with local sites.

The tracker provides you with full-featured traffic tools:

- Built-in simple mechanisms of [traffic filtering](filter.md) and the ability to connect flows to full-featured filtering campaigns with deep visit analytics.
- Smart [split testing](split.md) with no limit on the number of ways and flexible visit filtering settings.
- Extended [statistics](stats.md) with support for goals and UTM tags analytics.
- Parking [domains](domain.md) to work with traffic with the ability to link flows to the main page.
- Various [methods](method.md) of working with links, including proxy requests and the ability to work as a local file on your hosting.
- Support for [local sites](sites.md) in your own repository.
