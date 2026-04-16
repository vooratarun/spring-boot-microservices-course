# Order Service Documentation

## Overview

The **Order Service** is a critical microservice responsible for order management, event publishing, and inter-service communication. It implements OAuth2 security, resilience patterns, and event-driven architecture for asynchronous communication with other services.

## Service Details

| Property | Value |
|----------|-------|
| **Module** | `order-service` |
| **Port** | `8081` (default) |
| **Database** | PostgreSQL |
| **Message Broker** | RabbitMQ |
| **Auth Provider** | Keycloak (OAuth2) |
| **Package** | `com.sivalabs.bookstore.orders` |
| **Main Class** | `OrderServiceApplication` |

## Architecture

### Order Service Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    REST Controller Layer                   │
│                   (OrderController)                        │
│              Handles HTTP Requests/Responses               │
├────────────────────────────────────────────────────────────┤
│                   Service Layer                            │
│     ┌──────────────────┐        ┌──────────────────┐      │
│     │   OrderService   │        │ OrderEventPubl.  │      │
│     │  - Create Orders │        │  - Event Publish │      │
│     │  - Query Orders  │        │                  │      │
│     └──────────────────┘        └──────────────────┘      │
├────────────────────────────────────────────────────────────┤
│                Validation & Security Layer                 │
│     ┌──────────────────┐        ┌──────────────────┐      │
│     │  OrderValidator  │        │ SecurityService  │      │
│     │  - Catalog Check │        │  - User Context  │      │
│     └──────────────────┘        └──────────────────┘      │
├────────────────────────────────────────────────────────────┤
│              Repository / Data Access Layer                │
│     ┌──────────────────┐        ┌──────────────────┐      │
│     │  OrderRepository │        │ OrderEventRepo.  │      │
│     │   - JPA Access   │        │  - Event Storage │      │
│     └──────────────────┘        └──────────────────┘      │
├────────────────────────────────────────────────────────────┤
│         External Service Integration                       │
│     ┌──────────────────┐        ┌──────────────────┐      │
│     │  Catalog Service │        │  Message Broker  │      │
│     │  (RestClient)    │        │   (RabbitMQ)     │      │
│     └──────────────────┘        └──────────────────┘      │
├────────────────────────────────────────────────────────────┤
│         Database & Infrastructure                         │
│          PostgreSQL    │    RabbitMQ    │  Keycloak       │
└────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. OrderServiceApplication
**File**: `src/main/java/com/sivalabs/bookstore/orders/OrderServiceApplication.java`

Entry point of the Order Service.

### 2. OrderController
**File**: `src/main/java/com/sivalabs/bookstore/orders/web/controllers/OrderController.java`

REST controller handling order-related HTTP requests.

**Endpoints**:

| Method | Path | Description | Auth Required | Status Code |
|--------|------|-------------|---------------|------------|
| `POST` | `/api/orders` | Create new order | ✓ OAuth2 | 201 Created |
| `GET` | `/api/orders` | Get user's orders | ✓ OAuth2 | 200 OK |
| `GET` | `/api/orders/{orderNumber}` | Get order details | ✓ OAuth2 | 200 OK |

**Controller Implementation**:
```java
@RestController
@RequestMapping("/api/orders")
@SecurityRequirement(name = "security_auth")
class OrderController {
    private final OrderService orderService;
    private final SecurityService securityService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    CreateOrderResponse createOrder(@Valid @RequestBody CreateOrderRequest request) {
        String userName = securityService.getLoginUserName();
        return orderService.createOrder(userName, request);
    }

    @GetMapping
    List<OrderSummary> getOrders() {
        String userName = securityService.getLoginUserName();
        return orderService.findOrders(userName);
    }

    @GetMapping(value = "/{orderNumber}")
    OrderDTO getOrder(@PathVariable(value = "orderNumber") String orderNumber) {
        String userName = securityService.getLoginUserName();
        return orderService
                .findUserOrder(userName, orderNumber)
                .orElseThrow(() -> new OrderNotFoundException(orderNumber));
    }
}
```

### 3. OrderService
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/OrderService.java`

Business logic layer for order operations.

**Key Methods**:
- `createOrder(String userName, CreateOrderRequest request)` - Create new order
- `findOrders(String userName)` - Get user's orders
- `findUserOrder(String userName, String orderNumber)` - Get specific order
- `findOrder(String orderNumber)` - Get order by number

**Order Creation Flow**:
1. Validate input
2. Extract user info from security context
3. Validate items against Catalog Service
4. Create OrderEntity in database
5. Publish OrderCreatedEvent to RabbitMQ

### 4. OrderEntity
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/OrderEntity.java`

JPA entity representing an order in the database.

**Fields**:
- `id` - Primary key
- `orderNumber` - Unique order identifier
- `userName` - Customer username
- `customerName` - Customer full name
- `customerEmail` - Customer email
- `customerPhone` - Customer phone
- `deliveryAddress` - Address entity
- `items` - List of order items
- `status` - Order status (PENDING, CONFIRMED, DELIVERED, CANCELLED)
- `createdAt` - Creation timestamp
- `updatedAt` - Last update timestamp

### 5. OrderItemEntity
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/OrderItemEntity.java`

JPA entity representing an individual order item.

**Fields**:
- `id` - Primary key
- `order` - Reference to parent order
- `code` - Product code
- `name` - Product name
- `price` - Price at purchase time
- `quantity` - Quantity ordered

### 6. OrderEventPublisher
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/OrderEventPublisher.java`

Publishes order events to RabbitMQ.

**Supported Events**:
- `OrderCreatedEvent` - Published when order is created
- `OrderDeliveredEvent` - Published when order is delivered
- `OrderCancelledEvent` - Published when order is cancelled
- `OrderErrorEvent` - Published when order processing fails

### 7. OrderValidator
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/OrderValidator.java`

Validates orders before processing.

**Validations**:
- Validate customer information
- Check product availability via Catalog Service
- Validate order items
- Check inventory (if applicable)

### 8. SecurityService
**File**: `src/main/java/com/sivalabs/bookstore/orders/domain/SecurityService.java`

Handles security-related operations.

**Methods**:
- `getLoginUserName()` - Get current user's username
- `getLoginUserEmail()` - Get current user's email

### 9. ApplicationProperties
**File**: `src/main/java/com/sivalabs/bookstore/orders/ApplicationProperties.java`

Configuration properties for the service.

```java
@ConfigurationProperties(prefix = "orders")
public record ApplicationProperties(
        String catalogServiceUrl,
        String orderEventsExchange,
        String newOrdersQueue,
        String deliveredOrdersQueue,
        String cancelledOrdersQueue,
        String errorOrdersQueue) {}
```

## API Requests & Responses

### Create Order

**Request**:
```http
POST /api/orders
Content-Type: application/json
Authorization: Bearer <JWT_TOKEN>

{
  "customer": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "1234567890"
  },
  "deliveryAddress": {
    "addressLine1": "123 Main Street",
    "addressLine2": "Apt 4B",
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
      "quantity": 2
    },
    {
      "code": "P002",
      "name": "Microservices Patterns",
      "price": 59.99,
      "quantity": 1
    }
  ]
}
```

**Response (201 Created)**:
```json
{
  "orderNumber": "ORD-20240115-00001",
  "status": "PENDING",
  "message": "Order created successfully"
}
```

**Response (400 Bad Request)**:
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Invalid order: Customer email is required",
  "errors": [
    {
      "field": "customer.email",
      "message": "must not be blank"
    }
  ]
}
```

### Get Orders

**Request**:
```http
GET /api/orders
Authorization: Bearer <JWT_TOKEN>
```

**Response (200 OK)**:
```json
[
  {
    "orderNumber": "ORD-20240115-00001",
    "customerName": "John Doe",
    "status": "PENDING",
    "totalPrice": 159.97,
    "createdAt": "2024-01-15T10:30:00Z"
  },
  {
    "orderNumber": "ORD-20240114-00001",
    "customerName": "John Doe",
    "status": "DELIVERED",
    "totalPrice": 49.99,
    "createdAt": "2024-01-14T15:45:00Z"
  }
]
```

### Get Order Details

**Request**:
```http
GET /api/orders/ORD-20240115-00001
Authorization: Bearer <JWT_TOKEN>
```

**Response (200 OK)**:
```json
{
  "orderNumber": "ORD-20240115-00001",
  "customerName": "John Doe",
  "customerEmail": "john@example.com",
  "customerPhone": "1234567890",
  "status": "PENDING",
  "deliveryAddress": {
    "addressLine1": "123 Main Street",
    "addressLine2": "Apt 4B",
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
      "quantity": 2
    },
    {
      "code": "P002",
      "name": "Microservices Patterns",
      "price": 59.99,
      "quantity": 1
    }
  ],
  "totalPrice": 159.97,
  "createdAt": "2024-01-15T10:30:00Z"
}
```

## Database Schema

### Orders Table

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(100) NOT NULL UNIQUE,
    user_name VARCHAR(255) NOT NULL,
    customer_name VARCHAR(255) NOT NULL,
    customer_email VARCHAR(255) NOT NULL,
    customer_phone VARCHAR(20),
    delivery_address_id BIGINT,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (delivery_address_id) REFERENCES addresses(id)
);
```

### Order Items Table

```sql
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    product_code VARCHAR(100) NOT NULL,
    product_name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    quantity INTEGER NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

### Order Events Table

```sql
CREATE TABLE order_events (
    id SERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

## Security Configuration

### OAuth2 Configuration

**File**: `src/main/java/com/sivalabs/bookstore/orders/config/SecurityConfig.java`

Configures Spring Security with OAuth2 resource server.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8180/realms/bookstore
          jwk-set-uri: http://localhost:8180/realms/bookstore/protocol/openid-connect/certs
```

### JWT Token Validation

- Validates JWT tokens from Keycloak
- Extracts user principal from JWT claims
- All endpoints require valid token (except public endpoints)

## Event System

### Event Types

The Order Service publishes events to RabbitMQ for other services to consume:

**1. OrderCreatedEvent**
```json
{
  "eventType": "order.created",
  "orderNumber": "ORD-20240115-00001",
  "customer": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "totalPrice": 159.97,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**2. OrderDeliveredEvent**
```json
{
  "eventType": "order.delivered",
  "orderNumber": "ORD-20240115-00001",
  "customer": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**3. OrderCancelledEvent**
```json
{
  "eventType": "order.cancelled",
  "orderNumber": "ORD-20240115-00001",
  "reason": "Customer requested cancellation",
  "customer": {
    "name": "John Doe",
    "email": "john@example.com"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**4. OrderErrorEvent**
```json
{
  "eventType": "order.error",
  "orderNumber": "ORD-20240115-00001",
  "reason": "Payment processing failed",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### RabbitMQ Configuration

**File**: `src/main/java/com/sivalabs/bookstore/orders/config/RabbitMQConfig.java`

Defines exchanges, queues, and bindings for order events.

**Configuration**:
```yaml
orders:
  orderEventsExchange: order-events-exchange
  newOrdersQueue: new-orders-queue
  deliveredOrdersQueue: delivered-orders-queue
  cancelledOrdersQueue: cancelled-orders-queue
  errorOrdersQueue: error-orders-queue
```

## Resilience Patterns

### Circuit Breaker (Resilience4j)

Protects against cascading failures when calling Catalog Service.

**Configuration**:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      catalogService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 3
```

### Retry Logic

Automatic retries for failed requests.

```yaml
resilience4j:
  retry:
    instances:
      catalogService:
        maxAttempts: 3
        waitDuration: 1000
```

## Distributed Job Scheduling

### ShedLock Integration

Ensures scheduled tasks run only once in a distributed environment.

**Configuration**:
```java
@EnableScheduling
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class SchedulerConfig {
    // Configuration for ShedLock
}
```

## Configuration

### Application Properties

**File**: `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest

server:
  port: 8081

orders:
  catalogServiceUrl: http://localhost:8080
  orderEventsExchange: order-events-exchange
  newOrdersQueue: new-orders-queue
  deliveredOrdersQueue: delivered-orders-queue
  cancelledOrdersQueue: cancelled-orders-queue
  errorOrdersQueue: error-orders-queue
```

## Dependencies

### Core Dependencies

```xml
<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Security OAuth2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<!-- RabbitMQ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<!-- Resilience4j -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.3.0</version>
</dependency>

<!-- ShedLock -->
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
    <version>6.9.2</version>
</dependency>

<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

## Testing

### Integration Tests

Tests are located in: `src/test/java/com/sivalabs/bookstore/orders/`

**Example Test**:
```java
@Sql("/test-orders.sql")
class OrderControllerTests extends AbstractIT {

    @Test
    void shouldCreateOrderSuccessfully() {
        mockGetProductByCode("P100", "Product 1", new BigDecimal("25.50"));
        
        var payload = """
            {
                "customer" : {
                    "name": "Siva",
                    "email": "siva@gmail.com",
                    "phone": "999999999"
                },
                "deliveryAddress" : { ... },
                "items": [ ... ]
            }
        """;

        given().contentType(ContentType.JSON)
                .header("Authorization", "Bearer " + getToken())
                .body(payload)
                .when()
                .post("/api/orders")
                .then()
                .statusCode(HttpStatus.CREATED.value())
                .body("orderNumber", notNullValue());
    }
}
```

### Running Tests

```bash
# Run all tests
./mvnw test -pl order-service

# Run specific test class
./mvnw test -pl order-service -Dtest=OrderControllerTests

# Run with coverage
./mvnw clean test jacoco:report -pl order-service
```

## Running the Service

### Development

```bash
cd order-service
./mvnw spring-boot:run
```

### Docker

```bash
# Build image
./mvnw spring-boot:build-image -pl order-service

# Run container
docker run -p 8081:8081 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/orders \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=postgres \
  -e SPRING_RABBITMQ_HOST=rabbitmq \
  --network bookstore-network \
  sivaprasadreddy/bookstore-order-service:0.0.1-SNAPSHOT
```

## Monitoring

### Health Endpoint

```bash
curl http://localhost:8081/actuator/health
```

### Metrics

```bash
curl http://localhost:8081/actuator/prometheus
```

### API Documentation

```
http://localhost:8081/swagger-ui.html
```

## Future Enhancements

1. Implement payment processing
2. Add order fulfillment workflow
3. Implement inventory reservations
4. Add order history and analytics
5. Implement order notifications via SMS
6. Add order API rate limiting
7. Implement distributed tracing

---

**Last Updated**: April 2026  
**Version**: 1.0

