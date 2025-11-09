# 🧭 **AKS Resource Reservation & `kubeReserved` Explained**

When you deploy an AKS cluster, **not all node resources** are available for your pods.
Some resources are **reserved** for the **Kubernetes system components** (like kubelet, kube-proxy, container runtime, and OS processes).

These reservations are controlled by the following:

* **`kubeReserved`** → for Kubernetes system daemons
* **`systemReserved`** → for OS-level processes (systemd, journald, etc.)
* **`evictionHard` thresholds** → for proactive memory pressure eviction

---

## 🧩 **1️⃣ Understanding Key Terms**

| Term               | Description                                                              |
| ------------------ | ------------------------------------------------------------------------ |
| **Capacity**       | The total physical resources of the node (e.g. 4 CPUs, 16Gi memory)      |
| **Allocatable**    | The remaining resources available for pods after reservations            |
| **kubeReserved**   | CPU, memory, and PID reserved for kubelet, container runtime, kube-proxy |
| **systemReserved** | Resources reserved for OS and background system processes                |
| **evictionHard**   | Threshold to trigger eviction when free resources go below limits        |

---

## ⚙️ **2️⃣ How to Inspect Current Reservation**

You already used this — let’s explain your exact commands 👇

```bash
kubectl describe node aks-nodepool1-29211799-vmss00000g | grep -e Hostname: -e Capacity: -e Allocatable -e 'Kubelet Version' -e pods: -e cpu: -e memory:
```

Shows:

* Node name
* Total `Capacity`
* `Allocatable` resources after reservation

Then you checked kubelet config:

```bash
kubectl proxy &
curl -sSL "http://localhost:8001/api/v1/nodes/aks-nodepool1-29211799-vmss00000g/proxy/configz" | grep -i kubeReserved
```

Output:

```json
"kubeReserved":{"cpu":"100m","memory":"1792Mi","pid":"1000"}
```

This means:

* 0.1 CPU core
* 1792 MiB memory
* 1000 PIDs
  are reserved for kubelet and related processes.

---

## 🧮 **3️⃣ How AKS Calculates Resource Reservations**

AKS **automatically configures `kubeReserved` and `systemReserved`** depending on the **VM size (node SKU)**.

### 🔹 **Memory reservation formula (approximation):**

| Node Size (RAM) | % Reserved for system/kube processes |
| --------------- | ------------------------------------ |
| < 1 GB          | 25%                                  |
| 1–4 GB          | 20%                                  |
| 4–8 GB          | 10%                                  |
| 8–16 GB         | 6%                                   |
| 16–128 GB       | 2%                                   |
| >128 GB         | ~1%                                  |

In practice, for AKS:

> **Allocatable ≈ 75–90% of total capacity** depending on node size.

So when you said “allocatable is ~25% less than capacity” — yes, that matches the **system + kube reservation overhead** on smaller node types.

### Example:

Suppose you have a node:

```
Capacity:
  cpu: 4
  memory: 16Gi
Allocatable:
  cpu: 3.8
  memory: 14.2Gi
```

Reserved:

* CPU ≈ 0.2 cores (5%)
* Memory ≈ 1.8 Gi (11%)

Those values come from combined:

```yaml
kubeReserved:
  cpu: 100m
  memory: 1792Mi
systemReserved:
  memory: 1024Mi
evictionHard:
  memory.available: 100Mi
```

---

## 🧩 **4️⃣ Where These Values Are Configured**

In AKS, these come from **the managed node agent settings** and are not directly editable (since kubelet flags are managed by Azure).
However, you can **view** them with:

```bash
curl -sSL "http://localhost:8001/api/v1/nodes/<node-name>/proxy/configz" | jq '.kubeletConfig'
```

Key sections:

* `.kubeletConfig.kubeReserved`
* `.kubeletConfig.systemReserved`
* `.kubeletConfig.evictionHard`

---

## ⚙️ **5️⃣ Resource Reservation Breakdown**

Example AKS kubelet config snippet:

```json
"kubeReserved": {
  "cpu": "100m",
  "memory": "1792Mi",
  "pid": "1000"
},
"systemReserved": {
  "cpu": "100m",
  "memory": "1024Mi"
},
"evictionHard": {
  "memory.available": "100Mi"
}
```

**Total Memory Reserved:**

```
1792Mi (kubeReserved)
+ 1024Mi (systemReserved)
+ 100Mi  (eviction buffer)
= 2916Mi (~2.9Gi)
```

On a 16Gi node:

```
(16Gi - 2.9Gi) = ~13.1Gi allocatable
≈ 82% allocatable memory
```

So, **Allocatable ≈ 75–85%** is totally normal.

---

## 🧠 **6️⃣ Optimization and Best Practices**

In AKS, you can’t directly modify `kubeReserved` (since it’s managed), but you can **optimize node utilization**:

### ✅ Recommendations:

1. **Use larger node sizes** (reduces % reserved overhead).
2. **Use autoscaling** to match workloads (Cluster Autoscaler).
3. **Request resources properly** in pods (`requests` and `limits`).
4. **Avoid overcommitting small nodes** — reservation ratio is higher.
5. **Use system node pool** for system pods (monitoring, ingress, etc.).
6. **Use user node pools** for workloads — keeps system overhead separate.

---

## 🧾 **7️⃣ Verify with a Real Example**

```bash
kubectl describe node aks-nodepool1-xxxxx | egrep "Capacity|Allocatable|cpu|memory"
```

Example output:

```
Capacity:
  cpu:                4
  memory:             16325536Ki
Allocatable:
  cpu:                3900m
  memory:             13842528Ki
```

Then check kubelet reservation:

```bash
curl -sSL "http://localhost:8001/api/v1/nodes/aks-nodepool1-xxxxx/proxy/configz" | jq '.kubeletConfig.kubeReserved'
```

Result:

```json
{
  "cpu": "100m",
  "memory": "1792Mi",
  "pid": "1000"
}
```

→ Difference = ~1792Mi + system + eviction buffer
→ Matches your observation of ~25% reservation.

---

## 📘 **8️⃣ TL;DR Summary**

| Parameter                | Purpose                   | Typical Value             |
| ------------------------ | ------------------------- | ------------------------- |
| **kubeReserved**         | For Kubernetes components | 100m CPU, ~1.7–2GB Memory |
| **systemReserved**       | For OS background tasks   | 100m CPU, ~1GB Memory     |
| **evictionHard**         | Prevent node exhaustion   | 100Mi memory buffer       |
| **Allocatable ≈ 75–85%** | What’s left for pods      | Auto-managed by AKS       |

---


