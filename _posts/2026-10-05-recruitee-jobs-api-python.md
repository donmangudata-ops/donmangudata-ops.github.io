---
layout: post
title: "Recruitee Jobs API: Pull Open Jobs From a Recruitee Career Site in Python"
subtitle: "The free public offers endpoint, its salary field, and a hosted option for lists that mix ATS platforms"
description: "Use Recruitee's public offers API to pull open jobs and salary ranges from any Recruitee career site in Python, then scale it to a mixed-ATS company list with one schema."
seo_title: "Recruitee Jobs API: Jobs and Salaries in Python"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /recruitee-jobs-api-python/
date: 2026-10-04 21:00:00 +0000
cadence_day: 7
published: true
---
*Disclosure: I built the Apify Actor mentioned near the end, and it is paid. This article was drafted with an AI assistant. The endpoint was checked against live Recruitee career sites on October 5, 2026.*

Recruitee is a career-site and hiring platform used mostly by Dutch and other European mid-size companies. Every company on it gets a career site at `{company}.recruitee.com`, and behind that site sits a public, read-only JSON endpoint that lists every published job, with no key and no login.

## Step 1: find the company subdomain

The subdomain is the first part of the career site's host name. Some companies keep it on `{company}.recruitee.com` directly; others point a custom domain at it, such as `werkenbij.vandebron.nl`. For a custom domain, open any job page and look at the apply link or page source for `recruitee.com` — the subdomain in that link is the one the API uses.

## Step 2: call the offers endpoint

```
GET https://{company}.recruitee.com/api/offers/
```

```python
import requests

def recruitee_jobs(company):
    url = f"https://{company}.recruitee.com/api/offers/"
    resp = requests.get(url, timeout=30)
    if resp.status_code == 404:
        return None
    resp.raise_for_status()
    return resp.json()["offers"]

jobs = recruitee_jobs("vandebron")
print(len(jobs), "jobs")
for j in jobs[:5]:
    print(j["title"], "|", j["location"], "|", j["department"], "|", j["careers_url"])
```

When I ran this, Vandebron (a Dutch energy company) had 11 open jobs, and Duravermeer, a construction group, had 352. Both came back in one response, with no paging.

## Fields worth knowing

- `title`, `slug`, `department`, `employment_type_code`
- `location`, plus `city`, `state_name`, `country`, `postal_code` split out
- `remote`, `hybrid`, `on_site` as three separate booleans, not one enum
- `published_at`, `created_at`, `updated_at`
- `careers_url`, the public job page; `careers_apply_url`, the direct apply link
- `description`, the full job text as HTML, included in the list response, no second request needed
- `status`, always `published` for jobs this endpoint returns
- `salary`, a structured field, present on every job even when empty

### Reading the salary

Unlike most career-page APIs, Recruitee returns a `salary` object on every posting, not just the ones with pay listed:

```python
def salary_range(job):
    s = job.get("salary") or {}
    if s.get("min") is None:
        return None
    return s["min"], s["max"], s["currency"], s["period"]

for j in jobs:
    s = salary_range(j)
    if s:
        print(j["title"], s)
```

On Vandebron's board, a business analyst role came back as `('3000', '3700', 'EUR', 'month')`. On Duravermeer's larger board, pay shows up in both shapes companies actually use: a site supervisor role at `('4291.63', '6256.72', 'EUR', 'month')` and an hourly trade role at `('18.95', '18.95', 'EUR', 'hour')`. Reading `period` matters, since comparing an hourly rate to a monthly one without converting first gives a meaningless number.

## Where this stops being enough

For one company, this is ten lines of code. The trouble starts with a list:

- **Mixed systems.** A list of European companies rarely runs on just one ATS. Greenhouse, Lever, Personio and Teamtailor show up in the same segment as Recruitee.
- **Finding subdomains.** A list of company websites does not tell you the Recruitee subdomain, or whether the company uses Recruitee at all; a fair number sit behind a custom career-site domain like Vandebron's.
- **Mixed pay units.** As above, hourly and monthly figures need converting before they can be compared or filtered.
- **What changed.** New, closed and reposted jobs need stored history to detect.

## A hosted option for mixed ATS lists

Recruitee is one of 22 systems read by [ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api), an Apify Actor I built. You give it company names, websites or job board links, and for a Recruitee company it reads the same public offers endpoint, then returns every job in the same fixed schema it uses for Greenhouse, Lever, Ashby and the rest.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("conserving_celerytop/live-career-page-jobs-api").call(
    run_input={
        "companies": [
            "https://vandebron.recruitee.com",
            "https://werkenbij.vandebron.nl",
            "figma.com",
        ],
    },
    max_total_charge_usd=Decimal("0.50"),
)

for row in client.dataset(run.default_dataset_id).iterate_items():
    if row["rowType"] == "job":
        print(row["companySlug"], "|", row["title"], "|", row["countryCode"], "|", row["postedAt"])
    elif row["rowType"] == "status":
        print(row["company"], "->", row["companyStatus"])
```

Pass either the `{company}.recruitee.com` address or a custom career-site domain that embeds Recruitee; the Actor resolves either one to the same board. A company on none of the 22 supported systems gets one status row instead of silently disappearing, so a mixed list of 300 companies tells you which ones it could not place, not just the ones it could.

### Only new and closed jobs

Set `"onlyNewJobs": true` with a `"monitorName"` to get a feed instead of a one-off pull. The first run returns everything; later runs return only jobs that are new (`change: "new"`) or that closed, across every system in the list, Recruitee included.

## Pricing

Free plan prices on the Apify Store, checked October 5, 2026. Check the Store page before relying on them.

- $0.045 per company, including up to 1,000 of its open jobs ($0.0428 on Starter, $0.0405 on Scale, $0.036 on Business and above). A company with more than 1,000 open jobs is charged $0.01 for each further 1,000.
- A later check with `onlyNewJobs` is $0.002 per started 1,000 open jobs on the board.
- Recruitee descriptions are included in the list response, so they cost nothing extra; that only applies to a handful of other systems on the list (JazzHR, Paylocity, Freshteam, JOIN, Workday, Eightfold).
- Single-system versions of this reader exist for [Greenhouse](https://apify.com/conserving_celerytop/greenhouse-jobs-api), [Lever](https://apify.com/conserving_celerytop/lever-jobs-api), [Ashby](https://apify.com/conserving_celerytop/ashby-jobs-api) and [Workday](https://apify.com/conserving_celerytop/workday-jobs-api); there is no Recruitee-only version, since Recruitee boards are rarely the only system on a real company list.

## When not to use it

- **One or two Recruitee boards.** The free endpoint above is simpler and costs nothing.
- **Search across every Recruitee company.** Neither the endpoint nor the Actor searches by keyword across companies you have not named.
- **Your own candidate data.** Reading applications, pipelines or interview notes needs Recruitee's authenticated API with a key from the company's own account. The public offers endpoint only shows published jobs.

## FAQ

**Do I need a Recruitee API key to read job postings?** No. The `/api/offers/` endpoint is public and read-only for published jobs.

**Does every Recruitee company expose this endpoint?** Every company whose career site is live on Recruitee's own domain or a custom domain pointed at it does, as long as the career site itself is public.

**Is salary always filled in?** No. The `salary` object is always present, but `min` and `max` are `null` on any job where the company chose not to publish pay.

## Related guides

- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)
- [Ashby Job Board API: Read Open Jobs and Salary Ranges in Python](/ashby-job-board-api-python/)
- [Lever API Job Postings: Pull Open Jobs From Lever With Python](/lever-postings-api-python/)
- [SmartRecruiters Postings API: Pull Open Jobs From a Company in Python](/smartrecruiters-jobs-api-python/)

---

*Recruitee is a trademark of its owner. This article and the Actor are not affiliated with or endorsed by Recruitee. The Actor reads only jobs published on public career sites, with no login.*
