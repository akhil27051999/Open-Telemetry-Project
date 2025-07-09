# 🛰️ OpenTelemetry Demo service to service flow

## 📐 Service-to-Service flow in cronological order

### 1. Foundational Infrastructure 
1. OpenSearch starts first to store logs and traces that will be sent from OpenTelemetry Collector.
2. Jaeger starts next as it will be used by OpenTelemetry Collector to visualize distributed traces.
3. Prometheus begins to scrape metrics from services and store time-series data.
4. Grafana starts and connects to Prometheus and OpenSearch to visualize logs, metrics, and traces.
5. OpenTelemetry Collector then starts and becomes the central point where all services send traces, logs, and metrics.
6. Kafka boots up to act as the messaging queue between checkout → accounting and fraud detection services.
7. Valkey (Redis) starts to serve as the fast in-memory database for the Cart Service.

| Service         | Purpose                                      |
|----------------|----------------------------------------------|
| **OpenSearch**  | Backend for storing logs and traces          |
| **Jaeger**      | Trace UI (used by OpenTelemetry Collector)   |
| **Prometheus**  | Metrics collection                           |
| **Grafana**     | Visualization of metrics/logs/traces         |
| **OTel Collector** | Aggregates traces, logs, and metrics       |
| **Kafka**       | Messaging bus (used by Checkout, Fraud, etc.)|
| **Valkey (Redis)** | Cart service backend cache                |

---

### 2. Feature Flag Services

1. flagd starts and listens for feature flag evaluations requested by various services.
2. flagd-ui is brought up to allow UI-based configuration of the flags used by flagd.

| Service     | Purpose                             |
|-------------|-------------------------------------|
| **flagd**   | Feature flag evaluation engine      |
| **flagd-ui**| Web UI for managing feature flags   |

---

### 3. Core Backend Microservices (Business Logic)

1. Currency Service starts to provide currency conversion during checkout.
2. Quote Service starts to offer shipping cost estimations.
3. Product Catalog starts to offer all product listings, used by recommendation, frontend, and other services.
4. Ad Service is initialized to deliver relevant ads to the frontend.
5. Image Provider starts and statically serves product images, mostly accessed by frontend.
6. Recommendation Service starts to provide personalized product recommendations based on viewed product IDs.
7. Cart Service comes up and connects with Valkey to persist shopping cart data per user.
8. Payment Service starts to handle payment processing during checkout (mocked for demo).
9. Email Service is ready to send confirmation emails after successful orders.
10. Fraud Detection Service comes online and listens to Kafka to check for fraudulent transactions.
11. Accounting Service also listens to Kafka to record completed transactions.
12. Shipping Service starts to calculate shipping costs and generate tracking information.

| Service             | Role                                       | Dependencies    |
|---------------------|--------------------------------------------|-----------------|
| Currency Service     | Currency conversion                        | -               |
| Quote Service        | Returns shipping quotes                    | -               |
| Product Catalog      | Product metadata and search                | -               |
| Ad Service           | Delivers ads to frontend                   | -               |
| Image Provider       | Static image host (NGINX)                  | -               |
| Recommendation       | Product recommendations                    | Product Catalog |
| Cart Service         | Cart logic with Valkey backend             | Valkey          |
| Payment Service      | Payment processing                         | -               |
| Email Service        | Sends confirmation emails                  | -               |
| Fraud Detection      | Fraud analysis via Kafka                   | Kafka           |
| Accounting           | Order ledger storage (mocked)              | Kafka           |
| Shipping Service     | Calculates shipping cost                   | Quote Service   |

---

### 4. Orchestration Services

1. Checkout Service starts now that all its dependencies (cart, currency, payment, shipping, email, product catalog, kafka, flagd) are ready.
   It coordinates the end-to-end flow when a user clicks on "Checkout", calling all necessary services to place the order.

| Service          | Description                                  |
|------------------|----------------------------------------------|
| **Checkout**     | Central service that coordinates orders       |
|                  | Depends on: cart, payment, currency, shipping, email, product-catalog, kafka, flagd |

---

### 5. Frontend Services (UI Layer)

#### Frontend service starts to provide the user-facing UI. It communicates with:

1. Ad Service for product ads
2. Cart Service for cart items
3. Checkout for placing orders
4. Currency Service for displaying prices
5. Product Catalog for listings
6. Shipping Service for delivery estimates
7. Image Provider for product images
8. Recommendation Service for product suggestions
9. Quote Service for estimated shipping costs
10. Frontend Proxy (Envoy) starts as a reverse proxy, exposing services like frontend, Grafana, Jaeger, flagd-ui, and load generator.
11. React Native App (if present) starts to offer a mobile interface for users using the same backend services.

| Service           | Description                                 |
|-------------------|---------------------------------------------|
| **Frontend**      | React + Next.js app with API routes         |
|                   | Depends on: ad, cart, checkout, currency, product-catalog, shipping, image-provider, recommendation, quote |
| **Frontend Proxy**| Envoy reverse proxy for frontend + observability UIs |
|                   | Depends on: frontend, jaeger, grafana, load-generator, flagd-ui |
| **React Native App** | Mobile version (optional)                |

---

### 6. Load Testing Services

1. Load Generator starts last to simulate user traffic to the frontend.
   It performs automated UI interactions (viewing products, adding to cart, placing orders) to test the system under load.

| Service           | Description                                 |
|-------------------|---------------------------------------------|
| **Load Generator**| Locust-based tool to simulate traffic       |
|                   | Depends on: frontend                        |


## 🧭 Summary of Chronological Service-to-Service Flow

**From Infrastructure → Feature Flags → Core Services → Orchestration → UI → Testing**

This order ensures:

- All telemetry, logging, and storage systems are ready before services start emitting data.
- Core services have their dependencies like Redis, Kafka, or Product APIs initialized.
- Checkout can fully function by calling all core services.
- Frontend can render pages correctly by retrieving data from backend services.
- Load Generator simulates real users only after all services are operational.



