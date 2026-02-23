# 1. Kafka Core Foundations

## Table of Contents

- [1.1 Kafka Architecture (Brokers, Topics, Partitions, Leaders & Followers, Replication & ISR)](#11-kafka-architecture-brokers-topics-partitions-leaders--followers-replication--isr)
- [Key mental model](#key-mental-model)

---

## 1.1 Kafka Architecture (Brokers, Topics, Partitions, Leaders & Followers, Replication & ISR)

Kafka is fundamentally a **distributed, append-only commit log** optimized for high-throughput publish-subscribe messaging at scale.

```ascii
                              KAFKA ARCHITECTURE HOÀN CHỈNH (KRaft Mode - 2026)

   ┌──────────────────────┐          Publish (key + value + timestamp)
   │   PRODUCERS          │ ────────────────────────────────►
   │ • Batching           │
   │ • linger.ms          │
   │ • Compression        │
   │ • Idempotent         │
   │ • Transactions       │
   └──────────────────────┘

                               ▲
                               │
                               │  (acks=all / min.insync.replicas)
                               │
   ┌─────────────────────────────────────────────────────────────────────┐
   │                         KAFKA CLUSTER (KRaft)                       │
   │                                                                     │
   │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
   │   │  Broker 1    │◄─►│  Broker 2    │◄─►│  Broker 3    │   ...      │
   │   │ (Controller) │   │              │   │              │            │
   │   │ Leader P0    │   │ Follower P0  │   │ Follower P0  │            │
   │   │ Follower P1  │   │ Leader P1    │   │ Follower P1  │            │
   │   └──────────────┘   └──────────────┘   └──────────────┘            │
   │                                                                     │
   │   KRaft Quorum (5 controllers khuyến nghị)                         │
   │   Metadata topic: __cluster_metadata                                │
   │                                                                     │
   │   TOPICS                                                            │
   │   • my-topic                                                        │
   │     ├── Partition 0  (Leader: Broker1, Replicas: 1,2,3 | ISR: 1,2)  │
   │     ├── Partition 1  (Leader: Broker2, Replicas: 2,3,1 | ISR: 2,3)  │
   │     └── ...                                                         │
   │                                                                     │
   │   Replication Factor = 3                                            │
   │   High Watermark (HW) + Leader Epoch                                │
   └─────────────────────────────────────────────────────────────────────┘
                               │
                               │  Fetch / Poll
                               ▼

   ┌──────────────────────┐
   │   CONSUMERS          │
   │ • Consumer Groups    │
   │ • Group Coordinator  │
   │ • Rebalancing        │
   │   (CooperativeSticky)│
   │ • Offset Commit      │
   └──────────────────────┘

                               │
                               ▼

   ┌─────────────────────────────────────────────────────────────┐
   │                KAFKA ECOSYSTEM (Production)                 │
   │                                                             │
   │  • Kafka Streams (Stateful, RocksDB, EOS)                   │
   │  • Kafka Connect + Debezium (CDC)                           │
   │  • Schema Registry (Avro/Protobuf)                          │
   │  • MirrorMaker 2 (Multi-region / Active-Active)             │
   │  • Cruise Control (Auto-rebalance)                          │
   │  • Monitoring: Prometheus + Grafana + Burrow                │
   └─────────────────────────────────────────────────────────────┘
```

### 1.1.1 **Broker**

Kafka Broker là server lưu trữ partitions và phục vụ request của producer và consumer. Một Kafka cluster thường gồm nhiều brokers để đảm bảo scalability và fault tolerance.

— Các tiến trình máy chủ riêng lẻ trong cụm Kafka. Chúng lưu trữ dữ liệu, phục vụ các yêu cầu của nhà sản xuất/người tiêu dùng và xử lý việc sao chép dữ liệu. Một cụm sản xuất hoạt động tốt có ít nhất 3 broker (số lẻ được ưu tiên để đảm bảo số lượng tối thiểu cần thiết trong chế độ KRaft).

**Broker** là đơn vị cốt lõi của Kafka cluster – đây chính là **server process** chạy trên mỗi máy chủ (physical hoặc container).

Mỗi Broker chịu trách nhiệm:

- Nhận và lưu trữ message từ Producer
- Phục vụ message cho Consumer (pull model)
- Thực hiện replication (leader ↔ follower)
- Xử lý metadata (ở chế độ KRaft)

Một Kafka cluster thường có **3–9+ brokers** (số lẻ để dễ quorum). Broker **không** phải là “queue server” kiểu truyền thống – nó là **distributed commit log server**.

**Kiến trúc bên trong một Broker (Kafka 4.1.x – KRaft only)**

```
┌─────────────────────────────────────────────────────────────┐
│                    Kafka Broker Process                     │
│                                                             │
│  ┌──────────────┐   ┌──────────────────────┐   ┌──────────┐ │
│  │ Request      │   │ Log Manager          │   │ Network  │ │
│  │ Processor    │◄─►│ (Segment Files)      │◄─►│ Layer    │ │
│  │ (N threads)  │   │ • .log               │   │ (Selector│ │
│  └──────────────┘   │ • .index             │   │ + Acceptor)│
│                     │ • .timeindex         │   └──────────┘ │
│                     │ • .leader-epoch-checkpoint │         │
│                     └──────────────────────┘               │
│                                                             │
│  ┌──────────────────────┐   ┌─────────────────────────────┐ │
│  │ Replica Manager      │   │ KRaft Controller (nếu là     │ │
│  │ • Leader/Follower    │   │   controller node)           │ │
│  │ • Fetcher threads    │   │   • __cluster_metadata topic │ │
│  │ • ISR tracking       │   └─────────────────────────────┘ │
│  └──────────────────────┘                                   │
│                                                             │
│  Storage: SSD/NVMe bắt buộc cho production                  │
│  File system: XFS hoặc ext4 với noatime, nodiratime         │
└─────────────────────────────────────────────────────────────┘
```

- **Log Segments**: Mỗi partition = 1 directory chứa nhiều segment (mặc định 1GB/segment). Broker dùng `LogCleaner` để xóa segment theo retention.
- **Page Cache**: Linux page cache cực kỳ quan trọng – Kafka hầu như **không** đọc từ disk mà đọc từ page cache (zero-copy sendfile).
- **Request Pipeline**:
  - Acceptor thread (1) → Selector threads (num.network.threads) → Request Handler threads (num.io.threads).
  - Mặc định: network=3, io=8 → phải tune theo core CPU.
- **Controller Role (KRaft)**: Một số broker kiêm luôn Controller (khuyến nghị 5 controller nodes riêng hoặc co-located). Controller quản lý toàn bộ metadata qua compact topic `__cluster_metadata`.
- **Replica Fetcher**: Mỗi follower có fetcher thread riêng kéo data từ leader (replica.fetch.max.bytes, replica.fetch.wait.max.ms).
- **Purgatory**: Nơi lưu các request đang chờ (produce/fetches) khi chưa đủ acks hoặc data.

**Adavanced**

1. Quorum là gì?

- Quorum = số lượng tối thiểu các nút (nodes) phải đồng ý (agree) để một hoạt động nào đó được coi là hợp lệ và có thể tiến hành.
  - Trong hầu hết các hệ thống phân tán dùng cơ chế đồng thuận (consensus), quorum thường là đa số (majority).
  - Công thức phổ biến nhất:
    Quorum = ⌊N/2⌋ + 1
    (N là tổng số nút trong nhóm)

- Mục đích chính của quorum:
  - Đảm bảo tính nhất quán (consistency)
  - Ngăn chặn split-brain (hai phần của cluster tự cho mình là đúng, dẫn đến dữ liệu mâu thuẫn)
  - Cho phép hệ thống vẫn hoạt động khi có một số nút bị lỗi (fault tolerance)

2. Log Segments ?

- **Log Segments trong Kafka** là một trong những phần quan trọng nhất để hiểu cách Kafka lưu trữ dữ liệu một cách hiệu quả, bền vững và có khả năng scale cao. Chúng ta sẽ đào sâu theo đúng format mentorship: từ senior đến staff-level.

Mỗi **partition** trong Kafka **không phải là một file log duy nhất** mà được chia thành nhiều **log segments** (các file segment liên tiếp).

- Mỗi segment là một file vật lý trên disk (thường ~1GB).
- Partition = chuỗi các segment nối tiếp: segment cũ → segment mới hơn → active segment (đang write).
- Producer luôn append vào **active segment cuối cùng**.
- Khi segment đạt giới hạn (size hoặc time), Kafka **roll** (cuộn) sang segment mới.
- Retention policy (xóa dữ liệu cũ) hoạt động ở mức **segment**: xóa nguyên segment khi hết hạn, chứ không xóa từng message.

Lợi ích lớn:

- Dễ xóa dữ liệu cũ (chỉ xóa file).
- Hiệu suất đọc/ghi cao nhờ OS page cache + zero-copy.
- Hỗ trợ compaction (nếu cleanup.policy=compact).

Cấu trúc thư mục của một partition (ví dụ topic orders, partition 0 trên broker):

```text
/kafka-logs/orders-0/
├── 00000000000000000000.index          ← sparse offset index
├── 00000000000000000000.timeindex      ← timestamp index
├── 00000000000000000000.snapshot       ← leader epoch checkpoint
├── 00000000000000000000.leader-epoch-checkpoint
├── 00000000000000000000.log            ← dữ liệu thực (record batches)
├── 00000000012345678901.log            ← segment tiếp theo
├── 00000000012345678901.index
└── ... (active segment không có .done)
```

**Các file chính trong mỗi segment**:

- `*.log` — chứa record batches (compressed nếu producer bật compression).
- `*.index` — offset index (sparse: ~1 entry mỗi 4KB hoặc configurable).
- `*.timeindex` — timestamp → offset mapping.
- `*.snapshot` & `*.leader-epoch-checkpoint` — hỗ trợ leader election và tiered storage.

**Rolling segment triggers** (bất kỳ điều kiện nào xảy ra trước):

- `log.segment.bytes` đạt ngưỡng (default **1 GiB = 1073741824 bytes** – vẫn giữ nguyên đến 2026).
- `log.segment.ms` đạt thời gian (default không set, tức dựa size).
- `log.roll.hours` / `log.roll.jitter.hours` (dùng để stagger rolling tránh spike I/O).
- Broker khởi động lại hoặc manual roll.

**Retention & Deletion**:

- Broker có background thread (`LogCleaner` hoặc `LogManager`) kiểm tra định kỳ (`log.retention.check.interval.ms` = 300s default).
- Segment nào **hoàn toàn** nằm ngoài retention window (`retention.ms` hoặc `retention.bytes`) → bị xóa.
- High-water mark + log start offset quyết định segment nào còn readable.
- Với **tiered storage** (KIP-405, production-ready từ ~4.0+): segment cũ được offload lên remote (S3/HDFS), local chỉ giữ "hotset" (log.local.retention.\*).

**Performance internals**:

- Kafka dùng **append-only** → sequential write → cực nhanh trên SSD.
- Đọc dùng **mmap** + page cache → hầu như không đọc disk nếu data còn hot.
- Zero-copy (sendfile) khi consumer fetch → không copy qua userspace.

3. Leader Election

- **Leader Election** là quá trình bầu chọn một broker (hoặc controller trong KRaft) làm **leader** cho một partition hoặc cho toàn bộ metadata cluster.

- Mỗi partition luôn có **chỉ một leader** duy nhất: tất cả produce và consume (read/write) đều đi qua leader.
- Các broker khác là **followers** → replicate data từ leader.
- Khi leader chết (crash, network partition, restart), cluster phải nhanh chóng bầu leader mới từ danh sách **ISR (In-Sync Replicas)** để tránh downtime.

4. Tiered Storage

**Tiered Storage** (KIP-405) tách storage thành hai tầng:

- **Local tier** (hot): SSD/NVMe trên broker → giữ dữ liệu mới (tail của log), latency thấp.
- **Remote tier** (cold): Object storage rẻ (S3, GCS, Azure Blob) → lưu segment cũ.

→ Cho phép retention rất dài (tháng/năm) mà không cần mua thêm disk local đắt đỏ.

Producer/Consumer vẫn hoạt động bình thường → fetch dữ liệu cũ sẽ tự động pull từ remote nếu cần.

---

### 1.1.2 **Producer**

Producer là client chịu trách nhiệm publish records vào Kafka Topic. Producer có thể gửi message theo batch để tăng throughput. Partition được quyết định bằng hashing của message key hoặc partitioner strategy.

- **Producer** là client gửi message vào Kafka.
- Producer có thể gửi message đến **topic** bất kỳ.
- Producer có thể chọn **partition** để gửi message (hoặc để Kafka tự chọn).
- Producer có thể chọn **key** để gửi message (key sẽ được dùng để chọn partition).

Khi producer gọi producer.send(record), message không được gửi ngay lập tức. Kafka producer là một batching + asynchronous engine cực kỳ tối ưu throughput. Toàn bộ flow gồm 2 phía:

- **Producer Client** (JVM/.NET/Python/Go…): batching, partitioning, compression, retry, idempotence.
- **Kafka Broker (Leader của partition)**: append vào log segment, replicate, update High-Watermark, trả response.

Mục tiêu: đạt hàng trăm nghìn messages/giây với latency thấp và exactly-once nếu cần.

**Deep Dive Internals**

#### Phía Producer Client (các bước 1–9)

1. **Tạo ProducerRecord**  
   `new ProducerRecord(topic, key, value)` → thêm headers, timestamp (nếu không có thì System.currentTimeMillis).

2. **Serializer**  
   KeySerializer & ValueSerializer chạy (StringSerializer, AvroSerializer, Protobuf…).

3. **Partitioner**
   - Nếu có key: `murmur2(key) % numPartitions` (default UniformStickyPartitioner từ Kafka 2.4+ để giảm latency).
   - Không key: round-robin hoặc sticky (gửi liên tục vào 1 partition trong 1 khoảng thời gian để batch tốt hơn).

4. **RecordAccumulator (bộ nhớ đệm chính)**  
   Message được gói thành **RecordBatch** (MemoryRecords) và thêm vào **Deque<RecordBatch>** tương ứng với (topic-partition).  
   Các config quan trọng:
   - `batch.size` (default 16KB) → batch đầy → sẵn sàng gửi.
   - `linger.ms` (default 0) → chờ thêm message (thường set 5–20ms ở production).
   - `buffer.memory` (default 32MB) → nếu đầy → block hoặc throw exception.

5. **Compression (nếu bật)**  
   Gzip / Snappy / LZ4 / Zstd áp dụng **trên toàn batch** (rất hiệu quả).

6. **Sender Thread (background thread duy nhất)**  
   Thức dậy khi:
   - Batch đầy
   - linger.ms hết hạn
   - `max.block.ms` hoặc buffer đầy
   - Manual `flush()`

7. **Metadata Refresh (rất quan trọng)**  
   Nếu metadata cũ (> `metadata.max.age.ms` = 5 phút) hoặc chưa biết leader của partition → gửi **MetadataRequest** (API key 3) đến một bootstrap broker bất kỳ.  
   Trong **KRaft 4.2**: metadata lấy từ **\_\_cluster_metadata** topic (compact) do Active Controller quản lý → cực nhanh, không còn ZooKeeper bottleneck.

8. **Xây dựng ProduceRequest (API key 0, version hiện tại ~12–13 ở 4.2)**  
   Gói nhiều batch (có thể nhiều topic-partition trong 1 request).  
   Thêm:
   - `acks` (0/1/all/-1)
   - `timeout.ms`
   - Nếu `enable.idempotence=true`: Producer ID (PID), Epoch, Sequence Number (per partition).
   - Nếu transactional: Transactional ID + control records (BEGIN/COMMIT).

!!! Sequence Number: Số thứ tự của message trong partition. Mỗi khi producer gửi message đến partition, sequence number sẽ tăng lên 1. Mỗi producer giữ sequence number riêng cho từng partition.

9. **Gửi TCP ProduceRequest**  
   Kết nối trực tiếp đến **Leader broker** của partition (không qua controller).  
   Sử dụng **Kafka NetworkClient** với async I/O (Netty-like).

#### Phía Broker – Leader (các bước 10–18)

10. **Network Layer (Acceptor + Selector threads)**  
    Acceptor nhận TCP connection → Selector (num.network.threads = 3 default) đọc byte từ socket buffer → parse thành Request.

11. **RequestChannel Queue**  
    Request được đẩy vào hàng đợi (bounded queue).

12. **Request Handler Threads (num.io.threads = 8 default)**  
    Lấy request ra → Authentication → Authorization (ACL) → Deserialize.

13. **ReplicaManager.appendRecords()**
    - Tìm partition → LogManager.getOrCreateLog().
    - Append **RecordBatch** vào **active segment** (.log file) – append-only, sequential write cực nhanh.
    - Viết index (.index, .timeindex) và leader-epoch-checkpoint.
    - Tính checksum (CRC32 hoặc CRC64).

14. **Update Log End Offset (LEO)**  
    LEO tăng lên.

15. **Xử lý theo acks**:
    - **acks=0**: Trả response ngay (fire-and-forget, nguy hiểm).
    - **acks=1**: Trả ngay sau khi append local (chỉ leader).
    - **acks=all** (hoặc -1): Chờ followers replicate đến High-Watermark (HW).

16. **Replication (Follower side)**  
    Followers gửi **FetchRequest** liên tục đến leader (replica.fetch.max.bytes).  
    Leader trả data → Followers append → gửi ack về leader.  
    Leader update **ISR** và **HW** (High-Watermark = offset đã replicate full ISR).

17. **Response**  
    ProduceResponse gửi về producer với **baseOffset** + error code (nếu có).

18. **Producer nhận response**
    - Thành công → callback `onSuccess(RecordMetadata)` (offset, timestamp…).
    - Thất bại → retry (max `retries` = Integer.MAX_VALUE default, `retry.backoff.ms`).
    - Nếu idempotent: kiểm tra sequence để dedup.

---

### 1.1.3 **Consumer**

Consumer đọc message từ topic partitions và duy trì offset để biết đã đọc tới đâu. Consumer thường hoạt động trong consumer group để scale việc xử lý message.

-**Kafka** KHÔNG push data mà consumer phải pull data. Consumer làm 3 việc:

```
poll() → fetch records
process → business logic
commit → move checkpoint

👉 Offset = progress checkpoint, KHÔNG phải acknowledgement.

Partition Cursor Manager
+
Stateful Offset Controller
+
Pull-based Stream Processor
```

Kafka hoạt động theo cơ chế **pull-based**, nghĩa là consumer chủ động yêu cầu dữ liệu từ broker thay vì broker đẩy dữ liệu đến consumer vì nó liên quan đến scalability, backpressure, và fault tolerance.

1. Flow Control (Backpressure): Kafka không biết consumer có thể xử lý nhanh đến mức nào, nên consumer phải tự pull data. Nếu consumer xử lý chậm hơn producer, data sẽ tích tụ trong buffer của consumer, dẫn đến memory overflow. Pull-based giúp consumer kiểm soát tốc độ xử lý.

2. Scalability: Pull-based giúp consumer dễ dàng scale up hoặc scale down. Nếu consumer cần xử lý nhiều data hơn, chỉ cần tăng số lượng consumer. Nếu consumer cần xử lý ít data hơn, chỉ cần giảm số lượng consumer.

3. Fault Tolerance: Pull-based giúp consumer dễ dàng phục hồi sau khi bị lỗi. Nếu consumer bị lỗi, chỉ cần restart consumer và nó sẽ tiếp tục đọc data từ offset cuối cùng.

> Quy tắc quan trọng: Một partition chỉ được đọc bởi một consumer trong cùng group.

-**Kafka Consumer Architecture**

```
Consumer Application
      │
      ├── Consumer Group
      │        │
      │        ├── Member 1
      │        ├── Member 2
      │        └── Member N
      │
      └── Group Coordinator (Broker)
```

> **Consumer Group**

Consumer Group là tập hợp các consumer cùng chia sẻ việc đọc dữ liệu từ một topic. Kafka đảm bảo mỗi partition trong một topic chỉ được một consumer trong cùng group xử lý tại một thời điểm. Consumer group giúp scale horizontal việc xử lý message và mỗi group sẽ duy trì offset riêng cho từng partition.

- Mỗi group có thể có nhiều consumer
- Mỗi consumer có thể có nhiều partition
- Trong một consumer group, mỗi partition chỉ được một consumer instance đọc tại một thời điểm.

> **Group Coordinator** Consumer group coordinator phát hiện consumer chết -> Trigger rebalance -> Reassign partitions -> Consumer mới resume từ offset cũ

- Một broker được chọn để:
  - Quản lý membership
  - Rebalance
  - Lưu group state
  - Quản lý offset commit

Khi một consumer crash, group coordinator sẽ phát hiện thông qua heartbeat timeout. Sau đó Kafka trigger rebalance để phân phối lại partitions cho các consumer còn lại. Consumer mới sẽ tiếp tục đọc từ offset cuối cùng đã commit.

> **Offset** — trái tim của Consumer

- Offset = vị trí đọc trong partition.
- Kafka lưu offset trong internal topic: `__consumer_offsets`.

> **Consumer Group Rebalancing**

- Consumer Group cho phép scale horizontally.

```
Topic partitions = đơn vị song song
Consumer instance = worker
Group coordinator = scheduler
```

- Rebalance xảy ra khi:
  - Một consumer join group
  - Một consumer leave group
  - Một partition chuyển từ leader này sang leader khác

- Rebalance có thể dẫn đến:
  - Dừng processing
  - Commit offset
  - Reopen connection

> **Offset Commit**

- Offset commit là quá trình lưu vị trí đọc trong partition.
- Auto commit vs Manual commit

> **Poll Loop Internal Mechanics**

```
while(true):
    poll()
    process
    heartbeat

poll() điều khiển:
- Heartbeat
- Rebalance
- Session timeout

Nếu không gọi poll đủ thường xuyên → consumer bị xem là chết.
```

> **Auto commit** là cơ chế Kafka consumer tự động lưu offset định kỳ mà không cần application gọi commit thủ công. Kafka client sẽ tự động commit offset sau mỗi khoảng thời gian.

- **Auto commit** được kích hoạt bởi `enable.auto.commit=true` (mặc định).
- Kafka sẽ commit offset sau mỗi `auto.commit.interval.ms` (mặc định 5 giây).
- Auto commit chỉ xảy ra khi consumer đã fetch message.

- **Manual commit** là quá trình lưu vị trí đọc trong partition thủ công.
- Kafka sẽ commit offset khi consumer gọi `commitSync()` hoặc `commitAsync()`.

> **Pause / Resume**

- Pause / Resume cho phép consumer tạm thời NGỪNG fetch message từ một hoặc nhiều partition nhưng vẫn giữ membership trong consumer group.

-Note: Pause poll vẫn chạy và fetch dừng nên không ảnh hưởng đến rebalance

- Use Cases chuẩn Production
  - Async worker pool:

```
poll batch
↓
đẩy vào worker queue
↓
queue đầy → pause
↓
queue rỗng → resume
```

    - Rate limiting external API
    - Large batch processing

> **Observability**

- Consumer Lag: lag = log_end_offset - committed_offset

- Consumer Lag = 0 → đang real-time
- Consumer Lag > 0 → đang chậm
- Consumer Lag < 0 → lỗi

#### Fan-out Architecture

- Fan-out là kỹ thuật cho phép nhiều consumer group đọc cùng một topic mà không ảnh hưởng lẫn nhau.

```
┌────────────────────────────────────────────────────────┐
│                       Kafka Cluster                    │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Topic A     │  │  Topic B     │  │  Topic C     │  │
│  │  Partition 1 │  │  Partition 1 │  │  Partition 1 │  │
│  │  Partition 2 │  │  Partition 2 │  │  Partition 2 │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
└────────────────────────────────────────────────────────┘
```

- Mỗi topic có thể có nhiều consumer group
- Mỗi consumer group có thể có nhiều consumer
- Mỗi partition chỉ có 1 consumer

- Fan-out cho phép scale horizontally
- Fan-out cho phép scale independently

> Mỗi consumer group có offset riêng → độc lập hoàn toàn.

#### Consumer Backpressure Architecture

> Consumer Backpressure Architecture là cách thiết kế consumer để tự điều chỉnh tốc độ đọc dữ liệu từ Kafka dựa trên khả năng xử lý thực tế của hệ thống downstream.

```
            Kafka
              │
              ▼
        Consumer Poll Loop
              │
              ▼
        Internal Buffer/Queue
              │
      ┌───────┴────────┐
      ▼                ▼
 Worker 1         Worker 2
      ▼                ▼
      └────── DB/API ──┘
              │
              ▼
        Commit offset
```

- Key idea: 👉 decouple ingestion và processing

- Real-world Big Tech Pattern

```
Kafka Consumer
   ↓
bounded queue
   ↓
async worker pool
   ↓
adaptive pause/resume
   ↓
batched offset commit
```

1. Poller (ingestion layer)

- poll message
- đẩy vào queue nội bộ
- Không xử lý business logic.

2. Processing Workers

- async workers
- thread pool / promise pool
- xử lý message

→ quyết định pause/resume.

---

### 1.1.4 **Topics**

Topic là logical stream của dữ liệu. Kafka lưu dữ liệu topic dưới dạng các partition commit logs.

- Named logical feeds/channels. Every message belongs to exactly one topic. Topics are multi-producer, multi-consumer.

- **Partitions** — The unit of parallelism and scalability. Each topic is split into multiple ordered, immutable logs called partitions. A single partition lives on one broker at a time (the leader), but is replicated to others.

- **Leaders & Followers** — For each partition: exactly one broker is the **leader** (all reads and writes go through it). The other replicas are **followers** (they passively replicate from the leader).

- **Replication & ISR (In-Sync Replicas)** — Every partition has a replication factor (usually 3 in production). ISR is the dynamic subset of replicas that are fully caught up with the leader (within replica.lag.time.max.ms). Only replicas in ISR are eligible to become leader.

Key mental model: Kafka provides **durability** via replication, **availability** via leader + followers, and **scalability** via partitioning.

### 1.2.2 Deep Dive Internals

- **Log Structure** — Each partition is a sequence of segment files on disk (e.g., 00000000000000012345.log + .index + .timeindex). Messages are never updated or deleted individually—only whole segments are deleted/expired based on retention.

- **Partition → Broker Mapping** — Controlled by the **controller** (in KRaft mode: a quorum of controller nodes; pre-4.0: ZooKeeper). Assignment is rack/zone-aware by default if broker.rack is set.

- **High-Water Mark (HW) & Leader Epoch** — The HW is the largest offset that all ISR members have replicated. Consumers only read up to HW to avoid seeing uncommitted data during failover. Leader epochs prevent "zombie leaders" after network partitions.

- **ISR Management** — Followers send fetch requests to leaders. If a follower falls behind > replica.lag.time.max.ms (default 30s), it's removed from ISR. If ISR drops below min.insync.replicas, producers with acks=all fail. When the leader recovers or a new leader is elected, followers catch up and re-join ISR.

- **KRaft Era (2025–2026 reality)** — As of early 2026, Kafka 4.1.x is current (bug-fix releases ongoing), and KRaft is the **only** supported mode since 4.0 (ZooKeeper fully removed). Metadata lives in a compact \_\_cluster_metadata topic managed by the controller quorum. This removes ZooKeeper as a failure domain and scales metadata ops dramatically better for clusters with 100k+ partitions.

Staff nuance: In large clusters, ISR shrinkage is the #1 cause of write unavailability. Monitor under_replicated_partitions and offline_partitions_count religiously.

- **Idempotent Producer**

- **Kafka Transactions**

- **Read Committed**

- **Exactly-once semantics (EOS)**

- **At-least-once**

- **At-most-once**

| Strategy     | Ordering     | Performance | Safety     | Use case         |
| ------------ | ------------ | ----------- | ---------- | ---------------- |
| No key       | ❌           | ⭐⭐⭐⭐⭐  | ⭐         | Logs             |
| Key-based    | ✅ (per key) | ⭐⭐⭐⭐    | ⭐⭐⭐     | User events      |
| Custom       | Depends      | Depends     | Depends    | Advanced systems |
| acks=0       | ❌           | ⭐⭐⭐⭐⭐  | ❌         | Metrics          |
| acks=1       | Partial      | ⭐⭐⭐⭐    | ⭐⭐       | Normal apps      |
| acks=all     | ✅           | ⭐⭐⭐      | ⭐⭐⭐⭐⭐ | Production       |
| Exactly once | ✅           | ⭐⭐        | ⭐⭐⭐⭐⭐ | Finance          |

### 1.2.3 Production Story / Real Failure Case

Mid-2025 at a large fintech (processing ~800k msg/s peak): They ran replication.factor=3, min.insync.replicas=2, but had not enabled rack-aware assignment properly. During an AWS AZ outage, two brokers in the same AZ died simultaneously. For ~18% of partitions, the leader + both followers were in that AZ → ISR dropped to 0 instantly.

Result: Total write outage for those partitions for ~40 minutes until manual intervention (unclean.leader.election.enable=true temporarily). Downstream payment processing stalled, SLA breach cost ~$180k in refunds/credits.

Postmortem root causes:

- No AZ diversity in replica placement
- No proactive alerting on ISR size == 1
- No chaos-tested failover

They later migrated to KRaft + strict rack awareness + auto-rebalancing with Cruise Control.

### 1.2.4 Best Practices + Common Mistakes

**Best Practices** (2026 production standard):

- replication.factor = 3 (minimum); 5 for very high durability needs
- min.insync.replicas = 2 (sweet spot: survives 1 broker down without blocking producers)
- unclean.leader.election.enable = false (protect data consistency)
- Use rack-aware replica assignment (broker.rack config)
- Monitor: under_replicated_partitions > 0, offline_partitions_count > 0, ISR shrink events
- For new clusters in 2026: start with KRaft (no ZooKeeper ever)

**Common Mistakes** (still seen in 2026):

- Setting replication.factor=1 in "dev" then copying to prod
- acks=1 + replication.factor=3 → fast but zero real durability
- Over-partitioning early (e.g., 1000 partitions for 10 MB/s topic) → rebalance storms
- Not monitoring ISR shrinkage during rolling restarts

### 1.2.5 System Design / Interview Perspective

Interview question: "Design a Kafka cluster for 500 microservices sending 2 GB/s peak with 99.99% availability."

Key points to cover:

- Partition count: Start conservative (rule of thumb: aim for 100–200 MB/s per partition max), scale up later.
- Replication: factor=3, min.insync=2, rack-aware across 3 AZs.
- Broker count: At least 9 (3 per AZ) for even distribution.
- KRaft quorum: 5 controllers (odd number) or co-located broker+controller with dedicated nodes for large clusters.
- Trade-off: More partitions → better consumer parallelism, but slower rebalances and higher metadata overhead.
- Edge case: What if entire AZ fails? → With rack awareness + min.insync=2, writes continue as long as 2 AZs are alive.

Staff mindset: Always quantify: "With 3 AZs, replication=3, rack-aware → p99.99 availability possible, but test it."

### 1.2.6 Q&A

1. If ISR shrinks to 1 (only leader left), what happens to producers with acks=all? What about consumers?

```
1. Scenario: min.insync.replicas <= ISR size
replication.factor=3
min.insync.replicas=1
acks=all

So if ISR = {leader} … => The leader is still “all ISR”
So technically:
- producer writes succeed
- Kafka will ack immediately

2. Scenario: min.insync.replicas > ISR size
replication.factor=3
min.insync.replicas=2
acks=all

Now if ISR shrinks:
ISR size = 1
min.insync.replicas = 2

=> Kafka will reject writes.
```

2. Why does Kafka not allow decreasing the number of partitions on a topic after creation?

```
Partition decrease is hard because of 4 invariants Kafka must preserve:
- Invariant #1 — Offsets Are Partition-Scoped and Immutable
Kafka tracks: (topic, partition, offset)
Example:
TopicA-P0 offset 1200
TopicA-P1 offset 300
TopicA-P2 offset 900

Consumers store committed offsets like:
consumer-group-X:
  P0 → 1200
  P1 → 300
  P2 → 900

Now imagine removing Partition 2.
What happens to offset 900?
    - Where does that data go?
    - How does the consumer resume?

Kafka cannot remap offsets safely.
Offsets are permanent identifiers.

- Invariant #2 — Partition Is the Unit of Ordering
Kafka guarantees ordering only within a partition:
If Kafka merged partitions:
    - msg ordering across partitions becomes ambiguous
    - replay semantics break

- Invariant #3 — Partition Placement + Replication Metadata
Each partition has:
* leader broker
* follower replicas
* ISR set
Shrinking partitions would require:
* reassigning leaders
* rewriting replica logs
* updating ISR safely

That’s basically a distributed reshuffle with data movement.

Kafka doesn’t do that automatically because it risks:
* downtime
* inconsistency
* data loss

- Invariant #4 — Keys and Hash Partitioning Would Break
Most production topics use keyed partitioning:
Now if partitions shrink:
So suddenly:
    - old events are in Partition 2
    - new events go to Partition 1
```

## 1.2 Producer Internals (Batching, linger.ms, compression, Idempotent Producer, Transactions)

### 1.2.1 Concept Explanation

Producers write records to partitions (key → partition via hash or custom). Key features: batching for throughput, idempotence for safe retries, transactions for atomic multi-topic writes.

### 1.2.2 Deep Dive Internals

- **Batching**: Producer buffers records per partition → sends when batch.size reached or linger.ms elapsed. Compression (gzip/snappy/lz4/zstd) applied per batch.
- **Idempotent Producer**: enable.idempotence=true → producer ID + sequence per partition. Broker deduplicates on seq mismatch. Retries safe.
- **Transactions**: transactional.id → init → begin → send (multiple topics/partitions) → commit/abort. Uses control messages + fencing. Guarantees exactly-once when combined with consumer EOS.

### 1.2.3 Production Story / Real Failure Case

E-commerce peak: linger.ms=0 → tiny batches → 40% CPU on brokers for metadata/acks. Switched to linger.ms=5, batch.size=1MB, compression=lz4 → throughput 3×, latency stable. Another case: no idempotence + network blip → massive duplicates → downstream billing double-charged until fix.

### 1.2.4. Best Practices + Common Mistakes

- linger.ms=5–20 ms, batch.size=1–2 MB, compression=lz4 (best perf/durability balance)
- Always enable idempotence unless using transactions (which imply it)
- Mistake: acks=0 + no idempotence → fast but lost messages on broker failure

### 1.2.5. System Design / Interview Perspective

High-throughput producer: tune batch + compression + acks=all + idempotence. For cross-topic atomicity (e.g., order + payment), use transactions (but accept higher latency).

### 1.2.6. Follow-up Questions

- Why does increasing linger.ms improve throughput but hurt latency p99?

```
  - linger.ms is basically: “How long the producer is willing to wait before sending a batch.”
  - Kafka producers don’t send each message immediately.

  - They try to batch messages together.
    So:
    - low linger → send quickly, smaller batches
    - high linger → wait longer, bigger batches
```

- What happens if two producers use the same transactional.id?

```
Kafka maintains producer transaction state in the Transaction Coordinator.
When a producer starts:

1. It sends InitProducerId request
2. Coordinator assigns a ProducerId (PID) + epoch
3. Epoch increases every time the producer restarts

Producer Epoch = Ownership Version
Kafka uses epochs to enforce:
    | Only the latest producer instance may write.
So:
    - Producer A starts → epoch = 5
    - Producer B starts with same transactional.id → epoch = 6

Now Kafka says:
✅ Producer B is the new owner
❌ Producer A is now fenced
```

---

## 1.3 Consumer Internals (Consumer Groups, Group Coordinator, Rebalancing strategies, Offset management)

### 1.3.1 Concept Explanation

Consumers in a group share topic partitions (each partition → one consumer). Group coordinator (broker) manages membership, offset commits.

### 1.3.2 Deep Dive Internals

- **Group Coordinator**: Broker owning group metadata (based on group.id hash).
- **Rebalancing**: On join/leave/failure → stop consumption → assign partitions (Range, RoundRobin, CooperativeSticky since 2.4+).
- **Offset Management**: Committed to \_\_consumer_offsets topic. auto.commit vs manual. seek/toBeginning/toEnd.
- **Cooperative Rebalancing** (default since 3.0+): Incremental → only revoke changed partitions.

### 1.3.3 Production Story / Real Failure Case

Streaming job: static membership → consumer restart → full stop-the-world rebalance → 15-min lag spike. Switched to cooperative → rebalance <30s, lag minimal.

### 1.3.4 Best Practices + Common Mistakes

- Use CooperativeStickyAssignor
- session.timeout.ms=10–45s, heartbeat.interval.ms=1/3 of that
- Manual offset commit after processing
- Mistake: auto.commit + long processing → duplicates on crash

### 1.3.5 System Design / Interview Perspective

Scale consumers = partition count. For low-latency, small max.poll.records + frequent poll(). For batch, larger poll + longer processing.

### 1.3.6 Follow-up Questions

- Why was RangeAssignor bad for uneven partition counts?
- What triggers a rebalance in cooperative mode vs eager?
- How do offsets behave if consumer leaves group without commit?

---

## 1.4 Delivery Semantics (At-most-once, At-least-once, Exactly-once)

### 1.4.1 Concept Explanation

- At-most-once: acks=0 or auto.commit + no retry → possible loss
- At-least-once: acks=1/all + retries + manual commit after process → possible duplicates
- Exactly-once: idempotent producer + transactions + EOS consumer

### 1.4.2 Deep Dive Internals

Exactly-once requires:

- Producer: idempotent + transactional
- Consumer: isolation.level=read_committed
- Offsets committed inside transaction

### 1.4.3 Production Story / Real Failure Case

Payment system: at-least-once → crash after process before commit → double payments. Fixed with transactions + read_committed.

### 1.4.4 Best Practices + Common Mistakes

- Default to at-least-once + idempotent downstream
- Use exactly-once only when needed (costly)
- Mistake: think acks=all = exactly-once (no — needs full chain)

### 1.4.5 System Design / Interview Perspective

Trade-off: exactly-once = higher latency & broker load. Prefer at-least-once + dedup when possible.

### 1.4.6 Follow-up Questions

- Can you achieve exactly-once without transactions?

```
Kafka transactions solve two coupled problems:
Problem A — Producer duplicates
Producer retries can create duplicates:
    * network timeout
    * broker failover
    * leader change
Idempotent producer solves some of this.
But only inside a single partition session.

Problem B — Consumer offset + output atomicity
The hardest part:
    | Processing + producing output + committing offsets must be atomic.
Otherwise:
Crash window:
    1. Consume message
    2. Produce result
    3. Crash before offset commit
On restart:
    * message is re-consumed
    * output is produced again
Duplicate side effects.

```

- What's the performance hit of read_committed?

```
To understand the cost, you need to know what Kafka must do.
With transactions, Kafka introduces:
    * Transaction Coordinator
    * Producer epochs
    * Transaction markers
    * Aborted transaction tracking
When a consumer uses read_committed, Kafka must filter data based on transaction state.
---
What extra work happens?
1. Consumer must consult transaction metadata
Each partition maintains:
    * Last Stable Offset (LSO)
LSO = offset up to which all transactions are resolved.
Consumer can only read up to LSO.
So Kafka adds an extra rule:
    | Don’t deliver records beyond LSO until commit/abort markers arrive.

2. Transaction markers add control traffic
Every transaction commit produces markers written to all touched partitions.
This increases:
    * broker IO
    * replication load
    * network chatter

3. Fetch response filtering
Broker must remove aborted records before returning them.
That means:
    * more CPU per fetch
    * more complicated read path

4. Increased end-to-end latency
Because consumers cannot read “in-flight” transactional batches until committed.
So you introduce an artificial delay:
    | Commit boundary becomes a latency boundary.
```

### 1.4.7 Mini Exercise

Produce → consume with different semantics:

- acks=0 + auto.commit → simulate loss
- acks=all + idempotence + manual commit after process
- Add transactions → observe atomic multi-send

Reply next with:

- Answers to all follow-up questions across sections
- Results/observations from any exercises you ran
- Which sub-topic felt weakest?

---

# 2. Production Engineering & Operations

## 2.1 Partition scaling trade-offs

### 2.1.1 Concept Explanation (Clear Senior-Level Understanding)

Partition scaling involves increasing (or rarely, effectively decreasing via topic recreation) the number of partitions in a topic to handle higher throughput or more consumers. More partitions allow parallel processing (e.g., more consumers in a group), but introduce overhead in metadata, rebalancing, and resource usage. You can't decrease partitions natively without data loss/recreation, so plan ahead.

### 2.1.2 Deep Dive Internals (Staff-Level Depth)

Adding partitions: Use `kafka-topics --alter --add-partitions`. New partitions get assigned to brokers via the controller. Existing data stays; only new messages distribute across all. Can't shrink due to offset immutability—offsets are per-partition, so reducing would orphan data.

Overhead: Each partition adds ~1-2MB memory per broker (for indexes), increases ZooKeeper/KRaft metadata size, and prolongs rebalances (coordinator must reassign all). In KRaft (standard in 4.1+ as of Feb 2026), metadata scales better, but 10k+ partitions still hit controller quorum limits.

Rule of thumb: Throughput per partition ~100-200 MB/s max; beyond that, split.

### 2.1.3 Production Story / Real Failure Case

At a streaming platform (~1M msg/s), initial 10 partitions/topic sufficed, but Black Friday spiked to 5M/s. Added 50 partitions mid-peak—triggered massive rebalance, consumers stalled 10 mins, lag hit 1hr. Worse: Uneven assignment hot-spotted brokers, causing OOM. Recovery: Rolled back adds, optimized consumers first.

Root: No pre-testing; added too many at once.

### 2.1.4 Best Practices + Common Mistakes

- Start with 1-10 partitions, monitor throughput/consumer lag, add in small increments (e.g., double).
- Use Cruise Control for auto-rebalancing post-add.
- Rack-aware when adding to avoid AZ imbalance.
- Mistakes: Adding during peak (do off-hours); over-partitioning (e.g., 1k for low-traffic) → bloated \_\_consumer_offsets topic.

### 2.1.5 System Design / Interview Perspective

Interview: "Scale Kafka for 10GB/s." Calculate: partitions = throughput / per-partition-max (e.g., 50 for 200MB/s each). Trade-off: More partitions = better scale but slower recovery (e.g., 1k partitions = 5-10min rebalance). Prefer key partitioning for order vs. round-robin for load.

### 2.1.6 Follow-up Questions to Challenge My Thinking

- What if you need to "decrease" partitions? What's the safe workaround?
- How does partition count affect KRaft controller performance vs. ZooKeeper?
- Why might too few partitions limit your consumer group scaling?

### 2.1.7 Mini Exercise or Practical Task

In your local KRaft cluster: Create topic with 2 partitions. Produce 10k msgs. Add 4 partitions. Consume with 2 consumers → observe reassignment via `kafka-consumer-groups --describe`. Note rebalance time. Try with 100 partitions—see metadata bloat (check broker logs).

## 2.2 Consumer Lag & Performance Tuning

### 2.2.1 Concept Explanation (Clear Senior-Level Understanding)

Consumer lag is the delta between latest producer offset and committed consumer offset (e.g., messages behind). Tuning involves optimizing fetch/poll configs, processing speed, and cluster resources to minimize lag under load.

### 2.2.2 Deep Dive Internals (Staff-Level Depth)

Lag = end_offset - committed_offset per partition. Consumers poll (fetch.min.bytes, fetch.max.wait.ms) to batch fetches. Tuning: Increase max.poll.records for batch processing; reduce session.timeout.ms for faster failure detection but more heartbeats.

Bottlenecks: Slow downstream (e.g., DB writes) → backpressure. Auto-scaling consumers via K8s signals on lag metrics. In 4.1+, cooperative rebalance minimizes lag spikes during scaling.

### 2.2.3 Production Story / Real Failure Case

Log analytics system: Lag spiked to 2 days during data surge. Cause: max.poll.records=500 too high + slow regex processing → poll timeouts, rebalances every 5 mins. Fixed by threading processing + lag-based alerting → recovered in 4hrs, but lost real-time insights.

### 2.2.4 Best Practices + Common Mistakes

- Tune fetch.max.bytes=50MB, max.poll.records=1000-5000 based on msg size.
- Monitor lag via Burrow or Kafka's consumer_lag metric.
- Pause/resume consumption on high lag.
- Mistakes: Auto-commit enabled → false low lag; ignoring CPU/network in consumers.

### 2.2.5 System Design / Interview Perspective

Design for p99 lag <1min: Partition count >= consumers needed; use async processing. Trade-off: Larger fetches = less overhead but higher memory.

### 2.2.6 Follow-up Questions to Challenge My Thinking

- How do you tune for bursty vs. steady traffic?
- What's the impact of max.poll.interval.ms on lag during slow processing?
- Why might lag persist even with idle consumers?

### 2.2.7 Mini Exercise or Practical Task

In cluster: Produce at 1k msg/s. Consume with default configs → measure lag (kafka-consumer-groups). Tune max.poll.records up/down, add slow sleep(0.1) in processing → fix lag. Use JMX metrics for before/after.

## 2.3 Broker Failures & Recovery

### 2.3.1 Concept Explanation (Clear Senior-Level Understanding)

Broker failures (crash, network) trigger leader elections and replication catch-up. Recovery: New leaders from ISR, followers resync logs.

### 2.3.2 Deep Dive Internals (Staff-Level Depth)

On failure: Controller detects via heartbeat loss, elects new leader from ISR. Followers fetch from new leader up to HW. If unclean election, possible data loss/duplicates. Auto.leader.rebalance.enable for preferred leaders.

In KRaft, faster detection (ms vs. seconds in ZooKeeper).

### 2.3.3 Production Story / Real Failure Case

Cloud outage: Broker OOM from unthrottled producers. ISR intact, quick election, but recovery fetch overwhelmed network → cascading lags. 20-min full recovery after throttling.

### 2.3.4 Best Practices + Common Mistakes

- replica.fetch.max.bytes throttle.
- Controlled shutdowns.
- Mistakes: No replica.lag.time.max.ms tuning → stale ISR.

### 2.3.5 System Design / Interview Perspective

HA: 3+ brokers/AZ. Trade-off: Higher replication = slower recovery but less risk.

### 2.3.6 Follow-up Questions to Challenge My Thinking

- Difference in recovery time: clean vs. unclean election?
- How to minimize data loss in total broker loss?
- Impact of broker failure on producers?

### 2.3.7 Mini Exercise or Practical Task

Kill broker → watch recovery (kafka-topics --describe). Measure time to ISR restore.

## 2.4 Leader Election Scenarios

### 2.4.1 Concept Explanation (Clear Senior-Level Understanding)

Leader election happens on partition creation, broker failure, or rebalancing. Ensures single write point.

### 2.4.2 Deep Dive Internals (Staff-Level Depth)

Controller elects from preferred replicas or ISR. Epoch bumps prevent old leaders. Offline vs. shutdown differ in grace.

### 2.4.3 Production Story / Real Failure Case

Rolling upgrade: Uncontrolled shutdowns → frequent elections, throughput drop 30%. Fixed with controlled mode.

### 2.4.4 Best Practices + Common Mistakes

- preferred.leader.election.
- Mistakes: unclean enabled always → inconsistencies.

### 2.4.5 System Design / Interview Perspective

Minimize via stable clusters. Trade-off: Fast election vs. data safety.

### 2.4.6 Follow-up Questions to Challenge My Thinking

- Scenario: Leader down, ISR=1?
- KRaft vs. ZooKeeper election speed?

### 2.4.7 Mini Exercise or Practical Task

Force election by shutting leader → observe.

## 2.5 Retention Policies (Time vs Size)

### 2.5.1 Concept Explanation (Clear Senior-Level Understanding)

Retention deletes old segments: time-based (log.retention.hours) or size-based (log.retention.bytes). Time for TTL, size for storage caps.

### 2.5.2 Deep Dive Internals (Staff-Level Depth)

Segments checked periodically; min.cleanable.dirty.ratio for compaction. Time uses timestamps, size total log.

### 2.5.3 Production Story / Real Failure Case

Audit logs: Time retention=7d, but surge → disk full before expiry. Switched to size + time.

### 2.5.4 Best Practices + Common Mistakes

- Hybrid: Both + compaction.
- Mistakes: Infinite retention → OOM.

### 2.5.5 System Design / Interview Perspective

Cost: Time = predictable, size = burst-proof.

### 2.5.6 Follow-up Questions to Challenge My Thinking

- Time vs. size in compliance?
- Compaction interaction?

### 2.5.7 Mini Exercise or Practical Task

Set retentions → produce → watch deletion.

## 2.6 Handling Message Duplication & Ordering Guarantees

### 2.6.1 Concept Explanation (Clear Senior-Level Understanding)

Duplication from retries; ordering per-partition (FIFO), not global.

### 2.6.2 Deep Dive Internals (Staff-Level Depth)

Idempotence dedups; transactions atomic. Key null = round-robin, no order.

### 2.6.3 Production Story / Real Failure Case

No idempotence → duplicates in analytics.

### 2.6.4 Best Practices + Common Mistakes

- Idempotence default.
- Mistakes: Assuming global order.

### 2.6.5 System Design / Interview Perspective

Per-key order for events.

### 2.6.6 Follow-up Questions to Challenge My Thinking

- Global order how?
- Duplication without idempotence?

### 2.6.7 Mini Exercise or Practical Task

Produce with/without idempotence → simulate failure.
