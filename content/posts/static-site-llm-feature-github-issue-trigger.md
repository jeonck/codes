+++
title = "Adding an LLM feature to a static site without ever putting a token in the browser"
date = 2026-07-30
summary = "Two options came up first — an API key in localStorage, or a Cloudflare Worker proxy — and neither was it. A GitHub-issue trigger pattern was requested instead."
tags = ["github-actions", "architecture", "security", "trial-and-error"]
+++

Fully static hosting like GitHub Pages sometimes still needs a "click a button, an LLM generates
something, the site reflects it" feature. The recurring question is where the credentials live.

## The two options proposed first (both declined)

1. **API key in browser localStorage** — the user pastes a key and the client calls the LLM
   directly. Simplest option, but the key is exposed client-side.
2. **Proxy through a Cloudflare Worker** — the key stays hidden, but now there's a whole extra
   piece of infrastructure to run (and pay for), and the Worker itself becomes a new attack
   surface.

Both options share the same shape: either trust something in the browser, or stand up a new
server. Neither was the direction wanted.

## The pattern that was chosen: GitHub issues as the trigger

The browser's only job is **opening a GitHub issue**. Actual execution happens inside GitHub
Actions, using a token that already lives in the repo.

```
"Add Transcript ✏️" button on the site
        │  (click navigates to a GitHub issue form)
        ▼
GitHub issue created (the user authenticates via their own GitHub account)
        │  the issues event triggers a workflow
        ▼
GitHub Actions workflow
        │  runs the agent using a repo secret token
        ▼
Result gets committed → site rebuilds
```

No secret ever touches the browser. Authentication is GitHub's own login; execution privilege
exists only inside the Actions workflow.

## The gate a public repo needs

On a public repo, an issue-triggered workflow can be fired by anyone who opens an issue. An
owner-only guard has to be the first thing the workflow checks.

```yaml
jobs:
  run-agent:
    if: github.event.issue.user.login == github.repository_owner
    runs-on: ubuntu-latest
    steps:
      - run: echo "owner-only guard passed"
```

## When to reach for this pattern

When a request comes in for "AI functionality on a static page," don't lead with a
browser-stored-token approach. This structure is already validated — offer it as the default.
