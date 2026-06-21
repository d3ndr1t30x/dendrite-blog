---
title: "Setting Up This Blog: A Log of the Build Log"
date: 2026-06-21
draft: false
description: "How blog.hakvault.com actually got stood up — Hugo, GitHub Pages, and three separate DNS/Actions failures before it worked."
tags: ["build-log", "hugo", "infosec", "infra"]
---

This blog exists because I wanted permanent documentation that nobody can take from me. Platforms get demonetized, algorithms shift, accounts get flagged — none of that happens to a folder of Markdown files sitting in a git repo I control. Fitting, then, that the first real post here is about standing the thing up. Wasn't a clean one-shot deploy. Worth recording exactly where it went sideways.

## The stack decision

Started by almost going down the Ghost-on-a-VPS route — self-hosted Ghost via Docker on Oracle's free-tier cloud. Backed out of it once I was honest about what I actually wanted: zero ongoing maintenance. A VPS means an OS to patch, a Docker daemon to babysit, the occasional "why did my server stop responding" debugging session. None of that is what I'm here for.

Landed on the actual right shape instead: **Hugo** (static site generator) building plain Markdown into static HTML, hosted free on **GitHub Pages**, pointed at a `blog.hakvault.com` subdomain through Cloudflare DNS. Nothing running, nothing to patch. Write a file, push, done.

## Build environment vs. real environment

First wrinkle, before any DNS was even involved: Hugo ships as a compiled binary, not something installable through plain pip/npm in a sandboxed build environment. The actual site source — config, theme templates, the first post — got built and structurally validated (TOML parses, Go template braces balance, frontmatter is valid YAML) without ever actually running the `hugo` binary. First real render happened on my own machine.

Worth normalizing: building something "blind" like that means the first real test is genuinely the first real test. It worked first try, which was a relief — but it's a reminder that structural validation and "actually renders correctly" are different bars.

## Installing Hugo on Windows

Downloaded the extended binary straight from the GitHub releases page rather than going through a package manager. That meant manually:

1. Unzipping the `.exe` to a permanent location (`C:\Hugo\bin\`)
2. Adding that folder to system PATH via Environment Variables
3. Opening a **new** terminal — PATH changes don't apply retroactively to an already-open shell, which cost a few minutes of "why isn't this working" before remembering that

Once `hugo version` printed cleanly, that part was done. Would probably use `winget install Hugo.Hugo.Extended` next time — same result, fewer manual steps.

## The `<your-username>` placeholder mistake

Copy-pasted a git remote command that still had a literal `<your-username>` placeholder in it instead of swapping in the real GitHub handle first. `git push` failed with an HTTP 400 — not a particularly clear error for "your remote URL is garbage." Fixed with:

```
git remote set-url origin https://github.com/d3ndr1t30x/dendrite-blog.git
```

`git remote add` silently does nothing if a remote already exists under that name — it doesn't overwrite, it just errors quietly enough to miss. `set-url` is the one that actually replaces it.

## GitHub Actions: round one

Pushed the repo, workflow kicked off automatically, and immediately failed:

```
HttpError: Not Found - .../rest/pages/pages#get-a-apiname-pages-site
Get Pages site failed. Please verify that the repository has Pages enabled
and configured to build using GitHub Actions...
```

Reasonable guess at the time: the Pages "Source" setting hadn't actually saved as "GitHub Actions" before the push triggered the workflow run — a timing issue. Re-ran the job. Same error.

## GitHub Actions: round two

Tried having the workflow auto-enable Pages itself via the `configure-pages` action's `enablement: true` parameter, to sidestep needing the UI setting to be exactly right beforehand. Different error came back:

```
HttpError: Resource not accessible by integration - .../rest/pages/pages#create-a-apiname-pages-site
Create Pages site failed.
```

More informative, in hindsight — this confirmed the Pages site genuinely didn't exist on GitHub's backend yet, and the Actions bot doesn't have permission to *create* one from scratch via API. Only a repo admin clicking through the actual web UI can do that part. No way around it with workflow permissions alone.

## Finding the actual setting

The repo's Settings → Pages screen wasn't showing what I expected — kept seeing a sidebar of mostly-empty bullet points and a "Verified domains" section, no obvious Source dropdown. Turned out the real control was further down the same page, past content that wasn't rendering fully on first load. Found it eventually:

```
GitHub Pages is currently disabled. Select a source below to enable...
Source: Branch
```

There it was — Source was set to "Branch," never actually switched to "GitHub Actions" despite an earlier attempt to change it. Flipped it. That was the real fix; the `enablement: true` workaround got reverted afterward since it wasn't solving the actual problem.

## DNS: the CNAME that wasn't there

With Pages source correctly set, the build/deploy went green. Site existed. Custom domain didn't work yet.

```
DNS check unsuccessful
blog.hakvault.com is improperly configured
Domain's DNS record could not be retrieved (InvalidDNSError).
```

Ran `nslookup -type=CNAME blog.hakvault.com` to check what was actually resolving. Got back a zone SOA record instead of a CNAME answer — DNS's way of saying "no CNAME exists for that name, here's the zone info instead." The record either never saved in Cloudflare or wasn't there yet.

Added it properly: CNAME, `blog` → `d3ndr1t30x.github.io`, proxy status set to grey (DNS only) rather than orange (proxied) — GitHub needs to see the real DNS target during its verification handshake, and Cloudflare's proxy can mask that on first issuance.

## DNS round two: wrong error, right direction

After the CNAME was actually in place, the error changed:

```
Domain does not resolve to the GitHub Pages server (NotServedByPagesError).
```

`nslookup` now correctly returned the CNAME target (`d3ndr1t30x.github.io`), proxy was confirmed grey, so the DNS itself was right. This one turned out to be GitHub's verification check just being stale — it had cached an earlier failed check and hadn't re-run since the DNS got fixed. Fix was almost annoyingly simple: clear the custom domain field in Pages settings, save, wait a few seconds, type it back in, save again. Forces a fresh check instead of relying on the cached result. Came back green.

## The last one: baseURL

Domain showed verified. Cert issued. Visited `blog.hakvault.com` anyway — 404, GitHub's generic "nothing here" page.

Diagnostic step that actually narrowed it down: checked the raw `https://d3ndr1t30x.github.io/dendrite-blog/` URL directly. That one worked fine. Which meant the build and deploy were both completely correct — the problem was isolated specifically to the custom domain routing, not the site itself.

Root cause was in the GitHub Actions workflow file, not DNS or Pages settings at all:

```yaml
hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
```

That dynamic `base_url` value gets calculated by the `configure-pages` action — and at build time, before the custom domain was fully confirmed active, it had calculated the default `github.io` project-pages URL instead of the actual custom domain. Every internal link Hugo generated during that build was baked in against the wrong base. Worked on the raw URL by coincidence (since that *was* the base it built against), broke on the real domain.

Fix: stop relying on the dynamic value, hardcode the real one.

```yaml
hugo --minify --baseURL "https://blog.hakvault.com/"
```

Pushed. Rebuilt. Domain worked.

## What actually mattered, in order

Looking back at the failure sequence, the pattern was: **each error message was real and specific, but each one was also just one layer of a stack of separate problems** — Pages-not-enabled, then DNS-not-set, then DNS-not-rechecked, then baseURL-baked-wrong. None of them were the same bug wearing different sentences; they were four genuinely different things that all had to be correct simultaneously for the site to resolve. Worth remembering next time something like this drags out — don't assume round two's error is just round one persisting. Read it fresh.

## Result

`blog.hakvault.com` — live, auto-deploying on every push via GitHub Actions, zero servers to maintain, fully owned. The actual setup, once correct, takes about ten minutes for the next person doing this from scratch. Getting there the first time took a lot longer than that.

Cheatsheet for posting from here on is in the Obsidian vault. Next post should be something that isn't about the blog itself.
