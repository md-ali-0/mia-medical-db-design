# MIA Medical Italia — NestJS Backend Development Guide

> Monolithic NestJS backend for the MIA Medical Italia platform.
> PostgreSQL 16+ · TypeORM · JWT + SPID/CIE · Multi-tenant · Italian fiscal compliance

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Tech Stack](#tech-stack)
3. [Project Structure](#project-structure)
4. [Module Breakdown](#module-breakdown)
5. [Database & ORM Setup](#database--orm-setup)
6. [Authentication & Authorization](#authentication--authorization)
7. [Multi-Tenancy Implementation](#multi-tenancy-implementation)
8. [API Design](#api-design)
9. [Key Flows](#key-flows)
10. [File Storage](#file-storage)
11. [Email, SMS & Push Notifications](#email-sms--push-notifications)
12. [Invoice & SDI Integration](#invoice--sdi-integration)
13. [Search](#search)
14. [Caching](#caching)
15. [Queue & Background Jobs](#queue--background-jobs)
16. [Error Handling](#error-handling)
17. [Testing Strategy](#testing-strategy)
18. [Deployment & DevOps](#deployment--devops)
19. [Environment Variables Reference](#environment-variables-reference)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENTS                                  │
│   Public Website (Next.js SSR)  ·  Mobile App (React Native)   │
│   Admin Dashboard (Next.js SPA)                                │
└────────────────────────────┬────────────────────────────────────┘
                             │ HTTPS (REST + WebSocket)
┌────────────────────────────▼────────────────────────────────────┐
│                     NestJS Monolith                              │
│                                                                  │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │   Auth   │  │ Products │  │  Orders  │  │ Rentals  │       │
│   │  Module  │  │  Module  │  │  Module  │  │  Module  │       │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │ Invoices │  │Inventory │  │  Branch  │  │   RBAC   │       │
│   │  Module  │  │  Module  │  │  Module  │  │  Module  │       │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│   │Payments  │  │  Blog    │  │ Support  │  │  GDPR    │       │
│   │  Module  │  │  Module  │  │  Module  │  │  Module  │       │
│   └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                                                  │
│   ┌──────────────────────────────────────────────────────┐      │
│   │               SHARED / COMMON                         │      │
│   │   Guards · Interceptors · Pipes · Filters · Utils    │      │
│   └──────────────────────────────────────────────────────┘      │
└────┬───────────┬──────────────┬──────────────┬──────────────────┘
     │           │              │              │
  ┌──▼──┐   ┌───▼──┐     ┌─────▼──┐    ┌──────▼─────┐
  │ PG  │   │Redis │     │  S3 /  │    │  BullMQ    │
  │ DB  │   │Cache │     │ Minio  │    │  Queues    │
  └─────┘   └──────┘     └────────┘    └────────────┘
```

### Why Monolithic?

- **Stage:** Early startup — team of 1-3 devs
- **Complexity:** Moderate — 17 modules, but all tightly coupled through branch context
- **Migration path:** NestJS modules are self-contained — easy to extract as microservices later if needed
- **Deployment:** Single container, simpler DevOps, lower cost

---

## 2. Tech Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Runtime** | Node.js | 20 LTS | JavaScript runtime |
| **Framework** | NestJS | 10.x | Backend framework |
| **Language** | TypeScript | 5.x | Type safety |
| **ORM** | TypeORM | 0.3.x | Database access & migrations |
| **Database** | PostgreSQL | 16+ | Primary data store |
| **Cache** | Redis | 7.x | Session cache, rate limiting, queue broker |
| **Queue** | BullMQ | 5.x | Background jobs (email, SDI, notifications) |
| **Auth** | Passport.js | 0.7.x | Authentication strategies |
| **JWT** | @nestjs/jwt | 10.x | Token management |
| **Validation** | class-validator + class-transformer | — | Request validation + DTO transform |
| **API Docs** | @nestjs/swagger | 7.x | OpenAPI / Swagger documentation |
| **File Storage** | AWS S3 / MinIO | — | Product images, invoices, PDFs |
| **Email** | @nestjs-modules/mailer + nodemailer | — | Transactional emails |
| **SMS** | Twilio SDK | — | SMS notifications |
| **Push** | firebase-admin | — | FCM push notifications |
| **Payment** | Stripe SDK | — | Payment processing |
| **Geospatial** | PostGIS + pg functions | — | Store locator queries |
| **Search** | PostgreSQL tsvector (phase 1) → Meilisearch (phase 2) | — | Product search |
| **Testing** | Jest + Supertest | — | Unit + E2E tests |
| **Linting** | ESLint + Prettier | — | Code quality |

---

## 3. Project Structure

```
mia-medical-backend/
├── .env.example                        # Environment variables template
├── .env                                # Local env (gitignored)
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
├── package.json
├── docker-compose.yml                  # Local dev: PG + Redis + MinIO
├── Dockerfile
│
├── src/
│   ├── main.ts                         # App bootstrap
│   ├── app.module.ts                   # Root module
│   │
│   ├── config/                         # Configuration
│   │   ├── config.module.ts
│   │   ├── database.config.ts          # TypeORM config
│   │   ├── redis.config.ts
│   │   ├── s3.config.ts
│   │   ├── jwt.config.ts
│   │   ├── mail.config.ts
│   │   └── app.config.ts               # General app config
│   │
│   ├── common/                         # Shared utilities
│   │   ├── constants/
│   │   │   ├── permissions.ts          # Permission resource:action constants
│   │   │   ├── rental-statuses.ts
│   │   │   └── order-statuses.ts
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts
│   │   │   ├── current-branch.decorator.ts
│   │   │   ├── permissions.decorator.ts
│   │   │   ├── public.decorator.ts
│   │   │   └── api-pagination.decorator.ts
│   │   ├── dto/
│   │   │   ├── pagination.dto.ts
│   │   │   ├── pagination-response.dto.ts
│   │   │   └── api-response.dto.ts
│   │   ├── entities/
│   │   │   └── base.entity.ts          # id, created_at, updated_at, deleted_at
│   │   ├── enums/
│   │   │   ├── user-type.enum.ts
│   │   │   ├── auth-provider.enum.ts
│   │   │   ├── order-status.enum.ts
│   │   │   ├── rental-status.enum.ts
│   │   │   ├── payment-status.enum.ts
│   │   │   ├── invoice-type.enum.ts
│   │   │   └── ... (all DB enums)
│   │   ├── exceptions/
│   │   │   ├── business.exception.ts
│   │   │   └── not-found.exception.ts
│   │   ├── filters/
│   │   │   ├── http-exception.filter.ts
│   │   │   └── typeorm-exception.filter.ts
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   ├── permissions.guard.ts
│   │   │   └── branch-context.guard.ts
│   │   ├── interceptors/
│   │   │   ├── response-transform.interceptor.ts
│   │   │   ├── audit-log.interceptor.ts
│   │   │   └── branch-filter.interceptor.ts
│   │   ├── middleware/
│   │   │   └── request-logger.middleware.ts
│   │   ├── pipes/
│   │   │   └── parse-uuid.pipe.ts
│   │   └── utils/
│   │       ├── money.util.ts           # cents ↔ decimal conversion
│   │       ├── codice-fiscale.util.ts  # Italian fiscal code validation
│   │       ├── partita-iva.util.ts     # Italian VAT number validation
│   │       └── slug.util.ts
│   │
│   ├── modules/
│   │   │
│   │   ├── auth/
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   ├── jwt.strategy.ts
│   │   │   │   ├── jwt-refresh.strategy.ts
│   │   │   │   ├── spid.strategy.ts
│   │   │   │   └── cie.strategy.ts
│   │   │   ├── dto/
│   │   │   │   ├── login.dto.ts
│   │   │   │   ├── register.dto.ts
│   │   │   │   ├── refresh-token.dto.ts
│   │   │   │   ├── forgot-password.dto.ts
│   │   │   │   └── reset-password.dto.ts
│   │   │   └── guards/
│   │   │       ├── spid-auth.guard.ts
│   │   │       └── cie-auth.guard.ts
│   │   │
│   │   ├── users/
│   │   │   ├── users.module.ts
│   │   │   ├── users.controller.ts
│   │   │   ├── users.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── user.entity.ts
│   │   │   │   ├── user-auth-provider.entity.ts
│   │   │   │   ├── customer-profile.entity.ts
│   │   │   │   ├── user-address.entity.ts
│   │   │   │   ├── refresh-token.entity.ts
│   │   │   │   └── password-reset-token.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-user.dto.ts
│   │   │       ├── update-user.dto.ts
│   │   │       ├── update-profile.dto.ts
│   │   │       └── create-address.dto.ts
│   │   │
│   │   ├── branches/
│   │   │   ├── branches.module.ts
│   │   │   ├── branches.controller.ts
│   │   │   ├── branches.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── branch.entity.ts
│   │   │   │   └── branch-operating-hours.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-branch.dto.ts
│   │   │       ├── update-branch.dto.ts
│   │   │       └── store-locator.dto.ts
│   │   │
│   │   ├── rbac/
│   │   │   ├── rbac.module.ts
│   │   │   ├── roles.controller.ts
│   │   │   ├── permissions.controller.ts
│   │   │   ├── rbac.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── role.entity.ts
│   │   │   │   ├── permission.entity.ts
│   │   │   │   ├── role-permission.entity.ts
│   │   │   │   └── user-branch-role.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-role.dto.ts
│   │   │       ├── assign-role.dto.ts
│   │   │       └── update-permissions.dto.ts
│   │   │
│   │   ├── products/
│   │   │   ├── products.module.ts
│   │   │   ├── products.controller.ts
│   │   │   ├── products.service.ts
│   │   │   ├── categories.controller.ts
│   │   │   ├── categories.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── category.entity.ts
│   │   │   │   ├── product.entity.ts
│   │   │   │   ├── product-media.entity.ts
│   │   │   │   ├── product-variant.entity.ts
│   │   │   │   ├── product-spec.entity.ts
│   │   │   │   ├── product-rental-tier.entity.ts
│   │   │   │   └── branch-product-override.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-product.dto.ts
│   │   │       ├── update-product.dto.ts
│   │   │       ├── product-filter.dto.ts
│   │   │       ├── create-category.dto.ts
│   │   │       └── branch-override.dto.ts
│   │   │
│   │   ├── cart/
│   │   │   ├── cart.module.ts
│   │   │   ├── cart.controller.ts
│   │   │   ├── cart.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── cart.entity.ts
│   │   │   │   └── cart-item.entity.ts
│   │   │   └── dto/
│   │   │       ├── add-to-cart.dto.ts
│   │   │       └── update-cart-item.dto.ts
│   │   │
│   │   ├── orders/
│   │   │   ├── orders.module.ts
│   │   │   ├── orders.controller.ts
│   │   │   ├── orders.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── order.entity.ts
│   │   │   │   ├── order-item.entity.ts
│   │   │   │   └── order-status-history.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-order.dto.ts
│   │   │       ├── update-order-status.dto.ts
│   │   │       └── order-filter.dto.ts
│   │   │
│   │   ├── rentals/
│   │   │   ├── rentals.module.ts
│   │   │   ├── rentals.controller.ts
│   │   │   ├── rentals.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── rental.entity.ts
│   │   │   │   ├── rental-status-history.entity.ts
│   │   │   │   ├── rental-photo.entity.ts
│   │   │   │   └── rental-extension.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-rental.dto.ts
│   │   │       ├── extend-rental.dto.ts
│   │   │       ├── checkin-rental.dto.ts
│   │   │       └── rental-filter.dto.ts
│   │   │
│   │   ├── inventory/
│   │   │   ├── inventory.module.ts
│   │   │   ├── inventory.controller.ts
│   │   │   ├── inventory.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── inventory-unit.entity.ts
│   │   │   │   └── calibration-record.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-unit.dto.ts
│   │   │       ├── update-unit.dto.ts
│   │   │       └── record-calibration.dto.ts
│   │   │
│   │   ├── payments/
│   │   │   ├── payments.module.ts
│   │   │   ├── payments.controller.ts
│   │   │   ├── payments.service.ts
│   │   │   ├── stripe.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── payment.entity.ts
│   │   │   │   └── deposit.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-payment.dto.ts
│   │   │       └── webhook-event.dto.ts
│   │   │
│   │   ├── invoices/
│   │   │   ├── invoices.module.ts
│   │   │   ├── invoices.controller.ts
│   │   │   ├── invoices.service.ts
│   │   │   ├── sdi.service.ts           # SDI XML generation & transmission
│   │   │   ├── invoice-pdf.service.ts   # PDF generation
│   │   │   ├── entities/
│   │   │   │   ├── invoice.entity.ts
│   │   │   │   ├── invoice-line-item.entity.ts
│   │   │   │   └── invoice-sequence.entity.ts
│   │   │   └── dto/
│   │   │       ├── create-invoice.dto.ts
│   │   │       └── invoice-filter.dto.ts
│   │   │
│   │   ├── wishlist/
│   │   │   ├── wishlist.module.ts
│   │   │   ├── wishlist.controller.ts
│   │   │   ├── wishlist.service.ts
│   │   │   └── entities/
│   │   │       ├── wishlist.entity.ts
│   │   │       └── wishlist-item.entity.ts
│   │   │
│   │   ├── notifications/
│   │   │   ├── notifications.module.ts
│   │   │   ├── notifications.controller.ts
│   │   │   ├── notifications.service.ts
│   │   │   ├── push.service.ts          # FCM
│   │   │   ├── email.service.ts
│   │   │   ├── sms.service.ts           # Twilio
│   │   │   ├── entities/
│   │   │   │   ├── notification.entity.ts
│   │   │   │   └── push-token.entity.ts
│   │   │   └── templates/
│   │   │       ├── order-confirmation.hbs
│   │   │       ├── rental-reminder.hbs
│   │   │       ├── password-reset.hbs
│   │   │       └── welcome.hbs
│   │   │
│   │   ├── support/
│   │   │   ├── support.module.ts
│   │   │   ├── support.controller.ts
│   │   │   ├── support.service.ts
│   │   │   └── entities/
│   │   │       ├── support-ticket.entity.ts
│   │   │       └── ticket-message.entity.ts
│   │   │
│   │   ├── blog/
│   │   │   ├── blog.module.ts
│   │   │   ├── blog.controller.ts
│   │   │   ├── blog.service.ts
│   │   │   └── entities/
│   │   │       ├── blog-post.entity.ts
│   │   │       ├── blog-category.entity.ts
│   │   │       └── blog-tag.entity.ts
│   │   │
│   │   ├── reviews/
│   │   │   ├── reviews.module.ts
│   │   │   ├── reviews.controller.ts
│   │   │   ├── reviews.service.ts
│   │   │   └── entities/
│   │   │       ├── product-review.entity.ts
│   │   │       ├── product-question.entity.ts
│   │   │       └── product-answer.entity.ts
│   │   │
│   │   ├── gdpr/
│   │   │   ├── gdpr.module.ts
│   │   │   ├── gdpr.controller.ts
│   │   │   ├── gdpr.service.ts
│   │   │   └── entities/
│   │   │       ├── gdpr-request.entity.ts
│   │   │       └── communication-preference.entity.ts
│   │   │
│   │   ├── audit/
│   │   │   ├── audit.module.ts
│   │   │   ├── audit.service.ts
│   │   │   └── entities/
│   │   │       └── audit-log.entity.ts
│   │   │
│   │   └── health/
│   │       ├── health.module.ts
│   │       └── health.controller.ts     # Health check endpoint
│   │
│   └── database/
│       ├── migrations/                  # TypeORM migrations
│       │   └── ... (auto-generated)
│       └── seeds/
│           ├── permissions.seed.ts      # Seed all permission resource:action pairs
│           ├── roles.seed.ts            # Seed preset roles with permissions
│           └── branches.seed.ts         # Seed initial branches (Roma, Firenze)
│
├── test/
│   ├── jest-e2e.config.ts
│   ├── app.e2e-spec.ts
│   └── fixtures/
│       ├── users.fixture.ts
│       ├── products.fixture.ts
│       └── branches.fixture.ts
│
└── docs/
    └── api/                             # Generated OpenAPI docs
```

---

## 4. Module Breakdown

### Module Dependencies (import graph)

```
AppModule
├── ConfigModule (global)
├── TypeOrmModule (global)
├── CacheModule (Redis, global)
├── BullModule (global)
├── HealthModule
├── AuthModule ──────────────► UsersModule
├── BranchesModule
├── RbacModule ──────────────► UsersModule, BranchesModule
├── ProductsModule ──────────► BranchesModule
├── CartModule ──────────────► ProductsModule, UsersModule, BranchesModule
├── OrdersModule ────────────► CartModule, ProductsModule, PaymentsModule, InvoicesModule
├── RentalsModule ───────────► ProductsModule, InventoryModule, PaymentsModule, InvoicesModule
├── InventoryModule ─────────► ProductsModule, BranchesModule
├── PaymentsModule
├── InvoicesModule ──────────► BranchesModule, UsersModule
├── WishlistModule ──────────► ProductsModule
├── NotificationsModule
├── SupportModule
├── BlogModule
├── ReviewsModule ───────────► ProductsModule
├── GdprModule ──────────────► UsersModule
└── AuditModule (global)
```

### API Prefix Convention

All API routes follow: `/api/v1/{module}/{resource}`

```
Public (no auth):
  POST   /api/v1/auth/login
  POST   /api/v1/auth/register
  POST   /api/v1/auth/refresh
  POST   /api/v1/auth/forgot-password
  POST   /api/v1/auth/reset-password
  GET    /api/v1/auth/spid/callback
  GET    /api/v1/auth/cie/callback
  GET    /api/v1/products
  GET    /api/v1/products/:slug
  GET    /api/v1/categories
  GET    /api/v1/branches
  GET    /api/v1/branches/nearby
  GET    /api/v1/blog/posts
  GET    /api/v1/blog/posts/:slug

Customer (JWT required):
  GET    /api/v1/me                        # Current user profile
  PATCH  /api/v1/me
  GET    /api/v1/me/orders
  GET    /api/v1/me/rentals
  GET    /api/v1/me/invoices
  GET    /api/v1/me/notifications
  GET    /api/v1/me/wishlist
  POST   /api/v1/cart/items
  GET    /api/v1/cart
  POST   /api/v1/checkout/order
  POST   /api/v1/checkout/rental

Admin (JWT + permissions required):
  # Branch-scoped — requires X-Branch-Id header
  GET    /api/v1/admin/dashboard/stats
  CRUD   /api/v1/admin/products
  CRUD   /api/v1/admin/orders
  CRUD   /api/v1/admin/rentals
  CRUD   /api/v1/admin/inventory
  CRUD   /api/v1/admin/invoices
  CRUD   /api/v1/admin/support/tickets
  CRUD   /api/v1/admin/blog/posts
  CRUD   /api/v1/admin/reviews

  # SuperAdmin only
  CRUD   /api/v1/admin/branches
  CRUD   /api/v1/admin/roles
  CRUD   /api/v1/admin/permissions
  CRUD   /api/v1/admin/users
  GET    /api/v1/admin/audit-logs

Webhooks:
  POST   /api/v1/webhooks/stripe
  POST   /api/v1/webhooks/sdi
```

---

## 5. Database & ORM Setup

### TypeORM Configuration

```typescript
// src/config/database.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export const getDatabaseConfig = (): TypeOrmModuleOptions => ({
  type: 'postgres',
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT, 10),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
  ssl: process.env.DB_SSL === 'true' ? { rejectUnauthorized: false } : false,
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  migrations: [__dirname + '/../database/migrations/*{.ts,.js}'],
  synchronize: false,      // NEVER true in production
  logging: process.env.DB_LOGGING === 'true',
  migrationsRun: false,
  extra: {
    max: parseInt(process.env.DB_POOL_MAX, 10) || 20,
    min: parseInt(process.env.DB_POOL_MIN, 10) || 5,
  },
});
```

### Base Entity

```typescript
// src/common/entities/base.entity.ts
import {
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
} from 'typeorm';

export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn({ type: 'timestamptz', name: 'created_at' })
  createdAt: Date;

  @UpdateDateColumn({ type: 'timestamptz', name: 'updated_at' })
  updatedAt: Date;

  @DeleteDateColumn({ type: 'timestamptz', name: 'deleted_at', nullable: true })
  deletedAt: Date | null;
}
```

### Migration Commands

```bash
# Generate migration from entity changes
npx typeorm migration:generate src/database/migrations/MigrationName -d src/config/typeorm-cli.config.ts

# Run migrations
npx typeorm migration:run -d src/config/typeorm-cli.config.ts

# Revert last migration
npx typeorm migration:revert -d src/config/typeorm-cli.config.ts

# Seed database
npx ts-node src/database/seeds/run-seeds.ts
```

---

## 6. Authentication & Authorization

### Auth Flow

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Client  │────►│  Login   │────►│  JWT     │────►│ Protected│
│          │     │ Endpoint │     │ Issued   │     │ Resource │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                       │
                                       ▼
                              ┌──────────────────┐
                              │  Access Token    │ 15 min expiry
                              │  Refresh Token   │ 30 day expiry
                              │  (stored in DB)  │
                              └──────────────────┘
```

### JWT Payload

```typescript
interface JwtPayload {
  sub: string;          // user.id (UUID)
  email: string;
  userType: UserType;   // 'customer' | 'admin'
  isSuperAdmin: boolean;
  iat: number;
  exp: number;
}
```

### SPID Integration

```typescript
// src/modules/auth/strategies/spid.strategy.ts
// SPID uses SAML 2.0 — use passport-saml
// Configure with IdP metadata from AgID
// Callback URL: /api/v1/auth/spid/callback

// Required SPID attributes:
// - spidCode (unique identifier)
// - name, familyName
// - fiscalNumber (codice fiscale)
// - email
```

### Permission Guard

```typescript
// src/common/guards/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions = this.reflector.get<string[]>(
      'permissions',
      context.getHandler(),
    );
    if (!requiredPermissions) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const branchId = request.headers['x-branch-id'];

    // SuperAdmin bypasses all permission checks
    if (user.isSuperAdmin) return true;

    // Check user's permissions for the specified branch
    const userPermissions = await this.rbacService.getUserPermissions(
      user.id,
      branchId,
    );

    return requiredPermissions.every(p => userPermissions.includes(p));
  }
}
```

### Usage in Controllers

```typescript
@Controller('api/v1/admin/products')
@UseGuards(JwtAuthGuard, PermissionsGuard)
export class AdminProductsController {

  @Get()
  @Permissions('products:read')
  findAll(@CurrentBranch() branchId: string) { ... }

  @Post()
  @Permissions('products:write')
  create(@Body() dto: CreateProductDto) { ... }

  @Delete(':id')
  @Permissions('products:delete')
  remove(@Param('id') id: string) { ... }
}
```

---

## 7. Multi-Tenancy Implementation

### Branch Context

Every admin request carries the branch context via `X-Branch-Id` header.

```typescript
// src/common/decorators/current-branch.decorator.ts
export const CurrentBranch = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    return request.headers['x-branch-id'];
  },
);

// src/common/guards/branch-context.guard.ts
@Injectable()
export class BranchContextGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const branchId = request.headers['x-branch-id'];
    const user = request.user;

    // SuperAdmin can access any branch (or "all")
    if (user.isSuperAdmin) return true;

    // Verify user has a role assignment for this branch
    const hasAccess = await this.rbacService.userHasBranchAccess(
      user.id,
      branchId,
    );

    if (!hasAccess) throw new ForbiddenException('No access to this branch');
    return true;
  }
}
```

### Query Scoping

```typescript
// Every branch-scoped query adds WHERE branch_id = :branchId
// Use a reusable method in services:

async findAll(branchId: string, filters: FilterDto) {
  const qb = this.repo.createQueryBuilder('order')
    .where('order.branch_id = :branchId', { branchId })
    .andWhere('order.deleted_at IS NULL');

  // SuperAdmin "all branches" mode: omit branch filter
  if (branchId === 'all') {
    qb.where('order.deleted_at IS NULL');
  }

  return qb.getManyAndCount();
}
```

---

## 8. API Design

### Standard Response Format

```typescript
// Success
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "perPage": 20,
    "total": 150,
    "totalPages": 8
  }
}

// Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Codice fiscale non valido",
    "details": [
      { "field": "codiceFiscale", "message": "Formato non valido" }
    ]
  }
}
```

### Pagination

```typescript
// GET /api/v1/products?page=1&perPage=20&sort=name&order=asc
export class PaginationDto {
  @IsOptional() @IsInt() @Min(1) page?: number = 1;
  @IsOptional() @IsInt() @Min(1) @Max(100) perPage?: number = 20;
  @IsOptional() @IsString() sort?: string;
  @IsOptional() @IsIn(['asc', 'desc']) order?: 'asc' | 'desc' = 'asc';
}
```

### Soft Delete Convention

```typescript
// Never hard-delete. Use TypeORM softRemove:
await this.repo.softRemove(entity);

// All queries automatically exclude deleted_at IS NOT NULL
// when using @DeleteDateColumn()
```

---

## 9. Key Flows

### Rental Lifecycle

```
Customer Action          API Endpoint                    Status Transition
─────────────────────────────────────────────────────────────────────────
Customer books rental → POST /checkout/rental          → reserved
Payment authorized    → Stripe webhook                 → deposit_held
Branch dispatches     → PATCH /admin/rentals/:id       → dispatched
Customer picks up     → PATCH /admin/rentals/:id       → checked_out → active
Customer requests ext → POST /me/rentals/:id/extend    → extension_requested
Admin approves ext    → PATCH /admin/rentals/:id       → active (extended)
Return initiated      → PATCH /me/rentals/:id/return   → return_pending
Customer returns      → PATCH /admin/rentals/:id       → checked_in
Admin inspects        → PATCH /admin/rentals/:id       → inspected
No damage → released  → PATCH /admin/rentals/:id       → completed
Damage → charge       → PATCH /admin/rentals/:id       → completed (deposit charged)
```

### Invoice Auto-Detection

```typescript
// src/modules/invoices/invoices.service.ts
detectInvoiceType(profile: CustomerProfile): InvoiceType {
  if (profile.euVatNumber) return InvoiceType.INTRA_EU;
  if (profile.codiceUnivoco) return InvoiceType.PA;
  if (profile.partitaIva) return InvoiceType.B2B;
  return InvoiceType.B2C;
}
```

### Gapless Invoice Numbering

```typescript
// Must use SELECT ... FOR UPDATE within a transaction
async generateInvoiceNumber(branchId: string): Promise<string> {
  return this.dataSource.transaction(async (manager) => {
    const year = new Date().getFullYear();
    const branch = await manager.findOne(Branch, { where: { id: branchId } });

    let sequence = await manager
      .createQueryBuilder(InvoiceSequence, 'seq')
      .setLock('pessimistic_write')
      .where('seq.branch_id = :branchId AND seq.year = :year', { branchId, year })
      .getOne();

    if (!sequence) {
      sequence = manager.create(InvoiceSequence, {
        branchId, year, lastSequence: 0,
      });
    }

    sequence.lastSequence += 1;
    await manager.save(sequence);

    return `${year}/${branch.code}/${String(sequence.lastSequence).padStart(5, '0')}`;
    // Example: 2026/ROM/00042
  });
}
```

---

## 10. File Storage

### S3 / MinIO Setup

```typescript
// src/config/s3.config.ts
// Use AWS SDK v3 or MinIO client
// Buckets:
//   mia-product-media     — product images/videos
//   mia-invoices           — generated invoice PDFs + XML
//   mia-rental-photos      — check-in/out photos
//   mia-documents          — rental agreements, calibration certs
//   mia-blog               — blog post images

// All uploads return a CDN-fronted URL
// Use presigned URLs for direct upload from mobile app
```

### Upload Flow

```typescript
// Admin uploads product image:
// 1. POST /api/v1/admin/upload/presigned-url → returns S3 presigned PUT URL
// 2. Client uploads directly to S3
// 3. Client sends S3 key back: POST /api/v1/admin/products/:id/media
// 4. Server stores the URL in product_media table
```

---

## 11. Email, SMS & Push Notifications

### Notification Service

```typescript
// src/modules/notifications/notifications.service.ts
// Dispatches notifications based on user's communication preferences

// Events that trigger notifications:
// - Order confirmed      → email + push
// - Order shipped        → email + sms + push
// - Rental starting soon → push + sms (1 day before)
// - Rental expiring      → push + sms + email (2 days before)
// - Rental overdue       → push + sms + email (daily)
// - Invoice ready        → email + in_app
// - Support ticket reply → email + push
// - New blog post        → push (if subscribed)
// - Password reset       → email only
// - Welcome              → email only
```

### Email Templates

Use Handlebars (`.hbs`) templates in `/src/modules/notifications/templates/`.
All emails in Italian with the MIA Medical branding.

### SMS via Twilio

```typescript
// Rental reminders + order delivery notifications
// Use Twilio Verify for phone verification
// Italian phone format: +39 XXX XXXXXXX
```

---

## 12. Invoice & SDI Integration

### SDI (Sistema di Interscambio) Flow

```
Invoice Created → Generate FatturaPA XML → Sign with digital cert
     → Transmit to SDI → Track response → Update sdi_status
```

### Integration Options

1. **FattureInCloud API** (recommended for MVP) — handles XML generation, signing, transmission
2. **Direct SDI integration** — requires digital certificate, SOAP/REST API with Agenzia delle Entrate
3. **Internal FatturaPA XML** — generate XML internally, use SDI proxy service for transmission

### XML Format: FatturaPA

```typescript
// Generate FatturaPA XML following AgID specifications
// Required fields per invoice type:
// B2C: CodiceDestinatario = "0000000", CodiceFiscale
// B2B: CodiceDestinatario = SDI code, PartitaIVA
// PA:  CodiceDestinatario = Codice Univoco (7 chars)
// EU:  IdFiscaleIVA with country prefix
```

---

## 13. Search

### Phase 1: PostgreSQL Full-Text Search

```typescript
// Use tsvector columns already in the DB schema
// Products: search_vector (name + short_description + description)
// Blog: search_vector (title + excerpt + body)

// Query:
const results = await this.repo
  .createQueryBuilder('product')
  .where("product.search_vector @@ plainto_tsquery('italian', :query)", { query })
  .orderBy("ts_rank(product.search_vector, plainto_tsquery('italian', :query))", 'DESC')
  .getMany();
```

### Phase 2: Meilisearch

```
When catalogue grows > 500 products, add Meilisearch for:
- Typo tolerance
- Faceted filtering
- Instant search suggestions
- Sync via BullMQ job on product create/update/delete
```

---

## 14. Caching

### Redis Strategy

| Key Pattern | TTL | Content |
|---|---|---|
| `product:{id}` | 5 min | Product detail |
| `products:list:{hash}` | 2 min | Product list with filters |
| `categories:tree` | 30 min | Category hierarchy |
| `branches:all` | 30 min | All active branches |
| `branch:{id}:hours` | 30 min | Operating hours |
| `user:{id}:permissions:{branchId}` | 5 min | Resolved permissions |
| `rate-limit:{ip}` | 1 min | API rate limiting |

### Cache Invalidation

```typescript
// Use @CacheEvict pattern:
// On product update → evict product:{id} + products:list:*
// On role change → evict user:{id}:permissions:*
```

---

## 15. Queue & Background Jobs

### BullMQ Queues

| Queue | Jobs | Priority |
|---|---|---|
| `email` | Send transactional emails | Normal |
| `sms` | Send SMS via Twilio | Normal |
| `push` | Send push notifications via FCM | Normal |
| `invoice` | Generate PDF, transmit to SDI | High |
| `rental-reminder` | Daily: check expiring rentals, send reminders | Low (CRON) |
| `overdue-check` | Daily: mark overdue rentals | Low (CRON) |
| `gdpr-export` | Generate user data export ZIP | Low |
| `gdpr-delete` | Hard-delete user data (GDPR erasure) | Low |
| `audit-cleanup` | Monthly: archive old audit logs | Low (CRON) |
| `calibration-alert` | Weekly: check upcoming calibrations | Low (CRON) |

### CRON Jobs

```typescript
// src/modules/rentals/rental-cron.service.ts
@Cron('0 8 * * *')  // Every day at 8:00 AM Rome time
async checkExpiringRentals() {
  // Find rentals expiring in 2 days → queue rental-reminder
}

@Cron('0 0 * * *')  // Midnight
async markOverdueRentals() {
  // Find active rentals past expected_end_date → set status to 'overdue'
}
```

---

## 16. Error Handling

### Error Codes

```typescript
// src/common/constants/error-codes.ts
export const ErrorCodes = {
  // Auth
  INVALID_CREDENTIALS: 'AUTH_001',
  TOKEN_EXPIRED: 'AUTH_002',
  ACCOUNT_DISABLED: 'AUTH_003',
  SPID_AUTH_FAILED: 'AUTH_004',

  // RBAC
  NO_PERMISSION: 'RBAC_001',
  NO_BRANCH_ACCESS: 'RBAC_002',

  // Products
  PRODUCT_NOT_FOUND: 'PROD_001',
  PRODUCT_NOT_AVAILABLE: 'PROD_002',
  INSUFFICIENT_STOCK: 'PROD_003',

  // Orders
  ORDER_NOT_FOUND: 'ORD_001',
  INVALID_STATUS_TRANSITION: 'ORD_002',

  // Rentals
  RENTAL_NOT_FOUND: 'RNT_001',
  NO_UNITS_AVAILABLE: 'RNT_002',
  RENTAL_ALREADY_RETURNED: 'RNT_003',
  EXTENSION_CONFLICT: 'RNT_004',

  // Payments
  PAYMENT_FAILED: 'PAY_001',
  DEPOSIT_ALREADY_RELEASED: 'PAY_002',

  // Invoices
  INVOICE_GENERATION_FAILED: 'INV_001',
  SDI_TRANSMISSION_FAILED: 'INV_002',

  // Validation
  VALIDATION_ERROR: 'VAL_001',
  INVALID_CODICE_FISCALE: 'VAL_002',
  INVALID_PARTITA_IVA: 'VAL_003',
};
```

---

## 17. Testing Strategy

### Layers

```
Unit Tests (Jest)
├── Services — business logic with mocked repositories
├── Guards — permission checks with mocked RBAC service
├── Utils — money conversion, fiscal code validation
└── Pipes — validation logic

Integration Tests (Jest + TestContainers)
├── Repository — actual PG queries
└── Service + Repository — full module integration

E2E Tests (Supertest)
├── Auth flows — login, register, SPID callback
├── Product CRUD — public and admin endpoints
├── Checkout — order + rental full flow
├── RBAC — permission enforcement
└── Multi-tenancy — branch isolation
```

### Test Database

```typescript
// Use testcontainers for PostgreSQL in CI
// docker-compose.test.yml for local dev with separate test DB
// Each test suite uses transactions that rollback after test
```

---

## 18. Deployment & DevOps

### Docker Compose (Local Dev)

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgis/postgis:16-3.4
    environment:
      POSTGRES_DB: mia_medical
      POSTGRES_USER: mia_dev
      POSTGRES_PASSWORD: mia_dev_pass
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: mia_minio
      MINIO_ROOT_PASSWORD: mia_minio_pass
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data

volumes:
  pg_data:
  minio_data:
```

### Production Deployment

```
Options:
1. Docker container on AWS ECS / Google Cloud Run / Fly.io
2. VPS with PM2 + Nginx reverse proxy

Recommended stack:
- App: Docker container (NestJS)
- DB: Managed PostgreSQL (Supabase / Neon / RDS)
- Cache: Managed Redis (Upstash / ElastiCache)
- Storage: AWS S3 / Cloudflare R2
- CDN: Cloudflare
- CI/CD: GitHub Actions
```

---

## 19. Environment Variables Reference

See `.env.example` file for the complete template. Key groups:

| Group | Variables | Required |
|---|---|---|
| **App** | `NODE_ENV`, `PORT`, `APP_URL`, `API_PREFIX` | Yes |
| **Database** | `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` | Yes |
| **Redis** | `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` | Yes |
| **JWT** | `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`, `JWT_ACCESS_EXPIRY`, `JWT_REFRESH_EXPIRY` | Yes |
| **SPID** | `SPID_ENTITY_ID`, `SPID_CERT_PATH`, `SPID_KEY_PATH`, `SPID_IDP_METADATA_URL` | Prod |
| **CIE** | `CIE_ENTITY_ID`, `CIE_CERT_PATH`, `CIE_KEY_PATH` | Prod |
| **Stripe** | `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `STRIPE_PUBLIC_KEY` | Yes |
| **S3** | `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`, `S3_ENDPOINT` | Yes |
| **Email** | `MAIL_HOST`, `MAIL_PORT`, `MAIL_USER`, `MAIL_PASSWORD`, `MAIL_FROM` | Yes |
| **Twilio** | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER` | Prod |
| **FCM** | `FIREBASE_PROJECT_ID`, `FIREBASE_PRIVATE_KEY`, `FIREBASE_CLIENT_EMAIL` | Prod |
| **SDI** | `SDI_PROVIDER`, `SDI_API_KEY`, `SDI_ENVIRONMENT` | Prod |
| **Sentry** | `SENTRY_DSN` | Prod |

---

## Quick Start

```bash
# 1. Clone and install
git clone <repo-url>
cd mia-medical-backend
npm install

# 2. Start infrastructure
docker-compose up -d

# 3. Configure environment
cp .env.example .env
# Edit .env with your values

# 4. Run migrations
npm run migration:run

# 5. Seed database
npm run seed

# 6. Start development server
npm run start:dev

# Server runs on http://localhost:3000
# Swagger docs: http://localhost:3000/api/docs
```

---

*End of Backend Development Guide v1.0*
