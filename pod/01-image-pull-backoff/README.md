# Troubleshooting: ImagePullBackOff

## 1. Issue overview
`ImagePullBackOff` is a waiting condition where the container runtime (such as `containerd` inside Minikube) repeatedly fails to pull a specified container image from the container registry, causing the Kubelet to enter an exponential backoff loop before retrying.

In Kubernetes:
- When a Pod is scheduled onto a node, the Kubelet initiates an image pull request via the Container Runtime Interface (CRI).
- If the initial pull fails, the container status transitions to `ErrImagePull`.
- When the failure persists across retries, Kubelet backs off with increasing delay intervals (10s, 20s, 40s... up to 5 minutes) to avoid overloading the image registry and node network. During this backoff period, the Pod status displays `ImagePullBackOff`.

**Expected behavior:** The container image exists in the registry, network/authentication succeeds, Kubelet pulls the image layers, and the container transitions to `ContainerCreating` and then `Running`.  
**Actual behavior:** Kubelet cannot find the image tag, returns a 404/manifest unknown error, and places the container into `ImagePullBackOff`.

---

## 2. Common causes
1. **Invalid or misspelled image tag or repository name:** The specified tag does not exist on Docker Hub or the target registry.
2. **Missing image pull secrets:** The image is in a private registry and no `imagePullSecrets` was supplied.
3. **Registry rate limiting:** Exceeding public registry pull limits (e.g., Docker Hub anonymous rate limits).
4. **Network or DNS issues on the node:** Minikube node cannot resolve the registry host or route to the internet.
5. **Architecture mismatch:** The requested image manifest does not support the node's CPU architecture (e.g., pulling an ARM64-only image onto an AMD64 node).

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured to interact with your Minikube cluster.
- Internet connectivity on the Minikube node to reach Docker Hub (`registry-1.docker.io`).

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-image-pull-backoff"
```

Apply the invalid Pod manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Inspect the Pod status:

```powershell
kubectl get pods pod-image-pull-backoff
```

Initially, the status may show `ContainerCreating` or `ErrImagePull`. After approximately 15 to 30 seconds, query again:

```powershell
kubectl get pods pod-image-pull-backoff
```

You will observe:
```text
NAME                     READY   STATUS             RESTARTS   AGE
pod-image-pull-backoff   0/1     ImagePullBackOff   0          35s
```

View the cluster events sorted by creation time:

```powershell
kubectl get events --field-selector involvedObject.name=pod-image-pull-backoff --sort-by=.metadata.creationTimestamp
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Check Pod status and basic details
```powershell
kubectl get pod pod-image-pull-backoff -o wide
```
Confirm the Pod phase is `Pending` or `Waiting`, and identify which node the Pod was assigned to.

### Step 2: Inspect detailed Pod events and container state
```powershell
kubectl describe pod pod-image-pull-backoff
```

Look at the **Containers** section:
- `State: Waiting`
- `Reason: ImagePullBackOff`

Look at the **Events** table at the bottom of the output:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  45s                default-scheduler  Successfully assigned default/pod-image-pull-backoff to minikube
  Normal   Pulling    25s (x2 over 44s)  kubelet            Pulling image "nginx:nonexistent-tag-99999"
  Warning  Failed     24s (x2 over 43s)  kubelet            Failed to pull image "nginx:nonexistent-tag-99999": rpc error: code = NotFound desc = failed to pull and unpack image ... manifest unknown
  Warning  Failed     24s (x2 over 43s)  kubelet            Error: ErrImagePull
  Normal   BackOff    12s (x2 over 42s)  kubelet            Back-off pulling image "nginx:nonexistent-tag-99999"
  Warning  Failed     12s (x2 over 42s)  kubelet            Error: ImagePullBackOff
```

### Step 3: Extract the exact failure message programmatically
```powershell
kubectl get pod pod-image-pull-backoff -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
```

**Diagnostic analysis:**
- The message explicitly contains `manifest unknown` or `not found`.
- This confirms that network connectivity to Docker Hub is functioning, but the specific tag `nonexistent-tag-99999` does not exist in the public repository.
- Notice why `ImagePullBackOff` is not the initial error: the initial failure was `ErrImagePull`. `ImagePullBackOff` indicates that Kubelet is pausing before attempting another pull.

---

## 6. Root cause
In `invalid-pod.yaml`, line 11 specifies `image: nginx:nonexistent-tag-99999`. Because tag `nonexistent-tag-99999` is not published to Docker Hub's `library/nginx` repository, the registry returns a 404 manifest unknown error.

---

## 7. Fix the issue
Because the `image` field of an active standalone Pod is an immutable field or triggers an in-place container recreation, delete the failing Pod and apply the corrected manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` changes the image to `nginx:1.25-alpine`, a valid, lightweight public image, and sets `imagePullPolicy: IfNotPresent`.

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready` state:

```powershell
kubectl wait --for=condition=Ready pod/pod-image-pull-backoff --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-image-pull-backoff -o wide
```

Expected output:
```text
NAME                     READY   STATUS    RESTARTS   AGE
pod-image-pull-backoff   1/1     Running   0          25s
```

Verify the container logs:

```powershell
kubectl logs pod-image-pull-backoff
```

The output should confirm the Nginx web server started and is ready for requests.

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-invalid-pod-status.png` — Output of `kubectl get pods` showing `ImagePullBackOff`.
- [ ] `02-describe-pod-events.png` — Output of `kubectl describe pod pod-image-pull-backoff` showing the `manifest unknown` and `Back-off pulling image` events.
- [ ] `03-fixed-manifest.png` — Viewing `fixed-pod.yaml` showing the valid image tag `nginx:1.25-alpine`.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-image-pull-backoff` showing `STATUS: Running` and `READY: 1/1`.

---

## 10. Cleanup
Remove the test resources created during this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm that the Pod is terminated:

```powershell
kubectl get pod pod-image-pull-backoff
```

---

## 11. Key learning
- **Root cause:** Specifying a nonexistent image tag (`nginx:nonexistent-tag-99999`) caused the registry to return `manifest unknown`.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to examine the Events table for the exact registry error code and message.
- **Correction:** Reference a verified, existing image tag (`nginx:1.25-alpine`).
- **Production consideration:** Always use immutable, explicit image digests or semantic version tags in production instead of loose tags or typos, and verify image accessibility in CI/CD before updating Kubernetes manifests.
