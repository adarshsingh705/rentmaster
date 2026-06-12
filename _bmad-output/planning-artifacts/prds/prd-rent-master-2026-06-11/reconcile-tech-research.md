---
reconcile_source: technical-rent-master-platform-architecture-research-2026-06-11.md
reconcile_target: prd.md
date: 2026-06-11
---

# PRD Reconciliation: Technical Architecture Research vs PRD

## Scope

Gaps in Phase 1 scope: content present in the technical research that is relevant to Phase 1 but not reflected in the PRD — missing requirements, uncovered risk factors, important constraints, or user-facing behaviours the research describes that the PRD silently omits.

Excluded from this report: pure implementation detail (code samples, SDK calls), infrastructure choices already locked in (Vercel/Render/Supabase selections), Phase 2+ items (digital agreements, tenant app, AI insights, multi-city), and anything the PRD covers even if worded differently.

---

## Gap 1 — WhatsApp Template Pre-Approval as an Explicit Phase 1 Dependency

**Research finding (Section 6 + Section 10, Recommendation 3 + Section 11 roadmap):**
The research identifies WhatsApp template approval as **the critical path** for Phase 1 and states explicitly: "Submit templates in Week 1; approval takes 24–72 hours; rejections require revision. Templates are the critical path." The research lists 6 required templates: `rent_reminder_3days`, `rent_reminder_due_today`, `rent_receipt`, `rent_overdue`, `tenant_welcome`, `agreement_signed`. It also specifies template design rules (no URLs in body — links must be buttons; no promotional language; purely transactional).

**PRD coverage:**
OQ-2 asks about template names and content, and notes templates must be submitted in Week 1. However:
- The template `tenant_welcome` (new tenant onboarding message) is not listed in the PRD at all. FR-13 (Tenant Onboarding) creates a tenant but there is no corresponding FR for sending a welcome WhatsApp message to the new tenant.
- The template design constraint (no URLs in template body — payment links must be placed as CTA buttons, not inline text) is not reflected anywhere in the PRD's FR-20 or FR-18 requirements. This constraint affects how payment links appear to tenants and has UX implications.
- There is no functional requirement covering the tenant welcome message. If it is intentionally out of scope it should be listed in Section 8.

**Recommended PRD action:**
1. Add an FR or a clarifying note in FR-13/FR-20 addressing whether a welcome WhatsApp is sent when a tenant is onboarded. If yes, it needs a template and must be in scope; if no, add to Section 8 out-of-scope.
2. Add to NFR or FR-20 the constraint that payment links must be delivered as WhatsApp template CTA buttons (not inline URL text in the body), because this affects template design submitted to Meta.

---

## Gap 2 — Penny-Drop Confirmation via Cashfree Webhook (Distinct from Payment Webhook)

**Research finding (Section 5 — Payment Integration):**
The research describes the bank verification flow as: Cashfree sends a penny-drop confirmation via webhook after the transfer completes. The DB field `bank_verified` is set `true` only after this webhook arrives.

**PRD coverage:**
FR-8 and FR-9 describe penny-drop initiation and the `bank_verified` flag. UJ-1 says "within 10 minutes, penny-drop completes → dashboard updates to 'Bank Verified ✓'." However, there is no FR covering:
- The inbound webhook route that receives Cashfree's penny-drop confirmation.
- How `bank_verified` transitions from `false` to `true` — specifically, whether it is via a Cashfree webhook POST, a polling call, or manual admin action.
- What the Realtime dashboard update looks like when penny-drop status changes (UJ-1 describes the end state but no FR defines the mechanism).

The existing payment webhook FR-24/FR-25 handles `PAYMENT_SUCCESS` events for rent records only. The penny-drop confirmation is a separate Cashfree event type that is not addressed by any FR.

**Recommended PRD action:**
Add an FR (or extend FR-8/FR-9) that covers: (a) the Cashfree penny-drop confirmation event arriving at `/api/webhooks/payment`, (b) HMAC verification applies to this event too, (c) `bank_verified` set `true` on confirmation, and (d) the Supabase Realtime event that triggers the dashboard "Bank Verified ✓" update in UJ-1.

---

## Gap 3 — RLS Misconfiguration as a Critical Risk with No PRD Mitigation Requirement

**Research finding (Section 11 — Technical Risk Register):**
The research rates RLS misconfiguration as **Medium probability / Critical impact** and specifies the mitigation: "Write automated RLS tests: tenant A cannot read tenant B's data." This is explicitly called out as a production risk, not just an implementation note.

**PRD coverage:**
NFR-2 (Multi-Tenant Isolation Dual Layer) defines the security model correctly. However, there is no testability requirement — no NFR or FR states that cross-owner data isolation must be verified by automated tests. The PRD defines the security requirement but is silent on the verification obligation.

For a platform handling financial records and personal data, the absence of a stated testing requirement for the most critical security property is a gap.

**Recommended PRD action:**
Add to NFR-2 (or as a new NFR) a testability consequence: "Automated tests must verify that an authenticated Owner A cannot retrieve, update, or delete data belonging to Owner B via any API route or Supabase Realtime subscription. At minimum one cross-owner isolation test per resource type (properties, rooms, tenants, rent_records, owner_payment_settings)."

---

## Gap 4 — Supabase `service_role` Key Misuse Prevention

**Research finding (Section 10, Recommendation 5):**
The research flags this as a named risk: "Never use `service_role` in frontend code. Only use it in server-side API routes or Render.com workers. A leaked service role key gives anyone full database access." This is treated as a non-negotiable constraint, not just a preference.

**PRD coverage:**
NFR-4 (Secrets in Environment Variables) says API keys must not be hardcoded. NFR-2 covers RLS enforcement. However, neither NFR explicitly distinguishes between the Supabase `anon` key (safe for client-side use, RLS enforced) and the `service_role` key (bypasses all RLS, must never reach the browser or be embedded in client bundles). The distinction is critical because a developer using `service_role` in a Next.js client component would silently bypass all RLS — defeating NFR-2 entirely — with no PRD requirement catching it.

**Recommended PRD action:**
Add to NFR-2 or NFR-4: "The Supabase `service_role` key must only be used in server-side contexts (NestJS API routes, Render.com worker). It must never be referenced in frontend components, client-side SDK initialisation, or any code that reaches the browser. The `anon` key (RLS-enforced) is the only Supabase key permitted in client-side code."

---

## Gap 5 — Aadhaar Document Upload (Supabase Storage) Has No FR

**Research finding (Section 4 — Database Schema):**
The schema includes `tenants.aadhaar_doc_path` and `tenants.selfie_path` referencing Supabase Storage. The schema comment describes these as document storage fields. Section 9 (Security/Compliance) specifies: "Full doc stored encrypted in Supabase Storage" and section-level PDPB 2023 notes list this as a compliance requirement.

**PRD coverage:**
FR-13 (Tenant Onboarding) mentions "optional Aadhaar last-4 digits" and notes "full document upload is a separate operation via Supabase Storage." NFR-14 (PDPB 2023) says "Aadhaar documents stored encrypted in Supabase Storage." However, there is no FR that defines:
- The API endpoint or UI flow for uploading the Aadhaar document or selfie.
- Who initiates the upload (the Owner, not the Tenant, based on the platform model).
- Access control — only the Owner who uploaded the document can retrieve it (Supabase Storage bucket policy).
- Whether document upload is a Phase 1 requirement or explicitly deferred.

If document upload is Phase 1 (required for KYC), it needs an FR. If it is deferred, it must appear in Section 8.

**Recommended PRD action:**
Either: (a) add an FR for Tenant KYC document upload (Owner uploads Aadhaar scan/selfie for a Tenant; stored in a private Supabase Storage bucket scoped to `owner_id`; Storage RLS mirrors DB RLS), or (b) explicitly add "Tenant document/KYC upload" to the Section 8 out-of-scope list.

---

## Gap 6 — Scalability Ceiling Constraints Not Surfaced as Constraints

**Research finding (Section 12 — Scalability Ceiling Analysis):**
The research gives concrete free-tier limits relevant to Phase 1 operation:
- Upstash Redis free tier: 10,000 commands/day ≈ ceiling of ~3,000 active tenants before paid upgrade required.
- Render.com free worker: restarts occasionally — acceptable only for Phase 0; Render Starter ($7/month / ~₹600/month) needed from Month 3 for always-on reliability.

**PRD coverage:**
NFR-7 (BullMQ Persistence) states jobs survive Render restarts. Section 11 mentions Phase 3 timelines. However, the PRD has no explicit constraint or assumption noting that:
- The BullMQ reliability guarantee (NFR-7) requires Render Starter tier (always-on); on the free tier, restarts are frequent enough that "jobs resume" is not equivalent to "jobs fire on time."
- The Upstash Redis free tier imposes a practical tenant ceiling for Phase 1. Exceeding ~3,000 active tenants requires a paid Redis upgrade.

These are operational constraints that affect the Phase 1 reliability SLA (NFR-7) and are not currently flagged.

**Recommended PRD action:**
Add to NFR-7 or a new NFR: "BullMQ persistence guarantees (NFR-7) require the Render.com background worker to run on Starter tier (always-on) by the time the platform has paying customers. Free tier worker restarts may delay — but not lose — scheduled jobs." Optionally note the Upstash Redis free-tier ceiling as an operational assumption.

---

## Summary Table

| # | Gap | Severity | PRD Section to Update |
|---|---|---|---|
| 1 | `tenant_welcome` WhatsApp template not in scope; payment link must be a CTA button (not inline URL) | Medium | FR-13, FR-20, Section 8 |
| 2 | No FR for Cashfree penny-drop confirmation webhook and `bank_verified` state transition mechanism | High | FR-8/FR-9 (or new FR) |
| 3 | No automated RLS cross-owner isolation test requirement | High | NFR-2 |
| 4 | No explicit constraint distinguishing Supabase `anon` vs `service_role` key usage | High | NFR-2 or NFR-4 |
| 5 | Aadhaar/KYC document upload has no FR and is not explicitly deferred in Section 8 | Medium | FR-13 or Section 8 |
| 6 | BullMQ reliability SLA implicitly depends on paid Render tier; Upstash free-tier tenant ceiling not noted | Low | NFR-7 or Assumptions |
