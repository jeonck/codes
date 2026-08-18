+++
title = "Commit message language is a per-repo decision — and the automation template needs updating too"
date = 2026-08-11
summary = "An English-facing site had commits piling up in Korean. Switching new commits to English seemed simple, until one easy-to-miss spot turned up."
tags = ["git", "convention", "github-actions", "trial-and-error"]
+++

It's easy to assume commit message language is consistent by default — until one repo turned out
not to be.

## The mismatch found

The site itself was English-facing, but its commit log was mostly Korean. The outward-facing text
(the site) and the not-outward-facing text (the commit history) had drifted into different
languages.

## Decision

- **Scope: commit messages only.** Code comments and chat replies stay Korean.
- **New commits only, going forward.** The six existing Korean commits were left untouched —
  force-pushing a public branch to rewrite history for a purely cosmetic reason isn't worth the
  risk.

```bash
# history stays as-is. none of this:
git rebase -i --root
git push --force
```

## The easy-to-miss spot: commits automation generates

It's tempting to think the job is done once human-authored commits switch to English. This repo
also had a daily GitHub Actions workflow that **generates its own commit message**. Miss that
template, and the automated commits keep landing in Korean.

```yaml
# .github/workflows/daily.yml — the part that got missed
- run: |
    git commit -m "데일리 업데이트: $(date +%F)"   # this needed fixing too
```

```yaml
    git commit -m "Daily update: $(date +%F)"
```

## Takeaway

Changing a commit message convention means checking both "commits a human types" and "commits
automation types" separately. The latter is quick to find with a grep across the repo's
workflows.

```bash
grep -rn "git commit -m" .github/workflows/
```
