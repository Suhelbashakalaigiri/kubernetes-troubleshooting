# Troubleshooting: Resource Exhaustion (Cluster Capacity Saturation)

## 1. Issue overview
Resource exhaustion occurs when a multi-replica workload requests aggregate CPU or memory that exceeds the total allocatable capacity of the Kubernetes cluster, leaving some Pods running while others remain permanently trapped in `Pending`.

In Kubernetes:
- The scheduler runs the `NodeResourcesFit` plugin to evaluate node allocatable capacity against incoming Pod resource requests.
- When an application scales to 4 replicas and each replica requests 3 CPUs and 3Gi of RAM, the aggregate demand is 12 CPUs and 12Gi RAM.
- On a standard single-node Minikube cluster (typically 2–4 CPUs and ~4Gi RAM), the node can only accommodate 1 replica. The remaining 3 replicas stay `Pending` with `FailedScheduling: Insufficient cpu, Insufficient memory`.

**Expected behavior:** All 4 replicas are scheduled, start running, and achieve `4/4 Ready`.  
**Actual behavior:** Only 1 replica schedules; the remaining 3 replicas remain `Pending` due to resource exhaustion.

---

## 2. Common causes
1. **Unrealistic container resource requests:** Requesting full CPU cores (`cpu: "3"`) instead of millicores (`cpu: "300m"`).
2. **Cluster capacity limits:** Cluster lacks enough nodes or node size to fulfill aggregate replica requests.
3. **No Horizontal/Cluster Autoscaler:** In cloud environments, node autoscaling is disabled or has reached maximum node limits.
4. **Namespace ResourceQuota exceeded:** Workload violates a namespace-level `ResourceQuota`.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\01-resource-exhaustion"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Check the Deployment status after 20 seconds:

```powershell
kubectl get deployment deploy-resource-exhaustion
```

Notice that only a fraction of replicas are ready:
```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deploy-resource-exhaustion   0/4     4            0           30s
```

Check the Pods:

```powershell
kubectl get pods -l app=resource-exhaustion-demo
```

Output:
```text
NAME                                          READY   STATUS    RESTARTS   AGE
deploy-resource-exhaustion-7d94bc5f8d-4k2jm   0/1     Pending   0          35s
deploy-resource-exhaustion-7d94bc5f8d-7x9lp   0/1     Pending   0          35s
deploy-resource-exhaustion-7d94bc5f8d-b8d9q   0/1     Pending   0          35s
deploy-resource-exhaustion-7d94bc5f8d-m5zqx   0/1     Pending   0          35s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect scheduling events on a Pending Pod
```powershell
$pendingPod = (kubectl get pods -l app=resource-exhaustion-demo --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pendingPod
```

Examine the **Events** section:
```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  20s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory.
```

### Step 2: Compare node allocatable capacity with Pod requests
```powershell
kubectl describe node minikube | Select-String "Allocatable:","Allocated resources:","  cpu ","  memory " -Context 0,5
```

**Diagnostic analysis:**
- Total allocatable capacity on the node cannot accommodate the 3 CPU cores and 3Gi RAM requested per container.
- The scheduler correctly rejects scheduling the remaining Pods to prevent node starvation.

---

## 6. Root cause
In `invalid-deployment.yaml`, lines 21-24 request `cpu: "3"` and `memory: "3Gi"` per replica. For 4 replicas, this requires 12 CPUs and 12Gi RAM, vastly exceeding node capacity.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` right-sizes requests to `cpu: 50m` (0.05 core) and `memory: 64Mi`, with limits at `cpu: 200m` and `memory: 128Mi`.

---

## 8. Verify the fix
Wait for all 4 replicas to roll out and become Ready:

```powershell
kubectl rollout status deployment/deploy-resource-exhaustion --timeout=60s
```

Check the Deployment metrics:

```powershell
kubectl get deployment deploy-resource-exhaustion
kubectl get pods -l app=resource-exhaustion-demo
```

Expected output:
```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deploy-resource-exhaustion   4/4     4            4           2m
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-pending-replicas-status.png` — Output of `kubectl get pods` showing Pods stuck in `Pending`.
- [ ] `02-insufficient-cpu-event.png` — Output of `kubectl describe pod` showing `1 Insufficient cpu, 1 Insufficient memory`.
- [ ] `03-node-capacity-inspection.png` — Output of `kubectl describe node` showing allocatable capacity.
- [ ] `04-verified-all-4-ready.png` — Output of `kubectl get deployment` showing `READY: 4/4`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-resource-exhaustion
```

---

## 11. Key learning
- **Root cause:** Excessive resource requests caused cluster capacity saturation, preventing Pods from being scheduled.
- **Most useful diagnostic command:** `kubectl describe pod <pending-pod>` to identify `Insufficient cpu` / `Insufficient memory` scheduler warnings.
- **Correction:** Right-size CPU and memory requests according to observed application baselines.
- **Production consideration:** Implement `LimitRange` objects in namespaces to enforce default and maximum container request constraints.
