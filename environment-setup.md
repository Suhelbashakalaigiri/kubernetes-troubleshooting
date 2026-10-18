# Environment Setup and Validation Guide

This guide details the prerequisites, verification steps, and health checks required to configure a local Windows workstation for executing the exercises in the `kubernetes-troubleshooting` repository.

For the high-level project overview, see [README.md](README.md).  
For the standard diagnostic procedure, see [troubleshooting-workflow.md](troubleshooting-workflow.md).

---

## 1. Purpose

The troubleshooting exercises in this repository simulate real-world failure states by deploying intentionally misconfigured manifests to a live cluster. To ensure that failures are deterministic and caused by the manifest misconfigurations—rather than an unstable local cluster—the underlying workstation and Minikube environment must be validated before starting any exercise.

---

## 2. Prerequisites

The following software and environment dependencies are required:

| Component | Requirement | Role in Troubleshooting |
| :--- | :--- | :--- |
| **Operating System** | Windows 10 / 11 (64-bit) | Workstation host OS. |
| **Terminal** | Windows PowerShell 5.1+ or PowerShell 7+ | Command shell used for all commands and file operations. |
| **Container Engine** | Docker Desktop (or Hyper-V / VirtualBox) | Backend virtualization driver hosting the Minikube node. |
| **Kubernetes Engine** | Minikube | Single-node local Kubernetes cluster. |
| **Kubernetes CLI** | `kubectl` | Primary command-line client used to inspect and manage cluster objects. |
| **Version Control** | Git | Manages the repository, branches, and diff tracking. |
| **Network** | Stable Internet Connectivity | Required to download public container images (`nginx:1.25-alpine`, `busybox:1.36`). |

---

## 3. Installation and Setup

If any required tools are not yet installed on your system, install them using their official documentation:

- **Docker Desktop:** [Docker Desktop for Windows Installation Guide](https://docs.docker.com/desktop/setup/install/windows-install/)  
  *Ensure that the Docker engine is running and WSL 2 backend or Hyper-V is enabled.*
- **Minikube:** [Minikube Getting Started Guide](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fwindows%2Fx86_64)  
  *Minikube can be installed via Windows Package Manager (`winget install Kubernetes.minikube`) or Chocolatey (`choco install minikube`).*
- **kubectl:** [Install and Set Up kubectl on Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)  
  *Install via winget (`winget install Kubernetes.kubectl`), Chocolatey, or direct binary download.*
- **Git:** [Git for Windows Download](https://git-scm.com/download/win)

> [!IMPORTANT]
> Do not reinstall or reconfigure tools that are already installed and functioning on your machine. Run the verification commands in Section 4 first.

---

## 4. Version and Installation Checks

Open Windows PowerShell and check that all required CLI binaries are installed and accessible in your system's `PATH`:

```powershell
docker --version
minikube version
kubectl version --client
git --version
```

### Interpreting Output:
- Each command should return its respective version string without errors.
- If PowerShell returns `'<command>' is not recognized as the name of a cmdlet, function, script file, or operable program`, the binary is either not installed or its directory has not been added to your system environment variable `$env:PATH`.

To inspect your current `PATH` in PowerShell:

```powershell
$env:PATH -split ';'
```

---

## 5. Start and Validate Minikube

### Step 1: Check Current Minikube Status
Check whether a Minikube cluster instance is already running:

```powershell
minikube status
```

- **If already running:** The output will indicate `host: Running`, `kubelet: Running`, `apiserver: Running`, and `kubeconfig: Configured`. Proceed directly to Section 6.
- **If stopped:** Start the cluster as described in Step 2.

### Step 2: Start Minikube (If Stopped)
Start Minikube using the driver appropriate for your environment:

For Docker Desktop (the default on Windows with WSL 2):
```powershell
minikube start --driver=docker
```

For Hyper-V (requires Administrator privileges):
```powershell
minikube start --driver=hyperv
```

### Step 3: Validate Cluster Node and System Pods
Confirm that the Kubernetes node is online and system components are healthy:

```powershell
kubectl get nodes
```

Expected output:
```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   v1.30.x (or similar)
```

Confirm that the core control plane and DNS pods are running:

```powershell
kubectl get pods -n kube-system
```

Expected output:
All system pods—including `coredns`, `kube-apiserver`, `kube-controller-manager`, `kube-proxy`, and `storage-provisioner`—should report `STATUS: Running` or `Completed`.

---

## 6. Navigate to the Repository

Navigate to your local repository directory in Windows PowerShell:

```powershell
# Navigate to the repository root directory
Set-Location -Path "$HOME\Documents\project\GO Projects\kubernetes-troubleshooting"

# Verify current working directory
Get-Location
```

To begin a specific troubleshooting scenario, change directory into the corresponding issue folder using PowerShell path notation:

```powershell
# Example: Navigating to the ImagePullBackOff scenario
Set-Location -Path ".\pod\01-image-pull-backoff"

# List directory contents
Get-ChildItem
```

Each issue directory contains:
- `README.md` — The step-by-step issue runbook.
- `invalid-*.yaml` — The manifest that deliberately introduces the error.
- `fixed-*.yaml` — The corrected manifest that resolves the error.
- `screenshots/` — The directory reserved for saving verification evidence.

---

## 7. Initial Health Check

Before executing troubleshooting scenarios, run this rapid 4-step health check sequence in PowerShell:

```powershell
# 1. Verify Minikube status
minikube status

# 2. Verify active kubectl context is set to minikube
kubectl config current-context

# 3. Verify the node is Ready
kubectl get nodes

# 4. Verify API communication and cluster endpoints
kubectl cluster-info
```

### Acceptance Criteria:
1. `minikube status` confirms `apiserver: Running`.
2. `kubectl config current-context` outputs `minikube`.
3. `kubectl get nodes` shows `minikube` in the `Ready` state.
4. `kubectl cluster-info` displays accessible URLs for the Kubernetes control plane.

---

## 8. Common Environment Problems and Diagnostics

### Problem 1: `kubectl` Commands Hang or Return Connection Refused
- **Symptom:** `Unable to connect to the server: dial tcp 127.0.0.1:XXXXX: connectex: No connection could be made...`
- **Cause:** Minikube is stopped, or the VM IP address changed after host sleep/reboot.
- **Diagnostic:** Run `minikube status`.
- **Solution:** Run `minikube start` to resume the cluster and refresh your local `~/.kube/config`.

### Problem 2: Wrong Active Context
- **Symptom:** Commands execute against an unexpected cluster or fail with unauthorized errors.
- **Diagnostic:** Run `kubectl config current-context`.
- **Solution:** Switch context explicitly back to Minikube:
  ```powershell
  kubectl config use-context minikube
  ```

### Problem 3: Docker Daemon is Not Running
- **Symptom:** `minikube start --driver=docker` fails with `Exiting due to PROVIDER_DOCKER_NOT_RUNNING`.
- **Cause:** Docker Desktop has not been launched or WSL 2 integration is stopped.
- **Diagnostic:** Run `docker ps` in PowerShell.
- **Solution:** Open Docker Desktop from the Windows Start menu and wait until the whale icon in the taskbar indicates the engine is running.

### Problem 4: Container Image Pull Timeouts
- **Symptom:** Workloads that depend on valid images hang in `ContainerCreating` or report network timeouts.
- **Cause:** Minikube node lacks DNS routing or internet access through the host network adapter.
- **Diagnostic:** Test external connectivity from inside the Minikube node:
  ```powershell
  minikube ssh -- ping -c 3 registry-1.docker.io
  ```
- **Solution:** Restart Docker Desktop or check VPN/corporate proxy settings that may intercept egress traffic.

### Problem 5: Missing Tools in PowerShell `$env:PATH`
- **Symptom:** PowerShell fails to recognize `minikube` or `kubectl`.
- **Cause:** Tool binaries were downloaded without updating the system environment variables.
- **Solution:** Append the directory containing your binaries to `$env:PATH` or configure it permanently in your Windows System Properties (`sysdm.cpl`).

---

## 9. Environment Safety

- **Context Isolation:** Always verify `kubectl config current-context` before running any commands. The exercises in this repository must **never** be executed against shared staging, QA, or production clusters.
- **No Production Secrets:** The manifests in this repository intentionally use dummy configuration strings and dummy base64 values. Never substitute real API keys, cloud tokens, or production passwords into test manifests.
- **Scoped Teardown:** When cleaning up an exercise, always use the explicit manifest files provided (e.g. `kubectl delete -f .\fixed-pod.yaml`). Never run broad, destructive commands such as:
  ```powershell
  # DO NOT RUN:
  kubectl delete all --all
  ```
- **Clean Node State:** Do not modify the underlying Minikube node's systemd services or container runtime configurations unless specifically instructed by a scenario.

---

## 10. Ready-to-Troubleshoot Checklist

Complete this checklist prior to starting any issue in the repository:

- [ ] Docker Desktop is running (if using the Docker driver).
- [ ] `minikube status` confirms all components are `Running`.
- [ ] `kubectl config current-context` outputs `minikube`.
- [ ] `kubectl get nodes` confirms node status is `Ready`.
- [ ] Internet access is verified for pulling base images.
- [ ] PowerShell is navigated to the intended issue directory (e.g. `Set-Location .\pod\01-image-pull-backoff`).
- [ ] You have read the issue `README.md` and understand the reproduction steps.
