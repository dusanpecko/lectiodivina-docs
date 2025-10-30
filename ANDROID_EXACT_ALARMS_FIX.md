# Android Exact Alarms Fix - Lokálne notifikácie

## Problém
Od Android 13 (API 33+) je potrebné explicitné povolenie používateľa pre plánovanie presných alarmov. Bez tohto povolenia zlyhávalo naplánovanie lokálnych notifikácií s chybou:

```
PlatformException(exact_alarms_not_permitted, Exact alarms are not permitted, null, null)
```

## Riešenie

### 1. AndroidManifest.xml
Pridané povolenia pre presné alarmy:

```xml
<!-- Povolenie pre presné alarmy (Android 13+) -->
<uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM"/>
<uses-permission android:name="android.permission.USE_EXACT_ALARM"/>
```

**Poznámky:**
- `SCHEDULE_EXACT_ALARM` - Bežné povolenie, používateľ ho môže revoke
- `USE_EXACT_ALARM` - Pre kritické aplikácie (budík, kalendár), nevyžaduje user action

### 2. LocalNotificationsService

Pridané nové metódy:

```dart
/// Skontroluj či má aplikácia povolenie na presné alarmy (Android 13+)
Future<bool> canScheduleExactAlarms() async

/// Požiadaj o povolenie pre presné alarmy (Android 13+)
Future<void> requestExactAlarmPermission() async
```

Upravená logika plánovania notifikácií:

```dart
// Skontroluj povolenie
final canSchedule = await canScheduleExactAlarms();

// Použij správny scheduling mode
final scheduleMode = canSchedule
    ? AndroidScheduleMode.exactAllowWhileIdle    // Presné notifikácie
    : AndroidScheduleMode.inexactAllowWhileIdle; // Nepresné (fallback)
```

### 3. NotificationSettingsScreen

#### Warning Banner
Zobrazuje sa keď povolenie chýba:
- Vysvetľuje potrebu povolenia
- Informuje o nepresnom doručení (do 15 min oneskorenie)
- Ponúka tlačidlo na request povolenia

#### Dialog pred zapnutím notifikácií
Keď používateľ zapína notifikácie bez povolenia:
- Upozorní na potrebu povolenia
- Ponúkne dve možnosti:
  - "Povoliť" - Otvorí systémové nastavenia
  - "Pokračovať bez povolenia" - Použije inexact scheduling

### 4. User Flow

```
Používateľ otvorí Notification Settings
         ↓
    Kontrola povolenia
         ↓
    Chýba povolenie?
    ├─ ÁNO → Zobraz orange warning banner
    │         ├─ Klik na "Povoliť presné notifikácie"
    │         └─ Android otvorí systémové nastavenia
    │             ├─ Používateľ povolí
    │             └─ Späť do app → Banner zmizne ✓
    │
    └─ NIE → Všetko OK, bez warnings
```

## Technické detaily

### Scheduling Modes

| Mode | Presnosť | Vyžaduje povolenie | Batéria |
|------|----------|-------------------|---------|
| `exactAllowWhileIdle` | ±0 min | ✓ | Vyššia spotreba |
| `inexactAllowWhileIdle` | ±15 min | ✗ | Optimalizovaná |

### Kompatibilita

- **Android 12 a nižšie**: Povolenie sa automaticky udeľuje, bez user action
- **Android 13+**: Vyžaduje explicitný súhlas používateľa

### Fallback stratégia

Aplikácia funguje aj bez povolenia:
1. Detekuje chýbajúce povolenie
2. Automaticky prepne na `inexact` scheduling
3. Notifikácie prídu s miernym oneskorením (Android optimalizuje batériu)
4. Používateľ je informovaný a môže kedykoľvek povoliť presné alarmy

## Testovanie

### Testovací scenár

1. **Inštalácia na Android 13+**
   ```bash
   flutter build apk --debug
   flutter install
   ```

2. **Prvé otvorenie Notification Settings**
   - Očakávaný výsledok: Orange warning banner
   - UI obsahuje tlačidlo "Povoliť presné notifikácie"

3. **Zapnutie Daily Lectio bez povolenia**
   - Očakávaný výsledok: Dialog s upozornením
   - Možnosť pokračovať alebo povoliť

4. **Request povolenia**
   ```
   Klik na tlačidlo → Android Settings → Povolenie
   ```
   - Očakávaný výsledok: Banner zmizne po návrate

5. **Overenie notifikácií**
   - Počkaj na 9:00 (alebo testovací čas)
   - Očakávaný výsledok: Notifikácia príde presne o čase

### Debug logy

```
✅ Povolenie udelené:
I/flutter: 📱 Daily lectio notifications enabled

❌ Povolenie chýba:
I/flutter: ⚠️ Cannot schedule exact alarms - permission not granted. Using inexact scheduling.
```

## Best Practices

1. **Nikdy nevynútiť povolenie** - Vždy ponúknuť fallback
2. **Jasne vysvetliť benefit** - Používateľ pochopí prečo je povolenie potrebné
3. **Testovať oba režimy** - Exact aj inexact musia fungovať
4. **Logovať povolenia** - Pre debugging production issues

## Príbuzné súbory

- `/mobile/android/app/src/main/AndroidManifest.xml`
- `/mobile/lib/services/local_notifications_service.dart`
- `/mobile/lib/screens/notification_settings_screen.dart`

## Ďalšie zdroje

- [Android Docs - Schedule exact alarms](https://developer.android.com/develop/background-work/services/alarms/schedule)
- [Flutter Local Notifications - Exact alarms](https://pub.dev/packages/flutter_local_notifications#-scheduling-notifications)
- [Android 13 breaking changes](https://developer.android.com/about/versions/13/changes/alarms)

## Changelog

**2025-10-28**: Implementované complete riešenie pre Android 13+ exact alarms
- Pridané permissions do manifest
- Implementované permission checks a requests
- UI warnings a dialogs
- Fallback na inexact scheduling
- Dokumentácia a best practices
