# Troubleshooting: Secret Configuration Error

## 1. Issue overview
A Secret configuration error occurs when a workload attempts to reference an external `Secret` or a specific key within a Secret that cannot be resolved by the Kubelet during container construction.

In Kubernetes:
- Kubernetes Secrets store sensitive data (e.g. tokens, certificates, passwords) as base64-encoded key-value pairs.
- **Important Security Principle:** Base64 encoding is an encoding scheme, **NOT encryption**. Any user with read access to the Secret in the namespace can decode the raw bytes with `[System.Convert]::FromBase64String`.
- If a Pod references a Secret key that does not exist (e.g. Secret has key `api-key` but Pod requests `api_token`), Kubelet halts container creation with `CreateContainerConfigError`.

**Expected behavior:** Kubelet resolves `api_token` from Secret `app-secrets`, decodes the value, and injects it into the container environment.  
**Actual behavior:** Secret `app-secrets` defines key `api-key` instead of `api_token`, locking the Pod into `CreateContainerConfigError`.

---

## 2. Common causes
1. **Mismatched Secret key name:** Typo or separator discrepancy (`api-key` vs `api_token`).
2. **Missing Secret object:** The Secret was never created in the cluster before deploying the application.
3. **Namespace mismatch:** The Secret was applied in namespace `kube-system` or `prod` while the Pod runs in `default`.
4. **Invalid base64 payload:** Malformed base64 strings in the Secret data block.
5. **RBAC restrictions:** The ServiceAccount used by the Pod or cluster operator lacks permissions to read Secrets.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\02-secret-configuration-error"
```

Apply the dummy Secret and Deployment:

```powershell
kubectl apply -f .\invalid-secret-example.yaml
kubectl apply -f .\invalid-deployment.yaml
```

Check the Pod status:

```powershell
kubectl get pods -l app=secret-demo
```

Observe `CreateContainerConfigError`:
```text
NAME                                   READY   STATUS                       RESTARTS   AGE
deploy-secret-error-6bf46c6499-9m2kx   0/1     CreateContainerConfigError   0          20s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Pod status and container error message
```powershell
$pod = (kubectl get pods -l app=secret-demo -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $pod
```

Examine the **Containers** status block:
```text
State:          Waiting
  Reason:       CreateContainerConfigError
  Message:      secret "app-secrets" key "api_token" not found
```

### Step 2: Inspect the keys present in Secret `app-secrets`
List the data keys inside the Secret:

```powershell
kubectl get secret app-secrets -o jsonpath='{.data}'
```

Output:
```text
{"api-key":"ZGVtby1zZWNyZXQta2V5LTEyMzQ1"}
```

Notice:
- The Secret defines key `api-key`.
- The Deployment demands key `api_token`.

### Step 3: Demonstrate base64 decoding vs encryption
Explain how base64 is decoded in PowerShell:

```powershell
$encoded = (kubectl get secret app-secrets -o jsonpath='{.data.api-key}')
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

Output:
```text
demo-secret-key-12345
```
*Note: This proves that base64 is plaintext obfuscation, not cryptographic encryption.*

---

## 6. Root cause
In `invalid-secret-example.yaml`, the key is named `api-key`, whereas `invalid-deployment.yaml` references `key: api_token`.

---

## 7. Fix the issue
Apply the fixed Secret manifest:

```powershell
kubectl apply -f .\fixed-secret-example.yaml
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-secret-example.yaml` aligns the Secret key to `api_token`.

---

## 8. Verify the fix
Wait for the Deployment rollout to complete:

```powershell
kubectl rollout status deployment/deploy-secret-error --timeout=60s
```

Check Pod status:

```powershell
kubectl get pods -l app=secret-demo
```

Expected output:
```text
NAME                                   READY   STATUS    RESTARTS   AGE
deploy-secret-error-5f49cb87d4-8j5kz   1/1     Running   0          25s
```

Inspect container logs to confirm successful secret loading:

```powershell
$pod = (kubectl get pods -l app=secret-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod
```

Expected output:
```text
Application loaded secret: demo-secret-key-12345
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-secret-config-error-status.png` — Output of `kubectl get pods` showing `CreateContainerConfigError`.
- [ ] `02-describe-pod-secret-key-notfound.png` — Output of `kubectl describe pod` showing `secret "app-secrets" key "api_token" not found`.
- [ ] `03-secret-keys-inspection.png` — Output showing Secret data keys `{"api-key": ...}`.
- [ ] `04-verified-running-secret.png` — Output of `kubectl logs` verifying `Application loaded secret: demo-secret-key-12345`.

---

## 10. Cleanup
Delete the test resources:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
kubectl delete -f .\fixed-secret-example.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-secret-error
kubectl get secret app-secrets
```

---

## 11. Key learning
- **Root cause:** The Deployment requested a Secret key (`api_token`) that differed from the key in the Secret (`api-key`).
- **Most useful diagnostic command:** `kubectl describe pod <name>` to read the exact missing Secret key name.
- **Correction:** Align key names between Secret `data` and Pod `secretKeyRef.key`.
- **Production consideration:** Enable Kubernetes Encryption at Rest for etcd and use external secret managers (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) with Secrets Store CSI Driver.
