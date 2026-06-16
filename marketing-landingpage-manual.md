# Marketingový manuál – Schlieger & značky

> Pracovní referenční dokument pro marketing a tvorbu leadů. Stav k **14. 6. 2026**. Žije a doplňuje se.
> Vlastník: Markéta Chrbolková (spolumajitelka Schlieger s.r.o.). IČO Schlieger s.r.o.: **28787803**.

Tenhle manuál drží na jednom místě: jak vznikají a kam tečou leady, jak se publikují landing pages, jak se měří (GTM / GA4 / BigQuery), jak se staví týdenní obsah a jak se pracuje se značkou. Slouží jako onboarding pro tým i jako rychlá reference.

---

## 1. Jak tečou leady (pipeline)

Tok jednoho leadu od návštěvníka po systém:

**Návštěvník → Landing page (Netlify) → formulář → Lead Gateway (ingest) → systém → výsledek (result)**

Souběžně běží měření: na LP je nasazený GTM/GA4 tag, který posílá eventy do GA4 a odtud (přes BigQuery export) do reportů a dashboardu.

### 1.1 Lead Gateway

Leady z formulářů se neposílají přímo do CRM, ale přes **lead gateway** – Supabase edge funkci (`make-server-9924f985`, cesta `lead-gateway`). Gateway lead přijme, vrátí hned potvrzení a zpracuje ho asynchronně.

**Ingest endpointy:**

| Prostředí | URL |
|---|---|
| **PROD** | `https://cwertkgbliffhzrynrxt.supabase.co/functions/v1/make-server-9924f985/lead-gateway/ingest` |
| **DEV** | `https://vdgvdjdjsdbncudzzibx.supabase.co/functions/v1/make-server-9924f985/lead-gateway/ingest` |

**Protokol:**

- Metoda: **POST**
- Hlavička: **`X-Gateway-Key`** (gateway klíč pro dané prostředí – uložen v `lead-gateway.env` vedle manuálu; drž v env proměnné, nikdy ne natvrdo ve veřejném kódu LP)
- Odpověď: **`202 Accepted`** ihned (`{ success, accepted, gateway_log_id }`), zpracování běží asynchronně.
- Výsledek zpracování: **GET** na `…/lead-gateway/result/<gateway_log_id>` → `status` (`completed`/`failed`), `action`, `match_field`, `campaign_id`, `campaign_lead_id`, `errors`.

**Struktura payloadu** (OVĚŘENO na DEV 14. 6. 2026 – takto lead projde, `status: completed`). Pole **nejsou plochá** – vše je zanořené pod `eventDetails`, kontakt pod `eventDetails.userData`:

```bash
curl -X POST "$GATEWAY_DEV_INGEST_URL" \
  -H "Content-Type: application/json" \
  -H "X-Gateway-Key: $GATEWAY_DEV_KEY" \
  -d '{
    "eventDetails": {
      "tenantId": "fve_nove",            // camelCase, uvnitř eventDetails (viz §1.3)
      "adId": "",
      "note": "Lead z LP kalkulacka-nzu",
      "userData": {
        "name": "Jan Novák",
        "email": "jan@example.cz",       // povinný aspoň email NEBO telefon
        "phone": "+420700000000",
        "zip": "11000",
        "city": "Praha"
      }
    }
  }'
# → 202 Accepted: { "success": true, "accepted": true, "gateway_log_id": "..." }

# Stav zpracování (dosadit gateway_log_id z odpovědi):
curl "$GATEWAY_DEV_RESULT_URL/<gateway_log_id>" -H "X-Gateway-Key: $GATEWAY_DEV_KEY"
# → { "status": "completed", "action": "person_matched", "match_field": "phone",
#     "campaign_id": "...", "campaign_lead_id": "...", "errors": [] }
```

**Validační pravidla (z testů 14. 6. 2026):**

- `tenantId` musí být **camelCase a uvnitř `eventDetails`**. Ploché `tenant_id` gateway ignoruje → `failed: "No active config for tenantId: "` (prázdná hodnota = pole se nepřečetlo).
- `userData` musí mít **aspoň `email` nebo `phone`**, jinak `failed: "Missing both email and phone – at least one is required"`.
- Osoba se páruje přes email/telefon → `action: person_matched` / `person_created`; `match_field` říká podle čeho.
- Kam patří UTM (`utm_content`) pro spárování s kreativou – zatím neověřeno, doplnit (pravděpodobně do `userData` nebo `eventDetails`).

### 1.2 Prostředí PROD vs DEV

- **DEV** (`vdgvdjdjsdbncudzzibx`) – pro testování formulářů a nových LP. Sem posílej, dokud LP není ověřená.
- **PROD** (`cwertkgbliffhzrynrxt`) – ostrý provoz. Přepnout endpoint až po otestování na DEV.
- Liší se jen subdoména projektu, zbytek cesty je stejný. Gateway klíč je pro každé prostředí jiný.

### 1.3 tenant_id – směrování leadu na kampaň

Každý lead nese identifikátor tenanta (v DB sloupec `tenant_id`, v JSON payloadu **`eventDetails.tenantId`** – camelCase, viz §1.1), který určí, do které kampaně / divize se zařadí. Aktivní CZ tenanti („nové leady"):

| tenant_id | Kampaň | Divize | Aktivní | Přiřazeno |
|---|---|---|---|---|
| `fve_nove` | FVE – nové leady | cz | ano | NULL |
| `tc_nove` | TČ – nové leady | cz | ano | NULL |
| `zatepleni_nove` | Zateplení – nové leady | cz | ano | NULL |
| `nzu_schlieger_nove` | NZÚ Schlieger – nové leady | cz | ano | NULL |
| `nzu_ciperka_nove` | NZÚ Čiperka – nové leady | cz | ano | NULL |

Orientační napojení na značku/LP *(odvozeno z názvu – ověřit)*: `fve_nove` → Schlieger FVE · `tc_nove` → Warmteo (tepelná čerpadla) · `zatepleni_nove` → Čiperka (fasáda/zateplení) · `nzu_schlieger_nove` → Schlieger NZÚ (LP `schlieger-nzu`) · `nzu_ciperka_nove` → Čiperka NZÚ (LP `stavbyciperka-kalkulacka`).

> **Pozor:** sloupec `přiřazeno` je u všech **NULL** = lead zatím není směrován na konkrétního vlastníka/obchodníka. Doplnit (viz §9).
> Na LP posílej `eventDetails.tenantId` odpovídající produktu té stránky – jinak lead spadne do špatné (nebo žádné) kampaně.

### 1.4 Postup otestování leadu (DEV)

1. Vezmi DEV endpoint + `GATEWAY_DEV_KEY` z `lead-gateway.env`.
2. Pošli **POST** na `…/lead-gateway/ingest` se strukturou z §1.1 (`eventDetails.tenantId` + `userData` s emailem nebo telefonem).
3. Z odpovědi (`202`) si vezmi **`gateway_log_id`**.
4. **GET** `…/lead-gateway/result/<gateway_log_id>` → ověř `status: completed` a `action` / `campaign_id`.
5. Když `failed`, koukni na `errors` (nejčastěji špatné zanoření `tenantId` nebo chybějící kontakt).
6. Po ověření přepni na **PROD** endpoint + PROD klíč.

> **Ověřeno 14. 6. 2026:** testovací lead na `fve_nove` přes DEV → `status: completed`, `action: person_matched` (spárováno přes telefon), vytvořen `campaign_lead_id`. Pozn.: test matchnul existující osobu podle telefonu – pro čisté testy použij unikátní testovací email/telefon.

---

## 2. Landing pages a publikace (Netlify)

LP se **publikují přes Netlify**. Vznikají jako statické HTML (standard `lp-standard.skill`, tracking boilerplate `lp-tracking-boilerplate.html`).

**Netlify tým:** `marketa-chrbolkova` („DomiDomi Webs", Free).

**Živé LP:**

| LP | Site name | Live URL | Cílová doména |
|---|---|---|---|
| Kalkulačka nároku NZÚ (Schlieger) | `schlieger-nzu` | https://schlieger-nzu.netlify.app | `nzu.schlieger.cz` |
| Kalkulačka ceny fasády (Čiperka) | `stavbyciperka-kalkulacka` | https://stavbyciperka-kalkulacka.netlify.app | `kalkulacka.stavbyciperka.cz` |

Site ID: schlieger-nzu = `cbfdec6e-4e4a-4faf-8a2b-d63f26e1d023`; stavbyciperka-kalkulacka = `4ca23c77-61e7-40e7-9060-423f4e2a9fcc`.

**Deploy (mechanika):**

- Buildí se do `outputs/build/…` (index.html + `netlify.toml` s `publish="."` a hlavičkou `X-Robots-Tag: noindex` u testovacích LP).
- Nasazení přes Netlify MCP `deploy-site`: vrátí jednorázový `npx @netlify/mcp@latest --site-id <id> --proxy-path <TOKEN> --no-wait` (token je dočasný → vždy si vyžádej nový). Globální `npm install` nejde (práva) → stačí `npx` cache.
- Pozor na 45s limit shellu → `--no-wait` + polling přes `get-deploy-for-site`.

**Domény / DNS:**

- `nzu.schlieger.cz`: nutný **explicitní CNAME** `nzu` → `schlieger-nzu.netlify.app`, protože `schlieger.cz` má **wildcard `*.schlieger.cz`** (jinak spadne na catch-all a SSL selže). Po přidání domény v Netlify vyřeší Let's Encrypt SSL.
- `kalkulacka.stavbyciperka.cz`: CNAME `kalkulacka` → `stavbyciperka-kalkulacka.netlify.app`. `stavbyciperka.cz` wildcard nemá → čistší.

**Stav / TODO LP:**

- Formuláře na LP zatím **leady neodesílají** → napojit na lead gateway (§1) nebo elegantně na Netlify Forms.
- Čiperka LP: doplnit reálné IČO/tel/e-mail (teď placeholdery) a ověřit sazby Kč/m² (orientačně: zateplení 1 300–2 000, omítka 450–800, oboje 1 500–2 200).

---

## 3. Měření: GTM, GA4, BigQuery

### 3.1 Registr měřicích ID (Google Tag Manager)

U každého webu/značky: **G-** = GA4 measurement ID, **GT-** = Google tag ID. Pozor: `GT-` ≠ klasický `GTM-` kontejner (např. warmteo.de má navíc GTM kontejner `GTM-MP8GHJ5G`).

| Web / značka | GA4 (G-) | Google tag (GT-) |
|---|---|---|
| www.schlieger.cz – GA4 | `G-CTSNDPMX5P` | (jen GA4) |
| www.schlieger.de | `G-L5143J37M1` | `GT-PLVQNQP` |
| www.schlieger.sk | `G-3JEVHZ2QQ0` | `GT-MBNDPSN` |
| warmteo.cz | `G-HSXHPCXLJC` | `GT-TW5DMV9Q` |
| warmteo.de | `G-6NWWQFN5DM` | `GT-NFRRR58B` |
| stavbyciperka.cz (Čiperka) | `G-TKFQG48WB8` | `GT-PH39HG3M` |
| domidomi.cz | `G-R3LY8K4H37` | `GT-WV8WTLQC` |
| dotaceproseniory.cz | `G-C1N3NKHQDX` | `GT-TNF9PZR` |
| www.dostupnedotace.cz | `G-2EKF0T3PQT` | `GT-PHXQK65` |
| Schlieger vzdělávání | `G-6KBVEL40B3` | `GT-TBBKB7V` |
| Schlieger vzdělávání SK | `G-Q5TH0RX1WE` | `GT-PZZKP7J` |
| Schlieger Intranet | `G-0ZXGV6X058` | `GT-MR57QPB` |
| Průvodce úvodním školením | `G-LH67W4570Q` | `GT-W6B3J357` |
| (Untitled tag) | `G-RPTHLR8TG3` | `GT-TQV8XTB` |
| (Untitled tag) | `G-9L5ZE0FR6Z` | — |

> Dvě **„Untitled tag"** v TM stojí za pojmenování, ať se neztratí přehled.

### 3.2 GA4 properties → web (pro čtení dat z BigQuery)

Číselné GA4 property ID (jiné než měřicí `G-` výše) slouží k dotazování dat v BigQuery (datasety `analytics_<property_id>`):

| Property ID | Web |
|---|---|
| 294425925 | schlieger.cz (největší traffic) |
| 402029952 | schlieger.de |
| 478471920 | stavbyciperka.cz |
| 468413301 | warmteo.de |
| 468415061 | warmteo.cz |
| 491546531 | dostupnedotace.cz (nízký traffic) |
| 491525374 | lumixo.cz (nízký traffic) |

Neaktivní / zastavený export: 381359999, 381352072, 402529355.

### 3.3 BigQuery

- GCP projekt: **`schlieger`** („Schlieger Data Lakehouse"), lokace EU. Čte se přes BigQuery MCP (`execute_sql_readonly`, `project_id = schlieger`).
- Nativní GA4 export: datasety `analytics_<property_id>`, tabulky `events_YYYYMMDD` (+1 den zpoždění). Navíc `raw_ga4` a `raw_kbc_ga4` (Keboola).
- **Marketingový model** (dataset `dm_marketing`) – views nad `raw_kbc_*` a `raw_google_ads`:
  - `dim_ad_hierarchy` – source / ad_id / ad_name / adset / campaign. **Granularita:** Facebook/Sklik/Bing až na úroveň reklamy, **Google jen na úroveň kampaně**.
  - `fact_mkt_ad_performance_stats` – spend / impressions / clicks (měny normalizované na CZK).
  - `mkt_reporting_2_0` (+`_pivot`) – vrcholový KPI report: plán vs. skutečnost leadů/spend/CPL po produktu/marketu/týdnu.
  - Leady: `cdm.dim_lead` → `vw_first_lead_source` (first-touch atribuce přes e-mail/mobil). **Kreativní granularita leadů jen z GA4 `generate_lead` (utm_content)**, ne z CRM (tam je `lead_source` hrubý).

### 3.4 Měřicí smyčka a známá díra ⚠️

Měřicí ID v TM ≠ záruka, že se měří konverze. **Aktuální stav:**

- Standardní form eventy z boilerplate (`view_form` / `begin_form` / `generate_lead`) **nechodí ani na jedné property** (0 za 30 dní).
- Výjimka: **warmteo.de** měří formuláře, ale pod nestandardními názvy (`form_step1`, `form_step2`, `form_sent`, `LeadCapture`, `cta_button_click`…).
- GA4 → BigQuery export pro hlavní domény **funguje** (schlieger.cz, stavbyciperka.cz…). Díra tedy NENÍ v exportu, ale v **chybějících GTM triggerech na standardní názvy eventů** + nesouladu pojmenování.
- Dashboard ukazuje pomlčky, protože tabulka `mkt_daily_metrics` (Supabase `marketing-ops`, id `cfdbtosjqhejmpufxwqi`) je prázdná – čeká na backfill z BQ.
- **domidomi.cz nemá živý GA4 export** v žádné aktivní property → zůstane 0, dokud se nenapojí.

**Co dořešit (jednorázově, ručně – přes MCP to nejde):** v GTM každé domény nastavit GA4 Configuration tag + triggery na `view_form / begin_form / generate_lead` (nebo sjednotit boilerplate na existující názvy), pak backfill `mkt_daily_metrics` z BigQuery.

---

## 4. Značky a weby (přehled)

| Značka | Doména(y) | Téma | Akcentní barva |
|---|---|---|---|
| **Schlieger** | schlieger.cz / .de / .sk | Fotovoltaika (FVE), dotace NZÚ | červená `#DA000F` |
| **Čiperka** (stavbyciperka) | stavbyciperka.cz | Zateplení, fasády | oranžová `#F08A1D` |
| **Warmteo** | warmteo.cz / .de | Topení, tepelná čerpadla | zlatá/gold |
| **DomiDomi** | domidomi.cz | Nábor / HR | — |
| Dostupné dotace / Dotace pro seniory | dostupnedotace.cz, dotaceproseniory.cz | Dotační rozcestníky | — |

> Pozn.: **Warmteo = topení, ne fotovoltaika** (častá záměna). Schlieger = FVE.

---

## 5. Značka a vizuály (brand)

### 5.1 Barvy a písmo

- **Schlieger červená: `#DA000F`** (RGB 218,0,15) – oficiální z loga; černá `#000000`. *(V některých starších obsahových šablonách se objevuje `#E2001A` – sjednotit na `#DA000F`.)*
- **Čiperka oranžová: `#F08A1D`**, doplňková navy `#10243E`.
- Písmo: **Poppins** (Bold / SemiBold / Medium / Regular), záloha Open Sans.

### 5.2 Brand assety (Google Drive)

Oficiální assety jsou na Drivu ve složce **LOGOS** (`127jh1y8qiQigfbyD4FzZ6tb2DDo6Tf4g`), podsložky po značkách (SCHLIEGER, CIPERKA_LOGO, WARMTEO, DOMIDOMI, NZU LOGO…). Schlieger logo PDF (vektor) ve složce `1Q0mN4GxybnpbVGaeGiFcBCPvCLp7G5j8`; Poppins TTF na Drivu („Poppins").

> **Logo NIKDY negenerovat AI** – AI generátory ho vždy překreslí špatně. Správné logo se musí vložit jako reálný soubor (kompozice z PDF/PNG).

### 5.3 Fotky a vizuály

- Pro **hero / klíčové konverzní vizuály** preferuj **reálné fotky realizací**, ne AI – u rozhodnutí o penězích (dotace) reálná fotka budí víc důvěry; AI „stock" vzhled konverzi podkopává.
- Použitelné hero fotky na Drivu: složka **„FVE Upravene"** (`1XWYhdN_TQSSVz_zD_1LhXmmf5Yo-Nn6p`); **DJI_0235** = rodinný dům s panely = nejlepší hero (používá ho LP `kalkulacka-nzu`).
- Pozor: „reference WARMTEO" na Drivu jsou **topení**, ne FVE.
- AI (Higgsfield) používej jen jako doplněk/koncept, ne na hlavní hero. Vždy nabídni výběr + krátké zdůvodnění z hlediska konverze.

---

## 6. Týdenní obsahové balíčky

Týdenní obsah se připravuje jako **produkční balíček** pro tým (Markéta → podřízeným): master Word + Excel publikační kalendář + složka „Obsahy" se samostatným `.docx` ke každému obsahu.

- **Kam ukládat:** workspace „Marketing Schlieger" → složka pojmenovaná dle týdne (např. `Podklady obsah 15-21.6.2026`).
- **Brandy v balíčku:** Schlieger (FVE/NZÚ, CTA „kalkulačka nároku"), Čiperka (zateplení/fasáda, formulář „cena do 1 minuty"). DomiDomi (HR) dle potřeby.
- **Čísla NZÚ** se přebírají ze schváleného plánu – nevymýšlet.

### 6.1 UTM konvence

- `utm_campaign` – jednotná pro celý týden (např. `nzu_2026-06`)
- `utm_content` – `MMDD_brand_format_varianta` (např. `0615_schlieger_carousel_b`)
- `utm_medium` – `social` / `paid_social`
- Měření per varianta (A/B/C) → výkon vidět v GA4 i CRM.

### 6.2 Kód kreativy (párování spend × leady)

Aby šel lead/kreativa spárovat s útratou, musí být **stejný „kód kreativy" v UTM i v názvu reklamy**. Doporučený tvar:

```
{brand}_{produkt}_{publikum}_{format}_{YYYYMM}_v{NN}
```

Vyhodnocení kreativy = join spend (`dim_ad_hierarchy` × `fact_mkt_ad_performance_stats`) × leady (GA4 `generate_lead` by `utm_content`). Detaily v `konvence-nazvoslovi-kreativ.md`.

---

## 7. Reputace a GEO (PR strategie)

Cíl: aby se Schlieger pozitivně zobrazoval v Google AI přehledech i u asistentů (ChatGPT/Gemini/Perplexity) a bránil se vnímané negativní kampani.

> **Důležitý kontext:** část nejsilnějších negativ nepochází od anonymní konkurence, ale z **autoritativních zdrojů** (vyloučení z České fotovoltaické asociace, investigativní série HN, recenzní portály). Soud navíc označil dosavadní obranu (předběžná opatření + exekuce vůči kritikům) za **šikanu** → Streisandův efekt. Tyhle zdroje **nejdou SEO-potlačit** a potlačování prokazatelně škodí.

**Doporučený postup – jen legitimní ORM:**

- Pravé ověřené recenze (Google / Heureka), transparentní řešení reklamací.
- Nezávislá bezpečnostní certifikace montáží, proaktivní earned media.
- AI-čitelný obsah: FAQ, čísla, tabulky.
- **Nikdy:** falešné/placené recenze ani gag orders / exekuce vůči kritikům či novinářům.

Plán: `Schlieger_PR_a_AI_strategie.docx` ve složce Marketing Schlieger.

---

## 8. Rychlý odkazovník

**Spuštění nové kampaně / LP – checklist:**

1. Postav LP (`lp-standard.skill`) + vlož tracking boilerplate a správné měřicí ID dle §3.1.
2. Napoj formulář na **DEV** gateway se správným `tenant_id` (§1.3), otestuj (POST → 202 → GET result).
3. Po ověření přepni na **PROD** gateway.
4. Deploy na Netlify (`deploy-site`, nový token), nastav CNAME + doménu (pozor na wildcard u schlieger.cz).
5. Zkontroluj, že GTM má triggery na `view_form / begin_form / generate_lead` (jinak se konverze nezměří – viz §3.4).
6. UTM dle konvence (§6.1), kód kreativy v UTM i názvu reklamy (§6.2).
7. Po spuštění ověř data v GA4 / BigQuery; backfill `mkt_daily_metrics` pokud dashboard ukazuje pomlčky.

**Klíčové identifikátory:**

- IČO Schlieger s.r.o.: `28787803`
- BigQuery projekt: `schlieger` · marketingový dataset: `dm_marketing`
- Supabase marketing-ops: `cfdbtosjqhejmpufxwqi` · gateway PROD `cwertkgbliffhzrynrxt` / DEV `vdgvdjdjsdbncudzzibx`
- Netlify tým: `marketa-chrbolkova`

**Související soubory ve workspace:**

- `zadani-LP-tracking.md`, `lp-tracking-boilerplate.html` – tracking standard
- `postup-automatizace-LP.md` – automatizace tvorby LP
- `konvence-nazvoslovi-kreativ.md` – pojmenování kreativ
- `Schlieger_PR_a_AI_strategie.docx` – reputace/GEO

---

## 9. Otevřené úkoly (TODO)

- [ ] Napojit formuláře LP (schlieger-nzu, Čiperka) na lead gateway → reálné odesílání leadů (se správným `tenant_id`).
- [ ] Přiřadit tenanty (`přiřazeno` = NULL u všech 5) → nasměrovat leady na vlastníky/obchodníky.
- [ ] V GTM doplnit triggery na standardní form eventy (všechny hlavní domény) nebo sjednotit boilerplate na názvy, které warmteo.de už měří.
- [ ] Backfill `mkt_daily_metrics` z BigQuery → dashboard přestane ukazovat pomlčky.
- [ ] Napojit GA4 export pro domidomi.cz (dnes 0).
- [ ] Pojmenovat dvě „Untitled tag" v Tag Manageru.
- [ ] Sjednotit Schlieger červenou na `#DA000F` napříč šablonami.
- [ ] Čiperka LP: doplnit IČO/tel/e-mail + ověřit sazby Kč/m².
```
