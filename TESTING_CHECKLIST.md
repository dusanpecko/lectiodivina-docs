# 🧪 Testing Checklist — Lectio Divina

> **Jediný zdroj pravdy** pre testovanie. Zlúčené z: notification settings, RLS CRUD, iOS background audio.
> Posledná konsolidácia: **3. jún 2026**

## Obsah
1. [Notifikácie — nastavenia](#1-notifikácie--nastavenia)
2. [RLS — CRUD operácie](#2-rls--crud-operácie)
3. [iOS — background audio](#3-ios--background-audio)

---

# 1. Notifikácie — nastavenia


## ✅ Pred Testovaním

### 1. Aplikujte SQL Migrácie

**V Supabase SQL Editore spustite**:

1. ✅ `supabase/migrations/20251011_notification_rls_policies.sql`
2. ✅ `supabase/migrations/20251011_sample_notification_topics.sql`
3. ✅ `supabase/migrations/20251011_users_rls_policies.sql` (NOVÉ)

### 2. Hot Restart Flutter App

```bash
# V terminále kde beží `flutter run`:
R  # (capital R)
```

---

## 🧪 Test Cases

### Test 1: Základné Načítanie ✅

**Kroky**:
1. Otvorte app
2. Prihláste sa
3. Profile → Nastavenia notifikácií

**Expected**:
- ✅ Loading indicator sa zobrazí
- ✅ Načíta sa 8 tém z databázy
- ✅ Zobrazí sa 4 kategórie (Spiritual, Educational, News, Reminders)
- ✅ Emoji ikony sa zobrazujú správne
- ✅ Žiadne error logy v console

**Console Logs**:
```
💡 Fetching notification preferences from Supabase...
✅ Fetched 8 topics and 3 preferences from Supabase
```

---

### Test 2: Toggle Preference (App → Database) ✅

**Kroky**:
1. Na notification settings screen
2. Zapnite tému "Denné zamyslenia"
3. Kliknite FAB "Save changes"
4. Počkajte na success message

**Expected**:
- ✅ Switch sa zmení na ON
- ✅ FAB button sa zobrazí (pending changes)
- ✅ Po save: zelená snackbar "Preferences saved"
- ✅ FAB button zmizne (no pending changes)
- ✅ Switch ostane ON (data persisted)

**Console Logs**:
```
💡 Bulk updating 1 topic preferences in Supabase
✅ Bulk updated 1 preferences successfully
🔄 Force refreshing notification preferences from database
✅ Fetched 8 topics and 4 preferences from Supabase
```

**Verify v Supabase**:
- Table Editor → `user_notification_preferences`
- Malo by existovať: `user_id: YOUR_ID, topic_id: 1, is_enabled: true`

---

### Test 3: Web → App Sync ✅

**Kroky**:
1. **V Web Dashboard**: Zmeňte preference (napr. vypnite "Modlitby")
2. **V App**: Zavrite notification settings screen
3. **V App**: Znova otvorte notification settings

**Expected**:
- ✅ App načíta FRESH data z databázy (nie cache)
- ✅ "Modlitby" je vypnuté (syncované s webom)
- ✅ Console: `🔄 Force refresh - ignoring cache`

**Alternative Method** - Pull-to-Refresh:
1. Na notification settings screen
2. Potiahnite obrazovku nadol (pull-to-refresh)
3. Počkajte na loading

**Expected**:
- ✅ Refresh indicator sa zobrazí
- ✅ Data sa načítajú z DB
- ✅ Zmeny z webu sú viditeľné

---

### Test 4: Bulk Update ✅

**Kroky**:
1. Zmeňte viacero tém naraz (napr. 3 témy)
2. Kliknite Save

**Expected**:
- ✅ Všetky zmeny sa uložia naraz (bulk upsert)
- ✅ Success message sa zobrazí
- ✅ Po refresh sú všetky zmeny persisted

**Console Logs**:
```
💡 Bulk updating 3 topic preferences in Supabase
✅ Bulk updated 3 preferences successfully
```

---

### Test 5: Offline Mode ✅

**Kroky**:
1. Vypnite WiFi/Data
2. Otvorte notification settings
3. Skúste toggle preference

**Expected**:
- ✅ Načíta sa CACHED verzia (ak existuje)
- ✅ Console: `📦 Using cached notification preferences`
- ⚠️ Toggle nebude fungovať (no internet)
- ❌ Red snackbar: "Failed to save"

**Po obnovení internetu**:
1. Pull-to-refresh
2. ✅ Fresh data z DB

---

### Test 6: Error Handling ✅

**Scenario A: Permission Denied** (ak RLS nie je nastavené)
**Expected**:
- ❌ Error screen sa zobrazí
- 🔄 "Retry" button
- Console: `⚠️ Using stale cache due to error` (fallback)

**Scenario B: User Not Authenticated**
**Expected**:
- ❌ Error: "User not authenticated"
- 🔄 Automatický redirect na login

---

### Test 7: First Time User (No Preferences) ✅

**Kroky**:
1. Nový user sa prihlási
2. Otvorte notification settings

**Expected**:
- ✅ Všetky 8 tém sa zobrazia
- ✅ Všetky sú OFF (default state)
- ✅ User môže zapnúť témy a uložiť
- ✅ Po save sa vytvorí nový záznam v DB

**Console Logs**:
```
✅ Fetched 8 topics and 0 preferences from Supabase
```

---

### Test 8: Cache Invalidation After Update ✅

**Kroky**:
1. Toggle preference a save
2. Pozrite console logs

**Expected**:
- ✅ Po save: automatický `forceRefresh: true`
- ✅ Fresh data z DB (nie stará cache)
- ✅ Console: `🔄 Force refreshing notification preferences from database`

---

## 📊 Success Criteria

### ✅ All Tests Pass:
- [ ] Test 1: Základné načítanie - OK
- [ ] Test 2: Toggle preference - OK
- [ ] Test 3: Web → App sync - OK
- [ ] Test 4: Bulk update - OK
- [ ] Test 5: Offline mode - OK
- [ ] Test 6: Error handling - OK
- [ ] Test 7: First time user - OK
- [ ] Test 8: Cache invalidation - OK

### ✅ Console Logs Clean:
- [ ] Žiadne červené error logy (❌)
- [ ] Len success logy (✅) a info logy (💡)
- [ ] `forceRefresh` logika funguje

### ✅ Database Verification:
- [ ] `user_notification_preferences` obsahuje správne záznamy
- [ ] `is_enabled` hodnoty matchujú UI
- [ ] Žiadne duplicate záznamy (unique constraint funguje)

---

## 🐛 Known Issues (FIXED)

### ❌ Duplicate Key Error - FIXED ✅
- **Fix**: `onConflict: 'user_id,topic_id'` v upsert
- **Status**: Resolved

### ❌ Stará Cache Data - FIXED ✅
- **Fix**: `forceRefresh: true` pri otvorení screen
- **Status**: Resolved

### ❌ Permission Denied (users table) - FIXED ✅
- **Fix**: RLS policies migration
- **Status**: Requires DB migration

---

## 📱 User Flow

### Happy Path:
1. User otvorí notification settings → **fresh data z DB**
2. User zmení preferences → **pending changes** (visual feedback)
3. User klikne Save → **bulk update do DB**
4. **Auto-refresh** → aktuálne dáta z DB
5. Success message → UI aktuálny

### Edge Cases:
- **Offline**: Cached data + error na save
- **Permission denied**: Error screen + retry
- **No preferences**: Všetky témy OFF (default)
- **Web changes**: Pull-to-refresh alebo re-open screen

---

## 🚀 Production Readiness

### Before Release:
- [ ] All tests pass (8/8)
- [ ] All SQL migrations applied
- [ ] RLS policies verified
- [ ] Cache strategy tested (fresh on open, cache on error)
- [ ] Error handling tested (offline, permission denied)
- [ ] Web ↔ App sync verified

### Performance:
- [ ] Load time < 2 seconds
- [ ] Pull-to-refresh smooth
- [ ] No unnecessary DB queries (cache on error only)

---

**Last Updated**: 11. október 2025, 22:30  
**Status**: 🟢 READY FOR TESTING  
**Critical Path**: Apply users RLS migration → Hot restart → Test all 8 cases

---

# 2. RLS — CRUD operácie


**Dátum:** 24. november 2025  
**Účel:** Otestovať CRUD operácie po zapnutí RLS  

---

## 🎯 Príprava pred testovaním

### 1. Spustiť SQL skript
```bash
# V Supabase SQL Editor spustite:
backend/sql/ENABLE_RLS_FINAL.sql
```

### 2. Overiť RLS status
```sql
-- Malo by vrátiť ✅ RLS ENABLED pre všetky tabuľky
SELECT 
  tablename,
  CASE 
    WHEN c.relrowsecurity THEN '✅ RLS ENABLED'
    ELSE '❌ RLS DISABLED'
  END as rls_status
FROM pg_tables t
JOIN pg_class c ON c.relname = t.tablename
WHERE schemaname = 'public'
  AND tablename IN ('users', 'subscriptions', 'donations', 'orders', 'products')
ORDER BY tablename;
```

### 3. Overiť admin rolu
```sql
-- Overte, že máte admin rolu
SELECT id, email, role FROM users WHERE email = 'your-email@example.com';

-- Ak nemáte, nastavte:
UPDATE users SET role = 'admin' WHERE email = 'your-email@example.com';
```

---

## 📋 TEST 1: USERS TABLE

### ✅ Read (SELECT)
**Ako user:**
```
1. Prihláste sa do aplikácie
2. Choďte na /profile
3. ✅ Očakávané: Vidíte svoj profil
4. ❌ Nevidíte profily iných userov
```

**Test v Supabase:**
```sql
-- Ako authenticated user (použite svoj JWT token)
SELECT * FROM users WHERE id = auth.uid();
-- ✅ Mali by ste vidieť svoj záznam

SELECT * FROM users WHERE id != auth.uid();
-- ❌ Nemali by ste vidieť iné záznamy (ak nie ste admin)
```

### ✅ Update (UPDATE)
**Ako user:**
```
1. Na /profile zmeňte "Full Name"
2. Kliknite "Save Changes"
3. ✅ Očakávané: Úspešne uložené
4. Refresh stránku → zmena je viditeľná
```

**Test v Supabase:**
```sql
-- Ako authenticated user
UPDATE users SET full_name = 'New Name Test' WHERE id = auth.uid();
-- ✅ Mali by ste úspešne updatovať svoj záznam

UPDATE users SET full_name = 'Hack Attempt' WHERE id != auth.uid();
-- ❌ Chyba: insufficient privilege (správne!)
```

### ✅ Delete (DELETE)
**Ako user:**
```
1. Na /profile → Security → Delete Account
2. Zadajte "DELETE MY ACCOUNT"
3. Potvďte
4. ✅ Očakávané: Účet zmazaný, redirect na login
```

**Test v Supabase:**
```sql
-- Ako authenticated user
DELETE FROM users WHERE id = auth.uid();
-- ✅ Mali by ste úspešne zmazať svoj účet

DELETE FROM users WHERE id != auth.uid();
-- ❌ Chyba: insufficient privilege (správne!)
```

**Výsledok:**
- [ ] ✅ Read own profile - OK
- [ ] ✅ Update own profile - OK
- [ ] ❌ Cannot read other profiles - OK
- [ ] ❌ Cannot update other profiles - OK
- [ ] ✅ Can delete own account - OK

---

## 📋 TEST 2: SUBSCRIPTIONS TABLE

### ✅ Read (SELECT)
**Ako user:**
```
1. Na /profile → "Moje predplatné"
2. ✅ Očakávané: Vidíte svoje predplatné (ak máte)
3. ❌ Nevidíte predplatné iných userov
```

**Test v Supabase:**
```sql
-- Ako authenticated user
SELECT * FROM subscriptions WHERE user_id = auth.uid();
-- ✅ Mali by ste vidieť svoje predplatné

SELECT * FROM subscriptions WHERE user_id != auth.uid();
-- ❌ Nemali by ste vidieť iné predplatné
```

### ✅ Create via Stripe Webhook
**Test:**
```
1. Choďte na /support
2. Kliknite "Become a Supporter" (€3/mes)
3. Dokončite Stripe checkout (test card: 4242 4242 4242 4242)
4. ✅ Očakávané: 
   - Webhook vytvorí záznam v subscriptions
   - Vidíte nové predplatné na /profile
```

**Verify v Supabase:**
```sql
-- Webhook používa service_role, takže by mal úspešne vytvoriť záznam
SELECT * FROM subscriptions WHERE user_id = 'your-user-id' ORDER BY created_at DESC LIMIT 1;
-- ✅ Nové predplatné by malo byť viditeľné
```

### ✅ Cancel Subscription
**Test:**
```
1. Na /profile → "Moje predplatné"
2. Kliknite "Zrušiť predplatné"
3. Potvďte
4. ✅ Očakávané: 
   - API volanie na /api/cancel-subscription
   - Stripe webhook updatne subscription
   - Status zmení na "canceled"
```

**Výsledok:**
- [ ] ✅ Read own subscriptions - OK
- [ ] ❌ Cannot read other subscriptions - OK
- [ ] ✅ Stripe webhook creates subscription - OK
- [ ] ✅ Cancel subscription works - OK

---

## 📋 TEST 3: DONATIONS TABLE

### ✅ Read (SELECT)
**Ako user:**
```
1. Na /profile → "Moje príspevky"
2. ✅ Očakávané: Vidíte svoje dary
3. ❌ Nevidíte dary iných userov
```

### ✅ Create via Stripe Checkout
**Test:**
```
1. Choďte na /support
2. Scroll dolu → "One-time donation"
3. Vyberte sumu (napr. €10)
4. Dokončite checkout
5. ✅ Očakávané:
   - Webhook vytvorí donation
   - Vidíte nový dar na /profile
```

**Verify v Supabase:**
```sql
SELECT * FROM donations WHERE user_id = 'your-user-id' ORDER BY created_at DESC LIMIT 1;
-- ✅ Nový dar by mal byť viditeľný
```

### ✅ Anonymous Donation
**Test:**
```
1. Odhlásť sa
2. Choďte na /support
3. Vytvorte anonymous donation
4. ✅ Očakávané:
   - Webhook vytvorí donation s user_id = NULL
   - is_anonymous = true
```

**Výsledok:**
- [ ] ✅ Read own donations - OK
- [ ] ❌ Cannot read other donations - OK
- [ ] ✅ Stripe webhook creates donation - OK
- [ ] ✅ Anonymous donations work - OK

---

## 📋 TEST 4: ORDERS TABLE

### ✅ Read (SELECT)
**Ako user:**
```
1. Na /profile → "Moje objednávky"
2. ✅ Očakávané: Vidíte svoje objednávky
3. ❌ Nevidíte objednávky iných userov
```

### ✅ Create Order via Checkout
**Test:**
```
1. Choďte na /shop
2. Pridajte produkt do košíka
3. Checkout → dokončite platbu
4. ✅ Očakávané:
   - Webhook vytvorí order + order_items
   - Vidíte novú objednávku na /profile
```

**Verify v Supabase:**
```sql
SELECT * FROM orders WHERE user_id = 'your-user-id' ORDER BY created_at DESC LIMIT 1;
-- ✅ Nová objednávka by mala byť viditeľná

SELECT * FROM order_items WHERE order_id = (
  SELECT id FROM orders WHERE user_id = 'your-user-id' ORDER BY created_at DESC LIMIT 1
);
-- ✅ Order items by mali byť viditeľné
```

**Výsledok:**
- [ ] ✅ Read own orders - OK
- [ ] ❌ Cannot read other orders - OK
- [ ] ✅ Stripe webhook creates order - OK
- [ ] ✅ Order items visible - OK

---

## 📋 TEST 5: PRODUCTS TABLE

### ✅ Read (SELECT) - Public Access
**Ako návštevník (neprihlásen):**
```
1. Choďte na /shop
2. ✅ Očakávané: Vidíte všetky aktívne produkty
3. Vidíte ceny, obrázky, popis
```

**Test v Supabase:**
```sql
-- Bez autentifikácie (anon key)
SELECT * FROM products WHERE is_active = true;
-- ✅ Mali by ste vidieť aktívne produkty

SELECT * FROM products WHERE is_active = false;
-- ❌ Nemali by ste vidieť neaktívne produkty
```

### ✅ Admin Management
**Ako admin:**
```
1. Prihláste sa ako admin
2. Choďte na /admin/shop
3. Vytvorte nový produkt
4. ✅ Očakávané: Úspešne vytvorený
5. Editujte produkt
6. ✅ Očakávané: Úspešne updatovaný
7. Zmazať produkt
8. ✅ Očakávané: Úspešne zmazaný
```

**Test v Supabase:**
```sql
-- Ako admin user
INSERT INTO products (name, slug, price, stock, is_active) 
VALUES ('Test Product', 'test-product', 9.99, 10, true);
-- ✅ Mali by ste úspešne vytvoriť produkt

UPDATE products SET price = 12.99 WHERE slug = 'test-product';
-- ✅ Mali by ste úspešne updatovať

DELETE FROM products WHERE slug = 'test-product';
-- ✅ Mali by ste úspešne zmazať
```

**Výsledok:**
- [ ] ✅ Public can view active products - OK
- [ ] ❌ Public cannot view inactive products - OK
- [ ] ✅ Admin can create products - OK
- [ ] ✅ Admin can update products - OK
- [ ] ✅ Admin can delete products - OK

---

## 📋 TEST 6: NOTIFICATION PREFERENCES

### ✅ Read & Update
**Ako user:**
```
1. Na /profile → "Nastavenia notifikácií"
2. ✅ Očakávané: Vidíte všetky notification topics
3. Zapnite/vypnite tému
4. Kliknite "Save changes"
5. ✅ Očakávané: Úspešne uložené
6. Refresh stránku → zmeny sú viditeľné
```

**Test v Supabase:**
```sql
-- Ako authenticated user
SELECT * FROM user_notification_preferences WHERE user_id = auth.uid();
-- ✅ Mali by ste vidieť svoje preferences

-- Vytvorte novú preference
INSERT INTO user_notification_preferences (user_id, topic_id, is_enabled)
VALUES (auth.uid(), (SELECT id FROM notification_topics LIMIT 1), true);
-- ✅ Mali by ste úspešne vytvoriť

-- Update preference
UPDATE user_notification_preferences 
SET is_enabled = false 
WHERE user_id = auth.uid();
-- ✅ Mali by ste úspešne updatovať
```

**Výsledok:**
- [ ] ✅ Read own preferences - OK
- [ ] ✅ Create/update preferences - OK
- [ ] ❌ Cannot modify other users preferences - OK

---

## 📋 TEST 7: ARTICLES (PUBLIC READ)

### ✅ Public Read
**Ako návštevník:**
```
1. Choďte na /news alebo /articles
2. ✅ Očakávané: Vidíte publikované články
3. Kliknite na článok → čítate celý obsah
```

**Test v Supabase:**
```sql
-- Bez autentifikácie
SELECT * FROM articles WHERE status = 'published';
-- ✅ Mali by ste vidieť publikované články

SELECT * FROM articles WHERE status = 'draft';
-- ❌ Nemali by ste vidieť draft články
```

### ✅ Admin Write
**Ako admin:**
```
1. Prihláste sa ako admin
2. Choďte na /admin/articles
3. Vytvorte nový článok (draft)
4. ✅ Očakávané: Úspešne vytvorený
5. Publikujte článok (status = 'published')
6. ✅ Očakávané: Viditeľný na /news
```

**Výsledok:**
- [ ] ✅ Public can read published articles - OK
- [ ] ❌ Public cannot read draft articles - OK
- [ ] ✅ Admin can create/edit articles - OK

---

## 📋 TEST 8: ADMIN-ONLY TABLES

### ✅ Notification Logs
**Ako admin:**
```
1. Choďte na /admin/notifications
2. Kliknite "View logs"
3. ✅ Očakávané: Vidíte notification logs
```

**Test v Supabase:**
```sql
-- Ako admin
SELECT * FROM notification_logs ORDER BY created_at DESC LIMIT 10;
-- ✅ Mali by ste vidieť logy

-- Ako regular user
-- ❌ Nemali by ste vidieť logy
```

### ✅ Beta Feedback
**Ako user:**
```
1. V aplikácii kliknite "Send feedback"
2. Vyplňte formulár
3. Submit
4. ✅ Očakávané: Feedback vytvorený
5. Na /profile → nemôžete vidieť feedback iných
```

**Ako admin:**
```
1. Choďte na /admin/beta-feedback
2. ✅ Očakávané: Vidíte všetky feedbacky
```

**Výsledok:**
- [ ] ✅ Users can create feedback - OK
- [ ] ✅ Users can view own feedback - OK
- [ ] ❌ Users cannot view other feedback - OK
- [ ] ✅ Admins can view all feedback - OK

---

## 🚨 Critical Tests - MUSÍ FUNGOVAŤ

### 1. Stripe Webhooks
```bash
# Otestujte webhook endpoint
curl -X POST https://lectio.one/api/webhooks/stripe \
  -H "Content-Type: application/json" \
  -H "Stripe-Signature: test" \
  -d '{"type":"test"}'

# ✅ Očakávané: 200 OK (alebo 400 pre invalid signature)
# ❌ NIE: 403 Forbidden (znamená RLS problém)
```

### 2. Service Role Prístup
```javascript
// V backend API route s SUPABASE_SERVICE_ROLE_KEY
const { data, error } = await supabase
  .from('subscriptions')
  .insert({ user_id: userId, tier: 'supporter', ... });

// ✅ Očakávané: Úspešne vytvorené
// ❌ NIE: Permission denied
```

### 3. User Profile Update
```
1. Prihláste sa
2. Na /profile zmeňte email alebo meno
3. ✅ Očakávané: Úspešne uložené
4. ❌ NIE: 403 Forbidden alebo "insufficient privilege"
```

---

## 📊 Final Checklist

### Backend API Routes (service_role)
- [ ] ✅ `/api/webhooks/stripe` - webhook môže vytvárať subscriptions
- [ ] ✅ `/api/webhooks/stripe` - webhook môže vytvárať donations
- [ ] ✅ `/api/webhooks/stripe` - webhook môže vytvárať orders
- [ ] ✅ `/api/admin/*` - admin endpoints fungujú
- [ ] ✅ `/api/ai-*` - AI endpoints fungujú (service_role)

### User Operations
- [ ] ✅ User môže čítať svoj profil
- [ ] ✅ User môže updatovať svoj profil
- [ ] ✅ User môže zmazať svoj účet
- [ ] ✅ User vidí svoje subscriptions/donations/orders
- [ ] ❌ User NEVIDÍ cudzie data

### Public Access
- [ ] ✅ Public môže vidieť aktívne produkty
- [ ] ✅ Public môže vidieť publikované články
- [ ] ✅ Public môže vidieť lectio sources
- [ ] ✅ Public môže vidieť rosary mysteries
- [ ] ✅ Public môže vidieť notification topics

### Admin Access
- [ ] ✅ Admin môže spravovať všetky tabuľky
- [ ] ✅ Admin môže vidieť notification logs
- [ ] ✅ Admin môže vidieť všetky objednávky
- [ ] ✅ Admin môže vidieť všetky feedbacky

---

## 🎉 Po úspešnom otestovaní

### 1. Zmazať dočasné súbory
```bash
rm backend/sql/DISABLE_RLS_TEMPORARILY.sql
git commit -m "feat: Enable RLS on all tables with proper policies"
```

### 2. Updatovať dokumentáciu
```markdown
✅ RLS zapnuté na všetkých tabuľkách
✅ Otestované všetky CRUD operácie
✅ Webhooks fungujú
✅ Admin panel funguje
```

### 3. Deploy na production
```bash
# Najprv otestujte na staging!
git push origin main
```

---

## ❌ Ak niečo nefunguje

### Debug checklist:
1. ✅ Overte, že ste prihlásený
2. ✅ Overte admin rolu: `SELECT role FROM users WHERE id = auth.uid()`
3. ✅ Skontrolujte browser console pre errory
4. ✅ Skontrolujte Supabase logs
5. ✅ Overte, že používate správny Supabase key (anon vs service_role)

### Common errors:
```
❌ "insufficient privilege"
→ RLS politika blokuje operáciu
→ Skontrolujte USING/WITH CHECK clauses

❌ "row-level security policy"
→ Politika neexistuje pre danú operáciu
→ Pridajte policy pre INSERT/UPDATE/DELETE

❌ "permission denied for table"
→ RLS nie je správne nastavené
→ Overte ALTER TABLE ... ENABLE ROW LEVEL SECURITY
```

---

**Status:** ⏳ Ready for testing  
**Estimated time:** 30-60 minút  
**Critical:** Otestujte Stripe webhooks!

---

# 3. iOS — background audio


## Príprava
- [ ] Build aplikácie pre iOS zariadenie
- [ ] Nainštalovať aplikáciu na fyzické iOS zariadenie (simulátor nepodporuje background audio správne)
- [ ] Uistiť sa, že zariadenie má pripojenie na internet (pre streamovanie audio)

## Test 1: Základné prehrávanie v pozadí
1. [ ] Otvoriť aplikáciu
2. [ ] Prejsť na Lectio screen
3. [ ] Vybrať audio režim (short/long)
4. [ ] Spustiť prvý audio track
5. [ ] Počkať kým sa track začne prehrávať (aspoň 5 sekúnd)
6. [ ] **Zamknúť obrazovku** (stlačiť power button)
7. [ ] **Očakávaný výsledok**: Audio pokračuje v prehrávaní
8. [ ] Počkať kým sa track dokončí
9. [ ] **Očakávaný výsledok**: Automaticky začne interlude hudba
10. [ ] Počkať kým sa interlude dokončí
11. [ ] **Očakávaný výsledok**: Automaticky začne ďalší track
12. [ ] Odomknúť obrazovku
13. [ ] **Očakávaný výsledok**: UI zobrazuje správny track a pozíciu

## Test 2: Lock screen controls
1. [ ] Spustiť audio track
2. [ ] Zamknúť obrazovku
3. [ ] Otvoriť Control Center / Lock screen media controls
4. [ ] **Očakávaný výsledok**: Zobrazuje sa správny názov tracku
5. [ ] Stlačiť **Next** button
6. [ ] **Očakávaný výsledok**: Preskočí na ďalší track
7. [ ] Stlačiť **Previous** button
8. [ ] **Očakávaný výsledok**: Vráti sa na predchádzajúci track
9. [ ] Stlačiť **Pause** button
10. [ ] **Očakávaný výsledok**: Audio sa pozastaví
11. [ ] Stlačiť **Play** button
12. [ ] **Očakávaný výsledok**: Audio pokračuje

## Test 3: Prechod medzi aplikáciami
1. [ ] Spustiť audio track
2. [ ] Prejsť do inej aplikácie (napr. Safari)
3. [ ] **Očakávaný výsledok**: Audio pokračuje v prehrávaní
4. [ ] Počkať kým sa track dokončí
5. [ ] **Očakávaný výsledok**: Automaticky začne interlude a potom ďalší track
6. [ ] Vrátiť sa do Lectio Divina aplikácie
7. [ ] **Očakávaný výsledok**: UI zobrazuje správny stav

## Test 4: Celý playlist v pozadí
1. [ ] Otvoriť Lectio screen s viacerými trackmi (napr. 4-5 trackov)
2. [ ] Nastaviť audio režim na "short" alebo "long"
3. [ ] Spustiť prvý track
4. [ ] **Okamžite zamknúť obrazovku**
5. [ ] Nechať zariadenie zamknuté počas celého playlistu
6. [ ] **Očakávaný výsledok**: 
   - Všetky tracky sa prehrávajú v správnom poradí
   - Medzi trackami sa prehráva interlude hudba
   - Po poslednom tracku sa prehráva finálna meditačná hudba
7. [ ] Odomknúť obrazovku po skončení
8. [ ] **Očakávaný výsledok**: UI zobrazuje, že playlist sa dokončil

## Test 5: Prerušenie telefonátom
1. [ ] Spustiť audio track
2. [ ] Zamknúť obrazovku
3. [ ] Zavolať na zariadenie z iného telefónu
4. [ ] **Očakávaný výsledok**: Audio sa automaticky pozastaví
5. [ ] Prijať hovor
6. [ ] Ukončiť hovor
7. [ ] **Očakávaný výsledok**: Audio sa automaticky obnoví (alebo je možné ho obnoviť cez lock screen controls)

## Test 6: Režim Nerušiť (Do Not Disturb)
1. [ ] Aktivovať DND režim v aplikácii
2. [ ] Spustiť audio track
3. [ ] Zamknúť obrazovku
4. [ ] **Očakávaný výsledok**: 
   - Audio pokračuje v prehrávaní
   - Žiadne notifikácie neprerušujú prehrávanie
5. [ ] Deaktivovať DND režim
6. [ ] **Očakávaný výsledok**: Audio stále hrá

## Test 7: Nízka batéria
1. [ ] Počkať kým batéria klesne pod 20% (alebo použiť simuláciu)
2. [ ] Spustiť audio track
3. [ ] Zamknúť obrazovku
4. [ ] **Očakávaný výsledok**: Audio pokračuje aj pri nízkej batérii
5. [ ] Aktivovať Low Power Mode
6. [ ] **Očakávaný výsledok**: Audio stále pokračuje

## Známe problémy a očakávané správanie

### ✅ Správne správanie
- Audio pokračuje v pozadí aj pri zamknutej obrazovke
- Všetky tracky v playliste sa prehrávajú v správnom poradí
- Interlude hudba sa prehráva medzi trackami
- Lock screen controls fungujú správne
- UI sa aktualizuje po odomknutí obrazovky

### ⚠️ Možné problémy
- Ak je slabé internetové pripojenie, môže dôjsť k buffering pauzám
- Pri veľmi dlhých interlude môže iOS ukončiť session (riešené v kóde)
- Pri prerušení iným audio zdrojom (napr. Siri) sa môže audio pozastaviť

## Debugging
Ak audio prestane hrať v pozadí:
1. Skontrolovať Xcode console logs pre chybové hlášky
2. Hľadať "❌" alebo "⚠️" v logoch
3. Overiť, že `Info.plist` obsahuje `UIBackgroundModes` s `audio`
4. Overiť, že audio session je aktívna: hľadať "🎵 AudioSession configured for background playback"

## Reportovanie problémov
Pri reportovaní problému uveďte:
- [ ] iOS verzia
- [ ] Model zariadenia
- [ ] Ktorý test zlyhal
- [ ] Xcode console logs (ak sú dostupné)
- [ ] Kroky na reprodukciu
