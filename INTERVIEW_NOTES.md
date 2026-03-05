# INTERVIEW_NOTES

Use this as your mock-interview talk track. README stays run-focused; this file is presentation-focused.

## A) Multithreading / Thread Pool / CompletableFuture

Why:
- My services make I/O calls to other services.
- If I block request threads serially, throughput drops and latency spikes.
- I use a dedicated thread pool for async orchestration so Tomcat request threads are not tied up waiting on slow downstream calls.

What I implemented:
- `order-service/src/main/java/org/example/com/orderservice/config/AppConfig.java`
  - `ThreadPoolTaskExecutor` bean: `orderWorkflowExecutor`
  - Core concepts: core pool size, max pool size, queue capacity
- `order-service/src/main/java/org/example/com/orderservice/workflow/OrderWorkflowServiceImpl.java`
  - `CompletableFuture.supplyAsync(...)` for parallel product + inventory checks
  - `CompletableFuture.allOf(...).join()` to synchronize both tasks
  - fallback/error response when downstream fails
- `order-service/src/main/java/org/example/com/orderservice/controller/OrderController.java`
  - `POST /orders/preview` endpoint to demonstrate async orchestration

How I explain it in interview:
- "This is an I/O-bound workflow, so I parallelized independent remote calls."
- "I isolated async work on a dedicated executor instead of using common pool defaults."
- "I merge results only after both futures complete and return deterministic response states."

How to demo:
1. Start services.
2. Run `POST /orders/preview` with valid product and quantity.
3. Use Postman Runner for 5-20 iterations.
4. Show stable responses and discuss thread pool behavior.

Tradeoffs I mention:
- Too many threads -> context switching and memory pressure.
- Queue too large -> hidden latency and delayed failures.
- Rejection policy should be explicit in production.
- Add timeout/circuit breaker for robust production behavior.

Metrics to discuss:
- Active threads
- Queue depth
- Rejected task count
- P95/P99 latency of preview endpoint

Note on ExecutorService term:
- `ThreadPoolTaskExecutor` is Spring's executor abstraction and internally backed by Java thread pool mechanics.
- In a pure Java style, same pattern can be done with `ExecutorService` + `CompletableFuture`.

## B) Transactions (DB + application-level consistency)

DB-level transaction scope:
- `order-service/src/main/java/org/example/com/orderservice/impl/OrderServiceImpl.java`
  - `@Transactional` on `createOrder`, `cancelOrder`, `markOrderReserved`, `markOrderRejected`
  - `@Transactional(readOnly = true)` on `getOrder`
- `inventory-service/src/main/java/org/example/com/inventoryservice/impl/InventoryServiceImpl.java`
  - `@Transactional` on `upsertInventory`, `reserve`, `release`
  - `@Transactional(readOnly = true)` on `getInventory`

How I explain it:
- "Within one service boundary, transaction guarantees atomicity and rollback behavior for DB writes."
- "Across services, I avoid distributed XA transactions; I use event-driven eventual consistency."

Application-level consistency (Saga-like):
- Order created -> event `order-created`
- Inventory reserves/rejects -> event `inventory-reserved` or `inventory-rejected`
- Order updates final status based on inventory result

Idempotency and delivery model (talking points):
- Kafka is at-least-once, so consumers should be idempotent.
- Reservation table keyed by `orderId` is a foundation for idempotent processing.
- In production, I'd add outbox pattern + deduplication key + retry policy.

Demo scenario:
1. Create order with valid quantity -> status becomes `RESERVED`.
2. Create order with invalid quantity -> status becomes `REJECTED:INVALID_QUANTITY`.
3. Cancel reserved order -> inventory is released.

## C) Security (Gateway + Filter chain + service validation)

Gateway-level security:
- `api-gateway/src/main/java/org/example/com/apigateway/filters/AuthenticationFilter.java`
  - `GlobalFilter` checks header `role: admin`
  - blocks unauthorized requests before routing

Service-level API security:
- `ai-assistant-service/src/main/java/org/example/com/aiassistantservice/config/SecurityPlatformConfig.java`
  - injects JWT secret into shared validator
- `ai-assistant-service/src/main/java/org/example/com/aiassistantservice/controller/AiAssistantController.java`
  - validates bearer token with `JwtTokenValidator`
- `user-service/src/main/java/org/example/com/userservice/config/JwtAuthenticationFilter.java`
  - `OncePerRequestFilter` in Spring Security chain

How I explain filter chain:
- "Request first hits gateway filter for coarse-grained policy."
- "Then service-level filters/controllers perform fine-grained JWT validation."
- "This is defense in depth: gateway compromise does not fully bypass service controls."

Method-level security note:
- Current code uses filter/controller validation.
- Next hardening step: add `@EnableMethodSecurity` + `@PreAuthorize` on sensitive methods.

Demo script:
1. Call protected endpoint without `role` header -> `401` from gateway.
2. Call `/ai/assist` with role but invalid/missing JWT -> `401` from AI service.
3. Call with valid role and valid JWT -> `200` response.

## Short Senior-level close

"I split concerns by layer: async orchestration for throughput, transactional boundaries for local consistency, and multi-layer security for defense in depth. For distributed consistency, I chose event-driven Saga-style patterns over distributed DB transactions because it scales better and is operationally safer in microservices."
