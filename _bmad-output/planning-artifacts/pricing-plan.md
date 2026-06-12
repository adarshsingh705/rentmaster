# RentMaster Pricing Plan
**Date:** 2026-06-12  
**Author:** Adarsh 

---

## 1. The Market Gap (Why This Works)

No platform in India today does all three things together:

| What RentMaster Does | What Exists in India |
|---|---|
| WhatsApp automation (reminders + receipts) | Nothing for PG owners below ₹1,500/month |
| Cashfree payment links (money goes directly to owner bank — no holding) | NoBroker collects and holds funds |
| Bed-based PG + room-based residential in one product | No one distinguishes PG beds vs residential rooms |
| Under ₹500/month for full automation | TenantCloud is closest but no Indian gateway/WhatsApp |

**Bottom line:** RentMaster is in an uncontested space. The closest "competitors" (Zimply, NoBroker Pay) are either dead/basic tools without automation, or listing platforms that do payments differently.

---

## 2. Why Beds + Rooms, Not Tenant Count

**Old thinking:** count tenants → limit tenants per plan  
**Problem:** "20 tenants" means nothing to an owner. They think in rooms and beds.

**New thinking:**
- A **room-based** owner has 1 tenant per room → 5 rooms = 5 tenants (simple)
- A **bed-based PG** owner has 2-4 tenants per room → 5 rooms × 3 beds = 15 tenants (complex)

The **Free plan** supports room-based only. This is:
1. Simpler to build and support
2. Lower WhatsApp message volume (fewer tenants per room)
3. A natural upgrade path when the owner adds a PG or expands beds

---

## 3. Your Real Costs Every Month

### Fixed Infra (roughly)

| Service | Cost | Notes |
|---|---|---|
| Supabase (DB + Auth + Realtime) | ₹2,100/month | Free tier works for first 30-40 customers |
| Redis / Upstash (BullMQ job queues) | ₹840/month | Free tier works for first 100 jobs/day |
| NestJS hosting (Railway) | ₹1,680/month | Starter plan has cold-starts |
| Interakt WhatsApp BSP | ₹999/month | Starter plan, up to 1,000 conversations |
| Domain + misc | ₹83/month | |
| **Total fixed** | **~₹5,700/month** | Ballpark for production |

### Variable: WhatsApp Cost Per Bed

Meta charges **~₹0.58 per utility message** (each reminder or receipt).  
RentMaster sends **3 messages per tenant per month** (3-day warning, due-date, overdue).

```
Cost per bed per month = 3 × ₹0.58 = ₹1.74
```

| Beds on platform | Monthly WhatsApp cost |
|---|---|
| 50 beds | ₹87 |
| 200 beds | ₹348 |
| 500 beds | ₹870 |
| 1,000 beds | ₹1,740 |

At 500 beds the total platform cost is only ₹870 + ₹5,700 = **₹6,570/month**. Very manageable.

---

## 4. The Plans

### FREE — ₹0/month

**Who it's for:** Residential landlord with 3-5 flats in one building. Currently using Excel + WhatsApp manually.

| What's included | Value |
|---|---|
| 1 property | Fixed |
| Up to **5 rooms** | Hard limit |
| **Room-based only** — 1 tenant per room | Bed-based PG not available on Free |
| Automated WhatsApp reminders | 3-day + due-date sequence |
| Cashfree payment links | Full |
| WhatsApp receipts | Full |
| Live dashboard | Full |
| PDF/CSV export | Not included |
| Support | Docs only |

**Your infra cost for a Free user:** ~₹1.74 × 5 rooms = ₹8.70/month in WhatsApp. Nearly zero.  
**Goal of Free tier:** Let them feel the value of automation so they upgrade when they grow.

---

### STARTER — ₹249/month (or ₹2,490/year)

**Who it's for:** Small PG owner, 1 property, 10-20 beds, collecting ₹80,000-₹1,60,000/month.  
Platform fee = **0.15-0.3% of their rent revenue.** Extremely good value.

| What's included | Value |
|---|---|
| 1 property | Fixed |
| Up to **15 rooms** | |
| Up to **20 beds total** | PG bed-based unlocked |
| Full automated reminders (3-step) | 3-day + due-date + overdue |
| Manual overdue trigger | Owner can push a one-off reminder |
| Cashfree payment links | Full |
| WhatsApp receipts + owner alerts | Full |
| Live dashboard | Full |
| PDF/CSV export | Not included |
| Support | Email, 48h response |

**Your gross margin at 20 beds:** ₹249 revenue − ₹34.80 WhatsApp = **₹214 gross (86% margin)**

---

### PRO — ₹499/month (or ₹4,990/year)

**Who it's for:** Medium PG operator, 2-3 properties, 30-60 beds, collecting ₹2-6 lakhs/month.

| What's included | Value |
|---|---|
| Up to **3 properties** | |
| Up to **40 rooms** | |
| Up to **60 beds total** | |
| Everything in Starter | Full |
| **PDF rent receipts** | New |
| **CSV export** (monthly collection) | New |
| Digital agreements / eSign | Phase 3 (coming) |
| Support | Priority WhatsApp, 24h |

**Your gross margin at 60 beds:** ₹499 − ₹104 WhatsApp = **₹395 gross (79% margin)**

---

### SCALE — ₹999/month (or ₹9,990/year)

**Who it's for:** Large PG operator, 5-10 properties, 100+ beds, collecting ₹5-20 lakhs/month.

| What's included | Value |
|---|---|
| Up to **10 properties** | |
| **Unlimited rooms + beds** | |
| Everything in Pro | Full |
| Analytics dashboard (Phase 4) | Collection trends, occupancy rates |
| Tenant behaviour trends | Who pays late, who always pays early |
| REST API access | |
| Custom WhatsApp templates | |
| Dedicated account manager | |
| Priority phone support | |

**Your gross margin at 200 beds:** ₹999 − ₹348 WhatsApp = **₹651 gross (65% margin)**

---

## 5. Revenue: What You Need

### Your Monthly Target Costs

| Item | Monthly |
|---|---|
| Infra (Supabase + Redis + Railway + Interakt) | ₹5,700 |
| Your salary target | ₹50,000 |
| **Total needed** | **₹55,700/month** |

### How Many Customers to Get There

Realistic plan mix: **50% Starter + 35% Pro + 15% Scale**  
Average revenue per customer at this mix: **~₹449/month**

| Customers | Monthly Revenue | What It Pays |
|---|---|---|
| 20 customers | ₹8,980 | Covers infra only |
| 50 customers | ₹22,450 | Infra + ₹17,000 salary |
| 80 customers | ₹35,920 | Infra + ₹30,000 salary |
| **125 customers** | **₹56,125** | **Full break-even (infra + ₹50K salary)** |
| 200 customers | ₹89,800 | Infra + ₹84,000 salary |
| 300 customers | ₹1,34,700 | Comfortable |

**You need ~125 paying customers to be fully sustainable.** That's 125 PG owners in India — the country has millions of them.

---

## 6. When Free Users Upgrade (Conversion Triggers)

These are the moments to show the upgrade prompt in the UI:

1. **Room limit hit** — "You've reached 5/5 rooms. Add more rooms — upgrade to Starter."
2. **Bed-sharing request** — "Your tenant wants a shared room. Unlock bed-based PG on Starter."
3. **Second property added** — "Starter is needed to manage multiple properties."
4. **PDF receipt requested** — "Download PDF receipts — available on Pro and above."
5. **Growth message** — "You've collected ₹1,85,000 this month through RentMaster. Scale your PG on Pro."

---

## 7. Key Decisions Still Open

| Question | Options | Recommended |
|---|---|---|
| Annual pricing discount | 10% / 15% / 2 months free | 2 months free (feels bigger) |
| WhatsApp cost: absorb or pass on? | Absorb in plan price / charge extra above threshold | Absorb — simpler pricing |
| Referral program | Refer 1 paying owner → unlock 10 extra beds free | Yes — low cost, viral |
| Starter price: ₹199 or ₹249? | ₹199 feels cheap, ₹249 signals quality | ₹249 |
| GST invoice for owners | Phase 4 premium add-on / included | Phase 4 add-on (NFR-16) |

---

## 8. What To Update in the UX

The current `subscription.html` uses tenant-count limits (20/50/unlimited). It needs:

- [ ] Replace "20 tenants" → "15 rooms or 20 beds" on Starter card
- [ ] Replace "50 tenants" → "40 rooms or 60 beds" on Pro card
- [ ] Add "Room-based only" restriction badge on Free plan
- [ ] Add "Bed-based PG unlocked" badge on Starter+
- [ ] Add ₹249 Starter plan (currently missing — gap between Free ₹0 and Pro ₹599)
- [ ] Add annual billing toggle (monthly / yearly, 2 months free)
- [ ] Update FAQ with "What counts as a bed vs a room?" explanation
- [ ] Add conversion trigger UI (upgrade prompts at limit points)

---

## 9. One-Line Summary

> **Give 5 rooms free forever. Charge ₹249/month when they want PG beds or more rooms. Need 125 paying customers to cover infra + salary. No competitor does this under ₹500 in India.**

---

*Full brainstorming session log: `_bmad-output/brainstorming/brainstorming-session-2026-06-12-1015.md`*
