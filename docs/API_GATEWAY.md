# API Gateway Documentation

## Overview

The **API Gateway** is the single entry point for all client requests to the BookStore microservices. It routes requests to appropriate backend services, aggregates API documentation, and provides cross-cutting concerns like request/response filtering, rate limiting, and security.

## Service Details

| Property | Value |
|----------|-------|
| **Module** | `api-gateway` |
| **Port** | `8080` (default) |
| **Type** | Reactive (Spring Cloud Gateway) |
| **Package** | `com.sivalabs.bookstore.gateway` |
| **Main Class** | `ApiGatewayApplication` |

## Architecture

### Gateway Request Flow

```
                        ┌─────────────────┐
                        │   Client/WebApp │
                        └────────┬────────┘
                                 │
                                 ▼
                    ┌────────────────────────────┐
                    │    API Gateway (Port 8080) │
                    │  Spring Cloud Gateway      │
                    └────────────┬────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
            ┌──────────────┐ ┌──────────────┐
            │   Catalog    │ │    Order     │
            │   Service    │ │   Service    │
            │  (Port 8081) │ │  (Port 8082) │
            └──────────────┘ └──────────────┘
                    │            │
                    └────────┬───┘
                             ▼
            ┌──────────────────────────────┐
            │   Keycloak Auth              │
            │  (OAuth2/OIDC)               │
            └──────────────────────────────┘
```

## Routing Configuration

### Request Routing

The API Gateway routes requests to backend services based on path patterns:

| Path Pattern | Destination | Service | Port |
|------------|-------------|---------|------|
| `/catalog/**` | http://localhost:8081 | Catalog Service | 8081 |
| `/orders/**` | http://localhost:8082 | Order Service | 8082 |
| `/swagger-ui.html` | Aggregated docs | All services | - |

### Route Definition

**File**: `src/main/resources/application.yml`

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: catalog-service
          uri: http://localhost:8081
          predicates:
            - Path=/catalog/**
          filters:
            - StripPrefix=1
            
        - id: order-service
          uri: http://localhost:8082
          predicates:
            - Path=/orders/**
          filters:
            - StripPrefix=1
```

### Example Route Mapping

**Client Request**:
```
GET http://localhost:8080/catalog/api/products
```

**Routes to**:
```
GET http://catalog-service:8081/api/products
```

**Client Request**:
```
POST http://localhost:8080/orders/api/orders
Authorization: Bearer <JWT_TOKEN>
```

**Routes to**:
```
POST http://order-service:8082/api/orders
Authorization: Bearer <JWT_TOKEN>
```

## API Documentation Aggregation

### OpenAPI/Swagger Integration

The API Gateway aggregates OpenAPI documentation from all backend services into a single Swagger UI.

**Access aggregated documentation**:
```
http://localhost:8080/swagger-ui.html
```

### Documentation Features

- **Aggregated Endpoints** - View all service endpoints in one place
- **Cross-Service Operations** - See dependencies between services
- **Authentication** - Test endpoints with OAuth2 tokens
- **Request/Response Examples** - View sample payloads

## Core Components

### 1. ApiGatewayApplication
**File**: `src/main/java/com/sivalabs/bookstore/gateway/ApiGatewayApplication.java`

Entry point of the API Gateway.

```java
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

### 2. Gateway Configuration
**File**: `src/main/resources/application.yml`

Central configuration for all routing and gateway features.

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials
      
      routes:
        # Catalog Service Routes
        - id: catalog-service
          uri: http://localhost:8081
          predicates:
            - Path=/catalog/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Forwarded-For,${X-Forwarded-For}
            
        # Order Service Routes
        - id: order-service
          uri: http://localhost:8082
          predicates:
            - Path=/orders/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Forwarded-For,${X-Forwarded-For}
```

## Features

### 1. Request Routing

Routes incoming requests to appropriate backend services.

```yaml
predicates:
  - Path=/pattern/**
  - Method=GET,POST
  - Header=X-Header-Name,regex-value
```

### 2. Request/Response Filtering

Modifies requests and responses in flight.

```yaml
filters:
  - StripPrefix=1                    # Remove path prefix
  - AddRequestHeader=X-Custom,value  # Add custom headers
  - AddResponseHeader=X-Resp,value   # Add response headers
  - RewritePath=/api/(.*),$1         # Rewrite paths
```

### 3. Load Balancing

Distributes load across multiple service instances.

```yaml
uri: lb://catalog-service  # Uses load balancer
```

### 4. Circuit Breaking

Prevents cascading failures.

```yaml
filters:
  - CircuitBreaker=myCircuitBreaker
```

### 5. Rate Limiting

Controls request rates.

```yaml
filters:
  - RequestRateLimiter=myRateLimiter
```

### 6. Retry Policy

Retries failed requests.

```yaml
filters:
  - Retry=3
```

### 7. CORS Handling

Cross-Origin Resource Sharing configuration.

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "http://localhost:3000,http://localhost:8083"
            allowedMethods: GET,POST,PUT,DELETE,OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
```

## Security

### OAuth2 Token Forwarding

The gateway forwards OAuth2 tokens from clients to backend services.

**Request with Token**:
```http
GET /catalog/api/products
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Forwarded to Backend**:
```http
GET /api/products
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
X-Forwarded-For: <original-client-ip>
```

### Security Headers

The gateway can add security headers:

```yaml
filters:
  - AddResponseHeader=X-Content-Type-Options,nosniff
  - AddResponseHeader=X-Frame-Options,DENY
  - AddResponseHeader=X-XSS-Protection,1; mode=block
```

## Health Check Endpoint

```bash
curl http://localhost:8080/actuator/health
```

**Response**:
```json
{
  "status": "UP",
  "components": {
    "discoveryComposite": {
      "status": "UP"
    },
    "gateway": {
      "status": "UP"
    }
  }
}
```

## Metrics & Monitoring

### Prometheus Metrics

```bash
curl http://localhost:8080/actuator/prometheus
```

**Key Metrics**:
- `spring_cloud_gateway_requests_total` - Total requests
- `spring_cloud_gateway_requests_seconds` - Request duration
- `spring_cloud_gateway_route_tags` - Route information

### Logging

Enable detailed logging:

```yaml
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    org.springframework.security: DEBUG
```

## Configuration

### Application Properties

**File**: `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: api-gateway
  
  cloud:
    gateway:
      # Routes configuration
      routes:
        - id: catalog-service
          uri: http://localhost:8081
          predicates:
            - Path=/catalog/**
          filters:
            - StripPrefix=1
        
        - id: order-service
          uri: http://localhost:8082
          predicates:
            - Path=/orders/**
          filters:
            - StripPrefix=1
      
      # Global filters
      default-filters:
        - AddRequestHeader=X-API-Gateway,true
      
      # CORS configuration
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "*"
            allowedMethods: "*"
            allowedHeaders: "*"

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
```

### Environment Variables

```bash
SPRING_CLOUD_GATEWAY_ROUTES_0_URI=http://catalog-service:8081
SPRING_CLOUD_GATEWAY_ROUTES_1_URI=http://order-service:8082
```

## Dependencies

### Core Dependencies

```xml
<!-- Spring Cloud Gateway -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway-server-webflux</artifactId>
</dependency>

<!-- Spring Boot Actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- OpenAPI/Swagger Aggregation -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webflux-ui</artifactId>
    <version>2.8.9</version>
</dependency>

<!-- Prometheus Metrics -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- OpenTelemetry Tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

## Running the Service

### Development

```bash
cd api-gateway
./mvnw spring-boot:run
```

### Docker

```bash
# Build image
./mvnw spring-boot:build-image -pl api-gateway

# Run container
docker run -p 8080:8080 \
  -e SPRING_CLOUD_GATEWAY_ROUTES_0_URI=http://catalog-service:8081 \
  -e SPRING_CLOUD_GATEWAY_ROUTES_1_URI=http://order-service:8082 \
  --network bookstore-network \
  sivaprasadreddy/bookstore-api-gateway:0.0.1-SNAPSHOT
```

## Testing

### Integration Tests

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiGatewayTest {

    @LocalServerPort
    int port;

    @Test
    void shouldRouteToCalogService() {
        given()
            .baseUri("http://localhost:" + port)
            .when()
            .get("/catalog/api/products")
            .then()
            .statusCode(200);
    }
}
```

### Running Tests

```bash
./mvnw test -pl api-gateway
```

## Example Use Cases

### Get Products Through Gateway

```bash
curl http://localhost:8080/catalog/api/products?page=1
```

Routes to: `http://catalog-service:8081/api/products?page=1`

### Create Order Through Gateway

```bash
curl -X POST http://localhost:8080/orders/api/orders \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

Routes to: `http://order-service:8082/api/orders` (with forwarded auth token)

## Distributed Tracing

The gateway integrates with OpenTelemetry for distributed tracing.

**Trace propagation**:
- Incoming trace context is extracted
- Trace ID is propagated to backend services
- All logs are correlated with trace ID

**View traces in Tempo**:
```
http://localhost:3200
```

## Performance Tuning

### Connection Pooling

```yaml
spring:
  cloud:
    gateway:
      httpclient:
        connect-timeout: 5000
        response-timeout: 10s
        pool:
          max-idle-time: 30000
          max-life-time: 60000
```

### Request Size Limits

```yaml
server:
  tomcat:
    max-http-post-size: 10MB
```

## Troubleshooting

### Service Not Responding

1. Check backend service is running
2. Verify network connectivity
3. Check gateway logs for errors
4. Verify routing configuration

### Slow Response

1. Check backend service performance
2. Monitor gateway metrics
3. Verify no circuit breakers are open
4. Check database performance

### Request Timeouts

1. Increase `response-timeout`
2. Increase connect timeout
3. Check backend service load
4. Verify network latency

## Advanced Configuration

### Custom Filters

```java
@Component
public class CustomGatewayFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // Add custom logic
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

### Request/Response Logging

```yaml
logging:
  level:
    org.springframework.cloud.gateway.filter.factory: DEBUG
    org.springframework.web.server.adapter.HttpWebHandlerAdapter: DEBUG
```

## Future Enhancements

1. Add API versioning support
2. Implement API key authentication
3. Add request validation
4. Implement API rate limiting per user
5. Add request/response transformation
6. Implement service discovery (Eureka/Consul)
7. Add webhook routing
8. Implement API analytics

---

**Last Updated**: April 2026  
**Version**: 1.0

