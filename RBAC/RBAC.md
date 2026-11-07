# Kubernetes RBAC (Role-Based Access Control) – Production-Level Deep Dive

## 1. What is RBAC in Kubernetes?

RBAC stands for *Role-Based Access Control*.

It is a Kubernetes authorization mechanism used to control who can perform which actions on which Kubernetes resources.

It follows the principle of *Least Privilege*:

**Give users, applications, and teams only the permissions they actually need.**

### Real-world examples

- Developer: Can view Pods and Deployments in the development namespace.
- DevOps Engineer: Can create, update, and delete workloads in selected namespaces.
- Monitoring application: Can read Pods, Services, and Endpoints.
- CI/CD pipeline: Can deploy applications but cannot modify RBAC permissions.
- Platform Admin: Can manage cluster-wide resources.

RBAC answers three important questions:

1. **Who?** – User, Group, or ServiceAccount.
2. **What?** – Resources such as Pods, Deployments, Secrets, and Services.
3. **Which actions?** – get, list, watch, create, update, patch, delete.

---

## 2. How Kubernetes RBAC Works

When a request reaches the Kubernetes API Server, Kubernetes processes it through several security stages.

### Request flow

1. User or application sends a request to the API Server.
2. Authentication verifies the identity.
3. Authorization checks whether the identity has permission.
4. RBAC evaluates the applicable Roles and RoleBindings or ClusterRoles and ClusterRoleBindings.
5. Admission control checks additional policies.
6. If all required checks pass, the API Server processes the request.

### Authentication vs Authorization vs Admission Control

| Component | Responsibility | Example |
|---|---|---|
| Authentication | Who are you? | Verify a ServiceAccount token |
| Authorization | What can you do? | Can this identity delete Pods? |
| Admission Control | Is this request allowed? | Reject a privileged container |

**Important:**

- Authentication does not grant permissions.
- RBAC does not authenticate users.
- Admission policies can reject a request even if RBAC allows it.

### Kubernetes authorization modes

- Node
- RBAC
- Webhook
- ABAC (legacy)
- AlwaysAllow
- AlwaysDeny

A Kubernetes cluster can configure multiple authorization modes.

RBAC is the commonly used authorization mechanism in production Kubernetes environments.

---

## 3. Four Main RBAC Objects

Kubernetes RBAC has four primary objects:

1. Role
2. RoleBinding
3. ClusterRole
4. ClusterRoleBinding

### 3.1 Role

A Role defines permissions inside a particular namespace.

Example:

A developer should be able to read Pods in the `development` namespace.

**role.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
```

**Explanation:**

- `apiGroups: [""]` means the core API group.
- `resources: ["pods"]` means permissions apply to Pods.
- `verbs` defines allowed operations.
- `metadata.namespace` determines where this Role applies.

**Important:** A Role does not grant permissions by itself. You need a RoleBinding to associate it with a subject.

### 3.2 RoleBinding

A RoleBinding connects a Role or ClusterRole to a User, Group, or ServiceAccount.

**rolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: development
subjects:
  - kind: User
    name: developer-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**Explanation:**

- `subjects`: Who receives the permissions?
- `roleRef`: Which Role provides the permissions?
- `kind: User`: Refers to an authenticated user identity.
- `roleRef.kind: Role`: References a Role in the same namespace.

Now `developer-user` can read Pods in the `development` namespace.

They cannot automatically read Pods in another namespace.

### 3.3 ClusterRole

A ClusterRole defines permissions that can be used across the cluster.

It can define permissions for:

- Cluster-scoped resources.
- Namespaced resources across namespaces when bound using ClusterRoleBinding.
- Namespaced resources in one namespace when bound using RoleBinding.

**clusterrole.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-viewer
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch
```

This ClusterRole defines read-only access to Pods, Services, and Endpoints.

However, it does not grant access until a binding connects it to a subject.

### 3.4 ClusterRoleBinding

A ClusterRoleBinding grants the permissions in a ClusterRole across the entire cluster.

**clusterrolebinding.yaml**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-pod-viewer
subjects:
  - kind: User
    name: monitoring-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Now `monitoring-user` can read Pods, Services, and Endpoints across all namespaces.

### Role vs ClusterRole

| Feature | Role | ClusterRole |
|---|---|---|
| Definition scope | Namespace | Cluster |
| Namespaced resources | Yes | Yes |
| Cluster-scoped resources | No | Yes |
| Used with RoleBinding | Yes | Yes |
| Used with ClusterRoleBinding | No | Yes |
| Typical use | Namespace access | Reusable or cluster-wide permissions |

### RoleBinding vs ClusterRoleBinding

| Feature | RoleBinding | ClusterRoleBinding |
|---|---|---|
| References Role | Yes | No |
| References ClusterRole | Yes | Yes |
| Permission scope | One namespace | Cluster-wide |
| Typical use | Team access | Cluster-wide monitoring |

**Important production concept:**

A ClusterRole does not automatically mean cluster-wide access.

Its binding determines the effective scope.

For example, a ClusterRole containing Pod permissions can be attached through a RoleBinding in `development`. The subject then receives those permissions only in `development`.

---

## 4. Understanding RBAC API Groups

Kubernetes resources belong to API groups.

The `apiGroups` field identifies the API group of the resource.

| Resource | API Group |
|---|---|
| Pods | `""` |
| Services | `""` |
| ConfigMaps | `""` |
| Secrets | `""` |
| Namespaces | `""` |
| Deployments | `apps` |
| StatefulSets | `apps` |
| DaemonSets | `apps` |
| Jobs | `batch` |
| CronJobs | `batch` |
| Ingress | `networking.k8s.io` |
| NetworkPolicy | `networking.k8s.io` |
| Roles | `rbac.authorization.k8s.io` |
| RoleBindings | `rbac.authorization.k8s.io` |
| HorizontalPodAutoscaler | `autoscaling` |
| PodDisruptionBudget | `policy` |
| PersistentVolumeClaims | `""` |
| PersistentVolumes | `""` |

### Examples

To read Deployments:

```yaml
apiGroups: ["apps"]
resources: ["deployments"]
verbs: ["get", "list", "watch"]
```

To read Pods:

```yaml
apiGroups: [""]
resources: ["pods"]
verbs: ["get", "list", "watch"]
```

To read Ingress resources:

```yaml
apiGroups: ["networking.k8s.io"]
resources: ["ingresses"]
verbs: ["get", "list", "watch"]
```

### How to identify API groups?

```bash
kubectl api-resources
```

Display API groups:

```bash
kubectl api-resources -o wide
```

Search for a specific resource:

```bash
kubectl api-resources | grep deployment
```

---

## 5. Understanding RBAC Verbs

Verbs define what operations an identity can perform.

| Verb | Meaning |
|---|---|
| get | Read one specific resource |
| list | List resources |
| watch | Watch resource changes |
| create | Create resources |
| update | Replace or update an existing resource |
| patch | Apply partial updates |
| delete | Delete a resource |
| deletecollection | Delete multiple resources |
| bind | Bind roles to subjects |
| escalate | Grant permissions beyond your own |
| impersonate | Act as another identity |

### Read-only permissions

```yaml
verbs:
  - get
  - list
  - watch
```

### Deployment management permissions

```yaml
verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
```

### Important: get vs list vs watch

- `get`: Retrieve an individual object.
- `list`: Retrieve a collection of objects.
- `watch`: Receive notifications about changes to objects.

**Production note:** Applications that continuously monitor Kubernetes resources commonly require all three permissions.

---

## 6. Resource-Level Permissions and Subresources

You can restrict access to specific resource types.

Example: Allow reading only Pods and Services.

```yaml
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
    verbs:
      - get
      - list
      - watch
```

You can also restrict access to specific subresources.

### Common subresources

- `pods/log`
- `pods/exec`
- `pods/portforward`
- `pods/attach`
- `deployments/scale`
- `pods/status`

Example: Allow reading Pod logs.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-log-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources:
      - pods
      - pods/log
    verbs:
      - get
      - list
```

### Production security warning

Permissions for these resources require special attention:

- `secrets`
- `pods/exec`
- `pods/attach`
- `pods/portforward`
- `serviceaccounts/token`
- `roles`
- `rolebindings`

For example:

- Access to `pods/exec` may allow someone to execute commands inside application containers.
- Access to Secrets may expose passwords, API keys, and certificates.
- Access to RoleBindings may allow someone to grant permissions to other identities.

Do not grant these permissions casually.

---

## 7. Resource Names – Restrict Access to Specific Objects

RBAC supports `resourceNames` to restrict certain operations to specific named resources.

Example: Allow access only to a ConfigMap named `app-config`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-config-reader
  namespace: development
rules:
  - apiGroups: [""]
    resources:
      - configmaps
    resourceNames:
      - app-config
    verbs:
      - get
      - update
      - patch
```

This subject cannot use these permissions to access other ConfigMaps.

**Important limitations:**

- `resourceNames` is not a general-purpose filtering mechanism for every operation.
- It cannot restrict `create` by resource name because the object name may not be known at authorization time.
- A `list` or `watch` request generally needs a matching `metadata.name` field selector to be restricted by `resourceNames`.

Example:

```bash
kubectl get configmaps \
  --field-selector metadata.name=app-config \
  -n development
```

---

## 8. ServiceAccounts – RBAC for Applications

ServiceAccounts provide identities for applications and workloads running inside Kubernetes.

Examples:

- Monitoring agents
- CI/CD deployment agents
- External Secrets controllers
- Operators
- Automation tools
- Custom Kubernetes applications

A Pod can use a ServiceAccount to authenticate to the Kubernetes API Server.

### Create a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: monitoring
```

### Create a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-reader
  namespace: monitoring
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - endpoints
    verbs:
      - get
      - list
      - watch
```

### Create a RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-reader-binding
  namespace: monitoring
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: monitoring
roleRef:
  kind: Role
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

### Attach ServiceAccount to a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-agent
  namespace: monitoring
spec:
  serviceAccountName: monitoring-sa
  containers:
    - name: agent
      image: nginx:1.29
```

The Pod uses `monitoring-sa` as its Kubernetes identity.

### Verify ServiceAccount

```bash
kubectl get serviceaccounts -n monitoring
```

Describe it:

```bash
kubectl describe serviceaccount monitoring-sa -n monitoring
```

### Production best practice

Set:

```yaml
automountServiceAccountToken: false
```

when a workload does not need to communicate with the Kubernetes API.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: application
  namespace: production
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false
  containers:
    - name: app
      image: nginx:1.29
```

This reduces unnecessary exposure of Kubernetes API credentials.

If the application needs API access, configure the required identity and token mounting deliberately.

---

## 9. ServiceAccount Tokens – Production-Level Understanding

Modern Kubernetes uses short-lived, rotating ServiceAccount tokens through the TokenRequest API.

The projected ServiceAccount token is commonly mounted inside a Pod.

Typical path:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

Other projected files may include:

```text
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
/var/run/secrets/kubernetes.io/serviceaccount/namespace
```

Kubernetes can rotate projected tokens automatically.

### Important security points

- Do not hardcode ServiceAccount tokens in application code.
- Do not store long-lived bearer tokens in Git.
- Avoid manually creating legacy long-lived ServiceAccount token Secrets.
- Use short-lived tokens wherever possible.
- Disable token automounting when API access is unnecessary.
- Restrict permissions assigned to the ServiceAccount.
- Protect access to Pods that use privileged ServiceAccounts.

### Request a token manually

```bash
kubectl create token monitoring-sa -n monitoring
```

This command requests a token for the ServiceAccount.

Do not expose the resulting token in logs, terminal recordings, or shared documentation.

---

## 10. Production Scenario 1 – Developer Access to a Namespace

### Requirement

A development team should:

- View Pods.
- View Deployments.
- View Services.
- Create Deployments.
- Update Deployments.
- Delete Deployments.
- Not access Secrets.
- Not modify RBAC.
- Not access other namespaces.

### Role

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
      - create
      - update
      - patch
      - delete
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
  - kind: Group
    name: development-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

### Production design

Use groups rather than creating separate RoleBindings for every individual developer.

Your identity provider should authenticate users and provide the appropriate group membership.

The Kubernetes API Server must be configured to receive those group identities from the authentication mechanism.

### Expected access

| Operation | Allowed? |
|---|---|
| Read Pods | Yes |
| Read Services | Yes |
| Read Deployments | Yes |
| Create Deployments | Yes |
| Delete Deployments | Yes |
| Read Secrets | No |
| Modify Roles | No |
| Access production namespace | No, unless separately granted |

---

## 11. Production Scenario 2 – CI/CD Pipeline Access

### Requirement

A CI/CD pipeline should deploy applications into the `production` namespace.

It needs:

- Read Deployments.
- Create Deployments.
- Update Deployments.
- Patch Deployments.
- Read Pods.
- Read Services.
- Read ConfigMaps.
- No access to Secrets.
- No permission to create RBAC resources.
- No cluster-admin access.

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: production
automountServiceAccountToken: false
```

### Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer-role
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources:
      - deployments
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch

  - apiGroups: [""]
    resources:
      - pods
      - services
      - configmaps
    verbs:
      - get
      - list
      - watch
```

### RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: cicd-deployer
    namespace: production
roleRef:
  kind: Role
  name: cicd-deployer-role
  apiGroup: rbac.authorization.k8s.io
```

### Production considerations

1. Do not use the `default` ServiceAccount for deployment automation.
2. Use a dedicated ServiceAccount for each pipeline or trust boundary.
3. Avoid `cluster-admin`.
4. Do not grant Secret access unless strictly required.
5. Restrict the pipeline's ability to modify its own RBAC.
6. Use short-lived credentials and secure authentication.
7. Keep production deployment permissions separate from development permissions.
8. Audit deployment identities and their actions.
9. Restrict who can modify pipeline definitions and deployment credentials.

**Important:** `automountServiceAccountToken: false` prevents automatic token mounting in Pods. If the CI/CD runner itself needs Kubernetes API access, configure an explicit, secure token delivery or external authentication mechanism.

Also, RBAC does not restrict which container images a user deploys or whether a Pod can mount a sensitive Secret. Use admission policies and workload security controls for those requirements.

---

## 12. Production Scenario 3 – Monitoring Application

Monitoring tools such as Prometheus or custom monitoring agents may need to discover Kubernetes resources.

Typical requirements:

- List Pods.
- Watch Pods.
- Read Services.
- Read Endpoints.
- Read Nodes.
- Read Namespaces.

Some monitoring systems need cluster-wide discovery.

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-discovery
rules:
  - apiGroups: [""]
    resources:
      - pods
      - services
      - endpoints
      - nodes
      - namespaces
    verbs:
      - get
      - list
      - watch
```

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-sa
  namespace: monitoring
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-discovery-binding
subjects:
  - kind: ServiceAccount
    name: prometheus-sa
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-discovery
  apiGroup: rbac.authorization.k8s.io
```

### Why ClusterRoleBinding?

Because monitoring needs to discover resources across multiple namespaces.

A RoleBinding would restrict these permissions to the namespace containing the binding.

**Production note:** Only include resources actually required by your monitoring configuration.

For example, if monitoring does not need to read Nodes, remove `nodes` from the rule.
