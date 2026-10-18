# Troubleshooting: Invalid Database Credentials

## 1. Issue overview
An invalid database credentials issue occurs when an application successfully reaches the database service, but its authentication handshake is rejected due to mismatched passwords, rotated credentials, or malformed secret payloads.

In Kubernetes:
- Applications typically inject database passwords from Kubernetes `Secret` objects.
- When database passwords are rotated or updated in the database server, but the Kubernetes Secret is not updated, subsequent connection attempts fail with authentication errors (e.g. MySQL `ERROR 1045: Access denied`).
- If an application aborts execution upon authentication failure, it enters `CrashLoopBackOff`.

**Expected behavior:** The application authenticates with the correct password and opens a database connection session.  
**Actual behavior:** The Secret contains an invalid password (`wrong-password-999`), causing the authentication check to fail with `Access denied` and exit code 1.

---

## 2. Common causes
1. **Password rotation mismatch:** Database password changed, but Kubernetes Secret was not updated.
2. **Whitespace or newline in base64 encoding:** Running `echo "password"` (which includes a trailing `
`) instead of `echo -n "password"` before base64 encoding.
3. **Mismatched database username:** Application connects as `root` instead of the allocated application user.
4. **Host authorization restriction (`@'%'` vs `@'localhost'`):** Database grants user access only from localhost, rejecting connections from the Kubernetes Pod subnet.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\05-invalid-database-credentials"
```

Apply the invalid Secret and Deployment:

```powershell
kubectl apply -f .\invalid-secret-example.yaml
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods -l app=db-auth-demo
```

Observe `CrashLoopBackOff`:
```text
NAME                                      READY   STATUS             RESTARTS      AGE
deploy-db-auth-failure-6cf49b87d4-8j5kz   0/1     CrashLoopBackOff   2 (10s ago)   30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Read previous container logs
```powershell
$pod = (kubectl get pods -l app=db-auth-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod --previous
```

Output:
```text
Authenticating against database...
ERROR 1045 (28000): Access denied for user 'app_user'@'%' (using password: YES)
```

### Step 2: Inspect the Secret's decoded value in PowerShell
Verify the current value stored in the Secret:

```powershell
$encoded = (kubectl get secret db-credentials -o jsonpath='{.data.DB_PASSWORD}')
$decoded = [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
Write-Host "Decoded password: $decoded"
```

Output:
```text
Decoded password: wrong-password-999
```
*Notice that the decoded password is `wrong-password-999`, which is rejected by the database.*

---

## 6. Root cause
`invalid-secret-example.yaml` contains `wrong-password-999`. The database rejects the credential during the handshake, triggering an authentication exit.

---

## 7. Fix the issue
Apply the fixed Secret manifest:

```powershell
kubectl apply -f .\fixed-secret-example.yaml
kubectl apply -f .\fixed-deployment.yaml
```

Restart the Deployment to pick up the updated Secret:

```powershell
kubectl rollout restart deployment/deploy-db-auth-failure
```

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-db-auth-failure --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=db-auth-demo
```

Expected output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
deploy-db-auth-failure-58f79bc94-4d8zk    1/1     Running   0          20s
```

Verify the container logs:

```powershell
$newPod = (kubectl get pods -l app=db-auth-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod
```

Expected output:
```text
Authenticating against database...
Authentication successful. Database session established.
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-auth-failure-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-access-denied-logs.png` — Output of `kubectl logs --previous` showing `ERROR 1045: Access denied`.
- [ ] `03-decoded-secret-inspection.png` — PowerShell output showing decoded `wrong-password-999`.
- [ ] `04-verified-session-established.png` — Output of `kubectl logs` verifying `Database session established`.

---

## 10. Cleanup
Delete all test resources:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
kubectl delete -f .\fixed-secret-example.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-db-auth-failure
kubectl get secret db-credentials
```

---

## 11. Key learning
- **Root cause:** Secret contained an invalid password, prompting the database to reject the connection with `Access denied`.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to identify specific authentication error codes (like `ERROR 1045`).
- **Correction:** Update the Kubernetes Secret with valid credentials and restart the consuming Deployment.
- **Production consideration:** Automate credential rotation using operators (e.g. AWS Secrets Manager CSI, Vault Agent) that dynamically update secrets and signal application pods to re-authenticate without downtime.
