# Spring Boot Microservices Documentation Index

Welcome to the comprehensive documentation for the Spring Boot Microservices Course project. This documentation covers all aspects of the BookStore microservices application.

## 📋 Quick Navigation

### Getting Started
- **[Main Microservices Guide](./MICROSERVICES_GUIDE.md)** - Start here! Complete overview of the project architecture, all microservices, and technology stack.

### Individual Microservice Documentation

1. **[Catalog Service](./docs/CATALOG_SERVICE.md)**
   - Product management and browsing
   - REST API for product catalog
   - Database schema and migrations
   - Testing and deployment

2. **[Order Service](./docs/ORDER_SERVICE.md)**
   - Order creation and management
   - Event publishing and handling
   - OAuth2 security with Keycloak
   - Resilience patterns and circuit breakers
   - Distributed job scheduling

3. **[Notification Service](./docs/NOTIFICATION_SERVICE.md)**
   - Event-driven notifications
   - Email sending configuration
   - RabbitMQ message processing
   - Extensible notification system

4. **[API Gateway](./docs/API_GATEWAY.md)**
   - Request routing and load balancing
   - API documentation aggregation
   - Security and CORS handling
   - Rate limiting and circuit breaking

5. **[Bookstore WebApp](./docs/BOOKSTORE_WEBAPP.md)**
   - Customer-facing web application
   - Product browsing interface
   - Shopping cart and ordering
   - Thymeleaf templates and Alpine.js

### Infrastructure & Deployment

6. **[Deployment & Infrastructure Guide](./docs/DEPLOYMENT_INFRASTRUCTURE.md)**
   - Local development setup
   - Docker and Docker Compose
   - Kubernetes deployment
   - Database setup and management
   - Monitoring with Prometheus, Grafana, Loki, Tempo
   - Production deployment strategies
   - Operational tasks and maintenance

---

## 🎯 Project Overview

### What is BookStore?
BookStore is a complete microservices-based e-commerce application for selling books. It demonstrates modern Spring Boot practices including:

- **Microservices Architecture** - Independent, scalable services
- **Event-Driven Communication** - Async messaging with RabbitMQ
- **API Gateway** - Single entry point for all services
- **OAuth2 Security** - Secure authentication with Keycloak
- **Observability** - Complete monitoring and tracing
- **Containerization** - Docker and Kubernetes ready
- **Database Persistence** - PostgreSQL with Flyway migrations
- **Testing** - Comprehensive test coverage

### Core Services

| Service | Purpose | Port |
|---------|---------|------|
| **Catalog Service** | Product management and browsing | 8081 |
| **Order Service** | Order creation and management | 8082 |
| **Notification Service** | Event-driven notifications | 8083 |
| **API Gateway** | Request routing and aggregation | 8080 |
| **Bookstore WebApp** | Customer-facing web UI | 8084 |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT / WEB BROWSER                     │
└────────────────────────────┬────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Bookstore      │
                    │  WebApp         │
                    │  (Port 8084)    │
                    └────────┬────────┘
                             │
                    ┌────────▼───────────────────┐
                    │   API Gateway (Port 8080)  │
                    │  - Routes requests         │
                    │  - Aggregates docs         │
                    │  - Rate limiting           │
                    └────────┬───────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐      ┌──────▼──────┐      ┌──────▼──────┐
   │ Catalog  │      │   Order     │      │Notification │
   │ Service  │      │   Service   │      │ Service     │
   │(Port8081)│      │(Port 8082)  │      │(Port 8083)  │
   └────┬─────┘      └──────┬──────┘      └──────┬──────┘
        │                   │                    │
        └────────┬──────────┘                    │
                 │                               │
        ┌────────▼────────────────────┐         │
        │    PostgreSQL Database      │         │
        └─────────────────────────────┘   ┌─────▼──────────────┐
                                          │   RabbitMQ         │
                                          │   (Message Broker) │
                                          └────────────────────┘
```

---

## 🚀 Quick Start

### Prerequisites
- Java 21
- Docker & Docker Compose
- Git

### Get Up and Running (5 minutes)

```bash
# 1. Clone repository
git clone https://github.com/sivalabs/spring-boot-microservices-course.git
cd spring-boot-microservices-course

# 2. Start infrastructure
cd deployment/docker-compose
docker-compose -f infra.yml up -d

# 3. Build services
cd ../..
./mvnw clean install

# 4. Run services (in separate terminals)
cd catalog-service && ./mvnw spring-boot:run     # Terminal 1
cd order-service && ./mvnw spring-boot:run       # Terminal 2
cd notification-service && ./mvnw spring-boot:run # Terminal 3
cd api-gateway && ./mvnw spring-boot:run         # Terminal 4
cd bookstore-webapp && ./mvnw spring-boot:run    # Terminal 5

# 5. Access application
# WebApp: http://localhost:8084
# API Gateway: http://localhost:8080
# Keycloak: http://localhost:8180 (admin/admin)
```

---

## 📚 Documentation Structure

### By Role

**For Developers**
- Start with: [Main Microservices Guide](./MICROSERVICES_GUIDE.md)
- Then read: Individual service documentation (Catalog, Order, Notification, etc.)
- Reference: API endpoints and examples in each service guide

**For DevOps/Infrastructure**
- Start with: [Deployment & Infrastructure Guide](./docs/DEPLOYMENT_INFRASTRUCTURE.md)
- Topics covered: Docker, Kubernetes, monitoring, databases, backups

**For Architects**
- Start with: [Main Microservices Guide](./MICROSERVICES_GUIDE.md) - Architecture section
- Then read: Individual service documentation for design patterns

### By Service

Each service has comprehensive documentation including:
- Component overview and architecture
- API endpoints and examples
- Database schema
- Configuration options
- Deployment instructions
- Testing approaches

---

## 🔧 Key Technologies

### Framework & Language
- **Java 21** - Latest LTS version
- **Spring Boot 3.5.3** - Latest stable
- **Maven** - Build tool

### Data & Persistence
- **PostgreSQL** - Relational database
- **Spring Data JPA** - ORM framework
- **Flyway** - Database migrations

### Messaging & Events
- **RabbitMQ** - Message broker
- **Spring AMQP** - Message queue support

### Security
- **Spring Security** - Authentication & authorization
- **OAuth2** - Token-based security
- **Keycloak** - Identity provider

### API & Gateway
- **Spring Cloud Gateway** - API Gateway
- **RestClient** - Declarative HTTP client
- **OpenAPI/Swagger** - API documentation

### Frontend
- **Thymeleaf** - Server-side templates
- **Bootstrap** - CSS framework
- **Alpine.js** - Lightweight JavaScript

### Monitoring & Observability
- **Prometheus** - Metrics collection
- **Grafana** - Visualization
- **Loki** - Log aggregation
- **Tempo** - Distributed tracing
- **OpenTelemetry** - Instrumentation

### Testing
- **JUnit 5** - Unit testing
- **RestAssured** - REST API testing
- **Testcontainers** - Integration testing
- **WireMock** - HTTP mocking

---

## 📖 Documentation Sections

### [MICROSERVICES_GUIDE.md](./MICROSERVICES_GUIDE.md)
Comprehensive guide covering:
- Project overview and features
- System architecture with diagrams
- Complete microservices descriptions
- Full technology stack
- Getting started instructions
- API documentation access
- Service running procedures

### [CATALOG_SERVICE.md](./docs/CATALOG_SERVICE.md)
Details for Catalog Service:
- Service architecture and components
- REST API endpoints
- Database schema
- Configuration options
- Testing guide
- Deployment instructions

### [ORDER_SERVICE.md](./docs/ORDER_SERVICE.md)
Complete Order Service documentation:
- Order creation flow
- Event publishing system
- OAuth2 security setup
- Resilience patterns
- API request/response examples
- Testing strategies
- RabbitMQ configuration

### [NOTIFICATION_SERVICE.md](./docs/NOTIFICATION_SERVICE.md)
Notification Service guide:
- Message-driven architecture
- Email configuration
- Event listeners setup
- Database schema
- Testing approaches
- Extensibility options

### [API_GATEWAY.md](./docs/API_GATEWAY.md)
API Gateway documentation:
- Request routing configuration
- API documentation aggregation
- Security and CORS setup
- Performance tuning
- Monitoring and troubleshooting

### [BOOKSTORE_WEBAPP.md](./docs/BOOKSTORE_WEBAPP.md)
WebApp documentation:
- Web application architecture
- Thymeleaf templates
- Alpine.js functionality
- OAuth2 client configuration
- UI/UX features
- Testing guide

### [DEPLOYMENT_INFRASTRUCTURE.md](./docs/DEPLOYMENT_INFRASTRUCTURE.md)
Infrastructure and deployment guide:
- Local development setup
- Docker and Docker Compose
- Kubernetes deployment
- Database management
- RabbitMQ setup
- Keycloak authentication
- Monitoring stack (Prometheus, Grafana, Loki, Tempo)
- Production deployment strategies
- Operational tasks and maintenance

---

## 🎓 Learning Path

### Beginner Path (Start here!)
1. Read: [MICROSERVICES_GUIDE.md](./MICROSERVICES_GUIDE.md)
2. Run: Local development setup
3. Explore: Each service individually
4. Try: Simple API calls and UI interactions

### Intermediate Path (Advanced concepts)
1. Study: Event-driven architecture (Order → Notification)
2. Understand: OAuth2 security flow
3. Learn: Circuit breaker and resilience patterns
4. Implement: Custom modifications

### Advanced Path (Production ready)
1. Master: Kubernetes deployment
2. Setup: Full monitoring stack
3. Practice: Blue-green deployments
4. Implement: Custom CI/CD pipelines

---

## 🔍 API Examples

### Get Products
```bash
curl http://localhost:8080/catalog/api/products?page=1
```

### Create Order (requires auth)
```bash
curl -X POST http://localhost:8080/orders/api/orders \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

### View Swagger UI
```
http://localhost:8080/swagger-ui.html
```

See individual service documentation for complete API details.

---

## 🧪 Testing

### Run All Tests
```bash
./mvnw clean test
```

### Run Service-Specific Tests
```bash
./mvnw test -pl catalog-service
./mvnw test -pl order-service
./mvnw test -pl notification-service
```

### Run with Coverage
```bash
./mvnw clean test jacoco:report
```

---

## 📊 Monitoring Access

| Tool | URL | Default Credentials |
|------|-----|-------------------|
| **Grafana** | http://localhost:3000 | admin/admin |
| **Prometheus** | http://localhost:9090 | N/A |
| **RabbitMQ** | http://localhost:15672 | guest/guest |
| **Keycloak** | http://localhost:8180/admin | admin/admin |
| **Loki** | Via Grafana | N/A |
| **Tempo** | http://localhost:3200 | N/A |

---

## 🐛 Troubleshooting

### Service Won't Start
- Check ports are not in use: `lsof -i :8080`
- Verify database is running: `docker ps | grep postgres`
- Check logs: `./mvnw spring-boot:run`

### Database Connection Issues
- Verify PostgreSQL is running
- Check SPRING_DATASOURCE_URL environment variable
- Verify credentials in application.yml

### Message Queue Issues
- Verify RabbitMQ is running
- Check RabbitMQ admin console: http://localhost:15672
- View queue depths and dead letter queues

### Authentication Issues
- Access Keycloak admin: http://localhost:8180/admin
- Verify realm and client configuration
- Check JWT token validity

See individual service documentation for detailed troubleshooting.

---

## 🔗 External Resources

### Official Documentation
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Spring Cloud Gateway](https://spring.io/projects/spring-cloud-gateway)
- [Spring Security](https://spring.io/projects/spring-security)

### Learning Resources
- [SivaLabs Blog](https://sivalabs.in)
- [Spring Boot Tutorials](https://www.sivalabs.in/spring-boot-tutorials/)
- [YouTube Course](https://www.youtube.com/playlist?list=PLuNxlOYbv61g_ytin-wgkecfWDKVCEDmB)

### Tools & Technologies
- [Keycloak](https://www.keycloak.org/)
- [RabbitMQ](https://www.rabbitmq.com/)
- [PostgreSQL](https://www.postgresql.org/)
- [Docker](https://www.docker.com/)
- [Kubernetes](https://kubernetes.io/)

---

## 📝 Document Updates

This documentation is maintained alongside the codebase. For the latest versions:

- **Last Updated**: April 2026
- **Version**: 1.0
- **Repository**: https://github.com/sivalabs/spring-boot-microservices-course

---

## ❓ Getting Help

### Documentation Issues
If you find issues in the documentation:
1. Check the relevant service guide
2. Check troubleshooting section
3. Refer to inline code comments
4. Create an issue on GitHub

### Code Questions
- Review service source code
- Check test examples
- Consult Spring Boot documentation
- Join the community discussions

---

## 📋 Document Checklist

- ✅ Main Microservices Guide
- ✅ Catalog Service Documentation
- ✅ Order Service Documentation  
- ✅ Notification Service Documentation
- ✅ API Gateway Documentation
- ✅ Bookstore WebApp Documentation
- ✅ Deployment & Infrastructure Guide
- ✅ Documentation Index (this file)

---

**Happy Learning! 🎉**

Start with [MICROSERVICES_GUIDE.md](./MICROSERVICES_GUIDE.md) and explore the microservices world!


