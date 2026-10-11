---
layout: post
title: "Business Entity Search for Colorado and Connecticut With Open Data"
subtitle: "Both states publish their registries as open datasets. Here are the queries, the traps I found, and when a tool is worth it"
description: "How to search Colorado and Connecticut business entities by name or ID with open data: the two datasets, a query you can run yourself, common traps and the cost of a tool."
seo_title: "Business Entity Search: Colorado and Connecticut Guide"
tags: ["business-entity-search", "colorado", "connecticut", "open-data", "apify"]
permalink: /business-entity-search-colorado-connecticut/
date: 2026-10-10 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in this guide, and it is paid. The first half works with free tools only, so you do not need the Actor to follow it. This article was drafted with an AI assistant. I am not affiliated with the State of Colorado or the State of Connecticut.*

A business entity search answers one question: does this company exist in a state registry, and in what shape? Colorado and Connecticut publish their registries as open datasets, so you can run the search yourself over a plain web request. This guide shows the queries, the traps I found, and when it is worth letting a tool do the work.

## The two datasets

| | Colorado | Connecticut |
|---|---|---|
| Dataset | Business Entities in Colorado | Connecticut Business Registry - Business Master |
| Portal and ID | data.colorado.gov, 4ykn-tg5h | data.ct.gov, n7gp-d28j |
| Rows on 9 October 2026 | 3,120,546 | 1,301,912 |
| Licence field in the metadata | PUBLIC_DOMAIN | PUBLIC_DOMAIN |

I read the row counts with a count query through a fetch tool on 9 October 2026. My build environment cannot call the portals directly, so every number in this guide comes from that fetch tool or from a local fixture run, not from a live run of the Actor.

Colorado counted 1,144,901 entities with the status Delinquent and 1,021,598 in Good Standing in its status breakdown. Connecticut counted 463,873 Active entities out of 1,301,912 rows, 357,062 Forfeited and 320,002 Dissolved. A raw match of a company name therefore often returns a dead entity, and you should read the status before you trust a hit.

## Run the query yourself

A search by name is one request. This request finds Colorado entities whose name starts with ACME, three rows, in order of entity ID:

```text
https://data.colorado.gov/resource/4ykn-tg5h.json?$where=entityname like 'ACME%'&$order=entityid&$limit=3
```

I read this query on 9 October 2026 and got ACME CLEANERS, INC. (Voluntarily Dissolved, formed 19 March 1947), ACME ELECTRIC COMPANY, INC. and ACME RADIO SUPPLY, INC. Matching is case sensitive. A lower case pattern returned nothing on the Connecticut portal, while `upper(name) like 'ACME VIEW%'` found "ACME View Property Management LLC".

New registrations are a date filter. Colorado stores the date in `entityformdate` and Connecticut in `date_registration`:

```text
$where=entityformdate >= '2026-10-08T00:00:00' AND entityformdate < '2026-10-09T00:00:00'
```

On 8 October 2026 the counts were 605 new Colorado entities and 96 new Connecticut entities.

## Traps that cost me time

1. **Names with an ampersand.** Percent-encode the query. A raw `&` in "Eight & Change" splits the query parameters, and the request failed in my fetch tool. Encoded as `%26`, it found the Colorado entity "Eight & Change LLC".
2. **Colorado writes the status into the name.** A dissolved entity reads "ACME CLEANERS, INC., Dissolved November 8, 1984". An exact match on the clean name finds nothing, so use a prefix match.
3. **Connecticut has no principal address.** The master table holds a billing address. A Connecticut company can show a billing address in Massachusetts.
4. **Future dates exist.** Colorado held a row with a formation date of 1 November 2026 on 9 October 2026. End a new filings window at today.
5. **People are in the data.** The Colorado dataset has first and last name columns for the registered agent, and the Connecticut dataset has email columns. Ask only for the columns you need, and drop the rest.
6. **Throttling.** The Socrata documentation says requests without an app token come from a shared pool per IP address and may be throttled with HTTP 429. It gives no number. Send one request at a time and slow down on a 429.
7. **Terms.** The Colorado terms page asks applications that use its data to carry a disclaimer: "The data made available here has been modified for use from its original source, which is the State of Colorado." Read both terms pages before you build on the data. I read them through a summarising fetch tool, so confirm the wording yourself.

## When a tool is worth it

The queries above cover a handful of names. A tool starts to pay when you check hundreds of names, run a new filings feed every morning, or want both states in one dataset with the same fields.

My Actor does that for these two states: one row per entity with name, ID, status, type, formation date, jurisdiction and the city, state and ZIP of the principal address. It returns no registered agent who is a person, no officers and no emails. A name that finds nothing returns a free status row. The price is $2.00 per 1,000 entity records, and a first run with Max results 10 costs at most $0.02.

This is a crowded niche. A search for "business entity search" in the Apify Store returned 658 results on 9 October 2026, and several of them read the same two portals. Some cover 26 states. Mine covers two, states a licence field for each, adds the new filings mode at the same price, and keeps the people out.

The Actor is unofficial and not affiliated with the State of Colorado or the State of Connecticut. You can find it as [Business Entity Search: Colorado and Connecticut](https://apify.com/conserving_celerytop/us-state-open-business-registry-api).
