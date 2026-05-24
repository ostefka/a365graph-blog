---
layout: default
title: "Jak vypadá tenant s 258 Copilot agenty — průvodce M365 Copilot Package Management API"
description: "Delší, názorový článek o Microsoft 365 Copilot Package Management API — co vrací, co se v dokumentaci nedočtete, a co jsme zjistili při běhu proti reálnému tenantu s 258 vlastními agenty."
permalink: /article-copilot-package-management-api-cs
---

<p style="font-size: 0.95em; margin: 0 0 1.5em 0;">
  <a href="{{ '/article-copilot-package-management-api' | relative_url }}">&larr; English</a>
  &nbsp;|&nbsp;
  <strong>Česky</strong>
  &nbsp;|&nbsp;
  <a href="{{ '/' | relative_url }}">← Zpět na seznam článků</a>
</p>

# Jak vypadá tenant s 258 Copilot agenty

> **TL;DR** — Microsoft uvolnil v Graphu endpoint
> (`/beta/copilot/admin/catalog/packages`), díky kterému Copilot admin
> *konečně* vidí všechny vlastní agenty, deklarativní copiloty, boty
> i Office add-iny ve svém tenantu — napříč Copilot Studiem, Agent Builderem,
> Teams Toolkitem, SharePointem, AI Foundry i sideloadovanými balíčky —
> v jednom stránkovaném JSON feedu. Spustili jsme ho proti reálnému tenantu,
> našli 258 packages a obraz "agent sprawlu", který z dat vystoupil, je
> zajímavější než kterákoli prezentace.
>
> Jako pomůcku — abyste viděli, co API skutečně vrací, aniž byste museli
> napsat jediný řádek kódu — jsme výstup sanitizovali a dali za read-only
> dashboard na **[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**.
> *Tenhle článek je o Graph API. Dashboard je jen obohacení, ukázka toho, co
> v JSON řádcích reálně sedí.*

> **Předpoklady.** Tohle je admin povrch Microsoft 365 Copilotu, ale licenčně
> sedí za *samostatným* produktem. Abyste ho mohli použít, potřebujete:
>
> - Tenant musí mít licenci na **[Microsoft Agent 365](https://www.microsoft.com/microsoft-agent-365)** —
>   podle [oficiální dokumentace](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/api/admin-settings/package/overview)
>   *přístup k Package Management API vyžaduje licenci Microsoft Agent 365.*
>   Samotná licence M365 Copilot **nestačí**.
> - Volající uživatel musí mít roli **Copilot Admin** (nebo Global Admin /
>   Cloud App Admin).
> - Token musí být **delegated**; app-only zatím není podporovaný
>   (viz problém níže).
>
> Oficiální dokumentace: **[Package Management API overview (preview)](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/api/admin-settings/package/overview)**.

![Souhrnný dashboard s přehledem 258 vlastních Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

## Proč to má smysl řešit

Každý Microsoft 365 tenant, který za posledních 18 měsíců zapnul Copilot,
skončil se stejným typem problému:

- Marketing si v **Copilot Studiu** postavil "Brand Voice Coach".
- Vývojář publikoval "Knowledge Q&amp;A" přes **Agent Builder** rovnou z chatu.
- IT Ops nahrál custom-engine agenta pro ServiceNow přes **Teams Toolkit** jako sideload.
- SharePoint admin vytvořil **AgentSkill** pro "shrň tento dokument".
- Někdo z týmu zapnul tři Microsoftem dodávané deklarativní agenty ze storu.
- Dodavatel doručil zabalený **bot** s manifestem ještě z roku 2024.

Každý žije v jiném builderu, dodává se v jiném SKU a — donedávna — se
spravoval přes jiný blade v jiném admin portálu. Když jste se chtěli zeptat
*"kolik vlastně máme AI agentů a kdo je vlastní?"*, museli jste proklikat
Teams Admin Center, Power Platform admin centrum, SharePoint admin centrum
a CSV, které vám manuálně vyexportoval tenant admin.

**Copilot Agent &amp; app Package Management API** (preview) tohle řeší
*na datové úrovni*. Nesjednocuje uživatelské prostředí — sjednocuje
**inventář**. Každý balíček, který lze v Microsoft 365 Copilotu nainstalovat,
povolit nebo přiřadit, se objeví v jedné stránkované kolekci se stejnou
strukturou, stejnými governance poli a stejnou gramatikou filtrů.

A přesně tahle jediná chybějící ingredience bránila tomu, aby se daly dělat
seriózní **agent governance** workflows.

## API na jedné obrazovce

**Base URL:** `https://graph.microsoft.com/beta/copilot/admin/catalog/packages`

**Scope:** `CopilotPackages.Read.All` (delegated — viz problém níže)

**Method:** `GET` (list i detail), `PATCH` (block / unblock), `DELETE` (odstranění)

### List endpoint

```
GET /beta/copilot/admin/catalog/packages
    ?$filter=supportedHosts/any(h:h eq 'Copilot')
    &amp;$top=100
```

Vrací standardní OData v4 stránkovanou kolekci: `value[]` + `@odata.nextLink`,
dokud nevyčerpáte celou sadu. Žádný `$count`, žádný `$expand`, ale `$filter`
je překvapivě silný.

### Detail endpoint

```
GET /beta/copilot/admin/catalog/packages/{id}
```

Vrací všechno, co je v list payloadu, **plus**:

- `availableTo` / `deployedTo` enumy (viz níže — tohle je governance zlatý důl)
- `installAssignedUsers` / `installAssignedGroups` — *kdo to může nainstalovat*
- `enabledAssignedUsers` / `enabledAssignedGroups` — *kdo to má právě zapnuté*
- `elementDefinitions[]` — manifesty pro každou součást (deklarativní agent,
  bot, action…) včetně **AAD App ID**, manifest ID a u deklarativních agentů
  i inline `instructions` a bloku `capabilities`

Na dashboard stačí list endpoint. Detail endpoint potřebujete na opravdovou
lifecycle práci: dohledat vlastníka osiřelých agentů, zablokovat balíček
podle ID, ověřit verzi manifestu uvnitř.

### Mutating endpointy

```
PATCH  /beta/copilot/admin/catalog/packages/{id}    { "isBlocked": true }
DELETE /beta/copilot/admin/catalog/packages/{id}
```

Tohle jsou páky, které admini opravdu chtějí: zastavit chybujícího agenta,
aby ho nikdo nemohl nainstalovat, nebo balíček úplně odstranit z tenantu.
(Smazání *balíčku* neodstraní podkladovou Bot resource ani AAD App
Registration — jen ho vyhodí z Copilot katalogu.)

## Problém, na který vás v dokumentaci nikdo neupozorní: jen **delegated**

Přečtete si API reference, zkopírujete si client-credentials boilerplate
z poslední Graph automatizace, vložíte si **application** oprávnění
`CopilotPackages.Read.All`… a dostanete suché HTTP 403 s tělem, které
neřekne nic užitečného.

**Copilot package endpointy nepřijímají app-only tokeny.** Vyžadují
**uživatelský** token a ten uživatel musí být Copilot Admin (nebo Global /
Cloud App Admin). Microsoft potvrdil, že je to v preview záměrné — API
je tvarované pro *lidské* adminy, kteří spouštějí governance workflows,
ne pro neobsluhované service principaly.

Praktický důsledek: cokoli na tomhle stavíte, vyžaduje sign-in. V našem
inventory toolu používáme **MSAL device-code flow** se serializovaným
on-disk token cachem, takže je to jedno přihlášení na stroj a každý další
běh je už tichý. Příslušný kód žije v `agentsreports/auth.py` — zhruba
120 řádků, většinou proto, že jsme chtěli mít hezké progress logy ve
smyčce pro polling.

Pokud nemůžete použít device-code (CI pipeline, headless stroj), záložní
postup vypadá takhle:

```bash
az login --scope https://graph.microsoft.com/CopilotPackages.Read.All \
         --allow-no-subscriptions
az account get-access-token \
    --scope https://graph.microsoft.com/CopilotPackages.Read.All \
    --query accessToken -o tsv > .token
```

…a načíst token z `$COPILOT_TOKEN` nebo `.token`. Tokeny platí hodinu,
takže to stačí na ad-hoc analýzu, ne na dlouho běžící službu.

## Rozluštěné schéma

Každý package object má zhruba tuhle strukturu:

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

Pár polí si zaslouží bližší pohled.

### `type` — lifecycle původ

- `shared` — **publikováno** přes oficiální builder pipelines (Copilot Studio
  publish, Agent Builder publish, Foundry agent registration). Počítá se to
  jako first-class obsah tenantu.
- `lob` — **line-of-business**: privátně nahráno do tenantu, typicky balíček
  postavený v Teams Toolkitu a uploadovaný přes Teams Admin Center.
- `sideloaded` — nainstalováno přímo koncovým uživatelem přes "Upload custom
  app", bez povýšení do tenant inventáře. **Sideloadované balíčky jsou cesta,
  kudy se do tenantu dostávají shadow agenti** — obcházejí admin review.

V našem reálném tenantu jsme našli `shared:212`, `lob:31`, `sideloaded:15`.
Patnáct sideloadů — každý z nich je potřeba projít a buď povýšit, zablokovat,
nebo smazat.

### `elementTypes` — co je vlastně uvnitř

Jeden balíček může nést víc **element definitions** najednou. Zajímavé jsou:

- **`DeclarativeCopilots`** — YAML-ovitý manifest agenta s instructions,
  conversation starters, capabilities (Graph Connectors, code interpreter,
  image generator) a `actions[]` (OpenAPI / Power Platform action pluginy).
  Tohle jsou *ti* Microsoft 365 Copilot agenti v běžném smyslu slova.
- **`AgentMetadatas`** — boti postavení v Copilot Studiu, jak deklarativní,
  tak custom-engine. Metadata zabaluje Power Platform environment ID
  a published bot ID.
- **`Bots`** — klasické Azure Bot Service boty včetně custom-engine agentů
  (CEAs), kteří mluví na Container App nebo Functions backend. Většina starých
  "Teams botů" končí tady.
- **`AgentSkills`** — znovupoužitelné definice LLM toolů uložené jako Markdown
  front-matter v SharePointu, vystavované agentům on-demand. Nové, málo
  zdokumentované a hodně zajímavé (viz screenshot).
- **`AgenticUserTemplates`** — předpřipravené konverzační šablony pro
  agentic Copilot UI (jeden v našem tenantu — zjevně někdo experimentoval).

Rozložení v našem reálném tenantu: 94 AgentMetadatas, 86 DeclarativeCopilots,
74 Bots, 3 AgentSkills, 1 AgenticUserTemplate.

### `availableTo` vs `deployedTo` — dvojice, která řídí váš audit

Tahle dvojice polí je v celém API **jednoznačně nejhůř zdokumentovaná** —
a zároveň ta, na které pro governance nejvíc záleží.

| Pole          | Význam                                                     |
| ------------- | ---------------------------------------------------------- |
| `availableTo` | *kdo má povoleno tenhle balíček nainstalovat* (assignment) |
| `deployedTo`  | *kdo ho má právě zapnutý / připnutý* (aktivní footprint)   |

Obě používají stejný enum: `everyone`, `specific`, `admin`, `nobody`.

Kombinace a co znamenají v praxi:

- `available=everyone, deployed=everyone` — **plošně aktivní**. Reálný
  tenant-wide agent. Berte jako produkci. (Většinou obsah od Microsoftu.)
- `available=everyone, deployed=specific` — **dostupné, ale adopce vázne**.
  Marketing to nasadil, organizace si to nebere. Signál o adopci.
- `available=specific, deployed=specific` — **pilot s vymezeným rozsahem**.
  Zdravý stav pro agenta před GA. Ověřte, že přiřazené skupiny pořád
  existují.
- `available=admin, deployed=admin` — **jen pro adminy**. Často legacy boti,
  které nikdo nikdy nepublikoval širšímu publiku. **Kandidáti na smazání.**
- `available=nobody, deployed=nobody` — osiřelé. Pravděpodobně omylem
  publikované a nedotažené. Bezpečně smazatelné.
- `available=nobody, deployed=specific` — **nebezpečí**. Balíček už nikdo
  nemůže nainstalovat, ale uživatelé, kteří ho mají, ho mají pořád. Tohle
  je stav krátce po nouzovém zablokování, *před* následným úklidem. Pokud
  to vidíte a sami jste nic neblokovali, něco se pokazilo.

V dashboardu na [a365graph.ai-news.cz](https://a365graph.ai-news.cz/) tuhle
dvojici vizualizujeme na záložce **Governance** — je to zdaleka
nejužitečnější jednotlivá věc, na kterou se dá v inventáři dívat.

### `ownerId` — a co znamená `null`

`ownerId` je *AAD ObjectId uživatele uvedeného jako vlastník balíčku*.
V dobře vedeném tenantu má každý vlastní balíček vlastníka. V reálném tenantu
mnoho z nich vlastníka nemá — a **to je bug, ne feature**:

- U deklarativních agentů vytvořených z **chatu ("New Copilot agent")**
  builder zapíše agenta do vašeho osobního sandboxu a `ownerId` jste *vy*.
- U **shared / publikovaných** deklarativních agentů z Copilot Studia je
  `ownerId` převzaté od vlastníka Power Platform environmentu (často
  service account).
- U **obsahu od Microsoftu** a SharePoint AgentSkills bývá `ownerId` rutinně
  null. S tím se nedá nic moc dělat.
- U **third-party botů sideloadovaných přes Teams Toolkit** je `ownerId` to,
  co vývojář nastavil při publishi. Často null. **Tohle jsou agenti, na
  které nemáte komu zavolat, když se něco pokazí.**

V našem tenantu: **29 osiřelých agentů** s `ownerId=null`. Většina jsou
Microsoft samples a defaultní "Your developer name", které někdo zapomněl
změnit. Ale dva byly business-critical boti, jejichž původní vývojář
mezitím odešel z firmy.

Záložka Governance vyplaví přesně tenhle seznam:

![Governance záložka: blocked a orphan agents]({{ "/assets/images/governance.png" | relative_url }})

### `manifestId` &amp; `appId` — křížové reference

Každý balíček si nese ještě dvě ID, která umožňují cross-reference
s *reálnými* podkladovými prostředky:

- `manifestId` → GUID **Teams app manifestu** (odpovídá `id` v `manifest.json`).
  Použijte ho na vyhledání balíčku v Teams Admin Center.
- `appId` → Application ID **AAD App Registration**. Použijte ho na nalezení
  Azure Bot resource, audit App-permission grantů, dohledání Conditional
  Access assignments atd.

Pokud auditujete security posture, **`appId` je to, co potřebujete**. Řádek
v Graph package vám řekne, že nějaký Copilot custom agent existuje;
odpovídající App Registration vám prozradí, jaká dostal oprávnění, jestli
byl secret rotován a jestli se s ním v poslední době někdo přihlásil.

## Co jsme našli v reálném tenantu: 258 packages

Spustili jsme inventory script proti reálnému tenantu (data ve veřejném
dashboardu jsou plně sanitizovaná — jména, GUIDs i e-maily jsou nahrazené
placeholdery). Klíčová čísla:

| Metrika                       | Počet | Poznámka                                          |
| ----------------------------- | ----: | ------------------------------------------------- |
| Vlastní packages              |   258 | lob + shared + sideloaded                         |
| Blocked                       |     1 | `isBlocked=true` — admin to zablokoval natvrdo    |
| Osiřelí agenti                |    29 | bez `ownerId` — bez konkrétního maintainera       |
| Zastaralý manifest            |    53 | `manifestVersion` ≤ 1.22 nebo `devPreview`        |
| Copilot-hosted agenti         |   146 | žijí v Microsoft 365 Copilot hostu                |
| Postavené v Copilot Studiu    |   144 |                                                   |
| Postavené v Agent Builderu    |    61 |                                                   |
| Postavené v Teams Toolkitu    |    16 |                                                   |
| Sideloaded (shadow IT?)       |    15 |                                                   |
| Bots (Azure Bot Service)      |    74 | klasické CEAs + legacy Teams boti                 |
| Deklarativní copiloti         |    86 | moderní povrch Copilot agentů                     |

Pár závěrů, které překvapily i tým, který tenant provozuje:

1. **Skladba platforem je opravdu roztříštěná.** Copilot Studio dominuje
   počtem (144), ale *není* většinový — 114 packages bylo postaveno jinde,
   s pěti dalšími buildery v aktivním používání. Jakýkoli governance příběh
   o "našich Copilot Studio agentech" míjí 44 % povrchu.

2. **53 packages se zastaralým manifestem je časovaná bomba ve stylu
   propadlých TLS certifikátů.** Teams app manifest schéma je teď na v1.22
   a **deklarativní agenti, kteří neobsahují `webApplicationInfo`, prostě
   renderují s prázdnými input fieldy** — z Adaptive Cards se na straně
   klienta vyhazují `Input.Text` / `Input.ChoiceSet`. (Narazili jsme přesně
   na tenhle problém při stavbě jednoho z in-house agentů.)

3. **Sideloady samy o sobě nutně špatné nejsou — jsou ale neviditelné pro
   většinu admin toolingu.** Patnáct `sideloaded` packages byli uživatelé,
   kteří si nahráli vlastní `.zip` přes `atk install`. Dva z nich byly
   prototypy, které se potichu staly důležitými. Žádný z nich nebyl
   v katalogu Teams Admin Center. Bez tohohle API bychom o jejich existenci
   nevěděli.

4. **Povrch "Cowork Skills" je reálný a roste.** `AgentSkills` element type
   je úplně nový — tři v našem tenantu, všechny vystavené přes inline-skills
   systém, který čte Markdown front-matter ze známého SharePoint folderu.
   Tohle jsou stavební kostky **agent-tool-mesh**, který Microsoft postupně
   nasazuje pod hlavičkou "skills" v Copilot Studiu.

![Cowork Skills: znovupoužitelné definice LLM toolů v SharePointu]({{ "/assets/images/cowork-skills.png" | relative_url }})

## Co API přináší, vizualizováno

Abychom celou věc zhmotnili — a abyste nemuseli sami pouštět inventory script
jen kvůli tomu, abyste viděli tvar dat — vzali jsme výstup z jednoho
reálného tenantu, sanitizovali ho a dali za malý read-only dashboard na
**[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**.

Dashboard není pointou článku; **tou je API.** Každá z obrazovek níže je jen
jiný pohled na stejný JSON, který endpoint vrací — je tu proto, aby ukázala,
jaký governance obraz je teď v dosahu jednoho HTTP volání.

### Dashboard — single-screen overview

![Souhrnný dashboard s přehledem 258 vlastních Copilot packages]({{ "/assets/images/dashboard.png" | relative_url }})

Čtyři KPI (Custom packages · Blocked · Orphan agents · Outdated manifest)
a čtyři breakdowny (Builder · Element Type · Source Type · Host) — všechno
odvozené přímo z polí list endpointu. Tohle si Copilot admin teď může
vygenerovat kdykoli proti jakémukoli tenantu.

### Agents — prohledávatelný inventář

![Inventory tabulka: každý vlastní Copilot package v tenantu]({{ "/assets/images/agents.png" | relative_url }})

Každý balíček v tenantu jako jeden řádek, prohledávatelný podle
name / publisher / ID a filtrovatelný podle typu, builderu a elementu.
Za každým řádkem sedí kompletní `/packages/{id}` payload — element
definitions, přiřazení uživatelé, skupiny, verze manifestu. Všechno z API.

### Cowork Skills — povrch AgentSkills

![AgentSkills: znovupoužitelné definice LLM skillů napříč agenty]({{ "/assets/images/cowork-skills.png" | relative_url }})

Samostatný pohled na element type `AgentSkills` — nové SharePointem hostované
definice LLM toolů, které Copilot agenti skládají za běhu. Až do tohohle API
neexistoval admin povrch, který by je vůbec dokázal vyjmenovat.

### Governance — action queue

![Governance: blocked + orphan agenti vyžadující pozornost admina]({{ "/assets/images/governance.png" | relative_url }})

Dva seznamy, které mění JSON dump ve workflow: **blocked** balíčky (řídí se
polem `isBlocked`) a **orphan** agenti (řídí se `ownerId=null`). Obojí padá
z API zadarmo.

## Vzory, které jsme zkusili a stojí za přejmutí

Pět drobných patternů, na kterých jsme skončili a které se ukázaly jako
vysoce hodnotné:

### 1. `$filter` používejte na straně serveru, ne klienta

OData filtry se na tomhle endpointu dobře skládají a server už stránkuje.
Netahejte si všech 258 packages a nefiltrujte v Pythonu — pošlete filter.

```python
# "agenti na Microsoft 365 Copilotu modifikovaní za posledních 30 dní"
$filter = (
    "supportedHosts/any(h:h eq 'Copilot') and "
    "elementTypes/any(e:e eq 'DeclarativeCopilots') and "
    "lastModifiedDateTime gt 2026-04-24T00:00:00Z"
)
```

### 2. Cachujte device-code refresh token na disk

MSAL nabízí `SerializableTokenCache`. Persistujte ho do souboru
`.token_cache.bin` (gitignored), aby další běhy byly tiché. Přidali jsme
malou kontrolu na soubor `.token` jako fallback pro headless stroje.

### 3. Retry na 424 *i* 429

Graph package endpoint někdy škrtí provoz s **HTTP 424** místo standardní
429. Je to kuriozita beta API. Náš `GraphClient` bere obojí jako
retryovatelné s exponential backoff:

```python
retryable = {424, 429}
if resp.status_code in retryable or resp.status_code >= 500:
    wait = retry_after or min(2 ** attempt, 30)
    time.sleep(wait); continue
```

### 4. Pro governance vždy `--details`

List endpoint je rychlý, ale neobsahuje `installAssignedUsers` / `groups`
ani `elementDefinitions`. Pro skutečnou governance chcete detail každého
balíčku. Počítejte ~2 ms na volání při warm cache, ~50 ms při cold — pro
258 packages to vyjde řádově na 15 s end-to-end.

### 5. Sanitizujte ještě před publikací

Každý reálný inventář má rozeseté jména zákazníků, jména zaměstnanců,
e-mailové adresy, SharePoint site IDs a Azure subscription GUIDs. Pokud
dashboard publikujete veřejně (jako my), dělejte redakci v **build time**,
zapište sanitizační pravidla do JSON konfigu, který se dá zreviewovat,
a spusťte `grep` proti publikovanému artefaktu jako unit test:

```bash
# build selže, pokud propadne nějaký reálný GUID
! grep -REo '[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}' \
    swa/public/data/ | grep -v 'guid-[0-9]'
```

## Co API zatím chybí

Endpoint je výborný. Není ale dokonalý.

- **Žádný `$count`** na kolekci. Nevíte, kolik stránek máte, dokud je
  nevyčerpáte. Není to zásadní problém, ale komplikuje to progress UI.
- **Žádný `$expand` pro `elementDefinitions`.** Vždy je dotahujete přes N+1.
  Pro 258 packages to znamená 258 dalších HTTP volání. Dá se cachovat, ale
  je to otravné.
- **Žádné webhooks / change feed.** Na kolekci není `delta()` — nemůžete se
  přihlásit k odběru "řekni mi, když se publikuje nový agent". Musíte
  pollovat a diffovat. My to děláme nightly do malého JSON snapshotu
  v blob storage; *ten* diff potom pohání Slack alert.
- **`devPreview` manifest agenti se prosakují skrz.** Staří deklarativní
  agenti postavení proti dev-preview manifest schématu se objevují vedle
  aktuálních packages bez jakékoli vizuální značky. Najít je můžete přes
  `manifestVersion` uvnitř `elementDefinitions[].manifest`, ale parse semveru
  je na vás.
- **App-only je pořád blokované.** Už zmíněné. Den, kdy dorazí app-only
  varianta, se nightly inventory stane triviální i ze service principalu.

Nic z toho není blocker — je to typ vyladění, které preview API sbírají
cestou ke GA.

## Co bychom chtěli dál od Microsoftu

Tři přání, v pořadí priority:

1. **App-only podpora** s úzce omezeným oprávněním (`CopilotPackages.Read.All`
   už funguje delegated; vydejte app-only variantu). Dnes každá
   nightly-inventory automatizace potřebuje účet svázaný s konkrétním
   uživatelem.
2. **Kolekce `delta()`** na packages endpointu, stejná struktura jako
   `users/delta` a `groups/delta`. Tohle jediné přidání by nahradilo 80 %
   polling-and-diffing infrastruktury, kterou se lidi chystají stavět.
3. **Sub-resource `assignedUsers/$count`** na každém balíčku. Chceme
   odpovídat na "který agent je *opravdu* používaný 10k+ uživateli?" bez
   nutnosti enumerovat každý assignment.

## Vyzkoušejte si to

Jestli máte tenant s licencí na [Microsoft Agent 365](https://www.microsoft.com/microsoft-agent-365)
a účet s rolí Copilot Admin, nejjednodušší end-to-end volání proti tomuhle
API vypadá v podstatě takhle:

```bash
az login --scope https://graph.microsoft.com/CopilotPackages.Read.All \
         --allow-no-subscriptions
TOKEN=$(az account get-access-token \
  --scope https://graph.microsoft.com/CopilotPackages.Read.All \
  --query accessToken -o tsv)

curl -s -H "Authorization: Bearer $TOKEN" \
  "https://graph.microsoft.com/beta/copilot/admin/catalog/packages?\$top=100" \
  | jq '.value | length, .[0]'
```

Pokud na výstupu uvidíte počet a jeden package objekt — gratulujeme, máte
kompletní inventář Copilot agentů svého tenantu v jednom stránkovaném feedu.

Pro o něco pohodlnější obal (MSAL device-code, OData paging, retry na
424/429, generátor Markdown reportu) je Python klient, který jsme použili,
publikovaný jako `agentsreports/` v source repu. Existuje proto, abyste
nemuseli přepisovat stejných 200 řádků.

A pokud se chcete **podívat, co API dává, ještě než napíšete řádek kódu**,
sanitizovaný živý extract trvale žije na
**[a365graph.ai-news.cz](https://a365graph.ai-news.cz/)**.

---

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/article-copilot-package-management-api' | relative_url }}">&larr; English</a>
  &nbsp;|&nbsp;
  <a href="{{ '/' | relative_url }}">← Zpět na seznam článků</a>
</p>
