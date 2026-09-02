# Returnolio weboldal – kivitelezési terv

Készült: 2026-09-02. Tervező: Fable. Kivitelező: Opus.
A terv a `thomasbodri/returnolio-website` repóra vonatkozik (a `thomasbodri/Portfolio`
repo egy üres Next.js starter, nem a Returnolio oldal; lásd 0. pont).

---

## 0. Hol van a kód, és mi a kiindulási állapot

| Mi | Hol |
| --- | --- |
| Marketing oldal (returnolio.com) | `thomasbodri/returnolio-website`, Next 16 App Router, MDX docs + blog |
| Webapp (app.returnolio.com) | `thomasbodri/returnolio-webapp` |
| Deploy | GitHub Actions → rsync → Hetzner VPS, PM2 `returnolio-website` (port 3002), Cloudflare Tunnel |
| Prod útvonal | `/home/claudeuser/website` a VPS-en |
| Feliratkozás-adatok | `/home/claudeuser/website-data/waitlist.jsonl` és `newsletter.jsonl` a VPS-en, repón kívül |

**Opus a `returnolio-website` repóban dolgozzon**, egy feature branchen, PR-rel `main`-re.
A `main`-re push azonnal prod deploy (`.github/workflows/deploy.yml`), ezért minden
kapcsolót külön, apró PR-ben kell átbillenteni. A `pr-check.yml` lefuttatja a
guardrail lintet, az a11y lintet és a buildet; soha nem szabad `next build`-et futtatni
a VPS-en (OOM).

### Kritikus leletek, amik nincsenek a feladatlistán, de a lista pontjait blokkolják

1. **Minden oldal canonical URL-je a főoldalra mutat.** `src/app/layout.tsx:96`
   `alternates: { canonical: siteConfig.url }` globálisan, és sem a blog, sem a docs
   `generateMetadata` nem írja felül. Ha ma kapcsolnánk be az indexelést, a Google a
   docs és blog oldalakat a főoldal duplikátumaként dobná el. Ez az SEO feladat 1. lépése.
2. **`public/llms.txt` kiszivárogtatja a pontos súlyozást** (Quality 25%, RTEP 35%,
   Momentum 20%, Value 20%, Smart Money 0%). Pontosan ez a "Coca-Cola recept", amit a
   methodology oldalakról 2026-09-02-án tudatosan kivettek. Ráadásul elavult
   ("pre-launch, waitlist", "nine years"). A guardrail lint (`scripts/docs-guardrail-lint.cjs`)
   csak `content/`, `src/components/sections`, `src/components/legal` alatt keres,
   a `public/` nincs benne, ezért nem szólt.
3. **A sitemap nem tartalmazza a docs aloldalakat**, csak a `/docs` gyökeret
   (`src/app/sitemap.ts`). ~38 methodology/feature/account oldal hiányzik belőle.
4. **A feliratkozók egy JSONL fájlba mennek a szerveren, és senki nem nézi.** Nincs admin
   nézet, nincs értesítés, nincs ESP. Részletek a 6. pontban.
5. **Két külön lista van**: a "Join us now" szekció a `/api/waitlist`-re POST-ol
   (`waitlist.jsonl`), miközben létezik egy `/api/newsletter` is (`newsletter.jsonl`),
   amit senki nem hív. A blog sidebar hírlevél-doboza pedig disabled placeholder.
6. **Törött link**: `content/blog/welcome-to-returnolio.mdx:43` a `/pricing`-ra linkel,
   ami nem route (a pricing a főoldal `#pricing` szekciója). A README és a
   LAUNCH_CHECKLIST is `/about`, `/pricing` route-okról beszél, amik nincsenek.
7. **Néma hibák**: a docs "Was this helpful?" widget a nem létező `/api/docs/feedback`-re
   POST-ol; a footer social linkek (`siteConfig.social`) olyan profilokra mutatnak,
   amikről a kódkomment szerint nem tudni, léteznek-e.

---

## 1. Methodology oldalak – "recept nélkül, FOMO-val, virálisan"

### Jelenlegi állapot
- 9 oldal a `content/docs/methodology/` alatt. 2026-09-02-án már újraírták őket úgy,
  hogy a pilléreket és a bemeneteket megmutatják, de a súlyozást nem
  (`rtep-score.mdx`: "How the five come together into one number is the part we keep
  to ourselves."). A guardrail lint tiltja a százalékos súlyokat és a `20/30/0/30/20`
  jellegű felosztást.
- Jogi korlát, amit tiszteletben kell tartani: Paddle AUP + "financial publisher, not
  adviser" pozíció (`docs/PAYMENT-COMPLIANCE.md`, `proof.tsx` fejkommentje). Tilos:
  "signals", "picks", "follow us", "$10k becomes $X", "beat the market" headline.
  A FOMO-t a *módszer exkluzivitására* és a *munka mennyiségére* kell építeni, nem
  hozamígéretre.

### Teendők
1. **`public/llms.txt` azonnali javítás** (külön, első PR): a súlyok törlése, a
   "pre-launch/waitlist" és "nine years" sorok frissítése a jelenlegi, docs-ban használt
   megfogalmazásra ("thirteen years, point-in-time", "1,000+ companies"). A pillérek
   maradhatnak név szerint, a súly nem. A "Monthly Top 10 = the team's own actual
   portfolio" mondatot igazítani a `proof.tsx` compliance-szövegéhez ("disclosed with
   the monthly edition, after the fact").
2. **Guardrail lint kiterjesztése** a `public/*.txt` fájlokra és a `src/app` alatti
   oldalakra (`TARGET_DIRS` bővítése), hogy ez ne fordulhasson elő újra.
3. **FOMO-réteg a methodology oldalakon** (MDX + 2-3 új MDX komponens a
   `src/components/mdx/` alatt):
   - `<KeptInternal>` callout minden methodology oldal tetejére vagy végére: "What we
     publish / What stays internal" két oszlop. A tiltott dolog listázása maga a horog
     ("the blend weights", "the fair-value method selection rule", "the monthly
     re-ranking cut-off"). Ez a klasszikus "titkos recept" framing, jogilag tiszta.
   - `<WorkBehindIt>` számsáv: 13 év point-in-time adat, 12 038 EDGAR filing-dátum,
     ~1 000 cég havonta újrapontozva, 0 manuális felülírás. Ezek már szerepelnek a
     `proof.tsx`-ben, tehát compliance-ellenőrzött számok.
   - `<SeeItOnATicker>`: minden pillér-oldal végén egy konkrét, valós ticker-példa
     kártya ("Open NVDA in the app and see all five pillars"), ami az app deep-linkjére
     visz (`${appUrl}/ticker/NVDA`, az útvonalat a webapp repóból kell ellenőrizni).
     Ez a "látod, hogy működik, de csak belül" horog.
   - `<ShareBar>`: X / LinkedIn / copy-link gombok előre megírt, idézhető mondattal
     (pl. "Returnolio treats R&D as an asset, not an expense. Most screeners don't.").
     Egy erős, önállóan idézhető "one-liner" oldalanként a frontmatterben
     (`shareLine:` mező), ez lesz a share szöveg és az OG kép fő sora is.
   - **Dinamikus OG kép docs oldalanként**: `src/app/docs/[...slug]/opengraph-image.tsx`
     az oldal címével és a `shareLine`-nal, a meglévő `src/app/opengraph-image.tsx`
     stílusában. Enélkül minden megosztott docs-link ugyanazt az általános kártyát
     mutatja.
4. **Szöveg-átnézés a 9 oldalon** a fenti komponensek beillesztésével. Szabály: minden
   oldal a "mit tudsz meg" és a "mit nem adunk ki" kontrasztjára épüljön, egy idézhető
   mondattal, egy app-deeplinkkel. A `backtest.mdx` maradjon őszinte (a vesztes évek
   bent maradnak, ez a hitelesség maga a viralitás-alap).
5. Frontmatter bővítés `src/lib/docs.ts`-ben: `shareLine?: string`.

### Elfogadás
- `node scripts/docs-guardrail-lint.cjs` zöld; `grep -rn "25%\|35%" public/llms.txt` üres.
- Minden methodology oldalon van KeptInternal + ShareBar + egyedi OG kép
  (`curl -I https://returnolio.com/docs/methodology/rtep-score/opengraph-image` 200).

---

## 2. Linkek – minden link működjön és jó helyre vigyen

### Ismert hibák
| Hol | Hiba | Javítás |
| --- | --- | --- |
| `content/blog/welcome-to-returnolio.mdx:43` | `/pricing` nem route | `/#pricing` |
| `src/components/docs/was-this-helpful.tsx:116` | `/api/docs/feedback` nem létezik | route létrehozása (append JSONL a `website-data/docs-feedback.jsonl`-be, ugyanaz a minta mint `signup-store.ts`), vagy a widget eltávolítása |
| `src/lib/site-config.ts:147-154` | social profilok létezése nem igazolt | Tamás megerősítése után a nem létezőket kivenni (halott social link a footerben rontja a bizalmat és az SEO-t) |
| `src/app/layout.tsx:428` | `sameAs: twitter.com/returnolio` | a valós, létező profilok listája (x.com), lásd GEO |
| `README.md`, `docs/LAUNCH_CHECKLIST.md` | `/about`, `/pricing` route-ként | anchor-ként javítani (dokumentációs hiba, de félrevezeti a következő fejlesztőt) |
| `src/components/sections/social-proof.tsx` | Trustpilot placeholder | 5. pont |

### Rendszerszintű megoldás
1. **Linkellenőrző script**: `scripts/check-links.mjs`. `next build` + `next start` után
   végigjárja a sitemap URL-jeit, kigyűjti az `<a href>`-eket, ellenőrzi:
   belső linkek 200-at adnak (redirect is számít), `#anchor` linkek célja létezik a
   cél-oldalon (`id=`), külső linkek HEAD-re nem 4xx/5xx (időtúllépés csak warning).
   Beépítés a `pr-check.yml`-be blokkolóként, a külső linkekre `continue-on-error`.
2. **MDX-linkek lintje** a guardrail mellé: `](/...)` hivatkozások a `content/` alatt
   csak létező route-ra vagy létező docs slugra mutathatnak (a `getAllDocs()` listája
   alapján). Ez fogja meg a következő `/pricing`-szerű hibát build-időben.
3. **CTA-célok kézi ellenőrzése** (Opus egyszer, checklistben dokumentálva):
   `signupUrl` → `/#pricing`, pricing kártyák → `app.returnolio.com/checkout?plan=…`
   ill. `login?mode=register&plan=free`, `loginUrl` → `/login`, `/checkout` → app redirect,
   MCP oldal → `mcp.returnolio.com/mcp`, "Open the app" → app. Mindegyik legyen élő
   (curl 200/302).

---

## 3. Indexelés – a Google találja meg az oldalt

### Miért nem található ma
`src/lib/site-config.ts:91` `SEARCH_HIDDEN = true`. Ettől: `robots.txt` = `Disallow: /`,
nincs sitemap hivatkozás, minden oldalon `noindex, nofollow` meta. A kódkomment szerint
a jogi átnézésre vár ("Flip to false when the legal round closes"). A `site:returnolio.com`
keresés ma 0 találatot ad.

### Sorrend (fontos, ne fordítva)
1. **Előbb az SEO-alapok** (4. pont: canonical, sitemap, per-oldal metadata), utána a kapcsoló.
   Ha előbb kapcsolunk, a Google rossz canonicalokkal és fél sitemappel indexel, és
   hetekbe telik, mire újrakúszik.
2. **Kapcsoló PR** (egyetlen apró PR, hogy visszavonható legyen):
   - `SEARCH_HIDDEN = false`
   - `.lighthouserc.json`: `categories:seo` vissza `warn`-ra vagy `error`-ra, a küszöböt
     a `LH_PRESET=desktop npm run audit:lighthouse https://returnolio.com` mért értékéből
   - `src/app/robots.test.ts` frissítése
   - Deploy után ellenőrzés: `curl https://returnolio.com/robots.txt` (Allow /,
     Sitemap sor), `curl -s https://returnolio.com | grep -i robots` (index, follow).
3. **Cloudflare**: ellenőrizni, hogy a bot-fight / "Block AI scrapers" beállítás nem
   blokkolja a Googlebot/Bingbot-ot (a README szerint a datacenter IP-ket a CF challenge
   elkapja; a verified botok kivételt kapnak, de ezt a CF dashboardon kell látni).
4. **Search Console + Bing Webmaster** (csak Tamás tudja): property felvétele DNS
   verifikációval, sitemap beküldése, "Request indexing" a főoldalra, a `/docs`-ra és
   3-4 kulcs docs oldalra. Bing azért is kell, mert a ChatGPT keresés Bing-alapú.
5. **IndexNow** (opcionális, olcsó): `public/<key>.txt` + deploy utáni ping a Bing/Yandex
   IndexNow végpontjára a sitemap URL-jeivel. Egy 15 soros lépés a `deploy.yml`-ben.
6. **Külső jelek** (Tamás): az app.returnolio.com footeréből link a returnolio.com-ra
   és a `/docs`-ra; a social profilok bio-jában a domain; a Beehiiv hírlevél footerében
   a domain. Backlink nélkül egy új domain hónapokig a 3. oldalon marad.

---

## 4. SEO optimalizálás

### Technikai (kód)
1. **Per-oldal canonical**: a globális `alternates.canonical` törlése a `layout.tsx`-ből,
   helyette a blog `[slug]`, docs `[...slug]`, `blog/topic/[topic]`, `legal/[slug]`,
   `support`, `blog`, `docs` oldalakon `alternates: { canonical: \`${siteConfig.url}${path}\` }`.
   Legegyszerűbb egy `canonicalFor(path)` helper a `site-config.ts`-ben.
2. **Sitemap teljessé tétele** (`src/app/sitemap.ts`): `getAllDocs()` alapján minden
   docs oldal `lastModified: lastUpdated`-del; `BLOG_TOPICS` topic-oldalak; `/support`;
   a 4 legal oldal. `www.returnolio.com` → apex 301 (Cloudflare Bulk Redirect vagy
   `next.config.ts` redirect a `host` alapján).
3. **Strukturált adat**:
   - blog `[slug]`: `BlogPosting` (headline, datePublished, author, image, publisher →
     `#organization`), `BreadcrumbList`
   - docs `[...slug]`: `TechArticle` + `BreadcrumbList`
   - `Organization.sameAs`: a valós social profilok
   - `FAQPage` már megvan a főoldalon, marad
4. **Metadata**: docs oldalakon `openGraph` blokk (ma csak title/description); a
   `keywords` meta törölhető (a Google nem használja, zaj). Title template marad.
5. **OG képek**: docs és blog oldalanként dinamikus (1. pont 3. alpont). Blog: cover
   képből vagy címből.
6. **RSS feed**: `src/app/feed.xml/route.ts` a blog postokból. Kell az SEO-nak, a
   Beehiiv "RSS-to-send" automatizálásnak és az AI-crawlereknek is. `<link rel="alternate"
   type="application/rss+xml">` a layoutba.
7. **Lighthouse SEO kategória** visszakapcsolása a kapcsoló PR-ben (3. pont).

### Tartalmi
8. **Kulcsszó-térkép** oldalanként (Opus a docs frontmatter description-jeit igazítsa):
   - főoldal: "stock research platform for long-term investors", "fundamental stock
     scoring"
   - rtep-score: "economic profit stock score", "what is RTEP"
   - fair-value: "fair value calculator stocks", "intrinsic value model"
   - backtest: "point-in-time backtest", "look-ahead bias"
   - smart-money: "insider trading tracker", "congress stock trades"
   - mcp: "Claude MCP stock research", "connect Claude to stock data"
   Egy kulcsszó egy oldalra, H1-ben és az első bekezdésben szerepeljen, a description
   155 karakter alatt.
9. **Belső linkelés**: minden blog post legalább 2 docs oldalra, minden docs oldal
   1 blog postra linkeljen (a `related-posts.tsx` mintájára egy `RelatedDocs` a blogban).

---

## 5. GEO (Generative Engine Optimization) – hogy a ChatGPT/Claude/Perplexity idézzen

1. **`llms.txt` újraírása** (1. pont 1. alpont) + **`llms-full.txt`** generálása build-időben
   a docs MDX-ekből (script a `scripts/` alá, a `getDocsSearchIndex()` "plain text"
   kimenetét használva; a guardrail lint fusson rá is). Ez a fájl az, amit az AI
   crawlerek egyben beolvasnak.
2. **`robots.ts`**: explicit `allow` a `GPTBot`, `ClaudeBot`, `PerplexityBot`,
   `Google-Extended`, `Bingbot` user-agentekre (a `*` szabály mellett, hogy egy
   későbbi tiltás se érintse őket véletlenül). Cloudflare "Block AI bots" kapcsoló
   **kikapcsolva** a returnolio.com zónán (Tamás, dashboard).
3. **Entitás-definíció**: a főoldal első `<p>`-je és a `siteConfig.description` legyen
   egy tiszta, idézhető definíció-mondat ("Returnolio is a stock research platform that
   scores 1,000+ US, UK and Canadian companies 0-100 on business quality, economic
   profit, value, momentum and insider activity."). Ugyanez szó szerint az
   `Organization.description`-ben, az `llms.txt`-ben és az About szekcióban. Az AI
   modellek a konzisztens, ismételt definíciót veszik át.
4. **Docs oldalak markdown-változata**: `src/app/docs/[...slug]/page.md/route.ts`
   (vagy `Accept: text/markdown` content negotiation a middleware-ben), ami a nyers
   MDX-et adja vissza JSX nélkül. Az AI-keresők tisztább forrást kapnak, a `llms.txt`
   erre linkeljen.
5. **Kérdés-válasz formátum**: minden methodology oldalon egy "Common questions"
   H2 alatt 3 rövid Q&A (H3 kérdés + 2 mondatos válasz) + `FAQPage` schema az adott
   oldalon. Ez az, amit a generatív keresők snippetként átvesznek.
6. **Statisztika-oldal**: egy `/docs/data/by-the-numbers` oldal a nyers, idézhető
   számokkal (lefedett cégek száma, filing-dátumok, backtest évek, frissítési
   gyakoriság). Az AI-válaszok szeretik a számszerű forrást idézni.
7. **Bing indexelés** (3. pont 4. alpont) a ChatGPT miatt kötelező.

---

## 6. "Join us now" feliratkozók – a No1 hiba

### Mi történik ma
`newsletter-cta.tsx` → `POST /api/waitlist` → `saveSignup("waitlist", email)` →
`/home/claudeuser/website-data/waitlist.jsonl` a Hetzner VPS-en. Az adat **megvan**
(a deploy rsync nem törli, mert repón kívül van), csak semmi nem mutatja meg. Ugyanez
az `/api/newsletter` → `newsletter.jsonl`, amit semmi nem hív.

**Első lépés, még kód előtt (Tamás vagy Opus SSH-val):**
```
ssh <deploy-user>@<hetzner> 'wc -l /home/claudeuser/website-data/*.jsonl; tail -5 /home/claudeuser/website-data/waitlist.jsonl'
```
Ez megmondja, hány feliratkozó gyűlt össze eddig. Ezeket importálni kell a Beehiiv-be.

### Javítás (három réteg, mindegyik kell)
1. **Admin nézet**: `src/app/admin/signups/page.tsx` (a `/admin/*` már Cloudflare Access
   mögött van, csak a tulaj emailje engedett). Beolvassa mindkét JSONL-t, táblázat
   email + dátum + forrás, összesítő szám, CSV export gomb. `force-dynamic`. Tab
   felvétele az `admin/layout.tsx` `TABS` listájába.
2. **Azonnali értesítés** minden új feliratkozásról: a `saveSignup` után egy
   `notifySignup(email, list)` hívás. Opció A: Telegram bot (a webapp már használ
   Telegram alertet a LAUNCH_CHECKLIST szerint, ugyanaz a bot token használható,
   env `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`). Opció B: email a
   `support@returnolio.com`-ra a webapp `SUPPORT_INGEST_URL` mintájára. **Javaslat: A**,
   mert már van infrastruktúra és azonnal látható telefonon. Hiba esetén nem dobjon
   500-at a felhasználónak (fire-and-forget, `console.error`).
3. **Beehiiv továbbítás** (7. pont). A JSONL marad fallbackként és auditként.
4. **Egy lista**: az `/api/newsletter` és `/api/waitlist` összevonása egy
   `/api/subscribe` route-ba `source` mezővel (`home-cta`, `blog-sidebar`, `docs-nudge`,
   `gate`). A régi két útvonal 308-cal ide. A `coming-soon-gate.tsx` és a
   `newsletter-cta.tsx` közös `useSubscribe()` hookot használjon, ne másolt kód legyen.
5. **Blog sidebar** `NewsletterCard` élesítése ugyanezzel a hookkal (ma disabled).
6. **Docs oldalak `<UpgradeNudge>`** mellé egy `<SubscribeNudge>` is (methodology
   oldalak végén), mert a FOMO-tartalom természetes konverziós pontja a hírlevél.
7. **Teszt**: `src/app/api/subscribe/route.test.ts` (vitest már be van állítva):
   valid email → 200 + store hívás; invalid → 400; rate limit → 429; Beehiiv hiba →
   mégis 200 (a JSONL-be ment) + hiba logolva.

---

## 7. Beehiiv bekötés

### Amit csak Tamás tud (Opus nem tud fiókot csinálni)
1. Beehiiv fiók: beehiiv.com → publication "Returnolio" → custom domain beállítása
   (`news.returnolio.com` vagy `newsletter.returnolio.com`, Cloudflare DNS CNAME +
   DKIM/SPF rekordok a Beehiiv által megadott értékekkel).
2. Settings → API → API key generálása; a Publication ID (`pub_…`) kimásolása.
3. Titkok elhelyezése: **GitHub repo secrets** (`BEEHIIV_API_KEY`, `BEEHIIV_PUBLICATION_ID`)
   ÉS a VPS-en a PM2 env-ben (`ecosystem.config.cjs` `env` blokk vagy `.env` fájl a
   `/home/claudeuser/website`-ben), mert ezek **szerveroldali** (nem `NEXT_PUBLIC_`)
   változók, a runtime-nak kellenek, nem a buildnek. A `deploy.yml` `.env.production`
   lépése ma csak build-időset ír; Opus adjon hozzá egy rsync-elt `.env.runtime`
   fájlt vagy dokumentálja, hogy a VPS-en kézzel kell beállítani.
4. Double opt-in: Beehiiv-ben eldönteni (EU/GDPR miatt ajánlott bekapcsolni).
5. Welcome email megírása a Beehiiv-ben (az első automatikus levél).

### Kód (Opus)
1. `src/lib/beehiiv.ts`: `subscribe({ email, source, utm })` →
   `POST https://api.beehiiv.com/v2/publications/{id}/subscriptions` body:
   `{ email, reactivate_existing: true, send_welcome_email: true, utm_source: "returnolio.com", utm_medium: source }`,
   `Authorization: Bearer`. 10 s timeout, hibánál `throw`. Ha az env hiányzik,
   no-op + egyszeri warning (dev környezet).
2. `/api/subscribe` route: `await saveSignup(...)` (mindig), majd `beehiiv.subscribe`
   try/catch-ben. Válasz mindig 200, ha a JSONL mentés sikerült.
3. **Backlog import script**: `scripts/import-signups-to-beehiiv.mjs` – beolvassa a
   két JSONL-t, dedupe, Beehiiv API-n egyesével feltölti (rate limit: 1/s), naplózza
   az eredményt. Egyszer kell lefuttatni a VPS-en a kulcsokkal. `send_welcome_email:
   false` az importnál, hogy a régi feliratkozók ne kapjanak "welcome"-ot hónapokkal
   később; helyette egy külön Beehiiv kampány "we're live".
4. `next.config.ts` CSP `connect-src`: **nem kell** bővíteni, mert a hívás
   szerveroldali. Ha Beehiiv embed formot használnánk (nem javasolt, a saját form
   szebb és már kész), akkor kellene.
5. `docs/LAUNCH_CHECKLIST.md` Beehiiv sorának frissítése, README "Newsletter" szekció.

### Elfogadás
- Feliratkozás a prod oldalon → 30 mp-en belül Telegram üzenet + a cím megjelenik a
  Beehiiv Audience listában + egy sor a `waitlist.jsonl`-ben + látszik `/admin/signups`-on.
- Backlog import után a Beehiiv subscriber szám = JSONL egyedi címek száma.

---

## 8. Social proof szekció

### Jelenlegi állapot és jogi keret
`src/components/sections/social-proof.tsx` létezik (5 kép-placeholder + Trustpilot
placeholder), de **ki van kapcsolva** a `page.tsx`-ben ezzel az indokkal: "invented
reviews are a blacklisted unfair practice under EU consumer law". Ez helytálló: az
EU UCPD I. melléklet 23b-23c pontja (Omnibus irányelv, 2022 óta) kifejezetten tiltja
a hamis fogyasztói véleményeket, és Magyarországon a GVH bírságol érte. A Paddle
felülvizsgálat is nézi. Egy egyéni vállalkozó neve alatt futó oldalnál ez személyes
kockázat.

**Ezért a terv: a szekció megépül és bekerül, OpenArt-képekkel, de a képek és a
szövegek nem állítanak nem létező ügyfél-véleményt.** Ami mehet:

1. **"Built on" / "Data from" sáv**: az adatforrások és technológiák logói, amiket
   ténylegesen használunk (SEC EDGAR, Cloudflare, Paddle, Anthropic MCP). A guardrail
   tiltja a data vendor nevét (FMP), azt nem. Ez a "logo strip" formátum, amit a
   felhasználó kért, valós tartalommal.
2. **"What the numbers say" kártyák** OpenArt-generált háttérképekkel: 13 év, 12 038
   filing, 1 000+ cég, 0 override, a publikált vesztes évek száma. Kép + egy szám +
   egy sor. Ez vizuálisan pont úgy néz ki, mint egy testimonial-grid, de számot mutat,
   nem személyt.
3. **Alapítói kártya**: Tamás valódi fotója vagy OpenArt-portré illusztráció (jelölve,
   hogy illusztráció) + egy valódi idézet arról, miért épült a rendszer + "own money
   since June 2026" (ez a `live-portfolio.tsx` valós adata).
4. **Trustpilot**: valódi profil létrehozása (Tamás), a widget beágyazása amint van
   3+ valódi vélemény. Addig a placeholder helyett egy "Leave the first review" link.
5. **Valós user-idézetek slotja**: a szekció legyen adatvezérelt
   (`src/content/testimonials.json`), üres listánál a grid nem renderel. Amint jön
   valódi email/DM/tweet engedéllyel, egy JSON-sor és megjelenik. Az OpenArt-képek
   itt háttér/avatar-illusztrációként használhatók, a név és a szöveg valódi.
6. **Képgenerálás** (Tamás, OpenArt): stílus a design-systemhez igazítva (dark/light
   variáns, gold + emerald akcent, absztrakt pénzügyi/adat motívumok, nem arcok
   testimonialnak). Méret 1200×800, WebP, a `public/images/social/` alá, `next/image`.
   Opus a komponenst építi, a képek nevét és méretét előre rögzíti a
   `docs/media-assets.md`-ben, hogy Tamás pontosan tudja, mit generáljon.

Ha Tamás mégis kitalált testimonialokat akar, azt itt külön döntésként rögzítjük
(9. pont kérdés), és a kockázat az övé. Ebben a tervben nem szerepel.

---

## 9. Blog

### Jelenlegi állapot
4 MDX a `content/blog/` alatt, mind placeholder (a `top-10-ideas-2026.mdx` saját
magáról írja, hogy az, és kitalált számokat tartalmaz). `BLOG_HIDDEN = true` prodon,
staging látja. A blog infrastruktúra (topic oldalak, related posts, sidebar, TTS
"Listen" gomb) kész.

### Teendők
1. **Placeholder törlése**: `top-10-ideas-2026.mdx` ki (a Top 10 fizetős, blogon nem
   adunk ki tickereket, compliance). A másik három átírása valódi tartalomra.
2. **Első 6 poszt** (Opus írja, Tamás átnézi; mindegyik 900-1 400 szó, egy kulcsszó,
   2 docs-link, ShareBar, `shareLine` frontmatter):
   | Slug | Cím | Topic | Kulcsszó |
   | --- | --- | --- | --- |
   | `welcome-to-returnolio` | Why we built a calmer stock research tool | getting-started | stock research for long-term investors |
   | `what-is-economic-profit` | Economic profit: the number most screeners ignore | methodology | economic profit vs net income |
   | `why-rtep-not-pe` | Why P/E alone keeps picking value traps | methodology | P/E ratio value trap |
   | `how-we-backtested-13-years` | We backtested 13 years and lost 7 of them. Here's why we published anyway | methodology | point-in-time backtest look-ahead bias |
   | `sunday-routine-long-term` | The 30-minute Sunday routine | getting-started | weekly investing routine |
   | `insiders-and-congress` | What insiders and Congress buy, and how to read it without overreacting | smart-money | congress stock trades tracker |
   Plusz egy 7.: `claude-mcp-stock-research` – "Ask Claude about any stock with
   Returnolio's MCP" (features, kulcsszó: Claude MCP finance). Ez a legkeresettebb új
   téma és egyedi a piacon.
3. **Compliance a blogban**: nincs ticker-ajánlás, nincs "buy/sell", a backtest-számok
   csak a `proof.tsx`/`backtest.mdx` kanonikus értékei (run 21d3f5d0). A guardrail lint
   fut a `content/blog`-ra is, ez megfogja a régi számokat.
4. **`BLOG_HIDDEN = false`** külön PR-ben, amikor legalább 4 poszt kész. Ezzel együtt:
   sitemap, RSS, nav "Blog" visszajön (automatikus a flag alapján).
5. **Beehiiv RSS-to-send**: a `feed.xml`-t a Beehiiv automation olvassa, így minden
   új poszt kimegy hírlevélként kézi munka nélkül.
6. **Szerző**: "Tamás" helyett teljes név + rövid bio + fotó (`author` mező bővítése
   objektummá a `blog.ts`-ben), mert az E-E-A-T (Google) és az AI-idézés is szerzőt
   keres. `Person` schema a `BlogPosting.author`-ban.
7. **Blog OG kép** dinamikusan (`src/app/blog/[slug]/opengraph-image.tsx`).

---

## 10. Végrehajtási sorrend és PR-bontás (Opus)

Sorrend elve: előbb ami adatot veszít vagy szivárogtat, utána az SEO-alap, csak
utána a láthatósági kapcsolók, végül a tartalom.

| # | PR | Tartalom | Kapcsoló? |
| --- | --- | --- | --- |
| 1 | `fix/llms-txt-and-guardrail` | llms.txt súlyok ki + guardrail `public/`-ra | nem |
| 2 | `feat/signups-admin-and-notify` | `/admin/signups`, Telegram értesítés, `/api/subscribe` összevonás, hook, tesztek | nem |
| 3 | `feat/beehiiv` | `beehiiv.ts`, route bekötés, import script, blog sidebar élesítés | env kell (Tamás) |
| 4 | `fix/links` | `/pricing` link, feedback endpoint, social linkek tisztítása, link-checker script + CI | nem |
| 5 | `feat/seo-foundation` | per-oldal canonical, teljes sitemap, JSON-LD, RSS, OG képek docs+blog, www redirect, robots AI-botok | nem |
| 6 | `feat/methodology-fomo` | MDX komponensek, 9 oldal átírás, shareLine, Q&A blokkok, by-the-numbers oldal, llms-full.txt | nem |
| 7 | `feat/social-proof` | szekció adatvezérelt újraépítése, képslotok, visszakapcsolás a `page.tsx`-ben | igen (szekció) |
| 8 | `content/blog-first-posts` | 6-7 poszt, placeholder törlés, szerző-objektum | nem |
| 9 | `chore/search-visible` | `SEARCH_HIDDEN=false`, lighthouse seo on, robots teszt | **igen** |
| 10 | `chore/blog-visible` | `BLOG_HIDDEN=false` | **igen** |

A 9. és 10. PR csak akkor mehet, ha Tamás jelezte, hogy a jogi átnézés lezárult
(vagy explicit vállalja nélküle). Minden PR-nél: `npm run lint`, `npm run lint:a11y`,
`node scripts/docs-guardrail-lint.cjs`, `npm test`, `npx next build` lokálisan zöld.

---

## 11. Amit csak Tamás tud megcsinálni (fiókok, DNS, dashboardok)

- [ ] Beehiiv fiók + publication + custom sending domain + API key + Publication ID
- [ ] Titkok: `BEEHIIV_API_KEY`, `BEEHIIV_PUBLICATION_ID`, `TELEGRAM_BOT_TOKEN`,
      `TELEGRAM_CHAT_ID` a GitHub secretsbe és a VPS runtime env-be
- [ ] Google Search Console + Bing Webmaster property, sitemap beküldés (a 9. PR után)
- [ ] Cloudflare: AI-bot blokkolás ki, verified bots engedve, `www` → apex redirect
- [ ] Social profilok: melyik létezik (YouTube, TikTok, X, Instagram, Threads, Pinterest)
- [ ] Trustpilot business profil
- [ ] OpenArt képek a 8. pont specifikációja szerint
- [ ] Jogi átnézés lezárása vagy döntés, hogy nélküle indexelünk
- [ ] Blog posztok átnézése és jóváhagyása publikálás előtt
- [ ] Alapítói fotó + 2-3 mondatos bio a blog szerzőkártyához

---

## 12. Nyitott kérdések Tamásnak

1. **Jogi kör**: a `SEARCH_HIDDEN` kódkomment szerint az indexelés a jogi átnézésre
   vár. Lezárult? Ha nem, indexeljünk nélküle? (Enélkül a 3. feladat nem teljesíthető.)
2. **Social proof**: elfogadod a 8. pontban leírt, valós adatokra épülő változatot,
   vagy ragaszkodsz kitalált ügyfél-idézetekhez? Utóbbit tudjuk építeni, de a jogi
   kockázatot (UCPD 23b, GVH, Paddle) írásban vállalnod kell.
3. **Értesítés csatorna** feliratkozáskor: Telegram (mint a webapp) vagy email?
4. **Beehiiv sending domain**: `news.returnolio.com` jó?
5. **Social profilok**: a hat footer-linkből melyik létezik ténylegesen?
6. **Blog szerző**: teljes néven és fotóval jelenjen meg "Tamás Bodri" (E-E-A-T miatt
   erősen ajánlott)?
7. **Meddig FOMO**: a methodology oldalakon a "What stays internal" lista konkrétan
   nevezze meg, mit nem adunk ki (súlyok, cut-off, fair-value választási szabály)?
   Ez a legerősebb horog, de egyben térkép is annak, aki reverse-engineerelni akar.
8. **A `top-10-ideas-2026` placeholder** törölhető, vagy helyette egy "how the Top 10
   gets picked" teaser poszt kell ticker nélkül?
