# 🐳 Docker Compose 

- Docker Compose is a tool for defining and managing multi-container Docker applications using a simple YAML configuration file (docker-compose.yml).
- Instead of running individual `docker run` commands for each service (e.g., frontend, backend, database), Compose lets us define and run all of them together with a single command.

## 🧱 Docker Compose File Structure 

### 1️⃣ **Services**
Each containerized microservice is defined as a service. Key components:

- **`image:`** The image to be used or built
- **`container_name:`** Custom name for the container
- **`build:`**
  - `context:` Build context directory
  - `dockerfile:` Dockerfile path for service
  - `cache_from:` Reuses previous builds to optimize performance
- **`deploy:`** Memory/resource limits for containers
- **`restart:`** Ensures containers restart on failure (`unless-stopped`)
- **`environment:`** Injects environment variables into containers
- **`depends_on:`** Ensures a service only starts once its dependencies are ready
- **`logging:`** Defines the logging driver and its configuration

### 2️⃣ **Networks**
```yaml
networks:
  default:
    name: opentelemetry-demo
    driver: bridge
```
- Used to connect services internally.

### 3️⃣ **Volumes (Storage)**
```yaml
volumes:
  db-data:
```
- Persist data between container restarts.

---
# 📦 Otel Docker Compose File Explaination To Test Services Locally

## 🔁 Global Logging Configuration
```yaml
x-default-logging: &logging
  driver: "json-file"
  options:
    max-size: "5m"
    max-file: "2"
    tag: "{{.Name}}"
```

- `x-default-logging: ` - This is an anchor (YAML extension) defining a reusable block named logging.
- `&logging` -  Declares an anchor reference, allowing you to reuse this block elsewhere using *logging.
- `driver: "json-file" ` -  Specifies the Docker logging driver. json-file is the default, storing logs in JSON format.
- `options: ` - Custom options for the logging driver:
  - `max-size: "5m"` – log files will rotate when they reach 5 megabytes.
  - `max-file: "2" ` – keeps only the 2 most recent rotated logs.
  - `tag: "{{.Name}}" ` – log tag uses container name dynamically ({{.Name}} is a Docker log template).

## 🌐 Networking

```yaml
networks:
  default:
    name: opentelemetry-demo
    driver: bridge
```

- Defines a network named opentelemetry-demo using the bridge driver.
- `default: ` - whcih means this network will be the default one for all services unless another is specified.

## 🧱 Core Services

### 📦 Accounting Service

- **This service calculates the total amount of sold products. This is only mocked and received orders are printed out.**

```yaml
  accounting:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-accounting
    container_name: accounting
    build:
      context: ./
      dockerfile: ${ACCOUNTING_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-accounting
    deploy:
      resources:
        limits:
          memory: 120M
    restart: unless-stopped
    environment:
      - KAFKA_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=accounting
    depends_on:
      otel-collector:
        condition: service_started
      kafka:
        condition: service_healthy
    logging: *logging
```

- `accounting: ` - service name (used in DNS, etc.)
- `image: ` - defines the image name dynamically using environment variables:
  - `${IMAGE_NAME}` -  base name (e.g., otel-demo)
  - `${DEMO_VERSION} ` - tag version (e.g., v1)
- `container_name: accounting` - Sets the container name explicitly (optional, usually auto-generated)

- `build:` - Build configuration if you're building the image locally:
  - `context: ./ ` - current directory is the build context.
  - `dockerfile: ` - path to Dockerfile, provided via env variable ${ACCOUNTING_DOCKERFILE}.
  - `cache_from: ` - uses an existing image for build cache to speed up builds.

- `deploy: ` - Used mostly in Docker Swarm, but some fields work locally:
  - Limits memory usage to 120MB for the container.

- `Restart policy:` - Automatically restarts the container unless it's manually stopped.
  
- `environment:` – sets environment variables inside the container:
  - `KAFKA_ADDR` – Kafka broker address (value taken from .env)
  - `OTEL_EXPORTER_OTLP_ENDPOINT` – where the app sends telemetry (to OpenTelemetry Collector).
  - `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` – OTEL metric config (value from env).
  - `OTEL_RESOURCE_ATTRIBUTES` – service metadata.
  - `OTEL_SERVICE_NAME=accounting` – explicitly defines service name for telemetry.

- `depends_on:` – Ensures accounting starts only after these dependencies are ready. specifies service startup order:
  - otel-collector must be started
  - kafka must be healthy

- `logging: *logging` -  Applies the shared logging configuration defined earlier (*logging is the anchor reference to &logging).

---

### 📢 Ad Service

- **This service determines appropriate ads to serve to users based on context keys. The ads will be for products available in the store.**

```yaml
  ad:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-ad
    container_name: ad
    build:
      context: ./
      dockerfile: ${AD_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-ad
      args:
        OTEL_JAVA_AGENT_VERSION: ${OTEL_JAVA_AGENT_VERSION}
    deploy:
      resources:
        limits:
          memory: 300M
    restart: unless-stopped
    ports:
      - "${AD_PORT}"
    environment:
      - AD_PORT
      - FLAGD_HOST
      - FLAGD_PORT
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_LOGS_EXPORTER=otlp
      - OTEL_SERVICE_NAME=ad
      # Workaround on OSX for https://bugs.openjdk.org/browse/JDK-8345296
      - _JAVA_OPTIONS
    depends_on:
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```

- `AD_PORT` - Port on which the Ad Service listens for HTTP requests
- `FLAGD_HOST` - Hostname/IP of the Feature Flag Daemon (flagd) for dynamic config
- `FLAGD_PORT` - Port on which flagd is running
- `OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}` - Endpoint of the OpenTelemetry Collector to export telemetry data (traces/metrics/logs)
- `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` - Configures whether metrics are cumulative or delta (e.g., 'delta', 'cumulative')
- `OTEL_RESOURCE_ATTRIBUTES` - Metadata about the service (e.g., service.version, deployment.environment)
- `OTEL_LOGS_EXPORTER=otlp` - Specifies that logs should be exported using OTLP protocol
- `OTEL_SERVICE_NAME=ad` - Logical name of this service in traces and metrics (shown in tools like Jaeger/Grafana)

### 🛒 Cart Service

- **This service maintains items placed in the shopping cart by users. It interacts with a Valkey caching service for fast access to shopping cart data.**

```yaml
  cart:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-cart
    container_name: cart
    build:
      context: ./
      dockerfile: ${CART_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-cart
    deploy:
      resources:
        limits:
          memory: 160M
    restart: unless-stopped
    ports:
      - "${CART_PORT}"                           
    environment:
      - CART_PORT                                
      - FLAGD_HOST                               
      - FLAGD_PORT                               
      - VALKEY_ADDR                              
      - OTEL_EXPORTER_OTLP_ENDPOINT              
      - OTEL_RESOURCE_ATTRIBUTES                 
      - OTEL_SERVICE_NAME=cart                  
      - ASPNETCORE_URLS=http://*:${CART_PORT}     
    depends_on:
      valkey-cart:
        condition: service_started
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```

- `CART_PORT` - Port on which the Cart service will run
- `FLAGD_HOST` - Hostname/IP of the Feature Flag Daemon (flagd)
- `FLAGD_PORT` - Port on which flagd is listening
- `VALKEY_ADDR` - Redis-compatible Valkey address for storing cart data (e.g., valkey:6379)
- `OTEL_EXPORTER_OTLP_ENDPOINT` - Endpoint of the OpenTelemetry Collector for exporting telemetry data
- `OTEL_RESOURCE_ATTRIBUTES` - Metadata used for identifying service instance (e.g., environment, version)
- `OTEL_SERVICE_NAME=cart` - Name used by observability tools for this service
- `ASPNETCORE_URLS=http://*:${CART_PORT}` - ASP.NET Core setting to listen on all interfaces on the specified port
 
---

### 📦 Checkout Service

- **This service is responsible to process a checkout order from the user. The checkout service will call many other services in order to process an order.**

```yaml
  checkout:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-checkout
    container_name: checkout
    build:
      context: ./
      dockerfile: ${CHECKOUT_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-checkout
    deploy:
      resources:
        limits:
          memory: 20M
    restart: unless-stopped
    ports:
      - "${CHECKOUT_PORT}"
    environment:
      - FLAGD_HOST
      - FLAGD_PORT
      - CHECKOUT_PORT
      - CART_ADDR
      - CURRENCY_ADDR
      - EMAIL_ADDR
      - PAYMENT_ADDR
      - PRODUCT_CATALOG_ADDR
      - SHIPPING_ADDR
      - KAFKA_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=checkout
    depends_on:
      cart:
        condition: service_started
      currency:
        condition: service_started
      email:
        condition: service_started
      payment:
        condition: service_started
      product-catalog:
        condition: service_started
      shipping:
        condition: service_started
      otel-collector:
        condition: service_started
      kafka:
        condition: service_healthy
      flagd:
        condition: service_started
    logging: *logging
```
  - `FLAGD_HOST` - Hostname/IP of the Feature Flag Daemon
  - `FLAGD_PORT` - Port for flagd
  - `CHECKOUT_PORT` - Port on which the checkout service listens
  - `CART_ADDR`, `CURRENCY_ADDR`, `EMAIL_ADDR`, `PAYMENT_ADDR`, `PRODUCT_CATALOG_ADDR`, `SHIPPING_ADDR` - Internal service DNS names
  - `KAFKA_ADDR` - Kafka broker address for message handling
  - `OTEL_EXPORTER_OTLP_ENDPOINT` - OTLP endpoint for OpenTelemetry Collector
  - `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` - Metrics temporality setting (e.g., delta/cumulative)
  - `OTEL_RESOURCE_ATTRIBUTES` - Metadata about the service instance
  - `OTEL_SERVICE_NAME=checkout` - Name used in telemetry tools

---

### 💱 Currency Service

 - **This service provides functionality to convert amounts between different currencies.**

```yaml
  currency:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-currency
    container_name: currency
    build:
      context: ./
      dockerfile: ${CURRENCY_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-currency
      args:
        OPENTELEMETRY_CPP_VERSION: ${OPENTELEMETRY_CPP_VERSION}
    deploy:
      resources:
        limits:
          memory: 20M
    restart: unless-stopped
    ports:
      - "${CURRENCY_PORT}"
    environment:
      - CURRENCY_PORT
      - VERSION=${IMAGE_VERSION}
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_RESOURCE_ATTRIBUTES=${OTEL_RESOURCE_ATTRIBUTES},service.name=currency   # The C++ SDK does not support OTEL_SERVICE_NAME
    depends_on:
      otel-collector:
        condition: service_started
    logging: *logging
```
  - `CURRENCY_PORT` - Port the service runs on
  - `VERSION` - Application version passed from IMAGE_VERSION
  - `OTEL_EXPORTER_OTLP_ENDPOINT` - OpenTelemetry Collector endpoint
  - `OTEL_RESOURCE_ATTRIBUTES` - Resource attributes including `service.name=currency` since OTEL_SERVICE_NAME isn't supported in C++ SDK

---

### 📧 Email Service

- **This service will send a confirmation email to the user when an order is placed.**

```yaml
  email:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-email
    container_name: email
    build:
      context: ./
      dockerfile: ${EMAIL_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-email
    deploy:
      resources:
        limits:
          memory: 100M
    restart: unless-stopped
    ports:
      - "${EMAIL_PORT}"
    environment:
      - APP_ENV=production
      - EMAIL_PORT
      - OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}/v1/traces
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=email
    depends_on:
      otel-collector:
        condition: service_started
    logging: *logging
```

  - `APP_ENV=production` - Sets app environment to production
  - `EMAIL_PORT` - Port the service runs on
  - `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` - OTLP endpoint for exporting traces
  - `OTEL_RESOURCE_ATTRIBUTES` - Resource attributes for observability
  - `OTEL_SERVICE_NAME=email` - Service name for telemetry

---
### 🔍 Fraud Detection Service

- **This service analyses incoming orders and detects malicious customers. This is only mocked and received orders are printed out.**

```yaml
  fraud-detection:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-fraud-detection
    container_name: fraud-detection
    build:
      context: ./
      dockerfile: ${FRAUD_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-fraud-detection
      args:
        OTEL_JAVA_AGENT_VERSION: ${OTEL_JAVA_AGENT_VERSION}
    deploy:
      resources:
        limits:
          memory: 300M
    restart: unless-stopped
    environment:
      - FLAGD_HOST
      - FLAGD_PORT
      - KAFKA_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_INSTRUMENTATION_KAFKA_EXPERIMENTAL_SPAN_ATTRIBUTES=true
      - OTEL_INSTRUMENTATION_MESSAGING_EXPERIMENTAL_RECEIVE_TELEMETRY_ENABLED=true
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=fraud-detection
    depends_on:
      otel-collector:
        condition: service_started
      kafka:
        condition: service_healthy
    logging: *logging
```

  - `FLAGD_HOST`, `FLAGD_PORT` - Feature flag daemon configuration
  - `KAFKA_ADDR` - Kafka broker connection string
  - `OTEL_EXPORTER_OTLP_ENDPOINT` - Endpoint for OpenTelemetry Collector
  - `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` - OTLP metrics setting
  - `OTEL_INSTRUMENTATION_KAFKA_EXPERIMENTAL_SPAN_ATTRIBUTES` - Enable Kafka experimental span metadata
  - `OTEL_INSTRUMENTATION_MESSAGING_EXPERIMENTAL_RECEIVE_TELEMETRY_ENABLED` - Enable receive side telemetry
  - `OTEL_RESOURCE_ATTRIBUTES` - Resource metadata
  - `OTEL_SERVICE_NAME=fraud-detection` - Service name for observability

---
### 🌐 Frontend Service

- **The frontend is responsible to provide a UI for users, as well as an API leveraged by the UI or other clients. The application is based on Next.JS to provide a React web-based UI and API routes.**

```yaml
  frontend:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-frontend
    container_name: frontend
    build:
      context: ./
      dockerfile: ${FRONTEND_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-frontend
    deploy:
      resources:
        limits:
          memory: 250M
    restart: unless-stopped
    ports:
      - "${FRONTEND_PORT}"
    environment:
      - PORT=${FRONTEND_PORT}
      - FRONTEND_ADDR
      - AD_ADDR
      - CART_ADDR
      - CHECKOUT_ADDR
      - CURRENCY_ADDR
      - PRODUCT_CATALOG_ADDR
      - RECOMMENDATION_ADDR
      - SHIPPING_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_RESOURCE_ATTRIBUTES=${OTEL_RESOURCE_ATTRIBUTES}
      - ENV_PLATFORM
      - OTEL_SERVICE_NAME=frontend
      - PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - WEB_OTEL_SERVICE_NAME=frontend-web
      - OTEL_COLLECTOR_HOST
      - FLAGD_HOST
      - FLAGD_PORT
    depends_on:
      ad:
        condition: service_started
      cart:
        condition: service_started
      checkout:
        condition: service_started
      currency:
        condition: service_started
      product-catalog:
        condition: service_started
      quote:
        condition: service_started
      recommendation:
        condition: service_started
      shipping:
        condition: service_started
      otel-collector:
        condition: service_started
      image-provider:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```
  - `PORT`, `FRONTEND_ADDR` - Frontend port and internal DNS address
  - `AD_ADDR`, `CART_ADDR`, `CHECKOUT_ADDR`, etc. - DNS addresses for dependencies
  - `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_RESOURCE_ATTRIBUTES` - OpenTelemetry settings
  - `ENV_PLATFORM` - Runtime environment identifier (e.g., dev/stage/prod)
  - `OTEL_SERVICE_NAME=frontend` - Telemetry service name
  - `PUBLIC_OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` - Public endpoint to send traces
  - `WEB_OTEL_SERVICE_NAME=frontend-web` - Logical web app identifier in telemetry
  - `OTEL_COLLECTOR_HOST`, `FLAGD_HOST`, `FLAGD_PORT` - Supporting observability and feature flags

---
### 🌐 Frontend Proxy (Envoy)

- **The frontend proxy is used as a reverse proxy for user-facing web interfaces such as the frontend, Jaeger, Grafana, load generator, and feature flag service.**

```yaml
  frontend-proxy:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-frontend-proxy
    container_name: frontend-proxy
    build:
      context: ./
      dockerfile: ${FRONTEND_PROXY_DOCKERFILE}
    deploy:
      resources:
        limits:
          memory: 65M
    restart: unless-stopped
    ports:
      - "${ENVOY_PORT}:${ENVOY_PORT}"
      - 10000:10000
    environment:
      - FRONTEND_PORT
      - FRONTEND_HOST
      - LOCUST_WEB_HOST
      - LOCUST_WEB_PORT
      - GRAFANA_PORT
      - GRAFANA_HOST
      - JAEGER_PORT
      - JAEGER_HOST
      - OTEL_COLLECTOR_HOST
      - IMAGE_PROVIDER_HOST
      - IMAGE_PROVIDER_PORT
      - OTEL_COLLECTOR_PORT_GRPC
      - OTEL_COLLECTOR_PORT_HTTP
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=frontend-proxy
      - ENVOY_PORT
      - FLAGD_HOST
      - FLAGD_PORT
      - FLAGD_UI_HOST
      - FLAGD_UI_PORT
    depends_on:
      frontend:
        condition: service_started
      load-generator:
        condition: service_started
      jaeger:
        condition: service_started
      grafana:
        condition: service_started
      flagd-ui:
        condition: service_started
    dns_search: ""
```
  - Includes ports, hostnames, and telemetry config for all connected services (frontend, Grafana, Jaeger, etc.)
  - `ENVOY_PORT` - Proxy listening port
  - `OTEL_SERVICE_NAME=frontend-proxy` - Service name for tracing traffic through Envoy
---

### 🖼️ Image Provider

- **This service provides the images which are used in the frontend. The images are statically hosted on a NGINX instance. The NGINX server is instrumented with the nginx-otel module.**

```yaml
  image-provider:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-image-provider
    container_name: image-provider
    build:
      context: ./
      dockerfile: ${IMAGE_PROVIDER_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-image-provider
    deploy:
      resources:
        limits:
          memory: 120M
    restart: unless-stopped
    ports:
      - "${IMAGE_PROVIDER_PORT}"
    environment:
      - IMAGE_PROVIDER_PORT
      - OTEL_COLLECTOR_HOST
      - OTEL_COLLECTOR_PORT_GRPC
      - OTEL_SERVICE_NAME=image-provider
      - OTEL_RESOURCE_ATTRIBUTES
    depends_on:
      otel-collector:
        condition: service_started
    logging: *logging
```

  - `IMAGE_PROVIDER_PORT` - Service port
  - `OTEL_COLLECTOR_HOST`, `OTEL_COLLECTOR_PORT_GRPC` - OTLP endpoint for tracing
  - `OTEL_SERVICE_NAME=image-provider` - Service name for observability
  - `OTEL_RESOURCE_ATTRIBUTES` - Metadata

---
### 🧪 Load Generator

- **The load generator is based on the Python load testing framework Locust. By default it will simulate users requesting several different routes from the frontend.**

```yaml
  load-generator:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-load-generator
    container_name: load-generator
    build:
      context: ./
      dockerfile: ${LOAD_GENERATOR_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-load-generator
    deploy:
      resources:
        limits:
          memory: 1500M
    restart: unless-stopped
    ports:
      - "${LOCUST_WEB_PORT}"
    environment:
      - LOCUST_WEB_PORT
      - LOCUST_USERS
      - LOCUST_HOST
      - LOCUST_HEADLESS
      - LOCUST_AUTOSTART
      - LOCUST_BROWSER_TRAFFIC_ENABLED=true
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=load-generator
      - PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python
      - LOCUST_WEB_HOST=0.0.0.0
      - FLAGD_HOST
      - FLAGD_PORT
    depends_on:
      frontend:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```
  - `LOCUST_*` - Locust load testing configurations
  - `OTEL_EXPORTER_OTLP_*` - Telemetry export configs
  - `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_SERVICE_NAME=load-generator` - Resource metadata and service ID
  - `FLAGD_HOST`, `FLAGD_PORT` - Feature flag service
---

### 💳 Payment Service

- **This service is responsible to process credit card payments for orders. It will return an error if the credit card is invalid or the payment cannot be processed.**

```yaml
  payment:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-payment
    container_name: payment
    build:
      context: ./
      dockerfile: ${PAYMENT_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-payment
    deploy:
      resources:
        limits:
          memory: 120M
    restart: unless-stopped
    ports:
      - "${PAYMENT_PORT}"
    environment:
      - PAYMENT_PORT
      - FLAGD_HOST
      - FLAGD_PORT
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=payment
    depends_on:
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```
  - `PAYMENT_PORT`, `FLAGD_HOST`, `FLAGD_PORT` - Service & feature flag config
  - `OTEL_*` - OpenTelemetry configuration
  - `OTEL_SERVICE_NAME=payment` - Logical telemetry service name
---

### 🎁 Product Catalog

- **This service is responsible to return information about products. The service can be used to get all products, search for specific products, or return details about any single product.**

```yaml
  product-catalog:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-product-catalog
    container_name: product-catalog
    build:
      context: ./
      dockerfile: ${PRODUCT_CATALOG_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-product-catalog
    deploy:
      resources:
        limits:
          memory: 20M
    restart: unless-stopped
    ports:
      - "${PRODUCT_CATALOG_PORT}"
    environment:
      - PRODUCT_CATALOG_PORT
      - PRODUCT_CATALOG_RELOAD_INTERVAL
      - FLAGD_HOST
      - FLAGD_PORT
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=product-catalog
    volumes:
      - ./src/product-catalog/products:/usr/src/app/products
    depends_on:
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```
  - `PRODUCT_CATALOG_PORT` - Exposed port
  - `PRODUCT_CATALOG_RELOAD_INTERVAL` - Reload interval for product list
  - `FLAGD_*`, `OTEL_*` - Feature flags & observability
  - `OTEL_SERVICE_NAME=product-catalog` - Telemetry identifier

---
### 🧾 Quote Service

- **This service is responsible for calculating shipping costs, based on the number of items to be shipped. The quote service is called from Shipping Service via HTTP.**

```yaml
  quote:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-quote
    container_name: quote
    build:
      context: ./
      dockerfile: ${QUOTE_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-quote
    deploy:
      resources:
        limits:
          memory: 40M
    restart: unless-stopped
    ports:
      - "${QUOTE_PORT}"
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_PHP_AUTOLOAD_ENABLED=true
      - QUOTE_PORT
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=quote
      - OTEL_PHP_INTERNAL_METRICS_ENABLED=true
    depends_on:
      otel-collector:
        condition: service_started
    logging: *logging
```
  - `QUOTE_PORT` - Service port
  - `OTEL_*` - OpenTelemetry (tracing, metrics, PHP autoload)
  - `OTEL_SERVICE_NAME=quote` - Logical name for tracing system

---
### 🎯 Recommendation Service

- **This service is responsible to get a list of recommended products for the user based on existing product IDs the user is browsing.**

```yaml
  recommendation:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-recommendation
    container_name: recommendation
    build:
      context: ./
      dockerfile: ${RECOMMENDATION_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-recommendation
    deploy:
      resources:
        limits:
          memory: 500M               # This is high to enable supporting the recommendationCache feature flag use case
    restart: unless-stopped
    ports:
      - "${RECOMMENDATION_PORT}"
    environment:
      - RECOMMENDATION_PORT
      - PRODUCT_CATALOG_ADDR
      - FLAGD_HOST
      - FLAGD_PORT
      - OTEL_PYTHON_LOG_CORRELATION=true
      - OTEL_EXPORTER_OTLP_ENDPOINT
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=recommendation
      - PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python
    depends_on:
      product-catalog:
        condition: service_started
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```
  - `RECOMMENDATION_PORT` - Exposed port
  - `PRODUCT_CATALOG_ADDR` - Upstream service
  - `FLAGD_*` - Feature flag config
  - `OTEL_*` - Observability settings including log correlation
  - `PROTOCOL_BUFFERS_PYTHON_IMPLEMENTATION=python` - Ensures compatibility for Python-based telemetry export

---
### 🚚 Shipping Service

- **This service is responsible for providing shipping information including pricing and tracking information, when requested from Checkout Service.**

```yaml
  shipping:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-shipping
    container_name: shipping
    build:
      context: ./
      dockerfile: ${SHIPPING_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-shipping
    deploy:
      resources:
        limits:
          memory: 20M
    restart: unless-stopped
    ports:
      - "${SHIPPING_PORT}"
    environment:
      - SHIPPING_PORT
      - QUOTE_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_GRPC}
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=shipping
    depends_on:
      otel-collector:
        condition: service_started
    logging: *logging
```
  - `SHIPPING_PORT`, `QUOTE_ADDR` - Service port and dependency
  - `OTEL_*` - Observability configs (uses gRPC port)
  - `OTEL_SERVICE_NAME=shipping` - Telemetry ID
---

## 📚 Dependent Services

### 🚩 Flagd Service

- **flagd is the feature flagging daemon used in modern cloud-native applications.**

```yaml
  flagd:
    image: ${FLAGD_IMAGE}
    container_name: flagd
    deploy:
      resources:
        limits:
          memory: 75M
    restart: unless-stopped
    environment:
      - FLAGD_OTEL_COLLECTOR_URI=${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_GRPC}
      - FLAGD_METRICS_EXPORTER=otel
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=flagd
    command: [
      "start",
      "--uri",
      "file:./etc/flagd/demo.flagd.json"
    ]
    ports:
      - 8013
    volumes:
      - ./src/flagd:/etc/flagd
    logging:
      *logging
```
  - `FLAGD_OTEL_COLLECTOR_URI` - OTLP target for telemetry from flagd
  - `FLAGD_METRICS_EXPORTER=otel` - Export metrics using OTLP
  - `OTEL_RESOURCE_ATTRIBUTES` - Service metadata
  - `OTEL_SERVICE_NAME=flagd` - Service ID for flagd

---

### 🖥️ Flagd UI

```yaml
  flagd-ui:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-flagd-ui
    container_name: flagd-ui
    build:
      context: ./
      dockerfile: ${FLAGD_UI_DOCKERFILE}
    deploy:
      resources:
        limits:
          memory: 75M
    restart: unless-stopped
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=flagd-ui
    ports:
      - "${FLAGD_UI_PORT}"
    depends_on:
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    volumes:
      - ./src/flagd:/app/data
```
  - `OTEL_*` - Observability export config
  - `OTEL_SERVICE_NAME=flagd-ui` - Telemetry logical ID
---

### 📬 Kafka

- **This is used as a message queue service to connect the checkout service with the accounting and fraud detection services.**

```yaml
  kafka:
    image: ghcr.io/open-telemetry/demo:1.12.0-kafka
    container_name: kafka
    build:
      context: ./
      dockerfile: ${KAFKA_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-kafka
      args:
        OTEL_JAVA_AGENT_VERSION: ${OTEL_JAVA_AGENT_VERSION}
    deploy:
      resources:
        limits:
          memory: 620M
    restart: unless-stopped
    environment:
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=kafka
      - KAFKA_HEAP_OPTS=-Xmx400m -Xms400m
      # Workaround on OSX for https://bugs.openjdk.org/browse/JDK-8345296
      - _JAVA_OPTIONS
    healthcheck:
      test: nc -z kafka 9092
      start_period: 10s
      interval: 5s
      timeout: 10s
      retries: 10
    logging: *logging
```
  - `KAFKA_ADVERTISED_LISTENERS` - Broker configuration
  - `OTEL_*` - Tracing and metrics config
  - `KAFKA_HEAP_OPTS` - JVM memory tuning
  - `_JAVA_OPTIONS` - Additional options (e.g., OSX workaround)
---

### 🗄️ Valkey (Redis Replacement)
```yaml
  valkey-cart:
    image: ${VALKEY_IMAGE}
    container_name: valkey-cart
    user: valkey
    deploy:
      resources:
        limits:
          memory: 20M
    restart: unless-stopped
    ports:
      - "${VALKEY_PORT}"
    logging: *logging
```
  - No environment variables, just ports and memory limits
---
## 🔭 Telemetry Components

### 🕵️ Jaeger

- **jaeger is an open-source distributed tracing system used for monitoring and troubleshooting microservices.**
- **It helps you visualize the flow of requests through your architecture — from frontend to backend — and understand latency, failures, and bottlenecks**

```yaml
  jaeger:
    image: ${JAEGERTRACING_IMAGE}
    container_name: jaeger
    command:
      - "--memory.max-traces=25000"
      - "--query.base-path=/jaeger/ui"
      - "--prometheus.server-url=http://${PROMETHEUS_ADDR}"
      - "--prometheus.query.normalize-calls=true"
      - "--prometheus.query.normalize-duration=true"
    deploy:
      resources:
        limits:
          memory: 1200M
    restart: unless-stopped
    ports:
      - "${JAEGER_PORT}"         # Jaeger UI
      - "${OTEL_COLLECTOR_PORT_GRPC}"
    environment:
      - METRICS_STORAGE_TYPE=prometheus
    logging: *logging
```
  - `METRICS_STORAGE_TYPE=prometheus` - Stores metrics in Prometheus backend

---

### 📈 Grafana

- **Grafana is an open-source visualization and dashboard tool used to monitor metrics, logs, and traces from various data sources like Prometheus, OpenSearch, Loki, and more.**
- **It helps you understand your system's health in real-time by turning raw observability data into interactive dashboards and alerts.**
  
```yaml
  grafana:
    image: ${GRAFANA_IMAGE}
    container_name: grafana
    deploy:
      resources:
        limits:
          memory: 120M
    restart: unless-stopped
    environment:
      - "GF_INSTALL_PLUGINS=grafana-opensearch-datasource"
    volumes:
      - ./src/grafana/grafana.ini:/etc/grafana/grafana.ini
      - ./src/grafana/provisioning/:/etc/grafana/provisioning/
    ports:
      - "${GRAFANA_PORT}"
    logging: *logging
```
  - `GF_INSTALL_PLUGINS` - Pre-install required Grafana plugins (like OpenSearch)

---

### 📦 OpenTelemetry Collector

- **The OpenTelemetry Collector is a vendor-neutral, pluggable service that receives, processes, and exports telemetry data — including metrics, logs, and traces — from your microservices to backends like Prometheus, Jaeger, Grafana, OpenSearch, etc.**
  
```yaml
  otel-collector:
    image: ${COLLECTOR_CONTRIB_IMAGE}
    container_name: otel-collector
    deploy:
      resources:
        limits:
          memory: 200M
    restart: unless-stopped
    command: [ "--config=/etc/otelcol-config.yml", "--config=/etc/otelcol-config-extras.yml" ]
    user: 0:0
    volumes:
      - ${HOST_FILESYSTEM}:/hostfs:ro
      - ${DOCKER_SOCK}:/var/run/docker.sock:ro
      - ${OTEL_COLLECTOR_CONFIG}:/etc/otelcol-config.yml
      - ${OTEL_COLLECTOR_CONFIG_EXTRAS}:/etc/otelcol-config-extras.yml
    ports:
      - "${OTEL_COLLECTOR_PORT_GRPC}"
      - "${OTEL_COLLECTOR_PORT_HTTP}"
    depends_on:
      jaeger:
        condition: service_started
      opensearch:
        condition: service_healthy
    logging: *logging
    environment:
      - ENVOY_PORT
      - HOST_FILESYSTEM
      - OTEL_COLLECTOR_HOST
      - OTEL_COLLECTOR_PORT_GRPC
      - OTEL_COLLECTOR_PORT_HTTP
```

  - `OTEL_COLLECTOR_HOST`, `OTEL_COLLECTOR_PORT_*` - Used by exporters to send data
  - `HOST_FILESYSTEM` - Allows host metrics collection from `/hostfs`

---
### 📊 Prometheus

- **Prometheus is an open-source monitoring and alerting system built for recording real-time metrics from services and infrastructure.**
- **It is widely used in cloud-native environments, especially with Kubernetes and microservices.**
  
```yaml
  prometheus:
    image: ${PROMETHEUS_IMAGE}
    container_name: prometheus
    command:
      - --web.console.templates=/etc/prometheus/consoles
      - --web.console.libraries=/etc/prometheus/console_libraries
      - --storage.tsdb.retention.time=1h
      - --config.file=/etc/prometheus/prometheus-config.yaml
      - --storage.tsdb.path=/prometheus
      - --web.enable-lifecycle
      - --web.route-prefix=/
      - --web.enable-otlp-receiver
      - --enable-feature=exemplar-storage
    volumes:
      - ./src/prometheus/prometheus-config.yaml:/etc/prometheus/prometheus-config.yaml
    deploy:
      resources:
        limits:
          memory: 300M
    restart: unless-stopped
    ports:
      - "${PROMETHEUS_PORT}:${PROMETHEUS_PORT}"
    logging: *logging
```
  - No environment section, configured entirely via command-line arguments and mounted config file
---

### 🔎 OpenSearch

- **OpenSearch is an open-source search, log analytics, and observability platform, originally forked from Elasticsearch.**
- **It is used for:**
  - **Search and indexing of log and trace data**
  - **Storage of structured and unstructured documents**
  - **Dashboards to explore logs, metrics, and traces via OpenSearch Dashboards (like Kibana)**


```yaml
  opensearch:
    image: ${OPENSEARCH_IMAGE}
    container_name: opensearch
    deploy:
      resources:
        limits:
          memory: 1.1G
    restart: unless-stopped
    environment:
      - cluster.name=demo-cluster
      - node.name=demo-node
      - bootstrap.memory_lock=true
      - discovery.type=single-node
      - OPENSEARCH_JAVA_OPTS=-Xms300m -Xmx300m
      - DISABLE_INSTALL_DEMO_CONFIG=true
      - DISABLE_SECURITY_PLUGIN=true
      # Workaround on OSX for https://bugs.openjdk.org/browse/JDK-8345296
      - _JAVA_OPTIONS
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    ports:
      - "9200"
    healthcheck:
      test: curl -s http://localhost:9200/_cluster/health | grep -E '"status":"(green|yellow)"'
      start_period: 10s
      interval: 5s
      timeout: 30s
      retries: 10
    logging: *logging
```
  - `cluster.name`, `node.name` - Cluster identity
  - `bootstrap.memory_lock=true` - Prevent memory swapping
  - `discovery.type=single-node` - For dev mode
  - JVM tuning via `OPENSEARCH_JAVA_OPTS`
  - OSX workaround via `_JAVA_OPTIONS`















