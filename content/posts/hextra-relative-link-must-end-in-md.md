+++
title = "Hugo Hextra theme: in-content relative links must end in .md"
date = 2026-07-07
summary = "An extensionless relative link like [lab](../ci-cd/lab-github-actions) renders fine locally but 404s under GitHub Pages' pretty URLs."
tags = ["hugo", "hextra", "github-pages", "trial-and-error"]
+++

On a docs site built with Hugo [Hextra](https://imfing.github.io/hextra/), some in-content
relative links only break after deploy. If `hugo server` looks fine locally but the same page
404s on GitHub Pages, this is a likely cause.

## Cause

Hextra's link render hook (`themes/hextra/layouts/_markup/render-link.html`) only rewrites an
internal link into a clean URL if the link target ends in `.md`.

```go-html-template
{{/* summary of the logic inside render-link.html */}}
{{ if strings.HasSuffix $url.Path ".md" }}
  {{/* .GetPage resolves the page and rewrites to a clean URL */}}
{{ end }}
```

A link without `.md` never enters that branch — it's emitted **verbatim** into the HTML. So
`[lab](../ci-cd/lab-github-actions)` isn't resolved at all; under pretty-URL routing that literal
path doesn't exist, hence the 404.

## The rule

Always suffix internal links with `.md`, whether they point at a page or a section.

```md
<!-- page link -->
[GitHub Actions lab](../ci-cd/lab-github-actions.md)

<!-- section (directory) link -->
[CI/CD section](../ci-cd/_index.md)
```

## Catching every broken link before deploy

Rather than clicking through pages one by one, grep for every relative link missing `.md`:

```bash
grep -rnoE '\]\([^)]+\)' content/ \
  | grep -vE 'https?:|#|mailto:' \
  | grep -v '\.md'
```

An empty result means it's safe to ship. This rule isn't called out prominently in Hextra's own
docs, so it tends to eat a fair bit of debugging time the first time it's hit.
