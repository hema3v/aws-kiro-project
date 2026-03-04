# Unit 6: Monitoring & Audit Service
## Complete Design Document

**Version**: 1.0  
**Date**: 2026-03-04  
**Status**: Complete

---

## 1. FUNCTIONAL DESIGN

### 1.1 Service Overview

The Monitoring & Audit Service provides observability, logging, metrics collection, distributed tracing, data lineage tracking, and audit trail management for the entire platform.

### 1.2 Core Capabilities

**Centralized Logging**
- Collect logs from all services
- Structured JSON logging
- Log aggregation with ELK Stack
- Log retention policies (30/90/365 days)

**Metrics Collection**
- System metrics (CPU, memory, disk)
- Application metrics (requests, errors, latency)
- Business metrics (data processed, quality scores)
- Custom metrics via Micrometer

**Distributed Tracing**
- Request correlation across services
- Performance bottleneck identification
- Spring Cloud Sleuth + Zipkin integration

**Data Lineage Tracking**
- Complete data journey from source to destination
- Transformation history
- Storage locations
- Queryable lineage API

**Audit Trail Management**
- User action logging
- System event tracking
- Compliance reporting
- 2-year retention

**Health Monitoring**
- Service health checks
- Database connectivity
- Kafka consumer lag
- Alert management

### 1.3 Key Components

```
MonitoringAuditService
├── logging/
│   ├── LogAggregationService
│   └── LogRetentionService
├── metrics/
│   ├── MetricsCollectorService
│   └── MetricsExportService
├── tracing/
│   ├── TracingService
│   └── SpanCollectorService
├── lineage/
│   ├── LineageTrackingService
│   └── LineageQueryService
├── audit/
│   ├── AuditLogService
│   └── AuditQueryService
└── health/
    ├── HealthCheckService
    └── AlertService
```

---

## 2. NFR REQUIREMENTS

### 2.1 Performance
- **NFR-P-001**: Process 10,000 log entries/second
- **NFR-P-002**: Metrics collection latency < 100ms
- **NFR-P-003**: Lineage tracking latency < 100ms (async)
- **NFR-P-004**: Audit log write latency < 50ms

### 2.2 Scalability
- **NFR-S-001**: Support 100+ services sending logs/metrics
- **NFR-S-002**: Handle 1TB+ of log data
- **NFR-S-003**: Store 100M+ lineage records

### 2.3 Reliability
- **NFR-R-001**: Service uptime 99.9%
- **NFR-R-002**: Zero data loss for audit logs
- **NFR-R-003**: Automatic failover for log collection

### 2.4 Security
- **NFR-SEC-001**: Encrypt audit logs at rest
- **NFR-SEC-002**: Role-based access to logs and metrics
- **NFR-SEC-003**: Tamper-proof audit trails

---

## 3. NFR DESIGN

### 3.1 High-Throughput Log Processing

**Async Logging**
```java
@Async("loggingExecutor")
public CompletableFuture<Void> logEvent(LogEvent event) {
    return CompletableFuture.runAsync(() -> {
        logRepository.save(event);
    });
}
```

**Batch Processing**
```java
@Scheduled(fixedDelay = 5000)
public void flushLogBatch() {
    List<LogEvent> batch = logBuffer.drain();
    if (!batch.isEmpty()) {
        logRepository.saveAll(batch);
    }
}
```

### 3.2 Metrics Export

**Prometheus Integration**
```java
@Configuration
public class MetricsConfig {
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "monitoring-audit-service");
    }
}
```

### 3.3 Distributed Tracing

**Sleuth Configuration**
```yaml
spring:
  sleuth:
    sampler:
      probability: 0.1  # 10% sampling in production
  zipkin:
    base-url: http://zipkin:9411
```

---

## 4. INFRASTRUCTURE DESIGN

### 4.1 Compute Resources

**Production**
- Instances: 3-5 (auto-scaling)
- CPU: 2 cores per instance
- Memory: 4GB per instance

### 4.2 Storage

**MongoDB (Lineage & Audit)**
- Deployment: Replica Set (3 nodes)
- Storage: 2TB SSD
- Retention: 2 years for audit, 1 year for lineage

**Elasticsearch (Logs)**
- Nodes: 3
- Storage: 1TB SSD per node
- Retention: 30 days (hot), 90 days (warm), 365 days (cold)

**Prometheus (Metrics)**
- Storage: 200GB SSD
- Retention: 15 days (raw), 90 days (aggregated)

### 4.3 Infrastructure Components

**ELK Stack**
- Elasticsearch: 3 nodes
- Logstash: 2 instances
- Kibana: 1 instance
- Filebeat: On all service instances

**Monitoring Stack**
- Prometheus: 1 instance
- Grafana: 1 instance
- Alertmanager: 1 instance

**Tracing**
- Zipkin: 1 instance
- Storage: Elasticsearch

---

## 5. CODE GENERATION

### 5.1 Project Structure

```
monitoring-audit-service/
├── src/main/java/com/analytics/monitoring/
│   ├── MonitoringAuditApplication.java
│   ├── config/
│   │   ├── MongoConfig.java
│   │   ├── MetricsConfig.java
│   │   └── SecurityConfig.java
│   ├── controller/
│   │   ├── LineageController.java
│   │   ├── AuditController.java
│   │   ├── MetricsController.java
│   │   └── HealthController.java
│   ├── service/
│   │   ├── LineageTrackingService.java
│   │   ├── AuditLogService.java
│   │   ├── MetricsCollectorService.java
│   │   └── HealthCheckService.java
│   ├── model/
│   │   ├── LineageRecord.java
│   │   ├── AuditLog.java
│   │   └── HealthStatus.java
│   └── repository/
│       ├── LineageRepository.java
│       └── AuditLogRepository.java
├── src/main/resources/
│   ├── application.yml
│   └── logback-spring.xml
└── pom.xml
```

### 5.2 Key Implementation - Lineage Tracking

```java
@Service
@RequiredArgsConstructor
public class LineageTrackingService {
    
    private final LineageRepository lineageRepository;
    
    @Async
    public CompletableFuture<Void> trackEvent(LineageEvent event) {
        return CompletableFuture.runAsync(() -> {
            LineageRecord record = lineageRepository
                .findByRecordId(event.getRecordId())
                .orElse(new LineageRecord(event.getRecordId()));
            
            record.addEvent(event);
            lineageRepository.save(record);
        });
    }
    
    public LineageRecord getLineage(UUID recordId) {
        return lineageRepository.findByRecordId(recordId)
            .orElseThrow(() -> new LineageNotFoundException(recordId));
    }
}
```

### 5.3 Key Implementation - Audit Logging

```java
@Service
@RequiredArgsConstructor
public class AuditLogService {
    
    private final AuditLogRepository auditLogRepository;
    
    public void logUserAction(String userId, String action, 
                              String resource, Map<String, Object> details) {
        AuditLog log = AuditLog.builder()
            .timestamp(Instant.now())
            .userId(userId)
            .action(action)
            .resource(resource)
            .details(details)
            .ipAddress(getCurrentIpAddress())
            .build();
        
        auditLogRepository.save(log);
    }
    
    public Page<AuditLog> queryAuditLogs(AuditLogQuery query, Pageable pageable) {
        return auditLogRepository.findByQuery(query, pageable);
    }
}
```

### 5.4 API Endpoints

**Lineage API**
```
GET  /api/v1/monitoring/lineage/{recordId}
POST /api/v1/monitoring/lineage (internal)
```

**Audit API**
```
GET  /api/v1/monitoring/audit
POST /api/v1/monitoring/audit (internal)
```

**Health API**
```
GET  /api/v1/monitoring/health
GET  /api/v1/monitoring/metrics
```

### 5.5 Configuration

```yaml
spring:
  application:
    name: monitoring-audit-service
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/monitoring
  
  sleuth:
    sampler:
      probability: ${TRACING_SAMPLE_RATE:0.1}
  
  zipkin:
    base-url: ${ZIPKIN_URL:http://localhost:9411}

monitoring:
  lineage:
    retention-days: 365
  audit:
    retention-days: 730
  metrics:
    export:
      prometheus:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

### 5.6 Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-sleuth-zipkin</artifactId>
    </dependency>
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>
</dependencies>
```

---

## 6. DEPLOYMENT

### 6.1 Docker Configuration

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/monitoring-audit-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 6.2 Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring-audit-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: monitoring-audit
  template:
    metadata:
      labels:
        app: monitoring-audit
    spec:
      containers:
      - name: monitoring-audit
        image: monitoring-audit-service:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
```

---

## 7. SUCCESS CRITERIA

- [ ] Capture 100% of events from all services
- [ ] Process 10,000 log entries/second
- [ ] Lineage tracking latency < 100ms
- [ ] Audit log write latency < 50ms
- [ ] Zero data loss for audit logs
- [ ] Complete API documentation
- [ ] 80%+ test coverage
- [ ] Grafana dashboards configured
- [ ] Alert rules defined

---

## Document Status

**Status**: Complete  
**All Stages**: Functional Design, NFR Requirements, NFR Design, Infrastructure Design, Code Generation  
**Next**: Unit 1 (Data Ingestion Service)

---
