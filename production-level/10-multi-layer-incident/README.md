# Troubleshooting: Multi-Layer Incident (Layered Root Cause Analysis)

## 1. Issue overview
In real-world production outages, incidents are rarely caused by a single isolated flaw. Complex incidents frequently involve multiple compounding failures across different abstraction layers. Attempting to fix one layer without systematic diagnosis leads to frustration when the service remains offline.

This exercise reproduces a **Triple-Layer Production Incident**:
1. **Layer 1 (Application/Pod Health):** The container is running, but fails its readiness probe (`/health-check` returns 404), keeping the Pod unready (`READY: 0/1`).
2. **Layer 2 (Service Discovery):** The Service selector is mismatched (`app: incident-wrong-selector` vs `app: incident-backend`), leaving `Endpoints: <none>`.
3. **Layer 3 (Network Routing):** The Service defines `targetPort: 8080` instead of port 80, ensuring that even if endpoints were found, traffic would fail with `Connection refused`.

**Expected behavior:** All layers function in harmony: Pods pass readiness checks, Service selects healthy Pods, and network ports proxy traffic correctly.  
**Actual behavior:** Client requests fail completely due to compounding flaws across Pod health, Service selectors, and network port mapping.

---

## 2. Common causes
1. **Multiple engineers making partial hotfixes during an incident:** Modifying Services while others modify Pod templates without coordination.
2. **Configuration drift between staging and production:** Service definitions from one environment combined with Deployment templates from another.
3. **Cascading failures:** A readiness probe failure triggers endpoint removal, while an emergency port change causes connection refused.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\10-multi-layer-incident"
```

Apply the invalid manifests:

```powershell
kubectl apply -f .\invalid-deployment.yaml
kubectl apply -f .\invalid-service.yaml
```

Test connecting through the Service:

```powershell
$svcIP = (kubectl get service svc-multi-layer -o jsonpath='{.spec.clusterIP}')
kubectl run test-client --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -T 3 -q -O - "http://$svcIP"
```

Observe that the connection fails immediately.

---

## 5. Identify and diagnose the issue: Layer-by-Layer Runbook

Follow this systematic 3-layer diagnostic procedure:

### Layer 1: Diagnose Pod Health and Readiness
Inspect the Deployment and Pods:

```powershell
kubectl get deployment deploy-multi-layer
kubectl get pods -l app=incident-backend
```

Output:
```text
NAME                                  READY   STATUS    RESTARTS   AGE
deploy-multi-layer-7b949bc5df-5d6kz   0/1     Running   0          30s
deploy-multi-layer-7b949bc5df-9m8lx   0/1     Running   0          30s
```
*Layer 1 Failure Detected: Pods are `Running`, but `READY: 0/1`.*

Describe a Pod to find the cause:
```powershell
$pod = (kubectl get pods -l app=incident-backend -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod | Select-String "Readiness probe failed"
```
Output:
```text
Warning  Unhealthy  Readiness probe failed: HTTP probe failed with statuscode: 404
```
**Layer 1 Root Cause:** Readiness probe targets `/health-check`, which returns 404.

---

### Layer 2: Diagnose Service Discovery and Selectors
Inspect the Service endpoints:

```powershell
kubectl get endpoints svc-multi-layer
```

Output:
```text
NAME              ENDPOINTS   AGE
svc-multi-layer   <none>      1m
```
*Layer 2 Failure Detected: Endpoints list is `<none>`.*

Compare Service selector with Pod labels:
```powershell
kubectl get service svc-multi-layer -o jsonpath='{"Service Selector: "}{.spec.selector}{"
"}'
kubectl get pods -l troubleshooting=production --show-labels
```
Output:
```text
Service Selector: {"app":"incident-wrong-selector"}
Pod Labels: app=incident-backend
```
**Layer 2 Root Cause:** Service selector `incident-wrong-selector` does not match Pod label `incident-backend`.

---

### Layer 3: Diagnose Network Port Mapping
Inspect Service port mappings:

```powershell
kubectl describe service svc-multi-layer | Select-String "Port:","TargetPort:"
```

Output:
```text
Port:              http  80/TCP
TargetPort:        8080/TCP
```
Inspect container listening ports:
```powershell
kubectl exec $pod -- netstat -tuln
```
Output:
```text
tcp        0      0 0.0.0.0:80              LISTEN
```
**Layer 3 Root Cause:** Service proxies to `targetPort: 8080`, while Nginx listens on port `80`.

---

## 6. Root cause summary
| Layer | Misconfiguration | Manifest Location |
| :--- | :--- | :--- |
| **Layer 1** | Readiness probe queries non-existent `/health-check` (404) | `invalid-deployment.yaml` |
| **Layer 2** | Service selector specifies `app: incident-wrong-selector` | `invalid-service.yaml` |
| **Layer 3** | Service `targetPort` routes to port 8080 instead of 80 | `invalid-service.yaml` |

---

## 7. Fix the issue
Apply the comprehensive fixed manifests:

```powershell
kubectl apply -f .\fixed-deployment.yaml
kubectl apply -f .\fixed-service.yaml
```

The fixes resolve all three layers:
1. `fixed-deployment.yaml` corrects the readiness probe path to `/`.
2. `fixed-service.yaml` corrects the selector to `app: incident-backend`.
3. `fixed-service.yaml` corrects `targetPort: 80`.

---

## 8. Verify the fix

### Verify Layer 1 (Pod Readiness)
```powershell
kubectl rollout status deployment/deploy-multi-layer --timeout=60s
kubectl get deployment deploy-multi-layer
```
Expected output: `READY: 2/2`.

### Verify Layer 2 (Endpoints)
```powershell
kubectl get endpoints svc-multi-layer
```
Expected output: 2 Pod IP addresses populated.

### Verify Layer 3 (End-to-End Connectivity)
```powershell
$svcIP = (kubectl get service svc-multi-layer -o jsonpath='{.spec.clusterIP}')
kubectl run verify-client --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```
All layers are verified healthy and traffic is served seamlessly.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-layer1-unready-pods.png` — Output showing `READY: 0/1` and probe 404 warnings.
- [ ] `02-layer2-empty-endpoints.png` — Output of `kubectl get endpoints` showing `<none>`.
- [ ] `03-layer3-port-mismatch.png` — Output comparing `TargetPort: 8080` with container port 80.
- [ ] `04-verified-incident-resolved.png` — Output showing `2/2 Ready`, populated endpoints, and `Welcome to nginx` response.

---

## 10. Cleanup
Delete all test resources:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
kubectl delete -f .\fixed-service.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-multi-layer
kubectl get service svc-multi-layer
```

---

## 11. Key learning
- **Root cause:** Compounding misconfigurations across Pod readiness, Service selectors, and targetPort routing caused a multi-layer outage.
- **Most useful diagnostic sequence:** Follow bottom-up layer verification: Pod container status → Pod readiness → Service selector & Endpoints → TargetPort & listening socket.
- **Correction:** Simultaneously correct all three layers to restore service health.
- **Production consideration:** In high-severity incidents, avoid hasty ad-hoc mutations; execute methodical layer-by-layer checks to avoid fixing one symptom while introducing another.
