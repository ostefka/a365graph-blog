---
layout: post
title: "Jak vypadá tenant s 258 Copilot agenty — průvodce M365 Copilot Package Management API"
date: 2026-05-24 18:00:00 +0200
lang: cs
categories: copilot governance graph-api
tags: [microsoft-365-copilot, agent-365, graph-api, governance, agent-sprawl, jekyll]
excerpt: >
  Dlouhé, názorové čtení o Microsoft 365 Copilot Agent &amp; app
  Package Management API — co vrací, co vám oficiální dokumentace
  neřekne, a co jsme zjistili po spuštění proti reálnému tenantu
  s 258 vlastními agenty. Včetně živého demo dashboardu na
  a365graph.ai-news.cz.
permalink: /2026/05/24/copilot-package-management-api-cs/
---

<p style="font-size: 0.9em; color: #666;">
  <a href="{{ '/2026/05/24/copilot-package-management-api/' | relative_url }}">🇬🇧 English</a> &middot;
  🇨🇿 Česká verze
</p>

> **TL;DR** — Microsoft vydal Graph endpoint
> (`/beta/copilot/admin/catalog/packages`), který *konečně* dovolí Copilot
> adminovi vidět všechny vlastní agenty, deklarativní copiloty, boty
> a Office add-iny ve svém tenantu — napříč Copilot Studio, Agent Builder,
> Teams Toolkit, SharePoint, AI Foundry i sideloadovanými balíčky —
> v jednom stránkovaném JSON feedu. Pustili jsme to proti reálnému
> tenantu, našli 258 packages, postavili nad tím statický dashboard, a obrázek
> "agent sprawl", který z toho vypadl, je zajímavější než kterákoli prezentace.
>
> Živé demo (sanitizované, read-only): **[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**
>
> ![Souhrnný dashboard pro 258 vlastních Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

---

## Proč na tom záleží

Každý Microsoft 365 tenant, který za posledních 18 měsíců zapnul Copilot,
skončil se stejným tvarem problému:

- Marketing si v **Copilot Studiu** postavil "Brand Voice Coach".
- Vývojář publikoval "Knowledge Q&amp;A" přes **Agent Builder** přímo z chatu.
- IT Ops sideloadoval custom-engine agenta pro ServiceNow přes **Teams Toolkit**.
- SharePoint admin vytvořil **AgentSkill** pro "shrň tento dokument".
- MVP zapnul tři Microsoftem dodávané deklarativní agenty ze storu.
- Dodavatel doručil zabalený **bot** s manifestem ještě z roku 2024.

Každý žije v jiném builderu, ships v jiném SKU, a — donedávna — byl
spravovaný přes jiný blade v jiném admin portálu. Snaha odpovědět na otázku
*"kolik máme vlastně AI agentů a kdo je vlastní?"* znamenala proklikávat
Teams Admin Center, Power Platform admin centrum, SharePoint admin centrum
a CSV, které vám manuálně vyexportoval tenant admin.

**Copilot Agent &amp; app Package Management API** (preview) tohle řeší
*na datové úrovni*. Nesjednocuje uživatelské zážitky — sjednocuje
**inventář**. Každý balíček, který je v Microsoft 365 Copilotu možné
nainstalovat, povolit nebo přiřadit, se objeví v jedné stránkované kolekci,
se stejným tvarem, stejnými governance poli a stejnou filtr gramatikou.

A to je přesně ta jediná ingredience, která chyběla, aby se daly dělat
seriózní **agent governance** workflows.

---

## API na jedné obrazovce

**Base URL:** `https://graph.microsoft.com/beta/copilot/admin/catalog/packages`

**Scope:** `CopilotPackages.Read.All` (delegated — viz gotcha níže)

**Method:** `GET` (list a detail), `PATCH` (block / unblock), `DELETE` (odstranění)

### List endpoint

```
GET /beta/copilot/admin/catalog/packages
    ?$filter=supportedHosts/any(h:h eq 'Copilot')
    &amp;$top=100
```

Vrací standardní OData v4 stránkovanou kolekci: `value[]` + `@odata.nextLink`,
dokud nevyčerpáte set. Žádný `$count`, žádný `$expand`, ale `$filter` je
překvapivě silný.

### Detail endpoint

```
GET /beta/copilot/admin/catalog/packages/{id}
```

Vrací všechno, co je v list payloadu, **plus**:

- `availableTo` / `deployedTo` enumy (viz níže — tohle je governance zlato)
- `installAssignedUsers` / `installAssignedGroups` — *kdo to může instalovat*
- `enabledAssignedUsers` / `enabledAssignedGroups` — *kdo to právě má zapnuté*
- `elementDefinitions[]` — manifesty pro každou součást (deklarativní agent,
  bot, action…) včetně **AAD App ID**, manifest ID a u deklarativních agentů
  i inline `instructions` a bloku `capabilities`

List endpoint stačí na dashboard. Detail endpoint je to, co potřebujete na
opravdovou lifecycle práci (najít vlastníka orphaned agentů, zablokovat
balíček podle ID, ověřit verzi manifestu uvnitř).

### Mutating endpointy

```
PATCH  /beta/copilot/admin/catalog/packages/{id}    { "isBlocked": true }
DELETE /beta/copilot/admin/catalog/packages/{id}
```

Tohle jsou páky, které admini opravdu chtějí: zastavit chybujícího agenta,
aby ho nikdo nemohl nainstalovat, nebo balíček úplně odstranit z tenantu.
(Smazání *balíčku* neodstraní podkladovou Bot resource ani AAD App
Registration — jen ho vyhodí z Copilot katalogu.)

---

## Gotcha, na kterou vás nikdo neupozorní: **pouze delegated**

Přečtete si API reference, zkopírujete si svůj client-credentials boilerplate
z poslední Graph automatizace, vložíte si **application** oprávnění
`CopilotPackages.Read.All`… a dostanete plochý HTTP 403 s tělem, které
neřekne nic užitečného.

**Copilot package endpointy nepřijímají app-only tokeny.** Vyžadují
**uživatelský** token, a ten uživatel musí být Copilot Admin (nebo
Global / Cloud App Admin). Tým potvrdil, že je to v preview záměrné — API
povrch je tvarovaný pro *lidské* adminy, kteří spouštějí governance
workflows, ne pro unattended service principaly.

Praktický důsledek: cokoli na tomhle stavíte, potřebuje sign-in. V našem
inventory toolu používáme **MSAL device-code flow** se serializovaným
on-disk token cache, takže je to jedno přihlášení na stroj a všechny další
spuštění jsou tichá. Relevantní kód žije v
`agentsreports/auth.py` — zhruba 120 řádků, většinou
proto, že jsme chtěli hezké progress logy v polling smyčce.

Když nemůžete použít device-code (CI pipeline, headless box), fallback je:

```bash
az login --scope https://graph.microsoft.com/CopilotPackages.Read.All \
         --allow-no-subscriptions
az account get-access-token \
    --scope https://graph.microsoft.com/CopilotPackages.Read.All \
    --query accessToken -o tsv > .token
```

…a načíst token z `$COPILOT_TOKEN` nebo `.token`. Tokeny platí hodinu, takže
je to v pohodě pro ad-hoc analýzu, ne pro dlouhoběžící službu.

---

## Schema, dekódované

Každý package object má zhruba tenhle tvar:

```jsonc
{
  "id": "guid",                 // package ID (to pro /packages/{id})
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
  "type": "shared|lob|sideloaded",   // původ / lifecycle bucket
  "supportedHosts":  ["Copilot", "Teams", "Outlook"],
  "elementTypes":   ["DeclarativeCopilots", "Bots", "AgentSkills"],
  "supportedBuilders": ["Copilot Studio", "Agent Builder", "Agents Toolkit",
                        "SharePoint", "Foundry", "Unspecified"],
  "availableTo": "everyone|specific|admin|nobody",
  "deployedTo":  "everyone|specific|admin|nobody",
  "ownerId":     "guid|null"
}
```

Pár z těchhle polí si zaslouží bližší pohled.

### `type` — lifecycle původ

- `shared` — **publikováno** přes oficiální builder pipelines (Copilot Studio
  publish, Agent Builder publish, Foundry agent registration). Počítá se jako
  first-class tenant content.
- `lob` — **line-of-business**: privátně nahráno do tenantu, typicky balíček
  postavený v Teams Toolkitu a nahraný přes Teams Admin Center.
- `sideloaded` — nainstalováno přímo koncovým uživatelem přes "Upload custom
  app", bez promote do tenant inventáře. **Sideloadované balíčky jsou cesta,
  kudy se do tenantu dostávají shadow agenti** — obcházejí admin review.

V našem reálném tenantu jsme našli `shared:212`, `lob:31`, `sideloaded:15`.
Patnáct sideloads — každý z nich potřebuje prohlédnout a buď povýšit,
zablokovat, nebo smazat.

### `elementTypes` — co je vlastně uvnitř

Jeden balíček může nést víc **element definitions** najednou. Zajímavé jsou:

- **`DeclarativeCopilots`** — YAML-ish manifest agenta s instructions,
  conversation starters, capabilities (Graph Connectors, code interpreter,
  image generator) a `actions[]` (OpenAPI / Power Platform action pluginy).
  Tohle jsou *ti* Microsoft 365 Copilot agenti v běžném smyslu.
- **`AgentMetadatas`** — boti postavení v Copilot Studiu, jak deklarativní,
  tak custom-engine. Metadata zabaluje Power Platform environment ID a
  published bot ID.
- **`Bots`** — klasické Azure Bot Service boty, včetně custom-engine agentů
  (CEAs), kteří mluví na Container App nebo Functions backend. Většina
  starých "Teams botů" landuje sem.
- **`AgentSkills`** — znovupoužitelné definice LLM toolů uložené jako
  Markdown front-matter v SharePointu, vystavované agentům on-demand. Nové,
  málo zdokumentované a hodně zajímavé (viz screenshot).
- **`AgenticUserTemplates`** — předpřipravené konverzační šablony pro
  agentic Copilot UI (jeden v našem tenantu — zjevně někdo experimentoval).

Rozdělení v našem reálném tenantu: 94 AgentMetadatas, 86 DeclarativeCopilots,
74 Bots, 3 AgentSkills, 1 AgenticUserTemplate.

### `availableTo` vs `deployedTo` — kombo, které řídí váš audit

Tohle je **jednoznačně nejvíc poddokumentovaná** dvojice polí v celém API
a zároveň to, co nejvíc rozhoduje o governance.

| Pole          | Význam                                                     |
| ------------- | ---------------------------------------------------------- |
| `availableTo` | *kdo má dovoleno tenhle balíček nainstalovat* (assignment) |
| `deployedTo`  | *kdo ho právě má zapnutý/pinnutý* (active footprint)       |

Obě berou stejný enum: `everyone`, `specific`, `admin`, `nobody`.

Kombinace a co znamenají v praxi:

- `available=everyone, deployed=everyone` — **broadly active**. Skutečný
  tenant-wide agent. Treat as production. (Většina Microsoft content.)
- `available=everyone, deployed=specific` — **dostupné, ale adopce vázne**.
  Marketing to rolloutoval, ale org to nebere. Adoption signal.
- `available=specific, deployed=specific` — **scoped pilot**. Zdravý stav
  pro pre-GA agenta. Ověřte, že přiřazené skupiny pořád existují.
- `available=admin, deployed=admin` — **admin-only**. Často legacy boti,
  které nikdy nikdo nepublikoval širšímu publiku. **Kandidáti na smazání.**
- `available=nobody, deployed=nobody` — orphaned. Pravděpodobně omylem
  publikované a nedotažené. Bezpečně mazatelné.
- `available=nobody, deployed=specific` — **danger**. Balíček už nikdo
  nemůže nainstalovat, ale uživatelé, kteří ho mají, ho pořád mají. Tohle
  je stav přesně po emergency blocku a *před* úklidem. Pokud tohle vidíte
  a sami jste nic neblokovali, něco se pokazilo.

V dashboardu na [a365graph.ai-news.cz](https://a365graph.ai-news.cz/) tohle
kombo vizualizujeme na **Governance** záložce — je to zdaleka nejužitečnější
jednotlivá věc, na kterou se dá koukat.

### `ownerId` — a co znamená `null`

`ownerId` je *AAD ObjectId uživatele uvedeného jako vlastník balíčku*.
V dobře vedeném tenantu má každý vlastní balíček jednoho. V reálném tenantu
mnoho z nich nemá — a **to je bug, ne feature**:

- U deklarativních agentů vytvořených z **chatu ("New Copilot agent")**
  builder zapíše agenta do vašeho osobního sandboxu a `ownerId` jste *vy*.
- U **shared / publikovaných** deklarativních agentů z Copilot Studia
  `ownerId` přichází z vlastníka Power Platform environmentu (často
  service account).
- U **Microsoft content** a SharePoint AgentSkills je `ownerId` rutinně
  null. Nic se s tím nedá dělat.
- U **third-party botů sideloadovaných přes Teams Toolkit** je `ownerId`
  cokoli, co vývojář nastavil při publish. Často null. **Tohle jsou agenti,
  které nemáte komu zavolat, když se něco rozbije.**

V našem tenantu: **29 orphan agentů** s `ownerId=null`. Většina jsou
Microsoft samples a defaultní "Your developer name", které někdo zapomněl
změnit. Ale dva byly business-critical boti, jejichž původní vývojář
firmu opustil.

Governance záložka surfacuje přesně tenhle seznam:

![Governance záložka: blocked agents a orphan agents]({{ "/assets/images/governance.png" | relative_url }})

### `manifestId` &amp; `appId` — křížové reference

Každý balíček nese dvě další ID, která dovolují cross-reference s *reálnými*
podkladovými resources:

- `manifestId` → GUID **Teams app manifestu** (matchuje `id`
  v `manifest.json`). Použijte ho na vyhledání balíčku v Teams Admin Centeru.
- `appId` → **AAD App Registration** Application ID. Použijte ho na nalezení
  bot's Azure Bot resource, audit App-permission grantů, najití
  Conditional Access assignments atd.

Pokud auditujete security posture, **`appId` je to, co potřebujete**. Řádek
v Graph package vám řekne, že existuje Copilot custom agent; odpovídající
App Registration vám řekne, jaká dostal permissions, jestli byl secret
rotován a jestli se s ním v poslední době někdo přihlásil.

---

## Co jsme našli v reálném tenantu: 258 packages

Pustili jsme inventory script proti reálnému tenantu (data jsou ve veřejném
dashboardu plně sanitizovaná — jména, GUIDs, emaily jsou všechno nahrazené
placeholdery). Headline čísla:

| Metrika                       | Počet | Poznámka                                          |
| ----------------------------- | ----: | ------------------------------------------------- |
| Vlastní packages              |   258 | lob + shared + sideloaded                         |
| Blocked                       |     1 | `isBlocked=true` — admin hard-blocked             |
| Orphan agenti                 |    29 | bez `ownerId` — bez konkrétního maintainera       |
| Outdated manifest             |    53 | `manifestVersion` ≤ 1.22 nebo `devPreview`        |
| Copilot-hosted agenti         |   146 | žijí v Microsoft 365 Copilot hostu                |
| Postaveno v Copilot Studiu    |   144 |                                                   |
| Postaveno v Agent Builderu    |    61 |                                                   |
| Postaveno v Teams Toolkitu    |    16 |                                                   |
| Sideloaded (shadow IT?)       |    15 |                                                   |
| Bots (Azure Bot Service)      |    74 | klasické CEAs + legacy Teams boti                 |
| Deklarativní copiloti         |    86 | moderní Copilot agent povrch                      |

Pár takeaways, které překvapily i tým, který tenant provozuje:

1. **Mix platforem je opravdu fragmentovaný.** Copilot Studio dominuje
   počtem (144), ale *není* většinový — 114 packages bylo postaveno někde
   jinde, s pěti dalšími buildery v aktivním používání. Jakýkoli governance
   příběh, který říkáte o "našich Copilot Studio agentech", míjí 44 %
   povrchu.

2. **53 packages se zastaralým manifestem je TLS-cert-style časovaná bomba.**
   Teams app manifest schema je teď na v1.22 a **deklarativní agenti, kteří
   neobsahují `webApplicationInfo`, prostě renderují s prázdnými input
   fieldy** — Adaptive Cards mají `Input.Text` / `Input.ChoiceSet` strippnuté
   client-side. (Narazili jsme na přesně tenhle problém při stavbě jednoho
   z in-house agentů — viz
   poznámky o `webApplicationInfo`
   pro forenzní rozbor.)

3. **Sideloads nutně nejsou špatné — ale jsou neviditelné pro většinu
   admin toolingu.** Patnáct `sideloaded` packages byli uživatelé, kteří
   si nahráli vlastní `.zip` přes `atk install`. Dva byly prototypy, které
   se potichu staly důležité. Žádný z nich nebyl v Teams Admin Center
   katalogu. Bez tohohle API bychom o jejich existenci nevěděli.

4. **"Cowork Skills" povrch je reálný a roste.** `AgentSkills` element type
   je úplně nový — tři v našem tenantu, všechny vystavené přes inline-skills
   systém, který čte Markdown front-matter ze známého SharePoint folderu.
   Tohle jsou **agent-tool-mesh** stavební bloky, které Microsoft rolloutuje
   pod Copilot Studio "skills" hlavičkou.

![Cowork Skills: znovupoužitelné LLM tool definice v SharePointu]({{ "/assets/images/cowork-skills.png" | relative_url }})

---

## Dashboard: [a365graph.ai-news.cz](https://a365graph.ai-news.cz/)

JSON inventář jsme přetavili do plně statického dashboardu. **Bez backendu,
bez databáze, bez auth** — API extract je předem sanitizovaný, zapečený
v build time do pár JSON souborů a servírovaný z Azure Static Web Apps
na Free tieru za managed TLS certem.

Je to existence proof: celý governance povrch se vejde do 234 kB JS
bundle (72 kB gzipped) a renderuje pod jednu vteřinu.

Co tam je:

### 1. Dashboard — single-screen overview

![Souhrnný dashboard pro 258 vlastních Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

Čtyři KPI dlaždice (Custom packages · Blocked · Orphan agents · Outdated
manifest) a čtyři breakdown bary (Builder · Element Type · Source Type ·
Host). Tohle by si měl Copilot admin pouštět každé pondělí ráno.

### 2. Agents — prohledávatelný inventář

![Inventory tabulka: každý vlastní Copilot package v tenantu]({{ "/assets/images/agents.png" | relative_url }})

258 řádků v TanStack-Table s full-text vyhledáváním napříč name / publisher
/ ID, plus tři faceted filtry (Type · Builder · Element). Klik na kterýkoli
řádek vás přenese do per-package detailu s plným JSONem, element
definitions a seznamy přiřazených uživatelů / skupin.

### 3. Cowork Skills — AgentSkills povrch

![AgentSkills: znovupoužitelné LLM skill definice napříč agenty]({{ "/assets/images/cowork-skills.png" | relative_url }})

Karty pro každý `AgentSkills` element v tenantu. Každá ukazuje SharePoint
folder, v němž žije, site ID, na kterém je hostovaná, `embedded` flag a
plný popis (který slouží i jako LLM tool description). Tohle jsou stavební
kostky další vlny Copilot agentů, které budou skládané za běhu.

### 4. Governance — action queue

![Governance: blocked + orphan agenti vyžadující pozornost admina]({{ "/assets/images/governance.png" | relative_url }})

Dvě tabulky, na které admin může reagovat přímo: **blocked** (už hard-blocked
— je to záměr?) a **orphan** (bez vlastníka — přiřaďte ho, nebo balíček
odstraňte). Nic složitého, ale přesně tahle obrazovka mění JSON dump
v workflow.

### Z čeho je to postavené

- **Frontend:** Vite + React + TypeScript + Tailwind + TanStack Table.
- **Datová pipeline:** Python script
  (`scripts/build_swa_data.py`), který zavolá Graph
  endpoint, projde každý JSON node, scrubne GUIDs na tokeny stylu
  `guid-0001`, redactne emaily, aplikuje malý `sanitize.json`
  customer-name nahrazení (např. reálná jména → "Contoso" / "Fabrikam")
  a vyplivne čtyři JSON soubory do `public/data/`. Nula PII opouští
  build step.
- **Hosting:** Azure Static Web Apps (Free) v West Europe, managed TLS,
  custom doména `a365graph.ai-news.cz`. Cena: 0 EUR / měsíc.
- **Deploy:** jeden shell script
  (`swa/deploy.sh`) — Python rebuild, Vite build,
  fetch deployment tokenu, push přes `@azure/static-web-apps-cli`.
  ~30 s end-to-end.

Celé to je tak malé, že je to skoro trapné. Pointa je: **jakmile máte to
API, zbytek je triviální.** Většina engineering effortu šla do sanitizace,
ne do vizualizace.

---

## Věci, které jsme zkusili a které si můžete vzít

Pět malých patternů, na kterých jsme skončili a které se ukázaly jako
high-value:

### 1. `$filter` používejte server-side, ne client-side

OData filtry se na tomhle endpointu skládají dobře a server už stránkuje.
Netahejte všech 258 packages a nefiltrujte v Pythonu — pushněte filter.

```python
# "agenti na Microsoft 365 Copilotu modifikovaní za posledních 30 dní"
$filter = (
    "supportedHosts/any(h:h eq 'Copilot') and "
    "elementTypes/any(e:e eq 'DeclarativeCopilots') and "
    "lastModifiedDateTime gt 2026-04-24T00:00:00Z"
)
```

### 2. Cachujte device-code refresh token na disk

MSAL ships `SerializableTokenCache`. Persistujte ji do souboru pod
`.token_cache.bin` (gitignored), aby další běhy byly tiché. Přidali jsme
malou kontrolu na `.token` soubor jako fallback pro headless boxy.

### 3. Retry na 424 *i* 429

Graph package endpoint někdy throttluje s **HTTP 424** místo standardní
429. Tohle je beta-API kuriozita. Náš `GraphClient` bere obojí jako
retryable s exponential backoff:

```python
retryable = {424, 429}
if resp.status_code in retryable or resp.status_code >= 500:
    wait = retry_after or min(2 ** attempt, 30)
    time.sleep(wait); continue
```

### 4. Pro governance vždycky `--details`

List endpoint je rychlý, ale nenese `installAssignedUsers` / `groups` ani
`elementDefinitions`. Pro reálnou governance chcete per-package detail.
Rozpočet ~2 ms na volání na warm cache, ~50 ms cold — pro 258 packages
to je cca 15 s end-to-end.

### 5. Sanitizace předtím, než publikujete

Každý reálný inventář má rozházená jména zákazníků, jména zaměstnanců,
emailové adresy, SharePoint site ID a Azure subscription GUIDs. Pokud
dashboard publikujete veřejně (jak my), dělejte redakci v **build time**,
zapište sanitizační pravidla do JSON konfigu, který se dá review, a běžte
`grep` proti publikovanému artefaktu jako unit test:

```bash
# failne build, pokud propadne nějaký reálný GUID
! grep -REo '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' \
    swa/public/data/ | grep -v 'guid-[0-9]'
```

---

## Co API ještě chybí

Endpoint je výborný. Ne dokonalý.

- **Žádný `$count`** na kolekci. Nevíte, kolik stránek máte, dokud je
  nevyčerpáte. Není to showstopper, ale komplikuje to progress UI.
- **Žádný `$expand` pro `elementDefinitions`.** Vždycky uděláte N+1 na jejich
  získání. Pro 258 packages to je 258 extra HTTP volání. Cacheable, ale
  otravné.
- **Žádné webhooks / change feed.** Není `delta()` na kolekci — nemůžete
  se přihlásit k odběru "řekni mi, když se publikuje nový agent". Musíte
  pollovat a diffovat. My to děláme nightly do malého JSON snapshotu
  v blob storage; *ten* diff pak řídí Slack alert.
- **`devPreview` manifest agenti prosakují.** Staří deklarativní agenti
  postavení proti dev-preview manifest schématu se objevují vedle aktuálních
  packages bez vizuální značky. Jsou objevitelní přes `manifestVersion`
  uvnitř `elementDefinitions[].manifest`, ale parse semveru je na vás.
- **App-only je pořád zablokované.** Už probráno. Den, kdy tohle dorazí,
  se nightly inventory stane triviální v service principalu.

Nic z toho není blocker — je to typ pollish, který preview API sbírá
cestou k GA.

---

## Co bychom chtěli dál od Microsoftu

Tři přání, v pořadí priority:

1. **App-only support** s těsně scopnutým permission
   (`CopilotPackages.Read.All` už funguje delegated; vydejte app-only
   variantu). Dnes každá nightly-inventory automatizace potřebuje
   human-bound účet.
2. **`delta()` collection** na packages endpointu, stejný tvar jako
   `users/delta` a `groups/delta`. Tohle jedno přidání by nahradilo 80 %
   polling-and-diffing infrastruktury, kterou se lidi chystají postavit.
3. **`assignedUsers/$count`** sub-resource na každém balíčku. Chceme
   odpovídat na "který agent je *opravdu* používaný 10k+ uživateli?" bez
   enumerace každého assignmentu.

---

## Vyzkoušejte si to

Kód, který pohání živé demo, je dost malý na to, abyste ho přečetli
za odpoledne:

- Inventory tool — API klient + report generátor:
  `agentsreports/`
  (Python · MSAL device-code · OData paging · governance metriky)
- Statický dashboard — Vite/React/TS, bez backendu:
  `swa/`
  (TanStack Table · Tailwind · React Router · 234 kB total bundle)
- Build &amp; sanitize pipeline:
  `scripts/build_swa_data.py`
  (GUID scrubber · email redactor · ordered name replacements)

Repo je strukturované tak, abyste mohli:

1. Nastavit `TENANT_ID` a `CLIENT_ID` v `.env`.
2. `python -m agentsreports.inventory --details` — vyplivne
   `out/packages.json`, `out/packages.csv`, `out/report.md`.
3. *(Volitelně)* `bash swa/deploy.sh` na publikaci vlastního dashboardu.

Pokud se chcete **napřed podívat, jak to vypadá**, sanitizované demo žije
trvale na **[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**.

---

*Komentáře &amp; opravy vítány. Tahle stránka je generovaná z
`docs/_posts/` přes GitHub Pages — otevřete PR,
pokud chcete něco opravit.*
