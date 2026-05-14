# MIA Medical Italia — Design System & Style Guide

> Accessibility-first design for elderly/disabled users. Italian medical equipment franchise.

---

## Table of Contents

1. [Design Philosophy](#design-philosophy)
2. [Color System](#color-system)
3. [Typography](#typography)
4. [Spacing & Layout](#spacing--layout)
5. [Grid System](#grid-system)
6. [Buttons](#buttons)
7. [Forms & Inputs](#forms--inputs)
8. [Cards](#cards)
9. [Navigation](#navigation)
10. [Icons](#icons)
11. [Feedback & States](#feedback--states)
12. [Accessibility Standards](#accessibility-standards)
13. [Platform-Specific Notes](#platform-specific-notes)
14. [Component Reference](#component-reference)

---

## 1. Design Philosophy

### Core Principles

| Principle | Guideline |
|---|---|
| **Elderly-first** | Every design decision is evaluated through the lens of a 70+ year old user with reduced vision, limited dexterity, and no tech background. |
| **Clarity over aesthetics** | If a design choice makes something prettier but harder to understand, reject it. |
| **Minimal cognitive load** | One action per screen. No multi-step decisions. Linear flows only. |
| **Large touch targets** | Minimum 48×48px (mobile), recommended 56×56px. No targets smaller than 44×44px. |
| **High contrast** | WCAG AAA (7:1) for body text, WCAG AA (4.5:1) minimum for all interactive elements. |
| **Forgiving interactions** | Every destructive action is reversible. Confirmation dialogs for important actions. Double-tap protection on payment buttons. |
| **Italian-first** | All UI copy in Italian. No English jargon. Medical terms explained in simple language. |

### Design Mantra

> **"Se mia nonna non riesce a usarlo, è sbagliato."**
> (If my grandmother can't use it, it's wrong.)

---

## 2. Color System

### Primary Palette

| Role | Hex | Name | Usage |
|---|---|---|---|
| **Primary** | `#0066CC` | Blu Medico | Primary actions, links, active states |
| **Primary Dark** | `#004C99` | Blu Scuro | Hover states, headers |
| **Primary Light** | `#E6F0FF` | Blu Chiaro | Backgrounds, highlights, selected states |

### Secondary Palette

| Role | Hex | Name | Usage |
|---|---|---|---|
| **Secondary** | `#00875A` | Verde Salute | Success, confirmation, "available" badges |
| **Secondary Dark** | `#006644` | Verde Scuro | Hover on success elements |
| **Secondary Light** | `#E6F5EE` | Verde Chiaro | Success backgrounds |

### Accent & Action Colors

| Role | Hex | Name | Usage |
|---|---|---|---|
| **Accent** | `#FF8800` | Arancione Caldo | CTAs, promotions, rental highlights |
| **Accent Dark** | `#CC6D00` | Arancione Scuro | Hover state |
| **Accent Light** | `#FFF3E6` | Arancione Chiaro | Promo backgrounds |
| **Danger** | `#CC3333` | Rosso Attenzione | Errors, destructive actions, overdue |
| **Danger Light** | `#FFE6E6` | Rosso Chiaro | Error backgrounds |
| **Warning** | `#E6A800` | Giallo Avviso | Warnings, pending states |
| **Warning Light** | `#FFF8E6` | Giallo Chiaro | Warning backgrounds |

### Neutral Palette

| Role | Hex | Usage |
|---|---|---|
| **Text Primary** | `#1A1A2E` | Headings, body text |
| **Text Secondary** | `#4A4A68` | Subtitles, secondary info |
| **Text Muted** | `#8888A0` | Hints, placeholders, disabled |
| **Border** | `#D0D0DC` | Input borders, dividers |
| **Border Light** | `#E8E8F0` | Card borders, subtle dividers |
| **Background** | `#F5F5FA` | Page background |
| **Surface** | `#FFFFFF` | Cards, modals, elevated surfaces |

### Admin Dashboard Colors

| Role | Hex | Usage |
|---|---|---|
| **Sidebar BG** | `#1A1A2E` | Admin sidebar |
| **Sidebar Text** | `#C8C8D8` | Nav items |
| **Sidebar Active** | `#0066CC` | Active nav item |
| **Table Stripe** | `#FAFAFE` | Alternating table rows |

### Color Usage Rules

1. **Never use color alone** to convey information — always pair with text, icons, or patterns
2. Body text on white: use `#1A1A2E` (contrast ratio 15.3:1 — WCAG AAA)
3. Interactive elements: minimum contrast ratio 4.5:1 against background
4. Focus indicators: 3px solid `#0066CC` outline with 2px offset
5. Avoid pure red/green combinations (color blindness)

---

## 3. Typography

### Font Family

| Type | Font | Fallback | Reason |
|---|---|---|---|
| **Primary** | `Inter` | `-apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif` | Highly legible, excellent at large sizes, open-source, great Italian character support |
| **Monospace** | `JetBrains Mono` | `'Courier New', monospace` | Admin code/IDs only |

### Type Scale (Public Website & Mobile)

> **All sizes are minimums.** Use `rem` units for scalability.

| Token | Size | Weight | Line Height | Usage |
|---|---|---|---|---|
| `display-1` | 40px / 2.5rem | 700 | 1.2 | Hero headlines |
| `display-2` | 32px / 2rem | 700 | 1.25 | Section titles |
| `heading-1` | 28px / 1.75rem | 600 | 1.3 | Page titles |
| `heading-2` | 24px / 1.5rem | 600 | 1.35 | Card titles, section headers |
| `heading-3` | 20px / 1.25rem | 600 | 1.4 | Sub-section headers |
| `body-large` | 18px / 1.125rem | 400 | 1.7 | **Default body text** |
| `body` | 16px / 1rem | 400 | 1.6 | Secondary body text |
| `body-small` | 14px / 0.875rem | 400 | 1.5 | Captions, metadata (use sparingly) |
| `label` | 16px / 1rem | 500 | 1.4 | Form labels, button text |
| `overline` | 13px / 0.8125rem | 600 | 1.3 | Overline text, category labels (uppercase) |

### Type Scale (Admin Dashboard)

| Token | Size | Weight | Usage |
|---|---|---|---|
| `admin-title` | 24px | 600 | Page titles |
| `admin-subtitle` | 18px | 500 | Section headers |
| `admin-body` | 15px | 400 | Default text |
| `admin-small` | 13px | 400 | Table data, metadata |
| `admin-label` | 13px | 600 | Form labels (uppercase) |

### Typography Rules

1. **Minimum body font: 18px** on public site / mobile app. Never go below 16px anywhere.
2. **No thin/light weights** (100-300) anywhere in the system. Minimum weight is 400.
3. **Line height ≥ 1.5** for body text. Headings can use 1.2-1.4.
4. **Letter spacing:** 0 for body, +0.02em for overline/uppercase text.
5. **Maximum line length:** 70 characters for body text (readability).
6. **No justified text.** Always left-aligned (right-aligned for numbers in tables).
7. **Link underlines:** Always visible. Don't rely on color alone.

---

## 4. Spacing & Layout

### Spacing Scale

Uses a **4px base unit** with a harmonic scale:

| Token | Value | Usage |
|---|---|---|
| `space-1` | 4px | Inline icon gaps |
| `space-2` | 8px | Tight element spacing |
| `space-3` | 12px | Input padding (vertical) |
| `space-4` | 16px | Default element spacing |
| `space-5` | 20px | Input padding (horizontal) |
| `space-6` | 24px | Card padding, section gaps |
| `space-8` | 32px | Between sections |
| `space-10` | 40px | Large section breaks |
| `space-12` | 48px | Page section spacing |
| `space-16` | 64px | Major page divisions |
| `space-20` | 80px | Hero/banner padding |

### Border Radius

| Token | Value | Usage |
|---|---|---|
| `radius-sm` | 6px | Small elements, badges |
| `radius-md` | 10px | Buttons, inputs |
| `radius-lg` | 16px | Cards, modals |
| `radius-xl` | 24px | Feature cards, hero sections |
| `radius-full` | 9999px | Pills, avatars |

### Shadows

| Token | Value | Usage |
|---|---|---|
| `shadow-sm` | `0 1px 3px rgba(0,0,0,0.08)` | Subtle lift (cards at rest) |
| `shadow-md` | `0 4px 12px rgba(0,0,0,0.1)` | Elevated elements (dropdowns) |
| `shadow-lg` | `0 8px 24px rgba(0,0,0,0.12)` | Modals, floating elements |
| `shadow-focus` | `0 0 0 3px rgba(0,102,204,0.4)` | Focus ring |

---

## 5. Grid System

### Public Website

| Breakpoint | Name | Columns | Gutter | Margin |
|---|---|---|---|---|
| 0-599px | **Mobile** | 4 | 16px | 16px |
| 600-959px | **Tablet** | 8 | 24px | 32px |
| 960-1279px | **Desktop** | 12 | 24px | 40px |
| 1280px+ | **Wide** | 12 | 32px | auto (max-width: 1200px) |

### Admin Dashboard

| Area | Width |
|---|---|
| Sidebar (collapsed) | 64px |
| Sidebar (expanded) | 260px |
| Content area | fluid (min: 800px) |
| Max content width | 1400px |

---

## 6. Buttons

### Sizes

| Size | Height | Padding | Font Size | Usage |
|---|---|---|---|---|
| **Large** | 56px | 24px 32px | 18px (600) | **Primary CTAs, checkout, mobile** |
| **Medium** | 48px | 16px 24px | 16px (500) | Secondary actions |
| **Small** | 40px | 12px 16px | 14px (500) | Admin dashboard, compact UI |

### Variants

| Variant | Background | Text | Border | Usage |
|---|---|---|---|---|
| **Primary** | `#0066CC` | `#FFFFFF` | none | Main actions: "Aggiungi al carrello", "Procedi" |
| **Primary Hover** | `#004C99` | `#FFFFFF` | none | — |
| **Secondary** | `#FFFFFF` | `#0066CC` | 2px `#0066CC` | Alternative actions: "Dettagli", "Salva" |
| **Accent** | `#FF8800` | `#FFFFFF` | none | Promotions: "Noleggia ora", "Offerta speciale" |
| **Danger** | `#CC3333` | `#FFFFFF` | none | Destructive: "Elimina", "Annulla ordine" |
| **Ghost** | transparent | `#0066CC` | none | Tertiary: "Indietro", "Annulla" |
| **Disabled** | `#E8E8F0` | `#8888A0` | none | Inactive state |

### Button Rules

1. **Minimum touch target: 48×48px.** Even small buttons have a 48px hit area.
2. **Full-width on mobile** for primary actions.
3. **Max 2 buttons** visible at any time. Avoid decision paralysis.
4. **Loading state:** Show spinner + "Attendere..." text. Disable click.
5. **Icons:** Left side only. 20px icon with 8px gap to text.
6. **No icon-only buttons** on public site (always include text label).

---

## 7. Forms & Inputs

### Input Fields

| Property | Value |
|---|---|
| Height | 56px (public), 44px (admin) |
| Border | 2px solid `#D0D0DC` |
| Border (focus) | 2px solid `#0066CC` + focus shadow |
| Border (error) | 2px solid `#CC3333` |
| Border radius | 10px |
| Font size | 18px (public), 15px (admin) |
| Padding | 12px 20px |
| Background | `#FFFFFF` |
| Placeholder color | `#8888A0` |

### Form Rules

1. **One column layout** for public-facing forms. Never side-by-side inputs on mobile.
2. **Labels always above inputs.** Never floating labels (confusing for elderly).
3. **Label font:** 16px, weight 500, color `#1A1A2E`.
4. **Helper text:** Below input, 14px, color `#4A4A68`.
5. **Error messages:** Below input, 14px, color `#CC3333`, with ⚠ icon.
6. **Required indicator:** Red asterisk `*` after label text.
7. **Autofill-friendly:** Use proper `autocomplete` attributes.
8. **Validation:** Inline validation on blur. Show success ✓ for valid fields.
9. **Select dropdowns:** Use native `<select>` on mobile. Custom dropdown on desktop.
10. **Date pickers:** Native on mobile, calendar widget on desktop. Format: `DD/MM/YYYY` (Italian).

### Checkbox & Radio

| Property | Value |
|---|---|
| Size | 24×24px (public), 20×20px (admin) |
| Border | 2px solid `#D0D0DC` |
| Checked color | `#0066CC` |
| Label spacing | 12px from control |
| Label font | 18px, weight 400 |

---

## 8. Cards

### Product Card

```
┌─────────────────────────────────┐
│         [Product Image]         │  aspect-ratio: 4/3
│         280×210px min           │  object-fit: cover
├─────────────────────────────────┤
│  CATEGORY LABEL (overline)      │  13px, #0066CC, uppercase
│  Product Name                   │  20px, 600 weight
│  Brief description...           │  16px, #4A4A68, 2 lines max
│                                 │
│  ★★★★☆ (4.2)  ·  12 recensioni │  16px
│                                 │
│  €29,90 /giorno                 │  24px, 700 weight, #1A1A2E
│  IVA inclusa                    │  14px, #8888A0
│                                 │
│  [  Aggiungi al carrello  ]     │  Full-width button, 48px
└─────────────────────────────────┘
  Border: 1px #E8E8F0
  Radius: 16px
  Shadow: shadow-sm
  Padding: 0 (image) + 24px (content)
  Hover: shadow-md + translateY(-2px)
```

### Info Card (Store Locator, Dashboard Stats)

```
┌──────────────────────────────────┐
│  🏥  Branch Name                 │
│  Via Roma 123, Roma              │
│  📞 06 1234567                   │
│  🕐 Lun-Ven: 9:00-18:00         │
│  📍 15 min in auto               │
│                                  │
│  [ Indicazioni ]  [ Catalogo ]   │
└──────────────────────────────────┘
  Padding: 24px
  Radius: 16px
  Border: 1px #E8E8F0
```

---

## 9. Navigation

### Public Website — Top Navigation

```
┌──────────────────────────────────────────────────────────────────┐
│  📍 Roma: Via Talamini 44  ·  📞 800 031962  ·  📧 info@...    │  ← Top bar (dark)
├──────────────────────────────────────────────────────────────────┤
│  [LOGO]        🔍 Search        👤 Account    🛒 Cart (2)       │  ← Main header
├──────────────────────────────────────────────────────────────────┤
│  Home    Noleggio ▾    Vendita ▾    Punti Vendita    Blog       │  ← Nav links
└──────────────────────────────────────────────────────────────────┘

Mobile: Hamburger menu with full-screen overlay.
Sticky header on scroll (main header only).
```

### Admin Dashboard — Sidebar Navigation

```
┌────────────────────┐
│  [LOGO]            │
│  ─────────────────  │
│  🏢 Branch: Roma ▾ │  ← Branch switcher
│  ─────────────────  │
│  📊 Dashboard      │
│  📦 Prodotti       │
│    └ Catalogo      │
│    └ Categorie     │
│  🛒 Ordini         │
│  🔄 Noleggi        │
│  📅 Calendario     │
│  📦 Inventario     │
│  🧾 Fatture        │
│  💬 Supporto       │
│  📝 Blog           │
│  ⭐ Recensioni     │
│  ─────────────────  │
│  👥 Utenti         │
│  🔐 Ruoli          │
│  ⚙️ Impostazioni   │
│  ─────────────────  │
│  👤 Mario R.       │
│  Esci              │
└────────────────────┘

Width: 260px expanded / 64px collapsed
BG: #1A1A2E
Active item: #0066CC left border + light bg
```

---

## 10. Icons

### Icon System

| Property | Value |
|---|---|
| Library | Lucide Icons (open-source, consistent, accessible) |
| Sizes | 20px (default), 24px (navigation), 16px (inline) |
| Stroke | 2px |
| Color | Inherit from text color |

### Key Icons Map

| Function | Icon | Label |
|---|---|---|
| Search | `search` | Cerca |
| Cart | `shopping-cart` | Carrello |
| Account | `user` | Account |
| Menu | `menu` | Menu |
| Close | `x` | Chiudi |
| Back | `arrow-left` | Indietro |
| Location | `map-pin` | Posizione |
| Phone | `phone` | Chiama |
| Calendar | `calendar` | Calendario |
| Notification | `bell` | Notifiche |
| Settings | `settings` | Impostazioni |
| Download | `download` | Scarica |
| Upload | `upload` | Carica |
| Edit | `pencil` | Modifica |
| Delete | `trash-2` | Elimina |
| Check | `check` | Fatto |
| Warning | `alert-triangle` | Attenzione |
| Info | `info` | Informazioni |
| Rental | `repeat` | Noleggio |
| Purchase | `shopping-bag` | Acquista |

### Icon Rules

1. **Always pair with text labels** on public site.
2. Admin dashboard may use icon-only for toolbar actions (with tooltip).
3. Decorative icons: `aria-hidden="true"`. Functional icons: proper `aria-label`.

---

## 11. Feedback & States

### Loading States

| Element | Treatment |
|---|---|
| Page load | Skeleton screens (not spinners) |
| Button loading | Spinner inside button + "Attendere..." |
| Data table | Shimmer rows |
| Image loading | Blurred placeholder → sharp image |

### Empty States

| Context | Message | Action |
|---|---|---|
| Empty cart | "Il tuo carrello è vuoto" | "Scopri i prodotti" button |
| No orders | "Non hai ancora effettuato ordini" | "Inizia lo shopping" button |
| No results | "Nessun risultato per [query]" | Suggestions + clear button |

### Toast Notifications

| Type | Background | Icon | Duration |
|---|---|---|---|
| Success | `#E6F5EE` | ✓ green | 4 seconds |
| Error | `#FFE6E6` | ⚠ red | Manual dismiss |
| Warning | `#FFF8E6` | ⚠ amber | 6 seconds |
| Info | `#E6F0FF` | ℹ blue | 4 seconds |

Position: Top-center on mobile, top-right on desktop.
Font: 16px. Min height: 48px. Border-radius: 10px.

### Status Badges

| Status | Background | Text | Context |
|---|---|---|---|
| Attivo | `#E6F5EE` | `#006644` | Active rental |
| In elaborazione | `#E6F0FF` | `#004C99` | Processing order |
| In consegna | `#FFF3E6` | `#CC6D00` | Out for delivery |
| Completato | `#E6F5EE` | `#006644` | Completed order |
| Scaduto | `#FFE6E6` | `#CC3333` | Overdue rental |
| In attesa | `#FFF8E6` | `#E6A800` | Pending |
| Annullato | `#F0F0F5` | `#8888A0` | Cancelled |

Badge: padding `4px 12px`, border-radius `9999px` (pill), font `13px 600`.

---

## 12. Accessibility Standards

### Compliance Target: WCAG 2.1 AA (aiming for AAA where possible)

| Requirement | Implementation |
|---|---|
| **Keyboard navigation** | All interactive elements focusable. Visible focus ring (3px `#0066CC`). Tab order follows visual order. |
| **Screen readers** | Semantic HTML. ARIA labels on all icons/images. Live regions for dynamic content. |
| **Color contrast** | Body text: 7:1 (AAA). Large text: 4.5:1. Interactive: 3:1 minimum. |
| **Text resize** | All text in `rem`. Layout works up to 200% zoom. |
| **Motion** | Respect `prefers-reduced-motion`. No auto-playing videos. Minimal animations. |
| **Touch targets** | Minimum 48×48px. 8px gap between adjacent targets. |
| **Error identification** | Errors described in text (not color alone). Error linked to field. |
| **Time limits** | Cart: 30 min timeout with warning. Checkout: no timeout. Hold timers: visible countdown. |
| **Language** | `lang="it"` on HTML. Switch to `lang="en"` for code/technical terms. |

### Elderly-Specific Accommodations

1. **Font size toggle** in header (A / A+ / A++) — stores preference
2. **High contrast mode** toggle
3. **No hover-dependent functionality** — everything works with click/tap
4. **Breadcrumb navigation** on every page
5. **"Where am I?" indicator** — current step highlighted in multi-step flows
6. **Confirmation before leaving** checkout (unsaved changes)
7. **Phone number** prominently visible on every page for human support
8. **No CAPTCHAs** — use honeypot or invisible reCAPTCHA

---

## 13. Platform-Specific Notes

### Mobile App (Native)

| Guideline | Detail |
|---|---|
| Min touch target | 56×56px (larger than web) |
| Font scale | Follow system font size settings |
| Navigation | Bottom tab bar (max 5 items): Home, Cerca, Noleggi, Notifiche, Account |
| Pull to refresh | On all list screens |
| Haptic feedback | On button presses, successful actions |
| Safe areas | Respect iOS notch / Android status bar |
| Offline mode | Cached catalogue browsable. Banner: "Sei offline" |

### Admin Dashboard

| Guideline | Detail |
|---|---|
| Min viewport | 1024×768px |
| Sidebar | Collapsible (hamburger on < 1200px) |
| Tables | Horizontal scroll on narrow views. Sticky first column. |
| Data density | Compact spacing allowed (admin users are trained) |
| Keyboard shortcuts | Listed in `?` modal |

---

## 14. Component Reference

### Full Component List

| Component | Public | Admin | Mobile |
|---|---|---|---|
| Button (primary/secondary/accent/ghost) | ✓ | ✓ | ✓ |
| Input (text/email/phone/number) | ✓ | ✓ | ✓ |
| Textarea | ✓ | ✓ | ✓ |
| Select / Dropdown | ✓ | ✓ | native |
| Checkbox | ✓ | ✓ | ✓ |
| Radio | ✓ | ✓ | ✓ |
| Toggle / Switch | — | ✓ | ✓ |
| Search Input | ✓ | ✓ | ✓ |
| Product Card | ✓ | — | ✓ |
| Branch Card | ✓ | — | ✓ |
| Status Badge | ✓ | ✓ | ✓ |
| Toast Notification | ✓ | ✓ | ✓ |
| Modal / Dialog | ✓ | ✓ | bottom-sheet |
| Breadcrumb | ✓ | ✓ | — |
| Pagination | ✓ | ✓ | infinite-scroll |
| Tabs | ✓ | ✓ | ✓ |
| Accordion | ✓ | — | ✓ |
| Stepper (checkout) | ✓ | — | ✓ |
| Data Table | — | ✓ | — |
| Date Picker | ✓ | ✓ | native |
| File Upload | — | ✓ | camera |
| Skeleton Loader | ✓ | ✓ | ✓ |
| Avatar | — | ✓ | ✓ |
| Branch Switcher | — | ✓ | — |
| Star Rating | ✓ | ✓ | ✓ |
| Price Display | ✓ | ✓ | ✓ |
| Cart Drawer | ✓ | — | ✓ |

---

## CSS Variables (Design Tokens)

```css
:root {
    /* ─── Colors ─── */
    --color-primary: #0066CC;
    --color-primary-dark: #004C99;
    --color-primary-light: #E6F0FF;
    --color-secondary: #00875A;
    --color-secondary-dark: #006644;
    --color-secondary-light: #E6F5EE;
    --color-accent: #FF8800;
    --color-accent-dark: #CC6D00;
    --color-accent-light: #FFF3E6;
    --color-danger: #CC3333;
    --color-danger-light: #FFE6E6;
    --color-warning: #E6A800;
    --color-warning-light: #FFF8E6;

    --color-text-primary: #1A1A2E;
    --color-text-secondary: #4A4A68;
    --color-text-muted: #8888A0;
    --color-border: #D0D0DC;
    --color-border-light: #E8E8F0;
    --color-bg: #F5F5FA;
    --color-surface: #FFFFFF;

    /* Admin */
    --color-sidebar-bg: #1A1A2E;
    --color-sidebar-text: #C8C8D8;
    --color-sidebar-active: #0066CC;
    --color-table-stripe: #FAFAFE;

    /* ─── Typography ─── */
    --font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    --font-mono: 'JetBrains Mono', 'Courier New', monospace;

    --text-display-1: 700 2.5rem/1.2 var(--font-family);
    --text-display-2: 700 2rem/1.25 var(--font-family);
    --text-h1: 600 1.75rem/1.3 var(--font-family);
    --text-h2: 600 1.5rem/1.35 var(--font-family);
    --text-h3: 600 1.25rem/1.4 var(--font-family);
    --text-body-lg: 400 1.125rem/1.7 var(--font-family);
    --text-body: 400 1rem/1.6 var(--font-family);
    --text-body-sm: 400 0.875rem/1.5 var(--font-family);
    --text-label: 500 1rem/1.4 var(--font-family);
    --text-overline: 600 0.8125rem/1.3 var(--font-family);

    /* ─── Spacing ─── */
    --space-1: 4px;
    --space-2: 8px;
    --space-3: 12px;
    --space-4: 16px;
    --space-5: 20px;
    --space-6: 24px;
    --space-8: 32px;
    --space-10: 40px;
    --space-12: 48px;
    --space-16: 64px;
    --space-20: 80px;

    /* ─── Borders ─── */
    --radius-sm: 6px;
    --radius-md: 10px;
    --radius-lg: 16px;
    --radius-xl: 24px;
    --radius-full: 9999px;

    /* ─── Shadows ─── */
    --shadow-sm: 0 1px 3px rgba(0,0,0,0.08);
    --shadow-md: 0 4px 12px rgba(0,0,0,0.1);
    --shadow-lg: 0 8px 24px rgba(0,0,0,0.12);
    --shadow-focus: 0 0 0 3px rgba(0,102,204,0.4);

    /* ─── Transitions ─── */
    --transition-fast: 150ms ease;
    --transition-normal: 250ms ease;
    --transition-slow: 400ms ease;
}
```

---

*End of Design System v1.0*
