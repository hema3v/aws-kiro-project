# Code Generation Plan
## Unit 3: Storage Management Service

**Version**: 1.0  
**Date**: 2026-03-04

---

## 1. Code Generation Overview

This document outlines the complete code generation plan for Unit 3 (Storage Management Service), including project structure, key components, and implementation checklist.

---

## 2. Project Structure

```
storage-management-service/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/analytics/storage/
│   │   │       ├── StorageManagementApplication.java
│   │   │       ├── config/
│   │   │       │   ├── DatabaseConfig.java
│   │   │       │   ├── KafkaConfig.java
│   │   │       │   ├── RedisConfig.java
│   │   │       │   ├── SecurityConfig.java
│   │   │       │   ├── AsyncConfig.java
│   │   │       │   └── Resilience4jConfig.java
│   │   │       ├── controller/
│   │   │       │   ├── StorageController.java
│   │   │       │   └── SchemaController.java
│   │   │       ├── service/
│   │   │       │   ├── StorageService.java
│   │   │       │   ├── RoutingEngine.java
│   │   │       │   ├── ValidationEngine.java
│   │   │       │   ├── WriteCoordinator.java
│   │   │       │   ├── BatchManager.java
│   │   │       │   ├── SchemaService.java
│   │   │       │   ├── LineageTracker.java
│   │   │       │   └── MetricsService.java
│   │   │       ├── writer/
│   │   │       │   ├── DatabaseWriter.java (interface)
│   │   │       │   ├── PostgreSQLWriter.java
│   │   │       │   ├── MongoDBWriter.java
│   │   │       │   ├── TimescaleDBWriter.java
│   │   │       │   └── SnowflakeWriter.java
│   │   │       ├── consumer/
│   │   │       │   └── StorageKafkaConsumer.java
│   │   │       ├── model/
│   │   │       │   ├── ProcessedDataRecord.java
│   │   │       │   ├── DatabaseTarget.java
│   │   │       │   ├── RoutingRule.java
│   │   │       │   ├── Schema.java
│   │   │       │   ├── WriteResult.java
│   │   │       │   └── LineageRecord.java
│   │   │       ├── repository/
│   │   │       │   ├── RoutingRuleRepository.java
│   │   │       │   ├── SchemaRepository.java
│   │   │       │   └── LineageRepository.java
│   │   │       ├── exception/
│   │   │       │   ├── StorageException.java
│   │   │       │   ├── RoutingException.java
│   │   │       │   ├── ValidationException.java
│   │   │       │   └── WriteException.java
│   │   │       └── health/
│   │   │           ├── DatabaseHealthIndicator.java
│   │   │           └── KafkaHealthIndicator.java
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-staging.yml
│   │       ├── application-prod.yml
│   │       └── logback-spring.xml
│   └── test/
│       └── java/
│           └── com/analytics/storage/
│               ├── service/
│               │   ├── RoutingEngineTest.java
│               │   ├── ValidationEngineTest.java
│               │   └── BatchManagerTest.java
│               ├── writer/
│               │   ├── PostgreSQLWriterTest.java
│               │   └── MongoDBWriterTest.java
│               └── integration/
│                   ├── StorageIntegrationTest.java
│                   └── KafkaIntegrationTest.java
├── pom.xml
├── Dockerfile
├── docker-compose.yml
├── README.md
└── .gitignore
```

---

## 3. Implementation Checklist

### Phase 1: Project Setup
- [ ] Initialize Spring Boot project (Maven/Gradle)
- [ ] Configure dependencies (pom.xml)
- [ ] Setup application properties
- [ ] Configure logging (logback-spring.xml)
- [ ] Create Docker configuration
- [ ] Setup Git repository

### Phase 2: Core Models
- [ ] ProcessedDataRecord.java
- [ ] DatabaseTarget.java
- [ ] RoutingRule.java
- [ ] Schema.java
- [ ] WriteResult.java
- [ ] LineageRecord.java

### Phase 3: Configuration
- [ ] DatabaseConfig.java (PostgreSQL, MongoDB, TimescaleDB)
- [ ] KafkaConfig.java (consumer configuration)
- [ ] RedisConfig.java (caching)
- [ ] SecurityConfig.java (JWT authentication)
- [ ] AsyncConfig.java (thread pools)
- [ ] Resilience4jConfig.java (circuit breakers, retry)

### Phase 4: Core Services
- [ ] RoutingEngine.java (routing logic)
- [ ] ValidationEngine.java (schema validation)
- [ ] WriteCoordinator.java (write orchestration)
- [ ] BatchManager.java (batch accumulation)
- [ ] SchemaService.java (schema management)
- [ ] LineageTracker.java (lineage tracking)
- [ ] MetricsService.java (Prometheus metrics)

### Phase 5: Database Writers
- [ ] DatabaseWriter.java (interface)
- [ ] PostgreSQLWriter.java
- [ ] MongoDBWriter.java
- [ ] TimescaleDBWriter.java
- [ ] SnowflakeWriter.java

### Phase 6: Kafka Integration
- [ ] StorageKafkaConsumer.java
- [ ] Message deserialization
- [ ] Error handling and DLQ

### Phase 7: REST APIs
- [ ] StorageController.java (storage stats, manual write)
- [ ] SchemaController.java (schema CRUD)
- [ ] Exception handlers
- [ ] OpenAPI documentation

### Phase 8: Repositories
- [ ] RoutingRuleRepository.java (JPA)
- [ ] SchemaRepository.java (JPA)
- [ ] LineageRepository.java (MongoDB)

### Phase 9: Health & Monitoring
- [ ] DatabaseHealthIndicator.java
- [ ] KafkaHealthIndicator.java
- [ ] Prometheus metrics endpoints
- [ ] Actuator configuration

### Phase 10: Testing
- [ ] Unit tests (80%+ coverage)
- [ ] Integration tests (Testcontainers)
- [ ] Performance tests
- [ ] End-to-end tests

### Phase 11: Documentation
- [ ] README.md
- [ ] API documentation (Swagger)
- [ ] Deployment guide
- [ ] Configuration guide

### Phase 12: Deployment
- [ ] Dockerfile
- [ ] docker-compose.yml
- [ ] Kubernetes manifests
- [ ] CI/CD pipeline

---

## 4. Key Implementation Files

### 4.1 Main Application Class

**File**: `StorageManagementApplication.java`

```java
package com.analytics.storage;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableKafka
@EnableCaching
@EnableAsync
@EnableScheduling
public class StorageManagementApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(StorageManagementApplication.class, args);
    }
}
```

### 4.2 Routing Engine

**File**: `service/RoutingEngine.java`

```java
package com.analytics.storage.service;

import com.analytics.storage.model.*;
import com.analytics.storage.repository.RoutingRuleRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class RoutingEngine {
    
    private final RoutingRuleRepository routingRuleRepository;
    
    public List<DatabaseTarget> determineTargets(ProcessedDataRecord record) {
        log.debug("Determining routing targets for record: {}", record.getRecordId());
        
        List<DatabaseTarget> targets = new ArrayList<>();
        
        // Step 1: Get default target based on entity type
        DatabaseTarget defaultTarget = getDefaultTarget(record.getEntityType());
        targets.add(defaultTarget);
        
        // Step 2: Apply routing rules
        List<RoutingRule> rules = getApplicableRules(record);
        for (RoutingRule rule : rules) {
            if (evaluateRule(rule, record)) {
                targets.addAll(rule.getTargets());
            }
        }
        
        // Step 3: Remove duplicates
        targets = deduplicateTargets(targets);
        
        // Step 4: Filter by availability
        targets = filterAvailableTargets(targets);
        
        log.info("Routing targets determined: {} for record: {}", 
            targets.stream().map(DatabaseTarget::getType).collect(Collectors.toList()),
            record.getRecordId());
        
        return targets;
    }
    
    @Cacheable(value = "defaultTargets", key = "#entityType")
    private DatabaseTarget getDefaultTarget(String entityType) {
        // Default routing logic
        return switch (entityType.toLowerCase()) {
            case "customer", "transaction", "business", "user" -> 
                DatabaseTarget.builder().type("postgresql").build();
            case "raw_data", "processed_data", "audit_log" -> 
                DatabaseTarget.builder().type("mongodb").build();
            case "metric", "event" -> 
                DatabaseTarget.builder().type("timescaledb").build();
            default -> 
                DatabaseTarget.builder().type("postgresql").build();
        };
    }
    
    @Cacheable(value = "routingRules", key = "#record.entityType")
    private List<RoutingRule> getApplicableRules(ProcessedDataRecord record) {
        return routingRuleRepository.findByEntityTypeAndEnabledTrue(record.getEntityType());
    }
    
    private boolean evaluateRule(RoutingRule rule, ProcessedDataRecord record) {
        // Implement rule evaluation logic
        // This would use a rule engine or expression evaluator
        return true; // Placeholder
    }
    
    private List<DatabaseTarget> deduplicateTargets(List<DatabaseTarget> targets) {
        return targets.stream()
            .distinct()
            .collect(Collectors.toList());
    }
    
    private List<DatabaseTarget> filterAvailableTargets(List<DatabaseTarget> targets) {
        // Check database health and filter unavailable targets
        return targets; // Placeholder
    }
}
```

### 4.3 Batch Manager

**File**: `service/BatchManager.java`

```java
package com.analytics.storage.service;

import com.analytics.storage.model.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;
import java.util.*;
import java.util.concurrent.*;

@Slf4j
@Service
@RequiredArgsConstructor
public class BatchManager {
    
    private final WriteCoordinator writeCoordinator;
    private final MetricsService metricsService;
    
    @Value("${storage.batch.size:1000}")
    private int batchSize;
    
    @Value("${storage.batch.timeout:5000}")
    private long batchTimeout;
    
    private final Map<String, List<ProcessedDataRecord>> batches = new ConcurrentHashMap<>();
    private final Map<String, ScheduledFuture<?>> flushTimers = new ConcurrentHashMap<>();
    private ScheduledExecutorService scheduler;
    
    @PostConstruct
    public void init() {
        scheduler = Executors.newScheduledThreadPool(10);
        log.info("BatchManager initialized with batchSize={}, batchTimeout={}ms", 
            batchSize, batchTimeout);
    }
    
    @PreDestroy
    public void shutdown() {
        log.info("Shutting down BatchManager, flushing all batches");
        batches.keySet().forEach(this::flushBatch);
        scheduler.shutdown();
    }
    
    public void addToBatch(ProcessedDataRecord record, DatabaseTarget target) {
        String batchKey = target.getType() + ":" + record.getEntityType();
        
        batches.computeIfAbsent(batchKey, k -> {
            scheduleFlush(k);
            return new CopyOnWriteArrayList<>();
        }).add(record);
        
        metricsService.recordBatchSize(batchKey, batches.get(batchKey).size());
        
        if (batches.get(batchKey).size() >= batchSize) {
            log.debug("Batch size reached for {}, flushing", batchKey);
            flushBatch(batchKey);
        }
    }
    
    private void scheduleFlush(String batchKey) {
        ScheduledFuture<?> future = scheduler.schedule(
            () -> flushBatch(batchKey),
            batchTimeout,
            TimeUnit.MILLISECONDS
        );
        flushTimers.put(batchKey, future);
    }
    
    private void flushBatch(String batchKey) {
        List<ProcessedDataRecord> batch = batches.remove(batchKey);
        ScheduledFuture<?> timer = flushTimers.remove(batchKey);
        
        if (timer != null) {
            timer.cancel(false);
        }
        
        if (batch != null && !batch.isEmpty()) {
            log.info("Flushing batch: {} with {} records", batchKey, batch.size());
            
            String[] parts = batchKey.split(":");
            DatabaseTarget target = DatabaseTarget.builder()
                .type(parts[0])
                .build();
            
            writeCoordinator.executeBatchWrite(batch, target);
        }
    }
}
```

### 4.4 PostgreSQL Writer

**File**: `writer/PostgreSQLWriter.java`

```java
package com.analytics.storage.writer;

import com.analytics.storage.model.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.jdbc.core.BatchPreparedStatementSetter;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.List;

@Slf4j
@Component("postgresqlWriter")
@RequiredArgsConstructor
public class PostgreSQLWriter implements DatabaseWriter {
    
    private final JdbcTemplate jdbcTemplate;
    
    @Override
    public WriteResult write(ProcessedDataRecord record) {
        try {
            String sql = buildInsertSql(record);
            jdbcTemplate.update(sql, prepareParameters(record));
            
            return WriteResult.builder()
                .success(true)
                .recordId(record.getRecordId())
                .database("postgresql")
                .build();
        } catch (Exception e) {
            log.error("Failed to write record to PostgreSQL: {}", record.getRecordId(), e);
            return WriteResult.builder()
                .success(false)
                .recordId(record.getRecordId())
                .error(e.getMessage())
                .build();
        }
    }
    
    @Override
    public WriteResult batchWrite(List<ProcessedDataRecord> records) {
        try {
            String sql = buildBatchInsertSql(records.get(0));
            
            int[] results = jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
                @Override
                public void setValues(PreparedStatement ps, int i) throws SQLException {
                    ProcessedDataRecord record = records.get(i);
                    setParameters(ps, record);
                }
                
                @Override
                public int getBatchSize() {
                    return records.size();
                }
            });
            
            log.info("Batch write completed: {} records written to PostgreSQL", results.length);
            
            return WriteResult.builder()
                .success(true)
                .recordsWritten(results.length)
                .database("postgresql")
                .build();
        } catch (Exception e) {
            log.error("Failed to batch write to PostgreSQL", e);
            return WriteResult.builder()
                .success(false)
                .error(e.getMessage())
                .build();
        }
    }
    
    private String buildInsertSql(ProcessedDataRecord record) {
        // Build INSERT SQL based on entity type
        return "INSERT INTO " + getTableName(record.getEntityType()) + 
               " (...) VALUES (...) ON CONFLICT (...) DO UPDATE SET ...";
    }
    
    private String buildBatchInsertSql(ProcessedDataRecord record) {
        return buildInsertSql(record);
    }
    
    private Object[] prepareParameters(ProcessedDataRecord record) {
        // Extract parameters from record
        return new Object[]{};
    }
    
    private void setParameters(PreparedStatement ps, ProcessedDataRecord record) throws SQLException {
        // Set parameters for batch insert
    }
    
    private String getTableName(String entityType) {
        return entityType.toLowerCase() + "s";
    }
}
```

---

## 5. Maven Dependencies (pom.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <groupId>com.analytics</groupId>
    <artifactId>storage-management-service</artifactId>
    <version>1.0.0</version>
    <name>Storage Management Service</name>
    
    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-mongodb</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
        
        <!-- Database Drivers -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>net.snowflake</groupId>
            <artifactId>snowflake-jdbc</artifactId>
            <version>3.14.3</version>
        </dependency>
        
        <!-- Connection Pool -->
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>
        
        <!-- Resilience4j -->
        <dependency>
            <groupId>io.github.resilience4j</groupId>
            <artifactId>resilience4j-spring-boot3</artifactId>
            <version>2.1.0</version>
        </dependency>
        
        <!-- Micrometer (Metrics) -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        
        <!-- OpenAPI Documentation -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
            <version>2.2.0</version>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>testcontainers</artifactId>
            <version>1.19.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <version>1.19.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>mongodb</artifactId>
            <version>1.19.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>kafka</artifactId>
            <version>1.19.3</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 6. Configuration Files

### 6.1 application.yml

```yaml
spring:
  application:
    name: storage-management-service
  
  datasource:
    url: jdbc:postgresql://${POSTGRES_HOST:localhost}:5432/${POSTGRES_DB:analytics}
    username: ${POSTGRES_USER:postgres}
    password: ${POSTGRES_PASSWORD:postgres}
    hikari:
      maximum-pool-size: 50
      minimum-idle: 10
      connection-timeout: 30000
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/${MONGO_DB:analytics}
      username: ${MONGO_USER:}
      password: ${MONGO_PASSWORD:}
    
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASSWORD:}
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: storage-management
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 500
    listener:
      ack-mode: manual
      concurrency: 3

storage:
  routing:
    rules-refresh-interval: 60000
  batch:
    size: 1000
    timeout: 5000
  retry:
    max-attempts: 3
    backoff:
      initial-delay: 1000
      multiplier: 2
      max-delay: 10000

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

logging:
  level:
    com.analytics.storage: INFO
    org.springframework.kafka: WARN
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

---

## Document Status

**Status**: Complete  
**Next**: Unit 3 Complete - Move to Unit 6

---
