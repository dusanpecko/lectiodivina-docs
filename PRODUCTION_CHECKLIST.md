# 📱 Lectio Divina - Produkčný Checklist

## 🎯 Metodológia kontroly

Pre každú obrazovku/komponent:
1. **Jazyková kontrola** (ChatGPT) - gramatika, preklepy, konzistencia prekladov SK/EN/ES
2. **Kódová kontrola** (Claude Opus) - best practices, error handling, memory leaks, async/await
3. **Vizuálna kontrola** (Claude Opus + manuálne) - UI/UX, responzivita, dark mode

**Poznámka:** Dokumentáciu a návod k jednotlivým stránkam nevytvárame - zameriavame sa čisto na kontrolu kvality pred produkciou.

---

## 📋 TODO LIST

### 🏠 1. HLAVNÉ OBRAZOVKY

| Obrazovka | Jazyk | Kód | Vizuál | Poznámky |
|-----------|:-----:|:---:|:------:|----------|
| main.dart | - | ✅ | - | Lokalizácia, const syntax, appLogger |
| Home Screen | - | ✅ | ⬜ | Lokalizácia dní, tr() opravené |
| Lectio Screen | - | ✅ | ⬜ | 6 sekcií lokalizované, DND dočasne deaktivované |
| Settings Screen | - | ✅ | ⬜ | DND sekcia ODSTRÁNENÁ (dočasne) |
| Profile Screen | ✅ | ✅ | ⬜ | Tier badges + subscription.tier lokalizované |
| About Screen | ✅ | ✅ | ⬜ | Kompletne lokalizované, žiadne problémy |
| Intention Submit Screen | ✅ | ✅ | ⬜ | Pridané 3 chýbajúce error hlášky |
| Auth Screen | ⬜ | ⬜ | ⬜ | 🐛 Apple Sign-In bug |

---

### ⚠️ 1.2 DOČASNE DEAKTIVOVANÉ FUNKCIE

| Funkcia | Dôvod | Kde obnoviť |
|---------|-------|-------------|
| DND (Nerušiť) | iOS: Apple nepovoľuje priamy prístup k Focus API. Android: Vyžaduje špeciálne povolenia. Nefunguje spoľahlivo. | settings_screen.dart, lectio_screen.dart, lectio_audio_service.dart |
| Background Play nastavenia | Nedokončené, TODO | settings_screen.dart |

---

### 🔧 1.3 SERVICES (Kód)

| Service | Stav | Poznámky |
|---------|:----:|----------|
| app_logger.dart | ✅ | NOVÝ - centrálny logger s release filtrom |
| fcm_service.dart | ✅ | appLogger |
| local_notifications_service.dart | ✅ | appLogger |
| notification_api.dart | ✅ | appLogger |
| lectio_audio_service.dart | ✅ | appLogger |
| prayer_focus_service.dart | ✅ | appLogger |
| credentials_service.dart | ✅ | appLogger |
| do_not_disturb_service.dart | ✅ | appLogger |
| notification_settings_screen.dart | ✅ | appLogger |

---

### 🙏 2. MODULY

| Modul | Jazyk | Kód | Vizuál | Poznámky |
|-------|:-----:|:---:|:------:|----------|
| Rosary - Zoznam | ⬜ | ⬜ | ⬜ | |
| Rosary - Detail | ⬜ | ⬜ | ⬜ | |
| Prayer Intentions | ⬜ | ⬜ | ⬜ | |
| Spiritual Exercises - Zoznam | ⬜ | ⬜ | ⬜ | |
| Spiritual Exercises - Detail | ⬜ | ⬜ | ⬜ | |
| Spiritual Exercises - Registrácia | ⬜ | ⬜ | ⬜ | |
| Donation Screen | ⬜ | ⬜ | ⬜ | |
| News - Zoznam | ⬜ | ⬜ | ⬜ | |
| News - Detail | ⬜ | ⬜ | ⬜ | |
| Notes | ⬜ | ⬜ | ⬜ | |
| Intro/Onboarding | ⬜ | ⬜ | ⬜ | |

---

### 🎨 3. KOMPONENTY

| Komponent | Kód | Vizuál | Poznámky |
|-----------|:---:|:------:|----------|
| Audio Player Card | ⬜ | ⬜ | |
| Module Button | ⬜ | ⬜ | |
| Loading/Error widgets | ⬜ | ⬜ | |
| Navigation (FAB menu) | ⬜ | ⬜ | |

---

### 🌐 4. LOKALIZÁCIA

| Súbor | Kompletnosť | Gramatika | Poznámky |
|-------|:-----------:|:---------:|----------|
| sk.json | ⬜ | ⬜ | |
| en.json | ⬜ | ⬜ | |
| es.json | ⬜ | ⬜ | |
| Porovnanie kľúčov | ⬜ | - | Žiadne chýbajúce |

---

### 📱 5. PLATFORMY

| Položka | iOS | Android | Poznámky |
|---------|:---:|:-------:|----------|
| Permissions | ⬜ | ⬜ | |
| Push notifications | ⬜ | ⬜ | |
| Background audio | ⬜ | ⬜ | |
| Deep links | ⬜ | ⬜ | |

---

### ⚙️ 6. BUILD & RELEASE

| Položka | Stav | Poznámky |
|---------|:----:|----------|
| `flutter analyze` - 0 errors | ⬜ | |
| `flutter test` - pass | ⬜ | |
| Version number aktualizovaný | ⬜ | |
| Release build iOS | ⬜ | |
| Release build Android | ⬜ | |

---

### 🧪 7. TESTOVANIE

| Test | iOS | Android | Poznámky |
|------|:---:|:-------:|----------|
| Fresh install | ⬜ | ⬜ | |
| Upgrade z predošlej verzie | ⬜ | ⬜ | |
| Offline mode | ⬜ | ⬜ | |
| Všetky jazyky (SK/EN/ES) | ⬜ | ⬜ | |
| Light/Dark mode | ⬜ | ⬜ | |

---

## 📝 ZNÁME PROBLÉMY

| Problém | Priorita | Stav |
|---------|:--------:|:----:|
| **🐛 Apple Sign-In nefunguje** | **Vysoká** | ⬜ |
| Android BackgroundAudioManager fallback | Nízka | ⬜ |
| RadioListTile deprecated (ignorované) | Info | ✅ |

### 🍎 Apple Sign-In Bug - Detaily
- **Symptóm:** Po kliknutí na "Sign in with Apple" sa otvorí Apple auth, ale vráti chybu
- **Funguje:** Email/heslo prihlásenie ✅, Google OAuth2 ✅
- **TODO:** 
  - [ ] Skontrolovať Supabase Apple provider konfiguráciu
  - [ ] Overiť Apple Developer Console (Service ID, callback URL)
  - [ ] Skontrolovať Bundle ID vs Service ID
  - [ ] Pozrieť Supabase logy pre konkrétnu chybu

---

## 📊 POSTUP

- **Začiatok kontroly:** 7. december 2025
- **Aktuálna obrazovka:** Auth Screen (ďalšia v poradí)
- **Dokončené:** 13 / ~30 položiek (main.dart + 9 services + 3 screens)

---

## ✅ LEGENDA

- ⬜ = Nekontrolované
- 🔄 = V procese
- ✅ = Hotové
- ❌ = Problém - potrebuje opravu

