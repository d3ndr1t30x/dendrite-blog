---
title: "Building and Shipping Dendrite RevShell"
date: 2026-06-20
draft: false
description: "I built my own version of revshells.com from scratch with Flask, deployed it for free, and put it live on revshells.hakvault.com in a weekend."
tags: ["tools", "flask", "infosec", "build-log"]
---

I wanted a reverse shell payload reference for the brand — something in the same spirit as revshells.com, but mine: my domain, my design, my payload list. This is the build log.

## Why build this instead of just using revshells.com

Two reasons. First, it's a genuinely useful tool to have under my own domain — a reference I control, that I can extend, that points back to HackVault instead of someone else's project. Second, it was a small enough scope to actually finish. I've learned that the projects I ship are the small ones — narrow scope, clear "done" state, demoable the moment it's live.

## The stack

Kept it simple on purpose:

- **Flask** for the backend — handles live IP/port substitution server-side, so the payload strings are always correctly rendered without duplicating template logic in JavaScript
- **Vanilla JS** on the frontend — fetches payloads from a small JSON API, no framework needed for something this size
- A `shells.py` module holding all the payload templates as plain Python dicts — adding a new shell is just appending an entry, no other code changes required

The payload list covers the usual suspects — bash, python, php, powershell, perl, ruby, socat, netcat, golang, awk — each with notes on requirements and a one-click base64 / PowerShell `-EncodedCommand` encode option where relevant. There's also a listener cheat-sheet panel: `nc`, `rlwrap`, `ncat --ssl`, `socat`, and the Metasploit `multi/handler` setup.

## Things I had to get right

**Input validation actually mattered here.** The LHOST/LPORT fields get reflected straight back into the payload templates and the page itself, so they needed real server-side validation — not just "looks like an IP," but a proper fallback to safe defaults if someone throws garbage (or something malicious) into the field. Tested it with injection attempts during the build; everything sanitizes correctly back to defaults.

**Design needed its own identity**, not just a recolored clone of revshells.com. Landed on a dark theme built around the actual Dendrite palette — black background, `#292A30` panel grey, white text, `#02EA48` green for every interactive element and the brand mark. Simple, legible, consistent with everything else under the HackVault umbrella.

## Shipping it

Free hosting was the other constraint — I wasn't going to pay to host a free tool. Ended up on **Render**, which runs the Flask app on its free tier (with the standard caveat: it spins down after 15 minutes idle, so the first hit after a quiet stretch takes a few seconds to wake up — fine for what this is).

DNS was the last piece. Domain's on Cloudflare, so:

1. Added a CNAME record: `revshells` → the Render service's `.onrender.com` target
2. Left the proxy status grey (DNS only) for the first deploy, since Render needs to complete its own TLS cert issuance against the real origin before Cloudflare's proxy gets involved
3. Once the cert came back active, `revshells.hakvault.com` was live

## What's next

More payload categories, probably a few more reference tools in the same shape — there's a clear pattern now: small, scoped, useful, shippable in a weekend. This blog is partly a record of that pattern as I keep running it.

Live at [revshells.hakvault.com](https://revshells.hakvault.com).
