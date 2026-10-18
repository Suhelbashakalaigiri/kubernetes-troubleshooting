# Troubleshooting: OOMKilled (Out of Memory)

## 1. Issue overview
`OOMKilled` (Out of Memory Killed) is a container termination condition in Kubernetes indicating that the Linux kernel Out-Of-Memory killer terminated a container process because its physical memory consumption exceeded its assigned cgroup memory limit (`resources.limits.memory`) or the node itself ran out of memory.

In Kubernetes:
- When memory limits are specified, Kubelet configures memory control groups (cgroups) for the container.
- If the processes inside the container exceed the cgroup limit, the Linux kernel triggers the cgroup OOM killer, sending `SIGKILL` (signal 9) to the highest-consuming process.
- The process terminates with **Exit Code 137** (128 + 9 for SIGKILL).
- Kubelet records the container termination state with `Reason: OOMKilled`.
- If `restartPolicy: Always` is set, the container will repeatedly crash and restart into `CrashLoopBackOff`. With `restartPolicy: Never`, the Pod remains in `OOMKilled` for straightforward inspection.

**Expected behavior:** The container operates within its configured memory boundaries, finishes memory allocation, and continues running.  
**Actual behavior:** The container allocates 50Mi of RAM via a tmpfs buffer inside a cgroup limited to 20Mi, triggering an immediate kernel OOM termination with exit code 137.

---

## 2. Common causes
1. **Underestimated memory limits:** Setting `resources.limits.memory` lower than standard application baseline or JVM heap requirements.
2. **Memory leaks:** Application fails to release unused objects, accumulating memory until the limit is breached.
3. **Spikes in concurrent workload:** High traffic or large batch payloads requiring transient in-memory buffering.
4. **JVM or runtime misconfiguration:** Runtimes (like Java) not recognizing cgroup v1/v2 memory limits and sizing heap based on total host physical RAM.
5. **Node-level memory pressure:** The node itself runs low on memory, forcing the kernel to evict or terminate high-memory containers.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-oomkilled"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods pod-oomkilled
```

Within a few seconds, the Pod transitions to `OOMKilled`:
```text
NAME            READY   STATUS      RESTARTS   AGE
pod-oomkilled   0/1     OOMKilled   0          12s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect container termination state and exit code
```powershell
kubectl describe pod pod-oomkilled
```

Examine the **Containers** status section:
```text
State:          Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
Ready:          False
Restart Count:  0
Limits:
  memory:  20Mi
Requests:
  memory:  10Mi
```

### Step 2: Check Pod events
```powershell
kubectl get events --field-selector involvedObject.name=pod-oomkilled --sort-by=.metadata.creationTimestamp
```

Observe container creation, execution, and subsequent termination:
```text
Normal   Scheduled  25s   default-scheduler  Successfully assigned default/pod-oomkilled to minikube
Normal   Pulled     24s   kubelet            Container image "busybox:1.36" already present on machine
Normal   Created    24s   kubelet            Created container memory-consumer
Normal   Started    24s   kubelet            Started container memory-consumer
```

### Step 3: Check container logs prior to termination
```powershell
kubectl logs pod-oomkilled
```

Output:
```text
Allocating memory beyond cgroup limit...
```
*Notice that the container printed the initial message, but never printed `'Done'` because the kernel terminated it midway through memory allocation.*

### Step 4: Extract exit code and termination reason via jsonpath
```powershell
kubectl get pod pod-oomkilled -o jsonpath='{"Reason: "}{.status.containerStatuses[0].state.terminated.reason}{"
ExitCode: "}{.status.containerStatuses[0].state.terminated.exitCode}{"
"}'
```

Output:
```text
Reason: OOMKilled
ExitCode: 137
```

**Diagnostic analysis:**
- Exit code 137 is calculated as `128 + 9` (`SIGKILL`), indicating the process was forcefully killed from outside user space.
- The `Reason: OOMKilled` attribute explicitly confirms that the kernel cgroup controller, rather than an application-level `exit()`, terminated the container due to memory boundary enforcement.

---

## 6. Root cause
In `invalid-pod.yaml`, the container specifies a memory limit of `20Mi`:
```yaml
resources:
  limits:
    memory: "20Mi"
```
The container command attempts to write 50 Megabytes of data to `/dev/shm` (shared memory / tmpfs), which is stored directly in physical RAM and accounted against the cgroup memory limit. The Linux kernel OOM killer triggers when usage hits the 20Mi threshold.

---

## 7. Fix the issue
Delete the terminated Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` increases the memory limit to `64Mi` (with a `32Mi` request) and performs a safe `10M` allocation, ensuring memory usage stays well below the cgroup limit.

---

## 8. Verify the fix
Wait for the Pod to initialize:

```powershell
kubectl wait --for=condition=Ready pod/pod-oomkilled --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-oomkilled -o wide
```

Expected output:
```text
NAME            READY   STATUS    RESTARTS   AGE
pod-oomkilled   1/1     Running   0          22s
```

Inspect the container logs to verify completion:

```powershell
kubectl logs pod-oomkilled
```

Expected output:
```text
Allocating memory safely within limit...
Allocation successful
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-oomkilled-status.png` — Output of `kubectl get pods` showing `STATUS: OOMKilled`.
- [ ] `02-describe-pod-exitcode-137.png` — Output of `kubectl describe pod pod-oomkilled` showing `Reason: OOMKilled` and `Exit Code: 137`.
- [ ] `03-logs-partial-execution.png` — Output of `kubectl logs pod-oomkilled` showing the log message cut off at memory allocation.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-oomkilled` showing `STATUS: Running` with 0 restarts.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-oomkilled
```

---

## 11. Key learning
- **Root cause:** The container's memory consumption exceeded its cgroup memory limit (`20Mi`), causing the Linux kernel to terminate the process with SIGKILL (Exit code 137).
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to verify `Reason: OOMKilled` and `Exit Code: 137`.
- **Correction:** Right-size memory limits based on actual application profiling or optimize memory usage inside the application.
- **Production consideration:** Set memory requests to average baseline consumption and limits to peak safe tolerance, and monitor container memory usage using Prometheus or metrics-server (`kubectl top pod`).
