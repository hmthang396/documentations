# Management Tools - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về 3 management tools chính của GCP, biết cách sử dụng, cấu hình, best practices, và exam scenarios.

---

## Mục Lục

1. [Cloud Console & gcloud CLI](#1-cloud-console--gcloud-cli)
2. [Cloud Monitoring & Logging](#2-cloud-monitoring--logging)
3. [Deployment Manager & Terraform](#3-deployment-manager--terraform)
4. [So Sánh & Chọn Tool](#so-sánh--chọn-tool)
5. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Cloud Console & gcloud CLI

### 1.1 Cloud Console (Web UI)

**Cloud Console là gì?**
- Web-based user interface cho GCP
- Manage resources (compute, storage, databases, etc.)
- View metrics, logs, documentation
- Deploy applications
- Access Cloud Shell (built-in terminal)

**Accessing Cloud Console:**
```
https://console.cloud.google.com/
```

**Main Components:**

```
Cloud Console
├─ Projects dropdown (select current project)
├─ Search bar (search resources)
├─ Main navigation menu
│   ├─ Compute
│   ├─ Storage
│   ├─ Databases
│   ├─ Networking
│   ├─ Tools
│   └─ ...
├─ Notifications
├─ Cloud Shell button
└─ Account/Settings
```

**Console Features:**

1. **Create Resources**
   - Click "Create" buttons
   - Wizards guide you through setup
   - Visual validation of settings

2. **Manage Resources**
   - List view of resources
   - Inline editing (many settings)
   - Bulk actions

3. **Cloud Shell**
   - Built-in terminal (no installation needed)
   - Pre-installed: gcloud, kubectl, gsutil
   - 5GB persistent storage
   - Free tier: 50 hours/month

4. **Documentation**
   - Links to relevant docs
   - API reference
   - Samples

**Cloud Console Limitations:**
```
✗ Can't automate (manual clicking)
✗ Not suitable for infrastructure-as-code
✗ Hard to track changes
✗ Difficult for complex setups
→ Use gcloud CLI instead
```

---

### 1.2 gcloud CLI (Command Line)

**gcloud CLI adalah apa?**
- Command-line interface untuk GCP
- Control semua GCP resources
- Part of Google Cloud SDK
- Scriptable & automatable
- Better for:
  - Infrastructure automation
  - CI/CD pipelines
  - Complex operations
  - Batch operations

#### Installation

**Linux/macOS:**
```bash
# 1. Download SDK
curl https://sdk.cloud.google.com | bash

# 2. Restart shell
exec -l $SHELL

# 3. Initialize
gcloud init

# 4. Authenticate
gcloud auth login

# 5. Set default project
gcloud config set project PROJECT_ID
```

**Windows:**
```
Download installer from https://cloud.google.com/sdk/docs/install
Run installer
gcloud init
gcloud auth login
```

#### Basic gcloud Structure

```bash
gcloud <service> <resource> <action> [options]

Examples:
gcloud compute instances create my-instance
gcloud sql instances list
gcloud storage buckets create gs://my-bucket
gcloud container clusters describe my-cluster
```

#### Authentication

```bash
# Login with user account
gcloud auth login

# Set application default credentials (for applications)
gcloud auth application-default login

# List authenticated accounts
gcloud auth list

# Switch account
gcloud config set account USER@example.com

# Create & use service account
gcloud iam service-accounts create my-sa
gcloud auth activate-service-account \
  --key-file=/path/to/key.json
```

#### Configuration Management

```bash
# View current config
gcloud config list

# Set project
gcloud config set project my-project-id

# Set default region/zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Create named configuration (for multiple projects)
gcloud config configurations create prod
gcloud config set project prod-project-id
gcloud config activate prod

# List configurations
gcloud config configurations list
```

#### Common gcloud Commands

**Compute Engine:**
```bash
# Create instance
gcloud compute instances create my-instance \
  --machine-type=n1-standard-1 \
  --image-family=debian-11 \
  --zone=us-central1-a

# SSH to instance
gcloud compute ssh my-instance --zone=us-central1-a

# List instances
gcloud compute instances list

# Stop instance
gcloud compute instances stop my-instance --zone=us-central1-a

# Start instance
gcloud compute instances start my-instance --zone=us-central1-a

# Delete instance
gcloud compute instances delete my-instance --zone=us-central1-a

# Set tags
gcloud compute instances add-tags my-instance \
  --tags=http-server,https-server \
  --zone=us-central1-a
```

**Cloud Storage:**
```bash
# Create bucket
gcloud storage buckets create gs://my-bucket \
  --location=us-central1 \
  --default-storage-class=STANDARD

# List buckets
gcloud storage buckets list

# Upload file
gcloud storage cp file.txt gs://my-bucket/

# Download file
gcloud storage cp gs://my-bucket/file.txt ./

# Delete bucket (must be empty)
gcloud storage buckets delete gs://my-bucket

# Set bucket lifecycle
gcloud storage buckets update gs://my-bucket \
  --lifecycle-config=lifecycle.yaml
```

**Cloud SQL:**
```bash
# Create instance
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1

# List instances
gcloud sql instances list

# Create database
gcloud sql databases create mydb --instance=my-db

# Create user
gcloud sql users create appuser \
  --instance=my-db \
  --password=PASSWORD

# Describe instance
gcloud sql instances describe my-db
```

**GKE:**
```bash
# Create cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n1-standard-2

# Get cluster credentials
gcloud container clusters get-credentials my-cluster \
  --zone=us-central1-a

# List clusters
gcloud container clusters list

# Describe cluster
gcloud container clusters describe my-cluster \
  --zone=us-central1-a

# Delete cluster
gcloud container clusters delete my-cluster \
  --zone=us-central1-a
```

**IAM:**
```bash
# Add IAM binding
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/compute.instanceAdmin.v1

# List IAM bindings
gcloud projects get-iam-policy PROJECT_ID

# Remove IAM binding
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/compute.viewer
```

#### Filtering & Formatting Output

```bash
# Filter instances by label
gcloud compute instances list --filter="labels.env=prod"

# Filter by status
gcloud compute instances list --filter="status:RUNNING"

# Format output as JSON
gcloud compute instances list --format=json

# Format output as table (custom columns)
gcloud compute instances list \
  --format="table(name, zone, machine_type.machine_type().basename())"

# Output as CSV
gcloud compute instances list --format=csv(name,zone,status)
```

#### gcloud with Scripts

```bash
#!/bin/bash
# Backup all databases

DATABASES=$(gcloud sql instances list --format="value(name)")

for db in $DATABASES; do
  echo "Backing up $db..."
  gcloud sql backups create \
    --instance=$db \
    --description="Automated backup"
done

echo "Backup complete!"
```

---

### 1.3 Cloud Shell

**Built-in Terminal Environment:**

```
Cloud Shell Features:
✓ Pre-installed: gcloud, kubectl, gsutil, terraform
✓ 5GB persistent storage (home directory)
✓ 50 hours/month free
✓ Accessible from Cloud Console
✓ No local setup required
✓ Secure connection
```

**Using Cloud Shell:**

```bash
# Click Cloud Shell button in Console
# Or access via:
# https://shell.cloud.google.com/

# Start container immediately
# Home directory persists between sessions
# All gcloud commands available
```

---

### 1.4 gcloud Exam Tips

**Key Concepts:**
- gcloud is primary CLI tool
- Structure: gcloud <service> <resource> <action>
- Service accounts for automation
- Configuration files for settings
- Filtering & formatting output
- Scripting for batch operations

**Common gcloud Commands to Know:**
```
gcloud compute instances create/list/delete/ssh
gcloud storage buckets create/list/delete
gcloud sql instances create/list/describe
gcloud container clusters create/list/get-credentials
gcloud iam service-accounts create
gcloud config set/list
gcloud auth login/activate-service-account
```

---

## 2. Cloud Monitoring & Logging

### 2.1 Cloud Monitoring (Metrics)

**Cloud Monitoring adalah apa?**
- Collect metrics from GCP resources
- Create dashboards
- Set up alerts
- Application Performance Monitoring (APM)
- Trace requests across services

**Metrics Collected Automatically:**

```
Compute Engine:
  - CPU utilization
  - Memory usage
  - Disk I/O
  - Network throughput
  - Uptime

Cloud SQL:
  - CPU utilization
  - Memory usage
  - Disk I/O
  - Connections
  - Replication lag

GKE:
  - Pod CPU/memory
  - Container metrics
  - Network metrics
  - Volume metrics

And many more...
```

#### Creating Dashboards

**Via Console:**
```
Cloud Console → Monitoring → Dashboards → Create Dashboard
→ Add Chart → Select metric → Configure
```

**Via gcloud/API:**
```bash
# Create dashboard via API
gcloud monitoring dashboards create --config-from-file=dashboard.yaml
```

**Dashboard YAML Example:**
```yaml
displayName: "Web Application Dashboard"
mosaicLayout:
  columns: 12
  tiles:
    # CPU Utilization Chart
    - width: 6
      height: 4
      widget:
        title: "CPU Utilization"
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: |
                    metric.type="compute.googleapis.com/instance/cpu/utilization"
                    resource.type="gce_instance"
                    resource.label.instance_id="my-instance"

    # Memory Usage Chart
    - xPos: 6
      width: 6
      height: 4
      widget:
        title: "Memory Usage"
        xyChart:
          dataSets:
            - timeSeriesQuery:
                timeSeriesFilter:
                  filter: |
                    metric.type="compute.googleapis.com/instance/memory/usage"
```

#### Metrics & Labels

**Metric Types:**
```
• Gauge: Current value (CPU %, temperature)
• Delta: Cumulative value (errors, requests)
• Rate: Value per second (requests/sec)
```

**Labels:**
Additional dimensions for filtering

```
metric: compute.googleapis.com/instance/cpu/utilization
labels:
  resource_type: gce_instance
  instance_id: my-instance-123
  zone: us-central1-a
  project_id: my-project
```

#### Custom Metrics

Application-generated metrics

```python
# Python example using monitoring client
from google.cloud import monitoring_v3

def write_custom_metric(project_id, metric_value):
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"
    
    series = monitoring_v3.TimeSeries()
    series.metric.type = 'custom.googleapis.com/my_metric'
    
    now = time.time()
    seconds = int(now)
    nanos = int((now - seconds) * 10 ** 9)
    interval = monitoring_v3.TimeInterval(
        {"end_time": {"seconds": seconds, "nanos": nanos}}
    )
    
    point = monitoring_v3.Point(
        {"interval": interval, "value": {"double_value": metric_value}}
    )
    
    series.points = [point]
    client.create_time_series(name=project_name, time_series=[series])
```

---

### 2.2 Alerts

Set up notifications when metrics exceed thresholds

#### Creating Alerts

**Via Console:**
```
Cloud Monitoring → Alerting → Create Policy
→ Select metric → Set threshold → Configure notification
```

**Via YAML:**
```yaml
displayName: "High CPU Alert"
conditions:
  - displayName: "CPU > 80%"
    conditionThreshold:
      filter: |
        metric.type="compute.googleapis.com/instance/cpu/utilization"
        resource.type="gce_instance"
      comparison: COMPARISON_GT
      thresholdValue: 0.8
      duration: 300s  # Alert if exceeded for 5 minutes
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN

notificationChannels:
  - projects/PROJECT_ID/notificationChannels/123456

alertStrategy:
  autoClose: 86400s  # Auto-close after 24 hours
```

**Notification Channels:**
```
• Email
• Slack
• PagerDuty
• SMS
• Webhook
• Cloud Pub/Sub
```

#### Alert Conditions

```
Threshold-based:
  metric > value for duration
  
Multiple conditions (AND/OR):
  (CPU > 80% AND Memory > 90%) OR Disk > 95%
  
Rate of change:
  metric increasing too fast
  
Anomaly detection:
  unusual pattern detected
```

---

### 2.3 Cloud Logging

**Cloud Logging adalah apa?**
- Centralized log collection & management
- Logs from GCP resources
- Application logs
- System logs
- Query & analyze logs
- Log retention & archival

**Logs Collected Automatically:**

```
Cloud Logging receives:
• Cloud Audit Logs (who did what, when)
• System logs (from VMs, GKE, etc.)
• Application logs (stderr, stdout)
• Cloud Run logs
• Cloud Functions logs
• And more...
```

#### Log Structure

```
{
  "timestamp": "2024-04-28T12:34:56.789Z",
  "severity": "ERROR",
  "logName": "projects/my-project/logs/my-app",
  "message": "Database connection failed",
  "sourceLocation": {
    "file": "database.py",
    "line": 42,
    "function": "connect"
  },
  "labels": {
    "instance_id": "my-instance",
    "environment": "production"
  },
  "jsonPayload": {
    "error_code": 1045,
    "error_message": "Access denied for user 'app'@'localhost'"
  }
}
```

#### Querying Logs

**Logs Explorer (Web UI):**
```
Cloud Logging → Logs Explorer
```

**Query Language (Logging Query Language - LQL):**

```bash
# Basic query
resource.type="gce_instance"
severity="ERROR"

# Time-based
timestamp>="2024-04-28T00:00:00Z"
timestamp<="2024-04-28T23:59:59Z"

# Text search
textPayload=~"database.*error"

# JSON fields
jsonPayload.error_code=1045

# Complex queries
(resource.type="gce_instance" OR resource.type="k8s_container")
AND severity>="ERROR"
AND timestamp>="2024-04-28T00:00:00Z"

# Count errors per service
resource.labels.service_name
severity="ERROR"
```

**Via gcloud CLI:**
```bash
# Read logs
gcloud logging read "resource.type=gce_instance AND severity=ERROR" \
  --limit=50 \
  --format=json

# With filtering
gcloud logging read "resource.type=gce_instance" \
  --limit=100 \
  --filter="jsonPayload.request_path=~/^/api/"
```

#### Log Retention

```bash
# Set retention policy (default: 30 days)
# Via CLI: Use sink configuration
# Via Console: Logs → Logs Routing → Create Sink
```

#### Log Sinks

Export logs to external systems

```bash
# Create sink to Cloud Storage
gcloud logging sinks create my-sink \
  gs://my-logs-bucket \
  --log-filter='resource.type="gce_instance"'

# Create sink to BigQuery
gcloud logging sinks create my-bq-sink \
  bigquery.googleapis.com/projects/my-project/datasets/logs \
  --log-filter='severity="ERROR"'

# Create sink to Pub/Sub
gcloud logging sinks create my-pubsub-sink \
  pubsub.googleapis.com/projects/my-project/topics/logs \
  --log-filter='resource.type="cloud_run"'
```

**Sink Use Cases:**
```
1. Long-term archival (Cloud Storage)
2. Analytics (BigQuery)
3. Real-time processing (Pub/Sub)
4. External systems (webhook)
```

#### Log-based Metrics

Create metrics from logs

```bash
# Create metric from ERROR logs
gcloud logging metrics create error_count \
  --description="Count of ERROR logs" \
  --log-filter='severity="ERROR"'

# Use metric in alerts
# (Reference: logging.googleapis.com/user/error_count)
```

---

### 2.4 Application Logging Best Practices

**Structured Logging:**

```python
import json
import logging
import sys

# Structured logging (JSON format)
def log_event(severity, message, **kwargs):
    log_entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "severity": severity,
        "message": message,
        **kwargs  # Custom fields
    }
    print(json.dumps(log_entry), file=sys.stdout)

# Usage
log_event("INFO", "User login", user_id=123, ip="203.0.113.1")
log_event("ERROR", "Database error", error_code=1045, retry=True)
```

**Log Levels:**
```
DEBUG: Development debugging
INFO: General information
WARNING: Warning messages
ERROR: Error messages
CRITICAL: Critical errors
```

**What to Log:**
```
✓ Application errors
✓ Important state changes
✓ Request/response data
✓ Authentication events
✓ Performance metrics

✗ Passwords/credentials
✗ Sensitive PII
✗ Full request bodies (unless necessary)
✗ Every single operation (noisy)
```

---

### 2.5 Monitoring & Logging Exam Tips

**Key Concepts:**
- Automatic metric collection
- Custom dashboards
- Threshold-based alerts
- Cloud Logging for centralized logs
- Logs Explorer for querying
- Log sinks for exporting
- Log-based metrics
- Structured logging

**Common Scenarios:**
1. "Alert when CPU > 80%" → Monitoring threshold alert
2. "Track all errors" → Cloud Logging + Log-based metric
3. "Export logs to BigQuery" → Log sink
4. "Monitor application performance" → Custom metrics + dashboard
5. "Analyze request patterns" → Logs Explorer + querying

---

## 3. Deployment Manager & Terraform

### 3.1 Infrastructure as Code (IaC) Overview

**What is IaC?**
```
Define infrastructure in code
├─ Version control
├─ Repeatable deployments
├─ Documentation
├─ Rollback capability
├─ Automated testing

Tools:
├─ Terraform (multi-cloud, HCL)
├─ Deployment Manager (GCP-specific, YAML/Jinja2)
├─ Cloud Formation (AWS)
├─ Pulumi (Python, Go, etc.)
```

**Benefits:**
```
✓ Version control infrastructure
✓ Reproducible environments
✓ Faster deployments
✓ Fewer manual errors
✓ Easy collaboration
✓ Disaster recovery
```

---

### 3.2 Google Cloud Deployment Manager

**Deployment Manager adalah apa?**
- GCP's native IaC tool
- YAML/Jinja2 templates
- Manages GCP resources
- State management (automatic)
- Template library

#### Basic Deployment Template

```yaml
# deployment.yaml
resources:
  # Create VPC network
  - name: my-network
    type: compute.v1.network
    properties:
      autoCreateSubnetworks: false

  # Create subnet
  - name: my-subnet
    type: compute.v1.subnetwork
    properties:
      network: $(ref.my-network.selfLink)
      ipCidrRange: 10.0.0.0/24
      region: us-central1

  # Create firewall rule
  - name: allow-http
    type: compute.v1.firewall
    properties:
      network: $(ref.my-network.selfLink)
      allowed:
        - IPProtocol: tcp
          ports: [80, 443]
      sourceRanges: [0.0.0.0/0]

  # Create instance
  - name: my-instance
    type: compute.v1.instance
    properties:
      zone: us-central1-a
      machineType: zones/us-central1-a/machineTypes/n1-standard-1
      networkInterfaces:
        - network: $(ref.my-network.selfLink)
          subnetwork: $(ref.my-subnet.selfLink)
      disks:
        - boot: true
          initializeParams:
            sourceImage: projects/debian-cloud/global/images/family/debian-11
      metadata:
        items:
          - key: startup-script
            value: |
              #!/bin/bash
              apt-get update
              apt-get install -y apache2
              echo "Hello from $(hostname)" > /var/www/html/index.html

outputs:
  instanceName:
    value: $(ref.my-instance.name)
  instanceIP:
    value: $(ref.my-instance.networkInterfaces[0].networkIP)
```

#### Deployment Manager Commands

```bash
# Create deployment
gcloud deployment-manager deployments create my-deployment \
  --config=deployment.yaml

# List deployments
gcloud deployment-manager deployments list

# Describe deployment
gcloud deployment-manager deployments describe my-deployment

# Update deployment
gcloud deployment-manager deployments update my-deployment \
  --config=deployment.yaml

# Delete deployment
gcloud deployment-manager deployments delete my-deployment

# Get deployment events
gcloud deployment-manager operations list --deployment=my-deployment

# Preview changes (dry-run)
gcloud deployment-manager deployments create my-deployment \
  --config=deployment.yaml \
  --preview
```

#### Deployment Manager Templates (Jinja2)

Reusable templates with variables

```yaml
# instance-template.jinja
resources:
  - name: {{ properties.instance_name }}
    type: compute.v1.instance
    properties:
      zone: {{ properties.zone }}
      machineType: zones/{{ properties.zone }}/machineTypes/{{ properties.machine_type }}
      networkInterfaces:
        - network: {{ properties.network }}
      disks:
        - boot: true
          initializeParams:
            sourceImage: {{ properties.image }}
      metadata:
        items:
          - key: startup-script
            value: |
              {{ properties.startup_script }}

outputs:
  - name: instance_id
    value: $(ref.{{ properties.instance_name }}.id)
```

**Using Template:**
```yaml
# main.yaml
imports:
  - path: instance-template.jinja

resources:
  - name: my-instances
    type: instance-template.jinja
    properties:
      instance_name: web-server-1
      zone: us-central1-a
      machine_type: n1-standard-1
      network: my-network
      image: projects/debian-cloud/global/images/family/debian-11
      startup_script: |
        apt-get update
        apt-get install -y apache2

  - name: my-instances-2
    type: instance-template.jinja
    properties:
      instance_name: web-server-2
      zone: us-central1-b
      machine_type: n1-standard-1
      network: my-network
      image: projects/debian-cloud/global/images/family/debian-11
      startup_script: |
        apt-get update
        apt-get install -y apache2
```

---

### 3.3 Terraform

**Terraform adalah apa?**
- Multi-cloud IaC tool by HashiCorp
- HCL language (more intuitive than YAML)
- Provider-based (AWS, GCP, Azure, etc.)
- State file (tracks infrastructure)
- Plan before apply

#### Terraform vs Deployment Manager

```
Terraform:
  ✓ Multi-cloud support
  ✓ Large community
  ✓ More powerful language (HCL)
  ✓ Better state management
  ✓ More providers/resources
  ✗ Requires local state management

Deployment Manager:
  ✓ Native GCP integration
  ✓ Simple YAML
  ✓ No state file (Google manages)
  ✗ GCP-only
  ✗ Limited language
```

#### Terraform Basics

**Installation:**
```bash
# Download from terraform.io
# Or via package manager
# macOS:
brew install terraform

# Linux:
wget https://releases.hashicorp.com/terraform/1.4.6/terraform_1.4.6_linux_amd64.zip
unzip terraform_1.4.6_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

**Basic Terraform Configuration:**

```hcl
# main.tf

# Configure the provider
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = "my-project-id"
  region  = "us-central1"
}

# Create VPC network
resource "google_compute_network" "my_network" {
  name                    = "my-network"
  auto_create_subnetworks = false
}

# Create subnet
resource "google_compute_subnetwork" "my_subnet" {
  name          = "my-subnet"
  ip_cidr_range = "10.0.0.0/24"
  region        = "us-central1"
  network       = google_compute_network.my_network.id
}

# Create firewall rule
resource "google_compute_firewall" "allow_http" {
  name    = "allow-http"
  network = google_compute_network.my_network.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
}

# Create instance
resource "google_compute_instance" "my_instance" {
  name         = "my-instance"
  machine_type = "n1-standard-1"
  zone         = "us-central1-a"

  boot_disk {
    initialize_params {
      image = "debian-cloud/debian-11"
      size  = 20
    }
  }

  network_interface {
    network    = google_compute_network.my_network.id
    subnetwork = google_compute_subnetwork.my_subnet.id
  }

  metadata_startup_script = <<-EOT
              #!/bin/bash
              apt-get update
              apt-get install -y apache2
              echo "Hello from $(hostname)" > /var/www/html/index.html
            EOT

  tags = ["http-server", "https-server"]
}

# Output values
output "instance_ip" {
  value = google_compute_instance.my_instance.network_interface[0].network_ip
}

output "instance_public_ip" {
  value = google_compute_instance.my_instance.network_interface[0].access_config[0].nat_ip
}
```

#### Terraform Workflow

```bash
# 1. Initialize Terraform (download providers, create .terraform)
terraform init

# 2. Format code (optional but good practice)
terraform fmt

# 3. Validate configuration
terraform validate

# 4. Plan changes (shows what will be created/updated/deleted)
terraform plan -out=tfplan

# 5. Review plan
# (Open tfplan or check console output)

# 6. Apply changes (execute the plan)
terraform apply tfplan

# 7. Verify resources created
terraform state list
terraform state show google_compute_instance.my_instance

# 8. Update configuration
# (Edit main.tf)

# 9. Plan again
terraform plan

# 10. Apply updates
terraform apply

# 11. Destroy all resources (careful!)
terraform destroy
```

#### Terraform State

**State File (terraform.tfstate):**
```json
{
  "version": 4,
  "terraform_version": "1.4.6",
  "resources": [
    {
      "type": "google_compute_instance",
      "name": "my_instance",
      "instances": [
        {
          "attributes": {
            "id": "my-instance",
            "machine_type": "n1-standard-1",
            "zone": "us-central1-a",
            ...
          }
        }
      ]
    }
  ]
}
```

**State Management:**
```
Local state (development):
  ├─ terraform.tfstate (in current directory)
  └─ terraform.tfstate.backup

Remote state (production):
  ├─ GCS bucket
  ├─ Terraform Cloud
  └─ Other backends
```

**Remote State (Recommended):**

```hcl
# main.tf
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
  }
}

# Then:
# gcloud storage buckets create gs://my-terraform-state
# terraform init (choose remote state)
```

#### Terraform Variables

```hcl
# variables.tf
variable "instance_count" {
  type        = number
  default     = 1
  description = "Number of instances to create"
}

variable "machine_type" {
  type        = string
  default     = "n1-standard-1"
  description = "Machine type for instances"
}

variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# main.tf
resource "google_compute_instance" "my_instance" {
  count        = var.instance_count
  machine_type = var.machine_type
  # ...
}

# terraform.tfvars
instance_count = 3
machine_type   = "n1-standard-2"
environment    = "prod"

# Or pass via command line
# terraform apply -var="instance_count=5"
```

#### Terraform Modules

Reusable configurations

```
# Module structure
modules/
  └─ vpc/
      ├─ main.tf
      ├─ variables.tf
      └─ outputs.tf
  └─ instance/
      ├─ main.tf
      ├─ variables.tf
      └─ outputs.tf
```

**Using Modules:**

```hcl
# modules/vpc/main.tf
resource "google_compute_network" "network" {
  name                    = var.network_name
  auto_create_subnetworks = false
}

resource "google_compute_subnetwork" "subnet" {
  name          = var.subnet_name
  network       = google_compute_network.network.id
  ip_cidr_range = var.ip_cidr_range
  region        = var.region
}

# modules/vpc/outputs.tf
output "network_id" {
  value = google_compute_network.network.id
}

output "subnet_id" {
  value = google_compute_subnetwork.subnet.id
}

# main.tf (root module)
module "vpc" {
  source = "./modules/vpc"
  
  network_name = "my-network"
  subnet_name  = "my-subnet"
  ip_cidr_range = "10.0.0.0/24"
  region       = "us-central1"
}

module "instances" {
  source = "./modules/instance"
  
  count        = 3
  instance_name = "web-server-${count.index + 1}"
  network_id   = module.vpc.network_id
  subnet_id    = module.vpc.subnet_id
}
```

---

### 3.4 Deployment Manager vs Terraform

| Aspect | Deployment Manager | Terraform |
|--------|-------------------|-----------|
| **Cloud** | GCP only | Multi-cloud |
| **Language** | YAML/Jinja2 | HCL |
| **State** | Google-managed | Local/Remote |
| **Community** | Smaller | Large & active |
| **Learning** | Easier (YAML) | Steeper curve (HCL) |
| **Flexibility** | Limited | More powerful |
| **Best for** | GCP-only projects | Multi-cloud, complex |

---

### 3.5 Management Tools Exam Tips

**Key Concepts:**
- gcloud CLI for automation
- Cloud Console for visual management
- Cloud Monitoring for metrics & alerts
- Cloud Logging for centralized logs
- Infrastructure as Code (IaC)
- Deployment Manager (GCP-native)
- Terraform (multi-cloud)
- State management

**Common Scenarios:**
1. "Automate resource creation" → Terraform or Deployment Manager
2. "Alert when disk space full" → Cloud Monitoring threshold alert
3. "Track all API calls" → Cloud Logging + Cloud Audit Logs
4. "Export logs for analytics" → Log sink to BigQuery
5. "Create reusable configs" → Terraform modules

---

## So Sánh & Chọn Tool

### Tool Selection Matrix

| Use Case | Cloud Console | gcloud CLI | Deployment Manager | Terraform |
|----------|---------------|-----------|-------------------|-----------|
| Quick testing | ✓ | ✓ | - | - |
| Automation | - | ✓ | ✓ | ✓ |
| Multi-cloud | - | - | - | ✓ |
| GCP-native | ✓ | ✓ | ✓ | ✓ |
| Version control | - | ✓ | ✓ | ✓ |
| Templates | - | - | ✓ | ✓ |
| Learning curve | Easiest | Easy | Medium | Hard |

### Decision Tree

```
Need to manage infrastructure?
├─ YES: Need multi-cloud?
│   ├─ YES → Terraform
│   └─ NO: GCP only?
│       ├─ YES: Prefer YAML?
│       │   ├─ YES → Deployment Manager
│       │   └─ NO → Terraform
│       └─ Complex setup?
│           ├─ YES → Terraform
│           └─ NO → Deployment Manager
└─ NO: Need to query/monitor?
    ├─ Metrics → Cloud Monitoring
    └─ Logs → Cloud Logging
```

---

## Thực Hành & Scenarios

### Scenario 1: Multi-Tier Application with Terraform

**Requirements:**
- VPC network
- 2 subnets (web, database)
- Web servers (with autoscaling)
- Cloud SQL database
- Load balancer

**Setup:**

```hcl
# variables.tf
variable "project_id" {
  type = string
}

variable "region" {
  type    = string
  default = "us-central1"
}

variable "environment" {
  type    = string
  default = "production"
}

# main.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
  
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# VPC Network
resource "google_compute_network" "app_network" {
  name                    = "${var.environment}-network"
  auto_create_subnetworks = false
}

# Web Subnet
resource "google_compute_subnetwork" "web_subnet" {
  name          = "${var.environment}-web-subnet"
  ip_cidr_range = "10.0.1.0/24"
  region        = var.region
  network       = google_compute_network.app_network.id
  
  private_ip_google_access = true
}

# Database Subnet
resource "google_compute_subnetwork" "db_subnet" {
  name          = "${var.environment}-db-subnet"
  ip_cidr_range = "10.0.2.0/24"
  region        = var.region
  network       = google_compute_network.app_network.id
  
  private_ip_google_access = true
}

# Firewall - Allow HTTP/HTTPS
resource "google_compute_firewall" "allow_http" {
  name    = "${var.environment}-allow-http"
  network = google_compute_network.app_network.name

  allow {
    protocol = "tcp"
    ports    = ["80", "443"]
  }

  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["http-server"]
}

# Firewall - Allow internal communication
resource "google_compute_firewall" "allow_internal" {
  name    = "${var.environment}-allow-internal"
  network = google_compute_network.app_network.name

  allow {
    protocol = "tcp"
    ports    = ["0-65535"]
  }
  allow {
    protocol = "udp"
    ports    = ["0-65535"]
  }

  source_ranges = ["10.0.0.0/8"]
}

# Instance Template
resource "google_compute_instance_template" "web_template" {
  name_prefix = "web-template-"
  
  machine_type = "n1-standard-1"
  
  disk {
    source_image = "debian-cloud/debian-11"
    auto_delete  = true
  }

  network_interface {
    network    = google_compute_network.app_network.name
    subnetwork = google_compute_subnetwork.web_subnet.name
  }

  metadata_startup_script = file("${path.module}/startup-script.sh")

  tags = ["http-server"]
}

# Managed Instance Group
resource "google_compute_instance_group_manager" "web_igm" {
  name = "${var.environment}-web-igm"
  
  base_instance_name = "web-server"
  instance_template  = google_compute_instance_template.web_template.id
  
  zone = "${var.region}-a"
  
  target_size = 2
  
  auto_healing_policies {
    health_check      = google_compute_health_check.web_health.id
    initial_delay_sec = 300
  }
}

# Health Check
resource "google_compute_health_check" "web_health" {
  name = "${var.environment}-web-health"

  http_health_check {
    port = "80"
  }
}

# Autoscaling Policy
resource "google_compute_autoscaler" "web_autoscaler" {
  name   = "${var.environment}-web-autoscaler"
  target = google_compute_instance_group_manager.web_igm.id

  autoscaling_policy {
    min_replicas    = 2
    max_replicas    = 5
    cooldown_period = 60

    cpu_utilization {
      target = 0.6
    }
  }
}

# Cloud SQL
resource "google_sql_database_instance" "app_db" {
  name             = "${var.environment}-mysql-instance"
  database_version = "MYSQL_8_0"
  region           = var.region

  settings {
    tier              = "db-n1-standard-2"
    availability_type = "REGIONAL"

    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      transaction_log_retention_days = 7
    }

    ip_configuration {
      private_network = google_compute_network.app_network.id
      require_ssl     = true
    }
  }
}

# Cloud SQL Database
resource "google_sql_database" "app_db" {
  name     = "appdb"
  instance = google_sql_database_instance.app_db.name
}

# Cloud SQL User
resource "google_sql_user" "app_user" {
  name     = "appuser"
  instance = google_sql_database_instance.app_db.name
  password = random_password.db_password.result
}

# Random password for database
resource "random_password" "db_password" {
  length  = 16
  special = true
}

# Load Balancer (Backend Service)
resource "google_compute_backend_service" "web_backend" {
  name            = "${var.environment}-web-backend"
  health_checks   = [google_compute_health_check.web_health.id]
  protocol        = "HTTP"
  session_affinity = "NONE"

  backend {
    group           = google_compute_instance_group_manager.web_igm.instance_group
    balancing_mode  = "RATE"
    max_rate_per_endpoint = 100
  }
}

# URL Map
resource "google_compute_url_map" "web_lb" {
  name            = "${var.environment}-web-lb"
  default_service = google_compute_backend_service.web_backend.id
}

# HTTP Proxy
resource "google_compute_target_http_proxy" "web_http_proxy" {
  name    = "${var.environment}-web-http-proxy"
  url_map = google_compute_url_map.web_lb.id
}

# Forwarding Rule
resource "google_compute_global_forwarding_rule" "web_lb_rule" {
  name       = "${var.environment}-web-lb-rule"
  ip_version = "IPV4"
  load_balancing_scheme = "EXTERNAL"
  port_range = "80"
  target     = google_compute_target_http_proxy.web_http_proxy.id
}

# Outputs
output "load_balancer_ip" {
  value = google_compute_global_forwarding_rule.web_lb_rule.ip_address
}

output "database_private_ip" {
  value = google_sql_database_instance.app_db.private_ip_address
}

output "database_password" {
  value     = random_password.db_password.result
  sensitive = true
}
```

**Deploy:**
```bash
terraform init
terraform plan
terraform apply
```

---

### Scenario 2: Monitoring & Alerting Setup

**Requirements:**
- Monitor CPU utilization
- Alert if CPU > 80% for 5 minutes
- Send alerts to Slack

**Setup:**

```bash
# 1. Create notification channel (Slack)
gcloud alpha monitoring channels create \
  --display-name="Slack Alert" \
  --type=slack \
  --channel-labels=channel_name="#alerts"

# Get the channel ID from output

# 2. Create alert policy
CHANNEL_ID="projects/my-project/notificationChannels/123456"

gcloud alpha monitoring policies create \
  --notification-channels=$CHANNEL_ID \
  --display-name="High CPU Alert" \
  --condition-display-name="CPU > 80%" \
  --condition-threshold-value=0.8 \
  --condition-threshold-duration=300s \
  --metric-type=compute.googleapis.com/instance/cpu/utilization \
  --filter='resource.type="gce_instance"'

# Or via YAML config
gcloud monitoring policies create --policy-from-file=alert_policy.yaml
```

**alert_policy.yaml:**
```yaml
displayName: "High CPU Alert"
conditions:
  - displayName: "CPU > 80%"
    conditionThreshold:
      filter: |
        metric.type="compute.googleapis.com/instance/cpu/utilization"
        resource.type="gce_instance"
      comparison: COMPARISON_GT
      thresholdValue: 0.8
      duration: 300s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_MEAN

notificationChannels:
  - projects/my-project/notificationChannels/123456

alertStrategy:
  autoClose: 86400s
```

---

### Scenario 3: Logging & Exporting Logs

**Requirements:**
- Collect all ERROR logs
- Export to BigQuery for analysis
- Create metric from ERROR count

**Setup:**

```bash
# 1. Create BigQuery dataset
bq mk --dataset \
  --location=US \
  --description="Log dataset" \
  logs_dataset

# 2. Create log sink to BigQuery
gcloud logging sinks create error-to-bigquery \
  bigquery.googleapis.com/projects/my-project/datasets/logs_dataset \
  --log-filter='severity="ERROR"'

# 3. Grant permissions (if needed)
gcloud projects add-iam-policy-binding my-project \
  --member=serviceAccount:cloud-logs@system.gserviceaccount.com \
  --role=roles/bigquery.dataEditor

# 4. Create log-based metric
gcloud logging metrics create error_count \
  --description="Count of ERROR logs" \
  --log-filter='severity="ERROR"'

# 5. View logs in Cloud Logging
gcloud logging read 'severity="ERROR"' \
  --limit=50 \
  --format=json
```

---

## Summary Checklist

### Cloud Console & gcloud
- [ ] Know Cloud Console basic navigation
- [ ] Familiar with gcloud command structure
- [ ] Common gcloud commands (compute, storage, sql, container)
- [ ] Authentication (user account, service account)
- [ ] Configuration management (project, region, zone)
- [ ] Output filtering & formatting
- [ ] Cloud Shell basics

### Cloud Monitoring & Logging
- [ ] Automatic metric collection
- [ ] Dashboard creation
- [ ] Alert policies (threshold, notification channels)
- [ ] Cloud Logging basics
- [ ] Logs Explorer querying
- [ ] Log sinks (export to storage, BigQuery, Pub/Sub)
- [ ] Log-based metrics
- [ ] Structured logging best practices

### Deployment Manager & Terraform
- [ ] Infrastructure as Code benefits
- [ ] Deployment Manager (YAML/Jinja2)
- [ ] Terraform (HCL, multi-cloud)
- [ ] Terraform workflow (init, plan, apply, destroy)
- [ ] State management
- [ ] Variables & outputs
- [ ] Modules (reusability)
- [ ] When to use each tool

---

**End of Management Tools Guide**
