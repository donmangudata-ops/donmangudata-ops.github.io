---
layout: post
title: "Hiring Signals for Sales: Score Your Account List From Job Postings"
subtitle: "Pull open roles for each account, score them in Python, and get a weekly list of accounts that changed"
description: "Turn job postings into hiring signals for outbound: pull open roles for each account, score them in Python, and get a weekly list of accounts that changed."
seo_title: "Hiring Signals for Sales: Score Accounts From Jobs"
tags: ["sales", "lead-generation", "python", "automation", "apify"]
permalink: /hiring-signals-for-sales/
date: 2026-09-29 00:00:00 +0000
cadence_day: 3
published: true
---
*Disclosure: I built the Apify Actor used in this guide, and it is paid. This article was drafted with an AI assistant. Field names and prices were checked on September 27, 2026.*

A company that opens three sales roles this month is about to change how it sells. One that posts its first data engineer is about to buy data tools. Job postings are one of the few buying signals a company publishes on purpose, with dates, in public.

Most signal tools sell that data inside a larger platform. If you already have an account list, you can build the core of it yourself: read each account's careers page, count what they are hiring for, and flag the changes. This guide does that in Python.

## Which hiring signals matter

Pick signals that map to what you sell. Some that hold up in practice:

- **Hiring in your buyer's team.** Selling to RevOps? Watch for sales and RevOps roles. Selling dev tools? Watch engineering.
- **A first hire in a function.** "Founding data engineer" or "first marketing hire" means someone is about to pick tools with no incumbent.
- **New leadership.** An open VP or director role often comes before a budget and a vendor review.
- **Speed.** Many roles posted in the last 30 days, compared with the total open.
- **Expansion.** Jobs in a country where the account had none before.
- **Tools in the job text.** A post that asks for Salesforce, Snowflake or HubSpot experience tells you what they already run.

None of these proves intent. They tell you where to look first, and they give the first line of an email something true to say.

## Step 1: one summary row per account

[ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api) reads the public careers pages of the companies you give it: Greenhouse, Lever, Ashby, Workday and 18 other job board systems. With `outputMode` set to `companies`, it returns one summary row per account instead of one row per job.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("accounts.txt") as f:          # one website or domain per line
    accounts = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/live-career-page-jobs-api").call(
    run_input={
        "companies": accounts[:500],
        "outputMode": "companies",
        "includeDescription": True,     # adds topTools and first hires found in the job text
    },
    max_total_charge_usd=Decimal("25.00"),  # a hard cap, not the price; see "What it costs"
)

rows = [r for r in client.dataset(run.default_dataset_id).iterate_items() if r["rowType"] == "company"]
```

Each summary row includes:

| Field | Meaning |
|---|---|
| `openJobs` | Open roles |
| `jobsPostedLast7Days`, `jobsPostedLast30Days` | Recent roles (null if the board gives no dates) |
| `functionCounts`, `functionCountsLast30Days` | Roles per function, such as `{"sales": 12, "engineering": 40}` |
| `salesShare`, `engineeringShare` | Share of sales and engineering roles |
| `leadershipRoles` | Up to 5 open director, VP and C-level roles, newest first |
| `firstHireRoles` | Up to 5 roles described as a first or founding hire |
| `topTools` | Tools named most often in the job text, with a category |
| `countries` | Where the open roles are |

Accounts it cannot resolve come back as rows with `rowType` set to `status` and a reason in `companyStatus`, such as `no_job_board_found`. Websites work when the site links to its job board; a board link always works best.

## Step 2: score the accounts

Here is a simple score for a team that sells to sales leaders. Change the weights to fit your product.

```python
def score(r, function="sales"):
    recent = (r.get("functionCountsLast30Days") or {}).get(function, 0)
    total = (r.get("functionCounts") or {}).get(function, 0)
    leaders = [x for x in (r.get("leadershipRoles") or []) if x.get("jobFunction") == function]
    firsts = [x for x in (r.get("firstHireRoles") or []) if x.get("jobFunction") == function]
    return recent * 3 + total + len(leaders) * 10 + len(firsts) * 15

ranked = sorted(rows, key=score, reverse=True)
for r in ranked[:25]:
    leads = ", ".join(x["title"] for x in (r.get("leadershipRoles") or [])[:2])
    name = r.get("companySlug") or r["company"]
    print(f'{name:<20} score={score(r):>3}  sales_open={(r.get("functionCounts") or {}).get("sales", 0):>3}  {leads}')
```

Push the top 25 to your CRM with the job titles as context. "Saw you're hiring a VP Sales and four AEs in Austin" beats a generic opener, and it is true.

## Step 3: a weekly list of what changed

A snapshot is useful once. The value is in change. Add three fields and schedule it:

```python
run_input = {
    "companies": accounts[:500],
    "outputMode": "companies",
    "onlyNewJobs": True,
    "monitorName": "accounts-weekly",
}
```

The first run records every open job. Each later run adds `newJobs`, `closedJobs`, `newFunctions` (functions with open roles now that had none last time, for example `["sales"]`) and `newCountries`. Save the input as a task in Apify Console, add a weekly schedule, and send the result to Slack, Google Sheets or a webhook. An account whose `newFunctions` contains your buyer's team is worth a look that week.

## What it costs

On September 27, 2026 the Store price was $0.045 per account, with up to 1,000 jobs included, and $0.0428, $0.0405 or $0.036 on paid Apify plans. Prices can change, so check the [Store page](https://apify.com/conserving_celerytop/live-career-page-jobs-api) before a large run. How it adds up:

- The first run costs one company lookup per account. In `companies` mode that is the whole price per account, however many jobs it has. So the first run for 500 accounts costs $22.50 on the free plan.
- Job descriptions, needed for `topTools` and first hires found in the text, are free on most systems. Workday, Eightfold and a few smaller systems, such as JazzHR and Paylocity, cost $0.01 per started block of 200 jobs described.
- Each weekly check after the first costs $0.002 per 1,000 open jobs on each board (checked September 27, 2026). Most accounts have fewer than 1,000 open jobs, so 500 accounts cost about $1 a week.
- `max_total_charge_usd` in the code is a hard cap. The run stops before it spends more.

Apify's free plan includes $5 of credit a month.

## Limits you should know

- **Function labels are read from titles.** Some jobs land in the wrong function, so read the titles before you act on a count. `salesShare` and `engineeringShare` use simpler keyword rules.
- **It reads the companies you list.** It will not find new accounts that are hiring. Bring a list from your CRM or a data provider.
- **Not every company has a careers page system.** Small companies that post only on LinkedIn come back as not found.
- **Old postings are included.** Some companies keep evergreen roles open all year. `jobsPostedLast30Days` separates new hiring from old listings.

## FAQ

**Is hiring really a buying signal?**
It is a timing signal. It shows where budget and headcount are going, which is often where new tools are bought. Use it to prioritize, not as proof.

**Can I get the hiring manager's name?**
No. The Actor reads job postings only and returns no personal data.

**How is this different from a signals platform?**
You get the raw counts and job titles for your own list, at a per-account price, and you decide the scoring. Platforms bundle more sources and contact data.


## Related guides

- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)
- [Lever API Job Postings: Pull Open Jobs From Lever With Python](/lever-postings-api-python/)
- [Ashby Job Board API: Read Open Jobs and Salary Ranges in Python](/ashby-job-board-api-python/)

---

*This article and the Actor are not affiliated with or endorsed by Greenhouse, Lever, Ashby, Workday or any other job board. The Actor reads only jobs published on public careers pages, with no login.*
