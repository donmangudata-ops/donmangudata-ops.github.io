---
layout: post
title: "RSS to JSON API: Read Many Feeds in Python"
subtitle: "A free feedparser script first, then a hosted API for long lists of RSS and Atom feeds"
description: "Turn RSS and Atom feeds into JSON rows in Python: a free feedparser script, and a $1 per 1,000 items API for long feed lists with date filters."
seo_title: "RSS to JSON API: Read Many Feeds in Python"
tags: ["python", "api", "rss", "apify"]
permalink: /rss-to-json-api/
date: 2026-10-02 00:00:00 +0000
cadence_day: 4
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

An RSS to JSON API takes a feed address and gives you the items as structured data, so a script, an automation or an AI agent can use them without parsing XML. The formats are old and stable, which is why this is a small job for one feed and an annoying one for two hundred.

This guide shows the free way first, then a hosted option for long lists.

## What you are converting

RSS 2.0, RSS 1.0 and Atom all describe a list of items. Each item usually has a title, a link, a date and a short summary. The trouble is in the details:

- Dates come in different formats, and some feeds leave them out.
- Summaries often contain HTML.
- Links can be relative.
- Some feeds are served gzipped, and some addresses return an HTML page instead of a feed.

A good converter normalizes these and tells you which feeds failed.

## The free way: feedparser

The `feedparser` library reads all three formats and handles most of the quirks above.

```python
import json
import feedparser

FEEDS = [
    "https://www.nasa.gov/news-release/feed/",
    "https://www.gov.uk/search/news-and-communications.atom",
]

rows = []
for url in FEEDS:
    d = feedparser.parse(url, agent="feed-example/0.1 (+https://example.com/contact)")
    if d.bozo and not d.entries:
        rows.append({"feedUrl": url, "status": "feed_error"})
        continue
    for e in d.entries[:5]:
        rows.append({
            "feedUrl": url,
            "feedTitle": d.feed.get("title"),
            "title": e.get("title"),
            "url": e.get("link"),
            "publishedAt": e.get("published"),
            "summary": e.get("summary"),
            "categories": [t.term for t in e.get("tags", [])],
        })

print(json.dumps(rows, indent=2)[:1500])
```

This is enough for a few feeds. Things you will add as the list grows: a robots.txt check, a request timeout, parallel fetching, date parsing into one format, HTML stripping, and a record of which feeds are dead so you do not read them forever.

## A hosted option: RSS Feed Reader

[RSS Feed Reader](https://apify.com/conserving_celerytop/rss-feed-reader) is an Apify Actor I built for lists of feeds. It reads RSS 2.0, RSS 1.0 and Atom, including gzipped feeds, and returns one row per item, newest first, with dates in ISO 8601, HTML removed from summaries and relative links made absolute.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("feeds.txt") as f:
    feeds = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/rss-feed-reader").call(
    run_input={
        "feedUrls": feeds,
        "maxItemsPerFeed": 25,
        "publishedSince": "7 days",
    },
    max_total_charge_usd=Decimal("2.00"),
)

items = list(client.dataset(run.default_dataset_id).iterate_items())
good = [r for r in items if r["status"] == "ok"]
bad = [r for r in items if r["status"] != "ok"]
print(len(good), "items,", len(bad), "feeds with problems")
```

Useful input fields:

- `feedUrls`: the feed addresses.
- `maxItemsPerFeed`: 1 to 5,000, default 50.
- `publishedSince`: a date such as `2026-09-01`, or a period such as `24 hours` or `7 days`.
- `includeContent`: adds the full item text when the feed carries it.
- `maxResults`: a cap on items for the whole run.

## What the output looks like

```json
{
  "feedUrl": "https://www.nasa.gov/news-release/feed/",
  "feedTitle": "NASA",
  "title": "Uncovering the Valleys Hidden Below Greenland's Ice",
  "url": "https://science.nasa.gov/earth/earth-observatory/uncovering-the-valleys-hidden-below-greenlands-ice/",
  "publishedAt": "2026-09-28T04:00:00.000Z",
  "summary": "Greenland is capped with a vast ice sheet ...",
  "categories": ["Earth"],
  "status": "ok",
  "charged": true
}
```

A feed that cannot be read gives one row with a `status` of `feed_error`, `not_a_feed`, `robots_disallowed` or `not_found`, plus the reason in `error`. Those rows are free. Author names are not returned.

## What it costs

Store price on October 2, 2026. Check the Store page before relying on it.

- $0.001 per item, which is $1 per 1,000 items.
- Apify adds $0.00005 per run for each GB of memory at start.
- Broken feeds, addresses that are not feeds and invalid entries cost nothing.
- Example from the Actor page: 40 feeds, 25 new items from each, once a day. That is 1,000 items, or $1.00 a day, about $30 a month.
- You can set a spending limit on a run, as in the script above. The Actor stops when it is reached.

To read only what is new, set `publishedSince` to the length of your schedule and run it daily. Without that, you pay again each day for items you already have.

## Alternatives

- **feedparser alone.** Free, and the right choice if you have a handful of feeds and a server of your own. You run and maintain it.
- **A feed reader with an API**, such as Feedly or Inoreader. They are built for people who read, they are priced as subscriptions, and API access has its own terms. Check them if you also want a reading interface.
- **Hosted RSS to JSON converters.** Several exist, usually with a free tier that has a request limit. Compare the limits and whether they give you a date filter and failure reporting.
- **n8n, Make or Zapier feed nodes.** Good for triggering a workflow when one feed changes. Less comfortable for a table built from hundreds of feeds.

## Limits

- It reads feed content only, not the articles the items link to. For the article text you need a separate extractor.
- Items without a date cannot be filtered by `publishedSince`.
- It follows each site's robots.txt, so a feed the site disallows comes back free and empty.
- Full content is only there when the feed includes it.

## FAQ

**What is the easiest way to convert RSS to JSON in Python?**
Use `feedparser`, as in the script above, and dump the entries with `json.dumps`. It handles RSS and Atom.

**Can I convert many RSS feeds at once?**
Yes. Loop over the list, ideally with a thread pool and a timeout per request, or send the whole list to a hosted API.

**How do I get only new items from a feed?**
Keep the newest date or link you saw last time and drop older ones. With the Actor, set `publishedSince` and schedule the run.

**Does RSS Feed Reader need a login or API key for the feeds?**
No. It reads public feeds. You need an Apify token to call the Actor.
