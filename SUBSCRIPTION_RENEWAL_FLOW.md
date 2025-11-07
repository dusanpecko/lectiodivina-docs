# Subscription Renewal Flow

## 🔄 Ako funguje automatické obnovenie subscription

### 1. **Vytvorenie subscription**
Keď user zaplatí prvýkrát:
- Frontend → `/api/checkout/subscription` 
- Vytvorí Stripe Checkout Session
- User zadá kartu a zaplatí
- Stripe webhook: `checkout.session.completed`
- Handler: `handleSubscriptionCreated()` 
- Vytvorí záznam v `subscriptions` tabuľke

### 2. **Mesačné obnovenie (automatic renewal)**
Stripe automaticky:
- Každý mesiac (na `current_period_end` dátum)
- Automaticky stiahne platbu z karty
- Pošle webhook: `invoice.paid` ✅
- Handler: `handleInvoicePaid()`
- **Aktualizuje `current_period_end` na ďalší mesiac**

### 3. **Webhook Events**

#### `invoice.paid` (Pri obnovení)
```typescript
async function handleInvoicePaid(invoice: Stripe.Invoice) {
  // Získa subscription ID z invoice
  const subscriptionId = invoice.subscription;
  
  // Stiahne aktuálne dáta zo Stripe
  const subscription = await stripe.subscriptions.retrieve(subscriptionId);
  
  // Aktualizuje databázu s novým billing period
  await supabase.from('subscriptions').update({
    status: 'active',
    current_period_start: new Date(...),
    current_period_end: new Date(...),  // ← Nový dátum!
  });
}
```

#### `customer.subscription.updated` (Pri zmene plánu/stavu)
```typescript
async function handleSubscriptionUpdated(subscription: Stripe.Subscription) {
  // Aktualizuje status, dátumy, cancel_at_period_end
  await supabase.from('subscriptions').update({...});
}
```

#### `customer.subscription.deleted` (Pri zrušení)
```typescript
async function handleSubscriptionDeleted(subscription: Stripe.Subscription) {
  // Nastaví status na 'cancelled'
  await supabase.from('subscriptions').update({
    status: 'cancelled'
  });
}
```

#### `invoice.payment_failed` (Pri neúspešnej platbe)
```typescript
async function handlePaymentFailed(invoice: Stripe.Invoice) {
  // Nastaví status na 'past_due'
  await supabase.from('subscriptions').update({
    status: 'past_due'
  });
}
```

### 4. **Timeline príkladu**

```
Day 1 (2025-11-06):
  ✅ User zaplatí €3
  📝 created_at: 2025-11-06
  📝 current_period_end: 2025-12-06
  📊 Status: active

Day 31 (2025-12-06):
  💳 Stripe automaticky stiahne €3
  🔔 Webhook: invoice.paid
  📝 current_period_end: 2026-01-06  ← Aktualizované!
  📊 Status: active

Day 61 (2026-01-06):
  💳 Stripe automaticky stiahne €3
  🔔 Webhook: invoice.paid
  📝 current_period_end: 2026-02-06  ← Aktualizované!
  📊 Status: active
```

### 5. **Stavy subscription**

| Status | Popis |
|--------|-------|
| `active` | Normálne fungujúce, platí sa |
| `past_due` | Platba zlyhala, skúša znovu |
| `cancelled` | User zrušil |
| `incomplete` | Čaká na prvú platbu |
| `incomplete_expired` | Prvá platba nevyšla, session expiroval |
| `trialing` | V skúšobnom období (ak máte trial) |
| `unpaid` | Platba zlyhala viackrát |

### 6. **Testing obnovenia**

#### Test Mode:
```bash
# Stripe CLI umožňuje trigger webhook manuálne
stripe trigger invoice.payment_succeeded

# Alebo konkrétny subscription
stripe subscriptions update sub_xxx --trial_end now
```

#### Production:
- Stripe automaticky posiela webhooks
- Webhook endpoint: `https://yourdomain.com/api/webhooks/stripe`
- Musí byť nakonfigurovaný v Stripe Dashboard

### 7. **Webhook Secret**

**Local Development:**
```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
# Webhook Secret: whsec_xxxxx (do .env.local)
```

**Production:**
```
Stripe Dashboard → Webhooks → Add endpoint
URL: https://yourdomain.com/api/webhooks/stripe
Events: 
  - checkout.session.completed
  - invoice.paid ← DÔLEŽITÉ pre renewal!
  - customer.subscription.updated
  - customer.subscription.deleted
  - invoice.payment_failed

Webhook Secret: whsec_xxxxx (do production env)
```

### 8. **Čo sa aktualizuje pri renewal**

```sql
UPDATE subscriptions 
SET 
  current_period_start = '2025-12-06',  -- Nový začiatok
  current_period_end = '2026-01-06',    -- Nový koniec (+1 mesiac)
  status = 'active',                     -- Zostáva active
  updated_at = NOW()                     -- Timestamp aktualizácie
WHERE stripe_subscription_id = 'sub_xxx';
```

### 9. **Frontend zobrazenie**

V `/account` stránke user vidí:
```
Active Subscription
─────────────────────
Plan: Supporter
Price: €3/month
Status: Active
Renews on: 2025-12-06  ← Automaticky sa aktualizuje každý mesiac
```

### 10. **Troubleshooting**

**Problém: Subscription sa neobnovila v DB**
- ✅ Check: Je webhook listener aktívny?
- ✅ Check: Je `invoice.paid` event nakonfigurovaný?
- ✅ Check: Console logs v `/api/webhooks/stripe`
- ✅ Check: Stripe Dashboard → Developers → Webhooks → Logs

**Problém: Test mode neposiela webhooks**
- ✅ Musí bežať: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`
- ✅ Webhook secret musí byť v `.env.local`

**Problém: Production webhooks nefungujú**
- ✅ URL musí byť HTTPS (nie HTTP)
- ✅ Webhook endpoint musí byť verejne prístupný
- ✅ Správny webhook secret v production env
