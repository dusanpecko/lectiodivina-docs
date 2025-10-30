# Delete Account API Endpoint

## 📋 Overview

Backend endpoint pre bezpečné vymazanie používateľského účtu vrátane všetkých osobných dát a Supabase Auth účtu.

**Endpoint:** `DELETE /api/user/delete-account`

**GDPR Compliance:** ✅ Právo na vymazanie (Right to Erasure) - Článok 17

---

## 🔐 Autorizácia

**Required:** Bearer token v Authorization header

```bash
Authorization: Bearer <user_access_token>
```

Token musí byť platný Supabase access token aktuálneho používateľa.

---

## 📤 Request

### Headers:
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

### Body:
Žiadny request body nie je potrebný.

---

## 📥 Response

### ✅ Success (200 OK)
```json
{
  "success": true,
  "message": "Account successfully deleted",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "deletedAt": "2025-10-28T12:34:56.789Z"
}
```

### ⚠️ Partial Success (207 Multi-Status)
Dáta vymazané, ale Auth účet zlyhal:
```json
{
  "warning": "User data deleted but auth account deletion failed",
  "details": "Error message from Supabase Auth",
  "dataDeleted": true,
  "authDeleted": false
}
```

### ❌ Error Responses

**401 Unauthorized - Missing/Invalid Token:**
```json
{
  "error": "Missing or invalid authorization header"
}
```

**401 Unauthorized - Expired Token:**
```json
{
  "error": "Invalid or expired token"
}
```

**500 Internal Server Error:**
```json
{
  "error": "Internal server error",
  "details": "Detailed error message"
}
```

---

## 🗑️ Čo sa vymaže?

Endpoint vymaže v tomto poradí:

1. **FCM Tokens** (`user_fcm_tokens`)
   - Všetky registrované device tokeny používateľa

2. **Notification Preferences** (`user_notification_preferences`)
   - Nastavenia notifikácií a topic subscriptions

3. **Notes** (`notes`)
   - Všetky používateľské poznámky

4. **Intentions** (`intentions`)
   - Modlitebné úmysly (ak existujú)

5. **User Record** (`users`)
   - Hlavný záznam používateľa v databáze

6. **Supabase Auth Account**
   - Definitívne vymazanie z Auth systému

---

## 🔒 Bezpečnosť

### Validácie:
- ✅ Token musí byť platný a neprezradený
- ✅ User môže vymazať len svoj vlastný účet
- ✅ Používa sa `SUPABASE_SERVICE_ROLE_KEY` (admin permissions)
- ✅ Service role key nie je nikdy exposed na klientovi

### Chránené údaje:
- Service role key je len na serveri
- Token validácia cez Supabase Auth
- Cascade delete zabezpečuje konzistenciu DB

---

## 💻 Flutter Implementation

```dart
Future<void> deleteAccount() async {
  final user = supabase.auth.currentUser;
  if (user == null) return;

  // 1. Potvrdenie od používateľa
  final confirmed = await showDialog<bool>(
    context: context,
    builder: (ctx) => AlertDialog(
      title: Text('Vymazať účet?'),
      content: Text('Naozaj chcete natrvalo vymazať váš účet a všetky dáta?'),
      actions: [
        TextButton(
          onPressed: () => Navigator.of(ctx).pop(false),
          child: Text('Zrušiť'),
        ),
        ElevatedButton(
          onPressed: () => Navigator.of(ctx).pop(true),
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.red,
          ),
          child: Text('Vymazať účet'),
        ),
      ],
    ),
  );

  if (confirmed != true) return;

  // 2. Získaj access token
  final session = supabase.auth.currentSession;
  final token = session?.accessToken;

  // 3. Zavolaj API
  final response = await http.delete(
    Uri.parse('https://lectio.one/api/user/delete-account'),
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Bearer $token',
    },
  );

  // 4. Handle response
  if (response.statusCode == 200 || response.statusCode == 207) {
    await supabase.auth.signOut();
    Navigator.of(context).pushNamedAndRemoveUntil('/', (route) => false);
    
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Účet bol úspešne vymazaný')),
    );
  } else {
    final error = jsonDecode(response.body);
    throw Exception(error['error'] ?? 'Failed to delete account');
  }
}
```

---

## 🧪 Testovanie

### Manual Test (cURL):

```bash
# 1. Získaj token (login cez aplikáciu a skopíruj z developer tools)
TOKEN="your_access_token_here"

# 2. Zavolaj delete endpoint
curl -X DELETE https://lectio.one/api/user/delete-account \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"

# Expected: 200 OK s JSON response
```

### Test Checklist:

- [ ] ✅ Valid token - úspešné vymazanie
- [ ] ❌ Missing token - 401 error
- [ ] ❌ Invalid token - 401 error
- [ ] ❌ Expired token - 401 error
- [ ] ✅ Partial deletion (data OK, auth fail) - 207 response
- [ ] ✅ Soft delete option (zakomentované v kóde)

---

## 🔄 Soft Delete Option

V kóde je zakomentovaná možnosť soft delete - namiesto definitívneho vymazania len označí účet ako deleted:

```typescript
// SOFT DELETE OPTION (zakomentované)
await supabaseAdmin
  .from('users')
  .update({ 
    deleted_at: new Date().toISOString(),
    email: `deleted_${userId}@deleted.com`
  })
  .eq('id', userId);
```

**Výhody soft delete:**
- Možnosť obnovy účtu
- Audit trail (kto a kedy vymazal)
- Zachovanie referencií v DB

**Nevýhody:**
- GDPR požaduje skutočné vymazanie
- Zaberá miesto v DB
- Komplikovanejšie query (filter deleted_at IS NULL)

---

## 📊 Database Schema

```sql
-- users table
CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users(id),
  email TEXT,
  full_name TEXT,
  avatar_url TEXT,
  role TEXT DEFAULT 'user',
  created_at TIMESTAMP DEFAULT NOW(),
  deleted_at TIMESTAMP NULL  -- Pre soft delete (ak používaš)
);

-- Cascade delete nastavenia
ALTER TABLE user_fcm_tokens
  ADD CONSTRAINT user_fcm_tokens_user_id_fkey
  FOREIGN KEY (user_id) 
  REFERENCES users(id) 
  ON DELETE CASCADE;

ALTER TABLE user_notification_preferences
  ADD CONSTRAINT user_notification_preferences_user_id_fkey
  FOREIGN KEY (user_id) 
  REFERENCES users(id) 
  ON DELETE CASCADE;

ALTER TABLE notes
  ADD CONSTRAINT notes_user_id_fkey
  FOREIGN KEY (user_id) 
  REFERENCES users(id) 
  ON DELETE CASCADE;
```

---

## 🐛 Troubleshooting

### Problem: Auth account not deleted but data is

**Príčina:** Supabase Auth admin deleteUser() zlyhalo

**Riešenie:** 
1. Check Supabase service role key je správny
2. Check permissions v Supabase dashboard
3. User dostane 207 response - informuj ho o čiastočnom úspechu

### Problem: Foreign key constraint violations

**Príčina:** Chýba CASCADE DELETE na foreign keys

**Riešenie:**
```sql
-- Pridaj CASCADE DELETE pre všetky FK
ALTER TABLE table_name
  DROP CONSTRAINT constraint_name,
  ADD CONSTRAINT constraint_name
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;
```

### Problem: 401 Unauthorized hneď po delete

**Príčina:** Normálne správanie - token je invalidated po delete

**Riešenie:** Nie je chyba - redirect na login screen

---

## 📝 Changelog

**v1.0** (28. október 2025)
- ✅ Initial implementation
- ✅ Complete data deletion
- ✅ Supabase Auth deletion
- ✅ Error handling
- ✅ GDPR compliance
- ✅ Flutter integration

---

## 🔗 Related Documentation

- [GDPR_DATA_EXPORT.md](./GDPR_DATA_EXPORT.md) - Data export feature
- [PROFILE_SCREEN.md](./PROFILE_SCREEN.md) - Profile screen implementation
- [SUPABASE_AUTH.md](./SUPABASE_AUTH.md) - Authentication guide

---

*Last updated: 28. október 2025*  
*Version: 1.0*  
*Status: Production-Ready*
