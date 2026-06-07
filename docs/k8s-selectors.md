# Kubernetes Labels and Selectors

## What is a Label?

Key/value pairs attached to pods (or any k8s object).

```yaml
metadata:
  labels:
    app: nginx
    environment: production
    tier: frontend
```

Labels are just tags. Nothing happens automatically by adding them.  
Selectors are what USE those tags.

---

## What is a Selector?

A query that finds objects by their labels.

This is how:
- A **Service** finds which Pods to send traffic to
- A **Deployment** knows which Pods it owns

```
Pod (labels: app=nginx, env=production)
         ▲
         │  selector: app=nginx  →  matches!
         │
      Service
```

---

## Two Types of Selectors

### 1. Equality-based

Match by exact value. Three operators: `=`, `==`, `!=`

---

#### `=` (equals)

Find pods where environment is exactly production.

```bash
kubectl get pods -l environment=production
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-equal
spec:
  selector:
    environment: production     # finds pods where environment=production
  ports:
    - port: 80
      targetPort: 8080
```

```
pod-1  environment=production   ✅ matches
pod-2  environment=staging      ❌ no match
pod-3  environment=qa           ❌ no match
```

---

#### `==` (equals — same as `=`)

`==` and `=` do the same thing. Both are valid.

```bash
kubectl get pods -l environment==production
```

```yaml
# in YAML there is no difference — both write the same way
spec:
  selector:
    environment: production
```

---

#### `!=` (not equals)

Find pods where tier is anything EXCEPT frontend.

```bash
kubectl get pods -l tier!=frontend
```

```yaml
# != cannot be used directly in matchLabels
# use matchExpressions instead
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-notequal
spec:
  selector:
    matchExpressions:
      - key: tier
        operator: NotIn
        values:
          - frontend
  template:
    metadata:
      labels:
        tier: backend
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
pod-1  tier=backend     ✅ matches (not frontend)
pod-2  tier=cache       ✅ matches (not frontend)
pod-3  tier=frontend    ❌ no match
```

---

#### combining equality operators

Multiple conditions separated by comma = AND

```bash
kubectl get pods -l environment=production,tier!=frontend
```

```
pod-1  environment=production, tier=backend    ✅ both match
pod-2  environment=production, tier=frontend   ❌ tier fails
pod-3  environment=staging,    tier=backend    ❌ env fails
```

---

### 2. Set-based

Match against a set of values. Four operators: `in`, `notin`, `exists`, `!`

---

#### `in`

Find pods where environment is one of the listed values.

```bash
kubectl get pods -l 'environment in (production,qa)'
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-in
spec:
  selector:
    matchExpressions:
      - key: environment
        operator: In
        values:
          - production
          - qa
  template:
    metadata:
      labels:
        environment: production
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
pod-1  environment=production   ✅ in the list
pod-2  environment=qa           ✅ in the list
pod-3  environment=staging      ❌ not in the list
```

---

#### `notin`

Find pods where tier is NOT any of the listed values.

```bash
kubectl get pods -l 'tier notin (frontend,cache)'
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-notin
spec:
  selector:
    matchExpressions:
      - key: tier
        operator: NotIn
        values:
          - frontend
          - cache
  template:
    metadata:
      labels:
        tier: backend
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
pod-1  tier=backend     ✅ not in (frontend, cache)
pod-2  tier=database    ✅ not in (frontend, cache)
pod-3  tier=frontend    ❌ in the excluded list
pod-4  tier=cache       ❌ in the excluded list
```

---

#### `exists` (key only, any value)

Find pods that HAVE the label key `partition` — value doesn't matter.

```bash
kubectl get pods -l 'partition'
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-exists
spec:
  selector:
    matchExpressions:
      - key: partition
        operator: Exists
  template:
    metadata:
      labels:
        partition: customerA
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
pod-1  partition=customerA   ✅ key exists
pod-2  partition=customerB   ✅ key exists
pod-3  (no partition label)  ❌ key missing
```

---

#### `!` (does not exist — key must be absent)

Find pods that do NOT have the label key `partition` at all.

```bash
kubectl get pods -l '!partition'
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-doesnotexist
spec:
  selector:
    matchExpressions:
      - key: partition
        operator: DoesNotExist
  template:
    metadata:
      labels:
        app: nginx          # no partition label intentionally
    spec:
      containers:
      - name: nginx
        image: nginx
```

```
pod-1  (no partition label)  ✅ key is absent
pod-2  (no partition label)  ✅ key is absent
pod-3  partition=customerA   ❌ key exists
```

---

#### combining set-based operators

```bash
kubectl get pods -l 'environment in (production,qa),tier notin (frontend,cache)'
```

```
pod-1  environment=production, tier=backend    ✅ both match
pod-2  environment=qa,         tier=database   ✅ both match
pod-3  environment=staging,    tier=backend    ❌ env not in list
pod-4  environment=production, tier=frontend   ❌ tier excluded
```

---

## Equality vs Set — Side by Side

```yaml
# Equality-based — matchLabels (simple)
spec:
  selector:
    matchLabels:
      environment: production
      tier: backend

# Set-based — matchExpressions (advanced)
spec:
  selector:
    matchExpressions:
      - key: environment
        operator: In
        values: [production, qa]
      - key: tier
        operator: NotIn
        values: [frontend, cache]

# Both combined — valid, acts as AND
spec:
  selector:
    matchLabels:
      app: myapp                        # exact match
    matchExpressions:
      - key: environment
        operator: In
        values: [production, qa]        # set match
```

---

## All Operators Quick Reference

| Operator | Type | kubectl | YAML operator |
|---|---|---|---|
| `=` | Equality | `env=production` | `matchLabels` |
| `==` | Equality | `env==production` | `matchLabels` |
| `!=` | Equality | `tier!=frontend` | `NotIn` |
| `in` | Set | `'env in (a,b)'` | `In` |
| `notin` | Set | `'tier notin (a,b)'` | `NotIn` |
| `exists` | Set | `'partition'` | `Exists` |
| `!` (not exists) | Set | `'!partition'` | `DoesNotExist` |

---

## How Service Uses Selector to Find Pods

```
Pods in cluster:

  pod-1  labels: app=nginx, env=production
  pod-2  labels: app=nginx, env=staging
  pod-3  labels: app=redis, env=production

Service selector:
  app: nginx
  env: production

Result:
  traffic goes to → pod-1 only
                    pod-2 ❌ env doesn't match
                    pod-3 ❌ app doesn't match
```

---

## Common Mistake

Selector and Pod label must match exactly — typo = pod not picked up.

```yaml
# Deployment selector
selector:
  matchLabels:
    app: myapp

# Pod template labels
template:
  metadata:
    labels:
      app: myapp      # ✅ matches
      app: my-app     # ❌ deployment won't manage this pod
```

---

## Quick Commands

```bash
# filter by label
kubectl get pods -l app=nginx

# show labels in output
kubectl get pods --show-labels

# add a label
kubectl label pod mypod version=v1

# remove a label
kubectl label pod mypod version-

# overwrite a label
kubectl label pod mypod version=v2 --overwrite
```
