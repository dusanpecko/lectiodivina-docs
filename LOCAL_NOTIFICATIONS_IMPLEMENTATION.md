# Lokálne notifikácie - Implementácia

## Prehľad
Implementovali sme kompletný systém lokálnych notifikácií pre aplikáciu Lectio Divina. Systém poskytuje tri typy notifikácií:

1. **Welcome notifikácie** - 3 dni po registrácii
2. **Denné lectio notifikácie** - každý deň o 9:00 ráno (7 dní dopredu s cache)
3. **Pripomenutie modlitby** - používateľom nastaviteľný čas

## Architektúra

### LocalNotificationsService
- **Lokácia**: `/lib/services/local_notifications_service.dart`
- **Pattern**: Singleton
- **Závislosti**: 
  - `flutter_local_notifications`
  - `shared_preferences` 
  - `supabase_flutter`
  - `timezone`

### Notification IDs
```dart
static const int welcomeNotificationId = 1000;
static const int dailyLectioBaseId = 2000; // 2000-2006 pre 7 dní
static const int prayerReminderBaseId = 3000; // 3000+ pre rôzne časy
```

## Funkcie

### 1. Welcome Notifikácie
- **Trigger**: 3 dni po registrácii používateľa
- **Scheduling**: Automaticky pri prihlásení (main.dart)
- **Obsah**: Motivačná správa o aplikácii
- **Payload**: `{"type": "welcome"}`

```dart
await LocalNotificationsService.instance.setupRegistrationNotification();
```

### 2. Denné Lectio Notifikácie  
- **Čas**: Každý deň o 9:00 ráno
- **Cache**: 7 dní dopredu (offline podpora)
- **Obsah**: Načítaný z Supabase `lectio_sources` + `liturgical_calendar`
- **Payload**: `{"type": "daily_lectio", "date": "2024-01-01"}`

```dart
// Zapnutie denných notifikácií
await service.scheduleDailyLectioNotifications();

// Vypnutie denných notifikácií  
await service.cancelDailyLectioNotifications();
```

### 3. Pripomenutie Modlitby
- **Čas**: Používateľom nastaviteľný
- **Periodicita**: Každý deň v nastavenom čase
- **Payload**: `{"type": "prayer_reminder"}`

```dart
// Nastavenie času pripomenutia
await service.schedulePrayerReminder(TimeOfDay(hour: 18, minute: 0));

// Zrušenie pripomenutia
await service.cancelPrayerReminder();
```

## UI Integrácia

### NotificationSettingsScreen
- **Lokácia**: `/lib/screens/notification_settings_screen.dart`
- **Nové sekcie**:
  - Lokálne notifikácie (oddelené od FCM)
  - Switch pre denné lectio
  - Time picker pre pripomenutie modlitby

### UI Komponenty
```dart
// Switch pre denné lectio
Switch(
  value: _dailyLectioEnabled,
  onChanged: (value) => _onDailyLectioChanged(value),
)

// Time picker pre modlitbu
IconButton(
  icon: const Icon(Icons.access_time),
  onPressed: _onPrayerReminderTimeChanged,
)
```

## Navigation Handling

### Main.dart Integration
```dart
// Callback pre lokálne notifikácie
void _handleLocalNotificationTap(String? payload) {
  if (payload == 'daily_lectio') {
    navigatorKey.currentState?.pushNamed('/lectio');
  } else {
    // Navigate to home for other notifications
  }
}

// Setup v main funkcii
LocalNotificationsService.instance.setNotificationCallback(_handleLocalNotificationTap);
```

## Cache Systém

### Offline Podpora
- **Cache kľúč**: `cached_lectio_data`
- **Interval**: 7 dní dopredu
- **Refresh**: Každý deň o polnoci
- **Fallback**: Ak cache chýba, generic správa

### Cache Štruktúra
```json
{
  "2024-01-01": {
    "title": "1. januára - Nový rok",
    "lectio_text": "Dnes sa zamyslíme...",
    "actio_text": "Tvoja dnešná úloha...",
    "reference": "Mt 1,1-16"
  }
}
```

## Permissions

### Android
- `android.permission.POST_NOTIFICATIONS` (API 33+)
- `android.permission.SCHEDULE_EXACT_ALARM`
- `android.permission.USE_EXACT_ALARM`

### iOS  
- UNUserNotificationCenter autorization request
- Automaticky handled v `initialize()`

## Timezone Podpora

### Setup
```dart
import 'package:timezone/timezone.dart' as tz;
import 'package:timezone/data/latest.dart' as tz;

// V main.dart
tz.initializeTimeZones();
```

### Scheduling
```dart
final scheduledDate = tz.TZDateTime.from(
  DateTime.now().add(Duration(days: 3)),
  tz.local,
);
```

## Error Handling

### Logging
- **Logger**: Instance-based logger pre debugging
- **Errors**: Caught a logged, fallback na generic správy
- **Network**: Graceful degradation ak Supabase nie je dostupný

### Fallbacks
```dart
// Ak cache chýba
final fallbackText = "Dnes si nájdi chvíľu na modlitbu a zamyslenie...";

// Ak scheduling zlyha  
catch (e) {
  _logger.e('❌ Error scheduling notification: $e');
  return false;
}
```

## Testovanie

### Manuálne Testovanie
1. Registruj nového používateľa → Skontroluj welcome notification za 3 dni
2. Zapni denné lectio → Skontroluj notification o 9:00
3. Nastav pripomenutie modlitby → Skontroluj v nastavenom čase
4. Tap na notification → Skontroluj navigation

### Debug Logs
```dart
_logger.i('📅 Scheduling notification for: $scheduledDate');
_logger.i('📱 Notification tapped: ${response.payload}');
_logger.i('💾 Cache contains ${cachedData.length} days');
```

## Budúce Vylepšenia

### Možné Rozšírenia
1. **Týždenné súhrny**: Notifikácia s týždenným prehľadom
2. **Personalizácia**: Rôzne časy pre rôzne dni
3. **Citáty**: Denné citáty zo svätých
4. **Progress tracking**: Notifikácie o pokroku v duchovnom raste
5. **Feast days**: Špeciálne notifikácie pre sviatky

### Optimalizácie
1. **Battery**: Intelligent scheduling na základe používania
2. **Bandwidth**: Delta updates pre cache
3. **Storage**: Compression pre cached data
4. **UX**: Swipe actions v notification tray

## Súhrn
Lokálne notifikácie sú teraz plne implementované s:
- ✅ 3 typy notifikácií (welcome, daily lectio, prayer reminder)
- ✅ Offline cache pre 7 dní
- ✅ UI integrácia v settings screen  
- ✅ Navigation handling
- ✅ Timezone podpora
- ✅ Error handling a logging
- ✅ Permissions management

Systém je pripravený na produkčné použitie a poskytuje spoľahlivé lokálne notifikácie pre používateľov aplikácie Lectio Divina.