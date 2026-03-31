# Storage Services - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về 3 storage services chính của GCP, biết khi nào sử dụng, cấu hình, best practices, và pricing.

---

## Mục Lục

1. [Cloud Storage (Object Storage)](#1-cloud-storage-object-storage)
2. [Cloud Filestore (Managed NFS)](#2-cloud-filestore-managed-nfs)
3. [Persistent Disks](#3-persistent-disks)
4. [So Sánh & Chọn Service](#so-sánh--chọn-service)
5. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Cloud Storage (Object Storage)

### 1.1 Khái Niệm & Tổng Quan

**Cloud Storage là gì?**
- Object Storage service (giống S3 trên AWS)
- Lưu trữ dữ liệu không có cấu trúc: files, documents, images, videos, backups
- Highly available & durable (99.999999999% durability = 11 nines)
- No file size limit (từ bytes đến exabytes)
- Accessible via HTTP/HTTPS, API, gsutil CLI

**Tại sao dùng Cloud Storage?**
- Lưu trữ media files, backups
- Data lake cho analytics
- Static website hosting
- Application artifacts
- Log storage & archival
- Cost-effective long-term storage

**Khi không dùng Cloud Storage:**
- Need file-like access (NFS) → Cloud Filestore
- Need block storage cho instances → Persistent Disks
- Need database → Cloud SQL, Firestore

---

### 1.2 Buckets & Objects

#### Buckets

**Definition:**
- Container cho objects (like a folder)
- Global namespace: Names phải unique **across all GCP** (rất important)
- Không thể rename, chỉ có thể delete & recreate

**Creating Buckets:**

```bash
# Create bucket
gsutil mb -l us-central1 -c STANDARD gs://my-unique-bucket-name/

# Or using gcloud
gcloud storage buckets create gs://my-unique-bucket-name \
  --location=us-central1 \
  --default-storage-class=STANDARD
```

**Naming Rules:**
- 3-63 characters
- Chỉ lowercase letters, numbers, hyphens, underscores, periods
- Cannot start/end with hyphen
- Cannot be IP address format (e.g., 192.168.1.1)

**Bucket Properties:**
- Location: Region hoặc multi-region (không thể change sau tạo)
- Storage class: Standard, Nearline, Coldline, Archive
- Default encryption: Google-managed hoặc customer-managed
- Versioning: Enable/disable
- Logging: Log access to bucket
- CORS: Cross-Origin Resource Sharing

```bash
# List buckets
gsutil ls

# Get bucket info
gsutil bucketinfo gs://my-bucket/
```

#### Objects

**Definition:**
- Files bên trong bucket
- Unique key (name) trong bucket
- Metadata: Timestamps, content-type, custom metadata

**Naming:**
- Không có thực sự "folders" (tuy nhiên có thể dùng "/" để simulate)
- gs://bucket-name/path/to/file.txt
- Nên dùng meaningful names (sắp xếp tốt hơn với timestamps, hashes)

**Uploading Objects:**

```bash
# Upload single file
gsutil cp file.txt gs://my-bucket/

# Upload with directory structure
gsutil cp file.txt gs://my-bucket/folder/subfolder/

# Upload directory (recursive)
gsutil -m cp -r ./local-dir gs://my-bucket/

# Upload with options
gsutil -h "Cache-Control:public, max-age=3600" \
  cp index.html gs://my-bucket/
```

**Downloading Objects:**

```bash
# Download
gsutil cp gs://my-bucket/file.txt ./

# Download directory
gsutil -m cp -r gs://my-bucket/folder ./

# Streaming download
gsutil cat gs://my-bucket/file.txt
```

**Deleting Objects:**

```bash
# Delete object
gsutil rm gs://my-bucket/file.txt

# Delete multiple objects (wildcard)
gsutil -m rm gs://my-bucket/folder/**

# Delete bucket (must be empty)
gsutil rb gs://my-bucket/
```

---

### 1.3 Storage Classes

**Core Concept:**
Storage class = balance giữa cost, availability, retrieval time

#### Standard

**Characteristics:**
- Designed for: Frequently accessed data
- Availability: 99.95% SLA (monthly uptime)
- Retrieval cost: None
- Minimum storage duration: None
- Access latency: Milliseconds
- Throughput: High

**Pricing:**
- Storage: $0.020/GB/month (highest)
- Retrieval: Free

**Best for:**
- Active datasets
- Web content serving
- Data backups
- Development/testing

**Example:**
```bash
gcloud storage buckets create gs://my-bucket \
  --default-storage-class=STANDARD
```

#### Nearline

**Characteristics:**
- Designed for: Infrequently accessed data (once per month)
- Availability: 99.95% SLA
- Retrieval cost: $0.01 per GB
- Minimum storage duration: 30 days
- Access latency: Milliseconds
- Throughput: High

**Pricing:**
- Storage: $0.010/GB/month (50% cheaper than Standard)
- Retrieval: $0.01/GB

**Best for:**
- Monthly backups
- Archive data accessed occasionally
- Data lake backups
- Cost-effective storage with occasional access

**When to use:**
If you access data ≤ 1 time per month, Nearline is cheaper overall

```
Example: 1TB monthly access
Standard: (30 days × 0.020) = $0.60 + Free retrieval = $0.60
Nearline: (30 days × 0.010) + ($0.01 × 1) = $0.30 + $0.01 = $0.31
```

**Important:** Minimum 30-day storage. Deleting before 30 days incurs early deletion charge.

#### Coldline

**Characteristics:**
- Designed for: Rarely accessed data (once per quarter)
- Availability: 99.95% SLA
- Retrieval cost: $0.05 per GB
- Minimum storage duration: 90 days
- Access latency: Milliseconds
- Throughput: High

**Pricing:**
- Storage: $0.004/GB/month (very cheap)
- Retrieval: $0.05/GB

**Best for:**
- Quarterly backups
- Disaster recovery data
- Compliance/archival data
- Long-term storage with rare access

**When to use:**
If you access data ≤ 4 times per year

```
Example: 1TB yearly access (1x per quarter)
Standard: $0.20/month × 12 = $2.40/year
Coldline: $0.004/month × 12 + ($0.05 × 4) = $0.048 + $0.20 = $0.248/year
```

#### Archive

**Characteristics:**
- Designed for: Long-term archival (rarely accessed)
- Availability: 99.95% SLA
- Retrieval cost: $0.12 per GB
- Minimum storage duration: 365 days
- Access latency: Milliseconds (but slower restore)
- Throughput: Standard

**Pricing:**
- Storage: $0.0012/GB/month (cheapest)
- Retrieval: $0.12/GB

**Best for:**
- Compliance archives (7-10 year retention)
- Disaster recovery (1 year+)
- Regulatory requirements

**When to use:**
If you store data > 1 year and access very rarely

```
Example: 1TB stored for 3 years, accessed 1x per year
Standard: $0.20/month × 36 = $7.20
Archive: $0.0012/month × 36 + ($0.12 × 3) = $0.043 + $0.36 = $0.403
```

#### Comparison Table

| Aspect | Standard | Nearline | Coldline | Archive |
|--------|----------|----------|----------|---------|
| Monthly Cost (per GB) | $0.020 | $0.010 | $0.004 | $0.0012 |
| Retrieval Cost | Free | $0.01/GB | $0.05/GB | $0.12/GB |
| Min Storage Duration | None | 30 days | 90 days | 365 days |
| Suitable Frequency | Monthly+ | Monthly | Quarterly | Yearly+ |
| SLA | 99.95% | 99.95% | 99.95% | 99.95% |

---

### 1.4 Lifecycle Management

**Purpose:**
Automatically manage objects based on age, storage class, or other criteria

#### Lifecycle Rules

```yaml
# lifecycle.yaml
lifecycle:
  rule:
    # Rule 1: Archive old backups
    - action:
        type: SetStorageClass
        storageClass: ARCHIVE
      condition:
        age: 365  # After 365 days
        
    # Rule 2: Delete very old backups
    - action:
        type: Delete
      condition:
        age: 2555  # After 7 years
        
    # Rule 3: Transition to Coldline after 3 months
    - action:
        type: SetStorageClass
        storageClass: COLDLINE
      condition:
        age: 90
```

**Conditions (Matching Rules):**
- `age`: Days since object created
- `createdBefore`: Specific date
- `matchesStorageClass`: Current storage class
- `numNewerVersions`: For versioned objects
- `isLive`: For versioned objects (live vs non-current)
- `matchesPrefix`: Object name prefix

```bash
# Set lifecycle policy
gcloud storage buckets update gs://my-bucket \
  --lifecycle-config=lifecycle.yaml
```

**Common Patterns:**

```yaml
# Pattern 1: Auto-archive 1-year backups
lifecycle:
  rule:
    - action:
        type: SetStorageClass
        storageClass: ARCHIVE
      condition:
        age: 365

# Pattern 2: Delete temporary files after 7 days
lifecycle:
  rule:
    - action:
        type: Delete
      condition:
        age: 7
        matchesPrefix: ['temp/', 'cache/']

# Pattern 3: Tiered storage strategy
lifecycle:
  rule:
    # 30 days: Stay in Standard
    - action:
        type: SetStorageClass
        storageClass: NEARLINE
      condition:
        age: 30
    # 90 days: Move to Coldline
    - action:
        type: SetStorageClass
        storageClass: COLDLINE
      condition:
        age: 90
    # 365 days: Move to Archive
    - action:
        type: SetStorageClass
        storageClass: ARCHIVE
      condition:
        age: 365
    # 2555 days: Delete
    - action:
        type: Delete
      condition:
        age: 2555
```

**Important Notes:**
- Transitions must follow hierarchy: Standard → Nearline → Coldline → Archive
- Cannot transition backwards (Archive → Coldline not allowed)
- Processing time: Up to 24 hours
- Lifecycle actions processed asynchronously

---

### 1.5 Versioning

**Purpose:**
Keep multiple versions of objects; restore previous versions

#### Enabling Versioning

```bash
gcloud storage buckets update gs://my-bucket \
  --versioning=enabled
```

#### How Versioning Works

```
File: document.txt

Day 1: Upload v1 (version ID: 1000)
Day 2: Upload v2 (version ID: 1001)
Day 3: Upload v3 (version ID: 1002)

List current: document.txt (v3)
List all versions: 
  - document.txt (v1)
  - document.txt (v2)
  - document.txt (v3)
```

#### Operations with Versioning

```bash
# List all versions
gsutil ls -A gs://my-bucket/document.txt

# Download specific version
gsutil cp gs://my-bucket/document.txt#1001 ./document-v2.txt

# Delete current version (keeps history)
gsutil rm gs://my-bucket/document.txt

# Delete specific version
gsutil rm gs://my-bucket/document.txt#1001

# Restore previous version (re-upload)
gsutil cp gs://my-bucket/document.txt#1001 gs://my-bucket/document.txt
```

**Implications:**
- Storage increases (every version takes space)
- Versioning enables accidental deletion recovery
- Useful for: Compliance, audit trails, rollback capability

**Cost:** Each version is counted separately in storage costs

---

### 1.6 Access Control

#### IAM (Identity & Access Management)

**Recommended Approach:** Use IAM roles

**Bucket-Level Roles:**
- `roles/storage.admin`: Full control
- `roles/storage.objectAdmin`: Manage objects
- `roles/storage.objectViewer`: View objects
- `roles/storage.objectCreator`: Create objects
- `roles/storage.legacyBucketOwner`: Legacy (don't use)

```bash
# Grant role to user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/storage.objectViewer

# Grant role to service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-app@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectAdmin
```

#### Signed URLs (Temporary Access)

**Purpose:**
Allow temporary access without credentials

```bash
# Generate signed URL (valid for 1 hour)
gsutil signurl -d 1h /path/to/service-account-key.json gs://my-bucket/file.txt

# In code (Python)
from google.cloud import storage
from google.auth.transport import requests

def generate_signed_url(bucket_name, blob_name, version_id=None):
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name, generation=version_id)
    
    url = blob.generate_signed_url(
        version_id=version_id,
        expiration=datetime.timedelta(hours=1),
        method="GET"
    )
    return url
```

**Use Cases:**
- Share files with external users
- Temporary download links
- Form submission endpoints

#### ACL (Access Control Lists) - Deprecated

**Note:** ACLs are legacy, IAM is recommended

```bash
# Grant public read access (NOT recommended)
gsutil acl ch -u AllUsers:R gs://my-bucket/public-file.txt

# Better approach: Use IAM
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

#### Uniform Bucket-Level Access (ULA)

**Purpose:**
Disable object-level ACLs, use only bucket-level IAM

```bash
# Enable ULA (recommended)
gcloud storage buckets update gs://my-bucket \
  --uniform-bucket-level-access
```

**Benefits:**
- Simpler, more consistent access control
- Easier to audit permissions
- No ACL complexity

---

### 1.7 Encryption

#### Google-Managed Encryption (Default)

- Automatic encryption at rest
- No configuration needed
- Google manages keys

```bash
# Already enabled by default
gcloud storage buckets describe gs://my-bucket | grep encryption
```

#### Customer-Managed Encryption (CMEK)

Use your own encryption keys via Cloud KMS

```bash
# Create encryption key
gcloud kms keyrings create my-keyring --location=us-central1
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption

# Grant Cloud Storage service account access to key
gcloud kms keys add-iam-policy-binding my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --member=serviceAccount:service-PROJECT_ID@gs-project-accounts.iam.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter

# Update bucket to use CMEK
gcloud storage buckets update gs://my-bucket \
  --default-encryption-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key
```

**When to use CMEK:**
- Compliance requirements
- Want to manage own keys
- Need key rotation control
- Data sovereignty

**Trade-offs:**
- More control
- More complex
- Higher cost
- Performance: Minimal impact

---

### 1.8 Logging & Monitoring

#### Access Logging

Track who accessed what, when

```bash
# Create log bucket (Cloud Storage)
gsutil mb gs://access-logs/

# Enable logging on main bucket
gcloud storage buckets update gs://my-bucket \
  --enable-object-logging \
  --log-bucket=gs://access-logs/
```

**Log Format:**
```
time_taken(ms), bytes_sent, c_ip, c_user_agent, c_referer, 
s_bucket, s_object, s_action, cs_uri_scheme, cs_host, 
cs_method, cs_bytes, cs_host_header, cs_protocol, cs_referer, 
cs_user_agent, s_status, s_error_code, s_error_message, ...
```

#### Monitoring Metrics

Cloud Storage exports metrics:
- Object count
- Total bytes
- Request counts
- Error rates

View in Cloud Console → Cloud Storage → Metrics

#### Cloud Logging Integration

Logs automatically collected

```bash
# Query logs via Cloud Logging
gcloud logging read "resource.type=gcs_bucket AND jsonPayload.bucket_name=my-bucket" \
  --limit=50
```

---

### 1.9 Signed Requests

Allow clients to perform operations (PUT, DELETE, etc.) without authentication

```python
# Generate signed request
from google.cloud import storage

def generate_signed_request(bucket_name, blob_name):
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    
    url = blob.generate_signed_url(
        version_id=None,
        expiration=datetime.timedelta(hours=1),
        method="PUT"  # Allow PUT
    )
    return url

# Client can then PUT to URL without authentication
import requests
signed_url = generate_signed_request('my-bucket', 'file.txt')
requests.put(signed_url, data=open('file.txt', 'rb'))
```

---

### 1.10 Multipart Upload (Large Files)

For large files, multipart upload is more reliable

```bash
# gsutil automatically uses multipart for large files
gsutil cp large-file.iso gs://my-bucket/

# Parallel upload (faster)
gsutil -m cp large-file.iso gs://my-bucket/
```

**Python:**
```python
from google.cloud import storage

storage_client = storage.Client()
bucket = storage_client.bucket('my-bucket')
blob = bucket.blob('large-file.iso')

blob.upload_from_filename('large-file.iso')
```

---

### 1.11 Content Delivery & Caching

#### Cloud CDN Integration

```bash
# Enable Cloud CDN for Cloud Storage
gcloud compute backend-buckets create my-cdn-backend \
  --gcs-bucket-name=my-bucket \
  --enable-cdn
```

#### Cache Control

```bash
# Set cache headers
gsutil -h "Cache-Control:public, max-age=3600" \
  cp file.html gs://my-bucket/

# Set on existing object
gsutil setmeta -h "Cache-Control:public, max-age=3600" \
  gs://my-bucket/file.html
```

---

### 1.12 Static Website Hosting

```bash
# 1. Create bucket
gsutil mb gs://my-website.com

# 2. Upload files
gsutil cp -r ./public/* gs://my-website.com/

# 3. Set main page
gcloud storage buckets update gs://my-website.com \
  --web-main-page-suffix=index.html \
  --web-error-page=404.html

# 4. Make public (via IAM)
gcloud storage buckets add-iam-policy-binding gs://my-website.com \
  --member=allUsers \
  --role=roles/storage.objectViewer

# 5. Access via: https://storage.googleapis.com/my-website.com/index.html
# Or custom domain (via CNAME)
```

---

### 1.13 Data Transfer Services

#### Offline Transfer

For very large data (petabytes), GCP offers:
- **Transfer Appliance**: Physical device shipped to you
- **Storage Transfer Service**: Parallel transfers from on-premises/S3
- **BigQuery Data Transfer**: For databases

#### Storage Transfer Service

```bash
# Transfer from S3 to Cloud Storage
gcloud transfer operations create \
  --source-bucket=s3://my-aws-bucket \
  --destination-bucket=gs://my-gcp-bucket \
  --region=us-central1
```

---

### 1.14 Data Integrity & Validation

#### CRC32C Checksum

gsutil verifies data integrity automatically

```bash
# Force verification
gsutil -D cp file.txt gs://my-bucket/
```

#### MD5

```bash
# Set custom MD5
gsutil -h "x-goog-meta-md5:abc123..." cp file.txt gs://my-bucket/
```

---

### 1.15 Pricing & Cost Optimization

#### Pricing Components

1. **Storage**: Per GB/month (varies by class)
2. **Retrieval**: Per GB (only for Nearline, Coldline, Archive)
3. **Operations**: Per 10,000 operations
4. **Network**: Data transfer (egress charges)
5. **Special**: KMS, transfers, etc.

#### Storage Pricing Breakdown

```
Standard:
  $0.020/GB/month storage
  $0 retrieval
  Total for 1TB: $20.48/month

Nearline (accessed 1x/month):
  $0.010/GB/month storage = $10.24
  $0.01/GB retrieval (1TB) = $10.24
  Total: $20.48/month (same as Standard for monthly access)

Coldline (accessed 1x/quarter):
  $0.004/GB/month = $4.10
  $0.05/GB retrieval (1TB) = $51.20
  Total per retrieval: $4.10 + $51.20 = $55.30/quarter ($18.43/month average)

Archive (accessed 1x/year):
  $0.0012/GB/month = $1.23
  $0.12/GB retrieval (1TB) = $122.88
  Total: $1.23 + $122.88 = $124.11/year
```

#### Cost Optimization Strategies

1. **Choose right storage class**
   - Analyze access patterns
   - Use lifecycle policies

2. **Lifecycle transitions**
   - Standard → Nearline → Coldline → Archive
   - Automatic aging strategy

3. **Compression**
   - Compress before uploading (gzip, bzip2)
   - Reduce storage size 50-80%

4. **Deduplication**
   - Don't store duplicate data
   - Use content-addressable storage (hash-based names)

5. **Early deletion penalties**
   - Avoid frequent deletes from Nearline (30-day minimum)
   - Plan retention periods

6. **Regional buckets**
   - Multi-region costs more
   - Use single region if possible

---

### 1.16 Cloud Storage Exam Tips

**Key Concepts:**
- Bucket naming (global uniqueness, rules)
- Storage classes (Standard, Nearline, Coldline, Archive)
- Lifecycle policies (SetStorageClass, Delete)
- Versioning & recovery
- IAM vs ACL (prefer IAM)
- Signed URLs (temporary access)
- Encryption (Google-managed vs CMEK)
- Access logging
- Multipart uploads
- Cost optimization strategies

**Common Scenarios:**
1. "Store backup, accessed quarterly" → Coldline
2. "7-year compliance archival" → Archive
3. "Allow temporary download link" → Signed URL
4. "Restrict access to service account" → IAM role
5. "Auto-transition old backups to archive" → Lifecycle policy
6. "Public static website" → Cloud Storage + IAM

---

## 2. Cloud Filestore (Managed NFS)

### 2.1 Khái Niệm & Tổng Quan

**Cloud Filestore là gì?**
- Managed Network File System (NFS)
- POSIX-compliant shared file system
- Mount từ Compute Engine, GKE, or on-premises
- Linux kernel NFS client compatible

**Tại sao dùng Cloud Filestore?**
- Applications require shared file system
- Multiple instances/pods access same files
- POSIX semantics needed
- Existing NFS-based applications

**Khi không dùng Cloud Filestore:**
- Single instance/pod → Persistent Disk
- Object storage → Cloud Storage
- Database → Cloud SQL, Firestore
- Temporary storage → Local SSD

---

### 2.2 Filestore Instances

#### Creating Instances

```bash
# Create Filestore instance
gcloud filestore instances create my-filestore \
  --tier=standard \
  --file-share=name=share1,capacity=1TB \
  --network=default \
  --region=us-central1 \
  --zone=us-central1-a
```

**Parameters:**
- `--tier`: standard, premium, enterprise (performance levels)
- `--file-share`: Share name & capacity
- `--network`: VPC network (must be in same VPC as clients)
- `--region`: Region
- `--zone`: Zone (within region)

#### Tiers (Performance Levels)

| Tier | Performance | Use Case | SLA |
|------|-------------|----------|-----|
| Standard | 64 Mbps, 500 IOPS | General file serving | 99.9% |
| Premium | 1,024 Mbps, 16,000 IOPS | Performance-critical apps | 99.95% |
| Enterprise | 1,024+ Mbps, 16,000+ IOPS | Mission-critical | 99.99% |

**Selection:**
- Standard: Development, non-critical apps
- Premium: Production, databases
- Enterprise: Mission-critical, SLA requirements

#### Capacity

Minimum/Maximum capacity by tier:
- Standard: 1TB - 10TB
- Premium: 1TB - 10TB
- Enterprise: 1TB - 10TB

```bash
# Resize (can increase only)
gcloud filestore instances update my-filestore \
  --file-share=name=share1,capacity=2TB \
  --region=us-central1
```

---

### 2.3 Mounting Filestore

#### From Compute Engine

```bash
# 1. SSH to instance
gcloud compute ssh my-instance --zone=us-central1-a

# 2. Install NFS client (if not present)
sudo apt-get update
sudo apt-get install -y nfs-common

# 3. Get Filestore IP (from gcloud output or console)
# Example: 10.0.0.2:/share1

# 4. Create mount point
sudo mkdir -p /mnt/filestore

# 5. Mount
sudo mount -t nfs 10.0.0.2:/share1 /mnt/filestore

# 6. Verify
df -h /mnt/filestore

# 7. Persistent mount (add to /etc/fstab)
echo "10.0.0.2:/share1 /mnt/filestore nfs defaults,nofail 0 0" | sudo tee -a /etc/fstab
```

#### From GKE

```yaml
# Create PersistentVolume for NFS
apiVersion: v1
kind: PersistentVolume
metadata:
  name: filestore-pv
spec:
  capacity:
    storage: 1T
  accessModes:
    - ReadWriteMany  # Important: shared access
  nfs:
    server: 10.0.0.2  # Filestore IP
    path: "/share1"

---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: filestore-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1T
  volumeName: filestore-pv

---
# Pod using Filestore
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: gcr.io/project/app:latest
    volumeMounts:
    - name: filestore
      mountPath: /data
  volumes:
  - name: filestore
    persistentVolumeClaim:
      claimName: filestore-pvc
```

---

### 2.4 Access Control

#### Network Access

Must be in same VPC network

```bash
# Filestore instance must be in same VPC as clients
gcloud filestore instances create my-filestore \
  --network=default  # Same VPC as instances
```

#### File-level Permissions

Standard Unix permissions

```bash
# Set permissions
sudo chmod 755 /mnt/filestore/
sudo chown user:group /mnt/filestore/

# Verify
ls -la /mnt/filestore/
```

#### IP Restrictions

Firestore instance has fixed IP in VPC network

```bash
# Firewall rules to restrict access (optional)
gcloud compute firewall-rules create allow-filestore \
  --allow=tcp:111,tcp:2049,tcp:20048,udp:111,udp:2049,udp:20048 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=filestore-client
```

---

### 2.5 Backups

#### Manual Backups

```bash
# Create backup
gcloud filestore backups create my-backup \
  --instance=my-filestore \
  --instance-region=us-central1

# List backups
gcloud filestore backups list

# Restore from backup
gcloud filestore instances create restored-instance \
  --tier=standard \
  --file-share=name=share1,capacity=1TB \
  --network=default \
  --backup=my-backup
```

---

### 2.6 High Availability

#### Multi-Zone Replication

Filestore automatically replicates within zone (locally)

For cross-zone HA:
- Create instances in multiple zones
- Sync data (rsync, custom scripts)
- Application-level failover

#### Snapshots

Filestore doesn't have native snapshots, use external tools:

```bash
# Manual backup via rsync
rsync -av /mnt/filestore/ /backup/filestore-backup/
```

---

### 2.7 Use Cases

**1. Shared ML Training Data**
```
Multiple instances training on same dataset
→ Store in Filestore, all read concurrently
```

**2. Content Management System**
```
Web servers serving same content
→ CMS uploads to Filestore NFS share
→ Web servers read from Filestore
```

**3. Batch Processing**
```
Batch job writes intermediate files
→ Multiple workers read same files
→ Filestore allows concurrent read/write with locking
```

**4. Application Logs**
```
Multiple servers log to shared location
→ Centralized log location via NFS
```

---

### 2.8 Limits & Considerations

**Limits:**
- Max instances per project: 50
- Max capacity per instance: 10TB (with exceptions)
- Max throughput: 1GB/s (depends on tier)
- Quota: Check per region

**Performance Considerations:**
- Network latency affects performance
- Same-zone instances have better performance
- Premium/Enterprise tiers for production

**Backup/Restore:**
- Backups take time (depends on size)
- Restore creates new instance
- No live replication to other locations

---

### 2.9 Pricing & Cost Optimization

#### Pricing

```
Standard tier:
  $0.30/GB/month (capacity provisioned)
  Minimum: 1TB × $0.30 = $300/month

Premium tier:
  $0.60/GB/month
  Minimum: 1TB × $0.60 = $600/month

Enterprise tier:
  $1.20/GB/month
  Minimum: 1TB × $1.20 = $1200/month

Snapshots:
  $0.05/GB/month
```

#### Cost Optimization

1. **Right-size capacity**
   - Don't over-provision
   - Can increase later

2. **Choose appropriate tier**
   - Standard for non-critical
   - Premium for production

3. **Consolidate sharing**
   - Fewer instances sharing = fewer Filestore instances

---

### 2.10 Cloud Filestore Exam Tips

**Key Concepts:**
- NFS shared file system
- Tiers (Standard, Premium, Enterprise)
- VPC network requirement
- Mount point configuration
- POSIX semantics
- Capacity limits (1-10TB)
- Access modes (ReadWriteMany)
- GKE integration

**Common Scenarios:**
1. "Multiple pods need shared data" → Filestore + ReadWriteMany
2. "Legacy NFS application" → Cloud Filestore
3. "Shared training data for ML" → Filestore
4. "Web servers sharing content" → Filestore

---

## 3. Persistent Disks

### 3.1 Khái Niệm & Tổng Quan

**Persistent Disks là gì?**
- Network-attached block storage
- Independent from Compute Engine instances
- Can attach/detach while instance running
- Regional or zonal resource
- Automatically encrypted

**Tại sao dùng Persistent Disks?**
- Additional storage beyond boot disk
- Single-instance block storage
- Database files, application data
- Snapshots for backups
- Persistent data across stop/restart

**Khi không dùng Persistent Disks:**
- Multiple instances sharing → Cloud Filestore
- Object storage → Cloud Storage
- Very high performance → Local SSD
- Temporary data → Local SSD

---

### 3.2 Disk Types

#### PD-Standard

**Characteristics:**
- Network-attached HDD
- Throughput-optimized
- Performance: ~60 IOPS, ~6 MB/s per GB
- Latency: ~20ms typical

**When to use:**
- Throughput-intensive workloads
- Large sequential reads/writes
- Cost-sensitive deployments
- Non-critical systems

**Pricing:** $0.040/GB/month

**Example:**
100GB Standard PD:
- Performance: ~6,000 IOPS, ~600 MB/s
- Cost: $4/month

#### PD-Balanced (Recommended)

**Characteristics:**
- Network-attached SSD (hybrid)
- Balanced between price & performance
- IOPS & throughput scale with size
- Variable performance based on capacity

**Typical Performance:**
- 100GB: ~1,500 IOPS, ~150 MB/s
- 500GB: ~7,500 IOPS, ~750 MB/s
- 1TB: ~15,000 IOPS, ~1,500 MB/s

**When to use:**
- Production workloads (recommended)
- Databases (non-enterprise)
- Most applications
- Good balance of cost & performance

**Pricing:** $0.170/GB/month

#### PD-SSD

**Characteristics:**
- Network-attached SSD
- Highest performance (consistent)
- IOPS: ~3-30 IOPS per GB (up to 100k total)
- Throughput: ~24 MB/s per GB (up to 1.5 GB/s total)
- Latency: Sub-millisecond

**When to use:**
- High-performance databases (MySQL, PostgreSQL)
- Transactional systems
- Real-time analytics
- When performance is critical

**Pricing:** $0.340/GB/month (most expensive)

#### Comparison

| Type | IOPS/GB | Throughput | Price | Use Case |
|------|---------|-----------|-------|----------|
| Standard | 60 | 6 MB/s | $0.040 | Throughput, cost-sensitive |
| Balanced | Variable | Variable | $0.170 | General purpose (recommended) |
| SSD | ~30 | 24 MB/s | $0.340 | High-performance databases |

---

### 3.3 Zonal vs Regional Persistent Disks

#### Zonal Persistent Disks

```
Single Zone (e.g., us-central1-a)
├─ Replication: Local replication within zone
├─ Availability: 99.95% SLA
├─ Failure handling: Automatic recovery within zone
└─ Use case: Single-zone deployments
```

**Creating:**
```bash
gcloud compute disks create my-disk \
  --size=100GB \
  --type=pd-ssd \
  --zone=us-central1-a
```

#### Regional Persistent Disks

```
Two Zones (e.g., us-central1-a & us-central1-b)
├─ Replication: 2 copies across zones
├─ Availability: 99.99% SLA (higher)
├─ Failure handling: Automatic failover
└─ Use case: High-availability apps
```

**Creating:**
```bash
gcloud compute disks create my-regional-disk \
  --type=pd-ssd \
  --region=us-central1 \
  --replica-zones=us-central1-a,us-central1-b \
  --size=100GB
```

**Regional Disks Benefits:**
- Survive single-zone failure
- Automatic failover
- Better for production critical systems

**Regional Disks Trade-offs:**
- Slightly higher cost (2 copies)
- Cannot use with all instance types
- Zone affinity rules apply

---

### 3.4 Creating & Attaching Disks

#### Create Disk

```bash
# Create disk
gcloud compute disks create my-disk \
  --size=100GB \
  --type=pd-balanced \
  --zone=us-central1-a

# List disks
gcloud compute disks list --zones=us-central1-a
```

#### Attach to Instance

```bash
# Attach disk to instance
gcloud compute instances attach-disk my-instance \
  --disk=my-disk \
  --zone=us-central1-a

# Verify attachment
gcloud compute instances describe my-instance --zone=us-central1-a | grep -A5 disks
```

#### Format & Mount

```bash
# SSH to instance
gcloud compute ssh my-instance --zone=us-central1-a

# List devices
lsblk

# Format disk (creates filesystem)
sudo mkfs.ext4 /dev/sdb

# Create mount point
sudo mkdir -p /mnt/data

# Mount disk
sudo mount /dev/sdb /mnt/data

# Verify
df -h /mnt/data

# Persistent mount (survives reboot)
echo "/dev/sdb /mnt/data ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab
```

#### Detach Disk

```bash
# Unmount (from instance)
sudo umount /mnt/data

# Detach from instance
gcloud compute instances detach-disk my-instance \
  --disk=my-disk \
  --zone=us-central1-a

# Verify
gcloud compute disks list
```

---

### 3.5 Resizing Disks

#### Increase Size

```bash
# Resize disk (online, instance can be running)
gcloud compute disks resize my-disk \
  --size=200GB \
  --zone=us-central1-a

# SSH to instance, resize filesystem
gcloud compute ssh my-instance --zone=us-central1-a
sudo resize2fs /dev/sdb

# Verify new size
df -h /mnt/data
```

**Important:** Cannot decrease size (online)

#### Create Disk from Snapshot

When you need to increase size permanently:
```bash
# Snapshot current disk
gcloud compute disks snapshot my-disk \
  --snapshot-names=my-snapshot

# Create larger disk from snapshot
gcloud compute disks create my-disk-v2 \
  --source-snapshot=my-snapshot \
  --size=200GB \
  --zone=us-central1-a
```

---

### 3.6 Snapshots

#### Creating Snapshots

```bash
# Snapshot disk
gcloud compute disks snapshot my-disk \
  --snapshot-names=my-snapshot-20240428

# List snapshots
gcloud compute snapshots list

# Describe snapshot
gcloud compute snapshots describe my-snapshot-20240428
```

**Snapshot Characteristics:**
- Incremental: Only changes since last snapshot
- Global resource: Can use in any region
- Point-in-time: Exact state at snapshot time
- Cost: Charged per GB stored

#### Creating Disks from Snapshots

```bash
# Create disk from snapshot
gcloud compute disks create restored-disk \
  --source-snapshot=my-snapshot-20240428 \
  --zone=us-central1-a

# Attach to instance
gcloud compute instances attach-disk my-instance \
  --disk=restored-disk \
  --zone=us-central1-a
```

#### Automating Snapshots

```bash
# Snapshot schedule
gcloud compute resource-policies create-vm-snapshot-schedule my-policy \
  --vm-names=my-instance \
  --daily-schedule \
  --start-time=03:00 \
  --location=us-central1-a
```

#### Snapshot Scope

```
Zonal Snapshot
  ├─ Can create disk in same zone
  └─ Can create disk in different zone (in same region)

Regional Snapshot
  ├─ Can create disk in any zone in region
  └─ Can create disk in any zone in any region
```

---

### 3.7 Disk Images (Advanced)

#### Creating Custom Images from Disks

```bash
# Create image from disk
gcloud compute images create my-image \
  --source-disk=my-disk \
  --source-disk-zone=us-central1-a

# Create instance from image
gcloud compute instances create new-instance \
  --image=my-image \
  --zone=us-central1-a
```

#### Use Cases:
- Golden image with pre-installed software
- Distribute configuration to multiple instances
- Faster instance creation

---

### 3.8 Encryption

#### Google-Managed Encryption (Default)

Automatic encryption at rest, transparent to user

```bash
# Already enabled
gcloud compute disks describe my-disk | grep encryption
```

#### Customer-Managed Encryption (CMEK)

```bash
# Create KMS key (same as Cloud Storage)
gcloud kms keyrings create my-keyring --location=us-central1
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption

# Create disk with CMEK
gcloud compute disks create my-disk \
  --type=pd-ssd \
  --size=100GB \
  --kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key \
  --zone=us-central1-a
```

---

### 3.9 Performance Metrics

#### Monitoring Disk Performance

```bash
# Check current performance (CPU, I/O)
gcloud compute instances describe my-instance --zone=us-central1-a | grep -A10 serviceAccounts
```

#### Expected Performance by Disk Type

**PD-Standard (100GB):**
- Read/Write IOPS: ~6,000
- Read/Write throughput: ~600 MB/s

**PD-Balanced (100GB):**
- Read/Write IOPS: ~1,500
- Read/Write throughput: ~150 MB/s

**PD-SSD (100GB):**
- Read/Write IOPS: ~3,000
- Read/Write throughput: ~2.4 GB/s

---

### 3.10 GKE Integration

#### StorageClass for Persistent Disks

```yaml
# Define StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true

---
# Use in PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi

---
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: app
    image: gcr.io/project/app:latest
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

---

### 3.11 Pricing & Cost Optimization

#### Pricing

```
PD-Standard:
  Storage: $0.040/GB/month
  100GB: $4/month
  1TB: $40/month

PD-Balanced:
  Storage: $0.170/GB/month
  100GB: $17/month
  1TB: $170/month

PD-SSD:
  Storage: $0.340/GB/month
  100GB: $34/month
  1TB: $340/month

Snapshots:
  $0.05/GB/month
```

#### Cost Optimization

1. **Choose right disk type**
   - Standard for sequential access
   - Balanced for general purpose
   - SSD for transactional

2. **Right-size capacity**
   - Start small, grow as needed
   - Monitor actual usage

3. **Snapshot management**
   - Delete old snapshots
   - Lifecycle policies for snapshots

4. **Regional disks vs Zonal**
   - Zonal cheaper (~10-15% less)
   - Regional for HA critical systems

---

### 3.12 Persistent Disks Exam Tips

**Key Concepts:**
- Block storage (not object)
- Types: Standard, Balanced, SSD
- Zonal vs Regional
- Snapshots (incremental, global)
- Attach/detach (no restart needed)
- Resize (increase only)
- Encryption (Google-managed & CMEK)
- Images (from disks)
- GKE integration (StorageClass, PVC)

**Common Scenarios:**
1. "Database needs fast storage" → PD-SSD
2. "Cost-effective general purpose" → PD-Balanced
3. "Backup data" → Snapshots
4. "Disaster recovery" → Regional PD
5. "High-availability GKE app" → Regional PD + StorageClass

---

## So Sánh & Chọn Service

### Quick Decision Matrix

| Requirement | Cloud Storage | Filestore | Persistent Disk |
|---|---|---|---|
| **Data Type** | Objects (files) | Shared files | Block storage |
| **Access Pattern** | HTTP, API | POSIX NFS | Block device |
| **Multi-instance** | No (per-instance) | Yes (shared) | No (single instance) |
| **Scaling** | Unlimited | 1-10TB | Theoretically unlimited |
| **Use Case** | Backups, media, archival | Shared data, NFS apps | Database, single instance |
| **Cost** | Cheap | Medium | Medium-High |
| **Consistency** | Eventual | Strong | Strong |

### Detailed Comparison

#### Cloud Storage vs Persistent Disk

```
Cloud Storage:
  ✓ Object storage (unstructured)
  ✓ Unlimited capacity
  ✓ Cheap for large volumes
  ✓ Multi-region replication
  ✓ Versioning & lifecycle policies
  ✗ No block device semantics
  ✗ Higher latency
  → Best for: Backups, archives, media files

Persistent Disk:
  ✓ Block storage (structured)
  ✓ POSIX semantics
  ✓ Low latency
  ✓ Snapshots for backups
  ✗ Limited to instance
  ✗ More expensive
  → Best for: Databases, application data
```

#### Cloud Storage vs Filestore

```
Cloud Storage:
  ✓ Object-based (any format)
  ✓ Cheapest at scale
  ✓ Accessed via HTTP/API
  ✗ No concurrent writes

Filestore:
  ✓ Shared between instances
  ✓ POSIX semantics
  ✓ Concurrent read/write
  ✗ Single VPC network
  ✗ More expensive
  → Best for: Shared datasets, NFS apps
```

#### Filestore vs Persistent Disk

```
Filestore:
  ✓ Multiple instances access simultaneously
  ✓ NFS protocol
  ✓ ReadWriteMany access mode
  ✗ Network-dependent performance
  ✗ Fixed capacity

Persistent Disk:
  ✓ Attached to single instance
  ✓ Block device (lower latency)
  ✓ Can be resized online
  ✗ Single instance only
  → Best for: Single instance with high I/O
```

---

### Scenario-Based Selection

**"I need to store backups"**
→ Cloud Storage (cheapest long-term)

**"I need database for web app"**
→ Persistent Disk (block storage, attached to instance)

**"Multiple containers need same data"**
→ Filestore (shared NFS)

**"I need static media files"**
→ Cloud Storage (with Cloud CDN)

**"High-performance database"**
→ Persistent Disk PD-SSD (attached to single instance)

**"Archive compliance data (7 years)"**
→ Cloud Storage Archive class

**"Shared training data for ML"**
→ Cloud Filestore or Cloud Storage

---

## Thực Hành & Scenarios

### Scenario 1: Web Server with Static Files

**Requirement:**
- Store images, CSS, JS
- Serve to many users worldwide
- Cost-effective

**Solution: Cloud Storage + Cloud CDN**

```bash
# 1. Create bucket
gsutil mb -l us-central1 gs://my-static-assets

# 2. Upload files
gsutil -m cp -r ./public/* gs://my-static-assets/

# 3. Create Cloud CDN backend
gcloud compute backend-buckets create static-backend \
  --gcs-bucket-name=my-static-assets \
  --enable-cdn

# 4. Create load balancer pointing to backend
gcloud compute url-maps create static-map \
  --default-service=static-backend

# 5. Make public
gcloud storage buckets add-iam-policy-binding gs://my-static-assets \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

---

### Scenario 2: Database Server with Snapshots

**Requirement:**
- Production MySQL database
- High performance (fast reads/writes)
- Daily backups
- Disaster recovery

**Solution: Compute Engine + Persistent Disk (PD-SSD) + Snapshots**

```bash
# 1. Create instance
gcloud compute instances create db-server \
  --machine-type=n1-standard-4 \
  --image-family=ubuntu-2204-lts \
  --zone=us-central1-a \
  --boot-disk-type=pd-ssd \
  --boot-disk-size=50GB

# 2. Create and attach data disk (PD-SSD for performance)
gcloud compute disks create db-data \
  --size=500GB \
  --type=pd-ssd \
  --zone=us-central1-a

gcloud compute instances attach-disk db-server \
  --disk=db-data \
  --zone=us-central1-a

# 3. SSH and mount
gcloud compute ssh db-server --zone=us-central1-a
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /data
sudo mount /dev/sdb /data
echo "/dev/sdb /data ext4 defaults,nofail 0 0" | sudo tee -a /etc/fstab

# 4. Install MySQL
sudo apt-get update
sudo apt-get install -y mysql-server
# Configure to use /data directory

# 5. Set up automated snapshots
gcloud compute resource-policies create-vm-snapshot-schedule db-backup \
  --vm-names=db-server \
  --daily-schedule \
  --start-time=02:00 \
  --location=us-central1-a \
  --retention-days=30
```

---

### Scenario 3: Shared Training Data for ML

**Requirement:**
- Multiple training instances access same data
- Fast reads (avoid network bottleneck)
- High availability

**Solution: Cloud Filestore + Multiple Compute Instances**

```bash
# 1. Create Filestore instance
gcloud filestore instances create training-data \
  --tier=premium \
  --file-share=name=train,capacity=2TB \
  --network=default \
  --region=us-central1 \
  --zone=us-central1-a

# 2. Note Filestore IP (e.g., 10.0.0.2)

# 3. Create 3 training instances
for i in {1..3}; do
  gcloud compute instances create train-$i \
    --machine-type=n1-standard-8 \
    --image-family=ubuntu-2204-lts \
    --zone=us-central1-a
done

# 4. Mount Filestore on each instance
# (SSH to each and run:)
sudo mkdir -p /data
sudo mount -t nfs 10.0.0.2:/train /data
echo "10.0.0.2:/train /data nfs defaults,nofail 0 0" | sudo tee -a /etc/fstab

# 5. All instances now share /data
# Training scripts read from /data
```

---

### Scenario 4: Archive Old Backups

**Requirement:**
- 10TB daily backups
- Access last 30 days (hot)
- Keep 7 years for compliance
- Minimize cost

**Solution: Cloud Storage + Lifecycle Policies**

```bash
# 1. Create bucket
gsutil mb -l us-central1 gs://backup-archive

# 2. Create lifecycle policy
cat > lifecycle.yaml << EOF
lifecycle:
  rule:
    # 30 days: Keep in Standard
    - action:
        type: SetStorageClass
        storageClass: NEARLINE
      condition:
        age: 30
    # 90 days: Move to Coldline
    - action:
        type: SetStorageClass
        storageClass: COLDLINE
      condition:
        age: 90
    # 1 year: Move to Archive
    - action:
        type: SetStorageClass
        storageClass: ARCHIVE
      condition:
        age: 365
    # 7 years: Delete (or keep forever)
    # (Remove this rule if want to keep forever)
    # - action:
    #     type: Delete
    #   condition:
    #     age: 2555  # 7 years
EOF

gcloud storage buckets update gs://backup-archive \
  --lifecycle-config=lifecycle.yaml

# 3. Upload daily backup script
# (Every day, upload backup)
# gsutil cp backup-$(date +%Y%m%d).tar.gz gs://backup-archive/
```

**Cost Analysis:**
```
10TB/day × 30 days = 300TB

Standard (30 days): 100TB × $0.020 = $2.00
Nearline (30-90 days): 100TB × $0.010 = $1.00 + retrieval
Coldline (90-365 days): 100TB × $0.004 = $0.40 + retrieval
Archive (365 days-2555 days): 1600TB × $0.0012 = $1.92

Total monthly cost: ~$5.32 (very cheap for 7-year archive)
```

---

### Scenario 5: GKE Persistent Storage

**Requirement:**
- Stateful application (database)
- High availability (survive node failure)
- Persistent storage across pod restarts

**Solution: GKE + Regional Persistent Disks + StatefulSet**

```yaml
# 1. StorageClass for Regional PD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: regional-disk
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true

---
# 2. StatefulSet with PVC
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgresql
        image: postgres:14
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: regional-disk
      resources:
        requests:
          storage: 100Gi

---
# 3. Service for pod discovery
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  clusterIP: None
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

**Deploy:**
```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=password=secure-password

# Deploy StatefulSet
kubectl apply -f statefulset.yaml

# Verify PVC created
kubectl get pvc
# Output: data-database-0   Bound   pvc-xxxxx   100Gi   ...

# Access database
kubectl exec -it database-0 -- psql -U postgres
```

---

## Summary Checklist

### Cloud Storage
- [ ] Understand bucket creation & global naming
- [ ] Know storage classes (Standard, Nearline, Coldline, Archive)
- [ ] Can create lifecycle policies
- [ ] Understand versioning
- [ ] Know IAM vs ACL (prefer IAM)
- [ ] Can generate signed URLs
- [ ] Understand Google-managed vs CMEK encryption
- [ ] Can enable access logging
- [ ] Know cost optimization (storage classes, lifecycle)
- [ ] Can estimate costs

### Cloud Filestore
- [ ] Understand NFS shared file system
- [ ] Know tiers (Standard, Premium, Enterprise)
- [ ] Can mount from Compute Engine & GKE
- [ ] Understand VPC network requirement
- [ ] Know capacity limits (1-10TB)
- [ ] ReadWriteMany access mode
- [ ] Understand HA (replication, backups)
- [ ] Can estimate costs

### Persistent Disks
- [ ] Understand block storage (not object)
- [ ] Know disk types (Standard, Balanced, SSD)
- [ ] Understand Zonal vs Regional
- [ ] Can create, attach, detach disks
- [ ] Understand snapshots (incremental, global)
- [ ] Can resize disks (online)
- [ ] Can create images from disks
- [ ] Know encryption options
- [ ] GKE integration (StorageClass, PVC)
- [ ] Can estimate costs

---

**End of Storage Services Guide**
