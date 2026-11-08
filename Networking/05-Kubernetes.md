# Kubernetes Networking – Production Architecture and Troubleshooting

## 53. Production Kubernetes Network Architecture

A typical production cluster may have the following architecture:

```text
                      Internet
                         |
                         v
                Cloud Load Balancer
                         |
                         v
                  Ingress / Gateway
                         |
              -----------------------
              |                     |
              v                     v
        Frontend Service       API Service
              |                     |
          ------------          ------------
          |    |     |          |    |     |
         Pod  Pod   Pod        Pod  Pod   Pod
              |                     |
              -----------------------
                         |
                         v
                    Database
                         |
                         v
                  Persistent Storage
```

Additional components:

- CoreDNS for DNS resolution.
- CNI for Pod networking.
- NetworkPolicy for traffic isolation.
- Monitoring for network observability.
- Cloud firewall and security groups.
- Load balancer health checks.

---

## 54. Multi-Node Kubernetes Networking

Example cluster:

```text
                 Kubernetes Cluster

       Worker Node 1        Worker Node 2
      ----------------     ----------------
      |              |     |              |
      |   Pod A      |     |    Pod C     |
      |   Pod B      |     |    Pod D     |
      |              |     |              |
      ----------------     ----------------
             |                    |
             ----------------------
                      |
                CNI Networking
                      |
                Underlay Network
```

Pods on different nodes should be able to communicate according to the configured network model and policies.

### Validate node connectivity

```bash
kubectl get nodes -o wide
```

### Validate Pod IPs

```bash
kubectl get pods -A -o wide
```

### Validate cross-node traffic

Deploy test Pods on different nodes and test their connectivity using their Service endpoints or Pod IPs.

---

## 55. Kubernetes Networking on AWS

AWS EKS commonly uses the AWS VPC CNI.

The AWS VPC CNI integrates Pod networking with the AWS VPC network.

Depending on configuration, Pods can receive VPC IP addresses from node network interfaces.

Architecture:

```text
                 AWS VPC
                    |
          ---------------------
          |                   |
       Node 1              Node 2
          |                   |
       Pod A               Pod C
       Pod B               Pod D
          |                   |
          ---------------------
                    |
                VPC Routing
```

### Important AWS networking components

| Component | Purpose |
|---|---|
| VPC | Network isolation boundary |
| Subnet | IP allocation range |
| Route Table | Network routing |
| Security Group | Network traffic filtering |
| Network ACL | Subnet-level filtering |
| VPC CNI | Pod network integration |
| Load Balancer | External or internal traffic entry |
| NAT Gateway | Outbound internet access for private subnets |

### Production considerations

- Plan sufficient subnet IP capacity.
- Understand Pod IP allocation.
- Monitor available subnet addresses.
- Use security groups appropriately.
- Restrict public endpoint access.
- Configure private subnet egress.
- Understand prefix delegation where supported.
- Monitor CNI allocation failures.

---

## 56. Kubernetes Networking on Azure

Azure AKS provides several networking models depending on configuration.

Common networking options include:

- Azure CNI.
- Azure CNI Overlay.
- Kubenet in supported legacy or existing environments.

Architecture:

```text
                Azure Virtual Network
                         |
                -------------------
                |                 |
             Node 1            Node 2
                |                 |
              Pods              Pods
                |                 |
                -------------------
                         |
                  Azure Routing
```

### Important Azure networking components

- Virtual Network.
- Subnets.
- Network Security Groups.
- Azure Load Balancer.
- Azure NAT Gateway.
- Azure CNI.
- Private cluster networking.

### Production considerations

- Select the networking model carefully.
- Plan subnet address capacity.
- Understand Pod IP allocation.
- Configure NSGs.
- Monitor IP exhaustion.
- Review outbound connectivity.
- Use private networking where required.

---

## 57. Kubernetes Networking on GCP

GKE uses Google Cloud networking integrations.

Depending on the cluster configuration, Pod networking can use alias IP ranges.

Architecture:

```text
                 Google VPC
                     |
             -----------------
             |               |
          Node 1           Node 2
             |               |
           Pods             Pods
             |               |
             -----------------
                     |
                VPC Routing
```

### Important components

- VPC.
- Subnet secondary IP ranges.
- Firewall rules.
- Cloud Load Balancing.
- Cloud NAT.
- GKE networking configuration.

### Production considerations

- Plan secondary IP ranges.
- Monitor IP capacity.
- Configure firewall rules.
- Understand private cluster access.
- Review outbound connectivity.
- Monitor load-balancer health.

---

## 58. Network Troubleshooting Methodology

When an application cannot communicate, troubleshoot layer by layer.

### Step 1: Check Pod status

```bash
kubectl get pods -n production -o wide
```

Check:

- Pod status.
- Pod IP.
- Node assignment.
- Readiness.

### Step 2: Check Service

```bash
kubectl get svc -n production
```

### Step 3: Check EndpointSlices

```bash
kubectl get endpointslices -n production
```

### Step 4: Check DNS

```bash
kubectl exec -it debug-pod -n production -- \
  nslookup backend-service
```

### Step 5: Check connectivity

```bash
kubectl exec -it debug-pod -n production -- \
  curl -v http://backend-service:80
```

### Step 6: Check NetworkPolicy

```bash
kubectl get networkpolicy -n production
```

### Step 7: Check CNI

```bash
kubectl get pods -n kube-system
```

### Step 8: Check node networking

On the node, where access is authorized:

```bash
ip addr
ip route
ip link
```

### Step 9: Check node firewall

Depending on the node OS and networking implementation:

```bash
iptables-save
```

Or:

```bash
nft list ruleset
```

### Step 10: Check application logs

```bash
kubectl logs <pod-name> -n production
```

---

## 59. Common Kubernetes Networking Issues

| Issue | Possible cause | What to check |
|---|---|---|
| Pod cannot reach another Pod | CNI or routing issue | CNI logs, routes |
| Service not reachable | Wrong selector or no endpoints | Service, EndpointSlices |
| DNS not resolving | CoreDNS issue | CoreDNS Pods and logs |
| External access fails | Load balancer or firewall | Service, LB, security rules |
| Ingress returns 404 | Host or path mismatch | Ingress rules |
| Ingress returns 502 | Backend connectivity issue | Controller logs, Service |
| Ingress returns 503 | No ready endpoints | EndpointSlices, readiness |
| Cross-node traffic fails | Overlay or underlay issue | CNI, routes, MTU |
| Large requests hang | MTU problem | Interface MTU, packet path |
| Egress fails | NetworkPolicy or NAT issue | Egress rules, routing |
| Intermittent connectivity | Endpoint, node, DNS, or capacity issue | Metrics and logs |

---

## 60. Network Debugging Tools

Useful tools include:

| Tool | Purpose |
|---|---|
| curl | HTTP connectivity |
| wget | HTTP testing |
| nslookup | DNS lookup |
| dig | Detailed DNS lookup |
| ping | Basic ICMP reachability |
| traceroute | Route path investigation |
| ip | Interface and route inspection |
| ss | Socket inspection |
| tcpdump | Packet capture |
| iptables | Firewall rule inspection |
| nft | nftables inspection |
| conntrack | Connection tracking inspection |

### Example debugging Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
  namespace: production
spec:
  containers:
    - name: debug
      image: nicolaka/netshoot
      command: ["sleep", "3600"]
  restartPolicy: Never
```

Apply:

```bash
kubectl apply -f network-debug.yaml
```

Open shell:

```bash
kubectl exec -it network-debug -n production -- bash
```

Test DNS:

```bash
dig backend-service.production.svc.cluster.local
```

Test HTTP:

```bash
curl -v http://backend-service
```

Inspect routes:

```bash
ip route
```

Inspect sockets:

```bash
ss -tulpn
```

Capture traffic where permissions and container capabilities allow:

```bash
tcpdump -i any port 8080
```

**Security note:** Debug containers and packet captures can expose sensitive information. Restrict their use and remove temporary debugging resources after investigation.

---

## 61. Network Monitoring and Observability

Production networking needs monitoring for:

- Packet drops.
- Network errors.
- DNS latency.
- Connection failures.
- TCP retransmissions.
- Pod network utilization.
- Node network utilization.
- Load-balancer health.
- Service request latency.
- NetworkPolicy drops.

### Common monitoring solutions

| Tool | Purpose |
|---|---|
| Prometheus | Metrics collection |
| Grafana | Metrics dashboards |
| Cilium Hubble | Network observability for Cilium |
| Calico observability tools | Network and policy visibility |
| tcpdump | Packet inspection |
| Cloud monitoring | Infrastructure network metrics |
| OpenTelemetry | Application and distributed tracing |

### Example monitoring architecture

```text
Pods / Nodes / CNI
        |
        v
    Metrics
        |
        v
   Prometheus
        |
        v
     Grafana
        |
        v
   Alert Manager
        |
        v
   Operations Team
```

### Useful metrics to monitor

- Network receive bytes.
- Network transmit bytes.
- Packet receive errors.
- Packet transmit errors.
- Packet drops.
- TCP retransmissions.
- DNS request latency.
- DNS error rate.
- Load-balancer backend health.
- Service request latency.

---

## 62. Production Network Alerts

Examples of alert conditions:

| Alert | Reason |
|---|---|
| High DNS latency | Applications may experience lookup delays |
| CoreDNS unavailable | Internal service discovery may fail |
| High packet drops | Possible network congestion or misconfiguration |
| Pod IP exhaustion | New Pods may fail to obtain IPs |
| Load balancer unhealthy | External application availability risk |
| High TCP retransmissions | Possible packet loss or network congestion |
| CNI agent unavailable | Pod networking may be affected |
| High network utilization | Potential bandwidth saturation |

Set alert thresholds based on workload baselines and service-level objectives.

---

## 63. Production High Availability

A production networking architecture should avoid single points of failure.

Important practices:

1. Run multiple application replicas.
2. Spread replicas across nodes and zones.
3. Use reliable CNI components.
4. Run CoreDNS with suitable replicas and resources.
5. Use health checks.
6. Configure load-balancer health monitoring.
7. Use PodDisruptionBudgets where appropriate.
8. Monitor node and network failures.
9. Test failover.
10. Maintain a rollback strategy for networking upgrades.

### Example PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: frontend
```

This helps maintain at least two available matching Pods during supported voluntary disruptions, provided enough replicas are available.

---

## 64. Production Network Capacity Planning

Plan for:

- Pod IP capacity.
- Node IP capacity.
- Service IP capacity.
- DNS query volume.
- Bandwidth.
- Connection tracking capacity.
- Load-balancer limits.
- Firewall rule limits.
- NAT gateway capacity.
- Cross-zone traffic costs.

### Important planning questions

1. How many Pods will the cluster run?
2. How many nodes will be added?
3. What is the Pod IP allocation model?
4. How much IP capacity does each node require?
5. How much external traffic is expected?
6. What are the expected peak connections?
7. Will workloads communicate across availability zones?
8. Are there overlapping IP ranges with corporate networks?

---

## 65. Kubernetes Networking Production Best Practices

### Architecture

- Use a CNI appropriate for your environment.
- Plan Pod and Service CIDRs.
- Avoid network range overlap.
- Understand overlay versus native routing.
- Design for multi-node and multi-zone traffic.

### Security

- Apply default-deny NetworkPolicies where appropriate.
- Allow only required application communication.
- Restrict external exposure.
- Use TLS.
- Protect DNS and application dependencies.
- Combine NetworkPolicy with cloud firewall controls.

### Reliability

- Use multiple replicas.
- Configure readiness probes.
- Monitor CoreDNS.
- Monitor CNI health.
- Monitor load balancer health.
- Test node and zone failures.

### Operations

- Maintain network architecture documentation.
- Monitor network metrics.
- Keep CNI upgrades controlled.
- Test NetworkPolicies before rollout.
- Use packet capture for difficult failures.
- Maintain a network troubleshooting runbook.

---

## 66. Kubernetes Networking Interview Questions

### Q1. What is CNI?

CNI stands for Container Network Interface. It is used to configure network connectivity for containers and Pods.

### Q2. What is a Pod IP?

It is the IP address assigned to a Pod for network communication.

### Q3. Can containers in the same Pod communicate through localhost?

Yes. Containers in the same Pod share a network namespace.

### Q4. What is a Kubernetes Service?

A Service provides a stable endpoint for accessing a group of Pods.

### Q5. What is the difference between ClusterIP and NodePort?

ClusterIP exposes a Service internally, while NodePort exposes it through a port on each node.

### Q6. What is LoadBalancer Service?

It requests an external load balancer from supported infrastructure.

### Q7. What is CoreDNS?

CoreDNS provides DNS-based service discovery inside Kubernetes.

### Q8. What is kube-proxy?

It commonly implements Kubernetes Service networking on nodes using supported networking mechanisms.

### Q9. What is an EndpointSlice?

It stores endpoint information associated with Services.

### Q10. What is Ingress?

Ingress is an API resource that defines HTTP and HTTPS routing rules.

### Q11. What is an Ingress Controller?

It implements Ingress rules and processes incoming traffic.

### Q12. What is Gateway API?

It is a Kubernetes API model for managing traffic entry points and routing.

### Q13. What is NetworkPolicy?

It defines rules controlling ingress and egress traffic for selected Pods.

### Q14. Does NetworkPolicy work with every CNI?

No. The CNI must support NetworkPolicy enforcement.

### Q15. What is a headless Service?

A Service with `clusterIP: None`, commonly used for StatefulSets and direct endpoint discovery.

### Q16. What is VXLAN?

VXLAN is an encapsulation technology used to build overlay networks.

### Q17. What is eBPF?

eBPF enables programs to run in the Linux kernel and can be used for networking, security, and observability.

### Q18. What is MTU?

MTU is the maximum packet size supported by a network interface at a given layer.

### Q19. Why does DNS fail inside Kubernetes?

Possible reasons include CoreDNS issues, incorrect Service names, DNS network restrictions, or upstream DNS problems.

### Q20. How do you troubleshoot a Service that is not reachable?

Check the Service selector, Pod labels, EndpointSlices, readiness, ports, NetworkPolicies, DNS, and CNI connectivity.

### Q21. How do you check Pod IPs?

```bash
kubectl get pods -A -o wide
```

### Q22. How do you check DNS?

```bash
kubectl exec -it debug-pod -- \
  nslookup kubernetes.default
```

### Q23. How do you check NetworkPolicies?

```bash
kubectl get networkpolicy -A
```

### Q24. What is the difference between Ingress and LoadBalancer?

LoadBalancer exposes a Service using an external load-balancing implementation. Ingress defines HTTP/HTTPS routing rules and requires a controller.

### Q25. What causes a 503 from Ingress?

Common causes include missing backend endpoints, unready Pods, incorrect Service configuration, or controller-to-backend connectivity problems.

### Q26. What causes a 502 from Ingress?

Common causes include backend connection failures, incorrect ports, application failures, or proxy configuration issues.

### Q27. How do you monitor Kubernetes networking?

Use Prometheus, Grafana, CNI observability, cloud network monitoring, logs, and packet analysis.

### Q28. What is the difference between overlay and native routing?

Overlay networking encapsulates traffic between nodes, while native routing uses the underlying network to route Pod traffic.

### Q29. Why is MTU important in Kubernetes?

Incorrect MTU settings can cause packet fragmentation, packet drops, and failures with larger network requests.

### Q30. How do you secure Kubernetes networking?

Use NetworkPolicies, cloud firewall rules, TLS, restricted external exposure, least privilege, and network monitoring.

---

## 67. Final Kubernetes Networking Revision

| Concept | Purpose |
|---|---|
| CNI | Pod network configuration |
| Pod IP | Pod communication |
| Pod CIDR | Pod address allocation |
| Service | Stable application endpoint |
| ClusterIP | Internal Service access |
| NodePort | Node-level Service exposure |
| LoadBalancer | External load-balancer integration |
| ExternalName | External DNS alias |
| CoreDNS | DNS service discovery |
| kube-proxy | Common Service networking implementation |
| EndpointSlice | Service backend endpoint information |
| Ingress | HTTP/HTTPS routing resource |
| Ingress Controller | Implements Ingress |
| Gateway API | Advanced traffic routing API |
| NetworkPolicy | Pod network isolation |
| VXLAN | Overlay network encapsulation |
| eBPF | Kernel-level networking and observability |
| MTU | Maximum packet size |
| Prometheus | Metrics collection |
| Grafana | Monitoring dashboards |

### Most useful commands

```bash
# Pods and Pod IPs
kubectl get pods -A -o wide

# Services
kubectl get svc -A

# EndpointSlices
kubectl get endpointslices -A

# Ingress
kubectl get ingress -A

# Gateway API
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A

# NetworkPolicies
kubectl get networkpolicy -A

# CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Nodes and Pod CIDRs
kubectl get nodes -o wide

# DNS test
kubectl exec -it debug-pod -- nslookup kubernetes.default

# HTTP test
kubectl exec -it debug-pod -- curl -v http://backend-service

# Service details
kubectl describe svc backend-service -n production
```
