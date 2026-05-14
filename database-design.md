# MIA Medical Italia — Database Design Document

> PostgreSQL 16+ | Multi-tenant (branch-per-tenant) | Italian fiscal compliance

---

## Table of Contents

1. [Design Principles](#design-principles)
2. [Schema Overview](#schema-overview)
3. [Module 1 — Branches & Multi-Tenancy](#module-1--branches--multi-tenancy)
4. [Module 2 — Users & Authentication](#module-2--users--authentication)
5. [Module 3 — RBAC (Role-Based Access Control)](#module-3--rbac)
6. [Module 4 — Product Catalogue](#module-4--product-catalogue)
7. [Module 5 — Cart](#module-5--cart)
8. [Module 6 — Orders (Purchase)](#module-6--orders-purchase)
9. [Module 7 — Rentals](#module-7--rentals)
10. [Module 8 — Inventory & Calibration](#module-8--inventory--calibration)
11. [Module 9 — Payments & Deposits](#module-9--payments--deposits)
12. [Module 10 — Invoicing & SDI](#module-10--invoicing--sdi)
13. [Module 11 — Wishlist](#module-11--wishlist)
14. [Module 12 — Notifications & Push Tokens](#module-12--notifications--push-tokens)
15. [Module 13 — Support Tickets](#module-13--support-tickets)
16. [Module 14 — Blog / Magazine](#module-14--blog--magazine)
17. [Module 15 — Reviews & Q&A](#module-15--reviews--qa)
18. [Module 16 — GDPR & Communication Preferences](#module-16--gdpr--communication-preferences)
19. [Module 17 — Audit Log](#module-17--audit-log)
20. [Indexes & Performance Notes](#indexes--performance-notes)
21. [Entity-Relationship Summary](#entity-relationship-summary)

---

## Design Principles

| Principle | Detail |
|---|---|
| **Multi-tenancy** | Every branch-scoped table carries a `branch_id` FK. RLS (Row-Level Security) policies enforce isolation at the DB layer. |
| **Soft-delete everywhere** | `deleted_at TIMESTAMPTZ` on all mutable tables. Hard-delete only via GDPR erasure jobs. |
| **Timestamps** | Every table has `created_at` and `updated_at` (auto-set via trigger). |
| **UUID primary keys** | `gen_random_uuid()` for all PKs — no sequential IDs exposed to clients. |
| **Money as integer cents** | All monetary values stored as `BIGINT` (cents). Avoids floating-point issues. Currency is always EUR (single-currency system). |
| **Italian locale** | All text search uses `italian` dictionary for `tsvector`. |
| **Fiscal compliance** | Invoice numbering uses per-branch gapless sequences. SDI XML status tracked per invoice. |

---

## Schema Overview

```
mia_medical
├── branches, branch_operating_hours
├── users, user_auth_providers, customer_profiles, user_addresses
├── roles, permissions, role_permissions, user_branch_roles
├── categories, products, product_media, product_variants, product_specs,
│   product_rental_tiers, branch_product_overrides
├── carts, cart_items
├── orders, order_items, order_status_history
├── rentals, rental_status_history, rental_photos, rental_extensions
├── inventory_units, calibration_records
├── payments, deposits
├── invoices, invoice_line_items, invoice_sequences, credit_notes
├── wishlists, wishlist_items
├── notifications, push_tokens
├── support_tickets, ticket_messages
├── blog_posts, blog_categories, blog_post_categories, blog_tags, blog_post_tags
├── product_reviews, product_questions, product_answers
├── gdpr_requests, communication_preferences
└── audit_logs
```

---

## SQL DDL

### Extensions & Enums

```sql
-- Required extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";      -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "postgis";       -- geospatial (drive-time, store locator)
CREATE EXTENSION IF NOT EXISTS "pg_trgm";       -- fuzzy text search

-- ─── ENUMS ───────────────────────────────────────────────

CREATE TYPE user_type AS ENUM ('customer', 'admin');
CREATE TYPE auth_provider AS ENUM ('email', 'spid', 'cie');
CREATE TYPE product_type AS ENUM ('purchase', 'rental', 'both');
CREATE TYPE rental_tier_unit AS ENUM ('daily', 'weekly', 'monthly');
CREATE TYPE order_status AS ENUM (
    'pending_payment', 'confirmed', 'processing',
    'ready_for_pickup', 'out_for_delivery', 'delivered',
    'cancelled', 'refunded'
);
CREATE TYPE delivery_method AS ENUM ('branch_pickup', 'home_delivery');
CREATE TYPE rental_status AS ENUM (
    'reserved', 'deposit_held', 'dispatched', 'checked_out',
    'active', 'extension_requested', 'return_pending',
    'checked_in', 'inspected', 'completed',
    'cancelled', 'overdue'
);
CREATE TYPE rental_photo_type AS ENUM ('check_out', 'check_in', 'damage');
CREATE TYPE payment_status AS ENUM ('pending', 'succeeded', 'failed', 'refunded', 'partially_refunded');
CREATE TYPE payment_method AS ENUM ('credit_card', 'apple_pay', 'google_pay', 'bank_transfer', 'cash');
CREATE TYPE deposit_status AS ENUM ('held', 'released', 'partially_charged', 'fully_charged');
CREATE TYPE invoice_type AS ENUM ('b2c', 'b2b', 'pa', 'intra_eu');
CREATE TYPE sdi_status AS ENUM ('draft', 'pending', 'sent', 'accepted', 'rejected', 'error');
CREATE TYPE ticket_status AS ENUM ('open', 'in_progress', 'waiting_customer', 'resolved', 'closed');
CREATE TYPE ticket_priority AS ENUM ('low', 'medium', 'high', 'urgent');
CREATE TYPE blog_post_status AS ENUM ('draft', 'published', 'archived');
CREATE TYPE review_source AS ENUM ('internal', 'trustpilot', 'google', 'other');
CREATE TYPE gdpr_request_type AS ENUM ('data_export', 'account_deletion');
CREATE TYPE gdpr_request_status AS ENUM ('pending', 'processing', 'completed', 'rejected');
CREATE TYPE notification_channel AS ENUM ('push', 'email', 'sms', 'in_app');
CREATE TYPE unit_condition AS ENUM ('new', 'good', 'fair', 'needs_repair', 'decommissioned');
CREATE TYPE day_of_week AS ENUM ('monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday');
```

---

## Module 1 — Branches & Multi-Tenancy

```sql
-- ─── BRANCHES ────────────────────────────────────────────

CREATE TABLE branches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(10) NOT NULL UNIQUE,          -- e.g. 'ROM', 'FIR'
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) NOT NULL UNIQUE,
    email           VARCHAR(255),
    phone           VARCHAR(30),
    whatsapp        VARCHAR(30),
    address_line1   VARCHAR(300) NOT NULL,
    address_line2   VARCHAR(300),
    city            VARCHAR(100) NOT NULL,
    province        VARCHAR(5) NOT NULL,                  -- e.g. 'RM', 'FI'
    postal_code     VARCHAR(10) NOT NULL,
    country         CHAR(2) NOT NULL DEFAULT 'IT',
    location        GEOGRAPHY(POINT, 4326) NOT NULL,      -- PostGIS for geospatial queries
    timezone        VARCHAR(50) NOT NULL DEFAULT 'Europe/Rome',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE branch_operating_hours (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    day_of_week     day_of_week NOT NULL,
    open_time       TIME NOT NULL,
    close_time      TIME NOT NULL,
    is_closed       BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (branch_id, day_of_week)
);
```

---

## Module 2 — Users & Authentication

```sql
-- ─── USERS (both customers and admin staff) ─────────────

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_type       user_type NOT NULL DEFAULT 'customer',
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(30),
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    avatar_url      TEXT,
    locale          VARCHAR(10) NOT NULL DEFAULT 'it',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    is_superadmin   BOOLEAN NOT NULL DEFAULT FALSE,       -- only 1 superadmin
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    phone_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── AUTH PROVIDERS (SPID, CIE, email/password) ─────────

CREATE TABLE user_auth_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider        auth_provider NOT NULL,
    provider_uid    VARCHAR(500),                          -- SPID/CIE subject ID
    password_hash   TEXT,                                  -- only for 'email' provider
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (user_id, provider),
    UNIQUE (provider, provider_uid)
);

-- ─── CUSTOMER PROFILES (Italian fiscal data) ────────────

CREATE TABLE customer_profiles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    -- Italian fiscal identifiers (determines invoice type)
    codice_fiscale  VARCHAR(16),                           -- B2C identifier
    partita_iva     VARCHAR(11),                           -- B2B VAT number
    codice_univoco  VARCHAR(7),                            -- PA (Public Administration)
    eu_vat_number   VARCHAR(20),                           -- Intra-EU VAT
    pec             VARCHAR(255),                          -- Certified email (PEC)
    -- Company data (for B2B / PA)
    company_name    VARCHAR(300),
    -- Derived invoice type (auto-detected from above fields)
    invoice_type    invoice_type NOT NULL DEFAULT 'b2c',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── USER ADDRESSES ─────────────────────────────────────

CREATE TABLE user_addresses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    label           VARCHAR(50),                           -- e.g. 'Casa', 'Ufficio'
    first_name      VARCHAR(100) NOT NULL,
    last_name       VARCHAR(100) NOT NULL,
    address_line1   VARCHAR(300) NOT NULL,
    address_line2   VARCHAR(300),
    city            VARCHAR(100) NOT NULL,
    province        VARCHAR(5) NOT NULL,
    postal_code     VARCHAR(10) NOT NULL,
    country         CHAR(2) NOT NULL DEFAULT 'IT',
    phone           VARCHAR(30),
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);
```

---

## Module 3 — RBAC

> **This is the most critical system.** SuperAdmin is the only hardcoded role. All other roles
> are fully customizable. A user is assigned a role **per branch**.

```sql
-- ─── PERMISSIONS ─────────────────────────────────────────
-- Granular: resource:action (e.g. 'products:read', 'orders:write', 'invoices:void')

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,                 -- e.g. 'products', 'orders'
    action          VARCHAR(100) NOT NULL,                 -- e.g. 'read', 'write', 'delete', 'void'
    description     VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (resource, action)
);

-- ─── ROLES ───────────────────────────────────────────────
-- Customizable by SuperAdmin. Presets: Admin, Moderator, Delivery Manager, etc.

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    description     VARCHAR(500),
    is_preset       BOOLEAN NOT NULL DEFAULT FALSE,        -- shipped with system
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── ROLE ↔ PERMISSIONS (many-to-many) ───────────────────

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (role_id, permission_id)
);

-- ─── USER ↔ BRANCH ↔ ROLE ───────────────────────────────
-- Same user can have different roles on different branches.

CREATE TABLE user_branch_roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_by     UUID REFERENCES users(id),             -- who assigned this
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (user_id, branch_id)                            -- one role per branch per user
);
```

---

## Module 4 — Product Catalogue

```sql
-- ─── CATEGORIES (hierarchical, self-referencing) ────────

CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES categories(id) ON DELETE SET NULL,
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) NOT NULL UNIQUE,
    description     TEXT,
    image_url       TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── PRODUCTS (master catalogue) ────────────────────────

CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sku             VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(300) NOT NULL,
    slug            VARCHAR(300) NOT NULL UNIQUE,
    description     TEXT,                                  -- rich text (HTML/Markdown)
    short_description TEXT,
    product_type    product_type NOT NULL,                  -- purchase / rental / both
    base_price_cents BIGINT NOT NULL DEFAULT 0,            -- purchase price in cents (EUR)
    vat_rate        NUMERIC(5,2) NOT NULL DEFAULT 22.00,   -- Italian standard VAT 22%
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    -- SEO
    seo_title       VARCHAR(200),
    seo_description VARCHAR(500),
    seo_keywords    TEXT,
    -- Calibration (for medical equipment)
    calibration_interval_days INT,                         -- NULL = no calibration needed
    -- Full-text search
    search_vector   TSVECTOR,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- Auto-update search_vector
CREATE OR REPLACE FUNCTION products_search_vector_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('italian', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('italian', COALESCE(NEW.short_description, '')), 'B') ||
        setweight(to_tsvector('italian', COALESCE(NEW.description, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_search_vector
    BEFORE INSERT OR UPDATE OF name, short_description, description
    ON products
    FOR EACH ROW EXECUTE FUNCTION products_search_vector_trigger();

-- ─── PRODUCT ↔ CATEGORIES (many-to-many) ────────────────

CREATE TABLE product_categories (
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    category_id     UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, category_id)
);

-- ─── PRODUCT MEDIA ──────────────────────────────────────

CREATE TABLE product_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    alt_text        VARCHAR(300),
    media_type      VARCHAR(20) NOT NULL DEFAULT 'image',  -- image, video, document
    sort_order      INT NOT NULL DEFAULT 0,
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── PRODUCT VARIANTS ───────────────────────────────────

CREATE TABLE product_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    sku             VARCHAR(50) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,                 -- e.g. 'Large', 'Blue'
    price_cents     BIGINT,                                -- NULL = use base_price
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── PRODUCT SPECS (key-value) ──────────────────────────

CREATE TABLE product_specs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    spec_key        VARCHAR(100) NOT NULL,
    spec_value      VARCHAR(500) NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,

    UNIQUE (product_id, spec_key)
);

-- ─── RENTAL PRICING TIERS ───────────────────────────────

CREATE TABLE product_rental_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    tier_unit       rental_tier_unit NOT NULL,              -- daily / weekly / monthly
    price_cents     BIGINT NOT NULL,                       -- per unit in cents
    min_units       INT NOT NULL DEFAULT 1,                -- minimum rental period
    max_units       INT,                                   -- NULL = no max
    deposit_cents   BIGINT NOT NULL DEFAULT 0,             -- security deposit
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (product_id, tier_unit)
);

-- ─── BRANCH PRODUCT OVERRIDES ───────────────────────────
-- Per-branch custom pricing, availability, stock. Does NOT modify master catalogue.

CREATE TABLE branch_product_overrides (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    price_cents     BIGINT,                                -- NULL = use master price
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,
    stock_count     INT NOT NULL DEFAULT 0,
    is_visible      BOOLEAN NOT NULL DEFAULT TRUE,         -- category visibility per branch
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (branch_id, product_id)
);
```

---

## Module 5 — Cart

```sql
-- ─── CARTS (server-side for logged-in users) ────────────

CREATE TABLE carts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,  -- 1 cart per user
    branch_id       UUID REFERENCES branches(id) ON DELETE SET NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cart_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id         UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    variant_id      UUID REFERENCES product_variants(id) ON DELETE SET NULL,
    quantity        INT NOT NULL DEFAULT 1 CHECK (quantity > 0),
    -- For rental items
    is_rental       BOOLEAN NOT NULL DEFAULT FALSE,
    rental_tier_id  UUID REFERENCES product_rental_tiers(id) ON DELETE SET NULL,
    rental_units    INT,                                   -- number of days/weeks/months
    rental_start    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (cart_id, product_id, variant_id, is_rental)
);
```

---

## Module 6 — Orders (Purchase)

```sql
-- ─── ORDERS ─────────────────────────────────────────────

CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    order_number    VARCHAR(50) NOT NULL UNIQUE,            -- human-readable order #
    status          order_status NOT NULL DEFAULT 'pending_payment',
    delivery_method delivery_method NOT NULL,
    -- Pricing
    subtotal_cents  BIGINT NOT NULL,
    vat_cents       BIGINT NOT NULL,
    delivery_fee_cents BIGINT NOT NULL DEFAULT 0,
    total_cents     BIGINT NOT NULL,
    -- Delivery address (snapshot at order time)
    delivery_address_id UUID REFERENCES user_addresses(id) ON DELETE SET NULL,
    delivery_address_snapshot JSONB,                        -- frozen copy of address
    -- Estimated date
    estimated_delivery_date DATE,
    actual_delivery_date DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id),
    variant_id      UUID REFERENCES product_variants(id),
    product_name    VARCHAR(300) NOT NULL,                  -- snapshot
    variant_name    VARCHAR(200),                           -- snapshot
    sku             VARCHAR(50) NOT NULL,                   -- snapshot
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price_cents BIGINT NOT NULL,
    vat_rate        NUMERIC(5,2) NOT NULL,
    line_total_cents BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── ORDER STATUS HISTORY ───────────────────────────────

CREATE TABLE order_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    old_status      order_status,
    new_status      order_status NOT NULL,
    changed_by      UUID REFERENCES users(id),             -- NULL = system
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 7 — Rentals

```sql
-- ─── RENTALS ────────────────────────────────────────────

CREATE TABLE rentals (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    rental_number   VARCHAR(50) NOT NULL UNIQUE,
    product_id      UUID NOT NULL REFERENCES products(id),
    variant_id      UUID REFERENCES product_variants(id),
    inventory_unit_id UUID,                                -- FK added after inventory_units table
    status          rental_status NOT NULL DEFAULT 'reserved',
    -- Pricing
    rental_tier_id  UUID NOT NULL REFERENCES product_rental_tiers(id),
    tier_unit       rental_tier_unit NOT NULL,
    units_booked    INT NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    subtotal_cents  BIGINT NOT NULL,
    vat_rate        NUMERIC(5,2) NOT NULL,
    vat_cents       BIGINT NOT NULL,
    total_cents     BIGINT NOT NULL,
    deposit_cents   BIGINT NOT NULL DEFAULT 0,
    -- Dates
    start_date      DATE NOT NULL,
    expected_end_date DATE NOT NULL,
    actual_end_date DATE,
    -- Delivery
    delivery_method delivery_method NOT NULL,
    delivery_address_snapshot JSONB,
    -- Inspection
    checkout_notes  TEXT,
    checkin_notes   TEXT,
    damage_report   TEXT,
    damage_charge_cents BIGINT DEFAULT 0,
    -- Agreement
    agreement_pdf_url TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── RENTAL STATUS HISTORY ──────────────────────────────

CREATE TABLE rental_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rental_id       UUID NOT NULL REFERENCES rentals(id) ON DELETE CASCADE,
    old_status      rental_status,
    new_status      rental_status NOT NULL,
    changed_by      UUID REFERENCES users(id),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── RENTAL PHOTOS (check-in / check-out / damage) ─────

CREATE TABLE rental_photos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rental_id       UUID NOT NULL REFERENCES rentals(id) ON DELETE CASCADE,
    photo_type      rental_photo_type NOT NULL,
    url             TEXT NOT NULL,
    caption         VARCHAR(300),
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── RENTAL EXTENSIONS ──────────────────────────────────

CREATE TABLE rental_extensions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rental_id       UUID NOT NULL REFERENCES rentals(id) ON DELETE CASCADE,
    old_end_date    DATE NOT NULL,
    new_end_date    DATE NOT NULL,
    additional_units INT NOT NULL,
    additional_cents BIGINT NOT NULL,
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 8 — Inventory & Calibration

```sql
-- ─── INVENTORY UNITS (individual physical items) ────────

CREATE TABLE inventory_units (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    product_id      UUID NOT NULL REFERENCES products(id),
    variant_id      UUID REFERENCES product_variants(id),
    serial_number   VARCHAR(100),
    asset_tag       VARCHAR(100),
    condition       unit_condition NOT NULL DEFAULT 'new',
    is_available    BOOLEAN NOT NULL DEFAULT TRUE,          -- not rented / not in repair
    purchase_date   DATE,
    purchase_cost_cents BIGINT,
    notes           TEXT,
    last_calibration_date DATE,
    next_calibration_date DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- Add FK from rentals to inventory_units
ALTER TABLE rentals
    ADD CONSTRAINT fk_rental_inventory_unit
    FOREIGN KEY (inventory_unit_id) REFERENCES inventory_units(id) ON DELETE SET NULL;

-- ─── CALIBRATION RECORDS ────────────────────────────────

CREATE TABLE calibration_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id         UUID NOT NULL REFERENCES inventory_units(id) ON DELETE CASCADE,
    calibrated_at   DATE NOT NULL,
    next_due        DATE NOT NULL,
    performed_by    VARCHAR(200),                           -- technician name / company
    certificate_url TEXT,
    notes           TEXT,
    recorded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 9 — Payments & Deposits

```sql
-- ─── PAYMENTS ───────────────────────────────────────────

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Polymorphic: either order or rental
    order_id        UUID REFERENCES orders(id) ON DELETE SET NULL,
    rental_id       UUID REFERENCES rentals(id) ON DELETE SET NULL,
    user_id         UUID NOT NULL REFERENCES users(id),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    amount_cents    BIGINT NOT NULL,
    method          payment_method NOT NULL,
    status          payment_status NOT NULL DEFAULT 'pending',
    -- Payment gateway references
    gateway         VARCHAR(50),                            -- 'stripe', 'paypal', etc.
    gateway_payment_id VARCHAR(500),
    gateway_response JSONB,
    refunded_cents  BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    CHECK (
        (order_id IS NOT NULL AND rental_id IS NULL) OR
        (order_id IS NULL AND rental_id IS NOT NULL)
    )
);

-- ─── DEPOSITS (rental security deposits) ────────────────

CREATE TABLE deposits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rental_id       UUID NOT NULL REFERENCES rentals(id) ON DELETE CASCADE,
    payment_id      UUID REFERENCES payments(id),
    amount_cents    BIGINT NOT NULL,
    status          deposit_status NOT NULL DEFAULT 'held',
    charged_cents   BIGINT NOT NULL DEFAULT 0,             -- damage charge amount
    released_at     TIMESTAMPTZ,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 10 — Invoicing & SDI

```sql
-- ─── INVOICE SEQUENCES (per-branch gapless) ────────────
-- Format: YEAR/BRANCH_CODE/SEQUENCE

CREATE TABLE invoice_sequences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id) ON DELETE CASCADE,
    year            INT NOT NULL,
    last_sequence   INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (branch_id, year)
);

-- ─── INVOICES ───────────────────────────────────────────

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID NOT NULL REFERENCES branches(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    order_id        UUID REFERENCES orders(id) ON DELETE SET NULL,
    rental_id       UUID REFERENCES rentals(id) ON DELETE SET NULL,
    -- Fiscal number: YEAR/BRANCH_CODE/SEQUENCE
    invoice_number  VARCHAR(30) NOT NULL UNIQUE,
    invoice_type    invoice_type NOT NULL,
    -- Customer fiscal data snapshot (frozen at invoice time)
    customer_fiscal_snapshot JSONB NOT NULL,
    -- Amounts
    subtotal_cents  BIGINT NOT NULL,
    vat_cents       BIGINT NOT NULL,
    total_cents     BIGINT NOT NULL,
    -- SDI (Sistema di Interscambio)
    sdi_status      sdi_status NOT NULL DEFAULT 'draft',
    sdi_identifier  VARCHAR(100),                          -- SDI response ID
    sdi_xml_url     TEXT,                                  -- signed XML file URL
    sdi_sent_at     TIMESTAMPTZ,
    sdi_response    JSONB,
    -- Credit note reference
    is_credit_note  BOOLEAN NOT NULL DEFAULT FALSE,
    credit_note_for UUID REFERENCES invoices(id) ON DELETE SET NULL,
    -- PDF
    pdf_url         TEXT,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── INVOICE LINE ITEMS ─────────────────────────────────

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    description     VARCHAR(500) NOT NULL,
    quantity        INT NOT NULL DEFAULT 1,
    unit_price_cents BIGINT NOT NULL,
    vat_rate        NUMERIC(5,2) NOT NULL,
    line_total_cents BIGINT NOT NULL,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 11 — Wishlist

```sql
CREATE TABLE wishlists (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE wishlist_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    wishlist_id     UUID NOT NULL REFERENCES wishlists(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    notify_on_available BOOLEAN NOT NULL DEFAULT FALSE,    -- "notify me" feature
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (wishlist_id, product_id)
);
```

---

## Module 12 — Notifications & Push Tokens

```sql
-- ─── NOTIFICATIONS ──────────────────────────────────────

CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    channel         notification_channel NOT NULL DEFAULT 'in_app',
    title           VARCHAR(300) NOT NULL,
    body            TEXT NOT NULL,
    data            JSONB,                                 -- action URL, deep link, etc.
    is_read         BOOLEAN NOT NULL DEFAULT FALSE,
    read_at         TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── PUSH TOKENS (FCM / APNs) ───────────────────────────

CREATE TABLE push_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token           TEXT NOT NULL UNIQUE,
    platform        VARCHAR(10) NOT NULL,                   -- 'ios', 'android', 'web'
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 13 — Support Tickets

```sql
CREATE TABLE support_tickets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_id       UUID REFERENCES branches(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    ticket_number   VARCHAR(30) NOT NULL UNIQUE,
    subject         VARCHAR(300) NOT NULL,
    status          ticket_status NOT NULL DEFAULT 'open',
    priority        ticket_priority NOT NULL DEFAULT 'medium',
    assigned_to     UUID REFERENCES users(id),
    -- Related entities (optional)
    order_id        UUID REFERENCES orders(id) ON DELETE SET NULL,
    rental_id       UUID REFERENCES rentals(id) ON DELETE SET NULL,
    resolved_at     TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE ticket_messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ticket_id       UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
    sender_id       UUID NOT NULL REFERENCES users(id),
    body            TEXT NOT NULL,
    attachments     JSONB,                                 -- array of file URLs
    is_internal     BOOLEAN NOT NULL DEFAULT FALSE,        -- staff-only note
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 14 — Blog / Magazine

```sql
CREATE TABLE blog_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(200) NOT NULL,
    slug            VARCHAR(200) NOT NULL UNIQUE,
    description     TEXT,
    sort_order      INT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE blog_tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL UNIQUE,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE blog_posts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    author_id       UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500) NOT NULL,
    slug            VARCHAR(500) NOT NULL UNIQUE,
    excerpt         TEXT,
    body            TEXT NOT NULL,                          -- rich text
    featured_image_url TEXT,
    status          blog_post_status NOT NULL DEFAULT 'draft',
    published_at    TIMESTAMPTZ,
    -- SEO
    seo_title       VARCHAR(200),
    seo_description VARCHAR(500),
    -- Full-text search
    search_vector   TSVECTOR,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE blog_post_categories (
    post_id         UUID NOT NULL REFERENCES blog_posts(id) ON DELETE CASCADE,
    category_id     UUID NOT NULL REFERENCES blog_categories(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, category_id)
);

CREATE TABLE blog_post_tags (
    post_id         UUID NOT NULL REFERENCES blog_posts(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES blog_tags(id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Blog search trigger
CREATE OR REPLACE FUNCTION blog_posts_search_vector_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('italian', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('italian', COALESCE(NEW.excerpt, '')), 'B') ||
        setweight(to_tsvector('italian', COALESCE(NEW.body, '')), 'C');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_blog_posts_search_vector
    BEFORE INSERT OR UPDATE OF title, excerpt, body
    ON blog_posts
    FOR EACH ROW EXECUTE FUNCTION blog_posts_search_vector_trigger();
```

---

## Module 15 — Reviews & Q&A

```sql
-- ─── PRODUCT REVIEWS ────────────────────────────────────

CREATE TABLE product_reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,  -- NULL for imported
    branch_id       UUID REFERENCES branches(id),
    source          review_source NOT NULL DEFAULT 'internal',
    external_id     VARCHAR(200),                          -- Trustpilot/Google review ID
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title           VARCHAR(300),
    body            TEXT,
    author_name     VARCHAR(200),                          -- for imported reviews
    is_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    is_published    BOOLEAN NOT NULL DEFAULT TRUE,
    admin_reply     TEXT,
    admin_reply_by  UUID REFERENCES users(id),
    admin_reply_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

-- ─── PRODUCT Q&A ────────────────────────────────────────

CREATE TABLE product_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    question        TEXT NOT NULL,
    is_published    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE TABLE product_answers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES product_questions(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),    -- staff or customer
    answer          TEXT NOT NULL,
    is_official     BOOLEAN NOT NULL DEFAULT FALSE,        -- staff answer
    is_published    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);
```

---

## Module 16 — GDPR & Communication Preferences

```sql
-- ─── GDPR REQUESTS ──────────────────────────────────────

CREATE TABLE gdpr_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    request_type    gdpr_request_type NOT NULL,
    status          gdpr_request_status NOT NULL DEFAULT 'pending',
    processed_by    UUID REFERENCES users(id),
    data_export_url TEXT,                                   -- signed URL for data download
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ─── COMMUNICATION PREFERENCES ──────────────────────────

CREATE TABLE communication_preferences (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    email_marketing BOOLEAN NOT NULL DEFAULT FALSE,
    sms_marketing   BOOLEAN NOT NULL DEFAULT FALSE,
    push_marketing  BOOLEAN NOT NULL DEFAULT FALSE,
    email_orders    BOOLEAN NOT NULL DEFAULT TRUE,
    sms_orders      BOOLEAN NOT NULL DEFAULT TRUE,
    push_orders     BOOLEAN NOT NULL DEFAULT TRUE,
    email_rentals   BOOLEAN NOT NULL DEFAULT TRUE,
    sms_rentals     BOOLEAN NOT NULL DEFAULT TRUE,
    push_rentals    BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Module 17 — Audit Log

```sql
-- ─── AUDIT LOG (immutable) ──────────────────────────────

CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID REFERENCES users(id) ON DELETE SET NULL,
    branch_id       UUID REFERENCES branches(id) ON DELETE SET NULL,
    action          VARCHAR(100) NOT NULL,                  -- e.g. 'product.update', 'role.assign'
    resource_type   VARCHAR(100) NOT NULL,                  -- e.g. 'product', 'order', 'role'
    resource_id     UUID,
    old_data        JSONB,
    new_data        JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition audit_logs by month for performance
-- (recommend implementing with pg_partman in production)
```

---

## Indexes & Performance Notes

```sql
-- ═══════════════════════════════════════════════════════════
-- INDEXES
-- ═══════════════════════════════════════════════════════════

-- Branch lookups
CREATE INDEX idx_branches_location ON branches USING GIST (location);
CREATE INDEX idx_branches_active ON branches (is_active) WHERE deleted_at IS NULL;

-- Users
CREATE INDEX idx_users_email ON users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_type ON users (user_type) WHERE deleted_at IS NULL;

-- Auth providers
CREATE INDEX idx_user_auth_providers_user ON user_auth_providers (user_id);

-- RBAC
CREATE INDEX idx_user_branch_roles_user ON user_branch_roles (user_id);
CREATE INDEX idx_user_branch_roles_branch ON user_branch_roles (branch_id);
CREATE INDEX idx_role_permissions_role ON role_permissions (role_id);

-- Products
CREATE INDEX idx_products_search ON products USING GIN (search_vector);
CREATE INDEX idx_products_type ON products (product_type) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_active ON products (is_active) WHERE deleted_at IS NULL;
CREATE INDEX idx_product_categories_product ON product_categories (product_id);
CREATE INDEX idx_product_categories_category ON product_categories (category_id);
CREATE INDEX idx_product_media_product ON product_media (product_id);
CREATE INDEX idx_product_variants_product ON product_variants (product_id);

-- Branch overrides
CREATE INDEX idx_branch_product_overrides_branch ON branch_product_overrides (branch_id);
CREATE INDEX idx_branch_product_overrides_product ON branch_product_overrides (product_id);

-- Orders
CREATE INDEX idx_orders_branch ON orders (branch_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_user ON orders (user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_orders_status ON orders (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_order_items_order ON order_items (order_id);

-- Rentals
CREATE INDEX idx_rentals_branch ON rentals (branch_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_rentals_user ON rentals (user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_rentals_status ON rentals (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_rentals_dates ON rentals (start_date, expected_end_date) WHERE deleted_at IS NULL;
CREATE INDEX idx_rentals_unit ON rentals (inventory_unit_id);

-- Inventory
CREATE INDEX idx_inventory_units_branch ON inventory_units (branch_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_inventory_units_product ON inventory_units (product_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_inventory_units_available ON inventory_units (branch_id, product_id, is_available)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_inventory_calibration_due ON inventory_units (next_calibration_date)
    WHERE deleted_at IS NULL AND next_calibration_date IS NOT NULL;

-- Payments
CREATE INDEX idx_payments_order ON payments (order_id) WHERE order_id IS NOT NULL;
CREATE INDEX idx_payments_rental ON payments (rental_id) WHERE rental_id IS NOT NULL;
CREATE INDEX idx_payments_user ON payments (user_id);

-- Invoices
CREATE INDEX idx_invoices_branch ON invoices (branch_id);
CREATE INDEX idx_invoices_user ON invoices (user_id);
CREATE INDEX idx_invoices_sdi_status ON invoices (sdi_status);
CREATE INDEX idx_invoices_issued ON invoices (issued_at);

-- Notifications
CREATE INDEX idx_notifications_user_unread ON notifications (user_id, is_read)
    WHERE is_read = FALSE;

-- Support tickets
CREATE INDEX idx_tickets_branch ON support_tickets (branch_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_tickets_user ON support_tickets (user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_tickets_status ON support_tickets (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_tickets_assigned ON support_tickets (assigned_to) WHERE deleted_at IS NULL;

-- Blog
CREATE INDEX idx_blog_posts_search ON blog_posts USING GIN (search_vector);
CREATE INDEX idx_blog_posts_status ON blog_posts (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_blog_posts_author ON blog_posts (author_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_blog_posts_published ON blog_posts (published_at DESC)
    WHERE status = 'published' AND deleted_at IS NULL;

-- Reviews
CREATE INDEX idx_reviews_product ON product_reviews (product_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_reviews_published ON product_reviews (product_id, is_published)
    WHERE deleted_at IS NULL;

-- Audit log
CREATE INDEX idx_audit_logs_user ON audit_logs (user_id);
CREATE INDEX idx_audit_logs_resource ON audit_logs (resource_type, resource_id);
CREATE INDEX idx_audit_logs_created ON audit_logs (created_at DESC);
CREATE INDEX idx_audit_logs_branch ON audit_logs (branch_id) WHERE branch_id IS NOT NULL;
```

---

## `updated_at` Auto-Trigger

```sql
-- Generic trigger function for updated_at
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply to all tables with updated_at
-- (generate these programmatically in migration scripts)
-- Example:
-- CREATE TRIGGER trg_set_updated_at
--     BEFORE UPDATE ON branches
--     FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## Entity-Relationship Summary

```
branches ─────────────┬─── branch_operating_hours
                      ├─── branch_product_overrides ──── products
                      ├─── orders ─── order_items
                      ├─── rentals ─── rental_photos, rental_extensions
                      ├─── inventory_units ─── calibration_records
                      ├─── invoices ─── invoice_line_items
                      ├─── invoice_sequences
                      ├─── support_tickets ─── ticket_messages
                      └─── audit_logs

users ────────────────┬─── user_auth_providers
                      ├─── customer_profiles
                      ├─── user_addresses
                      ├─── user_branch_roles ──── roles ──── role_permissions ──── permissions
                      ├─── carts ─── cart_items
                      ├─── orders
                      ├─── rentals
                      ├─── payments
                      ├─── wishlists ─── wishlist_items
                      ├─── notifications
                      ├─── push_tokens
                      ├─── support_tickets
                      ├─── blog_posts
                      ├─── product_reviews
                      ├─── product_questions ─── product_answers
                      ├─── gdpr_requests
                      └─── communication_preferences

products ─────────────┬─── product_categories ──── categories (self-referencing)
                      ├─── product_media
                      ├─── product_variants
                      ├─── product_specs
                      ├─── product_rental_tiers
                      ├─── branch_product_overrides
                      ├─── inventory_units
                      ├─── product_reviews
                      ├─── product_questions
                      └─── wishlist_items

blog_posts ───────────┬─── blog_post_categories ──── blog_categories
                      └─── blog_post_tags ──── blog_tags
```

---

## Table Count Summary

| Module | Tables | Description |
|---|---|---|
| Branches | 2 | branches, operating_hours |
| Users & Auth | 4 | users, auth_providers, customer_profiles, addresses |
| RBAC | 4 | permissions, roles, role_permissions, user_branch_roles |
| Products | 7 | categories, products, media, variants, specs, rental_tiers, branch_overrides |
| Cart | 2 | carts, cart_items |
| Orders | 3 | orders, order_items, status_history |
| Rentals | 4 | rentals, status_history, photos, extensions |
| Inventory | 2 | inventory_units, calibration_records |
| Payments | 2 | payments, deposits |
| Invoicing | 3 | invoices, line_items, sequences |
| Wishlist | 2 | wishlists, wishlist_items |
| Notifications | 2 | notifications, push_tokens |
| Support | 2 | tickets, ticket_messages |
| Blog | 5 | posts, categories, tags, post_categories, post_tags |
| Reviews & Q&A | 3 | reviews, questions, answers |
| GDPR | 2 | gdpr_requests, communication_preferences |
| Audit | 1 | audit_logs |
| **Total** | **50** | |

---

## Key Design Decisions

1. **Invoice type auto-detection**: `customer_profiles.invoice_type` is derived from which fiscal fields are populated — `partita_iva` → B2B, `codice_univoco` → PA, `eu_vat_number` → Intra-EU, otherwise → B2C. Enforce via application logic or a trigger.

2. **Gapless invoice numbering**: Use `SELECT ... FOR UPDATE` on `invoice_sequences` within a transaction to guarantee gapless sequential numbers per branch per year.

3. **Order/rental snapshots**: Product name, price, and address are snapshotted into order/invoice records at creation time. This ensures historical accuracy even if master data changes later.

4. **Deposit flow**: Deposit is `held` (authorized but not captured) at booking → `released` when equipment is returned in good condition → `partially_charged` or `fully_charged` if damage is found.

5. **Multi-branch users**: `user_branch_roles` allows a single admin user to have different roles across different branches with a UNIQUE constraint on `(user_id, branch_id)`.

6. **PostGIS for store locator**: `branches.location` uses `GEOGRAPHY(POINT, 4326)` for accurate distance calculations. Drive-time requires an external routing API (e.g., OSRM, Google Directions) — the DB stores coordinates, the app layer computes drive-time.

7. **Blog writer isolation**: "Writer" role has `blog_posts:write_own` permission. The application layer checks `blog_posts.author_id = current_user.id` for write operations.

8. **Audit log partitioning**: Recommend partitioning `audit_logs` by month using `pg_partman` for production — this table grows fastest.
