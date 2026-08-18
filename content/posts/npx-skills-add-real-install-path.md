+++
title = "npx skills add installs to ~/.agents/skills, not ~/.claude/skills"
date = 2026-07-26
summary = "The install command exits clean and the files exist on disk, but the skill still won't show up in the session — check the path first."
tags = ["claude-code", "skills", "trial-and-error"]
+++

What to check when a third-party skill was just installed but isn't recognized by the current
session.

## Symptom

```bash
npx skills add Vincentwei1021/video-shotcraft
```

The command exits successfully and the files land on disk, but the skill never shows up in
`/help` or the session's available-skills list.

## Cause

`npx skills add` installs into **`~/.agents/skills/<skill-name>/`**, not
`~/.claude/skills/`. That path is **shared** between Claude Code and Codex. On top of that, an
already-running session doesn't rescan the skills directory — a newly installed skill only shows
up after starting a **new** session.

```bash
ls ~/.agents/skills/
# video-shotcraft shows up here (note: the repo name is "shotcraft", no "r")
```

## Using it right away, without a new session

No need to wait for a new session — just read the SKILL.md directly and follow it in place.

```bash
cat ~/.agents/skills/video-shotcraft/SKILL.md
```

## Takeaway

"Installed but not showing up" usually isn't a failed install — it's one of two things: wrong
path assumption (checking the legacy `~/.claude/skills/`), or a session that hasn't rescanned.
Neither needs a reinstall, just a check.
