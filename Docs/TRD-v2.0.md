# Technical Requirements Document (TRD)
## Bitlord's Computer Parts — Event-Driven Order & Inventory System

**Version:** 2.0
**Date:** 2026-05-22
**Author:** Malinga Bandara (BitLord)
**Status:** Active

---

## 1. System Overview

This document defines the technical architecture, stack, service contracts, event schemas, security model, documentation structure, and frontend specification for the Bitlord's Computer Parts system across Phases 2, 3, and 4. It builds directly on the Phase 1 foundation documented in TRD v1.0.

**Phase 1** (Archived) — Core microservices, Kafka messaging, Eureka, API Gateway, Zipkin, Prometheus/Grafana.

**Phase 2** — Authentication service, JWT + RBAC, real-time stock validation via Feign Client, HTML email templates with Spring Mail.

**Phase 3** — `Instructions/` documentation folder: API reference, architecture guide, setup guide, sample data, Kafka event schema reference.

**Phase 4** — CDN-based React SPA: Login, Sign-Up, Home, About, Shopping, Admin Dashboard.

> **Code Comment Requirement (all Phase 2+ code):** Every class, method, Kafka producer/consumer, and non-trivial code block must include inline comments explaining what it does and why. This is mandatory for maintainability and portfolio readability.

---

## 2. Phase 1 — Inherited Stack (Reference)

Refer to TRD v1.0 for full detail. Summary of inherited components:

| Component | Technology | Port |
|-----------|-----------|------|
| Order Service | Spring Boot 3.2 / Java 17 | 8081 |
| Inventory Service | Spring Boot 3.2 / Java 17 | 8082 |
| Notification Service | Spring Boot 3.2 / Java 17 | 8083 |
| API Gateway | Spring Cloud Gateway | 8080 |
| Eureka Server | Spring Cloud Netflix | 8761 |
| Apache Kafka | Confluent 7.6.0 | 9092 |
| MySQL | 8.0 | 3306 |
| Zipkin | openzipkin/zipkin | 9411 |
| Prometheus | prom/prometheus | 9090 |
| Grafana | grafana/grafana | 3000 |

---

## 3. Phase 2 — Authentication, RBAC & Enhanced Notifications

### 3.1 New Service: `auth-service` (Port 8084)

**Responsibility:** Manages user registration, login, JWT issuance, token refresh, and user role management. Acts as the single source of truth for identity across the system.

**Database:** `bitlord_auth` (MySQL — new schema, separate from order and inventory databases)

#### 3.1.1 Entities

```java
// Represents a system user — customer, admin, or super admin
User {
    Long id                     // Primary key
    String name                 // Display name
    String email                // Unique login identifier
    String passwordHash         // BCrypt-hashed password — never store plaintext
    Role role                   // Enum: CUSTOMER, ADMIN, SUPER_ADMIN
    boolean active              // Soft-disable accounts without deletion
    LocalDateTime createdAt
    LocalDateTime updatedAt
}

// Stores refresh tokens for session management
RefreshToken {
    Long id
    String token                // UUID-based secure random token
    User user                   // FK to User
    LocalDateTime expiresAt
    boolean revoked             // True after logout or rotation
}
```

#### 3.1.2 REST Endpoints

| Method | Path | Auth Required | Role | Description |
|--------|------|--------------|------|-------------|
| POST | `/auth/register` | No | — | Customer self-registration |
| POST | `/auth/login` | No | — | Login; returns access + refresh tokens |
| POST | `/auth/refresh` | No | — | Exchange refresh token for new access token |
| POST | `/auth/logout` | Yes | Any | Revoke refresh token |
| POST | `/auth/admin/create` | Yes | SUPER_ADMIN | Create a new admin account |
| PATCH | `/auth/admin/{userId}/status` | Yes | SUPER_ADMIN | Activate / deactivate an admin |
| GET | `/auth/users` | Yes | SUPER_ADMIN | List all users |
| GET | `/auth/users/me` | Yes | Any | Get own profile |

#### 3.1.3 JWT Configuration

```yaml
# application.yml — auth-service
app:
  jwt:
    secret: ${JWT_SECRET}             # 256-bit secret from environment variable
    access-token-expiry: 900          # 15 minutes in seconds
    refresh-token-expiry: 604800      # 7 days in seconds
```

```java
// JWT utility — key responsibilities:
// 1. Generate signed access tokens embedding userId, email, and role as claims
// 2. Validate token signature and expiry
// 3. Extract claims from a valid token
public class JwtUtil {
    // Uses HMAC-SHA256 signing with the configured secret
    public String generateAccessToken(User user) { ... }

    // Returns true only if signature is valid and token is not expired
    public boolean validateToken(String token) { ... }

    // Extracts the role claim used by the gateway and services for RBAC
    public Role extractRole(String token) { ... }
}
```

#### 3.1.4 API Gateway — JWT Validation Filter

The API Gateway validates the JWT on every inbound request before routing. Services trust the gateway and do not re-validate tokens.

```java
// JwtAuthFilter.java — Spring Cloud Gateway GlobalFilter
// This filter runs before routing to any downstream service.
// It extracts the Authorization header, validates the token,
// and either forwards the request with enriched headers (X-User-Id, X-User-Role)
// or rejects it with 401 Unauthorized.
@Component
public class JwtAuthFilter implements GlobalFilter {

    // Paths that bypass JWT validation
    private static final List<String> PUBLIC_PATHS = List.of(
        "/auth/register", "/auth/login", "/auth/refresh"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1. Skip public paths
        // 2. Extract Bearer token from Authorization header
        // 3. Validate with JwtUtil
        // 4. Inject X-User-Id and X-User-Role headers for downstream services
        // 5. Return 401 if token is missing or invalid
    }
}
```

#### 3.1.5 Role-Based Access Control Matrix

| Endpoint Group | CUSTOMER | ADMIN | SUPER_ADMIN |
|----------------|----------|-------|-------------|
| `POST /api/orders` | ✅ | ✅ | ✅ |
| `GET /api/orders` (own orders) | ✅ | — | — |
| `GET /api/orders` (all orders) | ❌ | ✅ | ✅ |
| `PATCH /api/orders/{id}/status` | ❌ | ✅ | ✅ |
| `GET /api/inventory` | ✅ | ✅ | ✅ |
| `POST /api/inventory` | ❌ | ✅ | ✅ |
| `PATCH /api/inventory/{sku}/adjust` | ❌ | ✅ | ✅ |
| `POST /auth/admin/create` | ❌ | ❌ | ✅ |
| `PATCH /auth/admin/{id}/status` | ❌ | ❌ | ✅ |

---

### 3.2 Real-Time Stock Validation (Order Service Enhancement)

The Phase 1 stock check was asynchronous (Kafka-based). Phase 2 adds a **synchronous pre-check** using OpenFeign before any Kafka event is published.

```java
// StockValidationClient.java — Feign Client in Order Service
// Calls Inventory Service synchronously to check stock BEFORE publishing the order event.
// This prevents invalid events from entering Kafka and gives the customer
// an immediate HTTP response rather than waiting for async processing.
@FeignClient(name = "INVENTORY-SERVICE")
public interface StockValidationClient {

    // Returns a map of SKU -> availableQuantity for the requested items
    @PostMapping("/api/inventory/validate")
    StockValidationResponse validateStock(@RequestBody StockValidationRequest request);
}
```

```java
// OrderService.java — placeOrder() method flow:
// 1. Receive order request from controller
// 2. Call StockValidationClient.validateStock() synchronously
// 3. If any SKU has insufficient stock:
//    a. Build a 409 Conflict response listing each failed SKU and its available quantity
//    b. Publish NO Kafka events — the order is rejected at this layer
// 4. If all SKUs pass:
//    a. Persist order with PENDING status
//    b. Publish 'order-placed' event to Kafka
//    c. Return 201 Created with order details
```

**New Inventory Endpoint (for internal Feign calls):**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/inventory/validate` | Internal (gateway-bypass or service token) | Batch stock availability check |

**Request/Response:**
```json
// POST /api/inventory/validate — Request
{
  "items": [
    { "sku": "GPU-RTX4070", "requestedQuantity": 2 },
    { "sku": "RAM-DDR5-32", "requestedQuantity": 1 }
  ]
}

// Response — 200 OK
{
  "allAvailable": false,
  "results": [
    { "sku": "GPU-RTX4070", "requestedQuantity": 2, "availableQuantity": 1, "sufficient": false },
    { "sku": "RAM-DDR5-32", "requestedQuantity": 1, "availableQuantity": 15, "sufficient": true }
  ]
}
```

---

### 3.3 HTML Email Templates (Notification Service Enhancement)

All emails are rendered from **Thymeleaf HTML templates** and sent via Spring Mail.

#### 3.3.1 Dependencies (pom.xml addition)
```xml
<!-- Thymeleaf for HTML email templates -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

#### 3.3.2 Template Files Structure
```
notification-service/
└── src/main/resources/
    └── templates/email/
        ├── order-confirmed.html      // Sent when inventory check passes
        ├── order-rejected.html       // Sent when stock is insufficient
        ├── order-shipped.html        // Sent when admin marks order as SHIPPED
        ├── order-delivered.html      // Sent when admin marks order as DELIVERED
        └── low-stock-alert.html      // Sent to admin/warehouse on low stock
```

#### 3.3.3 Template Design Spec

All templates share a common base structure:

```
┌────────────────────────────────────────────┐
│  [BITLORD'S COMPUTER PARTS]  Logo + Brand  │  Dark header (#0f0f1a background)
├────────────────────────────────────────────┤
│                                            │
│  [Event Title / Icon]                      │  e.g. "✅ Order Confirmed"
│                                            │
│  Dear [Customer Name],                     │
│  [Contextual message body]                 │
│                                            │
│  ┌──────────── Order Summary ─────────┐    │
│  │ Order ID  │ ORD-xxxx-xxxx          │    │
│  │ Items     │ RTX 4070 x1  $599.99   │    │
│  │ Total     │ $599.99                │    │
│  └──────────────────────────────────┘    │
│                                            │
│  [Call-to-Action Button]                   │
│                                            │
├────────────────────────────────────────────┤
│  © 2026 Bitlord's Computer Parts           │  Footer
└────────────────────────────────────────────┘
```

**Colour palette for emails:**
- Background: `#0f0f1a` (dark navy)
- Card background: `#1a1a2e` (slightly lighter)
- Accent / CTA buttons: `#7c3aed` (purple)
- Success accent: `#10b981` (green)
- Warning / rejection accent: `#ef4444` (red)
- Text: `#e2e8f0` (off-white)

#### 3.3.4 Email Content Matrix

| Template | Subject | Key Content |
|----------|---------|-------------|
| `order-confirmed.html` | ✅ Your order has been confirmed — #ORD-xxxx | Order ID, item list, total, estimated processing note |
| `order-rejected.html` | ⚠️ We couldn't complete your order — #ORD-xxxx | Reason (insufficient stock), per-item available qty, shop link |
| `order-shipped.html` | 🚚 Your order is on its way! | Order ID, items, shipping confirmation note |
| `order-delivered.html` | 📦 Order delivered — Thank you! | Order summary, thank-you message, feedback CTA |
| `low-stock-alert.html` | 🔴 Low Stock Alert — [SKU] | Product name, SKU, category, current qty, threshold, manage stock link |

---

### 3.4 Updated Docker Compose (Phase 2 additions)

```yaml
# Addition to existing docker-compose.yml

  # Second MySQL database for the auth service
  mysql-auth:
    image: mysql:8.0
    ports:
      - "3307:3306"
    environment:
      MYSQL_ROOT_PASSWORD: bitlord123
      MYSQL_DATABASE: bitlord_auth
    volumes:
      - mysql_auth_data:/var/lib/mysql

volumes:
  mysql_data:
  mysql_auth_data:        # New volume for auth database
```

### 3.5 Updated API Gateway Routes (Phase 2)

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Existing routes from Phase 1...

        # Auth service — public paths bypass JWT filter
        - id: auth-service-public
          uri: lb://AUTH-SERVICE
          predicates:
            - Path=/auth/register, /auth/login, /auth/refresh
          filters:
            - RemoveRequestHeader=Authorization   # Clean pass-through for public auth endpoints

        # Auth service — protected paths require JWT
        - id: auth-service-protected
          uri: lb://AUTH-SERVICE
          predicates:
            - Path=/auth/**
```

---

## 4. Phase 3 — Documentation

### 4.1 Instructions Folder Structure

```
bitlord-computer-parts/
└── Instructions/
    ├── SETUP.md
    ├── HOW-IT-WORKS.md
    ├── API-REFERENCE.md
    ├── KAFKA-EVENTS.md
    └── SAMPLE-DATA.md
```

### 4.2 `SETUP.md` — Contents Specification

- **Prerequisites:** Java 17, Maven 3.9, Docker Desktop, Git.
- **Step 1:** Clone repository.
- **Step 2:** Configure environment variables (`.env` file template provided).
- **Step 3:** Start infrastructure with `docker-compose up -d`.
- **Step 4:** Startup order for Spring Boot services (Eureka → Gateway → Auth → Order → Inventory → Notification).
- **Step 5:** Verify health endpoints.
- **Step 6:** Import Postman collection (link or file reference).

### 4.3 `HOW-IT-WORKS.md` — Contents Specification

- System architecture diagram (ASCII or embedded image).
- Service responsibility summary table.
- Order placement flow (step-by-step with Feign + Kafka sequence).
- Low-stock alert flow.
- JWT authentication flow (register → login → attach token → call API).
- Kafka topic ownership table.

### 4.4 `API-REFERENCE.md` — Contents Specification

For every endpoint, document:

```markdown
## POST /auth/register

**Auth Required:** No
**Description:** Register a new customer account.

### Request Body
| Field    | Type   | Required | Description          |
|----------|--------|----------|----------------------|
| name     | String | Yes      | Customer display name |
| email    | String | Yes      | Unique email address  |
| password | String | Yes      | Min 8 characters      |

### Success Response — 201 Created
{ "message": "Registration successful", "userId": 12 }

### Error Responses
| Status | Code               | Description                   |
|--------|--------------------|-------------------------------|
| 400    | VALIDATION_ERROR   | Missing or invalid fields     |
| 409    | EMAIL_EXISTS       | Email already registered      |
```

### 4.5 `KAFKA-EVENTS.md` — Contents Specification

- Table of all topics with producer and consumer.
- Full JSON schema for each event with field-level description.
- Notes on event ordering guarantees and partition strategy.

### 4.6 `SAMPLE-DATA.md` — Contents Specification

Ready-to-copy `curl` commands for:
1. Register a customer.
2. Login and capture the JWT.
3. Seed 10 products across all categories.
4. Place a valid order.
5. Place an order that triggers a stock rejection.
6. Manually adjust stock as admin.
7. Trigger a low-stock alert by setting stock to 3.

---

## 5. Phase 4 — Frontend Web Application

### 5.1 Technology Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| Framework | React 18 | Via CDN (esm.sh or unpkg) or Vite build |
| Styling | Tailwind CSS (CDN) + custom CSS vars | Dark space-tech aesthetic |
| HTTP Client | Axios | For API calls to gateway :8080 |
| Routing | React Router v6 | Client-side SPA routing |
| State | React Context + useReducer | Auth state, cart state |
| Icons | Lucide React (CDN) | Consistent icon set |
| Notifications | React-Toastify | Order success/error feedback |
| Charts (Admin) | Recharts | Sales summary dashboard charts |

### 5.2 Design System

**Colour Palette:**
```css
--bg-primary:    #0a0a14;   /* Deep space black */
--bg-secondary:  #0f0f1e;   /* Card background */
--bg-elevated:   #1a1a2e;   /* Elevated panels */
--accent:        #7c3aed;   /* Purple — primary CTA */
--accent-hover:  #6d28d9;
--accent-glow:   rgba(124, 58, 237, 0.3);
--success:       #10b981;   /* Green — in stock, confirmed */
--warning:       #f59e0b;   /* Amber — low stock */
--danger:        #ef4444;   /* Red — out of stock, failed */
--text-primary:  #f1f5f9;
--text-secondary:#94a3b8;
--border:        rgba(255,255,255,0.08);
```

**Typography:**
- Headings: `Space Grotesk` (Google Fonts CDN)
- Body: `Inter` (Google Fonts CDN)
- Monospace (order IDs, SKUs): `JetBrains Mono`

**Component Style Rules:**
- Cards: `border-radius: 12px`, subtle `1px` border with `--border`, glass-morphism `backdrop-filter: blur(12px)` on overlays.
- Buttons: Rounded pill primary (`border-radius: 9999px`), with purple gradient and glow on hover.
- Tables: Zebra-striped with `--bg-elevated` alternating rows, sticky header.
- Badges: Pill-shaped, colour-coded by status (PENDING=amber, CONFIRMED=green, FAILED=red, SHIPPED=blue).

### 5.3 Application Structure

```
frontend/
├── index.html
├── src/
│   ├── main.jsx
│   ├── App.jsx                   // Router setup, protected route wrapper
│   ├── context/
│   │   ├── AuthContext.jsx       // JWT state, login/logout actions
│   │   └── CartContext.jsx       // Cart items, add/remove/quantity
│   ├── api/
│   │   ├── axiosInstance.js      // Base URL, auth header interceptor, token refresh
│   │   ├── authApi.js
│   │   ├── orderApi.js
│   │   └── inventoryApi.js
│   ├── pages/
│   │   ├── LoginPage.jsx
│   │   ├── SignUpPage.jsx
│   │   ├── HomePage.jsx
│   │   ├── AboutPage.jsx
│   │   ├── ShoppingPage.jsx
│   │   ├── CartPage.jsx
│   │   ├── OrderHistoryPage.jsx
│   │   └── admin/
│   │       ├── AdminDashboard.jsx
│   │       ├── InventoryManager.jsx
│   │       ├── OrderManager.jsx
│   │       ├── SalesSummary.jsx
│   │       └── UserManager.jsx   // Super Admin only
│   └── components/
│       ├── Navbar.jsx
│       ├── Footer.jsx
│       ├── ProductCard.jsx
│       ├── CartDrawer.jsx
│       ├── StatusBadge.jsx
│       ├── ProtectedRoute.jsx
│       └── AdminRoute.jsx
```

### 5.4 Page Specifications

#### Login Page
- Centered card on deep-space background with animated gradient blob.
- Fields: Email, Password (show/hide toggle).
- Error banner for invalid credentials.
- Link to Sign-Up.
- On success: redirect to Home (customer) or Admin Dashboard (admin/super admin).

#### Sign-Up Page
- Same aesthetic as Login.
- Fields: Full Name, Email, Password, Confirm Password.
- Inline validation (password strength indicator).
- On success: auto-login and redirect to Home.

#### Home Page
- Full-width hero with headline, subtext, and "Shop Now" CTA button.
- Animated particles or star-field background (CSS/canvas).
- "Featured Products" section — 4 cards pulled from the inventory API (highest stock).
- "Category Grid" — 10 category tiles with icons.
- "Why BitLord?" section — 3 trust-pillar cards (fast shipping, genuine parts, expert support).

#### About Page
- Brand story section with founder narrative.
- Stats bar: years in business, products stocked, orders fulfilled, satisfied customers.
- Team / mission section.
- Contact info with styled card.

#### Shopping Page
- Sidebar filter: Category (checkbox list), Price range (slider), In Stock only (toggle).
- Search bar at top.
- Product grid (3 columns desktop, 1 column mobile).
- Each `ProductCard` shows: image placeholder/icon, name, SKU, category, price, stock badge, "Add to Cart" button (disabled if out of stock).
- Pagination or infinite scroll.

#### Admin Dashboard
- Sidebar navigation with sections: Overview, Inventory, Orders, Users (Super Admin).
- **Overview:** KPI cards (total orders today, revenue, low-stock count, active products).
- **Inventory Manager:** Searchable table with inline edit for stock quantity and threshold. "Add Product" modal with full product form.
- **Order Manager:** Filterable table by status. Status dropdown to advance order lifecycle.
- **Sales Summary:** Line chart of daily orders (last 30 days), bar chart of revenue by category.
- **User Manager (Super Admin only):** Table of all users. "Create Admin" button. Toggle active/inactive per user.

### 5.5 API Integration Details

```javascript
// axiosInstance.js
// Attaches the JWT from AuthContext to every request.
// On 401 response, attempts silent token refresh via /auth/refresh.
// If refresh fails, clears auth state and redirects to /login.

const api = axios.create({ baseURL: 'http://localhost:8080' });

// Request interceptor — attach access token
api.interceptors.request.use(config => {
    const token = getAccessToken(); // From in-memory store, not localStorage
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
});

// Response interceptor — handle 401 with token refresh
api.interceptors.response.use(
    response => response,
    async error => {
        if (error.response?.status === 401 && !error.config._retry) {
            error.config._retry = true;
            const newToken = await refreshAccessToken();
            error.config.headers.Authorization = `Bearer ${newToken}`;
            return api(error.config);
        }
        return Promise.reject(error);
    }
);
```

### 5.6 Environment Configuration

```javascript
// config.js — frontend environment config
const config = {
    API_BASE_URL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:8080',
};
```

---

## 6. Updated Project Structure (All Phases)

```
bitlord-computer-parts/
├── docker-compose.yml
├── prometheus.yml
├── README.md
│
├── Instructions/                         // Phase 3
│   ├── SETUP.md
│   ├── HOW-IT-WORKS.md
│   ├── API-REFERENCE.md
│   ├── KAFKA-EVENTS.md
│   └── SAMPLE-DATA.md
│
├── eureka-server/
├── api-gateway/                          // Phase 2: JWT filter added
├── auth-service/                         // Phase 2: NEW
│   └── src/main/java/com/bitlord/authservice/
│       ├── controller/AuthController.java
│       ├── service/AuthService.java
│       ├── service/JwtService.java
│       ├── model/User.java
│       ├── model/RefreshToken.java
│       ├── model/Role.java (enum)
│       ├── repository/UserRepository.java
│       ├── repository/RefreshTokenRepository.java
│       ├── dto/RegisterRequest.java
│       ├── dto/LoginRequest.java
│       ├── dto/AuthResponse.java
│       ├── config/SecurityConfig.java
│       └── config/DataInitializer.java   // Seeds SUPER_ADMIN on startup
│
├── order-service/                        // Phase 2: Feign client added
│   └── src/main/java/com/bitlord/orderservice/
│       ├── feign/StockValidationClient.java   // NEW
│       └── ... (existing structure)
│
├── inventory-service/                    // Phase 2: /validate endpoint added
│   └── src/main/java/com/bitlord/inventoryservice/
│       ├── controller/StockValidationController.java  // NEW
│       └── ... (existing structure)
│
├── notification-service/                 // Phase 2: HTML templates
│   └── src/main/resources/templates/email/
│       ├── order-confirmed.html
│       ├── order-rejected.html
│       ├── order-shipped.html
│       ├── order-delivered.html
│       └── low-stock-alert.html
│
└── frontend/                            // Phase 4: NEW
    ├── index.html
    ├── vite.config.js
    ├── package.json
    └── src/
        └── (structure per Section 5.3)
```

---

## 7. Security Summary (All Phases)

| Concern | Implementation |
|---------|---------------|
| Password storage | BCrypt (strength 12) via Spring Security |
| Access tokens | JWT signed with HMAC-SHA256, 15-min expiry |
| Refresh tokens | UUID stored hashed in DB, 7-day expiry, single-use rotation |
| Token transport | Authorization: Bearer header (never cookies, never URL params) |
| Frontend token storage | In-memory JS variable (not localStorage) |
| Admin endpoint protection | Gateway-level RBAC check on `X-User-Role` header |
| Secrets management | All credentials via environment variables; never hardcoded |
| CORS | Gateway allows `http://localhost:5173` in dev; configured via env var in prod |

---

## 8. Phase Completion Checklist

### Phase 2
- [ ] `auth-service` running with register, login, refresh, logout
- [ ] Default SUPER_ADMIN seeded on startup
- [ ] JWT filter active in API Gateway
- [ ] RBAC enforced per role matrix (Section 3.1.2)
- [ ] Feign-based sync stock check working in Order Service
- [ ] HTTP 409 returned with structured error on stock failure
- [ ] All 5 HTML email templates rendering correctly in Mailtrap
- [ ] All new code includes inline comments

### Phase 3
- [ ] `Instructions/` folder committed to repository
- [ ] All 5 documentation files complete
- [ ] Sample curl commands tested and working
- [ ] Kafka event schemas up to date

### Phase 4
- [ ] All 7 pages implemented and styled
- [ ] Login → Home flow working with real JWT
- [ ] Shopping page loads products from Inventory API
- [ ] Cart → Checkout → Order placed successfully via API
- [ ] Admin dashboard CRUD operations working
- [ ] Super Admin user management functional
- [ ] Responsive on mobile viewport

---

*Document End — TRD v2.0*
