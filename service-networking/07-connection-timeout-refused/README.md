# Troubleshooting: Connection Timeout vs. Connection Refused

## 1. Issue overview
In TCP networking and Kubernetes troubleshooting, **Connection Refused** and **Connection Timeout** are two fundamentally different failure modes that point to completely different root causes along the network path:

| Failure Mode | TCP Mechanism | What It Means in Kubernetes |
| :--- | :--- | :--- |
| **Connection Refused** | The destination host receives the TCP SYN packet and immediately replies with a **TCP RST** packet. | • The network path, routing, DNS, and IP address are **valid and reachable**.<br>• However, **no process is listening** on the requested destination port inside the target container. |
| **Connection Timeout** | The client sends a TCP SYN packet and **never receives any reply** until the client socket timeout expires. | • Packets are being **silently dropped or blackholed** along the network path.<br>• Caused by NetworkPolicies, security groups, firewall drops, non-routable IPs, or node network partitions. |

**Expected behavior:** The client connects through the Service to container port 80, receiving HTTP 200.  
**Actual behavior:** The Service targets port 9999 (where nothing is listening), causing the Linux kernel inside the Pod to immediately return a TCP RST, resulting in `Connection refused`.

---

## 2. Common causes
### Causes of Connection Refused:
1. Application process crashed or not running (`CrashLoopBackOff`).
2. Service `targetPort` misconfigured to a port where no container process is listening.
3. Application bound to `127.0.0.1` (localhost) instead of `0.0.0.0` (all interfaces).

### Causes of Connection Timeout:
1. NetworkPolicy ingress/egress rules dropping SYN packets.
2. Cloud Security Groups / external firewalls blocking node communication.
3. Asymmetric routing or broken CNI tunnel overlay (e.g. VXLAN/Geneve MTU mismatch).
4. Client connecting to an unassigned IP or invalid external subnet.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\07-connection-timeout-refused"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-service.yaml
```

Wait for the Pod to become Ready:

```powershell
kubectl wait --for=condition=Ready pod -l app=timeout-refused-demo --timeout=30s
```

Test connecting to the Service from an ephemeral client Pod:

```powershell
$svcIP = (kubectl get service svc-timeout-refused -o jsonpath='{.spec.clusterIP}')
kubectl run net-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- nc -z -v -w 3 $svcIP 80
```

Observe the immediate **Connection Refused** response:
```text
nc: 10.96.X.X (10.96.X.X:80): Connection refused
```
*Note that the error occurs instantly (< 100ms), proving the packet reached the target host.*

Now test what a **Connection Timeout** looks like by sending a SYN packet to a non-existent, unassigned blackhole IP (e.g. `10.96.254.254`):

```powershell
kubectl run timeout-test --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- nc -z -v -w 3 10.96.254.254 80
```

Observe the **Connection Timeout**:
```text
nc: 10.96.254.254 (10.96.254.254:80): Operation timed out
```
*Notice that the client waited the full 3 seconds before timing out, because packets were silently dropped.*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Trace the network path
Inspect the Service endpoint configuration:

```powershell
kubectl describe service svc-timeout-refused
```

Output:
```text
Port:              http  80/TCP
TargetPort:        9999/TCP
Endpoints:         10.244.0.X:9999
```

### Step 2: Inspect the Pod's active listening sockets
```powershell
$pod = (kubectl get pods -l app=timeout-refused-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $pod -- netstat -tuln
```

Output:
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      
```
- Nginx listens on port `80`.
- Service forwards to port `9999`.
- Because the Pod IP is reachable, the kernel receives the SYN on port 9999, sees no listening socket, and immediately transmits a TCP RST packet back to the client (`Connection refused`).

---

## 6. Root cause
In `invalid-service.yaml`, line 35 specifies:
```yaml
targetPort: 9999
```
Because no process listens on port 9999, the destination container rejects the TCP handshake with a TCP Reset (RST), causing `Connection refused`.

---

## 7. Fix the issue
Apply the corrected manifest:

```powershell
kubectl apply -f .\fixed-service.yaml
```

The corrected manifest `fixed-service.yaml` updates `targetPort: 9999` to `targetPort: 80`.

---

## 8. Verify the fix
Test the Service connection again:

```powershell
$svcIP = (kubectl get service svc-timeout-refused -o jsonpath='{.spec.clusterIP}')
kubectl run verify-net --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- nc -z -v -w 3 $svcIP 80
```

Expected output:
```text
10.96.X.X (10.96.X.X:80) open
```

Verify HTTP response:

```powershell
kubectl run verify-http --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP:80" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-connection-refused-output.png` — Output of `nc` showing immediate `Connection refused`.
- [ ] `02-connection-timeout-comparison.png` — Output of `nc` showing 3-second `Operation timed out`.
- [ ] `03-targetport-9999-describe.png` — Output of `kubectl describe service` showing `TargetPort: 9999`.
- [ ] `04-verified-port-open.png` — Output of `nc` showing port 80 `open` after fix.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-service.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get service svc-timeout-refused
kubectl get deployment deploy-timeout-refused
```

---

## 11. Key learning
- **Root cause:** Service targetPort 9999 targeted an unmapped port, prompting the kernel to reply with TCP RST (Connection refused).
- **Most useful diagnostic command:** `nc -z -v -w <sec> <ip> <port>` to observe immediate RST vs packet drop timeout.
- **Correction:** Align `targetPort` with the verified listening port of the container.
- **Production consideration:** Always distinguish connection timeouts (look at NetworkPolicies, security groups, routing) from connection refused (look at application crash status, container port bindings, and 0.0.0.0 vs 127.0.0.1 bindings).
