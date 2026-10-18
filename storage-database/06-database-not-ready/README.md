# Troubleshooting: Database Not Ready (Race Condition at Startup)

## 1. Issue overview
A "Database Not Ready" issue occurs when an application container starts up before its backend database has completed schema migrations or opened its listening socket, causing the application to crash immediately upon boot.

In Kubernetes:
- Pods and Deployments start concurrently; Kubernetes does not automatically enforce service startup dependencies across different Deployments.
- If an application lacks built-in startup retry loops and expects the database to be operational from the first millisecond, it exits with return code 1.
- **Recommended Architectural Pattern:** Use `initContainers` to wait for database port availability or readiness checks before the main application container is launched.

**Expected behavior:** The application waits for database readiness before executing main business logic.  
**Actual behavior:** The application checks for database readiness, encounters an unready state, and immediately crashes into `CrashLoopBackOff`.

---

## 2. Common causes
1. **Concurrent startup race conditions:** Web application starts in 2 seconds while the MySQL/PostgreSQL container takes 20 seconds to run entrypoint scripts.
2. **Missing retry loops in application code:** Code throws an unhandled exception on the first failed query instead of retrying.
3. **Database in recovery / replay mode:** Database is replaying WAL logs after a restart and refusing client connections.
4. **Lack of initContainers:** Application Pod lacks a pre-flight readiness gate.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\06-database-not-ready"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods -l app=db-not-ready-demo
```

Observe `CrashLoopBackOff`:
```text
NAME                                   READY   STATUS             RESTARTS      AGE
deploy-db-not-ready-75f84bc6d4-8j5kz   0/1     CrashLoopBackOff   2 (10s ago)   30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Read previous container logs
```powershell
$pod = (kubectl get pods -l app=db-not-ready-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod --previous
```

Output:
```text
Starting application...
Querying database...
FATAL: Database is not ready to accept connections!
```

**Diagnostic analysis:**
- The container exits with code 1.
- The log explicitly reports: `FATAL: Database is not ready to accept connections!`.
- The application started before the database was ready.

---

## 6. Root cause
`invalid-deployment.yaml` checks for a readiness marker and immediately terminates with `exit 1` without waiting or retrying.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` adds an `initContainer` (`wait-for-db`) that blocks the main application container from launching until the database readiness prerequisite is verified.

---

## 8. Verify the fix
Monitor the initialization phase:

```powershell
kubectl get pods -l app=db-not-ready-demo
```

Notice the init container phase:
```text
NAME                                   READY   STATUS     RESTARTS   AGE
deploy-db-not-ready-5bf78cd5b4-9p2xl   0/1     Init:0/1   0          5s
```

Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-db-not-ready --timeout=60s
```

Verify Pod status:

```powershell
kubectl get pods -l app=db-not-ready-demo
```

Expected output:
```text
NAME                                   READY   STATUS    RESTARTS   AGE
deploy-db-not-ready-5bf78cd5b4-9p2xl   1/1     Running   0          25s
```

Check the init container logs:

```powershell
$newPod = (kubectl get pods -l app=db-not-ready-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod -c wait-for-db
```

Output:
```text
Waiting for database readiness...
Database is ready.
```

Check the main container logs:

```powershell
kubectl logs $newPod -c app
```

Output:
```text
Starting application...
Querying database...
Application running.
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloop-not-ready.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-fatal-not-ready-logs.png` — Output of `kubectl logs --previous` showing `FATAL: Database is not ready`.
- [ ] `03-init-container-running.png` — Output showing `STATUS: Init:0/1`.
- [ ] `04-verified-ready-application.png` — Output showing `Application running` with 0 restarts.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-db-not-ready
```

---

## 11. Key learning
- **Root cause:** The application crashed because it lacked startup retry logic when its database dependency was not yet ready.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to identify startup race conditions.
- **Correction:** Implement `initContainers` to wait for dependent services or add robust connection retries inside application code.
- **Production consideration:** Use lightweight init containers with `nc -z -w 3 <db-service> <port>` or custom health probes to ensure orderly service startup order in microservice deployments.
