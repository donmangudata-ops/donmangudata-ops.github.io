---
layout: post
title: "Which CVEs to Fix First? A KEV and EPSS Check in One Table"
subtitle: "How well CVSS works as a filter, what CISA KEV and the EPSS score add, and a four-step triage rule you can run each morning"
description: "Which CVEs to fix first: join NVD CVSS, the CISA KEV catalog and the EPSS score in one table, with real counts from 11 October 2026 and a triage rule."
seo_title: "Which CVEs to Fix First: CVSS, KEV and EPSS Together"
tags: ["cve", "cisa-kev", "epss", "vulnerability-management", "apify"]
permalink: /which-cves-to-fix-first-kev-epss/
date: 2026-10-12 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in this guide, and it is paid. Every number below can also be reproduced with a short script. This article was drafted with an AI assistant. All numbers come from runs on 11 October 2026.*

The National Vulnerability Database (NVD) held 404,209 CVE records on 11 October 2026. No team can fix all of them. The usual first filter is the CVSS score: fix the 9.0 and above first. This post checks how well that filter works, using two free signals that sit next to CVSS: the CISA Known Exploited Vulnerabilities (KEV) catalog and the FIRST EPSS score.

## The three signals

CVSS says how bad a flaw could be. It is a severity score from 0 to 10 and the NVD publishes it for most CVEs.

KEV says that attackers use a flaw now. CISA keeps a list of CVEs with proof of exploitation in the wild. On 11 October 2026 the list held 1,739 CVEs. CISA publishes the list under the CC0 1.0 licence.

EPSS says how likely exploitation is. FIRST publishes a daily score from 0 to 1 for each CVE. It is the estimated probability of exploitation in the next 30 days.

The three live on three different websites with three different formats. To compare them you join them on the CVE ID. I wrote an Apify Actor, CVE KEV EPSS API, that does this join and returns one row per CVE. I built it, so read this post with that in mind. Every number below is something you can also get with a short script of your own.

## What the join shows

I ran the Actor with the filter "Known exploited only" and a maximum of 2,000 rows. It returned 1,739 rows, one for each KEV entry. The run took 43 seconds, used 4 NVD requests, 18 EPSS requests and 1 KEV request, and peaked at 182 MB of memory.

Here is what the table says about CVSS as a filter:

- 1,546 of the 1,739 exploited CVEs have a CVSS score of 7.0 or higher. A "fix 7 and above" rule catches 89 percent of them.
- 627 have a score of 9.0 or higher. A "critical only" rule catches 36 percent.
- 193 exploited CVEs score below 7.0. Five of them score below 4.0. A CVSS rule skips all of them, although attackers use every one.
- 47 of those 193 also have an EPSS score of 0.5 or higher. They are low on severity and high on predicted exploitation.

Now the other direction. Of the 1,739 exploited CVEs, 79 have a CVSS score of 9.0 or higher and an EPSS score below 0.1. The score says critical. The model says unlikely. They are in KEV anyway, so exploitation is a fact for these.

## EPSS and KEV overlap, but not fully

I also read the top of the EPSS list. 797 CVEs have an EPSS score above 0.9. 466 of them are in KEV. The other 331 are not in KEV. These 331 have a very high predicted chance of exploitation and no entry in the catalog yet. A KEV-only list misses them. An EPSS-only list does not tell you which ones attackers already use.

The median EPSS score inside KEV is 0.49. 456 of the 1,739 KEV entries have an EPSS score below 0.1. EPSS is a prediction that is trained on past data, and old exploited flaws can get low scores. So the two signals answer two different questions, and you want both in one row.

## Ransomware and the pace of new entries

KEV has a field for known use in ransomware campaigns. 361 of the 1,739 entries, 20.8 percent, carry the value Known. A CVE with that flag deserves a faster ticket than the same CVSS score without it.

CISA added 34 entries in the 30 days before 11 October, and 43 in September 2026. In the first nine months of 2026 the monthly count ran between 17 and 43. That is about one new KEV entry per day. For a security team, this is a list short enough to read each morning.

## A simple triage rule from the table

You can build a rule from these columns without any model of your own:

1. First, anything with inKev set to true and a kevDueDate in the past or within the next 7 days. These are the urgent ones.
2. Next, anything with kevRansomwareUse set to Known.
3. Next, anything with an EPSS score of 0.5 or higher that is not yet in KEV. Watch it every day.
4. Last, the rest by CVSS score.

Run the Actor each morning with "KEV added in the last days" set to 1 and a few rows come back. The cost on the Free plan is $0.003 per row plus $0.00005 for the start, so a day with 2 new entries costs $0.00605.

## How to run it

1. Open the Actor and leave the default input: KEV added in the last 30 days, 10 rows. The run took 1 second and returned 10 rows.
2. Change "KEV added in the last days" to 7 for a weekly digest.
3. Fill "Minimum EPSS score" with 0.9 to read the top of the EPSS list.
4. Enter CVE IDs, one per line, to check your own list. Each ID takes one NVD request, 6 seconds apart. 100 IDs take about 10 minutes.
5. Download JSON, CSV or Excel.

## What to keep in mind

The NVD asks for this notice: "This product uses data from the NVD API but is not endorsed or certified by the NVD." The Actor is not affiliated with NIST, CISA or FIRST. The NVD public limit without a key is 5 requests in 30 seconds, and the Actor sleeps 6 seconds between NVD requests. When the NVD refuses a request, the Actor waits, tries twice and then stops with a clear message. It does not rotate IP addresses and does not use a key.

New CVEs often have no CVSS score for days. Their cvssScore is null, and the row is still returned. If you want only scored CVEs, set the minimum CVSS score.

The numbers above are one day of data. EPSS changes daily and the KEV catalog grows. Run the same filters again next week and the counts will differ.

The Actor page is [CVE KEV EPSS API on the Apify Store](https://apify.com/conserving_celerytop/cve-kev-epss-api). A free Python version of the same join is in the [skill file on GitHub](https://github.com/donmangudata-ops/web-data-skills/tree/main/skills/cve-kev-epss-api).
