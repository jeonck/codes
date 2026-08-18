+++
title = "GitHub Pages custom subdomains: static/CNAME alone is never enough"
date = 2026-08-16
summary = "The repo can have its CNAME file and the Pages API can have its cname field set — but without an actual DNS record at the registrar, certificate issuance stays blocked."
tags = ["github-pages", "dns", "https", "trial-and-error"]
+++

A checklist assembled after repeating the same "why isn't this one working" cycle across
several `*.metacog.co.kr` subdomains on GitHub Pages.

## The easy mistake

It's tempting to think filling in the repo side (a `static/CNAME` file, the Pages API's `cname`
field) is the whole job. It's only half of it. Adding the actual CNAME record at the **domain
registrar's DNS** is a separate, and manual, step.

## Cause

GitHub can't issue a Let's Encrypt certificate until the DNS record genuinely points that
subdomain at `jeonck.github.io.`. Until then, `https_enforced` stays `false`, and hitting the
site over HTTPS either throws a certificate error or 404s.

## The order that works

1. Add a `static/CNAME` file to the repo with just the subdomain.

   ```
   codes.metacog.co.kr
   ```

2. Add the CNAME record directly in the registrar's DNS panel (usually not something an agent
   can do via API/CLI).

   ```
   codes  CNAME  jeonck.github.io.
   ```

3. Wait for propagation, then confirm.

   ```bash
   dig +short codes.metacog.co.kr
   # passing once this returns jeonck.github.io.
   ```

4. Only once propagation is confirmed, turn on HTTPS enforcement.

   ```bash
   gh api repos/jeonck/<repo>/pages --method PUT --field https_enforced=true
   ```

## Verifying the deploy before DNS has propagated

Deployment correctness can be checked independently of DNS propagation by pinning the resolved
IP directly:

```bash
curl --resolve codes.metacog.co.kr:80:185.199.108.153 http://codes.metacog.co.kr/
```

That separates "the site itself is broken" from "just waiting on DNS" as two distinct failure
modes.
