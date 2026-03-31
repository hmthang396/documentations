# Networking Services - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về 3 networking services chính của GCP, biết khi nào sử dụng, cấu hình, best practices, và exam scenarios.

---

## Mục Lục

1. [Virtual Private Cloud (VPC)](#1-virtual-private-cloud-vpc)
2. [Cloud Load Balancing](#2-cloud-load-balancing)
3. [Cloud Armor](#3-cloud-armor)
4. [So Sánh & Chọn Service](#so-sánh--chọn-service)
5. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Virtual Private Cloud (VPC)

### 1.1 Khái Niệm & Tổng Quan

**VPC là gì?**
- Virtual Private Cloud - private network environment
- Isolate resources trong GCP
- Global resource (tuy nhiên subnets là regional)
- Mỗi project có 1 default VPC
- Custom VPC cho specialized networking needs

**Tại sao dùng VPC?**
- Isolate resources (security)
- Control traffic flow (firewall rules)
- IP addressing scheme
- Network segmentation
- On-premises connectivity (VPN, Interconnect)
- Multi-tier architecture

**VPC Architecture:**

```
┌─────────────────────────────────────────────────────────┐
│                     VPC Network                         │
│  (Global, 1 per project)                               │
│                                                         │
│  ├─ Subnet (us-central1)  [Regional]                  │
│  │   ├─ Instance A (10.0.1.0/24)                      │
│  │   ├─ Instance B                                     │
│  │   └─ Cloud SQL                                      │
│  │                                                      │
│  ├─ Subnet (europe-west1)  [Regional]                 │
│  │   ├─ Instance C                                     │
│  │   └─ GKE Cluster                                    │
│  │                                                      │
│  └─ Routes                                             │
│      ├─ Default route (0.0.0.0/0 → Internet)          │
│      └─ Custom route                                   │
└─────────────────────────────────────────────────────────┘
```

---

### 1.2 VPCs & Subnets

#### Creating VPC

**Default VPC:**
```
Automatically created per project:
  Name: default
  Subnets: auto-created in each region
  Routes: default (0.0.0.0/0)
```

**Custom VPC:**
```bash
# Create custom VPC
gcloud compute networks create my-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Alternative: auto subnets
gcloud compute networks create my-vpc \
  --subnet-mode=auto \
  --bgp-routing-mode=regional
```

**VPC Modes:**
- **Auto mode**: Automatically creates subnets in each region
  - 10.0.0.0/8 block divided across regions
  - Simpler, but less control
  
- **Custom mode**: You define subnets manually
  - Full control over IP ranges
  - Can use any RFC 1918 private address space
  - Recommended for production

#### Subnet Structure

**Regional Resource:**
Each subnet belongs to exactly one region

```
VPC: my-vpc
├─ Subnet: us-central1
│   Region: us-central1
│   Zones: us-central1-a, us-central1-b, us-central1-c
│   IPv4 CIDR: 10.0.1.0/24
│   Available IPs: 251 (3 reserved: network, gateway, broadcast)
│
├─ Subnet: europe-west1
│   Region: europe-west1
│   Zones: europe-west1-b, europe-west1-c, europe-west1-d
│   IPv4 CIDR: 10.0.2.0/24
│   Available IPs: 251
```

**Creating Subnets:**

```bash
# Create subnet
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24

# List subnets
gcloud compute networks subnets list --network=my-vpc

# Get subnet details
gcloud compute networks subnets describe my-subnet \
  --region=us-central1
```

**Subnet Properties:**
- `name`: Subnet name
- `network`: Parent VPC
- `region`: Region (cannot change)
- `ipCidrRange`: Subnet CIDR block
- `secondaryIpRange`: Additional ranges (for GKE)
- `privateIpGoogleAccess`: Enable private access to Google APIs

#### Expanding CIDR Blocks (Only Increase)

```bash
# Expand subnet range (can only increase, not decrease)
gcloud compute networks subnets expand-ip-range my-subnet \
  --region=us-central1 \
  --prefix-length=23  # /24 → /23 (doubles capacity)
```

---

### 1.3 IP Addressing

#### Internal IP Addresses (Private)

**Primary IP:**
- Automatically assigned từ subnet CIDR range
- Hoặc manually assigned
- Persistent across stop/restart
- Cannot be changed after assignment (must recreate instance)

**Ephemeral IPs:**
- Temporary, assigned at boot
- Lost when instance stops
- Reassigned on restart

```bash
# Assign static internal IP
gcloud compute addresses create my-static-ip \
  --region=us-central1 \
  --subnet=my-subnet \
  --address=10.0.1.10

# Attach to instance
gcloud compute instances create my-instance \
  --private-network-ip=10.0.1.10
```

#### External IP Addresses (Public)

**Ephemeral IP:**
- Assigned when instance created
- Released when instance stops
- Reassigned on restart
- Free if instance running continuously
- Charged if reserved but not used

**Static IP:**
- Persists across stop/restart
- Reserved exclusively for you
- Charged even if not in use
- Can move between instances (same region)

```bash
# Reserve static external IP
gcloud compute addresses create my-external-ip \
  --region=us-central1

# Assign to instance
gcloud compute instances create my-instance \
  --address=my-external-ip

# Or attach to existing instance
gcloud compute instances delete-access-config my-instance \
  --access-config-name="external-nat"
gcloud compute instances add-access-config my-instance \
  --address=my-external-ip
```

**Cost Implications:**
```
Ephemeral external IP (continuously running): Free
Ephemeral external IP (not running): Charged
Static IP (in use): Free
Static IP (reserved but unused): Charged ($0.01/hour)
```

#### Alias IP Ranges (Secondary IPs)

Multiple IP ranges per subnet or network interface

```bash
# Create subnet with secondary range (for GKE)
gcloud compute networks subnets create my-subnet \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --secondary-range pods=10.4.0.0/14,services=10.0.16.0/20

# Create instance with alias IP
gcloud compute instances create my-instance \
  --network-interface=network=my-vpc,subnet=my-subnet,\
aliases=10.0.1.10/32,10.0.1.11/32
```

**Use Cases:**
- Multiple services per instance
- Container networking (GKE)
- Legacy applications with fixed IPs

#### Private Google Access

Allow instances to access Google APIs privately (without external IP)

```bash
# Enable Private Google Access
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access

# Now instances can access Cloud Storage, BigQuery, etc.
# without external IP
```

---

### 1.4 Routes & Routing

**Route Definition:**
Rules determining where traffic goes

#### Default Routes

```
Destination CIDR    Next Hop              Priority
─────────────────────────────────────────────────
10.0.0.0/24         local (in VPC)        0
127.0.0.1/32        local                 1000
169.254.169.254/32  local (metadata)      1000
0.0.0.0/0           internet gateway      1000
```

**Internet Gateway:**
- Default next hop for 0.0.0.0/0
- Allows egress to internet
- Requires external IP for return traffic

#### Custom Routes

**Destination-based routing:**

```bash
# Route to on-premises via VPN
gcloud compute routes create on-prem-route \
  --destination-range=192.168.0.0/16 \
  --next-hop-vpn-tunnel=my-vpn-tunnel \
  --priority=1000

# Route to specific instance
gcloud compute routes create route-to-instance \
  --destination-range=172.16.0.0/16 \
  --next-hop-instance=gateway-instance \
  --next-hop-instance-zone=us-central1-a

# Route to load balancer
gcloud compute routes create route-to-lb \
  --destination-range=203.0.113.0/24 \
  --next-hop-ilb=internal-load-balancer
```

#### Route Priority

Lower priority executes first
```
Priority 0:   Execute first
Priority 1000: Default routes
Priority 65534: Last resort
```

#### How Routing Works

```
Instance sends packet to destination
  ↓
Check routing table (from lowest to highest priority)
  ↓
Match destination with route
  ↓
Send to next hop:
  - local (within VPC)
  - internet gateway (external)
  - VPN tunnel (on-premises)
  - instance (forwarding)
  - load balancer
```

---

### 1.5 Firewall Rules

**Purpose:**
Control ingress (inbound) and egress (outbound) traffic

#### Default Firewall Behavior

```
Ingress: DENY ALL (except internal communication)
Egress: ALLOW ALL
```

#### Creating Firewall Rules

```bash
# Allow HTTP/HTTPS from internet
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server \
  --priority=1000

# Allow SSH from specific IP
gcloud compute firewall-rules create allow-ssh \
  --allow=tcp:22 \
  --source-ranges=203.0.113.0/24 \
  --target-tags=admin \
  --priority=1000

# Allow traffic from another VPC
gcloud compute firewall-rules create allow-from-internal \
  --allow=tcp:3306 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=database

# Deny egress (restrict outbound)
gcloud compute firewall-rules create deny-external-db \
  --direction=EGRESS \
  --allow=none \
  --deny=tcp:5432 \
  --destination-ranges=0.0.0.0/0 \
  --target-tags=restricted
```

#### Firewall Rule Components

```yaml
Name: allow-http
Direction: INGRESS
Priority: 1000
Action: ALLOW
Match:
  - Protocol: tcp
  - Port: 80, 443
Source: 0.0.0.0/0  (or specific IP range)
Target:
  - Tags: web-server  (or service account)
  - All instances in target

Disabled: false
```

**Detailed Components:**

| Component | Options | Notes |
|-----------|---------|-------|
| Direction | INGRESS, EGRESS | Ingress = inbound |
| Priority | 0-65534 | Lower = higher priority |
| Action | ALLOW, DENY | First match wins |
| Protocol | tcp, udp, icmp, esp | Or all |
| Port | 0-65535 | Or ranges: 80,443 or 1024-2048 |
| Source/Dest | IP ranges, tags, SAs | Multiple allowed |
| Target | Tags or SAs | Rule applies to these |

#### Network Tags vs Service Accounts

**Network Tags (simpler, IP-independent):**
```bash
# Create rule with tags
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --target-tags=web-server

# Apply tag to instance
gcloud compute instances create my-instance \
  --tags=web-server
```

**Service Accounts (better for large deployments):**
```bash
# Create service account
gcloud iam service-accounts create web-app-sa

# Create rule with service account
gcloud compute firewall-rules create allow-http \
  --allow=tcp:80 \
  --target-service-accounts=web-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Assign service account to instance
gcloud compute instances create my-instance \
  --service-account=web-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

#### Evaluating Firewall Rules

```
Packet arrives at instance
  ↓
Check firewall rules (by priority)
  ↓
First matching rule determines action
  ↓
If rule says ALLOW → Traffic passes
If rule says DENY → Traffic dropped
If no rule matches → Default action (DENY for ingress, ALLOW for egress)
```

**Example:**
```
Rules (by priority):
1. deny-ssh (priority 100): DENY tcp:22 from 0.0.0.0/0
2. allow-http (priority 1000): ALLOW tcp:80 from 0.0.0.0/0
3. allow-ssh (priority 2000): ALLOW tcp:22 from 10.0.0.0/8

Scenario 1: SSH from 203.0.113.1
  → Match rule 1 (deny-ssh)
  → DENIED

Scenario 2: HTTP from 203.0.113.1
  → Match rule 2 (allow-http)
  → ALLOWED

Scenario 3: SSH from 10.0.0.5
  → No match at priority 100
  → No match at priority 1000
  → Match rule 3 (allow-ssh)
  → ALLOWED
```

---

### 1.6 VPC Peering

**Purpose:**
Connect two VPC networks together

#### VPC Peering Characteristics

```
VPC A ←→ VPC B
├─ Direct connectivity (Google backbone)
├─ Low latency communication
├─ Private IP address ranges can overlap
├─ No overlap in CIDR blocks required!
├─ Bidirectional rules required
└─ Cross-region & cross-project supported
```

#### Creating VPC Peering

```bash
# 1. Create peering from VPC A → VPC B
gcloud compute networks peerings create peering-ab \
  --network=vpc-a \
  --peer-project=other-project-id \
  --peer-network=vpc-b \
  --auto-create-routes

# 2. In other project, create reverse peering
gcloud compute networks peerings create peering-ba \
  --network=vpc-b \
  --peer-project=original-project-id \
  --peer-network=vpc-a \
  --auto-create-routes

# 3. Verify peering is established
gcloud compute networks peerings list --network=vpc-a
gcloud compute networks peerings describe peering-ab \
  --network=vpc-a
```

#### Export/Import Custom Routes

By default, custom routes are NOT shared via peering

```bash
# Export custom routes from VPC A
gcloud compute networks peerings update peering-ab \
  --network=vpc-a \
  --export-custom-routes \
  --import-custom-routes

# Import routes from VPC B
gcloud compute networks peerings update peering-ab \
  --network=vpc-a \
  --import-custom-routes
```

#### Cost

```
Cross-region peering: Charged
  $0.02/GB in
  $0.02/GB out

Same-region peering: Free
```

---

### 1.7 Shared VPC

**Purpose:**
Share VPC network across multiple projects

#### Shared VPC Architecture

```
Organization
├─ Admin Project (Host Project)
│   └─ Shared VPC Network
│       └─ Subnets
│
├─ Project A (Service Project)
│   └─ Resources use Shared VPC
│
├─ Project B (Service Project)
│   └─ Resources use Shared VPC
│
└─ Project C (Service Project)
```

#### Enabling Shared VPC

**In Host Project:**

```bash
# 1. Enable Shared VPC on project
gcloud compute shared-vpc enable PROJECT_ID

# 2. Create VPC (will be shared)
gcloud compute networks create shared-vpc

# 3. Grant Service Projects access
gcloud compute shared-vpc associated-projects add PROJECT_A \
  --host-project=HOST_PROJECT_ID

gcloud compute shared-vpc associated-projects add PROJECT_B \
  --host-project=HOST_PROJECT_ID
```

**In Service Project:**
Resources can now use shared VPC subnets

```bash
# Create instance using shared VPC
gcloud compute instances create my-instance \
  --network=projects/HOST_PROJECT_ID/global/networks/shared-vpc \
  --subnet=projects/HOST_PROJECT_ID/regions/us-central1/subnetworks/shared-subnet
```

#### Permissions

**Host Project (Shared VPC Admin):**
- Can manage shared VPC
- Can delegate subnet creation to service projects

**Service Project (User):**
- Can create resources in shared subnets
- Cannot modify subnets

---

### 1.8 Cloud VPN

**Purpose:**
Securely connect on-premises networks to GCP VPC

#### VPN Fundamentals

```
On-Premises Network                GCP VPC
192.168.0.0/16                     10.0.0.0/16
       │                                  │
   [VPN Gateway]←──── IPSec ────→[VPN Gateway]
       │                                  │
       └── Encrypted Tunnel ──────────────┘
```

#### Creating Cloud VPN Connection

```bash
# 1. Create VPN gateway on GCP side
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

# 2. Create VPN tunnel
gcloud compute vpn-tunnels create my-tunnel \
  --vpn-gateway=my-vpn-gateway \
  --peer-address=203.0.113.1 \
  --shared-secret=YOUR_SECRET_KEY \
  --router=my-router \
  --region=us-central1

# 3. Create route for on-premises traffic
gcloud compute routes create route-to-onprem \
  --destination-range=192.168.0.0/16 \
  --next-hop-vpn-tunnel=my-tunnel \
  --next-hop-vpn-tunnel-region=us-central1
```

#### VPN Characteristics

```
High Availability: Can have multiple tunnels
Bandwidth: Limited by tunnel capacity (typically 1.5Gbps per tunnel)
Latency: Higher than Interconnect (IPSec overhead)
Cost: $0.30/tunnel/day + data charges
Redundancy: Active-active or active-passive
```

---

### 1.9 Cloud Interconnect

**Purpose:**
High-bandwidth, dedicated connection to GCP

#### Types

**Dedicated Interconnect:**
- Direct dedicated connection (10Gbps or 100Gbps)
- Very high availability
- Low latency
- Expensive (setup + monthly fee)
- Best for: Large data transfers, mission-critical connectivity

**Partner Interconnect:**
- Via service provider
- More affordable
- Better availability
- Easier to provision

#### Vs VPN

| Aspect | VPN | Dedicated Interconnect | Partner Interconnect |
|--------|-----|----------------------|----------------------|
| Bandwidth | 1.5 Gbps | 10-100 Gbps | Varies |
| Cost | Low | High | Medium |
| Latency | Higher | Lower | Lower |
| Setup Time | Hours | Weeks | Days |
| Failover | Auto | Manual | Manual |
| Use Case | Small to medium | Large scale | Medium scale |

---

### 1.10 VPC Flow Logs

**Purpose:**
Monitor network traffic (for troubleshooting, security, compliance)

#### Enabling Flow Logs

```bash
# Enable on subnet
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs

# Enable on VPC (all subnets)
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-flow-logs \
  --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sample-rate=0.5  # Log 50% of flows
```

#### Flow Log Data

Each log entry contains:
```
source IP, destination IP
source port, destination port
protocol
timestamp
bytes/packets sent
action (ACCEPT/DROP)
TCP flags
```

#### Use Cases

```
1. Troubleshooting connectivity issues
2. Identifying DDoS attacks
3. Monitoring inter-VPC traffic
4. Compliance auditing
5. Performance analysis
```

---

### 1.11 VPC Best Practices

#### Network Segmentation

```
Single VPC, Multiple Subnets by Purpose

VPC: production
├─ Subnet: web-tier (10.0.1.0/24)
│   └─ Web servers
│
├─ Subnet: app-tier (10.0.2.0/24)
│   └─ Application servers
│
└─ Subnet: db-tier (10.0.3.0/24)
    └─ Databases
```

#### Firewall Rule Best Practices

```
1. Principle of Least Privilege
   - Allow minimum necessary traffic
   - Deny by default
   - Whitelist access

2. Use Network Tags
   - More maintainable than IP lists
   - Tag-based rules

3. Separate rules by function
   - allow-http
   - allow-https
   - allow-ssh (restricted IPs)
   - allow-internal-rpc

4. Document rules
   - Add descriptions
   - Track business requirements
```

#### IP Address Planning

```
Use RFC 1918 Private Ranges:
- 10.0.0.0/8 (Class A)
- 172.16.0.0/12 (Class B)
- 192.168.0.0/16 (Class C)

Plan for growth:
- Document all subnets
- Reserve ranges for future
- Avoid overlapping ranges with on-premises
- Plan for VPC peering (can overlap but awkward)
```

---

### 1.12 VPC Exam Tips

**Key Concepts:**
- Global resource (subnets are regional)
- Auto vs Custom mode
- Internal & external IP addresses (ephemeral vs static)
- Firewall rules (priority, direction, action)
- Routes (destination-based routing)
- VPC peering (bidirectional, can cross projects)
- Shared VPC (centralized network)
- Private Google Access (access APIs privately)
- Network tags (better than IPs for firewall rules)

**Common Scenarios:**
1. "Allow SSH from specific IP" → Firewall rule with source IP + tcp:22
2. "Multiple projects share VPC" → Shared VPC
3. "Connect on-premises to GCP" → Cloud VPN or Interconnect
4. "Route traffic to specific instance" → Custom route
5. "Allow traffic between services in same VPC" → Internal firewall rules

---

## 2. Cloud Load Balancing

### 2.1 Khái Niệm & Tổng Quan

**Cloud Load Balancing là gì?**
- Fully managed load balancing service
- Distribute traffic across backends
- Global service (anycast IP)
- Multiple load balancer types
- Auto-scaling integration
- High availability & fault tolerance

**Tại sao dùng Cloud Load Balancing?**
- Distribute traffic to multiple backends
- Auto-failover if backend fails
- Session persistence/stickiness
- SSL/TLS termination
- Rate limiting & DDoS protection
- Global traffic distribution

**Load Balancer Hierarchy:**

```
Users (from internet)
       ↓
Forwarding Rule (IP:Port)
       ↓
Load Balancer (Layer 4 or 7)
       ↓
Backend Service/Pool
       ↓
Health Check
       ↓
Instances (could be auto-scaling)
```

---

### 2.2 Load Balancer Types

#### HTTP(S) Load Balancer (Layer 7)

**Characteristics:**
- Layer 7 (Application) load balancing
- URL-based routing
- SSL/TLS termination
- Global
- Best for: Web applications, APIs, microservices

**Routing Capabilities:**
```
Host-based:
  api.example.com → API backend service
  web.example.com → Web backend service

Path-based:
  example.com/api/* → API backend service
  example.com/* → Web backend service

Combined:
  api.example.com/v1/* → API v1 backend service
  api.example.com/v2/* → API v2 backend service
```

**Components:**
```
HTTP(S) Load Balancer
├─ Frontend (Forwarding Rule)
│   └─ IP:80/443
│
├─ URL Map (routing rules)
│   ├─ Host: *.example.com → backend service A
│   └─ Path: /api/* → backend service B
│
├─ Backend Services
│   ├─ Backend Service A (for web)
│   │   ├─ Instance Group
│   │   ├─ Health Check
│   │   └─ Session Affinity
│   │
│   └─ Backend Service B (for API)
│       ├─ Instance Group
│       ├─ Health Check
│       └─ Timeout
│
└─ SSL Certificate (for HTTPS)
```

#### Network Load Balancer (Layer 4)

**Characteristics:**
- Layer 4 (Transport) load balancing
- Ultra-high performance
- Very high throughput (millions of RPS)
- Extreme low latency
- Global
- Best for: Gaming, IoT, real-time applications

**When to use:**
- Extreme performance needed
- Non-HTTP protocols (TCP, UDP)
- Millions of requests per second
- Gaming servers
- Live video streaming

#### TCP/UDP Load Balancer (Layer 4)

**Internal:**
- Private IP address
- Regional service
- Internal traffic only
- VPC-native

**External:**
- Public IP address
- Global service
- Internet-facing

**TCP Load Balancer (Non-proxied):**
```
Direct connection, no termination
Lower latency, simpler
Good for: Database replication, custom protocols
```

**TCP Proxy:**
```
Connection termination
Higher latency, more features
Good for: SSL/TLS with custom protocols
```

---

### 2.3 Creating HTTP(S) Load Balancer

#### Step-by-Step Creation

**1. Create Backend Service**

```bash
# Create instance template
gcloud compute instance-templates create web-template \
  --machine-type=f1-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y apache2
    echo "Hello from $(hostname)" > /var/www/html/index.html'

# Create instance group (with autoscaling)
gcloud compute instance-groups managed create web-ig \
  --base-instance-name=web \
  --template=web-template \
  --size=2 \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-cpu-utilization=0.6

# Create health check
gcloud compute health-checks create http web-health-check \
  --port=80 \
  --request-path=/ \
  --check-interval=10s \
  --timeout=5s \
  --unhealthy-threshold=2 \
  --healthy-threshold=2

# Create backend service
gcloud compute backend-services create web-backend \
  --health-checks=web-health-check \
  --global \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --session-affinity=NONE \
  --timeout=30s

# Add instance group to backend service
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-ig \
  --instance-group-zone=us-central1-a \
  --global
```

**2. Create URL Map (Routing Rules)**

```bash
# Create URL map (basic)
gcloud compute url-maps create web-map \
  --default-service=web-backend

# Or advanced routing with host/path conditions
gcloud compute url-maps create api-map \
  --default-service=web-backend

# Add host rule
gcloud compute url-maps add-path-rule api-map \
  --service=api-backend \
  --path-matcher=api-paths \
  --paths=/api/*

# Add host rule
gcloud compute url-maps add-host-rule api-map \
  --hosts=api.example.com \
  --path-matcher=api-paths
```

**3. Create HTTPS Certificate (if using HTTPS)**

```bash
# Create self-signed certificate (for testing)
gcloud compute ssl-certificates create web-cert \
  --certificate=/path/to/cert.pem \
  --private-key=/path/to/key.pem

# Or use managed certificate
gcloud compute ssl-certificates create managed-cert \
  --domains=example.com,www.example.com
```

**4. Create Target HTTPS Proxy**

```bash
# Create HTTP proxy
gcloud compute target-http-proxies create web-proxy \
  --url-map=web-map

# Or HTTPS proxy
gcloud compute target-https-proxies create web-https-proxy \
  --url-map=web-map \
  --ssl-certificates=web-cert
```

**5. Create Forwarding Rule (Frontend)**

```bash
# Create forwarding rule (HTTP)
gcloud compute forwarding-rules create web-lb-rule \
  --global \
  --load-balancing-scheme=EXTERNAL \
  --target-http-proxy=web-proxy \
  --address=web-ip \
  --ports=80

# Or HTTPS
gcloud compute forwarding-rules create web-https-lb-rule \
  --global \
  --load-balancing-scheme=EXTERNAL \
  --target-https-proxy=web-https-proxy \
  --address=web-ip \
  --ports=443
```

**6. Verify Load Balancer**

```bash
# Get public IP
gcloud compute forwarding-rules describe web-lb-rule \
  --global

# Test
curl http://<PUBLIC_IP>
```

---

### 2.4 Health Checks

**Purpose:**
Determine if backend is healthy

#### Health Check Types

**HTTP/HTTPS:**
```bash
gcloud compute health-checks create http my-health-check \
  --port=80 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --unhealthy-threshold=2 \
  --healthy-threshold=2
```

**Configuration:**
```
port: Port to check
request-path: Path to request (default: /)
check-interval: Interval between checks (default: 10s)
timeout: Timeout for response (default: 5s)
unhealthy-threshold: Failures before marking unhealthy (default: 2)
healthy-threshold: Successes before marking healthy (default: 2)
```

**TCP:**
```bash
gcloud compute health-checks create tcp my-tcp-check \
  --port=3306
```

**Example Health Check Endpoint:**

```python
@app.route('/health')
def health():
    # Check database connectivity
    db.connection.ping()
    
    # Check disk space
    if disk_usage > 95:
        return {'status': 'unhealthy'}, 503
    
    return {'status': 'healthy'}, 200
```

#### Health Check Behavior

```
Instance created
  ↓
Health check request
  ↓
Response 200 → Mark HEALTHY
Response 500 → Mark UNHEALTHY
Timeout → Retry
  ↓
Unhealthy threshold reached → Remove from LB pool
  ↓
Requests stop going to this instance
  ↓
Instance continues to receive requests?
No - check graceful shutdown in app
```

---

### 2.5 Session Affinity (Stickiness)

**Purpose:**
Route requests from same client to same backend

#### Types

**CLIENT_IP (default):**
```
Route based on client IP
Pros: No session replication needed
Cons: Uneven distribution if many clients from same IP
```

**GENERATED_COOKIE:**
```
LB inserts cookie
Client sends cookie back
Routes to same backend
Pros: Even distribution
Cons: Client must accept cookies
```

**HTTP_COOKIE:**
```
Application-set cookie
LB uses value to route
Pros: Use existing cookies
Cons: Cookie value must be stable
```

**Configuration:**
```bash
gcloud compute backend-services update web-backend \
  --session-affinity=CLIENT_IP \
  --global
```

---

### 2.6 Content-Based Routing

#### Path-Based Routing

```yaml
# URL map with path matching
apiVersion: compute.cnrm.cloud.google.com/v1beta1
kind: ComputeUrlMap
metadata:
  name: api-url-map
spec:
  defaultService: web-backend
  hostRules:
  - hosts:
    - api.example.com
    pathMatcher: api-paths
  pathMatchers:
  - name: api-paths
    defaultService: web-backend
    pathRules:
    - paths:
      - /v1/users*
      service: api-v1-backend
    - paths:
      - /v2/users*
      service: api-v2-backend
    - paths:
      - /admin*
      service: admin-backend
```

#### Host-Based Routing

```bash
# Add host rule to URL map
gcloud compute url-maps add-host-rule api-map \
  --hosts=api.example.com \
  --path-matcher=api-paths

gcloud compute url-maps add-host-rule web-map \
  --hosts=www.example.com \
  --path-matcher=web-paths
```

---

### 2.7 Internal Load Balancer

**Characteristics:**
- Private IP address (internal VPC)
- Regional (not global)
- For internal traffic only
- Low latency (same region)

#### Creating Internal LB

```bash
# 1. Create health check
gcloud compute health-checks create http internal-health-check \
  --port=8080 \
  --request-path=/health

# 2. Create backend service (regional, internal)
gcloud compute backend-services create internal-backend \
  --health-checks=internal-health-check \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL \
  --protocol=TCP

# 3. Add backends
gcloud compute backend-services add-backend internal-backend \
  --instance-group=internal-ig \
  --instance-group-zone=us-central1-a \
  --region=us-central1

# 4. Create forwarding rule (internal)
gcloud compute forwarding-rules create internal-lb \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL \
  --backend-service=internal-backend \
  --subnet=my-subnet \
  --address=10.0.1.100
```

---

### 2.8 Traffic Splitting & Canary Deployments

**Purpose:**
Gradually shift traffic to new backend

#### URL Map with Traffic Split

```bash
# Create URL map with traffic splitting
gcloud compute url-maps add-path-rule my-map \
  --service=stable-backend \
  --path-matcher=canary \
  --paths=/api/*

# Add traffic split to URL map
gcloud compute url-maps add-path-rule my-map \
  --service=canary-backend \
  --path-matcher=canary-split \
  --paths=/api/* \
  --traffic-split=stable-backend:0.9,canary-backend:0.1
```

---

### 2.9 Load Balancer Exam Tips

**Key Concepts:**
- Global vs Regional (HTTP(S) = Global, Internal = Regional)
- Layer 4 vs Layer 7 (Network = Layer 4, HTTP(S) = Layer 7)
- Backend services (group of instances/services)
- Health checks (determine backend health)
- Session affinity/stickiness
- URL routing (host-based, path-based)
- Forwarding rules (frontend entry point)
- Traffic splitting (canary deployments)

**Common Scenarios:**
1. "Route API traffic separately from web" → HTTP(S) LB with host/path rules
2. "Ensure requests go to same backend" → Session affinity
3. "Gradually roll out new version" → Traffic splitting
4. "High-performance gaming server" → Network Load Balancer
5. "Internal service communication" → Internal Load Balancer

---

## 3. Cloud Armor

### 3.1 Khái Niệm & Tổng Quan

**Cloud Armor là gì?**
- DDoS protection & Web Application Firewall (WAF)
- Layer 7 (Application) protection
- Works with Cloud Load Balancing
- Rules-based (allow, deny, rate limit)
- Logging & monitoring

**Tại sao dùng Cloud Armor?**
- Protect against DDoS attacks
- Block malicious traffic
- Rate limiting
- Geographic restrictions
- Bot protection
- Compliance requirements

**Cloud Armor Workflow:**

```
Request arrives at LB
       ↓
Cloud Armor evaluates rules
       ↓
Match rule found?
       ↓
  ├─ ALLOW → Request passes to backend
  ├─ DENY → 403 Forbidden returned
  ├─ RATE_BASED_BAN → Check rate
  │   ├─ Within limit → Allow
  │   └─ Exceeded → Ban (429 Too Many Requests)
  └─ Challenge → CAPTCHA or other challenge
       ↓
Request logged & metrics updated
```

---

### 3.2 Security Policies

**Security Policy:**
Container for Cloud Armor rules

#### Creating Security Policy

```bash
# Create policy
gcloud compute security-policies create web-policy \
  --description="Policy for web servers"

# List policies
gcloud compute security-policies list

# Describe policy
gcloud compute security-policies describe web-policy
```

#### Attaching to Backend Service

```bash
# Attach security policy to backend service
gcloud compute backend-services update web-backend \
  --security-policy=web-policy \
  --global
```

---

### 3.3 Security Policy Rules

#### Rule Structure

```yaml
Priority: 1000
Match:
  VersionedExpr: "SQLi_v33"  # Predefined rules
Condition:
  - IPv4Address: [203.0.113.1]
  - RequestUri: [/admin/*]
Action: ALLOW | DENY | RATE_BASED_BAN | REDIRECT_TO_GOOGLE_RECAPTCHA | THROTTLE
```

#### Predefined Rule Sets

**OWASP ModSecurity Core Rule Set (CRS):**
Protection against:
- SQL injection
- Cross-site scripting (XSS)
- Local file inclusion
- Remote code execution
- PHP injection
- Session fixation

```bash
# Add predefined rules
gcloud compute security-policies rules create 1001 \
  --security-policy=web-policy \
  --action=deny-403 \
  --rules="sqli-v33-crs_v1_0_0-sqli_attack_sql_injection__30101,\
xss-v33-crs_v1_0_0-xss_attack_cross_site_scripting__30201"
```

#### Custom Rules

**IP-Based Blocking:**

```bash
# Block specific IP
gcloud compute security-policies rules create 100 \
  --security-policy=web-policy \
  --action=deny-403 \
  --description="Block malicious IP" \
  --match-expr="origin.ip in ['203.0.113.1', '203.0.113.2']"

# Block IP range
gcloud compute security-policies rules create 101 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="origin.ip in ['203.0.113.0/24']"
```

**Geographic Blocking:**

```bash
# Block specific countries
gcloud compute security-policies rules create 200 \
  --security-policy=web-policy \
  --action=deny-403 \
  --description="Block requests from high-risk countries" \
  --match-expr="origin.region_code in ['KP', 'IR']"
```

**Rate Limiting:**

```bash
# Rate limit by IP
gcloud compute security-policies rules create 300 \
  --security-policy=web-policy \
  --action=rate-based-ban \
  --rate-limit-options-exceed-action=deny-429 \
  --rate-limit-options-rate-limit-threshold-count=100 \
  --rate-limit-options-rate-limit-threshold-interval-sec=60 \
  --rate-limit-options-ban-duration-sec=600 \
  --description="Ban IPs with >100 requests/min"
```

**Header-Based Rules:**

```bash
# Block if missing user-agent
gcloud compute security-policies rules create 400 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="!has(request.headers['user-agent'])"

# Allow only specific user-agent
gcloud compute security-policies rules create 401 \
  --security-policy=web-policy \
  --action=allow \
  --match-expr="request.headers['user-agent'].contains('MyApp')"
```

**Path-Based Rules:**

```bash
# Protect admin paths
gcloud compute security-policies rules create 500 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="request.path.startsWith('/admin')" \
  --description="Block all /admin requests from internet"
```

---

### 3.4 CEL Expression Language

Cloud Armor uses CEL (Common Expression Language) for rules

**Common Expressions:**

```yaml
# IP-based
origin.ip in ['203.0.113.1', '203.0.113.2']
origin.ip.matches('203.0.113.*')

# Geographic
origin.region_code in ['US', 'CA', 'MX']

# Headers
request.headers['user-agent'].contains('bot')
!has(request.headers['x-forwarded-for'])

# Path
request.path.startsWith('/admin')
request.path == '/secret'
request.path.matches('.*\\.php$')

# Method
request.method == 'POST'
request.method in ['POST', 'PUT']

# Query parameters
request.query.contains('debug=true')

# Combinations
origin.ip in ['203.0.113.0/24'] && request.path.startsWith('/api')
(origin.region_code == 'CN' || origin.region_code == 'RU') && request.method == 'POST'
```

---

### 3.5 Actions

#### ALLOW

```
Allow traffic to backend
Counted in metrics
```

```bash
gcloud compute security-policies rules create 1000 \
  --security-policy=web-policy \
  --action=allow \
  --match-expr="origin.ip in ['10.0.0.0/8']"  # Allow internal
```

#### DENY

```
Block request
Return 403 Forbidden
Logged
```

```bash
gcloud compute security-policies rules create 1001 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="origin.region_code in ['KP']"
```

#### RATE_BASED_BAN

```
Count requests by key (IP, etc.)
If exceed threshold → Ban
Ban lasts specified duration
Return 429 Too Many Requests
```

```bash
gcloud compute security-policies rules create 1002 \
  --security-policy=web-policy \
  --action=rate-based-ban \
  --rate-limit-options-exceed-action=deny-429 \
  --rate-limit-options-rate-limit-threshold-count=100 \
  --rate-limit-options-rate-limit-threshold-interval-sec=60 \
  --rate-limit-options-ban-duration-sec=600
```

**Parameters:**
```
rate-limit-threshold-count: Max requests allowed (100)
rate-limit-threshold-interval-sec: Time window (60 seconds)
ban-duration-sec: How long to ban (600 seconds = 10 min)
exceed-action: Action when threshold exceeded (deny-429)
```

#### CHALLENGE

CAPTCHA or JavaScript challenge

```bash
gcloud compute security-policies rules create 1003 \
  --security-policy=web-policy \
  --action=redirect-to-google-recaptcha
```

**Behavior:**
```
User request arrives
  ↓
Cloud Armor challenges (CAPTCHA)
  ↓
User solves CAPTCHA
  ↓
Request sent to backend with token
  ↓
Backend receives request
```

#### REDIRECT_TO_GOOGLE_RECAPTCHA

Same as challenge (for CAPTCHA)

#### THROTTLE

Limit request rate (advanced rate limiting)

---

### 3.6 Rule Priority

Rules evaluated top-to-bottom

```
Priority 1000: First rule evaluated
Priority 1001: Second rule
...
Priority 65535: Last rule (default allow)
```

**Best Practice:**

```bash
# 1. Most restrictive (highest priority = lower number)
gcloud compute security-policies rules create 100 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="origin.ip in ['203.0.113.0/24']"

# 2. Rate limiting
gcloud compute security-policies rules create 1000 \
  --security-policy=web-policy \
  --action=rate-based-ban

# 3. Allow internal
gcloud compute security-policies rules create 2000 \
  --security-policy=web-policy \
  --action=allow \
  --match-expr="origin.ip in ['10.0.0.0/8']"

# 4. Default (highest priority number = evaluated last)
gcloud compute security-policies rules create 65535 \
  --security-policy=web-policy \
  --action=allow
```

---

### 3.7 Logging & Monitoring

#### Enabling Logging

```bash
# Logs sent to Cloud Logging automatically
# View in Cloud Console → Cloud Logging → Security Policies
```

#### Example Log Entry

```json
{
  "timestamp": "2024-04-28T12:34:56.789Z",
  "requestHeaders": {
    "user-agent": "bot/1.0",
    "x-forwarded-for": "203.0.113.1"
  },
  "requestUrl": "https://example.com/admin?debug=true",
  "requestMethod": "GET",
  "sourceIp": "203.0.113.1",
  "policyName": "web-policy",
  "matchedRulePriority": 100,
  "matchedRuleAction": "deny(403)",
  "matchedRuleCondition": "origin.ip in ['203.0.113.0/24']"
}
```

#### Creating Alerts

```bash
# Create alert for DDoS
gcloud monitoring policies create \
  --notification-channels=CHANNEL_ID \
  --display-name="High request rate from single IP" \
  --condition-display-name="Requests > 1000/min" \
  --metric-type=compute.googleapis.com/https/request_count \
  --filter-labels=policy_name=web-policy
```

---

### 3.8 DDoS Protection Strategies

#### Layer 3/4 DDoS (Network)

```
Volumetric attacks (UDP floods, DNS amplification)
├─ Handled by Google Cloud edge infrastructure (free)
└─ Automatic mitigation (doesn't reach backend)
```

#### Layer 7 DDoS (Application)

```
HTTP floods, SlowLoris, etc.
├─ Cloud Armor rate limiting
├─ Bot detection
└─ Geographic blocking
```

#### Multi-Layered Defense

```
1. Network Level (Free, Google backbone)
   - Absorb massive floods
   - Rate limiting at edge

2. Cloud Armor Level (Paid)
   - Rate limiting by IP
   - Geographic filtering
   - Bot detection

3. Application Level
   - Input validation
   - Caching
   - Graceful degradation

4. Infrastructure
   - Auto-scaling
   - Health checks
   - Failover
```

---

### 3.9 Bot Management

**Predefined Bot Rules:**

```bash
# Block known crawlers/bots
gcloud compute security-policies rules create 1000 \
  --security-policy=web-policy \
  --action=deny-403 \
  --rules="jsonwebtoken_blocklist_v1,\
malicious_bot_v1"
```

**Custom Bot Detection:**

```bash
# Block if missing JavaScript headers (bots often can't execute JS)
gcloud compute security-policies rules create 1001 \
  --security-policy=web-policy \
  --action=challenge \
  --match-expr="!has(request.headers['x-client-js']) && \
                request.headers['user-agent'].contains('bot')"
```

---

### 3.10 Use Cases

#### Scenario 1: Protect Admin Interface

```bash
# 1. Create rule - block non-internal IPs
gcloud compute security-policies rules create 100 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="request.path.startsWith('/admin') && \
                origin.ip !in ['203.0.113.0/24']"

# 2. Allow internal IPs
gcloud compute security-policies rules create 101 \
  --security-policy=web-policy \
  --action=allow \
  --match-expr="origin.ip in ['203.0.113.0/24']"
```

#### Scenario 2: API Rate Limiting

```bash
# Rate limit API endpoint
gcloud compute security-policies rules create 200 \
  --security-policy=api-policy \
  --action=rate-based-ban \
  --rate-limit-options-exceed-action=deny-429 \
  --rate-limit-options-rate-limit-threshold-count=1000 \
  --rate-limit-options-rate-limit-threshold-interval-sec=60 \
  --rate-limit-options-ban-duration-sec=3600 \
  --match-expr="request.path.startsWith('/api')"
```

#### Scenario 3: Geographic Blocking

```bash
# Block high-risk countries
gcloud compute security-policies rules create 300 \
  --security-policy=web-policy \
  --action=deny-403 \
  --match-expr="origin.region_code in ['KP', 'IR', 'SY']"
```

---

### 3.11 Cloud Armor Pricing

```
Policy management: Free
Rules: $5 per policy per month + $1 per rule per month
Requests evaluated: $0.75 per million requests
Rate limiting: Included
Logging: Standard Cloud Logging charges
```

**Example Cost:**
```
1 policy with 10 rules:
  Policy: $5
  Rules: $10 (10 × $1)
  
1 billion requests/month:
  Evaluation: $0.75 × 1000 = $750
  
Total: $765/month
```

---

### 3.12 Cloud Armor Exam Tips

**Key Concepts:**
- Layer 7 WAF & DDoS protection
- Works with Cloud Load Balancer
- Rules-based (match conditions, actions)
- CEL expression language for matching
- Actions: ALLOW, DENY, RATE_BASED_BAN, CHALLENGE
- Rule priority (lower number = evaluated first)
- Predefined rules (OWASP CRS)
- Logging & monitoring
- Rate limiting (by IP, duration)

**Common Scenarios:**
1. "Block requests from specific country" → Geographic rule + DENY
2. "Rate limit API" → RATE_BASED_BAN with threshold
3. "Protect admin interface" → Path rule + DENY for non-internal IPs
4. "Block known attack patterns" → Predefined OWASP rules
5. "Challenge suspicious traffic" → CHALLENGE rule with CAPTCHA

---

## So Sánh & Chọn Service

### VPC vs Load Balancing vs Cloud Armor

| Aspect | VPC | Load Balancing | Cloud Armor |
|--------|-----|-----------------|-------------|
| **Layer** | Network (L3-L4) | Transport/App (L4-L7) | Application (L7) |
| **Purpose** | Network isolation | Traffic distribution | Attack protection |
| **Scope** | All resources | Backends | Requests to LB |
| **Cost** | Free | Pay per rule/request | Pay per request |

### Decision Tree

```
Need network isolation?
├─ YES → VPC (subnets, firewall rules, routes)
└─ NO → Continue

Need to distribute traffic?
├─ YES → Cloud Load Balancer
│  ├─ URL routing needed? → HTTP(S) LB
│  ├─ Extreme performance? → Network LB
│  └─ Internal only? → Internal LB
└─ NO → Continue

Need DDoS/WAF protection?
├─ YES → Cloud Armor (attach to LB)
└─ NO → Done
```

---

## Thực Hành & Scenarios

### Scenario 1: Multi-Tier Application with Load Balancing

**Architecture:**
```
Internet
    ↓
Cloud Load Balancer (public IP)
    ↓
    ├─ /api/* → API Backend Service → API Instances
    └─ /* → Web Backend Service → Web Instances (Filestore for shared content)
```

**Setup:**

```bash
# 1. Create VPC
gcloud compute networks create app-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# 2. Create subnets
gcloud compute networks subnets create web-subnet \
  --network=app-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --enable-private-ip-google-access

gcloud compute networks subnets create api-subnet \
  --network=app-vpc \
  --region=us-central1 \
  --range=10.0.2.0/24 \
  --enable-private-ip-google-access

# 3. Create firewall rules
gcloud compute firewall-rules create allow-lb \
  --network=app-vpc \
  --allow=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server

gcloud compute firewall-rules create allow-internal \
  --network=app-vpc \
  --allow=tcp:0-65535,udp:0-65535 \
  --source-ranges=10.0.0.0/8 \
  --target-tags=internal

# 4. Create instance templates
gcloud compute instance-templates create web-template \
  --network=app-vpc \
  --subnet=web-subnet \
  --machine-type=f1-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server,internal \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y apache2
    echo "Web Server $(hostname)" > /var/www/html/index.html'

gcloud compute instance-templates create api-template \
  --network=app-vpc \
  --subnet=api-subnet \
  --machine-type=f1-micro \
  --image-family=debian-11 \
  --image-project=debian-cloud \
  --tags=http-server,internal \
  --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install -y python3 python3-pip
    pip3 install flask
    # API code here'

# 5. Create instance groups (with autoscaling)
gcloud compute instance-groups managed create web-ig \
  --base-instance-name=web \
  --template=web-template \
  --size=2 \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-cpu-utilization=0.6

gcloud compute instance-groups managed create api-ig \
  --base-instance-name=api \
  --template=api-template \
  --size=2 \
  --zone=us-central1-a \
  --enable-autoscaling \
  --min-num-replicas=2 \
  --max-num-replicas=5 \
  --target-cpu-utilization=0.6

# 6. Create health checks
gcloud compute health-checks create http web-health \
  --port=80 \
  --request-path=/

gcloud compute health-checks create http api-health \
  --port=8080 \
  --request-path=/health

# 7. Create backend services
gcloud compute backend-services create web-backend \
  --health-checks=web-health \
  --global \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --timeout=30s

gcloud compute backend-services create api-backend \
  --health-checks=api-health \
  --global \
  --load-balancing-scheme=EXTERNAL \
  --protocol=HTTP \
  --timeout=30s

# 8. Add backends
gcloud compute backend-services add-backend web-backend \
  --instance-group=web-ig \
  --instance-group-zone=us-central1-a \
  --global

gcloud compute backend-services add-backend api-backend \
  --instance-group=api-ig \
  --instance-group-zone=us-central1-a \
  --global

# 9. Create URL map with routing
gcloud compute url-maps create app-map \
  --default-service=web-backend

gcloud compute url-maps add-path-rule app-map \
  --service=api-backend \
  --path-matcher=api-paths \
  --paths=/api/*

# 10. Create HTTP proxy
gcloud compute target-http-proxies create app-proxy \
  --url-map=app-map

# 11. Create forwarding rule
gcloud compute forwarding-rules create app-rule \
  --global \
  --target-http-proxy=app-proxy \
  --address=app-ip \
  --ports=80

# 12. Get IP
gcloud compute forwarding-rules describe app-rule --global
```

---

### Scenario 2: Protect API with Cloud Armor

**Requirements:**
- Rate limit API to 100 requests/min per IP
- Block requests from certain countries
- Protect /admin path

**Setup:**

```bash
# 1. Create security policy
gcloud compute security-policies create api-policy

# 2. Block admin from internet
gcloud compute security-policies rules create 100 \
  --security-policy=api-policy \
  --action=deny-403 \
  --match-expr="request.path.startsWith('/admin') && \
                origin.ip !in ['203.0.113.0/24']"

# 3. Block specific countries
gcloud compute security-policies rules create 200 \
  --security-policy=api-policy \
  --action=deny-403 \
  --match-expr="origin.region_code in ['KP', 'IR']"

# 4. Rate limit API
gcloud compute security-policies rules create 1000 \
  --security-policy=api-policy \
  --action=rate-based-ban \
  --rate-limit-options-exceed-action=deny-429 \
  --rate-limit-options-rate-limit-threshold-count=100 \
  --rate-limit-options-rate-limit-threshold-interval-sec=60 \
  --rate-limit-options-ban-duration-sec=600

# 5. Attach to backend service
gcloud compute backend-services update api-backend \
  --security-policy=api-policy \
  --global

# 6. Test
# Simulate high traffic from single IP
for i in {1..150}; do
  curl http://<LB_IP>/api/users &
done
# Should get 429 Too Many Requests after 100 requests
```

---

### Scenario 3: VPC Peering for Multi-Project Setup

**Architecture:**
```
Project A (Production)
├─ VPC A (10.0.0.0/16)
└─ Instances

        ↕ VPC Peering

Project B (Analytics)
├─ VPC B (10.1.0.0/16)
└─ Analytics instances
```

**Setup:**

```bash
# In Project A
export PROJECT_A=project-a-id
export PROJECT_B=project-b-id

# 1. Create VPC A
gcloud compute networks create vpc-a \
  --subnet-mode=custom \
  --project=$PROJECT_A

gcloud compute networks subnets create subnet-a \
  --network=vpc-a \
  --region=us-central1 \
  --range=10.0.0.0/16 \
  --project=$PROJECT_A

# 2. Create VPC B (in other project)
gcloud compute networks create vpc-b \
  --subnet-mode=custom \
  --project=$PROJECT_B

gcloud compute networks subnets create subnet-b \
  --network=vpc-b \
  --region=us-central1 \
  --range=10.1.0.0/16 \
  --project=$PROJECT_B

# 3. Create peering A→B (in Project A)
gcloud compute networks peerings create peering-a-to-b \
  --network=vpc-a \
  --peer-project=$PROJECT_B \
  --peer-network=vpc-b \
  --auto-create-routes \
  --project=$PROJECT_A

# 4. Create peering B→A (in Project B)
gcloud compute networks peerings create peering-b-to-a \
  --network=vpc-b \
  --peer-project=$PROJECT_A \
  --peer-network=vpc-a \
  --auto-create-routes \
  --project=$PROJECT_B

# 5. Create instances in both VPCs
gcloud compute instances create instance-a \
  --network=vpc-a \
  --subnet=subnet-a \
  --zone=us-central1-a \
  --project=$PROJECT_A

gcloud compute instances create instance-b \
  --network=vpc-b \
  --subnet=subnet-b \
  --zone=us-central1-a \
  --project=$PROJECT_B

# 6. Test connectivity
gcloud compute ssh instance-a --zone=us-central1-a --project=$PROJECT_A
# From instance-a, ping instance-b (10.1.x.x)
ping 10.1.0.10  # Should work
```

---

### Scenario 4: Internal Load Balancer for Database Access

**Architecture:**
```
Application tier (Web)
    ↓
Internal Load Balancer (10.0.2.100:3306)
    ↓
Database instances
```

**Setup:**

```bash
# 1. Create database instances
gcloud compute instances create db-1 \
  --network=app-vpc \
  --subnet=db-subnet \
  --zone=us-central1-a \
  --tags=database

gcloud compute instances create db-2 \
  --network=app-vpc \
  --subnet=db-subnet \
  --zone=us-central1-b \
  --tags=database

# 2. Allow traffic to database
gcloud compute firewall-rules create allow-mysql \
  --network=app-vpc \
  --allow=tcp:3306 \
  --source-tags=web \
  --target-tags=database

# 3. Create health check for MySQL
gcloud compute health-checks create tcp db-health \
  --port=3306

# 4. Create backend service (regional, internal)
gcloud compute backend-services create db-backend \
  --health-checks=db-health \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL \
  --protocol=TCP

# 5. Add backends
gcloud compute backend-services add-backend db-backend \
  --instance-group=db-ig \
  --instance-group-zone=us-central1-a \
  --region=us-central1

# 6. Create forwarding rule (internal)
gcloud compute forwarding-rules create db-lb \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL \
  --backend-service=db-backend \
  --subnet=db-subnet \
  --address=10.0.2.100

# 7. Web instances connect to 10.0.2.100:3306 (internal LB)
# No single point of failure!
```

---

## Summary Checklist

### VPC
- [ ] Understand global vs regional (VPC global, subnets regional)
- [ ] Auto vs Custom subnet mode
- [ ] Internal & external IP addresses (ephemeral vs static)
- [ ] Private Google Access (access APIs without external IP)
- [ ] Firewall rules (priority, direction, action, targets)
- [ ] Network tags (better than IPs for firewall rules)
- [ ] Routes (destination-based routing)
- [ ] VPC Peering (bidirectional, cross-project)
- [ ] Shared VPC (multi-project)
- [ ] Cloud VPN (on-premises connectivity)
- [ ] VPC Flow Logs (traffic monitoring)

### Cloud Load Balancing
- [ ] HTTP(S) LB (Layer 7, global)
- [ ] Network LB (Layer 4, extreme performance)
- [ ] Internal LB (regional, private)
- [ ] Backend services (group of instances/services)
- [ ] Health checks (HTTP, TCP)
- [ ] URL routing (host-based, path-based)
- [ ] Session affinity/stickiness
- [ ] Forwarding rules (frontend entry point)
- [ ] Traffic splitting (canary deployments)
- [ ] SSL/TLS termination
- [ ] Pricing model

### Cloud Armor
- [ ] Layer 7 DDoS/WAF protection
- [ ] Works with Cloud Load Balancer
- [ ] Security policies & rules
- [ ] CEL expression language
- [ ] Actions: ALLOW, DENY, RATE_BASED_BAN, CHALLENGE
- [ ] Rule priority (lower = evaluated first)
- [ ] Predefined rules (OWASP CRS)
- [ ] Rate limiting (by IP, duration)
- [ ] Geographic blocking
- [ ] Logging & monitoring
- [ ] Pricing

---

**End of Networking Services Guide**
