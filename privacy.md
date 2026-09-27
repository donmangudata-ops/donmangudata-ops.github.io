---
layout: page
title: Privacy policy
description: "Privacy policy for the Jobs MCP Server: what data it processes, where it goes, how long it is kept, and how to contact us."
permalink: /privacy/
---
*Effective date: September 27, 2026. Last updated: September 27, 2026.*

This policy covers the **Jobs MCP Server** (the "connector"), a remote MCP server published by Don Mangu on the Apify platform, and this website. "We" means Don Mangu. Contact: [don.mangu.data@gmail.com](mailto:don.mangu.data@gmail.com).

## What the connector processes

When you use the connector, it processes:

1. **Tool inputs.** What your AI client sends in a tool call: company names, job board links or websites, search words, locations and filters, page size and cursor. The tools do not ask for personal data, and you should not put personal data in them.
2. **Your Apify API token.** Your client sends it to the Apify platform, which checks it and starts the connector in your own Apify account. The connector uses the access the platform gives that run to start the job searches in your account. The connector does not read, copy, log or store your token.
3. **Results.** The job listings returned by the searches: public job postings from company career pages, such as job title, location, salary when published, posted date and links.
4. **Technical data.** The Apify platform keeps its own records of runs, such as start time, status and usage, under your Apify account.

## How the data is used

Tool inputs are used only to run the search you asked for and return its results to your AI client. We do not use your inputs or results for anything else, do not build profiles, and do not use them to train models.

## Where the data goes

- **Apify.** The connector and the searches run on the Apify platform (Apify Technologies s.r.o.) under your own Apify account and are billed there. Apify's handling of your account data is covered by [Apify's privacy policy](https://apify.com/privacy-policy).
- **Public job boards.** To read open jobs, the searches request the public career pages and job board APIs of the companies you name or of the companies in our lists (for example Greenhouse, Lever, Ashby or Workday). Those requests contain the company's board name or website, not your inputs as a whole and not your token.
- **Your AI client.** Results go back to the AI application you connected, such as Claude. How that application handles your conversations is covered by its own privacy policy.

We do not sell, rent or share your data with anyone else, and we do not show ads.

## Storage and retention

- The connector itself keeps no copy of your inputs, results or token after it answers a call. It writes no logs that contain them; its logs hold only the tool name, the number of results and timing.
- The search runs store their inputs and results in your own Apify account, like any Actor run you start. They are kept for your Apify plan's data retention period (7 days for unnamed storages on the free plan at the time of writing) and you can delete them in Apify Console at any time.
- As the Actor developer, we can see aggregate statistics that Apify gives developers (such as number of runs and users). We do not use them to identify you.
- Emails you send to our support address are kept as long as needed to answer them, and at most 2 years.

## Your choices and rights

You can stop using the connector at any time by removing it from your AI client, and revoke your Apify token in Apify Console. You can delete the stored runs in your Apify account. For any question about your data, or to ask us to delete support emails, write to us. If you are in the EU or the UK, you can also complain to your data protection authority.

## Security

Your token travels only over HTTPS, from your client to the Apify platform. We ask you to send it in the `Authorization` header, not in the URL, so it does not appear in logs or browser history. Report security issues to [don.mangu.data@gmail.com](mailto:don.mangu.data@gmail.com).

## Children

The connector is not meant for children under 16.

## Changes

We will post any change to this policy on this page and update the date above.

## Contact

Don Mangu, [don.mangu.data@gmail.com](mailto:don.mangu.data@gmail.com), [donmangudata-ops.github.io](https://donmangudata-ops.github.io).
