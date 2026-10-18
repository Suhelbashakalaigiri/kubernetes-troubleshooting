# Troubleshooting: Wrong Command or Arguments

## 1. Issue overview
A "Wrong Command or Arguments" issue occurs when a container's `command` (entrypoint) or `args` specification provides invalid, unrecognized, or malformed CLI parameters to the container's executable process, causing it to terminate with a non-zero exit code immediately upon startup.

In Kubernetes:
- `command` overrides the Docker `ENTRYPOINT`.
- `args` overrides the Docker `CMD`.
- If an invalid argument is passed (e.g. `sleep invalid-duration-not-a-number`), the binary's command-line parser fails, outputs an error to `stderr`, and exits with return code 1.
- Under `restartPolicy: OnFailure` or `Always`, Kubelet attempts to restart the container, triggering `CrashLoopBackOff`.

**Expected behavior:** The container command parses valid arguments and executes its background daemon.  
**Actual behavior:** The container executes `sleep invalid-duration-not-a-number`, which fails CLI parsing and exits immediately with code 1.

---

## 2. Common causes
1. **Syntax errors in CLI arguments:** Passing non-numeric values to flags expecting integers or durations.
2. **Flag changes between application versions:** Using flags from a newer or older version of the software.
3. **Misplaced command vs args:** Specifying executable paths in `args` instead of `command`, leading to unexpected argument concatenation.
4. **Incorrect shell quoting:** Shell commands passed without `sh -c` or with broken quotes.
5. **Path issues:** Specifying relative binary paths not in the container's `$PATH`.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\03-wrong-command-args"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Monitor the Pod status:

```powershell
kubectl get pod pod-wrong-args
```

Observe the transition from `Error` to `CrashLoopBackOff`:
```text
NAME             READY   STATUS             RESTARTS      AGE
pod-wrong-args   0/1     CrashLoopBackOff   2 (15s ago)   35s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect container termination state and exit code
```powershell
kubectl describe pod pod-wrong-args
```

Examine the **Containers** status block:
```text
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```

### Step 2: Read container logs
```powershell
kubectl logs pod-wrong-args
```

Output:
```text
sleep: invalid number 'invalid-duration-not-a-number'
```

If the container is currently waiting between restarts, check previous logs:

```powershell
kubectl logs pod-wrong-args --previous
```

Output:
```text
sleep: invalid number 'invalid-duration-not-a-number'
```

### Step 3: Inspect the Pod's configured command and args
```powershell
kubectl get pod pod-wrong-args -o jsonpath='{"Command: "}{.spec.containers[0].command}{"
Args: "}{.spec.containers[0].args}{"
"}'
```

Output:
```text
Command: ["sleep","invalid-duration-not-a-number"]
Args: 
```

**Diagnostic analysis:**
- The process fails with Exit Code 1.
- The stderr log explicitly reports `sleep: invalid number 'invalid-duration-not-a-number'`.
- The root cause is an invalid CLI argument passed to the executable.

---

## 6. Root cause
In `invalid-pod.yaml`, line 12 defines:
```yaml
command: ["sleep", "invalid-duration-not-a-number"]
```
The `sleep` binary requires a numeric duration (e.g. `3600`), but received a non-numeric string.

---

## 7. Fix the issue
Delete the crashing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` provides a valid command:
```yaml
command: ["sh", "-c", "echo 'Application started successfully'; sleep 3600"]
```

---

## 8. Verify the fix
Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-wrong-args --timeout=60s
```

Verify the Pod status:

```powershell
kubectl get pod pod-wrong-args -o wide
```

Expected output:
```text
NAME             READY   STATUS    RESTARTS   AGE
pod-wrong-args   1/1     Running   0          20s
```

Verify the container logs:

```powershell
kubectl logs pod-wrong-args
```

Expected output:
```text
Application started successfully
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloop-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-sleep-invalid-number-logs.png` — Output of `kubectl logs pod-wrong-args` showing `sleep: invalid number`.
- [ ] `03-describe-pod-exitcode-1.png` — Output of `kubectl describe pod` showing `Exit Code: 1`.
- [ ] `04-verified-running-pod.png` — Output of `kubectl logs` and `kubectl get pod` showing healthy running state.

---

## 10. Cleanup
Delete the test Pod:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-wrong-args
```

---

## 11. Key learning
- **Root cause:** The container was passed an invalid non-numeric argument to `sleep`, causing an immediate exit code 1.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to read stdout/stderr from the crashed process.
- **Correction:** Provide valid command-line arguments compatible with the container binary's CLI specification.
- **Production consideration:** Always test custom entrypoint commands in a local container runtime (`docker run --rm <image> <command>`) before committing manifests to Kubernetes.
