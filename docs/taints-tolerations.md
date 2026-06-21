# Taints and Tolerations

## Concept

Taint  = applied to a NODE. Repels pods.
Toleration = applied to a POD. Permits scheduling on a tainted node.

Toleration does not attract a pod to a node, it only removes a block.
For attraction, use nodeSelector / nodeAffinity.

```
Node (tainted)                      Pod
┌────────────────────┐              ┌───────────────────┐
│ taint:               │  no toleration  │ no tolerations      │
│ gpu=true:NoSchedule  │◄──BLOCKED───────│                     │
└────────────────────┘              └───────────────────┘

┌────────────────────┐              ┌───────────────────┐
│ taint:               │   matching      │ toleration:          │
│ gpu=true:NoSchedule  │◄──ALLOWED───────│ key=gpu value=true   │
│                       │                 │ effect=NoSchedule    │
└────────────────────┘              └───────────────────┘
```

To force placement (not just permit), combine taint/toleration (repel others)
with nodeSelector/nodeAffinity (attract this pod) on the same node.

## Add / remove a taint

```bash
# add
kubectl taint nodes <node-name> <key>=<value>:<effect>

# remove (trailing dash)
kubectl taint nodes <node-name> <key>=<value>:<effect>-
```

Example:

```bash
kubectl taint nodes gpu-node-1 gpu=true:NoSchedule
```

## Effects

| Effect           | New pods (no toleration) | Running pods (no toleration) |
|------------------|---------------------------|-------------------------------|
| NoSchedule       | blocked                   | unaffected                    |
| PreferNoSchedule | soft avoid, not guaranteed| unaffected                    |
| NoExecute        | blocked                   | evicted                       |

### NoSchedule

Hard block at scheduling time only. The scheduler will not place a new pod
on the node unless it has a matching toleration.

Does NOT touch pods that are already running on the node. If you taint a
node AFTER pods are already running there, those pods keep running —
NoSchedule has no retroactive effect.

```
timeline:
  t0: pod-x running on node1 (no toleration)
  t1: kubectl taint node1 key=val:NoSchedule
  t2: pod-x is STILL running  <- NoSchedule does not evict
  t3: new pod-y (no toleration) tries to schedule on node1 -> BLOCKED
```

Use case: stop new workloads from landing on a node (e.g. draining
candidate, dedicated node pool) without disturbing what's already there.

### PreferNoSchedule

Same idea as NoSchedule but it's a preference, not a guarantee. The
scheduler scores tainted nodes lower and tries other nodes first. If no
untainted/tolerated node is available, the pod still schedules on the
tainted node.

```
2 nodes, both can fit the pod:
  node1: no taint              -> scheduler picks this one
  node2: key=val:PreferNoSchedule

3rd scenario, only node2 has capacity:
  node1: no taint, but full     -> can't fit
  node2: key=val:PreferNoSchedule -> pod lands here anyway
```

Use case: "prefer general-purpose nodes, but it's fine to use the
GPU/expensive node if nothing else fits."

### NoExecute

The only effect that acts on pods already running on the node, not just
new scheduling decisions. Two things happen at once:

1. New pods without a matching toleration -> blocked from scheduling.
2. Existing pods without a matching toleration -> evicted immediately
   (or after `tolerationSeconds` if a toleration with that field is set).

```
timeline:
  t0: pod-x running on node1 (no toleration)
  t1: kubectl taint node1 key=val:NoExecute
  t2: pod-x is EVICTED  <- NoExecute is retroactive
```

This is exactly the mechanism behind node failure handling. When a node
goes unhealthy, the control plane auto-adds:

```
node.kubernetes.io/not-ready:NoExecute
node.kubernetes.io/unreachable:NoExecute
```

Every pod gets a default toleration for these two taints with
`tolerationSeconds: 300` (set automatically unless overridden), which is
why a pod stays scheduled for ~5 minutes after its node goes unreachable
before getting rescheduled elsewhere.

```yaml
# this is added implicitly by Kubernetes if you don't specify your own
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

Use case: enforce hard isolation (security-sensitive nodes, GPU nodes you
never want shared workloads touching even transiently) and the automatic
node-health eviction behavior above.

## Pod toleration

```yaml
tolerations:
- key: "gpu"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

## Match / no-match

```
Node taint: gpu=true:NoSchedule

Pod A: key=gpu, value=true,  effect=NoSchedule   -> MATCH    -> scheduled
Pod B: key=gpu, value=false, effect=NoSchedule   -> NO MATCH -> blocked
Pod C: no toleration                              -> NO MATCH -> blocked
```

## Operator: Equal vs Exists

```
Equal   -> key AND value must match exactly
Exists  -> only key must be present, value ignored
```

```yaml
tolerations:
- key: "gpu"
  operator: "Exists"
  effect: "NoSchedule"
```

This tolerates any taint with key=gpu, regardless of value.

## tolerationSeconds (NoExecute only)

How long a pod stays bound to the node after a NoExecute taint appears,
before it's evicted.

```yaml
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

No tolerationSeconds set -> pod tolerates that taint forever (never evicted for it).

## Built-in taints (auto-applied by Kubernetes)

```
node.kubernetes.io/not-ready
node.kubernetes.io/unreachable
node.kubernetes.io/disk-pressure
node.kubernetes.io/memory-pressure
node.kubernetes.io/unschedulable
node-role.kubernetes.io/control-plane:NoSchedule   (control plane nodes)
```

`not-ready` / `unreachable` use NoExecute -> this is what evicts pods when a
node goes down.

## Check taints on a node

```bash
kubectl describe node <node-name> | grep Taints
```

## Check tolerations on a pod

```bash
kubectl get pod <pod-name> -o yaml | grep -A5 tolerations
```

## GPU node use case

Taint the GPU MachineSet so only GPU workloads land there:

```bash
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
```

GPU Operator daemonsets and inference pods carry the matching toleration.
Everything else gets repelled automatically — no nodeSelector needed on
the repel side.

## CKA notes

- Taint/toleration = repel only. Not the same as node affinity (= attract).
- Multiple taints on one node -> pod must tolerate ALL of them to schedule.
- Trailing dash on kubectl taint removes the taint.
- NoExecute is the only effect that can evict already-running pods.
