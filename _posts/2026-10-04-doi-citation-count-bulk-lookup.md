---
layout: post
title: "DOI Citation Count Bulk Lookup in Python"
subtitle: "Get citation counts and metadata for a list of DOIs from Crossref, with a free script and a $0.75 per 1,000 API"
description: "Do a DOI citation count bulk lookup in Python: the free Crossref REST API script, what the counts mean, and a $0.75 per 1,000 works API for long DOI lists."
seo_title: "DOI Citation Count Bulk Lookup in Python"
tags: ["python", "api", "research", "apify"]
permalink: /doi-citation-count-bulk-lookup/
date: 2026-10-04 00:00:00 +0000
cadence_day: 6
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

You have a reading list, a department's publication list or a spreadsheet of DOIs, and you want to know how often each paper has been cited. That is a DOI citation count bulk lookup. Crossref, the registry that issues most scholarly DOIs, publishes the metadata for free, so you can do this without a subscription.

## What the number means

Crossref's citation count is the number of works registered with Crossref that cite the paper. It is not the same as the count in Google Scholar, Scopus or Web of Science, which index different sources, so their numbers differ. Use it to compare papers inside one list, not to quote as the one true figure. If you report it, say where it came from.

## The free way: the Crossref REST API

Crossref has a public REST API. Each work has a field called `is-referenced-by-count`. Crossref asks you to identify yourself with an email address in the request, which puts you in their "polite" pool.

```python
import requests
from urllib.parse import quote

MAILTO = "you@example.com"

def lookup(doi):
    r = requests.get(
        "https://api.crossref.org/works/" + quote(doi, safe=""),
        params={"mailto": MAILTO},
        timeout=30,
    )
    if r.status_code == 404:
        return {"doi": doi, "status": "not_found"}
    r.raise_for_status()
    m = r.json()["message"]
    return {
        "doi": doi,
        "title": (m.get("title") or [""])[0],
        "journal": (m.get("container-title") or [""])[0],
        "year": (m.get("issued", {}).get("date-parts") or [[None]])[0][0],
        "citations": m.get("is-referenced-by-count"),
        "status": "ok",
    }

print(lookup("10.1038/nature12373"))
```

For a few dozen DOIs this is all you need. For thousands, add a pause between calls, retries for server errors, and a cache so a rerun does not repeat work. Check Crossref's current rate guidance before you run it in parallel.

## A hosted option: Crossref Works Search

[Crossref Works Search](https://apify.com/conserving_celerytop/crossref-works-search) is an Apify Actor I built. It reads Crossref's public metadata live. Give it a list of DOIs, plain or as doi.org links, up to 10,000 per run, and it returns one row per work.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import csv, os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("dois.txt") as f:
    dois = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/crossref-works-search").call(
    run_input={"dois": dois},
    max_total_charge_usd=Decimal("5.00"),
)

with open("citations.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["doi", "title", "journal", "year", "citationCount", "status"])
    for r in client.dataset(run.default_dataset_id).iterate_items():
        w.writerow([r["doi"], r.get("title"), r.get("journal"),
                    r.get("publishedYear"), r.get("citationCount"), r["status"]])
```

When you pass `dois`, the search fields such as keywords and author are not used.

## Sample output

Sample row from the Actor page (the count changes over time).

```json
{
  "doi": "10.1038/nature12373",
  "doiUrl": "https://doi.org/10.1038/nature12373",
  "title": "Nanometre-scale thermometry in a living cell",
  "authors": ["G. Kucsko", "P. C. Maurer", "N. Y. Yao"],
  "authorCount": 8,
  "journal": "Nature",
  "publisher": "Springer Science and Business Media LLC",
  "type": "journal-article",
  "publishedDate": "2013-07-31",
  "publishedYear": 2013,
  "citationCount": 1827,
  "referencesCount": 30,
  "status": "ok"
}
```

The authors list is shortened here. Other fields in every row include `issn`, `volume`, `issue`, `pages`, `license`, `funders`, `subjects` and `links`. Abstracts are off by default. Turn on `includeAbstracts` to add them when the publisher deposited one.

A DOI that Crossref does not know comes back with the status `not_found`, and an entry that is not a DOI comes back as `invalid_input`. Both are free. That matters because DOIs from other registries, such as DataCite, are not in Crossref.

## What it costs

Store price on October 2, 2026. Check the Store page before relying on it.

- $0.00075 per work, which is $0.75 per 1,000, on every plan.
- Apify adds $0.00005 per run for each GB of memory at start.
- Unknown DOIs and entries that are not DOIs are free.
- Example from the Actor page: citation counts for a reading list of 150 DOIs cost about $0.11. A literature scan of 2,000 journal articles costs $1.50.

A work that appears in two runs is charged twice, so for a recurring job filter by publication date.

## Searching instead of looking up

The same Actor can find works when you do not have DOIs yet: by keywords, author, journal, ISSN, funder, date range and work type, sorted by relevance, date or most cited. Up to 10,000 works per search.

```python
run_input = {
    "query": "large language models",
    "publishedFrom": "2025-01",
    "types": ["journal-article"],
    "sortBy": "most-cited",
    "maxResults": 500,
}
```

## Alternatives

- **The Crossref REST API directly.** Free, and the script above is the starting point. Best if you want full control and have time for rate limits and retries.
- **OpenAlex.** A free, open index of scholarly works with its own citation counts. A good second opinion.
- **Semantic Scholar and Google Scholar.** Useful for individual papers.
- **Scopus and Web of Science.** The standard in many institutions, and paid. If your university has access, use them for numbers you will publish.

## Limits

- The count only includes citations from works registered with Crossref.
- It covers works with Crossref DOIs. Preprints on arXiv and datasets from DataCite are mostly absent unless Crossref holds them.
- Counts change over time. Note the date you ran the lookup.

## FAQ

**How do I get citation counts for a list of DOIs?**
Request each DOI from the Crossref API and read `is-referenced-by-count`, or send the list to a bulk tool and take the `citationCount` column.

**Why is the Crossref count lower than Google Scholar?**
Google Scholar indexes a different, wider set of sources, so its count differs. Crossref counts only works registered with it.

**Do I need an API key for Crossref?**
No. The public API needs no key. Adding your email in `mailto` is the polite way to use it.

**Can I look up DOIs written as links?**
Yes. The Actor accepts plain DOIs and doi.org links, and a script can strip the prefix before calling the API.

## Related guides

- [RSS to JSON API: Read Many Feeds in Python](/rss-to-json-api/)
- [PERM Prevailing Wage Data by Employer](/perm-prevailing-wage-data-by-employer/)
