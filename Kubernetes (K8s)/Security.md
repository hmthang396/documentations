# Phase 8: Security (Realistic, Not Idealistic)

## 1. RBAC & ServiceAccount: The "Who" and the "What"

In Kubernetes, we don't give people "root" access. We use **Role-Based Access Control (RBAC)**.

### The Mental Model: The Office Building

- **Subject (Who):** The person or the app (e.g., `Developer-A` or `Payment-App`).
- **Role (What):** A set of keys. "This set of keys can open the supply closet and read the mail."
- **Binding (The Glue):** Attaching the set of keys to the person. "Developer-A is now holding the 'Mail-Viewer' keys."

### ServiceAccounts: Apps are People Too

A **ServiceAccount** is an identity for a Pod. If your app needs to talk to the K8s API (e.g., to list other pods), it must use its ServiceAccount.

> [!CAUTION]
> **The Default Leak:** By default, every namespace has a `default` ServiceAccount. In older K8s versions, this was automatically mounted with broad permissions. This allowed an attacker who hacked one web server to take over the whole cluster.

---

## 2. NetworkPolicy: The Internal Firewall

By default, **every Pod in Kubernetes can talk to every other Pod**. This is a hacker's dream.

### The Mental Model: The Hotel Hallway

Without NetworkPolicies, every guest can walk into any other guest's room. **NetworkPolicy** is the door lock.

- **Ingress:** Who is allowed to come **IN** to my pod? (e.g., "Only the Frontend can talk to the Backend").
- **Egress:** Where is my pod allowed to go **OUT**? (e.g., "The Database is NOT allowed to talk to the internet").

### Why do we need this?

If a hacker hacks your web server, the first thing they do is "Lateral Movement"—trying to find your Database. A NetworkPolicy stops them at the first hop.

---

## 3. Pod Security: Stopping the Breakout

Containers share the same **Kernel** as the server. If a container runs as `root`, it can potentially "escape" and take over the physical server.

### The Mental Model: The Prisoner with a Spoon

A container is a prisoner. Running as `root` is like giving the prisoner a heavy-duty spoon. Eventually, they might dig a hole through the wall.

**The Fix: Pod Security Standards**

- **Privileged=false:** Don't let the container see the server's hardware.
- **RunAsNonRoot:** Force the app to run as a normal user.
- **ReadOnlyRootFilesystem:** If a hacker gets in, they can't even download their malware because they can't save files!

---

## 4. Supply Chain Security: The Poisoned Barrel

You might have a secure cluster, but what if the **Image** you downloaded from the internet has a back door built-in?

### Real Attack Vectors:

1.  **Typosquatting:** You try to download `nginx`, but you accidentally type `nginxx`. You just downloaded a malicious miner.
2.  **Base Image Vulnerabilities:** Your app is secure, but the version of `node:14` you used has an old SSL bug.

### The Solution:

- **Scan your images:** Use tools like Trivy or Grype in your CI/CD.
- **Private Registry:** Only allow images from your company's own secure vault.
- **Signatures:** Use "Cosign" to prove that the image in Prod is the _exact same_ one that passed the tests.

---

## 5. Realistic Misconfigurations: How Clusters Actually Get Hacked

1.  **Dashboard exposed to the internet:** This is how Tesla's K8s cluster was hacked to mine Bitcoin.
2.  **Secrets in ConfigMaps:** As we learned in Phase 3, ConfigMaps are plaintext.
3.  **Over-privileged ServiceAccounts:** Giving a web-app "ClusterAdmin" just because it was "easier to debug."

---

### Key Takeaways

- **RBAC** = Least privilege (give only the keys needed).
- **NetworkPolicy** = Zero Trust (deny everything by default).
- **PodSecurity** = Hardened walls (no root, no breakout).
- **Supply Chain** = Trusted sources (verify before you run).

### Common Mistakes

- Using `hostNetwork: true`. This lets the pod see all the traffic on the server. **Extremely dangerous.**
- Running everything in the `default` namespace. It makes it impossible to write clean security rules.

### Production Tips

- Install a **Policy Engine** (like Kyverno or OPA Gatekeeper). It's like a police officer that automatically deletes any Pod that tries to run as `root`.

---

### 💡 Knowledge Check

Your "Reporting App" needs to read some data from the Kubernetes API.

1. Should you give your personal admin credentials to the app via a Secret?
2. If not, what component should you use to give the app its own identity?
3. How do you ensure the app can _only_ "Read" and not "Delete" resources?
