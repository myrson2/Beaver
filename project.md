# Backend Engineering Portfolio Roadmap
## Order & Inventory Management System (E-Commerce / Cafe POS)

---

## 1. Project Vision & Architecture Strategy

This project is a high-reliability backend system for managing inventory, product options, and order execution. Designed for flexibility, it supports both traditional e-commerce platforms and fast-paced retail or cafe POS operations.

The development roadmap prioritizes **core reliability, concurrency control, and clean application architecture** over bloated database schemas.

---

## 2. Phase 1 Schema (Streamlined 8-Table Baseline)

Phase 1 focuses on a lean, normalized schema that supports authentication, role-based access control, catalog options, order execution, and inventory logs without redundant complexity.

```
       +--------------+           +--------------+
       |     User     |           |    Asset     |
       +--------------+           +--------------+
       | id (PK)      |           | id (PK)      |
       | email        |           | url          |
       | password     |           | entity_type  |
       | role (Enum)  |           | entity_id    |
       +-------+------+           +--------------+
               |
               | (1:N)
               v
       +--------------+           +--------------+
       |    Order     |<--(1:N)---|  OrderItem   |
       +--------------+           +--------------+
       | id (PK)      |           | id (PK)      |
       | user_id (FK) |           | order_id(FK) |
       | status       |           | product_id(FK|
       | total_amount |           | unit_price   |
       +--------------+           | quantity     |
                                  | options_json |
                                  +-------+------+
                                          |
                                          | (N:1)
+--------------+                  +-------v------+          +---------------+
|   Category   |<--------(1:N)----|   Product    |<--(1:N)--| ProductOption |
+--------------+                  +--------------+          +---------------+
| id (PK)      |                  | id (PK)      |          | id (PK)       |
| name         |                  | category_id  |          | product_id(FK)|
+--------------+                  | price        |          | option_name   |
                                  | stock_qty    |          | extra_price   |
                                  +-------+------+          +---------------+
                                          |
                                          | (1:N)
                                          v
                                  +--------------+
                                  |   StockLog   |
                                  +--------------+
                                  | id (PK)      |
                                  | product_id(FK|
                                  | change_qty   |
                                  | reason       |
                                  +--------------+
```

### Table Breakdown

1. **`User`**: Handles authentication and static RBAC roles (`customer`, `staff`, `admin`).
2. **`Asset`**: Poly-functional media table for storing image URLs linked to products or system entities.
3. **`Category`**: Simple catalog classification for menus or store departments.
4. **`Product`**: Core inventory catalog holding base prices and active stock counts.
5. **`ProductOption`**: Customizations and add-ons (e.g., *Oat Milk*, *Size: Large*, *Gift Wrap*).
6. **`Order`**: Parent transactional header capturing order status, payment methods, and final totals.
7. **`OrderItem`**: Historical line item preserving price snapshots and custom selections at the moment of checkout.
8. **`StockLog`**: Audit ledger tracking every inventory modification with reasons (`sale`, `restock`, `spoilage`, `adjustment`).

---

## 3. Implementation Phases

### Phase 1: Core Systems, Security & Static RBAC (MVP)
**Objective**: Build a secure, functional API capable of managing accounts, catalog items, and completing orders with basic authorization rules.

* **Authentication & Authorization**:
  * Secure password hashing (Bcrypt/Argon2).
  * Stateless JWT authentication (access & refresh token strategy).
  * **Static Enum RBAC Integration**:
    * `customer`: Can view catalog, create orders, view personal order history.
    * `staff`: Can view stock, manage order status transitions (`pending` $\rightarrow$ `preparing` $\rightarrow$ `completed`).
    * `admin`: Full access (manage users, adjust stock manually, modify catalog items, delete resources).

* **API Endpoints**:

| Route | Method | Access | Description |
| :--- | :--- | :--- | :--- |
| `/api/auth/register` | `POST` | Public | Register a new customer account |
| `/api/auth/login` | `POST` | Public | Authenticate and obtain JWT |
| `/api/products` | `GET` | Public | List products and available options |
| `/api/products` | `POST` | Admin | Create product entries and base options |
| `/api/orders` | `POST` | Customer | Submit a new order (triggers stock deduction) |
| `/api/orders/{id}/status` | `PATCH` | Staff/Admin | Update order fulfillment lifecycle |
| `/api/inventory/adjust` | `POST` | Admin | Manually restock or log damaged inventory |

* **Database Migrations**: Set up schema tracking using Alembic.

---

### Phase 2: High Reliability & Concurrency Control (Backend Showcase)
**Objective**: Transform the project into a senior-level portfolio piece by solving real-world distributed system challenges.

* **Pessimistic / Optimistic Locking**:
  * Prevent race conditions where two simultaneous checkouts sell the last remaining stock item.
  * Implement explicit SQL locking (`SELECT ... FOR UPDATE`) during order processing transactions.
* **Atomic Transactions & Ledger Integrity**:
  * Enforce DB-level transactions guaranteeing that `Order` creation, `Product.stock_quantity` deduction, and `StockLog` entry either **all succeed or all rollback**.
* **Automated Testing Suite**:
  * Build a full testing pipeline with `pytest`.
  * Unit tests for pricing logic and option calculations.
  * Integration tests simulating concurrent checkouts using async test runners.

---

### Phase 3: System Scaling & Advanced Features (Optional Polish)
**Objective**: Add asynchronous background task processing and caching to demonstrate enterprise readiness.

* **Background Task Processing**:
  * Integrate Celery or Redis Queue (RQ) for background workers.
  * Asynchronously process email receipt generation and low-stock notification triggers.
* **Read-Heavy Caching Layer**:
  * Implement Redis caching for product catalog lists (`GET /api/products`) to reduce database load.
  * Invalidate cache on administrative updates (`POST`, `PATCH`, `DELETE`).
* **Docker & Observability**:
  * Containerize the app, database, and background workers using `docker-compose`.
  * Add structured logging and health check endpoints.

---

## 4. Key Engineering Callouts for Portfolio/Resumes

When presenting this project to technical recruiters or hiring managers, highlight the following backend competencies:

1. **Transaction Guarantees**: *"Engineered atomic transaction handling across multi-table writes using PostgreSQL isolation levels to guarantee data consistency during order creation."*
2. **Race Condition Prevention**: *"Implemented database-level pessimistic locking (`FOR UPDATE`) on inventory rows to safely handle high-concurrency checkouts without overselling stock."*
3. **Role-Based Security**: *"Designed modular dependency injection middleware in FastAPI enforcing Role-Based Access Control (RBAC) across public, operational staff, and admin tiers."*
4. **Audit Traceability**: *"Constructed an append-only inventory log system that tracks historical stock movements alongside real-time counts for auditing capabilities."*