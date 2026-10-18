# Troubleshooting: Migration Job Failure (BackoffLimitExceeded)

## 1. Issue overview
A Kubernetes `Job` is a controller designed for batch workloads that run until completion. When a database migration Job fails repeatedly, the Job controller exhausts its retry budget (`backoffLimit`) and marks the Job condition `Failed` with reason `BackoffLimitExceeded`.

In Kubernetes:
- When a Job container exits with a non-zero exit code and `restartPolicy: Never` is set, the Job controller creates a new Pod to retry the execution.
- Retries are governed by `spec.backoffLimit` (default is 6).
- If the number of failed retries reaches `backoffLimit`, the controller ceases retrying, sets `Failed: True`, and leaves the failed Pods intact so operators can inspect their logs.

**Expected behavior:** The migration Job runs once, applies database schema changes, exits with code 0, and reports `COMPLETIONS: 1/1`.  
**Actual behavior:** The migration script crashes with a SQL syntax error, fails 2 times, and halts in `BackoffLimitExceeded`.

---

## 2. Common causes
1. **Non-idempotent migration scripts:** Script attempts to create a table that already exists without `IF NOT EXISTS`.
2. **Missing database credentials or environment variables:** Script cannot authenticate to database.
3. **Database connection timeout:** Migration Job starts before database server is accepting connections.
4. **Insufficient Job memory/CPU:** Schema migration process OOMKilled when processing large tables.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\storage-database\08-migration-job-failure"
```

Apply the invalid Job:

```powershell
kubectl apply -f .\invalid-job.yaml
```

Monitor the Job status over 25 seconds:

```powershell
kubectl get jobs -l troubleshooting=storage
```

Observe that the Job fails after exhausting retries:
```text
NAME               COMPLETIONS   DURATION   AGE
job-db-migration   0/1           20s        25s
```

Check the Job Pods:

```powershell
kubectl get pods -l app=db-migration
```

Output:
```text
NAME                     READY   STATUS   RESTARTS   AGE
job-db-migration-5k8qm   0/1     Error    0          25s
job-db-migration-9j2xl   0/1     Error    0          15s
job-db-migration-m4pxz   0/1     Error    0          5s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect Job conditions and events
```powershell
kubectl describe job job-db-migration
```

Examine the **Conditions** block:
```text
Conditions:
  Type     Status  Reason
  ----     ------  ------
  Failed   True    BackoffLimitExceeded
```

Look at the **Events** section:
```text
Events:
  Type     Reason                Age   From            Message
  ----     ------                ----  ----            -------
  Normal   SuccessfulCreate      30s   job-controller  Created pod: job-db-migration-5k8qm
  Normal   SuccessfulCreate      20s   job-controller  Created pod: job-db-migration-9j2xl
  Normal   SuccessfulCreate      10s   job-controller  Created pod: job-db-migration-m4pxz
  Warning  BackoffLimitExceeded  5s    job-controller  Job has reached the specified backoff limit
```

### Step 2: Read the migration failure logs from a failed Pod
```powershell
$failedPod = (kubectl get pods -l app=db-migration -o jsonpath='{.items[0].metadata.name}')
kubectl logs $failedPod
```

Output:
```text
Executing database migration V1.2.0...
SQL Syntax Error: Table 'user_profiles' already exists
```

**Diagnostic analysis:**
- The Job failed because the migration script encountered an unhandled SQL error: `Table 'user_profiles' already exists`.
- Because the script exited with code 1, the Job controller retried until reaching `backoffLimit: 2`.
- Once the limit was reached, the Job was permanently marked `BackoffLimitExceeded`.

---

## 6. Root cause
In `invalid-job.yaml`, the migration script is not idempotent and fails when `user_profiles` already exists, exiting with code 1.

---

## 7. Fix the issue
Because Job specifications are immutable after creation, delete the failed Job and apply the fixed manifest:

```powershell
kubectl delete -f .\invalid-job.yaml
kubectl apply -f .\fixed-job.yaml
```

The corrected manifest `fixed-job.yaml` ensures the script is idempotent (`CREATE TABLE IF NOT EXISTS`) and exits cleanly with code 0.

---

## 8. Verify the fix
Wait for the Job to complete:

```powershell
kubectl wait --for=condition=complete job/job-db-migration --timeout=60s
```

Check the Job status:

```powershell
kubectl get jobs job-db-migration
```

Expected output:
```text
NAME               COMPLETIONS   DURATION   AGE
job-db-migration   1/1           4s         15s
```

Inspect the successful Job logs:

```powershell
$successPod = (kubectl get pods -l app=db-migration -o jsonpath='{.items[0].metadata.name}')
kubectl logs $successPod
```

Expected output:
```text
Executing database migration V1.2.0...
CREATE TABLE IF NOT EXISTS user_profiles (id INT PRIMARY KEY);
Database migration completed successfully.
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-job-failed-status.png` — Output of `kubectl get jobs` showing `0/1 completions`.
- [ ] `02-backofflimit-exceeded-describe.png` — Output of `kubectl describe job` showing `BackoffLimitExceeded`.
- [ ] `03-sql-error-logs.png` — Output of `kubectl logs` showing the SQL syntax error.
- [ ] `04-verified-job-complete.png` — Output of `kubectl get jobs` showing `COMPLETIONS: 1/1`.

---

## 10. Cleanup
Delete the Job:

```powershell
kubectl delete -f .\fixed-job.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get job job-db-migration
```

---

## 11. Key learning
- **Root cause:** The migration script failed due to an unhandled SQL error, exhausting the Job's `backoffLimit`.
- **Most useful diagnostic command:** `kubectl describe job <name>` to check `BackoffLimitExceeded`, followed by `kubectl logs` on one of the failed Job Pods.
- **Correction:** Make migration scripts idempotent (`IF NOT EXISTS`) and ensure clean zero exit codes on completion.
- **Production consideration:** Set `ttlSecondsAfterFinished` on Jobs to automatically clean up finished Job Pods and avoid cluttering namespace resources.
