# Build Instructions
## Data Integration and Analytics Platform

**Date**: 2026-03-04  
**Status**: Ready for Implementation  
**Build System**: Maven (Backend), npm/Vite (Frontend)

---

## Overview

This document provides comprehensive build instructions for all 6 units of the Data Integration and Analytics Platform. The platform consists of 5 backend microservices (Java/Spring Boot) and 1 frontend application (React/TypeScript).

---

## Prerequisites

### Development Environment
- **Java**: JDK 17 or higher
- **Node.js**: v18 or higher
- **npm**: v9 or higher
- **Maven**: 3.8 or higher
- **Docker**: 20.10 or higher
- **Docker Compose**: 2.0 or higher
- **Git**: 2.30 or higher

### Infrastructure Dependencies
- **PostgreSQL**: 15+ (for metadata, configuration, audit)
- **MongoDB**: 6+ (for document storage, audit logs)
- **TimescaleDB**: 2.10+ (for time-series data)
- **Redis**: 7+ (for caching)
- **Apache Kafka**: 3.4+ (for messaging)
- **Elasticsearch**: 8+ (for logging)

---

## Build Order

The services should be built in the following order due to dependencies:

1. **Unit 6**: Monitoring & Audit Service (foundation)
2. **Unit 3**: Storage Management Service (foundation)
3. **Unit 1**: Data Ingestion Service
4. **Unit 2**: Data Processing Pipeline
5. **Unit 4**: Analytics API Service
6. **Unit 5**: Dashboard Frontend

---

## Unit 6: Monitoring & Audit Service

### Location
```
monitoring-audit-service/
```

### Build Commands
```bash
cd monitoring-audit-service

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package (creates JAR)
mvn package

# Skip tests if needed
mvn package -DskipTests

# Build Docker image
docker build -t monitoring-audit-service:1.0.0 .
```

### Build Output
- **JAR**: `target/monitoring-audit-service-1.0.0.jar`
- **Docker Image**: `monitoring-audit-service:1.0.0`

### Configuration Files
- `src/main/resources/application.yml` - Main configuration
- `src/main/resources/application-dev.yml` - Development profile
- `src/main/resources/application-prod.yml` - Production profile
- `src/main/resources/logback-spring.xml` - Logging configuration

### Environment Variables Required
```bash
# MongoDB
MONGODB_HOST=localhost
MONGODB_PORT=27017
MONGODB_DATABASE=monitoring_audit
MONGODB_USERNAME=admin
MONGODB_PASSWORD=<password>

# Elasticsearch
ELASTICSEARCH_HOST=localhost
ELASTICSEARCH_PORT=9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=<password>

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Prometheus
PROMETHEUS_ENABLED=true
PROMETHEUS_PORT=9090
```

### Verification
```bash
# Check JAR was created
ls -lh target/monitoring-audit-service-1.0.0.jar

# Verify Docker image
docker images | grep monitoring-audit-service

# Check dependencies
mvn dependency:tree
```

---

## Unit 3: Storage Management Service

### Location
```
storage-management-service/
```

### Build Commands
```bash
cd storage-management-service

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package (creates JAR)
mvn package

# Build Docker image
docker build -t storage-management-service:1.0.0 .
```

### Build Output
- **JAR**: `target/storage-management-service-1.0.0.jar`
- **Docker Image**: `storage-management-service:1.0.0`

### Configuration Files
- `src/main/resources/application.yml` - Main configuration
- `src/main/resources/application-dev.yml` - Development profile
- `src/main/resources/application-prod.yml` - Production profile

### Environment Variables Required
```bash
# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=analytics_db
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=<password>

# MongoDB
MONGODB_HOST=localhost
MONGODB_PORT=27017
MONGODB_DATABASE=analytics_documents
MONGODB_USERNAME=admin
MONGODB_PASSWORD=<password>

# TimescaleDB
TIMESCALE_HOST=localhost
TIMESCALE_PORT=5432
TIMESCALE_DATABASE=analytics_timeseries
TIMESCALE_USERNAME=postgres
TIMESCALE_PASSWORD=<password>

# Snowflake (optional for dev)
SNOWFLAKE_ACCOUNT=<account>
SNOWFLAKE_USERNAME=<username>
SNOWFLAKE_PASSWORD=<password>
SNOWFLAKE_DATABASE=ANALYTICS_DW
SNOWFLAKE_WAREHOUSE=COMPUTE_WH

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=<password>
```

### Verification
```bash
# Check JAR was created
ls -lh target/storage-management-service-1.0.0.jar

# Verify Docker image
docker images | grep storage-management-service

# Test database connections
mvn test -Dtest=DatabaseConnectionTest
```

---

## Unit 1: Data Ingestion Service

### Location
```
data-ingestion-service/
```

### Build Commands
```bash
cd data-ingestion-service

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package (creates JAR)
mvn package

# Build Docker image
docker build -t data-ingestion-service:1.0.0 .
```

### Build Output
- **JAR**: `target/data-ingestion-service-1.0.0.jar`
- **Docker Image**: `data-ingestion-service:1.0.0`

### Configuration Files
- `src/main/resources/application.yml` - Main configuration
- `src/main/resources/application-dev.yml` - Development profile
- `src/main/resources/application-prod.yml` - Production profile
- `src/main/resources/connectors/` - Connector configurations

### Environment Variables Required
```bash
# PostgreSQL (metadata)
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=ingestion_metadata
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=<password>

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Redis (job state)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=<password>

# AWS S3 (file storage)
AWS_ACCESS_KEY_ID=<key>
AWS_SECRET_ACCESS_KEY=<secret>
AWS_REGION=us-east-1
AWS_S3_BUCKET=data-ingestion-files

# Scheduler
SCHEDULER_ENABLED=true
SCHEDULER_POOL_SIZE=10
```

### Verification
```bash
# Check JAR was created
ls -lh target/data-ingestion-service-1.0.0.jar

# Verify Docker image
docker images | grep data-ingestion-service

# Test connectors
mvn test -Dtest=ConnectorTest
```

---

## Unit 2: Data Processing Pipeline

### Location
```
data-processing-pipeline/
```

### Build Commands
```bash
cd data-processing-pipeline

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package (creates JAR)
mvn package

# Build Docker image
docker build -t data-processing-pipeline:1.0.0 .
```

### Build Output
- **JAR**: `target/data-processing-pipeline-1.0.0.jar`
- **Docker Image**: `data-processing-pipeline:1.0.0`

### Configuration Files
- `src/main/resources/application.yml` - Main configuration
- `src/main/resources/application-dev.yml` - Development profile
- `src/main/resources/application-prod.yml` - Production profile
- `src/main/resources/rules/` - Business rule definitions

### Environment Variables Required
```bash
# PostgreSQL (metadata, rules)
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DATABASE=processing_metadata
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=<password>

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_CONSUMER_GROUP=data-processing-group
KAFKA_CONSUMER_THREADS=10

# Redis (deduplication cache)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=<password>

# Processing Configuration
PROCESSING_BATCH_SIZE=1000
PROCESSING_PARALLEL_THREADS=10
PROCESSING_QUALITY_THRESHOLD=85
```

### Verification
```bash
# Check JAR was created
ls -lh target/data-processing-pipeline-1.0.0.jar

# Verify Docker image
docker images | grep data-processing-pipeline

# Test rule engine
mvn test -Dtest=RuleEngineTest
```

---

## Unit 4: Analytics API Service

### Location
```
analytics-api-service/
```

### Build Commands
```bash
cd analytics-api-service

# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package (creates JAR)
mvn package

# Build Docker image
docker build -t analytics-api-service:1.0.0 .
```

### Build Output
- **JAR**: `target/analytics-api-service-1.0.0.jar`
- **Docker Image**: `analytics-api-service:1.0.0`

### Configuration Files
- `src/main/resources/application.yml` - Main configuration
- `src/main/resources/application-dev.yml` - Development profile
- `src/main/resources/application-prod.yml` - Production profile

### Environment Variables Required
```bash
# PostgreSQL (read replica)
POSTGRES_READ_HOST=localhost
POSTGRES_READ_PORT=5432
POSTGRES_READ_DATABASE=analytics_db
POSTGRES_READ_USERNAME=readonly
POSTGRES_READ_PASSWORD=<password>

# MongoDB (read replica)
MONGODB_READ_HOST=localhost
MONGODB_READ_PORT=27017
MONGODB_READ_DATABASE=analytics_documents
MONGODB_READ_USERNAME=readonly
MONGODB_READ_PASSWORD=<password>

# TimescaleDB (read replica)
TIMESCALE_READ_HOST=localhost
TIMESCALE_READ_PORT=5432
TIMESCALE_READ_DATABASE=analytics_timeseries
TIMESCALE_READ_USERNAME=readonly
TIMESCALE_READ_PASSWORD=<password>

# Snowflake
SNOWFLAKE_ACCOUNT=<account>
SNOWFLAKE_USERNAME=<username>
SNOWFLAKE_PASSWORD=<password>
SNOWFLAKE_DATABASE=ANALYTICS_DW
SNOWFLAKE_WAREHOUSE=COMPUTE_WH

# Redis (caching)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=<password>
REDIS_CACHE_TTL=3600

# WebSocket
WEBSOCKET_ENABLED=true
WEBSOCKET_PORT=8081
```

### Verification
```bash
# Check JAR was created
ls -lh target/analytics-api-service-1.0.0.jar

# Verify Docker image
docker images | grep analytics-api-service

# Test query engine
mvn test -Dtest=QueryEngineTest
```

---

## Unit 5: Dashboard Frontend

### Location
```
dashboard-frontend/
```

### Build Commands
```bash
cd dashboard-frontend

# Install dependencies
npm install

# Run linting
npm run lint

# Run type checking
npm run type-check

# Build for development
npm run build:dev

# Build for production
npm run build

# Build Docker image
docker build -t dashboard-frontend:1.0.0 .
```

### Build Output
- **Build Directory**: `dist/`
- **Docker Image**: `dashboard-frontend:1.0.0`

### Configuration Files
- `.env.development` - Development environment variables
- `.env.production` - Production environment variables
- `vite.config.ts` - Vite build configuration
- `tsconfig.json` - TypeScript configuration

### Environment Variables Required
```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:8080
VITE_WEBSOCKET_URL=ws://localhost:8081

# Feature Flags
VITE_ENABLE_REAL_TIME=true
VITE_ENABLE_EXPORT=true
VITE_ENABLE_ADVANCED_FILTERS=true

# Analytics
VITE_ANALYTICS_ENABLED=false
```

### Verification
```bash
# Check build output
ls -lh dist/

# Verify assets were generated
ls -lh dist/assets/

# Check bundle size
npm run analyze

# Verify Docker image
docker images | grep dashboard-frontend
```

---

## Multi-Service Build

### Build All Services
```bash
#!/bin/bash
# build-all.sh

set -e

echo "Building all services..."

# Backend services
services=("monitoring-audit-service" "storage-management-service" "data-ingestion-service" "data-processing-pipeline" "analytics-api-service")

for service in "${services[@]}"; do
  echo "Building $service..."
  cd $service
  mvn clean package -DskipTests
  docker build -t $service:1.0.0 .
  cd ..
  echo "$service build complete!"
done

# Frontend
echo "Building dashboard-frontend..."
cd dashboard-frontend
npm install
npm run build
docker build -t dashboard-frontend:1.0.0 .
cd ..
echo "dashboard-frontend build complete!"

echo "All services built successfully!"
```

### Parallel Build (faster)
```bash
#!/bin/bash
# build-all-parallel.sh

set -e

echo "Building all services in parallel..."

# Backend services
(cd monitoring-audit-service && mvn clean package -DskipTests && docker build -t monitoring-audit-service:1.0.0 .) &
(cd storage-management-service && mvn clean package -DskipTests && docker build -t storage-management-service:1.0.0 .) &
(cd data-ingestion-service && mvn clean package -DskipTests && docker build -t data-ingestion-service:1.0.0 .) &
(cd data-processing-pipeline && mvn clean package -DskipTests && docker build -t data-processing-pipeline:1.0.0 .) &
(cd analytics-api-service && mvn clean package -DskipTests && docker build -t analytics-api-service:1.0.0 .) &

# Frontend
(cd dashboard-frontend && npm install && npm run build && docker build -t dashboard-frontend:1.0.0 .) &

# Wait for all builds to complete
wait

echo "All services built successfully!"
```

---

## Docker Compose Build

### Build with Docker Compose
```bash
# Build all services
docker-compose build

# Build specific service
docker-compose build monitoring-audit-service

# Build with no cache
docker-compose build --no-cache

# Build and start
docker-compose up --build
```

---

## CI/CD Pipeline Build

### GitHub Actions Example
```yaml
name: Build All Services

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-backend:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service:
          - monitoring-audit-service
          - storage-management-service
          - data-ingestion-service
          - data-processing-pipeline
          - analytics-api-service
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      
      - name: Build with Maven
        run: |
          cd ${{ matrix.service }}
          mvn clean package
      
      - name: Build Docker image
        run: |
          cd ${{ matrix.service }}
          docker build -t ${{ matrix.service }}:${{ github.sha }} .
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: ${{ matrix.service }}-jar
          path: ${{ matrix.service }}/target/*.jar

  build-frontend:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
          cache-dependency-path: dashboard-frontend/package-lock.json
      
      - name: Install dependencies
        run: |
          cd dashboard-frontend
          npm ci
      
      - name: Build
        run: |
          cd dashboard-frontend
          npm run build
      
      - name: Build Docker image
        run: |
          cd dashboard-frontend
          docker build -t dashboard-frontend:${{ github.sha }} .
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: dashboard-frontend-dist
          path: dashboard-frontend/dist/
```

---

## Build Troubleshooting

### Common Issues

#### Maven Build Failures
```bash
# Clear Maven cache
rm -rf ~/.m2/repository

# Update dependencies
mvn clean install -U

# Skip tests temporarily
mvn package -DskipTests

# Increase memory
export MAVEN_OPTS="-Xmx2048m -XX:MaxPermSize=512m"
```

#### npm Build Failures
```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Use legacy peer deps
npm install --legacy-peer-deps
```

#### Docker Build Failures
```bash
# Clear Docker cache
docker system prune -a

# Build with no cache
docker build --no-cache -t service:1.0.0 .

# Check Docker disk space
docker system df
```

#### Out of Memory
```bash
# Increase Docker memory (Docker Desktop)
# Settings > Resources > Memory > 8GB+

# Increase Maven memory
export MAVEN_OPTS="-Xmx4096m"

# Increase Node memory
export NODE_OPTIONS="--max-old-space-size=4096"
```

---

## Build Verification Checklist

### Backend Services
- [ ] All Maven builds complete without errors
- [ ] All unit tests pass
- [ ] JAR files created in target/ directories
- [ ] Docker images built successfully
- [ ] No security vulnerabilities (mvn dependency-check)
- [ ] Code coverage meets threshold (80%+)

### Frontend
- [ ] npm install completes without errors
- [ ] TypeScript compilation succeeds
- [ ] Linting passes (no errors)
- [ ] Build completes successfully
- [ ] dist/ directory contains all assets
- [ ] Docker image built successfully
- [ ] Bundle size within acceptable limits

### Integration
- [ ] All services can be built together
- [ ] Docker Compose build succeeds
- [ ] No port conflicts
- [ ] Environment variables documented
- [ ] Configuration files present

---

## Next Steps

After successful builds:
1. Proceed to Unit Test Instructions
2. Run Integration Tests
3. Perform Performance Tests
4. Execute Security Scans
5. Deploy to Development Environment

---

## Document Status

**Status**: Complete  
**Date**: 2026-03-04  
**Version**: 1.0.0  
**Next**: unit-test-instructions.md

---
