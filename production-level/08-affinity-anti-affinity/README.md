# Troubleshooting: Affinity and Anti-Affinity Scheduling Blockages

## 1. Issue overview
Pod anti-affinity allows operators to prevent Pods of the same service from running on the same failure domain (e.g. same node, rack, or availability zone). However, configuring **hard anti-affinity** on a cluster with fewer failure domains than desired replicas creates an impossible scheduling constraint.

In Kubernetes:
- `requiredDuringSchedulingIgnoredDuringExecution` (Hard rule): The scheduler **must** satisfy the rule, or the Pod will not be scheduled.
- `preferredDuringSchedulingIgnoredDuringExecution` (Soft rule): The scheduler attempts to spread Pods across nodes, but will co-locate Pods on the same node if no alternatives exist.
- When a 2-replica Deployment specifies hard anti-affinity with `topologyKey: kubernetes.io/hostname` on a 1-node Minikube cluster:
  - The first Pod schedules onto `minikube`.
  - The second Pod cannot schedule onto `minikube` because an identical Pod already runs there.
  - Because no other nodes exist, the second replica remains in `Pending` forever.

**Expected behavior:** Both replicas schedule and run successfully.  
**Actual behavior:** 1 replica runs; the 2nd replica stays `Pending` with `0/1 nodes are available: 1 node(s) had volume node-affinity or was anti-affinity violated`.

---

## 2. Common causes
1. **Hard anti-affinity exceeding node count:** Requesting 5 replicas with hard hostname anti-affinity on a 3-node cluster.
2. **AZ anti-affinity during cloud outage:** Hard AZ anti-affinity when an availability zone is degraded or out of capacity.
3. **Missing or mismatched `topologyKey`:** Specifying a topology label that nodes do not possess.
4. **Circular pod affinity dependencies:** Pod A requires Pod B on the same node, while Pod B requires Pod A, causing a deadlock.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\08-affinity-anti-affinity"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pods after 15 seconds:

```powershell
kubectl get pods -l app=anti-affinity-demo
```

Observe that 1 Pod runs, while the 2nd stays `Pending`:
```text
NAME                                    READY   STATUS    RESTARTS   AGE
deploy-anti-affinity-5f849bc5df-5k2ml   1/1     Running   0          25s
deploy-anti-affinity-5f849bc5df-9m8lx   0/1     Pending   0          25s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the Pending Pod's events
```powershell
$pendingPod = (kubectl get pods -l app=anti-affinity-demo --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pendingPod
```

Examine the **Events** section:
```text
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  18s   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint, 1 node(s) had volume node-affinity or was anti-affinity violated.
```

### Step 2: Diagnostic analysis
- The rule is `requiredDuringSchedulingIgnoredDuringExecution`.
- The topology key is `kubernetes.io/hostname`.
- Since Minikube has only 1 node, the hard anti-affinity rule cannot be satisfied for more than 1 replica.

---

## 6. Root cause
In `invalid-pod.yaml`, line 18 enforces hard anti-affinity (`requiredDuringSchedulingIgnoredDuringExecution`) on `topologyKey: "kubernetes.io/hostname"`. The second replica cannot schedule because the single node already hosts a matching Pod.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` switches to soft anti-affinity:
```yaml
preferredDuringSchedulingIgnoredDuringExecution:
- weight: 100
  podAffinityTerm:
    labelSelector:
      matchExpressions:
      - key: app
        operator: In
        values:
        - anti-affinity-demo
    topologyKey: "kubernetes.io/hostname"
```

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-anti-affinity --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=anti-affinity-demo -o wide
```

Expected output:
```text
NAME                                    READY   STATUS    RESTARTS   AGE   NODE
deploy-anti-affinity-7b949bc5df-2k9mp   1/1     Running   0          20s   minikube
deploy-anti-affinity-7b949bc5df-8j5kz   1/1     Running   0          20s   minikube
```
Both replicas are now running cleanly.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-anti-affinity-pending-status.png` — Output showing 1 Running and 1 Pending replica.
- [ ] `02-anti-affinity-violated-event.png` — Output of `kubectl describe pod` showing `anti-affinity violated`.
- [ ] `03-soft-affinity-manifest.png` — Viewing `fixed-pod.yaml` showing `preferredDuringScheduling`.
- [ ] `04-verified-both-replicas-running.png` — Output of `kubectl get pods` showing `2/2 Running`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-anti-affinity
```

---

## 11. Key learning
- **Root cause:** Hard pod anti-affinity on a cluster with fewer failure domains than replicas permanently locked excess replicas in `Pending`.
- **Most useful diagnostic command:** `kubectl describe pod <pending-pod>` to inspect the `anti-affinity violated` event.
- **Correction:** Use soft anti-affinity (`preferredDuringSchedulingIgnoredDuringExecution`) to spread pods when nodes are available without blocking scheduling when nodes are scarce.
- **Production consideration:** Reserve hard anti-affinity for specialized workloads (e.g. Cassandra or ZooKeeper nodes) where co-location violates Byzantine fault tolerance.
