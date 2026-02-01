# Phase 3: Configuration & Secrets

## 1. ConfigMap vs. Secret: The Open Box and the Locked Box

In Kubernetes, we never bake configuration directly into the container image. Why? Because you want to use the **same image** for Development, Staging, and Production.

We use two primary tools to inject this configuration.

### The Mental Model: The Office Desk

- **ConfigMap:** ConfigMap là object dùng để tách cấu hình KHÔNG nhạy cảm ra khỏi container image, để ứng dụng có thể thay đổi cấu hình mà không cần build lại image
- **Secret:** là đối tượng dùng để lưu trữ dữ liệu nhạy cảm (sensitive data) mà không nên hard-code trong image, YAML, hay source code.

### Why do we need this?

If you change a configuration value (e.g., your DB address), you don't want to rebuild and push a new 500MB Docker image. You just update the **ConfigMap**, and Kubernetes manages the change.

### What is the difference between ConfigMap and Secret?

| Tiêu chí   | ConfigMap      | Secret                                     |
| ---------- | -------------- | ------------------------------------------ |
| Dữ liệu    | Không nhạy cảm | Nhạy cảm                                   |
| Encoding   | Plain text     | Base64                                     |
| RBAC       | Bình thường    | Thường bị giới hạn                         |
| Lưu ở etcd | Plain          | Base64 (⚠️ không phải encryption mặc định) |

### Storage

- ConfigMap: được lưu trong etcd của Kubernetes Control Plane

```
ConfigMap
  ↓
kube-apiserver
  ↓
etcd (distributed key-value store)
```

- Secret: Secret không ghi xuống disk, Lưu trong memory (tmpfs)

```
Secret
 ↓
etcd (Base64)
 ↓
kube-apiserver
 ↓
kubelet
 ↓
tmpfs trên Node
```

### Apply

#### ConfigMap:

1. Cách 1 – Hot Reload (đẹp nhất, khó nhất)

- Cấu hình ConfigMap mount dạng volume => ConfigMap update => File đổi trong container
- App reload config bằng cách Signal, File watcher, HTTP endpoint, Polling

2. Cách 2 - Rolling Restart (thực tế nhất)

- Trigger restart khi ConfigMap đổi bằng cách : Thêm hash vào Pod template

```
annotations:
  configmap-hash: {{ sha256sum of configmap }}
```

→ ConfigMap đổi
→ Deployment rollout

- Kubernetes đảm bảo không downtime

```
strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

3. Cách 3 – Sidecar Reloader (rất phổ biến)
   Pattern:

```
[ App ] ← SIGHUP ← [ Config Reloader Sidecar ]
```

- Sidecar:
  - Watch file
  - Gửi signal / HTTP call

- Tool thực tế:
  - stakater/reloader
  - configmap-reload (Prometheus)

4. Cách 4 – Blue/Green hoặc Canary Config

```
Config v1 → Pod set A
Config v2 → Pod set B
→ Route 5% traffic
→ OK → 100%
```

#### Secret:

1. Cách 1: Inject vào ENV (phổ biến)

```
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

2. Cách 2: Mount thành file (an toàn hơn)

```
volumes:
- name: secret-vol
  secret:
    secretName: db-secret

volumeMounts:
- name: secret-vol
  mountPath: /etc/secrets
  readOnly: true
```

---

## 2. Injection Methods: How the App Gets the Data

There are two main ways to give this data to your app.

### A. Environment Variables (The ID Badge)

- **How it works:** The value is "stamped" onto the process when it starts.
- **Best for:** Simple values, single-line strings, "True/False" flags.
- **Problem:** If the app is already running and you change the ConfigMap, the environment variable **will NOT update** until the app restarts.

### B. Volume Mounts (The Shared Folder)

- **How it works:** K8s creates a "virtual file" inside your container (e.g., `/etc/config/settings.json`).
- **Best for:** Large configuration files (JSON, YAML, XML).
- **Benefit:** If you update the ConfigMap, K8s will eventually **update the file** inside the running container without a restart (though your app must be smart enough to "re-read" the file).

---

## 3. The Secret Truth: Secrets are Not Very Secret (by default)

This is the most common production misconception.

> [!WARNING]
> By default, Kubernetes Secrets are just **Base64 encoded**, NOT encrypted. Base64 is like writing a password in "Pig Latin"—it's trivial to decode.

### Why not store secrets in Git?

If you put your `secret.yaml` in GitHub, every developer (and every hacker with access) has your production passwords. Even if you delete it later, it stays in the Git history forever.

### How Secrets leak in real systems:

1.  **Logging:** An app crashes and prints all its environment variables (including passwords) to the logs.
2.  **Process Listings:** In some environments, running `ps aux` can show the command-line arguments (which might include secrets).
3.  **The "Describe" Leak:** A developer runs `kubectl describe pod` and sees the plaintext values of ConfigMaps (though Secrets are masked here, they aren't in the YAML).

---

### Key Takeaways

- **ConfigMap** = Public stuff. **Secret** = Private stuff.
- **Never** store sensitive data in a ConfigMap.
- Use **Volume Mounts** if you want configuration to update without a restart.
- **Base64 encoding is NOT encryption.**

### Common Mistakes

- Naming a ConfigMap `env-config` and putting your DB password in it.
- Forgetting that Secrets have a size limit (usually 1MB). They aren't for large files.

### Production Tips

- Use an **External Secrets Manager** (like HashiCorp Vault, AWS Secrets Manager, or GCP Secret Manager) and sync them into K8s. This is the "Principal Engineer" way.
- Avoid using environment variables for secrets; prefer **Volume Mounts** because they are less likely to be accidentally printed in crash logs.

---

### 💡 Knowledge Check

If you have a `settings.yaml` file that is 500KB and contains your app's theme colors and API endpoints, should you inject it as an **Environment Variable** or a **Volume Mount**? Why?
