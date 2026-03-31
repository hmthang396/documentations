# Database Services - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về 4 database services chính của GCP, biết khi nào sử dụng, cấu hình, architecture, best practices, và exam scenarios.

---

## Mục Lục

1. [Cloud SQL (Relational Database)](#1-cloud-sql-relational-database)
2. [Cloud Firestore (NoSQL Document)](#2-cloud-firestore-nosql-document)
3. [Cloud Bigtable (NoSQL Wide-Column)](#3-cloud-bigtable-nosql-wide-column)
4. [Cloud Spanner (Distributed SQL)](#4-cloud-spanner-distributed-sql)
5. [So Sánh & Chọn Service](#so-sánh--chọn-service)
6. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Cloud SQL (Relational Database)

### 1.1 Khái Niệm & Tổng Quan

**Cloud SQL là gì?**
- Fully managed relational database service
- Hỗ trợ: MySQL, PostgreSQL, SQL Server
- Tự động backups, patches, updates
- High availability options (multi-zone failover)
- Pay per instance-hour + storage

**Tại sao dùng Cloud SQL?**
- Existing SQL applications
- ACID transactions required
- Complex queries & joins
- Structured relational data
- Don't want to manage infrastructure

**Khi không dùng Cloud SQL:**
- NoSQL data → Firestore, Bigtable
- Petabyte scale → BigQuery
- Extreme performance → Bigtable
- Global distributed → Cloud Spanner

---

### 1.2 Databases & Instances

#### Instance Types

**Shared-Core (Budget):**
```bash
# f1-micro, g1-small (burstable)
# Suitable for: Development, testing, small production
# Cost: ~$7-15/month
gcloud sql instances create dev-db \
  --database-version=MYSQL_8_0 \
  --tier=db-f1-micro \
  --region=us-central1
```

**Standard (Production):**
```bash
# db-n1-standard-1 to db-n1-standard-4
# Fixed performance, 1-4 vCPU, 3.75-15GB RAM
# Cost: ~$30-150/month
gcloud sql instances create prod-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL  # High availability
```

**High-Memory:**
```bash
# db-n1-highmem-2 to db-n1-highmem-32
# Large datasets, 2-32 vCPU, 13-208GB RAM
# Cost: High (reserved instances available)
gcloud sql instances create analytics-db \
  --database-version=MYSQL_8_0 \
  --tier=db-n1-highmem-4 \
  --region=us-central1
```

#### Instance Components

```
Cloud SQL Instance
├─ Storage (SSD/HDD, 10GB - 65TB)
├─ Backups (automatic daily + on-demand)
├─ Read replicas (same or cross-region)
├─ Databases (multiple per instance)
├─ Users & Permissions
└─ Flags (configuration)
```

#### Creating Instances

```bash
# Detailed instance creation
gcloud sql instances create my-database \
  --database-version=MYSQL_8_0 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=PD-SSD \
  --storage-size=100GB \
  --storage-auto-increase \
  --storage-auto-increase-limit=500GB \
  --backup-start-time=03:00 \
  --enable-bin-log \
  --retained-backups-count=30 \
  --transaction-log-retention-days=7

# Verify
gcloud sql instances describe my-database
```

---

### 1.3 Databases & Tables

#### Creating Database

```bash
# Create database
gcloud sql databases create myapp \
  --instance=my-database \
  --charset=utf8mb4 \
  --collation=utf8mb4_unicode_ci
```

#### Creating Users

```bash
# Create user with password
gcloud sql users create appuser \
  --instance=my-database \
  --password=STRONG_PASSWORD

# Or create user without password (IAM authentication)
gcloud sql users create iam-user \
  --instance=my-database \
  --type=CLOUD_IAM_USER
```

#### Connecting to Database

**From Compute Engine (via Cloud SQL Proxy):**

```bash
# 1. SSH to instance
gcloud compute ssh my-instance --zone=us-central1-a

# 2. Install Cloud SQL Proxy
curl https://dl.google.com/cloudsql/cloud_sql_proxy.linux.amd64 -o cloud_sql_proxy
chmod +x cloud_sql_proxy

# 3. Start proxy (creates localhost connection)
./cloud_sql_proxy -instances=PROJECT_ID:us-central1:my-database=tcp:3306 &

# 4. Connect via localhost
mysql -h 127.0.0.1 -u appuser -p myapp

# Alternative: Use private IP (if VPC-native)
mysql -h 10.0.0.10 -u appuser -p myapp
```

**From Application (using connector):**

```python
# Python example using Cloud SQL Python Connector
from google.cloud.sql.connector import Connector
import os

connector = Connector()

def getconn():
    return connector.connect(
        "project_id:region:instance_name",
        "pymysql",
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASS"],
        db=os.environ["DB_NAME"],
    )

# Use connection
import sqlalchemy
engine = sqlalchemy.create_engine(
    "mysql+pymysql://",
    creator=getconn,
)
```

---

### 1.4 Networking & Connectivity

#### Public IP

```bash
# Authorize specific IPs
gcloud sql instances patch my-database \
  --authorized-networks=203.0.113.0/24

# List authorized networks
gcloud sql instances describe my-database | grep -A5 authorizedNetworks
```

**Security Concerns:**
- Exposed to internet
- Requires strong passwords
- Can be blocked by firewall

#### Private IP (VPC-Native)

**More secure option:**

```bash
# Create instance with private IP only
gcloud sql instances create my-database \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --network=my-vpc \
  --no-assign-ip  # No public IP

# Enable Private Service Connection (required first time)
# Done automatically with --network option
```

**Private IP Benefits:**
```
✓ Not exposed to internet
✓ Internal VPC communication only
✓ Lower latency (same VPC)
✓ Better security
✓ No public IP charges
```

#### Cloud SQL Proxy

Secure tunnel from app to database

```
Application → Cloud SQL Proxy → Encrypted Tunnel → Cloud SQL
```

**Advantages:**
- Doesn't require storing database IP/credentials in app
- IAM-based authentication
- Automatic encryption
- Works with both public & private IPs

---

### 1.5 Backups & Recovery

#### Automatic Backups

```bash
# Backups run daily (configurable time)
gcloud sql instances patch my-database \
  --backup-start-time=03:00 \
  --retained-backups-count=30  # Keep 30 backups
```

**Backup Properties:**
```
Frequency: Daily (automatic)
Retention: Configurable (default 7 days)
Location: Same region as instance
Cost: Included in instance cost
Restoration: Point-in-time (with transaction logs)
```

#### On-Demand Backups

```bash
# Create on-demand backup
gcloud sql backups create \
  --instance=my-database \
  --description="Before migration"

# List backups
gcloud sql backups list --instance=my-database

# Restore from backup
gcloud sql backups restore BACKUP_ID \
  --backup-instance=my-database
```

#### Point-in-Time Recovery (PITR)

Restore to any point in time (requires transaction logs)

```bash
# Enable transaction log retention
gcloud sql instances patch my-database \
  --transaction-log-retention-days=7

# Restore to specific timestamp
gcloud sql backups restore \
  --backup-instance=my-database \
  --point-in-time=2024-04-28T12:34:00Z
```

---

### 1.6 High Availability (Regional Setup)

#### Multi-Zone Failover

**How it works:**
```
Primary (us-central1-a)
    ↓ (replication)
Standby (us-central1-b)
```

```bash
# Create regional instance (multi-zone)
gcloud sql instances create ha-database \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL
```

**HA Characteristics:**
```
Automatic Failover: Yes (if primary fails)
Failover Time: ~3 minutes
Data Loss: None (synchronous replication)
Cost: Double (pay for primary + standby)
Downtime: Brief during failover
```

#### Read Replicas

Secondary databases for read scaling

```bash
# Create read replica (same region)
gcloud sql instances create my-database-read-1 \
  --master-instance-name=my-database

# Create read replica (different region)
gcloud sql instances create my-database-read-asia \
  --master-instance-name=my-database \
  --region=asia-southeast1
```

**Read Replica Characteristics:**
```
Purpose: Scale read workloads
Promotion: Can be promoted to standalone
Lag: Few seconds (asynchronous replication)
Cost: Pay per replica
Failover: Manual (not automatic)
```

---

### 1.7 Maintenance & Patching

#### Automatic Updates

```bash
# Set maintenance window
gcloud sql instances patch my-database \
  --maintenance-window-day=SUN \
  --maintenance-window-hour=03 \
  --maintenance-window-duration=4
```

**Maintenance Properties:**
```
Timing: Automatic during maintenance window
Downtime: 3-5 minutes (typically)
Backups: Automatic before maintenance
Frequency: Monthly patches
```

#### Database Flags

Customize database behavior

```bash
# Set flags
gcloud sql instances patch my-database \
  --database-flags=slow_query_log=on,long_query_time=2

# Clear flag
gcloud sql instances patch my-database \
  --clear-database-flags
```

**Common Flags:**
```
MySQL:
  slow_query_log: on/off
  long_query_time: seconds
  max_connections: connection limit

PostgreSQL:
  log_statement: all/ddl/mod/none
  shared_preload_libraries: extensions
```

---

### 1.8 Replication & Migration

#### Cloud Database Migration Service

Migrate from on-premises to Cloud SQL

```bash
# Continuous replication
gcloud database-migration connections create my-source \
  --database-engine=mysql \
  --host=on-prem-db.example.com \
  --port=3306 \
  --username=repl_user \
  --password=PASSWORD

# Test connection
gcloud database-migration connections describe my-source
```

---

### 1.9 Monitoring & Performance Insights

#### Monitoring Metrics

Cloud SQL exports metrics:
```
CPU utilization
Memory usage
Disk I/O
Network I/O
Connections
Replication lag (replicas)
Query performance
```

#### Slow Query Log

```bash
# Enable slow query log (already shown in flags section)
# View in Cloud Console → Cloud SQL → Logs
```

#### Performance Insights

Visualize slow queries and bottlenecks

```bash
# View in Cloud Console
# Cloud SQL → Instance → Insights
```

---

### 1.10 Pricing & Cost Optimization

#### Pricing Components

```
Instance: per vCPU-hour
Memory: per GB-hour
Storage: per GB/month
Backup storage: per GB/month
Network egress: per GB
Read replicas: per instance-hour
HA (standby): Double cost
```

**Example Pricing (US):**
```
db-n1-standard-2 (2 vCPU, 7.5GB RAM):
  $0.35/hour = ~$250/month

Storage (100GB SSD):
  $0.17/GB/month = $17/month

Backup (30 backups, 100GB each):
  $0.026/GB/month = $78/month

Total: ~$345/month
```

#### Cost Optimization

```
1. Right-size instance
   - Start small, scale up as needed
   - Monitor actual usage

2. Use appropriate storage type
   - SSD: faster but expensive
   - HDD: cheaper, slower

3. Shared-core for dev/test
   - f1-micro much cheaper (~$7/month)
   - Sufficient for development

4. Reserved instances (if predictable)
   - 1-year or 3-year discounts
   - 25-55% savings

5. Read replicas for scaling
   - Instead of upgrading instance tier
   - More cost-effective for read-heavy workloads

6. Archive old data
   - Move to BigQuery or Cloud Storage
   - Smaller database = lower costs
```

---

### 1.11 Cloud SQL Exam Tips

**Key Concepts:**
- Fully managed (Google handles patches, backups)
- Three database types (MySQL, PostgreSQL, SQL Server)
- Instance types (shared-core, standard, high-memory)
- High availability (regional = multi-zone failover)
- Read replicas (for read scaling, manual failover)
- Backups (automatic daily + PITR with transaction logs)
- Private IP (VPC-native, more secure)
- Cloud SQL Proxy (secure connection)
- Automated maintenance (configurable window)

**Common Scenarios:**
1. "Need ACID transactions" → Cloud SQL
2. "Existing MySQL application" → Cloud SQL
3. "Scale read workloads" → Read replicas
4. "Mission-critical database" → Regional HA setup
5. "Migrate from on-premises" → DMS + Cloud SQL
6. "Development environment" → Shared-core instance

---

## 2. Cloud Firestore (NoSQL Document)

### 2.1 Khái Niệm & Tổng Quan

**Cloud Firestore là gì?**
- Fully managed NoSQL document database
- Document-oriented (JSON-like documents)
- Real-time synchronization
- Serverless (auto-scaling)
- Strong consistency (within regions)
- Mobile-friendly

**Tại sao dùng Cloud Firestore?**
- Real-time data synchronization
- Mobile/web applications
- Document-based data model
- Rapid development
- Serverless (no infrastructure)
- Easy scalability

**Khi không dùng Cloud Firestore:**
- Structured relational data with complex joins → Cloud SQL
- Petabyte-scale analytics → BigQuery
- Time-series data → Cloud Bigtable
- Complex transactions across documents → Cloud SQL

---

### 2.2 Data Model

#### Collections & Documents

**Structure:**
```
Database (default or named)
├─ Collection: users
│   ├─ Document: user-123
│   │   ├─ name: "John Doe"
│   │   ├─ email: "john@example.com"
│   │   ├─ age: 30
│   │   ├─ address (subcollection or nested)
│   │   └─ createdAt: 2024-04-28T12:00:00Z
│   │
│   └─ Document: user-456
│       ├─ name: "Jane Smith"
│       └─ ...
│
├─ Collection: products
│   ├─ Document: product-1
│   │   ├─ name: "Laptop"
│   │   ├─ price: 999.99
│   │   └─ inventory: 50
│   │
│   └─ Document: product-2
│       └─ ...
```

**Key Terminology:**
- **Collection**: Like a table (contains documents)
- **Document**: Single record (JSON-like object)
- **Field**: Key-value pair within document
- **Subcollection**: Collection within document

#### Data Types

```python
# Python example
document = {
    "name": "John",          # String
    "age": 30,               # Number (int)
    "balance": 123.45,       # Number (float)
    "active": True,          # Boolean
    "joinedAt": Timestamp.now(),  # Timestamp
    "tags": ["vip", "premium"],   # Array
    "metadata": {            # Map (nested object)
        "lastLogin": Timestamp.now(),
        "loginCount": 5
    },
    "reference": db.collection("users").document("user-123")  # Reference
}
```

---

### 2.3 Creating & Managing Data

#### Creating Documents

```python
# Python client library
from firebase_admin import firestore

db = firestore.client()

# 1. Add with auto-generated ID
db.collection("users").add({
    "name": "John Doe",
    "email": "john@example.com",
    "age": 30
})

# 2. Set with specific ID
db.collection("users").document("user-123").set({
    "name": "Jane Smith",
    "email": "jane@example.com",
    "age": 25
})

# 3. Update existing document
db.collection("users").document("user-123").update({
    "age": 26
})

# 4. Increment/Array operations
db.collection("users").document("user-123").update({
    "age": firestore.Increment(1),  # Increment by 1
    "tags": firestore.ArrayUnion(["new-tag"]),  # Add to array
    "history": firestore.ArrayRemove(["old-entry"])  # Remove from array
})
```

#### Reading Documents

```python
# 1. Get single document
doc = db.collection("users").document("user-123").get()
if doc.exists:
    print(doc.to_dict())

# 2. List all documents in collection
docs = db.collection("users").stream()
for doc in docs:
    print(doc.id, doc.to_dict())

# 3. Query with conditions
query = db.collection("users").where("age", ">=", 30)
docs = query.stream()

# 4. Complex query
query = db.collection("users")\
    .where("age", ">=", 25)\
    .where("active", "==", True)\
    .order_by("createdAt", direction=firestore.Query.DESCENDING)\
    .limit(10)
docs = query.stream()
```

#### Deleting Documents

```python
# 1. Delete field
db.collection("users").document("user-123").update({
    "age": firestore.DELETE_FIELD
})

# 2. Delete entire document
db.collection("users").document("user-123").delete()

# 3. Delete collection (delete all docs first)
batch = db.batch()
docs = db.collection("users").stream()
for doc in docs:
    batch.delete(doc.reference)
batch.commit()
```

---

### 2.4 Querying & Indexing

#### Query Operators

```python
# Comparison operators
db.collection("users").where("age", "==", 30)    # Equals
db.collection("users").where("age", "!=", 30)    # Not equal
db.collection("users").where("age", "<", 30)     # Less than
db.collection("users").where("age", "<=", 30)    # Less or equal
db.collection("users").where("age", ">", 30)     # Greater than
db.collection("users").where("age", ">=", 30)    # Greater or equal

# Array operators
db.collection("users").where("tags", "array-contains", "vip")
db.collection("users").where("tags", "array-contains-any", ["vip", "premium"])

# In operator
db.collection("users").where("status", "in", ["active", "pending"])

# Compound queries
db.collection("users")\
    .where("age", ">", 25)\
    .where("status", "==", "active")
```

#### Composite Indexes

For complex queries, Firestore needs composite indexes

```
Query:
  where("age", ">", 25)
  .where("status", "==", "active")
  .orderBy("createdAt")

Required Index:
  Collection: users
  Fields:
    - age (ASCENDING)
    - status (ASCENDING)
    - createdAt (DESCENDING)
```

**Creating Indexes:**

Firestore can auto-suggest indexes when needed

```bash
# List indexes
gcloud firestore indexes list

# Create index manually
gcloud firestore indexes create \
  --collection=users \
  --field=age \
  --field=status \
  --field=createdAt
```

---

### 2.5 Real-Time Synchronization

One of Firestore's killer features

```python
# Listen for real-time updates
def on_snapshot(docs, changes, read_time):
    for doc in docs:
        print(f"Document: {doc.id}")
        print(f"Data: {doc.to_dict()}")

query = db.collection("users")
listener = query.on_snapshot(on_snapshot)

# Stop listening
listener.unsubscribe()
```

**Use Cases:**
```
1. Collaborative editing (Google Docs-like)
2. Live chat applications
3. Real-time leaderboards
4. Live notifications
5. Real-time dashboards
6. Multiplayer games
```

---

### 2.6 Transactions & Batches

#### Transactions

ACID guarantees for multiple operations

```python
from google.cloud import firestore

def transfer_money(from_user, to_user, amount):
    db = firestore.client()
    
    transaction = db.transaction()
    
    @firestore.transactional
    def transfer_in_transaction(transaction):
        # Read
        from_doc = db.collection("users").document(from_user).get()
        to_doc = db.collection("users").document(to_user).get()
        
        from_balance = from_doc.get("balance")
        to_balance = to_doc.get("balance")
        
        # Check
        if from_balance < amount:
            raise Exception("Insufficient funds")
        
        # Update
        transaction.update(
            db.collection("users").document(from_user),
            {"balance": from_balance - amount}
        )
        transaction.update(
            db.collection("users").document(to_user),
            {"balance": to_balance + amount}
        )
    
    transfer_in_transaction(transaction)
```

**Transaction Properties:**
```
Isolation: Serializable
Atomicity: All or nothing
Max operations: 500
Max 25GB document
Automatic retry: Yes
```

#### Batch Writes

Non-transactional batch operations

```python
# Batch write (up to 500 operations)
batch = db.batch()

for i in range(10):
    batch.set(
        db.collection("users").document(f"user-{i}"),
        {"name": f"User {i}"}
    )

batch.commit()
```

---

### 2.7 Security Rules

Control who can access what data

#### Basic Rules

```
Datastore Rules (Firestore Security Rules)

match /databases/{database}/documents {
  // Allow anyone to read
  match /{document=**} {
    allow read;
  }
  
  // Allow writes only to authenticated users
  match /users/{userId} {
    allow read, write: if request.auth != null;
  }
  
  // Allow writes only by document owner
  match /users/{userId} {
    allow read, write: if request.auth.uid == userId;
  }
  
  // Allow writes with validation
  match /posts/{postId} {
    allow create: if request.resource.data.title != null
                  && request.resource.data.title.size() > 0;
    allow update: if resource.data.author == request.auth.uid;
  }
}
```

#### Advanced Rules

```
// Nested collections
match /users/{userId}/posts/{postId} {
  allow read: if request.auth.uid == userId;
  allow write: if request.auth.uid == userId
               && request.resource.data.title.size() > 0;
}

// Array checks
match /events/{eventId} {
  allow read: if request.auth.uid in resource.data.attendees;
}

// Time-based rules
match /announcements/{announcementId} {
  allow read: if resource.data.expiresAt > now;
}

// Custom claims (from auth)
match /admin/content {
  allow write: if request.auth.token.admin == true;
}
```

---

### 2.8 Monitoring & Performance

#### Quotas & Limits

```
Document size: 1MB max
Collection size: Unlimited
Field size: 1.5MB max
Document write throughput: 1 write/second
Read throughput: 10,000 reads/second
```

#### Query Performance

Firestore automatically uses indexes. For complex queries:
```
1. Add composite indexes
2. Use more specific filters
3. Limit result set
4. Use pagination
```

---

### 2.9 Pricing & Cost Optimization

#### Pricing Model

```
Reads: $0.06 per 100K reads
Writes: $0.18 per 100K writes
Deletes: $0.02 per 100K deletes
Storage: $0.18 per GB/month

Free tier: 50K reads, 20K writes, 20K deletes/day
```

**Example Cost:**
```
1M reads/day:
  $0.06 × (1M/100K) × 30 = $18/month

100K writes/day:
  $0.18 × (100K/100K) × 30 = $5.40/month

10GB storage:
  $0.18 × 10 = $1.80/month

Total: ~$25/month
```

#### Cost Optimization

```
1. Denormalize strategically
   - Avoid reads on related documents
   - Cache computed values

2. Batch operations
   - Use batch writes
   - Reduce roundtrips

3. Index selectively
   - Only needed indexes
   - Unused indexes increase write cost

4. Pagination
   - Don't fetch all documents if not needed

5. Real-time listeners
   - Disable when not needed
   - Each listener counts as reads
```

---

### 2.10 Firestore Exam Tips

**Key Concepts:**
- Document database (JSON-like)
- Collections & documents
- Real-time synchronization
- Queries & composite indexes
- Transactions (ACID, max 500 ops)
- Security rules (fine-grained access)
- Auto-scaling (serverless)
- Strong consistency
- Pay-per-operation pricing

**Common Scenarios:**
1. "Need real-time data sync" → Firestore
2. "Mobile app with offline support" → Firestore (has offline cache)
3. "Rapid development, dynamic schema" → Firestore
4. "Complex analytical queries" → Cloud SQL
5. "Petabyte-scale analytics" → BigQuery

---

## 3. Cloud Bigtable (NoSQL Wide-Column)

### 3.1 Khái Niệm & Tổng Quan

**Cloud Bigtable là gì?**
- Fully managed wide-column (columnar) NoSQL database
- HBase-compatible
- Designed for petabyte-scale data
- Extreme low latency, high throughput
- Pay per node (not serverless)
- Time-series, analytics, IoT data

**Tại sao dùng Cloud Bigtable?**
- Time-series data (IoT, metrics, logs)
- Massive throughput (millions of ops/sec)
- Petabyte scale
- Low latency (milliseconds)
- Sequential access patterns

**Khi không dùng Cloud Bigtable:**
- Small datasets → Firestore
- Complex queries → Cloud SQL
- Analytics → BigQuery
- Real-time sync → Firestore

---

### 3.2 Data Model

#### Bigtable Structure

```
Bigtable Instance
├─ Cluster (deployment unit)
│   ├─ Nodes (storage & compute)
│   └─ Replication (across zones)
│
└─ Table (data container)
    ├─ Row Key (primary key)
    │   ├─ Column Family 1
    │   │   ├─ Column 1
    │   │   ├─ Column 2
    │   │   └─ ...
    │   │
    │   └─ Column Family 2
    │       ├─ Column 1
    │       └─ ...
    │
    └─ Timestamps (versioning)
```

**Example Data:**

```
Row Key: user-123
├─ profile (column family)
│   ├─ name: "John Doe"
│   ├─ email: "john@example.com"
│   └─ age: 30
│
└─ activity (column family)
    ├─ last_login: 2024-04-28 12:00:00
    ├─ login_count: 157
    └─ last_ip: "203.0.113.1"
```

#### Key Design

Row keys should enable efficient access patterns

```
Good Row Keys:
  user-123            (prefix for user data)
  user-123#2024-04    (time-ordered user activity)
  2024-04-28#sensor-1 (time-series data)

Bad Row Keys:
  123 (too sequential, hot spots)
  2024-04-28T12:34:56 (all at once write, not distributed)
```

---

### 3.3 Creating Tables

```bash
# 1. Create Bigtable instance
gcloud bigtable instances create my-instance \
  --cluster=my-cluster \
  --cluster-zone=us-central1-b \
  --cluster-num-nodes=3

# 2. Create table
gcloud bigtable tables create my-table \
  --instance=my-instance \
  --cluster=my-cluster \
  --max-versions=3  # Keep 3 versions per cell

# 3. Create column families
gcloud bigtable column-families create cf1 \
  --instance=my-instance \
  --table=my-table \
  --max-versions=3

gcloud bigtable column-families create cf2 \
  --instance=my-instance \
  --table=my-table \
  --max-versions=1
```

---

### 3.4 Reading & Writing Data

#### Client Libraries

```python
# Python client
from google.cloud import bigtable

client = bigtable.Client(project="my-project", admin=True)
instance = client.instance("my-instance")
table = instance.table("my-table")

# Write
row = table.row(b"user-123")
row.set_cell(b"profile", b"name", b"John Doe")
row.set_cell(b"profile", b"age", b"30")
row.commit()

# Read
row = table.read_row(b"user-123")
for cf, cols in row.cells.items():
    print(f"Column family: {cf}")
    for col, cells in cols.items():
        for cell in cells:
            print(f"  {col}: {cell.value} (timestamp: {cell.timestamp})")

# Batch write
rows_to_write = [
    table.row(b"user-124"),
    table.row(b"user-125")
]
# ... add cells to rows ...
table.mutate_rows(rows_to_write)

# Scan range
rows = table.read_rows(start_key=b"user-100", end_key=b"user-200")
for row in rows:
    print(row)
```

#### Delete Operations

```python
# Delete specific cell
row = table.row(b"user-123")
row.delete(b"profile", b"temp_field")
row.commit()

# Delete entire row
table.delete_rows([b"user-123"])

# Delete column family
table.delete_column_family(b"old_data")
```

---

### 3.5 Performance & Scaling

#### Instance Configuration

```bash
# Create high-performance instance (for production)
gcloud bigtable instances create prod-instance \
  --cluster=prod-cluster \
  --cluster-zone=us-central1-b \
  --cluster-num-nodes=10  # 10 nodes for high throughput

# Scale nodes (can be done online)
gcloud bigtable clusters update prod-cluster \
  --instance=prod-instance \
  --num-nodes=20
```

**Node Count Guidance:**
```
Development: 1-3 nodes
Small production: 3-5 nodes
Medium production: 5-10 nodes
Large scale: 10-100+ nodes
```

#### Throughput Expectations

```
Per node:
  Reads: ~40,000 QPS (queries per second)
  Writes: ~10,000 QPS

Example:
  10 nodes = 400,000 reads/sec or 100,000 writes/sec
```

---

### 3.6 Replication & Disaster Recovery

#### Multi-Cluster Replication

```bash
# Create instance with 2 clusters (for HA)
gcloud bigtable instances create replicated-instance \
  --cluster=us-cluster \
  --cluster-zone=us-central1-b \
  --cluster-num-nodes=5 \
  --cluster=eu-cluster \
  --cluster-zone=europe-west1-b \
  --cluster-num-nodes=5
```

**Replication Characteristics:**
```
Synchronous: Strong consistency
Asynchronous: Higher availability, potential inconsistency
RPO (Recovery Point Objective): Minutes
RTO (Recovery Time Objective): Seconds
Cost: Double (multiple clusters)
```

---

### 3.7 Cloud Bigtable Use Cases

#### Time-Series Data

```
Sensor readings:
  Row Key: 2024-04-28T12:34:56#sensor-1
  Column Families:
    - temperature
    - humidity
    - pressure
```

#### IoT Data Collection

```
Device metrics:
  Row Key: device-123#2024-04-28
  Column Families:
    - metrics (battery, signal strength, etc.)
    - events (errors, warnings)
```

#### Analytics & Metrics

```
Application metrics:
  Row Key: 2024-04-28#app-server-1
  Column Families:
    - cpu
    - memory
    - network
```

---

### 3.8 Pricing & Cost Optimization

#### Pricing

```
Storage: $0.01 per GB/month
Nodes: $0.65 per node-hour
Replication: Double cost per additional cluster
Backups: $0.10 per GB/month
```

**Example Cost:**
```
3 nodes (basic production):
  3 × $0.65/hour × 730 hours = $1,423/month

100GB storage:
  $0.01 × 100 = $1/month

Total: ~$1,424/month
```

#### Cost Optimization

```
1. Right-size node count
   - Monitor actual usage
   - Scale based on throughput needs

2. Use appropriate column families
   - Each CF has its own cache
   - Too many CFs = wasted memory

3. Compression
   - Enable compression for storage efficiency

4. TTL (Time-to-Live)
   - Auto-delete old data
   - Reduce storage costs
```

---

### 3.9 Cloud Bigtable Exam Tips

**Key Concepts:**
- Wide-column (columnar) database
- Time-series optimized
- Petabyte-scale
- Extreme throughput (millions ops/sec)
- Row key design critical
- Column families (grouping)
- HBase API
- Multi-cluster replication
- Pay per node (not serverless)

**Common Scenarios:**
1. "Store millions of events per second" → Bigtable
2. "Time-series metrics collection" → Bigtable
3. "IoT sensor data" → Bigtable
4. "Low latency massive scale" → Bigtable
5. "Complex queries" → Cloud SQL or BigQuery

---

## 4. Cloud Spanner

### 4.1 Khái Niệm & Tổng Quan

**Cloud Spanner là gì?**
- Globally distributed relational database
- Horizontal scalability (sharding built-in)
- Strong consistency across regions (globally)
- ACID transactions across rows/tables/regions
- Pay per node
- Combines SQL + distributed systems

**Tại sao dùng Cloud Spanner?**
- Global applications needing strong consistency
- Horizontal scaling of SQL database
- Multi-region high availability
- Mission-critical applications
- Financial systems requiring ACID

**Khi không dùng Cloud Spanner:**
- Single region → Cloud SQL (cheaper)
- Non-relational → Firestore
- Massive scale analytics → BigQuery
- Simple applications → Firestore
- Cost-sensitive → Cloud SQL

---

### 4.2 Architecture

#### Spanner Cluster

```
Spanner Instance
├─ Configuration: Multi-region or Regional
│
├─ Database 1
│   └─ Tables, Indexes
│
└─ Database 2
    └─ Tables, Indexes
```

**Configuration Types:**

```
Regional (Single Region):
  ├─ Replication: 3 copies in same region
  ├─ Availability: 99.99% SLA
  └─ Use: High availability, single region

Multi-Region:
  ├─ Replication: Across multiple regions
  ├─ Availability: 99.999% SLA (five nines)
  └─ Use: Global applications, disaster recovery
```

---

### 4.3 Creating Instances & Databases

```bash
# 1. Create Spanner instance (regional)
gcloud spanner instances create my-instance \
  --config=regional-us-central1 \
  --description="Production Spanner" \
  --nodes=3

# 2. Create instance (multi-region)
gcloud spanner instances create global-instance \
  --config=nam-eur-asia1 \
  --description="Global Spanner" \
  --nodes=3

# 3. Create database
gcloud spanner databases create my-db \
  --instance=my-instance

# 4. Create tables
gcloud spanner databases ddl update my-db \
  --instance=my-instance \
  --ddl="
    CREATE TABLE users (
      id INT64 NOT NULL,
      name STRING(100),
      email STRING(100),
      created_at TIMESTAMP,
    ) PRIMARY KEY(id)
  "
```

---

### 4.4 Querying & Transactions

#### Standard SQL

```python
from google.cloud import spanner

client = spanner.Client()
instance = client.instance("my-instance")
database = instance.database("my-db")

# Query
with database.snapshot() as snapshot:
    results = snapshot.execute_sql(
        "SELECT id, name, email FROM users WHERE id = @user_id",
        params={"user_id": 123}
    )
    for row in results:
        print(row)

# Write
with database.batch() as batch:
    batch.insert(
        "users",
        columns=["id", "name", "email"],
        values=[
            [1, "John Doe", "john@example.com"],
            [2, "Jane Smith", "jane@example.com"]
        ]
    )
```

#### Transactions

Full ACID transactions across rows, tables, regions

```python
def transfer_money(database, from_id, to_id, amount):
    def update_accounts(transaction):
        # Read
        from_result = transaction.execute_sql(
            "SELECT balance FROM accounts WHERE id = @id",
            params={"id": from_id}
        )
        from_balance = next(from_result)[0]
        
        to_result = transaction.execute_sql(
            "SELECT balance FROM accounts WHERE id = @id",
            params={"id": to_id}
        )
        to_balance = next(to_result)[0]
        
        # Validate
        if from_balance < amount:
            raise ValueError("Insufficient funds")
        
        # Update
        transaction.execute_update(
            "UPDATE accounts SET balance = balance - @amount WHERE id = @id",
            params={"amount": amount, "id": from_id}
        )
        
        transaction.execute_update(
            "UPDATE accounts SET balance = balance + @amount WHERE id = @id",
            params={"amount": amount, "id": to_id}
        )
    
    database.run_in_transaction(update_accounts)
```

**Transaction Properties:**
```
Isolation Level: Serializable (strongest)
Atomicity: All or nothing
Consistency: Strong (across regions)
Durability: Replicated in real-time
Conflicts: Automatic retry
```

---

### 4.5 Global Consistency

#### How Spanner Ensures Global Consistency

```
Atomic Clocks (TrueTime):
  ├─ Actual atomic clocks in datacenters
  ├─ Leap-second handling
  └─ Bounded uncertainty

Result:
  ├─ Replicas know exact order of transactions
  ├─ No clock skew issues
  └─ Strong consistency without heavy locking
```

**Implications:**
```
Reads are always consistent (no stale data)
Writes are globally ordered
No eventual consistency complexity
Queries same regardless of region
```

---

### 4.6 Schema & Indexes

#### Creating Indexes

```bash
# Create index
gcloud spanner databases ddl update my-db \
  --instance=my-instance \
  --ddl="
    CREATE INDEX idx_users_email
    ON users(email)
  "

# Create composite index
gcloud spanner databases ddl update my-db \
  --instance=my-instance \
  --ddl="
    CREATE INDEX idx_users_created
    ON users(created_at DESC, id ASC)
  "
```

---

### 4.7 Multi-Region Strategy

#### Regional Configuration

```bash
# Option 1: Regional (single region, 99.99% SLA)
gcloud spanner instances create regional-instance \
  --config=regional-us-central1 \
  --nodes=3

# Option 2: Multi-region (99.999% SLA)
gcloud spanner instances create global-instance \
  --config=nam-eur-asia1 \
  --nodes=3
```

**Multi-Region Characteristics:**
```
Regions: North America, Europe, Asia (3 regions)
Replication: Strong consistency across regions
Failover: Automatic
Latency: 10-100ms between regions
Cost: Higher
```

---

### 4.8 Monitoring & Performance

#### Query Performance

```python
# Use query stats to optimize
database = instance.database("my-db")

# Simple query vs optimized query
# Query 1 (good): SELECT id, name FROM users WHERE id = 1
# Query 2 (bad): SELECT * FROM users WHERE name LIKE '%John%'

# Spanner metrics available in Cloud Console
# - Query latency
# - CPU utilization
# - Replication lag
```

---

### 4.9 Pricing & Cost Optimization

#### Pricing Model

```
Nodes: $3.90 per node-hour
Storage: $0.30 per GB/month
Backup: $0.10 per GB/month
```

**Example Cost:**
```
3 nodes (minimum for regional):
  3 × $3.90/hour × 730 hours = $8,541/month

10GB storage:
  $0.30 × 10 = $3/month

Total: ~$8,544/month (expensive!)
```

#### Cost Optimization

```
1. Start with 3 nodes (minimum)
   - Scale only if needed
   
2. Only scale horizontally when needed
   - Spanner is expensive
   - Consider Cloud SQL first
   
3. Use appropriate regions
   - Regional cheaper than multi-region
   - Multi-region only if needed

4. Archive old data
   - Move to BigQuery
   - Smaller database = fewer nodes?
   - Spanner cost is mostly nodes, not storage
```

---

### 4.10 Cloud Spanner Exam Tips

**Key Concepts:**
- Globally distributed SQL database
- Strong consistency across regions (TrueTime)
- Horizontal scalability (sharding)
- ACID transactions across rows/tables/regions
- Regional vs Multi-region configuration
- Pay per node (expensive)
- HPA (Horizontal Pod Autoscaling) built-in

**Common Scenarios:**
1. "Global application with strong consistency" → Spanner
2. "Financial system across regions" → Spanner
3. "Need ACID transactions globally" → Spanner
4. "Single region SQL needs" → Cloud SQL
5. "Cost-sensitive relational" → Cloud SQL

---

## So Sánh & Chọn Service

### Database Comparison Matrix

| Aspect | Cloud SQL | Firestore | Bigtable | Spanner |
|--------|-----------|-----------|----------|---------|
| **Type** | Relational SQL | Document NoSQL | Wide-column NoSQL | Distributed SQL |
| **Scale** | Single region | Continental | Petabytes | Global |
| **Consistency** | Strong (single) | Strong | Eventual | Strong (global) |
| **Transactions** | ACID | Multi-doc | Row-level | Multi-row, global |
| **Cost Model** | Per instance | Per operation | Per node | Per node |
| **Best For** | Traditional apps | Real-time, mobile | Time-series, IoT | Global, strong consistency |
| **Pricing** | $$ | $ | $$$ | $$$$ |

### Quick Decision Tree

```
Is it relational data?
├─ YES: Need global consistency?
│   ├─ YES → Cloud Spanner
│   └─ NO → Cloud SQL
└─ NO: Need real-time sync?
    ├─ YES → Firestore
    └─ NO: Time-series/IoT?
        ├─ YES → Bigtable
        └─ NO → Firestore or BigQuery (analytics)
```

### Detailed Comparison

#### Cloud SQL vs Cloud Spanner

```
Cloud SQL (Best: Single region SQL)
  ✓ Cheaper ($30-150/month)
  ✓ Fully managed (no nodes to manage)
  ✓ Simple to set up
  ✗ Limited horizontal scaling
  ✗ No global transactions

Cloud Spanner (Best: Global + strong consistency)
  ✓ Global distribution
  ✓ Strong consistency across regions
  ✓ Unlimited horizontal scaling
  ✗ Very expensive ($8k+/month minimum)
  ✗ Complex to set up
```

#### Firestore vs Bigtable

```
Firestore (Best: Real-time, mobile apps)
  ✓ Serverless (pay per operation)
  ✓ Real-time synchronization
  ✓ Offline support
  ✓ Mobile SDKs
  ✗ Limited querying
  ✗ Cost grows with scale

Bigtable (Best: Time-series, massive scale)
  ✓ Extreme throughput (millions ops/sec)
  ✓ Petabyte scale
  ✓ Low latency
  ✓ Optimized for sequential access
  ✗ Pay per node (minimum 3 nodes = $1,400/month)
  ✗ Limited query capabilities
```

---

## Thực Hành & Scenarios

### Scenario 1: E-Commerce Platform

**Requirements:**
- Product catalog (searchable, relational)
- User profiles & authentication
- Orders with ACID transactions
- Real-time inventory updates

**Architecture:**

```
Frontend
  ├─ Firestore (product catalog, real-time updates)
  ├─ Cloud SQL (user profiles, orders with ACID)
  └─ Cloud Storage (product images)
```

**Setup:**

```bash
# 1. Create Cloud SQL instance for orders/users
gcloud sql instances create ecommerce-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL

# 2. Create databases
gcloud sql databases create users_db --instance=ecommerce-db
gcloud sql databases create orders_db --instance=ecommerce-db

# 3. Create tables
# (Schema not shown for brevity)

# 4. Set up Firestore for products
# (Via Cloud Console or Firebase CLI)
```

**Usage:**

```python
# Cloud SQL - Order with ACID transaction
def place_order(sql_connection, user_id, items, firestore_db):
    cursor = sql_connection.cursor()
    cursor.execute("BEGIN TRANSACTION")
    
    try:
        # 1. Check inventory in Firestore
        for item_id, qty in items:
            doc = firestore_db.collection("inventory").document(item_id).get()
            if doc.get("quantity") < qty:
                raise ValueError(f"Insufficient stock for {item_id}")
        
        # 2. Create order in Cloud SQL (transactional)
        cursor.execute(
            "INSERT INTO orders (user_id, created_at, status) VALUES (%s, NOW(), 'pending')",
            (user_id,)
        )
        order_id = cursor.lastrowid
        
        # 3. Add items to order
        for item_id, qty, price in items:
            cursor.execute(
                "INSERT INTO order_items (order_id, item_id, qty, price) VALUES (%s, %s, %s, %s)",
                (order_id, item_id, qty, price)
            )
        
        # 4. Update inventory in Firestore (real-time)
        batch = firestore_db.batch()
        for item_id, qty in items:
            batch.update(
                firestore_db.collection("inventory").document(item_id),
                {"quantity": firestore.Increment(-qty)}
            )
        batch.commit()
        
        cursor.execute("COMMIT")
        return order_id
        
    except Exception as e:
        cursor.execute("ROLLBACK")
        raise e
```

---

### Scenario 2: IoT Metrics Collection

**Requirements:**
- Collect metrics from millions of sensors
- Store time-series data
- Query recent data (last hour, day)
- Archive old data

**Architecture:**

```
IoT Sensors
  ↓
Cloud Bigtable (recent data, millions writes/sec)
  ↓
Cloud Storage Archive (old data, via lifecycle)
```

**Setup:**

```bash
# 1. Create Bigtable instance
gcloud bigtable instances create iot-instance \
  --cluster=iot-cluster \
  --cluster-zone=us-central1-b \
  --cluster-num-nodes=5

# 2. Create table
gcloud bigtable tables create metrics \
  --instance=iot-instance \
  --max-versions=1

# 3. Create column families (by sensor type)
gcloud bigtable column-families create temperature \
  --instance=iot-instance \
  --table=metrics

gcloud bigtable column-families create humidity \
  --instance=iot-instance \
  --table=metrics
```

**Usage:**

```python
from google.cloud import bigtable
from datetime import datetime
import time

client = bigtable.Client(project="my-project")
instance = client.instance("iot-instance")
table = instance.table("metrics")

# Row key: timestamp#sensor-id (enables time-ordered queries)
def write_sensor_data(sensor_id, temperature, humidity):
    row_key = f"{datetime.now().isoformat()}#{sensor_id}".encode()
    row = table.row(row_key)
    
    row.set_cell(b"temperature", b"value", str(temperature).encode())
    row.set_cell(b"humidity", b"value", str(humidity).encode())
    
    row.commit()

# Query last hour of data for specific sensor
def get_recent_metrics(sensor_id, hours=1):
    import time
    cutoff_time = (time.time() - hours * 3600)
    cutoff_timestamp = datetime.fromtimestamp(cutoff_time).isoformat()
    
    prefix = sensor_id.encode()
    rows = table.read_rows(
        start_key=f"{cutoff_timestamp}#{sensor_id}".encode(),
        end_key=f"{datetime.now().isoformat()}#{sensor_id}\x00".encode()
    )
    
    for row in rows:
        print(f"Row: {row.key}, Data: {row.cells}")
```

---

### Scenario 3: Global Financial System

**Requirements:**
- Accounts worldwide (strong consistency)
- Transfers across accounts/regions
- ACID compliance (regulatory)
- Instant consistency globally

**Architecture:**

```
Cloud Spanner (globally distributed, strong consistency)
  ├─ Accounts table (user accounts worldwide)
  ├─ Transactions table (all transfers)
  └─ Ledger table (audit trail)
```

**Setup:**

```bash
# 1. Create Spanner instance (multi-region)
gcloud spanner instances create fintech-instance \
  --config=nam-eur-asia1 \
  --description="Global fintech platform" \
  --nodes=3

# 2. Create database
gcloud spanner databases create financial-db \
  --instance=fintech-instance

# 3. Create schema
gcloud spanner databases ddl update financial-db \
  --instance=fintech-instance \
  --ddl="
    CREATE TABLE accounts (
      account_id INT64 NOT NULL,
      owner_id STRING(100),
      balance NUMERIC,
      currency STRING(3),
      created_at TIMESTAMP,
    ) PRIMARY KEY(account_id);
    
    CREATE TABLE transactions (
      tx_id INT64 NOT NULL,
      from_account INT64,
      to_account INT64,
      amount NUMERIC,
      timestamp TIMESTAMP,
      status STRING(20),
    ) PRIMARY KEY(tx_id);
    
    CREATE INDEX idx_transactions_timestamp
    ON transactions(timestamp DESC);
  "
```

**Usage:**

```python
from google.cloud import spanner
from decimal import Decimal

client = spanner.Client()
instance = client.instance("fintech-instance")
database = instance.database("financial-db")

def transfer_money_globally(from_account_id, to_account_id, amount):
    def do_transfer(transaction):
        # Read from_account
        from_result = transaction.execute_sql(
            """
            SELECT balance FROM accounts
            WHERE account_id = @account_id
            """,
            params={"account_id": from_account_id}
        )
        from_balance = next(from_result)[0]
        
        # Validate
        if from_balance < amount:
            raise ValueError("Insufficient funds")
        
        # Debit from_account
        transaction.execute_update(
            """
            UPDATE accounts
            SET balance = balance - @amount
            WHERE account_id = @account_id
            """,
            params={"amount": amount, "account_id": from_account_id}
        )
        
        # Credit to_account
        transaction.execute_update(
            """
            UPDATE accounts
            SET balance = balance + @amount
            WHERE account_id = @account_id
            """,
            params={"amount": amount, "account_id": to_account_id}
        )
        
        # Log transaction (for audit)
        transaction.execute_update(
            """
            INSERT INTO transactions
            (tx_id, from_account, to_account, amount, timestamp, status)
            VALUES (GENERATE_UUID(), @from, @to, @amount, CURRENT_TIMESTAMP(), 'completed')
            """,
            params={
                "from": from_account_id,
                "to": to_account_id,
                "amount": amount
            }
        )
    
    # Execute with automatic retry on conflict
    database.run_in_transaction(do_transfer)
```

---

### Scenario 4: Real-Time Collaboration App

**Requirements:**
- Real-time data synchronization
- Offline support
- Mobile & web clients
- Dynamic schema (rapid development)

**Architecture:**

```
Firestore (real-time, offline-first)
  ├─ Documents (user documents)
  ├─ Presence (who's online)
  └─ Comments (real-time collaboration)
```

**Setup:**

```bash
# Set up via Firebase Console or gcloud
# Create Firestore database in native mode
gcloud firestore databases create \
  --location=nam5 \
  --database-id=default
```

**Usage:**

```python
from firebase_admin import firestore
import firebase_admin
from firebase_admin import credentials

# Initialize Firebase
cred = credentials.Certificate('service-account-key.json')
firebase_admin.initialize_app(cred)
db = firestore.client()

# Real-time document collaboration
def create_document(user_id, title):
    doc_ref = db.collection("documents").document()
    doc_ref.set({
        "title": title,
        "owner": user_id,
        "created_at": firestore.SERVER_TIMESTAMP,
        "content": "",
        "collaborators": [user_id]
    })
    return doc_ref.id

# Listen for real-time changes
def watch_document(doc_id, callback):
    doc_ref = db.collection("documents").document(doc_id)
    doc_ref.on_snapshot(callback)

# Update with conflict-free merging
def update_content(doc_id, content):
    db.collection("documents").document(doc_id).update({
        "content": content,
        "updated_at": firestore.SERVER_TIMESTAMP
    })

# Query active documents
def get_user_documents(user_id):
    docs = db.collection("documents")\
        .where("collaborators", "array-contains", user_id)\
        .order_by("updated_at", direction=firestore.Query.DESCENDING)\
        .limit(20)
    
    return [doc.to_dict() for doc in docs.stream()]
```

---

## Summary Checklist

### Cloud SQL
- [ ] Understand instance types (shared-core, standard, high-memory)
- [ ] Know database engines (MySQL, PostgreSQL, SQL Server)
- [ ] Understand backups & recovery (automatic daily + PITR)
- [ ] Know HA setup (regional = multi-zone failover)
- [ ] Read replicas (for read scaling, cross-region)
- [ ] Private IP vs Public IP (VPC-native)
- [ ] Cloud SQL Proxy (secure connections)
- [ ] Database flags (configuration)
- [ ] Pricing model & cost optimization
- [ ] Migration (from on-premises)

### Cloud Firestore
- [ ] Collections & documents (JSON-like)
- [ ] Queries & composite indexes
- [ ] Real-time synchronization
- [ ] Transactions (ACID, max 500 ops)
- [ ] Security rules (fine-grained access)
- [ ] Offline support (mobile)
- [ ] Pricing (per operation)
- [ ] Firestore vs Datastore modes
- [ ] Subcollections
- [ ] Cost optimization (denormalization)

### Cloud Bigtable
- [ ] Wide-column database model
- [ ] Time-series optimization
- [ ] Row key design (critical)
- [ ] Column families
- [ ] Extreme throughput & scale
- [ ] Multi-cluster replication
- [ ] HBase API compatibility
- [ ] TTL (time-to-live)
- [ ] Pricing (per node minimum)
- [ ] When to use (time-series, IoT, massive scale)

### Cloud Spanner
- [ ] Globally distributed SQL
- [ ] Strong consistency across regions (TrueTime)
- [ ] Horizontal scaling (sharding)
- [ ] ACID transactions (global)
- [ ] Regional vs Multi-region
- [ ] Pricing (very expensive, per node)
- [ ] When to use (global + strong consistency)
- [ ] When NOT to use (Cloud SQL cheaper for single region)

---

**End of Database Services Guide**
