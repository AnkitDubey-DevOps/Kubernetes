# Kubernetes Networking – NetworkPolicy and Network Security

## 37. What Is NetworkPolicy?

NetworkPolicy controls network communication to and from selected Pods.

It can restrict:

- Ingress traffic.
- Egress traffic.
- Pod-to-Pod communication.
- Namespace-to-namespace communication.
- Communication to selected IP address ranges.

NetworkPolicy is implemented by the cluster's networking plugin.

**Important:** Creating a NetworkPolicy does not guarantee enforcement unless the installed network implementation supports it.

---

## 38. Default Kubernetes Network Behavior

In a cluster without restrictive NetworkPolicies, Pods can generally communicate with one another.

Example:

```text
Frontend Pod
     |
     v
Backend Pod
     |
     v
Database Pod
```

This is convenient but may be too permissive for production.

A production cluster should define explicit communication requirements.

---

## 39. Default Deny All Ingress

This policy blocks ingress traffic to all Pods in the selected namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

Meaning:

- `podSelector: {}` selects all Pods in the namespace.
- `policyTypes: Ingress` applies ingress isolation.
- No ingress rules means no incoming connections are allowed, subject to NetworkPolicy semantics.

---

## 40. Default Deny All Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

This blocks egress traffic from selected Pods, subject to the network implementation.

**Important:** If you apply default-deny egress, explicitly allow required DNS and application dependencies. Otherwise, applications may stop working.

---

## 41. Allow Frontend to Backend

Requirement:

- Frontend Pods should connect to backend Pods.
- Backend Pods listen on TCP port 8080.
- Other ingress sources should remain blocked if no other policy allows them.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

Traffic:

```text
Frontend Pod
     |
     | TCP 8080
     v
Backend Pod
```

---

## 42. Allow Traffic from Another Namespace

Requirement:

Allow monitoring Pods in the `monitoring` namespace to access metrics on backend Pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

**Important:** The namespaceSelector and podSelector in the same `from` entry are combined with logical AND.

This allows only Pods matching the selected Pod label in the selected namespace.

---

## 43. Allow DNS Egress

When egress is restricted, allow DNS queries to the cluster DNS Pods.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Production note:** DNS Pod labels and DNS architecture can vary. Verify the actual CoreDNS labels and whether your CNI applies policy to DNS traffic as expected.

---

## 44. Allow Egress to an External IP Range

Example: Allow TCP traffic to an approved external network.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
      ports:
        - protocol: TCP
          port: 443
```

`203.0.113.0/24` is an example documentation address range. Replace it with the actual approved destination range.

---

## 45. NetworkPolicy Rules and Important Semantics

| Field | Meaning |
|---|---|
| podSelector | Selects Pods affected by the policy |
| namespaceSelector | Selects namespaces |
| ipBlock | Selects an IP range |
| ingress | Defines allowed incoming traffic |
| egress | Defines allowed outgoing traffic |
| ports | Restricts allowed ports and protocols |
| policyTypes | Defines ingress and/or egress isolation |

### Important behavior

- Multiple policies can select the same Pod.
- Allowed traffic from applicable policies is additive.
- A policy does not create a deny rule that overrides an allow rule in another policy.
- Ingress and egress isolation are independent.
- Policies are namespace-scoped.
- Standard NetworkPolicy does not provide portable FQDN-based egress policies.

---

## 46. NetworkPolicy Troubleshooting

Check policies:

```bash
kubectl get networkpolicy -A
```

Describe a policy:

```bash
kubectl describe networkpolicy allow-frontend-to-backend \
  -n production
```

Check Pod labels:

```bash
kubectl get pods -n production --show-labels
```

Check namespace labels:

```bash
kubectl get namespaces --show-labels
```

Test application connectivity:

```bash
kubectl exec -it frontend-pod -n production -- \
  curl -v http://backend-service:80
```

Troubleshooting sequence:

1. Confirm the CNI supports NetworkPolicy.
2. Check which Pods the policy selects.
3. Verify Pod labels.
4. Verify namespace labels.
5. Check ingress and egress policy types.
6. Verify the destination port.
7. Verify DNS egress.
8. Inspect CNI logs and policy status.
9. Check node firewall and routing rules.

---

## 47. Overlay Networking vs Native Routing

### Overlay networking

An overlay network encapsulates traffic between nodes.

Example:

```text
Worker Node 1
    |
  Pod A
    |
    v
Overlay Encapsulation
    |
    v
Underlay Network
    |
    v
Overlay Decapsulation
    |
    v
Worker Node 2
    |
  Pod B
```

Examples of overlay technologies include VXLAN and Geneve.

Advantages:

- Can simplify Pod networking across nodes.
- Reduces the need for the underlying network to know every Pod route.
- Useful in many cloud and on-premises environments.

Trade-offs:

- Encapsulation overhead.
- MTU planning.
- Additional packet-processing cost.
- More complex packet troubleshooting.

### Native routing

Native routing uses the underlying network to route Pod traffic.

```text
Pod A
  |
  v
Node 1
  |
  v
Underlay Routing
  |
  v
Node 2
  |
  v
Pod B
```

Advantages:

- May avoid overlay encapsulation overhead.
- Can simplify some traffic paths.
- Can integrate with cloud-native routing.

Trade-offs:

- Requires suitable routing integration.
- Needs careful IP address planning.
- Depends on infrastructure capabilities.

---

## 48. VXLAN

VXLAN stands for Virtual Extensible LAN.

It encapsulates Layer 2 Ethernet frames inside UDP packets.

It is commonly used to create overlay networks.

Concept:

```text
Original Pod Packet
       |
       v
VXLAN Encapsulation
       |
       v
UDP Packet
       |
       v
Physical Network
       |
       v
VXLAN Decapsulation
       |
       v
Destination Pod
```

Production considerations:

- Understand the encapsulation overhead.
- Configure a suitable MTU.
- Verify firewall rules for required transport traffic.
- Monitor packet drops and fragmentation.

---

## 49. eBPF Networking

eBPF allows programs to run in the Linux kernel at supported hook points.

Networking implementations can use eBPF for:

- Packet processing.
- Service load balancing.
- NetworkPolicy enforcement.
- Observability.
- Traffic redirection.

Cilium is a popular Kubernetes networking implementation using eBPF.

### eBPF networking architecture

```text
Pod Traffic
    |
    v
Linux Kernel
    |
    v
eBPF Programs
    |
-----------------------
|          |          |
Routing   Policy   Observability
    |
    v
Destination
```

### Production benefits

- Efficient packet processing in suitable workloads.
- Rich network visibility.
- Advanced policy features.
- Potential replacement of kube-proxy functionality.

### Production considerations

- Kernel compatibility.
- CNI compatibility.
- Upgrade planning.
- Operational expertise.
- Observability and troubleshooting tooling.

---

## 50. MTU in Kubernetes Networking

MTU stands for Maximum Transmission Unit.

It defines the maximum packet size that can be transmitted over a network interface without fragmentation at that layer.

Overlay encapsulation reduces the usable MTU for inner packets.

Example:

```text
Underlay MTU: 1500
Overlay Overhead: 50

Possible Pod MTU: 1450
```

The actual overhead depends on the encapsulation method and configuration.

### Common MTU symptoms

- Large requests fail.
- Small requests succeed.
- Connections hang.
- TLS handshakes fail intermittently.
- Cross-node traffic behaves differently from same-node traffic.

### Check interface MTU

```bash
ip link
```

```bash
ip link show eth0
```

### Production best practices

- Validate the MTU across the complete traffic path.
- Account for overlay overhead.
- Avoid inconsistent MTU configurations.
- Test large packet transfers.
- Check cloud network and VPN MTU settings.

---

## 51. Kubernetes Network Security Layers

A production cluster should use multiple layers of security.

```text
             External Traffic
                    |
                    v
             Cloud Firewall
                    |
                    v
              Load Balancer
                    |
                    v
             Ingress / Gateway
                    |
                    v
              Kubernetes Service
                    |
                    v
               NetworkPolicy
                    |
                    v
                  Pod
                    |
                    v
            Application Security
```

Different layers solve different problems.

| Layer | Security responsibility |
|---|---|
| Cloud firewall | Infrastructure-level network filtering |
| Security groups | Cloud network access control |
| Load balancer | External traffic entry and distribution |
| Ingress/Gateway | HTTP routing and TLS handling |
| NetworkPolicy | Pod-level network isolation |
| RBAC | Kubernetes API authorization |
| Pod Security Admission | Pod security configuration |
| Application | Authentication and authorization |

---

## 52. Production Network Security Checklist

- [ ] Apply default-deny policies where appropriate.
- [ ] Explicitly allow required application traffic.
- [ ] Allow DNS when egress is restricted.
- [ ] Restrict access to databases.
- [ ] Restrict management ports.
- [ ] Restrict external egress.
- [ ] Verify CNI policy enforcement.
- [ ] Monitor denied traffic.
- [ ] Review network rules regularly.
- [ ] Use TLS for sensitive communication.
- [ ] Combine network controls with RBAC.
- [ ] Test policies before production rollout.
