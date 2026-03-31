# Security & Identity - Hướng Dẫn Chi Tiết Cho Associate Cloud Engineer

**Mục tiêu**: Hiểu sâu về Cloud IAM và Cloud KMS, biết cách cấu hình access control, quản lý keys, encryption, best practices, và exam scenarios.

---

## Mục Lục

1. [Cloud IAM (Identity & Access Management)](#1-cloud-iam-identity--access-management)
2. [Cloud Key Management Service (KMS)](#2-cloud-key-management-service-kms)
3. [So Sánh & Best Practices](#so-sánh--best-practices)
4. [Thực Hành & Scenarios](#thực-hành--scenarios)

---

## 1. Cloud IAM (Identity & Access Management)

### 1.1 Khái Niệm & Tổng Quan

**Cloud IAM là gì?**
- Manage WHO has access to WHAT resources and HOW
- Identity: WHO (users, groups, service accounts)
- Resource: WHAT (Compute Engine, Cloud Storage, etc.)
- Role: HOW (permissions granted)

**Core Concept:**
```
Identity + Role + Resource = Access
```

**Principles:**
```
1. Principle of Least Privilege
   - Grant minimum permissions needed
   - Start restrictive, add permissions as needed
   
2. Identity should be verified
   - Know who is accessing
   
3. Access is auditable
   - Log who did what, when
```

---

### 1.2 IAM Concepts

#### Identities (WHO)

**Google Account:**
- Personal Google account (alice@gmail.com)
- Full Google account with password, 2FA

**Service Account:**
- Non-human account (my-app@project.iam.gserviceaccount.com)
- For applications, services, automated tasks
- Has keys (JSON), not passwords

**Google Group:**
- Collection of accounts
- Can grant roles to entire group

**Cloud Identity / Workspace:**
- Enterprise identity management
- User & group management
- Directory sync

**Example Identities:**
```
Users:
  user:alice@example.com
  user:bob@example.com

Groups:
  group:developers@example.com
  group:admins@example.com

Service Accounts:
  serviceAccount:my-app@project.iam.gserviceaccount.com
  serviceAccount:cloud-run@project.iam.gserviceaccount.com

Special:
  allUsers (anyone on internet)
  allAuthenticatedUsers (any GCP user)
```

#### Resources (WHAT)

Every GCP service has resources:
```
Project → Org → Folder
  ├─ Compute Engine instances
  ├─ Cloud Storage buckets
  ├─ Cloud SQL instances
  ├─ GKE clusters
  └─ etc.
```

**Resource Hierarchy:**
```
Organization
└─ Folders
    └─ Projects
        └─ Resources (instances, buckets, etc.)
```

#### Roles (HOW)

Collections of permissions

**Predefined Roles:**
```
Format: roles/service.roleName

Examples:
  roles/viewer                           # Read-only
  roles/editor                           # Read+write+delete
  roles/owner                            # Full control
  
  roles/compute.admin                    # Compute Engine admin
  roles/storage.objectViewer             # Cloud Storage viewer
  roles/cloudsql.admin                   # Cloud SQL admin
  roles/container.admin                  # GKE admin
  
  roles/iam.serviceAccountUser           # Can use service account
  roles/iam.workloadIdentityUser         # Workload Identity
```

**Custom Roles:**
- Define custom set of permissions
- Useful for specific job functions

**Basic Roles (Legacy, not recommended):**
```
roles/viewer     → Viewer role
roles/editor     → Editor role
roles/owner      → Owner role

❌ Don't use these in production!
→ Use predefined roles instead (more granular)
```

**Role Categories:**

1. **Viewer Role**
   - Read-only access
   - Cannot modify resources
   - Example: roles/storage.objectViewer

2. **Editor Role**
   - Read + write + delete
   - Can modify resources
   - Example: roles/compute.instanceAdmin

3. **Admin Role**
   - Full administrative access
   - Can manage IAM bindings
   - Example: roles/compute.admin

4. **Service-Specific Roles**
   - Fine-grained permissions
   - Example: roles/storage.objectCreator (can create but not delete)

---

### 1.3 IAM Bindings (Policy)

**IAM Policy:**
Binding of identities to roles on a resource

```yaml
# IAM Policy Structure
bindings:
  - members:
      - user:alice@example.com
      - serviceAccount:app@project.iam.gserviceaccount.com
    role: roles/compute.instanceAdmin.v1

  - members:
      - group:developers@example.com
    role: roles/storage.objectViewer

  - members:
      - allUsers
    role: roles/viewer
```

#### Granting Roles

**Via gcloud:**

```bash
# Grant role to user
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/compute.admin

# Grant role to service account
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-app@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Grant role to group
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=group:developers@example.com \
  --role=roles/editor

# Grant to allUsers (public access, use carefully!)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=allUsers \
  --role=roles/viewer
```

**Via Console:**
```
Cloud Console → IAM & Admin → IAM
→ Grant Access
→ Select principal
→ Assign role
→ Save
```

#### Listing & Revoking Roles

```bash
# List all IAM bindings
gcloud projects get-iam-policy PROJECT_ID

# Revoke role
gcloud projects remove-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/compute.admin

# List roles for specific user
gcloud projects get-iam-policy PROJECT_ID \
  --flatten="bindings[].members" \
  --filter="bindings.members:user:alice@example.com" \
  --format="table(bindings.role)"
```

---

### 1.4 Service Accounts

**Service Account = Non-human identity**

Used by:
- Applications
- Services
- Automated tasks
- GKE pods
- Cloud Functions
- Cloud Run

#### Creating Service Accounts

```bash
# Create service account
gcloud iam service-accounts create my-app-sa \
  --display-name="My Application Service Account" \
  --description="Used by my-app for Cloud SQL access"

# List service accounts
gcloud iam service-accounts list

# Get service account details
gcloud iam service-accounts describe my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

#### Service Account Keys

Methods to authenticate as service account:

**JSON Key File (Traditional):**
```bash
# Create key
gcloud iam service-accounts keys create ~/my-key.json \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Use in application
export GOOGLE_APPLICATION_CREDENTIALS="~/my-key.json"

# List keys
gcloud iam service-accounts keys list \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Delete key (revoke)
gcloud iam service-accounts keys delete KEY_ID \
  --iam-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

**Security Best Practices for Keys:**
```
✓ Rotate keys regularly (annual minimum)
✓ Store securely (not in version control!)
✓ Use secret management (Secret Manager)
✓ Limit key scope (use Workload Identity if possible)
✓ Monitor key usage (Cloud Audit Logs)

✗ Don't hardcode keys in code
✗ Don't commit to Git
✗ Don't share keys
✗ Don't use same key for all services
```

#### Granting Roles to Service Accounts

```bash
# Grant storage viewer role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# Grant Cloud SQL client role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/cloudsql.client
```

#### Using Service Accounts

**In Compute Engine:**
```bash
# Create instance with service account
gcloud compute instances create my-instance \
  --service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com

# Access GCP services automatically
# (No need to pass credentials in code)
```

**In Cloud Run:**
```bash
# Deploy service with specific service account
gcloud run deploy my-service \
  --image=gcr.io/PROJECT_ID/my-service:latest \
  --service-account=my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

**In GKE (Workload Identity):**
```yaml
# Kubernetes service account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-ksa
  namespace: default

---
# Pod using service account
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app-ksa
  containers:
  - name: app
    image: gcr.io/PROJECT_ID/my-app:latest
```

```bash
# Bind Kubernetes SA to GCP SA (Workload Identity)
gcloud iam service-accounts add-iam-policy-binding \
  my-app-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[default/my-app-ksa]"

# Annotate Kubernetes SA
kubectl annotate serviceaccount my-app-ksa \
  iam.gke.io/gcp-service-account=my-app-gsa@PROJECT_ID.iam.gserviceaccount.com
```

#### Service Account Impersonation

Allow another identity to assume service account role

```bash
# Grant alice permission to impersonate service account
gcloud iam service-accounts add-iam-policy-binding \
  my-app-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser \
  --member=user:alice@example.com

# Alice can now access as my-app-sa
gcloud config set account my-app-sa@PROJECT_ID.iam.gserviceaccount.com
```

---

### 1.5 Resource-Level IAM

Grant roles at specific resource level (not just project)

```bash
# Grant role on specific bucket
gsutil iam ch user:alice@example.com:objectViewer \
  gs://my-bucket

# Or via gcloud
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member=user:alice@example.com \
  --role=roles/storage.objectViewer

# Grant role on specific instance
gcloud compute instances add-iam-policy-binding my-instance \
  --zone=us-central1-a \
  --member=user:alice@example.com \
  --role=roles/compute.osLogin

# Grant role on specific Cloud SQL instance
gcloud sql instances patch my-db \
  --database-flags=cloudsql_iam_authentication=on

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=roles/cloudsql.instanceUser
```

---

### 1.6 Roles at Different Levels

IAM inheritance: Parent permissions apply to children

```
Organization Level
  └─ Folder Level (inherit from org)
      └─ Project Level (inherit from folder)
          └─ Resource Level (inherit from project)
```

**Example:**
```
Organization IAM:
  └─ group:admins@example.com → roles/resourcemanager.organizationAdmin

Folder IAM (inherits org bindings):
  └─ group:team-a@example.com → roles/resourcemanager.folderEditor

Project IAM (inherits folder bindings):
  └─ user:alice@example.com → roles/editor

Resource IAM (inherits project bindings):
  └─ user:bob@example.com → roles/storage.objectCreator
```

**Result:**
```
admins@example.com: Can do everything (org level)
team-a@example.com: Can edit folders & projects
alice@example.com: Can edit project resources
bob@example.com: Can create objects in specific bucket
```

---

### 1.7 Custom Roles

Create roles with specific permissions

```bash
# Create custom role from YAML
cat > custom-role.yaml << EOF
title: "Custom Database Admin"
description: "Admin access for Cloud SQL only"
includedPermissions:
  - cloudsql.instances.get
  - cloudsql.instances.list
  - cloudsql.instances.update
  - cloudsql.databases.create
  - cloudsql.databases.delete
  - cloudsql.users.create
  - cloudsql.users.delete
EOF

# Create role
gcloud iam roles create customDatabaseAdmin \
  --project=PROJECT_ID \
  --file=custom-role.yaml

# Grant custom role
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=user:alice@example.com \
  --role=projects/PROJECT_ID/roles/customDatabaseAdmin

# Update role
gcloud iam roles update customDatabaseAdmin \
  --project=PROJECT_ID \
  --file=custom-role.yaml

# Delete role
gcloud iam roles delete customDatabaseAdmin \
  --project=PROJECT_ID
```

---

### 1.8 Cloud Audit Logs

**Track WHO did WHAT, WHEN, WHERE**

Three types:

**Admin Activity Audit Logs:**
- API calls that modify resources
- Always enabled
- Retained 400 days
- Example: Create instance, Delete bucket

**Data Access Audit Logs:**
- Read operations
- Not enabled by default
- Higher cost
- Example: Read file from Cloud Storage

**System Events Audit Logs:**
- Events not initiated by user
- Always enabled
- Example: GKE scaling, Cloud SQL failover

#### Viewing Audit Logs

```bash
# View all audit logs
gcloud logging read "resource.type=gce_instance" \
  --limit=50 \
  --format=json

# View only admin activity
gcloud logging read "protoPayload.methodName=compute.instances.insert" \
  --limit=50

# View data access logs
gcloud logging read "protoPayload.resourceName=~\"storage.googleapis.com/.*\"" \
  --limit=50

# Export audit logs
gcloud logging sinks create audit-sink \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/audit_logs \
  --log-filter='severity="ERROR" OR protoPayload.status.code!=0'
```

#### Enable Data Access Logs

```bash
# Set IAM policy to enable data access logging
gcloud projects get-iam-policy PROJECT_ID \
  --format=json > policy.json

# Edit policy.json to add auditConfigs
# Then update:
gcloud projects set-iam-policy PROJECT_ID policy.json
```

---

### 1.9 Cloud IAM Best Practices

**1. Principle of Least Privilege**
```
✓ Grant minimum permissions needed
✓ Review permissions regularly
✓ Remove unnecessary permissions
✓ Use custom roles if predefined too broad
```

**2. Use Service Accounts**
```
✓ For applications, not users
✓ Rotate keys regularly
✓ Use Workload Identity in GKE
✓ Monitor service account usage
```

**3. Use Groups**
```
✓ Manage permissions via groups
✓ Easier to manage access at scale
✓ Easier to onboard/offboard users
```

**4. Separation of Duties**
```
✓ Different roles for different responsibilities
✓ No single person can approve & execute critical changes
✓ Use custom roles to enforce separation
```

**5. Audit & Monitor**
```
✓ Enable audit logs
✓ Review who has admin access
✓ Alert on suspicious access patterns
✓ Regular access reviews
```

**6. Workload Identity**
```
✓ Use Workload Identity in GKE
✓ Avoids key management
✓ More secure than keys
```

**7. Deny Policies (Advanced)**
```
✓ Explicit deny overrides allow
✓ Use for exceptions & quarantine
✓ Prevent accidental changes
```

---

### 1.10 Cloud IAM Exam Tips

**Key Concepts:**
- Identity, Resource, Role, Binding
- Service accounts (non-human accounts)
- Predefined roles (Viewer, Editor, Admin)
- Principle of least privilege
- Resource-level IAM
- IAM inheritance
- Custom roles
- Audit logs
- Workload Identity

**Common Scenarios:**
1. "Grant storage read access to user" → roles/storage.objectViewer
2. "App needs Cloud SQL access" → Create service account + role
3. "Team needs to manage project" → Group + roles/editor
4. "Workload Identity in GKE" → Bind KSA to GSA
5. "Track who accessed data" → Enable data access logs

---

## 2. Cloud Key Management Service (KMS)

### 2.1 Khái Niệm & Tổng Quan

**Cloud KMS là gì?**
- Managed encryption key service
- Create, manage, use encryption keys
- FIPS 140-2 compliant
- Audit all key usage
- Rotate keys automatically

**Why Use KMS?**
```
✓ Centralized key management
✓ Reduced operational burden
✓ Compliance (SOC 2, PCI-DSS, HIPAA)
✓ Automatic key rotation
✓ Audit trail of all key usage
✓ Hardware security module (HSM) option
```

**Encryption Types:**
```
1. Server-Side Encryption (SSE)
   - Google encrypts data
   - You manage keys (or Google manages)

2. Client-Side Encryption
   - You encrypt before sending
   - Google stores encrypted data

3. Customer-Managed Encryption Keys (CMEK)
   - You manage encryption keys via KMS
   - Google performs encryption/decryption
```

---

### 2.2 KMS Concepts

**Key Hierarchy:**

```
KMS Resource Hierarchy
├─ Project (billing unit)
├─ KeyRing (container for keys)
│   └─ Key (cryptographic key)
│       └─ Key Version (actual key material)
│           ├─ Primary (active)
│           └─ Replica (for rollback, asymmetric)
└─ Location (regional, multi-regional)
```

**Key Terminology:**

1. **Key Ring:**
   - Container for keys
   - Regional resource
   - Cannot change location
   - Cannot be renamed/deleted

2. **Key:**
   - Cryptographic key
   - Different purposes:
     - Encrypt/Decrypt
     - Sign/Verify
     - Asymmetric keys
   - Has multiple versions

3. **Key Version:**
   - Actual key material
   - Created on rotation
   - Can be destroyed (but not immediately)

---

### 2.3 Creating Keys

```bash
# 1. Create KeyRing
gcloud kms keyrings create my-keyring \
  --location=us-central1

# 2. Create encryption key
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption

# Or create with rotation schedule
gcloud kms keys create my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption \
  --rotation-schedule-duration=7776000s  # 90 days

# 3. List keyrings
gcloud kms keyrings list --location=us-central1

# 4. List keys
gcloud kms keys list \
  --location=us-central1 \
  --keyring=my-keyring

# 5. Get key version info
gcloud kms keys versions list \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key
```

---

### 2.4 Encrypting & Decrypting Data

#### Using gcloud

```bash
# Encrypt plaintext
echo "Secret data" | gcloud kms encrypt \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key \
  --plaintext-file=- \
  --ciphertext-file=-

# Decrypt ciphertext
gcloud kms decrypt \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key \
  --ciphertext-file=encrypted.txt \
  --plaintext-file=decrypted.txt
```

#### Using Python

```python
from google.cloud import kms

def encrypt_with_kms(project_id, location, key_ring, key, plaintext):
    client = kms.KeyManagementServiceClient()
    
    key_path = client.crypto_key_path(
        project_id, location, key_ring, key
    )
    
    response = client.encrypt(
        request={
            "name": key_path,
            "plaintext": plaintext.encode("utf-8"),
        }
    )
    
    return response.ciphertext

def decrypt_with_kms(project_id, location, key_ring, key, ciphertext):
    client = kms.KeyManagementServiceClient()
    
    key_path = client.crypto_key_path(
        project_id, location, key_ring, key
    )
    
    response = client.decrypt(
        request={
            "name": key_path,
            "ciphertext": ciphertext,
        }
    )
    
    return response.plaintext.decode("utf-8")

# Usage
plaintext = "Secret database password"
ciphertext = encrypt_with_kms("my-project", "us-central1", "my-keyring", "my-key", plaintext)
decrypted = decrypt_with_kms("my-project", "us-central1", "my-keyring", "my-key", ciphertext)
```

---

### 2.5 Key Rotation

**Automatic Rotation:**
```bash
# Create key with auto-rotation
gcloud kms keys create auto-rotating-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=encryption \
  --rotation-schedule-duration=7776000s  # 90 days

# Update rotation schedule
gcloud kms keys update my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --rotation-schedule-duration=2592000s  # 30 days
```

**Manual Rotation:**
```bash
# Create new key version (manual rotation)
gcloud kms keys versions create \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key

# Set primary version
gcloud kms keys versions update VERSION_ID \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key \
  --state=primary

# Destroy old version (after delay)
gcloud kms keys versions destroy VERSION_ID \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key
```

**Rotation Frequency:**
```
Google Cloud storage services: Every 90 days (auto)
Custom applications: Every 30-90 days (recommended)
High-security apps: Every 30 days minimum
```

---

### 2.6 IAM for KMS

Grant permissions to use keys

**Common Roles:**
```
roles/cloudkms.admin
  - Full KMS management

roles/cloudkms.cryptoKeyEncrypterDecrypter
  - Use key to encrypt/decrypt
  - Most common for applications

roles/cloudkms.cryptoKeyDecrypter
  - Only decrypt (read-only)

roles/cloudkms.publicKeyGetter
  - For asymmetric keys
```

#### Setting Up Key Access

```bash
# Grant application permission to use key
gcloud kms keys add-iam-policy-binding my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --member=serviceAccount:my-app@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter

# Grant user permission to decrypt
gcloud kms keys add-iam-policy-binding my-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --member=user:alice@example.com \
  --role=roles/cloudkms.cryptoKeyDecrypter
```

---

### 2.7 Using KMS with GCP Services

#### Cloud Storage Encryption

```bash
# Create bucket with CMEK
gsutil mb -b on \
  -c STANDARD \
  -l us-central1 \
  -p PROJECT_ID \
  gs://my-encrypted-bucket

# Set encryption key
gcloud storage buckets update gs://my-encrypted-bucket \
  --default-encryption-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key
```

#### Cloud SQL Encryption

```bash
# Create SQL instance with CMEK
gcloud sql instances create my-db \
  --database-version=POSTGRES_15 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --kms-key-name=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key

# Or update existing instance
gcloud sql instances patch my-db \
  --kms-key-name=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key
```

#### Persistent Disk Encryption

```bash
# Create disk with CMEK
gcloud compute disks create my-disk \
  --size=100GB \
  --type=pd-ssd \
  --kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key \
  --zone=us-central1-a
```

#### BigQuery Encryption

```bash
# Create dataset with CMEK
bq mk \
  --location=us-central1 \
  --kms-key=projects/PROJECT_ID/locations/us-central1/keyRings/my-keyring/cryptoKeys/my-key \
  my_dataset
```

---

### 2.8 Key Destruction & Recovery

**Key Version Destruction:**
```bash
# Schedule key version for destruction
gcloud kms keys versions destroy VERSION_ID \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key

# Key version enters 30-day destruction window
# Can be restored during this time

# Cancel destruction (restore)
gcloud kms keys versions restore VERSION_ID \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=my-key

# After 30 days: Automatically destroyed
```

**Important:** Destroying key versions cannot be undone after the grace period

---

### 2.9 Asymmetric Keys

Create key pairs for signing/verification

```bash
# Create asymmetric signing key
gcloud kms keys create asymmetric-sign-key \
  --location=us-central1 \
  --keyring=my-keyring \
  --purpose=asymmetric-sign

# Get public key
gcloud kms keys versions get-public-key VERSION_ID \
  --location=us-central1 \
  --keyring=my-keyring \
  --key=asymmetric-sign-key
```

**Use Cases:**
```
- Verifying JWTs
- Signing certificates
- Code signing
- Digital signatures
```

---

### 2.10 KMS Audit & Monitoring

**Audit Logs:**
All KMS operations logged in Cloud Audit Logs

```bash
# Query KMS audit logs
gcloud logging read "protoPayload.serviceName=cloudkms.googleapis.com" \
  --limit=50

# View specific key usage
gcloud logging read "protoPayload.resourceName=~\".*my-key.*\"" \
  --limit=50
```

**Monitoring:**
- Monitor key rotation
- Alert on failed encrypt/decrypt
- Track key usage patterns

---

### 2.11 Pricing

```
API calls:
  $0.06 per 10,000 key operations (encrypt/decrypt/sign/verify)

Key storage:
  $0.06 per key version per month

HSM backing (if using):
  $1.00 per HSM-backed key version per month
```

**Example Cost:**
```
Key: my-key
Versions: 2 (current + backup)
Operations: 100K per month

Cost = (2 × $0.06) + ($0.06 × 100K/10K) = $0.12 + $0.60 = $0.72/month
```

---

### 2.12 KMS Best Practices

**1. Key Rotation**
```
✓ Enable automatic rotation (every 90 days)
✓ Monitor rotation timeline
✓ Keep rotation logs
```

**2. Key Isolation**
```
✓ Different keys for different purposes
✓ Don't share keys across services
✓ Service-specific keys in KMS
```

**3. Least Privilege**
```
✓ Grant cloudkms.cryptoKeyEncrypterDecrypter
✓ Not cloudkms.admin (too broad)
✓ Service accounts only for applications
```

**4. Monitoring**
```
✓ Enable audit logs
✓ Monitor key usage
✓ Alert on unusual patterns
```

**5. Key Backups**
```
✓ Maintain key version replicas
✓ Test recovery procedures
✓ Document backup procedures
```

**6. CMEK vs Google-Managed**
```
Google-Managed:
  ✓ Simpler (no KMS setup)
  ✗ No control over keys
  ✗ Google can access

CMEK (Customer-Managed):
  ✓ You control keys
  ✓ Meet compliance requirements
  ✗ More operational burden
  ✗ Higher cost
```

---

### 2.13 KMS Exam Tips

**Key Concepts:**
- KeyRing, Key, Key Version hierarchy
- Encryption vs Decryption
- Key Rotation (automatic & manual)
- IAM permissions for KMS
- CMEK for GCP services
- Asymmetric keys (signing)
- Audit logging
- Key destruction & recovery

**Common Scenarios:**
1. "Encrypt data in Cloud Storage" → CMEK on bucket
2. "Application needs to decrypt secrets" → cloudkms.cryptoKeyDecrypter role
3. "Rotate encryption keys" → Set rotation schedule
4. "Sign JWTs" → Asymmetric signing key
5. "Compliance requires key management" → Cloud KMS with CMEK

---

## So Sánh & Best Practices

### Cloud IAM vs Cloud KMS

| Aspect | Cloud IAM | Cloud KMS |
|--------|-----------|-----------|
| **Purpose** | Access control | Encryption key management |
| **Manages** | Who can access what | Encryption keys |
| **Scope** | All GCP resources | Data encryption |
| **Granularity** | Resource-level | Key-level |
| **Audit** | Audit logs | Key usage logs |

### Security Checklist

**Access Control (IAM):**
- [ ] Use service accounts for apps (not user accounts)
- [ ] Apply principle of least privilege
- [ ] Use predefined roles when possible
- [ ] Create custom roles only when necessary
- [ ] Grant roles at resource level (not project)
- [ ] Use groups for managing access at scale
- [ ] Enable Workload Identity in GKE
- [ ] Monitor service account usage
- [ ] Regularly review IAM bindings

**Encryption (KMS):**
- [ ] Use CMEK for sensitive data
- [ ] Enable automatic key rotation
- [ ] Store keys in KMS (not in code)
- [ ] Use different keys for different purposes
- [ ] Grant minimal KMS permissions
- [ ] Monitor key usage
- [ ] Test key recovery procedures
- [ ] Comply with encryption standards
- [ ] Document key management procedures

---

## Thực Hành & Scenarios

### Scenario 1: Secure Application with Service Accounts

**Requirements:**
- Application runs in Compute Engine
- Access Cloud SQL & Cloud Storage
- Minimal permissions
- Audit all access

**Setup:**

```bash
# 1. Create service account
gcloud iam service-accounts create myapp-sa \
  --display-name="My App Service Account"

# 2. Create roles with minimal permissions
# Cloud SQL: Read from specific database
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:myapp-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/cloudsql.client

# Cloud Storage: Read specific bucket
gcloud storage buckets add-iam-policy-binding gs://myapp-data \
  --member=serviceAccount:myapp-sa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# 3. Create instance with service account
gcloud compute instances create myapp-server \
  --machine-type=n1-standard-1 \
  --zone=us-central1-a \
  --service-account=myapp-sa@PROJECT_ID.iam.gserviceaccount.com

# 4. Verify permissions work
gcloud compute ssh myapp-server --zone=us-central1-a

# In the instance:
# Try to access Cloud Storage (should work)
gsutil ls gs://myapp-data

# Try to list all buckets (should fail - not allowed)
gsutil ls  # This should fail

# 5. Enable audit logging
gcloud logging sinks create app-audit \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/audit_logs \
  --log-filter='protoPayload.authenticationInfo.principalEmail="myapp-sa@PROJECT_ID.iam.gserviceaccount.com"'
```

---

### Scenario 2: CMEK for Cloud Storage

**Requirements:**
- Encrypt sensitive data with customer-managed keys
- Automatic key rotation
- Audit key usage

**Setup:**

```bash
# 1. Create KMS resources
gcloud kms keyrings create storage-keyring \
  --location=us-central1

gcloud kms keys create storage-key \
  --location=us-central1 \
  --keyring=storage-keyring \
  --purpose=encryption \
  --rotation-schedule-duration=7776000s  # 90 days

# 2. Grant Cloud Storage service account access to key
gcloud kms keys add-iam-policy-binding storage-key \
  --location=us-central1 \
  --keyring=storage-keyring \
  --member=serviceAccount:service-PROJECT_NUMBER@gcp-sa-cloud-storage.iam.gserviceaccount.com \
  --role=roles/cloudkms.cryptoKeyEncrypterDecrypter

# 3. Create bucket with CMEK
gsutil mb gs://encrypted-data-bucket

gcloud storage buckets update gs://encrypted-data-bucket \
  --default-encryption-key=projects/PROJECT_ID/locations/us-central1/keyRings/storage-keyring/cryptoKeys/storage-key

# 4. Upload encrypted file
gsutil cp sensitive-file.txt gs://encrypted-data-bucket/

# 5. Verify encryption
gsutil stat gs://encrypted-data-bucket/sensitive-file.txt
# Should show: Encryption algorithm: AES256
# Customer-managed encryption key: [key path]

# 6. Monitor key usage
gcloud logging read 'protoPayload.resourceName=~".*storage-key.*"' \
  --limit=50 \
  --format=json
```

---

### Scenario 3: Workload Identity in GKE

**Requirements:**
- GKE pods access Cloud Storage
- Use Workload Identity (no keys needed)
- Minimal permissions

**Setup:**

```bash
# 1. Enable Workload Identity on cluster
gcloud container clusters create my-cluster \
  --workload-pool=PROJECT_ID.svc.id.goog

# 2. Create Kubernetes service account
kubectl create serviceaccount gke-app-sa

# 3. Create GCP service account
gcloud iam service-accounts create gke-app-gsa

# 4. Bind KSA to GSA (Workload Identity)
gcloud iam service-accounts add-iam-policy-binding \
  gke-app-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:PROJECT_ID.svc.id.goog[default/gke-app-sa]"

# 5. Grant Cloud Storage access to GSA
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=serviceAccount:gke-app-gsa@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.objectViewer

# 6. Annotate KSA with GSA email
kubectl annotate serviceaccount gke-app-sa \
  iam.gke.io/gcp-service-account=gke-app-gsa@PROJECT_ID.iam.gserviceaccount.com

# 7. Deploy pod with KSA
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: gke-app
spec:
  serviceAccountName: gke-app-sa
  containers:
  - name: app
    image: google/cloud-sdk:slim
    command:
    - sh
    - -c
    - |
      # No credentials needed!
      gsutil ls gs://my-bucket
EOF

# 8. Verify it works
kubectl logs gke-app
# Should list contents of gs://my-bucket
```

---

### Scenario 4: Multi-User Project Access

**Requirements:**
- Multiple teams accessing same project
- Different permissions for each team
- Easy onboarding/offboarding
- Audit all access

**Setup:**

```bash
# 1. Create groups
# (Admin team handles this in Cloud Identity/Workspace)
# groups:
#   - admins@company.com
#   - developers@company.com
#   - data-analysts@company.com

# 2. Grant group-level access
# Admin team: Full project access
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=group:admins@company.com \
  --role=roles/editor

# Developers: Compute Engine access only
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=group:developers@company.com \
  --role=roles/compute.admin

# Data analysts: BigQuery access only
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member=group:data-analysts@company.com \
  --role=roles/bigquery.admin

# 3. Resource-level access
# Developers can only access dev bucket
gcloud storage buckets add-iam-policy-binding gs://dev-bucket \
  --member=group:developers@company.com \
  --role=roles/storage.objectAdmin

# Data analysts can only read data bucket
gcloud storage buckets add-iam-policy-binding gs://data-bucket \
  --member=group:data-analysts@company.com \
  --role=roles/storage.objectViewer

# 4. Enable audit logging
gcloud logging sinks create iam-audit \
  bigquery.googleapis.com/projects/PROJECT_ID/datasets/audit_logs \
  --log-filter='protoPayload.methodName=~"SetIamPolicy|AddBinding|RemoveBinding"'

# 5. Onboarding new developer
# Admin adds to group: developers@company.com
# Automatically gets compute.admin role
# (No individual bindings needed)

# 6. Offboarding
# Admin removes from group: developers@company.com
# Access revoked immediately
```

---

## Summary Checklist

### Cloud IAM
- [ ] Understand identity types (users, groups, service accounts)
- [ ] Know predefined roles (Viewer, Editor, Admin)
- [ ] Understand IAM bindings (member + role + resource)
- [ ] Can create & manage service accounts
- [ ] Know service account key management
- [ ] Understand Workload Identity
- [ ] Can grant roles at project & resource level
- [ ] Know IAM inheritance
- [ ] Understand audit logs
- [ ] Follow principle of least privilege

### Cloud KMS
- [ ] KeyRing, Key, Key Version hierarchy
- [ ] Creating keys & key rings
- [ ] Encrypting/decrypting with KMS
- [ ] Automatic key rotation
- [ ] Using KMS with GCP services (CMEK)
- [ ] Key destruction & recovery
- [ ] Asymmetric keys (signing)
- [ ] KMS IAM permissions
- [ ] Monitoring key usage
- [ ] Follow encryption best practices

---

**End of Security & Identity Guide**
