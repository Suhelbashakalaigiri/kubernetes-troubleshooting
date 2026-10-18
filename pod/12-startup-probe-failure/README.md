# Troubleshooting: Startup Probe Failure

## 1. Issue overview
A startup probe failure occurs when a container with a defined `startupProbe` fails to pass its startup check within the maximum time window budgeted by its probe configuration.

In Kubernetes:
- The `startupProbe` was introduced specifically to protect slow-starting containers (e.g. legacy Java apps, large machine learning models, or apps running database migrations on startup) from being prematurely terminated by liveness probes.
- When a startup probe is defined, **all liveness and readiness probes are disabled** until the startup probe successfully succeeds.
- If the startup probe does not succeed within its total budgeted time:
  $$\text{Total Startup Budget} = \text{initialDelaySeconds} + (\text{periodSeconds} \times \text{failureThreshold})$$
  the Kubelet considers the startup failed, kills the container, and initiates a restart.
- Because the container is restarted every time before it finishes initializing, it is trapped in an infinite startup loop.

**Expected behavior:** The startup probe allows sufficient time for container bootstrapping, detects the completion marker (`/tmp/started`), and hands over lifecycle checks to readiness/liveness probes.  
**Actual behavior:** The application requires 15 seconds to initialize, but the startup probe budget allows only 5 seconds ($1\text{s} + 2\text{s} \times 2 = 5\text{s}$), causing Kubelet to kill the container at 5 seconds repeatedly.

---

## 2. Common causes
1. **Inadequate startup budget:** Setting `failureThreshold` or `periodSeconds` too low for cold-start initialization or heavy startup routines.
2. **Slow external dependency connection during startup:** Database connection pool initialization or remote configuration fetch is slow.
3. **Large dataset download or decompression:** Initial download or extraction of models, assets, or databases takes longer than the budgeted time.
4. **Heavy garbage collection or CPU throttling during boot:** JIT compilation or CPU limits slow down initial startup phase.
5. **Incorrect probe command or path:** Probe checks for a file or HTTP route that is never created or created in a different directory.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod
-startup-probe-failure"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Monitor the Pod status over 20 to 35 seconds:

```powershell
kubectl get pods pod-startup-probe-failure
```

Observe that the container restarts repeatedly without ever becoming Ready:
```text
NAME                         READY   STATUS    RESTARTS      AGE
pod-startup-probe-failure    0/1     Running   2 (10s ago)   30s
```

After several failures, Kubelet enters backoff:
```text
NAME                         READY   STATUS             RESTARTS      AGE
pod-startup-probe-failure    0/1     CrashLoopBackOff   3 (15s ago)   50s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and restart reasons
```powershell
kubectl describe pod pod-startup-probe-failure
```

Examine the **Containers** probe configuration:
```text
Startup:  exec [cat /tmp/started] delay=1s timeout=1s period=2s #success=1 #failure=2
```
*Note the total budget:* $1\text{s} + (2\text{s} \times 2) = 5\text{s}$.

Examine the **Events** table:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  45s                default-scheduler  Successfully assigned default/pod-startup-probe-failure to minikube
  Normal   Pulled     44s                kubelet            Container image "busybox:1.36" already present on machine
  Normal   Created    44s                kubelet            Created container slow-app
  Normal   Started    44s                kubelet            Started container slow-app
  Warning  Unhealthy  12s (x6 over 38s)  kubelet            Startup probe failed: cat: can't open '/tmp/started': No such file or directory
  Normal   Killing    12s (x2 over 32s)  kubelet            Container slow-app failed startup probe, will be restarted
```

### Step 2: Inspect previous container logs
```powershell
kubectl logs pod-startup-probe-failure --previous
```

Output:
```text
Application bootstrapping...
```
*Notice that `'Application online'` was never printed because the container was killed at 5 seconds, while the script needed 15 seconds.*

### Step 3: Calculate the required startup budget
The application takes 15 seconds (`sleep 15; touch /tmp/started`).  
The invalid probe configuration allowed:
$$\text{initialDelaySeconds} (1) + \text{periodSeconds} (2) \times \text{failureThreshold} (2) = 5\text{ seconds}$$
Since $5\text{s} < 15\text{s}$, the container is killed every time before it can ever pass the probe.

**Diagnostic analysis:**
- The event explicitly states: `Startup probe failed: cat: can't open '/tmp/started': No such file or directory` followed by `Container slow-app failed startup probe, will be restarted`.
- Logs prove the application was actively bootstrapping and had not crashed on its own.
- The root cause is a startup probe budget that is shorter than the application's genuine initialization time.

---

## 6. Root cause
In `invalid-pod.yaml`, lines 18-21 specify:
```yaml
startupProbe:
  exec:
    command: ["cat", "/tmp/started"]
  initialDelaySeconds: 1
  periodSeconds: 2
  failureThreshold: 2
```
The startup budget is 5 seconds, but the container workload requires 15 seconds to create `/tmp/started`. Kubelet kills the container at second 5 on every restart attempt.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` increases `periodSeconds: 3` and `failureThreshold: 10`, providing a total startup budget of:
$$2\text{s} + (3\text{s} \times 10) = 32\text{ seconds}$$
This allows comfortable headroom for the 15-second bootstrap process to complete.

---

## 8. Verify the fix
Wait for the Pod to complete startup and become `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-startup-probe-failure --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-startup-probe-failure -o wide
```

Expected output:
```text
NAME                         READY   STATUS    RESTARTS   AGE
pod-startup-probe-failure    1/1     Running   0          25s
```

Verify the container logs to confirm successful startup:

```powershell
kubectl logs pod-startup-probe-failure
```

Expected output:
```text
Application bootstrapping...
Application online
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-restarting-startup-status.png` — Output of `kubectl get pods` showing incrementing restarts during startup.
- [ ] `02-startup-failed-events.png` — Output of `kubectl describe pod` showing `Startup probe failed ... can't open '/tmp/started'` and `Container slow-app failed startup probe, will be restarted`.
- [ ] `03-logs-incomplete-bootstrap.png` — Output of `kubectl logs --previous` showing initialization cut off before completion.
- [ ] `04-verified-online-pod.png` — Output of `kubectl logs` and `kubectl get pod` showing `Application online` and `STATUS: Running` with 0 restarts.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-startup-probe-failure
```

---

## 11. Key learning
- **Root cause:** The startup probe budget ($1\text{s} + 2\text{s} \times 2 = 5\text{s}$) was shorter than the application's bootstrap time (15s), causing Kubelet to terminate the container before it could initialize.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to calculate the probe budget ($initialDelay + period \times failureThreshold$) and compare it with the time required by the application.
- **Correction:** Increase `failureThreshold` to allocate sufficient time for worst-case cold starts without modifying the liveness probe parameters.
- **Production consideration:** Use startup probes alongside liveness probes for slow-starting applications: configure the startup probe with a high `failureThreshold` to accommodate slow bootup, while keeping the liveness probe with a short timeout to catch real deadlocks quickly once running.
