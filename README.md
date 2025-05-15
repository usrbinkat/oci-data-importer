# OCI Data Importer

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
![Project Status](https://img.shields.io/badge/Status-Pre--Alpha-red)

A Kubernetes operator for importing OCI container images as persistent volume data, focused on VM disk image management.

## Status: Pre-Alpha

> **Warning**: This project is in pre-alpha stage. APIs, behavior, and implementation are subject to significant change without notice. Not recommended for production use.

## Overview

OCI Data Importer bridges the gap between OCI container registries and Kubernetes persistent storage. It enables platform operators to treat container registries as a distribution mechanism for VM disk images and other large data files by:

1. Managing the lifecycle of VM disk images as Kubernetes Custom Resources (CRs)
2. Automatically importing container images to persistent volumes on demand
3. Implementing format conversion and preparation for consumption by VM orchestrators

## Architecture

OCI Data Importer builds upon emerging Kubernetes features like the alpha `ImageVolume` feature and implements a custom [Kubernetes Volume Populator](https://github.com/kubernetes-csi/lib-volume-populator) for compatibility with current stable Kubernetes versions.

### Components

- **OCI Data Importer Controller**: Watches for VMDiskImage CRs and orchestrates import operations
- **Volume Populator**: Implements the Kubernetes volume populator pattern for persisting image data
- **Helper Tools**: Utilizes `skopeo` and `qemu-img` for image manipulation

## Motivation

### Challenges with VM Disk Images in Kubernetes

Platform teams managing VM workloads in Kubernetes face significant challenges:

- VM images are large binary files that don't easily fit into existing Kubernetes patterns
- Distributing VM images through container registries has benefits but lacks standardized extraction
- Existing solutions often require proprietary tooling or complex integrations

### The OCI Data Importer Approach

This project addresses these challenges by:

1. Leveraging container registries' established security and distribution mechanisms
2. Providing a declarative Kubernetes-native API for image management
3. Automating the import process from registry to persistent volume
4. Supporting conversion between VM disk image formats
5. Integrating with the CSI ecosystem

## Features

- [x] Project planning and architecture definition
- [x] Initial exploration of Kubernetes `ImageVolume` feature
- [ ] VMDiskImage Custom Resource Definition
- [ ] Custom volume populator implementation
- [ ] OCI image extraction and conversion
- [ ] Integration with popular VM orchestration platforms
- [ ] Comprehensive documentation and examples

## Installation

> Coming soon. This project is under active development.

## Usage

### Example VMDiskImage Resource

```yaml
apiVersion: virtualization.k8s.io/v1alpha1
kind: VMDiskImage
metadata:
  name: ubuntu-22.04
spec:
  source:
    imageRef: "docker.io/library/ubuntu:22.04"
  format: qcow2
  size: 20Gi
```

### Example PVC Referencing VMDiskImage

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vm-disk
  annotations:
    virtualization.k8s.io/disk-importer: "ubuntu-22.04"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

## Development Journey

This project began by exploring the alpha `ImageVolume` feature introduced in Kubernetes 1.31+. During testing, we discovered that while the API for this feature exists, the implementation in container runtimes is incomplete, resulting in mount failures.

As a pragmatic alternative, we pivoted to using the [lib-volume-populator](https://github.com/kubernetes-csi/lib-volume-populator) approach, which provides a stable framework for implementing custom volume population while positioning the project to adopt `ImageVolume` when it matures.

## Getting Started with Development

```bash
# Clone the repository
git clone https://github.com/usrbinkat/oci-data-importer.git
cd oci-data-importer

# Check available commands
task --list

# Create a test Kubernetes cluster with ImageVolume feature enabled
task cluster:create

# Run test pods with ImageVolume
task create-test-pod

# Test the ImageVolume feature gate end-to-end
task test-feature-gate

# Clean up the cluster and configurations
task clean:all

# Check versions of tools
task version:check
```

## Comparison with Other Approaches

| Approach | Pros | Cons |
|----------|------|------|
| **OCI Data Importer** | Kubernetes-native, registry integration, flexible | Pre-alpha status |
| **Manual Image Download** | Simple, works everywhere | No automation, security concerns |
| **CSI Drivers** | Production-ready, vendor support | Often vendor-specific, limited formats |
| **ImageVolume** | Future Kubernetes standard | Alpha-only, incomplete runtime support |

## Community and Contribution

This project aims to follow the [CNCF Code of Conduct](https://github.com/cncf/foundation/blob/master/code-of-conduct.md).

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for details.

## License

Apache 2.0 - See [LICENSE](LICENSE) for more information.