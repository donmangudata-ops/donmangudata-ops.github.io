---
layout: post
title: "Company Name to Domain Bulk Lookup in Python"
subtitle: "Turn a list of company names into official website domains, checked against each company's own homepage"
description: "Company name to domain bulk lookup in Python: a DIY approach and its pitfalls, and a $1.50 per 1,000 found API that shows its evidence and charges only for matches."
seo_title: "Company Name to Domain Bulk Lookup in Python"
tags: ["python", "api", "sales", "apify"]
permalink: /company-name-to-domain-bulk/
date: 2026-10-04 00:00:00 +0000
cadence_day: 6
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

A common spreadsheet problem: a column of company names, and no website. The list could be CRM accounts, trade show exhibitors, grant recipients or suppliers. Most tools you want to run next, from a tech stack check to a DNS lookup, need a domain. That makes company name to domain bulk lookup a small job that comes up often.

## Why it is harder than it looks

Guessing `name.com` works for some companies and fails for many:

- The `.com` may belong to someone else, or be parked or for sale.
- The company may use `.io`, `.ai`, a country ending, or a different word entirely.
- Names such as "Globex GmbH" and "Globex" should resolve to the same site.
- Some domains redirect to a different final domain.

The core question is always the same: does the page at this domain actually say it is this company? A match on the name alone is not enough.

## A DIY approach

You can write it in an afternoon, and the steps are worth knowing:

1. Clean the name: lowercase, remove suffixes like Inc, Ltd, GmbH.
2. Build candidates: the name joined together and hyphenated, with endings such as com, io, co, ai, net, org.
3. Check DNS for each candidate and drop those that do not resolve.
4. Fetch the homepage of the ones that do, respecting robots.txt, with a user agent that names you.
5. Look for the company name in the page title, the `og:site_name` tag and schema.org data.
6. Discard parked pages and redirects to social networks or directories.

```python
import re
import requests

def candidates(name, tlds=("com", "io", "co", "ai", "net", "org")):
    base = re.sub(r"\b(inc|llc|ltd|gmbh|corp|co|company)\b\.?", "", name.lower())
    base = re.sub(r"[^a-z0-9 ]", "", base).strip()
    words = base.split()
    stems = {"".join(words), "-".join(words)}
    return [f"{s}.{t}" for s in stems for t in tlds if s]

def names_company(domain, name):
    try:
        html = requests.get(f"https://{domain}/", timeout=10,
                            headers={"User-Agent": "domain-check/0.1 (+https://example.com/contact)"}).text.lower()
    except requests.RequestException:
        return False
    return name.lower() in html[:20000]
```

The last function is deliberately crude. Matching the name anywhere in the HTML will accept pages that only mention the company, so a real version needs the title and site name checks from step 5. That scoring is where most of the time goes.

## A hosted option: Company Name to Domain

[Company Name to Domain](https://apify.com/conserving_celerytop/company-name-to-domain) is an Apify Actor I built that does the steps above. There is no search engine and no third-party company database behind it. It reads the company's own homepage and returns the domain only when that page names the company.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import csv, os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("companies.txt") as f:
    names = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/company-name-to-domain").call(
    run_input={
        "companies": names,
        "minConfidence": "medium",
        "maxCompanies": len(names),
    },
    max_total_charge_usd=Decimal("5.00"),
)

with open("domains.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["input", "domain", "confidence", "status"])
    for r in client.dataset(run.default_dataset_id).iterate_items():
        w.writerow([r["inputName"], r.get("domain"), r.get("confidence"), r["status"]])
```

Two input tricks. Set `country` to a two-letter code such as `de` or `uk` to try that country's endings first, or add the country per line: `Globex GmbH | de`. And set `maxCompanies`, which defaults to 100 and goes up to 10,000.

## Sample output

The Actor page shows this made-up company to illustrate the format:

```json
{
  "inputName": "Acme Widgets Inc.",
  "domain": "acmewidgets.com",
  "website": "https://acmewidgets.com/",
  "confidence": "high",
  "score": 100,
  "siteName": "Acme Widgets Inc.",
  "matchedSignals": [
    { "signal": "schemaOrg", "match": "exact", "value": "Acme Widgets Inc." },
    { "signal": "title", "match": "exact", "value": "Acme Widgets | Industrial widgets since 1990" }
  ],
  "rejectedCandidates": [{ "domain": "acmewidgets.io", "reason": "parked_or_for_sale" }],
  "status": "found",
  "charged": true
}
```

The evidence is the useful part. `matchedSignals` says why the answer was given and `rejectedCandidates` lists what was looked at and turned down. Confidence is high when the homepage names the company in its structured data, site name or title, medium when the name appears clearly but in fewer places, and low for a partial match, returned only if you choose low.

## What it costs

Store price on October 2, 2026. Check the Store page before relying on it.

- $0.0015 per domain found at medium confidence or above, which is $1.50 per 1,000.
- Apify adds $0.00005 per run for each GB of memory at start.
- Companies with no match, invalid entries and errors are free.
- Example from the Actor page: 2,000 names, 1,300 found, costs 1,300 x $0.0015 = $1.95. The 700 misses cost nothing.

Your hit rate depends on your list. Run 100 names first and look at the misses.

## Alternatives

- **Do it by hand or with a search engine.** Accurate for ten companies. It does not scale.
- **Wikidata.** For well-known organizations, the official website property is public and free to query. Small and private companies are often missing.
- **Commercial enrichment tools**, such as Clay, Apollo or ZoomInfo. They cover many companies, including ones whose domain has nothing to do with the name. They are worth it if you need contact data too.
- **Your own script.** The sketch above is a fair start if you want to own the logic.

## Limits

- It finds domains that follow from the company name. A company whose domain has no relation to its name will be not found, and so will one whose site blocks automated visits. Adding the country code often helps.
- It skips sites whose robots.txt does not allow it.
- If a domain redirects, you get the final domain, with the starting one in `triedDomain`.
- It returns no email addresses, phone numbers or people's names.

## FAQ

**How do I find a company's website from its name in bulk?**
Generate likely domains, check which resolve, and confirm the homepage names the company. Or send the list to a tool that does those checks.

**Is a company name to domain lookup accurate?**
It depends on the method. Checking the homepage for the company name is stronger than guessing, and a confidence score tells you which rows to review by hand.

**What if the same company has several domains?**
The Actor returns the main match and lists other matches in `otherMatches`.

**Can I use the domains with other tools?**
Yes. A domain list feeds a DNS lookup, a tech stack check or a CRM import directly.

## Related guides

- [Bulk MX Record Lookup in Python](/bulk-mx-record-lookup/)
- [Tech Stack Lookup API: CMS, Ecommerce and Analytics for Any Website](/tech-stack-lookup-api-python/)
- [Bulk SPF DMARC Checker in Python](/bulk-spf-dmarc-checker/)
