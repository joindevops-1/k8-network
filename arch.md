# Kubernetes Architecture

## High Level

```
┌─────────────────────────────────────────────────────┐
│                   Control Plane                      │
│  API Server  │  etcd  │  Scheduler  │  Controller   │
└─────────────────────────────────────────────────────┘
                        │
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   ┌────┴────┐     ┌────┴────┐     ┌────┴────┐
   │ Worker1 │     │ Worker2 │     │ Worker3 │
   │ kubelet │     │ kubelet │     │ kubelet │
   │ kube-   │     │ kube-   │     │ kube-   │
   │ proxy   │     │ proxy   │     │ proxy   │
   │ pods    │     │ pods    │     │ pods    │
   └─────────┘     └─────────┘     └─────────┘
```

---

## Control Plane Components

### 1. API Server
- **Entry point** for everything
- Every kubectl command hits API server
- Validates and processes all requests
- Only component that talks to etcd

```
kubectl apply -f pod.yaml
        │
        ▼
   API Server  ──► etcd (stores state)
        │
        ▼
   Scheduler, Controller get notified
```

### 2. etcd
- **Database** of the cluster
- Stores entire cluster state (pods, configs, secrets)
- Key-value store
- If etcd dies → cluster has no memory

```
etcd stores:
- all pod definitions
- all configmaps / secrets
- all node info
- all deployments, services
```

### 3. Scheduler
- Watches for **unscheduled pods**
- Decides **which node** a pod runs on
- Considers CPU, memory, taints, affinity, topology

```
New pod created (no node assigned)
        │
        ▼
   Scheduler sees it
        │
        ▼
   Checks all nodes
   - enough CPU? memory?
   - any taints blocking?
   - affinity rules?
        │
        ▼
   Assigns pod to best node
```

### 4. Controller Manager
- Runs all **controllers** in one process
- Constantly watches cluster state
- Makes actual state match desired state

```
Controllers inside it:
- Deployment controller   → maintains replica count
- Node controller         → detects node failures
- Service controller      → manages endpoints
- Job controller          → manages batch jobs
```

```
Desired: 3 replicas
Actual:  2 replicas (one crashed)
                │
                ▼
        Controller sees gap
                │
                ▼
        Creates new pod ✅
```

---

## Worker Node Components

### 1. kubelet
- **Agent** running on every node
- Talks to API server
- Ensures pods are running on that node
- Reports node health back to control plane

```
API Server: "run this pod on Node1"
        │
        ▼
   kubelet on Node1
        │
        ▼
   tells container runtime to start pod
        │
        ▼
   monitors pod, reports status back
```

### 2. kube-proxy
- Runs on every node
- Manages **iptables rules** for Services
- Routes traffic to correct pod IPs

```
request → ClusterIP:80
               │
          kube-proxy
          iptables rules
               │
          Pod IP:8080  ✅
```

### 3. Container Runtime
- Actually **runs the containers**
- Default in EKS: **containerd**
- kubelet tells it what to run

```
kubelet → containerd → container running
```

---

## How It All Works Together

```
kubectl apply -f deployment.yaml

Step 1: kubectl → API Server (validate & store in etcd)

Step 2: Deployment Controller sees new deployment
        → creates ReplicaSet
        → creates Pods (unscheduled)

Step 3: Scheduler sees unscheduled pods
        → picks best node for each pod
        → updates pod with node assignment

Step 4: kubelet on chosen node sees pod assigned to it
        → tells containerd to pull image & start container

Step 5: Pod starts running
        → kubelet reports status to API server
        → API server updates etcd
```

---

## In EKS Specifically

```
AWS manages:                You manage:
────────────────────        ────────────────────
API Server                  Worker Nodes
etcd                        kubelet
Scheduler                   Your pods/deployments
Controller Manager          Namespaces
Control Plane HA            Node scaling (or CA)
```

> In EKS you never see or touch the control plane — AWS runs it for you.
> You only interact with worker nodes and your workloads.

---

## Summary Table

| Component | Lives On | Purpose |
|---|---|---|
| API Server | Control Plane | Entry point, validates all requests |
| etcd | Control Plane | Cluster database |
| Scheduler | Control Plane | Assigns pods to nodes |
| Controller Manager | Control Plane | Maintains desired state |
| kubelet | Worker Node | Runs pods on the node |
| kube-proxy | Worker Node | Service traffic routing |
| containerd | Worker Node | Actually runs containers |