# Kubernetes Networking – DNS, CoreDNS and kube-proxy

## 13. Kubernetes DNS

Kubernetes provides internal DNS resolution so applications can communicate using Service names instead of IP addresses.

Example:

```text
backend-service.production.svc.cluster.local
```

Applications can use this DNS name to access a Service.

### Standard Service DNS format

```text
<SERVICE>.<NAMESPACE>.svc.cluster.local
```

Example:

```text
mysql.production.svc.cluster.local
```

### DNS resolution examples

From the same namespace:

```bash
curl http://backend-service
```

From another namespace:

```bash
curl http://backend-service.production
```

Fully qualified DNS name:

```bash
curl http://backend-service.production.svc.cluster.local
```

---

## 14. CoreDNS

CoreDNS provides DNS services inside Kubernetes.

Responsibilities:

- Resolves Kubernetes Service names.
- Resolves Pod DNS names when configured.
- Forwards external DNS queries.
- Supports DNS plugins and configuration.
- Provides DNS caching and other DNS functionality.

### CoreDNS architecture

```text
Application Pod
      |
      |
 DNS Query
      |
      v
CoreDNS Service
      |
      |
CoreDNS Pods
      |
-----------------------
|                     |
Kubernetes DNS      External DNS
Records             Forwarding
```

### Check CoreDNS Pods

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

### Check CoreDNS Service

```bash
kubectl get svc -n kube-system kube-dns
```

### Inspect CoreDNS configuration

```bash
kubectl get configmap coredns \
  -n kube-system -o yaml
```

### Check CoreDNS logs

```bash
kubectl logs -n kube-system \
  -l k8s-app=kube-dns
```

### Common CoreDNS configuration

A typical Corefile contains:

```text
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

Exact configuration depends on the Kubernetes distribution and CoreDNS version.

---

## 15. Pod DNS Configuration

A Pod normally receives DNS settings from the cluster DNS configuration.

Inspect a Pod:

```bash
kubectl exec -it nginx -- cat /etc/resolv.conf
```

Example:

```text
nameserver 10.96.0.10
search production.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The actual nameserver and search domains depend on the cluster configuration.

### DNS policy options

| DNS Policy | Purpose |
|---|---|
| ClusterFirst | Uses cluster DNS for normal queries |
| Default | Inherits node DNS configuration |
| ClusterFirstWithHostNet | Cluster DNS behavior for hostNetwork Pods |
| None | Uses custom DNS settings |

### Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dns-test
spec:
  dnsPolicy: ClusterFirst
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Custom DNS configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
      - 10.96.0.10
    searches:
      - production.svc.cluster.local
  containers:
    - name: test
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
```

Ensure custom nameservers are reachable and correctly configured.

---

## 16. DNS Troubleshooting

Create a temporary debugging Pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Check DNS resolution:

```bash
kubectl exec dns-test -- \
  nslookup kubernetes.default
```

Check Service resolution:

```bash
kubectl exec dns-test -- \
  nslookup backend-service.production.svc.cluster.local
```

Check DNS configuration:

```bash
kubectl exec dns-test -- cat /etc/resolv.conf
```

Check CoreDNS:

```bash
kubectl get pods -n kube-system
kubectl logs -n kube-system \
  -l k8s-app=kube-dns
```

### Common DNS problems

| Problem | Possible reason |
|---|---|
| NXDOMAIN | Service name or namespace is wrong |
| Timeout | CoreDNS unreachable or DNS traffic blocked |
| SERVFAIL | DNS server or forwarding problem |
| Intermittent resolution | Network, CoreDNS capacity, or upstream DNS issue |
| External domain fails | Upstream DNS forwarding issue |

### Production troubleshooting sequence

1. Check the Service exists.
2. Check the namespace.
3. Verify the DNS name.
4. Inspect `/etc/resolv.conf`.
5. Check CoreDNS Pods.
6. Check CoreDNS logs.
7. Check CoreDNS Service endpoints.
8. Verify network connectivity to DNS.
9. Check upstream DNS forwarding.

---

## 17. kube-proxy

kube-proxy is a Kubernetes networking component commonly responsible for implementing Service virtual IP behavior on nodes.

It watches Service and EndpointSlice information and configures the node's networking rules.

Depending on configuration, kube-proxy can use:

- iptables
- IPVS
- nftables

Some CNI and eBPF-based implementations replace kube-proxy functionality.

### kube-proxy architecture

```text
Kubernetes API Server
        |
        |
 Services / EndpointSlices
        |
        v
    kube-proxy
        |
        |
-----------------------
|          |          |
iptables  IPVS      nftables
        |
        v
Service Traffic
        |
        v
Backend Pods
```

### Check kube-proxy Pods

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-proxy
```

### Check kube-proxy logs

```bash
kubectl logs -n kube-system \
  -l k8s-app=kube-proxy
```

### Inspect kube-proxy ConfigMap

```bash
kubectl get configmap kube-proxy \
  -n kube-system -o yaml
```

### Inspect iptables rules

On a node with suitable privileges:

```bash
iptables-save
```

Inspect IPVS:

```bash
ipvsadm -Ln
```

The appropriate command depends on the configured mode and node operating system.

---

## 18. iptables vs IPVS vs nftables

| Feature | iptables | IPVS | nftables |
|---|---|---|---|
| Technology | Netfilter rules | Kernel virtual server | Netfilter framework |
| Service handling | Rule-based | Virtual server-based | Rule-based |
| Typical use | Common kube-proxy mode | Alternative kube-proxy mode | Supported newer kube-proxy mode |
| Inspection | iptables-save | ipvsadm | nft list ruleset |

Do not assume every Kubernetes cluster uses the same proxy mode.

---

## 19. Service Traffic Flow

Example:

```text
Application Pod
       |
       v
backend-service
       |
       v
ClusterIP
       |
       v
Service forwarding implementation
       |
       v
Endpoint
       |
       v
Backend Pod
```

The Service's virtual IP is not normally an application process listening on a network interface. The networking implementation handles the virtual IP behavior.

### Inspect Service

```bash
kubectl describe svc backend-service
```

### Inspect EndpointSlices

```bash
kubectl get endpointslices \
  -l kubernetes.io/service-name=backend-service
```

---

## 20. Session Affinity

By default, Service traffic can be distributed across available endpoints.

A Service can use ClientIP session affinity.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  sessionAffinity: ClientIP
  ports:
    - port: 80
      targetPort: 8080
```

This attempts to send traffic from the same client IP to the same endpoint for the configured affinity period.

**Production note:** Session affinity is not a substitute for application-level session management. Prefer stateless application design where possible.

---

## 21. Internal Traffic Policy

A Service can influence whether traffic is routed to local-node endpoints or endpoints across the cluster.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: local-backend
spec:
  internalTrafficPolicy: Local
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

With `Local`, traffic sent to the Service from a node is routed only to endpoints on that same node.

If no local endpoint exists, traffic can fail.

---

## 22. External Traffic Policy

For NodePort and LoadBalancer Services, `externalTrafficPolicy` controls how externally received traffic is handled.

### Cluster

```yaml
externalTrafficPolicy: Cluster
```

- Traffic may be forwarded to endpoints on other nodes.
- Source IP may be obscured depending on the network path.

### Local

```yaml
externalTrafficPolicy: Local
```

- Traffic is routed only to local-node endpoints.
- Can preserve the original source IP in supported configurations.
- Nodes without local endpoints may not serve traffic.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

**Production considerations:**

- Understand source IP preservation.
- Check load-balancer health checks.
- Ensure endpoint placement is suitable.
- Avoid unexpected traffic loss on nodes without local endpoints.

---

## 23. Kubernetes Networking Traffic Flow

### Pod-to-Pod on the same node

```text
Pod A
  |
  v
Node Network
  |
  v
Pod B
```

### Pod-to-Pod across nodes

```text
Worker Node 1
    |
  Pod A
    |
    v
CNI Routing / Overlay
    |
    v
Worker Node 2
    |
  Pod B
```

### Pod-to-Service

```text
Application Pod
      |
      v
Service DNS
      |
      v
ClusterIP
      |
      v
Service Networking
      |
      v
Backend Pod
```

### External-to-Pod

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Ingress / Gateway
   |
   v
Service
   |
   v
Pod
```

---

## 24. Production DNS and Service Best Practices

- Use Service DNS names rather than Pod IPs.
- Use readiness probes to control endpoint eligibility.
- Monitor CoreDNS latency and error rates.
- Ensure DNS capacity for application scale.
- Avoid excessive DNS queries.
- Understand DNS search domains and `ndots`.
- Use headless Services for appropriate StatefulSet workloads.
- Validate DNS forwarding to external systems.
- Avoid making application availability depend on a single manually configured DNS endpoint.
- Test Service and DNS behavior during Pod and node failures.
