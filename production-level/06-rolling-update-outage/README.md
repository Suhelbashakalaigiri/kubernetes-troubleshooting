# Troubleshooting: Rolling Update Outage (Missing Readiness Probe Warmup)

## 1. Issue overview
A rolling update outage occurs when an updated Deployment terminates healthy old Pods before newly created replacement Pods have finished warming up and are actually capable of serving traffic.

In Kubernetes:
- If a container **omits a `readinessProbe`**, Kubernetes considers the container `Ready: True` **the instant its process starts**.
- If the application takes 8 seconds to warm up (e.g. database schema check, loading caches, JVM boot), the Deployment controller terminates the old, working Pods immediately upon the new Pod's launch.
- Client requests sent during this 8-second window hit an unready container, resulting in dropped requests or HTTP 503 Service Unavailable errors.

**Expected behavior:** The rolling update keeps old Pods active until new Pods pass readiness checks, ensuring continuous service availability.  
**Actual behavior:** Lacking a readiness probe, new Pods are marked ready immediately at second 0, old Pods are killed, and clients experience an 8-second service blackout.

---

## 2. Common causes
1. **Missing readiness probes:** Assuming a running process equals a ready service.
2. **Missing `minReadySeconds`:** Deployment updates proceed before new Pods demonstrate stability.
3. **Aggressive `maxUnavailable` with slow startup:** Terminating capacity before replacements initialize.
4. **Failure to handle SIGTERM gracefully:** Applications exit immediately without completing active in-flight requests.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\06-rolling-update-outage"
```

Apply the unhardened Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Wait 2 seconds and immediately check the Pod status:

```powershell
kubectl get pods -l app=warmup-demo
```

Notice that the Pod reports `READY: 1/1` within 2 seconds:
```text
NAME                                    READY   STATUS    RESTARTS   AGE
deploy-warmup-outage-68c467bc45-5k2ml   1/1     Running   0          3s
```

Now check the container logs:

```powershell
$pod = (kubectl get pods -l app=warmup-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod
```

Output:
```text
Bootstrapping server (warming up caches)...
```
*Notice: The container is still in its 8-second sleep, yet Kubernetes already marked it `READY: 1/1`! Any traffic routed to it now will fail.*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the Pod's readiness probe configuration
```powershell
kubectl get pod $pod -o jsonpath='{.spec.containers[0].readinessProbe}'
```

Output:
```text
# (empty)
```
*No readiness probe is defined.*

### Step 2: Diagnostic conclusion
- Without a readiness probe, Kubernetes relies purely on process exit status.
- Because `sh` is running, Kubernetes treats the container as 100% available, causing zero-downtime rolling updates to fail.

---

## 6. Root cause
`invalid-deployment.yaml` omits a `readinessProbe`. Kubernetes marks the container ready immediately upon launch, terminating old Pods while new Pods are still warming up.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` adds a `readinessProbe` and `minReadySeconds: 3`.

---

## 8. Verify the fix
Trigger a rollout update:

```powershell
kubectl rollout restart deployment/deploy-warmup-outage
```

Monitor the rollout:

```powershell
kubectl get pods -l app=warmup-demo
```

Observe that new Pods remain `READY: 0/1` during the 8-second warmup:
```text
NAME                                    READY   STATUS    RESTARTS   AGE
deploy-warmup-outage-5cf4b977bc-7j9xk   0/1     Running   0          4s
```

After 8 seconds, the probe succeeds, the Pod transitions to `1/1 Ready`, and old Pods are safely terminated:

```powershell
kubectl rollout status deployment/deploy-warmup-outage --timeout=60s
```

Expected output:
```text
deployment "deploy-warmup-outage" successfully rolled out
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-premature-ready-status.png` — Output showing `READY: 1/1` within 3 seconds while still warming up.
- [ ] `02-warmup-logs-incomplete.png` — Output of `kubectl logs` showing `warming up caches` while Pod was marked Ready.
- [ ] `03-fixed-readiness-probe-active.png` — Output showing new Pod held in `0/1 Ready` until warmup completes.
- [ ] `04-verified-rollout-success.png` — Output of `kubectl rollout status` showing clean zero-downtime transition.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-warmup-outage
```

---

## 11. Key learning
- **Root cause:** Lacking a readiness probe, the Deployment marked cold-starting Pods ready immediately, causing traffic blackouts during updates.
- **Most useful diagnostic command:** Comparing `kubectl logs` timestamps with `kubectl get pod` Ready transitions.
- **Correction:** Implement an HTTP or TCP `readinessProbe` and configure `minReadySeconds`.
- **Production consideration:** Always combine readiness probes with `preStop` hooks (e.g. `sleep 5`) to allow upstream load balancers and ingress controllers to drain traffic before terminating pods.
