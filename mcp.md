---
layout: page
title: Jobs MCP Server
description: "Documentation for the Jobs MCP Server: two read-only job tools for Claude and other AI agents, how to connect with your own Apify token, pricing, limits and support."
permalink: /mcp/
---
*Last updated: October 6, 2026. The server is live on Apify as the Actor [Jobs MCP Server](https://apify.com/conserving_celerytop/jobs-mcp-server).*

The Jobs MCP Server is a remote MCP server that gives Claude and other AI agents two read-only tools for job data:

- **Open jobs of named companies**, read live from their career pages.
- **A search of jobs open today** at about 820 tech, AI, remote-first, European tech and startup companies.

It runs on the Apify platform. You connect with your own Apify API token, so every search runs in your own Apify account and is billed there. We never receive or store your token.

## Tools

### get_company_jobs

Returns the open jobs of 1 to 25 companies. Each company can be a job board link (such as `boards.greenhouse.io/stripe`), a website (`stripe.com`) or a name (`stripe`). It reads Greenhouse, Lever, Ashby, Workday, Workable and 17 more job boards.

| Input | Meaning |
|---|---|
| `companies` | 1 to 25 job board links, websites or company names. Required unless `cursor` is given |
| `titleIncludes` | Keep jobs whose title has one of these words or phrases |
| `location` | A city, US state, Canadian province, country or country code; several separated by commas |
| `remoteOnly` | Keep only fully remote jobs |
| `maxJobsPerCompany` | At most this many jobs per company, newest first. Default 100, at most 1,000 |
| `pageSize`, `cursor` | Page size (default 25, at most 100) and the `nextCursor` of the previous result |

### search_tech_jobs

Searches jobs open today at about 820 companies, newest first. At least one filter is required.

| Input | Meaning |
|---|---|
| `keywords` | Words or phrases in the job title, such as `data engineer` |
| `location`, `remoteOnly` | Place, and fully remote jobs only |
| `workplaceTypes` | `remote`, `hybrid`, `onsite` |
| `seniorities` | `intern`, `entry`, `mid`, `senior`, `staff_principal`, `lead_manager`, `director`, `vp`, `c_level` |
| `jobFunctions` | 18 functions, such as `engineering`, `data`, `product`, `sales` |
| `companyLists` | `ai-companies`, `tech-companies`, `remote-first`, `europe-tech`, `startups`. Default: all five |
| `postedSince` | A date such as `2026-09-01`, or a period such as `7 days` |
| `hasSalary` | Keep only jobs that publish pay |
| `maxResults` | The most matching jobs to find. Default 50, at most 500 |
| `pageSize`, `cursor` | As above |

### What a result looks like

Each result has a list of `jobs` (company, title, department, location, country code, workplace type, seniority, job function, salary when published, posted date, job link and apply link), `totalRows`, and `nextCursor` for the next page, or `null` on the last page. `get_company_jobs` also lists `companiesWithoutJobs` with the reason, such as `not_found`.

Both tools are annotated as read-only. They start searches in your Apify account and read public job boards; they never write to a job board or change your other Apify data.

## How to connect

1. Create a free Apify account at [apify.com](https://apify.com) and copy your API token from [Settings, API & Integrations](https://console.apify.com/settings/integrations).
2. The server URL is `https://conserving-celerytop--jobs-mcp-server.apify.actor/mcp` (the Actor's **Endpoints** tab on Apify shows the same URL).
3. Add it to your client:
   - **Claude (claude.ai, Claude Desktop):** Settings, Connectors, **Add custom connector**. Paste the URL. In **Advanced settings** choose **No sign-in**, and under **Request headers** add `authorization` with the value `Bearer <your Apify token>`. Request headers are in beta in Claude; if you do not see them, use Claude Code or another client for now.
   - **Claude Code:** `claude mcp add --transport http jobs <server URL> --header "Authorization: Bearer <your Apify token>"`
   - **Other clients:** any MCP client with Streamable HTTP and custom headers. Send the token in the `Authorization` header, never in the URL.
4. Ask, for example: "Which engineering jobs are open at Stripe and Datadog?" or "Find remote senior data engineer jobs posted in the last 7 days."

## Pricing

All charges go to your own Apify account.

| What | Price |
|---|---|
| Each successful tool call, including each further page | $0.005 |
| `get_company_jobs`, charged by the ATS Jobs API Actor | $0.045 per company on the Apify free plan, less on paid plans, up to 1,000 jobs included |
| `search_tech_jobs`, charged by the Tech Jobs Search Actor | $1.15 per 1,000 matching jobs on the free plan, less on paid plans |
| Apify platform usage of the server | 256 MB of memory while you use it, until it has been idle for the idle timeout |

Failed calls and calls with invalid input are not charged by the server. Apify's free plan includes $5 of credit every month. Prices checked on September 27, 2026.

## Limits

- Up to 25 companies per call of `get_company_jobs`; up to 500 matching jobs per call of `search_tech_jobs`; up to 100 jobs per page.
- A search that runs longer than 2 minutes returns a cursor. Call the tool again with it; the search is not repeated or charged twice.
- Cursors work as long as your Apify plan keeps run data.
- Results are what each company's own job board shows at the time of the call. Some company names cannot be matched to a job board; a job board link always works best.

## Privacy

See the [privacy policy](/privacy/).

## Support

Email [don.mangu.data@gmail.com](mailto:don.mangu.data@gmail.com). We aim to answer within 3 working days. Security reports go to the same address.

Built by Don Mangu. Not affiliated with Anthropic, Apify, or the companies whose jobs the tools return. Company names are trademarks of their owners.
