# GoveeLife Button and Tapo H100 Home Assistant Doorbell - Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Build a local Home Assistant-controlled doorbell using:

```text
GoveeLife B5122 Wireless Button
        |
        | Bluetooth
        v
ESPHome Bluetooth Proxy
        |
        v
Home Assistant OS
        |
        | Automation
        v
TP-Link Tapo H100
        |
        v
Audible Doorbell Chime
```

The physical button and chime do not communicate directly.

Home Assistant receives the GoveeLife button event and independently activates the Tapo H100.

The documented deployment operates with the H100 blocked from Internet access after provisioning.

## Requirements

- Home Assistant OS.
- GoveeLife Wireless Mini Smart Button Sensor B5122.
- TP-Link Tapo H100.
- ESP32 configured as an ESPHome Bluetooth proxy.
- Govee Bluetooth integration.
- TP-Link Smart Home integration.
- 2.4 GHz IoT Wi-Fi.
- Android or iOS device capable of running the Tapo application for initial H100 provisioning.
- Firewall capable of controlling H100 Internet access.

The documented deployment uses two ESP32 Bluetooth proxies, but two are not required if one provides adequate Bluetooth coverage.

## Important Device Identification

The physical GoveeLife product is:

```text
B5122
```

Home Assistant reports the device as:

```text
H5122
```

Both identifiers refer to the button used in this deployment.

## Quick Setup

### 1. Verify Bluetooth Proxy Infrastructure

Verify at least one ESPHome Bluetooth proxy is online in Home Assistant.

The required input path is:

```text
GoveeLife Button
        |
        | Bluetooth
        v
ESP32 Bluetooth Proxy
        |
        | ESPHome
        v
Home Assistant
```

The button does not require Wi-Fi.

### 2. Add the GoveeLife Button

Place the B5122 within usable Bluetooth range of an ESPHome Bluetooth proxy.

In Home Assistant, verify the:

```text
Govee Bluetooth
```

integration detects the device.

The documented deployment reports:

```text
Physical designation:
B5122

Home Assistant model:
H5122
```

Press the button and verify Home Assistant receives the button event.

Do not continue until the event is reliably received.

## H100 Provisioning

The following H100 provisioning behavior was observed on the documented deployment.

It should not be treated as an officially supported TP-Link provisioning method or as guaranteed behavior across firmware or application versions.

### 3. Prepare the IoT Network

The desired steady-state policy is:

```text
H100 -> Local network:       ALLOW
H100 -> Home Assistant path: ALLOW
H100 -> Internet/WAN:        BLOCK
```

During the documented provisioning process:

```text
Phone Internet access:
ALLOWED

H100 Internet access:
BLOCKED
```

This distinction is important.

### 4. Begin Tapo Provisioning

Temporarily install the Tapo application and authenticate with a TP-Link ID.

Allow the phone running the application to access the Internet.

Begin provisioning the H100.

Provide the H100 with the intended 2.4 GHz IoT Wi-Fi configuration.

### 5. Allow the H100 to Join Wi-Fi

The documented deployment produced:

```text
Wi-Fi provisioning:       SUCCESS
Local network access:     SUCCESS
TP-Link cloud onboarding: FAILED
Tapo application binding: NOT OBSERVED
```

The H100 joined the IoT Wi-Fi network even though cloud onboarding did not complete.

Verify the H100 receives an IP address before continuing.

### 6. Verify H100 Local Reachability

Confirm the H100 remains connected to the IoT network.

The documented deployment subsequently showed:

```text
Tapo application:
Devices: 0
```

while the H100 remained locally reachable.

Do not assume another H100, firmware version, or Tapo application version will behave identically.

## Add the H100 to Home Assistant

### 7. Add the TP-Link Integration

In Home Assistant:

```text
Settings
-> Devices & services
-> Add Integration
-> TP-Link Smart Home
```

If automatic discovery does not locate the H100, enter its IP address.

Example:

```text
192.0.2.100
```

Enter only the hostname or IP address.

Do not enter:

```text
http://192.0.2.100
```

### 8. Verify H100 Control

After integration, verify the H100 exposes the required entities.

The documented deployment provides:

```text
Siren control
Alarm sound selection
Duration control
```

Directly activate the H100 from Home Assistant.

Confirm an audible tone plays.

Also verify the alarm sound can be changed if that functionality is required.

## Create the Doorbell Automation

### 9. Create the Trigger

Create a new Home Assistant automation.

Use the GoveeLife button event as the trigger.

Conceptually:

```text
WHEN

GoveeLife Button 1
event received
```

Use the actual device/event exposed by the local Home Assistant installation.

Do not copy assumed device IDs, entity IDs, or event identifiers from another deployment.

### 10. Add the H100 Action

Configure:

```text
THEN

Turn on siren

Target:
Tapo Chime

Duration:
1 second
```

The resulting automation is:

```text
GoveeLife Button
        |
        | Button event
        v
Home Assistant
        |
        | Turn on siren
        | Duration: 1 second
        v
Tapo H100
        |
        v
Doorbell Tone
```

Save the automation.

### 11. Test the Doorbell

Press the GoveeLife button.

Expected:

```text
Button press
    |
    v
Bluetooth event
    |
    v
ESPHome Bluetooth Proxy
    |
    v
Govee Bluetooth
    |
    v
Home Assistant Automation
    |
    v
TP-Link Smart Home
    |
    v
H100 Chime
```

The H100 should activate for approximately one second.

## Validate

Confirm:

```text
[ ] ESPHome Bluetooth proxy online
[ ] Govee Bluetooth integration active
[ ] B5122/H5122 visible in Home Assistant
[ ] Button press event received
[ ] H100 connected to IoT Wi-Fi
[ ] H100 available through TP-Link Smart Home
[ ] Direct H100 siren control works
[ ] H100 alarm sound control works, if required
[ ] Doorbell automation executes
[ ] H100 rings for 1 second
[ ] Physical button consistently triggers the chime
```

## Power-Cycle Test

Disconnect power from the H100.

Wait several seconds and restore power.

Verify:

```text
H100 boots
    |
    v
Reconnects to IoT Wi-Fi
    |
    v
Home Assistant reconnects
    |
    v
Direct H100 control works
    |
    v
GoveeLife button rings H100
```

The documented deployment passed this test.

## WAN-Isolated Operation

After provisioning, the documented H100 operates with:

```text
Local network:
ALLOWED

Required Home Assistant communication:
ALLOWED

Internet/WAN:
BLOCKED
```

Do not broadly enable H100 Internet access merely because local control stops working.

First troubleshoot:

```text
H100 power
Wi-Fi association
IP address
Routing
Firewall policy
TP-Link Smart Home integration
Home Assistant logs
```

## TP-Link Account Status

The documented deployment used a temporary TP-Link ID during provisioning.

A permanent deletion request was subsequently submitted.

At the time of the source documentation:

```text
TP-Link ID deletion request:
SUBMITTED

Permanent deletion:
PENDING

Operation after confirmed deletion:
NOT YET VERIFIED
```

Therefore, do not claim that the documented H100 is verified to operate indefinitely without a TP-Link account.

If reproducing this configuration, validate account-independent operation before relying on it.

## Post-Deletion Validation

If the temporary TP-Link ID is permanently deleted, verify:

```text
[ ] H100 remains available in Home Assistant
[ ] Direct siren activation works
[ ] Alarm sound control works
[ ] Doorbell automation works
[ ] H100 remains blocked from WAN
[ ] H100 survives a power cycle
[ ] Doorbell works after H100 power cycle
[ ] TP-Link Smart Home integration reload succeeds
[ ] Doorbell works after integration reload
[ ] Home Assistant restart succeeds
[ ] Bluetooth proxy infrastructure returns
[ ] GoveeLife button returns
[ ] H100 returns
[ ] Doorbell works after Home Assistant restart
```

Only after these tests pass should the deployment be described as validated for continued operation after TP-Link account deletion.

## Bluetooth Placement

The B5122/H5122 depends on adequate Bluetooth reception.

If button events are intermittent:

```text
1. Verify the ESP32 Bluetooth proxy is online.
2. Verify the H5122 remains available.
3. Press the button several times.
4. Confirm Home Assistant receives each event.
5. Test the button closer to a Bluetooth proxy.
6. Review Bluetooth diagnostics or RSSI where available.
```

Do not troubleshoot the H100 until the button event itself has been confirmed.

The documented deployment has two Bluetooth proxies, but individual proxy redundancy for this button has not been independently validated.

## Environmental Placement

The documented B5122/H5122 has not been tested for weatherproof or water-resistant operation.

Do not assume it is suitable for unprotected outdoor exposure based on this deployment.

Install it in an appropriately protected location unless the specific hardware documentation establishes otherwise.

## Common Problems

### Button Press Does Not Ring H100

First test the H100 directly from Home Assistant.

```text
Home Assistant
    |
    v
Tapo H100 siren
```

If the H100 rings, troubleshoot:

```text
GoveeLife button
Bluetooth reception
ESP32 Bluetooth proxy
Govee Bluetooth integration
Button event
Automation trigger
Automation trace
```

If the H100 does not ring directly, troubleshoot the H100 path instead.

### GoveeLife Button Is Unavailable

Check:

```text
Button battery
Bluetooth range
ESP32 Bluetooth proxy status
ESPHome connectivity
Govee Bluetooth integration
```

### H100 Is Unavailable

Check:

```text
H100 power
IoT Wi-Fi association
IP address
DHCP lease
HAOS routing
Firewall policy
TP-Link Smart Home integration
```

Do not immediately restore Internet access.

### Doorbell Automation Does Not Execute

Verify the button event first.

Then inspect:

```text
Settings
-> Automations & scenes
-> Doorbell automation
-> Traces
```

Confirm the trigger fired and the H100 siren action executed.

## Security Notes

Recommended steady-state configuration:

```text
GoveeLife button:
Bluetooth only

ESP32 Bluetooth proxy:
Local Home Assistant infrastructure

H100:
IoT network

H100 Internet:
Blocked

Home Assistant -> H100:
Allowed as required
```

Do not expose the H100 or ESPHome Bluetooth proxies directly to the public Internet.

Permit only the network communication required for local operation.

## Final Architecture

```text
+----------------------+
| GoveeLife B5122      |
| HA model: H5122      |
+----------+-----------+
           |
           | Bluetooth
           v
+----------------------+
| ESP32 Bluetooth      |
| Proxy                |
+----------+-----------+
           |
           | ESPHome
           v
+----------------------+
| Home Assistant OS    |
|                      |
| Govee Bluetooth      |
| Automation           |
| TP-Link Smart Home   |
+----------+-----------+
           |
           | Local network
           v
+----------------------+
| TP-Link Tapo H100    |
|                      |
| Audible Chime        |
+----------------------+
```

Home Assistant provides the interoperability layer.

The GoveeLife button and Tapo H100 do not need to belong to the same vendor ecosystem or communicate directly.

## Full Documentation

This quickstart intentionally omits:

```text
Hardware cost analysis
Extended architecture rationale
Optional camera discussion
Potential future automations
Detailed provisioning observations
TP-Link product registration investigation
Account deletion survey
Authentication speculation
Firmware-management rationale
Component replacement discussion
Extended failure analysis
Design philosophy
```

See the full TP-Link Tapo H100 Local Home Assistant Doorbell Implementation documentation for those details.

## Related Search Keywords

home-assistant, tapo-h100, goveelife-b5122, govee-h5122, esphome, bluetooth-proxy, esp32, smart-doorbell, local-control, smart-home, home-automation, iot-security

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-08-31 | Initial GoveeLife and Tapo H100 Home Assistant doorbell quickstart. | projectfong |
