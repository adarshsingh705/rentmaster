# RentMaster Pricing Plan
**Date:** 2026-06-12 (Automation-Premium Revision)
**Author:** Adarsh

---

## 1. The Core Pricing Argument — Automation is the Product

Every competitor sells a **management tool**. RentMaster sells **automated rent collection**. These are different products at different price points.

| What you're selling | Competitor equivalent | What competitors charge | What RentMaster should charge |
|---|---|---|---|
| Dashboard + manual tracking | GoPGMS, PGManager | ₹180–₹300/month | N/A — we don't sell this |
| WhatsApp reminders (credit-limited) | TrackMyPG | ₹299–₹999/month | N/A — ours is unlimited |
| **Automated collection + direct bank transfer + WhatsApp** | **Nobody** | **Doesn't exist** | **₹1,199–₹2,999/month** |

**The automation stack no competitor has built:**
1. Auto-triggered 3-step WhatsApp reminder sequence (3-day, due-date, overdue)
2. Cashfree payment link embedded in the reminder — tenant pays in one tap
3. Money lands **directly in owner's bank** — no holding, no delay
4. Auto-generated WhatsApp receipt sent to tenant
5. Owner gets real-time alert when payment is received

This workflow eliminates the most painful part of owning a PG: chasing rent manually.

---

## 2. Where We Stand vs Competitors

Research date: 2026-06-12. All prices verified from live websites.

| Platform | Entry Paid | WhatsApp | Direct Payments | Notes |
|---|---|---|---|---|
| **GoPGMS** | ₹9/bed/month (20 beds = ₹180) | ❌ None | ❌ None | Just a tracking tool |
| **TrackMyPG** | ₹299/mo — 50 credits only | Limited (credits) | ❌ None | No collection |
| **PGManager** | ₹300/mo + GST | ❌ None | ❌ None | Basic ledger |
| **RentOk** | ₹25–₹100/bed/month (quote) | ❌ No automation | ❌ None | Quote-based |
| **RentMaster** | ₹1,199/mo | ✅ Unlimited, absorbed | ✅ Direct to owner bank | **Only one that collects** |

### ROI Story for a PG Owner on Starter Plan

A typical 20-bed PG collects ~₹1,20,000/month:
- Manual collection time saved: 12 hrs/month × ₹150/hr = **₹1,800/month**
- Faster collections (3 days faster × 20 beds): **₹3,000–5,000/month cash-flow value**
- Reduced defaults (automation reduces 2–3 late defaults/month): **₹10,000+/month**
- **Total value delivered: ₹15,000–17,000/month**
- RentMaster Starter cost: ₹1,199/month = **~7% of the value delivered**

This is not a ₹299 tool. It is a ₹1,199 tool that pays for itself inside the first week.

---

## 3. The Unified "Slot" Billing Unit

**1 bed (PG) = 1 slot. 1 room (residential) = 1 slot. 1 shop/office (commercial) = 1 slot (at a premium rate).**

The billing unit is a slot. It doesn't matter what type of property — beds, rooms, and shops are all counted as slots. Commercial slots carry a premium rate because the value delivered (higher rent amounts, CAM billing, escalation tracking) is greater.

### Why "Slot" Works

| Property type | How slots are counted | Slot rate |
|---|---|---|
| Bed-based PG | Each bed = 1 slot | ₹60/slot (Tier 1) |
| Room-based flat/house | Each room = 1 slot | ₹60/slot (Tier 1) |
| Commercial shop/office | Each unit = 1 slot | ₹80/slot (Tier 1) |
| Mixed portfolio | All types combined | Blended — see below |

The WhatsApp cost driver is the same regardless of property type: 1 tenant per slot × 3 messages/month × ₹0.58 = ₹1.74/slot. Commercial slots are priced at a premium to reflect additional features (CAM billing, escalation alerts, agreement expiry tracking) and higher value delivered.

### How the System Handles a Mixed-Portfolio Owner at Login

```
Owner Account
├── Property A: "Sunrise PG"  (type: Bed-based PG)
│     Room 101 → 3 beds = 3 slots × ₹60 = ₹180
│     Room 102 → 3 beds = 3 slots × ₹60 = ₹180
│     Room 103 → 3 beds = 3 slots × ₹60 = ₹180
│     Subtotal: 9 residential slots = ₹540
│
├── Property B: "Andheri 2BHK"  (type: Room-based)
│     Room 1 → 1 slot × ₹60 = ₹60
│     Room 2 → 1 slot × ₹60 = ₹60
│     Subtotal: 2 residential slots = ₹120
│
└── Property C: "Sunshine Complex"  (type: Commercial)
      Shop 1 → 1 slot × ₹80 = ₹80
      Shop 2 → 1 slot × ₹80 = ₹80
      Subtotal: 2 commercial slots = ₹160

Total: 11 residential slots (₹660) + 2 commercial slots (₹160) = ₹820/month
```

The owner sees one number on their billing page: **"13 active slots (11 residential + 2 commercial) — ₹820/month."**

### Commercial Slot Rates

| Tier | Residential/PG Rate | Commercial Rate | Notes |
|---|---|---|---|
| Tier 1 (1–30 slots) | ₹60/slot | ₹80/slot | +₹20 premium for CAM + escalation features |
| Tier 2 (31–100 slots) | ₹50/slot | ₹65/slot | Volume discount applies |
| Tier 3 (101+ slots) | ₹40/slot | ₹50/slot | Large operator rate |

Commercial slots are counted separately from residential slots for tier placement (a 5 residential + 3 commercial owner is in Tier 1 for both, not combined into Tier 2).

### Property Type is Set Once, at Property Setup

The `property-setup.html` already captures this:
- **"Bed-based PG"** → beds are counted per room
- **"Room-based"** → rooms are the unit

No separate billing configuration needed. The property type drives the slot count automatically.

---

## 3B. "Build Your Plan" — The Custom Pricing Calculator

The slot model IS a custom pricing calculator. An owner with 5 rooms pays exactly 5 × ₹60 = ₹300/month — not ₹499 for a plan that also covers 40 rooms they don't have. This is the "pay only for what you need" model Adarsh wants, and it's already built into the pricing structure.

**What's missing is the UI** — an interactive calculator on the pricing page and onboarding that makes the math visible.

### Calculator Interaction

```
Build Your Plan:

Rooms/flats:  [ 5 ]  × ₹60 = ₹300      (Tier 1)
PG beds:      [ 0 ]  × ₹60 = ₹0
Shops:        [ 0 ]  × ₹80 = ₹0

━━━━━━━━━━━━━━━━━━━━━━━━━
Monthly:    ₹300
Annual:     ₹3,000  (save ₹600 — 2 months free)
━━━━━━━━━━━━━━━━━━━━━━━━━
[ Start 14-day trial — no card needed ]
```

As the owner changes numbers, the price updates live. When they cross 30 slots, the label changes to "You've entered Tier 2 — rate drops to ₹50/slot" and the total automatically recalculates.

**This answers "I never want to pay for more than I need."** — 5 rooms pays ₹300. Not ₹499. Not ₹1,199. Exactly ₹300.

### Cashfree Subscription for Platform Fees

RentMaster collects its own subscription fees using the same Cashfree infrastructure it provides to owners:

1. Owner completes trial → sees "Your plan: 5 slots = ₹300/month"
2. "Set up auto-pay" button → Cashfree UPI AutoPay mandate
3. Owner authorizes once (30 seconds) in GPay/PhonePe
4. Every month on 5th: Cashfree auto-debits ₹300 → owner gets WhatsApp receipt
5. If slots change: system updates mandate amount and WhatsApps owner

**Product story:** "We automate rent collection for your tenants using the same system we use to collect our own subscription. That's how much we trust it."

---

## 4. Your Real Costs Every Month

### Fixed Infra

| Service | Cost | Notes |
|---|---|---|
| Supabase (DB + Auth + Realtime) | ₹2,100/month | Free tier covers first 30–40 customers |
| Redis / Upstash (BullMQ queues) | ₹840/month | Free tier covers first 100 jobs/day |
| NestJS hosting (Railway) | ₹1,680/month | Starter plan |
| Interakt WhatsApp BSP | ₹999/month | Up to 1,000 conversations |
| Domain + misc | ₹83/month | |
| **Total fixed** | **₹5,700/month** | |

### Variable: WhatsApp Cost Per Bed

Meta: ~₹0.58/utility message. RentMaster sends 3 messages per tenant per month.

```
Cost per bed per month = 3 × ₹0.58 = ₹1.74
```

| Beds on platform | Monthly WhatsApp cost |
|---|---|
| 50 beds | ₹87 |
| 200 beds | ₹348 |
| 500 beds | ₹870 |

Even at 500 beds, WhatsApp cost is ₹870 — a tiny fraction of revenue.

---

## 5. Per-Slot Pricing Tiers

### TRIAL — ₹0 (14 days, no credit card)

Full Tier 1 features. Auto-expires. Conversion prompts at day 10 and day 13.

---

### TIER 1 — ₹60/slot/month  (1–30 slots)

**Who it's for:** 9-bed PG, small residential landlord, anyone getting started.

| Slots | Monthly cost | Owner context |
|---|---|---|
| 9 slots (9-bed PG) | ₹540 | ₹54K rent collected → 1% platform fee |
| 20 slots (20-bed PG) | ₹1,200 | ₹1.2L rent → 1% fee |
| 30 slots (30-bed PG) | ₹1,800 | ₹1.8L rent → 1% fee |

**Included:** Unlimited WhatsApp automation · Cashfree direct payments · PDF receipts · Dashboard · Unlimited properties

**Your margin:** ₹1,800 (30 slots revenue) − ₹52 WhatsApp = **₹1,748 gross (97%)**

---

### TIER 2 — ₹50/slot/month  (31–100 slots)

**Who it's for:** 90-bed single hostel, medium multi-property owner, mixed portfolio.

**Billing for 90 beds:** 30×₹60 + 60×₹50 = ₹1,800 + ₹3,000 = **₹4,800/month**

| Slots | Monthly cost | Owner context |
|---|---|---|
| 90 slots (90-bed hostel) | ₹4,800 | ₹6.3L rent → 0.76% platform fee |
| 50 slots (50-bed PG) | ₹2,700 | ₹3.5L rent → 0.77% fee |
| 35 slots (30 beds + 5 rooms) | ₹2,050 | Mixed portfolio — slots combine |

**Added in Tier 2:** CSV export · eSign (Phase 3) · Priority WhatsApp 24h support

**Your margin at 90 slots:** ₹4,800 − ₹157 WhatsApp = **₹4,643 gross (97%)**

---

### TIER 3 — ₹40/slot/month  (101+ slots)

**Who it's for:** Large hostel chains, 100+ beds, multiple cities.

**Billing for 150 beds:** 30×₹60 + 70×₹50 + 50×₹40 = ₹1,800 + ₹3,500 + ₹2,000 = **₹7,300/month**

| Slots | Monthly cost | Owner context |
|---|---|---|
| 150 slots | ₹7,300 | ₹10.5L rent → 0.70% platform fee |
| 200 slots | ₹9,800 | ₹14L rent → 0.70% fee |

**Added in Tier 3:** REST API · Analytics dashboard (Phase 4) · Custom WhatsApp templates · Dedicated account manager

**Annual pricing:** 2 months free — Tier 1: ₹50/slot · Tier 2: ₹42/slot · Tier 3: ₹33/slot

---

## 6. The Critical Number — 5 Customers Covers Infra

```
5 × average Tier 1 customer (20 slots × ₹60) = 5 × ₹1,200 = ₹6,000 > ₹5,700 infra ✓
OR: 1 × 90-bed Tier 2 customer = ₹4,800 alone → almost covers infra solo
OR: 2 × 50-bed Tier 2 customers = ₹5,400 → covers infra
```

A **single 90-bed hostel owner** on Tier 2 nearly covers all your infrastructure.

---

## 7. Revenue: What You Need

### Monthly Target Costs

| Item | Monthly |
|---|---|
| Infra (Supabase + Redis + Railway + Interakt) | ₹5,700 |
| Your salary target | ₹50,000 |
| **Total needed** | **₹55,700/month** |

### Customer Mix + Break-Even

Realistic mix: **60% Starter + 25% Pro + 15% Scale**  
Average revenue/customer: `0.60×1,199 + 0.25×1,999 + 0.15×2,999 = ₹1,669/month`

| Customers | Monthly Revenue | What It Pays |
|---|---|---|
| 5 | ₹8,345 | **Infra fully covered** |
| 10 | ₹16,690 | Infra + ₹10,990 salary |
| 20 | ₹33,380 | Infra + ₹27,680 salary |
| **34 customers** | **₹56,746** | **Full break-even (infra + ₹50K salary)** |
| 50 | ₹83,450 | Infra + ₹77,750 salary |
| 75 | ₹1,25,175 | Very comfortable |

**You need just 34 paying customers for full sustainability.**  
At the previous ₹299–₹599 pricing, you needed 165 customers.  
The automation premium turns a 165-customer problem into a **34-customer problem.**

---

## 8. When Trial Users Upgrade (Conversion Triggers)

1. **Trial expires at day 14** — "Your 14-day trial has ended. Continue with full automation from ₹1,199/month."
2. **Bed limit hit** — "You've reached 20/20 beds. Upgrade to Pro for 60 beds."
3. **Second property added** — "Pro plan needed for multiple properties."
4. **CSV requested** — "Download monthly reports — available on Pro and above."
5. **Growth message** — "You collected ₹1,80,000 this month via RentMaster. Scale up on Pro."

---

## 9. Price Positioning vs Competitors

```
TrackMyPG Pro   ₹599/mo  →  50 WhatsApp credits, NO payments
GoPGMS          ₹180/mo  →  20 beds tracked, NO automation, NO payments
PGManager       ₹300/mo  →  75 beds tracked, NO automation, NO payments

RentMaster Starter ₹1,199/mo → UNLIMITED automation + DIRECT payments + PDF receipts

Premium vs TrackMyPG Pro:  2× the price, incomparably better product
Premium vs GoPGMS:         6.6× the price, 10× the value delivered
```

**The one-liner for sales conversations:**  
> "TrackMyPG reminds your tenants. RentMaster collects your money. That's why it costs more."

---

## 10. Key Decisions Still Open

| Question | Recommended |
|---|---|
| Trial card required? | No card — reduce friction |
| Annual discount | 1 month free (₹11,990 / ₹19,990 / ₹29,990) |
| WhatsApp cost: absorb or pass on? | Absorb — it's the differentiator, margins are 88–97% |
| Referral program | Refer 1 owner → ₹1,199 credit (1 free month) |
| GST invoice | Phase 4 add-on |

---

## 11. What To Update in the UX

- [ ] Update Starter card: ₹1,199/month, 20 beds, unlimited WhatsApp, direct payments
- [ ] Update Pro card: ₹1,999/month, 60 beds
- [ ] Update Scale card: ₹2,999/month, unlimited
- [ ] Update annual prices: ₹11,990 / ₹19,990 / ₹29,990
- [ ] Update comparison table with competitor row ("vs TrackMyPG")
- [ ] Update FAQ: new prices, ROI explanation
- [ ] Update 14-day trial CTA on all cards

---

## 12. 12-Month Revenue Projection

**Assumptions:** Avg ₹1,669/customer/month · 60% Starter + 25% Pro + 15% Scale mix  
**Fixed costs:** ₹5,700/month infra | **Break-even:** ₹55,700/month (infra + ₹50K salary)

### Key Milestones

| Milestone | Month | Customers | Monthly Revenue | What It Means |
|---|---|---|---|---|
| Infra covered | **Month 2** | 5 | ₹8,345 | Platform costs ₹0 net — all infra paid |
| ₹25K salary | **Month 5** | 18 | ₹30,042 | Half salary viable — can go full-time |
| Full break-even | **Month 8** | 34 | ₹56,746 | ₹50K salary + all infra fully paid |
| Comfortable profit | **Month 12** | 55 | ₹91,795 | ~₹36K net profit after all costs |

---

### Month-by-Month Growth Table

```
Month | Customers | Revenue/mo  | vs Infra ₹5,700 | vs Break-even ₹55,700 | Status
------|-----------|-------------|-----------------|------------------------|-------
  M1  |     2     |   ₹3,338    |    -₹2,362      |       -₹52,362         | 🔴 Building
  M2  |     5     |   ₹8,345    |    +₹2,645 ✓   |       -₹47,355         | 🟡 Infra covered
  M3  |     8     |  ₹13,352    |    +₹7,652      |       -₹42,348         | 🟡 Growing
  M4  |    12     |  ₹20,028    |   +₹14,328      |       -₹35,672         | 🟡 Growing
  M5  |    18     |  ₹30,042    |   +₹24,342 ✓   |       -₹25,658         | 🟢 ₹25K salary
  M6  |    24     |  ₹40,056    |   +₹34,356      |       -₹15,644         | 🟢 Growing
  M7  |    30     |  ₹50,070    |   +₹44,370      |        -₹5,630         | 🟢 Near target
  M8  |    34     |  ₹56,746    |   +₹51,046 ✓   |        +₹1,046 ✓      | ✅ FULL BREAK-EVEN
  M9  |    39     |  ₹65,091    |   +₹59,391      |        +₹9,391         | ✅ Profitable
  M10 |    44     |  ₹73,436    |   +₹67,736      |       +₹17,736         | ✅ Growing profit
  M11 |    50     |  ₹83,450    |   +₹77,750      |       +₹27,750         | ✅ Strong
  M12 |    55     |  ₹91,795    |   +₹86,095      |       +₹36,095         | ✅ Very comfortable
```

---

### Visual Revenue Curve

```
₹92K  ┤                                                                ████ M12
₹83K  ┤                                                          ████
₹73K  ┤                                                     ████
₹65K  ┤                                                ████
₹56K  ┤ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ BREAK-EVEN ─ ─ ─ ─ ─
      ┤                                           ████  ← M8: 34 customers ✅
₹40K  ┤                                     ████
₹30K  ┤                               ████  ← M5: ₹25K salary ✓
₹20K  ┤                         ████
₹13K  ┤                    ████
₹8.3K ┤         ████  ← M2: infra covered ✓
₹5.7K ┤ ─ ─ ─ INFRA COST ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
₹3.3K ┤  ████
      └──M1───M2───M3───M4───M5───M6───M7───M8───M9──M10──M11──M12
```

---

### Old Pricing vs New Pricing — Impact

| Metric | Old (₹299–₹999) | New (₹1,199–₹2,999) |
|---|---|---|
| Avg revenue/customer | ₹344/month | **₹1,669/month** |
| Customers to cover infra | 17 | **5** |
| Customers to full break-even | 165 | **34** |
| Months to full break-even | ~12 | **~8** |
| Revenue at 55 customers | ₹18,920 | **₹91,795** |

**The automation premium turns a 12-month, 165-customer problem into an 8-month, 34-customer problem.**

---

### Growth Rate Assumption

- **Months 1–2:** 2–5 customers — beta users, personal network
- **Months 3–5:** +3–6/month — word of mouth in PG owner WhatsApp groups
- **Months 6–9:** +5–6/month — referral program active
- **Months 10–12:** +5/month — steady compounding

**34 customers = 0.0017% of India's ~2M PG owners.** This is very achievable.

---

## 13. One-Line Summary

> **Charge ₹1,199/month because no competitor collects rent automatically. 5 customers cover all infra costs. Need 34 paying customers for full ₹50K salary. In 8 months.**

---

*Competitor research: `_bmad-output/planning-artifacts/research/pricing-competitor-analysis-2026-06-12.md`*
*Previous brainstorming: `_bmad-output/brainstorming/brainstorming-session-2026-06-12-1015.md`*
