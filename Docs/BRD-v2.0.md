# Business Requirements Document (  )
## Bitlord's Computer Parts — Event-Driven Order & Inventory System

**Version:** 2.0
**Date:** 2026-05-22
**Author:** Malinga Bandara (BitLord)
**Status:** Active

---

## 1. Executive Summary

Bitlord's Computer Parts is a computer hardware retailer requiring a modern, scalable, full-stack system to manage product orders, track inventory in real time, notify stakeholders of critical events, and serve customers through a polished web storefront. This document defines the business requirements across four development phases, building incrementally from the Phase 1 foundation.

**Phase 1** (Archived — completed) established the core microservices backbone: Order Service, Inventory Service, Notification Service, Kafka messaging, Eureka discovery, API Gateway, Zipkin tracing, and Prometheus/Grafana observability.

**Phase 2** extends the system with authentication, role-based access control, real-time stock validation, and production-quality email notifications.

**Phase 3** delivers comprehensive developer and API documentation alongside structured sample data.

**Phase 4** delivers a modern, production-grade frontend web application serving both customers and administrators.

---

## 2. Business Objectives

| # | Objective | Phase |
|---|-----------|-------|
| BO-01 | Enable customers to place and track orders for computer hardware in real time. | 1 ✅ |
| BO-02 | Maintain accurate, up-to-date inventory levels across all product categories automatically. | 1 ✅ |
| BO-03 | Notify relevant parties of important order and stock events promptly via structured emails. | 1 ✅ / 2 |
| BO-04 | Reduce manual intervention in stock management by automating inventory deduction upon order placement. | 1 ✅ |
| BO-05 | Provide a foundation for future scalability as the business grows. | 1 ✅ |
| BO-06 | Protect all administrative operations behind secure, role-based authentication. | 2 |
| BO-07 | Allow a Super Admin to manage the full user hierarchy including creating other admins. | 2 |
| BO-08 | Validate real-time stock availability at the point of order placement and inform customers immediately when stock is insufficient. | 2 |
| BO-09 | Ensure all API behaviour and integration details are clearly documented for future maintainers and collaborators. | 3 |
| BO-10 | Provide customers and admins with a visually compelling, intuitive web experience. | 4 |

---

## 3. Scope

### 3.1 In Scope (All Phases)

**Phase 2 — Auth, RBAC & Enhanced Notifications**
- JWT-based authentication for customers and administrators.
- Role hierarchy: `SUPER_ADMIN > ADMIN > CUSTOMER`.
- Super Admin can register and manage Admin accounts.
- Customer self-registration and login.
- Admin-only protected endpoints for inventory management and order administration.
- Real-time stock check at the point of order placement with synchronous feedback.
- Professionally structured HTML email notifications for order events and low-stock alerts.

**Phase 3 — Documentation**
- `Instructions/` folder inside the project repository.
- API documentation in Markdown covering all endpoints with request/response examples.
- Project architecture and how-it-works guide.
- Sample data scripts for seeding inventory and placing test orders.
- Kafka event schema reference.

**Phase 4 — Frontend Web Application**
- CDN-based React web application (no local build toolchain required for basic delivery).
- Pages: Login, Sign-Up, Home, About, Shopping, and Admin Dashboard.
- Customer-facing: product browsing, cart, order placement, order history.
- Admin-facing: product and stock management, order overview, sales summary.
- Modern, attractive, responsive UI design.

### 3.2 Out of Scope (All Phases)
- Payment gateway integration.
- Multi-warehouse management.
- Supplier purchase order management.
- Native mobile applications.

---

## 4. Stakeholders

| Stakeholder | Role | Interest |
|-------------|------|----------|
| Customer | End user | Browse products, place orders, receive order status updates |
| Admin | Internal staff | Manage stock, view and update orders, receive low-stock alerts |
| Super Admin | System owner | Full administrative access including user management |
| Developer (BitLord) | Builder | Build, maintain, and document the system |

---

## 5. Business Rules

### 5.1 Inherited from Phase 1

| ID | Rule |
|----|------|
| BR-01 | An order can only be confirmed if sufficient stock is available for all items. |
| BR-02 | Inventory stock must be decremented immediately upon order confirmation. |
| BR-03 | If stock is insufficient, the order must be automatically rejected with a reason. |
| BR-04 | A low-stock alert must be triggered when a product's quantity falls below its defined minimum threshold. |
| BR-05 | All order status changes must generate a notification event. |
| BR-06 | An order must have a unique identifier traceable across all services. |
| BR-07 | Notification delivery failure must not affect the order or inventory process. |

### 5.2 Phase 2 — New Business Rules

| ID | Rule |
|----|------|
| BR-08 | All API endpoints must require a valid JWT access token except `/auth/login` and `/auth/register`. |
| BR-09 | Only a `SUPER_ADMIN` may create or deactivate an `ADMIN` account. |
| BR-10 | An `ADMIN` may manage inventory and view all orders but may not manage other admin accounts. |
| BR-11 | A `CUSTOMER` may only view and manage their own orders. |
| BR-12 | Stock availability must be validated synchronously before the order event is published to Kafka. If stock is zero or below the requested quantity, the order must be rejected at API level before any event is published. |
| BR-13 | When an order is rejected due to insufficient stock, the customer must receive an email explaining the situation with current stock details and a prompt to check back later. |
| BR-14 | Admin notification emails for low stock must include the product name, SKU, current quantity, and threshold. |
| BR-15 | All notification emails must use a structured HTML template reflecting the Bitlord's Computer Parts brand. |

---

## 6. Functional Requirements

### Phase 1 (Archived — for reference)
See BRD v1.0. All Phase 1 requirements (FR-01 through FR-15) are implemented.

---

### Phase 2 — Authentication, RBAC & Enhanced Notifications

**Authentication**
- **FR-P2-01:** The system shall provide a `/auth/register` endpoint for customer self-registration (name, email, password).
- **FR-P2-02:** The system shall provide a `/auth/login` endpoint that returns a signed JWT access token and a refresh token on successful credential validation.
- **FR-P2-03:** The system shall provide a `/auth/refresh` endpoint to issue a new access token using a valid refresh token.
- **FR-P2-04:** The system shall provide a `/auth/logout` endpoint that invalidates the refresh token.

**User & Role Management**
- **FR-P2-05:** The system shall maintain a `users` table with fields: id, name, email, hashed password, role (`CUSTOMER`, `ADMIN`, `SUPER_ADMIN`), active flag, and timestamps.
- **FR-P2-06:** A `SUPER_ADMIN` shall be able to create an `ADMIN` account via a protected endpoint.
- **FR-P2-07:** A `SUPER_ADMIN` shall be able to deactivate or reactivate any admin account.
- **FR-P2-08:** The system shall seed one default `SUPER_ADMIN` account on first startup if none exists.

**Order Service — Stock Check**
- **FR-P2-09:** When a customer places an order, the Order Service shall call the Inventory Service synchronously (via Feign Client) to validate stock before accepting the order.
- **FR-P2-10:** If stock is insufficient for any line item, the Order Service shall return HTTP 409 with a structured error body listing the affected SKUs and their available quantities.
- **FR-P2-11:** The system shall NOT publish an `order-placed` Kafka event for orders that fail the synchronous stock check.

**Email Notifications**
- **FR-P2-12:** All outbound emails shall use responsive HTML templates with the Bitlord's Computer Parts brand header, footer, and colour scheme.
- **FR-P2-13:** Order confirmation emails shall include: order ID, item list with quantities and prices, total amount, and estimated status.
- **FR-P2-14:** Order rejection emails (insufficient stock) shall include: the reason, the specific items that failed, current available stock, and a call-to-action to revisit the store.
- **FR-P2-15:** Low-stock alert emails sent to admins shall include: product name, SKU, category, current stock, and the defined threshold.
- **FR-P2-16:** Shipped and delivered status emails shall include the order summary and a thank-you message.

---

### Phase 3 — Documentation

- **FR-P3-01:** The repository shall contain an `Instructions/` folder at the project root.
- **FR-P3-02:** An `API-REFERENCE.md` file shall document every REST endpoint across all services, including: HTTP method, path, auth requirement, request body schema, success response, and error responses.
- **FR-P3-03:** A `HOW-IT-WORKS.md` file shall explain the full system architecture, service interactions, Kafka event flow, and startup sequence.
- **FR-P3-04:** A `SAMPLE-DATA.md` file shall provide ready-to-use curl commands and JSON payloads for: registering users, logging in, seeding inventory, placing orders, adjusting stock, and triggering low-stock alerts.
- **FR-P3-05:** A `KAFKA-EVENTS.md` file shall document all Kafka topic names, event schemas with field descriptions, and producer/consumer ownership per service.
- **FR-P3-06:** A `SETUP.md` file shall provide a step-by-step local environment setup guide covering: prerequisites, Docker Compose startup, service startup order, and environment variable configuration.

---

### Phase 4 — Frontend Web Application

**General**
- **FR-P4-01:** The frontend shall be a React-based single-page application (SPA) served via a static host or CDN-compatible build.
- **FR-P4-02:** The application shall be fully responsive and optimised for desktop and mobile viewports.
- **FR-P4-03:** The application shall communicate exclusively with the backend through the API Gateway at port 8080.
- **FR-P4-04:** JWT tokens shall be stored in memory (not localStorage) and refreshed automatically using the `/auth/refresh` endpoint.

**Pages**

- **FR-P4-05 — Login Page:** Email/password form with validation, error messaging, and link to Sign-Up.
- **FR-P4-06 — Sign-Up Page:** Registration form for customers (name, email, password, confirm password) with real-time validation.
- **FR-P4-07 — Home Page:** Hero section with featured products, promotional banners, category highlights, and navigation to the Shopping page.
- **FR-P4-08 — About Page:** Company story, mission statement, product category overview, and contact information.
- **FR-P4-09 — Shopping Page:** Product catalogue with category filter, search bar, stock badges, and add-to-cart functionality.
- **FR-P4-10 — Cart & Checkout:** Cart summary with quantity adjustment and order placement button that calls the Order API.
- **FR-P4-11 — Order History (Customer):** List of the logged-in customer's orders with status badges.
- **FR-P4-12 — Admin Dashboard:** Protected route available only to `ADMIN` and `SUPER_ADMIN` roles, containing:
  - Inventory management table (add product, update stock, set threshold).
  - Order management table (view all orders, update status).
  - Sales summary panel (total orders, revenue, low-stock alerts count).
  - User management panel (Super Admin only: create admins, deactivate accounts).

---

## 7. Non-Functional Requirements

| ID | Requirement | Target | Phase |
|----|-------------|--------|-------|
| NFR-01 | **Availability** | Services maintain 99%+ uptime under normal load. | All |
| NFR-02 | **Scalability** | Each microservice independently scalable. | All |
| NFR-03 | **Fault Tolerance** | Notification failures must not cascade. | All |
| NFR-04 | **Observability** | All inter-service calls and Kafka events traceable end-to-end. | All |
| NFR-05 | **Latency** | Order placement API response under 500ms. | All |
| NFR-06 | **Security** | All endpoints protected via JWT; passwords hashed with BCrypt. | 2+ |
| NFR-07 | **Auditability** | All order status transitions logged with timestamps. | All |
| NFR-08 | **Code Quality** | All service and Kafka event handler classes must include inline code comments explaining logic. | 2+ |
| NFR-09 | **Frontend Performance** | Initial page load under 3 seconds on a standard connection. | 4 |
| NFR-10 | **Accessibility** | Frontend meets WCAG 2.1 AA colour contrast minimum. | 4 |

---

## 8. Success Criteria

| Phase | Criterion | Measurement |
|-------|-----------|-------------|
| 2 | Authenticated order placement works end-to-end | JWT-authenticated POST to `/api/orders` succeeds and produces correct Kafka events |
| 2 | Stock check prevents over-ordering | HTTP 409 returned with correct detail when stock is insufficient |
| 2 | Email templates are professional and branded | HTML emails render correctly in Mailtrap for all event types |
| 2 | RBAC is enforced | Admin-only endpoints return 403 for customer tokens |
| 3 | Documentation is complete | All endpoints, events, and setup steps documented with working sample data |
| 4 | Frontend connects to live backend | Login, shopping, cart, and checkout flow work against the real API |
| 4 | Admin dashboard is functional | Stock and order management operations complete successfully via UI |

---

*Document End — BRD v2.0*
