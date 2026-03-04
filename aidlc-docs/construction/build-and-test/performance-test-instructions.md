# Performance Test Instructions
## Data Integration and Analytics Platform

**Date**: 2026-03-04  
**Status**: Ready for Testing  
**Test Tools**: JMeter, Gatling, K6, Artillery

---

## Overview

Performance tests verify that the system meets performance requirements under various load conditions. These tests validate throughput, latency, scalability, and resource utilization.

**Performance Targets**:
- Data Ingestion: 1M records/hour
- Data Processing: 1M records/hour, < 50ms validation
- Storage: 10K records/sec, < 500ms latency (p95)
- Analytics API: < 200ms response (p95), < 5s query time
- Dashboard: < 3s load time, < 1s update latency

---

## Test Strategy

### Performance Test Types
1. **Load Testing**: Verify system under expected load
2. **Stress Testing**: Find breaking points (2x normal load)
3. **Spike Testing**: Handle sudden traffic spikes
4. **Endurance Testing**: Sustained load over time (memory leaks, degradation)
5. **Scalability Testing**: Verify horizontal scaling

### Test Environment
- **Infrastructure**: Production-like environment
- **Data Volume**: Realistic data sets
- **Monitoring**: Real-time metrics collection
- **Baseline**: Establish performance baseline

---

## Performance Test 1: Data Ingestion Throughput

### Test Objective
Verify ingestion service can handle 1M records/hour (278 records/second).

### Test Configuration

#### JMeter Test Plan
```xml
<!-- ingestion-load-test.jmx -->
<jmeterTestPlan>
  <hashTree>
    <TestPlan>
      <stringProp name="TestPlan.comments">Data Ingestion Load Test</stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup>
        <stringProp name="ThreadGroup.num_threads">50</stringProp>
        <stringProp name="ThreadGroup.ramp_time">60</stringProp>
        <stringProp name="ThreadGroup.duration">3600</stringProp>
        <stringProp name="ThreadGroup.delay">0</stringProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy>
          <stringProp name="HTTPSampler.domain">localhost</stringProp>
          <stringProp name="HTTPSampler.port">8080</stringProp>
          <stringProp name="HTTPSampler.path">/api/ingestion/ingest</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
          <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

#### Gatling Test Script
```scala
// IngestionLoadTest.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class IngestionLoadTest extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
  
  val ingestionScenario = scenario("Data Ingestion Load Test")
    .exec(
      http("Ingest Data")
        .post("/api/ingestion/ingest")
        .body(StringBody("""
          {
            "sourceType": "API",
            "sourceId": "load-test-${randomUUID()}",
            "data": {
              "transactions": [
                {"id": "${randomInt()}", "amount": ${randomAmount()}, "customer": "Customer ${randomInt()}"}
              ]
            }
          }
        """)).asJson
        .check(status.is(200))
        .check(jsonPath("$.status").is("SUCCESS"))
    )
  
  setUp(
    ingestionScenario.inject(
      rampUsersPerSec(10) to 278 during (5 minutes),
      constantUsersPerSec(278) during (55 minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lt(1000),
     global.successfulRequests.percent.gt(99)
   )
}
```

### Run Test
```bash
# JMeter
jmeter -n -t ingestion-load-test.jmx -l results.jtl -e -o report/

# Gatling
mvn gatling:test -Dgatling.simulationClass=IngestionLoadTest

# K6
k6 run --vus 50 --duration 1h ingestion-load-test.js
```

### Success Criteria
- Throughput: ≥ 278 requests/second (1M/hour)
- Response Time (p95): < 1000ms
- Error Rate: < 1%
- CPU Usage: < 80%
- Memory Usage: Stable (no leaks)

---

## Performance Test 2: Data Processing Throughput

### Test Objective
Verify processing pipeline can handle 1M records/hour with < 50ms validation time.

### Test Configuration

#### K6 Test Script
```javascript
// processing-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '5m', target: 278 }, // Ramp up to 278 rps
    { duration: '55m', target: 278 }, // Stay at 278 rps
  ],
  thresholds: {
    http_req_duration: ['p(95)<50'], // 95% of requests < 50ms
    errors: ['rate<0.01'], // Error rate < 1%
  },
};

export default function () {
  const payload = JSON.stringify({
    recordId: `test-${Date.now()}-${Math.random()}`,
    data: {
      field1: 'value1',
      field2: 'value2',
      field3: 123,
      field4: true,
    },
  });
  
  const params = {
    headers: {
      'Content-Type': 'application/json',
    },
  };
  
  const response = http.post('http://localhost:8081/api/processing/validate', payload, params);
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'validation time < 50ms': (r) => r.timings.duration < 50,
    'quality score >= 85': (r) => JSON.parse(r.body).qualityScore >= 85,
  }) || errorRate.add(1);
}
```

### Run Test
```bash
k6 run processing-load-test.js --out influxdb=http://localhost:8086/k6
```

### Success Criteria
- Throughput: ≥ 278 records/second
- Validation Time (p95): < 50ms
- Quality Score: ≥ 85% for valid data
- Error Rate: < 1%
- Kafka Lag: < 1000 messages

---

## Performance Test 3: Storage Write Performance

### Test Objective
Verify storage service can write 10,000 records/second with < 500ms latency (p95).

### Test Configuration

#### Gatling Test Script
```scala
// StorageLoadTest.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class StorageLoadTest extends Simulation {
  
  val httpProtocol = http
    .baseUrl("http://localhost:8082")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
  
  val storageScenario = scenario("Storage Write Load Test")
    .exec(
      http("Store Data")
        .post("/api/storage/store")
        .body(StringBody("""
          {
            "entityType": "TRANSACTION",
            "data": {
              "id": "${randomUUID()}",
              "amount": ${randomAmount()},
              "customer": "Customer ${randomInt()}",
              "timestamp": "${now()}"
            }
          }
        """)).asJson
        .check(status.is(200))
        .check(jsonPath("$.status").is("SUCCESS"))
    )
  
  setUp(
    storageScenario.inject(
      rampUsersPerSec(1000) to 10000 during (2 minutes),
      constantUsersPerSec(10000) during (10 minutes)
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile3.lt(500),
     global.successfulRequests.percent.gt(99.9)
   )
}
```

### Run Test
```bash
mvn gatling:test -Dgatling.simulationClass=StorageLoadTest
```

### Success Criteria
- Throughput: ≥ 10,000 records/second
- Write Latency (p95): < 500ms
- Error Rate: < 0.1%
- Database Connections: Stable pool usage
- Disk I/O: < 80% utilization

---

## Performance Test 4: Analytics API Response Time

### Test Objective
Verify analytics API responds in < 200ms (p95) with caching.

### Test Configuration

#### Artillery Test Script
```yaml
# analytics-api-load-test.yml
config:
  target: "http://localhost:8083"
  phases:
    - duration: 300
      arrivalRate: 50
      name: "Warm up"
    - duration: 600
      arrivalRate: 100
      name: "Sustained load"
    - duration: 300
      arrivalRate: 200
      name: "Spike"
  processor: "./analytics-processor.js"

scenarios:
  - name: "Revenue Analytics"
    weight: 40
    flow:
      - get:
          url: "/api/analytics/revenue?period=MONTHLY&year=2026"
          expect:
            - statusCode: 200
            - contentType: json
            - hasProperty: totalRevenue
  
  - name: "Customer Analytics"
    weight: 30
    flow:
      - get:
          url: "/api/analytics/customers?segment=PREMIUM"
          expect:
            - statusCode: 200
            - contentType: json
  
  - name: "Trend Analysis"
    weight: 20
    flow:
      - get:
          url: "/api/analytics/trends?metric=REVENUE&period=DAILY"
          expect:
            - statusCode: 200
            - contentType: json
  
  - name: "Report Generation"
    weight: 10
    flow:
      - post:
          url: "/api/analytics/reports/generate"
          json:
            reportType: "REVENUE_SUMMARY"
            format: "PDF"
            period: "MONTHLY"
          expect:
            - statusCode: 200
```

### Run Test
```bash
artillery run analytics-api-load-test.yml --output report.json
artillery report report.json
```

### Success Criteria
- Response Time (p95): < 200ms
- Query Time: < 5 seconds
- Cache Hit Rate: > 70%
- Concurrent Users: 100+
- Error Rate: < 0.5%

---

## Performance Test 5: Dashboard Load Time

### Test Objective
Verify dashboard loads in < 3 seconds and updates in < 1 second.

### Test Configuration

#### Lighthouse CI
```javascript
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000/dashboard'],
      numberOfRuns: 5,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.9 }],
        'first-contentful-paint': ['error', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 3000 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['error', { maxNumericValue: 300 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',
    },
  },
};
```

#### WebPageTest Script
```javascript
// dashboard-performance-test.js
const WebPageTest = require('webpagetest');
const wpt = new WebPageTest('www.webpagetest.org');

wpt.runTest('http://localhost:3000/dashboard', {
  location: 'Dulles:Chrome',
  runs: 5,
  firstViewOnly: false,
  video: true,
  connectivity: '4G',
}, (err, result) => {
  if (err) return console.error(err);
  
  console.log('Test Results:');
  console.log('Load Time:', result.data.median.firstView.loadTime);
  console.log('First Byte:', result.data.median.firstView.TTFB);
  console.log('Start Render:', result.data.median.firstView.render);
  console.log('Speed Index:', result.data.median.firstView.SpeedIndex);
});
```

### Run Test
```bash
# Lighthouse CI
lhci autorun

# WebPageTest
node dashboard-performance-test.js

# Custom load test
npm run test:performance
```

### Success Criteria
- Load Time: < 3 seconds
- First Contentful Paint: < 2 seconds
- Largest Contentful Paint: < 3 seconds
- Time to Interactive: < 3.5 seconds
- Cumulative Layout Shift: < 0.1
- Real-time Update Latency: < 1 second

---

## Performance Test 6: End-to-End Latency

### Test Objective
Verify end-to-end latency from ingestion to dashboard display is < 10 seconds (p95).

### Test Configuration

```javascript
// e2e-latency-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import ws from 'k6/ws';

export const options = {
  vus: 10,
  duration: '30m',
};

export default function () {
  const startTime = Date.now();
  const recordId = `e2e-test-${startTime}-${Math.random()}`;
  
  // 1. Ingest data
  const ingestPayload = JSON.stringify({
    sourceType: 'API',
    sourceId: 'e2e-test',
    data: {
      recordId: recordId,
      transactions: [
        { id: recordId, amount: 1000, customer: 'Test Customer' }
      ]
    },
  });
  
  const ingestResponse = http.post(
    'http://localhost:8080/api/ingestion/ingest',
    ingestPayload,
    { headers: { 'Content-Type': 'application/json' } }
  );
  
  check(ingestResponse, {
    'ingestion successful': (r) => r.status === 200,
  });
  
  // 2. Connect to WebSocket and wait for update
  const url = 'ws://localhost:8083/ws/updates';
  const response = ws.connect(url, {}, function (socket) {
    socket.on('open', () => {
      socket.send(JSON.stringify({ subscribe: 'revenue-updates' }));
    });
    
    socket.on('message', (data) => {
      const message = JSON.parse(data);
      if (message.recordId === recordId) {
        const endTime = Date.now();
        const latency = endTime - startTime;
        
        check(latency, {
          'e2e latency < 10s': (l) => l < 10000,
        });
        
        socket.close();
      }
    });
    
    socket.setTimeout(() => {
      console.error('Timeout waiting for update');
      socket.close();
    }, 15000);
  });
  
  sleep(1);
}
```

### Run Test
```bash
k6 run e2e-latency-test.js
```

### Success Criteria
- End-to-End Latency (p95): < 10 seconds
- Ingestion Time: < 1 second
- Processing Time: < 3 seconds
- Storage Time: < 2 seconds
- Query Time: < 2 seconds
- WebSocket Delivery: < 1 second

---

## Stress Testing

### Test Objective
Find system breaking points by testing at 2x normal load.

### Test Configuration

```scala
// StressTest.scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class StressTest extends Simulation {
  
  val httpProtocol = http.baseUrl("http://localhost:8080")
  
  val stressScenario = scenario("Stress Test")
    .exec(http("Ingest").post("/api/ingestion/ingest").body(StringBody("...")).asJson)
  
  setUp(
    stressScenario.inject(
      rampUsersPerSec(100) to 556 during (5 minutes), // 2x normal load
      constantUsersPerSec(556) during (30 minutes),
      rampUsersPerSec(556) to 1000 during (5 minutes), // Push to breaking point
    )
  ).protocols(httpProtocol)
}
```

### Run Test
```bash
mvn gatling:test -Dgatling.simulationClass=StressTest
```

### Success Criteria
- System handles 2x load gracefully
- Degradation is gradual, not sudden
- Error rate increases predictably
- System recovers after load reduction
- No data loss or corruption

---

## Endurance Testing

### Test Objective
Verify system stability under sustained load for extended period (24 hours).

### Test Configuration

```javascript
// endurance-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1h', target: 200 }, // Ramp up
    { duration: '22h', target: 200 }, // Sustained load
    { duration: '1h', target: 0 }, // Ramp down
  ],
};

export default function () {
  const response = http.post('http://localhost:8080/api/ingestion/ingest', '...');
  check(response, { 'status is 200': (r) => r.status === 200 });
  sleep(1);
}
```

### Run Test
```bash
k6 run endurance-test.js --out influxdb=http://localhost:8086/k6
```

### Success Criteria
- No memory leaks (stable memory usage)
- No performance degradation over time
- No connection pool exhaustion
- No disk space issues
- Error rate remains stable
- Response times remain consistent

---

## Scalability Testing

### Test Objective
Verify horizontal scaling improves performance linearly.

### Test Configuration

```bash
#!/bin/bash
# scalability-test.sh

# Test with different instance counts
for instances in 1 2 4 8; do
  echo "Testing with $instances instances..."
  
  # Scale service
  kubectl scale deployment data-ingestion-service --replicas=$instances
  
  # Wait for pods to be ready
  kubectl wait --for=condition=ready pod -l app=data-ingestion-service --timeout=300s
  
  # Run load test
  k6 run --vus 100 --duration 10m ingestion-load-test.js --out json=results-$instances.json
  
  # Collect metrics
  echo "Instances: $instances" >> scalability-results.txt
  jq '.metrics.http_req_duration.values.p95' results-$instances.json >> scalability-results.txt
  jq '.metrics.http_reqs.values.rate' results-$instances.json >> scalability-results.txt
done
```

### Run Test
```bash
./scalability-test.sh
```

### Success Criteria
- Throughput increases linearly with instances
- Response time remains stable or improves
- No bottlenecks in shared resources
- Load balancing is effective
- Auto-scaling triggers correctly

---

## Performance Monitoring

### Real-Time Monitoring During Tests

```bash
# Prometheus queries
# CPU usage
rate(process_cpu_seconds_total[5m])

# Memory usage
process_resident_memory_bytes

# Request rate
rate(http_requests_total[5m])

# Response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Error rate
rate(http_requests_total{status=~"5.."}[5m])
```

### Grafana Dashboards
- System Overview (CPU, Memory, Disk, Network)
- Application Metrics (Request Rate, Response Time, Error Rate)
- Database Performance (Connections, Query Time, Throughput)
- Kafka Metrics (Lag, Throughput, Consumer Group Status)
- JVM Metrics (Heap, GC, Threads)

---

## Performance Test Checklist

- [ ] Test environment matches production
- [ ] Baseline performance established
- [ ] Load tests pass all criteria
- [ ] Stress tests identify breaking points
- [ ] Endurance tests show stability
- [ ] Scalability tests verify horizontal scaling
- [ ] Monitoring dashboards configured
- [ ] Performance bottlenecks identified
- [ ] Optimization recommendations documented
- [ ] Test reports generated

---

## Performance Optimization Recommendations

### If Ingestion is Slow
- Increase parallel connectors
- Optimize database queries
- Add connection pooling
- Implement batch processing
- Scale horizontally

### If Processing is Slow
- Optimize validation logic
- Increase Kafka partitions
- Add more consumer threads
- Implement caching for rules
- Optimize transformation logic

### If Storage is Slow
- Increase batch sizes
- Optimize database indexes
- Add write-behind caching
- Use connection pooling
- Consider sharding

### If API is Slow
- Implement aggressive caching
- Optimize database queries
- Add read replicas
- Implement query result pagination
- Use materialized views

### If Dashboard is Slow
- Optimize bundle size
- Implement code splitting
- Add CDN for static assets
- Optimize images and assets
- Implement lazy loading

---

## Next Steps

After successful performance tests:
1. Analyze results and identify bottlenecks
2. Implement optimizations
3. Re-run tests to verify improvements
4. Document performance characteristics
5. Establish performance monitoring in production

---

## Document Status

**Status**: Complete  
**Date**: 2026-03-04  
**Version**: 1.0.0  
**Next**: build-and-test-summary.md

---
