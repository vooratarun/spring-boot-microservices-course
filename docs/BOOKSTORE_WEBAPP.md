# Bookstore WebApp Documentation

## Overview

The **Bookstore WebApp** is a customer-facing web application built with Spring Boot, Thymeleaf, and Alpine.js. It provides a user interface for customers to browse products, manage shopping carts, place orders, and track order status. The application integrates with other microservices through the API Gateway.

## Service Details

| Property | Value |
|----------|-------|
| **Module** | `bookstore-webapp` |
| **Port** | `8083` (default) |
| **Type** | Spring Boot MVC |
| **Template Engine** | Thymeleaf |
| **Frontend Framework** | Alpine.js + Bootstrap |
| **Package** | `com.sivalabs.bookstore.webapp` |
| **Main Class** | `BookstoreWebappApplication` |

## Architecture

### WebApp Architecture

```
┌────────────────────────────────────────────────────────────┐
│                        Web Browser                         │
│        (Chrome, Firefox, Safari, Edge, etc.)               │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
        ┌────────────────────────────────────┐
        │   Bookstore WebApp (Port 8083)     │
        │                                    │
        │  ┌────────────────────────────────┐│
        │  │   Spring Boot Controller        ││
        │  │   - ProductController           ││
        │  │   - OrderController             ││
        │  │   - AuthController              ││
        │  └────────────────────────────────┘│
        │  ┌────────────────────────────────┐│
        │  │   Thymeleaf Templates           ││
        │  │   - products.html               ││
        │  │   - cart.html                   ││
        │  │   - orders.html                 ││
        │  │   - order_details.html          ││
        │  └────────────────────────────────┘│
        │  ┌────────────────────────────────┐│
        │  │   Alpine.js (Client-side JS)   ││
        │  │   - Dynamic cart updates        ││
        │  │   - API interactions            ││
        │  │   - Form validation             ││
        │  └────────────────────────────────┘│
        │  ┌────────────────────────────────┐│
        │  │   Bootstrap CSS Framework      ││
        │  │   - Responsive design           ││
        │  │   - UI components               ││
        │  └────────────────────────────────┘│
        └────────────────┬───────────────────┘
                         │
        ┌────────────────┼───────────────────┐
        │                │                   │
        ▼                ▼                   ▼
    ┌──────────┐    ┌─────────────┐   ┌──────────────┐
    │ Keycloak │    │ API Gateway │   │ Static Files │
    │ (OAuth2) │    │ (Port 8080) │   │ (CSS, JS)    │
    └──────────┘    └─────────────┘   └──────────────┘
                         │
        ┌────────────────┼───────────────────┐
        │                │                   │
        ▼                ▼                   ▼
    ┌────────────┐ ┌────────────┐   ┌──────────────┐
    │  Catalog   │ │   Order    │   │ Notification │
    │  Service   │ │  Service   │   │  Service     │
    └────────────┘ └────────────┘   └──────────────┘
```

## Core Components

### 1. BookstoreWebappApplication
**File**: `src/main/java/com/sivalabs/bookstore/webapp/BookstoreWebappApplication.java`

Entry point of the WebApp.

### 2. ProductController
**File**: `src/main/java/com/sivalabs/bookstore/webapp/web/controllers/ProductController.java`

Handles product browsing functionality.

**Endpoints**:

| Method | Path | Description | Returns |
|--------|------|-------------|---------|
| `GET` | `/` | Home/redirect to products | Redirect to /products |
| `GET` | `/products` | Product listing page | HTML page |
| `GET` | `/api/products` | Get products (AJAX) | JSON |

**Controller Code**:
```java
@Controller
class ProductController {
    private final CatalogServiceClient catalogService;

    ProductController(CatalogServiceClient catalogService) {
        this.catalogService = catalogService;
    }

    @GetMapping
    String index() {
        return "redirect:/products";
    }

    @GetMapping("/products")
    String showProductsPage(@RequestParam(name = "page", defaultValue = "1") int page, Model model) {
        model.addAttribute("pageNo", page);
        return "products";
    }

    @GetMapping("/api/products")
    @ResponseBody
    PagedResult<Product> products(@RequestParam(name = "page", defaultValue = "1") int page) {
        return catalogService.getProducts(page);
    }
}
```

### 3. OrderController
**File**: `src/main/java/com/sivalabs/bookstore/webapp/web/controllers/OrderController.java`

Handles order-related functionality.

**Endpoints**:

| Method | Path | Description | Auth Required |
|--------|------|-------------|---------------|
| `GET` | `/cart` | Shopping cart page | ✗ |
| `POST` | `/api/orders` | Create order | ✓ |
| `GET` | `/orders` | Order history page | ✓ |
| `GET` | `/orders/{orderNumber}` | Order details page | ✓ |
| `GET` | `/api/orders` | Get orders (AJAX) | ✓ |
| `GET` | `/api/orders/{orderNumber}` | Get order details (AJAX) | ✓ |

**Controller Code**:
```java
@Controller
class OrderController {
    private final OrderServiceClient orderServiceClient;
    private final SecurityHelper securityHelper;

    @GetMapping("/cart")
    String cart() {
        return "cart";
    }

    @PostMapping("/api/orders")
    @ResponseBody
    OrderConfirmationDTO createOrder(@Valid @RequestBody CreateOrderRequest orderRequest) {
        return orderServiceClient.createOrder(getHeaders(), orderRequest);
    }

    @GetMapping("/orders/{orderNumber}")
    String showOrderDetails(@PathVariable String orderNumber, Model model) {
        model.addAttribute("orderNumber", orderNumber);
        return "order_details";
    }

    @GetMapping("/api/orders/{orderNumber}")
    @ResponseBody
    OrderDTO getOrder(@PathVariable String orderNumber) {
        return orderServiceClient.getOrder(getHeaders(), orderNumber);
    }

    @GetMapping("/orders")
    String showOrders() {
        return "orders";
    }

    @GetMapping("/api/orders")
    @ResponseBody
    List<OrderSummary> getOrders() {
        return orderServiceClient.getOrders(getHeaders());
    }

    private Map<String, ?> getHeaders() {
        String accessToken = securityHelper.getAccessToken();
        return Map.of("Authorization", "Bearer " + accessToken);
    }
}
```

### 4. CatalogServiceClient
**File**: `src/main/java/com/sivalabs/bookstore/webapp/clients/catalog/CatalogServiceClient.java`

Declarative HTTP client for Catalog Service.

```java
public interface CatalogServiceClient {
    @GetExchange("/catalog/api/products")
    PagedResult<Product> getProducts(@RequestParam(name = "page", defaultValue = "1") int page);

    @GetExchange("/catalog/api/products/{code}")
    Optional<Product> getProductByCode(@PathVariable String code);
}
```

### 5. OrderServiceClient
**File**: `src/main/java/com/sivalabs/bookstore/webapp/clients/orders/OrderServiceClient.java`

Declarative HTTP client for Order Service.

```java
public interface OrderServiceClient {
    @PostExchange("/orders/api/orders")
    OrderConfirmationDTO createOrder(
            @RequestHeader Map<String, ?> headers, 
            @RequestBody CreateOrderRequest orderRequest);

    @GetExchange("/orders/api/orders")
    List<OrderSummary> getOrders(@RequestHeader Map<String, ?> headers);

    @GetExchange("/orders/api/orders/{orderNumber}")
    OrderDTO getOrder(@RequestHeader Map<String, ?> headers, @PathVariable String orderNumber);
}
```

### 6. SecurityHelper
**File**: `src/main/java/com/sivalabs/bookstore/webapp/services/SecurityHelper.java`

Helper for security-related operations.

**Methods**:
- `getAccessToken()` - Get current user's access token
- `getUserName()` - Get current user's username
- `getUserEmail()` - Get current user's email

### 7. ApplicationProperties
**File**: `src/main/java/com/sivalabs/bookstore/webapp/ApplicationProperties.java`

Configuration properties for the WebApp.

```java
@ConfigurationProperties(prefix = "bookstore")
public record ApplicationProperties(String apiGatewayUrl) {}
```

## Frontend Pages

### 1. Home / Products Page
**File**: `src/main/resources/templates/products.html`

Main product listing page with pagination.

**Features**:
- Product grid display
- Pagination
- Add to cart functionality
- Search/filter (future enhancement)

**Alpine.js Integration**:
```html
<div x-data="{ cart: [] }">
    <button @click="addToCart(product)" class="btn btn-primary">
        Add to Cart
    </button>
</div>
```

### 2. Shopping Cart Page
**File**: `src/main/resources/templates/cart.html`

Shopping cart management page.

**Features**:
- Display cart items
- Update quantities
- Remove items
- Calculate totals
- Checkout button

**Alpine.js State**:
```html
<div x-data="cartStore()">
    <template x-for="item in cart.items">
        <div>
            <input type="number" x-model="item.quantity" @change="updateQuantity(item)">
            <button @click="removeItem(item)">Remove</button>
        </div>
    </template>
    <button @click="checkout()">Proceed to Checkout</button>
</div>
```

### 3. Checkout/Order Placement
**File**: `src/main/resources/templates/checkout.html` (embedded in cart.html)

Order placement form with customer and delivery information.

**Features**:
- Customer information form
- Delivery address form
- Order review
- Place order button

**Form Validation** (using Alpine.js):
```html
<form @submit.prevent="submitOrder">
    <input type="email" x-model="form.customer.email" required>
    <input type="tel" x-model="form.customer.phone" required>
    <button type="submit" :disabled="!isFormValid">Place Order</button>
</form>
```

### 4. Orders List Page
**File**: `src/main/resources/templates/orders.html`

User's order history page.

**Features**:
- List all orders
- Order status
- Order date
- Total amount
- View details link

**Alpine.js**:
```html
<div x-data="{ orders: [] }" @load="fetchOrders()">
    <template x-for="order in orders">
        <tr>
            <td x-text="order.orderNumber"></td>
            <td x-text="order.status"></td>
            <td x-text="formatDate(order.createdAt)"></td>
            <td x-text="'$' + order.totalPrice.toFixed(2)"></td>
        </tr>
    </template>
</div>
```

### 5. Order Details Page
**File**: `src/main/resources/templates/order_details.html`

Detailed view of a specific order.

**Features**:
- Order number and status
- Customer information
- Delivery address
- Order items with prices
- Order total
- Order timeline (future enhancement)

## HTML Templates

All templates use:
- **Thymeleaf** for server-side rendering
- **Bootstrap** for CSS framework
- **Alpine.js** for client-side interactivity

### Template Structure

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title th:text="${title}">Page Title</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
</head>
<body>
    <nav th:include="fragments/header :: header"></nav>
    
    <div class="container mt-4">
        <main th:include="fragments/${page} :: content"></main>
    </div>
    
    <footer th:include="fragments/footer :: footer"></footer>
    
    <script src="/js/app.js"></script>
</body>
</html>
```

## Client-Side JavaScript (Alpine.js)

### Example: Cart Store

**File**: `src/main/resources/static/js/stores/cart.js`

```javascript
function cartStore() {
    return {
        items: [],
        
        addToCart(product) {
            const existing = this.items.find(item => item.code === product.code);
            if (existing) {
                existing.quantity++;
            } else {
                this.items.push({
                    code: product.code,
                    name: product.name,
                    price: product.price,
                    quantity: 1
                });
            }
            this.saveToLocalStorage();
        },
        
        removeItem(item) {
            this.items = this.items.filter(i => i.code !== item.code);
            this.saveToLocalStorage();
        },
        
        updateQuantity(item) {
            if (item.quantity <= 0) {
                this.removeItem(item);
            } else {
                this.saveToLocalStorage();
            }
        },
        
        getTotalPrice() {
            return this.items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
        },
        
        saveToLocalStorage() {
            localStorage.setItem('cart', JSON.stringify(this.items));
        },
        
        loadFromLocalStorage() {
            const saved = localStorage.getItem('cart');
            if (saved) {
                this.items = JSON.parse(saved);
            }
        },
        
        checkout() {
            if (this.items.length === 0) {
                alert('Cart is empty');
                return;
            }
            // Proceed to checkout
        }
    }
}
```

## Static Files

### CSS Files
- **Location**: `src/main/resources/static/css/`
- **Bootstrap**: CDN hosted
- **Custom CSS**: `styles.css`

### JavaScript Files
- **Location**: `src/main/resources/static/js/`
- **Alpine.js**: CDN hosted
- **App Logic**: `app.js`

## Configuration

### Application Properties

**File**: `src/main/resources/application.yml`

```yaml
spring:
  application:
    name: bookstore-webapp
  security:
    oauth2:
      client:
        registration:
          bookstore:
            client-id: bookstore-app
            client-secret: ${CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8083/login/oauth2/code/bookstore
            scope: openid,profile,email
        provider:
          bookstore:
            issuer-uri: http://localhost:8180/realms/bookstore
            user-name-attribute: name

bookstore:
  apiGatewayUrl: http://localhost:8080

server:
  port: 8083
```

### OAuth2 Security Configuration

**File**: `src/main/java/com/sivalabs/bookstore/webapp/config/SecurityConfig.java`

Configures Spring Security with OAuth2.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          bookstore:
            client-id: bookstore-app
            client-secret: secret
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8083/login/oauth2/code/bookstore
```

## Dependencies

### Core Dependencies

```xml
<!-- Spring Boot Web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- Spring Security OAuth2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>

<!-- Thymeleaf -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Bootstrap & Alpine.js (via CDN, no dependency needed) -->
```

## Testing

### Integration Tests

```java
@SpringBootTest
class ProductControllerTest {

    @MockBean
    private CatalogServiceClient catalogServiceClient;

    @Test
    void shouldDisplayProductsPage() {
        List<Product> products = Arrays.asList(
            new Product("P001", "Book 1", 29.99),
            new Product("P002", "Book 2", 39.99)
        );
        
        given(catalogServiceClient.getProducts(1))
            .willReturn(new PagedResult<>(products, 1, 2, 2, true, false));
        
        // Test display
    }
}
```

### Running Tests

```bash
./mvnw test -pl bookstore-webapp
```

## Running the Service

### Development

```bash
cd bookstore-webapp
./mvnw spring-boot:run
```

Access at: `http://localhost:8083`

### Docker

```bash
# Build image
./mvnw spring-boot:build-image -pl bookstore-webapp

# Run container
docker run -p 8083:8083 \
  -e BOOKSTORE_APIGATEWYURL=http://api-gateway:8080 \
  -e SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_BOOKSTORE_CLIENT_SECRET=secret \
  --network bookstore-network \
  sivaprasadreddy/bookstore-bookstore-webapp:0.0.1-SNAPSHOT
```

## User Flow

### 1. Browse Products
1. User visits `http://localhost:8083`
2. Redirected to `/products`
3. Products loaded via AJAX from Catalog Service
4. User can add items to cart (stored in localStorage)

### 2. Place Order
1. User navigates to `/cart`
2. Reviews cart items
3. Clicks "Proceed to Checkout"
4. Login required (redirects to Keycloak if not authenticated)
5. Fills customer and delivery information
6. Submits order (calls Order Service via API Gateway)
7. Order confirmation displayed

### 3. View Orders
1. User navigates to `/orders`
2. Orders loaded via AJAX from Order Service
3. Can click on order to view details
4. Order timeline shows status updates

## Security Considerations

1. **OAuth2 Authentication** - Users must authenticate via Keycloak
2. **Token Management** - AccessToken stored securely (httpOnly cookie recommended)
3. **CSRF Protection** - Spring Security's default CSRF protection enabled
4. **XSS Prevention** - Thymeleaf auto-escapes by default
5. **CORS** - Configured to allow requests from browser

## Performance Optimization

1. **Client-side Caching** - Cart stored in localStorage
2. **Lazy Loading** - Products loaded on demand
3. **CDN** - Bootstrap and Alpine.js from CDN
4. **Minified Assets** - JavaScript and CSS minified in production

## Monitoring

### Health Endpoint

```bash
curl http://localhost:8083/actuator/health
```

### Metrics

```bash
curl http://localhost:8083/actuator/prometheus
```

## Future Enhancements

1. Add product search and filtering
2. Implement wishlist functionality
3. Add product reviews and ratings
4. Implement order notifications
5. Add customer profile management
6. Implement payment integration
7. Add analytics tracking
8. Implement multi-language support
9. Add dark mode theme
10. Implement progressive web app (PWA) features

---

**Last Updated**: April 2026  
**Version**: 1.0

