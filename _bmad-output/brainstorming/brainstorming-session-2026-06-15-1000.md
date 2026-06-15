---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/pricing-plan.md
  - _bmad-output/planning-artifacts/prds/prd-rent-master-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/epics.md
session_topic: 'RentMaster Feature Expansion — Electricity Billing + Shop Owners + Custom Plan Calculator + Cashfree Auto-Pay Mandates'
session_goals: 'Design electricity bill collection, commercial/shop property support, build-your-plan UI, and Cashfree UPI AutoPay mandates for tenants'
selected_approach: 'All techniques combined'
techniques_used: ['Competitive Gap Analysis', 'Data Model Design', 'Unit Economics', 'User Flow Mapping', 'Blue Ocean']
ideas_generated: []
context_file: ''
session_continued: false
---

# Brainstorming Session — Feature Expansion

**Facilitator:** Adarsh  
**Date:** 2026-06-15  
**Continues from:** brainstorming-session-2026-06-12-1015.md (Pricing Model)

---

## Pre-Session Discovery: What Already Exists

Before diving into the 4 new topics, a key observation from reading the pricing-plan.md:

**The per-slot model is already designed.** The ₹60/slot Tier 1 pricing means:
- 5 rooms × ₹60 = **₹300/month** ← exactly what Adarsh described ("5 rooms at ₹50 = ₹300")  
- 10 rooms × ₹60 = **₹600/month**
- 20 rooms × ₹60 = **₹1,200/month**

The "build your custom plan" idea the user described **IS the existing slot model** — it just needs a **pricing calculator UI** on the onboarding/pricing page. This is a UX gap, not a pricing architecture gap.

---

## TOPIC 1 — Electricity Bill Collection

### Why This Is Critical for PG Owners

In India, electricity is one of the most pain-filled parts of PG management:
- Owner reads meter every month (walks to the meter box)
- Calculates manually: (closing − opening reading) × rate
- Texts each tenant separately with the amount
- Tenant pays it separately or confusingly alongside rent
- No audit trail, disputes are common

**Electricity is a MONTHLY problem for every PG tenant.** Solving it dramatically increases RentMaster's stickiness — an owner who uses it for electricity AND rent has no reason to leave.

---

### Three Billing Modes (Cover Every Indian PG Type)

#### Mode 1: Per-Room Sub-Meter (Most Common in Larger PGs)
Each room has its own sub-meter. Owner reads it once a month.

**How it works:**
1. Owner sets up room: enters meter number, rate per unit (e.g., ₹7/unit)
2. Monthly: opens RentMaster → "Enter Meter Readings" flow
3. Inputs: Room 101 → Opening: 4,520 → Closing: 4,650 → System calculates: 130 units × ₹7 = ₹910
4. System auto-adds ₹910 to Room 101's rent for that month
5. Combined payment link: Rent ₹8,000 + Electricity ₹910 = **₹8,910 due**
6. WhatsApp message shows breakdown clearly

#### Mode 2: Flat Rate per Room/Bed (Most Common in Small PGs)
Simpler PGs charge a fixed electricity amount regardless of usage.

**How it works:**
- Owner sets: "₹500/bed/month electricity"
- System auto-adds ₹500 to every bed's rent record
- No meter reading required
- WhatsApp: "Rent ₹7,000 + Electricity ₹500 = ₹7,500"

#### Mode 3: Shared Bill Split (Apartments + Small Houses)
Owner receives one electricity bill from the board and splits it equally.

**How it works:**
1. Owner receives EB bill: ₹4,200 for the month
2. Opens RentMaster: "Enter Shared Bill" → inputs ₹4,200
3. System divides by active tenant count (e.g., 6 tenants): ₹700/tenant
4. Added to each tenant's rent record

---

### Data Model Requirements

**New table: `electricity_config`**
```sql
id uuid
property_id uuid → FK properties
billing_mode ENUM ('sub_meter', 'flat_rate', 'shared_split')
rate_per_unit DECIMAL  -- for sub_meter mode
flat_amount DECIMAL    -- for flat_rate mode
created_at, updated_at
```

**New table: `meter_readings`**
```sql
id uuid
room_id uuid → FK rooms
month_year VARCHAR(7)   -- e.g. "2026-06"
opening_reading DECIMAL
closing_reading DECIMAL
units_consumed DECIMAL  -- computed: closing − opening
electricity_amount DECIMAL  -- computed: units × rate
entered_by uuid → FK owners
created_at
```

**Modify `rent_records`:**
```sql
electricity_amount DECIMAL DEFAULT 0
total_amount DECIMAL  -- rent_amount + electricity_amount
electricity_mode ENUM ('sub_meter', 'flat_rate', 'shared_split', 'none')
```

---

### UI/UX Flow for Electricity Collection

**Room Setup (added field):**
- Toggle: "Include electricity billing?" → if yes → select mode
- If sub_meter: enter meter number + rate per unit
- If flat_rate: enter fixed monthly amount
- If shared_split: (configured at property level, not room level)

**Monthly Collection Flow (new step before sending reminders):**
- On 28th of month: dashboard shows "Enter meter readings before reminders go out"
- Batch reading entry screen: table with all rooms → opening (auto-filled from last month's closing) → closing → auto-calculated amount
- "Generate Bills" button → creates rent records with electricity amounts
- Reminders then go out with combined total

**Tenant WhatsApp Message (updated template):**
```
Hi [Name], your June 2026 bill is ready:
🏠 Rent: ₹8,000
⚡ Electricity (130 units × ₹7): ₹910
💳 Total Due: ₹8,910
Pay here: [link]
Due date: 5th July
```

---

### Edge Cases to Handle

| Scenario | Solution |
|---|---|
| New tenant moves in mid-month | Prorate electricity: (days_occupied / total_days) × amount |
| Meter replaced (reading resets to 0) | "Meter replaced" flag on reading entry — system stores separately |
| Owner forgets to enter readings | Auto-reminder WhatsApp to owner on 27th: "Enter meter readings to generate bills" |
| AC room premium | Option: add fixed AC surcharge per room (flat amount on top of sub-meter) |
| Common area electricity | Separate field at property level: distribute as fixed amount per tenant |
| Owner not on sub-meter mode but has one-off high bill | Manual override: add electricity amount to individual rent record manually |

---

### Blue Ocean Opportunities in Electricity Billing

1. **Meter reading OCR** — Owner photographs meter → camera crops + reads number automatically (ML) — Phase 4 feature
2. **Usage trend per room** — "Room 201 uses 20% more power than average this month" → helps owner detect AC misuse
3. **Electricity advance tracking** — Some owners collect 2-month electricity deposit → track it in system
4. **Annual consumption summary** — Owner can export tenant electricity usage per year
5. **Rate change notification** — If state electricity board changes rates → system can update all configs at once

---

### Revenue Impact of Electricity Feature

**As a paid add-on:**
- Electricity billing module: +₹10/slot/month
- A 20-slot owner: +₹200/month additional revenue for RentMaster
- This is meaningful but small vs base subscription

**As a free inclusion:**
- Include in all tiers → increases stickiness → better retention
- Retention improvement worth more than ₹200/month extra

**Recommendation:** Include in all paid tiers (Tier 1+), not in trial. Makes trial-to-paid conversion have a "you lose electricity automation too" message.

---

## TOPIC 2 — Shop / Commercial Property Owners

### The Opportunity

The PRD currently says "Commercial/enterprise property managers — not targeted until Phase 4+." This was based on complexity assumptions. But there's a simpler path:

**Many PG owners also own ground-floor shops.** If RentMaster already manages their PG, and they have 3 shops on the ground floor, they want to manage those too in the same dashboard. This is not a new customer type — it's the SAME customer with an additional property type.

**Target: Small commercial landlords who:**
- Own 1-10 shops/offices in one or two buildings
- Currently use the same manual WhatsApp + Excel process
- Want the same automated reminder + collection system

---

### What's Different About Shops

| Dimension | Residential/PG | Commercial Shop |
|---|---|---|
| Unit term | Room / Bed | Shop / Unit / Office / Cabin |
| Tenants per unit | 1–5 (shared beds) | 1 (always 1 business) |
| Monthly rent | ₹5K–₹25K | ₹10K–₹5,00,000 |
| Agreement length | 11 months | 3–5 years |
| Security deposit | 1–3 months | 6–12 months |
| CAM charges | Rare | Common (maintenance + security + cleaning) |
| GST on rent | No | Yes (if tenant is registered) |
| Rent escalation clause | Informal | Formal (5–10% per year) |
| Payment method | UPI (under ₹1L) | NEFT/RTGS (for amounts above ₹1L) |

---

### Minimum Viable Commercial Feature Set

**What we add (Phase 2, not Phase 4):**

1. **Property type: "Commercial"** — new option in property setup alongside PG / Residential
2. **Unit type terminology** — when property is Commercial, "Room" → "Shop", "Tenant" → "Business Tenant"
3. **CAM Charges** — optional line item per shop: ₹X/month (maintenance, security, common area)
4. **Rent escalation tracker** — owner sets escalation % (e.g., 5%) and next escalation date → RentMaster reminds 60 days before
5. **Security deposit tracking** — record deposit amount paid; show it on tenant profile
6. **Agreement expiry alerts** — flag shops where agreement expires within 60/30 days

**What we DON'T add in Phase 2 (keep it simple):**
- GST invoice generation (complex, Phase 4)
- Digital agreement execution
- Revenue-share model (mall scenarios)
- NEFT-only payment (Cashfree handles amounts up to ₹5L via UPI corporate accounts)

---

### Commercial in the Slot Model

The existing pricing-plan.md uses: 1 bed = 1 slot, 1 room = 1 slot. Simple extension:

**1 shop/unit = 1 slot — but at a premium rate.**

Why premium? Shop owners:
- Have higher rent amounts → value of automation is higher
- Will likely want CAM billing, escalation tracking (extra features)
- Fewer units but willing to pay more per unit

**Proposed commercial slot rate:**
- Tier 1: ₹80/slot/month (vs ₹60/slot for residential)
- Tier 2: ₹65/slot/month (vs ₹50/slot)
- Tier 3: ₹50/slot/month (vs ₹40/slot)

**Mixed portfolio example:**
```
Owner has:
├── PG: 20 beds = 20 slots × ₹60 = ₹1,200/month
└── Shops: 3 units = 3 slots × ₹80 = ₹240/month
Total: ₹1,440/month
```

System shows one bill: "23 slots (20 residential + 3 commercial) — ₹1,440/month"

---

### Commercial Whatsapp Templates Needed

```
[Rent Reminder]
Hi [Business Name], your shop rent of ₹35,000 for June 2026 is due on 1st June.
🏪 Shop: Unit 3, Ground Floor, Sunshine Complex
Landlord: [Owner Name]
Pay here: [link]

[CAM Reminder — if applicable]
Rent ₹35,000 + CAM ₹2,500 = Total ₹37,500
```

---

### Who This Unlocks as New Customers

| Segment | Size | Context |
|---|---|---|
| Building owner (mixed use) | Large | Already targeting for PG — shop is free acquisition |
| Small market/complex owner | Medium | 5-15 shops in one building, high value per unit |
| Office park owner | Small | 10-20 cabins, higher ticket, needs escalation tracking |

**Estimate:** 20-30% of existing PG target owners also have commercial units. This is not a new acquisition cost — it's expanding ARPU from existing customers.

---

## TOPIC 3 — Custom Build-Your-Plan (Pricing Calculator)

### Key Discovery

The existing pricing-plan.md **already IS per-unit pricing**: ₹60/slot/month Tier 1. A user with 5 rooms pays 5 × ₹60 = ₹300/month. This is the "custom plan" concept — it's just not presented that way in the UI.

**The gap is not the pricing model — it's the calculator UI and the Cashfree auto-subscription.**

### The "Build Your Plan" Calculator

**Where it lives:** Pricing page, onboarding Step 0 (before sign-up), and billing settings page.

**User interaction:**
```
┌─────────────────────────────────────────────────────┐
│  Build Your Plan                                    │
│                                                     │
│  I have:                                            │
│  [  5  ] rooms / flats        (residential)         │
│  [  0  ] beds                 (PG property)         │
│  [  0  ] shops / offices      (commercial)          │
│                                                     │
│  ─────────────────────────────────────────────────  │
│  Your plan:                                         │
│  5 residential slots × ₹60 = ₹300                  │
│  Monthly total: ₹300/month                          │
│  Annual (2 months free): ₹3,000/year                │
│                                                     │
│  [Start 14-day free trial →]                        │
└─────────────────────────────────────────────────────┘
```

Key UX principle: **Show the math transparently.** "5 × ₹60 = ₹300" is more trustworthy than "Starter Plan — ₹1,199/month (up to 30 slots)." The user sees they're paying exactly for what they have.

**Slider variant for onboarding:**
- Slide beds from 0 to 200+, price updates in real time
- "You're in Tier 1 (₹60/slot)" label shows which tier they're in
- When they cross 30 slots: label changes to "You've entered Tier 2 (₹50/slot)" — and total goes DOWN per unit

---

### Custom Plan + Cashfree Subscription for Platform Fees

This is the most powerful idea: **RentMaster uses its own Cashfree integration to collect its own subscription fees.**

**The flow:**
1. Owner finishes trial (day 14) or signs up to paid
2. "Set up auto-pay for your RentMaster subscription — one-time setup, never think about it again"
3. Owner authorizes ₹300/month (calculated for their 5 slots) via UPI AutoPay or e-NACH
4. Every month on 5th: Cashfree auto-debits ₹300 from owner's bank
5. Owner gets WhatsApp: "Your RentMaster subscription of ₹300 was auto-debited for July 2026. Thank you!"

**When the owner's slot count changes:**
- Owner adds 5 more rooms → total now 10 slots → ₹600/month
- System sends WhatsApp: "Your plan has been updated: 10 slots × ₹60 = ₹600/month. Your next debit on 5th Aug will be ₹600."
- Cashfree subscription updated via API

**This product story is powerful:**
> "We automate rent collection for your tenants. We automate subscription collection for ourselves. Same system, same trust."

---

### Annual vs Monthly Plan

Per pricing-plan.md: annual = 2 months free (10 months price for 12 months).

For annual: owner pays upfront OR sets up a single annual mandate:
- 5 slots × ₹60 × 10 months = ₹3,000/year (instead of ₹3,600)
- UPI AutoPay for annual amount → Cashfree charges ₹3,000 once on anniversary date

---

### What NOT to Build in "Custom Plans"

The previous brainstorming session flagged per-unit pricing as risky due to "billing anxiety." That risk is addressed by:
1. Owner sets their slot count ONCE at setup (not variable monthly)
2. System only changes billing if owner explicitly adds properties
3. Price is locked and transparent — no mystery charges

**Do NOT build:**
- Variable monthly billing based on actual usage (causes anxiety)
- Billing based on "active tenant count this month" (confusing)
- Complex add-on bundles beyond electricity module

---

## TOPIC 4 — Cashfree Auto-Payment Mandates for Tenants

### Current State vs Dream State

**Current state:** WhatsApp reminder → tenant sees link → tenant clicks → pays → receipt
**Dream state:** Tenant authorizes once → money debited automatically every month → receipt

This is a **fundamental upgrade in the value proposition**:
- Current: "We remind your tenants to pay"
- With mandates: "We collect your rent for you, automatically"

---

### How Cashfree Subscriptions Work (Technical)

Cashfree offers a **Subscriptions API** that handles:
- UPI AutoPay (NPCI) — works on PhonePe, GPay, Paytm, BHIM
- e-NACH — bank account direct debit (2-3 day activation)
- eMandate — net banking authorization

**Recommended: UPI AutoPay** for Indian tenants:
- Familiar (they already use PhonePe/GPay)
- Instant activation (30 seconds)
- Limit: ₹1,00,000/transaction (sufficient for almost all residential rents)
- Mobile-first — perfect for WhatsApp-led flow

**API flow:**
```
1. POST /subscriptions → create subscription plan (amount, cycle: monthly, start_date)
2. GET /subscriptions/{id}/auth_link → get authorization URL
3. Owner sends URL to tenant via WhatsApp
4. Tenant opens URL → authorizes in their UPI app
5. System receives: SUBSCRIPTION_AUTHORIZED webhook → store mandate ID
6. On due date: Cashfree auto-charges → PAYMENT_SUCCESS webhook → mark paid → send receipt
7. If fails: PAYMENT_FAILED → fallback to payment link reminder flow
```

---

### Integration with Existing System

**Changes to tenant onboarding:**
- After tenant is added: "Send auto-pay setup request?" → yes → WhatsApp with mandate link
- Tenant profile shows: "Auto-pay: Active / Pending / Not set up"

**Changes to reminder engine (BullMQ):**
- For tenants with active mandate: skip "generate payment link" step
- Wait for Cashfree auto-debit on due date
- If Cashfree debits successfully: PAYMENT_SUCCESS webhook → mark paid → send receipt (same as current)
- If auto-debit fails: immediately send payment link reminder (fall back to current flow)

**New webhook events to handle:**
- `SUBSCRIPTION_AUTHORIZED` — mandate is active, no immediate payment
- `SUBSCRIPTION_CHARGE_SUCCESS` — payment successful this month
- `SUBSCRIPTION_CHARGE_FAILED` — auto-debit failed → trigger reminder fallback
- `SUBSCRIPTION_CANCELLED` — mandate cancelled by tenant (owner notification needed)

---

### Failure Handling (Critical)

| Scenario | Response |
|---|---|
| Auto-debit fails (insufficient funds) | Immediately send payment link WhatsApp → fall back to manual |
| Auto-debit fails second time | Notify owner: "Arjun's auto-pay failed twice. Take action." |
| Tenant cancels mandate | Notify owner immediately → switch tenant back to manual mode |
| Rent amount changes | Mandate must be updated: send new auth link to tenant |
| Tenant moves out | Cancel mandate immediately on move-out |

---

### Tenant Communication Script (WhatsApp)

**Initial setup request:**
```
Hi Arjun! Priya's PG Koramangala uses RentMaster.
Set up auto-pay for your monthly rent of ₹8,000 — 
pay automatically every month without lifting a finger.
Click to set up (takes 30 seconds): [link]
Powered by UPI AutoPay (safe & reversible anytime)
```

**After successful setup:**
```
✅ Auto-pay set up successfully!
Your rent of ₹8,000 will be auto-debited on 1st of every month.
You'll receive a receipt WhatsApp after each deduction.
To cancel anytime: your UPI app → Manage Mandates
```

**Receipt after auto-debit:**
```
✅ Rent paid automatically!
Amount: ₹8,000
For: July 2026
Property: Priya's PG, Room 3
No action needed. See you next month!
```

---

### How This Changes the Product Conversation

**With payment link reminders:**
> "RentMaster sends WhatsApp reminders so tenants don't forget to pay."

**With auto-pay mandates:**
> "Once your tenants set up auto-pay, rent collection is fully zero-touch. You don't send reminders. Cashfree debits the rent on the due date and you get a WhatsApp notification. That's it."

This is a marketing step-change. It turns RentMaster from a "reminder tool" to a "rent collection engine."

---

### Variable Amount Challenge (Relevant for Electricity)

UPI AutoPay supports **variable amount mandates** with a maximum limit set:
- Set max: ₹12,000/month for a tenant whose rent is ₹8,000 (buffer for electricity)
- Each month: Cashfree charges the actual amount (rent + electricity for that month)
- Tenant's UPI app shows: "Up to ₹12,000/month may be debited"

This enables a combined rent + electricity mandate with variable monthly total.

---

### Revenue & Adoption Considerations

**For owners:** Mandate setup is a strong upsell pitch. "Auto-collect both rent AND electricity? Set it up once, done forever."

**Tenant adoption challenge:** Some tenants are wary of mandates. Solutions:
- Emphasize it's "reversible anytime" in the WhatsApp message
- Show the max limit prominently (trust signal)
- Owner can also keep some tenants on manual flow — mandate is optional per-tenant

**Platform differentiation:**
- GoPGMS: no mandates
- TrackMyPG: no mandates
- NoBroker: payment collection but NOT direct to owner (they hold funds)
- **RentMaster: mandate → money goes DIRECTLY to owner's bank. Only platform to offer this.**

---

## SYNTHESIZED FEATURE ROADMAP

### Phase 2 (Next after current Phase 1):
- [ ] Electricity billing module (all 3 modes)
- [ ] Commercial property type (shops/offices)
- [ ] "Build Your Plan" pricing calculator UI
- [ ] Cashfree auto-pay mandate setup for tenants (UPI AutoPay first)

### Phase 3:
- [ ] Electricity meter OCR (camera reads meter number)
- [ ] Commercial escalation clause tracker
- [ ] Combined rent + electricity variable mandate
- [ ] Annual mandate / subscription for platform billing

### Phase 4:
- [ ] GST invoice generation for commercial tenants
- [ ] Commercial digital agreement templates
- [ ] Electricity usage analytics

---

## UPDATED SLOT PRICING MODEL (With Commercial + Electricity)

### Slot Rates

| Slot Type | Tier 1 (1-30 slots) | Tier 2 (31-100 slots) | Tier 3 (100+) |
|---|---|---|---|
| Residential room | ₹60/slot | ₹50/slot | ₹40/slot |
| PG bed | ₹60/slot | ₹50/slot | ₹40/slot |
| Commercial shop/unit | ₹80/slot | ₹65/slot | ₹50/slot |

### Electricity Add-on (Optional)

| Mode | Price |
|---|---|
| Include electricity billing | +₹10/slot/month |
| (covers sub-meter, flat-rate, shared-split, unlimited meter entries) | |

### Mixed Portfolio Example (Owner with PG + Shops + Electricity)

```
10 PG beds    × ₹60 = ₹600
3 shops       × ₹80 = ₹240
Electricity   × ₹10 × 13 total units = ₹130
─────────────────────────────
Monthly total: ₹970/month
```

---

## NEW FUNCTIONAL REQUIREMENTS (For PRD Update)

### Electricity Module
- FR-E1: Electricity Config — Owner can set electricity billing mode per property/room
- FR-E2: Meter Reading Entry — Owner can enter monthly meter readings in bulk for all rooms
- FR-E3: Electricity Amount Calculation — System calculates electricity amount from readings
- FR-E4: Combined Bill Generation — Rent record includes electricity amount; payment link is for combined total
- FR-E5: Combined WhatsApp Template — Reminder message shows rent + electricity breakdown
- FR-E6: Owner Reminder for Readings — System sends owner WhatsApp on 27th to enter readings before reminders

### Commercial Module
- FR-C1: Commercial Property Type — Owner can create a property with type "Commercial"
- FR-C2: Commercial Unit Terminology — Shop/Unit/Office terminology in all UX
- FR-C3: CAM Charges — Owner can set monthly CAM charge per commercial unit
- FR-C4: Security Deposit Tracking — Record and display deposit amount per commercial tenant
- FR-C5: Rent Escalation Tracker — Owner sets escalation % and date; system alerts 60/30 days before
- FR-C6: Agreement Expiry Alerts — System flags commercial agreements expiring within 60 days
- FR-C7: Commercial Slot Billing — Commercial units billed at ₹80/slot (premium rate)

### Custom Plan Calculator
- FR-P1: Plan Calculator UI — Interactive calculator on pricing page and onboarding
- FR-P2: Real-time price calculation — As user enters slot counts, total price updates live
- FR-P3: Cashfree Subscription for Platform Fee — After trial, owner sets up UPI AutoPay mandate for RentMaster subscription
- FR-P4: Plan Update on Slot Change — When owner adds properties, WhatsApp notification of new billing amount

### Auto-Pay Mandates
- FR-M1: Mandate Setup Flow — Owner can initiate mandate setup for a tenant from tenant profile
- FR-M2: Tenant Mandate WhatsApp — System sends WhatsApp to tenant with UPI AutoPay link
- FR-M3: Mandate Authorization Webhook — System receives SUBSCRIPTION_AUTHORIZED → marks tenant as auto-pay
- FR-M4: Auto-Debit Reminder Skip — Tenants with active mandates skip payment link reminder
- FR-M5: Auto-Debit Receipt — On SUBSCRIPTION_CHARGE_SUCCESS → send WhatsApp receipt
- FR-M6: Auto-Debit Failure Fallback — On SUBSCRIPTION_CHARGE_FAILED → immediately send payment link reminder
- FR-M7: Mandate Cancellation Handling — On SUBSCRIPTION_CANCELLED → notify owner, revert tenant to manual
- FR-M8: Move-Out Mandate Cancellation — On tenant move-out, auto-cancel any active mandate

---

## OPEN QUESTIONS / DECISIONS NEEDED

| Question | Options | Recommendation |
|---|---|---|
| Electricity: free or paid add-on? | Free in all tiers / +₹10/slot | Include in all paid tiers (stickiness) |
| Commercial: Phase 2 or Phase 4? | Phase 2 (same stack) / Phase 4 (wait) | Phase 2 — same stack, high ARPU from existing owners |
| Commercial slot premium? | Same rate / ₹80/slot | ₹80/slot — value delivered is higher |
| Mandate: which type first? | UPI AutoPay / e-NACH / Both | UPI AutoPay first — instant setup, familiar |
| Plan calculator: before or after sign-up? | Before (reduces friction) / After | Before sign-up on pricing page |
| Variable mandate for electricity? | Yes (combined) / No (separate links) | Phase 3 — keep Phase 2 simple with separate bill |

---

*Session completed: 2026-06-15. All 4 topics covered: Electricity Billing · Commercial Properties · Custom Plan Calculator · Cashfree Auto-Pay Mandates.*
*Linked planning doc: `_bmad-output/planning-artifacts/feature-expansion-electricity-commercial-autopay.md`*
