# Troubleshooting: ConfigMap Misconfiguration (Missing Key Reference)

## 1. Issue overview
A ConfigMap misconfiguration occurs when a workload references a ConfigMap key that does not exist in the target ConfigMap data map, causing the Kubelet to halt container creation with `CreateContainerConfigError`.

In Kubernetes:
- When a container uses `valueFrom.configMapKeyRef`, the Kubelet must resolve the specific `key` from the ConfigMap's `data` block before launching the container.
- If the ConfigMap exists, but does not contain the requested key name (e.g. ConfigMap has `DATABASE_URL` but Pod requests `DB_HOST`), Kubelet cannot resolve the environment variable.
- Unless `optional: true` is explicitly configured, the container remains in the `Waiting` state with reason `CreateContainerConfigError`.

**Expected behavior:** Kubelet retrieves the key `DB_HOST` from `app-config`, injects it as an environment variable, and starts the container.  
**Actual behavior:** ConfigMap `app-config` defines key `DATABASE_URL` instead of `DB_HOST`, causing Kubelet to block container startup.

---

## 2. Common causes
1. **Case-sensitivity or typo in key name:** Using `DB_HOST` in Pod spec while ConfigMap defines `db_host` or `DATABASE_URL`.
2. **Missing data entry in ConfigMap:** ConfigMap was updated, but a required environment variable key was accidentally omitted.
3. **Referencing wrong ConfigMap name:** Pod spec references an outdated or staging ConfigMap.
4. **Omitting `optional: true`:** Non-critical configuration keys halt container startup when missing.
5. **Namespace mismatch:** ConfigMap deployed to a different namespace than the Deployment.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\01-configmap-misconfiguration"
```

Apply the ConfigMap and Deployment:

```powershell
kubectl apply -f .\invalid-configmap.yaml
kubectl apply -f .\invalid-deployment.yaml
```

Check the Pod status after 10 seconds:

```powershell
kubectl get pods -l app=configmap-demo
```

Observe `CreateContainerConfigError`:
```text
NAME                                      READY   STATUS                       RESTARTS   AGE
deploy-configmap-error-746fb98c4d-6x8pq   0/1     CreateContainerConfigError   0          20s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod events and container error message
```powershell
$pod = (kubectl get pods -l app=configmap-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod
```

Examine the **Containers** status block:
```text
State:          Waiting
  Reason:       CreateContainerConfigError
  Message:      configmap "app-config" key "DB_HOST" not found
```

### Step 2: Compare Pod reference with actual ConfigMap data
Extract the required key from the Pod spec:

```powershell
kubectl get pod $pod -o jsonpath='{.spec.containers[0].env[?(@.name=="DB_HOST")].valueFrom.configMapKeyRef}'
```

Output:
```text
{"key":"DB_HOST","name":"app-config"}
```

Now inspect the actual keys present in ConfigMap `app-config`:

```powershell
kubectl get configmap app-config -o yaml
```

Output:
```text
data:
  DATABASE_URL: postgres://db.internal:5432/production
```

**Diagnostic analysis:**
- The ConfigMap `app-config` exists.
- The Pod requires key `DB_HOST`.
- The ConfigMap only contains key `DATABASE_URL`.
- Because the key is missing and `optional: true` is not set, Kubelet refuses to start the container.

---

## 6. Root cause
In `invalid-configmap.yaml`, the data key is named `DATABASE_URL`, whereas `invalid-deployment.yaml` references key `DB_HOST`.

---

## 7. Fix the issue
Apply the fixed ConfigMap and Deployment manifests:

```powershell
kubectl apply -f .\fixed-configmap.yaml
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-configmap.yaml` aligns the data key to `DB_HOST`.

---

## 8. Verify the fix
Wait for the Pod to become Ready:

```powershell
kubectl rollout status deployment/deploy-configmap-error --timeout=60s
```

Verify Pod status:

```powershell
kubectl get pods -l app=configmap-demo
```

Expected output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
deploy-configmap-error-5cf7b7d7b5-2k9mp   1/1     Running   0          25s
```

Verify that the environment variable is populated inside the container:

```powershell
$pod = (kubectl get pods -l app=configmap-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod
```

Expected output:
```text
Database host is postgres://db.internal:5432/production
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-create-container-config-error.png` — Output of `kubectl get pods` showing `CreateContainerConfigError`.
- [ ] `02-describe-pod-key-not-found.png` — Output of `kubectl describe pod` showing `key "DB_HOST" not found`.
- [ ] `03-configmap-keys-mismatch.png` — Output of `kubectl get configmap app-config -o yaml` showing `DATABASE_URL`.
- [ ] `04-verified-running-with-logs.png` — Output of `kubectl logs` verifying `Database host is postgres:...`.

---

## 10. Cleanup
Delete the Deployment and ConfigMap:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
kubectl delete -f .\fixed-configmap.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-configmap-error
kubectl get configmap app-config
```

---

## 11. Key learning
- **Root cause:** The Deployment referenced key `DB_HOST`, but the ConfigMap defined key `DATABASE_URL`.
- **Most useful diagnostic command:** `kubectl describe pod <name>` to view the exact missing key name in the container message.
- **Correction:** Align key names between ConfigMap `data` and Pod `configMapKeyRef.key`.
- **Production consideration:** Use CI schema validation or Helm/Kustomize templates to generate ConfigMaps and Pod env references from a single source of truth, preventing manual key divergence.
