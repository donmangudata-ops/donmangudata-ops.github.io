---
layout: post
title: "Bulk MX Record Lookup in Python"
subtitle: "A dnspython script for a few domains, and a $2 per 1,000 API for lists, with mail provider detection"
description: "Run a bulk MX record lookup in Python: a free dnspython script, how to spot Google Workspace and Microsoft 365 from MX hosts, and a $2 per 1,000 domain API."
seo_title: "Bulk MX Record Lookup in Python: Free Script and API"
tags: ["python", "dns", "email", "apify"]
permalink: /bulk-mx-record-lookup/
date: 2026-10-02 00:00:00 +0000
cadence_day: 4
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

An MX record tells the world which servers receive email for a domain. A bulk MX record lookup asks that question for a whole list at once. People do it to see which companies use Google Workspace or Microsoft 365, to find domains that cannot receive mail at all, and to clean a list before sending to it.

## What an MX record contains

Each MX record has a priority number and a host name. Lower numbers are tried first. A domain can have several. The host name is the useful part: `aspmx.l.google.com` points to Google, and a name ending in `mail.protection.outlook.com` points to Microsoft. Hosts such as `mx.zoho.com` or a security gateway tell their own story.

A domain with no MX record may still accept mail through its A record in some setups, but in practice it is a sign that nobody reads email there.

## The free way: dnspython

```python
import dns.resolver

def mx(domain):
    try:
        answers = dns.resolver.resolve(domain, "MX", lifetime=5)
    except dns.resolver.NXDOMAIN:
        return {"domain": domain, "status": "not_found", "mx": []}
    except (dns.resolver.NoAnswer, dns.resolver.NoNameservers, dns.exception.Timeout):
        return {"domain": domain, "status": "no_mx", "mx": []}
    records = sorted((r.preference, str(r.exchange).rstrip(".")) for r in answers)
    return {"domain": domain, "status": "ok", "mx": records}

for d in ["apify.com", "wikipedia.org", "gov.uk"]:
    print(mx(d))
```

Run this over a list with a thread pool of 10 to 20 workers and it is fast. The costs are the ones you would expect: you need to handle timeouts and retries, your own resolver's rate limits apply, and you decide what counts as a failure.

## Detect the mail provider from the MX host

Once you have the hosts, a simple suffix match covers the common cases.

```python
PROVIDERS = {
    "Google Workspace": ("google.com", "googlemail.com"),
    "Microsoft 365": ("mail.protection.outlook.com",),
    "Zoho": ("zoho.com", "zoho.eu"),
}

def provider(hosts):
    for name, suffixes in PROVIDERS.items():
        if any(h.endswith(s) for h in hosts for s in suffixes):
            return name
    return "other"
```

Treat the result as a good guess. A domain behind a filtering service, such as a gateway that forwards to the real mailbox, shows the gateway and not the provider.

## A hosted option: DNS Lookup

[DNS Lookup](https://apify.com/conserving_celerytop/dns-lookup) is an Apify Actor I built for lists of domains. It reads A, AAAA, CNAME, MX, NS, TXT, CAA and SOA records, and can add SPF and DMARC. For this job you only need MX.

```bash
pip install "apify-client>=3.2,<4"
export APIFY_TOKEN=<YOUR_APIFY_TOKEN>
```

```python
import csv, os
from decimal import Decimal
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

with open("domains.txt") as f:
    domains = [line.strip() for line in f if line.strip()]

run = client.actor("conserving_celerytop/dns-lookup").call(
    run_input={
        "domains": domains,
        "recordTypes": ["MX"],
        "includeEmailPolicy": False,
        "maxResults": len(domains),
    },
    max_total_charge_usd=Decimal("10.00"),
)

with open("mx.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["domain", "status", "hasMx", "mx_hosts"])
    for row in client.dataset(run.default_dataset_id).iterate_items():
        hosts = [m["exchange"] for m in row.get("mx", [])]
        w.writerow([row["domain"], row["status"], row.get("hasMx"), " ".join(hosts)])
```

Website addresses and email addresses also work in `domains`. The Actor takes the domain from them.

## Sample output

```json
{
  "domain": "apify.com",
  "mx": [
    { "priority": 1, "exchange": "aspmx.l.google.com" }
  ],
  "hasMx": true,
  "status": "ok",
  "charged": true
}
```

Mail servers come back sorted by priority. A domain that does not exist returns `not_found`, a resolver problem returns `dns_error`, and an entry that is not a domain returns `invalid_domain`. If one record type times out, the row is still returned with that type listed in `failedTypes`.

## What it costs

Store price on October 2, 2026. Check the Store page before relying on it.

- $0.002 per domain looked up, which is $2 per 1,000.
- Apify adds $0.00005 per run for each GB of memory at start.
- Domains that do not exist, DNS errors and invalid entries are free.
- Example: 5,000 domains cost 5,000 x $0.002 = $10.00.

## Alternatives

- **dnspython or `dig` yourself.** Free and fully under your control. Best for a few hundred domains or if you already run a resolver. A shell loop with `dig +short MX` is enough for a one-off.
- **Online MX lookup tools**, for example MXToolbox. Good for one domain at a time. Their bulk features and limits vary, so check before planning around them.
- **Email verification services.** They check whether a mailbox exists, which an MX lookup cannot. They cost more per address because they do more.
- **A paid enrichment database.** It may tell you the provider without any lookup, but it shows what was true when it last crawled, not what DNS says today.

## Limits

- An MX lookup shows where mail is routed, not whether a given mailbox exists.
- Results match what a public resolver returns. Split-horizon setups inside a company network can differ.
- It does not check whether a domain is registered. A domain with no DNS records is simply `not_found`.

## FAQ

**How do I check MX records for many domains at once?**
Loop over the list with `dnspython` and a thread pool, or send the list to a DNS API. Both give you one row per domain.

**How can I tell if a company uses Google Workspace or Microsoft 365?**
Look at the MX hosts. Google hosts end in `google.com` or `googlemail.com`, and Microsoft hosts end in `mail.protection.outlook.com`.

**What does it mean when a domain has no MX record?**
Usually that it does not receive email. Check the A record and the domain's purpose before dropping it from a list.

**Is a bulk MX lookup legal?**
It reads public DNS records, the same ones every mail server reads when it sends mail. It does not contact the mail servers.
