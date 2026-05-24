---
layout: default
title: About
description: About this site
permalink: /about/
---

# About

These are notes from running the **Microsoft 365 Copilot Agent &amp; app
Package Management API** (`/beta/copilot/admin/catalog/packages`) against a
real tenant and turning the result into a small governance dashboard.

- **Live demo:** [a365graph.ai-news.cz](https://a365graph.ai-news.cz/) — sanitised, read-only.
- **Source:** the same repo this site lives in — [github.com/ostefka/a365graph-blog](https://github.com/ostefka/a365graph-blog).
- **Stack of the demo:** Vite + React + TS + Tailwind + TanStack Table, hosted on
  Azure Static Web Apps (Free tier).
- **Data pipeline:** a Python script that calls the Graph endpoint with MSAL
  device-code auth, scrubs GUIDs / emails / customer names, and writes JSON
  files consumed by the dashboard.

All names, customer references, employee names, GUIDs and email addresses on
the demo site are **synthetic** — real customer names are replaced with
Contoso / Fabrikam, every GUID is reassigned to a `guid-NNNN` token, every
email becomes `redacted@example.com`. Sanitisation rules are applied at build
time and are reviewable in `swa/sanitize.json`.

[← Back to articles]({{ "/" | relative_url }})
