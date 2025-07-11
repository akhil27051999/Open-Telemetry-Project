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
### Accounting Service - Dockerfile Explanation

This Dockerfile is a multi-stage build for a .NET 8-based microservice called **Accounting**, designed for cross-platform containerization and observability using OpenTelemetry. The process begins with a `builder` stage that uses the official `.NET SDK 8.0` image and targets a specific platform using the `--platform=${BUILDPLATFORM}` flag. It defines build-time arguments such as `TARGETARCH` for architecture-specific builds (e.g., `amd64`, `arm64`) and sets the build configuration to `Release` by default. The service source code is copied into the container, including the `demo.proto` file for gRPC or protocol buffer definitions, and dependencies are restored with `dotnet restore`. The `dotnet build` command then compiles the project targeting the specified Linux architecture and outputs the intermediate build files to `/app/build`.

The second stage, named `publish`, uses the previously built artifacts to publish the final optimized output using `dotnet publish`. This command produces a trimmed deployment package inside `/app/publish`, with the `UseAppHost=false` flag to avoid platform-specific executable overhead, which keeps the image size smaller.

The final stage uses the lightweight `mcr.microsoft.com/dotnet/aspnet:8.0` base runtime image to run the service. It switches to a non-root user (`app`) for security best practices, sets the working directory to `/app`, and copies the published output from the previous stage. It then briefly switches to the root user to create and set ownership on a log directory (`/var/log/opentelemetry/dotnet`) and the instrumentation script (`instrument.sh`) so that the `app` user can access them. Finally, the container reverts to the non-root user and uses the `instrument.sh` script as its entrypoint, wrapping the service launch command `dotnet Accounting.dll`. This design ensures that the application is built efficiently, runs securely, and is ready for observability through OpenTelemetry instrumentation.

  
## 📢 Ad Service 

**This service determines appropriate ads to serve to users based on context keys. The ads will be for products available in the store.**

### Local Build

The Ad service requires at least JDK 17 to build and uses gradlew to
compile/install/distribute. Gradle wrapper is already part of the source code.
To build Ad Service, run:

```sh
./gradlew installDist

<or>

./gradlew installDist -PprotoSourceDir=./proto
```

It will create an executable script
`src/ad/build/install/oteldemo/bin/Ad`.

To run the Ad Service:

```sh
export AD_PORT=8080
export FEATURE_FLAG_GRPC_SERVICE_ADDR=featureflagservice:50053
./build/install/opentelemetry-demo-ad/bin/Ad
```

### Upgrading Gradle

If you need to upgrade the version of gradle then run

```sh
./gradlew wrapper --gradle-version <new-version>
```

### Dockerfile
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
### Ad Service - Dockerfile Explanation

This Dockerfile defines a multi-stage build process for a Java-based microservice (in this case, likely the **Ad** service) using the **Eclipse Temurin JDK and JRE 21** as the base images. The first stage, labeled `builder`, uses the full JDK image (`eclipse-temurin:21-jdk`) to compile and assemble the application. It sets the working directory to `/usr/src/app/` and copies essential Gradle build files like `gradlew`, `settings.gradle`, and `build.gradle`, along with the `.gradle` directory. It ensures the Gradle wrapper (`gradlew`) is executable, then runs initial Gradle commands including `./gradlew` to bootstrap the project and `./gradlew downloadRepos` to pre-download all required dependencies, which improves build performance and repeatability.

Next, the source code is copied into the container, including the protocol buffer files (`./pb`) mapped to a directory named `./proto`. After reapplying executable permissions to `gradlew`, it runs the command `./gradlew installDist -PprotoSourceDir=./proto`, which compiles the Java application, processes the protobuf files, and installs a distributable version of the application to the `build/install` directory using the `installDist` Gradle plugin. The use of `-PprotoSourceDir` suggests the app supports protocol buffer-based gRPC or messaging interfaces.

The final stage uses the lighter `eclipse-temurin:21-jre` runtime image to package and run the application without the overhead of the JDK. It sets the same working directory and copies all build artifacts from the `builder` stage. It also sets an environment variable `AD_PORT` with a default value of `9099`, which is likely the internal port the Ad service listens on. Finally, it defines the entrypoint using the shell script produced by the Gradle distribution install: `./build/install/opentelemetry-demo-ad/bin/Ad`. This script launches the service using the precompiled and properly structured Java application. This multi-stage approach ensures a clean separation between build-time and runtime dependencies, resulting in a smaller, production-ready image with all necessary application logic and protobuf integrations baked in.

## 🛒 Cart Service 

**This service maintains items placed in the shopping cart by users. It interacts with a Valkey caching service for fast access to shopping cart data.**

### Local Build
Run `dotnet restore` and `dotnet build`.

### Dockerfile
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

### Cart Service - Dockerfile Explanation

This Dockerfile is a multi-stage build setup designed for the **Cart** microservice, written in .NET 8 and optimized for lightweight Alpine-based deployment. The first stage, named `builder`, uses the full .NET SDK image (`mcr.microsoft.com/dotnet/sdk:8.0`) with platform awareness via the `--platform=$BUILDPLATFORM` directive. It accepts a build-time argument `TARGETARCH` to ensure architecture-specific builds (such as `amd64` or `arm64`). The working directory is set to `/usr/src/app/`, and source files from the `src/cart` directory and protobuf definitions from the `pb` directory are copied into the container.

The build process starts with `dotnet restore`, where the `cart.csproj` project file is restored with dependencies specific to the musl-libc Linux runtime (`linux-musl-$TARGETARCH`), suitable for Alpine-based images. Then, `dotnet publish` is used to compile and publish the application without restoring again (`--no-restore`) and outputs the final build artifacts into `/cart`.

The second and final stage uses the ultra-lightweight `mcr.microsoft.com/dotnet/runtime-deps:8.0-alpine3.20` image, which only includes the minimum dependencies needed to run a .NET app. This significantly reduces the image size and attack surface. It copies the published build from the previous stage into `/usr/src/app/`, sets an environment variable `DOTNET_HOSTBUILDER__RELOADCONFIGONCHANGE=false` to disable configuration reload-on-change for performance and stability in containers, exposes the service port `${CART_PORT}`, and sets the entrypoint to run the `cart` binary directly. This structure ensures a small, secure, and performance-optimized container ready for production deployment.

## 📦 Checkout Service

**This service is responsible to process a checkout order from the user. The checkout service will call many other services in order to process an order.**

### Local Build

To build the service binary, run:

```sh
go build -o /go/bin/checkout/
```

### Dockerfile

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

### Checkout Service - Dockerfile Explanation

This Dockerfile uses a multi-stage build process for the **Checkout** microservice, written in Go, optimized for small image size and fast runtime. The first stage is a `builder` stage that uses the official `golang:1.22-alpine` image. It sets the working directory to `/usr/src/app/` and performs module dependency management and binary compilation using advanced `RUN --mount` directives. These `--mount` flags are part of Docker BuildKit and are used here for efficient caching and binding of files during the build process.

The first `RUN` command mounts a cache for downloaded Go modules (`/go/pkg/mod`) and binds the `go.sum` and `go.mod` files from the `./src/checkout/` directory into the container, allowing `go mod download` to fetch the exact dependency versions specified. The second `RUN` command uses additional mounts to cache Go build artifacts and bind the entire application source directory (`./src/checkout`) for compilation. It compiles the application with `go build` and uses linker flags `-s -w` to strip debugging information and reduce the binary size. The final output is placed in `/go/bin/checkout/`.

The second stage uses a minimal `alpine` base image for the final runtime container. It sets the working directory, copies the statically compiled Go binary from the builder stage into the container, and exposes the port defined by the `CHECKOUT_PORT` environment variable. Finally, it defines the entrypoint as `./checkout`, launching the service when the container starts. This build approach ensures a compact, production-grade image with minimal overhead, faster startup, and enhanced security due to the absence of unnecessary build tools and dependencies.

## 💱 Currency Service 

**This service provides functionality to convert amounts between different currencies.**

### Dockerfile

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

### Currency Service - Dockerfile Explanation

This Dockerfile defines a multi-stage build and runtime process for the **Currency** microservice, implemented in **C++** and instrumented with **OpenTelemetry** for observability. The first stage, labeled `builder`, uses the `alpine:3.18` base image due to its minimal footprint. It installs all necessary build tools and dependencies such as `git`, `cmake`, `make`, `g++`, `grpc-dev`, `protobuf-dev`, and `linux-headers` using `apk`.

An argument `OPENTELEMETRY_CPP_VERSION` is passed to clone a specific version of the [OpenTelemetry C++ SDK](https://github.com/open-telemetry/opentelemetry-cpp). The SDK is cloned with `--depth 1` for shallow cloning to reduce image size and built using `cmake` with flags to enable OTLP (OpenTelemetry Protocol) over gRPC, disable tests and examples, and ensure optimal build configurations. The SDK is installed into the system to be used by the Currency service.

After installing the SDK, the Currency service source code (from `./src/currency`) and its protobuf definition (`demo.proto`) are copied into the image. The Currency service is then built using `cmake` and `make`, producing a binary that's installed into the `/usr/local` directory.

The second stage, named `release`, starts again from a clean `alpine:3.18` image and installs only the essential runtime dependencies (`grpc-dev` and `protobuf-dev`). It copies the compiled binaries and dependencies from the builder stage's `/usr/local` directory into the new runtime image.

The container exposes the port specified by the `CURRENCY_PORT` environment variable and sets the entrypoint to run the `currency` binary using a `sh -c` wrapper, passing in the port dynamically. This separation between build and runtime stages results in a secure, lean, and production-ready image for deploying the C++ Currency microservice with OpenTelemetry integration.


## 📧 Email Service 

**This service will send a confirmation email to the user when an order is placed.**

### Local Build

We use `bundler` to manage dependencies. To get started, simply `bundle install`.

### Running locally

You may run this service locally with `bundle exec ruby email_server.rb`.


### Dockerfile

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

### Email Service - Dockerfile Explanation

This Dockerfile builds and runs the **Email** microservice, written in **Ruby**, using a multi-stage Docker build to separate the dependency installation phase from the final runtime image. It starts from a slimmed-down `ruby:3.2.2-slim` base image for both the builder and release stages, ensuring a minimal and efficient container footprint.

In the `builder` stage, the working directory is set to `/tmp`, and the `Gemfile` and `Gemfile.lock` files—required for installing Ruby dependencies—are copied into the container. Instead of using Alpine's `apk` (commented out), it uses `apt-get` to install `build-essential`, which provides compilers and libraries required for native gem extensions. Then, `bundle install` is executed to install all dependencies into the default Ruby bundler path.

The `release` stage begins again from the same slim Ruby base image. The working directory is switched to `/email_server`, and the application code from `./src/email/` is copied over. The `Gemfile.lock` file is given write permissions (`chmod 666`) to avoid permission errors if regenerated. Finally, the installed gems from the builder stage are copied over from `/usr/local/bundle/`.

The container exposes the port defined by the `EMAIL_PORT` environment variable and uses `bundle exec ruby email_server.rb` as the entrypoint, which launches the Ruby-based Email microservice. This multi-stage setup ensures a clean separation of build tools and results in a lightweight, production-optimized runtime image.


## 🚩 Flagd-UI Service

### Local development

To run the app locally for development you must copy
`src/flagd/demo.flagd.json` into `src/flagd-ui/data/demo.flagd.json`
(create the directory and file if they do not exist yet). Make sure you're in the `src/flagd-ui` directory and run the following command:

```bash
npm run dev
```

Then you must navigate to `localhost:4000/feature`.

### Dockerfile

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

### Flagd UI Service - Dockerfile Explanation

This Dockerfile defines a multi-stage build for the **Flagd UI** service, a web frontend built using **Node.js** and **Next.js**. The goal is to create a lightweight, production-ready image by separating the build and runtime environments.

The first stage uses the official `node:20` image. It sets the working directory to `/app`, then copies `package.json` and `package-lock.json` from the `./src/flagd-ui/` directory to install the project's dependencies using `npm ci`. This ensures a clean and deterministic install based on the lockfile.

Next, the full project source is copied into the image, and the `npm run build` command compiles the Next.js frontend into production-ready static assets in the `.next/` directory.

The second stage uses the smaller `node:20-alpine` image for production runtime. It sets the same working directory and copies the `package*.json` files again to install only **production** dependencies using `npm ci --only=production`, reducing the image size and attack surface.

After that, it copies selected build artifacts from the `builder` stage:
- `instrumentation.ts` – likely used for OpenTelemetry instrumentation
- `next.config.mjs` – Next.js configuration file
- `.next/` – compiled static build assets

The container exposes port `4000` (used by Next.js apps in production), and the service is started using the command `npm start`, which typically runs `next start`.

This Dockerfile structure ensures efficient builds, clean separation of environments, and a lean production container for serving the Flagd UI.

## 🔍 Fraud Detection Service 

**This service analyses incoming orders and detects malicious customers. This is only mocked and received orders are printed out.**

### Local Build

To build the protos and the service binary, run from the repo root:

```sh
cp -r ../../pb/ src/main/proto/
./gradlew shadowJar
```

### Dockerfile

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
### Fraud Detection Service - Dockerfile Explanation

This Dockerfile sets up the **Fraud Detection** microservice, built using **Java** and **Gradle**, with integrated **OpenTelemetry Java agent** for observability. It uses a multi-stage build to separate the compilation from the final production runtime, resulting in a secure and minimal image.

The first stage uses the official `gradle:8-jdk17` image with JDK 17. It sets the working directory to `/usr/src/app/` and copies the service source code from `./src/fraud-detection/`. Additionally, the protobuf definition files from `./pb/` are placed into `./src/main/proto/`, following standard Java/Gradle structure for proto compilation.

Then, it runs `gradle shadowJar`, which compiles the project and packages it into a single executable JAR with all dependencies included (`fraud-detection-1.0-all.jar`). This is necessary for standalone execution in the minimal runtime environment.

The second stage uses the **Distroless Java 17** base image from Google (`distroless/java17-debian11`), which contains only the Java runtime and no package manager or shell, making it extremely secure and small in size.

It sets the working directory and copies the executable JAR from the builder stage. Then, it downloads and adds the **OpenTelemetry Java Agent** dynamically from the GitHub release corresponding to the provided `OTEL_JAVA_AGENT_VERSION` build argument. This agent allows for automatic instrumentation of the Java application for tracing and metrics.

The `JAVA_TOOL_OPTIONS` environment variable is set to enable the OpenTelemetry agent, and the container is launched using the command:
```bash
java -jar fraud-detection-1.0-all.jar
```

## 🌐 Frontend-Proxy Service

**The frontend proxy is used as a reverse proxy for user-facing web interfaces such as the frontend, Jaeger, Grafana, load generator, and feature flag service.**

###  Dockerfile

```Dockerfile
FROM envoyproxy/envoy:v1.32-latest
RUN apt-get update && apt-get install -y gettext-base && apt-get clean && rm -rf /var/lib/apt/lists/*

USER envoy
WORKDIR /home/envoy
COPY ./src/frontend-proxy/envoy.tmpl.yaml envoy.tmpl.yaml

ENTRYPOINT ["/bin/sh", "-c", "envsubst < envoy.tmpl.yaml > envoy.yaml && envoy -c envoy.yaml;"]
```

### Frontend-Proxy Service - Dockerfile Explaination

The Frontend Proxy service is a lightweight container built on top of `envoyproxy/envoy:v1.32-latest`. This service acts as a reverse proxy in front of the frontend and other exposed services. It leverages **Envoy** for service routing, observability, and telemetry injection. 

The Dockerfile starts by installing `gettext-base`, which provides the `envsubst` command used to substitute environment variables in the Envoy configuration template at runtime. It switches to the `envoy` user for better container security and sets the working directory to `/home/envoy`. The `envoy.tmpl.yaml` file is copied into the container, which serves as the configuration template.

The `ENTRYPOINT` command runs a shell script that uses `envsubst` to generate the actual `envoy.yaml` from the template using injected environment variables and then starts Envoy with the final configuration. This setup enables dynamic configuration based on the runtime environment and is useful for environments like staging, production, or development where values (like service hostnames or ports) may differ.

This container is designed to be lightweight, secure, and responsive to environment-specific configurations, making it an essential gateway for routing and monitoring traffic across microservices in the OpenTelemetry demo.

## 🖥️ Frontend Service 

**The frontend is responsible to provide a UI for users, as well as an API leveraged by the UI or other clients. The application is based on Next.JS to provide a React web-based UI and API routes.**

### Build Locally

By running `docker compose up` at the root of the project you'll have access to the
frontend client by going to <http://localhost:8080/>.

### Local development

Currently, the easiest way to run the frontend for local development is to execute

```shell
docker compose run --service-ports -e NODE_ENV=development --volume $(pwd)/src/frontend:/app --volume $(pwd)/pb:/app/pb --user node --entrypoint sh frontend
```

from the root folder.

It will start all of the required backend services
and within the container simply run `npm run dev`.
After that the app should be available at <http://localhost:8080/>.

### Dockerfile

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

### Frontend Service - Dockerfile Explaination

The Frontend service is a Node.js (Next.js-based) application designed to provide the user interface for the OpenTelemetry demo. The Dockerfile follows a multi-stage build pattern with three stages: `deps`, `builder`, and `runner`.

In the `deps` stage, the base image `node:20-alpine` is used for its small footprint. The `libc6-compat` package is installed for native compatibility with gRPC libraries, and the service dependencies are installed using `npm ci` to ensure clean, reproducible builds.

The `builder` stage also uses the Alpine-based Node image and installs `protobuf-dev` and `protoc`, which are required for compiling `.proto` files used by gRPC. The `pb` folder and frontend source code are copied, followed by gRPC stub generation (`npm run grpc:generate`) and the Next.js build (`npm run build`).

The `runner` stage is the production container. It sets up a non-root user (`nextjs`) for security and installs only the runtime dependencies (`protobuf-dev` and `protoc`). Configuration files (`next.config.js`, `Instrumentation.js`) and assets (public folder, package.json, and built `.next` directory) are copied in. The application runs under the `nextjs` user, exposing port `8080`, and is started using the `npm start` command.

This Dockerfile design ensures a secure, efficient, and portable frontend deployment, fully integrated with OpenTelemetry for observability and gRPC for inter-service communication.


## 🖼️ Image-provider Service
**This service provides the images which are used in the frontend. The images are statically hosted on a NGINX instance. The NGINX server is instrumented with the nginx-otel module.**

### Dockerfile

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

### Image Provider Service - Dockerfile Explaination

The Image Provider service is a lightweight NGINX-based static file server that integrates OpenTelemetry for observability. This Dockerfile is based on the `nginx:1.27.0-otel` image, which includes support for the OpenTelemetry NGINX module. In the first step, the system is updated and the `lsb-release` package is installed to help determine the Debian distribution codename for adding the correct NGINX package repository.

Next, the OpenTelemetry NGINX module (`nginx-module-otel`) is installed using the appropriate Debian repo. A directory `/static` is created and populated with static image assets from the `src/image-provider/static` directory, which the NGINX server will serve.

The container exposes the port defined by the `IMAGE_PROVIDER_PORT` environment variable, and `SIGQUIT` is specified as the stop signal for graceful shutdown.

The core of the configuration lies in using a templated `nginx.conf.template` file. At runtime, the `CMD` uses `envsubst` to replace variables such as `$OTEL_COLLECTOR_HOST`, `$IMAGE_PROVIDER_PORT`, `$OTEL_COLLECTOR_PORT_GRPC`, and `$OTEL_SERVICE_NAME` in the template file, dynamically generating a valid NGINX configuration. The final configuration is printed for debug visibility before launching NGINX in the foreground with `daemon off`.

This setup makes the image provider service observability-friendly, lightweight, and production-ready, ideal for serving static content with telemetry integration.

## 🧪 Load Generator Service

**The load generator is based on the Python load testing framework Locust. By default it will simulate users requesting several different routes from the frontend.**

### Dockerfile

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

### Load Generator Service Dockerfile Explained

This Dockerfile sets up a Load Generator service using Python, Locust, and Playwright to simulate user traffic for testing and observability purposes in a microservices environment.

The process begins with the `python:3.12-slim-bookworm` base image for a minimal and efficient Python environment. A multi-stage build is used to separate the dependency installation and final image creation for better performance and smaller size.

- Apt is updated quietly (`-qq`) to reduce output noise.
- The `g++` compiler is installed as it is required to build native Python dependencies.
- All Python dependencies listed in `requirements.txt` (typically including `locust`, `playwright`, and related plugins) are installed into a custom directory `/reqs` to avoid polluting the global environment.

- Sets the working directory to `/usr/src/app/`.
- Copies the pre-installed Python packages from the builder stage into the global Python path (`/usr/local`) of the final image.
- Copies essential test files like `locustfile.py` (Locust test script) and `people.json` (sample data) into the container.
- Sets environment variables:
  - `LOCUST_PLAYWRIGHT=1` to enable Locust-Playwright integration for browser-based testing.
  - `PLAYWRIGHT_BROWSERS_PATH=/opt/pw-browsers` to specify where Playwright should install its browser binaries.
- Installs Chromium browser (and necessary dependencies) required by Playwright.
- Uses `locust` as the entry point with `--skip-log-setup` to avoid double logging in container logs.

This setup ensures that the Load Generator is browser-capable, lightweight, and observability-ready for stress-testing modern distributed systems using real browser behavior and Locust's load simulation capabilities.


## 💳 Payment Service

**This service is responsible to process credit card payments for orders. It will return an error if the credit card is invalid or the payment cannot be processed.**

### Local Build

Copy the `demo.proto` file to this directory and run `npm ci`

### Dockerfile

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

### Payment Service Dockerfile Explaination

This Dockerfile is used to containerize the Payment microservice, which is built with Node.js and depends on protocol buffers (gRPC communication). It uses a two-stage build to ensure an optimized production-ready image.

The build process starts from the lightweight `node:22-alpine` image. It sets the working directory to `/usr/src/app/` and copies only the `package.json` and `package-lock.json` files from the `src/payment` directory. It installs required native build tools (`python3`, `make`, `g++`) to support the compilation of any native Node.js modules during dependency installation. Then, `npm ci --omit=dev` is run to install only production dependencies.

In the second stage, it uses the same `node:22-alpine` base but drops root privileges by switching to the `node` user for improved container security. It sets the same working directory and the `NODE_ENV=production` environment variable. The `node_modules/` folder is copied over from the build stage with correct ownership to avoid permission issues. Then, it copies the full application source from `src/payment/` and the shared protobuf definition file `demo.proto` for inter-service communication. The container exposes the configured `${PAYMENT_PORT}` and uses `npm run start` as the default command to launch the payment service.

This setup ensures the payment service is lightweight, production-ready, secure, and capable of communicating with other services in a gRPC-enabled microservices architecture.

## 🎁 Product Catalog Service

**This service is responsible to return information about products. The service can be used to get all products, search for specific products, or return details about any single product.**

## Local Build

To build the service binary, run:

```sh
export PRODUCT_CATALOG_PORT=<any-unique-port>
go build -o product-catalog . 
```
When this service is run the output should be similar to the following

```
INFO[0000] Loaded 10 products                           
INFO[0000] Product Catalog gRPC server started on port: 8088 
```

### Dockerfile

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

### Product Catalog Service Dockerfile Explained

This Dockerfile is designed to build and package the Product Catalog microservice written in Go. It uses a multi-stage build to keep the final image lightweight and optimized for production.

The first stage uses the `golang:1.22-alpine` image as the base for building the Go application. It sets the working directory to `/usr/src/app/`. To speed up builds and reduce redundant downloads, Go build and module caches are mounted using the `--mount=type=cache` flags. The module definition files `go.mod` and `go.sum` are copied in and used to download dependencies via `go mod download`. Then the rest of the source code is copied, and the Go compiler builds the binary with `go build -o product-catalog .`, producing an executable named `product-catalog`.

The final image uses a minimal `alpine` base to ensure a small attack surface and fast startup. It again sets the working directory to `/usr/src/app/` and copies the `products/` directory that contains the static product data required by the service. It also copies the built Go binary from the builder stage. The default `PRODUCT_CATALOG_PORT` is set to `8088` via environment variable, and the entrypoint runs the compiled binary directly.

This structure ensures the service is fast, secure, and lightweight, making it ideal for deployment in containerized microservices environments.


## 🧾 Quota Service

**This service is responsible for calculating shipping costs, based on the number of items to be shipped. The quote service is called from Shipping Service via HTTP.**

### Local Build

To build and run the quote service locally:

```sh
docker build src/quote --target base -t quote
cd src/quote
docker run --rm -it -v $(pwd):/var/www -e QUOTE_PORT=8999 -p "8999:8999" quote
```

Then, send some curl requests:

```sh
curl --location 'http://localhost:8999/getquote' \
--header 'Content-Type: application/json' \
--data '{"numberOfItems":3}'
```


### Dockerfile

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

### Quote Service Dockerfile Explained

This Dockerfile builds and packages the Quote microservice written in PHP using a multi-stage build. The service runs with PHP 8.3 CLI and includes OpenTelemetry support for observability.

The first stage starts from the official `php:8.3-cli` image and adds the PHP extensions required for this service. It uses the `docker-php-extension-installer` to install important extensions like `opcache` (for performance), `pcntl` (for process control), `protobuf` (for gRPC support), and `opentelemetry` (for tracing). The working directory is set to `/var/www`, and the default command runs the application via `php public/index.php`. It also changes the user to `www-data` for better security and exposes the port defined by the environment variable `QUOTE_PORT`.

The second stage uses the `composer:2.7` image to handle PHP dependencies. It copies the `composer.json` file from the quote service's source code and runs a `composer install` with optimized flags to install only the production dependencies (skipping dev packages, scripts, and plugins). The resulting vendor directory is staged for final use.

The final stage is based on the previously defined `base` image. It copies the `vendor/` directory from the vendor stage into `/var/www`, along with the application code from `./src/quote/`. This ensures a clean and production-ready PHP environment that includes all necessary code and dependencies, optimized for containerized deployment.


## 📱 React-Native App Service 

This was created using npx create-expo-app@latest. 
Content was taken from the web app example in src/frontend and modified to work in a React Native environment.

### Building the app Locally

Unlike the other components under src/ which run within containers this
app must be built and then run on a mobile simulator on your machine or a
physical device. If this is your first time running a React Native app then in
order to execute the steps under "Build on your host machine" you will need to
setup your local environment for Android or iOS development or both following
[this guide](https://reactnative.dev/docs/set-up-your-environment).
Alternatively for Android you can instead follow the steps under "Build within a
container" to leverage a container to build the app's apk for you.

### Build on your host machine

Before building the app you will need to install the dependencies for the app.

```bash
cd src/react-native-app
npm install
```

#### Android: Build and run app

To run on Android, the following command will compile the Android app and deploy
it to a running Android simulator or connected device. It will also start a
a server to provide the JS Bundle required by the app.

```bash
npm run android
```

#### iOS: Setup dependencies

Before building for iOS you will need to setup the iOS dependency management
using CocoaPods. This command only needs to be run the first time before
building the app for iOS.

```bash
cd ios && pod install && cd ..
```

#### iOS: Build and run with XCode

To run on iOS you may find it cleanest to build through the XCode IDE. In order
to start a server to provide the JS Bundle, run the following command (feel free
to ignore the output commands referring to opening an iOS simulator, we'll do
that directly through XCode in the next step).

```bash
npm run start
```

Then open XCode, open this as an existing project by opening
`src/react-native-app/ios/react-native-app.xcworkspace` then trigger the build
by hitting the Play button or from the menu using Product->Run.

#### iOS: Build and run from the command-line

You can build and run the app using the command line with the following
command. This will compile the iOS app and deploy it to a running iOS simulator
and start a server to provide the JS Bundle.

```bash
npm run ios
```


### Dockerfile

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

### React Native Android Build Dockerfile Explaination

This Dockerfile is designed to automate the building of a React Native Android application into an APK using a multi-stage build process. 

It starts from the `reactnativecommunity/react-native-android:v13.2.1` image, which provides all the necessary Android SDKs, NDKs, and build tools required for compiling a React Native project. The working directory is set to `/reactnativesrc/`, where the entire project is copied into the container. The build process begins by installing all required dependencies using `npm install`, followed by moving into the `android/` directory and giving execution permission to the `gradlew` wrapper script. The APK is built using the `./gradlew assembleRelease` command, which compiles the Android app in release mode and generates the APK under the `android/app/build/outputs/apk/release/` directory.

The final stage uses the minimalist `scratch` image to keep the resulting image as lightweight as possible. It copies the built APK file from the builder stage into the image and sets it as the container entrypoint. This allows the image to directly expose the compiled `app-release.apk` at the container level, which can then be extracted or used by downstream CI/CD workflows.


## 🤖 Recommendation Service 

**This service is responsible to get a list of recommended products for the user based on existing product IDs the user is browsing.**

### Local Build

To build the protos, run from the root directory:

```sh
make docker-generate-protobuf
```

### Dockerfile

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

###  Recommendation Service Dockerfile Explaination

This Dockerfile defines a containerized environment for the Recommendation microservice, which is written in Python and instrumented for observability with OpenTelemetry.

The build starts from the official `python:3.12-slim-bookworm` image to ensure a lightweight yet modern Python environment. It sets the working directory to `/usr/src/app` and begins by copying the `requirements.txt` file into the container. After upgrading `pip`, it installs all Python dependencies listed in the requirements file.

Once dependencies are installed, the rest of the application source code is copied into the container. It then runs `opentelemetry-bootstrap -a install`, which scans installed dependencies and installs the relevant OpenTelemetry instrumentation libraries automatically to enable tracing and metrics collection.

An environment variable `RECOMMENDATION_PORT` is defined (defaulting to 1010), which the application uses to determine which port to listen on. Finally, the container is configured to start by running `recommendation_server.py` as the entry point using Python.

## 🚚 Shipping Service 

**This service is responsible for providing shipping information including pricing and tracking information, when requested from Checkout Service.**

### Local Test

```sh
cargo test
```
### Dockerfile

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

### Shipping Service Dockerfile Explaination

This Dockerfile defines a robust multi-architecture build process for the `shipping` microservice written in Rust. The build starts from the official `rust:1.76` image and accepts build-time arguments such as `BUILDPLATFORM`, `TARGETPLATFORM`, and `TARGETARCH` to support cross-compilation. Depending on the target platform (e.g., `linux/arm64`, `linux/amd64`), it installs the necessary cross-compilers and toolchains using `apt-get` and `rustup`. The Dockerfile sets the working directory to `/app/`, copies in the Rust-based shipping source code and shared protobuf definitions, and then builds the project using `cargo build` with the `dockerproto` feature enabled. For cross-compilation, it sets the appropriate compiler environment variables before building and copies the resulting binary to a standard location.

To support health checking in Kubernetes and service meshes, the container also downloads and installs the `grpc_health_probe` binary corresponding to the target architecture. The final production image is based on `debian:bookworm-slim` and contains only the compiled `shipping` binary and the health probe tool. It sets `/app` as the working directory, exposes the `SHIPPING_PORT`, and uses the `shipping` binary as the container entry point. This clean and modular design ensures the service runs efficiently across different platforms while supporting gRPC health checks out of the box.







































