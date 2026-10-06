---
layout: post
title: "French Company Search with the Free Government API"
subtitle: "Look up French companies by name, SIREN or activity code with curl and Python, and what to do when the list gets long"
description: "French company search with the free government API: look up a SIREN, filter by NAF code and department, and avoid the traps. Plus a $3 per 1,000 companies Actor."
seo_title: "French Company Search: Free Government API Guide"
tags: ["siren", "sirene", "company-data", "api", "apify"]
permalink: /french-company-search/
date: 2026-10-06 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The commands and outputs below were run on October 4, 2026.*

If you need to search French companies, you do not have to scrape a commercial site. The French government runs a free company search service on top of the SIRENE business register. It returns JSON, needs no key and no account, and its data is published under the Licence Ouverte 2.0. This guide shows how to use it, where it surprises people, and when a hosted version saves time.

## What you can ask for

Each French company has a SIREN, a 9 digit number. Its establishments have a SIRET, which is the SIREN plus 5 digits. You can search by name, SIREN, SIRET or VAT number, and you can filter by activity code (NAF), department, postal code, legal form, size category, employee band and revenue.

## One company by SIREN

```bash
curl -s -A "french-company-guide/1.0" \
  "https://recherche-entreprises.api.gouv.fr/search?q=652014051&est_entrepreneur_individuel=false&minimal=true&include=siege,finances,complements,tva&per_page=1"
```

Reduced with `jq`, the answer for SIREN 652014051 was:

```json
{
  "siren": "652014051",
  "name": "CARREFOUR",
  "legal_form": "5599",
  "naf": "64.20Z",
  "size": "GE",
  "postal_code": "91300",
  "city": "MASSY",
  "vat": "FR14652014051",
  "revenue": { "year": "2025", "ca": 81149000000, "net": 0 }
}
```

Three parts of that URL matter. `est_entrepreneur_individuel=false` leaves out sole proprietors, whose company name is the name of a person. `minimal=true` with `include=siege,finances,complements,tva` keeps the answer to company data. Without them, the answer includes a list of directors with names and birth months, which you probably do not want to store.

## A list by activity and place

This Python script walks the pages and prints five software companies (NAF 62.01Z) with a head office in Paris. It sends one request at a time, which is well under the limit of 7 per second.

```python
import time
import requests

API = "https://recherche-entreprises.api.gouv.fr/search"
HEADERS = {"User-Agent": "french-company-guide-example/1.0"}


def search(params, max_results=50):
    """Yield companies whose head office matches, one request at a time."""
    base = {
        "est_entrepreneur_individuel": "false",  # leave out sole proprietors
        "minimal": "true",
        "include": "siege,finances,tva",  # company data only, no officers
        "per_page": 25,
        **params,
    }
    sent = 0
    page = 1
    while sent < max_results and page <= 400:
        r = requests.get(API, params={**base, "page": page}, headers=HEADERS, timeout=30)
        r.raise_for_status()
        data = r.json()
        for c in data["results"]:
            # place filters match any establishment, so check the head office
            if params.get("departement") and c["siege"]["departement"] != params["departement"]:
                continue
            yield c
            sent += 1
            if sent >= max_results:
                return
        if page >= data["total_pages"]:
            return
        page += 1
        time.sleep(0.3)  # the service allows 7 requests per second


for c in search({"activite_principale": "62.01Z", "departement": "75", "etat_administratif": "A"}, max_results=5):
    fin = c.get("finances") or {}
    year = max(fin) if fin else None
    print(c["siren"], c["nom_complet"], c["siege"]["code_postal"], fin[year]["ca"] if year else None)
```

The real output of that run:

```text
849239587 SNAPDESK (SNAPDESK) 75010 2036401
452225790 SILICON STORE 75008 None
477550370 SERENSIA (SERENSIA - ARADUS - SERENSIA BY QUADIENT) 75009 0
813754389 UNITY TECHNOLOGIES SARL 75013 19496273
949180210 THE SANE INTENTION 75016 None
```

## Traps to know about

- **Place filters match any establishment.** A search for postal code 75008 returned a company with its head office in 44300. The script above checks the head office itself. Without that check, your Paris list contains companies from other regions.
- **10,000 results at most.** The service stops there, with 25 results per page. For a bigger list, split it with filters, for example one search per department.
- **Revenue is often missing.** Only companies that publish accounts have revenue and net income. In the output above, two of five have none.
- **Name searches are loose.** The service ranks by relevance and also matches trade names and other text. A search for the brand Doctolib returned five entities, among them the company itself and a works council that carries its name. Use a SIREN when you need one exact company.
- **People are in the data.** Officer names, birth months and sole proprietor names are all reachable through the same endpoint. If you build a company list, keep to the company fields.
- **Connections drop now and then.** In my tests a few requests were reset and worked on the second try. Add a retry with a pause.

## The licence

The service is published under the Licence Ouverte 2.0. It gives a "droit non exclusif et gratuit de libre Réutilisation" and asks you to name the source and the date of the last update. Keep the `date_mise_a_jour` field, or your fetch date, next to what you publish. The licence also says the information may contain personal data and that reuse must respect data protection law. That is one more reason to leave officers out.

## When a hosted version helps

The scripts above are enough for a few hundred companies. If you run lists of thousands, schedule the pulls, or feed an AI agent or a no-code tool, writing and maintaining the paging, retries, de-duplication and filters is the part that costs time.

I built an Actor for that: [French Company Search API](https://apify.com/conserving_celerytop/french-company-search-api) on the Apify Store. It takes names, SIREN or SIRET numbers or filters, returns one row per company with 41 fields, skips sole proprietors, never requests officers, and checks the head office when you filter by place. It costs $3.00 per 1,000 companies. In a local test it returned 1,000 companies in 10.6 seconds without a place filter and in 22.7 seconds with a Paris department filter.

The free API is a good tool and you can use it directly. The Actor is for people who prefer to pay a few dollars instead of maintaining the script.
