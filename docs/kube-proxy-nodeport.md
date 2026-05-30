# How kube-proxy Works with NodePort

## Setup

3 nodes, pod running only on Node1:

```
Node1 → runs nginx pod   IP: 10.0.1.1
Node2 → no nginx pod     IP: 10.0.2.1
Node3 → no nginx pod     IP: 10.0.3.1
```

You create a NodePort service on port `32311`.

---

## What happens when you hit Node3 IP?

```
curl http://10.0.3.1:32311
```

You might think — pod is on Node1, why would Node3 work?

---

## The Answer — kube-proxy

When you create a NodePort service, **kube-proxy runs on every node**.

It does one job — writes `iptables` rules on every node:

```
"any traffic on port 32311 → send to nginx pod at 10.0.1.5:80"
```

This rule is written on Node1, Node2, and Node3 — **before any request even comes in**.

---

## Request Flow (hitting Node3)

```
you
 │
 └──► Node3:32311
          │
          ├── iptables rule matches port 32311
          │
          ├── DNAT: rewrite destination
          │   from → 10.0.3.1:32311
          │   to   → 10.0.1.5:80 (nginx pod on Node1)
          │
          └──► packet travels over cluster network to Node1
                    │
                    └──► nginx pod handles request
                              │
                              └──► response goes back Node1 → Node3 → you
```

---

## Key Point

kube-proxy does **not** sit in the request path.

It only programs iptables rules ahead of time. The Linux kernel handles the actual packet forwarding — no proxy process involved at request time.

```
kube-proxy job = write rules (done once when svc is created)
iptables job   = forward packets (done on every request, in kernel)
```

---

## iptables Rules kube-proxy Writes

```
KUBE-NODEPORTS
  └── tcp port 32311 → KUBE-SVC-NGINX

KUBE-SVC-NGINX
  └── KUBE-SEP-POD1 → DNAT to 10.0.1.5:80
```

If you have multiple nginx pods, it load balances:

```
KUBE-SVC-NGINX
  ├── KUBE-SEP-POD1 → 10.0.1.5:80  (50%)
  └── KUBE-SEP-POD2 → 10.0.2.7:80  (50%)
```

---

## kube-proxy Modes

| Mode | How |
|---|---|
| `iptables` | default, writes NAT rules in kernel |
| `ipvs` | faster, better for large clusters |
| `userspace` | old, deprecated |

Check your cluster mode:

```bash
kubectl get configmap kube-proxy-config -n kube-system -o yaml | grep mode
```

---

## AWS / EKS Note

For NodePort to work on EKS with public node IPs:

- Security Group must allow inbound TCP on `32311` for all nodes
- NodePort range is `30000–32767` by default

```bash
# verify your service
kubectl get svc

# check which node your pod is on
kubectl get pods -o wide

# test
curl http://<any-node-public-ip>:32311
```

---

## When NOT to use NodePort

NodePort is fine for learning and testing.

In production, use:
- **LoadBalancer service** → AWS gives you a single ELB DNS endpoint
- **Ingress** → path/host based routing, one entry point for multiple services
