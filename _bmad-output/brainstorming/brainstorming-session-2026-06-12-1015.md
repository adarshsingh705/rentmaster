---
stepsCompleted: [1, 2, 3, 4]
inputDocuments: []
session_topic: 'RentMaster pricing model — competitive analysis + bed/room-based tier design'
session_goals: 'Design a profitable pricing model (beds × rooms × properties) where Free covers 1 property/5 rooms, paid tiers cover infra costs + founder salary'
selected_approach: 'All techniques combined'
techniques_used: ['Competitive Benchmarking', 'Unit Economics Modeling', 'Value-Based Pricing', 'Progressive Tier Flow', 'Blue Ocean Analysis']
ideas_generated: []
context_file: ''
---

# Brainstorming Session Results

**Facilitator:** Adarsh
**Date:** 2026-06-12

## Session Overview

**Topic:** RentMaster Pricing Model — Competitor Benchmarking + Bed/Room-Based Tier Design  
**Goals:**
- Map what Indian PG/rental SaaS competitors charge
- Rethink plan limits around beds and rooms (not just tenant count)
- Free tier: 1 property · 5 rooms · room-based only
- Paid tiers: must cover infra + WhatsApp BSP costs + founder salary

---

## TECHNIQUE 1 — Competitive Benchmarking

### Indian Market Competitors

| Platform | Type | What They Offer | Pricing | Gap vs RentMaster |
|---|---|---|---|---|
| NoBroker | Listing + rent pay | Rent agreement, police verification, online rent transfer | Free basic / ₹1,999/yr agreement | No WhatsApp automation, no reminder engine |
| MyGate | Society mgmt | Gate management, visitor, maintenance billing | ₹99/flat/month | Built for gated societies, not PG |
| Apnacomplex | Society mgmt | Maintenance billing, facility booking, committee tools | ₹35-150/flat/month | Housing societies only, no PG |
| Zimply | PG mgmt app | Basic tenant register, manual receipts | Free / freemium | No Cashfree, no WhatsApp BSP |
| Stanza Living | Managed operator | They lease + manage PG, not an owner SaaS | Not a tool | Operator, not a platform for owners |
| Zolo / OYO Life | Managed operator | Same — they operate the PG, not software | Not a tool | Same — not a competitor |
| Little Hotelier | Hospitality PMS | Channel manager, booking engine, PMS | ₹1,500-3,000/month | Hospitality overkill, no Indian payment UX |
| eZee Absolute | Hospitality PMS | Full hotel PMS, reporting, OTA connect | ₹3,000+/month | Way too complex, no WhatsApp, too expensive |

### International Tools Used in India

| Platform | Pricing | Why India Owners Can't Fully Use It |
|---|---|---|
| TenantCloud | Free up to 75 units / $12/month | No WhatsApp, no UPI/Cashfree, no Hindi |
| Landlord Studio | Free 3 props / £4.99/month | No WhatsApp, no Indian payment gateway |
| Buildium | $50+/month | Expensive, US lease laws, no WhatsApp |
| Rentec Direct | $45+/month | US-focused, no Indian features |

### The Blue Ocean Finding

**No platform in India delivers:**
- WhatsApp automation (reminder + receipt) via BSP
- Cashfree Beneficiary Transfer (no holding funds)
- PG-specific bed/room model
- Under ₹500/month

**RentMaster's clear lane:** Automated WhatsApp rent collection for small-to-medium Indian PG owners who currently use Excel + manual WhatsApp. This is an uncontested space.

---

## TECHNIQUE 2 — Unit Economics Modeling

### Monthly Infrastructure Costs (Fixed)

| Service | Free Tier | Production Tier | Notes |
|---|---|---|---|
| Supabase | ₹0 (500MB, 2GB BW) | ₹2,100/month (Pro) | Need Pro at ~50+ customers |
| Redis / Upstash (BullMQ queues) | ₹0 (10K cmds/day) | ₹840/month (Pro) | Need at ~200+ jobs/day |
| NestJS hosting (Railway) | ₹840/month (Starter) | ₹1,680/month (Pro) | Sleep wake-up issues on Starter |
| Domain + SSL | ₹83/month (annualized) | ₹83/month | Fixed |
| **Total Fixed Infra** | **~₹920/month** | **~₹4,700/month** | Scales in 2 steps |

### WhatsApp Costs (Variable, Per-Bed)

Meta charges per **utility template message** (each WhatsApp reminder/receipt):
- 1 utility conversation = ₹0.58 (opens 24h window)
- RentMaster sends 3 messages per tenant per month: 3-day warning, due date, overdue
- **Cost per bed/tenant per month = 3 × ₹0.58 = ~₹1.74**

| Beds on Platform | WhatsApp Cost/Month | + Fixed Infra | Total Cost/Month |
|---|---|---|---|
| 50 beds | ₹87 | ₹920 | ~₹1,000 |
| 200 beds | ₹348 | ₹4,700 | ~₹5,050 |
| 500 beds | ₹870 | ₹4,700 | ~₹5,570 |
| 1,000 beds | ₹1,740 | ₹4,700 | ~₹6,440 |
| 5,000 beds | ₹8,700 | ₹8,000 | ~₹16,700 |

**Key insight:** Infra cost per bed drops dramatically at scale. At 5,000 beds the platform costs only ₹3.34/bed/month to run.

### Founder Salary Requirement

| Target Salary | Break-Even Customers Needed at ₹249 avg | At ₹399 avg | At ₹599 avg |
|---|---|---|---|
| ₹30,000/month | 140 | 87 | 58 |
| ₹50,000/month | 220 | 138 | 92 |
| ₹75,000/month | 320 | 200 | 134 |
| ₹1,00,000/month | 420 | 263 | 175 |

*(Includes ₹5,000/month infra in all calculations)*

**First break-even milestone (infra only, ₹5,000/month):** just 13 customers at ₹399/month.

---

## TECHNIQUE 3 — Value-Based Pricing

### What Is a PG Owner's Monthly Revenue?

| Property Size | Beds | Avg Rent/Bed | Monthly Revenue | RentMaster Value |
|---|---|---|---|---|
| Tiny PG | 5 beds | ₹7,000 | ₹35,000 | Saves ~2 hrs/month chasing |
| Small PG | 15 beds | ₹8,000 | ₹1,20,000 | Recovers ~₹6,000 in late payments |
| Medium PG | 30 beds | ₹9,000 | ₹2,70,000 | Recovers ~₹13,500 in late payments |
| Large PG | 60 beds | ₹10,000 | ₹6,00,000 | Recovers ~₹30,000 in late payments |

*Assumption: RentMaster recovers ~5% late payments through automation (conservative)*

### Price-to-Value Ratio

At ₹249/month for a 15-bed PG owner collecting ₹1,20,000:
- Platform fee = **0.2% of their revenue** — unbeatable ROI
- Saves 3-4 hours of WhatsApp follow-up per month
- Eliminates 1-2 bad payers per month (₹8,000+ recovered)

**Result: Any price under ₹1,000/month is deeply undervaluing the service for medium+ PGs.**

---

## TECHNIQUE 4 — Progressive Flow: Pricing Shapes

### Option A: Flat Tier by Bed/Room Count *(Recommended)*

Simple, predictable, easy to market. Owner knows exactly what they'll pay.

| Plan | Price | Rooms (residential) | Beds (PG) | Properties |
|---|---|---|---|---|
| **Free** | ₹0 | 5 rooms | Not allowed (room-only) | 1 |
| **Starter** | ₹249/month | 15 rooms | 20 beds | 1 |
| **Pro** | ₹499/month | 40 rooms | 60 beds | 3 |
| **Scale** | ₹999/month | Unlimited | Unlimited | 10 |

### Option B: Per-Unit Pricing

Charge per active room/bed per month. Feels fair, but unpredictable billing → hurts conversions.

| Unit | Price | 10 units | 30 units | 60 units |
|---|---|---|---|---|
| Per room/bed | ₹25/unit/month | ₹250 | ₹750 | ₹1,500 |

*Problem: Owners get billing anxiety. They might avoid adding tenants. Not recommended.*

### Option C: Properties-First (NoBroker model)

Charge by property count. Simple but doesn't match PG owner mental model. A 60-bed single-property PG pays same as a 5-bed single-property residential. Unfair value capture. **Not recommended.**

### Option D: Hybrid — Flat Tier + Beds Add-On

Base plan flat fee + ₹15/additional bed above threshold. Flexible but complex billing. Could work for enterprise but confusing for small owners. **Maybe for Phase 4.**

---

## TECHNIQUE 5 — Blue Ocean: What Competitors Haven't Done

Ideas that none of the competitors offer:

1. **WhatsApp-first onboarding** — no app install needed, owner sets up via WhatsApp bot
2. **Tenant self-onboarding via WhatsApp** — tenant scans QR → submits ID → auto-added
3. **Payment-split for sharing beds** — 2 tenants share 1 bed, system splits ₹8,000 into 2 × ₹4,000 rent links
4. **Maintenance complaint tracker per room** — tenant WhatsApps complaint → auto-logged against room
5. **Rent-hike broadcast** — owner sets new rent from dashboard → all tenants WhatsApp notified in 1 click
6. **Occupancy prediction alerts** — tenant's lease ends in 30 days → owner gets WhatsApp nudge to find replacement
7. **Peer benchmarking** — "Your PG in Koramangala collects rent 3 days faster than average. Here's how."
8. **Agreement renewal reminders** — 60 days before agreement expiry → auto-WhatsApp to owner
9. **GST receipt option** (for owners who need it) — premium add-on at Phase 4
10. **Free tier referral unlock** — refer 1 paying owner → unlock 10 beds for free

---

## RECOMMENDED PLAN DESIGN

### The Winning Structure

**Logic:** Gate on the *complexity* of use case, not just quantity. Room-based (1 tenant per room) is simpler → give it free. Bed-based (PG, multiple per room) requires more infra, more WhatsApp messages, more value → charge for it.

---

### FREE — ₹0/month

| Parameter | Value |
|---|---|
| Properties | 1 |
| Room type | **Room-based only** (1 tenant per room) |
| Beds per room | Not applicable |
| Rooms | Up to **5 rooms** |
| Automated WhatsApp reminders | Yes (3-day + due date sequence) |
| Cashfree payment links | Yes |
| WhatsApp receipts | Yes |
| Live dashboard | Yes |
| PDF receipts / CSV | No |
| Support | Docs only |

**Target owner:** Residential landlord, 3-5 flats in one building. Manages ₹30,000-₹50,000/month rent. Currently using Excel + manual WhatsApp.

**Why this works as acquisition:** They get full automation, feel the real value, then grow into paid plans when they add more rooms or switch to PG model.

---

### STARTER — ₹249/month *(or ₹2,490/year = 2 months free)*

| Parameter | Value |
|---|---|
| Properties | 1 |
| Room type | Room-based + **Bed-based PG** |
| Rooms | Up to **15 rooms** |
| Beds total | Up to **20 beds** |
| Automated WhatsApp reminders | Yes (full 3-step: 3-day, due, overdue) |
| Cashfree payment links | Yes |
| WhatsApp receipts | Yes |
| Manual overdue trigger (FR-19) | Yes |
| Live dashboard | Yes |
| PDF receipts / CSV | No |
| Support | Email (48h) |

**Target owner:** Small PG owner, 1 property with 10-20 beds. Collects ₹80,000-₹1,60,000/month. Platform fee = 0.15-0.3% of revenue.

**Infra cost at 20 beds:** 20 × ₹1.74 = ₹34.80 WhatsApp. Revenue ₹249. **Gross margin: 86%.**

---

### PRO — ₹499/month *(or ₹4,990/year = 2 months free)*

| Parameter | Value |
|---|---|
| Properties | Up to **3** |
| Room type | Room-based + Bed-based PG |
| Rooms | Up to **40 rooms** |
| Beds total | Up to **60 beds** |
| Automated WhatsApp reminders | Yes (full sequence + overdue escalation) |
| Cashfree payment links | Yes |
| WhatsApp receipts | Yes |
| Manual overdue trigger | Yes |
| Live dashboard | Yes |
| PDF rent receipts | Yes |
| CSV export (monthly) | Yes |
| Digital agreements / eSign | Phase 3 |
| Support | Priority WhatsApp (24h) |

**Target owner:** Medium PG owner with 1-3 properties, 30-60 beds, collecting ₹2,40,000-₹6,00,000/month.

**Infra cost at 60 beds:** 60 × ₹1.74 = ₹104 WhatsApp. Revenue ₹499. **Gross margin: 79%.**

---

### SCALE — ₹999/month *(or ₹9,990/year = 2 months free)*

| Parameter | Value |
|---|---|
| Properties | Up to **10** |
| Room type | Room-based + Bed-based PG + Mixed |
| Rooms / Beds | **Unlimited** |
| Everything in Pro | Yes |
| Analytics dashboard | Yes |
| Tenant behaviour trends | Yes |
| API access (REST) | Yes |
| Dedicated account manager | Yes |
| Custom WhatsApp template messages | Yes |
| Priority phone support | Yes |

**Target owner:** Large PG operator, 5-10 properties, 100+ beds, collecting ₹8,00,000+/month.

**Infra cost at 200 beds:** 200 × ₹1.74 = ₹348. Revenue ₹999. **Gross margin: 65%.**

---

## REVENUE MILESTONES

| Milestone | Customers | Assumed Plan Mix | Monthly Revenue | What It Covers |
|---|---|---|---|---|
| First income | 20 customers | 15 Starter + 5 Pro | ₹5,235 | Infra costs only |
| Ramen profitable | 50 customers | 30 Starter + 15 Pro + 5 Scale | ₹19,470 | Infra + ₹14,000 salary |
| Sustainable | 100 customers | 50 Starter + 35 Pro + 15 Scale | ₹38,490 | Infra + ₹33,000 salary |
| **Break-even target** | **150 customers** | **70 Starter + 55 Pro + 25 Scale** | **₹57,920** | **Infra + ₹53,000 salary** |
| Comfortable | 250 customers | 100 Starter + 100 Pro + 50 Scale | ₹1,02,400 | Infra + ₹97,000 salary |

*Mix assumption: 50% Starter / 35% Pro / 15% Scale — typical for SaaS B2B in India*

---

## FREE-TO-PAID CONVERSION TRIGGERS

The moments when a Free user naturally upgrades:

1. **"You have 5/5 rooms — add more to upgrade to Starter"** — natural growth trigger
2. **"Your tenant wants to share a room — upgrade to unlock bed-based PG"** — feature gate
3. **"You added a second property — Starter is needed for multi-property"** — property expansion
4. **"Export your June collection as PDF/CSV — available on Starter and above"** — admin need
5. **WhatsApp usage notification** — "Your free plan includes automated reminders for up to 5 rooms. You're at the limit."

---

## COMPETITOR MOAT SUMMARY

| What RentMaster Has | Status of Competitors |
|---|---|
| WhatsApp BSP (Interakt) automation | None of the Indian tools have this |
| Cashfree Beneficiary Transfer (no fund holding) | NoBroker does payment but holds funds |
| PG bed/room duality in one product | No competitor distinguishes room vs bed |
| Free tier with real automation | TenantCloud does but no Indian gateway |
| Under ₹500/month full PG management | Nothing in India under ₹1,000 is this complete |

---

## ACTION ITEMS

- [ ] Update `subscription.html` to reflect bed/room-based plan limits (replace tenant-count with rooms/beds)
- [ ] Add "Bed-based PG" as a feature gate visible on the Free plan card
- [ ] Add per-plan WhatsApp message volume indicators (so owner understands the infra)
- [ ] Add annual billing option (2 months free) to all paid cards
- [ ] Consider adding a "refer a friend → unlock 10 beds free" referral hook to the Free plan
- [ ] Add the ₹249 Starter plan (fill the gap between ₹0 and ₹499)

---

*Session completed: 2026-06-12. All techniques run: Competitive Benchmarking · Unit Economics · Value-Based Pricing · Progressive Flow · Blue Ocean.*
