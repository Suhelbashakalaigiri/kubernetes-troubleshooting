# Standard Kubernetes Troubleshooting Workflow

A reusable, systematic 12-stage troubleshooting procedure for investigating, diagnosing, and resolving Kubernetes issues in a structured, repeatable manner.

For the project overview, see [README.md](README.md).  
For workstation prerequisites and health checks, see [environment-setup.md](environment-setup.md).

---

## 1. Understand the Problem

Before executing commands or modifying files, review the issue runbook:

1. **Read the Issue Runbook:** Open the `README.md` file in the target issue directory.
2. **Identify the Affected Resource:** Determine the primary API object involved (e.g. Pod, Deployment, Service, PVC, Ingress, Job, NetworkPolicy).
3. **Establish Expected vs. Actual Behavior:** Clarify what a healthy workload should do compared to the failure symptom being demonstrated.
4. **Identify Dependencies and Prerequisites:** Check if the issue relies on specific namespaces, ConfigMaps, Secrets, or StorageClasses.

---

## 2. Check the Environment

Ensure that your cluster is stable before introducing failure scenarios. Refer to [environment-setup.md](environment-setup.md) if any check fails:

```powershell
# Verify cluster status
minikube status

# Confirm the active kubectl context is minikube
kubectl config current-context

# Verify that the cluster node is Ready
kubectl get nodes
```

> [!WARNING]
> Never begin a troubleshooting exercise if the cluster node is in `NotReady` or control-plane components are degraded, as external cluster issues will invalidate scenario diagnostics.

---

## 3. Reproduce the Issue

Navigate to the target issue directory in Windows PowerShell:

```powershell
# Example: Navigating to an issue directory
Set-Location -Path ".\pod\01-image-pull-backoff"

# List directory contents
Get-ChildItem
```

### Execution Rules:
- **Inspect Before Applying:** Examine `invalid-*.yaml` to understand what resource is being created.
- **Apply Only the Invalid Manifest:** Run:
  ```powershell
  kubectl apply -f .\invalid-pod.yaml
  ```
- **Allow Time for Controllers:** Some failures appear immediately (e.g. invalid volume references rejected by admission), while others take 15 to 45 seconds to manifest as Kubelet backoffs or probe failure thresholds accumulate.
- **Never Apply Both Simultaneously:** Do not apply `fixed-*.yaml` at the same time as `invalid-*.yaml`.

---

## 4. Collect Evidence

Gather empirical evidence from the cluster using non-destructive inspection commands:

### Core Workload Commands

```powershell
# 1. View high-level status, node assignment, and IP
kubectl get pods -o wide

# 2. Inspect cluster-wide events sorted by timestamp
kubectl get events --sort-by=.metadata.creationTimestamp

# 3. View comprehensive object status, conditions, and event stream
kubectl describe pod <pod-name>

# 4. View stdout/stderr logs from the current container process
kubectl logs <pod-name>

# 5. View logs from a previous, crashed container instance
kubectl logs <pod-name> --previous
```

### Resource-Specific Evidence Commands

| Resource Type | Recommended Diagnostic Commands |
| :--- | :--- |
| **Deployments** | `kubectl rollout status deployment/<name>`<br>`kubectl get rs -l <label>`<br>`kubectl describe deployment <name>` |
| **Services** | `kubectl get endpoints <service-name>`<br>`kubectl get endpointslices -l kubernetes.io/service-name=<name>`<br>`kubectl describe service <name>` |
| **Storage / PVCs** | `kubectl get pvc`<br>`kubectl describe pvc <claim-name>`<br>`kubectl get storageclass` |
| **Jobs** | `kubectl describe job <job-name>`<br>`kubectl get pods -l job-name=<job-name>` |
| **NetworkPolicies** | `kubectl describe networkpolicy <policy-name>` |
| **HPA** | `kubectl get hpa`<br>`kubectl describe hpa <hpa-name>` |
| **RBAC** | `kubectl auth can-i <verb> <resource> --as=<service-account>`<br>`kubectl describe role <role-name>` |

---

## 5. Diagnose the Root Cause

Systematically isolate symptoms from underlying causes.

### Symptom vs. Root Cause Analysis

An observed status is merely a high-level symptom reported by Kubelet or a controller:

| Observed Symptom | Why It Occurs | Underlying Root Causes |
| :--- | :--- | :--- |
| **`ImagePullBackOff`** | Kubelet paused after repeated image pull failures. | Typo in image tag, private registry authentication missing, or network DNS failure (`ErrImagePull`). |
| **`CrashLoopBackOff`** | Container process starts, exits with non-zero code, and Kubelet backs off restarting. | Missing configuration variable, uncaught code exception, unmapped port collision, or CLI argument syntax error. |
| **`Pending`** | Pod accepted by API server, but cannot run. | Unsatisfiable `nodeSelector`, unfulfilled PVC, untolerated node taint, or excessive resource requests (`FailedScheduling`). |
| **`OOMKilled`** | Linux kernel terminated process with SIGKILL (Exit code 137). | Container memory consumption exceeded `resources.limits.memory` or node experienced memory pressure. |
| **`ContainerCreating`** | Kubelet cannot initialize container sandbox. | Volume mount dependency missing (`FailedMount`), CNI IP allocation failure, or image layer unpacking delay. |

### Diagnostic Principles:
1. **Never Assume an Error String Tells the Full Story:** A `CrashLoopBackOff` status tells you *that* the container crashed; only `kubectl logs <pod> --previous` or `kubectl describe pod` reveals *why*.
2. **Inspect the Object's Conditions Block:** Look at `.status.conditions` in `kubectl describe` (`Initialized`, `Ready`, `ContainersReady`, `PodScheduled`).
3. **Use JSONPath for Precise Field Extraction:**
   ```powershell
   # Extract the exact waiting message
   kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].state.waiting.message}'
   ```

---

## 6. Apply the Fix

Once the root cause has been established:

1. **Review the Fixed Manifest:** Inspect `fixed-*.yaml` to understand the configuration correction.
2. **Understand Mutability Rules:**
   - **Mutable Fields:** Deployments, Services, ConfigMaps, and Secrets can typically be updated in-place with:
     ```powershell
     kubectl apply -f .\fixed-deployment.yaml
     ```
   - **Immutable Fields:** Standalone Pod specifications (e.g. `nodeSelector`, `containers[*].image` on running pods) and PVC `storageClassName` cannot be edited in-place. Delete the invalid resource first:
     ```powershell
     kubectl delete -f .\invalid-pod.yaml
     kubectl apply -f .\fixed-pod.yaml
     ```
3. **Apply Only the Intended Fix:** Do not make extraneous changes to unrelated cluster objects.

---

## 7. Verify Recovery

Do not assume an issue is resolved simply because `kubectl apply` completed without syntax errors. Prove resolution through multiple layers of verification:

### Verification Layer 1: Resource Status
```powershell
# For Deployments: Confirm rollout completion
kubectl rollout status deployment/<deployment-name> --timeout=60s

# For Pods: Wait for the Ready condition
kubectl wait --for=condition=Ready pod/<pod-name> --timeout=60s

# Check wide output
kubectl get pods -o wide
```

### Verification Layer 2: Functional Application Validation
- **Inspect Clean Logs:** Verify that startup errors are absent:
  ```powershell
  kubectl logs <pod-name>
  ```
- **Test In-Cluster Network Connectivity:** If a Service was repaired, query its endpoint using an ephemeral debug container:
  ```powershell
  $svcIP = (kubectl get service <service-name> -o jsonpath='{.spec.clusterIP}')
  kubectl run test-client --image=busybox:1.36 --restart=Never --rm -i --timeout=10s -- wget -q -O - "http://$svcIP"
  ```
- **Verify Endpoints:** Confirm that the EndpointSlice controller has populated target IPs:
  ```powershell
  kubectl get endpoints <service-name>
  ```

---

## 8. Capture Evidence

To document the successful investigation, capture terminal screenshots of key milestones:

### What to Capture:
1. **Initial Failure State:** `kubectl get` showing the error status (`ImagePullBackOff`, `Pending`, `CrashLoopBackOff`, etc.).
2. **Diagnostic Evidence:** `kubectl describe` highlighting the specific event, condition, or log line establishing the root cause.
3. **The Correction:** Applying the fixed manifest or displaying the corrected YAML block.
4. **Successful Verification:** Output proving recovery (`STATUS: Running`, `READY: 1/1`, rollout successfully completed).

### Screenshot Storage Rules:
- Save all images directly into the issue's existing `screenshots/` directory.
- Use numbered, descriptive filenames:
  ```text
  01-invalid-status.png
  02-diagnostic-event.png
  03-fixed-manifest.png
  04-verified-running.png
  ```
- **Policy:** Never commit placeholder or fake screenshots. Keep `screenshots/` empty until genuine manual captures are generated.

---

## 9. Document the Findings

Review the issue's `README.md` and confirm that all observed behaviors match the documentation:
- Were error codes and messages identical to the documented output?
- Did any unexpected cluster-specific behaviors occur?
- Record any local environment nuances under Section 11 (*Key Learning*) if necessary.

---

## 10. Clean Up Safely

Always restore your cluster to a pristine state before proceeding to the next scenario:

```powershell
# Clean up using the fixed manifest
kubectl delete -f .\fixed-pod.yaml --ignore-not-found=true

# Confirm that test resources are fully terminated
kubectl get pods -l app=<issue-label>
```

### Safety Principles:
- **Targeted Teardown:** Only delete resources created specifically for that exercise.
- **Never Run Broad Commands:** Avoid `kubectl delete all --all` or sweeping namespace deletions.
- **Confirm Context:** Double-check `kubectl config current-context` before executing delete operations.

---

## 11. Troubleshooting Completion Checklist

Track your progress through each troubleshooting exercise using this 7-step checklist:

- [ ] **1. Problem Understood:** Read issue `README.md` and identified expected behavior.
- [ ] **2. Environment Checked:** Verified Minikube is `Running` and node is `Ready`.
- [ ] **3. Issue Reproduced:** Applied `invalid-*.yaml` and observed the documented error state.
- [ ] **4. Evidence Collected:** Ran diagnostic sequence (`describe`, `logs`, `events`) and extracted the root cause.
- [ ] **5. Fix Applied:** Applied `fixed-*.yaml` and understood why the change resolves the issue.
- [ ] **6. Recovery Verified:** Confirmed `Running`/`Ready` state and tested application connectivity.
- [ ] **7. Evidence & Cleanup:** Captured terminal screenshots into `screenshots/` and deleted test resources.

---

## 12. Escalation and Advanced Diagnosis

If an issue cannot be resolved through standard pod-level diagnostics, escalate your investigation across deeper cluster tiers:

1. **Node-Level Health:** Check node resources and disk pressure:
   ```powershell
   kubectl describe node minikube | Select-String "Conditions:","Allocated resources:" -Context 0,6
   ```
2. **Kubelet and Runtime Logs:** Inspect systemd journals inside the Minikube node:
   ```powershell
   minikube ssh -- journalctl -u kubelet -n 50 --no-pager
   ```
3. **CoreDNS and Cluster Networking:** Verify that CoreDNS and kube-proxy are healthy:
   ```powershell
   kubectl get pods -n kube-system -l k8s-app=kube-dns
   kubectl logs -n kube-system -l k8s-app=kube-dns --tail=30
   ```
4. **Storage Provisioner Logs:** If PVCs fail to bind, inspect the storage provisioner:
   ```powershell
   kubectl logs -n kube-system -l integration-test=storage-provisioner
   ```
5. **API Server Authorization:** Test authorization for specific operations using `kubectl auth can-i`.
