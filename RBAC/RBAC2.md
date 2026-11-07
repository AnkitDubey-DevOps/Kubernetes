## 13. Aggregated ClusterRoles

**Aggregated ClusterRoles** allow Kubernetes to combine permissions from multiple ClusterRoles into one ClusterRole.

They are useful for extending built-in roles without directly modifying them.

### Example

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-view-role
  labels:
    rbac.example.com/aggregate-to-view: "true"
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
```

A ClusterRole can aggregate roles using an aggregation rule:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: custom-aggregated-view
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-view: "true"
rules: []
```

**Important points:**
- Kubernetes automatically manages the aggregated role's rules.
- Labels determine which ClusterRoles are included.
- Useful for extending default user-facing roles.
- Avoid manually editing controller-managed aggregated rules.

---

## 14. Built-in ClusterRoles

Kubernetes provides predefined ClusterRoles for common administrative tasks.

| ClusterRole | Purpose |
|---|---|
| `cluster-admin` | Full access across the cluster |
| `admin` | Administrative access within a namespace |
| `edit` | Read and modify most namespace resources |
| `view` | Read-only access to most namespace resources |

### Example: Give a user read-only access

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-access
  namespace: development
subjects:
  - kind: User
    name: developer@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

**Production considerations:**
- Avoid assigning `cluster-admin` to normal users.
- Prefer namespace-scoped RoleBindings.
- Review built-in roles before granting access.
- Remember that `edit` can include powerful permissions, such as creating workloads that use ServiceAccounts.

---

## 15. Dangerous RBAC Permissions and Privilege Escalation

Some permissions can indirectly provide administrative access.

### High-risk permissions

| Permission | Risk |
|---|---|
| `secrets: get/list` | Exposes credentials and tokens |
| `roles: create/update` | Can create or modify permissions |
| `rolebindings: create` | Can grant permissions to identities |
| `clusterrolebindings: create` | Can grant cluster-wide access |
| `serviceaccounts: create` | Can create identities used by workloads |
| `pods: create` | Can create workloads using available ServiceAccounts |
| `pods/exec: create` | Can execute commands inside containers |
| `pods/attach: create` | Can attach to running containers |
| `nodes/proxy: get` | Can provide access to node APIs |
| `impersonate` | Can act as another user or group |

### Example: Dangerous wildcard permission

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: dangerous-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
```

This grants extremely broad permissions.

**Never use wildcard permissions casually in production.**

### Privilege escalation protections

Kubernetes RBAC includes special checks around permissions such as `bind`, `escalate`, and `impersonate`.

- `bind`: Allows binding roles that the caller could not otherwise grant.
- `escalate`: Allows creating or updating roles with permissions the caller does not possess.
- `impersonate`: Allows acting as another identity.

**Production best practices:**
- Restrict who can create Roles and RoleBindings.
- Restrict access to Secrets.
- Audit ServiceAccount permissions.
- Avoid unnecessary `pods/exec`.
- Review wildcard permissions regularly.
- Separate cluster administrators from application operators.

---

## 16. Namespace Isolation Using RBAC

Namespaces provide logical separation, while RBAC controls access within that separation.

### Example: Developer access only to development namespace

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
      - create
      - update
      - patch
      - delete
```

Bind the role to a user:

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

The developer can manage permitted resources in `development`, but does not automatically receive access to `production`.

**Important:** RBAC namespace isolation is not complete security isolation. Use NetworkPolicies, Pod Security Admission, resource quotas, and other controls where appropriate.

---

## 17. RBAC Troubleshooting Commands

### Check current identity

```bash
kubectl auth whoami
```

### Check whether an action is allowed

```bash
kubectl auth can-i get pods
```

### Check access in a specific namespace

```bash
kubectl auth can-i get pods -n development
```

### Check whether a user can delete deployments

```bash
kubectl auth can-i delete deployments \
  --as=developer@example.com \
  -n development
```

### Check whether a ServiceAccount can list pods

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:development:app-sa \
  -n development
```

### List Roles

```bash
kubectl get roles -A
```

### List RoleBindings

```bash
kubectl get rolebindings -A
```

### List ClusterRoles

```bash
kubectl get clusterroles
```

### List ClusterRoleBindings

```bash
kubectl get clusterrolebindings
```

### Describe a Role

```bash
kubectl describe role developer-role -n development
```

### Describe a RoleBinding

```bash
kubectl describe rolebinding developer-binding -n development
```

### Inspect a ServiceAccount

```bash
kubectl describe serviceaccount app-sa -n development
```

### Test all permissions for a user

```bash
kubectl auth can-i --list \
  --as=developer@example.com \
  -n development
```

**Troubleshooting checklist:**
1. Confirm the correct Kubernetes context.
2. Confirm the identity being authenticated.
3. Check the namespace.
4. Check the Role or ClusterRole rules.
5. Check the RoleBinding or ClusterRoleBinding.
6. Confirm the subject name and kind.
7. Confirm the `roleRef` points to the correct role.
8. Check API groups, resources, and verbs.
9. Use `kubectl auth can-i` to verify the exact action.

---

## 18. Common RBAC Errors

### Error 1: Forbidden

```text
Error from server (Forbidden):
pods is forbidden:
User "developer" cannot list resource "pods"
in API group "" in namespace "development"
```

**Possible reasons:**
- Missing Role or ClusterRole permission.
- Missing RoleBinding or ClusterRoleBinding.
- Incorrect namespace.
- Incorrect username or ServiceAccount identity.
- Missing `list` verb.

### Error 2: RoleBinding references the wrong role

Check:

```yaml
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

Make sure the referenced role exists in the same namespace as the RoleBinding.

### Error 3: Wrong API group

For core resources such as Pods and Services:

```yaml
apiGroups: [""]
```

For Deployments:

```yaml
apiGroups: ["apps"]
```

### Error 4: Missing verb

If the application needs to watch Pods, the role must include:

```yaml
verbs:
  - get
  - list
  - watch
```

### Error 5: Wrong ServiceAccount name

The identity format is:

```text
system:serviceaccount:<namespace>:<serviceaccount-name>
```

Example:

```text
system:serviceaccount:development:app-sa
```

### Error 6: Role exists but access is denied

A Role does not grant permissions by itself. It must be connected to a subject using a RoleBinding.

---

## 19. RBAC Auditing and Governance

Production Kubernetes environments need regular RBAC reviews.

### What should be audited?

- Users with `cluster-admin`.
- ServiceAccounts with broad permissions.
- Roles containing wildcard resources or verbs.
- ClusterRoleBindings.
- Access to Secrets.
- Permissions to create Pods.
- Permissions to create or modify Roles and RoleBindings.
- Unused or outdated RoleBindings.
- Access granted to external automation systems.

### Useful audit commands

```bash
kubectl get clusterrolebindings -o wide
```

```bash
kubectl get rolebindings -A -o wide
```

```bash
kubectl get roles -A
```

```bash
kubectl get clusterroles
```

### Production governance practices

1. Follow least privilege.
2. Review access periodically.
3. Use separate identities for applications and humans.
4. Keep RBAC manifests in Git.
5. Review permission changes through pull requests.
6. Remove unused bindings.
7. Monitor Kubernetes audit logs.
8. Avoid sharing ServiceAccount credentials.
9. Document ownership of privileged roles.
10. Use separate access policies for development, staging, and production.

### GitOps example

Store RBAC manifests in a Git repository:

```text
kubernetes/
├── namespaces/
│   ├── development.yaml
│   └── production.yaml
├── rbac/
│   ├── developer-role.yaml
│   ├── developer-binding.yaml
│   ├── monitoring-role.yaml
│   └── monitoring-binding.yaml
└── serviceaccounts/
    └── app-serviceaccount.yaml
```

Apply manifests through your deployment pipeline:

```bash
kubectl apply -f kubernetes/rbac/
```

---

## 20. RBAC vs ABAC

Kubernetes supports different authorization approaches.

### RBAC

Role-Based Access Control grants permissions through Roles and RoleBindings.

Example:

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
```

### ABAC

Attribute-Based Access Control evaluates attributes of the user, resource, and request against authorization policies.

| Feature | RBAC | ABAC |
|---|---|---|
| Full form | Role-Based Access Control | Attribute-Based Access Control |
| Permission model | Roles and bindings | Attributes and policies |
| Configuration | Kubernetes API objects | Policy configuration |
| Management | Easier to manage through Kubernetes manifests | Requires policy management |
| Common use | Standard Kubernetes authorization | Attribute-driven authorization scenarios |

**Production recommendation:** RBAC is the commonly used authorization approach for Kubernetes clusters. Use narrowly scoped roles and bindings, and avoid broad access grants.
