# Troubleshooting: DNS Resolution Failure

## 1. Issue overview
A DNS resolution failure occurs when a Pod in a Kubernetes cluster cannot resolve a Service name, FQDN (Fully Qualified Domain Name), or external hostname into an IP address.

In Kubernetes:
- Cluster DNS is managed by **CoreDNS** (running in the `kube-system` namespace).
- Standard Kubernetes internal FQDN format follows:
  $$\text{<service-name>}.\text{<namespace>}.\text{svc}.\text{cluster.local}$$
- When a client Pod queries a Service within the **same namespace**, it can use the short name `auth-service`.
- When querying across namespaces, the namespace must be explicitly included (e.g. `auth-service.default.svc.cluster.local`).
- If an application queries an incorrect namespace (such as `.production.` when the service is in `.default.`), CoreDNS returns `NXDOMAIN` (Non-Existent Domain).

**Expected behavior:** The client Pod resolves the backend Service IP and establishes communication.  
**Actual behavior:** The client Pod queries `auth-service.production.svc.cluster.local`, which does not exist, causing `nslookup` to fail with `NXDOMAIN`.

---

## 2. Common causes
1. **Wrong namespace in FQDN:** Querying a service under the wrong namespace (e.g. `production` instead of `default`).
2. **Typo in Service name:** Name in DNS lookup does not match the `metadata.name` of the target Service.
3. **CoreDNS Pods down or crashing:** CoreDNS deployment in `kube-system` is in `CrashLoopBackOff` or has 0 available replicas.
4. **Invalid resolv.conf configuration:** Pod's `/etc/resolv.conf` has missing nameservers or broken `ndots:5` configuration.
5. **NetworkPolicy blocking port 53 (UDP/TCP):** Egress network policy prevents the client Pod from reaching CoreDNS on port 53.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\04-dns-resolution-failure"
```

Apply the invalid test manifest:

```powershell
kubectl apply -f .\invalid-dns-test.yaml
```

Wait 10 seconds for the client Pod to start, then inspect its logs:

```powershell
kubectl logs dns-test-client
```

Observe the repeated DNS failure:
```text
Server:         10.96.0.10
Address:        10.96.0.10:53

** server can't find auth-service.production.svc.cluster.local: NXDOMAIN
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Check CoreDNS health in the cluster
Before assuming an application configuration error, verify that the cluster DNS infrastructure itself is healthy:

```powershell
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Expected output:
```text
NAME                       READY   STATUS    RESTARTS   AGE
coredns-768b85b76f-5x4lp   1/1     Running   0          5d
```
CoreDNS is running and healthy, proving this is not an infrastructure outage.

### Step 2: Inspect the client Pod's DNS resolver settings
Inspect `/etc/resolv.conf` inside the client Pod:

```powershell
kubectl exec dns-test-client -- cat /etc/resolv.conf
```

Expected output:
```text
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```
*Notice that the default search domain is `default.svc.cluster.local`.*

### Step 3: Test resolving the short name vs the failing FQDN
Test the short name `auth-service`:

```powershell
kubectl exec dns-test-client -- nslookup auth-service
```

Expected output:
```text
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   auth-service.default.svc.cluster.local
Address: 10.96.X.X
```

Now test the failing name:

```powershell
kubectl exec dns-test-client -- nslookup auth-service.production.svc.cluster.local
```

Output:
```text
** server can't find auth-service.production.svc.cluster.local: NXDOMAIN
```

**Diagnostic analysis:**
- CoreDNS is functioning properly.
- The Service `auth-service` exists in the `default` namespace.
- The client application was configured to query `auth-service.production.svc.cluster.local`, which returns `NXDOMAIN` because no such service exists in namespace `production`.

---

## 6. Root cause
In `invalid-dns-test.yaml`, line 52 queries:
```yaml
command: ["sh", "-c", "while true; do nslookup auth-service.production.svc.cluster.local; sleep 10; done"]
```
The Service was created in the `default` namespace, not `production`.

---

## 7. Fix the issue
Delete the failing client Pod and apply the fixed manifest:

```powershell
kubectl delete pod dns-test-client --ignore-not-found=true
kubectl apply -f .\fixed-dns-test.yaml
```

The corrected manifest `fixed-dns-test.yaml` updates the DNS lookup to the correct FQDN:
`auth-service.default.svc.cluster.local` (or the short name `auth-service`).

---

## 8. Verify the fix
Inspect the client Pod logs:

```powershell
kubectl logs -f dns-test-client --tail=10
```

Expected output:
```text
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   auth-service.default.svc.cluster.local
Address: 10.96.X.X
```

Verify that HTTP connectivity through DNS succeeds:

```powershell
kubectl exec dns-test-client -- wget -q -O - http://auth-service.default.svc.cluster.local | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-dns-nxdomain-logs.png` — Output of `kubectl logs dns-test-client` showing `NXDOMAIN`.
- [ ] `02-coredns-pods-running.png` — Output of `kubectl get pods -n kube-system -l k8s-app=kube-dns` showing CoreDNS is healthy.
- [ ] `03-resolv-conf-output.png` — Output of `kubectl exec cat /etc/resolv.conf`.
- [ ] `04-verified-dns-resolution.png` — Output of `kubectl logs` showing successful resolution to `10.96.X.X`.

---

## 10. Cleanup
Delete all test resources:

```powershell
kubectl delete -f .\fixed-dns-test.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get service auth-service
kubectl get pod dns-test-client
```

---

## 11. Key learning
- **Root cause:** The client queried a non-existent namespace in the service FQDN, returning `NXDOMAIN`.
- **Most useful diagnostic command:** `kubectl exec <pod> -- nslookup <name>` to test DNS resolution directly from the client Pod's network namespace.
- **Correction:** Use the correct namespace in the FQDN (`<service>.<namespace>.svc.cluster.local`) or use the short name for same-namespace traffic.
- **Production consideration:** Be aware of `ndots:5` in Kubernetes: querying unqualified domain names can cause up to 4 sequential DNS queries before reaching the external DNS resolver, causing DNS latency if CoreDNS is overloaded.
