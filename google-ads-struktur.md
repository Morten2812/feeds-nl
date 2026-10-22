# Google Ads – struktur og dataflow

**Webshop:** [naturlaboratoriet.dk](https://naturlaboratoriet.dk)

Overblik over hvordan webshop, produktfeed, Google Merchant Center, Google Tag
Manager, cookie-samtykke og Google Ads hænger sammen.

## Konto-oversigt

| Konto | ID | Status |
|---|---|---|
| Google Ads | 742-775-1648 | Aktiv |
| Google Ads | 368-030-3695 | Lukket — tjek om noget (konverteringer, historik) skal overføres herfra |
| Google Merchant Center | 5755493074 ("Naturlaboratoriet") | Aktiv — 834 produkter (832 godkendt, 2 begrænset), 9 notifikationer, 2.21K klik/28 dage (-9,3%). Aktiveret 30. mar. 2026 |
| Google Merchant Center | 5755394053 ("Naturlaboratoriet") | ⚠️ Duplikat — aktiveret 21. aug. 2026, kun tilgængelig via "Virksomhedsadministrator" (dukker ikke op i den normale kontoskifter). Formentlig tom/ubrugt. **Lav prioritet — udskudt for nu** |

> Noter fra Merchant Center-screenshot: fanen **"Kræver handling"** under
> Produkter har ting der bør tjekkes (12 notifikationer) — det er typisk
> afviste produkter eller advarsler (manglende GTIN, forkert pris-format
> osv.), som betyder de produkter ikke vises i Shopping-annoncer før rettet.

## Hvor finder du det i Google Ads-fladen

Google Ads, Google Analytics og Google Merchant Center hænger sammen via
**"Linked accounts"** — men Google Tag Manager er et separat værktøj, ikke
noget du "linker" på samme måde:

1. **Merchant Center + Google Analytics (GA4):**
   Skruenøgle-ikon ("Tools and settings") øverst til højre → kolonnen
   **"Setup"** → **"Linked accounts"**. Her ses status og et "Details"-link
   for hver.
2. **Google Tag Manager:**
   Ligger på et helt andet site: **tagmanager.google.com** — ikke inde i
   Google Ads. Google Ads "ser" kun GTM indirekte, via
   konverteringsopsætningen: Skruenøgle → **"Measurement"** → **"Conversions"**.
   Der kan du se om en konverteringshandling er sat op med Google Tag direkte
   på sitet, eller via GTM.

> **Opdateret, bekræftet via "Dataadministration" i jeres konto:**
> - **Google Analytics (GA4):** ✅ linket (1 link)
> - **Google Merchant Center:** ✅ linket — men **2 link**, dvs. to
>   Merchant Center-konti er koblet til denne Google Ads-konto. Bør
>   undersøges hvorfor (fx en gammel/ubrugt konto ved siden af den aktive
>   5755493074).
> - **Google-tag:** findes ("Untitled tag" — ikke omdøbt). ✅ Bekræftet
>   (2026-10-22): installeres **direkte på sitet af Google for WooCommerce**
>   via gtag, ikke via GTM.

> **Opdateret, bekræftet via Google Tag Manager:**
> - **GTM-container:** ✅ `GTM-NBTT238`, installeret på `www.naturlaboratoriet.dk`
> - **Google-tags i kontoen:**
>   - GA4: `G-N4QYM1SC3Q`
>   - Unified Google tag: `GT-55B3V2VK`
>   - Google Ads conversion (= "Untitled tag"): `AW-472880718`
> - ✅ Container kører som **live-version** (ingen preview/auth-miljø sat) —
>   bekræftet via GTM4WP-pluginnets indstillinger i wp-admin
> - ✅ GTM4WP har indbygget **WooCommerce-integration** (egen undermenu til
>   ecommerce-events)
> - ⚠️ Stadig ikke bekræftet: om tags rent faktisk **fyrer** (trigger-setup
>   inde i containeren), og om **Consent Mode v2** er konfigureret — pluginnet
>   har en dedikeret menu **"Consent mode & consent tools"**, som er næste
>   sted at tjekke

## Overblik (diagram)

Det store billede: hvilke spor data følger fra webshoppen til Google Ads.
Hvordan pluginnerne arbejder sammen inde i WordPress, står i afsnittet
[Samspil mellem plugins](#samspil-mellem-plugins-wordpress).

```mermaid
flowchart TD
    subgraph Shop["Webshop – naturlaboratoriet.dk (WordPress + WooCommerce)"]
        W["Produktdata<br/>titel, pris, lager, billede"]
        CMP["Complianz + WP Consent API<br/>✅ cookie-banner, Consent Mode v2 virker"]
        GFW["Google for WooCommerce<br/>✅ Ads-tag AW-472880718 + feed"]
        GTM4WP["GTM4WP<br/>✅ indsætter GTM-container"]
    end

    subgraph Google["Google"]
        MC["Merchant Center 5755493074<br/>✅ 834 produkter, 832 godkendt"]
        MC2["Merchant Center #2 5755394053<br/>⚠️ duplikat, formentlig tom"]
        GTM["GTM-container GTM-NBTT238<br/>⚠️ GA4-fyring ikke testet"]
        GA4["Google Analytics 4<br/>G-N4QYM1SC3Q"]
        ADS["Google Ads 742-775-1648<br/>✅ aktiv, 10 kampagner på pause"]
        ADSOLD["Google Ads 368-030-3695<br/>🔒 lukket"]
    end

    W --> GFW
    CMP -- "samtykke" --> GFW
    CMP -- "samtykke" --> GTM4WP

    GFW -- "① produktfeed" --> MC
    GFW -- "② køb + remarketing (gtag)" --> ADS
    GTM4WP --> GTM
    GTM -- "③ adfærd + e-commerce" --> GA4

    MC -- "kontolink" --> ADS
    GA4 -- "kontolink" --> ADS
    MC2 -. "linket, bør fjernes" .-> ADS
    ADSOLD -. "historik?" .-> ADS
```

**De tre spor:**
1. **Produktfeed:** Google for WooCommerce → Merchant Center → Google Ads (Shopping/PMax)
2. **Konverteringer:** Google for WooCommerce → Google Ads, direkte via gtag
3. **Analyse:** GTM4WP → GTM-container → GA4 → Google Ads

> **Signaturforklaring:** ✅ bekræftet · ⚠️ ikke bekræftet endnu ·
> 🔒 lukket/inaktiv konto.

## Samspil mellem plugins (WordPress)

Fire plugins på sitet arbejder sammen om samtykke og sporing. De taler ikke
direkte med hinanden. De mødes to steder: i **WP Consent API**, som er et fælles
register over samtykke inde i WordPress, og i **dataLayer**, som er en fælles
"postkasse" i den besøgendes browser, hvor alle scripts lægger beskeder til Google.

```mermaid
flowchart LR
    V(["👤 Besøgende"])

    subgraph WP["WordPress – naturlaboratoriet.dk"]
        WOO["<b>WooCommerce</b><br/>produkter, kurv, ordrer"]
        CMPLZ["<b>Complianz</b><br/>cookie-banner"]
        WPCA["<b>WP Consent API</b><br/>fælles samtykke-register"]
        GFW["<b>Google for WooCommerce</b><br/>Ads-tag + produktfeed"]
        GTM4WP["<b>GTM4WP</b><br/>indsætter GTM-container"]
    end

    DL[("<b>dataLayer</b><br/>fælles postkasse<br/>i browseren")]

    subgraph G["Google"]
        GTMC["GTM-container<br/>GTM-NBTT238"]
        GA4["Google Analytics 4<br/>G-N4QYM1SC3Q"]
        ADS["Google Ads<br/>AW-472880718"]
        MC["Merchant Center<br/>5755493074"]
    end

    V -- "Godkend / Afvis" --> CMPLZ
    CMPLZ -- "1 · registrerer samtykke" --> WPCA
    CMPLZ -- "cmplz_event_*" --> DL
    WPCA -- "2 · læser samtykke" --> GFW
    GFW -- "3 · consent default + update" --> DL

    WOO -- "produktdata" --> GFW
    GFW -- "produktfeed" --> MC
    GFW -- "gtag: køb, remarketing<br/>(direkte, uden om GTM)" --> ADS

    WOO -- "kurv/ordre-data" --> GTM4WP
    GTM4WP -- "e-commerce events<br/>view_item_list, purchase …" --> DL
    GTM4WP -- "indlæser" --> GTMC
    DL --> GTMC
    GTMC -- "GA4-tags<br/>(⚠️ fyring ikke testet)" --> GA4

    MC -. "kontolink" .-> ADS
    GA4 -. "kontolink" .-> ADS
```

**Hvem gør hvad**

| Plugin | Opgave | Sender til Google |
|---|---|---|
| **WooCommerce** | Selve webshoppen: produkter, kurv, ordrer | Intet direkte. Leverer data til de to Google-plugins |
| **Complianz** | Viser cookie-banneret og gemmer valget | Intet direkte. Melder samtykket til WP Consent API og lægger `cmplz_event_*` i dataLayer |
| **WP Consent API** | Fælles samtykke-register, som andre plugins kan læse | Intet. Det er bindeleddet mellem Complianz og Google for WooCommerce |
| **Google for WooCommerce** | Google Ads-tag, købskonvertering og produktfeed | Ads (`AW-472880718`) direkte via gtag, feed til Merchant Center, Consent Mode-signaler |
| **GTM4WP** | Indsætter GTM-containeren og e-commerce-events | GA4 via GTM-containeren |

**Tommelfingerregler**
- **Google Ads håndteres af Google for WooCommerce.** Opret ikke Ads-konverterings-tags i GTM, for så tælles hvert køb to gange.
- **GA4 håndteres af GTM** (via GTM4WP). Lad ikke Google for WooCommerce, Rank Math eller Site Kit også indsætte GA4.
- **Consent Mode kræver alle tre led:** Complianz → WP Consent API → Google for WooCommerce. Deaktiveres WP Consent API, kommer der ingen `update`, og Google Ads ser alle som "afvist".

## Navigationsstier — hvor jeg fandt det

| Hvad | Sti |
|---|---|
| Google Ads-konti (aktiv + lukket) | Google Ads → profilbillede (øverst til højre) → kontoliste |
| Merchant Center produkter/tal (5755493074) | merchants.google.com → **Produkter** (venstre menu) → Alle produkter |
| Merchant Center-duplikat fundet (5755394053) | Google Ads → **Værktøjer** (skruenøgle, venstre sidebar) → **Dataadministration** → Google Merchant Center → "Administrer og tilknyt" → fane "Linket (2)" |
| Bekræftelse af duplikat via Business Manager | Google Ads → Merchant Center-række → "Foretag administration i Virksomhedsadministrator" → **Virksomhedsadministrator** → Apps og tjenester |
| GA4-link + Google-tag ("Untitled tag" / AW-472880718) | Google Ads → **Værktøjer** → **Dataadministration** |
| GTM container-ID (GTM-NBTT238) | tagmanager.google.com → fanen **Konti** |
| Google-tags (G-N4QYM1SC3Q, GT-55B3V2VK, AW-472880718) | tagmanager.google.com → fanen **Google-tags** |
| Platform WordPress + WooCommerce | wp-admin → venstre menu (WooCommerce synlig) |
| GTM4WP-plugin fundet | wp-admin → **Plugins** → Installed plugins → søg "GTM" |
| GTM-container bekræftet live + placering | wp-admin → Plugins → GTM4WP → **Settings** → "Google Tag Manager container" |
| *(mangler)* Consent Mode-status | wp-admin → Plugins → GTM4WP → Settings → **"Consent mode & consent tools"** |
| *(mangler)* Feed-plugin | wp-admin → **Plugins** → søg "feed" |

## Komponenterne

### 1. Webshop
Kilden til al data. Produktinfo (titel, beskrivelse, pris, lagerstatus,
billede, GTIN/EAN osv.) ligger her og skal holdes opdateret — det er det
Merchant Center senere henter.

### 2. Produktfeed
Et XML- eller CSV-dokument som webshoppen (eller et feed-værktøj) genererer
automatisk, i det format Google kræver. Opdateres typisk dagligt eller
hyppigere, så priser/lager altid er korrekte. Dette repo (`feeds-nl`) er
formentlig stedet feed-logikken/konfigurationen for det hollandske marked
ligger.

### 3. Google Merchant Center (GMC)
Henter produktfeed'en på et fast interval, **godkender eller afviser**
produkter (ud fra Googles regler — manglende GTIN, forkert pris-format,
billedkrav osv.), og stiller de godkendte produkter til rådighed for Google
Ads.

- **Linkes** til en Google Ads-konto (konto-til-konto link, ikke en tag).
- Fejl/advarsler på produkter tjekkes her — ikke i Google Ads.

### 4. Google Tag Manager (GTM)
Et "container"-script der lægges på hele websitet. I stedet for at indsætte
Google Ads/Analytics-kode direkte i sitet, styres alle tags (målinger,
remarketing, konverteringssporing) centralt fra GTM's interface.

Typiske tags i en webshop-opsætning:
- **Google Ads conversion tracking** — måler køb/leads der kommer fra annoncer
- **Google Ads remarketing tag** — bygger lister af besøgende til retargeting
- **GA4-konfiguration** — generel adfærdsmåling (sidevisninger, add-to-cart, køb)
- **Enhanced conversions** — sender hashed kundedata (email/telefon) med for
  bedre match-rate

### 5. Cookie-samtykke (CMP) + Consent Mode v2
Et cookie-banner (Cookiebot, OneTrust, CookieYes, Complianz, e.l.) spørger
besøgende om samtykke, **før** marketing-/statistik-cookies må sættes — et
krav under GDPR, og dermed relevant for et `.nl`-marked.

- CMP'en sender samtykke-status til GTM via **Google Consent Mode v2**
  (signalerne `ad_storage`, `ad_user_data`, `ad_personalization`,
  `analytics_storage`).
- GTM bruger de signaler til at afgøre **om** og **hvordan** tags fyrer:
  - Samtykke givet → fulde tags (cookies sættes, fuld sporing)
  - Samtykke afvist → Google kan stadig modtage **cookieless pings** (hvis
    Consent Mode er korrekt opsat), som bruges til modellering — men ingen
    cookies sættes, og ingen personlige data gemmes.
- **Uden korrekt Consent Mode-opsætning** mister Google Ads konverteringsdata
  fra besøgende der afviser cookies, hvilket forringer budoptimering
  (særligt i Performance Max, som er meget data-afhængig).

### 6. Google Ads
Her bygges kampagnerne:
- **Shopping / Performance Max**: bruger produkterne fra Merchant Center
  direkte — ingen separat annoncetekst/billede, Google sammensætter ud fra
  feed-data.
- **Search/Display/Video**: bruger konverteringsdata fra GTM-tagget til at
  optimere bud, og remarketing-lister til target.

## Dataflow, trin for trin

1. Produktdata opdateres i webshoppen.
2. Feed-filen regenereres (automatisk, scheduled).
3. Merchant Center henter feed'en, validerer og godkender produkter.
4. Godkendte produkter er tilgængelige for Google Ads via kontolinket.
5. Besøgende lander på sitet → cookie-banner viser sig → samtykke
   registreres.
6. GTM modtager samtykke-status → fyrer tags i overensstemmelse hermed.
7. Events (sidevisning, add-to-cart, køb) sendes til GA4 og Google Ads.
8. Google Ads bruger konverteringsdata + produktdata til at optimere bud og
   vise annoncer (Shopping/PMax) til relevante brugere.
9. Remarketing-lister (fra GTM-tagget) bruges til at vise annoncer til
   tidligere besøgende.

## Status og næste skridt (konkret for naturlaboratoriet.dk)

1. ✅ **Produktfeed** → Merchant Center — virker (834 produkter, 832 godkendt).
2. ✅ **Merchant Center linket til Google Ads** (742-775-1648).
3. ✅ **GTM-container installeret** (`GTM-NBTT238` via GTM4WP-plugin, live).
4. ✅ **Google-tags oprettet** (GA4, GT-unified, Google Ads `AW-472880718`).
5. ✅ **CMP/cookie-banner installeret** — **Complianz** sat op via guiden
   (GDPR/EU-region, Accepter/Afvis/Gem præferencer-knapper, cookie- og
   scriptblokering aktiveret, Google Fonts selv-hostes, ingen TCF-framework
   nødvendigt). Complianz fandt og aktiverede automatisk integration til
   **GTM4WP**.
6. ✅ **Consent Mode v2 virker** (testet 2026-10-22 i browserkonsollen)
   - Complianz-guide: statistik = "Google Tag Manager affyrer dette script",
     reklame-scripts = Ja, ingen TCF-framework
   - `consent default` (alt `denied`, EU/EØS + GB/CH, `wait_for_update: 500`)
     sendes af **Google for WooCommerce** — ikke af Complianz
   - `consent update` (`granted`) manglede, indtil **WP Consent API** blev
     installeret (manuelt via Simply.com-filhåndtering, da plugin-installation
     fra wp-admin fejler). Nu: Complianz → WP Consent API → Google for
     WooCommerce → `gtag('consent','update')`
   - ✅ **Afvis** testet: `update` med alt `denied` (korrekt)
   - ⚠️ **Google Ads-tagget (`AW-472880718`) indsættes direkte af Google for
     WooCommerce**, ikke via GTM → opret **ikke** Ads-konverterings-tag i GTM
     (dobbelttælling)
7. ⬜ **Bekræft at tags rent faktisk fyrer** — tjek triggers inde i
   GTM-containeren + brug Preview-mode, både med og uden samtykke givet.
8. ⬜ **Gennemgå de 10 eksisterende kampagner** (alle sat på pause pt.) —
   afklar om pausen er bevidst, og genaktivér/opdatér når tracking (trin 5-7)
   er på plads. Først herefter er det værd at overveje at bygge en helt ny
   kampagne.

> Rækkefølgen er vigtig: **cookie-banner og Consent Mode bør være på plads
> før kampagner genaktiveres** — ellers mangler Google Ads konverteringsdata
> fra besøgende der afviser cookies, hvilket giver dårligere budoptimering
> fra dag ét.

## Vigtige begreber

| Begreb | Betydning |
|---|---|
| **Feed** | Den fil Merchant Center henter produktdata fra |
| **Linked account** | Forbindelsen mellem Merchant Center og Google Ads (ikke en tag) |
| **Container (GTM)** | Samlingen af alle tags/triggers for ét website |
| **Consent Mode v2** | Googles standard for at signalere samtykke-status til Google-tags |
| **Enhanced conversions** | Ekstra matchdata (hashed email/telefon) for bedre konverteringssporing |
| **Performance Max (PMax)** | Kampagnetype der bruger produktfeed + AI til automatisk at vise annoncer på tværs af Googles flader |

## Åbne spørgsmål / ting at bekræfte

> Denne fil er lavet generisk, da jeg ikke kender jeres konkrete opsætning.
> Tjek/udfyld følgende for at gøre den præcis for jer. Skærmbilleder af
> GTM-containeret, cookie-bannerets indstillinger og Merchant Center er en
> hurtig måde at få dem udfyldt på.

- [x] **Platform:** ✅ **WordPress + WooCommerce** (bekræftet via wp-admin).
- [x] **GTM-installation:** ✅ sat op via pluginnet **"GTM4WP – A Google Tag
      Manager (GTM) plugin for WordPress"** (Thomas Geiger, v2.0.6), som
      lægger containeret `GTM-NBTT238` ind på sitet.
- [x] Feed: ✅ **Google for WooCommerce**, bekræftet i pluginnets Settings
      (Merchant Center 5755493074 + Google Ads 742-775-1648 forbundet via
      info@naturlaboratoriet.dk og Jetpack). 236 klik på gratis
      produktvisninger i okt. 2026
- [ ] Google for WooCommerce-dashboard viser fejl *"Unexpected token '<' …
      is not valid JSON"*: REST API-kald returnerer HTML. Formentlig samme
      serverproblem som plugin-installationen. Tjek Værktøjer → Webstedstilstand
- [ ] Google Ads: **erklæring om EU-politiske annoncer** mangler på kampagner.
      Skal udfyldes ("indeholder ikke politiske annoncer") før genaktivering
- [x] CMP: **Complianz** + **WP Consent API**
- [x] Consent Mode v2: ✅ virker via Google for WooCommerce (se trin 6)
- [ ] Plugin-installation fra wp-admin fejler generelt — wp-admin-filer har
      blandede datoer (mulig halv WP-opdatering). Tjek med Simply.com
- [ ] 37 plugin-opdateringer + 13 inaktive plugins — opdatér/ryd op (backup først)
- [ ] Bruges Enhanced Conversions / server-side tagging?
- [ ] Hvor mange Merchant Center/Google Ads-konti er der (fx ét per land/marked)?
