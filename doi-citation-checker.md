---
layout: post
title: "DOI Citation Count Checker: Free Browser Tool"
subtitle: "Paste one or more DOIs and see the title, citation count, year and journal from Crossref, free and with no login"
description: "Free browser tool: paste one or more DOIs and get the title, citation count, publication year and journal from Crossref's public API. Runs in your browser, no login, no server."
seo_title: "DOI Citation Count Checker: Free Browser Tool"
tags: ["doi", "crossref", "citations", "research", "tools"]
permalink: /doi-citation-checker/
date: 2026-10-05 00:00:00 +0000
published: true
---
{% comment %}
CORS check, curl with an Origin header, 2026-10-05:
  api.crossref.org/works/10.1038/nature12373?mailto=... 200, access-control-allow-origin: *
  api.crossref.org/works/10.9999/not-a-real-doi-xyz     404, access-control-allow-origin: * (same header on the 404)
  OPTIONS preflight (Access-Control-Request-Method: GET) also answered 200 with the same header.
Crossref's own documentation: no sign-up or API key is needed for the public REST API ("we don't want to
unnecessarily burden developers... with cumbersome API tokens or registration processes"), and adding a
mailto parameter or a mailto: in the User-Agent moves a request into their faster "polite pool". This tool
sends mailto=don.mangu.data@gmail.com on every request for that reason; it is this site's own contact
address, not anything typed into the page.
{% endcomment %}

*Disclosure: I built the Apify Actor linked at the bottom of this page, and it is paid. This page was drafted with an AI assistant. The Crossref endpoint and its CORS header were tested on October 5, 2026. The tool runs in your browser, sends nothing to my servers, and has no analytics or tracking.*

Crossref, the registry that issues most scholarly DOIs, publishes each work's metadata for free through a public JSON endpoint, no login required. One of the fields is `is-referenced-by-count`, the number of other Crossref-registered works that cite it. This tool asks Crossref for each DOI you paste and shows the title, that count, the year and the journal.

{% raw %}
<style>
  .doic { max-width: 760px; margin: 1.5em 0; font-family: inherit; }
  .doic textarea { width: 100%; min-height: 6.5em; padding: .55em .7em; font-size: 1em; font-family: inherit; border: 1px solid #888; border-radius: 4px; box-sizing: border-box; resize: vertical; }
  .doic .row { display: flex; gap: .5em; flex-wrap: wrap; align-items: center; margin-top: .5em; }
  .doic button { padding: .55em 1.1em; font-size: 1em; border: 1px solid #333; border-radius: 4px; background: #222; color: #fff; cursor: pointer; }
  .doic button[disabled] { opacity: .5; cursor: not-allowed; }
  .doic .hint { font-size: .9em; color: #555; margin: .5em 0; }
  .doic table { border-collapse: collapse; width: 100%; margin: 1em 0; font-size: .95em; }
  .doic th, .doic td { border: 1px solid #ccc; padding: .4em .6em; text-align: left; vertical-align: top; }
  .doic .ok { font-weight: 600; }
  .doic .scroll { max-height: 420px; overflow: auto; }
  .doic .msg { margin: .8em 0; }
  .doic .count { font-variant-numeric: tabular-nums; }
</style>

<div class="doic" id="doic">
  <form id="doic-form" autocomplete="off">
    <label for="doic-input" style="position:absolute;left:-9999px">DOIs, one per line</label>
    <textarea id="doic-input" placeholder="10.1038/nature12373&#10;https://doi.org/10.1126/science.1157784&#10;10.1016/j.cell.2021.01.001" required></textarea>
    <div class="row">
      <button type="submit" id="doic-go">Check citation counts</button>
      <span id="doic-count" class="hint"></span>
    </div>
  </form>
  <p class="hint">One DOI per line (commas also work). Plain DOIs or full <code>doi.org</code> links both work. Up to 25 at a time, sent to Crossref's public API straight from your browser, a few at once so the tool stays inside Crossref's normal usage.</p>
  <div id="doic-out" class="msg" aria-live="polite"></div>
</div>

<script>
(function () {
  var MAILTO = 'don.mangu.data@gmail.com';
  var MAX_DOIS = 25;
  var CONCURRENCY = 4;

  var form = document.getElementById('doic-form');
  var input = document.getElementById('doic-input');
  var btn = document.getElementById('doic-go');
  var countEl = document.getElementById('doic-count');
  var out = document.getElementById('doic-out');
  var rows = [];

  function parseDois(raw) {
    var parts = raw.split(/[\n,]+/).map(function (s) { return s.trim(); }).filter(Boolean);
    var seen = {};
    var out = [];
    parts.forEach(function (p) {
      var s = p.replace(/^https?:\/\/(dx\.)?doi\.org\//i, '').replace(/^doi:\s*/i, '').trim();
      if (!s) return;
      var key = s.toLowerCase();
      if (seen[key]) return;
      seen[key] = true;
      out.push(s);
    });
    return out;
  }

  input.addEventListener('input', function () {
    var n = parseDois(input.value).length;
    countEl.textContent = n ? (n + (n === 1 ? ' DOI' : ' DOIs') + (n > MAX_DOIS ? ' (first ' + MAX_DOIS + ' will be checked)' : '')) : '';
  });

  function fetchOne(doi) {
    var url = 'https://api.crossref.org/works/' + encodeURIComponent(doi) + '?mailto=' + encodeURIComponent(MAILTO);
    return fetch(url, { method: 'GET', headers: { 'Accept': 'application/json' } })
      .then(function (r) {
        if (r.status === 404) return { doi: doi, status: 'not found' };
        if (!r.ok) return { doi: doi, status: 'error ' + r.status };
        return r.json().then(function (d) {
          var m = d.message || {};
          var year = null;
          try { year = (m.issued && m.issued['date-parts'] && m.issued['date-parts'][0] && m.issued['date-parts'][0][0]) || null; } catch (e) {}
          return {
            doi: doi,
            status: 'ok',
            title: (m.title && m.title[0]) || '(no title on record)',
            journal: (m['container-title'] && m['container-title'][0]) || '',
            year: year,
            citations: (typeof m['is-referenced-by-count'] === 'number') ? m['is-referenced-by-count'] : null
          };
        });
      })
      .catch(function () { return { doi: doi, status: 'request failed' }; });
  }

  function runPool(items, limit, worker) {
    var results = new Array(items.length);
    var next = 0;
    function pump() {
      if (next >= items.length) return Promise.resolve();
      var i = next++;
      return worker(items[i]).then(function (r) { results[i] = r; return pump(); });
    }
    var runners = [];
    for (var k = 0; k < Math.min(limit, items.length); k++) runners.push(pump());
    return Promise.all(runners).then(function () { return results; });
  }

  function csvCell(v) {
    v = v == null ? '' : String(v);
    if (/^[=+\-@\t\r]/.test(v)) v = "'" + v;
    return '"' + v.replace(/"/g, '""') + '"';
  }

  function download() {
    var lines = ['doi,status,title,citations,year,journal'];
    rows.forEach(function (r) {
      lines.push([r.doi, r.status, r.title || '', r.citations == null ? '' : r.citations, r.year || '', r.journal || ''].map(csvCell).join(','));
    });
    var blob = new Blob(['﻿' + lines.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    var a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'doi-citations.csv';
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    setTimeout(function () { URL.revokeObjectURL(a.href); }, 1000);
  }

  function el(tag, text) {
    var e = document.createElement(tag);
    if (text != null) e.textContent = text;
    return e;
  }

  function render(results) {
    out.textContent = '';
    rows = results;
    var ok = results.filter(function (r) { return r.status === 'ok'; });
    var bad = results.filter(function (r) { return r.status !== 'ok'; });

    var summary;
    if (ok.length && !bad.length) {
      summary = 'Found ' + ok.length + ' of ' + results.length + (results.length === 1 ? ' DOI' : ' DOIs') + ' on Crossref.';
    } else if (ok.length) {
      summary = 'Found ' + ok.length + ' of ' + results.length + '; ' + bad.length + ' did not resolve (wrong DOI, or Crossref does not have it).';
    } else {
      summary = 'None of these resolved on Crossref. Check the DOIs and try again.';
    }
    out.appendChild(el('p', summary));

    var t = el('table');
    var head = el('tr');
    ['DOI', 'Title', 'Citations', 'Year', 'Journal / venue'].forEach(function (h) { head.appendChild(el('th', h)); });
    t.appendChild(head);
    results.forEach(function (r) {
      var tr = el('tr');
      var tdDoi = el('td');
      var a = el('a', r.doi); a.href = 'https://doi.org/' + r.doi; a.rel = 'noopener nofollow'; a.target = '_blank';
      tdDoi.appendChild(a);
      tr.appendChild(tdDoi);
      if (r.status === 'ok') {
        tr.appendChild(el('td', r.title));
        var tdC = el('td', r.citations == null ? '-' : String(r.citations));
        tdC.className = 'ok count';
        tr.appendChild(tdC);
        tr.appendChild(el('td', r.year ? String(r.year) : '-'));
        tr.appendChild(el('td', r.journal || '-'));
      } else {
        var tdS = el('td', r.status);
        tdS.setAttribute('colspan', '4');
        tr.appendChild(tdS);
      }
      t.appendChild(tr);
    });

    var wrap = el('div'); wrap.className = 'scroll';
    wrap.appendChild(t);
    out.appendChild(wrap);

    if (ok.length) {
      var dl = el('button', 'Download CSV (' + results.length + ' rows)');
      dl.type = 'button';
      dl.addEventListener('click', download);
      out.appendChild(dl);
    }
  }

  form.addEventListener('submit', function (e) {
    e.preventDefault();
    var all = parseDois(input.value);
    if (!all.length) { out.textContent = 'Paste at least one DOI.'; return; }
    var dois = all.slice(0, MAX_DOIS);
    btn.disabled = true;
    out.textContent = 'Checking ' + dois.length + (dois.length === 1 ? ' DOI' : ' DOIs') + ' on Crossref...';
    runPool(dois, CONCURRENCY, fetchOne).then(function (results) {
      render(results);
      btn.disabled = false;
    });
  });
})();
</script>
{% endraw %}

## What the fields mean

- **Citations** is Crossref's `is-referenced-by-count`: the number of other works registered with Crossref that cite this one. It is not the same number as Google Scholar, Scopus or Web of Science, which index different sources, so treat it as one data point, not the final word, and say where it came from if you quote it.
- **not found** means Crossref has no record for that string as a DOI. Check for a typo, or the DOI may belong to another registration agency such as DataCite (common for datasets), which Crossref does not cover.
- Counts change over time. The number you see is live at the moment you click, not cached.

## How it works, and what it does not do

Your browser sends one GET request per DOI straight to `api.crossref.org/works/<doi>`, four at a time, with a `mailto` parameter identifying this site so Crossref's polite pool handles the requests. Nothing is stored and nothing is sent to this site; there are no cookies, analytics or third-party scripts on this page. The tool checks up to 25 DOIs per click, pasted by hand, which is what the free public API is for. It does not search by keyword, author or journal, and it does not fetch abstracts or reference lists.

## Need more than 25 at a time, on a schedule, or without typing them in?

For a list of DOIs read from a file, a run larger than you'd paste by hand, or a lookup that repeats on a schedule, I built a paid Apify Actor, [Crossref Works Search](https://apify.com/conserving_celerytop/crossref-works-search). Give it up to 10,000 DOIs in one run (as a list, a CSV column or doi.org links) and it returns one row per work as JSON or CSV, with no copy-pasting into a browser form. It also doubles as a search tool, finding works by keyword, author, journal, funder or date range when you don't have DOIs yet. On the base price tier it is charged per work found; check the current price on the Actor page. For the Python version of the same free lookup, see [DOI Citation Count Bulk Lookup in Python](/doi-citation-count-bulk-lookup/).
