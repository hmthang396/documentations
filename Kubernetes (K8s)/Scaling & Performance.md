# Phase 6: Scaling & Performance

## 1. Requests vs. Limits: The Resource Contract

In Kubernetes, you must tell the system how much "fuel" your app needs. We do this using **Requests** and **Limits**.

### The Mental Model: The All-You-Can-Eat Buffet

- **Request (The Reservation):** "I am coming with 4 people. Please save a table for 4." Kubernetes uses this to decide which node has enough space for you. If you request 2 CPUs, K8s will only put you on a server that has at least 2 CPUs free.
- **Limit (The Bouncer):** "You can eat as much as you want, but if you try to take more than 10 plates, the bouncer will stop you." This is the maximum amount of resource the container is allowed to use.

#### Deep Dive: What is `1000m`?

In K8s, CPU is measured in **millicores**.

- `1000m` = 1 vCPU (1 core on your server).
- `500m` = 0.5 vCPU.
  K8s uses **Completely Fair Scheduler (CFS) Quotas** to enforce this. If you set a limit of `500m`, the Linux kernel gives your container 50ms of CPU time for every 100ms period.

---

## 2. The Two Ways to Die: Throttling vs. OOMKill

What happens if your app tries to go over its limit? It depends on the resource.

### A. CPU: Throttling (The Slow Death)

CPU is a "compressible" resource.

- **What happens:** If your limit is 1 CPU and you try to use 2, Kubernetes doesn't kill your app. Instead, it **slows you down**. It gives you "half the time" on the processor.
- **Result:** Your app's response time (latency) spikes. It feels like the app is lagging.

### B. Memory: OOMKill (The Instant Death)

Memory is "incompressible."

- **What happens:** If your limit is 1GB and you try to use 1.01GB, the system has no choice. It **kills the container immediately**.
- **Result:** You see the status `OOMKilled` (Out of Memory Killed). K8s will then restart the pod (if you have a Deployment).

#### Deep Dive: OOM Scores

When a Node runs out of memory, the Linux kernel's **OOM Killer** decides which process to kill. Kubernetes influences this by assigning an `oom_score_adj`:

1.  **Guaranteed (Request = Limit):** Lowest priority to be killed (-997).
2.  **Burstable (Request < Limit):** Middle ground.
3.  **BestEffort (No Requests/Limits):** First to be sacrificed (+1000).

---

## 3. The Scaling Trinity

### 1. Horizontal Pod Autoscaler (HPA)

- **What it does:** Adds more Pods when traffic is high.
- **Deep Dive: The Stabilization Window:** To prevent "flapping" (scaling up and down too quickly), HPA has a **stabilization window** (default 5 mins for downscaling). It looks at the peak usage over that window before deciding to kill a pod.
- **Requirement:** You MUST have a `metrics-server` installed in your cluster, and your pods MUST have CPU/Mem **Requests** defined.

### 2. Vertical Pod Autoscaler (VPA)

- **What it does:** Makes existing Pods "bigger" (gives them more CPU/Memory).
- **Deep Dive: Modes:**
  - `Initial`: Only sets the size when the pod is first created.
  - `Recommender`: Only tells you what you _should_ set, but does nothing.
  - `Auto`: Automatically kills and recreates the pod with the new size.
- **Constraint:** You cannot use VPA and HPA on the same metric (e.g., CPU).

### 3. Cluster Autoscaler

- **What it does:** Adds more **Nodes** (Servers) to the cluster.
- **Deep Dive: How it thinks:** It doesn't look at CPU usage. It looks at **Pending Pods**. If a Pod says "I need 2GB RAM" and every node is full, the Pod stays `Pending`. Cluster Autoscaler sees this and triggers the cloud provider to launch a new VM.

---

## 4. Noisy Neighbors: The Nightmare of Shared Servers

Since many Pods live on one Node, a "bad" Pod can ruin the day for everyone else.

### The Problem:

If Pod A has **no limits**, it can eat all the CPU on the server, leaving nothing for Pod B. Even if Pod B has a "Request," the physical CPU is busy.

### The Solution:

**Always set both requests and limits.**

#### Deep Dive: Priority and Preemption

What if a critical "Payment Service" needs to start, but the cluster is full of low-priority "Email Testers"?
You can use **PriorityClass**.

- K8s will **Preempt** (kill) lower-priority pods to make room for high-priority ones.
- High-priority pods "queue jump" in the Scheduler.

---

### Key Takeaways

- **Requests** are for scheduling (where to put the pod).
- **Limits** are for safety (stopping a runaway pod).
- **CPU Throttles**; **Memory Kills**.
- **HPA** scales the app; **Cluster Autoscaler** scales the hardware.

### Common Mistakes

- **Setting Limits too low:** Your app will constantly be OOMKilled or throttled, leading to a terrible user experience.
- **Setting no Requests:** K8s will "overpack" the server like a crowded subway car. When everyone tries to move at once, the whole server crashes.
- **Using VPA and HPA together on the same metric:** They will fight each other. One will try to make the pod bigger, while the other tries to add more pods.

### Production Tips

- **Observe before setting:** Run your app in staging for 24 hours without limits to see its "natural" resource usage, then set your limits based on that data.
- **The Goldilocks Zone:** Set your Request at your "average" usage and your Limit at your "peak" usage.

---

### 💡 Knowledge Check

If your application's CPU usage is constantly at its **Limit**, but its Memory usage is only at 20% of its **Request**, what will the user experience be like? Will the Pod be restarted?
