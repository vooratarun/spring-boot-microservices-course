# Catalog Service Documentation

## Overview

The **Catalog Service** is a RESTful microservice responsible for managing the product catalog (books) in the BookStore application. It provides endpoints for browsing, searching, and retrieving product information.

## Service Details

| Property | Value |
|----------|-------|
| **Module** | `catalog-service` |
| **Port** | `8080` (default) |
| **Database** | PostgreSQL |
| **Package** | `com.sivalabs.bookstore.catalog` |
| **Main Class** | `CatalogServiceApplication` |

## Architecture

### Layered Architecture

```
┌─────────────────────────────────────────┐
│         REST Controller Layer            │
│      (ProductController)                 │
├─────────────────────────────────────────┤
│      Service / Business Logic Layer      │
│         (ProductService)                 │
├─────────────────────────────────────────┤
│        Repository / Data Access Layer    │
│      (ProductRepository - JPA)           │
├─────────────────────────────────────────┤
│           Database Layer                 │
│         (PostgreSQL Database)            │
└─────────────────────────────────────────┘
```

## Core Components

### 1. CatalogServiceApplication
**File**: `src/main/java/com/sivalabs/bookstore/catalog/CatalogServiceApplication.java`

Entry point of the Catalog Service.

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class CatalogServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(CatalogServiceApplication.class, args);
    }
}
```

### 2. ProductController
**File**: `src/main/java/com/sivalabs/bookstore/catalog/web/controllers/ProductController.java`

REST controller handling product-related HTTP requests.

**Endpoints**:

| Method | Path | Description | Parameters |
|--------|------|-------------|-----------|
| `GET` | `/api/products` | Get paginated list of products | `page` (query param, default=1) |
| `GET` | `/api/products/{code}` | Get product by code | `code` (path param) |

**Controller Code**:
```java
@RestController
@RequestMapping("/api/products")
class ProductController {
    private final ProductService productService;

    ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    PagedResult<Product> getProducts(@RequestParam(name = "page", defaultValue = "1") int pageNo) {
        return productService.getProducts(pageNo);
    }

    @GetMapping("/{code}")
    ResponseEntity<Product> getProductByCode(@PathVariable String code) {
        return productService
                .getProductByCode(code)
                .map(ResponseEntity::ok)
                .orElseThrow(() -> ProductNotFoundException.forCode(code));
    }
}
```

### 3. ProductService
**File**: `src/main/java/com/sivalabs/bookstore/catalog/domain/ProductService.java`

Business logic layer for product operations.

**Key Methods**:
- `getProducts(int pageNo)` - Fetch paginated products
- `getProductByCode(String code)` - Fetch single product by code
- `getAllProducts()` - Fetch all products

### 4. ProductRepository
**File**: `src/main/java/com/sivalabs/bookstore/catalog/domain/ProductRepository.java`

Spring Data JPA repository for database operations.

```java
public interface ProductRepository extends JpaRepository<ProductEntity, Long> {
    Optional<ProductEntity> findByCode(String code);
}
```

### 5. ProductEntity
**File**: `src/main/java/com/sivalabs/bookstore/catalog/domain/ProductEntity.java`

JPA entity representing a product in the database.

**Fields**:
- `id` - Primary key
- `code` - Product code (unique)
- `name` - Product name
- `description` - Product description
- `imageUrl` - Image URL
- `price` - Product price
- `createdAt` - Creation timestamp

### 6. Product (DTO)
**File**: `src/main/java/com/sivalabs/bookstore/catalog/domain/Product.java`

Data Transfer Object for product responses.

### 7. ProductMapper
**File**: `src/main/java/com/sivalabs/bookstore/catalog/domain/ProductMapper.java`

Maps between `ProductEntity` and `Product` DTO.

### 8. ApplicationProperties
**File**: `src/main/java/com/sivalabs/bookstore/catalog/ApplicationProperties.java`

Configuration properties for the service.

## API Responses

### Get Products (Paginated)

**Request**:
```http
GET /api/products?page=1
```

**Response (200 OK)**:
```json
{
  "data": [
    {
      "code": "P001",
      "name": "Spring in Action",
      "description": "Learn Spring framework",
      "price": 49.99,
      "imageUrl": "https://example.com/image.jpg"
    },
    {
      "code": "P002",
      "name": "Microservices Patterns",
      "description": "Design microservices",
      "price": 59.99,
      "imageUrl": "https://example.com/image2.jpg"
    }
  ],
  "pageNumber": 1,
  "pageSize": 10,
  "totalElements": 25,
  "totalPages": 3,
  "isFirst": true,
  "isLast": false,
  "hasNext": true,
  "hasPrevious": false
}
```

### Get Product by Code

**Request**:
```http
GET /api/products/P001
```

**Response (200 OK)**:
```json
{
  "code": "P001",
  "name": "Spring in Action",
  "description": "Learn Spring framework",
  "price": 49.99,
  "imageUrl": "https://example.com/image.jpg"
}
```

**Response (404 Not Found)**:
```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "status": 404,
  "error": "Not Found",
  "message": "Product with code: P001 not found"
}
```

## Database Schema

The Catalog Service uses a PostgreSQL database with Flyway migrations for version control.

### Products Table

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    code VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    price DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Migrations

Flyway migrations are located in: `src/main/resources/db/migration/`

**Example**:
```
V1__initial_schema.sql - Initial database setup
V2__seed_products.sql - Initial product data
```

## Configuration

### Application Properties

**File**: `src/main/resources/application.yml`

Key configurations:

```yaml
spring:
  application:
    name: catalog-service
  datasource:
    url: jdbc:postgresql://localhost:5432/catalog
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true

server:
  port: 8080
  
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
```

### Environment Variables

For production deployment, set:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-host:5432/catalog
SPRING_DATASOURCE_USERNAME=catalog_user
SPRING_DATASOURCE_PASSWORD=secure_password
```

## Dependencies

### Core Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>

<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.8.9</version>
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
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>

<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <scope>test</scope>
</dependency>
```

## Testing

### Unit Tests

Tests are located in: `src/test/java/com/sivalabs/bookstore/catalog/`

**Example Test**:
```java
@Sql("/test-data.sql")
class ProductControllerTest extends AbstractIT {

    @Test
    void shouldReturnProducts() {
        given().contentType(ContentType.JSON)
                .when()
                .get("/api/products")
                .then()
                .statusCode(200)
                .body("data", hasSize(10))
                .body("totalElements", is(15))
                .body("pageNumber", is(1));
    }
}
```

### Running Tests

```bash
# Run all tests
./mvnw test -pl catalog-service

# Run specific test class
./mvnw test -pl catalog-service -Dtest=ProductControllerTest

# Run with coverage
./mvnw clean test jacoco:report -pl catalog-service
```

## Monitoring & Observability

### Health Endpoint

```bash
curl http://localhost:8080/actuator/health
```

### Metrics Endpoint

```bash
curl http://localhost:8080/actuator/prometheus
```

### API Documentation

Interactive Swagger/OpenAPI documentation:
```
http://localhost:8080/swagger-ui.html
```

### Log Aggregation

Logs are sent to Loki for aggregation. Query in Grafana:

```
{service="catalog-service"}
```

## Running the Service

### Development

```bash
cd catalog-service
./mvnw spring-boot:run
```

### Docker Build

```bash
./mvnw spring-boot:build-image -pl catalog-service
```

### Docker Run

```bash
docker run -p 8080:8080 \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/catalog \
  -e SPRING_DATASOURCE_USERNAME=postgres \
  -e SPRING_DATASOURCE_PASSWORD=postgres \
  --network bookstore-network \
  sivaprasadreddy/bookstore-catalog-service:0.0.1-SNAPSHOT
```

## Exception Handling

### GlobalExceptionHandler

**File**: `src/main/java/com/sivalabs/bookstore/catalog/web/exception/GlobalExceptionHandler.java`

Handles exceptions and returns standardized error responses.

**Handled Exceptions**:
- `ProductNotFoundException` - Product not found (404)
- `ValidationException` - Invalid request (400)
- `General Exceptions` - Server errors (500)

## Performance Considerations

1. **Database Indexing**: Ensure indexes on frequently queried columns (code, name)
2. **Pagination**: Always use pagination for list endpoints
3. **Caching**: Consider Redis caching for product lookups
4. **Connection Pooling**: HikariCP is configured by default

## Security

- Read-only endpoints (no authentication required)
- Suitable for public product browsing
- Future consideration: Add product management endpoints with security

## Future Enhancements

1. Add product filtering (by category, price range)
2. Implement full-text search
3. Add product reviews and ratings
4. Implement caching with Redis
5. Add product inventory management
6. Implement product recommendations

---

**Last Updated**: April 2026  
**Version**: 1.0

