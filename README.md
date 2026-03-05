# springcloud3

Simple Spring Cloud sample for interview practice:
- Eureka service discovery
- API Gateway routing
- `user-service`, `product-service`, `order-service`, `inventory-service`, `ai-assistant-service`
- `security-platform` shared JWT validation module
- MySQL persistence
- Kafka event-driven workflow for order and inventory

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
