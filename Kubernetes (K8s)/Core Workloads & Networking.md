# Phase 2: Core Workloads & Networking

## 1. The Pod Lifecycle: From Birth to Burial

A Pod doesn't just "start." it go through a series of states. Understanding these is the #1 skill for debugging.

### The Mental Model: The Patient in a Hospital

1.  **Pending (Admission):** The Pod has been accepted by the system, but it's waiting for a bed (a Node). Maybe the hospital is full, or they are waiting for a special equipment (an Image download).
2.  **Running (Under Treatment):** The Pod is assigned to a Node and the containers have been created. At least one container is still running or is in the process of starting.
3.  **Succeeded (Discharged):** The Pod did its job (like a CronJob or a Batch process) and exited with zero errors. It is now "retired."
4.  **Failed (Code Blue):** All containers in the Pod have terminated, and at least one container terminated in failure (nonzero exit code).
5.  **Unknown (Lost Signal):** The state of the Pod cannot be obtained. This usually happens because the Worker Node decided to stop talking to the Master.

---

## 2. Workload Controllers: Who is in Charge?

In Phase 1, we said the "Controller Manager" is the repairman. But there are different _types_ of repairmen for different jobs.

### 1. Deployment (The Factory Manager) - **MOST COMMON**

**Deployment** (hay **Kubernetes Deployment**) là một **tài nguyên (resource)** quan trọng nhất trong Kubernetes để **quản lý việc triển khai, cập nhật và duy trì ứng dụng stateless** (ứng dụng không lưu trạng thái cục bộ, ví dụ: web server, API backend, microservices).

Nói đơn giản:  
Deployment giúp bạn khai báo "tôi muốn ứng dụng X chạy với Y replicas, image phiên bản Z" → Kubernetes sẽ tự động:

- Tạo đủ số Pod.
- Giữ cho ứng dụng luôn chạy đúng số lượng replicas.
- Cập nhật phiên bản mới (rolling update) mà không downtime.
- Rollback về phiên bản cũ nếu lỗi.
- Tự heal khi Pod chết.

Nó là cách phổ biến nhất để chạy ứng dụng production trong Kubernetes (thay vì dùng trực tiếp Pod hay ReplicaSet).

- **What it does:** Ensures a specific number of "stateless" pods (like web servers) are running.
- **Mental Model:** You tell the manager, "I want 5 identical blue widgets." If one breaks, he makes a new blue widget. If you want Green widgets, he replaces the blue ones one by one (**Rolling Update**).
- **Why use it:** For almost everything. Web APIs, microservices, etc.

### 2. ReplicaSet (The Factory Foreman)

**ReplicaSet** (hay **Kubernetes ReplicaSet**) là một **tài nguyên (resource)** cơ bản trong Kubernetes, chịu trách nhiệm **duy trì một tập hợp ổn định các Pod giống nhau** (replicas) đang chạy tại bất kỳ thời điểm nào.

Nói đơn giản:  
ReplicaSet giống như một **"người quản lý ca làm việc"** — bạn bảo "tôi muốn luôn có đúng 3 Pod chạy ứng dụng X" → nó sẽ:

- Tạo đủ Pod nếu thiếu.
- Tự động tạo Pod mới nếu Pod nào chết hoặc bị xóa.
- Xóa bớt Pod nếu bạn scale xuống (ví dụ: từ 5 xuống 3).

Nó đảm bảo **high availability** và **load balancing cơ bản** cho ứng dụng stateless.

- **What it does:** The low-level component that actually maintains the number of pods.
- **Note:** You almost never touch this directly. The **Deployment** manages the ReplicaSet for you.

### 3. StatefulSet (The Hotel Manager)

**StatefulSet** là một **tài nguyên (workload controller)** trong Kubernetes được thiết kế đặc biệt để quản lý các **ứng dụng có trạng thái (stateful applications)** — những ứng dụng cần duy trì **dữ liệu bền vững**, **định danh cố định** và **thứ tự rõ ràng** giữa các instance (ví dụ: database, message queue, distributed cache...).

Nói ngắn gọn:  
StatefulSet giống như **"Deployment cho ứng dụng có bộ nhớ"** — nó đảm bảo mỗi Pod có **tên cố định**, **storage riêng biệt không đổi**, và **tạo/xóa theo thứ tự** (ordered scaling/termination), thay vì ngẫu nhiên như Deployment.

- **What it does:** For apps that need a "name" and "memory" (State).
- **Mental Model:** In a Deployment, pods are like **Cattle** (nameless, replaceable). In a StatefulSet, pods are like **Pets** (they have names: `db-0`, `db-1`).
- **Why use it:** Databases (PostgreSQL, MySQL), Kafka, ZooKeeper. They need to keep their data even if they restart.

### 4. DaemonSet (The Housekeeping Staff)

**DaemonSet** là một **tài nguyên workload controller** trong Kubernetes, được thiết kế để đảm bảo **một Pod chạy trên mọi Node** (hoặc một tập hợp Node cụ thể) trong cụm, và duy trì Pod đó **luôn chạy** trên các Node đó.

Nói ngắn gọn:  
DaemonSet giống như một **"agent chạy khắp nơi"** — bạn muốn một công cụ giám sát, logging, networking, hoặc proxy chạy trên **tất cả các worker node** (hoặc tất cả node khớp điều kiện) → DaemonSet sẽ tự động tạo Pod tương ứng trên mỗi Node, và nếu Node mới được thêm vào cụm → nó tự tạo Pod mới; nếu Node bị xóa → Pod cũng bị xóa theo.

- **What it does:** Ensures that **one copy** of a Pod is running on **every single node**.
- **Mental Model:** Every room in the hotel needs a smoke detector.
- **Why use it:** Logging agents (Fluentd), monitoring agents (Prometheus Node Exporter), Network plugins.

---

### Bảng so sánh tổng quan

| Đặc điểm                   | ReplicaSet                                        | Deployment                                             | StatefulSet                                                                              | DaemonSet                                                                                                                       |
| -------------------------- | ------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **Mục đích chính**         | Đảm bảo **số lượng Pod replicas chính xác**       | Quản lý **stateless app** với rollout, rollback, scale | Quản lý **stateful app** (có trạng thái, identity cố định)                               | Đảm bảo **1 Pod chạy trên mọi Node** (hoặc subset Node)                                                                         |
| **Loại ứng dụng phù hợp**  | Stateless (ít dùng trực tiếp)                     | Stateless (web, API, microservices)                    | Stateful (DB: MySQL, PostgreSQL, MongoDB, Redis cluster, Kafka, Cassandra, ZooKeeper...) | Node-level agents (logging: Fluentd/Fluent Bit, monitoring: node-exporter, network: kube-proxy, Istio sidecar, security: Falco) |
| **Pod identity / Tên Pod** | Ngẫu nhiên (hash suffix, ví dụ: app-abc123)       | Ngẫu nhiên (qua ReplicaSet)                            | **Cố định, thứ tự** (app-0, app-1, app-2...)                                             | Thường ngẫu nhiên nhưng gắn với Node                                                                                            |
| **Thứ tự tạo/xóa Pod**     | Song song, không thứ tự                           | Song song                                              | **Thứ tự nghiêm ngặt** (tạo từ 0 → N, xóa từ N → 0)                                      | Không thứ tự (tạo trên Node sẵn sàng)                                                                                           |
| **Storage (PVC)**          | Không sticky (Pod chết → PVC có thể gắn Pod khác) | Không sticky                                           | **Sticky** (PVC gắn cố định với Pod identity, ví dụ: data-mysql-0)                       | Thường dùng hostPath, emptyDir hoặc local PV (không cần sticky lớn)                                                             |
| **DNS / Network identity** | Không ổn định                                     | Không ổn định                                          | **Ổn định** (qua Headless Service: app-0.my-svc...)                                      | Không cần ổn định (thường expose qua hostPort hoặc node IP)                                                                     |
| **Scaling**                | Horizontal (scale replicas)                       | Horizontal + elastic                                   | Horizontal nhưng **ordered**                                                             | Tự động scale theo số Node (không scale thủ công)                                                                               |
| **Rolling update**         | Không hỗ trợ                                      | **Có** (RollingUpdate mặc định, Recreate)              | **Có**, ordered (từ cũ → mới hoặc ngược)                                                 | **Có**, cập nhật từng Node (rolling)                                                                                            |
| **Rollback**               | Không                                             | **Có** (revision history)                              | Không (không dùng ReplicaSet, không rollback revision)                                   | Không (nhưng update image sẽ rolling)                                                                                           |
| **Self-healing**           | Có (tạo lại Pod chết)                             | Có                                                     | Có                                                                                       | Có (nếu Node mới join → tạo Pod)                                                                                                |
| **Sử dụng phổ biến**       | Ít (Kubernetes khuyến nghị dùng Deployment)       | **Rất phổ biến** cho stateless production              | Phổ biến cho database/cluster stateful                                                   | Phổ biến cho agent hệ thống                                                                                                     |
| **Controller quản lý**     | ReplicaSet Controller                             | Deployment Controller (tạo/quản lý ReplicaSet)         | StatefulSet Controller                                                                   | DaemonSet Controller                                                                                                            |
| **Ví dụ YAML kind**        | `kind: ReplicaSet`                                | `kind: Deployment`                                     | `kind: StatefulSet`                                                                      | `kind: DaemonSet`                                                                                                               |

---

### Key Takeaways

- **Pending** usually means "I can't find a place to live" or "I can't download the image."
- **Deployment** = Stateless apps (replaceable).
- **StatefulSet** = Stateful apps (unique identities).
- **DaemonSet** = Infrastructure tools (one per node).

### Common Mistakes

- Using a **Deployment** for a Database. If the pod moves, the DB might lose its disk or its identity, causing data corruption.
- Letting Pods stay in **Pending** forever. Usually, this is because you requested more CPU/RAM than your servers actually have.

### Production Tips

- Always look at the **Events** (`kubectl describe pod`) when a pod is stuck in Pending. K8s will tell you exactly why, e.g., "0/3 nodes are available: 3 Insufficient cpu."

---

### 💡 Knowledge Check

If you are deploying a **NGINX Web Server** that just serves static HTML, which controller should you use? What if you are deploying a **MongoDB Cluster** where each node needs to know who the "Primary" is?

---

## 3. Kubernetes Networking: How Pods Talk

Networking in K8s follows one golden rule: **Every Pod gets its own IP address.**

### The Mental Model: The Apartment Complex

- **A Container** is a person in an apartment.
- **A Pod** is the Apartment itself. It has its own mailbox (IP).
- **The World** is the street.

In K8s, any Pod can talk to any other Pod using just its IP, no matter which server they are on. K8s handles the "routing" behind the scenes.

### The Problem: Pods are Mortal

Pods die and are reborn with **new IPs**. If your Web App is talking to "Database at 10.0.0.5" and the Database restarts as 10.0.0.9, your Web App breaks.

---

## 4. Services: The Stable Front Door

**Service** trong Kubernetes (hay **Kubernetes Service**) là một **tài nguyên abstraction mạng** (network abstraction) quan trọng nhất, giúp **expose** (tiếp cận) một nhóm **Pod** (thường là các Pod chạy cùng ứng dụng) qua một **địa chỉ IP ổn định** và **DNS name** duy nhất, đồng thời cung cấp **load balancing** (cân bằng tải) giữa các Pod backend.

Nói ngắn gọn:  
Pod có IP thay đổi liên tục (khi restart, reschedule, scale...), nhưng **Service** cung cấp một **VIP (Virtual IP)** cố định + DNS name (ví dụ: `my-service.default.svc.cluster.local`) để các Pod khác hoặc bên ngoài truy cập ổn định. Không có Service → Pod khó giao tiếp với nhau một cách đáng tin cậy.

A **Service** is a permanent IP address (and DNS name) that sits in front of a group of Pods.

### The Mental Model: The Receptionist

You don't call the doctor directly. You call the **Main Office** (The Service). The receptionist knows which doctors are currently in the building and routes your call to one of them.

### Service Types (The 3 Flavors)

1.  **ClusterIP (Internal Only - Default):**
    - **What it is:** An IP reachable only from _inside_ the cluster.
    - - **Use case:** Your Database. You don't want the internet talking to your DB.
2.  **NodePort (The Quick Hack):**
    - **What it is:** Opens a specific port (e.g., 30005) on **every single server** (Node) in your cluster. If you visit `AnyNodeIP:30005`, you reach the service.
    - - **Use case:** Simple testing, or when you aren't on a cloud provider.
3.  **LoadBalancer (The Production Way):**
    - **What it is:** K8s asks your cloud provider (AWS/GCP/Azure) to create a "Real" Load Balancer with a public IP.
    - - **Use case:** Your public-facing website.
4.  **Headless Service**
    - **What it is:** A Service that does not have a ClusterIP. It is used to select a set of Pods and return their IPs directly.
    - - **Use case:** Stateful applications that need to know the IPs of the Pods they are talking to.

### Các loại Service phổ biến (types - cập nhật 2026 vẫn giữ nguyên 4 loại chính)

| Type             | Mô tả                                                               | Accessibility                    | Khi nào dùng?                                        | IP/DNS ví dụ                                 |
| ---------------- | ------------------------------------------------------------------- | -------------------------------- | ---------------------------------------------------- | -------------------------------------------- |
| **ClusterIP**    | Mặc định. Tạo VIP chỉ dùng **trong cụm** (internal)                 | Chỉ trong cluster                | Giao tiếp giữa microservices, backend API            | 10.96.x.x / my-svc.default.svc.cluster.local |
| **NodePort**     | Expose trên **mỗi Node** tại một port tĩnh (30000-32767)            | Từ ngoài qua <NodeIP>:<NodePort> | Dev/test, khi chưa có LoadBalancer                   | NodeIP:30080                                 |
| **LoadBalancer** | Tạo external load balancer (cloud provider: AWS ELB, GCP, Azure...) | Từ internet (public IP)          | Production expose web/API ra ngoài                   | External LB IP                               |
| **ExternalName** | Map Service đến **DNS name bên ngoài** (không proxy, chỉ CNAME)     | Redirect đến external DNS        | Integrate legacy/external services (ví dụ: DB cloud) | CNAME to mydb.example.com                    |

- **Headless Service** (không phải type riêng, mà là `clusterIP: None`): Không tạo VIP, trả về trực tiếp IP của tất cả Pod (dùng cho StatefulSet discovery, như database cluster).

### Note

# 1️⃣ Service KHÔNG phải là một process

Trước hết phải phá một hiểu nhầm rất phổ biến:

> ❌ Service **không** chạy như một container
> ❌ Không có “service pod”

👉 **Service chỉ là rule mạng**, được triển khai bởi **kube-proxy** trên **mỗi Node**

---

## 5. Ingress: The Traffic Cop

**Ingress** trong Kubernetes là một **tài nguyên API** (Ingress resource) dùng để **quản lý và expose** các dịch vụ HTTP/HTTPS từ **bên ngoài cụm** (internet hoặc client ngoài) vào các **Service** bên trong cluster một cách **thông minh**, dựa trên **rules** như domain (host), path, TLS termination, v.v.

Nói ngắn gọn:  
Ingress giúp bạn **expose nhiều ứng dụng** (nhiều Service) qua **một IP công cộng duy nhất** (thay vì tạo nhiều LoadBalancer tốn kém), với khả năng **route traffic** theo URL (ví dụ: example.com/api → service-api, example.com/web → service-web), hỗ trợ **HTTPS**, **load balancing**, và nhiều tính năng web nâng cao.

Ingress **không phải là load balancer thực tế** — nó chỉ là **cấu hình declarative**. Phần thực thi (proxy, route) do **Ingress Controller** đảm nhận.

### The Mental Model: The Airport Gate

- **Load Balancer:** The Airport itself (One entry point).
- **Ingress:** The flight board. It says: "If you are looking for `api.myapp.com`, go to Gate 1. If you want `blog.myapp.com`, go to Gate 2."

**Traffic Flow:**
`Internet` —> `External Load Balancer` —> `Ingress Controller (Nginx/Envoy)` —> `Service` —> `Pod`

---

### Key Takeaways

- **Pods** have IPs, but they change.
- **Services** provide a stable identity (DNS/IP).
- **ClusterIP** is for internal talk; **LoadBalancer** is for external.
- **Ingress** is the smart router that saves you money by sharing one IP for many services.

### Common Mistakes

- Trying to use Pod IPs in your configuration files. **Always use the Service Name.**
- Opening a **NodePort** for everything. This is a security risk.

### Production Tips

- Always use a **DNS name** for your services. K8s has an internal DNS server. If your service is named `my-db`, any pod can just talk to `http://my-db`.

### Note

Ingress gồm 2 phần (cực kỳ quan trọng)

1. Ingress Resource (YAML)

Chỉ là cấu hình

Không xử lý traffic

2. Ingress Controller

Là process thật (Pod)

Nhận traffic và xử lý

### Flow Production:

```
Internet
   ↓
LoadBalancer(prod) / NodePort(lab)
   ↓
Ingress Controller (NGINX / HAProxy / Traefik)
   ↓
ClusterIP Service
   ↓
Pod
```

---

### 💡 Knowledge Check (The Grand Finale of Phase 2)

You have a "Frontend" pod that needs to talk to a "Backend" pod.

1.  Does the Frontend need a Service?
2.  Does the Backend need a Service?
3.  Which Service Type should the Backend use?
