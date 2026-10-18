# Troubleshooting: Failed Scheduling (Insufficient Node Resources)

## 1. Issue overview
`FailedScheduling` is a scheduler event indicating that the Kubernetes scheduler (`kube-scheduler`) cannot place a Pod onto any node in the cluster because no node satisfies the Pod's compute resource requests, affinity rules, or scheduling constraints.

In Kubernetes:
- The scheduler runs the `NodeResourcesFit` plugin during the filtering phase of the scheduling cycle.
- The scheduler calculates available allocatable CPU and memory on each node by subtracting existing resource requests of assigned Pods from the node's total allocatable capacity.
- If a Pod requests more CPU or memory than any single node has available, the Pod remains in the `Pending` phase and a `FailedScheduling` warning event is recorded.

**Expected behavior:** The scheduler finds a node with sufficient allocatable CPU and memory, assigns the Pod to that node, and Kubelet creates the container.  
**Actual behavior:** The Pod requests 500 CPU cores and 500 GiB of memory—vastly exceeding the capacity of any Minikube or standard physical node—leaving the Pod unscheduled in `Pending`.

---

## 2. Common causes
1. **Unrealistic resource requests:** Typo in CPU or memory requests (e.g., requesting `500` CPUs instead of `500m`, or `50Gi` instead of `50Mi`).
2. **Cluster capacity exhaustion:** Existing workloads have consumed all allocatable CPU or memory across all nodes.
3. **Resource fragmentation:** Sufficient cluster-wide resources exist in aggregate, but no single node has enough contiguous capacity to host the Pod.
4. **DaemonSet or system overhead:** System reserved capacity reduces the allocatable capacity below what user workloads request.
5. **Node taints or unschedulable state:** Nodes are cordoned (`SchedulingDisabled`) or tainted.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-failed-scheduling"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status:

```powershell
kubectl get pods pod-failed-scheduling -o wide
```

Observe that the Pod stays in `Pending` with no node assigned:
```text
NAME                    READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-failed-scheduling   0/1     Pending   0          20s   <none>   <none>   <none>           <none>
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and scheduler diagnostics
```powershell
kubectl describe pod pod-failed-scheduling
```

Check the **Events** section:
```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  18s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

### Step 2: Compare Pod requests with Node allocatable capacity
Inspect the allocatable resources and existing allocations of the Minikube node:

```powershell
kubectl describe node minikube | Select-String -Pattern "Allocatable:","Allocated resources:","  cpu ","  memory " -Context 0,5
```

Alternatively, view node allocatable values directly via jsonpath:

```powershell
kubectl get nodes -o jsonpath='{"Node: "}{.items[*].metadata.name}{"
Allocatable CPU: "}{.items[*].status.allocatable.cpu}{"
Allocatable Memory: "}{.items[*].status.allocatable.memory}{"
"}'
```

Expected output (typical Minikube VM):
```text
Node: minikube
Allocatable CPU: 2 (or 4)
Allocatable Memory: 3883716Ki (approximately 3.8 GiB)
```

Compare this with the Pod's resource requests:

```powershell
kubectl get pod pod-failed-scheduling -o jsonpath='{.spec.containers[0].resources.requests}'
```

Output:
```text
{"cpu":"500","memory":"500Gi"}
```

**Diagnostic analysis:**
- The node has 2–4 CPU cores and ~4 GiB of RAM allocatable.
- The Pod requests 500 CPU cores and 500 GiB of RAM.
- The `NodeResourcesFit` filter rejects the node with `Insufficient cpu, 1 Insufficient memory`.
- Because preemption cannot free 500 CPUs on a node that only physically has 2–4 cores, preemption is aborted and the Pod remains `Pending`.

---

## 6. Root cause
In `invalid-pod.yaml`, lines 13-16 request:
```yaml
resources:
  requests:
    cpu: "500"
    memory: "500Gi"
```
The requested CPU and memory exceed the maximum allocatable resources of the node by orders of magnitude.

---

## 7. Fix the issue
Delete the unschedulable Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` right-sizes the container requests to `50m` CPU (0.05 core) and `64Mi` memory, which easily fits onto any standard Minikube node.

---

## 8. Verify the fix
Wait for the Pod to be scheduled and reach `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-failed-scheduling --timeout=60s
```

Check the Pod status and assigned node:

```powershell
kubectl get pod pod-failed-scheduling -o wide
```

Expected output:
```text
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod-failed-scheduling   1/1     Running   0          20s   10.244.0.X   minikube   <none>           <none>
```

Verify that the node assignment is now set to `minikube` and the Pod is in `Running` status.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-failed-scheduling-status.png` — Output of `kubectl get pods` showing `STATUS: Pending`.
- [ ] `02-describe-pod-insufficient-resources.png` — Output of `kubectl describe pod` showing `0/1 nodes are available: 1 Insufficient cpu, 1 Insufficient memory`.
- [ ] `03-node-allocatable-comparison.png` — Output comparing node allocatable CPU/memory with Pod requested resources.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-failed-scheduling -o wide` showing `Running` on `minikube`.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-failed-scheduling
```

---

## 11. Key learning
- **Root cause:** The Pod requested 500 CPUs and 500Gi memory, vastly exceeding node capacity and causing the scheduler's `NodeResourcesFit` filter to reject all nodes.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the `FailedScheduling` message and `kubectl describe node <node-name>` to check allocatable capacity.
- **Correction:** Adjust container resource requests (`requests.cpu` and `requests.memory`) to reflect actual application consumption.
- **Production consideration:** Use `LimitRange` objects in namespaces to enforce default, minimum, and maximum resource request limits, preventing developers from unintentionally submitting excessive resource requests that starve clusters.
