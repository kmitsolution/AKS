## 🧭 **1️⃣ Kubernetes Dashboard Overview — What’s the Difference on AKS?**

| Feature              | Vanilla Kubernetes                     | AKS                                    |
| -------------------- | -------------------------------------- | -------------------------------------- |
| Dashboard Deployment | Same YAML manifest                     | ✅ Same                                 |
| Authentication       | Token/Service Account                  | ✅ Same (or Azure AD if enabled)        |
| Network Access       | NodePort/Proxy                         | ✅ Use `kubectl proxy` or Azure Bastion |
| Permissions          | ClusterRoleBindings                    | ✅ Use Azure AD RBAC or K8s RBAC        |
| Hosting              | Cluster control plane managed by Azure | ✅ Managed                              |

✅ So: deployment YAML is the same — but the **auth and access** methods differ slightly.

---

## ⚙️ **2️⃣ Deploy Kubernetes Dashboard in AKS**

Run this on your local machine (with your AKS kubeconfig set):

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Verify:

```bash
kubectl get pods -n kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard
```

✅ Output example:

```
NAME                                    READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-xxxx               1/1     Running   0          1m
```

---

## 🔐 **3️⃣ Create a Dashboard Admin Service Account**

> ⚠️ For production, create namespace-scoped roles instead of cluster-admin.
> But for initial setup/testing, cluster-admin is fine.

```yaml
# dashboard-admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: dashboard-admin-user
  namespace: kubernetes-dashboard
```

Apply:

```bash
kubectl apply -f dashboard-admin-user.yaml
```

---

## 🔑 **4️⃣ Get Access Token**

In AKS (Kubernetes v1.24+), tokens are short-lived, so use:

```bash
kubectl -n kubernetes-dashboard create token dashboard-admin-user
```

✅ Example output:

```
eyJhbGciOiJSUzI1NiIsInR5cCIg...
```

> Note: older versions used secrets (`kubectl describe secret`), but AKS now uses **service account tokens**.

---

## 🌐 **5️⃣ Access Dashboard via Proxy (Recommended)**

Start the proxy:

```bash
kubectl proxy
```

Output:

```
Starting to serve on 127.0.0.1:8001
```

Now open:
👉 [http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/)

✅ You’ll see the Dashboard login screen.

Select **“Token”**, paste the token from the previous step, and click **Sign In**.

---

## ⚠️ **6️⃣ Avoid Exposing Dashboard Publicly**

In AKS, you should **not** expose Dashboard with `LoadBalancer` — it would expose your cluster control plane to the internet.

If you absolutely must test it via IP, do:

```bash
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard
```

Change:

```yaml
type: ClusterIP
```

→

```yaml
type: LoadBalancer
```

Then check:

```bash
kubectl get svc -n kubernetes-dashboard
```

**But** — this is insecure unless protected behind Azure Firewall, Private Endpoint, or VPN.

✅ Best practice: Use `kubectl proxy`, Azure Bastion, or a private jump VM inside your AKS VNet.

---

## 🧱 **7️⃣ (Optional) Create a Read-Only User (Safer)**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-readonly
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: dashboard-readonly
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: dashboard-readonly
  namespace: kubernetes-dashboard
```

Apply:

```bash
kubectl apply -f dashboard-readonly.yaml
kubectl -n kubernetes-dashboard create token dashboard-readonly
```

✅ Use that token to log in as a **read-only** user.

---

## 🧠 **8️⃣ (Optional) Integrate Dashboard with Azure AD**

If your AKS cluster has **Azure AD authentication enabled** (`--enable-aad`), you can let users log in with their Azure credentials instead of service account tokens.

Steps overview:

1. Enable managed AAD when creating AKS:

   ```bash
   az aks create -g aksgroup -n akscluster --enable-aad --enable-azure-rbac
   ```
2. Give users roles:

   ```bash
   az role assignment create \
     --assignee user@yourtenant.onmicrosoft.com \
     --role "Azure Kubernetes Service RBAC Reader" \
     --scope $(az aks show -n akscluster -g aksgroup --query id -o tsv)
   ```
3. Access dashboard — browser redirects to Azure login.

This is the **most secure way** to use Dashboard in enterprise environments.

---

## 🧾 **9️⃣ Troubleshooting**

| Problem                 | Cause                         | Fix                              |
| ----------------------- | ----------------------------- | -------------------------------- |
| “Forbidden” after login | SA has no role binding        | Create ClusterRoleBinding        |
| “Token expired”         | Token lifetime expired        | Re-run `kubectl create token`    |
| “404 Not Found”         | Proxy URL wrong               | Ensure `/proxy/` path is correct |
| “LoadBalancer pending”  | Public IP restrictions in AKS | Use `kubectl proxy` instead      |

---

## ✅ **10️⃣ Quick Command Summary**

| Step                 | Command                                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------------------------- |
| Deploy Dashboard     | `kubectl apply -f https://raw.githubusercontent.com/.../recommended.yaml`                                  |
| Create admin account | `kubectl apply -f dashboard-admin-user.yaml`                                                               |
| Get token            | `kubectl -n kubernetes-dashboard create token dashboard-admin-user`                                        |
| Start proxy          | `kubectl proxy`                                                                                            |
| Open UI              | `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/` |

---

## 🧩 **11️⃣ (Optional) Remove Dashboard**

If you need to remove it later:

```bash
kubectl delete -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
kubectl delete -f dashboard-admin-user.yaml
```

---

### ✅ **In Short — AKS Dashboard Setup**

> 1️⃣ Apply Dashboard manifest
> 2️⃣ Create ServiceAccount & ClusterRoleBinding
> 3️⃣ Get access token
> 4️⃣ Start `kubectl proxy`
> 5️⃣ Access Dashboard via `localhost:8001`

---


