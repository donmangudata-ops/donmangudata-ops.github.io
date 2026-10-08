---
layout: post
title: "Sitemap Extractor Returns No URLs? 6 Causes and How to Check Each One"
subtitle: "Most sites have a sitemap. The tool just looked in the wrong place or could not read the file. Here is how to find out which, with a short Python script"
description: "Why a sitemap extractor returns zero URLs: wrong path, sitemap index, gzip, plain-text sitemaps, blocked requests, no sitemap. A check for each and a Python script."
seo_title: "Sitemap Extractor Returns No URLs: 6 Causes and Fixes"
tags: ["sitemap", "sitemap-extractor", "seo", "python", "apify"]
permalink: /sitemap-extractor-no-urls/
date: 2026-10-09 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The Python script below was run on October 8, 2026, against local test files (a sitemap index, a gzipped sitemap and a text sitemap), not against a live site.*

You run a sitemap extractor on a site. The run ends with zero URLs, or with far fewer than the site has. In most cases the site has a sitemap. The tool just looked in the wrong place or could not read the file.

Here are the six causes I see most, and a quick check for each.

## 1. The sitemap is not at /sitemap.xml

Many sites use another path, such as `/sitemap_index.xml`, `/wp-sitemap.xml` or `/sitemaps/main.xml`.

**Check:** open `https://example.com/robots.txt` and look for lines that start with `Sitemap:`. That is the path the site declares. A good extractor reads robots.txt first.

## 2. The file is a sitemap index, not a sitemap

A sitemap index lists other sitemap files, not pages. If the tool reads only the first file, you get a short list of `.xml` links and no pages.

**Check:** open the file. If you see `<sitemapindex>` and `<sitemap><loc>` tags, it is an index. The tool must follow each `<loc>` and read those files too.

## 3. The sitemap is gzipped

Large sites often serve `sitemap.xml.gz`. A tool that does not unpack gzip sees binary data and finds nothing.

**Check:** the URL ends in `.gz`, or the file starts with the bytes `1f 8b`.

## 4. The sitemap is plain text

The sitemap protocol also allows a `.txt` file with one URL per line. XML-only parsers skip it.

**Check:** open the file. If you see bare URLs and no tags, it is a text sitemap.

## 5. The server blocks the request

Some sites return 403 or a bot check to unknown clients. The run "succeeds" with no data.

**Check:** look at the run log for 403, 429 or an HTML page where XML was expected. If the site blocks automated access on purpose, respect that.

## 6. The site has no sitemap

Some sites have none. No tool can extract URLs from a file that does not exist. You then need a crawler that follows links instead.

**Check:** robots.txt has no `Sitemap:` line, and `/sitemap.xml` returns 404.

## A Python script that handles causes 1 to 4

It reads robots.txt, follows sitemap indexes, unpacks gzip and reads text sitemaps. Standard library only. Put a real contact in the User-Agent.

```python
import gzip
import xml.etree.ElementTree as ET
from urllib.request import Request, urlopen

NS = "{http://www.sitemaps.org/schemas/sitemap/0.9}"
HEADERS = {"User-Agent": "sitemap-check/1.0 (+contact: you@example.com)"}


def fetch(url):
    with urlopen(Request(url, headers=HEADERS), timeout=30) as r:
        data = r.read()
    if url.endswith(".gz") or data[:2] == b"\x1f\x8b":
        data = gzip.decompress(data)
    return data


def sitemaps_from_robots(site):
    text = fetch(site.rstrip("/") + "/robots.txt").decode("utf-8", "replace")
    found = [line.split(":", 1)[1].strip() for line in text.splitlines()
             if line.lower().startswith("sitemap:")]
    return found or [site.rstrip("/") + "/sitemap.xml"]


def urls_from_sitemap(url, seen=None):
    seen = seen if seen is not None else set()
    if url in seen:
        return []
    seen.add(url)
    data = fetch(url)
    if not data.lstrip().startswith(b"<"):  # plain-text sitemap
        return [l.strip() for l in data.decode("utf-8", "replace").splitlines() if l.strip()]
    root = ET.fromstring(data)
    if root.tag == NS + "sitemapindex":  # follow every child sitemap
        out = []
        for loc in root.iter(NS + "loc"):
            out += urls_from_sitemap(loc.text.strip(), seen)
        return out
    return [loc.text.strip() for loc in root.iter(NS + "loc")]


if __name__ == "__main__":
    import sys
    site = sys.argv[1]
    urls = []
    for sm in sitemaps_from_robots(site):
        urls += urls_from_sitemap(sm)
    print(len(urls), "URLs")
    print("\n".join(urls[:10]))
```

Run it as `python sitemap_urls.py https://example.com`. On my test files (an index that points to a gzipped XML sitemap and a text sitemap) it returned all 4 URLs.

## Don't pay for an empty run

Cause 6 is common. Before you pick a hosted tool, check how it charges when a site has no sitemap. My [Sitemap Extractor](https://apify.com/conserving_celerytop/sitemap-url-extractor?utm_source=blog&utm_medium=guide&utm_campaign=sitemap-no-urls) on Apify handles causes 1 to 4 and adds last-modified dates and a path filter. It charges $0.50 per 1,000 URLs, and sites without a sitemap and invalid entries are free.
