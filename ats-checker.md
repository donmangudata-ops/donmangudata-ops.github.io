---
layout: post
title: "Which ATS Does This Company Use? Free Job Board Checker"
subtitle: "Type a company slug and see whether it hosts jobs on Greenhouse, Lever or Ashby, with the open job count and a CSV download"
description: "Free browser tool: enter a company slug and check the public Greenhouse, Lever and Ashby job boards. Shows which ATS answers, the number of open jobs, and a CSV of titles, locations and links."
seo_title: "Which ATS Does This Company Use? Free Job Board Checker"
tags: ["ats", "greenhouse", "lever", "ashby", "tools"]
permalink: /ats-checker/
date: 2026-10-02 00:00:00 +0000
published: true
---
{% comment %}
CORS check, curl with an Origin header, 2026-10-02:
  boards-api.greenhouse.io/v1/boards/stripe/jobs          200, access-control-allow-origin: *
  api.lever.co/v0/postings/palantir?mode=json             200, Access-Control-Allow-Origin: *
  api.ashbyhq.com/posting-api/job-board/ashby             200, access-control-allow-origin: *
Unknown slug returns 404 on all three. Lever's OPTIONS preflight also answered 200 with the same header (not needed, requests are simple GETs).
Not tested here: Lever EU region (api.eu.lever.co), so EU-hosted Lever boards are not covered by this page.
{% endcomment %}

*Disclosure: I built the Apify Actor linked at the bottom of this page, and it is paid. This page was drafted with an AI assistant. The three job board endpoints were tested on October 2, 2026. The tool runs in your browser, sends nothing to my servers, and has no analytics or tracking.*

Many companies host their careers page on a hosted applicant tracking system (ATS). Greenhouse, Lever and Ashby each publish the open jobs of a company board through a public JSON endpoint, with no login. This tool asks all three for the company slug you type and tells you which one answers.

{% raw %}
<style>
  .atsc { max-width: 760px; margin: 1.5em 0; font-family: inherit; }
  .atsc form { display: flex; gap: .5em; flex-wrap: wrap; }
  .atsc input[type=text] { flex: 1 1 260px; padding: .55em .7em; font-size: 1em; border: 1px solid #888; border-radius: 4px; }
  .atsc button { padding: .55em 1.1em; font-size: 1em; border: 1px solid #333; border-radius: 4px; background: #222; color: #fff; cursor: pointer; }
  .atsc button[disabled] { opacity: .5; cursor: not-allowed; }
  .atsc .hint { font-size: .9em; color: #555; margin: .5em 0; }
  .atsc table { border-collapse: collapse; width: 100%; margin: 1em 0; font-size: .95em; }
  .atsc th, .atsc td { border: 1px solid #ccc; padding: .4em .6em; text-align: left; vertical-align: top; }
  .atsc .ok { font-weight: 600; }
  .atsc .scroll { max-height: 360px; overflow: auto; }
  .atsc .msg { margin: .8em 0; }
</style>

<div class="atsc" id="atsc">
  <form id="atsc-form" autocomplete="off">
    <label for="atsc-slug" style="position:absolute;left:-9999px">Company slug or careers URL</label>
    <input type="text" id="atsc-slug" placeholder="e.g. stripe, palantir, ashby" maxlength="80" required>
    <button type="submit" id="atsc-go">Check</button>
  </form>
  <p class="hint">Use the part of the careers URL after the host, for example <code>stripe</code> from <code>boards.greenhouse.io/stripe</code>. You can also paste a full Greenhouse, Lever or Ashby careers URL. One company at a time. The three requests go straight from your browser to the three public APIs.</p>
  <div id="atsc-out" class="msg" aria-live="polite"></div>
</div>

<script>
(function () {
  var form = document.getElementById('atsc-form');
  var input = document.getElementById('atsc-slug');
  var btn = document.getElementById('atsc-go');
  var out = document.getElementById('atsc-out');
  var rows = [];
  var lastSlug = '';

  function parseSlug(raw) {
    var s = raw.trim();
    var m = s.match(/(?:boards|job-boards)(?:\.eu)?\.greenhouse\.io\/(?:embed\/job_board\?for=)?([A-Za-z0-9_-]+)/i) ||
            s.match(/jobs\.lever\.co\/([A-Za-z0-9_-]+)/i) ||
            s.match(/jobs\.ashbyhq\.com\/([A-Za-z0-9_.%-]+)/i);
    if (m) s = m[1];
    s = s.toLowerCase();
    return /^[a-z0-9][a-z0-9_.%-]{0,79}$/.test(s) ? s : '';
  }

  function safeUrl(u) {
    return typeof u === 'string' && /^https?:\/\//i.test(u) ? u : '';
  }

  var boards = [
    { name: 'Greenhouse',
      url: function (s) { return 'https://boards-api.greenhouse.io/v1/boards/' + encodeURIComponent(s) + '/jobs'; },
      parse: function (d) {
        return (d.jobs || []).map(function (j) {
          return { title: j.title, location: j.location && j.location.name, url: j.absolute_url };
        });
      } },
    { name: 'Lever',
      url: function (s) { return 'https://api.lever.co/v0/postings/' + encodeURIComponent(s) + '?mode=json'; },
      parse: function (d) {
        return (Array.isArray(d) ? d : []).map(function (j) {
          return { title: j.text, location: j.categories && j.categories.location, url: j.hostedUrl };
        });
      } },
    { name: 'Ashby',
      url: function (s) { return 'https://api.ashbyhq.com/posting-api/job-board/' + encodeURIComponent(s); },
      parse: function (d) {
        return (d.jobs || []).map(function (j) {
          return { title: j.title, location: j.location, url: j.jobUrl };
        });
      } }
  ];

  function check(b, slug) {
    return fetch(b.url(slug), { method: 'GET', headers: { 'Accept': 'application/json' } })
      .then(function (r) {
        if (r.status === 404) return { board: b.name, status: 'not found', jobs: [] };
        if (!r.ok) return { board: b.name, status: 'error ' + r.status, jobs: [] };
        return r.json().then(function (d) {
          return { board: b.name, status: 'found', jobs: b.parse(d) };
        });
      })
      .catch(function () { return { board: b.name, status: 'request failed', jobs: [] }; });
  }

  function csvCell(v) {
    v = v == null ? '' : String(v);
    if (/^[=+\-@\t\r]/.test(v)) v = "'" + v;
    return '"' + v.replace(/"/g, '""') + '"';
  }

  function download() {
    var lines = ['ats,company_slug,title,location,url'];
    rows.forEach(function (r) {
      lines.push([r.ats, lastSlug, r.title, r.location, r.url].map(csvCell).join(','));
    });
    var blob = new Blob(['﻿' + lines.join('\r\n')], { type: 'text/csv;charset=utf-8' });
    var a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'jobs-' + lastSlug + '.csv';
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

  function render(slug, results) {
    out.textContent = '';
    var hits = results.filter(function (r) { return r.status === 'found' && r.jobs.length > 0; });
    var empty = results.filter(function (r) { return r.status === 'found' && r.jobs.length === 0; });

    var t = el('table');
    var head = el('tr');
    ['Board', 'Result', 'Open jobs'].forEach(function (h) { head.appendChild(el('th', h)); });
    t.appendChild(head);
    results.forEach(function (r) {
      var tr = el('tr');
      tr.appendChild(el('td', r.board));
      var td = el('td', r.status);
      if (r.status === 'found') td.className = 'ok';
      tr.appendChild(td);
      tr.appendChild(el('td', r.status === 'found' ? String(r.jobs.length) : '-'));
      t.appendChild(tr);
    });

    var summary;
    if (hits.length) {
      summary = 'Jobs found on ' + hits.map(function (r) { return r.board; }).join(' and ') + ' for "' + slug + '".';
    } else if (empty.length) {
      summary = 'The board exists on ' + empty.map(function (r) { return r.board; }).join(' and ') + ' but lists 0 open jobs right now.';
    } else if (results.some(function (r) { return r.status === 'request failed' || r.status.indexOf('error') === 0; })) {
      summary = 'No board answered, and at least one request failed. Try again in a minute.';
    } else {
      summary = 'No Greenhouse, Lever or Ashby board answers to "' + slug + '". The slug may differ from the company name, or the company uses another ATS or a Lever EU board, which this tool does not check.';
    }
    out.appendChild(el('p', summary));
    out.appendChild(t);

    rows = [];
    hits.forEach(function (r) {
      r.jobs.forEach(function (j) {
        rows.push({ ats: r.board, title: j.title || '', location: j.location || '', url: safeUrl(j.url) });
      });
    });

    if (rows.length) {
      var dl = el('button', 'Download CSV (' + rows.length + ' jobs)');
      dl.type = 'button';
      dl.addEventListener('click', download);
      out.appendChild(dl);

      var wrap = el('div'); wrap.className = 'scroll';
      var jt = el('table');
      var jh = el('tr');
      ['ATS', 'Title', 'Location'].forEach(function (h) { jh.appendChild(el('th', h)); });
      jt.appendChild(jh);
      rows.slice(0, 200).forEach(function (r) {
        var tr = el('tr');
        tr.appendChild(el('td', r.ats));
        var td = el('td');
        if (r.url) {
          var a = el('a', r.title); a.href = r.url; a.rel = 'noopener nofollow'; a.target = '_blank';
          td.appendChild(a);
        } else { td.textContent = r.title; }
        tr.appendChild(td);
        tr.appendChild(el('td', r.location));
        jt.appendChild(tr);
      });
      wrap.appendChild(jt);
      out.appendChild(wrap);
      if (rows.length > 200) out.appendChild(el('p', 'Showing the first 200 rows. The CSV has all ' + rows.length + '.'));
    }
  }

  form.addEventListener('submit', function (e) {
    e.preventDefault();
    var slug = parseSlug(input.value);
    if (!slug) { out.textContent = 'Enter a company slug (letters, digits, - _ .) or a careers URL.'; return; }
    btn.disabled = true;
    out.textContent = 'Checking Greenhouse, Lever and Ashby...';
    lastSlug = slug;
    Promise.all(boards.map(function (b) { return check(b, slug); })).then(function (res) {
      render(slug, res);
      setTimeout(function () { btn.disabled = false; }, 3000);
    });
  });
})();
</script>
{% endraw %}

## What the result means

- **found** means that board answered with a list of jobs for this slug. A company can have more than one board, for example after an ATS migration.
- **not found** means the board returned 404 for this slug. The company may use the same name with a different slug, or a different ATS.
- The count is the number of open postings the public endpoint returns at the moment you click. It can differ from what a careers page shows, because some companies hide postings from the public endpoint or filter their own page.
- Slugs are chosen by the company. If the name does not work, open the company's careers page and read the slug from the link behind its "Apply" button.

## How it works, and what it does not do

Your browser sends one GET request to each of three public endpoints per click, so three requests in total, and the button is locked for a few seconds afterwards. The endpoints are `boards-api.greenhouse.io/v1/boards/<slug>/jobs`, `api.lever.co/v0/postings/<slug>?mode=json` and `api.ashbyhq.com/posting-api/job-board/<slug>`. Nothing is stored, nothing is sent to this site, and there are no cookies, analytics or third party scripts on this page. The tool checks one company at a time on purpose. It does not cover Workday, SmartRecruiters, Personio or other systems, and it does not read job descriptions.

## Need many companies, or more than three ATS?

For a list of companies, or for systems beyond these three, I built a paid Apify Actor, [Career Page Jobs Scraper: Greenhouse, Lever, Ashby +19 ATS](https://apify.com/conserving_celerytop/live-career-page-jobs-api). It takes a list of companies and returns the open jobs as JSON or CSV. On the base price tier it is charged at $0.045 per company, and you can check the current price on the Actor page.
