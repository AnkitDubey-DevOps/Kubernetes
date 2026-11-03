# Kubernetes HPA and VPA Interview Notes

HPA adds or removes pods to match load. VPA changes the CPU and memory of each pod. These notes explain both in simple words, with the questions interviewers ask most.

## Table of Contents

1. [HPA (Horizontal Pod Autoscaler)](#1-hpa-horizontal-pod-autoscaler)
2. [VPA (Vertical Pod Autoscaler)](#2-vpa-vertical-pod-autoscaler)
3. [HPA vs VPA](#3-hpa-vs-vpa)
4. [Most-asked HPA interview questions](#4-most-asked-hpa-interview-questions)
5. [Most-asked VPA interview questions](#5-most-asked-vpa-interview-questions)
6. [Scenario-based questions and troubleshooting](#6-scenario-based-questions-and-troubleshooting)
7. [Quick revision cheat sheet](#7-quick-revision-cheat-sheet)

---

## 1. HPA (Horizontal Pod Autoscaler)

HPA changes the number of pod replicas of a Deployment, StatefulSet or ReplicaSet based on load. More traffic means more pods; less traffic means fewer pods. This is called scaling out and scaling in.

### How it works in simple words

1. HPA is a control loop inside the kube-controller-manager. It wakes up every 15 seconds by default.
2. It reads metrics for the pods, for example average CPU use.
3. It compares the current value with your target value.
4. It calculates how many replicas are needed and updates the `replicas` field of the Deployment.
5. The Deployment then creates or removes pods.

### The formula (very commonly asked)

```text
desiredReplicas = ceil( currentReplicas × currentMetricValue / targetMetricValue )
```

Example: 4 pods are running at 90% average CPU and the target is 60%. Then desired = ceil(4 × 90 / 60) = ceil(6) = 6 pods.

### What HPA needs to work

- **Metrics Server** installed in the cluster. It gives CPU and memory numbers through the Metrics API (`metrics.k8s.io`).
- **Resource requests** set on the containers. CPU utilization is calculated as a percentage of the request, so without a request HPA cannot compute it.
- A scalable target such as a Deployment. DaemonSets cannot be scaled by HPA.

### Types of metrics HPA can use (autoscaling/v2)

| Metric type | What it means | Example |
| --- | --- | --- |
| Resource | CPU or memory of pods | Average CPU at 60% |
| Pods | A custom metric per pod | Requests per second per pod |
| Object | A metric of another object | Requests per second on an Ingress |
| External | A metric from outside the cluster | Queue length in SQS or Kafka lag |
| ContainerResource | CPU or memory of one container in a pod | Only the app container, not the sidecar |

Custom and external metrics need an adapter such as Prometheus Adapter, because Metrics Server only gives CPU and memory.

### Example YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
```

### Important HPA settings

- **minReplicas / maxReplicas**: the lower and upper limit. HPA never goes outside this range.
- **behavior**: controls how fast it scales up or down, using policies and a stabilization window.
- **Stabilization window**: HPA looks at the recent past and picks the safest value, so it does not flap (scale up and down again and again). Scale down uses 300 seconds by default; scale up uses 0.
- **Tolerance**: if the ratio is within about 10% of 1.0, HPA does nothing. This avoids tiny changes.
- **Cooldown**: older docs mention 3 minutes for scale up and 5 minutes for scale down. In current versions this is handled by the stabilization window.

### Handy commands

```bash
kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=10
kubectl get hpa
kubectl describe hpa web-hpa
kubectl top pods
```

---

## 2. VPA (Vertical Pod Autoscaler)

VPA changes the CPU and memory **requests and limits** of the containers inside a pod. It watches real usage over time and recommends (or sets) better values. This is called scaling up and scaling down, or right-sizing.

VPA is not built into Kubernetes. It is an add-on from the Kubernetes autoscaler project and must be installed separately. It also needs Metrics Server.

### The 3 components

| Component | Job in simple words |
| --- | --- |
| Recommender | Watches current and past usage and calculates the best CPU and memory values. |
| Updater | Checks if running pods have wrong values. If yes, it evicts the pod so it can be recreated. |
| Admission Controller | A webhook that changes the resource requests of a new pod when it is created. |

Flow: Recommender calculates, Updater evicts the pod, the controller (Deployment) creates a new pod, and the Admission Controller sets the new values on that pod.

### Update modes (very commonly asked)

| Mode | What happens |
| --- | --- |
| Off | Only gives recommendations. Nothing is changed. Safest for learning and production checks. |
| Initial | Sets resources only when a pod is created. Never touches running pods. |
| Recreate | Sets resources at creation and also evicts running pods to apply new values. |
| Auto | Currently works like Recreate. Newer versions can apply changes in place without restart where the cluster supports it. |

Newer VPA releases also add an in-place mode that tries to resize the pod without restarting it, using Kubernetes in-place pod resize. It depends on your Kubernetes and VPA version, so check the docs for your cluster before relying on it.

### Example YAML

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

### Reading the recommendation

`kubectl describe vpa web-vpa` shows these values for each container:

- **Lower Bound**: the minimum the pod needs. Below this it will likely struggle.
- **Target**: the value VPA recommends. This is what it applies.
- **Uncapped Target**: the target before your `minAllowed` and `maxAllowed` limits are applied.
- **Upper Bound**: the maximum useful value. Above this is probably waste.

### Limits and things to remember

- In the classic modes, changing resources **restarts the pod** (eviction). Plan for this.
- Do not run **HPA and VPA on the same CPU or memory metric** for the same workload. They fight each other.
- Use at least 2 replicas, otherwise eviction causes downtime. Add a PodDisruptionBudget.
- VPA works best for workloads that cannot scale out easily, such as databases, caches and single-instance apps.
- It does not support scaling a Job or a bare pod that has no controller.
- Memory is not compressible. If a pod uses too much memory it is OOMKilled, and VPA raises the recommendation after that.

---

## 3. HPA vs VPA

Use HPA for stateless apps that handle more load by adding copies. Use VPA when one pod needs the right size, or when the app cannot run in many copies.

| Point | HPA | VPA |
| --- | --- | --- |
| What it changes | Number of pods | CPU and memory of each pod |
| Direction | Horizontal (out and in) | Vertical (up and down) |
| Built into Kubernetes | Yes | No, separate install |
| Pod restart needed | No, only new pods are added | Yes in classic modes, because pods are evicted |
| Best for | Stateless web apps and APIs | Databases, caches, batch, unknown sizing |
| Metrics | CPU, memory, custom, external | CPU and memory usage history |
| Reaction speed | Fast (about 15 s loop) | Slow, learns over hours or days |
| Needs resource requests | Yes, for utilization targets | Sets them for you |
| Main risk | Flapping, hitting maxReplicas | Restarts, conflict with HPA |

### Can HPA and VPA run together?

Yes, but only if they use **different metrics**. Common safe setups:

- HPA on custom metrics (requests per second, queue length) and VPA on CPU and memory.
- VPA in `Off` mode only to give recommendations, while HPA does the real scaling. You then update the requests yourself.

What you must avoid is HPA scaling on CPU while VPA is also changing the CPU request. VPA changes the request, so the utilization percentage HPA sees also changes, and the two keep disturbing each other.

### Related autoscalers interviewers mix in

| Tool | What it scales |
| --- | --- |
| Cluster Autoscaler | Number of nodes. Adds a node when pods are Pending because there is no room, removes nodes that stay underused. |
| Karpenter | Also scales nodes, but picks the right instance type directly and is faster. |
| KEDA | Event-driven scaling of pods from sources like Kafka, SQS or cron. It can scale to zero, which plain HPA cannot do by default. |
| Addon Resizer | Resizes a single add-on pod, mainly used for Metrics Server and similar components. |

**Big picture**: HPA adds pods, and when the nodes are full the pods go Pending. Then Cluster Autoscaler adds nodes so those pods can run. HPA and Cluster Autoscaler work as a team.

---

## 4. Most-asked HPA interview questions

**Q1. What is HPA?**
HPA automatically increases or decreases the number of pod replicas of a workload based on metrics like CPU, memory or custom values. It keeps your app responsive during high load and saves cost during low load.

**Q2. How does HPA decide the number of pods?**
It uses the formula `desired = ceil(current replicas × current metric / target metric)`. If 4 pods run at 90% CPU and the target is 60%, it asks for 6 pods.

**Q3. What do you need before HPA works?**
Metrics Server must be installed, and the pods must have resource requests defined. Without requests, CPU percentage cannot be calculated and the HPA shows `<unknown>`.

**Q4. How often does HPA check metrics?**
Every 15 seconds by default. It is set by the kube-controller-manager flag `--horizontal-pod-autoscaler-sync-period`.

**Q5. What is the difference between autoscaling/v1 and v2?**
v1 supports only CPU utilization. v2 supports memory, custom metrics, external metrics, multiple metrics at once, and the `behavior` field for tuning scale speed.

**Q6. What happens if you give HPA more than one metric?**
HPA calculates the desired replicas for each metric separately and picks the **highest** one. That way every metric stays under its target.

**Q7. What is flapping and how do you stop it?**
Flapping means pods keep scaling up and down repeatedly because the metric moves around the target. Use the stabilization window in `behavior`, set sensible scale-down policies, and keep a gap between min and max replicas.

**Q8. What is the stabilization window?**
A time window in which HPA remembers its past recommendations and chooses the safest one. The default is 300 seconds for scale down and 0 for scale up. It makes scale down slow and careful.

**Q9. Why does the HPA show `<unknown>` for targets?**
Common causes: Metrics Server is not running, the pod has no CPU request, the pods are not ready yet, or the metrics API is not reachable. Check `kubectl describe hpa` and `kubectl top pods`.

**Q10. Can HPA scale to zero?**
Not by default. `minReplicas` must be at least 1. The `HPAScaleToZero` feature gate exists for object or external metrics, but most teams use KEDA for scale to zero.

**Q11. Can HPA scale a DaemonSet?**
No. A DaemonSet runs one pod per node, so there is no replica count to change.

**Q12. CPU Utilization vs AverageValue, what is the difference?**
`Utilization` is a percentage of the pod's CPU request (for example 60%). `AverageValue` is an absolute number (for example 500m CPU per pod).

**Q13. How do you scale on custom metrics like requests per second?**
Install a metrics pipeline such as Prometheus plus Prometheus Adapter. It exposes the metric through the custom metrics API, and the HPA uses a `Pods`, `Object` or `External` metric type.

**Q14. What does the `behavior` field do?**
It controls the speed of scaling. You can limit scale up to "4 pods per minute" or scale down to "50% per minute", and set a stabilization window for each direction.

**Q15. What happens when HPA reaches maxReplicas but load is still high?**
It stops adding pods. The app may slow down or fail. You should raise maxReplicas, check node capacity, and make sure Cluster Autoscaler can add nodes.

**Q16. Why are new pods Pending after HPA scales up?**
The cluster has no free capacity. HPA only creates pods; Cluster Autoscaler or Karpenter is needed to add nodes.

**Q17. How does HPA treat a pod that is still starting up?**
Pods that are not ready, or that are very new, are handled carefully so their startup CPU spike does not trigger extra scale up. You can tune this with the CPU initialization period and readiness probes.

**Q18. Can you use HPA and a manual `kubectl scale` together?**
HPA will overwrite your manual replica count at its next check. If you use GitOps, do not hard-code `replicas` in the Deployment YAML, or each sync will fight the HPA.

---

## 5. Most-asked VPA interview questions

**Q1. What is VPA?**
VPA automatically adjusts the CPU and memory requests (and limits) of containers based on how much they really use. It fixes the common problem of guessing resource values wrongly.

**Q2. Why do we need VPA when we can set requests ourselves?**
People usually guess. If requests are too high you waste money; if too low the pod gets throttled or OOMKilled. VPA uses real usage data to right-size, and keeps adjusting as the app changes.

**Q3. Is VPA part of core Kubernetes?**
No. It is a separate add-on in the kubernetes/autoscaler project. You install its CRDs and its three components, and it also needs Metrics Server.

**Q4. Explain the three VPA components.**
Recommender calculates the best values from usage history. Updater evicts pods that have wrong values. Admission Controller sets the correct values on pods when they are created.

**Q5. What are the VPA update modes?**
`Off` (recommend only), `Initial` (apply at pod creation only), `Recreate` (apply and evict running pods) and `Auto` (currently same as Recreate, and can use in-place resize on newer versions).

**Q6. Does VPA restart pods?**
In the classic modes, yes. To change requests it evicts the pod and a new one starts with new values. That is why you need multiple replicas and a PodDisruptionBudget. Newer in-place resize support can avoid restarts, but depends on your versions.

**Q7. How would you use VPA safely in production?**
Start with `updateMode: "Off"`, read the recommendations for a few days, then apply them through your normal deployment process. Or use `Initial` mode. Move to `Auto` only for workloads that can handle restarts.

**Q8. What do Lower Bound, Target, Upper Bound and Uncapped Target mean?**
Lower Bound is the minimum the pod needs. Target is the recommended value. Upper Bound is the most it should ever need. Uncapped Target is the target before minAllowed and maxAllowed are applied.

**Q9. Why should we not use HPA and VPA together on CPU?**
HPA uses CPU utilization as a percentage of the CPU request. VPA keeps changing that request, so the percentage changes too. Both react to each other and the system becomes unstable.

**Q10. How do minAllowed and maxAllowed help?**
They are guard rails in `resourcePolicy`. VPA will never recommend below minAllowed or above maxAllowed, which stops it from asking for more than a node can give.

**Q11. Does VPA change limits too?**
By default VPA keeps the ratio between request and limit. If your request is 100m and limit is 200m, then when it raises the request to 300m it raises the limit to 600m. You can change this with `controlledValues` (`RequestsOnly` or `RequestsAndLimits`).

**Q12. Which workloads suit VPA?**
Stateful or hard-to-scale-out apps such as databases, caches, monitoring tools, and any app where you do not know the right size. It is not a good fit for apps that must never restart.

**Q13. What if the recommended value is bigger than any node?**
The pod will stay Pending because no node can host it. Set a sensible `maxAllowed`, and make sure your node sizes can handle it.

**Q14. How long does VPA take to give good recommendations?**
It needs some usage history, often a few hours to days. Recommendations from a very short window can be wrong, so do not trust them right after a fresh install.

**Q15. How do you stop VPA from touching one container, for example a sidecar?**
In `containerPolicies`, give that container `mode: "Off"`. VPA will then ignore it.

---

## 6. Scenario-based questions and troubleshooting

**S1. Your app gets a sudden traffic spike every morning at 9 AM and HPA is too slow. What do you do?**
Lower the CPU target (for example 50%) so HPA reacts earlier, raise the scale-up policy in `behavior` (add more pods per minute), keep a higher `minReplicas` before 9 AM, or use KEDA with a cron trigger to scale up in advance.

**S2. HPA shows `<unknown>/60%`. How do you debug?**
Step 1: `kubectl get apiservice v1beta1.metrics.k8s.io` to see if Metrics Server is available. Step 2: `kubectl top pods` to check metrics are returned. Step 3: confirm the Deployment has CPU `requests`. Step 4: `kubectl describe hpa` and read the Events and Conditions.

**S3. Pods keep scaling up and down every few minutes. How do you fix it?**
This is flapping. Add a longer scale-down stabilization window, limit scale-down speed with a policy, use a wider gap between min and max, and check if the metric itself is noisy (for example because of a sidecar or a slow readiness probe).

**S4. HPA scaled to 20 pods but 8 are Pending. Why?**
The nodes are full. Check `kubectl describe pod` for "Insufficient cpu". The fix is Cluster Autoscaler or Karpenter, or adding bigger nodes. Also check if resource requests are unusually large.

**S5. Your database pod keeps getting OOMKilled. Which autoscaler helps?**
VPA. A database cannot be scaled by adding pods easily, so right-size its memory. Start with VPA in `Off` mode to see the recommendation, then raise the memory request and limit.

**S6. Your team wants HPA and VPA on the same service. How do you set it up?**
Use HPA on a custom metric like requests per second or queue length, and VPA for CPU and memory. Or run VPA in `Off` mode and apply the recommendations manually. Never let both control the same CPU or memory metric.

**S7. You want to save cost on 200 microservices with wrong requests. What is your plan?**
Install VPA in `Off` mode on all of them, collect recommendations for 1 to 2 weeks, review the numbers, update requests in your Git repo, and then move only safe services to `Initial` or `Auto`.

**S8. After a VPA update your service had a short outage. Why?**
VPA evicted the only running pod. Run at least 2 replicas, add a PodDisruptionBudget, use readiness probes, and consider `Initial` mode so changes happen only during normal deploys.

**S9. Your queue-based worker should scale with the number of messages. What do you use?**
Use an External metric (queue length) with HPA through an adapter, or KEDA, which has ready-made scalers for SQS, Kafka, RabbitMQ and many more, and can scale down to zero.

**S10. GitOps keeps resetting the replica count that HPA set. Why?**
The Deployment YAML in Git has a fixed `replicas` value, so each sync puts it back. Remove `replicas` from the manifest (or tell the tool to ignore that field) so HPA owns it.

### Quick troubleshooting table

| Symptom | Likely cause | Check or fix |
| --- | --- | --- |
| HPA shows `<unknown>` | No Metrics Server or no CPU requests | `kubectl top pods`, add req
