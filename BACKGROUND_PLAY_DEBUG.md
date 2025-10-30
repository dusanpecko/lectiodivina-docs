# Background Play Implementation - Lectio Divina

## Prehľad

Background Play umožňuje aplikácii prehrávať audio aj keď je v pozadí, s media notifikáciami a ovládacími prvkami v systémovom tray.

## Kľúčové komponenty

### 1. LectioAudioHandler (`lib/services/lectio_audio_service.dart`)
Hlavný handler pre background audio založený na `audio_service` package.

**Funkcie:**
- Správa audio playbacku cez `just_audio`
- Media notifikácie s ovládacími prvkami (play/pause/stop)
- Automatické pokračovanie na ďalšie sekcie
- App lifecycle management
- Background play nastavenia

**Kľúčové metódy:**
```dart
// Inicializácia
await _configureAudioSession()
await _loadBackgroundPlaySetting()

// Playback ovládanie
Future<void> play() async
Future<void> pause() async
Future<void> stop() async

// Section management
void _onAudioCompleted() // Auto-progress to next section
void _tryPlayNextSection()
void setOnSectionCompleted(Function callback)

// App lifecycle
Future<void> onAppLifecycleStateChanged(String state)
```

### 2. BackgroundAudioManager (`lib/services/background_audio_manager.dart`)
Singleton wrapper poskytujúci jednoduché API pre UI komponenty.

**Funkcie:**
- Inicializácia audio service
- Fallback na AudioPlayer pri chybách
- GetIt registration pre dependency injection
- Simplified API pre UI

**API:**
```dart
// Inicializácia
await BackgroundAudioManager().initialize()

// Playback control
await BackgroundAudioManager().play(url, title, artist)
await BackgroundAudioManager().pause()
await BackgroundAudioManager().stop()

// Settings
await BackgroundAudioManager().setBackgroundPlayEnabled(bool)

// State streams
Stream<PlaybackState> get playbackStateStream
bool get isPlaying
```

### 3. Android Configuration

**AndroidManifest.xml** permissions:
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
```

**AudioService** definition:
```xml
<service
    android:name="com.ryanheise.audioservice.AudioService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="true">
    <intent-filter>
        <action android:name="android.media.browse.MediaBrowserService" />
    </intent-filter>
</service>
```

**Media Button Receiver:**
```xml
<receiver android:name="com.ryanheise.audioservice.MediaButtonReceiver"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_BUTTON" />
    </intent-filter>
</receiver>
```

## Testovanie a Status

### Posledné testy:
- ✅ Background audio konfigurácia opravená
- ✅ Android permissions pridané
- ✅ Notification icon vytvorená
- ✅ Background Audio Manager inicializácia pridaná do main.dart
- 🔄 Testing prebieha na iOS simulátore

### Známe problémy:
1. **AndroidManifest assertion error** - OPRAVENÉ v v1.1
2. **Missing notification icon** - OPRAVENÉ automaticky vytvorené
3. **Missing permissions** - OPRAVENÉ pridané FOREGROUND_SERVICE_MEDIA_PLAYBACK a WAKE_LOCK

### Aktuálny stav testingu:
```
User reported: "na simulatore som dal do pozadia aplikaciu dohralo audio pustilo meditacnu hudbo ale potom uz nespustilo dalsie audio az ked som otvoril apku"
```

Toto naznačuje problém s automatickým pokračovaním na ďalšie sekcie v background mode.

## Debugging Background Continuity

### Problém:
Audio hrá prvú sekciu v pozadí, ale nepostupuje automaticky na ďalšie sekcie.

### Možné príčiny:
1. `_onAudioCompleted()` sa nevolá správne v background mode
2. App lifecycle state ovplyvňuje audio progression logic
3. Timer/callback mechanizmus je prerušený v pozadí

### Debug kroky:
1. Skontrolovať logy pre `🎵 Audio dokončené:` events
2. Overiť či sa volá `_tryPlayNextSection()` v background
3. Testovať `onAppLifecycleStateChanged()` handling

## Integrácia s Lectio Screen

### Použitie v LectioScreen:
```dart
// Initialize background audio manager callback
BackgroundAudioManager().setOnSectionCompleted(() {
  // Handle automatic section progression
  _playNextAudioSection();
});

// Play audio with background support
await BackgroundAudioManager().play(
  audioUrl,
  title: sectionTitle,
  artist: 'Lectio Divina'
);

// Cleanup
BackgroundAudioManager().clearOnSectionCompleted();
```

### Settings Integration:
V `settings_screen.dart` je Background Play toggle implementovaný.

## Dependencies

```yaml
dependencies:
  audio_service: ^0.18.15
  just_audio: ^0.9.42
  get_it: ^8.0.0
```

## Changelog

### v1.1.0 (2025-10-29)
- 🔧 Opravená AndroidManifest assertion error
- ➕ Pridané chýbajúce Android permissions
- 🎨 Automaticky vytvorená notification icon
- 🔄 Debugging background audio continuity
- 📱 Enhanced app lifecycle management

### v1.0.0 (2025-10-29)
- Implementácia základného background play
- Media notifikácie s play/pause/stop controls
- Android foreground service setup
- Automatické pokračovanie na ďalšie sekcie
- Settings integration pre enable/disable
- App lifecycle management