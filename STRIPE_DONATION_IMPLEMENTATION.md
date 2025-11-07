# 💳 Stripe Donation Implementation - Technická špecifikácia

**Dátum:** 1. november 2025  
**Platforma:** Stripe Payments + Stripe Billing  
**Use case:** Recurring donations pre Lectio.one

---

## ✅ **ÁNO, STRIPE MÁ DONATION PRODUKT!**

### **Stripe má 2 riešenia pre dary:**

1. **Stripe Donations** (beta)
   - Špecificky pre NGO/charities
   - Nižšie fees (1.5% + €0.25 vs 2.9% + €0.30)
   - Tax receipt automatizácia
   - **Require:** Non-profit status

2. **Stripe Subscriptions** (štandardné)
   - Funguje ako "recurring donation"
   - Použijeme ako donation tiers
   - **Toto použijeme!** (pretože nemáme ešte NGO status)

---

## 🏗️ **ARCHITEKTÚRA RIEŠENIA**

### **Stack:**
```
Frontend (Flutter) 
    ↓
Backend (Next.js API Routes)
    ↓
Stripe API (Subscriptions + Checkout)
    ↓
Webhooks → Supabase (user_subscriptions table)
```

### **Stripe Products:**

```javascript
// Tier 1: Modlím sa za vás (FREE)
// Žiadny Stripe produkt - len internal tracking

// Tier 2: Priateľ Lectio
const tier2 = {
  product: {
    name: "Priateľ Lectio - Ročný dar",
    description: "Podporte Lectio.one a získajte prístup k premium obsahu",
    type: "service" // nie "good" - pretože digitálne
  },
  price: {
    unit_amount: 3000, // €30 v centoch
    currency: "eur",
    recurring: {
      interval: "year",
      interval_count: 1
    }
  }
}

// Tier 3: Patron Lectio
const tier3Monthly = {
  product: { name: "Patron Lectio - Mesačný dar" },
  price: {
    unit_amount: 2000, // €20
    currency: "eur",
    recurring: { interval: "month" }
  }
}

const tier3Yearly = {
  product: { name: "Patron Lectio - Ročný dar" },
  price: {
    unit_amount: 20000, // €200 (zľava €40)
    currency: "eur",
    recurring: { interval: "year" }
  }
}

// Tier 4: Zakladateľ Lectio
const tier4Monthly = {
  product: { name: "Zakladateľ Lectio - Mesačný dar" },
  price: {
    unit_amount: 5000, // €50
    currency: "eur",
    recurring: { interval: "month" }
  }
}

const tier4Yearly = {
  product: { name: "Zakladateľ Lectio - Ročný dar" },
  price: {
    unit_amount: 50000, // €500 (zľava €100)
    currency: "eur",
    recurring: { interval: "year" }
  }
}
```

---

## 🆔 **ID A VARIABILNÝ SYMBOL**

### **ÁNO! Každý user bude mať:**

#### **1. Stripe Customer ID**
```javascript
// Pri prvej platbe vytvoríme Stripe Customer
const customer = await stripe.customers.create({
  email: "dusan@example.com",
  name: "Dušan Pecko",
  metadata: {
    supabase_user_id: "abc-123-def", // Prepojenie s Supabase
    tier: "patron", // Ktorý tier
    signup_date: "2025-11-01"
  }
});

// Returns:
{
  id: "cus_ABC123XYZ", // ← Stripe Customer ID
  email: "dusan@example.com",
  ...
}
```

**Tento `cus_ABC123XYZ` je unikátny identifikátor usera v Stripe.**

#### **2. Stripe Subscription ID**
```javascript
// Pri subscribe vytvoríme subscription
const subscription = await stripe.subscriptions.create({
  customer: "cus_ABC123XYZ",
  items: [{ price: "price_patron_monthly" }],
  metadata: {
    tier: "patron",
    source: "mobile_app"
  }
});

// Returns:
{
  id: "sub_DEF456GHI", // ← Subscription ID
  customer: "cus_ABC123XYZ",
  status: "active",
  current_period_end: 1704067200, // Unix timestamp
  ...
}
```

#### **3. Variabilný symbol (Invoice Number)**
```javascript
// Každá platba má Invoice s unikátnym číslom
{
  invoice: {
    id: "in_JKL789MNO",
    number: "LECTIO-2025-0001", // ← Variabilný symbol
    customer: "cus_ABC123XYZ",
    subscription: "sub_DEF456GHI",
    amount_paid: 2000, // €20
    currency: "eur",
    created: 1698840000
  }
}
```

### **Formát variabilného symbolu (custom):**

```javascript
// Môžeme si vytvoriť vlastný formát
const generateInvoiceNumber = (userId, timestamp) => {
  const year = new Date(timestamp).getFullYear();
  const userShort = userId.substring(0, 6).toUpperCase();
  const random = Math.floor(Math.random() * 9999).toString().padStart(4, '0');
  
  return `LECTIO-${year}-${userShort}-${random}`;
  // Príklad: LECTIO-2025-ABC123-0042
}

// Použitie v Stripe invoice
await stripe.invoices.create({
  customer: customerId,
  auto_advance: true,
  metadata: {
    custom_invoice_number: generateInvoiceNumber(userId, Date.now())
  }
});
```

---

## 📊 **DATABÁZOVÁ SCHÉMA (Supabase)**

### **Nová tabuľka: `user_donations`**

```sql
CREATE TABLE user_donations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- User info
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  email TEXT NOT NULL,
  
  -- Stripe IDs
  stripe_customer_id TEXT UNIQUE NOT NULL, -- cus_ABC123XYZ
  stripe_subscription_id TEXT UNIQUE, -- sub_DEF456GHI (nullable ak one-time)
  
  -- Donation details
  tier TEXT NOT NULL CHECK (tier IN ('modlitba', 'priatel', 'patron', 'zakladatel')),
  amount_cents INTEGER NOT NULL, -- v centoch (2000 = €20)
  currency TEXT DEFAULT 'eur',
  interval TEXT CHECK (interval IN ('month', 'year', 'one_time')),
  
  -- Status
  status TEXT NOT NULL CHECK (status IN ('active', 'canceled', 'past_due', 'trialing')),
  current_period_start TIMESTAMPTZ,
  current_period_end TIMESTAMPTZ,
  cancel_at_period_end BOOLEAN DEFAULT false,
  
  -- Variabilný symbol
  last_invoice_number TEXT, -- LECTIO-2025-ABC123-0042
  
  -- Metadata
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  canceled_at TIMESTAMPTZ,
  
  -- Indexes
  CONSTRAINT unique_user_subscription UNIQUE(user_id, stripe_subscription_id)
);

CREATE INDEX idx_user_donations_user_id ON user_donations(user_id);
CREATE INDEX idx_user_donations_stripe_customer ON user_donations(stripe_customer_id);
CREATE INDEX idx_user_donations_status ON user_donations(status);
CREATE INDEX idx_user_donations_tier ON user_donations(tier);
```

### **Tabuľka: `donation_transactions` (história platieb)**

```sql
CREATE TABLE donation_transactions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- References
  user_id UUID REFERENCES auth.users(id),
  donation_id UUID REFERENCES user_donations(id),
  
  -- Stripe IDs
  stripe_invoice_id TEXT UNIQUE NOT NULL, -- in_JKL789MNO
  stripe_payment_intent_id TEXT, -- pi_MNO012PQR
  
  -- Payment details
  amount_cents INTEGER NOT NULL,
  currency TEXT DEFAULT 'eur',
  status TEXT NOT NULL CHECK (status IN ('paid', 'pending', 'failed', 'refunded')),
  
  -- Invoice
  invoice_number TEXT UNIQUE, -- LECTIO-2025-ABC123-0042 (variabilný symbol)
  invoice_pdf_url TEXT, -- Stripe hosted PDF
  
  -- Tax receipt
  tax_receipt_sent BOOLEAN DEFAULT false,
  tax_receipt_sent_at TIMESTAMPTZ,
  
  -- Timestamps
  paid_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Metadata
  metadata JSONB -- Extra info z Stripe
);

CREATE INDEX idx_transactions_user_id ON donation_transactions(user_id);
CREATE INDEX idx_transactions_invoice_number ON donation_transactions(invoice_number);
CREATE INDEX idx_transactions_paid_at ON donation_transactions(paid_at DESC);
```

---

## 🔐 **API ENDPOINTS (Next.js)**

### **1. Create Checkout Session**

```typescript
// /api/donations/create-checkout
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  try {
    const { userId, tier, interval } = await req.json();
    
    // Get user from Supabase
    const supabase = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_KEY!
    );
    
    const { data: user } = await supabase
      .from('users')
      .select('email, full_name')
      .eq('id', userId)
      .single();
    
    if (!user) {
      return NextResponse.json({ error: 'User not found' }, { status: 404 });
    }
    
    // Mapping tier → Stripe Price ID
    const priceMap = {
      'priatel_year': process.env.STRIPE_PRICE_FRIEND_YEARLY!,
      'patron_month': process.env.STRIPE_PRICE_PATRON_MONTHLY!,
      'patron_year': process.env.STRIPE_PRICE_PATRON_YEARLY!,
      'zakladatel_month': process.env.STRIPE_PRICE_FOUNDER_MONTHLY!,
      'zakladatel_year': process.env.STRIPE_PRICE_FOUNDER_YEARLY!,
    };
    
    const priceId = priceMap[`${tier}_${interval}`];
    
    if (!priceId) {
      return NextResponse.json({ error: 'Invalid tier/interval' }, { status: 400 });
    }
    
    // Create or retrieve Stripe Customer
    let customer;
    const { data: existingDonation } = await supabase
      .from('user_donations')
      .select('stripe_customer_id')
      .eq('user_id', userId)
      .single();
    
    if (existingDonation?.stripe_customer_id) {
      customer = await stripe.customers.retrieve(existingDonation.stripe_customer_id);
    } else {
      customer = await stripe.customers.create({
        email: user.email,
        name: user.full_name,
        metadata: {
          supabase_user_id: userId,
        },
      });
    }
    
    // Create Checkout Session
    const session = await stripe.checkout.sessions.create({
      customer: customer.id,
      mode: 'subscription',
      line_items: [
        {
          price: priceId,
          quantity: 1,
        },
      ],
      success_url: `${process.env.APP_URL}/donation/success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${process.env.APP_URL}/donation/cancel`,
      metadata: {
        user_id: userId,
        tier: tier,
      },
      subscription_data: {
        metadata: {
          user_id: userId,
          tier: tier,
        },
      },
      // Tax receipt support
      invoice_creation: {
        enabled: true,
      },
    });
    
    return NextResponse.json({ 
      sessionId: session.id,
      url: session.url 
    });
    
  } catch (error) {
    console.error('Checkout error:', error);
    return NextResponse.json({ error: 'Failed to create checkout' }, { status: 500 });
  }
}
```

### **2. Stripe Webhooks**

```typescript
// /api/webhooks/stripe
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';
import { buffer } from 'micro';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET!;

export async function POST(req: NextRequest) {
  const buf = await buffer(req);
  const sig = req.headers.get('stripe-signature')!;
  
  let event: Stripe.Event;
  
  try {
    event = stripe.webhooks.constructEvent(buf, sig, webhookSecret);
  } catch (err) {
    console.error('Webhook signature verification failed:', err);
    return NextResponse.json({ error: 'Invalid signature' }, { status: 400 });
  }
  
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_KEY!
  );
  
  // Handle different event types
  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;
      await handleCheckoutComplete(session, supabase);
      break;
    }
    
    case 'customer.subscription.created':
    case 'customer.subscription.updated': {
      const subscription = event.data.object as Stripe.Subscription;
      await handleSubscriptionUpdate(subscription, supabase);
      break;
    }
    
    case 'customer.subscription.deleted': {
      const subscription = event.data.object as Stripe.Subscription;
      await handleSubscriptionCanceled(subscription, supabase);
      break;
    }
    
    case 'invoice.paid': {
      const invoice = event.data.object as Stripe.Invoice;
      await handleInvoicePaid(invoice, supabase);
      break;
    }
    
    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice;
      await handlePaymentFailed(invoice, supabase);
      break;
    }
  }
  
  return NextResponse.json({ received: true });
}

async function handleCheckoutComplete(
  session: Stripe.Checkout.Session, 
  supabase: any
) {
  const userId = session.metadata?.user_id;
  const tier = session.metadata?.tier;
  
  if (!userId) return;
  
  // Create or update donation record
  await supabase.from('user_donations').upsert({
    user_id: userId,
    stripe_customer_id: session.customer,
    stripe_subscription_id: session.subscription,
    tier: tier,
    status: 'active',
    created_at: new Date().toISOString(),
  });
}

async function handleSubscriptionUpdate(
  subscription: Stripe.Subscription,
  supabase: any
) {
  const { data: donation } = await supabase
    .from('user_donations')
    .select('*')
    .eq('stripe_subscription_id', subscription.id)
    .single();
  
  if (!donation) return;
  
  await supabase
    .from('user_donations')
    .update({
      status: subscription.status,
      amount_cents: subscription.items.data[0].price.unit_amount,
      current_period_start: new Date(subscription.current_period_start * 1000),
      current_period_end: new Date(subscription.current_period_end * 1000),
      cancel_at_period_end: subscription.cancel_at_period_end,
      updated_at: new Date().toISOString(),
    })
    .eq('id', donation.id);
}

async function handleSubscriptionCanceled(
  subscription: Stripe.Subscription,
  supabase: any
) {
  await supabase
    .from('user_donations')
    .update({
      status: 'canceled',
      canceled_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
    })
    .eq('stripe_subscription_id', subscription.id);
}

async function handleInvoicePaid(
  invoice: Stripe.Invoice,
  supabase: any
) {
  const { data: donation } = await supabase
    .from('user_donations')
    .select('*')
    .eq('stripe_customer_id', invoice.customer)
    .single();
  
  if (!donation) return;
  
  // Generate custom invoice number
  const invoiceNumber = `LECTIO-${new Date().getFullYear()}-${donation.user_id.substring(0, 6).toUpperCase()}-${String(Math.floor(Math.random() * 9999)).padStart(4, '0')}`;
  
  // Store transaction
  await supabase.from('donation_transactions').insert({
    user_id: donation.user_id,
    donation_id: donation.id,
    stripe_invoice_id: invoice.id,
    stripe_payment_intent_id: invoice.payment_intent,
    amount_cents: invoice.amount_paid,
    currency: invoice.currency,
    status: 'paid',
    invoice_number: invoiceNumber,
    invoice_pdf_url: invoice.invoice_pdf,
    paid_at: new Date(invoice.status_transitions.paid_at! * 1000),
  });
  
  // Update last invoice on donation
  await supabase
    .from('user_donations')
    .update({
      last_invoice_number: invoiceNumber,
      updated_at: new Date().toISOString(),
    })
    .eq('id', donation.id);
  
  // Send tax receipt email (if applicable)
  // TODO: Implement email service
}

async function handlePaymentFailed(
  invoice: Stripe.Invoice,
  supabase: any
) {
  const { data: donation } = await supabase
    .from('user_donations')
    .select('*')
    .eq('stripe_customer_id', invoice.customer)
    .single();
  
  if (!donation) return;
  
  await supabase
    .from('user_donations')
    .update({
      status: 'past_due',
      updated_at: new Date().toISOString(),
    })
    .eq('id', donation.id);
  
  // Send payment failed email
  // TODO: Implement email notification
}
```

### **3. Get Donation Status**

```typescript
// /api/donations/status
export async function GET(req: NextRequest) {
  const userId = req.nextUrl.searchParams.get('userId');
  
  if (!userId) {
    return NextResponse.json({ error: 'Missing userId' }, { status: 400 });
  }
  
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_KEY!
  );
  
  const { data: donation } = await supabase
    .from('user_donations')
    .select(`
      *,
      donation_transactions(
        invoice_number,
        amount_cents,
        paid_at,
        invoice_pdf_url
      )
    `)
    .eq('user_id', userId)
    .eq('status', 'active')
    .single();
  
  if (!donation) {
    return NextResponse.json({ 
      hasDonation: false,
      tier: 'free' 
    });
  }
  
  return NextResponse.json({
    hasDonation: true,
    tier: donation.tier,
    status: donation.status,
    amount: donation.amount_cents / 100,
    currency: donation.currency,
    interval: donation.interval,
    currentPeriodEnd: donation.current_period_end,
    lastInvoiceNumber: donation.last_invoice_number,
    stripeCustomerId: donation.stripe_customer_id, // Pre user display
    transactions: donation.donation_transactions,
  });
}
```

### **4. Cancel Subscription**

```typescript
// /api/donations/cancel
export async function POST(req: NextRequest) {
  const { userId } = await req.json();
  
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_KEY!
  );
  
  const { data: donation } = await supabase
    .from('user_donations')
    .select('stripe_subscription_id')
    .eq('user_id', userId)
    .single();
  
  if (!donation?.stripe_subscription_id) {
    return NextResponse.json({ error: 'No active donation' }, { status: 404 });
  }
  
  // Cancel at period end (nie okamžite!)
  const subscription = await stripe.subscriptions.update(
    donation.stripe_subscription_id,
    { cancel_at_period_end: true }
  );
  
  return NextResponse.json({ 
    success: true,
    message: 'Donation will be canceled at period end',
    periodEnd: new Date(subscription.current_period_end * 1000),
  });
}
```

---

## 🌐 **WEB + MOBILE IMPLEMENTATION**

### **MÔŽEŠ PODPORIŤ AJ NA WEBE AJ V APKE! ✅**

**3 spôsoby:**
1. **Web** → Stripe Checkout (redirect)
2. **Mobile (webview)** → Otvorí Stripe Checkout v browseri (0% Apple/Google fee!)
3. **Mobile (native)** → Stripe Payment Sheet v apke (lepšia UX)

---

## 🌐 **1. WEB IMPLEMENTATION (Next.js)**

### **Frontend - Donation Button**

```typescript
// src/components/DonationButton.tsx
'use client';

import { useState } from 'react';

interface DonationButtonProps {
  tier: 'priatel' | 'patron' | 'zakladatel';
  interval: 'month' | 'year';
  userId: string;
}

export default function DonationButton({ tier, interval, userId }: DonationButtonProps) {
  const [loading, setLoading] = useState(false);
  
  const tierLabels = {
    priatel: '💙 Priateľ Lectio',
    patron: '💜 Patron Lectio',
    zakladatel: '🌟 Zakladateľ Lectio',
  };
  
  const prices = {
    priatel_year: '€30/rok',
    patron_month: '€20/mes',
    patron_year: '€200/rok',
    zakladatel_month: '€50/mes',
    zakladatel_year: '€500/rok',
  };
  
  const handleDonation = async () => {
    setLoading(true);
    
    try {
      // Call backend to create checkout session
      const response = await fetch('/api/donations/create-checkout', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ userId, tier, interval }),
      });
      
      const { url } = await response.json();
      
      // Redirect to Stripe Checkout
      window.location.href = url;
      
    } catch (error) {
      console.error('Donation error:', error);
      alert('Chyba pri vytváraní platby. Skúste znova.');
      setLoading(false);
    }
  };
  
  return (
    <button
      onClick={handleDonation}
      disabled={loading}
      className="w-full bg-blue-600 text-white py-3 px-6 rounded-lg font-semibold hover:bg-blue-700 disabled:opacity-50"
    >
      {loading ? 'Načítavam...' : `Podporiť - ${prices[`${tier}_${interval}`]}`}
    </button>
  );
}
```

### **Donation Page**

```typescript
// src/app/donation/page.tsx
import DonationButton from '@/components/DonationButton';
import { getUser } from '@/lib/auth'; // Your auth function

export default async function DonationPage() {
  const user = await getUser();
  
  if (!user) {
    return <div>Prosím prihláste sa</div>;
  }
  
  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-4xl font-bold text-center mb-4">
        🙏 Podporte Lectio.one
      </h1>
      
      <p className="text-center text-lg mb-8">
        Nie predávame vieru, zdieľame ju. Všetok obsah je ZADARMO.
      </p>
      
      <div className="grid md:grid-cols-3 gap-6">
        {/* Tier 2: Priateľ */}
        <div className="border rounded-lg p-6 shadow-lg">
          <h3 className="text-2xl font-bold mb-2">💙 Priateľ Lectio</h3>
          <p className="text-3xl font-bold text-blue-600 mb-4">€30/rok</p>
          <ul className="mb-6 space-y-2">
            <li>✅ Newsletter</li>
            <li>✅ E-book</li>
            <li>✅ 14 dní offline</li>
            <li>✅ Prioritná podpora</li>
          </ul>
          <DonationButton tier="priatel" interval="year" userId={user.id} />
        </div>
        
        {/* Tier 3: Patron */}
        <div className="border-2 border-purple-500 rounded-lg p-6 shadow-xl relative">
          <div className="absolute -top-3 left-1/2 transform -translate-x-1/2 bg-purple-500 text-white px-4 py-1 rounded-full text-sm font-semibold">
            Populárne!
          </div>
          <h3 className="text-2xl font-bold mb-2">💜 Patron Lectio</h3>
          <p className="text-3xl font-bold text-purple-600 mb-4">€20/mes</p>
          <ul className="mb-6 space-y-2">
            <li>✅ Všetky kurzy ZADARMO</li>
            <li>✅ Premium audio</li>
            <li>✅ Early access k videám</li>
            <li>✅ Fyzické dary</li>
          </ul>
          <DonationButton tier="patron" interval="month" userId={user.id} />
          <div className="mt-2">
            <DonationButton tier="patron" interval="year" userId={user.id} />
            <p className="text-sm text-center mt-1 text-gray-600">
              Ročne ušetríte €40!
            </p>
          </div>
        </div>
        
        {/* Tier 4: Zakladateľ */}
        <div className="border rounded-lg p-6 shadow-lg bg-gradient-to-br from-yellow-50 to-yellow-100">
          <h3 className="text-2xl font-bold mb-2">🌟 Zakladateľ</h3>
          <p className="text-3xl font-bold text-yellow-600 mb-4">€50/mes</p>
          <ul className="mb-6 space-y-2">
            <li>✅ LIFETIME ACCESS</li>
            <li>✅ Hlas v rozvoji</li>
            <li>✅ VIP dary</li>
            <li>✅ Founding Patron badge</li>
          </ul>
          <DonationButton tier="zakladatel" interval="month" userId={user.id} />
          <div className="mt-2">
            <DonationButton tier="zakladatel" interval="year" userId={user.id} />
            <p className="text-sm text-center mt-1 text-gray-600">
              Ročne ušetríte €100!
            </p>
          </div>
        </div>
      </div>
      
      {/* Trust badges */}
      <div className="mt-12 text-center">
        <p className="text-gray-600 mb-4">
          🔒 Bezpečné platby cez Stripe | � Všetky karty | 🇪🇺 SEPA
        </p>
        <p className="text-sm text-gray-500">
          100% darov ide na obsah a infraštruktúru. Žiadne reklamy. Žiadne predaje dát.
        </p>
      </div>
    </div>
  );
}
```

### **Success/Cancel Pages**

```typescript
// src/app/donation/success/page.tsx
export default function DonationSuccessPage() {
  return (
    <div className="max-w-2xl mx-auto p-6 text-center">
      <div className="text-6xl mb-4">🙏</div>
      <h1 className="text-4xl font-bold mb-4">Ďakujeme!</h1>
      <p className="text-xl mb-8">
        Vaša podpora pomáha tisícom ľudí nájsť Boha každý deň.
      </p>
      <p className="mb-8">
        Potvrdenie platby sme vám poslali emailom.
      </p>
      <a
        href="/"
        className="inline-block bg-blue-600 text-white py-3 px-8 rounded-lg font-semibold hover:bg-blue-700"
      >
        Späť na Lectio.one
      </a>
    </div>
  );
}

// src/app/donation/cancel/page.tsx
export default function DonationCancelPage() {
  return (
    <div className="max-w-2xl mx-auto p-6 text-center">
      <div className="text-6xl mb-4">😔</div>
      <h1 className="text-4xl font-bold mb-4">Platba zrušená</h1>
      <p className="text-xl mb-8">
        Nič sa nestalo. Môžete to skúsiť znova kedykoľvek.
      </p>
      <a
        href="/donation"
        className="inline-block bg-blue-600 text-white py-3 px-8 rounded-lg font-semibold hover:bg-blue-700"
      >
        Skúsiť znova
      </a>
    </div>
  );
}
```

---

## �📱 **2. FLUTTER IMPLEMENTATION**

### **Spôsob A: Webview (ODPORÚČAM! - 0% Apple/Google fee)**

```dart
// lib/screens/donation_screen.dart
import 'package:flutter/material.dart';
import 'package:url_launcher/url_launcher.dart';

class DonationScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Podporte Lectio.one')),
      body: SingleChildScrollView(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            Text(
              '🙏 Nie predávame vieru, zdieľame ju.',
              style: Theme.of(context).textTheme.headlineSmall,
              textAlign: TextAlign.center,
            ),
            SizedBox(height: 16),
            Text(
              'Všetok obsah je ZADARMO. Vaše dary nám pomáhajú udržiavať aplikáciu.',
              textAlign: TextAlign.center,
            ),
            SizedBox(height: 32),
            
            // Tier cards
            _buildTierCard(
              context,
              title: '💙 Priateľ Lectio',
              price: '€30/rok',
              benefits: ['Newsletter', 'E-book', '14 dní offline'],
              tier: 'priatel',
              interval: 'year',
            ),
            SizedBox(height: 16),
            
            _buildTierCard(
              context,
              title: '💜 Patron Lectio',
              price: '€20/mes',
              badge: 'Populárne!',
              benefits: ['Všetky kurzy', 'Premium audio', 'Fyzické dary'],
              tier: 'patron',
              interval: 'month',
            ),
            SizedBox(height: 16),
            
            _buildTierCard(
              context,
              title: '🌟 Zakladateľ Lectio',
              price: '€50/mes',
              benefits: ['LIFETIME ACCESS', 'Hlas v rozvoji', 'VIP'],
              tier: 'zakladatel',
              interval: 'month',
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildTierCard(
    BuildContext context, {
    required String title,
    required String price,
    required List<String> benefits,
    required String tier,
    required String interval,
    String? badge,
  }) {
    return Card(
      elevation: 4,
      child: Padding(
        padding: EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            if (badge != null)
              Container(
                padding: EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                decoration: BoxDecoration(
                  color: Colors.amber,
                  borderRadius: BorderRadius.circular(12),
                ),
                child: Text(badge, style: TextStyle(fontSize: 12, fontWeight: FontWeight.bold)),
              ),
            SizedBox(height: 8),
            Text(title, style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
            SizedBox(height: 8),
            Text(price, style: TextStyle(fontSize: 28, color: Colors.blue, fontWeight: FontWeight.bold)),
            SizedBox(height: 16),
            ...benefits.map((b) => Padding(
              padding: EdgeInsets.only(bottom: 8),
              child: Row(
                children: [
                  Icon(Icons.check_circle, color: Colors.green, size: 20),
                  SizedBox(width: 8),
                  Text(b),
                ],
              ),
            )).toList(),
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: () => _openDonationWeb(context, tier, interval),
              style: ElevatedButton.styleFrom(
                minimumSize: Size(double.infinity, 48),
              ),
              child: Text('Podporiť'),
            ),
          ],
        ),
      ),
    );
  }
  
  // WEBVIEW APPROACH (odporúčam - 0% Apple fee!)
  Future<void> _openDonationWeb(BuildContext context, String tier, String interval) async {
    final userId = 'YOUR_USER_ID'; // Get from auth
    final url = Uri.parse('https://lectiodivina.org/donation?tier=$tier&interval=$interval&userId=$userId');
    
    if (await canLaunchUrl(url)) {
      await launchUrl(
        url,
        mode: LaunchMode.externalApplication, // Otvorí v Safari/Chrome
      );
    } else {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Nepodarilo sa otvoriť webovú stránku')),
      );
    }
  }
}
```

**Dependencies:**
```yaml
# pubspec.yaml
dependencies:
  url_launcher: ^6.2.1 # Pre otvorenie webu
```

### **Spôsob B: Native Stripe (lepšia UX, ale zložitejšie)**

```yaml
# pubspec.yaml
dependencies:
  flutter_stripe: ^10.1.1
  http: ^1.1.0
```

### **2. Initialize Stripe**

```dart
// lib/main.dart
import 'package:flutter_stripe/flutter_stripe.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Initialize Stripe
  Stripe.publishableKey = dotenv.env['STRIPE_PUBLISHABLE_KEY']!;
  Stripe.merchantIdentifier = 'merchant.com.lectiodivina';
  
  await Stripe.instance.applySettings();
  
  runApp(MyApp());
}
```

### **3. Donation Service**

```dart
// lib/services/donation_service.dart
import 'package:flutter_stripe/flutter_stripe.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';

class DonationService {
  final String baseUrl = 'https://your-backend.com/api';
  
  // Start checkout flow
  Future<void> startDonation({
    required String userId,
    required String tier,
    required String interval, // 'month' or 'year'
  }) async {
    try {
      // 1. Create checkout session on backend
      final response = await http.post(
        Uri.parse('$baseUrl/donations/create-checkout'),
        headers: {'Content-Type': 'application/json'},
        body: json.encode({
          'userId': userId,
          'tier': tier,
          'interval': interval,
        }),
      );
      
      if (response.statusCode != 200) {
        throw Exception('Failed to create checkout session');
      }
      
      final data = json.decode(response.body);
      final sessionId = data['sessionId'];
      
      // 2. Present Stripe checkout
      await Stripe.instance.presentPaymentSheet();
      
      // 3. Success!
      print('Donation successful!');
      
    } on StripeException catch (e) {
      print('Stripe error: ${e.error.localizedMessage}');
      rethrow;
    } catch (e) {
      print('Error: $e');
      rethrow;
    }
  }
  
  // Get donation status
  Future<DonationStatus> getDonationStatus(String userId) async {
    final response = await http.get(
      Uri.parse('$baseUrl/donations/status?userId=$userId'),
    );
    
    if (response.statusCode == 200) {
      final data = json.decode(response.body);
      return DonationStatus.fromJson(data);
    }
    
    throw Exception('Failed to get donation status');
  }
  
  // Cancel donation
  Future<void> cancelDonation(String userId) async {
    final response = await http.post(
      Uri.parse('$baseUrl/donations/cancel'),
      headers: {'Content-Type': 'application/json'},
      body: json.encode({'userId': userId}),
    );
    
    if (response.statusCode != 200) {
      throw Exception('Failed to cancel donation');
    }
  }
}

class DonationStatus {
  final bool hasDonation;
  final String tier;
  final String status;
  final double amount;
  final String interval;
  final DateTime? currentPeriodEnd;
  final String? lastInvoiceNumber;
  final String? stripeCustomerId;
  
  DonationStatus({
    required this.hasDonation,
    required this.tier,
    required this.status,
    required this.amount,
    required this.interval,
    this.currentPeriodEnd,
    this.lastInvoiceNumber,
    this.stripeCustomerId,
  });
  
  factory DonationStatus.fromJson(Map<String, dynamic> json) {
    return DonationStatus(
      hasDonation: json['hasDonation'] ?? false,
      tier: json['tier'] ?? 'free',
      status: json['status'] ?? '',
      amount: (json['amount'] ?? 0).toDouble(),
      interval: json['interval'] ?? '',
      currentPeriodEnd: json['currentPeriodEnd'] != null
          ? DateTime.parse(json['currentPeriodEnd'])
          : null,
      lastInvoiceNumber: json['lastInvoiceNumber'],
      stripeCustomerId: json['stripeCustomerId'],
    );
  }
}
```

### **4. Donation Screen**

```dart
// lib/screens/donation_screen.dart
class DonationScreen extends StatelessWidget {
  final DonationService _donationService = DonationService();
  final String userId;
  
  DonationScreen({required this.userId});
  
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Podporte Lectio.one')),
      body: SingleChildScrollView(
        padding: EdgeInsets.all(16),
        child: Column(
          children: [
            // Header
            Text(
              '🙏 Nie predávame vieru, zdieľame ju.',
              style: Theme.of(context).textTheme.headlineSmall,
              textAlign: TextAlign.center,
            ),
            SizedBox(height: 16),
            Text(
              'Všetok obsah je ZADARMO. Vaše dary nám pomáhajú udržiavať aplikáciu a vytvárať nový obsah.',
              textAlign: TextAlign.center,
            ),
            SizedBox(height: 32),
            
            // Tier 2: Priateľ
            _buildTierCard(
              context,
              title: '💙 Priateľ Lectio',
              price: '€30/rok',
              description: 'Newsletter, e-book, 14 dní offline',
              onTap: () => _donate('priatel', 'year'),
            ),
            
            // Tier 3: Patron
            _buildTierCard(
              context,
              title: '💜 Patron Lectio',
              price: '€20/mesiac',
              badge: 'Populárne!',
              description: 'Všetky kurzy ZADARMO, premium audio, fyzické dary',
              onTap: () => _donate('patron', 'month'),
            ),
            
            // Tier 4: Zakladateľ
            _buildTierCard(
              context,
              title: '🌟 Zakladateľ Lectio',
              price: '€50/mesiac',
              description: 'LIFETIME ACCESS, hlas v rozvoji, VIP benefity',
              onTap: () => _donate('zakladatel', 'month'),
            ),
          ],
        ),
      ),
    );
  }
  
  Widget _buildTierCard(
    BuildContext context, {
    required String title,
    required String price,
    required String description,
    required VoidCallback onTap,
    String? badge,
  }) {
    return Card(
      margin: EdgeInsets.only(bottom: 16),
      child: InkWell(
        onTap: onTap,
        child: Padding(
          padding: EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text(title, style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
                  if (badge != null)
                    Chip(label: Text(badge), backgroundColor: Colors.amber),
                ],
              ),
              SizedBox(height: 8),
              Text(price, style: TextStyle(fontSize: 24, color: Colors.blue, fontWeight: FontWeight.bold)),
              SizedBox(height: 8),
              Text(description),
              SizedBox(height: 16),
              ElevatedButton(
                onPressed: onTap,
                child: Text('Vybrať'),
                style: ElevatedButton.styleFrom(
                  minimumSize: Size(double.infinity, 48),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
  
  Future<void> _donate(String tier, String interval) async {
    try {
      await _donationService.startDonation(
        userId: userId,
        tier: tier,
        interval: interval,
      );
      
      // Show success
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Ďakujeme za vašu podporu! 🙏')),
      );
      
      // Navigate to success page
      Navigator.pushNamed(context, '/donation-success');
      
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Chyba: $e')),
      );
    }
  }
}
```

---

## 💰 **STRIPE FEES**

### **Štandardné fees (bez NGO status):**

| Typ platby | Fee |
|-----------|-----|
| EU karty | **2.9% + €0.30** |
| Non-EU karty | 3.9% + €0.30 |
| SEPA Direct Debit | 0.8% (max €5) |
| Apple Pay / Google Pay | 2.9% + €0.30 |

### **Príklady:**

| Donation | Stripe fee | Netto |
|----------|-----------|-------|
| €30/rok | €1.17 | €28.83 |
| €20/mes (€240/rok) | €7.26/rok | €232.74 |
| €50/mes (€600/rok) | €18.15/rok | €581.85 |

### **S NGO status (Stripe Donations):**

| Donation | Fee (1.5% + €0.25) | Netto |
|----------|-------------------|-------|
| €30/rok | €0.70 | €29.30 |
| €240/rok | €3.85 | €236.15 |
| €600/rok | €9.25 | €590.75 |

**💡 Odporúčanie:** Registruj sa ako NGO/o.z. čo najskôr → ušetríš ~50% na fees!

---

## 📧 **VARIABILNÝ SYMBOL V PRAXI**

### **Kde user uvidí variabilný symbol:**

1. **V aplikácii (profil):**
```
👤 Môj profil
   💳 Donation Status: Active
   💜 Tier: Patron Lectio
   💰 €20/mesiac
   
   📋 Váš donation ID: 
   LECTIO-2025-ABC123-0042
   
   📄 Posledná platba:
   - Dátum: 1. november 2025
   - Suma: €20.00
   - Invoice: LECTIO-2025-ABC123-0042
   [Stiahnuť PDF]
```

2. **V Stripe invoice (email):**
```
Invoice from Lectio.one

Invoice Number: LECTIO-2025-ABC123-0042
Date: November 1, 2025
Amount Paid: €20.00

Customer ID: cus_ABC123XYZ
Subscription: Patron Lectio - Mesačný dar
```

3. **V tax receipte (daňový doklad):**
```
DAŇOVÝ DOKLAD O DARE

Darca: Dušan Pecko
Číslo daru: LECTIO-2025-ABC123-0042
Dátum: 1.11.2025
Suma: €20.00

Tento dar je daňovo odpočítateľný podľa § 50 zákona o dani z príjmov.
```

### **Stripe Customer ID vs Invoice Number:**

| ID Type | Príklad | Použitie |
|---------|---------|----------|
| **Stripe Customer ID** | `cus_ABC123XYZ` | Internal tracking, Stripe API calls |
| **Subscription ID** | `sub_DEF456GHI` | Subscription management |
| **Invoice Number** | `LECTIO-2025-ABC123-0042` | **Variabilný symbol**, user-facing, tax receipts |

---

## 🏆 **ODPORÚČANIE: WEB + MOBILE WEBVIEW**

### **Prečo webview approach:**

✅ **0% Apple/Google commission** (najdôležitejšie!)  
✅ **Jednoduchšia implementácia** (1 Stripe kód pre web aj mobile)  
✅ **Compliance** s Apple/Google policies (external payments)  
✅ **Stripe riešenia PCI** security  
✅ **Rýchlejší vývoj** (3-4 dni vs 7-10 dní)  
✅ **Jeden codebase** pre všetky platformy  

### **Apple/Google Fee Comparison:**

| Approach | Apple/Google Fee | Stripe Fee | Total Fee |
|----------|------------------|-----------|-----------|
| **In-App Purchase** | 30% | 0% | **30%** � |
| **Stripe Native** | 15% (workaround) | 2.9% | **~18%** |
| **Webview** | **0%** ✅ | 2.9% | **2.9%** 🎉 |

**Príklad:**  
€20 donation → In-App: €14 netto | Webview: €19.12 netto  
**Rozdiel: +€5.12 (36% viac!)** �🚀

---

## 🚀 **IMPLEMENTATION CHECKLIST**

### **Backend (Next.js):**
- [ ] Setup Stripe account
- [ ] Get API keys (publishable + secret)
- [ ] Create Stripe Products & Prices (5 tiers)
- [ ] Setup webhook endpoint (`/api/webhooks/stripe`)
- [ ] Implement `/api/donations/create-checkout`
- [ ] Implement `/api/webhooks/stripe`
- [ ] Implement `/api/donations/status`
- [ ] Implement `/api/donations/cancel`
- [ ] Create Supabase tables (`user_donations`, `donation_transactions`)
- [ ] Setup RLS policies
- [ ] Test webhook events (use Stripe CLI)

### **Frontend - Web (Next.js):**
- [ ] Create `/donation` page with tier cards
- [ ] Create `DonationButton` component
- [ ] Create `/donation/success` page
- [ ] Create `/donation/cancel` page
- [ ] Add donation status to user profile
- [ ] Display invoice history
- [ ] Add cancel donation UI

### **Frontend - Mobile (Flutter):**
- [ ] Add `url_launcher` package
- [ ] Create `DonationScreen` with tier cards
- [ ] Implement webview launch to web donation page
- [ ] Add deep link handling for success/cancel redirects
- [ ] Show donation status in profile
- [ ] Add badge display (💙💜🌟)

### **Testing:**
- [ ] Test web checkout flow (sandbox)
- [ ] Test mobile → web redirect
- [ ] Test webhooks (Stripe CLI: `stripe listen --forward-to localhost:3000/api/webhooks/stripe`)
- [ ] Test subscription updates
- [ ] Test cancellations
- [ ] Test invoice generation
- [ ] Test deep links back to app
- [ ] Test on iOS (Safari)
- [ ] Test on Android (Chrome)

### **Go-live:**
- [ ] Switch to production API keys
- [ ] Setup production webhooks (`https://lectiodivina.org/api/webhooks/stripe`)
- [ ] Enable live mode in Stripe
- [ ] Monitor first transactions
- [ ] Setup alerts for failed payments (Stripe Dashboard)
- [ ] Add analytics tracking (conversion rates)

---

## 🔗 **DEEP LINKING (Mobile → Web → Back to App)**

### **Success/Cancel Redirects:**

```typescript
// Backend: Create checkout session with mobile-aware URLs
const session = await stripe.checkout.sessions.create({
  // ... other config
  success_url: isMobile(req) 
    ? `lectio://donation/success?session_id={CHECKOUT_SESSION_ID}` // Deep link
    : `${process.env.APP_URL}/donation/success?session_id={CHECKOUT_SESSION_ID}`, // Web
  cancel_url: isMobile(req)
    ? `lectio://donation/cancel`
    : `${process.env.APP_URL}/donation/cancel`,
});

function isMobile(req: NextRequest) {
  const userAgent = req.headers.get('user-agent') || '';
  return /mobile|android|iphone/i.test(userAgent);
}
```

### **Flutter Deep Link Setup:**

```yaml
# android/app/src/main/AndroidManifest.xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="lectio" />
</intent-filter>
```

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleTypeRole</key>
    <string>Editor</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>lectio</string>
    </array>
  </dict>
</array>
```

```dart
// lib/main.dart
import 'package:uni_links/uni_links.dart';

void initDeepLinks() async {
  // Handle initial link if app was closed
  final initialLink = await getInitialLink();
  if (initialLink != null) {
    handleDeepLink(initialLink);
  }
  
  // Handle links while app is running
  linkStream.listen((String? link) {
    if (link != null) {
      handleDeepLink(link);
    }
  });
}

void handleDeepLink(String link) {
  final uri = Uri.parse(link);
  
  if (uri.path == '/donation/success') {
    // Show success screen
    Navigator.pushNamed(context, '/donation-success');
    
    // Refresh user donation status
    _refreshDonationStatus();
  } else if (uri.path == '/donation/cancel') {
    // Show cancel message
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text('Platba zrušená')),
    );
  }
}
```

**Dependencies:**
```yaml
# pubspec.yaml
dependencies:
  uni_links: ^0.5.1 # Deep linking
```

---

## 🔗 **ENVIRONMENT VARIABLES**

```bash
# .env.local (backend)
STRIPE_SECRET_KEY=sk_live_xxxxxxxxxxxxx
STRIPE_PUBLISHABLE_KEY=pk_live_xxxxxxxxxxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxxxxxxxxxx

# Stripe Price IDs (create in Stripe Dashboard)
STRIPE_PRICE_FRIEND_YEARLY=price_xxxxxxxxxxxxx
STRIPE_PRICE_PATRON_MONTHLY=price_xxxxxxxxxxxxx
STRIPE_PRICE_PATRON_YEARLY=price_xxxxxxxxxxxxx
STRIPE_PRICE_FOUNDER_MONTHLY=price_xxxxxxxxxxxxx
STRIPE_PRICE_FOUNDER_YEARLY=price_xxxxxxxxxxxxx

SUPABASE_URL=https://xxxxxxxxxxxxx.supabase.co
SUPABASE_SERVICE_KEY=xxxxxxxxxxxxx
APP_URL=https://lectiodivina.org
```

```bash
# .env (Flutter)
STRIPE_PUBLISHABLE_KEY=pk_live_xxxxxxxxxxxxx
BACKEND_URL=https://your-backend.com
```

---

## 📊 **DASHBOARD (Admin)**

### **Stripe Dashboard má:**

✅ Real-time revenue tracking  
✅ Customer list s vyhľadávaním  
✅ Subscription management  
✅ Invoice generation  
✅ Payment failure alerts  
✅ Refund handling  
✅ Export do CSV/Excel  
✅ Tax reporting  
✅ Churn analysis  

**Screenshot view:**
```
📊 Revenue Today: €245.00
👥 Active Donors: 127
💰 MRR: €2,540
📈 Growth: +12% this month

Recent Transactions:
- Dušan P. - €20.00 - Patron (Successful)
- Jana K. - €30.00 - Priateľ (Successful)
- Martin S. - €50.00 - Zakladateľ (Successful)
```

---

## ✅ **SÚHRN: ÁNO, STRIPE JE PERFEKTNÝ!**

### **Čo Stripe poskytuje:**

✅ **Donation produkt** (alebo subscription ako donation)  
✅ **Každý user má Customer ID** (`cus_ABC123XYZ`)  
✅ **Každá subscription má ID** (`sub_DEF456GHI`)  
✅ **Každá platba má Invoice Number** (variabilný symbol: `LECTIO-2025-ABC123-0042`)  
✅ **Recurring payments** (mesačné/ročné)  
✅ **Webhooks** pre sync s databázou  
✅ **Tax receipts** (PDF invoices)  
✅ **PCI compliance** (bezpečnosť)  
✅ **Multiple payment methods** (karty, SEPA, Apple/Google Pay)  
✅ **Dashboard** pre admin  
✅ **Cancellation handling**  
✅ **Failed payment recovery**  
✅ **WEB + MOBILE support** (jeden kód, všade)  

### **Fees:**
- 2.9% + €0.30 (štandardné)
- 1.5% + €0.25 (s NGO statusom)

### **🏆 ODPORÚČANÝ APPROACH:**

**Webview (mobile → web) = 0% Apple/Google fee!**

| Donation | Stripe Fee | Apple Fee | Netto |
|----------|-----------|-----------|-------|
| €20/mes | €0.88 | **€0** ✅ | **€19.12** |
| €30/rok | €1.17 | **€0** ✅ | **€28.83** |
| €50/mes | €1.75 | **€0** ✅ | **€48.25** |

**vs In-App Purchase:**

| Donation | Stripe Fee | Apple Fee (30%) | Netto |
|----------|-----------|-----------------|-------|
| €20/mes | €0 | €6.00 😱 | **€14.00** |

**→ Webview = +36% viac netto! 🚀**

### **Implementation Timeline:**

| Task | Time | Platform |
|------|------|----------|
| Stripe setup | 1 deň | Backend |
| API endpoints | 2 dni | Backend |
| Web UI | 2 dni | Web |
| Mobile webview | 1 deň | Mobile |
| Testing | 1 deň | All |
| **TOTAL** | **7 dní** | 🚀 |

### **Next steps:**
1. ✅ Vytvor Stripe account
2. ✅ Implementuj web donation page (2 dni)
3. ✅ Implementuj backend API (2 dni)
4. ✅ Pridaj webview do Flutter app (1 deň)
5. ✅ Setup webhooks a testing (1 deň)
6. ✅ Setup deep links (1 deň)
7. **LAUNCH! 🚀** (7 dní total)

---

**Verzia:** 1.0  
**Platform:** Stripe Subscriptions  
**Estimated development time:** 7-10 dní  
**Dátum:** 1. november 2025
