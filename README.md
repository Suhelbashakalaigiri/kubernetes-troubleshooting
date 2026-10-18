# Kubernetes Troubleshooting Guide

A practical, hands-on Kubernetes troubleshooting repository designed for diagnosing, understanding, and resolving real-world failures on Windows using Windows PowerShell, Minikube, and `kubectl`.

---

## 1. Project Overview

Kubernetes is a declarative container orchestration platform composed of multiple decoupled control-plane and node-level controllers. When workloads fail, the root cause rarely presents itself as a simple error message; instead, operators observe high-level symptoms across Pod phases, container waiting reasons, controller conditions, and event streams.

This repository provides an organized catalog of common Kubernetes failure modes. Each scenario includes:
- **An intentionally misconfigured manifest** (`invalid-*.yaml`) that safely reproduces a specific failure in an isolated environment.
- **A corrected manifest** (`fixed-*.yaml`) demonstrating the architectural and configuration fix.
- **A step-by-step troubleshooting runbook** (`README.md`) detailing the symptoms, diagnostics, root-cause analysis, verification steps, and safe cleanup.
- **A designated screenshots directory** (`screenshots/`) for storing terminal evidence captured during manual scenario reproduction.

For detailed instructions on preparing your workstation, refer to the [Environment Setup Guide](environment-setup.md). To learn the standard diagnostic methodology, refer to the [Troubleshooting Workflow](troubleshooting-workflow.md).

---

## 2. Project Objectives

- **Develop Practical Troubleshooting Skills:** Move beyond theory to hands-on investigation of broken workloads inside a live Minikube cluster.
- **Isolate Symptoms from Root Causes:** Learn to differentiate between high-level conditions (such as `CrashLoopBackOff` or `Pending`) and their underlying causes (such as unhandled exceptions, missing configuration keys, or unschedulable resource constraints).
- **Master Native Diagnostic Tooling:** Utilize `kubectl describe`, `kubectl logs`, JSONPath queries, event filtering, and API inspection without relying on third-party black-box tools.
- **Establish a Reproducible Knowledge Base:** Maintain a structured, peer-reviewable runbook collection suitable for personal learning, onboarding, and production incident response.

---

## 3. Repository Structure

The repository is structured into six functional troubleshooting categories containing 49 self-contained issue folders:

```text
kubernetes-troubleshooting/
├── README.md                      # Main project documentation (this file)
├── environment-setup.md           # Local workstation & Minikube setup guide
├── troubleshooting-workflow.md    # Standard 12-stage troubleshooting procedure
├── issue-index.md                 # Complete index of all 49 scenarios
│
├── pod/                           # Pod-level lifecycle & container runtime issues (12)
├── deployment/                    # Workload rollouts, replicas & strategy issues (5)
├── service-networking/            # Endpoints, routing, DNS & Ingress issues (7)
├── configuration-application/     # ConfigMaps, Secrets, entrypoints & refresh issues (7)
├── storage-database/              # PVCs, volume mounts, credentials & migration issues (8)
└── production-level/              # Capacity, PDBs, HPA, RBAC & multi-layer incidents (10)
```

### Representative Issue Folder Structure

Each issue folder is isolated, uniquely named, and organized uniformly:

```text
category/
└── 01-issue-name/
    ├── README.md                  # Comprehensive 11-section troubleshooting guide
    ├── invalid-*.yaml             # Intentionally misconfigured manifest reproducing the failure
    ├── fixed-*.yaml               # Corrected manifest resolving the failure
    └── screenshots/               # Directory for storing captured terminal evidence
```

*Note on Screenshots:* The `screenshots/` directory is reserved for capturing terminal evidence after executing an exercise. Initial scenarios under `pod/01` through `pod/04` contain verified captures, while remaining scenarios contain empty placeholder directories awaiting manual execution.

---

## 4. Troubleshooting Categories

| Category | Focus Area | Scenarios Covered |
| :--- | :--- | :--- |
| [**`pod/`**](pod/) | Pod Lifecycle & Runtime | Image pull failures (`ImagePullBackOff`, `ErrImagePull`), crash loops, scheduling delays, container startup errors (`CreateContainerConfigError`, `CreateContainerError`), memory exhaustion (`OOMKilled`), failed volume mounts, and probe failures (readiness, liveness, startup). |
| [**`deployment/`**](deployment/) | Controller & Workloads | Stalled rollouts, unready replica metrics, rollback procedures (`kubectl rollout undo`), deadline timeouts (`ProgressDeadlineExceeded`), and rolling update strategy outages (`maxUnavailable: 100%`). |
| [**`service-networking/`**](service-networking/) | Services, DNS & Ingress | Mismatched label selectors (empty endpoints), `targetPort` misconfigurations, Minikube NodePort routing boundaries on Windows, CoreDNS resolution (`NXDOMAIN`), Ingress 404/502 errors, NetworkPolicy isolation, and TCP RST vs. timeout analysis. |
| [**`configuration-application/`**](configuration-application/) | Config, Secrets & Code | Missing ConfigMap/Secret keys, base64 transparency, invalid entrypoint arguments, environment variable immutability vs. dynamic volume updates, application startup crashes, probe dependency coupling, and CLI version drift. |
| [**`storage-database/`**](storage-database/) | Storage & Stateful Workloads | PVCs stuck in `Pending`, unbound volume mounts, API admission referential integrity, database host resolution failures, authentication rejections, `initContainers` startup gates, state loss on ephemeral containers, and migration Job backoff limits. |
| [**`production-level/`**](production-level/) | Cluster Reliability & Security | Cluster capacity exhaustion, node unreadiness tolerations, HPA missing CPU requests, PDB drain deadlocks, RBAC missing verbs, warmup outages, node taints, hard anti-affinity deadlocks, ServiceAccount token 403s, and triple-layer incidents. |

---

## 5. Standard Issue Documentation Format

Every issue-specific `README.md` adheres to a strict 11-section runbook structure:

1. **Issue Overview:** Architectural explanation of the condition and expected vs. actual behavior.
2. **Common Causes:** Top operational reasons for the failure in production clusters.
3. **Prerequisites:** Cluster state and requirements needed prior to execution.
4. **Reproduce the Issue:** Exact PowerShell commands to apply the invalid manifest and inspect initial symptoms.
5. **Identify and Diagnose:** Step-by-step diagnostic sequence using `kubectl describe`, logs, and JSONPath filters.
6. **Root Cause:** Detailed technical explanation of the manifest flaw.
7. **Fix the Issue:** Exact command to apply the corrected manifest and the rationale behind the fix.
8. **Verify the Fix:** Verification commands (`kubectl rollout status`, `kubectl wait`, log checks) proving recovery.
9. **Screenshot Checklist:** Itemized list of terminal outputs to capture and save in `screenshots/`.
10. **Cleanup:** Scoped PowerShell commands to delete only the resources created for that exercise.
11. **Key Learning:** Summary of the root cause, primary diagnostic tool, correction, and production prevention tips.

---

## 6. Getting Started

Before attempting any troubleshooting exercise, verify that your local environment is operational:

### 1. Check Cluster and Context
Open Windows PowerShell and check your Minikube status and active context:

```powershell
minikube status
kubectl config current-context
kubectl get nodes
```

Expected output indicates that Minikube is `Running`, the context is set to `minikube`, and the node status is `Ready`.

### 2. Navigate to an Issue Directory
Change directory into any issue folder using PowerShell paths:

```powershell
Set-Location -Path ".\pod\01-image-pull-backoff"
```

### 3. Follow the Issue Guide
Open the local `README.md` file, follow the reproduction steps, diagnose the failure, apply the fix, and clean up afterwards.

For full environment prerequisites, see the [Environment Setup Guide](environment-setup.md).  
For the end-to-end diagnostic methodology, see the [Troubleshooting Workflow](troubleshooting-workflow.md).

---

## 7. Project Status

- **Architecture and Manifests:** Complete. All 49 issue directories, 85+ Kubernetes YAML manifests, and 49 detailed runbooks have been authored and structurally validated.
- **Scenario Verification:** In progress. Troubleshooting exercises are executed incrementally. Scenarios `pod/01` through `pod/04` have been manually executed with verified terminal screenshots captured in their respective `screenshots/` folders. Remaining levels are prepared for progressive manual verification.
- **Scope:** Scenarios are designed for standalone reproduction on a standard single-node Minikube cluster running on Windows.

---

## 8. Safety and Best Practices

- **Verify Your Context:** Always confirm you are connected to your local `minikube` context before executing commands (`kubectl config current-context`). Never run test scripts against staging or production clusters.
- **Resource Isolation:** Exercises use unique, descriptive resource names (e.g. `pod-image-pull-backoff`, `deploy-failed-rollout`) to prevent collisions with existing workloads.
- **Scoped Cleanup:** Run only the cleanup commands specified in each issue's `README.md`. Never execute broad deletion commands (such as `kubectl delete all --all`) that could destroy unrelated cluster resources.
- **Credential Hygiene:** All Secret manifests in this repository use dummy values (e.g. `demo-secret-key-12345`). Never commit real passwords, tokens, or private keys to source control.
- **Dry-Run Validation:** Use `kubectl apply --dry-run=client -f <file>` to validate manifest syntax before applying changes to the cluster.

---

## 9. Future Improvements

- Complete manual screenshot capture across all remaining categories (`deployment/`, `service-networking/`, `configuration-application/`, `storage-database/`, `production-level/`).
- Integrate manifest linting and API schema validation using `kubeconform` in a GitHub Actions CI pipeline.
- Add multi-node Minikube scenarios (`minikube start --nodes 2`) for advanced pod topology spread and affinity testing.
- Expand stateful database scenarios to include automated backup, snapshot, and restore workflows.
