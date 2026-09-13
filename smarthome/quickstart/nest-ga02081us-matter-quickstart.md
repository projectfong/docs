# Google Nest Thermostat GA02081-US with Home Assistant Matter - Quickstart

Author: projectfong  
Copyright (c) 2026 Fong

---

## Summary

Connect a Google Nest Thermostat GA02081-US to Home Assistant OS using Matter while retaining a restricted network security model.

The tested deployment path is:

```text
GA02081-US
    |
    v
Google Home bootstrap
    |
    v
Firmware update if required
    |
    v
Google Matter fabric
    |
    v
Matter -> Manage
    |
    | Phone physically close
    v
Share to Home Assistant
    |
    v
Home Assistant Matter fabric
    |
    v
Verify local HA control
    |
    v
Apply restricted Internet policy
```

Home Assistant communicates with the thermostat using a local Matter CASE session over UDP 5540.

However, testing with firmware 2.2-9 showed that completely blocking thermostat Internet access eventually caused its operational IPv6/Matter state to fail.

Therefore:

```text
Home Assistant command path:
LOCAL

Completely Internet-independent operation:
NOT PROVEN

Internet-independent Matter operation:
DISPROVEN FOR THE TESTED CONFIGURATION
```

---

## Requirements

- Google Nest Thermostat GA02081-US.
- Home Assistant OS.
- Home Assistant Matter integration.
- Home Assistant Matter Server.
- Android Home Assistant Companion app.
- Google Home app.
- Working local IPv6.
- Working mDNS.
- Local routing between Home Assistant and the thermostat.
- Firewall policy permitting required local Matter traffic.
- Temporary Internet access for Google Home provisioning and commissioning.

---

## Important Findings

The documented testing established several non-obvious requirements:

```text
Google Home bootstrap:
Required for tested workflow

Phone physically close:
Required during successful Matter sharing

Thermostat Matter -> Manage:
Required during successful recommissioning

Home Assistant Matter:
Local CASE / UDP 5540

Google Matter fabric:
Can be removed individually

Google Home device relationship:
Separate from Google Matter fabric

Google Home -> Remove device:
Removed HA Matter fabric during testing

Complete Internet blocking:
Eventually caused Matter failure

Commissioning TCP 80:
Required during tested recommissioning
```

---

## Thermostat Preparation

### 1. Install the Thermostat

Install the GA02081-US according to the actual HVAC equipment and wiring.

Before modifying thermostat wiring:

```text
1. Turn off HVAC power.
2. Verify power is off with a multimeter.
3. Measure between R and C.
4. Confirm approximately 0 VAC.
5. Only then handle the conductors.
```

Do not rely solely on a non-contact voltage tester for 24 VAC HVAC wiring.

Avoid shorting:

```text
R -> C
```

or:

```text
R -> chassis/ground
```

### 2. Add the Thermostat to Google Home

The tested workflow requires the thermostat to be enrolled through Google Home before sharing it with Home Assistant.

The sequence is:

```text
Factory/default thermostat
        |
        v
Google Home
        |
        v
Firmware update if required
        |
        v
Google Matter fabric
        |
        v
Home Assistant Matter fabric
```

Do not assume Matter-capable firmware eliminates the Google Home bootstrap requirement after a factory reset.

---

## Commissioning Network Policy

### 3. Temporarily Allow Commissioning Internet Services

The restricted steady-state firewall policy was not sufficient during recommissioning.

The tested commissioning policy required:

```text
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

Conceptually:

```text
Source:
Nest thermostat

Destination:
Internet

Temporary commissioning services:
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

TCP 80 was observed during Google Home setup and was required for the tested recommissioning workflow.

It can be removed after commissioning.

### 4. Update Firmware

On the thermostat:

```text
Settings
-> Version
-> Check for update
```

Allow Internet access during the update.

Verify the installed firmware supports Matter.

The documented testing was performed using:

```text
2.2-9
```

Treat this as the tested version rather than assuming it remains the latest release.

---

## Verify Home Assistant Matter

### 5. Check Existing Matter Infrastructure

Before troubleshooting the Nest, verify:

```text
Matter integration loaded
Matter Server running
Local IPv6 operational
mDNS operational
Required routing operational
Home Assistant Companion app installed
```

If other Matter devices already operate correctly, that provides useful evidence that the base Home Assistant Matter infrastructure is functioning.

### 6. Verify Local Matter Policy

Observed Matter operational traffic used:

```text
UDP 5540
```

mDNS uses:

```text
UDP 5353
```

For segmented networks, provide the required:

```text
IPv4 routing where applicable
IPv6 routing
Firewall policy
mDNS reflection
```

Do not configure only:

```text
TCP 5540
```

and assume Matter is permitted.

---

## Prepare the Thermostat for Sharing

### 7. Open Matter Manage

Go physically to the thermostat.

Navigate to:

```text
Settings
-> Matter
-> Manage
```

Leave the thermostat on this screen.

Do not navigate away before completing the Google Home to Home Assistant Matter sharing process.

This was required during the successful recommissioning procedure documented for this thermostat.

### 8. Keep the Phone Close

Keep the Android phone physically close to the thermostat.

Verify:

```text
Bluetooth:
Enabled

Wi-Fi:
Enabled

Home Assistant Companion:
Installed

Google Play Services:
Operational

Required permissions:
Enabled
```

The successful tested state was:

```text
Thermostat:
Matter -> Manage

Phone:
Physically close

Google Home:
Ready to share Matter device
```

---

## Add the Thermostat to Home Assistant

### 9. Start Home Assistant Matter Addition

In the Home Assistant Companion app:

```text
Settings
-> Connectivity
-> Matter
-> Add device
```

Select:

```text
Yes, it's already in use
```

Then select:

```text
Google Home
```

This starts the Matter multi-admin workflow.

### 10. Share From Google Home

While the thermostat remains on:

```text
Matter -> Manage
```

open Google Home.

Navigate to:

```text
Nest Thermostat
-> Linked Matter apps & services
-> Link apps & services
-> Home Assistant
```

Complete the handoff while remaining physically close to the thermostat.

---

## Successful Commissioning

### 11. Verify Home Assistant

After successful commissioning, verify:

```text
Home Assistant Matter node appears
Thermostat is online
Current temperature updates
HVAC operating state updates
Setpoint changes work
HVAC mode changes work
Local thermostat changes appear in HA
```

Do not modify the commissioning firewall policy until Home Assistant control has been verified.

---

## Google Matter Fabric

### 12. Understand the Two Google Relationships

The Google Matter fabric and Google Home account relationship are separate.

```text
Google Matter fabric
!=
Google Home device membership
!=
Google account association
```

Removing only:

```text
Google LLC
```

from the thermostat's Matter fabric list preserved the Home Assistant Matter fabric during testing.

However, the thermostat remained:

```text
Present in Google Home
Associated with Google
Able to use Google cloud services
```

Removing the Google Matter fabric does not create a cloud-free thermostat.

### 13. Optional: Remove Only the Google Matter Fabric

After Home Assistant Matter control has been verified, the Google Matter fabric can be removed individually from the thermostat.

On the thermostat, locate:

```text
Google LLC
```

in Matter fabric management and use its specific remove control.

Expected transition:

```text
Before:

Google Home Matter fabric
Home Assistant Matter fabric

After:

Home Assistant Matter fabric
```

Home Assistant Matter remained operational immediately after this operation during testing.

---

## Critical Google Home Warning

Do not use:

```text
Google Home
-> Remove device
```

merely to eliminate Google cloud connectivity.

During testing, this caused:

```text
Google Home device removal
        |
        v
Thermostat Matter Leave
        |
        v
Home Assistant fabric removed
        |
        v
Thermostat removed from HA Matter
```

The Home Assistant Matter Server received:

```text
basicInformation.leave
```

Treat Google Home device removal as a decommissioning operation unless future testing proves otherwise.

---

## Apply Restricted Internet Policy

### 14. Remove TCP 80

After commissioning succeeds, remove:

```text
TCP 80
```

from the thermostat Internet policy.

The restricted steady-state services tested were:

```text
TCP 443
TCP 11095
TCP 53
UDP 53
```

Local Matter communication remains:

```text
UDP 5540
```

This represents the restricted policy tested in the documented environment.

It does not prove that every listed Internet service is permanently required.

### 15. Maintain Internal Segmentation

Conceptually:

```text
Nest -> Trusted internal networks
DENY

Nest -> HAOS Matter
ALLOW required local Matter traffic

Nest -> Required local infrastructure
ALLOW as required

Nest -> Internet
RESTRICT
```

Do not confuse restricted Internet access with unrestricted access to trusted internal networks.

---

## Do Not Completely Block Internet Yet

Testing showed that completely blocking thermostat Internet access was not stable.

Observed sequence:

```text
Internet blocked
        |
        v
Wi-Fi remained up
        |
        v
IPv4 remained reachable
        |
        v
Operational IPv6 eventually failed
        |
        v
Matter probes failed
        |
        v
HA Matter node went offline
        |
        v
Thermostat rebooted
        |
        v
Matter remained unavailable
        |
        v
Internet restored
        |
        v
New operational IPv6 address appeared
        |
        v
Matter service returned
        |
        v
HA CASE session automatically restored
```

Therefore:

```text
DO NOT ASSUME:

Local Matter control
=
Internet independence
```

The Home Assistant command path was local.

The thermostat itself still demonstrated an Internet dependency affecting operational Matter stability.

---

## Troubleshooting

### Google Home Says "Can't set up device"

First verify the physical commissioning state.

On the thermostat:

```text
Settings
-> Matter
-> Manage
```

Leave it there.

Then:

```text
Move the phone physically close to the thermostat.
```

Verify:

```text
Bluetooth enabled
Wi-Fi enabled
Home Assistant Companion installed
Google Play Services operational
Required permissions enabled
Commissioning Internet services allowed
```

Retry the handoff.

### Google Home Setup Does Not Complete

Verify temporary Internet access includes:

```text
TCP 80
TCP 443
TCP 11095
TCP 53
UDP 53
```

TCP 80 was required during the tested recommissioning procedure.

### Matter Server Shows No Commissioning Attempt

If Matter Server logs contain no:

```text
PASE
Commissioning
New node
Fabric addition
```

the failure may be occurring before Home Assistant Matter Server participates.

Check, in order:

```text
1. Thermostat is on Matter -> Manage.
2. Phone is physically close.
3. Bluetooth is enabled.
4. Companion app is being used.
5. Google Play Services are operational.
6. Google Home handoff is correct.
7. Phone permissions are correct.
8. Commissioning Internet access is available.
```

Only then move deeper into Home Assistant Matter Server troubleshooting.

### Matter Advertisement Is Visible but Pairing Fails

Check:

```bash
avahi-browse -rt _matterc._udp
```

A visible advertisement proves only:

```text
Matter commissioning advertisement exists
```

It does not prove:

```text
Bluetooth commissioning succeeded
Google Home handoff succeeded
Home Assistant received the request
PASE completed
Home Assistant fabric was added
```

### Matter Stops Working After Internet Is Blocked

Check IPv4 reachability:

```bash
ping <THERMOSTAT_IPV4>
```

Then inspect Matter Server logs for:

```text
All probes failed, closing session
```

```text
address is unreachable
```

```text
offline grace period expired
```

Inspect operational Matter advertisements:

```bash
avahi-browse -rt _matter._tcp
```

Do not assume:

```text
IPv4 ping succeeds
=
Matter is healthy
```

During testing:

```text
Wi-Fi:
UP

IPv4:
UP

IPv6 link-local:
UP

Operational IPv6:
FAILED

Home Assistant Matter:
OFFLINE
```

### Reboot Does Not Restore Matter

A thermostat reboot while Internet remained blocked did not restore Matter during testing.

Restoring Internet without another reboot caused:

```text
Operational IPv6 address returns
        |
        v
Matter advertisement returns
        |
        v
CASE session established
        |
        v
Home Assistant node online
```

If reproducing this test, do not assume reboot alone proves whether the underlying failure has been resolved.

### Home Assistant Matter Disappears After Google Home Removal

If the thermostat was removed using:

```text
Google Home
-> Remove device
```

the Home Assistant fabric may also have been removed.

During the documented test, Home Assistant received:

```text
basicInformation.leave
```

and removed the node.

Recommissioning will be required.

---

## Recommissioning

If rebuilding after Google Home device removal:

```text
Clean up stale Google account/device data
        |
        v
Temporarily allow commissioning Internet services
        |
        v
Add thermostat to Google Home
        |
        v
Update firmware if required
        |
        v
Open thermostat Matter -> Manage
        |
        v
Keep phone physically close
        |
        v
Start HA Matter addition
        |
        v
Google Home -> Link apps & services
        |
        v
Share to Home Assistant
        |
        v
Verify HA control
        |
        v
Remove TCP 80
        |
        v
Apply restricted steady-state Internet policy
```

---

## Validation Checklist

```text
[ ] Thermostat installed correctly
[ ] HVAC operation verified
[ ] Thermostat added to Google Home
[ ] Matter-capable firmware installed

[ ] Home Assistant Matter integration operational
[ ] Matter Server operational
[ ] Local IPv6 operational
[ ] Required routing operational
[ ] mDNS operational

[ ] Commissioning TCP 80 allowed
[ ] Commissioning TCP 443 allowed
[ ] Commissioning TCP 11095 allowed
[ ] DNS allowed

[ ] Thermostat opened to Matter -> Manage
[ ] Thermostat left on Matter -> Manage
[ ] Phone physically close
[ ] Bluetooth enabled
[ ] Wi-Fi enabled
[ ] Companion app used

[ ] Google Home multi-admin handoff succeeds
[ ] Home Assistant Matter node appears
[ ] Temperature updates
[ ] Setpoint control works
[ ] HVAC mode works
[ ] Local changes appear in HA

[ ] Google Matter fabric handling understood
[ ] Google Home Remove device NOT used for cloud isolation

[ ] TCP 80 removed after commissioning
[ ] Restricted Internet policy applied
[ ] Internal segmentation retained

[ ] Matter stability monitored
[ ] Operational IPv6 monitored
[ ] Home Assistant entity availability monitored
```

---

## Current Proven Architecture

```text
                    Local Matter
                    UDP 5540
                         |
                         v
Home Assistant OS <-> GA02081-US
Matter Server          Thermostat
                          |
                          | Restricted Internet
                          v
                    Google/Nest Services
```

The proven security model is:

```text
Nest -> Trusted internal networks:
DENY

Nest -> HAOS required Matter services:
ALLOW

Nest -> Required local infrastructure:
ALLOW as required

Nest -> Internet:
RESTRICT
```

---

## Proven vs. Unproven

### Proven During Testing

```text
HA Matter command path is local.
Matter uses UDP 5540.
GA02081-US uses Wi-Fi for normal Matter transport.
Google Home bootstrap is required for tested workflow.
Phone proximity matters during commissioning.
Matter -> Manage was required during recommissioning.
Google Matter fabric can be removed independently.
Removing Google Matter fabric preserves HA Matter.
Google Home account relationship remains afterward.
Complete Internet blocking eventually caused Matter failure.
IPv4 remained reachable during that failure.
Reboot with Internet blocked did not restore Matter.
Restoring Internet restored operational Matter.
Google Home Remove device removed HA Matter during testing.
TCP 80 was required during recommissioning.
TCP 80 could be removed afterward.
TCP 11095 traffic was observed.
```

### Not Yet Proven

```text
Fully Internet-independent long-term operation
Exact long-term necessity of TCP 11095
Exact long-term necessity of TCP 443
Exact steady-state DNS requirements
Cloud association removal without losing HA Matter
Future firmware behavior
```

Do not promote unresolved observations into deployment requirements until they are validated.

---

## Full Documentation

This quickstart intentionally omits:

```text
Cold-spare details
Detailed HVAC migration history
Full firmware history
Extended packet analysis
Neighbor Discovery evidence
Detailed IPv6 failure investigation
Firewall NAT explanation
Long-form testing chronology
Complete Matter log analysis
Extended Google account cleanup investigation
Detailed lessons learned
Open research questions
```

See the full Google Nest Thermostat GA02081-US Home Assistant Matter Setup and Testing documentation for the complete testing and troubleshooting record.

---

## Related Search Keywords

google-nest-thermostat, ga02081-us, home-assistant, haos, matter, google-home, matter-multi-admin, nest-matter, udp-5540, tcp-11095, mdns, avahi, ipv6, local-matter, bluetooth-commissioning, matter-manage, nest-internet-dependency

---

## Revision Control

| Version | Date | Summary | Author |
| --- | --- | --- | --- |
| **1.0.0** | 2026-09-13 | Initial GA02081-US Home Assistant Matter commissioning and restricted-network quickstart. | projectfong |
