# Troubleshooting: Failed Rollout

## 1. Issue overview
A failed rollout occurs when an updated Deployment specification is submitted to the cluster, but the new ReplicaSet cannot successfully launch and stabilize the desired number of healthy Pods.

In Kubernetes:
- Deployments manage Pod updates through declarative rollout strategies (default: `RollingUpdate`).
- When a rollout starts, the Deployment controller creates a new ReplicaSet and begins scaling it up while gradually scaling down the old ReplicaSet.
- If the new Pods cannot start (for example, due to an invalid image tag, crashing startup command, or missing dependency), the rollout halts in an incomplete state.
- Existing old Pods may remain running to maintain availability, but the Deployment cannot achieve its target revision.

**Expected behavior:** The Deployment creates a new ReplicaSet, pulls the new image, provisions 3 healthy Pods, and completes the rollout.  
**Actual behavior:** The new Pods enter `ImagePullBackOff` due to a nonexistent image tag, causing `kubectl rollout status` to hang indefinitely.

---

## 2. Common causes
1. **Invalid or unavailable container image/tag:** Pod template references a tag or digest that does not exist in the registry.
2. **Missing imagePullSecrets:** New image is in a private registry and credentials are not configured.
3. **Container startup crashes:** Entrypoint or command syntax errors trigger `CrashLoopBackOff` in new Pods.
4. **Failing readiness probes:** New Pods run but never pass readiness checks, preventing the Deployment controller from proceeding with rollout.
5. **Resource quota exhaustion:** The cluster has insufficient CPU or memory to create the surge Pods requested by the rolling update.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\deployment\01-failed-rollout"
```

Apply the invalid Deployment manifest:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Track the rollout status in real time:

```powershell
kubectl rollout status deployment/deploy-failed-rollout --timeout=30s
```

Observe that the rollout fails to complete and times out:
```text
Waiting for deployment "deploy-failed-rollout" rollout to finish: 0 of 3 updated replicas are available...
error: timed out waiting for the condition
```

Check the Pod status:

```powershell
kubectl get pods -l app=failed-rollout-demo
```

Output:
```text
NAME                                     READY   STATUS             RESTARTS   AGE
deploy-failed-rollout-7d5f8b9b4c-8j7k2   0/1     ImagePullBackOff   0          45s
```

---

## 5. Identify and diagnose the issue

Follow this diagnostic sequence to uncover the rollout blocker:

### Step 1: Inspect Deployment status and conditions
```powershell
kubectl describe deployment deploy-failed-rollout
```

Check the **Conditions** section:
```text
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    True    ReplicaSetUpdated
```
Notice `Replicas: 3 desired | 1 updated | 1 total | 0 available | 1 unavailable`. The Deployment attempted to surge 1 replica, but it is not available.

### Step 2: Identify active ReplicaSets
```powershell
kubectl get rs -l app=failed-rollout-demo
```

Output:
```text
NAME                               DESIRED   CURRENT   READY   AGE
deploy-failed-rollout-7d5f8b9b4c   1         1         0       60s
```

### Step 3: Inspect Pod events in the failing ReplicaSet
Find the failing Pod name and describe it:

```powershell
$podName = (kubectl get pods -l app=failed-rollout-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $podName
```

Examine the **Events** section:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  1m                 default-scheduler  Successfully assigned default/deploy-failed-rollout-... to minikube
  Normal   Pulling    1m                 kubelet            Pulling image "nginx:nonexistent-tag-rollout-9999"
  Warning  Failed     58s                kubelet            Failed to pull image "nginx:nonexistent-tag-rollout-9999": ... manifest unknown
  Warning  Failed     58s                kubelet            Error: ErrImagePull
  Normal   BackOff    45s (x2 over 57s)  kubelet            Back-off pulling image "nginx:nonexistent-tag-rollout-9999"
  Warning  Failed     45s (x2 over 57s)  kubelet            Error: ImagePullBackOff
```

**Diagnostic analysis:**
- The Deployment cannot complete rollout because its new ReplicaSet is blocked by an image pull error.
- The Kubelet explicitly reports `manifest unknown` for image `nginx:nonexistent-tag-rollout-9999`.

---

## 6. Root cause
In `invalid-deployment.yaml`, line 20 specifies `image: nginx:nonexistent-tag-rollout-9999`. Because this tag does not exist, the newly created Pods cannot start, halting the rollout.

---

## 7. Fix the issue
Apply the corrected Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` points to `nginx:1.25-alpine`, a valid, lightweight public image.

---

## 8. Verify the fix
Monitor the rollout until completion:

```powershell
kubectl rollout status deployment/deploy-failed-rollout --timeout=60s
```

Expected output:
```text
deployment "deploy-failed-rollout" successfully rolled out
```

Check that all 3 replicas are running and ready:

```powershell
kubectl get deployment deploy-failed-rollout
kubectl get pods -l app=failed-rollout-demo
```

Expected output:
```text
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deploy-failed-rollout   3/3     3            3           2m
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-rollout-timeout.png` — Output of `kubectl rollout status` timing out.
- [ ] `02-deployment-describe-conditions.png` — Output of `kubectl describe deployment` showing `MinimumReplicasUnavailable`.
- [ ] `03-failing-pod-events.png` — Output of `kubectl describe pod` showing `manifest unknown` and `ImagePullBackOff`.
- [ ] `04-verified-rollout-success.png` — Output of `kubectl rollout status` showing successfully rolled out and `3/3 Ready`.

---

## 10. Cleanup
Remove the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-failed-rollout
```

---

## 11. Key learning
- **Root cause:** An invalid image tag halted the rollout during the rolling update surge phase.
- **Most useful diagnostic command:** `kubectl rollout status deployment/<name>` to identify stuck rollouts, followed by inspecting the newly created ReplicaSet's Pods.
- **Correction:** Update the Pod template with a valid, accessible image tag.
- **Production consideration:** Always set a realistic `progressDeadlineSeconds` on Deployments and run pre-deployment image tag checks in your CI/CD pipeline.
