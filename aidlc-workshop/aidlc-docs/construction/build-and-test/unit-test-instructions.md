# Unit Test Instructions
## Data Integration and Analytics Platform

**Date**: 2026-03-04  
**Status**: Ready for Testing  
**Test Framework**: JUnit 5 (Backend), Jest + React Testing Library (Frontend)

---

## Overview

This document provides comprehensive unit testing instructions for all 6 units. Unit tests verify individual components, classes, and functions in isolation.

**Coverage Target**: 80%+ for all services

---

## Test Strategy

### Backend Services (JUnit 5)
- **Framework**: JUnit 5 + Mockito + AssertJ
- **Spring Testing**: @SpringBootTest, @WebMvcTest, @DataJpaTest
- **Mocking**: Mockito for dependencies
- **Test Containers**: For database integration tests
- **Coverage Tool**: JaCoCo

### Frontend (Jest + RTL)
- **Framework**: Jest + React Testing Library
- **Mocking**: Jest mocks for API calls
- **Component Testing**: User-centric testing approach
- **Coverage Tool**: Jest built-in coverage

---

## Unit 6: Monitoring & Audit Service Tests

### Location
```
monitoring-audit-service/src/test/java/
```

### Test Categories

#### 1. Service Layer Tests
```bash
# Run all service tests
mvn test -Dtest=*ServiceTest

# Specific tests
mvn test -Dtest=LoggingServiceTest
mvn test -Dtest=MetricsServiceTest
mvn test -Dtest=TracingServiceTest
mvn test -Dtest=LineageServiceTest
mvn test -Dtest=AuditServiceTest
mvn test -Dtest=HealthMonitoringServiceTest
```

**Key Test Cases**:
- Log entry creation and batching
- Metrics collection and export
- Trace span creation and propagation
- Lineage tracking and retrieval
- Audit trail creation and querying
- Health check execution and alerting

#### 2. Repository Layer Tests
```bash
# Run all repository tests
mvn test -Dtest=*RepositoryTest

# Specific tests
mvn test -Dtest=AuditLogRepositoryTest
mvn test -Dtest=LineageRepositoryTest
mvn test -Dtest=MetricsRepositoryTest
```

**Key Test Cases**:
- CRUD operations
- Custom query methods
- Pagination and sorting
- Data integrity constraints

#### 3. Controller Layer Tests
```bash
# Run all controller tests
mvn test -Dtest=*ControllerTest

# Specific tests
mvn test -Dtest=LoggingControllerTest
mvn test -Dtest=MetricsControllerTest
mvn test -Dtest=AuditControllerTest
```

**Key Test Cases**:
- Request validation
- Response formatting
- Error handling
- Authentication/authorization

#### 4. Kafka Consumer Tests
```bash
mvn test -Dtest=*ConsumerTest
```

**Key Test Cases**:
- Message consumption
- Deserialization
- Error handling
- Retry logic

### Run All Unit Tests
```bash
cd monitoring-audit-service

# Run all tests
mvn test

# Run with coverage
mvn test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

### Expected Coverage
- **Service Layer**: 85%+
- **Repository Layer**: 90%+
- **Controller Layer**: 80%+
- **Utility Classes**: 90%+
- **Overall**: 80%+

---

## Unit 3: Storage Management Service Tests

### Location
```
storage-management-service/src/test/java/
```

### Test Categories

#### 1. Routing Service Tests
```bash
mvn test -Dtest=DataRoutingServiceTest
mvn test -Dtest=RoutingStrategyTest
mvn test -Dtest=DatabaseSelectorTest
```

**Key Test Cases**:
- Entity type routing
- Data structure routing
- Query pattern routing
- Multi-factor routing algorithm
- Dual write scenarios
- Routing rule evaluation

#### 2. Database Writer Tests
```bash
mvn test -Dtest=PostgresWriterTest
mvn test -Dtest=MongoDBWriterTest
mvn test -Dtest=TimescaleDBWriterTest
mvn test -Dtest=SnowflakeWriterTest
```

**Key Test Cases**:
- Single record write
- Batch write operations
- Transaction handling
- Error handling and retry
- Connection pooling
- Write performance

#### 3. Schema Management Tests
```bash
mvn test -Dtest=SchemaManagerTest
mvn test -Dtest=SchemaEvolutionTest
mvn test -Dtest=SchemaValidationTest
```

**Key Test Cases**:
- Schema registration
- Schema evolution
- Version management
- Compatibility checking
- Schema validation

#### 4. Lineage Tracking Tests
```bash
mvn test -Dtest=LineageTrackerTest
mvn test -Dtest=LineageQueryServiceTest
```

**Key Test Cases**:
- Lineage record creation
- Lineage chain tracking
- Lineage querying
- Async processing

#### 5. Saga Orchestration Tests
```bash
mvn test -Dtest=SagaOrchestratorTest
mvn test -Dtest=CompensationHandlerTest
```

**Key Test Cases**:
- Saga execution
- Compensation logic
- Failure scenarios
- State management

### Run All Unit Tests
```bash
cd storage-management-service

# Run all tests
mvn test

# Run with coverage
mvn test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

### Expected Coverage
- **Routing Logic**: 90%+
- **Database Writers**: 85%+
- **Schema Management**: 85%+
- **Lineage Tracking**: 80%+
- **Overall**: 80%+

---

## Unit 1: Data Ingestion Service Tests

### Location
```
data-ingestion-service/src/test/java/
```

### Test Categories

#### 1. Connector Tests
```bash
# Database connectors
mvn test -Dtest=PostgresConnectorTest
mvn test -Dtest=MySQLConnectorTest
mvn test -Dtest=MongoDBConnectorTest
mvn test -Dtest=OracleConnectorTest

# API connectors
mvn test -Dtest=RestApiConnectorTest
mvn test -Dtest=SoapApiConnectorTest
mvn test -Dtest=GraphQLConnectorTest

# File processors
mvn test -Dtest=CsvFileProcessorTest
mvn test -Dtest=JsonFileProcessorTest
mvn test -Dtest=XmlFileProcessorTest
mvn test -Dtest=ExcelFileProcessorTest

# Streaming consumers
mvn test -Dtest=KafkaConsumerTest
mvn test -Dtest=RabbitMQConsumerTest
mvn test -Dtest=SQSConsumerTest
```

**Key Test Cases**:
- Connection establishment
- Data extraction
- Error handling
- Retry logic
- Credential management
- Large file handling

#### 2. Job Management Tests
```bash
mvn test -Dtest=JobSchedulerTest
mvn test -Dtest=JobExecutorTest
mvn test -Dtest=JobStateManagerTest
```

**Key Test Cases**:
- Job scheduling
- Job execution
- State transitions
- Concurrent job handling
- Job cancellation
- Job recovery

#### 3. Webhook Handler Tests
```bash
mvn test -Dtest=WebhookHandlerTest
mvn test -Dtest=SignatureVerificationTest
```

**Key Test Cases**:
- Webhook reception
- Signature verification
- Payload processing
- Error handling

### Run All Unit Tests
```bash
cd data-ingestion-service

# Run all tests
mvn test

# Run connector tests only
mvn test -Dtest=*ConnectorTest

# Run with coverage
mvn test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

### Expected Coverage
- **Connectors**: 85%+
- **Job Management**: 90%+
- **Webhook Handlers**: 85%+
- **Overall**: 80%+

---

## Unit 2: Data Processing Pipeline Tests

### Location
```
data-processing-pipeline/src/test/java/
```

### Test Categories

#### 1. Validation Tests
```bash
mvn test -Dtest=SchemaValidatorTest
mvn test -Dtest=DataTypeValidatorTest
mvn test -Dtest=BusinessRuleValidatorTest
```

**Key Test Cases**:
- Schema validation (valid/invalid)
- Data type checking
- Required field validation
- Format validation
- Business rule validation

#### 2. Cleansing Tests
```bash
mvn test -Dtest=DataCleanserTest
mvn test -Dtest=DeduplicationServiceTest
mvn test -Dtest=NormalizationServiceTest
```

**Key Test Cases**:
- Null handling
- Whitespace trimming
- Deduplication logic
- Data normalization
- Invalid data handling

#### 3. Transformation Tests
```bash
mvn test -Dtest=DataTransformerTest
mvn test -Dtest=MappingEngineTest
mvn test -Dtest=EnrichmentServiceTest
```

**Key Test Cases**:
- Field mapping
- Data type conversion
- Calculated fields
- Data enrichment
- Complex transformations

#### 4. Rule Engine Tests
```bash
mvn test -Dtest=RuleEngineTest
mvn test -Dtest=RuleEvaluatorTest
mvn test -Dtest=RuleLoaderTest
```

**Key Test Cases**:
- Rule loading
- Rule evaluation
- Condition matching
- Action execution
- Rule priority

#### 5. Quality Scoring Tests
```bash
mvn test -Dtest=QualityScorerTest
mvn test -Dtest=QualityMetricsTest
```

**Key Test Cases**:
- Completeness scoring
- Accuracy scoring
- Consistency scoring
- Overall quality calculation
- Threshold checking

### Run All Unit Tests
```bash
cd data-processing-pipeline

# Run all tests
mvn test

# Run validation tests only
mvn test -Dtest=*ValidatorTest

# Run with coverage
mvn test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

### Expected Coverage
- **Validation**: 90%+
- **Cleansing**: 85%+
- **Transformation**: 85%+
- **Rule Engine**: 90%+
- **Quality Scoring**: 85%+
- **Overall**: 80%+

---

## Unit 4: Analytics API Service Tests

### Location
```
analytics-api-service/src/test/java/
```

### Test Categories

#### 1. Query Engine Tests
```bash
mvn test -Dtest=QueryEngineTest
mvn test -Dtest=QueryBuilderTest
mvn test -Dtest=QueryOptimizerTest
```

**Key Test Cases**:
- Query parsing
- Query building
- Query optimization
- Multi-database queries
- Filter application
- Aggregation logic

#### 2. KPI Calculation Tests
```bash
mvn test -Dtest=KPICalculatorTest
mvn test -Dtest=RevenueMetricsTest
mvn test -Dtest=CustomerMetricsTest
mvn test -Dtest=OperationalMetricsTest
```

**Key Test Cases**:
- Revenue calculations
- Customer metrics
- Operational metrics
- Period comparisons
- Growth rate calculations

#### 3. Analytics Service Tests
```bash
mvn test -Dtest=RevenueAnalyticsServiceTest
mvn test -Dtest=CustomerAnalyticsServiceTest
mvn test -Dtest=TrendAnalysisServiceTest
```

**Key Test Cases**:
- Revenue analysis
- Customer segmentation
- Trend detection
- Forecasting logic
- Anomaly detection

#### 4. Report Generation Tests
```bash
mvn test -Dtest=ReportGeneratorTest
mvn test -Dtest=CsvExporterTest
mvn test -Dtest=ExcelExporterTest
mvn test -Dtest=PdfExporterTest
```

**Key Test Cases**:
- Report generation
- Format conversion
- Data export
- Template rendering

#### 5. Caching Tests
```bash
mvn test -Dtest=CacheServiceTest
mvn test -Dtest=CacheEvictionTest
```

**Key Test Cases**:
- Cache hit/miss
- Cache eviction
- Cache invalidation
- TTL handling

#### 6. WebSocket Tests
```bash
mvn test -Dtest=WebSocketHandlerTest
mvn test -Dtest=RealTimeUpdateServiceTest
```

**Key Test Cases**:
- Connection handling
- Message broadcasting
- Subscription management
- Error handling

### Run All Unit Tests
```bash
cd analytics-api-service

# Run all tests
mvn test

# Run query engine tests only
mvn test -Dtest=*QueryTest

# Run with coverage
mvn test jacoco:report

# View coverage report
open target/site/jacoco/index.html
```

### Expected Coverage
- **Query Engine**: 85%+
- **KPI Calculations**: 90%+
- **Analytics Services**: 85%+
- **Report Generation**: 80%+
- **Caching**: 85%+
- **Overall**: 80%+

---

## Unit 5: Dashboard Frontend Tests

### Location
```
dashboard-frontend/src/__tests__/
```

### Test Categories

#### 1. Component Tests
```bash
# Run all component tests
npm test -- --testPathPattern=components

# Specific components
npm test -- DashboardLayout.test.tsx
npm test -- RevenueChart.test.tsx
npm test -- CustomerTable.test.tsx
npm test -- KPICard.test.tsx
npm test -- FilterPanel.test.tsx
```

**Key Test Cases**:
- Component rendering
- Props handling
- User interactions
- Conditional rendering
- Error states
- Loading states

#### 2. Hook Tests
```bash
npm test -- --testPathPattern=hooks

# Specific hooks
npm test -- useAnalytics.test.ts
npm test -- useWebSocket.test.ts
npm test -- useFilters.test.ts
```

**Key Test Cases**:
- Hook initialization
- State updates
- Side effects
- Cleanup logic
- Error handling

#### 3. Redux Tests
```bash
npm test -- --testPathPattern=store

# Specific slices
npm test -- analyticsSlice.test.ts
npm test -- dashboardSlice.test.ts
npm test -- userSlice.test.ts
```

**Key Test Cases**:
- Action creators
- Reducers
- Selectors
- Async thunks
- State transformations

#### 4. Utility Tests
```bash
npm test -- --testPathPattern=utils

# Specific utilities
npm test -- formatters.test.ts
npm test -- validators.test.ts
npm test -- calculations.test.ts
```

**Key Test Cases**:
- Data formatting
- Validation logic
- Calculations
- Helper functions

#### 5. API Client Tests
```bash
npm test -- --testPathPattern=api

npm test -- analyticsApi.test.ts
npm test -- authApi.test.ts
```

**Key Test Cases**:
- API calls
- Request formatting
- Response handling
- Error handling
- Retry logic

### Run All Unit Tests
```bash
cd dashboard-frontend

# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run in watch mode
npm test -- --watch

# Run specific test file
npm test -- DashboardLayout.test.tsx

# Update snapshots
npm test -- -u
```

### Expected Coverage
- **Components**: 80%+
- **Hooks**: 85%+
- **Redux**: 90%+
- **Utilities**: 90%+
- **API Clients**: 85%+
- **Overall**: 80%+

---

## Running All Unit Tests

### Sequential Execution
```bash
#!/bin/bash
# run-all-unit-tests.sh

set -e

echo "Running all unit tests..."

# Backend services
services=("monitoring-audit-service" "storage-management-service" "data-ingestion-service" "data-processing-pipeline" "analytics-api-service")

for service in "${services[@]}"; do
  echo "Testing $service..."
  cd $service
  mvn test jacoco:report
  cd ..
  echo "$service tests complete!"
done

# Frontend
echo "Testing dashboard-frontend..."
cd dashboard-frontend
npm test -- --coverage --watchAll=false
cd ..
echo "dashboard-frontend tests complete!"

echo "All unit tests complete!"
```

### Parallel Execution
```bash
#!/bin/bash
# run-all-unit-tests-parallel.sh

set -e

echo "Running all unit tests in parallel..."

# Backend services
(cd monitoring-audit-service && mvn test jacoco:report) &
(cd storage-management-service && mvn test jacoco:report) &
(cd data-ingestion-service && mvn test jacoco:report) &
(cd data-processing-pipeline && mvn test jacoco:report) &
(cd analytics-api-service && mvn test jacoco:report) &

# Frontend
(cd dashboard-frontend && npm test -- --coverage --watchAll=false) &

# Wait for all tests to complete
wait

echo "All unit tests complete!"
```

---

## Coverage Reports

### Backend Coverage (JaCoCo)
```bash
# Generate coverage report
mvn test jacoco:report

# View report
open target/site/jacoco/index.html

# Coverage thresholds in pom.xml
<configuration>
  <rules>
    <rule>
      <element>BUNDLE</element>
      <limits>
        <limit>
          <counter>LINE</counter>
          <value>COVEREDRATIO</value>
          <minimum>0.80</minimum>
        </limit>
      </limits>
    </rule>
  </rules>
</configuration>
```

### Frontend Coverage (Jest)
```bash
# Generate coverage report
npm test -- --coverage --watchAll=false

# View report
open coverage/lcov-report/index.html

# Coverage thresholds in package.json
"jest": {
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

---

## Test Troubleshooting

### Common Issues

#### Tests Failing Due to Dependencies
```bash
# Update dependencies
mvn clean install -U

# Clear test cache
mvn clean test

# For frontend
rm -rf node_modules package-lock.json
npm install
```

#### Database Connection Issues
```bash
# Use Test Containers
@Testcontainers
@SpringBootTest
class ServiceTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
}

# Or use H2 in-memory database
spring.datasource.url=jdbc:h2:mem:testdb
```

#### Flaky Tests
```bash
# Run test multiple times
mvn test -Dtest=FlakyTest -Dsurefire.rerunFailingTestsCount=3

# For frontend
npm test -- --testNamePattern="flaky test" --maxWorkers=1
```

#### Slow Tests
```bash
# Run tests in parallel
mvn test -T 4

# For frontend
npm test -- --maxWorkers=4
```

---

## Test Quality Checklist

### Backend Tests
- [ ] All service methods tested
- [ ] All repository methods tested
- [ ] All controller endpoints tested
- [ ] Edge cases covered
- [ ] Error scenarios tested
- [ ] Mocks used appropriately
- [ ] Test data is realistic
- [ ] Tests are independent
- [ ] Tests are fast (< 1s each)
- [ ] Coverage meets 80% threshold

### Frontend Tests
- [ ] All components tested
- [ ] All hooks tested
- [ ] All Redux slices tested
- [ ] User interactions tested
- [ ] API calls mocked
- [ ] Error states tested
- [ ] Loading states tested
- [ ] Accessibility tested
- [ ] Tests are maintainable
- [ ] Coverage meets 80% threshold

---

## Next Steps

After successful unit tests:
1. Proceed to Integration Test Instructions
2. Fix any failing tests
3. Improve coverage for low-coverage areas
4. Review and refactor test code
5. Document any test-specific setup requirements

---

## Document Status

**Status**: Complete  
**Date**: 2026-03-04  
**Version**: 1.0.0  
**Next**: integration-test-instructions.md

---
