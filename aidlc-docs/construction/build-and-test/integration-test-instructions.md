# Integration Test Instructions
## Data Integration and Analytics Platform

**Date**: 2026-03-04  
**Status**: Ready for Testing  
**Test Framework**: JUnit 5 + TestContainers (Backend), Cypress (Frontend E2E)

---

## Overview

Integration tests verify that multiple components, services, and systems work together correctly. These tests validate end-to-end data flows, service interactions, and system integration points.

**Test Scope**: Service-to-service communication, database interactions, message queue integration, API contracts

---

## Test Strategy

### Integration Test Types
1. **Service Integration**: Test interactions between microservices
2. **Database Integration**: Test data persistence and retrieval
3. **Message Queue Integration**: Test Kafka message flow
4. **API Contract Testing**: Verify API contracts between services
5. **End-to-End Flows**: Test complete business workflows

### Test Environment
- **Containers**: Docker + TestContainers for infrastructure
- **Databases**: PostgreSQL, MongoDB, TimescaleDB, Redis
- **Message Queue**: Kafka
- **Service Mesh**: All 6 services running

---

## Infrastructure Setup

### Docker Compose for Integration Tests
```yaml
# docker-compose.integration-test.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_pass
    ports:
      - "5432:5432"
  
  mongodb:
    image: mongo:6
    environment:
      MONGO_INITDB_ROOT_USERNAME: test_user
      MONGO_INITDB_ROOT_PASSWORD: test_pass
    ports:
      - "27017:27017"
  
  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_DB: test_timeseries
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_pass
    ports:
      - "5433:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"
  
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
```

### Start Test Infrastructure
```bash
# Start all infrastructure services
docker-compose -f docker-compose.integration-test.yml up -d

# Wait for services to be ready
./scripts/wait-for-services.sh

# Verify services are running
docker-compose -f docker-compose.integration-test.yml ps
```

---

## Integration Test 1: Data Ingestion to Storage Flow

### Test Objective
Verify complete data flow from ingestion through processing to storage.

### Test Scenario
```
External Source → Unit 1 (Ingestion) → Kafka → Unit 2 (Processing) → Kafka → Unit 3 (Storage) → Database
```

### Test Steps

#### 1. Setup Test Data
```java
@SpringBootTest
@Testcontainers
class DataIngestionToStorageIntegrationTest {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Autowired
    private DataIngestionService ingestionService;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    void testCompleteDataFlow() {
        // 1. Ingest data
        IngestionRequest request = IngestionRequest.builder()
            .sourceType("API")
            .sourceId("test-api-1")
            .data(createTestData())
            .build();
        
        IngestionResponse response = ingestionService.ingest(request);
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        
        // 2. Verify message in Kafka (raw-data-ingested topic)
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                ConsumerRecords<String, String> records = consumeFromKafka("raw-data-ingested");
                assertThat(records).isNotEmpty();
            });
        
        // 3. Wait for processing (data-ready-for-storage topic)
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                ConsumerRecords<String, String> records = consumeFromKafka("data-ready-for-storage");
                assertThat(records).isNotEmpty();
            });
        
        // 4. Verify data in PostgreSQL
        await().atMost(Duration.ofSeconds(10))
            .untilAsserted(() -> {
                Integer count = jdbcTemplate.queryForObject(
                    "SELECT COUNT(*) FROM transactions WHERE source_id = ?",
                    Integer.class,
                    "test-api-1"
                );
                assertThat(count).isGreaterThan(0);
            });
        
        // 5. Verify data lineage
        List<LineageRecord> lineage = lineageService.getLineage(response.getRecordId());
        assertThat(lineage).hasSize(3); // Ingestion -> Processing -> Storage
    }
}
```

### Run Test
```bash
cd data-ingestion-service
mvn test -Dtest=DataIngestionToStorageIntegrationTest
```

### Expected Results
- Data successfully ingested
- Messages published to Kafka topics
- Data processed and validated
- Data stored in PostgreSQL
- Complete lineage tracked

---

## Integration Test 2: Multi-Database Storage Routing

### Test Objective
Verify that data is correctly routed to appropriate databases based on routing rules.

### Test Scenario
```
Storage Service receives data → Routing logic evaluates → Data written to correct database(s)
```

### Test Steps

```java
@SpringBootTest
@Testcontainers
class MultiDatabaseRoutingIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6");
    
    @Container
    static GenericContainer<?> timescaledb = new GenericContainer<>("timescale/timescaledb:latest-pg15")
        .withExposedPorts(5432);
    
    @Autowired
    private StorageService storageService;
    
    @Test
    void testTransactionRoutingToPostgres() {
        // Transaction data should go to PostgreSQL
        StorageRequest request = StorageRequest.builder()
            .entityType("TRANSACTION")
            .data(createTransactionData())
            .build();
        
        StorageResponse response = storageService.store(request);
        
        assertThat(response.getTargetDatabase()).isEqualTo("POSTGRESQL");
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        
        // Verify in PostgreSQL
        Transaction transaction = transactionRepository.findById(response.getRecordId());
        assertThat(transaction).isNotNull();
    }
    
    @Test
    void testCustomerDocumentRoutingToMongoDB() {
        // Customer profile should go to MongoDB
        StorageRequest request = StorageRequest.builder()
            .entityType("CUSTOMER_PROFILE")
            .data(createCustomerProfileData())
            .build();
        
        StorageResponse response = storageService.store(request);
        
        assertThat(response.getTargetDatabase()).isEqualTo("MONGODB");
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        
        // Verify in MongoDB
        CustomerProfile profile = customerProfileRepository.findById(response.getRecordId());
        assertThat(profile).isNotNull();
    }
    
    @Test
    void testTimeSeriesRoutingToTimescaleDB() {
        // Sensor data should go to TimescaleDB
        StorageRequest request = StorageRequest.builder()
            .entityType("SENSOR_DATA")
            .data(createSensorData())
            .build();
        
        StorageResponse response = storageService.store(request);
        
        assertThat(response.getTargetDatabase()).isEqualTo("TIMESCALEDB");
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
    }
    
    @Test
    void testDualWriteForHighValueTransaction() {
        // High-value transaction should be written to both PostgreSQL and Snowflake
        StorageRequest request = StorageRequest.builder()
            .entityType("TRANSACTION")
            .data(createHighValueTransactionData(15000.00)) // > $10,000
            .build();
        
        StorageResponse response = storageService.store(request);
        
        assertThat(response.getTargetDatabases()).contains("POSTGRESQL", "SNOWFLAKE");
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
    }
}
```

### Run Test
```bash
cd storage-management-service
mvn test -Dtest=MultiDatabaseRoutingIntegrationTest
```

### Expected Results
- Transactions routed to PostgreSQL
- Customer profiles routed to MongoDB
- Time-series data routed to TimescaleDB
- High-value transactions dual-written
- All writes successful

---

## Integration Test 3: Data Processing Pipeline

### Test Objective
Verify data validation, cleansing, transformation, and quality scoring.

### Test Scenario
```
Raw data → Validation → Cleansing → Transformation → Quality Scoring → Output
```

### Test Steps

```java
@SpringBootTest
@Testcontainers
class DataProcessingPipelineIntegrationTest {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );
    
    @Autowired
    private ProcessingPipeline pipeline;
    
    @Test
    void testValidDataProcessing() {
        // Valid data should pass all stages
        RawData rawData = createValidRawData();
        
        ProcessingResult result = pipeline.process(rawData);
        
        assertThat(result.getStatus()).isEqualTo("SUCCESS");
        assertThat(result.getQualityScore()).isGreaterThanOrEqualTo(85);
        assertThat(result.getValidationErrors()).isEmpty();
        assertThat(result.getProcessedData()).isNotNull();
    }
    
    @Test
    void testInvalidDataRejection() {
        // Invalid data should be rejected
        RawData rawData = createInvalidRawData();
        
        ProcessingResult result = pipeline.process(rawData);
        
        assertThat(result.getStatus()).isEqualTo("REJECTED");
        assertThat(result.getValidationErrors()).isNotEmpty();
        assertThat(result.getQualityScore()).isLessThan(85);
    }
    
    @Test
    void testDataCleansing() {
        // Data with issues should be cleansed
        RawData rawData = RawData.builder()
            .field1("  value with spaces  ")
            .field2(null)
            .field3("UPPERCASE")
            .build();
        
        ProcessingResult result = pipeline.process(rawData);
        
        assertThat(result.getProcessedData().getField1()).isEqualTo("value with spaces");
        assertThat(result.getProcessedData().getField2()).isEqualTo("DEFAULT");
        assertThat(result.getProcessedData().getField3()).isEqualTo("uppercase");
    }
    
    @Test
    void testDeduplication() {
        // Duplicate data should be detected
        RawData data1 = createDuplicateData();
        RawData data2 = createDuplicateData();
        
        ProcessingResult result1 = pipeline.process(data1);
        ProcessingResult result2 = pipeline.process(data2);
        
        assertThat(result1.getStatus()).isEqualTo("SUCCESS");
        assertThat(result2.getStatus()).isEqualTo("DUPLICATE");
        assertThat(result2.getDuplicateOf()).isEqualTo(result1.getRecordId());
    }
    
    @Test
    void testBusinessRuleApplication() {
        // Business rules should be applied
        RawData rawData = createDataForRuleTest();
        
        ProcessingResult result = pipeline.process(rawData);
        
        assertThat(result.getAppliedRules()).contains("DISCOUNT_RULE", "TAX_RULE");
        assertThat(result.getProcessedData().getDiscount()).isEqualTo(10.0);
        assertThat(result.getProcessedData().getTax()).isEqualTo(8.5);
    }
}
```

### Run Test
```bash
cd data-processing-pipeline
mvn test -Dtest=DataProcessingPipelineIntegrationTest
```

### Expected Results
- Valid data processed successfully
- Invalid data rejected with errors
- Data cleansed correctly
- Duplicates detected
- Business rules applied
- Quality scores calculated

---

## Integration Test 4: Analytics API to Database

### Test Objective
Verify analytics queries across multiple databases with caching.

### Test Scenario
```
API Request → Query Engine → Multi-Database Query → Cache → Response
```

### Test Steps

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
class AnalyticsApiIntegrationTest {
    
    @LocalServerPort
    private int port;
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);
    
    @Test
    void testRevenueAnalyticsQuery() {
        // Setup test data
        setupRevenueTestData();
        
        // Query revenue analytics
        String url = "http://localhost:" + port + "/api/analytics/revenue?period=MONTHLY&year=2026";
        
        ResponseEntity<RevenueAnalyticsResponse> response = restTemplate.getForEntity(
            url,
            RevenueAnalyticsResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getTotalRevenue()).isGreaterThan(0);
        assertThat(response.getBody().getMonthlyData()).hasSize(12);
    }
    
    @Test
    void testCustomerAnalyticsQuery() {
        // Setup test data
        setupCustomerTestData();
        
        // Query customer analytics
        String url = "http://localhost:" + port + "/api/analytics/customers?segment=PREMIUM";
        
        ResponseEntity<CustomerAnalyticsResponse> response = restTemplate.getForEntity(
            url,
            CustomerAnalyticsResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getTotalCustomers()).isGreaterThan(0);
        assertThat(response.getBody().getSegments()).containsKey("PREMIUM");
    }
    
    @Test
    void testCachingBehavior() {
        // First request (cache miss)
        long start1 = System.currentTimeMillis();
        ResponseEntity<RevenueAnalyticsResponse> response1 = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/analytics/revenue?period=MONTHLY&year=2026",
            RevenueAnalyticsResponse.class
        );
        long duration1 = System.currentTimeMillis() - start1;
        
        // Second request (cache hit)
        long start2 = System.currentTimeMillis();
        ResponseEntity<RevenueAnalyticsResponse> response2 = restTemplate.getForEntity(
            "http://localhost:" + port + "/api/analytics/revenue?period=MONTHLY&year=2026",
            RevenueAnalyticsResponse.class
        );
        long duration2 = System.currentTimeMillis() - start2;
        
        // Cache hit should be significantly faster
        assertThat(duration2).isLessThan(duration1 / 2);
        assertThat(response1.getBody()).isEqualTo(response2.getBody());
    }
    
    @Test
    void testMultiDatabaseAggregation() {
        // Query that requires data from multiple databases
        String url = "http://localhost:" + port + "/api/analytics/comprehensive?includeAll=true";
        
        ResponseEntity<ComprehensiveAnalyticsResponse> response = restTemplate.getForEntity(
            url,
            ComprehensiveAnalyticsResponse.class
        );
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getTransactionData()).isNotNull(); // From PostgreSQL
        assertThat(response.getBody().getCustomerProfiles()).isNotNull(); // From MongoDB
        assertThat(response.getBody().getTimeSeriesMetrics()).isNotNull(); // From TimescaleDB
    }
}
```

### Run Test
```bash
cd analytics-api-service
mvn test -Dtest=AnalyticsApiIntegrationTest
```

### Expected Results
- Revenue analytics queries successful
- Customer analytics queries successful
- Caching improves performance
- Multi-database aggregation works
- Response times within SLA

---

## Integration Test 5: Monitoring and Audit Integration

### Test Objective
Verify that all services correctly integrate with monitoring and audit service.

### Test Scenario
```
Service Operation → Log Entry → Metrics → Trace → Lineage → Audit Trail
```

### Test Steps

```java
@SpringBootTest
@Testcontainers
class MonitoringAuditIntegrationTest {
    
    @Container
    static MongoDBContainer mongodb = new MongoDBContainer("mongo:6");
    
    @Container
    static GenericContainer<?> elasticsearch = new GenericContainer<>(
        "docker.elastic.co/elasticsearch/elasticsearch:8.8.0"
    ).withExposedPorts(9200);
    
    @Autowired
    private AuditService auditService;
    
    @Autowired
    private LineageService lineageService;
    
    @Autowired
    private MetricsService metricsService;
    
    @Test
    void testAuditTrailCreation() {
        // Perform operation
        String userId = "test-user";
        String action = "CREATE_TRANSACTION";
        Map<String, Object> details = Map.of("amount", 1000.00);
        
        auditService.logAction(userId, action, details);
        
        // Verify audit trail
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                List<AuditLog> logs = auditService.getAuditLogs(userId, action);
                assertThat(logs).isNotEmpty();
                assertThat(logs.get(0).getAction()).isEqualTo(action);
            });
    }
    
    @Test
    void testLineageTracking() {
        // Create lineage chain
        String recordId = UUID.randomUUID().toString();
        
        lineageService.trackIngestion(recordId, "API", "test-api-1");
        lineageService.trackProcessing(recordId, "VALIDATION", "SUCCESS");
        lineageService.trackStorage(recordId, "POSTGRESQL", "transactions");
        
        // Verify complete lineage
        List<LineageRecord> lineage = lineageService.getLineage(recordId);
        
        assertThat(lineage).hasSize(3);
        assertThat(lineage.get(0).getStage()).isEqualTo("INGESTION");
        assertThat(lineage.get(1).getStage()).isEqualTo("PROCESSING");
        assertThat(lineage.get(2).getStage()).isEqualTo("STORAGE");
    }
    
    @Test
    void testMetricsCollection() {
        // Record metrics
        metricsService.recordIngestionMetric("API", 100, 5000);
        metricsService.recordProcessingMetric("VALIDATION", 100, 95, 3000);
        metricsService.recordStorageMetric("POSTGRESQL", 100, 2000);
        
        // Verify metrics
        await().atMost(Duration.ofSeconds(5))
            .untilAsserted(() -> {
                MetricsSummary summary = metricsService.getSummary();
                assertThat(summary.getIngestionCount()).isGreaterThan(0);
                assertThat(summary.getProcessingCount()).isGreaterThan(0);
                assertThat(summary.getStorageCount()).isGreaterThan(0);
            });
    }
}
```

### Run Test
```bash
cd monitoring-audit-service
mvn test -Dtest=MonitoringAuditIntegrationTest
```

### Expected Results
- Audit trails created correctly
- Lineage tracked end-to-end
- Metrics collected from all services
- Logs searchable in Elasticsearch

---

## Integration Test 6: End-to-End Dashboard Flow

### Test Objective
Verify complete flow from data ingestion to dashboard display.

### Test Scenario
```
Ingest Data → Process → Store → Analytics API → Dashboard Display → Real-time Update
```

### Test Steps (Cypress E2E)

```javascript
// cypress/integration/end-to-end-flow.spec.js

describe('End-to-End Data Flow', () => {
  before(() => {
    // Setup: Ingest test data via API
    cy.request('POST', 'http://localhost:8080/api/ingestion/ingest', {
      sourceType: 'API',
      sourceId: 'test-api-e2e',
      data: {
        transactions: [
          { id: 1, amount: 1000, customer: 'Customer A' },
          { id: 2, amount: 2000, customer: 'Customer B' }
        ]
      }
    }).then((response) => {
      expect(response.status).to.eq(200);
    });
    
    // Wait for processing
    cy.wait(5000);
  });
  
  it('should display ingested data in dashboard', () => {
    // Login
    cy.visit('http://localhost:3000');
    cy.get('[data-testid="username"]').type('test-user');
    cy.get('[data-testid="password"]').type('test-password');
    cy.get('[data-testid="login-button"]').click();
    
    // Navigate to dashboard
    cy.url().should('include', '/dashboard');
    
    // Verify revenue KPI updated
    cy.get('[data-testid="total-revenue"]').should('contain', '3000');
    
    // Verify transaction count
    cy.get('[data-testid="transaction-count"]').should('contain', '2');
    
    // Verify chart displays data
    cy.get('[data-testid="revenue-chart"]').should('be.visible');
    
    // Verify table shows transactions
    cy.get('[data-testid="transaction-table"]').within(() => {
      cy.get('tbody tr').should('have.length', 2);
    });
  });
  
  it('should receive real-time updates', () => {
    // Open dashboard
    cy.visit('http://localhost:3000/dashboard');
    
    // Get initial revenue
    cy.get('[data-testid="total-revenue"]').invoke('text').then((initialRevenue) => {
      // Trigger new data ingestion
      cy.request('POST', 'http://localhost:8080/api/ingestion/ingest', {
        sourceType: 'API',
        sourceId: 'test-api-realtime',
        data: {
          transactions: [
            { id: 3, amount: 5000, customer: 'Customer C' }
          ]
        }
      });
      
      // Verify dashboard updates in real-time (via WebSocket)
      cy.get('[data-testid="total-revenue"]', { timeout: 10000 })
        .should('not.contain', initialRevenue);
    });
  });
  
  it('should export data successfully', () => {
    cy.visit('http://localhost:3000/dashboard');
    
    // Click export button
    cy.get('[data-testid="export-button"]').click();
    
    // Select CSV format
    cy.get('[data-testid="export-format"]').select('CSV');
    
    // Confirm export
    cy.get('[data-testid="confirm-export"]').click();
    
    // Verify download started
    cy.get('[data-testid="export-status"]').should('contain', 'Download started');
  });
});
```

### Run Test
```bash
cd dashboard-frontend

# Start all services first
docker-compose up -d

# Run Cypress tests
npm run cypress:run

# Or open Cypress UI
npm run cypress:open
```

### Expected Results
- Data flows from ingestion to dashboard
- Dashboard displays correct data
- Real-time updates work via WebSocket
- Export functionality works
- All interactions smooth and responsive

---

## API Contract Testing

### Test Objective
Verify API contracts between services using Pact.

### Producer Test (Analytics API)
```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Provider("analytics-api-service")
@PactBroker(host = "localhost", port = "9292")
class AnalyticsApiContractTest {
    
    @LocalServerPort
    private int port;
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        context.verifyInteraction();
    }
    
    @BeforeEach
    void before(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }
    
    @State("revenue data exists")
    void revenueDataExists() {
        // Setup test data for contract verification
        setupRevenueTestData();
    }
}
```

### Consumer Test (Dashboard Frontend)
```javascript
// pact/analytics-api.pact.spec.js
import { Pact } from '@pact-foundation/pact';

describe('Analytics API Contract', () => {
  const provider = new Pact({
    consumer: 'dashboard-frontend',
    provider: 'analytics-api-service',
    port: 1234,
  });
  
  before(() => provider.setup());
  after(() => provider.finalize());
  
  describe('GET /api/analytics/revenue', () => {
    before(() => {
      return provider.addInteraction({
        state: 'revenue data exists',
        uponReceiving: 'a request for revenue analytics',
        withRequest: {
          method: 'GET',
          path: '/api/analytics/revenue',
          query: 'period=MONTHLY&year=2026',
        },
        willRespondWith: {
          status: 200,
          headers: { 'Content-Type': 'application/json' },
          body: {
            totalRevenue: 100000,
            monthlyData: Matchers.eachLike({
              month: 'January',
              revenue: 10000,
            }),
          },
        },
      });
    });
    
    it('returns revenue analytics', async () => {
      const response = await fetch('http://localhost:1234/api/analytics/revenue?period=MONTHLY&year=2026');
      const data = await response.json();
      
      expect(data.totalRevenue).toBe(100000);
      expect(data.monthlyData).toHaveLength(12);
    });
  });
});
```

---

## Running All Integration Tests

### Sequential Execution
```bash
#!/bin/bash
# run-all-integration-tests.sh

set -e

echo "Starting integration test infrastructure..."
docker-compose -f docker-compose.integration-test.yml up -d

echo "Waiting for services to be ready..."
./scripts/wait-for-services.sh

echo "Running integration tests..."

# Backend services
services=("monitoring-audit-service" "storage-management-service" "data-ingestion-service" "data-processing-pipeline" "analytics-api-service")

for service in "${services[@]}"; do
  echo "Testing $service..."
  cd $service
  mvn verify -P integration-test
  cd ..
done

# Frontend E2E tests
echo "Running E2E tests..."
cd dashboard-frontend
npm run cypress:run
cd ..

echo "Stopping infrastructure..."
docker-compose -f docker-compose.integration-test.yml down

echo "All integration tests complete!"
```

---

## Test Results and Reporting

### Generate Test Reports
```bash
# Backend: Surefire reports
mvn surefire-report:report

# View report
open target/site/surefire-report.html

# Frontend: Cypress reports
npm run cypress:run --reporter mochawesome

# View report
open cypress/reports/mochawesome.html
```

---

## Integration Test Checklist

- [ ] Infrastructure services running
- [ ] All service-to-service integrations tested
- [ ] Database integrations verified
- [ ] Kafka message flows tested
- [ ] API contracts validated
- [ ] End-to-end flows working
- [ ] Real-time updates functioning
- [ ] Error scenarios handled
- [ ] Performance within acceptable limits
- [ ] Test reports generated

---

## Next Steps

After successful integration tests:
1. Proceed to Performance Test Instructions
2. Fix any integration issues
3. Review and optimize slow integrations
4. Document any environment-specific setup
5. Update API contracts if needed

---

## Document Status

**Status**: Complete  
**Date**: 2026-03-04  
**Version**: 1.0.0  
**Next**: performance-test-instructions.md

---
