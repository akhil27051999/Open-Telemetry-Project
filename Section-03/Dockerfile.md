## 📦 Accounting Service 
**This service calculates the total amount of sold products. This is only mocked and received orders are printed out.**

### Local Build

To build the service binary, run:

```sh
cp pb/demo.proto src/accouting/proto/demo.proto # root context
dotnet build # accounting service context
```

### Dockerfile

```Dockerfile

FROM --platform=${BUILDPLATFORM} mcr.microsoft.com/dotnet/sdk:8.0 AS builder
ARG TARGETARCH
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["/src/accounting/", "Accounting/"]
COPY ["/pb/demo.proto", "Accounting/proto/"]
RUN dotnet restore "./Accounting/Accounting.csproj" -r linux-$TARGETARCH
WORKDIR "/src/Accounting"

RUN dotnet build "./Accounting.csproj" -r linux-$TARGETARCH -c $BUILD_CONFIGURATION -o /app/build

# -----------------------------------------------------------------------------

FROM builder AS publish
ARG TARGETARCH
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./Accounting.csproj" -r linux-$TARGETARCH -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# -----------------------------------------------------------------------------

FROM mcr.microsoft.com/dotnet/aspnet:8.0
USER app
WORKDIR /app
COPY --from=publish /app/publish .

USER root
RUN mkdir -p "/var/log/opentelemetry/dotnet"
RUN chown app "/var/log/opentelemetry/dotnet"
RUN chown app "/app/instrument.sh"
USER app

ENTRYPOINT ["./instrument.sh", "dotnet", "Accounting.dll"]
```
### Explaination

- This Dockerfile is a multi-stage build for a .NET 8-based microservice called Accounting, designed for cross-platform containerization and observability using OpenTelemetry. The process begins with a builder stage that uses the official .NET SDK 8.0 image and targets a specific platform using the --platform=${BUILDPLATFORM} flag. It defines build-time arguments such as TARGETARCH for architecture-specific builds (e.g., amd64, arm64) and sets the build configuration to Release by default. The service source code is copied into the container, including the demo.proto file for gRPC or protocol buffer definitions, and dependencies are restored with dotnet restore. The dotnet build command then compiles the project targeting the specified Linux architecture and outputs the intermediate build files to /app/build.
  
- The second stage, named publish, uses the previously built artifacts to publish the final optimized output using dotnet publish. This command produces a trimmed deployment package inside /app/publish, with the UseAppHost=false flag to avoid platform-specific executable overhead, which keeps the image size smaller.
  
- The final stage uses the lightweight mcr.microsoft.com/dotnet/aspnet:8.0 base runtime image to run the service. It switches to a non-root user (app) for security best practices, sets the working directory to /app, and copies the published output from the previous stage. It then briefly switches to the root user to create and set ownership on a log directory (/var/log/opentelemetry/dotnet) and the instrumentation script (instrument.sh) so that the app user can access them. Finally, the container reverts to the non-root user and uses the instrument.sh script as its entrypoint, wrapping the service launch command dotnet Accounting.dll. This design ensures that the application is built efficiently, runs securely, and is ready for observability through OpenTelemetry instrumentation.
## 📢 Ad Service Dockerfile

```Dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /usr/src/app/

COPY gradlew* settings.gradle* build.gradle .
COPY ./gradle ./gradle

RUN chmod +x ./gradlew
RUN ./gradlew
RUN ./gradlew downloadRepos

COPY . .
COPY ./pb ./proto
RUN chmod +x ./gradlew
RUN ./gradlew installDist -PprotoSourceDir=./proto

#####################################################

FROM eclipse-temurin:21-jre

WORKDIR /usr/src/app/

COPY --from=builder /usr/src/app/ ./

ENV AD_PORT 9099

ENTRYPOINT ["./build/install/opentelemetry-demo-ad/bin/Ad"]
```

## 🛒 Cart Service Dockerfile

```Dockerfile

FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS builder
ARG TARGETARCH

WORKDIR /usr/src/app/

COPY ./src/cart/ ./
COPY ./pb/ ./pb/

RUN dotnet restore ./src/cart.csproj -r linux-musl-$TARGETARCH

RUN dotnet publish ./src/cart.csproj -r linux-musl-$TARGETARCH --no-restore -o /cart

# -----------------------------------------------------------------------------

# https://mcr.microsoft.com/v2/dotnet/runtime-deps/tags/list
FROM mcr.microsoft.com/dotnet/runtime-deps:8.0-alpine3.20

WORKDIR /usr/src/app/
COPY --from=builder /cart/ ./

ENV DOTNET_HOSTBUILDER__RELOADCONFIGONCHANGE=false

EXPOSE ${CART_PORT}
ENTRYPOINT [ "./cart" ]
```
## 📦 Checkout Service Dockerfile

```Dockerfile

FROM golang:1.22-alpine AS builder

WORKDIR /usr/src/app/

RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=bind,source=./src/checkout/go.sum,target=go.sum \
    --mount=type=bind,source=./src/checkout/go.mod,target=go.mod \
    go mod download

RUN --mount=type=cache,target=/go/pkg/mod/ \
    --mount=type=cache,target=/root/.cache/go-build \
    --mount=type=bind,rw,source=./src/checkout,target=. \
    go build -ldflags "-s -w" -o /go/bin/checkout/ ./

FROM alpine

WORKDIR /usr/src/app/

COPY --from=builder /go/bin/checkout/ ./

EXPOSE ${CHECKOUT_PORT}
ENTRYPOINT [ "./checkout" ]
```
## 💱 Currency Service Dockerfile

```Dockerfile

FROM alpine:3.18 AS builder

RUN apk update && apk add git cmake make g++ grpc-dev protobuf-dev linux-headers

ARG OPENTELEMETRY_CPP_VERSION

RUN git clone --depth 1 --branch v${OPENTELEMETRY_CPP_VERSION} https://github.com/open-telemetry/opentelemetry-cpp \
    && cd opentelemetry-cpp/ \
    && mkdir build \
    && cd build \
    && cmake .. -DCMAKE_CXX_STANDARD=17 -DCMAKE_POSITION_INDEPENDENT_CODE=ON \
          -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF \
          -DWITH_EXAMPLES=OFF -DWITH_OTLP_GRPC=ON -DWITH_ABSEIL=ON \
    && make -j$(nproc || sysctl -n hw.ncpu || echo 1) install && cd ../..

COPY ./src/currency /currency
COPY ./pb/demo.proto /currency/proto/demo.proto

RUN cd /currency \
    && mkdir -p build && cd build \
    && cmake .. \
    && make -j$(nproc || sysctl -n hw.ncpu || echo 1) install


FROM alpine:3.18 AS release

RUN apk update && apk add grpc-dev protobuf-dev
COPY --from=builder /usr/local /usr/local

EXPOSE ${CURRENCY_PORT}
ENTRYPOINT ["sh", "-c", "./usr/local/bin/currency ${CURRENCY_PORT}"]

```

## 📧 Email Service Dockerfile


```Dockerfile
FROM ruby:3.2.2-slim AS base

FROM base AS builder

WORKDIR /tmp

COPY ./src/email/Gemfile ./src/email/Gemfile.lock ./

#RUN apk update && apk add make gcc musl-dev gcompat && bundle install
RUN apt-get update && apt-get install build-essential -y && bundle install
FROM base AS release

WORKDIR /email_server

COPY ./src/email/ .

RUN chmod 666 ./Gemfile.lock

COPY --from=builder /usr/local/bundle/ /usr/local/bundle/


EXPOSE ${EMAIL_PORT}
ENTRYPOINT ["bundle", "exec", "ruby", "email_server.rb"]

```

## 🚩 Flagd-UI Service Dockerfile

```Dockerfile

FROM node:20 AS builder

WORKDIR /app

COPY ./src/flagd-ui/package*.json ./

RUN npm ci

COPY ./src/flagd-ui/. ./

RUN npm run build

# -----------------------------------------------------------------------------

FROM node:20-alpine

WORKDIR /app

COPY ./src/flagd-ui/package*.json ./

RUN npm ci --only=production

COPY --from=builder /app/src/instrumentation.ts ./instrumentation.ts
COPY --from=builder /app/next.config.mjs ./next.config.mjs

COPY --from=builder /app/.next ./.next

EXPOSE 4000

CMD ["npm", "start"]
```

## 🔍 Fraud Detection Service Dockerfile

```Dockerfile
FROM --platform=${BUILDPLATFORM} gradle:8-jdk17 AS builder

WORKDIR /usr/src/app/

COPY ./src/fraud-detection/ ./
COPY ./pb/ ./src/main/proto/
RUN gradle shadowJar

# -----------------------------------------------------------------------------

FROM gcr.io/distroless/java17-debian11

ARG OTEL_JAVA_AGENT_VERSION
WORKDIR /usr/src/app/

COPY --from=builder /usr/src/app/build/libs/fraud-detection-1.0-all.jar ./
ADD --chmod=644 https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v$OTEL_JAVA_AGENT_VERSION/opentelemetry-javaagent.jar /app/opentelemetry-javaagent.jar
ENV JAVA_TOOL_OPTIONS=-javaagent:/app/opentelemetry-javaagent.jar

ENTRYPOINT [ "java", "-jar", "fraud-detection-1.0-all.jar" ]
```

## 🌐 Frontend-Proxy Service Dockerfile

```Dockerfile
FROM envoyproxy/envoy:v1.32-latest
RUN apt-get update && apt-get install -y gettext-base && apt-get clean && rm -rf /var/lib/apt/lists/*

USER envoy
WORKDIR /home/envoy
COPY ./src/frontend-proxy/envoy.tmpl.yaml envoy.tmpl.yaml

ENTRYPOINT ["/bin/sh", "-c", "envsubst < envoy.tmpl.yaml > envoy.yaml && envoy -c envoy.yaml;"]
```

## 🖥️ Frontend Service Dockerfile

```Dockerfile

FROM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat

WORKDIR /app

COPY ./src/frontend/package*.json ./
RUN npm ci

FROM node:20-alpine AS builder
RUN apk add --no-cache libc6-compat protobuf-dev protoc
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY ./pb ./pb
COPY ./src/frontend .

RUN npm run grpc:generate
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
RUN apk add --no-cache protobuf-dev protoc

ENV NODE_ENV=production

RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

COPY --from=builder /app/next.config.js ./
COPY --from=builder /app/utils/telemetry/Instrumentation.js ./
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=deps /app/node_modules ./node_modules

COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

ENV PORT=8080
EXPOSE ${PORT}

ENTRYPOINT ["npm", "start"]
```

## 🖼️ Image-provider Service Dockerfile

```Dockerfile
FROM nginx:1.27.0-otel

RUN apt-get update ; apt-get install lsb-release --no-install-recommends --no-install-suggests -y

# This file is needed for nginx-module-otel to be found.
RUN echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian `lsb_release -cs` nginx" | tee /etc/apt/sources.list.d/nginx.list

RUN apt-get update ; apt-get install nginx-module-otel --no-install-recommends --no-install-suggests -y

RUN mkdir /static
COPY src/image-provider/static /static

EXPOSE ${IMAGE_PROVIDER_PORT}

STOPSIGNAL SIGQUIT

COPY src/image-provider/nginx.conf.template /nginx.conf.template

# Start nginx
CMD ["/bin/sh" , "-c" , "envsubst '$OTEL_COLLECTOR_HOST $IMAGE_PROVIDER_PORT $OTEL_COLLECTOR_PORT_GRPC $OTEL_SERVICE_NAME' < /nginx.conf.template > /etc/nginx/nginx.conf && cat  /etc/nginx/nginx.conf && exec nginx -g 'daemon off;'"]
```

## 🧪 Load Generator Service Dockerfile

```Dockerfile
FROM python:3.12-slim-bookworm AS base

FROM base AS builder
RUN apt-get -qq update \
    && apt-get install -y --no-install-recommends g++ \
    && rm -rf /var/lib/apt/lists/*

COPY ./src/load-generator/requirements.txt .
RUN pip install --prefix="/reqs" -r requirements.txt

FROM base
WORKDIR /usr/src/app/
COPY --from=builder /reqs /usr/local
COPY ./src/load-generator/locustfile.py .
COPY ./src/load-generator/people.json .
ENV LOCUST_PLAYWRIGHT=1
ENV PLAYWRIGHT_BROWSERS_PATH=/opt/pw-browsers
RUN playwright install --with-deps chromium
ENTRYPOINT ["locust", "--skip-log-setup"]
```

## 💳 Payment Service Dockerfile

```Dockerfile
FROM node:22-alpine AS build

WORKDIR /usr/src/app/

COPY ./src/payment/package*.json ./

RUN apk add --no-cache python3 make g++ && npm ci --omit=dev

# -----------------------------------------------------------------------------

FROM node:22-alpine

USER node
WORKDIR /usr/src/app/
ENV NODE_ENV=production

COPY --chown=node:node --from=build /usr/src/app/node_modules/ ./node_modules/
COPY ./src/payment/ ./
COPY ./pb/demo.proto ./

EXPOSE ${PAYMENT_PORT}
ENTRYPOINT [ "npm", "run", "start" ]
```

## 🎁 Product Catalog Service Dockerfile

```Dockerfile
FROM golang:1.22-alpine AS builder

WORKDIR /usr/src/app/

# Use Go build cache for dependencies
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    mkdir -p /root/.cache/go-build

# Copy
COPY go.mod go.sum ./

# Download
RUN go mod download

# Copy the rest of the source code
COPY . .

RUN go build -o product-catalog .

####################################

FROM alpine AS release

WORKDIR /usr/src/app/

COPY ./products/ ./products/
COPY --from=builder /usr/src/app/product-catalog/ ./

ENV PRODUCT_CATALOG_PORT 8088
ENTRYPOINT [ "./product-catalog" ]
```

## 🧾 Quota Service Dockerfile

```Dockerfile
FROM php:8.3-cli AS base

ADD https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions /usr/local/bin/
RUN chmod +x /usr/local/bin/install-php-extensions \
  && install-php-extensions \
    opcache \
    pcntl \
    protobuf \
    opentelemetry

WORKDIR /var/www
CMD ["php", "public/index.php"]
USER www-data
EXPOSE ${QUOTE_PORT}

FROM composer:2.7 AS vendor

WORKDIR /tmp/
COPY ./src/quote/composer.json .

RUN composer install \
    --ignore-platform-reqs \
    --no-interaction \
    --no-plugins \
    --no-scripts \
    --no-dev \
    --prefer-dist

FROM base AS final
COPY --from=vendor /tmp/vendor/ ./vendor/
COPY ./src/quote/ /var/www
```

## 📱 React-Native App Service Dockerfile

```Dockerfile
FROM reactnativecommunity/react-native-android:v13.2.1 AS builder

WORKDIR /reactnativesrc/
COPY . .

RUN npm install
WORKDIR android/
RUN chmod +x gradlew
RUN ./gradlew assembleRelease

FROM scratch
COPY --from=builder /reactnativesrc/android/app/build/outputs/apk/release/app-release.apk /reactnativeapp.apk
ENTRYPOINT ["/reactnativeapp.apk"]
```

## 🎯 Recommendation Service Dockerfile

```Dockerfile
FROM python:3.12-slim-bookworm AS base

WORKDIR /usr/src/app

COPY requirements.txt ./

RUN pip install --upgrade pip
RUN pip install -r requirements.txt

COPY . .

RUN opentelemetry-bootstrap -a install 

ENV RECOMMENDATION_PORT 1010

ENTRYPOINT ["python", "recommendation_server.py"]
```
## 🚚 Shipping Service Dockerfile

```Dockerfile
FROM --platform=${BUILDPLATFORM} rust:1.76 AS builder
ARG TARGETARCH
ARG TARGETPLATFORM
ARG BUILDPLATFORM

RUN echo Building on ${BUILDPLATFORM} for ${TARGETPLATFORM}

# Check if we are doing cross-compilation, if so we need to add in some more dependencies and run rustup
RUN if [ "${TARGETPLATFORM}" = "${BUILDPLATFORM}" ] ; then \
        apt-get update && apt-get install --no-install-recommends -y g++ libc6-dev libprotobuf-dev protobuf-compiler ca-certificates; \
    elif [ "${TARGETPLATFORM}" = "linux/arm64" ] ; then \
        apt-get update && apt-get install --no-install-recommends -y g++-aarch64-linux-gnu libc6-dev-arm64-cross libprotobuf-dev protobuf-compiler ca-certificates && \
        rustup target add aarch64-unknown-linux-gnu && \
        rustup toolchain install stable-aarch64-unknown-linux-gnu; \
    elif [ "${TARGETPLATFORM}" = "linux/amd64" ] ; then \
        apt-get update && apt-get install --no-install-recommends -y g++-x86-64-linux-gnu libc6-amd64-cross libprotobuf-dev protobuf-compiler ca-certificates && \
        rustup target add x86_64-unknown-linux-gnu && \
        rustup toolchain install stable-x86_64-unknown-linux-gnu; \
    else \
        echo "${TARGETPLATFORM} is not supported"; \
        exit 1; \
    fi

WORKDIR /app/

COPY /src/shipping/ /app/
COPY /pb/ /app/proto/

# Compile or crosscompile
RUN if [ "${TARGETPLATFORM}" = "${BUILDPLATFORM}" ] ; then \
        cargo build -r --features="dockerproto"; \
    elif [ "${TARGETPLATFORM}" = "linux/arm64" ] ; then \
        env CARGO_TARGET_AARCH64_UNKNOWN_LINUX_GNU_LINKER=aarch64-linux-gnu-gcc \
            CC_aarch64_unknown_linux_gnu=aarch64-linux-gnu-gcc \
            CXX_aarch64_unknown_linux_gnu=aarch64-linux-gnu-g++ \
        cargo build -r --features="dockerproto" --target aarch64-unknown-linux-gnu && \
        cp /app/target/aarch64-unknown-linux-gnu/release/shipping /app/target/release/shipping; \
    elif [ "${TARGETPLATFORM}" = "linux/amd64" ] ; then \
        env CARGO_TARGET_X86_64_UNKNOWN_LINUX_GNU_LINKER=x86_64-linux-gnu-gcc \
            CC_x86_64_unknown_linux_gnu=x86_64-linux-gnu-gcc \
            CXX_x86_64_unknown_linux_gnu=x86_64-linux-gnu-g++ \
        cargo build -r --features="dockerproto" --target x86_64-unknown-linux-gnu && \
        cp /app/target/x86_64-unknown-linux-gnu/release/shipping /app/target/release/shipping; \
    else \
        echo "${TARGETPLATFORM} is not supported"; \
        exit 1; \
    fi


ENV GRPC_HEALTH_PROBE_VERSION=v0.4.24
RUN wget -qO/bin/grpc_health_probe https://github.com/grpc-ecosystem/grpc-health-probe/releases/download/${GRPC_HEALTH_PROBE_VERSION}/grpc_health_probe-linux-${TARGETARCH} && \
    chmod +x /bin/grpc_health_probe

FROM debian:bookworm-slim AS release

WORKDIR /app
COPY --from=builder /app/target/release/shipping /app/shipping
COPY --from=builder /bin/grpc_health_probe /bin/grpc_health_probe

EXPOSE ${SHIPPING_PORT}
ENTRYPOINT ["/app/shipping"]
```







































