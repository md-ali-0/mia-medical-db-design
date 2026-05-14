# MIA Medical Italia — Full Stack Documentation

Complete PostgreSQL database schema, NestJS backend development guide, and UI/UX design system for the MIA Medical Italia platform.

## Overview

### Database
- **54 tables** across **17 modules**
- PostgreSQL 16+ with PostGIS, pg_trgm, pgcrypto
- Multi-tenant (branch-per-tenant) with Row-Level Security
- Italian fiscal compliance (SDI electronic invoicing)
- Full SQL DDL with triggers, indexes, and constraints

### Backend (NestJS Monolithic)
- **17 NestJS modules** with complete architecture guide
- TypeORM + PostgreSQL + Redis + BullMQ
- JWT + SPID/CIE authentication, RBAC, multi-tenancy
- Full `.env.example` with 80+ environment variables
- Docker Compose for local dev (PG + Redis + MinIO)

### UI/UX
- **Design System** — colors, typography, spacing, components, accessibility standards
- **11 Interactive Wireframes** — public site + admin dashboard
- Elderly-friendly design (WCAG AA/AAA, large touch targets, high contrast)
- 🔗 **[Live Preview](https://wireframes-sshizxks.devinapps.com)**

## Modules

| # | Module | Tables |
|---|--------|--------|
| 1 | Branches & Multi-Tenancy | 2 |
| 2 | Users & Authentication (SPID, CIE, email) | 6 |
| 3 | RBAC (Role-Based Access Control) | 4 |
| 4 | Product Catalogue | 8 |
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
- [`database-design.md`](./database-design.md) — Full database design document (54 tables, SQL DDL, indexes, triggers, ER diagram, design decisions)

### Backend
- [`backend-guide.md`](./backend-guide.md) — NestJS monolithic backend development guide (architecture, modules, folder structure, auth, multi-tenancy, queues, deployment)
- [`.env.example`](./.env.example) — Environment variables template (80+ variables, fully documented)

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

## Quick Start (Backend)

```bash
# 1. Clone and install
git clone https://github.com/md-ali-0/mia-medical-db-design.git
cd mia-medical-db-design

# 2. Copy env template
cp .env.example .env

# 3. Start infrastructure (PostgreSQL + Redis + MinIO)
docker-compose up -d

# 4. Install NestJS dependencies
npm install

# 5. Run migrations & seed
npm run migration:run
npm run seed

# 6. Start dev server
npm run start:dev
# → http://localhost:3000
# → Swagger: http://localhost:3000/api/docs
```

## License

Private — MIA Medical Italia
