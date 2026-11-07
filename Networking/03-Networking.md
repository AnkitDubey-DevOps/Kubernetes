# Kubernetes Networking – Ingress, Gateway API and Load Balancing

## 25. What Is Ingress?

Ingress is a Kubernetes API object that defines HTTP and HTTPS routing rules for services inside a cluster.

It allows external users to access multiple applications using one external entry point.

Example:

```text
                    Internet
                       |
                       v
                Load Balancer
                       |
                       v
               Ingress Controller
                       |
             ---------------------
             |                   |
          /app1                 /app2
             |                   |
             v                   v
       Service A             Service B
             |                   |
          Pods                 Pods
```

### Important distinction

**Ingress is a configuration resource, not the actual traffic-handling implementation.**

An Ingress Controller must be installed to implement Ingress rules.

Popular implementations include:

- NGINX Ingress Controller
- Traefik
- HAProxy
- Kong
- Cloud-provider-specific ingress controllers

Check installed controllers and namespaces because they differ by cluster distribution.

---

## 26. Ingress Example

Requirement:

- `app.example.com` routes to frontend.
- `api.example.com` routes to backend.

### Step 1: Create frontend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: production
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 8080
```

### Step 2: Create backend Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

### Step 3: Create Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: application-ingress
  namespace: production
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Check:

```bash
kubectl get ingress -n production
```

**Note:** This example assumes an installed Ingress Controller that recognizes the `nginx` IngressClass.

---

## 27. Ingress Path Routing

You can route multiple paths on the same hostname.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
spec:
  ingressClassName: nginx
  rules:
    - host: example.com
      http:
        paths:
          - path: /frontend
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80

          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend
                port:
                  number: 80
```

### Path types

| Path Type | Meaning |
|---|---|
| Exact | Exact path match |
| Prefix | Path prefix match |
| ImplementationSpecific | Controller-specific matching behavior |

---

## 28. TLS with Ingress

TLS enables encrypted HTTPS communication.

Kubernetes commonly stores TLS certificates in Secrets.

### Create TLS Secret

```bash
kubectl create secret tls app-tls \
  --cert=tls.crt \
  --key=tls.key \
  -n production
```

### Ingress TLS configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-ingress
  namespace: production
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

### Production TLS considerations

- Use certificates from trusted issuers.
- Automate certificate renewal.
- Monitor expiration.
- Restrict access to TLS Secrets.
- Enforce HTTPS where appropriate.
- Configure secure TLS versions and cipher suites through the chosen controller.
- Understand whether TLS terminates at the load balancer, ingress, gateway, or application.

---

## 29. Ingress Controller Troubleshooting

Check Ingress:

```bash
kubectl get ingress -A
```

Describe Ingress:

```bash
kubectl describe ingress application-ingress \
  -n production
```

Check controller Pods:

```bash
kubectl get pods -A
```

Check controller logs:

```bash
kubectl logs <controller-pod> -n <controller-namespace>
```

Check IngressClass:

```bash
kubectl get ingressclass
```

Check Service backends:

```bash
kubectl get endpointslices -n production
```

Common problems:

| Problem | Possible cause |
|---|---|
| 404 | Host or path does not match |
| 502 | Backend connection failure |
| 503 | No healthy backend endpoints |
| TLS error | Invalid or mismatched certificate |
| External IP pending | Load-balancer integration issue |
| Ingress ignored | Wrong IngressClass or controller configuration |

---

## 30. Gateway API

Gateway API is a Kubernetes networking API for advanced traffic management.

It provides a more expressive and role-oriented model than traditional Ingress.

Main resources:

| Resource | Responsibility |
|---|---|
| GatewayClass | Defines a controller implementation |
| Gateway | Defines traffic entry points and listeners |
| HTTPRoute | Defines HTTP routing rules |
| GRPCRoute | Defines gRPC routing rules |
| TCPRoute | Defines TCP routing rules |
| TLSRoute | Defines TLS routing rules |

Support for individual route types depends on the installed Gateway API implementation and version.

### Gateway API architecture

```text
External Users
      |
      v
 GatewayClass
      |
      v
    Gateway
      |
      v
  HTTPRoute
      |
  -----------
  |         |
Service A  Service B
```

### Example Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: production
spec:
  gatewayClassName: example-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      hostname: app.example.com
      allowedRoutes:
        namespaces:
          from: Same
```

The GatewayClass must exist and be supported by an installed Gateway controller.

### Example HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: frontend-route
  namespace: production
spec:
  parentRefs:
    - name: production-gateway
  hostnames:
    - app.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```

Check resources:

```bash
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
```

---

## 31. Ingress vs Gateway API

| Feature | Ingress | Gateway API |
|---|---|---|
| API model | Simpler HTTP/HTTPS routing | More expressive routing model |
| Main resource | Ingress | Gateway and Routes |
| Ownership model | Often controller-centric | Separates infrastructure and route ownership |
| HTTP routing | Supported | Supported |
| Advanced routing | Often controller-specific | Standardized route resources |
| Protocol support | Mainly HTTP/HTTPS | HTTP, gRPC, TCP and other supported types |
| Role separation | Limited | Stronger role separation |

**Production recommendation:** Use the API model supported by your organization and controller. Gateway API is a strong choice for new environments that need advanced routing and clear infrastructure ownership.

---

## 32. Load Balancing in Kubernetes

Load balancing distributes traffic across multiple application instances.

Kubernetes uses several layers of load balancing.

### Layer 1: External Load Balancer

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Kubernetes Service / Gateway
```

### Layer 2: Service Load Distribution

```text
Service
  |
  |---- Pod 1
  |---- Pod 2
  |---- Pod 3
```

### Layer 3: Ingress or Gateway Routing

```text
Client
  |
  v
Ingress / Gateway
  |
  |---- Frontend Service
  |
  |---- Backend Service
  |
  |---- API Service
```

---

## 33. LoadBalancer Implementation

In cloud environments, the Kubernetes Service controller may integrate with a cloud load balancer.

In bare-metal environments, a separate implementation may be required, such as MetalLB or another supported solution.

Check:

```bash
kubectl get svc -A
```

Look for Services with:

```text
TYPE: LoadBalancer
```

If external IP remains pending, check:

- Cloud controller configuration.
- Load-balancer implementation.
- Network address allocation.
- Cloud permissions.
- Firewall and security-group rules.
- Controller logs.

---

## 34. Health Checks and Readiness

Load balancers and Kubernetes traffic routing depend on healthy endpoints.

### Readiness probe example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.27
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
```

A Pod that is not ready is generally excluded from normal Service endpoint traffic.

**Production recommendation:** Configure readiness probes based on actual application readiness, not just whether the process is running.

---

## 35. External DNS

External DNS automation can synchronize Kubernetes resources with DNS provider records.

Example:

```text
app.example.com
       |
       v
External DNS Record
       |
       v
Load Balancer IP
       |
       v
Gateway / Ingress
       |
       v
Service
```

Common implementations integrate with cloud DNS providers.

**Security considerations:**

- Restrict DNS provider credentials.
- Limit managed DNS zones.
- Use separate permissions for production.
- Monitor record changes.
- Avoid exposing internal services unintentionally.

---

## 36. Production Traffic Management Checklist

- [ ] Use a supported Ingress or Gateway controller.
- [ ] Define ownership of Gateway and route resources.
- [ ] Configure TLS correctly.
- [ ] Automate certificate renewal.
- [ ] Configure readiness probes.
- [ ] Validate backend endpoints.
- [ ] Configure load-balancer health checks.
- [ ] Restrict public exposure.
- [ ] Monitor 4xx and 5xx responses.
- [ ] Monitor connection counts and latency.
- [ ] Test failover.
- [ ] Validate DNS records.
