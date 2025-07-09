## 🔁 Global Logging Configuration
```yaml
x-default-logging: &logging
  driver: "json-file"
  options:
    max-size: "5m"
    max-file: "2"
    tag: "{{.Name}}"
```

#### x-default-logging: 
* This is an anchor (YAML extension) defining a reusable block named logging.

#### &logging
* Declares an anchor reference, allowing you to reuse this block elsewhere using *logging.

#### driver: "json-file" 
* Specifies the Docker logging driver. json-file is the default, storing logs in JSON format.

#### options: 
* Custom options for the logging driver:

- `max-size: "5m"` – log files will rotate when they reach 5 megabytes.
- `max-file: "2" ` – keeps only the 2 most recent rotated logs.
- `tag: "{{.Name}}" ` – log tag uses container name dynamically ({{.Name}} is a Docker log template).

🌐 Network Definition
yaml
Copy
Edit
networks:
  default:
    name: opentelemetry-demo
    driver: bridge
Defines a network named opentelemetry-demo using the bridge driver.

default: means this network will be the default one for all services unless another is specified.

🧱 Service Block: accounting
yaml
Copy
Edit
services:
  # ******************
  # Core Demo Services
  # ******************
  # Accounting service
This is a comment block organizing and labeling services logically.

📦 Accounting Service
yaml
Copy
Edit
  accounting:
    image: ${IMAGE_NAME}:${DEMO_VERSION}-accounting
accounting: – service name (used in DNS, etc.)

image: – defines the image name dynamically using environment variables:

${IMAGE_NAME} – base name (e.g., otel-demo)

${DEMO_VERSION} – tag version (e.g., v1)

Results in something like: otel-demo:v1-accounting

yaml
Copy
Edit
    container_name: accounting
Sets the container name explicitly (optional, usually auto-generated).

yaml
Copy
Edit
    build:
      context: ./
      dockerfile: ${ACCOUNTING_DOCKERFILE}
      cache_from:
        - ${IMAGE_NAME}:${IMAGE_VERSION}-accounting
build: – Build configuration if you're building the image locally:

context: ./ – current directory is the build context.

dockerfile: – path to Dockerfile, provided via env variable ${ACCOUNTING_DOCKERFILE}.

cache_from: – uses an existing image for build cache to speed up builds.

yaml
Copy
Edit
    deploy:
      resources:
        limits:
          memory: 120M
deploy: – Used mostly in Docker Swarm, but some fields work locally:

Limits memory usage to 120MB for the container.

yaml
Copy
Edit
    restart: unless-stopped
Restart policy:

Automatically restarts the container unless it's manually stopped.

yaml
Copy
Edit
    environment:
      - KAFKA_ADDR
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://${OTEL_COLLECTOR_HOST}:${OTEL_COLLECTOR_PORT_HTTP}
      - OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE
      - OTEL_RESOURCE_ATTRIBUTES
      - OTEL_SERVICE_NAME=accounting
environment: – sets environment variables inside the container:

KAFKA_ADDR – Kafka broker address (value taken from .env)

OTEL_EXPORTER_OTLP_ENDPOINT – where the app sends telemetry (to OpenTelemetry Collector).

OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE – OTEL metric config (value from env).

OTEL_RESOURCE_ATTRIBUTES – service metadata.

OTEL_SERVICE_NAME=accounting – explicitly defines service name for telemetry.

yaml
Copy
Edit
    depends_on:
      otel-collector:
        condition: service_started
      kafka:
        condition: service_healthy
depends_on: – specifies service startup order:

otel-collector must be started

kafka must be healthy

Ensures accounting starts only after these dependencies are ready.

yaml
Copy
Edit
    logging: *logging
Applies the shared logging configuration defined earlier (*logging is the anchor reference to &logging).

