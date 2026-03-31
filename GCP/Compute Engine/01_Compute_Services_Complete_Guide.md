# Compute Services - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về 4 compute services chính của GCP, biết khi nào sử dụng, cấu hình, và best practices.

---

## Mục Lục

1. [Compute Engine (IaaS)](#1-compute-engine-iaas)
2. [App Engine (PaaS)](#2-app-engine-paas)
3. [Cloud Run (Serverless Container)](#3-cloud-run-serverless-container)
4. [Kubernetes Engine (GKE)](#4-kubernetes-engine-gke)
5. [So Sánh & Chọn Service](#so-sánh--chọn-service)
6. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Compute Engine (IaaS)

### 1.1 Khái Niệm & Tổng Quan

**Compute Engine là gì?**
- Infrastructure-as-a-Service (IaaS)
- Máy ảo (Virtual Machines) được quản lý
- Full control over OS, software, networking
- Pay for what you use (per hour/minute)
- Tương tự AWS EC2

**Tại sao dùng Compute Engine?**
- Cần full control over infrastructure
- Legacy applications không thể containerize
- Custom OS, libraries cần thiết
- Long-running workloads
- High-performance computing (HPC)
- Database servers, web servers

**Anatomy of a Compute Engine Instance**

```
┌─────────────────────────────────────────┐
│      Compute Engine Instance            │
├─────────────────────────────────────────┤
│ • Machine Type (CPU, Memory)            │
│ • Boot Disk (OS image)                  │
│ • Additional Persistent Disks           │
│ • Network Interface(s)                  │
│ • Service Account                       │
│ • Metadata/Custom Metadata              │
│ • Startup/Shutdown Scripts              │
│ • Labels & Tags                         │
└─────────────────────────────────────────┘
```

---

### 1.2 Machine Types (Loại Máy)

#### Định Nghĩa
Machine type = combination of CPU + Memory + GPU/TPU options

#### Predefined Machine Types

**General-Purpose (Balanced)**
- E2 series: `e2-medium`, `e2-standard-2`, `e2-standard-4`, `e2-standard-8`, `e2-standard-16`, `e2-standard-32`
  - Giá rẻ nhất
  - Dùng cho: Web servers, small databases, dev/test
  - Performance: Burstable (có thể vượt giới hạn tạm thời)

- N1 series: `n1-standard-1`, `n1-standard-2`, `n1-standard-4`, ..., `n1-standard-96`
  - Stable performance
  - Dùng cho: Production workloads
  - Phổ biến nhất

- N2 series: `n2-standard-2`, `n2-standard-4`, ..., `n2-standard-80`
  - Generational update của N1
  - Better price-to-performance
  - Support up to 80 vCPU

- N2D series: AMD-based
  - Giá rẻ hơn Intel
  - Dùng cho: Price-sensitive workloads

**Compute-Optimized**
- C2 series: `c2-standard-4`, `c2-standard-8`, `c2-standard-16`, `c2-standard-30`, `c2-standard-60`
  - High CPU-to-memory ratio
  - Dùng cho: Heavy computation, batch processing, scientific computing

- C3 series: Latest compute-optimized
  - Better performance than C2
  - Up to 180 vCPU

**Memory-Optimized**
- M1 series: `m1-megamem-96`, `m1-ultramem-40`, `m1-ultramem-160`
  - High memory-to-CPU ratio
  - Dùng cho: Large in-memory databases, analytics

- M2 series: Updated memory-optimized
  - Up to 12 TB memory
  - Dùng cho: SAP HANA, large data processing

**Custom Machine Types**
- Tạo machine type riêng
- Flexible vCPU and memory allocation
- Có giới hạn: vCPU từ 0.25-96, Memory từ 0.9GB-5.5TB
- Tính phí dựa trên actual resources

```yaml
# Ví dụ custom machine type
Machine Type: custom-8-32768
# 8 vCPU, 32GB RAM
```

#### Machine Type Selection Logic

```
Chọn machine type nào?
├─ Web application? → E2 hoặc N1-standard
├─ Heavy computation? → C2 hoặc C3
├─ Large database? → M1 hoặc M2
├─ Dev/test, cheap? → E2
├─ Production, stable? → N1 hoặc N2
└─ Custom needs? → Custom machine type
```

---

### 1.3 Images (Hình Ảnh OS)

#### Public Images
Google cung cấp sẵn các OS images:
- Linux: Debian, Ubuntu, CentOS, RHEL, Fedora, Rocky Linux
- Windows: Windows Server 2012 R2, 2016, 2019, 2022
- Các specialized images: OpenShift, SAP, Secure Boot

#### Machine Images
Khác với OS images:
- Bao gồm toàn bộ VM configuration
- Include: Boot disk, data disks, network settings, metadata
- Dùng để replicate instances với exact configuration
- Can be used as template for Managed Instance Groups

**Ví dụ**: Tạo instance A với custom setup, tạo machine image từ nó, sau đó tạo instance B từ machine image → B có setup giống hệt A

#### Custom Images
- Tạo từ instance hoặc boot disk hiện tại
- Lưu trữ ở Cloud Storage
- Có thể share với projects khác
- Dùng để bake application vào image

**Best Practice**: Bake application vào custom image thay vì startup script
- Faster startup
- Consistent deployments

#### Image Family & Versioning
```
Image Family: debian-11
├─ Image: debian-11-bullseye-v20240101
├─ Image: debian-11-bullseye-v20240102
└─ Image: debian-11-bullseye-v20240103

# Khi deploy, có thể chỉ định:
gcloud compute instances create ... --image-family=debian-11
# → Tự động dùng latest image trong family
```

---

### 1.4 Storage (Ổ Đĩa)

#### Boot Disk
- OS disk, required
- Mặc định 10GB
- Có thể resize (up to 2TB), thậm chí resize while running
- Storage type: PD-Standard, PD-Balanced, PD-SSD
- Cannot be detached and reattached to another instance

#### Persistent Disks (PD)
Network-attached block storage

**Types:**
1. **Standard PD (PD-Standard)**
   - Giá: $$ (rẻ nhất)
   - Performance: ~60 IOPS, ~6 MB/s per GB
   - Dùng cho: Throughput-intensive applications
   - Example: Database backups, bulk processing

2. **Balanced PD (PD-Balanced)**
   - Giá: $$$ (trung bình)
   - Performance: Good balance
   - IOPS: ~3-15 IOPS per GB (tinh chỉnh)
   - Dùng cho: Most production workloads
   - **Recommended cho production**

3. **SSD PD (PD-SSD)**
   - Giá: $$$$ (đắt nhất)
   - Performance: ~30 IOPS, ~24 MB/s per GB
   - Latency: Sub-millisecond
   - Dùng cho: Databases, transactional systems
   - IOPS: Up to 100,000 IOPS for 65TB disk

**Characteristics:**
- Network-attached (không phải local)
- Can be detached từ instance
- Persistent across stop/restart
- Snapshots: Incremental, point-in-time recovery
- Replication: Standard (1 copy), Regional (cross-zone replication)

#### Local SSD
- Physically attached to host
- Very high IOPS (500K+) and throughput (7.3 GB/s)
- Dùng cho: Temporary data, caches, scratch space
- **Data lost** nếu instance stops/terminates
- Cannot be detached
- Cannot be snapshotted
- Fixed sizes: 375GB, 750GB, 1500GB, 3000GB, 6000GB per local SSD

#### Disk Size & Performance Metrics

```
Performance tăng theo size
250GB Standard PD: ~15 IOPS, 1.5 MB/s
1TB Standard PD: ~60 IOPS, 6 MB/s
3TB Standard PD: ~180 IOPS, 18 MB/s

Balanced PD performance varies, cần check documentation
```

#### Adding Disks to Instance

```bash
# Create new persistent disk
gcloud compute disks create my-disk --size=100GB --zone=us-central1-a

# Attach disk to instance
gcloud compute instances attach-disk my-instance --disk=my-disk --zone=us-central1-a

# SSH vào instance, format disk, mount
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /mnt/data
sudo mount /dev/sdb /mnt/data
```

#### Snapshots
- Point-in-time copy của disk
- Stored ở Cloud Storage
- Incremental: Chỉ lưu changes từ last snapshot
- Global resource: Có thể dùng cross-region
- Dùng để: Backup, disaster recovery, data migration

```bash
# Create snapshot
gcloud compute disks snapshot my-disk --snapshot-names=my-snapshot

# Create disk từ snapshot
gcloud compute disks create restored-disk \
  --source-snapshot=my-snapshot \
  --zone=us-central1-a
```

---

### 1.5 Networking Fundamentals

#### Network Interface
Mỗi instance có ít nhất 1 network interface
- Primary interface: Cannot be removed
- Secondary interfaces: Có thể add/remove
- Mỗi interface có: Primary IP, secondary IPs (alias), public IP (optional)

#### IP Addressing

**Primary IP (Internal IP)**
- Automatic từ subnet CIDR range
- Hoặc manual assignment
- Persistent across stop/restart
- Used for internal communication

**External IP (Public IP)**
- Optional
- Ephemeral: Lost khi instance stops (temporary)
- Static: Persists across stops (reserved)
- Dùng để: Access từ internet
- Tính phí nếu instance khác region hoặc unused

```bash
# Reserve static external IP
gcloud compute addresses create my-static-ip \
  --region=us-central1

# Assign tới instance
gcloud compute instances create my-instance \
  --address=my-static-ip
```

**Alias IP Ranges**
- Multiple IPs cho một network interface
- /32 for IPv4 (single IP)
- Dùng cho: Container networking, multi-tenant setups

#### Firewall Rules
Define connectivity giữa instances và external networks

**Default Rules:**
- Deny all ingress (incoming traffic)
- Allow all egress (outgoing traffic)

**Creating Custom Rules:**
```yaml
Name: allow-http
Direction: INGRESS
Priority: 1000 (lower = higher priority)
Action: ALLOW
Match:
  - Protocol: tcp
  - Port: 80
Source: 0.0.0.0/0
Target: http-server (tag)
```

**Rules Components:**
- Priority: 0-65534 (lower executes first)
- Direction: INGRESS or EGRESS
- Source/Destination: IP ranges, service accounts, network tags
- Protocol & Port: TCP, UDP, ICMP
- Action: ALLOW or DENY

**Network Tags:**
- Logical labels assigned to instances
- Firewall rules can target tags
- Better than IP-based rules (IPs change, tags don't)

```bash
# Create firewall rule
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

# Tag instance
gcloud compute instances create my-instance \
  --tags=http-server
```

#### Routes
Determine traffic path từ instance đi đâu

Default route: `0.0.0.0/0` → Internet Gateway (allows outbound to internet)

Custom route dùng để:
- Route to on-premises networks (via Cloud VPN/Interconnect)
- Route to other VPC networks (via peering)

---

### 1.6 Metadata & Custom Metadata

#### Metadata Server
Instance có thể query metadata dari `http://metadata.google.internal/computeMetadata/v1/`

**Built-in Metadata:**
```
instance/name
instance/id
instance/machine-type
instance/zone
instance/service-accounts/default/identity
instance/service-accounts/default/email
instance/tags
instance/labels
```

**Querying Metadata (di dalam instance):**
```bash
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/name

# Output: my-instance
```

#### Custom Metadata
Key-value pairs yang bisa pass ke instance

```bash
gcloud compute instances create my-instance \
  --metadata=startup-script="#!/bin/bash\necho 'Hello' > /tmp/hello.txt"
```

**Accessing Custom Metadata:**
```bash
# Di dalam instance
curl -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/guest-attributes/

# Di Google Cloud Console, bisa set custom key-value pairs
```

---

### 1.7 Startup Scripts & Shutdown Scripts

#### Startup Script
Execute tự động khi instance boot

```bash
#!/bin/bash
set -e

# Update system
apt-get update
apt-get install -y nginx

# Start service
systemctl start nginx
```

**Best Practices:**
- Keep short (< 5 minutes)
- Idempotent (safe to run multiple times)
- Logging: Output to `/var/log/syslog` atau `/var/log/startup-scripts.log`
- Check exit status

**Debugging Startup Script:**
```bash
# SSH ke instance
gcloud compute ssh my-instance --zone=us-central1-a

# Check logs
tail -f /var/log/syslog
# or
tail -f /var/log/startup-scripts.log
```

#### Shutdown Script
Execute tiba trước khi instance dihentikan/dihapus

Dùng để:
- Graceful shutdown
- Cleanup operations
- Send notifications

```bash
#!/bin/bash
# Graceful shutdown
systemctl stop nginx
# Save state
cp /var/data/state.json /tmp/backup/
```

---

### 1.8 Instance Templates & Instance Groups

#### Instance Templates
"Blueprint" untuk creating instances

Contains:
- Machine type
- Image
- Boot disk size
- Network configuration
- Tags, labels
- Service account
- Startup scripts

Cannot be updated, only create new version

```bash
gcloud compute instance-templates create my-template \
  --machine-type=n1-standard-1 \
  --image-family=debian-11 \
  --boot-disk-size=20GB \
  --tags=web-server \
  --metadata=startup-script='#!/bin/bash\napt-get update\napt-get install -y nginx'
```

#### Instance Groups

**Unmanaged Instance Groups**
- Manual collection of instances
- Không auto-scaling, healing
- Dùng cho: Legacy setups, mixed configurations
- Rarely used

**Managed Instance Groups (MIG)**
- Auto-scaling, auto-healing
- Health checks, rolling updates
- Dùng dengan instance templates

**MIG Types:**

1. **Stateless MIGs**
   - Instances có thể replaced anytime
   - Không persistent data
   - Perfect untuk: Web servers, stateless APIs
   - Configuration: Instances identically configured

2. **Stateful MIGs**
   - Preserve instance state khi replace
   - Terbuat lên persistent disks, local SSDs
   - Perfect untuk: Databases, stateful applications

#### Autoscaling Policy

```yaml
Target CPU Utilization: 60%
Target Load Balancing Utilization: 80%
Minimum Instances: 2
Maximum Instances: 10
Cool Down Period: 60 seconds
```

**Metrics MIG có thể scale on:**
- CPU utilization
- Load balancer metrics
- Custom metrics (from Cloud Monitoring)

#### Rolling Updates

Gradually update instances trong MIG

```bash
gcloud compute instance-groups managed set-autoscaling my-mig \
  --max-num-replicas=10 \
  --min-num-replicas=2 \
  --target-cpu-utilization=0.6 \
  --zone=us-central1-a

# Rolling update
gcloud compute instance-groups managed rolling-action start-update \
  my-mig \
  --version=template=my-new-template \
  --zone=us-central1-a
```

---

### 1.9 Service Accounts (untuk Compute Engine)

Instance menggunakan Service Account untuk:
- Access GCP resources (Cloud Storage, Cloud SQL, etc.)
- Authentication ke GCP APIs

**Default Service Account:**
- `[PROJECT-ID]-compute@developer.gserviceaccount.com`
- Automatically created per project
- Has Editor role by default (not recommended for production)

**Best Practice:**
- Create custom service account dengan minimal permissions
- Assign specific roles

```bash
# Create service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My App Service Account"

# Grant role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Use sa when creating instance
gcloud compute instances create my-instance \
  --service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Scopes (Legacy approach, phải biết vì exam):**
- Cloud SDK uses scopes để limit permissions
- Format: `https://www.googleapis.com/auth/[service]`
- Default scopes: `https://www.googleapis.com/auth/cloud-platform` (deprecated)

Prefer IAM roles over scopes.

---

### 1.10 Zones, Regions & Global Concepts

#### Zones (Availability Zones)
- Specific location within region
- Format: `region-zone` (e.g., `us-central1-a`)
- Instance nằm trong zone cụ thể
- **Không thể move instance between zones** (phải recreate)

#### Regions
- Contains 2-4 zones
- All zones in region tính là cùng location
- Lower latency giữa resources cùng region
- Format: `region` (e.g., `us-central1`)

#### Global Resources
- Available across all regions
- Examples: Images, snapshots, static external IPs, instance templates

#### Important Implications
```
Instance A ở us-central1-a
Instance B ở us-central1-b
→ Same region, different zone
→ Low latency, internal IP connectivity

Instance A ở us-central1-a
Instance B ở europe-west1-b
→ Different region
→ High latency, possible cross-region charges
```

---

### 1.11 Resizing & Changing Machine Types

#### Changing Machine Type
Instance must be stopped:
```bash
# Stop instance
gcloud compute instances stop my-instance --zone=us-central1-a

# Change machine type (must have same CPU series)
gcloud compute instances set-machine-type my-instance \
  --machine-type=n1-standard-4 \
  --zone=us-central1-a

# Start instance
gcloud compute instances start my-instance --zone=us-central1-a
```

**Constraints:**
- Instance harus stopped
- Cannot change CPU series (n1→n2 requires recreation)
- Can only change within compatible architectures

#### Resizing Boot Disk
Can be done while instance running:
```bash
# Resize boot disk
gcloud compute disks resize my-instance \
  --size=50GB \
  --zone=us-central1-a

# In instance, resize filesystem
sudo resize2fs /dev/sda1
```

---

### 1.12 Pricing & Cost Optimization

#### Pricing Components
1. **Compute**: Per vCPU per hour + Memory per GB per hour
2. **Storage**: Persistent disk, snapshots per GB
3. **Network**: Data transfer (free ingress, paid egress)
4. **Licensing**: Windows, SQL Server licenses

#### Cost Optimization Strategies

**Committed Use Discounts (CUDs)**
- Commit 1-3 years → 30-70% discount
- Must predetermine: Machine type, region
- Best untuk: Predictable, long-term workloads

**Sustained Use Discounts**
- Automatic discount (up to 30%) for resources run 25%+ of month
- No commitment required
- Best untuk: High utilization workloads

**Preemptible VMs**
- Up to 80% cheaper
- Can be terminated anytime (2-minute notice)
- Max 24 hours runtime
- Best untuk: Batch jobs, fault-tolerant apps, dev/test

```bash
# Create preemptible instance
gcloud compute instances create my-instance \
  --preemptible
```

**Recommendation:**
- Use E2 untuk non-critical workloads
- Use Committed Discounts untuk predictable loads
- Use Preemptible for batch processing
- Rightsize: Monitor actual usage, adjust machine type

---

### 1.13 Compute Engine Exam Tips

**Key Concepts to Know:**
- Machine types & when to use each
- Difference between PD-Standard, PD-Balanced, PD-SSD
- Boot disk vs Persistent disks vs Local SSD
- Startup scripts (idempotent, logging)
- Service accounts & IAM roles
- Instance templates & Managed Instance Groups
- Firewall rules & network tags
- Snapshots (incremental, global)
- Zones & regions (can't move instances)
- Preemptible VMs (use case, limitations)

**Common Exam Scenarios:**
1. "You need high performance database server" → PD-SSD or local SSD
2. "You need cost-effective web server" → E2 + PD-Balanced
3. "You need auto-scaling website" → Instance template + MIG + Load balancer
4. "You need to allow SSH from specific IP" → Firewall rule (tcp:22) + source IP
5. "You need to run batch jobs cheaply" → Preemptible VMs

---

## 2. App Engine (PaaS)

### 2.1 Khái Niệm & Tổng Quan

**App Engine là gì?**
- Platform-as-a-Service (PaaS)
- Fully managed, serverless platform
- Automatically scales based on traffic
- No infrastructure management
- Deploy từ source code hoặc containers
- Pay per request (Standard) hoặc per instance (Flexible)

**Tại sao dùng App Engine?**
- Don't want to manage servers
- Rapid development & deployment
- Automatic scaling (scale to zero)
- Integration với GCP services
- Simplified operations

**Khi không dùng App Engine:**
- Need low-level OS control → Compute Engine
- Need custom runtime → Cloud Run
- Need to run containers → GKE hoặc Cloud Run
- Legacy applications requiring Windows/complex setup

---

### 2.2 Standard vs Flexible Environment

#### Standard Environment

**Characteristics:**
- Instances tạo/destruc rất nhanh (milliseconds)
- Auto-scale xuống 0 (không có running instances khi no traffic)
- Runtimes: Python 3.7+, Node.js 12+, Java 8/11/17, Go 1.11+, PHP 7.2+, Ruby 2.7+
- Free tier: 28 frontend instance hours/day

**Limitations:**
- No arbitrary packages (chỉ pre-approved libraries)
- Request timeout: 24 hours (practical limit ~10 minutes)
- Memory per instance: 128MB - 512MB (can configure)
- Disk writes: Read-only filesystem (except /tmp)

**Latency:**
- Cold start: ~1-2 seconds (application loading)
- Warm requests: ~100-200ms

**Best for:**
- Quick prototypes
- MVPs
- Simple web applications
- Development environment

```python
# Python example - app.yaml
runtime: python39

handlers:
- url: /.*
  script: auto
```

#### Flexible Environment

**Characteristics:**
- Instances tạo từ Docker containers
- Instances luôn running (min 1 instance)
- Custom runtime bất kỳ (Python, Node, Go, custom dockerfile)
- Request timeout: 60 minutes
- Memory: Configurable (1GB - 32GB per instance)
- Disk: Can write to `/srv` directory

**Latency:**
- Cold start: ~15-30 seconds
- Warm requests: ~100-200ms

**Best for:**
- Complex applications
- Long-running processes
- Custom dependencies
- Existing Docker containers

```dockerfile
# Dockerfile cho Flexible App Engine
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

#### Quick Decision Tree

```
Standard hoặc Flexible?
├─ Cần custom dockerfile? → Flexible
├─ Cần install arbitrary packages? → Flexible
├─ Cần long-running processes? → Flexible
├─ Cần disk writes? → Flexible
├─ Đơn giản, quick deploy? → Standard
└─ Cần cost efficiency? → Standard
```

---

### 2.3 App Engine Configuration (app.yaml)

#### Minimal Configuration

```yaml
runtime: python39
```

#### Complete Configuration

```yaml
# Identifier
service: my-service  # Default: "default" service

# Runtime
runtime: python39
env: standard  # or 'flexible'
python_version: 3.9

# Health Check (Flexible only)
health_check:
  enable_health_check: true
  check_interval_sec: 40
  timeout_sec: 4
  unhealthy_threshold: 2
  healthy_threshold: 2
  restart_threshold: 60

# Instance Configuration
instance_class: F1  # F1, F2, F4, F4_1G (Standard only)
                    # For Flexible: automatic based on resources

# Manual or Automatic Scaling
automatic_scaling:  # Standard
  min_instances: 1
  max_instances: 10
  min_idle_instances: 1
  max_idle_instances: 2

# Or for Flexible:
manual_scaling:
  instances: 3

# Environment Variables
env_variables:
  FLASK_ENV: "production"
  DATABASE_URL: "postgresql://..."

# Secret Environment Variables (reference from Secret Manager)
env_variables:
  DB_PASSWORD: ${DB_PASSWORD}  # Reads from Secret Manager

# Handlers (routing)
handlers:
- url: /admin/.*
  script: auto
  secure: always  # Redirect HTTP to HTTPS
  
- url: /api/.*
  script: auto
  
- url: /.*
  script: auto

# Static Files
handlers:
- url: /static
  static_dir: static
  
- url: /(.*\.(gif|png|jpg))$
  static_files: static/\1
  upload: static/.*\.(gif|png|jpg)$

# Request Timeout
env: standard

# Network (Flexible)
network:
  instance_tag: my-instance-tag
```

#### Handlers & Routing
```yaml
handlers:
- url: /admin/*
  secure: always  # HTTP→HTTPS redirect
  script: auto
  
- url: /api/v1/.*
  script: auto
  
- url: /(.*\.gif|.*\.jpg)$  # Static files
  static_files: static/\1
  upload: static/.*\.(gif|jpg)$
```

**Handler Matching:**
- Evaluated top-to-bottom
- Exact match: `/admin/users`
- Regex: `/api/.*`
- Catch-all: `/.*`

**Secure Setting:**
- `always`: Redirect HTTP to HTTPS
- `optional`: Allow both
- `optional_with_redirect`: Like 'always'
- `never`: Disable HTTPS

---

### 2.4 Deploying to App Engine

#### Local Development

**Python:**
```bash
# Install dependencies
pip install -r requirements.txt

# Run locally
python app.py  # atau gunicorn -b :8080 app:app
```

**Node.js:**
```bash
npm install
npm start
```

#### Deployment

```bash
# Deploy to default service
gcloud app deploy

# Deploy specific app.yaml
gcloud app deploy app.yaml

# Deploy specific version
gcloud app deploy --version=v1

# Deploy with specific traffic split
gcloud app deploy --promote  # Set 100% traffic
gcloud app deploy --no-promote  # Don't send traffic yet

# Deploy multiple services
gcloud app deploy services/api/app.yaml services/web/app.yaml
```

#### Versioning & Traffic Splitting

Multiple versions dapat run simultaneously

```bash
# List versions
gcloud app versions list

# Split traffic between versions (A/B testing)
gcloud app services set-traffic default \
  --splits=v1=0.5,v2=0.5  # 50-50 split

# Gradual rollout
gcloud app services set-traffic default \
  --splits=v1=0.8,v2=0.2  # 80% old, 20% new
```

**Use Cases:**
- A/B testing
- Canary deployments
- Rolling updates

---

### 2.5 Services & Microservices Architecture

App Engine dapat host multiple services

```
App Engine Application
├─ default service (deployed dari root app.yaml)
├─ api service (deployed dari services/api/app.yaml)
└─ admin service (deployed dari services/admin/app.yaml)
```

```bash
# Structure
project/
├─ app.yaml  # default service
├─ services/
│  ├─ api/
│  │  └─ app.yaml  # api service
│  └─ admin/
│     └─ app.yaml  # admin service
```

```bash
# Deploy specific service
gcloud app deploy services/api/app.yaml

# Access services
# default: https://PROJECT_ID.appspot.com
# api: https://api-SERVICE_ID.appspot.com
# admin: https://admin-SERVICE_ID.appspot.com
```

**Benefits:**
- Independent scaling per service
- Independent versioning per service
- Shared infrastructure & resources

---

### 2.6 Environment Variables & Secrets

#### Environment Variables
```yaml
env_variables:
  FLASK_ENV: "production"
  API_KEY: "abc123"  # NOT for secrets!
  MAX_CONNECTIONS: "100"
```

**Access in code:**
```python
import os
env = os.getenv("FLASK_ENV", "development")
```

#### Secret Management (Secure Way)

**Using Secret Manager:**
```yaml
env_variables:
  DATABASE_PASSWORD: ${DB_PASSWORD}
```

```bash
# Create secret
gcloud secrets create db-password --data-file=- <<< "my-password"

# Grant App Engine service account access
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:PROJECT_ID@appspot.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

**In code:**
```python
from google.cloud import secretmanager

def access_secret_version(secret_id, version_id="latest"):
    client = secretmanager.SecretManagerServiceClient()
    project_id = "my-project"
    name = f"projects/{project_id}/secrets/{secret_id}/versions/{version_id}"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

db_password = access_secret_version("db-password")
```

---

### 2.7 Cloud Tasks Integration

Queue tasks dari App Engine untuk background processing

```python
# app.py
from google.cloud import tasks_v2

def push_task(queue, project, location, payload):
    client = tasks_v2.CloudTasksClient()
    parent = client.queue_path(project, location, queue)
    
    task = {
        'http_request': {
            'http_method': tasks_v2.HttpMethod.POST,
            'url': 'https://my-service.appspot.com/process',
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(payload).encode()
        }
    }
    
    response = client.create_task(request={'parent': parent, 'task': task})
    return response
```

---

### 2.8 Scheduling (Cron Jobs)

**cron.yaml:**
```yaml
cron:
- description: "send daily summary"
  url: /scheduled/send-summary
  schedule: "every day 08:00"
  
- description: "cleanup old data"
  url: /scheduled/cleanup
  schedule: "every 6 hours"
```

Endpoint handling:
```python
@app.route('/scheduled/send-summary')
def send_summary():
    # Verify request from Cron
    auth_header = request.headers.get('X-Appengine-Cron', False)
    if auth_header != 'true':
        return 'Unauthorized', 401
    
    # Do work
    return 'Summary sent', 200
```

---

### 2.9 Custom Domains & HTTPS

```bash
# Map custom domain
gcloud app custom-domains create www.mydomain.com

# HTTPS automatically provisioned (free SSL cert)
```

---

### 2.10 Monitoring & Logging

Logs automatically collected & shipped to Cloud Logging

```python
import logging
logging.info("Application started")
logging.warning("Something might be wrong")
logging.error("Error occurred")
```

Access logs via Cloud Console or gcloud:
```bash
gcloud app logs read

# Filter logs
gcloud app logs read --limit 50 --service=default --version=v1
```

---

### 2.11 Pricing & Cost Optimization

#### Standard Environment Pricing
- Instance hours: $0.07/hour (F1 class)
- Outbound traffic: $0.12/GB
- Cloud Storage (if used): $0.020/GB/month
- Free tier: 28 F1-hours/day

#### Flexible Environment Pricing
- vCPU: $0.0598/hour
- Memory: $0.0063/GB/hour
- Outbound traffic: $0.12/GB

#### Cost Optimization
1. **Use Standard for stateless web apps** (cheaper, faster scale)
2. **Set min_instances wisely** (higher = faster but more cost)
3. **Use traffic splitting** for gradual rollouts (faster feedback)
4. **Monitor resource usage** adjust instance class

---

### 2.12 App Engine Exam Tips

**Key Concepts:**
- Standard: Fast scaling, cheaper, limited
- Flexible: Custom runtime, slower scaling, more powerful
- app.yaml: Runtime, handlers, scaling, environment variables
- Versioning & traffic splitting
- Service account & IAM
- Startup time (cold start) implications
- Environment variables & secrets
- Cloud Tasks, Cron, Scheduling

**Common Scenarios:**
1. "Quick MVP deployment" → Standard Environment
2. "Long-running background jobs" → Cloud Tasks + Flexible
3. "Custom Docker image" → Flexible or Cloud Run
4. "A/B testing" → Versioning with traffic splitting
5. "Secure database password" → Secret Manager

---

## 3. Cloud Run (Serverless Container)

### 3.1 Khái Niệm & Tổng Quan

**Cloud Run là gì?**
- Fully managed, serverless container platform
- Deploy Docker containers
- Auto-scales from 0 to thousands
- Pay per request (billed to nearest 100ms)
- Stateless HTTP containers (or event-driven)
- No infrastructure management

**Tại sao dùng Cloud Run?**
- Want container flexibility of Cloud Run
- Serverless auto-scaling (scale to zero)
- Fast startup (seconds)
- Simple pricing (per request)
- Easy to containerize existing apps

**Khi không dùng Cloud Run:**
- Long-running batch jobs (> 1 hour)
- Persistent connections
- Need multiple ports per container
- Need GPUs/TPUs

---

### 3.2 How Cloud Run Works

#### Container Lifecycle

```
1. Push image to Artifact Registry / GCR
       ↓
2. Deploy to Cloud Run
       ↓
3. Cloud Run pulls image
       ↓
4. Incoming request arrives
       ↓
5. Container starts (or reuses warm instance)
       ↓
6. Request processed
       ↓
7. Response returned
       ↓
8. Container may be killed if idle (scale to zero)
```

#### Key Characteristics
- **Stateless**: Each request isolated
- **Containers**: Full Docker support
- **Fast Startup**: Seconds, not minutes
- **Cost**: Only pay for active requests (milliseconds)
- **Limits**: Max 1 hour timeout, max 32GB memory per request

---

### 3.3 Dockerfile & Container Image

#### Minimal Dockerfile (Python)

```dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
```

**Important Points:**
- Must listen on `0.0.0.0:PORT` (environment variable `PORT`, default 8080)
- Stateless (no files persist)
- /tmp available (8GB), but temporary
- Dockerfile must be optimized for size

#### Node.js Example

```dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
```

#### Build & Push Image

```bash
# Build locally
docker build -t my-app:latest .

# Push to Artifact Registry
gcloud builds submit --tag=gcr.io/PROJECT_ID/my-app:latest
# or
docker push gcr.io/PROJECT_ID/my-app:latest

# List images
gcloud artifacts repositories list
```

---

### 3.4 Deploying to Cloud Run

#### Deploy from Image

```bash
# Deploy
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated \
  --memory=512Mi \
  --cpu=1 \
  --timeout=3600 \
  --concurrency=10 \
  --min-instances=0 \
  --max-instances=100

# Output:
# Service URL: https://my-service-[hash].run.app
```

#### Deploy from Source

Cloud Run dapat build from source code directly:

```bash
gcloud run deploy my-service \
  --source=. \  # Current directory (must have Dockerfile)
  --region=us-central1 \
  --platform=managed
```

---

### 3.5 Cloud Run Configuration

#### Memory & CPU

```bash
--memory=128Mi to 32Gi  # Default: 512Mi
--cpu=1 to 8            # Default: 1 CPU
--timeout=300s to 3600s # Default: 300s
```

**CPU Allocation:**
- CPU allocated based on memory
- More memory = more CPU automatically

**Memory Presets:**
- 128Mi-512Mi: Limited CPU
- 1Gi+: Full CPU allocation

#### Concurrency

Maximum concurrent requests per container instance:

```bash
--concurrency=10  # Each instance handles max 10 concurrent requests
```

**Decision:**
- High concurrency: Fewer instances, more efficient
- Low concurrency: Better isolation, predictable performance

#### Min/Max Instances

```bash
--min-instances=0   # Can scale to zero (default)
--max-instances=100 # Maximum scale limit
```

**Min Instances:**
- 0: Scale to zero (cheaper but cold starts)
- 1+: Always running (faster response, higher cost)

**Use Case:**
- Predictable traffic: min=1
- Sporadic traffic: min=0
- Very unpredictable: min=1 for faster response

#### Timeout

```bash
--timeout=3600  # Max 1 hour (3600 seconds)
```

---

### 3.6 Environment Variables

#### Set Environment Variables

```bash
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --set-env-vars=FLASK_ENV=production,DEBUG=false
```

#### In Code (Python)

```python
import os
debug = os.getenv('DEBUG', 'false') == 'true'
flask_env = os.getenv('FLASK_ENV', 'development')
```

#### Secrets (Secure Way)

```bash
# Create secret
gcloud secrets create db-password --data-file=- <<< "password"

# Grant Cloud Run service account access
gcloud secrets add-iam-policy-binding db-password \
  --member=serviceAccount:PROJECT_ID@appspot.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor

# Deploy with secret
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --set-env-vars=DB_PASSWORD=secret://db-password
```

---

### 3.7 Service Accounts & IAM

#### Default Service Account
- `PROJECT_ID@appspot.gserviceaccount.com`
- Has limited permissions by default

#### Using Custom Service Account

```bash
# Create service account
gcloud iam service-accounts create my-cloud-run-sa

# Grant roles
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-cloud-run-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Deploy with service account
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --service-account=my-cloud-run-sa@PROJECT_ID.iam.gserviceaccount.com
```

#### Accessing GCP Resources from Container

```python
# Python client library auto-discovers credentials
from google.cloud import storage

client = storage.Client()  # Automatically uses service account
buckets = client.list_buckets()
```

---

### 3.8 Networking & Security

#### Ingress Control (Who can call your service)

```bash
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --ingress=all                    # Anyone
  # or
  --ingress=internal               # Only GCP traffic
  # or
  --ingress=internal-and-cloud-load-balancing
```

**Options:**
- `all`: Public internet (default)
- `internal`: Only GCP internal traffic
- `internal-and-cloud-load-balancing`: Internal + via load balancer

#### Cloud Load Balancer Integration

```bash
# Create load balancer pointing to Cloud Run
gcloud compute backend-services create my-backend \
  --global \
  --protocol=HTTP2 \
  --load-balancing-scheme=external
```

#### VPC Connector

Connect Cloud Run to VPC network:

```bash
# Create VPC connector
gcloud compute networks vpc-access connectors create my-connector \
  --network=default \
  --region=us-central1 \
  --range=10.8.0.0/28

# Deploy with VPC connector
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-app:latest \
  --vpc-connector=my-connector \
  --region=us-central1
```

---

### 3.9 Authentication & Authorization

#### Public vs Private Services

```bash
# Public (allow unauthenticated)
gcloud run deploy my-service --allow-unauthenticated

# Private (require authentication)
gcloud run deploy my-service --no-allow-unauthenticated
```

#### Invoke Private Service

```bash
# Using service account key
gcloud run deploy invoker-service \
  --image=gcr.io/PROJECT_ID/invoker:latest \
  --service-account=invoker-sa@PROJECT_ID.iam.gserviceaccount.com

# Grant Cloud Run Invoker role
gcloud run services add-iam-policy-binding my-service \
  --member=serviceAccount:invoker-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/run.invoker
```

#### Request Authentication (Token)

```python
import google.auth.transport.requests
import google.oauth2.id_token

# Get identity token
def get_service_url():
    credentials, project = google.auth.default()
    request = google.auth.transport.requests.Request()
    url = "https://my-service-hash.run.app"
    auth_req = google.oauth2.id_token.Request(request, token_uri="https://oauth2.googleapis.com/token")
    token = google.oauth2.id_token.fetch_id_token(auth_req, url)
    return token
```

---

### 3.10 Pub/Sub Integration

Cloud Run can be triggered by Pub/Sub messages

```bash
# Create topic
gcloud pubsub topics create my-topic

# Create subscription
gcloud pubsub subscriptions create my-subscription \
  --topic=my-topic \
  --push-endpoint=https://my-service-hash.run.app/handle-message \
  --push-auth-service-account=PROJECT_ID@appspot.gserviceaccount.com

# Grant Pub/Sub service account permission to invoke service
gcloud run services add-iam-policy-binding my-service \
  --member=serviceAccount:PROJECT_ID@appspot.gserviceaccount.com \
  --role=roles/run.invoker
```

**In Application:**

```python
@app.route('/handle-message', methods=['POST'])
def handle_message():
    envelope = request.get_json()
    payload = envelope['message']['data']  # base64 encoded
    
    # Process message
    return '', 204  # 204 No Content
```

---

### 3.11 Cloud Run with Cloud Build (CI/CD)

```yaml
# cloudbuild.yaml
steps:
# Step 1: Build image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app:$SHORT_SHA', '.']

# Step 2: Push image
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', 'gcr.io/$PROJECT_ID/my-app:$SHORT_SHA']

# Step 3: Deploy to Cloud Run
- name: 'gcr.io/cloud-builders/run'
  args: ['deploy', 'my-service',
         '--image=gcr.io/$PROJECT_ID/my-app:$SHORT_SHA',
         '--region=us-central1',
         '--platform=managed']

images:
  - 'gcr.io/$PROJECT_ID/my-app:$SHORT_SHA'
```

---

### 3.12 Monitoring & Logging

#### Cloud Logging Integration

Logs automatically collected

```bash
# View logs
gcloud run logs read my-service --limit=50 --region=us-central1
```

**In Application:**

```python
import logging
logging.info("Processing request")
logging.warning("Something suspicious")
logging.error("Error occurred")
```

#### Metrics

Cloud Run exports metrics:
- Request count
- Request latencies
- Error rates
- Container instance count

View in Cloud Console → Cloud Run → Metrics

---

### 3.13 Pricing & Cost Optimization

#### Pricing Components
1. **Requests**: $0.40 per 1M requests
2. **Compute**: $0.00001667 per vCPU-second
3. **Memory**: $0.000002084 per GB-second
4. **Outbound network**: $0.12/GB

#### Cost Optimization
1. **Right-size containers** (128Mi sufficient for many apps)
2. **Optimize concurrency** (higher = fewer instances)
3. **Use min-instances=0** (unless you need warm starts)
4. **Reduce timeout** (only keep what you need)
5. **Monitor cold starts** (may need min-instances=1)

**Example Cost:**
- 1M requests per month
- 512Mi memory
- 1 second average duration
- Request cost: $0.40
- Compute cost: $0.00001667 × 500M vCPU-ms = $6.67
- Total: ~$7/month

---

### 3.14 Cloud Run Exam Tips

**Key Concepts:**
- Serverless, stateless, containerized
- Auto-scales 0 to thousands
- Pay per request (milliseconds)
- Max 1 hour timeout
- Ingress control (public vs internal)
- Service accounts & IAM
- VPC connectors for private resources
- Pub/Sub integration
- Environment variables & secrets
- Cold start considerations

**Common Scenarios:**
1. "Run Docker container with auto-scaling" → Cloud Run
2. "Trigger by Pub/Sub message" → Cloud Run + Pub/Sub subscription
3. "Long-running batch job" → NOT Cloud Run (max 1 hour)
4. "Private service accessible only from VPC" → Cloud Run + VPC connector
5. "Cost-effective auto-scaling" → Cloud Run (pay per request)

---

## 4. Kubernetes Engine (GKE)

### 4.1 Khái Niệm & Tổng Quan

**GKE là gì?**
- Managed Kubernetes service
- Deploy containerized applications at scale
- Automatic management of control plane
- Pay per node (worker nodes), free control plane
- Container orchestration & auto-scaling

**Tại sao dùng GKE?**
- Need Kubernetes orchestration
- Complex microservices architecture
- Stateful applications (databases, caches)
- Multi-service deployments
- Self-healing & auto-restart

**Khi không dùng GKE:**
- Simple single-container app → Cloud Run
- Full VMs needed → Compute Engine
- Don't want Kubernetes complexity → App Engine
- Very cost-sensitive small apps → Cloud Run

---

### 4.2 Kubernetes Basics (Quick Review)

If you don't know Kubernetes, you need to learn:

**Pods:**
- Smallest deployable unit
- Wrapper around 1+ containers
- Containers in pod share network namespace
- Usually 1 container per pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: gcr.io/project/my-image:latest
    ports:
    - containerPort: 8080
```

**Deployments:**
- Manage pods (create, scale, update)
- Desired state → Kubernetes maintains it

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: gcr.io/project/my-image:latest
        ports:
        - containerPort: 8080
```

**Services:**
- Expose pods to network
- Service discovery
- Load balancing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer  # ClusterIP, NodePort, LoadBalancer, ExternalName
```

**ConfigMaps & Secrets:**
- Store configuration
- Store sensitive data

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "localhost"
  CACHE_TTL: "3600"

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded
```

**Persistent Volumes:**
- Storage for pods
- Decouple storage from pod lifecycle

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

### 4.3 GKE Cluster Architecture

#### Cluster Components

```
GKE Cluster
├── Control Plane (Managed by Google)
│   ├── API Server
│   ├── Scheduler
│   ├── Controller Manager
│   └── etcd (data store)
│
└── Node Pools (Managed by you)
    ├── Node Pool 1 (n1-standard-2)
    │   ├── Node 1
    │   ├── Node 2
    │   └── Node 3
    │
    └── Node Pool 2 (g4-dn-custom - with GPU)
        ├── Node 1
        └── Node 2
```

#### Cluster Types

**Standard Cluster:**
```bash
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n1-standard-2
```

**Autopilot Cluster:**
- Google manages entire cluster (including nodes)
- Better for: Hands-off Kubernetes
- More expensive but simpler

```bash
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --autopilot  # Google manages everything
```

**Difference:**
- Standard: You manage nodes, Google manages control plane
- Autopilot: Google manages nodes & control plane

---

### 4.4 Node Pools

Group of nodes with same configuration

```bash
# Create cluster with node pool
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n1-standard-2

# Add additional node pool
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --machine-type=n1-standard-4 \
  --num-nodes=2 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --accelerator=type=nvidia-tesla-k80,count=1
```

**Use Cases:**
- Different machine types for different workloads
- GPU/TPU nodes for ML workloads
- Separate scaling policies
- Cost optimization (preemptible nodes + regular nodes)

#### Node Taints & Tolerations

Control which pods can run on nodes

```yaml
# Node has taint
taint: workload=batch:NoSchedule

# Pod tolerates taint
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  tolerations:
  - key: workload
    operator: Equal
    value: batch
    effect: NoSchedule
  containers:
  - name: batch-processor
    image: gcr.io/project/batch:latest
```

---

### 4.5 Creating & Managing Clusters

#### Create Cluster

```bash
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n1-standard-2 \
  --disk-size=20 \
  --disk-type=pd-standard \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --enable-autorepair \
  --enable-autoupgrade
```

**Key Options:**
- `--zone`: Where to create cluster
- `--num-nodes`: Initial number of nodes
- `--machine-type`: Node machine type
- `--enable-autoscaling`: Auto-scale nodes
- `--enable-autorepair`: Auto-repair unhealthy nodes
- `--enable-autoupgrade`: Auto-upgrade Kubernetes version

#### Get Credentials

```bash
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a

# Now kubectl commands work
kubectl get nodes
kubectl get pods
```

#### Cluster Upgrades

```bash
# Check available versions
gcloud container get-server-config --zone=us-central1-a

# Upgrade cluster
gcloud container clusters upgrade my-cluster \
  --cluster-version=1.25.0 \
  --zone=us-central1-a

# Upgrade node pool
gcloud container node-pools update default-pool \
  --cluster=my-cluster \
  --node-version=1.25.0 \
  --zone=us-central1-a
```

---

### 4.6 Deploying Applications to GKE

#### kubectl Basics

```bash
# Create deployment from YAML
kubectl apply -f deployment.yaml

# Create deployment imperatively
kubectl create deployment my-app \
  --image=gcr.io/project/my-app:latest \
  --replicas=3

# Scale deployment
kubectl scale deployment my-app --replicas=5

# Expose deployment
kubectl expose deployment my-app \
  --port=80 \
  --target-port=8080 \
  --type=LoadBalancer

# View resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
```

#### Rolling Updates

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod during update
      maxUnavailable: 0  # 0 pods can be unavailable
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: gcr.io/project/my-app:v2
        ports:
        - containerPort: 8080
```

---

### 4.7 Networking in GKE

#### Services

**ClusterIP (Default):**
- Internal service, only accessible within cluster
- No external access
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

**NodePort:**
- Expose on all nodes
- Access via `<node-ip>:<port>`
- Range: 30000-32767

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

**LoadBalancer:**
- External load balancer
- GCP creates Cloud Load Balancer
- Exposes IP address

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

**ExternalName:**
- Map to external DNS name

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: external.example.com
```

#### Ingress

Advanced routing (Layer 7)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

### 4.8 Auto-Scaling

#### Pod Auto-Scaling (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
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

```bash
# Create HPA imperatively
kubectl autoscale deployment my-app \
  --min=2 \
  --max=10 \
  --cpu-percent=70
```

#### Node Auto-Scaling (Cluster Autoscaler)

Already enabled during cluster creation:
```bash
--enable-autoscaling --min-nodes=1 --max-nodes=5
```

Cluster Autoscaler automatically:
- Adds nodes when pods can't be scheduled
- Removes nodes when underutilized

---

### 4.9 Storage in GKE

#### Storage Classes

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: regional-pd
allowVolumeExpansion: true
```

#### PersistentVolumeClaims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
```

#### Using PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-app
    image: gcr.io/project/my-app:latest
    volumeMounts:
    - name: data-volume
      mountPath: /data
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: data-pvc
```

---

### 4.10 ConfigMaps & Secrets

#### ConfigMap (Non-sensitive data)

```bash
# Create from literal
kubectl create configmap app-config \
  --from-literal=DATABASE_HOST=localhost \
  --from-literal=CACHE_TTL=3600

# Create from file
kubectl create configmap app-config \
  --from-file=config.ini
```

```yaml
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-app
    image: gcr.io/project/my-app:latest
    envFrom:
    - configMapRef:
        name: app-config
```

#### Secret (Sensitive data)

```bash
# Create secret
kubectl create secret generic db-secret \
  --from-literal=password=secret123
```

```yaml
# Use in Pod
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-app
    image: gcr.io/project/my-app:latest
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

---

### 4.11 Namespaces

Logical isolation within cluster

```bash
# Create namespace
kubectl create namespace production
kubectl create namespace development

# Use namespace
kubectl apply -f deployment.yaml --namespace=production

# Set default namespace
kubectl config set-context --current --namespace=production
```

**Use Cases:**
- Separate environments (dev, staging, prod)
- Multi-tenancy
- Resource quotas per team

---

### 4.12 Monitoring & Logging in GKE

#### Integration with Cloud Monitoring & Cloud Logging

Automatic by default:
- Container logs → Cloud Logging
- Metrics (CPU, memory, etc.) → Cloud Monitoring

```bash
# View logs
gcloud logging read "resource.type=k8s_container" --limit=50

# Or use kubectl
kubectl logs <pod-name>
kubectl logs <pod-name> -f  # Follow logs

# Multiple containers
kubectl logs <pod-name> -c <container-name>

# Previous logs (if pod crashed)
kubectl logs <pod-name> --previous
```

#### Setting Resource Requests & Limits

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-app
    image: gcr.io/project/my-app:latest
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Requests**: Minimum resources guaranteed
**Limits**: Maximum resources allowed

---

### 4.13 Service Accounts in GKE

Pods use service accounts to access APIs

```bash
# Create service account
kubectl create serviceaccount my-app-sa --namespace=production

# Grant permissions
kubectl create clusterrolebinding my-app-sa-binding \
  --clusterrole=roles/editor \
  --serviceaccount=production:my-app-sa
```

```yaml
# Use in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      serviceAccountName: my-app-sa
      containers:
      - name: my-app
        image: gcr.io/project/my-app:latest
```

**Workload Identity:**
Allow pods to impersonate GCP service accounts
```bash
# Bind Kubernetes SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  my-gcp-sa@project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:project.svc.id.goog[namespace/k8s-sa]"
```

---

### 4.14 Pricing & Cost Optimization

#### Pricing
- Per node per hour
- Control plane: Free
- Additional services: Load balancer, persistent disks, etc.

#### Cost Optimization

1. **Right-size nodes**
   - Don't over-provision
   - Monitor actual usage

2. **Cluster autoscaling**
   - Remove idle nodes
   - Increase during traffic peaks

3. **Preemptible nodes**
   - 70% cheaper
   - Can be interrupted anytime
   - Suitable for fault-tolerant workloads

```bash
gcloud container node-pools create preemptible-pool \
  --cluster=my-cluster \
  --preemptible \
  --zone=us-central1-a
```

4. **Reserved instances**
   - Long-term discount
   - 30-70% cheaper
   - Commit 1-3 years

---

### 4.15 GKE Exam Tips

**Key Concepts:**
- Cluster architecture (control plane, nodes, node pools)
- Deployments, Services, Ingress
- ConfigMaps & Secrets
- PersistentVolumes & PersistentVolumeClaims
- Auto-scaling (pods & nodes)
- Namespaces
- Service accounts & RBAC
- Storage classes
- Monitoring & logging

**Common Scenarios:**
1. "Expose application to internet" → Service (LoadBalancer) or Ingress
2. "Scale pods based on CPU" → Horizontal Pod Autoscaler
3. "Run GPU workload" → Node pool with GPUs + taints & tolerations
4. "Separate team environments" → Namespaces + resource quotas
5. "Store sensitive data" → Kubernetes Secrets + Secret Manager binding

---

## So Sánh & Chọn Service

### Quick Decision Matrix

| Requirement | Compute Engine | App Engine | Cloud Run | GKE |
|---|---|---|---|---|
| **Control Level** | Full | Medium | Low | High |
| **Scaling** | Manual/MIG | Auto | Auto 0→∞ | Auto |
| **Startup Time** | Minutes | Seconds | Seconds | Seconds |
| **Cold Start Cost** | None | Minimal | Per req | Per node |
| **Max Runtime** | Unlimited | 24h | 1h | Unlimited |
| **Suitable for** | VMs, DBs | Web apps | APIs, event | Microservices |
| **Complexity** | Medium | Low | Low | High |
| **Cost** | Low-High | Medium | Very Low | Medium-High |

### Scenario-Based Selection

**"I have a legacy web application"**
→ Compute Engine (need OS control)

**"I have a simple Python web app"**
→ App Engine Standard (fastest to deploy)

**"I have a Docker container"**
→ Cloud Run (simple, cost-effective)

**"I have complex microservices"**
→ GKE (Kubernetes orchestration)

**"I need auto-scaling to zero"**
→ Cloud Run (serverless)

**"I need full OS control + auto-scaling"**
→ Compute Engine + Managed Instance Groups

**"I need long-running batch jobs"**
→ Compute Engine or GKE (not Cloud Run)

**"I need database workload"**
→ Compute Engine (need persistent state)

---

## Thực Hành & Scenarios

### Scenario 1: Deploy Web Application

**Requirement:**
- Simple web app
- Auto-scale based on traffic
- Minimal management

**Solution: App Engine Standard**

```bash
# 1. Create app.yaml
cat > app.yaml << EOF
runtime: python39
env: standard
instance_class: F1

handlers:
- url: /.*
  script: auto

env_variables:
  FLASK_ENV: production
EOF

# 2. Deploy
gcloud app deploy

# 3. Access
gcloud app browse
```

---

### Scenario 2: Deploy Containerized API

**Requirement:**
- Docker container
- Fast scaling (including 0)
- Low cost

**Solution: Cloud Run**

```bash
# 1. Create Dockerfile
cat > Dockerfile << EOF
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["gunicorn", "-b", "0.0.0.0:8080", "app:app"]
EOF

# 2. Build & push
gcloud builds submit --tag=gcr.io/PROJECT_ID/my-api:latest

# 3. Deploy
gcloud run deploy my-api \
  --image=gcr.io/PROJECT_ID/my-api:latest \
  --platform=managed \
  --region=us-central1 \
  --allow-unauthenticated
```

---

### Scenario 3: Deploy Microservices

**Requirement:**
- Multiple services
- Service discovery
- Auto-scaling
- Stateful components

**Solution: GKE**

```bash
# 1. Create cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3

# 2. Get credentials
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a

# 3. Deploy services
kubectl apply -f api-deployment.yaml
kubectl apply -f api-service.yaml
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f ingress.yaml

# 4. Check status
kubectl get pods
kubectl get services
```

---

### Scenario 4: Database Server

**Requirement:**
- Long-running database
- Persistent storage
- Full OS control

**Solution: Compute Engine**

```bash
# 1. Create instance with appropriate storage
gcloud compute instances create db-server \
  --machine-type=n1-standard-4 \
  --image-family=ubuntu-2204-lts \
  --boot-disk-size=30GB \
  --boot-disk-type=pd-ssd \
  --zone=us-central1-a \
  --service-account=default

# 2. SSH
gcloud compute ssh db-server --zone=us-central1-a

# 3. Install database
sudo apt-get update
sudo apt-get install -y postgresql-14
# ... configure ...
```

---

### Scenario 5: Background Jobs

**Requirement:**
- Process messages from queue
- Parallel processing
- Cost-effective

**Solution: Cloud Run + Pub/Sub OR GKE + Kafka**

```bash
# Cloud Run approach:
# 1. Create Pub/Sub topic
gcloud pubsub topics create job-queue

# 2. Deploy Cloud Run service
gcloud run deploy job-processor \
  --image=gcr.io/PROJECT_ID/processor:latest \
  --platform=managed

# 3. Create subscription
gcloud pubsub subscriptions create job-subscription \
  --topic=job-queue \
  --push-endpoint=https://job-processor-hash.run.app/process \
  --push-auth-service-account=PROJECT_ID@appspot.gserviceaccount.com

# Publish jobs
gcloud pubsub topics publish job-queue --message='{"task": "process_file"}'
```

---

## Summary Checklist

### Compute Engine
- [ ] Understand machine types (E2, N1, N2, C2, M1)
- [ ] Know storage types (Standard, Balanced, SSD, Local SSD)
- [ ] Can create instances with startup scripts
- [ ] Understand snapshots & images
- [ ] Know firewall rules & network tags
- [ ] Familiar with service accounts
- [ ] Know zones vs regions
- [ ] Understand MIGs & autoscaling
- [ ] Know preemptible VMs
- [ ] Can estimate costs

### App Engine
- [ ] Understand Standard vs Flexible
- [ ] Can write app.yaml
- [ ] Know handlers & routing
- [ ] Understand versioning & traffic splitting
- [ ] Can deploy applications
- [ ] Know environment variables & secrets
- [ ] Understand Cloud Tasks integration
- [ ] Know cron/scheduler
- [ ] Can estimate costs

### Cloud Run
- [ ] Understand serverless containerized model
- [ ] Can write Dockerfile
- [ ] Know memory & CPU allocation
- [ ] Understand concurrency & min/max instances
- [ ] Know ingress control
- [ ] Understand service accounts
- [ ] Know Pub/Sub integration
- [ ] Understand VPC connectors
- [ ] Can estimate costs

### GKE
- [ ] Understand cluster architecture
- [ ] Know Deployments, Services, Ingress
- [ ] Can write basic Kubernetes YAML
- [ ] Understand autoscaling (pods & nodes)
- [ ] Know ConfigMaps & Secrets
- [ ] Understand storage (PV, PVC, StorageClass)
- [ ] Know namespaces
- [ ] Understand service accounts & Workload Identity
- [ ] Can estimate costs

---

**End of Compute Services Guide**
