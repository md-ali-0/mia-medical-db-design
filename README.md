# MIA Medical Italia — Database Design

Complete PostgreSQL database schema for the MIA Medical Italia platform.

## Overview

- **50 tables** across **17 modules**
- PostgreSQL 16+ with PostGIS, pg_trgm, pgcrypto
- Multi-tenant (branch-per-tenant) with Row-Level Security
- Italian fiscal compliance (SDI electronic invoicing)
- Full SQL DDL with triggers, indexes, and constraints

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

## File

- [`database-design.md`](./database-design.md) — Full database design document with SQL DDL, indexes, triggers, ER diagram, and design decisions.

## License

Private — MIA Medical Italia
