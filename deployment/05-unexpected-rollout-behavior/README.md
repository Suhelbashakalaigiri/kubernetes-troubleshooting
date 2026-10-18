# Troubleshooting: Unexpected Rollout Behavior (Aggressive RollingUpdate Outage)

## 1. Issue overview
Deployments allow fine-grained control over rolling updates through `strategy.rollingUpdate.maxSurge` and `strategy.rollingUpdate.maxUnavailable`. Misconfiguring these parameters can cause unexpected rollout dynamics, such as total service outages during updates.

In Kubernetes:
- `maxSurge`: Maximum number or percentage of Pods that can be created above `spec.replicas`.
- `maxUnavailable`: Maximum number or percentage of Pods that can be unavailable during the update process.
- When `maxUnavailable: 100%` and `maxSurge: 0` are configured, the Deployment controller terminates **ALL existing Pods simultaneously** before creating new Pods.
- This inadvertently transforms what was intended as a zero-downtime rolling update into an aggressive destructive update, dropping all active user connections.

**Expected behavior:** Updates roll out incrementally with zero downtime, maintaining at least 100% of desired capacity throughout the update.  
**Actual behavior:** Setting `maxUnavailable: 100%` and `maxSurge: 0` drops available replicas to 0 immediately upon updating the Pod template.

---

## 2. Common causes
1. **Misunderstanding `maxUnavailable: 100%`:** Setting `maxUnavailable` to 100% under the mistaken assumption that it means "all pods should be updated quickly".
2. **Setting both `maxSurge: 0` and `maxUnavailable: 0`:** Kubernetes API rejects this combination because no progress can be made.
3. **Missing readiness probes during rolling updates:** New Pods are counted as available before they are ready to accept traffic, terminating old Pods prematurely.
4. **Zero-replica scaling during rollouts:** Unintended HPA or external controller scaling during an active update.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\deployment\05-unexpected-rollout-behavior"
```

### Step 1: Deploy initial state with 4 replicas
Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
kubectl rollout status deployment/deploy-unexpected-rollout --timeout=60s
```

Verify that 4 replicas are active:

```powershell
kubectl get deployment deploy-unexpected-rollout
```

Output:
```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deploy-unexpected-rollout    4/4     4            4           20s
```

### Step 2: Trigger a template update and observe unexpected downtime
Update the container image:

```powershell
kubectl set image deployment/deploy-unexpected-rollout web=nginx:1.25-alpine
```

Immediately monitor the Deployment status:

```powershell
kubectl get deployment deploy-unexpected-rollout
```

Observe that available replicas instantly dropped to 0:
```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deploy-unexpected-rollout    0/4     4            0           35s
```
*Notice that READY is 0/4 and AVAILABLE is 0. All 4 running Pods were terminated at once!*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the rollingUpdate strategy configuration
```powershell
kubectl get deployment deploy-unexpected-rollout -o jsonpath='{.spec.strategy.rollingUpdate}'
```

Output:
```text
{"maxSurge":0,"maxUnavailable":"100%"}
```

### Step 2: Inspect Deployment events
```powershell
kubectl describe deployment deploy-unexpected-rollout
```

Examine the **Events** section:
```text
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  30s   deployment-controller  Scaled down replica set deploy-unexpected-rollout-... to 0
  Normal  ScalingReplicaSet  30s   deployment-controller  Scaled up replica set deploy-unexpected-rollout-... to 4
```

**Diagnostic analysis:**
- Because `maxSurge: 0`, the controller cannot create any new Pods while 4 old Pods exist.
- Because `maxUnavailable: 100%`, the controller is permitted to terminate all 4 old Pods at once.
- The controller scaled down the old ReplicaSet to 0 *before* the new ReplicaSet could become ready, causing a complete application outage.

---

## 6. Root cause
In `invalid-deployment.yaml`, lines 12-14 specify:
```yaml
rollingUpdate:
  maxUnavailable: 100%
  maxSurge: 0
```
This forces all existing Pods to terminate simultaneously before new Pods are initialized.

---

## 7. Fix the issue
Apply the corrected Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` establishes safe zero-downtime rolling update parameters:
```yaml
rollingUpdate:
  maxUnavailable: 0
  maxSurge: 1
```
- `maxUnavailable: 0` guarantees that all 4 baseline replicas remain running and serving traffic at all times during updates.
- `maxSurge: 1` provisions 1 new Pod at a time above the replica count, terminating an old Pod only after the new Pod passes readiness.

---

## 8. Verify the fix
Monitor the update with the new strategy:

```powershell
kubectl rollout status deployment/deploy-unexpected-rollout --timeout=60s
```

Check the Deployment metrics:

```powershell
kubectl get deployment deploy-unexpected-rollout
```

Expected output:
```text
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deploy-unexpected-rollout    4/4     4            4           2m
```

Verify that the strategy parameters are now safe:

```powershell
kubectl get deployment deploy-unexpected-rollout -o jsonpath='{.spec.strategy.rollingUpdate}'
```

Output:
```text
{"maxSurge":1,"maxUnavailable":0}
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-initial-4-replicas.png` — Output of `kubectl get deployment` showing 4/4 Ready.
- [ ] `02-zero-available-outage.png` — Output showing `AVAILABLE: 0` immediately after updating image with `maxUnavailable: 100%`.
- [ ] `03-deployment-events-scaledown.png` — Output of `kubectl describe deployment` showing old ReplicaSet scaled to 0 before new was ready.
- [ ] `04-verified-safe-strategy.png` — Output of jsonpath showing `maxSurge: 1, maxUnavailable: 0` and `4/4 Ready`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-unexpected-rollout
```

---

## 11. Key learning
- **Root cause:** `maxUnavailable: 100%` with `maxSurge: 0` permitted the controller to terminate all existing Pods simultaneously, causing a full outage during updates.
- **Most useful diagnostic command:** `kubectl describe deployment <name>` and jsonpath inspection of `.spec.strategy.rollingUpdate`.
- **Correction:** Configure `maxUnavailable: 0` and `maxSurge: 1` (or `25%` / `25%`) to guarantee zero-downtime rolling updates.
- **Production consideration:** In mission-critical environments, pair `maxUnavailable: 0` with proper `readinessProbe` and `minReadySeconds: 10` to verify container stability before proceeding with rolling updates.
