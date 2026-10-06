# Retail Inventory and Sales Management System

> **DBMS PBL Project** — *Design and Implementation of a Database Management System for Retail Inventory and Sales Management System*

A full-stack retail store management application built on a **normalised MySQL database**.
The store manager can manage products, categories, suppliers, customers and stock, sell through
a point-of-sale (POS) screen, print invoices, track payments and view live analytics. Every
number on screen comes from SQL queries against MySQL. Nothing is hard-coded.

The **Database** page inside the app shows the DBMS side of the project: a live ER
diagram, table schemas read from `INFORMATION_SCHEMA`, 22 runnable SQL demonstration queries
and a live **transaction ROLLBACK** demonstration.

---

## Table of contents

1. [Features](#1-features)
2. [Technology stack](#2-technology-stack)
3. [Database schema](#3-database-schema)
4. [ER diagram explanation](#4-er-diagram-explanation)
5. [Installation (Windows + VS Code)](#5-installation-windows--vs-code)
6. [MySQL setup](#6-mysql-setup)
7. [Environment variables](#7-environment-variables)
8. [Running the backend](#8-running-the-backend)
9. [Running the frontend](#9-running-the-frontend)
10. [Demo credentials](#10-demo-credentials)
11. [API documentation](#11-api-documentation)
12. [Important SQL queries](#12-important-sql-queries)
13. [DBMS concepts demonstrated](#13-dbms-concepts-demonstrated)
14. [Project structure](#14-project-structure)
15. [Presentation demo script](#15-presentation-demo-script)
16. [Troubleshooting](#16-troubleshooting)
17. [Future enhancements](#17-future-enhancements)

---

## 1. Features

| Module | What it does |
|---|---|
| **Login** | Demo-level JWT authentication with three roles (Admin, Manager, Cashier). Passwords are stored as bcrypt hashes. |
| **Dashboard** | 8 KPI cards, a daily/weekly/monthly sales line chart, a sales-by-category donut, top-selling products, inventory status, best customers, recent sales and low-stock alerts. All values are computed live in SQL. |
| **Products** | Full CRUD with search, category/supplier/status filters, sortable columns, pagination, a view modal (sales stats and stock history) and validation (unique SKU, non-negative prices). |
| **Categories** | CRUD, search and product count per category. Deleting a category that still has products is blocked by `ON DELETE RESTRICT`. |
| **Suppliers** | CRUD, search, a details view and the list of products each supplier provides. |
| **Customers** | CRUD, search, purchase history, total spent, number of orders, average order and favourite products. |
| **Inventory** | Live stock with In Stock / Low Stock / Out of Stock badges (green, amber, red), filters, sorting by quantity, stock adjustment (in, out, set) and a stock-movement audit log. |
| **New Sale (POS)** | Pick a customer, search or add products, change quantities, apply a ₹ or % discount, auto-calculate GST, choose a payment method, then complete the sale **in one database transaction**. |
| **Sales History** | Search, date-range filters, payment method/status filters, a sale-detail modal, CSV export and a print-ready invoice. |
| **Invoice** | Professional, print-friendly tax invoice (*Print → Save as PDF* also works). |
| **Payments** | Payment records with Paid/Pending/Failed badges, status filters and a "mark pending as paid/failed" action, kept in sync with the sale in a transaction. |
| **Reports** | 9 reports (daily, weekly, monthly, product, category, customer, inventory, low stock, supplier), each with a date filter, chart, totals row, **CSV export** and **Print**. |
| **Database** | ER diagram, live table schemas (PK/FK/UNIQUE/CHECK/indexes), relationship list, 22 runnable SQL queries, sale-transaction SQL and a live ROLLBACK demonstration. |
| **Settings** | Store name, email, phone, address, GSTIN, currency, tax rate, low-stock threshold and invoice prefix. Stored in the database. |
| **Global** | Top-bar search (products, customers, invoices, suppliers), notifications (low stock and pending payments), toasts, confirmation dialogs, skeleton loaders, empty states, and a responsive layout with a mobile drawer. |

---

## 2. Technology stack

| Layer | Technology |
|---|---|
| Frontend | React 18, Vite 5, Tailwind CSS 3, React Router 6, Recharts, Lucide React icons |
| Backend | Node.js (18+), Express 4, `mysql2` (raw SQL with parameterised queries), `bcryptjs`, `jsonwebtoken` |
| Database | **MySQL 8.0+** (InnoDB) |
| Tooling | `concurrently` (runs API + web together), a custom DB setup script, an API test script |

> **Why raw SQL instead of an ORM?** This is a DBMS project, so every query is written in plain SQL and
> stays visible in the code. That keeps every JOIN, GROUP BY and transaction easy to explain.
> All user values are passed as `?` placeholders, which prevents SQL injection.

---

## 3. Database schema

Database name: **`retail_inventory_db`** — 11 tables + 1 view. Full DDL: [`database/schema.sql`](database/schema.sql).

| # | Table | Purpose | Key columns |
|---|---|---|---|
| 1 | `users` | Staff who log in and bill sales | **PK** `id`, UQ `email`, `role` ENUM |
| 2 | `categories` | Product categories | **PK** `id`, UQ `category_name` |
| 3 | `suppliers` | Vendors | **PK** `id`, UQ `supplier_name`, UQ `email` |
| 4 | `products` | Product catalogue | **PK** `id`, UQ `sku`, **FK** `category_id`, **FK** `supplier_id`, CHECK prices ≥ 0 |
| 5 | `inventory` | Current stock (1 row per product) | **PK** `id`, **FK+UQ** `product_id`, CHECK `quantity >= 0` |
| 6 | `customers` | Customers | **PK** `id`, UQ `phone`, UQ `email` |
| 7 | `sales` | Invoice header | **PK** `id`, UQ `invoice_no`, **FK** `customer_id`, **FK** `user_id`, CHECK `discount <= subtotal` |
| 8 | `sale_items` | Invoice lines (M:N between sales and products) | **PK** `id`, **FK** `sale_id`, **FK** `product_id`, `subtotal` = GENERATED column |
| 9 | `payments` | Payment attempts for a sale | **PK** `id`, **FK** `sale_id`, CHECK `amount > 0` |
| 10 | `stock_movements` | Audit trail of every stock change | **PK** `id`, **FK** `product_id`, `user_id`, `sale_id` |
| 11 | `settings` | Single-row store configuration | **PK** `id`, CHECK `id = 1` |
| — | `v_product_stock` *(VIEW)* | products ⋈ categories ⋈ suppliers ⋈ inventory + derived `stock_status` | — |

### Referential actions (chosen per relationship)

| Foreign key | ON DELETE | Why |
|---|---|---|
| `products.category_id → categories.id` | **RESTRICT** | A category that is still in use cannot be deleted |
| `products.supplier_id → suppliers.id` | **SET NULL** | Products stay in the catalogue if a supplier is removed |
| `inventory.product_id → products.id` | **CASCADE** | The stock row is meaningless without its product |
| `sales.customer_id → customers.id` | **SET NULL** | Sales history is kept as a "walk-in" sale |
| `sales.user_id → users.id` | **RESTRICT** | Staff who billed sales cannot be deleted |
| `sale_items.sale_id → sales.id` | **CASCADE** | Line items belong to their invoice |
| `sale_items.product_id → products.id` | **RESTRICT** | A product with sales history cannot be deleted |
| `payments.sale_id → sales.id` | **CASCADE** | Payments belong to their sale |
| `stock_movements.*` | CASCADE / SET NULL | Audit rows follow the product and keep the history |

### Normalisation (3NF)

- **1NF**: every column is atomic. A sale's products are stored as separate rows in `sale_items`, not as a list.
- **2NF**: no partial dependencies. Product details depend on `products.id`, not on part of a composite key.
- **3NF**: no transitive dependencies. Category and supplier names are stored once and referenced by FK.
- `sale_items.unit_price` is the *price at the time of sale* (a historical fact, not a duplicate).
  `sale_items.subtotal` is a **generated column**, so it can never disagree with `quantity × unit_price`.

---

## 4. ER diagram explanation

```
 categories 1 ────── N products N ────── 1 suppliers
                          │ 1
                          ├──────── 1 inventory          (1 : 1)
                          │ 1
                          └──────── N sale_items N ────── 1 sales N ────── 1 customers
                                                            │ 1   N
                                                            │     └────── 1 users
                                                            └──────── N payments
 stock_movements: N ── 1 products, N ── 1 sales, N ── 1 users   (audit trail)
```

- **Categories 1 : N Products**: each product belongs to exactly one category.
- **Suppliers 1 : N Products**: a supplier supplies many products (optional for a product).
- **Products 1 : 1 Inventory**: `inventory.product_id` is both a FK and UNIQUE.
- **Customers 1 : N Sales** and **Users 1 : N Sales**: who bought and who billed.
- **Sales 1 : N Sale_Items** and **Products 1 : N Sale_Items**: `sale_items` resolves the
  **many-to-many** relationship between sales and products.
- **Sales 1 : N Payments**: a sale can have several attempts (e.g. a failed card, then UPI).

The in-app **Database → ER Diagram** tab draws this diagram with PK/FK/UQ markers and live row
counts. Hover over any table to highlight its relationships.

---

## 5. Installation (Windows + VS Code)

### Prerequisites

| Software | Version | Download |
|---|---|---|
| Node.js (LTS) | 18 or newer | https://nodejs.org |
| MySQL Server | 8.0 or newer | https://dev.mysql.com/downloads/installer/ |
| VS Code | any | https://code.visualstudio.com |

Check them in a terminal (**VS Code → Terminal → New Terminal**):

```powershell
node -v      # v18+ 
npm -v
mysql --version   # optional; MySQL Workbench is fine too
```

### Steps

1. Open the `retail-inventory-system` folder in VS Code (**File → Open Folder…**).
2. Install all dependencies (root, server and client) with **one command**:
   ```powershell
   npm run install:all
   ```
3. Create your environment file (see [section 7](#7-environment-variables)):
   ```powershell
   copy server\.env.example server\.env
   ```
   Open `server\.env` and set `DB_PASSWORD` to **your MySQL root password**.
4. Create the database, tables and sample data:
   ```powershell
   npm run setup-db
   ```
   You should see every table with its row count, e.g. `products 32`, `sales 51`.
5. Start the backend and the frontend together:
   ```powershell
   npm run dev
   ```
6. Open **http://localhost:5173** and log in with `admin@retail.com` / `admin123`.

---

## 6. MySQL setup

1. Install **MySQL Server 8.0** with the MySQL Installer. Choose *Developer Default* or
   *Server only*, keep port **3306**, and remember the **root password** you set.
2. Make sure the service is running: press `Win + R` → `services.msc` → **MySQL80** → *Start*.
3. Database creation is automatic: `npm run setup-db` connects with the credentials from
   `server\.env`, **drops and recreates** `retail_inventory_db`, then runs
   `database/schema.sql` and `database/seed.sql`.

**Manual alternative (MySQL Workbench):** open and execute, in order:
`database/schema.sql` → `database/seed.sql`. Then `database/queries.sql` and
`database/crud_examples.sql` can be run to explore the data.

> Re-run `npm run setup-db` at any time to reset the demo data. The sample sales are dated
> relative to today, so the dashboard always shows recent activity.

---

## 7. Environment variables

File: `server/.env` (copy of [`server/.env.example`](server/.env.example)). **Never commit this file.**

| Variable | Example | Description |
|---|---|---|
| `DB_HOST` | `localhost` | MySQL host |
| `DB_PORT` | `3306` | MySQL port |
| `DB_USER` | `root` | MySQL user |
| `DB_PASSWORD` | `your_mysql_password` | MySQL password |
| `DB_NAME` | `retail_inventory_db` | Database name (created by `setup-db`) |
| `DATABASE_URL` | `mysql://root:pass@localhost:3306/retail_inventory_db` | *Optional*; overrides the `DB_*` values. URL-encode special characters (`@` → `%40`). |
| `PORT` | `5000` | API port |
| `CLIENT_ORIGIN` | `http://localhost:5173` | Allowed CORS origin |
| `JWT_SECRET` | long random string | Secret for signing login tokens |
| `JWT_EXPIRES_IN` | `12h` | Login session length |

---

## 8. Running the backend

```powershell
cd server
npm run dev          # http://localhost:5000/api  (auto-restarts on file changes)
```

On start the server prints the MySQL version and product count, or a clear message if it cannot connect.
Health check: http://localhost:5000/api/health

**Automated API tests** (with the server running, in a second terminal):

```powershell
npm run test:api     # from the project root
```

This runs **111 checks**: login, CRUD for every module, validation and constraint errors, the
sale transaction (stock must decrease), oversell protection, payments, all reports, all SQL demo
queries, the rollback demo and role permissions.

---

## 9. Running the frontend

```powershell
cd client
npm run dev          # http://localhost:5173
```

Vite forwards every `/api` request to `http://localhost:5000`, so both must be running
(`npm run dev` in the root starts both).

**Single-port production build:** `npm start` in the root builds the React app and serves it
from Express at http://localhost:5000.

---

## 10. Demo credentials

| Role | Email | Password | Can do |
|---|---|---|---|
| **Admin** | `admin@retail.com` | `admin123` | Everything, including Settings |
| Manager | `manager@retail.com` | `manager123` | Catalogue, stock, payments, reports |
| Cashier | `cashier@retail.com` | `cashier123` | POS sales, customers, viewing data |

The login page has one-click buttons that fill these in.

---

## 11. API documentation

Base URL: `http://localhost:5000/api`. All routes except `/health` and `/auth/login` need the header
`Authorization: Bearer <token>`. List endpoints accept `?page=&limit=&search=&sort=&order=ASC|DESC`
and return `{ data: [...], pagination: { page, limit, total, totalPages } }`.
Errors return `{ success: false, message, errors? }` with status 400 / 401 / 403 / 404 / 409.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Database connectivity check |
| POST | `/auth/login` | `{ email, password }` → `{ token, user }` |
| GET | `/auth/me` | Current user |
| GET | `/dashboard` | All dashboard statistics, trends and lists |
| GET | `/products` | List (filters: `category_id`, `supplier_id`, `status`) |
| GET | `/products/:id` | Product + sales stats + recent stock movements |
| POST | `/products` | Create product **and** its inventory row (transaction); optional `opening_stock` |
| PUT | `/products/:id` | Update product |
| DELETE | `/products/:id` | Delete (409 if the product has sales) |
| GET / POST | `/categories` | List (with product counts) / create |
| GET / PUT / DELETE | `/categories/:id` | Details with products / update / delete (409 if in use) |
| GET / POST | `/suppliers` | List / create |
| GET / PUT / DELETE | `/suppliers/:id` | Details with supplied products / update / delete |
| GET / POST | `/customers` | List (orders, total spent) / create |
| GET / PUT / DELETE | `/customers/:id` | Profile + purchase history / update / delete |
| GET | `/inventory` | Stock list + summary (filters: `status`, `category_id`) |
| PUT | `/inventory/:productId` | Adjust stock `{ type: add\|remove\|set, quantity, reason }` |
| GET | `/inventory/movements` | Stock audit log |
| GET | `/sales` | List (filters: `from`, `to`, `payment_method`, `payment_status`) + summary |
| GET | `/sales/:id` | Sale with customer, items, payments and store info |
| POST | `/sales` | **Transactional sale** `{ customer_id, items:[{product_id, quantity}], discount, payment_method, payment_status }` |
| GET | `/payments` | List (filters: `status`, `method`) + per-status summary |
| PUT | `/payments/:id` | Update a pending payment's status (sale status kept in sync) |
| GET | `/reports` | List of available reports |
| GET | `/reports/:type` | `daily-sales`, `weekly-sales`, `monthly-sales`, `product-sales`, `category-sales`, `customer-purchases`, `inventory`, `low-stock`, `suppliers` (`?from=YYYY-MM-DD&to=YYYY-MM-DD`) |
| GET | `/database/schema` | Live tables, columns, keys, checks, indexes, views from `INFORMATION_SCHEMA` |
| GET | `/database/queries` | Predefined demonstration queries (parsed from `database/queries.sql`) |
| POST | `/database/queries/:id/run` | Run one predefined query in a **READ ONLY** transaction |
| POST | `/database/transaction-demo` | Live atomicity/ROLLBACK demonstration |
| GET / PUT | `/settings` | Store settings (PUT: Admin only) |
| GET | `/search?q=` | Global search |
| GET | `/notifications` | Low stock and pending payment alerts |

> **Security note:** the SQL demo never runs SQL sent from the browser. It only runs the
> predefined `SELECT` queries in `database/queries.sql`, by id, inside a read-only transaction.

---

## 12. Important SQL queries

All 22 queries are in [`database/queries.sql`](database/queries.sql) and can be run from the
**Database → SQL Queries** tab. A few highlights:

```sql
-- Products with category names (INNER JOIN)
SELECT p.product_name, c.category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.id;

-- Products below reorder level
SELECT p.product_name, i.quantity, p.reorder_level
FROM products p JOIN inventory i ON i.product_id = p.id
WHERE i.quantity <= p.reorder_level;

-- Top-selling products (GROUP BY + ORDER BY)
SELECT p.product_name, SUM(si.quantity) AS total_sold
FROM sale_items si JOIN products p ON si.product_id = p.id
GROUP BY p.id, p.product_name
ORDER BY total_sold DESC LIMIT 10;

-- Monthly sales
SELECT DATE_FORMAT(sale_date, '%Y-%m') AS month, COUNT(*) AS orders, SUM(total_amount) AS revenue
FROM sales GROUP BY DATE_FORMAT(sale_date, '%Y-%m') ORDER BY month;

-- Products never sold (NOT EXISTS subquery)
SELECT p.product_name FROM products p
WHERE NOT EXISTS (SELECT 1 FROM sale_items si WHERE si.product_id = p.id);

-- Customers spending more than the average customer (HAVING + nested subquery)
SELECT c.customer_name, SUM(s.total_amount) AS total_spent
FROM customers c JOIN sales s ON s.customer_id = c.id
GROUP BY c.id, c.customer_name
HAVING SUM(s.total_amount) > (SELECT AVG(t) FROM (SELECT SUM(total_amount) t FROM sales
                                                  WHERE customer_id IS NOT NULL GROUP BY customer_id) x);
```

The full list covers: products + categories, products + suppliers, reorder level, total sales
(COUNT/SUM/AVG/MIN/MAX), today's sales, top sellers, monthly sales, sales by category, top customers,
inventory value with `ROLLUP`, never-sold products, best sellers with `HAVING`, sales with customer
details, quantity sold per product, suppliers by product count, above-average customers, payment
summary, last 7 days, querying the view, categories with counts, `RANK() OVER (PARTITION BY …)` and
the stock audit trail.

[`database/crud_examples.sql`](database/crud_examples.sql) shows INSERT / UPDATE / DELETE, statements
that violate constraints on purpose, and the complete sale transaction.

### The sale transaction (`POST /api/sales`, see `server/services/saleService.js`)

```sql
START TRANSACTION;
SELECT ... FROM inventory WHERE product_id IN (...) FOR UPDATE;   -- lock stock rows
INSERT INTO sales (...) VALUES (...);                             -- header
UPDATE sales SET invoice_no = 'INV-000052' WHERE id = ...;
INSERT INTO sale_items (sale_id, product_id, quantity, unit_price) VALUES (...), (...);
UPDATE inventory SET quantity = quantity - ? WHERE product_id = ? AND quantity >= ?;  -- reduce stock
INSERT INTO stock_movements (...) VALUES (...);                   -- audit
INSERT INTO payments (...) VALUES (...);
COMMIT;   -- or ROLLBACK if any step fails
```

Example: stock 50 → customer buys 3 → stock **47**, and the dashboard, inventory and
products pages all show 47 because they read from the same table.

---

## 13. DBMS concepts demonstrated

| ✓ | Concept | Where to see it |
|---|---|---|
| ✓ | Relational database | 11 related InnoDB tables (`database/schema.sql`) |
| ✓ | Primary keys | `id` in every table; shown as **PK** in the ER diagram / schema viewer |
| ✓ | Foreign keys | 11 FK constraints; shown as **FK** with the referenced table |
| ✓ | Normalisation (3NF) | Database → Overview → *Normalization* card |
| ✓ | CRUD operations | Products, Categories, Suppliers, Customers, Inventory pages |
| ✓ | SQL queries | Database → SQL Queries (22 runnable queries) |
| ✓ | Joins | INNER / LEFT JOIN across up to 4 tables (reports, dashboard, view) |
| ✓ | Aggregate functions | COUNT, SUM, AVG, MIN, MAX with GROUP BY / HAVING / ROLLUP |
| ✓ | Transactions (ACID) | POS sale, stock adjustment, product creation, payment update, rollback demo |
| ✓ | Referential integrity | Try deleting the *Groceries* category (RESTRICT) or a supplier (SET NULL) |
| ✓ | Relationships | 1 : 1 (product–inventory), 1 : N, M : N resolved by `sale_items` |
| ✓ | Constraints | NOT NULL, UNIQUE, CHECK (`quantity >= 0`, prices ≥ 0, `discount <= subtotal`) |
| ✓ | Views | `v_product_stock` |
| ✓ | Indexes | On names, dates, statuses and all FKs (Table Schemas tab) |
| ✓ | Subqueries | Correlated, `NOT EXISTS`, derived tables |
| ✓ | Generated columns | `sale_items.subtotal` |
| ✓ | Window functions | `RANK() OVER (PARTITION BY category …)` |
| ✓ | Concurrency control | `SELECT … FOR UPDATE` row locks during a sale |
| ✓ | Data dictionary | Schema viewer built from `INFORMATION_SCHEMA` |

---

## 14. Project structure

```
retail-inventory-system/
├── package.json              # root scripts: install:all, setup-db, dev, start, test:api
├── README.md
├── database/
│   ├── schema.sql            # DDL: tables, keys, constraints, indexes, view
│   ├── seed.sql              # sample data (generated, dates relative to today)
│   ├── queries.sql           # 22 demo queries (also used by the Database page)
│   └── crud_examples.sql     # INSERT/UPDATE/DELETE + transaction examples
├── server/
│   ├── server.js             # Express app
│   ├── .env.example
│   ├── config/               # env + MySQL connection pool + transaction helper
│   ├── routes/index.js       # all REST routes
│   ├── controllers/          # one file per module (SQL lives here)
│   ├── services/             # saleService (transaction), queryLibrary (parses queries.sql)
│   ├── middleware/           # JWT auth, roles, MySQL-error → friendly-message handler
│   ├── utils/                # validation, pagination, errors
│   └── scripts/              # setup-db.js, generate-seed.js, api-test.js
└── client/
    ├── index.html
    ├── vite.config.js        # /api proxy → :5000
    └── src/
        ├── App.jsx           # routes
        ├── layouts/          # sidebar + top bar
        ├── pages/            # Dashboard, Products, …, DatabasePage, Settings
        ├── components/       # UI kit, ER diagram, SQL code viewer, charts
        ├── context/          # auth, toasts, store settings
        ├── hooks/            # data fetching, list state
        ├── services/api.js   # fetch wrapper
        └── utils/            # formatting (₹, dates), CSV export
```

---

## 15. Presentation demo script

1. **Login** as Admin and show the **Dashboard** (all values are live SQL).
2. **Products**: add a product (show the duplicate-SKU and negative-price validation), edit it, view it.
3. **Categories**: try deleting *Groceries* to show the **FK RESTRICT** error.
4. **Inventory**: note the stock of *Premium Basmati Rice 5kg*.
5. **New Sale**: sell 2 of it, complete the sale, show the *"Inventory updated 48 → 46"* panel and print the invoice.
6. Back on **Inventory** / **Dashboard**: the stock and today's sales have changed.
7. Try to sell more than the available stock to show the oversell protection.
8. **Sales History**: filter by date, open a sale's details.
9. **Payments**: mark a pending payment as paid.
10. **Reports**: switch reports, change dates, export CSV.
11. **Database**: ER diagram → Table Schemas → run SQL queries → **Transactions → Run demo** (live ROLLBACK).

---

## 16. Troubleshooting

| Problem | Fix |
|---|---|
| `ER_ACCESS_DENIED_ERROR` | Wrong `DB_USER`/`DB_PASSWORD` in `server\.env` |
| `ECONNREFUSED 3306` | MySQL service is stopped. Start **MySQL80** in `services.msc` |
| "Cannot reach the MySQL database" in the app | Run `npm run setup-db`, then restart `npm run dev` |
| "Cannot reach the server" toast | The backend isn't running. Use `npm run dev` from the root |
| Port 5000 already in use | Change `PORT` in `server\.env` **and** the proxy target in `client\vite.config.js` |
| `npm` blocks esbuild's install script (npm 11+) | Already approved in `client/package.json` (`allowScripts`); or run `npm approve-scripts esbuild` |
| PowerShell says "running scripts is disabled" | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once, or use *Command Prompt* |

---

## 17. Future enhancements

- Purchase orders and goods-received notes (auto-restock from suppliers)
- Barcode scanning in the POS and barcode label printing
- Returns / refunds with reverse stock movements
- Batch and expiry-date tracking for groceries (FEFO)
- Multi-store / multi-warehouse inventory
- Per-product GST slabs (0%, 5%, 12%, 18%, 28%) and GST reports (GSTR-1)
- Customer loyalty points and SMS/WhatsApp e-invoices
- Stored procedures and triggers for low-stock alerts
- Scheduled database backups and an audit log for all edits
- Unit tests for the frontend and CI pipeline
