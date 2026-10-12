---
layout: post
title: "How to Find Large Building Permits in Austin, Seattle and San Francisco"
subtitle: "Three open data portals, three formats. What the permit data holds, a weekly list of large projects, and where the data is thin"
description: "Find large building permits in Austin, Seattle and San Francisco from open data: fields, a weekly run, completeness checks and a $2 per 1,000 Actor."
seo_title: "Large Building Permits: Austin, Seattle, San Francisco"
tags: ["building-permits", "open-data", "socrata", "construction-leads", "apify"]
permalink: /large-building-permits-austin-seattle-san-francisco/
date: 2026-10-12 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in this guide, and it is paid. A free script route is covered too. This article was drafted with an AI assistant. All numbers come from runs on 11 October 2026.*

Building permits are public records. Three big US cities publish them as open data and update the files every day. The files look different in each city, so a list of new projects for all three needs some work. This post shows what the data holds, what a run of the city-building-permits-api Actor returned on 11.10.2026, and where the data is thin. All numbers come from real runs on that day.

## The three data sets

Austin, Seattle and San Francisco run their open data portals on the same platform (Socrata). Each portal has a web API with a query language called SoQL. You can ask it for rows between two dates, sorted, 1,000 rows at a time.

| City | Rows in the data set | Newest issue date on 11.10.2026 |
|---|---|---|
| Austin | 2,379,658 | 09.10.2026 |
| Seattle | 193,605 | 09.10.2026 |
| San Francisco | 1,296,756 (1,150,767 permits) | 09.10.2026 |

The San Francisco data set repeats a permit once for each address. 1,379 rows were issued from 11.09.2026 to 10.10.2026, but only 1,231 distinct permits. The Actor keeps the row of the primary address, so each permit comes once.

The portals gave no throttling answer during the tests. A test sent 380 requests to the three portals: 180 in a row at about 2.5 requests per second, and a burst of 200 requests at about 19 per second. Every answer was HTTP 200. The Socrata documentation still says that a request without an app token comes from a shared pool and can be throttled. The Actor therefore sends one request per second, as the robots.txt files of the portals ask.

## One run, three cities

The input was this:

```json
{
  "cities": ["austin", "seattle", "san-francisco"],
  "lastDays": 7,
  "minCost": 100000,
  "limit": 1000
}
```

The run took 3 seconds and 3 requests. It returned 117 permits issued from 04.10.2026 to 09.10.2026 with an estimated cost of at least $100,000. Seattle gave 52, San Francisco 52 and Austin 13. The estimated costs add up to $246,743,113. The median permit costs $329,338.

The five largest permits of the week:

1. Seattle 7046346-CN, new mixed-use building at 2929 27th Ave S, $69,876,675.
2. Seattle 6979957-DM, demolition of a commercial office building at 123 Boylston Ave E, $57,000,000.
3. Seattle 7118487-CN, permanent supportive housing at 2402 2nd Ave, $16,012,212.
4. San Francisco 202609017533, interior work on the 5th and 6th floors of 300 Howard St, $15,451,442.
5. San Francisco 202509155257, building upgrades at 125 Mason St, $13,000,000.

Austin has only 13 rows in this list, although Austin issues more permits than the other two cities. The reason is the cost field. Austin fills a valuation for building permits and for some trade permits. In a run of the 3,334 newest Austin permits, 45% had a cost. Electrical, plumbing and mechanical permits often have none. A minimum cost leaves them out. If you want every Austin permit, leave the minimum cost empty.

## Narrow the list by keyword and type

Three more runs, each one request set of 1 to 3 requests and 1 to 3 seconds:

- Keyword solar, last 90 days, three cities: 246 permits. Austin 163, San Francisco 82 and Seattle 1. The median cost is $16,699.50. Seattle has so few because this data set lists building permits, and solar work may sit under another permit type. This was not checked.
- Permit type demolition, Seattle, last 90 days: 169 permits issued from 14.07.2026 to 09.10.2026. 11 of them list housing units. The median cost is $800,000.
- Permit type new construction, San Francisco, last 365 days, cost at least $1,000,000: 39 permits. The largest is 202512151804 at 772 Pacific Ave with $105,000,000.

## Is the list complete?

A list is only useful if it holds every row. The check was simple. The portals reported 4,588 (Austin), 456 (Seattle) and 1,231 (San Francisco, distinct permits) permits issued from 11.09.2026 to 10.10.2026. A run with those dates and a limit of 10,000 returned 6,275 permits: 4,588, 456 and 1,231. The run took 12 seconds, 9 requests and 134 MB of memory.

A second check compared 15 rows, 5 from each city, with the live portal record. The script fetched each record by its permit number and recomputed every output field by its own rules: type, status, dates, address, ZIP code, parcel, cost, housing units, coordinates and link. 15 of 15 rows matched.

## What the Actor leaves out

The portals hold more than permits. The Austin data set has the contractor name, phone number and address and the applicant name and phone number in the same row. The Actor asks the portal for a fixed list of columns, and the list has no contractor, applicant or owner column. A test sends those columns on purpose and checks that none of them reaches the dataset. The free text of a description can hold an e-mail address or a phone number. The Actor removes them. In 10,000 output rows, no description held an e-mail address or a phone number.

This also keeps the Actor inside the terms of the cities. The Seattle terms say that a list of individuals is "not to be used for a commercial purpose". A permit list is a list of properties and projects. A list of contractor names would be closer to a list of individuals, since a sole trader is a person.

## What it costs

The price is $0.002 per permit, which is $2.00 per 1,000, plus $0.00005 for each run start. The weekly run above costs 117 x $0.002 + $0.00005 = $0.23405. Four weekly runs cost $0.9362 per month. The largest run, 10,000 permits from all three cities, costs $20.00005. Paid Apify plans pay less per permit.

## Where the data is thin

- Costs are estimates that the city or the applicant enters. Some are placeholders such as $0 or $1. The Actor passes them on.
- Seattle lists no parcel. San Francisco and Austin do.
- San Francisco lists no permit class and no work class. The fields are null.
- The status words differ by city (Active, Final, Issued, Complete). The Actor does not merge them.
- The cities give the data "as is". Check a permit on the city page before you act on it.

## Try it

The Actor is on the Apify Store as [City Building Permits API](https://apify.com/conserving_celerytop/city-building-permits-api). The input above is the prefill of the public Task "Large Building Permits This Week: Austin, Seattle, SF". The free Python version of the same query is in the [skill file on GitHub](https://github.com/donmangudata-ops/web-data-skills/tree/main/skills/city-building-permits-api). This is an independent tool. It is not affiliated with the City of Austin, the City of Seattle or the City and County of San Francisco.
