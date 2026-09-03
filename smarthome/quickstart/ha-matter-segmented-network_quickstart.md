# Home Assistant Matter over Segmented Wi-Fi - Quickstart

Author: projectfong  
Copyright (c) 2026 projectfong

---

## Summary

Deploy Matter-over-Wi-Fi devices with Home Assistant while keeping Matter devices on a separate IoT network.

This architecture uses:

```text
Home Assistant OS
FortiGate or equivalent Layer 3 firewall/router
Internal IPv6 using ULA prefixes
SLAAC
Dedicated Avahi mDNS reflector
Isolated IoT Wi-Fi
Android Home Assistant Companion App
```

The final architecture is:

```text
                    Firewall / Router
                           |
             +-------------+-------------+
             |                           |
             v                           v
     Home Assistant                  IoT Wi-Fi
        Network                         |
             |                          |
             |                    Matter Device
             |                          |
             |                     USB Wi-Fi
             |                          |
             +------ vm-avahi ----------+
                       |
                 Avahi Reflector
```

The firewall/router remains responsible for Layer 3 routing and security policy.

`vm-avahi` reflects mDNS only.

It does not route or NAT traffic between the networks.

## Requirements

Before starting:

- Home Assistant OS.
- Home Assistant Matter Server.
- Android Home Assistant Companion App.
- Separate Home Assistant and IoT networks.
- Layer 3 firewall/router between those networks.
- IPv6 support on both Matter-relevant networks.
- Debian VM or equivalent Linux host for Avahi.
- Interface from the Avahi host to the Home Assistant network.
- Direct Layer 2 interface from the Avahi host to the IoT network.

The validated deployment used:

```text
Home Assistant network:
192.0.2.0/24
fd42:1234:5678:10::/64

IoT network:
198.51.100.0/24
fd42:1234:5678:20::/64
```

These are documentation examples.

Replace them with values appropriate for your environment.

## Quick Setup

### 1. Enable IPv6 on the Home Assistant Network

Assign an internal IPv6 `/64` to the Home Assistant network.

Example:

```text
fd42:1234:5678:10::/64
```

Configure the Layer 3 router to advertise the prefix using Router Advertisements.

Use:

```text
SLAAC:       Enabled
RA:          Enabled
DHCPv6:      Not required
IPv6 NAT:    Not required
```

Home Assistant should use automatic IPv6 configuration.

Verify from HAOS:

```bash
network info
```

Confirm:

```text
ipv6:
  method: auto
  ready: true
```

HAOS should have both:

```text
ULA IPv6 address
Link-local IPv6 address
```

### 2. Enable IPv6 on the IoT Network

Assign a separate `/64` to the IoT network.

Example:

```text
fd42:1234:5678:20::/64
```

Configure Router Advertisements and SLAAC.

Matter devices should receive an IPv6 address from this prefix.

No public IPv6 connectivity is required for this architecture.

### 3. Verify IPv6 Routing

Before troubleshooting Matter or mDNS, verify ordinary IPv6 connectivity between the required endpoints.

The intended path is:

```text
Home Assistant
      |
      v
Firewall / Router
      |
      v
IoT Network
      |
      v
Matter Device
```

Verify:

```text
IPv6 addressing
IPv6 routes
Firewall policy
Bidirectional reachability
```

Do not troubleshoot mDNS until basic unicast IPv6 routing works.

### 4. Configure Matter Firewall Policy

Permit the communication required between Home Assistant and the Matter endpoints.

Scope policy around:

```text
Source
Destination
Network direction
Required Matter traffic
```

The architecture should permit only the required communication between:

```text
HAOS
<->
Matter IoT devices
```

Home Assistant frontend access:

```text
TCP 8123
```

is separate from Matter device communication.

Do not expose the Home Assistant administrative interface to the IoT network unless explicitly required.

### 5. Prepare the Avahi VM

Create a small Debian VM.

Example:

```text
OS:       Debian 12
CPU:      1-2 vCPU
RAM:      1-2 GiB
Disk:     8 GiB
```

Provide two network interfaces:

```text
Interface 1:
Home Assistant network

Interface 2:
IoT network
```

The validated deployment uses:

```text
VirtIO NIC -> Home Assistant network
USB Wi-Fi  -> IoT Wi-Fi
```

The Avahi VM must have direct Layer 2 presence on both multicast domains.

### 6. Configure the Trusted Interface

The Home Assistant-side interface should retain the default route.

Example:

```text
Interface:
ens18

IPv4:
192.0.2.55/24

Gateway:
192.0.2.1

IPv6:
SLAAC
```

Verify:

```bash
ip -br addr
ip route
ip -6 route
```

The default IPv4 route should remain on the trusted interface.

### 7. Configure the IoT Wi-Fi Interface

Identify the wireless interface:

```bash
iw dev
```

Verify a functional wireless PHY:

```bash
iw phy
```

Do not continue if:

```bash
iw phy
```

does not return a usable wireless PHY.

Configure the IoT Wi-Fi connection using:

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

Protect the file:

```bash
chmod 600 /etc/wpa_supplicant/wpa_supplicant-iot.conf
```

Do not publish the real PSK.

### 8. Configure Automatic Wi-Fi Startup

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

Enable:

```bash
systemctl daemon-reload
systemctl enable --now wpa_supplicant-iot.service
```

Verify:

```bash
systemctl status wpa_supplicant-iot.service --no-pager
```

Expected:

```text
Active: active (running)
```

Verify association:

```bash
iw dev <wifi-interface> link
```

### 9. Configure IoT IPv4

Request an IPv4 address:

```bash
dhclient -v <wifi-interface>
```

The IoT interface should receive an address from the IoT network.

Example:

```text
198.51.100.57/24
```

Do not configure the IoT Wi-Fi interface as the VM's default route.

Expected routing concept:

```text
default via 192.0.2.1 dev ens18

192.0.2.0/24
    dev ens18

198.51.100.0/24
    dev <wifi-interface>
```

### 10. Configure Automatic IoT DHCP

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

### 11. Disable Routing on the Avahi VM

The reflector must not become an alternate router between the trusted and IoT networks.

Create:

```text
/etc/sysctl.d/99-avahi-reflector.conf
```

Add:

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

Do not configure NAT on the Avahi VM.

### 12. Install Avahi

Install:

```bash
apt update
apt install avahi-daemon avahi-utils -y
```

Back up the default configuration:

```bash
cp /etc/avahi/avahi-daemon.conf \
   /etc/avahi/avahi-daemon.conf.bak
```

### 13. Configure the Avahi Reflector

Edit:

```text
/etc/avahi/avahi-daemon.conf
```

Use:

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

The important controls are:

```ini
allow-interfaces=ens18,<wifi-interface>
```

and:

```ini
enable-reflector=yes
```

Only the intended Home Assistant and IoT interfaces should participate.

### 14. Start Avahi

Restart:

```bash
systemctl restart avahi-daemon
```

Verify:

```bash
systemctl status avahi-daemon --no-pager
```

Check mDNS listeners:

```bash
ss -ulpn | grep ':5353'
```

Expected:

```text
0.0.0.0:5353
[::]:5353
```

### 15. Verify Matter Discovery

Check Matter commissioning advertisements:

```bash
avahi-browse -rt _matterc._udp
```

Check operational Matter advertisements:

```bash
avahi-browse -rt _matter._tcp
```

Or:

```bash
avahi-browse -art | grep -i matter
```

Review output before publishing because Matter instance names and device identifiers may expose deployment-specific information.

### 16. Prepare the IoT SSID for Commissioning

Some wireless client-isolation configurations prevent the Android commissioner and Matter device from communicating locally.

During commissioning:

```text
Android Commissioner
        |
        v
    IoT Wi-Fi
        |
        v
 Matter Device
```

must be able to communicate as required by the commissioning workflow.

If client isolation blocks commissioning, temporarily disable intra-SSID isolation.

Restore the desired isolation configuration afterward if normal operation remains compatible with it.

### 17. Commission the Matter Device

Connect the Android commissioner to the IoT Wi-Fi network.

Put the Matter device into commissioning mode.

Add it through the Home Assistant Companion App.

The expected sequence is:

```text
Android commissioning
        |
        v
Device joins IoT Wi-Fi
        |
        v
Matter commissioning
        |
        v
Operational credentials installed
        |
        v
Matter operational mDNS advertisement
        |
        v
Avahi reflection
        |
        v
Home Assistant Matter Server
        |
        v
Operational Matter connection
```

### 18. Validate Home Assistant

Confirm:

```text
Device created
Entity created
HAOS control works
State reporting works
Device survives HAOS restart
Device survives Avahi restart
Local operation works without Internet
```

## Validation Checklist

```text
[ ] HAOS has ULA IPv6
[ ] IoT network has ULA IPv6
[ ] SLAAC works on both networks
[ ] IPv6 routing works between required endpoints
[ ] Firewall policy permits required Matter communication
[ ] Avahi VM has interfaces on both networks
[ ] Trusted interface remains the default route
[ ] IoT Wi-Fi interface associates successfully
[ ] IoT interface receives IPv4
[ ] IoT interface receives IPv6 through SLAAC
[ ] IPv4 forwarding is disabled on vm-avahi
[ ] IPv6 forwarding is disabled on vm-avahi
[ ] No NAT exists on vm-avahi
[ ] Avahi listens on UDP 5353
[ ] Avahi is restricted to intended interfaces
[ ] _matterc._udp is visible during commissioning
[ ] _matter._tcp is visible during operational discovery
[ ] Android can communicate with device during commissioning
[ ] Matter device appears in Home Assistant
[ ] Home Assistant control works
```

## Common Problems

### Phone Is on IoT Wi-Fi but Cannot Reach the Device

Check wireless client isolation.

If enabled, temporarily disable intra-SSID isolation and retry commissioning.

### HAOS Does Not Receive IPv6

Verify:

```text
Router Advertisement enabled
Correct /64 advertised
HAOS IPv6 method = auto
```

Then:

```bash
network info
```

Expected:

```text
method: auto
ready: true
```

### Matter Commissioning Progresses but Operational Reconnect Fails

Check Matter Server logs.

If commissioning credentials are successfully installed but the server reports behavior similar to:

```text
Resolving
no address known
peer-unreachable
```

check operational mDNS discovery.

Run:

```bash
avahi-browse -rt _matter._tcp
```

### `_matter._tcp` Exists on IoT but Home Assistant Cannot Discover It

Verify Avahi:

```bash
systemctl status avahi-daemon --no-pager
```

Verify UDP 5353:

```bash
ss -ulpn | grep ':5353'
```

Verify:

```ini
enable-reflector=yes
```

and confirm both intended interfaces appear in:

```ini
allow-interfaces=
```

### USB Wi-Fi Interface Exists but Cannot Connect

A Linux network interface does not prove the wireless PHY is functional.

Run:

```bash
iw phy
iw dev
```

If no usable PHY appears, fix the kernel/driver problem before troubleshooting credentials or firewall policy.

### Wi-Fi Works Until Reboot

Verify:

```bash
systemctl status wpa_supplicant-iot.service --no-pager
```

The dedicated supplicant service should be enabled.

### Wi-Fi Returns After Reboot but IPv4 Does Not

Verify:

```bash
systemctl status dhclient-iot.service --no-pager
```

Wi-Fi association and DHCP are separate operations.

### Avahi VM Routes Between Networks

Immediately verify:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
```

Both should return:

```text
0
```

The Avahi VM must not replace the Layer 3 firewall/router.

## Diagnostic Commands

Addresses:

```bash
ip -br addr
```

IPv4 routes:

```bash
ip route
```

IPv6 routes:

```bash
ip -6 route
```

Wireless PHY:

```bash
iw phy
```

Wireless interfaces:

```bash
iw dev
```

Wi-Fi link:

```bash
iw dev <wifi-interface> link
```

Wi-Fi service:

```bash
systemctl status wpa_supplicant-iot.service --no-pager
```

DHCP service:

```bash
systemctl status dhclient-iot.service --no-pager
```

Avahi:

```bash
systemctl status avahi-daemon --no-pager
```

mDNS listeners:

```bash
ss -ulpn | grep ':5353'
```

Matter commissioning discovery:

```bash
avahi-browse -rt _matterc._udp
```

Matter operational discovery:

```bash
avahi-browse -rt _matter._tcp
```

All Matter records:

```bash
avahi-browse -art | grep -i matter
```

Forwarding:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
sysctl net.ipv6.conf.default.forwarding
```

## Do Not Do This

Do not make `vm-avahi` a router.

Do not enable:

```text
IPv4 forwarding
IPv6 forwarding
NAT
```

on the reflector.

Do not assume that successful IPv6 routing means mDNS discovery also works.

Do not assume that successful mDNS discovery proves routed Matter communication works.

Do not flatten the network solely because Matter commissioning fails.

Do not expose Home Assistant TCP 8123 to the IoT network unless there is a separate requirement.

Do not publish:

```text
Wi-Fi credentials
Production IPv4 addresses
Production IPv6 prefixes
SLAAC interface identifiers
MAC addresses
BSSIDs
Matter instance identifiers
Internal DNS names
Administrative addresses
Serial numbers
```

## Final Architecture

```text
                     Firewall / Router
                           |
              +------------+------------+
              |                         |
              v                         v
      Home Assistant                IoT Wi-Fi
          Network                       |
              |                         |
              |                    Matter Device
              |                         |
              |                    USB Wi-Fi
              |                         |
              +------ vm-avahi ---------+
                         |
                  Avahi Reflector

IPv4 routing:
Firewall / Router

IPv6 routing:
Firewall / Router

Security policy:
Firewall / Router

mDNS reflection:
vm-avahi

NAT on vm-avahi:
NONE

IP forwarding on vm-avahi:
DISABLED
```

This preserves the Layer 3 security boundary while providing Matter with the routed IPv6 and cross-segment discovery required for local Home Assistant operation.

## Full Documentation

This quickstart intentionally omits the detailed Matter commissioning analysis, PASE and NOC troubleshooting, operational reconnect investigation, Debian installer behavior, USB Wi-Fi driver failure analysis, kernel upgrade process, DHCP lease discussion, packet-level reasoning, and extended architecture explanation.

See the full Home Assistant Matter over Segmented Wi-Fi implementation documentation for those details.

## Related Search Keywords

home-assistant, haos, matter, matter-over-wifi, matter-server, mdns, avahi, dns-sd, ipv6, ula, slaac, segmented-network, iot-security, home-automation

## Revision Control

| Version | Date | Summary | Author |
| ------- | ---- | ------- | ------ |
| 1.0.0 | 2026-08-31 | Initial Home Assistant Matter-over-segmented-Wi-Fi quickstart. | projectfong |
