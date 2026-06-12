---
status: draft
created: 2026-06-12
updated: 2026-06-12
project: rent-master
sources:
  - _bmad-output/planning-artifacts/prds/prd-rent-master-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/epics.md
---

## Foundation

**Form factor:** Mobile-first web. Primary surface is a 390px-wide phone viewport. All layouts and interactions are designed for one-handed thumb use. Desktop (1024px+) is a secondary surface for longer sessions — same IA, wider grid.

**UI system:** shadcn/ui with deep CSS variable override. DESIGN.md is the visual identity reference; this document covers behavioral delta only. Token names reference `{colors.*}`, `{typography.*}`, `{rounded.*}`, `{spacing.*}` from DESIGN.md.

**Target user:** Indian PG and residential property owner. Often non-technical. May manage 10–50 tenants. Checks the app during property rounds on a phone. Does not read instructions — the UI must be self-explanatory.

**Connectivity assumption:** 3G/4G India. All list screens must be usable within 2s on a cold load. Skeleton loading states, not spinners.

---

## Information Architecture

### Navigation model

Bottom tab bar — 4 tabs, always visible on mobile. Desktop promotes to a left sidebar.

| Tab | Icon | Screen |
|---|---|---|
| Home | House | Dashboard |
| Properties | Building | Property list |
| Tenants | Users | Tenant list |
| Settings | Gear | Account + bank |

### Screen hierarchy

```
Dashboard (Home)
├── [tap stat chip] → Filtered tenant list (Paid / Pending / Late)
├── [tap tenant row] → Tenant detail
│   ├── Payment history
│   └── Edit tenant
└── [Add Tenant button] → Tenant onboarding flow

Properties
├── Property list
├── [tap property] → Property detail
│   ├── Room list (room_based) or Bed list (bed_based)
│   ├── [tap room/bed] → Room/Bed detail + occupant
│   └── Add room / Add bed
└── Add Property → Property creation flow

Tenants
├── All-tenant list (searchable, filterable by status)
└── [tap row] → Tenant detail (same as from Dashboard)

Settings (hub — see Settings Hub Screen below)
├── Profile → Profile Edit screen
├── Bank account (add / verify OTP)
└── Notification preferences (inline in Settings hub)
```

### Deep linking

Every entity (property, room, bed, tenant) has a permanent URL. WhatsApp payment link opens tenant detail with payment CTA pre-focused. Bank OTP verification email links directly to the Settings > Bank screen.

---

## Voice and Tone

**Core principle:** The app speaks to Priya (a 42-year-old PG owner in Bengaluru) like a reliable accountant assistant — plain, direct, reassuring.

**Register:** Simple Hindi-English mix is natural for the audience, but the UI copy is English-only in Phase 1. Use short, active sentences. Avoid jargon ("beneficiary", "webhook", "payload" never appear).

**Status language:**

| State | Label | Avoid |
|---|---|---|
| Payment received | Paid | Completed, Success, Settled |
| Reminder sent, awaiting | Pending | Outstanding, Awaiting confirmation |
| Payment overdue | X days late | Delinquent, Overdue, Defaulted |
| Link sent | Reminder sent | Notification dispatched |

**Error messages:** Name the problem, state the fix, one sentence each.
- "Bank account not verified. Check your email for the OTP." ✓
- "An error occurred. Please try again." ✗

**Empty states:** Actionable, not apologetic.
- No tenants: "Add your first tenant to start collecting rent."
- No properties: "Add a property to get started."

**Amounts:** Always `₹X,XX,XXX` format. Never abbreviate (₹1.1L) — owners read exact numbers.

---

## Component Patterns

Visual specifications (colors, radius, typography) live in DESIGN.md. This section covers behavior only.

### TenantRow

- Tapping anywhere on the row navigates to Tenant Detail.
- Status chip is read-only in list views.
- Long-press on mobile (500ms) opens a contextual action sheet: "Send reminder", "Mark paid manually", "Move out".
- On desktop, row hover reveals an action overflow menu (three-dot icon, appears on hover).

### StatusChip

- Three states: Paid / Pending / Late. Driven by computed `payment.status` from the server — never set client-side except during the magic moment optimistic update.
- The magic moment transition (Pending → Paid) is the only chip animation. All other chip renders are instant.

### HeroStatsCard

- Paid / Pending / Late counts are tappable — each navigates to the filtered tenant list.
- Amount display is the current month's total expected rent. It does not change when payments land (the count-up on the magic moment shows collected amount, not expected).
- Month label ("June 2026") is the billing cycle, not calendar month. It advances on the 1st of each cycle.

### CollectionProgressBar

- Width = (paid count / total tenants) × 100%.
- On magic moment: width transition is 600ms ease — matches the hero counter tick step.
- Progress percentage label (`{colors.action}`) updates in sync with the bar width.

### BottomNav

- Active state: icon + label in `{colors.action}`.
- Badge on Tenants tab: count of "Late" tenants. Hidden when 0. Red dot only — no number badge (keeps it calm).
- No badge on Home — dashboard is always the source of truth.

### Forms (Tenant Onboarding, Property Creation)

- One question per screen pattern on mobile (wizard-style, not one long scroll).
- Inline validation on blur, not on keystroke.
- "Next" button disabled until required field is valid — no error toast for empty submits.
- Property type selection (room_based / bed_based) is a two-option card select, not a dropdown. Visual icons distinguish them. This selection is permanent — a confirmation dialog appears with "This cannot be changed later."
- Room capacity input (`1–20`) uses a stepper (+/−) not a free-text field.

### Payment Link Screen (tenant-facing)

- Minimal — outside the main app shell. Shows tenant name, property, amount, month, and a large Cashfree-hosted pay button.
- No nav, no app chrome.
- Status states: unpaid (pay button) / paid (green confirmation, no action).

### Move-out Flow

- Initiated from Tenant Detail → "Move out" action.
- Single confirmation screen: shows tenant name, room/bed, and move-out date (today by default, editable).
- Confirms: "Room will be available for a new tenant after move-out."
- Destructive action — button is `{colors.late}` not saffron. Requires explicit tap (no swipe-to-confirm).

---

## State Patterns

### Loading

- Skeleton screens on first load — match the exact layout of the populated state (same row height, same card structure).
- No full-page spinner. Spinners only inside the element being loaded (e.g., a button after tap).
- Stale-while-revalidate: cached data renders immediately; a subtle pulse animation on the stat numbers while fresh data loads.

### Empty

| Screen | Empty condition | Empty state |
|---|---|---|
| Dashboard | No tenants | Illustration-free card: "Add your first tenant to start collecting rent." + primary CTA |
| Property list | No properties | "Add a property to get started." + primary CTA |
| Tenant list | No tenants match filter | "No tenants match this filter." + "Clear filter" link |
| Payment history | No payments yet | "Payments will appear here once rent is collected." |

### Error

- Network error: persistent banner at top of screen — `{colors.late-bg}` background, `{colors.late}` text, retry button. Dismisses automatically when connection restores.
- Server error on form submit: inline error below the failing field. Never clears user input.
- Webhook failure (PAYMENT_FAILED): dashboard shows a "Payment failed" chip on the row. No toast — owner sees it on next visit.

### Optimistic updates

- Magic moment is the only optimistic update. The chip swaps immediately when the Realtime event fires. If the subsequent API confirm fails (edge case), the chip reverts with a silent re-render (no error toast — the row will self-correct on next refresh).

### Realtime (Supabase)

- Realtime connection indicator: none visible during normal operation. A subtle `{colors.text-tertiary}` dot in the header only when reconnecting (not connected). Disappears when stable.
- Magic moment fires only if the tenant row is visible in the current viewport. Off-screen tenants get a silent counter update; the row animates when scrolled into view within 10s, then settles instantly after that window.

---

## Interaction Primitives

### Tap / Click

Standard. All interactive elements respond at 150ms — no perceived lag.

### Long-press (mobile)

500ms threshold. Opens action sheet for TenantRow. Haptic feedback if available (`navigator.vibrate(10)`).

### Swipe (mobile)

Left-swipe on TenantRow reveals quick actions: "Remind" (amber) and "Move out" (red). Swipe right cancels. This is a secondary discovery surface — all actions also available in detail view.

### Pull-to-refresh

On Dashboard and Tenant list. Refreshes the payment status for the current billing cycle. Refresh indicator uses `{colors.action}` spinner.

### Scroll

Tenant lists: infinite scroll with 20-item pages. A "Load more" button appears after the first 20 items as a fallback (for users who miss the infinite trigger). Dashboard does not paginate — shows max 10 tenants with "View all" link.

### Keyboard (desktop)

- `N` — new tenant (from dashboard/tenant list)
- `/` — focus search on tenant list
- `Esc` — close modal / sheet
- Tab order follows visual reading order — no skip links needed (single-column mobile layout).

---

## Accessibility Floor

Visual contrast and color specifications live in DESIGN.md. This section covers behavioral accessibility.

**Touch targets:** 44×44px minimum on all interactive elements. `{spacing.xl}` (20px) padding on nav items.

**Font floor:** 11px minimum rendered size. The 11px Caption and Overline styles are read-only informational — never used for interactive labels.

**Screen reader:**
- All icons have `aria-label` or are wrapped in elements with visible text.
- Status chips have `role="status"` when updated dynamically (magic moment).
- Bottom nav: `role="navigation"`, active item `aria-current="page"`.
- Amount fields: `aria-label="Rent amount in rupees"`.

**Motion:**
- All animations respect `prefers-reduced-motion: reduce`. When reduced motion is preferred: magic moment skips glow/chip-pop animation and jumps directly to final state (chip = Paid, amount = green, counter updated). Toast still slides in.

**Language:** `lang="en"` on `<html>`. Amounts use `<span aria-label="rupees">₹</span>` pattern.

**Error association:** All form errors use `aria-describedby` linking to the error message element.

---

## Key Flows

### UJ-1: Priya onboards her PG (Epic 1–2)

Priya owns a 3-floor PG in Koramangala. She hears about RentMaster from a friend, opens it on Chrome on her Redmi phone at 9pm after dinner.

1. **Landing / Sign-up** — She taps "Sign in with Google". One tap, no password. She's on the dashboard in under 10 seconds.
2. **Add Property** — Empty dashboard shows "Add a property" CTA. She taps it. Screen asks: property name (types "Koramangala PG"), property type. She sees two cards: "Rooms" (icon: door) and "Beds" (icon: bed grid). She picks Beds — she rents individual beds in shared rooms.
3. **Add Rooms** — Next screen: "How many rooms?" She enters 6. App creates Room 1–6. She can rename them later.
4. **Add Beds per room** — App shows Room 1: "How many beds?" She enters 4. Repeats for each room. Done.
5. **Add first tenant** — Dashboard now shows property with empty beds. CTA: "Add Tenant". She picks Room 2 > Bed A. Enters: name (Rahul), phone (WhatsApp number), monthly rent (₹8,500), move-in date. Taps Save.
6. **Confirmation** — Dashboard updates. Rahul's row shows Pending for this month. She adds the rest of her tenants in the same flow over the next 10 minutes.

**Climax:** Priya sees her dashboard for the first time with all 24 tenants. The hero card shows ₹2,04,000 total expected. She feels in control.

---

### UJ-2: Arjun pays his rent via WhatsApp (Epic 3–5)

Arjun is a 26-year-old software engineer renting Bed B in Room 3. It's the 5th of the month.

1. **Reminder fires** — At 9am, BullMQ triggers a WhatsApp message via Interakt. Arjun gets a message from "RentMaster": "Hi Arjun, your rent of ₹7,000 for June is due. Pay now: [link]"
2. **Arjun taps link** — Opens a simple payment page (not the app, no login required). Shows: his name, property, room, amount, month. One big button: "Pay ₹7,000".
3. **Arjun pays** — Cashfree payment sheet opens. He selects UPI, pays in 20 seconds.
4. **Priya's dashboard updates** — Supabase Realtime fires. Priya is on her dashboard (she opens it to check every morning). Arjun's row glows green, chip swaps from Pending to Paid, amount turns green, hero counter ticks from 12 to 13 paid, progress bar advances from 75% to 81%.
5. **Toast** — A toast slides up: "Priya Nair paid ₹7,000" (persists 3 seconds).

**Climax:** Priya didn't have to call anyone. She didn't send a manual WhatsApp. She just watched money arrive.

---

### UJ-3: Priya reviews the month and moves out a tenant (Epic 2, 6)

It's the 28th of the month. Priya opens RentMaster on her phone during the morning.

1. **Dashboard** — Shows 20 Paid, 3 Pending, 1 Late. Collection progress: 87%.
2. **Late tenant** — She taps the "1" late chip. Filtered list shows only Suresh Kumar: 23 days late. She taps his row.
3. **Tenant detail** — Shows Suresh's payment history: paid on time for 6 months, late this month. She taps "Send reminder" — WhatsApp message fires immediately. A toast confirms: "Reminder sent to Suresh Kumar."
4. **Move-out flow** — Suresh has told her he's leaving. She taps "Move out" on his detail screen. A confirmation screen: "Suresh Kumar · Room 1 · Bed C will be vacated." Move-out date: today (editable). She taps the red "Confirm move-out" button.
5. **Bed freed** — Suresh's record is archived. Bed C in Room 1 is now available (active_tenant_id cleared). The bed shows as empty in the property view.
6. **New tenant** — Priya assigns a new tenant to Bed C immediately. She adds Vijay, ₹7,000/mo, move-in today.

**Climax:** Priya never worried about double-booking a bed or sending Vijay a ghost reminder. The system enforced the move-out before allowing the new assignment.

---

## Responsive & Platform

### Mobile (< 640px) — primary

- Single column layout, 16px horizontal padding
- Bottom tab nav, always visible
- Wizard-style forms (one question per screen)
- Swipe gestures on tenant rows
- Pull-to-refresh on list screens

### Tablet (640–1024px)

- Two-column: nav sidebar left (240px) + content right
- Dashboard: hero card full width, tenant list in 2-column grid
- Forms: single-column centered, max-width 480px

### Desktop (> 1024px)

- Three-column: nav sidebar (240px) + content (flex-1) + detail panel (360px, slides in on row tap)
- Dashboard: hero card + stats in top row, tenant list below
- Bottom nav replaced by persistent left sidebar
- Keyboard shortcuts active
- Hover states on rows (background surface-2 at 60%)
- No touch-specific patterns (long-press, swipe, pull-to-refresh)

### Browser support

Modern evergreen browsers. No IE. Safari 15+ (covers iPhone iOS 15+, dominant in India's aspirational segment). Chrome Android 90+.

---

## Settings Hub Screen

**Route:** `/settings` — 4th tab, gear icon, always accessible.

### Layout

Mobile: full-screen scrollable list inside the bottom-nav shell. Desktop: persistent sidebar shows Settings item as active; content area renders the same settings list, max-width 640px centered.

### Sections (in order)

| Section | Rows | Right element |
|---|---|---|
| Account | Profile (avatar + name + email) | Chevron → Profile Edit screen |
| Financials | Bank Account (last 4 digits + status chip) | Chip + Chevron → Bank Onboarding/Bank Status screen |
| Notifications | Rent Reminders toggle · Payment Alerts toggle | Toggle |
| More | Help & Support · Privacy Policy · Terms of Service | Chevron |
| (no section) | Log out button | — |

### SettingRow component

Each row: 52px min-height (touch target), `16px` horizontal padding. Layout: `[icon 36×36px] [12px gap] [label/sub flex-1] [right element]`. Icon containers are 36×36px rounded-10px, color-coded by semantic role (navy, green, amber, gray, red). Dividers between rows within a card are 1px `{colors.border}`, inset 16px.

No swipe actions on settings rows — all interactions are taps.

### Notification Toggles

- **Rent Reminders** — controls `owner_payment_settings.reminders_enabled`. When off, the daily cron still runs but skips this owner's tenants. On by default on first account setup.
- **Payment Alerts** — controls `owner_payment_settings.owner_alerts_enabled`. When off, `NotificationService.sendOwnerPaymentAlert` is skipped. On by default.

Toggle state is optimistic — UI flips immediately, PATCH request fires in background. On API failure the toggle reverts with a silent re-render (no error toast for this low-stakes preference).

### Log out

Full-width button, red text, white background, red border at 20% opacity. Not a card row — visually separate from the sections above. Tapping shows a brief confirm sheet on mobile ("Log out of RentMaster?") before invalidating the Refresh Token. On desktop, inline confirm instead.

### Bank Account row — states

| `bank_status` | Right element |
|---|---|
| `NOT_REGISTERED` | No chip · text: "Not connected" in `{colors.t3}` · Chevron |
| `PENDING_VERIFICATION` | Pending chip · Chevron |
| `VERIFIED` | Verified (green) chip · Chevron |
| `FAILED` | Failed (red) chip · Chevron |

---

## Profile Edit Screen

**Route:** `/settings/profile` — navigated from Settings Hub > Account row.

### Layout

Back arrow + "Profile" title in nav bar. Scrollable content. No bottom tab nav visible (sub-screen chrome). Save button sticky at the bottom of the scroll region.

### Avatar

80px circle, initials from `full_name` (first letter of each word, max 2). Background `{colors.navy}`, white text. Camera icon overlay (26px circle, bottom-right). Tapping the camera shows a disabled tooltip: "Photo upload coming in a future update." No actual file picker is wired in Phase 1.

### Form — Personal Info card

| Field | Type | Rules |
|---|---|---|
| Full Name | Text input, required | Validates on blur: min 2 chars, max 80 chars. "Please enter your full name." on empty. |
| Email | Read-only input | Pre-filled from Supabase Auth. Visual treatment: gray background (`{colors.bg}`), `{colors.t2}` text. Label suffix: "From your account" in `{colors.t3}`. Not editable in Phase 1 (email change flow is out of scope). |

### Form — Business Info card

| Field | Type | Rules |
|---|---|---|
| GSTIN | Optional text input | Uppercase forced on input. Validates on blur: must be exactly 15 chars matching `^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z]{1}[1-9A-Z]{1}Z[0-9A-Z]{1}$`. Error: "Invalid GSTIN format. Example: 29AABCU9603R1ZX". When blank, no validation runs (optional). |

GSTIN hint text always visible below the field: "15-character GST Identification Number. Example: `29AABCU9603R1ZX`"

### Account Info card (read-only)

Shows: Account type ("Property Owner") · Member since (month + year) · Bank account status chip. This card has no interactions — informational only.

### Save Changes button

- Disabled (opacity 0.45) when no fields differ from saved state.
- Enabled when any editable field has been changed.
- A small amber info banner appears above the button when in the unsaved state: "You have unsaved changes."
- On tap: button shows an inline spinner (navy spinner on navy button, opacity 0.7). On success: button returns to disabled state, banner disappears, brief toast: "Profile saved." On error: button re-enables, inline error below the failing field.

### Navigation guard

If user taps back arrow with unsaved changes, a bottom sheet confirms: "Discard changes?" with "Discard" (red) and "Keep editing" (navy) actions.

---

## Key Flows (continued)

### UJ-4: Priya updates her profile and adds GSTIN (Epic 1, FR-7)

Priya's accountant asks for her GST number to be added to the platform for future invoicing.

1. **Settings** — She taps the Settings tab. Sees her name "Priya Nair" and email in the Account row.
2. **Profile tap** — She taps the Account row. Profile Edit screen slides in.
3. **GSTIN field** — She taps the GSTIN field. Keyboard appears. She types her 15-character GSTIN. The field uppercases her input automatically.
4. **Blur validation** — She taps "Save Changes". The field validates: green outline on success.
5. **Save** — "Profile saved" toast appears. She taps back. Settings hub shows her updated name (if changed).

**Climax:** No form re-submission, no page reload. The GSTIN is stored and available for future Cashfree KYC.

---

### UJ-5: Priya disables rent reminders for a holiday month (Notification toggles)

It's December. Priya's tenants are going home for the holidays and she wants to pause automated reminders.

1. **Settings** — She opens the Settings tab.
2. **Notifications section** — She sees "Rent Reminders" is ON. She taps the toggle.
3. **Optimistic flip** — Toggle immediately shows OFF. PATCH fires in background.
4. **January 1st** — She opens Settings again and taps "Rent Reminders" back ON before the cron runs.

**Climax:** No support ticket. No code change. She controlled the reminder schedule herself in 3 taps.
