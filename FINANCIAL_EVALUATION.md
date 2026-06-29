# 💰 Finančné ohodnotenie projektu Lectio Divina

**Dátum vypracovania:** 3. jún 2026
**Verzia hodnotenia:** 2.0 (kompletná revízia)
**Predmet hodnotenia:** Softvérové aktíva projektu Lectio Divina – backend (Next.js) + mobilná aplikácia (Flutter)

> Toto hodnotenie nahrádza pôvodný odhad z 18. októbra 2025 (€102 500). Od vtedy projekt výrazne narástol v rozsahu, funkčnosti aj zrelosti. Web `web_parochia` nie je predmetom tohto hodnotenia.

---

## 1. Zhrnutie (Executive Summary)

Projekt **Lectio Divina** je vyspelý, produkčne nasadený full-stack ekosystém pozostávajúci z rozsiahlej **backendovej aplikácie (Next.js 16 / React 19)** a **mobilnej aplikácie (Flutter, v10.1.2)** pre iOS a Android. Hodnotenie vychádza z **metódy nákladov na nahradenie (Replacement Cost Method)** – odhaduje, koľko by stálo vyvinúť rovnaký softvér od nuly pri aktuálnych trhových sadzbách v strednej Európe.

| Komponent | Rozsah kódu | Odhadovaný čas | Odhadovaná hodnota (EUR) |
| :--- | :--- | :--- | :--- |
| **Backend (Next.js)** | ~504 TS/TSX súborov, ~130 000 riadkov | 1 600 – 2 200 hod. | 80 000 – 154 000 € |
| **Mobilná aplikácia (Flutter)** | ~109 Dart súborov, ~18 000 riadkov | 1 000 – 1 400 hod. | 50 000 – 98 000 € |
| **Celkom** | **~613 súborov, ~148 000 riadkov** | **2 600 – 3 600 hod.** | **130 000 – 252 000 €** |

### 🎯 Odporúčaná realistická hodnota softvérových aktív: **€170 000 – €200 000**
### Stredová (headline) hodnota: **≈ €186 000**

*Poznámka: Tento odhad predstavuje hodnotu vytvoreného softvéru (replacement cost). Nezahŕňa hodnotu značky, používateľskej základne, vygenerovaných príjmov, obsahu (liturgické texty, audio nahrávky), duševného vlastníctva ani prevádzkové náklady na infraštruktúru.*

---

## 2. Metodika

- **Prístup:** Replacement Cost Method – ohodnotenie nákladov na opätovné vytvorenie softvéru v rovnakom rozsahu a kvalite.
- **Vstupné dáta:** Priama analýza zdrojového kódu k 3. 6. 2026 (počty súborov, endpointov, obrazoviek, komponentov, závislostí, migrácií).
- **Hodinové sadzby:** €50 – €70 / hod. (blended sadzba zmiešaného tímu junior/medior/senior pre SK/CEE región).
- **Scenáre:** Konzervatívny (nižší počet hodín × €50), Realistický (stredný počet × €60), Optimistický (vyšší počet × €70).

---

## 3. Analýza Backendovej Aplikácie (Next.js)

Backend je rozsiahla monolitická Next.js aplikácia (App Router) plniaca úlohu API, administračného CMS aj verejného webu súčasne.

### 3.1 Technologický zásobník
- **Framework:** Next.js **16.1.1** (App Router), **React 19**, TypeScript 5 (strict)
- **Databáza:** Supabase (PostgreSQL) + SSR auth, Row-Level Security
- **Platby:** Stripe (`^19.3.0`), Mollie (`^4.4.0`), Apple/Google IAP (verifikácia účteniek)
- **AI:** OpenAI (`^6.0.1`), Google Gemini (`@google/generative-ai`)
- **Médiá:** FFmpeg (audio/video), Sharp (obrázky), react-easy-crop
- **Editory:** TipTap, Editor.js, TinyMCE
- **Notifikácie:** Firebase Admin (FCM), Nodemailer (e-mail)
- **Infraštruktúra:** Docker, PM2, Vercel, Upstash Redis (rate limiting), reCAPTCHA Enterprise

### 3.2 Rozsah (konkrétne metriky, jún 2026)

| Metrika | Hodnota |
| :--- | :--- |
| API endpointy (`route.ts`) | **148** |
| Stránky (`page.tsx`) – admin + verejné | **111** |
| React komponenty (`.tsx`) | **260** |
| TypeScript súbory spolu (`.ts` + `.tsx`) | **504** |
| SQL migrácie | **107** |
| Odhadovaný počet riadkov kódu | **~120 000 – 150 000** |
| Platobné brány | **3** (Stripe, Mollie, IAP) |
| Podporované jazyky obsahu | **8+** |

### 3.3 Kľúčové funkčné oblasti a zložitosť
- **E-commerce a platby (Veľmi vysoká):** predplatné, jednorazové dary, e-shop, objednávky, doprava, Stripe + Mollie + IAP, webhooky, validácia transakcií.
- **Bankové párovanie (Vysoká):** import XML výpisov, párovacie algoritmy, kontrola duplicít, rekonciliácia.
- **AI generovanie obsahu (Veľmi vysoká):** 12+ AI endpointov – generovanie Lectio Divina, článkov, obrázkov, audio (TTS), prekladov, kontrola gramatiky, sledovanie limitov podľa úrovne predplatného.
- **Podcastový systém (Vysoká):** spracovanie audia cez FFmpeg, generovanie RSS feedov a obalov, automatizované publikovanie (cron).
- **Liturgický kalendár (Vysoká):** viacjazyčný, predgenerovaný, verziovaný, mapovanie na lectio zdroje.
- **Notifikácie / FCM (Vysoká):** push na iOS + Android, témy, plánovanie, cielenie podľa jazyka a verzie aplikácie, logy doručenia.
- **Newsletter a kampane (Vysoká):** šablóny, správa kontaktov, hromadné rozosielanie, jazykové cielenie.
- **Administračný panel (Veľmi vysoká):** 50+ admin stránok, rich-text editory, hromadné operácie, import/export (XLSX), audit log.
- **Bezpečnosť (Stredná):** rate limiting (Redis), reCAPTCHA Enterprise, RLS, ochrana webhookov.

### 3.4 Odhad nákladov (Backend)
- **Konzervatívny:** 1 600 h × €50 = **€80 000**
- **Realistický:** 1 900 h × €60 = **€114 000**
- **Optimistický:** 2 200 h × €70 = **€154 000**

---

## 4. Analýza Mobilnej Aplikácie (Flutter)

Zrelá, produkčne nasadená aplikácia (verzia **10.1.2+5000013**) s pokročilou prácou s audiom, offline režimom a viacerými zdrojmi príjmov.

### 4.1 Technologický zásobník
- **Framework:** Flutter / Dart `^3.8.0`
- **Platformy:** iOS (min. 15.0), Android (minSDK 21, target 35)
- **Backend & dáta:** Supabase (Auth + PostgreSQL + sync)
- **Stav & DI:** Provider, RxDart, get_it
- **Audio:** just_audio, just_audio_background, audio_service, audio_session
- **Notifikácie:** Firebase Messaging, flutter_local_notifications, timezone
- **Auth:** e-mail/heslo, Google Sign-In, Apple Sign-In
- **Analytika:** Umami (privacy-first)

### 4.2 Rozsah (konkrétne metriky, jún 2026)

| Metrika | Hodnota |
| :--- | :--- |
| Dart súbory (`lib/`) | **109** |
| Obrazovky | **37** |
| Servisy | **20** |
| Widgety | **22** |
| Dátové modely | **9** |
| Testovacie súbory | **8** (unit, widget, integračné) |
| Externé balíčky | **45+** |
| Odhadovaný počet riadkov kódu | **~16 000 – 21 800** |
| Podporované jazyky | **3** (SK, EN, ES) |

### 4.3 Kľúčové funkčné oblasti a zložitosť
- **Pokročilý audio prehrávač (Veľmi vysoká):** prehrávanie na pozadí, ovládanie na zamknutej obrazovke, meditačné režimy a interlúdiá, offline sťahovanie, globálny mini-prehrávač.
- **Offline režim (Vysoká):** kešovanie čítaní Lectio, sťahovanie audia, kešovanie obrázkov, indikátor pripojenia.
- **Notifikácie (Vysoká):** FCM s témami, lokálne plánované notifikácie s časovými pásmami, deep-link smerovanie do 10+ typov obrazoviek.
- **Autentifikácia (Stredná-Vysoká):** e-mail, Google, Apple (s nonce), režim hosťa, bezpečné úložisko (Keychain/Keystore).
- **Monetizácia / IAP (Stredná):** úrovne predplatného, jednorazové dary, bankový prevod, in-app review.
- **Obsahové moduly (Vysoká):** Lectio Divina (6 krokov), Adorácia, Ruženec, Krížová cesta (14 zastavení), Duchovné cvičenia, Novinky, osobné Poznámky a Úmysly.
- **Lokalizácia (Stredná):** 3 jazyky s automatickou detekciou systémového jazyka.

### 4.4 Odhad nákladov (Mobilná aplikácia)
- **Konzervatívny:** 1 000 h × €50 = **€50 000**
- **Realistický:** 1 200 h × €60 = **€72 000**
- **Optimistický:** 1 400 h × €70 = **€98 000**

---

## 5. Celkové finančné hodnotenie

| Scenár | Backend | Mobil | **Spolu** |
| :--- | :--- | :--- | :--- |
| Konzervatívny | €80 000 | €50 000 | **€130 000** |
| Realistický | €114 000 | €72 000 | **€186 000** |
| Optimistický | €154 000 | €98 000 | **€252 000** |

### 🏆 Záverečná suma

**Lectio Divina Platform** (backend + mobilná aplikácia) je k 3. júnu 2026 ohodnotená na:

# ≈ €186 000 EUR

**Odporúčané rozpätie hodnoty softvérových aktív: €170 000 – €200 000.**

*Rast oproti hodnoteniu z októbra 2025 (€102 500) odráža výrazné rozšírenie funkcionality (podcastový systém, druhá platobná brána, IAP, AI pipeline, bankové párovanie), nárast rozsahu kódu a vyššiu zrelosť produktu.*

---

## 6. Hodnotenie kvality a zrelosti kódu

| Oblasť | Skóre | Poznámka |
| :--- | :--- | :--- |
| Architektúra | 9 / 10 | Čistá štruktúra App Router, jasné oddelenie zodpovedností |
| Kvalita kódu | 8.5 / 10 | 100 % TypeScript / typovo bezpečný Dart, 0 lint warningov v mobile |
| Úplnosť funkcií | 9 / 10 | Produkčne nasadené, široké pokrytie funkcií |
| Škálovateľnosť | 8.5 / 10 | Supabase + Next.js + Redis, pripravené na rast |
| Testovanie | 5 / 10 | Mobil má základné testy; backend bez automatizovaných testov |
| Monitoring | 4 / 10 | Chýba Sentry / štruktúrované logovanie v produkcii |
| Dokumentácia | 7 / 10 | Rozsiahle interné `docs/`, chýba formálna API špecifikácia |

---

## 7. Stav implementácie odporúčaní

Pôvodné odporúčania z roku 2025 boli čiastočne zapracované:

1.  **Refaktorizácia API štruktúry — ✅ Realizované.** API je logicky členené podľa funkcií v štruktúre Next.js App Router (`/api/admin`, `/api/lectio`, `/api/cron`, `/api/stripe`, `/api/mollie`, `/api/iap`, …).
2.  **Automatizované testy — ⚠️ Čiastočne realizované.** Mobilná aplikácia má 8 testovacích súborov (`flutter_test`, `mocktail`). Backend stále nemá automatizované testy – odporúča sa pokryť kritické endpointy (platby, autentifikácia, webhooky).
3.  **Dokumentácia API — ❌ Nerealizované.** Chýba formálna OpenAPI/Swagger špecifikácia pre 148 endpointov.
4.  **Monitoring a logovanie — ❌ Nerealizované.** Produkcia sa spolieha na `console.log`; chýba Sentry (error tracking) a štruktúrované logovanie (Pino/Winston).

---

## 8. Odporúčania na ďalšie zvýšenie hodnoty

| Priorita | Odporúčanie | Prínos |
| :--- | :--- | :--- |
| Vysoká | Sentry + štruktúrované logovanie (Pino) | Stabilita produkcie, rýchla diagnostika |
| Vysoká | Automatizované testy backendu (Vitest/Jest) pre platby a auth | Zníženie rizika regresií |
| Stredná | CI/CD pipeline (GitHub Actions) | Automatizované buildy a nasadenia |
| Stredná | OpenAPI/Swagger dokumentácia API | Jednoduchšia údržba a integrácie |
| Nízka | Migrácia `just_audio_background` z beta na stabilnú verziu | Spoľahlivosť audio prehrávania |

---

## 9. Riziká a obmedzenia hodnotenia

- **Závislosť na externých službách:** Supabase, Stripe, Mollie, Firebase, OpenAI – zmeny cien/podmienok ovplyvňujú prevádzku.
- **Chýbajúce testy a monitoring v backende** zvyšujú náklady na údržbu a riziko výpadkov.
- **Hodnota nezahŕňa obsah a IP:** liturgické texty, preklady a audio nahrávky majú samostatnú, nezapočítanú hodnotu.
- **Trhová hodnota vs. náklady:** skutočná predajná cena závisí aj od počtu aktívnych používateľov, príjmov a strategického záujmu kupujúceho.

---

*Hodnotenie vypracované analýzou zdrojového kódu k 3. júnu 2026. Ďalšiu revíziu odporúčame pri väčšej zmene rozsahu alebo po 12 mesiacoch.*
