# Troubleshooting: HPA Scaling Issues (Missing Resource Requests)

## 1. Issue overview
A Horizontal Pod Autoscaler (HPA) scaling failure occurs when the HPA controller cannot calculate resource utilization metrics for its target Deployment, causing `TARGETS` to display `<unknown>/50%` and preventing automated scaling.

In Kubernetes:
- When an HPA specifies `averageUtilization: 50` for CPU, the HPA controller queries the Metrics Server for current pod CPU usage and calculates:
  $$\text{Utilization Percentage} = \frac{\text{Actual CPU Usage}}{\text{spec.containers[*].resources.requests.cpu}} \times 100$$
- If the Deployment's container specification **omits `resources.requests.cpu`**, the denominator is zero.
- The controller cannot compute a percentage, marks condition `ScalingActive: False`, and refuses to scale the Deployment.

**Expected behavior:** The HPA computes current CPU percentage against requested CPU and scales replicas as load fluctuates.  
**Actual behavior:** The Deployment omits CPU requests, causing HPA to display `TARGETS: <unknown>/50%` with `FailedGetResourceMetric` events.

---

## 2. Common causes
1. **Missing `resources.requests.cpu`:** Deployment does not declare CPU requests.
2. **Metrics Server not installed or offline:** The `metrics-server` pod is not running in `kube-system`.
3. **Invalid target reference:** `scaleTargetRef` points to a non-existent Deployment name or wrong API version.
4. **Target utilization threshold too aggressive:** Setting threshold to 10% causes flapping (rapid scale up/down).

---

## 3. Prerequisites
- A running Minikube cluster (`minikube status`).
- `kubectl` configured for your cluster.

---

## 4. Reproduce the issue
Navigate to the issue folder in Windows PowerShell:

```powershell
Set-Location -Path ".\production-level\03-hpa-scaling-issues"
```

Apply the invalid manifests:

```powershell
kubectl apply -f .\invalid-hpa.yaml
```

Wait 15 seconds, then query the HPA:

```powershell
kubectl get hpa hpa-target-demo
```

Observe `<unknown>` in the `TARGETS` column:
```text
NAME              REFERENCE                       TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-target-demo   Deployment/deploy-hpa-target    <unknown>/50%   1         5         1          30s
```

---

## 5. Identify and diagnose the issue

Execute the diagnostic sequence below:

### Step 1: Inspect HPA events and conditions
```powershell
kubectl describe hpa hpa-target-demo
```

Examine the **Conditions** block:
```text
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the server could not compute a percentage of the requested resource (cpu), because the resource request is not set
```

Look at the **Events** section:
```text
Events:
  Type     Reason                   Age   From                       Message
  ----     ------                   ----  ----                       -------
  Warning  FailedGetResourceMetric  15s   horizontal-pod-autoscaler  the server could not compute a percentage of the requested resource (cpu), because the resource request is not set
```

### Step 2: Check the Deployment resource specification
```powershell
kubectl get deployment deploy-hpa-target -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

Output:
```text
{}
```
*The container has zero resource requests defined.*

**Diagnostic analysis:**
- The HPA warning explicitly identifies: `the resource request is not set`.
- Without a declared CPU request, the mathematical formula for percentage utilization cannot evaluate.

---

## 6. Root cause
In `invalid-hpa.yaml`, the Deployment container specification omits `resources.requests.cpu`, making it impossible for the HPA controller to calculate percentage CPU utilization.

---

## 7. Fix the issue
Apply the fixed manifest:

```powershell
kubectl apply -f .\fixed-hpa.yaml
```

The corrected manifest `fixed-hpa.yaml` defines `resources.requests.cpu: "50m"`.

---

## 8. Verify the fix
Wait 20 to 30 seconds for the Metrics Server to collect the next metrics scrape:

```powershell
kubectl get hpa hpa-target-demo
```

Expected output:
```text
NAME              REFERENCE                       TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hpa-target-demo   Deployment/deploy-hpa-target    0%/50%    1         5         1          2m
```
*Notice that `<unknown>/50%` has resolved to a real percentage (`0%/50%`).*

Verify that `ScalingActive` condition is now `True`:

```powershell
kubectl describe hpa hpa-target-demo | Select-String "ScalingActive","AbleToScale" -Context 0,1
```

---

## 9. Screenshot checklist
Capture the following screenshots and save them under `screenshots/`:

- [ ] `01-hpa-unknown-targets.png` — Output of `kubectl get hpa` showing `<unknown>/50%`.
- [ ] `02-failed-get-resource-metric.png` — Output of `kubectl describe hpa` showing `resource request is not set`.
- [ ] `03-empty-resources-check.png` — Output showing Deployment resources are empty `{}`.
- [ ] `04-verified-hpa-active.png` — Output of `kubectl get hpa` showing valid percentage (e.g. `0%/50%`).

---

## 10. Cleanup
Delete all resources:

```powershell
kubectl delete -f .\fixed-hpa.yaml --ignore-not-found=true
```

Confirm cleanup:

```powershell
kubectl get hpa hpa-target-demo
kubectl get deployment deploy-hpa-target
```

---

## 11. Key learning
- **Root cause:** The target Deployment lacked `resources.requests.cpu`, preventing HPA from calculating utilization percentages.
- **Most useful diagnostic command:** `kubectl describe hpa <name>` to view `FailedGetResourceMetric` events.
- **Correction:** Add explicit CPU requests to all containers targeted by an HPA.
- **Production consideration:** Always combine CPU requests with memory limits, and consider testing custom metrics (e.g. requests per second from Prometheus) alongside standard CPU utilization.
