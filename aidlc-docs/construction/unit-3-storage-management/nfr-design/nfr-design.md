# NFR Design
## Unit 3: Storage Management Service

**Version**: 1.0  
**Date**: 2026-03-04

---

## 1. Performance Design

### 1.1 Throughput Optimization

**Batch Processing Design**
```java
@Service
public class BatchManager {
    private final Map<String, BatchBuffer> buffers = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(10);
    
    @Value("${storage.batch.size}")
    private int batchSize;
    
    @Value("${storage.batch.timeout}")
    private long batchTimeout;
    
    public void addToBatch(ProcessedDataRecord record, DatabaseTarget target) {
        String key = target.getType() + ":" + record.getEntityType();
        
        BatchBuffer buffer = buffers.computeIfAbsent(key, k -> {
            BatchBuffer newBuffer = new BatchBuffer(batchSize, batchTimeout);
            scheduleFlush(k, newBuffer);
            return newBuffer;
        });
        
        buffer.add(record);
        
        if (buffer.size() >= batchSize) {
            flushBatch(key);
        }
    }
    
    private void scheduleFlush(String key, BatchBuffer buffer) {
        scheduler.schedule(() -> flushBatch(key), batchTimeout, TimeUnit.MILLISECONDS);
    }
    
    private void flushBatch(String key) {
        BatchBuffer buffer = buffers.remove(key);
        if (buffer != null && !buffer.isEmpty()) {
            executeBatchWrite(buffer.getRecords(), key);
        }
    }
}
```

**Connection Pool Configuration**
```yaml
spring:
  datasource:
    postgresql:
      hikari:
        maximum-pool-size: 50
        minimum-idle: 10
        connection-timeout: 30000
        idle-timeout: 600000
        max-lifetime: 1800000
        leak-detection-threshold: 60000
```

**Async Processing**
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "lineageExecutor")
    public Executor lineageExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("lineage-");
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "secondaryWriteExecutor")
    public Executor secondaryWriteExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("secondary-write-");
        executor.initialize();
        return executor;
    }
}
```

### 1.2 Latency Optimization

**Caching Strategy**
```java
@Service
public class SchemaCache {
    
    @Cacheable(value = "schemas", key = "#entityType")
    public Schema getSchema(String entityType) {
        return schemaRepository.findByEntityType(entityType);
    }
    
    @CacheEvict(value = "schemas", key = "#schema.entityType")
    public void updateSchema(Schema schema) {
        schemaRepository.save(schema);
    }
}

@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

**Prepared Statements**
```java
@Repository
public class PostgreSQLWriter {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void batchInsert(List<Customer> customers) {
        String sql = "INSERT INTO customers (customer_id, business_id, email, first_name, last_name) " +
                    "VALUES (?, ?, ?, ?, ?) " +
                    "ON CONFLICT (customer_id) DO UPDATE SET " +
                    "email = EXCLUDED.email, first_name = EXCLUDED.first_name, last_name = EXCLUDED.last_name";
        
        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Customer customer = customers.get(i);
                ps.setObject(1, customer.getCustomerId());
                ps.setObject(2, customer.getBusinessId());
                ps.setString(3, customer.getEmail());
                ps.setString(4, customer.getFirstName());
                ps.setString(5, customer.getLastName());
            }
            
            @Override
            public int getBatchSize() {
                return customers.size();
            }
        });
    }
}
```

---

## 2. Scalability Design

### 2.1 Horizontal Scaling

**Stateless Service Design**
- No local state stored in service instances
- All state in databases or Redis
- Kafka consumer groups for load distribution

**Kafka Consumer Configuration**
```yaml
spring:
  kafka:
    consumer:
      group-id: storage-management-${INSTANCE_ID:default}
      enable-auto-commit: false
      max-poll-records: 500
      fetch-min-size: 1024
      fetch-max-wait: 500
    listener:
      concurrency: 3
      ack-mode: manual
```

**Load Balancing**
```yaml
# Kubernetes Service
apiVersion: v1
kind: Service
metadata:
  name: storage-management-service
spec:
  selector:
    app: storage-management
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
  sessionAffinity: None
```

### 2.2 Database Scaling

**Read Replicas**
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
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource());
        targetDataSources.put("replica", replicaDataSource());
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(primaryDataSource());
        return routingDataSource;
    }
}
```

**Partitioning Strategy**
```sql
-- PostgreSQL table partitioning by date
CREATE TABLE transactions (
    transaction_id UUID,
    business_id UUID,
    transaction_date TIMESTAMP,
    amount DECIMAL(15,2),
    ...
) PARTITION BY RANGE (transaction_date);

CREATE TABLE transactions_2026_01 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE transactions_2026_02 PARTITION OF transactions
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

---

## 3. Reliability Design

### 3.1 Circuit Breaker Pattern

**Resilience4j Configuration**
```java
@Configuration
public class Resilience4jConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(10)
            .minimumNumberOfCalls(5)
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
}

@Service
public class DatabaseWriterWithCircuitBreaker {
    
    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;
    
    public WriteResult write(ProcessedDataRecord record, DatabaseTarget target) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker(target.getType());
        
        return circuitBreaker.executeSupplier(() -> 
            databaseWriter.write(record, target)
        );
    }
}
```

### 3.2 Retry Mechanism

**Retry Configuration**
```java
@Configuration
public class RetryConfig {
    
    @Bean
    public RetryRegistry retryRegistry() {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .waitDuration(Duration.ofSeconds(1))
            .retryExceptions(TransientDataAccessException.class, TimeoutException.class)
            .ignoreExceptions(ConstraintViolationException.class)
            .build();
        
        return RetryRegistry.of(config);
    }
}

@Service
public class DatabaseWriterWithRetry {
    
    @Autowired
    private RetryRegistry retryRegistry;
    
    public WriteResult write(ProcessedDataRecord record, DatabaseTarget target) {
        Retry retry = retryRegistry.retry(target.getType());
        
        return retry.executeSupplier(() -> 
            databaseWriter.write(record, target)
        );
    }
}
```

### 3.3 Health Checks

**Spring Actuator Health Indicators**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            if (conn.isValid(5)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("database", "PostgreSQL")
                .withDetail("error", e.getMessage())
                .build();
        }
        return Health.down().build();
    }
}

@Component
public class KafkaHealthIndicator implements HealthIndicator {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Override
    public Health health() {
        try {
            kafkaTemplate.send("health-check", "ping").get(5, TimeUnit.SECONDS);
            return Health.up()
                .withDetail("kafka", "Connected")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("kafka", "Disconnected")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## 4. Security Design

### 4.1 Authentication & Authorization

**JWT Security Configuration**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/v1/storage/**").authenticated()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter jwtAuthenticationConverter = 
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);
        
        return jwtAuthenticationConverter();
    }
}
```

**Role-Based Access Control**
```java
@RestController
@RequestMapping("/api/v1/storage")
public class StorageController {
    
    @GetMapping("/stats")
    @PreAuthorize("hasAnyRole('ADMIN', 'DATA_ENGINEER')")
    public ResponseEntity<StorageStats> getStats() {
        return ResponseEntity.ok(storageService.getStats());
    }
    
    @PostMapping("/schemas")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Schema> createSchema(@RequestBody Schema schema) {
        return ResponseEntity.ok(schemaService.create(schema));
    }
}
```

### 4.2 Data Security

**Credential Encryption**
```java
@Service
public class CredentialEncryptionService {
    
    @Value("${encryption.key}")
    private String encryptionKey;
    
    public String encrypt(String plainText) {
        try {
            SecretKeySpec secretKey = new SecretKeySpec(
                encryptionKey.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE, secretKey);
            byte[] encrypted = cipher.doFinal(plainText.getBytes());
            return Base64.getEncoder().encodeToString(encrypted);
        } catch (Exception e) {
            throw new EncryptionException("Failed to encrypt", e);
        }
    }
    
    public String decrypt(String encryptedText) {
        try {
            SecretKeySpec secretKey = new SecretKeySpec(
                encryptionKey.getBytes(), "AES");
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE, secretKey);
            byte[] decrypted = cipher.doFinal(
                Base64.getDecoder().decode(encryptedText));
            return new String(decrypted);
        } catch (Exception e) {
            throw new EncryptionException("Failed to decrypt", e);
        }
    }
}
```

**Sensitive Data Masking in Logs**
```java
@Component
public class SensitiveDataMasker {
    
    private static final Pattern EMAIL_PATTERN = 
        Pattern.compile("([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\\.[a-zA-Z]{2,})");
    
    private static final Pattern PHONE_PATTERN = 
        Pattern.compile("\\d{3}-\\d{3}-\\d{4}");
    
    public String maskSensitiveData(String message) {
        String masked = message;
        
        // Mask email
        masked = EMAIL_PATTERN.matcher(masked)
            .replaceAll("***@$2");
        
        // Mask phone
        masked = PHONE_PATTERN.matcher(masked)
            .replaceAll("***-***-****");
        
        return masked;
    }
}
```

---

## 5. Maintainability Design

### 5.1 Monitoring & Metrics

**Prometheus Metrics**
```java
@Service
public class MetricsService {
    
    private final Counter writeCounter;
    private final Counter writeSuccessCounter;
    private final Counter writeFailureCounter;
    private final Timer writeTimer;
    private final Gauge batchQueueSize;
    
    public MetricsService(MeterRegistry registry) {
        this.writeCounter = Counter.builder("storage.writes.total")
            .description("Total write operations")
            .tag("service", "storage-management")
            .register(registry);
        
        this.writeSuccessCounter = Counter.builder("storage.writes.success")
            .description("Successful write operations")
            .register(registry);
        
        this.writeFailureCounter = Counter.builder("storage.writes.failed")
            .description("Failed write operations")
            .register(registry);
        
        this.writeTimer = Timer.builder("storage.writes.duration")
            .description("Write operation duration")
            .register(registry);
        
        this.batchQueueSize = Gauge.builder("storage.batch.queue.size", 
            this::getBatchQueueSize)
            .description("Current batch queue size")
            .register(registry);
    }
    
    public void recordWrite(DatabaseTarget target, boolean success, long duration) {
        writeCounter.increment();
        if (success) {
            writeSuccessCounter.increment();
        } else {
            writeFailureCounter.increment();
        }
        writeTimer.record(duration, TimeUnit.MILLISECONDS);
    }
}
```

**Structured Logging**
```java
@Slf4j
@Service
public class StorageService {
    
    public WriteResult write(ProcessedDataRecord record, DatabaseTarget target) {
        MDC.put("recordId", record.getRecordId().toString());
        MDC.put("entityType", record.getEntityType());
        MDC.put("targetDatabase", target.getType());
        
        try {
            log.info("Starting write operation");
            WriteResult result = executeWrite(record, target);
            log.info("Write operation completed successfully");
            return result;
        } catch (Exception e) {
            log.error("Write operation failed", e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

### 5.2 Configuration Management

**Externalized Configuration**
```yaml
# application.yml
spring:
  application:
    name: storage-management-service
  profiles:
    active: ${SPRING_PROFILE:dev}
  config:
    import: optional:configserver:http://config-server:8888

storage:
  routing:
    rules-refresh-interval: ${ROUTING_RULES_REFRESH:60000}
  batch:
    postgresql:
      size: ${BATCH_SIZE_POSTGRESQL:1000}
      timeout: ${BATCH_TIMEOUT_POSTGRESQL:5000}
  retry:
    max-attempts: ${RETRY_MAX_ATTEMPTS:3}
```

**Feature Flags**
```java
@Service
public class FeatureFlagService {
    
    @Value("${features.dual-write.enabled:false}")
    private boolean dualWriteEnabled;
    
    @Value("${features.lineage-tracking.enabled:true}")
    private boolean lineageTrackingEnabled;
    
    public boolean isDualWriteEnabled() {
        return dualWriteEnabled;
    }
    
    public boolean isLineageTrackingEnabled() {
        return lineageTrackingEnabled;
    }
}
```

---

## 6. Deployment Design

### 6.1 Containerization

**Dockerfile**
```dockerfile
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY target/storage-management-service-1.0.0.jar app.jar

RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", \
    "app.jar"]

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1
```

### 6.2 Kubernetes Deployment

**Deployment Manifest**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-management-service
  labels:
    app: storage-management
spec:
  replicas: 3
  selector:
    matchLabels:
      app: storage-management
  template:
    metadata:
      labels:
        app: storage-management
    spec:
      containers:
      - name: storage-management
        image: storage-management-service:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: POSTGRES_HOST
          valueFrom:
            configMapKeyRef:
              name: database-config
              key: postgres.host
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: database-secrets
              key: postgres.password
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
```

**Horizontal Pod Autoscaler**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: storage-management-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: storage-management-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Document Status

**Status**: Complete  
**Next**: Infrastructure Design

---
