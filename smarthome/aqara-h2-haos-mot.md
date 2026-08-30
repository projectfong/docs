# Aqara Light Switch H2 US Local Home Assistant Matter-over-Thread Implementation

Author: projectfong  
Copyright (c) 2026 projectfong

> **Note:** Deployment-specific network prefixes, host addresses, Thread identifiers, USB serial identifiers, mDNS hostnames, and firewall interface/policy identifiers are represented by descriptive placeholders. Replace these placeholders only with values appropriate to your own environment.

---

## Summary

This document records the implementation of Aqara Light Switch H2 US smart wall switches in an older home where neutral conductors are not available in several existing switch boxes.

The design uses Matter over Thread with Home Assistant OS for local control, a dedicated Raspberry Pi 3 OpenThread Border Router, a SONOFF Dongle Lite MG21 configured as an OpenThread Radio Co-Processor, and the existing segmented IPv6 network infrastructure.

A separate SONOFF Dongle Lite MG21 is reserved for Zigbee through Home Assistant ZHA.

The implementation preserves the existing network segmentation rather than placing Home Assistant, the Thread Border Router, and IoT infrastructure onto a single flat network.

The deployed Thread path is:

```text
Aqara H2
    |
    | Matter over Thread
    v
Thread Mesh
    |
    v
SONOFF Dongle Lite MG21
    |
    | USB / OpenThread RCP
    v
Raspberry Pi 3
    |
    | OpenThread Border Router
    v
IoT Wi-Fi
<IOT_IPV6_PREFIX>
    |
    v
FortiGate
    |
    v
Home Assistant Network
<HA_IPV6_PREFIX>
    |
    v
Home Assistant OS
    |
    v
Matter Server
```

Thread introduces an additional IPv6 network behind the OTBR:

```text
<THREAD_OMR_PREFIX>
```

This Thread Off-Mesh-Routable prefix must be routable between Home Assistant and the Thread mesh.

Because the deployment uses separate network segments, additional IPv6 routing, firewall policy, and discovery considerations apply that may not be necessary on a flat network.

The implementation also reuses the existing Avahi mDNS reflector previously deployed for segmented Matter-over-Wi-Fi operation.

**Status:** Operational. Thread infrastructure, Raspberry Pi 3 OTBR, Home Assistant external OTBR integration, Thread credential synchronization, segmented IPv6 routing, bidirectional FortiGate policy, Aqara H2 Thread attachment, Matter commissioning, Home Assistant entity control, and physical switch validation have been completed successfully.

Validation Result: The dedicated Thread infrastructure, Thread mesh, Home Assistant OTBR integration, Thread OMR routing, required segmented-network IPv6 paths, Matter commissioning, and final H2 control have been validated.

---

## Purpose

The purpose of this implementation is to add locally controlled smart wall switches to existing electrical circuits without rewiring the home solely to provide neutral conductors at switch locations.

The implementation is designed to:

* Support existing switch locations without a neutral conductor.
* Accommodate older electrical wiring and existing switch-box topology.
* Retain normal physical ON/OFF operation at the wall.
* Provide local ON/OFF control through Home Assistant OS.
* Use Matter over Thread as the primary Aqara H2 integration.
* Avoid an Aqara proprietary hub.
* Avoid cloud integration by default.
* Avoid cloud dependency for normal switch operation.
* Reuse the existing Home Assistant Matter infrastructure.
* Add vendor-neutral Thread infrastructure for future Matter-over-Thread devices.
* Add a dedicated local Zigbee coordinator for future Zigbee devices.
* Keep Zigbee and Thread on separate dedicated radios.
* Preserve the existing segmented IoT network architecture.
* Reuse the existing IPv6 and Avahi/mDNS infrastructure where applicable.
* Preserve the FortiGate as the Layer 3 routing and firewall boundary.
* Avoid flattening the network solely to simplify Matter or Thread.
* Avoid replacing existing branch-circuit wiring solely to support smart switches.

This implementation is maintained as technical documentation for the deployed Home Assistant environment.

Validation Result: The architecture preserves the existing electrical and network segmentation design while adding local Matter-over-Thread capability.

---

## Segmented-Network Scope

This implementation is specifically designed for a segmented network.

Home Assistant OS and the Raspberry Pi OTBR do not reside on the same infrastructure subnet.

The relevant infrastructure networks are:

```text
Home Assistant Network
<HA_IPV6_PREFIX>

IoT Network
<IOT_IPV6_PREFIX>
```

Thread adds another routed IPv6 network:

```text
Thread OMR
<THREAD_OMR_PREFIX>
```

The resulting topology is:

```text
Home Assistant OS
<HA_IPV6_PREFIX>
        |
        v
    FortiGate
        |
        v
IoT Network
<IOT_IPV6_PREFIX>
        |
        v
Raspberry Pi 3 / OTBR
        |
        v
Thread OMR
<THREAD_OMR_PREFIX>
        |
        v
Matter-over-Thread Devices
```

This differs from a typical flat Home Assistant deployment where Home Assistant and the Thread Border Router may reside on the same directly connected LAN.

### Flat-Network Difference

On a flat network, several segmented-network steps documented here may not be required.

Examples include:

* Inter-VLAN IPv6 firewall policies.
* FortiGate static routing to the Thread OMR.
* FortiGate reverse-path-forwarding troubleshooting.
* Separate HAOS-to-Thread and Thread-to-HAOS firewall policy validation.
* Cross-segment mDNS reflection.
* Avahi reflector deployment.
* Inter-VLAN packet captures.
* FortiGate IPv6 debug-flow analysis.

These additional requirements exist because this implementation deliberately preserves network segmentation.

The objective is not:

```text
Make Matter work by placing everything on one LAN
```

The objective is:

```text
Make Matter over Thread work while preserving
the existing network security boundaries
```

Thread, Matter, IPv6, and Home Assistant still have their normal protocol requirements regardless of whether the infrastructure network is flat or segmented.

Validation Result: Network-specific complexity documented in this guide must not automatically be interpreted as a universal Matter-over-Thread requirement.

---

## Why the Aqara H2 Was Selected

The Aqara Light Switch H2 US was selected primarily because the home uses older electrical wiring and several existing wall-switch locations do not contain a neutral conductor.

Many current smart wall switches require:

```text
Line
Load
Neutral
Ground
```

The existing switch locations being upgraded cannot universally provide that wiring configuration without modifying or replacing existing branch-circuit wiring.

A whole-home or branch-circuit rewire solely to add neutral conductors for smart switches would introduce significant cost and unnecessary electrical work.

The Aqara H2 supports operation with or without a neutral conductor, allowing smart-switch functionality to be added while retaining the existing electrical topology.

The primary selection criteria are:

* No-neutral operation.
* Compatibility with older switch locations where neutral conductors were not originally brought into the switch box.
* Integrated wall-switch design.
* No additional relay module installed behind the wall switch.
* Matter-over-Thread support.
* Local integration with Home Assistant OS.
* No Aqara hub required when operated using Matter over Thread.
* No cloud integration required by default for normal operation.
* No Aqara cloud dependency required for normal Home Assistant ON/OFF control.
* Physical local ON/OFF control.
* Reusable Thread infrastructure rather than a vendor-specific bridge.
* Physical dimensions appropriate for the existing switch boxes.

The no-neutral capability is the primary electrical requirement.

Matter over Thread and Home Assistant integration provide the preferred automation architecture without requiring the home's existing electrical wiring to be replaced solely to accommodate smart switches.

Validation Result: The H2 meets the primary no-neutral requirement while providing a local Matter-over-Thread integration path.

---

## Electrical Safety

Electrical work must be performed according to applicable electrical codes, manufacturer instructions, and the actual wiring present at each switch location.

Before removing or installing a switch:

1. Identify the correct branch circuit.
2. Turn the circuit breaker OFF.
3. Verify that the circuit is actually de-energized using an appropriate voltage tester.
4. Do not rely solely on the breaker label.
5. Identify line, load, neutral if present, and the equipment-grounding path.
6. Verify the actual circuit topology before connecting the H2.
7. Follow the Aqara wiring instructions appropriate to neutral or no-neutral operation.

Wire color alone must not be treated as definitive conductor identification in older wiring.

A white conductor is not automatically a neutral.

The absence of a visible bare or green conductor does not automatically prove that an older metal box lacks an equipment-grounding path.

An equipment-grounding conductor must never be repurposed as a neutral conductor.

If the wiring cannot be positively identified, electrical work should stop until the circuit can be properly evaluated.

Validation Result: Each switch location requires individual electrical verification before installation.

---

## Aqara Light Switch H2 US

The selected switch is:

```text
Aqara Light Switch H2 US
2 Buttons
1 Channel
Model: WS-K02E
```

The H2 supports:

* Matter over Thread.
* Zigbee.
* Bluetooth for commissioning and configuration functions.
* Installation with a neutral conductor.
* Installation without a neutral conductor.
* Local physical switching.
* Smart ON/OFF control.
* An additional physical button that may be exposed for automation use depending on protocol and controller support.

For this implementation, the selected operational protocol is:

```text
Matter over Thread
```

Zigbee capability is not the primary transport for the H2.

Validation Result: Matter over Thread is the selected H2 operating mode.

---

## No-Neutral Electrical Requirement

The most important H2 capability for this implementation is operation without a neutral conductor at the wall-switch location.

A traditional no-neutral switch circuit can conceptually appear as:

```text
Electrical Panel
      |
     Line
      |
      v
  Wall Switch
      |
     Load
      |
      v
    Light
      |
    Neutral
      |
      v
Electrical Panel
```

The neutral terminates at the load rather than being carried into the wall-switch box.

The H2 allows the existing topology to remain in place:

```text
Line
  |
  v
Aqara H2
  |
  v
Switched Load
  |
  v
Light
  |
  v
Neutral
```

This avoids requiring a neutral conductor to be added solely for smart-switch functionality.

The manufacturer's documented minimum load and no-neutral requirements must be observed.

Validation Result: No-neutral capability directly addresses the existing older-home switch wiring.

---

## With-Neutral Installation

Where a neutral conductor is available, the H2 can use the neutral connection.

Conceptually:

```text
Line --------+
             |
             v
          Aqara H2
             |
             +-------- Load
             |
Neutral -----+
```

The installation method must always match the actual circuit and manufacturer's wiring instructions.

A neutral conductor must never be fabricated by repurposing an equipment-grounding conductor.

Validation Result: Both neutral and no-neutral installation modes are supported, allowing each switch location to be handled according to its actual wiring.

---

## Local-First and Cloud-Independent Operation

The H2 deployment is designed for local operation through Home Assistant OS.

The normal operational path is:

```text
Aqara H2
    |
    | Matter over Thread
    v
Thread Mesh
    |
    v
OpenThread Border Router
    |
    | Local IPv6
    v
Local Network
    |
    v
Home Assistant OS
    |
    v
Matter Server
    |
    v
Local Automations
```

No Aqara cloud integration is required by default for normal switch operation.

The implementation does not require:

* Aqara cloud integration in Home Assistant OS.
* An Aqara proprietary hub.
* Cloud connectivity for routine ON/OFF commands.
* Cloud connectivity for normal Home Assistant automations.
* A cloud service in the normal operational control path.

The OpenThread Border Router provides IPv6 connectivity between the Thread mesh and the existing IP infrastructure.

Home Assistant remains the Matter controller.

Cloud services are intentionally outside the normal control path.

Validation Result: Normal switch control is designed to remain local to the Home Assistant and Matter infrastructure.

---

## Core Components

| Component                     | Description                                                | Status              |
| ----------------------------- | ---------------------------------------------------------- | ------------------- |
| Aqara Light Switch H2 US      | No-neutral-capable Matter-over-Thread wall switch          | Operational         |
| Home Assistant OS             | Local automation and device-management platform            | Operational         |
| Home Assistant Matter Server  | Local Matter controller                                    | Operational         |
| SONOFF Dongle Lite MG21 #1    | EFR32MG21 USB radio dedicated to Zigbee                    | Separate deployment |
| ZHA                           | Home Assistant Zigbee integration                          | Separate deployment |
| SONOFF Dongle Lite MG21 #2    | EFR32MG21 USB radio dedicated to Thread                    | Operational         |
| Raspberry Pi 3                | External OpenThread Border Router host                     | Operational         |
| Ubuntu Server 24.04 LTS ARM64 | Raspberry Pi OTBR operating system                         | Operational         |
| Docker Engine                 | OTBR container runtime                                     | Operational         |
| OpenThread Border Router      | Routes IPv6 between Thread and infrastructure              | Operational         |
| IoT Wi-Fi                     | OTBR infrastructure network                                | Operational         |
| Avahi reflector               | Existing mDNS reflection between required network segments | Operational         |
| FortiGate                     | IPv6 routing, segmentation, and firewall policy            | Operational         |

Validation Result: The required Matter-over-Thread infrastructure is operational without introducing an Aqara proprietary hub.

---

## Matter, Thread, and OTBR Roles

Matter and Thread perform different functions.

Matter is the application protocol used by Home Assistant and the H2.

Thread is the IPv6 mesh transport carrying Matter traffic.

The relationship is:

```text
Matter
  |
  | Application protocol
  v
Thread
  |
  | IPv6 mesh transport
  v
IEEE 802.15.4
```

Existing Matter-over-Wi-Fi devices use:

```text
Matter Device
     |
    Wi-Fi
     |
IP Network
     |
Home Assistant
     |
Matter Server
```

The H2 instead uses:

```text
Aqara H2
    |
  Thread
    |
Thread Border Router
    |
IP Network
    |
Home Assistant
    |
Matter Server
```

A Matter controller does not automatically provide Thread radio connectivity.

A Matter-over-Thread endpoint therefore requires a Thread Border Router.

Validation Result: The existing Matter Server remains the controller while OTBR provides the Thread transport and border-routing function.

---

## Why Matter-over-Wi-Fi Working Did Not Prove Thread Would Work

Matter-over-Wi-Fi was already operational across the segmented network before the Thread deployment.

That proved several important components were already functional:

```text
Home Assistant Matter Server
FortiGate IPv6 routing
IoT IPv6 SLAAC
Home Assistant IPv6 SLAAC
Avahi mDNS reflection
Matter commissioning
Matter operational discovery
```

However, Matter-over-Wi-Fi endpoints receive IPv6 addresses directly from the infrastructure network.

For example:

```text
IoT Infrastructure Prefix
<IOT_IPV6_PREFIX>
```

FortiGate is directly connected to that network.

Thread devices are different.

They receive IPv6 addresses from a Thread OMR prefix behind the Border Router:

```text
Thread OMR
<THREAD_OMR_PREFIX>
```

Therefore:

```text
Matter-over-Wi-Fi

FortiGate
    |
    +---- <IOT_IPV6_PREFIX>
               |
               v
          Matter Device
```

is fundamentally different from:

```text
Matter-over-Thread

FortiGate
    |
    +---- <IOT_IPV6_PREFIX>
               |
               v
             OTBR
               |
               +---- <THREAD_OMR_PREFIX>
                            |
                            v
                       Thread Device
```

The second topology introduces another routed IPv6 network.

That difference became the primary segmented-network routing issue during deployment.

Validation Result: Existing Matter-over-Wi-Fi success does not by itself prove Thread OMR reachability.

---

## Dedicated Thread Border Router

The Thread Border Router is intentionally external to Home Assistant OS.

The deployed path is:

```text
Aqara H2
    |
  Thread
    |
SONOFF LMG21
    |
   USB
    |
Raspberry Pi 3
    |
   OTBR
    |
  Wi-Fi
    |
IoT Network
    |
FortiGate
    |
Home Assistant Network
    |
HAOS
```

The Raspberry Pi does not host Home Assistant.

The Raspberry Pi is not the Matter controller.

Its primary smart-home role is:

```text
OpenThread Border Router
```

Home Assistant remains responsible for Matter.

Validation Result: Thread transport and Matter control remain separate architectural roles.

---

## SONOFF Dongle Lite MG21 Thread Radio

The Thread radio is:

```text
SONOFF Zigbee 3.0 & Thread Dongle Lite
Chipset: Silicon Labs EFR32MG21
Role: OpenThread RCP
```

The radio was flashed using the official SONOFF web flasher.

Flasher:

```text
https://dongle.sonoff.tech/sonoff-dongle-flasher/
```

Installed firmware:

```text
OpenThread RCP
Version: 2.4.4 Stable
UART: 460800
Hardware flow control: disabled
```

OTBR later reported the RCP as:

```text
SL-OPENTHREAD/2.4.4.0_GitHub-7074a43e4
EFR32
Sep 3 2025 13:57:55
```

Validation Result: The EFR32MG21 is operating as a dedicated OpenThread RCP.

---

## Persistent Thread Radio Device Path

The Thread radio initially appears as:

```text
/dev/ttyUSB0
```

A persistent device identity is preferred because `/dev/ttyUSB0` can change if USB enumeration changes.

The deployed persistent path is represented as:

```text
/dev/serial/by-id/<THREAD_RCP_DEVICE_ID>
```

The Docker configuration maps that persistent host path to:

```text
/dev/ttyUSB0
```

inside the container.

The resulting OTBR radio URL is:

```text
spinel+hdlc+uart:///dev/ttyUSB0?uart-baudrate=460800
```

Hardware flow control is not enabled.

Validation Result: OTBR uses a persistent host USB identity while retaining a simple container-side `/dev/ttyUSB0` path.

---

## Raspberry Pi 3 OTBR Host

The Raspberry Pi 3 runs:

```text
Ubuntu Server 24.04 LTS ARM64
```

The installation is minimized and headless.

Primary interfaces:

```text
Infrastructure: wlan0
Thread:         wpan0
Thread RCP:     /dev/ttyUSB0
```

The Pi connects wirelessly to the existing IoT network.

The Thread RCP remains directly attached through USB.

The architecture is:

```text
Thread Device
     |
  Thread RF
     |
   LMG21
     |
    USB
     |
Raspberry Pi 3
     |
   OTBR
     |
   Wi-Fi
     |
IoT Network
```

Wi-Fi carries infrastructure IPv6 traffic.

It does not remotely transport the RCP serial protocol.

Validation Result: The Raspberry Pi 3 operates as a dedicated external OTBR appliance.

---

## OTBR Host Preparation

The upstream OpenThread host setup script was used with `wlan0` as the infrastructure interface:

```bash
curl -sSL https://raw.githubusercontent.com/openthread/ot-br-posix/refs/heads/main/etc/docker/border-router/setup-host | INFRA_IF_NAME=wlan0 bash
```

The resulting host settings include:

```text
net.ipv6.conf.wlan0.accept_ra = 2
net.ipv6.conf.wlan0.accept_ra_rt_info_max_plen = 64
net.ipv6.conf.all.forwarding = 1
net.ipv4.ip_forward = 1
```

These settings allow the Pi to perform the required border-routing role.

Validation Result: Host forwarding and IPv6 Router Advertisement handling are configured for OTBR operation.

---

## OTBR Docker Deployment

OTBR runs as a Docker container using host networking.

The deployed image is pinned by digest:

```text
openthread/border-router@sha256:1d3184a47c37fcaaa91601d71e8977b6c96db27d4292190c8e71b5151b8c5610
```

The deployed Compose configuration is:

```yaml
services:
    otbr:
        image: openthread/border-router@sha256:1d3184a47c37fcaaa91601d71e8977b6c96db27d4292190c8e71b5151b8c5610
        container_name: otbr
        restart: unless-stopped
        network_mode: host
        privileged: true
        devices:
            - /dev/serial/by-id/<THREAD_RCP_DEVICE_ID>:/dev/ttyUSB0
            - /dev/net/tun:/dev/net/tun
        environment:
            OT_RCP_DEVICE: "spinel+hdlc+uart:///dev/ttyUSB0?uart-baudrate=460800"
            OT_INFRA_IF: "wlan0"
            OT_THREAD_IF: "wpan0"
            OT_LOG_LEVEL: "5"
            OT_REST_LISTEN_ADDR: "0.0.0.0"
            OT_REST_LISTEN_PORT: "8081"
```

Start OTBR with:

```bash
cd ~/otbr
docker compose up -d
```

Verify:

```bash
docker ps
```

Validation Result: OTBR remains operational as a restartable Docker service using host networking.

---

## OTBR Process

The resulting OTBR agent uses:

```text
/usr/sbin/otbr-agent
-d5
-v
-s
--vendor-name OpenThread
--model-name BorderRouter
-I wpan0
-B wlan0
spinel+hdlc+uart:///dev/ttyUSB0?uart-baudrate=460800
trel://wlan0
--rest-listen-address 0.0.0.0
--rest-listen-port 8081
```

The major interface relationships are:

```text
-I wpan0
    Thread interface

-B wlan0
    Infrastructure interface

spinel+hdlc+uart:///dev/ttyUSB0
    Thread RCP

REST TCP 8081
    Home Assistant OTBR API
```

Validation Result: OTBR is bound to the intended Thread and infrastructure interfaces.

---

## OTBR Radio Validation

The OpenThread CLI can be queried through the running container.

Verify state:

```bash
docker exec otbr ot-ctl state
```

After Thread network formation, the observed state became:

```text
leader
```

Verify OTBR OpenThread version:

```bash
docker exec otbr ot-ctl version
```

Observed:

```text
OPENTHREAD/7e2923b; POSIX; Aug 30 2026 02:01:42
```

Verify RCP version:

```bash
docker exec otbr ot-ctl rcp version
```

Observed:

```text
SL-OPENTHREAD/2.4.4.0_GitHub-7074a43e4; EFR32; Sep 3 2025 13:57:55
```

Validation Result: OTBR successfully communicates with the SONOFF EFR32MG21 RCP.

---

## OTBR REST API

The OTBR REST API listens on:

```text
0.0.0.0:8081
```

Verify:

```bash
ss -lntp | grep 8081
```

The REST discovery endpoint can be tested with:

```bash
curl http://<OTBR-IP>:8081/.well-known/thread/br-rest
```

The API reported:

```text
API Version: 0.5.0
Base: /api/
```

Available API areas included:

```text
/api/node
/api/actions
/api/devices
/api/diagnostics
```

Validation Result: The OTBR REST API is reachable from the infrastructure network.

---

## Home Assistant External OTBR Integration

Home Assistant OS uses the Raspberry Pi as an external OpenThread Border Router.

The integration URL is:

```text
http://<OTBR-IP>:8081
```

Home Assistant successfully recognized the external Border Router as:

```text
OpenThread BorderRouter <INSTANCE_ID>
```

The Thread integration then formed and managed the Home Assistant Thread network.

Validation Result: HAOS successfully consumes the external Raspberry Pi OTBR through its REST interface.

---

## Thread Network

The resulting Thread network is represented as:

```text
<THREAD_NETWORK_NAME>
```

After network formation:

```bash
docker exec otbr ot-ctl state
```

returned:

```text
leader
```

The active dataset showed non-secret operational metadata such as the selected radio channel and channel mask.

The complete active dataset contains sensitive Thread credentials and must not be published.

Do not retain or publish values such as:

```text
Network Key
PSKc
Complete Active Operational Dataset
Matter setup credentials
```

Validation Result: The Raspberry Pi OTBR formed and operates the Home Assistant Thread network.

---

## Thread Credential Synchronization

The Android Home Assistant Companion App must have the intended Thread network credentials before commissioning a Thread endpoint.

The synchronization workflow used was:

```text
Home Assistant Companion App
    |
    v
Settings
    |
    v
Companion app
    |
    v
Troubleshooting
    |
    v
Sync Thread credentials
```

The synchronization completed successfully and reported that Thread credentials were added to the Android device.

Before synchronization, Matter commissioning reported that a Thread Border Router was required.

After synchronization, commissioning progressed to attempting connectivity to the Thread network.

Validation Result: Android Thread credentials were successfully synchronized from Home Assistant.

---

## Thread IPv6 Prefixes

The Thread network exposes multiple types of IPv6 addresses.

Two prefixes were particularly important during troubleshooting.

### Thread Mesh-Local Prefix

Represented as:

```text
<THREAD_MESH_LOCAL_PREFIX>
```

This is the Thread mesh-local prefix.

It must not be confused with the infrastructure-routable Thread OMR prefix.

### Thread OMR Prefix

Represented as:

```text
<THREAD_OMR_PREFIX>
```

The OpenThread network data confirmed the OMR prefix as present in Thread network data.

Conceptually:

```text
Prefixes:
<THREAD_OMR_PREFIX> <flags>
```

This is the prefix relevant to communication between Thread endpoints and the external IPv6 infrastructure.

Validation Result: `<THREAD_OMR_PREFIX>` is the Thread OMR used for routed communication outside the Thread mesh.

---

## Why the Thread OMR Matters

The Thread OMR is the routed IPv6 network representing Thread devices to the external infrastructure.

For example, during Aqara commissioning the switch registered an address represented as:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

That address is not part of the IoT Wi-Fi infrastructure prefix:

```text
<IOT_IPV6_PREFIX>
```

It belongs to:

```text
<THREAD_OMR_PREFIX>
```

behind the OTBR.

The routing relationship is therefore:

```text
FortiGate
    |
    | IoT network
    v
<IOT_IPV6_PREFIX>
    |
    v
Raspberry Pi 3 / OTBR
    |
    v
<THREAD_OMR_PREFIX>
```

This is the key network difference between the existing Matter-over-Wi-Fi deployment and the new Matter-over-Thread deployment.

Validation Result: The Thread OMR must be treated as an additional routed IPv6 network in the segmented environment.

---

## Home Assistant IPv6

Home Assistant OS uses the Home Assistant infrastructure prefix:

```text
<HA_IPV6_PREFIX>
```

An HAOS address is represented as:

```text
<HAOS_IPV6>/64
```

HAOS uses an IPv6 default route through the FortiGate.

Conceptually:

```text
default ::/0
    |
    v
FortiGate
```

HAOS does not require a separate static route for the Thread OMR.

When HAOS sends to:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

the destination does not match a directly connected HAOS network.

HAOS therefore uses its normal IPv6 default route.

FortiGate then performs the more-specific routing decision.

Validation Result: Thread routing is centralized at FortiGate rather than adding a special static route to HAOS.

---

## Why HAOS Does Not Need a Thread Static Route

Routing uses longest-prefix matching.

HAOS evaluates a Thread destination such as:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

against its local routing table.

Conceptually:

```text
Directly connected Thread OMR route? No
More-specific Thread OMR route?      No
IPv6 default route ::/0?             Yes
```

Therefore:

```text
HAOS
    |
    | IPv6 default route
    v
FortiGate
```

FortiGate then has the route:

```text
<THREAD_OMR_PREFIX>
    |
    v
Raspberry Pi 3 OTBR
```

This preserves the intended network architecture:

```text
Endpoint routing decisions
        |
        v
FortiGate
        |
        v
Segment-specific next hop
```

Validation Result: A Thread-specific route on HAOS is unnecessary when FortiGate is the IPv6 default gateway and has the appropriate OMR route.

---

## Segmented-Network Routing Problem

The first major commissioning problem was not Thread radio connectivity.

The Aqara successfully attached to the Thread mesh.

An observed child entry conceptually included:

```text
RLOC16:       <RLOC16>
LQ In:        3
Extended MAC: <THREAD_CHILD_EXT_MAC>
```

Later commissioning produced another attached child:

```text
RLOC16: <RLOC16>
LQ In:  3
```

The registered addresses included:

```text
<THREAD_ENDPOINT_MESH_LOCAL_IPV6>
<THREAD_ENDPOINT_OMR_IPV6>
```

The OMR address proved that the endpoint had received an infrastructure-routable Thread address.

However, Matter commissioning still failed at that point in troubleshooting.

Validation Result: Thread attachment was successful before full Matter commissioning succeeded, proving the initial failure was above the basic Thread radio-association layer.

---

## Why FortiGate Needed a Thread OMR Route

The existing FortiGate already knew about:

```text
Home Assistant:
<HA_IPV6_PREFIX>

IoT:
<IOT_IPV6_PREFIX>
```

Both are directly connected infrastructure networks.

FortiGate did not initially have a route for:

```text
<THREAD_OMR_PREFIX>
```

because that prefix exists behind the Raspberry Pi OTBR.

This caused a critical difference.

A packet sourced from the Thread OMR arrived at FortiGate through the IoT interface:

```text
<THREAD_OMR_SOURCE_IPV6>
    ->
<HAOS_DESTINATION_IPV6>
```

FortiGate received the packet but did not know that the source network legitimately existed behind the Pi.

The debug flow reported:

```text
reverse path check failed, drop
```

This was definitive evidence that the failure was occurring at FortiGate reverse-path validation.

Validation Result: A firewall ACCEPT policy alone was insufficient because FortiGate also required a valid route back toward the Thread OMR source prefix.

---

## FortiGate Thread OMR Static Route

A static IPv6 route was added for:

```text
Destination:
<THREAD_OMR_PREFIX>

Gateway:
<OTBR_INFRA_IPV6>

Interface:
<IOT_INTERFACE>
```

Conceptually:

```text
<THREAD_OMR_PREFIX>
        |
        v
<OTBR_INFRA_IPV6>
        |
        v
Raspberry Pi 3 / OTBR
```

FortiGate then reported conceptually:

```text
Routing entry for <THREAD_OMR_PREFIX>
Known via "static"
via <OTBR_INFRA_IPV6>, <IOT_INTERFACE>
```

This solved the reverse-path validation failure.

Validation Result: FortiGate now knows that `<THREAD_OMR_PREFIX>` is reachable through the Raspberry Pi OTBR on the IoT network.

---

## Why the Static Route Is a Segmented-Network Requirement

This static route exists because FortiGate is the Layer 3 router between Home Assistant and the IoT network.

A flat network may learn or reach the Thread OMR differently depending on the network topology and Router Advertisement behavior.

In this segmented architecture:

```text
HAOS VLAN
    |
FortiGate
    |
IoT VLAN
    |
OTBR
    |
Thread OMR
```

FortiGate must make the routing decision.

It cannot forward traffic to a network it does not know how to reach.

The route is therefore not a generic instruction that every Home Assistant Thread deployment requires a manually configured static route.

It is required here because:

* HAOS and OTBR are separated by FortiGate.
* FortiGate is the Layer 3 boundary.
* The Thread OMR exists behind the OTBR.
* The OMR was not otherwise present in the FortiGate routing table.

Validation Result: The route is an implementation-specific consequence of maintaining the segmented architecture.

---

## Thread-to-HAOS IPv6 Validation

Before adding the static route, a ping generated from the OpenThread stack toward HAOS failed:

```bash
docker exec otbr ot-ctl ping <HAOS_IPV6>
```

The Pi showed the request leaving the infrastructure interface:

```text
<OTBR_OMR_IPV6>
    >
<HAOS_IPV6>

ICMP6 echo request
```

FortiGate also saw the request arrive.

The initial FortiGate debug showed:

```text
reverse path check failed, drop
```

After adding the static OMR route, the same flow changed conceptually to:

```text
find a route via <HA_NETWORK_INTERFACE>
Check policy between <IOT_INTERFACE> -> <HA_NETWORK_INTERFACE>
Allowed by <POLICY_ID>
```

This proved:

```text
OTBR -> FortiGate        PASS
FortiGate RPF            PASS
FortiGate route          PASS
FortiGate policy         PASS
FortiGate -> HAOS        PASS
```

Validation Result: The static route corrected the Thread-originated FortiGate RPF failure.

---

## HAOS Reply Validation

A FortiGate packet capture after the routing correction showed the complete request and response:

```text
<IOT_INTERFACE> in
<THREAD_OMR_IPV6> -> HAOS
ICMP6 echo request

<HA_NETWORK_INTERFACE> out
<THREAD_OMR_IPV6> -> HAOS
ICMP6 echo request

<HA_NETWORK_INTERFACE> in
HAOS -> <THREAD_OMR_IPV6>
ICMP6 echo reply

<IOT_INTERFACE> out
HAOS -> <THREAD_OMR_IPV6>
ICMP6 echo reply
```

A simultaneous capture on the Raspberry Pi infrastructure interface showed:

```text
<THREAD_OMR_IPV6> -> HAOS
ICMP6 echo request

HAOS -> <THREAD_OMR_IPV6>
ICMP6 echo reply
```

This proved that:

* The Thread-originated packet reached FortiGate.
* FortiGate forwarded it to HAOS.
* HAOS received the packet.
* HAOS generated a reply.
* FortiGate routed the reply toward the IoT network.
* The reply reached the Raspberry Pi.

Validation Result: Thread OMR to HAOS infrastructure IPv6 routing became operational after the FortiGate static route was added.

---

## OTBR Local OMR Address Behavior

The OTBR itself had an OMR address represented by the placeholder:

```text
<OTBR_OMR_IPV6>
```

The Linux host showed this address on the Thread interface:

```text
inet6 <OTBR_OMR_IPV6>/64
scope global nodad
```

The local route table reported:

```text
local <OTBR_OMR_IPV6>
dev wpan0
proto kernel
```

A route lookup returned conceptually:

```text
local <OTBR_OMR_IPV6>
from ::
dev lo
table local
```

This explains why an ICMP reply to the OTBR's own OMR address does not need to appear as an ordinary forwarded packet back through `wpan0`.

The address belongs to the local OpenThread/Linux host.

Validation Result: The OTBR's own OMR address must be distinguished from a child endpoint OMR address when interpreting packet captures.

---

## Aqara Child Address Discovery

Attached Thread children can be viewed with:

```bash
docker exec otbr ot-ctl child table
```

Registered child IPv6 addresses can be viewed with:

```bash
docker exec otbr ot-ctl childip
```

During commissioning, the Aqara registered addresses represented by the following placeholders:

```text
<CHILD_ID>: <THREAD_ENDPOINT_MESH_LOCAL_IPV6>
<CHILD_ID>: <THREAD_ENDPOINT_OMR_IPV6>
```

The relevant infrastructure-routable address was:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

This allowed the actual Thread endpoint, rather than the OTBR itself, to be tested from HAOS.

Validation Result: `childip` provides direct evidence that the Aqara joined the Thread network and obtained an OMR address.

---

## HAOS-to-Thread Routing Test

HAOS attempted to reach the Aqara OMR address:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

FortiGate packet capture showed:

```text
<HA_NETWORK_INTERFACE> in
<HAOS_IPV6>
    ->
<THREAD_ENDPOINT_OMR_IPV6>
ICMP6 echo request
```

However, no corresponding IoT-side outbound packet initially appeared.

This proved:

```text
HAOS generated packet             PASS
HAOS default IPv6 route           PASS
Packet reached FortiGate          PASS
FortiGate static OMR route        PRESENT
FortiGate forwarding              FAIL
```

The next diagnostic step was FortiGate IPv6 debug flow.

Validation Result: The reverse direction failed independently of the previously corrected Thread-to-HAOS path.

---

## HAOS-to-Thread Firewall Failure

FortiGate debug flow showed:

```text
find a route:
gw-<OTBR_INFRA_IPV6>
via <IOT_INTERFACE>
```

This proved the static OMR route was working.

FortiGate then reported:

```text
Check policy between <HA_NETWORK_INTERFACE> -> <IOT_INTERFACE>
Denied by forward policy check (policy 0)
```

This identified the second segmented-network problem.

The routing table knew how to reach the Thread network, but no matching IPv6 firewall policy permitted HAOS to initiate traffic toward that network.

The failure was therefore:

```text
Routing       PASS
Firewall      FAIL
```

Validation Result: HAOS-to-Thread traffic required its own FortiGate IPv6 policy.

---

## Bidirectional FortiGate Policy Requirement

The deployment ultimately required IPv6 policy in both logical directions.

### Thread / OTBR to HAOS

Conceptually:

```text
Source:
Thread OMR
<THREAD_OMR_PREFIX>

Destination:
HAOS IPv6

Direction:
IoT -> Home Assistant network
```

This direction was validated after the OMR route corrected FortiGate reverse-path validation.

### HAOS to Thread

Conceptually:

```text
Source:
HAOS IPv6

Destination:
Thread OMR
<THREAD_OMR_PREFIX>

Direction:
Home Assistant network -> IoT
```

Without this direction, FortiGate reported:

```text
Denied by forward policy check (policy 0)
```

After the HAOS-to-Thread IPv6 policy was added, the missing reverse-direction firewall path was corrected.

Validation Result: The segmented design requires both routing knowledge and firewall authorization in the required traffic directions.

---

## Why Routing and Firewall Policy Are Separate

A valid route does not automatically permit traffic through a firewall.

The deployment demonstrated this directly.

After adding the Thread OMR route, FortiGate knew:

```text
<THREAD_OMR_PREFIX>
    ->
Raspberry Pi 3 OTBR
```

The route lookup succeeded:

```text
find a route:
gw-<OTBR_INFRA_IPV6>
via <IOT_INTERFACE>
```

But FortiGate still denied HAOS-initiated traffic:

```text
Denied by forward policy check (policy 0)
```

The two functions are separate:

```text
Routing:
Where should this packet go?

Firewall Policy:
Is this packet permitted to go there?
```

Both must succeed.

Validation Result: Static routing solved reachability knowledge; firewall policy separately authorized the traffic.

---

## Existing Matter-over-Wi-Fi Firewall Rules Were Not Sufficient

The existing Matter-over-Wi-Fi deployment used the normal IoT IPv6 range:

```text
<IOT_IPV6_PREFIX>
```

Thread endpoints instead use:

```text
<THREAD_OMR_PREFIX>
```

A policy or address object restricted to:

```text
<IOT_IPV6_PREFIX>
```

does not inherently represent:

```text
<THREAD_OMR_PREFIX>
```

The Thread OMR therefore needs to be considered independently when defining segmented-network policy.

Validation Result: Thread adds a separate routed security zone/prefix even though the OTBR infrastructure interface resides on the existing IoT network.

---

## Matter Firewall Service Considerations

The original Matter-over-Wi-Fi deployment was documented around TCP/UDP 5540.

Matter networking should not be assumed to use only one fixed destination port for every operational exchange.

Matter controllers and endpoints can use dynamically selected or advertised UDP ports as part of operational communication.

For this reason, firewall design should focus on tightly scoped:

```text
Source IPv6
Destination IPv6 or prefix
Interface direction
Required protocol behavior
```

rather than assuming that destination port 5540 alone universally represents all Matter communication.

This is especially important when troubleshooting a segmented deployment.

A broad temporary diagnostic policy may help prove a network path, but the final security policy should be scoped to the smallest practical source, destination, interface, and service set supported by the actual Matter traffic requirements.

Validation Result: Network segmentation should be enforced primarily by endpoint/prefix and interface scope rather than relying solely on a universal Matter port assumption.

---

## Existing Avahi and mDNS

The existing segmented Matter environment already uses Avahi to provide required mDNS/DNS-SD reflection between network segments.

The Thread implementation reuses that infrastructure.

The Avahi reflector is not the IPv6 router.

Its role is:

```text
mDNS / DNS-SD reflection
```

The FortiGate role is:

```text
Layer 3 IPv6 routing
Firewall policy
Network segmentation
```

The OTBR role is:

```text
Thread border routing
```

These functions must not be confused.

The architecture is:

```text
Thread Mesh
    |
   OTBR
    |
IoT Network
    |
    +----------------------+
    |                      |
    v                      v
FortiGate                Avahi
Routing / Policy         mDNS Reflection
    |                      |
    +----------+-----------+
               |
               v
       Home Assistant
```

Validation Result: Avahi provides discovery reflection but does not replace IPv6 routing between the Thread OMR and HAOS.

---

## Avahi Validation

The existing Avahi configuration is represented as:

```ini
[server]
use-ipv4=yes
use-ipv6=yes
allow-interfaces=<AVAHI_INTERFACE_1>,<AVAHI_INTERFACE_2>
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

The reflector observed the OTBR service:

```text
OpenThread BorderRouter <INSTANCE_ID>
_meshcop._udp
```

The OTBR advertised:

```text
hostname:
<OTBR_MDNS_HOSTNAME>.local

network:
<THREAD_NETWORK_NAME>
```

Matter services were also visible through the existing Avahi environment.

Validation Result: The existing reflector sees OTBR and Matter discovery traffic and was not the initial Thread routing failure.

---

## Flat-Network mDNS Difference

A flat deployment may not require Avahi.

If Home Assistant, the commissioning device, and required infrastructure services share the same multicast domain, normal mDNS can remain local to that network.

This deployment uses Avahi because Home Assistant and IoT infrastructure are intentionally separated.

Therefore:

```text
Avahi requirement
```

is an architectural consequence of this segmented environment, not an inherent requirement for every Matter-over-Thread installation.

Validation Result: Do not deploy an mDNS reflector solely because this guide uses one unless the target network actually requires cross-segment discovery.

---

## OTBR Host Firewall Findings

The host firewall state was checked during troubleshooting.

The important finding was that the demonstrated failure did not originate from the Raspberry Pi host firewall.

The IPv6 forwarding path was also validated.

OTBR and Docker install their own netfilter chains, including chains such as:

```text
OT_FORWARD_INGRESS
DOCKER-USER
DOCKER-FORWARD
```

The OTBR forwarding rules must not be indiscriminately flushed.

Commands such as:

```text
iptables -F
ip6tables -F
nft flush ruleset
```

should not be used as a generic troubleshooting step.

Docker and OTBR rely on netfilter integration.

Validation Result: The OTBR host firewall infrastructure was not responsible for the segmented-network routing failure, and Docker/OTBR netfilter state was preserved during troubleshooting.

---

## IPv6 Forwarding Validation

The Pi host was configured for IPv6 forwarding.

Relevant settings included:

```text
net.ipv6.conf.all.forwarding = 1
```

The `wpan0` interface was present and operational:

```text
wpan0
POINTOPOINT
MULTICAST
NOARP
UP
LOWER_UP
MTU 1280
```

The OMR prefix was installed on `wpan0`.

Validation Result: The Pi was operating as an IPv6 Thread Border Router rather than merely hosting an attached Thread radio.

---

## Matter Commissioning Progression

The commissioning failures changed as infrastructure problems were corrected.

### Initial Condition

Android reported that the device required a Thread Border Router.

Cause:

```text
Thread infrastructure / credentials not yet available to commissioner
```

### After OTBR Integration and Credential Synchronization

Commissioning progressed to:

```text
Checking connectivity to Thread network
```

and later failed with:

```text
Can't reach device
```

### Thread Attachment Evidence

Despite the failure, OTBR showed an attached child with strong link quality.

This proved:

```text
Bluetooth commissioning          progressed
Thread credentials               delivered
Aqara joined Thread              PASS
Thread RF connectivity           PASS
```

### After FortiGate OMR Routing Correction

Commissioning progressed farther and attempted to connect to Home Assistant.

Android then produced a generic:

```text
Something went wrong
```

### Reverse-Direction Network Diagnosis

Testing the actual Aqara OMR address from HAOS then exposed:

```text
Denied by forward policy check (policy 0)
```

The missing HAOS-to-Thread IPv6 policy was corrected.

### Final Result

After the required routing and firewall paths were corrected, Matter commissioning completed successfully and the H2 became operational in Home Assistant.

Validation Result: Commissioning progression directly tracked the removal of successive segmented-network blockers.

---

## Why "Can't Reach Device" Did Not Mean Thread RF Was Broken

A failed Matter commissioning dialog does not identify the exact network layer that failed.

During the failed attempt, OTBR showed:

```text
Child attached
LQ In: 3
```

and the endpoint had an OMR address:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

Therefore, at that stage, the Aqara had:

```text
Joined Thread              YES
Received OMR IPv6          YES
Strong Thread link         YES
Completed Matter setup     NOT YET
```

The failure existed farther along the path.

This is an important troubleshooting distinction.

Matter commissioning was subsequently completed after the routed and policy-controlled paths were corrected.

Validation Result: Thread attachment must be validated independently of the Matter commissioning result.

---

## End-to-End Segmented Matter-over-Thread Path

The final routed path is:

```text
Home Assistant OS
<HA_IPV6_PREFIX>
        |
        | IPv6 default route
        v
FortiGate
        |
        | IPv6 firewall policy
        |
        | Static route:
        | <THREAD_OMR_PREFIX>
        | via Pi OTBR
        v
IoT Network
<IOT_IPV6_PREFIX>
        |
        v
Raspberry Pi 3
<OTBR_INFRA_IPV6>
        |
        v
OpenThread Border Router
        |
        v
Thread OMR
<THREAD_OMR_PREFIX>
        |
        v
Aqara H2
```

The reverse path is:

```text
Aqara H2
    |
Thread
    |
OTBR
    |
IoT
    |
FortiGate
    |
Home Assistant Network
    |
HAOS Matter Server
```

Validation Result: The required Layer 3 topology is explicitly routed and policy-controlled rather than flattened.

---

## Quick Start - Segmented Network

This section summarizes the successful implementation path without reproducing the troubleshooting sequence.

### 1. Prepare the Thread Radio

Flash the SONOFF Dongle Lite MG21 as:

```text
OpenThread RCP
2.4.4 Stable
460800 baud
No hardware flow control
```

Record its persistent serial path.

### 2. Prepare the Raspberry Pi

Install:

```text
Ubuntu Server 24.04 LTS ARM64
Docker Engine
Docker Compose
```

Run the upstream OTBR host setup for the infrastructure interface.

For this implementation:

```text
wlan0
```

### 3. Deploy OTBR

Use:

```text
Thread interface:          wpan0
Infrastructure interface:  wlan0
RCP:                       /dev/ttyUSB0
REST API:                  TCP 8081
```

Verify:

```bash
docker exec otbr ot-ctl state
docker exec otbr ot-ctl version
docker exec otbr ot-ctl rcp version
```

### 4. Add OTBR to Home Assistant

Add the external OTBR using:

```text
http://<OTBR-IP>:8081
```

Verify Home Assistant recognizes the Border Router.

### 5. Form or Select the Thread Network

Verify:

```bash
docker exec otbr ot-ctl state
```

Expected after formation:

```text
leader
```

### 6. Synchronize Thread Credentials

From Android Home Assistant Companion:

```text
Settings
-> Companion app
-> Troubleshooting
-> Sync Thread credentials
```

### 7. Determine the Thread OMR

Run:

```bash
docker exec otbr ot-ctl netdata show
```

Identify the OMR prefix.

Represent it in documentation as:

```text
<THREAD_OMR_PREFIX>
```

Do not use the mesh-local prefix as the infrastructure route.

### 8. Configure Segmented-Network Routing

This step may not be necessary on a flat network.

On the Layer 3 router, provide reachability to:

```text
<THREAD_OMR_PREFIX>
```

through the OTBR infrastructure-side IPv6 address.

Conceptually:

```text
<THREAD_OMR_PREFIX>
    via <OTBR_INFRA_IPV6>
    interface <IOT_INTERFACE>
```

### 9. Configure Segmented-Network Firewall Policy

This step may not be necessary on a flat network without an intervening firewall.

Permit the required communication between:

```text
HAOS IPv6
    <->
Thread OMR <THREAD_OMR_PREFIX>
```

Keep the policy scoped to the intended interfaces and endpoints/prefixes.

### 10. Verify Avahi if Segments Require mDNS Reflection

This step is not inherently required on a flat network.

Verify existing discovery with:

```bash
avahi-browse -art | grep -i matter
```

and verify OTBR `_meshcop._udp` advertisements.

### 11. Commission the Aqara

Place the H2 into commissioning mode.

Commission through the Home Assistant Companion App.

Monitor:

```bash
docker exec otbr ot-ctl child table
docker exec otbr ot-ctl childip
```

### 12. Validate the Actual Endpoint

Identify the Aqara OMR address.

Verify HAOS can route toward that address through the Layer 3 firewall/router.

### 13. Validate Matter

Complete commissioning and confirm:

```text
Entity created
HAOS ON
HAOS OFF
Physical ON
Physical OFF
State reporting
Restart recovery
```

Validation Result: The quick-start procedure separates universal Thread requirements from segmented-network-specific requirements.

---

## Diagnostic Commands

### OTBR State

```bash
docker exec otbr ot-ctl state
```

### OTBR Version

```bash
docker exec otbr ot-ctl version
```

### RCP Version

```bash
docker exec otbr ot-ctl rcp version
```

### Active Thread Dataset

```bash
docker exec otbr ot-ctl dataset active
```

Do not publish the complete output without redacting credentials.

### Thread Network Data

```bash
docker exec otbr ot-ctl netdata show
```

### Thread Children

```bash
docker exec otbr ot-ctl child table
```

### Thread Child IPv6 Addresses

```bash
docker exec otbr ot-ctl childip
```

### OTBR IPv6 Addresses

```bash
docker exec otbr ot-ctl ipaddr
```

### Pi IPv6 Addresses

```bash
ip -6 addr
```

### Pi IPv6 Routes

```bash
ip -6 route
```

### Local IPv6 Routes

```bash
ip -6 route show table local
```

### Route Lookup

```bash
ip -6 route get <IPv6-address>
```

### IPv6 Rules

```bash
ip -6 rule
```

### Pi Packet Capture

```bash
sudo tcpdump -ni <INFRA_INTERFACE> 'icmp6'
```

### Thread Interface Capture

```bash
sudo tcpdump -ni wpan0 'icmp6'
```

### Host IPv6 Forwarding Rules

```bash
sudo ip6tables -nvL FORWARD --line-numbers
```

### UFW

```bash
sudo ufw status verbose
```

### nftables

```bash
sudo nft list ruleset
```

Validation Result: The command set covers Thread state, child addressing, IPv6 routing, host forwarding, and packet-level diagnostics.

---

## FortiGate Diagnostic Commands

### Verify Thread OMR Route

Use a destination address within the Thread OMR rather than prefix notation when performing the route lookup:

```text
get router info6 routing-table <THREAD_OMR_TEST_IPV6>
```

Expected conceptually:

```text
Routing entry for <THREAD_OMR_PREFIX>
Known via "static"
best
via <OTBR_INFRA_IPV6>, <IOT_INTERFACE>
```

### IPv6 Packet Capture

Example:

```text
diagnose sniffer packet any 'host <IPv6-address> and icmp6' 4 0 l
```

### IPv6 Debug Flow

IPv6 debug filtering uses `filter6`.

Example:

```text
diagnose debug reset
diagnose debug flow filter6 clear
diagnose debug flow filter6 saddr <source-ipv6>
diagnose debug flow filter6 daddr <destination-ipv6>
diagnose debug flow show function-name enable
diagnose debug flow trace start6 20
diagnose debug enable
```

Generate the test traffic.

Then stop:

```text
diagnose debug disable
diagnose debug flow trace stop
diagnose debug flow filter6 clear
```

Validation Result: FortiGate debug flow was essential for distinguishing routing, RPF, and firewall-policy failures.

---

## Troubleshooting Matrix

| Symptom                                                             | Actual or Likely Cause                                    | Validation / Resolution                                      |
| ------------------------------------------------------------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| Android says a Thread Border Router is required                     | Thread network or credentials unavailable to commissioner | Verify OTBR integration and synchronize Thread credentials   |
| OTBR cannot communicate with radio                                  | RCP configuration or serial issue                         | Verify persistent serial path, RCP firmware, and 460800 baud |
| OTBR remains disabled                                               | Thread network not yet formed/provisioned                 | Verify HA Thread integration and dataset                     |
| OTBR becomes leader                                                 | Thread network operational                                | Continue endpoint commissioning                              |
| H2 appears in `child table`                                         | H2 attached to Thread                                     | Check LQ and `childip`                                       |
| `childip` contains an OMR address                                   | Endpoint received infrastructure-routable Thread address  | Use that address for routed testing                          |
| H2 joins Thread but Android says "Can't reach device"               | Failure above Thread attachment layer                     | Check OMR routing, firewall, Matter, and discovery           |
| Thread packet reaches FortiGate but is dropped                      | Missing route can cause RPF failure                       | Run FortiGate debug flow                                     |
| FortiGate reports `reverse path check failed, drop`                 | No valid route back to Thread OMR                         | Add appropriate OMR route through OTBR                       |
| FortiGate route lookup shows Thread OMR via Pi                      | OMR route installed                                       | Continue firewall validation                                 |
| Thread -> HAOS reports `Allowed by Policy`                          | Forward policy matched                                    | Verify HAOS response                                         |
| HAOS packet reaches FortiGate but does not leave IoT                | Reverse-direction firewall issue likely                   | Run IPv6 debug flow                                          |
| `Denied by forward policy check (policy 0)`                         | No matching HAOS -> Thread IPv6 policy                    | Add scoped IPv6 policy                                       |
| HAOS has no explicit Thread OMR route                               | Normal when FortiGate is IPv6 default gateway             | Verify HAOS default route                                    |
| HAOS sends Thread OMR traffic to FortiGate                          | Correct routing behavior                                  | FortiGate handles OMR next hop                               |
| OTBR OMR ping reply reaches Pi but is not seen forwarded on `wpan0` | OTBR OMR address is local                                 | Check Linux local route before assuming forwarding failure   |
| Host firewall is not the demonstrated drop point                    | Failure exists elsewhere in the path                      | Continue routed-path diagnostics                             |
| OTBR/Docker netfilter chains exist                                  | Expected host behavior                                    | Do not indiscriminately flush iptables/nftables              |
| Existing Matter-over-Wi-Fi works but Thread does not                | Thread introduces separate OMR prefix                     | Validate Thread-specific routing and policy                  |
| mDNS visible in Avahi but endpoint unreachable                      | Discovery and unicast routing are separate                | Troubleshoot IPv6 routing independently                      |
| `_meshcop._udp` visible                                             | OTBR discovery present                                    | Continue Thread and routing validation                       |
| `_matter._tcp` visible                                              | Matter operational discovery present                      | Continue operational connectivity testing                    |
| Aqara child disappears after failed commissioning/reset             | Endpoint detached from Thread                             | Re-enter commissioning and monitor `child table`             |

Validation Result: Troubleshooting is organized by actual protocol layer and observed failure rather than assuming all commissioning failures have the same cause.

---

## Key Troubleshooting Lesson: Follow the Packet

The most useful troubleshooting method in this deployment was validating each hop independently.

For Thread to HAOS:

```text
Thread stack
    |
    v
wpan0
    |
    v
Infrastructure interface
    |
    v
FortiGate IoT
    |
    v
FortiGate HAOS interface
    |
    v
HAOS
```

For HAOS to Thread:

```text
HAOS
    |
    v
FortiGate
    |
    v
IoT
    |
    v
Pi infrastructure interface
    |
    v
OTBR
    |
    v
Thread endpoint
```

A successful packet at one hop does not prove the next hop works.

Examples from this deployment:

```text
Packet reached FortiGate
```

did not prove:

```text
FortiGate accepted the packet
```

Likewise:

```text
FortiGate had a route
```

did not prove:

```text
Firewall policy allowed the packet
```

The actual debug messages removed ambiguity.

Validation Result: Packet captures and debug flow provided stronger evidence than interpreting generic Matter commissioning errors.

---

## Security Properties

The final architecture preserves the security intent of the existing segmented network.

### Thread Devices Remain Behind the IoT Boundary

Thread endpoints are reached through:

```text
Thread
    |
OTBR
    |
IoT Network
    |
FortiGate
```

They are not placed directly on the Home Assistant network.

### FortiGate Remains the Layer 3 Security Boundary

FortiGate controls routed traffic between:

```text
Home Assistant Network
IoT Network
Thread OMR
```

### OTBR Does Not Replace the Firewall

OTBR provides Thread border routing.

It does not replace FortiGate security policy.

### Avahi Does Not Replace the Router

Avahi reflects discovery traffic.

It does not provide the Thread OMR route.

### No Network Flattening

The implementation does not require:

```text
HAOS + OTBR + IoT devices
```

to share one unrestricted subnet.

### No Aqara Cloud Dependency

Normal Matter control remains local.

### No Aqara Hub

The H2 uses the vendor-neutral Matter-over-Thread infrastructure.

### Thread Prefix Is Explicitly Routed

The Thread OMR is represented as:

```text
<THREAD_OMR_PREFIX>
```

rather than broadly routing all ULA space.

A broad route such as:

```text
fc00::/7
```

is not required for this implementation and would be unnecessarily expansive.

Validation Result: The implementation adds only the routing required for the actual Thread OMR rather than weakening the overall network segmentation model.

---

## Failure Domains

### Thread RCP Failure

If the Thread LMG21 fails:

```text
Thread mesh connectivity affected
Zigbee unaffected
Matter-over-Wi-Fi unaffected
```

### Raspberry Pi 3 Failure

If the Pi fails:

```text
OTBR unavailable
Matter-over-Thread unavailable
Zigbee unaffected
Matter-over-Wi-Fi unaffected
```

### IoT Wi-Fi Failure

If the Pi loses IoT Wi-Fi:

```text
Thread RF mesh may remain formed
OTBR infrastructure connectivity is lost
HAOS cannot communicate with Thread endpoints through this TBR
```

### FortiGate Route Failure

If the Thread OMR route is removed:

```text
FortiGate may fail RPF for Thread-originated traffic
HAOS-to-Thread routing becomes unavailable
```

### FortiGate Policy Failure

If either required policy direction is removed:

```text
One direction of Thread/HAOS communication may fail
Matter commissioning or operation may fail
```

### Avahi Failure

If cross-segment discovery reflection fails:

```text
Matter/OTBR discovery across segments may fail
Basic unicast IPv6 routing may remain operational
```

### Home Assistant Failure

If HAOS fails:

```text
Matter controller unavailable
Home Assistant automations unavailable
```

Physical H2 switching is expected to remain available independently of Home Assistant.

Validation Result: The architecture maintains distinct failure domains for Thread RF, OTBR, routing, firewall policy, discovery, and Matter control.

---

## Final Validation

The completed deployment was validated against the following checks:

| Test                    | Expected Result                                   |
| ----------------------- | ------------------------------------------------- |
| Physical ON             | Load energizes                                    |
| Physical OFF            | Load de-energizes                                 |
| HAOS ON                 | Load energizes                                    |
| HAOS OFF                | Load de-energizes                                 |
| State reporting         | Home Assistant reflects switch state              |
| Thread child attachment | Stable                                            |
| Thread OMR address      | Present                                           |
| Matter control          | Stable                                            |
| HAOS restart recovery   | Device reconnects                                 |
| Pi restart recovery     | OTBR and Thread connectivity return               |
| OTBR restart recovery   | Matter control returns                            |
| IoT Wi-Fi reconnect     | OTBR infrastructure connectivity returns          |
| Internet unavailable    | Normal local control remains operational          |
| Aqara hub absent        | Normal Home Assistant control remains operational |
| FortiGate segmentation  | Preserved                                         |
| Avahi discovery         | Operational where required                        |

Validation Result: Final PASS status was recorded after the physical H2 and Home Assistant Matter entity were confirmed operational.

---

## Important Lessons From This Deployment

### Thread Attachment and Matter Commissioning Are Different Milestones

An endpoint appearing in:

```bash
docker exec otbr ot-ctl child table
```

proves Thread attachment.

It does not prove Matter commissioning is complete.

### Matter-over-Wi-Fi Does Not Prove Matter-over-Thread Routing

Matter-over-Wi-Fi endpoints exist directly on an infrastructure prefix.

Thread endpoints exist behind the OTBR on an OMR prefix.

That introduces an additional routing relationship.

### Discovery and Routing Are Separate

Avahi can successfully reflect:

```text
_meshcop._udp
_matter._tcp
_matterc._udp
```

while unicast IPv6 routing is still broken.

Likewise, correct IPv6 routing does not automatically prove discovery is working.

### Routing and Firewall Policy Are Separate

The FortiGate can know exactly where a Thread network exists and still deny traffic because no firewall policy permits the flow.

### Reverse-Path Validation Matters

The initial Thread-to-HAOS failure was explicitly:

```text
reverse path check failed, drop
```

The Thread OMR static route corrected that condition.

### Both Directions Must Be Tested

The corrected Thread-to-HAOS path did not automatically establish HAOS-to-Thread connectivity.

The reverse direction independently failed with:

```text
Denied by forward policy check (policy 0)
```

### Do Not Flatten the Network Just Because Commissioning Fails

The deployment demonstrated that Matter over Thread can be integrated into the existing segmented architecture by solving the actual routing and policy requirements.

### Do Not Disable Host Firewall Infrastructure Without Evidence

Docker and OTBR netfilter rules exist for legitimate reasons.

Flushing iptables or nftables would introduce another variable rather than fix an upstream routing or firewall-policy failure demonstrated elsewhere in the path.

### Generic Matter Errors Are Not Root-Cause Diagnostics

Messages such as:

```text
Can't reach device
```

and:

```text
Something went wrong
```

did not identify the failed network layer.

Thread child tables, IPv6 routes, packet captures, and FortiGate debug flow provided the actual evidence.

Validation Result: Layer-by-layer validation was required to preserve segmentation while successfully building the Thread infrastructure.

---

## Operational Boundaries

The architecture deliberately separates several technologies.

### Zigbee

```text
Device
  |
Zigbee
  |
Coordinator
  |
ZHA
```

Zigbee terminates at Home Assistant.

### Thread

```text
Device
  |
Thread
  |
OTBR
  |
IPv6
```

Thread is routed into the IP infrastructure.

### Matter

```text
Endpoint
   |
Matter
   |
Matter Server
   |
Home Assistant
```

Matter provides the application-level smart-home relationship.

### mDNS

```text
Network Segment
     |
   Avahi
     |
Network Segment
```

mDNS provides discovery across the existing segmented network where configured.

### FortiGate

```text
Network
   |
Routing
   |
Firewall Policy
   |
Network
```

FortiGate remains the Layer 3 security boundary.

These components are related but perform separate functions.

Validation Result: Protocol and infrastructure boundaries are explicitly documented to prevent Zigbee, Thread, Matter, OTBR, Avahi, and FortiGate routing from being treated as interchangeable services.

---

## Future Expansion

The infrastructure provides Home Assistant with reusable local Thread capability.

After deployment:

```text
Home Assistant
     |
     +-- Wi-Fi
     |
     +-- Bluetooth / BLE
     |
     +-- Matter-over-Wi-Fi
     |
     +-- Zigbee
     |
     `-- Thread
           |
           `-- Matter-over-Thread
```

Future Matter-over-Thread endpoints can use:

```text
Home Assistant Matter Server
        |
        v
FortiGate
        |
        v
Raspberry Pi 3 OTBR
        |
        v
Thread Mesh
```

The Thread OMR route represents the Thread network rather than one individual Aqara endpoint.

Therefore another Thread device receiving an address within:

```text
<THREAD_OMR_PREFIX>
```

falls within the existing route.

A separate FortiGate static route is not required for every individual Thread endpoint as long as the Thread OMR remains unchanged.

Validation Result: The deployment creates reusable Thread infrastructure rather than a network path dedicated only to one H2 switch.

---

## Deployment Placeholder Reference

The following placeholders are intentionally used throughout this document to represent deployment-specific values:

| Placeholder                         | Meaning                                       |
| ----------------------------------- | --------------------------------------------- |
| `<HA_IPV6_PREFIX>`                  | Home Assistant infrastructure IPv6 prefix     |
| `<IOT_IPV6_PREFIX>`                 | IoT infrastructure IPv6 prefix                |
| `<THREAD_OMR_PREFIX>`               | Thread Off-Mesh-Routable IPv6 prefix          |
| `<THREAD_MESH_LOCAL_PREFIX>`        | Thread mesh-local IPv6 prefix                 |
| `<HAOS_IPV6>`                       | Home Assistant OS IPv6 address                |
| `<OTBR_INFRA_IPV6>`                 | OTBR infrastructure-side IPv6 address         |
| `<OTBR_OMR_IPV6>`                   | OTBR Thread OMR IPv6 address                  |
| `<THREAD_ENDPOINT_OMR_IPV6>`        | Thread endpoint OMR IPv6 address              |
| `<THREAD_ENDPOINT_MESH_LOCAL_IPV6>` | Thread endpoint mesh-local IPv6 address       |
| `<THREAD_RCP_DEVICE_ID>`            | Persistent USB serial/by-id identifier        |
| `<THREAD_NETWORK_NAME>`             | Thread network name                           |
| `<THREAD_CHILD_EXT_MAC>`            | Thread child Extended MAC                     |
| `<RLOC16>`                          | Thread child RLOC16                           |
| `<CHILD_ID>`                        | Thread child table identifier                 |
| `<OTBR_MDNS_HOSTNAME>`              | OTBR mDNS hostname                            |
| `<INSTANCE_ID>`                     | OTBR advertised instance identifier           |
| `<IOT_INTERFACE>`                   | Firewall/router IoT-side interface            |
| `<HA_NETWORK_INTERFACE>`            | Firewall/router Home Assistant-side interface |
| `<POLICY_ID>`                       | Firewall policy identifier                    |
| `<AVAHI_INTERFACE_1>`               | First Avahi reflector interface               |
| `<AVAHI_INTERFACE_2>`               | Second Avahi reflector interface              |
| `<THREAD_OMR_TEST_IPV6>`            | Test address within the Thread OMR            |

Do not replace these placeholders with production values when publishing or sharing this document.

---

## Related Search Keywords

home-assistant, matter, matter-over-thread, thread, openthread, otbr, aqara, fortigate, ipv6, smart-home, home-automation

---

## References

Primary upstream documentation should be consulted during installation and maintenance because Home Assistant, Matter, Thread, OTBR, Docker, FortiOS, and device firmware behavior can change over time.

* Aqara:

```text
https://www.aqara.com/
```

* Home Assistant Matter:

```text
https://www.home-assistant.io/integrations/matter/
```

* Home Assistant Thread:

```text
https://www.home-assistant.io/integrations/thread/
```

* Home Assistant OpenThread Border Router:

```text
https://www.home-assistant.io/integrations/otbr
```

* Home Assistant ZHA:

```text
https://www.home-assistant.io/integrations/zha/
```

* SONOFF Dongle documentation and firmware flasher:

```text
https://dongle.sonoff.tech/
```

* OpenThread:

```text
https://openthread.io/
```

Implementation-specific configuration should be validated against the currently installed versions before future changes are made.

Validation Result: Upstream product and platform documentation remains authoritative for version-dependent behavior.

---

## Revision Control

| Version | Date       | Summary                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Author      |
| ------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- |
| 1.0.0   | 2026-08-29 | Initial Aqara Light Switch H2 US no-neutral Matter-over-Thread implementation design, dedicated SONOFF LMG21 Zigbee coordinator, Raspberry Pi 3 external OTBR architecture, local-first control model, and integration with the existing segmented Home Assistant Matter environment.                                                                                                                                                                                                                                                                  | projectfong |
| 1.1.0   | 2026-08-29 | Expanded the planned Thread design into the deployed Raspberry Pi 3 OTBR implementation. Added Ubuntu Server 24.04 LTS ARM64, SONOFF LMG21 OpenThread RCP 2.4.4, Docker OTBR deployment, external HAOS OTBR integration, Thread credential synchronization, Thread OMR identification, FortiGate IPv6 static routing, reverse-path-forwarding diagnosis, bidirectional segmented-network firewall requirements, Aqara Thread child validation, packet-level troubleshooting, and explicit distinction between segmented and flat-network requirements. | projectfong |
| 1.2.0   | 2026-08-30 | Completed Matter commissioning and physical H2 validation. Updated deployment status to operational and replaced deployment-specific network, Thread, hardware, mDNS, and firewall identifiers with reusable documentation placeholders.                                                                                                                                                                                                                                                                                                               | projectfong |

---
