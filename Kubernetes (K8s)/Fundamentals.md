# Phase 1: Fundamentals (Mental Model First)

## 1. What Problem Does Kubernetes Actually Solve?

To understand Kubernetes (K8s), we first have to look at the evolution of how we deploy software.

### The Mental Model: The Infrastructure Tetris

Imagine you are a logistics manager for a shipping company.

1.  **The VM Era (The Big Trucks):**
    - In the old days, if you wanted to ship a package (your app), you had to hire a whole dedicated truck (a Virtual Machine).
    - **Problem:** Even if your package was small, you paid for the whole truck. It was heavy, slow to start, and if the truck broke down, your package was stuck until you manually moved it to another truck.

2.  **The Container Era (The Standard Boxes):**
    - Docker came along and said: "Let's put everything in standard-sized boxes (Containers)." These boxes are light and can fit in any truck.
    - **Improvement:** You can now pack many boxes into one truck. It's much more efficient.
    - **New Problem:** Who decides which box goes into which truck? If you have 1,000 boxes and 50 trucks, doing this by hand is a nightmare. This is **Bin Packing**.

3.  **The Kubernetes Era (The Automated Warehouse):**
    - Kubernetes is the **automated crane and logistics system** for your warehouse.
    - You don't tell the system: "Put Box A into Truck 1."
    - You tell the system: **"I want 3 copies of Box A to be somewhere in the warehouse at all times. I don't care where."**

    - **Definition:** Kubernetes là một hệ thống điều phối (orchestrator) để chạy và quản lý container ở quy mô lớn.

### Why do we need this? (The Value Proposition)

- **High Availability:** If a worker node (truck) crashes, K8s notices immediately and moves the containers to a healthy node.
- **Scalability:** If traffic spikes, you tell K8s "Give me 10 more boxes," and it finds space for them.
- **Resource Efficiency:** K8s plays "Infrastructure Tetris." It looks at how much CPU/RAM each box needs and packs them tightly to save you money on cloud bills.

### What happens if we don’t use it?

If you try to run complex systems with just Docker:

- **Manual Resuscitation:** When an app crashes at 3 AM, _you_ are the one who has to SSH in and restart the container.
- **Zombies:** You'll have containers running on servers that nobody remembers, eating up resources.
- **The Spreadsheet of Doom:** You'll end up keeping a manual list of which app is on which IP address, which will eventually go out of date, leading to a massive outage.

### What breaks in production?

In the early stages, people often fail to give Kubernetes enough "information" to do its job.
If you don't define **Resource Limits** (telling K8s how big the box is), one greedy box can grow until it takes over the whole truck, causing all other boxes in that truck to "suffocate" and crash. This is the **"Noisy Neighbor"** problem.

---

### Key Takeaways

- **Kubernetes is NOT a container engine.** It is a container **orchestrator**. It manages the lifecycle of containers.
- The primary goal is to **automate the operational tasks** that humans used to do manually.
- It shifts the focus from "Machine Management" to "Application Management."

### Common Mistakes

- Thinking Kubernetes "makes things simpler." It doesn't. It makes **complexity manageable**.
- Using K8s for a single static website. That's like hiring a logistics fleet to deliver a single letter. Use a simpler tool (Vercel, Cloud Run) instead.

### Production Tips

- Always think in terms of **Desired State**, not commands. Don't think "restart pod," think "I want this many pods running."

---

### 💡 Knowledge Check

If you have a cluster of 5 servers and you need to ensure your "Payment Service" is always running 3 instances for redundancy, what is the fundamental difference between doing this manually (e.g., with SSH + Docker) versus letting Kubernetes handle it?

---

## 2. Control Plane vs. Worker Node: The Brain and the Muscle

In Kubernetes, we divide the world into two main roles. It’s a classic **Manager-Worker** relationship.

### The Mental Model: The Restaurant

- **The Control Plane (The Front of House / Manager):**
  - This is where the manager sits. They take orders, keep the menu (config), look at the floor map, and decide which waiter is available to handle a specific table.
  - The manager doesn't cook the food; they **coordinate**.

  - **Definition:** Control Plane là “bộ não” của Kubernetes cluster.
    Nó chịu trách nhiệm:
    - nhận lệnh từ bạn (kubectl, CI/CD)
    - lưu trạng thái cluster
    - quyết định Pod chạy ở đâu
    - đảm bảo cluster luôn tiến về “desired state”
      👉 Worker Nodes chạy workload.
      👉 Control Plane điều phối và quản lý mọi thứ.

- **The Worker Nodes (The Kitchen / Cooks):**
  - These are the actual servers (VMs or physical machines) where your containers run.
  - They are the muscle. They do the heavy lifting: running the containers, handling the networking of the app, and reporting back to the manager: "I'm busy" or "I have space for more."

  - Node là một máy (VM hoặc physical server) trong Kubernetes cluster.
  - Worker Node là Node có nhiệm vụ chạy workload (Pods/Containers).
  - Hãy tưởng tượng cluster như một nhà máy:
    - Control Plane = quản lý + điều phối
    - Worker Nodes = công nhân + máy móc sản xuất
    - Pods = các công việc cụ thể

### ASCII Architecture

```text
       +---------------------------------------+
       |            CONTROL PLANE              |
       |  (The Brain - Decisions Happen Here)   |
       |                                       |
       |   +-----------+       +-----------+   |
       |   | API Server| <---> |    etcd   |   |
       |   +-----------+       +-----------+   |
       |         ^                   |         |
       |         |         +-----------------+ |
       |         v         | Scheduler       | |
       |   +-----------+   +-----------------+ |
       |   | Controller|                       |
       |   | Manager   |                       |
       |   +-----------+                       |
       +---------------------------------------+
                        |
            (Commands / Manifests via API)
                        |
       +---------------------------------------+
       |             WORKER NODES              |
       |         (The Muscle - Apps Run Here)  |
       |                                       |
       |  +----------+         +----------+    |
       |  | Node 1   |         | Node 2   |    |
       |  | [Pod A]  |         | [Pod B]  |    |
       |  | [Pod C]  |         |          |    |
       |  +----------+         +----------+    |
       +---------------------------------------+
```

### Why do we need this separation?

1.  **Fault Tolerance:** If a Worker Node dies, the Control Plane is still alive to notice it and reschedule the work.
2.  **Scalability:** You can add 1,000 more Worker Nodes without changing how you interact with the Control Plane.
3.  **Security:** You can restrict access so developers only talk to the API Server (Control Plane) and never have SSH access to the Worker Nodes themselves.

### What happens if we don’t use it? (The "God Node" Problem)

In old-school setups, we often ran the app and the configuration tools on the same machine.

- If that machine's kernel panicked, you lost both your app **and** your ability to fix it.
- Separating them means the "Governor" is isolated from the "Factory Floor."

### What breaks in production?

The biggest production nightmare is **"Brain Surgery" issues on the Control Plane**:

1.  **etcd instability:** If the database of the Control Plane (etcd) is slow or has disk latency, the entire cluster becomes "frozen." You can't start new pods or delete old ones. The cluster becomes a "read-only" graveyard.
2.  **API Server Overload:** If you have too many automated scripts hitting the API at once, the Control Plane can't respond to the Worker Nodes, leading to a "Split Brain" where nodes think they are alone and start doing weird things.

---

### Key Takeaways

- **Control Plane = Brain.** It manages the state and makes decisions.
- **Worker Node = Muscle.** It executes the work (containers).
- Production clusters usually have **multiple Control Plane nodes** (High Availability) so that if one manager goes on break, the others keep the restaurant running.

### Common Mistakes

- Running heavy workloads on the Control Plane nodes. Never do this. Keep them dedicated to management.
- Thinking that if the Control Plane goes down, your apps stop. **False.** Your apps (on Worker Nodes) keep running, but you lose the ability to change anything (no scaling, no updates).

### Production Tips

- Treat your Worker Nodes as **"Cattle, not Pets."** You should be able to kill a Worker Node at any time and trust the Control Plane to fix the situation.
- Always monitor **etcd latency**. It is the heartbeat of your cluster.

---

### 💡 Knowledge Check

If the Control Plane's API Server becomes unreachable for 1 hour, what happens to the users currently using your application? Can you deploy a new version of the app during this hour?

---

## 3. Pod, Container, Image: Why the "Pod" exists?

You already know what a **Container** is (your app in a box) and what an **Image** is (the blueprint for that box). But why did Kubernetes invent the **Pod**?

### The Mental Model: The Space Capsule

Imagine a mission to Mars.

- The **Image** is the blueprint for the capsule.
- The **Container** is the capsule actually sitting on the launchpad.
- The **Pod** is the **Life Support System** wrapped around that container.

✅ **Pod là đơn vị nhỏ nhất mà Kubernetes deploy và quản lý.**

Pod là một “wrapper” chạy **1 hoặc nhiều containers** cùng nhau, chia sẻ:

- Network (IP + port space)
- Storage volumes
- Lifecycle (start/stop together)

👉 Kubernetes không deploy container trực tiếp.
Nó deploy Pod.

In a pod, you might have:

1.  **The Main App:** The astronaut (your code).
2.  **The Sidecar:** A small robot that handles oxygen (logging/proxy/monitoring).

### Why do we need the Pod?

Containers are designed to run **exactly one process**. But in the real world, apps need "utility" processes to run right next to them.
A Pod allows two containers to:

1.  **Share the same IP:** They can talk to each other via `localhost`.
2.  **Share the same Filesystem:** They can both read/write to the same folder.
3.  **Die Together:** If the Pod is moved to another server, both containers move together.

### What happens if we don’t use it?

If K8s only managed individual containers, you'd have to manually coordinate that "App A" and its "Logger B" always end up on the same physical server. If they ended up on different servers, your logger wouldn't be able to see the app's files. It would be an orchestration nightmare.

### What breaks in production?

The **"Multiple Astronauts"** mistake.
Beginners often try to put two _completely different_ apps (like a Web App and a Database) in the same Pod because it "feels easier."
**Don't do this.** If the web app needs to scale to 10 copies but the DB only needs 1, you can't split them. Only put things in the same Pod if they **must** share a lifecycle.

---

## 4. The Magic Under the Hood: Internal Components

Let's look at the "Org Chart" of the Control Plane.

### 1. kube-apiserver (The Gatekeeper)

- **What it is:** The only way to talk to the cluster. Every command (kubectl, UI, even other K8s components) goes through here.
- **Production Tip:** If this is slow, nothing happens. It's the front door.

**kube-apiserver** là thành phần **quan trọng nhất** và là **trái tim** của **Kubernetes control plane** (mặt phẳng điều khiển).

Nói đơn giản:

kube-apiserver chính là **"cổng vào duy nhất"** chính thức để tương tác với cụm Kubernetes.

Nó cung cấp **Kubernetes API** (dưới dạng HTTP REST API) để mọi người và mọi thành phần khác giao tiếp với cụm.

### Vai trò chính của kube-apiserver

- Là **frontend** của toàn bộ control plane
- **Tiếp nhận** tất cả các yêu cầu (từ người dùng lẫn từ các thành phần hệ thống)
- **Xác thực** (authentication) → kiểm tra xem bạn có quyền không
- **Phân quyền** (authorization) → kiểm tra bạn được làm gì
- **Kiểm tra hợp lệ** (validation) của yêu cầu
- **Lưu trữ** trạng thái mong muốn vào **etcd** (sau khi đã validate)
- **Trả về** trạng thái hiện tại khi được hỏi (get, list, watch...)

### Ai giao tiếp với kube-apiserver?

| Ai gọi                    | Ví dụ lệnh / hành động                           | Mục đích                     |
| ------------------------- | ------------------------------------------------ | ---------------------------- |
| Người dùng                | `kubectl get pods`, `kubectl apply -f ...`       | Quản lý tài nguyên           |
| kube-controller-manager   | Theo dõi và điều chỉnh Deployment, ReplicaSet... | Duy trì trạng thái mong muốn |
| kube-scheduler            | Xem pod nào chưa được gán node → chọn node       | Quyết định đặt pod ở đâu     |
| kubelet (trên worker)     | Báo cáo trạng thái pod, nhận lệnh chạy pod mới   | Quản lý container thực tế    |
| kube-proxy                | Theo dõi Service/Endpoint thay đổi               | Cập nhật iptables / IPVS     |
| CI/CD pipeline            | Helm install, ArgoCD sync, GitOps...             | Triển khai ứng dụng tự động  |
| Horizontal Pod Autoscaler | Theo dõi metrics → tạo/xóa pod                   | Tự động scale                |

Tóm lại: **Hầu như không có hành động nào trong Kubernetes có thể xảy ra mà không đi qua kube-apiserver**.

### 2. etcd (The Source of Truth)

- etcd là một dự án mã nguồn mở (thuộc CNCF, đã graduated).
- Là **distributed key-value store** được thiết kế để:
  - **Nhất quán mạnh** (strong consistency) nhờ thuật toán **Raft**.
  - **Tính sẵn sàng cao** (highly available) khi chạy theo cụm (thường 3 hoặc 5 node).
  - **Độ tin cậy cao** cho dữ liệu quan trọng trong hệ thống phân tán.

Kubernetes chọn etcd làm **backing store** chính thức vì nó đáp ứng hoàn hảo nhu cầu của control plane.

### 3. kube-scheduler (The Matchmaker)

**kube-scheduler** là thành phần **bộ lập lịch** (scheduler) mặc định trong **Kubernetes control plane**, chịu trách nhiệm **quyết định Pod sẽ chạy trên Node nào** trong cụm.

Nói ngắn gọn:  
Khi bạn tạo một Pod (hoặc Deployment/StatefulSet/... tạo ra Pod), Pod ban đầu ở trạng thái **Pending** (chưa được gán Node). kube-scheduler sẽ "nhìn" toàn bộ cụm, tìm Node phù hợp nhất dựa trên tài nguyên, ràng buộc, ưu tiên... rồi **gán** Pod vào Node đó bằng cách cập nhật trường `nodeName` trong spec của Pod.

Sau khi được gán, **kubelet** trên Node đó mới nhận Pod và khởi chạy container thực tế.

### Vai trò chính của kube-scheduler

| Vai trò                     | Giải thích chi tiết                                                                                                        |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Watch Pod chưa gán Node** | Theo dõi qua kube-apiserver các Pod mới tạo hoặc Pending (không có nodeName)                                               |
| **Lọc (Filtering)**         | Loại bỏ các Node không đáp ứng yêu cầu (thiếu CPU/RAM, không match nodeSelector, taints không tolerate, affinity rules...) |
| **Chấm điểm (Scoring)**     | Đánh giá các Node còn lại theo điểm số (ví dụ: Node có nhiều tài nguyên trống hơn → điểm cao hơn)                          |
| **Chọn Node tốt nhất**      | Bind Pod vào Node có điểm cao nhất (hoặc random nếu bằng nhau)                                                             |
| **Cập nhật trạng thái**     | Ghi `nodeName` vào Pod object qua apiserver → etcd lưu trữ → kubelet nhận và chạy                                          |

→ kube-scheduler **không khởi chạy container**, chỉ quyết định "đặt ở đâu". Nó giống như một "nhân viên phân công việc" thông minh.

### 4. kube-controller-manager (The Repairman)

**kube-controller-manager** là thành phần **"người giám sát và tự động sửa chữa"** (reconciler) trong **Kubernetes control plane**, chịu trách nhiệm chạy hàng loạt **controller** (các vòng lặp điều khiển) để đảm bảo **trạng thái thực tế (actual state)** của cụm luôn khớp với **trạng thái mong muốn (desired state)** mà bạn khai báo.

- Là một **daemon** (quá trình chạy liên tục) nhúng nhiều **core controllers** mặc định của Kubernetes vào một binary duy nhất.
- Mỗi controller là một **vòng lặp điều khiển** (control loop) kiểu:  
  **Watch → Compare → Reconcile** (xem trạng thái → so sánh với desired → hành động để sửa nếu lệch).
- Tất cả controller đều giao tiếp **qua kube-apiserver** (watch/list/update/delete các object).

| Controller                          | Vai trò chính                                                      | Ví dụ thực tế                                        |
| ----------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------- |
| **ReplicationController**           | Duy trì số lượng pod chính xác theo replicas (cũ, ít dùng)         | Giữ đúng 3 pod cho RC cũ                             |
| **ReplicaSet**                      | Tương tự, nhưng dùng cho Deployment                                | Deployment scale → ReplicaSet tạo/xóa pod            |
| **Deployment**                      | Quản lý rollout, rollback, scaling Deployment                      | Rolling update, pause/resume, history                |
| **StatefulSet**                     | Quản lý pod có trạng thái (ordered, unique identity)               | Tạo pod theo thứ tự, giữ PVC gắn liền                |
| **DaemonSet**                       | Đảm bảo 1 pod chạy trên mọi (hoặc một số) node                     | node-exporter, logging agent chạy trên tất cả node   |
| **Job / CronJob**                   | Quản lý task chạy một lần hoặc định kỳ                             | Batch job hoàn thành → xóa pod                       |
| **Node Controller**                 | Theo dõi node status, đánh dấu NotReady, xóa pod khi node chết lâu | Node down 5 phút → evict pod để scheduler reschedule |
| **Endpoints Controller**            | Tạo và cập nhật Endpoints cho Service                              | Pod ready → thêm IP:port vào Endpoints               |
| **Service Account Controller**      | Tạo ServiceAccount mặc định cho namespace                          | Tự động tạo default SA                               |
| **Namespace Controller**            | Xóa tài nguyên khi namespace bị xóa                                | Xóa tất cả pod, service... trong namespace           |
| **Garbage Collector**               | Xóa object con khi owner bị xóa (ownerReferences)                  | Xóa pod khi ReplicaSet bị xóa                        |
| **TTL Controller**                  | Xóa job sau thời gian sống (TTL)                                   | Job hết hạn → tự xóa                                 |
| **Horizontal Pod Autoscaler (HPA)** | Theo dõi metrics → scale replicas lên/xuống                        | CPU > 80% → tăng replicas                            |

---

| Thành phần                  | Vai trò chính                                     | Ai truy cập chính?         | Nếu chết thì sao?                       |
| --------------------------- | ------------------------------------------------- | -------------------------- | --------------------------------------- |
| **kube-apiserver**          | Cổng vào, validate, auth, serve API               | kubectl, tất cả components | Không tạo/xem được object mới           |
| **etcd**                    | Lưu trữ trạng thái vĩnh viễn                      | Chỉ apiserver              | Cụm tê liệt, mất toàn bộ trạng thái     |
| **kube-scheduler**          | Quyết định Pod chạy trên Node nào                 | Tự động (watch apiserver)  | Pod mới vẫn Pending mãi, không tự scale |
| **kube-controller-manager** | Duy trì trạng thái (reconcile Deployment, HPA...) | Tự động                    | Deployment không scale/recreate Pod     |

---

### Key Takeaways

- **Pod** = The smallest unit K8s manages. It's a wrapper for one or more containers.
- **API Server** is the gateway; **etcd** is the memory.
- **Scheduler** decides _where_; **Controller** ensures _how many_.

### Production Tips

- Always use **Sidecars** for cross-cutting concerns (logging, security) rather than baking them into your main app image. This keeps your app "clean."

---

### 💡 Knowledge Check

If you want to run a "Log Relayer" that sends your app's logs to a central server, should you put it in the same Pod as your App, or a separate Pod? Why?

---

## 5. Declarative vs. Imperative: Why Intent Matters

This is the biggest "click" moment for new Kubernetes users.

- **Imperative (How):** "Go to the kitchen, pick up a pan, crack an egg, and fry it."
- **Declarative (What):** "I want a fried egg on my plate."

In traditional scripting (Imperative), you tell the server:

1.  `docker run my-app`
2.  `docker network connect my-net`
3.  `docker expose 80`

**In Kubernetes (Declarative), you write a Manifest (YAML):**
"I want 3 copies of this image, reachable on port 80."

### Why do we need this?

If you use Imperative commands and a server crashes, you have to run those commands again. If you use Declarative manifests, **Kubernetes knows your intent**. If a server crashes, K8s looks at your "desired state" (the YAML) and automatically restarts everything to match it.

### What happens if we don’t use it?

Without it, you have to write complex "Health Check" scripts that constantly ping your servers and run `docker restart` if they fail. You end up building a "Poor Man's Kubernetes" that is buggy and hard to maintain.

---

## 6. The Reconciliation Loop: The Heartbeat of K8s

How does K8s actually "decide" what to do? It's a simple, never-ending loop:

1.  **Observe:** Look at the cluster (Current State).
2.  **Diff:** Compare it to the YAML manifests (Desired State).
3.  **Act:** If they don't match, make changes to make them match.

### The Mental Model: The Thermostat

- **Desired State:** You set the thermometer to 72°F.
- **Observation:** The room is 68°F.
- **Action:** The thermostat turns on the heater.
- **Observation:** The room is 72°F.
- **Action:** The thermostat does nothing (matches desired state).

### What breaks in production?

**"Fight for Reality."**
Sometimes, two different controllers might have conflicting "Desired States."

- Controller A says: "I want 3 pods."
- Controller B says: "I want 0 pods."
  The cluster will enter a **Hot Loop** where it starts and kills pods forever, destroying your performance. This usually happens when people use two different automation tools (like Helm and a custom script) to manage the same resource.

---

### Key Takeaways

- **Declarative** = You tell K8s what you want, not how to do it.
- **Reconciliation Loop** = The engine that makes reality match your desires.
- K8s is essentially a bunch of "Thermostats" working together.

### Common Mistakes

- Trying to "fix" pods manually (e.g., SSHing in to change a file). **The Reconciliation Loop will destroy your changes** within minutes because they don't match the Desired State in the YAML.

---

### 💡 Knowledge Check

If you use `kubectl edit` to change the version of your app directly in the cluster, but your GitHub repo still has the old version in its YAML file, what might happen the next time your CI/CD pipeline runs?
