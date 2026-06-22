# Node Scheduling: nodeSelector, Node Affinity, Pod (Anti-)Affinity

How to control which node a pod lands on. Three mechanisms, increasing in power:

```
nodeSelector       -> simple key=value match (hard only)
nodeAffinity       -> match node LABELS, hard + soft, rich operators
podAffinity        -> co-locate WITH other pods
podAntiAffinity    -> keep AWAY from other pods
```

All of these are SCHEDULER-side filters. They do NOT evict running pods (except
`...IgnoredDuringExecution` makes that explicit — node label changes are ignored
once the pod is running).

---

## 0. Prereq: label your nodes

Affinity/selector match against node labels, so set them first.

```bash
# add a label
kubectl label node node01 disktype=ssd

# view labels
kubectl get nodes --show-labels
kubectl get node node01 -o jsonpath='{.metadata.labels}' | tr ',' '\n'

# remove a label (note trailing minus)
kubectl label node node01 disktype-
```

---

## 1. nodeSelector

Simplest. Pod schedules ONLY on nodes with ALL the given labels. Hard rule —
if no node matches, pod stays `Pending`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disktype: ssd          # node MUST have this exact label
  containers:
  - name: nginx
    image: nginx
```

```
              disktype=ssd ?
   node01 [disktype=ssd] -----> MATCH    -> eligible
   node02 [disktype=hdd] -----> NO MATCH -> skipped
   node03 [no label]     -----> NO MATCH -> skipped
```

Limitation: only `key=value` equality, ANDed together. No OR, no "exists", no
"in this list", no soft preference. For anything richer -> nodeAffinity.

---

## 2. Node Affinity

Match node labels with operators + hard/soft tiers.

### The two tiers

```
requiredDuringSchedulingIgnoredDuringExecution   -> HARD. No match = Pending.
preferredDuringSchedulingIgnoredDuringExecution  -> SOFT. Try, else schedule anyway.
```

Long names break into two parts:

```
[requiredDuringScheduling] [IgnoredDuringExecution]
        |                            |
   applies when                applies after pod
   scheduling pod              is already running
   (enforced)                  (label change = ignored,
                                pod NOT evicted)
```

### Why only two options exist

The naming follows `<X>DuringScheduling<Y>DuringExecution`, which implies four
combinations. Only two are actually implemented:

```
                          DuringExecution
                     Ignored          Required
                  +----------------+----------------+
During  required  | EXISTS         | planned, never |
Schedu-           |                | implemented    |
ling              +----------------+----------------+
        preferred | EXISTS         | does not exist |
                  +----------------+----------------+
```

The `...RequiredDuringExecution` half was meant to EVICT a running pod if the
node's labels changed so it no longer matched. That was never built — which is
why every current option ends in `IgnoredDuringExecution`: once scheduled, label
changes are ignored and the pod stays put.

Mnemonic:

```
DuringScheduling -> required = filter / preferred = score
DuringExecution  -> always Ignored  (no eviction on label change)
```

Same two options apply identically to nodeAffinity, podAffinity, podAntiAffinity.

### Operators

```
In             label value in {list}
NotIn          label value NOT in {list}      (anti-affinity for NODES)
Exists         label key present (no value)
DoesNotExist   label key absent
Gt             value > X   (integer)
Lt             value < X   (integer)
```

### Hard (required) example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-hard
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
  containers:
  - name: nginx
    image: nginx
```

```
   disktype In (ssd, nvme) ?
   node01 [disktype=ssd ] -> MATCH
   node02 [disktype=nvme] -> MATCH
   node03 [disktype=hdd ] -> NO MATCH
   node04 [no label     ] -> NO MATCH
```

### Soft (preferred) example with weight

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-soft
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80                 # 1-100, higher = stronger pull
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

```
   no SSD node free?
   -> still schedules on whatever node is available (soft = best effort)
   SSD node free?
   -> prefers it (weight nudges scheduler score)
```

### Term logic: OR vs AND (the gotcha)

```
nodeSelectorTerms:           <- list items are OR'd
- matchExpressions:          <- expressions inside one term are AND'd
  - key: A ...
  - key: B ...               (A AND B)
- matchExpressions:
  - key: C ...               OR (C)

Result: (A AND B) OR (C)
```

So to express OR between two conditions, use TWO `nodeSelectorTerms` entries.
To express AND, put both under ONE `matchExpressions`.

---

## 3. Pod Affinity / Anti-Affinity

Schedule relative to OTHER PODS, not node labels. Decision is based on whether
matching pods already run within a given topology (node, zone, etc).

### topologyKey — the core concept

A node label that defines the "domain" the rule applies within.

Read it as: "within each group of nodes sharing the same value of <topologyKey>,
apply this rule."

#### Well-known topology keys

`topologyKey` can be ANY node label, including custom ones you set yourself
(e.g. `rack`). These three are the standard, auto-populated ones:

```
kubernetes.io/hostname              -> one node      (most granular)
topology.kubernetes.io/zone         -> availability zone
topology.kubernetes.io/region       -> region         (broadest)
```

The hierarchy:

```
region                                  (e.g. us-east-1)
  +-- zone  (us-east-1a)
  |     +-- node01  hostname
  |     +-- node02  hostname
  +-- zone  (us-east-1b)
        +-- node03  hostname
        +-- node04  hostname
```

Pick the LEVEL the rule applies within:

```
topologyKey: kubernetes.io/hostname  -> "per node"   -> tightest spread/co-locate
topologyKey: .../zone                -> "per zone"   -> survive a zone outage
topologyKey: .../region              -> "per region" -> rarely used here
```

Notes:
- `kubernetes.io/hostname` has a UNIQUE value per node, so it always defines
  single-node domains -> the default for "one replica per node" anti-affinity.
- Older clusters use the deprecated beta names
  `failure-domain.beta.kubernetes.io/zone` and `.../region`. GA names are
  `topology.kubernetes.io/zone` and `.../region`. Recognize both on the CKA.

### Pod Affinity (co-locate)

"Put me on a node/zone that ALREADY has a pod matching this label."
Use case: app pod near its cache.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache          # find pods with app=cache
        topologyKey: kubernetes.io/hostname   # on the SAME node
  containers:
  - name: web
    image: nginx
```

```
   node01: [cache pod]  -> web schedules HERE (same node as cache)
   node02: []           -> not chosen
```

### Pod Anti-Affinity (spread apart)

"Do NOT put me where a matching pod already runs."
Use case: spread replicas across nodes for HA.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web        # avoid other app=web pods
            topologyKey: kubernetes.io/hostname  # one per node
      containers:
      - name: web
        image: nginx
```

```
   WITHOUT anti-affinity            WITH anti-affinity (hostname)
   node01: [web][web]               node01: [web]
   node02: [web]                    node02: [web]
   node03: []                       node03: [web]
   (clustered, risky)               (spread, HA)
```

Note: with 3 replicas + hard anti-affinity + only 2 nodes, the 3rd pod stays
`Pending` (no node left without a web pod). Use `preferred` if you want spread
as best-effort instead of a hard cap.

---

## 4. Quick comparison

```
                    matches      hard   soft   relative to
nodeSelector        node labels  yes    no     node
nodeAffinity        node labels  yes    yes    node
podAffinity         pod labels   yes    yes    other pods (+ topologyKey)
podAntiAffinity     pod labels   yes    yes    other pods (+ topologyKey)
```

```
Need simple node pin?         -> nodeSelector
Need OR / operators / soft?   -> nodeAffinity
Need to sit with other pods?  -> podAffinity
Need to spread pods apart?    -> podAntiAffinity
Need to REPEL pods from node? -> taints + tolerations (different mechanism)
```

Affinity = pod chooses node. Taints/tolerations = node repels pods. They are
complementary, often combined: taint dedicates a node, affinity pulls the right
pods onto it.

---

## 5. Debugging Pending pods

```bash
kubectl describe pod <name>
# look at Events for:
#   "0/3 nodes are available: 3 node(s) didn't match
#    Pod's node affinity/selector"   -> no node satisfies the rule

kubectl get nodes --show-labels      # confirm node labels exist
kubectl get pods -o wide             # see which node each pod is on (anti-affinity check)
```

Common causes:
- typo in label key/value (selector is case-sensitive, exact match)
- node label not set
- hard anti-affinity + replicas > available nodes -> permanent Pending
- AND vs OR term structure wrong (see 2: nodeSelectorTerms gotcha)
```
