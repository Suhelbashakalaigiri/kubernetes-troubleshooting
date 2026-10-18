# Troubleshooting: Pod Disruption Budget (Drain Operation Blocked)

## 1. Issue overview
A PodDisruptionBudget (PDB) limits the number of Pods of a replicated application that can be down simultaneously during voluntary disruptions (such as node drains for kernel upgrades, cluster autoscaler scale-downs, or node maintenance).

In Kubernetes:
- Setting `minAvailable: 100%` (or `maxUnavailable: 0`) on a 2-replica Deployment specifies that **all 2 replicas must remain available at all times**.
- When an administrator runs `kubectl drain <node>`, the Eviction API checks the PDB before terminating each Pod.
- Because evicting even one Pod reduces availability to 50% (violating the 100% budget), the Eviction API rejects the request with HTTP 429: `Cannot evict pod as it would violate the pod's disruption budget`.
- The node drain hangs indefinitely, blocking cluster upgrades.

**Expected behavior:** Voluntary disruptions permit evicting 1 Pod at a time while the remaining replica handles client traffic.  
**Actual behavior:** Setting `minAvailable: 100%` on 2 replicas prevents `kubectl drain` or Pod eviction.

---

## 2. Common causes
1. **`minAvailable: 100%` on small replica counts:** Requiring 100% availability prevents evicting any Pod when total replicas <= 2.
2. **`maxUnavailable: 0`:** Permanently forbids all voluntary disruptions.
3. **PDB selector selecting unready Pods:** If current Pods are unready, allowed disruptions drops to 0, blocking drain operations.
4. **Single-replica Deployment with a PDB:** A 1-replica workload with `minAvailable: 1` can never be voluntarily evicted.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\04-pod-disruption-availability"
```

Apply the invalid Deployment and PDB:

```powershell
kubectl apply -f .\invalid-pdb.yaml
kubectl rollout status deployment/deploy-pdb-blocked --timeout=60s
```

Inspect the PDB status:

```powershell
kubectl get pdb pdb-strict-block
```

Observe that `ALLOWED DISRUPTIONS` is `0`:
```text
NAME               MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
pdb-strict-block   100%            N/A               0                     25s
```

Test an eviction attempt using `kubectl drain` in dry-run mode:

```powershell
kubectl drain minikube --dry-run=client --ignore-daemonsets --delete-emptydir-data
```

Notice that the eviction API reports that the Pod cannot be drained:
```text
evicting pod default/deploy-pdb-blocked-...
error when evicting pods: Cannot evict pod as it would violate the pod's disruption budget.
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect PDB status metrics
```powershell
kubectl describe pdb pdb-strict-block
```

Output:
```text
Name:           pdb-strict-block
Namespace:      default
Min available:  100%
Selector:       app=pdb-demo
Status:
  Current:             2
  Desired:             2
  DisruptionsAllowed:  0
```
*Note that `DisruptionsAllowed: 0`. No Pods can be evicted!*

**Diagnostic analysis:**
- Total replicas: 2.
- `minAvailable: 100%` means at least 2 Pods must be healthy at all times.
- Therefore, $2 - 2 = 0$ disruptions allowed. Any drain operation is blocked.

---

## 6. Root cause
In `invalid-pdb.yaml`, line 28 sets `minAvailable: 100%`. For a 2-replica deployment, this leaves zero room for voluntary evictions.

---

## 7. Fix the issue
Apply the fixed PDB manifest:

```powershell
kubectl apply -f .\fixed-pdb.yaml
```

The corrected manifest `fixed-pdb.yaml` updates the disruption budget to:
```yaml
maxUnavailable: 1
```

---

## 8. Verify the fix
Check the updated PDB status:

```powershell
kubectl get pdb pdb-strict-block
```

Expected output:
```text
NAME               MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
pdb-strict-block   N/A             1                 1                     2m
```
*Notice: `ALLOWED DISRUPTIONS` is now `1`.*

Test dry-run drain again:

```powershell
kubectl drain minikube --dry-run=client --ignore-daemonsets --delete-emptydir-data
```

Eviction dry-run passes without disruption violations.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-zero-disruptions-allowed.png` — Output of `kubectl get pdb` showing `ALLOWED DISRUPTIONS: 0`.
- [ ] `02-drain-eviction-blocked.png` — Output showing `Cannot evict pod as it would violate the pod's disruption budget`.
- [ ] `03-describe-pdb-output.png` — Output of `kubectl describe pdb` showing `Min available: 100%`.
- [ ] `04-verified-allowed-disruptions-1.png` — Output of `kubectl get pdb` showing `ALLOWED DISRUPTIONS: 1`.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-pdb.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pdb pdb-strict-block
kubectl get deployment deploy-pdb-blocked
```

---

## 11. Key learning
- **Root cause:** `minAvailable: 100%` on 2 replicas allowed 0 disruptions, blocking all node drain and eviction operations.
- **Most useful diagnostic command:** `kubectl get pdb` to check `ALLOWED DISRUPTIONS`.
- **Correction:** Use `maxUnavailable: 1` or scale workloads to at least 3 replicas when using fractional disruption budgets.
- **Production consideration:** Never apply `minAvailable: 100%` to Deployments with fewer than 3 replicas in production clusters subject to automated node patching or Karpenter/Cluster Autoscaler node consolidation.
