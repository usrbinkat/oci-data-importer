# Kubernetes Operator Design for VM Disk Image Management - Research and Discovery Phase

## Project Objective

Design a Kubernetes operator that:
1. ~~Leverages the new image volume feature in Kubernetes 1.31+ as a source~~ Uses the lib-volume-populator to implement a custom solution for VM disk images
2. Uses volume populator libraries to handle disk image operations
3. Takes size-optimized `.raw` VM disk images from OCI registries
4. Copies disk content into a PVC (without direct image volume mounting in VM pods)
5. Resizes the disk images according to specifications
6. Prepares the PVC for use by VM launcher pods

## Progress Tracking

### Environment Setup
- [x] Kubernetes 1.31+ cluster with ImageVolume feature gate enabled
- [ ] Access to an OCI registry for hosting VM disk images
- [ ] Development tools and dependencies installed
- [ ] GitHub API credentials configured

### Research Progress
- [~] Volume Populators Research (60%)
- [x] Image Volumes Research (100%)
- [ ] Volume Data Source Validator Research (0%)
- [ ] VM Disk Image Management Research (0%)

### Discovery Progress
- [~] Proof-of-Concept Testing (50%)
- [~] Architecture Exploration (25%)
- [ ] API Design Research (0%)
- [ ] Implementation Research (0%)

### Deliverables
- [~] Technical Feasibility Report (25%)
- [ ] Architecture Design Document (0%)
- [ ] Implementation Plan (0%)

## Key Kubernetes Features to Leverage

### ~~Image Volumes (as Source Only)~~ Custom Volume Populator Implementation

Due to implementation limitations in current container runtimes with the ImageVolume feature, we'll implement a custom volume populator using the lib-volume-populator library:

- **Status**: Discovered that the ImageVolume feature is not fully implemented in container runtimes
- **Alternative Approach**: Create a custom volume populator using the lib-volume-populator library
- **Implementation Pattern**: Use Kubernetes' volume population mechanism which allows PVCs to reference custom resources

### Volume Populators

The lib-volume-populator provides a framework for implementing custom volume populators:

- **Controller Framework**: Provides a controller that watches for PVCs referencing specific data sources
- **Provider Function Pattern**: Implements three key functions:
  - `populateFn` - Initiates the volume population process
  - `populateCompleteFn` - Checks if population is complete
  - `populateCleanupFn` - Cleans up temporary resources
- **Custom Resource Integration**: Creates a custom resource for VM disk images that can be referenced by PVCs

## Revised Research Plan

### 1. Volume Populators Research

**Objective**: Understand the Kubernetes volume populator library and how to implement a custom VM disk image populator.

**Research Activities**:
- [x] Study the Kubernetes volume populator library (https://github.com/kubernetes-csi/lib-volume-populator)
- [~] Analyze the example hello-populator and provider-populator implementations
- [~] Understand the required interfaces and APIs
- [ ] Design a pattern for copying VM disk images from OCI registries to PVCs

**Key Questions to Answer**:
- [x] How does the volume populator library handle different data sources?
- [~] What is the lifecycle of a volume population operation?
- [ ] What are the security considerations and permissions required?
- [ ] How can we extend the library for VM disk image scenarios?

### 2. Image Acquisition Research (replacing original Image Volumes Research)

**Objective**: Determine the best approach for acquiring images from OCI registries using the volume populator pattern.

**Research Activities**:
- [x] Review the Kubernetes image volumes documentation
- [x] Test actual implementation status in Kubernetes 1.33.0 (found limitations)
- [~] Evaluate alternative methods for OCI image acquisition
- [ ] Design a reliable process for image downloading and extraction

**Key Questions to Answer**:
- [x] Is the ImageVolume feature fully implemented in current container runtimes? (No)
- [~] What are the best alternatives for accessing OCI content from operator pods?
- [ ] How can we securely download and extract images from OCI registries?
- [ ] What are the performance implications of different approaches?

### 3. Volume Data Source Validator Research

**Objective**: Understand how volume data sources are validated in Kubernetes.

**Research Activities**:
- [ ] Study the volume-data-source-validator (https://github.com/kubernetes-csi/volume-data-source-validator)
- [ ] Understand how it validates volume data sources
- [ ] Determine how to integrate with validation mechanisms

**Key Questions to Answer**:
- [ ] How does Kubernetes validate data sources for volumes?
- [ ] What validation is needed for custom volume populators?
- [ ] How does validation interact with the volume population process?

### 4. VM Disk Image Management Research

**Objective**: Understand best practices for VM disk image manipulation in containers.

**Research Activities**:
- [ ] Research best practices for managing and resizing `.raw` VM disk images
- [ ] Identify tools and libraries for disk manipulation within Kubernetes pods
- [ ] Study formats, conversion options, and optimization techniques

**Key Questions to Answer**:
- [ ] What tools are best suited for containerized disk image manipulation?
- [ ] How can disk images be resized efficiently and safely?
- [ ] What format conversion options should be supported?
- [ ] What are the performance and resource implications of disk operations?

## Discovery Activities

### 1. Proof-of-Concept Testing

**Objective**: Validate key technical assumptions with minimal working examples.

**Activities**:
- [x] Set up a Kubernetes 1.33.0 environment with ImageVolume feature gate enabled
- [x] Test ImageVolume implementation (identified limitations)
- [~] Explore volume populator implementation examples
- [ ] Create test OCI images containing sample disk images
- [ ] Create a simple custom volume populator for testing

**Expected Deliverables**:
- [~] Technical feasibility assessment
- [ ] Performance benchmarks for key operations
- [~] Identification of potential issues and limitations

### 2. Architecture Exploration

**Objective**: Define the high-level architecture of the operator.

**Activities**:
- [~] Define the components of the operator using the volume populator pattern
- [ ] Create a high-level diagram showing interaction between components
- [~] Outline the CRDs (Custom Resource Definitions) needed for VM disk images
- [ ] Design a multi-stage workflow for image preparation

**Expected Deliverables**:
- [ ] Architecture diagrams
- [ ] Component descriptions
- [ ] Workflow sequences
- [ ] Technical dependencies and requirements

### 3. API Design Research

**Objective**: Design user-friendly and effective API for the operator.

**Activities**:
- [ ] Research similar Kubernetes CRDs for inspiration
- [ ] Design VMDiskImage CRD with fields for:
  - [ ] Source image reference (OCI registry path)
  - [ ] Target PVC details
  - [ ] Resize specifications
  - [ ] Processing options (optimization, format conversion)
- [ ] Create example resources and validate their usability

**Expected Deliverables**:
- [ ] Draft CRD schemas
- [ ] API design documentation
- [ ] Example resource definitions
- [ ] Validation rules

### 4. Implementation Research

**Objective**: Research implementation approaches and technologies.

**Activities**:
- [ ] Evaluate lib-volume-populator as the core for implementation
- [ ] Research container image requirements for disk operations (skopeo, qemu-img)
- [ ] Determine security implications and permissions required
- [ ] Identify best practices for error handling and recovery

**Expected Deliverables**:
- [ ] Implementation recommendations
- [ ] Technology stack decisions
- [ ] Security considerations
- [ ] Error handling strategies

## Example Custom Resource Definition (CRD) for VM Disk Images

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: vmdiskimages.vm.example.com
spec:
  group: vm.example.com
  names:
    kind: VMDiskImage
    plural: vmdiskimages
    singular: vmdiskimage
    shortNames:
      - vdi
  scope: Namespaced
  versions:
    - name: v1alpha1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["sourceImage", "targetPVC", "targetSize"]
              properties:
                sourceImage:
                  type: string
                  description: "OCI image reference for the VM disk image"
                pullPolicy:
                  type: string
                  enum: ["Always", "IfNotPresent", "Never"]
                  default: "IfNotPresent"
                targetPVC:
                  type: object
                  required: ["name"]
                  properties:
                    name:
                      type: string
                      description: "Name of the target PVC"
                    createIfNotExist:
                      type: boolean
                      default: false
                    storageClass:
                      type: string
                      description: "Storage class for the PVC if it needs to be created"
                targetSize:
                  type: string
                  description: "Target size for the resized disk (e.g., '20Gi')"
                format:
                  type: string
                  enum: ["raw", "qcow2", "vmdk"]
                  default: "raw"
            status:
              type: object
              properties:
                phase:
                  type: string
                  enum: ["Pending", "ImageFetching", "Copying", "Resizing", "Ready", "Failed"]
                message:
                  type: string
                  description: "Detailed message about the current status"
```

## Volume Populator Implementation Approach

Based on the provider-populator example from lib-volume-populator, our implementation would:

```go
func populateFn(ctx context.Context, params populatorMachinery.PopulatorParams) error {
    // 1. Extract VMDiskImage details from params
    var diskImage VMDiskImage
    err := runtime.DefaultUnstructuredConverter.FromUnstructured(params.DataSource.UnstructuredContent(), &diskImage)
    if err != nil {
        return err
    }

    // 2. Create a job to:
    //    - Pull the container image from OCI registry
    //    - Extract the disk image
    //    - Resize/convert it as needed
    //    - Copy to the target PVC
    
    pod := &corev1.Pod{
        // Pod definition for disk image population
        // This pod will mount the target PVC and perform all needed operations
    }
    
    return params.KubeClient.Create(ctx, pod)
}

func populateCompleteFn(ctx context.Context, params populatorMachinery.PopulatorParams) (bool, error) {
    // Check if population job/pod has completed
    // Return true when complete, false if still in progress
}

func populateCleanupFn(ctx context.Context, params populatorMachinery.PopulatorParams) error {
    // Clean up any temporary resources after successful population
}
```

## Research and Discovery Deliverables

1. **Technical Feasibility Report**
   - [~] Assessment of the feasibility of using volume populators for VM disk images
   - [ ] Evaluation of performance, reliability, and limitations
   - [ ] Recommendations on technical approaches

2. **Architecture Design Document**
   - [ ] Detailed architecture of the proposed operator
   - [ ] Component descriptions and relationships
   - [ ] Workflow diagrams and sequences
   - [ ] API design and examples

3. **Implementation Plan**
   - [ ] Detailed plan for implementing the operator
   - [ ] Technology stack and dependencies
   - [ ] Resource requirements and timeline estimates
   - [ ] Risk assessment and mitigation strategies

## Tools and MCP Servers to Utilize

- **github-official**: To interact with GitHub repositories and fetch code examples
- **kubernetes**: To validate and test CRDs and operator components
- **openapi-schema**: To validate the schemas for the CRDs
- **perplexity-ask**: For additional research on specific technical challenges
- **sequentialthinking**: To work through complex design problems
- **fetch**: To retrieve documentation and examples from external sources

## Next Steps - Research Phase

1. [x] Set up a development environment with Kubernetes 1.31+ and the ImageVolume feature gate enabled
2. [x] Begin research on volume populator library and image volumes
3. [~] Create initial proof-of-concept tests to validate key assumptions
4. [~] Document findings and refine the technical approach
5. [ ] Develop architectural recommendations based on research results
6. [ ] Implement a simple POC using the lib-volume-populator library

<!-- 
INSTRUCTIONS FOR CONTINUATION:
After completing the volume populator research, continue this project by:
1. Creating a simple POC implementation based on the provider-populator example
2. Testing the POC with a sample VM disk image
3. Documenting the implementation details and challenges
4. Updating the plan with more specific implementation details

PROGRESS TRACKING:
As you work through this plan, maintain progress tracking by updating the checkboxes:
- When starting a task, update its status from [ ] to [~] (in progress)
- When completing a task, update its status from [~] to [x] (completed)
- For percentage-based tracking items, update the percentage (e.g., "Volume Populators Research (50%)")

This progress tracking will help maintain continuity across sessions and provide clear visibility into the current state of the project.
-->