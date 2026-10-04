---
layout: post
title: "SmartRecruiters Postings API: Pull Open Jobs From a Company in Python"
subtitle: "The free public Postings API, what it returns, and why no hosted Actor covers it yet"
description: "Use SmartRecruiters' public Postings API to pull open jobs from any company's SmartRecruiters board in Python, plus an honest note on why no hosted Actor in this series handles SmartRecruiters yet."
seo_title: "SmartRecruiters Postings API in Python"
tags: ["python", "api", "web-scraping", "apify", "recruitment"]
permalink: /smartrecruiters-jobs-api-python/
date: 2026-10-05 00:00:00 +0000
cadence_day: 8
published: true
---
*Disclosure: I built the Apify Actors mentioned near the end, and they are paid. This article was drafted with an AI assistant. The endpoint was checked against a live SmartRecruiters board on October 5, 2026.*

SmartRecruiters is an applicant tracking system used by large retail, hospitality and consumer brands. Like Greenhouse and Ashby, it exposes a public, read-only Postings API for any company's published jobs, no key and no login needed. Unlike those two, it is not yet one of the systems the multi-ATS Actor in this series reads, so this post is the free endpoint on its own, plus an honest answer on what to use instead for a mixed list.

## Step 1: find the company identifier

SmartRecruiters career sites live at `jobs.smartrecruiters.com/{Company}`. The `{Company}` part, exactly as capitalized there, is the identifier the API uses too.

## Step 2: call the Postings API

```
GET https://api.smartrecruiters.com/v1/companies/{Company}/postings
```

```python
import requests

def smartrecruiters_jobs(company, limit=100):
    url = f"https://api.smartrecruiters.com/v1/companies/{company}/postings"
    jobs, offset = [], 0
    while True:
        data = requests.get(url, params={"limit": limit, "offset": offset}, timeout=30).json()
        jobs.extend(data["content"])
        offset += limit
        if offset >= data["totalFound"]:
            break
    return jobs

jobs = smartrecruiters_jobs("Equinox")
print(len(jobs), "jobs")
for j in jobs[:5]:
    print(j["name"], "|", j["location"]["fullLocation"], "|", j["department"]["label"], "|", j["typeOfEmployment"]["label"])
```

When I ran this, Equinox's board returned 740 open jobs, paged 100 at a time behind `offset` and `limit`, since (unlike Greenhouse and Ashby) SmartRecruiters does not return a whole board in one response.

## Fields worth knowing

Each item in `content` has: `id`, `name`, `refNumber`, `releasedDate`, `department.label`, `function.label`, `typeOfEmployment.label`, `experienceLevel.label`, `industry.label`, and a `location` object with `city`, `region`, `country`, `remote`, `hybrid`, `latitude` and `longitude`. There is also `customField`, a list of company-defined fields such as brand or internal region, which varies by company and is worth inspecting before you rely on it.

The list endpoint does not include the full job description or the apply link. For those, call the single-posting endpoint:

```python
detail = requests.get(
    f"https://api.smartrecruiters.com/v1/companies/Equinox/postings/{jobs[0]['id']}",
    timeout=30,
).json()
print(detail["postingUrl"])
print(detail["applyUrl"])
print(detail["jobAd"]["sections"].keys())
```

`jobAd.sections` holds the description broken into the sections the company defined (such as a company blurb and a role description), each as HTML, rather than one plain-text blob.

## Where this stops being enough

The same problems show up as with any single-system endpoint, at a second company:

- **Pagination at scale.** A 740-job board already needs 8 requests; a watchlist of 50 such companies is hundreds of requests to manage, retry and rate-limit yourself.
- **Finding identifiers.** A list of company websites does not tell you the SmartRecruiters identifier, or whether a company is on SmartRecruiters at all.
- **Mixed systems.** SmartRecruiters tends to show up next to Greenhouse, Workday and iCIMS in the same retail or hospitality account lists, each with its own shape.
- **What changed.** Tracking new and closed postings over time needs stored history.

## Why there's no hosted option here, honestly

The other guides in this series end by pointing at [ATS Jobs API](https://apify.com/conserving_celerytop/live-career-page-jobs-api), an Apify Actor I built that reads 22 ATS platforms (Greenhouse, Lever, Ashby, Workday, Workable, Personio, Teamtailor, Recruitee and others) from one input. SmartRecruiters is not on that list yet. If I pointed you at it for SmartRecruiters specifically, it would quietly skip that company or return a "not found" status row, not an error, which is worse than saying nothing.

So for a SmartRecruiters-only list, the free endpoint above, with pagination handled as shown, is genuinely what I would use right now. For a mixed list where most companies run Greenhouse, Lever, Ashby, Workday, Recruitee or one of the Actor's other 20 systems, ATS Jobs API still saves the work on those, and you can run the script above separately for the SmartRecruiters names in the same list. Adding SmartRecruiters support is on the list for that Actor; this post will get a line added here once it ships, rather than a new post that contradicts it.

## When not to use it

- **One or two SmartRecruiters boards.** The free endpoint is simple and costs nothing; there is no paid shortcut worth paying for at this scale.
- **Search across every SmartRecruiters company.** The API reads the companies you name, not a search index of all of them.
- **Your own candidate pipeline.** That needs SmartRecruiters' authenticated API with a key from the company's own account. The Postings API only returns published jobs.

## FAQ

**Do I need a SmartRecruiters API key to read job postings?** No. The Postings API is public and read-only for published jobs.

**Why did my request return `totalFound: 0`?** Either the identifier's capitalization doesn't match the one in the company's `jobs.smartrecruiters.com` URL, or the company isn't on SmartRecruiters.

**Is there a rate limit?** SmartRecruiters does not publish one for this endpoint in its own docs. Page through politely, one board at a time, and cache the result.

## Related guides

- [Recruitee Jobs API: Pull Open Jobs From a Recruitee Career Site in Python](/recruitee-jobs-api-python/)
- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)
- [Ashby Job Board API: Read Open Jobs and Salary Ranges in Python](/ashby-job-board-api-python/)
- [Workday Jobs API: Pull Open Jobs From Workday Career Sites](/workday-jobs-api-python/)

---

*SmartRecruiters is a trademark of its owner. This article is not affiliated with or endorsed by SmartRecruiters. The script above reads only jobs that companies publish on public boards, with no login.*
