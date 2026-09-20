+++
title = "Adding PWA offline to a daily-updated static site: two of four features were already there, and the one that wasn't needs a rule"
date = 2026-09-20
summary = "Install icon and standalone mode came free with the manifest. The service worker is the only new part — and on a site whose value is today's deadlines, the cache strategy has to be network-first for HTML or the feature is worse than nothing."
tags = ["pwa", "service-worker", "static-site", "github-pages", "trial-and-error"]
+++

A settlement-guide site (three cities, static HTML on GitHub Pages, rebuilt every morning with
fresh events and deadline notices) got the question "should we add PWA install?" The pitch listed
four benefits: home-screen icon, standalone window, offline/fast loading, automatic updates.

## Check before building

| Feature | Already there? | What it took |
|---|---|---|
| Home-screen icon | yes — `site.webmanifest` with 192/512 icons, `apple-touch-icon` | nothing |
| Standalone window | yes — `"display": "standalone"` in the manifest | nothing |
| Offline / fast repeat loads | no | a service worker |
| Automatic updates | already the case *because* there was no service worker | a cache-versioning rule so it stays true |

Half the feature list was a manifest that had been sitting there since the favicon work. The
honest scope was "one `sw.js`", and the interesting part is the rule it has to follow.

## The rule: HTML is network-first

The default service-worker recipe (cache-first, "app shell") is exactly wrong for this site. Its
core content is a list of deadlines and a news feed that change daily; the worst possible
failure is a visitor seeing yesterday's deadline from cache while online. So:

- **HTML → network first.** Online: always the server. Offline: the cached copy of that page if
  it was visited before, otherwise `/offline/`.
- **`/assets/` → stale-while-revalidate.** Serve cache immediately, refresh in the background.
  This is the only part that makes repeat visits faster.
- **`.json` → not handled.** Daily data files fall through to the browser.
- **Other origins → not handled.** Analytics, the weather API, the CDN font.
- **Cache name = build timestamp.** Every deploy activates a new worker that deletes the old
  cache wholesale.

```js
var CACHE = 'site-' + '{{ sw_version }}';           // build.py fills the timestamp
var OFFLINE = '/offline/';

self.addEventListener('fetch', function (e) {
  var req = e.request, url = new URL(req.url);
  if (req.method !== 'GET' || url.origin !== self.location.origin) return;

  if (req.mode === 'navigate') {
    e.respondWith(fetch(req).then(function (res) {
      if (res.ok) { var copy = res.clone(); caches.open(CACHE).then(function (c) { c.put(req, copy); }); }
      return res;
    }).catch(function () {
      return caches.match(req).then(function (hit) { return hit || caches.match(OFFLINE); });
    }));
    return;
  }
  if (url.pathname.endsWith('.json')) return;
  if (url.pathname.indexOf('/assets/') === 0) {
    e.respondWith(caches.open(CACHE).then(function (c) {
      return c.match(req).then(function (hit) {
        var refresh = fetch(req).then(function (res) {
          if (res.ok) c.put(req, res.clone());
          return res;
        }).catch(function () { return hit; });
        return hit || refresh;
      });
    }));
  }
});
```

The worker is a Jinja template rendered by the build, not a static file — that's how the version
string gets in without a separate build step.

## The bug the first local test caught

The first version cached nothing on navigation. Cache listing after visiting two pages showed
only the precached files.

```js
// wrong: clone happens later, inside the .then — by then the browser has consumed the body
fetch(req).then(function (res) {
  caches.open(CACHE).then(function (c) { c.put(req, res.clone()); });
  return res;
});

// right: clone synchronously, before returning the response
fetch(req).then(function (res) {
  var copy = res.clone();
  caches.open(CACHE).then(function (c) { c.put(req, copy); });
  return res;
});
```

No error is thrown in the failing version; the `put` just quietly fails inside a promise nobody
awaits. The only way to see it is to list the cache.

## How it was tested (no device needed)

Serve the built site locally, open it in a browser, and drive it with three checks:

1. `caches.keys()` → one cache named with the current build timestamp; its `keys()` show the
   precache list.
2. Navigate to two documents, list the cache again → both URLs present.
3. **Stop the local server.** Navigate to a visited document → it renders. Navigate to an
   unvisited one → the offline page.

Then rebuild, call `registration.update()`, and confirm the old cache name is gone. Unregister
the worker before leaving the localhost origin so it doesn't shadow later previews.

## What was deliberately left out

- An install banner — the browser's own prompt is enough and a banner is noise.
- Push notifications — need a server.
- Caching the daily JSON — that's the data the site exists to keep fresh.

The privacy page got one sentence: pages you've opened are stored inside your browser so they
open offline; nothing leaves the device.
