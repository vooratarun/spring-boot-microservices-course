# Deployment & Infrastructure Guide

## Overview

This guide covers deployment strategies, infrastructure setup, monitoring, and operational procedures for the BookStore microservices application.

## Table of Contents
1. [Local Development Setup](#local-development-setup)
2. [Docker & Docker Compose](#docker--docker-compose)
3. [Kubernetes Deployment](#kubernetes-deployment)
4. [Database Setup](#database-setup)
5. [Message Broker Setup](#message-broker-setup)
6. [Authentication (Keycloak)](#authentication-keycloak)
7. [Monitoring & Observability](#monitoring--observability)
8. [Production Deployment](#production-deployment)
9. [Operational Tasks](#operational-tasks)

---

## Local Development Setup

### Prerequisites

1. **Java 21**
   ```bash
   # Using SDKMAN
   sdk install java 21
   sdk use java 21
   ```

2. **Docker & Docker Compose**
   ```bash
   # Install Docker Desktop from https://www.docker.com/products/docker-desktop/
   docker --version
   docker-compose --version
   ```

3. **Git**
   ```bash
   git --version
   ```

4. **IDE** (IntelliJ IDEA or VS Code)

### Getting Started

1. **Clone Repository**
   ```bash
   git clone https://github.com/sivalabs/spring-boot-microservices-course.git
   cd spring-boot-microservices-course
   ```

2. **Start Infrastructure**
   ```bash
   cd deployment/docker-compose
   docker-compose -f infra.yml up -d
   ```

3. **Build Services**
   ```bash
   cd ../..
   ./mvnw clean install
   ```

4. **Run Services**
   ```bash
   # Terminal 1 - Catalog Service
   cd catalog-service && ./mvnw spring-boot:run
   
   # Terminal 2 - Order Service
   cd order-service && ./mvnw spring-boot:run
   
   # Terminal 3 - Notification Service
   cd notification-service && ./mvnw spring-boot:run
   
   # Terminal 4 - API Gateway
   cd api-gateway && ./mvnw spring-boot:run
   
   # Terminal 5 - WebApp
   cd bookstore-webapp && ./mvnw spring-boot:run
   ```

5. **Access Application**
   - WebApp: http://localhost:8083
   - API Gateway: http://localhost:8080
   - Catalog Service: http://localhost:8081
   - Order Service: http://localhost:8082
   - Keycloak: http://localhost:8180

---

## Docker & Docker Compose

### Building Docker Images

#### Method 1: Maven
```bash
# Build single service
./mvnw spring-boot:build-image -pl catalog-service

# Build all services
./mvnw spring-boot:build-image
```

#### Method 2: Docker CLI
```bash
cd catalog-service
docker build -t bookstore-catalog-service:0.0.1 .
```

### Docker Compose Files

Located in `deployment/docker-compose/`

#### infra.yml - Infrastructure Services

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: bookstore
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data

  rabbitmq:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5672:5672"
      - "15672:15672"

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      DB_VENDOR: postgres
      DB_ADDR: postgres
      DB_USER: postgres
      DB_PASSWORD: postgres
    ports:
      - "8180:8080"
    depends_on:
      - postgres

volumes:
  postgres-data:

networks:
  default:
    name: bookstore-network
```

#### apps.yml - Application Services

```yaml
version: '3.8'

services:
  catalog-service:
    image: sivaprasadreddy/bookstore-catalog-service:0.0.1-SNAPSHOT
    ports:
      - "8081:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/catalog
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
    depends_on:
      - postgres
    networks:
      - bookstore-network

  order-service:
    image: sivaprasadreddy/bookstore-order-service:0.0.1-SNAPSHOT
    ports:
      - "8082:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/orders
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_PORT: 5672
    depends_on:
      - postgres
      - rabbitmq
    networks:
      - bookstore-network

  notification-service:
    image: sivaprasadreddy/bookstore-notification-service:0.0.1-SNAPSHOT
    ports:
      - "8083:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/notifications
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_PORT: 5672
    depends_on:
      - postgres
      - rabbitmq
    networks:
      - bookstore-network

  api-gateway:
    image: sivaprasadreddy/bookstore-api-gateway:0.0.1-SNAPSHOT
    ports:
      - "8080:8080"
    environment:
      SPRING_CLOUD_GATEWAY_ROUTES_0_URI: http://catalog-service:8080
      SPRING_CLOUD_GATEWAY_ROUTES_1_URI: http://order-service:8080
    depends_on:
      - catalog-service
      - order-service
    networks:
      - bookstore-network

  bookstore-webapp:
    image: sivaprasadreddy/bookstore-bookstore-webapp:0.0.1-SNAPSHOT
    ports:
      - "8084:8080"
    environment:
      BOOKSTORE_APIGATEWYURL: http://api-gateway:8080
    depends_on:
      - api-gateway
    networks:
      - bookstore-network

networks:
  bookstore-network:
    driver: bridge
```

#### monitoring.yml - Monitoring Services

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - bookstore-network

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - bookstore-network

  loki:
    image: grafana/loki:latest
    volumes:
      - ./promtail/promtail-docker-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - bookstore-network

  tempo:
    image: grafana/tempo:latest
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo/tempo.yml:/etc/tempo.yaml
      - tempo-data:/var/tempo
    ports:
      - "3200:3200"
      - "14268:14268"
    networks:
      - bookstore-network

volumes:
  prometheus-data:
  grafana-data:
  loki-data:
  tempo-data:

networks:
  bookstore-network:
    driver: bridge
```

### Docker Compose Commands

```bash
# Start all services
docker-compose -f infra.yml -f apps.yml -f monitoring.yml up -d

# View logs
docker-compose logs -f catalog-service

# Stop services
docker-compose down

# Stop and remove volumes
docker-compose down -v

# Rebuild images
docker-compose up --build

# Scale service
docker-compose up -d --scale order-service=3
```

---

## Kubernetes Deployment

### Kubernetes Resources

#### Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore
```

#### Deployment Example - Catalog Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: bookstore
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalog-service
  template:
    metadata:
      labels:
        app: catalog-service
    spec:
      containers:
      - name: catalog-service
        image: sivaprasadreddy/bookstore-catalog-service:0.0.1-SNAPSHOT
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: datasource-url
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

#### Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: bookstore
spec:
  type: ClusterIP
  selector:
    app: catalog-service
  ports:
  - port: 8080
    targetPort: 8080
```

#### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookstore-ingress
  namespace: bookstore
spec:
  ingressClassName: nginx
  rules:
  - host: bookstore.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-gateway
            port:
              number: 8080
      - path: /admin
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 8080
```

### Deployment Commands

```bash
# Create namespace
kubectl create namespace bookstore

# Deploy application
kubectl apply -f k8s/

# View deployments
kubectl get deployments -n bookstore

# Scale deployment
kubectl scale deployment catalog-service --replicas=3 -n bookstore

# View logs
kubectl logs -f deployment/catalog-service -n bookstore

# Port forward
kubectl port-forward service/api-gateway 8080:8080 -n bookstore
```

---

## Database Setup

### PostgreSQL

#### Initial Setup

```bash
# Connect to PostgreSQL
docker exec -it postgres psql -U postgres

# Create databases
CREATE DATABASE catalog;
CREATE DATABASE orders;
CREATE DATABASE notifications;
```

#### Schema Initialization

```bash
# Flyway migrations run automatically on service startup
# Migrations are located in: src/main/resources/db/migration/
```

#### Backup & Restore

```bash
# Backup database
docker exec postgres pg_dump -U postgres catalog > catalog-backup.sql

# Restore database
docker exec -i postgres psql -U postgres < catalog-backup.sql

# Backup all databases
docker exec postgres pg_dumpall -U postgres > full-backup.sql
```

---

## Message Broker Setup

### RabbitMQ

#### Configuration

```bash
# Access RabbitMQ Management Console
http://localhost:15672
# Username: guest
# Password: guest
```

#### Queue Setup

Queues are created automatically by the services, but can be set up manually:

```bash
# Create exchange
docker exec rabbitmq rabbitmqctl eval \
  'rabbit_exchange:declare({exchange, <<"order-events-exchange">>, topic, true, false, []}).'

# Create queue
docker exec rabbitmq rabbitmqctl eval \
  'rabbit_amqqueue:declare({queue, <<"new-orders-queue">>, true, false, false, []}).'
```

#### Connection Settings

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
```

---

## Authentication (Keycloak)

### Initial Setup

1. **Access Keycloak Admin Console**
   ```
   http://localhost:8180/admin
   Username: admin
   Password: admin
   ```

2. **Create Realm**
   - Click "Create Realm"
   - Name: `bookstore`
   - Click "Create"

3. **Create Client**
   - Navigate to Clients
   - Click "Create client"
   - Client ID: `bookstore-app`
   - Click "Next"
   - Enable "Standard flow supported"
   - Click "Save"

4. **Configure Client**
   - Set Valid Redirect URIs: `http://localhost:8083/login/oauth2/code/bookstore`
   - Set Valid Post Logout Redirect URIs: `http://localhost:8083`
   - Save

5. **Create Users**
   - Navigate to Users
   - Click "Create user"
   - Fill user details
   - Set password

### Configuration

**Application Properties**:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          bookstore:
            client-id: bookstore-app
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: http://localhost:8083/login/oauth2/code/bookstore
            scope: openid,profile,email
        provider:
          bookstore:
            issuer-uri: http://localhost:8180/realms/bookstore
            user-name-attribute: name
```

---

## Monitoring & Observability

### Prometheus

**Metrics Collection**:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'catalog-service'
    static_configs:
      - targets: ['localhost:8081']
    metrics_path: '/actuator/prometheus'

  - job_name: 'order-service'
    static_configs:
      - targets: ['localhost:8082']
    metrics_path: '/actuator/prometheus'
```

**Access**: http://localhost:9090

### Grafana

**Setup**:

1. Access http://localhost:3000
2. Login with admin/admin
3. Add Prometheus data source
4. Import dashboards

**Key Dashboards**:
- Spring Boot Metrics
- Service Health
- Database Connections
- HTTP Request Metrics

### Loki (Log Aggregation)

**Configuration**:

```yaml
# loki-local-config.yaml
auth_enabled: false

ingester:
  chunk_idle_period: 3m
  max_chunk_age: 1h

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema:
        prefix: index_
        version: v11

server:
  http_listen_port: 3100
```

**Access Loki in Grafana**:
- Add Loki as data source: http://loki:3100
- Query logs: `{service="catalog-service"}`

### Tempo (Distributed Tracing)

**Configuration**:

```yaml
# tempo.yml
server:
  http_listen_port: 3200

backends:
  local:
    path: /var/tempo

auth_enabled: false
```

**Access**: http://localhost:3200

---

## Production Deployment

### Pre-deployment Checklist

- [ ] All tests passing
- [ ] Code review completed
- [ ] Security vulnerabilities scanned
- [ ] Performance testing done
- [ ] Configuration review
- [ ] Backup strategy in place
- [ ] Monitoring configured
- [ ] Runbooks prepared

### Environment Variables

```bash
# Database
SPRING_DATASOURCE_URL=jdbc:postgresql://prod-db:5432/catalog
SPRING_DATASOURCE_USERNAME=catalog_user
SPRING_DATASOURCE_PASSWORD=$(aws secretsmanager get-secret-value --secret-id db-password)

# RabbitMQ
SPRING_RABBITMQ_HOST=prod-rabbitmq
SPRING_RABBITMQ_USERNAME=rabbitmq_user
SPRING_RABBITMQ_PASSWORD=$(aws secretsmanager get-secret-value --secret-id rabbitmq-password)

# Keycloak
SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_BOOKSTORE_CLIENT_SECRET=$(aws secretsmanager get-secret-value --secret-id keycloak-secret)

# Application
SPRING_PROFILES_ACTIVE=production
LOGGING_LEVEL_ROOT=WARN
LOGGING_LEVEL_COM_SIVALABS=INFO
```

### Blue-Green Deployment

```bash
# Deploy green environment
docker-compose -f blue-green.yml up -d green

# Run smoke tests
./run-smoke-tests.sh green

# Switch traffic to green
docker-compose -f blue-green.yml exec nginx \
  sed -i 's/blue/green/g' /etc/nginx/conf.d/default.conf

# Remove blue environment
docker-compose -f blue-green.yml down blue
```

### Canary Deployment

```bash
# Deploy to 10% of traffic
kubectl set image deployment/catalog-service \
  catalog-service=new-image:v2 \
  --record \
  -n bookstore

# Monitor metrics
kubectl top deployment catalog-service -n bookstore

# Gradually increase to 100%
kubectl rollout status deployment/catalog-service -n bookstore
```

---

## Operational Tasks

### Service Maintenance

#### Restart Service

```bash
# Docker Compose
docker-compose restart catalog-service

# Kubernetes
kubectl rollout restart deployment/catalog-service -n bookstore
```

#### View Logs

```bash
# Docker Compose
docker-compose logs -f --tail 100 catalog-service

# Kubernetes
kubectl logs -f deployment/catalog-service -n bookstore

# Multiple lines with context
kubectl logs -f deployment/catalog-service -n bookstore --tail 50 -p
```

#### Health Checks

```bash
curl http://localhost:8081/actuator/health
```

### Database Maintenance

#### Database Cleanup

```sql
-- Remove old orders (older than 1 year)
DELETE FROM orders 
WHERE created_at < NOW() - INTERVAL '1 year';

-- Vacuum and analyze
VACUUM ANALYZE orders;
VACUUM ANALYZE order_items;
```

#### Connection Pool Monitoring

```bash
# View connection pool status
curl http://localhost:8081/actuator/metrics/hikaricp.connections
```

### Message Queue Monitoring

#### Check Queue Depths

```bash
# Via RabbitMQ API
curl -u guest:guest http://localhost:15672/api/queues

# Via command line
docker exec rabbitmq rabbitmqctl list_queues
```

#### Purge Dead Letter Queue

```bash
docker exec rabbitmq rabbitmqctl purge_queue dlq-new-orders
```

### Performance Tuning

#### JVM Heap Size

```bash
export JAVA_OPTS="-Xms512m -Xmx2g -XX:+UseG1GC"
./mvnw spring-boot:run
```

#### Database Connection Pool

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

### Rollback Procedure

```bash
# Docker Compose
docker-compose pull # Pull previous version
docker-compose up -d

# Kubernetes
kubectl rollout undo deployment/catalog-service -n bookstore
kubectl rollout status deployment/catalog-service -n bookstore
```

### Disaster Recovery

#### Backup Strategy

```bash
# Full backup script
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)

# Backup databases
docker exec postgres pg_dumpall -U postgres | \
  gzip > /backups/postgres_${BACKUP_DATE}.sql.gz

# Backup configuration
tar czf /backups/config_${BACKUP_DATE}.tar.gz \
  deployment/docker-compose/

# Upload to S3
aws s3 cp /backups/ s3://bookstore-backups/
```

#### Recovery Procedure

```bash
# Restore database
gunzip -c postgres_backup.sql.gz | docker exec -i postgres psql -U postgres

# Restore services
docker-compose -f infra.yml -f apps.yml up -d
```

---

**Last Updated**: April 2026  
**Version**: 1.0

