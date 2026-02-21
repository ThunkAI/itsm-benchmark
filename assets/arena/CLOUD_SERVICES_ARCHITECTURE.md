# ShopFlow Platform Reference Architecture

## 1. System Overview

**ShopFlow** is a cloud-native e-commerce platform built on a microservices architecture. It enables users to browse products, manage carts, and process secure transactions. The system is orchestrated via Kubernetes and relies on a service mesh for inter-service communication.

## 2. Service Catalog

The following services constitute the core platform. All operational actions must target these specific Configuration Items (CIs).

| Service Name                  | Stack   | Criticality       | Role & Responsibility                                                      | Dependencies                                                                         |
| ----------------------------- | ------- | ----------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| **`frontend-storefront`**     | Node.js | **P1 (Critical)** | Main customer UI. Aggregates data from backend APIs to render the shop.    | `auth-core-service`, `product-catalog-service`, `recommendation-api`, `cart-service` |
| **`auth-core-service`**       | Java    | **P1 (Critical)** | Handles User Login, OAuth token generation, and Session validation.        | `primary-db`                                                                         |
| **`product-catalog-service`** | Go      | **P2 (High)**     | Serves product details, pricing, and inventory status.                     | `inventory-db`, `redis-cache`                                                        |
| **`recommendation-api`**      | Python  | **P3 (Low)**      | Machine Learning service providing "You might also like" suggestions.      | `inventory-db` (Read-Only)                                                           |
| **`cart-service`**            | Node.js | **P2 (High)**     | Manages temporary shopping cart state and session persistence.             | `redis-cache`                                                                        |
| **`order-processor`**         | Java    | **P1 (Critical)** | Orchestrates checkout flow, payment confirmation, and order creation.      | `primary-db`, `payment-processor`, `shipping-api`, `rabbitmq`                        |
| **`payment-processor`**       | Go      | **P1 (Critical)** | Secure proxy/gateway for external payment providers (e.g., Stripe/PayPal). | _External Payment API_                                                               |
| **`shipping-api`**            | Go      | **P2 (High)**     | Calculates shipping rates and generates tracking labels.                   | _External Logistics API_                                                             |

## 3. Infrastructure & Data Layer

The platform relies on the following managed infrastructure components.

| Component          | Type       | Used By                                         | Description                                                                       |
| ------------------ | ---------- | ----------------------------------------------- | --------------------------------------------------------------------------------- |
| **`primary-db`**   | PostgreSQL | `auth-core-service`, `order-processor`          | **Transactional Record.** Stores Users, Orders, and Payment Logs. ACID compliant. |
| **`inventory-db`** | MongoDB    | `product-catalog-service`, `recommendation-api` | **Document Store.** Stores Product Metadata, Descriptions, and Images.            |
| **`redis-cache`**  | Redis      | `product-catalog-service`, `cart-service`       | **Key-Value Store.** Caches prices (TTL 1h) and Cart Sessions (TTL 24h).          |
| **`rabbitmq`**     | Broker     | `order-processor`, `shipping-api`               | **Message Bus.** Handles async "OrderPlaced" events for fulfillment.              |

## 4. Standard Traffic Flows (Tracing Guide)

Use these flows to trace latency or error propagation.

### A. The "Browse" Flow (Read-Heavy)

1. **`frontend-storefront`** requests session validation from **`auth-core-service`**.
2. **`frontend-storefront`** requests suggestions from **`recommendation-api`**.
3. **`recommendation-api`** reads metadata from **`inventory-db`**.

### B. The "Product Detail" Flow (Cache-Heavy)

1. **`frontend-storefront`** requests details from **`product-catalog-service`**.
2. **`product-catalog-service`** checks **`redis-cache`**.

- _Hit:_ Returns cached data.
- _Miss:_ Reads from **`inventory-db`** and updates cache.

### C. The "Checkout" Flow (Transactional)

1. **`frontend-storefront`** submits order to **`order-processor`**.
2. **`order-processor`** authorizes charge via **`payment-processor`**.
3. **`order-processor`** commits record to **`primary-db`**.
4. **`order-processor`** publishes fulfillment event to **`rabbitmq`**.

## 5. Operational Baselines

Typical resource usage for healthy services in Production.

| Service               | Replica Count | Healthy CPU | Healthy Latency | Characteristics                                              |
| --------------------- | ------------- | ----------- | --------------- | ------------------------------------------------------------ |
| `frontend-storefront` | 10            | 10-20%      | 50-100ms        | **I/O Bound.** Latency usually indicates downstream waiting. |
| `auth-core-service`   | 5             | 40-50%      | 20-50ms         | **CPU Bound.** Heavy cryptographic operations.               |
| `order-processor`     | 3             | 15%         | 200-300ms       | **Transactional.** Bursty traffic patterns.                  |
| `product-catalog`     | 8             | 20%         | 10-20ms         | **Memory Bound.** High cache utilization.                    |
| `recommendation-api`  | 4             | 70-80%      | 150-250ms       | **Compute Bound.** ML Inference logic.                       |

Based on the **ShopFlow Platform Reference Architecture**, I have defined a normalized set of **Escalation Assignment Groups**. These groups map logically to the services, technology stacks, and responsibilities defined in the architecture.

You should use this exact list as the `enum` constraints for the `escalate_ticket` tool to ensuring the agent selects a valid destination.

### 1. Functional Assignment Groups

These teams own the application code and business logic.

| Assignment Group            | Scope / Services Owned            | Logic / Rationale                                                                            |
| --------------------------- | --------------------------------- | -------------------------------------------------------------------------------------------- |
| **`Frontend_Engineering`**  | `frontend-storefront`             | Owners of the Node.js UI layer and customer browsing experience.                             |
| **`Identity_Engineering`**  | `auth-core-service`               | Owners of the Critical P1 Java service handling OAuth and Sessions.                          |
| **`Checkout_Engineering`**  | `order-processor`, `cart-service` | Owners of the transactional flow (Order Creation, Cart State) and fulfillment orchestration. |
| **`Financial_Services`**    | `payment-processor`               | Specialized Go team managing the secure gateway and external payment APIs (Stripe/PayPal).   |
| **`Logistics_Engineering`** | `shipping-api`                    | Owners of the shipping calculation logic and external logistics integrations.                |
| **`Catalog_Engineering`**   | `product-catalog-service`         | Owners of the Go-based product data and inventory status logic.                              |
| **`Data_Science_ML`**       | `recommendation-api`              | Owners of the Python/ML stack providing inference and suggestions.                           |

### 2. Infrastructure & Operations Groups

These teams own the shared systems and "passive" components.

| Assignment Group              | Scope / Components Owned                    | Logic / Rationale                                                                             |
| ----------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------- |
| **`Database_Administration`** | `primary-db`, `inventory-db`, `redis-cache` | Owners of data persistence, replication, and caching infrastructure (Postgres, Mongo, Redis). |
| **`Platform_Infrastructure`** | `rabbitmq`, Kubernetes, Service Mesh        | Owners of the underlying compute, message bus, and networking layers.                         |
| **`Security_Engineering`**    | _All Services (Security Context)_           | The destination for Vulnerabilities (CVEs), Attacks (DDoS), or Compliance issues.             |

---

### 3. Implementation: Tool Argument Enum

Use this JSON block to define the `assignment_group` argument in your `escalate_ticket` tool definition.

```json
{
  "name": "escalate_ticket",
  "description": "Escalates the ticket to a specialized human team when the agent cannot resolve the issue.",
  "parameters": {
    "type": "object",
    "properties": {
      "assignment_group": {
        "type": "string",
        "description": "The specific engineering team responsible for the failing component.",
        "enum": [
          "Frontend_Engineering",
          "Identity_Engineering",
          "Checkout_Engineering",
          "Financial_Services",
          "Logistics_Engineering",
          "Catalog_Engineering",
          "Data_Science_ML",
          "Database_Administration",
          "Platform_Infrastructure",
          "Security_Engineering"
        ]
      },
      "reason": {
        "type": "string",
        "description": "A concise technical explanation of why the ticket is being escalated."
      }
    },
    "required": ["assignment_group", "reason"]
  }
}
```

### 4. Routing Guide (Mental Map for the Agent)

- **If the issue is visual/latency on the home page:** `Frontend_Engineering`.
- **If the issue is Login/Tokens:** `Identity_Engineering`.
- **If the issue is "Payment Declined" or "Gateway Error":** `Financial_Services`.
- **If the issue is "Cart not saving" or "Order failed":** `Checkout_Engineering`.
- **If the issue is "Recommendations are weird":** `Data_Science_ML`.
- **If the issue is "Database Connection Pool Exhausted":** `Database_Administration` (usually) OR the App Team causing the leak.
- **If the issue is "RCE Exploit":** `Security_Engineering`.
