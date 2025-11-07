# Stripe Webhook Setup Guide

## Problém
V test mode, Stripe checkout sessions nevytvárajú automaticky webhook eventy pre lokálny development cez Stripe CLI.

## Riešenie pre lokálny development

### Možnosť 1: Použiť Stripe CLI trigger (nie cez checkout)
```bash
stripe trigger checkout.session.completed
```
**Problém:** Nevytvorí správne metadata pre donation/subscription.

### Možnosť 2: Nakonfigurovať webhook endpoint v Stripe Dashboard (TEST MODE)

1. Choďte na https://dashboard.stripe.com/test/webhooks
2. Kliknite na "Add endpoint"
3. URL: Zatiaľ nemôžete použiť localhost - potrebujete ngrok
4. Eventy: Vyberte `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`

### Možnosť 3: Nasadiť na produkčný server (ODPORÚČAM)

**Pre production:**
1. Nasaďte aplikáciu na server (napr. Vercel, Railway)
2. V Stripe Dashboard → Webhooks → Add endpoint
3. URL: `https://vašadomena.sk/api/webhooks/stripe`
4. Prepnite na LIVE MODE
5. Vyberte eventy: `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`
6. Skopírujte webhook signing secret
7. Pridajte do `.env` na serveri: `STRIPE_WEBHOOK_SECRET=whsec_...`

## Aktuálny stav

✅ Test mode kľúče nakonfigurované
✅ Stripe CLI listener beží
✅ Checkout funguje
✅ Thank you page funguje
❌ Webhooks sa nezachytávajú automaticky v test mode
✅ Manuálne pridanie donation do DB funguje

## Pre produkciu

1. Nasaďte na server
2. Použite LIVE MODE kľúče (už máte zakomentované v .env.local)
3. Nastavte webhook endpoint v Stripe Dashboard na produkčnú URL
4. Všetko bude fungovať automaticky
