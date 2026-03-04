# Data Models
## Data Integration and Analytics Platform

**Version**: 1.0  
**Date**: 2026-03-04

---

## 1. Data Model Overview

The platform uses multiple databases optimized for different data types and access patterns. This document defines the core data models across all storage systems.

---

## 2. PostgreSQL Data Models (Relational)

### 2.1 Business Entity

**Table**: `businesses`

```sql
CREATE TABLE businesses (
    business_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    industry VARCHAR(100),
    country VARCHAR(2),
    timezone VARCHAR(50) DEFAULT 'UTC',
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB
);

CREATE INDEX idx_businesses_status ON businesses(status);
CREATE INDEX idx_businesses_industry ON businesses(industry);
```

### 2.2 Customer Entity

**Table**: `customers`

```sql
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID NOT NULL REFERENCES businesses(business_id),
    external_id VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(50),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active',
    acquisition_channel VARCHAR(50),
    acquisition_date DATE,
    lifetime_value DECIMAL(15,2) DEFAULT 0,
    total_orders INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB
);

CREATE INDEX idx_customers_business ON customers(business_id);
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_status ON customers(status);
CREATE INDEX idx_customers_external_id ON customers(business_id, external_id);
CREATE INDEX idx_customers_metadata ON customers USING GIN(metadata);
```

### 2.3 Transaction Entity

**Table**: `transactions`

```sql
CREATE TABLE transactions (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID NOT NULL REFERENCES businesses(business_id),
    customer_id UUID REFERENCES customers(customer_id),
    external_id VARCHAR(255),
    transaction_type VARCHAR(50) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) DEFAULT 'completed',
    transaction_date TIMESTAMP NOT NULL,
    payment_method VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB
);

CREATE INDEX idx_transactions_business ON transactions(business_id);
CREATE INDEX idx_transactions_customer ON transactions(customer_id);
CREATE INDEX idx_transactions_date ON transactions(transaction_date);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_type ON transactions(transaction_type);
CREATE INDEX idx_transactions_external_id ON transactions(business_id, external_id);

-- Partitioning by month for performance
CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### 2.4 Data Source Configuration

**Table**: `data_sources`

```sql
CREATE TABLE data_sources (
    source_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID NOT NULL REFERENCES businesses(business_id),
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- database, api, file, stream
    connection_config JSONB NOT NULL,
    schedule VARCHAR(100), -- cron expression
    status VARCHAR(20) DEFAULT 'active',
    last_sync_at TIMESTAMP,
    next_sync_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_data_sources_business ON data_sources(business_id);
CREATE INDEX idx_data_sources_type ON data_sources(type);
CREATE INDEX idx_data_sources_status ON data_sources(status);
```

### 2.5 Ingestion Job

**Table**: `ingestion_jobs`

```sql
CREATE TABLE ingestion_jobs (
    job_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID REFERENCES data_sources(source_id),
    business_id UUID NOT NULL REFERENCES businesses(business_id),
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

CREATE INDEX idx_ingestion_jobs_source ON ingestion_jobs(source_id);
CREATE INDEX idx_ingestion_jobs_status ON ingestion_jobs(status);
CREATE INDEX idx_ingestion_jobs_created ON ingestion_jobs(created_at DESC);
```

### 2.6 Data Quality Metrics

**Table**: `data_quality_metrics`

```sql
CREATE TABLE data_quality_metrics (
    metric_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id UUID REFERENCES ingestion_jobs(job_id),
    source_id UUID REFERENCES data_sources(source_id),
    business_id UUID NOT NULL REFERENCES businesses(business_id),
    metric_date DATE NOT NULL,
    total_records INTEGER NOT NULL,
    valid_records INTEGER NOT NULL,
    invalid_records INTEGER NOT NULL,
    completeness_score DECIMAL(5,2),
    accuracy_score DECIMAL(5,2),
    consistency_score DECIMAL(5,2),
    overall_quality_score DECIMAL(5,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_quality_metrics_date ON data_quality_metrics(metric_date);
CREATE INDEX idx_quality_metrics_source ON data_quality_metrics(source_id);
CREATE INDEX idx_quality_metrics_business ON data_quality_metrics(business_id);
```

### 2.7 Validation Rules

**Table**: `validation_rules`

```sql
CREATE TABLE validation_rules (
    rule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID REFERENCES businesses(business_id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    rule_type VARCHAR(50) NOT NULL, -- schema, business, quality
    condition JSONB NOT NULL,
    action VARCHAR(20) DEFAULT 'reject', -- reject, warn, correct
    enabled BOOLEAN DEFAULT true,
    priority INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_validation_rules_business ON validation_rules(business_id);
CREATE INDEX idx_validation_rules_type ON validation_rules(rule_type);
CREATE INDEX idx_validation_rules_enabled ON validation_rules(enabled);
```

### 2.8 User Management

**Table**: `users`

```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    status VARCHAR(20) DEFAULT 'active',
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);
```

**Table**: `user_roles`

```sql
CREATE TABLE user_roles (
    user_role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(user_id),
    role VARCHAR(50) NOT NULL, -- admin, business_owner, analyst, data_engineer, auditor
    business_id UUID REFERENCES businesses(business_id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_business ON user_roles(business_id);
CREATE UNIQUE INDEX idx_user_roles_unique ON user_roles(user_id, role, business_id);
```

### 2.9 Dashboard Configuration

**Table**: `dashboards`

```sql
CREATE TABLE dashboards (
    dashboard_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    business_id UUID REFERENCES businesses(business_id),
    user_id UUID REFERENCES users(user_id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    layout JSONB NOT NULL,
    is_public BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_dashboards_business ON dashboards(business_id);
CREATE INDEX idx_dashboards_user ON dashboards(user_id);
```

---

## 3. MongoDB Data Models (Document Store)

### 3.1 Raw Data Collection

**Collection**: `raw_data`

```javascript
{
  _id: ObjectId,
  jobId: UUID,
  sourceId: UUID,
  businessId: UUID,
  sourceType: String, // database, api, file, stream
  recordId: String,
  data: Object, // Original raw data
  metadata: {
    ingestedAt: ISODate,
    sourceEndpoint: String,
    fileInfo: {
      fileName: String,
      fileSize: Number,
      mimeType: String
    }
  },
  status: String, // pending, processed, failed
  createdAt: ISODate
}

// Indexes
db.raw_data.createIndex({ jobId: 1 })
db.raw_data.createIndex({ sourceId: 1 })
db.raw_data.createIndex({ businessId: 1 })
db.raw_data.createIndex({ status: 1 })
db.raw_data.createIndex({ createdAt: 1 })
db.raw_data.createIndex({ "metadata.ingestedAt": 1 })

// TTL Index - auto-delete after 30 days
db.raw_data.createIndex({ createdAt: 1 }, { expireAfterSeconds: 2592000 })
```

### 3.2 Processed Data Collection

**Collection**: `processed_data`

```javascript
{
  _id: ObjectId,
  recordId: UUID,
  businessId: UUID,
  sourceId: UUID,
  entityType: String, // customer, transaction, product, etc.
  data: Object, // Cleaned and transformed data
  validationResults: {
    passed: Boolean,
    rules: [
      {
        ruleId: UUID,
        ruleName: String,
        passed: Boolean,
        message: String
      }
    ]
  },
  transformations: [
    {
      type: String,
      field: String,
      originalValue: Mixed,
      newValue: Mixed,
      timestamp: ISODate
    }
  ],
  qualityScore: Number,
  processedAt: ISODate,
  storedAt: ISODate,
  storageLocation: {
    database: String,
    table: String,
    primaryKey: Mixed
  },
  createdAt: ISODate
}

// Indexes
db.processed_data.createIndex({ recordId: 1 }, { unique: true })
db.processed_data.createIndex({ businessId: 1 })
db.processed_data.createIndex({ sourceId: 1 })
db.processed_data.createIndex({ entityType: 1 })
db.processed_data.createIndex({ processedAt: 1 })
```

### 3.3 Data Lineage Collection

**Collection**: `data_lineage`

```javascript
{
  _id: ObjectId,
  recordId: UUID,
  businessId: UUID,
  entityType: String,
  currentLocation: {
    database: String,
    table: String,
    primaryKey: Mixed,
    timestamp: ISODate
  },
  history: [
    {
      timestamp: ISODate,
      event: String, // ingested, validated, transformed, stored, queried
      service: String,
      details: Object,
      userId: UUID // if user-initiated
    }
  ],
  createdAt: ISODate,
  updatedAt: ISODate
}

// Indexes
db.data_lineage.createIndex({ recordId: 1 })
db.data_lineage.createIndex({ businessId: 1 })
db.data_lineage.createIndex({ "currentLocation.database": 1 })
db.data_lineage.createIndex({ "history.timestamp": 1 })
```

### 3.4 Audit Logs Collection

**Collection**: `audit_logs`

```javascript
{
  _id: ObjectId,
  logId: UUID,
  timestamp: ISODate,
  userId: UUID,
  username: String,
  action: String, // login, query, export, update, delete
  resource: String, // API endpoint or resource identifier
  method: String, // GET, POST, PUT, DELETE
  ipAddress: String,
  userAgent: String,
  requestBody: Object,
  responseStatus: Number,
  duration: Number, // milliseconds
  businessId: UUID,
  metadata: Object
}

// Indexes
db.audit_logs.createIndex({ timestamp: -1 })
db.audit_logs.createIndex({ userId: 1 })
db.audit_logs.createIndex({ action: 1 })
db.audit_logs.createIndex({ businessId: 1 })
db.audit_logs.createIndex({ resource: 1 })

// TTL Index - auto-delete after 2 years
db.audit_logs.createIndex({ timestamp: 1 }, { expireAfterSeconds: 63072000 })
```

### 3.5 Error Logs Collection

**Collection**: `error_logs`

```javascript
{
  _id: ObjectId,
  errorId: UUID,
  timestamp: ISODate,
  service: String,
  severity: String, // error, warning, critical
  errorCode: String,
  errorMessage: String,
  stackTrace: String,
  context: {
    jobId: UUID,
    recordId: UUID,
    sourceId: UUID,
    businessId: UUID
  },
  resolved: Boolean,
  resolvedAt: ISODate,
  resolvedBy: UUID,
  metadata: Object
}

// Indexes
db.error_logs.createIndex({ timestamp: -1 })
db.error_logs.createIndex({ service: 1 })
db.error_logs.createIndex({ severity: 1 })
db.error_logs.createIndex({ resolved: 1 })
db.error_logs.createIndex({ "context.jobId": 1 })
```

---

## 4. TimescaleDB Data Models (Time-Series)

### 4.1 System Metrics

**Hypertable**: `system_metrics`

```sql
CREATE TABLE system_metrics (
    time TIMESTAMPTZ NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    metric_value DOUBLE PRECISION NOT NULL,
    tags JSONB,
    PRIMARY KEY (time, service_name, metric_name)
);

SELECT create_hypertable('system_metrics', 'time');

CREATE INDEX idx_system_metrics_service ON system_metrics(service_name, time DESC);
CREATE INDEX idx_system_metrics_name ON system_metrics(metric_name, time DESC);
CREATE INDEX idx_system_metrics_tags ON system_metrics USING GIN(tags);

-- Retention policy: keep 90 days
SELECT add_retention_policy('system_metrics', INTERVAL '90 days');
```

**Common Metrics**:
- `cpu_usage`: CPU utilization percentage
- `memory_usage`: Memory utilization percentage
- `request_count`: Number of requests
- `request_duration`: Request duration in ms
- `error_count`: Number of errors
- `active_connections`: Active database connections

### 4.2 Data Processing Metrics

**Hypertable**: `processing_metrics`

```sql
CREATE TABLE processing_metrics (
    time TIMESTAMPTZ NOT NULL,
    job_id UUID NOT NULL,
    source_id UUID,
    business_id UUID,
    metric_name VARCHAR(100) NOT NULL,
    metric_value DOUBLE PRECISION NOT NULL,
    tags JSONB,
    PRIMARY KEY (time, job_id, metric_name)
);

SELECT create_hypertable('processing_metrics', 'time');

CREATE INDEX idx_processing_metrics_job ON processing_metrics(job_id, time DESC);
CREATE INDEX idx_processing_metrics_source ON processing_metrics(source_id, time DESC);
CREATE INDEX idx_processing_metrics_business ON processing_metrics(business_id, time DESC);

-- Retention policy: keep 180 days
SELECT add_retention_policy('processing_metrics', INTERVAL '180 days');
```

**Common Metrics**:
- `records_ingested`: Number of records ingested
- `records_validated`: Number of records validated
- `records_failed`: Number of failed records
- `processing_duration`: Processing time in seconds
- `quality_score`: Data quality score

### 4.3 Business Metrics

**Hypertable**: `business_metrics`

```sql
CREATE TABLE business_metrics (
    time TIMESTAMPTZ NOT NULL,
    business_id UUID NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    metric_value DOUBLE PRECISION NOT NULL,
    dimension VARCHAR(100), -- Optional dimension (e.g., product_category)
    tags JSONB,
    PRIMARY KEY (time, business_id, metric_name)
);

SELECT create_hypertable('business_metrics', 'time');

CREATE INDEX idx_business_metrics_business ON business_metrics(business_id, time DESC);
CREATE INDEX idx_business_metrics_name ON business_metrics(metric_name, time DESC);
CREATE INDEX idx_business_metrics_dimension ON business_metrics(dimension, time DESC);

-- Continuous aggregates for hourly rollups
CREATE MATERIALIZED VIEW business_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    business_id,
    metric_name,
    AVG(metric_value) AS avg_value,
    MAX(metric_value) AS max_value,
    MIN(metric_value) AS min_value,
    SUM(metric_value) AS sum_value
FROM business_metrics
GROUP BY hour, business_id, metric_name;

SELECT add_continuous_aggregate_policy('business_metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

**Common Metrics**:
- `revenue`: Revenue amount
- `order_count`: Number of orders
- `customer_count`: Number of customers
- `average_order_value`: Average order value
- `conversion_rate`: Conversion rate percentage

---

## 5. Snowflake/Redshift Data Models (Data Warehouse)

### 5.1 Fact Table - Transactions

```sql
CREATE TABLE fact_transactions (
    transaction_key BIGINT IDENTITY(1,1) PRIMARY KEY,
    transaction_id VARCHAR(36) NOT NULL,
    business_key INTEGER NOT NULL,
    customer_key INTEGER,
    date_key INTEGER NOT NULL,
    time_key INTEGER NOT NULL,
    transaction_type VARCHAR(50),
    amount DECIMAL(15,2),
    currency VARCHAR(3),
    payment_method VARCHAR(50),
    status VARCHAR(20),
    source_system VARCHAR(100),
    created_date TIMESTAMP,
    updated_date TIMESTAMP
);

-- Distribution and sort keys (Redshift)
-- DISTKEY(business_key)
-- SORTKEY(date_key, business_key)
```

### 5.2 Dimension Table - Business

```sql
CREATE TABLE dim_business (
    business_key INTEGER IDENTITY(1,1) PRIMARY KEY,
    business_id VARCHAR(36) NOT NULL,
    business_name VARCHAR(255),
    industry VARCHAR(100),
    country VARCHAR(2),
    region VARCHAR(100),
    effective_date DATE,
    expiration_date DATE,
    is_current BOOLEAN DEFAULT true
);

-- DISTSTYLE ALL (Redshift - replicate to all nodes)
```

### 5.3 Dimension Table - Customer

```sql
CREATE TABLE dim_customer (
    customer_key INTEGER IDENTITY(1,1) PRIMARY KEY,
    customer_id VARCHAR(36) NOT NULL,
    business_id VARCHAR(36) NOT NULL,
    email VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    acquisition_channel VARCHAR(50),
    customer_segment VARCHAR(50),
    effective_date DATE,
    expiration_date DATE,
    is_current BOOLEAN DEFAULT true
);

-- DISTKEY(business_id)
```

### 5.4 Dimension Table - Date

```sql
CREATE TABLE dim_date (
    date_key INTEGER PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week INTEGER,
    day_name VARCHAR(10),
    day_of_month INTEGER,
    day_of_year INTEGER,
    week_of_year INTEGER,
    month INTEGER,
    month_name VARCHAR(10),
    quarter INTEGER,
    year INTEGER,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    fiscal_year INTEGER,
    fiscal_quarter INTEGER
);

-- DISTSTYLE ALL
```

### 5.5 Dimension Table - Time

```sql
CREATE TABLE dim_time (
    time_key INTEGER PRIMARY KEY,
    time_value TIME NOT NULL,
    hour INTEGER,
    minute INTEGER,
    second INTEGER,
    am_pm VARCHAR(2),
    time_period VARCHAR(20) -- morning, afternoon, evening, night
);

-- DISTSTYLE ALL
```

### 5.6 Aggregate Table - Daily Summary

```sql
CREATE TABLE agg_daily_summary (
    summary_date DATE NOT NULL,
    business_key INTEGER NOT NULL,
    total_revenue DECIMAL(15,2),
    total_transactions INTEGER,
    unique_customers INTEGER,
    average_order_value DECIMAL(15,2),
    new_customers INTEGER,
    returning_customers INTEGER,
    PRIMARY KEY (summary_date, business_key)
);

-- DISTKEY(business_key)
-- SORTKEY(summary_date, business_key)
```

---

## 6. Redis Cache Models (Key-Value)

### 6.1 Session Cache

**Key Pattern**: `session:{userId}`  
**TTL**: 1 hour  
**Value**:
```json
{
  "userId": "uuid",
  "username": "string",
  "roles": ["admin"],
  "businessIds": ["uuid"],
  "lastActivity": "2026-03-04T10:00:00Z"
}
```

### 6.2 API Response Cache

**Key Pattern**: `api:{endpoint}:{params_hash}`  
**TTL**: 5 minutes  
**Value**: JSON response

### 6.3 Dashboard Data Cache

**Key Pattern**: `dashboard:{dashboardId}:{userId}`  
**TTL**: 10 minutes  
**Value**: Dashboard configuration and data

### 6.4 Query Result Cache

**Key Pattern**: `query:{query_hash}`  
**TTL**: 15 minutes  
**Value**: Query results

---

## 7. Data Relationships

```
businesses (1) ----< (N) customers
businesses (1) ----< (N) transactions
businesses (1) ----< (N) data_sources
businesses (1) ----< (N) user_roles

customers (1) ----< (N) transactions

data_sources (1) ----< (N) ingestion_jobs
ingestion_jobs (1) ----< (N) data_quality_metrics

users (1) ----< (N) user_roles
users (1) ----< (N) dashboards
users (1) ----< (N) audit_logs
```

---

## 8. Data Model Conventions

### 8.1 Naming Conventions

- **Tables**: Lowercase with underscores (snake_case)
- **Columns**: Lowercase with underscores (snake_case)
- **Primary Keys**: `{table_name}_id` (UUID)
- **Foreign Keys**: `{referenced_table}_id`
- **Timestamps**: `created_at`, `updated_at`, `deleted_at`
- **Status Fields**: Use VARCHAR with predefined values

### 8.2 Standard Columns

All tables should include:
- Primary key (UUID)
- `created_at` (TIMESTAMP)
- `updated_at` (TIMESTAMP)
- `metadata` (JSONB) for extensibility

### 8.3 Soft Deletes

For audit purposes, use soft deletes:
- Add `deleted_at` TIMESTAMP column
- Add `is_deleted` BOOLEAN column
- Filter by `deleted_at IS NULL` or `is_deleted = false`

### 8.4 Audit Fields

For sensitive tables, include:
- `created_by` UUID
- `updated_by` UUID
- `version` INTEGER (for optimistic locking)

---

## 9. Data Validation Rules

### 9.1 Business Rules

- Email must be unique per business
- Transaction amount must be positive
- Customer status: active, inactive, suspended
- Transaction status: pending, completed, failed, refunded

### 9.2 Referential Integrity

- All foreign keys must have valid references
- Cascade deletes where appropriate
- Restrict deletes for critical relationships

### 9.3 Data Quality Rules

- Required fields must not be null
- Dates must be valid and logical (e.g., end_date > start_date)
- Numeric values within acceptable ranges
- Enum values from predefined lists

---

## Document Status

**Status**: Draft - Awaiting Approval  
**Prepared By**: AI Development Assistant  
**Date**: 2026-03-04

---
