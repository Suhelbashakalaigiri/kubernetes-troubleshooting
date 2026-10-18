# Troubleshooting: PVC Pending (Unsatisfiable StorageClass)

## 1. Issue overview
A PersistentVolumeClaim (PVC) remaining in the `Pending` state indicates that the Kubernetes storage control plane cannot bind the claim to an existing PersistentVolume (PV) or dynamically provision a new volume.

In Kubernetes:
- PVCs request storage with specific `accessModes`, `storageClassName`, and capacity.
- The `persistentvolume-controller` examines the requested `storageClassName`.
- If the StorageClass does not exist in the cluster (e.g. `non-existent-storage-class`), dynamic volume provisioning cannot be initiated.
- The PVC remains stuck in `STATUS: Pending` indefinitely, preventing any dependent Pods from starting.

**Expected behavior:** The dynamic provisioner creates a PersistentVolume backed by Minikube hostpath storage and binds the claim (`STATUS: Bound`).  
**Actual behavior:** The claim requests a non-existent StorageClass, leaving the PVC in `Pending`.

---

## 2. Common causes
1. **Non-existent or misspelled StorageClass:** Claim specifies a class name that has not been created in the cluster.
2. **Missing default StorageClass:** Claim omits `storageClassName`, but the cluster lacks an annotated default StorageClass (`storageclass.kubernetes.io/is-default-class: "true"`).
3. **Provisioner plugin offline:** The CSI driver or hostpath provisioner Pod in `kube-system` is crashed or not running.
4. **Volume binding mode delay:** StorageClass uses `volumeBindingMode: WaitForFirstConsumer`; the PVC intentionally stays `Pending` until a consuming Pod is scheduled.
5. **Requested capacity exceeds backend pool:** Requesting 10Ti when the underlying host or SAN has insufficient capacity.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\01-pvc-pending"
```

Apply the invalid PVC manifest:

```powershell
kubectl apply -f .\invalid-pvc.yaml
```

Check the PVC status after 10 seconds:

```powershell
kubectl get pvc pvc-pending-demo
```

Observe that `STATUS` is `Pending` with no volume bound:
```text
NAME               STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
pvc-pending-demo   Pending                                      non-existent-storage-class   25s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect PVC events and controller warnings
```powershell
kubectl describe pvc pvc-pending-demo
```

Examine the **Events** section:
```text
Events:
  Type     Reason              Age                From                         Message
  ----     ------              ----               ----                         -------
  Warning  ProvisioningFailed  15s (x3 over 30s)  persistentvolume-controller  storageclass.storage.k8s.io "non-existent-storage-class" not found
```

### Step 2: List available StorageClasses in the cluster
```powershell
kubectl get storageclass
```

Expected output in Minikube:
```text
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  5d
```
Notice:
- The valid storage class is `standard` (using `k8s.io/minikube-hostpath`).
- `non-existent-storage-class` does not exist.

---

## 6. Root cause
In `invalid-pvc.yaml`, line 9 specifies:
```yaml
storageClassName: non-existent-storage-class
```
Because no such StorageClass exists, dynamic volume provisioning fails.

---

## 7. Fix the issue
Because the `storageClassName` of a PVC is immutable after creation, delete the invalid claim and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pvc.yaml
kubectl apply -f .\fixed-pvc.yaml
```

The corrected manifest `fixed-pvc.yaml` configures `storageClassName: standard` and requests a lightweight `1Gi` volume.

---

## 8. Verify the fix
Wait for the PVC to bind:

```powershell
kubectl get pvc pvc-pending-demo
```

Expected output:
```text
NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-pending-demo   Bound    pvc-7d84b9c5-8j4k-4b21-9f9b-11a22b33c44d   1Gi        RWO            standard       10s
```
Verify that `STATUS` is `Bound` and a dynamic PersistentVolume was automatically allocated.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-pvc-pending-status.png` — Output of `kubectl get pvc` showing `STATUS: Pending`.
- [ ] `02-storageclass-not-found-event.png` — Output of `kubectl describe pvc` showing `storageclass ... not found`.
- [ ] `03-available-storageclasses.png` — Output of `kubectl get storageclass` showing `standard (default)`.
- [ ] `04-verified-pvc-bound.png` — Output of `kubectl get pvc` showing `STATUS: Bound`.

---

## 10. Cleanup
Delete the PVC:

```powershell
kubectl delete -f .\fixed-pvc.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pvc pvc-pending-demo
```

---

## 11. Key learning
- **Root cause:** The claim requested a non-existent StorageClass, preventing the storage controller from provisioning a volume.
- **Most useful diagnostic command:** `kubectl describe pvc <claim-name>` to read storage controller provisioning events.
- **Correction:** Reference a verified, existing StorageClass (such as `standard` in Minikube) or omit the field to use the cluster default.
- **Production consideration:** Pay attention to `volumeBindingMode: WaitForFirstConsumer` on modern CSI drivers: PVCs will intentionally remain `Pending` until a consuming Pod is scheduled to ensure the volume is provisioned in the same availability zone as the Pod.
