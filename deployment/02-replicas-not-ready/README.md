# Troubleshooting: Replicas Not Ready

## 1. Issue overview
A "Replicas Not Ready" condition occurs when a Deployment successfully schedules and runs its desired number of Pods, but some or all of the replicas fail their readiness criteria and cannot transition to the `Ready` or `Available` state.

In Kubernetes:
- The Deployment status tracks four distinct replica counts:
  - **Desired:** Number of replicas requested in `spec.replicas`.
  - **Current:** Number of Pods created by the active ReplicaSet.
  - **Ready:** Number of Pods passing readiness checks (`status.conditions[Ready].status == True`).
  - **Available:** Number of Pods that have been ready for at least `minReadySeconds`.
- When Pods are running (`STATUS: Running`) but unready (`READY: 0/1`), they cannot receive traffic from Kubernetes Services, and the Deployment reports `READY: 0/3`.

**Expected behavior:** 3 Pods are created, pass readiness checks, and report `3/3 Ready` and `3 Available`.  
**Actual behavior:** All 3 Pods are running, but their readiness probe queries `/health-check` which returns 404, resulting in `0/3 Ready`.

---

## 2. Common causes
1. **Misconfigured readiness probes:** Probe queries a non-existent path, wrong port, or invalid protocol.
2. **Slow application warmup:** Application initialization exceeds the probe's initial delay or timeout parameters.
3. **Deadlocks or unhandled exceptions:** Application process is running, but internal worker threads cannot answer HTTP health checks.
4. **Missing runtime configuration:** Application starts up in a degraded state because a required database connection or API endpoint is unreachable.
5. **Node port / socket conflicts:** Application fails to bind to its listening socket inside the container.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\deployment\02-replicas-not-ready"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Check the Deployment status after 15 seconds:

```powershell
kubectl get deployment deploy-replicas-not-ready
```

Notice the mismatch between desired replicas and ready replicas:
```text
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deploy-replicas-not-ready   0/3     3            0           25s
```

Check the Pods:

```powershell
kubectl get pods -l app=replicas-not-ready-demo
```

Output:
```text
NAME                                         READY   STATUS    RESTARTS   AGE
deploy-replicas-not-ready-7b949bc5df-5d6kz   0/1     Running   0          30s
deploy-replicas-not-ready-7b949bc5df-9m8lx   0/1     Running   0          30s
deploy-replicas-not-ready-7b949bc5df-p42zq   0/1     Running   0          30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Compare Deployment replica metrics
```powershell
kubectl get deployment deploy-replicas-not-ready -o jsonpath='{"Desired: "}{.spec.replicas}{"
Updated: "}{.status.updatedReplicas}{"
Ready: "}{.status.readyReplicas}{"
Available: "}{.status.availableReplicas}{"
"}'
```

Output:
```text
Desired: 3
Updated: 3
Ready: 
Available: 
```
*Note that `Ready` and `Available` are empty/zero.*

### Step 2: Inspect a Pod's conditions and events
Pick one of the Pods and describe it:

```powershell
$pod = (kubectl get pods -l app=replicas-not-ready-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod
```

Examine the **Conditions** block:
```text
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
```

Examine the **Events** block:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  45s                default-scheduler  Successfully assigned default/... to minikube
  Normal   Pulled     44s                kubelet            Container image "nginx:1.25-alpine" already present on machine
  Normal   Created    44s                kubelet            Created container web
  Normal   Started    44s                kubelet            Started container web
  Warning  Unhealthy  4s (x8 over 40s)   kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
```

### Step 3: Test the web server endpoint directly
Execute a probe check from inside the container:

```powershell
kubectl exec $pod -- wget -q -S -O /dev/null http://localhost:80/health-check 2>&1 | Select-String "HTTP/"
kubectl exec $pod -- wget -q -S -O /dev/null http://localhost:80/ 2>&1 | Select-String "HTTP/"
```

Output:
```text
HTTP/1.1 404 Not Found
HTTP/1.1 200 OK
```

**Diagnostic analysis:**
- The container is healthy and actively serving requests on port 80.
- The readiness probe failed because it queries `/health-check`, which does not exist in standard Nginx.
- Because readiness fails, all 3 replicas remain `0/1 Ready` and `0 Available`.

---

## 6. Root cause
In `invalid-deployment.yaml`, lines 24-27 configure the readiness probe:
```yaml
readinessProbe:
  httpGet:
    path: /health-check
    port: 80
```
Nginx returns HTTP 404 for `/health-check`, preventing all 3 replicas from achieving `Ready` status.

---

## 7. Fix the issue
Apply the corrected Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` sets `path: /`, matching the default root endpoint served by Nginx.

---

## 8. Verify the fix
Wait for all replicas to become Available:

```powershell
kubectl rollout status deployment/deploy-replicas-not-ready --timeout=60s
```

Verify the Deployment metrics:

```powershell
kubectl get deployment deploy-replicas-not-ready
kubectl get pods -l app=replicas-not-ready-demo
```

Expected output:
```text
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deploy-replicas-not-ready   3/3     3            3           2m
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-unready-replicas-deployment.png` — Output of `kubectl get deployment` showing `READY: 0/3`.
- [ ] `02-describe-pod-readiness-404.png` — Output of `kubectl describe pod` showing `Readiness probe failed with statuscode: 404`.
- [ ] `03-exec-endpoint-check.png` — Output of `kubectl exec` testing `/health-check` (404) vs `/` (200).
- [ ] `04-verified-all-replicas-ready.png` — Output of `kubectl get deployment` showing `READY: 3/3` and `AVAILABLE: 3`.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-replicas-not-ready
```

---

## 11. Key learning
- **Root cause:** An invalid readiness probe path (`/health-check`) prevented all running replicas from being recognized as Ready.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read readiness probe HTTP response codes and failure events.
- **Correction:** Align the readiness probe path and port with a verified application endpoint.
- **Production consideration:** Always distinguish between liveness probes (which restart containers) and readiness probes (which control traffic ingress); ensure readiness endpoints return 200 only when local dependencies are healthy.
