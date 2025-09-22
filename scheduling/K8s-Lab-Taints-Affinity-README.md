
# 🧪 Kubernetes Lab: Taints, Tolerations, and Node Affinity

This lab demonstrates **Taints, Tolerations, and Affinity/Anti-Affinity** using a local Kubernetes cluster created with **k3d** (1 control-plane + 3 workers).

---

## ⚙️ Prerequisites
- A running k3d cluster (e.g. `k3d cluster create dev --agents 3 --wait`)
- kubectl installed and configured (`kubectl config use-context k3d-dev`)

Check your nodes:
```bash
kubectl get nodes -L zone,region,performance
```

Expected (after labeling in step 1):
```
NAME                STATUS   ROLES                  AGE   VERSION   ZONE   REGION   PERFORMANCE
k3d-dev-server-0    Ready    control-plane,master   10m   v1.31.x
k3d-dev-agent-0     Ready    <none>                 10m   v1.31.x   scu    iberia   high
k3d-dev-agent-1     Ready    <none>                 10m   v1.31.x   bcn    iberia   high
k3d-dev-agent-2     Ready    <none>                 10m   v1.31.x   sab    iberia   standard
```

---

## 1️⃣ Label Nodes
```bash
kubectl label nodes k3d-dev-agent-0 zone=scu region=iberia performance=high --overwrite
kubectl label nodes k3d-dev-agent-1 zone=bcn region=iberia performance=high --overwrite
kubectl label nodes k3d-dev-agent-2 zone=sab region=iberia performance=standard --overwrite
```

---

## 2️⃣ Apply a Taint
Reserve **agent-0** for dedicated DB workloads:
```bash
kubectl taint nodes k3d-dev-agent-0 dedicated=db:NoSchedule
```

Check:
```bash
kubectl describe node k3d-dev-agent-0 | grep -i taints -A2
```

Expected:
```
Taints: dedicated=db:NoSchedule
```

---

## 3️⃣ Deployment Without Toleration
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-no-toleration
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
      - name: nginx
        image: nginx
```
Apply:
```bash
kubectl apply -f 01-web-no-toleration.yaml
kubectl get pods -l app=web -o wide
```

Expected → Pods **avoid agent-0**.

---

## 4️⃣ Deployment With Toleration + Node Affinity (Required)
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "db"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: performance
          operator: In
          values: ["high"]
```
Apply:
```bash
kubectl apply -f 02-db-with-toleration-and-affinity.yaml
kubectl get pods -l app=db -o wide
```

Expected → Pods scheduled on **agent-0** or **agent-1** (both `performance=high`).

---

## 5️⃣ Preferred Affinity With Weight
Prefer **Barcelona (zone=bcn)**, but not mandatory:
```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 90
  preference:
    matchExpressions:
    - key: zone
      operator: In
      values: ["bcn"]
```
Apply:
```bash
kubectl apply -f 03-api-preferred-zone.yaml
kubectl get pods -l app=api -o wide
```

Expected → Pods prefer **agent-1 (zone=bcn)**, but may fall back to others.

---

## 6️⃣ NoExecute Taint (Eviction)
1. Create a pod pinned to **agent-2**:
```yaml
nodeSelector:
  zone: sab
```
2. Apply taint:
```bash
kubectl taint nodes k3d-dev-agent-2 quarantine=true:NoExecute
```
Expected → Pod is **evicted immediately**.

3. With toleration and `tolerationSeconds`:
```yaml
tolerations:
- key: "quarantine"
  operator: "Equal"
  value: "true"
  effect: "NoExecute"
  tolerationSeconds: 60
```
Expected → Pod stays up to 60s before eviction.

---

## 7️⃣ Cleanup
```bash
kubectl delete deployment web-no-toleration db-with-toleration api-preferred-bcn
kubectl delete pod throwaway throwaway-grace --ignore-not-found
kubectl taint nodes k3d-dev-agent-0 dedicated- || true
kubectl taint nodes k3d-dev-agent-2 quarantine- || true
```

---

## ✅ What You Learned
- **Taints** (on nodes) repel pods.  
- **Tolerations** (on pods) allow exceptions.  
- **Node Affinity** controls pod placement based on node labels.  
- **Required** = hard rule, **Preferred** = soft rule with `weight`.  
- **NoExecute** = can evict existing pods.  
