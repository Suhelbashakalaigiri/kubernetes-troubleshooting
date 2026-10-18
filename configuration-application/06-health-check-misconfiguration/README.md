# Troubleshooting: Health-Check Misconfiguration

## 1. Issue overview
A health check misconfiguration occurs when a Pod's `livenessProbe` or `readinessProbe` is configured with incorrect endpoints, overly aggressive thresholds, or coupled to external dependencies, causing healthy applications to be repeatedly killed or denied traffic.

In Kubernetes:
- **Liveness Probes:** Answer *"Is the container process deadlocked or hung?"* If failed, Kubelet **kills and restarts the container**.
- **Readiness Probes:** Answer *"Is the container ready to receive client traffic?"* If failed, the Endpoint controller **removes the Pod from Service routing**, but **does NOT restart the container**.
- **Anti-Pattern (Deep Liveness Checking):** Pointing a liveness probe to a deep check (e.g. querying an external database or non-existent path `/deep-database-check`) is a catastrophic design flaw. If the database experiences transient latency or the path returns 404, Kubelet restarts every application container simultaneously, triggering cascading outages.

**Expected behavior:** The liveness probe validates local process vitality, keeping the container running.  
**Actual behavior:** The liveness probe queries `/deep-database-check` (404), causing Kubelet to kill and restart the container continuously.

---

## 2. Common causes
1. **Deep dependency checking in liveness probes:** Liveness probe tests external databases, message brokers, or third-party APIs.
2. **Aggressive timeoutSeconds:** Setting `timeoutSeconds: 1` on an application whose health check occasionally takes 1.2s under load.
3. **Low initialDelaySeconds:** Liveness probe fires before a heavy runtime (like the JVM) finishes warming up.
4. **Targeting wrong HTTP status codes:** Probing an endpoint that returns HTTP 401 Unauthorized or 403 Forbidden instead of 200.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\06-health-check-misconfiguration"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod over 20 to 30 seconds:

```powershell
kubectl get pods -l app=health-check-demo
```

Observe repeated restarts:
```text
NAME                                         READY   STATUS    RESTARTS      AGE
deploy-health-check-error-68c467bc45-5k2ml   1/1     Running   2 (10s ago)   25s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events for liveness failures
```powershell
$pod = (kubectl get pods -l app=health-check-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod
```

Examine the **Events** section:
```text
Events:
  Type     Reason     Age                From     Message
  ----     ------     ----               ----     -------
  Normal   Started    28s                kubelet  Started container web
  Warning  Unhealthy  10s (x4 over 24s)  kubelet  Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    10s (x2 over 20s)  kubelet  Container web failed liveness probe, will be restarted
```

### Step 2: Analyze the probe design flaw
- The warning states: `Liveness probe failed: HTTP probe failed with statuscode: 404` for `/deep-database-check`.
- Because this was configured as a **Liveness Probe**, Kubelet killed the container.
- If this application had been in production with 20 replicas and the database suffered a 5-second network hiccup, all 20 replicas would be killed and enter `CrashLoopBackOff` simultaneously.

---

## 6. Root cause
In `invalid-deployment.yaml`, lines 23-30 configure:
```yaml
livenessProbe:
  httpGet:
    path: /deep-database-check
    port: 80
  initialDelaySeconds: 2
  periodSeconds: 2
  failureThreshold: 2
```
Nginx returns HTTP 404 for `/deep-database-check`, prompting Kubelet to continuously terminate the container.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` follows production best practices:
1. Replaces the deep check with a lightweight local endpoint (`path: /`).
2. Provides realistic, stable timing parameters (`initialDelaySeconds: 5`, `periodSeconds: 5`, `failureThreshold: 3`, `timeoutSeconds: 2`).

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-health-check-error --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=health-check-demo
```

Expected output:
```text
NAME                                         READY   STATUS    RESTARTS   AGE
deploy-health-check-error-5cf4b977bc-k8m2p   1/1     Running   0          30s
```

Verify that events no longer report probe failures:

```powershell
$newPod = (kubectl get pods -l app=health-check-demo -o jsonpath='{.items[0].metadata.name}')
kubectl get events --field-selector involvedObject.name=$newPod
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-restarting-pod-status.png` — Output of `kubectl get pods` showing incrementing restarts.
- [ ] `02-liveness-404-killing-event.png` — Output of `kubectl describe pod` showing `Liveness probe failed with statuscode: 404` and `Container ... will be restarted`.
- [ ] `03-fixed-probe-manifest.png` — Viewing `fixed-deployment.yaml` showing local `/` check.
- [ ] `04-verified-stable-pod.png` — Output showing `STATUS: Running` with 0 restarts.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-health-check-error
```

---

## 11. Key learning
- **Root cause:** Pointing a liveness probe to an unmapped path (`/deep-database-check`) caused Kubelet to restart healthy containers.
- **Most useful diagnostic command:** `kubectl describe pod <name>` to check probe HTTP status codes and `Killing` events.
- **Correction:** Align liveness checks with local endpoints (e.g. `path: /`) and stable timeouts.
- **Production consideration:** **Never check downstream databases or caches in a liveness probe.** Reserve dependency checks for readiness probes so traffic is temporarily rerouted without restarting containers.
