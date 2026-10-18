# Troubleshooting: Rollout Rollback

## 1. Issue overview
Deployments maintain a revision history of Pod template modifications. When an update introduces a regression, invalid image, or breaking failure, Kubernetes allows administrators to rapidly undo the rollout and revert to a previously known healthy revision.

In Kubernetes:
- Each time a Deployment's `.spec.template` is changed, the Deployment controller records a new **revision** backed by an underlying ReplicaSet.
- `kubectl rollout undo` reverts the Deployment's template back to a specified prior revision.
- **Critical architectural distinction:**
  - `kubectl rollout undo` rolls back the **Kubernetes Pod template specification** (container images, environment variables, resource limits).
  - It does **NOT** roll back external database migrations, message queue schemas, or stateful external storage changes.

**Expected behavior:** The application is updated safely, but if the new revision fails, operators execute a rollback to restore the stable previous revision.  
**Actual behavior:** Updating to revision 2 introduces an invalid image tag, freezing the rollout until `kubectl rollout undo` is invoked to recover revision 1.

---

## 2. Common causes
1. **Broken application binaries or dependencies in new release:** The new container version crashes or fails regression tests in staging/production.
2. **Missing configuration in new release:** New code requires environment variables or Secrets not present in the cluster.
3. **Unexpected rollout blockage:** New image cannot be pulled or fails readiness probes, stalling the update.
4. **Performance degradation:** New revision consumes excessive memory/CPU under production traffic.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\deployment\03-rollout-rollback"
```

### Step 1: Deploy initial stable revision (Revision 1)
Apply the initial healthy Deployment:

```powershell
kubectl apply -f .\fixed-deployment.yaml
kubectl rollout status deployment/deploy-rollout-rollback --timeout=60s
```

Verify that Revision 1 is active with image `nginx:1.24-alpine`:

```powershell
kubectl get deployment deploy-rollout-rollback
kubectl get pods -l app=rollout-rollback-demo
```

### Step 2: Trigger the faulty update (Revision 2)
Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the rollout:

```powershell
kubectl rollout status deployment/deploy-rollout-rollback --timeout=25s
```

Observe that the rollout stalls:
```text
Waiting for deployment "deploy-rollout-rollback" rollout to finish: 1 out of 2 new replicas have been updated...
error: timed out waiting for the condition
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the rollout history
```powershell
kubectl rollout history deployment/deploy-rollout-rollback
```

Output:
```text
deployment.apps/deploy-rollout-rollback 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### Step 2: Inspect specific revision details
Examine Revision 1:

```powershell
kubectl rollout history deployment/deploy-rollout-rollback --revision=1
```
*Note that Revision 1 uses `Image: nginx:1.24-alpine`.*

Examine Revision 2:

```powershell
kubectl rollout history deployment/deploy-rollout-rollback --revision=2
```
*Note that Revision 2 uses `Image: nginx:bad-tag-v2-broken`.*

### Step 3: Inspect active ReplicaSets and failing Pods
```powershell
kubectl get rs -l app=rollout-rollback-demo
kubectl get pods -l app=rollout-rollback-demo
```

Output:
```text
NAME                                       READY   STATUS             RESTARTS   AGE
deploy-rollout-rollback-6c84b4d6b5-abcd1   1/1     Running            0          2m
deploy-rollout-rollback-6c84b4d6b5-abcd2   1/1     Running            0          2m
deploy-rollout-rollback-7df844cf56-bad01   0/1     ImagePullBackOff   0          45s
```
*Notice that the 2 old Pods from Revision 1 are still running to protect availability, while the new Pod from Revision 2 is stuck in `ImagePullBackOff`.*

---

## 6. Root cause
Revision 2 updated the Pod template to use `image: nginx:bad-tag-v2-broken`. Because the image cannot be pulled, the rollout cannot progress.

---

## 7. Fix the issue: Execute the Rollout Rollback
Instead of manually editing the YAML, instantly revert the Deployment to the proven Revision 1:

```powershell
kubectl rollout undo deployment/deploy-rollout-rollback --to-revision=1
```

Expected output:
```text
deployment.apps/deploy-rollout-rollback rolled back
```

---

## 8. Verify the fix
Monitor the rollback completion:

```powershell
kubectl rollout status deployment/deploy-rollout-rollback --timeout=60s
```

Expected output:
```text
deployment "deploy-rollout-rollback" successfully rolled out
```

Check the rollout history:

```powershell
kubectl rollout history deployment/deploy-rollout-rollback
```

Output:
```text
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```
*Note: Revision 1 was re-applied as Revision 3.*

Verify the running Pods and active image:

```powershell
kubectl get pods -l app=rollout-rollback-demo
kubectl get deployment deploy-rollout-rollback -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected output:
```text
nginx:1.24-alpine
```

All Pods are healthy, ready, and running the verified Revision 1 configuration.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-initial-revision-healthy.png` — Output of `kubectl get deployment` showing healthy Revision 1.
- [ ] `02-rollout-status-stalled.png` — Output of `kubectl rollout status` timing out on Revision 2.
- [ ] `03-rollout-history-comparison.png` — Output of `kubectl rollout history` showing revisions 1 and 2.
- [ ] `04-rollout-undo-executed.png` — Command execution of `kubectl rollout undo`.
- [ ] `05-verified-recovered-rollout.png` — Output of `kubectl get pods` showing healthy recovered state.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-rollout-rollback
```

---

## 11. Key learning
- **Root cause:** A faulty revision blocked the rolling update, requiring a rollback to restore service stability.
- **Most useful diagnostic command:** `kubectl rollout history deployment/<name>` to inspect prior revisions and images.
- **Correction:** Use `kubectl rollout undo deployment/<name> --to-revision=<N>` for immediate recovery.
- **Production consideration:** Annotate Deployments with `kubernetes.io/change-cause` using `kubectl annotate` or deployment manifests to make rollout histories readable and auditable.
