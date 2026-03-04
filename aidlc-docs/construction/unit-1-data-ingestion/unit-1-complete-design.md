# Unit 1: Data Ingestion Service
## Complete Design Document

**Version**: 1.0  
**Date**: 2026-03-04  
**Status**: Complete

---

## 1. FUNCTIONAL DESIGN

### 1.1 Service Overview

The Data Ingestion Service collects data from diverse external sources (databases, APIs, files, streaming) and publishes it to Kafka for processing. It handles connection management, scheduling, error handling, and initial validation.

### 1.2 Core Capabilities

**Database Connectivity**
- PostgreSQL, MySQL, MongoDB, Oracle, SQL Server
- Full and incremental extraction
- Connection pooling and retry logic
- Scheduled polling

**API Integration**
- REST APIs (OAuth 2.0, API Key, Basic, JWT)
- SOAP/XML web services
- GraphQL queries
- Rate limiting and throttling
- Webhook receivers

**File Processing**
- CSV, JSON, XML, Excel (XLS/XLSX)
- File upload via REST API
- FTP/SFTP monitoring
- Cloud storage (S3, Azure Blob, GCS)
- Streaming for large files (>100MB)

**Streaming Integration**
- Apache Kafka consumer
- RabbitMQ consumer
- AWS SQS consumer
- Azure Service Bus consumer
- Backpressure handling

**Job Management**
- Cron-based scheduling
- Job status tracking
- Manual triggering
- Job history and logs

### 1.3 Key Components

```
DataIngestionService
├── connectors/
│   ├── database/
│   │   ├── PostgreSQLConnector
│   │   ├── MySQLConnector
│   │   ├── MongoDBConnector
│   │   └── OracleConnector
│   ├── api/
│   │   ├── RestApiClient
│   │   ├── SoapClient
│   │   └── GraphQLClient
│   ├── file/
│   │   ├── CsvProcessor
│   │   ├── JsonProcessor
│   │   ├── XmlProcessor
│   │   └── ExcelProcessor
│   └── stream/
│       ├── KafkaStreamConsumer
│       ├── RabbitMQConsumer
│       └── SQSConsumer
├── services/
│   ├── IngestionJobService
│   ├── ConnectionPoolManager
│   ├── SchedulerService
│   └── ValidationService
├── controllers/
│   ├── FileUploadController
│   ├── ConnectionConfigController
│   ├── WebhookController
│   └── JobManagementController
└── publishers/
    └── KafkaEventPublisher
```

### 1.4 Data Flow

```
External Source → Connector → Validation → Kafka (raw-data-ingested) → Processing Pipeline
```

### 1.5 Business Rules

**BR-001: Connection Retry**
- Retry failed connections up to 3 times with exponential backoff
- Alert after 3 failures

**BR-002: Data Validation**
- Validate file format before processing
- Check API response structure
- Verify database query results

**BR-003: Rate Limiting**
- Respect API rate limits (configurable per source)
- Implement backoff when rate limit hit

**BR-004: Job Scheduling**
- Support cron expressions for flexible scheduling
- Prevent overlapping job executions
- Track last successful run

**BR-005: Large File Handling**
- Stream files >100MB in chunks
- Process in batches to avoid memory issues

---

## 2. NFR REQUIREMENTS

### 2.1 Performance
- **NFR-P-001**: Process 1M records/hour
- **NFR-P-002**: File upload throughput 100MB/sec
- **NFR-P-003**: API call latency < 5 seconds
- **NFR-P-004**: Database extraction 10,000 records/minute

### 2.2 Scalability
- **NFR-S-001**: Support 100+ data sources
- **NFR-S-002**: Handle 1000+ concurrent API calls
- **NFR-S-003**: Process files up to 10GB

### 2.3 Reliability
- **NFR-R-001**: Job success rate > 99%
- **NFR-R-002**: Automatic retry for transient failures
- **NFR-R-003**: Zero data loss during ingestion

### 2.4 Security
- **NFR-SEC-001**: Encrypt credentials at rest (AES-256)
- **NFR-SEC-002**: Use TLS for all external connections
- **NFR-SEC-003**: Validate webhook signatures

---

## 3. NFR DESIGN

### 3.1 High-Throughput Processing

**Parallel Processing**
```java
@Configuration
public class AsyncConfig {
    @Bean(name = "ingestionExecutor")
    public Executor ingestionExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("ingestion-");
        executor.initialize();
        return executor;
    }
}

@Service
public class IngestionJobService {
    @Async("ingestionExecutor")
    public CompletableFuture<JobResult> executeJob(UUID jobId) {
        // Execute ingestion job asynchronously
    }
}
```

**Batch Publishing to Kafka**
```java
@Service
public class KafkaEventPublisher {
    private final KafkaTemplate<String, RawDataEvent> kafkaTemplate;
    private final List<RawDataEvent> buffer = new CopyOnWriteArrayList<>();
    
    @Scheduled(fixedDelay = 5000)
    public void flushBatch() {
        if (!buffer.isEmpty()) {
            List<RawDataEvent> batch = new ArrayList<>(buffer);
            buffer.clear();
            
            batch.forEach(event -> 
                kafkaTemplate.send("raw-data-ingested", event.getRecordId(), event)
            );
        }
    }
}
```

### 3.2 Connection Management

**Connection Pooling**
```java
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        return new HikariDataSource(config);
    }
}
```

**Retry Logic**
```java
@Service
public class DatabaseConnector {
    @Retryable(
        value = {TransientDataAccessException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000, multiplier = 2)
    )
    public List<Record> extractData(DataSource source) {
        // Extract data with automatic retry
    }
}
```

### 3.3 Security Implementation

**Credential Encryption**
```java
@Service
public class CredentialService {
    @Value("${encryption.key}")
    private String encryptionKey;
    
    public String encryptCredential(String plainText) {
        // AES-256 encryption
        SecretKeySpec key = new SecretKeySpec(encryptionKey.getBytes(), "AES");
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key);
        byte[] encrypted = cipher.doFinal(plainText.getBytes());
        return Base64.getEncoder().encodeToString(encrypted);
    }
}
```

**Webhook Signature Verification**
```java
@RestController
public class WebhookController {
    public ResponseEntity<Void> receiveWebhook(
            @RequestHeader("X-Webhook-Signature") String signature,
            @RequestBody String payload) {
        
        if (!verifySignature(signature, payload)) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
        }
        
        processWebhook(payload);
        return ResponseEntity.ok().build();
    }
}
```

---

## 4. INFRASTRUCTURE DESIGN

### 4.1 Compute Resources

**Production**
- Instances: 3-5 (auto-scaling)
- CPU: 4 cores per instance
- Memory: 8GB per instance

### 4.2 Storage

**PostgreSQL (Configuration)**
- Instance: db.t3.medium
- Storage: 100GB SSD
- Purpose: Store data source configs, job history

**MongoDB (Raw Data Buffer)**
- Deployment: Replica Set (3 nodes)
- Storage: 500GB SSD per node
- Purpose: Temporary storage for raw data
- TTL: 30 days

### 4.3 Network

**Outbound Connections**
- External APIs: Port 443 (HTTPS)
- Databases: Ports 5432, 3306, 27017, 1521
- FTP/SFTP: Ports 21, 22
- Kafka: Port 9092

**Inbound Connections**
- REST API: Port 8080
- Webhooks: Port 8080

### 4.4 Infrastructure Cost

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| Service Instances | 3-5 avg 4 | $200/month | $800 |
| PostgreSQL | 1 | $100/month | $100 |
| MongoDB | 3 nodes | $200/month | $600 |
| Load Balancer | 1 | $25/month | $25 |
| Storage | Various | - | $100 |
| **Total** | | | **$1,625/month** |

---

## 5. CODE GENERATION

### 5.1 Project Structure

```
data-ingestion-service/
├── src/main/java/com/analytics/ingestion/
│   ├── DataIngestionApplication.java
│   ├── config/
│   │   ├── DatabaseConfig.java
│   │   ├── KafkaConfig.java
│   │   ├── SecurityConfig.java
│   │   ├── AsyncConfig.java
│   │   └── SchedulerConfig.java
│   ├── connector/
│   │   ├── DatabaseConnector.java (interface)
│   │   ├── PostgreSQLConnector.java
│   │   ├── MySQLConnector.java
│   │   ├── MongoDBConnector.java
│   │   ├── ApiConnector.java
│   │   ├── RestApiClient.java
│   │   ├── FileProcessor.java (interface)
│   │   ├── CsvProcessor.java
│   │   ├── JsonProcessor.java
│   │   └── ExcelProcessor.java
│   ├── controller/
│   │   ├── FileUploadController.java
│   │   ├── DataSourceController.java
│   │   ├── WebhookController.java
│   │   └── JobController.java
│   ├── service/
│   │   ├── IngestionJobService.java
│   │   ├── DataSourceService.java
│   │   ├── SchedulerService.java
│   │   ├── ValidationService.java
│   │   └── CredentialService.java
│   ├── publisher/
│   │   └── KafkaEventPublisher.java
│   ├── model/
│   │   ├── DataSource.java
│   │   ├── IngestionJob.java
│   │   ├── RawDataEvent.java
│   │   └── JobResult.java
│   └── repository/
│       ├── DataSourceRepository.java
│       └── IngestionJobRepository.java
├── src/main/resources/
│   ├── application.yml
│   └── logback-spring.xml
└── pom.xml
```

### 5.2 Key Implementation - Database Connector

```java
@Component
public class PostgreSQLConnector implements DatabaseConnector {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Autowired
    private KafkaEventPublisher publisher;
    
    @Override
    @Retryable(maxAttempts = 3)
    public JobResult extract(DataSource source) {
        String query = source.getExtractionQuery();
        List<Map<String, Object>> results = jdbcTemplate.queryForList(query);
        
        int count = 0;
        for (Map<String, Object> row : results) {
            RawDataEvent event = RawDataEvent.builder()
                .recordId(UUID.randomUUID())
                .sourceId(source.getSourceId())
                .sourceType("database")
                .data(row)
                .timestamp(Instant.now())
                .build();
            
            publisher.publish(event);
            count++;
        }
        
        return JobResult.builder()
            .success(true)
            .recordsProcessed(count)
            .build();
    }
}
```

### 5.3 Key Implementation - File Upload

```java
@RestController
@RequestMapping("/api/v1/ingestion")
public class FileUploadController {
    
    @Autowired
    private FileProcessor fileProcessor;
    
    @Autowired
    private IngestionJobService jobService;
    
    @PostMapping("/files/upload")
    public ResponseEntity<JobResponse> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("sourceType") String sourceType,
            @RequestParam("businessId") UUID businessId) {
        
        // Validate file
        if (file.isEmpty()) {
            return ResponseEntity.badRequest().build();
        }
        
        // Create job
        UUID jobId = jobService.createJob(businessId, sourceType);
        
        // Process file asynchronously
        CompletableFuture.runAsync(() -> {
            try {
                fileProcessor.process(file, jobId, businessId);
            } catch (Exception e) {
                jobService.markFailed(jobId, e.getMessage());
            }
        });
        
        return ResponseEntity.accepted()
            .body(new JobResponse(jobId, "accepted"));
    }
}
```

### 5.4 Key Implementation - Scheduler

```java
@Service
public class SchedulerService {
    
    @Autowired
    private DataSourceRepository dataSourceRepository;
    
    @Autowired
    private IngestionJobService jobService;
    
    @Scheduled(fixedDelay = 60000) // Check every minute
    public void checkScheduledJobs() {
        List<DataSource> sources = dataSourceRepository
            .findByStatusAndScheduleNotNull("active");
        
        for (DataSource source : sources) {
            if (shouldExecute(source)) {
                jobService.executeJob(source);
            }
        }
    }
    
    private boolean shouldExecute(DataSource source) {
        CronExpression cron = CronExpression.parse(source.getSchedule());
        LocalDateTime lastRun = source.getLastSyncAt();
        LocalDateTime nextRun = cron.next(lastRun);
        
        return LocalDateTime.now().isAfter(nextRun);
    }
}
```

### 5.5 API Endpoints

**File Upload**
```
POST /api/v1/ingestion/files/upload
Content-Type: multipart/form-data
```

**Data Source Management**
```
POST   /api/v1/ingestion/connections/database
POST   /api/v1/ingestion/connections/api
GET    /api/v1/ingestion/connections
GET    /api/v1/ingestion/connections/{id}
PUT    /api/v1/ingestion/connections/{id}
DELETE /api/v1/ingestion/connections/{id}
```

**Job Management**
```
GET  /api/v1/ingestion/jobs/{jobId}
GET  /api/v1/ingestion/jobs
POST /api/v1/ingestion/jobs/{jobId}/retry
```

**Webhooks**
```
POST /api/v1/ingestion/webhooks/{webhookId}
```

### 5.6 Configuration

```yaml
spring:
  application:
    name: data-ingestion-service
  
  datasource:
    url: jdbc:postgresql://${POSTGRES_HOST:localhost}:5432/ingestion
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/raw_data
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      batch-size: 16384
      linger-ms: 10
  
  servlet:
    multipart:
      max-file-size: 1GB
      max-request-size: 1GB

ingestion:
  file:
    temp-directory: /tmp/uploads
    allowed-types: csv,json,xml,xlsx
  
  database:
    connection-pool:
      max-size: 20
      min-idle: 5
  
  api:
    retry:
      max-attempts: 3
      backoff-delay: 1000
    timeout: 30000
  
  scheduler:
    enabled: true
    thread-pool-size: 10

encryption:
  key: ${ENCRYPTION_KEY}

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

### 5.7 Maven Dependencies

```xml
<dependencies>
    <!-- Spring Boot -->
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
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    
    <!-- File Processing -->
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.3</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-xml</artifactId>
    </dependency>
    <dependency>
        <groupId>com.opencsv</groupId>
        <artifactId>opencsv</artifactId>
        <version>5.7.1</version>
    </dependency>
    
    <!-- Apache Camel (Integration) -->
    <dependency>
        <groupId>org.apache.camel.springboot</groupId>
        <artifactId>camel-spring-boot-starter</artifactId>
        <version>4.0.0</version>
    </dependency>
    
    <!-- Scheduler -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
    
    <!-- Retry -->
    <dependency>
        <groupId>org.springframework.retry</groupId>
        <artifactId>spring-retry</artifactId>
    </dependency>
    
    <!-- HTTP Client -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
</dependencies>
```

---

## 6. DEPLOYMENT

### 6.1 Docker Configuration

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/data-ingestion-service-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

### 6.2 Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-ingestion-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: data-ingestion
  template:
    metadata:
      labels:
        app: data-ingestion
    spec:
      containers:
      - name: data-ingestion
        image: data-ingestion-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka:9092"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2000m"
          limits:
            memory: "8Gi"
            cpu: "4000m"
```

---

## 7. SUCCESS CRITERIA

- [ ] Successfully connect to all supported data source types
- [ ] Process 1M records/hour
- [ ] Job success rate > 99%
- [ ] Handle files up to 10GB
- [ ] API response time < 5 seconds
- [ ] Zero data loss during ingestion
- [ ] Complete API documentation (OpenAPI)
- [ ] 80%+ test coverage
- [ ] All credentials encrypted
- [ ] Webhook signature verification working

---

## Document Status

**Status**: Complete  
**All Stages**: Functional Design, NFR Requirements, NFR Design, Infrastructure Design, Code Generation  
**Next**: Unit 2 (Data Processing Pipeline)

---
