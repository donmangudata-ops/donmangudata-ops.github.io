---
layout: post
title: "Bulk Image Download Crashes With Out of Memory? Why and How to Avoid It"
subtitle: "Most bulk image tools keep every file in memory until the end. Here is why that breaks, what to change, and a Python script that writes as it goes"
description: "Why bulk image downloads run out of memory and how to avoid it: batches, size filters, dedupe by content, write each file as it arrives, split the ZIP. Python example."
seo_title: "Bulk Image Download Out of Memory: Causes and Fixes"
tags: ["bulk-image-downloader", "images", "python", "apify"]
permalink: /bulk-image-download-out-of-memory/
date: 2026-10-09 00:00:00 +0000
published: true
---
*Disclosure: I built the Apify Actor mentioned in the last section, and it is paid. This article was drafted with an AI assistant. The Python script below was run on October 8, 2026, against local test files, not against a live site.*

You give a tool a few thousand image links. It runs for a while, then stops with "out of memory". You get no files, and sometimes you still pay for the run.

The cause is almost always the same. The tool keeps every image in memory until the end, then builds one ZIP.

## Why memory runs out

- **All images stay in RAM.** 3,000 images at 1.5 MB each is about 4.5 GB. Many cloud runs have less memory than that.
- **One big ZIP at the end.** If the archive is built in memory, the peak is even higher.
- **Duplicates count twice.** Many pages reuse the same image under different URLs. Without dedupe, each copy uses memory.
- **Huge originals.** Some links point to 20 MB print files. One of them can tip a run over.

## What to do

1. **Split the job.** Send 500 links per run instead of 5,000. Merge the outputs later.
2. **Filter before you download.** Skip files above a size limit. You rarely need 40 MB originals.
3. **Dedupe by content, not by URL.** The same file at two URLs should be saved once.
4. **Write each file as it arrives.** Do not collect images in a list. Start a new ZIP part when the current one gets big.
5. **Check what a crash costs you.** Find out if you pay for images that never reach your output.

## A Python script that does 2 to 4

Standard library only. It skips files above 10 MB, saves each unique file once (SHA-256 of the bytes), writes it into the ZIP right away, and opens a new ZIP part after about 500 MB.

```python
import hashlib
import os
import zipfile
from urllib.request import Request, urlopen

HEADERS = {"User-Agent": "image-batch/1.0 (+contact: you@example.com)"}
MAX_BYTES = 10 * 1024 * 1024      # skip files above 10 MB
PART_LIMIT = 500 * 1024 * 1024    # start a new ZIP part after ~500 MB


def download_all(urls, out_dir="images"):
    os.makedirs(out_dir, exist_ok=True)
    seen, part, part_size = set(), 1, 0
    zf = zipfile.ZipFile(f"{out_dir}/images-part{part}.zip", "w")
    for url in urls:
        try:
            with urlopen(Request(url, headers=HEADERS), timeout=30) as r:
                size = int(r.headers.get("Content-Length") or 0)
                if size > MAX_BYTES:
                    print("skip (too big):", url)
                    continue
                data = r.read(MAX_BYTES + 1)
        except Exception as e:
            print("fail:", url, e)
            continue
        if len(data) > MAX_BYTES:
            print("skip (too big):", url)
            continue
        digest = hashlib.sha256(data).hexdigest()
        if digest in seen:  # same file under another URL
            continue
        seen.add(digest)
        if part_size + len(data) > PART_LIMIT:  # close this part, open the next
            zf.close()
            part, part_size = part + 1, 0
            zf = zipfile.ZipFile(f"{out_dir}/images-part{part}.zip", "w")
        name = digest[:16] + os.path.splitext(url.split("?")[0])[1][:5]
        zf.writestr(name, data)  # written now, not kept in a list
        part_size += len(data)
        del data
    zf.close()
    return len(seen), part
```

On my test set (two identical images under different names, one other image, one 11 MB file and one broken link) it saved 2 images, skipped the large file, reported the 404 and wrote one ZIP part.

Only download images you are allowed to copy.

## Where my tool fits

My [Bulk Image Downloader](https://apify.com/conserving_celerytop/bulk-image-downloader?utm_source=blog&utm_medium=guide&utm_campaign=image-oom) on Apify also reads web pages (img, srcset, lazy-load and Open Graph images), filters by type, width, height and file size, dedupes identical files by content, and splits very large exports into several ZIP parts. It charges $1 per 1,000 images saved. Skipped files are free.
