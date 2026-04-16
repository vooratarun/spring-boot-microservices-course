# Spring Boot Microservices - Complete Guide

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Microservices](#microservices)
4. [Technology Stack](#technology-stack)
5. [Getting Started](#getting-started)
6. [API Documentation](#api-documentation)
7. [Running the Services](#running-the-services)

---

## Project Overview

This is a comprehensive **Spring Boot Microservices Course** project that demonstrates building a complete BookStore application using modern microservices architecture. The application allows customers to browse a catalog of books, place orders, and receive notifications.

### Key Features
- **Product Catalog Management** - Browse and manage books
- **Order Management** - Create and track orders
- **Event-Driven Architecture** - Async communication via RabbitMQ
- **API Gateway** - Single entry point for all services
- **OAuth2 Security** - Secure endpoints using Keycloak
- **Real-time Notifications** - Email notifications for order events
- **Observability** - Monitoring via Prometheus, Grafana, Loki, and Tempo
- **Containerized** - Docker and Docker Compose support

---

## Architecture

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     CLIENT / WEB BROWSER                        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Bookstore      │
                    │  WebApp         │
                    │  (Thymeleaf +   │
                    │   Bootstrap)    │
                    └────────┬────────┘
                             │ (via JavaScript/API calls)
                    ┌────────▼────────────────────────┐
                    │     API Gateway                 │
                    │  (Spring Cloud Gateway)         │
                    │                                 │
                    │  - Route requests               │
                    │  - Aggregate docs               │
                    │  - Rate limiting                │
                    │  - Security                     │
                    └────────┬────────────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐      ┌──────▼──────┐      ┌──────▼──────┐
   │ Catalog  │      │   Order     │      │    Auth     │
   │ Service  │      │   Service   │      │  (Keycloak) │
   │          │      │             │      │             │
   │ - Books  │      │ - Orders    │      │ - OAuth2    │
   │ - Search │      │ - Events    │      │ - JWT       │
   │          │      │             │      │             │
   └────┬─────┘      └──────┬──────┘      └─────────────┘
        │                   │ (Event Publisher)
        └────────────┬──────┘
                     │ (RabbitMQ)
        ┌────────────▼────────────┐
        │  Notification Service   │
        │                         │
        │  - Listen to events     │
        │  - Send emails          │
        │  - Order notifications  │
        └────────────┬────────────┘
                     │ (Email)
                  [Email]


┌─────────────────────────────────────────────────────────┐
│              Data & Infrastructure                       │
├─────────────────────────────────────────────────────────┤
│  PostgreSQL  │  RabbitMQ  │  Redis  │  Prometheus      │
│  (Database)  │ (Messaging)│ (Cache) │ (Metrics)        │
└─────────────────────────────────────────────────────────┘
```

---

## Microservices

### 1. Catalog Service

**Purpose**: Manages the product catalog (books) for the bookstore.

**Package**: `com.sivalabs.bookstore.catalog`

**Main Class**: `CatalogServiceApplication`

**Port**: 8080 (Default)

**Key Components**:
- **ProductController**: REST endpoints for product management
- **ProductService**: Business logic for product operations
- **ProductRepository**: Database access layer
- **ProductEntity**: JPA entity for products

**API Endpoints**:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/products?page=1` | Get paginated list of products |
| GET | `/api/products/{code}` | Get product by code |

**Tech Stack**:
- Spring Boot 3.5.3
- Spring Data JPA
- PostgreSQL
- Flyway DB Migrations
- OpenAPI/Swagger Documentation

**Database**:
- PostgreSQL with Flyway migrations

**Features**:
- Paginated product listing
- Product search by code
- REST API with OpenAPI documentation
- Actuator endpoints for health checks

---

### 2. Order Service

**Purpose**: Handles order creation, management, and publishing order events.

**Package**: `com.sivalabs.bookstore.orders`

**Main Class**: `OrderServiceApplication`

**Port**: 8081 (Default)

**Key Components**:
- **OrderController**: REST endpoints for order management
- **OrderService**: Business logic for orders
- **OrderEventPublisher**: Publishes events to message broker
- **OrderEventService**: Processes and stores order events
- **SecurityService**: Handles OAuth2 security
- **OrderValidator**: Validates orders against catalog service

**API Endpoints**:

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| POST | `/api/orders` | Create new order | ✓ OAuth2 |
| GET | `/api/orders` | Get user's orders | ✓ OAuth2 |
| GET | `/api/orders/{orderNumber}` | Get specific order | ✓ OAuth2 |

**Tech Stack**:
- Spring Boot 3.5.3
- Spring Security + OAuth2 (Keycloak)
- Spring Data JPA
- RabbitMQ (AMQP)
- PostgreSQL
- Resilience4j (Circuit breaker pattern)
- ShedLock (Distributed job locking)
- RestClient for service calls

**Key Features**:
- Order creation with validation
- Event-driven architecture (publishes order events)
- OAuth2 security with Keycloak
- Resilience4j for fault tolerance
- ShedLock for scheduled task coordination
- Event sourcing pattern
- Calls Catalog Service to validate products

**Event Types Published**:
- `OrderCreatedEvent`
- `OrderDeliveredEvent`
- `OrderCancelledEvent`
- `OrderErrorEvent`

---

### 3. Notification Service

**Purpose**: Listens to order events and sends notifications to customers.

**Package**: `com.sivalabs.bookstore.notifications`

**Main Class**: `NotificationServiceApplication`

**Port**: 8082 (Default)

**Key Components**:
- **NotificationService**: Handles sending notifications
- **Event Listeners**: Listen to RabbitMQ messages
- **EmailSender**: Sends email notifications

**Supported Events**:
- Order Created
- Order Delivered
- Order Cancelled
- Order Processing Errors

**Tech Stack**:
- Spring Boot 3.5.3
- Spring AMQP (RabbitMQ)
- Spring Mail
- PostgreSQL
- Flyway DB Migrations

**Key Features**:
- Asynchronous event processing
- Email notifications (extensible for SMS, etc.)
- Message-driven POJO pattern
- Event logging and tracking
- Error handling and retry logic

**Configuration**:
- Configurable support email address
- Queue/Exchange configuration via properties

---

### 4. API Gateway

**Purpose**: Single entry point for all microservices. Routes requests, aggregates documentation, and handles cross-cutting concerns.

**Package**: `com.sivalabs.bookstore.gateway`

**Main Class**: `ApiGatewayApplication`

**Port**: 8080 (Default - acts as main entry point)

**Key Components**:
- Spring Cloud Gateway routing
- OpenAPI aggregation
- Request/response filtering
- Rate limiting configuration

**Tech Stack**:
- Spring Boot 3.5.3
- Spring Cloud Gateway
- Spring Cloud 2025.0.0
- WebFlux (Reactive)
- OpenAPI/Swagger aggregation

**Routing Configuration**:
- Routes `/catalog/**` → Catalog Service
- Routes `/orders/**` → Order Service
- Aggregates OpenAPI documentation

**Key Features**:
- Request routing and load balancing
- Cross-origin handling
- Request/response filtering
- Centralized API documentation
- Health checks
- Metrics collection

---

### 5. Bookstore WebApp

**Purpose**: Customer-facing web application for browsing products, placing orders, and tracking order status.

**Package**: `com.sivalabs.bookstore.webapp`

**Main Class**: `BookstoreWebappApplication`

**Port**: 8083 (Default)

**Key Components**:
- **ProductController**: Browse products
- **OrderController**: Order management UI
- **CatalogServiceClient**: HTTP client for catalog service
- **OrderServiceClient**: HTTP client for order service
- **SecurityHelper**: OAuth2 security handling

**Pages**:
- Product Listing (`/products`) - Browse books
- Shopping Cart (`/cart`) - Manage cart
- Orders (`/orders`) - View order history
- Order Details (`/orders/{orderNumber}`) - View specific order

**Tech Stack**:
- Spring Boot 3.5.3
- Spring Security + OAuth2 (Keycloak)
- Thymeleaf (Server-side rendering)
- Alpine.js (Lightweight JavaScript)
- Bootstrap (CSS framework)
- Declarative HTTP Interface (RestClient)

**Key Features**:
- Server-side rendering with Thymeleaf
- OAuth2 authentication
- Real-time product browsing
- Order creation and tracking
- Responsive design (Bootstrap)
- Interactive UI (Alpine.js)
- Integration with Order and Catalog services

**Service Clients**:
- Communicates with API Gateway for orders
- Communicates with API Gateway for catalog

---

## Technology Stack

### Core Framework
- **Java 21** - Latest LTS version
- **Spring Boot 3.5.3** - Latest stable version
- **Maven** - Build tool

### Data Access
- **Spring Data JPA** - ORM framework
- **PostgreSQL** - Relational database
- **Flyway** - Database migration tool

### Messaging & Events
- **RabbitMQ** - Message broker
- **Spring AMQP** - AMQP support

### Security
- **Spring Security** - Authentication & Authorization
- **OAuth2 Resource Server** - Token validation
- **Keycloak** - Identity provider

### Service Communication
- **Spring Cloud Gateway** - API Gateway
- **RestClient** - Declarative HTTP interfaces
- **Resilience4j** - Resilience patterns (Circuit breaker, retry)

### Job Scheduling
- **ShedLock** - Distributed job coordination
- **Spring Scheduler** - Task scheduling

### API Documentation
- **SpringDoc OpenAPI** - OpenAPI/Swagger generation
- **Swagger UI** - Interactive API documentation

### Frontend
- **Thymeleaf** - Server-side templating
- **Bootstrap** - CSS framework
- **Alpine.js** - Lightweight JavaScript framework

### Monitoring & Observability
- **Spring Boot Actuator** - Metrics and health endpoints
- **Micrometer** - Metrics collection
- **Prometheus** - Time-series database
- **Grafana** - Visualization
- **Loki** - Log aggregation
- **Tempo** - Distributed tracing
- **OpenTelemetry** - Observability framework

### Testing
- **JUnit 5** - Testing framework
- **Spring Boot Test** - Spring testing utilities
- **RestAssured** - REST API testing
- **Testcontainers** - Containerized dependencies for testing
- **WireMock** - HTTP mocking
- **Awaitility** - Async testing
- **Instancio** - Test data generation

### Development Tools
- **Spring Boot DevTools** - Live reload
- **Git Commit ID Plugin** - Git information in builds
- **Spotless** - Code formatting

---

## Getting Started

### Prerequisites
- **Java 21** (Install via [SDKMAN](https://sdkman.io/))
- **Docker Desktop** - For containerized services
- **Maven** - Included with project (mvnw)
- **Git** - For version control

### Quick Start

1. **Clone the repository**
```bash
git clone https://github.com/sivalabs/spring-boot-microservices-course.git
cd spring-boot-microservices-course
```

2. **Set up infrastructure with Docker Compose**
```bash
cd deployment/docker-compose
docker-compose -f infra.yml -f apps.yml up
```

3. **Build all services**
```bash
./mvnw clean install
```

4. **Access the services**

| Service | URL |
|---------|-----|
| Bookstore WebApp | http://localhost:8083 |
| API Gateway | http://localhost:8080 |
| Catalog Service | http://localhost:8081/api/products |
| Order Service | http://localhost:8082/api/orders |
| Keycloak | http://localhost:8180 |
| RabbitMQ Console | http://localhost:15672 |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |

### Environment Setup

**Required Environment Variables**:
- `SPRING_DATASOURCE_URL` - PostgreSQL connection
- `SPRING_DATASOURCE_USERNAME` - Database user
- `SPRING_DATASOURCE_PASSWORD` - Database password
- `SPRING_RABBITMQ_HOST` - RabbitMQ hostname
- `SPRING_RABBITMQ_PORT` - RabbitMQ port

**Default Keycloak Credentials**:
- Username: `admin`
- Password: `admin`

---

## API Documentation

### Accessing Swagger Documentation

Each service provides OpenAPI/Swagger documentation:

- **Catalog Service**: http://localhost:8081/swagger-ui.html
- **Order Service**: http://localhost:8082/swagger-ui.html
- **API Gateway**: http://localhost:8080/swagger-ui.html

### Aggregated API Documentation

The API Gateway aggregates documentation from all services at:
```
http://localhost:8080/swagger-ui.html
```

### Example API Calls

**Get Products**
```bash
curl -X GET "http://localhost:8080/catalog/api/products?page=1"
```

**Create Order** (Requires authentication)
```bash
curl -X POST "http://localhost:8080/orders/api/orders" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "customer": {
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "1234567890"
    },
    "deliveryAddress": {
      "addressLine1": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipCode": "10001",
      "country": "USA"
    },
    "items": [
      {
        "code": "P001",
        "name": "Spring in Action",
        "price": 49.99,
        "quantity": 1
      }
    ]
  }'
```

**Get Orders**
```bash
curl -X GET "http://localhost:8080/orders/api/orders" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

---

## Running the Services

### Development Mode

**Using IDE**:
1. Open project in IntelliJ IDEA or your favorite IDE
2. Run each service's main class individually

**Using Command Line**:
```bash
# Catalog Service
cd catalog-service
./mvnw spring-boot:run

# In another terminal - Order Service
cd order-service
./mvnw spring-boot:run

# In another terminal - Notification Service
cd notification-service
./mvnw spring-boot:run

# In another terminal - API Gateway
cd api-gateway
./mvnw spring-boot:run

# In another terminal - WebApp
cd bookstore-webapp
./mvnw spring-boot:run
```

### Production Mode

**Using Docker Compose**:
```bash
cd deployment/docker-compose
docker-compose -f infra.yml -f apps.yml -f monitoring.yml up -d
```

**Using Docker individually**:
```bash
# Build individual services
./mvnw spring-boot:build-image -pl catalog-service

# Run as container
docker run -p 8080:8080 \
  --network bookstore-network \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/catalog \
  sivaprasadreddy/bookstore-catalog-service:0.0.1-SNAPSHOT
```

### Running Tests

```bash
# Run all tests
./mvnw clean test

# Run tests for specific service
./mvnw test -pl catalog-service

# Run with coverage
./mvnw clean test jacoco:report
```

---

## Additional Resources

### Learning Objectives Covered
- Building Spring Boot REST APIs
- Database persistence with Spring Data JPA and Postgres
- Event-driven async communication with RabbitMQ
- OAuth2 security with Spring Security and Keycloak
- API Gateway implementation with Spring Cloud Gateway
- Resilience patterns with Resilience4j
- Distributed job scheduling with ShedLock
- HTTP interfaces with RestClient
- API documentation aggregation
- Containerization with Docker
- Testing with JUnit 5, RestAssured, Testcontainers
- Web applications with Thymeleaf and Alpine.js
- Monitoring with Grafana, Prometheus, Loki, Tempo

### Useful Links
- [SivaLabs Blog](https://sivalabs.in)
- [Spring Boot Tutorials](https://www.sivalabs.in/spring-boot-tutorials/)
- [YouTube Course Playlist](https://www.youtube.com/playlist?list=PLuNxlOYbv61g_ytin-wgkecfWDKVCEDmB)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)
- [Keycloak Documentation](https://www.keycloak.org/documentation)

---

**Last Updated**: April 2026  
**Version**: 1.0  
**Course Level**: Advanced

