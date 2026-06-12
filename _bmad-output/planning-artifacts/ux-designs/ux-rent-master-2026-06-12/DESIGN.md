---
status: draft
created: 2026-06-12
updated: 2026-06-12
project: rent-master
sources:
  - _bmad-output/planning-artifacts/prds/prd-rent-master-2026-06-11/prd.md
  - _bmad-output/planning-artifacts/epics.md
colors:
  background:        "#060B16"
  surface-1:         "#0D1526"
  surface-2:         "#131F35"
  border:            "#1C2D47"
  border-soft:       "#162038"
  text-primary:      "#F1F5F9"
  text-secondary:    "#7E96B5"
  text-tertiary:     "#3D5272"
  action:            "#F97316"
  action-dim:        "rgba(249,115,22,0.15)"
  link:              "#2563EB"
  paid:              "#34D399"
  paid-bg:           "rgba(16,185,129,0.12)"
  pending:           "#FBBF24"
  pending-bg:        "rgba(245,158,11,0.10)"
  late:              "#F87171"
  late-bg:           "rgba(239,68,68,0.10)"
typography:
  font-family:       "'Inter', -apple-system, BlinkMacSystemFont, sans-serif"
  display:           { size: "28px", weight: 800, tracking: "-0.02em", leading: "1.1" }
  heading:           { size: "17px", weight: 700, tracking: "-0.01em", leading: "1.3" }
  subheading:        { size: "15px", weight: 600, tracking: "0",       leading: "1.4" }
  body:              { size: "13px", weight: 400, tracking: "0",       leading: "1.55" }
  label:             { size: "12px", weight: 600, tracking: "0.01em",  leading: "1" }
  caption:           { size: "11px", weight: 500, tracking: "0.06em",  leading: "1.4" }
  overline:          { size: "11px", weight: 700, tracking: "0.10em",  leading: "1",  transform: "uppercase" }
rounded:
  card:   "12px"
  chip:   "4px"
  button: "8px"
  avatar: "50%"
  toast:  "12px"
  input:  "8px"
spacing:
  xs:   "4px"
  sm:   "8px"
  md:   "12px"
  lg:   "16px"
  xl:   "20px"
  "2xl": "24px"
  "3xl": "32px"
components:
  shadcn-base:    true
  shadcn-version: "latest"
  override-mode:  "CSS variable deep override — do not use shadcn default theme"
---

## Brand & Style

**Product:** RentMaster — automated rent collection SaaS for Indian PG and residential property owners.

**Personality:** Precise. Calm. Trustworthy. The owner is often not tech-savvy; the UI must feel like a capable professional assistant — not a startup toy. Think Razorpay Dashboard, not a consumer fintech app.

**What it is not:** Playful. Celebratory. Neon-heavy. The owner handles real money and real people. Every design choice earns its place by reducing anxiety, not adding excitement.

**Single accent:** Saffron (`#F97316`) is the only chromatic accent. It marks the one action worth taking right now. If two things are saffron at the same time, one of them is wrong.

**Icon system:** Lucide — stroke weight 2, size 18px in nav, 16px inline. Never use emoji as UI elements.

**Motion philosophy:** Motion communicates state change. It does not celebrate. The magic moment animation (row glow → chip swap → amount green → counter tick → toast) is the maximum expressiveness the product earns. All other transitions are functional — 150–250ms ease.

---

## Colors

### Palette

| Token | Hex | Role |
|---|---|---|
| `background` | `#060B16` | App background — only this layer |
| `surface-1` | `#0D1526` | Cards, panels, nav backgrounds |
| `surface-2` | `#131F35` | Input fields, nested surfaces, hero stats |
| `border` | `#1C2D47` | All borders, dividers, outlines |
| `border-soft` | `#162038` | Row dividers within cards |
| `text-primary` | `#F1F5F9` | Headings, amounts, names |
| `text-secondary` | `#7E96B5` | Labels, sub-text, descriptions |
| `text-tertiary` | `#3D5272` | Overlines, hints, placeholders |
| `action` | `#F97316` | Primary CTA, active nav, progress fill, % highlights |
| `action-dim` | `rgba(249,115,22,0.15)` | Action hover state backgrounds |
| `link` | `#2563EB` | Secondary actions, "View all" links |
| `paid` | `#34D399` | Paid status text + magic moment amount |
| `paid-bg` | `rgba(16,185,129,0.12)` | Paid chip background |
| `pending` | `#FBBF24` | Pending status text |
| `pending-bg` | `rgba(245,158,11,0.10)` | Pending chip background |
| `late` | `#F87171` | Late status text |
| `late-bg` | `rgba(239,68,68,0.10)` | Late chip background |

### Rules

- `action` appears in exactly one semantic role per screen: the primary CTA button, the active nav item, or the progress percentage — never all three competing for attention simultaneously.
- Status colors (paid/pending/late) are reserved for status chips and the magic moment amount transition. They never appear as decorative accents.
- No color gradients on interactive surfaces. The single permitted gradient: the hero card may use a very subtle `surface-2 → surface-1` radial at 12% opacity max to give the stats block depth.
- Dark backgrounds use `border` outlines to separate surfaces — never drop shadows.

### shadcn CSS Variable Map

```css
:root {
  --background:          0 0% 2%;          /* #060B16 */
  --foreground:          214 87% 97%;      /* #F1F5F9 */
  --card:                216 44% 11%;      /* #0D1526 */
  --card-foreground:     214 87% 97%;
  --popover:             216 44% 11%;
  --popover-foreground:  214 87% 97%;
  --primary:             24 95% 54%;       /* #F97316 */
  --primary-foreground:  0 0% 100%;
  --secondary:           216 35% 14%;      /* #131F35 */
  --secondary-foreground:214 30% 72%;      /* #7E96B5 */
  --muted:               216 35% 14%;
  --muted-foreground:    216 25% 50%;      /* #3D5272 */
  --accent:              24 95% 54%;
  --accent-foreground:   0 0% 100%;
  --destructive:         0 84% 72%;        /* #F87171 */
  --destructive-foreground: 0 0% 100%;
  --border:              215 44% 19%;      /* #1C2D47 */
  --input:               215 44% 19%;
  --ring:                24 95% 54%;
  --radius:              0.75rem;          /* 12px */
}
```

---

## Typography

**Font:** Inter. Loaded via `next/font/google` — no CDN round-trip.

### Scale

| Name | Size | Weight | Tracking | Use |
|---|---|---|---|---|
| Display | 28px | 800 | −0.02em | Hero amount (₹X,XX,XXX) |
| Heading | 17px | 700 | −0.01em | Page titles, logo |
| Subheading | 15px | 600 | 0 | Section titles, card headings |
| Body | 13px | 400 | 0 | Tenant names, descriptions, body text |
| Label | 12px | 600 | +0.01em | Form labels, card sub-titles |
| Caption | 11px | 500 | +0.06em | Sub-text, tenant room/bed line |
| Overline | 11px | 700 | +0.10em uppercase | Section overlines, stat labels |

### Rules

- Rupee symbol (`₹`) always renders at the step below the numeral weight: if amount is Display/800, the ₹ is Heading/600 `text-secondary`. This creates the visual hierarchy that reads "amount first."
- Month/year context ("June 2026 · Total Rent") is Overline — `text-tertiary`.
- Status in a chip: Label weight (600), no tracking override.
- Never use text-primary (`#F1F5F9`) below 11px — unreadable on mobile at night.

---

## Layout & Spacing

**Base unit:** 4px. All spacing is multiples of 4.

### Mobile-first column grid

- Mobile (< 640px): single column, 16px horizontal padding
- Tablet (640–1024px): two-column content area, 24px padding
- Desktop (> 1024px): max-width 1280px centered, three-column dashboard

### Card anatomy

```
┌─ Card (border: 1px solid border, radius: 12px, bg: surface-1) ─┐
│  Header: 14px 16px  [Card Title 600]     [Action link]          │
│  ─── border-soft ───────────────────────────────────────────    │
│  Row:  12px 0px vertical  (10px on first/last to avoid edge)    │
│  ─── border-soft ───────────────────────────────────────────    │
│  ...                                                             │
└─────────────────────────────────────────────────────────────────┘
```

- Card padding: `xl` (20px) on mobile, `2xl` (24px) on desktop
- Card gap (between cards): `md` (12px) on mobile, `lg` (16px) on desktop
- Hero stat block uses vertical dividers (`border`), not cards within cards

### Touch targets

Minimum 44×44px on all interactive elements. Nav items: 48px touch height. Chips are read-only (not tappable in list view) — no touch size requirement.

---

## Elevation & Depth

**Rule:** Elevation is expressed through border contrast, not shadow.

| Layer | Expression |
|---|---|
| Page background | `background` — no border |
| Surface (card, nav) | `surface-1` + `1px border` |
| Nested surface (input, hero stat area) | `surface-2` + `1px border` |
| Floating (toast, dropdown, modal) | `surface-2` + `1px border` + `box-shadow: 0 8px 24px rgba(0,0,0,0.4)` |

The toast is the only element that uses a drop shadow, because it floats above all other surfaces and must be unmistakably separate.

**Magic moment exception:** During the row-glow animation, a `0 0 12px rgba(52,211,153,0.08)` glow box-shadow is applied to the animated row's pseudo-border element. This is a one-time state transition effect, not a structural elevation choice.

---

## Shapes

- Cards, inputs, modals, toasts: `12px` radius (consistent, no variety)
- Buttons: `8px` radius (visually subordinate to cards)
- Status chips: `4px` radius (tight, data-density appropriate — not pill-shaped)
- Avatars: `50%` circle
- Progress bar: `99px` (fully rounded track and fill)
- Bottom nav container: `0` top corners (flows from phone shell), `16px` bottom corners if card-wrapped

**Rationale:** Tight chip radius (`4px`) is the key anti-pattern signal vs AI-generated UI. Pill chips (`20px+`) read as consumer-casual. `4px` reads as Vercel/Linear — data tool.

---

## Components

All components extend shadcn/ui with the CSS variable override above. Only behavioral and token deviations from shadcn defaults are documented here.

### StatusChip

```
Paid:    bg: paid-bg    · text: paid    · radius: 4px · padding: 2px 8px · font: Label
Pending: bg: pending-bg · text: pending · radius: 4px · padding: 2px 8px · font: Label
Late:    bg: late-bg    · text: late    · radius: 4px · padding: 2px 8px · font: Label
```

Not interactive in list views. Tappable only on tenant detail screen to open payment history.

### TenantRow

```
Height: auto (min 44px touch target enforced by padding)
Layout: [Avatar 34px] [12px gap] [Info flex-1] [Right aligned: Amount + Chip]
Divider: border-soft 1px (omit on last row)
Hover (desktop): background shifts to surface-2 at 60% opacity
```

Avatar: 34px circle. Background and text color are per-tenant — seed from first letter. Never use emoji or placeholder icons.

### HeroStatsCard

```
Background: surface-2 (nested surface, not surface-1)
Stat value: Display scale for amount, Heading/700 for counts
Stat dividers: vertical 1px border (not card separators)
Subtle radial gradient permitted: radial from top-right, action at 8% opacity max
```

### PrimaryButton

```
Background: action (#F97316)
Text: white, 14px, weight 700
Radius: 8px
Padding: 13px 16px
Hover: opacity 0.90
Active: scale(0.98) at 100ms
No box-shadow. No glow. No gradient.
```

### BottomNav

```
Background: surface-1
Top border: 1px border
Active item: action color (text + icon stroke)
Inactive: text-tertiary
Icon: Lucide SVG, 18px, stroke-width 2, stroke="currentColor"
Label: Overline scale but 10px / 500 (slightly softer than overline)
Touch target: flex column, 48px minimum height per item
```

### CollectionProgressBar

```
Track: surface-1, 4px height, radius 99px
Fill: action solid color (no gradient)
Percentage label: action, Label scale, right-aligned
Sub-labels: Caption scale, text-tertiary
```

### Toast (payment notification)

```
Background: surface-2
Border: 1px border
Radius: 12px
Padding: 10px 16px
Indicator: 8px filled circle in paid (#34D399) — no emoji
Text: body scale, text-primary, font-weight 600
Entry: translateY(8px) opacity-0 → translateY(0) opacity-1, 250ms ease
Exit: reverse, 250ms ease
Duration visible: 3000ms
Position: fixed bottom 72px, centered
```

### MagicMoment Animation

Fires when Supabase Realtime pushes `payment.status = paid` for a tenant in the visible list.

```
0ms:   Row border glows (border-color → rgba(52,211,153,0.35) · box-shadow 0 0 12px rgba(52,211,153,0.08))
       Row background fades to rgba(16,185,129,0.06)
180ms: Chip animates out (scale 0.75, opacity 0) · swaps class pending→paid · animates in (scale 1.1 → 1.0)
280ms: Amount color transitions text-primary → paid (#34D399)
420ms: Toast slides in from bottom
600ms: Hero paid count +1 · pending count −1 · total amount increments · progress % recalculates
2000ms: Row glow fades out
3500ms: Toast slides out
```

No confetti. No sound. No haptics API call. The precision of the sequence is the celebration.

---

## Do's and Don'ts

### Do

- Use `border` (1px `#1C2D47`) to separate every surface — not shadows
- Saffron for the one primary CTA on screen and the active nav item only
- Lucide SVG icons at stroke-width 2, never emoji as UI elements
- Status colors only on chips and the magic moment amount — nowhere else
- `4px` chip radius — tight and data-appropriate
- Inter font only — no display fonts, no system font fallback for headings
- Touch targets minimum 44px
- Progress bar fill: solid saffron, no gradient

### Don't

- Don't use `box-shadow` on cards or buttons (toast is the sole exception)
- Don't use emoji anywhere in the UI
- Don't use pill-shaped chips (`border-radius > 8px`)
- Don't add decorative gradients to card backgrounds
- Don't show two saffron elements competing for attention simultaneously
- Don't animate on page load — motion only on state changes triggered by user or realtime events
- Don't use color to decorate — only to communicate state
- Don't apply `action` color to text links; use `link` (#2563EB) for secondary navigation
