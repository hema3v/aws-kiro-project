# Unit 4: Analytics API Service
## Complete Design Document

**Version**: 1.0  
**Date**: 2026-03-04  
**Status**: Complete

---

## 1. FUNCTIONAL DESIGN

### 1.1 Service Overview

The Analytics API Service provides query and reporting capabilities for the dashboard. It queries data from all storage databases, performs aggregations, generates reports, and provides real-time updates via WebSocket.

### 1.2 Core Capabilities

**Query Engine**
- Query across multiple databases (PostgreSQL, MongoDB, TimescaleDB, Snowflake)
- Complex aggregations and joins
- Custom query builder
- Query optimization and caching

**KPI Calculations**
- Revenue metrics (total, growth, trends)
- Customer metrics (count, acquisition, retention, lifetime value)
- Operational metrics (data processed, quality scores)
- Business-specific KPIs

**Revenue Analytics**
- Revenue by period (day, week, month, quarter, year)
- Revenue by business/product/channel
- Revenue trends and forecasting
- Average order value

**Customer Analytics**
- Customer segmentation
- Acquisition analysis
- Retention and churn rates
- Lifetime value calculation
- Customer journey analysis

**Trend Analysis**
- Historical trend analysis
- Predictive forecasting
- Anomaly detection
- Pattern recognition

**Report Generation**
- Pre-built report templates
- Custom report builder
- Export formats (CSV, Excel, PDF)
- Scheduled reports

**Real-Time Updates**
- WebSocket connections
- Live dashboard updates
- Alert notifications
- Data refresh triggers

### 1.3 Key Components

```
AnalyticsAPIService
├── query/
│   ├── QueryEngine
│   ├── QueryBuilder
│   ├── QueryOptimizer
│   └── QueryExecutor
├── analytics/
│   ├── KPIService
│   ├── RevenueAnalyticsService
│   ├── CustomerAnalyticsService
│   └── TrendAnalysisService
├── report/
│   ├── ReportGenerator
│   ├── ExportService
│   └── SchedulerService
├── cache/
│   ├── CacheManager
│   └── CacheInvalidator
├── websocket/
│   ├── WebSocketHandler
│   └── MessageBroadcaster
└── controller/
    ├── AnalyticsController
    ├── ReportController
    └── WebSocketController
```

### 1.4 Data Flow

```
Dashboard Request
    ↓
API Gateway
    ↓
Analytics API Service
    ↓
Check Cache (Redis)
    ├─ Hit → Return cached result
    └─ Miss → Query databases
        ↓
    Query Execution (PostgreSQL, MongoDB, TimescaleDB, Snowflake)
        ↓
    Aggregation & Calculation
        ↓
    Cache Result
        ↓
    Return to Dashboard
```

### 1.5 Business Rules

**BR-001: Caching Strategy**
- Cache KPIs for 5 minutes
- Cache reports for 15 minutes
- Cache query results for 10 minutes
- Invalidate on data updates

**BR-002: Query Optimization**
- Use read replicas for queries
- Limit result sets to 10,000 records
- Timeout queries after 30 seconds
- Use database-specific optimizations

**BR-003: Real-Time Updates**
- Push updates every 5 seconds
- Batch multiple updates
- Limit concurrent WebSocket connections to 1000

**BR-004: Report Generation**
- Async generation for large reports
- Queue reports during peak hours
- Limit export file size to 100MB

---

## 2. NFR REQUIREMENTS

### 2.1 Performance
- **NFR-P-001**: API response time < 200ms (p95)
- **NFR-P-002**: Query execution < 5 seconds
- **NFR-P-003**: Report generation < 30 seconds
- **NFR-P-004**: WebSocket latency < 100ms

### 2.2 Scalability
- **NFR-S-001**: Support 100+ concurrent users
- **NFR-S-002**: Handle 1000+ API requests/minute
- **NFR-S-003**: Support 500+ WebSocket connections

### 2.3 Reliability
- **NFR-R-001**: Service uptime 99.9%
- **NFR-R-002**: Query accuracy 100%
- **NFR-R-003**: Automatic failover for database connections

### 2.4 Caching
- **NFR-C-001**: Cache hit rate > 70%
- **NFR-C-002**: Cache invalidation < 1 second
- **NFR-C-003**: Redis availability 99.9%

---

## 3. NFR DESIGN

### 3.1 Caching Implementation

**Redis Configuration**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("kpis", config.entryTtl(Duration.ofMinutes(5)));
        cacheConfigurations.put("reports", config.entryTtl(Duration.ofMinutes(15)));
        cacheConfigurations.put("queries", config.entryTtl(Duration.ofMinutes(10)));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build();
    }
}
```

**Cacheable Methods**
```java
@Service
public class KPIService {
    
    @Cacheable(value = "kpis", key = "#businessId + ':' + #startDate + ':' + #endDate")
    public KPIResponse getKPIs(UUID businessId, LocalDate startDate, LocalDate endDate) {
        // Calculate KPIs from databases
        return calculateKPIs(businessId, startDate, endDate);
    }
    
    @CacheEvict(value = "kpis", allEntries = true)
    public void invalidateKPICache() {
        // Called when new data is stored
    }
}
```

### 3.2 Query Optimization

**Read Replica Configuration**
```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://primary:5432/analytics")
            .build();
    }
    
    @Bean
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://replica:5432/analytics")
            .build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource());
        targetDataSources.put("replica", replicaDataSource());
        
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(replicaDataSource());
        
        return routingDataSource;
    }
}
```

**Query Timeout**
```java
@Service
public class QueryExecutor {
    
    @Transactional(timeout = 30)
    @Async("queryExecutor")
    public CompletableFuture<QueryResult> executeQuery(Query query) {
        return CompletableFuture.supplyAsync(() -> {
            // Execute query with timeout
            return jdbcTemplate.query(query.getSql(), query.getParameters());
        });
    }
}
```

### 3.3 WebSocket Implementation

**WebSocket Configuration**
```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/app");
    }
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/dashboard")
            .setAllowedOrigins("*")
            .withSockJS();
    }
}
```

**Real-Time Updates**
```java
@Service
public class DashboardUpdateService {
    
    @Autowired
    private SimpMessagingTemplate messagingTemplate;
    
    @Scheduled(fixedDelay = 5000)
    public void broadcastUpdates() {
        // Get latest KPIs
        KPIResponse kpis = kpiService.getLatestKPIs();
        
        // Broadcast to all connected clients
        messagingTemplate.convertAndSend("/topic/kpis", kpis);
    }
    
    public void sendAlert(Alert alert) {
        messagingTemplate.convertAndSend("/topic/alerts", alert);
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

### 4.2 Database Read Replicas

**PostgreSQL Read Replica**
- Instance: db.r6g.large
- Purpose: Analytics queries
- Replication lag: < 1 second

**MongoDB Read Replica**
- Included in replica set
- Purpose: Document queries

### 4.3 Redis Cluster

**Configuration**
- Deployment: Cluster mode (3 shards, 1 replica each)
- Instance: cache.r6g.large
- Memory: 13GB per node
- Eviction: allkeys-lru

### 4.4 Infrastructure Cost

| Component | Quantity | Unit Cost | Total |
|-----------|----------|-----------|-------|
| Service Instances | 3-5 avg 4 | $200/month | $800 |
| PostgreSQL Read Replica | 1 | $300/month | $300 |
| Redis Cluster | 6 nodes | $150/month | $900 |
| Load Balancer | 1 | $25/month | $25 |
| **Total** | | | **$2,025/month** |

---

## 5. CODE GENERATION

### 5.1 Project Structure

```
analytics-api-service/
├── src/main/java/com/analytics/api/
│   ├── AnalyticsAPIApplication.java
│   ├── config/
│   │   ├── DatabaseConfig.java
│   │   ├── CacheConfig.java
│   │   ├── WebSocketConfig.java
│   │   └── SecurityConfig.java
│   ├── controller/
│   │   ├── AnalyticsController.java
│   │   ├── ReportController.java
│   │   └── WebSocketController.java
│   ├── service/
│   │   ├── KPIService.java
│   │   ├── RevenueAnalyticsService.java
│   │   ├── CustomerAnalyticsService.java
│   │   ├── TrendAnalysisService.java
│   │   ├── QueryService.java
│   │   └── ReportService.java
│   ├── query/
│   │   ├── QueryEngine.java
│   │   ├── QueryBuilder.java
│   │   └── QueryExecutor.java
│   ├── report/
│   │   ├── ReportGenerator.java
│   │   └── ExportService.java
│   ├── websocket/
│   │   ├── DashboardUpdateService.java
│   │   └── WebSocketHandler.java
│   ├── model/
│   │   ├── KPIResponse.java
│   │   ├── RevenueAnalytics.java
│   │   ├── CustomerAnalytics.java
│   │   └── QueryRequest.java
│   └── repository/
│       ├── AnalyticsRepository.java
│       └── ReportRepository.java
├── src/main/resources/
│   ├── application.yml
│   └── queries/
│       └── predefined-queries.sql
└── pom.xml
```

### 5.2 Key Implementation - KPI Service

```java
@Service
@RequiredArgsConstructor
public class KPIService {
    
    private final JdbcTemplate jdbcTemplate;
    private final MongoTemplate mongoTemplate;
    
    @Cacheable(value = "kpis", key = "#businessId + ':' + #period")
    public KPIResponse getKPIs(UUID businessId, DateRange period) {
        // Revenue metrics
        BigDecimal totalRevenue = calculateTotalRevenue(businessId, period);
        BigDecimal revenueGrowth = calculateRevenueGrowth(businessId, period);
        
        // Customer metrics
        long totalCustomers = countTotalCustomers(businessId, period);
        long newCustomers = countNewCustomers(businessId, period);
        double retentionRate = calculateRetentionRate(businessId, period);
        
        // Operational metrics
        long recordsProcessed = countRecordsProcessed(businessId, period);
        double dataQualityScore = calculateAverageQualityScore(businessId, period);
        
        return KPIResponse.builder()
            .totalRevenue(totalRevenue)
            .revenueGrowth(revenueGrowth)
            .totalCustomers(totalCustomers)
            .newCustomers(newCustomers)
            .retentionRate(retentionRate)
            .recordsProcessed(recordsProcessed)
            .dataQualityScore(dataQualityScore)
            .period(period)
            .build();
    }
    
    private BigDecimal calculateTotalRevenue(UUID businessId, DateRange period) {
        String sql = """
            SELECT COALESCE(SUM(amount), 0) as total_revenue
            FROM transactions
            WHERE business_id = ?
              AND transaction_date BETWEEN ? AND ?
              AND status = 'completed'
            """;
        
        return jdbcTemplate.queryForObject(sql, BigDecimal.class,
            businessId, period.getStart(), period.getEnd());
    }
}
```

### 5.3 Key Implementation - Revenue Analytics

```java
@Service
@RequiredArgsConstructor
public class RevenueAnalyticsService {
    
    private final JdbcTemplate jdbcTemplate;
    
    @Cacheable(value = "revenue", key = "#businessId + ':' + #period + ':' + #groupBy")
    public RevenueAnalytics getRevenueAnalytics(UUID businessId, 
                                                DateRange period, 
                                                GroupBy groupBy) {
        String sql = buildRevenueQuery(groupBy);
        
        List<RevenueDataPoint> dataPoints = jdbcTemplate.query(sql,
            (rs, rowNum) -> RevenueDataPoint.builder()
                .period(rs.getString("period"))
                .revenue(rs.getBigDecimal("revenue"))
                .orders(rs.getInt("orders"))
                .averageOrderValue(rs.getBigDecimal("avg_order_value"))
                .build(),
            businessId, period.getStart(), period.getEnd());
        
        return RevenueAnalytics.builder()
            .dataPoints(dataPoints)
            .totalRevenue(calculateTotal(dataPoints))
            .averageRevenue(calculateAverage(dataPoints))
            .trend(calculateTrend(dataPoints))
            .build();
    }
    
    private String buildRevenueQuery(GroupBy groupBy) {
        String dateFormat = switch (groupBy) {
            case DAY -> "YYYY-MM-DD";
            case WEEK -> "YYYY-\"W\"IW";
            case MONTH -> "YYYY-MM";
            case QUARTER -> "YYYY-\"Q\"Q";
            case YEAR -> "YYYY";
        };
        
        return String.format("""
            SELECT 
                TO_CHAR(transaction_date, '%s') as period,
                SUM(amount) as revenue,
                COUNT(*) as orders,
                AVG(amount) as avg_order_value
            FROM transactions
            WHERE business_id = ?
              AND transaction_date BETWEEN ? AND ?
              AND status = 'completed'
            GROUP BY period
            ORDER BY period
            """, dateFormat);
    }
}
```

### 5.4 Key Implementation - Query Engine

```java
@Service
@RequiredArgsConstructor
public class QueryEngine {
    
    private final Map<String, DataSource> dataSources;
    
    @Async("queryExecutor")
    @Transactional(timeout = 30)
    public CompletableFuture<QueryResult> executeQuery(QueryRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            DataSource dataSource = dataSources.get(request.getDatabase());
            
            try (Connection conn = dataSource.getConnection();
                 PreparedStatement stmt = conn.prepareStatement(request.getSql())) {
                
                // Set parameters
                setParameters(stmt, request.getParameters());
                
                // Execute query
                try (ResultSet rs = stmt.executeQuery()) {
                    List<Map<String, Object>> rows = new ArrayList<>();
                    ResultSetMetaData metaData = rs.getMetaData();
                    
                    while (rs.next()) {
                        Map<String, Object> row = new HashMap<>();
                        for (int i = 1; i <= metaData.getColumnCount(); i++) {
                            row.put(metaData.getColumnName(i), rs.getObject(i));
                        }
                        rows.add(row);
                    }
                    
                    return QueryResult.builder()
                        .rows(rows)
                        .rowCount(rows.size())
                        .executionTime(System.currentTimeMillis())
                        .build();
                }
            } catch (SQLException e) {
                throw new QueryExecutionException("Query failed", e);
            }
        });
    }
}
```

### 5.5 Key Implementation - Report Generator

```java
@Service
@RequiredArgsConstructor
public class ReportGenerator {
    
    private final KPIService kpiService;
    private final RevenueAnalyticsService revenueService;
    
    @Async("reportExecutor")
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            Report report = new Report();
            report.setTitle(request.getTitle());
            report.setGeneratedAt(Instant.now());
            
            // Add KPIs
            if (request.includeKPIs()) {
                KPIResponse kpis = kpiService.getKPIs(
                    request.getBusinessId(), 
                    request.getPeriod()
                );
                report.addSection("KPIs", kpis);
            }
            
            // Add revenue analytics
            if (request.includeRevenue()) {
                RevenueAnalytics revenue = revenueService.getRevenueAnalytics(
                    request.getBusinessId(),
                    request.getPeriod(),
                    GroupBy.MONTH
                );
                report.addSection("Revenue", revenue);
            }
            
            return report;
        });
    }
    
    public byte[] exportToPDF(Report report) {
        // Use JasperReports or similar
        return pdfGenerator.generate(report);
    }
    
    public byte[] exportToExcel(Report report) {
        // Use Apache POI
        return excelGenerator.generate(report);
    }
}
```

### 5.6 API Endpoints

**KPIs**
```
GET /api/v1/analytics/kpis?businessId={id}&startDate={date}&endDate={date}
```

**Revenue Analytics**
```
GET /api/v1/analytics/revenue?businessId={id}&startDate={date}&endDate={date}&groupBy={period}
```

**Customer Analytics**
```
GET /api/v1/analytics/customers?businessId={id}&startDate={date}&endDate={date}
```

**Trends**
```
GET /api/v1/analytics/trends?metric={metric}&startDate={date}&endDate={date}&forecast={boolean}
```

**Custom Query**
```
POST /api/v1/analytics/query
Body: { "database": "postgresql", "query": {...} }
```

**Report Export**
```
GET /api/v1/analytics/reports/{reportId}/export?format={csv|excel|pdf}
```

**WebSocket**
```
ws://host/ws/dashboard
Subscribe: /topic/kpis, /topic/alerts
```

### 5.7 Configuration

```yaml
spring:
  application:
    name: analytics-api-service
  
  datasource:
    primary:
      url: jdbc:postgresql://${POSTGRES_PRIMARY:localhost}:5432/analytics
      username: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
    replica:
      url: jdbc:postgresql://${POSTGRES_REPLICA:localhost}:5432/analytics
      username: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
  
  data:
    mongodb:
      uri: mongodb://${MONGO_HOST:localhost}:27017/analytics
    redis:
      cluster:
        nodes:
          - ${REDIS_NODE1:localhost:6379}
          - ${REDIS_NODE2:localhost:6380}
          - ${REDIS_NODE3:localhost:6381}
  
  cache:
    type: redis
    redis:
      time-to-live: 600000

analytics:
  query:
    timeout: 30000
    max-results: 10000
  cache:
    kpis-ttl: 300000
    reports-ttl: 900000
  websocket:
    max-connections: 1000
    update-interval: 5000

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
```

### 5.8 Maven Dependencies

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
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- WebSocket -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    
    <!-- Cache -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
    
    <!-- Report Generation -->
    <dependency>
        <groupId>net.sf.jasperreports</groupId>
        <artifactId>jasperreports</artifactId>
        <version>6.20.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.poi</groupId>
        <artifactId>poi-ooxml</artifactId>
        <version>5.2.3</version>
    </dependency>
    
    <!-- GraphQL (Optional) -->
    <dependency>
        <groupId>com.graphql-java-kickstart</groupId>
        <artifactId>graphql-spring-boot-starter</artifactId>
        <version>15.0.0</version>
    </dependency>
</dependencies>
```

---

## 6. DEPLOYMENT

### 6.1 Docker Configuration

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/analytics-api-service-1.0.0.jar app.jar
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
  name: analytics-api-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: analytics-api
  template:
    metadata:
      labels:
        app: analytics-api
    spec:
      containers:
      - name: analytics-api
        image: analytics-api-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
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

- [ ] API response time < 200ms (p95)
- [ ] Query execution < 5 seconds
- [ ] Support 100+ concurrent users
- [ ] Cache hit rate > 70%
- [ ] WebSocket latency < 100ms
- [ ] Report generation < 30 seconds
- [ ] Query accuracy 100%
- [ ] Complete API documentation (OpenAPI)
- [ ] 80%+ test coverage
- [ ] Real-time updates functional

---

## Document Status

**Status**: Complete  
**All Stages**: Functional Design, NFR Requirements, NFR Design, Infrastructure Design, Code Generation  
**Next**: Unit 5 (Dashboard Frontend) - FINAL UNIT!

---
