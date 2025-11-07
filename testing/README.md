# 🧪 Testing Files

Tento adresár obsahuje testové súbory a scripty pre vývoj a debugging Lectio Divina aplikácie.

## 📁 Štruktúra súborov

### API Testing
- `test-api.js` - Test Bible bulk import API endpoint
- `test-api-endpoint.js` - Všeobecný API endpoint tester  
- `test-admin-endpoint.js` - Test admin API endpoints
- `test-api-with-uuid.sh` - Test API s UUID parametrami

### Data & Validation
- `test-data.json` - Testové dáta pre API requesty
- `test-validation.json` - Validačné test dáta
- `test-parsing.js` - Test parsovanie biblických veršov

### AI & Translations
- `test-ai-generator.sh` - Test AI Lectio Divina Generator
- `test-roadmap-translations.js` - Test roadmap translations
- `test-multiverse-parsing.js` - Test Multiverse API parsing

### Notifications & FCM
- `test-fcm-api.sh` - Test Firebase Cloud Messaging API
- `test-send-notification.js` - Test odosielanie notifikácií  
- `test-notification-logs.sh` - Test notification logs

## 🚀 Použitie

### JavaScript testy
```bash
node test-api.js
node test-parsing.js
```

### Shell scripty
```bash
chmod +x test-ai-generator.sh
./test-ai-generator.sh
```

### Curl testy
```bash
chmod +x test-fcm-api.sh
./test-fcm-api.sh
```

## 📝 Poznámky

- Tieto súbory boli presunuté z `/backend` adresára pre lepšiu organizáciu
- Obsahujú development a debugging kód, nie produkčnú logiku
- Používajú sa pri vývoji nových funkcií a testovaní API endpoints
- Pred spustením uistite sa, že backend server beží na `localhost:3000`

## 🔧 Konfigurácia

Väčšina testov predpokladá:
- Backend server na `http://localhost:3000`
- Platné API keys pre externé služby (OpenAI, Firebase)
- Správne nastavené environment variables

## 📚 Súvisiace dokumenty

- [API Documentation](../BIBLE_BULK_IMPORT_README.md)
- [AI Generator Guide](../AI_LECTIO_DIVINA_GENERATOR.md)
- [FCM Implementation](../FCM_IMPLEMENTATION.md)
- [Testing Checklist](../TESTING_CHECKLIST.md)