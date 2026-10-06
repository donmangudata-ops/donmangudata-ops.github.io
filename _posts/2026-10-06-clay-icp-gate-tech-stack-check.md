---
layout: post
title: "A Cheap ICP Gate in Clay Before You Pay for Enrichment"
subtitle: "One HTTP API column that checks tech stack and email provider first, so paid steps run only on rows that fit"
description: "Add a cheap tech stack and DNS check to a Clay table with an Apify Actor, then run paid enrichment only on rows that fit your ICP. Costs, setup and limits."
seo_title: "Clay ICP Gate: Tech Stack Check Before Enrichment"
tags: ["clay", "apify", "api", "lead-enrichment", "tech-stack"]
permalink: /clay-icp-gate-tech-stack-check/
date: 2026-10-06 05:50:00 +0000
published: true
---
Expensive lookups should not run on every row. Any row that does not fit your ICP only adds cost. A gate is a cheap check that runs first and decides which rows get the paid steps.

This post builds that gate with my Apify Actor, Website Tech Stack Detector, called from a Clay HTTP API column. I have not run it inside a Clay table, because a real HTTP API test needs a paid Clay plan and I have not bought one. I ran the Actor itself with the same one-domain input on Apify. Where a Clay detail comes from Clay's own docs, I say so. Where I could not confirm it, I say that too and tell you what to look for in your own table.

## One call covers the tech check and the DNS check

The Actor loads one homepage per domain and matches it against 7,628 open fingerprints from the webappanalyzer project. It returns the CMS, ecommerce platform, analytics, frameworks, CDN, hosting, payment processors and tag managers. It also reads the domain's public DNS records and reports the email provider, the MX hosts, whether an SPF record exists and the DMARC policy, with no extra request to the website.

So the DNS side of the gate comes in the same row. My separate DNS Lookup Actor exists, but it takes a list of domains and I have not built a one-row mode for it, so I would skip it here.

## What it costs

The Actor charges $0.002 per website whose homepage loaded, at the base price tier. Sites that cannot be read return a row with the reason and are not charged. Apify adds a start event of $0.00005 per run, and a row-by-row call is one run, so a loaded site costs about $0.00205. My own test runs showed $0.000 in the Console, so I have not seen these amounts on a real bill.

Clay counts the HTTP API call by its own rules, and those can change. Clay's pricing page decides how an HTTP API call is counted and which plan includes the column, so read it before you size a table.

## Set it up

1. Copy your Apify API token from Console, under Settings, then API & Integrations.
2. In Clay, click Add enrichment, search for HTTP API and select it. That is how Clay's HTTP API guide describes it.
3. Method POST, URL:

```
https://api.apify.com/v2/acts/conserving_celerytop~website-tech-stack-detector/run-sync-get-dataset-items
```

This URL pattern is the one in the Actor's own README, where it is shown with a list input. Apify's API docs describe the endpoint as one that runs an Actor and returns its dataset items, with a default timeout of 300 seconds. I ran the Actor with the single `domain` field on Apify, not through this URL, so test it once with curl before you point a table at it.

4. Headers: `Authorization` with `Bearer <YOUR_APIFY_TOKEN>`, and `Content-Type` with `application/json`. Apify's API docs recommend the Bearer header over putting the token in the URL. Clay's HTTP API guide says headers can be saved as a reusable account, which it says is encrypted at workspace level, or typed into the Headers field, where the key sits in plain text. I would use the saved account.
5. Body:

```json
{
    "domain": "stripe.com"
}
```

Replace the example with a reference to your domain column. Clay's HTTP API guide writes a column reference as a slash followed by the column name, and it says string values stay inside quotes. So the line in your table would look like `"domain": "/Domain"`, with your own column name. Clay's separate Apify integration page gives the opposite rule for that integration, with the token unquoted. They are two different column types, and I followed the HTTP API guide. Run one row and read the raw request before you run the rest. A full URL such as `https://www.stripe.com/pricing` also works, and the Actor checks the homepage. The `domain` field is used only when the `websites` list is empty.

## What comes back

My Stripe test took 5 seconds and returned one row with 40 fields. Each value is plain text, a number, true/false or null, with technology lists joined by commas. Here is a trimmed version with only the values I recorded:

```json
[
    {
        "domain": "stripe.com",
        "status": "ok",
        "pageTitle": "Stripe | Financial Infrastructure to Grow Your Revenue",
        "emailProvider": "Google Workspace",
        "charged": true
    }
]
```

The page title matched the live page, and the email provider matched the live MX records, which pointed at Google. The other fields exist but I did not note their values, so I leave them out rather than guess. Apify's API docs show this endpoint returning an array of dataset items, so expect a one-item array around the row. The `domain` value in your table is whatever your column holds. Clay's HTTP API guide has a Field paths to return setting that takes dot notation. I have not seen how it handles a one-item array, so run one row, look at the raw response and pick the path from what you see.

## Turn it into a gate

Decide your ICP in terms of columns the row already has. For example, a company that sells online has a value in `ecommerce`. A company that runs email on Google Workspace or Microsoft 365 shows that in `emailProvider`. A domain with no MX hosts probably has no working mailbox. Then write a rule that is true only when `status` is `ok` and your ICP rule holds, and run the paid steps only where it is true. Clay's HTTP API guide mentions conditional run formulas for that column. I did not check Clay's formula syntax or the same setting on other enrichment columns. Look for a run condition on each paid column and test your rule on a few rows first.

Several third-party Clay guides recommend filtering before expensive lookups. Their credit figures disagree with each other, so I quote none.

## Where it falls short

The Actor reads the first response from the server and does not run JavaScript. A chat widget or an analytics tool loaded later through a tag manager may be missed. It checks the homepage only, and only when robots.txt allows it, so some sites come back with `robots_disallowed`, `blocked` or `timeout` in `status`. Treat those as unknown, not as a failed ICP test.

It is not BuiltWith or Wappalyzer, and it is not affiliated with either. Coverage differs, so test it on a sample of your own list first.

I am not affiliated with Clay, and Clay has not reviewed or endorsed this post.
