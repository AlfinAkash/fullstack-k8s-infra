#  Fullstack Kubernetes Deployment

Kubernetes manifests to deploy the [autodeploy-fullstack-ec2](https://github.com/AlfinAkash/autodeploy-fullstack-ec2) application on any Kubernetes cluster.

## Stack

| Layer | Technology | Image |
|---|---|---|
| Frontend | React + Vite | `alfinakash/fullstack-frontend:0.4` |
| Backend | Node.js + Express | `alfinakash/fullstack-backend:0.4` |
| Database | PostgreSQL 15 | `postgres:15` |

---

##  File Structure

```
k8s/
├── namespace.yaml               # Namespace: fullstack
├── configmap.yaml               # App config (DB host, ports, env vars)
├── secret.yaml                  # DB password + JWT secret (base64)
├── persistentvolumeclaim.yaml   # 5Gi storage for PostgreSQL data
├── statefulset.yaml             # PostgreSQL StatefulSet
├── postgres-service.yaml        # PostgreSQL ClusterIP service
├── backend-deployment.yaml      # Backend Deployment
├── backend-service.yaml         # Backend ClusterIP service
├── frontend-deployment.yaml     # Frontend Deployment
└── frontend-service.yaml        # Frontend LoadBalancer service
```

---

##  Architecture

```
Internet
    │
    ▼
[ frontend-service :80 ]   (LoadBalancer)
    │
[ frontend-deployment ]    (React/Vite - port 3000)
    │
[ backend-service :5000 ]  (ClusterIP)
    │
[ backend-deployment ]     (Node/Express - port 5000)
    │  initContainer waits for Postgres to be ready
    │
[ postgres-service :5432 ] (ClusterIP)
    │
[ statefulset ]            (PostgreSQL 15)
    │
[ persistentvolumeclaim ]  (5Gi persistent disk)
```

---

##  Prerequisites

- `kubectl` installed and configured
- A running Kubernetes cluster (Minikube / EKS / GKE / AKS)
- `kubectl get nodes` shows nodes in `Ready` state

---

##  Update Secrets Before Deploying

The `secret.yaml` contains base64-encoded credentials. Change them for production:

```bash
# Encode your own values
echo -n "your-db-password" | base64
echo -n "your-jwt-secret"  | base64
```

Paste the output into `secret.yaml` under `DB_PASSWORD` and `JWT_SECRET`.

---

##  Deploy

### Apply all files in order:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f persistentvolumeclaim.yaml
kubectl apply -f statefulset.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
```

### Or apply everything at once:

```bash
kubectl apply -f .
```

---

##  Verify

```bash
# Check all pods are Running
kubectl get pods -n fullstack

# Check all services
kubectl get svc -n fullstack

# Check all resources at once
kubectl get all -n fullstack
```

Expected output:
```
NAME                                     READY   STATUS    RESTARTS
pod/postgres-statefulset-0               1/1     Running   0
pod/backend-deployment-xxx               1/1     Running   0
pod/frontend-deployment-xxx              1/1     Running   0
```

---

##  Access the Application

### Cloud (EKS / GKE / AKS):
```bash
kubectl get svc frontend-service -n fullstack
# Use the EXTERNAL-IP → http://<EXTERNAL-IP>
```

### Minikube:
```bash
# Option 1 - Tunnel
minikube tunnel
kubectl get svc frontend-service -n fullstack

# Option 2 - Direct service open
minikube service frontend-service -n fullstack

# Option 3 - Port forward
kubectl port-forward svc/frontend-service 3000:80 -n fullstack
# Open: http://localhost:3000
```

---

##  Useful Commands

```bash
# View logs
kubectl logs -l app=backend  -n fullstack --tail=50
kubectl logs -l app=frontend -n fullstack --tail=50
kubectl logs -l app=postgres -n fullstack --tail=50

# Follow live logs
kubectl logs -f deployment/backend-deployment -n fullstack

# Shell into backend container
kubectl exec -it deployment/backend-deployment -n fullstack -- /bin/sh

# Connect to PostgreSQL
kubectl exec -it postgres-statefulset-0 -n fullstack -- psql -U admin -d mydb

# Restart a deployment
kubectl rollout restart deployment/backend-deployment  -n fullstack
kubectl rollout restart deployment/frontend-deployment -n fullstack
```

---

##  Teardown

```bash
# Delete all resources
kubectl delete namespace fullstack
```

>  This deletes everything including the PersistentVolumeClaim (database data).

---

##  Related

- Application source: [autodeploy-fullstack-ec2](https://github.com/AlfinAkash/autodeploy-fullstack-ec2)
- Docker Hub: [alfinakash](https://hub.docker.com/u/alfinakash)
