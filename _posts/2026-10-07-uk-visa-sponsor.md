---
layout: post
title: "How to Check a UK Visa Sponsor in the Home Office Register"
subtitle: "Find out whether a company is a licensed sponsor, which routes it holds, and what the register cannot tell you"
description: "UK visa sponsor check with the Home Office register of licensed sponsors: download the CSV, read the rating and route columns, and avoid common mistakes."
seo_title: "UK Visa Sponsor Check: Home Office Register Guide"
tags: ["uk-visa-sponsor", "skilled-worker-visa", "open-data", "csv", "apify"]
permalink: /uk-visa-sponsor/
date: 2026-10-07 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The commands and outputs below were run on October 4, 2026. I am not affiliated with the UK Home Office.*

If you want to know whether a UK company is a licensed visa sponsor, you do not need a paid database. The Home Office publishes the register of licensed sponsors as a CSV file on GOV.UK. This guide shows what is in the file, how to search it, and where people get misled.

## What the register is

The register lists organisations that hold a sponsor licence for workers and temporary workers. GOV.UK describes it as a list of Worker and Temporary Worker sponsors, with the category of workers they are licensed to sponsor and their sponsorship rating. The version I used was dated 2 October 2026 and had 143,138 rows. It is published under the Open Government Licence v3.0, which allows reuse if you credit the source.

A row has five columns: Organisation Name, Town/City, County, Type & Rating, and Route.

## Reading the columns

- **Type & Rating** combines two facts, for example "Worker (A rating)" or "Temporary Worker (A rating)". In the file I read, 137,329 rows were Worker (A rating) and 87 were Worker (B rating). A few rows carry A (Premium), A (SME+) or a provisional UK Expansion Worker rating.
- **Route** is the immigration route the licence covers, such as Skilled Worker, Charity Worker or Global Business Mobility. One organisation appears once per route, so a company with three routes has three rows.
- **County** is empty in about two thirds of the rows. Do not rely on it.
- Names and towns contain stray spaces and mixed capitals, so trim and lower-case before you compare.

## Check one company with Python

GOV.UK exposes the page content as JSON. The attachment list holds the address of the current CSV, and the address changes every time the register is updated, so look it up each time instead of saving it.

```python
import csv, io, json, urllib.request

API = "https://www.gov.uk/api/content/government/publications/register-of-licensed-sponsors-workers"
page = json.load(urllib.request.urlopen(API))
csv_url = page["details"]["attachments"][0]["url"]
text = urllib.request.urlopen(csv_url).read().decode("utf-8-sig")

rows = list(csv.DictReader(io.StringIO(text)))
print(len(rows), "rows from", csv_url.rsplit("/", 1)[-1])

word = "monzo"
for r in rows:
    if word in r["Organisation Name"].lower().split():
        print(r["Organisation Name"].strip(), "|", r["Town/City"].strip(), "|", r["Type & Rating"], "|", r["Route"])
```

Output on 4 October 2026:

```text
143138 rows from SP_-_Worker_and_Temporary_Worker_Web_Register_-_2026-10-02.csv
Monzo Bank Ltd | London | Worker (A rating) | Skilled Worker
Monzo Bank Ltd | London | Worker (A rating) | Global Business Mobility: Senior or Specialist Worker
Monzo Bank Ltd | London | Worker (A rating) | Skilled Worker
```

Look at the first and third lines. They are the same row once the spaces are trimmed: the file contains the same row twice, one with a trailing space after London. If you count sponsors or routes, remove duplicates after you clean the text.

## Three mistakes to avoid

**Substring matching.** Searching for "tesco" inside every name returns 11 rows in this file, but only 6 have the word Tesco as a whole word. The others are names like ATESCO CONSULTANCY LTD and Notesco UK Limited. Match whole words, or compare the full name.

**Trading names.** Some rows read like "ACME LTD T/A SHOP". If you search for the shop name only, you can still find it, but searching for the company name that appears on a job advert may not. Try both.

**Treating the register as a job list.** A licence means the organisation is allowed to sponsor. It does not mean it has a vacancy, and a licence can be suspended or revoked after the file date. Check the current register before you rely on a result, and check the vacancy separately.

## A note on personal data

Some sponsors are sole traders who appear under their own name, for example a person's name followed by "T/A" and a shop name. If you build a list from the register, think about whether you should keep or publish those rows. Company names are one thing, a private person's name is another.

## Limits of the file

The file has no company number, no address beyond town and county, no website and no count of licences. For anything beyond the five columns, you need another source.

## If you do not want to write the code

For lists of companies, towns or routes, I built an unofficial Apify Actor called [UK Visa Sponsor Register Search](https://apify.com/conserving_celerytop/uk-visa-sponsor-register-api). It reads the current file on every run, matches whole words or exact names, filters by town, route and rating, leaves out rows that look like a person's own name, removes exact duplicates and stamps each row with the register date. It costs $2.00 per 1,000 rows. Checking Monzo, Octopus Energy, Revolut and Wise Payments returned 9 rows in a run that took about four seconds. The Python above does the same for one company at no cost, and for a single lookup it is the better choice.

Contains public sector information licensed under the Open Government Licence v3.0.
