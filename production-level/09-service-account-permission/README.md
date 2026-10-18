# Troubleshooting: Service Account API Permissions (In-Cluster 403 Forbidden)

## 1. Issue overview
Applications running inside Kubernetes frequently query the in-cluster API server (`https://kubernetes.default.svc`) to discover services, watch pods, or coordinate state. When a Pod uses the `default` ServiceAccount without an attached RoleBinding, API calls are rejected with HTTP 403 Forbidden.

In Kubernetes:
- Every Pod that omits `spec.serviceAccountName` is automatically assigned the `default` ServiceAccount in its namespace.
- Since Kubernetes 1.6+, the `default` ServiceAccount has **no permissions** beyond basic discovery.
- To interact with the Kubernetes API, Pods must be associated with a dedicated `ServiceAccount` bound to a `Role` or `ClusterRole` via a `RoleBinding`.

**Expected behavior:** The in-cluster client queries `https://kubernetes.default.svc` and receives a JSON list of pods.  
**Actual behavior:** Using the `default` ServiceAccount results in `HTTP 403 Forbidden: User "system:serviceaccount:default:default" cannot list resource "pods"`.

---

## 2. Common causes
1. **Omitting `serviceAccountName`:** Pod defaults to `default` ServiceAccount.
2. **Missing RoleBinding:** Dedicated ServiceAccount exists, but was never bound to a Role.
3. **AutomountServiceAccountToken disabled:** Pod specifies `automountServiceAccountToken: false`, preventing the token from being mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`.
4. **Token expiration or projection failure:** Projected token audience does not match the API server's expected audience.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\09-service-account-permission"
```

Apply the invalid Pod manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Wait 10 seconds for the Pod to run, then inspect its logs:

```powershell
kubectl logs pod-api-reader
```

Observe the 403 Forbidden API response:
```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods is forbidden: User "system:serviceaccount:default:default" cannot list resource "pods" in API group "" in the namespace "default"",
  "reason": "Forbidden",
  "details": {
    "resource": "pods"
  },
  "code": 403
}
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Identify which ServiceAccount the Pod is using
```powershell
kubectl get pod pod-api-reader -o jsonpath='{.spec.serviceAccountName}'
```

Output:
```text
default
```

### Step 2: Test API permissions for the `default` ServiceAccount
```powershell
kubectl auth can-i list pods --as=system:serviceaccount:default:default
```

Output:
```text
no
```

**Diagnostic analysis:**
- The Pod uses `serviceAccountName: default`.
- The `default` ServiceAccount has no permissions to list Pods.
- The in-cluster API server rejects the query with HTTP 403.

---

## 6. Root cause
`invalid-pod.yaml` omits `spec.serviceAccountName`, falling back to the `default` ServiceAccount which lacks RBAC permissions.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml`:
1. Creates `ServiceAccount/pod-reader-sa`.
2. Creates `Role/pod-reader-role` with `get` and `list` on `pods`.
3. Binds them with `RoleBinding/pod-reader-binding`.
4. Sets `serviceAccountName: pod-reader-sa` on the Pod.

---

## 8. Verify the fix
Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-api-reader --timeout=60s
```

Check the Pod logs:

```powershell
kubectl logs pod-api-reader
```

Expected output:
```json
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "resourceVersion": "..."
  },
  "items": [
    ...
  ]
}
```
The API returns a valid `PodList` JSON payload with HTTP 200 OK.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-api-forbidden-json.png` — Output of `kubectl logs` showing the 403 Forbidden status JSON.
- [ ] `02-cani-default-no.png` — Output of `kubectl auth can-i` returning `no` for the default account.
- [ ] `03-fixed-rbac-applied.png` — Output of `kubectl apply` showing SA, Role, RoleBinding created.
- [ ] `04-verified-podlist-response.png` — Output of `kubectl logs` showing successful `PodList` return.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-api-reader
kubectl get serviceaccount pod-reader-sa
```

---

## 11. Key learning
- **Root cause:** The Pod used the unprivileged `default` ServiceAccount, resulting in 403 Forbidden when calling the Kubernetes API.
- **Most useful diagnostic command:** `kubectl auth can-i <verb> <resource> --as=<service-account>` to verify in-cluster API permissions.
- **Correction:** Create a dedicated ServiceAccount, grant minimal needed RBAC permissions, and attach it to `spec.serviceAccountName`.
- **Production consideration:** If a workload does not need to communicate with the Kubernetes API, set `automountServiceAccountToken: false` on the Pod spec to reduce the attack surface.
