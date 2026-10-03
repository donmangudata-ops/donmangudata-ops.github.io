---
layout: post
title: "Bulk SPF DMARC Checker in Python"
subtitle: "Check SPF records and DMARC policies for a list of domains, with a free script and a $2 per 1,000 API"
description: "Build a bulk SPF DMARC checker in Python: a free dnspython script, how to read the results, and a $2 per 1,000 domain API that flags missing records."
seo_title: "Bulk SPF DMARC Checker in Python: Script and API"
tags: ["python", "dns", "email", "apify"]
permalink: /bulk-spf-dmarc-checker/
date: 2026-10-03 00:00:00 +0000
cadence_day: 5
published: true
---
*Disclosure: I built the Apify Actor in the second half of this article, and it is paid. This article was drafted with an AI assistant. Prices and field names were checked on October 2, 2026.*

If you send email for several domains, or manage them for clients, you eventually need a bulk SPF DMARC checker. The question is simple: for each domain, is there an SPF record, is there a DMARC policy, and what does that policy say?

This guide shows the free script, what to look for in the results, and a hosted option for long lists.

## What the two records do

**SPF** is a TXT record on the domain that starts with `v=spf1`. It lists the servers allowed to send mail for the domain. It ends with a rule such as `~all` (soft fail) or `-all` (hard fail).

**DMARC** is a TXT record on `_dmarc.<domain>` that starts with `v=DMARC1`. Its `p=` tag is the policy: `none` only monitors, `quarantine` sends failures to spam, and `reject` blocks them. Large mailbox providers have required SPF or DKIM, and a DMARC record, from bulk senders for some time now, so a missing record is worth fixing.

## The free way: dnspython

```python
import dns.resolver

def txt(name):
    try:
        answers = dns.resolver.resolve(name, "TXT", lifetime=5)
    except (dns.resolver.NXDOMAIN, dns.resolver.NoAnswer,
            dns.resolver.NoNameservers, dns.exception.Timeout):
        return []
    return ["".join(s.decode() for s in r.strings) for r in answers]

def check(domain):
    spf = [t for t in txt(domain) if t.lower().startswith("v=spf1")]
    dmarc = [t for t in txt("_dmarc." + domain) if t.lower().startswith("v=dmarc1")]
    tags = {}
    if dmarc:
        for part in dmarc[0].split(";"):
            if "=" in part:
                k, v = part.strip().split("=", 1)
                tags[k.lower()] = v
    return {"domain": domain, "spf": spf[0] if spf else None,
            "dmarc_policy": tags.get("p"), "dmarc_record": dmarc[0] if dmarc else None}

for d in ["apify.com", "gov.uk"]:
    print(check(d))
```

That covers one domain in a fraction of a second. For a list, add a thread pool and retries. Two things to watch: a domain can have more than one SPF record, which is itself an error, and TXT records longer than 255 characters are split into strings, which the join above handles.

## A hosted option: DNS Lookup

[DNS Lookup](https://apify.com/conserving_celerytop/dns-lookup) is an Apify Actor I built for lists of domains. With **SPF and DMARC** on, which is the default, each row has the SPF record, the DMARC policy tags and three true or false flags.

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
    run_input={"domains": domains, "includeEmailPolicy": True, "maxResults": len(domains)},
    max_total_charge_usd=Decimal("10.00"),
)

with open("email_auth.csv", "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["domain", "status", "hasSpf", "hasDmarc", "dmarc_p", "issue"])
    for row in client.dataset(run.default_dataset_id).iterate_items():
        if row["status"] != "ok":
            w.writerow([row["domain"], row["status"], "", "", "", row.get("error", "")])
            continue
        p = (row.get("dmarcPolicy") or {}).get("p")
        if not row.get("hasSpf"):
            issue = "no SPF"
        elif not row.get("hasDmarc"):
            issue = "no DMARC"
        elif p == "none":
            issue = "DMARC monitor only"
        else:
            issue = ""
        w.writerow([row["domain"], "ok", row.get("hasSpf"), row.get("hasDmarc"), p, issue])
```

## Sample output

```json
{
  "domain": "apify.com",
  "spf": "v=spf1 include:_spf.google.com ~all",
  "dmarcPolicy": {
    "v": "DMARC1",
    "p": "reject",
    "reportsRequested": true
  },
  "hasMx": true,
  "hasSpf": true,
  "hasDmarc": true,
  "status": "ok",
  "charged": true
}
```

The Actor returns the policy tags and whether reports are requested. It does not return the report addresses, because they can be personal mailboxes. If you need those, read the `_dmarc` record yourself with the script above.

Rows with `not_found`, `dns_error` or `invalid_domain` as the `status` are free and carry the reason in `error`.

## What it costs

Store price on October 2, 2026. Check the Store page before relying on it.

- $0.002 per domain looked up, which is $2 per 1,000.
- Apify adds $0.00005 per run for each GB of memory at start.
- Domains that do not exist, DNS errors and invalid entries are free.
- Example from the Actor page: 5,000 client domains cost 5,000 x $0.002 = $10.00.

## Alternatives

- **Your own script.** The one above is free and enough for a few hundred domains. You own retries and error handling.
- **`dig` in a shell loop.** Fine for a one-off audit.
- **Web checkers**, such as MXToolbox or Google's Admin Toolbox. Good for a single domain and for explanations of each error. Check their bulk limits before relying on them.
- **DMARC monitoring services**, such as dmarcian or Valimail. They process the aggregate reports that receivers send you, which tells you who is actually sending as your domain. A lookup cannot do that, and it is the better tool once you want to move from `p=none` to `reject`.

## Limits

- It reads and returns the records. It does not judge them for you, so the rules in your own script, like the one above, decide what counts as a problem.
- It does not test whether SPF stays under the 10 DNS lookup limit, and it does not look for DKIM keys, which need a selector name you have to know.
- It is a snapshot of DNS now, not a history. Schedule a run if you want change monitoring.

## FAQ

**How do I check SPF and DMARC for many domains?**
Query the TXT record of each domain for `v=spf1` and the TXT record of `_dmarc.<domain>` for `v=DMARC1`. Loop over the list, or use a bulk API.

**What DMARC policy should a domain have?**
`p=none` is a start, because it only collects reports. The usual goal is `quarantine` and then `reject` once you know all legitimate senders pass.

**Can a domain have two SPF records?**
It should not. Two records is an error that can make SPF fail, so the merge into one record is the fix.

**Does the checker work for domains that do not send email?**
Yes. A domain that never sends mail can publish `v=spf1 -all` and a DMARC `p=reject` policy, and a bulk check will show whether it does.

## Related guides

- [Bulk MX Record Lookup in Python](/bulk-mx-record-lookup/)
- [Tech Stack Lookup API: CMS, Ecommerce and Analytics for Any Website](/tech-stack-lookup-api-python/)
