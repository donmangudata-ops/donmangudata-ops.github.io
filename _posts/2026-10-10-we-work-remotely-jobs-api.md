---
layout: post
title: "We Work Remotely Jobs API: How to Get the Feed as Clean Rows"
subtitle: "There is no official key and no docs, only a public RSS feed with one rule. Here is what it holds, a free script route, and when a tool is worth it"
description: "How to read We Work Remotely jobs and RemoteOK jobs as clean rows: RSS fields, a free script route, attribution rules, new-only alerts and what a hosted Actor costs."
seo_title: "We Work Remotely Jobs API: RSS Feed to Clean Rows"
tags: ["we-work-remotely", "remoteok", "jobs-api", "rss", "apify"]
permalink: /we-work-remotely-jobs-api/
date: 2026-10-10 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in this guide, and it is paid. A free script route is covered too. This article was drafted with an AI assistant. The sample row below comes from a run on recorded test data in the live layout, not from the live sites. I am not affiliated with We Work Remotely or Remote OK.*

We Work Remotely has no official jobs API with keys and docs. What it has is a public RSS feed, and its feed page says: "Anyone can use the feed, all we ask is that you attribute the links back to We Work Remotely". That sentence is the licence for everything below.

## What the feed holds

There is one RSS feed per category, such as programming, design, devops or customer support. Each file lives at a path like weworkremotely.com/categories/remote-programming-jobs.rss. When I read the programming feed on 9 October 2026, each item had these child elements: title, region, category, description, pubDate, guid and link. The all-jobs feed also adds country, state, skills, type and expires_at. The title has the form "Company: Job title", so you split it at the first colon.

Two facts matter for your code. The description is HTML, and it can hold a contact email or phone number. Strip those before you store or show the text. And the feed carries no salary, so you cannot filter by pay on this source.

## Get the rows with a script

With Python and only the standard library you fetch a feed, loop over the item elements, split the title and keep the link. Wait one second between requests and send a User-Agent that names your tool and your contact email. Stop on a 403. The free Agent Skill that goes with the Actor holds a short, tested script for this. It works on both We Work Remotely and the RemoteOK API.

The script is fine for one feed. The work starts when you want all eleven categories, deduplication across feeds (the programming feed already holds three sub-feeds), a date filter and a daily check for unseen jobs. That is where scripts grow.

## Add RemoteOK

RemoteOK offers a public JSON API at remoteok.com/api. The first element of the list is a legal notice, not a job. The notice asks you to "link back (with follow, and without nofollow!) to the URL on Remote OK and mention Remote OK as a source". A job has position, company, tags, location, salary_min, salary_max, date, url and apply_url. A salary of 0 means none. RemoteOK has no category field, so you group jobs by their tags, and that grouping is approximate.

## What a clean row looks like

The Actor returns one row per job with the same fields for both sources. This sample comes from a run on recorded test data in the live layout, not from the live sites, which the build environment could not reach. Treat the values as examples:

- source: Remote OK
- title: Senior Backend Engineer
- company: Acme Data
- category: Programming (derived from tags)
- salaryMin and salaryMax: 120000 and 160000
- postedAt: 2026-10-08T04:26:40.000Z
- sourceUrl: the link to the listing on the source site
- attribution: "Source: Remote OK (https://remoteok.com)"

In a test run with a programming filter over two sources, 7 jobs were read and 4 matched the filter: 3 from We Work Remotely and 1 from RemoteOK. The job text had its email address and phone number replaced by a marker.

## Attribution in practice

Both sources ask for a link back. Show the source name next to every job and link the job title to its sourceUrl. For RemoteOK, use a normal followed link, and do not use the Remote OK logo, which RemoteOK states is a registered trademark. The attribution field gives you a ready line. This is not legal advice.

## Only the new jobs

For a daily alert you want the jobs that appeared since yesterday. The Actor keeps a list of delivered jobs per monitor name in your own Apify account. The first run returns everything and sets a baseline. Later runs return only jobs that were not delivered before. A job cut off by your result limit stays new for the next run.

## What it costs

The Actor costs $1.00 per 1,000 jobs, which is $0.001 per job, plus $0.00005 per run. A run that returns 100 jobs costs $0.10005. A run that finds nothing costs only the start fee. On 9 October 2026 I read the Store prices of remote job Actors. Five that read We Work Remotely or RemoteOK charge $0.001 per job. One charges $0.004 per listing and $0.035 for the detail page, and one charges $0.015 per job. The price is not the difference. The difference is the new-only mode, the category filter and the link back in every row.

## Run it

1. Open the Actor and pick the sources and categories.
2. Set Max results to 20 for a first test.
3. Start the run and open the Jobs view.
4. For a daily feed, schedule the run and keep the same monitor name.

The Actor is: https://apify.com/conserving_celerytop/remote-job-feeds-api. For jobs read straight from company career pages instead of job boards, see https://apify.com/conserving_celerytop/remote-jobs-search. This is an independent tool and is not affiliated with We Work Remotely or Remote OK.
