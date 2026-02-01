# Phase 4: Storage & State

## 1. Why State is Hard in Kubernetes (The Ephemerality Problem)

Kubernetes was originally designed for **Stateless** apps (like web servers). Stateless apps are like **Cattle**: if one dies, you just replace it with an identical one, and nobody cares.

**Stateful apps** (Databases, Cache, Queues) are like **Pets**. If a Database dies, it _must_ come back with the same data it had before, or your business is in trouble.

### The Problem

By default, when a Pod is deleted, its internal filesystem is **wiped clean**. If your MySQL database saves data to `/var/lib/mysql` inside the container, and that pod is moved to another server, that data is **gone forever**.

---

## 2. PV and PVC: The Plug and the Socket

To solve this, Kubernetes separates the **Storage** from the **Pod**.

### The Mental Model: The Wall Outlet

**PersistentVolume** (hay **PV** trong Kubernetes) là một **tài nguyên lưu trữ bền vững** (persistent storage resource) trong cụm Kubernetes, đại diện cho một **khối lưu trữ thực tế** (như đĩa cứng, NFS share, cloud disk EBS/GCE PD/Azure Disk, Ceph RBD...) đã được provision (cung cấp) sẵn.

Nói ngắn gọn:  
PV là **"két sắt lưu trữ"** cấp cụm (cluster-level), tồn tại độc lập với Pod. Dữ liệu trong PV **không mất** khi Pod chết, restart, reschedule, hoặc thậm chí xóa Deployment/StatefulSet (tùy reclaim policy). Nó giống như một "ổ cứng gắn ngoài" mà Pod có thể "mượn" để lưu dữ liệu lâu dài.

**PersistentVolumeClaim** (hay **PVC** trong Kubernetes) là một **tài nguyên yêu cầu lưu trữ** (storage request resource) mà developer hoặc ứng dụng tạo ra để "yêu cầu" một phần lưu trữ bền vững từ cụm. PVC giống như một **"phiếu mượn két sắt"** — bạn chỉ cần khai báo nhu cầu (dung lượng, access mode, storage class...), Kubernetes sẽ tự động tìm hoặc tạo **PersistentVolume (PV)** phù hợp để bind (gắn kết) với PVC đó.

Nói ngắn gọn:

- **PV** là "két sắt thực tế" (do admin provision hoặc dynamic tạo).
- **PVC** là "giấy yêu cầu mượn két" (do developer tạo).
- Khi PVC bind thành công với PV → Pod có thể mount PVC để dùng lưu trữ bền vững, dữ liệu không mất khi Pod chết/reschedule.

- **PersistentVolume (PV) - The Power Grid (Wall Outlet):** This is the actual physical disk (an AWS EBS volume, a GCP Disk, or a folder on a server). It's provided by the Administrator.
- **PersistentVolumeClaim (PVC) - The Power Cord (Plug):** This is the request from the Developer. "I need 10GB of storage with Read/Write access."

### Why do we need this?

It allows the Developer to say "I need a disk" without knowing _how_ that disk is made. They don't need to know if it's an SSD on Amazon or a hard drive in a local data center. They just "plug in" their claim.

---

## 3. StorageClass: The Vending Machine

In a large company, you don't want to wait for an administrator to manually create a PV every time a developer needs 1GB.

### The Mental Model: The Vending Machine

A **StorageClass** is a template. When a developer creates a PVC, the "StorageClass" automatically talks to the cloud provider, creates a disk, and wraps it in a PV. This is called **Dynamic Provisioning**.

---

## 4. StatefulSet: The "Pet" Controller

We learned about **Deployments** (for cattle) and **StatefulSets** (for pets).

### How StatefulSet handles storage:

When you use a StatefulSet, each pod gets its **own unique PVC**.

- `mysql-0` gets `data-mysql-0`
- `mysql-1` gets `data-mysql-1`

If `mysql-0` dies and is reborn on a different server, Kubernetes is smart enough to find the `data-mysql-0` disk and re-attach it to the new pod. **The identity and the data follow the pod.**

---

### What breaks in production? (Data Loss Scenarios)

1.  **The "Deletion" Trap:** By default, if you delete a PVC, the underlying PV (and all your data) might be deleted too (depending on the **Reclaim Policy**).
2.  **Multi-AZ Failures:** If your disk is in `US-East-1a` (Amazon Zone), but Kubernetes tries to start your Pod in `US-East-1b`, the Pod will stay in **Pending** forever. Disks usually cannot "travel" between zones.
3.  **ReadWriteOnce (RWO) Confusion:** Most cloud disks can only be attached to **one server at a time**. If you try to have 3 Pods on 3 different servers writing to the same disk, it will fail.

---

### Key Takeaways

- **PV** is the physical resource; **PVC** is the request for that resource.
- **StorageClass** automates the creation of disks.
- **StatefulSets** ensure that the same Pod always gets the same Disk.

### Common Mistakes

- Using `hostPath` (storing data on the server's local disk) in production. If the server dies, your data dies. Only use this for local testing.
- Thinking that "StatefulSet" magically backups your data. **It doesn't.** It only ensures the disk is re-attached. You still need a backup strategy!

### Production Tips

- Always set your **Reclaim Policy** to `Retain` for production databases. This way, if you accidentally delete the PVC, the physical disk is kept safe.
- Monitor your disk usage! A "Disk Full" error is the #1 cause of database corruption in Kubernetes.

---

### 💡 Knowledge Check

If you have a cluster with 3 nodes, and you are using a standard AWS EBS volume (which is `ReadWriteOnce`), can you scale your WordPress "Uploads" folder to 3 pods across all 3 nodes? Why or why not?
