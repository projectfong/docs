# Home Assistant Matter over Segmented Wi-Fi with FortiGate and Avahi

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

This document describes the implementation of Matter-over-Wi-Fi with Home Assistant across a segmented network, where Matter devices reside on an isolated IoT SSID while Home Assistant resides on a separate trusted network. The architecture preserves Layer 3 segmentation while adding internal IPv6, Matter routing, and controlled mDNS reflection. The design follows established cybersecurity frameworks and best practices, including concepts from NIST SP 800-171, with emphasis on segmentation, least-privilege traffic flows, secure design, and independent validation.

This is a sanitized version intended for documentation, public repositories, Wiki.js, Gitea, and other locations where production addressing and infrastructure identifiers should not be disclosed.

All network addresses and hardware-specific identifiers shown below are documentation examples and do not represent the production environment.

---

# 1. Scope

This document begins with the Matter implementation on the existing Home Assistant OS installation.

It does not cover the earlier migration from Home Assistant Container to Home Assistant OS.

Covered components include:

| Component         | Function                                                    |
| ----------------- | ----------------------------------------------------------- |
| Home Assistant OS | Main automation platform                                    |
| Matter Server     | Home Assistant Matter controller                            |
| FortiGate         | Routing, firewalling, IPv6 RA/SLAAC, and FortiAP management |
| `iot-matter`      | Sanitized name for the isolated IoT Wi-Fi SSID              |
| `vm-avahi`        | Dedicated mDNS reflector                                    |
| USB Wi-Fi adapter | Direct Layer 2 interface from `vm-avahi` to the IoT SSID    |
| Matter bulb       | Matter-over-Wi-Fi endpoint                                  |
| Android phone     | Android Matter commissioner                                 |

Validation Result: The documented architecture successfully supports Matter commissioning and operational control across segmented networks.

---

# 2. Sanitization Reference

The following documentation-only addressing is used throughout this document.

| Purpose                     | Documentation Value      |
| --------------------------- | ------------------------ |
| Home Assistant IPv4 network | `192.0.2.0/24`           |
| IoT IPv4 network            | `198.51.100.0/24`        |
| Home Assistant IPv6 network | `fd42:1234:5678:10::/64` |
| IoT IPv6 network            | `fd42:1234:5678:20::/64` |
| Internal IPv6 allocation    | `fd42:1234:5678::/48`    |
| IoT SSID                    | `iot-matter`             |
| Home Assistant host         | `192.0.2.54`             |
| Avahi HA interface          | `192.0.2.55`             |
| Avahi IoT interface         | `198.51.100.57`          |
| HA network gateway          | `192.0.2.1`              |
| IoT network gateway         | `198.51.100.1`           |

Hardware MAC addresses, BSSIDs, device-specific IPv6 interface identifiers, phone addresses, and other unnecessary infrastructure identifiers have been removed or generalized.

Next Step: Substitute local production values only in private operational documentation.

---

# 3. Final Architecture

The existing IPv4 architecture remains intact.

```text
                         FortiGate
                             |
             +---------------+---------------+
             |                               |
             |                               |
      Home Assistant                  iot-matter SSID
         Network
             |                               |
       192.0.2.0/24                  198.51.100.0/24
fd42:1234:5678:10::/64       fd42:1234:5678:20::/64
             |                               |
             |                               |
       HAOS 192.0.2.54                Matter Devices
             |                               |
             |                               |
             +--------- vm-avahi ------------+
                       |        |
                    ens18      Wi-Fi
                       |        |
                 192.0.2.55   198.51.100.57
```

The Avahi VM has interfaces on both networks, but it is not a router.

```text
IPv4 forwarding: OFF
IPv6 forwarding: OFF
NAT:             NONE
```

The FortiGate remains responsible for routed traffic between the networks.

Validation Result: The reflector is dual-homed without creating an alternate Layer 3 path around the firewall.

---

# 4. Network Addressing

## 4.1 Home Assistant Network

```text
Network:       192.0.2.0/24
HAOS:          192.0.2.54
vm-avahi:      192.0.2.55
Gateway:       192.0.2.1
```

## 4.2 IoT Network

```text
SSID:          iot-matter
Network:       198.51.100.0/24
Gateway:       198.51.100.1
vm-avahi:      198.51.100.57
```

Validation Result: Home Assistant and Matter devices remain on separate IPv4 networks.

---

# 5. IPv6 ULA Design

Matter requires usable local IPv6 connectivity.

Instead of introducing public IPv6, an internal ULA allocation is used:

```text
fd42:1234:5678::/48
```

The relevant networks receive separate `/64` prefixes.

## 5.1 Home Assistant Network

```text
fd42:1234:5678:10::/64
```

FortiGate:

```text
fd42:1234:5678:10::1/64
```

## 5.2 IoT Network

```text
fd42:1234:5678:20::/64
```

FortiGate:

```text
fd42:1234:5678:20::1/64
```

No public IPv6 connectivity is required.

No IPv6 Internet access is required.

No DHCPv6 server is required.

IPv6 addressing is provided using:

```text
SLAAC
Router Advertisements
```

Validation Result: Matter receives routable internal IPv6 without requiring public IPv6 exposure.

---

# 6. Example IPv6 Addresses

Exact production interface identifiers are intentionally omitted.

Example HAOS address:

```text
fd42:1234:5678:10::54/64
```

Example `vm-avahi` HA-side address:

```text
fd42:1234:5678:10::55/64
```

Example `vm-avahi` IoT-side address:

```text
fd42:1234:5678:20::57/64
```

Actual SLAAC addresses normally contain interface-specific host portions and should not be published unnecessarily.

---

# 7. Initial Matter Problem

The Matter bulb could join the IoT SSID, but Home Assistant commissioning failed.

The Android Matter interface displayed errors similar to:

```text
Connecting to Home Assistant

Something went wrong
```

At another point Android displayed:

```text
Checking network connectivity to iot-matter

Can't reach device

Make sure your phone is connected to Wi-Fi
```

even though the Android commissioner was connected to the IoT SSID.

This indicated that Wi-Fi association alone was not sufficient to prove end-to-end Matter commissioning functionality.

---

# 8. FortiAP Intra-SSID Isolation Problem

The IoT SSID had client isolation enabled.

This prevented the phone and Matter device from communicating locally during commissioning.

```text
Android Commissioner
        |
        X
        |
   Matter Device
```

Both devices were associated with the same SSID, but intra-SSID isolation prevented the required local communication.

## Cause

FortiAP client isolation intentionally prevents wireless clients on the same SSID from communicating directly.

## Fix

Temporarily disable FortiAP intra-SSID/client isolation during Matter commissioning.

Once disabled, commissioning progresses significantly farther.

Validation Result: Local commissioner-to-device communication succeeds after removing the Layer 2 isolation barrier.

---

# 9. Why IPv6 Was Required

The existing environment was primarily IPv4.

Matter depends heavily on IPv6 for operational communication and discovery.

The solution is not to redesign the entire environment for IPv6.

IPv6 is enabled only on:

```text
Home Assistant network
IoT Matter network
```

using internal ULA prefixes.

This provides Matter with routable IPv6 while leaving the existing IPv4 architecture intact.

---

# 10. FortiGate IPv6 Features

The relevant FortiGate features include:

```text
IPv6
Multicast Policy
Advanced Routing
```

Normal IPv4 multicast routing or PIM is not used as the Matter mDNS solution.

mDNS reflection is handled by Avahi.

Next Step: Configure IPv6 addressing and Router Advertisements on the two Matter-relevant network interfaces.

---

# 11. FortiGate SLAAC Configuration

## 11.1 IoT Interface

Example IPv4 configuration:

```text
198.51.100.1/24
```

Example IPv6 configuration:

```text
fd42:1234:5678:20::1/64
```

Router Advertisement:

```text
Enabled
```

Advertised prefix:

```text
fd42:1234:5678:20::/64
```

DHCPv6:

```text
Disabled
```

Example sanitized FortiGate configuration:

```text
config system interface
    edit "IoT"
        set vdom "root"
        set ip 198.51.100.1 255.255.255.0
        set allowaccess ping
        set type vap-switch
        set description "For IoT only"
        set device-identification enable
        set role lan
        set ip-managed-by-fortiipam disable
        config ipv6
            set ip6-address fd42:1234:5678:20::1/64
            set ip6-send-adv enable
            config ip6-prefix-list
                edit fd42:1234:5678:20::/64
                next
            end
        end
    next
end
```

The exact production interface and SSID names are intentionally excluded.

The VAP mapping can be verified with:

```bash
diagnose wireless-controller wlac -c vap
```

Validation Result: The IoT segment advertises the internal Matter IPv6 prefix using SLAAC.

---

# 12. Home Assistant Network SLAAC

The HA network receives:

```text
fd42:1234:5678:10::/64
```

with the FortiGate interface:

```text
fd42:1234:5678:10::1/64
```

HAOS is configured for automatic IPv6.

Run:

```bash
network info
```

Expected sanitized output:

```text
ipv6:
  address:
  - fd42:1234:5678:10::<host-id>/64
  - fe80::<link-local>/64
  method: auto
  ready: true
```

The exact SLAAC host identifier is intentionally omitted.

Validation Result: `method: auto` and `ready: true` confirm that SLAAC is operational.

---

# 13. Matter Firewall Policy

Matter traffic is restricted instead of allowing arbitrary communication between networks.

Matter operational traffic uses:

```text
TCP 5540
UDP 5540
```

Home Assistant frontend/API access remains separate:

```text
TCP 8123
```

## 13.1 HAOS to IoT

The IPv6 policy permits:

```text
Source:
HAOS IPv6

Destination:
fd42:1234:5678:20::/64

Service:
Matter TCP/UDP 5540

NAT:
OFF
```

## 13.2 IoT to HAOS

The reverse Matter policy permits:

```text
Source:
fd42:1234:5678:20::/64

Destination:
HAOS IPv6

Service:
Matter TCP/UDP 5540

NAT:
OFF
```

Validation Result: Matter receives the required routed service without introducing unrestricted inter-network access.

---

# 14. IPv6 Routing Validation

Before troubleshooting mDNS, IPv6 routing should be tested independently.

A device on the Home Assistant network should successfully reach a test endpoint on the IoT network using IPv6.

This validates:

```text
SLAAC                PASS
IPv6 addressing      PASS
FortiGate routing    PASS
IPv6 firewall        PASS
IPv6 unicast         PASS
```

This test is important because it separates ordinary IPv6 routing problems from multicast discovery problems.

---

# 15. Matter Server Log Analysis

Matter Server logs provide critical evidence during troubleshooting.

The important information is the sequence of events rather than environment-specific device addresses or identifiers.

Matter Server should select the expected Home Assistant network interface.

Example:

```text
Primary interface: enp0s18
```

Next Step: Review commissioning logs and identify exactly which Matter stage fails.

---

# 16. Initial Matter Communication

During troubleshooting, Matter Server receives an operational endpoint from Android.

Sanitized example:

```text
Operational address for undefined set to udp://198.51.100.<device>:5540
```

It then successfully establishes PASE:

```text
Establish PASE to device
Paired successfully
```

The commissioning sequence proceeds through operations such as:

```text
GetInitialData
GeneralCommissioning.ArmFailsafe
GeneralCommissioning.ConfigureRegulatoryInformation
OperationalCredentials.DeviceAttestation
OperationalCredentials.Certificates
addTrustedRootCertificate
addNoc
```

The important success condition is:

```text
addNoc statusCode: 0
```

This proves:

```text
Matter port 5540 routing        WORKED
PASE                            WORKED
Device attestation              WORKED
Certificate installation        WORKED
NOC installation                WORKED
```

Therefore, a subsequent failure is not automatically evidence of basic firewall connectivity failure.

---

# 17. Operational Reconnect Failure

Immediately afterward Matter Server enters the reconnect phase.

Example:

```text
Executing commissioning step 18.1: Reconnect
```

Then:

```text
IpServiceStatus: Resolving (no address known)
```

Eventually:

```text
Operative reconnection with device failed
peer-unreachable
```

This identifies a different failure domain.

Validation Result: Credential provisioning succeeds, but operational service discovery fails.

---

# 18. Root Cause: Matter Operational mDNS Discovery

After Matter credentials are installed, the device transitions into normal operational discovery.

The Matter device advertises its service using DNS-SD/mDNS.

However, mDNS is link-local multicast.

The Matter device resides on:

```text
iot-matter
```

while Home Assistant resides on:

```text
192.0.2.0/24
```

The FortiGate can route normal IPv4 and IPv6 packets between those networks, but the Matter mDNS advertisement remains on the IoT Layer 2 segment.

Matter Server therefore reports:

```text
no address known
```

during operational reconnect.

The failure sequence becomes:

```text
Initial direct Matter IP communication     PASS
PASE                                       PASS
Attestation                                PASS
NOC installation                           PASS
Operational mDNS discovery                 FAIL
Operational reconnect                      FAIL
```

## Cause

Routed IPv4/IPv6 connectivity does not automatically forward link-local mDNS discovery across Layer 3 boundaries.

## Fix

Deploy a dedicated mDNS reflector with direct presence on both relevant Layer 2 networks.

---

# 19. Why FortiGate Multicast Routing Was Not the Fix

mDNS uses:

```text
IPv4 multicast: 224.0.0.251
IPv6 multicast: ff02::fb
UDP:            5353
```

This is link-local discovery traffic.

Traditional multicast routing/PIM is not equivalent to an mDNS reflector.

Instead of converting the FortiGate into a multicast-routing solution, a dedicated Avahi reflector is deployed.

The FortiGate remains responsible for:

```text
Layer 3 routing
Firewall policy
IPv6 Router Advertisements
Network segmentation
```

Avahi is responsible only for:

```text
mDNS reflection
```

Validation Result: Routing and service discovery remain separate functions with clearly defined trust boundaries.

---

# 20. vm-avahi

A dedicated Debian VM is created on Proxmox.

Example configuration:

```text
Name:        vm-avahi
CPU:         2 vCPU
RAM:         2 GiB
Disk:        8 GiB
OS:          Debian 12
```

The production VM identifier is intentionally omitted.

The VM has:

```text
VirtIO NIC -> Home Assistant network
USB Wi-Fi -> IoT Matter SSID
```

---

# 21. Why 2 GiB RAM Was Used

The VM initially receives:

```text
2 GiB RAM
```

This provides adequate installation and troubleshooting headroom.

After installation, memory can potentially be reduced because Avahi itself has minimal resource requirements.

Any resource reduction should be validated before considering the configuration stable.

---

# 22. USB Wi-Fi Passthrough

A USB Wi-Fi adapter is passed directly into the Avahi VM through Proxmox.

Sanitized topology:

```text
Proxmox host
     |
USB passthrough
     |
vm-avahi
     |
USB Wi-Fi adapter
     |
iot-matter
```

The exact USB vendor/product identifier, adapter MAC address, and other hardware fingerprints are intentionally omitted.

The Proxmox host does not need to participate directly in the IoT Wi-Fi network.

Validation Result: The VM receives direct Layer 2 presence on the isolated wireless segment.

---

# 23. Debian Installer IPv6 Problem

During Debian installation, the installer may detect the IPv6 SLAAC address and conclude that networking is configured.

However, the ULA IPv6 network intentionally has no IPv6 Internet connectivity.

The installer can subsequently report:

```text
Bad archive mirror
```

The actual condition is:

```text
IPv6 local connectivity: YES
IPv6 Internet:           NO
IPv4 configuration:      MISSING
```

## Cause

Successful local SLAAC does not imply IPv6 Internet reachability.

## Fix

Return to the Debian network configuration screen and manually configure IPv4.

Once IPv4 is configured correctly, package installation can proceed normally.

---

# 24. HA-Side Interface

The VirtIO interface becomes:

```text
ens18
```

Example configuration:

```text
IPv4:    192.0.2.55/24
Gateway: 192.0.2.1
IPv6:    SLAAC
```

Expected sanitized state:

```text
ens18 UP
192.0.2.55/24
fd42:1234:5678:10::<host-id>/64
fe80::<link-local>/64
```

IPv4 default route:

```text
default via 192.0.2.1 dev ens18
```

Validation Result: The trusted-side interface remains the VM's default routed path.

---

# 25. USB Wi-Fi Driver Failure

Initially Debian may load an older staging driver for the USB Wi-Fi adapter.

The interface can appear in:

```bash
ip link
```

while:

```bash
iw dev
```

returns nothing.

Likewise:

```bash
iw phy
```

may return nothing.

This means Linux created a network interface but did not expose a usable cfg80211/nl80211 wireless PHY.

---

# 26. wpa_supplicant Failure

An association attempt using nl80211 may produce:

```text
nl80211: Driver does not support authentication/association or connect commands
```

Trying a legacy backend may also fail.

## Cause

The loaded kernel driver does not expose the wireless capabilities required by `wpa_supplicant`.

## Fix

Correct the driver/kernel problem before changing Wi-Fi credentials or firewall configuration.

---

# 27. Kernel Driver Hang

Kernel logs may show driver threads becoming blocked or command execution failures.

Example generalized symptoms:

```text
wireless command thread blocked
driver command execution failed
```

These symptoms implicate the kernel wireless driver rather than the Wi-Fi password, FortiAP, or USB passthrough.

---

# 28. Alternate Driver Attempt

A compatible in-kernel wireless driver can be tested after unloading the problematic staging driver.

Example workflow:

```bash
modinfo rtl8xxxu
modprobe -r r8188eu
modprobe rtl8xxxu
lsusb -t
```

If:

```text
Driver=
```

remains empty for the adapter, the device is still unbound.

Validation Result: Do not proceed with Wi-Fi configuration until a functional driver is actually bound.

---

# 29. Debian Bookworm Backports Fix

Bookworm backports can provide a newer kernel and Realtek firmware.

Add:

```text
deb http://deb.debian.org/debian bookworm-backports main contrib non-free-firmware
```

Then run:

```bash
apt update

apt install -t bookworm-backports \
  linux-image-amd64 \
  firmware-realtek
```

Reboot the VM.

Verify:

```bash
uname -r
```

The tested deployment used a Debian 12 backports 6.12-series kernel.

Validation Result: Confirm the new kernel is active before continuing.

---

# 30. Wireless Driver Validation

After the kernel upgrade, verify driver binding:

```bash
lsusb -t
```

Expected relevant result:

```text
Driver=rtl8xxxu
```

Then run:

```bash
sudo iw phy
sudo iw dev
```

A functional configuration returns a real wireless PHY.

Example:

```text
Wiphy phy0
```

The wireless interface should report:

```text
type managed
```

and support operations such as:

```text
authenticate
associate
connect
disconnect
```

Validation Result: A visible PHY confirms that the kernel wireless stack is now usable by `wpa_supplicant`.

---

# 31. Wi-Fi Interface

The adapter receives a predictable or hardware-derived Linux interface name.

Example:

```text
wlx<adapter-mac-derived-name>
```

The exact production interface name and MAC address are intentionally omitted.

Use:

```bash
iw dev
```

to identify the actual interface before creating systemd services.

---

# 32. wpa_supplicant Configuration

Create:

```text
/etc/wpa_supplicant/wpa_supplicant-iot.conf
```

Example:

```ini
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=0
country=US

network={
    ssid="iot-matter"
    psk=<64-character-derived-PSK>
}
```

Do not publish the real PSK.

Remove any plaintext password comment generated by:

```bash
wpa_passphrase
```

Protect the configuration:

```bash
chmod 600 /etc/wpa_supplicant/wpa_supplicant-iot.conf
```

Validation Result: The Wi-Fi credential remains locally protected and is not embedded in public documentation.

---

# 33. Missing Network Block Problem

Check configured networks:

```bash
wpa_cli -i <wifi-interface> list_networks
```

If the command returns only:

```text
network id / ssid / bssid / flags
```

verify that the configuration actually contains a:

```ini
network={
}
```

block.

## Cause

The control configuration alone does not define a Wi-Fi network.

## Fix

Add the intended SSID and PSK inside the `network={}` block.

---

# 34. Duplicate wpa_supplicant Problem

Repeated errors similar to:

```text
nl80211: kernel reports: Match already configured
```

can occur when multiple `wpa_supplicant` instances contend for the same interface.

## Cause

More than one supplicant process is managing the adapter.

## Fix

Stop the duplicate instance before launching the dedicated configuration.

Validation Result: Exactly one intended `wpa_supplicant` instance should manage the IoT Wi-Fi adapter.

---

# 35. Successful Wi-Fi Association

Verify configured networks:

```bash
wpa_cli -i <wifi-interface> list_networks
```

Expected:

```text
0       iot-matter       any       [CURRENT]
```

Verify the link:

```bash
iw dev <wifi-interface> link
```

Expected generalized output:

```text
Connected to <redacted-bssid>
SSID: iot-matter
freq: 2.4-GHz-channel
signal: <signal-strength>
```

The actual BSSID is intentionally excluded.

Validation Result: `[CURRENT]` and `Connected` confirm successful Layer 2 association.

---

# 36. Automatic Wi-Fi Startup

Create:

```text
/etc/systemd/system/wpa_supplicant-iot.service
```

Example:

```ini
[Unit]
Description=WPA supplicant for IoT Matter network
Wants=network-pre.target
Before=network.target
After=sys-subsystem-net-devices-<wifi-interface>.device
BindsTo=sys-subsystem-net-devices-<wifi-interface>.device

[Service]
Type=simple
ExecStart=/usr/sbin/wpa_supplicant -D nl80211 -i <wifi-interface> -c /etc/wpa_supplicant/wpa_supplicant-iot.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Replace:

```text
<wifi-interface>
```

with the actual local interface name.

Enable:

```bash
systemctl daemon-reload
systemctl enable --now wpa_supplicant-iot.service
```

Verify:

```bash
systemctl status wpa_supplicant-iot.service
```

Expected:

```text
Active: active (running)
```

The logs should contain a successful connection event.

Validation Result: Wi-Fi association persists across VM reboots.

---

# 37. IoT DHCP

Request an IPv4 lease:

```bash
dhclient -v <wifi-interface>
```

Sanitized expected sequence:

```text
DHCPOFFER of 198.51.100.57 from 198.51.100.1
DHCPACK of 198.51.100.57
```

Result:

```text
198.51.100.57/24
```

These addresses are documentation examples only.

---

# 38. Long DHCP Lease Behavior

A DHCP server can return an extremely long lease duration.

For practical operational purposes, such a lease may behave similarly to a persistent assignment while remaining technically DHCP-controlled.

A DHCP reservation should be used if deterministic assignment is required.

The production adapter MAC address is intentionally omitted.

Validation Result: Use a reservation rather than publishing or hard-coding a hardware identifier unnecessarily.

---

# 39. Automatic DHCP

Wi-Fi association alone does not restore IPv4 after reboot.

Create:

```text
/etc/systemd/system/dhclient-iot.service
```

Example:

```ini
[Unit]
Description=DHCP client for IoT Matter network
Requires=wpa_supplicant-iot.service
After=wpa_supplicant-iot.service

[Service]
Type=oneshot
ExecStart=/usr/sbin/dhclient -4 -v <wifi-interface>
ExecStop=/usr/sbin/dhclient -4 -r <wifi-interface>
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
systemctl daemon-reload
systemctl enable --now dhclient-iot.service
```

Validation Result: Both Layer 2 association and IPv4 configuration automatically recover after reboot.

---

# 40. Final vm-avahi Addressing

Expected sanitized addressing:

```text
ens18
  192.0.2.55/24
  fd42:1234:5678:10::<host-id>/64

<wifi-interface>
  198.51.100.57/24
  fd42:1234:5678:20::<host-id>/64
```

IPv4 routing:

```text
default via 192.0.2.1 dev ens18
192.0.2.0/24 dev ens18
198.51.100.0/24 dev <wifi-interface>
```

Critically, the Wi-Fi interface must not become the IPv4 default route.

Validation Result: Management/default traffic remains on the trusted-side interface.

---

# 41. Disable Routing on vm-avahi

Because the VM is dual-homed, it must not become an alternate router between the networks.

Create:

```text
/etc/sysctl.d/99-avahi-reflector.conf
```

Contents:

```text
net.ipv4.ip_forward=0
net.ipv6.conf.all.forwarding=0
net.ipv6.conf.default.forwarding=0
```

Apply:

```bash
sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv6.conf.default.forwarding
```

Expected:

```text
0
0
0
```

No NAT is configured.

Validation Result: The dual-homed VM cannot function as a normal IPv4 or IPv6 router.

---

# 42. Avahi Installation

Install:

```bash
apt install avahi-daemon avahi-utils -y
```

Back up the default configuration:

```bash
cp /etc/avahi/avahi-daemon.conf \
   /etc/avahi/avahi-daemon.conf.bak
```

Next Step: Restrict Avahi to the two interfaces required for Matter discovery.

---

# 43. Avahi Configuration

File:

```text
/etc/avahi/avahi-daemon.conf
```

Relevant configuration:

```ini
[server]
use-ipv4=yes
use-ipv6=yes
allow-interfaces=ens18,<wifi-interface>
ratelimit-interval-usec=1000000
ratelimit-burst=1000

[wide-area]
enable-wide-area=yes

[publish]
publish-hinfo=no
publish-workstation=no

[reflector]
enable-reflector=yes
reflect-ipv=no

[rlimits]
```

The critical controls are:

```ini
allow-interfaces=ens18,<wifi-interface>
```

and:

```ini
enable-reflector=yes
```

This restricts Avahi to exactly the intended networks.

Validation Result: mDNS reflection is limited to the trusted HA segment and isolated IoT segment.

---

# 44. Start Avahi

Restart:

```bash
systemctl restart avahi-daemon
```

Verify:

```bash
systemctl status avahi-daemon --no-pager
```

Expected behavior includes detection of the HA and IoT interfaces for mDNS.

Production addresses should not be copied into public documentation.

---

# 45. UDP 5353 Validation

Run:

```bash
ss -ulpn | grep ':5353'
```

Expected:

```text
0.0.0.0:5353
[::]:5353
```

Validation Result: Avahi is listening for mDNS over IPv4 and IPv6.

---

# 46. IoT mDNS Discovery Validation

Run:

```bash
avahi-browse -a
```

Expected results include services originating from devices on the IoT Wi-Fi interface.

Example service categories may include:

```text
_esphomelib._tcp
```

Device names should be redacted before publishing logs because hostnames can reveal hardware roles, naming conventions, or internal asset information.

The successful path is:

```text
FortiAP
   |
iot-matter
   |
USB Wi-Fi
   |
Linux wireless driver
   |
Debian
   |
Avahi
```

Validation Result: The reflector receives multicast DNS directly from the isolated Wi-Fi network.

---

# 47. Matter DNS-SD Validation

Check Matter-specific discovery:

```bash
avahi-browse -rt _matter._tcp
```

Then:

```bash
avahi-browse -rt _matterc._udp
```

Or:

```bash
avahi-browse -art | grep -i matter
```

Expected service types include:

```text
_matterc._udp
_matter._tcp
```

The services should be visible on the IoT-facing wireless interface over the applicable IP protocol.

Do not publish actual Matter instance identifiers unnecessarily.

Validation Result: Matter commissioning and operational DNS-SD advertisements exist on the IoT segment.

---

# 48. Home Assistant Matter Server mDNS

Verify that Home Assistant Matter Server reports:

```text
mdns: true
```

This confirms Matter Server itself is configured to use mDNS.

The missing component in a segmented topology is therefore the controlled transport of discovery advertisements from the isolated wireless segment to the Home Assistant network.

---

# 49. Final Commissioning

After Avahi reflection is operational, commission the Matter device again.

```text
Matter Device
    |
    |
iot-matter
    |
    |
vm-avahi
    |
Avahi reflector
    |
    |
HA network
    |
Matter Server
```

The device should appear in Home Assistant and become controllable.

Validation Result:

```text
CONNECTED
```

---

# 50. Why the Final Architecture Works

The complete commissioning sequence is:

```text
1. Android begins Matter commissioning.

2. Android and the Matter device communicate locally on the IoT SSID.

3. The Matter device joins the IoT SSID.

4. Matter Server communicates with the device on port 5540.

5. PASE succeeds.

6. Device attestation succeeds.

7. Home Assistant installs Matter operational credentials.

8. addNoc succeeds.

9. The device begins advertising operational Matter DNS-SD.

10. The advertisement remains local to the IoT network.

11. vm-avahi receives the advertisement through its Wi-Fi interface.

12. Avahi reflects mDNS onto ens18.

13. Home Assistant Matter Server receives the operational discovery record.

14. Matter Server resolves the device.

15. Operational Matter communication proceeds through the FortiGate.

16. Home Assistant controls the Matter device.
```

Before Avahi, the process stops between:

```text
Step 9
```

and:

```text
Step 13
```

Validation Result: mDNS reflection supplies the missing discovery path without replacing the FortiGate routing boundary.

---

# 51. Security Properties

## 51.1 IoT Isolation

Matter devices remain on:

```text
iot-matter
```

They are not moved onto the trusted Home Assistant network.

## 51.2 FortiGate Remains the Router

`vm-avahi` does not route packets.

```text
IPv4 forwarding = 0
IPv6 forwarding = 0
```

## 51.3 No NAT on vm-avahi

The reflector performs no address translation.

## 51.4 Restricted Avahi Interfaces

Avahi listens only on:

```text
ens18
<wifi-interface>
```

## 51.5 Restricted Matter Firewall Service

Matter traffic is restricted to:

```text
TCP 5540
UDP 5540
```

rather than permitting unrestricted inter-network traffic.

## 51.6 Internet Access

The Matter device does not require general Internet connectivity for normal local Home Assistant operation.

## 51.7 Information Disclosure

Public documentation should not contain:

```text
Production IPv4 addresses
Production IPv6 ULA prefixes
SLAAC interface identifiers
MAC addresses
FortiAP BSSIDs
Wi-Fi credentials
Matter device identifiers
Internal DNS names
Administrative interface addresses
VPN addresses
Firewall public addresses
Serial numbers
Backup identifiers
```

Validation Result: The architecture preserves network segmentation while minimizing unnecessary disclosure of infrastructure details.

---

# 52. Proxmox Backup

A Proxmox backup should be taken after Debian networking and USB Wi-Fi driver issues are resolved and before or after final Avahi configuration as appropriate.

The backup should cover:

```text
vm-avahi
VM configuration
VM disk
Network configuration
Avahi configuration
systemd Wi-Fi services
```

Production VM IDs, backup storage names, archive names, and infrastructure paths should not be included in public documentation unless required.

Validation Result:

```text
TASK OK
```

---

# 53. Boot Sequence

The expected startup sequence is:

```text
Debian boots
    |
    v
Updated kernel loads
    |
    v
Wireless driver binds USB adapter
    |
    v
Wi-Fi interface appears
    |
    v
wpa_supplicant-iot.service
    |
    v
Connect to iot-matter
    |
    +----------------------+
    |                      |
    v                      v
IPv6 SLAAC          dhclient-iot.service
    |                      |
    v                      v
IoT IPv6              IoT IPv4
    |                      |
    +----------+-----------+
               |
               v
          avahi-daemon
               |
               v
     Join IPv4/IPv6 mDNS
               |
               v
       Reflect mDNS between
       IoT and HA networks
```

Validation Result: Wireless connectivity, IP addressing, and mDNS reflection recover automatically after reboot.

---

# 54. Useful Diagnostic Commands

## Addresses

```bash
ip -br addr
```

Review before publishing because the output contains actual production addresses.

## IPv4 Routes

```bash
ip route
```

## IPv6 Routes

```bash
ip -6 route
```

## USB Devices

```bash
lsusb
```

USB vendor/product IDs should be reviewed before publication.

## USB Driver Binding

```bash
lsusb -t
```

Expected:

```text
Driver=rtl8xxxu
```

## Kernel

```bash
uname -r
```

## Wireless PHY

```bash
sudo iw phy
```

## Wireless Interfaces

```bash
sudo iw dev
```

## Wi-Fi Connection

```bash
iw dev <wifi-interface> link
```

## Wi-Fi Service

```bash
systemctl status wpa_supplicant-iot.service --no-pager
```

## DHCP Service

```bash
systemctl status dhclient-iot.service --no-pager
```

## Avahi Service

```bash
systemctl status avahi-daemon --no-pager
```

## UDP 5353

```bash
ss -ulpn | grep ':5353'
```

## All mDNS Services

```bash
avahi-browse -a
```

Review device and service names before publishing the output.

## Matter Operational Discovery

```bash
avahi-browse -rt _matter._tcp
```

## Matter Commissioning Discovery

```bash
avahi-browse -rt _matterc._udp
```

## All Matter mDNS Records

```bash
avahi-browse -art | grep -i matter
```

## Forwarding

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv6.conf.default.forwarding
```

Expected:

```text
0
0
0
```

Validation Result: These commands independently validate addressing, routing, wireless operation, mDNS, and forwarding state.

---

# 55. Troubleshooting Reference

| Symptom                                            | Root Cause                                  | Resolution                                     |
| -------------------------------------------------- | ------------------------------------------- | ---------------------------------------------- |
| Phone cannot reach device despite being on Wi-Fi   | FortiAP intra-SSID isolation                | Temporarily disable client isolation           |
| Device joins Wi-Fi but commissioning fails         | Failure occurs after Wi-Fi provisioning     | Inspect Matter Server logs                     |
| Matter Server reaches IoT device on port 5540      | Basic Matter routing works                  | Do not open arbitrary additional ports         |
| `addNoc` succeeds but reconnect fails              | Operational discovery problem               | Investigate mDNS                               |
| `Resolving (no address known)`                     | Matter Server cannot see operational DNS-SD | Deploy controlled Avahi reflection             |
| HAOS lacks usable Matter IPv6                      | No routed internal IPv6                     | Add ULA IPv6 and SLAAC                         |
| Debian installer reports bad mirror                | SLAAC works but ULA has no Internet route   | Configure IPv4 manually                        |
| USB Wi-Fi appears but `iw phy` is empty            | Kernel wireless driver problem              | Upgrade to a working kernel/driver combination |
| `wpa_supplicant` cannot authenticate using nl80211 | Driver does not expose proper wireless PHY  | Correct kernel/driver first                    |
| `wpa_cli list_networks` is empty                   | Missing `network={}` block                  | Add IoT network configuration                  |
| `Match already configured` repeatedly              | Multiple `wpa_supplicant` instances         | Stop duplicate process                         |
| Wi-Fi works until reboot                           | Association was launched manually           | Create systemd service                         |
| Wi-Fi connects after reboot but IPv4 is missing    | DHCP was launched manually                  | Create DHCP systemd service                    |
| `_matter._tcp` visible on Wi-Fi interface          | Operational Matter advertisements exist     | Validate reflection toward HA                  |
| Device connects after Avahi deployment             | mDNS reflection fixed operational discovery | Deployment successful                          |

Validation Result: Troubleshooting follows the actual Matter commissioning stages instead of assuming all failures are firewall problems.

---

# 56. Adding Future Matter Wi-Fi Devices

1. Verify `vm-avahi` is running.
2. Verify `wpa_supplicant-iot.service`.
3. Verify `dhclient-iot.service`.
4. Verify `avahi-daemon`.
5. Verify the USB Wi-Fi adapter is associated with the IoT SSID.
6. Put the Matter device into commissioning mode.
7. Connect the Android commissioner to the IoT SSID.
8. Temporarily disable intra-SSID isolation if required.
9. Confirm `_matterc._udp` appears.
10. Add the device through the Home Assistant Companion app.
11. Watch Matter Server logs if commissioning stalls.
12. Confirm `addNoc` succeeds.
13. Confirm `_matter._tcp` appears.
14. Confirm Home Assistant completes operational reconnect.
15. Test actual entity control.
16. Restore the desired SSID isolation configuration if compatible with normal operation.

Validation Result: The new Matter device remains isolated while retaining the minimum discovery and operational communication required by Home Assistant.

---

# 57. Smart Bulb Power Behavior

The physical wall switch should normally remain:

```text
ON
```

Home Assistant then logically turns the bulb on and off.

When Home Assistant turns the bulb off:

```text
LED output:   OFF
Wi-Fi:        ON
Matter:       ON
Electronics:  ON
```

The bulb remains reachable while consuming standby power rather than full illumination power.

---

# 58. Important Lessons

A failed Matter commissioning attempt does not automatically mean basic network connectivity is broken.

Matter Server logs can prove that:

```text
PASE succeeded
Attestation succeeded
NOC installation succeeded
```

before the failure occurs.

That narrows the problem specifically to:

```text
Operational discovery
```

The second major lesson is that:

```text
IPv6 routing
```

and:

```text
IPv6 multicast discovery
```

are separate problems.

A firewall can successfully route IPv6 between networks without causing link-local mDNS advertisements to cross the Layer 3 boundary.

Avahi addresses that specific discovery requirement without turning the reflector into an alternate router.

The third major lesson is that a visible USB network interface does not prove the wireless driver is operational.

A wireless adapter can appear to Linux while:

```bash
iw phy
```

returns no usable PHY.

Driver functionality should therefore be validated before troubleshooting WPA credentials, FortiAP configuration, or firewall rules.

---

# 59. Final Architecture

```text
                    FortiGate
                        |
          +-------------+-------------+
          |                           |
          |                           |
   Home Assistant                iot-matter
       Network                       |
          |                          |
          |                    Matter Device
          |                          |
          |                    USB Wi-Fi
          |                          |
          +------ vm-avahi ----------+
                    |
              Avahi Reflector
                    |
           mDNS only between
          intended interfaces

IPv4 routing:       FortiGate
IPv6 routing:       FortiGate
Matter TCP/UDP:     FortiGate policies
mDNS reflection:    vm-avahi
NAT on reflector:   NONE
IP forwarding:      DISABLED
```

Final validation:

```text
IPv4 HA network                         PASS
IPv4 IoT network                        PASS
IPv6 HA SLAAC                           PASS
IPv6 IoT SLAAC                          PASS
FortiGate IPv6 routing                  PASS
Matter TCP/UDP 5540                     PASS
USB Wi-Fi passthrough                   PASS
Wireless driver                         PASS
Automatic Wi-Fi association             PASS
Automatic IoT DHCP                      PASS
Avahi IPv4 mDNS                         PASS
Avahi IPv6 mDNS                         PASS
_matterc._udp discovery                 PASS
_matter._tcp discovery                  PASS
Matter commissioning                    PASS
Matter operational reconnect            PASS
Home Assistant device control           PASS
```

Deployment status:

```text
OPERATIONAL
```

---

# 60. Related Search Keywords

matter, matter-over-wifi, home-assistant, haos, matter-server, mdns, avahi, dns-sd, ipv6, ula, slaac, fortigate, fortinet, proxmox, debian, network-segmentation, iot-security, home-automation

---

## Distribution and Copyright

Copyright (c) 2026 Fong.

Permission is granted to redistribute this documentation in its original, unmodified form, provided appropriate credit is given to projectfong and a link to the projectfong GitHub page is included:

https://github.com/projectfong/

---

## Revision Control

| Version   | Date       | Summary                                                                                                                                                                                         | Author      |
| --------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------- |
| **1.0.0** | 2026-08-20 | Sanitized publication version of the Home Assistant Matter-over-segmented-Wi-Fi deployment documentation with production network addressing and infrastructure identifiers replaced or removed. | projectfong |
