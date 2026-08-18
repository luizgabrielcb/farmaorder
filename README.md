<div align="center">

## 💊 FarmaBook

**The pharmacy's order and shortage notebook, now digital.**

Order management system for pharmacies — orders, shortages, compoundings, and prescriptions.

[![CI](https://github.com/luizgabrielcb/farmabook/actions/workflows/ci.yml/badge.svg)](https://github.com/luizgabrielcb/farmabook/actions/workflows/ci.yml)
![Tests](https://img.shields.io/badge/tests-459%20passing-success)
![Coverage](https://img.shields.io/badge/coverage-94%25%20line-success)
![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5-6DB33F?logo=springboot&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-17-4169E1?logo=postgresql&logoColor=white)

</div>

## 📑 Table of Contents

- [Demo](#-demo)
- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [Features](#-features)
- [Domains in Detail](#-domains-in-detail)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Notable Design Decisions](#-notable-design-decisions)
- [Tests and Coverage](#-tests-and-coverage)
- [Getting Started](#-getting-started)
- [Running in Development](#-running-in-development)
- [How to Run at the Pharmacy](#-how-to-run-at-the-pharmacy)
- [REST API](#-rest-api)
- [Repository Structure](#-repository-structure)
- [Code Conventions](#-code-conventions)
- [Author](#-author)

---

## 📸 Demo

<img width="1919" height="999" alt="Screenshot 2026-06-17 at 20 46 37" src="https://github.com/user-attachments/assets/8ceecd25-479f-4d9b-b46b-0f152b887b94" />
<img width="1920" height="998" alt="Screenshot 2026-06-17 at 20 48 19" src="https://github.com/user-attachments/assets/5902b754-7b97-4771-807b-07edb589715c" />
<img width="1920" height="998" alt="Screenshot 2026-06-17 at 20 47 23" src="https://github.com/user-attachments/assets/c249564c-3f05-4106-8a8b-7c7a5660e06a" />

---

## 🩹 The Problem

This project was born out of a real pain point, lived firsthand during nearly **4 years working at a pharmacy**.

The pharmacy had two sacred notebooks at the counter: the **order notebook** (what the customer requested and the store needs to buy from the distributor) and the **shortage notebook** (what ran out on the shelf and needs restocking). And every day those notebooks take their toll:

- ✍️ **Manual, error-prone entries** — in the rush at the counter, it's easy to forget to log an order, or to forget to **cross out** one that's already been fulfilled. The notebook becomes a mix of pending and resolved requests, with no clarity whatsoever.
- 🔍 **Searching is a nightmare** — when a customer comes back **a month later** asking "what did I order again?", it's a page-by-page hunt trying to decipher whoever's handwriting.
- 🧑‍🤝‍🧑 **Nobody knows who did what** — who wrote it down? who placed the order with the distributor? who delivered it? The notebook doesn't answer. When something goes wrong, there's no one to ask.
- 📞 **Notifying the customer is tedious** — when the order arrives, someone has to remember to call, dial the number, figure out the message... and it often just doesn't happen.
- 🗑️ **History gets lost** — notebooks run out, get wet, get lost. And with them, every record of who bought what.

## ✅ The Solution

**FarmaBook** replaces those notebooks with a web system designed for the counter: fast to operate, with **permanent and searchable history**, **authorship tracking** at every step, and **automatic customer notifications via WhatsApp**.

Every order and every shortage stops being a scribble and becomes a record with a **well-defined lifecycle** — you can always see where each request stands, who handled it, and when. Nothing is truly deleted; even "removed" entries remain in the history. And when a product arrives, one click generates the WhatsApp link with the message ready to send.

---

## ✨ Features

- 🧾 **Orders with independent items** — an order groups multiple products, and each item has its own lifecycle (`PENDING → ORDERED → RECEIVED → DELIVERED`).
- 👣 **Authorship tracking at every step** — who ordered from the distributor, who received it, who delivered it and when: all automatically stamped with the responsible user.
- 💬 **Automatic WhatsApp on arrival** — when an order (or compounding) is marked as received, the system builds a `wa.me` link with the message ready to go, a time-appropriate greeting, and the number already normalized.
- 💰 **Per-item payment control** — including a "put it on the tab" flow: `TO_PAY → MAKE_NOTE → NOTED`, or straight to `PAID`.
- 📉 **Stock shortages** — log what ran out on the shelf, groupable into restocking requests by sales rep/distributor.
- 🔐 **PIN login** — optimized for counter operation; every action automatically records the operator.
- 🗂️ **Permanent, rename-proof history** — nothing is truly deleted, and customer/user/distributor names are frozen in the record, so the past survives even if a record is later renamed.

---

## 🧩 Domains in Detail

The analysis below reflects the actual model in the code (entities, statuses, and rules).

### 🧾 Orders (`order`)
The core of the system. An `Order` belongs to a customer and holds a list of `OrderItem`.

- **Item** → product, category, quantity, distributor, price, and its own status.
- **Item status** → `PENDING → ORDERED → RECEIVED → DELIVERED` (order matters — it's used in calculations).
- **Order status** → calculated as the **lowest status** among its items. Marked all items as received? The order becomes "received". Never typed manually.
- **Authorship stamp** → each transition records the trio *who / name / when* (`orderedBy…`, `receivedBy…`, `deliveredBy…`). When an item is **regressed** (e.g. received → ordered), the corresponding stamp is cleared.
- **Notification** → the **first** time an order reaches `RECEIVED`, a WhatsApp notification is generated and `notifiedAt` is set.
- **Payment** → each item has a payment status (`TO_PAY → MAKE_NOTE → NOTED`, or `PAID`); the order's payment status is the lowest among its items. `NOTED` and `PAID` are terminal states.
- **Bulk transitions** (`mark-as-*` on the order) are **lenient**: ineligible items are skipped. **Individual** transitions (on the item) are **strict**: they return `409 Conflict`.
- **Immutability** → `DELIVERED` items/orders cannot be modified, and an order with a delivered item cannot be deleted.

### 📉 Shortages (`shortage`)
`Shortage` records a product that ran out (product, category, quantity, cost price), with status `PENDING → ORDERED`. Multiple shortages can be grouped into a `ShortageOrder` — a restocking request directed at a **sales rep** and a **distributor**, also with a `PENDING → ORDERED` cycle and authorship tracking.

### 💬 Notifications (`notification`)
Every notification takes a snapshot of the phone number, name, message, and link at the time of sending, and references the order **or** compounding that triggered it. The link is a `wa.me` with the URL-encoded message, a time-based greeting (timezone `America/Sao_Paulo`), and the normalized phone number (digits only, `55` prefix). Notifications can be **resent** — which creates a new record, preserving the original.

### 👤 Users and Authentication (`auth`)
`User` has a name, PIN (hashed with **BCrypt**), role (`ADMIN` / `SELLER`), and an `active` flag. Authentication is deliberately minimal: every protected request sends the `X-Auth-Pin` header, and `AuthService` matches the PIN against active users — the resolved user becomes the "actor" stamped in the audit trail. This design is discussed under [Notable Design Decisions](#-notable-design-decisions).

### 🧱 Supporting Records (`customer`, `distributor`, `catalog`, `shared`)
`Customer` (name + WhatsApp phone number), `Distributor` (name) are the supporting registries. `Category` is the product category enum (`MEDICATIONS`, `PERFUMERY`, `SUPPLEMENTS`, `NATURAL_PRODUCTS`, `OTHER`). Everything inherits from `Auditable` (`createdAt`, `updatedAt`, `deletedAt`).

---

## 🏛 Architecture

```
┌──────────────┐      HTTP /api      ┌──────────────┐      JDBC      ┌──────────────┐
│   Frontend   │ ──────────────────► │   Backend    │ ─────────────► │  PostgreSQL  │
│ React + Vite │   (nginx proxy)     │ Spring Boot  │                │      17      │
│   :80        │                     │    :8080     │                │    :5432     │
└──────────────┘                     └──────────────┘                └──────────────┘
     SPA served via nginx,               REST API by domain,            schema 100%
   which proxies /api → :8080            ddl-auto: validate          managed by Flyway
```

The backend is organized **by domain** (not by technical layer), mirroring the areas of a pharmacy:

```
br.com.luizgabriel.farmabook
├── order          # orders and their items  ← core
├── shortage       # shortages + restocking requests
├── compounding    # compoundings  └─ pharmacy  (compounding pharmacies)
├── prescription   # prescriptions and their items (batch/expiry)
├── notification   # WhatsApp notifications (wa.me)
├── customer       # customers (with phone number)
├── distributor    # distributors
├── catalog        # product categories (enum)
├── auth           # users + PIN authentication
├── shared         # Auditable (createdAt/updatedAt/deletedAt)
├── exception      # exceptions + GlobalExceptionHandler
└── config         # JPA Auditing, BCrypt
```

Each domain follows the same structure: `Entity`, `Repository`, `Service` (business logic), `Controller` (REST), `Mapper` (MapStruct), and `dto/` (request/response records).

---

## 🧰 Tech Stack

### Backend
| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5 (Web, Data JPA, Validation) |
| Database | PostgreSQL 17 |
| Migrations | Flyway (versioned) |
| DTO↔Entity Mapping | MapStruct |
| Boilerplate | Lombok |
| PIN Hashing | Spring Security Crypto (BCrypt) |
| Testing | JUnit 5, Mockito, Testcontainers, REST Assured, JSON Unit |

### Frontend
| Layer | Technology |
|---|---|
| Language | TypeScript |
| Framework | React 19 |
| Build | Vite |
| Styling | TailwindCSS 4 |
| Components | Radix UI + lucide-react |
| Server State | TanStack Query |
| Routing | React Router 7 |
| HTTP | Axios |

### Infrastructure
- **Docker & Docker Compose** — `postgres` + `backend` + `frontend`
- **nginx** — serves the SPA and reverse-proxies `/api` (avoids CORS)
- **GitHub Actions** — CI runs `mvnw clean verify` on every push/PR to `main`

> The interface is entirely in **Portuguese**, with keyboard shortcuts designed for counter use (e.g. **F2** Shortages, **F3** Orders, **F4** Customers).

---

## 💡 Notable Design Decisions

> This section covers the **why** behind each choice — the *how* is in [Domains in Detail](#-domains-in-detail).

- **Derived status, never typed.** The order status is always the lowest status among its items, recalculated on every change. This makes it **impossible** to have the "inconsistent notebook" problem that existed on paper — you can't mark an order as delivered if an item is still pending.
- **Intentional denormalization for history.** Customer/user/distributor names are copied into the record. Renaming a registry entry does **not** rewrite the past — history must reflect what was true at the time, not the current state.
- **Soft delete everywhere** (via `@SQLDelete` + `@SQLRestriction`). The pharmacy needs auditing; "deleting" only sets `deleted_at` and the data remains queryable.
- **Schema only via Flyway** (`ddl-auto: validate`). Hibernate never touches the database schema; every change is a versioned, immutable `V{n}__*.sql` migration. This yields a reproducible, PR-reviewable schema.
- **Notification as a transition side effect.** WhatsApp isn't a loose button — it's triggered by the transition to "received" itself, ensuring the customer is notified consistently without relying on someone remembering to do it.
- **PIN authentication, designed for internal use.** FarmaBook was conceived as a counter tool on a **local store network** (the same model as a supermarket POS): a short PIN optimizes operator switching during service, and the machine sits physically behind the counter. That's why there's no session or Spring Security filter — just the `X-Auth-Pin` header resolved on each request. **If the system were exposed to the public internet**, this design would be hardened: replacing the PIN with a session token after the first login, rate limiting with lockout after failed attempts, and mandatory HTTPS. The current choice is a conscious trade-off for the deployment context, not an oversight.

---

## 🧪 Tests and Coverage

The backend has **459 automated tests**, split between **unit tests** (services, with Mockito) and **integration tests** (controllers, with Testcontainers + a real PostgreSQL instance).

| Metric | Coverage |
|---|---|
| Classes | **97%** (103/106) |
| Methods | **97%** (343/353) |
| Lines | **94%** (1624/1721) |

```bash
cd backend

./mvnw test                                       # all tests
./mvnw verify                                     # full build + tests (same as CI)
./mvnw test -Dtest=OrderServiceTest               # single class
./mvnw test -Dtest=OrderServiceTest#shouldRecalculateStatusWhenItemChanges   # single method
```

- **Unit tests** (`*ServiceTest`): JUnit 5 + Mockito + AssertJ, with in-memory fixtures. Cover the happy path and every failure branch (not found, conflict, validation), always verifying the side effect did **not** occur in error cases.
- **Integration tests** (`*ControllerTestIT`): a single shared `PostgreSQLContainer`, REST Assured, and JSON comparison with JSON Unit. State is reset between tests via `@Sql`.

The **CI pipeline** (GitHub Actions) runs `./mvnw clean verify` on every push and PR to `main`.

---

## 🚀 Getting Started

### Prerequisites
- [Docker](https://www.docker.com/) and Docker Compose
- For development: **JDK 21** and **Node.js 22+**

### Run Everything with Docker Compose
From the repository root:

```bash
docker compose up -d --build
```

| Service | URL | Description |
|---|---|---|
| `frontend` | http://localhost | React SPA served via nginx |
| `backend` | http://localhost:8080 | Spring Boot REST API |
| `postgres` | localhost:5432 | PostgreSQL 17 database |

Database credentials are read from a `.env` file at the root (not versioned). Copy the template and fill in your values:

```bash
cp .envTemplate .env
```

```bash
# .env
ENV_POSTGRES_DB=<database-name>
ENV_POSTGRES_USER=<username>
ENV_POSTGRES_PASSWORD=<strong-password>
```

---

## 🛠 Running in Development

### 1. Database (required before the backend)
```bash
docker compose up -d postgres
```

### 2. Backend
```bash
cd backend
./mvnw spring-boot:run        # applies Flyway migrations and starts the API on :8080
```
Build without tests:
```bash
./mvnw clean package -DskipTests
```

### 3. Frontend
```bash
cd frontend
npm install
npm run dev                   # Vite with HMR
```
> The HTTP client uses `baseURL: '/api'`; in production, nginx proxies it to the backend.

## 🌐 REST API

All protected endpoints require the `X-Auth-Pin` header. Listing endpoints return `Page<...>` and accept `?page`, `?size`, and `?sort=field,direction`.

### Authentication — `/auth`
| Method | Route | Description |
|---|---|---|
| `POST` | `/auth/validate-pin` | Validates a user's PIN |
| `POST` | `/auth/change-pin` | Changes the PIN |

### Orders — `/orders`
| Method | Route | Description |
|---|---|---|
| `POST` `GET` | `/orders` | Create / list (paginated) |
| `GET` `PUT` `DELETE` | `/orders/{id}` | Get / update / delete |
| `PATCH` | `/orders/{id}/mark-as-{ordered\|received\|delivered}` | Bulk transition (lenient) |
| `POST` `PUT` `DELETE` | `/orders/{id}/items[/{itemId}]` | Manage items |
| `PATCH` | `/orders/{id}/items/{itemId}/mark-as-{ordered\|received\|delivered}` | Individual transition (strict) |
| `PATCH` | `/orders/{id}/items/{itemId}/payment/mark-as-{paid\|to-pay\|make-note\|noted}` | Payment flow |

### Other Resources
| Resource | Base | Operations |
|---|---|---|
| Users | `/users` | CRUD + `activate` / `deactivate` |
| Customers | `/customers` | CRUD |
| Shortages | `/shortages` | CRUD + `mark-as-ordered` |
| Restocking Orders | `/shortage-orders` | CRUD + `mark-as-ordered` |
| Compoundings | `/compoundings` | CRUD + lifecycle + payment |
| Compounding Pharmacies | `/compounding-pharmacies` | CRUD |
| Prescriptions | `/prescriptions` | CRUD + items + `mark-as-received` |
| Distributors | `/distributors` | CRUD |
| Notifications | `/orders/{id}/notifications`, `/notifications/compoundings/{id}`, `/notifications/{id}/resend` | Query + resend |

Errors are standardized by `GlobalExceptionHandler`: `404` (not found), `409` (state conflict), `401` (invalid PIN), `400` (validation).

---

## 📁 Repository Structure

```
farmabook/
├── backend/                  # Spring Boot API (Maven module)
│   ├── src/main/java/...      # code organized by domain
│   ├── src/main/resources/
│   │   └── db/migration/      # Flyway migrations (V1__… onwards)
│   ├── src/test/             # unit (*ServiceTest) and integration (*ControllerTestIT) tests
│   ├── Dockerfile
│   └── pom.xml
├── frontend/                 # React + Vite SPA
│   ├── src/
│   │   ├── api/              # HTTP clients per resource
│   │   ├── pages/            # pages per domain
│   │   ├── components/       # ui (Radix), layout, shared
│   │   ├── context/          # PIN, Toast, Confirm
│   │   └── lib/              # axios client + utilities
│   ├── nginx.conf            # serves the SPA + proxies /api
│   └── Dockerfile
├── docker-compose.yml        # postgres + backend + frontend
├── CLAUDE.md                 # architecture guide and conventions for contributors
└── README.md
```

## 👤 Author

**Luiz Gabriel Costa Britto**

- GitHub: [@luizgabrielcb](https://github.com/luizgabrielcb)
- LinkedIn: [luizgabrielcbtto](https://www.linkedin.com/in/luizgabrielcbtto/)
