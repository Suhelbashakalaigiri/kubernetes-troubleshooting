# Troubleshooting: Failed Mount (Unbound PVC Mount Dependency)

## 1. Issue overview
A `FailedMount` issue occurs when a Pod is scheduled onto a node, but the Kubelet Volume Manager cannot attach or mount the requested volume because the underlying PersistentVolumeClaim is unfulfilled or unbound.

In Kubernetes:
- The Kubelet Volume Manager synchronizes volumes required by scheduled Pods before starting their containers.
- If a Pod mounts a PVC that is stuck in `Pending` (for instance, because its StorageClass does not exist), the Kubelet cannot mount the volume directory into `/var/lib/kubelet/pods/<uid>/volumes/`.
- The Pod remains stuck in `ContainerCreating` status.

**Expected behavior:** The PVC binds to storage, Kubelet mounts the volume into `/var/data`, and the container reaches `Running`.  
**Actual behavior:** The PVC is unbound due to an invalid StorageClass, causing Kubelet to log `FailedMount` warnings and freeze the Pod in `ContainerCreating`.

---

## 2. Common causes
1. **Referencing an unbound or Pending PVC:** Pod scheduled, but PVC cannot provision storage.
2. **Multi-Attach error:** `ReadWriteOnce` volume already attached to a different node.
3. **Missing ConfigMap/Secret volume source:** Volume references an uncreated configuration object.
4. **Host filesystem mount errors:** Disk corruption or missing local directory path for hostPath volumes.
5. **SubPath directory conflict:** Specifying `subPath` on a volume that already has a conflicting file at that path.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\02-failed-mount"
```

Apply the invalid manifests:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status after 15 seconds:

```powershell
kubectl get pod pod-storage-failed-mount
```

Observe `ContainerCreating`:
```text
NAME                       READY   STATUS              RESTARTS   AGE
pod-storage-failed-mount   0/1     ContainerCreating   0          30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events
```powershell
kubectl describe pod pod-storage-failed-mount
```

Examine the **Events** section:
```text
Events:
  Type     Reason       Age                From               Message
  ----     ------       ----               ----               -------
  Normal   Scheduled    45s                default-scheduler  Successfully assigned default/pod-storage-failed-mount to minikube
  Warning  FailedMount  10s (x3 over 44s)  kubelet            Unable to attach or mount volumes: unmounted volumes=[data-store]: timed out waiting for the condition
```

### Step 2: Trace the root cause to the PVC
Inspect the PVC referenced by volume `data-store`:

```powershell
kubectl get pvc pvc-broken-mount
```

Output:
```text
NAME               STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS                    AGE
pvc-broken-mount   Pending                                      non-existent-class-for-mount   50s
```

Describe the PVC:

```powershell
kubectl describe pvc pvc-broken-mount
```

Output:
```text
storageclass.storage.k8s.io "non-existent-class-for-mount" not found
```

**Diagnostic analysis:**
- The Pod is scheduled, but stuck in `ContainerCreating`.
- Kubelet reports `Unable to attach or mount volumes: unmounted volumes=[data-store]`.
- Checking the PVC reveals the claim is `Pending` because `non-existent-class-for-mount` does not exist.

---

## 6. Root cause
`pvc-broken-mount` specifies an invalid `storageClassName: non-existent-class-for-mount`. The storage controller cannot provision storage, leaving the PVC in `Pending` and preventing Kubelet from mounting the volume into the Pod.

---

## 7. Fix the issue
Delete the failing Pod and PVC, then apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` uses `storageClassName: standard`.

---

## 8. Verify the fix
Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-storage-failed-mount --timeout=60s
```

Check Pod and PVC status:

```powershell
kubectl get pod pod-storage-failed-mount -o wide
kubectl get pvc pvc-broken-mount
```

Expected output:
```text
NAME                       READY   STATUS    RESTARTS   AGE
pod-storage-failed-mount   1/1     Running   0          25s

NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-broken-mount   Bound    pvc-8b74c9d5-11a2-4c33-9f8e-44a55b66c77d   1Gi        RWO            standard       30s
```

Verify that the mount is active inside the container:

```powershell
kubectl exec pod-storage-failed-mount -- df -h /var/data
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-container-creating-status.png` — Output of `kubectl get pods` showing `STATUS: ContainerCreating`.
- [ ] `02-failed-mount-event.png` — Output of `kubectl describe pod` showing `Unable to attach or mount volumes`.
- [ ] `03-pvc-pending-diagnostic.png` — Output of `kubectl get pvc` showing `Pending`.
- [ ] `04-verified-running-mount.png` — Output of `kubectl get pod` and `df -h /var/data` showing healthy running mount.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-storage-failed-mount
kubectl get pvc pvc-broken-mount
```

---

## 11. Key learning
- **Root cause:** An unfulfilled PVC dependency trapped the Pod in `ContainerCreating` with `FailedMount` errors.
- **Most useful diagnostic command:** Tracing from `kubectl describe pod` to `kubectl describe pvc <claim-name>`.
- **Correction:** Fix the underlying PVC and ensure its StorageClass is capable of provisioning storage.
- **Production consideration:** Monitor `kubelet_volume_stats_*` metrics in Prometheus to detect unmounted or failing volume attachments before application alerts fire.
