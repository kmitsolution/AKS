# 🧭 **AKS + NGINX Ingress Controller Setup Guide (End-to-End)**

---

## ⚙️ **1️⃣ Create / Connect to AKS Cluster**

If you already have an AKS cluster, skip this step.

```bash
# Create a new AKS cluster
az group create --name aks-rg --location eastus
az aks create --resource-group aks-rg --name aks-demo --node-count 2 --enable-addons monitoring --generate-ssh-keys

# Get cluster credentials
az aks get-credentials --resource-group aks-rg --name aks-demo
```

Verify connection:

```bash
kubectl get nodes
```

---

## 🌐 **2️⃣ Deploy NGINX Ingress Controller (Azure LoadBalancer Version)**

> **Important:** Use the *cloud* deployment manifest for AKS — **not** the baremetal one.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.1/deploy/static/provider/cloud/deploy.yaml
```

Check namespace and components:

```bash
kubectl get ns
kubectl get all -n ingress-nginx
```

✅ You should see:

* Deployment: `ingress-nginx-controller`
* Service: `ingress-nginx-controller` (Type: LoadBalancer)
* EXTERNAL-IP assigned after a minute

Confirm LoadBalancer IP:

```bash
kubectl get svc -n ingress-nginx
```

Example:

```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller   LoadBalancer   10.0.152.216   51.8.202.220    80:30712/TCP,443:31846/TCP   1m
```

---

## 🧱 **3️⃣ Deploy Backend Applications (Pods & Services)**

We’ll create two simple apps — one with NGINX and one with Tomcat.

```bash
# Create two pods
kubectl run news-pod --image nginx
kubectl run mail-pod --image tomcat

# Expose them internally via ClusterIP
kubectl expose pod news-pod --name news --type ClusterIP --port 80 --target-port 80
kubectl expose pod mail-pod --name mail --type ClusterIP --port 8080 --target-port 8080
```

Verify services:

```bash
kubectl get svc
```

✅ You should see:

```
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
mail         ClusterIP   10.0.152.45    <none>        8080/TCP  1m
news         ClusterIP   10.0.111.32    <none>        80/TCP    1m
```

---

## 🌍 **4️⃣ Create Ingress Resource**

Create an `ingress.yaml` file:

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx   # Must match 'kubectl get ingressclass' output
  rules:
  - http:
      paths:
      - path: /news
        pathType: Prefix
        backend:
          service:
            name: news
            port:
              number: 80
      - path: /mail
        pathType: Prefix
        backend:
          service:
            name: mail
            port:
              number: 8080
```

Apply it:

```bash
kubectl apply -f ingress.yaml
```

Verify ingress:

```bash
kubectl get ingress
kubectl describe ingress minimal-ingress
```

✅ Expected:

```
Address: 51.8.202.220
Rules:
  /news -> news:80
  /mail -> mail:8080
```

---

## 🔍 **5️⃣ Validate Setup**

### 🧩 Check if controller picked up the ingress

```bash
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

Look for logs like:

```
Added Ingress: default/minimal-ingress
```

### 🧩 Test connectivity inside cluster

```bash
kubectl run testpod --image=busybox:1.28 --restart=Never -it -- sh
wget -qO- http://news
wget -qO- http://mail:8080
```

### 🧩 Test from outside (your machine)

```bash
curl -v http://<LoadBalancer_Public_IP>/news
curl -v http://<LoadBalancer_Public_IP>/mail
```

✅ Expected responses:

* `/news` → NGINX default welcome page
* `/mail` → Apache Tomcat default page

---

## 🔄 **6️⃣ Common Troubleshooting**

| Symptom                                    | Cause                              | Fix                                                 |
| ------------------------------------------ | ---------------------------------- | --------------------------------------------------- |
| `curl` to LoadBalancer IP hangs            | Wrong manifest (baremetal)         | Redeploy using **provider/cloud/deploy.yaml**       |
| Ingress shows node IP instead of LB IP     | Controller from baremetal manifest | Delete and redeploy NGINX with cloud manifest       |
| Ingress not detected                       | Wrong `ingressClassName`           | Use `kubectl get ingressclass` and update your YAML |
| Pod works via NodePort but not via Ingress | Controller not bound to LB         | Ensure service type = `LoadBalancer`                |
| 404 error from Ingress                     | Path mismatch or missing rewrite   | Add `nginx.ingress.kubernetes.io/rewrite-target: /` |

---

## 🧰 **7️⃣ Verification Commands Summary**

```bash
# Check ingress controller deployment
kubectl get all -n ingress-nginx

# Check ingress resource
kubectl get ingress
kubectl describe ingress minimal-ingress

# Check LoadBalancer IP
kubectl get svc -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Test routes
curl -v http://<LB-IP>/news
curl -v http://<LB-IP>/mail
```

---

## 💾 **8️⃣ Optional — Clean Up**

```bash
kubectl delete ingress minimal-ingress
kubectl delete svc news mail
kubectl delete pod news-pod mail-pod
kubectl delete ns ingress-nginx
```

---

## 🧾 **Final Architecture**

```
              +------------------------------+
              |     Azure LoadBalancer       |
              |   (Public IP: 51.8.202.220)  |
              +--------------+---------------+
                             |
                80/443 --->  Ingress NGINX Controller
                             |
                +------------+------------+
                |                         |
        /news → news Service (ClusterIP)  |
        /mail → mail Service (ClusterIP)  |
                |                         |
         NGINX Pod (80)           Tomcat Pod (8080)
```

---

## 🪶 **Full Command Script (Copy-Paste Ready)**

```bash
# Step 1 - Deploy NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.1/deploy/static/provider/cloud/deploy.yaml

# Step 2 - Deploy sample pods and services
kubectl run news-pod --image nginx
kubectl run mail-pod --image tomcat
kubectl expose pod news-pod --name news --type ClusterIP --port 80 --target-port 80
kubectl expose pod mail-pod --name mail --type ClusterIP --port 8080 --target-port 8080

# Step 3 - Create ingress resource
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /news
        pathType: Prefix
        backend:
          service:
            name: news
            port:
              number: 80
      - path: /mail
        pathType: Prefix
        backend:
          service:
            name: mail
            port:
              number: 8080
EOF

# Step 4 - Verify ingress and test
kubectl get svc -n ingress-nginx
kubectl get ingress
kubectl describe ingress minimal-ingress
curl -v http://<LB-PUBLIC-IP>/news
curl -v http://<LB-PUBLIC-IP>/mail
```

---

Would you like me to output this as a **Markdown (.md)** document (formatted with headings, code blocks, etc.) that you can directly save as your project notes or documentation?
I can generate it cleanly for your script folder.
