---
layout: post
title: "Ashby Job Board API: Read Open Jobs and Salary Ranges in Python"
subtitle: "Ashby's public posting API, how to read its salary tiers, and a hosted option for many Ashby boards"
description: "Use Ashby's public job posting API to pull open jobs and salary ranges from any Ashby board in Python, then scale it to many companies with one schema."
seo_title: "Ashby Job Board API: Jobs and Salaries in Python"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /ashby-job-board-api-python/
date: 2026-09-29 00:00:00 +0000
cadence_day: 3
published: true
---
*Disclosure: I built the Apify Actors mentioned in the second half, and they are paid. This article was drafted with an AI assistant. The endpoint was checked against live Ashby boards on September 26, 2026. Actor prices and field names were checked on September 27, 2026.*

Ashby has become a common applicant tracking system at startups and AI companies. Its hosted job boards live at `jobs.ashbyhq.com/{board}`, and Ashby documents a public posting API that returns the same jobs as JSON. For job data it also has something the older systems often lack, structured salary ranges, when the company chooses to publish them.

## The endpoint

```
GET https://api.ashbyhq.com/posting-api/job-board/{board}?includeCompensation=true
```

`{board}` is the part after `jobs.ashbyhq.com/`. For `jobs.ashbyhq.com/openai` it is `openai`. No key is needed to read published jobs. An unknown board returns HTTP 404.

```python
import requests

def ashby_jobs(board):
    url = f"https://api.ashbyhq.com/posting-api/job-board/{board}"
    resp = requests.get(url, params={"includeCompensation": "true"}, timeout=30)
    if resp.status_code == 404:
        return None
    resp.raise_for_status()
    return resp.json()["jobs"]

jobs = ashby_jobs("ashby")
print(len(jobs), "jobs")
for j in jobs[:5]:
    print(j["title"], "|", j["department"], "|", j["location"], "|", j["workplaceType"], "|", j["publishedAt"][:10])
```

The whole board comes back in one response. OpenAI's board had 830 jobs when I checked, with no paging.

## Fields worth knowing

- `title`, `department`, `team`, `employmentType`
- `location` plus `secondaryLocations` for jobs open in several places
- `isRemote` and `workplaceType`
- `publishedAt`, an ISO timestamp
- `jobUrl` and `applyUrl`
- `descriptionPlain` and `descriptionHtml`
- `isListed`, which is false for unlisted jobs that are reachable only by link
- `compensation`, only when you pass `includeCompensation=true`

### Reading the salary

`compensation.summaryComponents` is a list of parts: salary, bonus, equity and so on. Each has `compensationType`, `interval`, `currencyCode`, `minValue` and `maxValue`. The salary is the part whose type is `Salary`:

```python
def salary_range(job):
    comp = job.get("compensation") or {}
    for part in comp.get("summaryComponents") or []:
        if part.get("compensationType") == "Salary" and part.get("minValue") is not None:
            return part["minValue"], part["maxValue"], part["currencyCode"], part["interval"]
    return None

for j in jobs:
    s = salary_range(j)
    if s:
        print(j["title"], s)
```

On Ashby's own board, an engineering manager role in the EU returned `(110000, 185000, 'EUR', '1 YEAR')`. There is also `compensationTierSummary`, a display string such as "€110K - €185K • Offers Equity", which is handy for a UI but not for math.

A useful side effect: `publishedAt` shows how long a job has been open. The same board had a job first published in March 2024 and still listed in September 2026. Old open jobs are normal on career pages, and job aggregators often drop them.

## Where this stops being enough

For one board this is ten lines of code. The trouble starts with a list:

- **Board names.** A list of company websites does not tell you the Ashby board name, or whether the company uses Ashby at all.
- **Mixed systems.** Few lists are all Ashby. Greenhouse and Lever are common in the same segment, and European companies often use Personio, Teamtailor or Recruitee.
- **Other salary formats.** Ashby gives structured pay. Greenhouse usually does not in its list endpoint, and Lever only sometimes. Comparing pay across companies needs one format.
- **What changed.** New, closed and reposted jobs need stored history.

## A hosted option for many Ashby boards

[Ashby Jobs API](https://apify.com/conserving_celerytop/ashby-jobs-api) is an Apify Actor I built for this. You give it Ashby board links, company websites or plain names, it reads the same public posting API for each one, and it returns every job in a fixed format. Ashby salaries arrive as `salaryMin`, `salaryMax`, `salaryCurrency` and `salaryPeriod`, with `salaryRanges` holding one range per pay tier, plus `salaryAnnualMin` and `salaryAnnualMax` converted to a yearly figure.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("conserving_celerytop/ashby-jobs-api").call(
    run_input={
        "companies": [
            "https://jobs.ashbyhq.com/openai",
            "https://jobs.ashbyhq.com/ashby",
        ],
        "hasSalary": True,
        "minAnnualSalary": 150000,
        "minAnnualSalaryCurrency": "USD",
    },
    max_total_charge_usd=Decimal("0.50"),
)

for r in client.dataset(run.default_dataset_id).iterate_items():
    if r["rowType"] == "job":
        print(r["companySlug"], "|", r["title"], "|", r["salaryAnnualMin"], "-", r["salaryAnnualMax"], r["salaryCurrency"])
```

`hasSalary` keeps only jobs with published pay, from the board field or written in the job text. `minAnnualSalary` compares the yearly figure in the currency you name. There are no exchange rates, so jobs paid in euros or pounds are left out of a USD filter, and the `warning` field counts them. Set the currency to match the market you are looking at.

To watch a set of Ashby companies, add `"onlyNewJobs": true` and `"monitorName": "ashby-watch"`, save the input as a task, and schedule it. Later runs return only new and closed jobs.

If your list mixes Ashby with Greenhouse, Lever or Workday, [ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api) from the same author reads 22 systems with the same fields, at $0.045 per company.

## Pricing

Free plan prices on the Apify Store on September 27, 2026. Check the Store page before relying on them.

- $0.10 per company, including up to 1,000 of its open jobs ($0.09 on Scale, $0.07 on Business). Each further 1,000 jobs of that company is $0.045.
- A later check with `onlyNewJobs` is $0.002 per 1,000 open jobs on the board.
- Descriptions cost nothing extra.
- A name that turns out not to be on Ashby is still charged, because the board was looked up. Links to other job boards, websites that cannot be read, duplicates and invalid entries are free.
- Paid Apify plans pay less per company. The free plan gives $5 of credit a month, which covers about 50 companies.

The example above, two companies, costs $0.20.

## When not to use it

- One or two Ashby boards: the public endpoint is free and simple.
- A keyword search across every Ashby company: this reads the companies you list. It is not a search engine over all boards.
- Your own Ashby hiring data (candidates, interviews): that is Ashby's authenticated API, with a key from your Ashby admin.

## FAQ

**Does Ashby's job posting API need a key?**
No, not for published jobs.

**Why is `compensation` missing?**
Either you did not pass `includeCompensation=true`, or the company does not show pay on that job.

**Are unlisted jobs included?**
The API can return jobs with `isListed` set to false. Filter on it if you only want jobs shown on the public board. The Actor leaves unlisted jobs out.

---

*Ashby is a trademark of its owner. This article and the Actors are not affiliated with or endorsed by Ashby. The Actors read only jobs published on public boards, with no login.*
