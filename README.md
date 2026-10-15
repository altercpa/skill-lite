# AlterCPA Lite API — a skill for Claude

**Give your AI assistant the complete AlterCPA Lite API: every endpoint, parameter and error code, with OpenAPI specs.**

[![License: MIT](https://img.shields.io/github/license/altercpa/skill-lite)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/altercpa/skill-lite)](https://github.com/altercpa/skill-lite/releases/latest)

**English** · [Русский](README.ru.md)

![Purr the cat runs a brass switchboard that sorts glowing click-drops into lanes — cover illustration for the AlterCPA Lite API skill](.github/assets/cover.jpg)

## What is it

[AlterCPA Lite](https://www.altercpa.pro) is a self-hosted conversion tracker for affiliate marketers.
This is a [Claude](https://claude.com) **skill** that teaches Claude its HTTP API. Ask for an
integration, a script or a report in plain words — Claude writes calls with the right endpoints,
parameters and error handling instead of guessing.

## What it covers

| Area | What you can do |
|---|---|
| **Flows, sites, domains** | tracking campaigns, site setup, parked domains |
| **Filtering and cloaking** | the filter endpoint, filter campaigns, visit journal, statistics of checked visits |
| **Blacklists** | IP, network and header block-lists |
| **Tracker data** | clicks, leads, statistics, postback templates, traffic-source books |
| **Ad-spend connectors** | cost import from Facebook, TikTok, BIGO Ads, VK, MGid, LuckyAds, Dolphin, FBtool, MixAdvert, AdPartner |
| **Hack APIs** | Facebook and TikTok conversion pixels, Google Ads offline conversions, Postback 2.0 (HTTP and Telegram) |
| **Account and system** | profile, registration and password recovery; liveness check, users, time zone |

## Install

### Claude Code

Download [`altercpa-lite-api-skill.zip`](https://github.com/altercpa/skill-lite/releases/latest/download/altercpa-lite-api-skill.zip)
and unpack it into a skills folder:

```sh
unzip altercpa-lite-api-skill.zip -d ~/.claude/skills/     # every project
unzip altercpa-lite-api-skill.zip -d .claude/skills/       # this project only
```

On Windows unpack it into `%USERPROFILE%\.claude\skills\`.

Prefer git? Clone straight into the skills folder, and `git pull` is the whole update:

```sh
git clone https://github.com/altercpa/skill-lite.git ~/.claude/skills/altercpa-lite-api
```

### Claude.ai and Claude Desktop

In Claude's settings add a custom skill and upload the same zip.

## Try it

Nothing to call by hand — Claude loads the skill on its own whenever a request touches the
AlterCPA Lite API. For example:

- *Write a Python script that downloads yesterday's click and lead statistics from my AlterCPA Lite tracker as CSV.*
- *How do I add an IP address to a blacklist through the API? Show me a curl call.*
- *Set up Facebook Conversions API postbacks for my flow.*

Calls follow one pattern; the token is `{user_id}-{api_key}` from your tracker's profile page:

```
https://{tracker}/api/{app}/{func}.json?id={token}
```

## Update

Releases are tagged `vYYYY.MM.DD`. Watch the repository (*Watch → Custom → Releases*) to be
notified, then download the new zip — or run `git pull` if you cloned.

## What is inside

| Path | What |
|---|---|
| [`SKILL.md`](SKILL.md) | the skill's entry point |
| [`references/`](references) | the reference, one file per API area |
| [`spec/`](spec) | OpenAPI 3.1 specifications, one file per area |
| [`docs/en/`](docs/en) · [`docs/ru/`](docs/ru) | the API documentation, Markdown sources with their images |

Links such as `/track-flows` in the documentation point to screens of your own tracker installation.

## License

[MIT](LICENSE) — use it, copy it, change it, ship it.

The mascot artwork in [`.github/assets/`](.github/assets) is © AlterCPA and is not covered by the MIT license.
