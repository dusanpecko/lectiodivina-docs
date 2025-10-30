# FCM Push Notifications - Implementačná dokumentácia

## 📋 Prehľad

Systém pre push notifikácie v Lectio Divina aplikácii využíva:
- **Firebase Cloud Messaging (FCM)** pre doručovanie notifikácií
- **Supabase** pre ukladanie FCM tokenov a preferencií používateľov
- **Next.js API routes** pre backend endpoint na odosielanie notifikácií
- **Flutter FCM Service** pre mobilnú aplikáciu

## 🏗️ Architektúra

```
┌─────────────────┐
│  Mobile App     │
│  (Flutter)      │
└────────┬────────┘
         │ FCM Token
         │ Registration
         ↓
┌─────────────────┐      ┌──────────────┐
│  Supabase DB    │←─────│  Next.js API │
│  - user_fcm_    │      │  /api/       │
│    tokens       │      │  notifications│
│  - notification_│      │  /send       │
│    preferences  │      └───────┬──────┘
└─────────────────┘              │
                                 ↓
                         ┌───────────────┐
                         │ Firebase      │
                         │ Admin SDK     │
                         └───────┬───────┘
                                 │
                                 ↓
                         ┌───────────────┐
                         │ FCM Server    │
                         └───────┬───────┘
                                 │
                                 ↓
                         ┌───────────────┐
                         │ Mobile Device │
                         └───────────────┘
```

## 📦 Backend Komponenty

### 1. Firebase Admin SDK Inicializácia
**Súbor**: `/backend/src/lib/firebase-admin.ts`

Inicializuje Firebase Admin SDK s credentials z environment variables:
- `FIREBASE_PROJECT_ID`
- `FIREBASE_CLIENT_EMAIL`
- `FIREBASE_PRIVATE_KEY`

### 2. API Endpoint na odosielanie notifikácií
**Endpoint**: `POST /api/notifications/send`

**Request Body**:
```json
{
  "topicId": "daily-lectio",
  "title": "Dnešné zamyslenie",
  "body": "Prečítajte si dnešnú Lectio Divina",
  "data": {
    "type": "lectio",
    "lectioId": "123"
  },
  "localeCode": "sk",
  "userIds": ["user-1", "user-2"]
}
```

**Response**:
```json
{
  "success": true,
  "message": "Notifications sent successfully",
  "tokensCount": 150,
  "successCount": 148,
  "failureCount": 2
}
```

## 📱 Mobile (Flutter) Komponenty

### 1. FCM Service
**Súbor**: `/mobile/lib/services/fcm_service.dart`

**Funkcie**:
- ✅ Inicializácia FCM a získanie tokenu
- ✅ Registrácia tokenu na backend
- ✅ Handling foreground/background/terminated notifikácií
- ✅ Lokálne notifikácie pre foreground messages
- ✅ Deep linking z notifikácií
- ✅ Automatické aktualizácie tokenu pri zmene jazyka

### 2. Notification API Client
**Súbor**: `/mobile/lib/services/notification_api.dart`

**Funkcie**:
- Získanie notification preferences z backendu
- Update topic preferences
- FCM token management
- Offline caching preferencií

## 🗄️ Databázové tabuľky

### user_fcm_tokens
```sql
CREATE TABLE user_fcm_tokens (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  token TEXT NOT NULL UNIQUE,
  device_type TEXT NOT NULL, -- 'ios' | 'android'
  device_id TEXT,
  app_version TEXT,
  locale_code TEXT, -- 'sk' | 'en' | 'cz' | 'es' | 'de'
  is_active BOOLEAN DEFAULT true,
  last_used_at TIMESTAMP DEFAULT NOW(),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_fcm_tokens_user_id ON user_fcm_tokens(user_id);
CREATE INDEX idx_user_fcm_tokens_token ON user_fcm_tokens(token);
CREATE INDEX idx_user_fcm_tokens_active ON user_fcm_tokens(is_active);
```

### notification_topics
```sql
CREATE TABLE notification_topics (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name_sk TEXT NOT NULL,
  name_en TEXT NOT NULL,
  name_cs TEXT NOT NULL,
  name_es TEXT NOT NULL,
  name_de TEXT,
  icon TEXT, -- emoji
  category TEXT NOT NULL, -- 'spiritual' | 'educational' | 'news' | 'reminders'
  display_order INTEGER DEFAULT 0,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### user_notification_preferences
```sql
CREATE TABLE user_notification_preferences (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  topic_id UUID NOT NULL REFERENCES notification_topics(id) ON DELETE CASCADE,
  is_enabled BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, topic_id)
);

CREATE INDEX idx_user_notif_prefs_user_id ON user_notification_preferences(user_id);
CREATE INDEX idx_user_notif_prefs_topic_id ON user_notification_preferences(topic_id);
```

### notification_logs
```sql
CREATE TABLE notification_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  topic_id UUID REFERENCES notification_topics(id),
  title TEXT NOT NULL,
  body TEXT NOT NULL,
  locale_code TEXT,
  tokens_count INTEGER,
  success_count INTEGER,
  failure_count INTEGER,
  sent_at TIMESTAMP DEFAULT NOW(),
  sent_by UUID REFERENCES auth.users(id)
);

CREATE INDEX idx_notification_logs_sent_at ON notification_logs(sent_at DESC);
CREATE INDEX idx_notification_logs_topic_id ON notification_logs(topic_id);
```

## 🔐 Bezpečnosť

### API Key Authentication (voliteľné)
Pre ochranu endpointu na odosielanie notifikácií pridaj do `.env.local`:
```bash
NOTIFICATIONS_API_KEY=tvoj-bezpecny-api-key-123
```

Potom pri volaní API:
```bash
curl -X POST https://your-domain.com/api/notifications/send \
  -H "Authorization: Bearer tvoj-bezpecny-api-key-123" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

## 🧪 Testovanie

### 1. Test odoslania notifikácie
```bash
cd /Users/dusanpecko/lectiodivina/backend
node test-send-notification.js
```

### 2. Test z curl
```bash
curl -X POST http://localhost:3000/api/notifications/send \
  -H "Content-Type: application/json" \
  -d '{
    "topicId": "daily-lectio",
    "title": "Test Notifikácia",
    "body": "Toto je testovacia notifikácia",
    "data": {
      "type": "test",
      "timestamp": "'$(date +%s)'"
    },
    "localeCode": "sk"
  }'
```

## 📝 Použitie

### Backend - Odoslanie notifikácie

```typescript
// V Next.js API route alebo server action
import { sendPushNotification } from '@/lib/firebase-admin';

// Získaj FCM tokeny z databázy
const { data: tokens } = await supabase
  .from('user_fcm_tokens')
  .select('token')
  .eq('is_active', true)
  .eq('locale_code', 'sk');

// Odošli notifikácie
const result = await sendPushNotification(
  tokens.map(t => t.token),
  {
    title: 'Dnešné zamyslenie',
    body: 'Prečítajte si novú Lectio Divina'
  },
  {
    type: 'lectio',
    lectioId: '123'
  }
);

console.log(`Sent: ${result.successCount}, Failed: ${result.failureCount}`);
```

### Mobile - Inicializácia

```dart
// V main.dart
await Firebase.initializeApp();
await FcmService.instance.init(appLanguageCode);

// Handling kliknutia na notifikáciu
FcmService.instance.setNotificationCallback((message) {
  // Navigate to specific screen based on message.data
  final type = message.data['type'];
  if (type == 'lectio') {
    navigatorKey.currentState?.pushNamed(
      '/lectio',
      arguments: message.data['lectioId'],
    );
  }
});
```

## 🚀 Deployment

### 1. Environment Variables
Ubezpeč sa, že máš nastavené v produkcii:
```bash
FIREBASE_PROJECT_ID=lectio-divina-ef223
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxx@xxx.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"
NOTIFICATIONS_API_KEY=tvoj-prod-api-key
```

### 2. Vytvorenie DB tabuliek
Spusti SQL skripty v Supabase Dashboard:
1. `user_fcm_tokens`
2. `notification_topics`
3. `user_notification_preferences`
4. `notification_logs`

### 3. Vytvorenie notification topics
```sql
INSERT INTO notification_topics (name_sk, name_en, name_cs, name_es, icon, category, display_order) VALUES
('Denné zamyslenia', 'Daily Reflections', 'Denní úvahy', 'Reflexiones diarias', '🙏', 'spiritual', 1),
('Biblické výklady', 'Biblical Interpretations', 'Biblické výklady', 'Interpretaciones bíblicas', '📖', 'educational', 2),
('Modlitby', 'Prayers', 'Modlitby', 'Oraciones', '🕊️', 'spiritual', 3),
('Aktuality', 'News', 'Aktuality', 'Noticias', '📰', 'news', 4),
('Denné pripomienky', 'Daily Reminders', 'Denní připomínky', 'Recordatorios diarios', '⏰', 'reminders', 5);
```

## 🐛 Debugging

### Zapni debug logging na mobile
```dart
// V fcm_service.dart
_logger.level = Level.debug;
```

### Skontroluj FCM tokeny v DB
```sql
SELECT 
  token,
  user_id,
  device_type,
  locale_code,
  is_active,
  last_used_at
FROM user_fcm_tokens
WHERE is_active = true
ORDER BY last_used_at DESC;
```

### Skontroluj notification logs
```sql
SELECT 
  title,
  tokens_count,
  success_count,
  failure_count,
  sent_at
FROM notification_logs
ORDER BY sent_at DESC
LIMIT 10;
```

## 📚 Ďalšie zdroje

- [Firebase Cloud Messaging Docs](https://firebase.google.com/docs/cloud-messaging)
- [Flutter Firebase Messaging](https://pub.dev/packages/firebase_messaging)
- [Firebase Admin SDK](https://firebase.google.com/docs/admin/setup)

## ✅ Checklist pre go-live

- [ ] Firebase credentials nastavené v production
- [ ] DB tabuľky vytvorené
- [ ] Notification topics naplnené
- [ ] API endpoint testovaný
- [ ] Mobile app testovaná na iOS
- [ ] Mobile app testovaná na Android
- [ ] Push notifications permissions granted
- [ ] Background/foreground/terminated scenarios testované
- [ ] Deep linking z notifikácií funguje
- [ ] Rate limiting implementovaný (voliteľné)
- [ ] Monitoring/logging nastavený
