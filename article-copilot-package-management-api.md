---
layout: default
title: "What a tenant of 258 Copilot agents actually looks like — touring the M365 Copilot Package Management API"
description: "A long, opinionated read on the Microsoft 365 Copilot Package Management API — what it returns, the things its docs don't tell you, and what we found running it against a real tenant with 258 custom agents."
permalink: /article-copilot-package-management-api
---

<p style="font-size: 0.95em; margin: 0 0 1.5em 0;">
  <strong>English</strong> &nbsp;|&nbsp;
  <a href="{{ '/article-copilot-package-management-api-cs' | relative_url }}">Česky &rarr;</a>
  &nbsp;|&nbsp;
  <a href="{{ '/' | relative_url }}">← Back to articles</a>
</p>

# What a tenant of 258 Copilot agents actually looks like

> **TL;DR** — Microsoft shipped a Graph endpoint
> (`/beta/copilot/admin/catalog/packages`) that *finally* lets a Copilot admin
> see every custom agent, declarative copilot, bot, and Office add-in
> in their tenant — across Copilot Studio, Agent Builder, Teams Toolkit,
> SharePoint, AI Foundry, and sideloaded packages — in one paged JSON feed.
> We ran it against a real tenant, found 258 packages, built a static dashboard
> on top of it, and the picture it paints of "agent sprawl" is more interesting
> than any slide deck.
>
> Live demo (sanitised, read-only): **[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**

![Dashboard summary across 258 custom Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

## Why this matters

Every Microsoft 365 tenant that turned on Copilot in the last 18 months ended
up with the same shape of problem:

- Marketing built a "Brand Voice Coach" in **Copilot Studio**.
- A developer published a "Knowledge Q&amp;A" via **Agent Builder** from chat.
- IT Ops sideloaded a **Teams Toolkit** custom-engine agent for ServiceNow.
- A SharePoint admin created an **AgentSkill** for "summarise this document".
- An MVP enabled three Microsoft-made declarative agents from the store.
- A vendor delivered a packaged **bot** with a 2024-vintage `manifestVersion`.

Each lives in a different builder, ships in a different SKU, and — until
recently — was governed by a different blade in a different admin portal.
Trying to answer the question *"how many AI agents do we actually have, and who
owns them?"* meant clicking through Teams Admin Center, the Power Platform
admin centre, the SharePoint admin centre, and a CSV your tenant admin
exported manually.

The **Copilot Agent &amp; app Package Management API** (preview) fixes that
*from the data side*. It doesn't unify the experiences — it unifies the
**inventory**. Every package that can be installed, enabled, or assigned in
Microsoft 365 Copilot shows up in one paged collection, with the same shape,
the same governance fields, and the same filter grammar.

That's the only ingredient missing from doing serious **agent governance**.

## The API in one screen

**Base URL:** `https://graph.microsoft.com/beta/copilot/admin/catalog/packages`

**Scope:** `CopilotPackages.Read.All` (delegated — see the gotcha below)

**Method:** `GET` (list and detail), `PATCH` (block / unblock), `DELETE` (remove)

### List endpoint

```
GET /beta/copilot/admin/catalog/packages
    ?$filter=supportedHosts/any(h:h eq 'Copilot')
    &amp;$top=100
```

Returns a standard OData v4 paged collection: `value[]` + `@odata.nextLink`
until you've drained the set. No `$count`, no `$expand`, but `$filter` is
remarkably flexible.

### Detail endpoint

```
GET /beta/copilot/admin/catalog/packages/{id}
```

Returns everything in the list payload, **plus**:

- `availableTo` / `deployedTo` enums (more on these below — they're the
  governance gold)
- `installAssignedUsers` / `installAssignedGroups` — *who can install this*
- `enabledAssignedUsers` / `enabledAssignedGroups` — *who has it on right now*
- `elementDefinitions[]` — manifests for every constituent
  (declarative agent, bot, action, etc.) including the **AAD App ID**, manifest
  ID, and (for declarative agents) the inline `instructions` and
  `capabilities` block

The list endpoint is enough for a dashboard. The detail endpoint is what you
need for actual lifecycle work (find the owner of orphan agents, block a
package by ID, validate the manifest version inside).

### Mutating endpoints

```
PATCH  /beta/copilot/admin/catalog/packages/{id}    { "isBlocked": true }
DELETE /beta/copilot/admin/catalog/packages/{id}
```

These are the levers admins actually want: stop a misbehaving agent from being
installed by anyone, or remove the package from the tenant. (Removing the
*package* doesn't undeploy the underlying Bot resource or AAD App Registration
— it just takes it out of the Copilot catalog.)

## The gotcha nobody warns you about: this is **delegated only**

You will read the API reference, copy your client-credentials boilerplate from
the last Graph automation you wrote, paste in your `CopilotPackages.Read.All`
**application** permission … and get a flat HTTP 403 with a body that says
nothing useful.

**The Copilot package endpoints don't accept app-only tokens.** They require a
**user** token, and that user has to be a Copilot Admin (or Global / Cloud App
Admin). The team confirmed this is intentional during preview — the API surface
is shaped for *human* admins running governance workflows, not for unattended
service principals.

Practical consequence: anything you build on this needs a sign-in. For our
inventory tool we use **MSAL device-code flow** with a serialised on-disk
token cache, so it's a one-time login per machine and silent on every later
run. The relevant code lives in `agentsreports/auth.py` — about 120 lines,
mostly because we wanted nice progress logs through the polling loop.

If you can't do device-code (CI pipeline, headless box), the fallback is:

```bash
az login --scope https://graph.microsoft.com/CopilotPackages.Read.All \
         --allow-no-subscriptions
az account get-access-token \
    --scope https://graph.microsoft.com/CopilotPackages.Read.All \
    --query accessToken -o tsv > .token
```

…and read the token from `$COPILOT_TOKEN` or `.token`. Tokens last an hour, so
this is fine for ad-hoc analysis but not for a long-running service.

## The schema, decoded

Every package object has roughly this shape:

```jsonc
{
  "id": "guid",                 // package ID (the one for /packages/{id})
  "displayName": "Brand Voice Coach",
  "shortDescription": "…",
  "publisher": "Contoso Marketing",
  "publisherDomain": "contoso.com",
  "version": "1.4.2",
  "manifestId": "guid",         // Teams manifest ID
  "appId": "guid",              // AAD App Registration ID
  "isBlocked": false,
  "lastModifiedDateTime": "2026-04-30T12:13:14Z",
  "createdDateTime": "2025-09-01T08:00:00Z",
  "type": "shared|lob|sideloaded",   // origin / lifecycle bucket
  "supportedHosts":  ["Copilot", "Teams", "Outlook"],
  "elementTypes":   ["DeclarativeCopilots", "Bots", "AgentSkills"],
  "supportedBuilders": ["Copilot Studio", "Agent Builder", "Agents Toolkit",
                        "SharePoint", "Foundry", "Unspecified"],
  "availableTo": "everyone|specific|admin|nobody",
  "deployedTo":  "everyone|specific|admin|nobody",
  "ownerId":     "guid|null"
}
```

A handful of these deserve a closer look.

### `type` — the lifecycle origin

- `shared` — **published** through the official builder pipelines (Copilot
  Studio publish, Agent Builder publish, Foundry agent registration). Counts
  as first-class tenant content.
- `lob` — **line-of-business**: privately uploaded for the tenant, typically a
  Teams Toolkit-built package that was uploaded through the Teams Admin Center.
- `sideloaded` — installed directly by an end user via "Upload custom app",
  not promoted to tenant inventory. **Sideloaded packages are how shadow
  agents get into the tenant** — they bypass admin review.

In our real tenant we found `shared:212`, `lob:31`, `sideloaded:15`. Fifteen
sideloads — every one of them needs to be examined and either promoted, blocked
or removed.

### `elementTypes` — what's actually inside

A package can carry multiple **element definitions** at once. The interesting
ones are:

- **`DeclarativeCopilots`** — a YAML-ish agent manifest with instructions,
  conversation starters, capabilities (Graph Connectors, code interpreter,
  image generator), and `actions[]` (OpenAPI / Power Platform action plugins).
  These are *the* Microsoft 365 Copilot agents in the everyday sense.
- **`AgentMetadatas`** — Copilot Studio-built bots, both declarative and
  custom-engine. The metadata wraps the bot's Power Platform environment ID
  and the published bot ID.
- **`Bots`** — classic Azure Bot Service bots, including custom-engine agents
  (CEAs) talking to a Container App or Functions backend. Most legacy "Teams
  bots" land here.
- **`AgentSkills`** — reusable LLM tool definitions stored as Markdown
  front-matter in SharePoint, exposed to agents on-demand. New, lightly
  documented, and very interesting (see screenshot).
- **`AgenticUserTemplates`** — pre-canned conversation templates for the
  agentic Copilot UI (one in our tenant — clearly someone experimenting).

The breakdown in our real tenant: 94 AgentMetadatas, 86 DeclarativeCopilots,
74 Bots, 3 AgentSkills, 1 AgenticUserTemplate.

### `availableTo` vs `deployedTo` — the field combo that runs your audit

This is the **single most under-documented** pair of fields in the entire API,
and it's the one that matters most for governance.

| Field         | Meaning                                                    |
| ------------- | ---------------------------------------------------------- |
| `availableTo` | *who is allowed to install this package* (assignment)      |
| `deployedTo`  | *who currently has it enabled / pinned* (active footprint) |

Both take the same enum: `everyone`, `specific`, `admin`, `nobody`.

Combinations and what they mean in practice:

- `available=everyone, deployed=everyone` — **broadly active**. A real
  tenant-wide agent. Treat as production. (Most Microsoft-made content.)
- `available=everyone, deployed=specific` — **available but adoption is slow**.
  Marketing rolled it out but the org isn't picking it up. Adoption signal.
- `available=specific, deployed=specific` — **scoped pilot**. Healthy state
  for a not-yet-GA agent. Verify the assignment groups still exist.
- `available=admin, deployed=admin` — **admin-only**. Often legacy bots that
  were never published broadly. **Candidates for removal.**
- `available=nobody, deployed=nobody` — orphaned. Probably published in error
  and never followed up. Safe to delete.
- `available=nobody, deployed=specific` — **danger**. The package can no longer
  be installed by anyone, but users who already have it still have it. This is
  the state right after an emergency block and *before* you clean up. If you
  see this and you didn't block it intentionally, something went wrong.

For the dashboard at [a365graph.ai-news.cz](https://a365graph.ai-news.cz/) we
visualise this combo on the **Governance** tab — it's by far the most useful
single thing to look at.

### `ownerId` — and what `null` means

`ownerId` is *the AAD ObjectId of the user listed as the package owner*. In a
well-run tenant every custom package has one. In a real tenant, many don't —
and **that's the bug, not the feature**:

- For declarative agents created from **chat ("New Copilot agent")**, the
  builder writes the agent into your personal sandbox and `ownerId` is *you*.
- For **shared / published** declarative agents from Copilot Studio, `ownerId`
  comes from the Power Platform environment owner (often a service account).
- For **Microsoft-made content** and SharePoint AgentSkills, `ownerId` is
  routinely null. Nothing to do about that.
- For **third-party bots sideloaded via Teams Toolkit**, `ownerId` is whatever
  the developer set during package publish. Often null. **These are the
  agents you cannot ring up when something breaks.**

In our tenant: **29 orphan agents** with `ownerId=null`. Most are Microsoft
samples and "Your developer name" defaults that someone forgot to change.
But two were business-critical bots whose original developer left the company.

The Governance tab below surfaces this exact list:

![Governance tab: blocked agents and orphan agents]({{ "/assets/images/governance.png" | relative_url }})

### `manifestId` &amp; `appId` — the cross-references

Every package keeps two extra IDs that let you cross-reference with the *real*
backing resources:

- `manifestId` → the **Teams app manifest** GUID (matches `id` in
  `manifest.json`). Use this to look up a package in the Teams Admin Center.
- `appId` → the **AAD App Registration** Application ID. Use this to find the
  bot's Azure Bot resource, audit App-permission grants, find Conditional
  Access assignments, etc.

If you're auditing security posture, **the `appId` is what you need**. The
Graph package row tells you a Copilot custom agent exists; the corresponding
App Registration tells you what permissions it was granted, whether the secret
has been rotated, and whether anyone signed in with it lately.

## What we found in a real tenant: 258 packages

We ran the inventory script against a real tenant (data is fully sanitised in
the public dashboard — names, GUIDs, emails are all replaced with placeholders).
The headline numbers:

| Metric                     | Count | Notes                                           |
| -------------------------- | ----: | ----------------------------------------------- |
| Custom packages            |   258 | lob + shared + sideloaded                       |
| Blocked                    |     1 | `isBlocked=true` — admin hard-blocked it        |
| Orphan agents              |    29 | no `ownerId` — no individual maintainer         |
| Outdated manifest          |    53 | `manifestVersion` ≤ 1.22 or `devPreview`        |
| Copilot-hosted agents      |   146 | live in the Microsoft 365 Copilot host          |
| Built with Copilot Studio  |   144 |                                                 |
| Built with Agent Builder   |    61 |                                                 |
| Built with Teams Toolkit   |    16 |                                                 |
| Sideloaded (shadow IT?)    |    15 |                                                 |
| Bots (Azure Bot Service)   |    74 | classic CEAs + legacy Teams bots                |
| Declarative copilots       |    86 | the modern Copilot agent surface                |

A few takeaways that surprised even the team running the tenant:

1. **The platform mix is genuinely fragmented.** Copilot Studio dominates by
   count (144), but it's *not* a majority — 114 packages were built somewhere
   else, with five other builders all in active use. Any governance story you
   tell about "our Copilot Studio agents" is missing 44% of the surface.

2. **53 packages on outdated manifests is a TLS-cert-style time bomb.** The
   Teams app manifest schema is now at v1.22 and the **declarative agents
   that don't include `webApplicationInfo` simply render with blank input
   fields** — adaptive cards have `Input.Text` / `Input.ChoiceSet`
   stripped client-side. (We hit this exact issue building one of the
   in-house agents.)

3. **Sideloads aren't necessarily bad — but they're invisible to most admin
   tooling.** Fifteen `sideloaded` packages were users uploading their own
   `.zip` from `atk install`. Two were prototypes that had quietly become
   important. None were in the Teams Admin Center catalog. Without this API,
   we wouldn't have known they existed.

4. **The "Cowork Skills" surface is real and growing.** `AgentSkills`
   element type is brand new — three in our tenant, all exposed via the
   inline-skills system that reads Markdown front-matter from a known
   SharePoint folder. This is the **agent-tool-mesh** building blocks that
   Microsoft is rolling out under the Copilot Studio "skills" banner.

![Cowork Skills: reusable LLM tool definitions stored in SharePoint]({{ "/assets/images/cowork-skills.png" | relative_url }})

## The dashboard: [a365graph.ai-news.cz](https://a365graph.ai-news.cz/)

We turned the JSON inventory into a fully static dashboard. **No backend, no
database, no auth** — the API extract is pre-sanitised, baked into a handful of
JSON files at build time, and served from Azure Static Web Apps on the Free
tier behind a managed TLS cert.

It's an existence proof: the whole governance surface fits in a 234 kB JS
bundle (72 kB gzipped) and renders in under a second.

What's on it:

### 1. Dashboard — the single-screen overview

![Dashboard summary across 258 custom Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

Four KPI tiles (Custom packages · Blocked · Orphan agents · Outdated manifest)
and four breakdown bars (Builder · Element Type · Source Type · Host). This
is what a Copilot admin should be looking at every Monday morning.

### 2. Agents — searchable inventory

![Inventory table: every custom Copilot package in the tenant]({{ "/assets/images/agents.png" | relative_url }})

258 rows in a TanStack-Table with full-text search across name / publisher /
ID, and three faceted filters (Type · Builder · Element). Click any row to
drop into a per-package detail page with the full JSON, the element
definitions, and the assigned user / group lists.

### 3. Cowork Skills — the AgentSkills surface

![AgentSkills: reusable LLM skill definitions discoverable across agents]({{ "/assets/images/cowork-skills.png" | relative_url }})

Cards for every `AgentSkills` element in the tenant. Each one shows the
SharePoint folder it lives in, the site ID it's hosted on, the `embedded`
flag, and the full description (which doubles as the LLM tool description).
These are the building blocks the next wave of Copilot agents will compose at
runtime.

### 4. Governance — the action queue

![Governance: blocked + orphan agents requiring admin attention]({{ "/assets/images/governance.png" | relative_url }})

Two tables that an admin can act on directly: **blocked** (already hard-blocked
— is it intentional?) and **orphan** (no owner — assign one, or remove).
Nothing fancy, but this is the screen that turns a JSON dump into a workflow.

### What it's built from

- **Frontend:** Vite + React + TypeScript + Tailwind + TanStack Table.
- **Data pipeline:** a Python script (`scripts/build_swa_data.py`) that hits the
  Graph endpoint, walks every JSON node, scrubs GUIDs to `guid-0001`-style
  tokens, redacts email addresses, applies a small `sanitize.json` of
  customer-name replacements (e.g. real names → "Contoso" / "Fabrikam"), and
  emits four JSON files into `public/data/`. Zero PII leaves the build step.
- **Hosting:** Azure Static Web Apps (Free) in West Europe, managed TLS,
  custom domain `a365graph.ai-news.cz`. Cost: 0 EUR / month.
- **Deploy:** one shell script (`swa/deploy.sh`) — Python rebuild, Vite build,
  fetch deployment token, push via `@azure/static-web-apps-cli`. ~30 s
  end-to-end.

The whole thing is so small it's almost embarrassing. The point is: **once
you have the API, the rest is trivial.** Most of the engineering effort went
into sanitisation, not visualisation.

## Things we tried that you can copy

Five small patterns we landed on that turned out to be high-value:

### 1. Use `$filter` server-side, not client-side

OData filters compose well on this endpoint and the server already paginates.
Don't pull all 258 packages and filter in Python — push the filter.

```python
# "agents on Microsoft 365 Copilot, modified in the last 30 days"
$filter = (
    "supportedHosts/any(h:h eq 'Copilot') and "
    "elementTypes/any(e:e eq 'DeclarativeCopilots') and "
    "lastModifiedDateTime gt 2026-04-24T00:00:00Z"
)
```

### 2. Cache the device-code refresh token to disk

MSAL ships a `SerializableTokenCache`. Persist it to a file under
`.token_cache.bin` (gitignored) so subsequent runs are silent. We added a
tiny check for a `.token` file as a fallback for headless boxes.

### 3. Retry on 424 *and* 429

The Graph package endpoint sometimes throttles with **HTTP 424** instead of
the standard 429. This is a beta-API quirk. Our `GraphClient` treats both as
retryable with exponential backoff:

```python
retryable = {424, 429}
if resp.status_code in retryable or resp.status_code >= 500:
    wait = retry_after or min(2 ** attempt, 30)
    time.sleep(wait); continue
```

### 4. Always pull `--details` for governance

The list endpoint is fast but doesn't carry `installAssignedUsers` / `groups`
or the `elementDefinitions`. For real governance you want the per-package
detail. Budget ~2 ms per call on a warm cache, ~50 ms cold — for 258 packages
that's about 15 s end-to-end.

### 5. Pre-sanitise before publishing

Any real inventory has customer names, employee names, email addresses,
SharePoint site IDs, and Azure subscription GUIDs scattered through it. If
you're publishing the dashboard publicly (as we are), do the redaction at
**build time**, write the sanitisation rules into a JSON config you can
review, and run `grep` against the published artifact as a unit test:

```bash
# fails the build if any real GUID slips through
! grep -REo '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' \
    swa/public/data/ | grep -v 'guid-[0-9]'
```

## What's still missing from the API

The endpoint is excellent. Not perfect.

- **No `$count`** on the collection. You don't know how many pages you have
  until you've drained them. Not a showstopper, but it makes progress UIs
  harder.
- **No `$expand` for `elementDefinitions`.** You always do an N+1 to get them.
  For 258 packages that's 258 extra HTTP calls. Cacheable, but tedious.
- **No webhooks / change feed.** There is no
  `delta()` on the collection — you can't subscribe to "tell me when a new
  agent is published". You have to poll and diff. We do this nightly to a
  little JSON snapshot in blob storage; *that* diff is what drives the
  Slack alert.
- **`devPreview` manifest agents leak through.** Old declarative agents that
  were built against the dev-preview manifest schema show up alongside
  current packages with no visual marker. They're discoverable via the
  `manifestVersion` inside `elementDefinitions[].manifest`, but parsing
  semver is on you.
- **App-only is still blocked.** Already covered. The day this lands,
  nightly inventory becomes trivial in a service principal.

None of this is blocking — it's the kind of polish that a preview API
collects on the way to GA.

## What we'd like next from Microsoft

Three asks, in priority order:

1. **App-only support** with a tightly scoped permission
   (`CopilotPackages.Read.All` already works delegated; ship the app-only
   variant). Today every nightly-inventory automation needs a human-bound
   account.
2. **A `delta()` collection** on the packages endpoint, the same shape as
   `users/delta` and `groups/delta`. That single addition would replace 80% of
   the polling-and-diffing infrastructure people are about to build.
3. **An `assignedUsers/$count`** sub-resource on each package. We want to
   answer "which agent is *actually* used by 10k+ users?" without enumerating
   every assignment.

## Try it yourself

The code that powers the live demo is small enough to read in an afternoon:

- Inventory tool — the API client + report generator: `agentsreports/`
  (Python · MSAL device-code · OData paging · governance metrics)
- Static dashboard — Vite / React / TS, no backend: `swa/`
  (TanStack Table · Tailwind · React Router · 234 kB total bundle)
- Build &amp; sanitise pipeline: `scripts/build_swa_data.py`
  (GUID scrubber · email redactor · ordered name replacements)

The repo is structured so you can:

1. Set `TENANT_ID` and `CLIENT_ID` in `.env`.
2. `python -m agentsreports.inventory --details` — pulls `out/packages.json`,
   `out/packages.csv`, `out/report.md`.
3. *(Optional)* `bash swa/deploy.sh` to publish your own dashboard.

If you want to **see what it looks like first**, the sanitised demo lives
permanently at **[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**.

---

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/article-copilot-package-management-api-cs' | relative_url }}">Česky &rarr;</a>
  &nbsp;|&nbsp;
  <a href="{{ '/' | relative_url }}">← Back to articles</a>
</p>
