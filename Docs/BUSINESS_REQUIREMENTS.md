# Business Requirements Document (BRD)
## Bitlord's Computer Parts — Event-Driven Order & Inventory System

**Version:** 1.0  
**Date:** 2026-05-15  
**Author:** Malinga Bandara (BitLord)  
**Status:** Draft

---

## 1. Executive Summary

Bitlord's Computer Parts is a computer hardware retailer that requires a modern, scalable backend system to manage product orders, track inventory levels in real time, and notify stakeholders of critical order and stock events. This document defines the business requirements for an event-driven Order & Inventory Management System built on a microservices architecture.

---

## 2. Business Objectives

| # | Objective |
|---|-----------|
| BO-01 | Enable customers to place and track orders for computer hardware products in real time. |
| BO-02 | Maintain accurate, up-to-date inventory levels across all product categories automatically. |
| BO-03 | Notify relevant parties (customers, warehouse staff, administrators) of important order and stock events promptly. |
| BO-04 | Reduce manual intervention in stock management by automating inventory deduction upon order placement. |
| BO-05 | Provide a foundation for future scalability as the business grows. |

---

## 3. Scope

### 3.1 In Scope
- Order placement, order status tracking, and order lifecycle management.
- Inventory stock tracking per product (Computer Parts: CPUs, GPUs, RAM, SSDs, PSUs, Cases, Cooling, Motherboards).
- Automated inventory deduction when an order is confirmed.
- Low-stock alerting when inventory falls below defined thresholds.
- Email and system log notifications for key events (order placed, order confirmed, order failed, low stock).
- Admin-level visibility into orders and inventory.

### 3.2 Out of Scope
- Payment gateway integration (future phase).
- Customer-facing frontend UI (backend API only for this phase).
- Multi-warehouse management.
- Supplier purchase order management.

---

## 4. Stakeholders

| Stakeholder | Role | Interest |
|-------------|------|----------|
| Customer | End user | Place orders, receive order status updates |
| Warehouse Staff | Internal | Receive low-stock and fulfillment alerts |
| System Administrator | Internal | Monitor system health, manage inventory |
| Developer (BitLord) | Builder | Build and maintain the system |

---

## 5. Business Rules

| ID | Rule |
|----|------|
| BR-01 | An order can only be confirmed if sufficient stock is available for all items. |
| BR-02 | Inventory stock must be decremented immediately upon order confirmation. |
| BR-03 | If stock is insufficient, the order must be automatically rejected with a reason. |
| BR-04 | A low-stock alert must be triggered when a product's quantity falls below its defined minimum threshold. |
| BR-05 | All order status changes must generate a notification event. |
| BR-06 | An order must have a unique identifier traceable across all services. |
| BR-07 | Notification delivery failure must not affect the order or inventory process. |

---

## 6. Functional Requirements

### 6.1 Order Management
- **FR-01:** The system shall allow placement of a new order containing one or more computer part line items.
- **FR-02:** The system shall assign a unique Order ID to every new order.
- **FR-03:** The system shall track and expose the status of an order: `PENDING → CONFIRMED → SHIPPED → DELIVERED` or `FAILED`.
- **FR-04:** The system shall allow retrieval of order details by Order ID.
- **FR-05:** The system shall allow listing all orders with optional status filter.

### 6.2 Inventory Management
- **FR-06:** The system shall maintain a stock quantity for each product (SKU).
- **FR-07:** The system shall automatically reduce stock when an order is confirmed.
- **FR-08:** The system shall restore stock if an order confirmation fails.
- **FR-09:** The system shall expose an endpoint to view current stock levels per product.
- **FR-10:** The system shall expose an endpoint to manually adjust stock (admin use).
- **FR-11:** The system shall trigger a low-stock event when stock drops below a configurable threshold (default: 5 units).

### 6.3 Notification Management
- **FR-12:** The system shall send a notification when an order is placed.
- **FR-13:** The system shall send a notification when an order is confirmed or rejected.
- **FR-14:** The system shall send a notification when stock falls below the low-stock threshold.
- **FR-15:** Notifications shall be delivered via email (SMTP) and logged to the system.

---

## 7. Non-Functional Requirements

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-01 | **Availability** | Services should maintain 99%+ uptime under normal load. |
| NFR-02 | **Scalability** | Each microservice must be independently scalable. |
| NFR-03 | **Fault Tolerance** | Notification failures must not cascade to order or inventory services. |
| NFR-04 | **Observability** | All inter-service calls and Kafka events must be traceable end-to-end. |
| NFR-05 | **Latency** | Order placement API response time should be under 500ms. |
| NFR-06 | **Security** | All API endpoints must be protected (JWT-based authentication). |
| NFR-07 | **Auditability** | All order status transitions must be logged with timestamps. |

---

## 8. Product Categories (Reference Data)

The system must support inventory tracking for the following computer part categories:

- Central Processing Units (CPU)
- Graphics Processing Units (GPU)
- Memory / RAM
- Solid State Drives (SSD)
- Hard Disk Drives (HDD)
- Motherboards
- Power Supply Units (PSU)
- PC Cases / Chassis
- Cooling Solutions (Air / Liquid)
- Peripherals (Keyboard, Mouse, Monitor)

---

## 9. Order Lifecycle

```
Customer Places Order
        │
        ▼
  [PENDING] ──── Inventory Check ────► Insufficient Stock ──► [FAILED] ──► Notify Customer
        │
        │ Stock Available
        ▼
  [CONFIRMED] ──► Inventory Deducted ──► Notify Customer & Warehouse
        │
        ▼
  [SHIPPED] ──► Notify Customer
        │
        ▼
  [DELIVERED] ──► Notify Customer
```

---

## 10. Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Orders can be placed and tracked end-to-end | All order lifecycle states reachable via API |
| Inventory updates automatically on order confirmation | Stock level reflects deduction within 2 seconds |
| Notifications sent for all key events | Logs and email delivery confirmed per event type |
| System recovers gracefully from service failure | Individual service restart does not affect others |

---

*Document End — BRD v1.0*
