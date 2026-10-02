---
layout: post
title: "Lever API Job Postings: Pull Open Jobs From Lever With Python"
subtitle: "The public Postings API, EU boards, dates and salary, and a hosted option for many Lever sites"
description: "Read job postings from any Lever board with the public Postings API in Python: endpoints, EU boards, dates, salary, and a hosted option for many boards."
seo_title: "Lever Postings API: Pull Open Jobs With Python"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /lever-postings-api-python/
date: 2026-09-28 09:00:00 +0000
cadence_day: 2
published: true
---
*Disclosure: I built the Apify Actors mentioned in the second half, and they are paid. This article was drafted with an AI assistant. Endpoints were checked against live Lever boards on September 26, 2026. Actor prices and field names were checked on September 27, 2026.*

"Lever API" means two different things, and search results mix them up. One is the Lever API for customers, which reads candidates, opportunities and interviews and needs an API key from the company's Lever account. The other is the **Postings API**, which serves the jobs a company publishes on its Lever job site. The Postings API is public for reading, and it is the one you want for job data.

## The endpoint

Every Lever job site at `jobs.lever.co/{site}` has a matching JSON endpoint:

```
GET https://api.lever.co/v0/postings/{site}?mode=json
```

Boards hosted in Lever's EU region live at `jobs.eu.lever.co/{site}`, and their API host is `api.eu.lever.co`. If a site returns `{"ok": false, "error": "Document not found"}` on one host, try the other before you give up on it.

```python
import requests
from datetime import datetime, timezone

def lever_postings(site, eu=False):
    host = "api.eu.lever.co" if eu else "api.lever.co"
    resp = requests.get(f"https://{host}/v0/postings/{site}", params={"mode": "json"}, timeout=30)
    data = resp.json()
    if isinstance(data, dict) and data.get("ok") is False:
        return None  # no such site on this host
    return data

jobs = lever_postings("palantir")
print(len(jobs), "open postings")
for p in jobs[:5]:
    created = datetime.fromtimestamp(p["createdAt"] / 1000, tz=timezone.utc).date()
    cats = p["categories"]
    print(p["text"], "|", cats.get("team"), "|", cats.get("location"), "|", p.get("workplaceType"), "|", created)
```

On the day I ran it, Palantir's site returned 321 postings in one response.

## What each posting contains

The fields you will use most:

| Field | What it is |
|---|---|
| `id` | Posting id, stable while the job is open |
| `text` | The job title |
| `categories` | `team`, `department`, `location`, `commitment` (Full-time and so on) and `allLocations` |
| `workplaceType` | `on-site`, `hybrid`, `remote` or `unspecified` |
| `country` | Two-letter country code, when set |
| `createdAt` | Creation time in milliseconds since 1970 |
| `hostedUrl`, `applyUrl` | The public job page and its apply form |
| `descriptionPlain`, `lists`, `additionalPlain` | The description in plain text and bullet lists |
| `salaryRange` | Optional object with `currency`, `interval`, `min` and `max` |

`salaryRange` is optional, and in my checks many companies leave it empty, including Palantir and Spotify on that day. If pay matters to you, also search `descriptionPlain` and `additionalPlain`, where some companies write the range as text.

## Paging, filters and politeness

The endpoint accepts `skip` and `limit`, plus filters such as `team`, `location`, `commitment` and `department` (case sensitive when you pass several values). Without `limit` it returns the full list, which is usually what you want. Lever's docs describe a rate limit for application POSTs, not for reading postings, but the same manners apply: one request per site per run, cached, with a clear User-Agent.

## Where the one-script approach runs out

The script above is fine for a few companies. At 50 or 500 companies you run into four jobs you did not plan for:

1. **Finding the site name.** Your list is company names and websites, not Lever site slugs. Some companies are on the EU host, and many are not on Lever at all.
2. **Mixed systems.** Sales and recruiting lists mix Lever with Greenhouse, Ashby, Workday and smaller European systems.
3. **Change tracking.** Lever shows what is open. New and closed jobs since last week need stored state and a diff.
4. **One schema.** Location, seniority, function and salary need normalizing before a spreadsheet or CRM can use them.

## A hosted option

I built [Lever Jobs API](https://apify.com/conserving_celerytop/lever-jobs-api) on Apify for that case. It reads the same public Postings API, handles both regions, and returns one row per job in a fixed format. You pass Lever links, websites or company names.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("conserving_celerytop/lever-jobs-api").call(
    run_input={
        "companies": ["https://jobs.lever.co/palantir", "https://jobs.lever.co/spotify"],
        "jobFunctions": ["engineering", "data"],
        "postedSince": "14 days",
    },
    max_total_charge_usd=Decimal("0.50"),
)

rows = list(client.dataset(run.default_dataset_id).iterate_items())
jobs = [r for r in rows if r["rowType"] == "job"]
print(len(jobs), "matching jobs")
for r in jobs[:10]:
    print(r["companySlug"], "|", r["title"], "|", r["seniority"], "|", r["location"], "|", r["url"])
```

`createdAt` becomes an ISO date in `postedAt`. `categories.location` becomes `location` plus `countryCode`. The title is read for `seniority` (intern to C-level) and `jobFunction` (engineering, sales, data and so on), so you can filter the rows before they reach your spreadsheet. A company that cannot be found gets a status row that says why, and that row is not dropped.

For a watchlist, add `"onlyNewJobs": true` and a `"monitorName"`. After the first run, each run returns only new and closed postings. Save it as an Apify task and schedule it daily or weekly.

## Pricing

Free plan prices on the Apify Store on September 27, 2026. Check the Store page before you rely on them.

- **Lever Jobs API:** $0.10 per company with up to 1,000 open jobs ($0.09 on Scale, $0.07 on Business), then $0.045 per further 1,000. A later `onlyNewJobs` check is $0.002 per 1,000 open jobs. 100 Lever companies cost $10 for a full pull and about $0.20 per daily check after that. The example above, two companies, costs $0.20.
- **[ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api)** reads Lever and 21 other systems with the same fields. It costs $0.045 per company with up to 1,000 jobs, less than the Lever-only version, so it is worth a look even for an all-Lever list.

Paid Apify plans pay less per company. The free plan includes $5 of credit a month.

## When you should not use it

- A few Lever sites and no need for history: call the endpoint directly. It is free.
- You need every Lever job on the internet, searched by keyword: neither the endpoint nor the Actor does that. You supply the companies.
- You need candidates or pipeline data from your own Lever account: that is the authenticated Lever API, with a key from your admin.

## FAQ

**Does the Lever Postings API need an API key?**
Not for reading published postings. A key is needed to post applications and for the main Lever API.

**Why do I get "Document not found"?**
The site name is wrong, the board is on the EU host, or the company does not use Lever.

**How do I get the posting date?**
`createdAt` is milliseconds since 1970. Divide by 1,000 and convert to a date.


## Related guides

- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)
- [Ashby Job Board API: Read Open Jobs and Salary Ranges in Python](/ashby-job-board-api-python/)
- [Hiring Signals for Sales: Score Your Account List From Job Postings](/hiring-signals-for-sales/)

---

*Lever is a trademark of its owner. This article and the Actors are not affiliated with or endorsed by Lever. The Actors read only published postings, with no login.*
