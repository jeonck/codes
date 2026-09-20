+++
title = "Search still returned old results after deploy — it wasn't the service worker, it was max-age=600"
date = 2026-09-20
summary = "A static site had just gained a service worker, so when a freshly added search entry didn't show up, PWA was the obvious suspect. The worker was innocent; GitHub Pages' ten-minute HTTP cache on the JSON index was the cause, and the fix is a build-versioned URL in the HTML."
tags = ["pwa", "caching", "github-pages", "static-site", "trial-and-error"]
+++

Same day the service worker went live, a client-side search index gained new entries (news
items, not just documents). The deploy succeeded, `curl` showed the new `search-index.json` on
the server — and the user's browser still said *no results*. The question arrived as "is the PWA
breaking this?"

## Why the worker was innocent

The worker's fetch handler returns early for `.json`, so the browser handles those requests on
its own. And the search page's HTML is network-first, so the page itself was fresh. Nothing in
the worker could hand back an old index.

## What actually cached it

```
$ curl -sI https://<site>/search-index.json | grep -i cache
cache-control: max-age=600
```

GitHub Pages serves every file with a ten-minute `max-age`. The search page fetches
`/search-index.json` by a fixed URL, so a browser that had opened search shortly before the
deploy kept reusing its cached copy for up to ten minutes — no revalidation, no service worker
involved. A hard reload fixed it, which is exactly the kind of "fix" that hides the cause.

## The fix: version the URL from the HTML

The HTML is the one thing guaranteed fresh (network-first). So the build stamps the index URL
into the page, and the script reads it from there instead of hard-coding the path.

```html
<ul id="search-results" data-index="/search-index.json?v={{ build_ver }}"></ul>
```

```js
pending = fetch(out.getAttribute('data-index') || '/search-index.json')
```

`build_ver` is the build timestamp — the same value the service worker uses for its cache name.
Every deploy changes the query string; the HTTP cache treats it as a new resource.

The same trick had already been needed once that day for a different file: favicons. The browser
had cached an old letter-mark icon under `/assets/img/favicon.svg` and never asked again, so the
new city icon didn't appear. There the version is a content hash of the SVG rather than a build
timestamp, because the icon changes rarely and the URL should stay stable when it doesn't.

## The general rule

When a site with a service worker shows stale content, there are three caches in play, and only
one of them is yours:

| Layer | Who controls it | How to bust it |
|---|---|---|
| Service worker cache | you | cache name per build; network-first for anything that must be fresh |
| Browser HTTP cache | server headers (`max-age`) | a version/hash in the URL, stamped from something that *is* fresh |
| CDN edge | the host | usually the same URL versioning; otherwise wait it out |

Check which layer holds the old bytes before touching the worker. `curl -sI` on the file plus
"does the URL change between builds?" answers it in a minute, and the answer was the middle row.
