# Troubleshooting: NodePort Inaccessible

## 1. Issue overview
A NodePort Service exposes a workload on each node's IP at a static port in the range 30000–32767. Accessing a NodePort from a Windows host running Minikube involves two distinct diagnostic hurdles:
1. **Service configuration errors:** Misconfigured `targetPort` inside the Service manifest forwards traffic to a dead container port.
2. **Minikube on Windows networking boundaries:** In Windows environments, Minikube runs inside a virtualized Linux VM or Docker Desktop container. Because the VM network is isolated, attempting to browse directly to `localhost:30080` or the Minikube node IP from a Windows PowerShell terminal or browser fails by default unless bridged via `minikube service` or `kubectl port-forward`.

**Expected behavior:** The NodePort Service proxies traffic to targetPort 80, and operators use Minikube access mechanisms to reach the service from Windows.  
**Actual behavior:** `invalid-service.yaml` routes traffic to an incorrect `targetPort: 8080`, and browsing to the node port fails.

---

## 2. Common causes
1. **Misconfigured `targetPort`:** NodePort accepts traffic on port 30080, but forwards to the wrong container port (e.g. 8080 instead of 80).
2. **Host-to-VM network barrier in Minikube on Windows:** The Minikube node IP is inside a private hypervisor/WSL2 subnet not directly routable from the host OS.
3. **Firewall or security group blocks:** Port range 30000–32767 blocked by node OS firewalls.
4. **Kube-proxy routing failure:** `kube-proxy` iptables or IPVS rules not configured for the NodePort.
5. **NodePort conflict:** Attempting to assign a static `nodePort` already in use by another Service.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\03-nodeport-inaccessible"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-service.yaml
```

Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod -l app=nodeport-demo --timeout=30s
```

Check the NodePort Service:

```powershell
kubectl get service svc-nodeport
```

Output:
```text
NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
svc-nodeport   NodePort   10.96.X.X      <none>        80:30080/TCP   20s
```

Now retrieve the Minikube node IP:

```powershell
$nodeIP = (kubectl get node minikube -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
Write-Host "Node IP: $nodeIP"
```

Attempt connecting to the NodePort from an in-cluster test Pod:

```powershell
kubectl run nodeport-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -T 3 -q -O - "http://${nodeIP}:30080"
```

Observe the failure:
```text
wget: can't connect to remote host (...): Connection refused
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Diagnose the Service configuration
```powershell
kubectl describe service svc-nodeport
```

Examine the port mapping:
```text
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
NodePort:          <unset>  30080/TCP
Endpoints:         10.244.0.X:8080
```
Notice that `TargetPort` is set to `8080`, but the web server container is Nginx which listens on port `80`.

### Step 2: Understand Minikube on Windows network access
In Minikube on Windows:
- If using the `docker` driver (WSL2), the node IP belongs to the Docker bridge network, which Windows cannot route directly without an open tunnel.
- To safely expose and test any NodePort from Windows, Kubernetes provides:
  1. `minikube service <service-name> --url`: Starts a proxy tunnel or provides the mapped host port.
  2. `kubectl port-forward`: Forwards a local Windows host port directly to the Service.

---

## 6. Root cause
In `invalid-service.yaml`, line 34 specifies:
```yaml
targetPort: 8080
```
Traffic arriving on NodePort 30080 is routed to container port 8080 where no process is listening.

---

## 7. Fix the issue
Apply the fixed manifest:

```powershell
kubectl apply -f .\fixed-service.yaml
```

The corrected manifest `fixed-service.yaml` corrects `targetPort: 80`.

---

## 8. Verify the fix
Verify that the Service endpoints now target port 80:

```powershell
kubectl get endpoints svc-nodeport
```

Expected output:
```text
NAME           ENDPOINTS        AGE
svc-nodeport   10.244.0.X:80    2m
```

### Verification Path A: In-Cluster NodePort check
Test connecting to the Node IP and NodePort:

```powershell
$nodeIP = (kubectl get node minikube -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
kubectl run nodeport-verify --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://${nodeIP}:30080" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

### Verification Path B: Windows Host Access (Minikube / Port-Forward)
To access the Service from your local Windows web browser or PowerShell:

```powershell
# Method 1: Port-forward directly to localhost:8080 on Windows
kubectl port-forward service/svc-nodeport 8080:80
```
Then browse to `http://localhost:8080` in Windows.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-nodeport-service-created.png` — Output of `kubectl get service svc-nodeport` showing port `80:30080/TCP`.
- [ ] `02-targetport-mismatch-describe.png` — Output of `kubectl describe service` showing `TargetPort: 8080/TCP`.
- [ ] `03-incluster-nodeport-verified.png` — Output of wget showing `Welcome to nginx` via `NodeIP:30080`.
- [ ] `04-portforward-or-minikube-service.png` — Output of `kubectl port-forward` or `minikube service svc-nodeport --url`.

---

## 10. Cleanup
Delete the Deployment and Service:

```powershell
kubectl delete -f .\fixed-service.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get service svc-nodeport
kubectl get deployment deploy-nodeport
```

---

## 11. Key learning
- **Root cause:** The NodePort Service forwarded traffic to an incorrect `targetPort: 8080`.
- **Most useful diagnostic command:** `kubectl describe service <name>` to verify that `TargetPort` matches the container port, and using in-cluster test Pods to isolate cluster routing from host VM barriers.
- **Correction:** Align `targetPort` with container port 80.
- **Production consideration:** On cloud providers (AWS, Azure, GCP), prefer `type: LoadBalancer` with cloud load balancer integration or an Ingress controller rather than exposing static NodePorts on public internet nodes.
