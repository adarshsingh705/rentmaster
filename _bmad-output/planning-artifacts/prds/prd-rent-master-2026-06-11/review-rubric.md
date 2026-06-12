# PRD Quality Review — RentMaster Phase 1

## Overall verdict

This is a well-above-average solo-developer PRD: the journeys are concrete, most FRs are testable, the scope is honest, and security constraints are specific rather than generic. The main weaknesses are a handful of under-specified edge-cases that will force discovery during implementation (partial-payment state machine, OTP/penny-drop timing collisions, overdue threshold hard-coding), two unresolved architectural forks that remain in OQs but affect how the whole auth layer is coded, and a thin success-metric story that validates delivery plumbing but not the core thesis ("owners never chase tenants"). None of these are showstoppers, but three findings are high-severity and should be resolved before sprint planning begins.

---

## 1. Decision-readiness — adequate

The document is actionable for sprint-level planning. Trade-offs on auth architecture (Supabase-issued vs custom JWT) and Cashfree product tier are surfaced and indexed. Open Questions are real questions with real owners and real deadlines, not placeholder filler. However, two trade-offs are mentioned once and dropped without a recommended direction: (a) Cashfree Beneficiary Transfer vs Merchant Split Settlement (OQ-1) has a material effect on KYC path and payout timing but no recommended default; (b) Supabase Auth vs custom NestJS JWT (OQ-4) is flagged as "before Sprint 1" but the PRD body proceeds as though Supabase Auth wins — the contingency path ("FR-1 through FR-5 need revision") is not sketched, making the document harder to pivot from if OQ-4 goes the other way.

### Findings

- **high** — OQ-1 lacks a recommended default (§7, OQ-1) — The Cashfree product-tier question determines KYC complexity, payout timing, and fee structure. The PRD acknowledges it but provides no recommendation or elimination criteria. A decision-maker reading this cannot determine how much KYC overhead to plan for. *Fix:* Add a "provisional default" row: e.g., "Provisional: Beneficiary API (simpler KYC, deferred payout). Confirmed before Sprint 2." Explain the disqualifying condition for Split Settlement.

- **medium** — OQ-4 contingency path is described but not sketched (§4.1 assumption block, §7 OQ-4) — The PRD says "FR-1 through FR-5 need revision" if the custom-issuer path is chosen, but does not sketch what that revision looks like. If OQ-4 resolves to custom NestJS issuer mid-sprint, the team has zero pre-work to fall back to. *Fix:* Add a one-paragraph "If custom JWT issuer" note under §4.1 describing which FRs change and how (e.g., `JWTS_SECRET` env var, `jsonwebtoken` verify rather than `supabase.auth.getUser()`). This takes 10 minutes now and saves a sprint derail later.

- **low** — OQ-2 is listed as Critical-path in §7 but the in-body references (FR-18, FR-22, §4.6) note it only as ASSUMPTION (§9 A-3, A-4) — The blast radius is: if templates are not approved by Week 1, the entire reminder engine and receipt flow cannot be integration-tested. The PRD notes this but does not prescribe a fallback strategy (e.g., skip live Interakt in dev, use a mock BSP). *Fix:* Add a one-sentence fallback under §4.5: "If WhatsApp templates are not approved by Day 7, reminder integration tests run against a mock Interakt stub; live WhatsApp testing deferred to the sprint following approval."

---

## 2. Substance over theater — strong

The vision (§1) is specific: it names the mechanism ("BullMQ persistent reminder engine"), the user ("Indian PG owner with 5–50 beds"), and the exact chore eliminated ("monthly WhatsApp-and-follow-up"). It would not drop into a generic SaaS PRD unchanged. Persona journeys are grounded — Priya's 12-bed Koramangala PG, Arjun's ₹8,500 rent, Ravi and Sneha by name on the 5th. NFRs carry numeric thresholds (bcrypt cost ≥ 12, AES-256, 15-minute Access Token, 10-minute OTP TTL, 5 requests/hour rate limit, 2-second dashboard load P95, 1-second Realtime latency P95, 3-second webhook deadline). There is no generic "system should be scalable and reliable" boilerplate.

### Findings

- **low** — NFR-17 (Dead-Letter Visibility) is the one soft NFR (§5.5) — "Bull Board dashboard is a recommended Phase 1 addition but not a hard requirement" leaves the observability story unanchored. For a solo developer who must diagnose stuck jobs in production, knowing where and how to inspect the DLQ matters. *Fix:* Either make Bull Board a Phase 1 deliverable (it's a one-line NestJS module addition) or specify the fallback: "DLQ inspected via `redis-cli LRANGE bull:reminder-jobs:failed 0 -1` or Upstash Redis console." Pick one.

- **low** — §2.2 Non-Users exclusion "owners with fewer than 3 beds" (§2.2) has no corresponding constraint in FRs or onboarding flow — the registration path does not enforce a minimum bed count. If this is an economics rule (not a product rule), it should be labeled as such and the FR consequence should be: no system enforcement, advisory only. *Fix:* Clarify: "No enforcement in Phase 1; this is a go-to-market focus criterion, not a product gate."

---

## 3. Strategic coherence — adequate

The thesis is clear: replace manual WhatsApp rent chasing with a zero-touch automated engine. Every feature cluster serves that thesis. The Phase 1 → Phase 2+ progression is plausible (backend-first, UI second). However, the success metrics (§6) are entirely delivery/infrastructure metrics (reminder delivery rate, webhook success rate, Realtime latency) and do not validate whether the thesis is actually true. The core promise — "owner never has to manually chase a tenant" — has no corresponding metric. A product where 95% of reminders are delivered but 50% of tenants still pay cash does not validate the thesis.

### Findings

- **high** — Success metrics validate the plumbing, not the thesis (§6) — There is no metric that measures whether owners actually stop chasing tenants manually. The conversion metric (≥60% paid via link) comes closest, but 60% is not "never manually chase." The counter-metric "cash/manual payments still logged by owner" is the right instinct but is not elevated to a tracked success metric. *Fix:* Add one outcome metric: "Owner-initiated manual follow-up actions per 100 due rent records ≤ 20 by Day 60" (baseline: estimated 100, pre-platform). Alternatively: "% of paid Rent Records where `payment_method = gateway` ≥ 70% by end of month 2." This makes the thesis falsifiable.

- **medium** — Phase 2+ features are listed in §8 Out of Scope but there is no roadmap signal showing which Phase 1 architectural decisions were made specifically to enable them — The PRD mentions "PaymentService abstraction is ready; switching requires one env var change" (Razorpay) and "mobile app, digital agreements, subscription billing, analytics" as future phases. But neither the auth design (Supabase vs custom JWT) nor the data model discussion mentions which decisions are specifically load-bearing for Phase 2. Downstream architects and developers cannot tell which design choices are "best for now" vs "intentional future enabler." *Fix:* Add a single-sentence "Future-phase enabler" note to 2–3 key FRs (e.g., FR-5: "TenantGuard's `owner_id` injection pattern is the prerequisite for multi-property, multi-staff roles in Phase 3"; FR-16: "Idempotent UPSERT pattern is the prerequisite for bulk import in Phase 2").

---

## 4. Done-ness clarity — strong

The FR format with named "Consequences (testable)" blocks is the PRD's greatest strength. Nearly every FR has at least one concrete, QA-verifiable consequence (HTTP status codes, field formats, boolean state transitions, timing constraints). There are, however, three categories of under-specification that will surface as implementation ambiguity.

### Findings

- **high** — `partial` payment state is referenced in the Glossary and FR-15 but never populated — The Glossary defines `partial` as a valid Rent Record status. FR-15 allows waiving a `partial` record. FR-19 (manual remind) allows triggering on `partial`. But no FR creates a `partial` record, no FR describes when `amount_paid < amount_due` produces `partial` vs `pending`, and the Cashfree webhook section (FR-25) only handles `PAYMENT_SUCCESS` (implying full payment). This is a ghost state: it can be read and acted upon but never written. *Fix:* Either (a) add a FR covering partial payment detection ("If PAYMENT_SUCCESS `amount` < `rent_records.amount_due`, set `status = partial`, `amount_paid = event amount`, queue a follow-up reminder for the remaining balance"); or (b) explicitly exclude `partial` from Phase 1: "Partial payment status is reserved for Phase 2; Cashfree captures the full amount specified in the Payment Link; `partial` never written by system in Phase 1."

- **medium** — Penny-drop webhook handler is referenced but has no FR (§4.2, FR-9) — FR-8 states "`bank_verified` remains `false` until the Cashfree penny-drop confirmation webhook arrives." FR-9 shows `bankVerified: true` only after that webhook. But there is no FR-equivalent for the penny-drop webhook handler itself: no endpoint specified, no HMAC verification consequence, no consequence for a penny-drop failure event vs success event. The penny-drop is critical-path for onboarding, yet its inbound handler is invisible. *Fix:* Add FR-8A (or FR-23-equivalent): "Cashfree VERIFICATION_SUCCESS webhook sets `bank_verified = true` and `pg_beneficiary_id` confirmed on the `owner_payment_settings` row. VERIFICATION_FAILURE sets `bank_verified = false` and updates `bank_verification_status = 'failed'`; Owner's dashboard shows 'Verification Failed.' Handler must pass HMAC-SHA256 verification (same pattern as FR-24)."

- **medium** — Reminder deduplication window (20 hours) is hard-coded in two FRs with no rationale or configurability signal (§4.5, FR-17, FR-19) — The 20-hour window appears in FR-17 (cron deduplication) and FR-19 (manual trigger rate limit). There is no rationale for 20 hours (vs 23 hours to avoid cron drift, or 24 hours for simplicity). For a solo developer, a magic number without rationale becomes a support ticket when an owner complains they can't send a reminder at 8 AM after sending one at 7:15 AM the previous day. *Fix:* Add a one-sentence rationale: "20-hour window chosen to prevent duplicate sends within a calendar day while allowing the 7 AM daily cron to re-trigger for the next day (24h cron − 4h buffer)." Alternatively: expose as an env var `REMINDER_DEDUP_HOURS=20` for future tuning.

- **low** — FR-28 (Occupancy Grid) has behavioral but not data-loading consequences (§4.8) — FR-28 specifies what each tile shows but not how it's fetched (separate API call vs included in the dashboard load of FR-27), what happens when there are 0 tenants in a room (is "Vacant" a calculated state or a DB column?), and what the click-navigation target is (URL format). *Fix:* Add: "Occupancy data included in the same API response as FR-27 (joined query); `is_vacant` is calculated as `tenants.is_active = false OR tenants IS NULL`; click navigates to `/rooms/:id`."

- **low** — FR-6 (Password Reset) does not specify which email provider is authoritative (§4.1) — "via Resend (or Supabase built-in email)" is an OR that means neither is specified. For a Supabase Auth setup, the reset email is typically handled by Supabase; adding Resend creates a competing path. *Fix:* Pick one: "Password reset email sent by Supabase Auth's built-in email handler. Resend is used for OTP-only emails (FR-10A, FR-6). No custom email template for password reset in Phase 1."

---

## 5. Scope honesty — strong

The Out of Scope section (§8) is specific and non-trivial — it names concrete deferred items with phase numbers. The Assumptions Index (§9) is complete, cross-referenced to FRs and OQs, and includes a resolved item (A-7) with a datestamp. The OQ table has owners and deadlines. The PRD is candid about Phase 0 dependency ("assumed complete or parallel").

### Findings

- **medium** — WhatsApp template approval is a critical external dependency that is not in the risk log or a blocking criterion (§4.6, §7 OQ-2) — Template approval (24–72 hours per Meta, per OQ-2) is on the critical path but treated as a question rather than a gate. If approval takes longer (Meta rejects a template and requires resubmission), the entire §4.5 and §4.6 surface cannot be end-to-end tested. There is no mock/stub strategy called out. *Fix:* Add to §8 Out of Scope or a new §10 Risks: "Risk: Meta WhatsApp template rejection. Mitigation: submit templates in Week 1; use Interakt's sandbox environment with pre-approved test templates for CI. If approval delayed >2 weeks, defer live WhatsApp send to Sprint 4 and stub all `NotificationService.send()` calls in tests."

- **low** — "Phase 0 assumed complete or parallel" (§0) is not indexed as an assumption (§9) — If Phase 0 (manual UPI + basic CRUD) is genuinely a parallel workstream, it is a dependency that could delay Phase 1. *Fix:* Add to §9: "A-8 | §0 | Phase 0 (manual UPI + CRUD) is complete or completed in parallel with Phase 1 Sprint 1. If Phase 0 is incomplete when Sprint 2 begins, Tenant Management FRs (FR-11 to FR-15) may be duplicative or conflicting. | Open — confirm Phase 0 status before Sprint 1 kickoff."

---

## 6. Downstream usability — strong

The Glossary (§3) is specific, covers all major entities, and is used consistently throughout the FRs and journeys. FR IDs are not perfectly contiguous (FR-2A is an interpolated ID, FR-23 is referenced as "FR-23-equivalent" in FR-9 before it is defined), but the naming convention is clear and unique. Cross-references are accurate: UJ-1 through UJ-3 are correctly mapped to feature sections. The `notification_log` table is referenced consistently across FR-20, FR-21, FR-22, FR-23, FR-25, FR-26, NFR-16.

### Findings

- **medium** — FR-2A is an interpolated ID that breaks the contiguous numbering contract (§4.1) — FR-2A sits between FR-2 and FR-3. If a story-planning tool or downstream architecture doc references FRs by number, FR-2A is awkward to sort, reference in tests, or surface in a tracking system. *Fix:* Renumber: FR-2A → FR-2B is not much better; cleanest solution is FR-2 (Email Login), FR-3 (Google OAuth Login), FR-4 (Token Refresh), FR-5 (Logout), FR-6 (Route Protection), FR-7 (Password Reset) — shift numbers. If renumbering is too disruptive at this stage, add a mechanical note: "FR-2A is intentionally interpolated to keep Google OAuth adjacent to email login; downstream references should treat FR-2A as a first-class FR."

- **medium** — FR-9 references "FR-23-equivalent in payment settings flow" which does not resolve (§4.2) — The text reads: "`bankVerified: true` only after Cashfree penny-drop webhook has confirmed success (see FR-23-equivalent in payment settings flow)." FR-23 is the WhatsApp webhook handler, not the penny-drop handler. This cross-reference is either wrong or dangling. *Fix:* Remove the cross-reference or replace with the correct reference once the penny-drop handler FR is added (see §4 finding above). The intent should be: "`bankVerified: true` only after the penny-drop confirmation webhook sets it (see FR-8A)."

- **low** — `notification_log` table schema is described via NFR-16 and scattered FR consequences but never consolidated (§5.5) — For a solo developer building the DB migration, the table columns must be inferred by reading six separate FRs. *Fix:* Add to §3 Glossary or as a schema note under §4.6: "`notification_log` columns: `id`, `owner_id`, `tenant_id`, `rent_record_id`, `channel` (whatsapp), `type` (template name), `status` (sent | delivered | read | failed), `message_id` (Interakt), `raw_payload` (JSONB), `created_at`, `updated_at`." This is 5 lines and saves a schema discovery pass.

- **low** — `pg_order_id` is used in FR-25 but not defined in the Glossary (§3) — The term appears in FR-25 ("Lookup uses `pg_order_id`") and implicitly in FR-18 ("`pg_order_id` and `payment_link` stored"). It is a Cashfree concept but is not glossarised. *Fix:* Add to §3: "**pg_order_id** — Cashfree's canonical order reference, generated by PaymentService.createPaymentLink() and stored in `rent_records`. Used as the idempotency key for all Cashfree webhook lookups."

---

## 7. Shape fit — strong

The PRD is correctly shaped for its context: backend-heavy Phase 1, solo developer, launch-level product with real money and real compliance obligations. It does not over-specify UI (appropriate, as UX/architecture docs are downstream). It does not under-specify security (appropriate: bank data, RBI edge, multi-tenant isolation). The journeys are written at the right altitude — specific enough to drive implementation, not so specific they become wireframes. The Out of Scope section correctly defers mobile app, billing, agreements, and analytics to later phases.

### Findings

- **medium** — The PRD is backend-heavy but the dashboard FRs (FR-27, FR-28, FR-29) are frontend-facing with no API contract specified (§4.8) — For a solo developer who will also build the frontend, the lack of API response shapes for the dashboard endpoints means the backend and frontend will be designed simultaneously with no interface contract. This is acceptable for a co-located solo developer but creates risk at the architecture-doc handoff. *Fix:* Add response shape sketches for FR-27 (`GET /api/owner/dashboard/current-month` → `{ summary: {...}, records: [...] }`) to anchor the architecture doc. Three lines, prevents an interface ambiguity.

- **low** — Phase 1 is described as "intentionally backend-heavy" (§1) but there is no explicit statement about what the minimum viable frontend is (§1) — For sprint planning purposes, "backend-heavy" could mean: (a) backend only, test via Postman; (b) backend + minimal React dashboard with no design polish; (c) backend + designed dashboard. The journeys imply (c) but the scope statement implies (a) or (b). *Fix:* Add one sentence to §1 or §0: "Phase 1 frontend scope: a functional Next.js/React dashboard sufficient to execute UJ-1 through UJ-3; visual design polish deferred to Phase 2 after user feedback."

---

## Mechanical notes

1. **FR-9 dangling cross-reference**: "see FR-23-equivalent in payment settings flow" — FR-23 is the WhatsApp webhook handler. The penny-drop confirmation handler has no FR. This reference is broken.

2. **FR-2A interpolated ID**: Not contiguous with FR-2 and FR-3. If sprint tracking tools are used, this will cause sorting/filtering issues.

3. **`pg_order_id` not in Glossary**: Used in FR-18 and FR-25; not defined in §3.

4. **NFR numbering gap**: NFRs are numbered NFR-1 through NFR-17 in §5, but NFR-18 appears in §5.1 ("NFR-18: Bank Change OTP Security") — placed out of sequence within the Security subsection (after NFR-6). Renumber to NFR-7 to keep the Security block contiguous, or note it is intentionally appended.

5. **Email provider ambiguity**: FR-6 says "Resend (or Supabase built-in email)"; FR-10A Step 1 says "via Resend" unconditionally. These two FRs imply different authoritative email senders. Unify.

6. **`partial` status in `rent_records.status` enum**: Glossary lists `pending → paid | partial | waived` as the valid status transition, but no FR in the document can produce a `partial` state. This is a Glossary-to-FR gap.

7. **FR-26 ASSUMPTION note**: "Cashfree reuses the same order for retries; confirm with Cashfree docs whether a new order must be created after failure." This is already indexed as A-5/OQ-6, which is good. No additional fix needed.
