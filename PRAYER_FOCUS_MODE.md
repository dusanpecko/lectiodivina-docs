# Prayer Focus Mode - Tichý Režim Modlitby

## Koncept
**"Prayer Focus Mode"** - špeciálny režim aplikácie pre hlboké sústredenie sa na modlitbu a zamyslenie.

## Funkcie Tichého Režimu

### 1. **Automatické Aktivovanie**
- ✅ **Manuálne zapnutie** - tlačidlo v lectio screen
- ✅ **Časové spustenie** - automaticky o určitých časoch (napr. 6:00, 12:00, 18:00)
- ✅ **Location-based** - v kostole/kaplnke (GPS detekcia)
- ✅ **Calendar integration** - počas liturgických hodín

### 2. **Čo sa Deje v Tichom Režime**

#### **Notifikácie:**
- 🔕 **Všetky notifikácie stlmené** (okrem urgentných)
- 🔕 **Systémové notifikácie minimalizované**
- 🔕 **App notifications suspended**
- ⚠️ **Emergency calls allowed** - len kritické hovory

#### **UI Zmeny:**
- 🌙 **Dimmed interface** - nízka intenzita farieb
- 📖 **Reading-focused layout** - len text, minimálne UI
- 🎨 **Sepia/warm colors** - pohodlné pre oči
- ⏰ **Hidden time/battery** - minimálne rozptýlenie

#### **Audio Správanie:**
- 🎵 **Lower volume** - automaticky znížená hlasitosť
- 🎧 **Audio continues** - hudba/audio text pokračuje
- 🔊 **Soft transitions** - jemné prechody medzi textami
- 🎼 **Optional background sounds** - tiché liturgické zvuky

### 3. **Implementácia v Settings**

```dart
// Nová sekcia v SettingsScreen
Card(
  child: Column(
    children: [
      ListTile(
        leading: Icon(Icons.self_improvement),
        title: Text('Tichý režim modlitby'),
        subtitle: Text('Nerušenie počas zamyslenia'),
        trailing: Switch(
          value: _prayerFocusModeEnabled,
          onChanged: _onPrayerFocusModeChanged,
        ),
      ),
      
      // Nastavenia tichého režimu
      if (_prayerFocusModeEnabled) ...[
        ListTile(
          leading: Icon(Icons.schedule),
          title: Text('Automatické spustenie'),
          subtitle: Text('6:00, 12:00, 18:00'),
          trailing: Switch(
            value: _autoFocusMode,
            onChanged: _onAutoFocusModeChanged,
          ),
        ),
        
        ListTile(
          leading: Icon(Icons.location_on),
          title: Text('V kostole/kaplnke'),
          subtitle: Text('GPS detekcia'),
          trailing: Switch(
            value: _locationBasedFocus,
            onChanged: _onLocationFocusChanged,
          ),
        ),
        
        ListTile(
          leading: Icon(Icons.timer),
          title: Text('Doba trvania'),
          subtitle: Text('$_focusModeDuration minút'),
          onTap: _showDurationPicker,
        ),
      ],
    ],
  ),
)
```

### 4. **Technické Riešenie**

#### **Android:**
```dart
// Do Not Disturb integration
await AndroidFlutterSettings.setDoNotDisturbMode(true);

// Notification channels management
await _configureNotificationChannels(silentMode: true);

// Screen brightness control
await Screen.setBrightness(0.3);
```

#### **iOS:**
```dart
// Focus modes integration
await IOSFocusMode.enablePrayerFocus();

// Notification management
await UNUserNotificationCenter.setNotificationSettings(
  silent: true,
  critical: false,
);
```

### 5. **User Experience**

#### **Spustenie:**
1. **Manuálne** - floating button v lectio screen: 🧘‍♂️
2. **Automatické** - push notification: "Čas na modlitbu - aktivovať tichý režim?"
3. **Smart detection** - detekcia čítania > 5 minút

#### **Visual Feedback:**
- 🟡 **Subtle indicator** - malá ikona v status bar
- ⏱️ **Timer display** - zostávajúci čas (optional)
- 🌊 **Breathing animation** - jemná animácia na pozadí

#### **Ukončenie:**
- ⏰ **Automatic** - po nastavenom čase
- 👆 **Manual** - tap na indicator
- 🔔 **Emergency override** - pri urgentných notifikáciách

### 6. **Rozšírené Funkcie**

#### **Integration s existujúcimi systémami:**
- 📱 **iOS Focus Modes** - sync s Prayer focus
- 🤖 **Android Do Not Disturb** - automatic activation
- ⌚ **Apple Watch** - extend focus to watch
- 🏠 **HomeKit/SmartThings** - dim room lights

#### **Community Features:**
- 👥 **Family prayer time** - sync s rodinnými členmi
- ⛪ **Parish integration** - tichý režim počas omše
- 📊 **Prayer statistics** - tracking času v tichom režime

### 7. **Prioritné Implementovanie**

#### **Fáza 1** - Základné funkcie:
- ✅ Manuálne zapnutie/vypnutie
- ✅ Stlmenie notifikácií
- ✅ Dimmed UI
- ✅ Timer s automatic vypnutím

#### **Fáza 2** - Smart features:
- ✅ Automatické spustenie podľa času
- ✅ Location-based activation
- ✅ Audio adjustments
- ✅ Settings integration

#### **Fáza 3** - Advanced:
- ✅ System integration (iOS Focus/Android DND)
- ✅ Community features
- ✅ Smart home integration

## Technická Implementácia

### **Nové súbory:**
- `lib/services/prayer_focus_service.dart`
- `lib/models/prayer_focus_settings.dart`
- `lib/widgets/focus_mode_indicator.dart`

### **Zmeny v existujúcich súboroch:**
- `settings_screen.dart` - nová sekcia
- `lectio_screen.dart` - focus mode button
- `main.dart` - service initialization

### **Dependencies:**
```yaml
dependencies:
  geolocator: ^9.0.2  # GPS detection
  screen_brightness: ^0.2.2+1  # brightness control
  flutter_local_notifications: ^16.3.0  # existing
  permission_handler: ^11.1.0  # existing
```

---

**Záver:** Prayer Focus Mode by bol unique feature pre duchovné aplikácie - kombinácia technológie a spirituality pre hlbšie zamyslenie.