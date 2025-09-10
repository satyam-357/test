# 7.2 Performance Tuning Guide

*This section documents performance tuning procedures for the Email Agent microservice.*

## Performance Optimization

### 7.2.1 Database Performance Tuning

#### Connection Pool Optimization
**Configuration:**
```properties
# HikariCP connection pool settings
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
```

**Optimization Tips:**
- Set `maximum-pool-size` based on concurrent users (recommended: 2x CPU cores)
- Use `minimum-idle` to maintain warm connections
- Monitor connection leaks with `leak-detection-threshold`
- Adjust timeouts based on network latency

#### Query Optimization
**Indexing Strategy:**
```sql
-- User preferences lookup
CREATE INDEX idx_email_preferences_user_id ON email_preferences(user_id);

-- User lookup by email
CREATE INDEX idx_email_user_email ON email_user(email);

-- Composite index for common queries
CREATE INDEX idx_email_user_email_tenant ON email_user(email, tenant_id);
```

**JPA Optimization:**
```java
// Use @Query with specific fields instead of loading entire entities
@Query("SELECT u.timeZone FROM EmailUser u WHERE u.email = :email")
String findTimeZoneByEmail(@Param("email") String email);

// Use batch operations for bulk updates
@Modifying
@Query("UPDATE EmailUser u SET u.lastModifiedDate = :date WHERE u.email IN :emails")
int updateLastModifiedBatch(@Param("emails") List<String> emails, @Param("date") Date date);
```

### 7.2.2 Token Management Performance

#### Token Caching Optimization
**Configuration:**
```properties
# Token refresh interval (configurable)
token.refresh.interval=3300000

# Token cache size (per tenant)
token.cache.max-size=100
token.cache.expire-after-write=3600000
```

**Implementation:**
```java
// Use Caffeine cache for better performance
@Bean
public Cache<String, TokenData> tokenCache() {
    return Caffeine.newBuilder()
        .maximumSize(100)
        .expireAfterWrite(Duration.ofHours(1))
        .build();
}
```

#### Proactive Token Refresh
```java
// Refresh tokens before expiry
if (Instant.now().isAfter(tokenData.expiryTime.minus(Duration.ofMinutes(5)))) {
    refreshToken(clientKey);
}
```

### 7.2.3 API Call Optimization

#### Batch Operations
**Microsoft Graph Batch API:**
```java
// Use batch requests for multiple operations
List<BatchRequestPart> batchParts = new ArrayList<>();
for (String email : emails) {
    batchParts.add(createBatchRequestPart(email));
}
String batchResponse = graphApiClient.executeBatch(batchParts);
```

#### Connection Pooling for HTTP Clients
```java
@Bean
public CloseableHttpClient httpClient() {
    return HttpClients.custom()
        .setMaxConnTotal(100)
        .setMaxConnPerRoute(20)
        .setConnectionTimeToLive(30, TimeUnit.SECONDS)
        .build();
}
```

#### Retry Configuration
```java
@Bean
public RetryTemplate retryTemplate() {
    return RetryTemplate.builder()
        .maxAttempts(3)
        .exponentialBackoff(1000, 2, 10000)
        .retryOn(ConnectException.class)
        .retryOn(SocketTimeoutException.class)
        .build();
}
```

### 7.2.4 Memory Management

#### JVM Tuning
**JVM Parameters:**
```bash
# Heap size (adjust based on available memory)
-Xms2g -Xmx4g

# Garbage collection
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+UseStringDeduplication

# Memory monitoring
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/emailagent/
```

#### Object Pooling
```java
// Reuse objects to reduce GC pressure
private final ObjectPool<SimpleDateFormat> dateFormatPool = 
    new GenericObjectPool<>(new SimpleDateFormatFactory());

public String formatDate(Date date) {
    SimpleDateFormat formatter = null;
    try {
        formatter = dateFormatPool.borrowObject();
        return formatter.format(date);
    } finally {
        if (formatter != null) {
            dateFormatPool.returnObject(formatter);
        }
    }
}
```

### 7.2.5 Caching Strategy

#### Multi-Level Caching
```java
// L1: In-memory cache (fastest)
@Cacheable(value = "userPreferences", key = "#email")
public EmailPreferences getUserPreferences(String email) {
    return emailPreferencesDao.findByEmail(email);
}

// L2: Redis cache (distributed)
@Cacheable(value = "availabilityCache", key = "#emails + #startTime + #endTime")
public AvailabilityResponse getAvailability(List<String> emails, String startTime, String endTime) {
    return eventService.getAvailability(emails, startTime, endTime, 30, true);
}
```

#### Cache Configuration
```properties
# Redis configuration
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.timeout=2000
spring.redis.jedis.pool.max-active=20
spring.redis.jedis.pool.max-idle=10
spring.redis.jedis.pool.min-idle=5
```

### 7.2.6 Asynchronous Processing

#### Async Event Processing
```java
@Async("taskExecutor")
@EventListener
public void handleEventCreated(EventCreatedEvent event) {
    // Process event asynchronously
    eventService.processEventAsync(event.getEventId());
}

@Bean("taskExecutor")
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    executor.setThreadNamePrefix("EmailAgent-");
    executor.initialize();
    return executor;
}
```

#### Non-Blocking API Calls
```java
// Use WebClient for non-blocking HTTP calls
@Bean
public WebClient graphWebClient() {
    return WebClient.builder()
        .baseUrl("https://graph.microsoft.com/v1.0")
        .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer {token}")
        .build();
}
```

## Configuration Tuning

### 7.2.7 Application Properties Optimization

#### Database Configuration
```properties
# JPA/Hibernate optimization
spring.jpa.properties.hibernate.jdbc.batch_size=25
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.jdbc.batch_versioned_data=true
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=false
```

#### Logging Configuration
```properties
# Reduce logging overhead in production
logging.level.org.springframework=WARN
logging.level.org.hibernate=WARN
logging.level.com.enttribe=INFO
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} - %msg%n
```

#### Thread Pool Configuration
```properties
# Tomcat thread pool
server.tomcat.threads.max=200
server.tomcat.threads.min-spare=10
server.tomcat.accept-count=100
server.tomcat.max-connections=8192
```

### 7.2.8 External Service Configuration

#### Microsoft Graph Rate Limiting
```java
// Implement rate limiting
@Component
public class GraphRateLimiter {
    private final RateLimiter rateLimiter = RateLimiter.create(10.0); // 10 requests per second
    
    public void acquire() {
        rateLimiter.acquire();
    }
}
```

#### Timeout Configuration
```properties
# HTTP client timeouts
http.client.connection-timeout=5000
http.client.read-timeout=30000
http.client.write-timeout=30000
```

## Monitoring and Metrics

### 7.2.9 Performance Monitoring

#### Application Metrics
```java
@Component
public class PerformanceMetrics {
    private final MeterRegistry meterRegistry;
    private final Timer apiCallTimer;
    private final Counter errorCounter;
    
    public PerformanceMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.apiCallTimer = Timer.builder("api.call.duration")
            .description("API call duration")
            .register(meterRegistry);
        this.errorCounter = Counter.builder("api.errors")
            .description("API errors")
            .register(meterRegistry);
    }
}
```

#### Health Checks
```java
@Component
public class EmailAgentHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // Check database connectivity
        // Check Microsoft Graph connectivity
        // Check EWS connectivity
        return Health.up()
            .withDetail("database", "UP")
            .withDetail("graph", "UP")
            .withDetail("ews", "UP")
            .build();
    }
}
```

### 7.2.10 Load Testing

#### Performance Benchmarks
- **Concurrent Users**: Multiple concurrent users supported
- **API Throughput**: Expected requests per second based on configuration
- **Response Time**: Response times within acceptable limits
- **Memory Usage**: Memory usage within configured JVM limits
- **CPU Usage**: CPU usage within acceptable thresholds under normal load

#### Load Testing Tools
```bash
# Using Apache Bench
ab -n 1000 -c 10 -H "Authorization: Bearer {token}" \
   http://localhost:8088/emailagent/emailservice/ping

# Using JMeter for complex scenarios
# Test calendar operations, user management, availability checking
```

---

*This section provides comprehensive performance tuning guidance for optimizing the Email Agent microservice in production environments.*
