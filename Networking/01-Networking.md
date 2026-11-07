# Kubernetes Networking – Production-Level Notes

## 1. What Is Kubernetes Networking?

Kubernetes networking enables communication between:

- Pod to Pod
- Pod to Service
- Service to Pod
- Pod to external systems
- External users to applications inside the cluster
- Nodes to Nodes
- Control plane components to worker nodes

Kubernetes networking is designed around these fundamental requirements:

1. Every Pod gets its own IP address.
2. Pods can communicate with other Pods without NAT in a standard cluster networking model.
3. Nodes can communicate with Pods.
4. Services provide stable virtual IP addresses for accessing changing Pods.
5. Network plugins implement the actual networking functionality.

### Kubernetes networking architecture

```text
                 External Users
                       |
                       |
                 LoadBalancer
                       |
                    Ingress
                       |
                    Service
                       |
              -------------------
              |        |        |
            Pod-1    Pod-2    Pod-3
              |        |        |
              -------------------
                       |
                  CNI Network
                       |
                Worker Nodes
```

---

## 2. Kubernetes Networking Components

| Component | Responsibility |
|---|---|
| CNI | Configures Pod networking |
| Pod IP | Unique IP assigned to a Pod |
| Service | Stable endpoint for accessing Pods |
| CoreDNS | Internal DNS resolution |
| kube-proxy | Implements Service networking in common configurations |
| Ingress Controller | Handles HTTP/HTTPS external routing |
| Gateway API Controller | Implements Gateway API traffic routing |
| NetworkPolicy | Controls Pod network traffic |
| LoadBalancer | Exposes Services externally through supported infrastructure |
| EndpointSlice | Stores backend endpoint information |

---

## 3. Container Network Interface (CNI)

**CNI stands for Container Network Interface.**

It is a standard interface used by container runtimes and Kubernetes networking implementations to configure container network connectivity.

When Kubernetes creates a Pod:

1. The scheduler selects a node.
2. The kubelet asks the container runtime to create the Pod sandbox.
3. The runtime invokes the configured CNI implementation.
4. The CNI configures the Pod network interface.
5. The Pod receives an IP address.
6. Routes and other network configuration are established.

### Popular CNI implementations

| CNI | Main capabilities |
|---|---|
| Calico | Networking, NetworkPolicy, routing |
| Cilium | eBPF-based networking, NetworkPolicy, observability |
| Flannel | Basic Pod networking |
| Antrea | Networking and policy enforcement |
| AWS VPC CNI | Integrates Pods with AWS VPC networking |
| Azure CNI | Integrates Pods with Azure networking |
| OVN-Kubernetes | Overlay networking and policy capabilities |

### CNI architecture

```text
Kubernetes
    |
  kubelet
    |
Container Runtime
    |
  CNI Plugin
    |
-------------------------
|                       |
Network Interface     IP Address
|                       |
Routes                Connectivity
```

### Inspect CNI configuration

```bash
ls -l /etc/cni/net.d/
```

Inspect CNI binaries:

```bash
ls -l /opt/cni/bin/
```

Check Pod networking:

```bash
kubectl get pods -A -o wide
```

**Production considerations:**

- Select a CNI based on your cloud and networking requirements.
- Verify NetworkPolicy support before relying on it.
- Check MTU compatibility.
- Monitor IP address capacity.
- Plan Pod CIDRs before cluster deployment.
- Understand whether the CNI uses overlay or native routing.

---

## 4. Pod Networking

Every Kubernetes Pod receives its own IP address.

Containers inside the same Pod share the Pod network namespace.

This means containers inside one Pod can communicate through:

```text
localhost
```

### Example

```text
Pod IP: 10.244.1.10
       |
---------------------
|                   |
Container A       Container B
Port 8080          Port 9090
       |                   |
       --------localhost---
```

Container A can access Container B using:

```bash
curl http://localhost:9090
```

### Check Pod IP

```bash
kubectl get pods -o wide
```

Inspect a particular Pod:

```bash
kubectl describe pod nginx
```

Get only the Pod IP:

```bash
kubectl get pod nginx \
  -o jsonpath='{.status.podIP}'
```

### Test Pod-to-Pod connectivity

```bash
kubectl exec -it pod-a -- sh
```

Inside the container:

```bash
curl http://10.244.2.15:8080
```

Replace the IP with the destination Pod IP.

**Important:** Direct Pod IP communication is useful for troubleshooting, but applications should generally use Services instead of depending on Pod IP addresses.

---

## 5. Pod CIDR and Node CIDR

A Pod CIDR is an IP address range allocated for Pod networking.

Example:

```text
Pod Network: 10.244.0.0/16

Node 1: 10.244.1.0/24
Node 2: 10.244.2.0/24
Node 3: 10.244.3.0/24
```

Example IP allocation:

| Node | Pod CIDR | Example Pod IP |
|---|---|---|
| Worker 1 | 10.244.1.0/24 | 10.244.1.10 |
| Worker 2 | 10.244.2.0/24 | 10.244.2.10 |
| Worker 3 | 10.244.3.0/24 | 10.244.3.10 |

### Check node Pod CIDR

```bash
kubectl get nodes \
  -o custom-columns=NAME:.metadata.name,POD-CIDR:.spec.podCIDR
```

Inspect node details:

```bash
kubectl describe node worker-1
```

### Production planning

Avoid overlapping address ranges between:

- Pod CIDR
- Service CIDR
- VPC CIDR
- Corporate network CIDR
- VPN CIDR
- On-premises network CIDR

IP planning is especially important in hybrid-cloud and multi-cluster environments.

---

## 6. Kubernetes Service

A Service provides a stable network endpoint for a group of Pods.

Pods are temporary. Their IP addresses can change when they are recreated.

A Service provides a stable DNS name and virtual IP address.

### Without Service

```text
Application
    |
    |
Pod IP: 10.244.1.10

Pod gets recreated.

New Pod IP: 10.244.1.20
```

The application would need to discover the new IP.

### With Service

```text
Application
    |
    |
Service: 10.96.10.20
    |
-----------------------
|          |          |
Pod-1      Pod-2      Pod-3
```

The Service remains stable while its backend Pods can change.

---

## 7. Service Types

Kubernetes supports four commonly used Service types.

| Service Type | Purpose |
|---|---|
| ClusterIP | Internal cluster access |
| NodePort | Exposes a Service through a port on each node |
| LoadBalancer | Requests an external load balancer from supported infrastructure |
| ExternalName | Maps a Service to an external DNS name |

### 7.1 ClusterIP

ClusterIP is the default Service type.

It exposes an application inside the cluster.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

Meaning:

- `port`: Service port.
- `targetPort`: Container destination port.
- `selector`: Selects backend Pods.
- `type`: ClusterIP.

Apply:

```bash
kubectl apply -f backend-service.yaml
```

Check:

```bash
kubectl get svc -n production
```

Access from another Pod:

```bash
curl http://backend-service.production.svc.cluster.local
```

### 7.2 NodePort

NodePort exposes a Service through a port on each node.

Default NodePort range:

```text
30000-32767
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Access:

```text
http://NODE-IP:30080
```

**Production considerations:**

- Restrict node access through firewalls or security groups.
- Do not expose unnecessary node ports.
- Prefer a managed LoadBalancer or suitable ingress/gateway architecture for production external access.

### 7.3 LoadBalancer

A LoadBalancer Service requests an external load balancer from the infrastructure provider or a configured load-balancer implementation.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

Check:

```bash
kubectl get svc frontend-lb
```

Look for:

```text
EXTERNAL-IP
```

The address may remain pending if the cluster has no working load-balancer implementation or infrastructure integration.

### 7.4 ExternalName

ExternalName maps a Kubernetes Service name to an external DNS name.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
  namespace: production
spec:
  type: ExternalName
  externalName: database.example.com
```

Applications can resolve:

```text
external-database.production.svc.cluster.local
```

The DNS response points to:

```text
database.example.com
```

ExternalName does not create a proxy or load balancer.

---

## 8. Service Selectors and Endpoints

A Service selects Pods using labels.

Example Pod labels:

```yaml
metadata:
  labels:
    app: backend
```

Matching Service selector:

```yaml
selector:
  app: backend
```

If the selector does not match any Pods, the Service will not have usable backend endpoints.

### Check endpoints

```bash
kubectl get endpoints backend-service
```

Preferred modern inspection:

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service
```

Describe the Service:

```bash
kubectl describe svc backend-service
```

### Common problem

Service exists, but traffic does not reach Pods.

Check:

1. Pod labels.
2. Service selector.
3. Pod readiness.
4. Target port.
5. EndpointSlices.
6. NetworkPolicy.
7. Application listening address.

---

## 9. Service Port vs TargetPort vs NodePort

| Field | Meaning |
|---|---|
| port | Port exposed by the Service |
| targetPort | Port on the backend Pod |
| nodePort | Port exposed on cluster nodes |

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080
```

Traffic flow:

```text
Client
  |
  v
Node IP:30080
  |
  v
Service Port:80
  |
  v
Pod Port:8080
```

---

## 10. Headless Service

A headless Service does not allocate a normal ClusterIP.

Set:

```yaml
clusterIP: None
```

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  clusterIP: None
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

Use cases:

- StatefulSets.
- Databases.
- Distributed systems.
- Applications that need individual Pod DNS records.

A headless Service returns DNS records for selected endpoints rather than a normal virtual Service IP.

---

## 11. StatefulSet Networking

StatefulSets commonly use headless Services to provide stable network identities.

Example DNS names:

```text
database-0.database.production.svc.cluster.local
database-1.database.production.svc.cluster.local
database-2.database.production.svc.cluster.local
```

This is useful for clustered databases and distributed applications.

---

## 12. Important Production Networking Checklist

- [ ] Plan Pod CIDR and Service CIDR.
- [ ] Avoid overlapping network ranges.
- [ ] Select a suitable CNI.
- [ ] Verify cross-node Pod connectivity.
- [ ] Verify Service endpoint selection.
- [ ] Check readiness probes.
- [ ] Use Services instead of hardcoded Pod IPs.
- [ ] Restrict external exposure.
- [ ] Monitor Pod IP utilization.
- [ ] Validate MTU and routing.
- [ ] Test DNS resolution.
- [ ] Document network dependencies.
