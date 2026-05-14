# MIA Medical Italia — Database Design & UI/UX

Complete PostgreSQL database schema and UI/UX design system for the MIA Medical Italia platform.

## Overview

### Database
- **50 tables** across **17 modules**
- PostgreSQL 16+ with PostGIS, pg_trgm, pgcrypto
- Multi-tenant (branch-per-tenant) with Row-Level Security
- Italian fiscal compliance (SDI electronic invoicing)
- Full SQL DDL with triggers, indexes, and constraints

### UI/UX
- **Design System** — colors, typography, spacing, components, accessibility standards
- **11 Interactive Wireframes** — public site + admin dashboard
- Elderly-friendly design (WCAG AA/AAA, large touch targets, high contrast)
- 🔗 **[Live Preview](https://wireframes-sshizxks.devinapps.com)**

## Modules

| # | Module | Tables |
|---|--------|--------|
| 1 | Branches & Multi-Tenancy | 2 |
| 2 | Users & Authentication (SPID, CIE, email) | 4 |
| 3 | RBAC (Role-Based Access Control) | 4 |
| 4 | Product Catalogue | 7 |
| 5 | Cart | 2 |
| 6 | Orders (Purchase) | 3 |
| 7 | Rentals | 4 |
| 8 | Inventory & Calibration | 2 |
| 9 | Payments & Deposits | 2 |
| 10 | Invoicing & SDI | 3 |
| 11 | Wishlist | 2 |
| 12 | Notifications & Push Tokens | 2 |
| 13 | Support Tickets | 2 |
| 14 | Blog / Magazine | 5 |
| 15 | Reviews & Q&A | 3 |
| 16 | GDPR & Communication Preferences | 2 |
| 17 | Audit Log | 1 |

## Files

### Database
- [`database-design.md`](./database-design.md) — Full database design document with SQL DDL, indexes, triggers, ER diagram, and design decisions.

### UI/UX
- [`design-system.md`](./design-system.md) — Design system & style guide (colors, typography, spacing, components, accessibility)
- [`wireframes/index.html`](./wireframes/index.html) — Interactive wireframe prototypes (11 screens)

#### Wireframe Screens

**Public Website (7 screens):**
1. Homepage — Hero, categories, featured products, trust bar, CTA
2. Catalogo Prodotti — Filters sidebar, product grid, sorting
3. Dettaglio Prodotto — Gallery, rental tiers, pricing, availability
4. Checkout (4-step) — Stepper, form, invoice detection, order summary
5. Trova Punto Vendita — Search, store cards, map
6. Login/Registrazione — SPID, CIE, email/password
7. Area Cliente — Dashboard, active rentals, orders, notifications

**Admin Dashboard (4 screens):**
1. Dashboard Overview — Stats, revenue chart, recent orders, expiring rentals
2. Gestione Prodotti — Table, filters, CRUD actions, pagination
3. Ordini & Noleggi — Tab filtering, date range, order table
4. Ruoli & Permessi — Role cards, permission matrix (RBAC)

## License

Private — MIA Medical Italia
