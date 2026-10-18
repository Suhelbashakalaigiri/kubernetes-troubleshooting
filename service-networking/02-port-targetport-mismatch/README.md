# Troubleshooting: Port / targetPort Mismatch

## 1. Issue overview
A port / targetPort mismatch occurs when a Kubernetes Service forwards incoming traffic to a container port (`targetPort`) on which no application process is listening, resulting in immediate "connection refused" errors for connecting clients.

In Kubernetes:
- **`port`:** The port on which the Service itself listens within the cluster network (ClusterIP).
- **`targetPort`:** The port inside the target container where the Service proxies traffic.
- **`containerPort`:** Documented port inside the container specification where the application listens (such as port 80 for Nginx).
- If `targetPort` does not match the container process's actual listening port (e.g. Service forwards to port `8080` while Nginx listens on port `80`), endpoints are populated with the wrong port, and all connection attempts fail with TCP RST (connection refused).

**Expected behavior:** The Service forwards traffic from ClusterIP port 80 to container targetPort 80.  
**Actual behavior:** The Service forwards traffic to targetPort 8080 where nothing is listening, returning connection refused.

---

## 2. Common causes
1. **Typo in `targetPort`:** Forwarding to port 8080 instead of 80, or 3000 instead of 8000.
2. **Container port changed without updating Service:** Application migrated from HTTP (80) to HTTPS (443) or custom port, but Service was not updated.
3. **Named port mismatch:** Service `targetPort` references a port name (e.g. `http`) that does not match `name` in the container's `ports` definition.
4. **Application listening on `127.0.0.1` instead of `0.0.0.0`:** Container process binds only to loopback interface, rejecting packets from the Pod's veth interface.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\02-port-targetport-mismatch"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-service.yaml
```

Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod -l app=port-mismatch-demo --timeout=30s
```

Check the Service and its endpoints:

```powershell
kubectl get service svc-port-mismatch
kubectl get endpoints svc-port-mismatch
```

Output:
```text
NAME                ENDPOINTS          AGE
svc-port-mismatch   10.244.0.15:8080   25s
```
*Note that endpoints exist, but point to port 8080!*

Test connecting to the Service from an ephemeral client Pod:

```powershell
$svcIP = (kubectl get service svc-port-mismatch -o jsonpath='{.spec.clusterIP}')
kubectl run client-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -T 3 -q -O - "http://$svcIP:80"
```

Observe the connection failure:
```text
wget: can't connect to remote host (...): Connection refused
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect the Service ports and endpoint mapping
```powershell
kubectl describe service svc-port-mismatch
```

Examine the **Port** and **Endpoints** fields:
```text
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.244.0.15:8080
```
Notice that the Service forwards traffic to port `8080`.

### Step 2: Verify the container's actual listening port
Inspect what ports the container process is actually listening on:

```powershell
$pod = (kubectl get pods -l app=port-mismatch-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $pod -- netstat -tuln
```

Output:
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      
```
*Nginx is listening on port `80`, NOT port `8080`.*

### Step 3: Test direct Pod connectivity
Test connecting directly to the Pod's IP on port 80:

```powershell
$podIP = (kubectl get pod $pod -o jsonpath='{.status.podIP}')
kubectl run direct-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$podIP:80" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```
Direct connection to port 80 succeeds, confirming that the Pod is healthy and that the problem lies entirely in the Service `targetPort` routing to 8080.

---

## 6. Root cause
In `invalid-service.yaml`, line 35 specifies:
```yaml
ports:
- name: http
  port: 80
  targetPort: 8080
```
Nginx listens on port 80, not 8080. The Service forwards incoming traffic to port 8080, where the kernel immediately sends a TCP RST (`Connection refused`).

---

## 7. Fix the issue
Apply the fixed manifest:

```powershell
kubectl apply -f .\fixed-service.yaml
```

The corrected manifest `fixed-service.yaml` updates `targetPort: 8080` to `targetPort: 80`.

---

## 8. Verify the fix
Check the updated endpoints:

```powershell
kubectl get endpoints svc-port-mismatch
```

Expected output:
```text
NAME                ENDPOINTS        AGE
svc-port-mismatch   10.244.0.15:80   2m
```

Test connecting to the Service ClusterIP again:

```powershell
$svcIP = (kubectl get service svc-port-mismatch -o jsonpath='{.spec.clusterIP}')
kubectl run client-verify --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP:80" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-connection-refused-error.png` — Output of wget showing `Connection refused` when connecting through the Service.
- [ ] `02-describe-service-targetport.png` — Output of `kubectl describe service` showing `TargetPort: 8080/TCP`.
- [ ] `03-netstat-port-80.png` — Output of `kubectl exec netstat -tuln` proving container listens on port 80.
- [ ] `04-verified-service-response.png` — Output of wget showing `Welcome to nginx` after correcting `targetPort: 80`.

---

## 10. Cleanup
Delete the Deployment and Service:

```powershell
kubectl delete -f .\fixed-service.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get service svc-port-mismatch
kubectl get deployment deploy-port-mismatch
```

---

## 11. Key learning
- **Root cause:** The Service's `targetPort: 8080` did not match the container's listening port (80), resulting in connection refused errors.
- **Most useful diagnostic command:** `kubectl describe service <name>` to check `TargetPort` and running `kubectl exec <pod> -- netstat -tuln` to verify active listening ports.
- **Correction:** Align `targetPort` with the actual application listening port.
- **Production consideration:** Use named ports in both the container specification (`name: http`) and Service (`targetPort: http`); this decouples port numbers and prevents breaking Services when container port numbers change.
