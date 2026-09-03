# Aqara H2 Matter-over-Thread with Home Assistant - Quickstart

Author: projectfong  
Copyright (c) 2026 projectfong

---

## Summary

Deploy an Aqara Light Switch H2 US with Home Assistant using Matter over Thread, an external Raspberry Pi OpenThread Border Router (OTBR), and a dedicated SONOFF Dongle Lite MG21 OpenThread RCP.

This quickstart covers the validated segmented-network deployment where Home Assistant and the OTBR reside on different network segments.

The final path is:

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
    | OpenThread RCP
    v
Raspberry Pi / OTBR
    |
    v
IoT Network
    |
    v
Layer 3 Router / Firewall
    |
    v
Home Assistant OS
    |
    v
Matter Server
```

Estimated time:

```text
60-120 minutes
```

Additional time may be required for segmented-network routing or firewall troubleshooting.

## Requirements

Before starting:

- Home Assistant OS.
- Home Assistant Matter Server.
- Home Assistant Companion App on Android.
- Aqara Light Switch H2 US.
- Raspberry Pi or equivalent Linux OTBR host.
- Ubuntu Server 24.04 LTS ARM64 or equivalent supported Linux environment.
- Docker Engine.
- Docker Compose.
- SONOFF Dongle Lite MG21 dedicated to Thread.
- IPv6 connectivity between Home Assistant and the OTBR infrastructure network.
- Administrative access to the Layer 3 router/firewall.

For segmented networks, also expect to configure:

```text
Thread OMR IPv6 routing
HAOS -> Thread firewall policy
Thread -> HAOS firewall policy
Cross-segment mDNS where required
```

## Electrical Safety

The H2 is a mains-powered wall switch.

Before installation:

1. Turn the correct branch circuit OFF.
2. Verify the circuit is de-energized with an appropriate voltage tester.
3. Identify line, load, neutral if present, and the equipment-grounding path.
4. Follow the Aqara wiring instructions for the actual circuit.
5. Do not identify conductors solely by wire color.
6. Never use an equipment-grounding conductor as a neutral.

If the wiring cannot be positively identified, stop electrical work until the circuit can be properly evaluated.

## Quick Setup

### 1. Prepare the Thread Radio

Flash the SONOFF Dongle Lite MG21 as an OpenThread RCP.

The validated deployment used:

```text
Firmware:
OpenThread RCP

Version:
2.4.4 Stable

UART:
460800

Hardware flow control:
Disabled
```

After connecting the dongle to the OTBR host, identify its persistent device path:

```bash
ls -l /dev/serial/by-id/
```

Use the persistent path rather than relying on:

```text
/dev/ttyUSB0
```

because USB enumeration can change.

Example placeholder:

```text
/dev/serial/by-id/<THREAD_RCP_DEVICE_ID>
```

## 2. Prepare the OTBR Host

Install:

```text
Ubuntu Server 24.04 LTS ARM64
Docker Engine
Docker Compose
```

Determine the infrastructure interface.

The validated deployment uses:

```text
wlan0
```

Run the upstream OpenThread host setup:

```bash
curl -sSL https://raw.githubusercontent.com/openthread/ot-br-posix/refs/heads/main/etc/docker/border-router/setup-host | INFRA_IF_NAME=wlan0 bash
```

Verify IPv6 forwarding:

```bash
sysctl net.ipv6.conf.all.forwarding
```

Expected:

```text
net.ipv6.conf.all.forwarding = 1
```

## 3. Deploy OTBR

Create the OTBR Compose configuration.

```yaml
services:
    otbr:
        image: openthread/border-router
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

Replace:

```text
<THREAD_RCP_DEVICE_ID>
```

with the persistent serial identifier for the Thread radio.

Start OTBR:

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

The `otbr` container should be running.

## 4. Verify the Thread Radio

Check OTBR state:

```bash
docker exec otbr ot-ctl state
```

Check the OpenThread version:

```bash
docker exec otbr ot-ctl version
```

Check RCP communication:

```bash
docker exec otbr ot-ctl rcp version
```

Successful RCP version output confirms OTBR can communicate with the MG21 radio.

## 5. Verify the OTBR REST API

OTBR exposes its REST API on:

```text
TCP/8081
```

Verify:

```bash
ss -lntp | grep 8081
```

Test the API:

```bash
curl http://<OTBR-IP>:8081/.well-known/thread/br-rest
```

The API should respond with OTBR REST information.

## 6. Add the External OTBR to Home Assistant

In Home Assistant, configure the OpenThread Border Router integration using:

```text
http://<OTBR-IP>:8081
```

Verify Home Assistant recognizes the external Border Router.

## 7. Form the Thread Network

After Home Assistant provisions the Thread network, check:

```bash
docker exec otbr ot-ctl state
```

Expected operational state:

```text
leader
```

Do not publish the complete Thread Active Operational Dataset.

Protect:

```text
Network Key
PSKc
Complete Active Operational Dataset
Matter setup credentials
```

## 8. Synchronize Thread Credentials

On the Android Home Assistant Companion App:

```text
Settings
-> Companion app
-> Troubleshooting
-> Sync Thread credentials
```

Confirm that the Thread credentials are added to the Android device.

This step is required before the phone can properly commission the Matter-over-Thread endpoint onto the intended Thread network.

## 9. Identify the Thread OMR Prefix

Run:

```bash
docker exec otbr ot-ctl netdata show
```

Identify the Thread Off-Mesh-Routable prefix.

Represent it publicly as:

```text
<THREAD_OMR_PREFIX>
```

Do not confuse the OMR prefix with the Thread mesh-local prefix.

For infrastructure routing, the important network is:

```text
<THREAD_OMR_PREFIX>
```

## Segmented Network Setup

Skip or adapt these steps if Home Assistant and the OTBR are on a flat network without an intervening Layer 3 firewall.

The segmented topology is:

```text
Home Assistant Network
        |
        v
Layer 3 Router / Firewall
        |
        v
IoT Network
        |
        v
OTBR
        |
        v
Thread OMR
```

## 10. Route the Thread OMR

The Layer 3 router must know that:

```text
<THREAD_OMR_PREFIX>
```

is reachable through:

```text
<OTBR_INFRA_IPV6>
```

Conceptually:

```text
Destination:
<THREAD_OMR_PREFIX>

Next hop:
<OTBR_INFRA_IPV6>

Interface:
<IOT_INTERFACE>
```

Do not create a broad route such as:

```text
fc00::/7
```

when only the actual Thread OMR needs to be routed.

## 11. Configure Firewall Policy

Routing and firewall authorization are separate requirements.

Permit the required IPv6 communication between:

```text
HAOS
<->
Thread OMR
```

Conceptually:

```text
HAOS IPv6
    |
    v
<THREAD_OMR_PREFIX>
```

and:

```text
<THREAD_OMR_PREFIX>
    |
    v
HAOS IPv6
```

Keep rules scoped to the required:

```text
Source
Destination
Interface
Protocol/service requirements
```

Do not flatten the network simply to make Matter commissioning work.

## 12. Verify Cross-Segment Discovery

If the network requires mDNS reflection, verify the existing reflector can see Matter and OTBR advertisements.

Example:

```bash
avahi-browse -art | grep -i matter
```

Also verify:

```text
_meshcop._udp
```

for OTBR discovery.

An mDNS reflector is not required merely because Matter over Thread is being used.

It is required only where the network architecture prevents required multicast discovery from crossing segments normally.

## 13. Install the Aqara H2

Install the H2 according to:

```text
Actual circuit topology
Aqara installation instructions
Applicable electrical requirements
```

The H2 supports neutral and no-neutral installation.

Do not assume every switch box has the same wiring configuration.

Restore power only after installation has been verified.

## 14. Commission the Aqara H2

Place the H2 into commissioning mode.

Use the Home Assistant Companion App to add the Matter device.

While commissioning, monitor the Thread network:

```bash
docker exec otbr ot-ctl child table
```

Then check registered child addresses:

```bash
docker exec otbr ot-ctl childip
```

An attached H2 should appear as a Thread child.

The endpoint should also receive an address within:

```text
<THREAD_OMR_PREFIX>
```

## 15. Validate Thread Attachment

Check:

```bash
docker exec otbr ot-ctl child table
```

Thread child presence proves:

```text
Thread attachment:
PASS
```

Check:

```bash
docker exec otbr ot-ctl childip
```

An endpoint OMR address proves:

```text
Thread OMR addressing:
PASS
```

This does not by itself prove Matter commissioning is complete.

## 16. Validate End-to-End IPv6

Identify the actual endpoint OMR address:

```text
<THREAD_ENDPOINT_OMR_IPV6>
```

Verify the Layer 3 path:

```text
HAOS
  |
  v
Router / Firewall
  |
  v
OTBR
  |
  v
Thread Endpoint
```

Also verify the reverse direction:

```text
Thread Endpoint
  |
  v
OTBR
  |
  v
Router / Firewall
  |
  v
HAOS
```

Both directions must work.

## 17. Complete Matter Commissioning

After Thread attachment and routed IPv6 connectivity are operational, complete Matter commissioning.

Home Assistant should create the H2 device and corresponding entities.

## Validate

Confirm:

```text
[ ] OTBR container running
[ ] OTBR communicates with MG21 RCP
[ ] OTBR REST API reachable
[ ] Home Assistant recognizes external OTBR
[ ] Thread network operational
[ ] OTBR reports leader
[ ] Android Thread credentials synchronized
[ ] Thread OMR identified
[ ] Layer 3 router knows route to Thread OMR
[ ] HAOS -> Thread firewall path works
[ ] Thread -> HAOS firewall path works
[ ] Aqara appears in Thread child table
[ ] Aqara receives Thread OMR address
[ ] Matter commissioning completes
[ ] Home Assistant entity created
[ ] HAOS ON works
[ ] HAOS OFF works
[ ] Physical ON works
[ ] Physical OFF works
[ ] State reporting works
```

## Common Problems

### Android Says a Thread Border Router Is Required

Verify:

```text
OTBR is running
External OTBR is configured in Home Assistant
Thread network exists
Android Thread credentials are synchronized
```

Then repeat:

```text
Settings
-> Companion app
-> Troubleshooting
-> Sync Thread credentials
```

### OTBR Cannot Communicate with the Radio

Verify:

```bash
docker exec otbr ot-ctl rcp version
```

If communication fails, check:

```text
Persistent USB device path
OpenThread RCP firmware
460800 baud
Hardware flow control disabled
Docker device mapping
```

### Aqara Appears in `child table` but Commissioning Fails

If:

```bash
docker exec otbr ot-ctl child table
```

shows the H2, basic Thread attachment is already working.

Check:

```bash
docker exec otbr ot-ctl childip
```

If the H2 has an OMR address, investigate:

```text
Thread OMR routing
IPv6 firewall policy
Matter communication
mDNS discovery where required
```

Do not assume Thread RF is broken.

### Thread Traffic Reaches the Firewall but Is Dropped

Verify the Layer 3 router has a route for:

```text
<THREAD_OMR_PREFIX>
```

through:

```text
<OTBR_INFRA_IPV6>
```

A missing route can also cause reverse-path validation failures on stateful firewalls.

### Router Knows the OMR Route but Traffic Still Fails

A valid route does not automatically permit traffic.

Verify firewall policy for:

```text
HAOS -> Thread OMR
```

and:

```text
Thread OMR -> HAOS
```

### Matter-over-Wi-Fi Works but Thread Does Not

Matter-over-Wi-Fi endpoints normally reside directly on an infrastructure IPv6 prefix.

Matter-over-Thread endpoints reside behind the OTBR on:

```text
<THREAD_OMR_PREFIX>
```

Verify the additional routed Thread network.

### mDNS Works but the Device Is Unreachable

Discovery and unicast routing are separate.

Successful:

```text
_meshcop._udp
_matter._tcp
```

discovery does not prove IPv6 routing to the endpoint works.

Validate the routed IPv6 path separately.

## Diagnostic Commands

OTBR state:

```bash
docker exec otbr ot-ctl state
```

OpenThread version:

```bash
docker exec otbr ot-ctl version
```

RCP version:

```bash
docker exec otbr ot-ctl rcp version
```

Thread network data:

```bash
docker exec otbr ot-ctl netdata show
```

Thread children:

```bash
docker exec otbr ot-ctl child table
```

Thread child addresses:

```bash
docker exec otbr ot-ctl childip
```

OTBR IPv6 addresses:

```bash
docker exec otbr ot-ctl ipaddr
```

Host IPv6 addresses:

```bash
ip -6 addr
```

Host IPv6 routes:

```bash
ip -6 route
```

IPv6 route lookup:

```bash
ip -6 route get <IPv6-address>
```

Infrastructure packet capture:

```bash
sudo tcpdump -ni <INFRA_INTERFACE> 'icmp6'
```

Thread interface packet capture:

```bash
sudo tcpdump -ni wpan0 'icmp6'
```

## Do Not Do This

Do not troubleshoot by indiscriminately running:

```bash
iptables -F
ip6tables -F
nft flush ruleset
```

Docker and OTBR depend on host netfilter configuration.

Do not flatten the network before identifying the actual failed layer.

Do not publish:

```text
Thread Network Key
PSKc
Complete Active Operational Dataset
Matter setup credentials
Production IPv6 prefixes
Production host addresses
USB serial identifiers
Internal hostnames
Firewall policy identifiers
```

## Final Architecture

```text
Aqara H2
    |
    | Matter over Thread
    v
Thread Mesh
    |
    v
SONOFF MG21 RCP
    |
    v
Raspberry Pi OTBR
    |
    | IPv6
    v
IoT Network
    |
    v
Layer 3 Router / Firewall
    |
    | Routed and policy-controlled IPv6
    v
Home Assistant Network
    |
    v
Home Assistant OS
    |
    v
Matter Server
```

Normal H2 control remains local through the Matter-over-Thread infrastructure.

An Aqara proprietary hub is not required for this architecture.

## Full Documentation

This quickstart intentionally omits the detailed electrical selection rationale, Matter/Thread protocol explanation, OMR discovery process, FortiGate RPF investigation, packet captures, extended troubleshooting sequence, failure-domain analysis, and architecture discussion.

See the full Aqara Light Switch H2 US Matter-over-Thread implementation documentation for those details.

## Related Search Keywords

home-assistant, haos, aqara-h2, matter, matter-over-thread, thread, openthread, otbr, sonoff-mg21, ipv6, thread-omr, segmented-network, mdns

## Revision Control

| Version | Date | Summary | Author |
| ------- | ---- | ------- | ------ |
| 1.0.0 | 2026-08-31 | Initial Aqara H2 Matter-over-Thread segmented-network quickstart. | projectfong |
