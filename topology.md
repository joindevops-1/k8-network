# Topology Spread Constraint — Complete Guide

## What is it?

Controls **how pods are spread across your cluster** — across nodes, availability zones, regions.

---

## Does PDB Work Without Topology Spread?

**YES — PDB works completely independent of Topology Spread.**

They solve different problems:

```
PDB                                 Topology Spread
────────────────────────────────    ────────────────────────────────
How MANY pods are running?          WHERE are pods running?
Protects during drain/disruption    Protects during zone failure
Works at eviction time              Works at scheduling time
```

### Example — PDB without Topology Spread

```
All 4 pods in Zone A (no topology spread):

Zone A                    Zone B
┌──────────────────┐      ┌──────────────────┐
│ pod1 pod2        │      │                  │
│ pod3 pod4        │      │                  │
└──────────────────┘      └──────────────────┘

kubectl drain node in Zone A:
PDB minAvailable 50% = 2 pods must stay
→ drain waits, respects PDB ✅
→ PDB works fine!

BUT Zone A goes down (hardware failure):
→ all 4 pods gone ❌
→ PDB cannot help (involuntary disruption)
```

### Example — Both Together

```
2 pods in Zone A, 2 pods in Zone B (topology spread applied):

Zone A                    Zone B
┌──────────────────┐      ┌──────────────────┐
│ pod1 pod2        │      │ pod3 pod4        │
└──────────────────┘      └──────────────────┘

kubectl drain node in Zone A:
→ PDB protects — waits, reschedules ✅

Zone A goes down (hardware failure):
→ pod3, pod4 still running in Zone B ✅
→ Topology Spread saved you!
```

---

## Problem Without Topology Spread

```
You have 4 pods, 2 availability zones:

Zone A (ap-south-1a)        Zone B (ap-south-1b)
┌─────────────────┐         ┌─────────────────┐
│  pod1           │         │                 │
│  pod2           │         │                 │
│  pod3           │         │                 │
│  pod4           │         │                 │
└─────────────────┘         └─────────────────┘

Zone A goes down → ❌ all 4 pods gone = full outage!
```

---

## With Topology Spread

```
Zone A (ap-south-1a)        Zone B (ap-south-1b)
┌─────────────────┐         ┌─────────────────┐
│  pod1           │         │  pod3           │
│  pod2           │         │  pod4           │
└─────────────────┘         └─────────────────┘

Zone A goes down → ✅ pod3, pod4 still running in Zone B
```

---

## Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogue
  namespace: roboshop
spec:
  replicas: 4
  template:
    spec:
      topologySpreadConstraints:

        # Spread across availability zones
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: catalogue

        # Spread across nodes
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: catalogue
```

---

## Key Fields

### `maxSkew`
Maximum **difference** in pod count between zones/nodes:

```
maxSkew: 1

Zone A: 2 pods
Zone B: 1 pod
Difference: 1 ✅ allowed

Zone A: 3 pods
Zone B: 1 pod
Difference: 2 ❌ scheduler blocks — won't place more pods in Zone A
```

### `topologyKey`
Which node label to group by:

```
topology.kubernetes.io/zone      → spread across AZs
kubernetes.io/hostname           → spread across nodes
topology.kubernetes.io/region    → spread across regions
```

### `whenUnsatisfiable`

```
DoNotSchedule   → pod stays Pending if constraint cannot be met (strict)
ScheduleAnyway  → schedule anyway, best effort spreading (soft)
```

---

## How Scheduling Works Step by Step

```
4 pods, 2 zones, maxSkew: 1

Pod1 scheduled → Zone A
Zone A: 1   Zone B: 0   diff: 1 ✅

Pod2 scheduled → Zone B (balances it)
Zone A: 1   Zone B: 1   diff: 0 ✅

Pod3 scheduled → Zone A
Zone A: 2   Zone B: 1   diff: 1 ✅

Pod4 scheduled → Zone B (must go here to stay within maxSkew)
Zone A: 2   Zone B: 2   diff: 0 ✅

Result: perfectly balanced!
```

---

## PDB + Topology Spread Together

```
Topology Spread  →  ensures pods are in different zones  (scheduling time)
PDB              →  ensures minimum pods always running   (eviction time)

Together         →  zone failure safe + maintenance safe ✅✅
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogue
  namespace: roboshop
spec:
  replicas: 4
  template:
    spec:
      # Spread across zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: catalogue

---
# Protect during drain
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

## Feature Comparison

| | PDB | Topology Spread |
|---|---|---|
| Purpose | Limit pods going down | Spread pods across zones/nodes |
| Protects against | Node drains, upgrades | Zone failures, node failures |
| When it acts | During eviction | During pod scheduling |
| Works independently | ✅ Yes | ✅ Yes |
| Better together | ✅ Yes | ✅ Yes |

---

## What Each Protects Against

```
Scenario                          PDB         Topology Spread
──────────────────────────────    ────────    ───────────────
kubectl drain node                ✅ Yes       ❌ No
Node upgrade / maintenance        ✅ Yes       ❌ No
Rolling deployment                ✅ Yes       ❌ No
Zone failure (hardware)           ❌ No        ✅ Yes
Node failure (hardware)           ❌ No        ✅ Yes
All pods in one zone              ❌ No        ✅ Yes
```

---

## Golden Rules

```
1. PDB works perfectly fine WITHOUT Topology Spread
   — they are independent features

2. Topology Spread works perfectly fine WITHOUT PDB
   — but you lose drain protection

3. Use BOTH in production for full high availability
   — Topology Spread protects against zone failures
   — PDB protects against maintenance disruptions

4. Topology Spread only affects scheduling
   — once pods are running, it does nothing
   — it cannot move pods after they are placed

5. PDB only affects voluntary disruptions
   — zone failure bypasses PDB completely
   — that is why you also need Topology Spread
```

---

## Summary

```
Only PDB (no Topology Spread):
✅ Safe during node drain/maintenance
❌ All pods could be in one zone
❌ Zone failure = full outage

Only Topology Spread (no PDB):
✅ Pods spread across zones
❌ Node drain could take down too many pods at once
❌ No protection during maintenance

Both Together:
✅ Pods spread across zones
✅ Protected during node drain/maintenance
✅ Zone failure only takes down some pods
✅ Full high availability
```