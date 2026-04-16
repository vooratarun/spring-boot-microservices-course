# Notification Service Documentation

## Overview

The **Notification Service** is an event-driven microservice responsible for listening to order events and sending notifications to customers. It consumes messages from RabbitMQ and sends emails based on order events (creation, delivery, cancellation, and errors).

## Service Details

| Property | Value |
|----------|-------|
| **Module** | `notification-service` |
| **Port** | `8082` (default) |
| **Database** | PostgreSQL (for event logging) |
| **Message Broker** | RabbitMQ |
| **Package** | `com.sivalabs.bookstore.notifications` |
| **Main Class** | `NotificationServiceApplication` |

## Architecture

### Message-Driven Architecture

```
┌──────────────────────────────────────────────────────┐
│          Message Broker (RabbitMQ)                   │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │   order-events-exchange                        │ │
│  │   (order.* routing keys)                       │ │
│  └─────────────────┬────────────────────────────┬─┘ │
│                    │                            │    │
│        ┌───────────┴───────────┐       ┌──────┴──────────┐
│        │                       │       │                 │
│     [new-orders-queue]   [delivered-orders-queue]  [cancelled-orders-queue]
│        │                       │       │
└────────┼───────────────────────┼───────┼─────────────────┘
         │                       │       │
         ▼                       ▼       ▼
┌──────────────────────────────────────────────────────┐
│      Notification Service (Message Listeners)        │
│                                                      │
│  ┌────────────────┐    ┌────────────────┐           │
│  │ OrderCreated   │    │ OrderDelivered │           │
│  │ Listener       │    │ Listener       │           │
│  └────────┬───────┘    └────────┬───────┘           │
│           │                     │                   │
│  ┌────────▼────────────────────▼──────┐            │
│  │    NotificationService             │            │
│  │  - sendOrderCreatedNotification()  │            │
│  │  - sendOrderDeliveredNotification()│            │
│  │  - sendOrderCancelledNotification()│            │
│  │  - sendOrderErrorNotification()    │            │
│  └────────┬─────────────────────────┬─┘            │
│           │                         │              │
│           ▼                         ▼              │
│    [Email Sender]           [Database Logger]      │
│        (SMTP)                (PostgreSQL)           │
└──────────────────────────────────────────────────────┘
         │                         │
         ▼                         ▼
    [Email Server]         [Event Storage]
```

## Core Components

### 1. NotificationServiceApplication
**File**: `src/main/java/com/sivalabs/bookstore/notifications/NotificationServiceApplication.java`

Entry point of the Notification Service.

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class NotificationServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(NotificationServiceApplication.class, args);
    }
}
```

### 2. NotificationService
**File**: `src/main/java/com/sivalabs/bookstore/notifications/domain/NotificationService.java`

Core business logic for sending notifications.

**Key Methods**:
```java
public void sendOrderCreatedNotification(OrderCreatedEvent event)
public void sendOrderDeliveredNotification(OrderDeliveredEvent event)
public void sendOrderCancelledNotification(OrderCancelledEvent event)
public void sendOrderErrorEventNotification(OrderErrorEvent event)
private void sendEmail(String recipient, String subject, String content)
```

**Service Implementation**:
```java
@Service
public class NotificationService {
    private final JavaMailSender emailSender;
    private final ApplicationProperties properties;

    public void sendOrderCreatedNotification(OrderCreatedEvent event) {
        String message = """
            Order Created Notification
            Dear %s,
            Your order with orderNumber: %s has been created successfully.
            Thanks,
            BookStore Team
            """.formatted(event.customer().name(), event.orderNumber());
        
        log.info("\n{}", message);
        sendEmail(event.customer().email(), "Order Created Notification", message);
    }

    // Other notification methods...
}
```

### 3. Event Models

**File**: `src/main/java/com/sivalabs/bookstore/notifications/domain/models/`

Event DTOs for different order events:

#### OrderCreatedEvent
```java
public record OrderCreatedEvent(
    String orderNumber,
    Customer customer,
    LocalDateTime createdAt
) {}

public record Customer(String name, String email) {}
```

#### OrderDeliveredEvent
```java
public record OrderDeliveredEvent(
    String orderNumber,
    Customer customer,
    LocalDateTime deliveredAt
) {}
```

#### OrderCancelledEvent
```java
public record OrderCancelledEvent(
    String orderNumber,
    Customer customer,
    String reason,
    LocalDateTime cancelledAt
) {}
```

#### OrderErrorEvent
```java
public record OrderErrorEvent(
    String orderNumber,
    String reason,
    LocalDateTime errorAt
) {}
```

### 4. Event Listeners

**File**: `src/main/java/com/sivalabs/bookstore/notifications/domain/`

Message-driven POJOs that listen to RabbitMQ queues.

**Example Listener**:
```java
@Component
public class OrderCreatedEventListener {
    private final NotificationService notificationService;

    public OrderCreatedEventListener(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    @RabbitListener(queues = "${notification.newOrdersQueue}")
    public void listen(OrderCreatedEvent event) {
        log.info("Received OrderCreatedEvent: {}", event.orderNumber());
        notificationService.sendOrderCreatedNotification(event);
    }
}
```

### 5. ApplicationProperties
**File**: `src/main/java/com/sivalabs/bookstore/notifications/ApplicationProperties.java`

Configuration properties for the service.

```java
@ConfigurationProperties(prefix = "notification")
public record ApplicationProperties(
        String orderEventsExchange,
        String newOrdersQueue,
        String deliveredOrdersQueue,
        String cancelledOrdersQueue,
        String errorOrdersQueue,
        String supportEmail) {}
```

## RabbitMQ Configuration

### Message Broker Setup

**File**: `src/main/java/com/sivalabs/bookstore/notifications/config/RabbitMQConfig.java`

Defines exchanges, queues, and bindings for order events.

**Configuration**:
```yaml
notification:
  orderEventsExchange: order-events-exchange
  newOrdersQueue: new-orders-queue
  deliveredOrdersQueue: delivered-orders-queue
  cancelledOrdersQueue: cancelled-orders-queue
  errorOrdersQueue: error-orders-queue
  supportEmail: support@bookstore.com

spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

### Queue Configuration

**Queues**:
1. `new-orders-queue` - Receives OrderCreatedEvent messages
2. `delivered-orders-queue` - Receives OrderDeliveredEvent messages
3. `cancelled-orders-queue` - Receives OrderCancelledEvent messages
4. `error-orders-queue` - Receives OrderErrorEvent messages

**Exchange**: `order-events-exchange` (Topic exchange)

**Routing Keys**:
- `order.created` → new-orders-queue
- `order.delivered` → delivered-orders-queue
- `order.cancelled` → cancelled-orders-queue
- `order.error` → error-orders-queue

## Email Configuration

### SMTP Settings

**File**: `src/main/resources/application.yml`

```yaml
spring:
  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USERNAME}
    password: ${MAIL_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true
          connectiontimeout: 5000
          timeout: 5000
          writetimeout: 5000
```

### Email Templates

Email notifications are sent with the following templates:

**Order Created Email**:
```
===================================================
Order Created Notification
----------------------------------------------------
Dear {customer_name},
Your order with orderNumber: {order_number} has been created successfully.

Thanks,
BookStore Team
===================================================
```

**Order Delivered Email**:
```
===================================================
Order Delivered Notification
----------------------------------------------------
Dear {customer_name},
Your order with orderNumber: {order_number} has been delivered successfully.

Thanks,
BookStore Team
===================================================
```

**Order Cancelled Email**:
```
===================================================
Order Cancelled Notification
----------------------------------------------------
Dear {customer_name},
Your order with orderNumber: {order_number} has been cancelled.
Reason: {cancellation_reason}

Thanks,
BookStore Team
===================================================
```

**Order Error Email**:
```
===================================================
Order Processing Failure Notification
----------------------------------------------------
Hi {support_email},
The order processing failed for orderNumber: {order_number}.
Reason: {error_reason}

Thanks,
BookStore Team
===================================================
```

## Database Schema

### Notifications Table (Optional)

For storing notification history:

```sql
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(100) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    recipient_email VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'SENT',
    sent_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    error_message TEXT
);
```

## Configuration

### Application Properties

**File**: `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: notification-service
  datasource:
    url: jdbc:postgresql://localhost:5432/notifications
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
  mail:
    host: smtp.gmail.com
    port: 587
    username: your-email@gmail.com
    password: your-app-password
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
            required: true

server:
  port: 8082

notification:
  orderEventsExchange: order-events-exchange
  newOrdersQueue: new-orders-queue
  deliveredOrdersQueue: delivered-orders-queue
  cancelledOrdersQueue: cancelled-orders-queue
  errorOrdersQueue: error-orders-queue
  supportEmail: support@bookstore.com

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

## Dependencies

### Core Dependencies

```xml
<!-- Spring Boot AMQP -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<!-- Spring Boot Mail -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>

<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- PostgreSQL -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Flyway DB -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

### Test Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-test</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>rabbitmq</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

## Testing

### Unit Tests

Tests are located in: `src/test/java/com/sivalabs/bookstore/notifications/`

**Example Test**:
```java
class NotificationServiceTest {

    private NotificationService notificationService;
    private JavaMailSender emailSender;
    private ApplicationProperties properties;

    @BeforeEach
    void setUp() {
        emailSender = mock(JavaMailSender.class);
        properties = new ApplicationProperties(
            "exchange", "q1", "q2", "q3", "q4", "support@bookstore.com"
        );
        notificationService = new NotificationService(emailSender, properties);
    }

    @Test
    void shouldSendOrderCreatedNotification() {
        OrderCreatedEvent event = new OrderCreatedEvent(
            "ORD-123",
            new Customer("John", "john@example.com"),
            LocalDateTime.now()
        );

        notificationService.sendOrderCreatedNotification(event);

        verify(emailSender, times(1)).send(any(MimeMessage.class));
    }
}
```

### Integration Tests

```java
@SpringBootTest
@Testcontainers
class NotificationServiceIntegrationTest {

    @Container
    static RabbitMQContainer rabbitmq = new RabbitMQContainer("rabbitmq:latest");

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:latest");

    @Test
    void shouldListenAndProcessOrderCreatedEvent() {
        // Send event to RabbitMQ
        // Verify notification was sent
    }
}
```

### Running Tests

```bash
# Run all tests
./mvnw test -pl notification-service

# Run specific test class
./mvnw test -pl notification-service -Dtest=NotificationServiceTest

# Run with coverage
./mvnw clean test jacoco:report -pl notification-service
```

## Running the Service

### Development

```bash
cd notification-service
./mvnw spring-boot:run
```

### Docker

```bash
# Build image
./mvnw spring-boot:build-image -pl notification-service

# Run container
docker run -p 8082:8082 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/notifications \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=postgres \
  -e SPRING_RABBITMQ_HOST=rabbitmq \
  -e SPRING_MAIL_HOST=smtp.gmail.com \
  -e SPRING_MAIL_USERNAME=your-email@gmail.com \
  -e SPRING_MAIL_PASSWORD=your-app-password \
  --network bookstore-network \
  sivaprasadreddy/bookstore-notification-service:0.0.1-SNAPSHOT
```

## Monitoring

### Health Endpoint

```bash
curl http://localhost:8082/actuator/health
```

### Metrics

```bash
curl http://localhost:8082/actuator/prometheus
```

### RabbitMQ Management UI

```
http://localhost:15672
Username: guest
Password: guest
```

## Event Flow Example

### Scenario: Customer Places an Order

1. **Customer creates order** via bookstore-webapp
2. **Order Service receives request** and validates
3. **Order created in database** with PENDING status
4. **OrderCreatedEvent published** to RabbitMQ
5. **Notification Service receives** the event
6. **Email sent** to customer with order confirmation
7. **Event logged** in database (optional)

### Logs Output Example

```
2024-01-15 10:30:45 INFO  [notification-service] Received OrderCreatedEvent: ORD-20240115-00001
2024-01-15 10:30:46 INFO  [notification-service] 
===================================================
Order Created Notification
----------------------------------------------------
Dear John Doe,
Your order with orderNumber: ORD-20240115-00001 has been created successfully.

Thanks,
BookStore Team
===================================================
2024-01-15 10:30:47 INFO  [notification-service] Email sent to: john@example.com
```

## Error Handling

### Email Sending Failures

If email sending fails:
1. Exception is logged
2. Message is retried (if configured)
3. Error message is stored
4. Alert is triggered (if configured)

### Message Processing Failures

If message processing fails:
1. Message remains in queue
2. Service attempts redelivery
3. Message moves to dead letter queue after max retries
4. Admin notification sent

## Performance Considerations

1. **Async Processing** - Email sending happens asynchronously
2. **Message Batching** - Can batch multiple notifications
3. **Connection Pooling** - RabbitMQ and email connections pooled
4. **Rate Limiting** - Can limit email sending rate
5. **Caching** - Cache email templates if needed

## Security Considerations

1. **SMTP Credentials** - Store in environment variables, not in code
2. **Email Encryption** - Use TLS/STARTTLS for SMTP
3. **Message Validation** - Validate incoming messages before processing
4. **Sensitive Data** - Don't log sensitive customer information

## Extensibility

The notification service can be extended to support:

1. **SMS Notifications** - Add SMS provider integration
2. **Push Notifications** - Add FCM/APNs integration
3. **Slack Integration** - Send notifications to Slack
4. **SMS via Twilio** - For SMS notifications
5. **Custom Email Templates** - Support templating engines like Thymeleaf
6. **Notification Preferences** - Let customers choose notification channels
7. **Delivery Reports** - Track email delivery status

## Future Enhancements

1. Add notification preferences per user
2. Implement notification templates
3. Add SMS notification support
4. Implement push notifications
5. Add notification scheduling
6. Implement notification throttling
7. Add audit trail for notifications
8. Implement notification retries with backoff
9. Add webhook integration for external systems
10. Implement notification analytics

---

**Last Updated**: April 2026  
**Version**: 1.0

