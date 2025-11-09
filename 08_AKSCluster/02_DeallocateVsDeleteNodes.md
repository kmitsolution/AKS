
# 🧭 **1️⃣ What is Scale-Down Mode in AKS Node Pools?**

When AKS (via the **Cluster Autoscaler**) or **manual scaling** reduces the number of nodes in a node pool, AKS must decide **what to do** with the underlying Azure VMs.

That behavior is controlled by the **scale-down mode**.

---

# ⚙️ **2️⃣ Scale-Down Mode Options**

AKS supports **two modes** for node pools:

| Mode                     | Description                                    | Behavior                    | Cost Impact                                       |
| ------------------------ | ---------------------------------------------- | --------------------------- | ------------------------------------------------- |
| **`Delete`** *(default)* | Deletes VMs permanently when scaled down       | Removes VMs from VMSS       | 💰 You stop paying for those VMs completely       |
| **`Deallocate`**         | Stops (deallocates) VMs but keeps them in VMSS | Retains VMs and their disks | 💾 You pay only for OS & data disks (not compute) |

---

## 🔹 Example Behavior

| Operation            | Delete Mode                                 | Deallocate Mode                  |
| -------------------- | ------------------------------------------- | -------------------------------- |
| Scale down node pool | VMs are **deleted**                         | VMs are **stopped/deallocated**  |
| Scale up again       | AKS **creates new VMs**                     | AKS **restarts deallocated VMs** |
| Time to scale up     | Slightly longer (new VMs need to provision) | Faster (VMs already exist)       |
| Billing              | No charge                                   | Pay for storage (disks)          |

---

# 🧩 **3️⃣ When to Use Each Mode**

| Use Case                                   | Recommended Mode | Reason                                      |
| ------------------------------------------ | ---------------- | ------------------------------------------- |
| **Production / Auto-scaling clusters**     | `Delete`         | Keeps environment clean, lowers costs       |
| **Development / Test environments**        | `Deallocate`     | Faster restart when scaling back up         |
| **Workloads with predictable daily usage** | `Deallocate`     | Reduce cost during off-hours, resume faster |
| **Ephemeral / Spot node pools**            | `Delete`         | Spot VMs can be recreated easily            |

---

# 🧱 **4️⃣ How to Set Scale-Down Mode**

You can set it during **node pool creation** or **update an existing pool**.

---

### 🏗️ **Create Node Pool with Specific Mode**

```bash
az aks nodepool add \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --node-count 2 \
  --node-vm-size Standard_DS3_v2 \
  --mode User \
  --scale-down-mode Deallocate
```

✅ This will ensure when AKS scales down the pool, the nodes are **deallocated** (not deleted).

---

### 🔄 **Update Existing Node Pool Scale-Down Mode**

```bash
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --scale-down-mode Delete
```

or

```bash
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --scale-down-mode Deallocate
```

---

### 🔍 **Check the Current Mode**

```bash
az aks nodepool show \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --query scaleDownMode -o tsv
```

Output:

```
Delete
```

or

```
Deallocate
```

---

# 🧩 **5️⃣ Manual Node Pool Scale Operations**

### 🔹 **Scale Down Manually**

```bash
az aks nodepool scale \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --node-count 1
```

👉 Depending on `scaleDownMode`, AKS will either:

* Delete the VM (if mode = `Delete`)
* Deallocate it (if mode = `Deallocate`)

### 🔹 **Scale Up Manually**

```bash
az aks nodepool scale \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --node-count 3
```

If `Deallocate` mode was used earlier — previously stopped VMs may be restarted instead of new ones being created.

---

# 💾 **6️⃣ Example Verification on Azure Portal**

In the Azure Portal:

* Navigate to your **AKS-managed VM Scale Set** (e.g., `aks-userpool1-xxxx-vmss`).
* After scaling down:

  * If **Delete** mode: nodes disappear entirely from VMSS.
  * If **Deallocate** mode: nodes appear with **Status = Stopped (deallocated)**.

---

# 🧠 **7️⃣ Best Practices**

✅ **Recommended setup:**

| Environment          | Scale-Down Mode                 |
| -------------------- | ------------------------------- |
| Production           | `Delete`                        |
| Dev / Test           | `Deallocate`                    |
| CI/CD / Lab          | `Deallocate`                    |
| Autoscaling clusters | `Delete` (to keep clean states) |

✅ **Combine with cluster autoscaler:**
Enable autoscaling with your preferred mode:

```bash
az aks nodepool update \
  --resource-group <rg-name> \
  --cluster-name <aks-name> \
  --name userpool1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 5 \
  --scale-down-mode Delete
```

✅ **Watch out for costs:**
Even when deallocated:

* You stop paying for compute 💰
* But still pay for disks 💾

---

# 🧾 **8️⃣ Quick Reference Summary**

| Command                                               | Purpose                              |
| ----------------------------------------------------- | ------------------------------------ |
| `az aks nodepool show --query scaleDownMode`          | Check current scale-down mode        |
| `az aks nodepool update --scale-down-mode Delete`     | Set mode to delete                   |
| `az aks nodepool update --scale-down-mode Deallocate` | Set mode to deallocate               |
| `az aks nodepool add --scale-down-mode Deallocate`    | Create new pool with deallocate mode |
| `az aks nodepool scale`                               | Scale up/down node pool manually     |

---

# 📘 Example — Full Workflow

```bash
# Create a user node pool with deallocate mode
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name aks-demo \
  --name userpool1 \
  --mode User \
  --node-vm-size Standard_DS3_v2 \
  --node-count 2 \
  --scale-down-mode Deallocate

# Verify
az aks nodepool show -g aks-rg -n userpool1 --cluster-name aks-demo --query scaleDownMode

# Scale down manually
az aks nodepool scale -g aks-rg -n userpool1 --cluster-name aks-demo --node-count 0

# Check in Azure Portal → VMSS → VMs → should show “Stopped (deallocated)”
```

---

