# Troubleshooting: Pending Pod (Scheduling Constraint Failure)

## 1. Issue overview
The `Pending` phase indicates that a Pod has been accepted by the Kubernetes API server, but one or more of its containers have not been scheduled onto a node or cannot begin execution.

In Kubernetes:
- Pod scheduling is handled by `kube-scheduler`.
- The scheduler runs a two-step cycle: **Filtering** (identifying nodes that satisfy all Pod requirements) and **Scoring** (ranking feasible nodes).
- If no node passes the filtering phase (e.g., due to unsatisfiable `nodeSelector`, taints, or affinity rules), the Pod remains in `Pending` phase indefinitely until a matching node becomes available.

**Expected behavior:** The scheduler matches the Pod to an available node, binds the Pod to that node, and Kubelet initiates container creation.  
**Actual behavior:** The Pod specifies a `nodeSelector` requirement (`disktype=ssd-high-performance-nonexistent`) that matches zero nodes in the cluster, leaving the Pod stuck in `Pending`.

---

## 2. Common causes
1. **Unsatisfiable node selector or node affinity:** Demanding labels (`nodeSelector` or `nodeAffinity`) that no cluster node possesses.
2. **Insufficient cluster resources:** Node CPU or memory capacity is exhausted.
3. **Un-tolerated node taints:** Nodes are tainted (e.g., `node.kubernetes.io/unreachable` or dedicated workloads) and the Pod lacks matching tolerations.
4. **PersistentVolumeClaim unfulfilled:** Pod references a PVC that is not yet bound to a PV.
5. **Node limits reached:** Maximum Pod limit per node (`maxPods`) has been reached.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-pending-pod"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the status of the Pod:

```powershell
kubectl get pods pod-pending-scheduling -o wide
```

Observe that the Pod stays in `Pending` with no node assigned (`NODE: <none>`):
```text
NAME                      READY   STATUS    RESTARTS   AGE   IP       NODE     NOMINATED NODE   READINESS GATES
pod-pending-scheduling    0/1     Pending   0          30s   <none>   <none>   <none>           <none>
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and scheduler messages
```powershell
kubectl describe pod pod-pending-scheduling
```

Examine the **Node-Selectors** and **Events** sections:
```text
Node-Selectors:              disktype=ssd-high-performance-nonexistent
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/1 nodes are available: 1 node(s) didn't match Pod's node selector. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

### Step 2: Check available nodes and their labels
List all nodes in the cluster:

```powershell
kubectl get nodes
```

Inspect the labels assigned to your Minikube node:

```powershell
kubectl get nodes --show-labels
```

Search specifically for the requested label `disktype`:

```powershell
kubectl get nodes -l disktype=ssd-high-performance-nonexistent
```

Expected output:
```text
No resources found
```

### Step 3: Verify scheduling status via jsonpath
```powershell
kubectl get pod pod-pending-scheduling -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}'
```

Output:
```text
0/1 nodes are available: 1 node(s) didn't match Pod's node selector...
```

**Diagnostic analysis:**
- The scheduler explicitly states that `0/1 nodes are available` because `1 node(s) didn't match Pod's node selector`.
- Querying node labels confirmed that no node possesses `disktype=ssd-high-performance-nonexistent`.
- Because this constraint cannot be satisfied, the Pod cannot proceed past the `Pending` phase.

---

## 6. Root cause
`invalid-pod.yaml` defines a mandatory `nodeSelector`:
```yaml
nodeSelector:
  disktype: ssd-high-performance-nonexistent
```
Minikube nodes do not have this label, causing the default scheduler's filtering stage to reject all available nodes.

---

## 7. Fix the issue
Because the `nodeSelector` of an existing Pod is an immutable field, delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` removes the unsatisfiable `nodeSelector` requirement, allowing the scheduler to place the Pod onto any healthy node.

---

## 8. Verify the fix
Wait for the Pod to be scheduled and reach `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-pending-scheduling --timeout=60s
```

Check the Pod status and assigned node:

```powershell
kubectl get pod pod-pending-scheduling -o wide
```

Expected output:
```text
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod-pending-scheduling    1/1     Running   0          25s   10.244.0.X   minikube   <none>           <none>
```

Verify that the node assignment is now populated (`minikube`) and the Pod phase is `Running`.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-pending-pod-status.png` — Output of `kubectl get pods` showing `STATUS: Pending` and `NODE: <none>`.
- [ ] `02-failed-scheduling-event.png` — Output of `kubectl describe pod pod-pending-scheduling` highlighting the `FailedScheduling` warning event.
- [ ] `03-node-labels-check.png` — Output of `kubectl get nodes -l disktype=...` showing `No resources found`.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-pending-scheduling -o wide` showing `Running` on `minikube`.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-pending-scheduling
```

---

## 11. Key learning
- **Root cause:** The Pod specified a mandatory `nodeSelector` (`disktype: ssd-high-performance-nonexistent`) that did not match any node in the cluster.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the `FailedScheduling` event message.
- **Correction:** Remove invalid node selectors or apply matching labels to target nodes using `kubectl label nodes`.
- **Production consideration:** Prefer soft node affinity (`preferredDuringSchedulingIgnoredDuringExecution`) when possible, so Pods can still schedule if preferred hardware or topology labels are absent.
