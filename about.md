---
layout: page
title: About
permalink: /about/
---

These are notes from running the **Microsoft 365 Copilot Agent &amp; app
Package Management API** (`/beta/copilot/admin/catalog/packages`) against a
real tenant and turning the result into a small governance dashboard.

- **Live demo:** [a365graph.ai-news.cz](https://a365graph.ai-news.cz/) — sanitised, read-only.
- **Source repo:** the same repo this site lives in (`docs/` folder is the site).
- **Stack of the demo:** Vite + React + TS + Tailwind + TanStack Table, hosted on
  Azure Static Web Apps (Free tier).
- **Data pipeline:** a Python script that calls the Graph endpoint with MSAL
  device-code auth, scrubs GUIDs/emails/names, and writes JSON files.

All names, customer references, employee names, GUIDs and email addresses on
the demo site are **synthetic** — `ČEZ → Fabrikam`, `PPF → Contoso`,
`Ondrej Stefka → Alex Example`, every GUID is reassigned to a `guid-NNNN`
token, every email becomes `redacted@example.com`. The sanitisation rules
live in `swa/sanitize.json` and are applied at build time.

Open a PR if you spot a mistake. The blog posts are plain Markdown under
[`docs/_posts/`]({{ '/' | absolute_url }}).
