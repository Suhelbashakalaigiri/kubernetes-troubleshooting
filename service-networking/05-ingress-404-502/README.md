# Troubleshooting: Ingress 404 / 502 Errors

## 1. Issue overview
An Ingress resource provides HTTP/HTTPS routing from outside the cluster to internal Services. When an Ingress misbehaves, it typically produces one of two errors:
- **HTTP 404 (Not Found):** Returned directly by the Ingress Controller when no routing rule matches the incoming `Host` header or URL `path`.
- **HTTP 502 (Bad Gateway):** Returned by the Ingress Controller when a routing rule matches, but the controller cannot establish a connection to the backend Service (e.g. backend service port is closed, pods are unready, or connection refused).

In Kubernetes:
- Ingress resources require an active **Ingress Controller** (such as `ingress-nginx`). Without an Ingress controller, Ingress objects are inert declarations that perform no routing.
- In Minikube, the controller is enabled using `minikube addons enable ingress`.
- In `invalid-ingress.yaml`, the rule matches path `/api/v2`, but clients query root `/`, resulting in an immediate 404 from the Ingress Controller.

**Expected behavior:** The Ingress routes `demo.local/` to `svc-ingress-backend:80`, returning HTTP 200.  
**Actual behavior:** The Ingress rule requires `/api/v2`; requests to `demo.local/` return HTTP 404 from the Ingress Controller.

---

## 2. Common causes
1. **Path routing mismatch (404):** Ingress path is `/api/v2`, but client requests `/` (or missing rewrite-target annotations).
2. **Host header mismatch (404):** Request `Host` header does not match `spec.rules.host`.
3. **Backend Service port mismatch (502):** Ingress rule points to port 8080, but Service listens on port 80.
4. **Backend Pods unready or crashing (502):** Service has 0 healthy endpoints, forcing the Ingress controller to return 502.
5. **Missing Ingress Controller:** No controller is running in the cluster; `kubectl get ingress` shows an empty `ADDRESS`.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.
- *Ingress Controller prerequisite:* The Minikube ingress controller is required to test live ingress routing. Verify if it is running with:
  ```powershell
  kubectl get pods -n ingress-nginx
  ```
  *(If the controller is not running, run `minikube addons enable ingress` when permitted by your environment).*

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\05-ingress-404-502"
```

Apply the invalid manifests:

```powershell
kubectl apply -f .\invalid-ingress.yaml
```

Wait for the backend Pod and Ingress to be created:

```powershell
kubectl get ingress test-ingress
```

Output:
```text
NAME           CLASS   HOSTS        ADDRESS        PORTS   AGE
test-ingress   nginx   demo.local   192.168.49.2   80      20s
```

Test querying root `/` with the host header `demo.local`:

```powershell
$ingressIP = (kubectl get ingress test-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
if (-not $ingressIP) { $ingressIP = (kubectl get node minikube -o jsonpath='{.status.addresses[0].address}') }

kubectl run curl-ingress --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -S -O /dev/null "http://$ingressIP/" --header="Host: demo.local" 2>&1 | Select-String "HTTP/"
```

Observe the 404 response:
```text
HTTP/1.1 404 Not Found
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the Ingress routing rules
```powershell
kubectl describe ingress test-ingress
```

Examine the **Rules** table:
```text
Rules:
  Host        Path  Backends
  ----        ----  --------
  demo.local  
              /api/v2   svc-ingress-backend:80 (10.244.0.18:80)
```
Notice:
- The path is explicitly set to `/api/v2`.
- Any request sent to `/` does not match the rule and is routed to the default backend (HTTP 404).

### Step 2: Test the configured path `/api/v2`
Query `/api/v2`:

```powershell
kubectl run curl-path --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -S -O /dev/null "http://$ingressIP/api/v2" --header="Host: demo.local" 2>&1 | Select-String "HTTP/"
```

Output:
```text
HTTP/1.1 404 Not Found (or 200 if rewrite configured)
```
*Notice: `/api/v2` forwarded to Nginx, but Nginx has no `/api/v2` content unless rewrite annotations are used.*

### Step 3: Distinguish 404 vs 502
- **If you receive 404:** Check the Ingress `rules.http.paths` and `host` headers.
- **If you receive 502:** Check backend Service endpoints (`kubectl get endpoints svc-ingress-backend`) to see if backend Pods are crashing.

---

## 6. Root cause
In `invalid-ingress.yaml`, lines 46-47 specify:
```yaml
paths:
- path: /api/v2
  pathType: Prefix
```
The Ingress rule only routes requests under `/api/v2`. Standard client requests to `/` trigger an immediate HTTP 404 from the Ingress controller.

---

## 7. Fix the issue
Apply the fixed Ingress manifest:

```powershell
kubectl apply -f .\fixed-ingress.yaml
```

The corrected manifest `fixed-ingress.yaml` updates the path rule to:
```yaml
paths:
- path: /
  pathType: Prefix
```

---

## 8. Verify the fix
Query the root path `/` again:

```powershell
kubectl run curl-verify --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -S -O - "http://$ingressIP/" --header="Host: demo.local" 2>&1 | Select-String "Welcome to nginx","HTTP/1.1 200"
```

Expected output:
```text
HTTP/1.1 200 OK
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-ingress-status.png` — Output of `kubectl get ingress` showing IP and host.
- [ ] `02-ingress-404-error.png` — Output showing `HTTP/1.1 404 Not Found` on `/`.
- [ ] `03-describe-ingress-rules.png` — Output of `kubectl describe ingress` showing path `/api/v2`.
- [ ] `04-verified-ingress-200.png` — Output showing `HTTP/1.1 200 OK` after applying `path: /`.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-ingress.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get ingress test-ingress
```

---

## 11. Key learning
- **Root cause:** Ingress path `/api/v2` rejected requests to root `/`, resulting in Ingress Controller 404 errors.
- **Most useful diagnostic command:** `kubectl describe ingress <name>` to verify host and path matching rules against client URLs.
- **Correction:** Align Ingress paths (`path: /`) and pathType (`Prefix`) with expected client routes.
- **Production consideration:** Use `nginx.ingress.kubernetes.io/rewrite-target` annotations when proxying prefixed paths (e.g. `/api/v1`) to applications that expect root `/`.
