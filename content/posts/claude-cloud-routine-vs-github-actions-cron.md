+++
title = "Claude Cloud Routine vs. GitHub Actions cron: they look the same but you manage them differently"
date = 2026-08-17
summary = "A repo with no visible cron workflow isn't proof there's no automated publishing. The schedule and generation logic can live entirely outside the repo, on Claude's side."
tags = ["claude-code", "automation", "github-actions", "trial-and-error"]
+++

When maintaining a site that publishes a new post every day, the first thing worth confirming is
*where* the automation actually runs — otherwise it's easy to go debugging the wrong place.

## Two different patterns

A few earlier pipelines (`insight`, `invest-news`, etc.) followed a familiar shape:

- a `pipeline/` directory with a collector script in the repo
- a cron-triggered workflow under `.github/workflows/`
- Actions handling collect → generate → commit → deploy end to end

But a site built later had none of that — no `pipeline/` directory, no collector script anywhere
in the repo. The Actions workflow there did **only** build and deploy.

## Cause

Publishing for that site was owned by a **Claude Cloud Routine**. The routine itself performs the
web search and writes the Markdown post, then pushes it — and that push is what triggers the
Actions build. In other words, the logic deciding *what to write* lived outside the repo
entirely, on Claude's own schedule.

```bash
# searching the repo turns up nothing about the schedule or routine ID
grep -r "cron" .github/workflows/   # only the build/deploy workflow shows up
find . -name "pipeline"             # nothing
```

## How to check and manage it

- Routines can only be managed at https://claude.ai/code/routines (deletion included — only
  possible there).
- The schedule is a **fixed-UTC cron**, which matters across DST transitions. E.g. `0 11 * * *`
  UTC is 06:00 CDT, but once the clocks fall back to CST that becomes 05:00 local — the cron
  needs updating to `0 12 * * *` to keep it at 6 AM.
- To change publishing format or content, edit the repo's root `CLAUDE.md` rather than the
  routine's prompt — if the routine is designed to read that file to decide its behavior.

## Takeaway

When inheriting a site with automated publishing, don't assume "no cron workflow visible" means
"manually published." A Claude Cloud Routine living outside the repo is an increasingly common
shape for this.
