# Phase 7: Observability & Debugging

## 1. The Three Pillars of Observability

In a distributed system, "Monitoring" (knowing IF something is wrong) is not enough. We need **Observability** (knowing WHY something is wrong). We do this by looking through three different "lenses."

### The Mental Model: The Hospital MRI

- **Metrics (The Heart Rate Monitor):** High-level numbers (CPU, Memory, Error Rate). It tells you "The patient's heart rate is spiking!" but doesn't tell you why.
- **Logs (The Patient's Diary):** Granular text messages from the app. It says: "At 2 PM, I felt a sharp pain in my stomach." It gives you the **context**.
- **Tracing (The Dye in the Bloodstream):** Tracks a single request as it jumps from the Frontend to the Backend to the Database. It tells you exactly **where** the blockage is.

---

## 2. The Tools of the Trade

### A. Metrics (Prometheus & Grafana)

- **Prometheus:** The database that "scrapes" (pulls) metrics from your pods.
- **Grafana:** The dashboard that draws the pretty charts.
- **Why use it?** To see trends. Is memory usage growing over time (Memory Leak)? Does traffic always spike at 9 AM?

### B. Tracing (OpenTelemetry & Jaeger)

- **Problem:** If a request takes 10 seconds, which microservice is slow?
- **Solution:** OpenTelemetry attaches a "Trace ID" to the request. You can see a timeline of exactly how long each step took.

---

## 3. The `kubectl` Debugging Toolkit

When production is on fire, these are the 4 commands you use in order.

### 1. `kubectl get pods`

- **Check for:** `CrashLoopBackOff`, `Pending`, or high `RESTARTS`.
- **Analogy:** Checking if the building is still standing.

### 2. `kubectl describe pod <name>`

- **Check for:** The "Events" at the bottom.
- **What it reveals:** "Failed to pull image," "Insufficient CPU," or "MountVolume.SetUp failed."
- **Analogy:** Looking at the "Warning" lights on a car dashboard.

### 3. `kubectl logs <name>`

- **Check for:** Application stack traces, "Failed to connect to DB," or "JSON Parse Error."
- **Tip:** Use `-f` to stream logs or `--previous` to see why the _last_ instance crashed.
- **Analogy:** Reading the black box from a plane crash.

### 4. `kubectl exec -it <name> -- sh`

- **Action:** Jump inside the container to run commands.
- **Use case:** Can I ping the DB IP? Is the config file actually there?
- **Analogy:** Sending a doctor inside the room to check the patient.

---

## 4. The "Worked Locally, Broken in Prod" Checklist

If your Docker image works on your Mac but fails in K8s, it's usually one of these three:

1.  **Networking:** In Prod, you must use Service names (`my-svc`) instead of `localhost`.
2.  **RBAC/Permissions:** Your app might be trying to list other pods, but the K8s `ServiceAccount` doesn't have permission.
3.  **Resources:** Your production memory limit might be smaller than your laptop's RAM, causing an instant `OOMKill`.

---

## 5. The Detective Mindset (Workflow)

When an incident starts:

1.  **Don't Panic.** Look at the **Metrics** first to see the scale. (Is it all users or just 1%?)
2.  **Isolate the Component.** Use **Tracing** to see where the errors start.
3.  **Find the Root Cause.** Use **Logs** to see the specific error message.
4.  **Confirm the fix.** Check the **Metrics** again. Did the error line go down to zero?

---

### Key Takeaways

- **Metrics** tell you there is a fire; **Logs** tell you what started the fire; **Tracing** tells you which room the fire is in.
- **`kubectl describe`** is for infrastructure issues; **`kubectl logs`** is for application issues.
- Always check the **Events** before the **Logs**.

### Common Mistakes

- **Log Spamming:** Printing too much data. This will slow down your app and cost a fortune in storage.
- **No Readiness Probes:** If your app crashes but has no probe, K8s might keep sending traffic to a "Zombie" pod.

### Production Tips

- Include a **Correlation ID** in every log line. This allows you to search for all logs related to a single user request across 5 different microservices.

---

### 💡 Knowledge Check

Your "Order Service" is suddenly slow. `kubectl get pods` shows everything is `Running`.

1. Which tool (Metrics, Logs, or Tracing) should you check first to see _which_ part of the ordering process is slow?
2. If you see a timeout error in the logs, what `kubectl` command would you use to verify if the pod can actually reach the database?
