# Troubleshooting: NetworkPolicy Blocks Communication

## 1. Issue overview
A NetworkPolicy in Kubernetes allows operators to control ingress and egress traffic at the IP packet level (layer 3 / layer 4). By default, Pods are non-isolated and accept traffic from any source. Once a NetworkPolicy selects a Pod, that Pod becomes isolated, and all traffic not explicitly whitelisted is dropped.

In Kubernetes:
- A NetworkPolicy with `policyTypes: ["Ingress"]` and `ingress: []` creates a **Default Deny Ingress** rule.
- Any client trying to connect to the Pod encounters packets silently dropped, resulting in a connection timeout.
- **Critical CNI Plugin Requirement:**
  - Kubernetes NetworkPolicies are **declarative specifications enforced by the cluster CNI plugin**.
  - Default Minikube installations using the standard bridge network do not enforce NetworkPolicies unless started with a policy-enforcing CNI (e.g. `minikube start --cni=calico` or `cilium`).
  - If your cluster lacks a policy-enforcing CNI, the NetworkPolicy object exists in `etcd`, but packets will not be blocked.

**Expected behavior:** Authorized client Pods with label `role: frontend` can connect to `svc-secure-db`.  
**Actual behavior:** `invalid-networkpolicy.yaml` defines `ingress: []` (deny all ingress), blocking all connections.

---

## 2. Common causes
1. **Default Deny Ingress without allow rules:** Policy isolates Pods with `ingress: []`, blocking all traffic including health checks and monitoring.
2. **Missing namespaceSelector in cross-namespace policies:** Ingress rule specifies `podSelector` but omits `namespaceSelector`, accidentally restricting access to only the local namespace.
3. **Mismatched client labels in `from.podSelector`:** Client Pod does not carry the exact labels required by the NetworkPolicy.
4. **Port restriction in NetworkPolicy:** Policy permits traffic from the client, but restricts traffic to port 443 while client connects to port 80.
5. **CNI does not support NetworkPolicy:** Policy is valid YAML, but cluster CNI fails to enforce it.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\service-networking\06-networkpolicy-block"
```

Apply the invalid manifests:

```powershell
kubectl apply -f .\invalid-networkpolicy.yaml
```

Wait for the database Deployment to be Ready:

```powershell
kubectl wait --for=condition=Ready pod -l app=secure-db --timeout=30s
```

Now launch a client Pod with label `role: frontend` and attempt to connect:

```powershell
$svcIP = (kubectl get service svc-secure-db -o jsonpath='{.spec.clusterIP}')
kubectl run frontend-client --image=busybox:1.36 --labels="role=frontend" --restart=Never --rm -i --timeout=10s -- wget -T 3 -q -O - "http://$svcIP"
```

On a cluster with a policy-enforcing CNI (e.g. Calico), the connection times out:
```text
wget: download timed out
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect active NetworkPolicies selecting the target Pod
```powershell
kubectl get networkpolicies
```

Output:
```text
NAME                   POD-SELECTOR    AGE
block-all-ingress-db   app=secure-db   40s
```

### Step 2: Describe the NetworkPolicy rules
```powershell
kubectl describe networkpolicy block-all-ingress-db
```

Output:
```text
Name:         block-all-ingress-db
Namespace:    default
Labels:       troubleshooting=networking
Spec:
  PodSelector:     app=secure-db
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Not affecting egress traffic
  Policy Types:    Ingress
```
*Notice: `Allowing ingress traffic: <none>`. All ingress traffic to `app=secure-db` is blocked!*

### Step 3: Check CNI enforcement status
Verify which CNI is installed in your cluster:

```powershell
kubectl get pods -n kube-system | Select-String "calico","cilium","weave","flannel"
```

**Diagnostic analysis:**
- The NetworkPolicy explicitly targets `app=secure-db`.
- It defines `policyTypes: ["Ingress"]` with an empty `ingress: []` list.
- This creates a total ingress blackhole, causing client SYN packets to be dropped with connection timeouts.

---

## 6. Root cause
In `invalid-networkpolicy.yaml`, lines 47-49 specify:
```yaml
policyTypes:
- Ingress
ingress: []
```
This isolates the `secure-db` Pods and drops 100% of incoming traffic from all sources.

---

## 7. Fix the issue
Apply the fixed NetworkPolicy manifest:

```powershell
kubectl apply -f .\fixed-networkpolicy.yaml
```

The corrected manifest `fixed-networkpolicy.yaml` adds an ingress rule allowing traffic from Pods matching `role: frontend` on port 80:
```yaml
ingress:
- from:
  - podSelector:
      matchLabels:
        role: frontend
  ports:
  - protocol: TCP
    port: 80
```

---

## 8. Verify the fix
Describe the updated NetworkPolicy:

```powershell
kubectl describe networkpolicy block-all-ingress-db
```

Output:
```text
Allowing ingress traffic:
  Peer:
    PodSelector: role=frontend
  Port: 80/TCP
```

Test connecting from an authorized Pod (`role=frontend`):

```powershell
$svcIP = (kubectl get service svc-secure-db -o jsonpath='{.spec.clusterIP}')
kubectl run authorized-client --image=busybox:1.36 --labels="role=frontend" --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP" | Select-String "Welcome to nginx"
```

Expected output:
```text
<h1>Welcome to nginx!</h1>
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-networkpolicy-deny-all.png` — Output of `kubectl describe networkpolicy` showing `Allowing ingress traffic: <none>`.
- [ ] `02-client-timeout.png` — Output of wget timing out when attempting to reach the database.
- [ ] `03-fixed-networkpolicy-described.png` — Output showing `Peer: PodSelector: role=frontend, Port: 80/TCP`.
- [ ] `04-verified-authorized-traffic.png` — Output of authorized client successfully retrieving `Welcome to nginx`.

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-networkpolicy.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get networkpolicy block-all-ingress-db
kubectl get deployment deploy-secure-db
```

---

## 11. Key learning
- **Root cause:** An empty `ingress: []` block in a NetworkPolicy isolated the target Pods, dropping all incoming traffic.
- **Most useful diagnostic command:** `kubectl describe networkpolicy <name>` to view peer selectors and allowed port lists.
- **Correction:** Add explicit `from.podSelector` and `ports` entries to permit authorized traffic.
- **Production consideration:** Always remember that NetworkPolicies require a policy-enforcing CNI (like Calico or Cilium); without one, policies are silently ignored by the cluster networking layer.
