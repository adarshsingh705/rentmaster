# RentMaster Phase 1 — Service Integration & API Specifications
**Date:** 2026-06-12  
**Scope:** Complete list of external services, APIs, and infrastructure for Phase 1 MVP launch  
**Pricing:** All costs shown WITHOUT GST (add 18% GST later as per compliance)  
**Phase Duration:** Week 1–8 (Phase 0 foundation + Phase 1–2 core MVP)

---

## Overview

Phase 1 includes foundational infrastructure (Weeks 1–2), core CRUD + dashboard (Weeks 3–6), and automation with WhatsApp reminders (Weeks 7–8). Below is every external service with pricing, features, and implementation notes.

---

## 1. Supabase (Database + Auth + Realtime)

| Property | Details |
|---|---|
| **Purpose** | PostgreSQL database, phone OTP auth, row-level security, realtime webhooks, file storage |
| **Tier** | Free (MVP) → Pro ($25/month) when exceeding free limits |
| **Free Tier Includes** | 500MB DB, 2GB file storage, unlimited auth users, realtime (limited) |
| **Monthly Cost (Free)** | ₹0 |
| **Monthly Cost (Pro)** | ~₹2,100 (₹25 × 84 INR/USD) |
| **Annual Cost (Free)** | ₹0 |
| **Annual Cost (Pro)** | ~₹25,200 |

### Phase 1 Features Used

- **Database**: `owners`, `properties`, `rooms`, `tenants`, `rent_records`, `notification_log`, `owner_payment_settings` tables with RLS policies
- **Auth**: Phone OTP login (owner registration/login)
- **Realtime**: `postgres_changes` subscriptions for instant dashboard updates when rent status changes
- **Storage**: Tenant KYC docs (Aadhaar scans, selfies) — encrypted storage

### Critical Integration Points

```javascript
// Environment variables needed:
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx...
SUPABASE_SERVICE_ROLE_KEY=eyJxxx... (server-side only, never in frontend)
```

### Upgrade Path

- Free tier sufficient for first 500 owners (500MB DB = ~500K rent records)
- Upgrade to Pro ($2,100/month) when approaching DB limit around Month 3

---

## 2. Vercel (Frontend Hosting + API Routes)

| Property | Details |
|---|---|
| **Purpose** | Hosting Next.js 14 frontend, simple CRUD API routes, CDN delivery |
| **Tier** | Free (MVP) → Pro ($20/month) if needed for additional features |
| **Free Tier Includes** | Unlimited deployments, serverless functions (50GB bandwidth/month), Git auto-deploy |
| **Monthly Cost (Free)** | ₹0 |
| **Monthly Cost (Pro)** | ~₹1,680 (₹20 × 84 INR/USD) |
| **Annual Cost (Free)** | ₹0 |
| **Annual Cost (Pro)** | ~₹20,160 |

### Phase 1 Features Used

- Git push deploy (automatic on every commit to `main`)
- Serverless API routes: `/api/auth`, `/api/owner/*`, `/api/webhooks/*` (payment/WhatsApp)
- Environment variables per deployment (dev/prod)
- Automatic HTTPS + CDN caching

### Critical Integration Points

```javascript
// vercel.json config
{
  "env": {
    "NEXT_PUBLIC_SUPABASE_URL": "@supabase_url",
    "NEXT_PUBLIC_SUPABASE_ANON_KEY": "@supabase_anon_key",
    "PG_CLIENT_ID": "@pg_client_id",  // Cashfree (Phase 3)
    "PG_CLIENT_SECRET": "@pg_client_secret",
    "INTERAKT_API_KEY": "@interakt_api_key",  // WhatsApp (Phase 2)
  }
}
```

### Known Limitations (Why Render.com is Added)

- **No persistent background processes**: Vercel functions time out after 60 seconds (10 seconds on free tier)
- **No cron jobs**: Paid feature on Vercel ($150/month for cron); free on Render.com
- **No websockets**: Can't run persistent BullMQ workers; need separate service

---

## 3. Render.com (Background Worker + Cron)

| Property | Details |
|---|---|
| **Purpose** | Always-on Node.js process for BullMQ job scheduler, monthly rent record generation |
| **Tier** | Free (MVP with occasional restarts) → Starter ($7/month for reliability) |
| **Free Tier** | Up to 0.5 CPU, restarts every 15 minutes of inactivity |
| **Starter Tier** | 0.5 CPU, always-on, $7/month |
| **Monthly Cost (Free)** | ₹0 (acceptable for MVP with BullMQ Redis persistence) |
| **Monthly Cost (Starter)** | ~₹588 (₹7 × 84) |
| **Annual Cost (Free)** | ₹0 |
| **Annual Cost (Starter)** | ~₹7,056 |

### Phase 1 Features Used

- Background worker process (Node.js + Express)
- BullMQ job queue management
- Cron jobs (daily at 7 AM IST for rent reminders, monthly for rent record generation)
- Redis connection for job persistence

### Critical Integration Points

```javascript
// Environment variables for Render worker:
DATABASE_URL=postgresql://...  // Supabase connection
UPSTASH_REDIS_URL=redis://...  // Redis instance
INTERAKT_API_KEY=...           // WhatsApp API

// Render deploy script (package.json):
"scripts": {
  "start": "node workers/scheduler.js"
}
```

### Cost Optimization Strategy

- Start with **free tier** (restarts acceptable because BullMQ Redis persistence survives)
- Upgrade to **Starter** ($588/month) after 2+ months when reliability is critical
- At 1,000+ active tenants, upgrade to **Standard** ($12/month)

---

## 4. Upstash Redis (Job Queue Persistence)

| Property | Details |
|---|---|
| **Purpose** | Persistent Redis for BullMQ job queue, rent reminder scheduling |
| **Tier** | Free (MVP) → Pay-as-you-go ($0.2 per 100K commands above free tier) |
| **Free Tier** | 10,000 commands/day = ~300,000 commands/month |
| **MVP Estimation** | 500 owners × 2 reminders/month × 50 ops/reminder = 50K commands/month (within free) |
| **Monthly Cost (Free)** | ₹0 |
| **Monthly Cost (Pay-as-go)** | ~₹168 (₹200K commands at $0.2/100K) |
| **Annual Cost (Free)** | ₹0 |
| **Annual Cost (Pay-as-go)** | ~₹2,000 |

### Phase 1 Features Used

- BullMQ queue backend (all reminder jobs queued here)
- Job persistence (survives Render.com free-tier restarts)
- Automatic retry on failure
- Rate limiting per owner (max 2 reminders/day per tenant)

### Critical Integration Points

```javascript
import { Queue } from 'bullmq';
const redis = new Redis(process.env.UPSTASH_REDIS_URL, {
  tls: { rejectUnauthorized: false }  // Upstash requires TLS
});
const reminderQueue = new Queue('rent-reminders', { connection: redis });
```

### Monitoring

- Upstash dashboard shows command count in real-time
- Alert: upgrade to Standard tier ($18/month) if approaching 500K commands/month

---

## 5. Interakt (WhatsApp Business API via Meta BSP)

| Property | Details |
|---|---|
| **Purpose** | Send automated WhatsApp messages (reminders, receipts) to tenants, handle incoming replies |
| **Tier** | Standard monthly plan (₹999/month fixed) |
| **Included** | Unlimited message templates, unlimited WhatsApp numbers (for owners), message delivery tracking |
| **Monthly Cost** | ₹999 (fixed, no overage) |
| **Annual Cost** | ₹11,988 |
| **Setup Fee** | ₹0 (free account creation) |

### Phase 1 Features Used (Phase 2 Timeline)

- **6 pre-approved templates** (submitted by Week 1, approved by Week 7):
  - `rent_reminder_3days`: Due in 3 days, include payment link
  - `rent_reminder_due_today`: Due today, urgent
  - `rent_receipt`: Payment received confirmation with amount/date
  - `rent_overdue`: 5+ days overdue, payment link + penalty info
  - `tenant_welcome`: New tenant onboarding
  - `agreement_signed`: eSign completion notification

- **Message sending**: Via Interakt API endpoint (all messages are templated, no freeform)
- **Webhook**: Incoming messages + delivery status updates sent to your backend
- **Analytics**: Message delivery rate, read rate, failure reasons

### Critical Integration Points

```javascript
// Environment variables:
INTERAKT_API_KEY=xxx  // Bearer token for API

// API endpoint:
const INTERAKT_URL = 'https://api.interakt.ai/v1/public/message/';

// Sending a message:
await fetch(INTERAKT_URL, {
  method: 'POST',
  headers: { 'Authorization': `Basic ${btoa(API_KEY)}` },
  body: JSON.stringify({
    countryCode: '+91',
    phoneNumber: '9876543210',
    type: 'Template',
    template: {
      name: 'rent_reminder_3days',
      languageCode: 'en',
      bodyValues: ['Raj', '₹10,000', 'June 2026', 'https://pay.link...']
    }
  })
});
```

### Cost Optimization

- ₹999/month is **fixed** — price does NOT increase with message volume
- At 500 owners × 2 reminders/month = 1,000 messages/month — still ₹999
- Economical at any scale below 50K messages/month

### Template Approval Process (Critical Path)

| Template | Description | Approval Time | Status |
|---|---|---|---|
| `rent_reminder_3days` | Due in 3 days with link | 24–48 hours | Submit Week 1 |
| `rent_reminder_due_today` | Due today, urgent | 24–48 hours | Submit Week 1 |
| `rent_receipt` | Receipt after payment | 24–48 hours | Submit Week 1 |
| `rent_overdue` | 5+ days overdue | 24–48 hours | Submit Week 1 |
| `tenant_welcome` | New tenant intro | 24–48 hours | Submit Week 2 |
| `agreement_signed` | eSign confirmation | 48–72 hours | Submit Week 4 |

⚠️ **CRITICAL**: Submit all templates by **end of Week 1**. Approval can take 72 hours. Delayed template approval = delayed scheduler deployment.

---

## 6. Cashfree (Payment Gateway + Beneficiary API)

| Property | Details |
|---|---|
| **Purpose** | Automated rent collection from tenants, direct settlement to owner banks |
| **Tier** | SANDBOX (testing) → PRODUCTION (live payments) |
| **Features** | Payment links (tenant pays), Beneficiary Transfer API (settle to owner bank), webhooks |
| **Pricing** | 1.95% per transaction + ₹0 gateway fee |
| **Monthly Cost (Estimate)** | ₹0 (Phase 1 MVP, launch in Phase 3 Week 9) |
| **Phase 3 Estimate** | 50 owners × 1 payment/month × ₹1,000 avg = ₹50K revenue × 1.95% = ₹975/month cost |
| **Annual Cost (At scale)** | ~₹15,600 (₹1,300/month at 1,000 tenants) |

### Phase 1 Status

**NOT integrated in Phase 1** — Phase 0 uses personal UPI deep links, Phase 1 focus is CRUD + dashboard. Cashfree comes in Phase 3 (Week 9–10).

### Phase 3 Integration (For Reference)

- **Owner onboarding** → register bank account with Cashfree (penny-drop ₹1 verification)
- **Create payment link** → Cashfree generates unique link per rent record
- **Webhook** → auto-mark rent as paid when tenant pays
- **Settlement** → daily to owner's verified bank account

### Phase 4 Integration: Cashfree Subscriptions for Owner Billing (Week 11–12)

Cashfree also handles **recurring subscription billing** for owner plans. Same API key, different endpoint.

**How It Works:**

```javascript
// Owner signs up for Tier 1 plan (₹300/month)
const subscription = await cashfree.subscriptions.create({
  subscriptionId: `sub_owner_${owner.id}`,
  planId: 'plan_tier1_monthly',  // Created in Cashfree dashboard
  customerId: owner.id,
  // Cashfree auto-bills customer's card every month
  // Can pause/upgrade/downgrade via API
});

// Webhook on successful charge
// → Auto-update owner.plan in DB
// → Send receipt via email

// Owner can cancel anytime
await cashfree.subscriptions.cancel(subscriptionId);
```

**Benefits of using Cashfree for both:**
- ✅ Single payment provider integration (simpler, less overhead)
- ✅ Reuse webhook infrastructure
- ✅ Same HMAC security for both tenant rent + owner subscriptions
- ✅ Lower cost: 1.95% for both flows
- ✅ Unified customer/beneficiary management

### Environment Variables (Phase 3–4)

```javascript
PG_CLIENT_ID=xxx         // Cashfree API key (for both tenant rent + owner subscriptions)
PG_CLIENT_SECRET=xxx     // Cashfree secret
PG_WEBHOOK_SECRET=xxx    // For HMAC signature verification
PAYMENT_GATEWAY=cashfree // Single gateway for all payments
```

### Database: Subscription Billing Table (Phase 4)

```sql
CREATE TABLE owner_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES owners(id) ON DELETE CASCADE UNIQUE NOT NULL,
  cashfree_subscription_id TEXT NOT NULL UNIQUE,
  plan TEXT NOT NULL,  -- starter | growth | pro
  amount NUMERIC(10,2) NOT NULL,
  status TEXT DEFAULT 'active',  -- active | paused | cancelled
  started_at TIMESTAMPTZ NOT NULL,
  next_charge_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  
  INDEX (owner_id),
  INDEX (cashfree_subscription_id)
);

-- Subscription billing history
CREATE TABLE subscription_charges (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID REFERENCES owners(id) ON DELETE CASCADE NOT NULL,
  subscription_id TEXT NOT NULL,
  amount NUMERIC(10,2) NOT NULL,
  status TEXT,  -- success | failed | refunded
  charge_date TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  INDEX (owner_id),
  INDEX (charge_date)
);
```

---

## 7. Domain + Email (Resend or AWS SES)

| Property | Details |
|---|---|
| **Purpose** | Custom domain (rentmaster.in or similar), transactional email for invoices/receipts |
| **Domain** | Namecheap or GoDaddy (.in domain = ₹99–399/year) |
| **Email Service** | Resend (₹0 for 3,000 free emails/month) or AWS SES (₹0.10 per 1,000 sent) |
| **Monthly Cost (Domain)** | ₹8–33 amortized |
| **Monthly Cost (Email)** | ₹0–₹50 depending on volume |
| **Annual Cost** | ~₹100–500 |

### Phase 1 Features Used

- Custom domain: rentmaster.in (or .com if preferred)
- Transactional email: password reset, invoice PDF (Phase 4)
- From address: support@rentmaster.in

### Recommended Setup

```javascript
// Use Resend (free tier is plenty for MVP)
import { Resend } from 'resend';
const resend = new Resend(process.env.RESEND_API_KEY);

await resend.emails.send({
  from: 'support@rentmaster.in',
  to: 'owner@example.com',
  subject: 'Your RentMaster Invoice',
  html: `<p>Invoice attached...</p>`
});
```

---

## 8. GitHub + GitHub Actions (Version Control + CI/CD)

| Property | Details |
|---|---|
| **Purpose** | Source code hosting, automated testing, deployment pipeline |
| **Tier** | Free (unlimited public/private repos) |
| **Monthly Cost** | ₹0 |
| **Annual Cost** | ₹0 |

### Phase 1 Features Used

- Private repo for source code
- GitHub Actions: auto-deploy to Vercel on `git push main`
- GitHub Actions: run TypeScript type-check + ESLint before merge
- Branch protection: require PR review before merging to main

### CI/CD Pipeline (.github/workflows/deploy.yml)

```yaml
name: Deploy to Vercel

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci && npm run type-check && npm run lint
      - run: npx vercel deploy --prod --token ${{ secrets.VERCEL_TOKEN }}
```

---

## Summary: Phase 1 Infrastructure Costs

| Service | Free Tier | Monthly Cost | Annual Cost | Notes |
|---|---|---|---|---|
| **Supabase** | ₹0 (MVP) | ₹0–₹2,100 | ₹0–₹25,200 | Upgrade at Month 3 |
| **Vercel** | ₹0 | ₹0–₹1,680 | ₹0–₹20,160 | Free sufficient for MVP |
| **Render.com** | ₹0 | ₹0–₹588 | ₹0–₹7,056 | Free with restarts OK; upgrade Month 3 |
| **Upstash Redis** | ₹0 | ₹0–₹168 | ₹0–₹2,000 | Free tier ≈ 500 owners |
| **Interakt WhatsApp** | ₹0 (No free tier) | ₹999 | ₹11,988 | **Fixed cost, no overage** |
| **Cashfree** | N/A (Phase 3–4) | ₹0–₹1,200 | ₹0–₹14,400 | Both tenant rent + owner subscriptions; variable per transactions |
| **Domain** | N/A | ₹8–33 | ₹100–399 | One-time purchase + renewal |
| **Email (Resend)** | ₹0 | ₹0–₹50 | ₹0–₹600 | Free for <3K/month |
| **GitHub** | ₹0 | ₹0 | ₹0 | Free plan enough |
| --- | --- | --- | --- |
| **PHASE 1 TOTAL** | **₹0–999** | **₹999–₹5,700** | **₹11,988–₹68,400** | Scales with Supabase/Render |

### Cost Evolution

| Timeline | Scenario | Monthly Cost | Cashfree (1.95% trans) |
|---|---|---|---|
| **Week 1–8 (MVP Launch)** | Free tiers + Interakt | ₹999 | ₹0 |
| **Month 2 (10 paying customers)** | Add Render Starter ($588) | ₹1,587 | +₹100 (tenant rents) |
| **Month 3 (25 paying customers)** | Upgrade Supabase Pro ($2,100) | ₹3,687 | +₹150 |
| **Month 4 (50 paying customers)** | All infrastructure upgraded | ₹5,476 | +₹300 (Phase 3: tenant rents) |
| **Month 5 (Phase 4 Launch)** | Add Cashfree subscriptions for owner billing | ₹5,700–₹6,000 | +₹200–₹500 (owner subscriptions) |
| **Month 6+ (100+ paying customers)** | Full stack: infra + Cashfree (both flows) | ₹6,000–₹6,500 | Tenant + owner billing combined |

### Break-Even Timeline

At ₹5,476/month infrastructure cost with 3-tier pricing:

| Scenario | Customer Count | Avg Price | Monthly Revenue | Margin |
|---|---|---|---|---|
| 10 customers @ ₹500/mo avg | 10 | ₹500 | ₹5,000 | -₹476 (loss) |
| 20 customers @ ₹500/mo avg | 20 | ₹500 | ₹10,000 | ₹4,524 |
| **Break-even: ~17 customers** | 17 | ₹500 | ₹8,500 | ₹3,024 |

---

## Phase 1 Integration Checklist

### Week 1: Setup & Configuration

- [ ] Create Supabase project, run full schema migration + RLS policies
- [ ] Deploy Next.js boilerplate to Vercel with env vars configured
- [ ] Set up GitHub repo with branch protection rules
- [ ] Create Render.com account, set up background worker (dummy process for now)
- [ ] Create Upstash Redis instance, test connection
- [ ] Register Interakt account, submit 6 WhatsApp templates for approval
- [ ] Purchase domain (rentmaster.in) + configure DNS to Vercel
- [ ] Set up Resend account for transactional email

### Week 3–6: Core CRUD + Dashboard

- [ ] Build property/room/tenant management (CRUD)
- [ ] Build occupancy dashboard with real-time updates (Supabase Realtime)
- [ ] Tenant KYC upload (Supabase Storage)
- [ ] Manual "Mark as Paid" flow
- [ ] Rent history view

### Week 7–8: WhatsApp Automation (Phase 2)

- [ ] Deploy BullMQ worker to Render.com
- [ ] Write scheduler: daily cron at 7 AM IST
- [ ] Implement reminder sending via Interakt API
- [ ] Add monthly rent record generation
- [ ] Set up payment webhook listener (for Phase 3)

### Week 9–10: Payment Integration (Phase 3)

- [ ] Integrate Cashfree Beneficiary API (bank registration)
- [ ] Auto-create Cashfree payment links in reminder flow
- [ ] Handle Cashfree webhooks (auto-mark paid)
- [ ] Real-time dashboard updates on payment

### Week 11–12: Owner Subscription Billing (Phase 4)

- [ ] Create subscription plans in Cashfree dashboard (Tier 1: ₹300, Tier 2: ₹600, Tier 3: ₹1,300)
- [ ] Build owner plan selection flow during signup
- [ ] Integrate Cashfree subscriptions API for auto-billing
- [ ] Implement subscription webhook handler (subscription.charged, subscription.failed)
- [ ] Create owner_subscriptions + subscription_charges tables
- [ ] Send billing receipts via Resend email on successful charge
- [ ] Build owner subscription management dashboard (view plan, pause, upgrade, cancel)
- [ ] Handle failed payments + retry logic

---

## Key Integration Notes

### 1. Environment Variables (Master List)

Store these in `.env.local` (local dev) and Vercel/Render.com dashboard (production):

```
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# Vercel (auto-injected)
VERCEL_ENV=production

# Render.com worker
DATABASE_URL=postgresql://...
UPSTASH_REDIS_URL=redis://...

# WhatsApp (Interakt)
INTERAKT_API_KEY=xxx

# Cashfree (Phase 3 — Tenant Rent Collection)
PG_CLIENT_ID=xxx
PG_CLIENT_SECRET=xxx
PG_WEBHOOK_SECRET=xxx
PAYMENT_GATEWAY=cashfree

# Email
RESEND_API_KEY=xxx

# General
APP_URL=https://rentmaster.in
NODE_ENV=production
```

### 2. Webhook URL Routing

All external services send webhooks to **single endpoint** `/api/webhooks`:

```javascript
// pages/api/webhooks.js
export default async function handler(req, res) {
  const source = req.headers['user-agent']; // Identify sender

  if (source.includes('Cashfree')) {
    await handleCashfreeWebhook(req, res);
  } else if (source.includes('Interakt') || source.includes('Whatsapp')) {
    await handleWhatsAppWebhook(req, res);
  }
}
```

### 3. Security Best Practices

- **Never expose service role keys** in frontend code
- **Verify webhook signatures** before processing (HMAC-SHA256)
- **Encrypt bank details** before DB storage (AES-256)
- **Use HTTPS everywhere** (Vercel + Render auto-HTTPS)
- **Rate limit** API routes to prevent abuse

### 4. Monitoring & Alerts

Set up basic monitoring:

- **Supabase**: Monitor DB size, query count via dashboard
- **Vercel**: Monitor API route latency + function duration
- **Render.com**: Monitor worker uptime + job queue depth
- **Upstash**: Monitor Redis command count against free tier limit
- **Interakt**: Monitor WhatsApp delivery rate + failures

---

## Success Metrics

### Phase 1 (Week 1–8) ✅
✅ **Infrastructure:** All 8 core services deployed and tested  
✅ **Database:** Schema + RLS policies, data isolation verified  
✅ **Auth:** Owner phone OTP login working  
✅ **CRUD:** Property/room/tenant management functional  
✅ **Dashboard:** Real-time occupancy view  
✅ **KYC:** Tenant document upload working  
✅ **WhatsApp:** 6 templates approved, Interakt ready  
✅ **Cost:** ₹999–₹1,587/month

### Phase 3 (Week 9–10) ✅
✅ **Cashfree Tenant Payments:** Payment links + webhooks working  
✅ **Auto-Mark Paid:** Rent status updates when tenant pays  
✅ **Settlement:** Direct to owner bank accounts  
✅ **Cost:** +₹300–₹600/month (transaction-based)

### Phase 4 (Week 11–12) ✅
✅ **Cashfree Subscriptions:** Owner auto-billing setup  
✅ **Plan Management:** 3 tiers created (₹300/₹600/₹1,300)  
✅ **Auto-Debit:** Monthly recurring charges for owner subscriptions  
✅ **Webhooks:** subscription.charged, subscription.failed handling  
✅ **Billing History:** owner_subscriptions + subscription_charges tables  
✅ **Email Receipts:** Billing invoices sent via Resend  
✅ **Self-Service Dashboard:** View, pause, upgrade, cancel subscriptions  
✅ **Full Cost:** ₹6,000–₹6,500/month (at 100+ customers)

---

**Phase 1 Service Integration Document Complete**  
*Ready for implementation starting Week 1. Unified Cashfree integration for both tenant rent collection (Phase 3) and owner subscription billing (Phase 4).*
