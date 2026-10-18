# Troubleshooting: Missing Volume Reference (API Validation Rejection)

## 1. Issue overview
A missing volume reference occurs when a container specification defines a `volumeMounts` entry referencing a volume name that is not declared in the Pod's `spec.volumes` list.

In Kubernetes:
- The Kubernetes API server enforces strict schema validation and referential integrity upon admission.
- Every entry in `spec.containers[*].volumeMounts[*].name` **must have a corresponding matching entry** in `spec.volumes[*].name`.
- If an application manifest defines `volumeMounts: - name: data-volume`, but the volumes section names the volume `storage-volume-mismatched`, the API server rejects the resource immediately upon `kubectl apply` with an admission validation error.
- **Diagnostic distinction:** The resource is never created in etcd; no Pod is scheduled or created.

**Expected behavior:** The manifest specifies matching volume names, passes admission validation, and schedules the Pod.  
**Actual behavior:** The API server rejects the manifest with `spec.containers[0].volumeMounts[0].name: Not found: "data-volume"`.

---

## 2. Common causes
1. **Typo in volume name:** Name mismatch between `volumeMounts.name` and `volumes.name`.
2. **Copy-paste error:** Adding volume mounts to a container without copying the corresponding `volumes` stanza.
3. **Refactoring oversight:** Renaming a volume in `volumes` but forgetting to update container mounts.
4. **Multiple containers with partial volume mounts:** One container references a volume intended only for another.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\03-missing-volume-reference"
```

Attempt to apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Observe the immediate API server rejection:
```text
The Pod "pod-missing-volume-ref" is invalid: spec.containers[0].volumeMounts[0].name: Not found: "data-volume"
```

Verify that no Pod was created:

```powershell
kubectl get pod pod-missing-volume-ref
```

Output:
```text
Error from server (NotFound): pods "pod-missing-volume-ref" not found
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Analyze the admission validation error
- The error message explicitly points to:
  `spec.containers[0].volumeMounts[0].name: Not found: "data-volume"`
- This means the first container's first volume mount requests `"data-volume"`.
- The API server searched `spec.volumes` and found no volume named `"data-volume"`.

### Step 2: Compare manifest fields directly
Inspect lines 13-18 of `invalid-pod.yaml`:
```yaml
volumeMounts:
- name: data-volume
  mountPath: /var/data
volumes:
- name: storage-volume-mismatched
  emptyDir: {}
```
Notice that `data-volume` does not match `storage-volume-mismatched`.

---

## 6. Root cause
In `invalid-pod.yaml`, the container requests mount `name: data-volume`, but the volume is named `name: storage-volume-mismatched`. The API server rejects manifests violating referential integrity.

---

## 7. Fix the issue
Apply the fixed manifest:

```powershell
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` aligns both fields to `name: data-volume`.

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-missing-volume-ref --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-missing-volume-ref -o wide
```

Expected output:
```text
NAME                     READY   STATUS    RESTARTS   AGE
pod-missing-volume-ref   1/1     Running   0          20s
```

Verify the volume mount inside the running container:

```powershell
kubectl exec pod-missing-volume-ref -- df -h /var/data
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-api-validation-rejection.png` — Output of `kubectl apply` showing `Not found: "data-volume"`.
- [ ] `02-manifest-comparison.png` — Viewing `invalid-pod.yaml` showing mismatched volume names.
- [ ] `03-fixed-manifest-applied.png` — Successful application of `fixed-pod.yaml`.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod` showing `STATUS: Running`.

---

## 10. Cleanup
Delete the Pod:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-missing-volume-ref
```

---

## 11. Key learning
- **Root cause:** The container's `volumeMounts` name did not match any volume declared in `spec.volumes`, violating API schema referential integrity.
- **Most useful diagnostic command:** Reading the direct output of `kubectl apply` which pinpoints the exact field path (`spec.containers[0].volumeMounts[0].name`).
- **Correction:** Align volume mount names with the declared volume list.
- **Production consideration:** Use `kubectl apply --dry-run=client -f <file>` or `kubeconform` / `kubeval` in CI pipelines to catch referential integrity and schema errors before deployment.
