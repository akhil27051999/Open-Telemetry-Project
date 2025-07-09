# 🛰️ OpenTelemetry Demo service to service flow

## 📐 Service-to-Service flow in cronological order

### 1. Foundational Infrastructure (Telemetry, Messaging, Storage)

These services must start first as they support metrics, logs, traces, and messaging.

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

| Service     | Purpose                             |
|-------------|-------------------------------------|
| **flagd**   | Feature flag evaluation engine      |
| **flagd-ui**| Web UI for managing feature flags   |

---

### 3. Core Backend Microservices (Business Logic)

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

| Service          | Description                                  |
|------------------|----------------------------------------------|
| **Checkout**     | Central service that coordinates orders       |
|                  | Depends on: cart, payment, currency, shipping, email, product-catalog, kafka, flagd |

---

### 5. Frontend Services (UI Layer)

| Service           | Description                                 |
|-------------------|---------------------------------------------|
| **Frontend**      | React + Next.js app with API routes         |
|                   | Depends on: ad, cart, checkout, currency, product-catalog, shipping, image-provider, recommendation, quote |
| **Frontend Proxy**| Envoy reverse proxy for frontend + observability UIs |
|                   | Depends on: frontend, jaeger, grafana, load-generator, flagd-ui |
| **React Native App** | Mobile version (optional)                |

---

### 6. Load Testing Services

| Service           | Description                                 |
|-------------------|---------------------------------------------|
| **Load Generator**| Locust-based tool to simulate traffic       |
|                   | Depends on: frontend                        |


