# Troubleshooting: Dependency / Version Incompatibility

## 1. Issue overview
A dependency or version incompatibility occurs when a Kubernetes workload configuration specifies command-line arguments, environment parameters, or operational flags that are unsupported, deprecated, or removed in the container image's runtime version.

In Kubernetes:
- Upgrading container image versions (e.g. from version `1.x` to `2.x`) often introduces breaking CLI changes, removed subcommands, or stricter configuration parsing.
- If an existing Deployment manifest is retained with legacy flags while the image tag is updated, the new binary fails to parse the arguments and exits with an error.
- The Pod enters `CrashLoopBackOff`.

**Expected behavior:** The application binary parses valid arguments for its version and starts serving traffic.  
**Actual behavior:** The `busybox:1.36` binary `httpd` is invoked with an unsupported legacy flag `--enable-ssl-v3`, returning `httpd: unrecognized option '--enable-ssl-v3'` and exiting with code 1.

---

## 2. Common causes
1. **Upgrading image version without auditing CLI flags:** New application major release removes legacy flags.
2. **Switching base images:** Migrating from Debian-based images (with GNU coreutils) to Alpine-based images (with BusyBox/Musl) where flag syntax differs.
3. **Deprecated configuration directives:** Software removes legacy SSL/TLS ciphers or protocol flags.
4. **Library or dependency ABI mismatch:** Container image contains shared libraries incompatible with the compiled binary.

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\configuration-application\07-dependency-version-incompatibility"
```

Apply the invalid Deployment:

```powershell
kubectl apply -f .\invalid-deployment.yaml
```

Monitor the Pod status:

```powershell
kubectl get pods -l app=version-demo
```

Observe `CrashLoopBackOff`:
```text
NAME                                              READY   STATUS             RESTARTS      AGE
deploy-version-incompatibility-7c98b64b85-5j2mx   0/1     CrashLoopBackOff   2 (12s ago)   30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Read the container failure logs
```powershell
$pod = (kubectl get pods -l app=version-demo -o jsonpath='{.items[0].metadata.name}')
kubectl logs $pod --previous
```

Output:
```text
httpd: unrecognized option '--enable-ssl-v3'
BusyBox v1.36.1 (2023-05-19 14:11:54 UTC) multi-call binary.

Usage: httpd [-ifv] [-c CONFFILE] [-p [IP:]PORT] [-u USER[:GRP]] [-r REALM] [-h HOME]
```

### Step 2: Analyze the version and supported options
The output shows:
- The binary version is **BusyBox v1.36.1**.
- The binary explicitly lists its supported options: `[-ifv] [-c CONFFILE] [-p [IP:]PORT] [-u USER[:GRP]] [-r REALM] [-h HOME]`.
- `--enable-ssl-v3` is an unrecognized option.

---

## 6. Root cause
In `invalid-deployment.yaml`, line 21 defines:
```yaml
command: ["httpd", "-f", "--enable-ssl-v3"]
```
The binary `httpd` inside `busybox:1.36` does not support `--enable-ssl-v3`, causing the argument parser to abort with exit code 1.

---

## 7. Fix the issue
Apply the fixed Deployment manifest:

```powershell
kubectl apply -f .\fixed-deployment.yaml
```

The corrected manifest `fixed-deployment.yaml` uses valid flags supported by the current version:
```yaml
command: ["httpd", "-f", "-p", "8080"]
```

---

## 8. Verify the fix
Wait for the rollout to complete:

```powershell
kubectl rollout status deployment/deploy-version-incompatibility --timeout=60s
```

Check the Pod status:

```powershell
kubectl get pods -l app=version-demo
```

Expected output:
```text
NAME                                              READY   STATUS    RESTARTS   AGE
deploy-version-incompatibility-58b4b967dc-k9m2x   1/1     Running   0          25s
```

Verify that the process is running and listening on port 8080:

```powershell
$newPod = (kubectl get pods -l app=version-demo -o jsonpath='{.items[0].metadata.name}')
kubectl exec $newPod -- netstat -tuln
```

Expected output:
```text
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-crashloop-status.png` — Output of `kubectl get pods` showing `CrashLoopBackOff`.
- [ ] `02-unrecognized-option-logs.png` — Output of `kubectl logs` showing `httpd: unrecognized option '--enable-ssl-v3'`.
- [ ] `03-usage-options-output.png` — Output showing binary usage and supported flags.
- [ ] `04-verified-running-httpd.png` — Output of `kubectl get pods` and `netstat -tuln` showing healthy execution on port 8080.

---

## 10. Cleanup
Delete the Deployment:

```powershell
kubectl delete -f .\fixed-deployment.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get deployment deploy-version-incompatibility
```

---

## 11. Key learning
- **Root cause:** The Deployment passed an unsupported flag (`--enable-ssl-v3`) incompatible with the installed binary version.
- **Most useful diagnostic command:** `kubectl logs <pod-name> --previous` to inspect the binary's command usage output.
- **Correction:** Align configuration arguments with the supported CLI options of the container image version.
- **Production consideration:** Whenever bumping container image tags, review release notes and changelogs for deprecated or removed CLI flags and configuration directives.
