# DESIGN_SYSTEM.md — Service Hub ApS Visual Design System

## Product Philosophy

We are the simple alternative to complex systems. Every design decision must reduce cognitive load, not add to it.

### Core Rules
- Every element on screen must justify its existence: *"We show this because the user right now needs to..."*
- Flexibility is invisible until needed. If it cannot be hidden, it should probably be removed.
- Every removed choice increases the user's sense of control. Design for clarity, not options.
- The full workflow must always be visible: what is done, what is missing, where can I go back.
- Data visualizations must support decisions — not just show status.

---

## Design Principles (Gestalt-based)

You are designing interfaces that are simple, structured, and decision-focused.

1. Group related elements using **spacing** — not borders as a first resort.
2. Same function = same design. Be consistent with visual patterns.
3. Only **one primary action** per screen. Clear hierarchy always.
4. Use **contrast** to guide attention. Primary vs. secondary must be immediately obvious.
5. Align everything. Create a natural reading flow (top-left to bottom-right).
6. **Remove all non-essential elements.** If in doubt, leave it out.
7. Minimal design — but never at the cost of clarity.
8. Animations only to explain relationships or state changes. Never decorative.

Before designing any screen, define:
- The user's primary goal on this screen
- The single key action the user should take

---

## Color Palette

Use CSS variables throughout. Never hardcode hex values in components.

```css
:root {
  /* Neutrals */
  --color-bg:        #ebedf1; /* Bright Grey — default background */
  --color-bg-subtle: #d4d8df; /* Shy Blunt — cards, panels */
  --color-border:    #acadb1; /* Grey Timber Wolf — dividers, borders */
  --color-text-muted:#706f70; /* Smoked Pearl — secondary text, labels */
  --color-text:      #353536; /* Jet Black — primary text */
  --color-ink:       #080808; /* Reversed Grey — headings, emphasis */

  /* Accent / CTA */
  --color-accent:    #708a83; /* Misty Moor — hover states, highlights */
  --color-cta:       #476e66; /* Pond Newt — primary buttons, active states */
}
```

### Tailwind Usage
Map these to Tailwind config tokens (see tailwind.config.js):
- Background: `bg-bg`
- Primary button: `bg-cta hover:bg-accent`
- Primary text: `text-text`
- Muted text: `text-text-muted`
- Headings: `text-ink`
- Borders: `border-border`
- Cards: `border-bg-subtle`

**Never use arbitrary hex values like `bg-[#476e66]` — always use the named tokens.**

---

## Typography

**Font**: Roboto — loaded via Google Fonts.

```html
<link href="https://fonts.googleapis.com/css2?family=Roboto:wght@300;400;500;700&display=swap" rel="stylesheet">
```

| Role         | Size     | Weight | Line Height |
|--------------|----------|--------|-------------|
| H1           | 32px     | 700    | 1.2         |
| H2           | 24px     | 700    | 1.3         |
| H3           | 18px     | 500    | 1.4         |
| Body         | 16px     | 400    | 1.6         |
| Label/Small  | 14px     | 400    | 1.5         |
| Caption      | 12px     | 300    | 1.4         |

---

## Spacing

Base unit: **8px**. All spacing in multiples of 8.

| Token | Value | Tailwind |
|-------|-------|----------|
| xs    | 8px   | `p-2` / `m-2` |
| sm    | 16px  | `p-4` / `m-4` |
| md    | 24px  | `p-6` / `m-6` |
| lg    | 32px  | `p-8` / `m-8` |
| xl    | 48px  | `p-12` / `m-12` |
| 2xl   | 64px  | `p-16` / `m-16` |

**Layout is airy.** Use generous padding. Prefer `p-8` over `p-4` for cards and containers.

---

## Components

### Border Radius

| Element          | Radius   | Tailwind         |
|------------------|----------|------------------|
| Buttons          | 8px      | `rounded-lg`     |
| Cards / Panels   | 12px     | `rounded-xl`     |
| Input fields     | 8px      | `rounded-lg`     |
| Badges / Tags    | 9999px   | `rounded-full`   |
| Modals           | 16px     | `rounded-2xl`    |

### Buttons

Only **filled** buttons as primary style. One primary CTA per screen.

**Rules:**
- Maximum one primary button per screen
- Never place two primary buttons side by side
- Destructive actions use secondary style + confirmation dialog — never a red primary button

### Cards

White background, `rounded-xl`, `p-8`, `shadow-sm`, `border border-bg-subtle`.

### Input Fields

White background, `border-border`, `rounded-lg`, `px-4 py-3`, focus: `border-cta`.

### Status / Progress Indicators

Workflow must always show: completed / missing / where to go back.
- Completed: `bg-cta` circle with ✓
- Active: `bg-ink` circle with number
- Pending: `bg-bg-subtle` circle with number

---

## Interactions & Animation

- Transition duration: **200ms** for hover/focus states
- Use **ease-in-out** for state changes
- Use **ease-out** for elements entering the screen
- Never animate for decoration — only to communicate state or relationship

---

## Layout Rules

- Max content width: `max-w-5xl` (1024px) centered
- Page padding: `px-8 py-12` on desktop, `px-4 py-8` on mobile
- Use CSS Grid or Flexbox — never nested tables
- Section separation: use spacing (`mt-16`) before borders
- Every page must have one clear visual focal point

---

## Email Template

For transactional emails (magic links, invites, notifications):
- Background: `#ebedf1`
- Card: white, 16px radius, `#d4d8df` border, 48px padding
- Brand: "SERVICE HUB APS" in 12px uppercase `#706f70` with letter-spacing 0.08em
- CTA button: `#476e66` background, white text, 8px radius, 14px/28px padding
- Body: Roboto 16px, `#353536`
- Note/footer: 12px 300 weight, separated by 1px `#d4d8df` border-top
- Sender: `Service Hub ApS <noreply@shub.dk>`

---

## What to Avoid

- ❌ Decorative elements without function
- ❌ Multiple competing CTAs on the same screen
- ❌ Borders as the primary way to group elements (use spacing first)
- ❌ Showing all options at once — progressive disclosure instead
- ❌ Animations that don't explain anything
- ❌ Inconsistent component styles (same function = same design)
- ❌ Dense, cramped layouts — this system breathes
- ❌ Hardcoded hex values — always use the defined palette tokens
