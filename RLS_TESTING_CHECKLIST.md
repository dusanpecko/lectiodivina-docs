# ✅ RLS Testing Checklist - Lectio Divina

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
