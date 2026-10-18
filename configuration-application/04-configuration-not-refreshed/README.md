# Troubleshooting: Configuration Not Refreshed

## 1. Issue overview
A common Kubernetes misconception is that modifying a `ConfigMap` or `Secret` will automatically update environment variables inside already-running containers.

In Kubernetes:
- **Environment variables are immutable for running processes:** Operating system processes receive environment variables only at creation time (via the `execve` system call). Once a container process starts, the Linux kernel does not allow external processes to modify its `/proc/<pid>/environ` memory block.
- Updating a ConfigMap does **NOT** update environment variables in running Pods.
- To refresh environment-variable configuration, operators must trigger a rolling restart of the Deployment using:
  ```powershell
  kubectl rollout restart deployment/<name>
  ```
- **Volume Mounts vs. Environment Variables:** If a ConfigMap is mounted as a **Volume**, the Kubelet automatically updates the mounted files on disk using atomic symlink rotation (within ~60–90 seconds) without restarting the Pod.

**Expected behavior:** Updating `FEATURE_MODE` in `feature-flags` updates the application's active configuration.  
**Actual behavior:** The running container continues printing the old value `v1_basic` because environment variables cannot be refreshed without restarting the Pod or using volume mounts.

---

## 2. Common causes
1. **Expecting live env-var mutation:** Misunderstanding that Linux container processes cannot receive live env updates.
2. **Missing `kubectl rollout restart`:** Forgetting to restart the Deployment after updating a ConfigMap.
3. **Application caching configuration:** The application reads config files only on startup and does not watch the filesystem for changes.
4. **SubPath volume mounts:** ConfigMaps mounted via `subPath` do **NOT** receive automatic filesystem updates.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\04-configuration-not-refreshed"
```

Apply the initial ConfigMap and Deployment:

```powershell
kubectl apply -f .\configmap.yaml
kubectl apply -f .\invalid-deployment.yaml
kubectl rollout status deployment/deploy-config-refresh --timeout=60s
```

Check the active container logs:

```powershell
$pod = (kubectl get pods -l app=config-refresh-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod --tail=3
```

Output:
```text
Current mode: v1_basic
Current mode: v1_basic
```

Now update the ConfigMap in-place to `v2_advanced`:

```powershell
kubectl patch configmap feature-flags --type=merge -p '{"data":{"FEATURE_MODE":"v2_advanced"}}'
```

Verify that the ConfigMap now contains `v2_advanced`:

```powershell
kubectl get configmap feature-flags -o jsonpath='{.data.FEATURE_MODE}'
```

Now check the container logs again:

```powershell
kubectl logs $pod --tail=3
```

Observe that the container still prints `v1_basic`:
```text
Current mode: v1_basic
Current mode: v1_basic
```
*The running container's environment variable did NOT refresh!*

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Compare ConfigMap data with running Pod environment
Query the live ConfigMap:

```powershell
kubectl get configmap feature-flags -o yaml
```
*ConfigMap contains `FEATURE_MODE: v2_advanced`.*

Query the running Pod's actual process environment:

```powershell
kubectl exec $pod -- env | Select-String "FEATURE_MODE"
```

Output:
```text
FEATURE_MODE=v1_basic
```

**Diagnostic analysis:**
- The ConfigMap in etcd is updated to `v2_advanced`.
- The running Pod's process environment remains `v1_basic`.
- This proves that environment variable injection is static and only occurs when a Pod is first created.

---

## 6. Root cause
`invalid-deployment.yaml` injects configuration via `env.valueFrom.configMapKeyRef`. The Linux kernel does not allow updating environment variables of a running process; changes require a Pod restart or a mounted volume.

---

## 7. Fix the issue: Two Solutions

### Solution 1: Trigger a Rollout Restart (for env-var workloads)
```powershell
kubectl rollout restart deployment/deploy-config-refresh
kubectl rollout status deployment/deploy-config-refresh
```
*New Pods are created, reading the new `v2_advanced` value.*

### Solution 2: Use Mounted ConfigMap Volumes (for dynamic updates)
Apply `fixed-deployment.yaml`, which mounts the ConfigMap as a volume directory (`/etc/config/FEATURE_MODE`):

```powershell
kubectl apply -f .\fixed-deployment.yaml
kubectl rollout status deployment/deploy-config-refresh --timeout=60s
```

---

## 8. Verify the fix
Check the logs of the newly deployed volume-backed Pod:

```powershell
$newPod = (kubectl get pods -l app=config-refresh-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod --tail=3
```

Expected output:
```text
Volume mode: v2_advanced
Volume mode: v2_advanced
```

Now patch the ConfigMap again to `v3_dynamic`:

```powershell
kubectl patch configmap feature-flags --type=merge -p '{"data":{"FEATURE_MODE":"v3_dynamic"}}'
```

Inspect the mounted file inside the running container (wait ~30 seconds for Kubelet sync):

```powershell
Start-Sleep -Seconds 30
kubectl exec $newPod -- cat /etc/config/FEATURE_MODE
```

Expected output:
```text
v3_dynamic
```
*The volume-mounted file updated dynamically without restarting the Pod!*

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-initial-env-logs.png` — Output showing `Current mode: v1_basic`.
- [ ] `02-configmap-patched.png` — Output of `kubectl patch configmap` updating to `v2_advanced`.
- [ ] `03-stale-env-comparison.png` — Output showing ConfigMap has `v2_advanced` while `kubectl exec env` still has `v1_basic`.
- [ ] `04-verified-refreshed-output.png` — Output of `kubectl rollout restart` or volume-mounted dynamic update.

---

## 10. Cleanup
Delete the Deployment and ConfigMap:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
kubectl delete configmap feature-flags --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-config-refresh
kubectl get configmap feature-flags
```

---

## 11. Key learning
- **Root cause:** Environment variables injected from ConfigMaps are static and immutable for running processes.
- **Most useful diagnostic command:** Comparing `kubectl get configmap <name> -o yaml` with `kubectl exec <pod> -- env`.
- **Correction:** Run `kubectl rollout restart deployment/<name>` to reload environment variables, or mount ConfigMaps as Volumes for dynamic in-place file updates.
- **Production consideration:** Use configuration reloader operators (e.g. Stakater Reloader) to automatically trigger rolling updates whenever referenced ConfigMaps or Secrets are modified.
