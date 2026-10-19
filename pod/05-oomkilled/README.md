
# Troubleshooting: OOMKilled (Out of Memory)

## 1. Issue Overview

`OOMKilled` is a container termination reason reported when a container process is killed due to an out-of-memory condition.

In this exercise, the container attempts to retain a large string in memory while operating under a limited memory budget.

When a container exceeds its applicable cgroup memory limit, the Linux kernel can kill a process in that cgroup. Kubernetes reports the termination reason as `OOMKilled`.

An exit code of `137` is consistent with termination by `SIGKILL` (128 + 9), but exit code 137 alone does not prove an OOM kill. Check the termination reason as well.

**Expected behavior:** The invalid container starts, attempts to allocate memory, and is terminated before completing the allocation.

**Actual behavior:** The container terminates with `Reason: OOMKilled` and `Exit Code: 137`.

---

## 2. Common Causes

1. **Insufficient memory limits:** The configured memory limit is too low for the application's workload.
2. **Memory leaks:** The application retains memory it no longer needs.
3. **Traffic or workload spikes:** Concurrent requests or large payloads require additional memory.
4. **Runtime configuration:** Runtime memory settings, such as JVM heap sizing, do not fit within the container limit.
5. **Node memory pressure:** The node runs low on memory, potentially causing evictions or kernel OOM events.

---

## 3. Prerequisites

- Minikube installed and running.
- `kubectl` configured for Minikube.
- Windows PowerShell.
- `invalid-pod.yaml` and `fixed-pod.yaml` in this directory.

Verify the cluster:

```powershell
minikube status
kubectl get nodes
```

The Minikube node should be `Ready`.

---

## 4. Reproduce the Issue

Navigate to the exercise directory:

```powershell
Set-Location -Path ".\pod\05-oomkilled"
```

If you are already inside this directory, skip this command.

### Step 1: Apply the invalid manifest

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Watch the Pod:

```powershell
kubectl get pod pod-oomkilled -w
```

Press `Ctrl+C` after observing the termination.

The invalid manifest configures:

- Memory request: `16Mi`
- Memory limit: `32Mi`
- Allocation attempt: `50000000` bytes, approximately 47.7 MiB

The command used is:

```sh
MEMORY=$(head -c 50000000 /dev/zero | tr '\000' 'A')
```

The command generates a large string and attempts to retain it in a shell variable. The shell and other processes also consume memory, so the container can exceed its memory limit before the assignment completes.

The exact timing can vary with the shell and container runtime.

### Step 2: Inspect the termination

```powershell
kubectl describe pod pod-oomkilled
```

Look for:

```text
State:       Terminated
Reason:      OOMKilled
Exit Code:   137
```

The Pod may appear as `OOMKilled` in `kubectl get pods`, while its overall phase is `Failed`.

### Step 3: Inspect the logs

```powershell
kubectl logs pod-oomkilled
```

Expected output:

```text
Allocating memory beyond container limit...
```

The `Allocation completed` message is absent because the process was killed before reaching that line.

### Step 4: Inspect Pod events

```powershell
kubectl get events --field-selector involvedObject.name=pod-oomkilled --sort-by=.metadata.creationTimestamp
```

Look for events indicating that the Pod was scheduled and the container was created and started.

The container termination reason is the primary evidence for this exercise.

### Step 5: Extract the termination reason and exit code

```powershell
kubectl get pod pod-oomkilled -o jsonpath="{.status.containerStatuses[0].state.terminated.reason}"
```

```powershell
kubectl get pod pod-oomkilled -o jsonpath="{.status.containerStatuses[0].state.terminated.exitCode}"
```

Expected values:

```text
OOMKilled
137
```

---

## 5. Root Cause

The invalid manifest configures a memory limit of `32Mi`.

```yaml
resources:
  requests:
    memory: "16Mi"
  limits:
    memory: "32Mi"
```

The command attempts to retain approximately 47.7 MiB of string data in a shell variable. The shell's additional memory consumption contributes to the total.

This workload can exceed the container's memory limit, causing the kernel to kill the process.

The YAML is syntactically valid. The intentional fault is the mismatch between the workload's memory demand and the configured memory limit.

---

## 6. Fix the Issue

The fixed manifest keeps the same Pod name and BusyBox image, but uses a smaller allocation and a higher memory limit.

### Step 1: Delete the failed Pod

```powershell
kubectl delete pod pod-oomkilled
```

### Step 2: Apply the fixed manifest

```powershell
kubectl apply -f .\fixed-pod.yaml
```

The fixed manifest configures:

- Memory request: `32Mi`
- Memory limit: `64Mi`
- Allocation: `10M` written to `/dev/shm/safe.bin`

`/dev/shm` is a memory-backed temporary filesystem in this setup, so the write uses memory rather than ordinary persistent disk storage.

The smaller allocation and higher memory limit are intended to allow the command to complete successfully.

---

## 7. Verify the Fix

Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-oomkilled --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-oomkilled -o wide
```

Expected result:

```text
NAME            READY   STATUS    RESTARTS   AGE
pod-oomkilled   1/1     Running   0          ...
```

Inspect the logs:

```powershell
kubectl logs pod-oomkilled
```

Expected output:

```text
Allocating memory safely within limit...
Allocation successful
```

The container then executes `sleep 3600`, keeping it running for one hour.

If the Pod does not become Ready, inspect its logs, events, and termination state. Do not assume the fix succeeded until you verify the output.

---

## 8. Screenshot Checklist

Save screenshots in the `screenshots/` directory.

- [ ] `01-oomkilled-status.png` — Pod status showing `OOMKilled`.
- [ ] `02-describe-pod-exitcode-137.png` — Termination reason and exit code 137.
- [ ] `03-logs-partial-execution.png` — Initial log message without `Allocation completed`.
- [ ] `04-verified-running-pod.png` — Fixed Pod showing `Running` with zero restarts.
- [ ] `05-fixed-pod-logs.png` — Successful allocation message.

**Important:** Capture the first three screenshots before deleting the failed Pod. Then apply the fixed manifest and capture the verification screenshots.

Both manifests use the same Pod name, so they cannot create two separate Pods with that name in the same namespace simultaneously.

---

## 9. Cleanup

Delete the Pod created by the fixed manifest:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-oomkilled
```

A `NotFound` response is expected after successful cleanup.

---

## 10. Key Learning

- **Root cause:** The invalid workload attempts to retain more memory than the container's `32Mi` limit permits.
- **Diagnostic command:** `kubectl describe pod pod-oomkilled`.
- **Confirmation:** Verify `Reason: OOMKilled`; exit code 137 is supporting evidence.
- **Correction:** Reduce memory consumption or configure a limit based on measured workload requirements.
- **Production consideration:** Use workload measurements and load testing to size requests and limits. Monitor memory usage and OOM events. `kubectl top pod` requires Metrics Server to be available.
