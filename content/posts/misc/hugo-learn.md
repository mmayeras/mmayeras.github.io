---
title: "Build a Hugo blog with LoveIt"
date: 2023-03-24
tags:
- hugo
categories:
- misc
draft: true
code:
  maxShownLines: 500
---

Step-by-step guide to set up a Hugo blog using the [LoveIt theme](https://github.com/dillonzq/LoveIt) and deploy it on GitHub Pages, matching the setup of this site.

## Prerequisites

- `hugo` extended v0.157.0+ in `$PATH` (`brew install hugo` on macOS)
- A GitHub account with a repository named `<username>.github.io`
- `git` configured with SSH access to GitHub

## 1. Create the Hugo site

```bash
hugo new site myblog
cd myblog
git init
```

## 2. Add the LoveIt theme as a submodule

```bash
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
```

Set the theme in `config.toml`:

```toml
theme = "LoveIt"
```

## 3. Configure the site

Minimal working `config.toml` (Hugo 0.157.0+):

```toml
baseURL = "https://<username>.github.io/"
languageCode = "en"
title = "My Blog"
theme = "LoveIt"

[pagination]
  pagerSize = 12

[params]
  defaultTheme = "auto"

  [params.header]
    [params.header.title]
      name = "My Blog"

  [params.footer]
    enable = true

  [params.page.code]
    copy = true
    maxShownLines = 1000

[outputs]
  home     = ["HTML", "RSS", "JSON"]
  page     = ["HTML"]
  section  = ["HTML", "RSS"]
  taxonomy = ["HTML", "RSS"]
  term     = ["HTML"]

[privacy]
  [privacy.x]
    enableDNT = true
  [privacy.youtube]
    privacyEnhanced = true
```

> **Note:** Do not use the old `paginate`, `taxonomyTerm`, `privacy.twitter`, or `:filename` permalink keys — they were removed in Hugo 0.141.0–0.157.0. See the migration table at the end of this article.

## 4. Fix LoveIt shortcode resolution

The LoveIt theme (post-0.3.x) places shortcodes in `layouts/_shortcodes/` which is only resolved at the project level on some platforms. Copy the admonition shortcode to the site root to ensure it works in CI:

```bash
mkdir -p layouts/shortcodes
cp themes/LoveIt/layouts/_shortcodes/admonition.html layouts/shortcodes/admonition.html
```

## 5. Fix code block expansion (render hook timing bug)

The theme reads `maxShownLines` from `Page.Scratch`, which is not populated when render hooks run. Override the partial at the site level:

```bash
mkdir -p layouts/_partials/plugin
```

Create `layouts/_partials/plugin/code-block.html`:

```html
{{- $content := .Content -}}
{{- $lang := .Lang -}}
{{- $pageCode := .Page.Params.code | default dict -}}
{{- $siteCode := .Page.Site.Params.page.code | default dict -}}
{{- $code := merge $siteCode $pageCode -}}
{{- $maxShownLines := $code.maxShownLines | default 10 | int -}}
{{- $copy := $code.copy | default true -}}
{{- $lines := split $content "\n" | len -}}
{{- $options := dict "lineNoStart" 1 "lineNos" true -}}
{{- $options = .Options | partial "function/dict.html" | merge $options -}}
{{- $lineNoStart := $options.lineNoStart | int -}}
{{- $lineNos := $options.lineNos | partial "function/bool.html" -}}
{{- $options = dict "noClasses" false "lineNos" false | merge $options -}}
{{- $result := transform.Highlight $content $lang $options -}}
<div class="code-block{{ if $lineNos }} code-line-numbers{{ end }}{{ if le $lines $maxShownLines }} open{{ end }}" style="counter-reset: code-block {{ sub $lineNoStart 1 }}">
    <div class="code-header language-{{ $lang }}">
        <span class="code-title"><i class="arrow fas fa-angle-right" aria-hidden="true"></i></span>
        <span class="ellipses"><i class="fas fa-ellipsis-h" aria-hidden="true"></i></span>
        {{ if $copy }}<span class="copy" title="{{ T "copyToClipboard" }}"><i class="far fa-copy" aria-hidden="true"></i></span>{{ end }}
    </div>
    {{- $result -}}
</div>
```

## 6. Write your first article

```bash
mkdir -p content/posts/linux/my-first-post
cat > content/posts/linux/my-first-post/index.md << 'EOF'
---
title: "My First Post"
date: 2026-04-14
categories:
  - Linux
tags:
  - rhel
---

Article content here.
EOF
```

## 7. Preview locally

```bash
hugo serve
```

Open [http://localhost:1313](http://localhost:1313).

## 8. Set up GitHub Actions deployment

Create `.github/workflows/hugo.yml`:

```yaml
name: Deploy Hugo site to GitHub Pages

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.157.0
    steps:
      - name: Install Hugo
        run: |
          wget -O ${{ runner.temp }}/hugo.deb \
            https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
            && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build
        env:
          HUGO_ENVIRONMENT: production
        run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

In the GitHub repository settings → Pages, set the source to **GitHub Actions**.

## 9. Deploy

```bash
git add .
git commit -m "initial site setup"
git remote add origin git@github.com:<username>/<username>.github.io.git
git push -u origin main
```

The Actions workflow triggers automatically and deploys to `https://<username>.github.io`.

## Hugo 0.157.0 migration reference

| Removed / deprecated key | Replacement |
|--------------------------|-------------|
| `paginate = N` (top-level) | `[pagination] pagerSize = N` |
| `[privacy.twitter]` | `[privacy.x]` |
| `taxonomyTerm` in outputs | `term` |
| `:filename` permalink token | `:contentbasename` |
| `layouts/shortcodes/` (theme) | `layouts/_shortcodes/` |
| `layouts/partials/` (theme) | `layouts/_partials/` |
