# Unit 1: Data Ingestion Service
## Implementation Specification

**Unit ID**: UNIT-001  
**Priority**: High  
**Complexity**: High  
**Estimated Effort**: 2 sessions  
**Implementation Phase**: 2

---

## 1. Unit Overview

**Purpose**: Collect data from all external sources and publish to message broker for processing

**Business Value**: Enables the platform to integrate with any data source, providing the foundation for the entire data pipeline

---

## 2. Functional Requirements

### FR-1.1: Database Connectivity
- Support PostgreSQL, MySQL, MongoDB, Oracle, SQL Server
- Connection pooling with configurable limits
- Automatic reconnection on failure
- Support for both full and incremental extraction
- Configurable extraction queries

### FR-1.2: API Integration
- REST API client with multiple auth methods (OAuth 2.0, API Key, Basic, JWT)
- SOAP/XML web service support
- GraphQL query execution
- Rate limiting and throttling
- Retry logic with exponential backoff
- Response parsing and normalization

### FR-1.3: File Processing
- Support CSV, JSON, XML, Excel (XLS/XLSX)
- File upload via REST API
- FTP/SFTP monitoring and retrieval
- Cloud storage integration (S3, Azure Blob, GCS)
- Streaming for large files (> 100MB)
- Chunked processing

### FR-1.4: Streaming Integration
- Apache Kafka consumer
- RabbitMQ consumer
- AWS SQS consumer
- Azure Service Bus consumer
- Backpressure handling
- Offset management

### FR-1.5: Webhook Receivers
- Dynamic webhook endpoint creation
- Signature verification
- Payload validation
- Async processing

### FR-1.6: Job Management
- Job scheduling (cron-based)
- Job status tracking
- Job history and logs
- Manual job triggering
- Job cancellation

---

## 3. Technical Specifications

### 3.1 Component Architecture

```
DataIngestionService
├── connectors/
│   ├── DatabaseConnector
│   │   ├── PostgreSQLConnector
│   │   ├── MySQLConnector
│   │   ├── MongoDBConnector
│   │   └── OracleConnector
│   ├── ApiConnector
│   │   ├── RestApiClient
│   │   ├── SoapClient
│   │   └── GraphQLClient
│   ├── FileConnector
│   │   ├── CsvProcessor
│   │   ├── JsonProcessor
│   │   ├── XmlProcessor
│   │   └── ExcelProcessor
│   └── StreamConnector
│       ├── KafkaConsumer
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

### 3.2 Key Classes

**DatabaseConnector (Interface)**
```java
public interface DatabaseConnector {
    Connection connect(ConnectionConfig config);
    List<Record> extract(Connection conn, String query);
    void disconnect(Connection conn);
    boolean testConnection(ConnectionConfig config);
}
```

**IngestionJobService**
```java
@Service
public class IngestionJobService {
    public UUID createJob(DataSource source);
    public void executeJob(UUID jobId);
    public JobStatus getJobStatus(UUID jobId);
    public void cancelJob(UUID jobId);
    public List<IngestionJob> getJobHistory(UUID sourceId);
}
```

**KafkaEventPublisher**
```java
@Service
public class KafkaEventPublisher {
    public void publishRawData(RawDataEvent event);
    public void publishJobStatus(JobStatusEvent event);
}
```

---

## 4. API Endpoints

### 4.1 File Upload
```
POST /api/v1/ingestion/files/upload
Content-Type: multipart/form-data
Authorization: Bearer {token}

Request:
- file: binary
- sourceType: csv|json|xml|excel
- businessId: UUID
- metadata: JSON

Response: 202 Accepted
{
  "jobId": "uuid",
  "status": "accepted",
  "fileName": "data.csv",
  "recordCount": 0
}
```

### 4.2 Database Connection Configuration
```
POST /api/v1/ingestion/connections/database
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "name": "Production DB",
  "type": "postgresql",
  "host": "db.example.com",
  "port": 5432,
  "database": "production",
  "username": "user",
  "password": "encrypted",
  "extractionQuery": "SELECT * FROM orders WHERE updated_at > ?",
  "schedule": "0 */6 * * *"
}

Response: 201 Created
{
  "connectionId": "uuid",
  "name": "Production DB",
  "status": "active",
  "nextSync": "2026-03-04T12:00:00Z"
}
```

### 4.3 API Integration Configuration
```
POST /api/v1/ingestion/connections/api
Content-Type: application/json
Authorization: Bearer {token}

Request:
{
  "name": "Salesforce API",
  "url": "https://api.salesforce.com/v1/accounts",
  "method": "GET",
  "authType": "oauth2",
  "authConfig": {
    "clientId": "...",
    "clientSecret": "...",
    "tokenUrl": "..."
  },
  "schedule": "0 * * * *"
}

Response: 201 Created
{
  "integrationId": "uuid",
  "name": "Salesforce API",
  "status": "active"
}
```

### 4.4 Webhook Receiver
```
POST /api/v1/ingestion/webhooks/{webhookId}
Content-Type: application/json
X-Webhook-Signature: {signature}

Request: Any JSON payload

Response: 200 OK
{
  "received": true,
  "jobId": "uuid",
  "timestamp": "2026-03-04T10:00:00Z"
}
```

### 4.5 Job Status
```
GET /api/v1/ingestion/jobs/{jobId}
Authorization: Bearer {token}

Response: 200 OK
{
  "jobId": "uuid",
  "status": "processing",
  "source": "database",
  "recordsProcessed": 5000,
  "recordsFailed": 10,
  "startTime": "2026-03-04T10:00:00Z",
  "errors": [...]
}
```

---

## 5. Data Models

### 5.1 PostgreSQL Tables

**data_sources**
```sql
CREATE TABLE data_sources (
    source_id UUID PRIMARY KEY,
    business_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL,
    connection_config JSONB NOT NULL,
    schedule VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active',
    last_sync_at TIMESTAMP,
    next_sync_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**ingestion_jobs**
```sql
CREATE TABLE ingestion_jobs (
    job_id UUID PRIMARY KEY,
    source_id UUID REFERENCES data_sources(source_id),
    business_id UUID NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    records_total INTEGER DEFAULT 0,
    records_processed INTEGER DEFAULT 0,
    records_failed INTEGER DEFAULT 0,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB
);
```

### 5.2 MongoDB Collections

**raw_data**
```javascript
{
  _id: ObjectId,
  jobId: UUID,
  sourceId: UUID,
  businessId: UUID,
  sourceType: String,
  recordId: String,
  data: Object,
  metadata: {
    ingestedAt: ISODate,
    sourceEndpoint: String
  },
  status: String,
  createdAt: ISODate
}
```

---

## 6. Kafka Events

### 6.1 Raw Data Ingested Event
**Topic**: `raw-data-ingested`

```json
{
  "eventId": "uuid",
  "eventType": "raw-data-ingested",
  "timestamp": "2026-03-04T10:00:00Z",
  "jobId": "uuid",
  "sourceId": "uuid",
  "businessId": "uuid",
  "sourceType": "database",
  "recordId": "string",
  "data": {},
  "metadata": {}
}
```

---

## 7. Configuration

### 7.1 Application Properties
```yaml
spring:
  application:
    name: data-ingestion-service
  datasource:
    url: jdbc:postgresql://localhost:5432/analytics
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

ingestion:
  file:
    max-size: 1GB
    allowed-types: csv,json,xml,xlsx
    temp-directory: /tmp/uploads
  database:
    connection-pool:
      max-size: 20
      min-idle: 5
      connection-timeout: 30000
  api:
    retry:
      max-attempts: 3
      backoff-delay: 1000
    timeout: 30000
  scheduler:
    thread-pool-size: 10
```

---

## 8. Dependencies

### 8.1 External Dependencies
- Unit 6 (Monitoring & Audit Service)
- Apache Kafka
- PostgreSQL
- MongoDB

### 8.2 Maven Dependencies
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
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.dataformat</groupId>
        <artifactId>jackson-dataformat-xml</artifactId>
    </dependency>
    
    <!-- Apache Camel -->
    <dependency>
        <groupId>org.apache.camel.springboot</groupId>
        <artifactId>camel-spring-boot-starter</artifactId>
    </dependency>
    
    <!-- Scheduler -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-quartz</artifactId>
    </dependency>
</dependencies>
```

---

## 9. Testing Requirements

### 9.1 Unit Tests
- Database connector tests (with H2/Testcontainers)
- API client tests (with WireMock)
- File processor tests
- Kafka publisher tests (with EmbeddedKafka)
- Validation logic tests

### 9.2 Integration Tests
- End-to-end ingestion flow
- Kafka integration
- Database connectivity
- Error handling scenarios

### 9.3 Test Coverage Target
- Minimum 80% code coverage
- 100% coverage for critical paths

---

## 10. Success Criteria

- [ ] Successfully connect to all supported data source types
- [ ] Handle 1M records/hour throughput
- [ ] 99% job success rate
- [ ] Proper error handling and retry logic
- [ ] Complete audit trail for all ingestion activities
- [ ] All APIs documented with OpenAPI
- [ ] 80%+ test coverage

---

## Document Status

**Status**: Ready for Implementation  
**Prepared By**: AI Development Assistant  
**Date**: 2026-03-04

---
