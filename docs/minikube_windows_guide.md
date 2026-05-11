# Minikube on Windows — Complete Setup & Deployment Guide

This guide explains every command needed to install Minikube on Windows (using D drive),
deploy your fullstack application, and access it in the browser.

---

## What is Minikube?

Minikube runs a real Kubernetes cluster on your local Windows machine inside Docker.
It creates a virtual node inside a Docker container so you can test Kubernetes
deployments without needing a cloud provider like AWS or GKE.

---

## Prerequisites

Before running any command, make sure:

- **Docker Desktop** is installed and running (Engine must show "Running")
- **PowerShell is opened as Administrator**
  (Search PowerShell → Right click → Run as Administrator)
- You are on **D drive** (all files stored there, C drive is not used)

---

## Phase 1 — Install Minikube on D Drive

### Step 1 — Create the minikube folder on D drive

```powershell
New-Item -Path 'D:\' -Name 'minikube' -ItemType Directory -Force
```

**What this does:**
Creates a new folder `D:\minikube` where the minikube binary will be stored.
`-Force` means it won't throw an error if the folder already exists.

---

### Step 2 — Download minikube.exe into D drive

```powershell
$ProgressPreference = 'SilentlyContinue'; Invoke-WebRequest -OutFile 'D:\minikube\minikube.exe' -Uri 'https://github.com/kubernetes/minikube/releases/latest/download/minikube-windows-amd64.exe' -UseBasicParsing
```

**What this does:**
Downloads the latest stable minikube Windows binary from GitHub and saves it
as `minikube.exe` inside `D:\minikube`.
`$ProgressPreference = 'SilentlyContinue'` hides the download progress bar
so it runs faster without flickering output.

---

### Step 3 — Add D:\minikube to system PATH

```powershell
$oldPath = [Environment]::GetEnvironmentVariable('Path', [EnvironmentVariableTarget]::Machine)
if ($oldPath.Split(';') -inotcontains 'D:\minikube'){ [Environment]::SetEnvironmentVariable('Path', $('{0};D:\minikube' -f $oldPath), [EnvironmentVariableTarget]::Machine) }
```

**What this does:**
Adds `D:\minikube` to your Windows system PATH so you can type `minikube`
from any folder in PowerShell without typing the full path `D:\minikube\minikube.exe`.
It first reads the existing PATH, checks if `D:\minikube` is already in it,
and only adds it if it's not already there.

> **Note:** Requires Administrator PowerShell. If you get "registry access not allowed",
> close and reopen PowerShell as Administrator.

---

### Step 4 — Set Minikube data storage to D drive

```powershell
[Environment]::SetEnvironmentVariable('MINIKUBE_HOME', 'D:\minikube', [EnvironmentVariableTarget]::Machine)
```

**What this does:**
Sets an environment variable `MINIKUBE_HOME` pointing to `D:\minikube`.
This tells Minikube to store all its data (cluster files, configs, cached images)
in `D:\minikube` instead of the default `C:\Users\<you>\.minikube`.
This is critical when your C drive is nearly full.

> **After running Step 3 and Step 4, close PowerShell completely and reopen as Administrator.**
> Environment variables only take effect in new terminal sessions.

---

## Phase 2 — Install kubectl

```powershell
winget install Kubernetes.kubectl
```

**What this does:**
Installs `kubectl` — the command-line tool used to communicate with your
Kubernetes cluster. Every `kubectl` command you run sends instructions to
Minikube's cluster (create pods, check status, view logs, etc).
`winget` is the Windows package manager built into Windows 10/11.

> **After this command, close and reopen PowerShell as Administrator again.**

---

## Phase 3 — Verify Installations

### Check Minikube version

```powershell
minikube version
```

**What this does:**
Prints the installed Minikube version. Confirms that `minikube.exe` is correctly
installed and the PATH is set up. Expected output: `minikube version: v1.x.x`

---

### Check kubectl version

```powershell
kubectl version --client
```

**What this does:**
Prints the installed kubectl version (client side only, no cluster needed).
Confirms kubectl is correctly installed. Expected output: `Client Version: v1.x.x`

---

## Phase 4 — Start Minikube Cluster

```powershell
minikube start --driver=docker
```

**What this does:**
Starts a single-node Kubernetes cluster inside Docker Desktop.
`--driver=docker` tells Minikube to use Docker as the virtualization layer
instead of VirtualBox or Hyper-V. Minikube will pull the Kubernetes base image
and start all control plane components (API server, scheduler, etcd).
This takes 2–5 minutes on first run.

Expected output at the end:
```
Done! kubectl is now configured to use "minikube" cluster
```

---

### Verify the cluster node is ready

```powershell
kubectl get nodes
```

**What this does:**
Lists all nodes in the cluster. Should show one node named `minikube` with
status `Ready`. If status is `NotReady`, wait 30 seconds and run again.

Expected output:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.x.x
```

---

## Phase 5 — Clone Repo and Deploy Application

### Go to D drive

```powershell
cd D:\
```

**What this does:**
Changes current directory to D drive root so all files are stored on D drive.

---

### Clone the Kubernetes manifests repo

```powershell
git clone https://github.com/AlfinAkash/fullstack-k8s-infra.git
```

**What this does:**
Downloads all your Kubernetes YAML files from GitHub into a new folder
`D:\fullstack-k8s-infra\k8s`. This includes all 10 manifest files
(namespace, configmap, secret, pvc, statefulset, services, deployments).

---

### Enter the repo folder

```powershell
cd D:\fullstack-k8s\k8s
```

**What this does:**
Moves into the cloned folder so the `kubectl apply -f` commands can find
the YAML files without needing to type the full path.

---

### Apply namespace

```powershell
kubectl apply -f namespace.yaml
```

**What this does:**
Creates the `fullstack` namespace in Kubernetes. A namespace is a virtual
boundary that isolates your app's resources from other apps on the same cluster.
This must be applied first — all other resources reference `namespace: fullstack`.

---

### Apply ConfigMap

```powershell
kubectl apply -f configmap.yaml
```

**What this does:**
Creates a ConfigMap named `app-configmap` containing non-sensitive environment
variables: `DB_HOST=postgres-service`, `DB_PORT=5432`, `DB_NAME=mydb`,
`DB_USER=admin`, `PORT=5000`. These are injected into the backend and frontend
containers at runtime.

---

### Apply Secret

```powershell
kubectl apply -f secret.yaml
```

**What this does:**
Creates a Secret named `app-secret` containing sensitive values:
`DB_PASSWORD` and `JWT_SECRET` encoded in base64.
Kubernetes stores these separately from ConfigMaps and restricts access to them.

---

### Apply PersistentVolumeClaim

```powershell
kubectl apply -f persistentvolumeclaim.yaml
```

**What this does:**
Requests 5GB of persistent disk storage from Minikube for PostgreSQL.
Without this, every time the database pod restarts, all your data is wiped.
The PVC acts like a virtual hard drive attached to the PostgreSQL container.

---

### Apply StatefulSet (PostgreSQL)

```powershell
kubectl apply -f statefulset.yaml
```

**What this does:**
Creates the PostgreSQL 15 database pod as a StatefulSet.
StatefulSet is used instead of Deployment because databases need a stable
pod name (`postgres-statefulset-0`) and stable persistent storage.
It reads credentials from the ConfigMap and Secret created earlier.

---

### Apply PostgreSQL Service

```powershell
kubectl apply -f postgres-service.yaml
```

**What this does:**
Creates a ClusterIP Service named `postgres-service` on port 5432.
This gives the PostgreSQL pod a stable DNS name inside the cluster.
The backend uses `DB_HOST=postgres-service` to connect — Kubernetes DNS
resolves this name to the PostgreSQL pod's internal IP automatically.

---

### Apply Backend Deployment

```powershell
kubectl apply -f backend-deployment.yaml
```

**What this does:**
Creates the Node.js/Express backend pod using image `alfinakash/fullstack-backend:0.4`.
Includes an `initContainer` that waits until `postgres-service:5432` is reachable
before starting the backend — this prevents connection errors on startup.
All environment variables (DB_HOST, DB_USER, DB_PASSWORD, JWT_SECRET etc.)
are injected from the ConfigMap and Secret.

---

### Apply Backend Service

```powershell
kubectl apply -f backend-service.yaml
```

**What this does:**
Creates a ClusterIP Service named `backend-service` on port 5000.
The frontend uses this service name to make API calls to the backend.
ClusterIP means it's only accessible inside the cluster — not from the internet.

---

### Apply Frontend Deployment

```powershell
kubectl apply -f frontend-deployment.yaml
```

**What this does:**
Creates the React/Vite frontend pod using image `alfinakash/fullstack-frontend:0.4`
on port 3000. Injects `VITE_BACKEND_URL=http://backend-service:5000` so the
frontend knows where to send API calls.

---

### Apply Frontend Service

```powershell
kubectl apply -f frontend-service.yaml
```

**What this does:**
Creates a NodePort Service named `frontend-service` that maps external port 80
to container port 3000. This makes the frontend accessible from outside the cluster.
NodePort is used instead of LoadBalancer because Minikube doesn't support
cloud LoadBalancers natively.

---

## Phase 6 — Watch Pods Start Up

```powershell
kubectl get pods -n fullstack -w
```

**What this does:**
Lists all pods inside the `fullstack` namespace and watches for status changes
in real time. `-n fullstack` means look in the `fullstack` namespace.
`-w` means "watch" — it keeps refreshing automatically as pod statuses change.

You will see pods progress through these stages:
```
Pending       → Kubernetes is scheduling the pod onto a node
Init:0/1      → initContainer is running (backend waiting for postgres)
Running 0/1   → Container started but not ready yet
Running 1/1   → Pod is fully ready ✅
```

Wait until all 3 pods show `1/1 Running`:
```
NAME                                    READY   STATUS
postgres-statefulset-0                  1/1     Running
backend-deployment-xxxxxxxxx            1/1     Running
frontend-deployment-xxxxxxxxx           1/1     Running
```

Press `Ctrl+C` to stop watching once all pods are Running.

---

## Phase 7 — Open the Application

```powershell
minikube service frontend-service -n fullstack
```

**What this does:**
Finds the NodePort assigned to `frontend-service`, maps it to a localhost URL,
and automatically opens your default browser to that URL.
This is the Minikube way to access a service — it handles the internal IP
and port mapping for you. Your app will open at something like `http://127.0.0.1:PORT`.

---

## Quick Reference — Other Useful Commands

| Command | What it does |
|---|---|
| `kubectl get all -n fullstack` | Show all pods, services, deployments at once |
| `kubectl get svc -n fullstack` | Show all services and their ports |
| `kubectl get pvc -n fullstack` | Check if disk storage is bound |
| `kubectl logs -l app=backend -n fullstack` | View backend logs |
| `kubectl logs -l app=frontend -n fullstack` | View frontend logs |
| `kubectl logs -l app=postgres -n fullstack` | View PostgreSQL logs |
| `kubectl describe pod <name> -n fullstack` | Detailed info + events for a pod |
| `minikube stop` | Stop the cluster (saves state) |
| `minikube start --driver=docker` | Start the cluster again after stopping |
| `minikube dashboard` | Open Kubernetes web dashboard in browser |
| `kubectl delete namespace fullstack` | Delete everything and start fresh |

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `registry access not allowed` | Not running as Administrator | Right click PowerShell → Run as Administrator |
| PVC stuck in `Pending` | Missing storageClassName | Add `storageClassName: standard` to pvc yaml |
| Backend stuck at `Init:0/1` | Waiting for Postgres to start | Wait 1-2 minutes, Postgres is still starting |
| `minikube start` fails | Docker Desktop not running | Open Docker Desktop and wait for engine to start |
| `kubectl` not found | PATH not set or terminal not restarted | Close and reopen PowerShell after install |
| App doesn't open | Frontend service not ready | Check all pods are `1/1 Running` first |

---

## Architecture Summary

```
Your Browser
     │
     ▼
minikube service (NodePort :80)
     │
     ▼
frontend-deployment (React/Vite :3000)
     │  VITE_BACKEND_URL
     ▼
backend-service (ClusterIP :5000)
     │
     ▼
backend-deployment (Node/Express :5000)
     │  DB_HOST=postgres-service
     ▼
postgres-service (ClusterIP :5432)
     │
     ▼
postgres-statefulset (PostgreSQL 15)
     │
     ▼
PersistentVolumeClaim (5Gi disk on D:\minikube)
```

---

## Related Repos

- Application source code: https://github.com/AlfinAkash/autodeploy-fullstack-ec2
- Kubernetes manifests: https://github.com/AlfinAkash/fullstack-k8s-infra/tree/main/k8s
