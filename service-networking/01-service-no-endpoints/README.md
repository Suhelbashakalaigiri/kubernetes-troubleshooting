# Troubleshooting: Service Has No Endpoints

## 1. Issue overview
A "Service Has No Endpoints" issue occurs when a Kubernetes Service is created with a label selector that does not match any healthy, running Pods in the cluster, leaving the Service's endpoint list completely empty.

In Kubernetes:
- Services use label selectors (`spec.selector`) to identify target Pods.
- The Kubernetes Endpoints controller and EndpointSlice controller continuously query the API server for Pods matching the selector.
- When matching Pods are found and their `Ready` condition is `True`, their IPs are populated into the Service's `Endpoints` and `EndpointSlice` objects.
- If the selector is misspelled, mismatched, or target Pods are unready, the Service has zero endpoints (`<none>`), causing all requests sent to the Service IP or DNS name to be dropped or rejected with connection errors.

**Expected behavior:** The Service discovers the 2 backend Pods with label `app=backend-api`, registers their IPs in `Endpoints`, and load-balances client traffic.  
**Actual behavior:** The Service specifies `selector: app: wrong-backend-selector`, resulting in `Endpoints: <none>`.

---

## 2. Common causes
1. **Typo in Service label selector:** Key or value misspelled in `spec.selector` (e.g. `backend-api` vs `wrong-backend-selector`).
2. **Missing labels on Pod template:** Deployment Pod template (`spec.template.metadata.labels`) lacks the required labels.
3. **Pods are running but not Ready:** Pods match the selector, but fail readiness probes; unready Pods are excluded from endpoints.
4. **Namespace mismatch:** Service and target Pods reside in different namespaces (Services only select Pods in their own namespace).
5. **Headless service misconfiguration:** Service defined without selectors or with incorrect port mappings.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\01-service-no-endpoints"
```

Apply the invalid manifest containing the Deployment and misconfigured Service:

```powershell
kubectl apply -f .\invalid-service.yaml
```

Wait 10 seconds for the Deployment to start, then inspect the Service:

```powershell
kubectl get service svc-no-endpoints
```

Now check the Service's endpoints:

```powershell
kubectl get endpoints svc-no-endpoints
```

Observe that `ENDPOINTS` is empty (`<none>`):
```text
NAME               ENDPOINTS   AGE
svc-no-endpoints   <none>      30s
```

Check the EndpointSlice:

```powershell
kubectl get endpointslices -l kubernetes.io/service-name=svc-no-endpoints
```

Output:
```text
NAME                     ADDRESSTYPE   PORTS   ENDPOINTS   AGE
svc-no-endpoints-4xk9p   IPv4          80                  35s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Compare Service selector with Pod labels
Inspect the Service selector:

```powershell
kubectl get service svc-no-endpoints -o jsonpath='{"Service Selector: "}{.spec.selector}{"
"}'
```

Output:
```text
Service Selector: {"app":"wrong-backend-selector"}
```

Query Pods using the exact selector configured in the Service:

```powershell
kubectl get pods -l app=wrong-backend-selector
```

Output:
```text
No resources found in default namespace.
```

### Step 2: List actual running Pods and their labels
```powershell
kubectl get pods -l troubleshooting=networking --show-labels
```

Output:
```text
NAME                                   READY   STATUS    RESTARTS   AGE   LABELS
deploy-no-endpoints-7598c9f598-4k2jm   1/1     Running   0          50s   app=backend-api,troubleshooting=networking
deploy-no-endpoints-7598c9f598-b8d9q   1/1     Running   0          50s   app=backend-api,troubleshooting=networking
```
*Notice that the actual Pods have label `app=backend-api`, but the Service is looking for `app=wrong-backend-selector`.*

### Step 3: Inspect Service details
```powershell
kubectl describe service svc-no-endpoints
```

Examine the output:
```text
Selector:          app=wrong-backend-selector
Type:              ClusterIP
IP:                10.96.X.X
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
```

**Diagnostic analysis:**
- The 2 backend Pods are running and healthy (`1/1 Ready`).
- The Service selector `app=wrong-backend-selector` does not match the Pod label `app=backend-api`.
- Because there is no label overlap, the EndpointSlice controller cannot associate any Pod IPs with the Service.

---

## 6. Root cause
In `invalid-service.yaml`, lines 31-32 specify:
```yaml
selector:
  app: wrong-backend-selector
```
The Deployment's Pod template defines `app: backend-api`. Because the Service selector does not match the Pod labels, zero endpoints are registered.

---

## 7. Fix the issue
Apply the corrected manifest:

```powershell
kubectl apply -f .\fixed-service.yaml
```

The corrected manifest `fixed-service.yaml` updates the Service selector to:
```yaml
selector:
  app: backend-api
```

---

## 8. Verify the fix
Inspect the Service endpoints immediately:

```powershell
kubectl get endpoints svc-no-endpoints
```

Expected output:
```text
NAME               ENDPOINTS                           AGE
svc-no-endpoints   10.244.0.12:80,10.244.0.13:80       2m
```

Verify the EndpointSlice:

```powershell
kubectl get endpointslices -l kubernetes.io/service-name=svc-no-endpoints
```

Expected output:
```text
NAME                     ADDRESSTYPE   PORTS   ENDPOINTS               AGE
svc-no-endpoints-4xk9p   IPv4          80      10.244.0.12,10.244.0.13 2m
```

Test cluster network connectivity using an ephemeral client:

```powershell
$svcIP = (kubectl get service svc-no-endpoints -o jsonpath='{.spec.clusterIP}')
kubectl run curl-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-empty-endpoints.png` — Output of `kubectl get endpoints` showing `<none>`.
- [ ] `02-selector-pod-label-mismatch.png` — Output comparing `kubectl get svc -o jsonpath` selector with `kubectl get pods --show-labels`.
- [ ] `03-describe-service-none.png` — Output of `kubectl describe service` showing `Endpoints: <none>`.
- [ ] `04-verified-endpoints-populated.png` — Output of `kubectl get endpoints` showing both Pod IP addresses.

---

## 10. Cleanup
Delete the Deployment and Service:

```powershell
kubectl delete -f .\fixed-service.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get service svc-no-endpoints
kubectl get deployment deploy-no-endpoints
```

---

## 11. Key learning
- **Root cause:** The Service selector did not match the Pod template labels, preventing the EndpointSlice controller from discovering the backend Pods.
- **Most useful diagnostic command:** `kubectl get endpoints <service-name>` and comparing `kubectl get svc <name> -o yaml` selectors with `kubectl get pods --show-labels`.
- **Correction:** Align the Service's `spec.selector` with the Pod template's `metadata.labels`.
- **Production consideration:** Standardize label naming conventions (e.g. `app.kubernetes.io/name` and `app.kubernetes.io/instance` from recommended Kubernetes guidelines) to avoid label drift.
