# 🛒 E-shop / Podpora — analýza a návrh prepracovania

> Stav: **návrh** · Posledná aktualizácia: 18. jún 2026
> Kontext: shop bol len vo vývoji (nedokončený), **nejde do produkcie v tejto podobe** → prepisujeme čisto.

---

## 0. Rozhodnutia (fixné zadanie)
- **Stripe sa už NEPOUŽÍVA** — odstrániť všetky Stripe vetvy; jediný procesor = **Mollie**.
- **Rozsah: zatiaľ len SK + CZ** (≈80 % používateľov), **len v mobilnej aplikácii** (nie web). Ostatné krajiny neskôr.
- **Model = „podpor nás kúpou"**: používateľ vyberie produkt (obraz, tričko…), **zvolí sumu podpory** (10 / 25 / 50 / …), doplní **poštovné** + **adresu doručenia**, zaplatí, odošle.
- Treba: **faktúra**, **stavový proces objednávky**, **výstup do účtovníctva**.

---

## 1. Súčasný stav (čo už v kóde je — WIP)
Plný, ale nedokončený e-shop:
- **Frontend (web):** `[locale]/shop`, `shop/[slug]`, `cart` (localStorage), `checkout`.
- **Produkty:** `products` (JSONB `name`/`description` sk/en/cz/es, `price`, `stock`, `images`, `category`).
- **Objednávky:** `orders` + `order_items` (snapshot produktu); statusy `pending|paid|processing|shipped|completed|cancelled`.
- **Platby:** Stripe (produkty) **+** Mollie (dary/predplatné) **+** bankový prevod (`/admin/shop/banka`).
- **Doprava:** `shipping_zones` (8 zón, free-threshold).
- **Admin:** produkty, objednávky, subscriptions, donations, email šablóny, doprava, banka.
- **Emaily:** `email_templates` (multijazyčné) + `src/lib/email-sender.ts`.

### Tech-dlh / riziká (prečo prepísať)
- 🔴 **RLS vypnuté** na `orders`/`order_items` + `products` (`fix-orders-rls.sql`, `disable-products-rls.sql`).
- 🔴 **CAPTCHA vypnutá** na všetkých checkoutoch (TODO).
- 🔴 **Stock sa neodpočítava** po zaplatení → predaj nad sklad.
- 🟠 **Stripe** vetvy navyše (odstrániť).
- 🟠 **Košík len localStorage** (žiadne cross-device, abandoned-cart).
- 🟠 Chýba `/api/shipping/calculate`; RLS kontroluje `raw_user_meta_data->>'role'` (zvyšok kódu `users.role`).

---

## 2. Nová zjednodušená vízia (flow v appke)
```
1. „Chcem podporiť kúpou"  →  zoznam produktov (obraz, tričko, …)
2. Klik na produkt          →  detail + galéria
3. Výber sumy podpory        →  10 / 25 / 50 / … (prednastavené + voliteľne vlastná)
4. Poštovné                  →  podľa krajiny/zóny (alebo fix SK)
5. Adresa doručenia          →  meno, ulica, mesto, PSČ, krajina, telefón, e-mail
6. Zaplatiť                  →  IN-APP (Apple Pay / Google Pay / karta) — viď §3
7. Potvrdenie + faktúra      →  e-mail + v appke
```
Produkt je **„poďakovanie" za podporu** — cena = zvolená suma (variabilná, s odporúčanými stupňami).

---

## 3. 🔑 KRITICKÉ: platby — in-app vs. presmerovanie

### Problém
Súčasný tok presmeruje používateľa z appky na **Mollie hosted checkout (web)**. Odchod z appky na web = **veľký drop-off** (strata kontextu, dôvery, žiadna biometria, problém s návratom). Áno — toto je **pravdepodobná príčina**, že ľudia „dar nedokončia".

### Kľúčový fakt
**Apple Pay / Google Pay nie sú samostatný procesor — sú to platobné METÓDY.** Mollie ich vie spracovať. Takže to **nie je „Mollie ALEBO Apple Pay"** — ostávame na Mollie a pridáme Apple/Google Pay ako spôsob platby.

### Politika App Store / Google Play (dôležité)
- **Fyzický tovar** (obrazy, tričká — posielajú sa poštou) a **dary** sa platia **EXTERNÝM** procesorom (karta / Apple Pay / Google Pay / Mollie) — **Apple IAP sa NESMIE/nemusí použiť** (IAP je len pre digitálny obsah konzumovaný v appke).
- → **žiadnych 30 % Apple/Google**, a náš model je v súlade.

### Odporúčanie (2 stupne)
**Stupeň 1 — rýchla výhra (málo práce, veľký efekt):**
Mollie hosted checkout otvoriť v **in-app prehliadači** (iOS `ASWebAuthenticationSession`, Android `Chrome Custom Tabs`) + návrat cez **deep link**, namiesto `LaunchMode.externalApplication`. Používateľ neopustí appku, vráti sa automaticky. Výrazne zníži drop-off.

**Stupeň 2 — cieľový stav (najlepšia konverzia):**
**Natívny Apple Pay / Google Pay payment sheet priamo v appke** → token → backend → Mollie payment (`method: applepay` / Google Pay token). Jeden dotyk + biometria, **bez webu**. Karta ostáva ako fallback cez Stupeň 1.
- iOS: Apple Pay (Mollie má priame API pre `applePayPaymentToken`).
- Android: Google Pay cez Mollie (overiť presnú API cestu).
- Flutter: plugin `pay` (Apple/Google Pay sheet).

### Prečo zostať na Mollie (a nie „len Apple/Google Pay")
Apple/Google Pay potrebujú **procesor v pozadí** tak či tak. Mollie nám dáva **jedno miesto** pre: vyúčtovanie, refundy, **faktúry/účtovníctvo**, dary aj predplatné. Apple/Google Pay len **zlepší UX vstupu** do Mollie.

### Setup, ktorý treba (admin úloha)
- Apple Pay: Merchant ID, doménová verifikácia, payment processing certifikát prepojený s Mollie.
- Google Pay: merchant config v Mollie.
- Deep-link schéma pre návrat z platby.

### Odporúčanie navyše
Pridať **analytiku lievika** (Umami: `checkout_started` → `checkout_paid`), aby sme **zmerali**, či drop-off spôsobuje redirect (a potvrdili efekt zmeny).

### Mena
**E-shop (SK + CZ, app-only): EUR-only na štart.**
- CZ je mimo eurozóny (**CZK**), ale na štart **CZ platí v EUR** (karta/Apple/Google Pay to zvládnu; OZ účtuje v EUR → jednoduché faktúry).
- **CZK** = reálne rozšírenie čoskoro (CZ je veľký trh), ale pridáva FX + cenník v CZK + faktúry v CZK. Rozhodnúť podľa podielu CZ objednávok.
- Mýtus: Apple/Google Pay menu neukazujú automaticky — určuje ju **obchodník** v platbe.

**Dary (`donation_screen`) — SAMOSTATNÁ téma (nie tento e-shop):**
- Dary už chodia aj zo zahraničia (v EUR) → **EUR nie je blocker**.
- Lokálna mena u darov je **konverzná optimalizácia** s rovnakou účtovnou réžiou (FX, EUR settlement, kurz ku dňu).
- Odporúčanie: **najprv zmerať** objem darov per krajina/mena; ak je reálny non-EUR objem (USD/GBP/CZK), pridať **fixné stupne per mena** cez Mollie. Lacný medzikrok: zobraziť „≈ $27" pri EUR sume, ale **účtovať v EUR**. → patrí k `DONATION_TIERS_STRATEGY.md`.

---

## 4. Návrh dátového modelu (zjednodušený, Mollie-only)
- **products**: `id`, `slug`, `name`(JSONB), `description`(JSONB), `images[]`, `category`, `suggested_amounts`(int[] napr. {10,25,50}), `min_amount`(decimal), `is_active`, `stock`(voliteľné), `weight_g`(pre poštovné). _Cena nie je fixná — určí ju zvolená suma._
- **orders**: `id`, `order_number`(sekv.), `customer_email`, `shipping_address`(JSONB), `support_amount`, `shipping_cost`, `shipping_zone`, `total`, `status`, `payment_provider`('mollie'), `mollie_payment_id`, `invoice_number`, `invoice_url`, `tracking_number`, `created_at`, `paid_at`, `notes`.
- **order_items**: `order_id`, `product_id`, `product_snapshot`(JSONB), `amount`, `quantity`.
- (Ponechať `shipping_zones`, `email_templates`, `email_logs`.)
- **RLS poriadne:** produkty public-read (active), objednávky service-role/admin (žiadne `disable rls`).

---

## 5. Životný cyklus objednávky
```
objednávka (pending)  →  zaplatené (paid)  →  zabalené (packing)
       →  odoslané (shipped + tracking)  →  prevzaté (delivered)
       [vetvy: cancelled, refunded]
```
- Prechod statusov v admine (`/admin/shop/orders`); pri každom prechode **e-mail** zákazníkovi (šablóny už máme).
- `paid` nastaví **webhook Mollie** (nie klient) + vtedy: odpočítať stock, vygenerovať faktúru, poslať potvrdenie.

---

## 6. Faktúra
- Generovať **PDF faktúru** pri prechode na `paid`.
- Číslovanie: sekvenčné (napr. `2026/0001`).
- Údaje predajcu: **OZ lectio.one**, IČO `55971521`, DIČ, D-U-N-S®, adresa (Žilina) — máme z privacy.
- Položky: produkt + suma podpory + poštovné + spolu. Kupujúci: meno + adresa.
- Uložiť (`invoice_url` v Storage) + priložiť do potvrdzovacieho e-mailu + dostupné v appke.

---

## 7. Výstup do účtovníctva
- **ROZSAH: len e-shop objednávky.** Dary sa NErobia týmto exportom — idú z **bankového výpisu** a rieši ich účtovníčka priamo. (Dary = bez DPH.)
- **DPH per produkt:** každý produkt má príznak **`subject_to_vat`** (checkbox v admine). Niektoré produkty predávame s DPH 23%, iné bez. Export podľa toho vygeneruje 1 alebo 2 riadky na položku/doklad.
- **Mesačný export** zaplatených objednávok do CSV pre účtovnú firmu.
- **Formát potvrdený účtovníčkou** (vzory: `mobile/doc_xlsx/Lectio_one_vzor_CSV_OF_dary{,_s_DPH}.xlsx`):
  - CSV, oddeľovač **`;`**, kódovanie **UTF-8 s BOM**, desatinná čiarka **`,`** (napr. `20,00`).
  - Dátumy **`DD.MM.YYYY`** (s nulami: `01.06.2026`). Mena **EUR**.
  - **29 stĺpcov, presné názvy a poradie (nemeniť):** `cislo;datum;popis;suma;md;d;c_fa;datum_spl;partner;cbico;c_uct;str_md;str_d;zdroj_md;zdroj_d;proj_md;proj_d;dat_vyst;dat_dph;kod;mena;kv_cast;kv_cs;kv_mnoz;kv_mj;kv_cislo;kv_cislo_o;kv_oprava;icdph_sv;s_dph`
  - `cislo` = `OF 2026-001` (číslo dokladu), `c_fa` = `2026001` (číslo faktúry, číselné), `partner` = darca (anonym → „Anonymny darca").
  - Účty: **MD 311 / D 663** (dar). `kv_mnoz` = `0`. `datum_spl` = `dat_vyst` = `dat_dph` = dátum.
  - **Bez DPH** (dary OZ): jeden riadok, `kod` prázdny. (vzor `..._dary.xlsx`)
  - **S DPH 23%** (predaj tovaru e-shop): **dva riadky** na doklad — základ `MD 311/D 663` `kod=3`, DPH `MD 311/D 343` `kod=D`. Kód DPH potvrdiť podľa číselníka v účt. softvéri. (vzor `..._s_DPH.xlsx`)
- **Číslovanie OF**: sekvenčné per rok (`OF {rok}-{poradie}`) — generuje export/systém.
- Tlačidlo v admine (výber obdobia) + možný auto-export (cron).

---

## 8. ❓ Otvorené otázky (na rozhodnutie pred implementáciou)
1. **Predaj tovaru vs. dar + symbolický darček?** Ovplyvňuje **faktúru, DPH, účtovníctvo** (OZ = nezisk.). → konzultovať s účtovníčkou. _Najpravdepodobnejšie: predaj tovaru s faktúrou._
2. **Variabilná suma:** cena = zvolená suma podpory? Minimálna suma per produkt? Odporúčané stupne per produkt alebo globálne?
3. **Google Pay cez Mollie** — overiť presnú API podporu (Apple Pay je istý).
4. **Poštovné** — rozhodnuté: len **SK + CZ** (2 sadzby), 8 zón netreba. _(Mena pre CZ — viď §3.)_
5. **Apple Pay setup** — Merchant ID + certifikáty (treba spraviť pred Stupňom 2).

---

## 9. Navrhovaný postup (fázy)
- **P0 — Vyčistenie:** odstrániť Stripe, dorobiť čistý dátový model + RLS poriadne, odstrániť `disable-rls` hacky.
- **P1 — In-app platba:** Stupeň 1 (in-app browser + deep link) → hneskôr Stupeň 2 (Apple/Google Pay natívne). Webhook = jediný zdroj `paid`. Analytika lievika.
- **P2 — Objednávka & faktúra:** lifecycle statusy + e-maily + PDF faktúra + odpočet stock.
- **P3 — Účtovníctvo:** mesačný export.

---

## Súvisiace
- `DONATION_TIERS_STRATEGY.md` (stratégia darov/predplatného)
- `PRODUCTION_CHECKLIST.md` → roadmap „e-shop — prepracovať"
