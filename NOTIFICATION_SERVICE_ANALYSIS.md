# 🔔 Notifikačný systém — Analýza a návrh vylepšení

> Dátum: 12. februára 2026

---

## 📊 Aktuálny stav

### Architektúra

```
┌──────────────────────────────────────────────────────────────┐
│                     ADMIN UI (Next.js)                       │
│  /admin/notifications/new → POST /api/admin/send-notification│
│  - okamžité odoslanie ALEBO scheduled_at (uloženie do DB)    │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                  CRON (každú minútu)                          │
│  GET /api/cron/send-scheduled-notifications                  │
│  → scheduled_notifications WHERE status='pending'            │
│    AND scheduled_at <= NOW()                                 │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              FCM Multicast (firebase-admin.ts)                │
│  messaging.sendEachForMulticast(tokens[])                    │
│  - token-based (NIE FCM Topics)                              │
│  - filtrované podľa locale + user_notification_preferences   │
│  - opt-out model (defaultne = zapnuté)                       │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                    FLUTTER APPKA                              │
│                                                              │
│  ┌─────────────────────┐  ┌────────────────────────────────┐│
│  │   FcmService        │  │  LocalNotificationsService     ││
│  │   (push z backendu) │  │  (lokálne plánované)           ││
│  │                     │  │                                ││
│  │  • foreground show  │  │  • denné lectio (7 dní)        ││
│  │  • background show  │  │  • prayer reminder (7 dní)     ││
│  │  • token register   │  │  • welcome (1x po 3 dňoch)    ││
│  │  • onTap → navigate │  │  • timezone detection          ││
│  └─────────────────────┘  └────────────────────────────────┘│
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │  NotificationController — deep-link navigácia           ││
│  │  10+ screens: lectio, profile, settings, rosary...      ││
│  └─────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

---

## 🔍 Druhy notifikácií

### 1. Lokálne notifikácie (Flutter — `LocalNotificationsService`)

| Typ | Kedy | Čas | Kanál | Opakovanie |
|-----|------|-----|-------|------------|
| **Denné Lectio** | Každý deň | 09:00 (hardcoded) | `daily_lectio_channel` (MAX) | 7 dní dopredu, potom refresh |
| **Prayer Reminder** | Podľa user | User si vyberie čas | `prayer_reminder_channel` (HIGH) | 7 dní dopredu |
| **Welcome** | 3 dni po registrácii | 10:00 (hardcoded) | `welcome_channel` (HIGH) | Jednorazová |

**Ako to funguje:**
- Pri každom otvorení appky sa volá `refreshCacheIfNeeded()` → naplánuje ďalších 7 dní
- Stiahne skutočný text lectia zo Supabase pre každý deň (liturgical_calendar + lectio_sources)
- Používa `zonedSchedule()` s detekovaným timezone

### 2. Push notifikácie (Backend → FCM → Flutter)

| Typ | Odosielateľ | Cieľ | Filtrovanie |
|-----|-------------|------|-------------|
| **Admin broadcast** | Admin cez UI | Všetci s locale | Topic + preferences |
| **Scheduled** | Admin naplánuje | Podľa `scheduled_at` | Topic + locale + preferences |
| **API trigger** | Server-to-server | Podľa `userIds` / `topicId` | Token + preferences |

**Dostupné topics v DB:** `daily-readings`, `special-occasions`, `prayers`, `rosary`, `events`

---

## 🚨 Identifikované problémy

### 1. KRITICKÝ: Denné Lectio je len lokálne → prestane po 7 dňoch

**Problém:** Ak používateľ neotvorí appku 7+ dní, lokálne notifikácie sa minú a žiadne nové sa nenaplánujú. Používateľ prestane dostávať denné lectio notifikácie.

**Prečo:** `LocalNotificationsService` plánuje len 7 dní dopredu. Refresh nastáva len pri `initState` a `didChangeAppLifecycleState(resumed)`. Ak appka nie je otvorená, refresh sa nevolá.

**Dopad:** Neaktívni používatelia = presne tí, ktorých chceme re-engagovať, nedostanú notifikáciu.

### 2. STREDNÝ: Duplicitná FlutterLocalNotificationsPlugin inštancia

**Problém:** Dve nezávislé inštancie pluginu:

| # | Súbor | Inicializovaný? | Použitie |
|---|-------|-----------------|----------|
| 1 | `fcm_service.dart` L17 (globálna) | **NIE** | Background handler (iOS) |
| 2 | `local_notifications_service.dart` L28 (singleton) | **ÁNO** | Všetko ostatné |

Foreground FCM handler už používa singleton (`LocalNotificationsService.instance.plugin`), ale background handler stále používa neinicializovanú globálnu inštanciu.

### 3. STREDNÝ: Žiadne timezone na backende

**Problém:** Backend nepozná timezone používateľa. `scheduled_at` sa porovnáva s UTC serverového času. Ak admin naplánuje notifikáciu na "8:00 ráno", odošle sa o 8:00 UTC — čo je 9:00 CET ale 3:00 EST.

**Dopad:** Používatelia v iných timezone dostanú notifikáciu v nevhodnom čase.

### 4. NÍZKY: Hardcoded časy

| Konštanta | Hodnota | Súbor |
|-----------|---------|-------|
| `dailyLectioHour` | 9 | `audio_constants.dart` |
| `dailyLectioMinute` | 0 | `audio_constants.dart` |
| `welcomeNotificationHour` | 10 | `audio_constants.dart` |
| `Europe/Bratislava` fallback | hardcoded | `main.dart` L88 |

Denné lectio je vždy o 9:00 — nie je konfigurovateľné v nastaveniach.

### 5. NÍZKY: Žiadne čistenie neplatných tokenov

Backend loguje zlyhané tokeny do konzoly ale neoznačí ich ako `is_active = false`. Postupne sa hromadia mŕtve tokeny → zbytočné FCM requesty.

---

## ✅ Návrh riešenia

### Fáza 1: Duplicitná plugin inštancia (rýchly fix)

**Súbor:** `mobile/lib/services/fcm_service.dart`

**Problém:** Globálna `FlutterLocalNotificationsPlugin` na L17 nie je nikdy inicializovaná.

**Riešenie:**
1. Odstrániť globálnu premennú `flutterLocalNotificationsPlugin` (L17-18)
2. V background handleri `_showLocalNotification()` (L42) vytvoriť a inicializovať lokálnu inštanciu:
   ```dart
   @pragma('vm:entry-point')
   Future<void> _showLocalNotification(RemoteMessage message) async {
     final plugin = FlutterLocalNotificationsPlugin();
     await plugin.initialize(InitializationSettings(
       android: AndroidInitializationSettings('@mipmap/ic_launcher'),
       iOS: DarwinInitializationSettings(),
     ));
     await plugin.show(...);
   }
   ```
3. Foreground handler už používa `LocalNotificationsService.instance.plugin` → OK

---

### Fáza 2: Denné Lectio cez FCM push — HLAVNÁ ZMENA

**Cieľ:** Nahradiť lokálne 7-dňové plánovanie serverovým push cez FCM. Používateľ dostane notifikáciu aj keď neotvorí appku mesiace.

#### Celkový flow

```
┌─────────────────────────────────────────────────────────────────┐
│  VERCEL CRON (každú hodinu: 0 * * * *)                         │
│  GET /api/cron/send-daily-lectio                                │
│                                                                 │
│  1. Zisti aktuálnu UTC hodinu                                   │
│  2. Nájdi timezone skupiny kde je práve 8:00 ráno               │
│     (napr. UTC+1 → cron o 7:00 UTC, UTC-5 → cron o 13:00 UTC) │
│  3. Pre každý locale (sk, en, cs...):                           │
│     a) Stiahni dnes z liturgical_calendar → lectio_hlava        │
│     b) Stiahni z lectio_sources → actio_text + hlava            │
│     c) Filtruj tokeny: timezone_group + locale + daily-readings │
│     d) Pošli FCM multicast s deep linkom                        │
│  4. Zaloguj do daily_notification_log                           │
└──────────────────────────┬──────────────────────────────────────┘
                           │ FCM push
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  FLUTTER APPKA (aj keď je zatvorená/na pozadí)                  │
│                                                                 │
│  1. Systém zobrazí push notifikáciu:                            │
│     Titul: "Sv. Valentín — Mk 8,14-21"  (hlava)               │
│     Body:  "Zamysli sa: Ako dnes môžem..." (actio_text[:100])  │
│     Obrázok: (voliteľne z image_url)                           │
│                                                                 │
│  2. Používateľ klikne na notifikáciu                            │
│     ↓                                                           │
│  3. Appka sa otvorí → NotificationController                    │
│     ↓                                                           │
│  4. Zobrazí sa POPUP DIALÓG (štýl SiberianCMS):                │
│     ┌─────────────────────────────┐                             │
│     │      🕊️ [obrázok/ikona]     │                             │
│     │                             │                             │
│     │  Sv. Valentín — Mk 8,14-21 │                             │
│     │                             │                             │
│     │  Zamysli sa: Ako dnes       │                             │
│     │  môžem prejaviť lásku...    │                             │
│     │                             │                             │
│     │    ┌─────────────────┐      │                             │
│     │    │  Otvoriť Lectio │      │                             │
│     │    └─────────────────┘      │                             │
│     │                             │                             │
│     │        Zavrieť              │                             │
│     └─────────────────────────────┘                             │
│                                                                 │
│  5. Klik "Otvoriť Lectio" → LectioScreen(selectedDate: today) │
└─────────────────────────────────────────────────────────────────┘
```

---

#### A) Backend: Nový cron endpoint

**Nový súbor:** `backend/src/app/api/cron/send-daily-lectio/route.ts`

**Cron schedule:** `0 * * * *` (každú celú hodinu)

**Logika:**

```
1. AUTH: Overiť CRON_SECRET (Bearer token, rovnako ako send-scheduled-notifications)

2. TIMEZONE MATCHING:
   - Aktuálna UTC hodina = new Date().getUTCHours()
   - Cieľový lokálny čas = 8:00 (konfigurovateľné)
   - Hľadaj timezone skupiny kde: UTC_offset + UTC_hodina = 8
   - Príklad: o 7:00 UTC posielame pre UTC+1 (Europe/Bratislava, Europe/Prague)
   - Príklad: o 13:00 UTC posielame pre UTC-5 (America/New_York)
   - Query: SELECT DISTINCT timezone FROM user_fcm_tokens 
            WHERE is_active = true 
            AND EXTRACT(HOUR FROM NOW() AT TIME ZONE timezone) = 8

3. DEDUPLIKÁCIA:
   - Kontrola: SELECT * FROM daily_notification_log 
               WHERE date = today AND timezone_group = X AND locale_code = Y
   - Ak záznam existuje → skip (už odoslaný pre túto skupinu)

4. PRE KAŽDÝ LOCALE:
   a) Stiahni lectio dáta:
      SELECT lc.lectio_hlava, lc.celebration_title, lc.celebration_rank,
             ls.hlava, ls.actio_text, ls.reference
      FROM liturgical_calendar lc
      LEFT JOIN lectio_sources ls ON ls.hlava = lc.lectio_hlava 
                                  AND ls.lang = {locale}
      WHERE lc.datum = CURRENT_DATE AND lc.locale_code = {locale}

   b) Zostav notifikáciu:
      title: hlava alebo celebration_title (napr. "Sv. Valentín — Mk 8,14-21")
      body:  actio_text zostrihnutý na 120 znakov + "..."
      data:  { screen: "lectio", 
               screen_params: '{"date":"2026-02-12"}',
               type: "daily_lectio" }

   c) Filtruj tokeny:
      SELECT token FROM user_fcm_tokens uft
      WHERE uft.is_active = true
        AND uft.locale_code = {locale}
        AND uft.timezone IN ({matched_timezones})
        AND NOT EXISTS (
          SELECT 1 FROM user_notification_preferences unp
          WHERE unp.user_id = uft.user_id
            AND unp.topic_id = (SELECT id FROM notification_topics WHERE slug = 'daily-readings')
            AND unp.is_enabled = false
        )

   d) Pošli cez sendPushNotification(tokens, title, body, data)
   
   e) STALE TOKEN CLEANUP:
      Pre každý response[i] kde success = false:
        Ak error.code = 'messaging/registration-token-not-registered'
        → UPDATE user_fcm_tokens SET is_active = false WHERE token = tokens[i]

   f) Zaloguj:
      INSERT INTO daily_notification_log (date, locale_code, timezone_group, 
                                          tokens_sent, tokens_failed, lectio_title)

5. RESPONSE: { sent: X, failed: Y, timezones: [...], skipped: [...] }
```

**Vercel.json zmena:**
```json
{
  "path": "/api/cron/send-daily-lectio",
  "schedule": "0 * * * *"
}
```

---

#### B) Backend: Timezone v user_fcm_tokens

**SQL migrácia:** `backend/sql/add_timezone_to_fcm_tokens.sql`

```sql
ALTER TABLE user_fcm_tokens 
  ADD COLUMN IF NOT EXISTS timezone TEXT DEFAULT 'Europe/Bratislava';

-- Index pre rýchle groupovanie
CREATE INDEX IF NOT EXISTS idx_fcm_tokens_timezone 
  ON user_fcm_tokens(timezone, is_active, locale_code);
```

**API zmena:** `backend/src/app/api/user/fcm-tokens/route.ts`
- POST body prijíma nový field `timezone` (IANA formát)
- Upsert ukladá timezone pri registrácii/update tokenu

---

#### C) Backend: Stale token cleanup v firebase-admin.ts

**Existujúci súbor:** `backend/src/lib/firebase-admin.ts`

Po `sendEachForMulticast()` (L119) pridať:

```typescript
// Cleanup stale tokens
const staleTokens: string[] = [];
response.responses.forEach((resp, i) => {
  if (!resp.success && 
      resp.error?.code === 'messaging/registration-token-not-registered') {
    staleTokens.push(tokens[i]);
  }
});
if (staleTokens.length > 0) {
  await supabase
    .from('user_fcm_tokens')
    .update({ is_active: false })
    .in('token', staleTokens);
}
```

---

#### D) Flutter: Timezone pri registrácii tokenu

**Súbor:** `mobile/lib/services/fcm_service.dart` — metóda `_register()` (~L250)

Pridať timezone do upsert body:

```dart
// Získaj IANA timezone z LocalNotificationsService
final timezone = LocalNotificationsService.instance.currentTimezone;

await supabase.from('user_fcm_tokens').upsert({
  'user_id': userId,
  'token': token,
  'device_id': deviceId,
  'device_type': Platform.isIOS ? 'ios' : 'android',
  'app_version': appVersion,
  'locale_code': appLangCode,
  'is_active': true,
  'timezone': timezone,  // NOVÉ
});
```

---

#### E) Flutter: Popup dialóg po kliknutí na notifikáciu

**Súbor:** `mobile/lib/controllers/notification_controller.dart`

Aktuálny stav `_showRemoteNotificationDialog()` (L320-L438):
- Zobrazuje jednoduchý `AlertDialog` s titulkom, textom a tlačidlami
- Ak `screen` != null → tlačidlo "Otvoriť" naviguje cez `navigateToScreen()`

**Nový dizajn — SiberianCMS štýl popup:**

```
┌────────────────────────────────────────┐
│                                        │
│          🕊️  [Logo/Obrázok]            │  ← Kruhový obrázok alebo ikona
│                                        │     (image_url z FCM, alebo default)
│                                        │
│     Sv. Valentín — Mk 8,14-21         │  ← Titul (bold, 22px, center)
│                                        │
│     Zamysli sa: Ako dnes môžem         │  ← Body text (16px, center)
│     prejaviť svoju lásku blížnym...    │
│                                        │
│     ┌──────────────────────────┐       │
│     │    Otvoriť Lectio  →    │       │  ← Primárne tlačidlo
│     └──────────────────────────┘       │     (zaoblené, AppColors.primary)
│                                        │
│            Zavrieť                     │  ← Sekundárne (text only)
│                                        │
└────────────────────────────────────────┘
```

**Implementácia:**
- `Dialog` s `borderRadius: AppRadius.xxl` (30)
- Horný obrázok: `CachedNetworkImage` ak `image_url`, inak `assets/images/logo.png`
- Titul: `textTheme.titleLarge`, `textAlign: center`, `fontWeight: bold`
- Body: `textTheme.bodyMedium`, `textAlign: center`, max 4 riadky
- Primárne tlačidlo: `ElevatedButton` s `borderRadius: 30`, šírka 60%
- Sekundárne: `TextButton` "Zavrieť"

**Navigácia po kliknutí "Otvoriť Lectio":**
```dart
// Remote notification — už funguje cez navigateToScreen():
case 'lectio':
    final dateStr = params?['date'] as String?;
    final date = dateStr != null ? DateTime.tryParse(dateStr) : null;
    targetScreen = LectioScreen(selectedDate: date ?? DateTime.now());
    break;
```

Aktuálne `navigateToScreen('lectio')` už parsuje `date` z `screen_params` a otvára `LectioScreen(selectedDate: date)` — toto zostáva.

---

#### F) Flutter: Odstrániť lokálne denné lectio

**Súbory na úpravu:**

| Súbor | Zmena |
|-------|-------|
| `local_notifications_service.dart` | Odstrániť `_scheduleDailyLectioNotifications()`, `_downloadAndCacheLectioData()`, súvisiace cache kľúče. Ponechať prayer reminder + welcome |
| `audio_constants.dart` | Odstrániť `dailyLectioHour`, `dailyLectioMinute`, `scheduleDaysAhead`, `dailyLectioBaseId` |
| `notification_settings_screen.dart` | Daily lectio toggle: namiesto lokálneho `enableDailyLectio` → zapne/vypne FCM topic `daily-readings` cez `FcmService.updateTopicPreference()` |
| `main.dart` | Odstrániť hardcoded `tz.setLocalLocation('Europe/Bratislava')` — timezone detekcia zostáva v `LocalNotificationsService.initialize()` |

**Čo zostáva lokálne:**
- ✅ Prayer reminder (osobný čas, netreba server)
- ✅ Welcome notification (3 dni po registrácii)
- ❌ ~~Denné lectio~~ → presúva sa na server

---

#### G) Dátový tok — od Supabase po popup

```
liturgical_calendar (datum=dnes, locale_code=sk)
    │
    │ lectio_hlava = "Mk 8,14-21"
    ▼
lectio_sources (hlava="Mk 8,14-21", lang=sk, rok=N)
    │
    │ hlava, actio_text, celebration_title
    ▼
Backend cron zostaví FCM payload:
    {
      notification: {
        title: "Sv. Valentín — Mk 8,14-21",
        body: "Zamysli sa: Ako dnes môžem prejaviť..."
      },
      data: {
        screen: "lectio",
        screen_params: '{"date":"2026-02-14"}',
        type: "daily_lectio",
        topic_id: "uuid-daily-readings",
        locale: "sk",
        image_url: "https://lectio.one/images/lectio-daily.png"
      }
    }
    │
    │ FCM multicast
    ▼
Flutter: systémová notifikácia v tray
    │
    │ tap
    ▼
NotificationController.handleRemoteNotificationTap(message)
    │
    │ 800ms delay
    ▼
_showRemoteNotificationDialog(context, message)
    │
    │ Popup dialóg (SiberianCMS štýl)
    │ - obrázok/logo
    │ - titul: "Sv. Valentín — Mk 8,14-21"
    │ - body: "Zamysli sa: Ako dnes..."
    │ - tlačidlo "Otvoriť Lectio"
    ▼
navigateToScreen("lectio", screenParams: {"date":"2026-02-14"})
    │
    ▼
LectioScreen(selectedDate: DateTime(2026, 2, 14))
```

---

### Fáza 3: Timezone-aware scheduled notifikácie (budúcnosť)

**Backend:**
- Pridať `is_timezone_aware` + `target_local_time` do `scheduled_notifications`
- Upraviť cron: ak `is_timezone_aware = true` → groupovať podľa timezone, posielať keď je u usera `target_local_time`
- Admin UI: "Odošli o 9:00 lokálneho času používateľa" checkbox

### Fáza 4: Konfigurovateľný čas denného lectia (budúcnosť)

**Flutter:**
- Pridať time picker pre denné lectio (ako prayer reminder)
- Uložiť `preferred_lectio_time` do `user_fcm_tokens`
- Backend cron groupuje podľa timezone + preferred time

---

## 💡 Návrhy pre modernú duchovnú aplikáciu

### Engagement notifikácie

| Typ | Trigger | Príklad |
|-----|---------|---------|
| **Re-engagement** | Neaktivita 3, 7, 14 dní | "Dnes ťa čaká krásne čítanie z Jána 3:16 🙏" |
| **Streak reminder** | Používateľ má aktívny streak | "Máš 7-dňový streak! Pokračuj dnes." |
| **Streak lost** | Streak prerušený | "Tvoj 5-dňový streak sa prerušil. Začni odznova!" |
| **Milestone** | Streak 30, 100, 365 dní | "🎉 30 dní nepretržitej modlitby!" |

### Liturgické notifikácie

| Typ | Kedy | Príklad |
|-----|------|---------|
| **Svätec dňa** | Ráno | "Dnes si pripomíname sv. Valentína ❤️" |
| **Liturgické obdobie** | Začiatok obdobia | "Dnes začína Pôstne obdobie. Nech ťa sprevádza milosť." |
| **Veľké sviatky** | Vigília + deň | "Zajtra je Vianoce! Priprav sa na slávenie." |
| **Pôstne zamyslenie** | Počas pôstu | Špeciálne denné zamyslenie pre pôstne obdobie |

### Komunitné notifikácie

| Typ | Trigger | Príklad |
|-----|---------|---------|
| **Prayer Wall** | Reakcia na úmysel | "3 ľudia sa za teba modlia 🙏" |
| **Nový úmysel** | Priatelia v komunite | "Tvoj priateľ pridala nový modlitebný úmysel" |
| **Spoločná modlitba** | Admin plánuje | "Dnes o 20:00 spoločný ruženec za mier 📿" |

### Personalizované notifikácie

| Typ | Podmienka | Príklad |
|-----|-----------|---------|
| **Obľúbené čítania** | User čítal Jána najčastejšie | "Nové zamyslenie z Evanjelia podľa Jána" |
| **Dokončenie série** | Adorácie 3/7 hotové | "Chýbajú ti ešte 4 adorácie. Pokračuj!" |
| **Ranný/večerný režim** | Podľa aktivity usera | Ak otvára appku večer → posielať notif o 20:00 |
| **Quiet hours** | Nočný pokoj | Neposielať medzi 22:00 – 7:00 lokálneho času |

### Smart notifikácie (AI-powered)

| Typ | Popis |
|-----|-------|
| **Denné slovo** | AI vyberie krátky verš zo Svätého Písma relevantný k liturgickému dňu |
| **Personalizovaný push text** | Namiesto generického "Denné Lectio je pripravené" → kontext z dnešného čítania |
| **Optimal send time** | ML model zistí kedy user najčastejšie otvára appku → posielať v ten čas |

---

## 🗄️ DB schéma — navrhované zmeny

### Existujúce tabuľky

```sql
-- user_fcm_tokens — PRIDAŤ:
ALTER TABLE user_fcm_tokens 
  ADD COLUMN timezone TEXT DEFAULT 'Europe/Bratislava',
  ADD COLUMN preferred_lectio_time TIME DEFAULT '09:00',
  ADD COLUMN last_active_at TIMESTAMPTZ DEFAULT NOW();

-- scheduled_notifications — PRIDAŤ:
ALTER TABLE scheduled_notifications 
  ADD COLUMN is_timezone_aware BOOLEAN DEFAULT false,
  ADD COLUMN target_local_time TIME;
```

### Nové tabuľky

```sql
-- Automatické denné notifikácie
CREATE TABLE daily_notification_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date DATE NOT NULL,
  locale_code TEXT NOT NULL,
  timezone_group TEXT NOT NULL,
  tokens_sent INTEGER DEFAULT 0,
  tokens_failed INTEGER DEFAULT 0,
  lectio_title TEXT,
  sent_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(date, locale_code, timezone_group)
);

-- User engagement tracking
CREATE TABLE user_engagement (
  user_id UUID PRIMARY KEY REFERENCES auth.users(id),
  last_app_open TIMESTAMPTZ,
  current_streak INTEGER DEFAULT 0,
  longest_streak INTEGER DEFAULT 0,
  last_streak_date DATE,
  total_lectio_read INTEGER DEFAULT 0,
  re_engagement_sent_at TIMESTAMPTZ
);
```

---

## 📁 Súbory na úpravu

### Flutter (mobile/lib/)

| Súbor | Zmena |
|-------|-------|
| `services/fcm_service.dart` | Odstrániť globálnu `FlutterLocalNotificationsPlugin`, inicializovať v background handler |
| `services/local_notifications_service.dart` | Odstrániť `_scheduleDailyLectioNotifications()`, ponechať prayer reminder |
| `shared/audio_constants.dart` | Odstrániť `dailyLectioHour/Minute` konštanty |
| `screens/notification_settings_screen.dart` | Daily lectio toggle → ovláda FCM topic `daily-readings` namiesto lokálneho plánovania |
| `main.dart` | Odstrániť hardcoded `Europe/Bratislava`, posielať timezone pri token registrácii |

### Backend (backend/src/)

| Súbor | Zmena |
|-------|-------|
| `app/api/cron/send-daily-lectio/route.ts` | **NOVÝ** — cron pre denné lectio push |
| `app/api/user/fcm-tokens/route.ts` | Prijímať `timezone`, `preferred_lectio_time` |
| `lib/firebase-admin.ts` | Cleanup neplatných tokenov po odoslaní |
| `app/api/cron/send-scheduled-notifications/route.ts` | Timezone-aware porovnanie |
| `vercel.json` | Pridať cron pre daily lectio |

---

## 📋 Implementačný plán

### Fáza 1 — Duplicitná inštancia (rýchly fix, ~30 min) ✅
- [x] Odstrániť globálnu `flutterLocalNotificationsPlugin` z `fcm_service.dart` L17
- [x] Background handler: inicializovať + používať lokálnu inštanciu v isolate
- [ ] Otestovať foreground + background FCM notifikácie

### Fáza 2 — Denné lectio cez FCM push (hlavná zmena) ✅

**Backend (5 úloh):**
- [x] SQL migrácia: `ALTER TABLE user_fcm_tokens ADD COLUMN timezone` + `daily_notification_log` tabuľka
- [x] API: `fcm-tokens/route.ts` — prijímať a ukladať `timezone` pri upsert
- [x] Nový cron: `send-daily-lectio/route.ts` — každú hodinu, timezone-aware, stiahne lectio, pošle FCM
- [x] Stale token cleanup v `firebase-admin.ts` — `sendPushNotificationWithCleanup()` deaktivuje neplatné tokeny
- [x] `vercel.json` — pridať `{ "path": "/api/cron/send-daily-lectio", "schedule": "0 * * * *" }`

**Flutter (4 úlohy):**
- [x] `fcm_service.dart` — posielať `timezone` pri `_register()` tokenu
- [x] `notification_controller.dart` — nový popup dialóg (SiberianCMS štýl: obrázok + zaoblený dizajn)
- [x] `local_notifications_service.dart` — odstrániť `_scheduleDailyLectioNotifications()` a súvisiaci cache
- [x] `notification_settings_screen.dart` — daily lectio toggle → FCM topic `daily-readings`

**Cleanup:**
- [x] `audio_constants.dart` — odstrániť `dailyLectioHour/Minute/cacheValidityHours` (scheduleDaysAhead ponechané — používa prayer reminder)
- [x] `main.dart` — odstrániť hardcoded `Europe/Bratislava`

### Fáza 3 — Timezone-aware scheduled (budúcnosť, v10.1+)
- [x] Backend: `is_timezone_aware` + `target_local_time` v scheduled_notifications
- [x] Backend: admin UI "odošli v lokálnom čase"
- [x] Flutter: konfigurovateľný čas denného lectia (time picker)

### Fáza 4 — Smart notifikácie (v10.1+)
- [ ] Re-engagement notifikácie (3, 7, 14 dní neaktivity)
- [ ] Streak tracking + reminders
- [ ] Liturgické notifikácie (svätec dňa, obdobie)
- [ ] Personalizovaný text z denného čítania
- [ ] Quiet hours (neposielať v noci)
- [ ] Prayer Wall reakcie
