# Troubleshooting: CreateContainerConfigError

## 1. Issue overview
`CreateContainerConfigError` is a container waiting condition in Kubernetes that occurs when the Kubelet attempts to construct the configuration parameters for a container, but is unable to resolve one or more referenced external configuration objects, such as a missing `ConfigMap` or `Secret`.

In Kubernetes:
- When a Pod spec uses `valueFrom.configMapKeyRef`, `valueFrom.secretKeyRef`, `envFrom.configMapRef`, or `envFrom.secretRef`, Kubelet must resolve these objects from the Kubernetes API before instructing the container runtime to create the container.
- If the referenced ConfigMap or Secret does not exist (or the specified key within the object is missing), and the reference is not marked with `optional: true`, Kubelet fails to generate the container configuration.
- The container remains in the `Waiting` state with reason `CreateContainerConfigError`.

**Expected behavior:** Kubelet retrieves the referenced ConfigMap `app-database-config-missing`, resolves the `db_host` key, injects it into the container environment, and starts the container.  
**Actual behavior:** ConfigMap `app-database-config-missing` does not exist in the namespace, causing Kubelet to halt container creation with `CreateContainerConfigError`.

---

## 2. Common causes
1. **Referencing a non-existent ConfigMap or Secret:** The object name is misspelled or has not been deployed yet.
2. **Missing key within an existing ConfigMap or Secret:** The object exists, but does not contain the specific key specified in `key: <key-name>`.
3. **Namespace mismatch:** The ConfigMap or Secret was created in a different namespace than the Pod.
4. **RBAC permission issues:** Kubelet or service account lacks read permissions for the target Secret.
5. **Omitting `optional: true`:** When a configuration reference is purely optional, omitting `optional: true` turns a missing reference into a blocking error.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\pod-create-container-config-error"
```

Apply the invalid manifest:

```powershell
kubectl apply -f .\invalid-pod.yaml
```

Check the Pod status:

```powershell
kubectl get pods pod-create-container-config-error
```

Observe:
```text
NAME                               READY   STATUS                       RESTARTS   AGE
pod-create-container-config-error  0/1     CreateContainerConfigError   0          25s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod container statuses and events
```powershell
kubectl describe pod pod-create-container-config-error
```

Check the **Containers** status block:
```text
State:          Waiting
  Reason:       CreateContainerConfigError
  Message:      configmap "app-database-config-missing" not found
```

Check the **Events** section:
```text
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  30s                default-scheduler  Successfully assigned default/pod-create-container-config-error to minikube
  Normal   Pulling    29s                kubelet            Pulling image "busybox:1.36"
  Normal   Pulled     28s                kubelet            Successfully pulled image "busybox:1.36"
  Warning  Failed     10s (x3 over 27s)  kubelet            Error: configmap "app-database-config-missing" not found
```

### Step 2: Query for the referenced ConfigMap in the namespace
```powershell
kubectl get configmap app-database-config-missing
```

Expected output:
```text
Error from server (NotFound): configmaps "app-database-config-missing" not found
```

### Step 3: Extract the exact failure message via jsonpath
```powershell
kubectl get pod pod-create-container-config-error -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
```

Output:
```text
configmap "app-database-config-missing" not found
```

**Diagnostic analysis:**
- Notice that the image `busybox:1.36` was pulled successfully (`Successfully pulled image`). This is not an image pull or container runtime binary issue.
- The failure happened at the Kubelet configuration assembly stage when resolving environment variables.
- Because `configmap "app-database-config-missing" not found` is recorded, the root cause is explicitly identified as a missing configuration object.

---

## 6. Root cause
In `invalid-pod.yaml`, lines 14-17 specify:
```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-database-config-missing
      key: db_host
```
The ConfigMap `app-database-config-missing` does not exist in the active namespace, and `optional: true` was not specified.

---

## 7. Fix the issue
Delete the failing Pod and apply the fixed multi-document manifest, which defines both the required ConfigMap and the updated Pod:

```powershell
kubectl delete -f .\invalid-pod.yaml
kubectl apply -f .ixed-pod.yaml
```

The corrected manifest `fixed-pod.yaml` creates `ConfigMap/app-database-config` with `db_host: "postgres.internal.local"` and configures the Pod to reference this verified ConfigMap.

---

## 8. Verify the fix
Wait for the Pod to reach `Running` and `Ready`:

```powershell
kubectl wait --for=condition=Ready pod/pod-create-container-config-error --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pod pod-create-container-config-error -o wide
```

Expected output:
```text
NAME                               READY   STATUS    RESTARTS   AGE
pod-create-container-config-error  1/1     Running   0          18s
```

Check the container logs to verify the environment variable was injected:

```powershell
kubectl logs pod-create-container-config-error
```

Expected output:
```text
App running with database host: postgres.internal.local
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-create-container-config-error-status.png` — Output of `kubectl get pods` showing `CreateContainerConfigError`.
- [ ] `02-describe-pod-configmap-missing.png` — Output of `kubectl describe pod` highlighting `configmap "app-database-config-missing" not found`.
- [ ] `03-configmap-notfound.png` — Output of `kubectl get configmap` showing `NotFound`.
- [ ] `04-verified-running-with-env.png` — Output of `kubectl logs` verifying `App running with database host: postgres.internal.local`.

---

## 10. Cleanup
Delete all resources created by this exercise:

```powershell
kubectl delete -f .ixed-pod.yaml --ignore-not-found=true
```

Verify deletion:

```powershell
kubectl get pod pod-create-container-config-error
kubectl get configmap app-database-config
```

---

## 11. Key learning
- **Root cause:** The Pod referenced a non-existent ConfigMap without `optional: true`.
- **Most useful diagnostic command:** `kubectl describe pod <pod-name>` to read the exact missing ConfigMap/Secret name from the container waiting message.
- **Correction:** Create the missing ConfigMap or update the reference to point to a valid resource and key.
- **Production consideration:** Always apply configuration manifests (ConfigMaps and Secrets) prior to deploying dependent workloads in deployment pipelines, or specify `optional: true` when values are non-essential.
