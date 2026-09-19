+++
title = "46 identical doc-shipping loops in two days — encode the loop, don't search for a skill"
date = 2026-09-19
summary = "Looked for a skill to speed up a content pipeline. Nothing on the registry fit; the real win was a shell script for the mechanical steps, an instructions file for the judgment calls, and a CLI search that replaced browser scraping."
tags = ["claude-code", "skills", "automation", "workflow", "trial-and-error"]
+++

Over two days a static-site project accumulated **46 `content(...)` commits**, and each one
walked the same path by hand: research → verify → write → build → dead-link check → tests →
commit → push → wait for Actions → confirm the live URL → close browser task spaces. Eight to
ten tool calls per document, every time. The question was whether an installable skill could
compress that.

## What the registry had

`npx skills find` across the recurring tasks — web research, scraping, RSS, fact-checking,
link checking, GitHub Actions, Korean writing. Results with real install counts were either
generic methodology prose (a journalism fact-check workflow, less strict than the project's own
"official sources only, date every figure" rule), paid-API wrappers (Parallel deep research,
TypeSafe's typed-judgment model), or tools the repo already had (`checklinks.py`, a weekly
source-validation workflow).

One thing did fit, and it was two shell lines:

```bash
npx -y mcporter call --stdio 'uvx duckduckgo-mcp-server' search query="Gu Thai Duluth GA address" max_results=5
npx -y mcporter call --stdio 'uvx duckduckgo-mcp-server' fetch_content url="https://..."
```

Research had been done by opening DuckDuckGo result pages in an automated browser and scraping
them — 4–5 seconds a query plus task-space bookkeeping. The CLI returns the same results
instantly, runs in parallel, and `fetch_content` returned the body of pages that had been
`403` under `curl` (an Akamai-fronted city site, a JS-rendered county tax page). No skill
install needed; the two commands went into the project's instructions file.

## Where the time actually went

Reading the `writing-skills` guidance made the shape obvious:

> Don't create a skill for project-specific conventions (put them in your instructions file)
> or mechanical constraints (if it's enforceable with a script, automate it).

The loop split cleanly along that line.

{{< mermaid >}}
flowchart LR
  subgraph judgment["Judgment — CLAUDE.md (loaded every session)"]
    R["Research<br/>DDG search + fetch_content<br/>browser only for login/click"]
    V["Verify<br/>official page only<br/>date every figure"]
    W["Write<br/>front matter, sources,<br/>no markdown in YAML notes"]
  end
  subgraph mech["Mechanical — scripts/ship.sh (one command)"]
    B[build] --> L[dead-link check] --> T[tests] --> C[commit + push]
    C --> A["wait for the Actions run<br/>matching HEAD sha"] --> U[live URL 200] --> X[close browser spaces]
  end
  R --> V --> W --> B
{{< /mermaid >}}

- **`scripts/ship.sh <city> "<msg>" [/path/]`** — the six mechanical steps, `set -e`, stops at
  the first failure. `--check` runs only build → links → tests.
- **`CLAUDE.md`** — the rules that need judgment, written against the mistakes actually made
  that week: a program described as existing when it was only proposed, aggregator prices
  quoted as fact, `**bold**` markdown pasted into a YAML note and rendered as literal
  asterisks, cross-city links hand-written when the build already generates them.

## The bug the first run of the script exposed

`ship.sh` waited for "the most recent Actions run" and reported its success. The first real
use shipped only `CLAUDE.md` and the script itself — and the workflow has a `paths:` filter
(`cities/**`, `content/**`, …), so that commit **never triggered a run**. The script happily
reported the previous cron run's `success` as if it belonged to this commit.

```bash
# wrong: whichever run is newest
RID=$(gh run list --workflow daily.yml --limit 1 --json databaseId -q '.[0].databaseId')

# right: the run whose head_sha is this commit; if none appears, say so and skip the wait
SHA=$(git rev-parse HEAD)
RID=$(gh run list --workflow daily.yml --limit 5 --json databaseId,headSha \
      -q ".[] | select(.headSha==\"$SHA\") | .databaseId" | head -1)
[ -z "$RID" ] && echo "this commit did not trigger the deploy workflow (paths filter) — skipping"
```

A deploy check that can pass on someone else's run is worse than no check.

## Testing an instructions file the cheap way

The `writing-skills` rule is *no skill without a failing test first*. For a conventions file the
baseline failures were already on record (the week's mistakes). The green check was a
single-shot Haiku subagent given only `CLAUDE.md` and six retrieval questions — file path for a
new doc, required front-matter keys, what to do with a Yelp opening-hours value, whether to
hand-link another city's site, whether markdown is allowed in a YAML `detail`, the ship
command. Six for six, twenty seconds. Enough to know the file answers what it needs to before
it gets loaded into every future session.

## Two smaller fixes that came out of the same pass

- **Cross-city links, first mention per *section* not per document.** "Michigan" mentioned in
  passing in an early section had been taking the one link, leaving the tax-comparison table
  further down unlinked. Splitting on `<h2>` fixed it.
- **Link to the same document on the other city's site, not its home page.** "The consulate is
  closer than in Michigan or Texas" now lands on those cities' *community* pages — where the
  consulate section actually is — falling back to home only when the slug doesn't exist there.

## Takeaway

When a workflow repeats dozens of times, the skill registry is the wrong first stop. Sort the
steps: the ones a script can enforce go in a script; the ones that need judgment go in the
instructions file that loads every session; the one external tool that helped was a
two-line CLI, not a skill. And test the script on a commit that *doesn't* trigger your
deploy — that's where the false positive was hiding.
