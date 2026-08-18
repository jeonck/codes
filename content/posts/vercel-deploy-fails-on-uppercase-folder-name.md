+++
title = "If Vercel deploy fails for no obvious reason, check for uppercase in the folder name"
date = 2026-07-06
summary = "A project folder named CTI trips up both create-next-app and vercel deploy on name validation."
tags = ["vercel", "nextjs", "trial-and-error"]
+++

Scaffolding a Next.js project can hit a wall where nothing looks misconfigured, yet both
scaffolding and deploy fail.

## Symptom

The working directory was named `CTI`. Scaffolding and deploying straight from there:

```bash
cd /Users/mac/ws/claude/CTI
npx create-next-app@latest .     # fails name validation
vercel deploy                    # also fails name validation
```

## Cause

Neither `create-next-app` nor Vercel allow **uppercase characters** in a project/app name. Both
tools tried to reuse the directory name as the project name and rejected it at validation.

## Fix

Rather than renaming the directory (it was already in use for other things and renaming wasn't
desirable), the project was scaffolded with a lowercase name in a temp location, then moved into
the original directory — and the deploy target was linked explicitly with a lowercase project
name via `vercel link`.

```bash
# 1. Scaffold with a lowercase name in a temp location
npx create-next-app@latest /tmp/cti-sentinel

# 2. Move the generated files into the original uppercase directory
mv /tmp/cti-sentinel/* /Users/mac/ws/claude/CTI/

# 3. Deploy by linking to an explicit lowercase project name (avoids automatic name inference)
cd /Users/mac/ws/claude/CTI
vercel link --project cti-sentinel --yes
vercel deploy
```

## Takeaway

The local directory name and the deployed project name are independent — use that to get around
the validation rule without ever renaming the directory itself, via
`vercel link --project <lowercase-name>`.
