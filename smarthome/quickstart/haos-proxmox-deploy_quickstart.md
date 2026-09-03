# Home Assistant OS on Proxmox VE - Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Deploy Home Assistant OS (HAOS) as a dedicated Proxmox VE virtual machine using the official KVM/Proxmox QCOW2 image, configure networking, and optionally restore an existing Home Assistant backup.

Estimated time:

```text
20-45 minutes
```

This quickstart uses documentation-only infrastructure values. Replace them with values appropriate for the local environment.

## Requirements

Before starting:

- Working Proxmox VE host.
- Storage available for a new VM disk.
- Network bridge available for HAOS.
- Current HAOS KVM/Proxmox QCOW2 image.
- Current Home Assistant backup if migrating an existing installation.
- Administrative access to the Proxmox host.

Example values used below:

| Purpose | Example |
| --- | --- |
| VM ID | `200` |
| VM name | `vm-haos` |
| Storage | `vm-storage` |
| Bridge | `vmbr-admin` |
| Image directory | `/root/images/` |
| HAOS IPv4 | `192.0.2.54/24` |
| Gateway | `192.0.2.1` |
| DNS | `192.0.2.53` |

The IP addresses above are documentation examples and must be replaced with actual local network values.

## Quick Setup

### 1. Download the HAOS Image

Download the HAOS image for:

```text
KVM/Proxmox (.qcow2)
```

Place the compressed image on the Proxmox host.

Example:

```text
/root/images/haos_ova-<version>.qcow2.xz
```

HAOS uses a prebuilt virtual disk.

Do not treat the QCOW2 image as an installation ISO.

### 2. Decompress the Image

```bash
cd /root/images
unxz haos_ova-<version>.qcow2.xz
```

Verify:

```bash
ls -lh /root/images/haos_ova-<version>.qcow2
```

Expected result:

```text
The QCOW2 image exists and the .xz extension is no longer present.
```

### 3. Create the HAOS VM

Create a new VM shell or full-clone an existing VM template.

Example:

```text
VM ID: 200
Name:  vm-haos
```

Recommended starting configuration:

```text
BIOS:             OVMF (UEFI)
CPU type:         host
CPU cores:        4 or more
Memory:           4 GB or more
SCSI Controller:  VirtIO SCSI
Network Model:    VirtIO
```

If using a cloned template, remove the cloned operating system disk from the new VM.

Do not modify the original template.

### 4. Identify Proxmox Storage

Run:

```bash
pvesm status
```

Identify an active storage target suitable for VM disks.

Example:

```text
vm-storage
```

Do not assume the storage ID.

### 5. Import the HAOS Disk

Example:

```bash
qm importdisk 200 /root/images/haos_ova-<version>.qcow2 vm-storage
```

Verify:

```bash
qm config 200
```

The imported disk initially appears similar to:

```text
unused0: vm-storage:200/vm-200-disk-0.raw
```

This is expected.

### 6. Attach the Imported Disk

In Proxmox:

```text
VM 200
-> Hardware
-> Unused Disk 0
-> Edit
```

Attach it as:

```text
Bus/Device:
SCSI 0

SCSI Controller:
VirtIO SCSI
```

Verify:

```bash
qm config 200
```

Expected relevant configuration:

```text
scsi0: vm-storage:200/vm-200-disk-0.raw
scsihw: virtio-scsi-pci
```

### 7. Configure UEFI

Configure:

```text
VM 200
-> Hardware
-> BIOS
-> OVMF (UEFI)
```

Verify:

```bash
qm config 200
```

Expected:

```text
bios: ovmf
```

### 8. Configure the Boot Disk

Set the HAOS disk as the boot device:

```bash
qm set 200 --boot order=scsi0
```

Verify:

```bash
qm config 200 | grep -E '^(bios|boot|scsi0|efidisk)'
```

Expected relevant output:

```text
bios: ovmf
boot: order=scsi0
scsi0: vm-storage:200/vm-200-disk-0.raw
```

### 9. Verify VM Configuration

Before first boot, verify:

```text
BIOS:
OVMF (UEFI)

CPU:
host

Memory:
4 GB or more

Disk:
SCSI 0

SCSI Controller:
VirtIO SCSI

Network:
VirtIO

Boot:
scsi0
```

Run:

```bash
qm config 200
```

Correct any unexpected disk or boot configuration before starting HAOS.

### 10. Start HAOS

Start the VM and immediately open the Proxmox console.

A successful boot should reach the HAOS console:

```text
ha >
```

HAOS may initially display:

```text
Home Assistant Core: landingpage
```

This can be normal while Home Assistant Core is being prepared.

### 11. Check Networking

From:

```text
ha >
```

run:

```bash
network info
```

Identify the active interface.

Example:

```text
enp0s18
```

If DHCP provides the intended address, continue to Home Assistant initialization.

If static IPv4 is required, continue with the next step.

### 12. Configure Static IPv4

Example only:

```bash
network update enp0s18 --ipv4-method static --ipv4-address 192.0.2.54/24 --ipv4-gateway 192.0.2.1 --ipv4-nameserver 192.0.2.53
```

Replace:

```text
enp0s18
192.0.2.54/24
192.0.2.1
192.0.2.53
```

with the actual interface and network configuration.

Verify:

```bash
network info
```

Expected:

```text
IPv4 method: static
IPv4 ready: true
```

Do not configure HAOS networking using:

```text
/etc/netplan/
/etc/network/interfaces
```

HAOS networking is managed through its own management interface.

### 13. Open Home Assistant

Browse to:

```text
http://<haos-ip>:8123
```

During initial startup, Home Assistant may display:

```text
Preparing Home Assistant
```

Allow initialization to complete.

Do not reboot HAOS simply because initialization takes several minutes.

## Restore an Existing Home Assistant Backup

Skip this section for a new Home Assistant installation.

### 14. Prepare the Backup

Before migration, create a current backup from the existing Home Assistant environment.

Keep the previous Home Assistant installation intact until HAOS has been validated.

Do not operate the old and new Home Assistant instances simultaneously using the same production network identity or device integrations.

### 15. Restore During Onboarding

During HAOS onboarding, select the option to restore an existing Home Assistant backup.

Upload the current backup and begin restoration.

The migration path is:

```text
Existing Home Assistant
        |
        v
Home Assistant Backup
        |
        v
HAOS Onboarding
        |
        v
Restore
        |
        v
Home Assistant OS
```

During restoration:

```text
Do not reboot HAOS.
Do not restart Supervisor.
Do not restart the previous Home Assistant instance.
Allow Home Assistant to restart automatically.
```

Restore time depends on backup size, database size, storage performance, and integration initialization.

## Validate

After HAOS starts or the backup restoration completes, verify:

```text
[ ] Home Assistant frontend loads
[ ] Existing user account works
[ ] Dashboards load
[ ] Devices and services are present
[ ] Expected entities are present
[ ] Automations are present
[ ] Scripts are present
[ ] Integrations initialize
[ ] Static network configuration remains correct
[ ] Home Assistant logs contain no unexpected critical errors
```

Review:

```text
Settings
-> System
-> Logs
```

Also verify HAOS networking:

```bash
network info
```

## Common Problems

### Imported Disk Shows as Unused

Observed:

```text
unused0: vm-storage:200/vm-200-disk-0.raw
```

This is expected immediately after:

```bash
qm importdisk
```

Attach the disk through:

```text
VM
-> Hardware
-> Unused Disk 0
-> SCSI 0
```

Then verify:

```bash
qm config 200
```

### VM Does Not Boot HAOS

Check:

```bash
qm config 200
```

Verify:

```text
bios: ovmf
boot: order=scsi0
```

If the boot order is incorrect:

```bash
qm set 200 --boot order=scsi0
```

### HAOS Has No IPv4 Address

Check:

```bash
network info
```

If the network does not provide DHCP, configure the required static address.

Example:

```bash
network update enp0s18 --ipv4-method static --ipv4-address 192.0.2.54/24 --ipv4-gateway 192.0.2.1 --ipv4-nameserver 192.0.2.53
```

Replace all documentation values before use.

### Home Assistant Shows `landingpage`

Observed:

```text
Home Assistant Core: landingpage
```

HAOS may still be preparing Home Assistant Core.

Allow initialization to continue before treating this as a failure.

## Rollback

When migrating an existing Home Assistant deployment:

```text
New HAOS deployment
        |
        v
Restore backup
        |
        v
Validate HAOS
        |
        +---- Success ----> Retire previous deployment
        |
        `---- Failure ----> Stop HAOS and use previous deployment
```

Keep the previous Home Assistant installation stopped but intact until the new HAOS deployment has been validated.

Do not run both installations simultaneously with conflicting network identities or integrations.

## Security Notes

HAOS should be treated as a dedicated Home Assistant appliance.

Do not use HAOS as a general-purpose server for:

```text
Unrelated Docker Compose workloads
General Linux packages
Unrelated infrastructure services
General-purpose application hosting
```

Restrict access to:

```text
Proxmox administration
Home Assistant administration
HAOS administrative interfaces
```

according to the surrounding network security architecture.

Before publishing Proxmox or HAOS command output, remove:

```text
Production IP addresses
IPv6 addresses
MAC addresses
Internal hostnames
Bridge names
Storage identifiers
VM IDs where sensitive
Administrative tags
Internal DNS names
Gateway addresses
Backup identifiers
PCI or USB passthrough identifiers
```

## Key Commands

Check Proxmox storage:

```bash
pvesm status
```

Decompress HAOS:

```bash
unxz haos_ova-<version>.qcow2.xz
```

Import the HAOS disk:

```bash
qm importdisk <vm-id> /path/to/haos_ova-<version>.qcow2 <storage-id>
```

Display VM configuration:

```bash
qm config <vm-id>
```

Configure the boot disk:

```bash
qm set <vm-id> --boot order=scsi0
```

Check HAOS networking:

```bash
network info
```

Configure static IPv4:

```bash
network update <interface> --ipv4-method static --ipv4-address <address/prefix> --ipv4-gateway <gateway> --ipv4-nameserver <dns-server>
```

## Full Documentation

This quickstart intentionally omits the detailed architecture, deployment rationale, sanitization reference, evidence model, HAOS administration discussion, extended troubleshooting, and migration background.

See the full Home Assistant OS deployment on Proxmox VE documentation for those details.

## Related Search Keywords

home-assistant, home-assistant-os, haos, proxmox, kvm, qcow2, qm-importdisk, ovmf, uefi, virtio-scsi, static-ip, ha-cli, backup-restore, migration, docker-to-haos

## Revision Control

| Version | Date | Summary | Author |
| -------- | ---- | ------- | ------ |
| **1.0.0** | 2026-08-31 | Initial Home Assistant OS on Proxmox VE quickstart. | projectfong |
