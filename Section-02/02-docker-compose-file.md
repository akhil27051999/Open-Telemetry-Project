# 📦 Docker Compose File Explaination To Test Services Locally

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

- `ad: ` - service name (used in DNS, etc.)
- `image: ` - defines the image name dynamically using environment variables:
  - `${IMAGE_NAME}` -  base name (e.g., otel-demo)
  - `${DEMO_VERSION} ` - tag version (e.g., v1)
- `container_name: ad` - Sets the container name explicitly (optional, usually auto-generated)

- `build:` - Build configuration if you're building the image locally:
  - `context: ./ ` - current directory is the build context.
  - `dockerfile: ` - path to Dockerfile, provided via env variable ${AD_DOCKERFILE}.
  - `cache_from: ` - uses an existing image for build cache to speed up builds.

- `deploy: ` - Used mostly in Docker Swarm, but some fields work locally:
  - Limits memory usage to 300MB for the container.

- `Restart policy:` - Automatically restarts the container unless it's manually stopped.
  
- `environment:` – sets environment variables inside the container:
  - `AD_PORT` - Port on which the Ad Service listens for HTTP requests
  - `FLAGD_HOST` - Hostname/IP of the Feature Flag Daemon (flagd) for dynamic config
  - `FLAGD_PORT` - Port on which flagd is running
  - `OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}` - Endpoint of the OpenTelemetry Collector to export telemetry data (traces/metrics/logs)
  - `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` - Configures whether metrics are cumulative or delta (e.g., 'delta', 'cumulative')
  - `OTEL_RESOURCE_ATTRIBUTES` - Metadata about the service (e.g., service.version, deployment.environment)
  - `OTEL_LOGS_EXPORTER=otlp` - Specifies that logs should be exported using OTLP protocol
  - `OTEL_SERVICE_NAME=ad` - Logical name of this service in traces and metrics (shown in tools like Jaeger/Grafana)

- `depends_on:` – Ensures accounting starts only after these dependencies are ready. specifies service startup order:
  - otel-collector must be started
  - flagd must be started
    
- `logging: *logging` -  Applies the shared logging configuration defined earlier (*logging is the anchor reference to &logging).
---

### 🛒 Cart Service

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
      - "${CART_PORT}"                           # Maps the container's exposed port to the host using CART_PORT env variable
    environment:
      - CART_PORT                                # Port on which the Cart service will run
      - FLAGD_HOST                                # Hostname/IP of the Feature Flag Daemon (flagd)
      - FLAGD_PORT                                # Port on which flagd is listening
      - VALKEY_ADDR                               # Redis-compatible Valkey address for storing cart data (e.g., valkey:6379)
      - OTEL_EXPORTER_OTLP_ENDPOINT               # Endpoint of the OpenTelemetry Collector for exporting telemetry data
      - OTEL_RESOURCE_ATTRIBUTES                  # Metadata used for identifying service instance (e.g., environment, version)
      - OTEL_SERVICE_NAME=cart                    # Name used by observability tools for this service
      - ASPNETCORE_URLS=http://*:${CART_PORT}     # ASP.NET Core setting to listen on all interfaces on the specified port
    depends_on:
      valkey-cart:
        condition: service_started
      otel-collector:
        condition: service_started
      flagd:
        condition: service_started
    logging: *logging
```

- `environment:` – sets environment variables inside the container:
  - `CART_PORT` - Port on which the Cart service will run
  - `FLAGD_HOST` - Hostname/IP of the Feature Flag Daemon (flagd)
  - `FLAGD_PORT` - Port on which flagd is listening
  - `VALKEY_ADDR` - Redis-compatible Valkey address for storing cart data (e.g., valkey:6379)
  - `OTEL_EXPORTER_OTLP_ENDPOINT` - Endpoint of the OpenTelemetry Collector for exporting telemetry data
  - `OTEL_RESOURCE_ATTRIBUTES` - Metadata used for identifying service instance (e.g., environment, version)
  - `OTEL_SERVICE_NAME=cart` - Name used by observability tools for this service
  - `ASPNETCORE_URLS=http://*:${CART_PORT}` - ASP.NET Core setting to listen on all interfaces on the specified port
 
---
