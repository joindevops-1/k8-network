# Kubernetes NetworkPolicy in EKS — Roboshop Example

## Overview

By default, **any pod in the `default` namespace can reach any pod in the `roboshop` namespace** — Kubernetes has no restrictions until you apply a NetworkPolicy.

---

## Prerequisites — Enable NetworkPolicy Enforcement in EKS

NetworkPolicy resources are **ignored** unless your CNI enforces it. Enable it on VPC CNI (easiest for EKS):

```bash
# Enable NetworkPolicy on VPC CNI v1.14+
kubectl set env daemonset aws-node -n kube-system ENABLE_NETWORK_POLICY=true

# Verify — a new agent pod should appear on every node
kubectl get pods -n kube-system | grep network-policy

# Expected output:
# aws-network-policy-agent-xxxxx   1/1   Running   0   2m
# aws-network-policy-agent-yyyyy   1/1   Running   0   2m
```

---

## Step 1 — Create Namespace

```bash
kubectl create namespace roboshop
```

---

## Step 2 — Pod Manifests

### default-nginx.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-default
  namespace: default
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

### roboshop-nginx.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-roboshop
  namespace: roboshop
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx
```

### Apply pods

```bash
kubectl apply -f default-nginx.yaml
kubectl apply -f roboshop-nginx.yaml

# Verify
kubectl get pods -n default
kubectl get pods -n roboshop
```

---

## Step 3 — Get Pod IP (No Service Needed)

In EKS, VPC CNI assigns a **real VPC IP** to every pod — so you can curl pod IPs directly without a Service.

```bash
# Get roboshop nginx pod IP
kubectl get pod nginx-roboshop -n roboshop -o wide

# Expected output:
# NAME             READY   STATUS    IP            NODE
# nginx-roboshop   1/1     Running   10.0.1.45     ip-10-0-1-45
```

---

## Step 4 — TEST BEFORE Network Policy

```bash
# One-liner — exec into default pod and curl roboshop pod IP directly
kubectl exec nginx-default -n default -- curl --max-time 5 http://<ROBOSHOP_POD_IP>

# Example:
kubectl exec nginx-default -n default -- curl --max-time 5 http://10.0.1.45

# Expected Result:
# ✅ 200 OK — nginx welcome page returned
# Traffic flows freely — no policy applied yet
```

---

## Step 5 — Network Policy Manifests

### deny-all-ingress.yaml

Blocks **all incoming traffic** to every pod in the `roboshop` namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: roboshop
spec:
  podSelector: {}        # applies to ALL pods in roboshop
  policyTypes:
    - Ingress
  # no ingress rules = deny everything
```

### allow-same-namespace.yaml

Allows pods **within roboshop namespace** to talk to each other.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: roboshop
spec:
  podSelector: {}        # all pods in roboshop
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}  # only pods in same namespace (roboshop)
```

### Apply Network Policies

```bash
kubectl apply -f deny-all-ingress.yaml
kubectl apply -f allow-same-namespace.yaml

# Verify
kubectl get networkpolicy -n roboshop

# Expected output:
# NAME                   POD-SELECTOR   AGE
# deny-all-ingress       <none>         5s
# allow-same-namespace   <none>         5s
```

---

## Step 6 — TEST AFTER Network Policy

### From default namespace (should be BLOCKED)

```bash
kubectl exec nginx-default -n default -- curl --max-time 5 http://<ROBOSHOP_POD_IP>

# Expected Result:
# ❌ curl: (28) Connection timed out after 5000 milliseconds
# Traffic from default namespace is now BLOCKED ✅
```

### From roboshop namespace (should be ALLOWED)

```bash
kubectl exec nginx-roboshop -n roboshop -- curl --max-time 5 http://<ROBOSHOP_POD_IP>

# Expected Result:
# ✅ 200 OK — nginx welcome page returned
# Traffic within roboshop namespace is ALLOWED ✅
```

---

## Summary

```
BEFORE Network Policy
──────────────────────────────────────────────────────
default/nginx-default   ────────────► roboshop/nginx-roboshop   ✅ allowed
roboshop/nginx-roboshop ────────────► roboshop/nginx-roboshop   ✅ allowed


AFTER Network Policy
──────────────────────────────────────────────────────
default/nginx-default   ────────────► roboshop/nginx-roboshop   ❌ blocked
roboshop/nginx-roboshop ────────────► roboshop/nginx-roboshop   ✅ allowed
```

| From | To | Result |
|---|---|---|
| `default` namespace pod | `roboshop` pod | ❌ Blocked |
| `roboshop` pod | `roboshop` pod | ✅ Allowed |
| `kube-system` pod | `roboshop` pod | ❌ Blocked |

---

## Why Pod IP Works Without a Service

Because EKS uses **VPC CNI** — every pod gets a real VPC IP, so pod IPs are directly routable without any Service in between. NetworkPolicy enforces at the **pod IP level**, so blocking works the same whether you use a Service or direct Pod IP.

```
default/nginx-default  ──► 10.0.1.45 (real VPC IP) ──► roboshop/nginx-roboshop
                                  no service needed!
```

---

## Cleanup

```bash
kubectl delete pod nginx-default -n default
kubectl delete pod nginx-roboshop -n roboshop
kubectl delete networkpolicy deny-all-ingress -n roboshop
kubectl delete networkpolicy allow-same-namespace -n roboshop
```