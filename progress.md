# Progress Report: Kubernetes Operator for VM Disk Image Management

## Environment Requirements

### Kubernetes Configuration

For developing and testing a VM disk image operator that uses the `ImageVolume` feature, we need:

1. **Kubernetes Version**: 1.31+ (we're using 1.33.0)
2. **Required Feature Gates**:
   - `ImageVolume=true` - must be enabled on:
     - API Server
     - Controller Manager
     - Scheduler
     - Kubelet

### Talos Linux Configuration

When using Talos Linux to manage the Kubernetes cluster:

```yaml
# Example feature gate configuration
machine:
  kubelet:
    extraArgs:
      feature-gates: "ImageVolume=true"

cluster:
  apiServer:
    extraArgs:
      feature-gates: "ImageVolume=true"
  controllerManager:
    extraArgs:
      feature-gates: "ImageVolume=true"
  scheduler:
    extraArgs:
      feature-gates: "ImageVolume=true"
```

## Testing Image Volumes

To verify the `ImageVolume` feature is working:

```yaml
# Test pod configuration
apiVersion: v1
kind: Pod
metadata:
  name: image-volume-test
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: test-volume
      mountPath: /mnt/test
  volumes:
  - name: test-volume
    image:
      reference: docker.io/library/alpine:latest
```

## Automation Setup

- Created a Taskfile to automate cluster creation/destruction and testing
- Configuring the process to be repeatable for future development

## Implementation Status and Issues

### Current Status

- [x] Successfully created Kubernetes 1.33.0 cluster with Talos Linux
- [x] Confirmed feature gates are enabled on all required components
- [ ] Functional ImageVolume implementation

### ImageVolume Feature Testing Results

After testing the ImageVolume feature with several pod configurations, we found significant limitations:

1. **Container Runtime Issue**: When attempting to create pods with ImageVolumes, we consistently encounter errors:
   ```
   Error: failed to generate container spec: failed to apply OCI options: failed to mkdir '': mkdir : no such file or directory
   ```

2. **Root Cause Analysis**:
   - The error occurs in the containerd runtime while trying to create the container
   - The feature gate is properly enabled in Kubernetes components
   - Kubelet logs show the container creation is failing during volume initialization
   - Research indicates this is a known issue with the ImageVolume feature in current container runtimes

3. **Alpha Feature Limitations**:
   - The ImageVolume feature is still in alpha status in Kubernetes 1.33.0
   - While the Kubernetes API supports the feature, the container runtime implementation is incomplete
   - This has been reported by users across multiple Kubernetes distributions (kind, k3s, etc.)

## Updated Approach: lib-volume-populator

After discovering the limitations with the ImageVolume feature, we've identified an alternative approach using the lib-volume-populator library:

### Library Overview

The [lib-volume-populator](https://github.com/kubernetes-csi/lib-volume-populator) is an official Kubernetes CSI project that provides:

1. **Framework for Custom Volume Populators**:
   - Allows implementation of custom volume population mechanisms
   - Handles Kubernetes API interactions and lifecycle management
   - Works with current Kubernetes versions

2. **Implementation Pattern**:
   - Define custom resources (CRs) for data sources
   - Reference these CRs from PVCs using `dataSourceRef`
   - Controller watches for PVCs and manages the population process

3. **Key Components**:
   - Controller that watches for PVCs with specific data sources
   - Provider function callbacks for population logic
   - Custom resource definitions for data source specifications

### Implementation Plan for VM Disk Images

Based on our analysis of lib-volume-populator examples, we plan to:

1. **Create a VMDiskImage CRD**:
   - Define fields for source image, format, target size, etc.
   - Implement validation rules and status fields
   - Create example VMDiskImage resources

2. **Implement a Custom Volume Populator**:
   - Create a controller using the lib-volume-populator framework
   - Implement the three key functions:
     ```go
     func populateFn(ctx context.Context, params populatorMachinery.PopulatorParams) error {
         // Create a pod to pull and process the VM disk image
     }
     
     func populateCompleteFn(ctx context.Context, params populatorMachinery.PopulatorParams) (bool, error) {
         // Check if the population process is complete
     }
     
     func populateCleanupFn(ctx context.Context, params populatorMachinery.PopulatorParams) error {
         // Clean up temporary resources
     }
     ```

3. **Build a Helper Container Image**:
   - Include tools like skopeo for OCI image acquisition
   - Include qemu-img for disk image resizing and format conversion
   - Use this image in the populator pods

### Immediate Next Steps

1. Create a simple POC implementation based on the provider-populator example:
   - Define a basic VMDiskImage CRD
   - Implement a minimal controller
   - Test with a simple VM disk image

2. Develop the complete solution:
   - Refine the CRD with additional fields and validation
   - Implement robust error handling and status tracking
   - Add support for different image formats and optimization options

This approach provides a clear path forward that works with current Kubernetes versions while offering more flexibility than the built-in ImageVolume feature. It allows us to implement the desired VM disk image management features without waiting for the ImageVolume feature to mature.

## Comparing Approaches

| Feature | ImageVolume | Custom Volume Populator |
|---------|-------------|-------------------------|
| **Implementation Status** | Alpha, incomplete in runtimes | Production-ready framework |
| **Flexibility** | Limited to direct mounting | Custom logic, resizing, conversion |
| **Compatibility** | Requires specific K8s version | Works with current K8s versions |
| **Maintainability** | Dependent on upstream changes | Self-contained implementation |
| **Complexity** | Simple but inflexible | More complex but more powerful |

## Updated Timeline

1. Complete research and POC of volume populator implementation (1-2 weeks)
2. Develop full implementation of VM disk image populator (2-3 weeks)
3. Testing and validation with various VM disk image formats (1 week)
4. Documentation and packaging (1 week)

This revised approach positions us to deliver a functional VM disk image management solution using current Kubernetes capabilities, while maintaining the option to leverage the ImageVolume feature once it matures in future Kubernetes releases.