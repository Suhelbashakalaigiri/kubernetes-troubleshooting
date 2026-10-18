# Troubleshooting: Node Taints and Pod Tolerations

## 1. Issue overview
Taints and tolerations work together to ensure that Pods are not scheduled onto inappropriate nodes. A node taint acts as a repellent: unless a Pod explicitly declares a matching toleration, the scheduler will not place the Pod onto that node.

In Kubernetes:
- A taint consists of `key=value:effect` (e.g. `dedicated=special-workload:NoSchedule`).
- `NoSchedule`: The Kubernetes scheduler will not schedule any Pod onto the node unless it has a matching toleration.
- If a cluster has nodes dedicated to GPU workloads or specific tenants, regular Pods lacking matching tolerations remain in `Pending` with warning event `0/1 nodes are available: 1 node(s) had untolerated taint`.

**Expected behavior:** The Pod declares a matching toleration, allowing the scheduler to place it on the dedicated node.  
**Actual behavior:** The node has a `dedicated=special-workload:NoSchedule` taint, and the Pod has no toleration, leaving it stuck in `Pending`.

---

## 2. Common causes
1. **Missing toleration in Pod spec:** Pod requires dedicated hardware (e.g. GPU), but manifest omits the toleration.
2. **Typo in taint key, value, or effect:** Toleration specifies `NoExecute` when the taint is `NoSchedule`.
3. **Control plane node isolation:** Master/control plane nodes carry `node-role.kubernetes.io/control-plane:NoSchedule` by default.
4. **Node condition taints:** Automatic system taints (`node.kubernetes.io/disk-pressure`, `node.kubernetes.io/memory-pressure`).

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\07-taints-tolerations"
```

### Step 1: Apply a test taint to the Minikube node
Apply a temporary taint to simulate a dedicated workload node:

```powershell
kubectl taint nodes minikube dedicated=special-workload:NoSchedule
```

Verify that the taint is applied:

```powershell
kubectl describe node minikube | Select-String "Taints:" -Context 0,2
```

### Step 2: Apply the untolerating Pod manifest
```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status:

```powershell
kubectl get pod pod-taint-blocked
```

Observe that the Pod stays in `Pending`:
```text
NAME                READY   STATUS    RESTARTS   AGE
pod-taint-blocked   0/1     Pending   0          20s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect scheduler events
```powershell
kubectl describe pod pod-taint-blocked
```

Examine the **Events** section:
```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  15s   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {dedicated: special-workload}.
```

### Step 2: Inspect node taints
```powershell
kubectl get node minikube -o jsonpath='{.spec.taints}'
```

Output:
```text
[{"effect":"NoSchedule","key":"dedicated","value":"special-workload"}]
```

**Diagnostic analysis:**
- The node carries taint `dedicated=special-workload:NoSchedule`.
- The Pod spec does not define any tolerations for `dedicated`.
- The scheduler's filter plugin rejects the node, keeping the Pod in `Pending`.

---

## 6. Root cause
`invalid-pod.yaml` lacks a `tolerations` section matching the node's `dedicated=special-workload:NoSchedule` taint.

---

## 7. Fix the issue
Delete the pending Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` declares the matching toleration:
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "special-workload"
  effect: "NoSchedule"
```

---

## 8. Verify the fix
Wait for the Pod to schedule and become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-taint-blocked --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-taint-blocked -o wide
```

Expected output:
```text
NAME                READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
pod-taint-blocked   1/1     Running   0          15s   10.244.0.X   minikube   <none>           <none>
```
The Pod successfully schedules and runs on the tainted node.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-node-taint-applied.png` — Output showing taint applied to minikube.
- [ ] `02-untolerated-taint-event.png` — Output of `kubectl describe pod` showing `1 node(s) had untolerated taint`.
- [ ] `03-fixed-toleration-manifest.png` — Viewing `fixed-pod.yaml` showing the toleration block.
- [ ] `04-verified-running-on-tainted-node.png` — Output showing `pod-taint-blocked` in `Running` status on `minikube`.

---

## 10. Cleanup
Delete the Pod and remove the test taint from the node:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
kubectl taint nodes minikube dedicated:NoSchedule- --ignore-not-found=true
```

Verify that the taint was removed:

```powershell
kubectl describe node minikube | Select-String "Taints:" -Context 0,2
```

---

## 11. Key learning
- **Root cause:** The node carried a `NoSchedule` taint that the Pod lacked tolerations to bypass.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the `untolerated taint` message.
- **Correction:** Add a matching toleration to `spec.tolerations` or remove the taint from the node using `kubectl taint nodes <node> <key>-`.
- **Production consideration:** Use taints and tolerations in combination with node affinity to ensure dedicated nodes only run targeted workloads (taints repel other pods, while affinity attracts the intended pods).
