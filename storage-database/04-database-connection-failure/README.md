# Troubleshooting: Database Connection Failure

## 1. Issue overview
A database connection failure occurs when an application Pod crashes during startup because it cannot resolve the database hostname, reach the database port, or establish a network handshake with the storage tier.

In Kubernetes:
- Applications frequently connect to databases via internal Service DNS names (e.g. `mysql-service.default.svc.cluster.local:3306`).
- If the database service name is misspelled, the database Pod is crashed, or the hostname points to an unresolvable external domain (`mysql-db.invalid.internal`), the TCP connection fails.
- If the application lacks connection retry logic or circuit breakers, it aborts immediately and enters `CrashLoopBackOff`.

**Expected behavior:** The application connects to the database endpoint and starts serving traffic.  
**Actual behavior:** The application attempts to connect to `mysql-db.invalid.internal:3306`, fails, logs a fatal error, and crashes with exit code 1.

---

## 2. Common causes
1. **Unresolvable database hostname:** Database host URL is incorrect or points to a non-existent domain.
2. **Database Service or Pod is offline:** MySQL/PostgreSQL Pod is crashed or Service has no endpoints.
3. **Database port not exposed:** Service port is 3306, but database listens on custom port.
4. **NetworkPolicy blocking egress:** Application namespace lacks egress permissions to database namespace.
5. **Lack of connection retry logic:** Application crashes immediately on the first failed TCP handshake rather than backing off and retrying.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\04-database-connection-failure"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods -l app=db-conn-demo
```

Observe `CrashLoopBackOff`:
```text
NAME                                      READY   STATUS             RESTARTS      AGE
deploy-db-conn-failure-7f849bc5df-5k2ml   0/1     CrashLoopBackOff   2 (10s ago)   30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect container logs
```powershell
$pod = (kubectl get pods -l app=db-conn-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod --previous
```

Output:
```text
Connecting to primary database at mysql-db.invalid.internal:3306...
nc: bad address 'mysql-db.invalid.internal'
FATAL: Database connection timed out or host unreachable!
```

### Step 2: Test DNS resolution and connectivity from an ephemeral debug Pod
```powershell
kubectl run db-dns-check --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- nslookup mysql-db.invalid.internal
```

Output:
```text
** server can't find mysql-db.invalid.internal: NXDOMAIN
```

**Diagnostic analysis:**
- The application crashes with exit code 1 because `mysql-db.invalid.internal` returns `NXDOMAIN`.
- The application has a hard dependency on an unresolvable database host.

---

## 6. Root cause
In `invalid-deployment.yaml`, the application attempts a synchronous network check against `mysql-db.invalid.internal:3306`. Because this domain does not exist, the connection check fails and terminates the process.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` configures the application to start gracefully without crashing on unresolvable external hosts.

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-db-conn-failure --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=db-conn-demo
```

Expected output:
```text
NAME                                      READY   STATUS    RESTARTS   AGE
deploy-db-conn-failure-5cf7b7d7b5-2k9mp   1/1     Running   0          25s
```

Verify the container logs:

```powershell
$newPod = (kubectl get pods -l app=db-conn-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $newPod
```

Expected output:
```text
Connecting to database service...
Database connected in mock/standalone mode.
Application running...
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloop-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-fatal-db-connection-logs.png` — Output of `kubectl logs --previous` showing `FATAL: Database connection timed out`.
- [ ] `03-nslookup-nxdomain.png` — Output of `nslookup mysql-db.invalid.internal` showing `NXDOMAIN`.
- [ ] `04-verified-running-app.png` — Output showing healthy `Running` status and application running logs.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-db-conn-failure
```

---

## 11. Key learning
- **Root cause:** The application crashed immediately on startup because its configured database hostname was unresolvable.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to identify failed database endpoints.
- **Correction:** Configure valid database endpoints and implement exponential connection retry loops.
- **Production consideration:** Applications must implement resilient database connection retries (e.g. Polly, tenacity) rather than crashing on transient connection drops or temporary database cold starts.
