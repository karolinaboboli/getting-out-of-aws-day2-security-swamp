# ☸️ Kubernetes RBAC Configuration for Operations Engineers

## Access Model: Managed View Policy + Custom ClusterRole

Operations engineers get their access in two stacked layers on a single EKS
access entry:

| **Layer**                                   | **Grants**                                                      | **How**                                    |
|---------------------------------------------|-----------------------------------------------------------------|--------------------------------------------|
| `AmazonEKSAdminViewPolicy` (AWS-managed)    | View (`get/list/watch`) all resources cluster-wide              | EKS access policy on the access entry      |
| `ops-limited-access` (custom ClusterRole)   | Restart deployments, manage PVCs                                | `ops-engineers` K8s group on the same entry |

The managed policy covers all the read/view access, so the custom ClusterRole
only needs to add the **write/action deltas** the view policy doesn't grant.

## What Engineers CAN Do

- View all resources cluster-wide — pods, deployments, services, configmaps,
  namespaces, nodes, events, jobs, ingress, etc. (via `AmazonEKSAdminViewPolicy`)
- Read pod logs (kubectl logs)
- Restart deployments cluster-wide (`kubectl rollout restart`)
- Create and manage PersistentVolumeClaims in any namespace

## What Engineers CANNOT Do

- Create, update, or delete resources, except: restart deployments and manage PVCs
- Exec into pods (`kubectl exec`)
- Modify RBAC roles or bindings
- Create or modify namespaces
- Perform node-level write operations (view-only on nodes)

> ⚠️ **Secrets are viewable.** `AmazonEKSAdminViewPolicy` grants `get/list/watch`
> on **all** resources, which per AWS includes Kubernetes Secrets. If hiding
> secrets is a hard requirement, do not use this managed policy — replace it with
> a custom read-only ClusterRole that omits `secrets`.

## ClusterRole: ops-limited-access

This ClusterRole adds **only** the actions `AmazonEKSAdminViewPolicy` does not
grant. Save as `cluster-role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ops-limited-access
  labels:
    app.kubernetes.io/managed-by: security-team
    app.kubernetes.io/purpose: operations-access
rules:
  # Deployment restarts cluster-wide (kubectl rollout restart patches the
  # pod-template annotation, so patch + update are required)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["patch", "update"]
 
  # PersistentVolumeClaims - full management cluster-wide
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["create", "update", "patch", "delete"]
```

## ClusterRoleBinding: ops-limited-access-binding

Save as `cluster-role-binding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ops-limited-access-binding
  labels:
    app.kubernetes.io/managed-by: security-team
    app.kubernetes.io/purpose: operations-access
subjects:
  - kind: Group
    name: ops-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: ops-limited-access
  apiGroup: rbac.authorization.k8s.io
```

## Apply the Configuration

```bash
# 1. Apply the custom ClusterRole + binding (restart, PVCs, exec)
kubectl apply -f cluster-role.yaml
kubectl apply -f cluster-role-binding.yaml
```

The view layer is applied on the AWS side, by associating the managed policy to
the same access entry that maps the SSO role to the `ops-engineers` group (see
`../access-setup/eks-access-entries.md`):


## Verify the Configuration

```bash
# Check the ClusterRole + binding exist
kubectl get clusterrole ops-limited-access
kubectl get clusterrolebinding ops-limited-access-binding
 
# --- The custom ClusterRole deltas (as the ops-engineers group) ---
kubectl auth can-i patch deployments --as=ops-test --as-group=ops-engineers -A
# Expected: yes

kubectl auth can-i create persistentvolumeclaims --as=ops-test --as-group=ops-engineers -A
# Expected: yes

# Not granted by the ClusterRole (delete is never allowed)
kubectl auth can-i delete pods --as=ops-test --as-group=ops-engineers
# Expected: no

# Exec is deliberately withheld (use --subresource=exec for the accurate check)
kubectl auth can-i create pods --subresource=exec --as=ops-test --as-group=ops-engineers
# Expected: no
```

> Note: view access (list pods, get secrets, list nodes, etc.) comes from
> `AmazonEKSAdminViewPolicy`, not this ClusterRole, so `kubectl auth can-i`
> against the group alone won't reflect it. Verify view access by logging in as
> the SSO role and listing resources, or:
> ```bash
> aws eks list-associated-access-policies --cluster-name cicd-production \
>     --region eu-west-1 --principal-arn "$LIMITED_WORKING_ACCESS_ROLE_ARN"
> ```

## Troubleshooting

### "Forbidden" on view (e.g. `nodes is forbidden ... at the cluster scope`)

The `AmazonEKSAdminViewPolicy` isn't associated with the access entry (or the
entry doesn't exist). The custom ClusterRole does NOT grant view — view comes
from the managed policy. Check and fix:

```bash
aws eks list-associated-access-policies --cluster-name cicd-production \
    --region eu-west-1 --principal-arn "$SSO_ROLE_ARN"
```

### "Forbidden" on restart / PVC / exec

1\. Verify the ClusterRoleBinding exists:

```bash
kubectl describe clusterrolebinding ops-limited-access-binding
```

2\. Check the group name matches the EKS access entry configuration:

```bash
aws eks list-access-entries --cluster-name cicd-production
aws eks describe-access-entry --cluster-name cicd-production \
    --principal-arn "$SSO_ROLE_ARN" --query 'accessEntry.kubernetesGroups'
```

3\. Verify the user's identity:

```bash
kubectl auth whoami
```