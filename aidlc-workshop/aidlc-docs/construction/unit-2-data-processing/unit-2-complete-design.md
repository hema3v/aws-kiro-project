# Unit 2: Data Processing Pipeline
## Complete Design Document

**Version**: 1.0  
**Date**: 2026-03-04  
**Status**: Complete

---

## 1. FUNCTIONAL DESIGN

### 1.1 Service Overview

The Data Processing Pipeline validates, cleanses, and transforms raw data from the ingestion service before routing it to storage. It ensures data quality, applies business rules, and maintains data integrity throughout the transformation process.

### 1.2 Core Capabilities

**Schema Validation**
- Validate against predefined schemas
- Data type checking
- Required field validation
- Format validation (email, phone, date)

**Data Quality Checks**
- Completeness scoring
- Accuracy validation
- Consistency checks
- Timeliness verification
- Quality metrics calculation

**Data Cleansing**
- Standardization (formats, units, naming)
- Deduplication
- Outlier detection and handling
- Null value handling
- Data normalization

**Business Rule Application**
- Configurable rule engine
- Conditional transformations
- Data enrichment
- Cross-field validation

**Data Transformation**
- Field mapping and renaming
- Data type conversions
- Aggregation and summarization
- Calculated fields
- Multi-source data merging

**Error Handling**
- Dead letter queue for failed records
- Error classification
- Retry logic for transient failures
- Detailed error logging

### 1.3 Key Components

```
DataProcessingPipeline
├── validation/
│   ├── SchemaValidator
│   ├── DataTypeValidator
│   ├── BusinessRuleValidator
│   └── QualityValidator
├── cleansing/
│   ├── StandardizationService
│   ├── DeduplicationService
│   ├── NormalizationService
│   └── OutlierDetectionService
├── transformation/
│   ├── FieldMapper
│   ├── TypeConverter
│   ├── AggregationService
│   └── EnrichmentService
├── rules/
│   ├── RuleEngine
│   ├── RuleRepository
│   └── RuleEvaluator
├── quality/
│   ├── QualityMetricsCalculator
│   └── QualityReporter
└── consumer/
    └── KafkaDataConsumer
```

### 1.4 Data Flow

```
Kafka (raw-data-ingested)
    ↓
Consume & Deserialize
    ↓
Schema Validation
    ↓
Data Quality Checks
    ↓
Data Cleansing
    ↓
Business Rule Application
    ↓
Data Transformation
    ↓
Quality Scoring
    ↓
Kafka (data-ready-for-storage) OR Dead Letter Queue
```

### 1.5 Business Rules

**BR-001: Validation Priority**
- Schema validation first (fail fast)
- Business rules second
- Quality checks last

**BR-002: Quality Thresholds**
- Completeness > 90% to pass
- Accuracy > 95% to pass
- Overall quality score > 85% to pass

**BR-003: Deduplication Strategy**
- Match on primary key + business key
- Keep most recent record
- Log duplicate detection

**BR-004: Error Handling**
- Validation errors → reject immediately
- Transient errors → retry 3 times
- Persistent errors → dead letter queue

**BR-005: Data Enrichment**
- Lookup reference data when available
- Add calculated fields
- Standardize formats

---

## 2. NFR REQUIREMENTS

### 2.1 Performance
- **NFR-P-001**: Process 1M records/hour (same as ingestion)
- **NFR-P-002**: Validation latency < 50ms per record
- **NFR-P-003**: Transformation latency < 100ms per record
- **NFR-P-004**: End-to-end processing < 5 seconds (p95)

### 2.2 Scalability
- **NFR-S-001**: Horizontal scaling to 10 instances
- **NFR-S-002**: Handle burst traffic (2x normal load)
- **NFR-S-003**: Process records up to 10MB

### 2.3 Reliability
- **NFR-R-001**: Zero data loss during processing
- **NFR-R-002**: Exactly-once processing semantics
- **NFR-R-003**: Automatic recovery from failures

### 2.4 Data Quality
- **NFR-DQ-001**: Detect 100% of schema violations
- **NFR-DQ-002**: Identify 95%+ of duplicates
- **NFR-DQ-003**: Calculate quality scores for all records

---

## 3. NFR DESIGN

### 3.1 High-Throughput Processing

**Parallel Kafka Consumers**
```java
@Configuration
public class KafkaConfig {
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, RawDataEvent> 
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, RawDataEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(10); // 10 parallel consumers
        factory.getContainerProperties().setAckMode(AckMode.MANUAL);
        return factory;
    }
}
```

**Async Processing**
```java
@Service
public class ProcessingPipeline {
    
    @Async("processingExecutor")
    public CompletableFuture<ProcessedData> process(RawDataEvent event) {
        return CompletableFuture.supplyAsync(() -> {
            // Validate
            ValidationResult validation = validator.validate(event);
            if (!validation.isValid()) {
                throw new ValidationException(validation.getErrors());
            }
            
            // Cleanse
            CleanedData cleaned = cleanser.cleanse(event);
            
            // Transform
            ProcessedData processed = transformer.transform(cleaned);
            
            // Calculate quality
            processed.setQualityScore(qualityCalculator.calculate(processed));
            
            return processed;
        });
    }
}
```

### 3.2 Exactly-Once Processing

**Kafka Consumer Configuration**
```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      isolation-level: read_committed
    listener:
      ack-mode: manual
    producer:
      transaction-id-prefix: processing-
      acks: all
      enable-idempotence: true
```

**Transactional Processing**
```java
@Service
public class TransactionalProcessor {
    
    @Transactional
    @KafkaListener(topics = "raw-data-ingested")
    public void processMessage(ConsumerRecord<String, RawDataEvent> record,
                               Acknowledgment acknowledgment) {
        try {
            ProcessedData processed = process(record.value());
            
            // Publish to next topic
            kafkaTemplate.send("data-ready-for-storage", processed);
            
            // Commit offset
            acknowledgment.acknowledge();
        } catch (Exception e) {
            // Send to DLQ
            kafkaTemplate.send("processing-failed", record.value());
            acknowledgment.acknowledge();
        }
    }
}
```

### 3.3 Rule Engine Implementation

**Rule Definition**
```java
@Entity
public class ValidationRule {
    @Id
    private UUID ruleId;
    private String name;
    private String entityType;
    private String condition; // SpEL expression
    private String action; // reject, warn, correct
    private int priority;
    private boolean enabled;
}
```

**Rule Evaluation**
```java
@Service
public class RuleEngine {
    
    private final SpelExpressionParser parser = new SpelExpressionParser();
    
    public RuleEvaluationResult evaluate(ValidationRule rule, RawDataEvent event) {
        try {
            Expression expression = parser.parseExpression(rule.getCondition());
            StandardEvaluationContext context = new StandardEvaluationContext(event);
            
            Boolean result = expression.getValue(context, Boolean.class);
            
            return RuleEvaluationResult.builder()
                .ruleId(rule.getRuleId())
                .passed(result)
                .build();
        } catch (Exception e) {
            return RuleEvaluationResult.builder()
                .ruleId(rule.getRuleId())
                .passed(false)
                .error(e.getMessage())
                .build();
        }
    }
}
```

### 3.4 Quality Metrics Calculation

```java
@Service
public class QualityMetricsCalculator {
    
    public QualityScore calculate(ProcessedData data) {
        double completeness = calculateCompleteness(data);
        double accuracy = calculateAccuracy(data);
        double consistency = calculateConsistency(data);
        double timeliness = calculateTimeliness(data);
        
        double overall = (completeness * 0.3) + 
                        (accuracy * 0.3) + 
                        (consistency * 0.2) + 
                        (timeliness * 0.2);
        
        return QualityScore.builder()
            .completeness(completeness)
            .accuracy(accuracy)
            .consistency(consistency)
            .timeliness(timeliness)
            .overall(overall)
            .build();
    }
    
    private double calculateCompleteness(ProcessedData data) {
        int totalFields = data.getFieldCount();
        int nonNullFields = data.getNonNullFieldCount();
        return (double) nonNullFields / totalFields * 100;
    }
}
```

---

## 4. INFRASTRUCTURE DESIGN

### 4.1 Compute Resources

**Production**
- Instances: 5-10 (auto-scaling)
- CPU: 4 cores per instance
- Memory: 8GB per instance

### 4.2 Storage

**PostgreSQL (Rules & Metrics)**
- Instance: db.t3.medium
- Storage: 100GB SSD
- Purpose: Validation rules, quality metrics

**MongoDB (Processed Data)**
- Deployment: Replica Set (3 nodes)
- Storage: 1TB SSD per node
- Purpose: Processed data records
- TTL: 7 days

### 4.3 Kafka Topics

**Input Topic**: `raw-data-ingested`
- Partitions: 12
- Replication: 3
- Retention: 7 days

**Output Topics**:
- `data-validated` (partitions: 12)
- `data-transformed` (partitions: 12)
- `data-ready-for-storage` (partitions: 12)
- `data-quality-alerts` (partitions: 6)
- `processing-failed` (DLQ, partitions: 6)

### 4.4 Infrastructure Cost

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| Service Instances | 5-10 avg 7 | $200/month | $1,400 |
| PostgreSQL | 1 | $100/month | $100 |
| MongoDB | 3 nodes | $200/month | $600 |
| Kafka Topics | 5 topics | Included | $0 |
| **Total** | | | **$2,100/month** |

---

## 5. CODE GENERATION

### 5.1 Project Structure

```
data-processing-service/
├── src/main/java/com/analytics/processing/
│   ├── DataProcessingApplication.java
│   ├── config/
│   │   ├── KafkaConfig.java
│   │   ├── DatabaseConfig.java
│   │   └── AsyncConfig.java
│   ├── consumer/
│   │   └── DataConsumer.java
│   ├── validator/
│   │   ├── SchemaValidator.java
│   │   ├── DataTypeValidator.java
│   │   └── BusinessRuleValidator.java
│   ├── cleanser/
│   │   ├── StandardizationService.java
│   │   ├── DeduplicationService.java
│   │   └── NormalizationService.java
│   ├── transformer/
│   │   ├── FieldMapper.java
│   │   ├── TypeConverter.java
│   │   └── EnrichmentService.java
│   ├── rules/
│   │   ├── RuleEngine.java
│   │   └── RuleEvaluator.java
│   ├── quality/
│   │   ├── QualityMetricsCalculator.java
│   │   └── QualityReporter.java
│   ├── service/
│   │   ├── ProcessingPipeline.java
│   │   └── ErrorHandler.java
│   ├── model/
│   │   ├── RawDataEvent.java
│   │   ├── ProcessedData.java
│   │   ├── ValidationRule.java
│   │   └── QualityScore.java
│   └── repository/
│       ├── ValidationRuleRepository.java
│       └── QualityMetricsRepository.java
├── src/main/resources/
│   ├── application.yml
│   └── rules/
│       └── default-rules.json
└── pom.xml
```

### 5.2 Key Implementation - Processing Pipeline

```java
@Service
@RequiredArgsConstructor
public class ProcessingPipeline {
    
    private final SchemaValidator schemaValidator;
    private final BusinessRuleValidator ruleValidator;
    private final StandardizationService standardizer;
    private final DeduplicationService deduplicator;
    private final FieldMapper fieldMapper;
    private final QualityMetricsCalculator qualityCalculator;
    private final KafkaTemplate<String, ProcessedData> kafkaTemplate;
    
    public ProcessedData process(RawDataEvent event) {
        // Step 1: Schema Validation
        ValidationResult schemaResult = schemaValidator.validate(event);
        if (!schemaResult.isValid()) {
            throw new SchemaValidationException(schemaResult.getErrors());
        }
        
        // Step 2: Business Rule Validation
        ValidationResult ruleResult = ruleValidator.validate(event);
        if (!ruleResult.isValid() && ruleResult.getSeverity() == Severity.ERROR) {
            throw new BusinessRuleException(ruleResult.getErrors());
        }
        
        // Step 3: Standardization
        StandardizedData standardized = standardizer.standardize(event);
        
        // Step 4: Deduplication Check
        if (deduplicator.isDuplicate(standardized)) {
            throw new DuplicateRecordException(standardized.getBusinessKey());
        }
        
        // Step 5: Field Mapping & Transformation
        ProcessedData processed = fieldMapper.map(standardized);
        
        // Step 6: Quality Scoring
        QualityScore quality = qualityCalculator.calculate(processed);
        processed.setQualityScore(quality);
        
        // Step 7: Quality Threshold Check
        if (quality.getOverall() < 85.0) {
            kafkaTemplate.send("data-quality-alerts", processed);
        }
        
        return processed;
    }
}
```

### 5.3 Key Implementation - Schema Validator

```java
@Service
public class SchemaValidator {
    
    @Autowired
    private SchemaRepository schemaRepository;
    
    public ValidationResult validate(RawDataEvent event) {
        Schema schema = schemaRepository.findByEntityType(event.getEntityType())
            .orElseThrow(() -> new SchemaNotFoundException(event.getEntityType()));
        
        List<ValidationError> errors = new ArrayList<>();
        
        // Validate required fields
        for (Field field : schema.getRequiredFields()) {
            if (!event.hasField(field.getName()) || 
                event.getField(field.getName()) == null) {
                errors.add(new ValidationError(
                    field.getName(), 
                    "Required field is missing or null"
                ));
            }
        }
        
        // Validate data types
        for (Field field : schema.getFields()) {
            if (event.hasField(field.getName())) {
                Object value = event.getField(field.getName());
                if (!isValidType(value, field.getType())) {
                    errors.add(new ValidationError(
                        field.getName(),
                        "Invalid data type. Expected: " + field.getType()
                    ));
                }
            }
        }
        
        return ValidationResult.builder()
            .valid(errors.isEmpty())
            .errors(errors)
            .build();
    }
}
```

### 5.4 Key Implementation - Kafka Consumer

```java
@Service
@Slf4j
public class DataConsumer {
    
    @Autowired
    private ProcessingPipeline pipeline;
    
    @Autowired
    private KafkaTemplate<String, ProcessedData> kafkaTemplate;
    
    @KafkaListener(
        topics = "raw-data-ingested",
        groupId = "data-processing",
        concurrency = "10"
    )
    public void consume(ConsumerRecord<String, RawDataEvent> record,
                       Acknowledgment acknowledgment) {
        try {
            log.info("Processing record: {}", record.key());
            
            ProcessedData processed = pipeline.process(record.value());
            
            // Publish to storage topic
            kafkaTemplate.send("data-ready-for-storage", 
                processed.getRecordId().toString(), 
                processed);
            
            // Acknowledge
            acknowledgment.acknowledge();
            
            log.info("Successfully processed record: {}", record.key());
            
        } catch (ValidationException e) {
            log.error("Validation failed for record: {}", record.key(), e);
            sendToDeadLetterQueue(record.value(), e);
            acknowledgment.acknowledge();
            
        } catch (Exception e) {
            log.error("Processing failed for record: {}", record.key(), e);
            // Don't acknowledge - will retry
        }
    }
    
    private void sendToDeadLetterQueue(RawDataEvent event, Exception error) {
        FailedRecord failed = FailedRecord.builder()
            .originalEvent(event)
            .error(error.getMessage())
            .timestamp(Instant.now())
            .build();
        
        kafkaTemplate.send("processing-failed", failed);
    }
}
```

### 5.5 API Endpoints

**Reprocessing**
```
POST /api/v1/processing/reprocess
Body: { "jobId": "uuid", "recordIds": ["id1", "id2"] }
```

**Quality Metrics**
```
GET /api/v1/processing/quality/metrics?startDate=2026-03-01&endDate=2026-03-04
```

**Validation Rules**
```
GET    /api/v1/processing/rules
POST   /api/v1/processing/rules
PUT    /api/v1/processing/rules/{ruleId}
DELETE /api/v1/processing/rules/{ruleId}
```

### 5.6 Configuration

```yaml
spring:
  application:
    name: data-processing-service
  
  datasource:
    url: jdbc:postgresql://${POSTGRES_HOST:localhost}:5432/processing
    username: ${POSTGRES_USER}
    password: ${POSTGRES_PASSWORD}
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/processed_data
  
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    consumer:
      group-id: data-processing
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 500
    producer:
      acks: all
      enable-idempotence: true
      transaction-id-prefix: processing-
    listener:
      ack-mode: manual
      concurrency: 10

processing:
  validation:
    fail-fast: true
  quality:
    threshold:
      completeness: 90.0
      accuracy: 95.0
      overall: 85.0
  deduplication:
    enabled: true
    window-minutes: 60
  retry:
    max-attempts: 3
    backoff-delay: 1000

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
    
    <!-- Spring Batch (for batch processing) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-batch</artifactId>
    </dependency>
    
    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
    </dependency>
    
    <!-- SpEL for rule engine -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-expression</artifactId>
    </dependency>
    
    <!-- Apache Commons -->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-text</artifactId>
        <version>1.10.0</version>
    </dependency>
</dependencies>
```

---

## 6. DEPLOYMENT

### 6.1 Docker Configuration

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/data-processing-service-1.0.0.jar app.jar
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
  name: data-processing-service
spec:
  replicas: 5
  selector:
    matchLabels:
      app: data-processing
  template:
    metadata:
      labels:
        app: data-processing
    spec:
      containers:
      - name: data-processing
        image: data-processing-service:1.0.0
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
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: data-processing-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-processing-service
  minReplicas: 5
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 7. SUCCESS CRITERIA

- [ ] Process 1M records/hour
- [ ] Validation latency < 50ms per record
- [ ] Transformation latency < 100ms per record
- [ ] Zero data loss during processing
- [ ] Detect 100% of schema violations
- [ ] Identify 95%+ of duplicates
- [ ] Quality score calculation for all records
- [ ] Exactly-once processing semantics
- [ ] Complete API documentation
- [ ] 80%+ test coverage

---

## Document Status

**Status**: Complete  
**All Stages**: Functional Design, NFR Requirements, NFR Design, Infrastructure Design, Code Generation  
**Next**: Unit 4 (Analytics API Service)

---
