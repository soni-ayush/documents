# 🧵 Kubernetes Resource Management – CPU & Memory (Deep Dive)

---

## 📦 1. Resource Types in Kubernetes

| Resource | `requests` (soft)                | `limits` (hard)                          | Linux Mechanism                     |
|----------|----------------------------------|------------------------------------------|-------------------------------------|
| CPU      | Scheduling hint (soft guarantee) | Throttling cap (cannot use beyond limit) | `cpu.shares`, `cpu.cfs_quota_us`    |
| Memory   | Scheduling guarantee              | Hard cap (goes OOM if exceeded)          | `memory.limit_in_bytes`             |

---

## 📁 2. Where Are These Limits Stored?

Kubernetes uses **Linux cgroups** to enforce these limits at runtime:

- **Memory Limit:**  
  `/sys/fs/cgroup/memory/kubepods/<pod-id>/memory.limit_in_bytes`

- **CPU Control:**  
  ```
  /sys/fs/cgroup/cpu/kubepods/<pod-id>/cpu.shares
  /sys/fs/cgroup/cpu/kubepods/<pod-id>/cpu.cfs_quota_us
  /sys/fs/cgroup/cpu/kubepods/<pod-id>/cpu.cfs_period_us
  ```

---

## 💥 3. OOM Killer in Kubernetes

### 🔎 When does it run?

The **Out Of Memory (OOM) Killer** is triggered by the Linux kernel when:
- Memory allocation fails
- A process/container exceeds its memory limit (`memory.limit_in_bytes`)

### 🧠 Two Types of OOM Killers

| Type              | Applies To                                      | Behavior                                                                 |
|-------------------|--------------------------------------------------|--------------------------------------------------------------------------|
| **Global OOM Killer** | Processes **not in a memory cgroup**             | Looks at all processes, kills one with highest OOM score                  |
| **Cgroup OOM Killer** | **Pods/containers with memory limits** (via cgroups) | Triggers when the pod exceeds its limit. Only affects that pod’s cgroup. |

### 📊 How it Chooses What to Kill

- Linux uses **`oom_score_adj`** to adjust importance.
- The process with the **highest memory usage × oom_score_adj** is killed first.

Example log:
```
Memory cgroup out of memory: Kill process 12345 (java) score 987 or sacrifice child
Killed process 12345 (java) total-vm:1024000kB, anon-rss:800000kB
```

---

## ⚙️ 4. CPU Scheduling in Kubernetes & Linux

### 🔘 Key Concepts

- **CPU is compressible** — it can be shared and overcommitted.
- **Memory is not** — must fit into physical RAM.

CPU management in Kubernetes uses two mechanisms:

| Mechanism              | Purpose                        |
|------------------------|--------------------------------|
| `cpu.shares`           | Soft relative weight (fairness)|
| `cpu.cfs_quota_us`     | Hard runtime quota (throttling)|

---

## 🧮 5. `cpu.shares` – Soft Fairness (via CFS)

- Used by the **Completely Fair Scheduler (CFS)**.
- Helps the kernel fairly share CPU based on virtual runtime (`vruntime`).
- More shares = slower vruntime growth = more CPU time when contended.

### 🔢 Example:

| Container | cpu.shares |
|-----------|------------|
| A         | 512        |
| B         | 256        |
| C         | 256        |

→ A gets ~50% CPU, B and C ~25% each (if all are busy).

---

## 🔐 6. `cpu.cfs_quota_us` – Hard CPU Limit (Bandwidth Control)

- CFS Bandwidth Controller uses:
  - `cpu.cfs_period_us` (typically 100,000μs = 100ms)
  - `cpu.cfs_quota_us` (how much time the process can run in that window)

Example:  
For a 200m CPU limit:
- Quota = 20,000μs → means the container can only run **20ms per 100ms**, i.e., 20% CPU.

If it uses all its time before 100ms → **it gets throttled until the next window**.

---

## 🧪 7. How CPU Requests and Limits Work Together (10s Example)

### Setup:

| Container | CPU Request | CPU Limit | cpu.shares | Quota (per 100ms) |
|-----------|-------------|-----------|------------|-------------------|
| A         | 500m        | 500m      | 512        | 50,000μs (50%)    |
| B         | 250m        | 250m      | 256        | 25,000μs (25%)    |
| C         | 250m        | ❌ None   | 256        | Unlimited         |

### Every 100ms:
- A gets 50ms → throttled
- B gets 25ms → throttled
- C uses the rest (~25ms) or more if A/B sleep

### Total after 10,000ms:

| Container | Total CPU Time |
|-----------|----------------|
| A         | 5000ms         |
| B         | 2500ms         |
| C         | 2500ms         |

---

## 🧠 8. Combined Behavior: Scheduling vs Runtime

| Control Type | Description                                   | Affects            |
|--------------|-----------------------------------------------|--------------------|
| `cpu.shares` | Fair share during CPU contention              | Scheduling fairness|
| `cpu.cfs_quota` | Hard runtime budget (enforced every 100ms) | Runtime throttling |
| Memory Limit | Hard RAM cap                                  | OOM killer         |

---

## ✅ 9. Summary & Interview Takeaways

| Concept                    | What to Say                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| CPU Requests               | “Soft fairness for scheduling using cpu.shares”                            |
| CPU Limits                 | “Hard cap using cpu.cfs_quota_us; enforced every 100ms”                    |
| Memory Limits              | “Enforced using cgroup; triggers OOM if exceeded”                          |
| Global vs Cgroup OOM       | “Global applies to all processes; Cgroup OOM is isolated to that container”|
| CPU + Memory Best Practice | “Use requests for scheduling, limits for control; always set memory limits”|
| CFS                        | “Uses vruntime and cpu.shares for proportional fairness”                   |
| Bandwidth Control          | “Strict CPU cap; prevents noisy neighbor problems”                         |

---

## 📚 Further Reading

- [Understanding Resource Limits in Kubernetes: Memory](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9)
- [Understanding Resource Limits in Kubernetes: CPU Time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)

