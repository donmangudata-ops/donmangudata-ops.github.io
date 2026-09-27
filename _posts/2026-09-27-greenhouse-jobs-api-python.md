---
layout: post
title: "Greenhouse Jobs API: Get Every Open Job From a Board in Python"
subtitle: "The free public Job Board API, what it returns, where it gets hard, and a hosted option for many boards"
description: "Pull every open job from a Greenhouse board in Python: the free public endpoint, what it returns, its limits, and a hosted option for many boards."
seo_title: "Greenhouse Jobs API: Get Every Open Job in Python"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /greenhouse-jobs-api-python/
date: 2026-09-27 09:00:00 +0000
cadence_day: 1
published: true
---
*Disclosure: I built the Apify Actors mentioned near the end, and they are paid. This article was drafted with an AI assistant. The endpoints were checked against live boards on September 26, 2026. Actor prices and field names were checked on September 27, 2026.*

Greenhouse is the applicant tracking system behind the careers pages of a lot of tech companies. If you want those jobs as data, for a job site, a recruiting list or a sales signal, you do not need a Greenhouse account. Every public Greenhouse board has a read-only Job Board API that answers without a key.

This guide covers that free endpoint first, then what gets hard once you have more than a handful of companies.

## Step 1: find the board token

Each Greenhouse board has a short name, called the board token. You can see it in the board link:

- `https://boards.greenhouse.io/stripe` has the token `stripe`
- `https://job-boards.greenhouse.io/stripe` is the newer host, same token

Many companies show their jobs on their own site instead. Open any job there and look at the link. A `gh_jid=` parameter means the page embeds a Greenhouse board, and the page source usually loads a `boards.greenhouse.io/embed/job_board` script whose `for=` parameter is the token.

## Step 2: call the Job Board API

The list endpoint is `https://boards-api.greenhouse.io/v1/boards/{token}/jobs`. It returns every published job in one response, with no paging.

```python
import requests

token = "stripe"
url = f"https://boards-api.greenhouse.io/v1/boards/{token}/jobs"
data = requests.get(url, params={"content": "true"}, timeout=30).json()

print(data["meta"]["total"], "open jobs")
for job in data["jobs"][:5]:
    print(job["title"], "|", job["location"]["name"], "|", job["first_published"], "|", job["absolute_url"])
```

When I ran this, Stripe's board returned 702 jobs. Each job has `id`, `title`, `location.name`, `first_published`, `updated_at`, `requisition_id` and `absolute_url`. With `content=true` you also get `content` (the description as escaped HTML), `departments` and `offices`.

Two details that trip people up:

1. `content` is HTML with entities escaped, so run it through `html.unescape` before you parse it.
2. Pay ranges are not in the list. Request a single job with `/jobs/{id}?pay_transparency=true` and read `pay_input_ranges`. It is often an empty list, because many companies publish pay only in the description text.

That is all you need for one company. The API is public, documented by Greenhouse and fast. If you track two or three boards, stop here and write a small cron job.

## Where it gets harder

The work grows once your list grows.

- **Finding tokens.** A list of 300 company websites does not come with board tokens. Some redirect to Greenhouse, some embed it, and some use a different system entirely.
- **Other systems.** In most real account lists, a good share of companies run Lever, Ashby, Workday or a European ATS such as Personio or Teamtailor. Each has its own endpoint and field names.
- **New and closed jobs.** The API shows what is open now. To see what changed since yesterday you have to store every run and diff it yourself.
- **Normalizing.** Location strings, remote flags, seniority and salary all come in different shapes.

You can build all of that. I did, and it took far longer than the first script.

## A hosted option for many companies

I packaged that work as an Apify Actor called [Greenhouse Jobs API](https://apify.com/conserving_celerytop/greenhouse-jobs-api). You give it board links, company websites or plain names and it reads the same public Greenhouse endpoint for each one, then returns one row per job in a fixed format.

You need an Apify account and its API token (Console, Settings, API & Integrations).

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("conserving_celerytop/greenhouse-jobs-api").call(
    run_input={
        "companies": [
            "https://boards.greenhouse.io/stripe",
            "https://job-boards.greenhouse.io/dropbox",
            "figma.com",
        ],
        "titleIncludes": ["engineer"],
        "postedSince": "30 days",
    },
    max_total_charge_usd=Decimal("0.50"),
)

for row in client.dataset(run.default_dataset_id).iterate_items():
    if row["rowType"] == "job":
        print(row["companySlug"], "|", row["title"], "|", row["countryCode"], "|", row["postedAt"])
    elif row["rowType"] == "status":
        print(row["company"], "->", row["companyStatus"])
```

`max_total_charge_usd` is a hard cap on what the run can cost, which is worth setting while you test.

Every job row has the same fields: `title`, `department`, `location`, `countryCode`, `workplaceType`, `seniority`, `jobFunction`, `salaryMin`, `salaryMax`, `salaryCurrency`, `postedAt`, `url` and `jobKey`. `jobKey` stays the same between runs, so you can join runs on it. A company with nothing to return gets one status row, such as `not_found` or `no_open_jobs`, instead of silently disappearing.

### Only new and closed jobs

Set `"onlyNewJobs": true` and give the watchlist a `"monitorName"`. The first run returns everything. Each later run returns only jobs that are new since the last run (`change: "new"`) and jobs that closed (`change: "closed"`). Save the input as a task in Apify, add a daily schedule, and you have a new-jobs feed without storing anything yourself.

## What it costs

These are the free plan prices on the Apify Store on September 27, 2026. Check the Store page before you rely on them.

- **Greenhouse Jobs API:** $0.10 per company, which includes up to 1,000 of its open jobs ($0.09 on the Scale plan, $0.07 on Business). Each further 1,000 jobs of the same company is $0.045. A later check with `onlyNewJobs` is $0.002 per 1,000 open jobs on the board. So 100 companies cost $10 for the first full pull, and a daily `onlyNewJobs` check of them about $0.20. Descriptions cost nothing extra.
- **[ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api)**, the multi-board version from the same author, reads Greenhouse plus Lever, Ashby, Workday and 18 other systems with the same input and output. It costs $0.045 per company with up to 1,000 jobs, less than the single-board version, so it is worth a look even for a Greenhouse-only list, and it is the one to use when your list mixes systems.

There are single-system versions for [Lever](https://apify.com/conserving_celerytop/lever-jobs-api), [Ashby](https://apify.com/conserving_celerytop/ashby-jobs-api) and [Workday](https://apify.com/conserving_celerytop/workday-jobs-api) too. Paid Apify plans get lower per-company prices. Apify's free plan includes $5 of monthly credit.

## When not to use it

- **One or two boards.** The free endpoint above is simpler and costs nothing.
- **Search across all companies.** Neither the endpoint nor the Actor searches every Greenhouse board by keyword. You bring the list of companies. For keyword search across a fixed set of 824 startups and tech companies there is a separate Actor, [Tech Jobs Search](https://apify.com/conserving_celerytop/tech-jobs-search).
- **Your own candidates or applications.** That is Greenhouse's Harvest API, which needs a key from the company's Greenhouse account. The Job Board API only shows published jobs.

## FAQ

**Do I need a Greenhouse API key to read job postings?**
No. The Job Board API is read-only and public for published jobs. Keys are only for posting applications and for the Harvest API.

**Is there a rate limit?**
Greenhouse does not publish one for the Job Board API. Be polite: one request per board per run, and cache the result.

**Can I get jobs posted months ago that are still open?**
Yes. The API returns every open job, whatever its age. `first_published` tells you how old it is.

---

*Greenhouse is a trademark of its owner. This article and the Actors are not affiliated with or endorsed by Greenhouse. The Actors read only jobs that companies publish on public boards, with no login.*
