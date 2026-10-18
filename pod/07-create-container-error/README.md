# Troubleshooting: CreateContainerError

## 1. Issue overview
`CreateContainerError` is a container waiting condition in Kubernetes that indicates the container runtime (such as `containerd` or `cri-o`) failed to create the container environment or execute the container's OCI specification on the node.

In Kubernetes:
- `CreateContainerError` is **technically distinct** from:
  - `ImagePullBackOff` / `ErrImagePull`: Image pulling succeeded completely.
  - `CreateContainerConfigError`: All Kubernetes configuration references (ConfigMaps, Secrets) were found and resolved.
- During `CreateContainerError`, the runtime has already downloaded and unpacked the image rootfs, but fails when trying to instantiate the container process (e.g., executing a command binary that does not exist in the rootfs, attempting an invalid volume mount path, or encountering a Linux kernel OCI namespace error).

**Expected behavior:** The container runtime creates the container namespaces and launches the specified executable binary inside the container rootfs.  
**Actual behavior:** The container specification defines an executable (`/entrypoint-custom-nonexistent.sh`) that does not exist in the image filesystem, causing the OCI runtime to fail container creation with `CreateContainerError`.

---

## 2. Common causes
1. **Non-existent executable or entrypoint:** The `command` specifies a path or binary not present in the container image rootfs.
2. **Missing execution permissions:** The target script or binary inside the container does not have the executable bit (`chmod +x`) set.
3. **Mount path conflict:** Mounting a volume over a non-directory path or conflicting with internal container system mounts.
4. **Unsupported sysctl or security profile:** Specifying an invalid AppArmor, Seccomp, or SELinux profile that the node runtime rejects.
5. **Conflicting port names:** Specifying duplicate container port names within the same Pod.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-create-container-error"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status:

```powershell
kubectl get pods pod-create-container-error
```

Observe `CreateContainerError`:
```text
NAME                         READY   STATUS                 RESTARTS   AGE
pod-create-container-error   0/1     CreateContainerError   0          20s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod container state and error messages
```powershell
kubectl describe pod pod-create-container-error
```

Examine the **Containers** status section:
```text
State:          Waiting
  Reason:       CreateContainerError
  Message:      failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "/entrypoint-custom-nonexistent.sh": stat /entrypoint-custom-nonexistent.sh: no such file or directory: unknown
Ready:          False
```

Examine the **Events** table:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  35s                default-scheduler  Successfully assigned default/pod-create-container-error to minikube
  Normal   Pulling    34s                kubelet            Pulling image "nginx:1.25-alpine"
  Normal   Pulled     33s                kubelet            Successfully pulled image "nginx:1.25-alpine" in 1.1s
  Warning  Failed     12s (x3 over 32s)  kubelet            Error: failed to create containerd task: ... exec: "/entrypoint-custom-nonexistent.sh": stat /entrypoint-custom-nonexistent.sh: no such file or directory: unknown
```

### Step 2: Extract the OCI runtime error message via jsonpath
```powershell
kubectl get pod pod-create-container-error -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
```

Output:
```text
failed to create containerd task: ... exec: "/entrypoint-custom-nonexistent.sh": stat /entrypoint-custom-nonexistent.sh: no such file or directory: unknown
```

**Diagnostic analysis:**
- Notice that the image was pulled successfully (`Successfully pulled image "nginx:1.25-alpine"`).
- No ConfigMap or Secret references are used, distinguishing this from `CreateContainerConfigError`.
- The OCI runtime explicitly reports `stat /entrypoint-custom-nonexistent.sh: no such file or directory`. The binary specified in the `command` field does not exist inside the image filesystem.

---

## 6. Root cause
In `invalid-pod.yaml`, line 12 defines:
```yaml
command: ["/entrypoint-custom-nonexistent.sh"]
```
Because `/entrypoint-custom-nonexistent.sh` does not exist inside the `nginx:1.25-alpine` image, containerd/runc cannot execute the process when initializing the container.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` replaces the non-existent entrypoint with the standard, valid Nginx startup command:
```yaml
command: ["nginx", "-g", "daemon off;"]
```

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-create-container-error --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-create-container-error -o wide
```

Expected output:
```text
NAME                         READY   STATUS    RESTARTS   AGE
pod-create-container-error   1/1     Running   0          22s
```

Verify container logs to confirm Nginx initialization:

```powershell
kubectl logs pod-create-container-error
```

Expected output:
```text
Configuration loaded, starting nginx worker processes...
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-create-container-error-status.png` — Output of `kubectl get pods` showing `STATUS: CreateContainerError`.
- [ ] `02-describe-pod-oci-error.png` — Output of `kubectl describe pod` showing the OCI runtime create failed error message.
- [ ] `03-jsonpath-stat-error.png` — Output of jsonpath showing `stat ... no such file or directory`.
- [ ] `04-verified-running-pod.png` — Output of `kubectl get pod pod-create-container-error` showing `STATUS: Running`.

---

## 10. Cleanup
Delete the Pod created for this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get pod pod-create-container-error
```

---

## 11. Key learning
- **Root cause:** The container specified a custom command binary (`/entrypoint-custom-nonexistent.sh`) that does not exist in the image filesystem.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the exact OCI runtime create failure message.
- **Correction:** Specify an executable binary and arguments that are verified to exist inside the target container image.
- **Production consideration:** Test container entrypoint scripts locally with `docker run` or `podman run` prior to deploying to Kubernetes, and ensure scripts have correct Linux shebangs (`#!/bin/sh` or `#!/bin/bash`) and execute permissions.
