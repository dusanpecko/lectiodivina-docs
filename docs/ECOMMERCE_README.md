# 🛒 E-Commerce & Donation Platform for Lectio.one

Complete e-commerce and subscription/donation platform built with Next.js 15, Stripe, and Supabase.

## 📦 Features

### Physical Products Shop
- Product listing with categories (Books, Pens, Keychains, Journals, Calendars)
- Product detail pages
- Shopping cart (localStorage-based)
- Stripe Checkout integration
- Order management
- Inventory tracking
- Multi-language support (SK, EN, CZ, ES)

### Subscriptions & Donations
- **Subscription Tiers:**
  - Free (€0/month)
  - Supporter (€3/month)
  - Patron (€20/month)
  - Benefactor (€50/month)
  - Custom amount option

- **One-time Donations:**
  - Predefined amounts (€5, €10, €25, €50, €100)
  - Custom amount input
  - Optional message

### User Dashboard
- View active subscription
- Order history
- Donation history
- Manage subscription (upgrade/downgrade/cancel)

### Admin Panel
- Product management (CRUD)
- Order fulfillment
- Subscription overview
- Donation tracking
- Basic analytics

## 🚀 Setup Instructions

### 1. Database Setup

Run the SQL schema in Supabase SQL Editor:

```bash
# File: /backend/sql/e-commerce-schema.sql
```

This creates the following tables:
- `products` - Product catalog
- `orders` - Customer orders
- `order_items` - Line items for orders
- `subscriptions` - User subscriptions
- `donations` - One-time donations

### 2. Environment Variables

Create or update `.env.local`:

```env
# Existing Supabase variables
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key

# Stripe Configuration
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...

# Stripe Price IDs (create these in Stripe Dashboard)
STRIPE_PRICE_SUPPORTER=price_...
STRIPE_PRICE_PATRON=price_...
STRIPE_PRICE_BENEFACTOR=price_...

# Base URL
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

### 3. Create Stripe Products

1. Go to [Stripe Dashboard](https://dashboard.stripe.com/)
2. Navigate to **Products** → **Add product**
3. Create three subscription products:

**Supporter Subscription**
- Name: Lectio Divina Supporter
- Price: €3/month
- Copy the Price ID to `STRIPE_PRICE_SUPPORTER`

**Patron Subscription**
- Name: Lectio Divina Patron
- Price: €20/month
- Copy the Price ID to `STRIPE_PRICE_PATRON`

**Benefactor Subscription**
- Name: Lectio Divina Benefactor
- Price: €50/month
- Copy the Price ID to `STRIPE_PRICE_BENEFACTOR`

### 4. Configure Stripe Webhook

1. Install Stripe CLI: https://stripe.com/docs/stripe-cli
2. Test webhook locally:

```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

3. Copy the webhook secret to `STRIPE_WEBHOOK_SECRET`

4. For production, add webhook endpoint in Stripe Dashboard:
   - URL: `https://yourdomain.com/api/webhooks/stripe`
   - Events to listen to:
     - `checkout.session.completed`
     - `customer.subscription.updated`
     - `customer.subscription.deleted`
     - `invoice.payment_failed`

### 5. Install Dependencies

```bash
cd backend
npm install
```

New dependencies added:
- `stripe` - Stripe Node.js SDK
- `@stripe/stripe-js` - Stripe client SDK

### 6. Run Development Server

```bash
npm run dev
```

Visit:
- Shop: http://localhost:3000/shop
- Support: http://localhost:3000/support
- Cart: http://localhost:3000/cart

## 📁 Project Structure

```
backend/
├── sql/
│   └── e-commerce-schema.sql          # Database schema
├── src/
│   ├── app/
│   │   ├── shop/
│   │   │   └── page.tsx               # Product listing
│   │   ├── cart/
│   │   │   └── page.tsx               # Shopping cart
│   │   ├── checkout/
│   │   │   └── page.tsx               # Checkout form (TODO)
│   │   ├── support/
│   │   │   └── page.tsx               # Subscriptions & donations
│   │   ├── account/
│   │   │   └── page.tsx               # User dashboard (TODO)
│   │   ├── admin/
│   │   │   └── ...                    # Admin panel (TODO)
│   │   ├── thank-you/
│   │   │   └── page.tsx               # Post-checkout page
│   │   └── api/
│   │       ├── checkout/
│   │       │   ├── subscription/route.ts
│   │       │   └── donation/route.ts
│   │       └── webhooks/
│   │           └── stripe/route.ts    # Stripe webhook handler
│   ├── lib/
│   │   └── stripe.ts                  # Stripe utilities
│   └── types/
│       └── ecommerce.ts               # TypeScript types
```

## 🔧 API Endpoints

### Checkout
- `POST /api/checkout/subscription` - Create subscription checkout
- `POST /api/checkout/donation` - Create donation checkout
- `POST /api/checkout/product` - Create product checkout (TODO)

### Webhooks
- `POST /api/webhooks/stripe` - Handle Stripe events

## 🎨 Pages

### Public Pages
- `/shop` - Browse products
- `/shop/[slug]` - Product detail (TODO)
- `/cart` - Shopping cart
- `/checkout` - Checkout process (TODO)
- `/support` - Subscriptions & donations
- `/thank-you` - Confirmation page

### Authenticated Pages
- `/account` - User dashboard (TODO)
- `/admin` - Admin panel (TODO)

## 🔐 Security

### Row Level Security (RLS)
All tables have RLS enabled with policies:
- Products: Public read, admin write
- Orders: Users can view their own, admins view all
- Subscriptions: Users can view their own
- Donations: Users can view their own

### Admin Access
To make a user an admin, update their metadata in Supabase:

```sql
UPDATE auth.users
SET raw_user_meta_data = jsonb_set(
  COALESCE(raw_user_meta_data, '{}'::jsonb),
  '{role}',
  '"admin"'
)
WHERE email = 'admin@example.com';
```

## 🧪 Testing

### Test Mode
Stripe is in test mode by default. Use test cards:
- Success: `4242 4242 4242 4242`
- Decline: `4000 0000 0000 0002`

### Test Webhook Locally

```bash
stripe trigger checkout.session.completed
```

## 📝 TODO

- [ ] Product detail page `/shop/[slug]`
- [ ] Checkout page with shipping address form
- [ ] User account page with order/subscription history
- [ ] Admin panel for product and order management
- [ ] Email notifications (order confirmation, receipts)
- [ ] Product images upload
- [ ] Search and filtering
- [ ] Reviews and ratings
- [ ] Shipping cost calculation
- [ ] Multi-currency support

## 🐛 Troubleshooting

### Stripe Webhook Not Working
1. Check webhook secret is correct
2. Verify webhook URL is accessible
3. Check Stripe CLI logs: `stripe listen`

### Products Not Loading
1. Verify products exist in database
2. Check `is_active = true`
3. Verify RLS policies are correct

### Checkout Fails
1. Check Stripe keys are valid
2. Verify Price IDs match Stripe Dashboard
3. Check browser console for errors

## 📚 Documentation

- [Stripe Checkout Docs](https://stripe.com/docs/payments/checkout)
- [Stripe Subscriptions](https://stripe.com/docs/billing/subscriptions/overview)
- [Supabase RLS](https://supabase.com/docs/guides/auth/row-level-security)
- [Next.js App Router](https://nextjs.org/docs/app)

## 🤝 Support

For issues or questions, contact: admin@lectio.one

---

Built with ❤️ for Lectio Divina community
