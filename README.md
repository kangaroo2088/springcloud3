# springcloud3

Simple Spring Cloud sample for interview practice:
- Eureka service discovery
- API Gateway routing
- `user-service`, `product-service`, `order-service`, `inventory-service`, `ai-assistant-service`
- `security-platform` shared JWT validation module
- MySQL persistence
- Kafka event-driven workflow for order and inventory

## Interview Focus (3 Key Concepts)

### 1) Multithreading / Thread Pool / CompletableFuture

Goal: improve throughput and isolate slow I/O calls (product and inventory checks).

Where in code:
- `order-service/src/main/java/org/example/com/orderservice/config/AppConfig.java`
  - `ThreadPoolTaskExecutor` bean (`orderWorkflowExecutor`)
- `order-service/src/main/java/org/example/com/orderservice/workflow/OrderWorkflowServiceImpl.java`
  - `CompletableFuture.supplyAsync(...)` for parallel calls
  - `CompletableFuture.allOf(...).join()` to combine results

Look for:
- `ThreadPoolTaskExecutor` (Executor abstraction)
- `CompletableFuture`
- async orchestration in `/orders/preview`

How to demo:
- Send 5-20 requests quickly to `POST /orders/preview` (Postman Runner).
- Show response remains stable while two downstream calls run in parallel.
- Show thread names from Spring logs (thread pool prefix: `order-workflow-`).
- Explain backpressure controls in thread pool config:
  - core/max pool size
  - queue capacity
  - rejection policy (real-world tuning point)

### 2) Transactions (DB + application level)

Goal: keep single-service DB updates atomic; cross-service consistency via event-driven workflow.

Where in code:
- `order-service/src/main/java/org/example/com/orderservice/impl/OrderServiceImpl.java`
  - `@Transactional` on create/cancel/status updates
- `inventory-service/src/main/java/org/example/com/inventoryservice/impl/InventoryServiceImpl.java`
  - `@Transactional` on reserve/release/upsert

How to demo:
- Create order -> inventory reserves -> order marked `RESERVED`.
- Cancel order -> inventory released.
- Force failure (`quantity <= 0`) -> rejected event and order status `REJECTED:*`.
- Explain: this is service-local transaction + distributed eventual consistency (Saga-like flow through Kafka).

### 3) API Security (Filter + service-level JWT validation)

Goal: gateway-level access control + service-level token validation (defense in depth).

Where in code:
- `api-gateway/src/main/java/org/example/com/apigateway/filters/AuthenticationFilter.java`
  - Global filter requiring `role: admin` for protected routes
- `ai-assistant-service/src/main/java/org/example/com/aiassistantservice/config/SecurityPlatformConfig.java`
  - JWT secret wiring for shared validator
- `ai-assistant-service/src/main/java/org/example/com/aiassistantservice/controller/AiAssistantController.java`
  - validates `Authorization: Bearer ...` via `JwtTokenValidator`
- `user-service/src/main/java/org/example/com/userservice/config/JwtAuthenticationFilter.java`
  - `OncePerRequestFilter` in service filter chain

How to demo:
- No `role` header -> gateway returns `401`.
- Missing/invalid JWT on `/ai/assist` -> service returns `401`.
- Valid `role` + valid JWT -> `200`.
- Explain why gateway-only security is insufficient without service-level checks.

## Quick Start (Docker)

From this folder:

```bash
docker compose up --build
```

Services:
- Eureka: `http://localhost:8761`
- API Gateway: `http://localhost:8183`
- User Service: `http://localhost:8180`
- Product Service: `http://localhost:8081`
- Order Service: `http://localhost:8084`
- Inventory Service: `http://localhost:8085`
- AI Assistant Service: `http://localhost:8086`
- Kafka: `localhost:9092`
- MySQL: `localhost:3306`

## Architecture Overview

- `user-service`: signup/login and user retrieval
- `product-service`: product creation and query
- `order-service`: create/get/cancel orders and manage order status
- `inventory-service`: maintain stock and reserve/release inventory
- `ai-assistant-service`: AI-facing API endpoint with shared JWT validation utilities
- `security-platform`: reusable `JwtTokenValidator` and validation result model
- `api-gateway`: single entry point for `/user/**`, `/product/**`, `/orders/**`, `/inventory/**`
- `eureka`: service registry for service discovery

## Distributed Workflow (Order + Inventory)

1. Create order at `order-service` (`POST /orders`)
2. `order-service` publishes `order-created`
3. `inventory-service` consumes `order-created` and tries to reserve stock
4. `inventory-service` publishes either:
   - `inventory-reserved` (success)
   - `inventory-rejected` (out of stock / invalid quantity)
5. `order-service` consumes inventory result and updates order status

Cancellation flow:
- `POST /orders/{id}/cancel` publishes `order-cancelled`
- `inventory-service` consumes and releases reserved stock

## Core Endpoints (via Gateway)

- User:
  - `POST /user`
  - `POST /user/login`
  - `GET /user/{id}`
- Product:
  - `POST /product`
  - `GET /product/{id}`
- Inventory:
  - `PUT /inventory/{productId}`
  - `GET /inventory/{productId}`
- Order:
  - `POST /orders`
  - `GET /orders/{id}`
  - `POST /orders/{id}/cancel`
- AI:
  - `POST /ai/assist`
  - `GET /ai/health`

## Notes

- Gateway filter currently requires request header `role: admin`.
- Kafka topics used:
  - `product-created`
  - `order-created`
  - `order-cancelled`
  - `inventory-reserved`
  - `inventory-rejected`

## Postman Demo

Import:
- `postman/springcloud3-workflow.postman_collection.json`
- `postman/springcloud3-local.postman_environment.json`

Run requests in order:
1. Signup User
2. Login User
3. Create Product
4. Seed Inventory
5. Create Order
6. Get Order
7. Cancel Order
8. Get Inventory
