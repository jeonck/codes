+++
title = "LaTeX just doesn't work under Docusaurus MDX v3 (future.v4) — stop fighting it"
date = 2026-06-16
summary = "Installing remark-math + rehype-katex exactly by the book still breaks the build, because braces inside $$...$$ get parsed as JSX expressions."
tags = ["docusaurus", "mdx", "latex", "trial-and-error"]
+++

On a Docusaurus project with `future: { v4: true }` (MDX v3) enabled, adding math notation
(`$...$`, `$$...$$`) fails in a specific, repeatable way.

## Symptom

```bash
npm install remark-math@6 rehype-katex@7
```

Wiring the plugins into `docusaurus.config.ts` exactly as documented still produces an acorn
parse error at build time.

## Cause

MDX v3's `micromark-extension-mdx-expression` runs **before** `remark-math`. So a `{` inside a
block like `$$ C = M^e \mod n $$` gets mistaken for the start of a JSX expression and parsing
fails right there. No combination of `remark-math`/`rehype-katex` versions fixes this — it's an
execution-order issue, not a version-compat issue.

## Fix: drop it, use code spans + Unicode instead

Don't fight it — route around it.

```md
Inline math as code:
`C = M^e mod n`

Greek letters typed directly as Unicode:
φ(n) = (p-1)(q-1)
```

Best to not add `remarkMath` / `rehypeKatex` / the KaTeX CSS to `docusaurus.config.ts` at all —
it saves debugging time. This only bites on `future.v4` (MDX v3); check whether any docs lean
heavily on math notation before migrating to it.
