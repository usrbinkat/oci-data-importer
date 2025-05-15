# Journey Log: Kubernetes Operator for VM Disk Image Management

## Environment Setup Challenges

### Enabling the ImageVolume Feature Gate in Talos Kubernetes

We encountered challenges when attempting to enable the `ImageVolume` feature gate in a Talos-managed Kubernetes cluster:

1. **Initial Observation**: 
   - Our Kubernetes cluster (v1.33.0) was created using `talosctl cluster create`
   - The `ImageVolume` feature gate appeared to be disabled by default
   - Test pod with `volumes.image` specification had this field dropped in actual pod spec

2. **Configuration Attempts**:
   - Created various Talos configuration patches to enable feature gates
   - Initial patches caused cluster creation to time out at etcd stage
   - Experimenting with proper patch format and minimum required changes

3. **Pod Security Policy**:
   - Received Pod Security Policy warnings when creating test pods
   - Need to either:
     - Configure pods to comply with restricted PSP
     - Disable/modify the Pod Security Admission controller

### Configuration Cleanup Issues

A significant challenge we encountered was with leftover configuration artifacts affecting cluster recreation:

1. **Talos and Kubernetes Context Accumulation**:
   - Multiple failed cluster creation attempts left behind numerous contexts
   - Contexts from previous runs interfered with new cluster creation
   - Manual cleanup of these contexts was difficult due to "current context" restrictions

2. **Solution: Automated Cleanup**:
   - Created comprehensive Taskfile tasks to:
     - Clean Kubernetes contexts, clusters, and users
     - Reset kubeconfig to empty state
     - Clean Talos contexts and state directories
     - Destroy existing clusters before recreating

This experience highlighted the importance of thorough cleanup between test iterations when working with Kubernetes cluster tools.

## Successfully Enabling Feature Gates

After multiple attempts, we successfully created a Kubernetes cluster with the ImageVolume feature gate enabled:

1. **Successful Configuration**:
   - Simplified the Talos configuration patch to focus only on feature gate enablement
   - Removed Pod Security Policy customizations which were causing issues
   - Created a clean configuration focusing only on:
     - Kubelet feature gates
     - API Server feature gates
     - Controller Manager feature gates
     - Scheduler feature gates

2. **Verification**:
   - Confirmed the cluster was created successfully
   - All control plane and worker nodes reached Ready state
   - Configuration applied correctly to all components

## ImageVolume Feature Test Results

After successfully enabling the feature gates, we tested the ImageVolume feature:

1. **Initial Tests**:
   - Created test pods with ImageVolume configurations
   - Used various security context settings to comply with Pod Security standards
   - Tried different container images and volume configurations

2. **Consistent Error Pattern**:
   - All test pods consistently failed with the error:
     ```
     Error: failed to generate container spec: failed to apply OCI options: failed to mkdir "": mkdir : no such file or directory
     ```
   - This error occurred during container creation phase
   - The error was consistent across different test configurations

3. **Diagnostics**:
   - Examined kubelet logs which showed:
     - ImageVolume is recognized (image is pulled)
     - Container runtime (containerd) fails to create the container
     - The error occurs at mount setup time
   - Confirmed the issue is in the container runtime implementation

4. **Research Findings**:
   - The ImageVolume feature is in alpha status in Kubernetes 1.33.0
   - While the Kubernetes API supports the feature, container runtimes haven't fully implemented it
   - This is a known issue reported by users of various Kubernetes distributions
   - The feature is not yet ready for production use

## Exploring Alternative Approaches - lib-volume-populator

After discovering the limitations of the ImageVolume feature, we researched alternatives and found the lib-volume-populator library:

1. **Library Overview**:
   - Official Kubernetes CSI project for volume population
   - Provides a framework for implementing custom volume populators
   - Supports creating custom resources that can be referenced by PVCs
   - Uses a controller-based approach to manage volume population

2. **Implementation Pattern**:
   - Defines a custom resource (CR) for data sources
   - PVCs reference these custom resources using `dataSourceRef`
   - A controller watches for PVCs with references to these CRs
   - The controller creates temporary resources to populate the volumes

3. **Key Components**:
   - **Provider Function Config**: Implements three callback functions:
     - `populateFn`: Initiates the volume population process
     - `populateCompleteFn`: Checks if population has completed
     - `populateCleanupFn`: Cleans up temporary resources
   - **Volume Populator Config**: Configures the controller with resource types and namespaces
   - **Controller Pattern**: Handles Kubernetes API interactions and lifecycle management

4. **Example Implementations**:
   - **hello-populator**: Simple example that creates a text file in a PVC
   - **provider-populator**: Template for implementing provider-specific populators

## Adapting lib-volume-populator for VM Disk Images

We've outlined a plan to implement a custom volume populator for VM disk images:

1. **Custom Resource Definition**:
   - Create a `VMDiskImage` CRD with fields for source image, format, target size, etc.
   - Define validation rules for VM disk image specifications
   - Implement status fields to track the population process

2. **Populator Implementation**:
   - Create a controller using the lib-volume-populator framework
   - Implement the three key functions for population lifecycle:
     - `populateFn`: Creates a pod to pull and process the VM disk image
     - `populateCompleteFn`: Checks if the pod has completed
     - `populateCleanupFn`: Removes temporary resources
   - Use tools like skopeo and qemu-img for image acquisition and processing

3. **Deployment Model**:
   - Deploy the controller as a separate deployment in the cluster
   - Configure permissions and service accounts for the controller
   - Register the custom volume populator with Kubernetes

This approach provides a clear path forward that works with current Kubernetes versions while offering more flexibility than the built-in ImageVolume feature.

## Key Lessons Learned

1. **Alpha Feature Adoption**:
   - Testing alpha features early in the development process is critical
   - There can be significant gaps between API support and runtime implementation
   - Documentation for alpha features often lacks important implementation details

2. **Container Runtime Considerations**:
   - Features that interact with container runtimes need careful verification
   - Different container runtimes may have different levels of feature support
   - Error messages from container runtimes can be cryptic and require investigation

3. **Alternative Implementation Patterns**:
   - Kubernetes often provides multiple ways to achieve the same result
   - Custom controllers and CRDs offer flexibility when native features are lacking
   - Understanding the extension patterns in Kubernetes is valuable for implementing custom solutions

4. **Environment Setup Automation**:
   - Automated cleanup and setup processes are essential for iterative testing
   - Context and configuration management is critical for reliable testing
   - Declarative configuration helps in reproducing test environments

These experiences have provided valuable insights for our VM disk image management operator development and will inform our implementation approach moving forward.