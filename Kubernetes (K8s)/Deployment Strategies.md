# Phase 5: Deployment Strategies

## 1. RollingUpdate vs. Recreate: The Basics

How do you change the version of your app without your users noticing? In Kubernetes, there are two built-in strategies in the **Deployment** object.

### A. Recreate (The "Turn it off and on again")

- **What it does:** K8s kills all existing pods first, waits for them to die, and then starts the new ones.
- **Mental Model:** You close the restaurant for 1 hour to paint the walls.
- **Problem:** **Downtime.** Your users get a 404 or a timeout while the new pods are starting.
- **Why use it:** When your app doesn't support running two different versions at the same time (e.g., a massive database migration that isn't backward compatible).

### B. RollingUpdate (The "Default Path")

- **What it does:** K8s replaces pods one by one. It starts one new pod, waits for it to be ready, then kills one old pod.
- **Mental Model:** You replace the chairs in the restaurant one by one while people are still eating.
- **Benefit:** **Zero Downtime.** There is always a pod available to handle requests.

---

## 2. Advanced Strategies: Blue-Green and Canary

K8s doesn't have these "built-in" to the Deployment object, but we achieve them by manipulating **Services** and **Ingress**.

### Blue-Green (The Safety Switch)

- **How it works:** You deploy a completely new set of pods (Green) alongside the old ones (Blue). Once Green is ready, you tell the **Service** or **Load Balancer** to point to Green.
- **Analogy:** You build a whole new bridge next to the old one. Once it's done, you move the traffic cones to the new bridge in one second.
- **Benefit:** Instant rollback if something goes wrong.

### Canary (The Taste Test)

- **How it works:** You send **5% of traffic** to the new version and 95% to the old version. You watch for errors. If it's stable, you move more traffic.
- **Analogy:** You give a small piece of the new cake to one person. If they don't get sick, you serve it to the whole party.

---

## 3. Probes: The "Heartbeat" of Stability

Probes are how Kubernetes knows if your app is actually working. Without probes, a "Zero Downtime" deployment is just a fantasy.

1.  **Liveness Probe (Is it alive?):**
    - **Action if it fails:** K8s **restarts** the container.
    - **Use case:** Your app has a deadlock and is "stuck."
2.  **Readiness Probe (Is it ready?):**
    - **Action if it fails:** K8s **stops sending traffic** to this pod.
    - **Use case:** Your app is still loading a 1GB cache into memory.
3.  **Startup Probe (Is it done starting?):**
    - **Use case:** For slow-starting apps (like Java/Spring). It disables the other probes until the app is fully initialized so they don't kill the app while it's still trying to wake up.

---

## 4. What "Zero" really means in Zero-Downtime

In production, even with a "RollingUpdate," you might still see errors. Why?

### The Race Condition

When you kill a Pod:

1.  K8s sends a `SIGTERM` signal to the app.
2.  The app starts shutting down.
3.  **BUT**, the Load Balancer might still be sending new requests for a few milliseconds because it hasn't "heard" the news yet.

### How to fix it:

- **Graceful Shutdown:** Your app must catch the signal and finish handling existing requests before exiting.
- **PreStop Hooks:** You can tell K8s to "wait 5 seconds" after the signal before actually killing the container, giving the network time to catch up.

---

### Key Takeaways

- **RollingUpdate** is the standard; **Recreate** is the fallback for breaking changes.
- **Readiness Probes** are the most important for deployments. If you don't use them, K8s will send traffic to pods that aren't ready yet.
- **Blue-Green** is safer but uses 2x the memory/cost during the switch.

### Common Mistakes

- Using a **Liveness Probe** that points to a database. If the DB goes down, K8s will restart **every single web server**, making the outage worse. Liveness should only check the app itself.
- Setting the "initialDelaySeconds" too low. K8s will start checking the app before it even has a chance to boot, causing an infinite restart loop.

### Production Tips

- Always implement a `/healthz` and `/readyz` endpoint in your code.
- Use `maxUnavailable` and `maxSurge` in your YAML to control how fast a rolling update happens. (e.g., "Don't let more than 1 pod be down at a time").

---

### 💡 Knowledge Check

Your app takes 60 seconds to load a large configuration file before it can answer requests.

1. Which probe will ensure users don't see "Connection Refused" during a deployment?
2. If this probe fails, what will Kubernetes do to the pod?
