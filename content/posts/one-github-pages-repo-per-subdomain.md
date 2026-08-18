+++
title = "One subdomain = one GitHub Pages repo — running many sites off a single account"
date = 2026-08-14
summary = "It's easy to assume GitHub Pages only really serves one thing per account. In practice it's the CNAME file that decides routing."
tags = ["github-pages", "dns", "architecture", "trial-and-error"]
+++

Adding sites one after another under the same account eventually raises the question: "doesn't
GitHub Pages only support one site per account?" In practice, a repo can host as many sites as
needed — it just comes down to understanding how the domain routing works.

## The structure

Every site under the account lives in its **own GitHub repo**, and each repo has a `CNAME` file
holding a distinct `<subdomain>.metacog.co.kr` value.

```
jeonck/talktime   → static/CNAME → talktime.metacog.co.kr
jeonck/insight    → static/CNAME → insight.metacog.co.kr
jeonck/codes      → static/CNAME → codes.metacog.co.kr
```

On the DNS side, each of those subdomains is a **CNAME record** pointing at `jeonck.github.io`
(the nameservers live with the domain registrar, and there's no wildcard record — every
subdomain gets added individually). When a request comes in, GitHub looks at the `Host` header
and matches it against the `CNAME` file of each repo, then serves whichever one matches. That's
what lets many repos share a single Pages account **without colliding**.

## Adding a new site

A new subdomain site starts with a **new repo** — not a subdirectory of an existing site.

```bash
# 1. Create the repo, commit the CNAME
echo "codes.metacog.co.kr" > static/CNAME
git add static/CNAME && git commit -m "Add custom domain"

# 2. Enable Pages, serving from the repo root on main
gh api -X POST repos/jeonck/codes/pages \
  -f 'source[branch]=main' -f 'source[path]=/'

# 3. Only enforce HTTPS once the DNS record is confirmed live
dig +short codes.metacog.co.kr
```

## Common misconception

- "Can't a new site just be a subdirectory under one portfolio repo?" — no. Subdomain routing
  operates at the repo level, not the path level.
- The account's profile README repo (`jeonck/jeonck`) is not a site under this pattern — it's the
  special repo GitHub uses for the profile page, worth remembering separately so it doesn't get
  confused with the rest.
