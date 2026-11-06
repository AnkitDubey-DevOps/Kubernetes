
# Kubernetes Ingress and Egress - Deep Dive Notes

## 1. Introduction

In Kubernetes, **Ingress and Egress** describe the direction of network traffic.

- **Ingress:** Incoming traffic coming into an application or Pod.
- **Egress:** Outgoing traffic going from an application or Pod to another destination.

### Simple Example

Imagine you have a web application running inside a Kubernetes cluster.

- A user opens your website → Ingress traffic.
- Your application connects to a database or external API → Egress traffic.

**Remember:**
- Ingress = Incoming traffic.
- Egress = Outgoing traffic.

---

# PART 1: KUBERNETES INGRESS

## 2. What is Kubernetes Ingress?

Kubernetes Ingress is an API resource that manages external HTTP and HTTPS access to Services inside a Kubernetes cluster.

It allows users to access applications using:

- Hostnames (example.com).
- URL paths (/app, /api).
- HTTP and HTTPS.
- TLS/SSL configuration.

**Important:** Creating an Ingress resource alone does not make it work. You need an Ingress Controller to implement its rules.

### Basic Traffic Flow

```text
User / Browser
      |
      v
   Internet
      |
      v
Ingress Controller
      |
      v
Kubernetes Service
      |
      v
      Pod
```

---

## 3. What is an Ingress Controller?

An Ingress Controller is a component that watches Ingress resources and manages incoming traffic according to their rules.

It acts as the entry point for external HTTP/HTTPS traffic.

### Popular Ingress Controllers

- NGINX Ingress Controller
- Traefik
- HAProxy
- Kong
- AWS Load Balancer Controller

**Remember:**

- Ingress = Rules for routing traffic.
- Ingress Controller = Component that implements those rules.

---

## 4. How Ingress Works

Suppose you have two applications:

- Frontend application.
- Backend application.

You want users to access both through one domain.

Example:

```text
example.com/app  -> Frontend Service
example.com/api  -> Backend Service
```

The Ingress Controller receives the request and routes it to the correct Kubernetes Service.

### Traffic Flow

```text
                 User
                   |
                   v
             example.com
                   |
                   v
           Ingress Controller
                   |
             Ingress Rules
               /       \
              /         \
             v           v
       /app path       /api path
           |               |
           v               v
    Frontend Service  Backend Service
           |               |
           v               v
      Frontend Pod     Backend Pod
```

---

## 5. Ingress YAML Example

Let's create two applications and route traffic using Ingress.

### Step 1: Create Frontend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```

### Step 2: Create Backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

### Step 3: Create Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /app
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

**Explanation:**

- `/app` routes traffic to the frontend Service.
- `/api` routes traffic to the backend Service.
- `ingressClassName` identifies the Ingress Controller class.
- `Prefix` matches the specified path and its subpaths.

---

## 6. Ingress Path Types

Kubernetes supports three path types.

| Path Type | Meaning |
|---|---|
| `Exact` | Matches the exact URL path. |
| `Prefix` | Matches the URL path based on its prefix. |
| `ImplementationSpecific` | Matching behavior depends on the Ingress Controller. |

### Example

```yaml
paths:
  - path: /app
    pathType: Exact
```

This matches `/app`, but not `/app/home`.

```yaml
paths:
  - path: /app
    pathType: Prefix
```

This can match:

- `/app`
- `/app/home`
- `/app/dashboard`

---

## 7. Host-Based Routing

Host-based routing sends requests to different Services based on the domain name.

### Example

```text
frontend.example.com -> Frontend Service

backend.example.com  -> Backend Service
```

### YAML Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: frontend.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: backend.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

**Remember:** Host-based routing uses domain names to decide where traffic should go.

---

## 8. Path-Based vs Host-Based Routing

| Path-Based Routing | Host-Based Routing |
|---|---|
| Routes traffic based on URL paths. | Routes traffic based on domain names. |
| Uses paths like `/app` and `/api`. | Uses hosts like `app.example.com`. |
| Can use one domain for multiple applications. | Can use multiple domains for different applications. |

---

## 9. Ingress TLS / SSL Configuration

TLS is used to secure communication between users and applications through HTTPS.

Instead of HTTP, users access:

```text
https://example.com
```

Kubernetes Ingress can reference a TLS Secret containing the certificate and private key.

### Step 1: Create TLS Secret

```bash
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

### Step 2: Configure Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
spec:
  ingressClassName: nginx

  tls:
    - hosts:
        - example.com
      secretName: my-tls-secret

  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

**Important:** The Ingress Controller must support TLS configuration and be configured to use the referenced certificate.

---

## 10. What is ingressClassName?

`ingressClassName` tells Kubernetes which Ingress Controller should handle an Ingress resource.

Example:

```yaml
spec:
  ingressClassName: nginx
```

You can check available Ingress Classes using:

```bash
kubectl get ingressclass
```

---

## 11. Default Backend

A default backend handles requests that do not match any configured Ingress rule.

It can be configured using the `defaultBackend` field.

Example:

```yaml
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
```

---

## 12. What are Ingress Annotations?

Annotations are controller-specific settings used to customize Ingress behavior.

Depending on the controller, annotations can configure:

- Request size limits.
- Timeouts.
- Redirects.
- Authentication.
- Rate limiting.
- Rewrite behavior.

Example:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
```

**Note:** Annotation names and supported features depend on the Ingress Controller.

---

## 13. Ingress vs Service Types

Kubernetes provides different Service types for exposing applications.

| Service Type | Purpose |
|---|---|
| ClusterIP | Exposes a Service inside the cluster. |
| NodePort | Exposes a Service through a port on each Node. |
| LoadBalancer | Exposes a Service through an external load balancer when supported. |
| ExternalName | Maps a Service to an external DNS name. |

### Ingress vs LoadBalancer

| Ingress | LoadBalancer |
|---|---|
| Routes HTTP/HTTPS traffic using rules. | Exposes a Service through an external load balancer. |
| Can route multiple applications through one entry point. | Usually creates or uses an external load balancer for each Service. |
| Requires an Ingress Controller. | Requires load balancer support from the environment. |

**Important:** Ingress usually routes traffic to Services, commonly ClusterIP Services.

---

# PART 2: KUBERNETES EGRESS

## 14. What is Egress?

Egress means outgoing network traffic from a Pod to another destination.

A Pod may communicate with:

- Another Pod.
- Another Service.
- An external API.
- An external database.
- The internet.
- A DNS server.

### Example

Suppose your application needs to connect to an external payment API.

```text
Application Pod
       |
       | Outgoing Request
       v
 Kubernetes Network
       |
       v
   Internet
       |
       v
 Payment API
```

This outgoing traffic is called Egress traffic.

---

## 15. What is Egress Traffic Control?

Egress traffic control determines which destinations a Pod is allowed to communicate with.

For example, you may want:

- Frontend Pods to access backend Pods.
- Backend Pods to access a database.
- Only selected Pods to access external APIs.
- Application Pods to access DNS.

Kubernetes Network Policies can control Pod egress traffic when the cluster's networking implementation supports them.

---

## 16. What is Network Policy?

A NetworkPolicy is a Kubernetes resource used to control network communication involving Pods.

It can control:

- Ingress traffic to Pods.
- Egress traffic from Pods.

**Important:** NetworkPolicy does not work automatically in every Kubernetes networking setup. Your CNI (Container Network Interface) plugin must support NetworkPolicy enforcement.

Examples of CNI implementations with NetworkPolicy support include:

- Calico
- Cilium
- Antrea

---

## 17. Default Network Policy Behavior

By default, Pods are generally allowed to communicate with other Pods and network destinations, unless another networking rule restricts them.

Network Policies can change this behavior.

### Important Rules

1. If no NetworkPolicy selects a Pod for ingress isolation, its ingress traffic is not restricted by NetworkPolicy.
2. If no NetworkPolicy selects a Pod for egress isolation, its egress traffic is not restricted by NetworkPolicy.
3. Once a Pod is isolated for ingress or egress, only traffic allowed by applicable policies is permitted.
4. Multiple NetworkPolicies are additive: traffic allowed by any applicable policy is allowed.

---

## 18. Ingress and Egress in NetworkPolicy

NetworkPolicy has two important policy directions.

| Policy Type | Meaning |
|---|---|
| `Ingress` | Controls incoming traffic to selected Pods. |
| `Egress` | Controls outgoing traffic from selected Pods. |

### Example

```text
Frontend Pod
     |
     | Ingress traffic
     v
Backend Pod
     |
     | Egress traffic
     v
Database Pod
```

- Backend Pod receives incoming traffic from the frontend.
- Backend Pod sends outgoing traffic to the database.

---

## 19. Egress NetworkPolicy Example

Suppose you want a backend Pod to communicate only with a database Pod on port `5432`.

### NetworkPolicy YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-policy
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

### Explanation

- `podSelector` selects backend Pods.
- `policyTypes: Egress` enables outgoing traffic isolation.
- `to` identifies the destination database Pods.
- `port: 5432` allows PostgreSQL traffic.

**Result:** Selected backend Pods can send egress traffic to matching database Pods on TCP port 5432, subject to other applicable policies.

---

## 20. Ingress NetworkPolicy Example

Suppose you want the backend Pod to accept incoming traffic only from frontend Pods on port `8080`.

### NetworkPolicy YAML

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress-policy
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

### Explanation

- Selects backend Pods.
- Allows incoming traffic from frontend Pods.
- Allows traffic on TCP port 8080.
- Other ingress traffic is denied for the isolated Pods unless another applicable policy allows it.

---

## 21. Ingress and Egress Together

You can configure both directions in one NetworkPolicy.

### Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-network-policy
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress
    - Egress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

**Result:**

- Backend accepts incoming traffic from frontend on port 8080.
- Backend sends outgoing traffic to database on port 5432.
- Other traffic is denied for these isolated directions unless another applicable policy allows it.

---

## 22. What are podSelector, namespaceSelector and ipBlock?

These fields identify which Pods or IP addresses can communicate.

### 1. podSelector

Selects Pods using labels.

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

This selects frontend Pods in the same namespace as the NetworkPolicy.

### 2. namespaceSelector

Selects Pods from namespaces matching specified labels.

```yaml
from:
  - namespaceSelector:
      matchLabels:
        environment: production
```

### 3. ipBlock

Allows or denies traffic based on IP address ranges.

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 192.168.1.0/24
```

This allows matching egress traffic to the specified IP range.

**Remember:**

- `podSelector` = Select Pods.
- `namespaceSelector` = Select namespaces.
- `ipBlock` = Select IP ranges.

---

## 23. Allow DNS Egress Traffic

When you restrict egress traffic, DNS queries may also get blocked.

DNS is commonly used by Pods to resolve Service names and external domain names.

You may need to explicitly allow DNS traffic.

### Example

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

**Note:** The actual DNS server location and namespace depend on your cluster configuration. Adjust the selector if your DNS service runs elsewhere.

---

## 24. Important NetworkPolicy Concepts

### 1. Default Deny Ingress

Blocks incoming traffic to selected Pods unless an applicable policy allows it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

### 2. Default Deny Egress

Blocks outgoing traffic from selected Pods unless an applicable policy allows it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

### 3. Default Deny All Traffic

Blocks both ingress and egress traffic for selected Pods unless an applicable policy allows it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Important:** An empty `podSelector: {}` selects all Pods in the NetworkPolicy's namespace, not every Pod in the cluster.

---

## 25. Important Difference: Ingress Resource vs NetworkPolicy Ingress

These two concepts are different.

| Kubernetes Ingress | NetworkPolicy Ingress |
|---|---|
| Routes external HTTP/HTTPS traffic to Services. | Controls incoming network traffic to selected Pods. |
| Requires an Ingress Controller. | Requires CNI NetworkPolicy support. |
| Works with hosts and URL paths. | Works with Pod selectors, namespaces, IP ranges and ports. |
| Used for application exposure. | Used for network security. |

**Remember:** Ingress Resource is for routing. NetworkPolicy Ingress is for traffic control.

---

## 26. What is Gateway API?

Gateway API is a newer Kubernetes API for managing network traffic.

It provides more flexible routing capabilities than traditional Ingress.

Some important Gateway API resources are:

- `GatewayClass` - Defines the gateway controller class.
- `Gateway` - Represents the traffic entry point.
- `HTTPRoute` - Defines HTTP routing rules.
- `GRPCRoute` - Defines gRPC routing rules.

### Ingress vs Gateway API

| Ingress | Gateway API |
|---|---|
| Mainly supports HTTP/HTTPS routing. | Supports more advanced and flexible routing. |
| Uses the Ingress resource. | Uses Gateway and Route resources. |
| Often depends on annotations for advanced features. | Provides more standardized routing capabilities. |

---

## 27. Useful Kubernetes Commands

### Ingress Commands

```bash
# List Ingress resources
kubectl get ingress

# Describe Ingress
kubectl describe ingress <ingress-name>

# List Ingress Classes
kubectl get ingressclass

# Get Ingress YAML
kubectl get ingress <ingress-name> -o yaml

# List Services
kubectl get svc

# Check Pods
kubectl get pods -o wide
```

### NetworkPolicy Commands

```bash
# List NetworkPolicies
kubectl get networkpolicy

# Describe NetworkPolicy
kubectl describe networkpolicy <policy-name>

# Get NetworkPolicy YAML
kubectl get networkpolicy <policy-name> -o yaml

# Apply NetworkPolicy
kubectl apply -f network-policy.yaml

# Delete NetworkPolicy
kubectl delete networkpolicy <policy-name>
```

---

## 28. Troubleshooting Ingress and Egress

### Ingress Troubleshooting

If your application is not accessible, check:

1. Is the Ingress Controller running?
2. Does the Ingress resource exist?
3. Is the correct Ingress Class configured?
4. Are the host and path rules correct?
5. Does the backend Service exist?
6. Are the Service ports correct?
7. Are the Pods healthy and ready?
8. Is DNS pointing to the correct external address?
9. Is the TLS Secret configured correctly?

Useful commands:

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
kubectl get pods -A
kubectl get svc
kubectl get endpoints
```

### Egress Troubleshooting

If a Pod cannot communicate with another destination, check:

1. Is an egress NetworkPolicy blocking traffic?
2. Does the destination IP or DNS name resolve?
3. Is the destination port correct?
4. Is the destination application running?
5. Does the CNI support NetworkPolicy?
6. Is DNS traffic allowed?
7. Are firewall or cloud security rules blocking traffic?

Useful commands:

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <policy-name>
kubectl exec -it <pod-name> -- nslookup example.com
kubectl exec -it <pod-name> -- curl http://example.com
```

---

## 29. Interview Questions

**Q1. What is Kubernetes Ingress?**

Ingress is a Kubernetes API resource used to manage external HTTP and HTTPS access to Services.

**Q2. What is an Ingress Controller?**

It watches Ingress resources and implements their routing rules.

**Q3. What is Egress?**

Egress is outgoing network traffic from a Pod to another destination.

**Q4. What is the difference between Ingress and Egress?**

Ingress means incoming traffic, while Egress means outgoing traffic.

**Q5. Does Ingress work without an Ingress Controller?**

No. An Ingress Controller is required to implement the Ingress rules.

**Q6. What is a NetworkPolicy?**

A NetworkPolicy controls incoming and outgoing network traffic for selected Pods.

**Q7. What is the difference between Ingress and NetworkPolicy Ingress?**

Ingress routes external HTTP/HTTPS traffic to Services. NetworkPolicy Ingress controls incoming traffic to Pods.

**Q8. What is default deny egress?**

It blocks outgoing traffic from selected Pods unless an applicable NetworkPolicy allows it.

**Q9. What is podSelector?**

It selects Pods using labels.

**Q10. What is namespaceSelector?**

It selects namespaces using labels.

**Q11. What is ipBlock?**

It specifies an IP address range using CIDR notation.

**Q12. Does NetworkPolicy work with every CNI plugin?**

No. The CNI plugin must support NetworkPolicy enforcement.

**Q13. Why should DNS traffic be allowed in egress policies?**

Because Pods need DNS to resolve Service names and external domain names.

**Q14. Can multiple NetworkPolicies apply to the same Pod?**

Yes. Their allowed traffic rules are combined additively.

**Q15. What is Gateway API?**

It is a Kubernetes API that provides flexible and advanced traffic routing capabilities.

---
