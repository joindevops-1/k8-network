# PodDisruptionBudget (PDB) — Complete Guide

## What is PDB?

PDB is a policy that **limits how many pods can go down at the same time** during voluntary disruptions.

### Voluntary vs Involuntary Disruptions

```
Voluntary (PDB protects)          Involuntary (PDB cannot help)
─────────────────────────         ─────────────────────────────
kubectl drain node                Node hardware failure
kubectl delete pod                OOM kill
Node upgrade                      Kernel panic
Cluster autoscaler scale down     Network partition
Rolling deployment
```

---

## How PDB Works

```
kubectl drain node  →  sends Eviction API request for each pod
                                    │
                                    ▼
                            API Server checks PDB
                                    │
                          ┌─────────┴──────────┐
                          │                    │
                     Budget OK            Budget violated
                          │                    │
                     Evict pod            Return 429 (Too Many Requests)
                                               │
                                          kubectl drain
                                          retries after
                                          a few seconds
```

---

## Manifest

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: catalogue-pdb
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: catalogue

  # Option 1 — minimum pods that must be available
  minAvailable: 2          # at least 2 pods always running

  # Option 2 — maximum pods that can go down
  # maxUnavailable: 1      # at most 1 pod down at a time
```

> Use either `minAvailable` OR `maxUnavailable` — not both.

---

## Check PDB Status

```bash
kubectl get pdb -n roboshop

# Output:
# NAME            MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# catalogue-pdb   2               N/A               1                     5m

# ALLOWED DISRUPTIONS = how many pods can be taken down RIGHT NOW
```

---

## Scenario: 4 Pods, minAvailable 25%

```
25% of 4 = 1 pod must always be available
max 3 pods can go down at a time
ALLOWED DISRUPTIONS: 3
```

### Drain Node1
```
pod1 evicted → 3 remaining → 3 >= 1 ✅
pod1 rescheduled on another node
```

### Drain Node2
```
pod2 evicted → 3 remaining → 3 >= 1 ✅
pod2 rescheduled on another node
```

### Drain Node3
```
pod3 evicted → 1 remaining → 1 >= 1 ✅ (exactly at limit)
pod3 rescheduled on Node4
```

### Drain Node4 — PROBLEM!
```
all pods now on Node4
evicting any pod → 0 remaining → 0 >= 1 ❌ VIOLATION

API returns 429 → drain BLOCKS forever
No nodes left to reschedule pods ❌
```

---

## Rounding — How Percentages Calculate

Kubernetes uses **floor()** — always drops the decimal, never rounds up.

```
floor(0.25) = 0
floor(0.50) = 0
floor(0.75) = 0
floor(1.00) = 1
floor(1.25) = 1
floor(1.99) = 1
```

### maxUnavailable: 25% across replica counts

```
1 replica  → 25% of 1 = 0.25 → floor → 0  ❌ drain BLOCKED
2 replicas → 25% of 2 = 0.50 → floor → 0  ❌ drain BLOCKED
3 replicas → 25% of 3 = 0.75 → floor → 0  ❌ drain BLOCKED
4 replicas → 25% of 4 = 1.00 → floor → 1  ✅ drain ALLOWED
```

### minAvailable: 50% across replica counts

```
1 replica  → 50% of 1 = 0.50 → floor → 0  ❌ useless (0 pods guaranteed)
2 replicas → 50% of 2 = 1.00 → floor → 1  ✅ drain ALLOWED
```

### Mental Trick — Minimum Replicas Needed

```
100 / percentage = minimum replicas for PDB to work

25%  → 100/25 = 4 replicas minimum
50%  → 100/50 = 2 replicas minimum
33%  → 100/33 = 3 replicas minimum
10%  → 100/10 = 10 replicas minimum
```

---

## PDB + HPA Together

### Problem with absolute numbers

```
HPA scales down: 10 pods → 3 pods
PDB says minAvailable: 8

PDB blocks ALL drains ❌
No node can ever be drained
```

### Solution — always use percentage with HPA

```yaml
spec:
  minAvailable: "50%"    # adjusts automatically with replica count
```

```
HPA at 10 pods → minAvailable = 5 pods
HPA at 4 pods  → minAvailable = 2 pods
HPA at 2 pods  → minAvailable = 1 pod

PDB automatically adjusts ✅
```

### Safe HPA + PDB pattern

```yaml
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalogue-hpa
  namespace: roboshop
spec:
  minReplicas: 4          # never go below 4 (satisfies 25% rounding)
  maxReplicas: 20
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalogue

---
# PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: catalogue-pdb
  namespace: roboshop
spec:
  selector:
    matchLabels:
      app: catalogue
  minAvailable: "50%"
```

---

## Edge Case: 1 Replica + maxUnavailable 25%

```
25% of 1 = 0.25 → floor → 0
maxUnavailable = 0 → 0 pods can go down
drain BLOCKED forever ❌
```

### Options to fix

```bash
# Option 1 — scale up before drain
kubectl scale deployment catalogue -n roboshop --replicas=4
kubectl drain node1 --ignore-daemonsets

# Option 2 — patch PDB temporarily
kubectl patch pdb catalogue-pdb -n roboshop \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/minAvailable","value":0}]'

kubectl drain node1 --ignore-daemonsets

kubectl patch pdb catalogue-pdb -n roboshop \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/minAvailable","value":"25%"}]'

# Option 3 — force drain (dangerous, bypasses PDB)
kubectl drain node1 --ignore-daemonsets --disable-eviction
```

---

## Golden Rules

```
1. Always use percentage with HPA — never absolute numbers

2. HPA minReplicas >= 4   when using maxUnavailable: 25%
   HPA minReplicas >= 2   when using minAvailable: 50%

3. Single replica apps will always have downtime during drain
   — PDB cannot help when there is only 1 pod

4. When draining ALL nodes — last node always gets stuck
   — use Cluster Autoscaler or add node manually

5. 100 / percentage = minimum replicas needed for PDB to work
```

---

## Summary Table

| Config | Replicas | Allowed Disruptions | Drain Works? |
|---|---|---|---|
| `maxUnavailable: 25%` | 1 | 0 | ❌ Blocked |
| `maxUnavailable: 25%` | 2 | 0 | ❌ Blocked |
| `maxUnavailable: 25%` | 3 | 0 | ❌ Blocked |
| `maxUnavailable: 25%` | 4 | 1 | ✅ Works |
| `minAvailable: 50%` | 1 | 0 | ❌ Blocked |
| `minAvailable: 50%` | 2 | 1 | ✅ Works |
| `minAvailable: 2` (absolute) | 4 | 2 | ✅ Works |
| `minAvailable: 8` (absolute) + HPA | 3 | 0 | ❌ Blocked |