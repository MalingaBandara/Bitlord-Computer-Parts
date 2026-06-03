# Technical Requirements Document (TRD)
## Bitlord's Computer Parts — Event-Driven Order & Inventory System

**Version:** 1.0  
**Date:** 2026-05-15  
**Author:** Malinga Bandara (BitLord)  
**Status:** Draft

---

## 1. System Overview

This document defines the technical architecture, stack, service contracts, event schemas, and infrastructure configuration for the Bitlord's Computer Parts Event-Driven Order & Inventory System. The system is composed of three core microservices communicating asynchronously via Apache Kafka, with service discovery, API gateway, distributed tracing, and observability infrastructure.

---

## 2. Technology Stack

### 2.1 Core Services
| Component | Technology | Version |
|-----------|-----------|---------|
| Language | Java | 17 (LTS) |
| Framework | Spring Boot | 3.2.x |
| Build Tool | Maven | 3.9.x |
| ORM | Spring Data JPA + Hibernate | 6.x |
| Database | MySQL | 8.0 |
| Messaging | Apache Kafka | 3.6.x |
| Service Discovery | Netflix Eureka (Spring Cloud) | 2023.x |
| API Gateway | Spring Cloud Gateway | 2023.x |
| Distributed Tracing | Micrometer + Zipkin (B3 format) | 3.x |
| Email | Spring Mail (SMTP / Mailtrap for dev) | — |
| Security | Spring Security + JWT (jjwt) | 0.12.x |
| API Docs | Springdoc OpenAPI (Swagger UI) | 2.x |

### 2.2 Infrastructure (Docker Compose)
| Component | Image | Port |
|-----------|-------|------|
| Apache Kafka | `confluentinc/cp-kafka:7.6.0` | 9092 |
| Zookeeper | `confluentinc/cp-zookeeper:7.6.0` | 2181 |
| Zipkin | `openzipkin/zipkin` | 9411 |
| Prometheus | `prom/prometheus` | 9090 |
| Grafana | `grafana/grafana` | 3000 |
| MySQL | `mysql:8.0` | 3306 |
| Eureka Server | Custom Spring Boot image | 8761 |
| API Gateway | Custom Spring Boot image | 8080 |

### 2.3 CDN Dependencies (No local installs)
All frontend tooling and documentation assets loaded via CDN where applicable:
- Swagger UI assets via `springdoc-openapi` (auto-served)
- Actuator + Prometheus metrics via Spring Boot Actuator

---

## 3. Microservices Architecture

```
                        ┌─────────────────────────────────┐
                        │         API Gateway :8080        │
                        │    (Spring Cloud Gateway)        │
                        └────────────────┬────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
   ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
   │  Order Service   │      │Inventory Service │      │Notification Svc  │
   │     :8081        │      │     :8082        │      │     :8083        │
   └────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
            │                         │                          │
            └──────────────┬──────────┘                          │
                           │                                      │
                    ┌──────▼──────┐                               │
                    │    KAFKA    │ ◄─────────────────────────────┘
                    │  Broker     │
                    └─────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         order-placed  order-confirmed  low-stock-alert
                        order-failed

              ┌──────────────────────────────────────┐
              │       Eureka Discovery Server :8761   │
              └──────────────────────────────────────┘
              ┌──────────────────────────────────────┐
              │       Zipkin Tracing :9411            │
              └──────────────────────────────────────┘
              ┌──────────────────────────────────────┐
              │  Prometheus :9090 + Grafana :3000     │
              └──────────────────────────────────────┘
```

---

## 4. Service Definitions

### 4.1 Order Service (`order-service` — Port 8081)

**Responsibility:** Accepts order requests, manages order lifecycle, publishes order events to Kafka.

**Database:** `bitlord_orders` (MySQL)

**Key Entities:**
```java
Order {
    Long id (PK)
    String orderId (UUID)
    String customerId
    String customerEmail
    OrderStatus status  // PENDING, CONFIRMED, SHIPPED, DELIVERED, FAILED
    List<OrderItem> items
    BigDecimal totalAmount
    LocalDateTime createdAt
    LocalDateTime updatedAt
}

OrderItem {
    Long id (PK)
    String sku
    String productName
    Integer quantity
    BigDecimal unitPrice
}
```

**REST Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/orders` | Place a new order |
| GET | `/api/orders/{orderId}` | Get order by ID |
| GET | `/api/orders` | List all orders (with optional `?status=` filter) |
| PATCH | `/api/orders/{orderId}/status` | Update order status (admin) |

**Kafka Topics Published:**
- `order-placed` — on new order creation
- `order-status-updated` — on any status change

**Kafka Topics Consumed:**
- `inventory-reservation-result` — to confirm or fail the order

---

### 4.2 Inventory Service (`inventory-service` — Port 8082)

**Responsibility:** Manages product stock levels, processes inventory reservation requests from Kafka, publishes reservation results and low-stock alerts.

**Database:** `bitlord_inventory` (MySQL)

**Key Entities:**
```java
Product {
    Long id (PK)
    String sku (unique)
    String productName
    String category
    Integer stockQuantity
    Integer lowStockThreshold  // default: 5
    BigDecimal unitPrice
    LocalDateTime updatedAt
}

StockMovement {
    Long id (PK)
    String sku
    Integer quantityChange
    String reason  // ORDER_CONFIRMED, ORDER_FAILED, MANUAL_ADJUSTMENT
    String referenceOrderId
    LocalDateTime timestamp
}
```

**REST Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/inventory` | List all products with stock |
| GET | `/api/inventory/{sku}` | Get stock for specific SKU |
| POST | `/api/inventory` | Add new product (admin) |
| PATCH | `/api/inventory/{sku}/adjust` | Manually adjust stock (admin) |

**Kafka Topics Consumed:**
- `order-placed` — triggers stock reservation check

**Kafka Topics Published:**
- `inventory-reservation-result` — result of stock check (SUCCESS/FAILURE)
- `low-stock-alert` — when stock falls below threshold

---

### 4.3 Notification Service (`notification-service` — Port 8083)

**Responsibility:** Listens to all business events from Kafka and delivers notifications via email and system logs.

**No Database required** — stateless service.

**Kafka Topics Consumed:**
- `order-placed`
- `inventory-reservation-result`
- `order-status-updated`
- `low-stock-alert`

**Notification Matrix:**

| Event | Recipient | Channel |
|-------|-----------|---------|
| Order Placed | Customer | Email + Log |
| Order Confirmed | Customer | Email + Log |
| Order Failed (stock) | Customer | Email + Log |
| Low Stock Alert | Admin/Warehouse | Email + Log |
| Order Shipped | Customer | Email + Log |

**Email Config (application.yml):**
```yaml
spring:
  mail:
    host: smtp.mailtrap.io   # Dev: Mailtrap | Prod: SMTP provider
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail.smtp.auth: true
      mail.smtp.starttls.enable: true
```

---

## 5. Kafka Event Schemas

### 5.1 `order-placed`
```json
{
  "eventType": "ORDER_PLACED",
  "orderId": "ORD-uuid-here",
  "customerId": "CUST-001",
  "customerEmail": "customer@example.com",
  "items": [
    {
      "sku": "GPU-RTX4070",
      "productName": "NVIDIA RTX 4070",
      "quantity": 1,
      "unitPrice": 599.99
    }
  ],
  "totalAmount": 599.99,
  "timestamp": "2026-05-15T10:00:00Z"
}
```

### 5.2 `inventory-reservation-result`
```json
{
  "eventType": "INVENTORY_RESERVATION_RESULT",
  "orderId": "ORD-uuid-here",
  "status": "SUCCESS",  // or "FAILURE"
  "reason": null,        // or "Insufficient stock for SKU: GPU-RTX4070"
  "timestamp": "2026-05-15T10:00:01Z"
}
```

### 5.3 `low-stock-alert`
```json
{
  "eventType": "LOW_STOCK_ALERT",
  "sku": "GPU-RTX4070",
  "productName": "NVIDIA RTX 4070",
  "currentStock": 3,
  "threshold": 5,
  "timestamp": "2026-05-15T10:00:01Z"
}
```

### 5.4 `order-status-updated`
```json
{
  "eventType": "ORDER_STATUS_UPDATED",
  "orderId": "ORD-uuid-here",
  "customerEmail": "customer@example.com",
  "previousStatus": "PENDING",
  "newStatus": "CONFIRMED",
  "timestamp": "2026-05-15T10:00:02Z"
}
```

---

## 6. API Gateway Configuration

**Port:** 8080  
All external traffic routes through the gateway.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**

        - id: inventory-service
          uri: lb://INVENTORY-SERVICE
          predicates:
            - Path=/api/inventory/**

        - id: notification-service
          uri: lb://NOTIFICATION-SERVICE
          predicates:
            - Path=/api/notifications/**
```

---

## 7. Service Discovery — Eureka

**Port:** 8761  
Each service registers with Eureka on startup.

```yaml
# Each microservice application.yml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

---

## 8. Distributed Tracing — Zipkin

**Port:** 9411  
Uses Micrometer Tracing with Brave (B3 propagation) — compatible with Spring Boot 3.x.

```yaml
# Each microservice application.yml
management:
  tracing:
    sampling:
      probability: 1.0
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans
```

**Dependencies (pom.xml):**
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

> ⚠️ Note: Do NOT use `spring.zipkin.baseUrl` — that is the Spring Boot 2.x format. Spring Boot 3.x uses `management.zipkin.tracing.endpoint`.

---

## 9. Observability — Prometheus & Grafana

**Prometheus Port:** 9090 | **Grafana Port:** 3000

Each service exposes metrics via Spring Boot Actuator:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

**Prometheus scrape config (`prometheus.yml`):**
```yaml
scrape_configs:
  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8081']

  - job_name: 'inventory-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8082']

  - job_name: 'notification-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['host.docker.internal:8083']
```

**Grafana:** Import dashboard ID `4701` (JVM Micrometer) for instant JVM metrics visualization.

---

## 10. Docker Compose — Full Infrastructure

```yaml
version: '3.8'

services:

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9411:9411"

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  mysql:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: bitlord123
      MYSQL_DATABASE: bitlord_orders
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

---

## 11. Project Structure

```
bitlord-computer-parts/
├── docker-compose.yml
├── prometheus.yml
├── README.md
│
├── eureka-server/
│   └── src/main/...
│
├── api-gateway/
│   └── src/main/...
│
├── order-service/
│   ├── src/main/java/com/bitlord/orderservice/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── model/
│   │   ├── dto/
│   │   ├── kafka/
│   │   │   ├── producer/
│   │   │   └── consumer/
│   │   └── config/
│   └── src/main/resources/application.yml
│
├── inventory-service/
│   ├── src/main/java/com/bitlord/inventoryservice/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── model/
│   │   ├── dto/
│   │   ├── kafka/
│   │   │   ├── producer/
│   │   │   └── consumer/
│   │   └── config/
│   └── src/main/resources/application.yml
│
└── notification-service/
    ├── src/main/java/com/bitlord/notificationservice/
    │   ├── service/
    │   ├── kafka/consumer/
    │   ├── email/
    │   └── config/
    └── src/main/resources/application.yml
```

---

## 12. Security

- JWT-based authentication on all public-facing API Gateway routes.
- Admin endpoints (`PATCH`, `POST` on inventory) restricted to `ROLE_ADMIN`.
- Kafka topics are internal only — not exposed through the gateway.
- Environment variables used for all secrets (DB password, mail credentials, JWT secret). Never hardcoded.

---
---

*Document End — TRD v1.0*
