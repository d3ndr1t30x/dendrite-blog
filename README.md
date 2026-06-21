# Dendrite Log

The build-log blog for Dendrite / HackVault. Static site, built with Hugo,
deployed free on GitHub Pages, served on `blog.hakvault.com`.

This exists primarily as **permanent, self-owned documentation** — a record
of what gets built that can't be taken away by a platform policy change or
algorithm shift. Traffic/SEO is a nice-to-have, not the point.

## Why this couldn't be fully tested before you run it

Hugo ships as a compiled binary, not a pure-Python/Node package — and the
build environment used to generate these files can only reach a small
allowlist of domains (npm/pip package registries, not arbitrary GitHub
release-asset hosts). So the actual `hugo` binary couldn't be installed or
run there. Everything in here has been validated as far as possible without
it — TOML config parses correctly, template files have balanced `{{ }}`
delimiters, frontmatter is valid YAML, CSS is balanced and uses the exact
palette specified — but you'll be the first to actually run `hugo server`
against this and see it render. If something's visually off, it's almost
certainly a small template fix, not a structural problem.

## Local setup

1. **Install Hugo** (need the *extended* version, for the SCSS/asset pipeline
   support some themes use — this one doesn't strictly need it yet, but
   extended is the safer default):

   - macOS: `brew install hugo`
   - Windows: `choco install hugo-extended` or `winget install Hugo.Hugo.Extended`
   - Linux: download the `.deb`/`.rpm`/`.tar.gz` from
     https://github.com/gohugoio/hugo/releases (get the `extended` build)

   Verify: `hugo version`

2. **Run it locally:**

   ```bash
   cd dendrite-blog
   hugo server -D
   ```

   `-D` includes draft posts. Visit `http://localhost:1313`. Hugo live-reloads
   on file changes — edit a template or post, save, see it update instantly.

3. **If something looks broken**, the most likely culprits, roughly in order:
   - A typo in a `{{ }}` template tag (Hugo's error output is usually
     specific about file + line)
   - `hugo.toml`'s `theme = "dendrite"` not matching the folder name under
     `themes/` (should be fine here, but double check if you rename anything)
   - Missing front matter on a post (every post needs the `---` delimited
     block at the top)

## Project structure

```
dendrite-blog/
├── hugo.toml                  Site config — title, URL, social links, etc.
├── content/posts/             Your posts live here as .md files
├── archetypes/posts.md        Template used by `hugo new posts/foo.md`
├── themes/dendrite/           The custom theme
│   ├── layouts/
│   │   ├── _default/
│   │   │   ├── baseof.html     Wraps every page (head/header/footer)
│   │   │   ├── single.html     Individual post template
│   │   │   ├── list.html       Generic listing (used for tag pages)
│   │   │   └── terms.html      The /tags/ index page
│   │   ├── partials/
│   │   │   ├── head.html       <head> meta tags, OG tags, stylesheet link
│   │   │   ├── header.html     Site header — skull logo, nav, socials
│   │   │   └── footer.html     Site footer — socials, copyright line
│   │   └── index.html          Homepage — paginated post list
│   └── static/css/style.css    All styling — palette tokens at the top
├── static/CNAME                Tells GitHub Pages the custom domain
└── .github/workflows/hugo.yml  Auto-builds + deploys on every push to main
```

## Writing a new post

```bash
hugo new posts/my-next-build.md
```

This creates a file using the `archetypes/posts.md` template, pre-filled
with `draft: true`. Edit the file, write your content in Markdown below the
`---` frontmatter block, then flip `draft: false` when you're ready to
publish (or just delete that line — the GitHub Actions build uses
`hugo --minify` without `-D`, so drafts won't get published until you flip
that flag).

Frontmatter fields:

```yaml
---
title: "Post Title"
date: 2026-06-20
draft: false
description: "One-sentence summary, shown in the post list and as the meta description."
tags: ["tools", "flask", "build-log"]
---
```

Tags are freeform — use whatever makes sense (`hackvault`, `red-team-learning`,
`tools`, whatever categories emerge naturally). They automatically get their
own listing page at `/tags/<tag-name>/`.

## Deploying

### 1. Push to GitHub

```bash
cd dendrite-blog
git init
git add .
git commit -m "Initial commit — Dendrite Log"
gh repo create dendrite-blog --public --source=. --push
```

(Public repo, since GitHub Pages on the free tier requires a public repo
unless you're on GitHub Pro/Enterprise — worth knowing since `revshells` was
fine as private on Render, but this one needs to be public.)

### 2. Enable GitHub Pages

Repo → **Settings → Pages** → under "Build and deployment", set **Source**
to **GitHub Actions**. The workflow at `.github/workflows/hugo.yml` will
then run automatically on every push to `main` — it installs Hugo, builds
the site, and deploys it to Pages. No manual build step needed on your end
ever again after this.

Watch it run under the repo's **Actions** tab on first push — should take
under a minute.

### 3. Point your domain at it

Cloudflare → `hakvault.com` → **DNS → Records → Add record**:

| Field | Value |
|---|---|
| Type | `CNAME` |
| Name | `blog` |
| Target | `<your-github-username>.github.io` |
| Proxy status | **DNS only (grey cloud)** initially |

Same reasoning as the revshells setup — GitHub needs to issue/validate the
custom domain cleanly on first connection before you put Cloudflare's proxy
in front of it.

Then back in **Settings → Pages**, under "Custom domain", enter
`blog.hakvault.com` and save. GitHub will verify the CNAME DNS record exists,
then automatically provision an HTTPS certificate (usually a few minutes,
occasionally up to ~24h). The `static/CNAME` file in this repo also commits
the custom domain into the build output, so it stays configured even if the
Pages settings ever get reset.

Once you see the cert as active in the Pages settings, you're done. Hard
refresh `blog.hakvault.com` afterward — same caching note as before.

## What's already in here

- `content/posts/building-and-shipping-dendrite-revshell.md` — the first
  post, a build-log writeup of the revshells project. Ready to publish as-is,
  or edit it first if you want to add anything from your own memory of the
  build that I don't have (specific bugs you hit, exact timing, etc.)
