<div align="center">

# 🏪 HengPOS — Multi-Tenant Cloud Micro-POS SaaS

**A lightweight, high-velocity Point of Sale (POS) platform engineered for small businesses, cafes, and retail shops.**  
Designed for zero-friction cashier workflows, instant PromptPay QR payments, real-time shift reconciliation, and multi-tenant SaaS scalability.

[![Framework](https://img.shields.io/badge/Frontend%20%26%20API-Next.js%20App%20Router-black?style=flat-square&logo=next.js)](https://nextjs.org/)
[![ORM](https://img.shields.io/badge/ORM-Prisma%208%20(Contract%20Engine)-2D3748?style=flat-square&logo=prisma)](https://www.prisma.io/)
[![Database](https://img.shields.io/badge/Database-PostgreSQL-336791?style=flat-square&logo=postgresql)](https://www.postgresql.org/)
[![Style](https://img.shields.io/badge/Styling-Tailwind%20CSS-38bdf8?style=flat-square&logo=tailwindcss)](https://tailwindcss.com/)
[![Status](https://img.shields.io/badge/Status-Active%20MVP%20Development-success?style=flat-square)](#-core-modules--feature-matrix)

[System Architecture](#-system-architecture) •
[Developer Quick Guide](#-developer-quick-guide-start-here) •
[Core Modules](#-core-modules--feature-matrix) •
[Engineering Rationale](#-engineering-rationale) •
[Getting Started](#-getting-started)

---

</div>

## 📌 Executive Summary

**HengPOS (เฮงพอส)** is a commercial multi-tenant Cloud POS SaaS tailored specifically for small merchants, street food vendors, beverage stalls, and neighborhood retail shops in Thailand.

Rather than bloated enterprise ERP systems that overwhelm small shop owners, **HengPOS** focuses on the critical path of daily store operations:

1. **Sub-Second Cashier Terminal**: Touch-friendly UI with instant modifier selection (sweetness levels, toppings, add-ons) and barcode scanner support.
2. **Zero-Fee & Automated QR Payments**: Native EMVCo **PromptPay Dynamic QR** generation alongside webhook-driven Payment Gateway / Slip Verification integrations.
3. **Cash Drawer & Shift Accountability**: Strict shift opening/closing protocols comparing expected vs. actual cash to eliminate cashier discrepancies.
4. **Immutable Order Snapshots**: Financial records utilize strict `Numeric(10, 2)` precision and value-object snapshots so historical reports remain 100% accurate even when menu prices change.

---

## 🧭 Developer Quick Guide (Start Here)

If you are navigating or reviewing this repository, here is where to look based on your focus area:

| Focus Area | Target Path | What to Look For |
| :--- | :--- | :--- |
| **Database Contract (Prisma 8)** | [`src/prisma/contract.prisma`](./src/prisma/contract.prisma) | Multi-tenant schema (`Shop`), `Numeric(10,2)` financial fields, and `SelectedModifier` value objects |
| **Database Runtime & Singleton** | [`src/prisma/db.ts`](./src/prisma/db.ts) | Prisma 8 `@prisma/orm-postgres` runtime initialization and Next.js hot-reload singleton pattern |
| **POS Cashier Terminal UI** | [`src/app/(pos)/terminal`](./src/app) | Fast cart state management, modifier modals, queue numbering, and thermal receipt print layout |
| **Customer-Facing Display** | [`src/app/(pos)/customer-display`](./src/app) | Second-screen real-time order summary and dynamic PromptPay QR rendering |
| **Payment & Webhook Handlers** | [`src/app/api/webhooks`](./src/app) | Asynchronous payment confirmation (Omise / Stripe / Slip Verification API) |
| **Shop Owner Backoffice** | [`src/app/(dashboard)`](./src/app) | Daily sales analytics, profit margin calculation, shift audit logs, and inventory tracking |

---

## 🏛️ System Architecture

```mermaid
flowchart TD
    subgraph Storefront ["Storefront Tier (Client Devices)"]
        POS["Cashier Terminal\n(Tablet / Desktop / Thermal Printer)"]
        CFD["Customer Display\n(Dynamic QR & Live Cart)"]
        MGR["Owner Dashboard\n(Mobile / Web Analytics)"]
    end

    subgraph NextApp ["Next.js Full-Stack Application Tier"]
        SC["React Server Components\n& Server Actions"]
        API["Route Handlers\n(/api/orders, /api/webhooks)"]
    end

    subgraph PaymentExt ["External Payment Ecosystem"]
        PP["EMVCo PromptPay Engine\n(Zero-Fee Dynamic QR)"]
        PG["Payment Gateway / Slip API\n(Omise / Stripe / SlipOK)"]
    end

    subgraph DataLayer ["Persistence Tier (Prisma 8)"]
        ORM["@prisma/orm-postgres\n(Type-Safe Contract Client)"]
        DB[(PostgreSQL Database\nMulti-Tenant Schema)]
    end

    POS --> SC
    POS <--> CFD
    MGR --> SC
    POS --> PP
    PG -- "Webhook Callback" --> API
    SC & API --> ORM
    ORM --> DB
```

---

## 🗂️ Core Modules & Feature Matrix

| Module | Path / Scope | Role & Responsibility | Target User | Status |
| :--- | :--- | :--- | :--- | :--- |
| **SaaS Tenant & Auth** | `Shop`, `User` | Multi-shop isolation, subscription plan lifecycle (`MONTHLY`, `YEARLY`, `LIFETIME`), and 4-digit cashier PIN switching | Super Admin & Shop Owner | 🟢 Phase 1 (Ready) |
| **Catalog & Modifiers** | `Category`, `Product`, `ModifierGroup` | Menu management, SKU barcode lookup, cost vs. price tracking, and customizable topping groups | Shop Owner | 🟢 Phase 1 (Ready) |
| **POS Checkout Engine** | `Order`, `OrderItem` | High-speed billing, queue number generation, split payments, and historical price/cost snapshotting | Cashier | 🟢 Phase 1 (Active) |
| **QR & Payment Hub** | `Payment` | Cash change calculation, Dynamic PromptPay QR payload generation, and gateway webhook reconciliation | Cashier & Customer | 🟡 Phase 2 |
| **Shift & Cash Control** | `Shift` | Starting drawer cash record, expected cash calculation, and end-of-day variance auditing | Cashier & Shop Owner | 🟡 Phase 2 |
| **Inventory & CRM** | `StockLog`, `Customer` | Real-time stock deduction, low-stock alerts, and phone-number loyalty point accumulation | Shop Owner | 🟡 Phase 3 |

---

## 💡 Engineering Rationale

### Why This Stack & Architecture for a Micro-POS?

1. **Prisma 8 Contract-First ORM (`@prisma/orm-postgres`)**:
   - Eliminates heavy Rust engine binaries for ultra-fast serverless cold starts and minimal memory footprint.
   - Leverages native **Value Objects (`type SelectedModifier`)** stored as strongly-typed `jsonb` arrays inside `OrderItem`, avoiding expensive relational joins when rendering thousands of historical receipts.
2. **Strict Financial Precision (`Numeric(10, 2)`)**:
   - Floating-point math is never used for currency. All monetary columns (`price`, `cost`, `subTotal`, `netTotal`, `cashReceived`) use exact PostgreSQL `NUMERIC(10, 2)` types.
3. **Multi-Tenant by Design (`shopId` Isolation)**:
   - Every core entity is scoped to a `Shop` tenant with composite indexes (`@@index([shopId, createdAt])`, `@@index([shopId, status])`), allowing a single PostgreSQL instance to serve hundreds of small merchants at near-zero infrastructure cost.
4. **Hybrid Payment Strategy for Thai SMEs**:
   - Small merchants are sensitive to MDR transaction fees. HengPOS supports **Direct PromptPay Dynamic QR** (0% fee) alongside optional **Automated Slip Verification / Payment Gateways** for high-volume stores.

---

## 🏪 Domain & Feature Scope

- **Smart Product Modifiers**: Configure required/optional add-on groups (e.g., *"Sweetness Level"*, *"Extra Toppings"*) with per-item extra pricing.
- **Immutable Receipt Snapshots**: `OrderItem` captures `productName`, `unitPrice`, and `unitCost` at the exact moment of sale so future catalog edits never corrupt past financial reports.
- **Multi-Channel Tender Support**: Record transactions across `CASH`, `PROMPTPAY_MANUAL`, `PROMPTPAY_AUTO`, `CREDIT_CARD`, and `DELIVERY_APP` (Grab / LINE MAN / ShopeeFood).
- **Thermal Printer Ready**: Optimized layout for 58mm and 80mm receipt printers with customizable shop headers, tax IDs, and footer messages.

---

## 🚀 Getting Started

### Prerequisites

- **Node.js**: `>= 20.x`
- **PostgreSQL**: `>= 15.x` (Local Docker, Supabase, or Neon)
- **Package Manager**: `npm` or `pnpm`

### 1. Clone & Install Dependencies

```bash
git clone https://github.com/Sitepow/heng-pos.git
cd heng-pos
npm install
```

### 2. Configure Environment Variables

Create a `.env` file in the root directory:

```env
DATABASE_URL="postgresql://postgres:password@localhost:5432/hengpos?schema=public"
```

### 3. Emit Prisma 8 Contract & Initialize Database

```bash
# 1. Compile src/prisma/contract.prisma into contract.json & TypeScript definitions
npx prisma contract emit

# 2. Push the schema to PostgreSQL
npx prisma db init
```

### 4. Run the Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser to launch **HengPOS**.

---

## 📜 License

This project is proprietary software developed for the **HengPOS SaaS Platform**. All rights reserved.
