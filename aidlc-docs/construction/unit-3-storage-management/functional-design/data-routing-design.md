# Data Routing Design
## Unit 3: Storage Management Service - Functional Design

**Version**: 1.0  
**Date**: 2026-03-04  
**Unit**: Storage Management Service

---

## 1. Overview

The Storage Management Service is responsible for intelligently routing processed data to the appropriate database based on data characteristics, optimizing write performance, and maintaining complete data lineage.

---

## 2. Routing Strategy

### 2.1 Routing Decision Logic

The service uses a multi-factor decision algorithm to determine the optimal database for each data record:

```
Decision Factors:
1. Entity Type (customer, transaction, metric, document)
2. Data Structure (structured, semi-structured, unstructured)
3. Query Pattern (transactional, analytical, time-series)
4. Data Volume (small, medium, large)
5. Update Frequency (static, periodic, real-time)
6. Retention Period (short-term, medium-term, long-term)
```

### 2.2 Routing Rules

**PostgreSQL (Relational Database)**
- Entity Types: businesses, customers, transactions, users, roles
- Data Structure: Structured with fixed schema
- Query Pattern: Transactional (CRUD operations)
- Use Case: Operational data requiring ACID compliance
- Examples: Customer records, user accounts, business configurations

**MongoDB (Document Store)**
- Entity Types: raw_data, processed_data, logs, audit_logs
- Data Structure: Semi-structured, flexible schema
- Query Pattern: Document retrieval, flexible queries
- Use Case: Variable schema data, audit trails
- Examples: Raw ingested data, error logs, data lineage

**TimescaleDB (Time-Series Database)**
- Entity Types: metrics, events, sensor_data
- Data Structure: Time-stamped records
- Query Pattern: Time-range queries, aggregations
- Use Case: System metrics, business metrics, IoT data
- Examples: System performance metrics, business KPIs over time

**Snowflake/Redshift (Data Warehouse)**
- Entity Types: fact_transactions, dim_customer, dim_business
- Data Structure: Star/snowflake schema
- Query Pattern: Complex analytical queries
- Use Case: Historical analysis, reporting, BI
- Examples: Historical transactions, aggregated summaries

---

## 3. Routing Algorithm

### 3.1 Decision Tree

```
Input: ProcessedDataRecord

Step 1: Identify Entity Type
├─ If entity_type in [business, customer, transaction, user, role]
│  └─> Route to PostgreSQL
│
├─ If entity_type in [raw_data, processed_data, audit_log, error_log]
│  └─> Route to MongoDB
│
├─ If entity_type in [metric, event, sensor_data]
│  └─> Route to TimescaleDB
│
└─ If entity_type in [fact_*, dim_*] OR data_age > 90 days
   └─> Route to Snowflake/Redshift

Step 2: Apply Business Rules
├─ If requires_audit_trail = true
│  └─> Also write to MongoDB (data_lineage collection)
│
├─ If is_time_series = true AND retention > 90 days
│  └─> Write to TimescaleDB AND Snowflake (dual write)
│
└─ If is_high_value_transaction = true
   └─> Write to PostgreSQL AND Snowflake (dual write)

Step 3: Validate Routing Decision
├─ Check database availability
├─ Verify schema compatibility
└─ Confirm write permissions

Step 4: Execute Write
└─> Perform write operation with retry logic
```

### 3.2 Routing Configuration

**Routing Rules Table** (PostgreSQL)
```sql
CREATE TABLE routing_rules (
    rule_id UUID PRIMARY KEY,
    entity_type VARCHAR(100) NOT NULL,
    target_database VARCHAR(50) NOT NULL,
    priority INTEGER DEFAULT 1,
    conditions JSONB,
    enabled BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example rules
INSERT INTO routing_rules (entity_type, target_database, priority, conditions) VALUES
('customer', 'postgresql', 1, '{"data_structure": "structured"}'),
('transaction', 'postgresql', 1, '{"requires_acid": true}'),
('metric', 'timescaledb', 1, '{"is_time_series": true}'),
('audit_log', 'mongodb', 1, '{"flexible_schema": true}'),
('fact_transaction', 'snowflake', 1, '{"data_age_days": ">90"}');
```

---

## 4. Database Writer Components

### 4.1 PostgreSQL Writer

**Purpose**: Write structured operational data

**Features**:
- Batch writing for performance
- Transaction management
- Conflict resolution (upsert)
- Foreign key validation
- Partition management

**Write Strategies**:
- Single record insert
- Batch insert (up to 1000 records)
- Upsert (INSERT ... ON CONFLICT UPDATE)
- Bulk copy (COPY command for large datasets)

**Schema Management**:
- Automatic table creation (if configured)
- Schema version tracking
- Migration support

### 4.2 MongoDB Writer

**Purpose**: Write semi-structured and document data

**Features**:
- Flexible schema
- Bulk write operations
- TTL index management
- Collection sharding support

**Write Strategies**:
- Single document insert
- Bulk insert (up to 1000 documents)
- Update with upsert
- Replace document

**Collection Management**:
- Automatic collection creation
- Index creation and management
- TTL configuration

### 4.3 TimescaleDB Writer

**Purpose**: Write time-series data

**Features**:
- Hypertable management
- Automatic partitioning by time
- Continuous aggregates
- Retention policy enforcement

**Write Strategies**:
- Single row insert
- Batch insert optimized for time-series
- Streaming insert for real-time data

**Hypertable Management**:
- Automatic hypertable creation
- Chunk management
- Compression policies

### 4.4 Snowflake/Redshift Writer

**Purpose**: Write analytical data to warehouse

**Features**:
- Bulk loading (COPY command)
- Staging table pattern
- Merge operations (SCD Type 2)
- Partition management

**Write Strategies**:
- Stage to S3/Azure Blob → COPY to Snowflake
- Batch insert (for small volumes)
- Merge for dimension updates

**Warehouse Management**:
- Warehouse sizing
- Query optimization
- Clustering keys

---

## 5. Write Optimization

### 5.1 Batching Strategy

**Batch Configuration**:
```yaml
storage:
  batch:
    postgresql:
      size: 1000
      timeout: 5000ms
    mongodb:
      size: 1000
      timeout: 3000ms
    timescaledb:
      size: 5000
      timeout: 2000ms
    snowflake:
      size: 10000
      timeout: 30000ms
```

**Batching Logic**:
- Accumulate records in memory buffer
- Flush when batch size reached OR timeout expires
- Group by target database and entity type
- Maintain order for time-series data

### 5.2 Connection Pooling

**Pool Configuration**:
```yaml
storage:
  connection-pool:
    postgresql:
      max-size: 50
      min-idle: 10
      connection-timeout: 30000
    mongodb:
      max-size: 50
      min-idle: 10
    timescaledb:
      max-size: 30
      min-idle: 5
    snowflake:
      max-size: 20
      min-idle: 5
```

### 5.3 Write Performance Targets

| Database | Single Write | Batch Write (1000 records) | Throughput |
|----------|-------------|---------------------------|------------|
| PostgreSQL | < 10ms | < 500ms | 10,000 records/sec |
| MongoDB | < 5ms | < 300ms | 15,000 records/sec |
| TimescaleDB | < 8ms | < 400ms | 12,000 records/sec |
| Snowflake | N/A | < 5000ms | 5,000 records/sec |

---

## 6. Schema Management

### 6.1 Schema Registry

**Purpose**: Centralized schema definitions and versioning

**Schema Definition**:
```json
{
  "schemaId": "uuid",
  "entityType": "customer",
  "version": "1.0.0",
  "targetDatabase": "postgresql",
  "tableName": "customers",
  "fields": [
    {
      "name": "customer_id",
      "type": "UUID",
      "required": true,
      "primaryKey": true
    },
    {
      "name": "email",
      "type": "VARCHAR(255)",
      "required": true,
      "indexed": true
    }
  ],
  "indexes": [
    {
      "name": "idx_customers_email",
      "columns": ["email"],
      "unique": false
    }
  ]
}
```

### 6.2 Schema Evolution

**Versioning Strategy**:
- Semantic versioning (MAJOR.MINOR.PATCH)
- Backward compatibility for MINOR/PATCH changes
- Migration scripts for MAJOR changes

**Schema Changes**:
- Add column: Non-breaking (default value required)
- Remove column: Breaking (requires migration)
- Rename column: Breaking (requires migration)
- Change type: Breaking (requires migration)

### 6.3 Schema Validation

**Pre-Write Validation**:
- Verify schema exists for entity type
- Validate data against schema definition
- Check required fields
- Validate data types
- Enforce constraints

---

## 7. Data Lineage Tracking

### 7.1 Lineage Record Structure

**MongoDB Collection**: `data_lineage`

```javascript
{
  _id: ObjectId,
  recordId: UUID,
  businessId: UUID,
  entityType: String,
  currentLocation: {
    database: "postgresql",
    table: "customers",
    primaryKey: "customer-uuid-123",
    timestamp: ISODate("2026-03-04T10:00:00Z")
  },
  history: [
    {
      timestamp: ISODate("2026-03-04T09:00:00Z"),
      event: "ingested",
      source: "api-integration-1",
      details: {}
    },
    {
      timestamp: ISODate("2026-03-04T09:05:00Z"),
      event: "validated",
      service: "data-processing-pipeline",
      details: {}
    },
    {
      timestamp: ISODate("2026-03-04T09:10:00Z"),
      event: "transformed",
      service: "data-processing-pipeline",
      details: {}
    },
    {
      timestamp: ISODate("2026-03-04T09:15:00Z"),
      event: "stored",
      database: "postgresql",
      table: "customers",
      primaryKey: "customer-uuid-123"
    }
  ],
  createdAt: ISODate,
  updatedAt: ISODate
}
```

### 7.2 Lineage Tracking Process

**On Write Success**:
1. Create or update lineage record
2. Add "stored" event to history
3. Update currentLocation
4. Record database, table, and primary key
5. Timestamp the event

**On Write Failure**:
1. Add "storage_failed" event to history
2. Record error details
3. Do not update currentLocation

---

## 8. Error Handling

### 8.1 Write Failure Scenarios

**Database Unavailable**:
- Retry with exponential backoff (3 attempts)
- If all retries fail, send to dead letter queue
- Alert monitoring service

**Schema Mismatch**:
- Log validation error
- Send to error collection for manual review
- Do not retry automatically

**Constraint Violation**:
- Log constraint error (duplicate key, foreign key, etc.)
- Attempt conflict resolution if configured
- Otherwise, send to error collection

**Timeout**:
- Cancel operation
- Retry with smaller batch size
- If persistent, alert monitoring

### 8.2 Dead Letter Queue

**Kafka Topic**: `storage-failed`

**Message Structure**:
```json
{
  "recordId": "uuid",
  "entityType": "customer",
  "targetDatabase": "postgresql",
  "data": {},
  "error": {
    "code": "CONSTRAINT_VIOLATION",
    "message": "Duplicate key violation",
    "timestamp": "2026-03-04T10:00:00Z"
  },
  "retryCount": 3,
  "originalTopic": "data-ready-for-storage"
}
```

### 8.3 Retry Strategy

**Retry Configuration**:
```yaml
storage:
  retry:
    max-attempts: 3
    backoff:
      initial-delay: 1000ms
      multiplier: 2
      max-delay: 10000ms
    retryable-errors:
      - CONNECTION_TIMEOUT
      - DATABASE_UNAVAILABLE
      - TEMPORARY_FAILURE
```

---

## 9. Transaction Management

### 9.1 Single Database Transactions

**PostgreSQL**:
- Use Spring @Transactional
- ACID guarantees
- Rollback on error

**MongoDB**:
- Multi-document transactions (MongoDB 4.0+)
- Session-based transactions
- Rollback on error

### 9.2 Distributed Transactions

**Scenario**: Write to multiple databases (e.g., PostgreSQL + MongoDB for lineage)

**Strategy**: Saga Pattern
1. Write to primary database (PostgreSQL)
2. If successful, write lineage to MongoDB
3. If lineage write fails, log error but don't rollback primary write
4. Compensating transaction: Manual lineage correction

**Not Using**: Two-Phase Commit (2PC) - too slow and complex

---

## 10. Monitoring and Metrics

### 10.1 Key Metrics

**Write Metrics**:
- `storage.writes.total` - Total write operations
- `storage.writes.success` - Successful writes
- `storage.writes.failed` - Failed writes
- `storage.writes.duration` - Write duration (ms)
- `storage.batch.size` - Batch size distribution

**Database Metrics**:
- `storage.postgresql.connections` - Active connections
- `storage.mongodb.connections` - Active connections
- `storage.timescaledb.connections` - Active connections
- `storage.snowflake.connections` - Active connections

**Error Metrics**:
- `storage.errors.constraint_violation` - Constraint violations
- `storage.errors.timeout` - Timeout errors
- `storage.errors.unavailable` - Database unavailable errors

### 10.2 Health Checks

**Database Health**:
- Connection test every 30 seconds
- Query execution test
- Disk space check
- Replication lag check (if applicable)

**Service Health**:
- Kafka consumer lag
- Batch queue size
- Error rate
- Write throughput

---

## 11. API Design

### 11.1 Storage Statistics API

```
GET /api/v1/storage/stats
Authorization: Bearer {token}

Response:
{
  "databases": [
    {
      "type": "postgresql",
      "name": "analytics-db",
      "recordCount": 5000000,
      "sizeGB": 50.5,
      "lastWrite": "2026-03-04T10:00:00Z",
      "health": "healthy"
    }
  ],
  "totalRecords": 7000000,
  "totalSizeGB": 80.7
}
```

### 11.2 Schema Management API

```
GET /api/v1/storage/schemas
POST /api/v1/storage/schemas
GET /api/v1/storage/schemas/{schemaId}
PUT /api/v1/storage/schemas/{schemaId}
```

### 11.3 Manual Write API (for testing)

```
POST /api/v1/storage/write
Authorization: Bearer {token}

Request:
{
  "entityType": "customer",
  "targetDatabase": "postgresql",
  "data": {
    "customer_id": "uuid",
    "email": "test@example.com"
  }
}

Response: 201 Created
{
  "recordId": "uuid",
  "database": "postgresql",
  "table": "customers",
  "timestamp": "2026-03-04T10:00:00Z"
}
```

---

## 12. Configuration

### 12.1 Application Configuration

```yaml
spring:
  application:
    name: storage-management-service
  
  datasource:
    postgresql:
      url: jdbc:postgresql://localhost:5432/analytics
      username: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
    
  data:
    mongodb:
      uri: mongodb://localhost:27017/analytics
      username: ${MONGO_USER}
      password: ${MONGO_PASSWORD}
  
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: storage-management
      auto-offset-reset: earliest

storage:
  routing:
    rules-refresh-interval: 60000 # 1 minute
  
  batch:
    enabled: true
    default-size: 1000
    default-timeout: 5000
  
  retry:
    enabled: true
    max-attempts: 3
  
  lineage:
    enabled: true
    async: true
```

---

## Document Status

**Status**: Complete - Awaiting Approval  
**Prepared By**: AI Development Assistant  
**Date**: 2026-03-04  
**Next**: NFR Requirements

---
