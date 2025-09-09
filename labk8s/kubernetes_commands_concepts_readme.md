# Kubernetes Commands & Concepts – Handy README

A concise, practical reference to core Kubernetes concepts **and** the commands you’ll use every day. Examples assume a *nix shell and `kubectl >= 1.26`.

---

## Table of Contents
1. What Kubernetes Is (in 30 seconds)
2. Cluster, Contexts, and Namespaces
3. Pods
4. ReplicaSets
5. Deployments
6. Services & Ingress (quick primer)
7. ConfigMaps & Secrets (+ base64 encoding)
8. Probes (liveness, readiness, startup)
9. Scheduling (labels, selectors, taints/tolerations, affinity)
10. Resource Requests/Limits & Autoscaling (HPA)
11. Jobs & CronJobs
12. Storage: PVs & PVCs
13. Observability & Troubleshooting
14. One‑liners & Power Tips

---

## 1) What Kubernetes Is (in 30 seconds)
- **Kubernetes** orchestrates containers across a cluster of machines (nodes).
- You declare **desired state** in YAML (e.g., “3 replicas of this app”), and the **control plane** constantly converges reality to match.
- Core control plane pieces: **API Server**, **Scheduler**, **Controller Manager**, **etcd** (backing store). Nodes run **kubelet** and a container runtime (e.g., containerd).

---

## 2) Cluster, Contexts, and Namespaces
```bash
# Show cluster info & versions
kubectl cluster-info
kubectl version --short

# View/choose context (which cluster/user/namespace kubectl targets)
kubectl config get-contexts
kubectl config use-context <context-name>

# Current namespace, list and switch
kubectl config view --minify -o jsonpath='{..namespace}'
kubectl get ns
kubectl create ns demo
kubectl config set-context --current --namespace=demo
```

---

## 3) Pods
**Pod** = smallest deployable unit. Usually one container per Pod (sometimes sidecars).

**Minimal Pod YAML:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
spec:
  containers:
    - name: hello
      image: nginx:1.25
      ports:
        - containerPort: 80
```

**Commands:**
```bash
# Create from file / dry-run
kubectl apply -f pod.yaml
kubectl create -f pod.yaml --dry-run=client -o yaml

# List/get/describe
kubectl get pods -o wide
kubectl get pod hello-pod -o yaml
kubectl describe pod hello-pod

# Logs & exec
kubectl logs hello-pod
kubectl logs -f hello-pod
kubectl exec -it hello-pod -- sh

# Port-forward to your laptop
kubectl port-forward pod/hello-pod 8080:80

# Delete
kubectl delete pod hello-pod
```

---

## 4) ReplicaSets
**ReplicaSet** ensures a **fixed number of identical Pods**.
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: hello-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: web
          image: nginx:1.25
```
**Commands:**
```bash
kubectl apply -f rs.yaml
kubectl get rs
kubectl scale rs/hello-rs --replicas=5
kubectl describe rs hello-rs
kubectl delete rs hello-rs
```

> Note: You’ll rarely create ReplicaSets directly—**Deployments** manage them for you.

---

## 5) Deployments
**Deployment** manages stateless apps via **rolling updates** to ReplicaSets.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
```
**Commands:**
```bash
kubectl apply -f deploy.yaml
kubectl get deploy
kubectl describe deploy hello-deploy

# Scale
kubectl scale deploy/hello-deploy --replicas=5

# Rolling update (change the image)
kubectl set image deploy/hello-deploy web=nginx:1.26

# Observe rollout / history / undo
kubectl rollout status deploy/hello-deploy
kubectl rollout history deploy/hello-deploy
kubectl rollout undo deploy/hello-deploy --to-revision=1
```

---

## 6) Services & Ingress (quick primer)
- **Service** = stable virtual IP/DNS for reaching Pods. Types: `ClusterIP` (default, internal), `NodePort`, `LoadBalancer`.
- **Ingress** = HTTP(S) routing at Layer 7 via an Ingress Controller.

```yaml
# ClusterIP Service for our Deployment
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
```
```bash
kubectl apply -f svc.yaml
kubectl get svc
kubectl describe svc hello-svc

# Port-forward the Service to access it locally
kubectl port-forward svc/hello-svc 8080:80

# With namespace
kubectl -n demo port-forward svc/hello-svc 8080:80
```

---

## 7) ConfigMaps & Secrets (+ base64 encoding)
**ConfigMap** stores non‑sensitive config. **Secret** stores sensitive data (base64‑encoded—**not** encrypted by default!).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: "blue"
```
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:          # kubectl encodes these automatically
  DB_USER: admin
  DB_PASS: S3cr3t!
```

**Create via CLI:**
```bash
# ConfigMap from literals / file
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue
kubectl create configmap app-config-file \
  --from-file=app.properties

# Generic secret from literals / file
kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS='S3cr3t!'

# TLS secret (cert+key)
kubectl create secret tls my-tls \
  --cert=server.crt --key=server.key

# Docker registry secret (image pulls)
kubectl create secret docker-registry regcred \
  --docker-server=index.docker.io \
  --docker-username=<user> \
  --docker-password='<token>' \
  --docker-email=<you@example.com>
```

**Encode/Decode manually with base64:**
```bash
# Encode a value (Linux/macOS)
echo -n 'S3cr3t!' | base64  # -> UzNjcjN0IQ==

# Decode
echo -n 'UzNjcjN0IQ==' | base64 --decode
```

**Use in a Pod/Deployment:**
```yaml
spec:
  containers:
    - name: app
      image: busybox
      env:
        - name: APP_COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_COLOR
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASS
```

**Inspect (note: values in `data` are base64):**
```bash
kubectl get cm app-config -o yaml
kubectl get secret app-secret -o yaml

# Safely decode a single key
kubectl get secret app-secret -o jsonpath='{.data.DB_PASS}' | base64 --decode; echo
```

> **Security tips:** enable at-rest encryption for Secrets, restrict RBAC, consider external secret managers (e.g., External Secrets Operator + cloud KMS).

---

## 8) Probes: liveness, readiness, startup
```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
startupProbe:
  httpGet: { path: /start, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5
```

---

## 9) Scheduling: labels, selectors, taints/tolerations, affinity
```bash
# Label nodes/pods and select on them
kubectl label node <node> role=api
kubectl get nodes --show-labels
```
```yaml
# Tolerate a taint and prefer specific nodes
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: api
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: role
                operator: In
                values: ["api"]
```
```bash
# Manage taints on a node
kubectl taint nodes <node> dedicated=api:NoSchedule
kubectl taint nodes <node> dedicated-      # remove
```

---

## 10) Resources & Autoscaling (HPA)
```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```
```bash
# Horizontal Pod Autoscaler (CPU-based example)
kubectl autoscale deploy/hello-deploy --min=2 --max=10 --cpu-percent=60
kubectl get hpa
```

---

## 11) Jobs & CronJobs
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: once-off
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: task
          image: busybox
          command: ["/bin/sh","-c","echo hello; sleep 5"]
```
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly
spec:
  schedule: "0 3 * * *"   # 03:00 daily
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: task
              image: busybox
              command: ["/bin/sh","-c","date"]
```
```bash
kubectl apply -f job.yaml
kubectl get jobs,pods
kubectl logs job/once-off
kubectl apply -f cronjob.yaml
kubectl get cronjobs
```

---

## 12) Storage: PVs & PVCs (very quick)
```yaml
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
```
```yaml
# Mount in a Pod/Deployment
volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data-pvc
containers:
  - name: app
    volumeMounts:
      - name: data
        mountPath: /var/lib/data
```
```bash
kubectl apply -f pvc.yaml
kubectl get pvc,pv
```

---

## 13) Observability & Troubleshooting
```bash
# What’s running?
kubectl get all -A

# Events & describe are your best friends
kubectl get events --sort-by=.lastTimestamp
kubectl describe pod <pod>

# Logs (single/multi-container)
kubectl logs <pod>                   # default container
kubectl logs <pod> -c <container>
kubectl logs -f <pod>

# Exec a shell inside a container
kubectl exec -it <pod> -- sh         # or bash

# Top resources (requires metrics-server)
kubectl top nodes
kubectl top pods -A

# YAML live view & diff\ nkubectl get deploy <name> -o yaml
kubectl diff -f deploy.yaml

# Rollouts\ nkubectl rollout status deploy/<name>
kubectl rollout history deploy/<name>

# Delete stuck resources forcefully (last resort)
kubectl delete pod <pod> --grace-period=0 --force
```

---

## 14) One‑liners & Power Tips
```bash
# Switch namespace temporarily (-n) vs permanently (context)
kubectl -n kube-system get pods

# JSONPath extraction
kubectl get svc hello-svc -o jsonpath='{.spec.clusterIP}'

# Wide output & labels
kubectl get pods -o wide --show-labels

# Apply recursively and prune extras in the namespace
kubectl apply -k ./kustomize/overlays/prod
kubectl apply -f manifests/ --prune -l app=hello

# Create from stdin
cat <<'YAML' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata: { name: scratch }
YAML

# Quick busybox debug pod
kubectl run toolbox --rm -it --image=busybox -- /bin/sh

# Context/namespace helpers (kubectl plugins like kubens/kubectx are great)
# https://github.com/ahmetb/kubectx

# Port-forward a Service (not just Pod)
kubectl port-forward svc/hello-svc 8080:80

# Label/annotate resources
kubectl label deploy/hello-deploy tier=frontend --overwrite
kubectl annotate pod/<pod> note='investigate timeouts'

# Export resource to editable YAML
kubectl get deploy hello-deploy -o yaml > out.yaml
```

---

### Appendix: Common `kubectl` Verbs & Patterns
```
get | describe | logs | exec | create | apply | replace | delete | scale | set image | rollout | top
```

- **Declarative** > Imperative: keep YAML in git (`kubectl apply -f`).
- Prefer **Deployments** for stateless apps; **StatefulSets** for stateful.
- Use **RBAC** and least privilege.
- Protect **Secrets** (encryption at rest, external managers, restricted RBAC).
- Automate with **Kustomize** or **Helm**.

---

> **Cheat‑sheet mindset:** when stuck, `kubectl describe`, `kubectl get events`, and container `logs` usually reveal the issue. Keep your manifests small, labeled, and versioned.

