
# 11. FailedMount — Missing Volume Dependency

## Objective

Understand how to troubleshoot a Kubernetes Pod that fails to start because a required volume dependency is missing.

## Issue Description

The Pod references a ConfigMap named `non-existent-configmap` as a volume source. Since the ConfigMap does not exist in the Pod's namespace, Kubernetes cannot set up the volume.

The Pod can be scheduled to a node, but the kubelet fails to prepare the volume, preventing the container from starting.

## Files

- `invalid-pod.yaml` — Pod manifest that references a nonexistent ConfigMap.
- `fixed-pod.yaml` — Corrected Pod manifest using an `emptyDir` volume.

## Steps to Reproduce

### 1. Apply the invalid manifest

```powershell
kubectl apply -f .\invalid-pod.yaml
```

### 2. Check the Pod status

```powershell
kubectl get pod pod-failed-mount
```

The Pod may remain in `ContainerCreating` or another waiting state while Kubernetes attempts to prepare its volume.

### 3. Describe the Pod

```powershell
kubectl describe pod pod-failed-mount
```

Inspect the `Events` section for a message similar to:

```text
Warning  FailedMount  kubelet
MountVolume.SetUp failed for volume "missing-config-volume":
configmap "non-existent-configmap" not found
```

The exact message and timing may vary. Confirm that the events identify the missing ConfigMap as the cause.

## Expected Behavior

- The Pod is scheduled to a node.
- The kubelet attempts to prepare the ConfigMap volume.
- Volume setup fails because the referenced ConfigMap does not exist.
- A `FailedMount` event is reported.
- The container cannot start until the volume dependency is resolved.

## Root Cause

The Pod references a ConfigMap that does not exist in the same namespace as the Pod.

Kubernetes requires the referenced ConfigMap to be available when setting up a non-optional ConfigMap volume.

## Troubleshooting Commands

### Check the Pod status

```powershell
kubectl get pods
```

### Inspect Pod details and events

```powershell
kubectl describe pod pod-failed-mount
```

### Verify whether the ConfigMap exists

```powershell
kubectl get configmap non-existent-configmap
```

### List available ConfigMaps

```powershell
kubectl get configmaps
```

## Resolution

There are two possible approaches.

### Option 1: Create the required ConfigMap

If the application requires configuration data, create the ConfigMap with the expected configuration and ensure the name matches the reference in the Pod manifest.

### Option 2: Correct the volume definition

If the application does not require ConfigMap data, replace the ConfigMap volume with an appropriate volume type.

In this exercise, `fixed-pod.yaml` uses an `emptyDir` volume to demonstrate successful volume setup without requiring an external ConfigMap.

## Apply the Fix

### 1. Delete the failed Pod

```powershell
kubectl delete pod pod-failed-mount
```

### 2. Apply the fixed manifest

```powershell
kubectl apply -f .\fixed-pod.yaml
```

### 3. Verify the Pod status

```powershell
kubectl get pod pod-failed-mount -w
```

Wait for the Pod to reach `Running`, then press `Ctrl+C` to stop watching.

### 4. Verify the final state

```powershell
kubectl describe pod pod-failed-mount
```

Confirm that the container is running and no new `FailedMount` events are occurring.

## Key Learnings

- `FailedMount` indicates that Kubernetes could not successfully prepare or mount a required volume.
- A Pod may be scheduled successfully but still fail to start because of a volume dependency.
- `kubectl describe pod` is essential for identifying volume-related errors in the `Events` section.
- A missing ConfigMap can prevent a non-optional ConfigMap volume from being prepared.
- The correct fix depends on the application's requirements: restore the missing dependency or correct the volume configuration.
- `emptyDir` provides temporary storage for a Pod; it does not replace configuration data supplied by a ConfigMap.
