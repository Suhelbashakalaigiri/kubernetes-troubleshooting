# Troubleshooting: Deployment Not Progressing (ProgressDeadlineExceeded)

## 1. Issue overview
`ProgressDeadlineExceeded` is a Deployment condition that occurs when a Deployment rollout fails to make observable progress toward completion within its configured `progressDeadlineSeconds`.

In Kubernetes:
- The Deployment controller expects rollouts to make continuous forward progress (e.g., creating Pods, completing pulls, passing readiness checks).
- `progressDeadlineSeconds` defines the maximum time the controller will wait for a rollout to complete before marking the condition `Type: Progressing, Status: False, Reason: ProgressDeadlineExceeded`.
- Even after the deadline expires, Kubernetes does **NOT** automatically roll back or delete the Deployment; it simply reports the failure condition so automated CI/CD systems or monitoring alerts can take corrective action.

**Expected behavior:** The rollout completes well within `progressDeadlineSeconds`, resulting in `Type: Progressing, Status: True, Reason: NewReplicaSetAvailable`.  
**Actual behavior:** The container command exits with code 1 (`CrashLoopBackOff`), and because `progressDeadlineSeconds: 30` is configured, the Deployment controller flags `ProgressDeadlineExceeded` after 30 seconds.

---

## 2. Common causes
1. **Aggressive `progressDeadlineSeconds`:** Deadline set too low for images requiring long download times or complex startup routines.
2. **Persistent container crashes:** New Pods enter `CrashLoopBackOff` immediately upon start.
3. **Unschedulable surge Pods:** Insufficient cluster CPU or memory prevents scheduling new replicas within the deadline.
4. **Persistent volume binding delays:** Storage provisioner delays exceed the progress deadline.
5. **Image pull backoff:** Image registry rate-limiting or network latency exceeds the deadline.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\deployment\04-deployment-not-progressing"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Wait 35 to 45 seconds for the 30-second progress deadline to expire, then check the rollout status:

```powershell
kubectl rollout status deployment/deploy-not-progressing
```

Observe the deadline error:
```text
error: deployment "deploy-not-progressing" exceeded its progress deadline
```

Check the Deployment summary:

```powershell
kubectl get deployment deploy-not-progressing
```

Output:
```text
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deploy-not-progressing   0/2     2            0           50s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Deployment conditions for ProgressDeadlineExceeded
```powershell
kubectl describe deployment deploy-not-progressing
```

Examine the **Conditions** block:
```text
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      False   MinimumReplicasUnavailable
  Progressing    False   ProgressDeadlineExceeded
```

Look at the **Events** section:
```text
Events:
  Type     Reason                    Age   From                   Message
  ----     ------                    ----  ----                   -------
  Normal   ScalingReplicaSet         55s   deployment-controller  Scaled up replica set deploy-not-progressing-... to 2
  Warning  ProgressDeadlineExceeded  15s   deployment-controller  ReplicaSet "deploy-not-progressing-..." has timed out progressing.
```

### Step 2: Extract the Progressing condition message via jsonpath
```powershell
kubectl get deployment deploy-not-progressing -o jsonpath='{.status.conditions[?(@.type=="Progressing")].message}'
```

Expected output:
```text
ReplicaSet "deploy-not-progressing-..." has timed out progressing.
```

### Step 3: Inspect Pods to identify the underlying failure
```powershell
kubectl get pods -l app=not-progressing-demo
$pod = (kubectl get pods -l app=not-progressing-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod
```

Output:
```text
Failing startup task...
```
*The container process executes `exit 1`, causing an immediate crash loop that prevents the rollout from progressing.*

**Diagnostic analysis:**
- `ProgressDeadlineExceeded` is a symptom that signals the rollout took longer than `progressDeadlineSeconds: 30`.
- Inspecting the underlying Pod logs reveals the root cause: the container crashes on startup with `exit 1`.

---

## 6. Root cause
In `invalid-deployment.yaml`:
1. Line 9 specifies `progressDeadlineSeconds: 30`, setting an aggressive timeout.
2. Lines 21-22 execute a command that immediately exits with code 1:
   ```yaml
   command: ["sh", "-c", "echo 'Failing startup task...'; exit 1"]
   ```
The continuous container crash prevents the Deployment from satisfying its readiness requirements before the 30-second progress deadline expires.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` replaces the crashing command with a healthy `nginx:1.25-alpine` web server and restores the standard production `progressDeadlineSeconds: 600` (10 minutes).

---

## 8. Verify the fix
Monitor the rollout until completion:

```powershell
kubectl rollout status deployment/deploy-not-progressing --timeout=60s
```

Expected output:
```text
deployment "deploy-not-progressing" successfully rolled out
```

Verify that the `Progressing` condition has returned to `True`:

```powershell
kubectl get deployment deploy-not-progressing -o jsonpath='{"Type: "}{.status.conditions[?(@.type=="Progressing")].type}{"
Status: "}{.status.conditions[?(@.type=="Progressing")].status}{"
Reason: "}{.status.conditions[?(@.type=="Progressing")].reason}{"
"}'
```

Output:
```text
Type: Progressing
Status: True
Reason: NewReplicaSetAvailable
```

Check the active Pods:

```powershell
kubectl get pods -l app=not-progressing-demo
```

Output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
deploy-not-progressing-5c7478f77d-7j9xk   1/1     Running   0          25s
deploy-not-progressing-5c7478f77d-k8m2p   1/1     Running   0          25s
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-progress-deadline-timeout.png` — Output of `kubectl rollout status` showing `exceeded its progress deadline`.
- [ ] `02-deployment-describe-deadline-exceeded.png` — Output of `kubectl describe deployment` highlighting `ProgressDeadlineExceeded`.
- [ ] `03-crashing-pod-logs.png` — Output of `kubectl logs` showing the failing startup task.
- [ ] `04-verified-progressing-true.png` — Output of `kubectl rollout status` and jsonpath showing `Progressing: True` and `NewReplicaSetAvailable`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-not-progressing
```

---

## 11. Key learning
- **Root cause:** The container crashed continuously, exceeding the configured 30-second `progressDeadlineSeconds`.
- **Most useful diagnostic command:** `kubectl describe deployment <name>` to check for `ProgressDeadlineExceeded`, followed by `kubectl logs` on the active ReplicaSet's Pods.
- **Correction:** Fix the application crash and set a realistic `progressDeadlineSeconds` (default: 600s).
- **Production consideration:** Use `ProgressDeadlineExceeded` as a trigger in automated CD tools (ArgoCD, Spinnaker, Flux) to automatically alert on or rollback stalled deployments.
