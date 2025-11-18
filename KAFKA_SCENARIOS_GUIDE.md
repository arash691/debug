# Kafka Problem-Solution Scenarios Guide

A comprehensive scenario-based guide for Site Reliability Engineers and Senior Java Developers working with Apache Kafka.

## 📋 **Table of Contents**

### **SRE Operations Scenarios**
1. [High Consumer Lag During Peak Hours](#scenario-1-high-lag)
2. [Kafka Broker Disk Full](#scenario-2-disk-full)
3. [Consumer Group Stuck - Not Processing](#scenario-3-stuck-consumer)
4. [Under-Replicated Partitions After Deployment](#scenario-4-under-replicated)
5. [Message Loss During Broker Failure](#scenario-5-message-loss)
6. [Kafka Performance Degradation](#scenario-6-performance)
7. [Topic Compaction Not Working](#scenario-7-compaction)
8. [Rebalancing Hell - Consumers Keep Rebalancing](#scenario-8-rebalancing)

### **Java Development Scenarios**
9. [Producer Timeout Exceptions in Production](#scenario-9-producer-timeout)
10. [Consumer Processing Too Slow](#scenario-10-slow-consumer)
11. [Duplicate Message Processing](#scenario-11-duplicates)
12. [Poison Message Blocking Consumer](#scenario-12-poison-message)
13. [Out of Memory in Consumer Application](#scenario-13-oom)
14. [Dead Letter Queue Implementation](#scenario-14-dlq)
15. [Exactly-Once Semantics Implementation](#scenario-15-exactly-once)
16. [Handling Large Messages](#scenario-16-large-messages)

### **Advanced Scenarios**
17. [Zero-Downtime Topic Configuration Change](#scenario-17-config-change)
18. [Kafka Cluster Migration](#scenario-18-migration)
19. [Multi-Datacenter Replication Issues](#scenario-19-replication)
20. [Schema Evolution and Compatibility](#scenario-20-schema)

---

## **SRE Operations Scenarios**

---

## <a name="scenario-1-high-lag"></a>**Scenario 1: High Consumer Lag During Peak Hours**

### **📋 Context**
Your monitoring system alerts you at 10 AM on a Monday. Consumer lag for the `order-events` topic has spiked from 1,000 to 150,000 messages. Business is complaining that orders are not being processed in real-time.

### **🔍 Problem Symptoms**
- Consumer lag growing continuously
- Consumers appear to be running but not catching up
- Messages are piling up in Kafka
- Business impact: delayed order processing

### **🔎 Investigation Steps**

```bash
# 1. Check consumer lag immediately
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-processor-group \
  --describe

# Output shows:
# TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-events    0          500000          550000          50000
# order-events    1          480000          530000          50000
# order-events    2          490000          540000          50000

# 2. Check consumer pod status
kubectl get pods -n order-service -l app=order-consumer

# 3. Check consumer logs for errors
kubectl logs order-consumer-xyz -n order-service --tail=100

# 4. Check incoming message rate
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic order-events \
  --time -1

# 5. Check if consumers are alive
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-processor-group \
  --members
```

### **🎯 Root Cause Analysis**

Check logs and find:
```
[Consumer] Processing message took 15000ms
[Consumer] Database connection pool exhausted
[Consumer] Waiting for database connection...
```

**Root Cause:** Consumer is slow due to database bottleneck.

### **✅ Solution**

**Immediate Action (Stop the Bleeding):**

```bash
# 1. Scale up consumers immediately
kubectl scale deployment order-consumer -n order-service --replicas=12

# Monitor lag reduction
watch -n 5 'kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-processor-group \
  --describe | grep LAG'
```

**Short-term Fix (Within Hours):**

```java
// Optimize consumer configuration
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor-group");
        
        // Increase fetch size to process more messages per poll
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100); // Was 500, reduce to process faster
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024 * 1024); // 1MB
        
        // Increase session timeout to avoid rebalancing during slow processing
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 minutes
        
        return new DefaultKafkaConsumerFactory<>(props);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        
        // Increase concurrency
        factory.setConcurrency(3); // 3 threads per pod
        
        return factory;
    }
}

// Optimize database access
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    private static final int BATCH_SIZE = 50;
    private List<Order> orderBatch = new ArrayList<>();
    
    @KafkaListener(topics = "order-events", groupId = "order-processor-group")
    public void processOrder(OrderEvent event) {
        // Batch database writes instead of one-by-one
        orderBatch.add(convertToOrder(event));
        
        if (orderBatch.size() >= BATCH_SIZE) {
            orderRepository.saveAll(orderBatch);
            orderBatch.clear();
        }
    }
    
    // Flush remaining on shutdown
    @PreDestroy
    public void flushBatch() {
        if (!orderBatch.isEmpty()) {
            orderRepository.saveAll(orderBatch);
        }
    }
}
```

**Long-term Fix (Within Days):**

1. **Increase Database Connection Pool:**

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 50  # Was 10
      minimum-idle: 10
      connection-timeout: 30000
```

2. **Add Database Read Replicas:**

```java
@Configuration
public class DatabaseConfig {
    
    @Bean
    @ConfigurationProperties("spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DatabaseType.WRITE, writeDataSource());
        targetDataSources.put(DatabaseType.READ, readDataSource());
        
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(writeDataSource());
        
        return routingDataSource;
    }
}
```

3. **Increase Topic Partitions:**

```bash
# Increase partitions to allow more parallel consumers
kubectl exec -n kafka kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --topic order-events \
  --partitions 12  # Was 3

# Scale consumers to match
kubectl scale deployment order-consumer -n order-service --replicas=12
```

### **📊 Monitoring**

```yaml
# Add to Prometheus alerts
- alert: KafkaConsumerLagHigh
  expr: kafka_consumergroup_lag{group="order-processor-group"} > 10000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High consumer lag detected"
    
- alert: KafkaConsumerLagCritical
  expr: kafka_consumergroup_lag{group="order-processor-group"} > 50000
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Critical consumer lag - immediate action required"
```

### **📝 Post-Mortem Actions**
- [ ] Document the incident
- [ ] Set up alerting for database connection pool exhaustion
- [ ] Review and optimize slow database queries
- [ ] Implement consumer performance monitoring
- [ ] Consider caching frequently accessed data

---

## <a name="scenario-2-disk-full"></a>**Scenario 2: Kafka Broker Disk Full**

### **📋 Context**
3 AM alert: Kafka broker-1 disk usage is at 95%. Producers are starting to fail with "NOT_ENOUGH_REPLICAS" errors.

### **🔍 Problem Symptoms**
- Disk usage above 90% on broker
- Producer errors increasing
- Some partitions becoming unavailable
- Log segments not being cleaned up

### **🔎 Investigation Steps**

```bash
# 1. Check disk usage on all brokers
for i in 0 1 2; do
  echo "=== Broker kafka-$i ==="
  kubectl exec -n kafka kafka-$i -- df -h /var/lib/kafka/data
done

# Output:
# === Broker kafka-1 ===
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       100G   95G    5G  95% /var/lib/kafka/data

# 2. Find largest topics
kubectl exec -n kafka kafka-1 -- du -sh /var/lib/kafka/data/* | sort -rh | head -20

# 3. Check retention policies
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --all \
  --describe | grep retention

# 4. Check log cleaner status
kubectl logs kafka-1 -n kafka | grep "log-cleaner" | tail -50
```

### **🎯 Root Cause Analysis**

```bash
# Find the culprit
kubectl exec -n kafka kafka-1 -- du -sh /var/lib/kafka/data/* | sort -rh

# Output shows:
# 40G    /var/lib/kafka/data/analytics-events-0
# 35G    /var/lib/kafka/data/analytics-events-1
# 15G    /var/lib/kafka/data/user-activity-0
```

**Root Cause:** `analytics-events` topic has retention set to 30 days with high throughput, filling up disk.

### **✅ Solution**

**Immediate Action (Emergency - Within 15 minutes):**

```bash
# Option 1: Reduce retention temporarily for problematic topic
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name analytics-events \
  --alter \
  --add-config retention.ms=3600000  # 1 hour (was 30 days)

# Wait 5-10 minutes for cleanup to trigger
# Monitor disk usage
watch 'kubectl exec -n kafka kafka-1 -- df -h /var/lib/kafka/data'

# Option 2: Force log cleanup (if urgent)
kubectl exec -n kafka kafka-1 -- kafka-run-class.sh kafka.admin.DeleteRecordsCommand \
  --bootstrap-server localhost:9092 \
  --offset-json-file /tmp/delete-offsets.json

# Create delete-offsets.json:
cat > delete-offsets.json << EOF
{
  "partitions": [
    {
      "topic": "analytics-events",
      "partition": 0,
      "offset": 1000000
    },
    {
      "topic": "analytics-events", 
      "partition": 1,
      "offset": 1000000
    }
  ],
  "version": 1
}
EOF

kubectl cp delete-offsets.json kafka/kafka-1:/tmp/delete-offsets.json
```

**Short-term Fix (Within 1 hour):**

```bash
# Set proper retention for analytics topic
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name analytics-events \
  --alter \
  --add-config retention.ms=86400000  # 1 day instead of 30

# Increase segment size to improve compaction efficiency
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name analytics-events \
  --alter \
  --add-config segment.ms=3600000,segment.bytes=536870912  # 1 hour or 512MB
```

**Long-term Fix (Within 1 week):**

1. **Increase Disk Size:**

```bash
# Expand PVC (if storage class supports it)
kubectl patch pvc data-kafka-1 -n kafka -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'

# Wait for expansion
kubectl get pvc -n kafka -w
```

2. **Implement Tiered Storage (if using Kafka 2.8+):**

```yaml
# kafka-config.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-config
  namespace: kafka
data:
  server.properties: |
    # Enable tiered storage
    remote.log.storage.system.enable=true
    remote.log.storage.manager.class.name=org.apache.kafka.server.log.remote.storage.RemoteLogManager
    
    # S3 configuration
    remote.log.storage.manager.class.path=/opt/kafka/libs/kafka-tiered-storage-s3.jar
    rsm.config.remote.log.storage.manager.impl.prefix=s3.
    s3.bucket.name=kafka-tiered-storage
    s3.region=us-east-1
```

3. **Set Up Data Pipeline to Archive:**

```java
@Service
@Slf4j
public class AnalyticsArchiver {
    
    @Scheduled(cron = "0 0 2 * * ?") // Daily at 2 AM
    public void archiveOldData() {
        log.info("Starting analytics data archival...");
        
        Consumer<String, AnalyticsEvent> consumer = createConsumer();
        
        // Consume old data
        consumer.subscribe(Collections.singleton("analytics-events"));
        
        List<AnalyticsEvent> batch = new ArrayList<>();
        
        while (true) {
            ConsumerRecords<String, AnalyticsEvent> records = consumer.poll(Duration.ofSeconds(10));
            
            if (records.isEmpty()) {
                break;
            }
            
            records.forEach(record -> {
                // Only archive if older than 7 days
                if (record.timestamp() < System.currentTimeMillis() - (7 * 24 * 3600 * 1000)) {
                    batch.add(record.value());
                }
                
                if (batch.size() >= 1000) {
                    // Write to S3/Data Lake
                    archiveToS3(batch);
                    batch.clear();
                }
            });
        }
        
        if (!batch.isEmpty()) {
            archiveToS3(batch);
        }
        
        log.info("Analytics archival completed");
    }
    
    private void archiveToS3(List<AnalyticsEvent> events) {
        String fileName = "analytics-" + LocalDate.now() + "-" + UUID.randomUUID() + ".parquet";
        s3Client.putObject(PutObjectRequest.builder()
            .bucket("analytics-archive")
            .key(fileName)
            .build(), 
            convertToParquet(events));
    }
}
```

4. **Implement Automatic Retention Management:**

```java
@Service
@Slf4j
public class KafkaRetentionManager {
    
    @Scheduled(cron = "0 0 * * * ?") // Every hour
    public void adjustRetention() {
        AdminClient adminClient = createAdminClient();
        
        // Get all topics
        Set<String> topics = adminClient.listTopics().names().get();
        
        for (String topic : topics) {
            // Get topic size
            long topicSize = getTopicSize(topic);
            
            // If topic is too large, reduce retention
            if (topicSize > 50_000_000_000L) { // 50GB
                log.warn("Topic {} is too large ({}GB), reducing retention", 
                    topic, topicSize / 1_000_000_000);
                
                ConfigResource resource = new ConfigResource(ConfigResource.Type.TOPIC, topic);
                Config config = new Config(Arrays.asList(
                    new ConfigEntry("retention.ms", "86400000") // 1 day
                ));
                
                Map<ConfigResource, Config> configs = new HashMap<>();
                configs.put(resource, config);
                
                adminClient.alterConfigs(configs).all().get();
            }
        }
    }
}
```

### **📊 Monitoring**

```yaml
# Prometheus alerts
- alert: KafkaBrokerDiskHigh
  expr: (kafka_log_logmanager_size / kafka_log_logmanager_size_limit) > 0.80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Kafka broker {{ $labels.instance }} disk usage above 80%"

- alert: KafkaBrokerDiskCritical
  expr: (kafka_log_logmanager_size / kafka_log_logmanager_size_limit) > 0.90
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Kafka broker {{ $labels.instance }} disk usage above 90% - URGENT"
```

### **📝 Prevention Checklist**
- [ ] Set appropriate retention policies for all topics
- [ ] Monitor disk usage with alerts at 70%, 80%, 90%
- [ ] Implement automated retention adjustment
- [ ] Document retention requirements per topic
- [ ] Set up tiered storage for long-term data
- [ ] Regular capacity planning reviews

---

## <a name="scenario-3-stuck-consumer"></a>**Scenario 3: Consumer Group Stuck - Not Processing**

### **📋 Context**
Your payment processing consumer has been running for 2 hours with no errors in logs, but no messages are being processed. Consumer lag is not changing.

### **🔍 Problem Symptoms**
- Consumer pods are running (no crashes)
- No errors in logs
- Consumer lag is static (not increasing or decreasing)
- No rebalancing events
- Messages visible in topic but not consumed

### **🔎 Investigation Steps**

```bash
# 1. Check consumer group state
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-processor \
  --describe

# Look for:
# - CONSUMER-ID column: is it empty? (no active consumers)
# - LAG column: is it static?
# - HOST column: are consumers connected?

# 2. Check consumer members
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-processor \
  --members \
  --verbose

# 3. Check consumer state
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-processor \
  --state

# 4. Check application logs
kubectl logs payment-processor-xyz -n payment-service --tail=500

# 5. Check if consumer is actually running
kubectl exec payment-processor-xyz -n payment-service -- ps aux | grep java
```

### **🎯 Root Cause Analysis**

**Scenario A: Consumer is paused in code**

```bash
# Check logs and find:
[WARN] Consumer has been paused due to errors
[ERROR] Failed to process message 10 times, pausing consumer
```

**Scenario B: All partitions assigned to dead consumers**

```bash
# kafka-consumer-groups output shows:
TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID                    HOST
payment-events 0          1000           2000            1000   consumer-1-dead                /10.0.1.1
payment-events 1          1500           2500            1000   consumer-2-dead                /10.0.1.2
payment-events 2          2000           3000            1000   (no active consumer)           -
```

**Scenario C: Processing thread is stuck**

```bash
# Thread dump shows:
"KafkaConsumer-thread-1" waiting for lock
   java.lang.Thread.State: BLOCKED
```

### **✅ Solution**

**For Scenario A: Paused Consumer**

```java
// Problem code that pauses on errors
@KafkaListener(topics = "payment-events")
public void processPayment(PaymentEvent event) {
    try {
        paymentService.process(event);
    } catch (Exception e) {
        // DON'T DO THIS - pauses entire consumer
        kafkaListenerEndpointRegistry.getListenerContainer("paymentListener").pause();
        throw e;
    }
}

// Solution: Implement proper error handling
@Service
@Slf4j
public class PaymentConsumer {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> dlqTemplate;
    
    private final AtomicInteger errorCount = new AtomicInteger(0);
    private static final int MAX_ERRORS = 10;
    
    @KafkaListener(
        topics = "payment-events",
        groupId = "payment-processor",
        errorHandler = "kafkaErrorHandler"
    )
    public void processPayment(
        @Payload PaymentEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.OFFSET) long offset
    ) {
        try {
            paymentService.process(event);
            errorCount.set(0); // Reset on success
            
        } catch (RetryableException e) {
            log.warn("Retryable error processing payment {}, will retry", event.getId(), e);
            throw e; // Let Spring Kafka retry
            
        } catch (NonRetryableException e) {
            log.error("Non-retryable error processing payment {}, sending to DLQ", event.getId(), e);
            sendToDLQ(event, e);
            
        } catch (Exception e) {
            log.error("Unexpected error processing payment {}", event.getId(), e);
            
            if (errorCount.incrementAndGet() > MAX_ERRORS) {
                log.error("Too many errors ({}), sending to DLQ", errorCount.get());
                sendToDLQ(event, e);
                errorCount.set(0);
            } else {
                throw e; // Retry
            }
        }
    }
    
    private void sendToDLQ(PaymentEvent event, Exception e) {
        try {
            dlqTemplate.send("payment-events-dlq", event.getId(), event);
        } catch (Exception dlqError) {
            log.error("Failed to send to DLQ!", dlqError);
            // Last resort: write to file/database
            emergencyPersist(event, e);
        }
    }
}

// Configure error handler
@Bean
public KafkaListenerErrorHandler kafkaErrorHandler() {
    return (message, exception) -> {
        log.error("Error in consumer: {}", exception.getMessage());
        // Don't pause consumer
        return null;
    };
}
```

**For Scenario B: Dead Consumers Still Assigned**

```bash
# Force consumer group reset
# First, stop all consumers
kubectl scale deployment payment-processor -n payment-service --replicas=0

# Wait 30 seconds

# Delete consumer group offsets (will be recreated)
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-processor \
  --delete

# Restart consumers
kubectl scale deployment payment-processor -n payment-service --replicas=3

# Monitor recovery
watch 'kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group payment-processor \
  --describe'
```

**For Scenario C: Stuck Processing Thread**

```java
// Problem: Synchronous blocking call
@KafkaListener(topics = "payment-events")
public void processPayment(PaymentEvent event) {
    // This blocks the consumer thread!
    String result = externalApiClient.callSync(event); // Takes 30 seconds
    paymentService.save(result);
}

// Solution 1: Use async processing with thread pool
@Configuration
public class AsyncConfig {
    
    @Bean
    public Executor paymentProcessorExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("payment-processor-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
@Slf4j
public class PaymentConsumer {
    
    @Autowired
    @Qualifier("paymentProcessorExecutor")
    private Executor executor;
    
    @Autowired
    private PaymentService paymentService;
    
    @KafkaListener(
        topics = "payment-events",
        groupId = "payment-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void processPayment(PaymentEvent event, Acknowledgment acknowledgment) {
        // Submit to thread pool immediately
        CompletableFuture.runAsync(() -> {
            try {
                paymentService.processAsync(event);
                acknowledgment.acknowledge(); // Manual commit after success
            } catch (Exception e) {
                log.error("Error processing payment", e);
                // Handle error
            }
        }, executor);
    }
}

// Configure manual acknowledgment
@Bean
public ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> 
    kafkaListenerContainerFactory() {
    
    ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> factory = 
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
    return factory;
}

// Solution 2: Add timeout to external calls
@Service
public class ExternalApiClient {
    
    private final RestTemplate restTemplate;
    
    public ExternalApiClient() {
        SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(5000);  // 5 seconds
        factory.setReadTimeout(10000);     // 10 seconds
        this.restTemplate = new RestTemplate(factory);
    }
    
    @CircuitBreaker(name = "externalApi", fallbackMethod = "fallback")
    @TimeLimiter(name = "externalApi")
    public CompletableFuture<String> callAsync(PaymentEvent event) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return restTemplate.postForObject(apiUrl, event, String.class);
            } catch (Exception e) {
                log.error("External API call failed", e);
                throw e;
            }
        });
    }
    
    public CompletableFuture<String> fallback(PaymentEvent event, Exception e) {
        log.warn("Using fallback for payment {}", event.getId());
        return CompletableFuture.completedFuture("FALLBACK");
    }
}
```

### **🔧 Prevention Configuration**

```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        
        // Prevent stuck consumers with proper timeouts
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000); // 30 seconds
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 10000); // 10 seconds
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 minutes
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100); // Don't fetch too many
        
        // Enable auto-offset reset for new consumers
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        return props;
    }
}
```

### **📊 Health Check**

```java
@Component
public class ConsumerHealthIndicator implements HealthIndicator {
    
    @Autowired
    private KafkaListenerEndpointRegistry registry;
    
    private Instant lastMessageTime = Instant.now();
    private static final Duration HEALTHY_THRESHOLD = Duration.ofMinutes(5);
    
    @Override
    public Health health() {
        // Check if consumer containers are running
        Collection<MessageListenerContainer> containers = registry.getListenerContainers();
        
        for (MessageListenerContainer container : containers) {
            if (!container.isRunning()) {
                return Health.down()
                    .withDetail("consumer", container.getListenerId())
                    .withDetail("status", "not running")
                    .build();
            }
        }
        
        // Check if messages are being processed
        if (Duration.between(lastMessageTime, Instant.now()).compareTo(HEALTHY_THRESHOLD) > 0) {
            return Health.down()
                .withDetail("reason", "No messages processed in last 5 minutes")
                .withDetail("lastMessageTime", lastMessageTime)
                .build();
        }
        
        return Health.up()
            .withDetail("lastMessageTime", lastMessageTime)
            .build();
    }
    
    public void recordMessageProcessed() {
        this.lastMessageTime = Instant.now();
    }
}
```

---

## <a name="scenario-9-producer-timeout"></a>**Scenario 4: Producer Timeout Exceptions in Production**

### **📋 Context**
Your order service is experiencing intermittent `TimeoutException` when sending messages to Kafka. Some orders are failing to be created.

### **🔍 Problem Symptoms**
```
org.apache.kafka.common.errors.TimeoutException: 
  Failed to update metadata after 60000 ms.
  
org.apache.kafka.common.errors.TimeoutException:
  Expiring 5 record(s) for order-events-0: 30000 ms has passed
```

### **🔎 Investigation Steps**

```bash
# 1. Check Kafka broker status
kubectl get pods -n kafka
kubectl logs kafka-0 -n kafka --tail=100

# 2. Check network between app and Kafka
kubectl exec order-service-xyz -n order-service -- nc -zv kafka-service 9092
kubectl exec order-service-xyz -n order-service -- curl -v telnet://kafka-service:9092

# 3. Check producer metrics in application
curl http://order-service-xyz:8080/actuator/metrics/kafka.producer

# 4. Check for network policies
kubectl get networkpolicy -n order-service
kubectl describe networkpolicy kafka-access -n order-service

# 5. Check broker request queue
kubectl exec -n kafka kafka-0 -- kafka-run-class.sh kafka.tools.JmxTool \
  --object-name kafka.network:type=RequestChannel,name=RequestQueueSize \
  --jmx-url service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi
```

### **🎯 Root Cause Analysis**

**Common Causes:**
1. Network connectivity issues
2. Broker overloaded
3. Insufficient producer configuration
4. Too many in-flight requests
5. Large message batches

### **✅ Solution**

```java
@Configuration
@Slf4j
public class KafkaProducerConfig {
    
    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;
    
    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        
        // Serializers
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // === CRITICAL TIMEOUT CONFIGURATIONS ===
        
        // 1. Request timeout - time to wait for response from broker
        configProps.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000); // 30 seconds
        
        // 2. Delivery timeout - total time for send() to complete
        configProps.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000); // 2 minutes
        
        // 3. Metadata fetch timeout
        configProps.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 60000); // 1 minute
        
        // === RELIABILITY CONFIGURATIONS ===
        
        // Wait for all replicas to acknowledge (strongest guarantee)
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");
        
        // Number of retries
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);
        
        // Time between retries
        configProps.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 1000); // 1 second
        
        // === PERFORMANCE CONFIGURATIONS ===
        
        // Batch settings
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768); // 32KB
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 10); // Wait 10ms to batch
        
        // Compression
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        // Buffer memory
        configProps.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864); // 64MB
        
        // Max in-flight requests (careful with this!)
        configProps.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        // === IDEMPOTENCE (prevents duplicates) ===
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }
    
    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate() {
        KafkaTemplate<String, OrderEvent> template = new KafkaTemplate<>(producerFactory());
        
        // Set default topic
        template.setDefaultTopic("order-events");
        
        // Add producer listener for monitoring
        template.setProducerListener(new ProducerListener<String, OrderEvent>() {
            @Override
            public void onSuccess(ProducerRecord<String, OrderEvent> record, RecordMetadata metadata) {
                log.debug("Message sent successfully: topic={}, partition={}, offset={}", 
                    metadata.topic(), metadata.partition(), metadata.offset());
            }
            
            @Override
            public void onError(ProducerRecord<String, OrderEvent> record, RecordMetadata metadata, Exception exception) {
                log.error("Error sending message: key={}", record.key(), exception);
            }
        });
        
        return template;
    }
}

// Robust producer service with retry logic
@Service
@Slf4j
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    private static final int MAX_RETRIES = 3;
    private static final String TOPIC = "order-events";
    
    public void sendOrderEvent(OrderEvent event) {
        // Save to database first (outbox pattern)
        Order order = orderRepository.save(convertToOrder(event));
        order.setStatus(OrderStatus.PENDING_KAFKA);
        orderRepository.save(order);
        
        // Send to Kafka with retry
        sendWithRetry(event, order, 0);
    }
    
    private void sendWithRetry(OrderEvent event, Order order, int attempt) {
        try {
            ListenableFuture<SendResult<String, OrderEvent>> future = 
                kafkaTemplate.send(TOPIC, event.getOrderId(), event);
            
            // Add callback
            future.addCallback(
                result -> {
                    // Success
                    log.info("Order event sent successfully: orderId={}, partition={}, offset={}",
                        event.getOrderId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                    
                    // Update order status
                    order.setStatus(OrderStatus.SENT_TO_KAFKA);
                    orderRepository.save(order);
                },
                ex -> {
                    // Failure
                    log.error("Failed to send order event: orderId={}, attempt={}", 
                        event.getOrderId(), attempt, ex);
                    
                    if (attempt < MAX_RETRIES) {
                        // Retry with exponential backoff
                        int backoffMs = (int) Math.pow(2, attempt) * 1000;
                        
                        log.info("Retrying in {}ms... (attempt {}/{})", 
                            backoffMs, attempt + 1, MAX_RETRIES);
                        
                        try {
                            Thread.sleep(backoffMs);
                        } catch (InterruptedException ie) {
                            Thread.currentThread().interrupt();
                        }
                        
                        sendWithRetry(event, order, attempt + 1);
                    } else {
                        // Max retries exceeded - handle failure
                        handleFinalFailure(event, order, ex);
                    }
                }
            );
            
        } catch (Exception e) {
            log.error("Exception while sending to Kafka", e);
            
            if (attempt < MAX_RETRIES) {
                sendWithRetry(event, order, attempt + 1);
            } else {
                handleFinalFailure(event, order, e);
            }
        }
    }
    
    private void handleFinalFailure(OrderEvent event, Order order, Throwable ex) {
        log.error("Failed to send order event after {} retries: orderId={}", 
            MAX_RETRIES, event.getOrderId(), ex);
        
        // Mark as failed in database
        order.setStatus(OrderStatus.KAFKA_FAILED);
        order.setErrorMessage(ex.getMessage());
        orderRepository.save(order);
        
        // Send alert
        alertService.sendAlert("Kafka producer failure", 
            "Failed to send order " + event.getOrderId() + " to Kafka");
        
        // Could also write to a fallback queue or file
        fallbackQueue.add(event);
    }
}

// Background job to retry failed messages
@Component
@Slf4j
public class KafkaRetryJob {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private OrderEventProducer producer;
    
    @Scheduled(fixedDelay = 60000) // Every minute
    public void retryFailedMessages() {
        List<Order> failedOrders = orderRepository.findByStatus(OrderStatus.KAFKA_FAILED);
        
        if (failedOrders.isEmpty()) {
            return;
        }
        
        log.info("Found {} failed orders to retry", failedOrders.size());
        
        for (Order order : failedOrders) {
            try {
                OrderEvent event = convertToEvent(order);
                producer.sendOrderEvent(event);
            } catch (Exception e) {
                log.error("Failed to retry order {}", order.getId(), e);
            }
        }
    }
}
```

### **🔧 Circuit Breaker Pattern**

```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public CircuitBreakerRegistry circuitBreakerRegistry() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50) // Open circuit if 50% failures
            .waitDurationInOpenState(Duration.ofSeconds(30)) // Wait 30s before trying again
            .slidingWindowSize(10) // Consider last 10 calls
            .minimumNumberOfCalls(5) // Need at least 5 calls to calculate rate
            .build();
        
        return CircuitBreakerRegistry.of(config);
    }
}

@Service
@Slf4j
public class ResilientOrderProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    private final CircuitBreaker circuitBreaker;
    
    public ResilientOrderProducer(CircuitBreakerRegistry registry) {
        this.circuitBreaker = registry.circuitBreaker("kafka-producer");
    }
    
    public void sendOrder(OrderEvent event) {
        // Wrap in circuit breaker
        Try<SendResult<String, OrderEvent>> result = Try.ofSupplier(
            CircuitBreaker.decorateSupplier(circuitBreaker, () -> {
                try {
                    return kafkaTemplate.send("order-events", event).get(30, TimeUnit.SECONDS);
                } catch (Exception e) {
                    throw new RuntimeException("Failed to send to Kafka", e);
                }
            })
        );
        
        result
            .onSuccess(r -> log.info("Order sent: {}", event.getOrderId()))
            .onFailure(ex -> {
                if (ex instanceof CallNotPermittedException) {
                    log.error("Circuit breaker is OPEN - Kafka is unavailable");
                    // Use fallback mechanism
                    saveToDatabaseForLaterRetry(event);
                } else {
                    log.error("Failed to send order", ex);
                }
            });
    }
    
    private void saveToDatabaseForLaterRetry(OrderEvent event) {
        // Implementation for storing failed messages
    }
}
```

### **📊 Monitoring**

```java
@Component
public class KafkaProducerMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter successCounter;
    private final Counter failureCounter;
    private final Timer sendTimer;
    
    public KafkaProducerMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.successCounter = Counter.builder("kafka.producer.success")
            .description("Number of successful sends")
            .register(meterRegistry);
        this.failureCounter = Counter.builder("kafka.producer.failure")
            .description("Number of failed sends")
            .register(meterRegistry);
        this.sendTimer = Timer.builder("kafka.producer.send.duration")
            .description("Time taken to send message")
            .register(meterRegistry);
    }
    
    public void recordSuccess() {
        successCounter.increment();
    }
    
    public void recordFailure() {
        failureCounter.increment();
    }
    
    public void recordSendTime(long milliseconds) {
        sendTimer.record(milliseconds, TimeUnit.MILLISECONDS);
    }
}
```

---

## <a name="scenario-11-duplicates"></a>**Scenario 5: Duplicate Message Processing**

### **📋 Context**
Your payment processing system is charging customers twice for the same order. Kafka is delivering duplicate messages.

### **🔍 Problem Symptoms**
- Same message ID processed multiple times
- Customers complaining about double charges
- Database has duplicate records
- Logs show same order processed twice

### **🔎 Root Cause Analysis**

**Common Causes:**
1. Producer retries without idempotence
2. Consumer rebalancing during processing
3. Consumer crash before committing offset
4. At-least-once delivery semantics

### **✅ Solution: Idempotent Consumer**

```java
@Service
@Slf4j
public class IdempotentPaymentProcessor {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Autowired
    private ProcessedMessageRepository processedMessageRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @KafkaListener(topics = "payment-events", groupId = "payment-processor")
    @Transactional
    public void processPayment(
        @Payload PaymentEvent event,
        @Header(KafkaHeaders.RECEIVED_MESSAGE_KEY) String messageKey,
        @Header(KafkaHeaders.OFFSET) long offset,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic
    ) {
        String messageId = event.getPaymentId();
        String idempotencyKey = String.format("%s-%s-%d-%d", topic, messageKey, partition, offset);
        
        log.info("Processing payment: paymentId={}, idempotencyKey={}", messageId, idempotencyKey);
        
        // Check if already processed using database
        if (isAlreadyProcessed(idempotencyKey)) {
            log.warn("Payment already processed (duplicate): paymentId={}, idempotencyKey={}", 
                messageId, idempotencyKey);
            return; // Skip duplicate
        }
        
        try {
            // Process payment
            Payment payment = processPaymentLogic(event);
            
            // Save payment and processed message ID in same transaction
            paymentRepository.save(payment);
            recordProcessed(idempotencyKey, messageId);
            
            log.info("Payment processed successfully: paymentId={}", messageId);
            
        } catch (DuplicateKeyException e) {
            // Race condition - another instance processed it
            log.warn("Duplicate detected via database constraint: paymentId={}", messageId);
            // Don't throw exception - message is processed
            
        } catch (Exception e) {
            log.error("Error processing payment: paymentId={}", messageId, e);
            throw e; // Will retry
        }
    }
    
    private boolean isAlreadyProcessed(String idempotencyKey) {
        return processedMessageRepository.existsByIdempotencyKey(idempotencyKey);
    }
    
    private void recordProcessed(String idempotencyKey, String messageId) {
        ProcessedMessage processed = new ProcessedMessage();
        processed.setIdempotencyKey(idempotencyKey);
        processed.setMessageId(messageId);
        processed.setProcessedAt(Instant.now());
        processedMessageRepository.save(processed);
    }
    
    private Payment processPaymentLogic(PaymentEvent event) {
        // Business logic here
        Payment payment = new Payment();
        payment.setPaymentId(event.getPaymentId());
        payment.setAmount(event.getAmount());
        payment.setStatus(PaymentStatus.COMPLETED);
        return payment;
    }
}

// Entity for tracking processed messages
@Entity
@Table(name = "processed_messages",
       indexes = @Index(name = "idx_idempotency_key", columnList = "idempotency_key"),
       uniqueConstraints = @UniqueConstraint(columnNames = "idempotency_key"))
public class ProcessedMessage {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "idempotency_key", unique = true, nullable = false)
    private String idempotencyKey;
    
    @Column(name = "message_id", nullable = false)
    private String messageId;
    
    @Column(name = "processed_at", nullable = false)
    private Instant processedAt;
    
    // Getters and setters
}

// Repository
public interface ProcessedMessageRepository extends JpaRepository<ProcessedMessage, Long> {
    boolean existsByIdempotencyKey(String idempotencyKey);
}

// Cleanup old entries periodically
@Component
@Slf4j
public class ProcessedMessageCleanup {
    
    @Autowired
    private ProcessedMessageRepository repository;
    
    @Scheduled(cron = "0 0 2 * * ?") // Daily at 2 AM
    public void cleanupOldMessages() {
        Instant cutoff = Instant.now().minus(7, ChronoUnit.DAYS);
        int deleted = repository.deleteByProcessedAtBefore(cutoff);
        log.info("Cleaned up {} old processed message records", deleted);
    }
}
```

### **Alternative: Using Redis for Deduplication**

```java
@Service
@Slf4j
public class RedisIdempotentPaymentProcessor {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    @Autowired
    private PaymentService paymentService;
    
    private static final String PROCESSED_KEY_PREFIX = "processed:payment:";
    private static final Duration TTL = Duration.ofDays(7);
    
    @KafkaListener(topics = "payment-events", groupId = "payment-processor")
    public void processPayment(@Payload PaymentEvent event) {
        String messageId = event.getPaymentId();
        String redisKey = PROCESSED_KEY_PREFIX + messageId;
        
        // Try to set key with NX (only if not exists)
        Boolean wasSet = redisTemplate.opsForValue()
            .setIfAbsent(redisKey, "processing", TTL);
        
        if (Boolean.FALSE.equals(wasSet)) {
            log.warn("Payment already processed (duplicate): paymentId={}", messageId);
            return; // Duplicate, skip
        }
        
        try {
            // Process payment
            paymentService.process(event);
            
            // Update Redis value to mark as completed
            redisTemplate.opsForValue().set(redisKey, "completed", TTL);
            
            log.info("Payment processed successfully: paymentId={}", messageId);
            
        } catch (Exception e) {
            log.error("Error processing payment: paymentId={}", messageId, e);
            
            // Delete Redis key to allow retry
            redisTemplate.delete(redisKey);
            
            throw e; // Will retry
        }
    }
}
```

### **Database Unique Constraint Approach**

```java
@Entity
@Table(name = "payments",
       uniqueConstraints = @UniqueConstraint(columnNames = {"payment_id"}))
public class Payment {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "payment_id", unique = true, nullable = false)
    private String paymentId; // From Kafka message
    
    @Column(name = "amount", nullable = false)
    private BigDecimal amount;
    
    @Column(name = "status", nullable = false)
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;
    
    // Other fields
}

@Service
@Slf4j
public class PaymentProcessor {
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @KafkaListener(topics = "payment-events")
    @Transactional
    public void processPayment(@Payload PaymentEvent event) {
        try {
            // Try to save - will fail if duplicate due to unique constraint
            Payment payment = new Payment();
            payment.setPaymentId(event.getPaymentId());
            payment.setAmount(event.getAmount());
            payment.setStatus(PaymentStatus.COMPLETED);
            
            paymentRepository.save(payment);
            
            log.info("Payment processed: paymentId={}", event.getPaymentId());
            
        } catch (DataIntegrityViolationException e) {
            // Duplicate payment ID - this is fine, message already processed
            log.warn("Duplicate payment detected: paymentId={}", event.getPaymentId());
            // Don't rethrow - consider it successfully processed
            
        } catch (Exception e) {
            log.error("Error processing payment: paymentId={}", event.getPaymentId(), e);
            throw e; // Will retry
        }
    }
}
```

### **Producer-Side: Enable Idempotence**

```java
@Configuration
public class KafkaProducerConfig {
    
    @Bean
    public ProducerFactory<String, PaymentEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        
        // Enable idempotent producer (prevents duplicates from retries)
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // These are automatically set when idempotence is enabled:
        // acks=all
        // retries=Integer.MAX_VALUE
        // max.in.flight.requests.per.connection=5
        
        // But you can override if needed:
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### **Testing for Duplicates**

```java
@SpringBootTest
class DuplicateMessageTest {
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Test
    void testDuplicateMessageHandling() throws Exception {
        PaymentEvent event = new PaymentEvent();
        event.setPaymentId("TEST-123");
        event.setAmount(new BigDecimal("100.00"));
        
        // Send same message twice
        kafkaTemplate.send("payment-events", event.getPaymentId(), event);
        kafkaTemplate.send("payment-events", event.getPaymentId(), event);
        
        // Wait for processing
        Thread.sleep(2000);
        
        // Should only have one payment
        List<Payment> payments = paymentRepository.findByPaymentId("TEST-123");
        assertThat(payments).hasSize(1);
        assertThat(payments.get(0).getAmount()).isEqualTo(new BigDecimal("100.00"));
    }
}
```

---

## <a name="scenario-12-poison-message"></a>**Scenario 6: Poison Message Blocking Consumer**

### **📋 Context**
Your user notification consumer has stopped processing. It keeps trying to process one message, failing, and retrying indefinitely. All messages behind it are blocked.

### **🔍 Problem Symptoms**
```
[ERROR] Failed to process message, attempt 1
[ERROR] Failed to process message, attempt 2
[ERROR] Failed to process message, attempt 3
... (repeating forever)
```

### **🔎 Investigation**

```bash
# Check consumer lag - it's stuck
kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group notification-consumer \
  --describe

# Logs show same offset retrying
kubectl logs notification-consumer-xyz -n notification-service | grep "offset="
```

### **✅ Solution: Dead Letter Queue Pattern**

```java
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, NotificationEvent> 
        kafkaListenerContainerFactory() {
        
        ConcurrentKafkaListenerContainerFactory<String, NotificationEvent> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        
        factory.setConsumerFactory(consumerFactory());
        
        // Configure error handler with DLQ
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate(),
                (record, ex) -> {
                    // Send to DLQ topic
                    return new TopicPartition(
                        record.topic() + "-dlq",
                        record.partition()
                    );
                }),
            new FixedBackOff(1000L, 3L) // Retry 3 times with 1s delay
        ));
        
        return factory;
    }
}

@Service
@Slf4j
public class NotificationConsumer {
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private KafkaTemplate<String, NotificationEvent> dlqTemplate;
    
    @KafkaListener(
        topics = "notification-events",
        groupId = "notification-consumer"
    )
    public void processNotification(
        @Payload NotificationEvent event,
        @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.OFFSET) long offset
    ) {
        try {
            log.info("Processing notification: id={}, offset={}", event.getId(), offset);
            
            // Validate message
            validateMessage(event);
            
            // Process
            notificationService.send(event);
            
        } catch (ValidationException e) {
            // Don't retry validation errors - send to DLQ immediately
            log.error("Validation error for notification {}: {}", event.getId(), e.getMessage());
            sendToDLQ(event, e, "VALIDATION_ERROR");
            // Don't rethrow - consider it handled
            
        } catch (PoisonMessageException e) {
            // Known poison message - send to DLQ
            log.error("Poison message detected: {}", event.getId(), e);
            sendToDLQ(event, e, "POISON_MESSAGE");
            // Don't rethrow
            
        } catch (Exception e) {
            // Unexpected error - let it retry (will go to DLQ after max retries)
            log.error("Error processing notification: {}", event.getId(), e);
            throw e;
        }
    }
    
    private void validateMessage(NotificationEvent event) throws ValidationException {
        if (event.getId() == null) {
            throw new ValidationException("Message ID is null");
        }
        if (event.getUserId() == null) {
            throw new ValidationException("User ID is null");
        }
        if (event.getMessage() == null || event.getMessage().isEmpty()) {
            throw new ValidationException("Message content is empty");
        }
    }
    
    private void sendToDLQ(NotificationEvent event, Exception error, String reason) {
        try {
            // Add error information as headers
            ProducerRecord<String, NotificationEvent> record = 
                new ProducerRecord<>("notification-events-dlq", event.getId(), event);
            
            record.headers().add("error-reason", reason.getBytes());
            record.headers().add("error-message", error.getMessage().getBytes());
            record.headers().add("error-timestamp", 
                Instant.now().toString().getBytes());
            record.headers().add("original-topic", 
                "notification-events".getBytes());
            
            dlqTemplate.send(record);
            
            log.info("Sent to DLQ: id={}, reason={}", event.getId(), reason);
            
        } catch (Exception dlqError) {
            log.error("Failed to send to DLQ!", dlqError);
            // Last resort: save to database
            saveToDatabaseDLQ(event, error, reason);
        }
    }
    
    private void saveToDatabaseDLQ(NotificationEvent event, Exception error, String reason) {
        // Implementation to save failed messages to database
    }
}

// DLQ Consumer for manual review and reprocessing
@Service
@Slf4j
public class DLQConsumer {
    
    @Autowired
    private DLQRepository dlqRepository;
    
    @KafkaListener(
        topics = "notification-events-dlq",
        groupId = "dlq-consumer",
        autoStartup = "false" // Don't start automatically
    )
    public void processDLQMessage(
        @Payload NotificationEvent event,
        @Header("error-reason") String errorReason,
        @Header("error-message") String errorMessage,
        @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp
    ) {
        log.info("DLQ message received: id={}, reason={}", event.getId(), errorReason);
        
        // Save to database for manual review
        DLQRecord record = new DLQRecord();
        record.setMessageId(event.getId());
        record.setPayload(serializeToJson(event));
        record.setErrorReason(errorReason);
        record.setErrorMessage(errorMessage);
        record.setReceivedAt(Instant.ofEpochMilli(timestamp));
        record.setStatus(DLQStatus.PENDING_REVIEW);
        
        dlqRepository.save(record);
    }
}

// REST API to reprocess DLQ messages
@RestController
@RequestMapping("/api/dlq")
@Slf4j
public class DLQController {
    
    @Autowired
    private DLQRepository dlqRepository;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private KafkaTemplate<String, NotificationEvent> kafkaTemplate;
    
    @GetMapping
    public Page<DLQRecord> getDLQMessages(Pageable pageable) {
        return dlqRepository.findByStatus(DLQStatus.PENDING_REVIEW, pageable);
    }
    
    @PostMapping("/{id}/reprocess")
    public ResponseEntity<?> reprocessMessage(@PathVariable Long id) {
        DLQRecord record = dlqRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("DLQ record not found"));
        
        try {
            NotificationEvent event = deserializeFromJson(record.getPayload());
            
            // Try to process directly
            notificationService.send(event);
            
            // Mark as resolved
            record.setStatus(DLQStatus.RESOLVED);
            record.setResolvedAt(Instant.now());
            dlqRepository.save(record);
            
            log.info("DLQ message reprocessed successfully: id={}", id);
            return ResponseEntity.ok().build();
            
        } catch (Exception e) {
            log.error("Failed to reprocess DLQ message: id={}", id, e);
            return ResponseEntity.status(500).body(e.getMessage());
        }
    }
    
    @PostMapping("/{id}/retry")
    public ResponseEntity<?> retryToKafka(@PathVariable Long id) {
        DLQRecord record = dlqRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("DLQ record not found"));
        
        try {
            NotificationEvent event = deserializeFromJson(record.getPayload());
            
            // Send back to original topic
            kafkaTemplate.send("notification-events", event.getId(), event);
            
            record.setStatus(DLQStatus.RETRIED);
            record.setResolvedAt(Instant.now());
            dlqRepository.save(record);
            
            return ResponseEntity.ok().build();
            
        } catch (Exception e) {
            return ResponseEntity.status(500).body(e.getMessage());
        }
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<?> discardMessage(@PathVariable Long id) {
        DLQRecord record = dlqRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("DLQ record not found"));
        
        record.setStatus(DLQStatus.DISCARDED);
        record.setResolvedAt(Instant.now());
        dlqRepository.save(record);
        
        return ResponseEntity.ok().build();
    }
}
```

---

## <a name="scenario-14-dlq"></a>**Scenario 7: Implementing Dead Letter Queue Strategy**

### **📋 Context**
You need a robust error handling strategy for multiple consumer groups with different retry policies.

### **✅ Comprehensive DLQ Implementation**

```java
@Configuration
public class KafkaErrorHandlingConfig {
    
    @Bean
    public KafkaTemplate<String, Object> dlqKafkaTemplate() {
        return new KafkaTemplate<>(dlqProducerFactory());
    }
    
    @Bean
    public ProducerFactory<String, Object> dlqProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        return new DefaultKafkaProducerFactory<>(config);
    }
    
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> dlqTemplate) {
        // Custom recoverer with enhanced error information
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            dlqTemplate,
            (record, ex) -> {
                // Determine DLQ topic based on error type
                String dlqTopic = determineDLQTopic(record, ex);
                return new TopicPartition(dlqTopic, record.partition());
            }
        ) {
            @Override
            protected Headers createHeaders(ConsumerRecord<?, ?> record, 
                                           Exception exception, 
                                           TopicPartition deadLetterTopic) {
                Headers headers = super.createHeaders(record, exception, deadLetterTopic);
                
                // Add custom headers
                headers.add("dlq-timestamp", 
                    String.valueOf(System.currentTimeMillis()).getBytes());
                headers.add("dlq-reason", 
                    classifyError(exception).getBytes());
                headers.add("original-timestamp", 
                    String.valueOf(record.timestamp()).getBytes());
                
                return headers;
            }
        };
        
        // Retry configuration based on error type
        BackOff backOff = new ExponentialBackOffWithMaxRetries(3);
        
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);
        
        // Don't retry certain exceptions
        errorHandler.addNotRetryableExceptions(
            ValidationException.class,
            DeserializationException.class,
            MessageConversionException.class
        );
        
        return errorHandler;
    }
    
    private String determineDLQTopic(ConsumerRecord<?, ?> record, Exception ex) {
        if (ex instanceof ValidationException) {
            return record.topic() + "-dlq-validation";
        } else if (ex instanceof DeserializationException) {
            return record.topic() + "-dlq-deserialization";
        } else {
            return record.topic() + "-dlq";
        }
    }
    
    private String classifyError(Exception ex) {
        if (ex instanceof ValidationException) return "VALIDATION_ERROR";
        if (ex instanceof TimeoutException) return "TIMEOUT";
        if (ex instanceof DeserializationException) return "DESERIALIZATION_ERROR";
        if (ex instanceof SQLException) return "DATABASE_ERROR";
        return "UNKNOWN_ERROR";
    }
}

// Custom BackOff with max retries
class ExponentialBackOffWithMaxRetries implements BackOff {
    
    private final int maxRetries;
    private final ExponentialBackOff backOff;
    
    public ExponentialBackOffWithMaxRetries(int maxRetries) {
        this.maxRetries = maxRetries;
        this.backOff = new ExponentialBackOff(1000, 2.0);
        this.backOff.setMaxInterval(30000); // Max 30 seconds
    }
    
    @Override
    public BackOffExecution start() {
        return new BackOffExecution() {
            private int attempts = 0;
            private final BackOffExecution execution = backOff.start();
            
            @Override
            public long nextBackOff() {
                if (attempts++ >= maxRetries) {
                    return BackOffExecution.STOP;
                }
                return execution.nextBackOff();
            }
        };
    }
}

// DLQ Monitoring Service
@Service
@Slf4j
public class DLQMonitoringService {
    
    @Autowired
    private AdminClient adminClient;
    
    @Autowired
    private AlertService alertService;
    
    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void monitorDLQTopics() {
        try {
            ListTopicsResult topics = adminClient.listTopics();
            Set<String> topicNames = topics.names().get();
            
            // Find all DLQ topics
            Set<String> dlqTopics = topicNames.stream()
                .filter(name -> name.endsWith("-dlq"))
                .collect(Collectors.toSet());
            
            for (String dlqTopic : dlqTopics) {
                long messageCount = getMessageCount(dlqTopic);
                
                if (messageCount > 100) {
                    String originalTopic = dlqTopic.replace("-dlq", "");
                    alertService.sendAlert(
                        "High DLQ Message Count",
                        String.format("Topic %s has %d messages in DLQ", 
                            originalTopic, messageCount)
                    );
                }
            }
            
        } catch (Exception e) {
            log.error("Error monitoring DLQ topics", e);
        }
    }
    
    private long getMessageCount(String topic) {
        // Implementation to count messages in topic
        return 0; // Placeholder
    }
}
```

---

## <a name="scenario-13-oom"></a>**Scenario 8: Out of Memory in Consumer Application**

### **📋 Context**
Consumer pods are crashing with OOMKilled status. Memory usage grows steadily until the pod is killed.

### **🔍 Problem Symptoms**
```bash
kubectl get pods -n consumer-service
# NAME                      READY   STATUS      RESTARTS
# consumer-app-xyz          0/1     OOMKilled   5

kubectl describe pod consumer-app-xyz -n consumer-service
# Last State: Terminated
#   Reason: OOMKilled
#   Exit Code: 137
```

### **🔎 Investigation**

```bash
# Check memory limits
kubectl describe pod consumer-app-xyz -n consumer-service | grep -A 5 "Limits:"

# Get heap dump (if pod is still running)
kubectl exec consumer-app-xyz -n consumer-service -- \
  jmap -dump:format=b,file=/tmp/heap.hprof 1

# Copy heap dump locally
kubectl cp consumer-service/consumer-app-xyz:/tmp/heap.hprof ./heap.hprof

# Analyze with Eclipse MAT or VisualVM

# Check logs before crash
kubectl logs consumer-app-xyz -n consumer-service --previous | tail -200
```

### **🎯 Root Causes**

1. **Too many messages polled at once**
2. **Large messages held in memory**
3. **Memory leak in business logic**
4. **Insufficient heap size**

### **✅ Solution**

```yaml
# 1. Increase memory limits
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: consumer-app
spec:
  template:
    spec:
      containers:
      - name: consumer
        image: consumer-app:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"  # Increased from 512Mi
            cpu: "2000m"
        env:
        - name: JAVA_OPTS
          value: "-Xms512m -Xmx1536m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof"
```

```java
// 2. Optimize consumer configuration
@Configuration
public class KafkaConsumerConfig {
    
    @Bean
    public Map<String, Object> consumerConfigs() {
        Map<String, Object> props = new HashMap<>();
        
        // Reduce max.poll.records to avoid loading too many messages
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50); // Was 500
        
        // Reduce fetch size
        props.put(ConsumerConfig.FETCH_MAX_BYTES_CONFIG, 5242880); // 5MB (was 50MB)
        props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, 1048576); // 1MB
        
        // Increase poll interval to allow more processing time
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 minutes
        
        return props;
    }
}

// 3. Process messages in batches with proper memory management
@Service
@Slf4j
public class MemoryEfficientConsumer {
    
    @Autowired
    private OrderService orderService;
    
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processor",
        containerFactory = "batchListenerContainerFactory"
    )
    public void processOrderBatch(List<OrderEvent> events) {
        log.info("Processing batch of {} orders", events.size());
        
        // Process in smaller sub-batches
        int subBatchSize = 10;
        for (int i = 0; i < events.size(); i += subBatchSize) {
            int end = Math.min(i + subBatchSize, events.size());
            List<OrderEvent> subBatch = events.subList(i, end);
            
            try {
                // Process sub-batch
                orderService.processBatch(subBatch);
                
                // Clear references explicitly
                subBatch.clear();
                
                // Suggest GC if memory is getting high
                if (isMemoryHigh()) {
                    System.gc();
                }
                
            } catch (Exception e) {
                log.error("Error processing sub-batch", e);
            }
        }
    }
    
    private boolean isMemoryHigh() {
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        long maxMemory = runtime.maxMemory();
        double usagePercent = (double) usedMemory / maxMemory;
        return usagePercent > 0.8; // 80% threshold
    }
}

// 4. Use streaming processing for large messages
@Service
@Slf4j
public class StreamingMessageProcessor {
    
    @KafkaListener(topics = "large-file-events")
    public void processLargeFile(LargeFileEvent event) {
        // Don't load entire file into memory
        try (InputStream inputStream = s3Client.getObject(event.getS3Key());
             BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream))) {
            
            String line;
            List<String> batch = new ArrayList<>(1000);
            
            while ((line = reader.readLine()) != null) {
                batch.add(line);
                
                if (batch.size() >= 1000) {
                    processBatch(batch);
                    batch.clear(); // Clear to release memory
                }
            }
            
            // Process remaining
            if (!batch.isEmpty()) {
                processBatch(batch);
            }
            
        } catch (IOException e) {
            log.error("Error processing large file", e);
            throw new RuntimeException(e);
        }
    }
    
    private void processBatch(List<String> batch) {
        // Process batch
    }
}

// 5. Implement memory monitoring
@Component
@Slf4j
public class MemoryMonitor {
    
    @Scheduled(fixedDelay = 60000) // Every minute
    public void monitorMemory() {
        Runtime runtime = Runtime.getRuntime();
        
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;
        long maxMemory = runtime.maxMemory();
        
        double usagePercent = (double) usedMemory / maxMemory * 100;
        
        log.info("Memory usage: {}% ({} MB / {} MB)", 
            String.format("%.2f", usagePercent),
            usedMemory / 1024 / 1024,
            maxMemory / 1024 / 1024);
        
        if (usagePercent > 90) {
            log.error("⚠️ CRITICAL: Memory usage above 90%!");
            // Trigger alert
            alertService.sendAlert("High Memory Usage", 
                "Consumer memory usage: " + String.format("%.2f", usagePercent) + "%");
        }
    }
    
    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void logGCStats() {
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        
        for (GarbageCollectorMXBean gcBean : gcBeans) {
            log.info("GC Stats: {} - Count: {}, Time: {}ms",
                gcBean.getName(),
                gcBean.getCollectionCount(),
                gcBean.getCollectionTime());
        }
    }
}

// 6. Use WeakReferences for caching
@Service
public class CachingService {
    
    // Use WeakReference cache to allow GC to reclaim memory
    private final Map<String, WeakReference<CachedData>> cache = 
        new ConcurrentHashMap<>();
    
    public CachedData get(String key) {
        WeakReference<CachedData> ref = cache.get(key);
        
        if (ref != null) {
            CachedData data = ref.get();
            if (data != null) {
                return data; // Cache hit
            } else {
                cache.remove(key); // Was garbage collected
            }
        }
        
        // Cache miss - load and cache
        CachedData data = loadData(key);
        cache.put(key, new WeakReference<>(data));
        return data;
    }
    
    private CachedData loadData(String key) {
        // Load data
        return new CachedData();
    }
}
```

### **📊 Monitoring**

```yaml
# Prometheus metrics
- alert: ConsumerHighMemoryUsage
  expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Consumer heap usage above 85%"

- alert: ConsumerOOMKilled
  expr: kube_pod_container_status_terminated_reason{reason="OOMKilled"} > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Consumer pod was OOMKilled"
```

---

## **Advanced Scenarios**

## <a name="scenario-16-large-messages"></a>**Scenario 9: Handling Large Messages**

### **📋 Context**
You need to send messages larger than Kafka's default 1MB limit. Messages contain large JSON payloads or file attachments.

### **🔍 Problem Symptoms**
```
org.apache.kafka.common.errors.RecordTooLargeException: 
  The message is 5242880 bytes when serialized which is larger than 1048576
```

### **✅ Solution Strategies**

**Strategy 1: Increase Kafka Limits (Not Recommended)**

```bash
# Increase broker message size (requires careful consideration)
kubectl exec -n kafka kafka-0 -- kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type brokers \
  --entity-default \
  --alter \
  --add-config message.max.bytes=10485760  # 10MB

# Also need to update consumer
# max.partition.fetch.bytes=10485760
```

**Strategy 2: Claim Check Pattern (Recommended)**

```java
// Store large payload externally, send reference via Kafka
@Service
@Slf4j
public class ClaimCheckProducer {
    
    @Autowired
    private S3Client s3Client;
    
    @Autowired
    private KafkaTemplate<String, MessageReference> kafkaTemplate;
    
    public void sendLargeMessage(String key, LargePayload payload) {
        try {
            // 1. Upload payload to S3
            String s3Key = "kafka-payloads/" + UUID.randomUUID();
            byte[] data = serializePayload(payload);
            
            s3Client.putObject(PutObjectRequest.builder()
                .bucket("large-message-store")
                .key(s3Key)
                .build(),
                RequestBody.fromBytes(data));
            
            log.info("Uploaded large payload to S3: {}", s3Key);
            
            // 2. Send reference to Kafka
            MessageReference reference = new MessageReference();
            reference.setS3Bucket("large-message-store");
            reference.setS3Key(s3Key);
            reference.setPayloadSize(data.length);
            reference.setTimestamp(Instant.now());
            
            kafkaTemplate.send("order-events", key, reference);
            
            log.info("Sent message reference to Kafka: key={}", key);
            
        } catch (Exception e) {
            log.error("Failed to send large message", e);
            throw new RuntimeException(e);
        }
    }
}

@Service
@Slf4j
public class ClaimCheckConsumer {
    
    @Autowired
    private S3Client s3Client;
    
    @KafkaListener(topics = "order-events")
    public void processMessage(MessageReference reference) {
        try {
            // 1. Download from S3
            log.info("Downloading payload from S3: {}", reference.getS3Key());
            
            ResponseBytes<GetObjectResponse> response = s3Client.getObjectAsBytes(
                GetObjectRequest.builder()
                    .bucket(reference.getS3Bucket())
                    .key(reference.getS3Key())
                    .build()
            );
            
            byte[] data = response.asByteArray();
            LargePayload payload = deserializePayload(data);
            
            // 2. Process payload
            processPayload(payload);
            
            // 3. Delete from S3 after successful processing
            s3Client.deleteObject(DeleteObjectRequest.builder()
                .bucket(reference.getS3Bucket())
                .key(reference.getS3Key())
                .build());
            
            log.info("Processed and cleaned up large message: {}", reference.getS3Key());
            
        } catch (Exception e) {
            log.error("Failed to process large message", e);
            throw new RuntimeException(e);
        }
    }
}
```

**Strategy 3: Message Chunking**

```java
// Split large message into chunks
@Service
@Slf4j
public class ChunkingProducer {
    
    @Autowired
    private KafkaTemplate<String, MessageChunk> kafkaTemplate;
    
    private static final int CHUNK_SIZE = 900 * 1024; // 900KB chunks
    
    public void sendLargeMessage(String messageId, byte[] largeData) {
        int totalChunks = (int) Math.ceil((double) largeData.length / CHUNK_SIZE);
        
        log.info("Splitting message {} into {} chunks", messageId, totalChunks);
        
        for (int i = 0; i < totalChunks; i++) {
            int start = i * CHUNK_SIZE;
            int end = Math.min(start + CHUNK_SIZE, largeData.length);
            byte[] chunkData = Arrays.copyOfRange(largeData, start, end);
            
            MessageChunk chunk = new MessageChunk();
            chunk.setMessageId(messageId);
            chunk.setChunkIndex(i);
            chunk.setTotalChunks(totalChunks);
            chunk.setData(chunkData);
            chunk.setChecksum(calculateChecksum(chunkData));
            
            kafkaTemplate.send("chunked-messages", messageId, chunk);
        }
        
        log.info("Sent {} chunks for message {}", totalChunks, messageId);
    }
    
    private String calculateChecksum(byte[] data) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(data);
            return Base64.getEncoder().encodeToString(digest);
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException(e);
        }
    }
}

@Service
@Slf4j
public class ChunkingConsumer {
    
    // Store chunks until we have all of them
    private final Map<String, ChunkAssembler> assemblers = new ConcurrentHashMap<>();
    
    @KafkaListener(topics = "chunked-messages")
    public void processChunk(MessageChunk chunk) {
        String messageId = chunk.getMessageId();
        
        // Get or create assembler for this message
        ChunkAssembler assembler = assemblers.computeIfAbsent(
            messageId,
            id -> new ChunkAssembler(id, chunk.getTotalChunks())
        );
        
        // Add chunk
        assembler.addChunk(chunk);
        
        // Check if complete
        if (assembler.isComplete()) {
            try {
                byte[] completeMessage = assembler.assemble();
                
                // Process complete message
                processCompleteMessage(messageId, completeMessage);
                
                // Clean up
                assemblers.remove(messageId);
                
                log.info("Assembled and processed message: {}", messageId);
                
            } catch (Exception e) {
                log.error("Error assembling message: {}", messageId, e);
                assemblers.remove(messageId);
            }
        }
    }
    
    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void cleanupStaleAssemblers() {
        long cutoff = System.currentTimeMillis() - (30 * 60 * 1000); // 30 minutes
        
        assemblers.entrySet().removeIf(entry -> 
            entry.getValue().getCreatedAt() < cutoff
        );
    }
}

class ChunkAssembler {
    private final String messageId;
    private final int totalChunks;
    private final Map<Integer, MessageChunk> chunks;
    private final long createdAt;
    
    public ChunkAssembler(String messageId, int totalChunks) {
        this.messageId = messageId;
        this.totalChunks = totalChunks;
        this.chunks = new ConcurrentHashMap<>();
        this.createdAt = System.currentTimeMillis();
    }
    
    public void addChunk(MessageChunk chunk) {
        chunks.put(chunk.getChunkIndex(), chunk);
    }
    
    public boolean isComplete() {
        return chunks.size() == totalChunks;
    }
    
    public byte[] assemble() throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        
        for (int i = 0; i < totalChunks; i++) {
            MessageChunk chunk = chunks.get(i);
            if (chunk == null) {
                throw new IllegalStateException("Missing chunk: " + i);
            }
            
            // Verify checksum
            String calculated = calculateChecksum(chunk.getData());
            if (!calculated.equals(chunk.getChecksum())) {
                throw new IllegalStateException("Checksum mismatch for chunk: " + i);
            }
            
            baos.write(chunk.getData());
        }
        
        return baos.toByteArray();
    }
    
    public long getCreatedAt() {
        return createdAt;
    }
}
```

---

## **Quick Reference Guide**

### **Common Issues Checklist**

```bash
# 1. High Lag
✓ Check consumer pod status
✓ Check consumer logs for errors  
✓ Verify database/external service performance
✓ Check consumer configuration (max.poll.records, etc.)
✓ Consider scaling consumers

# 2. Consumer Not Processing
✓ Check if consumer group has active members
✓ Verify partitions are assigned
✓ Check for poison messages
✓ Look for rebalancing loops

# 3. Producer Failures
✓ Check Kafka broker status
✓ Verify network connectivity
✓ Check producer timeout settings
✓ Look at producer metrics

# 4. Duplicates
✓ Enable idempotent producer
✓ Implement idempotency in consumer
✓ Check for consumer rebalancing during processing

# 5. OOM Errors
✓ Check memory limits
✓ Reduce max.poll.records
✓ Implement batching
✓ Use streaming for large messages
```

---

**Created for SREs and Senior Java Developers**  
*Last Updated: November 18, 2025*

Remember: Always test solutions in a non-production environment first!

