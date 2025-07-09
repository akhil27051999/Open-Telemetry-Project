# 🔭  OpenTelemetry Project

## OpenTelemetry Introduction

- In today’s fast-paced IT industry, practical experience is the key to mastering DevOps. This course is specially curated to provide hands-on experience in real-time DevOps implementation using a highly popular E-Commerce demo project open-sourced by OpenTelemetry. This project is widely recognized as one of the best real-world applications for learning DevOps.
- This project follows a multi-microservice architecture, where each microservice is developed in different programming languages. This will help us understand and tackle real-world challenges that arise when working with diverse tech stacks in a production environment.

## OpenTelemetry Architecture
OpenTelemetry Demo is composed of microservices written in different programming languages that talk to each other over gRPC and HTTP; and a load generator which uses Locust to fake user traffic.

![Screenshot 2025-07-07 163956](https://github.com/user-attachments/assets/6bff90b6-6cb2-45a6-afa2-45cb99d6b756)


## 📦 OpenTelemetry Microservices Overview

| **Service**         | **Language**        | **Description**                                                                                      |
|---------------------|---------------------|------------------------------------------------------------------------------------------------------|
| `accounting`        | .NET                | Processes incoming orders and counts the sum of all orders *(mock)*.                                |
| `ad`                | Java                | Provides text ads based on given context words.                                                     |
| `cart`              | .NET                | Stores items in the user’s shopping cart in Valkey and retrieves it.                                |
| `checkout`          | Go                  | Retrieves user cart, prepares order, and orchestrates payment, shipping, and email notification.     |
| `currency`          | C++                 | Converts money amounts between currencies using real-time data from the European Central Bank. **(Highest QPS)** |
| `email`             | Ruby                | Sends users an order confirmation email *(mock)*.                                                    |
| `fraud-detection`   | Kotlin              | Analyzes incoming orders to detect fraud attempts *(mock)*.                                          |
| `frontend`          | TypeScript          | Serves the website via HTTP. Generates session IDs; no sign-up/login needed.                        |
| `load-generator`    | Python / Locust     | Simulates user shopping flows by continuously sending requests to the frontend.                      |
| `payment`           | JavaScript          | Charges credit card info *(mock)* and returns a transaction ID.                                     |
| `product-catalog`   | Go                  | Lists, searches, and retrieves individual products from a JSON file.                                 |
| `quote`             | PHP                 | Calculates shipping cost based on number of items.                                                   |
| `recommendation`    | Python              | Suggests products based on current items in the cart.                                                |
| `shipping`          | Rust                | Estimates shipping costs and ships items to the given address *(mock)*.                             |
| `react-native-app`  | TypeScript          | React Native mobile app providing a UI for the shopping services.                                   |


### 1. Accounting Service
This service calculates the total amount of sold products. This is only mocked and received orders are printed out.

### 2. Ad Service
This service determines appropriate ads to serve to users based on context keys. The ads will be for products available in the store.

### 3. Cart Service
This service maintains items placed in the shopping cart by users. It interacts with a Valkey caching service for fast access to shopping cart data.

### 4. Checkout Service
This service is responsible to process a checkout order from the user. The checkout service will call many other services in order to process an order.

### 5. Currency Service
This service provides functionality to convert amounts between different currencies.

### 6. Email Service
This service will send a confirmation email to the user when an order is placed.

### 7. Fraud Detection Service
This service analyses incoming orders and detects malicious customers. This is only mocked and received orders are printed out.

### 8. Frontend Service
The frontend is responsible to provide a UI for users, as well as an API leveraged by the UI or other clients. The application is based on Next.JS to provide a React web-based UI and API routes.

### 9. Frontend Proxy Service
The frontend proxy is used as a reverse proxy for user-facing web interfaces such as the frontend, Jaeger, Grafana, load generator, and feature flag service.

### 10. Image Provider Service
This service provides the images which are used in the frontend. The images are statically hosted on a NGINX instance. The NGINX server is instrumented with the nginx-otel module.

### 11. Kafka Service
This is used as a message queue service to connect the checkout service with the accounting and fraud detection services.

### 12. Load Generator Service
The load generator is based on the Python load testing framework Locust. By default it will simulate users requesting several different routes from the frontend.

### 13. Payment Service
This service is responsible to process credit card payments for orders. It will return an error if the credit card is invalid or the payment cannot be processed.

### 14. Product Catalog Service
This service is responsible to return information about products. The service can be used to get all products, search for specific products, or return details about any single product.

### 15. Quota Service
- This service is responsible for calculating shipping costs, based on the number of items to be shipped. The quote service is called from Shipping Service via HTTP.
- The Quote Service is implemented using the Slim framework and php-di for managing the Dependency Injection.
- The PHP instrumentation may vary when using a different framework.

### 16. React Native App Service
The React Native app provides a mobile UI for users on Android and iOS devices to interact with the demo’s services. It is built with Expo and uses Expo’s file-based routing to layout the screens for the app.

### 17. Recommendation Service
This service is responsible to get a list of recommended products for the user based on existing product IDs the user is browsing.

### 18. Shipping Service
- This service is responsible for providing shipping information including pricing and tracking information, when requested from Checkout Service.
- Shipping service is built with Actix Web, Tracing for logs and OpenTelemetry Libraries. All other sub-dependencies are included in Cargo.toml.
- Depending on your framework and runtime, you may consider consulting Rust docs to supplement. You’ll find examples of async and sync spans in quote requests and tracking IDs respectively.

## 📚 Key Highlights

- **Polyglot Architecture**: Each service is written in a different programming language to simulate real-world environments.
- **Mock Integrations**: Several services are mocked to simulate external behavior (email, payment, fraud, etc.).
- **High Load Simulation**: A load generator is included to stress test and analyze system behavior.
- **Frontend + Mobile**: A web frontend and a mobile React Native application are both provided.

## Project Links

#### A detailed documentation along with architecture diagram of the project is shared in the below link.
```
https://opentelemetry.io/docs/demo/
```
#### Project architecture diagram and the explaination is available at the below link.

- **Architecture**
```
https://opentelemetry.io/docs/demo/architecture/
```

- **Overview of microservices used in the project**
```
https://opentelemetry.io/docs/demo/services/
```



