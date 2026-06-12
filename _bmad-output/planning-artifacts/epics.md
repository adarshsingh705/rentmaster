---
stepsCompleted: [1, 2, 3, 4]
inputDocuments:
  - _bmad-output/planning-artifacts/prds/prd-rent-master-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/research/technical-rent-master-platform-architecture-research-2026-06-11.md
---

# RentMaster Phase 1 — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for RentMaster Phase 1, decomposing requirements from the final PRD and technical research into implementable stories for the Developer agent.

---

## Requirements Inventory

### Functional Requirements

FR-1: Owner Registration — An unauthenticated user can create a new Owner account using email and password.
FR-2: Owner Login — An Owner can log in with email and password to receive a new Access Token and Refresh Token pair.
FR-2A: Google OAuth Login — An Owner can sign in or create an account using their Google account ("Sign in with Google").
FR-3: Token Refresh — An Owner holding a valid Refresh Token can obtain a new Access Token without re-entering credentials.
FR-4: Logout — An authenticated Owner can log out, immediately invalidating their current Refresh Token.
FR-5: Route Protection (JwtGuard + TenantGuard) — Every API route under /api/owner/* requires a valid Access Token; TenantGuard injects owner_id into every request.
FR-6: Password Reset — An Owner can request a password reset email sent to their registered address.
FR-7: Owner Profile Completion — An authenticated Owner can set and update their full name and optional GSTIN.
FR-8: Bank Account Registration & Beneficiary Creation — An authenticated Owner can submit bank account details to register a Cashfree Beneficiary and initiate penny-drop verification; OTP (FR-10B) always required.
FR-9: Bank Verification Status — An authenticated Owner can view the current state of their bank account registration.
FR-10: Bank Account Re-Submission After Failure — An Owner whose penny-drop failed can re-submit corrected bank details (requires FR-10B OTP).
FR-10A: Penny-Drop Confirmation Webhook Handler — The system receives and processes the Cashfree penny-drop result webhook to update bank verification status and notify the Owner.
FR-10B: Bank Account OTP — Request & Verify — Every bank account submission (first-time, re-submission, change) requires a Bank Account OTP sent to the Owner's registered email, verified before any update is applied.
FR-11: Property CRUD — An authenticated Owner can create, read, update, and soft-delete their Properties.
FR-12: Room CRUD — An authenticated Owner can create, read, update, and soft-delete Rooms within their Properties.
FR-13: Tenant Onboarding — An authenticated Owner can add a Tenant to a Room with full name, phone, move-in date, monthly rent, rent due day, and optional fields; creates current-month Rent Record immediately.
FR-14: Tenant Move-Out — An authenticated Owner can record a move-out date for a Tenant, marking them inactive.
FR-15: Manual Rent Record Status Update (Waive / Partial) — An authenticated Owner can manually mark a pending Rent Record as waived or partial with amount and optional note.
FR-16: Monthly Rent Record Cron — The system automatically creates one rent_records row per active Tenant on the 1st of each month via BullMQ.
FR-17: Daily Reminder Cron Scan — The system scans rent_records at 07:00 IST daily and queues Reminder Jobs for all records meeting reminder criteria (3-days-before, due-today, 5+ days overdue).
FR-18: Payment Link Generation at Reminder Time — When a Reminder Job is processed, if the Rent Record has no Payment Link, the system generates one via PaymentService before sending the WhatsApp message.
FR-19: Manual Overdue Reminder Trigger — An authenticated Owner can manually trigger an overdue reminder for a specific pending Rent Record from the dashboard, bypassing the daily cron.
FR-20: Tenant Welcome WhatsApp on Move-In — When a new Tenant is added, the system sends a welcome WhatsApp message using the tenant_welcome template.
FR-21: WhatsApp Reminder Send — When a Reminder Job is processed, the system sends the appropriate pre-approved WhatsApp template to the Tenant's registered phone number.
FR-22: WhatsApp Payment Receipt — After PAYMENT_SUCCESS webhook processing, the system sends a WhatsApp receipt to the Tenant using the rent_receipt template.
FR-23: Owner Payment Notification — After PAYMENT_SUCCESS webhook processing, the system sends a WhatsApp notification to the Owner confirming which Tenant paid and the amount.
FR-24: WhatsApp Delivery Status Tracking — The system processes WhatsApp delivery status webhooks from Meta (via Interakt) and updates notification_log accordingly.
FR-25: Cashfree Webhook HMAC Verification — Every inbound POST to /api/webhooks/payment must pass HMAC-SHA256 signature verification on raw body before any processing.
FR-26: Auto-Mark-Paid on PAYMENT_SUCCESS — When a verified PAYMENT_SUCCESS webhook arrives for a known pending Rent Record, the system marks the record as paid and triggers notifications.
FR-27: PAYMENT_FAILED Handling — When a verified PAYMENT_FAILED webhook arrives, Rent Record stays pending; Owner optionally notified.
FR-28: Live Rent Status Board — An authenticated Owner can view all Rent Records for the current month with Supabase Realtime status updates (pending → paid live).
FR-29: Occupancy Grid — An authenticated Owner can view a room-level occupancy overview showing tenant names, room status, and current month rent status.
FR-30: Month Navigation & Historical View — An authenticated Owner can navigate to previous months to review historical Rent Records (read-only).

---

### Non-Functional Requirements

NFR-1: JWT Expiry & Rotation — Access Tokens expire in 15 minutes; Refresh Tokens stored hashed, rotated on every use, old token invalidated immediately.
NFR-2: Multi-Tenant Isolation (Dual Layer) — Supabase RLS enforces owner_id = auth.uid() at DB layer; NestJS TenantGuard provides secondary application-level check.
NFR-3: Webhook Signature Enforcement — All Cashfree webhooks verified via HMAC-SHA256 on raw body bytes; unverified requests rejected HTTP 401 with no side-effects.
NFR-4: Secrets in Environment Variables — All secrets in env vars, never hardcoded; Supabase service_role key restricted to server-side NestJS/worker only.
NFR-5: Bank Data Encryption at Rest — Bank account numbers AES-256 encrypted at application layer; only lastFourDigits ever returned in API responses.
NFR-6: Password Hashing — Passwords hashed via bcrypt (min cost factor 12); plaintext never logged or stored.
NFR-7: Bank Account OTP Security — OTP SHA-256 hashed, 10-minute TTL, single-use; 3 failed attempts invalidates; 5 requests/hour rate limit per Owner.
NFR-8: BullMQ Persistence — All Reminder Jobs and cron jobs persisted in Upstash Redis; worker restarts resume pending jobs automatically.
NFR-9: Webhook Idempotency — All webhook handlers check DB state before updating; same event processed twice produces identical state.
NFR-10: Reminder Deduplication — At most one reminder per Tenant per 20-hour window per Rent Record; checked at enqueue and at job processing.
NFR-11: Dashboard Initial Load — Current month Rent Records (up to 50 Tenants) load in ≤ 2 seconds (P95).
NFR-12: Realtime Update Latency — Supabase Realtime update visible on dashboard within 1 second (P95) of Cashfree webhook DB write.
NFR-13: Webhook Response Deadline — /api/webhooks/payment returns HTTP 200 within 3 seconds regardless of async processing time.
NFR-14: No RBI PA License Required — Rent money flows Tenant → Owner bank directly via Cashfree Beneficiary API; platform never intermediates funds.
NFR-15: PDPB 2023 (India) — Aadhaar last-4 only in DB; data-delete endpoint available; Privacy Policy and ToS required before Cashfree production KYC.
NFR-16: GST Invoicing — Not applicable Phase 1; no Owner subscription billing.
NFR-17: Notification Log — Every WhatsApp send attempt logged to notification_log with channel, type, status, message_id, raw payload.
NFR-18: Dead-Letter Visibility — BullMQ jobs exhausting all retries land in dead-letter queue; inspectable via Render.com logs.

---

### Additional Requirements

From Technical Architecture Research:

- **Backend Framework**: NestJS on top of Express (not raw Express); use NestJS DI, module system, Guards (JwtGuard, TenantGuard), Interceptors.
- **Frontend Framework**: Next.js 14 App Router + shadcn/ui + Tailwind CSS; hosted on Vercel free tier.
- **Hybrid Deployment**: Vercel (Next.js frontend + simple API routes) + Render.com (NestJS background worker for BullMQ + cron). Worker is a separate always-on process, not serverless.
- **Database**: Supabase PostgreSQL with Row Level Security. Prisma ORM for type-safe queries and migrations.
- **Auth**: Supabase Auth issues JWT; NestJS JwtGuard validates via supabase.auth.getUser(). Supabase Auth handles both email+password and Google OAuth natively.
- **Job Queue**: BullMQ + Upstash Redis (free tier: 10,000 commands/day). Use BullMQ repeatable jobs for cron — not node-cron in isolation.
- **Payment Gateway**: Cashfree via PaymentService abstraction layer (lib/payment-service.js). Gateway-agnostic: switching to Razorpay requires only env var change + zero DB migrations.
- **WhatsApp**: Interakt BSP (₹999/month). All 6 templates (rent_reminder_3days, rent_reminder_due_today, rent_overdue, rent_receipt, owner_payment_received, tenant_welcome) must be submitted to Meta for approval before sprint building the reminder engine. Payment links must be CTA buttons in templates, not inline URLs.
- **Email**: Resend (3,000 free/month) for password reset emails and bank OTP emails.
- **File Storage**: Supabase Storage for Aadhaar documents (Phase 2 — out of scope Phase 1).
- **Realtime**: Supabase Realtime (postgres_changes) for live dashboard updates — no Socket.IO needed.
- **CI/CD**: GitHub → Vercel (auto-deploy frontend) + Render.com (auto-deploy worker on push).
- **Database Schema**: Multi-tenant shared schema + RLS. Tables: owners, properties, rooms, tenants, rent_records, notification_log, owner_payment_settings. owner_id denormalised on every table for RLS and index performance.
- **⚠️ Critical Path**: WhatsApp templates must be submitted in Week 1 of development (24–72 hour Meta approval window). Approval blocks the entire reminder engine sprint.

---

### UX Design Requirements

No UX Design document exists for Phase 1. UX will be derived from User Journeys (UJ-1, UJ-2, UJ-3) and FR acceptance criteria in the PRD. A bmad-ux pass is recommended after epic/story creation.

---

### FR Coverage Map

| FR | Epic |
|---|---|
| FR-1 | Epic 1 |
| FR-2 | Epic 1 |
| FR-2A | Epic 1 |
| FR-3 | Epic 1 |
| FR-4 | Epic 1 |
| FR-5 | Epic 1 |
| FR-6 | Epic 1 |
| FR-7 | Epic 1 |
| FR-8 | Epic 3 |
| FR-9 | Epic 3 |
| FR-10 | Epic 3 |
| FR-10A | Epic 3 |
| FR-10B | Epic 3 |
| FR-11 | Epic 2 |
| FR-12 | Epic 2 |
| FR-13 | Epic 2 |
| FR-14 | Epic 2 |
| FR-15 | Epic 2 |
| FR-16 | Epic 5A |
| FR-17 | Epic 5A |
| FR-18 | Epic 5A |
| FR-19 | Epic 5A |
| FR-20 | Epic 4 (stub wired in Epic 2 Story 2.3) |
| FR-21 | Epic 4 |
| FR-22 | Epic 4 |
| FR-23 | Epic 4 |
| FR-24 | Epic 4 |
| FR-25 | Epic 5B |
| FR-26 | Epic 5B |
| FR-27 | Epic 5B |
| FR-28 | Epic 6 |
| FR-29 | Epic 6 |
| FR-30 | Epic 6 |

---

## Epic List

| # | Title | FRs | Depends On |
|---|---|---|---|
| Epic 1 | Secure Owner Authentication & Profile | FR-1, 2, 2A, 3, 4, 5, 6, 7 | — |
| Epic 2 | Property, Room & Tenant Management | FR-11, 12, 13, 14, 15 | Epic 1 (property_type drives room-based vs bed-based inventory) |
| Epic 3 | Bank Account Onboarding & Payment Link Infrastructure | FR-8, 9, 10, 10A, 10B | Epic 2 |
| Epic 4 | WhatsApp Notification System | FR-20, 21, 22, 23, 24 | Epic 2 (parallel with Epic 3, priority: Epic 3) |
| Epic 5A | Automated Rent Collection Scheduler | FR-16, 17, 18, 19 | Epic 3, Epic 4 |
| Epic 5B | Payment Webhook & Auto-Settlement | FR-25, 26, 27 | Epic 5A |
| Epic 6 | Live Collection Dashboard | FR-28, 29, 30 | Epic 5B |

**Build order:** Epic 1 → Epic 2 → Epics 3 & 4 (parallel, Epic 3 is critical-path priority) → Epic 5A → Epic 5B → Epic 6

**Key pre-build actions (before code):**
- Submit all 6 WhatsApp templates to Meta via Interakt on Day 1 of Epic 3 sprint (24–72 hr approval window)
- Confirm Cashfree Beneficiary Transfer API sandbox access before Epic 3 sprint begins

---

## Epic 1: Secure Owner Authentication & Profile

**Goal:** Establish the complete authentication foundation — registration, login (email+password and Google OAuth), token refresh/rotation, logout, password reset, and profile completion — so that every subsequent epic can rely on a secure, multi-tenant-isolated Owner identity.

**FRs:** FR-1, FR-2, FR-2A, FR-3, FR-4, FR-5, FR-6, FR-7
**NFRs:** NFR-1, NFR-2, NFR-4, NFR-6
**Additional:** NestJS JwtGuard + TenantGuard; EmailModule (Resend) extracted as shared module for Epic 3 reuse

### Story 1.0: Project Scaffolding & Environment Setup

As a developer,
I want the NestJS + Next.js monorepo scaffolded with Supabase, Prisma, and Resend wired up,
So that every subsequent story has a working foundation to build on.

**Acceptance Criteria:**

**Given** the repository is initialised
**When** `npm install` runs
**Then** both the NestJS worker (`/apps/api`) and Next.js frontend (`/apps/web`) build without errors

**Given** environment variables are configured per `.env.example`
**When** the NestJS server starts
**Then** Supabase client connects successfully; Prisma generates types from the schema; Resend client initialises

**Given** the initial Prisma migration runs
**When** `prisma migrate dev` executes
**Then** the `owners` table is created in Supabase with `owner_id`, `email`, `full_name`, `gstin`, `bank_verified`, `refresh_token_hash` columns; RLS enabled on the table with policy `owner_id = auth.uid()`

**And** `.env.example` documents all required keys: `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `JWT_SECRET`, `RESEND_API_KEY`; a `README.md` documents local dev setup steps including ngrok tunnel setup for webhook testing

### Story 1.1: Owner Registration with Email & Password

As an unauthenticated user,
I want to create a new Owner account using my email and password,
So that I can access the RentMaster platform.

**Acceptance Criteria:**

**Given** a user submits a valid email and password (min 8 chars) to `POST /api/auth/register`
**When** the email is not already registered
**Then** a new Owner account is created in Supabase Auth, an `owners` row is inserted with `owner_id = auth.uid()`, and HTTP 201 is returned with the owner's ID and email

**Given** a user submits a password shorter than 8 characters
**When** the request is validated
**Then** HTTP 400 is returned with a field-level validation error; no account is created

**Given** a user submits an email that is already registered
**When** Supabase Auth returns a duplicate error
**Then** HTTP 409 is returned; no duplicate account is created; plaintext password is never logged

### Story 1.2: Owner Login & JWT Token Pair Issuance

As a registered Owner,
I want to log in with my email and password,
So that I receive an Access Token and Refresh Token to use the platform.

**Acceptance Criteria:**

**Given** an Owner submits correct email and password to `POST /api/auth/login`
**When** Supabase Auth verifies the credentials
**Then** HTTP 200 is returned with an Access Token (15-min TTL) and a Refresh Token (7-day TTL); the Refresh Token is stored hashed (SHA-256) in the `owners` table, never in plaintext

**Given** an Owner submits incorrect credentials
**When** Supabase Auth rejects the request
**Then** HTTP 401 is returned with a generic "Invalid credentials" message; no token is issued; no information about whether the email exists is revealed

**Given** a valid login
**When** the Access Token is issued
**Then** the JWT payload contains `sub` (owner_id), `email`, and `exp`; bcrypt cost factor for password hashing is ≥ 12

### Story 1.3: Google OAuth Sign-In

As an unauthenticated user,
I want to sign in or create an account using my Google account,
So that I can access RentMaster without managing a separate password.

**Acceptance Criteria:**

**Given** a user initiates Google OAuth via `GET /api/auth/google`
**When** they complete the Google consent flow
**Then** Supabase Auth creates or retrieves the Owner account linked to that Google identity; an `owners` row exists for the returned `auth.uid()`; HTTP 200 is returned with an Access Token and Refresh Token pair

**Given** a Google OAuth user who already has an email+password account with the same email
**When** they sign in via Google
**Then** Supabase Auth links the Google identity to the existing account; the Owner can still log in via email+password

**Given** the OAuth callback fails (user cancels or network error)
**When** Supabase returns an error
**Then** HTTP 400 is returned; no partial account is created

### Story 1.4: Token Refresh & Rotation

As an authenticated Owner,
I want to exchange my Refresh Token for a new Access Token,
So that I stay logged in without re-entering my credentials.

**Acceptance Criteria:**

**Given** an Owner submits a valid (non-expired, non-invalidated) Refresh Token to `POST /api/auth/refresh`
**When** the hashed token matches the stored hash
**Then** HTTP 200 is returned with a new Access Token (15-min TTL) and a new Refresh Token (7-day TTL); the old Refresh Token hash is immediately deleted from the DB (rotation); plaintext of old token is never stored

**Given** an Owner submits an already-used (rotated-out) Refresh Token
**When** no matching hash exists in the DB
**Then** HTTP 401 is returned; no new tokens issued

**Given** an Owner submits an expired Refresh Token
**When** the TTL check fails
**Then** HTTP 401 is returned; Owner must re-authenticate

### Story 1.5: Logout & Refresh Token Invalidation

As an authenticated Owner,
I want to log out,
So that my Refresh Token is immediately invalidated and cannot be reused.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `POST /api/auth/logout` with a valid Access Token
**When** the request is processed
**Then** HTTP 200 is returned; the Owner's current Refresh Token hash is deleted from the `owners` table; the Access Token continues to be valid until its natural 15-min expiry (stateless — no blocklist needed)

**Given** an Owner calls logout with an invalid or expired Access Token
**When** JwtGuard rejects the token
**Then** HTTP 401 is returned

### Story 1.6: Route Protection via JwtGuard & TenantGuard

As the system,
I want every protected route to validate the Access Token and inject the owner's identity,
So that no Owner can access another Owner's data.

**Acceptance Criteria:**

**Given** a request to any route under `/api/owner/*` with a valid Access Token
**When** JwtGuard calls `supabase.auth.getUser()` and TenantGuard extracts `owner_id`
**Then** `request.ownerId` is populated; the request proceeds to the handler

**Given** a request with a missing or malformed Access Token
**When** JwtGuard validates the token
**Then** HTTP 401 is returned before the handler is reached; no DB query is made

**Given** a request with an expired Access Token
**When** JwtGuard validates the token
**Then** HTTP 401 is returned with a message indicating token expiry; no DB query is made

**And** Supabase RLS `owners` table policy `owner_id = auth.uid()` is in place and tested with both service role and anon role; RLS policy is annotated for Realtime compatibility (Epic 6)

### Story 1.7: Password Reset via Email

As a registered Owner who has forgotten their password,
I want to receive a password reset email,
So that I can regain access to my account.

**Acceptance Criteria:**

**Given** an Owner submits their registered email to `POST /api/auth/forgot-password`
**When** the email exists in Supabase Auth
**Then** Supabase Auth sends a password reset email via Resend; HTTP 200 is returned with a generic success message (same response whether email exists or not, to prevent enumeration)

**Given** the Owner clicks the reset link and submits a new password (min 8 chars) to `POST /api/auth/reset-password`
**When** the reset token is valid and not expired
**Then** Supabase Auth updates the password; all existing Refresh Tokens for the Owner are invalidated; HTTP 200 is returned

**Given** an invalid or expired reset token
**When** Supabase Auth rejects it
**Then** HTTP 400 is returned; password is not changed

**And** the `EmailModule` (Resend client) is extracted as a shared NestJS module importable by Epic 3's bank OTP flow

### Story 1.8: Owner Profile Completion

As an authenticated Owner,
I want to set and update my full name and optional GSTIN,
So that my profile is complete for Cashfree KYC and platform use.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `PATCH /api/owner/profile` with a full name (required) and optional GSTIN
**When** TenantGuard has injected `owner_id`
**Then** the `owners` row is updated (`full_name`, `gstin`); HTTP 200 is returned with the updated profile; RLS ensures only the Owner's own row is updated

**Given** a GSTIN is provided
**When** the value is validated
**Then** it must match the 15-character Indian GSTIN format; HTTP 400 returned if invalid

**Given** an Owner calls `GET /api/owner/profile`
**When** TenantGuard has injected `owner_id`
**Then** HTTP 200 returns `owner_id`, `email`, `full_name`, `gstin` (or null), `bank_verified` status

---

## Epic 2: Property, Room & Tenant Management

**Goal:** Give the Owner the full data model for their portfolio — properties (room-based or bed-based), rooms with configurable capacity, optional named beds, tenants assigned to the correct inventory unit with occupancy enforcement, move-out freeing the unit for reassignment, and manual rent record status updates.

**FRs:** FR-11, FR-12, FR-13, FR-14, FR-15
**NFRs:** NFR-2 (RLS on all new tables, `owner_id` denormalised), NFR-10 (rent_records structure supports dedup)
**Cross-epic note:** Story 2.4 wires `NotificationService` no-op stub for FR-20; real send lands in Epic 4

**Schema introduced in this epic:**

| Table | Key columns |
|---|---|
| `properties` | `property_type ENUM('room_based','bed_based') NOT NULL` |
| `rooms` | `capacity INT NOT NULL DEFAULT 1`, `floor TEXT NULL`; no `active_tenant_id` — occupancy checked via count query |
| `beds` | `id`, `owner_id`, `property_id`, `room_id`, `name`, `active_tenant_id UUID NULL`, `deleted_at` — bed_based only |
| `tenants` | `room_id`, `bed_id NULL` (null for room_based), `active BOOL` |

### Story 2.1: Property CRUD

As an authenticated Owner,
I want to create a Property and declare whether it is room-based or bed-based,
So that the system knows how to manage inventory and tenant assignments for that property.

**Acceptance Criteria:**

**Given** an Owner calls `POST /api/owner/properties` with a property name, optional address, and `property_type` (`room_based` or `bed_based`)
**When** TenantGuard has injected `owner_id`
**Then** a `properties` row is created with `owner_id` and `property_type` stored; HTTP 201 returns the new property including `property_type`

**Given** `property_type` is omitted or an invalid value
**When** the request is validated
**Then** HTTP 400 returned with a field-level error; no property created

**Given** an Owner calls `GET /api/owner/properties`
**When** TenantGuard has injected `owner_id`
**Then** HTTP 200 returns each property with `property_type`; soft-deleted properties excluded

**Given** an Owner calls `PATCH /api/owner/properties/:id` with updated name/address
**When** the property belongs to the requesting Owner
**Then** HTTP 200 returns the updated property; `property_type` cannot be changed after creation (HTTP 422 if attempted)

**Given** an Owner calls `DELETE /api/owner/properties/:id`
**When** the property belongs to the requesting Owner
**Then** `deleted_at` is set (soft delete); HTTP 200 returned

**And** RLS on `properties` enforces `owner_id = auth.uid()`; policy annotated for Realtime compatibility

### Story 2.2: Room CRUD

As an authenticated Owner,
I want to create, view, update capacity, and soft-delete Rooms within my Properties,
So that I can accurately represent how many tenants each room can hold and track occupancy.

**Acceptance Criteria:**

**Given** an Owner calls `POST /api/owner/properties/:propertyId/rooms` with a room name, optional `capacity` (integer ≥ 1, default 1), and optional `floor` (free-text, e.g. "Ground Floor", "1st Floor")
**When** the property belongs to the requesting Owner
**Then** a `rooms` row is created with `owner_id`, `property_id`, `capacity`, and `floor` (nullable) stored; HTTP 201 returns the new room including `floor`

**Given** an Owner calls `PATCH /api/owner/rooms/:id` to update name, `capacity`, or `floor`
**When** the new `capacity` is ≥ the current `active_tenant_count` in that room
**Then** the update is applied; HTTP 200 returned

**Given** an Owner tries to reduce `capacity` below the current active tenant count
**When** the validation runs
**Then** HTTP 409 returned ("Cannot reduce capacity below current occupancy — move out tenants first"); no change made

**Given** an Owner calls `GET /api/owner/properties/:propertyId/rooms`
**When** TenantGuard has injected `owner_id`
**Then** HTTP 200 returns each room with `capacity`, `active_tenant_count`, `occupancy_status` (`vacant` | `partial` | `full`), and `floor` (null if not set); for `bed_based` rooms also returns `bed_count` and `vacant_bed_count`; soft-deleted rooms excluded

**Given** an Owner calls `DELETE /api/owner/rooms/:id`
**When** `active_tenant_count = 0` for that room (and all beds vacant for bed_based)
**Then** `deleted_at` is set; HTTP 200 returned

**Given** the room has ≥ 1 active tenant
**When** delete is attempted
**Then** HTTP 409 returned ("Move out all tenants before deleting this room"); no deletion made

**And** RLS on `rooms` enforces `owner_id = auth.uid()`; policy annotated for Realtime compatibility

### Story 2.3: Bed CRUD (bed_based properties only)

As an authenticated Owner with a bed-based property,
I want to add, rename, and soft-delete named Beds within a Room,
So that I can track every individual rentable bed in my PG and see which are vacant.

**Acceptance Criteria:**

**Given** an Owner calls `POST /api/owner/rooms/:roomId/beds` with a bed name/label (e.g. "Bed A", "Bed 1")
**When** the room belongs to a `bed_based` property owned by the requesting Owner
**Then** a `beds` row is created with `owner_id`, `property_id`, `room_id` denormalised and `active_tenant_id = null`; HTTP 201 returns the new bed

**Given** an Owner calls `POST /api/owner/rooms/:roomId/beds` on a `room_based` property
**When** the property type is checked
**Then** HTTP 422 returned ("Beds are not applicable for room-based properties — use room capacity instead"); no bed created

**Given** an Owner calls `GET /api/owner/rooms/:roomId/beds`
**When** TenantGuard has injected `owner_id`
**Then** HTTP 200 returns all beds with `occupancy_status` (`vacant` or `occupied`) and `tenant_name` if occupied; soft-deleted beds excluded

**Given** an Owner calls `DELETE /api/owner/beds/:id`
**When** `beds.active_tenant_id IS NULL` (bed is vacant)
**Then** `deleted_at` is set; HTTP 200 returned

**Given** an Owner tries to delete a bed with an active tenant
**When** the occupancy check runs
**Then** HTTP 409 returned ("Bed is occupied — move out the tenant first"); no deletion made

**And** RLS on `beds` enforces `owner_id = auth.uid()`; policy annotated for Realtime compatibility

### Story 2.4: Tenant Onboarding

As an authenticated Owner,
I want to assign a new Tenant to a vacant Room or Bed depending on my property type,
So that the system prevents double-booking and tracks rent at the correct inventory level.

**Acceptance Criteria:**

**Given** an Owner calls `POST /api/owner/tenants` with full name (required), phone (required, 10-digit Indian mobile), move-in date, monthly rent (> 0), rent due day (1–28), `room_id`, and optional fields (email, Aadhaar last-4) — for a `room_based` property
**When** `COUNT(active tenants WHERE room_id = :id) < rooms.capacity`
**Then** a `tenants` row is created with `owner_id`, `property_id`, `room_id` denormalised; a `rent_records` row is created for the current month with `status = pending`; HTTP 201 returned

**Given** the target room (`room_based`) is at full capacity (`active_tenant_count = capacity`)
**When** the occupancy check runs
**Then** HTTP 409 returned ("Room is full — move out a tenant or increase room capacity before assigning"); no tenant or rent record created

**Given** an Owner calls `POST /api/owner/tenants` with `room_id` and `bed_id` — for a `bed_based` property
**When** `beds.active_tenant_id IS NULL` (bed is vacant)
**Then** a `tenants` row is created with `owner_id`, `property_id`, `room_id`, `bed_id` all denormalised; `beds.active_tenant_id` set to the new `tenant_id`; rent record created; HTTP 201 returned

**Given** the target bed (`bed_based`) already has an active tenant
**When** the occupancy check runs
**Then** HTTP 409 returned ("Bed is occupied — move out the current tenant before assigning a new one"); no tenant or rent record created

**Given** a `bed_id` is passed for a `room_based` property, or only `room_id` for `bed_based`
**When** property type is checked
**Then** HTTP 422 returned with a clear mismatch message; no tenant created

**Given** a phone number that is not a valid 10-digit Indian mobile
**When** the request is validated
**Then** HTTP 400 returned; no tenant or rent record created

**Given** a tenant is successfully onboarded
**When** the row is committed
**Then** `NotificationService.sendWelcome(tenant)` is called; Epic 2 implementation is a no-op stub (interface defined, injected as null-object); stub must not throw and must log at DEBUG level

**And** RLS on `tenants` and `rent_records` enforces `owner_id = auth.uid()`; policies annotated for Realtime compatibility

### Story 2.5: Tenant Move-Out

As an authenticated Owner,
I want to record a Tenant's move-out date,
So that their Room or Bed is immediately freed for reassignment and no future rent records are generated.

**Acceptance Criteria:**

**Given** an Owner calls `PATCH /api/owner/tenants/:id/move-out` with a `move_out_date` for a `room_based` tenant
**When** the tenant belongs to the requesting Owner and `move_out_date ≥ move_in_date`
**Then** `tenants.move_out_date` is set and `tenants.active = false`; the room's active count drops by 1 automatically (count query — no column to clear); HTTP 200 returned

**Given** the same move-out for a `bed_based` tenant
**When** conditions above are met
**Then** `tenants.move_out_date` set and `tenants.active = false`; `beds.active_tenant_id` set to `null` (bed is now vacant and assignable to a new tenant); HTTP 200 returned

**Given** the tenant already has `active = false`
**When** move-out is called again
**Then** HTTP 409 returned; no change made

**Given** a `move_out_date` earlier than `move_in_date`
**When** the request is validated
**Then** HTTP 400 returned; no changes made

### Story 2.6: Manual Rent Record Status Update (Waive / Partial)

As an authenticated Owner,
I want to manually mark a pending Rent Record as waived or partial,
So that I can record cash payments or goodwill waivers outside the automated payment flow.

**Acceptance Criteria:**

**Given** an Owner calls `PATCH /api/owner/rent-records/:id/status` with `status = waived` and an optional note
**When** the rent record belongs to the requesting Owner and currently has `status = pending`
**Then** `rent_records.status` is set to `waived`; `payment_method = null`; HTTP 200 returned with the updated record

**Given** an Owner calls the same endpoint with `status = partial`, `amount_paid` (> 0 and < `amount_due`), and `payment_method = cash`
**When** the rent record is currently `pending`
**Then** `rent_records.status = partial`, `amount_paid` and `payment_method` are recorded; HTTP 200 returned

**Given** an Owner attempts to update a rent record with `status = paid`
**When** the request is validated
**Then** HTTP 400 returned; `paid` status can only be set by the webhook handler (Epic 5B)

**Given** the rent record does not belong to the requesting Owner
**When** RLS is evaluated
**Then** HTTP 404 returned; no update made

---

## Epic 3: Bank Account Onboarding & Payment Link Infrastructure

**Goal:** Allow the Owner to register their bank account — always gated by email OTP — create a Cashfree Beneficiary, complete penny-drop verification via webhook, and handle re-submission after failure. This is the prerequisite for payment link generation in Epic 5A.

**FRs:** FR-8, FR-9, FR-10, FR-10A, FR-10B
**NFRs:** NFR-3 (HMAC on penny-drop webhook), NFR-5 (AES-256 encryption), NFR-7 (OTP security), NFR-9 (webhook idempotency)
**Pre-sprint action:** Submit all 6 WhatsApp templates to Meta via Interakt (24–72 hr approval window) on Day 1 of this sprint

### Story 3.1: Webhook Infrastructure Spike

As a developer,
I want the NestJS worker deployed and able to receive Cashfree webhooks,
So that all subsequent webhook-dependent stories can be developed and tested end-to-end.

**Acceptance Criteria:**

**Given** the NestJS worker is deployed to Render.com (or a tunnel is active for local dev)
**When** a test POST is sent to `/api/webhooks/payment` and `/api/webhooks/penny-drop`
**Then** the worker receives the request and returns HTTP 200; the request is visible in Render.com logs

**Given** Cashfree sandbox is configured with the worker's public URL as the webhook endpoint
**When** a sandbox penny-drop event is triggered
**Then** the NestJS worker receives the webhook payload; the raw body is available for HMAC verification

**And** ngrok (or equivalent) is documented in the project README for local dev so future stories don't repeat this setup

### Story 3.2: Bank Account OTP — Request & Verify

As an authenticated Owner,
I want to request and verify an email OTP before submitting any bank account details,
So that my bank account cannot be changed without my explicit email confirmation.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `POST /api/owner/bank/otp/request`
**When** fewer than 5 OTP requests have been made in the past hour for this Owner
**Then** a 6-digit OTP is generated, SHA-256 hashed and stored with a 10-minute TTL; the plaintext OTP is sent to the Owner's registered email via `EmailModule` (Resend); HTTP 200 returned

**Given** an Owner has made 5 or more OTP requests in the past hour
**When** a new request is made
**Then** HTTP 429 returned; no OTP sent

**Given** an Owner calls `POST /api/owner/bank/otp/verify` with the correct OTP within 10 minutes
**When** the hash matches and the OTP has not been used
**Then** HTTP 200 returned with a short-lived `otp_token` (signed JWT, 15-min TTL) that authorises one bank account submission; the OTP is marked single-use immediately

**Given** an Owner submits an incorrect OTP
**When** the hash does not match
**Then** HTTP 400 returned; attempt counter incremented; after 3 failed attempts the OTP is invalidated and a new request is required

**Given** an Owner submits a correct OTP that has expired (> 10 min)
**When** the TTL check fails
**Then** HTTP 400 returned; Owner must request a new OTP

### Story 3.3: Bank Account Registration & Cashfree Beneficiary Creation

As an authenticated Owner,
I want to submit my bank account details after OTP verification,
So that Cashfree can create a Beneficiary and initiate penny-drop verification.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `POST /api/owner/bank/register` with a valid `otp_token`, account holder name, account number, IFSC code, and account type
**When** the `otp_token` is valid (not expired, not already consumed)
**Then** the account number is AES-256 encrypted before storage; only `last_four_digits` stored in plaintext; `PaymentService.createBeneficiary()` is called; `otp_token` is consumed (single-use); `owner_payment_settings.bank_status` set to `PENDING_VERIFICATION`; HTTP 202 returned

**Given** the `otp_token` is missing, expired, or already consumed
**When** the request is validated
**Then** HTTP 403 returned; no bank details stored; Cashfree API not called

**Given** `PaymentService.createBeneficiary()` returns an error from Cashfree
**When** the API call fails
**Then** `bank_status` set to `CREATION_FAILED`; error details logged (no plaintext bank data in logs); HTTP 502 returned

**And** `PaymentService` is a gateway-agnostic abstraction; switching from Cashfree requires only an env var change; no direct Cashfree SDK calls outside this service

### Story 3.4: Bank Verification Status View

As an authenticated Owner,
I want to view the current state of my bank account registration,
So that I know whether penny-drop verification has passed, is pending, or has failed.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `GET /api/owner/bank/status`
**When** TenantGuard has injected `owner_id`
**Then** HTTP 200 returns `bank_status` (one of: `NOT_REGISTERED`, `PENDING_VERIFICATION`, `VERIFIED`, `FAILED`), `last_four_digits` (or null), and `account_holder_name`; full account number is never returned

**Given** no bank account has been registered
**When** the endpoint is called
**Then** HTTP 200 returns `{ bank_status: "NOT_REGISTERED" }`

### Story 3.5: Penny-Drop Confirmation Webhook Handler

As the system,
I want to receive and process Cashfree penny-drop result webhooks,
So that the Owner's bank verification status is automatically updated.

**Acceptance Criteria:**

**Given** Cashfree sends a POST to `/api/webhooks/penny-drop` with a valid HMAC-SHA256 signature
**When** the raw body bytes are verified against the signature using the Cashfree webhook secret
**Then** if result is `SUCCESS`, `owner_payment_settings.bank_status` is set to `VERIFIED` and `bank_verified = true` on the `owners` row; if result is `FAILED`, `bank_status` is set to `FAILED`; HTTP 200 returned within 3 seconds

**Given** a webhook with an invalid or missing HMAC signature
**When** verification fails
**Then** HTTP 401 returned; no DB changes made; attempt logged

**Given** the same penny-drop webhook event is received twice (duplicate delivery)
**When** the handler checks DB state before updating
**Then** idempotency is preserved — second delivery produces identical DB state; HTTP 200 returned both times

**And** after status update, a Supabase Realtime broadcast is triggered on the `owner_payment_settings` channel so the Owner's UI updates live

### Story 3.6: Bank Account Re-Submission After Failure

As an authenticated Owner whose penny-drop verification has failed,
I want to submit corrected bank account details,
So that I can retry verification without contacting support.

**Acceptance Criteria:**

**Given** an Owner's `bank_status = FAILED` and they have a valid `otp_token` from Story 3.2
**When** they call `POST /api/owner/bank/resubmit` with corrected details and the valid `otp_token`
**Then** existing bank details are overwritten (AES-256 encrypted); a new Cashfree Beneficiary is created; `bank_status` reset to `PENDING_VERIFICATION`; `otp_token` consumed; HTTP 202 returned

**Given** `bank_status` is `VERIFIED`
**When** re-submit is attempted
**Then** HTTP 409 returned; no changes made

**Given** `otp_token` is missing or invalid
**When** the request is validated
**Then** HTTP 403 returned; no bank details updated

---

## Epic 4: WhatsApp Notification System

**Goal:** Replace the Epic 2 `NotificationService` no-op stub with a real Interakt implementation, deliver all 6 WhatsApp template types, and track delivery status — so the reminder engine in Epic 5A has a working notification layer to dispatch through.

**FRs:** FR-20, FR-21, FR-22, FR-23, FR-24
**NFRs:** NFR-17 (notification_log for every send attempt)
**Pre-condition:** All 6 Meta WhatsApp templates must be approved before end-to-end testing; submitted Day 1 of Epic 3 sprint
**Meta rule:** Payment links in templates must be CTA buttons, not inline URLs

### Story 4.1: NotificationService — Interakt Adapter & notification_log

As the system,
I want a real Interakt WhatsApp adapter to replace the no-op stub,
So that all subsequent notification stories have a working send layer and every attempt is logged.

**Acceptance Criteria:**

**Given** the `NotificationService` interface from Epic 2 is in place
**When** the Interakt adapter is injected
**Then** `NotificationService.send(template, recipient, params)` calls the Interakt BSP API with the correct pre-approved template name, recipient phone, and template variables

**Given** any WhatsApp send attempt (success or failure)
**When** the Interakt API responds
**Then** a row is inserted into `notification_log` with: `channel = whatsapp`, `template_name`, `recipient_phone`, `status` (sent/failed), `message_id` (from Interakt), `raw_payload` (JSON), `created_at`; log insert must not block or throw on Interakt API failure

**Given** Interakt API returns an error (network failure, invalid template, etc.)
**When** the send fails
**Then** `notification_log.status = failed` with error detail; the calling code receives the failure without crashing; no retry in this story (retries are Epic 5A's BullMQ concern)

**And** Interakt BSP base URL and API key are read from env vars; never hardcoded

### Story 4.2: Tenant Welcome Message on Move-In

As a newly onboarded Tenant,
I want to receive a WhatsApp welcome message when my Owner adds me,
So that I know my rental agreement is registered and how to pay rent.

**Acceptance Criteria:**

**Given** a Tenant is successfully onboarded (Story 2.3 committed)
**When** `NotificationService.sendWelcome(tenant)` is called (previously a stub)
**Then** the `tenant_welcome` template is sent to `tenant.phone` via the Interakt adapter; `notification_log` row created; HTTP response to the Owner's onboarding request is not delayed by the WhatsApp call (fire-and-forget or async)

**Given** the Interakt API call fails for the welcome message
**When** the send fails
**Then** the tenant onboarding API still returns HTTP 201 (welcome message failure must not roll back tenant creation); failure is logged to `notification_log`

**Given** `tenant.phone` is not a valid WhatsApp-reachable number
**When** Interakt returns a delivery error
**Then** `notification_log.status = failed`; no retry; Owner is not notified of the failure in Phase 1

### Story 4.3: WhatsApp Reminder Send

As a Tenant with a pending rent payment,
I want to receive a WhatsApp reminder message,
So that I am reminded to pay before or after the due date.

**Acceptance Criteria:**

**Given** a Reminder Job is processed by BullMQ (Epic 5A) with a `reminder_type` of `3_days_before`, `due_today`, or `overdue`
**When** `NotificationService.sendReminder(tenant, rentRecord, reminderType, paymentLink)` is called
**Then** the matching pre-approved template (`rent_reminder_3days`, `rent_reminder_due_today`, or `rent_overdue`) is sent to `tenant.phone`; the payment link is included as a CTA button (not inline URL, per Meta rules); `notification_log` row created

**Given** the payment link is null or not yet generated
**When** the reminder send is attempted
**Then** the error is logged; the reminder job fails and enters BullMQ dead-letter queue (NFR-18); no partially-formed message sent to tenant

**And** this story defines the `sendReminder` interface and Interakt implementation; BullMQ job dispatch is Epic 5A's concern

### Story 4.4: WhatsApp Payment Receipt to Tenant

As a Tenant who has paid rent,
I want to receive a WhatsApp payment receipt,
So that I have confirmation of my payment.

**Acceptance Criteria:**

**Given** a `PAYMENT_SUCCESS` webhook is processed (Epic 5B) and `rent_records.status` is set to `paid`
**When** `NotificationService.sendReceipt(tenant, rentRecord)` is called
**Then** the `rent_receipt` template is sent to `tenant.phone` with amount paid, month, and property name; `notification_log` row created

**Given** the Interakt send fails
**When** the receipt call returns an error
**Then** failure is logged; the rent record remains `paid` (receipt failure must not affect payment status); no retry in Phase 1

### Story 4.5: Owner Payment Notification

As an Owner,
I want to receive a WhatsApp notification when a Tenant pays,
So that I am immediately informed of successful collections.

**Acceptance Criteria:**

**Given** a `PAYMENT_SUCCESS` webhook is processed (Epic 5B)
**When** `NotificationService.sendOwnerPaymentAlert(owner, tenant, rentRecord)` is called
**Then** the `owner_payment_received` template is sent to `owner.phone` with tenant name, amount, and month; `notification_log` row created

**Given** the Owner has no phone number on record
**When** the notification is attempted
**Then** the send is skipped; `notification_log` row created with `status = skipped`, reason `no_owner_phone`; no error thrown

### Story 4.6: WhatsApp Delivery Status Tracking

As the system,
I want to process WhatsApp delivery status webhooks from Meta via Interakt,
So that `notification_log` reflects actual delivery outcomes.

**Acceptance Criteria:**

**Given** Interakt sends a delivery status webhook to `POST /api/webhooks/whatsapp-status`
**When** the webhook payload contains a `message_id` and a `status` (delivered, read, failed)
**Then** the matching `notification_log` row (by `message_id`) is updated with the new status and `updated_at` timestamp; HTTP 200 returned

**Given** a delivery status webhook arrives for a `message_id` not found in `notification_log`
**When** the lookup fails
**Then** HTTP 200 returned (acknowledge to Interakt); a warning is logged; no error thrown

**Given** the same delivery status event arrives twice
**When** the handler processes the duplicate
**Then** idempotency is preserved — `notification_log` updated to the same state; HTTP 200 returned both times

---

## Epic 5A: Automated Rent Collection Scheduler

**Goal:** Stand up the BullMQ worker on Render.com, auto-generate monthly rent records via cron, run the daily reminder scan, generate Cashfree payment links at reminder time, and allow the Owner to manually trigger overdue reminders.

**FRs:** FR-16, FR-17, FR-18, FR-19
**NFRs:** NFR-8 (BullMQ + Upstash Redis persistence), NFR-10 (reminder dedup — 20-hr window), NFR-18 (dead-letter queue)
**Pre-conditions:** Epic 3 complete (bank VERIFIED); Epic 4 complete (NotificationService.sendReminder implemented)

### Story 5A.1: BullMQ Worker Setup & Upstash Redis Connection

As the system,
I want a persistent BullMQ worker connected to Upstash Redis,
So that all scheduled jobs survive server restarts and are never silently lost.

**Acceptance Criteria:**

**Given** the NestJS worker on Render.com starts
**When** it connects to Upstash Redis (URL + token from env vars)
**Then** the BullMQ connection is established; a health-check log line confirms Redis connectivity at startup

**Given** the Upstash Redis connection is temporarily unavailable
**When** the worker attempts to reconnect
**Then** BullMQ's built-in retry logic attempts reconnection; jobs already in the queue are not lost; a warning is logged per retry attempt

**Given** a job is enqueued and the worker restarts before processing it
**When** the worker comes back online
**Then** the pending job is picked up and processed (Upstash Redis persistence confirmed); no job is silently dropped

**And** `UPSTASH_REDIS_URL` and `UPSTASH_REDIS_TOKEN` are read from env vars; never hardcoded; Upstash free tier limit (10,000 commands/day) is documented in the worker README

### Story 5A.2: Monthly Rent Record Cron

As the system,
I want to automatically create one rent record per active Tenant on the 1st of each month,
So that the Owner never has to manually create records for recurring rent.

**Acceptance Criteria:**

**Given** a BullMQ repeatable job is scheduled for `0 0 1 * *` (00:00 IST on the 1st of each month)
**When** the cron fires
**Then** for every tenant where `active = true` and `move_out_date` is null or in the future, one `rent_records` row is inserted with `status = pending`, `amount_due = tenant.monthly_rent`, and `period_month` set to the current month/year; duplicate records for the same tenant + month are not created

**Given** a tenant was moved out before the 1st
**When** the cron fires
**Then** no rent record is created for that tenant

**Given** the cron job fails mid-run (e.g. DB error on record 7 of 20)
**When** the job retries via BullMQ
**Then** already-inserted records are not duplicated (idempotent insert — unique constraint on `tenant_id + period_month`); only missing records are created on retry

### Story 5A.3: Daily Reminder Cron Scan

As the system,
I want to scan pending rent records every day at 07:00 IST and enqueue reminder jobs for tenants who meet the reminder criteria,
So that reminders are sent automatically without Owner intervention.

**Acceptance Criteria:**

**Given** a BullMQ repeatable job is scheduled for `0 7 * * *` IST (07:00 daily)
**When** the cron fires
**Then** all `rent_records` with `status = pending` are scanned; for each record meeting any criteria below, a Reminder Job is enqueued:
- `due_date - today = 3 days` → `reminder_type = 3_days_before`
- `due_date = today` → `reminder_type = due_today`
- `today - due_date >= 5 days` → `reminder_type = overdue`

**Given** a tenant already received a reminder within the past 20 hours for the same rent record
**When** the scan evaluates that record
**Then** no Reminder Job is enqueued for that tenant (dedup check at enqueue time per NFR-10)

**Given** the daily scan enqueues 50 Reminder Jobs
**When** the BullMQ worker processes the queue
**Then** jobs are processed concurrently up to the worker's concurrency limit; Upstash Redis command budget is not exceeded (≤ 10,000 commands/day on free tier)

### Story 5A.4: Payment Link Generation at Reminder Time

As a Tenant receiving a reminder,
I want the reminder message to include a direct payment link,
So that I can pay rent in one tap without searching for payment details.

**Acceptance Criteria:**

**Given** a Reminder Job is dequeued by the BullMQ worker
**When** the associated `rent_records` row has no existing `payment_link`
**Then** `PaymentService.generatePaymentLink(rentRecord, tenant, beneficiaryId)` is called; the returned URL is stored on `rent_records.payment_link`; `NotificationService.sendReminder` is called with the link

**Given** a Reminder Job is dequeued and `rent_records.payment_link` already exists
**When** the job is processed
**Then** the existing link is reused; `PaymentService.generatePaymentLink` is not called again

**Given** `PaymentService.generatePaymentLink` fails (Cashfree API error)
**When** the job encounters the error
**Then** the Reminder Job fails; BullMQ retries up to 3 times with exponential backoff; after exhausting retries the job lands in the dead-letter queue (NFR-18); no reminder is sent to the tenant

**And** Owner's `bank_status = VERIFIED` is checked before calling `PaymentService`; if not verified the job fails immediately with a logged error

### Story 5A.5: Manual Overdue Reminder Trigger

As an authenticated Owner,
I want to manually trigger an overdue reminder for a specific pending Rent Record,
So that I can nudge a tenant outside the daily cron cycle.

**Acceptance Criteria:**

**Given** an authenticated Owner calls `POST /api/owner/rent-records/:id/remind`
**When** the rent record belongs to the requesting Owner and `status = pending`
**Then** a Reminder Job with `reminder_type = overdue` is enqueued immediately in BullMQ (bypassing the daily cron); HTTP 202 returned

**Given** the rent record already has a reminder sent within the past 20 hours
**When** the manual trigger is requested
**Then** HTTP 429 returned with time-until-next-allowed; no duplicate job enqueued (NFR-10 dedup applies to manual triggers)

**Given** the rent record has `status = paid` or `status = waived`
**When** the manual trigger is requested
**Then** HTTP 409 returned; no job enqueued

---

## Epic 5B: Payment Webhook & Auto-Settlement

**Goal:** Handle all inbound Cashfree payment webhooks — verify signatures, idempotently mark rent records paid, trigger receipt and owner notifications, and handle failures gracefully — completing the automated collection loop.

**FRs:** FR-25, FR-26, FR-27
**NFRs:** NFR-3 (HMAC-SHA256 on raw body), NFR-9 (webhook idempotency), NFR-13 (HTTP 200 within 3 seconds), NFR-14 (no RBI PA license — money flows tenant→owner directly)
**Pre-conditions:** Epic 5A complete (payment links exist); Epic 4 complete (sendReceipt + sendOwnerPaymentAlert implemented)

### Story 5B.1: Cashfree Webhook HMAC Verification

As the system,
I want to verify every inbound Cashfree payment webhook's HMAC-SHA256 signature before processing,
So that only legitimate Cashfree events can trigger payment state changes.

**Acceptance Criteria:**

**Given** Cashfree sends a POST to `/api/webhooks/payment` with a valid `x-webhook-signature` header
**When** the raw body bytes are HMAC-SHA256 verified using the Cashfree webhook secret
**Then** verification passes; the webhook payload is parsed and passed to the event handler; HTTP 200 is returned

**Given** a POST to `/api/webhooks/payment` with a missing or invalid signature
**When** HMAC verification fails
**Then** HTTP 401 returned immediately; no payload parsed; no DB changes made; attempt logged with source IP and timestamp

**Given** a valid webhook arrives
**When** the handler begins processing
**Then** HTTP 200 is returned within 3 seconds regardless of downstream processing time (async processing after immediate acknowledge per NFR-13); raw body bytes are used for HMAC — not the parsed JSON object

### Story 5B.2: Auto-Mark-Paid on PAYMENT_SUCCESS

As the system,
I want to automatically mark a Rent Record as paid when a verified PAYMENT_SUCCESS webhook arrives,
So that the Owner's dashboard updates instantly and both parties receive confirmation.

**Acceptance Criteria:**

**Given** a verified `PAYMENT_SUCCESS` webhook arrives for a known `pg_order_id` mapped to a pending `rent_records` row
**When** the handler processes the event
**Then** `rent_records.status` is set to `paid`, `payment_method = gateway`, `amount_paid = webhook.amount`, `paid_at = webhook.timestamp`; `NotificationService.sendReceipt(tenant, rentRecord)` is called; `NotificationService.sendOwnerPaymentAlert(owner, tenant, rentRecord)` is called; HTTP 200 returned

**Given** the same `PAYMENT_SUCCESS` webhook is received a second time
**When** the handler checks `rent_records.status`
**Then** idempotency preserved — record already `paid`; no duplicate notification sent; HTTP 200 returned (NFR-9)

**Given** a `PAYMENT_SUCCESS` webhook arrives for a `pg_order_id` not found in `rent_records`
**When** the lookup fails
**Then** HTTP 200 returned (acknowledge to Cashfree); warning logged with unknown `pg_order_id`; no DB changes made

**Given** a `PAYMENT_SUCCESS` webhook arrives for a rent record that is already `waived` or `partial`
**When** the handler checks status
**Then** `rent_records.status` is updated to `paid` (gateway payment overrides manual status); `payment_method = gateway`; notifications sent; HTTP 200 returned

### Story 5B.3: PAYMENT_FAILED Handling

As the system,
I want to handle PAYMENT_FAILED webhooks gracefully,
So that failed payment attempts are logged and the Rent Record stays pending for retry.

**Acceptance Criteria:**

**Given** a verified `PAYMENT_FAILED` webhook arrives for a known `pg_order_id`
**When** the handler processes the event
**Then** `rent_records.status` remains `pending`; the failure event is logged to `notification_log` with `channel = system`, `type = payment_failed`, raw payload; HTTP 200 returned

**Given** the Owner has a phone number on record
**When** a `PAYMENT_FAILED` event is processed
**Then** failure is logged only (Phase 1: no WhatsApp send for failed payments — template not in the 6 approved); no tenant notification sent

**Given** the same `PAYMENT_FAILED` webhook is received twice
**When** the handler processes the duplicate
**Then** idempotency preserved — `rent_records.status` remains `pending`; duplicate log entry not created; HTTP 200 returned

---

## Epic 6: Live Collection Dashboard

**Goal:** Give the Owner a real-time view of their entire rent portfolio — live status updates as payments arrive, room-level occupancy, and historical month navigation — so they can see the state of their collections without refreshing.

**FRs:** FR-28, FR-29, FR-30
**NFRs:** NFR-11 (initial load ≤ 2s P95), NFR-12 (Realtime update ≤ 1s P95 of DB write)
**Pre-condition:** Verify all RLS policies annotated for Realtime compatibility (flagged in Epics 1 & 2) before starting this epic

### Story 6.1: Live Rent Status Board with Supabase Realtime

As an authenticated Owner,
I want to see all Rent Records for the current month with live status updates,
So that I know which tenants have paid and which are still pending without refreshing the page.

**Acceptance Criteria:**

**Given** an authenticated Owner opens the dashboard
**When** the page loads
**Then** all `rent_records` for the current month are displayed with tenant name, amount due, due date, and current status (`pending`, `paid`, `waived`, `partial`); initial load completes in ≤ 2 seconds P95 for up to 50 tenants (NFR-11)

**Given** a `PAYMENT_SUCCESS` webhook is processed and `rent_records.status` is updated to `paid` in the DB
**When** Supabase Realtime fires a `postgres_changes` event on the `rent_records` table
**Then** the affected row's status updates from `pending` to `paid` within 1 second P95 (NFR-12) without a page reload; a visual indicator briefly marks the updated row

**Given** the Supabase Realtime subscription drops (network interruption)
**When** the connection is restored
**Then** the dashboard re-subscribes automatically and re-fetches current state to reconcile any missed events; no stale data is displayed

**And** the Realtime subscription is scoped to the authenticated Owner's records only (RLS enforced at subscription level); other Owners' records are never sent to the client

### Story 6.2: Occupancy Grid

As an authenticated Owner,
I want to see a room-level occupancy overview,
So that I can quickly identify vacant rooms and the current rent status of each occupied room.

**Acceptance Criteria:**

**Given** an authenticated Owner views the occupancy grid
**When** the page loads
**Then** each Property is shown with its Rooms; each Room displays: room name/number, current tenant name (or "Vacant"), and current month's rent status; vacant rooms are visually distinct from occupied ones

**Given** a Room has an active Tenant
**When** the grid renders
**Then** the tenant's current month `rent_records.status` is shown; if no rent record exists for the current month the room shows "No record"

**Given** a Room has been soft-deleted
**When** the grid renders
**Then** the deleted room does not appear in the occupancy grid

### Story 6.3: Month Navigation & Historical View

As an authenticated Owner,
I want to navigate to previous months and review historical Rent Records,
So that I can audit past collections and resolve disputes.

**Acceptance Criteria:**

**Given** an authenticated Owner is on the dashboard showing the current month
**When** they click "Previous Month"
**Then** the Rent Records for the prior month are loaded and displayed; the view is read-only (no status-update actions available for past months)

**Given** the Owner navigates back multiple months
**When** a month is selected
**Then** only `rent_records` rows with a matching `period_month` are shown; the Owner can navigate as far back as records exist

**Given** the Owner is viewing a historical month
**When** Supabase Realtime fires an event
**Then** the Realtime update applies only if it matches the currently displayed month; updates for other months do not alter the current view

**Given** there are no rent records for a selected historical month
**When** the month is loaded
**Then** an empty state message is shown ("No records for this month"); no error is thrown


