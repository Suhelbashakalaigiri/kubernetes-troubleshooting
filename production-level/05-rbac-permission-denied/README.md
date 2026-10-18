# Troubleshooting: RBAC Permission Denied (Missing Verbs)

## 1. Issue overview
Role-Based Access Control (RBAC) regulates access to Kubernetes API resources based on the roles assigned to users and ServiceAccounts. An RBAC permission denied issue occurs when a subject attempts an API operation for which it lacks the required verb or resource permission.

In Kubernetes:
- The API server evaluates incoming requests against `apiGroups`, `resources`, and `verbs`.
- Running `kubectl get pods` requires the **`list`** verb.
- Running `kubectl get pod <specific-name>` requires the **`get`** verb.
- Running `kubectl get pods -w` requires the **`watch`** verb.
- If a Role grants only `verbs: ["get"]`, attempting to list all Pods (`kubectl get pods`) is rejected with HTTP 403 Forbidden.

**Expected behavior:** The `audit-viewer-sa` ServiceAccount can list Pods in the namespace.  
**Actual behavior:** The Role grants only `get`, causing `kubectl get pods --as=...` to be rejected with `Error from server (Forbidden)`.

---

## 2. Common causes
1. **Missing `list` verb:** Role defines `get` but omits `list`, allowing inspection of individual items by name but blocking queries across the collection.
2. **Wrong `apiGroups`:** Specifying `""` (core group) for resources that belong to `apps` (e.g. Deployments).
3. **Role vs. ClusterRole confusion:** Binding a namespaced `Role` when cluster-wide access was intended.
4. **Subject namespace mismatch in RoleBinding:** The ServiceAccount namespace in `subjects` does not match the actual namespace where the ServiceAccount lives.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\05-rbac-permission-denied"
```

Apply the invalid RBAC manifests:

```powershell
kubectl apply -f .\invalid-rolebinding.yaml
```

Test listing pods impersonating the ServiceAccount:

```powershell
kubectl get pods --as=system:serviceaccount:default:audit-viewer-sa
```

Observe the 403 Forbidden rejection:
```text
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:audit-viewer-sa" cannot list resource "pods" in API group "" in the namespace "default"
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Use `kubectl auth can-i` to test permissions
Check if the ServiceAccount can `get` a specific pod:

```powershell
kubectl auth can-i get pods --as=system:serviceaccount:default:audit-viewer-sa
```

Output:
```text
yes
```

Now check if it can `list` pods:

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:audit-viewer-sa
```

Output:
```text
no
```

### Step 2: Inspect the Role rules
```powershell
kubectl describe role audit-viewer-role
```

Output:
```text
Resources  Non-Resource URLs  Resource Names  Verbs
---------  -----------------  --------------  -----
pods       []                 []              [get]
```
Notice: `verbs` contains only `[get]`.

**Diagnostic analysis:**
- `kubectl get pods` requires collection-level permission (`list`).
- The Role only granted item-level access (`get`).
- This produces the exact 403 Forbidden error observed.

---

## 6. Root cause
In `invalid-rolebinding.yaml`, line 17 specifies:
```yaml
verbs: ["get"]
```
The `list` and `watch` verbs are missing from the Role.

---

## 7. Fix the issue
Apply the fixed manifest:

```powershell
kubectl apply -f .\fixed-rolebinding.yaml
```

The corrected manifest `fixed-rolebinding.yaml` grants `verbs: ["get", "list", "watch"]`.

---

## 8. Verify the fix
Test permissions using `kubectl auth can-i`:

```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:audit-viewer-sa
```

Expected output:
```text
yes
```

Execute `kubectl get pods` under the ServiceAccount identity:

```powershell
kubectl get pods --as=system:serviceaccount:default:audit-viewer-sa
```

Expected output:
```text
No resources found in default namespace. (or list of active pods)
```
The command succeeds without Forbidden errors.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-rbac-forbidden-error.png` — Output showing `Error from server (Forbidden): pods is forbidden`.
- [ ] `02-auth-cani-list-no.png` — Output of `kubectl auth can-i list pods` returning `no`.
- [ ] `03-describe-role-get-only.png` — Output of `kubectl describe role` showing `[get]`.
- [ ] `04-verified-cani-yes.png` — Output of `kubectl auth can-i list pods` returning `yes`.

---

## 10. Cleanup
Delete all RBAC resources:

```powershell
kubectl delete -f .\fixed-rolebinding.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get role audit-viewer-role
kubectl get serviceaccount audit-viewer-sa
```

---

## 11. Key learning
- **Root cause:** The Role granted `get` but omitted `list`, causing collection queries to fail with 403 Forbidden.
- **Most useful diagnostic command:** `kubectl auth can-i <verb> <resource> --as=<subject>` to systematically test permission boundaries.
- **Correction:** Grant the full set of necessary verbs (`["get", "list", "watch"]`).
- **Production consideration:** Follow the principle of least privilege: do not grant wildcard `verbs: ["*"]` or cluster-admin privileges to application service accounts.
