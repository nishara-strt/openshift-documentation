# OpenShift Container Platform

## Technical Reference Guide: Installation & Node Lifecycle

### Covers

* Installation Methods
* Ignition
* RHCOS
* Machine Config Operator (MCO)
* Metal3
* OpenStack Ironic
* Zero Touch Provisioning (ZTP)

---

# 1. OpenShift Container Platform — Installation Overview

OpenShift Container Platform (OCP) provides multiple installation methods designed for different infrastructure environments, automation levels, and operational requirements.

| Method            | Type                 | Description                                                                                                                                                                        |
| ----------------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Interactive (IPI) | Internet-Based       | Installer-Provisioned Infrastructure. Fully automated installation where the installer creates and manages infrastructure resources. Best suited for cloud-connected environments. |
| Agent-Based       | Air-Gapped / Offline | Used for disconnected or restricted environments. A local agent handles node discovery and cluster bootstrap.                                                                      |
| Automated (IPI)   | Automated            | Fully automated deployment using predefined infrastructure definitions with minimal manual intervention.                                                                           |
| UPI               | Full Control         | User-Provisioned Infrastructure. Administrators manually provision and manage all infrastructure components. Provides maximum flexibility and control.                             |

## Installer-Provisioned Infrastructure (IPI)

IPI is the recommended installation method for most cloud-connected environments. The OpenShift installer provisions:

* Compute resources
* Networking
* Storage
* Bootstrap infrastructure

This minimizes manual setup and simplifies cluster deployment.

## User-Provisioned Infrastructure (UPI)

UPI is commonly used in:

* On-premises data centers
* Highly regulated environments
* Custom networking/storage environments

Administrators retain full control over:

* Servers
* Networking
* Load balancers
* DNS
* Storage systems

---

# 2. The Installation Program & Ignition Configuration

## 2.1 Role of the Installation Program

The `openshift-install` program generates all configuration artifacts required to bootstrap and initialize the cluster.

Its primary outputs are three Ignition configuration files.

## What is Ignition?

Ignition is a first-boot provisioning utility used by Red Hat CoreOS (RHCOS).

It runs exactly once during the first boot of a node and:

* Reads a JSON-based configuration file
* Configures the operating system
* Sets up networking
* Configures services
* Prepares the node for cluster membership

Ignition executes before the operating system reaches its normal runtime state.

---

## 2.2 The Three Ignition Configuration Files

| File            | Target Node         | Purpose                                                                                        |
| --------------- | ------------------- | ---------------------------------------------------------------------------------------------- |
| `bootstrap.ign` | Bootstrap Node      | Configures the temporary bootstrap node used to initialize the cluster.                        |
| `master.ign`    | Control Plane Nodes | Configures permanent control plane nodes running etcd and Kubernetes control plane components. |
| `worker.ign`    | Worker Nodes        | Configures worker nodes that run application workloads.                                        |

---

## 2.3 What Ignition Configures on First Boot

During first boot, Ignition performs:

* Filesystem configuration and partitioning
* Network configuration
* SSH authorized key injection
* Systemd unit creation
* kubelet configuration
* TLS certificate injection

### Important Note

Ignition runs only once. After first boot, ongoing node management is handled by the Machine Config Operator (MCO).

---

# 3. Cluster Bootstrap Sequence

The OpenShift installation process follows a strict bootstrap sequence.

---

## 3.1 Stage 1 — Bootstrap Node Initialization

The bootstrap node:

* Boots using `bootstrap.ign`
* Runs temporary cluster services:

  * Temporary etcd
  * Temporary Kubernetes API server
  * Machine Config Server

Its only purpose is to initialize the permanent control plane.

---

## 3.2 Stage 2 — Control Plane Formation

Control plane nodes boot using `master.ign`.

They:

* Contact the bootstrap Machine Config Server
* Download their complete configuration
* Form the permanent etcd cluster
* Start Kubernetes control plane services

Once etcd quorum is established, the permanent API server becomes operational.

---

## 3.3 Stage 3 — Bootstrap Removal & Worker Join

After the control plane becomes healthy:

* The bootstrap node is removed
* Worker nodes boot using `worker.ign`
* Workers contact the permanent API server
* Nodes join the cluster as compute nodes

---

## Bootstrap Flow Diagram

```text
Bootstrap Node (bootstrap.ign)
  └── Runs temporary etcd + API server
       └── Control Plane Nodes (master.ign)
              └── Permanent etcd quorum established
                     └── Bootstrap node removed
                            └── Worker Nodes (worker.ign) join cluster
```

---

# 4. Red Hat CoreOS (RHCOS) & Post-Installation Management

## 4.1 What is RHCOS?

Red Hat CoreOS (RHCOS) is the immutable, container-optimized operating system used by every OpenShift node.

It is:

* Derived from RHEL
* Optimized specifically for Kubernetes workloads
* Designed for consistency, security, and automation

Every OpenShift node runs:

* RHCOS
* kubelet
* CRI-O

---

## Core Components on an OpenShift Node

| Component  | Purpose                                 |
| ---------- | --------------------------------------- |
| RHCOS      | Immutable operating system              |
| kubelet    | Kubernetes node agent                   |
| CRI-O      | Container runtime                       |
| Ignition   | First-boot configuration utility        |
| rpm-ostree | Atomic OS image management              |
| SELinux    | Mandatory access control security layer |

---

## Why Immutable?

RHCOS is immutable, meaning:

* The OS filesystem is read-only during normal operation
* Direct manual OS modifications are discouraged
* Configuration changes are centrally managed

This ensures:

* Consistency
* Auditability
* Predictable upgrades
* Reduced configuration drift

---

## 4.2 The Machine Config Operator (MCO)

After installation, the Machine Config Operator (MCO) becomes responsible for node configuration management.

The MCO manages:

* OS upgrades
* SSH keys
* TLS certificates
* Kernel arguments
* MachineConfig CRDs

Configuration changes are rolled out safely across nodes.

---

## 4.3 rpm-ostree: Atomic OS Upgrades

RHCOS uses `rpm-ostree` for image-based operating system updates.

Unlike traditional package updates:

* The entire OS is treated as a single versioned image
* Updates are atomic
* Either the full update succeeds or the system rolls back safely

This prevents partial upgrade failures.

---

## RHCOS Architecture Overview

```text
RHCOS Node
  ├── kubelet
  ├── CRI-O
  ├── Ignition
  ├── rpm-ostree
  └── SELinux
```

---

## 4.4 Configuration Lifecycle Summary

| Phase             | Responsible Component   | Action                                                |
| ----------------- | ----------------------- | ----------------------------------------------------- |
| First Boot        | Ignition                | Configures OS, networking, SSH, kubelet, and services |
| Post-Installation | Machine Config Operator | Manages ongoing configuration changes                 |
| OS Updates        | rpm-ostree              | Performs atomic OS upgrades                           |

---

# 5. Bare Metal Provisioning: Metal3 & OpenStack Ironic

## 5.1 Why Bare Metal Requires Special Handling

Unlike cloud environments, bare metal servers require:

* Power management
* PXE boot configuration
* BIOS/BMC interaction
* Disk imaging

Metal3 and OpenStack Ironic automate this process.

---

## Simple Analogy

* **Ironic** = Physical technician working directly with servers
* **Metal3** = Kubernetes-native manager issuing provisioning instructions

Metal3 communicates with Kubernetes.
Ironic communicates with hardware.

---

## 5.2 Component Roles

| Component         | Role                  | Responsibility                                                    |
| ----------------- | --------------------- | ----------------------------------------------------------------- |
| Metal3            | Kubernetes Controller | Watches BareMetalHost CRDs and manages provisioning workflows     |
| BareMetalHost CRD | Kubernetes Object     | Represents a physical server                                      |
| OpenStack Ironic  | Hardware Executor     | Handles PXE boot, power control, BIOS config, and OS installation |

---

## 5.3 BareMetalHost (BMH) Lifecycle

| Phase          | Description                     |
| -------------- | ------------------------------- |
| Register       | BareMetalHost CRD is created    |
| Inspecting     | Hardware inventory collection   |
| Available      | Server ready for provisioning   |
| Provisioning   | PXE boot and RHCOS installation |
| Provisioned    | OS installed and booted         |
| Deprovisioning | Disk wipe and cleanup           |
| Available      | Server reusable again           |

---

## Important Detail — When Does Booting Happen?

The actual operating system installation and first boot occur during the **Provisioning** phase.

After the node reaches the **Provisioned** state:

* RHCOS boots from local disk
* kubelet starts
* The node begins joining the OpenShift cluster

---

# 6. Zero Touch Provisioning (ZTP)

## 6.1 What is ZTP?

Zero Touch Provisioning (ZTP) automates cluster deployment using GitOps-driven declarative manifests.

It is heavily used for:

* Single Node OpenShift (SNO)
* Edge deployments
* Remote spoke clusters

---

# Deprecation Notice

| CR | Status | Availability | Notes |
|---|---|---|
| SiteConfig | Deprecated | Deprecated in OCP 4.18 | Planned for future removal |
| ClusterInstance | Recommended | RHACM 2.12+ | Replacement for SiteConfig |

---

## 6.2 End-to-End ZTP Flow

### Step-by-Step Flow

1. ClusterInstance CR created in Git
2. BareMetalHost manifests generated
3. Metal3 detects BareMetalHost objects
4. Ironic provisions servers
5. Nodes boot into RHCOS
6. kubelet joins cluster
7. Cluster reaches Ready state

---

## ZTP Flow Diagram

```text
ClusterInstance / SiteConfig
        ↓
BareMetalHost CRD
        ↓
Metal3 watches BMH
        ↓
Ironic:
Inspect → PXE Boot → Write RHCOS Image
        ↓
Node boots into RHCOS
        ↓
kubelet starts
        ↓
Node registers with API server
        ↓
Cluster becomes Ready
```

---

## 6.3 Post-Provisioning Flow

| Step | Event                    | Details                                      |
| ---- | ------------------------ | -------------------------------------------- |
| 1    | OS Installation Complete | RHCOS written to disk                        |
| 2    | Node Reboot              | Boots from local disk                        |
| 3    | Ignition Runs            | Configures kubelet, certificates, networking |
| 4    | kubelet Starts           | Connects to API server                       |
| 5    | Node Joins Cluster       | CSR approved                                 |
| 6    | Node Ready               | All required pods operational                |

---

# 7. Summary

OpenShift combines multiple integrated technologies to deliver automated Kubernetes infrastructure across cloud, virtualized, and bare metal environments.

## Key Concepts

* Multiple installation methods support different deployment models
* Ignition handles one-time node initialization
* Bootstrap nodes initialize the permanent control plane
* RHCOS provides an immutable Kubernetes-optimized OS
* The Machine Config Operator manages ongoing node configuration
* rpm-ostree enables atomic OS upgrades
* Metal3 provides Kubernetes-native bare metal management
* OpenStack Ironic performs low-level hardware provisioning
* Zero Touch Provisioning automates edge and large-scale deployments using GitOps principles
