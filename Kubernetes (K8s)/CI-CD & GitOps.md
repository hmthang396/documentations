# Phase 9: CI/CD & GitOps

## 1. CI vs. CD in the Kubernetes World

In traditional systems, CI/CD was often one long script. In Kubernetes, we separate them strictly because they handle different parts of the "Source-to-Running" journey.

### Continuous Integration (CI): "The Factory"

- **Job:** Turn code into a Docker image.
- **Steps:** Lint -> Test -> Build Image -> Scan Image -> Push to Registry.
- **Result:** A versioned image (e.g., `my-app:v1.2.3`) sitting in a vault.

### Continuous Deployment (CD): "The Messenger"

- **Job:** Tell Kubernetes that the world has changed.
- **Steps:** Update the YAML to use `v1.2.3` and apply it to the cluster.
- **Analogy:** CI builds the car; CD drives it to the customer.

---

## 2. Helm vs. Kustomize: The Trade-off

How do you manage your YAML files for different environments (Dev, Staging, Prod)?

### Helm (The Templating Engine)

- **Mechanism:** Uses placeholders (e.g., `{{ .Values.replicaCount }}`). You provide a `values.yaml` file to fill them in.
- **Pros:** Packages everything into a "Chart." Great for sharing complex apps (like installing a whole Database with one command).
- **Cons:** "Template Hell." If your app is complex, the YAML becomes unreadable with if/else logic everywhere.

### Kustomize (The Patcher)

- **Mechanism:** Uses "Overlays." You have a `base` YAML and then a `pod-patch.yaml` for Prod that only changes the parts that are different.
- **Pros:** Pure YAML. No weird syntax. It's built into `kubectl` (`kubectl apply -k`).
- **Cons:** Harder to manage for extremely large, multi-component applications where everything changes between environments.

> [!TIP]
> **The Real-World Choice:** Use **Helm** for third-party tools (like Prometheus or NGINX) and **Kustomize** for your own internal apps where you want to keep the YAML simple and close to the code.

---

## 3. GitOps: The "Pull" Revolution (ArgoCD / Flux)

In the old days, our CI script would run `kubectl apply`. This is called **Push-based CD**.

### The Problem with Push:

1.  **Security:** Your CI tool (GitHub Actions/Jenkins) needs "Admin" keys to your cluster. If someone hacks your CI, they own your cloud.
2.  **Configuration Drift:** If a developer manually changes something using `kubectl edit`, the CI tool doesn't know. The "Source of Truth" (Git) is now different from the cluster.

### The Solution: Pull-based (GitOps)

You install a small agent (ArgoCD or Flux) **inside** your cluster.

- **How it works:** The agent watches your Git repo. If the Git repo changes, the agent "pulls" the change and applies it to itself.
- **Analogy:** Instead of me pushing food into your mouth (Push), you have a stomach that automatically eats whenever food appears on the table (Pull).

---

## 4. Rollback Strategies: The "Undo" Button

What happens if `v1.2.3` crashes in production?

### 1. The Git Revert (Standard GitOps)

You simply revert the commit in Git. ArgoCD sees the change and automatically "deploys" the previous version.

- **Pro:** Your Git history always matches what is in Production.

### 2. The Helm Rollback

You run `helm rollback <release>`.

- **Con:** This bypasses Git. Your cluster is now on `v1.2.2` but Git still says `v1.2.3`. **Avoid this in production.**

### 3. Progressive Delivery (The "Staff Engineer" way)

Using tools like **Argo Rollouts** or **Flagger**.

- If the new version has high error rates, the system **automatically** rolls back traffic to the old version without a human touching anything.

---

### Key Takeaways

- **CI** builds the image; **CD** updates the manifest.
- **Helm** is for packaging; **Kustomize** is for patching.
- **GitOps** moves security inside the cluster and prevents "Drift."

### Common Mistakes

- **Hardcoding tags:** Using `:latest` in your YAML. GitOps will never know when the image has changed! Always use specific tags (hashes or versions).
- **Storing Secrets in Git:** Even in GitOps, you still need a secret manager (like Sealed Secrets or External Secrets).

### Production Tips

- Treat your **Infrastructure as Code (IaC)**. No manual `kubectl` changes allowed in Production. If it's not in Git, it doesn't exist.

---

### 💡 Knowledge Check

You have a "Frontend" app. In Dev, you want 1 replica. In Prod, you want 10 replicas and a different API URL.

1. If you use **Kustomize**, how would you organize your folders?
2. If you use **ArgoCD**, and a developer manually deletes a Deployment using `kubectl delete`, what will ArgoCD do within 30 seconds?
