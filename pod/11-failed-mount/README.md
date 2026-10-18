# Troubleshooting: FailedMount (Missing Volume Dependency)

## 1. Issue overview
`FailedMount` is a volume manager condition where the Kubelet fails to attach, format, or mount a specified volume into the Pod's filesystem before starting its containers.

In Kubernetes:
- The Kubelet Volume Manager reconciler continuously synchronizes desired volumes with actual mounts on the node filesystem.
- When a Pod defines a `persistentVolumeClaim`, `configMap`, or `secret` volume source, Kubelet verifies the existence of the source object and ensures the underlying volume is attached and formatted before container runtime launch.
- If the volume dependency is missing or unfulfilled (such as a nonexistent `PersistentVolumeClaim`), Kubelet cannot mount the directory into `/var/lib/kubelet/pods/<pod-uid>/volumes/...`.
- The Pod remains stuck in the `ContainerCreating` status.

**Expected behavior:** Kubelet resolves the volume reference, mounts the storage into `/var/data`, and launches the container into `Running`.  
**Actual behavior:** The Pod references a PVC named `non-existent-pvc` which does not exist, generating continuous `FailedMount` warning events and trapping the Pod in `ContainerCreating`.

---

## 2. Common causes
1. **Missing or misspelled PersistentVolumeClaim:** The referenced PVC has not been created or the name has a typo.
2. **PVC in `Pending` state:** The PVC exists but cannot find or dynamically provision a matching `PersistentVolume` (e.g. StorageClass misconfiguration).
3. **Multi-Attach error (`VolumeAttachment` conflict):** A `ReadWriteOnce` volume is still attached to another node and cannot be attached to the current node.
4. **Missing non-optional ConfigMap or Secret:** Volume source references a ConfigMap or Secret that does not exist in the namespace.
5. **Filesystem permission / mount path errors:** Node mount point fails due to corrupt disk filesystems or host OS permissions.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod	-failed-mount"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status:

```powershell
kubectl get pods pod-failed-mount
```

Observe that the Pod stays in `ContainerCreating`:
```text
NAME               READY   STATUS              RESTARTS   AGE
pod-failed-mount   0/1     ContainerCreating   0          25s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and volume details
```powershell
kubectl describe pod pod-failed-mount
```

Examine the **Volumes** section:
```text
Volumes:
  missing-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  non-existent-pvc
    ReadOnly:   false
```

Examine the **Events** section:
```text
Events:
  Type     Reason       Age                From               Message
  ----     ------       ----               ----               -------
  Normal   Scheduled    35s                default-scheduler  Successfully assigned default/pod-failed-mount to minikube
  Warning  FailedMount  10s (x3 over 34s)  kubelet            MountVolume.SetUp failed for volume "missing-storage" : persistentvolumeclaim "non-existent-pvc" not found
```

### Step 2: Query for the referenced PVC in the namespace
```powershell
kubectl get pvc non-existent-pvc
```

Expected output:
```text
Error from server (NotFound): persistentvolumeclaims "non-existent-pvc" not found
```

### Step 3: Extract the exact failure message via jsonpath
```powershell
kubectl get events --field-selector involvedObject.name=pod-failed-mount,reason=FailedMount -o jsonpath='{.items[-1].message}'
```

Expected output:
```text
MountVolume.SetUp failed for volume "missing-storage" : persistentvolumeclaim "non-existent-pvc" not found
```

**Diagnostic analysis:**
- The Pod is scheduled successfully to `minikube`, proving this is not a scheduling failure.
- Container runtime has not started because volume setup is a mandatory prerequisite.
- The Kubelet Volume Manager repeatedly reports `persistentvolumeclaim "non-existent-pvc" not found`, establishing that a missing storage dependency is preventing container creation.

---

## 6. Root cause
In `invalid-pod.yaml`, the volume definition references:
```yaml
volumes:
- name: missing-storage
  persistentVolumeClaim:
    claimName: non-existent-pvc
```
The PVC `non-existent-pvc` does not exist in the active namespace, preventing the Kubelet Volume Manager from mounting the volume.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` uses a valid ephemeral `emptyDir: {}` volume for `/var/data`, which is created and mounted immediately by the Kubelet on the local node without requiring external storage claims.

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-failed-mount --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-failed-mount -o wide
```

Expected output:
```text
NAME               READY   STATUS    RESTARTS   AGE
pod-failed-mount   1/1     Running   0          22s
```

Verify that the volume is properly mounted inside the running container:

```powershell
kubectl exec pod-failed-mount -- df -h /var/data
```

Expected output:
```text
Filesystem                Size      Used Available Use% Mounted on
/dev/sda1                ...       ...        ...   ... /var/data
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-container-creating-status.png` — Output of `kubectl get pods` showing `STATUS: ContainerCreating`.
- [ ] `02-failed-mount-event.png` — Output of `kubectl describe pod` showing `MountVolume.SetUp failed for volume ... not found`.
- [ ] `03-pvc-notfound.png` — Output of `kubectl get pvc` showing `NotFound`.
- [ ] `04-verified-mount-running.png` — Output of `kubectl exec` verifying the mounted `/var/data` directory in the running Pod.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-failed-mount
```

---

## 11. Key learning
- **Root cause:** The Pod specified a volume backed by a `persistentVolumeClaim` that does not exist in the namespace.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the `FailedMount` event in the Events log.
- **Correction:** Create the required PVC/PV or update the Pod manifest to reference a valid storage volume (e.g., `emptyDir`).
- **Production consideration:** Always declare and provision storage dependencies (StorageClass, PV, PVC) before deploying stateful workloads, and verify that the target StorageClass supports dynamic provisioning.
