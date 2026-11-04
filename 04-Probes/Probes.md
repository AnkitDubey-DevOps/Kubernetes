
# Kubernetes Probes

Kubernetes Probes are used to check the health and availability of containers inside a Pod.

There are three types of probes:

1. Liveness Probe
2. Readiness Probe
3. Startup Probe

---

## 1. Liveness Probe

**Purpose:** Checks whether the container is alive or not.

- If the check fails, Kubernetes restarts the container.
- It helps to fix containers that are stuck or not responding.
- It does not check whether the application is ready to receive traffic.

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
    - name: nginx
      image: nginx
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
```

**Remember:** Liveness Probe = Restart the unhealthy container.

---

## 2. Readiness Probe

**Purpose:** Checks whether the application is ready to receive traffic.

- If the check succeeds, the Pod becomes ready to receive traffic.
- If the check fails, Kubernetes removes the Pod from Service endpoints.
- The container is not restarted just because the readiness check fails.

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
spec:
  containers:
    - name: nginx
      image: nginx
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
```

**Remember:** Readiness Probe = Allow or stop traffic.

---

## 3. Startup Probe

**Purpose:** Checks whether the application has started successfully.

- Useful for applications that take a long time to start.
- Until the startup probe succeeds, liveness and readiness probes are not executed.
- If the startup probe fails beyond its allowed failure threshold, Kubernetes restarts the container.

**Example:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
    - name: nginx
      image: nginx
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 10
```

**Remember:** Startup Probe = Check whether the application has started.

---

## 4. Difference Between All Three Probes

| Probe | Purpose | What happens on failure? |
|---|---|---|
| Liveness | Checks if the container is alive. | Container is restarted. |
| Readiness | Checks if the application can receive traffic. | Pod is removed from Service endpoints. |
| Startup | Checks if the application has started. | Container is restarted after the failure threshold is reached. |

---

## 5. Important Probe Parameters

| Parameter | Meaning |
|---|---|
| `initialDelaySeconds` | Time to wait before the first probe starts. |
| `periodSeconds` | How often the probe runs. |
| `timeoutSeconds` | Maximum time to wait for a probe response. |
| `failureThreshold` | Number of consecutive failures before the probe is considered failed. |
| `successThreshold` | Number of consecutive successes needed to consider the probe successful. |

---

## 6. Probe Types

Kubernetes supports different ways to perform health checks:

- **httpGet:** Sends an HTTP request to an endpoint.
- **tcpSocket:** Checks whether a TCP connection can be established.
- **exec:** Runs a command inside the container.

---

## 7. Useful Commands

```bash
# List Pods
kubectl get pods

# Check Pod details and probe events
kubectl describe pod <pod-name>

# Check Pod logs
kubectl logs <pod-name>

# Watch Pod status
kubectl get pods -w

# Apply Pod YAML
kubectl apply -f pod.yaml
```

---

## 8. Interview Questions

**Q1. What is a Liveness Probe?**

It checks whether the container is alive. If it fails repeatedly, Kubernetes restarts the container.

**Q2. What is a Readiness Probe?**

It checks whether the application is ready to receive traffic.

**Q3. What is a Startup Probe?**

It checks whether the application has started successfully.

**Q4. What happens when a Readiness Probe fails?**

The Pod is removed from Service endpoints until it becomes ready again. The container is not restarted just because of this failure.

**Q5. Why do we use a Startup Probe?**

To give slow-starting applications enough time to start before liveness and readiness checks begin.

**Q6. Can we use all three probes together?**

Yes. They can be configured together for the same container.
