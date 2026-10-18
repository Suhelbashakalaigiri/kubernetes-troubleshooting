# Troubleshooting: CrashLoopBackOff

## 1. Issue overview
`CrashLoopBackOff` is a container waiting state in Kubernetes where a container repeatedly starts, crashes (exits with a non-zero code or terminates prematurely), and is restarted by the Kubelet according to the Pod's `restartPolicy`.

In Kubernetes:
- `CrashLoopBackOff` is **not a Pod phase**; the Pod phase (`status.phase`) is typically `Running`.
- When the container process terminates unexpectedly, Kubelet restarts it. To prevent resource thrashing on the node, Kubelet applies an exponential backoff delay before each restart: 10s, 20s, 40s, 80s, 160s, capping at 300s (5 minutes).
- While waiting between restart attempts, the container's waiting reason is reported as `CrashLoopBackOff`.

**Expected behavior:** The container process initializes, enters its active event loop or server daemon, and remains running indefinitely.  
**Actual behavior:** The container process exits immediately with exit code 1, triggering Kubelet restart backoff.

---

## 2. Common causes
1. **Application runtime exceptions:** Uncaught exceptions, missing environment variables, or corrupt configuration files during initialization.
2. **Missing dependencies or command binaries:** Entrypoint script attempts to call a command or path that does not exist.
3. **Non-daemon / batch process in a web Pod:** Running a script that finishes and exits (exit code 0), but `restartPolicy: Always` forces Kubelet to restart it continuously.
4. **Port binding conflicts:** The process tries to bind to a network port already in use on the container or host.
5. **Memory or storage exhaustion:** Process killed by internal signals before completing startup.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue directory in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-crashloopbackoff"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Monitor the Pod status over 30 to 45 seconds:

```powershell
kubectl get pods pod-crashloopbackoff
```

Observe the transition from `Error` to `CrashLoopBackOff`, with the `RESTARTS` count incrementing:
```text
NAME                   READY   STATUS             RESTARTS      AGE
pod-crashloopbackoff   0/1     CrashLoopBackOff   2 (20s ago)   45s
```

---

## 5. Identify and diagnose the issue

Follow this diagnostic sequence to uncover the application-level crash:

### Step 1: Inspect container status and termination details
```powershell
kubectl describe pod pod-crashloopbackoff
```

Look at the **Containers** status block:
```text
State:          Terminated
  Reason:       Error
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
Ready:          False
Restart Count:  2
```
*Note that `Last State` indicates `Terminated` with `Exit Code: 1`.*

### Step 2: Retrieve container logs from the crashed instance
Because the container is currently waiting in backoff, standard `kubectl logs` may be empty or show the latest invocation. Use the `--previous` flag to inspect the logs of the container instance that just crashed:

```powershell
kubectl logs pod-crashloopbackoff --previous
```

Expected diagnostic output:
```text
Starting worker service...
Fatal error: configuration file missing
```

### Step 3: Check Pod phase vs Container state
```powershell
kubectl get pod pod-crashloopbackoff -o jsonpath='{"PodPhase: "}{.status.phase}{"
ContainerState: "}{.status.containerStatuses[0].state.waiting.reason}{"
ExitCode: "}{.status.containerStatuses[0].lastState.terminated.exitCode}{"
"}'
```

**Diagnostic analysis:**
- The Pod phase is `Running`, but the container is in `CrashLoopBackOff`.
- The `--previous` log reveals the root cause: the process executed an `exit 1` after printing `Fatal error: configuration file missing`.
- This proves the failure is inside the application process logic, not an infrastructure or image-pull issue.

---

## 6. Root cause
In `invalid-pod.yaml`, the container command executes:
```sh
sh -c "echo 'Starting worker service...'; echo 'Fatal error: configuration file missing'; exit 1"
```
The script deliberately exits with non-zero status `1`. Under the default `restartPolicy: Always`, Kubelet treats termination as abnormal and restarts the container with exponential backoff.

---

## 7. Fix the issue
Delete the crashing Pod and deploy the corrected manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` provides a persistent, healthy background loop (`while true; do echo 'Worker heartbeat'; sleep 30; done`) that maintains an active PID 1 process and does not exit.

---

## 8. Verify the fix
Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-crashloopbackoff --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-crashloopbackoff -o wide
```

Expected output:
```text
NAME                   READY   STATUS    RESTARTS   AGE
pod-crashloopbackoff   1/1     Running   0          20s
```

Inspect the live container logs to confirm ongoing execution:

```powershell
kubectl logs pod-crashloopbackoff
```

Output:
```text
Starting worker service...
Configuration loaded successfully
Worker heartbeat
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloopbackoff-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff` and non-zero restarts.
- [ ] `02-describe-pod-laststate.png` — Output of `kubectl describe pod pod-crashloopbackoff` highlighting `Last State: Terminated` with `Exit Code: 1`.
- [ ] `03-logs-previous.png` — Output of `kubectl logs pod-crashloopbackoff --previous` showing the fatal error message.
- [ ] `04-verified-running.png` — Output of `kubectl get pod pod-crashloopbackoff` showing `STATUS: Running` with 0 restarts.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Verify deletion:

```powershell
kubectl get pod pod-crashloopbackoff
```

---

## 11. Key learning
- **Root cause:** The container process exited with return code `1` immediately upon startup.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to read stdout/stderr from the terminated container instance before the restart backoff.
- **Correction:** Ensure container entrypoints execute long-running services or valid daemon processes without exiting unexpectedly.
- **Production consideration:** Use robust error logging and structured liveness/readiness probes, and ensure configuration prerequisites exist before starting main worker loops.
