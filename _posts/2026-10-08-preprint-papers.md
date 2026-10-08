---
layout: post
title: "arXiv API: Preprint Metadata as JSON or CSV, by Keyword, Date or ID List"
subtitle: "How to search arXiv by keyword, category and date, look up a list of IDs, save the rows, and stay inside the 3-second rule"
description: "arXiv API guide: search by keyword, category and date, look up a list of IDs, and save JSON or CSV with Python. Plus a $2 per 1,000 rows Actor."
seo_title: "arXiv API: Get Preprint Metadata as JSON or CSV"
tags: ["arxiv", "arxiv-api", "preprints", "biorxiv", "medrxiv", "apify"]
permalink: /arxiv-lookup-by-id-list/
date: 2026-10-08 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The commands and outputs below were run on October 7, 2026. I am not affiliated with arXiv, Cornell University, bioRxiv or medRxiv.*

If you need the titles, abstracts, categories and dates of arXiv preprints in a table, you do not need a paid tool. arXiv has a public API that returns this metadata for free. This guide shows how to search by keyword, category and date, how to look up a list of IDs, and how to save the rows as JSON or CSV. It also shows the mistakes I would expect a first-time user to make.

## The rules of the free API

arXiv publishes its terms for the API. Three points matter:

- Make no more than one request every three seconds, and use one connection at a time. The limit applies to all your machines together.
- The descriptive metadata (title, abstract, dates, categories) is released under CC0. You may store and reuse it.
- Do not store and serve the PDFs or source files of the papers without the permission of the copyright holder. Link to them instead.

arXiv also asks you to credit it: "Thank you to arXiv for use of its open access interoperability."

## One request from the command line

The endpoint is `https://export.arxiv.org/api/query`. It answers with an Atom XML feed. This request asks for 3 papers in the category q-bio.NC that were first submitted in September 2025:

```bash
curl -sG "https://export.arxiv.org/api/query" \
  --data-urlencode 'search_query=cat:q-bio.NC AND submittedDate:[202509010000 TO 202509302359]' \
  --data-urlencode "max_results=3" -A "my-script/1.0"
```

The server returned HTTP 200. The feed says `totalResults` is 124 and contains 3 entries. The count is useful: it tells you how many rows a full export would have before you start one.

Search terms use field prefixes: `ti:` for the title, `abs:` for the abstract, `cat:` for the category and `all:` for every field. Join them with AND, OR and ANDNOT. `submittedDate:[start TO end]` takes the format YYYYMMDDHHMM. Add `sortBy=submittedDate&sortOrder=descending` for newest first, and `start` and `max_results` to page through the list.

## Save rows as CSV or JSON with Python

This script uses only the standard library. It runs a keyword, category and date search, waits three seconds, then looks up three papers by ID. The result goes to CSV on screen. To get JSON, use `json.dump(rows, f)` in place of the CSV writer.

```python
import csv, sys, time, urllib.parse, urllib.request
import xml.etree.ElementTree as ET

ATOM = "{http://www.w3.org/2005/Atom}"
OPEN = "{http://a9.com/-/spec/opensearch/1.1/}"
ARX = "{http://arxiv.org/schemas/atom}"

def fetch(params):
    url = "https://export.arxiv.org/api/query?" + urllib.parse.urlencode(params)
    req = urllib.request.Request(url, headers={"User-Agent": "preprint-metadata-example/1.0"})
    root = ET.fromstring(urllib.request.urlopen(req, timeout=60).read())
    total = int(root.find(OPEN + "totalResults").text)
    rows = []
    for e in root.findall(ATOM + "entry"):
        rows.append({
            "id": e.find(ATOM + "id").text.split("/abs/")[-1],
            "published": e.find(ATOM + "published").text[:10],
            "category": e.find(ARX + "primary_category").get("term"),
            "title": " ".join(e.find(ATOM + "title").text.split()),
        })
    return total, rows

def show(params):
    total, rows = fetch(params)
    print("totalResults:", total)
    w = csv.DictWriter(sys.stdout, fieldnames=["id", "published", "category", "title"])
    w.writeheader()
    w.writerows(rows)

query = 'abs:"retrieval augmented generation" AND cat:cs.CL AND submittedDate:[202509010000 TO 202509302359]'
show({"search_query": query, "sortBy": "submittedDate", "sortOrder": "descending", "max_results": 5})
time.sleep(3)   # arXiv asks for one request every 3 seconds
show({"id_list": "1706.03762,1810.04805,hep-th/9901001"})
```

Output on 7 October 2026:

```text
totalResults: 90
id,published,category,title
2510.02388v2,2025-09-30,cs.CL,Learning to Route: A Rule-Driven Agent Framework for Hybrid-Source Retrieval-Augmented Generation
2510.00261v1,2025-09-30,cs.CL,Retrieval-Augmented Generation for Electrocardiogram-Language Models
2510.00137v1,2025-09-30,cs.IR,Optimizing What Matters: AUC-Driven Learning for Robust Neural Retrieval
2509.26383v5,2025-09-30,cs.CL,Efficient and Transferable Agentic Knowledge Graph RAG via Reinforcement Learning
2509.26184v5,2025-09-30,cs.IR,Auto-ARGUE: LLM-Based Report Generation Evaluation
totalResults: 3
id,published,category,title
1706.03762v7,2017-06-12,cs.CL,Attention Is All You Need
1810.04805v2,2018-10-11,cs.CL,BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding
hep-th/9901001v3,1999-01-01,hep-th,String Junctions and Their Duals in Heterotic String Theory
```

The script does not print author names. For a table of topics and dates you rarely need them.

## Look up a list of IDs

The `id_list` parameter takes arXiv IDs separated by commas. Both styles work: new IDs such as 1706.03762 and old IDs such as hep-th/9901001, as the rows above show. If you have a column of IDs in a sheet, join them with commas and send them in one request. The version number in the output (v7 for the Transformer paper) is the latest version at the time of the request, so strip it if you want a stable key.

## Four mistakes to avoid

**Reading `cat:` as the main category.** The filter `cat:` matches any category a paper lists, including cross-lists. In a first run of my q-bio.NC query, one of three rows had cs.LG as its primary category. If you need papers whose main category is q-bio.NC, check the primary category field after you download.

**Expecting exact words.** arXiv matches word stems, so a search for "generation" can also match related word forms. In my Actor test of the retrieval augmented generation search, 75 of the 100 newest rows had all three exact words in the title or abstract. If exact words matter, filter the abstract text yourself.

**Ignoring the count.** `totalResults` is 90 for the search above and 124 for the category query. Read it first. A broad search can have thousands of matches, and at one request every three seconds a full export takes time.

**Running scripts in parallel.** The 3-second limit covers all your machines together. Run one script at a time, with the sleep.

## If you do not want to write the code

For lists with many rows, filters and a schedule, I built an unofficial Apify Actor called [arXiv Preprints API](https://apify.com/conserving_celerytop/preprint-papers-api). You type one search per line or paste a list of IDs, choose a date range, categories and sort order, and export JSON, CSV or Excel. It adds the 3-second pacing, reads the pages for you and returns one row per preprint with the title, abstract, categories, version, dates, links and the DOI of the journal version when arXiv lists one. An ID that matches nothing returns a `not_found` row.

It costs $2.00 per 1,000 rows on the free Apify plan, and less on paid plans. In my tests on my own computer, two runs of 1,000 rows took 28.6 and 28.9 seconds and made 10 requests each. A lookup of five well known paper IDs returned five rows in about one second and costs 5 x $0.002 = $0.01. For one search, the Python above does the same for free, and it is the better choice. The Actor helps when you want a daily schedule, a long list, or a result without running code.

## Limits

- The Actor does not return author names, affiliations or e-mail fields. This is by design, because they are personal data. The title and abstract are free text and can still contain a name.
- It does not return PDFs or full text, only links to them. It also leaves out the comment and journal reference fields of arXiv.
- The arXiv limit of one request every three seconds applies to all users of the service together. Many runs at the same time could be slowed or blocked. I have not tested this.
- The Actor can also read bioRxiv and medRxiv. I have not tested that part against the live service, so I make no claims about its results. Check a short run before you rely on it.

Data: arXiv (arxiv.org). Thank you to arXiv for use of its open access interoperability.
