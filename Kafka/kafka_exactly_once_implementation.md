# Exactly-Once Semantics — Triển khai thực tế Step-by-Step

> **Context:** Hướng dẫn này dựa trên một use case thực tế: **Payment Processing System** — hệ thống xử lý giao dịch thanh toán, nơi mà duplicate hay data loss đều gây hậu quả nghiêm trọng. Mọi bước được giải thích với lý do "tại sao" chứ không chỉ "làm gì".

---

## Mục lục

1. [Kiến trúc tổng thể & Prerequisites](#1-kiến-trúc-tổng-thể--prerequisites)
2. [Step 1 — Infrastructure Setup](#step-1--infrastructure-setup)
3. [Step 2 — Producer với Idempotent + Transactions](#step-2--producer-với-idempotent--transactions)
4. [Step 3 — Consumer với read_committed](#step-3--consumer-với-read_committed)
5. [Step 4 — Consume-Transform-Produce Pattern](#step-4--consume-transform-produce-pattern)
6. [Step 5 — Error Handling & Recovery](#step-5--error-handling--recovery)
7. [Step 6 — Monitoring & Observability](#step-6--monitoring--observability)
8. [Step 7 — Testing EOS](#step-7--testing-eos)
9. [Step 8 — Production Checklist](#step-8--production-checklist)
10. [Real-world Gotchas](#real-world-gotchas)

---

## 1. Kiến trúc tổng thể & Prerequisites

### Use case

```
payment-requests topic
        │
        ▼
 Payment Processor Service
 (Consume → Validate → Enrich → Produce)
        │
        ├──► payment-processed topic  (thành công)
        └──► payment-failed topic     (thất bại)
```

**Yêu cầu business:**
- Mỗi payment request được xử lý **đúng một lần**
- Nếu service crash, khi restart phải tiếp tục từ chỗ dừng, không reprocess
- Output phải consistent: một request không được vừa xuất hiện ở `payment-processed` vừa `payment-failed`

### Prerequisites — Kafka broker config cần check

```bash
# Kiểm tra broker có hỗ trợ EOS không (cần Kafka 0.11+, khuyến nghị 2.5+)
kafka-broker-api-versions.sh --bootstrap-server localhost:9092 | grep -i transaction

# Broker configs cần verify (trong server.properties hoặc AdminClient):
# transaction.state.log.replication.factor=3       (production: phải >= 3)
# transaction.state.log.min.isr=2                  (production: phải >= 2)
# transaction.max.timeout.ms=900000                (15 phút max)
# transactional.id.expiration.ms=604800000         (7 ngày default)
```

**Tại sao `replication.factor=3` quan trọng?**

`__transaction_state` topic lưu trạng thái của tất cả transactions. Nếu topic này mất data (replication factor thấp + broker crash), toàn bộ transaction state bị corrupt → Service không thể tiếp tục.

---

## Step 1 — Infrastructure Setup

### 1.1 Tạo topics với cấu hình phù hợp

```bash
# Topic cho input - retention cao để có thể replay
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic payment-requests \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \       # 7 ngày
  --config min.insync.replicas=2 \        # Bắt buộc cho EOS
  --config cleanup.policy=delete

# Topic cho output - processed payments
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic payment-processed \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \      # 30 ngày
  --config min.insync.replicas=2

# Topic cho failed payments
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic payment-failed \
  --partitions 12 \
  --replication-factor 3 \
  --config min.insync.replicas=2
```

**Tại sao `min.insync.replicas=2`?**

Với `acks=all` (bắt buộc cho EOS), broker chỉ trả ACK khi tất cả ISR replicas đã ghi. Nếu `min.insync.replicas=1`, chỉ cần leader ghi là đủ — mất đi lợi ích của replication. ISR=2 đảm bảo ít nhất 2 brokers có data trước khi ACK.

### 1.2 Maven/Gradle dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>3.6.1</version>
    </dependency>
    
    <!-- Micrometer cho metrics -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
        <version>1.12.0</version>
    </dependency>
</dependencies>
```

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.apache.kafka:kafka-clients:3.6.1")
    implementation("io.micrometer:micrometer-registry-prometheus:1.12.0")
    testImplementation("org.testcontainers:kafka:1.19.3")
}
```

---

## Step 2 — Producer với Idempotent + Transactions

### 2.1 Configuration Factory

```java
// TransactionalProducerConfig.java
public class TransactionalProducerConfig {
    
    /**
     * Tạo config cho Exactly-Once producer.
     *
     * @param bootstrapServers  Kafka brokers
     * @param transactionalId   PHẢI unique per instance. Dùng hostname + pod name.
     *                          Nếu 2 instances cùng ID → chúng sẽ fence nhau liên tục.
     */
    public static Properties build(String bootstrapServers, String transactionalId) {
        Properties props = new Properties();
        
        // --- Core Connection ---
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        
        // --- EOS Critical Configs ---
        
        // enable.idempotence=true tự động set:
        //   acks=all, retries=MAX_INT, max.in.flight.requests.per.connection=5
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // transactional.id: Định danh "session" của producer.
        // Khi producer restart với cùng ID, broker tăng epoch → fence producer cũ (zombie).
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, transactionalId);
        
        // Timeout cho toàn bộ transaction.
        // Phải < transaction.max.timeout.ms của broker (default 900000ms = 15 phút).
        // Nếu quá thời gian, broker tự abort transaction.
        // Set thấp để detect stuck transactions sớm.
        props.put(ProducerConfig.TRANSACTION_TIMEOUT_CONFIG, 60_000); // 60 giây
        
        // --- Performance Tuning (không ảnh hưởng EOS guarantee) ---
        
        // Batch nhiều records lại trước khi gửi → giảm số RPCs
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65_536); // 64KB
        
        // Chờ tối đa 10ms để fill batch
        // QUAN TRỌNG: Linger.ms ảnh hưởng latency của từng send(),
        // nhưng KHÔNG ảnh hưởng đến khi nào transaction được committed.
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);
        
        // Buffer size cho tất cả unsent records
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67_108_864); // 64MB
        
        // Compression giảm network bandwidth đáng kể
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");
        
        // Request timeout phải < delivery.timeout.ms
        props.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30_000);
        
        // Tổng thời gian từ lúc send() đến khi nhận ACK (bao gồm retries)
        // Phải > request.timeout.ms để có ít nhất 1 lần retry
        props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120_000);
        
        return props;
    }
    
    /**
     * Generate unique transactional ID cho mỗi instance.
     * Dùng combo hostname + container/pod name để đảm bảo uniqueness.
     */
    public static String generateTransactionalId(String servicePrefix) {
        String hostname = getHostname();
        String podName = System.getenv().getOrDefault("POD_NAME", 
                                                       UUID.randomUUID().toString().substring(0, 8));
        return servicePrefix + "-" + hostname + "-" + podName;
        // Ví dụ: "payment-processor-kafka-worker-01-payment-pod-abc123"
    }
    
    private static String getHostname() {
        try {
            return InetAddress.getLocalHost().getHostName();
        } catch (UnknownHostException e) {
            return "unknown-" + System.currentTimeMillis();
        }
    }
}
```

### 2.2 Transactional Producer Wrapper

```java
// TransactionalProducer.java
public class TransactionalProducer implements Closeable {
    
    private static final Logger log = LoggerFactory.getLogger(TransactionalProducer.class);
    
    private final KafkaProducer<String, String> producer;
    private final String transactionalId;
    private volatile boolean initialized = false;
    
    public TransactionalProducer(String bootstrapServers, String servicePrefix) {
        this.transactionalId = TransactionalProducerConfig.generateTransactionalId(servicePrefix);
        Properties props = TransactionalProducerConfig.build(bootstrapServers, transactionalId);
        this.producer = new KafkaProducer<>(props);
        
        log.info("Created transactional producer with ID: {}", transactionalId);
    }
    
    /**
     * PHẢI gọi một lần khi khởi động, trước bất kỳ transaction nào.
     *
     * initTransactions() làm gì:
     * 1. Tìm Transaction Coordinator cho transactional.id này
     * 2. Đăng ký producer với coordinator (nhận PID + Epoch)
     * 3. Nếu có transaction dở dang (crash lần trước), coordinator sẽ abort nó
     *
     * Nếu coordinator không phản hồi trong timeout → TimeoutException
     * Nếu broker version không support → UnsupportedVersionException
     */
    public void init() {
        if (!initialized) {
            producer.initTransactions();
            initialized = true;
            log.info("Transactions initialized for producer: {}", transactionalId);
        }
    }
    
    /**
     * Execute một transaction với automatic commit/abort.
     *
     * Pattern này (functional style) đảm bảo:
     * - beginTransaction() luôn có commitTransaction() hoặc abortTransaction() tương ứng
     * - Không bao giờ bị quên abort khi có exception
     */
    public void executeTransaction(TransactionBlock block) {
        ensureInitialized();
        
        producer.beginTransaction();
        log.debug("Transaction begun for producer: {}", transactionalId);
        
        try {
            block.execute(producer);
            producer.commitTransaction();
            log.debug("Transaction committed for producer: {}", transactionalId);
            
        } catch (ProducerFencedException | InvalidProducerEpochException e) {
            // CRITICAL: Đây là "zombie fencing".
            // Producer khác (instance mới sau restart) đã claim transactional.id này.
            // Instance này đã bị fence → KHÔNG thể tiếp tục.
            // KHÔNG abort (không có quyền) → Đóng producer và shutdown service.
            log.error("Producer fenced! Another instance has taken over transactional.id={}. " +
                      "This instance must shut down.", transactionalId, e);
            producer.close(); // Đóng luôn
            throw new FatalProducerException("Producer fenced, must restart", e);
            
        } catch (AuthorizationException e) {
            // ACL issue → Fatal, không retry
            log.error("Authorization failed for transactional producer: {}", transactionalId, e);
            producer.close();
            throw new FatalProducerException("Authorization failed", e);
            
        } catch (KafkaException e) {
            // Lỗi có thể retry được (network timeout, broker not available, etc.)
            log.warn("Transaction failed, aborting. Will retry next poll cycle.", e);
            try {
                producer.abortTransaction();
            } catch (KafkaException abortEx) {
                log.error("Failed to abort transaction: {}", abortEx.getMessage());
                // abortTransaction() failed → không biết state của transaction
                // An toàn nhất là throw để caller reset consumer offset và retry
            }
            throw new RetryableTransactionException("Transaction aborted, retry", e);
        }
    }
    
    @Override
    public void close() {
        producer.close(Duration.ofSeconds(30));
        log.info("Producer closed: {}", transactionalId);
    }
    
    private void ensureInitialized() {
        if (!initialized) {
            throw new IllegalStateException("Producer not initialized. Call init() first.");
        }
    }
    
    @FunctionalInterface
    public interface TransactionBlock {
        void execute(KafkaProducer<String, String> producer) throws KafkaException;
    }
}
```

---

## Step 3 — Consumer với read_committed

### 3.1 Consumer Configuration

```java
// EosConsumerConfig.java
public class EosConsumerConfig {
    
    public static Properties build(String bootstrapServers, String groupId) {
        Properties props = new Properties();
        
        // --- Core ---
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        
        // --- EOS Critical ---
        
        // PHẢI = false khi dùng transactions.
        // Auto-commit không biết về transaction boundaries → có thể commit offset
        // của messages thuộc transaction chưa committed → inconsistency.
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        
        // read_committed: Chỉ trả về messages của committed transactions
        // (và non-transactional messages).
        // read_uncommitted (default): Trả về tất cả, kể cả messages của transactions
        // chưa committed hoặc đã aborted → Consumer có thể xử lý data bị rollback!
        props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
        
        // --- Offset Reset Policy ---
        // "earliest": Khi không có committed offset (lần đầu chạy) → đọc từ đầu
        // "latest": Bỏ qua messages cũ → Dùng cho services không cần reprocess
        // CHỌN DỰA TRÊN BUSINESS REQUIREMENT, KHÔNG PHẢI PREFERENCE
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        // --- Performance & Reliability ---
        
        // Số records tối đa trả về mỗi poll().
        // Cân nhắc: Batch nhỏ hơn → Transaction nhỏ hơn → Latency thấp hơn nhưng overhead cao hơn
        // Batch lớn hơn → Transaction lớn hơn → Throughput cao hơn nhưng nếu fail phải reprocess nhiều
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        
        // Heartbeat interval phải < session.timeout.ms / 3
        // Kafka dùng heartbeat để biết consumer còn sống
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 3_000);
        
        // Nếu không có heartbeat trong thời gian này → consumer bị coi là dead → rebalance
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30_000);
        
        // Thời gian tối đa giữa hai poll() liên tiếp.
        // Nếu processing một batch mất nhiều hơn thời gian này → consumer bị evict → rebalance!
        // Phải > thời gian xử lý thực tế của max.poll.records messages.
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300_000); // 5 phút
        
        return props;
    }
}
```

**Tại sao `max.poll.interval.ms` quan trọng với EOS?**

Nếu xử lý 500 records mất hơn 5 phút → Kafka evict consumer → Rebalance → Partition assign cho consumer khác → Consumer mới bắt đầu từ last committed offset → Các records bạn đang xử lý sẽ được reprocess. Với EOS, transaction abort thì OK, nhưng duplicate processing gây overhead.

---

## Step 4 — Consume-Transform-Produce Pattern

### 4.1 Data Models

```java
// PaymentRequest.java (input)
public record PaymentRequest(
    String requestId,      // Unique ID - dùng để track duplicate
    String fromAccount,
    String toAccount,
    BigDecimal amount,
    String currency,
    Instant timestamp
) {}

// PaymentResult.java (output)
public record PaymentResult(
    String requestId,      // Giữ nguyên requestId từ input để trace
    String status,         // "PROCESSED" hoặc "FAILED"
    String reason,         // Lý do nếu FAILED
    Instant processedAt
) {}
```

### 4.2 Main Processing Loop

```java
// PaymentProcessor.java
public class PaymentProcessor implements Runnable, Closeable {
    
    private static final Logger log = LoggerFactory.getLogger(PaymentProcessor.class);
    
    private final KafkaConsumer<String, String> consumer;
    private final TransactionalProducer producer;
    private final ObjectMapper objectMapper;
    private volatile boolean running = true;
    
    // Topics
    private static final String INPUT_TOPIC = "payment-requests";
    private static final String PROCESSED_TOPIC = "payment-processed";
    private static final String FAILED_TOPIC = "payment-failed";
    
    public PaymentProcessor(String bootstrapServers) {
        Properties consumerProps = EosConsumerConfig.build(
            bootstrapServers, 
            "payment-processor-group"
        );
        this.consumer = new KafkaConsumer<>(consumerProps);
        this.producer = new TransactionalProducer(bootstrapServers, "payment-processor");
        this.objectMapper = new ObjectMapper();
    }
    
    @Override
    public void run() {
        // Step 4.1: Init transactions TRƯỚC KHI subscribe
        producer.init();
        
        // Step 4.2: Subscribe
        consumer.subscribe(Collections.singletonList(INPUT_TOPIC));
        log.info("Payment processor started, subscribed to: {}", INPUT_TOPIC);
        
        // Step 4.3: Processing loop
        while (running) {
            try {
                processOneBatch();
            } catch (FatalProducerException e) {
                // Zombie fenced → Phải shutdown hoàn toàn
                log.error("Fatal error, shutting down processor", e);
                running = false;
            } catch (RetryableTransactionException e) {
                // Transaction abort → Reset consumer offset → Retry
                log.warn("Transaction aborted, resetting consumer position for retry");
                resetConsumerToLastCommitted();
            } catch (WakeupException e) {
                // Được trigger bởi consumer.wakeup() trong shutdown hook
                if (running) throw e; // Unexpected
                log.info("Consumer wakeup received, shutting down");
            } catch (Exception e) {
                log.error("Unexpected error in processing loop", e);
                // Tuỳ policy: crash and restart hoặc continue
                // Trong payment processing: crash để tránh silent data loss
                throw new RuntimeException("Unexpected error, processor must restart", e);
            }
        }
    }
    
    private void processOneBatch() {
        // --- Poll ---
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500));
        
        if (records.isEmpty()) {
            return; // Nothing to process
        }
        
        log.debug("Polled {} records", records.count());
        
        // --- Pre-process: deserialize và validate TRƯỚC khi begin transaction ---
        // Tại sao làm ngoài transaction?
        // 1. Transaction có timeout → không muốn tốn thời gian vào deserialization
        // 2. Nếu deserialize fail → abort transaction, không phải crash toàn bộ
        List<ProcessingRequest> requests = new ArrayList<>();
        List<ProcessingError> deserErrors = new ArrayList<>();
        
        for (ConsumerRecord<String, String> record : records) {
            try {
                PaymentRequest request = objectMapper.readValue(record.value(), PaymentRequest.class);
                requests.add(new ProcessingRequest(record, request));
            } catch (Exception e) {
                log.error("Failed to deserialize record at offset {}: {}", 
                          record.offset(), e.getMessage());
                // Đưa vào dead letter queue sau, không chặn batch
                deserErrors.add(new ProcessingError(record, e));
            }
        }
        
        // --- Business Logic: cũng làm TRƯỚC transaction ---
        // Mục đích: Transaction chỉ là commit phase, không phải processing phase
        // Tách biệt "compute result" với "commit result"
        List<ProcessedPayment> results = processPayments(requests);
        
        // --- Begin Transaction ---
        // Lúc này chỉ còn việc: ghi results ra Kafka + commit offset
        // Đây là phần ngắn nhất và được bảo vệ bởi transaction
        
        final List<ProcessedPayment> finalResults = results;
        final ConsumerRecords<String, String> finalRecords = records;
        
        producer.executeTransaction(prod -> {
            
            // Ghi tất cả results vào output topics
            for (ProcessedPayment result : finalResults) {
                String outputTopic = result.success() ? PROCESSED_TOPIC : FAILED_TOPIC;
                
                ProducerRecord<String, String> outputRecord = new ProducerRecord<>(
                    outputTopic,
                    result.request().requestId(),        // Key = requestId (cho ordering)
                    objectMapper.writeValueAsString(result.paymentResult())
                );
                
                prod.send(outputRecord, (metadata, exception) -> {
                    if (exception != null) {
                        // Send callback exception → transaction sẽ fail khi commitTransaction()
                        log.error("Failed to send to {}: {}", outputTopic, exception.getMessage());
                    }
                });
            }
            
            // Ghi deserialization errors vào dead letter queue (cũng trong transaction)
            for (ProcessingError error : deserErrors) {
                prod.send(new ProducerRecord<>(
                    "payment-dead-letter",
                    error.record().key(),
                    error.record().value()  // Raw value để debug sau
                ));
            }
            
            // ATOMIC OFFSET COMMIT: Đây là trái tim của EOS
            // sendOffsetsToTransaction() ghi offset commit vào __consumer_offsets
            // TRONG transaction này. Chỉ visible khi transaction commit.
            // → Nếu transaction abort: offset không được commit → Consumer sẽ reprocess batch này
            // → Nếu transaction commit: offset được commit → Consumer không reprocess
            Map<TopicPartition, OffsetAndMetadata> offsets = buildOffsetMap(finalRecords);
            
            prod.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
            // groupMetadata() bao gồm generation ID để detect rebalance trong quá trình này
        });
        
        log.info("Batch of {} records processed and committed", records.count());
    }
    
    /**
     * Business logic: validate và process payments.
     * Chạy NGOÀI transaction → có thể tốn thời gian, gọi external services.
     */
    private List<ProcessedPayment> processPayments(List<ProcessingRequest> requests) {
        return requests.stream()
            .map(req -> {
                try {
                    PaymentResult result = validateAndProcess(req.payment());
                    return new ProcessedPayment(req.payment(), result, true);
                } catch (InsufficientFundsException e) {
                    PaymentResult failed = new PaymentResult(
                        req.payment().requestId(), "FAILED", 
                        "Insufficient funds", Instant.now()
                    );
                    return new ProcessedPayment(req.payment(), failed, false);
                } catch (Exception e) {
                    // Lỗi không expect → đưa vào failed topic với reason
                    PaymentResult failed = new PaymentResult(
                        req.payment().requestId(), "FAILED",
                        "Processing error: " + e.getMessage(), Instant.now()
                    );
                    return new ProcessedPayment(req.payment(), failed, false);
                }
            })
            .collect(Collectors.toList());
    }
    
    private PaymentResult validateAndProcess(PaymentRequest request) {
        // Validate amount
        if (request.amount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidPaymentException("Amount must be positive");
        }
        
        // Giả lập xử lý
        // Trong thực tế: gọi account service, fraud detection, etc.
        return new PaymentResult(
            request.requestId(), "PROCESSED", null, Instant.now()
        );
    }
    
    /**
     * Build offset map: partition → offset+1 (offset+1 = next offset to read)
     */
    private Map<TopicPartition, OffsetAndMetadata> buildOffsetMap(
            ConsumerRecords<String, String> records) {
        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        
        for (TopicPartition partition : records.partitions()) {
            List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
            long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
            
            offsets.put(partition, new OffsetAndMetadata(
                lastOffset + 1,  // Commit offset+1 = "tôi đã xử lý đến offset này"
                "processed at " + Instant.now()  // Optional metadata, hữu ích cho debugging
            ));
        }
        
        return offsets;
    }
    
    /**
     * Reset consumer về last committed offset.
     * Gọi khi transaction abort để consumer reprocess batch vừa fail.
     */
    private void resetConsumerToLastCommitted() {
        // consumer.seek() về committed position
        for (TopicPartition partition : consumer.assignment()) {
            OffsetAndMetadata committed = consumer.committed(Set.of(partition))
                                                  .get(partition);
            if (committed != null) {
                consumer.seek(partition, committed.offset());
            } else {
                // Chưa có committed offset → về đầu
                consumer.seekToBeginning(Set.of(partition));
            }
        }
        log.info("Consumer positions reset to last committed offsets");
    }
    
    @Override
    public void close() {
        running = false;
        consumer.wakeup(); // Interrupt poll() nếu đang block
        // Actual cleanup trong run() thread
    }
    
    // --- Inner classes ---
    record ProcessingRequest(ConsumerRecord<String, String> record, PaymentRequest payment) {}
    record ProcessedPayment(PaymentRequest request, PaymentResult paymentResult, boolean success) {}
    record ProcessingError(ConsumerRecord<String, String> record, Exception error) {}
}
```

---

## Step 5 — Error Handling & Recovery

### 5.1 Exception Hierarchy

```java
// Base exception
public class TransactionException extends RuntimeException {
    public TransactionException(String message, Throwable cause) {
        super(message, cause);
    }
}

// FATAL: Không thể recover mà không restart service
// Triggers: ProducerFencedException, InvalidProducerEpochException, AuthorizationException
public class FatalProducerException extends TransactionException {
    public FatalProducerException(String message, Throwable cause) {
        super(message, cause);
    }
}

// RETRYABLE: Transaction aborted, có thể retry bằng cách reprocess batch
// Triggers: KafkaException thông thường (timeout, broker unavailable)
public class RetryableTransactionException extends TransactionException {
    public RetryableTransactionException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### 5.2 Retry Strategy với Exponential Backoff

```java
// RetryHandler.java
public class RetryHandler {
    
    private static final Logger log = LoggerFactory.getLogger(RetryHandler.class);
    
    private int consecutiveFailures = 0;
    private static final int MAX_CONSECUTIVE_FAILURES = 5;
    private static final long BASE_BACKOFF_MS = 1_000;   // 1 giây
    private static final long MAX_BACKOFF_MS = 60_000;   // 1 phút
    
    /**
     * Xử lý RetryableTransactionException với backoff.
     * Nếu quá nhiều lần fail liên tiếp → escalate thành Fatal.
     */
    public void handleRetryableFailure(RetryableTransactionException e) 
            throws InterruptedException {
        consecutiveFailures++;
        
        if (consecutiveFailures >= MAX_CONSECUTIVE_FAILURES) {
            throw new FatalProducerException(
                "Too many consecutive failures (" + consecutiveFailures + "), giving up", e
            );
        }
        
        long backoff = Math.min(
            BASE_BACKOFF_MS * (1L << consecutiveFailures), // Exponential: 2s, 4s, 8s, 16s...
            MAX_BACKOFF_MS
        );
        
        log.warn("Transaction failed ({}/{} consecutive). Backing off {}ms before retry.", 
                 consecutiveFailures, MAX_CONSECUTIVE_FAILURES, backoff);
        
        Thread.sleep(backoff);
    }
    
    public void recordSuccess() {
        if (consecutiveFailures > 0) {
            log.info("Recovered after {} failures", consecutiveFailures);
        }
        consecutiveFailures = 0;
    }
    
    public int getConsecutiveFailures() {
        return consecutiveFailures;
    }
}
```

### 5.3 Dead Letter Queue (DLQ) Pattern

```java
// DeadLetterQueueHandler.java
public class DeadLetterQueueHandler {
    
    private static final String DLQ_TOPIC = "payment-dead-letter";
    
    /**
     * Ghi message không thể xử lý vào DLQ.
     * QUAN TRỌNG: DLQ write phải nằm trong transaction nếu dùng EOS!
     * Nếu không, có thể: transaction abort nhưng DLQ đã ghi → message xuất hiện
     * ở DLQ dù chưa được thực sự "failed".
     */
    public void sendToDlq(
            KafkaProducer<String, String> producerInTransaction,
            ConsumerRecord<String, String> failedRecord,
            Exception reason) {
        
        // Headers để track lý do fail
        Headers headers = new RecordHeaders();
        headers.add("dlq-reason", reason.getMessage().getBytes(StandardCharsets.UTF_8));
        headers.add("dlq-original-topic", failedRecord.topic().getBytes(StandardCharsets.UTF_8));
        headers.add("dlq-original-partition", 
                    String.valueOf(failedRecord.partition()).getBytes(StandardCharsets.UTF_8));
        headers.add("dlq-original-offset", 
                    String.valueOf(failedRecord.offset()).getBytes(StandardCharsets.UTF_8));
        headers.add("dlq-timestamp", Instant.now().toString().getBytes(StandardCharsets.UTF_8));
        
        ProducerRecord<String, String> dlqRecord = new ProducerRecord<>(
            DLQ_TOPIC,
            null,                   // Partition = null, let Kafka decide
            failedRecord.key(),
            failedRecord.value(),   // Giữ nguyên raw value
            headers
        );
        
        // Gửi trong transaction hiện tại (producerInTransaction đang trong beginTransaction)
        producerInTransaction.send(dlqRecord);
    }
}
```

---

## Step 6 — Monitoring & Observability

### 6.1 Metrics cần monitor cho EOS

```java
// EosMetrics.java
public class EosMetrics {
    
    private final MeterRegistry registry;
    
    // Counters
    private final Counter transactionsCommitted;
    private final Counter transactionsAborted;
    private final Counter producerFenced;
    private final Counter messagesProcessed;
    private final Counter messagesSentToDlq;
    
    // Gauges & Timers
    private final Timer transactionDuration;
    private final Timer batchProcessingDuration;
    
    public EosMetrics(MeterRegistry registry, String serviceInstance) {
        this.registry = registry;
        
        Tags tags = Tags.of("instance", serviceInstance);
        
        transactionsCommitted = Counter.builder("kafka.transactions.committed")
            .description("Number of successfully committed transactions")
            .tags(tags)
            .register(registry);
            
        transactionsAborted = Counter.builder("kafka.transactions.aborted")
            .description("Number of aborted transactions")
            .tags(tags)
            .register(registry);
            
        producerFenced = Counter.builder("kafka.producer.fenced")
            .description("Number of times producer was fenced (zombie detection)")
            .tags(tags)
            .register(registry);
            
        messagesProcessed = Counter.builder("kafka.messages.processed")
            .tags(tags)
            .register(registry);
            
        messagesSentToDlq = Counter.builder("kafka.messages.dlq")
            .description("Messages sent to Dead Letter Queue")
            .tags(tags)
            .register(registry);
            
        transactionDuration = Timer.builder("kafka.transaction.duration")
            .description("Time from beginTransaction to commitTransaction")
            .tags(tags)
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
            
        batchProcessingDuration = Timer.builder("kafka.batch.processing.duration")
            .description("Total time to process one batch (including business logic)")
            .tags(tags)
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    // Getters...
    public Counter getTransactionsCommitted() { return transactionsCommitted; }
    public Counter getTransactionsAborted() { return transactionsAborted; }
    public Counter getProducerFenced() { return producerFenced; }
    public Timer getTransactionDuration() { return transactionDuration; }
    public Timer getBatchProcessingDuration() { return batchProcessingDuration; }
}
```

### 6.2 Kafka metrics cần alert

```yaml
# prometheus-rules.yaml
groups:
  - name: kafka_eos_alerts
    rules:
    
      # Alert khi transaction abort rate cao bất thường
      - alert: KafkaHighTransactionAbortRate
        expr: |
          rate(kafka_transactions_aborted_total[5m]) / 
          rate(kafka_transactions_committed_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka transaction abort rate (>5%)"
          description: "Transaction abort rate is {{ $value | humanizePercentage }}"
      
      # Alert khi consumer lag tăng liên tục (processing không kịp)
      - alert: KafkaConsumerLagGrowing
        expr: |
          increase(kafka_consumer_group_lag[10m]) > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag is growing"
      
      # Alert khi producer bị fence (có zombie instance)
      - alert: KafkaProducerFenced
        expr: kafka_producer_fenced_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka producer was fenced - zombie instance detected"
      
      # Alert khi DLQ có messages (cần human investigation)
      - alert: KafkaDlqHasMessages
        expr: kafka_messages_dlq_total > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Messages in Dead Letter Queue require attention"
```

### 6.3 JMX Metrics từ Kafka client cần track

```java
// Metrics quan trọng để monitor EOS health
// Lấy qua KafkaProducer.metrics() hoặc JMX

/*
Producer metrics:
  record-error-rate          → Tỉ lệ records bị lỗi (should be ~0 với EOS)
  record-retry-rate          → Tỉ lệ records được retry (cao = network vấn đề)
  batch-size-avg             → Batch size trung bình (thấp = không tận dụng batching)
  compression-rate-avg       → Tỉ lệ compression
  
  // EOS-specific:
  txn-init-time-ns-total     → Tổng thời gian init transactions
  txn-begin-time-ns-total    → Tổng thời gian beginTransaction()
  txn-send-offsets-time-ns   → Tổng thời gian sendOffsetsToTransaction()  
  txn-commit-time-ns-total   → Tổng thời gian commitTransaction()
  txn-abort-time-ns-total    → Tổng thời gian abortTransaction()

Consumer metrics:
  records-lag-max            → Consumer lag tối đa (quan trọng nhất)
  records-consumed-rate      → Records consumed/giây
  fetch-latency-avg          → Latency của fetch requests
*/
```

---

## Step 7 — Testing EOS

### 7.1 Integration Test với Testcontainers

```java
// PaymentProcessorEosTest.java
@Testcontainers
class PaymentProcessorEosTest {
    
    @Container
    static final KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0")
    ).withKraft(); // KRaft mode (không cần Zookeeper)
    
    private String bootstrapServers;
    private AdminClient adminClient;
    
    @BeforeEach
    void setup() throws Exception {
        bootstrapServers = kafka.getBootstrapServers();
        adminClient = AdminClient.create(Map.of(
            AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers
        ));
        
        createTopics("payment-requests", "payment-processed", "payment-failed",
                     "payment-dead-letter");
    }
    
    @Test
    @DisplayName("Verify exactly-once: producer crash mid-transaction leaves no partial state")
    void testProducerCrashMidTransaction() throws Exception {
        // Arrange: Gửi 5 payment requests
        List<PaymentRequest> requests = generatePaymentRequests(5);
        publishPaymentRequests(requests);
        
        // Act: Start processor, crash nó sau khi ghi 3 records nhưng trước khi commit
        PaymentProcessor processor = new CrashMidTransactionProcessor(bootstrapServers);
        processor.run(); // Sẽ crash sau 3 records
        
        // Start lại processor
        PaymentProcessor restartedProcessor = new PaymentProcessor(bootstrapServers);
        Thread processorThread = new Thread(restartedProcessor);
        processorThread.start();
        
        // Wait for processing
        Thread.sleep(5_000);
        restartedProcessor.close();
        processorThread.join(10_000);
        
        // Assert: Mỗi payment chỉ có đúng 1 kết quả
        Map<String, List<PaymentResult>> resultsByRequestId = 
            consumeAllResults("payment-processed", 5);
        
        for (PaymentRequest request : requests) {
            List<PaymentResult> results = resultsByRequestId.get(request.requestId());
            assertThat(results)
                .as("Request %s should have exactly one result", request.requestId())
                .hasSize(1);
        }
    }
    
    @Test
    @DisplayName("Verify exactly-once: duplicate input messages produce single output")
    void testDuplicateInputHandling() throws Exception {
        // Simulate: broker retry gây ra duplicate message trong input topic
        PaymentRequest request = generatePaymentRequests(1).get(0);
        publishPaymentRequests(List.of(request, request)); // Publish 2 lần
        
        // Process
        PaymentProcessor processor = new PaymentProcessor(bootstrapServers);
        Thread processorThread = new Thread(processor);
        processorThread.start();
        Thread.sleep(5_000);
        processor.close();
        processorThread.join(10_000);
        
        // Với EOS idempotent producer: broker dedup → chỉ 1 record trong output
        // Với business-level dedup: consumer dedup → chỉ 1 record được xử lý
        List<PaymentResult> results = consumeAllResults("payment-processed", 1)
            .get(request.requestId());
        
        assertThat(results).hasSize(1);
    }
    
    @Test
    @DisplayName("Verify read_committed: consumer should not see aborted transaction data")
    void testReadCommittedIsolation() throws Exception {
        // Arrange: Tạo một transaction rồi abort nó
        Properties producerProps = TransactionalProducerConfig.build(
            bootstrapServers, "test-producer-1"
        );
        KafkaProducer<String, String> testProducer = new KafkaProducer<>(producerProps);
        testProducer.initTransactions();
        
        testProducer.beginTransaction();
        testProducer.send(new ProducerRecord<>("payment-processed", "key1", "aborted-value"));
        testProducer.abortTransaction(); // Abort!
        
        // Ghi một record hợp lệ
        testProducer.beginTransaction();
        testProducer.send(new ProducerRecord<>("payment-processed", "key2", "committed-value"));
        testProducer.commitTransaction();
        testProducer.close();
        
        // Assert: Consumer với read_committed chỉ thấy committed record
        Properties consumerProps = EosConsumerConfig.build(bootstrapServers, "test-group");
        KafkaConsumer<String, String> testConsumer = new KafkaConsumer<>(consumerProps);
        testConsumer.subscribe(List.of("payment-processed"));
        
        ConsumerRecords<String, String> polled = testConsumer.poll(Duration.ofSeconds(5));
        
        List<String> values = StreamSupport.stream(polled.spliterator(), false)
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());
        
        assertThat(values)
            .containsExactly("committed-value")
            .doesNotContain("aborted-value");
        
        testConsumer.close();
    }
    
    // Helper methods...
    private void createTopics(String... topicNames) throws Exception {
        List<NewTopic> topics = Arrays.stream(topicNames)
            .map(name -> new NewTopic(name, 3, (short) 1))
            .collect(Collectors.toList());
        adminClient.createTopics(topics).all().get(10, TimeUnit.SECONDS);
    }
}
```

### 7.2 Chaos Testing

```bash
# Script để test EOS dưới điều kiện chaos
#!/bin/bash
echo "=== EOS Chaos Test ==="

# 1. Start processor
java -jar payment-processor.jar &
PROCESSOR_PID=$!

# 2. Send 1000 payments
./send-payments.sh 1000

# 3. Kill processor ngẫu nhiên (simulate crash)
sleep 5
kill -9 $PROCESSOR_PID
echo "Processor killed mid-processing"

# 4. Restart processor
java -jar payment-processor.jar &
PROCESSOR_PID=$!

# 5. Wait for completion
sleep 30

# 6. Verify: đếm số unique payment results
RESULT_COUNT=$(./count-unique-results.sh payment-processed)
INPUT_COUNT=$(./count-messages.sh payment-requests)

echo "Input count: $INPUT_COUNT"
echo "Unique results: $RESULT_COUNT"

if [ "$RESULT_COUNT" -eq "$INPUT_COUNT" ]; then
    echo "✅ PASS: Exactly-Once verified"
else
    echo "❌ FAIL: Expected $INPUT_COUNT, got $RESULT_COUNT"
fi
```

---

## Step 8 — Production Checklist

### 8.1 Broker configuration

```properties
# server.properties - Settings quan trọng cho EOS production

# Transaction state topic replication (CRITICAL)
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

# Thời gian giữ transaction state (cleanup sau khi complete)
# Default 7 ngày. Không nên giảm quá thấp.
transactional.id.expiration.ms=604800000

# Timeout tối đa cho một transaction (broker-side)
# Producer's transaction.timeout.ms phải <= giá trị này
transaction.max.timeout.ms=900000

# __consumer_offsets topic replication
offsets.topic.replication.factor=3
offsets.topic.num.partitions=50  # Scale dựa trên số consumer groups

# Min ISR cho tất cả topics (override per-topic nếu cần)
min.insync.replicas=2

# Để EOS hoạt động đúng, cần bounded ISR lag
replica.lag.time.max.ms=30000
```

### 8.2 Kubernetes Deployment

```yaml
# payment-processor-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-processor
spec:
  # QUAN TRỌNG: replicas quyết định số transactional.id cần quản lý
  # Mỗi pod PHẢI có transactional.id duy nhất
  replicas: 3
  
  # RollingUpdate để không có 2 pods cùng transactional.id đang active
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # Tối đa 1 pod mới trước khi terminate pod cũ
      maxUnavailable: 0 # Không terminate pod cũ trước khi pod mới ready
  
  template:
    spec:
      containers:
      - name: payment-processor
        image: payment-processor:latest
        
        env:
        # Inject pod name để tạo unique transactional.id
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
              
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka-service:9092"
          
        # Đặt resource limits để tránh OOM → crash → zombie issue
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        
        # Liveness probe: nếu fail → pod restart → transactional.id epoch tăng
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        # Readiness probe: chỉ nhận traffic khi đã init xong
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        
        # Graceful shutdown: đủ thời gian để complete transaction hiện tại
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        
      terminationGracePeriodSeconds: 60
```

### 8.3 Pre-production Checklist

```
Infrastructure:
  ✅ Kafka version >= 2.5 (để dùng EXACTLY_ONCE_V2 nếu dùng Streams)
  ✅ transaction.state.log.replication.factor = 3
  ✅ transaction.state.log.min.isr = 2  
  ✅ offsets.topic.replication.factor = 3
  ✅ min.insync.replicas = 2 cho tất cả critical topics
  ✅ Topics có cleanup.policy=delete (không phải compact) cho input topics

Producer Config:
  ✅ enable.idempotence = true
  ✅ transactional.id = unique per instance (dùng pod name/hostname)
  ✅ acks = all (tự động set bởi idempotence)
  ✅ transaction.timeout.ms < broker's transaction.max.timeout.ms
  ✅ initTransactions() được gọi khi startup

Consumer Config:  
  ✅ isolation.level = read_committed
  ✅ enable.auto.commit = false
  ✅ max.poll.interval.ms > thời gian xử lý 1 batch trong worst case
  ✅ Dùng sendOffsetsToTransaction() thay vì commitSync()

Code:
  ✅ ProducerFencedException → Shutdown ngay, không retry
  ✅ KafkaException → abortTransaction() → Reset consumer → Retry
  ✅ Transaction được giữ ngắn (chỉ ghi, không có business logic dài trong transaction)
  ✅ Dead Letter Queue cho messages không thể xử lý
  ✅ DLQ write nằm trong transaction

Monitoring:
  ✅ Alert cho transaction abort rate > threshold
  ✅ Alert cho consumer lag growing
  ✅ Alert cho producer fenced
  ✅ Alert cho DLQ có messages
  ✅ Dashboard: transactions/sec, abort rate, consumer lag, processing latency

Testing:
  ✅ Integration test: producer crash mid-transaction
  ✅ Integration test: duplicate input messages → single output
  ✅ Integration test: read_committed isolation
  ✅ Chaos test: kill broker mid-transaction
  ✅ Load test: verify EOS dưới high throughput
```

---

## Real-world Gotchas

### Gotcha 1: `transactional.id` và Pod Restart

**Vấn đề:** Kubernetes kill pod và start pod mới cùng tên. Pod mới dùng cùng `transactional.id`. Nếu pod cũ vẫn còn đang chạy (slow shutdown) → fence lẫn nhau.

**Giải pháp:**
```java
// Thêm timestamp vào transactional.id để mỗi "life" của pod là unique
String transactionalId = servicePrefix + "-" + podName + "-" + 
                         System.currentTimeMillis() / 10000; // Mới mỗi ~3 giờ
// Trade-off: transactionalId cũ expire sau transactional.id.expiration.ms
// Đảm bảo expiration đủ cao để cleanup xảy ra
```

Hoặc dùng Kafka Streams với `application.id` + `client.id` — Streams tự handle:
```java
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "payment-processor");
// Streams tự generate transactional.id = application.id + "-" + task.id
```

### Gotcha 2: `max.poll.interval.ms` vs Transaction Timeout

**Vấn đề:** Xử lý batch mất 4 phút. `max.poll.interval.ms=300000` (5 phút). OK! Nhưng `transaction.timeout.ms=60000` (1 phút). → Transaction timeout trước khi xử lý xong.

**Giải pháp:** Đảm bảo:
```
transaction.timeout.ms > (thời gian xử lý business logic + thời gian ghi Kafka)

NHƯNG:
max.poll.interval.ms > transaction.timeout.ms
```

**Hoặc tốt hơn:** Tách business logic ra khỏi transaction window (như đã trình bày ở Step 4).

### Gotcha 3: Schema Registry và Serialization

**Vấn đề:** Dùng Avro/Protobuf với Schema Registry. Idempotent producer gửi lại batch → Schema Registry được gọi lại → Latency tăng trong retry path.

**Giải pháp:** Cache schema locally:
```java
// Trong Avro producer config
props.put("schema.registry.url", "http://schema-registry:8081");
props.put("auto.register.schemas", false); // Không auto-register trong production
props.put("use.latest.version", true);     // Cache latest schema version
```

### Gotcha 4: Consumer Group Rebalance trong Transaction

**Vấn đề:** Rebalance xảy ra giữa lúc đang trong transaction → `sendOffsetsToTransaction()` fail với `RebalanceInProgressException`.

**Giải pháp:** Implement `ConsumerRebalanceListener`:
```java
consumer.subscribe(List.of(INPUT_TOPIC), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        // Partitions bị lấy đi trước rebalance
        // Nếu đang trong transaction → abort
        if (currentTransactionActive) {
            producer.abortTransaction();
            currentTransactionActive = false;
        }
    }
    
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        // Partitions mới được assign sau rebalance
        // Reset state nếu cần
        log.info("Assigned partitions: {}", partitions);
    }
});
```

### Gotcha 5: EOS với Spring Kafka

```java
// Trong Spring Boot, dùng @Transactional KHÔNG phải Spring transaction
// Phải dùng Kafka transaction manager

@Configuration
public class KafkaConfig {
    
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, 
                  "spring-payment-" + System.getenv("POD_NAME"));
        // ...
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTransactionManager<String, String> kafkaTransactionManager(
            ProducerFactory<String, String> pf) {
        return new KafkaTransactionManager<>(pf);
    }
    
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String> factory(
            ConsumerFactory<String, String> cf,
            KafkaTransactionManager<String, String> tm) {
        
        var factory = new ConcurrentKafkaListenerContainerFactory<String, String>();
        factory.setConsumerFactory(cf);
        
        // Bật EOS trong Spring Kafka
        factory.getContainerProperties().setEosMode(EosMode.V2);
        factory.getContainerProperties().setTransactionManager(tm);
        
        return factory;
    }
}

// Trong service:
@KafkaListener(topics = "payment-requests", groupId = "payment-processor-group")
@Transactional("kafkaTransactionManager")  // Dùng Kafka TM, không phải JPA!
public void process(PaymentRequest request, 
                    @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                    @Header(KafkaHeaders.OFFSET) long offset) {
    
    PaymentResult result = validateAndProcess(request);
    kafkaTemplate.send("payment-processed", request.requestId(), result);
    // Spring tự handle sendOffsetsToTransaction + commitTransaction
}
```

---

*Triển khai EOS trong production đòi hỏi hiểu sâu cả về Kafka internals lẫn distributed systems failure modes. Đây không phải "enable một flag" mà là một architectural decision với nhiều trade-off cần cân nhắc kỹ. Luôn bắt đầu bằng integration tests + chaos tests trước khi deploy production.*
