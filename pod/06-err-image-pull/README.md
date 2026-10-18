# Troubleshooting: ErrImagePull (Registry Resolution Failure)

## 1. Issue overview
`ErrImagePull` is the immediate error state returned by the Kubelet when an initial attempt to fetch a container image from a container registry fails.

In Kubernetes:
- When a Pod starts, Kubelet calls the container runtime (e.g., `containerd`) to resolve and download the requested image.
- If this operation fails immediately—such as when the registry domain cannot be resolved via DNS, network routing fails, or authentication credentials are rejected—Kubelet sets the container state to `Waiting` with reason `ErrImagePull`.
- **Relationship between `ErrImagePull` and `ImagePullBackOff`:**
  - `ErrImagePull` is the **direct pull failure event** emitted immediately after the CRI pull attempt returns an error.
  - `ImagePullBackOff` is the **rate-limiting backoff state** that Kubernetes enters immediately after `ErrImagePull` to delay subsequent pull attempts.
  - In issue `01-image-pull-backoff`, we examined an existing registry with a non-existent tag. In this issue (`06-err-image-pull`), we examine a complete registry host/domain resolution failure (`non-existent-registry.invalid`) where the image registry endpoint itself cannot be resolved.

**Expected behavior:** Kubelet resolves the container registry host IP address, connects over TLS, authenticates, and downloads the image.  
**Actual behavior:** The registry hostname `non-existent-registry.invalid` cannot be resolved via DNS, producing an immediate `ErrImagePull` error.

---

## 2. Common causes
1. **Invalid or non-existent registry URL:** Typo in the hostname or domain (e.g., `.invalid` or misspelled private registry URL).
2. **DNS failure on the cluster node:** CoreDNS or node `/etc/resolv.conf` cannot resolve external or internal private registry domain names.
3. **Private registry authentication failure:** Container registry requires credentials (HTTP 401 Unauthorized), but no `imagePullSecrets` is attached.
4. **Firewall or proxy blocking:** Node egress traffic to the registry port (443) is blocked by corporate firewalls or missing HTTP_PROXY configuration.
5. **Self-signed TLS certificate rejection:** Private registry uses a custom CA certificate not trusted by the container runtime.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-err-image-pull"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Immediately query the Pod status (within 5 to 10 seconds):

```powershell
kubectl get pods pod-err-image-pull
```

You will observe `ErrImagePull`:
```text
NAME                 READY   STATUS         RESTARTS   AGE
pod-err-image-pull   0/1     ErrImagePull   0          7s
```

*(Note: after approximately 15 seconds, the status will alternate with `ImagePullBackOff` as backoff commences).*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and error details
```powershell
kubectl describe pod pod-err-image-pull
```

Examine the **Events** section:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  18s                default-scheduler  Successfully assigned default/pod-err-image-pull to minikube
  Normal   Pulling    17s                kubelet            Pulling image "non-existent-registry.invalid/demo/backend:v1.0.0"
  Warning  Failed     16s                kubelet            Failed to pull image "non-existent-registry.invalid/demo/backend:v1.0.0": rpc error: code = Unknown desc = failed to resolve image "non-existent-registry.invalid/demo/backend:v1.0.0": no such host
  Warning  Failed     16s                kubelet            Error: ErrImagePull
```

### Step 2: Extract the exact CRI error message via jsonpath
```powershell
kubectl get pod pod-err-image-pull -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
```

Expected diagnostic output:
```text
rpc error: code = Unknown desc = failed to resolve image "non-existent-registry.invalid/demo/backend:v1.0.0": no such host
```

**Diagnostic analysis:**
- The key phrase is `failed to resolve image ... no such host`.
- This proves that containerd could not resolve the hostname `non-existent-registry.invalid` via DNS.
- Contrast this with Issue 01 (`manifest unknown`): here, communication with the registry never even started because the registry domain does not exist.

---

## 6. Root cause
In `invalid-pod.yaml`, line 11 specifies `image: non-existent-registry.invalid/demo/backend:v1.0.0`. The domain name `non-existent-registry.invalid` is invalid and cannot be resolved by DNS, causing an immediate CRI `ErrImagePull` resolution failure.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` changes the image to `nginx:1.25-alpine`, hosted on the official, public Docker Hub registry with `imagePullPolicy: IfNotPresent`.

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-err-image-pull --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-err-image-pull -o wide
```

Expected output:
```text
NAME                 READY   STATUS    RESTARTS   AGE
pod-err-image-pull   1/1     Running   0          20s
```

Inspect container events to confirm clean pull:

```powershell
kubectl get events --field-selector involvedObject.name=pod-err-image-pull --sort-by=.metadata.creationTimestamp
```

Expected events:
- `Pulling image "nginx:1.25-alpine"`
- `Successfully pulled image "nginx:1.25-alpine"`
- `Created container app`
- `Started container app`

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-err-image-pull-status.png` — Output of `kubectl get pods` showing `STATUS: ErrImagePull`.
- [ ] `02-describe-pod-no-such-host.png` — Output of `kubectl describe pod pod-err-image-pull` highlighting `failed to resolve image ... no such host`.
- [ ] `03-jsonpath-error-message.png` — Output of the jsonpath command extracting the exact DNS/CRI error string.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-err-image-pull` showing `STATUS: Running`.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-err-image-pull
```

---

## 11. Key learning
- **Root cause:** Specifying an unreachable registry domain (`non-existent-registry.invalid`) caused a DNS resolution failure during CRI image pull.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` or jsonpath on `.status.containerStatuses[0].state.waiting.message` to identify whether the failure is DNS (`no such host`), authentication (`401 Unauthorized`), or missing tag (`manifest unknown`).
- **Correction:** Provide a valid, resolvable image registry URL and repository path.
- **Production consideration:** Use enterprise container registries (e.g., Harbor, Artifact Registry, ECR, ACR) with automated mirror failover and ensure cluster nodes have proper corporate DNS resolvers and proxy settings.
