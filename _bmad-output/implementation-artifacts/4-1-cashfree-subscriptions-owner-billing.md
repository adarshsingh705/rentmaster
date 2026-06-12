---
baseline_commit: 
story_key: 4-1-cashfree-subscriptions-owner-billing
status: ready-for-dev
created_at: 2026-06-13
---

# Story 4.1: Cashfree Subscriptions for Owner Billing

**Epic:** Phase 4 — Owner Subscription Management  
**Priority:** High  
**Timeline:** Week 11–12  
**Story Points:** 13  

---

## Story

RentMaster needs automated monthly billing for owner subscriptions using **Cashfree Subscriptions API**. Owners select a tier during signup (Tier 1: ₹300/mo, Tier 2: ₹600/mo, Tier 3: ₹1,300/mo), and their card is auto-debit monthly. Successful charges trigger email receipts. Owners can pause, upgrade, or cancel anytime via dashboard. Failed charges send notifications.

---

## Acceptance Criteria

1. **Plan Creation in Cashfree Dashboard**
   - Tier 1: ₹300/month subscription plan created in Cashfree dashboard
   - Tier 2: ₹600/month subscription plan created
   - Tier 3: ₹1,300/month subscription plan created
   - Plans are in PRODUCTION environment (not sandbox)

2. **Owner Plan Selection at Signup**
   - Owner can select tier during registration OR skip for free trial
   - Selection stored in `owners.plan` column
   - Free tier (14-day trial) is default if no selection

3. **Subscription Creation API**
   - POST `/api/owner/subscription/create` endpoint created
   - Validates owner has bank account verified (or payment card registered)
   - Calls Cashfree subscription.create() with plan_id mapped to tier
   - Stores Cashfree subscription_id in `owner_subscriptions` table
   - Returns subscription confirmation with next charge date

4. **Subscription Webhook Handler**
   - POST `/api/webhooks/cashfree` receives subscription events
   - Verifies Cashfree HMAC-SHA256 signature before processing
   - Handles `subscription.charged` event:
     - Logs charge in `subscription_charges` table
     - Sends receipt email via Resend (₹{amount} charged, next charge date)
   - Handles `subscription.failed` event:
     - Logs failure in `subscription_charges` table
     - Sends failure notification to owner email
     - Logs warning in notification_log
   - Handles `subscription.cancelled` event:
     - Updates `owner_subscriptions.status` = "cancelled"
     - Downgrades owner.plan = "free"

5. **Billing History Tables**
   - `owner_subscriptions` table created with: owner_id, cashfree_subscription_id, plan, amount, status, started_at, next_charge_at, cancelled_at
   - `subscription_charges` table created with: owner_id, subscription_id, amount, status, charge_date
   - Both tables have proper indexes on owner_id and subscription_id
   - RLS policies ensure owners only see their own billing data

6. **Subscription Management Dashboard**
   - GET `/api/owner/subscription/status` returns current subscription details
   - POST `/api/owner/subscription/pause` pauses billing (via Cashfree API)
   - POST `/api/owner/subscription/resume` resumes billing
   - POST `/api/owner/subscription/upgrade` upgrades tier (cancel old, create new)
   - POST `/api/owner/subscription/cancel` cancels subscription
   - Dashboard shows: current plan, amount, next charge date, billing history (last 12 months)

7. **Email Receipts**
   - Charge success email sent via Resend includes:
     - Receipt number (reference to charge ID)
     - Amount charged (₹{amount})
     - Plan name (Tier 1/2/3)
     - Next charge date
     - Link to view billing history
   - Failed payment email includes:
     - Failed amount
     - Reason (if provided by Cashfree)
     - Action needed (update payment method)

8. **Error Handling & Retry Logic**
   - Failed charges automatically retry 3 times (Cashfree default)
   - No retry logic needed in app (Cashfree handles)
   - Owner notification sent after 3rd failed attempt
   - Failed charges do NOT downgrade plan (only cancellation downgrades)

---

## Tasks & Subtasks

### Task 1: Set Up Cashfree Subscription Plans in Dashboard
- [ ] Access Cashfree production account
- [ ] Create subscription plan: Tier 1 (₹300/month)
- [ ] Create subscription plan: Tier 2 (₹600/month)
- [ ] Create subscription plan: Tier 3 (₹1,300/month)
- [ ] Note plan IDs (plan_tier1_monthly, plan_tier2_monthly, plan_tier3_monthly)
- [ ] Test plan creation with sandbox first before production

### Task 2: Create Database Schema for Billing
- [ ] Create `owner_subscriptions` table with all fields + indexes
- [ ] Create `subscription_charges` table with all fields + indexes
- [ ] Add RLS policies (owner can only see own subscriptions)
- [ ] Create migration script for schema
- [ ] Test: owner A cannot query owner B's subscriptions

### Task 3: Build Subscription Creation Endpoint
- [ ] Create POST `/api/owner/subscription/create` endpoint
- [ ] Validate owner exists + authenticated
- [ ] Validate tier parameter (starter|growth|pro maps to plan_id)
- [ ] Call Cashfree subscription.create() with owner details
- [ ] Handle Cashfree API errors gracefully
- [ ] Store subscription in `owner_subscriptions` table
- [ ] Update `owners.plan` = tier selected
- [ ] Return subscription confirmation JSON

### Task 4: Implement Cashfree Webhook Handler
- [ ] Create POST `/api/webhooks/cashfree` endpoint for subscription events
- [ ] Verify HMAC-SHA256 signature (use Cashfree webhook secret)
- [ ] Parse subscription.charged event:
  - [ ] Extract owner_id, subscription_id, amount, charge_date
  - [ ] Insert into `subscription_charges` (status=success)
  - [ ] Send receipt email via Resend
- [ ] Parse subscription.failed event:
  - [ ] Insert into `subscription_charges` (status=failed)
  - [ ] Send failure notification email
  - [ ] Log to notification_log
- [ ] Parse subscription.cancelled event:
  - [ ] Update owner_subscriptions.status = cancelled
  - [ ] Update owners.plan = free
- [ ] Return 200 OK immediately (async processing)
- [ ] Test: replay webhook, confirm idempotency (no duplicate charges)

### Task 5: Build Subscription Management API Endpoints
- [ ] GET `/api/owner/subscription/status` — return current plan + billing history
- [ ] POST `/api/owner/subscription/pause` — call Cashfree pause API
- [ ] POST `/api/owner/subscription/resume` — call Cashfree resume API
- [ ] POST `/api/owner/subscription/upgrade` — cancel old + create new subscription
- [ ] POST `/api/owner/subscription/cancel` — call Cashfree cancel API
- [ ] All endpoints validate owner authentication
- [ ] All endpoints return 200 + updated status on success
- [ ] Test: pause → charge is skipped, resume → charges resume

### Task 6: Build Subscription Dashboard UI
- [ ] Create `/dashboard/billing` page in Next.js
- [ ] Display current plan (Tier 1/2/3) with amount/month
- [ ] Display next charge date
- [ ] Display last 12 months of charges (table: date, amount, status)
- [ ] Add buttons: Pause, Upgrade, Downgrade, Cancel
- [ ] Handle loading/error states
- [ ] Real-time: reflect charge status updates via Supabase Realtime
- [ ] Test: upgrade Tier 1 → Tier 2, confirm new amount on next charge

### Task 7: Email Receipt Implementation
- [ ] Create Resend email template for charge success
- [ ] Create Resend email template for charge failure
- [ ] Send email after subscription.charged webhook processed
- [ ] Send email after subscription.failed webhook processed
- [ ] Include receipt details: amount, plan, next charge date, reference ID
- [ ] Test: send test charge, verify email received with correct data

### Task 8: Testing & Validation
- [ ] Unit tests: subscription creation API (valid/invalid tier, missing auth)
- [ ] Unit tests: webhook handler (valid signature, invalid signature, idempotency)
- [ ] Integration tests: full flow — owner signs up → selects tier → first charge succeeds → email sent
- [ ] Integration tests: upgrade flow — pause current → create new → charge at new amount
- [ ] Integration tests: failed charge flow — charge fails → email sent → owner notified
- [ ] End-to-end test: use Cashfree sandbox to simulate real charges
- [ ] Run full regression suite (ensure no regressions in Phase 1–3)

---

## Dev Notes

### Architecture Context
- **Cashfree SDK:** Use `cashfree-pg` npm package (same as Phase 3 rent collection)
- **Single Gateway:** Same Cashfree account + webhook endpoint handles both:
  - Tenant rent collection (Phase 3, `rent_records` table)
  - Owner subscriptions (Phase 4, `owner_subscriptions` table)
- **Webhook Routing:** POST `/api/webhooks/cashfree` identifies event type and routes:
  - `payment.*` events → tenant rent logic
  - `subscription.*` events → owner billing logic

### Key Implementation Details
1. **Subscription ID Mapping:** Store Cashfree subscription_id (not plan_id) in `owner_subscriptions.cashfree_subscription_id` for future pause/upgrade/cancel calls
2. **Idempotency:** Use charge date + amount as idempotency key to prevent duplicate processing if webhook is delivered twice
3. **Email Timing:** Send receipt email AFTER webhook is processed, not during (use async job if needed)
4. **Plan Upgrade Logic:** When upgrading, cancel old subscription via Cashfree API, then create new subscription immediately. Gap in billing is acceptable (owner will be charged on new plan's next cycle).
5. **Free Tier Default:** If owner doesn't select a plan during signup, set `owners.plan = 'free'` with 14-day trial expiry. After 14 days, block dashboard access until plan is selected.

### Previous Learnings (From Phase 3)
- Cashfree webhook signature verification is critical — always verify before processing
- Respond 200 OK immediately to prevent Cashfree retry storms
- Store raw webhook payload in DB for debugging
- Use `subscription_id` (not order_id) for subscription events
- Test in Cashfree sandbox before pushing to production

### Testing Strategy
- Use Cashfree sandbox environment for development
- Simulate webhook by curl POST to localhost during testing
- Use production Cashfree credentials only after full sandbox validation
- Test edge case: what if owner's card is declined? (Cashfree retries 3x, then notifies)

---

## Dev Agent Record

### Implementation Plan
*To be filled after development begins*

### Completion Notes
*To be filled after all tasks complete*

### Debug Log
*To be filled during development if issues encountered*

---

## File List

*Files created/modified during implementation:*
- `lib/cashfree-subscription.js` — Cashfree subscription service (create, pause, resume, cancel)
- `pages/api/owner/subscription/create.js` — Owner subscription creation endpoint
- `pages/api/owner/subscription/status.js` — Get subscription status
- `pages/api/owner/subscription/pause.js` — Pause subscription
- `pages/api/owner/subscription/resume.js` — Resume subscription
- `pages/api/owner/subscription/upgrade.js` — Upgrade tier
- `pages/api/owner/subscription/cancel.js` — Cancel subscription
- `pages/api/webhooks/cashfree.js` — Enhanced webhook handler for subscription events
- `app/dashboard/billing/page.tsx` — Billing dashboard UI
- `migrations/add_billing_tables.sql` — Schema migration for owner_subscriptions + subscription_charges
- `emails/subscription-receipt.tsx` — Resend email template (charge success)
- `emails/subscription-failed.tsx` — Resend email template (charge failed)
- `__tests__/api/owner-subscription.test.js` — API endpoint tests
- `__tests__/webhooks/cashfree-subscription.test.js` — Webhook handler tests

---

## Change Log

*Document changes as implementation progresses*

---

**Status:** ready-for-dev  
**Assigned to:** Developer  
**Created:** 2026-06-13  
**Updated:** 2026-06-13
