# Troubleshooting: Database Persistent Storage (State Loss on Restart)

## 1. Issue overview
A database persistent storage issue occurs when a stateful workload is deployed without persistent storage mounts (such as `PersistentVolumeClaim`), storing its data files in ephemeral container layer storage.

In Kubernetes:
- Containers are **ephemeral by default**. When a Pod restarts, crashes, or is rescheduled to another node, its container root filesystem is completely destroyed and re-initialized from the base image.
- If a database writes data to `/var/lib/mysql` without mounting a `PersistentVolume`, all tables and database transactions are permanently destroyed upon Pod recreation.
- **Architectural Solution:** Stateful databases must mount a `PersistentVolumeClaim` to store data on persistent storage (SAN, cloud block storage, or hostpath) that survives Pod terminations.

**Expected behavior:** Database records written to disk survive container crashes, restarts, and node rescheduling.  
**Actual behavior:** The database stores data in container rootfs; killing the Pod deletes all records.

---

## 2. Common causes
1. **Omitting `volumeMounts` in database Pods:** Running databases without mounting a persistent volume.
2. **Mounting `emptyDir` instead of a PVC:** Using `emptyDir: {}` which is wiped as soon as the Pod is deleted.
3. **Mounting wrong directory:** Mounting the PVC to `/data` while MySQL writes to `/var/lib/mysql`.
4. **Deploying databases via Deployment instead of StatefulSet:** Deployments are designed for stateless workloads; single-replica databases require strict volume attachment guarantees.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\07-database-persistent-storage"
```

Apply the invalid Deployment (without volume mounts):

```powershell
kubectl apply -f .\invalid-mysql-deployment.yaml
kubectl rollout status deployment/deploy-mysql-ephemeral --timeout=60s
```

Simulate writing critical customer records to the database:

```powershell
$pod = (kubectl get pods -l app=mysql-ephemeral-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $pod -- mkdir -p /var/lib/mysql
kubectl exec $pod -- sh -c "echo 'CUSTOMER_ID_1001: ACTIVE' > /var/lib/mysql/data.txt"
```

Verify that the data exists:

```powershell
kubectl exec $pod -- cat /var/lib/mysql/data.txt
```

Output:
```text
CUSTOMER_ID_1001: ACTIVE
```

Now simulate a Pod failure or node restart by deleting the Pod:

```powershell
kubectl delete pod $pod
```

Wait for the Deployment controller to create a replacement Pod:

```powershell
Start-Sleep -Seconds 5
$newPod = (kubectl get pods -l app=mysql-ephemeral-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod
```

Observe that all data is permanently lost:
```text
Simulated database running...
Records stored in /var/lib/mysql: NONE (DATA LOST)
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod volume mounts
```powershell
kubectl get pod $newPod -o jsonpath='{.spec.containers[0].volumeMounts}'
```

Output:
```text
[{"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount","name":"kube-api-access-..."}]
```
*Notice that there are NO storage volume mounts attached to `/var/lib/mysql`.*

### Step 2: Diagnostic conclusion
- The database wrote directly to the writable container layer.
- When the container was deleted, containerd cleaned up the overlayfs layer, destroying all data.

---

## 6. Root cause
`invalid-mysql-deployment.yaml` lacked a `volumeMounts` section binding `/var/lib/mysql` to a `PersistentVolumeClaim`.

---

## 7. Fix the issue
Apply the PVC and corrected Deployment:

```powershell
kubectl apply -f .\fixed-pvc.yaml
kubectl apply -f .\fixed-mysql-deployment.yaml
kubectl rollout status deployment/deploy-mysql-ephemeral --timeout=60s
```

---

## 8. Verify the fix
Identify the new persistent Pod:

```powershell
$persistPod = (kubectl get pods -l app=mysql-ephemeral-demo -o jsonpath='{.items[0].metadata.name}')
```

Write records to the persistent mount:

```powershell
kubectl exec $persistPod -- sh -c "echo 'CUSTOMER_ID_2002: SAVED_TO_PV' > /var/lib/mysql/data.txt"
kubectl exec $persistPod -- cat /var/lib/mysql/data.txt
```

Now delete the Pod again:

```powershell
kubectl delete pod $persistPod
```

Wait for the replacement Pod to spin up and mount the same PVC:

```powershell
Start-Sleep -Seconds 8
$recoveredPod = (kubectl get pods -l app=mysql-ephemeral-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $recoveredPod
```

Expected output:
```text
Simulated database running with PersistentVolume...
Records stored in /var/lib/mysql: CUSTOMER_ID_2002: SAVED_TO_PV
```
*The data survived Pod termination and was cleanly recovered!*

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-initial-data-written.png` — Output showing data written to ephemeral container.
- [ ] `02-data-lost-after-deletion.png` — Output of logs showing `NONE (DATA LOST)` after deleting pod.
- [ ] `03-pvc-bound.png` — Output of `kubectl get pvc` showing `mysql-pvc-demo Bound`.
- [ ] `04-verified-persistent-recovery.png` — Output showing `CUSTOMER_ID_2002: SAVED_TO_PV` successfully preserved across deletion.

---

## 10. Cleanup
Delete the Deployment and PVC:

```powershell
kubectl delete -f .\fixed-mysql-deployment.yaml --ignore-not-found=true
kubectl delete -f .\fixed-pvc.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-mysql-ephemeral
kubectl get pvc mysql-pvc-demo
```

---

## 11. Key learning
- **Root cause:** Database stored data in the ephemeral container filesystem instead of a PersistentVolume.
- **Most useful diagnostic command:** `kubectl get pod <name> -o yaml` to inspect `volumeMounts` and `volumes`.
- **Correction:** Mount a `PersistentVolumeClaim` to the database data directory (`/var/lib/mysql`).
- **Production consideration:** Use `StatefulSet` with `volumeClaimTemplates` for production databases to ensure unique, stable persistent volumes per replica.
