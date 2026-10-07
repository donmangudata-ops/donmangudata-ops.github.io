---
layout: post
title: "Lighthouse Audit: Run It on One Page or a Hundred"
subtitle: "How to run a Lighthouse audit from Chrome and from the command line, how to read the scores, and what to do when the list gets long"
description: "How to run a Lighthouse audit from DevTools or the command line, read the scores and Core Web Vitals, and audit many URLs. Plus a $15 per 1,000 audits Actor."
seo_title: "Lighthouse Audit: How to Run and Read It"
tags: ["lighthouse", "core-web-vitals", "page-speed", "seo", "apify"]
permalink: /lighthouse-audit/
date: 2026-10-07 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The commands and outputs below were run on October 4, 2026, on a small test page served from the same machine, except the DevTools steps, which I did not run for this article.*

A Lighthouse audit tests one page load in a controlled setting and gives you four scores from 0 to 100: performance, accessibility, best practices and SEO. It also reports the Core Web Vitals that Google looks at for page experience. Lighthouse is open source (Apache 2.0), so you can run it yourself for free. This guide shows the two usual ways, how to read the numbers, and where running it by hand stops being practical.

## Method 1: Chrome DevTools

Open the page in Chrome, open DevTools, and choose the Lighthouse panel. Pick Mobile or Desktop, tick the categories you want, and start the audit. The report opens in the same panel. This is the quickest way to look at one page. Use a clean browser profile or a private window, because extensions change the result.

## Method 2: the command line

The command line version is the one you can repeat and script. You need Node 22.19 or newer for Lighthouse 13, and Chrome installed.

```bash
npx lighthouse "https://example.com/pricing" \
  --output=json --output-path=report.json --quiet \
  --only-categories=performance,accessibility,best-practices,seo \
  --chrome-flags="--headless=new"
```

Add `--preset=desktop` for the desktop profile. The default is mobile. On a machine where you run as root, such as a container, add `--no-sandbox` to the Chrome flags.

I ran this against a small local test page that has a render-blocking stylesheet, a render-blocking script and a 2.8 MB image shown at 300 pixels wide. Reduced with `jq`, the result was:

```json
{
  "scores": { "performance": 75, "accessibility": 80, "best-practices": 96, "seo": 82 },
  "lcp_ms": 15169,
  "cls": 0,
  "tbt_ms": 0,
  "fixes": [
    { "id": "image-delivery-insight", "summary": "Est savings of 2,784 KiB" },
    { "id": "render-blocking-insight", "summary": "Est savings of 530 ms" }
  ]
}
```

The large image is the reason for the slow Largest Contentful Paint, and Lighthouse names it as the first fix. That is the useful part of the report: the fixes are ranked by the time they should save.

## How to read the scores

The four category scores are weighted summaries, and Lighthouse colors them green from 90, orange from 50 and red below. For the Core Web Vitals, use the usual limits:

| Metric | Good | Needs improvement | Poor |
|---|---|---|---|
| Largest Contentful Paint (LCP) | up to 2.5 s | up to 4 s | over 4 s |
| Cumulative Layout Shift (CLS) | up to 0.1 | up to 0.25 | over 0.25 |
| Total Blocking Time (TBT) | up to 200 ms | up to 600 ms | over 600 ms |

TBT is a lab stand-in for responsiveness. Real-user responsiveness is measured by Interaction to Next Paint, which a single lab run cannot see.

## Why the same page scores differently

Lighthouse is a lab test, so timing and machine load move the numbers. I audited the same page three times and got performance scores of 73, 74 and 75. A change of one or two points is noise. Run the audit twice or three times, keep the device and settings the same, and look for changes that are larger than the spread you see.

## Auditing many URLs with a script

For a short list, a loop over a text file is enough. This script writes one CSV line per URL and marks the pages that failed. I tested it on three local addresses, one of which returned a 404 page.

```bash
#!/bin/bash
echo "url,performance,accessibility,best_practices,seo,lcp_ms,cls,tbt_ms" > scores.csv
while read -r url; do
  npx lighthouse "$url" --output=json --output-path=report.json --quiet \
    --only-categories=performance,accessibility,best-practices,seo \
    --chrome-flags="--headless=new" || { echo "$url,error" >> scores.csv; continue; }
  jq -r --arg u "$url" '[$u, (.categories.performance.score*100|round), (.categories.accessibility.score*100|round), (.categories["best-practices"].score*100|round), (.categories.seo.score*100|round), (.audits["largest-contentful-paint"].numericValue|round), .audits["cumulative-layout-shift"].numericValue, (.audits["total-blocking-time"].numericValue|round)] | @csv' report.json >> scores.csv
  sleep 2
done < urls.txt
```

The 404 page ended the Lighthouse run with a runtime error and no report, and the script wrote an `error` line for it. This is the case to plan for: a page that does not load gives no scores.

## Where the manual way stops

The script runs one audit at a time. In my test, three audits took 35 seconds on a local page, and a real site takes longer. A list of 200 pages on mobile and desktop is 400 audits, and you need a machine with Chrome, enough memory and some way to retry the pages that fail. If you check the same pages every week, you also want the rows in one table, not 400 report files.

## A hosted option

The Actor [Lighthouse Audit API](https://apify.com/conserving_celerytop/lighthouse-audit-api) runs the same Lighthouse engine on Apify. You paste a list of public URLs, choose mobile, desktop or both, and get one row per URL and device: the four scores, LCP, CLS and TBT with a good, needs improvement or poor rating, page weight, and up to ten ranked fixes. It costs $15 per 1,000 audits on the free plan, and less on paid Apify plans. A page that cannot be scored returns an error row, and error rows are free. Two audits run at the same time by default.

## Limits

- It is a lab test. It does not show what real visitors experience, and it cannot test pages behind a login.
- Only public pages you own or have permission to test belong in the list.
- The scores are not exact. Repeat an audit before you act on a small change.
- A hosted run does not change what Lighthouse measures. It saves the work of running Chrome, retrying and collecting the results.
