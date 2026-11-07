## 21. RBAC vs NetworkPolicy

RBAC and NetworkPolicy solve different security problems.

| Feature | RBAC | NetworkPolicy |
|---|---|---|
| Purpose | Controls Kubernetes API access | Controls Pod network traffic |
| Controls | Users, groups, ServiceAccounts | Pod ingress and egress |
| Example | Who can delete Pods? | Which Pods can connect to an API? |
| Scope | Kubernetes API authorization | Network communication |
| Enforcement | Kubernetes API server | Network plugin supporting NetworkPolicy |

### RBAC example

Allow an application identity to read Pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

### NetworkPolicy example

Allow traffic to an application only from Pods with a specific label:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: development
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

**Important:**
- RBAC does not control application network traffic.
- NetworkPolicy does not control Kubernetes API permissions.
- Use both for defense in depth.
- NetworkPolicy enforcement depends on the cluster's networking implementation.

---

## 22. RBAC vs Pod Security Admission

Pod Security Admission (PSA) controls the security posture of Pods.

RBAC controls who can perform actions against Kubernetes API resources.

| Feature | RBAC | Pod Security Admission |
|---|---|---|
| Purpose | Authorization | Pod security enforcement |
| Controls | API actions | Pod security settings |
| Example | Who can create Pods? | Can a Pod run privileged? |
| Configuration | Roles and bindings | Namespace labels |
| Enforcement | API authorization | Admission control |

### Pod Security Admission example

Apply the restricted security standard to a namespace:

```bash
kubectl label namespace development \
  pod-security.kubernetes.io/enforce=restricted
```

You can also configure warnings and audit levels:

```bash
kubectl label namespace development \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

**Important:** PSA does not replace RBAC. A user may have permission to create Pods but still be prevented from creating a Pod that violates the namespace's security policy.

---

## 23. Production RBAC Best Practices

### 23.1 Follow least privilege

Grant only the permissions required by a user or application.

Avoid:

```yaml
apiGroups: ["*"]
resources: ["*"]
verbs: ["*"]
```

Prefer:

```yaml
apiGroups: [""]
resources: ["pods"]
verbs: ["get", "list", "watch"]
```

### 23.2 Separate human and application identities

Use individual user identities for engineers and dedicated ServiceAccounts for workloads.

Do not reuse one ServiceAccount across unrelated applications.

### 23.3 Avoid cluster-admin for applications

Applications should rarely need cluster-wide administrative permissions.

Prefer namespace-scoped Roles and RoleBindings.

### 23.4 Restrict Secret access

Avoid granting broad access to Secrets.

```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

Only add this permission when the workload genuinely needs it.

### 23.5 Avoid unnecessary Pod execution permissions

Permissions such as `pods/exec` can allow interactive access to containers.

Grant them only to trusted operators.

### 23.6 Use resourceNames where possible

For resources with predictable names, restrict access to specific objects.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["application-config"]
    verbs: ["get", "update", "patch"]
```

**Note:** `resourceNames` has limitations for list and watch operations. Those requests generally need a matching `metadata.name` field selector.

### 23.7 Keep RBAC in Git

Use version control to track:
- Permission changes.
- Role definitions.
- RoleBindings.
- ServiceAccount assignments.
- Approval history.

### 23.8 Review permissions regularly

Remove:
- Unused users.
- Unused ServiceAccounts.
- Old RoleBindings.
- Excessive permissions.
- Temporary access that is no longer needed.

### 23.9 Use audit logging

Kubernetes audit logs help identify who performed an API action, when it happened, and which resource was involved.

### 23.10 Test before deployment

Use:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:development:app-sa \
  -n development
```

Test both allowed and denied actions.

---

## 24. Real Production Scenario: Developer Access

**Requirement:**

A developer should be able to:
- View Pods.
- View Deployments.
- View Services.
- View ConfigMaps.
- Restart Deployments by updating their Pod template.
- Work only in the `development` namespace.

The developer should not:
- Access Secrets.
- Create Roles or RoleBindings.
- Modify production resources.
- Access cluster-wide resources.

### Step 1: Create the namespace

```bash
kubectl create namespace development
```

### Step 2: Create the Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: development
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - configmaps
    verbs:
      - get
      - list
      - watch

  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - update
      - patch
```

### Step 3: Create the RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### Step 4: Apply the manifests

```bash
kubectl apply -f developer-role.yaml
kubectl apply -f developer-binding.yaml
```

### Step 5: Validate permissions

```bash
kubectl auth can-i get pods \
  --as=developer@example.com \
  -n development
```

Expected:

```text
yes
```

Check Secret access:

```bash
kubectl auth can-i get secrets \
  --as=developer@example.com \
  -n development
```

Expected:

```text
no
```

Check production access:

```bash
kubectl auth can-i get pods \
  --as=developer@example.com \
  -n production
```

Expected:

```text
no
```

**Production note:** Updating a Deployment can trigger a rollout, but this role does not grant permission to create or modify every other workload resource. Ensure the granted permissions match the actual operational workflow.

---

## 25. Real Production Scenario: Monitoring ServiceAccount

**Requirement:**

Prometheus or another monitoring application needs read-only access to discover Pods and Services.

### Step 1: Create a namespace

```bash
kubectl create namespace monitoring
```

### Step 2: Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: monitoring
```

### Step 3: Create a ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
  - apiGroups: [""]
    resources:
      - nodes
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
```

### Step 4: Create a ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-reader-binding
subjects:
  - kind: ServiceAccount
    name: prometheus
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

### Step 5: Apply manifests

```bash
kubectl apply -f prometheus-serviceaccount.yaml
kubectl apply -f prometheus-clusterrole.yaml
kubectl apply -f prometheus-clusterrolebinding.yaml
```

### Step 6: Validate

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:prometheus \
  -n default
```

Expected:

```text
yes
```

**Production note:** Monitoring integrations often need additional permissions depending on their discovery and metrics collection configuration. Grant only the permissions required by the actual deployment.

---

## 26. Real Production Scenario: CI/CD Deployment Access

**Requirement:**

A CI/CD system should deploy applications into the `development` namespace.

It needs permission to:
- Read Deployments.
- Create and update Deployments.
- Read Services.
- Create and update Services.
- Read ConfigMaps.
- Create and update ConfigMaps.

It should not have cluster-admin access.

### Step 1: Create a namespace

```bash
kubectl create namespace development
```

### Step 2: Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: development
```

### Step 3: Create a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer-role
  namespace: development
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch

  - apiGroups: [""]
    resources:
      - services
      - configmaps
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
```

### Step 4: Create a RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: development
subjects:
  - kind: ServiceAccount
    name: cicd-deployer
    namespace: development
roleRef:
  kind: Role
  name: cicd-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

### Step 5: Validate access

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:development:cicd-deployer \
  -n development
```

Expected:

```text
yes
```

Check cluster-wide access:

```bash
kubectl auth can-i create deployments \
  --as=system:serviceaccount:development:cicd-deployer \
  -n production
```

Expected:

```text
no
```

**Important production considerations:**
- Use short-lived credentials where supported.
- Avoid storing long-lived ServiceAccount tokens in CI/CD secrets.
- Restrict pipeline access to trusted repositories and workflows.
- Separate deployment identities by environment.
- Add permissions for other resources only when the pipeline needs them.

---

## 27. RBAC Hands-on Lab

Practice RBAC using a local Kubernetes cluster such as Minikube or Kind.

### Lab 1: Create a namespace

```bash
kubectl create namespace dev
```

### Lab 2: Create a ServiceAccount

```bash
kubectl create serviceaccount app-sa -n dev
```

### Lab 3: Create a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

### Lab 4: Create a RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: dev
subjects:
  - kind: ServiceAccount
    name: app-sa
    namespace: dev
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Lab 5: Apply manifests

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml
```

### Lab 6: Verify access

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:dev:app-sa \
  -n dev
```

Expected:

```text
yes
```

### Lab 7: Test a denied action

```bash
kubectl auth can-i delete pods \
  --as=system:serviceaccount:dev:app-sa \
  -n dev
```

Expected:

```text
no
```

### Lab 8: Test access in another namespace

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:dev:app-sa \
  -n default
```

Expected:

```text
no
```

### Lab 9: Inspect the binding

```bash
kubectl describe rolebinding pod-reader-binding -n dev
```

### Lab 10: Clean up

```bash
kubectl delete namespace dev
```

---

## 28. RBAC Limitations

RBAC is powerful, but it does not solve every Kubernetes security requirement.

| Requirement | Appropriate control |
|---|---|
| Control API access | RBAC |
| Control Pod network traffic | NetworkPolicy |
| Enforce Pod security standards | Pod Security Admission |
| Manage secrets securely | External secret manager and encryption |
| Restrict resource consumption | ResourceQuota and LimitRange |
| Enforce custom admission rules | Admission controllers or policy engines |
| Verify container images | Image signing and admission policies |
| Protect cluster infrastructure | Node and cloud infrastructure security |

### Important RBAC limitations

- RBAC does not provide network isolation.
- RBAC does not encrypt Secret data.
- RBAC does not validate container images.
- RBAC does not enforce CPU or memory limits.
- RBAC does not replace authentication.
- RBAC does not automatically provide complete tenant isolation.

---

## 29. Kubernetes RBAC Interview Questions

### Q1. What is RBAC in Kubernetes?

RBAC stands for Role-Based Access Control. It controls who can perform which actions on Kubernetes resources.

### Q2. What is the difference between Role and ClusterRole?

- Role grants permissions within a namespace.
- ClusterRole can grant permissions across the cluster or describe permissions for cluster-scoped resources.

### Q3. What is the difference between RoleBinding and ClusterRoleBinding?

- RoleBinding grants permissions within a namespace.
- ClusterRoleBinding grants permissions at cluster scope.

### Q4. Can a RoleBinding reference a ClusterRole?

Yes. A RoleBinding can reference a ClusterRole, but the permissions are granted only within the RoleBinding's namespace.

### Q5. What are Kubernetes RBAC verbs?

Common verbs include:

```text
get
list
watch
create
update
patch
delete
deletecollection
```

### Q6. How do you check whether a user can delete Pods?

```bash
kubectl auth can-i delete pods \
  --as=developer@example.com \
  -n development
```

### Q7. What is a ServiceAccount?

A ServiceAccount provides an identity for an application or workload running inside Kubernetes.

### Q8. What is the default ServiceAccount?

Every namespace normally has a ServiceAccount named `default`. Pods use it when no other ServiceAccount is specified.

### Q9. Does creating a Role grant permissions automatically?

No. A Role must be associated with a user, group, or ServiceAccount through a RoleBinding.

### Q10. What is the difference between authentication and authorization?

- Authentication verifies who the requester is.
- Authorization determines whether the requester can perform the requested action.

### Q11. What is the least privilege principle?

Grant only the minimum permissions needed to perform a task.

### Q12. How do you give a user read-only access to a namespace?

Create a RoleBinding referencing the built-in `view` ClusterRole, or create a custom read-only Role.

### Q13. Can RBAC restrict access to a specific Pod?

RBAC supports `resourceNames` to restrict certain permissions to named resources, subject to limitations on list and watch operations.

### Q14. What happens if no RBAC rule allows an action?

The request is denied when RBAC is the authorization mechanism and no other configured authorizer grants it.

### Q15. Can RBAC grant access to Secrets?

Yes. Permissions can be granted on Secrets, but access should be tightly restricted because Secrets may contain sensitive credentials.

### Q16. What is the purpose of `kubectl auth can-i`?

It checks whether a specified identity is authorized to perform an action on a resource.

### Q17. What is a ClusterRoleBinding?

It binds a ClusterRole to users, groups, or ServiceAccounts at cluster scope.

### Q18. How do you troubleshoot a Forbidden error?

Check the identity, namespace, Role or ClusterRole, binding, API group, resource, and verb. Then validate with `kubectl auth can-i`.

### Q19. Does RBAC control network traffic between Pods?

No. NetworkPolicy is used to control supported Pod network traffic.

### Q20. Why should applications avoid cluster-admin?

Cluster-admin grants extensive permissions. If an application is compromised, those permissions can significantly increase the impact of the compromise.

---

## 30. RBAC Quick Revision

| Component | Meaning |
|---|---|
| Role | Namespace-level permission definition |
| RoleBinding | Grants a Role or ClusterRole within a namespace |
| ClusterRole | Cluster-level or reusable permission definition |
| ClusterRoleBinding | Grants a ClusterRole across the cluster |
| ServiceAccount | Identity for workloads |
| Subject | User, Group, or ServiceAccount |
| API Group | API resource category |
| Resource | Pod, Deployment, Service, Secret, etc. |
| Verb | Action such as get, list, create, or delete |
| resourceNames | Restricts certain permissions to named resources |

### Most important commands

```bash
kubectl auth whoami
kubectl auth can-i get pods
kubectl auth can-i --list -n development
kubectl get roles -A
kubectl get rolebindings -A
kubectl get clusterroles
kubectl get clusterrolebindings
kubectl describe role <role-name> -n <namespace>
kubectl describe rolebinding <binding-name> -n <namespace>
```

### Remember these rules

1. Role defines permissions; it does not grant them by itself.
2. RoleBinding connects permissions to an identity.
3. ClusterRole can define permissions for cluster-scoped resources or reusable permissions.
4. RoleBinding can reference a ClusterRole but grants access only in its namespace.
5. ClusterRoleBinding grants permissions at cluster scope.
6. ServiceAccounts provide workload identities.
7. Kubernetes RBAC follows a deny-by-default model when no authorizer grants access.
8. Always follow least privilege.
9. Test permissions using `kubectl auth can-i`.
10. Keep production RBAC manifests version-controlled and audited.
