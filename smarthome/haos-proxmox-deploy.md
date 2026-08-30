# Home Assistant OS 18.2 Deployment on Proxmox VE

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the deployment of Home Assistant OS (HAOS) 18.2 as a virtual machine on Proxmox VE, including VM creation from an existing template, HAOS QCOW2 image preparation and import, UEFI configuration, storage attachment, boot configuration, static network configuration, initial HAOS startup, and restoration of an existing Home Assistant backup.

This sanitized version preserves the deployment procedure while replacing or removing production network addresses, hostnames, bridge names, VM identifiers, MAC addresses, storage identifiers, administrative tags, infrastructure paths, and other environment-specific information that should not be disclosed publicly. The deployment follows established cybersecurity frameworks and best practices, including NIST SP 800-171 concepts, through network segmentation, controlled administrative access, and separation of infrastructure roles.

All infrastructure identifiers and network addresses shown below are documentation examples and do not represent the production environment.

---

## Purpose

The purpose of this deployment is to migrate Home Assistant from a Docker-based Home Assistant Container installation to a dedicated Home Assistant OS virtual machine.

The migration provides:

* A dedicated Home Assistant appliance running under Proxmox VE.
* Supervisor-managed Home Assistant services and applications.
* Support for Home Assistant applications such as Matter Server.
* Separation between Home Assistant services and general-purpose Docker workloads.
* Proxmox-level VM management, backup, and recovery.
* Home Assistant-level backup and restore capabilities.
* Static network addressing appropriate for an infrastructure network.
* A rollback path through retention of the previous Docker deployment.

This document ends at restoration of the existing Home Assistant backup. Matter configuration, SSH application configuration, IoT network integration, and post-migration service validation are outside the completed scope of this procedure.

**Next Step:** Review the recorded environment and prerequisites before reproducing the deployment.

---

## Sanitization Reference

The following documentation-only values are used throughout this version.

| Purpose                     | Documentation Value |
| --------------------------- | ------------------- |
| Proxmox host                | `pve-host01`        |
| HAOS VM ID                  | `200`               |
| HAOS VM name                | `vm-haos`           |
| Administrative bridge       | `vmbr-admin`        |
| Home Assistant IPv4 network | `192.0.2.0/24`      |
| HAOS IPv4 address           | `192.0.2.54/24`     |
| Default gateway             | `192.0.2.1`         |
| DNS resolver                | `192.0.2.53`        |
| VM storage                  | `vm-storage`        |
| HAOS image directory        | `/root/images/`     |

The following values are intentionally removed or generalized:

```text
Production IPv4 addresses
Production IPv6 addresses
Proxmox node names
Production bridge names
Production storage IDs
VM-specific MAC addresses
Administrative tags
Internal infrastructure naming
Backup storage identifiers
Actual gateway addresses
Actual DNS resolver addresses
Environment-specific VM IDs
```

Validation Result: The procedure remains reproducible without exposing production infrastructure details.

---

## Deployment Environment

The following values describe the sanitized deployment model.

| Component           | Configuration                      | Notes                              |
| ------------------- | ---------------------------------- | ---------------------------------- |
| Hypervisor          | Proxmox VE                         | Existing virtualization platform   |
| Proxmox host        | `pve-host01`                       | Documentation-only hostname        |
| HAOS version        | 18.2                               | KVM/Proxmox QCOW2 image            |
| Home Assistant Core | 2026.8.2                           | Version observed during deployment |
| VM ID               | `200`                              | Documentation-only VM identifier   |
| VM name             | `vm-haos`                          | Proxmox VM name                    |
| VM CPU              | 6 vCPU                             | CPU type configured as `host`      |
| VM memory           | 4000 MB                            | Approximately 4 GB                 |
| VM disk             | 32 GB                              | Imported HAOS disk                 |
| Disk bus            | SCSI 0                             | Attached using VirtIO SCSI         |
| SCSI controller     | VirtIO SCSI                        | `virtio-scsi-pci`                  |
| BIOS                | OVMF (UEFI)                        | Used by the HAOS VM                |
| Network adapter     | VirtIO                             | Virtual NIC                        |
| Proxmox bridge      | `vmbr-admin`                       | Documentation-only bridge          |
| HAOS interface      | `enp0s18`                          | Detected inside HAOS               |
| HAOS IPv4           | `192.0.2.54/24`                    | Documentation-only static address  |
| VM storage          | `vm-storage`                       | Documentation-only storage ID      |
| HAOS image path     | `/root/images/haos_ova-18.2.qcow2` | Sanitized image path               |

Example VM description:

```text
vm-haos

OS: HAOS

IP: 192.0.2.54/24

Home Assistant OS

Deployment Date: 2026-08-20
```

**Validation Result:** VM identity, storage, CPU, memory, virtual networking, HAOS version, and static addressing requirements are documented without exposing production infrastructure.

---

## Architecture

The resulting architecture is:

```text
Virtualization Host
|
+-- Proxmox VE
    |
    +-- VM 200 - vm-haos
        |
        +-- Home Assistant OS 18.2
            |
            +-- Home Assistant Supervisor
            |
            +-- Home Assistant Core 2026.8.2
            |
            +-- Supervisor-managed applications
```

HAOS is treated as a dedicated Home Assistant appliance rather than a general-purpose Linux or Docker server.

General-purpose containers and unrelated infrastructure services remain outside HAOS.

**Next Step:** Obtain the HAOS KVM/Proxmox disk image.

---

# HAOS Installation Procedure

## 1. Download the HAOS Proxmox Image

For Proxmox VE, select:

```text
KVM/Proxmox (.qcow2)
```

The deployment used:

```text
haos_ova-18.2.qcow2.xz
```

Store the compressed image in an administrative staging directory.

Sanitized example:

```text
/root/images/haos_ova-18.2.qcow2.xz
```

The directory initially contains:

```text
/root/images/
`-- haos_ova-18.2.qcow2.xz
```

### Why QCOW2 Is Used

HAOS provides a prebuilt virtual disk for KVM/Proxmox.

This is not an ISO-based operating system installation.

The downloaded QCOW2 file contains the HAOS virtual disk and is imported directly into Proxmox.

Do not create an empty virtual disk and attempt to install HAOS onto it from the QCOW2 image.

**Validation Result:** The compressed HAOS QCOW2 image is present on the Proxmox host.

---

## 2. Clone the Existing Proxmox VM Template

An existing Linux VM template can be used as the starting VM shell.

The source template itself must not be modified.

Sanitized example:

```text
VM ID: 200
Name:  vm-haos
```

### Clone Mode

Use:

```text
Mode: Full Clone
```

A linked clone is unnecessary when the inherited operating system disk will not be retained.

The cloned VM provides an existing virtual hardware configuration.

```text
Existing VM Template
        |
        +-- Full Clone
              |
              +-- VM 200
                  |
                  +-- vm-haos
```

**Important:** Do not modify or remove disks from the original template.

All HAOS-specific modifications occur on the cloned VM.

**Validation Result:** The new HAOS VM exists independently of the source template.

---

## 3. Remove the Cloned Operating System Disk

The cloned VM initially inherits the source template's operating system disk.

That disk is not used by HAOS.

Remove the inherited operating system disk from the cloned VM before attaching the imported HAOS image.

The objective is:

```text
VM shell
+
required virtual hardware
+
no inherited operating system boot disk
```

Do not manually replace template disk files at the filesystem level.

Proxmox should manage the imported HAOS image through normal VM and storage mechanisms.

**Validation Result:** The inherited operating system disk is no longer used as the HAOS VM boot disk.

---

## 4. Decompress the HAOS Image

Change to the sanitized image directory:

```bash
cd /root/images
```

Decompress:

```bash
unxz haos_ova-18.2.qcow2.xz
```

Result:

```text
/root/images/haos_ova-18.2.qcow2
```

The image is approximately 1 GB before being expanded into its destination virtual disk.

The directory now contains:

```text
/root/images/
`-- haos_ova-18.2.qcow2
```

### Verify

```bash
ls -lh /root/images/haos_ova-18.2.qcow2
```

The `.xz` extension should no longer be present.

**Validation Result:** The HAOS QCOW2 disk image is successfully decompressed.

---

## 5. Identify the Proxmox VM Storage

Do not assume the Proxmox storage identifier.

Run:

```bash
pvesm status
```

Identify an active storage target suitable for VM disks.

Sanitized example:

```text
vm-storage
```

### Why This Check Matters

Proxmox commands use storage IDs rather than arbitrary filesystem paths.

For example:

```text
vm-storage
```

is the logical storage identifier supplied to:

```text
qm importdisk
```

**Validation Result:** The destination storage is active before the disk import begins.

---

## 6. Import the HAOS QCOW2 Disk

Sanitized command:

```bash
qm importdisk 200 /root/images/haos_ova-18.2.qcow2 vm-storage
```

The operation is:

```text
HAOS QCOW2 image
/root/images/haos_ova-18.2.qcow2
             |
             v
        qm importdisk
             |
             v
VM 200 -> vm-storage
```

After import, Proxmox associates the disk with the VM as an unused disk.

Example sanitized configuration:

```text
unused0: vm-storage:200/vm-200-disk-0.raw
```

### Verify

```bash
qm config 200
```

The imported disk should appear as unused until explicitly attached.

The `.raw` representation may be expected depending on the backing storage configuration.

**Validation Result:** The HAOS disk image is imported and associated with the VM.

---

## 7. Attach the Imported HAOS Disk

In the Proxmox web interface:

```text
VM 200
-> Hardware
-> Unused Disk 0
-> Edit
```

Attach using:

```text
Bus/Device:       SCSI 0
SCSI Controller:  VirtIO SCSI
Cache:            Default
Discard:          Off
IO thread:        Off
SSD emulation:    Off
Read-only:        Off
Backup:           On
Async IO:         Default
```

Then select:

```text
Add
```

The resulting sanitized configuration resembles:

```text
scsi0: vm-storage:200/vm-200-disk-0.raw,size=32G
scsihw: virtio-scsi-pci
```

### Disk Layout

```text
VM 200 - vm-haos
|
+-- VirtIO SCSI Controller
    |
    +-- scsi0
        |
        +-- vm-storage:200/vm-200-disk-0.raw
            |
            +-- 32 GB HAOS disk
```

### Verify

```bash
qm config 200
```

Confirm that the HAOS disk appears as `scsi0`.

**Validation Result:** The HAOS disk is attached as `scsi0` using VirtIO SCSI.

---

## 8. Change the VM Firmware to OVMF UEFI

If the cloned VM uses SeaBIOS, change it to OVMF UEFI.

In Proxmox:

```text
VM 200
-> Hardware
-> BIOS
-> Edit
```

Change:

```text
SeaBIOS
```

to:

```text
OVMF (UEFI)
```

Expected configuration:

```text
bios: ovmf
```

The tested deployment successfully booted using OVMF.

### Verify

```bash
qm config 200
```

Confirm:

```text
bios: ovmf
```

**Validation Result:** The HAOS VM is configured for UEFI boot.

---

## 9. Configure the HAOS Boot Disk

The inherited template configuration may still reference a virtual CD-ROM or network boot.

Example inherited state:

```text
boot: order=ide2;net0
```

This does not include the HAOS disk.

Configure `scsi0` as the boot device.

Sanitized command:

```bash
qm set 200 --boot order=scsi0
```

### Verify

```bash
qm config 200 | grep -E '^(bios|boot|scsi0|efidisk)'
```

Expected relevant output:

```text
bios: ovmf
boot: order=scsi0
scsi0: vm-storage:200/vm-200-disk-0.raw,size=32G
```

**Validation Result:** The imported HAOS disk is the VM's active boot device.

---

## 10. Verify the VM Configuration Before First Boot

The intended VM settings are:

```text
VM ID:            200
Name:             vm-haos
BIOS:             OVMF (UEFI)
CPU type:         host
CPU cores:        6
Sockets:          1
Memory:           4000 MB
SCSI Controller:  VirtIO SCSI
HAOS Disk:        scsi0
Disk Size:        32 GB
Network Model:    VirtIO
Bridge:           vmbr-admin
OS Type:          Linux
```

Sanitized example:

```text
bios: ovmf
boot: order=scsi0
cores: 6
cpu: host
memory: 4000
name: vm-haos
net0: virtio=<redacted-mac>,bridge=vmbr-admin
numa: 0
ostype: l26
scsi0: vm-storage:200/vm-200-disk-0.raw,size=32G
scsihw: virtio-scsi-pci
sockets: 1
```

The MAC address is intentionally omitted because it identifies a specific VM interface.

Environment-specific administrative tags are also omitted because they may disclose internal classification or operational naming conventions.

**Validation Result:** The VM configuration is ready for the first HAOS boot.

---

# Initial HAOS Boot

## 11. Start the HAOS VM

Start:

```text
vm-haos
```

Open the Proxmox console immediately to observe startup.

HAOS should initialize services including:

```text
Home Assistant OS Agent
Network Manager
Network target
containerd container runtime
HAOS Configuration Manager
```

This confirms:

* OVMF successfully boots the HAOS disk.
* The HAOS filesystem is readable.
* The imported virtual disk is functional.
* VirtIO devices are detected.
* HAOS reaches its normal operating environment.

**Validation Result:** HAOS successfully boots from the imported SCSI disk.

---

## 12. Identify the HAOS Network Interface

If the network does not provide DHCP, HAOS may boot without an IPv4 address.

Example console output:

```text
IPv4 addresses for enp0s18: (No address)
OS Version: Home Assistant OS 18.2
Home Assistant Core: landingpage
```

The interface is:

```text
enp0s18
```

At the HAOS CLI prompt:

```text
ha >
```

run:

```bash
network info
```

This confirms the active HAOS interface before making changes.

### Important

HAOS is not Ubuntu Server.

Do not configure HAOS networking through:

```text
/etc/netplan/
/etc/network/interfaces
```

or other conventional Debian/Ubuntu host networking procedures.

HAOS provides its own management interface and uses NetworkManager internally.

**Validation Result:** The HAOS virtual NIC is identified before assigning a static address.

---

# Static IPv4 Configuration

## 13. Configure the Static Address

This sanitized example uses:

```text
192.0.2.54/24
```

with:

```text
Gateway: 192.0.2.1
DNS:     192.0.2.53
```

These are documentation values only.

At the `ha >` prompt:

```bash
network update enp0s18 --ipv4-method static --ipv4-address 192.0.2.54/24 --ipv4-gateway 192.0.2.1 --ipv4-nameserver 192.0.2.53
```

### Command Breakdown

`network update enp0s18`

Updates the HAOS network configuration for the selected interface.

`--ipv4-method static`

Uses manually configured IPv4 instead of DHCP.

`--ipv4-address 192.0.2.54/24`

Assigns the documentation-only static IPv4 address and subnet prefix.

`--ipv4-gateway 192.0.2.1`

Defines the documentation-only default gateway.

`--ipv4-nameserver 192.0.2.53`

Defines the documentation-only DNS resolver.

### Verify

```bash
network info
```

Expected sanitized output:

```text
interface: enp0s18
ipv4:
  address:
  - 192.0.2.54/24
  method: static
  ready: true
```

The actual deployment must use the production gateway and DNS resolver rather than the documentation values.

**Validation Result:** HAOS is configured with static IPv4 networking.

---

# Home Assistant Initialization

## 14. Access HAOS from a Browser

With the documentation address, the Home Assistant URL would be:

```text
http://192.0.2.54:8123
```

This address is an example only and is not intended to be reachable on the public Internet.

During initial startup, the browser displays:

```text
Preparing Home Assistant
```

During this phase HAOS:

* Establishes network connectivity.
* Resolves external services through DNS.
* Reaches the required container registries.
* Downloads Home Assistant Core.
* Initializes Supervisor-managed services.

The deployment initialized:

```text
Home Assistant Core 2026.8.2
```

### Initial `landingpage` State

Before Core initialization completes, HAOS may report:

```text
Home Assistant Core: landingpage
```

This is a temporary initialization state and not necessarily a failure.

HAOS subsequently initializes:

```text
Home Assistant Core 2026.8.2
```

**Validation Result:** HAOS reaches required services and initializes Home Assistant Core.

---

# Existing Home Assistant Migration

## 15. Prepare the Existing Home Assistant Backup

The migration source is an existing Home Assistant Container deployment.

Before replacing it operationally, create a current Home Assistant backup.

The deployment backup contained:

```text
Settings and history
```

and corresponded to:

```text
Home Assistant 2026.8.2
```

The original backup size was small enough to upload directly through onboarding.

Exact timestamps, filenames, archive identifiers, and storage locations should be removed from public documentation unless required.

The previous Docker installation should not be immediately destroyed.

Retaining it provides a rollback path until HAOS is fully validated.

**Validation Result:** A current application-level backup is available before migration.

---

## 16. Use Restore During HAOS Onboarding

After HAOS preparation completes, onboarding provides an option to restore an existing Home Assistant backup.

Select the existing backup rather than creating a new Home Assistant environment.

The sanitized migration path is:

```text
Existing Home Assistant Container
|
+-- Home Assistant Backup
    |
    +-- Settings
    +-- History
    +-- Existing Home Assistant configuration
        |
        v
Upload during HAOS onboarding
        |
        v
HAOS 18.2
        |
        +-- Home Assistant Core 2026.8.2
```

Because the source backup and the new HAOS environment use the same Home Assistant Core release, no Core-version transition is required as part of the restore.

### Do Not Interrupt Restoration

Once restoration starts:

* Do not reboot the HAOS VM.
* Do not restart Supervisor.
* Do not unnecessarily refresh or restart the restore process.
* Do not start the previous Home Assistant instance unless performing a deliberate rollback.
* Allow Home Assistant to restart automatically as required.

Actual restore duration depends on:

```text
Backup size
Database size
Storage performance
Application data
Integration initialization
Supervisor startup
```

**Validation Result:** HAOS accepts the backup and begins restoration of the previous Home Assistant environment.

---

# Final State at End of This Procedure

At the end of the documented procedure:

```text
Proxmox VE
|
+-- VM 200 - vm-haos
    |
    +-- OVMF UEFI
    |
    +-- 6 vCPU
    |
    +-- 4000 MB RAM
    |
    +-- VirtIO Network
    |   |
    |   +-- vmbr-admin
    |   |
    |   +-- enp0s18
    |       |
    |       +-- 192.0.2.54/24
    |
    +-- VirtIO SCSI
        |
        +-- scsi0
            |
            +-- 32 GB HAOS disk
                |
                +-- Home Assistant OS 18.2
                    |
                    +-- Supervisor
                    |
                    +-- Home Assistant Core 2026.8.2
                        |
                        +-- Existing backup restoration
```

The migration has reached the backup restoration phase.

The original Home Assistant Container deployment should remain available but stopped until the HAOS migration is operationally validated.

**Validation Result:** The HAOS VM is installed, bootable, network-connected, running Home Assistant Core, and restoring the previous Home Assistant configuration.

---

# Post-Restore Validation

These steps occur after restoration and are not recorded as completed deployment actions.

* [ ] Wait for HAOS to complete backup restoration.
* [ ] Allow Home Assistant to restart.
* [ ] Open the Home Assistant frontend using the production address.
* [ ] Log in using the existing Home Assistant credentials.
* [ ] Verify existing dashboards.
* [ ] Verify `Settings -> Devices & services`.
* [ ] Verify existing integrations.
* [ ] Verify existing entities.
* [ ] Verify existing automations.
* [ ] Verify existing scripts.
* [ ] Verify existing ESPHome or other IoT integrations.
* [ ] Review `Settings -> System -> Logs`.
* [ ] Confirm the intended static IPv4 configuration remains active.
* [ ] Confirm the intended gateway and DNS configuration.
* [ ] Keep the previous Docker Home Assistant deployment stopped but intact during validation.
* [ ] Do not delete the previous Home Assistant Docker volume until HAOS is verified operationally.

**Next Step:** Complete post-restore validation before retiring the previous Home Assistant Container deployment.

---

# Rollback Strategy

The previous Home Assistant Container installation should not be deleted immediately after restoration.

The rollback model is:

```text
HAOS migration successful
        |
        +-- Validate HAOS
        |
        +-- Keep old Docker HA stopped
        |
        +-- Observe HAOS operation
        |
        +-- Retire old installation only after validation
```

If the HAOS migration fails before the previous environment is removed, the Docker-based Home Assistant deployment remains available as the rollback target.

Avoid operating both Home Assistant instances simultaneously using the same production identity, network address, or device integrations.

**Validation Result:** The migration retains an application rollback path until the new deployment is verified.

---

# HAOS Administration Notes

## HAOS Is Not Ubuntu

HAOS should not be treated as a conventional Ubuntu or Debian server.

It is a purpose-built Home Assistant operating system.

Routine operating system administration should not rely on:

```text
apt
apt-get
netplan
systemctl-based application deployment
manual Docker Compose stacks
general-purpose host package installation
```

Home Assistant-specific services are managed through the HAOS and Supervisor ecosystem.

---

## Container Runtime

HAOS includes the container infrastructure required by Home Assistant.

During boot, HAOS initializes:

```text
containerd container runtime
```

The presence of a container runtime does not make HAOS a general-purpose Docker host.

The intended architecture is:

```text
HAOS
|
+-- Supervisor-managed Home Assistant services
|
+-- Home Assistant applications
|
+-- Home Assistant Core
```

Unrelated Docker Compose workloads should remain on dedicated general-purpose Docker hosts or other appropriate VMs.

---

## SSH Access

HAOS does not expose a conventional general-purpose SSH server by default.

Routine SSH-style administrative access can be added after Home Assistant is operational through the appropriate Home Assistant application.

That configuration is outside the completed scope of this deployment.

The Proxmox console remains available for HAOS administrative access through:

```text
ha >
```

**Next Step:** Configure optional administrative access only after validating the restored Home Assistant instance.

---

# Security and Isolation Notes

* HAOS runs as a dedicated virtual appliance rather than sharing a general-purpose Docker host.
* The VM is attached only to the intended infrastructure bridge.
* Home Assistant uses static addressing appropriate for the surrounding environment.
* Network segmentation remains controlled by external infrastructure.
* HAOS is not used as a general-purpose application server.
* Administrative services should only be exposed to networks that require them.
* Matter and IoT connectivity should be evaluated separately from administrative network configuration.
* The previous Home Assistant environment is retained temporarily for rollback rather than immediately destroyed.
* Proxmox provides an infrastructure-level recovery boundary around HAOS.
* Home Assistant backups provide application-level recovery independently of VM-level backups.
* No external trust assumptions should be made merely because a device resides on an internal network.
* Access to Proxmox and Home Assistant administrative planes should remain restricted by the existing network security architecture.
* Public documentation should not disclose infrastructure addressing or administrative identifiers unless required.

The deployment follows established cybersecurity frameworks and best practices, including NIST SP 800-171 concepts, without representing or claiming certification or compliance status.

**Validation Result:** HAOS remains a dedicated VM with separate application-level and infrastructure-level recovery paths.

---

# Key Commands

## Display Proxmox Storage

```bash
pvesm status
```

Purpose:

Displays configured Proxmox storage and identifies the storage ID used for VM disk import.

Review storage names before publishing command output.

---

## Decompress HAOS

```bash
cd /root/images
unxz haos_ova-18.2.qcow2.xz
```

Purpose:

Extracts the downloaded HAOS QCOW2 disk image.

---

## Verify the HAOS Image

```bash
ls -lh /root/images/haos_ova-18.2.qcow2
```

Purpose:

Confirms that the decompressed HAOS image exists before import.

---

## Import HAOS Disk

```bash
qm importdisk 200 /root/images/haos_ova-18.2.qcow2 vm-storage
```

Purpose:

Imports the HAOS disk image into Proxmox storage and associates it with the sanitized example VM.

Replace:

```text
200
/root/images/haos_ova-18.2.qcow2
vm-storage
```

with the actual local values when applying the procedure privately.

---

## Display VM Configuration

```bash
qm config 200
```

Purpose:

Displays the complete Proxmox VM configuration.

Before publishing output, sanitize:

```text
MAC addresses
bridge names
storage names
tags
descriptions
PCI/USB passthrough IDs
network addresses
other internal identifiers
```

---

## Configure Boot Disk

```bash
qm set 200 --boot order=scsi0
```

Purpose:

Configures the imported HAOS SCSI disk as the boot device.

---

## Verify Boot Configuration

```bash
qm config 200 | grep -E '^(bios|boot|scsi0|efidisk)'
```

Expected sanitized example:

```text
bios: ovmf
boot: order=scsi0
scsi0: vm-storage:200/vm-200-disk-0.raw,size=32G
```

---

## Display HAOS Network Information

Run from:

```text
ha >
```

Command:

```bash
network info
```

Purpose:

Displays HAOS network interfaces and current IPv4/IPv6 configuration.

Review production addresses before publishing the output.

---

## Configure HAOS Static IPv4

Documentation example:

```bash
network update enp0s18 --ipv4-method static --ipv4-address 192.0.2.54/24 --ipv4-gateway 192.0.2.1 --ipv4-nameserver 192.0.2.53
```

Purpose:

Configures the HAOS interface with a static IPv4 address, default gateway, and DNS resolver.

The values in this document are examples only.

---

# Troubleshooting Reference

## HAOS Shows No IPv4 Address

Observed condition:

```text
IPv4 addresses for enp0s18: (No address)
```

### Cause

The connected network does not provide DHCP.

### Fix

Identify the interface:

```bash
network info
```

Then configure static IPv4 using the real production values.

Sanitized example:

```bash
network update enp0s18 --ipv4-method static --ipv4-address 192.0.2.54/24 --ipv4-gateway 192.0.2.1 --ipv4-nameserver 192.0.2.53
```

### Validation

```bash
network info
```

---

## VM Does Not Attempt to Boot HAOS

### Cause

The inherited VM boot order may still reference a virtual optical drive or network adapter rather than `scsi0`.

### Check

```bash
qm config 200
```

Example incorrect inherited state:

```text
boot: order=ide2;net0
```

### Fix

```bash
qm set 200 --boot order=scsi0
```

### Validate

```bash
qm config 200 | grep '^boot:'
```

Expected:

```text
boot: order=scsi0
```

---

## Imported Disk Appears as Unused

Example:

```text
unused0: vm-storage:200/vm-200-disk-0.raw
```

### Cause

`qm importdisk` imports the disk into Proxmox storage but does not automatically attach it as an active VM disk.

### Fix

In the Proxmox interface:

```text
VM
-> Hardware
-> Unused Disk 0
```

Attach using:

```text
Bus/Device: SCSI 0
```

### Validate

```bash
qm config 200
```

Expected sanitized output:

```text
scsi0: vm-storage:200/vm-200-disk-0.raw,size=32G
```

---

## HAOS Displays `landingpage`

Observed:

```text
Home Assistant Core: landingpage
```

### Cause

HAOS has booted, but Supervisor is still preparing or downloading the complete Home Assistant Core environment.

### Fix

Allow initialization to continue.

Do not treat `landingpage` by itself as an installation failure.

### Validation

Open the Home Assistant frontend using the configured production address.

The browser should display preparation progress and eventually transition to onboarding or backup restoration.

---

# Evidence and Operational Records

Useful evidence includes:

```text
HAOS deployment evidence
|
+-- Proxmox VM configuration
|   |
|   +-- qm config <vm-id>
|
+-- Proxmox storage state
|   |
|   +-- pvesm status
|
+-- HAOS network state
|   |
|   +-- network info
|
+-- Home Assistant backup
|   |
|   +-- Current migration backup
|
+-- Home Assistant system logs
    |
    +-- Post-restore startup and integration logs
```

Before publishing evidence, sanitize:

```text
Hostnames
IP addresses
MAC addresses
VM IDs where sensitive
storage names
bridge names
tags
DNS names
gateway addresses
backup archive names
internal paths
administrative identifiers
```

For future rebuilds, capture:

```bash
qm config <vm-id>
```

and retain a current Home Assistant backup independently of the VM.

The Home Assistant application backup and Proxmox VM backup serve different recovery purposes and should not be treated as substitutes.

**Next Step:** Capture a new HAOS-level backup after migration and post-restore validation are complete.

---

# Related Search Keywords

home-assistant, home-assistant-os, haos, proxmox, kvm, qcow2, qm-importdisk, ovmf, uefi, virtio-scsi, static-ip, ha-cli, backup-restore, migration, docker-to-haos, virtualization, rollback

---

## Revision Control

| Version   | Date       | Summary                                                                                                                                                                                                           | Author      |
| --------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-08-20 | Sanitized publication version of the HAOS 18.2 deployment on Proxmox VE through restoration of the existing Home Assistant backup, with production addressing and infrastructure identifiers removed or replaced. | projectfong |
| :::       |            |                                                                                                                                                                                                                   |             |
