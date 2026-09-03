# Home Assistant OS 18.2 Migration from Proxmox VE to Bare Metal

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document records the migration of Home Assistant OS (HAOS) 18.2 from an existing Proxmox VE virtual machine to a dedicated bare-metal Dell OptiPlex 7040 Micro.

The migration replaces the virtualized HAOS deployment with a purpose-built physical Home Assistant appliance while preserving the existing Home Assistant configuration through the native Home Assistant backup and restore process.

The procedure covers preparation of the existing HAOS virtual machine, creation of a final migration backup, installation of HAOS onto dedicated storage, preparation of the target hardware, validation of the fresh bare-metal installation, network cutover, restoration of the existing Home Assistant environment, and post-restore validation.

The migration source was running:

```text
Home Assistant OS:   18.2
Home Assistant Core: 2026.8.3
Supervisor:          2026.08.0
Architecture:        amd64
Machine:             qemux86-64
```

The target platform is a Dell OptiPlex 7040 Micro using a dedicated SATA SSD.

The Home Assistant backup restoration completed successfully. Supervisor job status reported:

```text
done: true
progress: 100
errors: []
```

The restored bare-metal system successfully started Home Assistant Core and the restored Supervisor-managed applications.

Post-restore validation identified two important network migration considerations:

1. Host-level static routes used for segmented networks may not automatically migrate to a new physical network interface.
2. IPv6 host identity may change when replacing a virtual network adapter with a physical network adapter.

Both conditions can affect integrations even when Home Assistant itself restores successfully.

## Purpose

The purpose of this migration is to move Home Assistant OS from a virtualized Proxmox VE environment to dedicated physical hardware.

The migration provides:

* A dedicated physical Home Assistant appliance.
* Direct access to CPU, memory, storage, USB, Ethernet, and other supported hardware.
* Removal of the Proxmox hypervisor dependency from Home Assistant operation.
* Elimination of the HAOS virtual machine layer.
* Dedicated local storage for Home Assistant OS.
* Continued use of Supervisor-managed Home Assistant applications.
* Home Assistant-native backup and restore capabilities.
* A retained rollback path through the original powered-off Proxmox VM.
* A platform suitable for future direct hardware integration.
* Clear separation between Home Assistant and general-purpose server workloads.

The existing Proxmox HAOS VM is retained but powered off until the bare-metal installation has completed restoration and operational validation.


**Next Step:** Review the recorded source and target environments before reproducing the migration.

---

## Migration Environment

The following values were observed during the migration.

| Component             | Source                 | Target                       |
| --------------------- | ---------------------- | ---------------------------- |
| Platform              | Proxmox VE VM          | Dell OptiPlex 7040 Micro     |
| Operating system      | Home Assistant OS 18.2 | Home Assistant OS 18.2       |
| Home Assistant Core   | 2026.8.3               | 2026.8.3                     |
| Supervisor            | 2026.08.0              | 2026.08.0                    |
| Architecture          | amd64                  | amd64                        |
| Machine type          | qemux86-64             | generic-x86-64               |
| CPU                   | Virtual CPU allocation | Intel Core i5-6500T          |
| Physical CPU topology | Virtualized            | 4 cores / 4 threads          |
| Memory                | Virtual allocation     | 16 GB physical RAM           |
| Storage               | Proxmox virtual disk   | Dedicated SATA SSD           |
| Firmware              | OVMF UEFI              | Dell UEFI                    |
| Network               | VirtIO virtual NIC     | Integrated physical Ethernet |
| IPv4 method           | Static                 | DHCP reservation             |
| IPv6 method           | Automatic              | Automatic                    |
| Backup                | HAOS native backup     | Restored during onboarding   |

Environment-specific network addresses, interface identifiers, hardware addresses, and backup identifiers should be recorded separately in private operational documentation.

**Validation Result:** The source and target platforms were identified and the migration backup was successfully restored to the bare-metal system.

---

## Migration Architecture

### Previous Architecture

```text
Physical Server
|
+-- Proxmox VE
    |
    +-- Home Assistant VM
        |
        +-- Home Assistant OS 18.2
            |
            +-- Supervisor
            |
            +-- Home Assistant Core
            |
            +-- Supervisor-managed applications
```

### Target Architecture

```text
Dedicated Physical System
|
+-- Home Assistant OS 18.2
    |
    +-- Supervisor
    |
    +-- Home Assistant Core
    |
    +-- Supervisor-managed applications
```

The migration removes the following dependency from the Home Assistant service path:

```text
Proxmox VE
```

HAOS becomes the operating system directly installed on the dedicated physical computer.

The target system is treated as a dedicated Home Assistant appliance rather than as a general-purpose Linux server.

**Next Step:** Record the existing HAOS configuration before shutting down the source VM.

---

# Pre-Migration Preparation

## 1. Record the Existing HAOS System Information

Before migration, inspect the existing HAOS system using the Home Assistant CLI.

Run:

```bash
ha info
```

Record:

```text
Architecture
HAOS version
Home Assistant Core version
Supervisor version
Machine type
Hostname
Timezone
Supported architecture
System state
```

The virtualized source should report a virtualization-specific machine type such as:

```text
machine: qemux86-64
```

The bare-metal installation should no longer identify itself as a QEMU virtual machine after migration.

**Validation Result:** The source HAOS system information is recorded before migration.

---

## 2. Install Terminal & SSH Before Migration

A virtualized deployment may rely on the hypervisor console for direct administrative access.

Because that console is no longer available after migration, install the Terminal & SSH application before creating the final migration backup when remote administrative access is required.

Recommended application settings include:

```text
Start on boot: Enabled
Watchdog:      Enabled
Auto update:   Based on local update policy
```

Installing the application before the final backup allows its configuration to be included in the migration backup.

The application also provides convenient access to the Home Assistant CLI after migration.

> [!IMPORTANT]
> Terminal & SSH operates inside a Supervisor-managed container and is not equivalent to the HAOS host shell.

**Validation Result:** Terminal & SSH is installed and included in the source HAOS environment before the final migration backup.

---

## 3. Record the Existing HAOS Network Configuration

Before shutting down the source VM, record the HAOS host network configuration.

Run:

```bash
ha network info
```

Record:

```text
Physical interface
IPv4 address and prefix
IPv4 gateway
IPv4 configuration method
DNS servers
IPv6 addresses and prefixes
IPv6 gateway
IPv6 configuration method
Host Internet status
Supervisor Internet status
```

Do not publish environment-specific:

```text
IP addresses
MAC addresses
IPv6 interface identifiers
Internal DNS addresses
Internal gateway addresses
```

unless disclosure is intentional.

### Docker Network

HAOS also maintains internal container networking.

When commands such as:

```bash
ip addr
```

are executed from inside the `core-ssh` application container, the output represents the container network namespace rather than the physical HAOS host interface.

For host network validation, use:

```bash
ha network info
```

rather than relying on `ip addr` from inside `core-ssh`.

### Record Static Routes for Segmented Networks

> [!WARNING]
> If HAOS communicates with devices across segmented networks, VLANs, or routed subnets using custom static routes, record those routes separately before migration.
>
> A Home Assistant backup can successfully restore Home Assistant configuration and integrations without recreating host-level static routes associated with the previous network interface or NetworkManager connection profile.

From the HAOS host shell, record IPv4 routes:

```bash
ip route
```

If custom IPv6 routes are used, also record:

```bash
ip -6 route
```

If routes are persisted through NetworkManager, record the active connection profiles:

```bash
nmcli connection show
```

Inspect the applicable connection profile:

```bash
nmcli -f ipv4.routes,ipv6.routes connection show "<connection-name>"
```

Record for each custom route:

```text
Destination network
Prefix length
Next-hop gateway
Address family
Associated HAOS interface
Associated NetworkManager connection
```

A generalized segmented-network route may resemble:

```text
<SEGMENTED_SUBNET>/<PREFIX> via <ROUTER_ADDRESS>
```

> [!IMPORTANT]
> Custom host routes should be treated as migration prerequisites in segmented environments.
>
> Retaining the same production IPv4 address does not guarantee that routes associated with the previous HAOS network configuration will exist on replacement hardware.

**Validation Result:** The production IPv4 and IPv6 HAOS host network configuration and required custom routes are captured before migration.

---

# Backup Preparation

## 4. Configure Automatic HAOS Backups

Before migration, establish an automatic Home Assistant backup policy.

An example policy is:

```text
Schedule:                 Daily
Backup before updating:   Enabled
Retention:                Defined
Encryption:               Enabled
```

The migration should not rely exclusively on the automatic backup schedule.

Create a separate manual backup immediately before the production cutover.

**Validation Result:** HAOS has an established automatic backup policy before migration.

---

## 5. Create the Final Migration Backup

Create a final manual backup specifically for the bare-metal migration.

Use a descriptive backup name that clearly identifies the migration backup.

The backup should include all required Home Assistant configuration and application data.

Depending on the environment, restorable content may include:

```text
Home Assistant
|
+-- Settings and history
|
+-- Share folder
|
+-- SSL certificates

Applications
|
+-- Matter services
|
+-- Voice services
|
+-- Terminal & SSH
|
+-- Other Supervisor-managed applications
```

Download the migration backup outside the HAOS system before shutting down the source VM.

If Home Assistant backup encryption is enabled, retain the corresponding recovery information independently of the HAOS appliance.

### Important

Do not rely on the only copy of a migration backup being stored inside the system being migrated.

The backup should exist independently of:

```text
Source HAOS VM
Target HAOS installation
Target storage device
```

**Validation Result:** A current manual migration backup is created and retained outside the source HAOS VM.

---

# Bare-Metal Hardware Preparation

## 6. Prepare the Target Hardware

The target hardware used for this migration is:

```text
Dell OptiPlex 7040 Micro
```

Observed hardware includes:

```text
CPU:      Intel Core i5-6500T
Cores:    4
Threads:  4
Memory:   16 GB
Storage:  Dedicated SATA SSD
Network:  Integrated wired Ethernet
Firmware: UEFI
```

The system is configured as a dedicated HAOS appliance.

Wired Ethernet is used for production connectivity.

### Storage Selection

A dedicated storage device was selected for HAOS rather than overwriting existing storage that might contain useful data.

The resulting storage configuration is:

```text
Dedicated Physical System
|
+-- Dedicated SATA SSD
    |
    +-- Home Assistant OS
```

**Validation Result:** The target hardware is physically prepared with dedicated storage for HAOS.

---

## 7. Configure Firmware for HAOS

The target system uses UEFI firmware.

Relevant firmware configuration includes:

```text
Boot mode:    UEFI
Secure Boot:  Disabled
SATA mode:    AHCI
```

Legacy boot is not required for the HAOS Generic x86-64 image.

TPM functionality is not required for this HAOS deployment.

### Boot Model

```text
System UEFI
|
+-- Dedicated SSD
    |
    +-- HAOS boot partitions
        |
        +-- Home Assistant OS
```

**Validation Result:** The target system is capable of directly booting the HAOS Generic x86-64 image using UEFI.

---

# HAOS Bare-Metal Installation

## 8. Obtain the HAOS Generic x86-64 Image

The bare-metal installation uses the Home Assistant OS Generic x86-64 image rather than the KVM/Proxmox QCOW2 image.

The installation model is:

```text
HAOS Generic x86-64 image
        |
        v
Disk imaging utility
        |
        v
Dedicated SSD
        |
        v
Bare-metal system
```

The HAOS image is a disk image rather than a conventional operating system installer ISO.

It is written directly to the target storage device.

**Validation Result:** The correct Generic x86-64 HAOS image is selected for the physical deployment.

---

## 9. Flash HAOS to the Target SSD

A disk imaging utility can be used to write the HAOS image directly to the target SSD.

The process is:

```text
HAOS Generic x86-64 image
        |
        v
Disk imaging utility
        |
        v
Dedicated SSD
```

Use an imaging utility that performs write verification when possible.

### Important

The HAOS disk image overwrites the target disk.

Verify the selected target device before starting the flash operation.

Do not select a disk containing data that must be retained.

**Validation Result:** HAOS is successfully written and validated on the target SSD.

---

# Initial Bare-Metal Boot

## 10. Boot the Bare-Metal System

Initially boot the newly imaged system while connected to an isolated or controlled test network when practical.

This allows the physical installation to be validated without disrupting the production HAOS VM.

HAOS should boot and present the Home Assistant onboarding interface.

Expected onboarding options include:

```text
Create my smart home
Upload backup
Home Assistant Cloud
```

This confirms:

* The target system successfully boots the HAOS image.
* UEFI configuration is functional.
* The target SSD is readable.
* HAOS initializes successfully.
* The integrated Ethernet adapter is detected.
* Network connectivity is available.
* Home Assistant onboarding is accessible.

Keep the production HAOS VM operational during this isolated test.

**Validation Result:** The fresh bare-metal HAOS installation successfully boots and reaches the Home Assistant onboarding interface.

---

# Production Network Cutover

## 11. Prepare Network Access for the New Physical NIC

The new physical Ethernet adapter presents a different MAC address from the previous Proxmox virtual NIC.

Networks that restrict unknown DHCP clients may therefore identify the bare-metal system as a new client.

The controlled migration process is:

```text
Temporarily authorize new client
        |
        v
Boot bare-metal HAOS
        |
        v
Identify physical NIC
        |
        v
Authorize or register client
        |
        v
Complete HAOS restoration
        |
        v
Verify production network configuration
        |
        v
Restore restrictive DHCP policy
```

Do not publish the physical adapter MAC address unless disclosure is intentional.

The temporary network policy change should not become the permanent production state.

**Validation Result:** Network access is temporarily adjusted to permit controlled onboarding of the new physical HAOS system.

---

## 12. Shut Down the Existing Proxmox HAOS VM

Before placing the restored bare-metal HAOS installation onto the production network using the existing Home Assistant identity, shut down the original Proxmox HAOS VM.

The required cutover state is:

```text
Old Proxmox HAOS VM:  OFF
New bare-metal HAOS:  ON
```

Do not operate both instances simultaneously when they are intended to represent the same Home Assistant environment.

### Rollback Preservation

The old VM should remain:

```text
Powered off
Retained
Unmodified
Not deleted
```

until the new bare-metal system passes validation.

**Validation Result:** The migration preserves the Proxmox VM as a rollback target while preventing simultaneous production operation.

---

# Backup Restoration

## 13. Upload the Migration Backup

From the fresh HAOS onboarding interface, select:

```text
Upload backup
```

Upload the final migration backup created before cutover.

For a complete migration, select all required components for restoration.

The restoration path is:

```text
Existing HAOS
|
+-- Home Assistant
|
+-- Migration Backup
    |
    +-- Settings and history
    +-- Share folder
    +-- SSL certificates
    +-- Applications
            |
            v
Upload during bare-metal HAOS onboarding
            |
            v
Bare-metal HAOS
```

**Validation Result:** The migration backup is accepted by the bare-metal HAOS onboarding environment.

---

## 14. Start the Restore Operation

After selecting the required backup components, start restoration.

During restoration:

* The backup is uploaded.
* HAOS accepts the restore request.
* Home Assistant configuration is restored.
* Supervisor-managed applications are restored.
* Home Assistant Core restarts automatically.

Verify restore status using:

```bash
ha jobs info
```

A successful restore should report:

```text
done: true
progress: 100
errors: []
```

Restored components may include:

```text
Share folder
SSL certificates
Home Assistant
Applications
Supervisor configuration
Home Assistant Core
```

**Validation Result:** The Home Assistant backup restore completes successfully with restore jobs at 100 percent and no reported errors.

---

# Network Behavior During Restoration

## 15. Do Not Assume the Old Interface Name Will Be Preserved

The source HAOS system uses a virtual network interface associated with the Proxmox VirtIO adapter.

The bare-metal system uses a physical Ethernet controller and therefore receives a different interface identity.

The production IPv4 address can remain consistent through:

```text
Static configuration
or
DHCP reservation
```

while the physical interface name and MAC address change.

> [!IMPORTANT]
> Do not create scripts, firewall policies, or migration procedures that unnecessarily depend on a specific Linux interface name.

**Validation Result:** The bare-metal HAOS host operates through its physical Ethernet interface with the intended production network configuration.

---

## 16. Verify IPv6 After Restoration

If the source HAOS IPv6 configuration uses:

```text
method: auto
```

the exact host portion of its IPv6 address may change after migration.

This occurs because the physical Ethernet interface has a different network identity from the previous virtual NIC.

Verify:

```text
IPv6 prefix
IPv6 gateway
IPv6 routing
Firewall policy
Matter connectivity
Thread connectivity
```

IPv6 should not be disabled as a migration troubleshooting shortcut when Home Assistant environments use Matter or Thread.

**Validation Result:** IPv6 remains operational on the bare-metal HAOS host.

---

## 16.1 Restore Static Routes for Segmented Networks

> [!WARNING]
> If Home Assistant communicates with devices across segmented networks, VLANs, or routed subnets using static routes configured on the HAOS host, verify and recreate those routes after migration.

A successful Home Assistant backup restoration does not guarantee that host-level static routes associated with the previous HAOS network interface or NetworkManager connection profile will be recreated on replacement hardware.

A generalized required route may resemble:

```text
<SEGMENTED_SUBNET>/<PREFIX> via <ROUTER_ADDRESS>
```

### Observed Symptoms

A missing route can cause multiple apparently unrelated integrations to fail or time out.

Examples include:

```text
OpenThread Border Router:
Failed setup, will retry

Smart-home integrations:
Failed setup, will retry

ESPHome:
Connection timeout
```

The common failure may be routing between HAOS and the segmented network rather than independent integration failures.

### Important Diagnostic Pattern

If several unrelated integrations fail immediately after migration and the affected devices share the same remote subnet, investigate routing before deleting integrations, re-pairing devices, or rebuilding configuration.

A typical failure pattern is:

```text
HAOS local access:              Working
HAOS Internet access:           Working
Supervisor Internet access:     Working
Devices on local subnet:        Working
Devices on routed IoT subnet:   Failing
Multiple integrations:          Failing
```

This strongly indicates a network-path problem.

### HAOS Host Shell Requirement

> [!IMPORTANT]
> The Terminal & SSH application runs inside the `core-ssh` container.
>
> `nmcli` is not available from the normal `core-ssh` shell, and the `login` command executed there does not enter the HAOS host operating system.

Host-level NetworkManager configuration requires access to the actual HAOS host shell.

From the physical HAOS console:

```text
homeassistant login: root
```

At the HA CLI prompt, enter:

```text
login
```

The resulting host shell provides access to commands such as:

```bash
ip route
ip -6 route
nmcli connection show
```

Identify the active NetworkManager connection before modifying route configuration.

Do not assume a particular NetworkManager connection name because replacement hardware may use a different profile.

After restoring the required route, affected integrations should be retested before any device re-pairing or integration reconfiguration is attempted.

**Validation Result:** Required static routes are restored and affected integrations recover.

---

## 16.2 Post-Migration Network Routing and IPv6 Identity

### Overview

After restoring HAOS to new physical hardware, application configuration can restore successfully while host-level networking differs from the previous virtual machine.

Two migration-specific conditions should be explicitly checked:

1. Required IPv4 or IPv6 static routes.
2. Host-specific IPv6 identity dependencies.

---

### IPv4 Static Routes May Not Be Preserved

A previous HAOS installation may require a static route such as:

```text
<SEGMENTED_SUBNET>/<PREFIX> via <ROUTER_ADDRESS>
```

After restoring HAOS to bare metal, that host-level route may not exist.

Symptoms can include:

* OpenThread Border Router integration failures.
* Smart-home integration failures.
* ESPHome device connection timeouts.
* Other connectivity failures involving the same routed network.

The HAOS backup can successfully restore Home Assistant configuration without recreating the required host-level NetworkManager route associated with the previous system.

> [!WARNING]
> Record all custom IPv4 and IPv6 routes before migration. Do not assume host-level routes will automatically transfer to different hardware.

---

### IPv6 Matter over Thread Failure

Matter over Thread relies heavily on functional IPv6 routing.

Matter Server logs may indicate a Thread device is known but unreachable:

```text
Resolving (address is unreachable)
```

Before rebuilding Matter or Thread configuration, verify whether the OpenThread Border Router:

* Attached to the existing Thread network.
* Loaded the expected Active Dataset.
* Exposes the expected Thread prefix.
* Routes the Thread prefix through `wpan0`.
* Provides Thread credentials to Matter Server.

A healthy Thread fabric combined with an unreachable Matter device can indicate an infrastructure routing or firewall issue.

---

### Verify HAOS Docker IPv6

Run:

```bash
ha docker info
```

Check:

```text
enable_ipv6
```

Where required by the environment, enable HAOS Docker IPv6:

```bash
ha docker options --enable-ipv6=true
```

Then reboot HAOS if required by the configuration change.

Verify again:

```bash
ha docker info
```

Expected:

```text
enable_ipv6: true
```

Docker IPv6 is one prerequisite and should not be assumed to resolve every IPv6 connectivity issue.

---

### HAOS IPv6 Address Can Change During Migration

The previous Proxmox HAOS VM and new physical HAOS system may receive different IPv6 host addresses even when they remain on the same IPv6 prefix.

This becomes important when infrastructure policies reference a host-specific address.

Examples include:

```text
Host-specific /128 firewall objects
IPv6 ACLs
Inter-VLAN IPv6 policies
Monitoring rules
Static routes
Security policies
```

If the firewall expects the previous HAOS IPv6 identity, traffic originating from the replacement address may no longer match the intended policy.

This can cause Matter Server to report Thread devices as unreachable even when:

* Matter Server is operational.
* The Matter fabric is intact.
* Thread credentials are intact.
* OTBR is operational.
* The Thread mesh is operational.
* Infrastructure routing exists.

---

### Resolution Options

Two valid approaches exist depending on network design:

```text
Option 1:
Update infrastructure policy to recognize the new HAOS IPv6 identity.

Option 2:
Restore the established HAOS IPv6 identity on the replacement system when intentionally required by the network architecture.
```

Whichever approach is used, validate that the resulting firewall and routing policy matches the HAOS address actually in use.

Do not publish production IPv6 host addresses, firewall objects, or internal gateway addresses unless disclosure is intentional.

No Matter device reset, Thread fabric recreation, or device recommissioning should be performed until the network path has been validated.

### IPv6 Thread Reachability Validation

After correcting the network path, test direct IPv6 connectivity from HAOS to a representative Thread device:

```bash
ping -6 -c 4 <THREAD_DEVICE_IPV6>
```

A successful result should resemble:

```text
4 packets transmitted
4 packets received
0 percent packet loss
```

This confirms the path:

```text
HAOS
|
+-- IPv6
    |
    +-- Infrastructure router or firewall
        |
        +-- OpenThread Border Router
            |
            +-- Thread network
                |
                +-- Matter over Thread device
```

**Validation Result:** HAOS successfully reaches the Matter over Thread network over IPv6.

---

### Migration Lesson

A HAOS migration to different hardware changes the underlying network interface.

Even when the same IPv4 address is preserved, IPv6 addressing may change because the new physical NIC has a different network identity.

This can affect:

* IPv6 firewall address objects.
* Host-specific `/128` firewall rules.
* Matter over Thread connectivity.
* Inter-VLAN IPv6 policies.
* IPv6 static routes.
* mDNS and multicast policies.
* Network ACLs referencing the previous HAOS IPv6 address.

For segmented environments, treat HAOS network identity as part of the migration configuration.

Before migration, record:

```bash
ha network info
```

From the HAOS host shell, also record:

```bash
ip route
ip -6 route
```

Document:

```text
Static IPv4 addresses
Static IPv6 addresses
IPv4 static routes
IPv6 static routes
IPv6 default gateway
Firewall objects referencing HAOS
Host-specific IPv4 rules
Host-specific IPv6 rules
Inter-VLAN policies
mDNS reflection policies
Matter and Thread network dependencies
```

Store environment-specific values in restricted operational documentation rather than general-purpose documentation.

---

### Post-Migration Network Validation

After restoring HAOS to new hardware, run:

```bash
ha network info
ha docker info
```

Confirm:

* Expected IPv4 addressing is present.
* Expected IPv6 addressing is present.
* IPv4 gateway is correct.
* IPv6 gateway is correct.
* DNS servers are correct.
* Required IPv4 static routes exist.
* Required IPv6 static routes exist.
* Docker IPv6 is enabled where required.
* Existing firewall objects still match the intended HAOS identity.
* HAOS can reach required routed IoT networks.
* HAOS can reach the OTBR infrastructure network.
* Matter Server can reach Matter over Thread devices.

For Matter over Thread, do not immediately factory-reset or recommission an offline device following a HAOS migration.

First verify:

```text
Home Assistant
      |
      | IPv6
      v
Infrastructure Router / Firewall
      |
      | IPv6 routing and policy
      v
OpenThread Border Router
      |
      | Thread
      v
Matter over Thread Device
```

A device showing as offline may indicate an infrastructure routing or firewall identity problem rather than a damaged Matter pairing or Thread fabric.

---

# Current State at End of This Procedure

At the end of the migration:

```text
Dedicated Physical System
|
+-- Intel Core i5-6500T
|
+-- 16 GB RAM
|
+-- Dedicated SATA SSD
    |
    +-- Home Assistant OS 18.2
        |
        +-- Home Assistant Supervisor
        |
        +-- Home Assistant Core
        |
        +-- Settings and history
        |
        +-- Share folder
        |
        +-- SSL certificates
        |
        +-- Supervisor-managed applications
```

The previous environment remains:

```text
Proxmox VE
|
+-- Previous HAOS VM
    |
    +-- Powered off
    +-- Retained for rollback
```

The backup restoration completes successfully and the system identifies as:

```text
Machine: generic-x86-64
```

**Validation Result:** The bare-metal HAOS installation is operational and the migration backup restoration completes successfully.

---

# Post-Restore Validation

## 17. Verify HAOS System Information

Run:

```bash
ha info
```

Confirm:

```text
Architecture: amd64
Machine:      generic-x86-64
State:        running
Supported:    true
```

The source system previously reported:

```text
machine: qemux86-64
```

The target should report:

```text
machine: generic-x86-64
```

This confirms HAOS is operating as a Generic x86-64 bare-metal installation.

**Validation Result:** Passed.

---

## 18. Verify Host Networking

Run:

```bash
ha network info
```

Verify:

```text
Physical Ethernet interface
Expected IPv4 network
Expected gateway
Expected DNS
Expected IPv6 network
Expected IPv6 gateway
Host Internet connectivity
Supervisor Internet connectivity
```

Do not use the `core-ssh` container's `ip addr` output as the authoritative source for HAOS host addressing.

### Verify Segmented-Network Routes

From the HAOS host shell:

```bash
ip route
```

If applicable:

```bash
ip -6 route
```

Verify all custom routes recorded before migration are present.

Then verify connectivity to representative devices on each routed network before modifying integrations.

**Validation Result:** Passed after required host-level routes are restored.

---

## 19. Verify Home Assistant Core

Run:

```bash
ha core info
```

Confirm:

```text
machine: generic-x86-64
state: running
```

Verify the expected Home Assistant Core version.

If a newer Core update becomes available immediately after migration, defer the update until the migrated system has completed validation and a new post-migration backup has been created.

**Validation Result:** Passed.

---

## 20. Verify Supervisor

Run:

```bash
ha supervisor info
```

Confirm:

```text
Healthy:   true
Supported: true
```

Verify restore job status:

```bash
ha jobs info
```

Expected:

```text
done: true
progress: 100
errors: []
```

**Validation Result:** Passed.

---

## 21. Verify HAOS Hardware Detection

Run:

```bash
ha hardware info
```

and:

```bash
ha os info
```

Verify:

```text
Target storage device
Board type
HAOS version
Graphics devices
Audio devices
USB devices
Serial devices
```

For USB serial devices, prefer persistent paths such as:

```text
/dev/serial/by-id/<DEVICE_IDENTIFIER>
```

over transient paths such as:

```text
/dev/ttyUSB0
```

when supported by the applicable integration.

Do not publish complete device serial identifiers unless required.

**Validation Result:** Passed for primary hardware detection.

---

## 22. Verify Restored Applications

Verify all required Supervisor-managed applications are present and started.

Examples may include:

```text
Matter Server
Piper
Terminal & SSH
Whisper
Other locally installed applications
```

**Validation Result:** Passed.

---

## 23. Verify Home Assistant Configuration

Validate the restored Home Assistant environment.

* [ ] Existing user accounts are available.
* [ ] Existing dashboards load.
* [ ] Existing areas are present.
* [ ] Existing devices are present.
* [ ] Existing entities are present.
* [ ] Existing integrations are loaded.
* [ ] Existing automations are present.
* [ ] Existing scripts are present.
* [ ] Existing helpers are present.
* [ ] Existing scenes are present.
* [ ] Existing history is available.
* [ ] Existing recorder data is available.
* [ ] Existing certificates are present where required.
* [ ] Existing shared files are present where required.

**Validation Status:** Continue operational validation.

---

## 24. Verify Matter and Thread

Matter and Thread should receive specific post-migration validation.

Verify:

* [ ] Matter Server is running.
* [ ] Existing Matter devices remain available.
* [ ] Thread integration is operational.
* [ ] OpenThread Border Router connectivity is operational.
* [ ] IPv6 connectivity is functional.
* [ ] Required multicast traffic is functional.
* [ ] Required mDNS discovery is functional.
* [ ] Existing Thread credentials remain available.
* [ ] Matter devices respond normally.

If the OpenThread Border Router resides on a routed network, verify HAOS routing before modifying Matter or Thread configuration.

A missing host-level route does not indicate failed OTBR restoration or a failed Thread radio.

**Validation Status:** Continue Matter and Thread validation.

---

## 25. Verify Voice Services

Verify restored voice-related applications such as:

```text
Piper
Whisper
Other configured voice services
```

Confirm that required voice pipelines operate normally after restoration.

**Validation Status:** Continue operational validation.

---

## 26. Verify Automations and External Integrations

Verify automations that depend on external infrastructure.

Examples include:

```text
IoT devices
Cameras
Voice assistants
Matter devices
Thread devices
Network services
DNS
mDNS
IPv6
MQTT
External API integrations
```

Review:

```text
Settings
-> System
-> Logs
```

for migration-related failures.

### Segmented Network Warning

> [!WARNING]
> If multiple integrations fail simultaneously after migration, determine whether the affected devices share a common routed subnet before modifying integrations.

Missing HAOS host routes can make otherwise healthy devices and integrations appear unavailable.

Do not immediately:

```text
Delete integrations
Re-pair devices
Reset Matter devices
Reset Thread devices
Reconfigure OTBR
Factory-reset IoT devices
```

Verify routing and firewall reachability first.

**Validation Status:** Continue operational validation.

---

## 27. Restore Restrictive DHCP Policy

After the physical HAOS client has been identified and authorized, restore any network policy temporarily modified during onboarding.

The intended final state is:

```text
Known HAOS physical client: Authorized
Unknown clients:           Restricted according to local policy
```

Do not leave temporary migration exceptions enabled unnecessarily.

**Validation Status:** Verify after migration networking is finalized.

---

# Backup Architecture After Bare-Metal Migration

## 28. Replace the Proxmox Backup Dependency

The previous HAOS VM may have been protected using Proxmox-level VM backups and snapshots.

After migration to bare metal, that recovery layer no longer exists.

The new backup architecture should include both local HAOS backups and off-host backup storage.

Example:

```text
Bare-Metal HAOS
|
+-- Local automatic backups
|   |
|   +-- Scheduled
|   +-- Defined retention
|   +-- Backup before updates
|
+-- Off-host backup
    |
    +-- Network storage
        |
        +-- Dedicated HAOS backup location
```

Use a dedicated service account for off-host backup storage where supported.

Recommended access model:

```text
HAOS backup account
|
+-- Dedicated backup location
    |
    +-- Read
    +-- Write

No administrative access
No unrelated storage access
No guest access
```

This follows established cybersecurity frameworks and best practices by applying least privilege, separation of duties, controlled access, and independent recovery storage.

### Post-Restore Backup Location Repair

A restored HAOS configuration may continue referencing a backup location that is no longer available.

This does not indicate failure of the Home Assistant backup restoration itself.

Remove obsolete destinations from the automatic backup configuration or replace them with the intended off-host backup location.

**Validation Status:** Off-host backup should be configured and tested before migration closure.

---

# Rollback Strategy

The original Proxmox HAOS VM remains the rollback target.

```text
Bare-metal HAOS migration
        |
        +-- Successful
        |     |
        |     +-- Validate services
        |     +-- Validate networking
        |     +-- Validate Matter/Thread
        |     +-- Validate backups
        |     +-- Retain VM temporarily
        |
        +-- Failed
              |
              +-- Power off bare-metal HAOS
              |
              +-- Restore original network state
              |
              +-- Boot original Proxmox HAOS VM
```

If rollback is required:

1. Power off the bare-metal HAOS system.
2. Ensure it is no longer using the production Home Assistant network identity.
3. Start the original Proxmox HAOS VM.
4. Verify network connectivity.
5. Verify Home Assistant services.
6. Investigate the bare-metal migration separately.

Do not run both systems simultaneously using the same Home Assistant production identity.

### Do Not Delete the Source VM Yet

The Proxmox HAOS VM should remain:

```text
Powered off
Intact
Recoverable
```

until the bare-metal system has operated successfully through an appropriate validation period.

**Validation Result:** The original HAOS VM remains available as a direct rollback path.

---

# HAOS Administration Notes

## HAOS Is a Dedicated Appliance

Home Assistant OS is a purpose-built operating system.

It should not be treated as a conventional Ubuntu or Debian server.

Routine administration should not rely on:

```text
apt
apt-get
netplan
manual Docker Compose deployments
general-purpose package installation
```

Home Assistant services should remain within the supported HAOS and Supervisor management model.

---

## Terminal & SSH

Terminal & SSH provides convenient administrative access after migration.

The application runs as a Supervisor-managed application.

Commands executed directly inside the application container may expose the application's container namespace rather than the HAOS host namespace.

For HAOS host network information, use:

```bash
ha network info
```

For general HAOS information, use:

```bash
ha info
```

### Terminal & SSH Does Not Provide the HAOS Host Shell

The `core-ssh` application is not equivalent to direct shell access to the HAOS host operating system.

For example:

```bash
nmcli
```

is not available from the normal `core-ssh` environment.

Running:

```bash
login
```

from the application container invokes the container's normal Linux login utility and does not enter the HAOS host shell.

Host-level NetworkManager administration must be performed through an appropriate HAOS host administrative path.

---

## Physical Console

The bare-metal system can be administered locally using:

```text
Monitor
Keyboard
HAOS local console
```

This replaces the Proxmox virtual console as the lowest-level local administrative path.

Maintaining the ability to connect a local monitor and keyboard provides a recovery path if network access to HAOS is unavailable.

To access the HAOS host shell from the physical console:

```text
homeassistant login: root
```

Then, from the HA CLI:

```text
login
```

The resulting host shell can be used for diagnostics including:

```bash
ip route
ip -6 route
nmcli connection show
```

This distinction is particularly important when managing static routes required by segmented networks.

---

# Security and Isolation Notes

* HAOS runs on dedicated physical hardware.
* The system is not intended to host unrelated general-purpose workloads.
* Administrative access should remain limited to trusted management networks.
* Production network addresses should be managed through controlled network configuration.
* IPv6 should remain available where required for Matter and Thread functionality.
* The physical Ethernet adapter should be explicitly authorized by network infrastructure where client authorization is enforced.
* Temporary relaxation of network access restrictions should be reversed after migration.
* The previous Proxmox HAOS VM remains powered off to prevent duplicate production instances.
* Home Assistant backup encryption should remain enabled where appropriate.
* Backup recovery information should be retained independently of HAOS.
* Off-host backups should use a dedicated service account with limited permissions.
* HAOS should not be directly exposed to untrusted networks unless explicitly required and appropriately protected.
* Network segmentation should continue to control communication between administrative, infrastructure, IoT, and other network roles.
* Custom static routes required to reach segmented networks should be independently documented.
* Custom static routes should be verified after HAOS hardware migrations.
* Matter and Thread connectivity should be validated rather than assuming successful backup restoration guarantees network functionality.
* Host-specific IPv4 and IPv6 firewall dependencies should be documented separately from general documentation.
* The old Proxmox VM should not be deleted until the physical HAOS deployment has been validated.

The design follows established cybersecurity frameworks and best practices, including secure configuration, least privilege, network segmentation, controlled administrative access, independent backups, and post-change validation.

**Validation Result:** The migration maintains a dedicated HAOS role, preserves rollback capability, and avoids combining unrelated services on the Home Assistant appliance.

---

# Key Commands

## Display HAOS System Information

```bash
ha info
```

Purpose:

Displays HAOS, Home Assistant Core, Supervisor, architecture, machine, and system information.

---

## Display HAOS Network Information

```bash
ha network info
```

Purpose:

Displays authoritative HAOS host network information including:

```text
Physical interfaces
IPv4 configuration
IPv6 configuration
Gateways
DNS servers
MAC addresses
Docker network
Host Internet status
Supervisor Internet status
```

Do not publish complete output without reviewing it for environment-specific network information.

---

## Display Home Assistant Core Information

```bash
ha core info
```

Purpose:

Displays Home Assistant Core state and configuration.

---

## Display Home Assistant Core Statistics

```bash
ha core stats
```

Purpose:

Displays Home Assistant Core resource statistics.

---

## Display Supervisor Information

```bash
ha supervisor info
```

Purpose:

Displays Home Assistant Supervisor state and configuration.

---

## Display Restore Job Information

```bash
ha jobs info
```

Purpose:

Displays Supervisor job state, including backup and restore operations.

A completed restore should report the applicable restore job as:

```text
done: true
progress: 100
errors: []
```

---

## Display HAOS Hardware Information

```bash
ha hardware info
```

Purpose:

Displays hardware detected by Home Assistant OS.

Review output before publication because persistent hardware identifiers may be included.

---

## Display HAOS Operating System Information

```bash
ha os info
```

Purpose:

Displays Home Assistant OS information.

---

## Display HAOS Host IPv4 Routes

From the HAOS host shell:

```bash
ip route
```

Purpose:

Displays the host IPv4 routing table.

This is particularly important when HAOS communicates with segmented networks through custom static routes.

Review output before publication because it may disclose internal addressing and routing topology.

---

## Display HAOS Host IPv6 Routes

From the HAOS host shell:

```bash
ip -6 route
```

Purpose:

Displays the host IPv6 routing table.

Review output before publication because it may disclose internal IPv6 prefixes and infrastructure addressing.

---

## Display NetworkManager Connections

From the HAOS host shell:

```bash
nmcli connection show
```

Purpose:

Displays NetworkManager connection profiles available on the HAOS host.

This command is not available from the normal Terminal & SSH `core-ssh` container.

---

# Network Baseline Template

Before migration, capture a network baseline using a restricted operational record.

```text
Interface: <INTERFACE>
MAC:       <MAC_ADDRESS>

IPv4:
  Address: <IPV4_ADDRESS>/<PREFIX>
  Gateway: <IPV4_GATEWAY>
  Method:  <STATIC_OR_AUTO>
  DNS:     <DNS_SERVERS>

IPv6:
  Address:
    <IPV6_ADDRESS>/<PREFIX>
  Gateway:
    <IPV6_GATEWAY>
  Method:
    <STATIC_OR_AUTO>

Internet:
  Host:       <STATUS>
  Supervisor: <STATUS>
```

Record required segmented-network routes separately:

```text
<SEGMENTED_SUBNET>/<PREFIX> via <ROUTER_ADDRESS>
```

This baseline should be retained until post-migration network validation is complete.

---

# Troubleshooting Reference

## Home Assistant Does Not Return After Restore

Do not immediately assume restoration failed.

The restore process may:

```text
Stop Home Assistant
Restore configuration
Restore application data
Restart Supervisor-managed components
Restart Home Assistant
Reconfigure services
```

Inspect the restore job:

```bash
ha jobs info
```

A successfully completed restore should report:

```text
done: true
progress: 100
errors: []
```

---

## Bare-Metal System Receives the Wrong IPv4 Address

Cause:

The restored static network configuration may not be associated with the new physical Ethernet interface, or the system may be using DHCP configuration.

Check:

```bash
ha network info
```

Compare the output against the restricted production network baseline.

Do not assume the old virtual interface name will exist on the physical system.

---

## `ip addr` Shows a Container Address

If the command is executed from:

```text
core-ssh
```

the displayed interface belongs to the Terminal & SSH application container.

This is not necessarily the production HAOS host address.

Use:

```bash
ha network info
```

to inspect the HAOS host network.

---

## `nmcli` Is Not Available

If:

```bash
nmcli
```

returns a command-not-found error from the Terminal & SSH environment, the command is being executed inside the application container.

Use the physical HAOS console and enter the host shell:

```text
homeassistant login: root
```

Then:

```text
login
```

From the resulting host shell:

```bash
nmcli connection show
```

can be used for host-level NetworkManager inspection and configuration.

---

## Integrations Fail After Migration to a Segmented Network

Symptoms may include:

```text
OpenThread Border Router:
Failed setup, will retry

Smart-home integration:
Failed setup, will retry

ESPHome:
Timeout while connecting
```

If multiple affected devices are located on the same remote subnet, verify HAOS routing before changing integration configuration.

From the HAOS host shell:

```bash
ip route
```

Verify required routes using the restricted network baseline.

A missing route can coexist with:

```text
host_internet: true
supervisor_internet: true
```

because Internet connectivity through the default gateway does not prove that a route exists to every segmented internal network.

Do not immediately:

```text
Delete integrations
Re-pair devices
Reset Matter devices
Reset Thread devices
Reconfigure OTBR
Factory-reset IoT devices
```

Verify routing and firewall reachability first.

---

## IPv6 Address Changes After Migration

The physical NIC has a different network identity from the previous virtual adapter.

The exact IPv6 interface identifier may therefore change.

Validate:

```text
IPv6 prefix
Gateway
Routing
Firewall policy
Host-specific address objects
Matter connectivity
Thread connectivity
```

Do not assume that retaining the same IPv6 prefix guarantees that host-specific firewall rules continue to match.

---

## New HAOS Client Cannot Obtain DHCP

The production DHCP environment may reject unknown clients.

The physical Ethernet adapter has a different MAC address from the old virtual adapter.

Temporarily authorize the new client or relax the applicable restriction long enough to complete controlled onboarding.

After identifying and authorizing the physical client, restore the restrictive DHCP policy.

---

## Matter Devices Fail After Migration

Verify:

```text
Matter Server
IPv6
Thread
OTBR
mDNS
Network segmentation
Static routes
Firewall policy
Host-specific IPv6 policy
```

A successful Home Assistant configuration restore does not prove that the underlying Matter and Thread network paths are operational.

If the OTBR resides on a routed network, verify HAOS routing before changing Matter or Thread configuration.

---

## Backup Location Is Unavailable After Restore

A restored HAOS configuration may contain a reference to a backup storage location that is no longer available.

This does not by itself indicate that the Home Assistant restore failed.

Review:

```text
Settings
-> System
-> Backups
-> Automatic backup configuration
```

Remove obsolete destinations or replace them with the intended off-host backup location.

---

## Migration Must Be Rolled Back

Power off:

```text
Bare-metal HAOS system
```

Then start:

```text
Original Proxmox HAOS VM
```

Do not operate both simultaneously with the same production Home Assistant network identity.

---

# Evidence and Operational Records

Useful migration evidence includes:

```text
HAOS bare-metal migration evidence
|
+-- Source HAOS system information
|   |
|   +-- ha info
|
+-- Source HAOS network configuration
|   |
|   +-- ha network info
|   +-- ip route
|   +-- ip -6 route
|   +-- custom static routes
|
+-- Migration backup
|
+-- Target hardware inventory
|
+-- HAOS installation
|   |
|   +-- Generic x86-64 image
|   +-- Disk imaging validation
|
+-- Restore operation
|   |
|   +-- Settings and history
|   +-- Share folder
|   +-- SSL certificates
|   +-- Supervisor-managed applications
|   +-- ha jobs info
|
+-- Post-migration evidence
    |
    +-- ha info
    +-- ha network info
    +-- ha core info
    +-- ha core stats
    +-- ha supervisor info
    +-- ha hardware info
    +-- ha os info
    +-- HAOS host routing table
    +-- Home Assistant logs
```

Operational evidence containing network addresses, device identifiers, credentials, certificates, or internal infrastructure topology should remain in restricted storage.

The source and target outputs should be retained long enough to establish that the migration completed successfully.

---

# Migration Completion Criteria

The migration should not be considered complete until all critical validation items have passed.

* [x] Backup restoration completed successfully.
* [x] Home Assistant web interface is accessible.
* [ ] Existing credentials work.
* [x] HAOS is operational.
* [x] Home Assistant Core is operational.
* [x] System no longer identifies as `qemux86-64`.
* [x] System identifies as `generic-x86-64`.
* [x] Physical Ethernet interface is identified.
* [x] Production IPv4 configuration is correct.
* [x] Production gateway is correct.
* [x] DNS is operational.
* [x] IPv6 is operational.
* [x] Host Internet connectivity is functional.
* [x] Supervisor Internet connectivity is functional.
* [x] Required static routes have been restored.
* [x] Routed integrations recover after route restoration.
* [x] Dashboards are restored.
* [x] Integrations are fully validated.
* [x] Devices and entities are fully validated.
* [x] Automations and scripts are restored.
* [x] History is available.
* [x] Required Supervisor-managed applications are restored.
* [x] Matter devices are fully validated.
* [x] Thread connectivity is fully validated.
* [x] OTBR integration connectivity is operational.
* [x] Required mDNS discovery is operational.
* [x] Logs have been reviewed for migration-related errors.
* [x] Restrictive network client policy has been restored.
* [x] A new post-migration backup has been created.
* [x] Unavailable legacy backup destinations have been remediated.
* [x] Off-host backup has been configured and tested.
* [x] Original Proxmox HAOS VM remains available during the validation period.

Only after these requirements are satisfied should retirement of the previous Proxmox HAOS VM be considered.

---

# Final Architecture

After successful validation, the intended production architecture is:

```text
Dedicated Physical System
|
+-- CPU
|
+-- Memory
|
+-- Dedicated SSD
|
+-- Wired Ethernet
|   |
|   +-- Production network
|   |
|   +-- Required routed networks
|
+-- Home Assistant OS
    |
    +-- Home Assistant Supervisor
    |
    +-- Home Assistant Core
    |
    +-- Supervisor-managed applications
    |
    +-- Home Assistant configuration
    |
    +-- Automations
    |
    +-- Integrations
    |
    +-- Matter / Thread services
    |
    +-- Local encrypted backups
        |
        +-- Off-host backup storage
```

The Proxmox virtualization layer is no longer part of the normal Home Assistant production service path.

---

# Related Search Keywords

home-assistant, home-assistant-os, haos, bare-metal, baremetal, proxmox, proxmox-to-bare-metal, dell-optiplex, optiplex-7040, generic-x86-64, backup-restore, migration-backup, ha-cli, static-ip, static-route, segmented-network, vlan, network-routing, ipv6, ipv6-routing, matter, matter-over-thread, thread, otbr, open-thread-border-router, esphome, migration, rollback, home-automation

---

## Revision Control

| Version   | Date       | Summary                                                                                                                                                                                                                                                                                                             | Author      |
| --------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-09-02 | Initial documentation of the Home Assistant OS migration from Proxmox VE to dedicated bare-metal hardware, including backup restoration, host networking considerations, segmented-network routing, IPv6 identity dependencies, Matter and Thread validation, rollback procedures, and post-migration verification. | projectfong |
