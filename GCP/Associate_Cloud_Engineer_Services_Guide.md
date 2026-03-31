# Hướng Dẫn Chi Tiết Các Dịch Vụ GCP Cho Associate Cloud Engineer

## Mục Lục
1. [Compute Services](#compute-services)
2. [Storage Services](#storage-services)
3. [Networking Services](#networking-services)
4. [Database Services](#database-services)
5. [Management Tools](#management-tools)
6. [Security & Identity](#security--identity)
7. [Developer Tools](#developer-tools)

---

## COMPUTE SERVICES

### 1. Compute Engine (IaaS)

#### Khái Niệm Cơ Bản
- Là dịch vụ máy ảo (Virtual Machines) của GCP
- Cho phép bạn tạo và quản lý các máy chủ ảo
- Tương tự như EC2 trên AWS

#### Các Thành Phần Chính

**Instance Types & Machine Types**
- Các loại máy được định nghĩa bằng CPU, Memory, và Disk
- Loại máy chung (General-purpose): n1, n2, e2
- Loại máy tính toán (Compute-optimized): c2, c3
- Loại máy bộ nhớ (Memory-optimized): m1, m2
- Có thể custom machine types theo nhu cầu

**Images (Hình ảnh)**
- Public Images: Ubuntu, Debian, CentOS, Windows Server, etc.
- Custom Images: Bạn có thể tạo từ các instance hiện tại
- Machine Images: Bao gồm OS, configuration, installed software

**Disks (Ổ đĩa)**
- Boot Disk: Chứa OS (mặc định 10GB)
- Persistent Disks: Gắn thêm vào instance
  - Standard PD: Giá rẻ, hiệu suất trung bình
  - Balanced PD: Cân bằng giá và hiệu suất
  - SSD PD: Hiệu suất cao, giá đắt
- Local SSD: Lưu trữ tạm thời, hiệu suất rất cao

**Zones & Regions**
- Zone: Vị trí địa lý cụ thể (vd: us-central1-a)
- Region: Vùng gồm nhiều zones (vd: us-central1)
- Instance nằm trong zone cụ thể, không thể di chuyển giữa zones

#### Các Tính Năng Quan Trọng

**Startup Scripts & Shutdown Scripts**
- Startup script: Chạy tự động khi instance khởi động
- Có thể dùng để cài đặt package, start services

**Metadata & Custom Metadata**
- Cách để pass configuration đến instance
- Instance có thể query metadata từ http://metadata.google.internal

**Instance Groups**
- Managed Instance Groups (MIG): Tự động scaling & healing
  - Stateless MIGs: Cho các ứng dụng stateless
  - Stateful MIGs: Giữ persistent disk, startup scripts
- Unmanaged Instance Groups: Tập hợp instances được tạo thủ công

**Snapshots & Images**
- Snapshots: Backup của persistent disks
- Có thể tạo images từ snapshots hoặc instances
- Dùng cho disaster recovery và replication

#### Thực Hành Quan Trọng
- Tạo instance từ Console, gcloud CLI, hoặc Terraform
- Kết nối SSH đến instances
- Thay đổi machine type (phải stop instance trước)
- Quản lý disks
- Sử dụng startup scripts

---

### 2. App Engine

#### Khái Niệm Cơ Bản
- Platform-as-a-Service (PaaS) cho ứng dụng web
- Tự động scaling, không cần quản lý infrastructure
- Hỗ trợ Python, Node.js, Java, Go, PHP, Ruby, .NET

#### Hai Loại Environment

**Standard Environment**
- Instances có thể start/stop nhanh
- Không cần Docker container
- Tự động scaling xuống 0 nếu không có traffic
- Giới hạn: Chỉ chạy được ngôn ngữ được hỗ trợ, timeout 24h
- Giá rẻ cho ứng dụng không có traffic cao

**Flexible Environment**
- Chạy trong Docker containers
- Instances chạy liên tục, không thể scale xuống 0
- Có thể cài đặt custom libraries, binary files
- Timeout: 60 phút/request
- Phù hợp cho ứng dụng phức tạp, long-running processes

#### Cấu Hình Ứng Dụng
- **app.yaml**: File cấu hình chính
  - runtime: Ngôn ngữ lập trình
  - env: Standard hoặc Flexible
  - handlers: URL routing
  - env_variables: Environment variables
  - automatic_scaling: Scaling configuration

#### Các Tính Năng
- **Versions**: Có thể deploy nhiều version, traffic splitting
- **Services**: Chia ứng dụng thành multiple services
- **Cloud Tasks**: Queue tasks để xử lý bất đồng bộ
- **Scheduled Jobs**: Cron jobs
- **Custom Domains**: Domain riêng

#### Thực Hành Quan Trọng
- Deploy ứng dụng đơn giản
- Phân tách traffic giữa versions
- Sử dụng environment variables
- Hiểu app.yaml configuration

---

### 3. Cloud Run

#### Khái Niệm Cơ Bản
- Serverless container platform
- Chỉ tính tiền khi container chạy (millisecond-level)
- Tự động scaling từ 0 đến hàng ngàn instances
- Stateless containers (HTTP requests hoặc Pub/Sub messages)

#### Cách Hoạt Động
- Push Docker image lên Artifact Registry hoặc Container Registry
- Cloud Run tạo containers từ image
- Mỗi request là một container instance mới (hoặc reuse existing)
- Không cần quản lý servers, patches, OS updates

#### Cấu Hình Quan Trọng
- **Memory**: 128MB - 8GB
- **CPU**: Allocated based on memory
- **Timeout**: Tối đa 3600 giây (1 giờ)
- **Concurrency**: Số requests một container xử lý đồng thời
- **Minimum Instances**: Minimum số instances luôn chạy (để tránh cold starts)
- **Maximum Instances**: Giới hạn scaling

#### Security & Networking
- Default: Chỉ truy cập qua HTTPS
- **Ingress Settings**: Ai được phép gửi traffic
  - All: Ai cũng có thể
  - Internal: Chỉ GCP traffic
  - Internal & load balancer
- **Service Accounts**: Cloud Run sử dụng service account để access resources khác

#### Các Trường Hợp Sử Dụng
- APIs (REST, GraphQL)
- Web applications
- Data processing (dưới 1 giờ)
- Webhooks
- Background jobs (qua Pub/Sub)

#### Thực Hành Quan Trọng
- Tạo Dockerfile đơn giản
- Deploy từ Container Registry
- Set environment variables
- Hiểu about cold starts
- Xử lý graceful shutdown (SIGTERM signal)

---

### 4. Kubernetes Engine (GKE)

#### Khái Niệm Cơ Bản
- Managed Kubernetes service
- GCP quản lý control plane, bạn quản lý nodes
- Tự động updates, patches, repairs
- Tích hợp với GCP services (Cloud Storage, Cloud SQL, etc.)

#### Kubernetes Basics (Cần Biết)
- **Pods**: Unit nhỏ nhất, chứa 1 hoặc nhiều containers
- **Deployments**: Quản lý pods, rolling updates, scaling
- **Services**: Internal/External networking cho pods
- **ConfigMaps & Secrets**: Cấu hình và credentials
- **Persistent Volumes**: Storage
- **Namespaces**: Logical isolation

#### Node Pools
- Tập hợp nodes với cấu hình giống nhau
- Default node pool tạo tự động
- Có thể tạo node pools thêm với machine types khác
- Dùng để isolate workloads

#### Networking
- **Service Types**:
  - ClusterIP: Internal communication chỉ
  - NodePort: Access qua node ports
  - LoadBalancer: External IP + load balancing
  - Ingress: URL-based routing
- **VPC-native**: Sử dụng GCP VPC (bắt buộc từ GKE 1.21+)

#### Cluster Scaling
- **Horizontal Pod Autoscaler (HPA)**: Scale pods dựa trên metrics
- **Vertical Pod Autoscaler (VPA)**: Recommend resource requests
- **Cluster Autoscaler**: Scale nodes dựa trên pod demands

#### Monitoring & Logging
- Integration với Cloud Logging, Cloud Monitoring
- Container logs tự động shipped
- Pod metrics (CPU, memory)

#### Thực Hành Quan Trọng
- Tạo GKE cluster
- Deploy ứng dụng (Deployment, Service)
- Scale pods
- Update ứng dụng (rolling updates)
- Tìm hiểu về namespaces

---

## STORAGE SERVICES

### 1. Cloud Storage (Object Storage)

#### Khái Niệm Cơ Bản
- Object storage (giống S3 trên AWS)
- Lưu trữ dữ liệu không có cấu trúc (files, documents, backups, media)
- Highly available, durable (99.999999999% durability)
- Không có file size limit

#### Bucket Structure
- Buckets: Container cho objects
  - Tên phải unique global
  - Một khi create không thể change location
  - Có thể transfer giữa regions nhưng phí tính
- Objects: File bên trong bucket
  - Tên duy nhất trong bucket
  - Version control có thể bật

#### Storage Classes

**Standard**
- Dùng cho frequently accessed data
- Giá cao nhất, performance tốt nhất
- SLA 99.95%

**Nearline**
- Dùng cho monthly accessed data
- Giá thấp hơn Standard, nhưng có retrieval cost
- Bắt buộc lưu trữ tối thiểu 30 ngày

**Coldline**
- Dùng cho quarterly accessed data
- Giá rẻ, retrieval cost cao
- Bắt buộc lưu trữ tối thiểu 90 ngày

**Archive**
- Dùng cho rarely accessed data, long-term archival
- Giá rẻ nhất
- Retrieval cost cao nhất, latency cao
- Bắt buộc lưu trữ tối thiểu 365 ngày

#### Lifecycle Management
- Tự động thay đổi storage class
- Tự động xóa objects sau khoảng thời gian
- Dựa trên:
  - Age: Ngày tạo
  - Creation time
  - Is live
  - Matches storage class
  - Matches prefix

#### Access Control

**IAM (Identity & Access Management)**
- Role-based access control
- Roles: Storage Admin, Storage Object Viewer, Storage Object Creator, etc.

**ACL (Access Control Lists)**
- Object-level access control
- Deprecated, IAM là recommended approach

**Signed URLs**
- Cho phép temporary access không cần credentials
- Có expiration time

#### Versioning
- Giữ multiple versions của object
- Có thể restore version cũ
- Tạo thêm storage cost

#### Logging & Monitoring
- Access logs: Ai access, khi nào, từ đâu
- Integration với Cloud Logging

#### Thực Hành Quan Trọng
- Tạo buckets, upload/download files
- Hiểu storage classes
- Tạo lifecycle policies
- Sử dụng signed URLs
- Quản lý permissions (IAM)

---

### 2. Cloud Filestore (Managed NFS)

#### Khái Niệm Cơ Bản
- Managed Network File System (NFS)
- Dùng cho ứng dụng cần shared file storage
- Mount từ Compute Engine instances hoặc GKE
- POSIX-compliant

#### Instances
- Tên phải unique trong project
- Capacity: 1TB - 100TB
- Tier: Standard, High-Scale

#### Performance
- Throughput: GB/s (tùy theo capacity)
- IOPS: Operations per second
- Latency: Low-latency

#### Sử Dụng
- Mount qua NFS protocol
- Instance phải trong cùng VPC hoặc connected networks
- File permissions: Standard Unix permissions

#### Thực Hành Quan Trọng
- Tạo Filestore instance
- Mount từ instance
- Hiểu about IP reservations

---

### 3. Persistent Disks (Compute Engine Storage)

#### Khái Niệm Cơ Bản
- Được covered ở Compute Engine section
- Network-attached block storage
- Tách biệt từ instances (có thể attach/detach)
- Incremental snapshots

#### Regional Persistent Disks
- Replicated trong region
- Higher availability
- Có thể attach trong GKE

#### Thực Hành Quan Trọng
- Tạo và attach disks
- Resize disks (while attached)
- Tạo snapshots

---

## NETWORKING SERVICES

### 1. Virtual Private Cloud (VPC)

#### Khái Niệm Cơ Bản
- Virtual network environment
- Isolate resources
- Global resource (tuy nhiên subnets là regional)
- Mỗi project có default VPC

#### Subnets
- Regional resources
- Chứa resources (instances, GKE, etc.)
- IP ranges (CIDR blocks)
- Private Google Access: Cho phép access Google APIs không qua internet

#### IP Addressing

**Primary IP Addresses**
- Auto-assigned hoặc manual
- Nằm trong subnet's CIDR range

**Secondary IP Ranges**
- Thêm IP ranges vào subnet
- Dùng cho multiple subnets logic

**External IP Addresses**
- Static: Persists ngoài instances
- Ephemeral: Temporary, lost when instance stops
- Dùng để access từ internet

**Alias IP Ranges**
- Multiple IPs cho một network interface
- Network interfaces: Instance có thể có multiple NICs

#### Routes & Routing
- Determine traffic path
- Default route: 0.0.0.0/0 đến internet gateway
- Custom routes: Để nhắm mục tiêu subnets, instances

#### Firewall Rules
- **Ingress (Inbound)**: Traffic vào instances
- **Egress (Outbound)**: Traffic ra ngoài instances
- Default rules:
  - Deny all ingress
  - Allow all egress
- Rules bao gồm: Priority, direction, source/destination, protocols/ports, action

#### VPC Peering
- Connect giữa VPCs
- Traffic flows qua Google backbone
- Tính phí cho cross-region peering

#### Shared VPC
- Share một VPC giữa multiple projects
- Central project: Host project
- Other projects: Service projects
- Centralized network management

#### Thực Hành Quan Trọng
- Tạo VPC, subnets
- Tạo firewall rules
- Assign external IPs
- Understand route priority

---

### 2. Cloud Load Balancing

#### Khái Niệm Cơ Bản
- Distribute traffic giữa multiple backends
- Global service
- Integrated với auto-scaling
- Nhiều loại load balancers

#### HTTP(S) Load Balancer
- Layer 7 (Application layer)
- URL-based routing: /api → backend A, /images → backend B
- Host-based routing
- Path-based routing
- SSL/TLS termination
- Cloud CDN integration

#### TCP/UDP Load Balancer
- Layer 4 (Transport layer)
- Network Load Balancer (global)
- Internal TCP/UDP Load Balancer (regional)

#### Internal Load Balancer
- Private IP
- Dùng cho internal traffic
- Regional service

#### Network Load Balancer
- Extreme performance, high throughput
- Ultra-low latency
- Real-time, gaming, IoT, large scale

#### Load Balancer Components
- **Frontend**: IP, port, protocol
- **Backend Service/Pool**: Group of instances/services
- **Health Checks**: Determine instance health
- **Session Affinity**: Route requests từ client tới same backend

#### Health Checks
- Gửi periodic requests tới instances
- Check HTTP response codes, TCP connections, etc.
- Mark instance unhealthy nếu fail

#### Thực Hành Quan Trọng
- Tạo HTTP(S) load balancer
- Configure health checks
- URL-based routing
- Session affinity

---

### 3. Cloud Armor

#### Khái Niệm Cơ Bản
- DDoS protection và web application firewall
- Layer 7 protection
- Kết hợp với Cloud Load Balancing
- Rules-based protection

#### Security Policies
- Match rules: IP addresses, countries, request headers, etc.
- Actions: Allow, deny, rate limit
- Logging

#### Thực Hành Quan Trọng
- Tạo security policies
- Block requests từ certain countries
- Rate limiting

---

## DATABASE SERVICES

### 1. Cloud SQL (Managed Relational Database)

#### Khái Niệm Cơ Bản
- Managed database service
- Hỗ trợ MySQL, PostgreSQL, SQL Server
- Tự động backups, updates, patching
- High availability: Multi-zone setup có available

#### Instances
- Instance: Database server
- Database: Schema bên trong instance
- Users: Database users với passwords

#### Machine Types
- Shared-core: dev/test, giá rẻ
- Standard: Production
- High memory: Large datasets

#### Storage
- Automatic storage increase
- Manual storage resize (up only)
- Backups: Automatic hàng ngày, on-demand
- Point-in-time recovery

#### Connectivity
- Public IP: Access từ internet (với authorized networks)
- Private IP: VPC-native, recommended for security
- Cloud SQL Proxy: Để access từ application servers

#### Replication
- Read replicas: Read-only copies (same region, cross-region)
- Cloud SQL failover: Automatic failover (High availability setup)

#### Migrations
- Database Migration Service
- Cloud DMS: Migrate từ on-premises databases

#### Thực Hành Quan Trọng
- Tạo Cloud SQL instance
- Tạo databases, users
- Backup & restore
- Connect applications
- Basic replication

---

### 2. Cloud Firestore (NoSQL Database)

#### Khái Niệm Cơ Bản
- Managed NoSQL database
- Document-oriented (JSON documents)
- Real-time syncing
- Serverless (auto-scaling)
- Dữ liệu tổ chức thành collections, documents, fields

#### Data Model
- **Collections**: Như tables, chứa documents
- **Documents**: JSON objects, có ID
- **Fields**: Key-value pairs bên trong documents
- **Subcollections**: Collections bên trong documents

#### Modes
- **Datastore Mode**: Legacy, standard Firestore
- **Native Mode**: Real-time updates, better queries (recommended)

#### Querying
- Indexes: Cần cho complex queries
- Composite indexes: Cho multiple field queries
- Automatic indexes cho single field
- Query operators: ==, <, >, <=, >=, !=, in, array-contains

#### Real-time Features
- Listen for changes
- Automatic updates khi data thay đổi
- Perfect cho collaborative apps, live leaderboards

#### Security
- Security Rules: Define access control
- Rules language: Có thể check user authentication, document data
- Firestore Rules Simulator: Test rules

#### Pricing
- Read, write, delete operations
- Storage
- Network bandwidth

#### Thực Hành Quan Trọng
- Tạo database
- Add collections, documents
- Query data
- Real-time listeners
- Security rules

---

### 3. Cloud Bigtable (NoSQL for Big Data)

#### Khái Niệm Cơ Bản
- Managed, sparse multi-dimensional sorted map
- HBase compatible
- Cho massive scale (petabytes)
- Tính phí per node (không serverless)
- Cho time-series data, IoT, analytics

#### Data Model
- **Rows**: Row keys (string)
- **Column families**: Group related columns
- **Columns**: Qualified name (family:column)
- **Cells**: Intersection of row, column, timestamp
- **Timestamps**: Multiple versions per cell

#### Clusters & Nodes
- Cluster: Deployment unit
- Nodes: Compute capacity
- Min 3 nodes
- Tính phí per node per hour

#### Replication
- Multi-cluster replication
- Automatic failover (high availability)

#### Access Patterns
- HBase API
- Python client library
- gcloud CLI

#### Thực Hành Quan Trọng
- Tạo Bigtable instance
- Tạo column families, tables
- Insert, read data
- Basic understanding of architecture

---

### 4. Cloud Spanner (Globally Distributed Database)

#### Khái Niệm Cơ Bản
- Managed relational database
- Horizontally scalable
- Globally distributed
- Strong consistency across regions
- ACID transactions
- Tính phí per node

#### Instances & Databases
- Instance: Group of nodes
- Database: Schema
- Nodes: Storage & compute capacity

#### Replication
- Regional: Single region
- Multi-region: 3+ regions, high availability
- Automatic failover

#### Thực Hành Quan Trọng
- Biết khi nào sử dụng Spanner
- Basic instance, database creation
- Understand global distribution

---

## MANAGEMENT TOOLS

### 1. Cloud Console (gcloud CLI)

#### gcloud Basics
- Command-line interface cho GCP
- Quản lý resources programmatically
- Installation: Phần của Google Cloud SDK

#### Cấu Trúc Lệnh
```
gcloud <service> <resource> <action> [options]
```
Ví dụ:
- `gcloud compute instances create my-instance`
- `gcloud sql instances list`
- `gcloud storage buckets create gs://my-bucket`

#### Cấu Hình
- Projects: `gcloud config set project PROJECT_ID`
- Region/Zone: `gcloud config set compute/region us-central1`
- Credentials: `gcloud auth login`, `gcloud auth application-default login`

#### Cần Biết Lệnh
- **Compute Engine**: instances create, delete, list, ssh, etc.
- **Cloud Storage**: storage buckets create, copy, list, etc.
- **Cloud SQL**: sql instances create, databases create, users create
- **GKE**: container clusters create, get-credentials, delete
- **IAM**: iam service-accounts create, roles add, etc.

#### Thực Hành Quan Trọng
- Cài đặt Cloud SDK
- Authenticate gcloud
- Các lệnh cơ bản
- Shell scripts sử dụng gcloud

---

### 2. Cloud Monitoring & Logging

#### Cloud Monitoring
- Collect metrics từ resources
- Custom metrics
- Dashboards: Visualize metrics
- Alerts: Trigger actions khi thresholds hit
- Uptime checks: Monitor external endpoints

#### Cloud Logging
- Centralized logging
- Log aggregation từ multiple sources
- Log Explorer: Query logs
- Log-based metrics: Metrics derived từ logs
- Log sinks: Route logs đến Pub/Sub, Cloud Storage, BigQuery

#### Metrics
- Resource metrics: CPU, memory, disk
- Custom metrics: Application-specific

#### Alerting
- Thresholds: Numerical
- Forecasts: Predict future issues
- Conditions: Complex conditions (AND, OR, etc.)
- Notification channels: Email, SMS, Slack, PagerDuty, etc.

#### Thực Hành Quan Trọng
- Tạo dashboards
- Tạo alerts
- Query logs
- Custom metrics

---

### 3. Deployment Manager & Terraform

#### Deployment Manager
- Infrastructure as Code (IaC)
- YAML configuration
- Deploy GCP resources
- Templates: Reusable configurations

#### Terraform (Not GCP-specific nhưng important)
- IaC tool (works với GCP, AWS, Azure)
- HCL language
- State management
- Plan & apply workflow

#### Infrastructure as Code Benefits
- Version control
- Reproducibility
- Documentation
- Collaboration

#### Thực Hành Quan Trọng
- Cơ bản về Deployment Manager
- Tạo simple Terraform configurations
- Hiểu state files

---

## SECURITY & IDENTITY

### 1. Cloud IAM (Identity & Access Management)

#### Khái Niệm Cơ Bản
- Quản lý access tới GCP resources
- Who (identity) can do What (actions) on Which resources (resources)
- Least privilege principle

#### Identities
- **Google Account**: Personal Google account
- **Service Account**: Non-human account (cho applications)
- **User Account**: Email ending @yourdomain.com
- **Group Account**: Collection of users
- **Cloud Identity**: Identity management service

#### Roles
- **Predefined Roles**: GCP-provided roles (recommended)
  - Viewer: Read-only access
  - Editor: Read/write/delete access
  - Admin: Full access
  - Service-specific roles
- **Custom Roles**: Define custom permissions

#### Permissions
- Fine-grained actions
- Format: `service.resource.action`
- Ví dụ: `compute.instances.create`, `storage.buckets.get`

#### Policy Binding
- Assign role tới identity
- Multiple identities, multiple roles
- Inheritance: Project level roles apply to resources bên trong project

#### Service Accounts
- Non-human account
- Có email like `name@project.iam.gserviceaccount.com`
- Keys: JSON key files để authenticate applications
- Scopes: Restricted permissions cho GCE instances

#### Service Account Impersonation
- Allow tính năng khác impersonate service account
- Used in CI/CD pipelines

#### Thực Hành Quan Trọng
- Tạo service accounts
- Assign roles tới identities
- Tạo & manage keys
- Understand predefined roles

---

### 2. Cloud Key Management Service (KMS)

#### Khái Niệm Cơ Bản
- Manage encryption keys
- Encrypt/decrypt data
- Key rotation
- Compliance (FIPS 140-2)

#### Key Hierarchy
- **KeyRings**: Container for keys
- **Keys**: Cryptographic key
- **Versions**: Different versions của key

#### Encryption
- Encrypt data: Plaintext → Ciphertext
- Decrypt data: Ciphertext → Plaintext
- Key rotation: Automatic or manual

#### Integration
- Cloud Storage: Encrypt buckets
- Cloud SQL: Encrypt databases
- Compute Engine: Encrypt disks
- BigQuery: Encrypt datasets

#### Thực Hành Quan Trọng
- Tạo KeyRings, keys
- Enable encryption trên resources
- Key rotation

---

### 3. VPC Service Controls

#### Khái Niệm Cơ Bản
- Perimeter-based security
- Restrict data exfiltration
- Access context manager

#### Security Perimeters
- Define boundary
- Resources bên trong perimeter
- Only allow access từ specific networks

#### Thực Hành Quan Trọng
- Basic understanding of perimeters
- Không commonly tested for Associate level

---

## DEVELOPER TOOLS

### 1. Cloud Source Repositories

#### Khái Niệm Cơ Bản
- Git repositories
- Private repositories
- Integration với Cloud Build

#### Thực Hành Quan Trọng
- Basic Git commands
- Connect từ Cloud Build

---

### 2. Cloud Build

#### Khái Niệm Cơ Bản
- Continuous Integration/Continuous Deployment (CI/CD)
- Build container images từ source code
- Push images tới Container Registry
- Deploy applications

#### Builds
- cloudbuild.yaml: Configuration file
- Steps: Sequence of build steps
- Build cache: Speed up builds

#### Triggers
- Connect source repository
- Automatic builds on commits
- Build configuration

#### Thực Hành Quan Trọng
- Tạo cloudbuild.yaml
- Basic build configuration
- Docker image building

---

### 3. Artifact Registry

#### Khái Niệm Cơ Bản
- Store container images, artifacts
- Replace Container Registry
- Private repositories
- Regional service (lower latency)

#### Artifact Types
- Docker images
- Maven, npm, Python packages
- Helm charts

#### Thực Hành Quan Trọng
- Push/pull images
- Set up repositories
- Authentication

---

### 4. Google Cloud SDK

#### Khái Niệm Cơ Bản
- Cloud SDK: gcloud, gsutil, kubectl
- Installation & setup
- Authentication

#### Thực Hành Quan Trọng
- Install Cloud SDK
- Authenticate
- Quản lý multiple projects

---

## ADDITIONAL IMPORTANT TOPICS

### 1. Cloud Pub/Sub

#### Khái Niệm Cơ Bản
- Messaging service
- Publish-subscribe pattern
- Asynchronous messaging
- At-least-once delivery

#### Topics & Subscriptions
- **Topics**: Named resources mà publishers gửi messages
- **Subscriptions**: Named resources dùng để nhận messages

#### Use Cases
- Decouple services
- Event-driven architectures
- Real-time analytics
- Task queues

#### Thực Hành Quan Trọng
- Tạo topics, subscriptions
- Publish, consume messages
- Understanding of message flow

---

### 2. Cloud Tasks

#### Khái Niệm Cơ Bản
- Distributed task queue service
- Schedule tasks để execute later
- Retry failed tasks
- Rate limiting

#### Queues
- Create queues
- Add tasks
- Tasks gồm: URL, payload, schedule time

#### Thực Hành Quan Trọng
- Tạo queues
- Create, execute tasks
- Basic scheduling

---

### 3. Cloud Scheduler

#### Khái Niệm Cơ Bản
- Cron jobs on GCP
- Schedule jobs (HTTP, Pub/Sub, App Engine)
- Timezone support
- Retry logic

#### Jobs
- Cron expression format (quartz format)
- Target: HTTP endpoint, Pub/Sub topic, App Engine
- HTTP authentication

#### Thực Hành Quan Trọng
- Create scheduled jobs
- Cron expression understanding
- Target configuration

---

### 4. BigQuery (Data Warehouse)

#### Khái Niệm Cơ Bản
- Serverless data warehouse
- Analyze massive datasets (petabytes)
- SQL queries
- Fast query execution (seconds to minutes)

#### Datasets & Tables
- **Datasets**: Container for tables
- **Tables**: Relational tables with schema

#### Query Optimization
- Columnar storage: Fast for analytical queries
- Partitioning: Divide tables by date, time
- Clustering: Group rows by values
- Cost: Based on data scanned

#### Thực Hành Quan Trọng
- Tạo datasets, tables
- Basic SQL queries
- Understanding of pricing model

---

### 5. Cloud Armor & DDoS Protection

Covered ở Networking section

---

## PRACTICE EXAM TIPS

### Areas to Focus
1. **Compute Services**: Biết khi nào sử dụng Compute Engine, App Engine, Cloud Run, GKE
2. **Storage**: Biết storage classes, lifecycle policies
3. **Networking**: VPC, subnets, firewall rules, load balancing
4. **IAM**: Service accounts, roles, permissions
5. **Databases**: Biết khi nào sử dụng SQL vs NoSQL (Firestore, Bigtable)
6. **Monitoring & Logging**: Dashboards, alerts, log queries
7. **gcloud CLI**: Commands cho các services chính
8. **Security**: Encryption, access control, VPC controls

### Exam Format
- Multiple choice questions
- Scenario-based questions
- "Which service would you use for..."
- "What would you do to achieve..."

### Study Strategy
1. Đọc Google Cloud Documentation
2. Thực hành với free tier
3. Làm practice exams
4. Tìm hiểu use cases cho mỗi service
5. Focus vào differences giữa services

### Recommended Practice Areas
- Tạo production-like architectures
- Troubleshooting scenarios
- Cost optimization
- Security best practices
- High availability designs

---

## QUICK REFERENCE: SERVICE COMPARISON TABLE

| Service | Type | Use Case | Scaling | Pricing |
|---------|------|----------|---------|---------|
| Compute Engine | IaaS | Full control, long-running | Manual or MIG | Per hour |
| App Engine | PaaS | Web apps, auto-scaling | Automatic | Per request |
| Cloud Run | Serverless | Containers, event-driven | Automatic | Per invocation |
| GKE | Orchestration | Containerized apps | HPA, Cluster autoscaler | Per node |
| Cloud SQL | Database | Relational data | Replicas, failover | Per hour |
| Firestore | NoSQL | Real-time, documents | Automatic | Per operation |
| Bigtable | NoSQL | Big data, time-series | Per node | Per node |
| Cloud Storage | Object Storage | Files, backups | N/A | Per GB |
| BigQuery | Data Warehouse | Analytics, large datasets | Automatic | Per TB scanned |

---

## REFERENCES & RESOURCES

- [Google Cloud Documentation](https://cloud.google.com/docs)
- [Associate Cloud Engineer Exam Guide](https://cloud.google.com/certification/cloud-engineer)
- [Cloud Architecture Center](https://cloud.google.com/architecture)
- [Google Cloud Hands-on Labs](https://www.cloudskillsboost.google/)

---

**Last Updated**: April 2026
**Prepared For**: Associate Cloud Engineer Certification Exam
