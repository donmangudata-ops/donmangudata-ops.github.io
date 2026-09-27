---
layout: post
title: "Workday Jobs API: Pull Open Jobs From Workday Career Sites"
subtitle: "What Workday offers outsiders, why scrapers break, and a hosted option priced per company"
description: "Get open jobs from any Workday career site in Python: how the links work, why DIY scrapers break, and a hosted API at a flat price per company."
seo_title: "Workday Jobs API: Pull Open Jobs From Career Sites"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /workday-jobs-api-python/
date: 2026-09-27 09:00:00 +0000
cadence_day: 1
published: true
---
*Disclosure: I built the Apify Actor used in this guide, and it is paid. This article was drafted with an AI assistant. Field names and prices were checked on September 27, 2026.*

Many large employers run their careers pages on Workday. Intel, Adobe and Salesforce are among them. If you searched for "Workday jobs API", you probably want those open jobs as data, for a job board, a recruiting list or a sales signal. This guide covers what Workday offers outsiders, why a do-it-yourself scraper is harder than it looks, and a hosted option priced per company.

## Two different things called "Workday API"

**Workday's official APIs** (REST and SOAP Web Services) belong to the employer. You need an integration account inside that company's Workday tenant, set up by its HR or IT team. They fit if you work there and want your own requisitions, not for reading other companies' jobs.

**Public career sites** are what job seekers see, at addresses like `https://intel.wd1.myworkdayjobs.com/External`. Anyone can open them without logging in, but Workday publishes no documented jobs API for outsiders.

## How a Workday career site link is built

Every link has three parts you need:

- the **tenant**, the company's name in Workday (`intel`)
- the **pod**, a data center label such as `wd1`, `wd5` or `wd12`
- the **site name**, the path after the host (`External`, `external_experienced`)

The site name matters: one company can run several sites, such as one for students and one for experienced hires. This parser pulls the three parts from any job or search link:

```python
from urllib.parse import urlparse

def parse_workday_link(link):
    u = urlparse(link)
    host = u.hostname or ""
    if not host.endswith(".myworkdayjobs.com"):
        return None
    tenant, pod = host.split(".")[:2]            # "intel", "wd1"
    parts = [p for p in u.path.split("/") if p]
    if parts and len(parts[0]) == 5 and parts[0][2] == "-":
        parts = parts[1:]                        # drop a locale such as "en-US"
    site = parts[0] if parts else None           # "External"
    return {"tenant": tenant, "pod": pod, "site": site}

print(parse_workday_link(
    "https://adobe.wd5.myworkdayjobs.com/en-US/external_experienced/job/San-Jose/Senior-Engineer_R123"
))
# {'tenant': 'adobe', 'pod': 'wd5', 'site': 'external_experienced'}
```

To find a company's link, open any job on its careers page. If the address contains `myworkdayjobs.com`, you have it; otherwise check the apply button's link.

## Why a do-it-yourself scraper takes longer than expected

Reading one site once is a weekend task. Keeping many sites running is not:

- **The page is a JavaScript app.** A plain HTTP request returns an empty shell, and the job list loads in pages.
- **Dates are text.** Lists say "Posted 3 Days Ago" or "Posted 30+ Days Ago", so older jobs have no exact date unless you open each job.
- **No salary field.** Pay, when it is published at all, sits inside the description text.
- **Locations are shortened.** A job in three cities shows as "US Washington DC (+2 more)".
- **Large sites.** Some employers list thousands of open jobs, and simple paging stops before the end.
- **Each employer sets its own rules.** Check the site's robots.txt before you read it, keep requests slow, and skip sites that do not allow automated access.

None of this is impossible, but it repeats for every site you add and breaks quietly when a site changes.

## A hosted option: Workday Jobs API

I packaged that work as an Apify Actor, [Workday Jobs API](https://apify.com/conserving_celerytop/workday-jobs-api). You paste career site links (or company websites that link to Workday) and get one row per open job in a fixed format, read live, and only where the employer's robots.txt allows it.

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

run = client.actor("conserving_celerytop/workday-jobs-api").call(
    run_input={
        "companies": [
            "https://intel.wd1.myworkdayjobs.com/External",
            "https://adobe.wd5.myworkdayjobs.com/external_experienced",
        ],
        "jobFunctions": ["engineering", "data"],
        "postedSince": "7 days",
    },
    max_total_charge_usd=Decimal("2.00"),   # a hard cap for this run
)

for job in client.dataset(run.default_dataset_id).iterate_items():
    if job["rowType"] == "job":
        print(job["title"], "|", job["location"], "|", job["postedAt"], "|", job["url"])
    else:
        print("status:", job["company"], job["companyStatus"])
```

Each job row has, among others:

| Field | Meaning |
|---|---|
| `title`, `department` | Job title and the category the site lists it under |
| `location`, `countryCode`, `city` | Location text and parsed parts |
| `workplaceType`, `remote` | Remote, hybrid or onsite |
| `employmentType` | Workday's time type, such as "Full time" |
| `jobFunction`, `seniority` | Read from the title |
| `postedAt` | Posted date; null for jobs 30 or more days old unless you add descriptions |
| `url`, `applyUrl` | The job page |

Set `includeDescription` to `true` to add the description (with contact details removed), the tools it names, years of experience and any salary written in the text. A company with no matching jobs gets one status row that says why, such as `no_matching_jobs` or `not_found`.

## Get only new jobs on a schedule

Add two fields and save the input as a task in Apify Console:

```python
run_input = {
    "companies": ["https://intel.wd1.myworkdayjobs.com/External"],
    "onlyNewJobs": True,
    "monitorName": "workday-weekly",
    "alertWebhookUrl": "https://hooks.slack.com/services/...",   # optional
}
```

The first check returns the full list. Later checks return only jobs that are new or closed since the last one (`change` is `new` or `closed`) and can post an alert to Slack, Teams or any webhook. Schedule the task daily or weekly.

For one summary row per company, with open jobs, recent postings and counts per function, set `outputMode` to `companies`.

## What it costs

On September 27, 2026 the Actor charged $0.10 per company, with up to 1,000 jobs included, on Apify's free plan ($0.09 on Scale, $0.07 on Business). Prices can change, so see the [Store page](https://apify.com/conserving_celerytop/workday-jobs-api) for the current price. How it adds up:

- $0.10 per company, also when a site lists no jobs or a website links no job board
- $0.05 for each further 1,000 jobs from one company
- $0.01 per started block of 200 descriptions, only when you ask for them
- $0.002 per later "only new jobs" check, per started 1,000 open jobs on the site

So 100 companies cost $10 for the first read, and a daily check of the same 100 about $0.20. Sites whose robots.txt refuses access, invalid links and duplicates are free. `max_total_charge_usd` is a hard cap.

## Limits you should know

- **It reads only Workday.** For a list that mixes Greenhouse, Lever, Ashby and others, use [ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api), which reads 22 job board systems including Workday, at $0.045 per company.
- **You bring the companies.** It does not search all Workday employers. Paste links, websites, or pick a ready-made list.
- **Salary comes from the text.** Workday has no pay field.

## FAQ

**Is there an official public Workday jobs API?**
Not for outsiders. Workday's documented APIs need credentials from the employer. Public career sites can be viewed by anyone, and this Actor reads what they show.

**Can I get the recruiter's name?**
No. Rows hold job data only, and contact details are removed from descriptions.

---

*This article and the Actor are not affiliated with or endorsed by Workday, Inc. or any employer whose site it reads. Workday is a trademark of its owner.*
