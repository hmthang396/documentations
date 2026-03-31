# Caching Strategies — Senior / Strong Senior / Staff Level

> Tài liệu này đi sâu vào tư duy thiết kế, trade-off, và các bài toán thực tế mà một kỹ sư cấp Senior trở lên phải nắm vững khi làm việc với hệ thống cache phân tán.

---

## Mục lục

1. [Tư duy nền tảng](#1-tư-duy-nền-tảng)
2. [Redis vs Memcached — Khi nào dùng gì](#2-redis-vs-memcached--khi-nào-dùng-gì)
3. [Các Cache Patterns cốt lõi](#3-các-cache-patterns-cốt-lõi)
4. [Cache Invalidation — Bài toán khó nhất](#4-cache-invalidation--bài-toán-khó-nhất)
5. [Cache Stampede & Thundering Herd](#5-cache-stampede--thundering-herd)
6. [Distributed Caching Architecture](#6-distributed-caching-architecture)
7. [Eviction Policies & Memory Management](#7-eviction-policies--memory-management)
8. [Consistency Models](#8-consistency-models)
9. [Cache Warming & Cold Start](#9-cache-warming--cold-start)
10. [Monitoring & Observability](#10-monitoring--observability)
11. [Security Considerations](#11-security-considerations)
12. [Interview-level Design Questions](#12-interview-level-design-questions)

---

## 1. Tư duy nền tảng

### Cache không phải là tối ưu hóa — đó là thiết kế kiến trúc

Kỹ sư Junior coi cache như một trick để tăng tốc. Kỹ sư Senior hiểu cache là **một layer trong data architecture** với lifecycle, consistency contract, và failure mode riêng của nó.

**Ba câu hỏi phải trả lời trước khi đặt cache:**

```
1. Cache ở đây giải quyết vấn đề gì? (latency / DB load / throughput?)
2. Data này có chịu được stale bao lâu? (tolerance window)
3. Khi cache miss hoặc crash, hệ thống còn hoạt động được không?
```

### The Caching Hierarchy

```
Client
  └── Browser Cache (HTTP Cache-Control)
        └── CDN / Edge Cache (Cloudflare, Akamai)
              └── API Gateway Cache
                    └── Application-level Cache (Redis/Memcached)
                          └── Database Query Cache (pg_bouncer, MySQL Query Cache)
                                └── Storage Engine Cache (Buffer Pool)
```

Mỗi tầng có latency target, invalidation strategy, và cost profile khác nhau. Staff engineer phải biết chọn đúng tầng thay vì nhét tất cả vào Redis.

---

## 2. Redis vs Memcached — Khi nào dùng gì

### So sánh trực tiếp

| Tiêu chí | Redis | Memcached |
|---|---|---|
| Data structures | String, Hash, List, Set, ZSet, Stream, HyperLogLog, Geo | String only |
| Persistence | RDB snapshot + AOF log | Không có |
| Replication | Master-Replica, Sentinel, Cluster | Không native |
| Transactions | MULTI/EXEC (optimistic), Lua scripts | Không |
| Pub/Sub | Có | Không |
| Memory efficiency | Tốt với nhiều kiểu dữ liệu | Tốt hơn với pure key-value đơn giản |
| Multi-threading | Single-threaded (I/O threaded từ Redis 6+) | Multi-threaded thật sự |
| Cluster mode | Redis Cluster native | Consistent hashing ở client |
| Throughput | ~100K-200K ops/sec (single node) | ~200K-400K ops/sec (multi-thread) |

### Khi nào chọn Memcached

- Workload là **pure caching**, không cần persistence hay complex data structures
- Cần **multi-threaded throughput** thực sự cho CPU-bound workload
- Team đã có infrastructure Memcached và không có lý do migration
- Cache item size **nhỏ và uniform** (< 1MB)

### Khi nào chọn Redis (phần lớn trường hợp)

- Cần Sorted Sets cho leaderboard, rate limiting, priority queue
- Cần Pub/Sub hoặc Streams cho event-driven architecture
- Cần distributed lock (SETNX + Lua script hoặc Redlock)
- Cần persistence để warm up cache sau restart
- Cần Lua scripting cho atomic complex operations
- Cần HyperLogLog cho approximate counting (UV, DAU)

> **Staff-level insight**: Trong production, Redis thắng Memcached ở 9/10 use case vì ecosystem và operational simplicity. Chỉ dùng Memcached khi profiling cho thấy Redis là bottleneck thực sự — không phải vì nghe "Memcached nhanh hơn".

---

## 3. Các Cache Patterns cốt lõi

### 3.1 Cache-Aside (Lazy Loading)

```
Application → Check Cache
                ├── HIT  → Return data
                └── MISS → Query DB → Write to Cache → Return data
```

```python
def get_user(user_id: str) -> User:
    cache_key = f"user:{user_id}"
    
    cached = redis.get(cache_key)
    if cached:
        return User.from_json(cached)
    
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    if user:
        redis.setex(cache_key, ttl=3600, value=user.to_json())
    
    return user
```

**Trade-offs:**
- ✅ Chỉ cache data thực sự được dùng
- ✅ Cache miss không gây outage (graceful degradation)
- ❌ Cache miss = 3 trips (cache + DB + cache write) → latency spike
- ❌ Data có thể stale trong TTL window
- ❌ Race condition giữa nhiều worker khi cache miss đồng thời

### 3.2 Write-Through

```
Application → Write to Cache → Write to DB (synchronous)
```

```python
def update_user(user_id: str, data: dict) -> User:
    user = db.update("UPDATE users SET ... WHERE id = ?", user_id, data)
    redis.setex(f"user:{user_id}", ttl=3600, value=user.to_json())
    return user
```

**Trade-offs:**
- ✅ Cache luôn consistent với DB
- ✅ Read luôn hit cache (nếu data đã được write)
- ❌ Write latency tăng (phải chờ cả cache write)
- ❌ Cache lưu cả data ít được read (write amplification)

### 3.3 Write-Behind (Write-Back)

```
Application → Write to Cache → Return → [Async] → Flush to DB
```

**Trade-offs:**
- ✅ Write latency thấp nhất
- ❌ **Data loss risk** nếu cache node crash trước khi flush
- ❌ Complex implementation (cần queue, retry logic)
- Dùng khi: counter increment, session data, analytics event aggregation

### 3.4 Read-Through

Cache layer tự động fetch từ DB khi miss — application chỉ giao tiếp với cache.

```
Application → Cache Layer
                ├── HIT  → Return
                └── MISS → Cache tự fetch DB → Populate → Return
```

Thường implement qua **cache proxy** như PgBouncer, ProxySQL, hoặc custom cache library.

### 3.5 Refresh-Ahead (Proactive Cache)

Cache tự gia hạn trước khi TTL expire, dựa trên access pattern dự đoán.

```python
def get_with_refresh_ahead(key: str, ttl: int, fetch_fn) -> Any:
    data, remaining_ttl = redis.get_with_ttl(key)
    
    if data is None:
        data = fetch_fn()
        redis.setex(key, ttl, data)
    elif remaining_ttl < ttl * 0.2:  # Còn < 20% TTL → refresh async
        background_task(lambda: redis.setex(key, ttl, fetch_fn()))
    
    return data
```

---

## 4. Cache Invalidation — Bài toán khó nhất

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

### 4.1 TTL-based Invalidation

Đơn giản nhất. Phù hợp khi data có **natural staleness tolerance** (product catalog, config, exchange rate...).

```python
# Sai — TTL quá uniform gây thundering herd
redis.setex(key, ttl=3600, value=data)

# Đúng — Jitter để phân tán expiration
import random
jitter = random.randint(-300, 300)  # ±5 phút
redis.setex(key, ttl=3600 + jitter, value=data)
```

### 4.2 Event-driven Invalidation

Invalidate cache ngay khi data thay đổi, thay vì chờ TTL.

**Pattern 1 — Direct Invalidation:**
```python
def update_product(product_id: str, data: dict):
    db.update(product_id, data)
    redis.delete(f"product:{product_id}")
    redis.delete(f"product_list:*")  # ❌ NGUY HIỂM với large keyspace
```

**Pattern 2 — CDC (Change Data Capture) + Message Queue:**

```
DB (MySQL binlog / Postgres WAL)
  └── Debezium / Maxwell
        └── Kafka topic: db.products.changes
              └── Cache Invalidator Service
                    └── Redis.delete(affected_keys)
```

Đây là pattern **Staff-level** cho distributed system: loose coupling, audit trail, replay capability.

### 4.3 Cache Tags / Dependency Tracking

Nhóm nhiều cache key dưới một "tag" để invalidate theo nhóm.

```python
class TaggedCache:
    def set(self, key: str, value: Any, tags: list[str], ttl: int):
        redis.setex(key, ttl, serialize(value))
        for tag in tags:
            redis.sadd(f"tag:{tag}", key)
            redis.expire(f"tag:{tag}", ttl + 3600)
    
    def invalidate_tag(self, tag: str):
        keys = redis.smembers(f"tag:{tag}")
        if keys:
            redis.delete(*keys)
        redis.delete(f"tag:{tag}")

# Usage
cache.set(
    key="product_page:123",
    value=rendered_html,
    tags=["product:123", "category:electronics", "brand:apple"],
    ttl=3600
)

# Khi update product 123:
cache.invalidate_tag("product:123")
```

### 4.4 Version-based Cache Keys

```python
def get_user_cache_key(user_id: str) -> str:
    version = redis.get(f"user:{user_id}:version") or "1"
    return f"user:{user_id}:v{version}"

def invalidate_user(user_id: str):
    # Increment version → old key tự stale, không cần delete
    redis.incr(f"user:{user_id}:version")
```

---

## 5. Cache Stampede & Thundering Herd

### Vấn đề

Khi một cache key expire, **hàng trăm request đồng thời** hit DB để fetch data → DB overload.

```
T=0: key "trending_products" expire
T=1ms: 1000 requests check cache → MISS
T=2ms: 1000 requests query DB simultaneously → DB dies
```

### Giải pháp 1: Mutex Lock (Probabilistic)

```python
import uuid

def get_with_lock(key: str, fetch_fn, ttl: int) -> Any:
    data = redis.get(key)
    if data:
        return deserialize(data)
    
    lock_key = f"lock:{key}"
    lock_token = str(uuid.uuid4())
    
    # Try acquire lock (NX = only set if not exists)
    acquired = redis.set(lock_key, lock_token, nx=True, ex=10)
    
    if acquired:
        try:
            data = fetch_fn()
            redis.setex(key, ttl, serialize(data))
            return data
        finally:
            # Release lock với Lua script để atomic check-and-delete
            lua_script = """
                if redis.call("get", KEYS[1]) == ARGV[1] then
                    return redis.call("del", KEYS[1])
                else
                    return 0
                end
            """
            redis.eval(lua_script, 1, lock_key, lock_token)
    else:
        # Wait và retry (exponential backoff)
        time.sleep(0.1)
        return get_with_lock(key, fetch_fn, ttl)
```

### Giải pháp 2: Probabilistic Early Expiration (XFetch)

Thay vì đợi key expire, **tự động refresh sớm** với xác suất tỉ lệ nghịch với remaining TTL.

```python
import math, random, time

def xfetch(key: str, fetch_fn, ttl: int, beta: float = 1.0) -> Any:
    """
    XFetch algorithm (Vattani et al., 2015)
    beta: điều chỉnh aggressiveness của early refresh (default=1.0)
    """
    result = redis.get_with_ttl(key)  # returns (value, remaining_ttl)
    
    if result:
        value, remaining_ttl = result
        # Tính xác suất refresh sớm
        delta = ttl - remaining_ttl  # thời gian đã cache
        if delta > 0:
            early_recompute_prob = math.exp(-remaining_ttl / (beta * delta))
            if random.random() < early_recompute_prob:
                # Async refresh
                background_refresh(key, fetch_fn, ttl)
        return deserialize(value)
    
    # True cache miss
    data = fetch_fn()
    redis.setex(key, ttl, serialize(data))
    return data
```

### Giải pháp 3: Stale-While-Revalidate

```python
def get_stale_while_revalidate(key: str, fetch_fn, ttl: int, stale_ttl: int) -> Any:
    """
    Trả về data cũ ngay lập tức, đồng thời refresh async trong background.
    stale_ttl: thêm bao lâu data có thể stale sau khi "expire"
    """
    stale_key = f"{key}:stale"
    
    # Check fresh cache
    data = redis.get(key)
    if data:
        return deserialize(data)
    
    # Fresh expired → check stale copy
    stale_data = redis.get(stale_key)
    if stale_data:
        # Trả về stale ngay, refresh async
        background_task(lambda: refresh_cache(key, stale_key, fetch_fn, ttl, stale_ttl))
        return deserialize(stale_data)
    
    # Both expired → synchronous fetch (cold start)
    return refresh_cache(key, stale_key, fetch_fn, ttl, stale_ttl)

def refresh_cache(key, stale_key, fetch_fn, ttl, stale_ttl):
    data = fetch_fn()
    serialized = serialize(data)
    redis.setex(key, ttl, serialized)
    redis.setex(stale_key, ttl + stale_ttl, serialized)
    return data
```

---

## 6. Distributed Caching Architecture

### 6.1 Redis Sentinel (High Availability)

```
          ┌─────────────┐
          │  Sentinel 1 │
          └──────┬──────┘
                 │ monitor
    ┌────────────┼────────────┐
    │            │            │
┌───▼───┐   ┌───▼───┐   ┌───▼───┐
│Master │──▶│Replica│   │Replica│
└───────┘   └───────┘   └───────┘
    ▲
    │ failover
┌───┴───┐
│  App  │ (discovers master via Sentinel)
└───────┘
```

- Sentinel **monitor** master và replica
- Nếu master down, Sentinel **bầu chọn** một replica lên làm master
- Client phải implement **Sentinel-aware** connection (hầu hết Redis client đã support)

**Limitation**: Vẫn là single master → write throughput bị giới hạn.

### 6.2 Redis Cluster (Horizontal Scaling)

```
Slot range: 0-16383 (16384 slots total)

Shard 1: slots 0-5460       Shard 2: slots 5461-10922      Shard 3: slots 10923-16383
┌─────────────────┐         ┌─────────────────┐             ┌─────────────────┐
│ Master (node 1) │         │ Master (node 3) │             │ Master (node 5) │
│ Replica(node 2) │         │ Replica(node 4) │             │ Replica(node 6) │
└─────────────────┘         └─────────────────┘             └─────────────────┘
```

```python
# Key hashing: CRC16(key) % 16384 → slot → node
# Hash tags: {user}.profile và {user}.settings → cùng slot
redis.set("{user:123}.profile", data)
redis.set("{user:123}.settings", config)
# → Đảm bảo cross-key operations (MGET, transactions) hoạt động
```

**Key considerations cho Staff:**

```python
# ❌ Sai — cross-slot operation sẽ fail trong cluster mode
redis.mget("user:1", "user:2", "user:3")  # Keys có thể ở khác slot

# ✅ Đúng — dùng hash tags
redis.mget("{user:1}", "{user:2}")  # Nếu muốn đảm bảo cùng slot

# Hoặc pipeline individual gets
pipe = redis.pipeline()
for uid in user_ids:
    pipe.get(f"user:{uid}")
results = pipe.execute()
```

### 6.3 Multi-Layer Caching (L1/L2)

```
Request
  └── L1 Cache (In-process, e.g. Caffeine/Guava in Java, functools.lru_cache)
        └── L2 Cache (Redis — shared across instances)
              └── Database
```

```python
from functools import lru_cache
import time

class TwoLevelCache:
    def __init__(self, redis_client, l1_ttl=60, l2_ttl=3600):
        self.redis = redis_client
        self.l1_ttl = l1_ttl
        self.l2_ttl = l2_ttl
        self._l1 = {}  # {key: (value, expire_at)}
    
    def get(self, key: str, fetch_fn):
        # L1 check
        if key in self._l1:
            value, expire_at = self._l1[key]
            if time.time() < expire_at:
                return value
            del self._l1[key]
        
        # L2 check
        cached = self.redis.get(key)
        if cached:
            value = deserialize(cached)
            self._l1[key] = (value, time.time() + self.l1_ttl)
            return value
        
        # DB fetch
        value = fetch_fn()
        self.redis.setex(key, self.l2_ttl, serialize(value))
        self._l1[key] = (value, time.time() + self.l1_ttl)
        return value
```

**Trade-off**: L1 tăng hiệu năng nhưng tạo **inconsistency window** giữa các app instances. Chỉ dùng cho data ít thay đổi và có thể tolerate stale ngắn.

---

## 7. Eviction Policies & Memory Management

### Redis Eviction Policies

| Policy | Hành vi | Dùng khi |
|---|---|---|
| `noeviction` | Trả lỗi khi full | Cache là source of truth (không được mất data) |
| `allkeys-lru` | Evict key ít dùng nhất (toàn bộ keyspace) | General cache workload |
| `allkeys-lfu` | Evict key ít truy cập nhất (frequency) | Workload có long tail access pattern |
| `volatile-lru` | LRU nhưng chỉ key có TTL | Mix cache + persistent data trong 1 Redis |
| `volatile-lfu` | LFU nhưng chỉ key có TTL | Tương tự volatile-lru |
| `volatile-ttl` | Evict key sắp expire trước | Muốn cache tự "clean up" theo thứ tự expire |
| `allkeys-random` | Random eviction | Không có pattern rõ ràng |

### Khi nào dùng LRU vs LFU?

```
LRU (Least Recently Used):
  - Tốt với workload có temporal locality (recently accessed = likely accessed again)
  - Workload: session data, recent activity feeds

LFU (Least Frequently Used):
  - Tốt với workload có popularity-based access (hot items luôn hot)
  - Workload: product catalog, content platform, recommendation results
  - Ít bị "cache pollution" bởi one-time bulk scan
```

### Memory Optimization

```
# Redis memory report
redis-cli info memory

# Kiểm tra memory usage per key type
redis-cli --bigkeys

# Object encoding — Redis tự optimize
# Hash với ít field → ziplist (compact)
# Hash với nhiều field → hashtable
# Ảnh hưởng bởi: hash-max-ziplist-entries, hash-max-ziplist-value
```

```python
# Sử dụng Hash thay vì String để lưu object — tiết kiệm ~30% memory
# ❌ Kém efficient
redis.set("user:123:name", "Alice")
redis.set("user:123:email", "alice@example.com")
redis.set("user:123:age", "30")

# ✅ Hiệu quả hơn
redis.hset("user:123", mapping={"name": "Alice", "email": "alice@example.com", "age": 30})
```

---

## 8. Consistency Models

### 8.1 Cache Consistency với Database

**Strong Consistency** — Không thực tế với distributed cache:
```
Write to DB → Sync write to Cache → Return (2-phase commit style)
```

**Eventual Consistency** — Phổ biến nhất:
```
Write to DB → Async invalidate/update Cache → Short inconsistency window
```

**Read-Your-Writes Consistency:**
```python
def update_user_profile(user_id: str, data: dict, request_id: str):
    db.update(user_id, data)
    
    # Invalidate cache ngay để user thấy change của chính mình
    redis.delete(f"user:{user_id}")
    
    # Đánh dấu write intent trong 30s (cho sticky session / read-your-writes)
    redis.setex(f"ryw:{user_id}", 30, "1")

def get_user_profile(user_id: str):
    # Nếu user vừa write trong 30s → bypass cache
    if redis.exists(f"ryw:{user_id}"):
        return db.get(user_id)
    
    return get_from_cache_or_db(user_id)
```

### 8.2 The "Double Delete" Pattern

Giải quyết race condition giữa cache write và DB write:

```python
def update_user(user_id: str, data: dict):
    # Step 1: Delete cache trước khi write DB
    redis.delete(f"user:{user_id}")
    
    # Step 2: Write DB
    db.update(user_id, data)
    
    # Step 3: Delete cache lần 2 (xử lý race với concurrent reads)
    time.sleep(0.1)  # Small delay
    redis.delete(f"user:{user_id}")
```

> **Vấn đề còn lại**: Window nhỏ giữa step 1 và step 3 vẫn có thể bị concurrent read populate stale data. Giải pháp tốt hơn là CDC-based invalidation.

---

## 9. Cache Warming & Cold Start

### Vấn đề Cold Start

Sau khi deploy mới hoặc Redis restart, cache empty → mọi request hit DB → **thundering herd + slow response**.

### Strategies

**1. Pre-warming từ DB snapshot:**

```python
async def warm_up_cache():
    """Chạy trước khi app nhận traffic"""
    
    # Warm top 10000 products by sales
    hot_products = db.query("""
        SELECT id FROM products 
        ORDER BY total_sales DESC 
        LIMIT 10000
    """)
    
    batch_size = 100
    for i in range(0, len(hot_products), batch_size):
        batch = hot_products[i:i+batch_size]
        pipe = redis.pipeline()
        for product_id in batch:
            data = db.get_product(product_id)
            pipe.setex(f"product:{product_id}", 3600, serialize(data))
        await pipe.execute()
        await asyncio.sleep(0.01)  # Throttle để không overload DB
    
    logger.info(f"Warmed {len(hot_products)} products")
```

**2. Lazy warming với priority queue:**

```python
# Xây dựng access log → xác định hot keys → warm theo thứ tự priority
# Access log: Redis Sorted Set — score = access_count
redis.zincrby("cache:access_log", 1, key)

# Top 1000 hot keys
hot_keys = redis.zrevrange("cache:access_log", 0, 999, withscores=True)
```

**3. Persistent cache với Redis RDB:**

```yaml
# redis.conf
save 900 1       # Snapshot sau 900s nếu có >= 1 change
save 300 10      # Snapshot sau 300s nếu có >= 10 changes
save 60 10000    # Snapshot sau 60s nếu có >= 10000 changes

# Sau restart, Redis tự load từ .rdb file → cache warm ngay lập tức
```

---

## 10. Monitoring & Observability

### Key Metrics phải track

```
Hit Rate = Cache Hits / (Cache Hits + Cache Misses)
  → Target: > 90% cho production cache
  → Alert nếu < 80% (thường báo hiệu bug hoặc data pattern thay đổi)

Memory Usage:
  → used_memory / maxmemory < 80% (để tránh eviction bất ngờ)
  → mem_fragmentation_ratio: 1.0-1.5 là healthy, > 1.5 → fragmentation cao

Eviction Rate:
  → evicted_keys tăng đột ngột → maxmemory quá thấp hoặc TTL quá ngắn

Connection Count:
  → connected_clients → spike báo hiệu connection leak

Latency:
  → latency_ms p99 < 1ms (local Redis)
  → latency_ms p99 < 5ms (remote Redis)
```

### Redis monitoring commands

```bash
# Real-time stats
redis-cli --stat

# Slowlog (commands > threshold)
redis-cli config set slowlog-log-slower-than 1000  # microseconds
redis-cli slowlog get 25

# Monitor (debug only — very expensive in production)
redis-cli monitor

# Info all metrics
redis-cli info all

# Keyspace analysis
redis-cli --scan --pattern "user:*" | wc -l
```

### Grafana Dashboard metrics (via redis_exporter)

```yaml
Panels cần có:
  - Cache hit rate (percentage, time series)
  - Memory usage vs maxmemory
  - Ops/sec by command type (GET, SET, DEL...)
  - Connected clients
  - Replication lag (master → replica)
  - Key evictions/sec
  - Network I/O
  - Keyspace hits vs misses
```

---

## 11. Security Considerations

### Authentication & Authorization

```
# Redis 6+ ACL
redis-cli ACL SETUSER app_service on >password ~cache:* +GET +SET +DEL +EXPIRE
redis-cli ACL SETUSER readonly_service on >password ~* +GET

# Chạy Redis với requirepass (legacy, vẫn cần cho Redis < 6)
requirepass "your-strong-password-here"
```

### Network Security

```
# Bind chỉ internal network — KHÔNG bao giờ expose Redis ra public
bind 127.0.0.1 10.0.0.1

# Disable dangerous commands trong production
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG ""
rename-command DEBUG ""

# Hoặc dùng ACL để restrict
redis-cli ACL SETUSER default on nopass ~* +@all -FLUSHALL -FLUSHDB -CONFIG
```

### TLS Encryption

```
# redis.conf
tls-port 6380
tls-cert-file /etc/ssl/redis/redis.crt
tls-key-file /etc/ssl/redis/redis.key
tls-ca-cert-file /etc/ssl/redis/ca.crt
```

### Data Sensitivity

```python
# Không cache sensitive data (passwords, tokens, PII) mà không encrypt
# Hoặc: chỉ cache non-sensitive fields

# ❌ Nguy hiểm
redis.set(f"user:{id}", json.dumps({"name": ..., "password_hash": ..., "ssn": ...}))

# ✅ An toàn hơn
safe_fields = {k: v for k, v in user.items() if k not in SENSITIVE_FIELDS}
redis.set(f"user:{id}", json.dumps(safe_fields))
```

---

## 12. Interview-level Design Questions

### Q1: Design a rate limiter using Redis

```python
def is_rate_limited(user_id: str, limit: int, window_seconds: int) -> bool:
    """
    Sliding window rate limiter.
    Sử dụng Redis Sorted Set với timestamp làm score.
    """
    now = time.time()
    window_start = now - window_seconds
    key = f"ratelimit:{user_id}"
    
    pipe = redis.pipeline()
    # Remove expired entries
    pipe.zremrangebyscore(key, 0, window_start)
    # Count current window
    pipe.zcard(key)
    # Add current request
    pipe.zadd(key, {str(now): now})
    # Reset TTL
    pipe.expire(key, window_seconds)
    
    _, count, _, _ = pipe.execute()
    
    return count >= limit

# Fixed window (simpler but burstable at window boundary):
def is_rate_limited_fixed(user_id: str, limit: int, window: int) -> bool:
    key = f"ratelimit:{user_id}:{int(time.time() // window)}"
    count = redis.incr(key)
    if count == 1:
        redis.expire(key, window)
    return count > limit
```

### Q2: Implement distributed lock với Redis

```python
class RedisLock:
    def __init__(self, redis_client, key: str, ttl: int = 30):
        self.redis = redis_client
        self.key = f"lock:{key}"
        self.ttl = ttl
        self.token = str(uuid.uuid4())
    
    def __enter__(self):
        deadline = time.time() + self.ttl
        while time.time() < deadline:
            if self.redis.set(self.key, self.token, nx=True, ex=self.ttl):
                return self
            time.sleep(0.05)
        raise LockAcquisitionError(f"Could not acquire lock: {self.key}")
    
    def __exit__(self, *args):
        # Lua script đảm bảo atomic check-and-delete
        script = """
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        """
        self.redis.eval(script, 1, self.key, self.token)

# Usage:
with RedisLock(redis, "checkout:order:123", ttl=10):
    process_order(123)
```

**Redlock** (multi-node lock): Acquire lock trên majority of N (≥ 3) Redis nodes. Chỉ thành công nếu acquire được trên > N/2 nodes. Bảo vệ khỏi single node failure.

### Q3: Design a leaderboard system

```python
# Redis Sorted Set — O(log N) cho mọi operation

def update_score(leaderboard: str, user_id: str, score: float):
    redis.zadd(leaderboard, {user_id: score})

def get_rank(leaderboard: str, user_id: str) -> int:
    # 0-indexed, ascending. ZREVRANK cho descending (highest = rank 0)
    return redis.zrevrank(leaderboard, user_id)

def get_top_n(leaderboard: str, n: int) -> list:
    return redis.zrevrange(leaderboard, 0, n-1, withscores=True)

def get_around_user(leaderboard: str, user_id: str, window: int = 5) -> list:
    rank = redis.zrevrank(leaderboard, user_id)
    start = max(0, rank - window)
    end = rank + window
    return redis.zrevrange(leaderboard, start, end, withscores=True)

# Leaderboard theo time window (weekly, monthly):
# Key pattern: leaderboard:weekly:2024-W01
# Tự động expire sau khi window kết thúc
```

### Q4: Cache cho hệ thống social feed (News Feed)

```
Fan-out on write (Push model):
  User A posts → Push to all followers' feed cache
  → Tốt cho: read-heavy, ít followers
  → Vấn đề: Celebrity problem (1M followers → 1M cache writes)

Fan-out on read (Pull model):
  User reads feed → Aggregate posts from all followees
  → Tốt cho: write-heavy, nhiều followers
  → Vấn đề: Read latency cao

Hybrid (Facebook/Twitter approach):
  - Regular users (<= 10K followers): Fan-out on write
  - Celebrities (> 10K followers): Fan-out on read (inject vào feed lúc đọc)
  - Cache per-user feed (Redis List hoặc Sorted Set by timestamp)
  - TTL: 24-48 giờ
```

---

## Tổng kết: Mindset của Staff Engineer về Caching

```
1. MEASURE FIRST — Đừng cache vì "có vẻ cần". Profile latency, DB load thực tế.

2. DEFINE SLO — Bao lâu stale là chấp nhận được? 
   Câu trả lời này quyết định toàn bộ invalidation strategy.

3. PLAN FOR FAILURE — Cache sẽ die. DB phải survive khi cache miss 100%.
   Implement circuit breaker, fallback, và load shedding.

4. AVOID PREMATURE OPTIMIZATION — Cache-aside + TTL đủ cho 80% use case.
   Chỉ leo thang lên CDC/write-behind khi có data chứng minh cần thiết.

5. OPERATIONAL COST — Cache phức tạp = operational burden.
   Write-behind cache không thể rollback dễ dàng. Event-driven invalidation
   cần thêm Kafka, CDC pipeline. Đánh giá team có maintain được không.

6. SECURITY BY DEFAULT — Không bao giờ expose Redis ra public internet.
   Không cache plaintext sensitive data. Luôn set maxmemory.
```

---

*Tài liệu này cover các chủ đề cốt lõi cho system design interview và production engineering ở level Senior → Staff. Các chủ đề nâng cao hơn bao gồm: Redis Modules (RedisSearch, RedisGraph), CRDT-based replication, và cache-aside với CAS (Compare-And-Swap) semantics.*
