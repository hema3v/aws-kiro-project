# Business Logic Design
## Unit 3: Storage Management Service - Functional Design

**Version**: 1.0  
**Date**: 2026-03-04  
**Unit**: Storage Management Service

---

## 1. Core Business Rules

### 1.1 Data Routing Rules

**BR-001: Entity Type Routing**
- **Rule**: Each entity type must have a default target database
- **Logic**: If no specific routing rule matches, use entity type default
- **Priority**: Low (fallback rule)

**BR-002: Time-Series Data Routing**
- **Rule**: All time-stamped data with retention > 30 days routes to TimescaleDB
- **Logic**: `IF is_time_series = true AND retention_days > 30 THEN route_to = 'timescaledb'`
- **Priority**: High

**BR-003: Historical Data Archival**
- **Rule**: Data older than 90 days routes to data warehouse
- **Logic**: `IF data_age_days > 90 THEN route_to = 'snowflake'`
- **Priority**: Medium

**BR-004: High-Value Transaction Dual Write**
- **Rule**: Transactions > $10,000 write to both PostgreSQL and Snowflake
- **Logic**: `IF entity_type = 'transaction' AND amount > 10000 THEN dual_write(['postgresql', 'snowflake'])`
- **Priority**: High

**BR-005: Audit Trail Requirement**
- **Rule**: All customer and transaction data must have lineage tracking
- **Logic**: `IF entity_type IN ['customer', 'transaction'] THEN track_lineage = true`
- **Priority**: Critical

---

### 1.2 Data Validation Rules

**BR-006: Required Fields Validation**
- **Rule**: All required fields must be present and non-null
- **Logic**: Validate against schema definition before write
- **Action**: Reject write if validation fails

**BR-007: Data Type Validation**
- **Rule**: Field values must match defined data types
- **Logic**: Type checking based on schema
- **Action**: Reject write if type mismatch

**BR-008: Foreign Key Validation**
- **Rule**: Foreign key references must exist in target database
- **Logic**: Check existence before insert (PostgreSQL only)
- **Action**: Reject write if foreign key invalid

**BR-009: Unique Constraint Validation**
- **Rule**: Unique fields must not have duplicates
- **Logic**: Check for existing value before insert
- **Action**: Update existing record (upsert) or reject

---

### 1.3 Write Strategy Rules

**BR-010: Batch Size Optimization**
- **Rule**: Batch size adapts based on database type and load
- **Logic**:
  ```
  IF database = 'postgresql' AND current_load < 50% THEN batch_size = 1000
  ELSE IF database = 'postgresql' AND current_load >= 50% THEN batch_size = 500
  ```
- **Priority**: Medium

**BR-011: Write Priority**
- **Rule**: Critical data writes have higher priority
- **Logic**: `IF entity_type IN ['transaction', 'customer'] THEN priority = 'high'`
- **Action**: Process high-priority writes first

**BR-012: Retry Logic**
- **Rule**: Transient failures trigger automatic retry
- **Logic**: Retry up to 3 times with exponential backoff
- **Retryable Errors**: Connection timeout, temporary unavailability
- **Non-Retryable Errors**: Constraint violation, schema mismatch

---

### 1.4 Data Lifecycle Rules

**BR-013: Data Retention**
- **Rule**: Each entity type has a defined retention period
- **Logic**:
  ```
  - audit_logs: 2 years
  - raw_data: 30 days
  - transactions: 7 years (compliance)
  - metrics: 90 days (TimescaleDB), then archive to Snowflake
  ```
- **Action**: Automatic deletion or archival after retention period

**BR-014: Data Archival**
- **Rule**: Old data moves from operational to analytical storage
- **Logic**: `IF data_age > retention_period THEN archive_to_warehouse()`
- **Schedule**: Daily batch job

**BR-015: Data Purging**
- **Rule**: Expired data is permanently deleted
- **Logic**: `IF data_age > retention_period + grace_period THEN delete()`
- **Grace Period**: 30 days after retention period

---

## 2. Business Workflows

### 2.1 Standard Write Workflow

```
1. Receive Data from Kafka
   ├─ Topic: data-ready-for-storage
   └─ Parse message

2. Validate Data
   ├─ Check schema exists
   ├─ Validate required fields
   ├─ Validate data types
   └─ Check business rules

3. Determine Routing
   ├─ Apply routing rules (priority order)
   ├─ Identify target database(s)
   └─ Check database availability

4. Prepare Write
   ├─ Transform data to target schema
   ├─ Add to batch buffer
   └─ Wait for batch size or timeout

5. Execute Write
   ├─ Begin transaction (if applicable)
   ├─ Write to target database
   ├─ Commit transaction
   └─ Record success metrics

6. Track Lineage
   ├─ Create/update lineage record
   ├─ Add storage event to history
   └─ Update current location

7. Acknowledge Kafka Message
   └─ Commit offset

Error Path:
├─ Validation Failure → Send to error collection
├─ Write Failure → Retry logic → Dead letter queue
└─ Lineage Failure → Log error (don't block write)
```

### 2.2 Dual Write Workflow

```
1. Identify Dual Write Requirement
   └─ Based on business rules (e.g., high-value transaction)

2. Prepare Primary Write
   └─ Target: PostgreSQL (operational database)

3. Execute Primary Write
   ├─ Write to PostgreSQL
   └─ IF success THEN continue ELSE abort

4. Prepare Secondary Write
   └─ Target: Snowflake (analytical database)

5. Execute Secondary Write (Async)
   ├─ Write to Snowflake
   └─ IF failure THEN log error (don't rollback primary)

6. Track Lineage
   └─ Record both storage locations

Note: Secondary write failure doesn't affect primary write
```

### 2.3 Batch Write Workflow

```
1. Accumulate Records
   ├─ Add incoming records to batch buffer
   ├─ Group by target database and entity type
   └─ Maintain insertion order

2. Trigger Batch Write
   ├─ Condition 1: Batch size reached (e.g., 1000 records)
   └─ Condition 2: Timeout expired (e.g., 5 seconds)

3. Prepare Batch
   ├─ Sort records (if needed)
   ├─ Validate all records
   └─ Remove invalid records

4. Execute Batch Write
   ├─ PostgreSQL: Use batch insert or COPY
   ├─ MongoDB: Use bulkWrite
   ├─ TimescaleDB: Use batch insert
   └─ Snowflake: Stage to S3 → COPY

5. Handle Partial Failures
   ├─ Identify failed records
   ├─ Retry failed records individually
   └─ Send persistent failures to DLQ

6. Update Metrics
   ├─ Record batch size
   ├─ Record success/failure counts
   └─ Record duration
```

### 2.4 Schema Evolution Workflow

```
1. Receive Schema Update Request
   └─ New schema version submitted

2. Validate Schema Changes
   ├─ Check backward compatibility
   ├─ Identify breaking changes
   └─ Validate field definitions

3. Create Migration Plan
   ├─ Generate ALTER TABLE statements
   ├─ Plan data migration (if needed)
   └─ Estimate downtime

4. Execute Migration
   ├─ Pause writes to affected table
   ├─ Apply schema changes
   ├─ Migrate existing data (if needed)
   └─ Resume writes

5. Update Schema Registry
   ├─ Register new schema version
   ├─ Mark old version as deprecated
   └─ Set migration timestamp

6. Notify Dependent Services
   └─ Publish schema update event
```

---

## 3. Business Logic Components

### 3.1 Routing Engine

**Purpose**: Determine target database(s) for each record

**Inputs**:
- ProcessedDataRecord
- Routing rules configuration
- Database health status

**Logic**:
```java
public class RoutingEngine {
    
    public List<DatabaseTarget> determineTargets(ProcessedDataRecord record) {
        List<DatabaseTarget> targets = new ArrayList<>();
        
        // Step 1: Apply entity type default
        DatabaseTarget defaultTarget = getDefaultTarget(record.getEntityType());
        targets.add(defaultTarget);
        
        // Step 2: Apply business rules (priority order)
        List<RoutingRule> applicableRules = getApplicableRules(record);
        for (RoutingRule rule : applicableRules) {
            if (rule.evaluate(record)) {
                targets.addAll(rule.getTargets());
            }
        }
        
        // Step 3: Remove duplicates
        targets = deduplicateTargets(targets);
        
        // Step 4: Check database availability
        targets = filterAvailableTargets(targets);
        
        return targets;
    }
    
    private boolean evaluateRule(RoutingRule rule, ProcessedDataRecord record) {
        // Evaluate rule conditions against record
        // Example: rule.condition = "amount > 10000"
        return expressionEvaluator.evaluate(rule.getCondition(), record);
    }
}
```

**Outputs**:
- List of target databases
- Routing decision metadata

---

### 3.2 Validation Engine

**Purpose**: Validate data against schema and business rules

**Inputs**:
- ProcessedDataRecord
- Schema definition
- Validation rules

**Logic**:
```java
public class ValidationEngine {
    
    public ValidationResult validate(ProcessedDataRecord record, Schema schema) {
        ValidationResult result = new ValidationResult();
        
        // Required fields validation
        for (Field field : schema.getRequiredFields()) {
            if (!record.hasField(field.getName()) || record.getField(field.getName()) == null) {
                result.addError("Missing required field: " + field.getName());
            }
        }
        
        // Data type validation
        for (Field field : schema.getFields()) {
            if (record.hasField(field.getName())) {
                Object value = record.getField(field.getName());
                if (!isValidType(value, field.getType())) {
                    result.addError("Invalid type for field: " + field.getName());
                }
            }
        }
        
        // Business rule validation
        for (ValidationRule rule : getValidationRules(schema.getEntityType())) {
            if (!rule.evaluate(record)) {
                result.addError("Business rule violation: " + rule.getName());
            }
        }
        
        return result;
    }
}
```

**Outputs**:
- ValidationResult (pass/fail)
- List of validation errors

---

### 3.3 Write Coordinator

**Purpose**: Coordinate writes to multiple databases

**Inputs**:
- ProcessedDataRecord
- List of target databases
- Write strategy (single, dual, batch)

**Logic**:
```java
public class WriteCoordinator {
    
    public WriteResult coordinateWrite(ProcessedDataRecord record, List<DatabaseTarget> targets) {
        WriteResult result = new WriteResult();
        
        if (targets.size() == 1) {
            // Single write
            result = executeSingleWrite(record, targets.get(0));
        } else {
            // Dual/multi write
            result = executeDualWrite(record, targets);
        }
        
        // Track lineage (async)
        if (result.isSuccess()) {
            lineageTracker.trackStorage(record, result.getStorageLocations());
        }
        
        return result;
    }
    
    private WriteResult executeDualWrite(ProcessedDataRecord record, List<DatabaseTarget> targets) {
        // Primary write (synchronous)
        DatabaseTarget primary = targets.get(0);
        WriteResult primaryResult = databaseWriters.get(primary.getType()).write(record);
        
        if (!primaryResult.isSuccess()) {
            return primaryResult; // Abort if primary fails
        }
        
        // Secondary writes (asynchronous)
        for (int i = 1; i < targets.size(); i++) {
            DatabaseTarget secondary = targets.get(i);
            CompletableFuture.runAsync(() -> {
                WriteResult secondaryResult = databaseWriters.get(secondary.getType()).write(record);
                if (!secondaryResult.isSuccess()) {
                    logger.error("Secondary write failed: " + secondary.getType());
                }
            });
        }
        
        return primaryResult;
    }
}
```

**Outputs**:
- WriteResult (success/failure)
- Storage locations
- Error details (if failed)

---

### 3.4 Batch Manager

**Purpose**: Manage batch accumulation and flushing

**Inputs**:
- Stream of ProcessedDataRecords
- Batch configuration

**Logic**:
```java
public class BatchManager {
    
    private Map<String, List<ProcessedDataRecord>> batches = new ConcurrentHashMap<>();
    private Map<String, ScheduledFuture<?>> flushTimers = new ConcurrentHashMap<>();
    
    public void addToBatch(ProcessedDataRecord record, DatabaseTarget target) {
        String batchKey = target.getType() + ":" + record.getEntityType();
        
        batches.computeIfAbsent(batchKey, k -> new ArrayList<>()).add(record);
        
        // Schedule flush timer if not already scheduled
        flushTimers.computeIfAbsent(batchKey, k -> 
            scheduler.schedule(() -> flushBatch(batchKey), batchTimeout, TimeUnit.MILLISECONDS)
        );
        
        // Check if batch size reached
        if (batches.get(batchKey).size() >= batchSize) {
            flushBatch(batchKey);
        }
    }
    
    private void flushBatch(String batchKey) {
        List<ProcessedDataRecord> batch = batches.remove(batchKey);
        ScheduledFuture<?> timer = flushTimers.remove(batchKey);
        
        if (timer != null) {
            timer.cancel(false);
        }
        
        if (batch != null && !batch.isEmpty()) {
            executeBatchWrite(batch, batchKey);
        }
    }
}
```

**Outputs**:
- Batch write execution
- Batch metrics

---

## 4. Business Calculations

### 4.1 Storage Cost Calculation

**Purpose**: Estimate storage costs for routing decisions

**Formula**:
```
storage_cost = (data_size_gb * cost_per_gb_per_month * retention_months) + 
               (query_frequency * cost_per_query)

Where:
- PostgreSQL: $0.10/GB/month, $0.001/query
- MongoDB: $0.25/GB/month, $0.002/query
- TimescaleDB: $0.15/GB/month, $0.001/query
- Snowflake: $0.02/GB/month, $0.005/query (but higher query cost)
```

**Usage**: Inform routing decisions for cost optimization

---

### 4.2 Write Performance Score

**Purpose**: Measure write performance by database

**Formula**:
```
performance_score = (success_rate * 0.4) + 
                   ((1 - normalized_latency) * 0.3) + 
                   ((1 - error_rate) * 0.3)

Where:
- success_rate = successful_writes / total_writes
- normalized_latency = current_latency / target_latency
- error_rate = failed_writes / total_writes
```

**Usage**: Identify performance issues, trigger alerts

---

### 4.3 Data Quality Score

**Purpose**: Calculate data quality for stored records

**Formula**:
```
quality_score = (completeness * 0.3) + 
               (accuracy * 0.3) + 
               (consistency * 0.2) + 
               (timeliness * 0.2)

Where:
- completeness = non_null_fields / total_fields
- accuracy = valid_values / total_values
- consistency = consistent_records / total_records
- timeliness = on_time_records / total_records
```

**Usage**: Track data quality metrics, trigger alerts

---

## 5. Business Constraints

### 5.1 Data Integrity Constraints

- **Primary Key Uniqueness**: No duplicate primary keys allowed
- **Foreign Key Validity**: All foreign keys must reference existing records
- **Not Null Constraints**: Required fields cannot be null
- **Check Constraints**: Field values must satisfy defined conditions
- **Unique Constraints**: Unique fields cannot have duplicates

### 5.2 Performance Constraints

- **Write Latency**: < 500ms for 95th percentile
- **Throughput**: Minimum 10,000 records/second
- **Batch Size**: Maximum 10,000 records per batch
- **Connection Pool**: Maximum 50 connections per database
- **Memory Usage**: Maximum 2GB for batch buffers

### 5.3 Compliance Constraints

- **Data Retention**: Must comply with regulatory requirements
- **Audit Trail**: Complete lineage for all customer and transaction data
- **Data Residency**: Data must stay in specified geographic region
- **Encryption**: All sensitive data encrypted at rest and in transit
- **Access Control**: Role-based access to databases

---

## Document Status

**Status**: Complete - Awaiting Approval  
**Prepared By**: AI Development Assistant  
**Date**: 2026-03-04  
**Next**: NFR Requirements

---
