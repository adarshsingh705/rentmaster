# RentMaster — Feature Expansion Planning
**Date:** 2026-06-15  
**Author:** Adarsh  
**Status:** Draft — pending PRD integration

---

## Overview

This document captures the design and decisions for 4 Phase 2+ feature areas raised on 2026-06-15:

1. **Electricity Bill Collection** — automated electricity billing alongside rent
2. **Commercial / Shop Properties** — support for shops, offices, and cabins
3. **Build-Your-Plan Calculator** — interactive pricing UI using the existing slot model
4. **Cashfree Auto-Pay Mandates** — UPI AutoPay for fully automated tenant rent collection

These features build on the Phase 1 foundation (NestJS + Cashfree + BullMQ + WhatsApp) without requiring a new stack.

---

## 1. Electricity Bill Collection

### Problem Statement

Indian PG and residential owners have two monthly collection problems:
- Rent (already solved by Phase 1)
- Electricity (currently 100% manual — meter reading → calculation → WhatsApp → separate payment)

Electricity billing is monthly, recurring, and directly tied to the tenant's room. It belongs in RentMaster.

### Three Billing Modes

| Mode | How Owner Sets It Up | How It Bills | Best For |
|---|---|---|---|
| **Sub-meter** | Meter ID + rate per unit (e.g., ₹7/unit) | Monthly readings entered → units × rate = amount | Larger PGs with room-level meters |
| **Flat rate** | Fixed amount per room/bed (e.g., ₹500/month) | Auto-added to every rent record | Small PGs, simpler billing |
| **Shared split** | None (property-level) | Owner enters total EB bill → divided by active tenant count | Apartments, small houses |

### Database Schema Additions

```sql
-- New table: electricity billing mode per property
CREATE TABLE electricity_config (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  property_id UUID NOT NULL REFERENCES properties(id) ON DELETE CASCADE,
  billing_mode VARCHAR(20) NOT NULL CHECK (billing_mode IN ('sub_meter', 'flat_rate', 'shared_split', 'none')),
  rate_per_unit DECIMAL(10,2),       -- sub_meter only
  flat_amount DECIMAL(10,2),         -- flat_rate only
  owner_id UUID NOT NULL REFERENCES owners(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Per-room meter config (sub_meter mode only)
CREATE TABLE room_meters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE,
  meter_number VARCHAR(50),
  current_reading DECIMAL(10,2) DEFAULT 0,  -- carry-forward closing reading
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Monthly meter readings
CREATE TABLE meter_readings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id),
  month_year CHAR(7) NOT NULL,              -- format: "2026-06"
  opening_reading DECIMAL(10,2) NOT NULL,
  closing_reading DECIMAL(10,2) NOT NULL,
  units_consumed DECIMAL(10,2) GENERATED ALWAYS AS (closing_reading - opening_reading) STORED,
  rate_per_unit DECIMAL(10,2) NOT NULL,     -- snapshot of rate at time of entry
  electricity_amount DECIMAL(10,2) GENERATED ALWAYS AS ((closing_reading - opening_reading) * rate_per_unit) STORED,
  owner_id UUID NOT NULL REFERENCES owners(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add to rent_records:
ALTER TABLE rent_records
  ADD COLUMN electricity_amount DECIMAL(10,2) DEFAULT 0,
  ADD COLUMN electricity_mode VARCHAR(20) DEFAULT 'none',
  ADD COLUMN total_amount DECIMAL(10,2);    -- rent_amount + electricity_amount
```

### API Endpoints Needed

```
POST   /api/owner/properties/:id/electricity-config     -- set mode
PUT    /api/owner/properties/:id/electricity-config     -- update mode
GET    /api/owner/properties/:id/electricity-config     -- get current config
POST   /api/owner/rooms/:id/meter-readings              -- enter monthly reading
GET    /api/owner/rooms/:id/meter-readings              -- history
POST   /api/owner/properties/:id/shared-bill            -- enter shared bill (split mode)
```

### WhatsApp Template Update

Existing template `rent_reminder` must be updated to include optional electricity breakdown:

```
Hi {{tenant_name}}, your {{month}} bill is ready:
🏠 Rent: ₹{{rent_amount}}
⚡ Electricity: ₹{{electricity_amount}}
💳 Total Due: ₹{{total_amount}}
Pay here: {{payment_link}}
Due: {{due_date}}
```

If electricity = 0, the electricity line is omitted.

**Note:** Meta WhatsApp template changes require re-approval. The existing `rent_reminder` template must be submitted for re-approval with the optional electricity component. Factor 5–7 business days into timeline.

### Owner Flow — Monthly Billing

```
Timeline:
  1st of month → BullMQ creates rent records (existing FR-16)
  27th of month → [NEW] System WhatsApp to owner: "Enter meter readings before reminders go out"
  28th–30th    → Owner enters readings in bulk table
  1st of next  → Reminders go out with electricity included in total
```

### Edge Cases

| Edge Case | Handling |
|---|---|
| Tenant moves in mid-month | Electricity prorated: (days_in_room / days_in_month) × amount |
| Meter replaced (reading resets) | "Meter Reset" flag on reading form → treated as day 1 reading |
| Owner forgets readings by 30th | Reminders go out with electricity_amount = 0, owner can trigger separately |
| AC surcharge | Flat surcharge field per room on top of sub-meter amount |
| Common area lights/lifts | Property-level flat amount split equally (variant of shared_split) |

### Pricing Decision

**Recommendation: Include electricity billing in all paid tiers (Tier 1+), not as add-on.**

Rationale:
- Electricity + rent = the two monthly pain points → solving both = maximum stickiness
- "Add ₹10/slot" creates friction and complexity during signup
- Retention value of electricity module >> ₹10/slot/month additional revenue
- Marketing angle: "Automate rent AND electricity from one dashboard"

---

## 2. Commercial / Shop Properties

### Why Phase 2, Not Phase 4

The PRD scoped commercial to Phase 4 based on complexity. But the actual implementation is:
- Same NestJS backend (property type field already exists)
- Same Cashfree payment links (work for any amount)
- Same WhatsApp templates (minor wording change)
- Same BullMQ reminder engine
- 2 new DB columns, 1 new table (CAM + escalation), minimal UI changes

**The complexity was overestimated. The majority of Phase 1 stack works unchanged for commercial.**

More importantly: 20–30% of existing PG owner prospects also own shops in the same building. Adding commercial support converts a property-manager customer into a full-portfolio customer — higher ARPU at zero acquisition cost.

### New Property Type: Commercial

**During property setup:** Add `commercial` to the property type selector alongside `pg` and `residential`.

**When property type = `commercial`:**
- "Rooms" → "Units" (or "Shops" / "Offices" — owner can name them)
- "Tenants" → "Business Tenants"
- Bed concept doesn't apply (always 1 occupant per unit)
- 3 new fields become available: CAM charge, security deposit, escalation clause

### Database Schema

```sql
-- Add to properties table:
ALTER TABLE properties
  ADD COLUMN property_type VARCHAR(20) NOT NULL DEFAULT 'residential'
    CHECK (property_type IN ('residential', 'pg', 'commercial'));

-- Commercial-specific config per unit (room):
CREATE TABLE commercial_configs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  room_id UUID NOT NULL REFERENCES rooms(id) ON DELETE CASCADE UNIQUE,
  cam_charge DECIMAL(10,2) DEFAULT 0,             -- monthly CAM
  security_deposit_paid DECIMAL(10,2) DEFAULT 0,  -- deposit received from tenant
  escalation_pct DECIMAL(5,2),                    -- e.g. 5.00 = 5%
  next_escalation_date DATE,                      -- when next increase is due
  agreement_start_date DATE,
  agreement_end_date DATE,
  owner_id UUID NOT NULL REFERENCES owners(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add to rent_records:
ALTER TABLE rent_records
  ADD COLUMN cam_amount DECIMAL(10,2) DEFAULT 0;
```

### Slot Billing for Commercial

Extend the slot billing logic in the billing service:

```typescript
// Existing logic
const residentialSlots = owner.rooms.filter(r => r.property.type !== 'commercial').length
// New
const commercialSlots = owner.rooms.filter(r => r.property.type === 'commercial').length

// Pricing
const residentialCost = residentialSlots <= 30
  ? residentialSlots * 60
  : 30 * 60 + Math.min(residentialSlots - 30, 70) * 50 + Math.max(residentialSlots - 100, 0) * 40

const commercialCost = commercialSlots <= 30
  ? commercialSlots * 80
  : 30 * 80 + Math.min(commercialSlots - 30, 70) * 65 + Math.max(commercialSlots - 100, 0) * 50

const totalMonthly = residentialCost + commercialCost
```

### Commercial Escalation Alerts (BullMQ)

New cron job: daily scan at 7 AM IST for commercial units where `next_escalation_date` is within 60 or 30 days.

```
WhatsApp to owner (60 days before):
"📋 Rent escalation due soon: Unit 3, Sunshine Complex
Current rent: ₹35,000 | Escalation: 5% | New rent: ₹36,750
Due date: 1st August 2026
You can update the rent in RentMaster now → [link]"
```

### Agreement Expiry Alerts (BullMQ)

Same pattern: scan for commercial agreements expiring in 60/30 days → WhatsApp to owner.

### What's NOT in Phase 2 Commercial

| Feature | Why Excluded | Target Phase |
|---|---|---|
| GST invoice generation | Complex tax logic, GSTIN validation | Phase 4 |
| Digital agreement execution | eSign integration | Phase 3 |
| NEFT-only payment | Cashfree UPI handles up to ₹1L; above that UPI may not work | Phase 4 |
| Revenue-share model (malls) | Entirely different billing structure | Phase 5+ |

---

## 3. Build-Your-Plan Calculator

### Core Insight

The existing slot pricing model (₹60/slot/month) IS a custom plan calculator — it just doesn't have a UI. An owner with 5 rooms pays exactly 5 × ₹60 = ₹300/month. This is:
- Fair: they pay for exactly what they use
- Transparent: the math is visible
- What Adarsh described: "5 rooms × ₹50 = ₹300"

The per-slot rate is ₹60 (not ₹50 as described), but the concept is identical. The gap is a pricing calculator UI.

### Calculator UI Design

**Location 1: Pricing page (before sign-up)**

```
How much will RentMaster cost you?

Property type:  ○ PG (beds)  ○ Residential (rooms)  ○ Commercial (shops)  ○ Mixed

How many units?
  Rooms/flats:  [____] × ₹60 = ₹____
  PG beds:      [____] × ₹60 = ₹____  
  Shops/offices:[____] × ₹80 = ₹____

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Your plan:  ₹____ / month
            ₹____ / year  (2 months free)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[ Start 14-day free trial — no card needed ]
```

Key design decisions:
- Show the calculation transparently (5 × ₹60 = ₹300, not just "₹300/month")
- Tier label appears when they cross thresholds: "You've entered Tier 2 — rate drops to ₹50/slot"
- Annual option shows total savings in rupees: "Save ₹600/year"

**Location 2: Onboarding step 0 (after sign-up, before adding properties)**

Same calculator, pre-populated as they add properties in the onboarding wizard. "Your current plan: 5 rooms = ₹300/month" updates live as they add rooms.

**Location 3: Billing settings page (for existing owners)**

Shows current slot count, current monthly amount, and lets owner preview what adding more slots would cost.

### Cashfree Subscription for Platform Fee

After the 14-day trial or on first subscription:

**Flow:**
1. Trial ends → "Your trial is over. Your plan: 5 slots = ₹300/month."
2. "Pay automatically every month with UPI AutoPay" button
3. Owner taps → Cashfree subscription mandate page
4. Authorizes in their UPI app (GPay / PhonePe / Paytm)
5. `SUBSCRIPTION_AUTHORIZED` webhook → mark owner subscription as active
6. 5th of each month: Cashfree auto-debits ₹300 → `SUBSCRIPTION_CHARGE_SUCCESS` → WhatsApp receipt to owner
7. If debit fails: 3-day grace → WhatsApp reminder → account suspended after 7 days

**Alternative: manual payment link** (for owners who don't want mandate)
- Less ideal — requires manual action each month
- Offer as fallback option, not default

### Subscription Lifecycle — When Slot Count Changes

**Design decision: one subscription per owner, updated in-place.** Never create separate subscriptions per property — two auto-debits on the same account is confusing. Never cancel + recreate on every change — owner re-authorizes every time is bad UX.

**The `max_amount` buffer is the key.** When the subscription is first created, set `max_amount` = current amount × 1.5 (rounded to nearest ₹100). Most slot additions fall within this buffer and need no re-authorization from the owner.

#### Schema

```sql
-- Tracks the owner's RentMaster platform subscription (one row per owner)
CREATE TABLE owner_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES owners(id) UNIQUE,
  cashfree_subscription_id VARCHAR(100) UNIQUE NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending_auth'
    CHECK (status IN ('pending_auth', 'active', 'suspended', 'cancelled')),
  current_amount DECIMAL(10,2) NOT NULL,   -- what Cashfree will debit next cycle
  max_amount DECIMAL(10,2) NOT NULL,       -- ceiling set on the mandate; updates within this need no re-auth
  next_billing_date DATE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### Update Logic (called whenever owner adds or removes a slot)

```typescript
async updateOwnerSubscription(ownerId: string): Promise<void> {
  const owner = await this.ownerService.findWithSlots(ownerId)
  const newAmount = this.calculateBilling(owner.slots)  // existing billing calc
  const sub = await this.ownerSubscriptionRepo.findByOwner(ownerId)

  if (newAmount === sub.current_amount) return  // nothing changed

  if (newAmount <= sub.max_amount) {
    // In-place update — no owner action needed
    await this.cashfreeService.updateSubscription(sub.cashfreeSubscriptionId, { amount: newAmount })
    await this.ownerSubscriptionRepo.update(sub.id, { current_amount: newAmount })
    await this.whatsappService.send(owner.phone, 'platform_plan_updated', {
      old_amount: sub.current_amount,
      new_amount: newAmount,
      slot_breakdown: this.buildSlotBreakdown(owner.slots),
      next_debit_date: sub.next_billing_date,
      effective: 'next billing cycle'  // no proration — see below
    })
  } else {
    // New amount exceeds the authorized ceiling → cancel + recreate
    await this.cashfreeService.cancelSubscription(sub.cashfreeSubscriptionId)
    const newMax = Math.ceil((newAmount * 1.5) / 100) * 100  // 1.5× buffer, rounded to ₹100
    const newSub = await this.cashfreeService.createSubscription({
      amount: newAmount,
      max_amount: newMax,
      cycle: 'monthly',
      start_date: sub.next_billing_date  // start from next cycle, not today
    })
    await this.ownerSubscriptionRepo.update(sub.id, {
      cashfree_subscription_id: newSub.id,
      current_amount: newAmount,
      max_amount: newMax,
      status: 'pending_auth'
    })
    // Owner must re-authorize once
    await this.whatsappService.send(owner.phone, 'platform_plan_reauth_required', {
      new_amount: newAmount,
      auth_link: newSub.auth_link,
      reason: 'Your portfolio grew beyond the authorized limit — one-time re-setup needed'
    })
  }
}
```

#### No Proration — New Amount Takes Effect Next Cycle

If an owner adds a shop on June 20th and the next billing date is July 5th:
- Shop management features are available immediately
- The extra ₹80/month is charged from July 5th, not prorated for June
- Owner gets the remainder of June's shop access free → feels like a benefit, not a gap

This eliminates billing complexity and never surprises the owner with a mid-cycle charge.

#### Initial Subscription Creation (max_amount formula)

```typescript
async createInitialSubscription(owner: Owner, amount: number): Promise<void> {
  const maxAmount = Math.ceil((amount * 1.5) / 100) * 100  // e.g. ₹1,200 → max ₹1,800

  const sub = await this.cashfreeService.createSubscription({
    amount,
    max_amount: maxAmount,
    cycle: 'monthly',
    start_date: addDays(new Date(), 1)  // first debit tomorrow after auth
  })

  await this.ownerSubscriptionRepo.create({
    owner_id: owner.id,
    cashfree_subscription_id: sub.id,
    status: 'pending_auth',
    current_amount: amount,
    max_amount: maxAmount
  })

  await this.whatsappService.send(owner.phone, 'platform_subscription_setup', {
    amount,
    auth_link: sub.auth_link
  })
}
```

#### Decision Matrix

| Scenario | Amount vs max_amount | Action | Owner experience |
|---|---|---|---|
| Add 1 shop (₹1,200 → ₹1,280) | Within buffer (max ₹1,800) | In-place update | WhatsApp only, no action needed |
| Add 5 more PG beds (₹1,200 → ₹1,500) | Within buffer (max ₹1,800) | In-place update | WhatsApp only, no action needed |
| Remove a property (₹1,200 → ₹900) | Always within max | In-place update (amount decreases) | WhatsApp with new lower amount |
| Add 20 more beds (₹1,200 → ₹2,400) | Exceeds buffer (max ₹1,800) | Cancel + recreate (new max ₹3,600) | WhatsApp with re-auth link |

---

## 4. Cashfree Auto-Pay Mandates for Tenants

### Value Proposition Shift

| Current (Phase 1) | With Mandates (Phase 2) |
|---|---|
| System reminds tenant to pay | System COLLECTS from tenant automatically |
| Tenant must click, open app, pay | Tenant does nothing after one-time setup |
| Owner celebrates when tenant pays | Owner only notices when it FAILS |
| "Automated reminders" | "Zero-touch rent collection" |

This changes the product from a workflow tool to an infrastructure layer.

### Technology: UPI AutoPay (Recommended Starting Point)

Cashfree Subscriptions API supports UPI AutoPay (NPCI/NACH infrastructure):
- Tenant authorizes via any UPI app (PhonePe, GPay, Paytm, BHIM)
- Max: ₹1,00,000/transaction (sufficient for residential rents)
- Setup time: < 30 seconds for tenant
- Money goes: Cashfree → Owner's bank (same as current payment link flow — no fund holding)

### Database Schema

```sql
-- Tenant mandate tracking
CREATE TABLE tenant_mandates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  cashfree_subscription_id VARCHAR(100) UNIQUE NOT NULL,
  mandate_type VARCHAR(20) DEFAULT 'upi_autopay'  -- 'upi_autopay', 'enach', 'emandate'
  status VARCHAR(20) NOT NULL                      -- 'pending', 'authorized', 'active', 'cancelled', 'failed'
  mandate_amount DECIMAL(10,2) NOT NULL,           -- amount mandate was created for
  max_amount DECIMAL(10,2),                        -- for variable mandates (Phase 3)
  authorized_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  owner_id UUID NOT NULL REFERENCES owners(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Add to tenants table:
ALTER TABLE tenants
  ADD COLUMN autopay_status VARCHAR(20) DEFAULT 'not_setup'  
    CHECK (autopay_status IN ('not_setup', 'pending', 'active', 'failed', 'cancelled'));
```

### BullMQ Integration Changes

**Existing flow (for tenants WITHOUT mandate):**
```
Daily cron 7AM → check reminder criteria → generate payment link → send WhatsApp
```

**New flow (for tenants WITH active mandate):**
```
Daily cron 7AM → check reminder criteria
  → if tenant.autopay_status = 'active':
      → skip payment link generation
      → skip reminder WhatsApp (Cashfree will auto-debit on due date)
      → log: "autopay_active - skipped reminder"
  → if tenant.autopay_status = 'pending':
      → send one-time nudge: "Set up auto-pay to never receive reminders again: [link]"
  → else (not_setup / failed / cancelled):
      → normal reminder flow (existing FR-17, FR-18, FR-21)
```

### Webhook Handlers to Add

```typescript
// New webhook event: mandate authorized
async handleSubscriptionAuthorized(event: CashfreeWebhookEvent) {
  const subscription = await this.subscriptionService.findByExternalId(event.data.subscription_id)
  await this.tenantMandateService.markAuthorized(subscription.tenantId)
  await this.tenantService.updateAutopayStatus(subscription.tenantId, 'active')
  // Send confirmation WhatsApp to tenant
  await this.whatsappService.send(tenant.phone, 'autopay_setup_confirmed', { amount: subscription.mandateAmount })
}

// New webhook event: auto-debit success
async handleSubscriptionChargeSuccess(event: CashfreeWebhookEvent) {
  // Same as PAYMENT_SUCCESS (existing FR-26)
  await this.rentService.markPaid(rentRecord.id, event.data.payment_amount)
  await this.whatsappService.send(tenant.phone, 'rent_receipt', { ... })
  await this.whatsappService.send(owner.phone, 'payment_received', { ... })
}

// New webhook event: auto-debit failed
async handleSubscriptionChargeFailed(event: CashfreeWebhookEvent) {
  await this.tenantMandateService.recordFailure(subscription.tenantId)
  // Immediately fall back to payment link
  const paymentLink = await this.paymentService.createPaymentLink(rentRecord)
  await this.whatsappService.send(tenant.phone, 'rent_reminder_autopay_failed', {
    payment_link: paymentLink,
    amount: rentRecord.rent_amount
  })
  // Notify owner
  await this.whatsappService.send(owner.phone, 'autopay_failed_owner', {
    tenant_name: tenant.name,
    amount: rentRecord.rent_amount
  })
}

// New webhook event: mandate cancelled
async handleSubscriptionCancelled(event: CashfreeWebhookEvent) {
  await this.tenantService.updateAutopayStatus(subscription.tenantId, 'cancelled')
  await this.whatsappService.send(owner.phone, 'autopay_cancelled_owner', { tenant_name: tenant.name })
}
```

### Mandate Setup Flow (From Dashboard)

```
Owner navigates to: Tenants → Arjun → "Set Up Auto-Pay"
  → System creates Cashfree subscription: amount = tenant.rent_amount, cycle = monthly, start = next_due_date
  → System gets auth_link from Cashfree
  → System sends WhatsApp to Arjun with auth_link
  → Tenant status → 'pending'

Arjun receives WhatsApp:
  → Opens link → Cashfree page → selects UPI app → authorizes
  → Cashfree fires SUBSCRIPTION_AUTHORIZED → status → 'active'

From next due date: Cashfree auto-debits monthly
```

### Move-Out Mandate Cancellation

```typescript
// Extend existing tenantService.recordMoveOut():
async recordMoveOut(tenantId: string, moveOutDate: Date) {
  // existing logic...
  
  // NEW: cancel active mandate if exists
  const mandate = await this.tenantMandateService.findActiveByTenant(tenantId)
  if (mandate) {
    await this.cashfreeService.cancelSubscription(mandate.cashfreeSubscriptionId)
    await this.tenantMandateService.markCancelled(mandate.id)
    await this.tenantService.updateAutopayStatus(tenantId, 'cancelled')
  }
}
```

### Rent Amount Change + Mandate Update

When owner updates a tenant's monthly rent:

```typescript
// When rent amount changes:
if (newRentAmount !== tenant.rent_amount) {
  const mandate = await this.tenantMandateService.findActiveByTenant(tenant.id)
  if (mandate) {
    // Cancel old mandate, create new one
    await this.cashfreeService.cancelSubscription(mandate.cashfreeSubscriptionId)
    await this.initiateNewMandate(tenant, newRentAmount)  // sends new auth link to tenant
    await this.whatsappService.send(owner.phone, 'mandate_update_required', {
      tenant_name: tenant.name,
      old_amount: tenant.rent_amount,
      new_amount: newRentAmount
    })
  }
}
```

### New WhatsApp Templates Required

| Template Name | Recipient | Trigger |
|---|---|---|
| `autopay_setup_request` | Tenant | Owner initiates mandate setup |
| `autopay_setup_confirmed` | Tenant | SUBSCRIPTION_AUTHORIZED received |
| `autopay_success_receipt` | Tenant | SUBSCRIPTION_CHARGE_SUCCESS |
| `autopay_failed_tenant` | Tenant | SUBSCRIPTION_CHARGE_FAILED (with fallback link) |
| `autopay_cancelled_tenant` | Tenant | Owner cancels or system cancels |
| `autopay_failed_owner` | Owner | SUBSCRIPTION_CHARGE_FAILED notification |
| `autopay_cancelled_owner` | Owner | Tenant cancels their mandate |

All templates require Meta pre-approval. Submit together in one batch (5–7 business days).

---

## 5. PRD Changes Required

The following sections of `prd-rent-master-2026-06-11/prd.md` need updating in the next PRD revision:

### Section 2.2 — Non-Users (Remove/Revise)
> ~~"Commercial/enterprise property managers — not targeted until Phase 4+"~~
> 
> **New:** "Commercial property managers targeting large chains (100+ shops, city-wide) are not a Phase 2 target. Small-to-medium commercial owners (1–10 shops in one building) are supported from Phase 2."

### Section 5 — Functional Requirements (Add)
See the FR-E1 through FR-M8 list in the brainstorming session document.

### New Section: Phase Roadmap
- Phase 1 (current): Rent collection automation for PG + residential
- Phase 2: Electricity billing + commercial properties + mandate auto-pay + plan calculator
- Phase 3: Variable mandates (rent + electricity combined), commercial agreements
- Phase 4: GST invoicing, analytics, API access

---

## 6. Epic Impact

The following Phase 2 epics should be created (to be broken into stories later):

| Epic | Estimated Stories | Complexity |
|---|---|---|
| Electricity Billing Module | 6–8 stories | Medium |
| Commercial Property Support | 8–10 stories | Medium |
| Plan Calculator + Cashfree Subscription for Platform Fees | 4–5 stories | Medium |
| Tenant Auto-Pay Mandates (UPI AutoPay) | 8–10 stories | High |

---

## 7. Revenue Impact of These Features

### Electricity Module (Free in All Paid Tiers)
- Increases stickiness → reduces churn → +15% estimated LTV improvement
- "One more reason to stay" — not a revenue driver directly but LTV impact is significant

### Commercial Slots (₹80/slot vs ₹60/slot for residential)
- Mixed portfolio owner: 10 PG beds + 3 shops = ₹600 + ₹240 = ₹840/month
- Without commercial: same owner pays only ₹600/month for PG
- **ARPU uplift: +₹240/month for existing customers who also own shops (free acquisition)**

### Mandate Auto-Pay
- Reduces churn: owners who use mandates have stronger lock-in
- Increases trust: "RentMaster guarantees collection" is a stronger product story → easier to sell at ₹1,199+
- May enable slightly higher pricing in future positioning

### Plan Calculator UI
- Reduces pricing page bounce rate: transparent math builds trust faster
- Faster conversion from trial to paid: owner knows exactly what they'll pay before signing up

---

*Reference: Brainstorming session: `_bmad-output/brainstorming/brainstorming-session-2026-06-15-1000.md`*  
*Feeds into: `_bmad-output/planning-artifacts/prds/prd-rent-master-2026-06-11/prd.md` Phase 2 update*
