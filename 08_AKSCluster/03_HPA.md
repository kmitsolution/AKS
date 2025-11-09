
# 🚀 **Horizontal Pod Autoscaler (HPA) in Kubernetes / AKS — Step-by-Step Guide**

---

## 🧩 **1️⃣ What is HPA?**

**Horizontal Pod Autoscaler (HPA)** automatically scales the number of pod replicas in a deployment, replica set, or stateful set based on **observed CPU utilization** or **custom metrics**.

* **Scales out (adds pods)** when CPU load increases
* **Scales in (removes pods)** when CPU load decreases
* It **does not** apply to DaemonSets or Jobs

HPA continuously monitors metrics via the **metrics-server**.

---

## ⚙️ **2️⃣ Prerequisites**

✅ Kubernetes Cluster (AKS, minikube, etc.)
✅ `metrics-server` installed
✅ `kubectl` configured

### 🔹 Check if metrics-server is running:

```bash
kubectl get deployment metrics-server -n kube-system
```

If not present, install it:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Verify:

```bash
kubectl top nodes
kubectl top pods
```

If you see CPU/memory usage → metrics-server works ✅

---

## 🧱 **3️⃣ Create a Deployment**

Create file `nginx-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  labels:
    app: nginx-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.9
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f nginx-deployment.yaml
```

Check:

```bash
kubectl get deployments
kubectl get pods
```

✅ Output example:

```
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy    1/1     1            1           1m
```

---

## 🌐 **4️⃣ Expose the Deployment as a Service**

Create `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app: nginx-app
spec:
  selector:
    app: nginx-app
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31111
```

Apply it:

```bash
kubectl apply -f nginx-service.yaml
```

Verify:

```bash
kubectl get svc
```

✅ Output:

```
NAME         TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
my-service   NodePort   10.0.12.93    <none>        80:31111/TCP   1m
```

Now your NGINX is exposed at `http://<NodeIP>:31111`.

---

## ⚙️ **5️⃣ Create a Horizontal Pod Autoscaler**

You can create it two ways:

### 🔸 Option 1: Command Line

```bash
kubectl autoscale deployment nginx-deploy --cpu-percent=20 --min=1 --max=5
```

### 🔸 Option 2: YAML File

Create `nginx-hpa.yaml`:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deploy
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 10
```

Apply:

```bash
kubectl apply -f nginx-hpa.yaml
```

---

## 🧩 **6️⃣ Verify HPA**

```bash
kubectl get hpa
```

✅ Output:

```
NAME     REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx    Deployment/nginx-deploy  2%/10%     1         10        1          1m
```

* `TARGETS` shows current CPU utilization vs target (10%)
* `REPLICAS` shows how many pods are running

---

## ⚙️ **7️⃣ Generate Load to Trigger Autoscaling**

Use a loop to continuously hit the service (simulate traffic):

### On a node or pod inside cluster:

```bash
while true; do wget -q -O- http://localhost:31111; done
```

If accessing externally:

```bash
while true; do curl -s http://<NodeIP>:31111 > /dev/null; done
```

This will increase CPU usage of the pods.

---

## 🔍 **8️⃣ Monitor Scaling Behavior**

Run:

```bash
kubectl get hpa -w
```

You’ll see something like:

```
NAME     REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
nginx    Deployment/nginx-deploy  95%/10%     1         10        3         2m
nginx    Deployment/nginx-deploy  115%/10%    1         10        5         3m
```

Now check the deployment:

```bash
kubectl get deployment nginx-deploy
```

✅ Output:

```
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy    5/5     5            5           5m
```

The number of pods scaled up automatically 🚀

---

## ⚙️ **9️⃣ Scale Back Down**

When you stop the load test, CPU utilization drops.
After a few minutes, HPA automatically scales pods down.

Check again:

```bash
kubectl get hpa
kubectl get pods
```

✅ You’ll see replicas return to 1 or 2.

---

## 🧾 **10️⃣ Clean Up**

When done:

```bash
kubectl delete -f nginx-hpa.yaml
kubectl delete -f nginx-service.yaml
kubectl delete -f nginx-deployment.yaml
```

---

# 📘 **Summary**

| Step | Command / File                           | Description             |
| ---- | ---------------------------------------- | ----------------------- |
| 1    | `kubectl apply -f nginx-deployment.yaml` | Deploy nginx            |
| 2    | `kubectl apply -f nginx-service.yaml`    | Expose deployment       |
| 3    | `kubectl apply -f nginx-hpa.yaml`        | Create autoscaler       |
| 4    | `kubectl get hpa`                        | Check autoscaler status |
| 5    | `while true; do wget ...; done`          | Generate load           |
| 6    | `kubectl get hpa -w`                     | Watch scaling           |
| 7    | Stop load                                | Observe scale down      |

---

# 🧠 **Key Points**

✅ HPA requires `metrics-server`
✅ Uses CPU utilization % (default metric)
✅ Works with Deployments, ReplicaSets, StatefulSets
✅ Automatically adjusts replica count between min/max
✅ Great for cost optimization and handling variable traffic

---

