# Home Assistant OS Migration from Proxmox VE to Bare Metal - Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Migrate Home Assistant OS from a Proxmox VE virtual machine to dedicated x86-64 hardware using the native Home Assistant backup and restore process.

The migration path is:

```text
Proxmox VE
|
+-- Existing HAOS VM
    |
    +-- Record system and network configuration
    |
    +-- Create migration backup
            |
            v
    Power off and retain VM
            |
            v
Dedicated x86-64 Hardware
|
+-- HAOS Generic x86-64
    |
    +-- Restore backup
    |
    +-- Validate network identity
    |
    +-- Restore custom routes
    |
    +-- Validate Matter / Thread
    |
    +-- Create new backup
```

A successful Home Assistant backup restore does not guarantee that host-level networking associated with the previous virtual NIC will be recreated on replacement hardware.

Pay particular attention to:

```text
IPv4 configuration
IPv6 configuration
Static routes
Firewall address objects
Host-specific firewall rules
Matter and Thread network paths
```

Keep the original Proxmox VM powered off and intact until the bare-metal system has passed validation.

---

## Requirements

- Existing Home Assistant OS VM on Proxmox VE.
- Supported x86-64 target system.
- UEFI firmware.
- Dedicated target SSD.
- Wired Ethernet recommended.
- Current Home Assistant backup.
- Backup recovery information if encryption is enabled.
- Independent copy of the migration backup.
- Administrative access to network infrastructure.
- Physical console access to the target system recommended.
- Record of any custom IPv4 or IPv6 routes.
- Record of firewall objects referencing HAOS addresses.

---

## 1. Record the Source System

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
```

The Proxmox installation should report a virtualization-specific machine type such as:

```text
qemux86-64
```

After migration, the bare-metal system should report:

```text
generic-x86-64
```

---

## 2. Record the Network Configuration

Run:

```bash
ha network info
```

Record privately:

```text
Interface
MAC address
IPv4 address
IPv4 gateway
IPv4 method
DNS servers
IPv6 addresses
IPv6 gateway
IPv6 method
```

Do not rely on:

```bash
ip addr
```

from the `Terminal & SSH` application as the authoritative HAOS host network configuration.

`Terminal & SSH` runs inside a Supervisor-managed container.

Use:

```bash
ha network info
```

for HAOS host addressing.

---

## 3. Record Custom Routes

If HAOS communicates with segmented networks, record all host routes before migration.

From the HAOS host shell:

```bash
ip route
```

For IPv6:

```bash
ip -6 route
```

If routes are persisted through NetworkManager:

```bash
nmcli connection show
```

Inspect the applicable connection:

```bash
nmcli -f ipv4.routes,ipv6.routes connection show "<connection-name>"
```

Record:

```text
Destination
Prefix
Next-hop gateway
Address family
Interface
NetworkManager connection
```

Example:

```text
<SEGMENTED_SUBNET>/<PREFIX> via <ROUTER_ADDRESS>
```

Do not assume custom routes will migrate automatically to a new physical NIC.

---

## 4. Record IPv6 and Firewall Dependencies

If firewall policy identifies HAOS using a specific IPv6 address, record that dependency.

Examples include:

```text
Host-specific /128 firewall objects
IPv6 ACLs
Inter-VLAN IPv6 rules
Matter / Thread firewall policies
```

A new physical NIC may receive a different IPv6 host address even when the system remains on the same `/64`.

Record these dependencies before migration rather than discovering them after Matter or Thread devices go offline.

---

## 5. Prepare Administrative Access

If the Proxmox console is currently an important HAOS administrative path, configure an alternative before migration.

For example:

```text
Terminal & SSH
```

Include it in the final backup.

Also retain physical console access to the bare-metal system:

```text
Monitor
Keyboard
HAOS local console
```

---

## 6. Create the Migration Backup

Immediately before cutover, create a manual Home Assistant backup.

Include all required:

```text
Home Assistant settings
History
Applications
Application data
Share folder
SSL certificates
```

Download the backup outside HAOS.

If backup encryption is enabled, retain the recovery information independently.

Do not keep the only copy on:

```text
Source HAOS VM
Target HAOS installation
Target SSD
```

---

## 7. Prepare the Bare-Metal System

Typical firmware configuration:

```text
Boot mode:   UEFI
Secure Boot: Disabled
SATA mode:   AHCI
```

Use settings appropriate for the actual hardware.

Install a dedicated SSD for HAOS.

Do not overwrite storage containing data that must be retained.

---

## 8. Install HAOS Generic x86-64

Download the Home Assistant OS:

```text
Generic x86-64
```

image.

This is a disk image rather than a conventional installer ISO.

Write it directly to the target SSD using an appropriate imaging utility.

```text
HAOS Generic x86-64
        |
        v
Disk Imaging Utility
        |
        v
Target SSD
```

Verify the target disk carefully before writing.

---

## 9. Perform an Initial Boot

Boot the new bare-metal system.

When practical, use an isolated or controlled test network first.

Confirm the Home Assistant onboarding interface appears.

Expected options include:

```text
Create my smart home
Upload backup
Home Assistant Cloud
```

This confirms:

```text
UEFI boot works
SSD is readable
HAOS initialized
Ethernet detected
Home Assistant onboarding available
```

---

## 10. Prepare the New Physical NIC

The physical NIC will have a different MAC address from the Proxmox VirtIO adapter.

If the network uses:

```text
DHCP reservations
Unknown-client blocking
MAC authorization
Port security
802.1X
```

authorize the new physical client before cutover.

If restrictions are temporarily relaxed, restore them afterward.

---

## 11. Shut Down the Proxmox VM

Before placing the replacement HAOS installation into production:

```text
Old Proxmox HAOS VM: OFF
New bare-metal HAOS: ON
```

Do not operate both systems simultaneously when they represent the same Home Assistant environment.

Keep the source VM:

```text
Powered off
Intact
Recoverable
```

---

## 12. Restore the Backup

From HAOS onboarding:

```text
Upload backup
```

Upload the migration backup and restore the required components.

Verify restore progress:

```bash
ha jobs info
```

A completed restore should report:

```text
done: true
progress: 100
errors: []
```

---

## 13. Verify Bare-Metal HAOS

Run:

```bash
ha info
```

Confirm:

```text
machine: generic-x86-64
state: running
supported: true
```

Verify the expected HAOS, Home Assistant Core, and Supervisor versions.

---

## 14. Verify Host Networking

Run:

```bash
ha network info
```

Verify:

```text
Physical interface
IPv4 address
IPv4 gateway
DNS
IPv6 address
IPv6 gateway
Host Internet
Supervisor Internet
```

Do not expect the physical interface name or MAC address to match the Proxmox VM.

---

## 15. Access the HAOS Host Shell

The normal `Terminal & SSH` shell is not the HAOS host operating-system shell.

For host-level NetworkManager work, use the physical HAOS console.

At:

```text
homeassistant login:
```

enter:

```text
root
```

At the HA CLI:

```text
login
```

The resulting host shell provides:

```bash
ip route
ip -6 route
nmcli connection show
```

---

## 16. Restore Missing Static Routes

From the HAOS host shell:

```bash
ip route
```

and, if applicable:

```bash
ip -6 route
```

Compare against the pre-migration baseline.

If a required route is missing, identify the current NetworkManager profile:

```bash
nmcli connection show
```

Restore the route through the appropriate physical-network connection.

Do not assume the old virtual-machine connection name still exists.

---

## 17. Check for Shared Failure Domains

If multiple integrations fail after migration, determine whether the affected devices share a routed network.

Example:

```text
HAOS local access:             Working
Internet access:               Working
Devices on local subnet:       Working
Devices on routed IoT subnet:  Failing
Multiple integrations:         Failing
```

Check routing before:

```text
Deleting integrations
Re-pairing devices
Resetting Matter devices
Resetting Thread devices
Reconfiguring OTBR
Factory-resetting IoT devices
```

A missing internal route can exist while:

```text
host_internet: true
supervisor_internet: true
```

---

## 18. Verify IPv6 Identity

Compare the new IPv6 configuration with the pre-migration network record.

A physical NIC can receive a different IPv6 host address even if the IPv6 prefix remains unchanged.

Example:

```text
Before:
fd00:1234:5678:10:aaaa:bbbb:cccc:dddd/64

After:
fd00:1234:5678:10:1111:2222:3333:4444/64
```

If firewall policy references the previous address as a `/128`, the replacement HAOS address may no longer match.

Resolve the difference according to the network design:

```text
Update infrastructure policy

or

Restore the established HAOS IPv6 identity
```

Then validate the resulting network path.

---

## 19. Verify Docker IPv6

For environments using Matter or Thread:

```bash
ha docker info
```

Check:

```text
enable_ipv6
```

If Docker IPv6 is required and disabled, configure it using the supported HAOS CLI for the installed release.

Example:

```bash
ha docker options --enable-ipv6=true
```

Verify afterward:

```bash
ha docker info
```

Expected when enabled:

```text
enable_ipv6: true
```

Docker IPv6 is only one part of the end-to-end IPv6 path.

---

## 20. Validate Matter and Thread

Verify:

```text
Matter Server
IPv6
Thread integration
OTBR
Thread credentials
mDNS
Multicast
Routing
Firewall policy
HAOS IPv6 identity
Matter devices
```

For Matter over Thread:

```text
Home Assistant
      |
      | IPv6
      v
Router / Firewall
      |
      | IPv6 routing and policy
      v
OpenThread Border Router
      |
      | Thread
      v
Matter over Thread Device
```

Do not factory-reset or recommission an offline Matter-over-Thread device until this path has been validated.

---

## 21. Test Thread IPv6 Reachability

When the routed IPv6 address of a representative Thread device is known:

```bash
ping -6 -c 4 <THREAD_DEVICE_IPV6>
```

A successful test should resemble:

```text
4 packets transmitted
4 packets received
0 percent packet loss
```

This validates the routed path from HAOS through the infrastructure network and OTBR to the Thread endpoint.

Do not publish actual production Thread IPv6 addresses.

---

## 22. Verify Core and Supervisor

Home Assistant Core:

```bash
ha core info
```

Supervisor:

```bash
ha supervisor info
```

Restore jobs:

```bash
ha jobs info
```

Confirm:

```text
Core running
Supervisor healthy
Supervisor supported
Restore completed without errors
```

Defer unrelated Home Assistant updates until migration validation is complete and a new backup exists.

---

## 23. Verify Hardware

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
Storage
Network hardware
USB devices
Serial devices
Audio devices
```

When supported, use persistent serial paths:

```text
/dev/serial/by-id/<DEVICE_IDENTIFIER>
```

instead of:

```text
/dev/ttyUSB0
```

Review command output before publishing because persistent hardware identifiers may be included.

---

## 24. Validate Home Assistant

Verify:

```text
Users
Dashboards
Areas
Devices
Entities
Integrations
Automations
Scripts
Helpers
Scenes
History
Recorder data
Certificates
Shared files
```

Also validate infrastructure-dependent integrations:

```text
ESPHome
Matter
Thread
OTBR
Voice services
Cameras
MQTT
DNS
mDNS
External APIs
```

Review:

```text
Settings
-> System
-> Logs
```

---

## 25. Restore Temporary Network Restrictions

If migration required temporarily changing:

```text
DHCP restrictions
Unknown-client blocking
Network admission controls
```

restore the intended policy.

Verify the new physical HAOS system remains operational afterward.

---

## 26. Repair Backup Destinations

After migration, review:

```text
Settings
-> System
-> Backups
```

A restored configuration may reference an unavailable backup location from the old environment.

Remove obsolete destinations and configure the intended off-host backup location.

Recommended architecture:

```text
Bare-Metal HAOS
|
+-- Local encrypted backups
|
+-- Off-host backup
    |
    +-- Dedicated backup storage
```

Use a dedicated backup account with only the permissions required for backup storage.

---

## 27. Create a Post-Migration Backup

After validation, create a new backup representing the working bare-metal configuration.

Store at least one copy outside the HAOS appliance.

---

## 28. Preserve Rollback

Keep the source Proxmox VM powered off until the bare-metal system has passed an appropriate validation period.

Rollback:

```text
Bare-metal problem
        |
        v
Power off bare-metal HAOS
        |
        v
Confirm production identity is free
        |
        v
Start original Proxmox HAOS VM
        |
        v
Validate Home Assistant
```

Never operate both systems simultaneously with the same production identity.

---

# Validation Checklist

```text
[ ] Migration backup stored independently
[ ] Backup recovery information available

[ ] Source HAOS information recorded
[ ] Source IPv4 configuration recorded
[ ] Source IPv6 configuration recorded
[ ] Custom IPv4 routes recorded
[ ] Custom IPv6 routes recorded
[ ] Firewall HAOS address objects recorded
[ ] Host-specific IPv6 policies recorded

[ ] Target HAOS boots
[ ] Source VM powered off
[ ] Backup restore completes without errors

[ ] Machine reports generic-x86-64
[ ] Physical NIC detected
[ ] IPv4 configuration correct
[ ] IPv6 configuration correct
[ ] IPv4 gateway correct
[ ] IPv6 gateway correct
[ ] DNS operational

[ ] Host Internet operational
[ ] Supervisor Internet operational

[ ] Custom IPv4 routes restored
[ ] Custom IPv6 routes restored
[ ] Firewall policy matches current HAOS identity

[ ] Docker IPv6 verified where required

[ ] Home Assistant Core operational
[ ] Supervisor healthy
[ ] Required applications restored
[ ] USB and serial hardware detected

[ ] Dashboards restored
[ ] Devices and entities restored
[ ] Integrations operational
[ ] Automations operational

[ ] Matter operational
[ ] Thread operational
[ ] OTBR operational
[ ] mDNS operational
[ ] Matter-over-Thread IPv6 path validated

[ ] Voice services operational
[ ] Logs reviewed

[ ] Temporary network exceptions removed
[ ] Backup destinations reviewed
[ ] New post-migration backup created
[ ] Off-host backup tested

[ ] Original Proxmox VM retained for rollback
```

---

# Key Commands

System:

```bash
ha info
```

Network:

```bash
ha network info
```

Docker:

```bash
ha docker info
```

Core:

```bash
ha core info
```

Supervisor:

```bash
ha supervisor info
```

Restore jobs:

```bash
ha jobs info
```

Hardware:

```bash
ha hardware info
```

Operating system:

```bash
ha os info
```

From the HAOS host shell:

```bash
ip route
ip -6 route
nmcli connection show
```

Inspect persisted routes:

```bash
nmcli -f ipv4.routes,ipv6.routes connection show "<connection-name>"
```

Thread IPv6 validation:

```bash
ping -6 -c 4 <THREAD_DEVICE_IPV6>
```

---

# Common Problems

## Multiple Integrations Fail After Migration

Check whether they share a remote routed network.

Then inspect:

```bash
ip route
ip -6 route
```

before modifying integrations.

---

## Matter-over-Thread Device Is Offline

Verify:

```text
HAOS IPv6 address
IPv6 gateway
Docker IPv6
IPv6 routes
Firewall objects
Firewall policy
OTBR
Thread network
Matter Server
```

Do not recommission the device until the network path has been validated.

---

## IPv6 Prefix Is Correct but Traffic Is Blocked

Check whether the firewall uses a host-specific `/128` object referring to the previous HAOS IPv6 address.

A valid new address on the same `/64` will not match the old `/128`.

---

## Internet Works but Segmented Devices Do Not

The following:

```text
host_internet: true
supervisor_internet: true
```

does not prove HAOS can reach every internal routed network.

Check host routes and firewall policy.

---

## `ip addr` Shows an Unexpected Container Address

If the command is being run inside `core-ssh`, use:

```bash
ha network info
```

instead.

---

## `nmcli` Is Missing

The command is probably being executed inside the `Terminal & SSH` container.

Use the HAOS host shell for NetworkManager administration.

---

# Security Notes

Maintain HAOS as a dedicated appliance.

Recommended controls include:

```text
Administrative access restricted to trusted networks
IoT segmentation retained
IPv6 retained for Matter and Thread where required
No unnecessary Internet exposure
Encrypted backups
Off-host backups
Dedicated backup credentials
Temporary migration exceptions removed after cutover
Firewall address objects reviewed after migration
Original VM kept offline during validation
```

Do not publish:

```text
Production IPv4 addresses
Production IPv6 addresses
MAC addresses
Internal gateways
Internal DNS servers
Static route details
Firewall address objects
Matter device addresses
Thread device addresses
Persistent hardware serial identifiers
Backup identifiers
Credentials
Tokens
Certificates
```

Keep those values in restricted operational documentation.

---

# Migration Lesson

The critical migration principle is:

```text
Home Assistant backup restore
        |
        +-- Restores Home Assistant configuration
        +-- Restores supported application data
        |
        X
        |
        +-- Does not prove host routes survived
        +-- Does not prove IPv6 identity survived
        +-- Does not prove firewall objects still match
        +-- Does not prove segmented-network connectivity
```

When HAOS moves to different hardware, validate the complete host network identity before rebuilding integrations.

---

# Full Documentation

This quickstart intentionally omits:

```text
Detailed migration history
Complete source and target hardware inventory
Extended backup discussion
Detailed Matter and Thread troubleshooting
Full NetworkManager analysis
Individual integration recovery records
Detailed security rationale
Evidence collection guidance
Extended HAOS administration notes
Full rollback discussion
```

See the full Home Assistant OS Migration from Proxmox VE to Bare Metal documentation for the complete migration and troubleshooting record.

---

## Related Search Keywords

home-assistant, home-assistant-os, haos, proxmox, bare-metal, generic-x86-64, backup-restore, migration-backup, ha-cli, static-route, segmented-network, vlan, ipv6, ipv6-routing, matter, matter-over-thread, thread, otbr, esphome, migration, rollback, home-automation

---

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-09-03 | Initial sanitized Home Assistant OS Proxmox-to-bare-metal migration quickstart. | projectfong |
