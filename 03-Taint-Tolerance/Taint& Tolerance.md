# Kubernetes Taints and Tolerations

## 1. Introduction

Taints and Tolerations are used in Kubernetes to control which Pods can run on which Nodes.

- **Taint:** Applied to a Node. It prevents unwanted Pods from being scheduled on that Node.
- **Toleration:** Applied to a Pod. It allows the Pod to tolerate a matching Node taint.

**Remember:**
- Taint = Repels Pods.
- Toleration = Allows Pods to tolerate taints.

---

## 2. What is a Taint?

A taint is applied to a Node to prevent certain Pods from being scheduled on it.

### Syntax

```bash
kubectl taint nodes <node-name> key=value:effect
```

### Example

```bash
kubectl taint nodes worker-node app=backend:NoSchedule
```

**Explanation:**
- `worker-node` = Node name.
- `app` = Taint key.
- `backend` = Taint value.
- `NoSchedule` = Taint effect.

---

## 3. Types of Taint Effects

Kubernetes provides three taint effects.

| Effect | Description |
|---|---|
| `NoSchedule` | New Pods cannot be scheduled without a matching toleration. |
| `PreferNoSchedule` | Kubernetes tries to avoid scheduling Pods on this Node. |
| `NoExecute` | New Pods are blocked, and existing non-tolerating Pods are evicted. |

### NoSchedule

```bash
kubectl taint nodes worker-node app=backend:NoSchedule
```

Prevents new Pods without matching tolerations from being scheduled.

### PreferNoSchedule

```bash
kubectl taint nodes worker-node app=backend:PreferNoSchedule
```

Kubernetes tries to avoid scheduling Pods on this Node.

### NoExecute

```bash
kubectl taint nodes worker-node app=backend:NoExecute
```

Prevents new Pods and evicts existing Pods that do not tolerate the taint.

---

## 4. What is a Toleration?

A toleration is added to a Pod YAML file.

It allows a Pod to tolerate a matching Node taint.

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "backend"
      effect: "NoSchedule"

  containers:
    - name: nginx
      image: nginx
```

Apply the YAML:

```bash
kubectl apply -f pod.yaml
```

This Pod can tolerate:

```bash
kubectl taint nodes worker-node app=backend:NoSchedule
```

**Note:** Toleration does not guarantee that a Pod will be scheduled on that Node. It only allows the Pod to tolerate the matching taint.

---

## 5. Toleration Operators

There are two commonly used operators.

### Equal

The key and value must match.

```yaml
tolerations:
  - key: "app"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"
```

### Exists

Only the key needs to exist. The value does not need to match.

```yaml
tolerations:
  - key: "app"
    operator: "Exists"
    effect: "NoSchedule"
```

**Difference:**

- `Equal` = Matches key and value.
- `Exists` = Matches only the key.

---

## 6. Multiple Taints and Tolerations

A Node can have multiple taints, and a Pod can have multiple tolerations.

### Add Multiple Taints

```bash
kubectl taint nodes worker-node app=backend:NoSchedule

kubectl taint nodes worker-node environment=production:NoExecute
```

### Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-pod
spec:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "backend"
      effect: "NoSchedule"

    - key: "environment"
      operator: "Equal"
      value: "production"
      effect: "NoExecute"

  containers:
    - name: nginx
      image: nginx
```

The Pod tolerates both taints.

---

## 7. Toleration Seconds

`tolerationSeconds` is used with the `NoExecute` effect.

It specifies how long a Pod can remain on a Node after a matching `NoExecute` taint is added.

### Example

```yaml
tolerations:
  - key: "app"
    operator: "Equal"
    value: "backend"
    effect: "NoExecute"
    tolerationSeconds: 60
```

**Meaning:** The Pod can remain on the Node for 60 seconds after the matching taint is applied. After that, Kubernetes can evict it.

---

## 8. Taints and Tolerations vs Node Affinity

| Taints and Tolerations | Node Affinity |
|---|---|
| Repel unwanted Pods from Nodes. | Attract Pods to specific Nodes. |
| Taints are applied to Nodes. | Uses Node labels. |
| Tolerations are added to Pods. | Affinity rules are added to Pods. |

**Remember:**
- Taints = Repel.
- Tolerations = Allow.
- Node Affinity = Attract.

---

## 9. Remove a Taint

Add `-` at the end of the command to remove a taint.

### Syntax

```bash
kubectl taint nodes <node-name> key=value:effect-
```

### Example

```bash
kubectl taint nodes worker-node app=backend:NoSchedule-
```

This removes the specified taint.

---

## 10. Useful Kubernetes Commands

```bash
# List Nodes
kubectl get nodes

# Describe a Node
kubectl describe node worker-node

# Add a taint
kubectl taint nodes worker-node app=backend:NoSchedule

# Remove a taint
kubectl taint nodes worker-node app=backend:NoSchedule-

# List Pods
kubectl get pods

# Check which Node is running a Pod
kubectl get pods -o wide

# Apply YAML
kubectl apply -f pod.yaml

# Delete a Pod
kubectl delete pod backend-pod
```

---

## 11. Interview Questions

**Q1. What is a taint?**

A taint is applied to a Node to repel Pods without matching tolerations.

**Q2. What is a toleration?**

A toleration allows a Pod to tolerate a matching Node taint.

**Q3. What are the three taint effects?**

- NoSchedule
- PreferNoSchedule
- NoExecute

**Q4. Does toleration guarantee Pod scheduling?**

No. It only allows the Pod to tolerate the matching taint.

**Q5. What is the difference between Equal and Exists?**

Equal matches key and value, while Exists matches only the key.

**Q6. What is NoExecute?**

It blocks new non-tolerating Pods and evicts existing non-tolerating Pods.

**Q7. What is tolerationSeconds?**

It defines how long a Pod can remain on a Node after a matching NoExecute taint is applied.
