# Phase 10: Operating Kubernetes Long-Term

## 1. Upgrading Clusters: The "Moving Train" Problem

Kubernetes releases a new minor version every ~4 months. You cannot stay on an old version forever because you will stop receiving security patches.

### The Mental Model: Changing Tires on a Moving Car

You have to upgrade the **Control Plane** first, and then the **Worker Nodes**, all while your app is still handling traffic.

### The Strategy:

1.  **Upgrade Control Plane:** Usually handled by your cloud provider (EKS/GKE/AKS).
2.  **Upgrade Worker Nodes:** We use a **Rolling Update** for nodes.
    - **Cordon:** Mark the node as "Don't put new pods here."
    - **Drain:** Kick the existing pods off the node (they move to other nodes).
    - **Terminate/Replace:** Delete the old node and bring in a new one with the new version.
    - **Uncordon:** Open the new node for business.

---

## 2. Surviving Failures

### A. Node Failures (Common)

- **What happens:** A server's RAM fails or it lost power.
- **K8s Reaction:** After a timeout (usually 5 mins), K8s marks the node as `NotReady` and starts recreating its pods on other servers.
- **War Story:** A "Zombied" node. The node is _partially_ alive. K8s thinks it's okay, but the pods can't reach the network. **Lesson:** Always have a health check (Probe) that tests _outbound_ connectivity.

### B. Control Plane Failures (Rare but Scarier)

- **What happens:** `etcd` (the database) gets corrupted or the `API Server` crashes.
- **K8s Reaction:** Your existing pods keep running! **BUT**, you cannot change anything. You can't deploy new code, scale up, or delete pods.
- **War Story:** `etcd` disk became 100% full. The cluster "froze." No logs, no commands, just silence. **Lesson:** Always monitor your Control Plane disk space and latency.

---

## 3. Cost Optimization: FinOps in K8s

Kubernetes makes it very easy to spend $10,000 without realizing it.

### Strategies to Save Money:

1.  **Right-Sizing:** Most developers set "Requests" too high. Use the `VPA` (Vertical Pod Autoscaler) in "Recommendation Mode" to see how much you _actually_ use.
2.  **Spot Instances:** Use "cheap" servers that AWS/GCP can take back at any time. Great for Batch jobs or Development environments.
3.  **Horizontal Pod Autoscaling (HPA):** Don't run 10 pods at night if you only need 2.

---

## 4. The "No K8s" Decision (When to Walk Away)

Kubernetes is a massive, complex machine. Sometimes, it's the wrong tool.

### Don't use Kubernetes if:

- **You only have 1 or 2 small apps:** Use a PaaS like Heroku, Vercel, or AWS App Runner. K8s is overkill.
- **You don't have a team to manage it:** K8s requires "Day 2" maintenance. If you don't have an SRE or DevOps person, you will eventually have an outage you can't fix.
- **Your app is a monolith with massive local storage needs:** K8s is built for distributed, small pieces.

---

## 5. Hard-Earned Lessons from the Trenches

1.  **The "Default" Namespace is a Trap:** Separate your apps by Namespace for better security, quotas, and cleanup.
2.  **Labels are the Engine:** Everything in K8s (Scaling, Networking, Scheduling) uses Labels. If you mess up your labels, the cluster becomes a mess.
3.  **Human Error > System Error:** 90% of K8s outages are caused by a human running `kubectl delete` on the wrong thing or a bad YAML commit. **Use GitOps.**

---

### The Final Mentorship Conclusion

You now have the mental models and the technical foundation of a **Senior Kubernetes Engineer**.

We've moved from **Web Servers (Phase 1)** to **Control Planes (Phase 2)**, through **Secrets (Phase 3)** and **Storage (Phase 4)**, into **Deployments (Phase 5)** and **Scaling (Phase 6)**. We've learned to **Observe (Phase 7)**, **Secure (Phase 8)**, and **Automate (Phase 9)**.

Kubernetes is not a destination; it's a way of thinking about systems as "Desired State." Keep that logic, and you'll be able to master any platform.

---

### 💡 Final Knowledge Check

Imagine you are the Principal Engineer for a startup. Your company is growing, and your AWS bill just doubled.

1. What is the first thing you would check in your K8s cluster to find "wasted" money?
2. If you want to move your "Non-Critical Batch Jobs" to Spot Instances to save 70% in costs, how do you ensure they don't get scheduled on the expensive "On-Demand" nodes where your Database lives? (Hint: Think back to Phase 2/Fundamental logic of where pods go).

**It's been a pleasure being your mentor. What's next on your journey?**
