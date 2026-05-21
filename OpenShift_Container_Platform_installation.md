**OpenShift Container Platform installation overview**
4 Methods of Installation:
-Interactive - internet based
-Local Agent-based - Airgapped
-Automated - IPI
-Full control - UPI

**About the Installation Program**
The installer creates three main Ignition files:
1. Bootstrap ignition config file
2. Master Node  " "
3. Worker Node " "

The installer creates three main Ignition files:

bootstrap.ign
master.ign
worker.ign

An Ignition file is a first-boot configuration file that tells a machine:
how to configure itself 
what services to start
how to join the cluster


The bootstrap node starts first using bootstrap.ign and temporarily runs critical services needed to initialize the cluster.

Control plane (master) nodes then boot using master.ign, form the permanent etcd cluster, and start Kubernetes control plane components.

Once the control plane becomes healthy, the bootstrap node is no longer needed and is removed, while worker nodes join using worker.ign.


**Post-installation**

RHCOS is the operating system used by OpenShift nodes.
It is based on RHEL, optimized for containers
and designed specifically for Kubernetes/OpenShift.

Every node in OpenShift usually runs RHCOS, kubelet, CRI-O.

During the first boot: Ignition configures the machine. It creates files, configures networking, adds SSH keys, starts services and configures kubelet.

After the cluster is installed, Ignition stops being used.

Now another component takes over i.e Machine Config Operator (MCO)

Machine Config Operator manages the OS updates, certificates, SSH keys, machine configs, kernel arguments.

rpm-ostree is the technology used for OS upgrades, rpm-ostree updates the OS as a single atomic image.Atomic means: Either the whole update succeeds, or nothing changes.

**Big Picture**

RHCOS
  ├── kubelet
  ├── CRI-O
  ├── Ignition
  ├── rpm-ostree
  └── SELinux

Installation:
  Ignition configures node

After installation:
  Machine Config Operator manages node

Updates:
  rpm-ostree atomic OS upgrades

Metal³ and Ironic are key components for managing physical servers (bare metal) in a cloud-native way:

Ironic: Originally from OpenStack, Ironic is a service that manages bare metal hardware. It handles tasks like powering servers on/off, installing operating systems, and configuring BIOS settings.Metal³ (Metal Cubed): This open source project brings bare metal host management to Kubernetes. It uses a Metal3-io/baremetal-operator to communicate with Ironic, allowing you to manage physical servers just like you would manage virtual machines or pods using Kubernetes custom resources.

Basically, Ironic does the low-level hardware "heavy lifting", while Metal3 provides the Kubernetes-native interface to control it.Metal3 - Controller

Ironic - ExecutorMetal3 = Kubernetes layer/controller

BareMetalHost CRD = Object used to describe and manage a physical server

OpenStack Ironic = Actually talks to the server (power on/off, PXE boot, provisioning)

Metal3 BareMetalHost (BMH) Lifecycle:

Register

↓

Inspecting

↓

Available

↓

Provisioning

↓

Provisioned

↓

Deprovisioning

↓

Available

1. Register

Create the BareMetalHost CRD.

Metal3 now knows:

Server exists

BMC details

Boot MAC

Credentials

"Metal3 discovered a physical server and learned how to access/control it using Redfish + BMC credentials."

2. Inspecting

OpenStack Ironic boots a small inspection image and collects:

CPU

RAM

Disk

NICs

After registration, OpenStack Ironic enters the Inspecting phase.

"I know the server exists.

Now let me check what hardware is inside it."

Inspecting phase = "Ironic scans and learns the server hardware details."

3. Available

Server is ready and waiting for provisioning.

provisioning:

state: available

Ironic successfully inspected the serverand says:"This machine is ready for provisioning."

"I checked the server hardware.Everything looks good.Now the server is ready for OS installation."

4. Provisioning

Ironic:

Sets PXE boot

Writes OS image

Configures disk

Reboots node

5. Provisioned

The OS is installed successfully.

OS installation succeeded But cluster bootstrap/join still pending OS installation completedANDserver rebooted into installed OS successfully

6. Deprovisioning

When deleting cluster/node:

Disk cleanup

Remove image

Reset boot order

7. Back to Available

Machine becomes reusable for another cluster.

Day-0 bare metal provisioning flow in OpenShift ZTP.

SiteConfig

↓

BareMetalHost created

↓

Metal3 watches BMH

↓

Ironic provisions node

↓

Agent joins cluster

↓

SNO/Cluster becomes Ready

Metal3/Ironic↓OS installed↓BMH = provisioned--------------------------------Now Kubernetes/OpenShift starts--------------------------------↓kubelet starts↓Node joins cluster↓Ready

Booting happens in Provisoning state in Bmh