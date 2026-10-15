# Teamwork

The tracker allows users to independently organize themselves into arbitration teams. Within a team, the leader has access to shared statistics for all participants and spend analytics for the entire team at once.

::: card
## The true teamwork

The tracker offers extremely limited tools for the work of an arbitration team. For serious team automation, we recommend using other products from our company - [AlterCPA Pro](https://www.altercpa.pro/en/altercpa) or [Cloud](https://www.altercpa.pro/en/cloud). They have a number of important advantages:

1. Full-featured offers with a ready-made set of landing pages and pre-landing pages, flexible settings for access and working conditions.
2. Connection to advertisers and affiliate networks is carried out at the system level. The webmaster does not have any access to personal accounts or affiliate network accounts.
3. Commissions and payments are set at the offer level, the real amounts of earnings are not visible to your employees.
4. Finance accounting is carried out automatically, each employee has his own personal account with his earnings and detailed analytics.
5. The entire team can be divided into departments (internal mini-teams) with their own responsible managers and special access to offers.

Using [AlterCPA Pro](https://www.altercpa.pro/en/altercpa) or [Cloud](https://www.altercpa.pro/en/cloud) you can create your own small affiliate program just for your own arbitration team and automate all work with leads and traffic.

[Choose your AlterCPA](https://www.altercpa.pro/ru/promo)
:::

## How to join a team?

Ask your leader to send you a special join link. Open it and click "Join". Before doing this, be sure to log into the tracker using your username and password.

> **Important!** From the moment you join the team, all your clicks and leads will be available to the team management. Any traffic that you uploaded before joining the team or after leaving it is available only to you.

## Creating an arbitration team

A team can be created through your tracker admin by providing a list of users and a leader, or independently. Go to the "[Gang](/gang)" section in the navigation menu, enter the name of your future team and click "Create".

Once created, you can invite users to join your arbitration team. To do this, send them a login link to join the team.

> **Important!** Team statistics are counted from the moment the user joins the team and cease to be counted the moment he leaves. All leads and clicks that the user had before joining the team or after he left are not available to team leads. All traffic that the user received while working in the team remains always available.

It is recommended to customize your team by providing a short description and opening the relevant sections.

- **Leaders.** A team can have several team leads. They all have the same access rights to user statistics and team management. Including, they can remove other leaders from their positions.
- **Team name.**Used only for beauty on the team page.
- **Description** Displayed on the team page. It is recommended to indicate here the rules of work, contacts of managers and financiers, links to important internal resources and similar information.
- **Show main page.** If you uncheck this checkbox, the description and member list page will be hidden from regular team members.
- **Show list of participants.** If you uncheck this box, only the description specified a couple of points above will be shown on the team page.
- **Sharing statistics.** Statistics sections of all team members become available to all participants. This can be useful in training teams to collectively analyze specific cases and errors.

If necessary, you can also change the login link to make old submitted links unavailable. You can change the link at any time after participants join.

## Blacklists

When working as part of an arbitration team, you can use shared team blacklists. By default, only team leads have access to edit them, but you can enable shared access for the entire team in the settings.

Active team lists are available for selection by all team members. Default lists are automatically activated for the entire team, just like system lists, and available both in campaigns and tracker.

[Blacklist documentation](blacklist.md)

## Statistics and analytics

Team leaders gain access to team analytics. It is presented in three sections: leads, clicks and statistics. Each section copies the corresponding section of personal statistics.

An additional field is added to the filters of these sections: user. You can select a specific team member and filter statistics by him. Statistics for a specific user can also be opened from the list of participants. To do this, click on the statistics button and select the section you are interested in.

[Statistics documentation](stats.md)

## Teammates' flows

Team leaders can view and customize their users' flows. The team flows section only shows flows that the user has created while working in the team. To create a new flow, first select the user in the search field. To create a flow, the user or team must have the appropriate limits available.

[Flows documentation](flows.md)

## Loading spend statistics

Team leaders can add team-wide spend downloads. This is convenient if only the leader has access to financial statistics. All operating principles are identical to individual downloadings.

[Spend analytics documentation](spend.md)

## Sites storage

The team can work with a shared site storage for all members. It is configured in the "Team - Sites" section similar to a personal storage. If you want to make your personal storage shared by the entire team, just copy its settings.
Useful features of the team site storage:

The team storage replaces personal user storages. Any team member will be able to add and edit sites, park domains to the new storage. An unpleasant detail: if a member had their own personal storage, access to it will be lost.
Team leads can add sites shared by all. This allows you to prepare the same offers for all team members. Ordinary members will be able to select sites in the ways of flows, view statistics on them, but will not be able to edit them.
Team leads can create sites for specific team members. This allows the team's technical specialists to create personal landing pages, pre-landing pages, and white sites. Like shared sites, they are available in flows but are not editable by members.

**Important!** User sites within the storage are not physically isolated from each other and are stored in a shared folder. Do not use the team storage to work with sensitive data, because all team members have access to upload sites with PHP files.
[Sites documentation](sites.md)

## API

Team leaders can use the API for the entire team. This allows you to get lead and click statistics, send spend statistics to the entire team. But most importantly, it allows you to set up a single postback on the advertiser's side, which will work with the entire team's flows.

[API documentation](api.md)
