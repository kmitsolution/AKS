# 🧭 **1️⃣ What Are Node Pools in AKS?**

In AKS, your cluster’s nodes are organized into **node pools**.
Each node pool is a group of VMs (same size + config) in the same **Virtual Machine Scale Set (VMSS)**.

You can have **multiple node pools** in a single AKS cluster — each pool can run a different type of workload (e.g. system pods vs user apps).

---

# 🧩 **2️⃣ System Node Pool vs User Node Pool**

| Type                 | Purpose                                 | What Runs Here                    | Key Properties                               |
| -------------------- | --------------------------------------- | --------------------------------- | -------------------------------------------- |
| **System Node Pool** | Runs critical system components         | Core AKS + Kubernetes system pods | Required; created automatically with cluster |
| **User Node Pool**   | Runs user workloads (apps, deployments) | Your application pods             | Optional; can be created as needed           |

---

## 🔹 **System Node Pool**

* Created automatically when you create the AKS cluster.
* Named like: `nodepool1` or `systempool`.
* Runs:

  * `kube-system` namespace pods (kube-proxy, coredns)
  * Monitoring agents (if enabled)
  * `omsagent`, `metrics-server`, `csi-driver`, etc.
* **Taint applied:**
  AKS taints the system node pool as:

  ```bash
  CriticalAddonsOnly=true:NoSchedule
  ```

  This ensures only system-critical pods are scheduled here.

✅ You **should not deploy** your application workloads on this pool.

---

## 🔹 **User Node Pool**

* You create this to run your **application workloads**.
* Typically larger node sizes (more CPU/RAM).
* You can:

  * Choose a VM SKU
  * Choose OS (Linux/Windows)
  * Set autoscaling and labels
  * Use different taints/tolerations

---

# ⚙️ **3️⃣ Check Existing Node Pools**

List all node pools:

```bash
az aks nodepool list --cluster-name <aks-cluster-name> --resource-group <rg-name> -o table
```

Example output:

```
Name        OsType    Mode      Count  VMSize     MaxPods
----------- --------- --------- ------ ---------- -------
systempool  Linux     System    1      Standard_DS2_v2  30
userpool1   Linux     User      2      Standard_DS3_v2  30
```

Here:

* `systempool` → System node pool
* `userpool1` → User node pool

---

# 🧱 **4️⃣ Create a User Node Pool**

You can create a user node pool anytime with the Azure CLI.

### ✅ Basic command:

```bash
az aks nodepool add \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --node-count 2 \
  --node-vm-size Standard_DS3_v2 \
  --mode User
```

### ✅ Explanation:

| Flag             | Description                       |
| ---------------- | --------------------------------- |
| `--name`         | Name of the new node pool         |
| `--mode User`    | Marks it as a *user* node pool    |
| `--node-vm-size` | Choose a VM size                  |
| `--node-count`   | Number of nodes                   |
| `--os-type`      | Linux or Windows (default: Linux) |

You can confirm mode:

```bash
az aks nodepool show --resource-group <rg-name> --cluster-name <aks-name> --name userpool1 --query mode -o tsv
```

Output:

```
User
```

---

# ⚙️ **5️⃣ Create a System Node Pool (Optional / Replace Default)**

If you want to create a **custom system node pool** (e.g., smaller VM size for system pods):

```bash
az aks nodepool add \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name systempool \
  --node-count 1 \
  --node-vm-size Standard_B4ms \
  --mode System
```

Then, you can delete the default one if needed:

```bash
az aks nodepool delete \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name nodepool1
```

⚠️ **Note:** You must always have **at least one system node pool** in the cluster.

---

# 🧩 **6️⃣ Identify Which Node Pool a Pod Is Running On**

You can check the node labels and see which pool a pod is assigned to:

```bash
kubectl get nodes --show-labels
```

Look for:

```
agentpool=systempool, kubernetes.azure.com/mode=system
agentpool=userpool1,  kubernetes.azure.com/mode=user
```

Or per pod:

```bash
kubectl get pod <pod-name> -o wide
```

---

# 🧠 **7️⃣ Best Practices**

✅ **Separate workloads:**

* System components → System Node Pool
* Your apps → User Node Pool

✅ **Optimize resources:**

* System pool → smaller, fewer nodes
* User pool → larger nodes, autoscale enabled

✅ **Use labels & taints:**
Label your pools:

```bash
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --labels role=userpool env=prod
```

Then in your deployments:

```yaml
nodeSelector:
  role: userpool
```

✅ **Use autoscaler for user pools:**

```bash
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5
```

---

# 📘 **8️⃣ Verification**

To verify what runs where:

```bash
kubectl get pods -A -o wide | grep <nodepool-name>
```

✅ You’ll see:

* `kube-system` pods on **systempool**
* Your app pods on **userpool1**

---

# 🧾 **Summary Table**

| Property               | System Node Pool                     | User Node Pool                |
| ---------------------- | ------------------------------------ | ----------------------------- |
| Mode                   | `System`                             | `User`                        |
| Runs                   | AKS system components                | Your workloads                |
| Default                | Yes                                  | Optional                      |
| Typical size           | Small (e.g. DS2_v2)                  | Larger (DS3_v2, DS5_v2, etc.) |
| Taints                 | `CriticalAddonsOnly=true:NoSchedule` | None                          |
| Recommended node count | 1–2                                  | Based on workload             |
| Can scale?             | Yes                                  | Yes                           |
| Required for AKS       | ✅                                    | ❌                             |

---

