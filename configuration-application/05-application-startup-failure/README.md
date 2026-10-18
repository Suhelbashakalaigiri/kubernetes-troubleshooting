# Troubleshooting: Application Startup Failure

## 1. Issue overview
An application startup failure occurs when a container process begins execution successfully, but its internal initialization code crashes or aborts due to missing application-level prerequisites (such as mandatory environment variables, missing directory structures, or invalid configuration flags).

In Kubernetes:
- The scheduler places the Pod onto a node (`PodScheduled: True`).
- Kubelet pulls the image and launches the container runtime process (`Started: True`).
- However, during the application's bootstrapping sequence, the process throws an unhandled error and exits with non-zero status `1`.
- Because the failure occurs inside user space, Kubelet interprets the exit as abnormal and enters `CrashLoopBackOff`.
- **Diagnostic distinction:** This is not a Kubernetes API validation error or a scheduling failure; the failure is strictly within the application's own code and startup checks.

**Expected behavior:** The application discovers the required environment variable `REQUIRED_STORAGE_PATH`, completes bootstrapping, and enters its active service loop.  
**Actual behavior:** The environment variable is missing, prompting the startup script to print a fatal error to stderr and exit with code 1.

---

## 2. Common causes
1. **Missing mandatory environment variables:** The application expects configuration keys that were not supplied in `spec.containers.env`.
2. **Missing required directory paths:** Application assumes `/var/log/app` or `/data` exists, but no volume or `mkdir` initialized it.
3. **Database connection failure on boot:** Application requires a synchronous database connection before opening its web port, crashing if the database is offline.
4. **Corrupt license or credential files:** File exists but fails cryptographic parsing or checksum checks during boot.
5. **Port collision inside container:** Application attempts to bind a socket already in use.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\05-application-startup-failure"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods -l app=startup-failure-demo
```

Observe the transition to `CrashLoopBackOff`:
```text
NAME                                       READY   STATUS             RESTARTS      AGE
deploy-startup-failure-58f79fbc94-4d8zk    0/1     CrashLoopBackOff   2 (10s ago)   30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Check Pod lifecycle phase
```powershell
$pod = (kubectl get pods -l app=startup-failure-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod
```

Examine the **Containers** status block:
```text
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```
*Note: Exit code 1 indicates the process terminated voluntarily due to an internal error.*

### Step 2: Read the application startup logs
Inspect stderr from the crashed container using `--previous`:

```powershell
kubectl logs $pod --previous
```

Output:
```text
Initializing application...
FATAL: REQUIRED_STORAGE_PATH environment variable is missing!
```

**Diagnostic analysis:**
- The container successfully ran `sh`.
- The application script explicitly logged the fatal reason: `REQUIRED_STORAGE_PATH environment variable is missing!`.
- The fix requires supplying the missing environment variable in the Deployment's container specification.

---

## 6. Root cause
In `invalid-deployment.yaml`, the startup script requires:
```sh
if [ -z "$REQUIRED_STORAGE_PATH" ]; then
  echo "FATAL: REQUIRED_STORAGE_PATH environment variable is missing!" >&2
  exit 1
fi
```
The manifest omits `REQUIRED_STORAGE_PATH` from `spec.containers.env`, triggering an intentional initialization failure.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` injects the required variable:
```yaml
env:
- name: REQUIRED_STORAGE_PATH
  value: "/var/data/app"
```

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-startup-failure --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=startup-failure-demo
```

Expected output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
deploy-startup-failure-749d685cbf-9p2xl   1/1     Running   0          20s
```

Verify the container startup logs:

```powershell
$newPod = (kubectl get pods -l app=startup-failure-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod
```

Expected output:
```text
Initializing application...
Starting service on /var/data/app...
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloop-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-fatal-startup-logs.png` — Output of `kubectl logs --previous` showing the `FATAL: ... missing` log message.
- [ ] `03-describe-pod-exit-code-1.png` — Output of `kubectl describe pod` showing `Exit Code: 1`.
- [ ] `04-verified-running-logs.png` — Output of `kubectl logs` verifying `Starting service on /var/data/app...`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-startup-failure
```

---

## 11. Key learning
- **Root cause:** The application aborted startup because a mandatory environment variable (`REQUIRED_STORAGE_PATH`) was absent.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to read stdout and stderr emitted before the process exited.
- **Correction:** Supply the missing configuration in the container's `env` section.
- **Production consideration:** Applications should fail fast with descriptive error messages when essential configuration is missing, and provide sane defaults wherever possible.
