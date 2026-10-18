# Troubleshooting: Node Not Ready (Node Eviction and Tolerations)

## 1. Issue overview
A `Node Not Ready` condition occurs when the cluster control plane stops receiving heartbeats from a node's Kubelet. When a node transitions to `Ready: False` or `Ready: Unknown`, Kubernetes automatically applies node taints (`node.kubernetes.io/not-ready:NoExecute` and `node.kubernetes.io/unreachable:NoExecute`).

In Kubernetes:
- Kubelet reports node health through node leases in the `kube-node-lease` namespace.
- If heartbeats cease (e.g. node kernel panic, disk saturation, network partition), the `node-lifecycle-controller` taints the node.
- Pods without explicit tolerations for these taints are **evicted** after the default `tolerationSeconds: 300` (5 minutes).
- For critical system daemons, operators must configure explicit `tolerations` with extended `tolerationSeconds` so temporary node reboots or network glitches do not trigger immediate destructive evictions.

**Expected behavior:** Critical Pods tolerate transient node unreachability, avoiding premature pod eviction.  
**Actual behavior:** Standard Pods lack custom tolerations, making them vulnerable to immediate eviction during node maintenance.

---

## 2. Common causes
1. **Kubelet service crashed or killed on node:** Systemd stops `kubelet.service` due to OOM or configuration errors.
2. **Node disk pressure:** Root partition fills to 100%, causing Kubelet to enter `DiskPressure` and refuse heartbeats.
3. **Container runtime hung:** `containerd` socket hangs, preventing Kubelet from reporting container status.
4. **Network partition between node and control plane:** Routing loss or firewall drop prevents reaching kube-apiserver port 6443.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\02-node-not-ready"
```

Apply the unhardened Pod manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod/pod-critical-workload --timeout=30s
```

Inspect the default tolerations automatically assigned by Kubernetes:

```powershell
kubectl get pod pod-critical-workload -o jsonpath='{.spec.tolerations}'
```

Output:
```text
[{"effect":"NoExecute","key":"node.kubernetes.io/not-ready","operator":"Exists","tolerationSeconds":300},{"effect":"NoExecute","key":"node.kubernetes.io/unreachable","operator":"Exists","tolerationSeconds":300}]
```
*Notice: Default `tolerationSeconds` is 300 (5 minutes). If the node is unreachable, the Pod is terminated at 300 seconds.*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect node conditions
```powershell
kubectl describe node minikube | Select-String "Conditions:","Ready" -Context 0,6
```

### Step 2: Check node lease heartbeats
```powershell
kubectl get leases -n kube-node-lease
```

Expected output:
```text
NAME       HOLDER     AGE
minikube   minikube   5d
```

### Step 3: Diagnostic analysis
- When a node goes down, the control plane adds the `node.kubernetes.io/not-ready:NoExecute` taint.
- The `node-lifecycle-controller` monitors `tolerationSeconds`.
- For critical workloads, setting a custom toleration with higher `tolerationSeconds` prevents thrashing during planned node maintenance or kernel updates.

---

## 6. Root cause
`invalid-pod.yaml` relies on default 300-second eviction behavior without explicit toleration controls for maintenance windows.

---

## 7. Fix the issue
Delete the test Pod and apply the hardened manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` configures explicit `node.kubernetes.io/not-ready` and `unreachable` tolerations with `tolerationSeconds: 600`.

---

## 8. Verify the fix
Verify the updated toleration configuration:

```powershell
kubectl get pod pod-critical-workload -o jsonpath='{.spec.tolerations[?(@.key=="node.kubernetes.io/not-ready")]}'
```

Expected output:
```text
{"effect":"NoExecute","key":"node.kubernetes.io/not-ready","operator":"Exists","tolerationSeconds":600}
```

Verify Pod is running:

```powershell
kubectl get pod pod-critical-workload -o wide
```

Expected output:
```text
NAME                    READY   STATUS    RESTARTS   AGE
pod-critical-workload   1/1     Running   0          20s
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-default-tolerations.png` — Output showing default `tolerationSeconds: 300`.
- [ ] `02-node-conditions-inspection.png` — Output of `kubectl describe node` showing Ready status.
- [ ] `03-kube-node-lease-check.png` — Output of `kubectl get leases -n kube-node-lease`.
- [ ] `04-verified-custom-tolerations.png` — Output showing updated `tolerationSeconds: 600`.

---

## 10. Cleanup
Delete the Pod:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-critical-workload
```

---

## 11. Key learning
- **Root cause:** Default node evictions occur after 300 seconds of node unreachability.
- **Most useful diagnostic command:** `kubectl describe node <name>` and `kubectl get leases -n kube-node-lease` to assess node heartbeat vitality.
- **Correction:** Add explicit `tolerations` for `node.kubernetes.io/not-ready` with tailored `tolerationSeconds`.
- **Production consideration:** Use `StatefulSets` with caution during node not-ready events: Kubernetes will not automatically delete and recreate volume-attached pods on another node to prevent split-brain data corruption.
