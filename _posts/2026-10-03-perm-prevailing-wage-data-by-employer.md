---
layout: post
title: "PERM Prevailing Wage Data by Employer"
subtitle: "Pull green card and H-1B wage filings for any company from the Department of Labor files, in Python"
description: "Get PERM prevailing wage data by employer in Python: what the Department of Labor files contain, a free route, and a $2 per 1,000 cases API with summaries."
seo_title: "PERM Prevailing Wage Data by Employer in Python"
tags: ["python", "api", "data", "apify"]
permalink: /perm-prevailing-wage-data-by-employer/
date: 2026-10-03 00:00:00 +0000
cadence_day: 5
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

When an employer sponsors a green card through PERM, or files a labor condition application for an H-1B worker, it tells the US Department of Labor what it plans to pay. Those filings are public. People look up PERM prevailing wage data by employer to benchmark pay, to see what a competitor offers, or to check an offer they received.

This guide covers what the data is, how to get it for free, and how to pull it for a list of employers in Python.

## What is in the data

The Department of Labor's Office of Foreign Labor Certification publishes disclosure files every quarter.

- **LCA files** cover H-1B, H-1B1 and E-3 applications, starting with fiscal year 2020. They include the offered wage, the prevailing wage and its level, from I (entry) to IV.
- **PERM files** cover permanent labor certifications for green cards, starting with fiscal year 2024 in the Actor below. They include the offered wage range.

The US fiscal year runs from October to September, so fiscal year 2026 started on October 1, 2025.

Two cautions that apply to every source. The wage is what the employer filed, which must be at least the prevailing wage. It is not the final salary, and it says nothing about bonus or equity. And one case can cover several positions, while the file lists the main worksite.

## The free way

Download the disclosure spreadsheets from the Department of Labor's OFLC performance data page, and filter them with pandas. This is fully legitimate and costs nothing. The practical issues are size, because a year of LCA data is a large spreadsheet, and columns that include contact details of attorneys and employer contacts, which you should drop before storing anything. Hourly, weekly and monthly wage units also need converting to a yearly figure before you can compare cases.

If you only need an occasional lookup, this is the better route. Do it once, save a cleaned file, and query that.

## A hosted option: H-1B & PERM Salary Data

[H-1B & PERM Salary Data](https://apify.com/conserving_celerytop/h1b-perm-salary-data) is an Apify Actor I built that reads those same Department of Labor files on each run, so new quarters appear when the department publishes them. It returns clean rows and converts wages to a yearly figure. Hourly is multiplied by 2,080, weekly by 52, bi-weekly by 26 and monthly by 12.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

All PERM cases for a few employers:

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("conserving_celerytop/h1b-perm-salary-data").call(
    run_input={
        "programs": ["perm"],
        "employers": ["Google", "Microsoft", "Amazon"],
        "fiscalYears": ["2025"],
        "maxResults": 500,
    },
    max_total_charge_usd=Decimal("2.00"),
)

for row in client.dataset(run.default_dataset_id).iterate_items():
    print(row["employerName"], row["jobTitle"], row["annualWage"], row["annualWageTo"])
```

Employer names match when the filed name contains your text, so `Google` also matches variants of the name. Leave `jobTitles` empty to get every case of the company.

To see how a company's pay is spread, use summary mode instead of case rows:

```python
run = client.actor("conserving_celerytop/h1b-perm-salary-data").call(
    run_input={
        "programs": ["lca"],
        "employers": ["Stripe"],
        "outputMode": "summary",
        "groupBy": "employerJobTitle",
        "minCasesPerGroup": 5,
        "maxResults": 50,
    },
    max_total_charge_usd=Decimal("2.00"),
)
```

Each summary row has `cases`, `employers`, `medianAnnualWage`, `p25AnnualWage`, `p75AnnualWage`, `p90AnnualWage`, the lowest and highest wage seen, and `medianAnnualPrevailingWage`.

## Sample output

One case row, from the Actor's own documentation (an LCA case):

```json
{
  "program": "LCA",
  "caseNumber": "I-200-25266-329529",
  "caseStatus": "Certified",
  "decisionDate": "2025-09-30",
  "employerName": "Pipe Technologies Inc.",
  "jobTitle": "Data Scientist",
  "socCode": "15-2051.00",
  "annualWage": 169541,
  "annualWageTo": 220000,
  "annualPrevailingWage": 169541,
  "pwWageLevel": "IV",
  "worksiteCity": "LONG ISLAND CITY",
  "worksiteState": "NY"
}
```

The wage level filter is described for LCA cases. Run a small PERM sample first, with `maxResults` at 10, and look at which prevailing wage columns are filled before you build on them.

The Actor does not read employer contact names, attorney or agent details, e-mail addresses, phone numbers or street addresses. Cases where the employer name looks like a private person, and PERM cases for live-in household work, are left out.

## What it costs

Store prices on October 2, 2026. Check the Store page before relying on them.

- $0.002 per case row, which is $2 per 1,000.
- $0.02 per summary row.
- Apify adds $0.00005 per run for each GB of memory at start.
- Rows that your filters leave out are free.
- Example: 500 PERM cases for three employers cost $1.00. A summary of 20 job titles at one employer costs 20 x $0.02 = $0.40.

A case search stops reading once it reaches `maxResults`. A summary reads a whole year and takes longer, about 25 seconds for a full year of LCA cases according to the Actor page.

## Alternatives

- **The Department of Labor files themselves.** Free and authoritative. You do the filtering and the unit conversion.
- **H-1B salary websites.** They republish the same files with a search box. Convenient for one company. Check what they let you export. The Actor page says it reads the Department of Labor files directly instead.
- **Levels.fyi and similar compensation sites.** They report total compensation, including equity, from self-reports. That is a different question from what was filed for a visa.
- **An immigration attorney or an HR benchmarking subscription**, if you need advice and not data.

## Limits

- Filed wages, not paid wages.
- A certified labor condition application is not an approved petition. USCIS decides the petition, and that data is separate.
- Companies file under legal entity names, so a brand can hide behind several names. Search for variants.

## FAQ

**Where can I find PERM prevailing wage data by employer?**
In the Department of Labor's OFLC disclosure files, or through a tool that reads them. Filter by employer name and fiscal year.

**What is a prevailing wage?**
The wage the Department of Labor treats as typical for a job in a location, which the offered wage must meet or beat.

**Is H-1B and PERM data public?**
Yes. The Department of Labor publishes the disclosure files for anyone to download.

**Can I see every case for one company?**
Yes. Put the company in `employers`, empty `jobTitles` and raise `maxResults`.

## Related guides

- [Hiring Signals for Sales: Score Your Account List From Job Postings](/hiring-signals-for-sales/)
- [Ashby Job Board API: Read Open Jobs and Salary Ranges in Python](/ashby-job-board-api-python/)
- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)
