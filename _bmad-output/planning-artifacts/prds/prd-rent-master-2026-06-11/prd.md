---
title: RentMaster Phase 1 — Automated Rent Collection, Billing & Payment Engine
status: draft
created: 2026-06-11
updated: 2026-06-15
---

# PRD: RentMaster Phase 1 — Automated Rent Collection, Billing & Payment Engine

## 0. Document Purpose

This PRD defines Phase 1 of the RentMaster platform — the fully automated rent collection, electricity billing, and payment engine for Indian PG, residential, and small commercial property owners. It is written for the engineering team (solo developer Adarsh), downstream UX/architecture workflows, and sprint planning.

Phase 0 (manual UPI + basic CRUD) is assumed complete or parallel. Phase 1 adds: a secure NestJS backend with JWT authentication (email + password), Cashfree payment automation (Beneficiary registration, payment link generation, webhook auto-mark-paid), BullMQ persistent reminder engine, Supabase Realtime dashboard, electricity billing module, commercial property support (shops/offices), tenant UPI AutoPay mandates, and owner subscription billing with a build-your-plan calculator.

**Updated 2026-06-15:** Features originally scoped to Phase 2/4 — electricity billing, commercial properties, tenant auto-pay mandates, and owner subscription billing — have been pulled into Phase 1. The full feature set is designed around a single launch.

**Inputs:** [Technical Architecture Research](../../research/technical-rent-master-platform-architecture-research-2026-06-11.md) · [Feature Expansion Planning](../../feature-expansion-electricity-commercial-autopay.md) · [Pricing Plan](../../pricing-plan.md). Technology decisions are locked in `.decision-log.md` and are not re-litigated here. This PRD covers *what* the system must do; implementation detail lives in the research document and the forthcoming architecture doc.

---

## 1. Vision

RentMaster Phase 1 transforms rent and electricity collection from monthly WhatsApp-and-follow-up chores into a fully automated engine. An owner registers once, connects their bank account, adds their properties and tenants — and from that moment the system takes over: generating payment links, sending WhatsApp reminders at exactly the right time, marking rents paid the instant money arrives via Cashfree, and showing the owner a live dashboard that updates in real time.

The core promise: **an Indian PG, residential, or small commercial property owner should never have to manually chase a tenant for rent or electricity.** Every reminder, every link, every receipt is automatic. Tenants who set up UPI AutoPay pay without being reminded at all. The owner's only job is to glance at the dashboard and act on genuine non-payments.

RentMaster supports three property types at launch:
- **PG / Hostel** — bed-based, multiple tenants per room
- **Residential** — room-based flats and houses, one tenant per room
- **Commercial** — shops, offices, and cabins, with CAM charges and escalation tracking

All three share the same Cashfree + WhatsApp automation stack. Billing is per-slot (1 bed = 1 slot, 1 room = 1 slot, 1 shop = 1 slot at a commercial premium). Owners pay via a Cashfree UPI AutoPay mandate on their own account — RentMaster uses the same collection mechanism it provides to its customers.

Phase 1 is backend-heavy by design. The NestJS API, JWT security layer, BullMQ scheduler, Cashfree integration, Supabase RLS multi-tenancy, and WhatsApp BSP form the foundation on which every future phase — mobile app, digital agreements, and analytics — is built.

---

## 2. Target User

### 2.1 Jobs To Be Done

- **Collect rent on time** without manually texting every tenant at the start of each month
- **Collect electricity bills** without reading meters, calculating units, and texting amounts manually
- **Know instantly when each tenant has paid** without checking bank statements
- **Send professional reminders** that neither embarrass the owner nor irritate the tenant
- **Prove payment to tenants** with a WhatsApp receipt they can reference for disputes
- **Onboard bank account once** and never think about payout routing again
- **Trust the system will not miss reminders** — even if the server restarts overnight
- **Manage shops and offices** with the same tool as PG/residential, not a separate one
- **Set up auto-pay once per tenant** and have rent debited automatically every month with no chasing
- **Pay the platform subscription automatically** — no manual renewal, no missed payment

### 2.2 Non-Users (Phase 1)

- **Tenants** — interact only via WhatsApp links and the Cashfree payment page; no platform login
- **Large commercial/enterprise property managers** — chains with 50+ shops, city-wide portfolios, or revenue-share/mall models are not targeted; small-to-medium commercial owners (1–10 shops in one building) are fully supported
- **Owners with fewer than 3 beds/rooms** — economics of bank onboarding don't justify the setup effort

### 2.3 Key User Journeys

**UJ-1. Priya registers, connects her bank, and is ready for automated collection.**
- **Persona + context:** Priya runs a 12-bed PG in Koramangala. She currently sends WhatsApp reminders manually on the 28th of each month and reconciles payments by checking her GPay history.
- **Entry state:** Not authenticated. Lands on the RentMaster registration page.
- **Path:**
  1. Enters email + password → account created; Access Token and Refresh Token issued; redirected to dashboard onboarding flow.
  2. Completes owner profile (full name, optional GSTIN).
  3. Adds Property: "Priya's PG Koramangala" → adds 4 rooms → adds 8 tenants with names, phone numbers, move-in dates, and monthly rent amounts.
  4. Navigates to Payment Settings → enters HDFC bank account number + IFSC + account holder name → submits.
  5. System registers Cashfree Beneficiary, initiates ₹1 penny-drop. Dashboard shows "Bank Verification: Pending."
  6. Within 10 minutes, penny-drop completes → dashboard updates to "Bank Verified ✓."
- **Climax:** Priya's bank is connected and verified. On the next 1st of the month, the BullMQ cron auto-generates rent records for all 8 tenants and the reminder engine begins.
- **Resolution:** Priya is on the dashboard with bank verified and tenants added. No further action needed until a tenant genuinely refuses to pay.
- **Edge case:** Penny-drop fails (wrong account number) → dashboard shows "Verification Failed — please re-enter your bank details" with a re-submit option; existing Beneficiary record is superseded.

---

**UJ-2. Arjun pays rent via a WhatsApp reminder link — Priya's dashboard updates in real time.**
- **Persona + context:** Arjun rents Room 3 at Priya's PG, ₹8,500/month, due on the 1st. Today is the 29th — 3 days before due date. He hasn't paid yet.
- **Entry state:** BullMQ daily cron has run at 7 AM IST. Arjun's rent record is `pending`.
- **Path:**
  1. BullMQ worker processes the reminder job → PaymentService.createPaymentLink() called → payment link stored in the rent record.
  2. Interakt BSP sends WhatsApp: *"Hi Arjun, your rent of ₹8,500 for July 2026 is due on 1st July. Pay here: [link]."*
  3. Arjun taps the link → Cashfree payment page → pays via GPay UPI in 30 seconds.
  4. Cashfree fires PAYMENT_SUCCESS webhook → NestJS handler verifies HMAC signature → confirms idempotency → updates rent record to `paid`.
  5. Supabase Realtime broadcasts the change → Priya's dashboard tile for Arjun flips from "Pending" to "Paid ✓" without a refresh.
  6. Arjun receives WhatsApp receipt: *"Payment of ₹8,500 received for July 2026. Thank you!"*
- **Climax:** Priya sees the live update and knows Arjun has paid. Arjun has a receipt on WhatsApp.
- **Resolution:** Rent record `status = paid`, `reminder_count` updated, `notification_log` has `sent` + `delivered` entries for both messages.
- **Edge case:** Webhook arrives twice for the same payment → idempotency check finds `status = paid` → second event silently discarded; no duplicate receipt sent.

---

**UJ-3. Priya reviews her monthly collection on the 5th and triggers a targeted overdue reminder.**
- **Persona + context:** It's the 5th of the month. 10 of 12 tenants have paid; 2 are still pending.
- **Entry state:** Authenticated. Dashboard open on current month.
- **Path:**
  1. Dashboard summary: *"10 / 12 paid · ₹85,000 / ₹1,02,000 collected · 2 pending."*
  2. Priya clicks the "Pending" filter → sees Ravi (Room 7, ₹9,500) and Sneha (Room 11, ₹8,000), each showing "Last reminded: 1st July."
  3. She clicks "Send Overdue Reminder" on Ravi's row.
  4. System queues an overdue WhatsApp job immediately (bypasses the daily cron).
  5. Ravi receives: *"Hi Ravi, your rent of ₹9,500 for July 2026 is 4 days overdue. Please pay: [link]."*
- **Climax:** Priya sent a targeted overdue reminder in 2 clicks.
- **Resolution:** Ravi's `reminder_count` incremented, `last_reminder_at` updated, `notification_log` entry created.

---

## 3. Glossary

- **Owner** — A registered RentMaster user (landlord or PG operator). One Owner has one account; their data is fully isolated from other Owners at the database level via Supabase RLS and at the application level via TenantGuard.
- **Tenant** — A renter managed by an Owner. Tenants do not have platform accounts; they interact only via WhatsApp links and the Cashfree payment page.
- **Property** — A physical building or PG managed by an Owner (e.g., "Priya's PG Koramangala"). One Owner can have multiple Properties. Each Property has a `property_type`: `pg`, `residential`, or `commercial`.
- **Room** — A lettable unit within a Property. For PG properties this holds multiple beds; for residential properties this holds one tenant; for commercial properties this represents one shop or office unit.
- **Slot** — The billing unit. 1 PG bed = 1 slot. 1 residential room = 1 slot. 1 commercial unit = 1 slot (billed at a premium rate). All of an Owner's slots across all properties are counted together to determine their monthly subscription cost.
- **Rent Record** — A single monthly billing entry for one Tenant (`rent_records` table). One Rent Record per Tenant per calendar month. Includes `rent_amount`, `electricity_amount`, and `cam_amount` (for commercial). Statuses: `pending` → `paid` | `partial` | `waived`.
- **Payment Link** — A Cashfree-generated URL stored in a Rent Record and sent to a Tenant via WhatsApp. The link amount = rent + electricity + CAM for that month.
- **Beneficiary** — An Owner's registered bank account in Cashfree's system. Referenced by `pg_beneficiary_id` in `owner_payment_settings`. Must exist and be bank-verified before Payment Links can be generated for that Owner's Tenants.
- **Penny-Drop** — A ₹1 test transfer Cashfree sends to the Owner's bank account to verify it is real and active. `bank_verified` is set `true` only after penny-drop confirmation.
- **Bank Change OTP** — A 6-digit one-time password sent to the Owner's registered email address, required to authorise any update to already-saved bank account details. Expires in 10 minutes. Single-use.
- **Reminder Job** — A BullMQ job that, when processed, generates a Payment Link (if absent) and sends a WhatsApp reminder to the Tenant. Skipped for tenants with an active UPI AutoPay mandate.
- **PaymentService** — The gateway-agnostic abstraction layer. All payment and subscription operations go through PaymentService; no code outside it references Cashfree directly.
- **TenantGuard** — NestJS guard that extracts and validates `owner_id` from the JWT and injects it into the request context. Every protected route reads `owner_id` from the guard — never from the request body.
- **Access Token** — Short-lived JWT (15-minute expiry) used to authenticate API requests.
- **Refresh Token** — Long-lived token (7-day expiry) stored hashed server-side. Rotated on every use; old token invalidated immediately.
- **Webhook** — An HTTP POST from Cashfree to a webhook endpoint on payment or subscription events. Must pass HMAC-SHA256 verification before any processing.
- **Electricity Config** — Per-property settings for electricity billing mode: `sub_meter` (owner enters monthly unit readings), `flat_rate` (fixed amount per room/bed), or `shared_split` (total EB bill divided equally among active tenants).
- **Meter Reading** — A monthly entry recorded by the Owner for a room's electricity meter: opening reading, closing reading. System calculates units consumed and electricity amount automatically.
- **CAM Charge** — Common Area Maintenance charge, applicable to commercial units only. A fixed monthly amount (security, housekeeping, corridor electricity) billed alongside rent.
- **UPI AutoPay Mandate** — A Cashfree Subscriptions API authorization that allows Cashfree to auto-debit a fixed or variable amount from a Tenant's UPI account on the rent due date each month. Once authorized, the Tenant pays rent without any manual action. Applicable to both tenant rent collection and Owner's RentMaster subscription.
- **Owner Subscription** — The Owner's monthly payment to RentMaster for platform access, billed per-slot via a Cashfree UPI AutoPay mandate. One subscription per Owner. Amount is updated in-place when slot count changes, subject to the `max_amount` buffer ceiling set at mandate creation.

---

## 4. Features

### 4.1 Email + Password Authentication & JWT Session Management

**Description:** Owners register and log in via two methods: email + password, and Google OAuth ("Sign in with Google"). On successful login the system issues an Access Token (15-minute expiry) and a Refresh Token (7-day expiry, rotated on use). NestJS JwtGuard validates the Access Token on every protected route. NestJS TenantGuard injects `owner_id` into each request context so no route can act on another Owner's data — even with a valid token. Both auth methods produce identical JWT shapes; downstream routes have no awareness of which method was used. Phone OTP login is deferred to a future phase.

[ASSUMPTION: Supabase Auth issues the JWT for both email+password and Google OAuth flows; NestJS JwtGuard validates via Supabase's `getUser()` rather than a standalone NestJS secret. Supabase natively supports both methods with no additional infrastructure. If a fully custom NestJS issuer is preferred, FR-1 through FR-5 need revision — see OQ-4.]

Realizes UJ-1.

**Functional Requirements:**

#### FR-1: Owner Registration
An unauthenticated user can create a new Owner account using email and password.

**Consequences (testable):**
- POST `/api/auth/register` creates an Owner record linked to the Supabase auth user ID.
- Returns `{ accessToken, refreshToken, expiresIn }` on success (HTTP 201).
- Duplicate email returns HTTP 409.
- Password shorter than 8 characters returns HTTP 422 with a field-level error message.
- Email format validation; invalid format returns HTTP 422.

**Out of Scope:** Apple / other social logins, phone/OTP registration.

---

#### FR-2: Owner Login
An Owner can log in with email and password to receive a new Access Token and Refresh Token pair.

**Consequences (testable):**
- POST `/api/auth/login` with correct credentials returns HTTP 200 with `{ accessToken, refreshToken, expiresIn: 900 }`.
- Incorrect password or unknown email returns HTTP 401. No distinction between the two (prevents email enumeration).
- Access Token is valid for exactly 15 minutes from issuance.

---

#### FR-2A: Google OAuth Login
An Owner can sign in or create an account using their Google account ("Sign in with Google").

**Consequences (testable):**
- "Sign in with Google" button on both the login and registration pages initiates the Google OAuth flow via Supabase Auth (Google as OAuth provider, configured in Supabase dashboard).
- On first Google sign-in for an email address not yet in the system, a new Owner record is created automatically — equivalent to the FR-1 registration flow, no separate registration step required.
- On subsequent Google sign-ins, the existing Owner account associated with that Google email is authenticated.
- Returns `{ accessToken, refreshToken, expiresIn }` — identical shape to FR-2 (password login).
- If the same email already exists as a password account, Google sign-in authenticates the same Owner account (email-based account linking handled by Supabase Auth). [ASSUMPTION: Supabase Auth merges accounts by email automatically when both Google OAuth and password auth share the same email address — see A-6.]
- NestJS TenantGuard treats Google-authenticated sessions identically to password-authenticated sessions; `owner_id` is injected the same way regardless of auth method.

**Out of Scope:** Linking or unlinking Google from an existing password account via the settings UI (Phase 2). Apple or other social providers (deferred).

---

#### FR-3: Token Refresh
An Owner holding a valid Refresh Token can obtain a new Access Token without re-entering credentials.

**Consequences (testable):**
- POST `/api/auth/refresh` with a valid Refresh Token returns a new Access Token and a new (rotated) Refresh Token.
- The old Refresh Token is invalidated immediately upon rotation.
- Expired or already-used Refresh Token returns HTTP 401, forcing re-login.

---

#### FR-4: Logout
An authenticated Owner can log out, immediately invalidating their current Refresh Token.

**Consequences (testable):**
- POST `/api/auth/logout` with a valid Access Token invalidates the associated Refresh Token.
- Subsequent use of the same Refresh Token returns HTTP 401.
- Logout does not require the Refresh Token in the request body — it invalidates the session bound to the Access Token's subject claim.

---

#### FR-5: Route Protection — JwtGuard + TenantGuard
Every API route under `/api/owner/*` requires a valid Access Token. TenantGuard injects `owner_id` into every request.

**Consequences (testable):**
- Request missing the `Authorization: Bearer <token>` header returns HTTP 401.
- Request with an expired Access Token returns HTTP 401.
- Request with a valid token has `owner_id` available in the route handler via the guard — no handler reads `owner_id` from the request body.
- A valid token from Owner A cannot retrieve Owner B's data: RLS enforces this at DB level; TenantGuard provides a secondary application-level check.

---

#### FR-6: Password Reset
An Owner can request a password reset email sent to their registered address.

**Consequences (testable):**
- POST `/api/auth/forgot-password` triggers a reset email via Resend (or Supabase built-in email).
- Reset link expires after 1 hour.
- Expired or already-used reset link returns HTTP 410.
- Requesting a reset for an unknown email returns HTTP 200 (prevents email enumeration) — no email is sent.

---

### 4.2 Owner Profile & Bank Account Onboarding

**Description:** After registration, Owners complete their profile and register a bank account with Cashfree. The initial registration initiates a ₹1 penny-drop to verify the account is real. Any subsequent change to saved bank details requires a Bank Change OTP sent to the Owner's registered email before the update is applied — this prevents unauthorised redirection of rent payouts. Until bank verification is complete, the system sends WhatsApp reminders without a Payment Link (fallback text). Bank account numbers are AES-256 encrypted at the application layer before DB storage and are never returned in API responses. Realizes UJ-1.

**Functional Requirements:**
  
#### FR-7: Owner Profile Completion
An authenticated Owner can set and update their full name and optional GSTIN.


**Consequences (testable):**
- PATCH `/api/owner/profile` updates the `owners` row scoped to the requesting `owner_id`.
- GSTIN validated as a 15-character alphanumeric string if provided; invalid format returns HTTP 422.
- Response includes the updated profile fields; never includes bank account details.

---

#### FR-8: Bank Account Registration & Beneficiary Creation
An authenticated Owner with no previously saved bank account can submit bank account details (account number, IFSC, account holder name) to register a Cashfree Beneficiary and initiate penny-drop verification. No OTP is required for first-time registration.

**Consequences (testable):**
- POST `/api/owner/payment-settings/bank` is accepted without an OTP step when `owner_payment_settings` has no existing bank record for the Owner.
- PaymentService.registerBeneficiary() called with the submitted details; returned `beneficiaryId` stored in `owner_payment_settings.pg_beneficiary_id`.
- Bank account number AES-256 encrypted before writing to `owner_payment_settings.bank_account_number`; plaintext never persists.
- PaymentService.verifyBankAccount() called immediately after beneficiary creation to initiate the penny-drop.
- `bank_verified` remains `false` until the Cashfree penny-drop confirmation webhook arrives.
- Cashfree API errors surface as a user-readable message (e.g., "Invalid IFSC code") — raw gateway errors are not forwarded to the client.

---

#### FR-9: Bank Verification Status
An authenticated Owner can view the current state of their bank account registration.

**Consequences (testable):**
- GET `/api/owner/payment-settings` returns `{ bankVerified, bankHolderName, ifsc, lastFourDigits, payoutMode }`.
- Full account number is never included in the response.
- `bankVerified: true` only after Cashfree penny-drop webhook has confirmed success (see FR-23-equivalent in payment settings flow).

---

#### FR-10: Bank Account Re-Submission After Failure
An Owner whose penny-drop failed can re-submit corrected bank details. This requires Bank Change OTP verification because bank details are already saved (even if unverified).

**Consequences (testable):**
- Re-submission follows the FR-10A two-step OTP flow (request OTP → verify OTP → apply update).
- After OTP verification, re-submission overwrites the existing `owner_payment_settings` row and creates a new Cashfree Beneficiary.
- Previous failed `pg_beneficiary_id` is superseded.
- `bank_verified` is reset to `false` and a new penny-drop is initiated.

---

#### FR-10A: Bank Change OTP — Request & Verify
An authenticated Owner who already has saved bank details (verified or unverified) must complete a Bank Change OTP flow before any update to bank account details is applied.

**Step 1 — Request OTP:**

**Consequences (testable):**
- POST `/api/owner/payment-settings/bank/request-otp` sends a 6-digit OTP to the Owner's registered email address via Resend.
- Returns HTTP 202 with `{ message: "OTP sent", expiresIn: 600 }`.
- OTP is stored hashed (SHA-256) in the session/DB with a 10-minute TTL — plaintext OTP never persisted.
- Re-requesting OTP before the previous one expires invalidates the old OTP and sends a fresh one (prevents replay of the previous code).
- Rate limit: maximum 5 OTP requests per Owner per hour; exceeding returns HTTP 429 with `retryAfter` timestamp.

**Step 2 — Verify OTP & Apply Update:**

**Consequences (testable):**
- POST `/api/owner/payment-settings/bank` with `{ otp, accountNumber, ifsc, holderName }` — OTP verified first before any bank update logic runs.
- Valid OTP + correct bank details: update applied (FR-8 flow), OTP immediately invalidated.
- Invalid OTP: returns HTTP 401 with remaining attempt count. After 3 consecutive failed attempts within the OTP's validity window, the OTP is invalidated and the Owner must request a new one.
- Expired OTP (past 10 minutes): returns HTTP 410; Owner must request a new OTP.
- OTP is single-use — a correct OTP cannot be submitted a second time.

**Out of Scope:** Phone SMS OTP for bank changes (email OTP only in Phase 1).

---

### 4.3 Property, Room & Tenant Management

**Description:** Owners manage the hierarchy: Property → Room → Tenant. Each Tenant has a monthly rent amount, a rent due day (default: 1st), and a move-in date. Active Tenants with no move-out date are included in the monthly rent record generation cron. Realizes UJ-1.

**Functional Requirements:**

#### FR-11: Property CRUD
An authenticated Owner can create, read, update, and soft-delete their Properties.

**Consequences (testable):**
- All Property records scoped to `owner_id` (RLS + TenantGuard enforced).
- Soft-delete sets `is_active = false`; excludes the Property from all cron and reminder operations.
- A Property with at least one active Tenant cannot be soft-deleted; returns HTTP 409 with a count of blocking Tenants.
- Property `city` field is required; `type` defaults to `pg`.

---

#### FR-12: Room CRUD
An authenticated Owner can create, read, update, and soft-delete Rooms within their Properties.

**Consequences (testable):**
- Room `owner_id` is set from TenantGuard context (never from the request body).
- A Room with an active Tenant cannot be soft-deleted; returns HTTP 409.
- `monthly_rent` on a Room is a default; per-Tenant rent can be overridden at Tenant creation.

---

#### FR-13: Tenant Onboarding
An authenticated Owner can add a Tenant to a Room with: full name, phone (Indian format), move-in date, monthly rent, rent due day (default: 1), optional deposit amount, optional email, optional Aadhaar last-4 digits.

**Consequences (testable):**
- Tenant `owner_id` set from TenantGuard context.
- Phone number validated and stored in E.164 format (`+91XXXXXXXXXX`).
- Aadhaar last-4 stored as text only; full document upload is a separate operation via Supabase Storage.
- Creating a Tenant in the current calendar month immediately creates a Rent Record for the current month with `amount_due = monthly_rent`. [ASSUMPTION: full month amount; no proration in Phase 1 — see OQ-5.]

---

#### FR-14: Tenant Move-Out
An authenticated Owner can record a move-out date for a Tenant, marking them inactive.

**Consequences (testable):**
- `tenants.move_out_date` set to the provided date; `is_active = false`.
- Monthly cron no longer generates Rent Records for this Tenant from the following month.
- Existing pending Rent Records for the current month are not auto-waived; the Owner must manually waive them if applicable.

---

#### FR-15: Manual Rent Record Waive
An authenticated Owner can mark a pending Rent Record as `waived` with an optional note.

**Consequences (testable):**
- PATCH `/api/owner/rent-records/:id` with `{ status: "waived", notes: "..." }` updates the record.
- Only `pending` or `partial` records can be waived; attempting to waive a `paid` record returns HTTP 409.
- Waived records are excluded from collection totals on the dashboard but remain visible in history.

---

### 4.3B Commercial Property Support

**Description:** Owners can create Properties of type `commercial` (shops, offices, cabins). Commercial units support CAM charges, security deposit tracking, rent escalation clauses, and agreement expiry alerts. The same Cashfree + WhatsApp automation stack is used. Commercial units are billed at a premium slot rate (₹80/slot Tier 1 vs ₹60 for residential/PG). Terminology in the UI adapts: "Room" → "Unit/Shop/Office", "Tenant" → "Business Tenant".

**Functional Requirements:**

#### FR-C1: Commercial Property Type
An authenticated Owner can create a Property with `property_type = commercial`.

**Consequences (testable):**
- POST `/api/owner/properties` accepts `type: "commercial"` alongside existing `pg` and `residential` values.
- Commercial properties show adapted terminology throughout the app.
- `commercial` properties are excluded from bed-count logic; each room/unit counts as exactly 1 slot.
- Slot billing for commercial units uses the commercial rate schedule (see FR-S1).

---

#### FR-C2: CAM Charge per Commercial Unit
An authenticated Owner can set a monthly CAM charge for a commercial unit.

**Consequences (testable):**
- Commercial unit config accepts `cam_charge DECIMAL` (default 0).
- When non-zero, `cam_charge` is added to the Rent Record's `cam_amount` on monthly cron run.
- WhatsApp reminder shows: Rent ₹X + CAM ₹Y = Total ₹Z.
- Payment Link amount = `rent_amount + cam_amount` (electricity is separate unless also configured).

---

#### FR-C3: Security Deposit Tracking for Commercial Tenants
An authenticated Owner can record the security deposit received from a commercial Business Tenant.

**Consequences (testable):**
- POST `/api/owner/tenants` (commercial) accepts `security_deposit_paid DECIMAL`.
- Deposit amount visible on tenant detail page.
- Not included in monthly billing — informational record only.

---

#### FR-C4: Rent Escalation Tracker
An authenticated Owner can set an escalation percentage and next escalation date on a commercial unit.

**Consequences (testable):**
- Commercial config accepts `escalation_pct DECIMAL` (e.g., 5.00 = 5%) and `next_escalation_date DATE`.
- BullMQ daily cron scans for commercial units where `next_escalation_date` is within 60 days → queues escalation alert job.
- WhatsApp to owner (60 days before): rent current amount, escalation %, calculated new rent, due date.
- WhatsApp to owner again at 30 days before escalation date.
- Owner must manually update the tenant's `monthly_rent` after the escalation; the system reminds but does not auto-update rent.

---

#### FR-C5: Agreement Expiry Alerts
The system alerts the Owner when a commercial agreement is expiring within 60 or 30 days.

**Consequences (testable):**
- Commercial config accepts `agreement_start_date DATE` and `agreement_end_date DATE`.
- BullMQ daily cron scans for agreements where `agreement_end_date` is within 60 days → queues expiry alert job.
- WhatsApp to owner at 60 days and again at 30 days: unit name, tenant name, expiry date.
- If `agreement_end_date` is null, no alert is sent (optional field).

---

### 4.4 Automated Monthly Rent Record Generation

**Description:** On the 1st of every month at 00:00 IST, a BullMQ cron job auto-creates one Rent Record per active Tenant. This operation is idempotent — re-running it never creates duplicates. For Tenants added mid-month, the Rent Record is created at Tenant creation (FR-13), not by the cron.

**Functional Requirements:**

#### FR-16: Monthly Rent Record Cron
The system automatically creates one `rent_records` row per active Tenant on the 1st of each month.

**Consequences (testable):**
- Cron fires at 00:00 IST on the 1st (18:30 UTC on the last day of the previous month).
- `amount_due` copied from `tenants.monthly_rent` at cron run time.
- UPSERT with `ON CONFLICT (tenant_id, month) DO NOTHING` — re-running the cron never creates duplicate records.
- The cron job is persisted in BullMQ (Upstash Redis); if the Render.com worker restarts mid-run, the job resumes and already-processed Tenants are not double-processed (idempotency guaranteed by the UPSERT).
- Tenants with `is_active = false` are excluded.

---

### 4.5 BullMQ Rent Reminder Engine

**Description:** A daily cron job at 07:00 IST scans all pending Rent Records and queues WhatsApp reminder jobs for Tenants due in 3 days, due today, and 5+ days overdue. Each job generates a Cashfree Payment Link (if absent) and sends a WhatsApp message via Interakt. All jobs are persisted in Upstash Redis and survive worker restarts. Realizes UJ-2, UJ-3.

**Functional Requirements:**

#### FR-17: Daily Reminder Cron Scan
The system scans `rent_records` at 07:00 IST daily and queues Reminder Jobs for all records meeting reminder criteria.

**Consequences (testable):**
- Queues a "3-days-before" Reminder Job for records where the due date is exactly 3 days away and `status = pending`.
- Queues a "due-today" Reminder Job for records due today with `status = pending`.
- Queues an "overdue" Reminder Job for records with a due date 5+ days past and `status = pending`.
- Skips any record where `last_reminder_at` is within the last 20 hours (deduplication check before enqueueing).
- All Reminder Jobs enqueued with: 3 max retry attempts, exponential backoff (1 min → 2 min → 4 min).
- Cron job itself is a BullMQ repeatable job (not `node-cron` in isolation) — persisted in Redis.

---

#### FR-18: Payment Link Generation at Reminder Time
When a Reminder Job is processed, if the Rent Record has no Payment Link, the system generates one via PaymentService before sending the WhatsApp message.

**Consequences (testable):**
- PaymentService.createPaymentLink() called only if `rent_records.payment_link IS NULL`.
- `pg_order_id` and `payment_link` stored in the Rent Record before the WhatsApp send attempt.
- If the Owner has no verified bank account (`bank_verified = false`), the Payment Link generation step is skipped; the WhatsApp message is still sent using a fallback template that omits the link. [ASSUMPTION: fallback template text is "Contact your landlord to arrange payment" — confirm with OQ-2.]
- Job failure (Cashfree API error or WhatsApp send error) triggers BullMQ retry. After 3 failed attempts, the job moves to the dead-letter queue and is logged to `notification_log` with `status = failed`.

---

#### FR-19: Manual Overdue Reminder Trigger
An authenticated Owner can manually trigger an overdue reminder for a specific pending Rent Record from the dashboard, bypassing the daily cron schedule. Realizes UJ-3.

**Consequences (testable):**
- POST `/api/owner/rent-records/:id/remind` enqueues a Reminder Job immediately.
- Returns HTTP 202 (Accepted) — does not block waiting for WhatsApp delivery.
- Subject to the 20-hour deduplication check: if `last_reminder_at` is within 20 hours, returns HTTP 429 with a `retryAfter` timestamp.
- Works only for Rent Records with `status = pending` or `partial`; attempting on a `paid` record returns HTTP 409.

---

### 4.6 WhatsApp Notification Delivery

**Description:** All Tenant-facing communications use pre-approved WhatsApp templates sent via the Interakt BSP. Every send attempt — success or failure — is logged to `notification_log`. Delivery status (sent → delivered → read → failed) is updated via the WhatsApp webhook. Realizes UJ-2, UJ-3.

**Functional Requirements:**

#### FR-20: WhatsApp Reminder Send
When a Reminder Job is processed, the system sends the appropriate pre-approved WhatsApp template to the Tenant's registered phone number.

**Consequences (testable):**
- Template selection by job type: `rent_reminder_3days` (3-days-before), `rent_reminder_due_today` (due today), `rent_overdue` (5+ days overdue).
- Template variables populated: Tenant name, amount due (₹ formatted), month (e.g., "July 2026"), Payment Link.
- Every send attempt creates a `notification_log` row: `channel = whatsapp`, `type = <template_name>`, `status = sent | failed`, `message_id = <Interakt message ID>`.
- Interakt API error marks the Reminder Job as failed and triggers BullMQ retry.

---

#### FR-21: WhatsApp Payment Receipt
After PAYMENT_SUCCESS webhook processing, the system sends a WhatsApp receipt to the Tenant.

**Consequences (testable):**
- Template: `rent_receipt`.
- Variables: Tenant name, amount paid (₹ formatted), month, payment date (DD/MM/YYYY).
- Sent asynchronously — after the webhook handler has returned HTTP 200.
- Receipt send failure is logged to `notification_log` with `status = failed` but does NOT roll back the Rent Record status update.

---

#### FR-22: Owner Payment Notification
After PAYMENT_SUCCESS webhook processing, the system sends a WhatsApp notification to the Owner confirming which Tenant paid and the amount.

**Consequences (testable):**
- Template: `owner_payment_received`. [ASSUMPTION: this template needs Meta approval — see OQ-2.]
- Variables: Tenant name, Room number, amount paid, month.
- Sent asynchronously after webhook processing; failure does not affect Rent Record state.

---

#### FR-23: WhatsApp Delivery Status Tracking
The system processes WhatsApp delivery status webhooks from Meta (via Interakt) and updates `notification_log` accordingly.

**Consequences (testable):**
- GET `/api/webhooks/whatsapp` handles Meta's Hub Verify handshake: returns `hub.challenge` if `hub.verify_token` matches `WHATSAPP_VERIFY_TOKEN` env var; returns HTTP 403 otherwise.
- POST `/api/webhooks/whatsapp` processes status events (`delivered`, `read`, `failed`) by updating `notification_log.status` where `message_id` matches.
- Updates are idempotent: processing the same status event twice produces no duplicate log entries.

---

### 4.7 Cashfree Webhook — Automated Payment Confirmation

**Description:** Cashfree POSTs to `/api/webhooks/payment` on every payment event. The handler verifies HMAC-SHA256 signature, responds HTTP 200 immediately, then asynchronously marks the Rent Record paid, triggers the WhatsApp receipt (FR-21), and triggers the owner notification (FR-22). The Supabase DB update automatically broadcasts to the Owner's live dashboard via Realtime. Realizes UJ-2.

**Functional Requirements:**

#### FR-24: Cashfree Webhook HMAC Verification
Every inbound POST to `/api/webhooks/payment` must pass HMAC-SHA256 signature verification before any processing.

**Consequences (testable):**
- Signature computed from raw request body bytes using `PG_WEBHOOK_SECRET` env var.
- Request with missing or invalid `x-webhook-signature` header returns HTTP 401; no DB side-effects occur.
- Signature must be verified against raw bytes — not against the parsed JSON object — to prevent body re-serialization manipulation.

---

#### FR-25: Auto-Mark-Paid on PAYMENT_SUCCESS
When a verified PAYMENT_SUCCESS webhook arrives for a known pending Rent Record, the system marks the record as paid.

**Consequences (testable):**
- `rent_records.status` updated to `paid`; `amount_paid` set to event amount; `paid_at` set to current timestamp; `payment_method = gateway`; `pg_provider = cashfree`.
- Idempotency: if `status` is already `paid`, the webhook is silently discarded — no DB update, no duplicate notification.
- Lookup uses `pg_order_id` (Cashfree's canonical order reference), not tenant ID or month.
- HTTP 200 is returned *before* async DB writes to stay within Cashfree's 3-second retry threshold.
- After DB update, FR-21 (Tenant receipt) and FR-22 (Owner notification) are enqueued asynchronously.

---

#### FR-26: PAYMENT_FAILED Handling
When a verified PAYMENT_FAILED webhook arrives, the Rent Record status remains `pending`; the Owner is optionally notified.

**Consequences (testable):**
- Rent Record `status` unchanged (remains `pending`).
- `notification_log` entry created with `type = payment_failed`.
- Existing Payment Link remains valid for retry. [ASSUMPTION: Cashfree reuses the same order for retries; confirm with Cashfree docs whether a new order must be created after failure.]

---

### 4.8 Real-Time Collection Dashboard

**Description:** The Owner's dashboard shows live rent collection status for the current month. When a Cashfree webhook marks a Rent Record as paid, the Owner's dashboard updates within 1 second — no manual refresh. Supabase Realtime broadcasts the DB change to all connected Owner sessions. RLS ensures each Owner only receives their own events. Realizes UJ-2, UJ-3.

**Functional Requirements:**

#### FR-27: Live Rent Status Board
An authenticated Owner can view all Rent Records for the current month with real-time status updates.

**Consequences (testable):**
- Dashboard summary row: total beds, paid count, pending count, total due (₹), total collected (₹).
- Tenant table rows: name, room number, amount due, current status (`pending` / `paid` / `overdue` / `waived`), last reminder sent (relative timestamp), reminder count.
- Supabase Realtime subscription on `rent_records` filtered by `owner_id = <current owner>` — an UPDATE event flips the Tenant's status row in the UI with no full page reload.
- RLS enforces that the Realtime subscription delivers only this Owner's events, never another Owner's.
- Dashboard load time (initial data fetch for up to 50 Tenants) ≤ 2 seconds (P95).

---

#### FR-28: Occupancy Grid
An authenticated Owner can view a room-level occupancy overview.

**Consequences (testable):**
- Each room tile shows: room number, current Tenant name(s), current month status, monthly rent amount.
- Vacant rooms clearly distinguished from occupied rooms.
- Tapping/clicking a room tile navigates to the room detail + tenant management view.

---

#### FR-29: Month Navigation & Historical View
An authenticated Owner can navigate to previous months to review historical Rent Records.

**Consequences (testable):**
- Month selector navigates the dashboard to display records for any past month since account creation.
- Historical months are read-only: reminder trigger (FR-19) disabled for past months.
- Historical records show `payment_method`, `paid_at`, and reminder history.
- Navigating to a month before the account creation date returns an empty state (not an error).

---

### 4.9 Electricity Billing Module

**Description:** Owners can configure electricity billing per property in one of three modes: sub-meter (owner enters monthly unit readings per room), flat-rate (fixed amount per room/bed), or shared-split (total EB bill divided equally among active tenants). The electricity amount is added to the Rent Record and included in the combined payment link and WhatsApp message. An owner reminder is sent on the 27th of each month to enter meter readings before reminders go out.

**Functional Requirements:**

#### FR-E1: Electricity Config per Property
An authenticated Owner can set the electricity billing mode for a Property.

**Consequences (testable):**
- POST/PUT `/api/owner/properties/:id/electricity-config` accepts `billing_mode` (`sub_meter` | `flat_rate` | `shared_split` | `none`), `rate_per_unit` (for sub_meter), and `flat_amount` (for flat_rate).
- Config is stored in `electricity_config` table scoped to `owner_id` and `property_id`.
- Default mode is `none` — electricity billing is opt-in.
- Changing mode mid-month applies from the following month's Rent Records; current month records are not retroactively updated.

---

#### FR-E2: Meter Reading Entry (Sub-Meter Mode)
An authenticated Owner can enter monthly meter readings for each room in bulk.

**Consequences (testable):**
- POST `/api/owner/rooms/:id/meter-readings` accepts `month_year` (format: "2026-06"), `opening_reading`, `closing_reading`.
- `units_consumed` and `electricity_amount` are computed server-side: `(closing − opening) × rate_per_unit`.
- Opening reading for a new month is auto-populated from the previous month's closing reading (if available).
- Duplicate submission for the same `room_id + month_year` returns HTTP 409.
- If closing reading < opening reading, returns HTTP 422 ("Closing reading cannot be less than opening reading — did the meter reset?").

---

#### FR-E3: Shared Bill Entry (Shared-Split Mode)
An authenticated Owner can enter the total EB bill amount for a property; the system divides it equally among active tenants.

**Consequences (testable):**
- POST `/api/owner/properties/:id/shared-bill` accepts `month_year` and `total_bill_amount`.
- System divides `total_bill_amount` by the count of active tenants in that property for that month.
- Each active tenant's `rent_records.electricity_amount` is updated with their share.
- Re-submitting a shared bill for the same property + month_year overwrites the previous split and recalculates.

---

#### FR-E4: Electricity Amount Applied to Rent Records
When meter readings or flat-rate config are present, the electricity amount is applied to Rent Records before reminders go out.

**Consequences (testable):**
- For `sub_meter` mode: `electricity_amount` on the Rent Record is populated from `meter_readings` when the reading for that month is entered.
- For `flat_rate` mode: `electricity_amount` is populated automatically at Rent Record creation (FR-16 cron) using the configured flat amount.
- `total_amount = rent_amount + electricity_amount + cam_amount` is computed and stored; the Payment Link is generated for `total_amount`.
- If electricity readings have not been entered by the time a reminder is due (sub-meter mode), the reminder is sent with `electricity_amount = 0` and the combined total excludes electricity. Owner is notified separately to enter readings.

---

#### FR-E5: Combined WhatsApp Reminder Template
The WhatsApp reminder message shows a breakdown of rent, electricity, and CAM when applicable.

**Consequences (testable):**
- If `electricity_amount > 0`: WhatsApp body includes "⚡ Electricity: ₹[amount]" line.
- If `cam_amount > 0`: WhatsApp body includes "🏢 CAM: ₹[amount]" line.
- If both are zero: existing single-line format is used (no layout change for standard rent-only tenants).
- `total_amount` shown as "💳 Total Due: ₹[total]" with payment link.
- Template requires Meta re-approval with optional electricity and CAM components (OQ-7).

---

#### FR-E6: Owner Meter Reading Reminder
On the 27th of each month, the system sends a WhatsApp reminder to Owners who have sub-meter electricity billing configured but have not yet entered readings for the current month.

**Consequences (testable):**
- BullMQ cron at 09:00 IST on the 27th scans `electricity_config` where `billing_mode = sub_meter`.
- For each room where no `meter_readings` row exists for the current month → owner receives WhatsApp: "Enter meter readings before reminders go out: [dashboard link]".
- If all readings are entered, no WhatsApp is sent.
- Uses template `electricity_reading_reminder`.

---

### 4.10 Tenant UPI AutoPay Mandates

**Description:** Owners can invite tenants to set up a UPI AutoPay mandate via Cashfree Subscriptions API. Once authorized, Cashfree auto-debits the tenant's rent on the due date each month — no reminder needed. If auto-debit fails, the system immediately falls back to sending a standard payment link reminder. Move-out automatically cancels the mandate.

**Functional Requirements:**

#### FR-M1: Initiate Mandate Setup for Tenant
An authenticated Owner can trigger a mandate setup request for a specific tenant from the tenant detail page.

**Consequences (testable):**
- POST `/api/owner/tenants/:id/mandate/initiate` creates a Cashfree Subscription for the tenant's `monthly_rent` amount with `max_amount = monthly_rent × 1.5` (rounded to ₹100) and `cycle = monthly`.
- Returns HTTP 202; the Cashfree auth link is sent to the tenant via WhatsApp (see FR-M2).
- A row is created in `tenant_mandates` with `status = pending`.
- If an active mandate already exists for this tenant, returns HTTP 409.

---

#### FR-M2: Tenant Mandate Setup WhatsApp
The system sends a WhatsApp message to the tenant with the UPI AutoPay authorization link.

**Consequences (testable):**
- Template `autopay_setup_request` sent to tenant's phone immediately after FR-M1.
- Message explains: amount, monthly frequency, "reversible anytime from your UPI app", authorization link.
- If the tenant does not authorize within 48 hours, a follow-up WhatsApp nudge is sent once.

---

#### FR-M3: Mandate Authorization Webhook
The system processes Cashfree's `SUBSCRIPTION_AUTHORIZED` webhook and marks the tenant's mandate as active.

**Consequences (testable):**
- `tenant_mandates.status` updated to `active`; `authorized_at` set.
- `tenants.autopay_status` updated to `active`.
- Confirmation WhatsApp sent to tenant: `autopay_setup_confirmed` template.
- Owner notified via WhatsApp: "[Tenant name] has set up auto-pay for ₹[amount]/month."
- Idempotent: processing the same webhook twice produces no duplicate state change.

---

#### FR-M4: Reminder Skip for Auto-Pay Tenants
The daily reminder cron skips payment link generation and WhatsApp reminder for tenants with an active mandate.

**Consequences (testable):**
- In FR-17 cron: before enqueueing a Reminder Job, check `tenants.autopay_status`.
- If `autopay_status = active`: skip reminder job, log `autopay_active — reminder skipped` to `notification_log`.
- Cashfree auto-debits on the due date; no additional action by RentMaster needed.

---

#### FR-M5: Auto-Debit Success — Receipt
On `SUBSCRIPTION_CHARGE_SUCCESS` webhook, the system marks the Rent Record paid and sends a WhatsApp receipt.

**Consequences (testable):**
- Same downstream logic as FR-25 (PAYMENT_SUCCESS): Rent Record marked `paid`, `payment_method = autopay`.
- Template `autopay_success_receipt` sent to tenant.
- Owner notified via existing `owner_payment_received` template.
- Idempotent: duplicate webhook for same charge produces no duplicate record update.

---

#### FR-M6: Auto-Debit Failure — Fallback to Payment Link
On `SUBSCRIPTION_CHARGE_FAILED` webhook, the system immediately falls back to generating a payment link and sending a standard reminder.

**Consequences (testable):**
- `tenant_mandates` records the failure; `tenants.autopay_status` remains `active` (single failure does not cancel mandate).
- PaymentService.createPaymentLink() called immediately for the pending Rent Record.
- Template `autopay_failed_tenant` sent to tenant with the payment link.
- Owner notified via `autopay_failed_owner` template.
- On second consecutive month failure: owner notified with stronger alert; `autopay_status` set to `failed`.

---

#### FR-M7: Mandate Cancellation Handling
On `SUBSCRIPTION_CANCELLED` webhook (tenant cancels from their UPI app), the system reverts the tenant to manual payment mode.

**Consequences (testable):**
- `tenant_mandates.status` updated to `cancelled`; `cancelled_at` set.
- `tenants.autopay_status` updated to `cancelled`.
- Owner notified via `autopay_cancelled_owner` WhatsApp: "[Tenant name] cancelled auto-pay. They will now receive manual reminders."
- From the next billing cycle, FR-17 cron treats this tenant normally (sends reminders).

---

#### FR-M8: Mandate Cancellation on Move-Out
When a tenant move-out is recorded (FR-14), any active mandate for that tenant is cancelled automatically.

**Consequences (testable):**
- On POST `/api/owner/tenants/:id/move-out`: system looks up active `tenant_mandates` row for the tenant.
- If found: PaymentService.cancelSubscription() called → Cashfree cancels the mandate.
- `tenant_mandates.status` set to `cancelled`; `tenants.autopay_status` set to `cancelled`.
- No WhatsApp sent to tenant for system-initiated cancellation on move-out.

---

### 4.11 Owner Subscription Billing & Plan Calculator

**Description:** After the 14-day free trial, Owners pay a monthly subscription fee calculated per-slot: ₹60/slot for residential/PG (Tier 1, 1–30 slots; Tier 2 ₹50, 31–100 slots; Tier 3 ₹40, 101+ slots) and ₹80/slot for commercial (Tier 1). Owners pay via a Cashfree UPI AutoPay mandate set up once — the same infrastructure the platform provides to their tenants. A "Build Your Plan" calculator on the pricing page shows the math transparently before sign-up. When an owner's slot count changes, the subscription is updated in-place if within the mandate's `max_amount` ceiling; otherwise the mandate is cancelled and a new one is created with a re-auth request.

**Functional Requirements:**

#### FR-S1: Slot Count & Billing Calculation
The system calculates an Owner's monthly subscription amount from their active slot count at any point in time.

**Consequences (testable):**
- `calculateBilling(slots)` returns the correct monthly amount for any combination of residential and commercial slots.
- Residential slots Tier 1 (1–30): ₹60/slot. Tier 2 (31–100): first 30 at ₹60, remainder at ₹50. Tier 3 (101+): first 30 at ₹60, next 70 at ₹50, remainder at ₹40.
- Commercial slots: same tier thresholds at ₹80 / ₹65 / ₹50.
- Residential and commercial slot counts are tiered independently (a 5 residential + 3 commercial owner is in Tier 1 for both types).
- Function is pure — used by the plan calculator UI, billing service, and subscription update service.

---

#### FR-S2: Build-Your-Plan Calculator (Pricing Page)
The pricing page displays an interactive calculator that shows the monthly cost for any slot combination before sign-up.

**Consequences (testable):**
- Calculator accepts: residential room count, PG bed count, commercial shop/office count.
- Monthly total updates in real time as counts change.
- Shows the formula: "5 rooms × ₹60 = ₹300/month" — not just the total.
- Annual option displayed: monthly × 10 months (2 months free): "₹3,000/year (save ₹600)".
- Tier boundary label appears when slot count crosses 30 or 100: "You've entered Tier 2 — rate drops to ₹50/slot."
- Calculator is also shown during onboarding (step 0, before adding properties) and on the billing settings page.

---

#### FR-S3: Owner Subscription Creation (Post-Trial)
At the end of the 14-day trial, the Owner sets up a Cashfree UPI AutoPay mandate for their platform subscription.

**Consequences (testable):**
- Trial expiry triggers a subscription setup prompt on the dashboard and a WhatsApp message.
- Owner clicks "Set up auto-pay" → Cashfree Subscriptions API creates a subscription with `amount = calculateBilling(owner.slots)`, `max_amount = amount × 1.5` (rounded to ₹100), `cycle = monthly`.
- Owner is directed to the Cashfree-hosted auth page → authorizes in their UPI app.
- `SUBSCRIPTION_AUTHORIZED` webhook → `owner_subscriptions.status = active`.
- Alternative: "Pay manually" fallback option generates a one-time Cashfree payment link for the monthly amount; owner must repeat this each month.
- 14-day trial is full-featured; no credit card required to start.

---

#### FR-S4: Subscription Update on Slot Change
When an Owner adds or removes a property, room, or bed, the system recalculates billing and updates the Cashfree subscription in-place if possible.

**Consequences (testable):**
- After any slot-count-changing operation (create property/room/tenant, delete/soft-delete room), `updateOwnerSubscription(ownerId)` is called.
- If `newAmount === currentAmount`: no action.
- If `newAmount ≤ owner_subscriptions.max_amount`: Cashfree subscription amount updated via API; `owner_subscriptions.current_amount` updated; WhatsApp sent to owner showing old amount, new amount, slot breakdown, next debit date.
- If `newAmount > owner_subscriptions.max_amount`: existing subscription cancelled; new subscription created with `max_amount = newAmount × 1.5`; `owner_subscriptions.status = pending_auth`; owner sent re-auth WhatsApp with new authorization link.
- New amount takes effect from the **next billing cycle** — no proration for mid-cycle slot additions.

---

#### FR-S5: Subscription Charge & Failure Handling
The system processes Owner subscription charge webhooks and handles payment failures with a grace period.

**Consequences (testable):**
- `SUBSCRIPTION_CHARGE_SUCCESS`: `owner_subscriptions` updated; WhatsApp receipt sent to owner: "Your RentMaster subscription of ₹[amount] was auto-debited for [month]."
- `SUBSCRIPTION_CHARGE_FAILED`: owner notified via WhatsApp; 3-day grace period — platform remains active.
- If not resolved within 7 days: owner account suspended (read-only); WhatsApp sent daily on days 4–7.
- Account reactivates immediately on successful payment.

---

## 5. Non-Functional Requirements

### 5.1 Security

- **NFR-1: JWT Expiry & Rotation.** Access Tokens expire in 15 minutes. Refresh Tokens are stored hashed (never plaintext). Refresh Token rotated on every use — old token invalidated immediately, preventing replay.
- **NFR-2: Multi-Tenant Isolation (Dual Layer).** Supabase RLS enforces `owner_id = auth.uid()` at the database layer. NestJS TenantGuard provides a secondary application-level check. A misconfigured RLS policy does not expose data (TenantGuard catches it); a misconfigured guard does not expose data (RLS catches it).
- **NFR-3: Webhook Signature Enforcement.** All Cashfree webhooks verified via HMAC-SHA256 on raw request body bytes before any logic runs. All WhatsApp webhooks verified via Hub verify token. Unverified requests rejected HTTP 401 with no side-effects.
- **NFR-4: Secrets in Environment Variables.** All API keys, DB credentials, JWT secrets, and webhook secrets live in environment variables. No secret is hardcoded or committed to version control.
- **NFR-5: Bank Data Encryption at Rest.** Bank account numbers AES-256 encrypted at the application layer before DB storage. Decrypted only inside PaymentService when calling Cashfree. Never returned in API responses — only `lastFourDigits` exposed.
- **NFR-6: Password Hashing.** Passwords hashed via bcrypt (min cost factor 12) or delegated to Supabase Auth. Plaintext passwords never logged or stored.
- **NFR-18: Bank Change OTP Security.** OTP stored SHA-256 hashed with a 10-minute TTL. Single-use: invalidated immediately on correct submission. 3 failed attempts within the validity window invalidates the OTP. 5 OTP request attempts per Owner per hour enforced (rate limit). Email OTP only — no SMS for bank changes in Phase 1.

### 5.2 Reliability

- **NFR-7: BullMQ Persistence.** All Reminder Jobs and cron jobs persisted in Upstash Redis. Render.com worker restarts do not lose pending jobs — they resume automatically.
- **NFR-8: Webhook Idempotency.** All webhook handlers verify current DB state before applying updates. Processing the same event twice produces identical DB state (no duplicates, no double-notifications).
- **NFR-9: Reminder Deduplication.** A Tenant receives at most one reminder per 20-hour window per Rent Record. Deduplication checked at enqueueing (FR-17) and again at job processing (FR-18).

### 5.3 Performance

- **NFR-10: Dashboard Initial Load.** Rent Records for the current month (up to 50 Tenants) load in ≤ 2 seconds (P95).
- **NFR-11: Realtime Update Latency.** Supabase Realtime update visible on Owner's dashboard within 1 second (P95) of the Cashfree webhook DB write.
- **NFR-12: Webhook Response Deadline.** `/api/webhooks/payment` returns HTTP 200 within 3 seconds (Cashfree's retry threshold), regardless of async DB processing time.

### 5.4 Compliance & Legal

- **NFR-13: No RBI PA License Required.** Rent money flows Tenant → Owner's bank directly via Cashfree Beneficiary API. The platform company's bank account never holds or intermediates tenant rent funds. This architecture explicitly avoids the RBI Payment Aggregator license requirement.
- **NFR-14: PDPB 2023 (India).** Aadhaar documents stored encrypted in Supabase Storage; only last-4 digits in the DB. Tenant phone numbers in E.164 format. Owner must accept a Privacy Policy and Terms of Service at registration (pages must exist before Cashfree production KYC). Tenant data-delete endpoint available for Owner-initiated erasure requests.
- **NFR-15: GST Invoicing.** Not applicable in Phase 1 (no Owner subscription billing). Cashfree may deduct TDS on payouts; this is Cashfree's responsibility, not the platform's.

### 5.5 Observability

- **NFR-16: Notification Log.** Every WhatsApp send attempt (success or failure) recorded in `notification_log`: channel, type, status, Interakt `message_id`, raw payload (JSONB) for debugging. Delivery status updates recorded in the same row.
- **NFR-17: Dead-Letter Visibility.** BullMQ jobs that exhaust all retries land in the dead-letter queue. Developer can inspect via Render.com logs. Bull Board dashboard is a recommended Phase 1 addition but not a hard requirement.

---

## 6. Success Metrics

| Metric | Phase 1 Target | Counter-metric |
|---|---|---|
| WhatsApp reminder delivery rate (sent / jobs queued) | ≥ 95% | Reminder failure rate per owner (dead-letter queue entries) |
| Cashfree payment conversion (paid via link / received link) | ≥ 60% in first 30 days | Cash / manual payments still logged by owner |
| Realtime dashboard latency (webhook DB write → UI update) | ≤ 1 second P95 | Events with latency > 5s (Supabase Realtime lag metric) |
| Webhook processing success rate | ≥ 99.9% | Failed webhooks not recovered after 3 retries |
| Bank onboarding completion (registered → bank verified) | ≥ 70% within 24 hours of registration | Drop-off at penny-drop step |
| Auth reliability (Refresh Token errors causing forced re-login) | < 0.1% of active sessions | Support reports of unexpected logouts |

---

## 7. Open Questions

| # | Question | Impact | Owner | Revisit |
|---|---|---|---|---|
| OQ-1 | Cashfree Beneficiary Transfer vs Merchant Split Settlement — which API for direct-to-owner routing? Beneficiary API pushes payout post-collection; Split Settlement splits at capture. Which Cashfree account type is registered? | High — determines Cashfree product tier and KYC path | Adarsh | Before Sprint 2 / payment integration sprint |
| OQ-2 | WhatsApp template names and approved content for: `rent_reminder_3days`, `rent_reminder_due_today`, `rent_overdue`, `rent_receipt`, `owner_payment_received`. Must be submitted to Meta via Interakt **in Week 1** — approval takes 24–72 hours and blocks the entire reminder engine build. | Critical path | Adarsh | Week 1 |
| OQ-3 | Cashfree sandbox vs production: OPC/GST registration status? Sandbox available without GST; production Beneficiary API requires full business KYC. | Medium — blocks production Cashfree testing | Adarsh | Before Sprint 3 |
| OQ-4 | Supabase Auth as JWT issuer (NestJS calls `supabase.auth.getUser()` to verify) vs fully custom NestJS JWT issuer (NestJS signs and verifies with its own secret)? Affects FR-1 through FR-5 architecture. | High — core auth decision | Adarsh | Before Sprint 1 |
| OQ-5 | Rent proration for mid-month Tenant move-in: full first month or prorated amount? Currently assumed full month in FR-13. | Low | Adarsh | Before tenant onboarding sprint |
| OQ-6 | PAYMENT_FAILED retry flow (FR-26): does Cashfree reuse the existing order for tenant retries, or must a new `createPaymentLink()` call be made? Affects whether the existing `payment_link` remains valid after a failure. | Medium | Adarsh | Before Cashfree integration sprint |
| OQ-7 | WhatsApp template `rent_reminder` must be updated with optional electricity and CAM components (FR-E5). Does Meta allow optional sections in a single template, or must separate templates be created for rent-only vs rent+electricity? Submit to Interakt in Week 1 alongside existing templates. | High — blocks electricity reminder feature | Adarsh | Week 1 |
| OQ-8 | Cashfree Subscriptions API: does updating an existing subscription amount (FR-S4 in-place update) require the owner to re-authorize, or is it processed silently if within `max_amount`? Confirm from Cashfree docs before building FR-S4. | High | Adarsh | Before subscription sprint |
| OQ-9 | For tenant UPI AutoPay (FR-M1): what is the maximum mandate amount Cashfree supports via UPI AutoPay? For high-rent commercial tenants (₹50,000+/month), will UPI AutoPay work or is e-NACH required? | Medium — affects commercial tenant mandate eligibility | Adarsh | Before mandate sprint |
| OQ-10 | Electricity billing (FR-E4): when sub-meter readings have not been entered by the time the 3-day reminder is due, should the reminder be sent with electricity excluded (and owner separately reminded to enter readings), or should the reminder be held until readings are entered? Holding reminders adds complexity; sending without electricity is simpler. | Medium | Adarsh | Before electricity sprint |
| OQ-11 | Commercial escalation (FR-C4): the system WhatsApps the owner 60 and 30 days before escalation but does not auto-update the rent. Should there be a one-tap "Apply escalation" action in the WhatsApp message that updates `tenants.monthly_rent` directly? | Low — UX decision | Adarsh | Before commercial sprint |

---

## 8. Out of Scope — Phase 1

- **Phone / OTP login** — deferred; email + password and Google OAuth are Phase 1 auth.
- **Apple / other social logins** — deferred; only Google OAuth in Phase 1.
- **Razorpay integration** — PaymentService abstraction is ready; switching requires one env var change and zero DB migrations. Deferred.
- **Tenant-facing app or portal** — Tenants interact via WhatsApp links and the Cashfree payment page only.
- **Digital rental agreements / eSign** — Phase 2.
- **CSV / PDF export** — Phase 2.
- **GST invoice generation for commercial tenants** — Phase 2 (GSTIN on owner profile captured in Phase 1; invoice generation deferred).
- **e-NACH / net banking mandates** — UPI AutoPay only for Phase 1; e-NACH added in Phase 2.
- **Variable-amount mandate (rent + electricity combined)** — Phase 2; Phase 1 uses separate payment link when electricity is included.
- **Annual mandate for owner subscription** — Phase 2; Phase 1 monthly mandate only.
- **Electricity meter OCR** — Phase 2; manual reading entry only in Phase 1.
- **Commercial digital agreement templates** — Phase 2.
- **Large commercial / enterprise property management** (chains of 50+ shops, revenue-share / mall models) — not targeted.
- **NEFT/RTGS-only payment for high-value commercial rents** — Cashfree UPI handles up to ₹1L; above that is deferred.
- **Mobile app / React Native** — Year 2.
- **AI-powered insights or natural language queries** — Future.
- **Multi-currency** — INR only.

---

## 9. Assumptions Index

| ID | Section | Assumption | Status |
|---|---|---|---|
| A-1 | FR-1 to FR-5 | Supabase Auth issues the JWT; NestJS JwtGuard validates via `supabase.auth.getUser()` rather than a custom secret | Open — OQ-4 |
| A-2 | FR-13 | Mid-month Tenant move-in generates a full-month Rent Record (no proration) | Open — OQ-5 |
| A-3 | FR-18 | If Owner has no verified bank, reminder is sent without a Payment Link; fallback message text is "Contact your landlord to arrange payment" | Open — confirm with Interakt template submission (OQ-2) |
| A-4 | FR-22 | WhatsApp template `owner_payment_received` — content and Meta approval TBD | Open — OQ-2 |
| A-5 | FR-26 | After PAYMENT_FAILED, the existing Cashfree order / Payment Link can be retried by the Tenant without a new link being generated | Open — OQ-6 |
| A-6 | FR-2A | Supabase Auth automatically merges a Google OAuth sign-in with an existing password account when both share the same email address — no duplicate Owner records created | Open — verify in Supabase Auth settings before building the login page |
| A-7 | FR-8, FR-10A | First-time bank account registration (no prior `owner_payment_settings` row) does NOT require OTP — OTP is only required when overwriting previously saved bank details | Confirmed by user intent; verify edge case: what if the Owner deletes their bank record? Re-adding should be treated as first-time (no OTP). |
| A-8 | FR-S1 | Residential and commercial slot tiers are calculated independently. A 5-residential + 3-commercial owner is in Tier 1 for both, not combined into a higher tier. | Confirmed by design decision 2026-06-15 |
| A-9 | FR-S4 | New subscription amount takes effect from the next billing cycle only — no proration when slots are added mid-cycle. Owner gains access to new-slot features immediately. | Confirmed by design decision 2026-06-15 |
| A-10 | FR-M1 | `max_amount` on the tenant mandate is set to `monthly_rent × 1.5` (rounded to ₹100) at mandate creation. This buffer allows rent amounts to increase slightly (e.g., annual rent hike) without requiring tenant re-authorization, as long as the new amount stays within the ceiling. | Open — verify Cashfree Subscriptions API behaviour when amount is updated within max_amount (OQ-8) |
| A-11 | FR-E4 | For sub-meter mode, if meter readings are not entered before the reminder due date, the reminder is sent with `electricity_amount = 0` (electricity excluded from that month's bill). The owner is reminded separately to enter readings (FR-E6). The combined amount is not held pending readings. | Open — OQ-10 |
| A-12 | FR-C4 | The escalation tracker is advisory only — it reminds the owner but does not auto-update `tenants.monthly_rent`. The owner must manually apply the new rent after the escalation date. | Confirmed by design decision 2026-06-15 |
