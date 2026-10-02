---
layout: post
title: "Tech Stack Lookup API: CMS, Ecommerce and Analytics for Any Website"
subtitle: "How detection works, a small DIY script, and a hosted API for lists of thousands of websites"
description: "Look up the CMS, ecommerce, analytics and payment tools of many websites in Python: how detection works, a small DIY script, and a $2 per 1,000 API."
seo_title: "Tech Stack Lookup API: CMS and Analytics in Python"
tags: ["python", "api", "web-scraping", "webdev", "apify"]
permalink: /tech-stack-lookup-api-python/
date: 2026-09-28 09:00:00 +0000
cadence_day: 2
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on September 27, 2026.*

Knowing what a website runs on is useful in a lot of jobs. An agency that builds on Shopify wants the Shopify stores in its market. A payments company wants to know who already uses Stripe. A sales team wants to sort 2,000 leads by CMS before it writes a single email.

You can do this by hand with a browser extension, one site at a time. For a list, you want an API. This guide explains how detection works, gives you a small script to try it yourself, and then shows a hosted option for large lists.

## How tech stack detection works

Most tools, commercial or open source, work the same way. They load a page and look for fingerprints:

- **Script and stylesheet URLs.** `cdn.shopify.com` means Shopify, `js.stripe.com` means Stripe, `/wp-content/` means WordPress.
- **HTML markers.** A `<meta name="generator">` tag, or attributes such as `data-wf-site` on Webflow sites.
- **Response headers and cookies.** `Server`, `X-Powered-By` and cookie names often name the platform or host.
- **Versions**, when a file name or tag includes one.

A widely used open fingerprint set started in the Wappalyzer project and is now maintained as [webappanalyzer](https://github.com/enthec/webappanalyzer), with thousands of technologies and their patterns. Accuracy depends on what the page shows. A tool that is loaded later by JavaScript, for example an analytics tag inside a tag manager, only appears if you run the page in a browser.

## A small script to try it yourself

This script checks a site's `robots.txt`, loads the homepage once, and looks for a few fingerprints. The patterns are examples, not a full set.

```python
import re
import requests
from urllib.robotparser import RobotFileParser

UA = "stack-check-example/0.1 (+https://example.com/contact)"
SIGNS = {
    "Shopify": [r"cdn\.shopify\.com", r"Shopify\.theme"],
    "WordPress": [r"/wp-content/", r"/wp-includes/"],
    "Webflow": [r"data-wf-site", r"webflow\.js"],
    "Google Analytics": [r"googletagmanager\.com/gtag/js", r"google-analytics\.com/analytics\.js"],
    "Google Tag Manager": [r"googletagmanager\.com/gtm\.js"],
    "HubSpot": [r"js\.hs-scripts\.com", r"js\.hsforms\.net"],
    "Stripe": [r"js\.stripe\.com"],
}

def check(domain):
    base = f"https://{domain}"
    robots = RobotFileParser()
    r = requests.get(base + "/robots.txt", headers={"User-Agent": UA}, timeout=20)
    robots.parse(r.text.splitlines() if r.status_code == 200 else [])
    if not robots.can_fetch(UA, base + "/"):
        return {"domain": domain, "status": "robots_disallowed", "found": []}
    resp = requests.get(base + "/", headers={"User-Agent": UA}, timeout=20)
    html = resp.text
    found = [name for name, pats in SIGNS.items() if any(re.search(p, html) for p in pats)]
    gen = re.search(r'<meta[^>]+name=["\']generator["\'][^>]+content=["\']([^"\']+)', html, re.I)
    if gen:
        found.append("generator: " + gen.group(1))
    return {"domain": domain, "status": resp.status_code, "server": resp.headers.get("server"), "found": found}

print(check("pypi.org"))
```

Put your own contact link in the User-Agent. Site owners appreciate knowing who is calling.

This works for a handful of sites. At a few thousand, the fingerprint list becomes the real work: keeping patterns current, scoring confidence, reading versions, and handling redirects, timeouts and blocked sites without losing track of which is which.

## A hosted option: Website Tech Stack Detector

[Website Tech Stack Detector](https://apify.com/conserving_celerytop/website-tech-stack-detector) is an Apify Actor I built for lists. It uses the open fingerprint set, more than 7,600 technologies, and makes one homepage request per site, only when that site's `robots.txt` allows it. You send up to 10,000 domains or URLs per run.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import csv, os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("sites.txt") as f:
    sites = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/website-tech-stack-detector").call(
    run_input={"websites": sites, "maxSites": len(sites), "categories": ["CMS", "Ecommerce", "Analytics"]},
    max_total_charge_usd=Decimal("5.00"),
)

cols = ["domain", "status", "cms", "ecommerce", "analytics", "paymentProcessors"]
with open("tech_stack.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(cols)
    for row in client.dataset(run.default_dataset_id).iterate_items():
        w.writerow([row["domain"], row["status"]] + [", ".join(row.get(c) or []) for c in cols[2:]])
```

Leave out `categories` to get everything it finds. Each row has:

- `technologies`: every match with `name`, `version` when the page shows one, `confidence` from 0 to 100 and `categories`
- quick columns ready for a spreadsheet: `cms`, `ecommerce`, `analytics`, `frameworks`, `cdn`, `hosting`, `paymentProcessors`, `tagManagers`
- `status`: `ok`, or the reason a site was not checked, such as `robots_disallowed`, `blocked`, `timeout` or `domain_not_found`

The same run also works over plain HTTP with Apify's `run-sync-get-dataset-items` endpoint, if you would rather not install the client.

## What it costs

Store prices on September 27, 2026. Check the Store page before relying on them.

- $0.002 per website whose homepage loaded, which is $2 per 1,000, on the free and Starter plans. Scale pays $0.0018 and Business $0.0016.
- Apify's start event adds $0.00005 per run for each GB of memory.
- Sites closed by `robots.txt`, blocked, timed out or not found cost nothing, and neither do duplicates or invalid entries.
- A site that loads but shows no known technology is charged, since the page was fetched and checked.
- Example: 500 domains where 460 load cost $0.92.

## Honest limits

- **One page, no JavaScript.** It reads the homepage HTML and headers the server sends. Tools injected later by scripts can be missed. If you need those, run a headless browser, which costs more per site.
- **Homepage only.** A checkout platform used only on `shop.example.com` will not show on `example.com`.
- **No usage history.** It tells you what a site runs today, not when it switched or what it spends. Commercial databases that crawl the web continuously are the tool for "every site that used X since 2021".
- **robots.txt is respected.** Some large sites disallow all bots, and those come back as `robots_disallowed`, free.

## FAQ

**Is there a free tech stack lookup API?**
For a few sites, a browser extension or a script like the one above is free. Most hosted APIs charge per lookup or per month.

**Can I look up which companies use a technology?**
Not directly. Send a list of candidate domains and filter the results by technology. The Actor does not keep a crawl of the whole web.

**Does it collect personal data?**
No. It reads public homepages once per site and does not store cookie values or personal data.


## Related guides

- [Hiring Signals for Sales: Score Your Account List From Job Postings](/hiring-signals-for-sales/)
- [Greenhouse Jobs API: Get Every Open Job From a Board in Python](/greenhouse-jobs-api-python/)

---

*This article and the Actor are not affiliated with or endorsed by any company whose technology it detects. The fingerprint data is open source under GPL-3.0.*
